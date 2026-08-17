# ADR-005: Entrega at-least-once com identificador de evento no header para deduplicação do cliente

**Status:** Aceito
**Data:** 2026-08-16
**Decisores:** Diego (Engenheiro Sênior, Plataforma), Larissa (Tech Lead)
**Consultados:** Sofia (Engenheira de Segurança), Bruno (Engenheiro Pleno), Marcos (Product Manager)
**ADRs relacionados:** [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md) descreve o mecanismo de reenvio que torna a duplicata possível, [ADR-001](ADR-001-padrao-outbox-no-mysql.md) define onde o identificador do evento é gerado

---

## Contexto

O desenho da feature torna a duplicata possível por construção. Diego colocou isso na mesa de forma explícita: a plataforma vai garantir at-least-once, então pode acontecer de o cliente receber o mesmo evento duas vezes, e ele precisa estar preparado ([09:24] Diego).

A duplicata não é acidente, é consequência de dois mecanismos decididos antes. O reenvio automático pode repetir um envio que o cliente já processou mas cuja resposta não chegou. E a queda do processo entre o envio e o registro do desfecho deixa o evento elegível para nova tentativa.

A pergunta que Bruno colocou, e que esta decisão responde, é como o cliente diferencia um evento novo de uma repetição ([09:25] Bruno).

## Decisão

Adotar a garantia de entrega **at-least-once**, sem tentar garantir exactly-once, e enviar um **identificador único do evento em header dedicado**, gerado no momento em que o evento entra na outbox ([09:25] Diego).

O identificador é único por evento e estável em todas as tentativas do mesmo evento, incluindo a reentrega que acontece após um replay de Dead Letter. A deduplicação acontece do lado do cliente, que descarta o evento cujo identificador já processou ([09:25] Diego).

A decisão vem acompanhada de um compromisso de produto: documentar a semântica de forma destacada no portal do desenvolvedor, para que o cliente saiba desde a integração que precisa deduplicar ([09:26] Marcos).

## Alternativas Consideradas

### Garantia exactly-once

Assegurar que cada evento chega exatamente uma vez ao cliente, eliminando a necessidade de deduplicação do lado dele.

Foi descartada porque exigiria coordenação dos dois lados e ficaria muito mais complexa. Diego argumentou que at-least-once com identificador de evento resolve 99 por cento dos casos e é o padrão adotado por plataformas de referência como Stripe e GitHub ([09:25] Diego).

**Trade-off que motivou o descarte:** ganharia uma promessa mais forte e uma integração mais simples para o cliente, ao custo de um protocolo de confirmação nos dois lados, com estado adicional para manter e novos modos de falha para tratar, desproporcional ao problema que resolve.

### Deduplicação no lado da plataforma, antes do envio

Manter registro do que já foi confirmado como entregue e suprimir o reenvio de qualquer evento nessa condição, poupando o cliente de receber repetido.

Não foi discutida na reunião, mas é a alternativa plausível imediata e merece registro, porque ela não resolve o problema real. A duplicata que importa nasce justamente da incerteza sobre o desfecho: a plataforma envia, o cliente processa, e a confirmação se perde por queda de processo ou timeout de rede. Nesse cenário a plataforma não sabe que houve entrega e reenvia de qualquer forma. Deduplicar antes do envio só elimina duplicatas que a plataforma já sabe serem desnecessárias, que não são o caso problemático.

**Trade-off que motivou o descarte:** ganharia redução marginal de envios repetidos, ao custo de estado adicional e da falsa impressão de que o cliente pode dispensar a deduplicação, o que é justamente o risco que se quer evitar.

## Consequências

### Positivas

- O desenho fica simples e previsível: a plataforma tenta até conseguir, e o identificador dá ao cliente a chave para lidar com o excedente.
- É o padrão de mercado, o que reduz a fricção de integração, porque integradores B2B já conhecem o modelo de outras plataformas ([09:25] Diego).
- Combina naturalmente com o reenvio automático de [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md): reenviar passa a ser seguro, e não uma operação de risco.
- O replay administrativo de um evento em Dead Letter preserva o identificador original, então para o cliente é uma reentrega do mesmo evento e a deduplicação continua funcionando sem caso especial.
- O identificador serve também como chave de correlação interna, ligando a gravação do evento, cada tentativa de entrega e o eventual registro na Dead Letter.

### Negativas e trade-offs explícitos

- **A responsabilidade da deduplicação é transferida para o cliente.** Sofia apontou isso na reunião, e a resposta foi que se trata do padrão de mercado ([09:25] Sofia, [09:25] Diego). O trade-off aceito é empurrar complexidade para fora em troca de manter a plataforma simples.
- **A plataforma não tem como garantir que o cliente deduplique.** Um cliente que ignore o header processa evento repetido e executa duas vezes a mesma ação de negócio, e a plataforma só descobre pelo relato dele.
- **O compromisso de documentação vira dependência de entrega.** Se a documentação no portal não sair, o trade-off aceito na reunião se converte em problema de suporte ([09:26] Marcos).
- **Nenhuma garantia adicional é oferecida para clientes que não conseguem deduplicar.** Não existe modo alternativo de entrega para quem não implementar a verificação.

## Referências

**Transcrição:** [09:24] Diego, [09:25] Bruno, [09:25] Diego, [09:25] Sofia, [09:26] Marcos, [09:26] Larissa, [09:44] Diego, [09:48] Larissa

**Código existente:**

| Arquivo | Relevância |
| --- | --- |
| [src/middlewares/request-logger.middleware.ts](../../src/middlewares/request-logger.middleware.ts) | Padrão já usado no projeto de gerar um identificador por requisição e devolvê-lo em header, mesmo princípio aplicado ao identificador de evento |
| [package.json](../../package.json) | Biblioteca de geração de identificador já é dependência direta do projeto, portanto nenhuma dependência nova é necessária |
