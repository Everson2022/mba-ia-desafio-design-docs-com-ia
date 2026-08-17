# ADR-001: Padrão Outbox no MySQL existente

**Status:** Aceito
**Data:** 2026-08-16
**Decisores:** Diego (Engenheiro Sênior, Plataforma), Larissa (Tech Lead), Bruno (Engenheiro Pleno, Pedidos)
**Consultados:** Marcos (Product Manager), Sofia (Engenheira de Segurança)
**ADRs relacionados:** [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md) descreve quem consome a outbox, [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md) descreve o que acontece quando a entrega falha, [ADR-007](ADR-007-snapshot-do-payload-na-insercao.md) descreve o conteúdo gravado

---

## Contexto

O OMS precisa notificar clientes B2B sempre que o status de um pedido muda ([09:00] Marcos). A mudança de status hoje acontece dentro de uma única transação em [order.service.ts:131-178](../../src/modules/orders/order.service.ts#L131-L178), que valida a transição pela máquina de estados, debita ou repõe estoque, atualiza a tabela de pedidos e grava o histórico.

A primeira pergunta colocada na reunião foi exatamente onde o disparo entra: dentro do serviço de pedidos de forma síncrona, ou em algum mecanismo assíncrono ([09:03] Larissa).

Três tensões precisam ser resolvidas ao mesmo tempo:

- **Consistência.** O evento precisa existir se e somente se a mudança de status foi confirmada. Não pode existir status alterado sem evento correspondente ([09:40] Bruno).
- **Desacoplamento.** A disponibilidade e a latência do sistema do cliente não podem afetar a operação interna do OMS ([09:04] Bruno).
- **Custo operacional.** O time é pequeno, e cada componente novo de infraestrutura tem custo real de provisionamento, monitoramento e operação ([09:07] Diego).

## Decisão

Adotar o **padrão Outbox sobre o MySQL que já está em produção**.

Quando o status do pedido muda, dentro da mesma transação SQL que atualiza o pedido e grava o histórico, o sistema também grava o evento em uma tabela de outbox ([09:06] Diego). Um processo separado lê essa tabela e executa as chamadas HTTP, fora da transação e fora do ciclo de requisição da API.

A garantia que isso dá foi descrita textualmente na reunião: se a transação principal commitou, o evento foi registrado, e se ela sofreu rollback, o evento some junto. Não existe inconsistência possível ([09:06] Diego).

Detalhes que fazem parte desta decisão:

- A gravação é feita por uma função que recebe o cliente transacional em uso, sem injetar o repositório de webhooks dentro do serviço de pedidos ([09:41] Bruno, [09:41] Diego).
- O filtro por status acontece na gravação, não no envio. Se nenhum endpoint do cliente ouve aquele status, nenhuma linha é gravada, o que economiza linha na tabela ([09:34] Bruno, [09:34] Diego).
- A tabela tem índice no campo de estado, com os valores pendente, processando, falhou e entregue, e índice em data de criação. O leitor busca apenas os pendentes, em lote pequeno ([09:08] Diego).
- Falha na gravação propaga a exceção e provoca rollback da transação inteira, inclusive da mudança de status. Esse é o comportamento desejado, não um efeito colateral ([09:40] Bruno, [09:41] Diego).

## Alternativas Consideradas

### Disparo síncrono dentro da transação de mudança de status

Chamar o endpoint HTTP do cliente diretamente de dentro da transação, antes do commit. É a alternativa mais simples de implementar, sem tabela nova e sem processo novo.

Foi descartada por dois motivos levantados na reunião. Primeiro, a transação de mudança de status já é pesada, porque atualiza o pedido, insere no histórico e decrementa o estoque dos produtos, e acrescentar uma chamada HTTP no meio disso faria qualquer cliente lento travar mudança de status para outros pedidos ([09:04] Bruno). Segundo, não existe resposta boa para o caso do cliente estar fora do ar: dar rollback na mudança de status não é aceitável, porque a mudança de negócio aconteceu, e ignorar a falha em silêncio viola o requisito de confiabilidade ([09:04] Bruno).

**Trade-off que motivou o descarte:** ganharia simplicidade de implementação e entrega imediata, ao custo de acoplar a disponibilidade da operação interna à disponibilidade de terceiros, o que é inaceitável para a operação mais crítica do sistema.

### Fila ou stream externo, como Redis Streams

Publicar o evento em um broker externo depois do commit da transação. É o padrão clássico de desacoplamento entre sistemas.

Foi descartada porque exigiria subir mais infraestrutura ([09:07] Larissa) e porque, nas palavras de Diego, o time é pequeno e subir um Redis Cluster para isso é overengineering, já que a outbox no MySQL existente resolve ([09:07] Diego).

Há também um problema de consistência que a reunião não precisou detalhar, mas que é inerente à alternativa: a publicação no broker acontece depois do commit, então uma queda do processo entre o commit e a publicação deixaria o status alterado sem evento publicado. É exatamente a lacuna que o padrão Outbox fecha.

**Trade-off que motivou o descarte:** ganharia capacidade de escala e recursos prontos de fila, ao custo de nova infraestrutura para operar e de uma janela de inconsistência entre banco e broker.

## Consequências

### Positivas

- Atomicidade garantida entre a mudança de status e o registro do evento. Não existe cenário em que os dois divirjam.
- Zero infraestrutura nova. O MySQL já está provisionado, monitorado e com rotina de backup.
- Nenhuma dependência npm nova, já que a gravação usa o mesmo cliente Prisma da transação.
- O estado do evento fica visível e consultável em tabela relacional, o que facilita diagnóstico, auditoria e consulta ad hoc, coisa que um broker externo esconderia.
- Custo proporcional ao interesse real: cliente sem endpoint interessado no status não gera linha nenhuma.

### Negativas e trade-offs explícitos

- **A tabela cresce continuamente.** Linhas já entregues se acumulam até que exista política de arquivamento, e essa política foi explicitamente declarada fora do escopo desta feature ([09:08] Diego). O trade-off aceito é conviver com crescimento monitorado agora em troca de não adiar a entrega.
- **A operação mais crítica do OMS passa a depender de um módulo novo.** Como o rollback conjunto é o comportamento desejado, uma falha na gravação do evento derruba a mudança de status. O trade-off aceito é assumir esse acoplamento em troca da garantia de consistência, com a contenção de manter a operação dentro da transação restrita a uma consulta e inserções simples ([09:41] Bruno).
- **A latência mínima passa a depender do consumidor.** O evento fica na tabela até o processo leitor buscá-lo, o que introduz um piso de latência tratado em [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md).
- **A transação de mudança de status fica marginalmente mais longa**, com uma consulta e algumas inserções a mais, o que aumenta levemente o tempo de retenção de lock.

## Referências

**Transcrição:** [09:03] Larissa, [09:04] Bruno, [09:06] Diego, [09:07] Larissa, [09:07] Diego, [09:08] Diego, [09:34] Bruno, [09:40] Bruno, [09:41] Bruno, [09:41] Diego, [09:48] Larissa

**Código existente:**

| Arquivo | Relevância |
| --- | --- |
| [src/modules/orders/order.service.ts](../../src/modules/orders/order.service.ts) | Transação de mudança de status, ponto exato onde a gravação do evento é acoplada |
| [src/modules/orders/order.status.ts](../../src/modules/orders/order.status.ts) | Máquina de estados que define quais transições existem e, portanto, quais eventos podem ser gerados |
| [prisma/schema.prisma](../../prisma/schema.prisma) | Padrão de modelagem a seguir na tabela nova, incluindo UUID, índices e nome de tabela em snake_case |
