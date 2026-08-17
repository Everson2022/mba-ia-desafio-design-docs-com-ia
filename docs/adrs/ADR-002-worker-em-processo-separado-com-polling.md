# ADR-002: Worker em processo separado, lendo a outbox em polling de 2 segundos

**Status:** Aceito
**Data:** 2026-08-16
**Decisores:** Diego (Engenheiro Sênior, Plataforma), Larissa (Tech Lead), Bruno (Engenheiro Pleno, Pedidos)
**Consultados:** Marcos (Product Manager)
**ADRs relacionados:** [ADR-001](ADR-001-padrao-outbox-no-mysql.md) define a tabela que este worker consome, [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md) define o comportamento em caso de falha na entrega, [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md) define o formato da entry-point nova

---

## Contexto

Com a decisão de gravar os eventos em uma outbox no MySQL ([ADR-001](ADR-001-padrao-outbox-no-mysql.md)), a pergunta seguinte foi como esse conteúdo é lido e entregue ([09:08] Larissa).

Duas restrições delimitam o espaço de solução:

- O requisito de produto é notificar em menos de 10 segundos, valor que os clientes tratam como tempo real ([09:02] Marcos).
- O MySQL não possui mecanismo nativo de notificação de processo externo, diferente do Postgres. Trigger de banco existe, mas apenas executa SQL, não avisa processo nenhum ([09:09] Diego).

Há também uma preocupação de ciclo de vida: se o consumidor viver dentro do processo da API, um reinício da API derruba o processamento de eventos ([09:11] Diego).

## Decisão

Executar a entrega em um **processo separado da API**, que lê a outbox em **polling a cada 2 segundos**, processando os eventos pendentes mais antigos em lote pequeno e marcando o desfecho de cada um ([09:09] Diego).

O que faz parte desta decisão:

- **Processo separado, obrigatoriamente.** O worker não pode rodar dentro da mesma instância da API, para que o ciclo de vida de um não afete o do outro ([09:11] Diego).
- **Entry-point própria**, ao lado da entry-point de API existente em [src/server.ts](../../src/server.ts), acionada por script dedicado ([09:11] Larissa), com a lógica de processamento residindo dentro do módulo de webhooks ([09:28] Bruno).
- **Instância própria do cliente Prisma**, apontando para o mesmo banco e a mesma URL de conexão, porque o cliente é por processo ([09:29] Diego, [09:30] Bruno).
- **Um único worker nesta fase.** O processamento segue a ordem de criação dos eventos, o que garante ordenação por pedido. Não há garantia de ordenação global, e isso fica registrado como limitação conhecida ([09:12] Diego, [09:13] Larissa).

O intervalo de 2 segundos foi escolhido por caber com folga no limiar de 10 segundos, e a reunião aceitou explicitamente que a latência mínima passa a ser de 2 segundos no pior caso ([09:10] Larissa, [09:10] Marcos).

## Alternativas Consideradas

### Trigger de banco para notificar o consumidor de forma reativa

Bruno levantou a possibilidade de usar trigger no banco em vez de polling, para ser mais reativo ([09:09] Bruno).

Foi descartada porque o MySQL não tem listener nativo equivalente ao NOTIFY e LISTEN do Postgres. A trigger existe, mas apenas executa SQL, não notifica processo externo. Para avisar o worker seria preciso improvisar algo como escrever em arquivo ou chamar um endpoint, o que a reunião classificou como solução artificial ([09:09] Diego).

**Trade-off que motivou o descarte:** ganharia latência menor que o piso de 2 segundos, ao custo de um mecanismo frágil e não idiomático para o banco em uso, sem benefício real diante de um requisito de 10 segundos que o polling já atende com folga.

### Worker rodando dentro do processo da API

Executar o laço de entrega dentro da própria aplicação Express, sem processo adicional. Dispensaria entry-point nova, script novo e um processo a mais para operar.

Foi descartada porque, se a API reinicia, o worker vai junto e o processamento para ([09:11] Diego). Deploys e reinícios de API são rotina, e a entrega de notificação não pode depender deles.

**Trade-off que motivou o descarte:** ganharia simplicidade operacional, com um único processo para subir e monitorar, ao custo de acoplar a continuidade da entrega ao ciclo de vida da API.

### Múltiplos workers em paralelo desde o início

Processar eventos com concorrência, para escalar a vazão de entrega.

Foi descartada para esta fase. Com múltiplos workers em paralelo, a garantia de ordenação se perde, e os caminhos conhecidos para recuperá-la, particionar por pedido ou usar lock pessimista, foram classificados como problema do futuro ([09:12] Diego, [09:13] Diego). Marcos reforçou que os clientes nunca pediram ordenação global, apenas saber se cada pedido deles mudou ([09:14] Marcos).

**Trade-off que motivou o descarte:** ganharia vazão e ausência de ponto único de falha, ao custo de complexidade de coordenação e perda da ordenação por pedido, sem demanda que justifique agora.

## Consequências

### Positivas

- Reinício ou queda da API não interrompe a entrega de eventos, e o inverso também vale.
- Ordenação por pedido garantida naturalmente, sem nenhum mecanismo de coordenação, pela combinação de worker único com leitura por ordem de criação.
- Nenhum mecanismo exótico de notificação. O desenho usa apenas consulta SQL indexada, sobre a stack já operada pelo time.
- Todo o estado do processamento vive no banco, então uma queda do worker não perde evento nenhum. O processamento retoma do ponto onde parou, e o efeito é atraso, não perda.
- O worker pode ser reiniciado, movido de máquina ou pausado sem qualquer impacto na API.

### Negativas e trade-offs explícitos

- **Latência mínima de 2 segundos no pior caso.** É consequência direta do polling e foi aceita de forma explícita na reunião ([09:10] Larissa). O trade-off é abrir mão de latência menor em troca de um mecanismo simples e idiomático.
- **Consulta constante ao banco mesmo com a outbox vazia.** O laço executa a cada 2 segundos independentemente de haver evento, o que gera carga de base permanente, ainda que pequena e indexada.
- **O worker é ponto único de falha nesta fase.** Se o processo cai e ninguém percebe, nenhum evento é entregue, e como nada falha visivelmente, o sintoma é apenas acúmulo silencioso. A mitigação depende de observar a contagem de pendentes e a idade do evento mais antigo.
- **Mais um processo para operar e implantar.** Passa a existir um segundo artefato executável, com ciclo de vida próprio, o que aumenta a superfície de operação.
- **Escalar horizontalmente exigirá trabalho adicional**, seja particionamento por pedido ou lock pessimista, já que a ordenação atual depende de haver um único worker ([09:13] Diego).

## Referências

**Transcrição:** [09:02] Marcos, [09:08] Larissa, [09:09] Diego, [09:09] Bruno, [09:10] Larissa, [09:10] Marcos, [09:11] Diego, [09:11] Larissa, [09:11] Bruno, [09:12] Diego, [09:13] Diego, [09:13] Larissa, [09:14] Marcos, [09:28] Bruno, [09:29] Diego, [09:30] Bruno, [09:48] Larissa

**Código existente:**

| Arquivo | Relevância |
| --- | --- |
| [src/server.ts](../../src/server.ts) | Molde da entry-point nova: bootstrap, log de início, tratamento de sinais de encerramento e desconexão do banco |
| [src/config/database.ts](../../src/config/database.ts) | Função de criação do cliente Prisma, reusada pelo worker para criar a própria instância |
| [package.json](../../package.json) | Padrão dos scripts de execução, base para o script novo do worker |
