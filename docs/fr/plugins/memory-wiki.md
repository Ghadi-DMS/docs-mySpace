---
read_when:
    - Vous voulez des connaissances persistantes au-delà de simples notes MEMORY.md
    - Vous configurez le plugin groupé memory-wiki
    - Vous voulez comprendre wiki_search, wiki_get ou le mode bridge
summary: 'memory-wiki : coffre de connaissances compilé avec provenance, affirmations, tableaux de bord et mode bridge'
title: Wiki mémoire
x-i18n:
    generated_at: "2026-04-08T06:01:13Z"
    model: gpt-5.4
    provider: openai
    source_hash: b78dd6a4ef4451dae6b53197bf0c7c2a2ba846b08e4a3a93c1026366b1598d82
    source_path: plugins/memory-wiki.md
    workflow: 15
---

# Wiki mémoire

`memory-wiki` est un plugin groupé qui transforme la mémoire durable en un
coffre de connaissances compilé.

Il **ne** remplace **pas** le plugin de mémoire actif. Le plugin de mémoire actif
reste responsable du rappel, de la promotion, de l’indexation et du dreaming. `memory-wiki`
se place à côté de lui et compile les connaissances durables dans un wiki navigable avec des pages déterministes,
des affirmations structurées, de la provenance, des tableaux de bord et des condensés lisibles par machine.

Utilisez-le lorsque vous voulez que la mémoire se comporte davantage comme une couche de connaissances maintenue
et moins comme un empilement de fichiers Markdown.

## Ce qu’il ajoute

- Un coffre wiki dédié avec une disposition de pages déterministe
- Des métadonnées structurées d’affirmations et de preuves, pas seulement du texte
- Une provenance, une confiance, des contradictions et des questions ouvertes au niveau des pages
- Des condensés compilés pour les consommateurs agent/runtime
- Des outils natifs au wiki pour search/get/apply/lint
- Un mode bridge facultatif qui importe les artefacts publics du plugin de mémoire actif
- Un mode de rendu compatible Obsidian et une intégration CLI facultatifs

## Comment il s’intègre à la mémoire

Voyez la séparation ainsi :

| Couche                                                  | Responsable de                                                                              |
| ------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| Plugin de mémoire actif (`memory-core`, QMD, Honcho, etc.) | Rappel, recherche sémantique, promotion, dreaming, runtime de mémoire                       |
| `memory-wiki`                                           | Pages wiki compilées, synthèses riches en provenance, tableaux de bord, search/get/apply spécifiques au wiki |

Si le plugin de mémoire actif expose des artefacts de rappel partagés, OpenClaw peut rechercher
dans les deux couches en un seul passage avec `memory_search corpus=all`.

Lorsque vous avez besoin d’un classement spécifique au wiki, de provenance ou d’un accès direct aux pages, utilisez plutôt les
outils natifs au wiki.

## Modes de coffre

`memory-wiki` prend en charge trois modes de coffre :

### `isolated`

Propre coffre, propres sources, sans dépendance à `memory-core`.

Utilisez-le lorsque vous voulez que le wiki soit son propre magasin de connaissances organisé.

### `bridge`

Lit les artefacts mémoire publics et les événements de mémoire du plugin de mémoire actif
via des points d’intégration publics du plugin SDK.

Utilisez-le lorsque vous voulez que le wiki compile et organise les
artefacts exportés du plugin de mémoire sans accéder aux composants internes privés du plugin.

Le mode bridge peut indexer :

- les artefacts mémoire exportés
- les rapports de rêve
- les notes quotidiennes
- les fichiers racine de la mémoire
- les journaux d’événements mémoire

### `unsafe-local`

Échappatoire explicite sur la même machine pour les chemins privés locaux.

Ce mode est volontairement expérimental et non portable. Utilisez-le uniquement si vous
comprenez la frontière de confiance et avez précisément besoin d’un accès au système de fichiers local que
le mode bridge ne peut pas fournir.

## Structure du coffre

Le plugin initialise un coffre comme ceci :

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

Le contenu géré reste à l’intérieur de blocs générés. Les blocs de notes humains sont préservés.

Les principaux groupes de pages sont :

- `sources/` pour les matériaux bruts importés et les pages adossées au bridge
- `entities/` pour les éléments durables, personnes, systèmes, projets et objets
- `concepts/` pour les idées, abstractions, modèles et politiques
- `syntheses/` pour les résumés compilés et les consolidations maintenues
- `reports/` pour les tableaux de bord générés

## Affirmations structurées et preuves

Les pages peuvent contenir des `claims` dans le frontmatter structuré, pas seulement du texte libre.

Chaque affirmation peut inclure :

- `id`
- `text`
- `status`
- `confidence`
- `evidence[]`
- `updatedAt`

Les entrées de preuve peuvent inclure :

- `sourceId`
- `path`
- `lines`
- `weight`
- `note`
- `updatedAt`

C’est ce qui fait que le wiki agit davantage comme une couche de croyances que comme un simple
dépôt de notes passif. Les affirmations peuvent être suivies, évaluées, contestées et résolues à partir des sources.

## Pipeline de compilation

L’étape de compilation lit les pages du wiki, normalise les résumés et émet des
artefacts stables orientés machine sous :

- `.openclaw-wiki/cache/agent-digest.json`
- `.openclaw-wiki/cache/claims.jsonl`

Ces condensés existent pour que les agents et le code runtime n’aient pas à analyser les
pages Markdown.

La sortie compilée alimente aussi :

- l’indexation wiki de premier passage pour les flux search/get
- la recherche de l’identifiant d’affirmation jusqu’à la page propriétaire
- des compléments d’invite compacts
- la génération de rapports/tableaux de bord

## Tableaux de bord et rapports de santé

Lorsque `render.createDashboards` est activé, la compilation maintient des tableaux de bord sous
`reports/`.

Les rapports intégrés incluent :

- `reports/open-questions.md`
- `reports/contradictions.md`
- `reports/low-confidence.md`
- `reports/claim-health.md`
- `reports/stale-pages.md`

Ces rapports suivent des éléments comme :

- les groupes de notes de contradiction
- les groupes d’affirmations concurrentes
- les affirmations sans preuve structurée
- les pages et affirmations à faible confiance
- l’ancienneté obsolète ou inconnue
- les pages avec des questions non résolues

## Recherche et récupération

`memory-wiki` prend en charge deux backends de recherche :

- `shared` : utiliser le flux de recherche mémoire partagé lorsqu’il est disponible
- `local` : rechercher dans le wiki localement

Il prend également en charge trois corpus :

- `wiki`
- `memory`
- `all`

Comportement important :

- `wiki_search` et `wiki_get` utilisent les condensés compilés comme premier passage lorsque c’est possible
- les identifiants d’affirmation peuvent être résolus jusqu’à la page propriétaire
- les affirmations contestées/obsolètes/récentes influencent le classement
- les libellés de provenance peuvent être conservés dans les résultats

Règle pratique :

- utilisez `memory_search corpus=all` pour un passage large de rappel
- utilisez `wiki_search` + `wiki_get` lorsque le classement spécifique au wiki,
  la provenance ou la structure de croyance au niveau de la page vous importent

## Outils d’agent

Le plugin enregistre ces outils :

- `wiki_status`
- `wiki_search`
- `wiki_get`
- `wiki_apply`
- `wiki_lint`

Ce qu’ils font :

- `wiki_status` : mode de coffre actuel, santé, disponibilité de la CLI Obsidian
- `wiki_search` : rechercher dans les pages wiki et, si configuré, dans les corpus mémoire partagés
- `wiki_get` : lire une page wiki par id/chemin ou revenir au corpus mémoire partagé
- `wiki_apply` : mutations ciblées de synthèse/métadonnées sans chirurgie libre des pages
- `wiki_lint` : vérifications structurelles, lacunes de provenance, contradictions, questions ouvertes

Le plugin enregistre également un complément de corpus mémoire non exclusif, afin que
`memory_search` et `memory_get` partagés puissent atteindre le wiki lorsque le plugin de mémoire actif
prend en charge la sélection de corpus.

## Comportement de l’invite et du contexte

Lorsque `context.includeCompiledDigestPrompt` est activé, les sections d’invite mémoire
ajoutent un instantané compilé compact depuis `agent-digest.json`.

Cet instantané est volontairement petit et à fort signal :

- pages principales uniquement
- principales affirmations uniquement
- nombre de contradictions
- nombre de questions
- qualificatifs de confiance/ancienneté

Ceci est opt-in car cela modifie la forme de l’invite et est surtout utile pour les
moteurs de contexte ou l’assemblage d’invites hérité qui consomme explicitement des compléments mémoire.

## Configuration

Placez la configuration sous `plugins.entries.memory-wiki.config` :

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

Principaux paramètres :

- `vaultMode` : `isolated`, `bridge`, `unsafe-local`
- `vault.renderMode` : `native` ou `obsidian`
- `bridge.readMemoryArtifacts` : importer les artefacts publics du plugin de mémoire actif
- `bridge.followMemoryEvents` : inclure les journaux d’événements en mode bridge
- `search.backend` : `shared` ou `local`
- `search.corpus` : `wiki`, `memory` ou `all`
- `context.includeCompiledDigestPrompt` : ajouter un instantané compact du condensé aux sections d’invite mémoire
- `render.createBacklinks` : générer des blocs liés déterministes
- `render.createDashboards` : générer des pages de tableau de bord

## CLI

`memory-wiki` expose également une interface CLI de premier niveau :

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

Consultez [CLI : wiki](/cli/wiki) pour la référence complète des commandes.

## Prise en charge d’Obsidian

Lorsque `vault.renderMode` est `obsidian`, le plugin écrit du
Markdown compatible Obsidian et peut éventuellement utiliser la CLI officielle `obsidian`.

Les flux de travail pris en charge incluent :

- la vérification d’état
- la recherche dans le coffre
- l’ouverture d’une page
- l’invocation d’une commande Obsidian
- l’accès direct à la note quotidienne

Ceci est facultatif. Le wiki fonctionne toujours en mode natif sans Obsidian.

## Flux de travail recommandé

1. Conservez votre plugin de mémoire actif pour le rappel/la promotion/le dreaming.
2. Activez `memory-wiki`.
3. Commencez par le mode `isolated`, sauf si vous voulez explicitement le mode bridge.
4. Utilisez `wiki_search` / `wiki_get` lorsque la provenance compte.
5. Utilisez `wiki_apply` pour des synthèses ciblées ou des mises à jour de métadonnées.
6. Exécutez `wiki_lint` après des changements significatifs.
7. Activez les tableaux de bord si vous voulez une visibilité sur l’obsolescence/les contradictions.

## Documentation associée

- [Vue d’ensemble de la mémoire](/fr/concepts/memory)
- [CLI : memory](/cli/memory)
- [CLI : wiki](/cli/wiki)
- [Vue d’ensemble du SDK de plugin](/fr/plugins/sdk-overview)
