---
read_when:
    - Ao estender qa-lab ou qa-channel
    - Ao adicionar cenários de QA com suporte do repositório
    - Ao criar automação de QA com maior realismo em torno do painel do Gateway
summary: Formato da automação privada de QA para qa-lab, qa-channel, cenários predefinidos e relatórios de protocolo
title: Automação E2E de QA
x-i18n:
    generated_at: "2026-04-08T05:26:34Z"
    model: gpt-5.4
    provider: openai
    source_hash: 57da147dc06abf9620290104e01a83b42182db1806514114fd9e8467492cda99
    source_path: concepts/qa-e2e-automation.md
    workflow: 15
---

# Automação E2E de QA

A pilha privada de QA foi criada para exercitar o OpenClaw de uma forma mais
realista e moldada por canais do que um único teste unitário consegue.

Peças atuais:

- `extensions/qa-channel`: canal de mensagens sintético com superfícies de MD, canal, thread,
  reação, edição e exclusão.
- `extensions/qa-lab`: UI de depuração e barramento de QA para observar a transcrição,
  injetar mensagens de entrada e exportar um relatório em Markdown.
- `qa/`: recursos predefinidos com suporte do repositório para a tarefa inicial e cenários
  básicos de QA.

O fluxo atual do operador de QA é um site de QA em dois painéis:

- Esquerda: painel do Gateway (Control UI) com o agente.
- Direita: QA Lab, mostrando a transcrição em estilo Slack e o plano de cenário.

Execute com:

```bash
pnpm qa:lab:up
```

Isso compila o site de QA, inicia a trilha do gateway com suporte do Docker e expõe a
página do QA Lab, onde um operador ou loop de automação pode dar ao agente uma
missão de QA, observar o comportamento real do canal e registrar o que funcionou, falhou ou
permaneceu bloqueado.

Para uma iteração mais rápida da UI do QA Lab sem recompilar a imagem Docker a cada vez,
inicie a pilha com um bundle do QA Lab montado por bind:

```bash
pnpm openclaw qa docker-build-image
pnpm qa:lab:build
pnpm qa:lab:up:fast
pnpm qa:lab:watch
```

`qa:lab:up:fast` mantém os serviços Docker em uma imagem pré-compilada e faz bind mount de
`extensions/qa-lab/web/dist` no contêiner `qa-lab`. `qa:lab:watch`
recompila esse bundle a cada alteração, e o navegador recarrega automaticamente quando o hash
do recurso do QA Lab muda.

## Recursos predefinidos com suporte do repositório

Os recursos predefinidos ficam em `qa/`:

- `qa/scenarios/index.md`
- `qa/scenarios/*.md`

Eles ficam intencionalmente no git para que o plano de QA seja visível tanto para humanos quanto para o
agente. A lista básica deve continuar ampla o suficiente para cobrir:

- chat por MD e em canal
- comportamento de thread
- ciclo de vida de ações de mensagens
- callbacks de cron
- recuperação de memória
- troca de modelo
- transferência para subagente
- leitura do repositório e da documentação
- uma pequena tarefa de compilação, como Lobster Invaders

## Relatórios

`qa-lab` exporta um relatório de protocolo em Markdown a partir da linha do tempo observada do barramento.
O relatório deve responder:

- O que funcionou
- O que falhou
- O que permaneceu bloqueado
- Quais cenários de acompanhamento vale a pena adicionar

## Documentação relacionada

- [Testing](/pt-BR/help/testing)
- [QA Channel](/pt-BR/channels/qa-channel)
- [Dashboard](/web/dashboard)
