---
read_when:
    - Vous voulez des modèles Z.AI / GLM dans OpenClaw
    - Vous avez besoin d’une configuration simple avec ZAI_API_KEY
summary: Utiliser Z.AI (modèles GLM) avec OpenClaw
title: Z.AI
x-i18n:
    generated_at: "2026-04-08T06:01:03Z"
    model: gpt-5.4
    provider: openai
    source_hash: 66cbd9813ee28d202dcae34debab1b0cf9927793acb00743c1c62b48d9e381f9
    source_path: providers/zai.md
    workflow: 15
---

# Z.AI

Z.AI est la plateforme d’API pour les modèles **GLM**. Elle fournit des API REST pour GLM et utilise des clés API
pour l’authentification. Créez votre clé API dans la console Z.AI. OpenClaw utilise le fournisseur `zai`
avec une clé API Z.AI.

## Configuration de la CLI

```bash
# Configuration générique de clé API avec détection automatique du point de terminaison
openclaw onboard --auth-choice zai-api-key

# Coding Plan Global, recommandé pour les utilisateurs de Coding Plan
openclaw onboard --auth-choice zai-coding-global

# Coding Plan CN (région Chine), recommandé pour les utilisateurs de Coding Plan
openclaw onboard --auth-choice zai-coding-cn

# API générale
openclaw onboard --auth-choice zai-global

# API générale CN (région Chine)
openclaw onboard --auth-choice zai-cn
```

## Extrait de configuration

```json5
{
  env: { ZAI_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "zai/glm-5.1" } } },
}
```

`zai-api-key` permet à OpenClaw de détecter le point de terminaison Z.AI correspondant à partir de la clé et
d’appliquer automatiquement l’URL de base correcte. Utilisez les choix régionaux explicites lorsque
vous souhaitez forcer une surface Coding Plan spécifique ou une surface d’API générale.

## Catalogue GLM inclus

OpenClaw initialise actuellement le fournisseur `zai` inclus avec :

- `glm-5.1`
- `glm-5`
- `glm-5-turbo`
- `glm-5v-turbo`
- `glm-4.7`
- `glm-4.7-flash`
- `glm-4.7-flashx`
- `glm-4.6`
- `glm-4.6v`
- `glm-4.5`
- `glm-4.5-air`
- `glm-4.5-flash`
- `glm-4.5v`

## Remarques

- Les modèles GLM sont disponibles sous la forme `zai/<model>` (exemple : `zai/glm-5`).
- Référence de modèle incluse par défaut : `zai/glm-5.1`
- Les identifiants `glm-5*` inconnus sont quand même résolus de manière différée sur le chemin du fournisseur inclus en synthétisant des métadonnées propres au fournisseur à partir du modèle `glm-4.7` lorsque l’identifiant correspond à la forme actuelle de la famille GLM-5.
- `tool_stream` est activé par défaut pour le streaming des appels d’outils Z.AI. Définissez `agents.defaults.models["zai/<model>"].params.tool_stream` sur `false` pour le désactiver.
- Voir [/providers/glm](/fr/providers/glm) pour un aperçu de la famille de modèles.
- Z.AI utilise l’authentification Bearer avec votre clé API.
