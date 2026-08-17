# Architectural Decision Records

Este diretório reúne os ADRs da feature **Sistema de Webhooks de Notificação de Pedidos**, extraídos da reunião técnica registrada em [TRANSCRICAO.md](../../TRANSCRICAO.md).

Cada ADR segue o formato MADR, com as seções Status, Contexto, Decisão, Alternativas Consideradas e Consequências, e registra o trade-off de forma explícita. As referências no formato `[hh:mm] Nome` apontam para a fala de origem na transcrição, e as referências de arquivo apontam para código que existe hoje no repositório.

## Índice

| ADR | Decisão | Origem principal na transcrição |
| --- | --- | --- |
| [ADR-001](ADR-001-padrao-outbox-no-mysql.md) | Padrão Outbox no MySQL existente, com o evento gravado na mesma transação da mudança de status | [09:03] a [09:08], [09:40] a [09:41] |
| [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md) | Worker em processo separado, lendo a outbox em polling de 2 segundos, com um único worker nesta fase | [09:08] a [09:14], [09:28] a [09:30] |
| [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md) | Reenvio com backoff exponencial, cinco tentativas e Dead Letter em tabela separada, com replay manual restrito a ADMIN | [09:14] a [09:18], [09:35] a [09:36] |
| [ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md) | Assinatura HMAC-SHA256 com secret exclusiva por endpoint e rotação com grace period de 24 horas | [09:19] a [09:24] |
| [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md) | Entrega at-least-once com identificador de evento no header para deduplicação do cliente | [09:24] a [09:26] |
| [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md) | Reuso máximo dos padrões existentes do projeto, sem dependência nova e sem alteração no middleware de erro | [09:27] a [09:30], [09:51] |
| [ADR-007](ADR-007-snapshot-do-payload-na-insercao.md) | Payload renderizado e congelado no momento da gravação na outbox | [09:43], [09:51] a [09:52] |

## Como os ADRs se relacionam

O [ADR-001](ADR-001-padrao-outbox-no-mysql.md) é a decisão raiz, e as demais decorrem dela ou a complementam. O [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md) define quem consome a outbox, o [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md) define o que acontece quando a entrega falha, o [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md) explica por que reenviar é seguro e o [ADR-007](ADR-007-snapshot-do-payload-na-insercao.md) define o que exatamente é gravado. O [ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md) trata da segurança da entrega, e o [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md) atravessa todos os outros, definindo como cada decisão é construída dentro da codebase existente.

## Decisões que não viraram ADR

Alguns pontos foram fechados na reunião mas permaneceram como detalhe de implementação, documentados no [FDD](../FDD.md) em vez de ADR próprio:

- Formato e campos do payload de entrega ([09:43] Diego), tratado dentro do [ADR-007](ADR-007-snapshot-do-payload-na-insercao.md) e detalhado no FDD.
- Conjunto de headers enviados na entrega ([09:44] Diego, [09:44] Sofia).
- Timeout de 10 segundos por chamada ([09:42] Diego), registrado no [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md) como definição do que conta como falha.
- Exigência de https e limite de 64KB por payload, que a própria reunião classificou como requisito e não como decisão arquitetural separada ([09:23] Sofia, [09:24] Larissa), registrados no [ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md) por pertencerem ao mesmo problema de segurança.
- Uso de UUID nos identificadores ([09:51] Larissa), coberto pelo [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md) como parte do reuso de padrões.

## Documentos relacionados

[PRD](../PRD.md) responde por que e o quê, o [RFC](../RFC.md) apresenta a proposta técnica e as alternativas submetidas à revisão, o [FDD](../FDD.md) detalha como construir e o [TRACKER](../TRACKER.md) mapeia a origem de cada item.
