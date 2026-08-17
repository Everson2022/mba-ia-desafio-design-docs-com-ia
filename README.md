# Da Reunião ao Documento: pacote de design docs gerado com IA

Entrega do desafio **Da Reunião ao Documento: Design Docs Gerados por IA**, do MBA de Engenharia de Software com IA da Full Cycle.

Este README documenta o processo de produção. Os documentos entregues estão em [docs/](docs/), e o enunciado original foi preservado em [DESAFIO.md](DESAFIO.md) para consulta lado a lado com a [checagem dos critérios de aceite](#checklist-dos-critérios-de-aceite).

**Atalhos:** [PRD](docs/PRD.md) · [RFC](docs/RFC.md) · [FDD](docs/FDD.md) · [ADRs](docs/adrs/) · [TRACKER](docs/TRACKER.md) · [Checklist de aceite](#checklist-dos-critérios-de-aceite)

---

## Sobre o desafio

O ponto de partida é a transcrição literal de uma reunião técnica de 55 minutos, com cinco participantes, na qual um time decidiu como construir um Sistema de Webhooks de Notificação de Pedidos para um OMS que já roda em produção. Nada além dessa transcrição foi registrado. A tarefa é transformar essa conversa, somada ao código da aplicação existente, em um pacote de documentação técnica acionável o suficiente para o time começar a implementar: PRD, RFC, FDD, ADRs, um tracker de rastreabilidade e este README.

A restrição que define o desafio não é escrever bonito, é não inventar. Toda informação registrada precisa ser rastreável a uma fala da transcrição ou a um arquivo do código. E a transcrição não é uma lista de requisitos: ela contém decisões fechadas, pontos explicitamente descartados, pontos adiados para o futuro e detalhes técnicos secundários misturados na conversa. Separar o que entra do que não entra é metade do trabalho. A entrega é puramente documental, e o código da aplicação não foi tocado.

---

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
| --- | --- |
| **Claude Code** (modelo Opus 5, contexto de 1M), rodando na extensão do VS Code | Ferramenta única de produção. Leitura da transcrição e do código, condução da entrevista estruturada do PRD, redação de todos os documentos, e verificação automatizada da entrega ao final |

O contexto de 1M foi determinante para o método adotado. A transcrição inteira, os arquivos relevantes do código e os documentos já produzidos couberam na mesma sessão, sem necessidade de fragmentar o trabalho em conversas separadas. Isso permitiu que cada documento fosse escrito com os anteriores à vista, o que é justamente o que evita duplicação entre PRD, RFC, ADR e FDD.

Duas capacidades da ferramenta foram usadas de forma deliberada e mudaram o resultado:

- **Leitura direta do repositório.** Em vez de descrever o código para a IA, ela leu os arquivos. Toda referência a caminho de arquivo, número de linha, nome de classe ou formato de resposta nos documentos foi extraída do código real, não da memória do modelo.
- **Execução de comandos para verificação.** Ao final, em vez de confiar na revisão por leitura, foram rodados scripts que conferem se cada timestamp citado existe de fato na transcrição e se cada caminho de arquivo existe de fato no repositório. Isso é detalhado em [Verificação final](#verificação-final).

---

## Workflow adotado

### Ordem de produção e por que ela mudou

O enunciado sugere produzir ADRs primeiro, depois RFC, FDD e PRD por último. A ordem seguida foi outra:

```
1. Contextualização        leitura do enunciado, da transcrição e mapeamento do código
2. PRD                     produzido por entrevista estruturada, 12 etapas
3. FDD                     produzido em uma passada, já com o PRD fechado
4. ADRs                    7 decisões, extraídas do que PRD e FDD já haviam consolidado
5. RFC                     consolidação da proposta, referenciando os ADRs
6. TRACKER                 varredura dos documentos prontos, com verificação automatizada
7. README                  este documento, por último
```

A inversão foi consequência do método, não descuido. O PRD foi produzido por entrevista, etapa a etapa, e essa entrevista funcionou como o levantamento completo da transcrição: ao chegar na etapa de decisões e trade-offs, as dez decisões já estavam mapeadas com timestamp, alternativa descartada e trade-off explícito. Os ADRs, escritos depois, formalizaram um material que já estava organizado, em vez de precisarem descobri-lo. O RFC veio depois dos ADRs, como o enunciado sugere, para poder referenciá-los por link.

### Como a interação foi organizada

O papel humano foi de maestro, e as intervenções mais importantes não foram pedidos de texto, foram decisões de escopo e de critério:

- Definir a regra de rastreabilidade como inegociável, e reafirmá-la quando o documento começou a acumular suposições
- Escolher, diante da lacuna de números na transcrição, entre inventar um valor plausível ou manter o impacto qualitativo
- Mandar remover as marcações de hipótese do PRD, o que forçou uma decisão binária: ou o item tem origem e fica sem marca, ou não tem origem e sai
- Cortar meta-comentário sobre o processo de dentro dos documentos de produto, mantendo cada texto no seu nível

O PRD foi o único documento produzido por entrevista, com uma pergunta por etapa e confirmação antes de avançar. Do FDD em diante, com o contexto já consolidado, os documentos foram produzidos em uma passada e revisados depois, o que foi mais rápido sem perda de qualidade.

### Checagem contínua contra os critérios de aceite

O enunciado traz uma lista fechada de critérios por documento, com mínimos numéricos. Em vez de deixar essa conferência para o fim, cada documento foi checado contra a lista logo depois de escrito, antes de passar para o próximo, o que evitou descobrir lacuna estrutural quando corrigir já custaria reescrever. O resultado consolidado está no [checklist dos critérios de aceite](#checklist-dos-critérios-de-aceite).

---

## Prompts customizados

Os prompts de entrevista do PRD e de geração do FDD vieram do curso. O que segue são os prompts escritos ou adaptados durante o processo, que foram os que efetivamente moldaram o resultado.

### 1. Contextualização inicial

Usado antes de qualquer documento, para que a IA lesse o material bruto em vez de trabalhar por suposição.

```
Leia o README.md e entenda o objetivo do desafio. Assim que você tiver entendido,
vamos começar a fazer o PRD.
```

Simples de propósito. O efeito buscado foi impedir que a produção começasse antes da leitura do enunciado, da transcrição e do código. A IA leu os três, mapeou a estrutura de módulos, a máquina de estados, a transação de mudança de status, as classes de erro, o middleware de autorização e o logger, e só então a entrevista começou.

### 2. Regra de rastreabilidade, reafirmada no meio do processo

Este foi o prompt mais determinante de toda a entrega. Foi dado quando o PRD já estava na metade e começava a acumular suposições razoáveis, porém sem origem.

```
PRD precisa estar relacionado à reunião e não inventar dados.
```

Uma frase, e ela reorganizou o documento inteiro. A partir dela, foram descartados: percentil de latência, meta de disponibilidade em percentual, nomes técnicos de métrica, mecanismo de supervisão de processo e um teto numérico de degradação de latência. O que ficou foi apenas o que a reunião disse ou o que o código mostra.

### 3. Corte das marcações de hipótese

Dado quando o PRD já estava escrito e ainda carregava sete itens rotulados como hipótese.

```
Esse **hipótese** acredito que possamos remover do PRD.
```

O pedido era de limpeza visual, mas a execução exigia uma decisão de conteúdo que valia a pena explicitar antes de agir: remover o rótulo e manter o texto transformaria suposição em afirmação, com aparência de decisão do time. A saída foi remover os sete itens inteiros. Um pedido de estilo, tratado como decisão de integridade.

### 4. Verificação por execução, não por leitura

Aplicado ao final de cada documento, e de forma sistemática no tracker.

```
Verifique se cada timestamp citado existe de fato na TRANSCRICAO.md com o falante
correto, e se cada caminho de arquivo citado existe de fato no repositório.
Rode o comando, não confie na leitura.
```

Este prompt é o que separa rastreabilidade real de rastreabilidade aparente. Um documento pode estar cheio de `[09:17] Diego` e ainda assim conter citação atribuída à pessoa errada ou a um horário que não existe. A verificação por script encontrou e corrigiu erros de contagem que a revisão por leitura não pegaria.

---

## Iterações e ajustes

Foram **cinco ciclos principais**, e o PRD sozinho consumiu três deles. Os ajustes concretos, na ordem em que aconteceram:

### 1. A lacuna de números no problema

Na etapa de problema e oportunidade do PRD, a IA sinalizou que a transcrição não fornece nenhum número de custo, tempo perdido ou volume. Marcos diz que o polling está "lento e caro" e cita o risco de churn, mas não dá frequência de polling, volume de requisições nem custo. Foram apresentadas três saídas: manter só o impacto qualitativo, incluir números marcados como hipótese, ou usar apenas os dois números reais da reunião, que são o limiar de 10 segundos e o prazo de três sprints.

A escolha foi a terceira. O PRD ficou sem métrica de "redução de X% de chamadas", que seria bonita e não teria origem.

### 2. Sete marcações de hipótese, e a decisão de remover

O PRD chegou a ter sete itens marcados como **hipótese**: percentil p99 da latência, meta de uptime de 99.9%, supervisão do worker com reinício automático, métricas do módulo, tracing distribuído, teto de 5ms de degradação e uma métrica de Dead Letter dentro de um risco. A marcação era honesta, mas poluía o documento.

Ao receber a instrução de removê-las, a saída não foi apagar o rótulo e manter o texto, o que seria o pior dos mundos: afirmação sem origem parecendo decisão do time. Foram removidos os sete itens inteiros, e a seção de observabilidade do PRD ficou com três itens, todos rastreáveis. O critério que ficou valendo dali em diante: se o item não passa no teste da coluna Localização do tracker, ele não entra.

### 3. Um número que se disfarçou de citação

O caso mais instrutivo apareceu no FDD, já na fase de revisão. A tabela de parâmetros configuráveis trazia a linha "tamanho do lote por ciclo: 50 eventos", com a origem apontando para [09:08] Diego. A fala existe e o falante está certo, mas Diego diz apenas "batch pequeno". O número 50 não veio da reunião, veio de plausibilidade.

Esse é o tipo de erro que a revisão por leitura não pega, porque a linha parece perfeitamente rastreável: tem timestamp válido, tem falante correto, tem uma fala real por trás. O que não tem é a informação específica que ela afirma. A linha foi corrigida para registrar o critério que a reunião de fato definiu, lote pequeno, deixando explícito que o valor exato é calibrado na implementação.

Vale como aviso geral sobre este tipo de documento: número redondo e plausível é a forma mais comum de invenção sobreviver a uma revisão.

### 4. O envelope de paginação escrito a partir do código

Ao escrever os contratos do FDD, a forma convencional de resposta paginada, com `items`, `page`, `pageSize` e `total` no mesmo nível, era a saída natural para quem escreve de memória. O código do projeto usa outro formato: o helper `paginated()` em [src/shared/http/response.ts](src/shared/http/response.ts) devolve `data` mais um objeto `pagination` aninhado, com `totalPages` calculado. Todos os contratos de listagem foram escritos a partir da leitura desse arquivo, não do formato esperado.

Na mesma passagem, outro detalhe foi separado para evitar erro de implementação: o limite de corpo de requisição da API é de 1mb, definido em [src/app.ts](src/app.ts), e não se confunde com o limite de 64KB do payload de evento, que é do fluxo de saída. Os dois números coexistem e significam coisas diferentes.

### 5. Contagens do tracker corrigidas por script

A seção de cobertura do tracker foi escrita inicialmente com números estimados de linhas por fonte. Ao rodar a verificação, os números reais eram outros, e a tabela foi corrigida. Foram então validadas por execução as 76 referências distintas de fala e os 23 caminhos de arquivo. Nenhum documento afirma um número de cobertura que não tenha sido contado.

### Ajustes menores ao longo do caminho

- Um bloco de "decisões secundárias que não sobem para o PRD" foi removido a pedido, por ser meta-comentário sobre o processo dentro de um documento de produto
- O esqueleto de FDD fornecido não previa seção de modelo de dados nem a seção obrigatória "Integração com o sistema existente". A primeira entrou como subseção de fluxos, a segunda como seção própria, e o desvio está declarado no documento
- Alternativas plausíveis mas não discutidas na reunião foram explicitamente marcadas como tal nos ADRs, para não passarem por deliberação do time

---

## Checklist dos critérios de aceite

Cada critério do [enunciado](DESAFIO.md) conferido contra a entrega, com o número real apurado. Os valores desta tabela foram contados por script, não estimados.

### PRD, [docs/PRD.md](docs/PRD.md)

- [x] Arquivo existe e está em Markdown
- [x] Contém todas as seções obrigatórias: resumo, contexto e problema, público-alvo e cenários de uso, objetivos e métricas, escopo, requisitos funcionais, requisitos não funcionais, arquitetura e abordagem, decisões e trade-offs, dependências, riscos e mitigação, critérios de aceitação, testes e validação
- [x] Mínimo de 8 requisitos funcionais: **11 requisitos**, RF-001 a RF-011, cada um com fluxo principal, fluxos alternativos, erros previstos e prioridade
- [x] Pelo menos 1 objetivo com métrica e meta quantitativa: **7 objetivos**, todos com métrica e meta, entre elas entrega abaixo de 10 segundos, 3 de 3 clientes migrados e zero chamada HTTP dentro da transação
- [x] "Fora de escopo" com pelo menos 2 itens descartados ou adiados na reunião: **8 itens**, todos com timestamp de origem no próprio PRD
- [x] "Riscos" com pelo menos 2 riscos com probabilidade, impacto e mitigação: **9 riscos**, todos com probabilidade, impacto, mitigação em subitens e plano de contingência

### RFC, [docs/RFC.md](docs/RFC.md)

- [x] Arquivo existe e está em Markdown
- [x] Contém todas as seções obrigatórias: metadados com autor, status, data e revisores, resumo executivo, contexto e problema, proposta técnica, alternativas consideradas, questões em aberto, impacto e riscos, decisões relacionadas
- [x] Revisores são os participantes da reunião: Marcos, Diego, Bruno e Sofia, com Larissa como autora
- [x] "Alternativas consideradas" com pelo menos 2 alternativas descartadas e o trade-off de cada uma: **6 alternativas**, entre elas disparo síncrono, fila externa, trigger de banco e exactly-once
- [x] "Questões em aberto" com pelo menos 2 pontos adiados ou não decididos: **5 questões**
- [x] Referencia com link pelo menos 2 ADRs: **os 7 ADRs**, mais o índice
- [x] Documento conciso, entre 2 e 4 páginas, sem descer ao detalhe de implementação do FDD: 2.275 palavras, sem DDL, sem payload de exemplo e sem matriz de erros

### FDD, [docs/FDD.md](docs/FDD.md)

- [x] Arquivo existe e está em Markdown
- [x] Contém todas as seções obrigatórias: contexto e motivação técnica, objetivos técnicos, escopo e exclusões, fluxos detalhados, contratos públicos, matriz de erros, estratégias de resiliência, observabilidade, dependências e compatibilidade, critérios de aceite técnicos, riscos e mitigação
- [x] Fluxos detalhados cobrindo criação do evento na outbox, processamento pelo worker, reenvio e Dead Letter: os quatro, mais rotação de secret e diagrama de sequência
- [x] "Contratos públicos" com pelo menos 4 endpoints HTTP com payload de exemplo e status codes: **7 endpoints HTTP**, mais o contrato da entrega ao cliente e a assinatura da função de publicação
- [x] Matriz de erros com códigos no prefixo `WEBHOOK_`: **11 códigos**, sendo 3 citados textualmente na reunião
- [x] "Integração com o sistema existente" com pelo menos 4 caminhos de arquivo reais: **16 arquivos**, todos verificados por script
- [x] "Observabilidade" cita métricas, logs e tracing: as três, com tracing por correlação de identificadores, já que o projeto não tem instrumentação distribuída
- [x] Critérios de aceite técnicos objetivos: **21 critérios** verificáveis por teste ou inspeção

### ADRs, [docs/adrs/](docs/adrs/)

- [x] Pasta contém entre 5 e 8 arquivos no formato `ADR-NNN-titulo-em-kebab-case.md`: **7 ADRs**, mais um `README.md` de índice
- [x] Cada ADR contém Status, Contexto, Decisão, Alternativas Consideradas e Consequências: **7 de 7**, verificado por script
- [x] Cobre pelo menos 5 das 6 decisões principais: **as 6**, outbox no MySQL, retry com backoff e DLQ, HMAC-SHA256 com secret por endpoint, at-least-once com identificador de evento, worker separado em polling e reuso dos padrões existentes. O sétimo ADR cobre o snapshot do payload
- [x] Pelo menos 1 ADR referencia arquivos, módulos ou padrões do código existente: **7 de 7**, cada um com tabela de referências de código. O ADR-006 sozinho referencia 15 arquivos
- [x] Alternativas consideradas com pelo menos 1 alternativa real por ADR: **21 alternativas** no conjunto, cada uma com o trade-off que motivou o descarte

### Tracker, [docs/TRACKER.md](docs/TRACKER.md)

- [x] Arquivo existe e segue o formato de tabela definido, com ID, Documento, Tipo, Conteúdo, Fonte e Localização
- [x] Pelo menos 80% dos itens identificáveis têm linha correspondente: **206 linhas**, cobrindo PRD, RFC, FDD e os 7 ADRs item a item
- [x] Pelo menos 70% das linhas com Fonte `TRANSCRICAO` e timestamp válido: **79%**, 163 de 206, com as 76 falas distintas conferidas por busca literal na transcrição
- [x] Pelo menos 5 linhas com Fonte `CODIGO` e caminho de arquivo real: **43 linhas**, cobrindo 23 arquivos distintos, todos existentes
- [x] Formato da localização segue `[hh:mm] Nome`: 100% das linhas de transcrição, sem intervalo e sem múltiplos falantes na mesma célula

### README, [README.md](README.md)

- [x] Contém todas as seções obrigatórias: sobre o desafio, ferramentas de IA, workflow adotado, prompts customizados, iterações e ajustes, como navegar a entrega
- [x] Pelo menos 1 ferramenta de IA utilizada: listada, com nota sobre o papel e sobre o que o contexto estendido permitiu no método
- [x] Pelo menos 2 prompts customizados em blocos de código: **4 prompts**
- [x] Pelo menos 2 iterações ou ajustes concretos: **5 iterações principais**, mais 3 ajustes menores

### Consistência geral

- [x] Nenhum requisito, decisão ou restrição contradiz a transcrição ou o código
- [x] Nenhum arquivo de código mencionado nos documentos é inexistente no repositório: verificado por script em todos os documentos
- [x] Código da aplicação não alterado: `git status` confirma que nada em `src/`, `prisma/` ou `tests/` foi modificado
- [x] Sem duplicação entre documentos: cada altura tratada uma única vez, conforme a [nota sobre a fronteira entre os documentos](#nota-sobre-a-fronteira-entre-os-documentos)

---

## Verificação final

Além da checagem por leitura, os itens abaixo foram conferidos rodando comandos sobre o repositório, porque revisão visual não pega erro de atribuição:

| Verificação | Resultado |
| --- | --- |
| Timestamps citados existem na transcrição, com o falante correto | 76 referências distintas, todas confirmadas por busca literal |
| Formato dos timestamps segue `[hh:mm] Nome` | 100% das linhas do tracker |
| Caminhos de arquivo citados existem no repositório | 23 arquivos distintos, todos confirmados |
| Links internos entre documentos | todos resolvem |
| Identificadores duplicados no tracker | nenhum |
| Seções obrigatórias por documento | conferidas uma a uma por script |
| Código da aplicação alterado | nenhum arquivo em `src/`, `prisma/` ou `tests/` foi modificado |

A verificação por execução foi o que encontrou os dois erros descritos nas iterações 3 e 5: um número sem origem disfarçado de citação e uma contagem de cobertura estimada em vez de contada.

---

## Como navegar a entrega

```
.
├── README.md                  este documento, o processo de produção
├── DESAFIO.md                 enunciado original, preservado para consulta
├── TRANSCRICAO.md             a fonte primária, não alterada
└── docs/
    ├── PRD.md                 por que e o quê, nível de produto
    ├── RFC.md                 proposta técnica submetida a revisão, nível de arquitetura
    ├── FDD.md                 como construir, nível de implementação
    ├── TRACKER.md             rastreabilidade de cada item à origem
    └── adrs/
        ├── README.md          índice e relação entre as decisões
        ├── ADR-001-padrao-outbox-no-mysql.md
        ├── ADR-002-worker-em-processo-separado-com-polling.md
        ├── ADR-003-retry-com-backoff-exponencial-e-dlq.md
        ├── ADR-004-hmac-sha256-com-secret-por-endpoint.md
        ├── ADR-005-entrega-at-least-once-com-x-event-id.md
        ├── ADR-006-reuso-dos-padroes-existentes-do-projeto.md
        └── ADR-007-snapshot-do-payload-na-insercao.md
```

### Ordem de leitura sugerida

1. **[TRANSCRICAO.md](TRANSCRICAO.md)**, se quiser conferir a fonte antes dos documentos. Não é obrigatório, o tracker aponta cada item de volta para ela
2. **[docs/PRD.md](docs/PRD.md)**, para entender o problema, o público, o escopo e o que ficou de fora
3. **[docs/RFC.md](docs/RFC.md)**, para a proposta técnica em nível de arquitetura, as alternativas descartadas e as cinco questões ainda em aberto
4. **[docs/adrs/README.md](docs/adrs/README.md)** e os sete ADRs, para o racional de cada decisão isolada, com o trade-off explícito
5. **[docs/FDD.md](docs/FDD.md)**, o documento mais longo, para o detalhe de implementação: modelo de dados, fluxos, contratos, matriz de erros e a seção de integração com o código existente
6. **[docs/TRACKER.md](docs/TRACKER.md)**, ao final ou em paralelo, para conferir a origem de qualquer item que gerar dúvida

### Se o tempo for curto

Leia o [PRD](docs/PRD.md) e a seção "Integração com o sistema existente" do [FDD](docs/FDD.md). Os dois juntos mostram o que a feature é e onde exatamente ela toca o sistema que já existe. Depois abra o [TRACKER](docs/TRACKER.md) e escolha três linhas ao acaso para conferir na transcrição.

### Para avaliar a entrega

O [checklist dos critérios de aceite](#checklist-dos-critérios-de-aceite) mapeia cada exigência do [enunciado](DESAFIO.md) ao número real apurado no pacote, com o caminho do documento correspondente. É o caminho mais curto para verificar cobertura sem precisar contar item por item.

### Nota sobre a fronteira entre os documentos

Os documentos não se repetem, e isso foi tratado como restrição de projeto. O PRD descreve a condição de erro, mas não o código de erro. O RFC apresenta a decisão e o trade-off, mas não o contrato do endpoint. O FDD traz payload, header e matriz de erros, mas não repete a justificativa da decisão, que está no ADR correspondente. Conteúdo duplicado entre documentos seria sinal de que algo está no lugar errado.
