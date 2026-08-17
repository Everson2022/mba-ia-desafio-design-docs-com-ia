### PRD: Order Management System, Sistema de Webhooks de Notificação de Pedidos

Versão: 1.0
Data: 2026-08-16
Responsável: Marcos, Product Manager

---

### Resumo

O Sistema de Webhooks de Notificação de Pedidos adiciona ao OMS um mecanismo de notificação HTTP de saída. Quando o status de um pedido muda, o sistema grava o evento na mesma transação SQL que já atualiza o pedido, grava o histórico e movimenta o estoque, e um worker em processo separado entrega esse evento nos endpoints que o cliente cadastrou, com assinatura HMAC-SHA256, reenvio automático em caso de falha e Dead Letter para o que não foi entregue.

A demanda veio de pedido formal de três clientes B2B, Atlas Comercial, MaxDistribuição e Nova Cargo, que hoje precisam consultar periodicamente a API de pedidos para descobrir se algo mudou ([09:00] Marcos). A Atlas sinalizou possibilidade de migrar para o concorrente caso a entrega não saia até o fim do trimestre ([09:00] Marcos).

A solução foi desenhada para não introduzir nenhuma infraestrutura nova. O padrão Outbox é implementado sobre o MySQL que já está em produção, e o módulo segue o mesmo formato dos módulos existentes do projeto ([09:07] Diego, [09:27] Bruno, [09:30] Larissa).

---

### Contexto e problema

Público-alvo
- Equipes técnicas dos clientes B2B que integram sistemas próprios com o OMS, hoje representadas por Atlas Comercial, MaxDistribuição e Nova Cargo ([09:00] Marcos)
- Usuários autenticados do OMS que representam esses clientes e operam o cadastro dos endpoints pela API ([09:32] Marcos)
- Administradores internos com role ADMIN, responsáveis por reprocessar eventos que falharam em definitivo ([09:36] Sofia)

Cenários de uso chave
- O cliente cadastra um endpoint e escolhe quais status de pedido quer receber, por exemplo apenas SHIPPED e DELIVERED ([09:31] Marcos, [09:33] Marcos)
- Um pedido muda de status e o cliente recebe a notificação assinada em menos de 10 segundos, sem precisar consultar a API ([09:02] Marcos, [09:06] Diego)
- O sistema do cliente fica indisponível por algumas horas e recebe o evento em uma tentativa posterior, sem intervenção manual de nenhum dos lados ([09:15] Diego, [09:17] Diego)
- O cliente descobre que a secret dele vazou e solicita uma nova pela API, mantendo a integração funcionando durante a migração ([09:21] Sofia, [09:22] Diego)
- O cliente consulta o histórico de entregas para conferir o que foi enviado, com resultado, payload, resposta e tempo de resposta ([09:34] Marcos)
- Um administrador reprocessa manualmente um evento que esgotou as tentativas e ficou na Dead Letter ([09:35] Diego)

Onde essa feature será implantada
- Sistema existente. O OMS em produção, em Node.js e TypeScript, com Express e MySQL via Prisma
- Novo módulo dentro de `src/modules`, com controller, service, repository, routes e schemas, no mesmo formato dos módulos atuais ([09:27] Bruno)
- Nova entry-point de processo para o worker, ao lado da entry-point de API que já existe em [src/server.ts](../src/server.ts), acionada por script próprio ([09:11] Larissa, [09:28] Bruno)
- Novas estruturas de persistência criadas por migration no MySQL existente. Sem banco novo, sem broker, sem infraestrutura adicional ([09:07] Larissa, [09:07] Diego)
- Entrega exclusivamente de backend, composta por endpoints REST e pelo worker. Nenhuma interface visual faz parte desta fase ([09:40] Larissa)

Problemas priorizados
- Os clientes B2B consultam periodicamente o endpoint de listagem de pedidos para descobrir mudanças de status, o que torna a integração deles lenta e cara e atrasa a percepção da mudança. Prioridade alta ([09:00] Marcos)
- Existe risco comercial concreto de perda de conta, com a Atlas Comercial sinalizando possibilidade de migrar para o concorrente caso a entrega não ocorra até o fim do trimestre. Prioridade alta ([09:00] Marcos)
- A plataforma não possui nenhum mecanismo de notificação externa, evento, fila ou webhook. O ciclo de vida do pedido é fechado dentro do método de mudança de status em [src/modules/orders/order.service.ts](../src/modules/orders/order.service.ts), que hoje atualiza o pedido, grava o histórico e movimenta o estoque dentro de uma transação, sem nenhum ponto de extensão. Prioridade alta ([09:04] Bruno)
- Notificar de forma síncrona dentro da transação é inviável, porque um cliente lento travaria a mudança de status de outros pedidos e um cliente fora do ar levantaria a questão sem resposta aceitável de reverter a mudança de status. Prioridade alta ([09:04] Bruno, [09:06] Diego)
- Os eventos carregam dados de pedidos para endpoints fora da infraestrutura da empresa, e hoje não existe forma de o cliente verificar que a requisição veio realmente da plataforma nem que o payload não foi adulterado no caminho. Prioridade alta ([09:19] Sofia)

---

### Objetivos e métricas

| Objetivo | Métrica | Meta |
| --- | --- | --- |
| Notificar o cliente da mudança de status sem que ele precise consultar a API | Tempo entre o commit da transação de mudança de status e a entrega no endpoint do cliente | Abaixo de 10 segundos ([09:02] Marcos) |
| Substituir o polling dos clientes B2B que solicitaram a feature | Clientes com endpoint cadastrado e recebendo notificação em produção | 3 de 3, Atlas Comercial, MaxDistribuição e Nova Cargo ([09:00] Marcos) |
| Não perder evento quando o cliente está indisponível | Destino final de cada evento gerado | 100% dos eventos terminam entregues ou registrados na Dead Letter, nenhum evento desaparece silenciosamente ([09:15] Diego, [09:18] Diego) |
| Não acoplar a operação interna de mudança de status à disponibilidade de terceiros | Chamadas HTTP externas executadas dentro da transação de mudança de status | Zero ([09:04] Bruno, [09:06] Diego) |
| Cobrir janelas de indisponibilidade do cliente sem intervenção manual | Janela total coberta pelas tentativas automáticas de reenvio | Cerca de 15 horas, com 5 tentativas em 1m, 5m, 30m, 2h e 12h ([09:17] Diego) |
| Permitir que o cliente valide origem e integridade do evento recebido | Requisições de webhook enviadas com assinatura no header | 100% assinadas com HMAC-SHA256, com secret exclusiva por endpoint ([09:20] Sofia, [09:21] Sofia) |
| Entregar dentro da janela comercial acordada com o cliente em risco | Prazo de entrega da feature em produção | Três sprints, com a revisão de segurança incluída ([09:46] Larissa, [09:46] Sofia) |

---

### Escopo

Incluso
- Cadastro, edição, remoção e listagem de endpoints de webhook por cliente ([09:31] Marcos, [09:33] Bruno)
- Filtro de eventos por endpoint, com o cliente escolhendo quais status quer ouvir, aplicado no momento da gravação do evento ([09:33] Marcos, [09:34] Bruno)
- Geração da secret pelo sistema no cadastro, devolvida ao cliente na resposta de criação ([09:31] Marcos)
- Rotação de secret pela API, com a anterior válida em paralelo por 24 horas ([09:21] Sofia)
- Gravação do evento em tabela de outbox, dentro da mesma transação que muda o status do pedido ([09:06] Diego, [09:40] Bruno)
- Worker em processo separado, lendo a outbox em polling de 2 segundos ([09:09] Diego, [09:11] Diego)
- Entrega por HTTP POST assinado com HMAC-SHA256, com headers de identificação do evento, assinatura, timestamp de envio e identificação do endpoint ([09:20] Sofia, [09:44] Diego, [09:44] Sofia)
- Timeout de 10 segundos por chamada, tratado como falha sujeita a reenvio ([09:42] Diego)
- Reenvio automático com backoff exponencial, 5 tentativas em 1m, 5m, 30m, 2h e 12h ([09:15] Diego, [09:17] Diego)
- Dead Letter em estrutura separada, com payload, motivo da falha e timestamp ([09:18] Diego)
- Endpoint administrativo de replay manual de Dead Letter, restrito a role ADMIN e com registro de quem executou ([09:18] Diego, [09:35] Diego, [09:36] Sofia)
- Consulta do histórico de entregas por endpoint, com resultado, payload, resposta e tempo de resposta ([09:34] Marcos)
- Payload congelado no momento da gravação, refletindo o estado do pedido no instante da transição ([09:52] Larissa, [09:52] Diego)
- Payload JSON enxuto, com identificação do evento, tipo, timestamp, dados básicos do pedido e a transição de status, sem os itens do pedido ([09:43] Diego)
- Validação de URL, aceitando apenas https e recusando http ([09:23] Sofia)
- Limite de 64KB por payload, com erro caso ultrapasse ([09:23] Sofia, [09:24] Diego)
- Reuso dos padrões existentes do projeto, incluindo a hierarquia de erros baseada em `AppError`, o prefixo de código de erro, os schemas de validação, o logger Pino, o middleware de erro centralizado e o middleware de autorização ([09:28] Bruno, [09:29] Bruno, [09:30] Larissa, [09:36] Larissa)
- Identificadores em UUID, seguindo o padrão do resto do projeto ([09:51] Larissa)

Fora de escopo
- Aviso por email ao cliente quando o webhook dele está falhando. Descartado nesta fase, para ser reavaliado depois de medir o impacto em produção ([09:37] Marcos, [09:37] Larissa, [09:38] Marcos)
- Dashboard ou painel visual para o cliente acompanhar os webhooks. Descartado, é projeto separado do time de frontend, e esta fase entrega apenas endpoints ([09:39] Marcos, [09:40] Larissa)
- Controle de vazão na saída, limitando o volume de chamadas disparadas para um mesmo cliente. Adiado, com a decisão de observar o comportamento em produção antes de implementar ([09:38] Diego, [09:39] Diego, [09:39] Larissa)
- Webhooks de entrada, do cliente para a plataforma. O fluxo é exclusivamente de saída ([09:02] Sofia, [09:02] Marcos, [09:03] Sofia)
- Arquivamento das linhas já entregues da outbox. Reconhecido como necessário e declarado fora do escopo desta feature ([09:08] Diego)
- Múltiplos workers em paralelo e garantia de ordenação global. Adiado, com a fase atual mantendo um único worker e ordenação por pedido, registrada como limitação conhecida ([09:12] Diego, [09:13] Diego, [09:13] Larissa)
- Restrição de role no CRUD de configuração de webhook. Adiado, com qualquer role autenticada operando o CRUD nesta fase e endurecimento previsto para o futuro ([09:36] Marcos, [09:37] Sofia)
- Garantia de entrega exactly-once. A plataforma oferece at-least-once, e a deduplicação é responsabilidade do cliente ([09:24] Diego, [09:25] Diego)

---

### Requisitos funcionais

#### RF-001 Cadastrar endpoint de webhook
O cliente registra uma URL que vai receber as notificações, informando quais status de pedido quer ouvir ([09:31] Marcos).

**Fluxo principal**
- Usuário autenticado envia a requisição de criação com o identificador do cliente, a URL de destino e a lista de status
- O sistema valida a URL e a lista de status
- O sistema gera a secret exclusiva daquele endpoint
- O sistema persiste o cadastro como ativo
- O sistema devolve o cadastro com a secret no corpo da resposta, única ocasião em que ela é exposta ([09:21] Sofia)

**Fluxos alternativos e exceções**
- O identificador do cliente vem no corpo ou no caminho da requisição, não é inferido do token, porque o token é de usuário operador e não de cliente ([09:32] Bruno, [09:32] Larissa)
- URL com http é recusada, apenas https é aceita ([09:23] Sofia)
- Qualquer role autenticada pode operar o cadastro nesta fase, sem restrição adicional ([09:37] Sofia)

**Erros previstos**
- URL inválida ou fora de https
- Lista de status vazia ou com valor inexistente na máquina de estados de pedidos
- Cliente inexistente

**Prioridade:** alta

---

#### RF-002 Editar endpoint de webhook
O cliente altera a URL, a lista de status ouvidos ou o estado ativo de um cadastro existente ([09:33] Bruno).

**Fluxo principal**
- Usuário autenticado envia a alteração parcial referenciando o endpoint
- O sistema valida os campos presentes na requisição
- O sistema persiste a alteração
- O sistema devolve o cadastro atualizado, sem expor a secret

**Fluxos alternativos e exceções**
- Campos não enviados permanecem inalterados
- Endpoint marcado como inativo deixa de gerar eventos novos ([09:21] Bruno)

**Erros previstos**
- Endpoint inexistente
- URL inválida ou fora de https
- Lista de status inválida

**Prioridade:** alta

---

#### RF-003 Remover endpoint de webhook
O cliente exclui um cadastro que não quer mais utilizar ([09:33] Bruno).

**Fluxo principal**
- Usuário autenticado solicita a remoção do endpoint
- O sistema remove o cadastro
- O sistema responde sem corpo

**Fluxos alternativos e exceções**
- O tratamento dos eventos já gravados na outbox para um endpoint removido não foi discutido na reunião e fica registrado como questão em aberto para o RFC

**Erros previstos**
- Endpoint inexistente

**Prioridade:** alta

---

#### RF-004 Listar endpoints de webhook de um cliente
O cliente consulta os cadastros que possui ([09:33] Bruno).

**Fluxo principal**
- Usuário autenticado consulta a lista filtrando pelo cliente
- O sistema devolve os cadastros paginados, sem a secret

**Fluxos alternativos e exceções**
- Cliente sem nenhum cadastro recebe lista vazia, não erro
- A paginação segue o helper já utilizado pelos demais módulos, em [src/shared/http/response.ts](../src/shared/http/response.ts)

**Erros previstos**
- Cliente inexistente

**Prioridade:** alta

---

#### RF-005 Rotacionar a secret de um endpoint
O cliente solicita uma nova secret sem interromper a integração em produção ([09:21] Sofia).

**Fluxo principal**
- Usuário autenticado solicita a rotação da secret do endpoint
- O sistema gera a nova secret
- O sistema mantém a secret anterior válida em paralelo por 24 horas
- O sistema devolve a nova secret no corpo da resposta

**Fluxos alternativos e exceções**
- Durante as 24 horas, o cliente pode validar a assinatura com qualquer uma das duas secrets, o que lhe dá tempo de migrar os sistemas dele ([09:21] Sofia)
- Passadas as 24 horas, a secret anterior deixa de valer
- O caso motivador registrado na reunião é o de cliente que vazou a secret no log da própria aplicação ([09:22] Diego)

**Erros previstos**
- Endpoint inexistente

**Prioridade:** alta

---

#### RF-006 Gravar o evento na outbox dentro da transação de mudança de status
Quando o status de um pedido muda, o evento correspondente é gravado na outbox dentro da mesma transação que já atualiza o pedido, grava o histórico e movimenta o estoque ([09:06] Diego, [09:40] Bruno).

**Fluxo principal**
- A mudança de status executa a transação existente em [src/modules/orders/order.service.ts](../src/modules/orders/order.service.ts)
- Dentro da transação, o sistema consulta os endpoints ativos do cliente que ouvem o status de destino
- Para cada endpoint encontrado, o sistema grava uma linha na outbox com o payload já renderizado
- A transação commita a mudança de status e os eventos em conjunto

**Fluxos alternativos e exceções**
- Se nenhum endpoint ativo do cliente ouve aquele status, nada é gravado, porque o filtro é aplicado na gravação e não no envio ([09:34] Marcos, [09:34] Bruno, [09:34] Diego)
- O payload é congelado no momento da gravação, de forma que alteração posterior no pedido não altera o evento já registrado ([09:52] Larissa, [09:52] Diego)
- Se a gravação falhar, a transação inteira sofre rollback, inclusive a mudança de status, porque não pode existir caso de status alterado sem evento correspondente ([09:40] Bruno, [09:41] Diego)
- A integração é feita por uma função que recebe o cliente transacional em uso, sem injetar o repositório de webhooks dentro do serviço de pedidos ([09:41] Bruno, [09:41] Diego)

**Erros previstos**
- Falha de persistência na gravação do evento, que propaga o erro e cancela a mudança de status

**Prioridade:** alta

---

#### RF-007 Despachar o evento para o endpoint do cliente
Um worker em processo separado lê os eventos pendentes e executa a entrega HTTP assinada ([09:09] Diego, [09:11] Diego).

**Fluxo principal**
- O worker consulta, a cada 2 segundos, os eventos pendentes mais antigos, em lote pequeno
- Para cada evento, o worker calcula a assinatura HMAC-SHA256 do corpo usando a secret do endpoint de destino
- O worker envia a requisição com os headers de identificação do evento, assinatura, timestamp de envio e identificação do endpoint
- Resposta de sucesso marca o evento como entregue e registra a tentativa no histórico de entregas

**Fluxos alternativos e exceções**
- O worker roda como processo separado da API, com instância própria do cliente Prisma, no mesmo banco e na mesma stack ([09:11] Diego, [09:29] Diego, [09:30] Bruno)
- A leitura da outbox se apoia em índice por estado e por data de criação ([09:08] Diego)
- A ordem de entrega segue a ordem de criação dos eventos, o que garante ordenação por pedido enquanto houver um único worker ([09:12] Diego)
- Resposta que não chega em 10 segundos é tratada como falha ([09:42] Diego)
- A entrega é at-least-once, e o cliente pode receber o mesmo evento mais de uma vez, deduplicando pelo identificador do evento ([09:24] Diego, [09:25] Diego)

**Erros previstos**
- Destino indisponível
- Resposta de erro do destino
- Estouro do timeout de 10 segundos

**Prioridade:** alta

---

#### RF-008 Reenviar automaticamente eventos que falharam
Evento que falha é reenviado com intervalos crescentes até o limite de tentativas ([09:15] Diego, [09:17] Diego).

**Fluxo principal**
- Após uma falha, o sistema registra a tentativa e o motivo
- O sistema agenda a próxima tentativa segundo a progressão de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas
- O worker só reprocessa o evento quando o horário agendado chega
- O ciclo se encerra no sucesso ou na quinta tentativa

**Fluxos alternativos e exceções**
- Sucesso em qualquer tentativa encerra o ciclo imediatamente
- A janela total de cerca de 15 horas foi dimensionada para cobrir indisponibilidade planejada do cliente, com base em caso real de cliente com duas horas de manutenção ([09:16] Diego, [09:17] Marcos)

**Erros previstos**
- Cada falha registra o motivo, para consulta posterior no histórico de entregas

**Prioridade:** alta

---

#### RF-009 Mover para Dead Letter o evento que esgotou as tentativas
Depois da quinta falha, o evento sai do fluxo automático e passa a ser registrado em estrutura separada ([09:18] Diego).

**Fluxo principal**
- Esgotadas as 5 tentativas, o sistema grava o evento na Dead Letter com o payload, o motivo da falha e o timestamp
- O sistema encerra o processamento automático daquele evento

**Fluxos alternativos e exceções**
- A estrutura separada foi escolhida para manter limpa a leitura da outbox principal e servir de evidência para diagnóstico e reprocessamento ([09:18] Diego)
- Nenhum aviso automático é enviado ao cliente nesta fase ([09:37] Larissa)

**Erros previstos**
- Nenhum erro exposto ao cliente, já que o registro na Dead Letter é o próprio desfecho do evento

**Prioridade:** alta

---

#### RF-010 Reprocessar manualmente um evento em Dead Letter
Um administrador recoloca no fluxo um evento que havia falhado em definitivo ([09:18] Diego, [09:35] Diego).

**Fluxo principal**
- Usuário com role ADMIN solicita o replay do evento
- O sistema recoloca o evento na outbox como pendente
- O worker volta a processar o evento no ciclo seguinte
- A operação fica registrada em log com a identificação de quem executou ([09:36] Sofia)

**Fluxos alternativos e exceções**
- Role diferente de ADMIN é recusada, reaproveitando o middleware de autorização existente em [src/middlewares/auth.middleware.ts](../src/middlewares/auth.middleware.ts) ([09:36] Larissa)
- A justificativa da restrição é que mexer em fila de entrega de notificação não é operação de operador comum ([09:36] Sofia)

**Erros previstos**
- Evento inexistente na Dead Letter
- Permissão insuficiente

**Prioridade:** alta

---

#### RF-011 Consultar o histórico de entregas de um endpoint
O cliente consulta as entregas recentes feitas para o endpoint dele ([09:34] Marcos).

**Fluxo principal**
- Usuário autenticado consulta as entregas de um endpoint
- O sistema devolve a lista com resultado de sucesso ou falha, payload enviado, resposta recebida e tempo de resposta

**Fluxos alternativos e exceções**
- A referência dada na reunião foi da ordem de 100 registros recentes, o que se resolve com paginação no mesmo padrão dos demais módulos ([09:34] Marcos)
- Endpoint sem entregas devolve lista vazia

**Erros previstos**
- Endpoint inexistente

**Prioridade:** media

---

### Requisitos não funcionais

Performance
- Latência entre o commit da transação de mudança de status e a entrega no endpoint do cliente abaixo de 10 segundos ([09:02] Marcos)
- Latência mínima de 2 segundos no pior caso, decorrente do ciclo de polling do worker, aceita explicitamente pelo time ([09:09] Diego, [09:10] Larissa)
- Timeout de 10 segundos por chamada HTTP de entrega ([09:42] Diego)
- Leitura da outbox em lote pequeno, apoiada em índice por estado e por data de criação, para que o acúmulo de eventos não degrade o worker ([09:08] Diego)

Disponibilidade
- Ciclos de vida da API e do worker são independentes, e o reinício de um não afeta o outro ([09:11] Diego, [09:11] Larissa)
- Indisponibilidade do endpoint do cliente não bloqueia nem reverte a mudança de status no OMS ([09:04] Bruno)
- Tolerância a indisponibilidade do cliente de cerca de 15 horas sem intervenção manual, coberta pelas tentativas de reenvio ([09:17] Diego)

Segurança e autorização
- Toda requisição de webhook é assinada com HMAC-SHA256 sobre o corpo do request, permitindo ao cliente verificar origem e integridade ([09:20] Sofia)
- Cada endpoint cadastrado possui secret própria, nunca uma secret global da plataforma, de forma que o vazamento de uma não comprometa as demais ([09:21] Sofia)
- A secret é rotacionável pela API, com a anterior válida em paralelo por 24 horas ([09:21] Sofia)
- URL de destino obrigatoriamente em https, com http recusado na validação de schema ([09:23] Sofia)
- Payload limitado a 64KB, com erro caso ultrapasse, sem truncamento ([09:23] Sofia, [09:24] Diego, [09:24] Larissa)
- O replay de Dead Letter exige role ADMIN e registra quem executou, para auditoria ([09:36] Sofia, [09:36] Larissa)
- Autenticação e autorização reaproveitam os middlewares existentes em [src/middlewares/auth.middleware.ts](../src/middlewares/auth.middleware.ts), sem middleware novo ([09:36] Larissa)
- A secret não pode aparecer em log. O logger Pino do projeto já possui lista de redação configurada em [src/shared/logger/index.ts](../src/shared/logger/index.ts), cobrindo hoje senha, token e header de autorização, e o campo da secret precisa ser incluído nessa lista. Item derivado do incidente de vazamento de secret em log do cliente citado na reunião ([09:22] Diego)

Observabilidade
- Logs estruturados com Pino, reaproveitando o logger já em uso em todo o projeto, sem introduzir biblioteca nova ([09:29] Bruno)
- Registro em log das operações do ciclo de vida do evento, incluindo o replay administrativo com identificação do executor ([09:36] Sofia)
- Erros do módulo tratados pelo middleware de erro centralizado já existente, que reconhece a hierarquia `AppError`, erros de validação e erros conhecidos do Prisma, em [src/middlewares/error.middleware.ts](../src/middlewares/error.middleware.ts) ([09:29] Bruno)

Confiabilidade e integridade de dados
- A gravação do evento acontece dentro da mesma transação SQL que muda o status, grava o histórico e movimenta o estoque. Commit confirma os dois lados e rollback cancela os dois lados ([09:06] Diego, [09:40] Bruno, [09:41] Diego)
- Garantia de entrega at-least-once, com identificador único por evento enviado no header para deduplicação do lado do cliente ([09:24] Diego, [09:25] Diego)
- Exactly-once não é oferecido, por exigir coordenação dos dois lados e complexidade desproporcional ([09:25] Diego)
- Payload congelado no momento da gravação, refletindo o estado do pedido no instante da transição e não no instante do envio ([09:52] Larissa, [09:52] Diego)
- Ordenação de entrega garantida por pedido, na ordem de criação dos eventos, enquanto houver um único worker. Não há garantia de ordenação global, e isso fica registrado como limitação conhecida ([09:12] Diego, [09:13] Larissa, [09:14] Marcos)
- Nenhum evento é descartado silenciosamente. O evento que esgota as tentativas permanece persistido na Dead Letter com payload e motivo da falha ([09:18] Diego)

Compatibilidade e portabilidade
- API REST em JSON, publicada sob o prefixo de versão já existente em [src/app.ts](../src/app.ts), seguindo o padrão dos módulos atuais
- Mesma stack e mesmo banco. O worker conecta na mesma URL de conexão da API, com instância própria do cliente Prisma, porque o cliente é por processo ([09:29] Diego, [09:30] Bruno)
- Novas tabelas criadas por migration no MySQL existente, sem novo banco e sem broker ([09:07] Larissa, [09:07] Diego)
- Identificadores em UUID, seguindo o padrão do resto do projeto ([09:51] Larissa)
- Payload em JSON, com timestamp em ISO 8601 ([09:43] Diego)
- Nova entry-point de processo, ao lado da entry-point de API existente em [src/server.ts](../src/server.ts), acionada por script próprio ([09:11] Larissa)

Compliance
- A reunião não levantou exigência regulatória, contratual ou legal específica, e nenhuma norma é registrada aqui por não haver origem identificável
- Existe exigência de trilha de auditoria da operação administrativa de replay, com identificação do executor ([09:36] Sofia)
- A revisão de segurança é obrigatória antes do deploy, com no mínimo dois dias úteis reservados ([09:46] Sofia)

Acessibilidade no frontend consumidor
- Não se aplica nesta fase. A entrega é exclusivamente de backend, composta por endpoints REST e worker, e dashboard ou painel visual para o cliente foi declarado fora de escopo, pertencendo a projeto separado do time de frontend ([09:39] Marcos, [09:40] Larissa)

---

### Arquitetura e abordagem

Abordagem
- Padrão Outbox sobre o MySQL que já está em produção, com entrega assíncrona executada por um worker em processo separado que lê a tabela em polling. O evento é gravado na mesma transação SQL que muda o status do pedido, de forma que commit e rollback carreguem os dois lados juntos ([09:06] Diego)
- Disparo síncrono dentro da transação foi recusado por acoplar a operação interna à disponibilidade do cliente ([09:04] Bruno, [09:06] Diego)
- Infraestrutura nova foi recusada por ser desproporcional ao tamanho do time ([09:07] Larissa, [09:07] Diego)

Componentes
- Módulo de webhooks dentro de `src/modules`, com controller, service, repository, routes e schemas, no mesmo formato dos módulos existentes ([09:27] Bruno)
- Função de publicação do evento, que recebe o cliente transacional em uso e é chamada de dentro da transação do serviço de pedidos ([09:41] Bruno, [09:41] Diego)
- Outbox, com o evento e o estado do processamento, indexada por estado e por data de criação ([09:06] Diego, [09:08] Diego)
- Persistência da configuração dos endpoints, com URL, secret, cliente e estado ativo ([09:21] Bruno, [09:21] Sofia)
- Dead Letter, separada da outbox, com payload, motivo da falha e timestamp ([09:18] Diego)
- Registro de entregas, com resultado, payload, resposta e tempo de resposta ([09:34] Marcos)
- Worker, com entry-point própria acionada por script dedicado e lógica de processamento residindo dentro do módulo de webhooks ([09:11] Larissa, [09:28] Bruno)
- Instância própria do cliente Prisma no worker, no mesmo banco e mesma URL de conexão ([09:29] Diego, [09:30] Bruno)
- Assinatura HMAC-SHA256 do corpo do request, com secret por endpoint ([09:20] Sofia, [09:21] Sofia)

Integrações
- Serviço de pedidos, no método de mudança de status que hoje atualiza o pedido, grava o histórico e movimenta o estoque dentro de uma transação, em [src/modules/orders/order.service.ts](../src/modules/orders/order.service.ts). É a alteração crítica no código existente ([09:40] Bruno)
- Endpoints HTTP dos clientes B2B, integração externa de saída, sempre em https ([09:23] Sofia)
- Nenhuma integração externa adicional, sem broker e sem serviço novo ([09:07] Diego)
- Comunicação síncrona em REST JSON para configuração, consulta e replay, e comunicação assíncrona via outbox lida em polling para a entrega dos eventos ([09:09] Diego, [09:31] Marcos, [09:33] Bruno)

### Decisões e trade-offs

#### Decisão: padrão Outbox no MySQL existente, em vez de disparo síncrono ou de infraestrutura de fila externa
- **Justificativa:** gravar o evento na mesma transação da mudança de status elimina a possibilidade de inconsistência, porque se a transação commitou o evento existe e se ela sofreu rollback o evento some junto ([09:06] Diego). Disparo síncrono foi recusado porque um cliente lento travaria a mudança de status de outros pedidos e um cliente fora do ar levantaria a questão sem resposta aceitável de reverter a mudança de status ([09:04] Bruno). Infraestrutura nova foi recusada por ser desproporcional ao tamanho do time ([09:07] Larissa, [09:07] Diego)
- **Trade-off:** a tabela de outbox cresce continuamente e vai exigir política de arquivamento, declarada fora do escopo desta feature ([09:08] Diego)

#### Decisão: worker em processo separado, lendo a outbox em polling de 2 segundos
- **Justificativa:** o MySQL não possui mecanismo nativo de notificação de processo externo, e improvisar aviso por arquivo ou por chamada de endpoint seria uma solução artificial ([09:09] Diego). O worker precisa ser processo separado porque, dentro da API, um reinício derrubaria o processamento ([09:11] Diego)
- **Trade-off:** consulta constante ao banco mesmo com a outbox vazia, e latência mínima de 2 segundos no pior caso, aceita por caber com folga no limiar de 10 segundos ([09:10] Larissa, [09:10] Marcos)

#### Decisão: um único worker, com ordenação garantida apenas por pedido
- **Justificativa:** com um worker apenas, o processamento segue a ordem de criação dos eventos e o cliente recebe as transições de cada pedido na ordem correta. Os clientes nunca pediram ordenação global, apenas saber se cada pedido deles mudou ([09:12] Diego, [09:14] Marcos)
- **Trade-off:** não há escala horizontal do processamento nesta fase, e escalar no futuro exigiria particionar por pedido ou usar lock pessimista. A ordenação global fica registrada como limitação conhecida ([09:13] Diego, [09:13] Larissa)

#### Decisão: cinco tentativas de reenvio, com backoff de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas
- **Justificativa:** três tentativas encerrariam o evento em cerca de 30 minutos, insuficiente diante de caso real de cliente com indisponibilidade planejada de duas horas ([09:16] Diego). Reenvio indefinido foi recusado por deixar o evento pendurado para sempre caso o cliente desapareça ([09:15] Diego)
- **Trade-off:** um evento pode levar cerca de 15 horas até ser considerado falha permanente, e nesse cenário o cliente recebe a notificação com atraso grande ou não recebe. O time aceitou por considerar que um cliente fora do ar por 15 horas já enfrenta problema próprio grave ([09:17] Marcos)

#### Decisão: Dead Letter em estrutura separada, e não marcação de falha na própria outbox
- **Justificativa:** mantém limpa a leitura da outbox principal e preserva o evento como evidência para diagnóstico e reprocessamento ([09:18] Diego)
- **Trade-off:** mais um ponto de persistência para operar, e a recuperação depende de ação humana, já que não existe aviso automático ao cliente nem ao time quando um evento cai na Dead Letter ([09:37] Larissa)

#### Decisão: assinatura HMAC-SHA256 com secret exclusiva por endpoint e rotação com 24 horas de convivência
- **Justificativa:** HMAC-SHA256 é padrão de mercado e qualquer cliente sério possui biblioteca para verificar ([09:20] Sofia). Secret por endpoint evita que o vazamento de uma comprometa todas ([09:21] Sofia). A janela de 24 horas dá tempo de o cliente migrar os sistemas dele, cenário motivado por caso real de vazamento de secret em log do próprio cliente ([09:21] Sofia, [09:22] Diego)
- **Trade-off:** durante as 24 horas existem duas secrets válidas simultaneamente para o mesmo endpoint, o que amplia temporariamente a superfície de validação, e a plataforma assume a responsabilidade de gestão e guarda de uma secret por endpoint cadastrado

#### Decisão: entrega at-least-once com identificador único de evento, em vez de exactly-once
- **Justificativa:** garantir exactly-once exigiria coordenação dos dois lados e complexidade muito maior, enquanto at-least-once com identificador de evento resolve a quase totalidade dos casos e é o padrão adotado por plataformas de referência de mercado ([09:25] Diego)
- **Trade-off:** transfere ao cliente a responsabilidade de deduplicação, ponto levantado explicitamente na reunião e aceito com o compromisso de documentar a semântica de forma destacada no portal do desenvolvedor ([09:25] Sofia, [09:26] Marcos)

#### Decisão: reuso máximo dos padrões já existentes no projeto
- **Justificativa:** o módulo segue o formato dos demais e reaproveita a hierarquia de erros baseada em `AppError`, o padrão de códigos de erro com prefixo do módulo, os schemas de validação, o logger Pino e o middleware de erro centralizado, que já trata a hierarquia de erros da aplicação, erros de validação e erros conhecidos do Prisma sem precisar de alteração ([09:29] Bruno, [09:30] Larissa), verificável em [src/middlewares/error.middleware.ts](../src/middlewares/error.middleware.ts) e em [src/shared/errors/http-errors.ts](../src/shared/errors/http-errors.ts)
- **Trade-off:** o módulo herda as limitações dos padrões atuais e abre mão de introduzir abordagens diferentes das já adotadas no projeto, em troca de consistência e de custo de integração próximo de zero

#### Decisão: payload congelado no momento da gravação na outbox
- **Justificativa:** o evento passa a refletir o estado do pedido no instante exato da transição de status, enquanto renderizar na hora do envio produziria situações inconsistentes, já que o pedido pode ter mudado nesse intervalo ([09:52] Larissa, [09:52] Diego)
- **Trade-off:** o payload entregue pode estar defasado em relação ao estado atual do pedido, especialmente em reenvios que ocorrem horas depois. O efeito é atenuado pelo desenho enxuto do payload, que não carrega os itens do pedido e deixa o detalhe para consulta posterior do cliente na API de pedidos ([09:43] Diego)

#### Decisão: replay administrativo restrito a role ADMIN, com o restante do CRUD aberto a qualquer role autenticada
- **Justificativa:** mexer em fila de entrega de notificação não é operação de operador comum, e a operação precisa registrar quem executou para auditoria ([09:36] Sofia). A autorização reaproveita o middleware existente ([09:36] Larissa)
- **Trade-off:** nesta fase qualquer usuário autenticado pode criar ou alterar a URL de destino de um webhook, inclusive de clientes que não são os dele. A reunião reconheceu o ponto e decidiu endurecer mais adiante ([09:36] Marcos, [09:37] Sofia)

---

### Dependências

#### Técnica: novas estruturas de persistência no MySQL
A feature depende da criação, por migration no banco existente, dos quatro pontos de persistência do módulo: outbox, Dead Letter, configuração dos endpoints e registro de entregas. Nenhum banco novo, nenhum broker e nenhuma infraestrutura adicional fazem parte da entrega ([09:06] Diego, [09:07] Diego, [09:18] Diego, [09:21] Bruno, [09:34] Marcos).

#### Técnica: alteração no serviço de pedidos
A gravação do evento precisa acontecer dentro da transação que já existe no método de mudança de status, em [src/modules/orders/order.service.ts](../src/modules/orders/order.service.ts). É a única alteração no código existente e é bloqueante para a feature inteira, porque sem ela não existe evento a entregar. A integração é feita por uma função que recebe o cliente transacional em uso, sem injetar o repositório de webhooks dentro do serviço de pedidos ([09:40] Bruno, [09:41] Bruno, [09:41] Diego).

#### Técnica: nova entry-point de processo para o worker
O worker precisa de ponto de entrada próprio, ao lado da entry-point de API existente em [src/server.ts](../src/server.ts), e de script dedicado para execução, com a lógica de processamento residindo dentro do módulo de webhooks ([09:11] Larissa, [09:28] Bruno).

#### Técnica: conexão de banco própria do processo do worker
O worker abre instância própria do cliente Prisma, apontando para o mesmo banco e a mesma URL de conexão da API, porque o cliente é por processo ([09:29] Diego, [09:30] Bruno).

#### Técnica: extensão da lista de redação do logger
O campo da secret precisa ser incluído na lista de redação já configurada em [src/shared/logger/index.ts](../src/shared/logger/index.ts), que hoje cobre senha, token e header de autorização, para que a secret não apareça em log. Dependência derivada do incidente de vazamento de secret em log do cliente citado na reunião ([09:22] Diego).

#### Organizacional: revisão de segurança antes do deploy
Sofia precisa de no mínimo dois dias úteis reservados para revisar o código de segurança antes de a feature subir, com atenção específica à assinatura HMAC e à geração da secret. A revisão é bloqueante para o deploy em produção ([09:46] Sofia, [09:47] Larissa, [09:49] Sofia).

#### Organizacional: sessão de revisão do documento de design antes da implementação
Larissa assumiu o compromisso de abrir o documento de design da feature e marcar uma sessão com Bruno e Diego para revisão conjunta antes de o time começar a codar ([09:50] Larissa).

#### Organizacional: documentação de integração no portal do desenvolvedor
Marcos precisa publicar no portal do desenvolvedor a documentação de como integrar via API, com destaque explícito para a semântica at-least-once e para a necessidade de o cliente deduplicar pelo identificador do evento. Sem isso, o trade-off aceito na reunião se converte em problema de suporte ([09:26] Marcos, [09:40] Marcos).

#### Organizacional: confirmação de prazo com o cliente em risco
Marcos ficou responsável por confirmar o prazo acordado com a Atlas Comercial e por atualizar os três clientes sobre a entrega ([09:47] Marcos, [09:49] Marcos).

#### Externa: implementação do lado do cliente
A feature só entrega valor se o cliente cumprir a parte dele: expor endpoint em https, verificar a assinatura HMAC-SHA256 recebida e deduplicar eventos repetidos pelo identificador do evento. Nenhum desses pontos está sob controle da plataforma ([09:20] Sofia, [09:23] Sofia, [09:25] Diego).

---

### Riscos e mitigação

#### Cliente não implementa deduplicação e processa o mesmo evento mais de uma vez
- **Probabilidade:** alta
- **Impacto:** o cliente executa duas vezes a mesma ação de negócio a partir de uma notificação repetida. Risco levantado explicitamente na reunião ([09:25] Sofia)
- **Mitigação:**
  - Documentação destacada da semântica at-least-once no portal do desenvolvedor ([09:26] Marcos)
  - Identificador único enviado no header em todas as tentativas do mesmo evento, permitindo a deduplicação do lado do cliente ([09:25] Diego)
  - Acompanhamento próximo dos três clientes iniciais durante a integração ([09:00] Marcos)
- **Plano de contingência:** o registro de entregas permite reconstituir exatamente o que foi enviado e quando, dando base para o cliente corrigir retroativamente ([09:34] Marcos)

#### Evento cai na Dead Letter e ninguém percebe
- **Probabilidade:** alta
- **Impacto:** a notificação nunca chega ao cliente e a descoberta depende de alguém consultar ativamente, porque o aviso automático por email foi descartado nesta fase ([09:37] Larissa)
- **Mitigação:**
  - Persistência do evento com payload e motivo da falha, preservando a possibilidade de recuperação ([09:18] Diego)
  - Consulta ao histórico de entregas, disponível ao cliente ([09:34] Marcos)
- **Plano de contingência:** replay manual pelo endpoint administrativo, restrito a ADMIN ([09:18] Diego, [09:35] Diego)

#### Crescimento da outbox sem política de arquivamento
- **Probabilidade:** media
- **Impacto:** degradação progressiva da leitura do worker e do banco, já que o arquivamento das linhas entregues foi declarado fora do escopo desta feature ([09:08] Diego)
- **Mitigação:**
  - Índice por estado e por data de criação, com leitura em lote pequeno ([09:08] Diego)
  - Filtro aplicado na gravação, de forma que evento sem nenhum endpoint interessado não chega a ocupar linha ([09:34] Bruno)
- **Plano de contingência:** limpeza ou arquivamento manual das linhas antigas até que a política automática seja implementada em fase seguinte

#### Falha na gravação do evento derruba a mudança de status do pedido
- **Probabilidade:** baixa
- **Impacto:** alto. A operação mais crítica do OMS passa a depender de um módulo novo, porque o rollback conjunto é o comportamento desejado e não pode existir status alterado sem evento correspondente ([09:40] Bruno, [09:41] Diego)
- **Mitigação:**
  - Testes de integração cobrindo os dois lados da transação, o commit que grava e o rollback que não deixa evento órfão
  - Manter a operação dentro da transação simples, restrita à consulta dos endpoints e à gravação das linhas ([09:41] Bruno)
  - Revisão da alteração no serviço de pedidos na sessão de design com Bruno e Diego ([09:50] Larissa)
- **Plano de contingência:** desativar os endpoints de webhook cadastrados. Como o filtro é aplicado na gravação, sem endpoint ativo interessado nenhuma linha é gravada e a transação de mudança de status volta ao comportamento anterior ([09:34] Bruno)

#### Queda do worker interrompe toda a entrega de notificações
- **Probabilidade:** media
- **Impacto:** nenhum evento é entregue enquanto o processo estiver fora, e o limiar de 10 segundos é ultrapassado. Como há um único worker, ele é ponto único de falha nesta fase ([09:12] Diego)
- **Mitigação:**
  - Processo separado da API, de forma que reinício da API não derrube o worker e vice-versa ([09:11] Diego)
  - Os eventos permanecem na outbox enquanto o worker está fora, o que produz atraso e não perda ([09:06] Diego)
- **Plano de contingência:** subir o worker novamente. O acúmulo é processado na ordem de criação, preservando a ordenação por pedido ([09:12] Diego)

#### Vazamento da secret pelo lado do cliente
- **Probabilidade:** media
- **Impacto:** alto. Um terceiro de posse da secret consegue forjar notificações que o cliente aceitaria como legítimas. Cenário com precedente real citado na reunião ([09:22] Diego)
- **Mitigação:**
  - Secret exclusiva por endpoint, isolando o alcance de um vazamento ([09:21] Sofia)
  - Rotação disponível pela API, com 24 horas de convivência para migração sem interrupção ([09:21] Sofia)
  - Secret exposta apenas na criação e na rotação, e incluída na lista de redação do logger ([09:31] Marcos, [09:22] Diego)
- **Plano de contingência:** rotação imediata da secret do endpoint afetado e, se necessário, desativação do endpoint até o cliente concluir a migração

#### Qualquer usuário autenticado pode alterar o destino de um webhook
- **Probabilidade:** media
- **Impacto:** alto. Alterar a URL de destino desvia dados de pedido para fora, e a reunião decidiu conscientemente não restringir role no CRUD nesta fase ([09:36] Marcos, [09:37] Sofia)
- **Mitigação:**
  - Registro em log das operações do módulo, permitindo rastrear quem alterou o quê ([09:29] Bruno)
  - Restrição de ADMIN aplicada onde o risco foi considerado maior, que é o replay ([09:36] Sofia)
  - Revisão de segurança antes do deploy, com Sofia ciente do trade-off ([09:46] Sofia)
- **Plano de contingência:** aplicar o middleware de autorização existente no CRUD como correção pontual, sem mudança estrutural, já que ele está disponível e em uso em [src/middlewares/auth.middleware.ts](../src/middlewares/auth.middleware.ts)

#### Volume de chamadas concentrado sobrecarrega o endpoint do cliente
- **Probabilidade:** media
- **Impacto:** medio. Um cliente com muitos pedidos mudando de status em janela curta recebe uma rajada de chamadas. O controle de vazão foi deliberadamente deixado fora de escopo ([09:38] Diego, [09:39] Larissa)
- **Mitigação:**
  - Observação do comportamento em produção antes de decidir, posição acordada na reunião ([09:39] Larissa)
  - Histórico de entregas permite medir a concentração real de envios por endpoint ([09:34] Marcos)
- **Plano de contingência:** implementar controle de vazão em fase seguinte, e desativar temporariamente o endpoint afetado caso a rajada esteja causando dano ao cliente

#### Atraso na entrega leva o cliente em risco a migrar para o concorrente
- **Probabilidade:** media
- **Impacto:** alto. A Atlas Comercial sinalizou possibilidade de migração caso a entrega não ocorra no prazo ([09:00] Marcos)
- **Mitigação:**
  - Escopo enxuto e fechado na própria reunião, estimado em três sprints com a revisão de segurança incluída ([09:46] Larissa)
  - Reserva antecipada dos dois dias úteis de revisão de segurança, evitando gargalo no final ([09:46] Sofia)
  - Comunicação direta com os clientes sobre prazo e forma de integração ([09:47] Marcos)
- **Plano de contingência:** renegociação de prazo com o cliente, apoiada na visibilidade de progresso por sprint

---

### Critérios de aceitação
Checklist objetivo que define se a feature está pronta.

- Um endpoint criado com URL https e lista de status retorna o cadastro com a secret no corpo da resposta, e a secret não aparece em nenhuma consulta posterior de listagem
- URL com http é recusada com erro de validação, tanto na criação quanto na edição
- Lista de status vazia é recusada na criação
- Endpoint marcado como inativo deixa de gerar eventos novos
- Endpoint vinculado a cliente inexistente é recusado com erro
- Uma mudança de status bem sucedida grava, na mesma transação, o pedido atualizado, o histórico de status e uma linha na outbox por endpoint ativo interessado
- Uma mudança de status que sofre rollback não deixa nenhuma linha órfã na outbox
- Se nenhum endpoint ativo do cliente ouve o status de destino, nenhuma linha é gravada na outbox
- O payload gravado permanece inalterado quando o pedido é modificado depois da gravação do evento
- Em condições normais, o evento é entregue no endpoint do cliente em menos de 10 segundos após o commit da transação
- A requisição chega ao cliente com assinatura HMAC-SHA256 do corpo, verificável com a secret do endpoint
- A alteração de um único byte do corpo invalida a assinatura
- A requisição chega com identificador do evento, assinatura, timestamp de envio e identificação do endpoint nos headers
- O identificador do evento é idêntico em todas as tentativas de entrega do mesmo evento
- Resposta que não chega em 10 segundos é tratada como falha e agenda reenvio, sem descartar o evento
- Evento cujo payload ultrapassa 64KB gera erro e não é enviado
- Uma falha agenda a próxima tentativa segundo a progressão de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas
- Um destino que falha nas cinco tentativas resulta em evento registrado na Dead Letter, com payload e motivo da falha preenchidos
- Sucesso em qualquer tentativa encerra o ciclo e nenhuma tentativa adicional é executada
- Um evento em Dead Letter reprocessado por um ADMIN volta a ser processado pelo worker
- Um usuário sem role ADMIN que tenta o replay recebe recusa por permissão insuficiente
- Toda execução de replay fica registrada em log com a identificação de quem executou
- O histórico de entregas de um endpoint mostra, por tentativa, o resultado de sucesso ou falha, o payload enviado, a resposta recebida e o tempo de resposta
- Após uma rotação, a nova secret é devolvida na resposta e assinaturas geradas com a secret anterior continuam válidas por 24 horas
- Passadas as 24 horas, a secret anterior deixa de ser válida
- O worker roda em processo separado, e o reinício da API não interrompe o processamento de eventos
- A secret não aparece em log em nenhum nível de severidade
- Todos os códigos de erro do módulo usam o prefixo `WEBHOOK_`
- Os erros do módulo são tratados pelo middleware de erro centralizado sem que ele precise ser alterado
- O módulo segue a mesma estrutura dos demais módulos do projeto, com controller, service, repository, routes e schemas
- A revisão de segurança foi concluída e aprovada antes do deploy em produção
- Os três clientes B2B que solicitaram a feature têm endpoint cadastrado e estão recebendo notificações

---

### Testes e validação

Tipos de teste obrigatórios
- Testes unitários da regra de seleção de endpoints na gravação do evento, cobrindo endpoint ativo que ouve o status, endpoint que não ouve, endpoint inativo e cliente sem nenhum endpoint
- Testes unitários do cálculo da assinatura HMAC-SHA256, incluindo a verificação de que alteração no corpo invalida a assinatura ([09:20] Sofia)
- Testes unitários do agendamento da próxima tentativa para cada uma das cinco posições do backoff ([09:17] Diego)
- Testes unitários de validação de schema, cobrindo URL fora de https, lista de status vazia e status inexistente na máquina de estados ([09:23] Sofia)
- Testes unitários da validade da secret anterior dentro e fora da janela de 24 horas ([09:21] Sofia)
- Testes de integração da transação de mudança de status, verificando que o commit grava o evento junto com o pedido e o histórico e que o rollback não deixa evento órfão ([09:40] Bruno)
- Testes de integração do ciclo completo de entrega, do evento pendente até a marcação como entregue
- Testes de integração do ciclo completo de falha, com destino que falha nas cinco tentativas terminando na Dead Letter com motivo preenchido ([09:18] Diego)
- Testes de integração do replay, verificando o retorno do evento à outbox e a recusa para role diferente de ADMIN ([09:35] Diego, [09:36] Sofia)
- Testes de integração do timeout, verificando que destino que não responde em 10 segundos é tratado como falha ([09:42] Diego)
- Testes de integração dos endpoints de configuração, cobrindo criação, edição, remoção, listagem, rotação e consulta de entregas
- Testes ponta a ponta do caminho completo, da mudança de status até a entrega no destino, conforme previsto na estimativa da reunião ([09:46] Larissa)
- Revisão manual de segurança conduzida por Sofia, com foco em assinatura HMAC e geração da secret, verificação de que a secret não aparece em log, de que URL fora de https é recusada e de que o replay é recusado para role diferente de ADMIN ([09:46] Sofia)

Estratégia de validação
- Os testes automatizados seguem a infraestrutura já existente no projeto, com Vitest e Supertest configurados e comando de teste disponível em [package.json](../package.json), sem introduzir framework novo, coerente com a decisão de reuso máximo dos padrões ([09:30] Larissa)
- O arquivo de preparação dos testes em [tests/setup.ts](../tests/setup.ts) limpa as tabelas entre casos e precisa passar a limpar também as novas tabelas do módulo, para que eventos de um teste não vazem para o seguinte
- A validação funcional acontece contra os endpoints reais, no mesmo formato dos testes que já existem para pedidos e autenticação em [tests/orders.test.ts](../tests/orders.test.ts) e [tests/auth.test.ts](../tests/auth.test.ts)
- A entrega ao destino é validada com destino controlado pelo próprio teste, permitindo simular sucesso, resposta de erro e ausência de resposta dentro do timeout
- A revisão de segurança encerra o ciclo de validação e é condição para liberar o deploy em produção ([09:46] Sofia, [09:47] Larissa)
- Teste de carga não faz parte da estratégia, porque a reunião não discutiu volume, throughput nem qualquer meta de capacidade
