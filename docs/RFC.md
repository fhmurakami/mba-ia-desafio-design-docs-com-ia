# RFC: Sistema de Webhooks de Notificação de Pedidos

## Metadados
- **Autor:** Equipe de Arquitetura e Engenharia
- **Status:** Proposto / Em Revisão
- **Data:** 17/09/2026
- **Revisores:** Larissa (Tech Lead), Marcos (Product Manager), Bruno (Engenheiro Pleno), Diego (Engenheiro Sênior), Sofia (Engenheira de Segurança)

## Resumo Executivo (TL;DR)
Esta RFC propõe a arquitetura para um sistema de Webhooks *outbound* projetado para notificar os clientes B2B (ex: Atlas Comercial, MaxDistribuição) em tempo real quando o status dos pedidos deles alterar. A solução baseia-se no padrão **Outbox Pattern persistido via MySQL** e integrado à transação principal do serviço de pedidos. O processamento assíncrono ficará a cargo de um **Worker Node isolado** executando *polling*, o qual fornecerá envio com semântica *at-least-once*, validações criptográficas (`HMAC-SHA256`), mecanismo elástico de tentativas (`Retry` exponencial) e tratamento robusto para retidos definitivos (`Dead Letter Queue`).

## Contexto e Problema
Clientes cruciais para a plataforma necessitam de atualização ágil (inferior a 10s) sobre a mudança de status de pedidos. Dada a ausência de push notifications, a tática adotada pelos clientes atualmente é efetuar requisições redundantes de *polling* nos endpoints da nossa API (`GET /orders`). Este paradigma deteriora a performance de leitura do banco de dados, amplia os custos e gera latências consideráveis de notificação comercial que comprometeram a permanência de um dos parceiros que considerou migrar para a concorrência se a experiência de integração não for reavaliada neste trimestre.

## Proposta Técnica
A estrutura sugere acoplar de forma coesa a notificação ao fluxo transacional já estabelecido pelo serviço `OrderService`, sem corromper as normativas de negócio e com reuso intensivo de padrões já maduros (`Pino` Logger, `Zod`, `AppError` e o middleware unificado):
1. **Transação Atômica c/ Outbox (MySQL):** A cada atualização (`changeStatus`), um snapshot completo e formatado do evento de mudança será persistido numa tabela colateral (`webhook_outbox`) aproveitando a exata mesma conexão e bloco `TX` do banco. Se não registrar o pedido não muda de status (atomicidade).
2. **Desacoplamento por Worker Isolado:** Ao invés da API lidar com disparo HTTP que impõe bloqueio, um script em processo separado da VM hospedeira (`src/worker.ts`) rodará uma tarefa iterativa de *polling* a cada 2 segundos vasculhando os eventos passíveis de processamento no DB, com ordenação via `created_at`.
3. **Segurança do End-point do Cliente:** Resguardar a confiança de cada push exigindo URLs autenticadas por certificado digital (`HTTPS`), injetando a hash resultante no header `X-Signature` via validação simétrica por uma `secret` de rotatividade administrada isoladamente por cada URL destino pelo cliente.
4. **Retry e Isolamento (DLQ):** Diante de ofensores momentâneos (queda do parceiro ou recusa 5xx/timeout na janela de 10s), eventos sofrerão enfileiramento por acréscimo temporal gradual (`backoff`: 1m, 5m, 30m, 2h, 12h, num máximo de 5 chances) antes de se caducarem na área de `Dead Letter`, manipulável exclusivamente por suporte administrativo capacitado.
5. **Deduplicação de Eventos:** Cada requisição veiculará um header `X-Event-Id` injetando UUID único na geração da transação; permitindo lidar com cenários intermitentes onde confirmaremos os retries assegurando retransmissão de entrega "pelo menos uma vez", isolando complexidades custosas de "exatamente uma".

## Alternativas Consideradas
* **Alternativa 1: Chamada HTTP síncrona dentro da pipeline do Pedido (`OrderService`)**
  * **Trade-off e Descarte:** Exigiria efetuar um fetch web bloqueante na transação crítica do banco. Qualquer latência excessiva ou falha do cliente causaria engasgos na transação, esgotaria conexões e exigiria cancelamento sistêmico de progressão de um pedido por falha no elo de notificação que nada compõe no domínio logístico; prontamente descartado.
* **Alternativa 2: Adoção de Infraestrutura de Stream (RabbitMQ / Redis PubSub / Kafka)**
  * **Trade-off e Descarte:** Traria overhead operacional incompatível para a atual estrutura pequena da equipe. Introduzir e sustentar a manutenção de outro subsistema, quando a latência de negócio demandada é atendida confortavelmente pela robustez transacional local do MySQL (com indexação seletiva), foi evitado para contornar desperdício e *overengineering*.
* **Alternativa 3: Notificação Acoplada por DB Triggers ativas**
  * **Trade-off e Descarte:** Explorada a ideia de disparar reações autônomas por procedures em banco. Descartou-se diante da limitação tecnológica do motor adotado (MySQL carece de listener nativo análogo ao pg_notify do Postgres) o que exigiria subterfúgios pesados para notificar a VM do worker (logs em arquivos, sys exec functions).

## Questões em Aberto
1. **Regulação de Volume em Massa de Saída (Rate Limiting de Notificações):** Não impomos artificialmente um limite de quantos POST requests faremos por segundo no envio ao cliente caso diversos pedidos finalizem abruptamente, mas mantemos o timeout restritivo. A decisão foi manter essa implementação latente para acompanhar o reflexo natural dessa volumetria primeiro.
2. **Alertas Preventivos (e.g. Disparo de e-mails para Falhas de DLQ):** Levou-se à discussão, a automatização de comunicados reativos por correio eletrônico que advertisse aos usuários sobre interrupções graves prolongadas nos seus endpoints. Conforme a limitação do backlog, esta implementação de monitoramento avançado fora postergada e ficará pendente até reanálise das adoções na "v2".

## Impacto e Riscos
* **Concorrência Operacional:** A opção de *polling* intensivo gera batidas recorrentes ao MySQL. Contudo, ao criar um índice de composição entre os filtros `status = PENDENTE` ordenados temporalmente e limitados a pequenos *batches*, a complexidade da query aproxima-se e garante I/O desprezível sobrecarregando em proporção ao que trará alívio do ofensor de leitura da versão atual da API.
* **Complexidade no Roteamento Cliente-side:** A migração de "pegar ativamente a resposta" para "receber assincronamente requisições e aplicar dedup" transfere a competência de concorrência e armazenamento de chaves de estado, mas como todo player corporativo do mercado (Webhooks via HMAC com UUID Identificadores), os envolvidos no pleito absorvem e comumente dispõem destes hardwares.

## Decisões Relacionadas
As seguintes ADRs (Architecture Decision Records) aprofundarão individualmente e pormenorizadamente a engenharia das escolhas consolidadas por esta proposta:
- [ADR-001-padrao-outbox-mysql.md](adrs/ADR-001-padrao-outbox-mysql.md)
- [ADR-002-politica-retry-dlq.md](adrs/ADR-002-politica-retry-dlq.md)
- [ADR-003-autenticacao-hmac-sha256.md](adrs/ADR-003-autenticacao-hmac-sha256.md)
- [ADR-004-garantia-at-least-once.md](adrs/ADR-004-garantia-at-least-once.md)
- [ADR-005-worker-em-polling.md](adrs/ADR-005-worker-em-polling.md)
- [ADR-006-reuso-padroes-projeto.md](adrs/ADR-006-reuso-padroes-projeto.md)
