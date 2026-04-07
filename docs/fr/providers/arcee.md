---
read_when:
    - Vous voulez utiliser Arcee AI avec OpenClaw
    - Vous avez besoin de la variable d’environnement de clé API ou du choix d’authentification CLI
summary: Configuration d’Arcee AI (authentification + sélection du modèle)
title: Arcee AI
x-i18n:
    generated_at: "2026-04-07T06:52:54Z"
    model: gpt-5.4
    provider: openai
    source_hash: fb04909a708fec08dd2c8c863501b178f098bc4818eaebad38aea264157969d8
    source_path: providers/arcee.md
    workflow: 15
---

# Arcee AI

[Arcee AI](https://arcee.ai) fournit un accès à la famille Trinity de modèles mixture-of-experts via une API compatible OpenAI. Tous les modèles Trinity sont sous licence Apache 2.0.

Les modèles Arcee AI sont accessibles directement via la plateforme Arcee ou via [OpenRouter](/fr/providers/openrouter).

- Fournisseur : `arcee`
- Authentification : `ARCEEAI_API_KEY` (direct) ou `OPENROUTER_API_KEY` (via OpenRouter)
- API : compatible OpenAI
- URL de base : `https://api.arcee.ai/api/v1` (direct) ou `https://openrouter.ai/api/v1` (OpenRouter)

## Démarrage rapide

1. Obtenez une clé API depuis [Arcee AI](https://chat.arcee.ai/) ou [OpenRouter](https://openrouter.ai/keys).

2. Définissez la clé API (recommandé : la stocker pour la Gateway) :

```bash
# Direct (plateforme Arcee)
openclaw onboard --auth-choice arceeai-api-key

# Via OpenRouter
openclaw onboard --auth-choice arceeai-openrouter
```

3. Définissez un modèle par défaut :

```json5
{
  agents: {
    defaults: {
      model: { primary: "arcee/trinity-large-thinking" },
    },
  },
}
```

## Exemple non interactif

```bash
# Direct (plateforme Arcee)
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice arceeai-api-key \
  --arceeai-api-key "$ARCEEAI_API_KEY"

# Via OpenRouter
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice arceeai-openrouter \
  --openrouter-api-key "$OPENROUTER_API_KEY"
```

## Remarque sur l’environnement

Si la Gateway s’exécute comme démon (launchd/systemd), assurez-vous que `ARCEEAI_API_KEY`
(ou `OPENROUTER_API_KEY`) est disponible pour ce processus (par exemple, dans
`~/.openclaw/.env` ou via `env.shellEnv`).

## Catalogue intégré

OpenClaw inclut actuellement ce catalogue Arcee intégré :

| Model ref                      | Nom                    | Entrée | Contexte | Coût (entrée/sortie par 1M) | Notes                                        |
| ------------------------------ | ---------------------- | ------ | -------- | --------------------------- | -------------------------------------------- |
| `arcee/trinity-large-thinking` | Trinity Large Thinking | text   | 256K     | $0.25 / $0.90               | Modèle par défaut ; raisonnement activé      |
| `arcee/trinity-large-preview`  | Trinity Large Preview  | text   | 128K     | $0.25 / $1.00               | Usage général ; 400B de paramètres, 13B actifs |
| `arcee/trinity-mini`           | Trinity Mini 26B       | text   | 128K     | $0.045 / $0.15              | Rapide et économique ; function calling      |

Les mêmes références de modèle fonctionnent pour les configurations directes et OpenRouter (par exemple `arcee/trinity-large-thinking`).

Le preset d’onboarding définit `arcee/trinity-large-thinking` comme modèle par défaut.

## Fonctionnalités prises en charge

- Streaming
- Utilisation d’outils / function calling
- Sortie structurée (mode JSON et schéma JSON)
- Réflexion étendue (Trinity Large Thinking)
