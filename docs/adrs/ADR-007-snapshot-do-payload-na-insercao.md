# ADR-007: Payload renderizado e congelado no momento da gravação na outbox

**Status:** Aceito
**Data:** 2026-08-16
**Decisores:** Larissa (Tech Lead), Diego (Engenheiro Sênior, Plataforma), Bruno (Engenheiro Pleno, Pedidos)
**ADRs relacionados:** [ADR-001](ADR-001-padrao-outbox-no-mysql.md) define a tabela onde o payload é gravado, [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md) explica por que a distância entre gravação e entrega pode chegar a horas

---

## Contexto

Com a outbox definida, restava uma pergunta de modelagem que Bruno levantou no fim da reunião: o evento guarda o payload já renderizado, ou guarda apenas a referência ao pedido e renderiza na hora do envio ([09:51] Bruno).

A pergunta não é acadêmica. Entre a gravação do evento e a entrega efetiva pode passar bastante tempo. No caminho feliz são 2 segundos, mas com reenvio a distância chega a cerca de 15 horas ([09:17] Diego), e um evento que passou pela Dead Letter e foi reprocessado por replay pode ser entregue muito depois disso ([09:35] Diego). Nesse intervalo, o pedido pode mudar de novo.

## Decisão

Renderizar o payload **no momento da gravação** e gravá-lo já serializado na outbox. O evento passa a ser um retrato do pedido no instante exato da transição de status ([09:52] Larissa, [09:52] Diego, [09:52] Bruno).

O conteúdo do payload é enxuto, com identificação do evento, tipo, timestamp, dados básicos do pedido e a transição de status. Os itens do pedido não vão no payload, para não inflar. O cliente que precisar do detalhe consulta a API de pedidos depois ([09:43] Diego).

Consequência direta desta decisão: uma alteração posterior no pedido não altera nenhum evento já gravado, e todas as tentativas de entrega do mesmo evento carregam exatamente o mesmo corpo.

## Alternativas Consideradas

### Guardar apenas a referência ao pedido e renderizar na hora do envio

Foi a alternativa colocada explicitamente por Bruno, guardar só o identificador do pedido e montar o payload no momento do envio ([09:51] Bruno). Economizaria espaço em disco, evitaria duplicar dados que já estão na tabela de pedidos e garantiria que o cliente sempre receba o estado mais recente.

Foi descartada por Larissa, que preferiu o payload renderizado já na gravação: se o pedido mudar depois, o evento ainda reflete o estado de quando o status mudou, e sem isso surgem casos estranhos ([09:52] Larissa). Diego concordou e fechou como snapshot na inserção ([09:52] Diego).

O caso estranho é concreto. Um evento de transição para expedido, reenviado 12 horas depois, renderizado na hora do envio, chegaria ao cliente com o status atual do pedido, que a essa altura pode já ser outro. O cliente receberia um evento que diz que o pedido foi expedido carregando dados de um pedido que já foi entregue, ou cancelado.

**Trade-off que motivou o descarte:** ganharia economia de armazenamento e ausência de duplicação de dados, ao custo de tornar o conteúdo do evento dependente do momento da entrega, o que quebra a própria semântica de notificação de evento.

### Payload completo, incluindo os itens do pedido

Alternativa implícita na discussão do formato do payload. Entregaria ao cliente tudo de uma vez, evitando que ele precise fazer uma chamada adicional.

Foi descartada por decisão de formato: não mandar itens para não inflar, e quem quiser detalhes consulta a API de pedidos depois ([09:43] Diego). Bruno concordou, observando que isso mantém o payload enxuto ([09:44] Bruno).

**Trade-off que motivou o descarte:** ganharia autonomia para o cliente, sem chamada de volta, ao custo de payload proporcional ao tamanho do pedido, com risco real de aproximar do teto de 64KB e com duplicação de dados que crescem em disco a cada evento.

## Consequências

### Positivas

- O evento é fiel ao momento da transição, que é a definição do que uma notificação de evento deveria ser.
- Todas as tentativas do mesmo evento entregam corpo idêntico, o que faz a assinatura permanecer válida e a deduplicação do cliente funcionar de forma previsível.
- A entrega não depende de o pedido ainda existir ou estar acessível no momento do envio. O worker não precisa consultar a tabela de pedidos para montar a requisição.
- O payload preservado na Dead Letter é exatamente o que deveria ter sido entregue, o que torna o replay uma reentrega fiel e não uma nova renderização.
- O payload enxuto mantém distância confortável do limite de 64KB, mesmo em pedidos grandes.

### Negativas e trade-offs explícitos

- **O conteúdo entregue pode estar defasado em relação ao estado atual do pedido.** Em um reenvio que ocorre horas depois, o cliente recebe o retrato do momento da transição, não o retrato de agora. É a contrapartida direta e aceita da fidelidade ao evento, e fica atenuada porque o payload é enxuto e o cliente pode consultar a API de pedidos quando precisar do estado corrente.
- **Duplicação de dados em disco.** O mesmo conteúdo passa a existir na tabela de pedidos e em cada linha de evento gerada, e como cada endpoint interessado gera uma linha própria, um pedido com vários endpoints multiplica a duplicação. O efeito se soma ao crescimento da outbox já registrado em [ADR-001](ADR-001-padrao-outbox-no-mysql.md).
- **Mudança de formato do payload não retroage.** Eventos já gravados permanecem no formato antigo, então uma alteração de formato precisa conviver com os dois enquanto houver evento pendente ou em Dead Letter.
- **O cliente que precisar dos itens do pedido faz uma chamada adicional**, o que aumenta o tráfego de volta para a API justo no momento em que ele reage à notificação.

## Referências

**Transcrição:** [09:17] Diego, [09:35] Diego, [09:43] Diego, [09:44] Bruno, [09:51] Bruno, [09:52] Larissa, [09:52] Diego, [09:52] Bruno

**Código existente:**

| Arquivo | Relevância |
| --- | --- |
| [src/modules/orders/order.service.ts](../../src/modules/orders/order.service.ts) | O pedido recarregado com relações ao final da transação é a fonte dos dados do snapshot |
| [prisma/schema.prisma](../../prisma/schema.prisma) | Modelo do pedido, que define quais campos existem para compor o payload, incluindo número do pedido, cliente e total |
