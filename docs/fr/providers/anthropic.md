---
read_when:
    - Vous voulez utiliser des modèles Anthropic dans OpenClaw
summary: Utiliser Anthropic Claude via des clés API ou Claude CLI dans OpenClaw
title: Anthropic
x-i18n:
    generated_at: "2026-04-07T06:53:10Z"
    model: gpt-5.4
    provider: openai
    source_hash: 423928fd36c66729985208d4d3f53aff1f94f63b908df85072988bdc41d5cf46
    source_path: providers/anthropic.md
    workflow: 15
---

# Anthropic (Claude)

Anthropic développe la famille de modèles **Claude** et fournit un accès via une API et
Claude CLI. Dans OpenClaw, les clés API Anthropic et la réutilisation de Claude CLI sont toutes deux
prises en charge. Les profils de jeton Anthropic hérités existants restent honorés à
l'exécution s'ils sont déjà configurés.

<Warning>
Le personnel d'Anthropic nous a indiqué que l'utilisation de Claude CLI de style OpenClaw était de nouveau autorisée, donc
OpenClaw considère la réutilisation de Claude CLI et l'utilisation de `claude -p` comme autorisées pour cette
intégration, sauf si Anthropic publie une nouvelle politique.

Pour les hôtes de passerelle de longue durée, les clés API Anthropic restent la voie de production la plus claire et
la plus prévisible. Si vous utilisez déjà Claude CLI sur l'hôte,
OpenClaw peut réutiliser directement cette connexion.

Documentation publique actuelle d'Anthropic :

- [Référence CLI Claude Code](https://code.claude.com/docs/en/cli-reference)
- [Vue d'ensemble du SDK Claude Agent](https://platform.claude.com/docs/en/agent-sdk/overview)

- [Utiliser Claude Code avec votre forfait Pro ou Max](https://support.claude.com/en/articles/11145838-using-claude-code-with-your-pro-or-max-plan)
- [Utiliser Claude Code avec votre forfait Team ou Enterprise](https://support.anthropic.com/en/articles/11845131-using-claude-code-with-your-team-or-enterprise-plan/)

Si vous voulez le chemin de facturation le plus clair, utilisez plutôt une clé API Anthropic.
OpenClaw prend aussi en charge d'autres options de type abonnement, notamment [OpenAI
Codex](/fr/providers/openai), [Qwen Cloud Coding Plan](/fr/providers/qwen),
[MiniMax Coding Plan](/fr/providers/minimax) et [Z.AI / GLM Coding
Plan](/fr/providers/glm).
</Warning>

## Option A : clé API Anthropic

**Idéal pour :** accès API standard et facturation à l'usage.
Créez votre clé API dans la console Anthropic.

### Configuration via CLI

```bash
openclaw onboard
# choisir : clé API Anthropic

# ou en non interactif
openclaw onboard --anthropic-api-key "$ANTHROPIC_API_KEY"
```

### Extrait de configuration Anthropic

```json5
{
  env: { ANTHROPIC_API_KEY: "sk-ant-..." },
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

## Valeurs par défaut de réflexion (Claude 4.6)

- Les modèles Anthropic Claude 4.6 utilisent par défaut la réflexion `adaptive` dans OpenClaw lorsqu'aucun niveau de réflexion explicite n'est défini.
- Vous pouvez remplacer cela par message (`/think:<level>`) ou dans les paramètres du modèle :
  `agents.defaults.models["anthropic/<model>"].params.thinking`.
- Documentation Anthropic connexe :
  - [Réflexion adaptive](https://platform.claude.com/docs/en/build-with-claude/adaptive-thinking)
  - [Réflexion étendue](https://platform.claude.com/docs/en/build-with-claude/extended-thinking)

## Mode rapide (API Anthropic)

Le basculement partagé `/fast` d'OpenClaw prend aussi en charge le trafic Anthropic public direct, y compris les requêtes authentifiées par clé API et par OAuth envoyées à `api.anthropic.com`.

- `/fast on` est mappé vers `service_tier: "auto"`
- `/fast off` est mappé vers `service_tier: "standard_only"`
- Valeur par défaut dans la configuration :

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-sonnet-4-6": {
          params: { fastMode: true },
        },
      },
    },
  },
}
```

Limites importantes :

- OpenClaw n'injecte des niveaux de service Anthropic que pour les requêtes directes vers `api.anthropic.com`. Si vous faites transiter `anthropic/*` via un proxy ou une passerelle, `/fast` laisse `service_tier` inchangé.
- Les paramètres explicites du modèle Anthropic `serviceTier` ou `service_tier` remplacent la valeur par défaut de `/fast` lorsque les deux sont définis.
- Anthropic signale le niveau effectif dans la réponse sous `usage.service_tier`. Sur les comptes sans capacité Priority Tier, `service_tier: "auto"` peut quand même se résoudre en `standard`.

## Mise en cache des prompts (API Anthropic)

OpenClaw prend en charge la fonctionnalité de mise en cache des prompts d'Anthropic. Ceci est **réservé à l'API** ; l'authentification par jeton Anthropic héritée n'honore pas les paramètres de cache.

### Configuration

Utilisez le paramètre `cacheRetention` dans la configuration de votre modèle :

| Value   | Cache Duration | Description              |
| ------- | -------------- | ------------------------ |
| `none`  | Aucune mise en cache     | Désactiver la mise en cache des prompts   |
| `short` | 5 minutes      | Par défaut pour l'authentification par clé API |
| `long`  | 1 heure         | Cache étendu           |

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": {
          params: { cacheRetention: "long" },
        },
      },
    },
  },
}
```

### Valeurs par défaut

Lors de l'utilisation de l'authentification par clé API Anthropic, OpenClaw applique automatiquement `cacheRetention: "short"` (cache de 5 minutes) à tous les modèles Anthropic. Vous pouvez remplacer cela en définissant explicitement `cacheRetention` dans votre configuration.

### Surcharges `cacheRetention` par agent

Utilisez les paramètres au niveau du modèle comme base, puis remplacez-les pour des agents spécifiques via `agents.list[].params`.

```json5
{
  agents: {
    defaults: {
      model: { primary: "anthropic/claude-opus-4-6" },
      models: {
        "anthropic/claude-opus-4-6": {
          params: { cacheRetention: "long" }, // base pour la plupart des agents
        },
      },
    },
    list: [
      { id: "research", default: true },
      { id: "alerts", params: { cacheRetention: "none" } }, // surcharge pour cet agent uniquement
    ],
  },
}
```

Ordre de fusion de la configuration pour les paramètres liés au cache :

1. `agents.defaults.models["provider/model"].params`
2. `agents.list[].params` (correspondance sur `id`, surcharge par clé)

Cela permet à un agent de conserver un cache de longue durée tandis qu'un autre agent sur le même modèle désactive la mise en cache pour éviter les coûts d'écriture sur un trafic en rafales / à faible réutilisation.

### Notes sur Bedrock Claude

- Les modèles Anthropic Claude sur Bedrock (`amazon-bedrock/*anthropic.claude*`) acceptent le passage direct de `cacheRetention` lorsqu'il est configuré.
- Les modèles Bedrock non Anthropic sont forcés à `cacheRetention: "none"` à l'exécution.
- Les valeurs par défaut intelligentes de la clé API Anthropic initialisent aussi `cacheRetention: "short"` pour les références de modèles Claude sur Bedrock lorsqu'aucune valeur explicite n'est définie.

## Fenêtre de contexte de 1M (bêta Anthropic)

La fenêtre de contexte 1M d'Anthropic est limitée par une bêta. Dans OpenClaw, activez-la par modèle
avec `params.context1m: true` pour les modèles Opus/Sonnet pris en charge.

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": {
          params: { context1m: true },
        },
      },
    },
  },
}
```

OpenClaw la mappe vers `anthropic-beta: context-1m-2025-08-07` sur les requêtes
Anthropic.

Cela ne s'active que lorsque `params.context1m` est explicitement défini sur `true` pour
ce modèle.

Condition requise : Anthropic doit autoriser l'utilisation du contexte long pour cet identifiant.

Remarque : Anthropic rejette actuellement les requêtes bêta `context-1m-*` lors de l'utilisation de
l'authentification par jeton Anthropic héritée (`sk-ant-oat-*`). Si vous configurez
`context1m: true` avec ce mode d'authentification hérité, OpenClaw journalise un avertissement et
revient à la fenêtre de contexte standard en ignorant l'en-tête bêta `context1m`
tout en conservant les bêtas OAuth requis.

## Backend Claude CLI

Le backend groupé Anthropic `claude-cli` est pris en charge dans OpenClaw.

- Le personnel d'Anthropic nous a indiqué que cette utilisation est de nouveau autorisée.
- OpenClaw considère donc la réutilisation de Claude CLI et l'utilisation de `claude -p` comme
  autorisées pour cette intégration, sauf si Anthropic publie une nouvelle politique.
- Les clés API Anthropic restent la voie de production la plus claire pour les hôtes de passerelle
  toujours actifs et pour un contrôle explicite de la facturation côté serveur.
- Les détails de configuration et d'exécution sont dans [/gateway/cli-backends](/fr/gateway/cli-backends).

## Remarques

- La documentation publique Claude Code d'Anthropic documente toujours l'utilisation directe de la CLI telle que
  `claude -p`, et le personnel d'Anthropic nous a indiqué que l'utilisation de Claude CLI de style OpenClaw est
  de nouveau autorisée. Nous considérons cette indication comme établie à moins qu'Anthropic
  ne publie un nouveau changement de politique.
- Le setup-token Anthropic reste disponible dans OpenClaw comme chemin pris en charge d'authentification par jeton, mais OpenClaw préfère désormais la réutilisation de Claude CLI et `claude -p` lorsque c'est disponible.
- Les détails d'authentification + les règles de réutilisation se trouvent dans [/concepts/oauth](/fr/concepts/oauth).

## Dépannage

**Erreurs 401 / jeton soudainement invalide**

- L'authentification par jeton Anthropic peut expirer ou être révoquée.
- Pour une nouvelle configuration, migrez vers une clé API Anthropic.

**Aucune clé API trouvée pour le fournisseur `anthropic`**

- L'authentification est **par agent**. Les nouveaux agents n'héritent pas des clés de l'agent principal.
- Relancez l'onboarding pour cet agent, ou configurez une clé API sur l'hôte de la passerelle,
  puis vérifiez avec `openclaw models status`.

**Aucun identifiant trouvé pour le profil `anthropic:default`**

- Exécutez `openclaw models status` pour voir quel profil d'authentification est actif.
- Relancez l'onboarding, ou configurez une clé API pour ce chemin de profil.

**Aucun profil d'authentification disponible (tous en délai d'attente / indisponibles)**

- Vérifiez `openclaw models status --json` pour `auth.unusableProfiles`.
- Les délais de refroidissement de limitation de débit Anthropic peuvent être ciblés par modèle, donc un modèle Anthropic apparenté
  peut encore être utilisable même si le modèle actuel est en refroidissement.
- Ajoutez un autre profil Anthropic ou attendez la fin du délai.

Plus d'informations : [/gateway/troubleshooting](/fr/gateway/troubleshooting) et [/help/faq](/fr/help/faq).
