---
read_when:
    - Vous voulez comprendre comment fonctionne la mémoire
    - Vous voulez savoir quels fichiers de mémoire écrire
summary: Comment OpenClaw se souvient des choses d’une session à l’autre
title: Vue d’ensemble de la mémoire
x-i18n:
    generated_at: "2026-04-08T06:00:58Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3bb8552341b0b651609edaaae826a22fdc535d240aed4fad4af4b069454004af
    source_path: concepts/memory.md
    workflow: 15
---

# Vue d’ensemble de la mémoire

OpenClaw se souvient des choses en écrivant des **fichiers Markdown simples** dans l’espace de travail de votre agent. Le modèle ne « se souvient » que de ce qui est enregistré sur le disque -- il n’y a pas d’état caché.

## Fonctionnement

Votre agent dispose de trois fichiers liés à la mémoire :

- **`MEMORY.md`** -- mémoire à long terme. Faits durables, préférences et décisions. Chargé au début de chaque session de message direct.
- **`memory/YYYY-MM-DD.md`** -- notes quotidiennes. Contexte courant et observations. Les notes d’aujourd’hui et d’hier sont chargées automatiquement.
- **`DREAMS.md`** (expérimental, facultatif) -- journal des rêves et résumés des passes de rêverie pour relecture humaine.

Ces fichiers se trouvent dans l’espace de travail de l’agent (par défaut `~/.openclaw/workspace`).

<Tip>
Si vous voulez que votre agent se souvienne de quelque chose, demandez-le-lui simplement : "Remember that I
prefer TypeScript." Il l’écrira dans le fichier approprié.
</Tip>

## Outils de mémoire

L’agent dispose de deux outils pour travailler avec la mémoire :

- **`memory_search`** -- trouve des notes pertinentes à l’aide d’une recherche sémantique, même lorsque la formulation diffère de l’original.
- **`memory_get`** -- lit un fichier de mémoire précis ou une plage de lignes.

Les deux outils sont fournis par le plugin de mémoire actif (par défaut : `memory-core`).

## Plugin compagnon Memory Wiki

Si vous voulez que la mémoire durable fonctionne davantage comme une base de connaissances maintenue que comme de simples notes brutes, utilisez le plugin groupé `memory-wiki`.

`memory-wiki` compile les connaissances durables dans un coffre wiki avec :

- une structure de page déterministe
- des affirmations et preuves structurées
- le suivi des contradictions et de la fraîcheur
- des tableaux de bord générés
- des synthèses compilées pour les consommateurs agent/runtime
- des outils natifs du wiki comme `wiki_search`, `wiki_get`, `wiki_apply` et `wiki_lint`

Il ne remplace pas le plugin de mémoire actif. Le plugin de mémoire actif reste responsable du rappel, de la promotion et de la rêverie. `memory-wiki` ajoute à côté une couche de connaissances riche en provenance.

Voir [Memory Wiki](/fr/plugins/memory-wiki).

## Recherche dans la mémoire

Lorsqu’un fournisseur d’embeddings est configuré, `memory_search` utilise une **recherche hybride** -- en combinant la similarité vectorielle (sens sémantique) avec la correspondance par mots-clés (termes exacts comme les identifiants et les symboles de code). Cela fonctionne immédiatement dès que vous disposez d’une clé API pour n’importe quel fournisseur pris en charge.

<Info>
OpenClaw détecte automatiquement votre fournisseur d’embeddings à partir des clés API disponibles. Si vous avez configuré une clé OpenAI, Gemini, Voyage ou Mistral, la recherche en mémoire est activée automatiquement.
</Info>

Pour en savoir plus sur le fonctionnement de la recherche, les options de réglage et la configuration des fournisseurs, consultez
[Memory Search](/fr/concepts/memory-search).

## Backends de mémoire

<CardGroup cols={3}>
<Card title="Intégré (par défaut)" icon="database" href="/fr/concepts/memory-builtin">
Basé sur SQLite. Fonctionne immédiatement avec la recherche par mots-clés, la similarité vectorielle et la recherche hybride. Aucune dépendance supplémentaire.
</Card>
<Card title="QMD" icon="search" href="/fr/concepts/memory-qmd">
Sidecar local-first avec reranking, expansion de requête et possibilité d’indexer des répertoires en dehors de l’espace de travail.
</Card>
<Card title="Honcho" icon="brain" href="/fr/concepts/memory-honcho">
Mémoire intersessions native IA avec modélisation des utilisateurs, recherche sémantique et prise en compte multi-agent. Installation par plugin.
</Card>
</CardGroup>

## Couche wiki de connaissances

<CardGroup cols={1}>
<Card title="Memory Wiki" icon="book" href="/fr/plugins/memory-wiki">
Compile la mémoire durable dans un coffre wiki riche en provenance avec affirmations, tableaux de bord, mode pont et workflows compatibles avec Obsidian.
</Card>
</CardGroup>

## Vidage automatique de la mémoire

Avant que la [compaction](/fr/concepts/compaction) ne résume votre conversation, OpenClaw exécute un tour silencieux qui rappelle à l’agent d’enregistrer le contexte important dans les fichiers de mémoire. C’est activé par défaut -- vous n’avez rien à configurer.

<Tip>
Le vidage de la mémoire évite la perte de contexte pendant la compaction. Si votre agent a des faits importants dans la conversation qui ne sont pas encore écrits dans un fichier, ils seront enregistrés automatiquement avant que le résumé n’ait lieu.
</Tip>

## Rêverie (expérimental)

La rêverie est une passe facultative de consolidation en arrière-plan pour la mémoire. Elle collecte des signaux à court terme, évalue les candidats et ne promeut que les éléments qualifiés vers la mémoire à long terme (`MEMORY.md`).

Elle est conçue pour conserver un signal élevé dans la mémoire à long terme :

- **Sur adhésion** : désactivée par défaut.
- **Planifiée** : lorsqu’elle est activée, `memory-core` gère automatiquement une tâche cron récurrente pour une passe complète de rêverie.
- **À seuil** : les promotions doivent passer des seuils de score, de fréquence de rappel et de diversité des requêtes.
- **Révisable** : les résumés de phase et les entrées du journal sont écrits dans `DREAMS.md` pour relecture humaine.

Pour le comportement des phases, les signaux de scoring et les détails du journal des rêves, consultez
[Dreaming (experimental)](/fr/concepts/dreaming).

## CLI

```bash
openclaw memory status          # Vérifier l’état de l’index et le fournisseur
openclaw memory search "query"  # Rechercher depuis la ligne de commande
openclaw memory index --force   # Reconstruire l’index
```

## Pour aller plus loin

- [Builtin Memory Engine](/fr/concepts/memory-builtin) -- backend SQLite par défaut
- [QMD Memory Engine](/fr/concepts/memory-qmd) -- sidecar local-first avancé
- [Honcho Memory](/fr/concepts/memory-honcho) -- mémoire intersessions native IA
- [Memory Wiki](/fr/plugins/memory-wiki) -- coffre de connaissances compilé et outils natifs du wiki
- [Memory Search](/fr/concepts/memory-search) -- pipeline de recherche, fournisseurs et
  réglage
- [Dreaming (experimental)](/fr/concepts/dreaming) -- promotion en arrière-plan
  du rappel à court terme vers la mémoire à long terme
- [Référence de configuration de la mémoire](/fr/reference/memory-config) -- tous les paramètres de configuration
- [Compaction](/fr/concepts/compaction) -- comment la compaction interagit avec la mémoire
