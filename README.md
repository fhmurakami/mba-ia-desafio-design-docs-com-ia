# Sistema de Webhooks: De Transcrição a Design Docs com IA

> **Nota:** As instruções originais do desafio podem ser encontradas no arquivo [`ENUNCIADO.md`](ENUNCIADO.md).

## Sobre o Desafio
Este repositório contém a entrega do desafio prático focado em transformar a transcrição bruta de uma reunião técnica em um pacote completo e maduro de especificações de design de software (Design Docs). O objetivo central foi atuar como um "maestro" de Inteligência Artificial, orientando-a a extrair requisitos funcionais, não funcionais e decisões arquiteturais do texto, garantindo que tudo estivesse coeso com a base de código preexistente do Order Management System (OMS).

A missão exigiu não apenas sumarizar, mas categorizar cada artefato em sua devida "altura": o PRD para regras de negócio, o RFC para propor a arquitetura, os ADRs para registrar formalmente as escolhas e o FDD para detalhar os contratos de API e integrações no código. Tudo isso amarrado por uma tabela rigorosa de rastreabilidade (Tracker).

## Ferramentas de IA Utilizadas
* **Claude / ChatGPT / Gemini (LLMs de interação direta)**: Utilizados como motores principais de raciocínio lógico e formatação Markdown. Eles desempenharam o papel de extrair os pontos-chave da reunião, formular os textos formais e organizar a rastreabilidade estruturada.
* **Agente de Automação (Antigravity/Cortex)**: Ferramenta de linha de comando baseada em IA acoplada ao terminal local, permitindo que a IA lesse ativamente o repositório (`ls`, `cat`), escrevesse os arquivos em seus devidos diretórios de forma autônoma e os comitasse, poupando o trabalho braçal de "copiar e colar" e permitindo focar estritamente na validação técnica.

## Workflow Adotado
A construção documental foi feita em etapas sequenciais e encadeadas (Chain of Thought), aproveitando o output do passo anterior como insumo para o próximo:
1. **Entendimento e Leitura Fria**: O primeiro passo foi pedir para a IA apenas *ler* o repositório, os enunciados e o `TRANSCRICAO.md` e gerar um resumo estrito sem criar arquivos.
2. **PRD & Tracker**: Instruí a IA a redigir o Product Requirement Document listando o escopo descartado. A partir dele, inicializamos o `TRACKER.md` obrigando que cada requisito fosse linkado a um `[hh:mm]` da fala.
3. **RFC & ADRs**: Com as regras em mãos, focamos nas decisões (ex: uso do MySQL, polling, HMAC). O RFC levantou as alternativas vetadas na call. Os ADRs aprofundaram as consequências de cada uma das seis escolhas.
4. **FDD e Aderência ao Código**: Por último, a IA buscou os arquivos reais (como `app-error.ts` e `order.service.ts`) para atrelar a arquitetura recém-concebida aos diretórios do boilerplate base.

## Prompts Customizados

**Prompt de Geração e Isolamento de Escopo (PRD):**
```text
Atue como um Product Manager Técnico sênior.
Leia o arquivo `TRANSCRICAO.md`. Extraia os requisitos de negócio e crie o arquivo docs/PRD.md.
Regra de Ouro: Identifique com precisão cirúrgica o que a equipe EXPLICITAMENTE DECIDIU ADIAR ou DESCARTAR e crie uma subseção de 'Fora de Escopo' (Ex: Envio de e-mail, dashboard, etc).
Não invente nenhuma funcionalidade. Todo requisito precisa ser mapeado no TRACKER.md referenciando o timestamp do orador.
```

**Prompt de Exploração de Código (FDD):**
```text
Atue como Engenheiro de Software Sênior. 
Antes de escrever o FDD (Feature Design Document), utilize o terminal para listar e ler (`ls` e `cat`) os arquivos nas pastas `src/modules/orders`, `src/middlewares` e `src/shared`.
Encontre 4 arquivos cruciais existentes e, ao escrever a seção "Integração com o sistema existente" no docs/FDD.md, cite-os pelo caminho exato explicando como eles sofrerão extensão ou reaproveitamento (sem alterar o código em si).
```

## Iterações e Ajustes
Embora a IA seja brilhante para formatar, ela tende a agrupar coisas precocemente ou alucinar:
1. **Primeira iteração de rastreabilidade (Tracker)**: A IA havia aglomerado vários arquivos num único tabelão. Foi necessário intervir manualmente com um prompt corretivo ordenando: *"Separe cada arquivo (PRD, RFC, FDD, ADR) em uma seção e tabela diferentes no TRACKER.md"*.
2. **Correção de duplicidades de fala**: Ocorreram momentos em que uma decisão foi debatida por múltiplos membros na call (ex: *Diego* e *Larissa*). Inicialmente, o mapeamento considerou apenas o primeiro orador. Tive que iterar e pedir: *"Verifique se não há mais de uma menção à mesma feature, e caso haja, adicione na coluna Localização separada por vírgulas"*.

## Como Navegar a Entrega
Para entender a jornada arquitetural da mesma forma como foi concebida, recomendo a leitura na seguinte ordem estrutural (do nível de Negócio para o nível de Implementação Técnica):

1. `docs/PRD.md`: Para entender "O quê" estamos construindo e "Por que" (Visão de Produto).
2. `docs/RFC.md`: Para ler a proposta arquitetural de alto nível e entender por que outras vias foram rejeitadas.
3. `docs/adrs/README.md` (e as respectivas ADRs na pasta): Para o aprofundamento isolado sobre os *trade-offs* adotados.
4. `docs/FDD.md`: Para enxergar "Como" os desenvolvedores aplicarão isso no código real (APIs, logs, middlewares).
5. `docs/TRACKER.md`: Consultar a qualquer momento para auditar a veracidade das informações apresentadas nos documentos acima contra a transcrição da reunião.
