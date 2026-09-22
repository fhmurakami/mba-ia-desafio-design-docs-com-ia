# ADR 003: Autenticação de Mensagem com HMAC-SHA256

## Status
Aceito

## Contexto
Visto que enviaremos notificações de webhooks para a rede aberta da internet aos nossos clientes contendo o payload e status dos seus pedidos logísticos, o cliente necessita de garantias inegáveis de que a mensagem partiu de nossa plataforma e não sofreu nenhuma mutação (Man-In-The-Middle).

## Decisão
O disparo trará uma assinatura embutida no cabeçalho `X-Signature`, formatada por um **HMAC utilizando o algoritmo SHA-256** sobre o binário do payload.
Haverá uma `secret` roteável exclusiva gerada por *endpoint* cadastrado pelo cliente. Essa chave deve ter suporte a rotatividade (*grace period* de 24 horas onde chave nova e antiga convivem). O protocolo TLS (apenas rotas em `HTTPS`) será validação mandatória pelo validador Zod já no cadastro via API.

## Alternativas Consideradas
* **Autenticação Mutual TLS (mTLS)**: O TLS mútuo é padrão de ouro para comunicações B2B sensíveis (bancos). Descartado por conta da severa dificuldade de suporte do lado dos clientes logísticos parceiros. Assinar via HMAC atinge segurança criptográfica excepcional sem requerer complexidade de proxy certificado deles.
* **Secret única Global por Customer/Account**: Descartado por falhas de resiliência. Se uma secret vazar nos logs de um microsserviço do cliente, o vazamento alcança todos os outros end-points registrados por ele no sistema. Chave única por cadastro assegura blindagem lateral.

## Consequências
* **Positivas**: Elevada aprovação pela comunidade, suporte farto nas linguagens modernas. Adição do *grace period* diminui os atritos de quebra de contrato.
* **Negativas**: Demanda maior esforço durante o fluxo de cadastro e deleção para gerir e devolver as chaves em segurança na configuração inicial e rotação. Requisito de processamento (hasheamento em memória no worker) a cada push gerado.
