# ADR-006: Reuso máximo dos padrões existentes do projeto no módulo de webhooks

**Status:** Aceito
**Data:** 2026-08-16
**Decisores:** Bruno (Engenheiro Pleno, Pedidos), Larissa (Tech Lead), Diego (Engenheiro Sênior, Plataforma)
**Consultados:** Sofia (Engenheira de Segurança)
**ADRs relacionados:** este ADR define como todos os demais são construídos dentro da codebase

---

## Contexto

O OMS tem convenções bem estabelecidas, seguidas por todos os módulos atuais. Bruno descreveu o padrão na reunião: cada domínio é um módulo em `src/modules` com controller, service, repository, routes e schemas, e propôs que webhooks siga exatamente igual ([09:27] Bruno).

Um módulo novo pode adotar essas convenções por inteiro, adotá-las em parte com desvios justificados, ou introduzir abordagem própria. A escolha tem consequência direta no custo de integração e no custo de manutenção.

Os padrões em jogo, todos verificáveis no repositório:

- Estrutura de módulo em cinco arquivos, visível em [src/modules/orders](../../src/modules/orders) e replicada em customers, products e users.
- Hierarquia de erros baseada em `AppError`, com código de erro em maiúsculas, definida em [src/shared/errors/app-error.ts](../../src/shared/errors/app-error.ts) e [src/shared/errors/http-errors.ts](../../src/shared/errors/http-errors.ts).
- Middleware de erro centralizado em [src/middlewares/error.middleware.ts](../../src/middlewares/error.middleware.ts).
- Validação de entrada com Zod pelo middleware em [src/middlewares/validate.middleware.ts](../../src/middlewares/validate.middleware.ts).
- Logger Pino em [src/shared/logger/index.ts](../../src/shared/logger/index.ts).
- Autenticação e autorização por papel em [src/middlewares/auth.middleware.ts](../../src/middlewares/auth.middleware.ts).
- Envelope de resposta paginada em [src/shared/http/response.ts](../../src/shared/http/response.ts).
- Identificadores em UUID no schema, visível em [prisma/schema.prisma](../../prisma/schema.prisma).

## Decisão

O módulo de webhooks **adota integralmente os padrões existentes**, sem quebrar convenção e sem introduzir dependência nova. A formulação da reunião foi reuso máximo do que já existe ([09:30] Larissa).

Na prática:

- **Estrutura de módulo.** Módulo novo em `src/modules/webhooks`, com controller, service, repository, routes e schemas, no mesmo formato dos demais ([09:27] Bruno). A lógica de processamento do worker fica em arquivo próprio dentro do módulo, e a entry-point do processo fica separada ([09:28] Bruno).
- **Erros.** As classes novas estendem `AppError` e as derivadas existentes, no mesmo molde de `InsufficientStockError` e `InvalidStatusTransitionError`, com códigos no prefixo `WEBHOOK_` ([09:28] Bruno, [09:29] Larissa). Os exemplos citados na reunião foram `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL` e `WEBHOOK_SECRET_REQUIRED` ([09:28] Bruno).
- **Middleware de erro.** Nenhuma alteração. O middleware central já trata `AppError`, erros de validação Zod e erros conhecidos do Prisma, então captura os erros do módulo sem precisar mudar nada ([09:29] Bruno).
- **Logger.** Nenhuma biblioteca nova. O Pino já está no projeto inteiro e é reusado como está ([09:29] Bruno). A única alteração é incluir o campo da secret na lista de redação, para que ela nunca apareça em log.
- **Autorização.** O middleware de papel existente é reaproveitado para restringir o replay a ADMIN ([09:36] Larissa).
- **Identificadores.** UUID, seguindo o padrão do resto do projeto, onde tudo é uuid ([09:51] Larissa).
- **Cliente de banco no worker.** Instância separada, criada pela mesma função de criação usada pela API, apontando para o mesmo banco e a mesma URL de conexão ([09:30] Bruno).

## Alternativas Consideradas

### Compartilhar a mesma instância do cliente Prisma entre API e worker

Diego levantou a pergunta de forma direta, ao tratar de infraestrutura compartilhada: o pool de conexão do Prisma já existe, então o worker abre o mesmo cliente ou um separado ([09:29] Diego).

Foi descartada porque o cliente Prisma é por processo. Como o worker roda em outro processo Node, precisa de instância nova, ainda que no mesmo banco e com a mesma URL de conexão ([09:30] Bruno).

**Trade-off que motivou o descarte:** compartilhar o cliente pareceria economizar conexões, mas é tecnicamente inviável entre processos distintos, e a instância separada tem como custo real apenas um pool de conexões adicional.

### Introduzir uma biblioteca de fila e agendamento de jobs para o worker

Alternativa plausível, não discutida na reunião, mas que costuma aparecer quando se implementa reenvio com backoff e fila de falhas: usar uma biblioteca pronta de jobs, que já traz agendamento, tentativas, backoff e fila de falhas permanentes.

Ela é incompatível com as decisões já tomadas. As bibliotecas dessa categoria no ecossistema Node dependem de Redis como backend, o que contradiz diretamente [ADR-001](ADR-001-padrao-outbox-no-mysql.md), que recusou infraestrutura nova ([09:07] Diego). Além disso, o estado do evento deixaria de viver em tabela relacional consultável e passaria a viver dentro da estrutura interna da biblioteca, o que dificultaria diagnóstico e auditoria.

**Trade-off que motivou o descarte:** ganharia mecanismos prontos de reenvio e fila, ao custo de infraestrutura nova, dependência nova e perda de visibilidade do estado, tudo aquilo que a reunião decidiu evitar.

### Criar hierarquia de erros própria do módulo, fora de `AppError`

Alternativa plausível, também não discutida: definir uma hierarquia paralela para ter controle total sobre a serialização das respostas de erro do módulo.

Não se sustenta diante do código existente. O middleware central já serializa qualquer `AppError` com status, código, mensagem e detalhes, formato usado por todos os demais módulos. Uma hierarquia própria criaria dois formatos de erro dentro da mesma API.

**Trade-off que motivou o descarte:** ganharia liberdade de formato, ao custo de inconsistência para quem consome a API e de alteração no middleware central, que hoje não precisa mudar.

## Consequências

### Positivas

- Custo de integração próximo de zero. Quem conhece o módulo de pedidos consegue trabalhar no de webhooks imediatamente.
- O middleware de erro não muda, e os erros do módulo passam a ser tratados automaticamente pelo caminho já existente.
- Nenhuma dependência npm nova, o que mantém build, inventário de segurança e superfície de manutenção inalterados.
- Os testes do módulo seguem o formato dos existentes, com a mesma infraestrutura de teste já configurada no projeto.
- O prefixo `WEBHOOK_` nos códigos de erro cria um espaço de nomes identificável, útil para filtrar log e diagnosticar ([09:29] Larissa).
- Melhorias futuras no logger, na hierarquia de erros ou no middleware se propagam para o módulo sem trabalho adicional.

### Negativas e trade-offs explícitos

- **O módulo herda as limitações dos padrões atuais.** Abre-se mão de introduzir abordagem diferente das já adotadas, em troca de consistência.
- **Acoplamento entre o serviço de pedidos e o módulo de webhooks.** É o único desvio do isolamento entre módulos, e existe porque a gravação precisa acontecer dentro da transação. O acoplamento é contido por usar uma função que recebe o cliente transacional, sem injetar o repositório inteiro ([09:41] Bruno, [09:41] Diego). Se um dia os módulos forem separados em serviços distintos, este é o ponto a refatorar.
- **O módulo tem um arquivo a mais que os demais**, para a lógica de processamento do worker, sem equivalente nos outros módulos ([09:28] Bruno). É um desvio pequeno, mas quebra a simetria exata de cinco arquivos.
- **A ausência de biblioteca de jobs significa implementar à mão** agendamento, contagem de tentativas e movimentação para a fila de falhas, código que uma biblioteca entregaria pronto e testado.
- **A lista de redação do logger precisa ser mantida manualmente.** Um campo novo com material sensível que não seja adicionado à lista vaza em log, e nada no projeto detecta isso automaticamente.

## Referências

**Transcrição:** [09:27] Bruno, [09:28] Bruno, [09:28] Diego, [09:29] Bruno, [09:29] Larissa, [09:29] Diego, [09:30] Bruno, [09:30] Larissa, [09:36] Larissa, [09:41] Bruno, [09:51] Larissa, [09:48] Larissa

**Código existente:**

| Arquivo | Como o módulo de webhooks o reutiliza |
| --- | --- |
| [src/modules/orders/order.routes.ts](../../src/modules/orders/order.routes.ts) | Molde do router do módulo, incluindo a aplicação do middleware de autenticação e a validação por rota |
| [src/modules/orders/order.controller.ts](../../src/modules/orders/order.controller.ts) | Molde do controller, com camada fina que delega ao service e encaminha erro ao middleware |
| [src/modules/orders/order.schemas.ts](../../src/modules/orders/order.schemas.ts) | Molde dos schemas Zod, incluindo validação de identificador, uso do enum de status e limites de paginação |
| [src/shared/errors/app-error.ts](../../src/shared/errors/app-error.ts) | Classe base de todos os erros do módulo, com status, código e detalhes |
| [src/shared/errors/http-errors.ts](../../src/shared/errors/http-errors.ts) | Classes derivadas reusadas, e molde concreto de erro de domínio nas classes de estoque insuficiente e transição inválida |
| [src/middlewares/error.middleware.ts](../../src/middlewares/error.middleware.ts) | Trata os erros do módulo sem nenhuma alteração |
| [src/middlewares/auth.middleware.ts](../../src/middlewares/auth.middleware.ts) | Autenticação em todas as rotas do módulo e autorização por papel no replay |
| [src/middlewares/validate.middleware.ts](../../src/middlewares/validate.middleware.ts) | Validação de corpo, parâmetros e query com os schemas do módulo |
| [src/shared/logger/index.ts](../../src/shared/logger/index.ts) | Logger reusado pela API e pelo worker, com extensão da lista de redação |
| [src/shared/http/response.ts](../../src/shared/http/response.ts) | Envelope de paginação nas listagens do módulo |
| [src/routes/index.ts](../../src/routes/index.ts) | Registro do router do módulo, no mesmo padrão de composição dos demais |
| [src/app.ts](../../src/app.ts) | Montagem da cadeia repository, service e controller, na mesma injeção manual existente |
| [src/server.ts](../../src/server.ts) | Molde da entry-point do worker, com bootstrap, tratamento de sinais e desconexão do banco |
| [src/config/database.ts](../../src/config/database.ts) | Função de criação do cliente Prisma, usada pelo worker para criar instância própria |
| [prisma/schema.prisma](../../prisma/schema.prisma) | Convenções de modelagem seguidas pelos modelos novos, incluindo UUID, índices e nome de tabela |
