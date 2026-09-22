# ADR 005: Worker Desacoplado Operando em Polling

## Status
Aceito

## Contexto
Com a eleição do Padrão *Outbox* (ADR-001) que armazena os gatilhos em uma tabela de eventos interna, torna-se necessário possuir um maestro sistêmico, que acorde periodicamente ou ativamente para descarregar o *buffer* pendente do MySQL na internet para o sistema do cliente.

## Decisão
Nossa deliberação estabelece a codificação de um script autônomo operando um laço ininterrupto de **Polling a cada 2 segundos**, provisionado como um processo Node Isolado (`src/worker.ts`), desconectado do serviço das rotas da API (`src/server.ts`).

## Alternativas Consideradas
* **Operar o Polling em paralelo no Processo Node principal (API)**: Embutir `setInterval` injetado na `server.ts`. Descartado firmemente. Operar lógicas de bloqueio HTTP I/O em filas na mesma base que serve requisições rest API impõe travamentos nocivos da *Event Loop* Node. Além disso, escalonamentos (aumento de pods k8s da web API) forçaria uma corrida insana de dezenas de workers batendo simultaneamente no *buffer* do BD sem *locks*.
* **DB Triggers nativas para Acionar Sub-processos**: Tratar a tabela de Outbox sob MySQL para instigar um evento de notificação real (usando binlogs nativos, por exemplo). Triggers relacionais p/ funções externas no MySQL limitam-se severamente e adicionariam lixo processual, dificultando depuração; rejeitado em comparação ao polling eficiente.

## Consequências
* **Positivas**: Total imutabilidade da base da aplicação que continuará isolada e operando fluidamente. Polling em 2s assegura a latência restrita do SLO exigido (<10s).
* **Negativas**: Será introduzida na etapa de CI/CD uma nova imagem/script (`npm run worker`), com a dependência de partilhar credenciais iguais, orquestrada como um nó separado da esteira atual do projeto.
