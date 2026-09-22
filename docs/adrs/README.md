# Architecture Decision Records (ADRs)

Este diretório contém os registros das decisões arquiteturais (ADRs) tomadas para a implementação do Sistema de Webhooks de Notificação de Pedidos.

Cada ADR segue uma estrutura padronizada (inspirada no modelo MADR) documentando:
- O contexto e problema;
- A decisão final;
- As alternativas que foram consideradas;
- As consequências (positivas e negativas) da decisão adotada.

## Índice de ADRs
- [ADR-001 - Padrão Outbox no MySQL](ADR-001-padrao-outbox-mysql.md)
- [ADR-002 - Política de Retry Exponencial e DLQ Isolada](ADR-002-politica-retry-dlq.md)
- [ADR-003 - Autenticação de Mensagem com HMAC-SHA256](ADR-003-autenticacao-hmac-sha256.md)
- [ADR-004 - Garantia de Entrega At-Least-Once](ADR-004-garantia-at-least-once.md)
- [ADR-005 - Worker Desacoplado Operando em Polling](ADR-005-worker-em-polling.md)
- [ADR-006 - Reuso de Padrões Base do Projeto (AppError, Pino, Middlewares)](ADR-006-reuso-padroes-projeto.md)
