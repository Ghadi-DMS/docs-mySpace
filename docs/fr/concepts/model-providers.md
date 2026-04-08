---
read_when:
    - Vous avez besoin d'une référence de configuration des modèles, fournisseur par fournisseur
    - Vous voulez des exemples de config ou des commandes d'intégration CLI pour les fournisseurs de modèles
summary: Vue d'ensemble des fournisseurs de modèles avec des exemples de config + des flux CLI
title: Fournisseurs de modèles
x-i18n:
    generated_at: "2026-04-08T06:02:59Z"
    model: gpt-5.4
    provider: openai
    source_hash: 558ac9e34b67fcc3dd6791a01bebc17e1c34152fa6c5611593d681e8cfa532d9
    source_path: concepts/model-providers.md
    workflow: 15
---

# Fournisseurs de modèles

Cette page couvre les **fournisseurs de LLM/modèles** (et non les canaux de chat comme WhatsApp/Telegram).
Pour les règles de sélection des modèles, consultez [/concepts/models](/fr/concepts/models).

## Règles rapides

- Les références de modèles utilisent `provider/model` (exemple : `opencode/claude-opus-4-6`).
- Si vous définissez `agents.defaults.models`, cela devient la liste d'autorisation.
- Assistants CLI : `openclaw onboard`, `openclaw models list`, `openclaw models set <provider/model>`.
- Les règles d'exécution de secours, les sondes de refroidissement et la persistance des remplacements de session sont
  documentées dans [/concepts/model-failover](/fr/concepts/model-failover).
- `models.providers.*.models[].contextWindow` est la métadonnée native du modèle ;
  `models.providers.*.models[].contextTokens` est la limite effective à l'exécution.
- Les plugins de fournisseur peuvent injecter des catalogues de modèles via `registerProvider({ catalog })` ;
  OpenClaw fusionne cette sortie dans `models.providers` avant d'écrire
  `models.json`.
- Les manifestes de fournisseur peuvent déclarer `providerAuthEnvVars` afin que les sondes d'authentification génériques basées sur l'environnement
  n'aient pas besoin de charger l'exécution du plugin. La carte restante des variables d'environnement du noyau
  est désormais réservée aux fournisseurs non plugin/du noyau et à quelques cas de précédence générique
  comme l'intégration Anthropic avec priorité à la clé API.
- Les plugins de fournisseur peuvent aussi posséder le comportement d'exécution du fournisseur via
  `normalizeModelId`, `normalizeTransport`, `normalizeConfig`,
  `applyNativeStreamingUsageCompat`, `resolveConfigApiKey`,
  `resolveSyntheticAuth`, `shouldDeferSyntheticProfileAuth`,
  `resolveDynamicModel`, `prepareDynamicModel`,
  `normalizeResolvedModel`, `contributeResolvedModelCompat`,
  `capabilities`, `normalizeToolSchemas`,
  `inspectToolSchemas`, `resolveReasoningOutputMode`,
  `prepareExtraParams`, `createStreamFn`, `wrapStreamFn`,
  `resolveTransportTurnState`, `resolveWebSocketSessionPolicy`,
  `createEmbeddingProvider`, `formatApiKey`, `refreshOAuth`,
  `buildAuthDoctorHint`,
  `matchesContextOverflowError`, `classifyFailoverReason`,
  `isCacheTtlEligible`, `buildMissingAuthMessage`, `suppressBuiltInModel`,
  `augmentModelCatalog`, `isBinaryThinking`, `supportsXHighThinking`,
  `resolveDefaultThinkingLevel`, `applyConfigDefaults`, `isModernModelRef`,
  `prepareRuntimeAuth`, `resolveUsageAuth`, `fetchUsageSnapshot`, et
  `onModelSelected`.
- Remarque : les `capabilities` de l'exécution du fournisseur sont des métadonnées partagées du runner (famille de fournisseurs, particularités des transcriptions/outils, indices de transport/cache). Ce n'est pas le
  même modèle que le [modèle public de capacités](/fr/plugins/architecture#public-capability-model)
  qui décrit ce qu'un plugin enregistre (inférence de texte, parole, etc.).

## Comportement du fournisseur possédé par le plugin

Les plugins de fournisseur peuvent désormais posséder la majeure partie de la logique spécifique au fournisseur tandis qu'OpenClaw conserve
la boucle d'inférence générique.

Répartition typique :

- `auth[].run` / `auth[].runNonInteractive` : le fournisseur possède les flux d'intégration/connexion
  pour `openclaw onboard`, `openclaw models auth`, et la configuration sans interface
- `wizard.setup` / `wizard.modelPicker` : le fournisseur possède les libellés de choix d'authentification,
  les alias hérités, les indices de liste d'autorisation pour l'intégration, et les entrées de configuration dans les sélecteurs d'intégration/de modèle
- `catalog` : le fournisseur apparaît dans `models.providers`
- `normalizeModelId` : le fournisseur normalise les identifiants de modèle hérités/preview avant
  la recherche ou la canonicalisation
- `normalizeTransport` : le fournisseur normalise `api` / `baseUrl` de la famille de transport
  avant l'assemblage générique du modèle ; OpenClaw vérifie d'abord le fournisseur correspondant,
  puis les autres plugins de fournisseur capables d'utiliser ce hook jusqu'à ce que l'un modifie réellement
  le transport
- `normalizeConfig` : le fournisseur normalise la config `models.providers.<id>` avant
  son utilisation à l'exécution ; OpenClaw vérifie d'abord le fournisseur correspondant, puis les autres
  plugins de fournisseur capables d'utiliser ce hook jusqu'à ce que l'un modifie réellement la config. Si aucun
  hook de fournisseur ne réécrit la config, les assistants Google intégrés de la famille Google
  normalisent encore les entrées prises en charge du fournisseur Google.
- `applyNativeStreamingUsageCompat` : le fournisseur applique des réécritures de compatibilité d'utilisation native en streaming pilotées par l'endpoint pour les fournisseurs de config
- `resolveConfigApiKey` : le fournisseur résout l'authentification par marqueur d'environnement pour les fournisseurs de config
  sans forcer le chargement complet de l'authentification à l'exécution. `amazon-bedrock` dispose aussi ici d'un
  résolveur intégré de marqueur d'environnement AWS, même si l'authentification d'exécution Bedrock utilise
  la chaîne par défaut du SDK AWS.
- `resolveSyntheticAuth` : le fournisseur peut exposer la disponibilité d'une authentification locale/autohébergée ou autre
  basée sur la config sans persister des secrets en clair
- `shouldDeferSyntheticProfileAuth` : le fournisseur peut marquer les espaces réservés des profils synthétiques stockés
  comme ayant une priorité inférieure à l'authentification basée sur l'environnement/la config
- `resolveDynamicModel` : le fournisseur accepte les identifiants de modèle qui ne sont pas encore présents dans le
  catalogue statique local
- `prepareDynamicModel` : le fournisseur a besoin d'un rafraîchissement des métadonnées avant de réessayer
  la résolution dynamique
- `normalizeResolvedModel` : le fournisseur a besoin de réécritures du transport ou de l'URL de base
- `contributeResolvedModelCompat` : le fournisseur contribue des indicateurs de compatibilité pour ses
  modèles fournisseur même lorsqu'ils arrivent via un autre transport compatible
- `capabilities` : le fournisseur publie des particularités de transcription/outillage/famille de fournisseur
- `normalizeToolSchemas` : le fournisseur nettoie les schémas d'outil avant que le
  runner embarqué ne les voie
- `inspectToolSchemas` : le fournisseur expose des avertissements de schéma spécifiques au transport
  après normalisation
- `resolveReasoningOutputMode` : le fournisseur choisit les contrats de sortie de raisonnement natifs ou balisés
- `prepareExtraParams` : le fournisseur applique des valeurs par défaut ou normalise les paramètres de requête par modèle
- `createStreamFn` : le fournisseur remplace le chemin de flux normal par un
  transport entièrement personnalisé
- `wrapStreamFn` : le fournisseur applique des wrappers de compatibilité requête/en-têtes/corps/modèle
- `resolveTransportTurnState` : le fournisseur fournit les
  en-têtes ou métadonnées natifs de transport par tour
- `resolveWebSocketSessionPolicy` : le fournisseur fournit les
  en-têtes natifs de session WebSocket ou la politique de refroidissement de session
- `createEmbeddingProvider` : le fournisseur possède le comportement des embeddings mémoire lorsqu'il
  relève du plugin de fournisseur plutôt que du commutateur central d'embeddings du noyau
- `formatApiKey` : le fournisseur formate les profils d'authentification stockés dans la chaîne
  `apiKey` attendue par le transport à l'exécution
- `refreshOAuth` : le fournisseur possède le rafraîchissement OAuth lorsque les rafraîchisseurs
  partagés `pi-ai` ne suffisent pas
- `buildAuthDoctorHint` : le fournisseur ajoute des conseils de réparation lorsque l'actualisation OAuth
  échoue
- `matchesContextOverflowError` : le fournisseur reconnaît les erreurs spécifiques au fournisseur
  de dépassement de fenêtre de contexte que les heuristiques génériques ne détecteraient pas
- `classifyFailoverReason` : le fournisseur mappe les erreurs brutes spécifiques au fournisseur de transport/API
  vers des raisons de basculement comme la limitation de débit ou la surcharge
- `isCacheTtlEligible` : le fournisseur décide quels identifiants de modèle amont prennent en charge le TTL du cache de prompt
- `buildMissingAuthMessage` : le fournisseur remplace l'erreur générique du magasin d'authentification
  par un conseil de récupération spécifique au fournisseur
- `suppressBuiltInModel` : le fournisseur masque les lignes amont obsolètes et peut renvoyer une
  erreur appartenant au fournisseur en cas d'échec de résolution directe
- `augmentModelCatalog` : le fournisseur ajoute des lignes de catalogue synthétiques/finales après
  la découverte et la fusion de config
- `isBinaryThinking` : le fournisseur possède l'UX de réflexion binaire activé/désactivé
- `supportsXHighThinking` : le fournisseur fait entrer certains modèles dans `xhigh`
- `resolveDefaultThinkingLevel` : le fournisseur possède la politique `/think` par défaut pour une
  famille de modèles
- `applyConfigDefaults` : le fournisseur applique des valeurs globales par défaut spécifiques au fournisseur
  lors de la matérialisation de la config en fonction du mode d'authentification, de l'environnement ou de la famille de modèles
- `isModernModelRef` : le fournisseur possède la correspondance des modèles préférés live/smoke
- `prepareRuntimeAuth` : le fournisseur transforme un identifiant configuré en un jeton d'exécution
  de courte durée
- `resolveUsageAuth` : le fournisseur résout les identifiants d'utilisation/quota pour `/usage`
  et les surfaces associées de statut/rapport
- `fetchUsageSnapshot` : le fournisseur possède la récupération/l'analyse de l'endpoint d'utilisation tandis que
  le noyau conserve le shell de résumé et le formatage
- `onModelSelected` : le fournisseur exécute des effets secondaires après sélection comme la
  télémétrie ou la tenue de session appartenant au fournisseur

Exemples intégrés actuels :

- `anthropic` : fallback de compatibilité ascendante Claude 4.6, conseils de réparation d'authentification, récupération de l'endpoint
  d'utilisation, métadonnées cache-TTL/famille de fournisseur, et valeurs par défaut globales de config
  sensibles à l'authentification
- `amazon-bedrock` : correspondance du dépassement de contexte possédée par le fournisseur et classification des
  raisons de basculement pour les erreurs Bedrock spécifiques de limitation/pas prêt, plus la famille de replay partagée `anthropic-by-model` pour les protections de politique de replay Claude uniquement
  sur le trafic Anthropic
- `anthropic-vertex` : protections de politique de replay Claude uniquement sur le trafic
  de messages Anthropic
- `openrouter` : identifiants de modèle pass-through, wrappers de requête, indices de capacité du fournisseur,
  assainissement des signatures de pensée Gemini sur le trafic proxy Gemini, injection de raisonnement proxy
  via la famille de flux `openrouter-thinking`, transfert des métadonnées de routage,
  et politique cache-TTL
- `github-copilot` : intégration/connexion de l'appareil, fallback de modèle à compatibilité ascendante,
  indices de transcription de réflexion Claude, échange de jeton d'exécution, et récupération de l'endpoint d'utilisation
- `openai` : fallback de compatibilité ascendante GPT-5.4, normalisation directe du transport OpenAI,
  conseils d'authentification manquante sensibles à Codex, suppression de Spark, lignes de catalogue synthétiques
  OpenAI/Codex, politique thinking/live-model, normalisation des alias de jetons d'utilisation
  (`input` / `output` et familles `prompt` / `completion`), la famille de flux partagée `openai-responses-defaults`
  pour les wrappers natifs OpenAI/Codex, métadonnées de famille de fournisseur, enregistrement intégré du fournisseur
  de génération d'images pour `gpt-image-1`, et enregistrement intégré du fournisseur
  de génération vidéo pour `sora-2`
- `google` et `google-gemini-cli` : fallback de compatibilité ascendante Gemini 3.1,
  validation native du replay Gemini, assainissement du replay bootstrap, mode de sortie de raisonnement
  balisé, correspondance des modèles modernes, enregistrement intégré du fournisseur de génération d'images
  pour les modèles Gemini image-preview, et enregistrement intégré du
  fournisseur de génération vidéo pour les modèles Veo ; l'OAuth Gemini CLI
  possède aussi le formatage des jetons de profil d'authentification, l'analyse des jetons d'utilisation et la récupération de l'endpoint de quota
  pour les surfaces d'utilisation
- `moonshot` : transport partagé, normalisation de la charge utile de réflexion appartenant au plugin
- `kilocode` : transport partagé, en-têtes de requête appartenant au plugin, normalisation de la charge utile de raisonnement,
  assainissement des signatures de pensée proxy-Gemini, et politique
  cache-TTL
- `zai` : fallback de compatibilité ascendante GLM-5, valeurs par défaut `tool_stream`, politique cache-TTL,
  politique binary-thinking/live-model, et authentification d'utilisation + récupération du quota ;
  les identifiants inconnus `glm-5*` sont synthétisés à partir du modèle intégré `glm-4.7`
- `xai` : normalisation native du transport Responses, réécritures d'alias `/fast` pour
  les variantes rapides de Grok, `tool_stream` par défaut, nettoyage spécifique à xAI des schémas d'outil /
  charges utiles de raisonnement, et enregistrement intégré du fournisseur de génération vidéo
  pour `grok-imagine-video`
- `mistral` : métadonnées de capacité appartenant au plugin
- `opencode` et `opencode-go` : métadonnées de capacité appartenant au plugin plus
  assainissement des signatures de pensée proxy-Gemini
- `alibaba` : catalogue de génération vidéo appartenant au plugin pour les références directes de modèles Wan
  comme `alibaba/wan2.6-t2v`
- `byteplus` : catalogues appartenant au plugin plus enregistrement intégré du fournisseur de génération vidéo
  pour les modèles Seedance texte-vers-vidéo/image-vers-vidéo
- `fal` : enregistrement intégré du fournisseur de génération vidéo pour les modèles vidéo tiers hébergés
  et enregistrement du fournisseur de génération d'images pour les modèles d'image FLUX, plus enregistrement intégré
  du fournisseur de génération vidéo pour les modèles vidéo tiers hébergés
- `cloudflare-ai-gateway`, `huggingface`, `kimi`, `nvidia`, `qianfan`,
  `stepfun`, `synthetic`, `venice`, `vercel-ai-gateway` et `volcengine` :
  catalogues appartenant au plugin uniquement
- `qwen` : catalogues appartenant au plugin pour les modèles texte plus enregistrements partagés des fournisseurs
  media-understanding et génération vidéo pour ses surfaces multimodales ; la génération vidéo Qwen utilise les endpoints vidéo DashScope Standard
  avec des modèles Wan intégrés comme `wan2.6-t2v` et `wan2.7-r2v`
- `runway` : enregistrement du fournisseur de génération vidéo appartenant au plugin pour les modèles natifs
  Runway basés sur des tâches comme `gen4.5`
- `minimax` : catalogues appartenant au plugin, enregistrement intégré du fournisseur de génération vidéo
  pour les modèles vidéo Hailuo, enregistrement intégré du fournisseur de génération d'images
  pour `image-01`, sélection hybride de politique de replay Anthropic/OpenAI,
  et logique auth/snapshot d'utilisation
- `together` : catalogues appartenant au plugin plus enregistrement intégré du fournisseur de génération vidéo
  pour les modèles vidéo Wan
- `xiaomi` : catalogues appartenant au plugin plus logique auth/snapshot d'utilisation

Le plugin `openai` intégré possède désormais les deux identifiants de fournisseur : `openai` et
`openai-codex`.

Cela couvre les fournisseurs qui s'intègrent encore dans les transports normaux d'OpenClaw. Un fournisseur
qui a besoin d'un exécuteur de requêtes entièrement personnalisé relève d'une surface d'extension distincte et plus profonde.

## Rotation des clés API

- Prend en charge la rotation générique des fournisseurs pour certains fournisseurs sélectionnés.
- Configurez plusieurs clés via :
  - `OPENCLAW_LIVE_<PROVIDER>_KEY` (remplacement live unique, priorité la plus élevée)
  - `<PROVIDER>_API_KEYS` (liste séparée par des virgules ou des points-virgules)
  - `<PROVIDER>_API_KEY` (clé principale)
  - `<PROVIDER>_API_KEY_*` (liste numérotée, par ex. `<PROVIDER>_API_KEY_1`)
- Pour les fournisseurs Google, `GOOGLE_API_KEY` est aussi inclus comme fallback.
- L'ordre de sélection des clés préserve la priorité et déduplique les valeurs.
- Les requêtes sont réessayées avec la clé suivante uniquement sur les réponses de limitation de débit (par
  exemple `429`, `rate_limit`, `quota`, `resource exhausted`, `Too many
concurrent requests`, `ThrottlingException`, `concurrency limit reached`,
  `workers_ai ... quota limit exceeded`, ou des messages périodiques de limite d'utilisation).
- Les échecs hors limitation de débit échouent immédiatement ; aucune rotation de clé n'est tentée.
- Lorsque toutes les clés candidates échouent, l'erreur finale du dernier essai est renvoyée.

## Fournisseurs intégrés (catalogue pi-ai)

OpenClaw est livré avec le catalogue pi-ai. Ces fournisseurs ne nécessitent **aucune**
config `models.providers` ; il suffit de définir l'authentification + choisir un modèle.

### OpenAI

- Fournisseur : `openai`
- Authentification : `OPENAI_API_KEY`
- Rotation facultative : `OPENAI_API_KEYS`, `OPENAI_API_KEY_1`, `OPENAI_API_KEY_2`, plus `OPENCLAW_LIVE_OPENAI_KEY` (remplacement unique)
- Exemples de modèles : `openai/gpt-5.4`, `openai/gpt-5.4-pro`
- CLI : `openclaw onboard --auth-choice openai-api-key`
- Le transport par défaut est `auto` (WebSocket d'abord, repli SSE)
- Remplacez par modèle via `agents.defaults.models["openai/<model>"].params.transport` (`"sse"`, `"websocket"` ou `"auto"`)
- Le préchauffage WebSocket OpenAI Responses est activé par défaut via `params.openaiWsWarmup` (`true`/`false`)
- Le traitement prioritaire OpenAI peut être activé via `agents.defaults.models["openai/<model>"].params.serviceTier`
- `/fast` et `params.fastMode` mappent les requêtes Responses directes `openai/*` vers `service_tier=priority` sur `api.openai.com`
- Utilisez `params.serviceTier` lorsque vous voulez un niveau explicite au lieu du basculement partagé `/fast`
- Les en-têtes d'attribution OpenClaw cachés (`originator`, `version`,
  `User-Agent`) s'appliquent uniquement au trafic OpenAI natif vers `api.openai.com`, pas
  aux proxys génériques compatibles OpenAI
- Les routes OpenAI natives conservent aussi `store` de Responses, les indices de cache de prompt, et
  la mise en forme de charge utile de compatibilité de raisonnement OpenAI ; les routes proxy non
- `openai/gpt-5.3-codex-spark` est volontairement supprimé dans OpenClaw car l'API OpenAI live le rejette ; Spark est traité comme réservé à Codex

```json5
{
  agents: { defaults: { model: { primary: "openai/gpt-5.4" } } },
}
```

### Anthropic

- Fournisseur : `anthropic`
- Authentification : `ANTHROPIC_API_KEY`
- Rotation facultative : `ANTHROPIC_API_KEYS`, `ANTHROPIC_API_KEY_1`, `ANTHROPIC_API_KEY_2`, plus `OPENCLAW_LIVE_ANTHROPIC_KEY` (remplacement unique)
- Exemple de modèle : `anthropic/claude-opus-4-6`
- CLI : `openclaw onboard --auth-choice apiKey`
- Les requêtes Anthropic publiques directes prennent en charge le basculement partagé `/fast` et `params.fastMode`, y compris le trafic authentifié par clé API et OAuth envoyé vers `api.anthropic.com` ; OpenClaw le mappe vers Anthropic `service_tier` (`auto` vs `standard_only`)
- Remarque Anthropic : le personnel d'Anthropic nous a indiqué que l'utilisation de Claude CLI façon OpenClaw est de nouveau autorisée, donc OpenClaw traite la réutilisation de Claude CLI et l'usage de `claude -p` comme autorisés pour cette intégration tant qu'Anthropic ne publie pas une nouvelle politique.
- Le setup-token Anthropic reste disponible comme chemin de jeton OpenClaw pris en charge, mais OpenClaw préfère désormais la réutilisation de Claude CLI et `claude -p` quand disponible.

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

### OpenAI Code (Codex)

- Fournisseur : `openai-codex`
- Authentification : OAuth (ChatGPT)
- Exemple de modèle : `openai-codex/gpt-5.4`
- CLI : `openclaw onboard --auth-choice openai-codex` ou `openclaw models auth login --provider openai-codex`
- Le transport par défaut est `auto` (WebSocket d'abord, repli SSE)
- Remplacez par modèle via `agents.defaults.models["openai-codex/<model>"].params.transport` (`"sse"`, `"websocket"` ou `"auto"`)
- `params.serviceTier` est aussi transféré sur les requêtes natives Codex Responses (`chatgpt.com/backend-api`)
- Les en-têtes d'attribution OpenClaw cachés (`originator`, `version`,
  `User-Agent`) sont attachés uniquement au trafic Codex natif vers
  `chatgpt.com/backend-api`, pas aux proxys génériques compatibles OpenAI
- Partage le même basculement `/fast` et la même config `params.fastMode` que `openai/*` direct ; OpenClaw le mappe vers `service_tier=priority`
- `openai-codex/gpt-5.3-codex-spark` reste disponible lorsque le catalogue OAuth Codex l'expose ; dépend des droits
- `openai-codex/gpt-5.4` conserve `contextWindow = 1050000` natif et une valeur d'exécution par défaut `contextTokens = 272000` ; remplacez la limite d'exécution avec `models.providers.openai-codex.models[].contextTokens`
- Remarque de politique : OpenAI Codex OAuth est explicitement pris en charge pour les outils/flux de travail externes comme OpenClaw.

```json5
{
  agents: { defaults: { model: { primary: "openai-codex/gpt-5.4" } } },
}
```

```json5
{
  models: {
    providers: {
      "openai-codex": {
        models: [{ id: "gpt-5.4", contextTokens: 160000 }],
      },
    },
  },
}
```

### Autres options hébergées de type abonnement

- [Qwen Cloud](/fr/providers/qwen) : surface de fournisseur Qwen Cloud plus mappage des endpoints Alibaba DashScope et Coding Plan
- [MiniMax](/fr/providers/minimax) : accès MiniMax Coding Plan via OAuth ou clé API
- [GLM Models](/fr/providers/glm) : endpoints Z.AI Coding Plan ou API générale

### OpenCode

- Authentification : `OPENCODE_API_KEY` (ou `OPENCODE_ZEN_API_KEY`)
- Fournisseur d'exécution Zen : `opencode`
- Fournisseur d'exécution Go : `opencode-go`
- Exemples de modèles : `opencode/claude-opus-4-6`, `opencode-go/kimi-k2.5`
- CLI : `openclaw onboard --auth-choice opencode-zen` ou `openclaw onboard --auth-choice opencode-go`

```json5
{
  agents: { defaults: { model: { primary: "opencode/claude-opus-4-6" } } },
}
```

### Google Gemini (clé API)

- Fournisseur : `google`
- Authentification : `GEMINI_API_KEY`
- Rotation facultative : `GEMINI_API_KEYS`, `GEMINI_API_KEY_1`, `GEMINI_API_KEY_2`, fallback `GOOGLE_API_KEY`, et `OPENCLAW_LIVE_GEMINI_KEY` (remplacement unique)
- Exemples de modèles : `google/gemini-3.1-pro-preview`, `google/gemini-3-flash-preview`
- Compatibilité : l'ancienne config OpenClaw utilisant `google/gemini-3.1-flash-preview` est normalisée en `google/gemini-3-flash-preview`
- CLI : `openclaw onboard --auth-choice gemini-api-key`
- Les exécutions Gemini directes acceptent aussi `agents.defaults.models["google/<model>"].params.cachedContent`
  (ou l'ancien `cached_content`) pour transférer un handle natif du fournisseur
  `cachedContents/...` ; les hits de cache Gemini remontent comme `cacheRead` dans OpenClaw

### Google Vertex et Gemini CLI

- Fournisseurs : `google-vertex`, `google-gemini-cli`
- Authentification : Vertex utilise gcloud ADC ; Gemini CLI utilise son flux OAuth
- Avertissement : l'OAuth Gemini CLI dans OpenClaw est une intégration non officielle. Certains utilisateurs ont signalé des restrictions de compte Google après l'utilisation de clients tiers. Consultez les conditions de Google et utilisez un compte non critique si vous choisissez de continuer.
- L'OAuth Gemini CLI est fourni dans le plugin `google` intégré.
  - Installez d'abord Gemini CLI :
    - `brew install gemini-cli`
    - ou `npm install -g @google/gemini-cli`
  - Activez-le : `openclaw plugins enable google`
  - Connectez-vous : `openclaw models auth login --provider google-gemini-cli --set-default`
  - Modèle par défaut : `google-gemini-cli/gemini-3-flash-preview`
  - Remarque : vous ne collez **pas** un identifiant client ou un secret dans `openclaw.json`. Le flux de connexion CLI stocke
    les jetons dans les profils d'authentification sur l'hôte de la passerelle.
  - Si les requêtes échouent après la connexion, définissez `GOOGLE_CLOUD_PROJECT` ou `GOOGLE_CLOUD_PROJECT_ID` sur l'hôte de la passerelle.
  - Les réponses JSON Gemini CLI sont analysées depuis `response` ; l'utilisation revient à
    `stats`, avec `stats.cached` normalisé en `cacheRead` OpenClaw.

### Z.AI (GLM)

- Fournisseur : `zai`
- Authentification : `ZAI_API_KEY`
- Exemple de modèle : `zai/glm-5.1`
- CLI : `openclaw onboard --auth-choice zai-api-key`
  - Alias : `z.ai/*` et `z-ai/*` sont normalisés en `zai/*`
  - `zai-api-key` détecte automatiquement l'endpoint Z.AI correspondant ; `zai-coding-global`, `zai-coding-cn`, `zai-global` et `zai-cn` forcent une surface spécifique

### Vercel AI Gateway

- Fournisseur : `vercel-ai-gateway`
- Authentification : `AI_GATEWAY_API_KEY`
- Exemple de modèle : `vercel-ai-gateway/anthropic/claude-opus-4.6`
- CLI : `openclaw onboard --auth-choice ai-gateway-api-key`

### Kilo Gateway

- Fournisseur : `kilocode`
- Authentification : `KILOCODE_API_KEY`
- Exemple de modèle : `kilocode/kilo/auto`
- CLI : `openclaw onboard --auth-choice kilocode-api-key`
- URL de base : `https://api.kilo.ai/api/gateway/`
- Le catalogue statique de secours inclut `kilocode/kilo/auto` ; la découverte live
  `https://api.kilo.ai/api/gateway/models` peut étendre davantage le
  catalogue d'exécution.
- Le routage amont exact derrière `kilocode/kilo/auto` appartient à Kilo Gateway,
  il n'est pas codé en dur dans OpenClaw.

Consultez [/providers/kilocode](/fr/providers/kilocode) pour les détails de configuration.

### Autres plugins de fournisseur intégrés

- OpenRouter : `openrouter` (`OPENROUTER_API_KEY`)
- Exemple de modèle : `openrouter/auto`
- OpenClaw applique les en-têtes d'attribution d'application documentés par OpenRouter uniquement lorsque
  la requête cible réellement `openrouter.ai`
- Les marqueurs `cache_control` Anthropic spécifiques à OpenRouter sont aussi limités
  aux routes OpenRouter vérifiées, et non à des URL proxy arbitraires
- OpenRouter reste sur le chemin de type proxy compatible OpenAI, donc la mise en forme de requête uniquement native à
  OpenAI (`serviceTier`, `store` de Responses,
  indices de cache de prompt, charges utiles de compatibilité de raisonnement OpenAI) n'est pas transférée
- Les références OpenRouter adossées à Gemini conservent uniquement l'assainissement des signatures de pensée proxy-Gemini ;
  la validation native du replay Gemini et les réécritures bootstrap restent désactivées
- Kilo Gateway : `kilocode` (`KILOCODE_API_KEY`)
- Exemple de modèle : `kilocode/kilo/auto`
- Les références Kilo adossées à Gemini conservent le même chemin d'assainissement des signatures de pensée
  proxy-Gemini ; `kilocode/kilo/auto` et les autres indices sans prise en charge du raisonnement proxy
  ignorent l'injection de raisonnement proxy
- MiniMax : `minimax` (clé API) et `minimax-portal` (OAuth)
- Authentification : `MINIMAX_API_KEY` pour `minimax` ; `MINIMAX_OAUTH_TOKEN` ou `MINIMAX_API_KEY` pour `minimax-portal`
- Exemple de modèle : `minimax/MiniMax-M2.7` ou `minimax-portal/MiniMax-M2.7`
- La configuration de l'intégration/de la clé API MiniMax écrit des définitions explicites du modèle M2.7 avec
  `input: ["text", "image"]` ; le catalogue de fournisseur intégré conserve les références de chat
  en texte seul jusqu'à la matérialisation de cette config fournisseur
- Moonshot : `moonshot` (`MOONSHOT_API_KEY`)
- Exemple de modèle : `moonshot/kimi-k2.5`
- Kimi Coding : `kimi` (`KIMI_API_KEY` ou `KIMICODE_API_KEY`)
- Exemple de modèle : `kimi/kimi-code`
- Qianfan : `qianfan` (`QIANFAN_API_KEY`)
- Exemple de modèle : `qianfan/deepseek-v3.2`
- Qwen Cloud : `qwen` (`QWEN_API_KEY`, `MODELSTUDIO_API_KEY` ou `DASHSCOPE_API_KEY`)
- Exemple de modèle : `qwen/qwen3.5-plus`
- NVIDIA : `nvidia` (`NVIDIA_API_KEY`)
- Exemple de modèle : `nvidia/nvidia/llama-3.1-nemotron-70b-instruct`
- StepFun : `stepfun` / `stepfun-plan` (`STEPFUN_API_KEY`)
- Exemples de modèles : `stepfun/step-3.5-flash`, `stepfun-plan/step-3.5-flash-2603`
- Together : `together` (`TOGETHER_API_KEY`)
- Exemple de modèle : `together/moonshotai/Kimi-K2.5`
- Venice : `venice` (`VENICE_API_KEY`)
- Xiaomi : `xiaomi` (`XIAOMI_API_KEY`)
- Exemple de modèle : `xiaomi/mimo-v2-flash`
- Vercel AI Gateway : `vercel-ai-gateway` (`AI_GATEWAY_API_KEY`)
- Hugging Face Inference : `huggingface` (`HUGGINGFACE_HUB_TOKEN` ou `HF_TOKEN`)
- Cloudflare AI Gateway : `cloudflare-ai-gateway` (`CLOUDFLARE_AI_GATEWAY_API_KEY`)
- Volcengine : `volcengine` (`VOLCANO_ENGINE_API_KEY`)
- Exemple de modèle : `volcengine-plan/ark-code-latest`
- BytePlus : `byteplus` (`BYTEPLUS_API_KEY`)
- Exemple de modèle : `byteplus-plan/ark-code-latest`
- xAI : `xai` (`XAI_API_KEY`)
  - Les requêtes xAI natives intégrées utilisent le chemin xAI Responses
  - `/fast` ou `params.fastMode: true` réécrit `grok-3`, `grok-3-mini`,
    `grok-4` et `grok-4-0709` vers leurs variantes `*-fast`
  - `tool_stream` est activé par défaut ; définissez
    `agents.defaults.models["xai/<model>"].params.tool_stream` sur `false` pour
    le désactiver
- Mistral : `mistral` (`MISTRAL_API_KEY`)
- Exemple de modèle : `mistral/mistral-large-latest`
- CLI : `openclaw onboard --auth-choice mistral-api-key`
- Groq : `groq` (`GROQ_API_KEY`)
- Cerebras : `cerebras` (`CEREBRAS_API_KEY`)
  - Les modèles GLM sur Cerebras utilisent les identifiants `zai-glm-4.7` et `zai-glm-4.6`.
  - URL de base compatible OpenAI : `https://api.cerebras.ai/v1`.
- GitHub Copilot : `github-copilot` (`COPILOT_GITHUB_TOKEN` / `GH_TOKEN` / `GITHUB_TOKEN`)
- Exemple de modèle Hugging Face Inference : `huggingface/deepseek-ai/DeepSeek-R1` ; CLI : `openclaw onboard --auth-choice huggingface-api-key`. Consultez [Hugging Face (Inference)](/fr/providers/huggingface).

## Fournisseurs via `models.providers` (personnalisé/URL de base)

Utilisez `models.providers` (ou `models.json`) pour ajouter des fournisseurs **personnalisés** ou
des proxys compatibles OpenAI/Anthropic.

Beaucoup des plugins de fournisseur intégrés ci-dessous publient déjà un catalogue par défaut.
Utilisez des entrées explicites `models.providers.<id>` uniquement lorsque vous voulez remplacer l'
URL de base, les en-têtes ou la liste de modèles par défaut.

### Moonshot AI (Kimi)

Moonshot est fourni comme plugin de fournisseur intégré. Utilisez le fournisseur intégré par
défaut et ajoutez une entrée explicite `models.providers.moonshot` uniquement lorsque vous
devez remplacer l'URL de base ou les métadonnées du modèle :

- Fournisseur : `moonshot`
- Authentification : `MOONSHOT_API_KEY`
- Exemple de modèle : `moonshot/kimi-k2.5`
- CLI : `openclaw onboard --auth-choice moonshot-api-key` ou `openclaw onboard --auth-choice moonshot-api-key-cn`

Identifiants de modèles Kimi K2 :

[//]: # "moonshot-kimi-k2-model-refs:start"

- `moonshot/kimi-k2.5`
- `moonshot/kimi-k2-thinking`
- `moonshot/kimi-k2-thinking-turbo`
- `moonshot/kimi-k2-turbo`

[//]: # "moonshot-kimi-k2-model-refs:end"

```json5
{
  agents: {
    defaults: { model: { primary: "moonshot/kimi-k2.5" } },
  },
  models: {
    mode: "merge",
    providers: {
      moonshot: {
        baseUrl: "https://api.moonshot.ai/v1",
        apiKey: "${MOONSHOT_API_KEY}",
        api: "openai-completions",
        models: [{ id: "kimi-k2.5", name: "Kimi K2.5" }],
      },
    },
  },
}
```

### Kimi Coding

Kimi Coding utilise l'endpoint compatible Anthropic de Moonshot AI :

- Fournisseur : `kimi`
- Authentification : `KIMI_API_KEY`
- Exemple de modèle : `kimi/kimi-code`

```json5
{
  env: { KIMI_API_KEY: "sk-..." },
  agents: {
    defaults: { model: { primary: "kimi/kimi-code" } },
  },
}
```

L'ancien `kimi/k2p5` reste accepté comme identifiant de modèle de compatibilité.

### Volcano Engine (Doubao)

Volcano Engine (火山引擎) fournit un accès à Doubao et à d'autres modèles en Chine.

- Fournisseur : `volcengine` (coding : `volcengine-plan`)
- Authentification : `VOLCANO_ENGINE_API_KEY`
- Exemple de modèle : `volcengine-plan/ark-code-latest`
- CLI : `openclaw onboard --auth-choice volcengine-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "volcengine-plan/ark-code-latest" } },
  },
}
```

L'intégration utilise par défaut la surface coding, mais le catalogue général `volcengine/*`
est enregistré en même temps.

Dans les sélecteurs de modèle d'intégration/configuration, le choix d'authentification Volcengine préfère à la fois les lignes
`volcengine/*` et `volcengine-plan/*`. Si ces modèles ne sont pas encore chargés,
OpenClaw revient au catalogue non filtré au lieu d'afficher un sélecteur vide
limité au fournisseur.

Modèles disponibles :

- `volcengine/doubao-seed-1-8-251228` (Doubao Seed 1.8)
- `volcengine/doubao-seed-code-preview-251028`
- `volcengine/kimi-k2-5-260127` (Kimi K2.5)
- `volcengine/glm-4-7-251222` (GLM 4.7)
- `volcengine/deepseek-v3-2-251201` (DeepSeek V3.2 128K)

Modèles coding (`volcengine-plan`) :

- `volcengine-plan/ark-code-latest`
- `volcengine-plan/doubao-seed-code`
- `volcengine-plan/kimi-k2.5`
- `volcengine-plan/kimi-k2-thinking`
- `volcengine-plan/glm-4.7`

### BytePlus (International)

BytePlus ARK fournit un accès aux mêmes modèles que Volcano Engine pour les utilisateurs internationaux.

- Fournisseur : `byteplus` (coding : `byteplus-plan`)
- Authentification : `BYTEPLUS_API_KEY`
- Exemple de modèle : `byteplus-plan/ark-code-latest`
- CLI : `openclaw onboard --auth-choice byteplus-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "byteplus-plan/ark-code-latest" } },
  },
}
```

L'intégration utilise par défaut la surface coding, mais le catalogue général `byteplus/*`
est enregistré en même temps.

Dans les sélecteurs de modèle d'intégration/configuration, le choix d'authentification BytePlus préfère à la fois les lignes
`byteplus/*` et `byteplus-plan/*`. Si ces modèles ne sont pas encore chargés,
OpenClaw revient au catalogue non filtré au lieu d'afficher un sélecteur vide
limité au fournisseur.

Modèles disponibles :

- `byteplus/seed-1-8-251228` (Seed 1.8)
- `byteplus/kimi-k2-5-260127` (Kimi K2.5)
- `byteplus/glm-4-7-251222` (GLM 4.7)

Modèles coding (`byteplus-plan`) :

- `byteplus-plan/ark-code-latest`
- `byteplus-plan/doubao-seed-code`
- `byteplus-plan/kimi-k2.5`
- `byteplus-plan/kimi-k2-thinking`
- `byteplus-plan/glm-4.7`

### Synthetic

Synthetic fournit des modèles compatibles Anthropic derrière le fournisseur `synthetic` :

- Fournisseur : `synthetic`
- Authentification : `SYNTHETIC_API_KEY`
- Exemple de modèle : `synthetic/hf:MiniMaxAI/MiniMax-M2.5`
- CLI : `openclaw onboard --auth-choice synthetic-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "synthetic/hf:MiniMaxAI/MiniMax-M2.5" } },
  },
  models: {
    mode: "merge",
    providers: {
      synthetic: {
        baseUrl: "https://api.synthetic.new/anthropic",
        apiKey: "${SYNTHETIC_API_KEY}",
        api: "anthropic-messages",
        models: [{ id: "hf:MiniMaxAI/MiniMax-M2.5", name: "MiniMax M2.5" }],
      },
    },
  },
}
```

### MiniMax

MiniMax est configuré via `models.providers` car il utilise des endpoints personnalisés :

- MiniMax OAuth (global) : `--auth-choice minimax-global-oauth`
- MiniMax OAuth (CN) : `--auth-choice minimax-cn-oauth`
- Clé API MiniMax (global) : `--auth-choice minimax-global-api`
- Clé API MiniMax (CN) : `--auth-choice minimax-cn-api`
- Authentification : `MINIMAX_API_KEY` pour `minimax` ; `MINIMAX_OAUTH_TOKEN` ou
  `MINIMAX_API_KEY` pour `minimax-portal`

Consultez [/providers/minimax](/fr/providers/minimax) pour les détails de configuration, les options de modèle et les extraits de config.

Sur le chemin de streaming compatible Anthropic de MiniMax, OpenClaw désactive la réflexion par
défaut sauf si vous la définissez explicitement, et `/fast on` réécrit
`MiniMax-M2.7` en `MiniMax-M2.7-highspeed`.

Répartition des capacités possédées par le plugin :

- Les valeurs par défaut texte/chat restent sur `minimax/MiniMax-M2.7`
- La génération d'images est `minimax/image-01` ou `minimax-portal/image-01`
- La compréhension d'image est `MiniMax-VL-01`, appartenant au plugin, sur les deux chemins d'authentification MiniMax
- La recherche web reste sur l'identifiant de fournisseur `minimax`

### Ollama

Ollama est fourni comme plugin de fournisseur intégré et utilise l'API native d'Ollama :

- Fournisseur : `ollama`
- Authentification : aucune requise (serveur local)
- Exemple de modèle : `ollama/llama3.3`
- Installation : [https://ollama.com/download](https://ollama.com/download)

```bash
# Installez Ollama, puis récupérez un modèle :
ollama pull llama3.3
```

```json5
{
  agents: {
    defaults: { model: { primary: "ollama/llama3.3" } },
  },
}
```

Ollama est détecté localement à `http://127.0.0.1:11434` lorsque vous l'activez avec
`OLLAMA_API_KEY`, et le plugin de fournisseur intégré ajoute Ollama directement à
`openclaw onboard` et au sélecteur de modèle. Consultez [/providers/ollama](/fr/providers/ollama)
pour l'intégration, le mode cloud/local et la configuration personnalisée.

### vLLM

vLLM est fourni comme plugin de fournisseur intégré pour les serveurs
compatibles OpenAI locaux/autohébergés :

- Fournisseur : `vllm`
- Authentification : facultative (dépend de votre serveur)
- URL de base par défaut : `http://127.0.0.1:8000/v1`

Pour activer l'auto-détection en local (n'importe quelle valeur fonctionne si votre serveur n'impose pas d'authentification) :

```bash
export VLLM_API_KEY="vllm-local"
```

Définissez ensuite un modèle (remplacez-le par l'un des identifiants renvoyés par `/v1/models`) :

```json5
{
  agents: {
    defaults: { model: { primary: "vllm/your-model-id" } },
  },
}
```

Consultez [/providers/vllm](/fr/providers/vllm) pour les détails.

### SGLang

SGLang est fourni comme plugin de fournisseur intégré pour des serveurs
compatibles OpenAI autohébergés rapides :

- Fournisseur : `sglang`
- Authentification : facultative (dépend de votre serveur)
- URL de base par défaut : `http://127.0.0.1:30000/v1`

Pour activer l'auto-détection en local (n'importe quelle valeur fonctionne si votre serveur n'impose pas
d'authentification) :

```bash
export SGLANG_API_KEY="sglang-local"
```

Définissez ensuite un modèle (remplacez-le par l'un des identifiants renvoyés par `/v1/models`) :

```json5
{
  agents: {
    defaults: { model: { primary: "sglang/your-model-id" } },
  },
}
```

Consultez [/providers/sglang](/fr/providers/sglang) pour les détails.

### Proxys locaux (LM Studio, vLLM, LiteLLM, etc.)

Exemple (compatible OpenAI) :

```json5
{
  agents: {
    defaults: {
      model: { primary: "lmstudio/my-local-model" },
      models: { "lmstudio/my-local-model": { alias: "Local" } },
    },
  },
  models: {
    providers: {
      lmstudio: {
        baseUrl: "http://localhost:1234/v1",
        apiKey: "LMSTUDIO_KEY",
        api: "openai-completions",
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 200000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

Remarques :

- Pour les fournisseurs personnalisés, `reasoning`, `input`, `cost`, `contextWindow` et `maxTokens` sont facultatifs.
  Lorsqu'ils sont omis, OpenClaw utilise par défaut :
  - `reasoning: false`
  - `input: ["text"]`
  - `cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 }`
  - `contextWindow: 200000`
  - `maxTokens: 8192`
- Recommandé : définissez des valeurs explicites qui correspondent aux limites de votre proxy/modèle.
- Pour `api: "openai-completions"` sur des endpoints non natifs (tout `baseUrl` non vide dont l'hôte n'est pas `api.openai.com`), OpenClaw force `compat.supportsDeveloperRole: false` pour éviter les erreurs fournisseur 400 sur les rôles `developer` non pris en charge.
- Les routes de type proxy compatibles OpenAI ignorent aussi la mise en forme de requête uniquement native à OpenAI :
  pas de `service_tier`, pas de `store` de Responses, pas d'indices de cache de prompt, pas de
  mise en forme de charge utile de compatibilité de raisonnement OpenAI, ni d'en-têtes d'attribution OpenClaw
  cachés.
- Si `baseUrl` est vide/omis, OpenClaw conserve le comportement OpenAI par défaut (qui se résout vers `api.openai.com`).
- Pour des raisons de sécurité, un `compat.supportsDeveloperRole: true` explicite est toujours remplacé sur les endpoints non natifs `openai-completions`.

## Exemples CLI

```bash
openclaw onboard --auth-choice opencode-zen
openclaw models set opencode/claude-opus-4-6
openclaw models list
```

Voir aussi : [/gateway/configuration](/fr/gateway/configuration) pour des exemples complets de configuration.

## Lié

- [Models](/fr/concepts/models) — configuration des modèles et alias
- [Model Failover](/fr/concepts/model-failover) — chaînes de secours et comportement de nouvelle tentative
- [Configuration Reference](/fr/gateway/configuration-reference#agent-defaults) — clés de config du modèle
- [Providers](/fr/providers) — guides de configuration par fournisseur
