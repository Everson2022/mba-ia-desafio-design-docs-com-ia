# ADR-004: Assinatura HMAC-SHA256 com secret exclusiva por endpoint e rotação com grace period de 24 horas

**Status:** Aceito
**Data:** 2026-08-16
**Decisores:** Sofia (Engenheira de Segurança), Larissa (Tech Lead)
**Consultados:** Diego (Engenheiro Sênior), Bruno (Engenheiro Pleno), Marcos (Product Manager)
**ADRs relacionados:** [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md) descreve o processo que assina e envia, [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md) descreve como a validação de https e os códigos de erro se encaixam nos padrões do projeto

---

## Contexto

A feature expõe eventos com dados de pedidos para endpoints que estão fora da infraestrutura da empresa. O cliente precisa conseguir validar duas coisas: que a requisição veio realmente da plataforma e que ninguém adulterou o payload no caminho ([09:19] Sofia).

Existe ainda um precedente concreto que pesa na decisão: a equipe já teve cliente que vazou secret no log da própria aplicação ([09:22] Diego).

## Decisão

Assinar o corpo de cada requisição com **HMAC-SHA256**, enviando a assinatura em um header dedicado, para que o cliente verifique do lado dele ([09:20] Sofia).

Três características fazem parte desta decisão:

- **Algoritmo HMAC-SHA256.** Escolhido por ser o padrão de mercado, com biblioteca disponível em qualquer stack séria ([09:20] Sofia).
- **Secret exclusiva por endpoint cadastrado**, nunca uma secret global da plataforma, porque com secret global o vazamento de uma comprometeria todas ([09:21] Sofia). A secret é gerada pelo sistema e devolvida ao cliente na criação do cadastro ([09:31] Marcos).
- **Rotação com grace period de 24 horas.** O cliente pede uma nova secret pela API, e a anterior permanece válida em paralelo por 24 horas, para ele ter tempo de migrar os sistemas. Passado esse prazo, a anterior morre ([09:21] Sofia).

Duas medidas complementares foram decididas junto, e ficam registradas aqui porque pertencem ao mesmo problema de segurança, ainda que a reunião tenha classificado ambas como requisito e não como decisão arquitetural separada:

- **TLS obrigatório.** A URL do webhook precisa ser https, e cadastro com http é recusado na validação de schema ([09:23] Sofia).
- **Limite de 64KB por payload**, com erro em vez de truncamento, porque um evento desse tamanho indica que algo está errado ([09:23] Sofia, [09:24] Diego, [09:24] Larissa).

## Alternativas Consideradas

### Secret global única da plataforma

Uma única secret compartilhada por toda a plataforma, distribuída a todos os clientes. Simplificaria a geração, o armazenamento e a rotação, que passaria a ser um único evento operacional.

Foi descartada de forma direta por Sofia: cada endpoint de webhook do cliente tem que ter secret única, porque se vaza uma, vaza tudo ([09:21] Sofia).

**Trade-off que motivou o descarte:** ganharia simplicidade de gestão, com uma secret para guardar em vez de uma por endpoint, ao custo de transformar qualquer vazamento isolado em comprometimento de todos os clientes ao mesmo tempo.

### Rotação imediata, sem período de convivência

Trocar a secret e invalidar a anterior no mesmo instante. É a forma mais simples de implementar, sem campo adicional, sem prazo de expiração e sem assinatura dupla no período de transição.

Foi descartada porque criaria janela de indisponibilidade na verificação do lado do cliente: entre a rotação e a atualização do sistema dele, toda entrega chegaria com assinatura que ele não consegue validar. A decisão explícita foi manter a antiga válida por 24 horas em paralelo, justamente para dar tempo de migração ([09:21] Sofia).

**Trade-off que motivou o descarte:** ganharia um modelo mais simples e uma superfície de validação menor, ao custo de tornar a rotação uma operação arriscada, o que na prática desencorajaria o cliente a rotacionar mesmo diante de suspeita de vazamento.

### Truncar o payload em vez de recusar quando ultrapassar o limite

Sofia colocou a alternativa de forma explícita ao discutir o limite de tamanho, perguntando se a plataforma deveria truncar ou errar, e se posicionou a favor de errar ([09:23] Sofia).

Foi descartada porque um evento que chega a esse tamanho indica que algo está errado, e entregar um payload truncado enviaria ao cliente um evento silenciosamente incompleto, que ele não teria como distinguir de um evento íntegro.

**Trade-off que motivou o descarte:** o truncamento ganharia entrega em qualquer cenário, ao custo de corromper a semântica do evento e de esconder um defeito em vez de expô-lo.

## Consequências

### Positivas

- O cliente consegue verificar origem e integridade de cada entrega sem nenhuma coordenação adicional com a plataforma.
- O alcance de um vazamento fica contido em um único endpoint, e não na base inteira.
- A rotação é uma operação segura, o que torna realista pedir ao cliente que rotacione diante de suspeita de vazamento, que é exatamente o caso já vivido pela equipe ([09:22] Diego).
- HMAC-SHA256 não exige biblioteca nova, porque o runtime já oferece o algoritmo de forma nativa.
- A exigência de https é validada no schema de entrada, portanto falha cedo, na borda, sem chegar ao banco ([09:23] Sofia).

### Negativas e trade-offs explícitos

- **A plataforma assume a guarda de uma secret por endpoint cadastrado.** Passa a existir material sensível armazenado em volume proporcional ao número de clientes e endpoints, com todas as obrigações que isso implica.
- **Durante as 24 horas de convivência existem duas secrets válidas para o mesmo endpoint.** É ampliação temporária e deliberada da superfície de validação, aceita em troca de rotação sem indisponibilidade.
- **A verificação fica do lado do cliente.** A plataforma assina, mas não tem como garantir que o cliente valida. Um cliente que ignore o header continua funcionando, sem nenhuma proteção efetiva.
- **A implementação da assinatura é um ponto de falha concentrado.** Assinatura calculada sobre um corpo diferente do que é enviado quebra a verificação de todos os clientes de uma vez, o que motiva a exigência de revisão de segurança dedicada antes do deploy, com no mínimo dois dias úteis reservados ([09:46] Sofia).
- **Evento acima de 64KB não é entregue.** Como a recusa acontece na gravação, ela provoca rollback da transação de mudança de status, o que transforma um problema de tamanho de payload em bloqueio de operação de negócio.

## Referências

**Transcrição:** [09:19] Sofia, [09:20] Sofia, [09:20] Bruno, [09:21] Sofia, [09:21] Bruno, [09:22] Diego, [09:22] Sofia, [09:23] Sofia, [09:24] Diego, [09:24] Larissa, [09:31] Marcos, [09:44] Sofia, [09:46] Sofia, [09:48] Larissa

**Código existente:**

| Arquivo | Relevância |
| --- | --- |
| [src/middlewares/validate.middleware.ts](../../src/middlewares/validate.middleware.ts) | Middleware de validação onde a exigência de https é aplicada, no mesmo padrão dos demais módulos |
| [src/shared/logger/index.ts](../../src/shared/logger/index.ts) | Lista de redação do logger, que hoje cobre senha, token e header de autorização e precisa passar a cobrir a secret |
| [src/shared/errors/http-errors.ts](../../src/shared/errors/http-errors.ts) | Classes de erro existentes que servem de base para os erros de validação de URL e de tamanho de payload |
