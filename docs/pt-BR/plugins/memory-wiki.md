---
read_when:
    - Você quer conhecimento persistente além de simples notas MEMORY.md
    - Você está configurando o plugin empacotado memory-wiki
    - Você quer entender wiki_search, wiki_get ou o modo bridge
summary: 'memory-wiki: cofre de conhecimento compilado com procedência, afirmações, painéis e modo bridge'
title: Wiki de Memória
x-i18n:
    generated_at: "2026-04-08T05:26:51Z"
    model: gpt-5.4
    provider: openai
    source_hash: b78dd6a4ef4451dae6b53197bf0c7c2a2ba846b08e4a3a93c1026366b1598d82
    source_path: plugins/memory-wiki.md
    workflow: 15
---

# Wiki de Memória

`memory-wiki` é um plugin empacotado que transforma memória durável em um
cofre de conhecimento compilado.

Ele **não** substitui o plugin de memória ativo. O plugin de memória ativo ainda
é responsável por recuperação, promoção, indexação e dreaming. O `memory-wiki`
fica ao lado dele e compila conhecimento durável em uma wiki navegável com
páginas determinísticas, afirmações estruturadas, procedência, painéis e
resumos legíveis por máquina.

Use-o quando você quiser que a memória se comporte mais como uma camada de
conhecimento mantida e menos como uma pilha de arquivos Markdown.

## O que ele adiciona

- Um cofre de wiki dedicado com layout de página determinístico
- Metadados estruturados de afirmações e evidências, não apenas prosa
- Procedência, confiança, contradições e perguntas em aberto no nível da página
- Resumos compilados para consumidores de agente/runtime
- Ferramentas nativas da wiki para search/get/apply/lint
- Modo bridge opcional que importa artefatos públicos do plugin de memória ativo
- Modo de renderização opcional compatível com Obsidian e integração com a CLI

## Como ele se encaixa com a memória

Pense na divisão assim:

| Camada                                                  | Responsável por                                                                            |
| ------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| Plugin de memória ativo (`memory-core`, QMD, Honcho, etc.) | Recuperação, busca semântica, promoção, dreaming, runtime de memória                    |
| `memory-wiki`                                           | Páginas de wiki compiladas, sínteses ricas em procedência, painéis, search/get/apply específicos da wiki |

Se o plugin de memória ativo expuser artefatos de recuperação compartilhados, o OpenClaw poderá buscar
nas duas camadas em uma única passagem com `memory_search corpus=all`.

Quando você precisar de classificação específica da wiki, procedência ou acesso
direto à página, use as ferramentas nativas da wiki.

## Modos de cofre

O `memory-wiki` oferece suporte a três modos de cofre:

### `isolated`

Cofre próprio, fontes próprias, sem dependência de `memory-core`.

Use isto quando quiser que a wiki seja seu próprio repositório de conhecimento curado.

### `bridge`

Lê artefatos públicos de memória e eventos de memória do plugin de memória ativo
por meio de interfaces públicas do SDK de plugin.

Use isto quando quiser que a wiki compile e organize os artefatos exportados
pelo plugin de memória sem acessar internamente partes privadas do plugin.

O modo bridge pode indexar:

- artefatos de memória exportados
- relatórios de dream
- notas diárias
- arquivos raiz de memória
- logs de eventos de memória

### `unsafe-local`

Saída de escape explícita na mesma máquina para caminhos privados locais.

Esse modo é intencionalmente experimental e não portátil. Use-o apenas quando
você entender o limite de confiança e precisar especificamente de acesso ao
sistema de arquivos local que o modo bridge não pode fornecer.

## Layout do cofre

O plugin inicializa um cofre assim:

```text
<vault>/
  AGENTS.md
  WIKI.md
  index.md
  inbox.md
  entities/
  concepts/
  syntheses/
  sources/
  reports/
  _attachments/
  _views/
  .openclaw-wiki/
```

O conteúdo gerenciado permanece dentro de blocos gerados. Blocos de notas
humanas são preservados.

Os principais grupos de páginas são:

- `sources/` para matéria-prima importada e páginas sustentadas por bridge
- `entities/` para coisas duráveis, pessoas, sistemas, projetos e objetos
- `concepts/` para ideias, abstrações, padrões e políticas
- `syntheses/` para resumos compilados e consolidações mantidas
- `reports/` para painéis gerados

## Afirmações estruturadas e evidências

As páginas podem carregar `claims` no frontmatter estruturado, não apenas texto livre.

Cada afirmação pode incluir:

- `id`
- `text`
- `status`
- `confidence`
- `evidence[]`
- `updatedAt`

Entradas de evidência podem incluir:

- `sourceId`
- `path`
- `lines`
- `weight`
- `note`
- `updatedAt`

É isso que faz a wiki funcionar mais como uma camada de crenças do que como um
depósito passivo de notas. As afirmações podem ser rastreadas, pontuadas,
contestadas e resolvidas de volta às fontes.

## Pipeline de compilação

A etapa de compilação lê páginas da wiki, normaliza resumos e emite artefatos
estáveis voltados para máquina em:

- `.openclaw-wiki/cache/agent-digest.json`
- `.openclaw-wiki/cache/claims.jsonl`

Esses resumos existem para que agentes e código de runtime não precisem extrair
informações de páginas Markdown.

A saída compilada também alimenta:

- indexação inicial da wiki para fluxos de search/get
- busca de claim-id até a página proprietária
- suplementos compactos de prompt
- geração de relatórios/painéis

## Painéis e relatórios de integridade

Quando `render.createDashboards` está ativado, a compilação mantém painéis em
`reports/`.

Os relatórios integrados incluem:

- `reports/open-questions.md`
- `reports/contradictions.md`
- `reports/low-confidence.md`
- `reports/claim-health.md`
- `reports/stale-pages.md`

Esses relatórios acompanham itens como:

- grupos de notas de contradição
- grupos de afirmações concorrentes
- afirmações sem evidência estruturada
- páginas e afirmações com baixa confiança
- desatualização ou atualização desconhecida
- páginas com perguntas não resolvidas

## Busca e recuperação

O `memory-wiki` oferece suporte a dois backends de busca:

- `shared`: usa o fluxo de busca de memória compartilhada quando disponível
- `local`: busca na wiki localmente

Ele também oferece suporte a três corpora:

- `wiki`
- `memory`
- `all`

Comportamento importante:

- `wiki_search` e `wiki_get` usam resumos compilados como primeira passagem quando possível
- ids de afirmação podem ser resolvidos de volta à página proprietária
- afirmações contestadas/desatualizadas/atuais influenciam a classificação
- rótulos de procedência podem ser preservados nos resultados

Regra prática:

- use `memory_search corpus=all` para uma passagem ampla de recuperação
- use `wiki_search` + `wiki_get` quando você se importa com classificação
  específica da wiki, procedência ou estrutura de crenças no nível da página

## Ferramentas do agente

O plugin registra estas ferramentas:

- `wiki_status`
- `wiki_search`
- `wiki_get`
- `wiki_apply`
- `wiki_lint`

O que elas fazem:

- `wiki_status`: modo de cofre atual, integridade, disponibilidade da CLI do Obsidian
- `wiki_search`: busca em páginas da wiki e, quando configurado, em corpora de memória compartilhada
- `wiki_get`: lê uma página da wiki por id/caminho ou usa o corpus de memória compartilhada como fallback
- `wiki_apply`: mutações estreitas de síntese/metadados sem cirurgia livre na página
- `wiki_lint`: verificações estruturais, lacunas de procedência, contradições, perguntas em aberto

O plugin também registra um suplemento de corpus de memória não exclusivo, para
que `memory_search` e `memory_get` compartilhados possam alcançar a wiki quando o plugin de memória
ativo oferecer suporte à seleção de corpus.

## Comportamento de prompt e contexto

Quando `context.includeCompiledDigestPrompt` está ativado, seções de prompt de memória
anexam um snapshot compilado compacto de `agent-digest.json`.

Esse snapshot é intencionalmente pequeno e de alto sinal:

- apenas as principais páginas
- apenas as principais afirmações
- contagem de contradições
- contagem de perguntas
- qualificadores de confiança/atualização

Isso é opt-in porque altera o formato do prompt e é principalmente útil para
mecanismos de contexto ou montagem legada de prompt que consumam explicitamente
suplementos de memória.

## Configuração

Coloque a configuração em `plugins.entries.memory-wiki.config`:

```json5
{
  plugins: {
    entries: {
      "memory-wiki": {
        enabled: true,
        config: {
          vaultMode: "isolated",
          vault: {
            path: "~/.openclaw/wiki/main",
            renderMode: "obsidian",
          },
          obsidian: {
            enabled: true,
            useOfficialCli: true,
            vaultName: "OpenClaw Wiki",
            openAfterWrites: false,
          },
          bridge: {
            enabled: false,
            readMemoryArtifacts: true,
            indexDreamReports: true,
            indexDailyNotes: true,
            indexMemoryRoot: true,
            followMemoryEvents: true,
          },
          ingest: {
            autoCompile: true,
            maxConcurrentJobs: 1,
            allowUrlIngest: true,
          },
          search: {
            backend: "shared",
            corpus: "wiki",
          },
          context: {
            includeCompiledDigestPrompt: false,
          },
          render: {
            preserveHumanBlocks: true,
            createBacklinks: true,
            createDashboards: true,
          },
        },
      },
    },
  },
}
```

Principais opções:

- `vaultMode`: `isolated`, `bridge`, `unsafe-local`
- `vault.renderMode`: `native` ou `obsidian`
- `bridge.readMemoryArtifacts`: importar artefatos públicos do plugin de memória ativo
- `bridge.followMemoryEvents`: incluir logs de eventos no modo bridge
- `search.backend`: `shared` ou `local`
- `search.corpus`: `wiki`, `memory` ou `all`
- `context.includeCompiledDigestPrompt`: anexar snapshot compacto do resumo às seções de prompt de memória
- `render.createBacklinks`: gerar blocos relacionados determinísticos
- `render.createDashboards`: gerar páginas de painel

## CLI

`memory-wiki` também expõe uma superfície de CLI de nível superior:

```bash
openclaw wiki status
openclaw wiki doctor
openclaw wiki init
openclaw wiki ingest ./notes/alpha.md
openclaw wiki compile
openclaw wiki lint
openclaw wiki search "alpha"
openclaw wiki get entity.alpha
openclaw wiki apply synthesis "Alpha Summary" --body "..." --source-id source.alpha
openclaw wiki bridge import
openclaw wiki obsidian status
```

Consulte [CLI: wiki](/cli/wiki) para a referência completa de comandos.

## Suporte ao Obsidian

Quando `vault.renderMode` é `obsidian`, o plugin grava Markdown compatível com
Obsidian e pode opcionalmente usar a CLI oficial `obsidian`.

Os fluxos de trabalho compatíveis incluem:

- sondagem de status
- busca no cofre
- abrir uma página
- invocar um comando do Obsidian
- ir para a nota diária

Isso é opcional. A wiki ainda funciona em modo nativo sem Obsidian.

## Fluxo de trabalho recomendado

1. Mantenha seu plugin de memória ativo para recuperação/promoção/dreaming.
2. Ative `memory-wiki`.
3. Comece com o modo `isolated`, a menos que você queira explicitamente o modo bridge.
4. Use `wiki_search` / `wiki_get` quando a procedência importar.
5. Use `wiki_apply` para sínteses estreitas ou atualizações de metadados.
6. Execute `wiki_lint` após alterações significativas.
7. Ative painéis se quiser visibilidade de desatualização/contradições.

## Documentos relacionados

- [Visão geral da memória](/pt-BR/concepts/memory)
- [CLI: memory](/cli/memory)
- [CLI: wiki](/cli/wiki)
- [Visão geral do SDK de plugin](/pt-BR/plugins/sdk-overview)
