# ADR 006: Reuso de Padrões Base do Projeto

## Status
Aceito

## Contexto
O software OMS base encontra-se estável, formatado por um padrão nítido: uma camada limpa de regras sob `src/modules/*` guiando domínios (orders, customers). Ele unifica suas falhas sob o middleware da raiz, padroniza as *promises* não cumpridas numa *Custom Exception Class* e loga com bibliotecas enxutas. O componente Webhooks (Tabela de Configuração e Disparador) será o mais novo cidadão dessa arquitetura.

## Decisão
Foi decretado o **Reaproveitamento Irrestrito** dos padrões locais:
- Nova funcionalidade agrupada em `src/modules/webhooks/`;
- Exceções injetadas no construtor `AppError` da raiz `src/shared/errors/`;
- Padronização no formato: identificador customizado prefixado ex: `WEBHOOK_NOT_FOUND`;
- Interceptador nativo, dispensando reescrever regras de Zod e respostas nos escopos de Erros HTTP via reuso estrito do middleware em `src/middlewares/error.middleware.ts`;
- Registrador unificado via `Pino` logando sobre os esquemas vigentes `src/shared/logger`.

## Alternativas Consideradas
* Nenhuma alternativa drástica de desvio foi alavancada (ex: Adicionar novas bibliotecas para *Express Route Catchers* e Winston pra Logger). Adicionar dependências heterogêneas no micro-projeto acarretaria apenas inchaço estrutural descabido.

## Consequências
* **Positivas**: Promove legibilidade sistêmica, unificação de padrões e o desenvolvedor se sente imediatamente familiar. Menor risco de vazar dados se todo erro passar no filtro central do middleware.
* **Negativas**: A adoção dos blocos engessados nos impede de incorporar ferramentas inovadoras específicas apenas para esse módulo. (Custo marginal e considerado pífio ante aos benefícios de manter as bases uníssonas).
