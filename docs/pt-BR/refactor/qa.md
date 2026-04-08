---
x-i18n:
    generated_at: "2026-04-08T05:27:22Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4a9066b2a939c5a9ba69141d75405f0e8097997b523164340e2f0e9a0d5060dd
    source_path: refactor/qa.md
    workflow: 15
---

# Refatoração de QA

Status: migração fundamental concluída.

## Objetivo

Mover o QA do OpenClaw de um modelo de definição dividida para uma única fonte da verdade:

- metadados do cenário
- prompts enviados ao modelo
- configuração e desmontagem
- lógica do harness
- asserções e critérios de sucesso
- artefatos e sugestões de relatório

O estado final desejado é um harness de QA genérico que carrega arquivos poderosos de definição de cenários em vez de codificar a maior parte do comportamento em TypeScript.

## Estado atual

A fonte primária da verdade agora vive em `qa/scenarios/index.md` mais um arquivo por
cenário em `qa/scenarios/*.md`.

Implementado:

- `qa/scenarios/index.md`
  - metadados canônicos do pacote de QA
  - identidade do operador
  - missão inicial
- `qa/scenarios/*.md`
  - um arquivo Markdown por cenário
  - metadados do cenário
  - associações de handlers
  - configuração de execução específica do cenário
- `extensions/qa-lab/src/scenario-catalog.ts`
  - parser do pacote Markdown + validação com zod
- `extensions/qa-lab/src/qa-agent-bootstrap.ts`
  - renderização do plano a partir do pacote Markdown
- `extensions/qa-lab/src/qa-agent-workspace.ts`
  - gera arquivos de compatibilidade semeados mais `QA_SCENARIOS.md`
- `extensions/qa-lab/src/suite.ts`
  - seleciona cenários executáveis por meio de associações de handlers definidas em Markdown
- protocolo do barramento de QA + UI
  - anexos inline genéricos para renderização de imagem/vídeo/áudio/arquivo

Superfícies divididas restantes:

- `extensions/qa-lab/src/suite.ts`
  - ainda concentra a maior parte da lógica executável de handlers personalizados
- `extensions/qa-lab/src/report.ts`
  - ainda deriva a estrutura do relatório a partir das saídas em tempo de execução

Portanto, a divisão da fonte da verdade foi corrigida, mas a execução ainda é majoritariamente baseada em handlers, e não totalmente declarativa.

## Como é a superfície real de cenários

Ao ler a suíte atual, é possível ver algumas classes distintas de cenários.

### Interação simples

- linha de base de canal
- linha de base de MD
- acompanhamento em thread
- troca de modelo
- continuidade após aprovação
- reação/edição/exclusão

### Mutação de configuração e tempo de execução

- patch de configuração para desabilitar skill
- aplicação de configuração com reativação após reinício
- inversão de capacidade após reinício da configuração
- verificação de divergência do inventário em tempo de execução

### Asserções de sistema de arquivos e repositório

- relatório de descoberta de código/documentação
- compilar Lobster Invaders
- busca de artefato de imagem gerada

### Orquestração de memória

- recuperação de memória
- ferramentas de memória em contexto de canal
- fallback em falha de memória
- classificação de memória de sessão
- isolamento de memória de thread
- varredura de sonho de memória

### Integração de ferramentas e plugins

- chamada de plugin-tools do MCP
- visibilidade de skill
- instalação dinâmica de skill
- geração nativa de imagem
- roundtrip de imagem
- compreensão de imagem a partir de anexo

### Vários turnos e vários atores

- transferência para subagente
- síntese com distribuição para subagentes
- fluxos no estilo recuperação após reinício

Essas categorias são importantes porque orientam os requisitos da DSL. Uma lista simples de prompt + texto esperado não é suficiente.

## Direção

### Fonte única da verdade

Usar `qa/scenarios/index.md` mais `qa/scenarios/*.md` como a fonte da verdade
autoral.

O pacote deve permanecer:

- legível para humanos em revisão
- analisável por máquina
- rico o suficiente para orientar:
  - execução da suíte
  - bootstrap do workspace de QA
  - metadados da UI do QA Lab
  - prompts de documentação/descoberta
  - geração de relatórios

### Formato de autoria preferido

Usar Markdown como formato de nível superior, com YAML estruturado dentro dele.

Formato recomendado:

- frontmatter YAML
  - id
  - title
  - surface
  - tags
  - refs de documentação
  - refs de código
  - substituições de modelo/provedor
  - pré-requisitos
- seções em prosa
  - objetivo
  - notas
  - dicas de depuração
- blocos YAML delimitados
  - setup
  - steps
  - assertions
  - cleanup

Isso oferece:

- melhor legibilidade em PR do que um JSON gigante
- contexto mais rico do que YAML puro
- parsing estrito e validação com zod

JSON bruto é aceitável apenas como uma forma intermediária gerada.

## Formato proposto para arquivo de cenário

Exemplo:

````md
---
id: image-generation-roundtrip
title: Image generation roundtrip
surface: image
tags: [media, image, roundtrip]
models:
  primary: openai/gpt-5.4
requires:
  tools: [image_generate]
  plugins: [openai, qa-channel]
docsRefs:
  - docs/help/testing.md
  - docs/concepts/model-providers.md
codeRefs:
  - extensions/qa-lab/src/suite.ts
  - src/gateway/chat-attachments.ts
---

# Objective

Verify generated media is reattached on the follow-up turn.

# Setup

```yaml scenario.setup
- action: config.patch
  patch:
    agents:
      defaults:
        imageGenerationModel:
          primary: openai/gpt-image-1
- action: session.create
  key: agent:qa:image-roundtrip
```

# Steps

```yaml scenario.steps
- action: agent.send
  session: agent:qa:image-roundtrip
  message: |
    Image generation check: generate a QA lighthouse image and summarize it in one short sentence.
- action: artifact.capture
  kind: generated-image
  promptSnippet: Image generation check
  saveAs: lighthouseImage
- action: agent.send
  session: agent:qa:image-roundtrip
  message: |
    Roundtrip image inspection check: describe the generated lighthouse attachment in one short sentence.
  attachments:
    - fromArtifact: lighthouseImage
```

# Expect

```yaml scenario.expect
- assert: outbound.textIncludes
  value: lighthouse
- assert: requestLog.matches
  where:
    promptIncludes: Roundtrip image inspection check
  imageInputCountGte: 1
- assert: artifact.exists
  ref: lighthouseImage
```
````

## Capacidades do runner que a DSL precisa cobrir

Com base na suíte atual, o runner genérico precisa de mais do que execução de prompt.

### Ações de ambiente e configuração

- `bus.reset`
- `gateway.waitHealthy`
- `channel.waitReady`
- `session.create`
- `thread.create`
- `workspace.writeSkill`

### Ações de turno do agente

- `agent.send`
- `agent.wait`
- `bus.injectInbound`
- `bus.injectOutbound`

### Ações de configuração e tempo de execução

- `config.get`
- `config.patch`
- `config.apply`
- `gateway.restart`
- `tools.effective`
- `skills.status`

### Ações de arquivo e artefato

- `file.write`
- `file.read`
- `file.delete`
- `file.touchTime`
- `artifact.captureGeneratedImage`
- `artifact.capturePath`

### Ações de memória e cron

- `memory.indexForce`
- `memory.searchCli`
- `doctor.memory.status`
- `cron.list`
- `cron.run`
- `cron.waitCompletion`
- `sessionTranscript.write`

### Ações de MCP

- `mcp.callTool`

### Asserções

- `outbound.textIncludes`
- `outbound.inThread`
- `outbound.notInRoot`
- `tool.called`
- `tool.notPresent`
- `skill.visible`
- `skill.disabled`
- `file.contains`
- `memory.contains`
- `requestLog.matches`
- `sessionStore.matches`
- `cron.managedPresent`
- `artifact.exists`

## Variáveis e referências a artefatos

A DSL deve dar suporte a saídas salvas e referências posteriores.

Exemplos da suíte atual:

- criar uma thread e depois reutilizar `threadId`
- criar uma sessão e depois reutilizar `sessionKey`
- gerar uma imagem e depois anexar o arquivo no turno seguinte
- gerar uma string de marcador de reativação e depois verificar que ela aparece mais tarde

Capacidades necessárias:

- `saveAs`
- `${vars.name}`
- `${artifacts.name}`
- referências tipadas para caminhos, chaves de sessão, ids de thread, marcadores e saídas de ferramentas

Sem suporte a variáveis, o harness continuará deixando a lógica de cenário vazar de volta para o TypeScript.

## O que deve permanecer como escape hatch

Um runner totalmente declarativo e puro não é realista na fase 1.

Alguns cenários são inerentemente pesados em orquestração:

- varredura de sonho de memória
- aplicação de configuração com reativação após reinício
- inversão de capacidade após reinício da configuração
- resolução de artefato de imagem gerada por timestamp/caminho
- avaliação de relatório de descoberta

Por enquanto, eles devem usar handlers personalizados explícitos.

Regra recomendada:

- 85-90% declarativo
- etapas `customHandler` explícitas para o restante mais difícil
- apenas handlers personalizados nomeados e documentados
- nenhum código inline anônimo no arquivo de cenário

Isso mantém o mecanismo genérico limpo e ainda permite progresso.

## Mudança de arquitetura

### Atual

O Markdown de cenário já é a fonte da verdade para:

- execução da suíte
- arquivos de bootstrap do workspace
- catálogo de cenários da UI do QA Lab
- metadados do relatório
- prompts de descoberta

Compatibilidade gerada:

- o workspace semeado ainda inclui `QA_KICKOFF_TASK.md`
- o workspace semeado ainda inclui `QA_SCENARIO_PLAN.md`
- o workspace semeado agora também inclui `QA_SCENARIOS.md`

## Plano de refatoração

### Fase 1: loader e schema

Concluído.

- adicionado `qa/scenarios/index.md`
- cenários divididos em `qa/scenarios/*.md`
- adicionado parser para conteúdo nomeado do pacote Markdown YAML
- validado com zod
- consumidores alterados para usar o pacote parseado
- removidos `qa/seed-scenarios.json` e `qa/QA_KICKOFF_TASK.md` no nível do repositório

### Fase 2: mecanismo genérico

- dividir `extensions/qa-lab/src/suite.ts` em:
  - loader
  - mecanismo
  - registro de ações
  - registro de asserções
  - handlers personalizados
- manter as funções auxiliares existentes como operações do mecanismo

Entregável:

- o mecanismo executa cenários declarativos simples

Começar com cenários que são principalmente prompt + espera + asserção:

- acompanhamento em thread
- compreensão de imagem a partir de anexo
- visibilidade e invocação de skill
- linha de base de canal

Entregável:

- primeiros cenários reais definidos em Markdown enviados pelo mecanismo genérico

### Fase 4: migrar cenários intermediários

- roundtrip de geração de imagem
- ferramentas de memória em contexto de canal
- classificação de memória de sessão
- transferência para subagente
- síntese com distribuição para subagentes

Entregável:

- variáveis, artefatos, asserções de ferramenta e asserções de log de requisição comprovados

### Fase 5: manter cenários difíceis em handlers personalizados

- varredura de sonho de memória
- aplicação de configuração com reativação após reinício
- inversão de capacidade após reinício da configuração
- divergência do inventário em tempo de execução

Entregável:

- mesmo formato de autoria, mas com blocos de etapas personalizadas explícitas quando necessário

### Fase 6: excluir o mapa de cenários codificado

Quando a cobertura do pacote estiver boa o suficiente:

- remover a maior parte da ramificação TypeScript específica de cenários de `extensions/qa-lab/src/suite.ts`

## Suporte a Slack falso / rich media

O barramento de QA atual é focado em texto.

Arquivos relevantes:

- `extensions/qa-channel/src/protocol.ts`
- `extensions/qa-lab/src/bus-state.ts`
- `extensions/qa-lab/src/bus-queries.ts`
- `extensions/qa-lab/src/bus-server.ts`
- `extensions/qa-lab/web/src/ui-render.ts`

Hoje, o barramento de QA oferece suporte a:

- texto
- reações
- threads

Ele ainda não modela anexos de mídia inline.

### Contrato de transporte necessário

Adicionar um modelo genérico de anexo do barramento de QA:

```ts
type QaBusAttachment = {
  id: string;
  kind: "image" | "video" | "audio" | "file";
  mimeType: string;
  fileName?: string;
  inline?: boolean;
  url?: string;
  contentBase64?: string;
  width?: number;
  height?: number;
  durationMs?: number;
  altText?: string;
  transcript?: string;
};
```

Depois adicionar `attachments?: QaBusAttachment[]` a:

- `QaBusMessage`
- `QaBusInboundMessageInput`
- `QaBusOutboundMessageInput`

### Por que genérico primeiro

Não construir um modelo de mídia exclusivo para Slack.

Em vez disso:

- um modelo genérico de transporte de QA
- vários renderizadores sobre ele
  - chat atual do QA Lab
  - futuro web de Slack falso
  - qualquer outra visualização de transporte falsa

Isso evita lógica duplicada e permite que cenários de mídia permaneçam agnósticos ao transporte.

### Trabalho de UI necessário

Atualizar a UI de QA para renderizar:

- pré-visualização de imagem inline
- player de áudio inline
- player de vídeo inline
- chip de anexo de arquivo

A UI atual já consegue renderizar threads e reações, então a renderização de anexos deve ser adicionada sobre o mesmo modelo de cartão de mensagem.

### Trabalho de cenário habilitado pelo transporte de mídia

Quando os anexos passarem pelo barramento de QA, poderemos adicionar cenários mais ricos de chat falso:

- resposta com imagem inline em Slack falso
- compreensão de anexo de áudio
- compreensão de anexo de vídeo
- ordenação mista de anexos
- resposta em thread com mídia preservada

## Recomendação

O próximo bloco de implementação deve ser:

1. adicionar loader de cenário em Markdown + schema zod
2. gerar o catálogo atual a partir de Markdown
3. migrar primeiro alguns cenários simples
4. adicionar suporte genérico a anexos no barramento de QA
5. renderizar imagem inline na UI de QA
6. depois expandir para áudio e vídeo

Este é o menor caminho que comprova ambos os objetivos:

- QA genérico definido por Markdown
- superfícies de mensagens falsas mais ricas

## Perguntas em aberto

- se os arquivos de cenário devem permitir templates de prompt em Markdown embutidos com interpolação de variáveis
- se setup/cleanup devem ser seções nomeadas ou apenas listas ordenadas de ações
- se referências a artefatos devem ser fortemente tipadas no schema ou baseadas em string
- se handlers personalizados devem viver em um único registro ou em registros por superfície
- se o arquivo de compatibilidade JSON gerado deve permanecer versionado durante a migração
