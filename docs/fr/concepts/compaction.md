---
read_when:
    - Vous voulez comprendre la compaction automatique et /compact
    - Vous déboguez de longues sessions qui atteignent les limites de contexte
summary: Comment OpenClaw résume les longues conversations pour rester dans les limites du modèle
title: Compaction
x-i18n:
    generated_at: "2026-04-08T02:14:13Z"
    model: gpt-5.4
    provider: openai
    source_hash: e6590b82a8c3a9c310998d653459ca4d8612495703ca0a8d8d306d7643142fd1
    source_path: concepts/compaction.md
    workflow: 15
---

# Compaction

Chaque modèle possède une fenêtre de contexte -- le nombre maximal de tokens qu’il peut traiter.
Lorsqu’une conversation approche cette limite, OpenClaw **compacte** les anciens messages
en un résumé afin que le chat puisse continuer.

## Fonctionnement

1. Les anciens tours de conversation sont résumés en une entrée compacte.
2. Le résumé est enregistré dans la transcription de la session.
3. Les messages récents sont conservés intacts.

Lorsque OpenClaw divise l’historique en blocs de compaction, il conserve les appels
d’outils de l’assistant associés à leurs entrées `toolResult` correspondantes. Si un point
de découpe tombe au milieu d’un bloc d’outil, OpenClaw déplace la limite afin que la paire reste ensemble et
que la fin actuelle non résumée soit préservée.

L’historique complet de la conversation reste sur le disque. La compaction modifie seulement ce que le
modèle voit au tour suivant.

## Compaction automatique

La compaction automatique est activée par défaut. Elle s’exécute lorsque la session approche de la limite
de contexte, ou lorsque le modèle renvoie une erreur de dépassement de contexte (dans ce cas,
OpenClaw compacte et réessaie). Les signatures de dépassement typiques incluent
`request_too_large`, `context length exceeded`, `input exceeds the maximum
number of tokens`, `input token count exceeds the maximum number of input
tokens`, `input is too long for the model` et `ollama error: context length
exceeded`.

<Info>
Avant la compaction, OpenClaw rappelle automatiquement à l’agent d’enregistrer les notes importantes
dans des fichiers de [mémoire](/fr/concepts/memory). Cela évite la perte de contexte.
</Info>

Utilisez le paramètre `agents.defaults.compaction` dans votre `openclaw.json` pour configurer le comportement de compaction (mode, tokens cibles, etc.).
La synthèse de compaction préserve les identifiants opaques par défaut (`identifierPolicy: "strict"`). Vous pouvez remplacer ce comportement avec `identifierPolicy: "off"` ou fournir un texte personnalisé avec `identifierPolicy: "custom"` et `identifierInstructions`.

Vous pouvez éventuellement spécifier un modèle différent pour la synthèse de compaction via `agents.defaults.compaction.model`. Cela est utile lorsque votre modèle principal est un modèle local ou de petite taille et que vous voulez que les résumés de compaction soient produits par un modèle plus performant. La surcharge accepte toute chaîne `provider/model-id` :

```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "model": "openrouter/anthropic/claude-sonnet-4-6"
      }
    }
  }
}
```

Cela fonctionne aussi avec des modèles locaux, par exemple un second modèle Ollama dédié à la synthèse ou un spécialiste de la compaction affiné :

```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "model": "ollama/llama3.1:8b"
      }
    }
  }
}
```

Lorsqu’il n’est pas défini, la compaction utilise le modèle principal de l’agent.

## Fournisseurs de compaction enfichables

Les plugins peuvent enregistrer un fournisseur de compaction personnalisé via `registerCompactionProvider()` sur l’API du plugin. Lorsqu’un fournisseur est enregistré et configuré, OpenClaw lui délègue la synthèse au lieu du pipeline LLM intégré.

Pour utiliser un fournisseur enregistré, définissez l’identifiant du fournisseur dans votre configuration :

```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "provider": "my-provider"
      }
    }
  }
}
```

Définir un `provider` force automatiquement `mode: "safeguard"`. Les fournisseurs reçoivent les mêmes instructions de compaction et la même politique de préservation des identifiants que le chemin intégré, et OpenClaw préserve toujours le contexte suffixe des tours récents et des tours scindés après la sortie du fournisseur. Si le fournisseur échoue ou renvoie un résultat vide, OpenClaw revient à la synthèse LLM intégrée.

## Compaction automatique (activée par défaut)

Lorsqu’une session approche ou dépasse la fenêtre de contexte du modèle, OpenClaw déclenche la compaction automatique et peut réessayer la requête d’origine en utilisant le contexte compacté.

Vous verrez :

- `🧹 Auto-compaction complete` en mode verbeux
- `/status` affichant `🧹 Compactions: <count>`

Avant la compaction, OpenClaw peut exécuter un tour de **vidage silencieux de la mémoire** pour stocker
des notes persistantes sur le disque. Consultez [Mémoire](/fr/concepts/memory) pour plus de détails et la configuration.

## Compaction manuelle

Tapez `/compact` dans n’importe quel chat pour forcer une compaction. Ajoutez des instructions pour guider
le résumé :

```
/compact Focus on the API design decisions
```

## Utiliser un modèle différent

Par défaut, la compaction utilise le modèle principal de votre agent. Vous pouvez utiliser un modèle plus
performant pour obtenir de meilleurs résumés :

```json5
{
  agents: {
    defaults: {
      compaction: {
        model: "openrouter/anthropic/claude-sonnet-4-6",
      },
    },
  },
}
```

## Avis de démarrage de la compaction

Par défaut, la compaction s’exécute silencieusement. Pour afficher un bref avis lorsque la compaction
commence, activez `notifyUser` :

```json5
{
  agents: {
    defaults: {
      compaction: {
        notifyUser: true,
      },
    },
  },
}
```

Lorsqu’il est activé, l’utilisateur voit un court message (par exemple, "Compacting
context...") au début de chaque exécution de compaction.

## Compaction vs élagage

|                  | Compaction                    | Élagage                          |
| ---------------- | ----------------------------- | -------------------------------- |
| **Ce qu’il fait** | Résume les anciennes conversations | Supprime les anciens résultats d’outils |
| **Enregistré ?**       | Oui (dans la transcription de session)   | Non (en mémoire uniquement, par requête) |
| **Portée**        | Conversation entière           | Résultats d’outils uniquement                |

[L’élagage de session](/fr/concepts/session-pruning) est un complément plus léger qui
supprime la sortie des outils sans résumer.

## Dépannage

**Compaction trop fréquente ?** La fenêtre de contexte du modèle est peut-être petite, ou les sorties
des outils peuvent être volumineuses. Essayez d’activer
[l’élagage de session](/fr/concepts/session-pruning).

**Le contexte semble obsolète après la compaction ?** Utilisez `/compact Focus on <topic>` pour
guider le résumé, ou activez le [vidage de la mémoire](/fr/concepts/memory) afin que les notes
persistent.

**Besoin de repartir de zéro ?** `/new` démarre une nouvelle session sans compaction.

Pour la configuration avancée (tokens de réserve, préservation des identifiants, moteurs de
contexte personnalisés, compaction côté serveur OpenAI), consultez la
[Analyse approfondie de la gestion de session](/fr/reference/session-management-compaction).

## Lié

- [Session](/fr/concepts/session) — gestion et cycle de vie des sessions
- [Élagage de session](/fr/concepts/session-pruning) — suppression des résultats d’outils
- [Contexte](/fr/concepts/context) — comment le contexte est construit pour les tours de l’agent
- [Hooks](/fr/automation/hooks) — hooks du cycle de vie de la compaction (before_compaction, after_compaction)
