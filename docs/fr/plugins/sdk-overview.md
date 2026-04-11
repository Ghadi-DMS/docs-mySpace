---
read_when:
    - Vous devez savoir depuis quel sous-chemin du SDK importer
    - Vous voulez une référence pour toutes les méthodes d'enregistrement sur OpenClawPluginApi
    - Vous recherchez un export spécifique du SDK
sidebarTitle: SDK Overview
summary: Import map, référence de l'API d'enregistrement et architecture du SDK
title: Vue d'ensemble du SDK de plugin
x-i18n:
    generated_at: "2026-04-11T02:46:28Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4bfeb5896f68e3e4ee8cf434d43a019e0d1fe5af57f5bf7a5172847c476def0c
    source_path: plugins/sdk-overview.md
    workflow: 15
---

# Vue d'ensemble du SDK de plugin

Le SDK de plugin est le contrat typé entre les plugins et le cœur. Cette page est la
référence pour **quoi importer** et **ce que vous pouvez enregistrer**.

<Tip>
  **Vous cherchez un guide pratique ?**
  - Premier plugin ? Commencez par [Getting Started](/fr/plugins/building-plugins)
  - Plugin de channel ? Voir [Channel Plugins](/fr/plugins/sdk-channel-plugins)
  - Plugin de provider ? Voir [Provider Plugins](/fr/plugins/sdk-provider-plugins)
</Tip>

## Convention d'import

Importez toujours depuis un sous-chemin spécifique :

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
```

Chaque sous-chemin est un petit module autonome. Cela permet de conserver un démarrage rapide et
d'éviter les problèmes de dépendances circulaires. Pour les helpers d'entrée/build spécifiques aux channels,
préférez `openclaw/plugin-sdk/channel-core` ; gardez `openclaw/plugin-sdk/core` pour
la surface ombrelle plus large et les helpers partagés tels que
`buildChannelConfigSchema`.

N'ajoutez pas et ne dépendez pas de surfaces de commodité nommées d'après des providers telles que
`openclaw/plugin-sdk/slack`, `openclaw/plugin-sdk/discord`,
`openclaw/plugin-sdk/signal`, `openclaw/plugin-sdk/whatsapp`, ou
de surfaces helper associées à une marque de channel. Les plugins intégrés doivent composer des sous-chemins
SDK génériques dans leurs propres barils `api.ts` ou `runtime-api.ts`, et le cœur
doit soit utiliser ces barils locaux au plugin, soit ajouter un contrat SDK générique étroit
lorsque le besoin est réellement inter-channel.

La map d'exports générée contient encore un petit ensemble de
surfaces helper de plugins intégrés telles que `plugin-sdk/feishu`, `plugin-sdk/feishu-setup`,
`plugin-sdk/zalo`, `plugin-sdk/zalo-setup` et `plugin-sdk/matrix*`. Ces
sous-chemins n'existent que pour la maintenance et la compatibilité des plugins intégrés ; ils sont
volontairement omis du tableau courant ci-dessous et ne constituent pas le
chemin d'import recommandé pour les nouveaux plugins tiers.

## Référence des sous-chemins

Les sous-chemins les plus couramment utilisés, regroupés par objectif. La liste complète générée de
plus de 200 sous-chemins se trouve dans `scripts/lib/plugin-sdk-entrypoints.json`.

Les sous-chemins helper réservés aux plugins intégrés apparaissent toujours dans cette liste générée.
Traitez-les comme des surfaces de détail d'implémentation/de compatibilité, sauf si une page de documentation
en promeut explicitement une comme publique.

### Entrée de plugin

| Sous-chemin                | Exports clés                                                                                                                            |
| -------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/plugin-entry`  | `definePluginEntry`                                                                                                                      |
| `plugin-sdk/core`          | `defineChannelPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase`, `defineSetupPluginEntry`, `buildChannelConfigSchema` |
| `plugin-sdk/config-schema` | `OpenClawSchema`                                                                                                                         |
| `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry`                                                                                                       |

<AccordionGroup>
  <Accordion title="Sous-chemins de channel">
    | Sous-chemin | Exports clés |
    | --- | --- |
    | `plugin-sdk/channel-core` | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
    | `plugin-sdk/config-schema` | Export du schéma Zod racine de `openclaw.json` (`OpenClawSchema`) |
    | `plugin-sdk/channel-setup` | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, plus `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
    | `plugin-sdk/setup` | Helpers partagés d'assistant de configuration, invites de liste d'autorisation, constructeurs d'état de configuration |
    | `plugin-sdk/setup-runtime` | `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
    | `plugin-sdk/setup-adapter-runtime` | `createEnvPatchedAccountSetupAdapter` |
    | `plugin-sdk/setup-tools` | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
    | `plugin-sdk/account-core` | Helpers de configuration/action-gate multi-comptes, helpers de fallback de compte par défaut |
    | `plugin-sdk/account-id` | `DEFAULT_ACCOUNT_ID`, helpers de normalisation d'identifiant de compte |
    | `plugin-sdk/account-resolution` | Recherche de compte + helpers de fallback par défaut |
    | `plugin-sdk/account-helpers` | Helpers étroits de liste de comptes/action sur compte |
    | `plugin-sdk/channel-pairing` | `createChannelPairingController` |
    | `plugin-sdk/channel-reply-pipeline` | `createChannelReplyPipeline` |
    | `plugin-sdk/channel-config-helpers` | `createHybridChannelConfigAdapter` |
    | `plugin-sdk/channel-config-schema` | Types de schéma de configuration de channel |
    | `plugin-sdk/telegram-command-config` | Helpers de normalisation/validation de commandes personnalisées Telegram avec fallback de contrat intégré |
    | `plugin-sdk/channel-policy` | `resolveChannelGroupRequireMention` |
    | `plugin-sdk/channel-lifecycle` | `createAccountStatusSink` |
    | `plugin-sdk/inbound-envelope` | Helpers partagés de route entrante + construction d'enveloppe |
    | `plugin-sdk/inbound-reply-dispatch` | Helpers partagés d'enregistrement et de répartition entrante |
    | `plugin-sdk/messaging-targets` | Helpers d'analyse/correspondance des cibles |
    | `plugin-sdk/outbound-media` | Helpers partagés de chargement des médias sortants |
    | `plugin-sdk/outbound-runtime` | Helpers de délégué d'identité/envoi sortants |
    | `plugin-sdk/thread-bindings-runtime` | Helpers de cycle de vie et d'adaptateur pour les liaisons de fil |
    | `plugin-sdk/agent-media-payload` | Constructeur historique de charge utile média d'agent |
    | `plugin-sdk/conversation-runtime` | Helpers de conversation/liaison de fil, appairage et liaison configurée |
    | `plugin-sdk/runtime-config-snapshot` | Helper d'instantané de configuration d'exécution |
    | `plugin-sdk/runtime-group-policy` | Helpers de résolution de politique de groupe à l'exécution |
    | `plugin-sdk/channel-status` | Helpers partagés d'instantané/résumé d'état de channel |
    | `plugin-sdk/channel-config-primitives` | Primitives étroites de schéma de configuration de channel |
    | `plugin-sdk/channel-config-writes` | Helpers d'autorisation d'écriture de configuration de channel |
    | `plugin-sdk/channel-plugin-common` | Exports de prélude partagés pour plugin de channel |
    | `plugin-sdk/allowlist-config-edit` | Helpers de lecture/modification de configuration de liste d'autorisation |
    | `plugin-sdk/group-access` | Helpers partagés de décision d'accès de groupe |
    | `plugin-sdk/direct-dm` | Helpers partagés d'authentification/garde de DM direct |
    | `plugin-sdk/interactive-runtime` | Helpers de normalisation/réduction de charges utiles de réponse interactive |
    | `plugin-sdk/channel-inbound` | Helpers d'anti-rebond entrant, de correspondance de mentions, d'aide de politique de mention et d'enveloppe |
    | `plugin-sdk/channel-send-result` | Types de résultat de réponse |
    | `plugin-sdk/channel-actions` | `createMessageToolButtonsSchema`, `createMessageToolCardSchema` |
    | `plugin-sdk/channel-targets` | Helpers d'analyse/correspondance des cibles |
    | `plugin-sdk/channel-contract` | Types de contrat de channel |
    | `plugin-sdk/channel-feedback` | Câblage des retours/réactions |
    | `plugin-sdk/channel-secret-runtime` | Helpers étroits de contrat de secret tels que `collectSimpleChannelFieldAssignments`, `getChannelSurface`, `pushAssignment`, et types de cible de secret |
  </Accordion>

  <Accordion title="Sous-chemins de provider">
    | Sous-chemin | Exports clés |
    | --- | --- |
    | `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry` |
    | `plugin-sdk/provider-setup` | Helpers sélectionnés de configuration de provider local/autohébergé |
    | `plugin-sdk/self-hosted-provider-setup` | Helpers ciblés de configuration de provider autohébergé compatible OpenAI |
    | `plugin-sdk/cli-backend` | Valeurs par défaut du backend CLI + constantes watchdog |
    | `plugin-sdk/provider-auth-runtime` | Helpers d'exécution de résolution de clé API pour les plugins de provider |
    | `plugin-sdk/provider-auth-api-key` | Helpers d'onboarding/écriture de profil de clé API tels que `upsertApiKeyProfile` |
    | `plugin-sdk/provider-auth-result` | Constructeur standard de résultat d'authentification OAuth |
    | `plugin-sdk/provider-auth-login` | Helpers interactifs partagés de connexion pour les plugins de provider |
    | `plugin-sdk/provider-env-vars` | Helpers de recherche de variables d'environnement d'authentification provider |
    | `plugin-sdk/provider-auth` | `createProviderApiKeyAuthMethod`, `ensureApiKeyFromOptionEnvOrPrompt`, `upsertAuthProfile`, `upsertApiKeyProfile`, `writeOAuthCredentials` |
    | `plugin-sdk/provider-model-shared` | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, constructeurs partagés de politique de replay, helpers de point de terminaison provider et helpers de normalisation d'identifiant de modèle tels que `normalizeNativeXaiModelId` |
    | `plugin-sdk/provider-catalog-shared` | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
    | `plugin-sdk/provider-http` | Helpers génériques de capacités HTTP/point de terminaison provider |
    | `plugin-sdk/provider-web-fetch-contract` | Helpers étroits de contrat de configuration/sélection web-fetch tels que `enablePluginInConfig` et `WebFetchProviderPlugin` |
    | `plugin-sdk/provider-web-fetch` | Helpers d'enregistrement/cache de provider web-fetch |
    | `plugin-sdk/provider-web-search-config-contract` | Helpers étroits de configuration/identifiants web-search pour les providers qui n'ont pas besoin de câblage d'activation de plugin |
    | `plugin-sdk/provider-web-search-contract` | Helpers étroits de contrat de configuration/identifiants web-search tels que `createWebSearchProviderContractFields`, `enablePluginInConfig`, `resolveProviderWebSearchPluginConfig`, et setters/getters d'identifiants limités par portée |
    | `plugin-sdk/provider-web-search` | Helpers d'enregistrement/cache/exécution de provider web-search |
    | `plugin-sdk/provider-tools` | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, nettoyage + diagnostics de schéma Gemini, et helpers de compatibilité xAI tels que `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
    | `plugin-sdk/provider-usage` | `fetchClaudeUsage` et similaires |
    | `plugin-sdk/provider-stream` | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, types de wrapper de flux, et helpers partagés de wrapper Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot |
    | `plugin-sdk/provider-onboard` | Helpers de patch de configuration d'onboarding |
    | `plugin-sdk/global-singleton` | Helpers de singleton/map/cache locaux au processus |
  </Accordion>

  <Accordion title="Sous-chemins d'authentification et de sécurité">
    | Sous-chemin | Exports clés |
    | --- | --- |
    | `plugin-sdk/command-auth` | `resolveControlCommandGate`, helpers de registre de commandes, helpers d'autorisation d'expéditeur |
    | `plugin-sdk/command-status` | Constructeurs de messages de commande/d'aide tels que `buildCommandsMessagePaginated` et `buildHelpMessage` |
    | `plugin-sdk/approval-auth-runtime` | Helpers de résolution d'approbateur et d'authentification d'action dans le même chat |
    | `plugin-sdk/approval-client-runtime` | Helpers de profil/filtre d'approbation native d'exécution |
    | `plugin-sdk/approval-delivery-runtime` | Adaptateurs de capacité/livraison d'approbation native |
    | `plugin-sdk/approval-gateway-runtime` | Helper partagé de résolution de gateway d'approbation |
    | `plugin-sdk/approval-handler-adapter-runtime` | Helpers légers de chargement d'adaptateur d'approbation native pour les points d'entrée de channel à chaud |
    | `plugin-sdk/approval-handler-runtime` | Helpers d'exécution plus larges du gestionnaire d'approbation ; préférez les surfaces adaptateur/gateway plus étroites lorsqu'elles suffisent |
    | `plugin-sdk/approval-native-runtime` | Helpers de cible d'approbation native + liaison de compte |
    | `plugin-sdk/approval-reply-runtime` | Helpers de charge utile de réponse d'approbation d'exécution/plugin |
    | `plugin-sdk/command-auth-native` | Helpers d'authentification de commande native + helpers de cible de session native |
    | `plugin-sdk/command-detection` | Helpers partagés de détection de commande |
    | `plugin-sdk/command-surface` | Normalisation du corps de commande et helpers de surface de commande |
    | `plugin-sdk/allow-from` | `formatAllowFromLowercase` |
    | `plugin-sdk/channel-secret-runtime` | Helpers étroits de collecte de contrat de secret pour les surfaces de secret de channel/plugin |
    | `plugin-sdk/secret-ref-runtime` | Helpers étroits `coerceSecretRef` et de typage SecretRef pour l'analyse de contrat de secret/configuration |
    | `plugin-sdk/security-runtime` | Helpers partagés de confiance, filtrage DM, contenu externe et collecte de secrets |
    | `plugin-sdk/ssrf-policy` | Helpers de liste d'autorisation d'hôte et de politique SSRF réseau privé |
    | `plugin-sdk/ssrf-runtime` | Helpers de dispatcher épinglé, fetch protégé par SSRF et politique SSRF |
    | `plugin-sdk/secret-input` | Helpers d'analyse d'entrée secrète |
    | `plugin-sdk/webhook-ingress` | Helpers de requête/cible webhook |
    | `plugin-sdk/webhook-request-guards` | Helpers de taille de corps de requête/délai d'expiration |
  </Accordion>

  <Accordion title="Sous-chemins d'exécution et de stockage">
    | Sous-chemin | Exports clés |
    | --- | --- |
    | `plugin-sdk/runtime` | Helpers larges d'exécution/journalisation/sauvegarde/installation de plugin |
    | `plugin-sdk/runtime-env` | Helpers étroits d'environnement d'exécution, logger, délai d'expiration, retry et backoff |
    | `plugin-sdk/channel-runtime-context` | Helpers génériques d'enregistrement et de recherche de contexte d'exécution de channel |
    | `plugin-sdk/runtime-store` | `createPluginRuntimeStore` |
    | `plugin-sdk/plugin-runtime` | Helpers partagés de commande/hook/http/interaction de plugin |
    | `plugin-sdk/hook-runtime` | Helpers partagés de pipeline de webhook/hook interne |
    | `plugin-sdk/lazy-runtime` | Helpers d'import/binding d'exécution paresseux tels que `createLazyRuntimeModule`, `createLazyRuntimeMethod` et `createLazyRuntimeSurface` |
    | `plugin-sdk/process-runtime` | Helpers d'exécution de processus |
    | `plugin-sdk/cli-runtime` | Helpers CLI de formatage, d'attente et de version |
    | `plugin-sdk/gateway-runtime` | Helpers de client gateway et de patch d'état de channel |
    | `plugin-sdk/config-runtime` | Helpers de chargement/écriture de configuration |
    | `plugin-sdk/telegram-command-config` | Normalisation du nom/de la description des commandes Telegram et vérifications de doublons/conflits, même lorsque la surface de contrat Telegram intégrée n'est pas disponible |
    | `plugin-sdk/approval-runtime` | Helpers d'approbation d'exécution/plugin, constructeurs de capacité d'approbation, helpers d'authentification/profil, helpers de routage/exécution natifs |
    | `plugin-sdk/reply-runtime` | Helpers partagés d'exécution entrante/réponse, fragmentation, répartition, heartbeat, planificateur de réponse |
    | `plugin-sdk/reply-dispatch-runtime` | Helpers étroits de répartition/finalisation de réponse |
    | `plugin-sdk/reply-history` | Helpers partagés d'historique de réponse sur fenêtre courte tels que `buildHistoryContext`, `recordPendingHistoryEntry` et `clearHistoryEntriesIfEnabled` |
    | `plugin-sdk/reply-reference` | `createReplyReferencePlanner` |
    | `plugin-sdk/reply-chunking` | Helpers étroits de fragmentation texte/Markdown |
    | `plugin-sdk/session-store-runtime` | Helpers de chemin de magasin de session + date de mise à jour |
    | `plugin-sdk/state-paths` | Helpers de chemin de répertoire d'état/OAuth |
    | `plugin-sdk/routing` | Helpers de route/clé de session/liaison de compte tels que `resolveAgentRoute`, `buildAgentSessionKey` et `resolveDefaultAgentBoundAccountId` |
    | `plugin-sdk/status-helpers` | Helpers partagés de résumé d'état de channel/compte, valeurs par défaut d'état d'exécution et helpers de métadonnées de problème |
    | `plugin-sdk/target-resolver-runtime` | Helpers partagés de résolution de cible |
    | `plugin-sdk/string-normalization-runtime` | Helpers de normalisation de slug/chaîne |
    | `plugin-sdk/request-url` | Extraire des URL chaîne depuis des entrées de type fetch/request |
    | `plugin-sdk/run-command` | Exécuteur de commande temporisé avec résultats stdout/stderr normalisés |
    | `plugin-sdk/param-readers` | Lecteurs de paramètres communs d'outil/CLI |
    | `plugin-sdk/tool-payload` | Extraire des charges utiles normalisées depuis des objets de résultat d'outil |
    | `plugin-sdk/tool-send` | Extraire des champs cibles d'envoi canoniques depuis des arguments d'outil |
    | `plugin-sdk/temp-path` | Helpers partagés de chemin de téléchargement temporaire |
    | `plugin-sdk/logging-core` | Logger de sous-système et helpers de masquage |
    | `plugin-sdk/markdown-table-runtime` | Helpers de mode tableau Markdown |
    | `plugin-sdk/json-store` | Petits helpers de lecture/écriture d'état JSON |
    | `plugin-sdk/file-lock` | Helpers de verrouillage de fichier réentrant |
    | `plugin-sdk/persistent-dedupe` | Helpers de cache de déduplication adossé au disque |
    | `plugin-sdk/acp-runtime` | Helpers ACP d'exécution/session et de répartition de réponse |
    | `plugin-sdk/agent-config-primitives` | Primitives étroites de schéma de configuration d'exécution d'agent |
    | `plugin-sdk/boolean-param` | Lecteur permissif de paramètre booléen |
    | `plugin-sdk/dangerous-name-runtime` | Helpers de résolution de correspondance de nom dangereux |
    | `plugin-sdk/device-bootstrap` | Helpers d'initialisation d'appareil et de jeton d'appairage |
    | `plugin-sdk/extension-shared` | Primitives helper partagées de channel passif, d'état et de proxy ambiant |
    | `plugin-sdk/models-provider-runtime` | Helpers de réponse de commande `/models`/provider |
    | `plugin-sdk/skill-commands-runtime` | Helpers de listage des commandes de Skills |
    | `plugin-sdk/native-command-registry` | Helpers de registre/build/sérialisation de commandes natives |
    | `plugin-sdk/agent-harness` | Surface expérimentale de plugin de confiance pour les harnais d'agent bas niveau : types de harnais, helpers de pilotage/abandon d'exécution active, helpers de pont d'outils OpenClaw et utilitaires de résultat de tentative |
    | `plugin-sdk/provider-zai-endpoint` | Helpers de détection de point de terminaison Z.AI |
    | `plugin-sdk/infra-runtime` | Helpers d'événement système/heartbeat |
    | `plugin-sdk/collection-runtime` | Petits helpers de cache borné |
    | `plugin-sdk/diagnostic-runtime` | Helpers d'indicateur et d'événement de diagnostic |
    | `plugin-sdk/error-runtime` | Graphe d'erreurs, formatage, helpers partagés de classification d'erreurs, `isApprovalNotFoundError` |
    | `plugin-sdk/fetch-runtime` | Helpers de fetch encapsulé, proxy et recherche épinglée |
    | `plugin-sdk/host-runtime` | Helpers de normalisation de nom d'hôte et d'hôte SCP |
    | `plugin-sdk/retry-runtime` | Helpers de configuration et d'exécuteur de retry |
    | `plugin-sdk/agent-runtime` | Helpers de répertoire/identité/espace de travail d'agent |
    | `plugin-sdk/directory-runtime` | Requête/déduplication de répertoire adossée à la configuration |
    | `plugin-sdk/keyed-async-queue` | `KeyedAsyncQueue` |
  </Accordion>

  <Accordion title="Sous-chemins de capacités et de tests">
    | Sous-chemin | Exports clés |
    | --- | --- |
    | `plugin-sdk/media-runtime` | Helpers partagés de récupération/transformation/stockage de médias plus constructeurs de charges utiles média |
    | `plugin-sdk/media-generation-runtime` | Helpers partagés de fallback de génération de médias, sélection de candidats et messagerie de modèle manquant |
    | `plugin-sdk/media-understanding` | Types de provider de compréhension des médias plus exports helper orientés provider pour image/audio |
    | `plugin-sdk/text-runtime` | Helpers partagés de texte/Markdown/journalisation tels que suppression du texte visible par l'assistant, helpers de rendu/fragmentation/tableau Markdown, helpers de masquage, helpers de balises de directive et utilitaires de texte sûr |
    | `plugin-sdk/text-chunking` | Helper de fragmentation de texte sortant |
    | `plugin-sdk/speech` | Types de provider de parole plus helpers orientés provider de directive, registre et validation |
    | `plugin-sdk/speech-core` | Helpers partagés de types de provider de parole, registre, directive et normalisation |
    | `plugin-sdk/realtime-transcription` | Types de provider de transcription en temps réel et helpers de registre |
    | `plugin-sdk/realtime-voice` | Types de provider de voix en temps réel et helpers de registre |
    | `plugin-sdk/image-generation` | Types de provider de génération d'image |
    | `plugin-sdk/image-generation-core` | Helpers partagés de types de génération d'image, fallback, authentification et registre |
    | `plugin-sdk/music-generation` | Types de provider/requête/résultat de génération musicale |
    | `plugin-sdk/music-generation-core` | Types partagés de génération musicale, helpers de fallback, recherche de provider et analyse de référence de modèle |
    | `plugin-sdk/video-generation` | Types de provider/requête/résultat de génération vidéo |
    | `plugin-sdk/video-generation-core` | Types partagés de génération vidéo, helpers de fallback, recherche de provider et analyse de référence de modèle |
    | `plugin-sdk/webhook-targets` | Registre de cibles webhook et helpers d'installation de route |
    | `plugin-sdk/webhook-path` | Helpers de normalisation de chemin webhook |
    | `plugin-sdk/web-media` | Helpers partagés de chargement de médias distants/locaux |
    | `plugin-sdk/zod` | `zod` réexporté pour les consommateurs du SDK de plugin |
    | `plugin-sdk/testing` | `installCommonResolveTargetErrorCases`, `shouldAckReaction` |
  </Accordion>

  <Accordion title="Sous-chemins de mémoire">
    | Sous-chemin | Exports clés |
    | --- | --- |
    | `plugin-sdk/memory-core` | Surface helper memory-core intégrée pour les helpers de gestionnaire/configuration/fichier/CLI |
    | `plugin-sdk/memory-core-engine-runtime` | Façade d'exécution d'indexation/recherche de mémoire |
    | `plugin-sdk/memory-core-host-engine-foundation` | Exports du moteur de fondation de l'hôte mémoire |
    | `plugin-sdk/memory-core-host-engine-embeddings` | Exports du moteur d'embeddings de l'hôte mémoire |
    | `plugin-sdk/memory-core-host-engine-qmd` | Exports du moteur QMD de l'hôte mémoire |
    | `plugin-sdk/memory-core-host-engine-storage` | Exports du moteur de stockage de l'hôte mémoire |
    | `plugin-sdk/memory-core-host-multimodal` | Helpers multimodaux de l'hôte mémoire |
    | `plugin-sdk/memory-core-host-query` | Helpers de requête de l'hôte mémoire |
    | `plugin-sdk/memory-core-host-secret` | Helpers de secret de l'hôte mémoire |
    | `plugin-sdk/memory-core-host-events` | Helpers de journal d'événements de l'hôte mémoire |
    | `plugin-sdk/memory-core-host-status` | Helpers d'état de l'hôte mémoire |
    | `plugin-sdk/memory-core-host-runtime-cli` | Helpers d'exécution CLI de l'hôte mémoire |
    | `plugin-sdk/memory-core-host-runtime-core` | Helpers d'exécution cœur de l'hôte mémoire |
    | `plugin-sdk/memory-core-host-runtime-files` | Helpers de fichier/exécution de l'hôte mémoire |
    | `plugin-sdk/memory-host-core` | Alias neutre vis-à-vis du fournisseur pour les helpers d'exécution cœur de l'hôte mémoire |
    | `plugin-sdk/memory-host-events` | Alias neutre vis-à-vis du fournisseur pour les helpers de journal d'événements de l'hôte mémoire |
    | `plugin-sdk/memory-host-files` | Alias neutre vis-à-vis du fournisseur pour les helpers de fichier/exécution de l'hôte mémoire |
    | `plugin-sdk/memory-host-markdown` | Helpers partagés de Markdown géré pour les plugins proches de la mémoire |
    | `plugin-sdk/memory-host-search` | Façade d'exécution de mémoire active pour l'accès au gestionnaire de recherche |
    | `plugin-sdk/memory-host-status` | Alias neutre vis-à-vis du fournisseur pour les helpers d'état de l'hôte mémoire |
    | `plugin-sdk/memory-lancedb` | Surface helper memory-lancedb intégrée |
  </Accordion>

  <Accordion title="Sous-chemins helper intégrés réservés">
    | Famille | Sous-chemins actuels | Utilisation prévue |
    | --- | --- | --- |
    | Browser | `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support` | Helpers de prise en charge du plugin browser intégré (`browser-support` reste le baril de compatibilité) |
    | Matrix | `plugin-sdk/matrix`, `plugin-sdk/matrix-helper`, `plugin-sdk/matrix-runtime-heavy`, `plugin-sdk/matrix-runtime-shared`, `plugin-sdk/matrix-runtime-surface`, `plugin-sdk/matrix-surface`, `plugin-sdk/matrix-thread-bindings` | Surface helper/d'exécution Matrix intégrée |
    | Line | `plugin-sdk/line`, `plugin-sdk/line-core`, `plugin-sdk/line-runtime`, `plugin-sdk/line-surface` | Surface helper/d'exécution LINE intégrée |
    | IRC | `plugin-sdk/irc`, `plugin-sdk/irc-surface` | Surface helper IRC intégrée |
    | Helpers spécifiques au channel | `plugin-sdk/googlechat`, `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles`, `plugin-sdk/bluebubbles-policy`, `plugin-sdk/mattermost`, `plugin-sdk/mattermost-policy`, `plugin-sdk/feishu-conversation`, `plugin-sdk/msteams`, `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`, `plugin-sdk/twitch` | Surfaces de compatibilité/helper de channel intégrées |
    | Helpers spécifiques à auth/plugin | `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`, `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`, `plugin-sdk/thread-ownership`, `plugin-sdk/voice-call` | Surfaces helper de fonctionnalité/plugin intégrées ; `plugin-sdk/github-copilot-token` exporte actuellement `DEFAULT_COPILOT_API_BASE_URL`, `deriveCopilotApiBaseUrlFromToken` et `resolveCopilotApiToken` |
  </Accordion>
</AccordionGroup>

## API d'enregistrement

Le callback `register(api)` reçoit un objet `OpenClawPluginApi` avec ces
méthodes :

### Enregistrement de capacités

| Méthode                                          | Ce qu'elle enregistre                 |
| ------------------------------------------------ | ------------------------------------- |
| `api.registerProvider(...)`                      | Inférence de texte (LLM)              |
| `api.registerAgentHarness(...)`                  | Exécuteur d'agent bas niveau expérimental |
| `api.registerCliBackend(...)`                    | Backend CLI d'inférence locale        |
| `api.registerChannel(...)`                       | Channel de messagerie                 |
| `api.registerSpeechProvider(...)`                | Synthèse texte-vers-parole / STT      |
| `api.registerRealtimeTranscriptionProvider(...)` | Transcription temps réel en streaming |
| `api.registerRealtimeVoiceProvider(...)`         | Sessions vocales temps réel duplex    |
| `api.registerMediaUnderstandingProvider(...)`    | Analyse d'image/audio/vidéo           |
| `api.registerImageGenerationProvider(...)`       | Génération d'image                    |
| `api.registerMusicGenerationProvider(...)`       | Génération musicale                   |
| `api.registerVideoGenerationProvider(...)`       | Génération vidéo                      |
| `api.registerWebFetchProvider(...)`              | Fournisseur de récupération / scraping web |
| `api.registerWebSearchProvider(...)`             | Recherche web                         |

### Outils et commandes

| Méthode                         | Ce qu'elle enregistre                         |
| ------------------------------- | --------------------------------------------- |
| `api.registerTool(tool, opts?)` | Outil d'agent (obligatoire ou `{ optional: true }`) |
| `api.registerCommand(def)`      | Commande personnalisée (contourne le LLM)     |

### Infrastructure

| Méthode                                        | Ce qu'elle enregistre                |
| ---------------------------------------------- | ------------------------------------ |
| `api.registerHook(events, handler, opts?)`     | Hook d'événement                     |
| `api.registerHttpRoute(params)`                | Point de terminaison HTTP de Gateway |
| `api.registerGatewayMethod(name, handler)`     | Méthode RPC de Gateway               |
| `api.registerCli(registrar, opts?)`            | Sous-commande CLI                    |
| `api.registerService(service)`                 | Service d'arrière-plan               |
| `api.registerInteractiveHandler(registration)` | Gestionnaire interactif              |
| `api.registerMemoryPromptSupplement(builder)`  | Section additive d'invite proche de la mémoire |
| `api.registerMemoryCorpusSupplement(adapter)`  | Corpus additif de recherche/lecture mémoire |

Les espaces de noms d'administration du cœur réservés (`config.*`, `exec.approvals.*`, `wizard.*`,
`update.*`) restent toujours `operator.admin`, même si un plugin essaie d'assigner une
portée de méthode gateway plus étroite. Préférez des préfixes spécifiques au plugin pour
les méthodes appartenant au plugin.

### Métadonnées d'enregistrement CLI

`api.registerCli(registrar, opts?)` accepte deux types de métadonnées de niveau supérieur :

- `commands` : racines de commandes explicites appartenant au registrar
- `descriptors` : descripteurs de commande à l'analyse utilisés pour l'aide de la CLI racine,
  le routage et l'enregistrement paresseux de la CLI du plugin

Si vous voulez qu'une commande de plugin reste chargée paresseusement dans le chemin CLI racine normal,
fournissez des `descriptors` qui couvrent chaque racine de commande de niveau supérieur exposée par ce
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

Utilisez `commands` seul uniquement lorsque vous n'avez pas besoin d'un enregistrement CLI racine paresseux.
Ce chemin de compatibilité eager reste pris en charge, mais il n'installe pas de
placeholders soutenus par des descripteurs pour le chargement paresseux au moment de l'analyse.

### Enregistrement de backend CLI

`api.registerCliBackend(...)` permet à un plugin de posséder la configuration par défaut d'un backend CLI
d'IA local tel que `codex-cli`.

- Le `id` du backend devient le préfixe provider dans des références de modèle comme `codex-cli/gpt-5`.
- La `config` du backend utilise la même structure que `agents.defaults.cliBackends.<id>`.
- La configuration utilisateur garde la priorité. OpenClaw fusionne `agents.defaults.cliBackends.<id>` au-dessus de la
  valeur par défaut du plugin avant d'exécuter la CLI.
- Utilisez `normalizeConfig` lorsqu'un backend a besoin de réécritures de compatibilité après fusion
  (par exemple normaliser d'anciennes formes de drapeaux).

### Emplacements exclusifs

| Méthode                                    | Ce qu'elle enregistre                                                                                                                                   |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `api.registerContextEngine(id, factory)`   | Moteur de contexte (un seul actif à la fois). Le callback `assemble()` reçoit `availableTools` et `citationsMode` afin que le moteur puisse adapter les ajouts à l'invite. |
| `api.registerMemoryCapability(capability)` | Capacité mémoire unifiée                                                                                                                                 |
| `api.registerMemoryPromptSection(builder)` | Constructeur de section d'invite mémoire                                                                                                                |
| `api.registerMemoryFlushPlan(resolver)`    | Résolveur de plan de vidage mémoire                                                                                                                     |
| `api.registerMemoryRuntime(runtime)`       | Adaptateur d'exécution mémoire                                                                                                                          |

### Adaptateurs d'embeddings mémoire

| Méthode                                        | Ce qu'elle enregistre                         |
| ---------------------------------------------- | --------------------------------------------- |
| `api.registerMemoryEmbeddingProvider(adapter)` | Adaptateur d'embeddings mémoire pour le plugin actif |

- `registerMemoryCapability` est l'API exclusive préférée pour les plugins mémoire.
- `registerMemoryCapability` peut aussi exposer `publicArtifacts.listArtifacts(...)`
  afin que les plugins compagnons puissent consommer les artefacts mémoire exportés via
  `openclaw/plugin-sdk/memory-host-core` au lieu d'accéder à la disposition privée d'un
  plugin mémoire spécifique.
- `registerMemoryPromptSection`, `registerMemoryFlushPlan` et
  `registerMemoryRuntime` sont des API exclusives de plugin mémoire compatibles avec l'historique.
- `registerMemoryEmbeddingProvider` permet au plugin mémoire actif d'enregistrer un
  ou plusieurs identifiants d'adaptateur d'embeddings (par exemple `openai`, `gemini`, ou un identifiant défini par un plugin personnalisé).
- La configuration utilisateur telle que `agents.defaults.memorySearch.provider` et
  `agents.defaults.memorySearch.fallback` se résout par rapport à ces identifiants
  d'adaptateur enregistrés.

### Événements et cycle de vie

| Méthode                                      | Ce qu'elle fait                |
| -------------------------------------------- | ------------------------------ |
| `api.on(hookName, handler, opts?)`           | Hook de cycle de vie typé      |
| `api.onConversationBindingResolved(handler)` | Callback de liaison de conversation |

### Sémantique de décision des hooks

- `before_tool_call` : renvoyer `{ block: true }` est terminal. Dès qu'un gestionnaire le définit, les gestionnaires de priorité inférieure sont ignorés.
- `before_tool_call` : renvoyer `{ block: false }` est traité comme aucune décision (comme si `block` était omis), pas comme une surcharge.
- `before_install` : renvoyer `{ block: true }` est terminal. Dès qu'un gestionnaire le définit, les gestionnaires de priorité inférieure sont ignorés.
- `before_install` : renvoyer `{ block: false }` est traité comme aucune décision (comme si `block` était omis), pas comme une surcharge.
- `reply_dispatch` : renvoyer `{ handled: true, ... }` est terminal. Dès qu'un gestionnaire revendique la répartition, les gestionnaires de priorité inférieure et le chemin de répartition de modèle par défaut sont ignorés.
- `message_sending` : renvoyer `{ cancel: true }` est terminal. Dès qu'un gestionnaire le définit, les gestionnaires de priorité inférieure sont ignorés.
- `message_sending` : renvoyer `{ cancel: false }` est traité comme aucune décision (comme si `cancel` était omis), pas comme une surcharge.

### Champs de l'objet API

| Champ                   | Type                      | Description                                                                                |
| ----------------------- | ------------------------- | ------------------------------------------------------------------------------------------ |
| `api.id`                | `string`                  | Identifiant du plugin                                                                      |
| `api.name`              | `string`                  | Nom d'affichage                                                                            |
| `api.version`           | `string?`                 | Version du plugin (facultatif)                                                             |
| `api.description`       | `string?`                 | Description du plugin (facultatif)                                                         |
| `api.source`            | `string`                  | Chemin source du plugin                                                                    |
| `api.rootDir`           | `string?`                 | Répertoire racine du plugin (facultatif)                                                   |
| `api.config`            | `OpenClawConfig`          | Instantané de configuration actuel (instantané d'exécution actif en mémoire lorsqu'il est disponible) |
| `api.pluginConfig`      | `Record<string, unknown>` | Configuration spécifique au plugin depuis `plugins.entries.<id>.config`                    |
| `api.runtime`           | `PluginRuntime`           | [Helpers d'exécution](/fr/plugins/sdk-runtime)                                                |
| `api.logger`            | `PluginLogger`            | Logger limité par portée (`debug`, `info`, `warn`, `error`)                                |
| `api.registrationMode`  | `PluginRegistrationMode`  | Mode de chargement actuel ; `"setup-runtime"` est la fenêtre légère de démarrage/configuration avant l'entrée complète |
| `api.resolvePath(input)` | `(string) => string`     | Résoudre un chemin relatif à la racine du plugin                                           |

## Convention de module interne

Dans votre plugin, utilisez des fichiers barils locaux pour les imports internes :

```
my-plugin/
  api.ts            # Exports publics pour les consommateurs externes
  runtime-api.ts    # Exports d'exécution internes uniquement
  index.ts          # Point d'entrée du plugin
  setup-entry.ts    # Entrée légère réservée à la configuration (facultative)
```

<Warning>
  N'importez jamais votre propre plugin via `openclaw/plugin-sdk/<your-plugin>`
  depuis le code de production. Faites passer les imports internes via `./api.ts` ou
  `./runtime-api.ts`. Le chemin SDK est uniquement le contrat externe.
</Warning>

Les surfaces publiques de plugin intégré chargées via façade (`api.ts`, `runtime-api.ts`,
`index.ts`, `setup-entry.ts` et autres fichiers d'entrée publics similaires) préfèrent désormais l'instantané
de configuration d'exécution actif lorsque OpenClaw est déjà en cours d'exécution. Si aucun instantané d'exécution
n'existe encore, elles se replient sur le fichier de configuration résolu sur le disque.

Les plugins de provider peuvent aussi exposer un baril de contrat local au plugin et étroit lorsqu'un
helper est intentionnellement spécifique au provider et n'a pas encore sa place dans un
sous-chemin SDK générique. Exemple intégré actuel : le provider Anthropic conserve ses helpers de flux Claude
dans sa propre surface publique `api.ts` / `contract-api.ts` au lieu de promouvoir la logique
d'en-tête bêta Anthropic et `service_tier` dans un contrat générique
`plugin-sdk/*`.

Autres exemples intégrés actuels :

- `@openclaw/openai-provider` : `api.ts` exporte des constructeurs de provider,
  des helpers de modèle par défaut et des constructeurs de provider temps réel
- `@openclaw/openrouter-provider` : `api.ts` exporte le constructeur de provider ainsi que
  des helpers d'onboarding/configuration

<Warning>
  Le code de production d'extension doit aussi éviter les imports `openclaw/plugin-sdk/<other-plugin>`.
  Si un helper est réellement partagé, promouvez-le vers un sous-chemin SDK neutre
  tel que `openclaw/plugin-sdk/speech`, `.../provider-model-shared`, ou une autre
  surface orientée capacité au lieu de coupler deux plugins ensemble.
</Warning>

## Voir aussi

- [Points d'entrée](/fr/plugins/sdk-entrypoints) — options `definePluginEntry` et `defineChannelPluginEntry`
- [Helpers d'exécution](/fr/plugins/sdk-runtime) — référence complète de l'espace de noms `api.runtime`
- [Configuration et config](/fr/plugins/sdk-setup) — packaging, manifestes, schémas de configuration
- [Tests](/fr/plugins/sdk-testing) — utilitaires de test et règles de lint
- [Migration du SDK](/fr/plugins/sdk-migration) — migration depuis les surfaces obsolètes
- [Internes des plugins](/fr/plugins/architecture) — architecture approfondie et modèle de capacité
