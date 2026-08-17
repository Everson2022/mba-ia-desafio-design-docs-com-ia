# TRACKER: rastreabilidade do pacote de documentação

Este documento mapeia cada item registrado no [PRD](PRD.md), no [RFC](RFC.md), no [FDD](FDD.md) e nos [ADRs](adrs/) à sua origem, que é sempre a transcrição da reunião técnica em [TRANSCRICAO.md](../TRANSCRICAO.md) ou o código-fonte da aplicação.

Serve como referência cruzada e como controle de integridade: item de documento que não consegue preencher a coluna Localização não tem origem identificável e não deveria estar no pacote.

**Convenção da coluna Localização.** Para `TRANSCRICAO`, timestamp e nome do falante no formato `[hh:mm] Nome`. Quando o mesmo item foi construído por mais de uma fala, a linha registra a fala que fecha o ponto, e as demais aparecem em linhas próprias quando trazem conteúdo distinto. Para `CODIGO`, caminho de arquivo real do repositório.

---

## Tabela de rastreabilidade

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | docs/PRD.md | Contexto | Três clientes B2B, Atlas Comercial, MaxDistribuição e Nova Cargo, pediram formalmente notificação de mudança de status | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Contexto | Clientes consultam periodicamente a API de pedidos, o que torna a integração deles lenta e cara | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-03 | docs/PRD.md | Restrição | Atlas sinalizou possibilidade de migrar para o concorrente se a entrega não sair até o fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-04 | docs/PRD.md | Requisito Não Funcional | Abaixo de 10 segundos é considerado tempo real pelo cliente | TRANSCRICAO | [09:02] Marcos |
| PRD-CTX-05 | docs/PRD.md | Restrição | Fluxo exclusivamente de saída, o cliente recebe e não envia | TRANSCRICAO | [09:02] Marcos |
| PRD-CTX-06 | docs/PRD.md | Contexto | Escopo confirmado como webhook de saída, o que simplifica o desenho | TRANSCRICAO | [09:03] Sofia |
| PRD-CTX-07 | docs/PRD.md | Público-alvo | Usuário do OMS que representa o cliente opera o cadastro pela API, autenticado com JWT do sistema | TRANSCRICAO | [09:32] Marcos |
| PRD-CTX-08 | docs/PRD.md | Público-alvo | Administrador com role ADMIN é o único autorizado a reprocessar evento em Dead Letter | TRANSCRICAO | [09:36] Sofia |
| PRD-CTX-09 | docs/PRD.md | Contexto | A plataforma não possui nenhum ponto de extensão para notificação externa no ciclo de vida do pedido | CODIGO | src/modules/orders/order.service.ts |
| PRD-CTX-10 | docs/PRD.md | Restrição | Notificação síncrona é inviável, a transação já atualiza pedido, histórico e estoque | TRANSCRICAO | [09:04] Bruno |
| PRD-CTX-11 | docs/PRD.md | Contexto | Eventos carregam dados de pedidos para fora da infraestrutura, sem prova de origem hoje | TRANSCRICAO | [09:19] Sofia |
| PRD-OBJ-01 | docs/PRD.md | Objetivo | Notificar sem polling, com entrega abaixo de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Objetivo | Substituir o polling dos três clientes B2B que solicitaram a feature | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-03 | docs/PRD.md | Objetivo | Nenhum evento desaparece, todo evento termina entregue ou registrado em Dead Letter | TRANSCRICAO | [09:18] Diego |
| PRD-OBJ-04 | docs/PRD.md | Objetivo | Zero chamada HTTP externa dentro da transação de mudança de status | TRANSCRICAO | [09:06] Diego |
| PRD-OBJ-05 | docs/PRD.md | Objetivo | Cobrir cerca de 15 horas de indisponibilidade do cliente sem intervenção manual | TRANSCRICAO | [09:17] Diego |
| PRD-OBJ-06 | docs/PRD.md | Objetivo | 100% das entregas assinadas, com secret exclusiva por endpoint | TRANSCRICAO | [09:21] Sofia |
| PRD-OBJ-07 | docs/PRD.md | Objetivo | Entregar em três sprints, com a revisão de segurança incluída | TRANSCRICAO | [09:46] Larissa |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastrar endpoint com URL, lista de status e secret gerada pelo sistema e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Editar endpoint, alterando URL, lista de status ou estado ativo | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Remover endpoint cadastrado | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Listar endpoints de um cliente | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Rotacionar secret pela API, com a anterior válida em paralelo por 24 horas | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Gravar o evento na outbox dentro da mesma transação que muda o status | TRANSCRICAO | [09:06] Diego |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Worker despacha o evento com payload assinado para o endpoint do cliente | TRANSCRICAO | [09:09] Diego |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Reenviar automaticamente com backoff, em cinco tentativas | TRANSCRICAO | [09:17] Diego |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Mover para Dead Letter em tabela separada o evento que esgotou as tentativas | TRANSCRICAO | [09:18] Diego |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Replay manual de Dead Letter por endpoint administrativo | TRANSCRICAO | [09:35] Diego |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Consultar histórico de entregas com resultado, payload, resposta e tempo de resposta | TRANSCRICAO | [09:34] Marcos |
| PRD-ESC-01 | docs/PRD.md | Escopo | Filtro de eventos por endpoint, aplicado no momento da gravação e não no envio | TRANSCRICAO | [09:34] Bruno |
| PRD-ESC-02 | docs/PRD.md | Escopo | Filtro de eventos é a lista de status que cada endpoint quer ouvir | TRANSCRICAO | [09:33] Marcos |
| PRD-ESC-03 | docs/PRD.md | Escopo | Payload congelado no momento da gravação, refletindo o estado da transição | TRANSCRICAO | [09:52] Larissa |
| PRD-ESC-04 | docs/PRD.md | Escopo | Payload JSON enxuto, sem os itens do pedido | TRANSCRICAO | [09:43] Diego |
| PRD-ESC-05 | docs/PRD.md | Escopo | Worker em processo separado, com entry-point própria e script de execução | TRANSCRICAO | [09:11] Larissa |
| PRD-ESC-06 | docs/PRD.md | Escopo | Identificadores em UUID, seguindo o padrão do projeto | TRANSCRICAO | [09:51] Larissa |
| PRD-RNF-01 | docs/PRD.md | Requisito Não Funcional | Latência mínima de 2 segundos no pior caso, aceita pelo time | TRANSCRICAO | [09:10] Larissa |
| PRD-RNF-02 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 segundos por chamada de entrega | TRANSCRICAO | [09:42] Diego |
| PRD-RNF-03 | docs/PRD.md | Requisito Não Funcional | Leitura em lote pequeno, com índice por estado e por data de criação | TRANSCRICAO | [09:08] Diego |
| PRD-RNF-04 | docs/PRD.md | Requisito Não Funcional | Ciclos de vida de API e worker independentes, reinício de um não afeta o outro | TRANSCRICAO | [09:11] Diego |
| PRD-RNF-05 | docs/PRD.md | Requisito Não Funcional | Indisponibilidade do cliente não bloqueia nem reverte a mudança de status | TRANSCRICAO | [09:04] Bruno |
| PRD-RNF-06 | docs/PRD.md | Requisito Não Funcional | Assinatura HMAC-SHA256 sobre o corpo do request | TRANSCRICAO | [09:20] Sofia |
| PRD-RNF-07 | docs/PRD.md | Requisito Não Funcional | URL obrigatoriamente https, com http recusado na validação | TRANSCRICAO | [09:23] Sofia |
| PRD-RNF-08 | docs/PRD.md | Requisito Não Funcional | Payload limitado a 64KB, com erro em vez de truncamento | TRANSCRICAO | [09:24] Diego |
| PRD-RNF-09 | docs/PRD.md | Requisito Não Funcional | Limite de 64KB classificado como requisito não funcional, não como decisão arquitetural | TRANSCRICAO | [09:24] Larissa |
| PRD-RNF-10 | docs/PRD.md | Requisito Não Funcional | Replay exige role ADMIN e registra quem executou, para auditoria | TRANSCRICAO | [09:36] Sofia |
| PRD-RNF-11 | docs/PRD.md | Requisito Não Funcional | Autorização reaproveita os middlewares existentes, sem middleware novo | CODIGO | src/middlewares/auth.middleware.ts |
| PRD-RNF-12 | docs/PRD.md | Requisito Não Funcional | Secret não pode aparecer em log, campo precisa entrar na lista de redação existente | CODIGO | src/shared/logger/index.ts |
| PRD-RNF-13 | docs/PRD.md | Requisito Não Funcional | Logs estruturados com Pino, sem biblioteca nova | TRANSCRICAO | [09:29] Bruno |
| PRD-RNF-14 | docs/PRD.md | Requisito Não Funcional | Erros tratados pelo middleware de erro centralizado, sem alteração nele | CODIGO | src/middlewares/error.middleware.ts |
| PRD-RNF-15 | docs/PRD.md | Requisito Não Funcional | Atomicidade entre mudança de status e gravação do evento | TRANSCRICAO | [09:40] Bruno |
| PRD-RNF-16 | docs/PRD.md | Requisito Não Funcional | Entrega at-least-once, com deduplicação do lado do cliente | TRANSCRICAO | [09:24] Diego |
| PRD-RNF-17 | docs/PRD.md | Requisito Não Funcional | Ordenação garantida por pedido, sem garantia global, registrada como limitação conhecida | TRANSCRICAO | [09:13] Larissa |
| PRD-RNF-18 | docs/PRD.md | Requisito Não Funcional | API REST sob o prefixo de versão já existente | CODIGO | src/app.ts |
| PRD-RNF-19 | docs/PRD.md | Requisito Não Funcional | Worker usa instância própria do cliente Prisma, no mesmo banco | TRANSCRICAO | [09:30] Bruno |
| PRD-RNF-20 | docs/PRD.md | Requisito Não Funcional | Timestamp do payload em ISO 8601 | TRANSCRICAO | [09:43] Diego |
| PRD-RNF-21 | docs/PRD.md | Compliance | Revisão de segurança obrigatória antes do deploy, com no mínimo dois dias úteis | TRANSCRICAO | [09:46] Sofia |
| PRD-RNF-22 | docs/PRD.md | Restrição | Acessibilidade não se aplica, a entrega é apenas de backend, sem interface visual | TRANSCRICAO | [09:40] Larissa |
| PRD-OOS-01 | docs/PRD.md | Fora de Escopo | Aviso por email quando o webhook do cliente falha, descartado nesta fase | TRANSCRICAO | [09:37] Larissa |
| PRD-OOS-02 | docs/PRD.md | Fora de Escopo | Dashboard visual para o cliente, projeto separado do time de frontend | TRANSCRICAO | [09:40] Larissa |
| PRD-OOS-03 | docs/PRD.md | Fora de Escopo | Controle de vazão na saída, adiado para observação em produção | TRANSCRICAO | [09:39] Larissa |
| PRD-OOS-04 | docs/PRD.md | Fora de Escopo | Webhooks de entrada, do cliente para a plataforma | TRANSCRICAO | [09:02] Marcos |
| PRD-OOS-05 | docs/PRD.md | Fora de Escopo | Arquivamento das linhas entregues da outbox, fora do escopo desta feature | TRANSCRICAO | [09:08] Diego |
| PRD-OOS-06 | docs/PRD.md | Fora de Escopo | Múltiplos workers em paralelo, tratado como problema do futuro | TRANSCRICAO | [09:13] Diego |
| PRD-OOS-07 | docs/PRD.md | Fora de Escopo | Restrição de role no CRUD de configuração, adiada | TRANSCRICAO | [09:37] Sofia |
| PRD-OOS-08 | docs/PRD.md | Fora de Escopo | Garantia exactly-once, descartada por exigir coordenação dos dois lados | TRANSCRICAO | [09:25] Diego |
| PRD-DEC-01 | docs/PRD.md | Decisão | Padrão Outbox no MySQL, em vez de disparo síncrono ou fila externa | TRANSCRICAO | [09:06] Diego |
| PRD-DEC-02 | docs/PRD.md | Decisão | Worker em processo separado, com polling de 2 segundos | TRANSCRICAO | [09:09] Diego |
| PRD-DEC-03 | docs/PRD.md | Decisão | Um único worker, com ordenação garantida apenas por pedido | TRANSCRICAO | [09:12] Diego |
| PRD-DEC-04 | docs/PRD.md | Decisão | Cinco tentativas com backoff de 1m, 5m, 30m, 2h e 12h | TRANSCRICAO | [09:17] Diego |
| PRD-DEC-05 | docs/PRD.md | Decisão | Dead Letter em tabela separada, não marcação na própria outbox | TRANSCRICAO | [09:18] Diego |
| PRD-DEC-06 | docs/PRD.md | Decisão | HMAC-SHA256 com secret por endpoint e rotação com 24 horas de convivência | TRANSCRICAO | [09:22] Sofia |
| PRD-DEC-07 | docs/PRD.md | Decisão | Entrega at-least-once com identificador de evento, em vez de exactly-once | TRANSCRICAO | [09:26] Larissa |
| PRD-DEC-08 | docs/PRD.md | Decisão | Reuso máximo dos padrões já existentes no projeto | TRANSCRICAO | [09:30] Larissa |
| PRD-DEC-09 | docs/PRD.md | Decisão | Payload congelado no momento da gravação | TRANSCRICAO | [09:52] Diego |
| PRD-DEC-10 | docs/PRD.md | Decisão | Replay restrito a ADMIN, com o restante do CRUD aberto a qualquer role autenticada | TRANSCRICAO | [09:36] Marcos |
| PRD-DEP-01 | docs/PRD.md | Dependência | Novas estruturas de persistência criadas por migration no banco existente | CODIGO | prisma/schema.prisma |
| PRD-DEP-02 | docs/PRD.md | Dependência | Alteração no serviço de pedidos, bloqueante para a feature inteira | TRANSCRICAO | [09:40] Bruno |
| PRD-DEP-03 | docs/PRD.md | Dependência | Nova entry-point de processo para o worker, ao lado da entry-point de API | CODIGO | src/server.ts |
| PRD-DEP-04 | docs/PRD.md | Dependência | Conexão de banco própria do processo do worker | TRANSCRICAO | [09:29] Diego |
| PRD-DEP-05 | docs/PRD.md | Dependência | Extensão da lista de redação do logger para cobrir a secret | TRANSCRICAO | [09:22] Diego |
| PRD-DEP-06 | docs/PRD.md | Dependência | Revisão de segurança antes do deploy, bloqueante | TRANSCRICAO | [09:49] Sofia |
| PRD-DEP-07 | docs/PRD.md | Dependência | Sessão de revisão do documento de design antes do início da implementação | TRANSCRICAO | [09:50] Larissa |
| PRD-DEP-08 | docs/PRD.md | Dependência | Documentação de integração no portal do desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| PRD-DEP-09 | docs/PRD.md | Dependência | Confirmação de prazo com os clientes | TRANSCRICAO | [09:47] Marcos |
| PRD-DEP-10 | docs/PRD.md | Dependência | Implementação do lado do cliente, com verificação de assinatura e deduplicação | TRANSCRICAO | [09:25] Diego |
| PRD-RISCO-01 | docs/PRD.md | Risco | Cliente não deduplica e processa o mesmo evento mais de uma vez | TRANSCRICAO | [09:25] Sofia |
| PRD-RISCO-02 | docs/PRD.md | Risco | Evento cai na Dead Letter e ninguém percebe, por não haver aviso automático | TRANSCRICAO | [09:37] Larissa |
| PRD-RISCO-03 | docs/PRD.md | Risco | Crescimento da outbox sem política de arquivamento | TRANSCRICAO | [09:08] Diego |
| PRD-RISCO-04 | docs/PRD.md | Risco | Falha na gravação do evento derruba a mudança de status | TRANSCRICAO | [09:41] Diego |
| PRD-RISCO-05 | docs/PRD.md | Risco | Queda do worker interrompe toda a entrega, por ser processo único | TRANSCRICAO | [09:12] Diego |
| PRD-RISCO-06 | docs/PRD.md | Risco | Vazamento de secret pelo lado do cliente, com precedente real | TRANSCRICAO | [09:22] Diego |
| PRD-RISCO-07 | docs/PRD.md | Risco | Qualquer usuário autenticado pode alterar o destino de um webhook | TRANSCRICAO | [09:37] Sofia |
| PRD-RISCO-08 | docs/PRD.md | Risco | Rajada de chamadas concentrada sobrecarrega o endpoint do cliente | TRANSCRICAO | [09:38] Diego |
| PRD-RISCO-09 | docs/PRD.md | Risco | Atraso na entrega leva o cliente em risco a migrar para o concorrente | TRANSCRICAO | [09:00] Marcos |
| PRD-TESTE-01 | docs/PRD.md | Estratégia de Teste | Testes ponta a ponta previstos na estimativa de sprints da reunião | TRANSCRICAO | [09:46] Larissa |
| PRD-TESTE-02 | docs/PRD.md | Estratégia de Teste | Revisão manual de segurança com foco em HMAC e geração de secret | TRANSCRICAO | [09:46] Sofia |
| PRD-TESTE-03 | docs/PRD.md | Estratégia de Teste | Infraestrutura de teste existente reusada, sem framework novo | CODIGO | package.json |
| PRD-TESTE-04 | docs/PRD.md | Estratégia de Teste | Limpeza entre casos precisa cobrir as tabelas novas do módulo | CODIGO | tests/setup.ts |
| PRD-TESTE-05 | docs/PRD.md | Estratégia de Teste | Validação funcional contra os endpoints reais, no formato dos testes existentes | CODIGO | tests/orders.test.ts |
| RFC-PROP-01 | docs/RFC.md | Decisão | Captura atômica do evento na transação, com função recebendo o cliente transacional | TRANSCRICAO | [09:41] Bruno |
| RFC-PROP-02 | docs/RFC.md | Decisão | Entrega assíncrona por worker separado em polling | TRANSCRICAO | [09:11] Diego |
| RFC-PROP-03 | docs/RFC.md | Decisão | Resiliência por timeout, reenvio com backoff e Dead Letter | TRANSCRICAO | [09:15] Diego |
| RFC-PROP-04 | docs/RFC.md | Decisão | Autenticidade e integridade por assinatura com secret por endpoint | TRANSCRICAO | [09:21] Sofia |
| RFC-PROP-05 | docs/RFC.md | Decisão | Semântica at-least-once com identificador de evento estável | TRANSCRICAO | [09:25] Diego |
| RFC-PROP-06 | docs/RFC.md | Decisão | Identificador do cliente vai no corpo ou no caminho, não no token | TRANSCRICAO | [09:32] Larissa |
| RFC-PROP-07 | docs/RFC.md | Decisão | Módulo novo no formato dos existentes, com erros no prefixo do módulo | TRANSCRICAO | [09:27] Bruno |
| RFC-ALT-01 | docs/RFC.md | Trade-off | Disparo síncrono descartado, travaria mudança de status de outros pedidos | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Trade-off | Fila ou stream externo descartado, exigiria infraestrutura nova para time pequeno | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Trade-off | Trigger de banco descartada, MySQL não notifica processo externo | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Trade-off | Exactly-once descartado, exigiria coordenação dos dois lados | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-05 | docs/RFC.md | Trade-off | Secret global descartada, vazamento de uma comprometeria todas | TRANSCRICAO | [09:21] Sofia |
| RFC-ALT-06 | docs/RFC.md | Trade-off | Três tentativas descartadas, matariam o evento em cerca de 30 minutos | TRANSCRICAO | [09:16] Diego |
| RFC-ALT-07 | docs/RFC.md | Trade-off | Reenvio indefinido descartado, deixaria o evento pendurado para sempre | TRANSCRICAO | [09:15] Diego |
| RFC-ABERTO-01 | docs/RFC.md | Questão em Aberto | Controle de vazão na saída, sem gatilho nem política definidos | TRANSCRICAO | [09:39] Diego |
| RFC-ABERTO-02 | docs/RFC.md | Questão em Aberto | Escala para múltiplos workers, caminhos citados mas não avaliados | TRANSCRICAO | [09:13] Diego |
| RFC-ABERTO-03 | docs/RFC.md | Questão em Aberto | Arquivamento da outbox, sem prazo, gatilho ou destino definidos | TRANSCRICAO | [09:08] Diego |
| RFC-ABERTO-04 | docs/RFC.md | Questão em Aberto | Endurecimento de permissão no CRUD, sem critério nem prazo | TRANSCRICAO | [09:37] Sofia |
| RFC-ABERTO-05 | docs/RFC.md | Questão em Aberto | Destino dos eventos gravados quando um endpoint é removido, não levantado na reunião | CODIGO | src/modules/orders/order.service.ts |
| RFC-IMP-01 | docs/RFC.md | Impacto | Único ponto de mudança em código de produção existente | TRANSCRICAO | [09:40] Bruno |
| RFC-IMP-02 | docs/RFC.md | Impacto | Prazo de três sprints, com revisão de segurança bloqueante incluída | TRANSCRICAO | [09:47] Larissa |
| RFC-IMP-03 | docs/RFC.md | Impacto | Expectativa comercial de entrega para o fim de novembro | TRANSCRICAO | [09:45] Marcos |
| FDD-FLUXO-01 | docs/FDD.md | Fluxo | Gravação do evento como última operação dentro da transação de mudança de status | TRANSCRICAO | [09:41] Bruno |
| FDD-FLUXO-02 | docs/FDD.md | Fluxo | Filtro por status aplicado na gravação, sem gravar linha quando ninguém ouve | TRANSCRICAO | [09:34] Diego |
| FDD-FLUXO-03 | docs/FDD.md | Fluxo | Falha na gravação propaga e provoca rollback da transação inteira | TRANSCRICAO | [09:41] Diego |
| FDD-FLUXO-04 | docs/FDD.md | Fluxo | Worker consulta pendentes mais antigos a cada 2 segundos, em lote pequeno | TRANSCRICAO | [09:09] Diego |
| FDD-FLUXO-05 | docs/FDD.md | Fluxo | Processamento sequencial em ordem de criação preserva ordenação por pedido | TRANSCRICAO | [09:12] Diego |
| FDD-FLUXO-06 | docs/FDD.md | Fluxo | Progressão de reenvio de 1m, 5m, 30m, 2h e 12h, com janela de cerca de 15 horas | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-07 | docs/FDD.md | Fluxo | Dead Letter guarda payload, motivo da falha e timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-08 | docs/FDD.md | Fluxo | Replay recoloca o evento na outbox como pendente | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-09 | docs/FDD.md | Fluxo | Rotação mantém a secret anterior válida em paralelo por 24 horas | TRANSCRICAO | [09:21] Sofia |
| FDD-DADOS-01 | docs/FDD.md | Modelo de Dados | Tabela de configuração guarda url, secret, cliente e estado ativo | TRANSCRICAO | [09:21] Bruno |
| FDD-DADOS-02 | docs/FDD.md | Modelo de Dados | Estados do evento na outbox: pendente, processando, falhou e entregue | TRANSCRICAO | [09:08] Diego |
| FDD-DADOS-03 | docs/FDD.md | Modelo de Dados | Índice por estado e por data de criação para a consulta do worker | TRANSCRICAO | [09:08] Diego |
| FDD-DADOS-04 | docs/FDD.md | Modelo de Dados | Registro de entregas guarda resultado, payload, resposta e tempo de resposta | TRANSCRICAO | [09:34] Marcos |
| FDD-DADOS-05 | docs/FDD.md | Modelo de Dados | Convenções de modelagem seguidas dos models existentes, com UUID e nome de tabela | CODIGO | prisma/schema.prisma |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | Cadastro de endpoint devolve a secret apenas na resposta de criação | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | Endpoints de edição, remoção e listagem de webhooks por cliente | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | Consulta de histórico de entregas, com referência de últimas 100 | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | Endpoint administrativo de replay de Dead Letter | TRANSCRICAO | [09:35] Diego |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | Header com identificador do evento, chave de deduplicação do cliente | TRANSCRICAO | [09:25] Diego |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | Header com assinatura HMAC do corpo do request | TRANSCRICAO | [09:20] Sofia |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | Header com timestamp do envio, para o cliente detectar replay attack | TRANSCRICAO | [09:44] Diego |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | Header com identificador do endpoint, para cliente com múltiplos cadastros | TRANSCRICAO | [09:44] Sofia |
| FDD-CONTRATO-09 | docs/FDD.md | Contrato | Payload com identificação do evento, tipo, timestamp, dados do pedido e transição | TRANSCRICAO | [09:43] Diego |
| FDD-CONTRATO-10 | docs/FDD.md | Contrato | Payload sem os itens do pedido, cliente consulta a API de pedidos se precisar | TRANSCRICAO | [09:44] Bruno |
| FDD-CONTRATO-11 | docs/FDD.md | Contrato | Envelope de paginação e de erro seguem os helpers e o middleware existentes | CODIGO | src/shared/http/response.ts |
| FDD-CONTRATO-12 | docs/FDD.md | Contrato | Limites de página e validação de identificador seguem os schemas existentes | CODIGO | src/modules/orders/order.schemas.ts |
| FDD-ERRO-01 | docs/FDD.md | Restrição | Códigos de erro do módulo com prefixo WEBHOOK_ | TRANSCRICAO | [09:29] Larissa |
| FDD-ERRO-02 | docs/FDD.md | Contrato | Códigos citados na reunião: not found, invalid url e secret required | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-03 | docs/FDD.md | Contrato | Erro de payload acima de 64KB, sem truncamento | TRANSCRICAO | [09:23] Sofia |
| FDD-ERRO-04 | docs/FDD.md | Contrato | Timeout de 10 segundos contabilizado como falha para reenvio | TRANSCRICAO | [09:42] Diego |
| FDD-ERRO-05 | docs/FDD.md | Integração | Classes novas estendem a hierarquia de erro existente | CODIGO | src/shared/errors/http-errors.ts |
| FDD-ERRO-06 | docs/FDD.md | Integração | Middleware de erro já trata a hierarquia, erros de validação e erros do Prisma | CODIGO | src/middlewares/error.middleware.ts |
| FDD-RES-01 | docs/FDD.md | Resiliência | Estado do evento vive no banco, queda do worker atrasa mas não perde | TRANSCRICAO | [09:06] Diego |
| FDD-RES-02 | docs/FDD.md | Resiliência | Sem circuit breaker nesta fase, controle de vazão foi adiado | TRANSCRICAO | [09:39] Diego |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Logs estruturados com o logger existente, sem biblioteca nova | CODIGO | src/shared/logger/index.ts |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Log de auditoria do replay, com identificação de quem executou | TRANSCRICAO | [09:36] Sofia |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | Correlação por identificador, no mesmo padrão do identificador de requisição existente | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-INT-01 | docs/FDD.md | Integração | Chamada de publicação dentro da transação de mudança de status | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Enum de status é o domínio válido da lista de eventos do endpoint | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Classe base de erro reusada pelas classes novas do módulo | CODIGO | src/shared/errors/app-error.ts |
| FDD-INT-04 | docs/FDD.md | Integração | Autenticação em todas as rotas e autorização por papel no replay | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-05 | docs/FDD.md | Integração | Validação de entrada com o middleware e os schemas do módulo | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-06 | docs/FDD.md | Integração | Registro do router do módulo no router principal | CODIGO | src/routes/index.ts |
| FDD-INT-07 | docs/FDD.md | Integração | Montagem da cadeia repository, service e controller na injeção manual existente | CODIGO | src/app.ts |
| FDD-INT-08 | docs/FDD.md | Integração | Entry-point do worker no molde da entry-point de API | CODIGO | src/server.ts |
| FDD-INT-09 | docs/FDD.md | Integração | Worker cria a própria instância do cliente Prisma | CODIGO | src/config/database.ts |
| FDD-INT-10 | docs/FDD.md | Integração | Parâmetros do worker validados no schema de ambiente existente | CODIGO | src/config/env.ts |
| FDD-INT-11 | docs/FDD.md | Integração | Router do módulo segue o padrão de aplicação de middleware por rota | CODIGO | src/modules/orders/order.routes.ts |
| FDD-INT-12 | docs/FDD.md | Integração | Script de execução do worker no molde dos scripts existentes | CODIGO | package.json |
| FDD-DEP-01 | docs/FDD.md | Dependência | Versão do MySQL em uso pelo projeto | CODIGO | docker-compose.yml |
| FDD-DEP-02 | docs/FDD.md | Dependência | Controller do módulo segue o padrão de resposta dos controllers existentes | CODIGO | src/modules/orders/order.controller.ts |
| ADR-001 | docs/adrs/ADR-001-padrao-outbox-no-mysql.md | Decisão | Padrão Outbox no MySQL, com gravação na mesma transação da mudança de status | TRANSCRICAO | [09:06] Diego |
| ADR-001-ALT-01 | docs/adrs/ADR-001-padrao-outbox-no-mysql.md | Trade-off | Disparo síncrono descartado, acoplaria a operação interna à disponibilidade do cliente | TRANSCRICAO | [09:04] Bruno |
| ADR-001-ALT-02 | docs/adrs/ADR-001-padrao-outbox-no-mysql.md | Trade-off | Fila externa descartada, exigiria infraestrutura nova | TRANSCRICAO | [09:07] Larissa |
| ADR-001-COD | docs/adrs/ADR-001-padrao-outbox-no-mysql.md | Integração | Transação de mudança de status como ponto de acoplamento | CODIGO | src/modules/orders/order.service.ts |
| ADR-002 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Decisão | Worker em processo separado, lendo a outbox em polling de 2 segundos | TRANSCRICAO | [09:11] Diego |
| ADR-002-ALT-01 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Trade-off | Trigger de banco descartada, MySQL não tem listener nativo | TRANSCRICAO | [09:09] Diego |
| ADR-002-ALT-02 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Trade-off | Worker dentro da API descartado, reinício da API derrubaria o processamento | TRANSCRICAO | [09:11] Diego |
| ADR-002-ALT-03 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Trade-off | Múltiplos workers descartados nesta fase, perderiam a ordenação | TRANSCRICAO | [09:12] Diego |
| ADR-002-COD | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Integração | Entry-point de API usada como molde para a entry-point do worker | CODIGO | src/server.ts |
| ADR-003 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | Cinco tentativas com backoff e Dead Letter em tabela separada | TRANSCRICAO | [09:17] Larissa |
| ADR-003-ALT-01 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Trade-off | Três tentativas descartadas, insuficiente diante de manutenção de duas horas | TRANSCRICAO | [09:16] Diego |
| ADR-003-ALT-02 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Trade-off | Reenvio indefinido descartado, evento ficaria pendurado para sempre | TRANSCRICAO | [09:15] Diego |
| ADR-003-ALT-03 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Trade-off | Marcação de falha na própria outbox descartada, poluiria a leitura do fluxo ativo | TRANSCRICAO | [09:18] Diego |
| ADR-003-COD | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Integração | Middleware de autorização reusado para restringir o replay a ADMIN | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | HMAC-SHA256 com secret por endpoint e rotação com 24 horas de convivência | TRANSCRICAO | [09:22] Sofia |
| ADR-004-ALT-01 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Trade-off | Secret global descartada, um vazamento comprometeria todos os clientes | TRANSCRICAO | [09:21] Sofia |
| ADR-004-ALT-02 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Trade-off | Rotação sem convivência descartada, criaria janela sem verificação válida | TRANSCRICAO | [09:21] Sofia |
| ADR-004-ALT-03 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Trade-off | Truncar payload grande descartado em favor de erro explícito | TRANSCRICAO | [09:23] Sofia |
| ADR-004-COD | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Integração | Lista de redação do logger precisa passar a cobrir a secret | CODIGO | src/shared/logger/index.ts |
| ADR-005 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | Entrega at-least-once com identificador de evento no header | TRANSCRICAO | [09:25] Diego |
| ADR-005-ALT-01 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Trade-off | Exactly-once descartado, exigiria coordenação e complexidade maior | TRANSCRICAO | [09:25] Diego |
| ADR-005-CONS | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Trade-off | Deduplicação transferida para o cliente, ponto levantado e aceito na reunião | TRANSCRICAO | [09:25] Sofia |
| ADR-005-COD | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Integração | Padrão existente de identificador por requisição devolvido em header | CODIGO | src/middlewares/request-logger.middleware.ts |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Reuso integral dos padrões do projeto, sem dependência nova | TRANSCRICAO | [09:30] Larissa |
| ADR-006-ALT-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Trade-off | Compartilhar o cliente Prisma entre API e worker descartado, é por processo | TRANSCRICAO | [09:30] Bruno |
| ADR-006-COD-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Integração | Estrutura de módulo em cinco arquivos seguida do módulo de pedidos | CODIGO | src/modules/orders/order.controller.ts |
| ADR-006-COD-02 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Integração | Classes de erro de domínio existentes usadas como molde | CODIGO | src/shared/errors/http-errors.ts |
| ADR-006-COD-03 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Integração | Middleware de erro trata os erros do módulo sem alteração | CODIGO | src/middlewares/error.middleware.ts |
| ADR-006-COD-04 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Integração | Helper de paginação reusado nas listagens do módulo | CODIGO | src/shared/http/response.ts |
| ADR-006-COD-05 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Integração | Convenção de identificador em UUID seguida dos models existentes | CODIGO | prisma/schema.prisma |
| ADR-007 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Decisão | Payload renderizado e congelado no momento da gravação | TRANSCRICAO | [09:52] Larissa |
| ADR-007-ALT-01 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Trade-off | Renderizar no envio descartado, o evento deixaria de refletir a transição | TRANSCRICAO | [09:51] Bruno |
| ADR-007-ALT-02 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Trade-off | Payload completo com itens descartado, para não inflar | TRANSCRICAO | [09:43] Diego |
| ADR-007-COD | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Integração | Pedido recarregado com relações ao fim da transação é a fonte do snapshot | CODIGO | src/modules/orders/order.service.ts |

---

## Cobertura

| Métrica | Valor |
| --- | --- |
| Total de linhas | 206 |
| Linhas com Fonte igual a TRANSCRICAO | 163, equivalente a 79% |
| Linhas com Fonte igual a CODIGO | 43, equivalente a 21% |
| Falas distintas da transcrição referenciadas | 76 |
| Arquivos distintos do repositório referenciados | 23 |
| Documentos cobertos | PRD, RFC, FDD e os 7 ADRs |
| Identificadores duplicados | Nenhum |

Todos os timestamps da coluna Localização seguem o formato `[hh:mm] Nome` e correspondem a falas existentes em [TRANSCRICAO.md](../TRANSCRICAO.md). Todos os caminhos de arquivo apontam para arquivos que existem no repositório.

## Itens sem origem na transcrição

Três itens do pacote não vêm de fala da reunião e estão registrados aqui de forma explícita, para que ninguém os leia como decisão do time:

| Item | Documento | Natureza |
| --- | --- | --- |
| Destino dos eventos já gravados quando um endpoint é removido | docs/RFC.md, docs/FDD.md | Lacuna identificada ao detalhar o desenho, registrada como questão em aberto e não como decisão |
| Classificação de probabilidade e impacto dos riscos | docs/PRD.md, docs/FDD.md | Avaliação de quem escreveu os documentos, a reunião levanta os riscos mas não os classifica |
| Destravamento de evento preso em processamento após queda do worker | docs/FDD.md | Consequência técnica necessária da garantia at-least-once decidida em [09:24] Diego, sem tratamento próprio na reunião |
