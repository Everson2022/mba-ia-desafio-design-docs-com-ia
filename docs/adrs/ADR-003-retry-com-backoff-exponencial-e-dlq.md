# ADR-003: Reenvio com backoff exponencial, cinco tentativas e Dead Letter em tabela separada

**Status:** Aceito
**Data:** 2026-08-16
**Decisores:** Diego (Engenheiro Sênior, Plataforma), Larissa (Tech Lead), Bruno (Engenheiro Pleno, Pedidos)
**Consultados:** Marcos (Product Manager), Sofia (Engenheira de Segurança)
**ADRs relacionados:** [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md) descreve o processo que executa as tentativas, [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md) explica por que o reenvio é seguro do ponto de vista do cliente

---

## Contexto

A entrega depende de um endpoint que está fora da infraestrutura da empresa e pode estar indisponível por motivos que a plataforma não controla. A pergunta colocada foi direta: se o cliente está offline, o que a plataforma faz ([09:14] Larissa).

Duas restrições reais orientaram a discussão:

- Existe precedente concreto de cliente com indisponibilidade planejada de duas horas, em manutenção ([09:16] Diego).
- Um evento que fica pendurado para sempre, porque o cliente sumiu, é um problema operacional próprio ([09:15] Diego).

O timeout por chamada também entra aqui, porque define o que conta como falha: 10 segundos, e cliente lento que não responde nesse prazo é tratado como falha e marcado para reenvio ([09:42] Diego).

## Decisão

Adotar **reenvio automático com backoff exponencial, limitado a cinco tentativas**, com a progressão de **1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas**, totalizando cerca de 15 horas entre a primeira falha e a última tentativa ([09:17] Diego).

Esgotadas as cinco tentativas, o evento é movido para uma **tabela separada de Dead Letter**, guardando o payload, o motivo da falha e o timestamp ([09:18] Diego).

A recuperação é manual, por **endpoint administrativo de replay**, que recoloca o evento na outbox como pendente ([09:18] Diego, [09:35] Diego). O endpoint exige role ADMIN e registra quem executou, porque mexer em fila de entrega de notificação não é operação de operador comum ([09:36] Sofia).

Conta como falha para efeito de reenvio: resposta fora da faixa de sucesso, erro de rede e estouro do timeout de 10 segundos ([09:42] Diego).

## Alternativas Consideradas

### Três tentativas em vez de cinco

Bruno propôs três tentativas, argumentando que seria mais agressivo ([09:16] Bruno).

Foi descartada porque três tentativas encerrariam o evento em cerca de 30 minutos. Diego apontou que, se o cliente teve indisponibilidade pela manhã, a plataforma retentaria três vezes em meia hora e mataria o evento, e que já houve cliente com indisponibilidade de duas horas em manutenção planejada ([09:16] Diego).

**Trade-off que motivou o descarte:** ganharia detecção mais rápida de cliente permanentemente quebrado e menos linhas em reenvio, ao custo de perder eventos de clientes que estavam apenas temporariamente fora, que é justamente o caso mais comum.

### Reenvio indefinido com backoff

Continuar tentando para sempre, com intervalos crescentes, sem teto de tentativas. Foi mencionada como posição defendida por parte do mercado ([09:15] Diego).

Foi descartada porque traz o problema de o evento ficar pendurado para sempre caso o cliente tenha sumido ([09:15] Diego).

**Trade-off que motivou o descarte:** ganharia entrega eventual em qualquer cenário de recuperação do cliente, ao custo de acúmulo indefinido de eventos vivos, sem nenhum ponto em que o sistema declare falha e peça intervenção humana.

### Marcar falha permanente na própria outbox, sem tabela separada

Larissa colocou a pergunta de forma explícita: fazer a Dead Letter em tabela separada ou apenas marcar o evento como falho na própria outbox ([09:17] Larissa).

Foi descartada em favor da tabela separada, que mantém mais limpa a leitura da outbox principal e preserva o evento como evidência para diagnóstico e reprocessamento ([09:18] Diego).

**Trade-off que motivou o descarte:** ganharia um modelo de dados menor, com uma tabela a menos, ao custo de misturar na mesma tabela os eventos do fluxo ativo e os que já encerraram, penalizando a consulta que o worker executa a cada 2 segundos.

## Consequências

### Positivas

- Cobre indisponibilidade real do cliente sem nenhuma intervenção manual, com janela de aproximadamente 15 horas, dimensionada a partir de um caso concreto e não de suposição.
- Nenhum evento é descartado em silêncio. O que não foi entregue fica persistido com payload e motivo, o que permite diagnóstico e recuperação.
- A leitura da outbox continua enxuta, porque o que encerrou sai do fluxo ativo.
- A recuperação é possível a qualquer momento, sem depender de o cliente reportar a falta.
- A restrição de ADMIN no replay, com registro de quem executou, dá trilha de auditoria para uma operação sensível ([09:36] Sofia).

### Negativas e trade-offs explícitos

- **Um evento pode levar cerca de 15 horas até ser considerado falha permanente.** Nesse cenário o cliente recebe a notificação com atraso grande, ou não recebe. A reunião aceitou por considerar que um cliente fora do ar por 15 horas já enfrenta problema próprio grave ([09:17] Marcos).
- **A recuperação depende de ação humana.** Não existe aviso automático quando um evento cai na Dead Letter, porque o email de alerta foi descartado nesta fase ([09:37] Larissa). O trade-off é assumir que a descoberta depende de consulta ativa em troca de reduzir o escopo desta entrega.
- **Mais um ponto de persistência para modelar, migrar e operar.**
- **Não há distinção entre falha do cliente e falha permanente de contrato.** Uma resposta de erro por endpoint inexistente consome as mesmas cinco tentativas ao longo de 15 horas que uma indisponibilidade temporária consumiria.
- **Sem controle de vazão na saída**, o reenvio pode se somar ao tráfego normal e concentrar chamadas em um cliente que está justamente se recuperando. O controle de vazão foi adiado de forma consciente ([09:38] Diego, [09:39] Larissa).

## Referências

**Transcrição:** [09:14] Larissa, [09:15] Diego, [09:15] Bruno, [09:16] Bruno, [09:16] Diego, [09:17] Diego, [09:17] Larissa, [09:17] Marcos, [09:18] Diego, [09:18] Bruno, [09:35] Diego, [09:36] Sofia, [09:36] Larissa, [09:37] Larissa, [09:42] Diego, [09:48] Larissa

**Código existente:**

| Arquivo | Relevância |
| --- | --- |
| [src/middlewares/auth.middleware.ts](../../src/middlewares/auth.middleware.ts) | Middleware de autorização por papel, reusado para restringir o replay a ADMIN |
| [src/shared/logger/index.ts](../../src/shared/logger/index.ts) | Logger usado para registrar quem executou o replay, exigência de auditoria |
