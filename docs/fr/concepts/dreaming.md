---
read_when:
    - Vous souhaitez que la promotion de la mémoire s'exécute automatiquement
    - Vous souhaitez comprendre le rôle de chaque phase de dreaming
    - Vous souhaitez ajuster la consolidation sans polluer `MEMORY.md`
summary: Consolidation de la mémoire en arrière-plan avec des phases légères, profondes et REM, plus un Journal des rêves
title: Dreaming (expérimental)
x-i18n:
    generated_at: "2026-04-08T06:51:00Z"
    model: gpt-5.4
    provider: openai
    source_hash: 0254f3b0949158264e583c12f36f2b1a83d1b44dc4da01a1b272422d38e8655d
    source_path: concepts/dreaming.md
    workflow: 15
---

# Dreaming (expérimental)

Dreaming est le système de consolidation de la mémoire en arrière-plan dans `memory-core`.
Il aide OpenClaw à déplacer des signaux forts à court terme vers une mémoire durable tout en
gardant le processus explicable et vérifiable.

Dreaming est **optionnel** et désactivé par défaut.

## Ce que Dreaming écrit

Dreaming conserve deux types de sortie :

- **État machine** dans `memory/.dreams/` (magasin de rappel, signaux de phase, points de contrôle d'ingestion, verrous).
- **Sortie lisible par les humains** dans `DREAMS.md` (ou `dreams.md` existant) et fichiers de rapport de phase facultatifs sous `memory/dreaming/<phase>/YYYY-MM-DD.md`.

La promotion à long terme continue d'écrire uniquement dans `MEMORY.md`.

## Modèle de phase

Dreaming utilise trois phases coopératives :

| Phase | Objectif                                  | Écriture durable  |
| ----- | ----------------------------------------- | ----------------- |
| Légère | Trier et préparer le contenu récent à court terme | Non               |
| Profonde  | Évaluer et promouvoir les candidats durables      | Oui (`MEMORY.md`) |
| REM   | Réfléchir aux thèmes et aux idées récurrentes     | Non               |

Ces phases sont des détails d'implémentation internes, et non des « modes »
distincts configurés par l'utilisateur.

### Phase légère

La phase légère ingère les signaux récents de mémoire quotidienne et les traces de rappel, les déduplique,
et prépare des lignes candidates.

- Lit l'état de rappel à court terme, les fichiers récents de mémoire quotidienne et les transcriptions de session expurgées lorsqu'elles sont disponibles.
- Écrit un bloc géré `## Light Sleep` lorsque le stockage inclut une sortie inline.
- Enregistre des signaux de renforcement pour le classement profond ultérieur.
- N'écrit jamais dans `MEMORY.md`.

### Phase profonde

La phase profonde décide de ce qui devient une mémoire à long terme.

- Classe les candidats à l'aide d'un score pondéré et de seuils de filtrage.
- Exige que `minScore`, `minRecallCount` et `minUniqueQueries` soient atteints.
- Réhydrate des extraits à partir de fichiers quotidiens actifs avant l'écriture, afin d'ignorer les extraits obsolètes ou supprimés.
- Ajoute les entrées promues à `MEMORY.md`.
- Écrit un résumé `## Deep Sleep` dans `DREAMS.md` et écrit éventuellement `memory/dreaming/deep/YYYY-MM-DD.md`.

### Phase REM

La phase REM extrait des motifs et des signaux réflexifs.

- Construit des résumés de thèmes et de réflexions à partir de traces récentes à court terme.
- Écrit un bloc géré `## REM Sleep` lorsque le stockage inclut une sortie inline.
- Enregistre des signaux de renforcement REM utilisés par le classement profond.
- N'écrit jamais dans `MEMORY.md`.

## Ingestion des transcriptions de session

Dreaming peut ingérer des transcriptions de session expurgées dans le corpus de dreaming. Lorsque
des transcriptions sont disponibles, elles sont intégrées à la phase légère en même temps que les signaux de mémoire
quotidienne et les traces de rappel. Le contenu personnel et sensible est expurgé
avant l'ingestion.

## Journal des rêves

Dreaming conserve également un **Journal des rêves** narratif dans `DREAMS.md`.
Après que chaque phase dispose de suffisamment de contenu, `memory-core` exécute un tour de sous-agent
en arrière-plan en mode best-effort (en utilisant le modèle d'exécution par défaut) et ajoute une courte entrée de journal.

Ce journal est destiné à la lecture humaine dans l'interface Dreams, pas à servir de source de promotion.

## Signaux de classement profond

Le classement profond utilise six signaux de base pondérés, plus le renforcement des phases :

| Signal              | Poids | Description                                       |
| ------------------- | ------ | ------------------------------------------------- |
| Fréquence           | 0.24   | Nombre de signaux à court terme accumulés par l'entrée |
| Pertinence          | 0.30   | Qualité moyenne de récupération pour l'entrée           |
| Diversité des requêtes     | 0.15   | Contextes distincts requête/jour qui l'ont fait remonter      |
| Récence             | 0.15   | Score de fraîcheur décroissant dans le temps                      |
| Consolidation       | 0.10   | Force de récurrence sur plusieurs jours                     |
| Richesse conceptuelle | 0.06   | Densité de tags conceptuels à partir de l'extrait/chemin             |

Les occurrences des phases légère et REM ajoutent un petit boost décroissant avec la récence depuis
`memory/.dreams/phase-signals.json`.

## Planification

Lorsqu'il est activé, `memory-core` gère automatiquement une tâche cron pour un balayage complet
de dreaming. Chaque balayage exécute les phases dans l'ordre : légère -> REM -> profonde.

Comportement de cadence par défaut :

| Paramètre              | Par défaut    |
| -------------------- | ----------- |
| `dreaming.frequency` | `0 3 * * *` |

## Démarrage rapide

Activer dreaming :

```json
{
  "plugins": {
    "entries": {
      "memory-core": {
        "config": {
          "dreaming": {
            "enabled": true
          }
        }
      }
    }
  }
}
```

Activer dreaming avec une cadence de balayage personnalisée :

```json
{
  "plugins": {
    "entries": {
      "memory-core": {
        "config": {
          "dreaming": {
            "enabled": true,
            "timezone": "America/Los_Angeles",
            "frequency": "0 */6 * * *"
          }
        }
      }
    }
  }
}
```

## Commande slash

```
/dreaming status
/dreaming on
/dreaming off
/dreaming help
```

## Workflow CLI

Utilisez la promotion CLI pour prévisualiser ou appliquer manuellement :

```bash
openclaw memory promote
openclaw memory promote --apply
openclaw memory promote --limit 5
openclaw memory status --deep
```

La commande manuelle `memory promote` utilise par défaut les seuils de la phase profonde, sauf remplacement
par des indicateurs CLI.

Expliquer pourquoi un candidat spécifique serait ou ne serait pas promu :

```bash
openclaw memory promote-explain "router vlan"
openclaw memory promote-explain "router vlan" --json
```

Prévisualiser les réflexions REM, les vérités candidates et la sortie de promotion profonde sans
rien écrire :

```bash
openclaw memory rem-harness
openclaw memory rem-harness --json
```

## Valeurs par défaut clés

Tous les paramètres se trouvent sous `plugins.entries.memory-core.config.dreaming`.

| Clé         | Par défaut    |
| ----------- | ----------- |
| `enabled`   | `false`     |
| `frequency` | `0 3 * * *` |

La politique des phases, les seuils et le comportement de stockage sont des détails
d'implémentation internes (pas une configuration destinée à l'utilisateur).

Consultez la [référence de configuration de la mémoire](/fr/reference/memory-config#dreaming-experimental)
pour la liste complète des clés.

## Interface Dreams

Lorsqu'il est activé, l'onglet **Dreams** de la Gateway affiche :

- l'état d'activation actuel de dreaming
- le statut au niveau des phases et la présence d'un balayage géré
- les nombres à court terme, à long terme et promus aujourd'hui
- l'heure de la prochaine exécution planifiée
- un lecteur extensible du Journal des rêves reposant sur `doctor.memory.dreamDiary`

## Liens connexes

- [Mémoire](/fr/concepts/memory)
- [Recherche dans la mémoire](/fr/concepts/memory-search)
- [CLI memory](/cli/memory)
- [Référence de configuration de la mémoire](/fr/reference/memory-config)
