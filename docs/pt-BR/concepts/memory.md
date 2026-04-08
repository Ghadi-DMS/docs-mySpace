---
read_when:
    - Você quer entender como a memória funciona
    - Você quer saber quais arquivos de memória escrever
summary: Como o OpenClaw se lembra das coisas entre sessões
title: Visão geral da memória
x-i18n:
    generated_at: "2026-04-08T05:26:41Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3bb8552341b0b651609edaaae826a22fdc535d240aed4fad4af4b069454004af
    source_path: concepts/memory.md
    workflow: 15
---

# Visão geral da memória

O OpenClaw se lembra das coisas gravando **arquivos Markdown simples** no
workspace do seu agente. O modelo só “se lembra” do que é salvo em disco -- não
há estado oculto.

## Como funciona

Seu agente tem três arquivos relacionados à memória:

- **`MEMORY.md`** -- memória de longo prazo. Fatos duradouros, preferências e
  decisões. Carregado no início de toda sessão de DM.
- **`memory/YYYY-MM-DD.md`** -- notas diárias. Contexto contínuo e observações.
  As notas de hoje e de ontem são carregadas automaticamente.
- **`DREAMS.md`** (experimental, opcional) -- Diário de Sonhos e resumos das
  varreduras de sonhos para revisão humana.

Esses arquivos ficam no workspace do agente (padrão `~/.openclaw/workspace`).

<Tip>
Se você quiser que seu agente se lembre de algo, basta pedir: "Lembre-se de que
eu prefiro TypeScript." Ele gravará isso no arquivo apropriado.
</Tip>

## Ferramentas de memória

O agente tem duas ferramentas para trabalhar com memória:

- **`memory_search`** -- encontra notas relevantes usando busca semântica,
  mesmo quando a formulação difere da original.
- **`memory_get`** -- lê um arquivo de memória específico ou um intervalo de
  linhas.

Ambas as ferramentas são fornecidas pelo plugin de memória ativo (padrão:
`memory-core`).

## Plugin complementar Memory Wiki

Se você quiser que a memória duradoura se comporte mais como uma base de
conhecimento mantida do que apenas notas brutas, use o plugin integrado
`memory-wiki`.

O `memory-wiki` compila o conhecimento duradouro em um cofre wiki com:

- estrutura de páginas determinística
- afirmações e evidências estruturadas
- rastreamento de contradições e atualidade
- painéis gerados
- resumos compilados para consumidores do agente/runtime
- ferramentas nativas de wiki como `wiki_search`, `wiki_get`, `wiki_apply` e `wiki_lint`

Ele não substitui o plugin de memória ativo. O plugin de memória ativo ainda
controla recordação, promoção e sonhos. O `memory-wiki` adiciona uma camada de
conhecimento rica em procedência ao lado dele.

Veja [Memory Wiki](/pt-BR/plugins/memory-wiki).

## Busca de memória

Quando um provedor de embeddings está configurado, `memory_search` usa **busca
híbrida** -- combinando similaridade vetorial (significado semântico) com
correspondência por palavra-chave (termos exatos como IDs e símbolos de código).
Isso funciona imediatamente assim que você tiver uma chave de API para qualquer
provedor compatível.

<Info>
O OpenClaw detecta automaticamente seu provedor de embeddings a partir das
chaves de API disponíveis. Se você tiver uma chave da OpenAI, Gemini, Voyage ou
Mistral configurada, a busca de memória será ativada automaticamente.
</Info>

Para detalhes sobre como a busca funciona, opções de ajuste e configuração do
provedor, veja [Memory Search](/pt-BR/concepts/memory-search).

## Backends de memória

<CardGroup cols={3}>
<Card title="Integrado (padrão)" icon="database" href="/pt-BR/concepts/memory-builtin">
Baseado em SQLite. Funciona imediatamente com busca por palavra-chave,
similaridade vetorial e busca híbrida. Sem dependências extras.
</Card>
<Card title="QMD" icon="search" href="/pt-BR/concepts/memory-qmd">
Sidecar local-first com reranqueamento, expansão de consulta e a capacidade de
indexar diretórios fora do workspace.
</Card>
<Card title="Honcho" icon="brain" href="/pt-BR/concepts/memory-honcho">
Memória entre sessões nativa de IA com modelagem de usuário, busca semântica e
consciência de múltiplos agentes. Instalação por plugin.
</Card>
</CardGroup>

## Camada wiki de conhecimento

<CardGroup cols={1}>
<Card title="Memory Wiki" icon="book" href="/pt-BR/plugins/memory-wiki">
Compila a memória duradoura em um cofre wiki rico em procedência com
afirmações, painéis, modo bridge e fluxos de trabalho compatíveis com Obsidian.
</Card>
</CardGroup>

## Descarga automática de memória

Antes que a [compactação](/pt-BR/concepts/compaction) resuma sua conversa, o OpenClaw
executa um turno silencioso que lembra o agente de salvar contexto importante
em arquivos de memória. Isso vem ativado por padrão -- você não precisa
configurar nada.

<Tip>
A descarga de memória evita perda de contexto durante a compactação. Se o seu
agente tiver fatos importantes na conversa que ainda não foram gravados em um
arquivo, eles serão salvos automaticamente antes que o resumo aconteça.
</Tip>

## Sonhos (experimental)

Sonhos é uma etapa opcional de consolidação de memória em segundo plano. Ela
coleta sinais de curto prazo, pontua candidatos e promove apenas itens
qualificados para a memória de longo prazo (`MEMORY.md`).

Ela foi projetada para manter a memória de longo prazo com alto sinal:

- **Opt-in**: desativado por padrão.
- **Agendado**: quando ativado, o `memory-core` gerencia automaticamente um job
  recorrente de cron para uma varredura completa de sonhos.
- **Com limiar**: as promoções precisam passar por critérios de pontuação,
  frequência de recordação e diversidade de consulta.
- **Revisável**: resumos de fase e entradas do diário são gravados em
  `DREAMS.md` para revisão humana.

Para comportamento por fase, sinais de pontuação e detalhes do Diário de
Sonhos, veja [Dreaming (experimental)](/pt-BR/concepts/dreaming).

## CLI

```bash
openclaw memory status          # Verificar status do índice e do provedor
openclaw memory search "query"  # Pesquisar pela linha de comando
openclaw memory index --force   # Reconstruir o índice
```

## Leitura adicional

- [Builtin Memory Engine](/pt-BR/concepts/memory-builtin) -- backend SQLite padrão
- [QMD Memory Engine](/pt-BR/concepts/memory-qmd) -- sidecar local-first avançado
- [Honcho Memory](/pt-BR/concepts/memory-honcho) -- memória nativa de IA entre sessões
- [Memory Wiki](/pt-BR/plugins/memory-wiki) -- cofre de conhecimento compilado e ferramentas nativas de wiki
- [Memory Search](/pt-BR/concepts/memory-search) -- pipeline de busca, provedores e
  ajuste
- [Dreaming (experimental)](/pt-BR/concepts/dreaming) -- promoção em segundo plano
  da recordação de curto prazo para a memória de longo prazo
- [Memory configuration reference](/pt-BR/reference/memory-config) -- todos os ajustes de configuração
- [Compaction](/pt-BR/concepts/compaction) -- como a compactação interage com a memória
