# ADR 004: Garantia de Entrega At-Least-Once

## Status
Aceito

## Contexto
Durante a veiculação de notificações push assíncronas, pode haver micro instabilidades de rede (timeout no ACKs HTTP, problemas do gateway do cliente) forçando o nosso serviço a disparar a mesma notificação repetidas vezes (retries) mesmo que, por trás das cortinas, o cliente tenha recebido e salvo os dados da primeira emissão.

## Decisão
Abriremos mão do conceito "Exatamente uma vez", aderindo formalmente ao compromisso **At-Least-Once (Pelo menos uma vez)**. Para possibilitar ao parceiro de se defender das duplicações sistêmicas, nossa API injetará no header HTTP o marcador unívoco `X-Event-Id` (gerado de um padrão UUID no momento da transação), transferindo o peso computacional de *deduplication* para o consumidor do webhook.

## Alternativas Consideradas
* **Exactly-once delivery**: A garantia rigorosa de entrega cravada de "uma e somente uma vez". Foi vetada devido aos altíssimos requerimentos de coordenação distribuída (handshakes adicionais, Two-Phase Commits e dependência absurda que prenderiam requisições de nosso Worker para coordenar *acks* definitivos de sistemas opacos e inacessíveis para nós).

## Consequências
* **Positivas**: Simplicidade e desacoplamento radical. O worker não aguarda estados distribuídos; ele envia, e baseia-se apenas no *status-code* puro da requisição para avançar seu *pointer* de mensagens com segurança. Reduz complexidade em escala.
* **Negativas**: Oneração no desenvolvedor do cliente. Ele deve obrigatoriamente manter cacheamento ou indexação sobre chaves idempotentes baseadas no *UUID* para abster-se de duplicação perniciosa. Devidamente destacado e minimizado num doc portal amigável.
