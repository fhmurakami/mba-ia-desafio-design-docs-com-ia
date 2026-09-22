# ADR 002: Política de Retry Exponencial e Dead Letter Queue (DLQ)

## Status
Aceito

## Contexto
Durante a tentativa de disparo dos webhooks para o cliente, podem ocorrer falhas de timeout, instabilidade de rede ou até recusa temporária do serviço final (erros 500+). Não podemos descartar o evento na primeira falha, e ao mesmo tempo, não podemos bloquear a fila por excesso de tentativas.

## Decisão
Implementaremos uma política de **backoff exponencial restrito a 5 tentativas** (intervalos de 1m, 5m, 30m, 2h, e 12h). Caso o envio falhe pela quinta vez, o registro sai da tabela ativa `webhook_outbox` e é transferido para uma tabela segregada de isolamento e log permanente denominada `webhook_dead_letter`.
Um endpoint restrito para administradores (`POST /admin/webhooks/dead-letter/:id/replay`) permitirá reinjetar manualmente o evento descartado.

## Alternativas Consideradas
* **Retries indefinidos com backoff prolongado (10+ tentativas)**: Descartado porque um evento persistente preso há semanas num cliente indisponível seria processado em vão e sujaria o banco. Uma indisponibilidade maior que 15 horas já acusa erro sistêmico do lado cliente.
* **Três tentativas de retry rápido (Agressivo)**: Descartado. Três retries curtos limitam a janela de salvaguarda a cerca de 30 minutos, tempo muito inferior a grandes paradas por manutenção que nossos clientes B2B operam.

## Consequências
* **Positivas**: Cobre janelas de paradas de parceiros e blinda o worker da nossa ponta de rodar desperdiçando recursos. O banco principal limpa registros crônicos rapidamente.
* **Negativas**: Necessidade de desenhar e modelar novos cruds de interface de manutenção (`Dead Letter Queue Replay`), onerando meia sprint no ciclo de entrega atual.
