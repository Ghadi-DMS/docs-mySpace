---
read_when:
    - Vous devez savoir depuis quel sous-chemin du SDK importer
    - Vous voulez une référence pour toutes les méthodes d'enregistrement sur OpenClawPluginApi
    - Vous recherchez une exportation spécifique du SDK
sidebarTitle: SDK Overview
summary: Carte des imports, référence de l'API d'enregistrement et architecture du SDK
title: Vue d'ensemble du Plugin SDK
x-i18n:
    generated_at: "2026-04-08T06:52:19Z"
    model: gpt-5.4
    provider: openai
    source_hash: a9141dc0303c91974fe693ce48ad9f7dc4179418ae262a96011ad565aae87d21
    source_path: plugins/sdk-overview.md
    workflow: 15
---

# Vue d'ensemble du Plugin SDK

Le SDK de plugin est le contrat typé entre les plugins et le cœur. Cette page
est la référence pour **quoi importer** et **ce que vous pouvez enregistrer**.

<Tip>
  **Vous cherchez un guide pratique ?**
  - Premier plugin ? Commencez par [Getting Started](/fr/plugins/building-plugins)
  - Plugin de canal ? Voir [Channel Plugins](/fr/plugins/sdk-channel-plugins)
  - Plugin de fournisseur ? Voir [Provider Plugins](/fr/plugins/sdk-provider-plugins)
</Tip>

## Convention d'import

Importez toujours depuis un sous-chemin spécifique :

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
```

Chaque sous-chemin est un module petit et autonome. Cela permet un démarrage
rapide et évite les problèmes de dépendances circulaires. Pour les helpers
d'entrée/de build spécifiques aux canaux, préférez `openclaw/plugin-sdk/channel-core` ; gardez `openclaw/plugin-sdk/core` pour
la surface ombrelle plus large et les helpers partagés comme
`buildChannelConfigSchema`.

N'ajoutez pas et ne dépendez pas de points d'entrée de commodité nommés d'après des fournisseurs tels que
`openclaw/plugin-sdk/slack`, `openclaw/plugin-sdk/discord`,
`openclaw/plugin-sdk/signal`, `openclaw/plugin-sdk/whatsapp`, ni de
points d'entrée helpers à marque de canal. Les plugins fournis doivent composer des
sous-chemins SDK génériques dans leurs propres barrels `api.ts` ou `runtime-api.ts`, et le cœur
doit soit utiliser ces barrels locaux au plugin, soit ajouter un contrat SDK générique étroit lorsque le besoin est réellement inter-canal.

La carte d'export générée contient toujours un petit ensemble de points d'entrée helpers de plugins fournis
tels que `plugin-sdk/feishu`, `plugin-sdk/feishu-setup`,
`plugin-sdk/zalo`, `plugin-sdk/zalo-setup` et `plugin-sdk/matrix*`. Ces
sous-chemins existent uniquement pour la maintenance et la compatibilité des plugins fournis ; ils
sont volontairement omis du tableau commun ci-dessous et ne constituent pas le chemin d'import recommandé pour les nouveaux plugins tiers.

## Référence des sous-chemins

Les sous-chemins les plus couramment utilisés, regroupés par usage. La liste complète générée des
plus de 200 sous-chemins se trouve dans `scripts/lib/plugin-sdk-entrypoints.json`.

Les sous-chemins helpers réservés aux plugins fournis apparaissent toujours dans cette liste générée.
Considérez-les comme des surfaces de détail d'implémentation/de compatibilité sauf si une page de documentation
en promeut explicitement un comme public.

### Entrée de plugin

| Sous-chemin                 | Exportations clés                                                                                                                     |
| --------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/plugin-entry`   | `definePluginEntry`                                                                                                                    |
| `plugin-sdk/core`           | `defineChannelPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase`, `defineSetupPluginEntry`, `buildChannelConfigSchema` |
| `plugin-sdk/config-schema`  | `OpenClawSchema`                                                                                                                       |
| `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry`                                                                                                      |

<AccordionGroup>
  <Accordion title="Sous-chemins de canal">
    | Sous-chemin | Exportations clés |
    | --- | --- |
    | `plugin-sdk/channel-core` | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
    | `plugin-sdk/config-schema` | Export du schéma Zod racine `openclaw.json` (`OpenClawSchema`) |
    | `plugin-sdk/channel-setup` | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, ainsi que `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
    | `plugin-sdk/setup` | Helpers partagés d'assistant de configuration, invites de liste d'autorisation, constructeurs d'état de configuration |
    | `plugin-sdk/setup-runtime` | `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
    | `plugin-sdk/setup-adapter-runtime` | `createEnvPatchedAccountSetupAdapter` |
    | `plugin-sdk/setup-tools` | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
    | `plugin-sdk/account-core` | Helpers de config multi-comptes/contrôle d'action, helpers de repli sur le compte par défaut |
    | `plugin-sdk/account-id` | `DEFAULT_ACCOUNT_ID`, helpers de normalisation d'identifiant de compte |
    | `plugin-sdk/account-resolution` | Helpers de recherche de compte + repli par défaut |
    | `plugin-sdk/account-helpers` | Helpers étroits de liste de comptes/action sur compte |
    | `plugin-sdk/channel-pairing` | `createChannelPairingController` |
    | `plugin-sdk/channel-reply-pipeline` | `createChannelReplyPipeline` |
    | `plugin-sdk/channel-config-helpers` | `createHybridChannelConfigAdapter` |
    | `plugin-sdk/channel-config-schema` | Types du schéma de config de canal |
    | `plugin-sdk/telegram-command-config` | Helpers de normalisation/validation des commandes personnalisées Telegram avec repli de contrat fourni |
    | `plugin-sdk/channel-policy` | `resolveChannelGroupRequireMention` |
    | `plugin-sdk/channel-lifecycle` | `createAccountStatusSink` |
    | `plugin-sdk/inbound-envelope` | Helpers partagés de route entrante + construction d'enveloppe |
    | `plugin-sdk/inbound-reply-dispatch` | Helpers partagés d'enregistrement et d'envoi entrants |
    | `plugin-sdk/messaging-targets` | Helpers d'analyse/correspondance des cibles |
    | `plugin-sdk/outbound-media` | Helpers partagés de chargement de médias sortants |
    | `plugin-sdk/outbound-runtime` | Helpers de délégation d'identité/d'envoi sortants |
    | `plugin-sdk/thread-bindings-runtime` | Helpers de cycle de vie et d'adaptation des liaisons de fil |
    | `plugin-sdk/agent-media-payload` | Constructeur hérité de payload média d'agent |
    | `plugin-sdk/conversation-runtime` | Helpers de conversation/liaison de fil, appairage et liaison configurée |
    | `plugin-sdk/runtime-config-snapshot` | Helper d'instantané de config d'exécution |
    | `plugin-sdk/runtime-group-policy` | Helpers de résolution de politique de groupe à l'exécution |
    | `plugin-sdk/channel-status` | Helpers partagés d'instantané/résumé de statut de canal |
    | `plugin-sdk/channel-config-primitives` | Primitifs étroits de schéma de config de canal |
    | `plugin-sdk/channel-config-writes` | Helpers d'autorisation d'écriture de config de canal |
    | `plugin-sdk/channel-plugin-common` | Exportations de prélude partagées pour plugins de canal |
    | `plugin-sdk/allowlist-config-edit` | Helpers de lecture/édition de config de liste d'autorisation |
    | `plugin-sdk/group-access` | Helpers partagés de décision d'accès aux groupes |
    | `plugin-sdk/direct-dm` | Helpers partagés d'authentification/de garde des messages privés directs |
    | `plugin-sdk/interactive-runtime` | Helpers de normalisation/réduction de payload de réponse interactive |
    | `plugin-sdk/channel-inbound` | Helpers de temporisation des entrants, correspondance de mention, politique de mention et d'enveloppe |
    | `plugin-sdk/channel-send-result` | Types de résultat de réponse |
    | `plugin-sdk/channel-actions` | `createMessageToolButtonsSchema`, `createMessageToolCardSchema` |
    | `plugin-sdk/channel-targets` | Helpers d'analyse/correspondance des cibles |
    | `plugin-sdk/channel-contract` | Types de contrat de canal |
    | `plugin-sdk/channel-feedback` | Câblage des retours/réactions |
    | `plugin-sdk/channel-secret-runtime` | Helpers étroits de contrat de secret tels que `collectSimpleChannelFieldAssignments`, `getChannelSurface`, `pushAssignment`, et types de cible de secret |
  </Accordion>

  <Accordion title="Sous-chemins de fournisseur">
    | Sous-chemin | Exportations clés |
    | --- | --- |
    | `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry` |
    | `plugin-sdk/provider-setup` | Helpers de configuration sélectionnés pour fournisseurs locaux/autohébergés |
    | `plugin-sdk/self-hosted-provider-setup` | Helpers ciblés de configuration de fournisseur autohébergé compatible OpenAI |
    | `plugin-sdk/cli-backend` | Valeurs par défaut du backend CLI + constantes de watchdog |
    | `plugin-sdk/provider-auth-runtime` | Helpers d'exécution de résolution de clé API pour plugins de fournisseur |
    | `plugin-sdk/provider-auth-api-key` | Helpers d'intégration/écriture de profil de clé API tels que `upsertApiKeyProfile` |
    | `plugin-sdk/provider-auth-result` | Constructeur standard de résultat d'authentification OAuth |
    | `plugin-sdk/provider-auth-login` | Helpers partagés de connexion interactive pour plugins de fournisseur |
    | `plugin-sdk/provider-env-vars` | Helpers de recherche de variables d'environnement d'authentification fournisseur |
    | `plugin-sdk/provider-auth` | `createProviderApiKeyAuthMethod`, `ensureApiKeyFromOptionEnvOrPrompt`, `upsertAuthProfile`, `upsertApiKeyProfile`, `writeOAuthCredentials` |
    | `plugin-sdk/provider-model-shared` | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, constructeurs partagés de politique de relecture, helpers de point de terminaison fournisseur, et helpers de normalisation d'identifiant de modèle tels que `normalizeNativeXaiModelId` |
    | `plugin-sdk/provider-catalog-shared` | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
    | `plugin-sdk/provider-http` | Helpers génériques de capacités HTTP/point de terminaison de fournisseur |
    | `plugin-sdk/provider-web-fetch-contract` | Helpers étroits de contrat de config/sélection web-fetch tels que `enablePluginInConfig` et `WebFetchProviderPlugin` |
    | `plugin-sdk/provider-web-fetch` | Helpers d'enregistrement/cache de fournisseur web-fetch |
    | `plugin-sdk/provider-web-search-contract` | Helpers étroits de contrat de config/d'identifiants web-search tels que `enablePluginInConfig`, `resolveProviderWebSearchPluginConfig`, et setters/getters d'identifiants à portée limitée |
    | `plugin-sdk/provider-web-search` | Helpers d'enregistrement/cache/exécution de fournisseur web-search |
    | `plugin-sdk/provider-tools` | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, nettoyage du schéma Gemini + diagnostics, et helpers de compatibilité xAI tels que `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
    | `plugin-sdk/provider-usage` | `fetchClaudeUsage` et similaires |
    | `plugin-sdk/provider-stream` | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, types de wrappers de flux, et helpers de wrappers partagés Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot |
    | `plugin-sdk/provider-onboard` | Helpers de patch de config d'intégration |
    | `plugin-sdk/global-singleton` | Helpers de singleton/map/cache locaux au processus |
  </Accordion>

  <Accordion title="Sous-chemins d'authentification et de sécurité">
    | Sous-chemin | Exportations clés |
    | --- | --- |
    | `plugin-sdk/command-auth` | `resolveControlCommandGate`, helpers de registre de commandes, helpers d'autorisation de l'expéditeur |
    | `plugin-sdk/approval-auth-runtime` | Résolution d'approbateur et helpers d'authentification d'action dans la même discussion |
    | `plugin-sdk/approval-client-runtime` | Helpers de profil/filtre d'approbation d'exécution native |
    | `plugin-sdk/approval-delivery-runtime` | Adaptateurs de capacité/livraison d'approbation native |
    | `plugin-sdk/approval-gateway-runtime` | Helper partagé de résolution de passerelle d'approbation |
    | `plugin-sdk/approval-handler-adapter-runtime` | Helpers légers de chargement d'adaptateur d'approbation native pour points d'entrée de canaux à chaud |
    | `plugin-sdk/approval-handler-runtime` | Helpers d'exécution plus larges pour gestionnaire d'approbation ; préférez les points d'entrée adaptateur/passerelle plus étroits lorsqu'ils suffisent |
    | `plugin-sdk/approval-native-runtime` | Helpers de cible d'approbation native + liaison de compte |
    | `plugin-sdk/approval-reply-runtime` | Helpers de payload de réponse d'approbation exec/plugin |
    | `plugin-sdk/command-auth-native` | Authentification de commande native + helpers natifs de cible de session |
    | `plugin-sdk/command-detection` | Helpers partagés de détection de commandes |
    | `plugin-sdk/command-surface` | Helpers de normalisation du corps de commande et de surface de commande |
    | `plugin-sdk/allow-from` | `formatAllowFromLowercase` |
    | `plugin-sdk/channel-secret-runtime` | Helpers étroits de collecte de contrat de secret pour surfaces de secret de canal/plugin |
    | `plugin-sdk/secret-ref-runtime` | Helpers étroits `coerceSecretRef` et de typage SecretRef pour l'analyse de contrat de secret/config |
    | `plugin-sdk/security-runtime` | Helpers partagés de confiance, filtrage des messages privés, contenu externe et collecte de secrets |
    | `plugin-sdk/ssrf-policy` | Helpers de liste d'autorisation d'hôtes et de politique SSRF de réseau privé |
    | `plugin-sdk/ssrf-runtime` | Helpers de répartiteur épinglé, fetch protégé contre SSRF et politique SSRF |
    | `plugin-sdk/secret-input` | Helpers d'analyse des entrées secrètes |
    | `plugin-sdk/webhook-ingress` | Helpers de requête/cible webhook |
    | `plugin-sdk/webhook-request-guards` | Helpers de taille de corps/délai d'attente de requête |
  </Accordion>

  <Accordion title="Sous-chemins d'exécution et de stockage">
    | Sous-chemin | Exportations clés |
    | --- | --- |
    | `plugin-sdk/runtime` | Helpers larges d'exécution/journalisation/sauvegarde/installation de plugin |
    | `plugin-sdk/runtime-env` | Helpers étroits d'environnement d'exécution, logger, délai d'attente, retry et backoff |
    | `plugin-sdk/channel-runtime-context` | Helpers génériques d'enregistrement et de recherche de contexte d'exécution de canal |
    | `plugin-sdk/runtime-store` | `createPluginRuntimeStore` |
    | `plugin-sdk/plugin-runtime` | Helpers partagés de commande/hook/http/interaction de plugin |
    | `plugin-sdk/hook-runtime` | Helpers partagés de pipeline de hooks webhook/internes |
    | `plugin-sdk/lazy-runtime` | Helpers de liaison/import d'exécution paresseuse tels que `createLazyRuntimeModule`, `createLazyRuntimeMethod` et `createLazyRuntimeSurface` |
    | `plugin-sdk/process-runtime` | Helpers d'exécution de processus |
    | `plugin-sdk/cli-runtime` | Helpers de formatage CLI, attente et version |
    | `plugin-sdk/gateway-runtime` | Helpers de client Gateway et de patch de statut de canal |
    | `plugin-sdk/config-runtime` | Helpers de chargement/écriture de config |
    | `plugin-sdk/telegram-command-config` | Normalisation du nom/de la description de commande Telegram et vérifications de doublons/conflits, même lorsque la surface de contrat Telegram fournie n'est pas disponible |
    | `plugin-sdk/approval-runtime` | Helpers d'approbation exec/plugin, constructeurs de capacité d'approbation, helpers d'authentification/de profil, helpers natifs de routage/exécution |
    | `plugin-sdk/reply-runtime` | Helpers partagés d'exécution entrante/de réponse, segmentation, envoi, heartbeat, planificateur de réponse |
    | `plugin-sdk/reply-dispatch-runtime` | Helpers étroits d'envoi/finalisation de réponse |
    | `plugin-sdk/reply-history` | Helpers partagés d'historique de réponse sur courte fenêtre tels que `buildHistoryContext`, `recordPendingHistoryEntry` et `clearHistoryEntriesIfEnabled` |
    | `plugin-sdk/reply-reference` | `createReplyReferencePlanner` |
    | `plugin-sdk/reply-chunking` | Helpers étroits de segmentation de texte/Markdown |
    | `plugin-sdk/session-store-runtime` | Helpers de chemin du magasin de session + `updated-at` |
    | `plugin-sdk/state-paths` | Helpers de chemin de répertoire d'état/OAuth |
    | `plugin-sdk/routing` | Helpers de route/clé de session/liaison de compte tels que `resolveAgentRoute`, `buildAgentSessionKey` et `resolveDefaultAgentBoundAccountId` |
    | `plugin-sdk/status-helpers` | Helpers partagés de résumé de statut de canal/compte, valeurs par défaut d'état d'exécution et helpers de métadonnées de problème |
    | `plugin-sdk/target-resolver-runtime` | Helpers partagés de résolution de cible |
    | `plugin-sdk/string-normalization-runtime` | Helpers de normalisation de slug/chaîne |
    | `plugin-sdk/request-url` | Extraire des URL de chaîne à partir d'entrées de type fetch/request |
    | `plugin-sdk/run-command` | Exécuteur de commandes temporisé avec résultats stdout/stderr normalisés |
    | `plugin-sdk/param-readers` | Lecteurs communs de paramètres d'outil/CLI |
    | `plugin-sdk/tool-send` | Extraire les champs cibles d'envoi canoniques des arguments d'outil |
    | `plugin-sdk/temp-path` | Helpers partagés de chemin de téléchargement temporaire |
    | `plugin-sdk/logging-core` | Helpers de logger de sous-système et de masquage |
    | `plugin-sdk/markdown-table-runtime` | Helpers de mode de tableau Markdown |
    | `plugin-sdk/json-store` | Petits helpers de lecture/écriture d'état JSON |
    | `plugin-sdk/file-lock` | Helpers de verrouillage de fichier réentrant |
    | `plugin-sdk/persistent-dedupe` | Helpers de cache de déduplication persisté sur disque |
    | `plugin-sdk/acp-runtime` | Helpers ACP d'exécution/session et d'envoi de réponse |
    | `plugin-sdk/agent-config-primitives` | Primitifs étroits de schéma de config d'exécution d'agent |
    | `plugin-sdk/boolean-param` | Lecteur tolérant de paramètre booléen |
    | `plugin-sdk/dangerous-name-runtime` | Helpers de résolution de correspondance de noms dangereux |
    | `plugin-sdk/device-bootstrap` | Helpers de bootstrap d'appareil et de jeton d'appairage |
    | `plugin-sdk/extension-shared` | Primitifs helpers partagés pour canal passif, statut et proxy ambiant |
    | `plugin-sdk/models-provider-runtime` | Helpers de réponse de commande `/models`/fournisseur |
    | `plugin-sdk/skill-commands-runtime` | Helpers de liste de commandes Skills |
    | `plugin-sdk/native-command-registry` | Helpers natifs de registre/construction/sérialisation de commandes |
    | `plugin-sdk/provider-zai-endpoint` | Helpers de détection de point de terminaison Z.AI |
    | `plugin-sdk/infra-runtime` | Helpers d'événement système/heartbeat |
    | `plugin-sdk/collection-runtime` | Petits helpers de cache borné |
    | `plugin-sdk/diagnostic-runtime` | Helpers de drapeau et d'événement de diagnostic |
    | `plugin-sdk/error-runtime` | Helpers de graphe d'erreurs, formatage, classification d'erreurs partagée, `isApprovalNotFoundError` |
    | `plugin-sdk/fetch-runtime` | Helpers de fetch encapsulé, proxy et recherche épinglée |
    | `plugin-sdk/host-runtime` | Helpers de normalisation de nom d'hôte et d'hôte SCP |
    | `plugin-sdk/retry-runtime` | Helpers de configuration et d'exécuteur de retry |
    | `plugin-sdk/agent-runtime` | Helpers de répertoire/identité/espace de travail d'agent |
    | `plugin-sdk/directory-runtime` | Requête/déduplication de répertoire adossée à la config |
    | `plugin-sdk/keyed-async-queue` | `KeyedAsyncQueue` |
  </Accordion>

  <Accordion title="Sous-chemins de capacité et de test">
    | Sous-chemin | Exportations clés |
    | --- | --- |
    | `plugin-sdk/media-runtime` | Helpers partagés de récupération/transformation/stockage de médias ainsi que constructeurs de payload média |
    | `plugin-sdk/media-generation-runtime` | Helpers partagés de bascule de génération de médias, sélection de candidats et messages de modèle manquant |
    | `plugin-sdk/media-understanding` | Types de fournisseur de compréhension média ainsi qu'exportations d'helpers d'image/audio côté fournisseur |
    | `plugin-sdk/text-runtime` | Helpers partagés de texte/Markdown/journalisation tels que suppression de texte visible par l'assistant, helpers de rendu/segmentation/tableau Markdown, helpers de masquage, helpers de balise de directive et utilitaires de texte sûr |
    | `plugin-sdk/text-chunking` | Helper de segmentation de texte sortant |
    | `plugin-sdk/speech` | Types de fournisseur vocal ainsi qu'helpers côté fournisseur pour directives, registre et validation |
    | `plugin-sdk/speech-core` | Types partagés de fournisseur vocal, helpers de registre, de directive et de normalisation |
    | `plugin-sdk/realtime-transcription` | Types de fournisseur de transcription en temps réel et helpers de registre |
    | `plugin-sdk/realtime-voice` | Types de fournisseur vocal en temps réel et helpers de registre |
    | `plugin-sdk/image-generation` | Types de fournisseur de génération d'image |
    | `plugin-sdk/image-generation-core` | Types partagés de génération d'image, helpers de bascule, d'authentification et de registre |
    | `plugin-sdk/music-generation` | Types de fournisseur/requête/résultat de génération musicale |
    | `plugin-sdk/music-generation-core` | Types partagés de génération musicale, helpers de bascule, recherche de fournisseur et analyse de référence de modèle |
    | `plugin-sdk/video-generation` | Types de fournisseur/requête/résultat de génération vidéo |
    | `plugin-sdk/video-generation-core` | Types partagés de génération vidéo, helpers de bascule, recherche de fournisseur et analyse de référence de modèle |
    | `plugin-sdk/webhook-targets` | Helpers de registre de cibles webhook et d'installation de route |
    | `plugin-sdk/webhook-path` | Helpers de normalisation de chemin webhook |
    | `plugin-sdk/web-media` | Helpers partagés de chargement de médias distants/locaux |
    | `plugin-sdk/zod` | `zod` réexporté pour les consommateurs du Plugin SDK |
    | `plugin-sdk/testing` | `installCommonResolveTargetErrorCases`, `shouldAckReaction` |
  </Accordion>

  <Accordion title="Sous-chemins mémoire">
    | Sous-chemin | Exportations clés |
    | --- | --- |
    | `plugin-sdk/memory-core` | Surface helper `memory-core` fournie pour gestionnaire/config/fichier/helpers CLI |
    | `plugin-sdk/memory-core-engine-runtime` | Façade d'exécution d'index/recherche mémoire |
    | `plugin-sdk/memory-core-host-engine-foundation` | Exportations du moteur fondation d'hôte mémoire |
    | `plugin-sdk/memory-core-host-engine-embeddings` | Exportations du moteur d'embeddings d'hôte mémoire |
    | `plugin-sdk/memory-core-host-engine-qmd` | Exportations du moteur QMD d'hôte mémoire |
    | `plugin-sdk/memory-core-host-engine-storage` | Exportations du moteur de stockage d'hôte mémoire |
    | `plugin-sdk/memory-core-host-multimodal` | Helpers multimodaux d'hôte mémoire |
    | `plugin-sdk/memory-core-host-query` | Helpers de requête d'hôte mémoire |
    | `plugin-sdk/memory-core-host-secret` | Helpers de secret d'hôte mémoire |
    | `plugin-sdk/memory-core-host-events` | Helpers de journal d'événements d'hôte mémoire |
    | `plugin-sdk/memory-core-host-status` | Helpers de statut d'hôte mémoire |
    | `plugin-sdk/memory-core-host-runtime-cli` | Helpers d'exécution CLI d'hôte mémoire |
    | `plugin-sdk/memory-core-host-runtime-core` | Helpers d'exécution cœur d'hôte mémoire |
    | `plugin-sdk/memory-core-host-runtime-files` | Helpers de fichier/exécution d'hôte mémoire |
    | `plugin-sdk/memory-host-core` | Alias neutre vis-à-vis du fournisseur pour les helpers d'exécution cœur d'hôte mémoire |
    | `plugin-sdk/memory-host-events` | Alias neutre vis-à-vis du fournisseur pour les helpers de journal d'événements d'hôte mémoire |
    | `plugin-sdk/memory-host-files` | Alias neutre vis-à-vis du fournisseur pour les helpers de fichier/exécution d'hôte mémoire |
    | `plugin-sdk/memory-host-markdown` | Helpers partagés de Markdown géré pour plugins proches de la mémoire |
    | `plugin-sdk/memory-host-search` | Façade d'exécution de mémoire active pour accès au gestionnaire de recherche |
    | `plugin-sdk/memory-host-status` | Alias neutre vis-à-vis du fournisseur pour les helpers de statut d'hôte mémoire |
    | `plugin-sdk/memory-lancedb` | Surface helper `memory-lancedb` fournie |
  </Accordion>

  <Accordion title="Sous-chemins helpers fournis réservés">
    | Famille | Sous-chemins actuels | Utilisation prévue |
    | --- | --- | --- |
    | Browser | `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support` | Helpers de prise en charge du plugin Browser fourni (`browser-support` reste le barrel de compatibilité) |
    | Matrix | `plugin-sdk/matrix`, `plugin-sdk/matrix-helper`, `plugin-sdk/matrix-runtime-heavy`, `plugin-sdk/matrix-runtime-shared`, `plugin-sdk/matrix-runtime-surface`, `plugin-sdk/matrix-surface`, `plugin-sdk/matrix-thread-bindings` | Surface helper/exécution Matrix fournie |
    | Line | `plugin-sdk/line`, `plugin-sdk/line-core`, `plugin-sdk/line-runtime`, `plugin-sdk/line-surface` | Surface helper/exécution LINE fournie |
    | IRC | `plugin-sdk/irc`, `plugin-sdk/irc-surface` | Surface helper IRC fournie |
    | Helpers spécifiques à un canal | `plugin-sdk/googlechat`, `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles`, `plugin-sdk/bluebubbles-policy`, `plugin-sdk/mattermost`, `plugin-sdk/mattermost-policy`, `plugin-sdk/feishu-conversation`, `plugin-sdk/msteams`, `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`, `plugin-sdk/twitch` | Points d'entrée de compatibilité/helper de canaux fournis |
    | Helpers spécifiques à l'authentification/au plugin | `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`, `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`, `plugin-sdk/thread-ownership`, `plugin-sdk/voice-call` | Points d'entrée helpers de fonctionnalité/plugin fournis ; `plugin-sdk/github-copilot-token` exporte actuellement `DEFAULT_COPILOT_API_BASE_URL`, `deriveCopilotApiBaseUrlFromToken` et `resolveCopilotApiToken` |
  </Accordion>
</AccordionGroup>

## API d'enregistrement

Le callback `register(api)` reçoit un objet `OpenClawPluginApi` avec ces
méthodes :

### Enregistrement de capacité

| Méthode                                          | Ce qu'elle enregistre            |
| ------------------------------------------------ | -------------------------------- |
| `api.registerProvider(...)`                      | Inférence de texte (LLM)         |
| `api.registerCliBackend(...)`                    | Backend CLI local d'inférence    |
| `api.registerChannel(...)`                       | Canal de messagerie              |
| `api.registerSpeechProvider(...)`                | Synthèse text-to-speech / STT    |
| `api.registerRealtimeTranscriptionProvider(...)` | Transcription temps réel en flux |
| `api.registerRealtimeVoiceProvider(...)`         | Sessions vocales duplex en temps réel |
| `api.registerMediaUnderstandingProvider(...)`    | Analyse d'image/audio/vidéo      |
| `api.registerImageGenerationProvider(...)`       | Génération d'image               |
| `api.registerMusicGenerationProvider(...)`       | Génération musicale              |
| `api.registerVideoGenerationProvider(...)`       | Génération vidéo                 |
| `api.registerWebFetchProvider(...)`              | Fournisseur de récupération / scraping web |
| `api.registerWebSearchProvider(...)`             | Recherche web                    |

### Outils et commandes

| Méthode                          | Ce qu'elle enregistre                         |
| ------------------------------- | --------------------------------------------- |
| `api.registerTool(tool, opts?)` | Outil d'agent (obligatoire ou `{ optional: true }`) |
| `api.registerCommand(def)`      | Commande personnalisée (contourne le LLM)     |

### Infrastructure

| Méthode                                         | Ce qu'elle enregistre                 |
| ---------------------------------------------- | ------------------------------------- |
| `api.registerHook(events, handler, opts?)`     | Hook d'événement                      |
| `api.registerHttpRoute(params)`                | Point de terminaison HTTP Gateway     |
| `api.registerGatewayMethod(name, handler)`     | Méthode RPC Gateway                   |
| `api.registerCli(registrar, opts?)`            | Sous-commande CLI                     |
| `api.registerService(service)`                 | Service en arrière-plan               |
| `api.registerInteractiveHandler(registration)` | Gestionnaire interactif               |
| `api.registerMemoryPromptSupplement(builder)`  | Section de prompt additive adjacente à la mémoire |
| `api.registerMemoryCorpusSupplement(adapter)`  | Corpus additif de recherche/lecture mémoire |

Les espaces de noms d'administration du cœur réservés (`config.*`, `exec.approvals.*`, `wizard.*`,
`update.*`) restent toujours `operator.admin`, même si un plugin tente d'attribuer une
portée de méthode Gateway plus étroite. Préférez des préfixes spécifiques au plugin pour
les méthodes appartenant au plugin.

### Métadonnées d'enregistrement CLI

`api.registerCli(registrar, opts?)` accepte deux types de métadonnées de premier niveau :

- `commands` : racines de commandes explicites appartenant au registrar
- `descriptors` : descripteurs de commande à l'analyse utilisés pour l'aide de la CLI racine,
  le routage et l'enregistrement paresseux de la CLI de plugin

Si vous voulez qu'une commande de plugin reste chargée paresseusement dans le chemin CLI racine normal,
fournissez des `descriptors` qui couvrent chaque racine de commande de premier niveau exposée par ce
registrar.

```typescript
api.registerCli(
  async ({ program }) => {
    const { registerMatrixCli } = await import("./src/cli.js");
    registerMatrixCli({ program });
  },
  {
    descriptors: [
      {
        name: "matrix",
        description: "Gérer les comptes Matrix, la vérification, les appareils et l'état du profil",
        hasSubcommands: true,
      },
    ],
  },
);
```

Utilisez `commands` seul uniquement lorsque vous n'avez pas besoin de l'enregistrement paresseux de la CLI racine.
Ce chemin de compatibilité eager reste pris en charge, mais il n'installe pas
d'espaces réservés adossés à des descripteurs pour le chargement paresseux à l'analyse.

### Enregistrement de backend CLI

`api.registerCliBackend(...)` permet à un plugin de posséder la config par défaut d'un
backend CLI IA local tel que `codex-cli`.

- Le `id` du backend devient le préfixe du fournisseur dans des références de modèle comme `codex-cli/gpt-5`.
- La `config` du backend utilise la même forme que `agents.defaults.cliBackends.<id>`.
- La config utilisateur a toujours priorité. OpenClaw fusionne `agents.defaults.cliBackends.<id>` par-dessus la
  valeur par défaut du plugin avant d'exécuter la CLI.
- Utilisez `normalizeConfig` lorsqu'un backend a besoin de réécritures de compatibilité après fusion
  (par exemple pour normaliser d'anciennes formes de drapeaux).

### Emplacements exclusifs

| Méthode                                     | Ce qu'elle enregistre                                                                                                                                        |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `api.registerContextEngine(id, factory)`   | Moteur de contexte (un seul actif à la fois). Le callback `assemble()` reçoit `availableTools` et `citationsMode` afin que le moteur puisse adapter les ajouts au prompt. |
| `api.registerMemoryCapability(capability)` | Capacité mémoire unifiée                                                                                                                                        |
| `api.registerMemoryPromptSection(builder)` | Constructeur de section de prompt mémoire                                                                                                                       |
| `api.registerMemoryFlushPlan(resolver)`    | Résolveur de plan de vidage mémoire                                                                                                                             |
| `api.registerMemoryRuntime(runtime)`       | Adaptateur d'exécution mémoire                                                                                                                                  |

### Adaptateurs d'embedding mémoire

| Méthode                                         | Ce qu'elle enregistre                         |
| ---------------------------------------------- | --------------------------------------------- |
| `api.registerMemoryEmbeddingProvider(adapter)` | Adaptateur d'embedding mémoire pour le plugin actif |

- `registerMemoryCapability` est l'API exclusive privilégiée pour les plugins mémoire.
- `registerMemoryCapability` peut aussi exposer `publicArtifacts.listArtifacts(...)`
  afin que des plugins compagnons puissent consommer des artefacts mémoire exportés via
  `openclaw/plugin-sdk/memory-host-core` au lieu d'accéder à la disposition privée d'un
  plugin mémoire spécifique.
- `registerMemoryPromptSection`, `registerMemoryFlushPlan` et
  `registerMemoryRuntime` sont des API exclusives de plugin mémoire compatibles avec l'héritage.
- `registerMemoryEmbeddingProvider` permet au plugin mémoire actif d'enregistrer un
  ou plusieurs identifiants d'adaptateur d'embedding (par exemple `openai`, `gemini` ou un identifiant personnalisé défini par plugin).
- La config utilisateur telle que `agents.defaults.memorySearch.provider` et
  `agents.defaults.memorySearch.fallback` se résout à partir de ces identifiants d'adaptateur enregistrés.

### Événements et cycle de vie

| Méthode                                       | Ce qu'elle fait              |
| -------------------------------------------- | ---------------------------- |
| `api.on(hookName, handler, opts?)`           | Hook de cycle de vie typé    |
| `api.onConversationBindingResolved(handler)` | Callback de liaison de conversation |

### Sémantique de décision des hooks

- `before_tool_call` : renvoyer `{ block: true }` est terminal. Dès qu'un gestionnaire le définit, les gestionnaires de priorité inférieure sont ignorés.
- `before_tool_call` : renvoyer `{ block: false }` est traité comme aucune décision (comme si `block` était omis), et non comme une surcharge.
- `before_install` : renvoyer `{ block: true }` est terminal. Dès qu'un gestionnaire le définit, les gestionnaires de priorité inférieure sont ignorés.
- `before_install` : renvoyer `{ block: false }` est traité comme aucune décision (comme si `block` était omis), et non comme une surcharge.
- `reply_dispatch` : renvoyer `{ handled: true, ... }` est terminal. Dès qu'un gestionnaire revendique l'envoi, les gestionnaires de priorité inférieure et le chemin d'envoi du modèle par défaut sont ignorés.
- `message_sending` : renvoyer `{ cancel: true }` est terminal. Dès qu'un gestionnaire le définit, les gestionnaires de priorité inférieure sont ignorés.
- `message_sending` : renvoyer `{ cancel: false }` est traité comme aucune décision (comme si `cancel` était omis), et non comme une surcharge.

### Champs de l'objet API

| Champ                    | Type                      | Description                                                                                 |
| ------------------------ | ------------------------- | ------------------------------------------------------------------------------------------- |
| `api.id`                 | `string`                  | Identifiant du plugin                                                                       |
| `api.name`               | `string`                  | Nom d'affichage                                                                             |
| `api.version`            | `string?`                 | Version du plugin (facultatif)                                                              |
| `api.description`        | `string?`                 | Description du plugin (facultatif)                                                          |
| `api.source`             | `string`                  | Chemin source du plugin                                                                     |
| `api.rootDir`            | `string?`                 | Répertoire racine du plugin (facultatif)                                                    |
| `api.config`             | `OpenClawConfig`          | Instantané de config actuel (instantané d'exécution en mémoire actif lorsqu'il est disponible) |
| `api.pluginConfig`       | `Record<string, unknown>` | Config spécifique au plugin provenant de `plugins.entries.<id>.config`                      |
| `api.runtime`            | `PluginRuntime`           | [Helpers d'exécution](/fr/plugins/sdk-runtime)                                                 |
| `api.logger`             | `PluginLogger`            | Logger à portée limitée (`debug`, `info`, `warn`, `error`)                                  |
| `api.registrationMode`   | `PluginRegistrationMode`  | Mode de chargement actuel ; `"setup-runtime"` est la fenêtre légère de démarrage/configuration avant l'entrée complète |
| `api.resolvePath(input)` | `(string) => string`      | Résoudre un chemin relatif à la racine du plugin                                            |

## Convention de module interne

Dans votre plugin, utilisez des fichiers barrel locaux pour les imports internes :

```
my-plugin/
  api.ts            # Exportations publiques pour les consommateurs externes
  runtime-api.ts    # Exportations d'exécution internes uniquement
  index.ts          # Point d'entrée du plugin
  setup-entry.ts    # Entrée légère réservée à la configuration (facultative)
```

<Warning>
  N'importez jamais votre propre plugin via `openclaw/plugin-sdk/<your-plugin>`
  depuis le code de production. Faites passer les imports internes par `./api.ts` ou
  `./runtime-api.ts`. Le chemin SDK n'est que le contrat externe.
</Warning>

Les surfaces publiques de plugin fourni chargées via façade (`api.ts`, `runtime-api.ts`,
`index.ts`, `setup-entry.ts` et fichiers d'entrée publics similaires) préfèrent désormais l'instantané
de config d'exécution actif lorsque OpenClaw est déjà en cours d'exécution. Si aucun instantané
d'exécution n'existe encore, elles reviennent à la config résolue sur le disque.

Les plugins de fournisseur peuvent également exposer un barrel de contrat local au plugin et étroit lorsqu'un
helper est volontairement spécifique au fournisseur et n'a pas encore sa place dans un sous-chemin SDK générique.
Exemple fourni actuel : le fournisseur Anthropic conserve ses helpers de flux Claude
dans son propre point d'entrée public `api.ts` / `contract-api.ts` au lieu
de promouvoir la logique d'en-tête bêta Anthropic et `service_tier` dans un contrat
générique `plugin-sdk/*`.

Autres exemples fournis actuels :

- `@openclaw/openai-provider` : `api.ts` exporte des constructeurs de fournisseur,
  des helpers de modèle par défaut et des constructeurs de fournisseur temps réel
- `@openclaw/openrouter-provider` : `api.ts` exporte le constructeur de fournisseur ainsi que
  des helpers d'intégration/de config

<Warning>
  Le code de production des extensions doit également éviter les imports `openclaw/plugin-sdk/<other-plugin>`.
  Si un helper est réellement partagé, faites-le évoluer vers un sous-chemin SDK neutre
  tel que `openclaw/plugin-sdk/speech`, `.../provider-model-shared` ou une autre
  surface orientée capacité au lieu de coupler deux plugins entre eux.
</Warning>

## Liens connexes

- [Entry Points](/fr/plugins/sdk-entrypoints) — options de `definePluginEntry` et `defineChannelPluginEntry`
- [Runtime Helpers](/fr/plugins/sdk-runtime) — référence complète de l'espace de noms `api.runtime`
- [Setup and Config](/fr/plugins/sdk-setup) — packaging, manifestes, schémas de config
- [Testing](/fr/plugins/sdk-testing) — utilitaires de test et règles de lint
- [SDK Migration](/fr/plugins/sdk-migration) — migration depuis des surfaces obsolètes
- [Plugin Internals](/fr/plugins/architecture) — architecture approfondie et modèle de capacité
