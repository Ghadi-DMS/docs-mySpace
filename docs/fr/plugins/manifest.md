---
read_when:
    - Vous créez un plugin OpenClaw
    - Vous devez livrer un schéma de configuration de plugin ou déboguer des erreurs de validation de plugin
summary: Exigences du manifeste de plugin + schéma JSON (validation stricte de la configuration)
title: Manifeste de plugin
x-i18n:
    generated_at: "2026-04-07T06:52:06Z"
    model: gpt-5.4
    provider: openai
    source_hash: 22d41b9f8748b1b1b066ee856be4a8f41e88b9a8bc073d74fc79d2bb0982f01a
    source_path: plugins/manifest.md
    workflow: 15
---

# Manifeste de plugin (`openclaw.plugin.json`)

Cette page concerne uniquement le **manifeste de plugin OpenClaw natif**.

Pour les dispositions de bundle compatibles, voir [Bundles de plugins](/fr/plugins/bundles).

Les formats de bundle compatibles utilisent des fichiers manifeste différents :

- Bundle Codex : `.codex-plugin/plugin.json`
- Bundle Claude : `.claude-plugin/plugin.json` ou la disposition de composant Claude
  par défaut sans manifeste
- Bundle Cursor : `.cursor-plugin/plugin.json`

OpenClaw détecte aussi automatiquement ces dispositions de bundle, mais elles ne sont pas validées
par rapport au schéma `openclaw.plugin.json` décrit ici.

Pour les bundles compatibles, OpenClaw lit actuellement les métadonnées du bundle ainsi que les
racines de Skills déclarées, les racines de commandes Claude, les valeurs par défaut `settings.json` des bundles Claude,
les valeurs par défaut LSP des bundles Claude et les packs de hooks pris en charge lorsque la disposition correspond
aux attentes d'exécution d'OpenClaw.

Chaque plugin OpenClaw natif **doit** livrer un fichier `openclaw.plugin.json` dans la
**racine du plugin**. OpenClaw utilise ce manifeste pour valider la configuration
**sans exécuter le code du plugin**. Les manifestes manquants ou invalides sont traités comme des
erreurs de plugin et bloquent la validation de la configuration.

Voir le guide complet du système de plugins : [Plugins](/fr/tools/plugin).
Pour le modèle de capacités natif et les recommandations actuelles de compatibilité externe :
[Modèle de capacités](/fr/plugins/architecture#public-capability-model).

## Ce que fait ce fichier

`openclaw.plugin.json` est la métadonnée qu'OpenClaw lit avant de charger le
code de votre plugin.

Utilisez-le pour :

- l'identité du plugin
- la validation de la configuration
- les métadonnées d'authentification et d'intégration initiale qui doivent être disponibles sans démarrer l'exécution du plugin
- les métadonnées d'alias et d'activation automatique qui doivent être résolues avant le chargement de l'exécution du plugin
- les métadonnées abrégées de possession de familles de modèles qui doivent activer automatiquement le
  plugin avant le chargement de l'exécution
- les instantanés statiques de possession de capacités utilisés pour le câblage de compatibilité intégré et
  la couverture des contrats
- les métadonnées de configuration spécifiques aux canaux qui doivent être fusionnées dans les surfaces de catalogue et de validation
  sans charger l'exécution
- les indications pour l'UI de configuration

Ne l'utilisez pas pour :

- enregistrer le comportement à l'exécution
- déclarer les points d'entrée du code
- les métadonnées d'installation npm

Ces éléments appartiennent au code de votre plugin et à `package.json`.

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
  "description": "Plugin de fournisseur OpenRouter",
  "version": "1.0.0",
  "providers": ["openrouter"],
  "modelSupport": {
    "modelPrefixes": ["router-"]
  },
  "cliBackends": ["openrouter-cli"],
  "providerAuthEnvVars": {
    "openrouter": ["OPENROUTER_API_KEY"]
  },
  "channelEnvVars": {
    "openrouter-chatops": ["OPENROUTER_CHATOPS_TOKEN"]
  },
  "providerAuthChoices": [
    {
      "provider": "openrouter",
      "method": "api-key",
      "choiceId": "openrouter-api-key",
      "choiceLabel": "Clé API OpenRouter",
      "groupId": "openrouter",
      "groupLabel": "OpenRouter",
      "optionKey": "openrouterApiKey",
      "cliFlag": "--openrouter-api-key",
      "cliOption": "--openrouter-api-key <key>",
      "cliDescription": "Clé API OpenRouter",
      "onboardingScopes": ["text-inference"]
    }
  ],
  "uiHints": {
    "apiKey": {
      "label": "Clé API",
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

| Champ                               | Obligatoire | Type                             | Signification                                                                                                                                                                                                    |
| ----------------------------------- | ----------- | -------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`                                | Oui         | `string`                         | ID canonique du plugin. C'est l'ID utilisé dans `plugins.entries.<id>`.                                                                                                                                         |
| `configSchema`                      | Oui         | `object`                         | Schéma JSON inline pour la configuration de ce plugin.                                                                                                                                                          |
| `enabledByDefault`                  | Non         | `true`                           | Marque un plugin intégré comme activé par défaut. Omettez-le, ou définissez toute valeur autre que `true`, pour laisser le plugin désactivé par défaut.                                                       |
| `legacyPluginIds`                   | Non         | `string[]`                       | IDs hérités qui sont normalisés vers cet ID canonique de plugin.                                                                                                                                                |
| `autoEnableWhenConfiguredProviders` | Non         | `string[]`                       | IDs de fournisseurs qui doivent activer automatiquement ce plugin lorsque l'authentification, la configuration ou les références de modèle les mentionnent.                                                    |
| `kind`                              | Non         | `"memory"` \| `"context-engine"` | Déclare un type exclusif de plugin utilisé par `plugins.slots.*`.                                                                                                                                               |
| `channels`                          | Non         | `string[]`                       | IDs de canaux détenus par ce plugin. Utilisés pour la découverte et la validation de la configuration.                                                                                                          |
| `providers`                         | Non         | `string[]`                       | IDs de fournisseurs détenus par ce plugin.                                                                                                                                                                      |
| `modelSupport`                      | Non         | `object`                         | Métadonnées abrégées de familles de modèles détenues par le manifeste, utilisées pour charger automatiquement le plugin avant l'exécution.                                                                     |
| `cliBackends`                       | Non         | `string[]`                       | IDs de backends d'inférence CLI détenus par ce plugin. Utilisés pour l'activation automatique au démarrage à partir de références de configuration explicites.                                                 |
| `providerAuthEnvVars`               | Non         | `Record<string, string[]>`       | Métadonnées d'environnement d'authentification fournisseur légères qu'OpenClaw peut inspecter sans charger le code du plugin.                                                                                 |
| `channelEnvVars`                    | Non         | `Record<string, string[]>`       | Métadonnées d'environnement de canal légères qu'OpenClaw peut inspecter sans charger le code du plugin. Utilisez-les pour une configuration de canal pilotée par l'environnement ou pour des surfaces d'authentification que les helpers génériques de démarrage/configuration doivent voir. |
| `providerAuthChoices`               | Non         | `object[]`                       | Métadonnées légères de choix d'authentification pour les sélecteurs d'intégration initiale, la résolution du fournisseur préféré et le câblage simple des indicateurs CLI.                                    |
| `contracts`                         | Non         | `object`                         | Instantané statique des capacités intégrées pour la parole, la transcription temps réel, la voix temps réel, la compréhension des médias, la génération d'images, la génération musicale, la génération vidéo, la récupération web, la recherche web et la possession d'outils. |
| `channelConfigs`                    | Non         | `Record<string, object>`         | Métadonnées de configuration de canal détenues par le manifeste, fusionnées dans les surfaces de découverte et de validation avant le chargement de l'exécution.                                              |
| `skills`                            | Non         | `string[]`                       | Répertoires Skills à charger, relatifs à la racine du plugin.                                                                                                                                                   |
| `name`                              | Non         | `string`                         | Nom lisible du plugin.                                                                                                                                                                                          |
| `description`                       | Non         | `string`                         | Résumé court affiché dans les surfaces du plugin.                                                                                                                                                               |
| `version`                           | Non         | `string`                         | Version informative du plugin.                                                                                                                                                                                  |
| `uiHints`                           | Non         | `Record<string, object>`         | Libellés UI, placeholders et indications de sensibilité pour les champs de configuration.                                                                                                                       |

## Référence `providerAuthChoices`

Chaque entrée `providerAuthChoices` décrit un choix d'intégration initiale ou d'authentification.
OpenClaw la lit avant le chargement de l'exécution du fournisseur.

| Champ                 | Obligatoire | Type                                            | Signification                                                                                   |
| --------------------- | ----------- | ----------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| `provider`            | Oui         | `string`                                        | ID du fournisseur auquel appartient ce choix.                                                   |
| `method`              | Oui         | `string`                                        | ID de méthode d'authentification vers laquelle dispatcher.                                      |
| `choiceId`            | Oui         | `string`                                        | ID stable de choix d'authentification utilisé par les flux d'intégration initiale et CLI.      |
| `choiceLabel`         | Non         | `string`                                        | Libellé destiné à l'utilisateur. S'il est omis, OpenClaw revient à `choiceId`.                 |
| `choiceHint`          | Non         | `string`                                        | Texte d'aide court pour le sélecteur.                                                           |
| `assistantPriority`   | Non         | `number`                                        | Les valeurs plus faibles sont triées plus tôt dans les sélecteurs interactifs pilotés par assistant. |
| `assistantVisibility` | Non         | `"visible"` \| `"manual-only"`                  | Cache le choix dans les sélecteurs de l'assistant tout en autorisant une sélection CLI manuelle. |
| `deprecatedChoiceIds` | Non         | `string[]`                                      | IDs de choix hérités qui doivent rediriger les utilisateurs vers ce choix de remplacement.      |
| `groupId`             | Non         | `string`                                        | ID de groupe facultatif pour regrouper des choix liés.                                          |
| `groupLabel`          | Non         | `string`                                        | Libellé destiné à l'utilisateur pour ce groupe.                                                 |
| `groupHint`           | Non         | `string`                                        | Texte d'aide court pour le groupe.                                                              |
| `optionKey`           | Non         | `string`                                        | Clé d'option interne pour les flux d'authentification simples à un seul indicateur.             |
| `cliFlag`             | Non         | `string`                                        | Nom d'indicateur CLI, tel que `--openrouter-api-key`.                                           |
| `cliOption`           | Non         | `string`                                        | Forme complète de l'option CLI, telle que `--openrouter-api-key <key>`.                         |
| `cliDescription`      | Non         | `string`                                        | Description utilisée dans l'aide CLI.                                                           |
| `onboardingScopes`    | Non         | `Array<"text-inference" \| "image-generation">` | Sur quelles surfaces d'intégration initiale ce choix doit apparaître. S'il est omis, la valeur par défaut est `["text-inference"]`. |

## Référence `uiHints`

`uiHints` est une map des noms de champs de configuration vers de petites indications de rendu.

```json
{
  "uiHints": {
    "apiKey": {
      "label": "Clé API",
      "help": "Utilisée pour les requêtes OpenRouter",
      "placeholder": "sk-or-v1-...",
      "sensitive": true
    }
  }
}
```

Chaque indication de champ peut inclure :

| Champ         | Type       | Signification                                |
| ------------- | ---------- | -------------------------------------------- |
| `label`       | `string`   | Libellé de champ destiné à l'utilisateur.    |
| `help`        | `string`   | Texte d'aide court.                          |
| `tags`        | `string[]` | Tags UI facultatifs.                         |
| `advanced`    | `boolean`  | Marque le champ comme avancé.                |
| `sensitive`   | `boolean`  | Marque le champ comme secret ou sensible.    |
| `placeholder` | `string`   | Texte de placeholder pour les entrées de formulaire. |

## Référence `contracts`

Utilisez `contracts` uniquement pour les métadonnées statiques de possession de capacités qu'OpenClaw peut
lire sans importer l'exécution du plugin.

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

| Champ                            | Type       | Signification                                                       |
| -------------------------------- | ---------- | ------------------------------------------------------------------- |
| `speechProviders`                | `string[]` | IDs de fournisseurs de parole détenus par ce plugin.                |
| `realtimeTranscriptionProviders` | `string[]` | IDs de fournisseurs de transcription temps réel détenus par ce plugin. |
| `realtimeVoiceProviders`         | `string[]` | IDs de fournisseurs de voix temps réel détenus par ce plugin.       |
| `mediaUnderstandingProviders`    | `string[]` | IDs de fournisseurs de compréhension des médias détenus par ce plugin. |
| `imageGenerationProviders`       | `string[]` | IDs de fournisseurs de génération d'images détenus par ce plugin.   |
| `videoGenerationProviders`       | `string[]` | IDs de fournisseurs de génération vidéo détenus par ce plugin.      |
| `webFetchProviders`              | `string[]` | IDs de fournisseurs de récupération web détenus par ce plugin.      |
| `webSearchProviders`             | `string[]` | IDs de fournisseurs de recherche web détenus par ce plugin.         |
| `tools`                          | `string[]` | Noms d'outils d'agent détenus par ce plugin pour les vérifications de contrat intégrées. |

## Référence `channelConfigs`

Utilisez `channelConfigs` lorsqu'un plugin de canal a besoin de métadonnées de configuration légères avant
le chargement de l'exécution.

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
          "label": "URL du homeserver",
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

| Champ         | Type                     | Signification                                                                                  |
| ------------- | ------------------------ | ---------------------------------------------------------------------------------------------- |
| `schema`      | `object`                 | Schéma JSON pour `channels.<id>`. Obligatoire pour chaque entrée déclarée de configuration de canal. |
| `uiHints`     | `Record<string, object>` | Libellés UI/placeholders/indications de sensibilité facultatifs pour cette section de configuration de canal. |
| `label`       | `string`                 | Libellé de canal fusionné dans les surfaces de sélection et d'inspection lorsque les métadonnées d'exécution ne sont pas prêtes. |
| `description` | `string`                 | Description courte du canal pour les surfaces d'inspection et de catalogue.                    |
| `preferOver`  | `string[]`               | IDs de plugins hérités ou de priorité inférieure que ce canal doit supplanter dans les surfaces de sélection. |

## Référence `modelSupport`

Utilisez `modelSupport` lorsqu'OpenClaw doit déduire votre plugin fournisseur à partir
d'IDs de modèles abrégés comme `gpt-5.4` ou `claude-sonnet-4.6` avant le chargement de l'exécution du plugin.

```json
{
  "modelSupport": {
    "modelPrefixes": ["gpt-", "o1", "o3", "o4"],
    "modelPatterns": ["^computer-use-preview"]
  }
}
```

OpenClaw applique cette priorité :

- les références explicites `provider/model` utilisent les métadonnées `providers` du manifeste propriétaire
- `modelPatterns` l'emporte sur `modelPrefixes`
- si un plugin non intégré et un plugin intégré correspondent tous deux, le plugin non intégré
  l'emporte
- les ambiguïtés restantes sont ignorées jusqu'à ce que l'utilisateur ou la configuration spécifie un fournisseur

Champs :

| Champ           | Type       | Signification                                                                 |
| --------------- | ---------- | ----------------------------------------------------------------------------- |
| `modelPrefixes` | `string[]` | Préfixes comparés avec `startsWith` aux IDs de modèles abrégés.               |
| `modelPatterns` | `string[]` | Sources regex comparées aux IDs de modèles abrégés après suppression du suffixe de profil. |

Les clés de capacité héritées de premier niveau sont obsolètes. Utilisez `openclaw doctor --fix` pour
déplacer `speechProviders`, `realtimeTranscriptionProviders`,
`realtimeVoiceProviders`, `mediaUnderstandingProviders`,
`imageGenerationProviders`, `videoGenerationProviders`,
`webFetchProviders` et `webSearchProviders` sous `contracts` ; le
chargement normal des manifestes ne traite plus ces champs de premier niveau comme possession
de capacités.

## Manifeste versus `package.json`

Les deux fichiers remplissent des rôles différents :

| Fichier                | À utiliser pour                                                                                                                    |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.plugin.json` | Découverte, validation de la configuration, métadonnées de choix d'authentification et indications UI qui doivent exister avant l'exécution du code du plugin |
| `package.json`         | Métadonnées npm, installation des dépendances et bloc `openclaw` utilisé pour les points d'entrée, le contrôle d'installation, la configuration ou les métadonnées de catalogue |

Si vous ne savez pas où placer une métadonnée, appliquez cette règle :

- si OpenClaw doit la connaître avant de charger le code du plugin, placez-la dans `openclaw.plugin.json`
- si elle concerne le packaging, les fichiers d'entrée ou le comportement d'installation npm, placez-la dans `package.json`

### Champs `package.json` qui affectent la découverte

Certaines métadonnées de plugin avant exécution vivent intentionnellement dans `package.json` sous le
bloc `openclaw` au lieu de `openclaw.plugin.json`.

Exemples importants :

| Champ                                                             | Signification                                                                                                                            |
| ----------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.extensions`                                             | Déclare les points d'entrée de plugins natifs.                                                                                           |
| `openclaw.setupEntry`                                             | Point d'entrée léger réservé à la configuration utilisé pendant l'intégration initiale et le démarrage différé des canaux.              |
| `openclaw.channel`                                                | Métadonnées légères de catalogue de canaux comme les libellés, chemins de documentation, alias et texte de sélection.                  |
| `openclaw.channel.configuredState`                                | Métadonnées légères du vérificateur d'état configuré qui peuvent répondre à « une configuration uniquement par environnement existe-t-elle déjà ? » sans charger toute l'exécution du canal. |
| `openclaw.channel.persistedAuthState`                             | Métadonnées légères du vérificateur d'authentification persistée qui peuvent répondre à « y a-t-il déjà quelque chose de connecté ? » sans charger toute l'exécution du canal. |
| `openclaw.install.npmSpec` / `openclaw.install.localPath`         | Indications d'installation/mise à jour pour les plugins intégrés et publiés en externe.                                                 |
| `openclaw.install.defaultChoice`                                  | Chemin d'installation préféré lorsque plusieurs sources d'installation sont disponibles.                                                 |
| `openclaw.install.minHostVersion`                                 | Version minimale prise en charge de l'hôte OpenClaw, avec un plancher semver comme `>=2026.3.22`.                                      |
| `openclaw.install.allowInvalidConfigRecovery`                     | Autorise un chemin de récupération étroit de réinstallation de plugin intégré lorsque la configuration est invalide.                   |
| `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen` | Permet aux surfaces de canal réservées à la configuration de se charger avant le plugin de canal complet au démarrage.                |

`openclaw.install.minHostVersion` est appliqué pendant l'installation et le
chargement du registre de manifestes. Les valeurs invalides sont rejetées ; les valeurs valides mais plus récentes ignorent le
plugin sur les hôtes plus anciens.

`openclaw.install.allowInvalidConfigRecovery` est volontairement étroit. Il
ne rend pas installables des configurations arbitrairement cassées. Aujourd'hui, il autorise seulement les
flux d'installation à récupérer de certains échecs d'obsolescence spécifiques à la mise à niveau de plugins intégrés, tels qu'un
chemin de plugin intégré manquant ou une entrée `channels.<id>` obsolète pour ce même plugin intégré. Les erreurs de configuration non liées continuent de bloquer l'installation et dirigent les opérateurs
vers `openclaw doctor --fix`.

`openclaw.channel.persistedAuthState` est une métadonnée de package pour un minuscule
module vérificateur :

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

Utilisez-la lorsque les flux de configuration, de doctor ou d'état configuré ont besoin d'une sonde d'authentification
simple oui/non avant le chargement du plugin de canal complet. L'export cible doit être une petite
fonction qui lit uniquement l'état persisté ; ne le faites pas passer par le barrel complet
d'exécution du canal.

`openclaw.channel.configuredState` suit la même forme pour des vérifications légères d'état
configuré uniquement via environnement :

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

Utilisez-la lorsqu'un canal peut répondre à l'état configuré à partir de l'environnement ou d'autres petites
entrées sans exécution. Si la vérification nécessite la résolution complète de la configuration ou la véritable
exécution du canal, conservez cette logique dans le hook du plugin `config.hasConfiguredState`.

## Exigences du schéma JSON

- **Chaque plugin doit livrer un schéma JSON**, même s'il n'accepte aucune configuration.
- Un schéma vide est acceptable (par exemple, `{ "type": "object", "additionalProperties": false }`).
- Les schémas sont validés au moment de la lecture/écriture de la configuration, pas à l'exécution.

## Comportement de validation

- Les clés inconnues `channels.*` sont des **erreurs**, sauf si l'ID de canal est déclaré par
  un manifeste de plugin.
- `plugins.entries.<id>`, `plugins.allow`, `plugins.deny` et `plugins.slots.*`
  doivent référencer des IDs de plugin **détectables**. Les IDs inconnus sont des **erreurs**.
- Si un plugin est installé mais possède un manifeste ou un schéma cassé ou manquant,
  la validation échoue et Doctor signale l'erreur du plugin.
- Si la configuration du plugin existe mais que le plugin est **désactivé**, la configuration est conservée et
  un **avertissement** est affiché dans Doctor + les journaux.

Voir [Référence de configuration](/fr/gateway/configuration) pour le schéma complet `plugins.*`.

## Notes

- Le manifeste est **obligatoire pour les plugins OpenClaw natifs**, y compris les chargements depuis le système de fichiers local.
- L'exécution charge toujours séparément le module du plugin ; le manifeste sert uniquement à
  la découverte + validation.
- Les manifestes natifs sont analysés avec JSON5, donc les commentaires, virgules finales et
  clés non entre guillemets sont acceptés tant que la valeur finale reste un objet.
- Seuls les champs de manifeste documentés sont lus par le chargeur de manifestes. Évitez d'ajouter
  ici des clés de premier niveau personnalisées.
- `providerAuthEnvVars` est le chemin de métadonnées léger pour les sondes d'authentification, la
  validation des marqueurs d'environnement et les surfaces similaires d'authentification fournisseur qui ne doivent pas démarrer l'exécution du plugin
  juste pour inspecter les noms d'environnement.
- `channelEnvVars` est le chemin de métadonnées léger pour le repli d'environnement shell, les invites de configuration
  et les surfaces de canal similaires qui ne doivent pas démarrer l'exécution du plugin
  juste pour inspecter les noms d'environnement.
- `providerAuthChoices` est le chemin de métadonnées léger pour les sélecteurs de choix d'authentification,
  la résolution `--auth-choice`, la correspondance du fournisseur préféré et l'enregistrement simple
  des indicateurs CLI d'intégration initiale avant le chargement de l'exécution du fournisseur. Pour les métadonnées d'assistant à l'exécution
  qui nécessitent le code fournisseur, voir
  [Hooks d'exécution du fournisseur](/fr/plugins/architecture#provider-runtime-hooks).
- Les types exclusifs de plugins sont sélectionnés via `plugins.slots.*`.
  - `kind: "memory"` est sélectionné par `plugins.slots.memory`.
  - `kind: "context-engine"` est sélectionné par `plugins.slots.contextEngine`
    (par défaut : `legacy` intégré).
- `channels`, `providers`, `cliBackends` et `skills` peuvent être omis lorsqu'un
  plugin n'en a pas besoin.
- Si votre plugin dépend de modules natifs, documentez les étapes de build et toute
  exigence d'allowlist du gestionnaire de paquets (par exemple, pnpm `allow-build-scripts`
  - `pnpm rebuild <package>`).

## Voir aussi

- [Créer des plugins](/fr/plugins/building-plugins) — prise en main des plugins
- [Architecture des plugins](/fr/plugins/architecture) — architecture interne
- [Vue d'ensemble du SDK](/fr/plugins/sdk-overview) — référence du SDK des plugins
