---
read_when:
    - Vous devez savoir depuis quel sous-chemin du SDK importer
    - Vous voulez une référence pour toutes les méthodes d’enregistrement sur OpenClawPluginApi
    - Vous recherchez un export SDK spécifique
sidebarTitle: SDK Overview
summary: Carte des imports, référence de l’API d’enregistrement et architecture du SDK
title: Vue d’ensemble du Plugin SDK
x-i18n:
    generated_at: "2026-04-07T06:54:06Z"
    model: gpt-5.4
    provider: openai
    source_hash: be651251bc1d8fc6548cb9efd4b7940be093f7f7cdd70d94dc31d5a1ebde80f3
    source_path: plugins/sdk-overview.md
    workflow: 15
---

# Vue d’ensemble du Plugin SDK

Le Plugin SDK est le contrat typé entre les plugins et le cœur. Cette page est la
référence pour **quoi importer** et **ce que vous pouvez enregistrer**.

<Tip>
  **Vous cherchez un guide pratique ?**
  - Premier plugin ? Commencez par [Premiers pas](/fr/plugins/building-plugins)
  - Plugin de canal ? Voir [Plugins de canal](/fr/plugins/sdk-channel-plugins)
  - Plugin provider ? Voir [Plugins provider](/fr/plugins/sdk-provider-plugins)
</Tip>

## Convention d’import

Importez toujours depuis un sous-chemin spécifique :

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
```

Chaque sous-chemin est un petit module autonome. Cela permet de garder un démarrage rapide et
d’éviter les problèmes de dépendances circulaires. Pour les helpers d’entrée/de build spécifiques aux canaux,
préférez `openclaw/plugin-sdk/channel-core` ; gardez `openclaw/plugin-sdk/core` pour
la surface ombrelle plus large et les helpers partagés tels que
`buildChannelConfigSchema`.

N’ajoutez pas et ne dépendez pas de points d’accès pratiques nommés d’après des providers, comme
`openclaw/plugin-sdk/slack`, `openclaw/plugin-sdk/discord`,
`openclaw/plugin-sdk/signal`, `openclaw/plugin-sdk/whatsapp`, ni de
points d’accès de helpers liés à une marque de canal. Les plugins intégrés doivent composer des
sous-chemins SDK génériques dans leurs propres barils `api.ts` ou `runtime-api.ts`, et le cœur
doit soit utiliser ces barils locaux au plugin, soit ajouter un contrat SDK générique et étroit lorsque le besoin est réellement inter-canal.

La carte d’exports générée contient encore un petit ensemble de
points d’accès de helpers de plugins intégrés tels que `plugin-sdk/feishu`, `plugin-sdk/feishu-setup`,
`plugin-sdk/zalo`, `plugin-sdk/zalo-setup` et `plugin-sdk/matrix*`. Ces
sous-chemins existent uniquement pour la maintenance et la compatibilité des plugins intégrés ; ils sont
volontairement omis du tableau commun ci-dessous et ne constituent pas le chemin d’import recommandé pour les nouveaux plugins tiers.

## Référence des sous-chemins

Les sous-chemins les plus couramment utilisés, regroupés par usage. La liste complète générée de
plus de 200 sous-chemins se trouve dans `scripts/lib/plugin-sdk-entrypoints.json`.

Les sous-chemins réservés de helpers intégrés apparaissent toujours dans cette liste générée.
Considérez-les comme des surfaces de détail d’implémentation/de compatibilité, sauf si une page de documentation
en promeut explicitement un comme public.

### Entrée de plugin

| Sous-chemin                | Exports clés                                                                                                                           |
| -------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/plugin-entry`  | `definePluginEntry`                                                                                                                    |
| `plugin-sdk/core`          | `defineChannelPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase`, `defineSetupPluginEntry`, `buildChannelConfigSchema` |
| `plugin-sdk/config-schema` | `OpenClawSchema`                                                                                                                       |
| `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry`                                                                                                     |

<AccordionGroup>
  <Accordion title="Sous-chemins de canal">
    | Sous-chemin | Exports clés |
    | --- | --- |
    | `plugin-sdk/channel-core` | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
    | `plugin-sdk/config-schema` | Export du schéma Zod racine `openclaw.json` (`OpenClawSchema`) |
    | `plugin-sdk/channel-setup` | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, ainsi que `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
    | `plugin-sdk/setup` | Helpers partagés pour l’assistant de configuration, prompts de liste d’autorisation, constructeurs d’état de configuration |
    | `plugin-sdk/setup-runtime` | `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
    | `plugin-sdk/setup-adapter-runtime` | `createEnvPatchedAccountSetupAdapter` |
    | `plugin-sdk/setup-tools` | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
    | `plugin-sdk/account-core` | Helpers de barrière d’action/de configuration multi-compte, helpers de repli du compte par défaut |
    | `plugin-sdk/account-id` | `DEFAULT_ACCOUNT_ID`, helpers de normalisation d’identifiant de compte |
    | `plugin-sdk/account-resolution` | Recherche de compte + helpers de repli par défaut |
    | `plugin-sdk/account-helpers` | Helpers étroits pour listes/actions de compte |
    | `plugin-sdk/channel-pairing` | `createChannelPairingController` |
    | `plugin-sdk/channel-reply-pipeline` | `createChannelReplyPipeline` |
    | `plugin-sdk/channel-config-helpers` | `createHybridChannelConfigAdapter` |
    | `plugin-sdk/channel-config-schema` | Types de schéma de configuration de canal |
    | `plugin-sdk/telegram-command-config` | Helpers de normalisation/validation des commandes personnalisées Telegram avec repli vers le contrat intégré |
    | `plugin-sdk/channel-policy` | `resolveChannelGroupRequireMention` |
    | `plugin-sdk/channel-lifecycle` | `createAccountStatusSink` |
    | `plugin-sdk/inbound-envelope` | Helpers partagés de routage entrant + construction d’enveloppe |
    | `plugin-sdk/inbound-reply-dispatch` | Helpers partagés d’enregistrement entrant et de distribution |
    | `plugin-sdk/messaging-targets` | Helpers d’analyse/correspondance des cibles |
    | `plugin-sdk/outbound-media` | Helpers partagés de chargement de médias sortants |
    | `plugin-sdk/outbound-runtime` | Helpers d’identité sortante et de délégation d’envoi |
    | `plugin-sdk/thread-bindings-runtime` | Helpers de cycle de vie des liaisons de fil et d’adaptateur |
    | `plugin-sdk/agent-media-payload` | Constructeur hérité de charge utile média pour agent |
    | `plugin-sdk/conversation-runtime` | Helpers de liaison conversation/fil, d’appairage et de liaison configurée |
    | `plugin-sdk/runtime-config-snapshot` | Helper d’instantané de configuration d’exécution |
    | `plugin-sdk/runtime-group-policy` | Helpers de résolution de politique de groupe à l’exécution |
    | `plugin-sdk/channel-status` | Helpers partagés d’instantané/résumé d’état de canal |
    | `plugin-sdk/channel-config-primitives` | Primitives étroites de schéma de configuration de canal |
    | `plugin-sdk/channel-config-writes` | Helpers d’autorisation d’écriture de configuration de canal |
    | `plugin-sdk/channel-plugin-common` | Exports de prélude partagés pour plugins de canal |
    | `plugin-sdk/allowlist-config-edit` | Helpers de lecture/modification de configuration de liste d’autorisation |
    | `plugin-sdk/group-access` | Helpers partagés de décision d’accès aux groupes |
    | `plugin-sdk/direct-dm` | Helpers partagés d’authentification/protection pour messages privés directs |
    | `plugin-sdk/interactive-runtime` | Helpers de normalisation/réduction de charge utile de réponse interactive |
    | `plugin-sdk/channel-inbound` | Helpers de debounce, correspondance de mention, enveloppe |
    | `plugin-sdk/channel-send-result` | Types de résultat de réponse |
    | `plugin-sdk/channel-actions` | `createMessageToolButtonsSchema`, `createMessageToolCardSchema` |
    | `plugin-sdk/channel-targets` | Helpers d’analyse/correspondance des cibles |
    | `plugin-sdk/channel-contract` | Types de contrat de canal |
    | `plugin-sdk/channel-feedback` | Câblage de feedback/réaction |
    | `plugin-sdk/channel-secret-runtime` | Helpers étroits de contrat de secrets tels que `collectSimpleChannelFieldAssignments`, `getChannelSurface`, `pushAssignment`, et types de cible de secrets |
  </Accordion>

  <Accordion title="Sous-chemins provider">
    | Sous-chemin | Exports clés |
    | --- | --- |
    | `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry` |
    | `plugin-sdk/provider-setup` | Helpers de configuration sélectionnés pour providers locaux/autohébergés |
    | `plugin-sdk/self-hosted-provider-setup` | Helpers ciblés de configuration de provider autohébergé compatible OpenAI |
    | `plugin-sdk/cli-backend` | Valeurs par défaut de backend CLI + constantes watchdog |
    | `plugin-sdk/provider-auth-runtime` | Helpers d’exécution de résolution de clé API pour plugins provider |
    | `plugin-sdk/provider-auth-api-key` | Helpers d’onboarding/d’écriture de profil de clé API tels que `upsertApiKeyProfile` |
    | `plugin-sdk/provider-auth-result` | Constructeur standard de résultat d’authentification OAuth |
    | `plugin-sdk/provider-auth-login` | Helpers partagés de connexion interactive pour plugins provider |
    | `plugin-sdk/provider-env-vars` | Helpers de recherche de variables d’environnement d’authentification provider |
    | `plugin-sdk/provider-auth` | `createProviderApiKeyAuthMethod`, `ensureApiKeyFromOptionEnvOrPrompt`, `upsertAuthProfile`, `upsertApiKeyProfile`, `writeOAuthCredentials` |
    | `plugin-sdk/provider-model-shared` | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, constructeurs partagés de politique de relecture, helpers de point de terminaison provider, et helpers de normalisation d’identifiant de modèle tels que `normalizeNativeXaiModelId` |
    | `plugin-sdk/provider-catalog-shared` | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
    | `plugin-sdk/provider-http` | Helpers génériques de capacités HTTP/point de terminaison provider |
    | `plugin-sdk/provider-web-fetch` | Helpers d’enregistrement/cache de provider web-fetch |
    | `plugin-sdk/provider-web-search-contract` | Helpers étroits de contrat de configuration/identifiants web-search tels que `enablePluginInConfig`, `resolveProviderWebSearchPluginConfig`, et accesseurs/setters d’identifiants à portée limitée |
    | `plugin-sdk/provider-web-search` | Helpers d’enregistrement/cache/exécution de provider web-search |
    | `plugin-sdk/provider-tools` | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, nettoyage de schéma Gemini + diagnostics, et helpers de compatibilité xAI tels que `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
    | `plugin-sdk/provider-usage` | `fetchClaudeUsage` et similaires |
    | `plugin-sdk/provider-stream` | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, types de wrapper de flux, et helpers partagés de wrapper pour Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot |
    | `plugin-sdk/provider-onboard` | Helpers de patch de configuration d’onboarding |
    | `plugin-sdk/global-singleton` | Helpers de singleton/map/cache locaux au processus |
  </Accordion>

  <Accordion title="Sous-chemins d’authentification et de sécurité">
    | Sous-chemin | Exports clés |
    | --- | --- |
    | `plugin-sdk/command-auth` | `resolveControlCommandGate`, helpers de registre de commandes, helpers d’autorisation d’expéditeur |
    | `plugin-sdk/approval-auth-runtime` | Helpers de résolution d’approbateur et d’authentification d’action dans la même discussion |
    | `plugin-sdk/approval-client-runtime` | Helpers de profil/filtre d’approbation d’exécution native |
    | `plugin-sdk/approval-delivery-runtime` | Adaptateurs natifs de capacité/livraison d’approbation |
    | `plugin-sdk/approval-native-runtime` | Helpers natifs de cible d’approbation + liaison de compte |
    | `plugin-sdk/approval-reply-runtime` | Helpers de charge utile de réponse d’approbation d’exécution/plugin |
    | `plugin-sdk/command-auth-native` | Authentification native de commande + helpers natifs de cible de session |
    | `plugin-sdk/command-detection` | Helpers partagés de détection de commandes |
    | `plugin-sdk/command-surface` | Helpers de normalisation du corps de commande et de surface de commande |
    | `plugin-sdk/allow-from` | `formatAllowFromLowercase` |
    | `plugin-sdk/channel-secret-runtime` | Helpers étroits de collecte de contrat de secrets pour les surfaces de secrets de canal/plugin |
    | `plugin-sdk/secret-ref-runtime` | Helpers étroits `coerceSecretRef` et de typage SecretRef pour l’analyse du contrat de secrets/de la configuration |
    | `plugin-sdk/security-runtime` | Helpers partagés de confiance, de filtrage des messages privés, de contenu externe et de collecte de secrets |
    | `plugin-sdk/ssrf-policy` | Helpers de liste d’autorisation d’hôte et de politique SSRF réseau privé |
    | `plugin-sdk/ssrf-runtime` | Helpers de dispatcher épinglé, de fetch protégé par SSRF et de politique SSRF |
    | `plugin-sdk/secret-input` | Helpers d’analyse d’entrée de secret |
    | `plugin-sdk/webhook-ingress` | Helpers de requête/cible de webhook |
    | `plugin-sdk/webhook-request-guards` | Helpers de taille de corps de requête/timeout |
  </Accordion>

  <Accordion title="Sous-chemins d’exécution et de stockage">
    | Sous-chemin | Exports clés |
    | --- | --- |
    | `plugin-sdk/runtime` | Helpers larges d’exécution, de journalisation, de sauvegarde et d’installation de plugin |
    | `plugin-sdk/runtime-env` | Helpers étroits d’environnement d’exécution, logger, timeout, retry et backoff |
    | `plugin-sdk/runtime-store` | `createPluginRuntimeStore` |
    | `plugin-sdk/plugin-runtime` | Helpers partagés de commande/hook/http/interaction pour plugin |
    | `plugin-sdk/hook-runtime` | Helpers partagés de pipeline de webhook/hook interne |
    | `plugin-sdk/lazy-runtime` | Helpers de chargement paresseux d’exécution/de liaison tels que `createLazyRuntimeModule`, `createLazyRuntimeMethod` et `createLazyRuntimeSurface` |
    | `plugin-sdk/process-runtime` | Helpers d’exécution de processus |
    | `plugin-sdk/cli-runtime` | Helpers de formatage CLI, d’attente et de version |
    | `plugin-sdk/gateway-runtime` | Helpers de client Gateway et de patch d’état de canal |
    | `plugin-sdk/config-runtime` | Helpers de chargement/écriture de configuration |
    | `plugin-sdk/telegram-command-config` | Normalisation des noms/descriptions de commandes Telegram et vérifications de doublons/conflits, même lorsque la surface de contrat Telegram intégrée n’est pas disponible |
    | `plugin-sdk/approval-runtime` | Helpers d’approbation d’exécution/plugin, constructeurs de capacités d’approbation, helpers auth/profil, helpers natifs de routage/d’exécution |
    | `plugin-sdk/reply-runtime` | Helpers partagés d’exécution entrante/de réponse, chunking, distribution, heartbeat, planificateur de réponse |
    | `plugin-sdk/reply-dispatch-runtime` | Helpers étroits de distribution/finalisation de réponse |
    | `plugin-sdk/reply-history` | Helpers partagés d’historique de réponse sur fenêtre courte tels que `buildHistoryContext`, `recordPendingHistoryEntry` et `clearHistoryEntriesIfEnabled` |
    | `plugin-sdk/reply-reference` | `createReplyReferencePlanner` |
    | `plugin-sdk/reply-chunking` | Helpers étroits de chunking texte/Markdown |
    | `plugin-sdk/session-store-runtime` | Helpers de chemin de stockage de session + updated-at |
    | `plugin-sdk/state-paths` | Helpers de chemin pour répertoires state/OAuth |
    | `plugin-sdk/routing` | Helpers de routage/clé de session/liaison de compte tels que `resolveAgentRoute`, `buildAgentSessionKey` et `resolveDefaultAgentBoundAccountId` |
    | `plugin-sdk/status-helpers` | Helpers partagés de résumé d’état de canal/compte, valeurs par défaut d’état d’exécution et helpers de métadonnées de problème |
    | `plugin-sdk/target-resolver-runtime` | Helpers partagés de résolution de cible |
    | `plugin-sdk/string-normalization-runtime` | Helpers de normalisation de slug/chaîne |
    | `plugin-sdk/request-url` | Extraire des URL chaîne depuis des entrées de type fetch/request |
    | `plugin-sdk/run-command` | Exécuteur de commande temporisé avec résultats stdout/stderr normalisés |
    | `plugin-sdk/param-readers` | Lecteurs communs de paramètres d’outil/CLI |
    | `plugin-sdk/tool-send` | Extraire les champs de cible d’envoi canoniques depuis les arguments d’outil |
    | `plugin-sdk/temp-path` | Helpers partagés de chemin de téléchargement temporaire |
    | `plugin-sdk/logging-core` | Helpers de logger de sous-système et de masquage |
    | `plugin-sdk/markdown-table-runtime` | Helpers de mode tableau Markdown |
    | `plugin-sdk/json-store` | Petits helpers de lecture/écriture d’état JSON |
    | `plugin-sdk/file-lock` | Helpers de verrou de fichier réentrant |
    | `plugin-sdk/persistent-dedupe` | Helpers de cache de déduplication persistant sur disque |
    | `plugin-sdk/acp-runtime` | Helpers ACP d’exécution/session et de distribution de réponse |
    | `plugin-sdk/agent-config-primitives` | Primitives étroites de schéma de configuration d’exécution d’agent |
    | `plugin-sdk/boolean-param` | Lecteur tolérant de paramètre booléen |
    | `plugin-sdk/dangerous-name-runtime` | Helpers de résolution de correspondance de noms dangereux |
    | `plugin-sdk/device-bootstrap` | Helpers d’initialisation d’appareil et de jeton d’appairage |
    | `plugin-sdk/extension-shared` | Primitives partagées de canal passif, d’état et de proxy ambiant |
    | `plugin-sdk/models-provider-runtime` | Helpers de réponse provider/commande `/models` |
    | `plugin-sdk/skill-commands-runtime` | Helpers de liste des commandes Skills |
    | `plugin-sdk/native-command-registry` | Helpers natifs de registre/construction/sérialisation de commandes |
    | `plugin-sdk/provider-zai-endpoint` | Helpers de détection d’endpoint Z.A.I |
    | `plugin-sdk/infra-runtime` | Helpers d’événement système/heartbeat |
    | `plugin-sdk/collection-runtime` | Petits helpers de cache borné |
    | `plugin-sdk/diagnostic-runtime` | Helpers d’indicateur et d’événement de diagnostic |
    | `plugin-sdk/error-runtime` | Graphe d’erreurs, formatage, helpers partagés de classification d’erreurs, `isApprovalNotFoundError` |
    | `plugin-sdk/fetch-runtime` | Helpers de fetch encapsulé, proxy et recherche épinglée |
    | `plugin-sdk/host-runtime` | Helpers de normalisation de nom d’hôte et d’hôte SCP |
    | `plugin-sdk/retry-runtime` | Helpers de configuration d’essais et d’exécuteur de retry |
    | `plugin-sdk/agent-runtime` | Helpers de répertoire/identité/espace de travail d’agent |
    | `plugin-sdk/directory-runtime` | Requête/déduplication de répertoire basée sur la configuration |
    | `plugin-sdk/keyed-async-queue` | `KeyedAsyncQueue` |
  </Accordion>

  <Accordion title="Sous-chemins de capacités et de tests">
    | Sous-chemin | Exports clés |
    | --- | --- |
    | `plugin-sdk/media-runtime` | Helpers partagés de récupération/transformation/stockage de médias ainsi que constructeurs de charge utile média |
    | `plugin-sdk/media-generation-runtime` | Helpers partagés de basculement media-generation, sélection de candidats et messages de modèle manquant |
    | `plugin-sdk/media-understanding` | Types de provider media-understanding ainsi qu’exports de helpers image/audio orientés provider |
    | `plugin-sdk/text-runtime` | Helpers partagés de texte/Markdown/journalisation tels que suppression du texte visible par l’assistant, rendu/chunking/tableaux Markdown, masquage, helpers de balises de directives et utilitaires de texte sûr |
    | `plugin-sdk/text-chunking` | Helper de chunking de texte sortant |
    | `plugin-sdk/speech` | Types de provider speech ainsi qu’helpers orientés provider pour directives, registre et validation |
    | `plugin-sdk/speech-core` | Helpers partagés de types, registre, directives et normalisation pour provider speech |
    | `plugin-sdk/realtime-transcription` | Types de provider de transcription en temps réel et helpers de registre |
    | `plugin-sdk/realtime-voice` | Types de provider de voix en temps réel et helpers de registre |
    | `plugin-sdk/image-generation` | Types de provider de génération d’image |
    | `plugin-sdk/image-generation-core` | Helpers partagés de types, basculement, authentification et registre pour image-generation |
    | `plugin-sdk/music-generation` | Types de provider/requête/résultat pour génération musicale |
    | `plugin-sdk/music-generation-core` | Helpers partagés de types, de basculement, de recherche de provider et d’analyse de model-ref pour music-generation |
    | `plugin-sdk/video-generation` | Types de provider/requête/résultat pour génération vidéo |
    | `plugin-sdk/video-generation-core` | Helpers partagés de types, de basculement, de recherche de provider et d’analyse de model-ref pour video-generation |
    | `plugin-sdk/webhook-targets` | Registre des cibles de webhook et helpers d’installation de route |
    | `plugin-sdk/webhook-path` | Helpers de normalisation de chemin de webhook |
    | `plugin-sdk/web-media` | Helpers partagés de chargement de médias distants/locaux |
    | `plugin-sdk/zod` | `zod` réexporté pour les consommateurs du Plugin SDK |
    | `plugin-sdk/testing` | `installCommonResolveTargetErrorCases`, `shouldAckReaction` |
  </Accordion>

  <Accordion title="Sous-chemins mémoire">
    | Sous-chemin | Exports clés |
    | --- | --- |
    | `plugin-sdk/memory-core` | Surface de helpers memory-core intégrée pour les helpers de gestionnaire/configuration/fichier/CLI |
    | `plugin-sdk/memory-core-engine-runtime` | Façade d’exécution d’indexation/recherche mémoire |
    | `plugin-sdk/memory-core-host-engine-foundation` | Exports du moteur fondation de l’hôte mémoire |
    | `plugin-sdk/memory-core-host-engine-embeddings` | Exports du moteur d’embeddings de l’hôte mémoire |
    | `plugin-sdk/memory-core-host-engine-qmd` | Exports du moteur QMD de l’hôte mémoire |
    | `plugin-sdk/memory-core-host-engine-storage` | Exports du moteur de stockage de l’hôte mémoire |
    | `plugin-sdk/memory-core-host-multimodal` | Helpers multimodaux de l’hôte mémoire |
    | `plugin-sdk/memory-core-host-query` | Helpers de requête de l’hôte mémoire |
    | `plugin-sdk/memory-core-host-secret` | Helpers de secret de l’hôte mémoire |
    | `plugin-sdk/memory-core-host-events` | Helpers de journal d’événements de l’hôte mémoire |
    | `plugin-sdk/memory-core-host-status` | Helpers d’état de l’hôte mémoire |
    | `plugin-sdk/memory-core-host-runtime-cli` | Helpers d’exécution CLI de l’hôte mémoire |
    | `plugin-sdk/memory-core-host-runtime-core` | Helpers d’exécution cœur de l’hôte mémoire |
    | `plugin-sdk/memory-core-host-runtime-files` | Helpers de fichier/d’exécution de l’hôte mémoire |
    | `plugin-sdk/memory-host-core` | Alias neutre vis-à-vis du fournisseur pour les helpers d’exécution du cœur de l’hôte mémoire |
    | `plugin-sdk/memory-host-events` | Alias neutre vis-à-vis du fournisseur pour les helpers de journal d’événements de l’hôte mémoire |
    | `plugin-sdk/memory-host-files` | Alias neutre vis-à-vis du fournisseur pour les helpers de fichier/d’exécution de l’hôte mémoire |
    | `plugin-sdk/memory-host-markdown` | Helpers partagés de Markdown géré pour les plugins proches de la mémoire |
    | `plugin-sdk/memory-host-search` | Façade active d’exécution mémoire pour l’accès au gestionnaire de recherche |
    | `plugin-sdk/memory-host-status` | Alias neutre vis-à-vis du fournisseur pour les helpers d’état de l’hôte mémoire |
    | `plugin-sdk/memory-lancedb` | Surface de helpers memory-lancedb intégrée |
  </Accordion>

  <Accordion title="Sous-chemins réservés de helpers intégrés">
    | Famille | Sous-chemins actuels | Usage prévu |
    | --- | --- | --- |
    | Browser | `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support` | Helpers de prise en charge du plugin Browser intégré (`browser-support` reste le baril de compatibilité) |
    | Matrix | `plugin-sdk/matrix`, `plugin-sdk/matrix-helper`, `plugin-sdk/matrix-runtime-heavy`, `plugin-sdk/matrix-runtime-shared`, `plugin-sdk/matrix-runtime-surface`, `plugin-sdk/matrix-surface`, `plugin-sdk/matrix-thread-bindings` | Surface intégrée de helpers/runtime Matrix |
    | Line | `plugin-sdk/line`, `plugin-sdk/line-core`, `plugin-sdk/line-runtime`, `plugin-sdk/line-surface` | Surface intégrée de helpers/runtime LINE |
    | IRC | `plugin-sdk/irc`, `plugin-sdk/irc-surface` | Surface intégrée de helpers IRC |
    | Helpers spécifiques aux canaux | `plugin-sdk/googlechat`, `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles`, `plugin-sdk/bluebubbles-policy`, `plugin-sdk/mattermost`, `plugin-sdk/mattermost-policy`, `plugin-sdk/feishu-conversation`, `plugin-sdk/msteams`, `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`, `plugin-sdk/twitch` | Points d’accès de compatibilité/helpers pour canaux intégrés |
    | Helpers spécifiques à l’authentification/au plugin | `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`, `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`, `plugin-sdk/thread-ownership`, `plugin-sdk/voice-call` | Points d’accès de helpers pour fonctionnalités/plugins intégrés ; `plugin-sdk/github-copilot-token` exporte actuellement `DEFAULT_COPILOT_API_BASE_URL`, `deriveCopilotApiBaseUrlFromToken` et `resolveCopilotApiToken` |
  </Accordion>
</AccordionGroup>

## API d’enregistrement

Le callback `register(api)` reçoit un objet `OpenClawPluginApi` avec ces
méthodes :

### Enregistrement des capacités

| Méthode                                          | Ce qu’elle enregistre             |
| ------------------------------------------------ | --------------------------------- |
| `api.registerProvider(...)`                      | Inférence texte (LLM)             |
| `api.registerCliBackend(...)`                    | Backend d’inférence CLI local     |
| `api.registerChannel(...)`                       | Canal de messagerie               |
| `api.registerSpeechProvider(...)`                | Synthèse texte-parole / STT       |
| `api.registerRealtimeTranscriptionProvider(...)` | Transcription temps réel en flux  |
| `api.registerRealtimeVoiceProvider(...)`         | Sessions vocales temps réel duplex |
| `api.registerMediaUnderstandingProvider(...)`    | Analyse d’image/audio/vidéo       |
| `api.registerImageGenerationProvider(...)`       | Génération d’image                |
| `api.registerMusicGenerationProvider(...)`       | Génération musicale               |
| `api.registerVideoGenerationProvider(...)`       | Génération vidéo                  |
| `api.registerWebFetchProvider(...)`              | Provider de récupération / scraping web |
| `api.registerWebSearchProvider(...)`             | Recherche web                     |

### Outils et commandes

| Méthode                         | Ce qu’elle enregistre                        |
| ------------------------------- | -------------------------------------------- |
| `api.registerTool(tool, opts?)` | Outil d’agent (obligatoire ou `{ optional: true }`) |
| `api.registerCommand(def)`      | Commande personnalisée (contourne le LLM)    |

### Infrastructure

| Méthode                                        | Ce qu’elle enregistre                  |
| ---------------------------------------------- | -------------------------------------- |
| `api.registerHook(events, handler, opts?)`     | Hook d’événement                       |
| `api.registerHttpRoute(params)`                | Endpoint HTTP Gateway                  |
| `api.registerGatewayMethod(name, handler)`     | Méthode RPC Gateway                    |
| `api.registerCli(registrar, opts?)`            | Sous-commande CLI                      |
| `api.registerService(service)`                 | Service en arrière-plan                |
| `api.registerInteractiveHandler(registration)` | Gestionnaire interactif                |
| `api.registerMemoryPromptSupplement(builder)`  | Section de prompt additive adjacente à la mémoire |
| `api.registerMemoryCorpusSupplement(adapter)`  | Corpus additif de recherche/lecture mémoire |

Les espaces de noms d’administration du cœur réservés (`config.*`, `exec.approvals.*`, `wizard.*`,
`update.*`) restent toujours `operator.admin`, même si un plugin tente d’assigner une
portée de méthode Gateway plus étroite. Préférez des préfixes spécifiques au plugin pour les
méthodes appartenant au plugin.

### Métadonnées d’enregistrement CLI

`api.registerCli(registrar, opts?)` accepte deux types de métadonnées de premier niveau :

- `commands` : racines de commandes explicites appartenant au registrar
- `descriptors` : descripteurs de commandes à l’analyse, utilisés pour l’aide CLI racine,
  le routage et l’enregistrement CLI paresseux du plugin

Si vous voulez qu’une commande de plugin reste chargée paresseusement dans le chemin CLI racine normal,
fournissez des `descriptors` couvrant chaque racine de commande de premier niveau exposée par ce
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
        description: "Gérer les comptes Matrix, la vérification, les appareils et l’état du profil",
        hasSubcommands: true,
      },
    ],
  },
);
```

Utilisez `commands` seul uniquement lorsque vous n’avez pas besoin d’un enregistrement CLI racine paresseux.
Ce chemin de compatibilité eager reste pris en charge, mais il n’installe pas
d’espaces réservés fondés sur des descripteurs pour le chargement paresseux au moment de l’analyse.

### Enregistrement de backend CLI

`api.registerCliBackend(...)` permet à un plugin de posséder la configuration par défaut d’un
backend CLI IA local tel que `codex-cli`.

- Le `id` du backend devient le préfixe provider dans des références de modèle comme `codex-cli/gpt-5`.
- Le `config` du backend utilise la même forme que `agents.defaults.cliBackends.<id>`.
- La configuration utilisateur reste prioritaire. OpenClaw fusionne `agents.defaults.cliBackends.<id>` sur la
  valeur par défaut du plugin avant d’exécuter la CLI.
- Utilisez `normalizeConfig` lorsqu’un backend a besoin de réécritures de compatibilité après fusion
  (par exemple pour normaliser d’anciennes formes de drapeaux).

### Emplacements exclusifs

| Méthode                                    | Ce qu’elle enregistre                |
| ------------------------------------------ | ------------------------------------ |
| `api.registerContextEngine(id, factory)`   | Moteur de contexte (un seul actif à la fois) |
| `api.registerMemoryPromptSection(builder)` | Constructeur de section de prompt mémoire |
| `api.registerMemoryFlushPlan(resolver)`    | Résolveur de plan de vidage mémoire  |
| `api.registerMemoryRuntime(runtime)`       | Adaptateur d’exécution mémoire       |

### Adaptateurs d’embeddings mémoire

| Méthode                                        | Ce qu’elle enregistre                             |
| --------------------------------------------- | ------------------------------------------------- |
| `api.registerMemoryEmbeddingProvider(adapter)` | Adaptateur d’embeddings mémoire pour le plugin actif |

- `registerMemoryPromptSection`, `registerMemoryFlushPlan` et
  `registerMemoryRuntime` sont exclusifs aux plugins mémoire.
- `registerMemoryEmbeddingProvider` permet au plugin mémoire actif d’enregistrer un
  ou plusieurs identifiants d’adaptateur d’embeddings (par exemple `openai`, `gemini` ou un identifiant
  personnalisé défini par le plugin).
- La configuration utilisateur telle que `agents.defaults.memorySearch.provider` et
  `agents.defaults.memorySearch.fallback` se résout par rapport à ces identifiants
  d’adaptateur enregistrés.

### Événements et cycle de vie

| Méthode                                      | Ce qu’elle fait              |
| -------------------------------------------- | ---------------------------- |
| `api.on(hookName, handler, opts?)`           | Hook de cycle de vie typé    |
| `api.onConversationBindingResolved(handler)` | Callback de liaison de conversation |

### Sémantique de décision des hooks

- `before_tool_call` : renvoyer `{ block: true }` est terminal. Dès qu’un gestionnaire le définit, les gestionnaires de priorité inférieure sont ignorés.
- `before_tool_call` : renvoyer `{ block: false }` est traité comme une absence de décision (comme si `block` était omis), et non comme un remplacement.
- `before_install` : renvoyer `{ block: true }` est terminal. Dès qu’un gestionnaire le définit, les gestionnaires de priorité inférieure sont ignorés.
- `before_install` : renvoyer `{ block: false }` est traité comme une absence de décision (comme si `block` était omis), et non comme un remplacement.
- `reply_dispatch` : renvoyer `{ handled: true, ... }` est terminal. Dès qu’un gestionnaire revendique la distribution, les gestionnaires de priorité inférieure et le chemin de distribution par défaut du modèle sont ignorés.
- `message_sending` : renvoyer `{ cancel: true }` est terminal. Dès qu’un gestionnaire le définit, les gestionnaires de priorité inférieure sont ignorés.
- `message_sending` : renvoyer `{ cancel: false }` est traité comme une absence de décision (comme si `cancel` était omis), et non comme un remplacement.

### Champs de l’objet API

| Champ                    | Type                      | Description                                                                                 |
| ------------------------ | ------------------------- | ------------------------------------------------------------------------------------------- |
| `api.id`                 | `string`                  | Identifiant du plugin                                                                       |
| `api.name`               | `string`                  | Nom d’affichage                                                                             |
| `api.version`            | `string?`                 | Version du plugin (facultatif)                                                              |
| `api.description`        | `string?`                 | Description du plugin (facultatif)                                                          |
| `api.source`             | `string`                  | Chemin source du plugin                                                                     |
| `api.rootDir`            | `string?`                 | Répertoire racine du plugin (facultatif)                                                    |
| `api.config`             | `OpenClawConfig`          | Instantané de configuration actuel (instantané d’exécution actif en mémoire lorsqu’il est disponible) |
| `api.pluginConfig`       | `Record<string, unknown>` | Configuration spécifique au plugin depuis `plugins.entries.<id>.config`                     |
| `api.runtime`            | `PluginRuntime`           | [Helpers d’exécution](/fr/plugins/sdk-runtime)                                                 |
| `api.logger`             | `PluginLogger`            | Logger à portée limitée (`debug`, `info`, `warn`, `error`)                                  |
| `api.registrationMode`   | `PluginRegistrationMode`  | Mode de chargement actuel ; `"setup-runtime"` est la fenêtre légère de démarrage/configuration avant l’entrée complète |
| `api.resolvePath(input)` | `(string) => string`      | Résoudre un chemin relativement à la racine du plugin                                       |

## Convention de module interne

Dans votre plugin, utilisez des fichiers barils locaux pour les imports internes :

```
my-plugin/
  api.ts            # Exports publics pour les consommateurs externes
  runtime-api.ts    # Exports d’exécution internes uniquement
  index.ts          # Point d’entrée du plugin
  setup-entry.ts    # Entrée légère réservée à la configuration (facultatif)
```

<Warning>
  N’importez jamais votre propre plugin via `openclaw/plugin-sdk/<your-plugin>`
  depuis le code de production. Faites passer les imports internes par `./api.ts` ou
  `./runtime-api.ts`. Le chemin SDK est uniquement le contrat externe.
</Warning>

Les surfaces publiques de plugins intégrés chargées par façade (`api.ts`, `runtime-api.ts`,
`index.ts`, `setup-entry.ts` et fichiers d’entrée publics similaires) préfèrent désormais
l’instantané de configuration d’exécution actif lorsqu’OpenClaw est déjà lancé. Si aucun instantané
d’exécution n’existe encore, elles reviennent au fichier de configuration résolu sur disque.

Les plugins provider peuvent également exposer un baril de contrat local au plugin et étroit lorsqu’un
helper est intentionnellement spécifique à un provider et n’a pas encore sa place dans un sous-chemin SDK générique. Exemple intégré actuel : le provider Anthropic conserve ses helpers de flux Claude dans son propre point d’accès public `api.ts` / `contract-api.ts` au lieu de promouvoir la logique d’en-tête bêta Anthropic et `service_tier` dans un contrat générique `plugin-sdk/*`.

Autres exemples intégrés actuels :

- `@openclaw/openai-provider` : `api.ts` exporte des constructeurs de provider,
  des helpers de modèle par défaut et des constructeurs de provider temps réel
- `@openclaw/openrouter-provider` : `api.ts` exporte le constructeur de provider ainsi que
  des helpers d’onboarding/de configuration

<Warning>
  Le code de production des extensions doit également éviter les imports
  `openclaw/plugin-sdk/<other-plugin>`. Si un helper est réellement partagé, promouvez-le vers un sous-chemin SDK neutre
  tel que `openclaw/plugin-sdk/speech`, `.../provider-model-shared` ou une autre
  surface orientée capacité, au lieu de coupler deux plugins entre eux.
</Warning>

## Voir aussi

- [Points d’entrée](/fr/plugins/sdk-entrypoints) — options de `definePluginEntry` et `defineChannelPluginEntry`
- [Helpers d’exécution](/fr/plugins/sdk-runtime) — référence complète de l’espace de noms `api.runtime`
- [Configuration et setup](/fr/plugins/sdk-setup) — packaging, manifestes, schémas de configuration
- [Tests](/fr/plugins/sdk-testing) — utilitaires de test et règles de lint
- [Migration du SDK](/fr/plugins/sdk-migration) — migration depuis les surfaces obsolètes
- [Internes des plugins](/fr/plugins/architecture) — architecture détaillée et modèle de capacités
