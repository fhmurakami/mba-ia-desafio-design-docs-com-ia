# Feature Design Document (FDD) - Sistema de Webhooks

## Contexto e Motivação Técnica
Os clientes corporativos B2B necessitam receber notificações instantâneas sobre mudanças nos estados de seus pedidos. Hoje eles sobrecarregam a API principal com requisições repetitivas de leitura (polling) tentando identificar atualizações. O sistema de Webhooks resolverá este gargalo repassando a responsabilidade de *push* da notificação para a nossa infraestrutura.

## Objetivos Técnicos
- Construir um módulo de `webhooks` independente do *core* de *orders*, minimizando acoplamento funcional.
- Escoar eventos de status de pedido em menos de 10 segundos para a rede externa de forma resiliente.
- Viabilizar retry exponencial automatizado sem sacrificar processamento síncrono.

## Escopo e Exclusões
**No escopo:** Tabela outbox e DLQ em MySQL; Worker desacoplado em processo separado realizando *polling*; Autenticação HMAC-SHA256; API CRUD de configurações.
**Fora de escopo (Exclusões):** Processamento "Exactly-once" (assumimos "At-least-once"); Emails transacionais alertando falhas; Rate limiting estrito para a taxa de saída.

## Fluxos Detalhados
1. **Criação do evento na Outbox**: O `OrderService` realiza a troca de status de um pedido. No mesmo objeto de transação do Prisma, a função de persistência do webhook avalia se há webhooks interessados naquele `customer_id` e `status`. Se houver, serializa o snapshot (JSON) do pedido com o `X-Event-Id` gerado e commita na tabela `webhook_outbox` como `PENDENTE`.
2. **Processamento pelo Worker**: O script `src/worker.ts` acorda a cada 2 segundos. Consulta a tabela filtrando por eventos pendentes. Ele obtém a `secret`, calcula a hash HMAC-SHA256 sobre o payload, e dispara a requisição HTTPS com timeout de 10 segundos.
3. **Retry e DLQ**: Se o webhook responder com sucesso (2xx), o evento é marcado como `DELIVERED`. Se falhar (timeout ou 5xx), a data de `next_attempt` é calculada com base na progressão (1m, 5m, 30m, 2h, 12h) e a contagem de falhas aumenta. Na 5ª falha, é inserido em `webhook_dead_letter` e apagado da fila principal.

## Contratos Públicos

### 1. Criar Webhook (POST /webhooks)
Cria o destino de notificação e retorna a *secret* para o cliente.
- **Request:**
```json
{
  "url": "https://cliente.com/webhook",
  "events": ["SHIPPED", "DELIVERED"]
}
```
- **Response (201 Created):**
```json
{
  "id": "wh_12345",
  "url": "https://cliente.com/webhook",
  "events": ["SHIPPED", "DELIVERED"],
  "secret": "whsec_ABCDEF1234567890",
  "active": true
}
```

### 2. Listar Webhooks do Cliente (GET /webhooks)
- **Response (200 OK):**
```json
{
  "data": [
    {
      "id": "wh_12345",
      "url": "https://cliente.com/webhook",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true
    }
  ]
}
```

### 3. Histórico de Entregas (GET /webhooks/:id/deliveries)
- **Response (200 OK):**
```json
{
  "data": [
    {
      "event_id": "evt_9876",
      "status": "DELIVERED",
      "response_code": 200,
      "payload": { "order_id": "ord_111", "to_status": "SHIPPED" },
      "delivered_at": "2026-09-22T10:00:00Z"
    }
  ]
}
```

### 4. Replay Administrativo da DLQ (POST /admin/webhooks/dead-letter/:id/replay)
- **Headers:** `Authorization: Bearer <token_admin>` (Requer role `ADMIN`)
- **Request:** (Vazio)
- **Response (202 Accepted):**
```json
{
  "message": "Evento re-enfileirado com sucesso."
}
```

## Matriz de Erros (WEBHOOK_*)
| Código HTTP | Erro Customizado | Motivo |
|---|---|---|
| 404 | `WEBHOOK_NOT_FOUND` | Webhook não pertence ao cliente logado ou não existe. |
| 400 | `WEBHOOK_INVALID_URL` | A URL fornecida não obedece padrão seguro (HTTPS). |
| 413 | `WEBHOOK_MAX_SIZE_EXCEEDED` | Evento excede 64KB de payload máximo permitido. |
| 403 | `WEBHOOK_SECRET_REQUIRED` | Rotação requerida mas dados insuficientes. |

## Estratégias de Resiliência
* **Timeouts:** Restrito a 10s no disparo pelo worker. Protege as *threads* de pendurarem indefinidamente aguardando *sockets* lentos do lado parceiro.
* **Retries & Backoff:** Proteção de tempestade de requisições, alongando o intervalo entre falhas de 1 min para 12 horas, desistindo para proteção de disco.

## Observabilidade
* **Métricas:** Volume de enfileiramento da `webhook_outbox`, contagem de retries e envios caídos para a DLQ (`webhook_dead_letter`).
* **Logs:** Registro textual via `Pino` das transições de retry, erros severos no worker e re-acionamento manual da DLQ pelo endpoint de auditoria.
* **Tracing:** Repasse do `X-Event-Id` rastreável em todo o pipeline.

## Dependências e Compatibilidade
O Worker necessita conectar na mesma base do Prisma MySQL e carregar instâncias de logging idênticas da API. Necessidade imperativa de garantir rotina de expiração na tabela Outbox futuramente.

## Integração com o sistema existente
1. `src/modules/orders/order.service.ts`: O método transacional `changeStatus` receberá o acoplamento da rotina auxiliar `publishWebhookEvent(tx, order)` para realizar a escrita conjunta do snapshot na outbox.
2. `src/shared/errors/app-error.ts`: A classe padrão `AppError` será importada e instanciada pela lógica do domínio de webhook para invocar a matriz descrita (ex: `throw new AppError('Webhook não encontrado', 404, 'WEBHOOK_NOT_FOUND')`).
3. `src/middlewares/error.middleware.ts`: Graças ao uso do `AppError`, o *Error Middleware* central da aplicação formatará e responderá 4xx/5xx sem requerer absolutamente nenhuma alteração para o CRUD de Webhooks.
4. `src/shared/logger/index.ts`: Importaremos o gerenciador base do logger (`Pino`) dentro do `src/worker.ts` para uniformizar a saída do terminal/container log, espelhando fielmente os metadados da API normal.

## Critérios de Aceite Técnicos
- [ ] O disparo de webhook possui header assinado com `X-Signature` HMAC-SHA256 validável.
- [ ] O evento de notificação transita com a carga UUID no `X-Event-Id` sem distorção.
- [ ] Tentar chamar `POST /admin/webhooks/...` sem token gerado por `role: ADMIN` retorna falha imediata.
- [ ] Uma mudança no pedido refletida pelo `changeStatus` trava (faz *rollback*) simultaneamente se a outbox de inserção falhar (integridade transacional).

## Riscos e Mitigação
* **Risco**: Timeout de rede entre o banco MySQL e o Worker rodando distante.
* **Mitigação**: O worker usará pooling de conexão e terá *healthchecks* com a persistência de Docker reiniciando-o na falha de I/O contínua.
