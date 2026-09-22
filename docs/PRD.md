# Product Requirement Document (PRD) - Sistema de Webhooks de Notificação de Pedidos

## Resumo e Contexto da Feature
Atualmente, o nosso Order Management System (OMS) processa pedidos de ponta a ponta. No entanto, não há nenhum mecanismo de notificação ativa (push) sobre as mudanças de estado dos pedidos. Nossos clientes B2B (como Atlas Comercial, MaxDistribuição e Nova Cargo) precisam ser notificados quando o status dos pedidos muda. Hoje, para suprir essa necessidade, eles fazem *polling* constante na API (chamadas repetidas de `GET /orders`), consumindo recursos desnecessariamente e recebendo as atualizações de forma não-ótima. O desenvolvimento dessa feature introduzirá um sistema de webhooks outbound (saindo do nosso sistema para os clientes) para entregar atualizações de pedidos em "tempo real" (menos de 10 segundos).

## Problema e Motivação
A abordagem de *polling* sobrecarrega a API de leitura e oferece uma experiência de baixa velocidade para os parceiros. A latência atual está insatisfatória e gerando insatisfação comercial a ponto da Atlas Comercial ameaçar migrar para a concorrência caso uma solução de notificação em tempo real não seja entregue até o fim do trimestre.

## Público-alvo e Cenários de Uso
* **Público-alvo:** Clientes B2B e parceiros comerciais integrados à nossa plataforma, bem como seus desenvolvedores que consomem nossa API.
* **Cenários de Uso:** 
  * O sistema B2B do cliente escuta ativamente notificações do OMS. Assim que um pedido muda para `SHIPPED`, o webhook dispara, o sistema do cliente é notificado e dispara o faturamento ou as operações logísticas de forma imediata na sua ponta.

## Objetivos e Métricas de Sucesso
* **Objetivo de Negócio:** Reter clientes-chave (Atlas Comercial, MaxDistribuição) garantindo a entrega do mecanismo no prazo de três sprints.
* **Objetivo Quantitativo e Meta:** Reduzir a carga de requisições de *polling* nos endpoints `GET /orders` por parte dos clientes B2B em, pelo menos, **80% em até 2 meses após o lançamento**.
* **Objetivo Técnico:** Assegurar que os webhooks sejam disparados em menos de 10 segundos da alteração do status.

## Escopo

### O que está no escopo (Incluso)
* Envio de webhooks outbound baseados na mudança de status de pedidos.
* CRUD (API) para o cliente gerenciar suas configurações de webhook (URL, secret, status desejados).
* Mecanismo de reenvio de falhas (retry) automático e fila de falhas permanentes (DLQ) com reprocessamento manual via endpoint administrativo.
* Autenticação das chamadas de webhook via HMAC-SHA256 para o cliente validar a proveniência dos dados.

### O que está fora do escopo (Descartado ou Adiado)
* **[Descartado] Disparo síncrono no fluxo do pedido:** A notificação não vai travar ou atrasar a transação do banco de dados na mudança de pedido. (Usaremos padrão *Outbox* assíncrono).
* **[Adiado] Envio de e-mail de alerta de falha:** O envio de e-mails para os clientes caso o webhook deles falhe várias vezes consecutivas foi adiado para uma próxima fase.
* **[Adiado] Dashboard visual de gerenciamento de Webhooks:** Todo o gerenciamento inicial será feito exclusivamente via API. Interfaces web para configuração ficam fora deste escopo e sob responsabilidade futura do time de Frontend.
* **[Adiado] Rate limiting na saída das chamadas:** Inicialmente, enviaremos webhooks conforme o volume da fila, sem rate limit do nosso lado para evitar sufocar o cliente. Fica para futura observação.
* **[Descartado] Garantia de ordem global (Ordering global) e Exactly-once:** Apenas `at-least-once` é garantido. Deduplicação fica a cargo do cliente via ID de evento.

## Requisitos Funcionais
* **[PRD-FR-01] Cadastro de Webhook:** Clientes devem poder cadastrar URLs de webhooks (HTTPS) e informar uma lista de status desejados para notificação via requisição `POST` autenticada na nossa API.
* **[PRD-FR-02] Geração de Secret:** Ao criar um webhook, o sistema deve gerar e retornar uma `secret` de autenticação única associada exclusivamente àquele endpoint.
* **[PRD-FR-03] Edição de Webhook:** Clientes devem poder atualizar suas configurações de webhook (ex: URL ou eventos ouvidos) via requisição `PATCH`.
* **[PRD-FR-04] Deleção e Listagem:** Clientes devem poder remover (`DELETE`) seus webhooks e listar (`GET`) as configurações ativas que possuem.
* **[PRD-FR-05] Rotação de Secret:** O sistema deve suportar a rotação da `secret` por parte do cliente, onde a secret antiga e a nova coexistem válidas durante um *grace period* de 24 horas antes da expiração definitiva da antiga.
* **[PRD-FR-06] Filtro de Eventos:** O sistema só deve gerar e enviar notificações de status de pedidos correspondentes aos eventos que o cliente explicitamente se inscreveu ao cadastrar o webhook.
* **[PRD-FR-07] Histórico de Envios (Deliveries):** Clientes devem poder visualizar o histórico de envios recentes do webhook (status sucesso/falha, payload enviado, response recebido e tempo) via requisição `GET /webhooks/:id/deliveries`.
* **[PRD-FR-08] Conteúdo do Payload:** O payload JSON enviado ao cliente deve ser um snapshot (gerado na inserção da transação) contendo: `event_id`, tipo de evento, timestamp (ISO 8601), `order_id`, `order_number`, status origem, status destino, total da compra e `customer_id`. (Itens do pedido não são incluídos).
* **[PRD-FR-09] Retry Policy:** Eventos falhos (cliente offline ou timeout) devem ser automaticamente retentados até 5 vezes utilizando um *backoff* exponencial de: 1min, 5min, 30min, 2h e 12h.
* **[PRD-FR-10] Replay Manual de DLQ:** Após esgotar o limite de tentativas (5 falhas), o evento deve ir para uma Dead Letter Queue. Administradores devem poder acionar o reprocessamento manual de eventos da DLQ via um endpoint `POST /admin/webhooks/dead-letter/:id/replay` restrito por role `ADMIN`.

## Requisitos Não Funcionais
* **Performance / Latência:** O disparo deve ocorrer num máximo de 10 segundos a partir do evento gerador, com um mecanismo de polling no banco de dados ocorrendo a cada 2 segundos.
* **Segurança da Comunicação:** Toda comunicação do webhook obriga TLS (`https://`). Além disso, os payloads devem ser assinados pelo servidor gerando um hash no header `X-Signature` usando a técnica HMAC com algoritmo SHA-256.
* **Limites de Payload e Timeouts:** O payload da requisição tem um limite máximo e não deve exceder 64KB. Timeout para requisição HTTP ao cliente é de estritos 10 segundos.
* **Resiliência de Envio (At-Least-Once):** O sistema garante que cada notificação seja entregue ao menos uma vez. O cliente deve lidar com potenciais requisições duplicadas via header `X-Event-Id` (UUID único).
* **Auditoria:** O acesso e acionamento ao endpoint de replay administrativo (`/admin/webhooks/...`) deve gerar registro de auditoria, marcando quem fez o reenvio.

## Decisões e Trade-offs Principais
* **Outbox Pattern no MySQL (Trade-off):** Adotou-se o MySQL por ser a infraestrutura atual, evitando trazer a complexidade e o custo de novos serviços (Kafka, Redis Streams). Em contrapartida, faremos *polling* a cada 2s sobre a tabela.
* **At-Least-Once vs Exactly-Once:** Escolheu-se repassar a responsabilidade de deduplicação ao cliente com um ID (`X-Event-Id`), trocando complexidade extrema no nosso serviço por uma prática padrão e acessível de mercado.
* **Processamento de Retry Finito:** Ao limitar em 5 tentativas máximas (espalhadas em ~15h), evitamos gargalos de eventos inviáveis trancando a fila eternamente. Se um cliente ficar offline por mais que isso, dependerá da intervenção manual via API de entregas/DLQ.

## Dependências
* Funcionalidade profundamente acoplada à atual transação do método `changeStatus` do `OrderService`.
* O Módulo de Segurança e permissões com o JWT, para a role `ADMIN`.

## Riscos e Mitigação
1. **Risco:** O cliente vazar a `secret` da aplicação dele inadvertidamente (ex: em logs expostos) e sofrer de adulteração/ataques.
   * **Probabilidade:** Média
   * **Impacto:** Alto
   * **Mitigação:** Arquitetura de `secret` gerada unicamente por endpoint de webhook, permitindo revogação isolada. Criação de endpoint para o próprio cliente iniciar a rotação de secret (grace period de 24h).
2. **Risco:** A chamada síncrona dentro da transação atual do pedido falhar e impactar todo o sistema de processamento de compras do core OMS.
   * **Probabilidade:** Alta (caso ocorra um deslize na arquitetura)
   * **Impacto:** Crítico
   * **Mitigação:** Utilização irrestrita do Padrão Outbox. A comunicação externa deve ser apartada das APIs do Core. O `OrderService` somente persiste no mesmo banco relacional um registro atômico; um `Worker` independente lidará com o tráfego HTTP.
3. **Risco:** Falha de cliente contínua que resulte num volume extremo de retenção em fila.
   * **Probabilidade:** Baixa / Média.
   * **Impacto:** Moderado.
   * **Mitigação:** Política de Backoff agressiva que escalona para até 12h entre a última tentativa, somada ao Timeout restrito a 10 segundos, para não estrangular nosso Worker.

## Critérios de Aceitação
* O cliente consegue criar, atualizar, listar, deletar um webhook validado em domínio https.
* A cada configuração, o cliente recebe uma secret de HMAC e, nas chamadas que seu servidor recebe, a validação bate perfeitamente.
* Se os servidores do cliente estiverem indisponíveis na primeira chamada, é possível observar no banco a transição sequencial com tempo aguardando segundo a política de retries definida até cair em DLQ.
* Um pedido que mude de status dispara com eficácia o evento de inserção, garantidamente em paralelo e com Atomicidade (rollback na webhook_outbox se o status falhar).
* As permissões de acesso validam usuários sem perfil `ADMIN` tentando repassar eventos enfileirados pela DLQ.

## Estratégia de Testes e Validação
* **Testes End-to-End (E2E):** Subir containers com `worker` simulando chamadas HTTP para servidores mockados e interceptando as requisições para validar headers (presença e correção do `X-Signature` e `X-Event-Id`), limites (timeout em 10s) e retries.
* **Testes de Transação:** Testar o `OrderService` provocando exceções no final do fluxo após a inserção na fila e validar que não ficou lixo retido na tabela de outbox ou na `order_status_history` (rollback geral).
* **Revisão de Segurança (Manual):** Dedicação de dois dias da engenharia de segurança para revisar de forma minuciosa toda rotina de geração de secrets, hash e tolerâncias (Timeouts/Payload size limit).
