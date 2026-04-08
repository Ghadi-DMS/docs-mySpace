---
read_when:
    - Vous voulez des modèles GLM dans OpenClaw
    - Vous avez besoin de la convention de nommage des modèles et de la configuration
summary: Aperçu de la famille de modèles GLM + comment l’utiliser dans OpenClaw
title: Modèles GLM
x-i18n:
    generated_at: "2026-04-08T06:00:55Z"
    model: gpt-5.4
    provider: openai
    source_hash: 79a55acfa139847b4b85dbc09f1068cbd2febb1e49f984a23ea9e3b43bc910eb
    source_path: providers/glm.md
    workflow: 15
---

# Modèles GLM

GLM est une **famille de modèles** (et non une entreprise) disponible via la plateforme Z.AI. Dans OpenClaw, les modèles GLM
sont accessibles via le fournisseur `zai` et des identifiants de modèle comme `zai/glm-5`.

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

## Modèles GLM actuellement inclus

OpenClaw initialise actuellement le fournisseur `zai` inclus avec ces références GLM :

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

- Les versions et la disponibilité de GLM peuvent changer ; consultez la documentation de Z.AI pour les dernières informations.
- La référence de modèle incluse par défaut est `zai/glm-5.1`.
- Pour plus de détails sur le fournisseur, voir [/providers/zai](/fr/providers/zai).
