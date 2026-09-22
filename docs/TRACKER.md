# Tracker de Rastreabilidade

Este documento mapeia cada item registrado nos documentos à sua origem na transcrição ou no código.

## PRD (Product Requirement Document)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-FR-01 | `docs/PRD.md` | Requisito Funcional | Cadastro de webhook via POST (URL e status) | `TRANSCRICAO` | `[09:31] Marcos`, `[09:32] Larissa` |
| PRD-FR-02 | `docs/PRD.md` | Requisito Funcional | Geração de secret por endpoint ao criar webhook | `TRANSCRICAO` | `[09:21] Sofia`, `[09:22] Sofia`, `[09:31] Marcos` |
| PRD-FR-03 | `docs/PRD.md` | Requisito Funcional | Edição de webhook via PATCH | `TRANSCRICAO` | `[09:33] Bruno` |
| PRD-FR-04 | `docs/PRD.md` | Requisito Funcional | Deleção (DELETE) e Listagem (GET) de webhooks | `TRANSCRICAO` | `[09:33] Bruno` |
| PRD-FR-05 | `docs/PRD.md` | Requisito Funcional | Rotação de secret com grace period de 24h | `TRANSCRICAO` | `[09:21] Sofia`, `[09:22] Sofia` |
| PRD-FR-06 | `docs/PRD.md` | Requisito Funcional | Filtro de eventos (status desejados pelo cliente) | `TRANSCRICAO` | `[09:33] Marcos`, `[09:34] Bruno` |
| PRD-FR-07 | `docs/PRD.md` | Requisito Funcional | Histórico de envios recentes (deliveries) | `TRANSCRICAO` | `[09:34] Marcos` |
| PRD-FR-08 | `docs/PRD.md` | Requisito Funcional | Conteúdo do payload gerado como snapshot | `TRANSCRICAO` | `[09:43] Diego`, `[09:52] Larissa`, `[09:52] Diego` |
| PRD-FR-09 | `docs/PRD.md` | Requisito Funcional | Retry policy com 5 tentativas e backoff | `TRANSCRICAO` | `[09:15] Diego`, `[09:17] Diego`, `[09:17] Larissa` |
| PRD-FR-10 | `docs/PRD.md` | Requisito Funcional | Replay manual de DLQ para administradores | `TRANSCRICAO` | `[09:18] Diego`, `[09:35] Diego`, `[09:36] Sofia`, `[09:36] Larissa` |
| PRD-NFR-01 | `docs/PRD.md` | Requisito Não Funcional | Latência alvo inferior a 10s para notificação | `TRANSCRICAO` | `[09:02] Marcos`, `[09:09] Diego`, `[09:10] Larissa` |
| PRD-NFR-02 | `docs/PRD.md` | Requisito Não Funcional | Segurança com HMAC-SHA256 e HTTPS | `TRANSCRICAO` | `[09:20] Sofia`, `[09:23] Sofia` |
| PRD-NFR-03 | `docs/PRD.md` | Requisito Não Funcional | Limite de tamanho de payload em 64KB | `TRANSCRICAO` | `[09:23] Sofia`, `[09:24] Diego`, `[09:24] Larissa` |
| PRD-NFR-04 | `docs/PRD.md` | Requisito Não Funcional | Resiliência at-least-once com header X-Event-Id | `TRANSCRICAO` | `[09:24] Diego`, `[09:25] Diego`, `[09:26] Larissa` |
| PRD-SCOPE-01 | `docs/PRD.md` | Restrição | Descartado: disparo síncrono no service de orders | `TRANSCRICAO` | `[09:04] Bruno`, `[09:04] Larissa`, `[09:06] Diego` |
| PRD-SCOPE-02 | `docs/PRD.md` | Restrição | Adiado: e-mail alertando sobre falhas repetidas | `TRANSCRICAO` | `[09:37] Marcos`, `[09:37] Larissa` |
| PRD-SCOPE-03 | `docs/PRD.md` | Restrição | Adiado: dashboard visual para webhooks | `TRANSCRICAO` | `[09:39] Marcos`, `[09:40] Larissa` |
| PRD-SCOPE-04 | `docs/PRD.md` | Restrição | Adiado: rate limiting de envios por parte do OMS | `TRANSCRICAO` | `[09:38] Diego`, `[09:39] Larissa` |
| PRD-SCOPE-05 | `docs/PRD.md` | Restrição | Descartado: ordering global (garantia de ordem) | `TRANSCRICAO` | `[09:12] Larissa`, `[09:12] Diego`, `[09:13] Larissa` |

## RFC (Request for Comments)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| RFC-ALT-01 | `docs/RFC.md` | Trade-off | Alternativa descartada: Disparo síncrono no OrderService | `TRANSCRICAO` | `[09:04] Bruno`, `[09:04] Larissa` |
| RFC-ALT-02 | `docs/RFC.md` | Trade-off | Alternativa descartada: Infraestrutura dedicada de Stream | `TRANSCRICAO` | `[09:07] Larissa`, `[09:07] Diego` |
| RFC-ALT-03 | `docs/RFC.md` | Trade-off | Alternativa descartada: Trigger no banco de dados (MySQL) | `TRANSCRICAO` | `[09:09] Bruno`, `[09:09] Diego` |
| RFC-OPEN-01 | `docs/RFC.md` | Restrição | Em aberto: Rate limiting do envio aos clientes | `TRANSCRICAO` | `[09:38] Diego`, `[09:39] Larissa` |
| RFC-OPEN-02 | `docs/RFC.md` | Restrição | Em aberto: Alerta via email em falhas de webhook | `TRANSCRICAO` | `[09:37] Marcos`, `[09:37] Larissa` |
| RFC-DEC-01 | `docs/RFC.md` | Decisão | Uso do Outbox Pattern integrado ao MySQL | `TRANSCRICAO` | `[09:06] Diego`, `[09:08] Larissa` |
| RFC-DEC-02 | `docs/RFC.md` | Decisão | Separação do Worker em processo de polling | `TRANSCRICAO` | `[09:09] Diego`, `[09:11] Larissa`, `[09:11] Diego` |

## ADRs (Architecture Decision Records)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-001 | `docs/adrs/ADR-001-padrao-outbox-mysql.md` | Decisão | Padrão Outbox no MySQL acoplado ao OrderService | `TRANSCRICAO` | `[09:06] Diego`, `[09:08] Larissa` |
| ADR-001-C | `docs/adrs/ADR-001-padrao-outbox-mysql.md` | Decisão | Padrão Outbox no MySQL acoplado ao OrderService | `CODIGO` | `src/modules/orders/order.service.ts` |
| ADR-002 | `docs/adrs/ADR-002-politica-retry-dlq.md` | Decisão | Política de retry com backoff e DLQ | `TRANSCRICAO` | `[09:15] Diego`, `[09:17] Larissa`, `[09:18] Diego` |
| ADR-003 | `docs/adrs/ADR-003-autenticacao-hmac-sha256.md` | Decisão | Autenticação HMAC-SHA256 com secret por endpoint | `TRANSCRICAO` | `[09:20] Sofia`, `[09:22] Sofia` |
| ADR-004 | `docs/adrs/ADR-004-garantia-at-least-once.md` | Decisão | Garantia at-least-once com X-Event-Id | `TRANSCRICAO` | `[09:24] Diego`, `[09:25] Diego`, `[09:26] Larissa` |
| ADR-005 | `docs/adrs/ADR-005-worker-em-polling.md` | Decisão | Worker em processo separado em polling | `TRANSCRICAO` | `[09:09] Diego`, `[09:10] Larissa`, `[09:11] Diego` |
| ADR-006 | `docs/adrs/ADR-006-reuso-padroes-projeto.md` | Decisão | Reuso dos padrões existentes do projeto | `TRANSCRICAO` | `[09:28] Bruno`, `[09:29] Bruno`, `[09:30] Larissa` |
| ADR-006-C1 | `docs/adrs/ADR-006-reuso-padroes-projeto.md` | Decisão | Reuso de AppError e padrões de projeto | `CODIGO` | `src/shared/errors/` |
| ADR-006-C2 | `docs/adrs/ADR-006-reuso-padroes-projeto.md` | Decisão | Reuso de middleware de erro existente | `CODIGO` | `src/middlewares/error.middleware.ts` |
| ADR-006-C3 | `docs/adrs/ADR-006-reuso-padroes-projeto.md` | Decisão | Reuso do Logger padrão da aplicação | `CODIGO` | `src/shared/logger/` |
