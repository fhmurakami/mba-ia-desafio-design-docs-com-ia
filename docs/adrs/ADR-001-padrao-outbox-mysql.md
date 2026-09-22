# ADR 001: Padrão Transactional Outbox no MySQL

## Status
Aceito

## Contexto
O processo atual de mudança de status (`changeStatus`) no serviço `OrderService` (localizado em `src/modules/orders/order.service.ts`) envolve uma transação de banco de dados complexa, que atualiza a tabela do pedido, insere histórico (`order_status_history`) e altera saldo de estoque. Precisamos notificar sistemas de clientes de forma segura assim que a mudança ocorrer, com latência inferior a 10s.

## Decisão
Decidimos adotar o **Transactional Outbox Pattern** no banco de dados MySQL já utilizado pela aplicação. O método `changeStatus` executará uma rotina auxiliar para injetar o evento (payload snapshot completo) dentro da mesma transação na tabela auxiliar `webhook_outbox`.

## Alternativas Consideradas
* **Disparo HTTP Síncrono no `OrderService`**: Inserir a chamada de disparo para o cliente em série no fluxo do pedido. Descartado, pois gargalaria o I/O da API (deixando clientes travados esperando a resposta HTTP de terceiros) e tornaria o controle da transação relacional instável, podendo causar rollback acidental se o webhook do cliente travar.
* **Stream de Mensageria (Redis ou RabbitMQ)**: Descartado por ser *overengineering* para o escopo e orçamento da equipe. O volume de requisições pode ser perfeitamente amparado pela ACID relacional do MySQL com a devida indexação (`status`, `created_at`).

## Consequências
* **Positivas**: Máxima garantia de entrega (*dual-write* evitado). A atômica do banco garante que um webhook não será enfileirado caso o pedido sofra um `rollback` nos estoques ou similar.
* **Negativas**: Aumento constante no tamanho das tabelas, forçando uma limpeza (arquivamento de webhooks entregues após 30 dias) em etapas de backlog futuro. Necessidade de provisionar o worker de disparo assíncrono à parte.
