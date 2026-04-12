---
read_when:
    - Vous créez un Plugin OpenClaw
    - Vous devez livrer un schéma de configuration de Plugin ou déboguer des erreurs de validation de Plugin
summary: Manifeste de Plugin + exigences du schéma JSON (validation stricte de la configuration)
title: Manifeste de Plugin
x-i18n:
    generated_at: "2026-04-12T23:28:44Z"
    model: gpt-5.4
    provider: openai
    source_hash: 93b57c7373e4ccd521b10945346db67991543bd2bed4cc8b6641e1f215b48579
    source_path: plugins/manifest.md
    workflow: 15
---

# Manifeste de Plugin (`openclaw.plugin.json`)

Cette page concerne uniquement le **manifeste de Plugin natif OpenClaw**.

Pour les formats de bundles compatibles, voir [Plugin bundles](/fr/plugins/bundles).

Les formats de bundle compatibles utilisent des fichiers de manifeste différents :

- Bundle Codex : `.codex-plugin/plugin.json`
- Bundle Claude : `.claude-plugin/plugin.json` ou la disposition par défaut des composants Claude
  sans manifeste
- Bundle Cursor : `.cursor-plugin/plugin.json`

OpenClaw détecte automatiquement ces dispositions de bundle également, mais elles ne sont pas validées
par rapport au schéma `openclaw.plugin.json` décrit ici.

Pour les bundles compatibles, OpenClaw lit actuellement les métadonnées du bundle ainsi que les racines de
Skills déclarées, les racines de commandes Claude, les valeurs par défaut `settings.json` des bundles Claude,
les valeurs par défaut LSP des bundles Claude, et les packs de hooks pris en charge lorsque la disposition correspond
aux attentes du runtime OpenClaw.

Chaque Plugin natif OpenClaw **doit** fournir un fichier `openclaw.plugin.json` à la
**racine du Plugin**. OpenClaw utilise ce manifeste pour valider la configuration
**sans exécuter le code du Plugin**. Les manifestes manquants ou invalides sont traités comme
des erreurs de Plugin et bloquent la validation de la configuration.

Voir le guide complet du système de Plugins : [Plugins](/fr/tools/plugin).
Pour le modèle natif de capacités et les recommandations actuelles de compatibilité externe :
[Capability model](/fr/plugins/architecture#public-capability-model).

## Rôle de ce fichier

`openclaw.plugin.json` correspond aux métadonnées qu’OpenClaw lit avant de charger le
code de votre Plugin.

Utilisez-le pour :

- l’identité du Plugin
- la validation de la configuration
- les métadonnées d’authentification et d’onboarding qui doivent être disponibles sans démarrer le runtime du Plugin
- les indications d’activation peu coûteuses que les surfaces du plan de contrôle peuvent inspecter avant le chargement du runtime
- les descripteurs de configuration peu coûteux que les surfaces de configuration/onboarding peuvent inspecter avant le chargement du runtime
- les métadonnées d’alias et d’activation automatique qui doivent être résolues avant le chargement du runtime du Plugin
- les métadonnées abrégées de propriété de famille de modèles qui doivent auto-activer le
  Plugin avant le chargement du runtime
- les instantanés statiques de propriété de capacités utilisés pour le câblage de compatibilité intégré et la couverture des contrats
- les métadonnées de configuration spécifiques aux canaux qui doivent être fusionnées dans les surfaces de catalogue et de validation
  sans charger le runtime
- les indications d’interface utilisateur pour la configuration

Ne l’utilisez pas pour :

- enregistrer le comportement du runtime
- déclarer des points d’entrée de code
- les métadonnées d’installation npm

Ces éléments relèvent de votre code de Plugin et de `package.json`.

## Exemple minimal

```json
{
  "id": "voice-call",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

## Exemple enrichi

```json
{
  "id": "openrouter",
  "name": "OpenRouter",
  "description": "Plugin fournisseur OpenRouter",
  "version": "1.0.0",
  "providers": ["openrouter"],
  "modelSupport": {
    "modelPrefixes": ["router-"]
  },
  "cliBackends": ["openrouter-cli"],
  "providerAuthEnvVars": {
    "openrouter": ["OPENROUTER_API_KEY"]
  },
  "providerAuthAliases": {
    "openrouter-coding": "openrouter"
  },
  "channelEnvVars": {
    "openrouter-chatops": ["OPENROUTER_CHATOPS_TOKEN"]
  },
  "providerAuthChoices": [
    {
      "provider": "openrouter",
      "method": "api-key",
      "choiceId": "openrouter-api-key",
      "choiceLabel": "Clé d’API OpenRouter",
      "groupId": "openrouter",
      "groupLabel": "OpenRouter",
      "optionKey": "openrouterApiKey",
      "cliFlag": "--openrouter-api-key",
      "cliOption": "--openrouter-api-key <key>",
      "cliDescription": "Clé d’API OpenRouter",
      "onboardingScopes": ["text-inference"]
    }
  ],
  "uiHints": {
    "apiKey": {
      "label": "Clé d’API",
      "placeholder": "sk-or-v1-...",
      "sensitive": true
    }
  },
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "apiKey": {
        "type": "string"
      }
    }
  }
}
```

## Référence des champs de premier niveau

| Field                               | Required | Type                             | What it means                                                                                                                                                                                                |
| ----------------------------------- | -------- | -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `id`                                | Yes      | `string`                         | ID canonique du Plugin. Il s’agit de l’ID utilisé dans `plugins.entries.<id>`.                                                                                                                              |
| `configSchema`                      | Yes      | `object`                         | Schéma JSON inline pour la configuration de ce Plugin.                                                                                                                                                       |
| `enabledByDefault`                  | No       | `true`                           | Marque un Plugin intégré comme activé par défaut. Omettez-le, ou définissez toute valeur autre que `true`, pour laisser le Plugin désactivé par défaut.                                                   |
| `legacyPluginIds`                   | No       | `string[]`                       | IDs hérités qui se normalisent vers cet ID canonique de Plugin.                                                                                                                                              |
| `autoEnableWhenConfiguredProviders` | No       | `string[]`                       | IDs de fournisseurs qui doivent auto-activer ce Plugin lorsque l’authentification, la configuration ou les références de modèle les mentionnent.                                                            |
| `kind`                              | No       | `"memory"` \| `"context-engine"` | Déclare un type de Plugin exclusif utilisé par `plugins.slots.*`.                                                                                                                                            |
| `channels`                          | No       | `string[]`                       | IDs de canaux détenus par ce Plugin. Utilisés pour la découverte et la validation de configuration.                                                                                                          |
| `providers`                         | No       | `string[]`                       | IDs de fournisseurs détenus par ce Plugin.                                                                                                                                                                   |
| `modelSupport`                      | No       | `object`                         | Métadonnées abrégées, détenues par le manifeste, sur les familles de modèles, utilisées pour charger automatiquement le Plugin avant le runtime.                                                            |
| `cliBackends`                       | No       | `string[]`                       | IDs de backends d’inférence CLI détenus par ce Plugin. Utilisés pour l’auto-activation au démarrage à partir de références de configuration explicites.                                                    |
| `commandAliases`                    | No       | `object[]`                       | Noms de commandes détenus par ce Plugin qui doivent produire une configuration sensible au Plugin et des diagnostics CLI avant le chargement du runtime.                                                    |
| `providerAuthEnvVars`               | No       | `Record<string, string[]>`       | Métadonnées d’environnement d’authentification fournisseur peu coûteuses qu’OpenClaw peut inspecter sans charger le code du Plugin.                                                                         |
| `providerAuthAliases`               | No       | `Record<string, string>`         | IDs de fournisseurs qui doivent réutiliser un autre ID de fournisseur pour la recherche d’authentification, par exemple un fournisseur de code qui partage la clé d’API et les profils d’authentification du fournisseur de base. |
| `channelEnvVars`                    | No       | `Record<string, string[]>`       | Métadonnées d’environnement de canal peu coûteuses qu’OpenClaw peut inspecter sans charger le code du Plugin. Utilisez-les pour les surfaces de configuration ou d’authentification de canaux pilotées par l’environnement que les helpers génériques de démarrage/configuration doivent voir. |
| `providerAuthChoices`               | No       | `object[]`                       | Métadonnées de choix d’authentification peu coûteuses pour les sélecteurs d’onboarding, la résolution du fournisseur préféré et le câblage simple des flags CLI.                                           |
| `activation`                        | No       | `object`                         | Indications d’activation peu coûteuses pour le chargement déclenché par fournisseur, commande, canal, route et capacité. Métadonnées uniquement ; le runtime du Plugin reste propriétaire du comportement réel. |
| `setup`                             | No       | `object`                         | Descripteurs de configuration/onboarding peu coûteux que les surfaces de découverte et de configuration peuvent inspecter sans charger le runtime du Plugin.                                               |
| `contracts`                         | No       | `object`                         | Instantané statique des capacités intégrées pour la parole, la transcription temps réel, la voix temps réel, la compréhension des médias, la génération d’images, la génération musicale, la génération vidéo, la récupération web, la recherche web et la propriété des outils. |
| `channelConfigs`                    | No       | `Record<string, object>`         | Métadonnées de configuration de canal détenues par le manifeste, fusionnées dans les surfaces de découverte et de validation avant le chargement du runtime.                                               |
| `skills`                            | No       | `string[]`                       | Répertoires de Skills à charger, relatifs à la racine du Plugin.                                                                                                                                             |
| `name`                              | No       | `string`                         | Nom de Plugin lisible par un humain.                                                                                                                                                                         |
| `description`                       | No       | `string`                         | Bref résumé affiché dans les surfaces du Plugin.                                                                                                                                                             |
| `version`                           | No       | `string`                         | Version informative du Plugin.                                                                                                                                                                               |
| `uiHints`                           | No       | `Record<string, object>`         | Libellés d’interface, placeholders et indications de sensibilité pour les champs de configuration.                                                                                                          |

## Référence `providerAuthChoices`

Chaque entrée `providerAuthChoices` décrit un choix d’onboarding ou d’authentification.
OpenClaw lit cela avant le chargement du runtime du fournisseur.

| Champ                 | Obligatoire | Type                                            | Ce que cela signifie                                                                                   |
| --------------------- | ----------- | ----------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| `provider`            | Oui         | `string`                                        | ID du fournisseur auquel ce choix appartient.                                                          |
| `method`              | Oui         | `string`                                        | ID de méthode d’authentification vers lequel router.                                                   |
| `choiceId`            | Oui         | `string`                                        | ID de choix d’authentification stable utilisé par les flux d’onboarding et de CLI.                    |
| `choiceLabel`         | Non         | `string`                                        | Libellé destiné à l’utilisateur. S’il est omis, OpenClaw utilise `choiceId` comme valeur de repli.   |
| `choiceHint`          | Non         | `string`                                        | Court texte d’aide pour le sélecteur.                                                                  |
| `assistantPriority`   | Non         | `number`                                        | Les valeurs les plus basses sont triées plus tôt dans les sélecteurs interactifs pilotés par l’assistant. |
| `assistantVisibility` | Non         | `"visible"` \| `"manual-only"`                  | Masque ce choix dans les sélecteurs de l’assistant tout en autorisant sa sélection manuelle en CLI.   |
| `deprecatedChoiceIds` | Non         | `string[]`                                      | IDs de choix hérités qui doivent rediriger les utilisateurs vers ce choix de remplacement.            |
| `groupId`             | Non         | `string`                                        | ID de groupe facultatif pour regrouper des choix liés.                                                 |
| `groupLabel`          | Non         | `string`                                        | Libellé destiné à l’utilisateur pour ce groupe.                                                        |
| `groupHint`           | Non         | `string`                                        | Court texte d’aide pour le groupe.                                                                     |
| `optionKey`           | Non         | `string`                                        | Clé d’option interne pour les flux d’authentification simples à un seul flag.                         |
| `cliFlag`             | Non         | `string`                                        | Nom du flag CLI, par exemple `--openrouter-api-key`.                                                   |
| `cliOption`           | Non         | `string`                                        | Forme complète de l’option CLI, par exemple `--openrouter-api-key <key>`.                              |
| `cliDescription`      | Non         | `string`                                        | Description utilisée dans l’aide CLI.                                                                  |
| `onboardingScopes`    | Non         | `Array<"text-inference" \| "image-generation">` | Sur quelles surfaces d’onboarding ce choix doit apparaître. S’il est omis, la valeur par défaut est `["text-inference"]`. |

## Référence `commandAliases`

Utilisez `commandAliases` lorsqu’un Plugin possède un nom de commande runtime que les utilisateurs peuvent
mettre par erreur dans `plugins.allow` ou essayer d’exécuter comme commande CLI racine. OpenClaw
utilise ces métadonnées pour les diagnostics sans importer le code runtime du Plugin.

```json
{
  "commandAliases": [
    {
      "name": "dreaming",
      "kind": "runtime-slash",
      "cliCommand": "memory"
    }
  ]
}
```

| Champ        | Obligatoire | Type              | Ce que cela signifie                                                    |
| ------------ | ----------- | ----------------- | ----------------------------------------------------------------------- |
| `name`       | Oui         | `string`          | Nom de commande appartenant à ce Plugin.                                |
| `kind`       | Non         | `"runtime-slash"` | Marque l’alias comme une commande slash de chat plutôt qu’une commande CLI racine. |
| `cliCommand` | Non         | `string`          | Commande CLI racine associée à suggérer pour les opérations CLI, si elle existe. |

## Référence `activation`

Utilisez `activation` lorsque le Plugin peut déclarer à faible coût quels événements du plan de contrôle
doivent l’activer ultérieurement.

Ce bloc contient uniquement des métadonnées. Il n’enregistre pas le comportement runtime, et il
ne remplace pas `register(...)`, `setupEntry` ni les autres points d’entrée runtime/Plugin.
Les consommateurs actuels l’utilisent comme indication de restriction avant un chargement plus large des Plugins, donc
l’absence de métadonnées d’activation ne coûte généralement que des performances ; elle ne doit pas
modifier le comportement tant que les replis hérités de propriété du manifeste existent encore.

```json
{
  "activation": {
    "onProviders": ["openai"],
    "onCommands": ["models"],
    "onChannels": ["web"],
    "onRoutes": ["gateway-webhook"],
    "onCapabilities": ["provider", "tool"]
  }
}
```

| Champ            | Obligatoire | Type                                                 | Ce que cela signifie                                                |
| ---------------- | ----------- | ---------------------------------------------------- | ------------------------------------------------------------------- |
| `onProviders`    | Non         | `string[]`                                           | IDs de fournisseurs qui doivent activer ce Plugin lorsqu’ils sont demandés. |
| `onCommands`     | Non         | `string[]`                                           | IDs de commandes qui doivent activer ce Plugin.                     |
| `onChannels`     | Non         | `string[]`                                           | IDs de canaux qui doivent activer ce Plugin.                        |
| `onRoutes`       | Non         | `string[]`                                           | Types de routes qui doivent activer ce Plugin.                      |
| `onCapabilities` | Non         | `Array<"provider" \| "channel" \| "tool" \| "hook">` | Indications générales de capacité utilisées par la planification d’activation du plan de contrôle. |

Consommateurs actifs actuels :

- la planification CLI déclenchée par commande revient au comportement hérité
  `commandAliases[].cliCommand` ou `commandAliases[].name`
- la planification de configuration/canal déclenchée par canal revient à la propriété héritée `channels[]`
  lorsque les métadonnées explicites d’activation de canal sont absentes
- la planification de configuration/runtime déclenchée par fournisseur revient à la propriété héritée
  `providers[]` et au `cliBackends[]` de premier niveau lorsque les métadonnées explicites d’activation de fournisseur sont absentes

## Référence `setup`

Utilisez `setup` lorsque les surfaces de configuration et d’onboarding ont besoin de métadonnées peu coûteuses, détenues par le Plugin,
avant le chargement du runtime.

```json
{
  "setup": {
    "providers": [
      {
        "id": "openai",
        "authMethods": ["api-key"],
        "envVars": ["OPENAI_API_KEY"]
      }
    ],
    "cliBackends": ["openai-cli"],
    "configMigrations": ["legacy-openai-auth"],
    "requiresRuntime": false
  }
}
```

Le `cliBackends` de premier niveau reste valide et continue de décrire les
backends d’inférence CLI. `setup.cliBackends` est la surface de descripteur spécifique à la configuration pour
les flux du plan de contrôle/configuration qui doivent rester limités aux métadonnées.

Lorsqu’ils sont présents, `setup.providers` et `setup.cliBackends` sont la surface
de recherche privilégiée, d’abord fondée sur le descripteur, pour la découverte de configuration. Si le descripteur ne fait
que restreindre le Plugin candidat et que la configuration a encore besoin de hooks runtime plus riches au moment de la configuration,
définissez `requiresRuntime: true` et conservez `setup-api` comme
chemin d’exécution de repli.

Étant donné que la recherche de configuration peut exécuter du code `setup-api` détenu par le Plugin, les valeurs
normalisées de `setup.providers[].id` et `setup.cliBackends[]` doivent rester uniques parmi
les Plugins découverts. Une propriété ambiguë échoue de manière conservatrice au lieu de choisir un gagnant selon l’ordre de découverte.

### Référence `setup.providers`

| Champ         | Obligatoire | Type       | Ce que cela signifie                                                                      |
| ------------- | ----------- | ---------- | ----------------------------------------------------------------------------------------- |
| `id`          | Oui         | `string`   | ID du fournisseur exposé pendant la configuration ou l’onboarding. Gardez les IDs normalisés globalement uniques. |
| `authMethods` | Non         | `string[]` | IDs des méthodes de configuration/authentification que ce fournisseur prend en charge sans charger le runtime complet. |
| `envVars`     | Non         | `string[]` | Variables d’environnement que les surfaces génériques de configuration/statut peuvent vérifier avant le chargement du runtime du Plugin. |

### Champs `setup`

| Champ              | Obligatoire | Type       | Ce que cela signifie                                                                                      |
| ------------------ | ----------- | ---------- | --------------------------------------------------------------------------------------------------------- |
| `providers`        | Non         | `object[]` | Descripteurs de configuration de fournisseurs exposés pendant la configuration et l’onboarding.          |
| `cliBackends`      | Non         | `string[]` | IDs de backends au moment de la configuration utilisés pour la recherche de configuration fondée d’abord sur le descripteur. Gardez les IDs normalisés globalement uniques. |
| `configMigrations` | Non         | `string[]` | IDs de migrations de configuration détenus par la surface de configuration de ce Plugin.                  |
| `requiresRuntime`  | Non         | `boolean`  | Indique si la configuration nécessite encore l’exécution de `setup-api` après la recherche par descripteur. |

## Référence `uiHints`

`uiHints` est une map reliant les noms de champs de configuration à de petites indications d’affichage.

```json
{
  "uiHints": {
    "apiKey": {
      "label": "API key",
      "help": "Used for OpenRouter requests",
      "placeholder": "sk-or-v1-...",
      "sensitive": true
    }
  }
}
```

Chaque indication de champ peut inclure :

| Champ         | Type       | Ce que cela signifie                      |
| ------------- | ---------- | ----------------------------------------- |
| `label`       | `string`   | Libellé du champ destiné à l’utilisateur. |
| `help`        | `string`   | Court texte d’aide.                       |
| `tags`        | `string[]` | Tags d’interface facultatifs.             |
| `advanced`    | `boolean`  | Marque le champ comme avancé.             |
| `sensitive`   | `boolean`  | Marque le champ comme secret ou sensible. |
| `placeholder` | `string`   | Texte d’exemple pour les champs de formulaire. |

## Référence `contracts`

Utilisez `contracts` uniquement pour les métadonnées statiques de propriété de capacités qu’OpenClaw peut
lire sans importer le runtime du Plugin.

```json
{
  "contracts": {
    "speechProviders": ["openai"],
    "realtimeTranscriptionProviders": ["openai"],
    "realtimeVoiceProviders": ["openai"],
    "mediaUnderstandingProviders": ["openai", "openai-codex"],
    "imageGenerationProviders": ["openai"],
    "videoGenerationProviders": ["qwen"],
    "webFetchProviders": ["firecrawl"],
    "webSearchProviders": ["gemini"],
    "tools": ["firecrawl_search", "firecrawl_scrape"]
  }
}
```

Chaque liste est facultative :

| Champ                            | Type       | Ce que cela signifie                                         |
| -------------------------------- | ---------- | ------------------------------------------------------------ |
| `speechProviders`                | `string[]` | IDs de fournisseurs de parole détenus par ce Plugin.         |
| `realtimeTranscriptionProviders` | `string[]` | IDs de fournisseurs de transcription temps réel détenus par ce Plugin. |
| `realtimeVoiceProviders`         | `string[]` | IDs de fournisseurs de voix temps réel détenus par ce Plugin. |
| `mediaUnderstandingProviders`    | `string[]` | IDs de fournisseurs de compréhension des médias détenus par ce Plugin. |
| `imageGenerationProviders`       | `string[]` | IDs de fournisseurs de génération d’images détenus par ce Plugin. |
| `videoGenerationProviders`       | `string[]` | IDs de fournisseurs de génération vidéo détenus par ce Plugin. |
| `webFetchProviders`              | `string[]` | IDs de fournisseurs de récupération web détenus par ce Plugin. |
| `webSearchProviders`             | `string[]` | IDs de fournisseurs de recherche web détenus par ce Plugin.  |
| `tools`                          | `string[]` | Noms d’outils d’agent détenus par ce Plugin pour les vérifications de contrat intégrées. |

## Référence `channelConfigs`

Utilisez `channelConfigs` lorsqu’un Plugin de canal a besoin de métadonnées de configuration peu coûteuses avant
le chargement du runtime.

```json
{
  "channelConfigs": {
    "matrix": {
      "schema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "homeserverUrl": { "type": "string" }
        }
      },
      "uiHints": {
        "homeserverUrl": {
          "label": "Homeserver URL",
          "placeholder": "https://matrix.example.com"
        }
      },
      "label": "Matrix",
      "description": "Connexion au homeserver Matrix",
      "preferOver": ["matrix-legacy"]
    }
  }
}
```

Chaque entrée de canal peut inclure :

| Champ         | Type                     | Ce que cela signifie                                                                  |
| ------------- | ------------------------ | ------------------------------------------------------------------------------------- |
| `schema`      | `object`                 | Schéma JSON pour `channels.<id>`. Obligatoire pour chaque entrée déclarée de configuration de canal. |
| `uiHints`     | `Record<string, object>` | Libellés/placeholders/indications de sensibilité d’interface facultatifs pour cette section de configuration de canal. |
| `label`       | `string`                 | Libellé du canal fusionné dans les surfaces de sélection et d’inspection lorsque les métadonnées runtime ne sont pas prêtes. |
| `description` | `string`                 | Courte description du canal pour les surfaces d’inspection et de catalogue.           |
| `preferOver`  | `string[]`               | IDs de Plugins hérités ou de priorité inférieure que ce canal doit devancer dans les surfaces de sélection. |

## Référence `modelSupport`

Utilisez `modelSupport` lorsqu’OpenClaw doit déduire votre Plugin fournisseur à partir de
raccourcis d’ID de modèles comme `gpt-5.4` ou `claude-sonnet-4.6` avant le chargement du runtime du Plugin.

```json
{
  "modelSupport": {
    "modelPrefixes": ["gpt-", "o1", "o3", "o4"],
    "modelPatterns": ["^computer-use-preview"]
  }
}
```

OpenClaw applique l’ordre de priorité suivant :

- les références explicites `provider/model` utilisent les métadonnées de manifeste `providers` du propriétaire
- `modelPatterns` a priorité sur `modelPrefixes`
- si un Plugin non intégré et un Plugin intégré correspondent tous deux, le Plugin non intégré
  l’emporte
- toute ambiguïté restante est ignorée jusqu’à ce que l’utilisateur ou la configuration spécifie un fournisseur

Champs :

| Champ           | Type       | Ce que cela signifie                                                       |
| --------------- | ---------- | ------------------------------------------------------------------------- |
| `modelPrefixes` | `string[]` | Préfixes comparés avec `startsWith` aux IDs abrégés de modèles.           |
| `modelPatterns` | `string[]` | Sources regex comparées aux IDs abrégés de modèles après suppression du suffixe de profil. |

Les clés de capacité héritées de premier niveau sont obsolètes. Utilisez `openclaw doctor --fix` pour
déplacer `speechProviders`, `realtimeTranscriptionProviders`,
`realtimeVoiceProviders`, `mediaUnderstandingProviders`,
`imageGenerationProviders`, `videoGenerationProviders`,
`webFetchProviders` et `webSearchProviders` sous `contracts` ; le
chargement normal du manifeste ne traite plus ces champs de premier niveau comme
une propriété de capacité.

## Manifeste versus `package.json`

Les deux fichiers remplissent des rôles différents :

| Fichier                | À utiliser pour                                                                                                                   |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.plugin.json` | Découverte, validation de configuration, métadonnées de choix d’authentification et indications d’interface qui doivent exister avant l’exécution du code du Plugin |
| `package.json`         | Métadonnées npm, installation des dépendances et bloc `openclaw` utilisé pour les points d’entrée, le filtrage d’installation, la configuration ou les métadonnées de catalogue |

Si vous ne savez pas où placer une métadonnée, utilisez cette règle :

- si OpenClaw doit la connaître avant de charger le code du Plugin, placez-la dans `openclaw.plugin.json`
- si elle concerne le packaging, les fichiers d’entrée ou le comportement d’installation npm, placez-la dans `package.json`

### Champs `package.json` qui affectent la découverte

Certaines métadonnées de Plugin avant runtime vivent volontairement dans `package.json` sous le
bloc `openclaw` plutôt que dans `openclaw.plugin.json`.

Exemples importants :

| Champ                                                             | Ce que cela signifie                                                                                                                        |
| ----------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.extensions`                                             | Déclare les points d’entrée de Plugins natifs.                                                                                              |
| `openclaw.setupEntry`                                             | Point d’entrée léger réservé à la configuration, utilisé pendant l’onboarding et le démarrage différé des canaux.                         |
| `openclaw.channel`                                                | Métadonnées légères de catalogue de canal comme les libellés, chemins de documentation, alias et texte de sélection.                      |
| `openclaw.channel.configuredState`                                | Métadonnées légères du vérificateur d’état configuré qui peuvent répondre à « une configuration uniquement via env existe-t-elle déjà ? » sans charger le runtime complet du canal. |
| `openclaw.channel.persistedAuthState`                             | Métadonnées légères du vérificateur d’authentification persistée qui peuvent répondre à « y a-t-il déjà quelque chose de connecté ? » sans charger le runtime complet du canal. |
| `openclaw.install.npmSpec` / `openclaw.install.localPath`         | Indications d’installation/mise à jour pour les Plugins intégrés et publiés externement.                                                   |
| `openclaw.install.defaultChoice`                                  | Chemin d’installation préféré lorsque plusieurs sources d’installation sont disponibles.                                                   |
| `openclaw.install.minHostVersion`                                 | Version minimale prise en charge de l’hôte OpenClaw, avec un plancher semver comme `>=2026.3.22`.                                        |
| `openclaw.install.allowInvalidConfigRecovery`                     | Autorise un chemin de récupération de réinstallation étroit pour un Plugin intégré lorsque la configuration est invalide.                 |
| `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen` | Permet aux surfaces de canal réservées à la configuration de se charger avant le Plugin de canal complet au démarrage.                    |

`openclaw.install.minHostVersion` est appliqué pendant l’installation et le
chargement du registre de manifestes. Les valeurs invalides sont rejetées ; les valeurs valides mais plus récentes ignorent le
Plugin sur les hôtes plus anciens.

`openclaw.install.allowInvalidConfigRecovery` est volontairement limité. Il
ne rend pas installables des configurations arbitrairement cassées. Aujourd’hui, il permet seulement aux flux d’installation
de récupérer à partir de certaines pannes obsolètes de mise à niveau d’un Plugin intégré, comme un
chemin de Plugin intégré manquant ou une entrée `channels.<id>` obsolète pour ce même
Plugin intégré. Les erreurs de configuration non liées bloquent toujours l’installation et redirigent les opérateurs
vers `openclaw doctor --fix`.

`openclaw.channel.persistedAuthState` est une métadonnée de package pour un minuscule
module de vérification :

```json
{
  "openclaw": {
    "channel": {
      "id": "whatsapp",
      "persistedAuthState": {
        "specifier": "./auth-presence",
        "exportName": "hasAnyWhatsAppAuth"
      }
    }
  }
}
```

Utilisez-le lorsque les flux de configuration, de doctor ou d’état configuré ont besoin d’une
vérification d’authentification oui/non peu coûteuse avant le chargement du Plugin de canal complet. L’export cible doit être une petite
fonction qui lit uniquement l’état persisté ; ne le faites pas transiter par le barrel runtime complet du
canal.

`openclaw.channel.configuredState` suit la même structure pour des vérifications peu coûteuses
de l’état configuré uniquement via env :

```json
{
  "openclaw": {
    "channel": {
      "id": "telegram",
      "configuredState": {
        "specifier": "./configured-state",
        "exportName": "hasTelegramConfiguredState"
      }
    }
  }
}
```

Utilisez-le lorsqu’un canal peut répondre à l’état configuré à partir de l’env ou d’autres petites
entrées hors runtime. Si la vérification nécessite la résolution complète de la configuration ou le véritable
runtime du canal, conservez plutôt cette logique dans le hook `config.hasConfiguredState` du Plugin.

## Exigences du schéma JSON

- **Chaque Plugin doit fournir un schéma JSON**, même s’il n’accepte aucune configuration.
- Un schéma vide est acceptable (par exemple, `{ "type": "object", "additionalProperties": false }`).
- Les schémas sont validés au moment de la lecture/écriture de la configuration, pas à l’exécution.

## Comportement de validation

- Les clés inconnues dans `channels.*` sont des **erreurs**, sauf si l’ID de canal est déclaré par
  un manifeste de Plugin.
- `plugins.entries.<id>`, `plugins.allow`, `plugins.deny` et `plugins.slots.*`
  doivent référencer des IDs de Plugin **détectables**. Les IDs inconnus sont des **erreurs**.
- Si un Plugin est installé mais possède un manifeste ou un schéma cassé ou manquant,
  la validation échoue et Doctor signale l’erreur du Plugin.
- Si une configuration de Plugin existe mais que le Plugin est **désactivé**, la configuration est conservée et
  un **avertissement** apparaît dans Doctor et dans les journaux.

Voir [Configuration reference](/fr/gateway/configuration) pour le schéma complet `plugins.*`.

## Remarques

- Le manifeste est **obligatoire pour les Plugins natifs OpenClaw**, y compris pour les chargements depuis le système de fichiers local.
- Le runtime charge toujours le module du Plugin séparément ; le manifeste sert uniquement à la
  découverte + validation.
- Les manifestes natifs sont analysés avec JSON5, donc les commentaires, les virgules finales et les
  clés non entre guillemets sont acceptés tant que la valeur finale reste un objet.
- Seuls les champs de manifeste documentés sont lus par le chargeur de manifeste. Évitez d’ajouter
  ici des clés personnalisées de premier niveau.
- `providerAuthEnvVars` est le chemin de métadonnées peu coûteux pour les sondes d’authentification, la
  validation des marqueurs d’environnement et les surfaces similaires d’authentification fournisseur qui ne doivent pas démarrer le runtime du Plugin
  uniquement pour inspecter les noms de variables d’environnement.
- `providerAuthAliases` permet à des variantes de fournisseurs de réutiliser les variables d’environnement d’authentification
  d’un autre fournisseur, ses profils d’authentification, son authentification adossée à la configuration et son choix
  d’onboarding par clé d’API sans coder cette relation en dur dans le cœur.
- `channelEnvVars` est le chemin de métadonnées peu coûteux pour le repli via l’environnement shell, les invites de configuration
  et les surfaces de canal similaires qui ne doivent pas démarrer le runtime du Plugin
  uniquement pour inspecter les noms de variables d’environnement.
- `providerAuthChoices` est le chemin de métadonnées peu coûteux pour les sélecteurs de choix d’authentification,
  la résolution de `--auth-choice`, le mapping du fournisseur préféré et l’enregistrement simple de flags CLI
  d’onboarding avant le chargement du runtime du fournisseur. Pour les métadonnées d’assistant runtime
  qui nécessitent le code du fournisseur, voir
  [Provider runtime hooks](/fr/plugins/architecture#provider-runtime-hooks).
- Les types de Plugin exclusifs sont sélectionnés via `plugins.slots.*`.
  - `kind: "memory"` est sélectionné par `plugins.slots.memory`.
  - `kind: "context-engine"` est sélectionné par `plugins.slots.contextEngine`
    (par défaut : `legacy` intégré).
- `channels`, `providers`, `cliBackends` et `skills` peuvent être omis lorsqu’un
  Plugin n’en a pas besoin.
- Si votre Plugin dépend de modules natifs, documentez les étapes de build et toute
  exigence de liste d’autorisation du gestionnaire de paquets (par exemple, pnpm `allow-build-scripts`
  - `pnpm rebuild <package>`).

## Connexe

- [Building Plugins](/fr/plugins/building-plugins) — prise en main des Plugins
- [Plugin Architecture](/fr/plugins/architecture) — architecture interne
- [SDK Overview](/fr/plugins/sdk-overview) — référence du SDK de Plugin
