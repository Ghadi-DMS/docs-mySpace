---
read_when:
    - Vous devez savoir depuis quel sous-chemin du SDK importer
    - Vous voulez une référence pour toutes les méthodes d’enregistrement sur OpenClawPluginApi
    - Vous recherchez une exportation spécifique du SDK
sidebarTitle: SDK Overview
summary: Map d’import, référence de l’API d’enregistrement et architecture du SDK
title: Aperçu du SDK Plugin
x-i18n:
    generated_at: "2026-04-18T06:44:10Z"
    model: gpt-5.4
    provider: openai
    source_hash: 05d3d0022cca32d29c76f6cea01cdf4f88ac69ef0ef3d7fb8a60fbf9a6b9b331
    source_path: plugins/sdk-overview.md
    workflow: 15
---

# Aperçu du SDK Plugin

Le SDK Plugin est le contrat typé entre les plugins et le cœur. Cette page est la
référence pour **quoi importer** et **ce que vous pouvez enregistrer**.

<Tip>
  **Vous cherchez un guide pratique ?**
  - Premier plugin ? Commencez par [Prise en main](/fr/plugins/building-plugins)
  - Plugin de canal ? Voir [Plugins de canal](/fr/plugins/sdk-channel-plugins)
  - Plugin de fournisseur ? Voir [Plugins de fournisseur](/fr/plugins/sdk-provider-plugins)
</Tip>

## Convention d’import

Importez toujours depuis un sous-chemin spécifique :

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
```

Chaque sous-chemin est un petit module autonome. Cela permet de garder un
démarrage rapide et d’éviter les problèmes de dépendances circulaires. Pour les helpers
d’entrée/de construction spécifiques aux canaux, privilégiez `openclaw/plugin-sdk/channel-core` ; réservez `openclaw/plugin-sdk/core` à
la surface ombrelle plus large et aux helpers partagés comme
`buildChannelConfigSchema`.

N’ajoutez pas et ne dépendez pas de points d’entrée de commodité nommés d’après des fournisseurs tels que
`openclaw/plugin-sdk/slack`, `openclaw/plugin-sdk/discord`,
`openclaw/plugin-sdk/signal`, `openclaw/plugin-sdk/whatsapp`, ni de
points d’entrée helpers marqués canal. Les plugins intégrés doivent composer des
sous-chemins génériques du SDK dans leurs propres barrels `api.ts` ou `runtime-api.ts`, et le cœur
doit soit utiliser ces barrels locaux au plugin, soit ajouter un contrat SDK
générique étroit lorsque le besoin est réellement transversal aux canaux.

La map d’export générée contient encore un petit ensemble de points d’entrée helpers de plugins intégrés
tels que `plugin-sdk/feishu`, `plugin-sdk/feishu-setup`,
`plugin-sdk/zalo`, `plugin-sdk/zalo-setup` et `plugin-sdk/matrix*`. Ces
sous-chemins existent uniquement pour la maintenance et la compatibilité des plugins intégrés ; ils sont
volontairement omis du tableau courant ci-dessous et ne constituent pas le
chemin d’import recommandé pour les nouveaux plugins tiers.

## Référence des sous-chemins

Les sous-chemins les plus couramment utilisés, regroupés par usage. La liste complète générée de
plus de 200 sous-chemins se trouve dans `scripts/lib/plugin-sdk-entrypoints.json`.

Les sous-chemins helpers réservés aux plugins intégrés figurent toujours dans cette liste générée.
Traitez-les comme des surfaces de détail d’implémentation/de compatibilité, sauf si une page de documentation
en promeut explicitement un comme public.

### Entrée de plugin

| Sous-chemin                 | Exportations principales                                                                                                               |
| --------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/plugin-entry`   | `definePluginEntry`                                                                                                                    |
| `plugin-sdk/core`           | `defineChannelPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase`, `defineSetupPluginEntry`, `buildChannelConfigSchema` |
| `plugin-sdk/config-schema`  | `OpenClawSchema`                                                                                                                       |
| `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry`                                                                                                      |

<AccordionGroup>
  <Accordion title="Sous-chemins de canal">
    | Sous-chemin | Exportations principales |
    | --- | --- |
    | `plugin-sdk/channel-core` | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
    | `plugin-sdk/config-schema` | Export Zod du schéma racine `openclaw.json` (`OpenClawSchema`) |
    | `plugin-sdk/channel-setup` | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, plus `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
    | `plugin-sdk/setup` | Helpers partagés d’assistant de configuration, invites d’allowlist, constructeurs d’état de configuration |
    | `plugin-sdk/setup-runtime` | `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
    | `plugin-sdk/setup-adapter-runtime` | `createEnvPatchedAccountSetupAdapter` |
    | `plugin-sdk/setup-tools` | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
    | `plugin-sdk/account-core` | Helpers de configuration multi-comptes/gate d’action, helpers de repli de compte par défaut |
    | `plugin-sdk/account-id` | `DEFAULT_ACCOUNT_ID`, helpers de normalisation d’identifiant de compte |
    | `plugin-sdk/account-resolution` | Recherche de compte + helpers de repli par défaut |
    | `plugin-sdk/account-helpers` | Helpers ciblés de liste de comptes/action de compte |
    | `plugin-sdk/channel-pairing` | `createChannelPairingController` |
    | `plugin-sdk/channel-reply-pipeline` | `createChannelReplyPipeline` |
    | `plugin-sdk/channel-config-helpers` | `createHybridChannelConfigAdapter` |
    | `plugin-sdk/channel-config-schema` | Types de schéma de configuration de canal |
    | `plugin-sdk/telegram-command-config` | Helpers de normalisation/validation des commandes personnalisées Telegram avec repli de contrat intégré |
    | `plugin-sdk/command-gating` | Helpers ciblés de gate d’autorisation de commande |
    | `plugin-sdk/channel-policy` | `resolveChannelGroupRequireMention` |
    | `plugin-sdk/channel-lifecycle` | `createAccountStatusSink` |
    | `plugin-sdk/inbound-envelope` | Helpers partagés de route entrante + construction d’enveloppe |
    | `plugin-sdk/inbound-reply-dispatch` | Helpers partagés d’enregistrement entrant et de distribution |
    | `plugin-sdk/messaging-targets` | Helpers d’analyse/de correspondance de cibles |
    | `plugin-sdk/outbound-media` | Helpers partagés de chargement de médias sortants |
    | `plugin-sdk/outbound-runtime` | Helpers de délégation d’envoi/d’identité sortante |
    | `plugin-sdk/poll-runtime` | Helpers ciblés de normalisation de sondage |
    | `plugin-sdk/thread-bindings-runtime` | Helpers de cycle de vie et d’adaptateur des liaisons de thread |
    | `plugin-sdk/agent-media-payload` | Constructeur hérité de payload média d’agent |
    | `plugin-sdk/conversation-runtime` | Helpers de liaison de conversation/thread, d’appairage et de liaison configurée |
    | `plugin-sdk/runtime-config-snapshot` | Helper d’instantané de configuration d’exécution |
    | `plugin-sdk/runtime-group-policy` | Helpers d’exécution de résolution de politique de groupe |
    | `plugin-sdk/channel-status` | Helpers partagés d’instantané/résumé d’état de canal |
    | `plugin-sdk/channel-config-primitives` | Primitives ciblées de schéma de configuration de canal |
    | `plugin-sdk/channel-config-writes` | Helpers d’autorisation d’écriture de configuration de canal |
    | `plugin-sdk/channel-plugin-common` | Exports de prélude partagés de plugin de canal |
    | `plugin-sdk/allowlist-config-edit` | Helpers de lecture/édition de configuration d’allowlist |
    | `plugin-sdk/group-access` | Helpers partagés de décision d’accès de groupe |
    | `plugin-sdk/direct-dm` | Helpers partagés d’authentification/garde de message direct |
    | `plugin-sdk/interactive-runtime` | Helpers de normalisation/réduction de payload de réponse interactive |
    | `plugin-sdk/channel-inbound` | Barrel de compatibilité pour le debounce entrant, la correspondance de mention, les helpers de politique de mention et les helpers d’enveloppe |
    | `plugin-sdk/channel-mention-gating` | Helpers ciblés de politique de mention sans la surface d’exécution entrante plus large |
    | `plugin-sdk/channel-location` | Helpers de contexte et de formatage d’emplacement de canal |
    | `plugin-sdk/channel-logging` | Helpers de journalisation de canal pour les rejets entrants et les échecs de saisie/d’accusé de réception |
    | `plugin-sdk/channel-send-result` | Types de résultat de réponse |
    | `plugin-sdk/channel-actions` | `createMessageToolButtonsSchema`, `createMessageToolCardSchema` |
    | `plugin-sdk/channel-targets` | Helpers d’analyse/de correspondance de cibles |
    | `plugin-sdk/channel-contract` | Types de contrat de canal |
    | `plugin-sdk/channel-feedback` | Câblage des retours/réactions |
    | `plugin-sdk/channel-secret-runtime` | Helpers ciblés de contrat de secret comme `collectSimpleChannelFieldAssignments`, `getChannelSurface`, `pushAssignment`, et types de cible de secret |
  </Accordion>

  <Accordion title="Sous-chemins de fournisseur">
    | Sous-chemin | Exportations principales |
    | --- | --- |
    | `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry` |
    | `plugin-sdk/provider-setup` | Helpers de configuration sélectionnés pour les fournisseurs locaux/autohébergés |
    | `plugin-sdk/self-hosted-provider-setup` | Helpers ciblés de configuration de fournisseur autohébergé compatible OpenAI |
    | `plugin-sdk/cli-backend` | Valeurs par défaut du backend CLI + constantes de watchdog |
    | `plugin-sdk/provider-auth-runtime` | Helpers d’exécution de résolution de clé API pour les plugins de fournisseur |
    | `plugin-sdk/provider-auth-api-key` | Helpers d’intégration/écriture de profil de clé API tels que `upsertApiKeyProfile` |
    | `plugin-sdk/provider-auth-result` | Constructeur standard de résultat d’authentification OAuth |
    | `plugin-sdk/provider-auth-login` | Helpers partagés de connexion interactive pour les plugins de fournisseur |
    | `plugin-sdk/provider-env-vars` | Helpers de recherche de variables d’environnement d’authentification fournisseur |
    | `plugin-sdk/provider-auth` | `createProviderApiKeyAuthMethod`, `ensureApiKeyFromOptionEnvOrPrompt`, `upsertAuthProfile`, `upsertApiKeyProfile`, `writeOAuthCredentials` |
    | `plugin-sdk/provider-model-shared` | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, constructeurs partagés de politique de rejeu, helpers de point de terminaison fournisseur et helpers de normalisation d’identifiant de modèle comme `normalizeNativeXaiModelId` |
    | `plugin-sdk/provider-catalog-shared` | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
    | `plugin-sdk/provider-http` | Helpers génériques de capacité HTTP/point de terminaison fournisseur |
    | `plugin-sdk/provider-web-fetch-contract` | Helpers ciblés de contrat de configuration/sélection web-fetch comme `enablePluginInConfig` et `WebFetchProviderPlugin` |
    | `plugin-sdk/provider-web-fetch` | Helpers d’enregistrement/cache de fournisseur web-fetch |
    | `plugin-sdk/provider-web-search-config-contract` | Helpers ciblés de configuration/d’identifiants web-search pour les fournisseurs qui n’ont pas besoin de câblage d’activation de plugin |
    | `plugin-sdk/provider-web-search-contract` | Helpers ciblés de contrat de configuration/d’identifiants web-search comme `createWebSearchProviderContractFields`, `enablePluginInConfig`, `resolveProviderWebSearchPluginConfig`, et setters/getters d’identifiants à portée limitée |
    | `plugin-sdk/provider-web-search` | Helpers d’enregistrement/cache/exécution de fournisseur web-search |
    | `plugin-sdk/provider-tools` | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, nettoyage de schéma Gemini + diagnostics, et helpers de compatibilité xAI comme `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
    | `plugin-sdk/provider-usage` | `fetchClaudeUsage` et similaires |
    | `plugin-sdk/provider-stream` | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, types de wrappers de flux, et helpers partagés de wrapper Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot |
    | `plugin-sdk/provider-onboard` | Helpers de correctif de configuration d’intégration |
    | `plugin-sdk/global-singleton` | Helpers de singleton/map/cache locaux au processus |
  </Accordion>

  <Accordion title="Sous-chemins d’authentification et de sécurité">
    | Sous-chemin | Exportations principales |
    | --- | --- |
    | `plugin-sdk/command-auth` | `resolveControlCommandGate`, helpers de registre de commandes, helpers d’autorisation d’expéditeur |
    | `plugin-sdk/command-status` | Constructeurs de messages de commande/d’aide comme `buildCommandsMessagePaginated` et `buildHelpMessage` |
    | `plugin-sdk/approval-auth-runtime` | Helpers de résolution d’approbateur et d’authentification d’action dans le même chat |
    | `plugin-sdk/approval-client-runtime` | Helpers de profil/filtre d’approbation exec natifs |
    | `plugin-sdk/approval-delivery-runtime` | Adaptateurs natifs de capacité/de livraison d’approbation |
    | `plugin-sdk/approval-gateway-runtime` | Helper partagé de résolution de Gateway d’approbation |
    | `plugin-sdk/approval-handler-adapter-runtime` | Helpers légers de chargement d’adaptateur d’approbation natif pour points d’entrée de canal à chaud |
    | `plugin-sdk/approval-handler-runtime` | Helpers d’exécution plus larges de gestionnaire d’approbation ; privilégiez les points d’entrée plus étroits adapter/gateway lorsqu’ils suffisent |
    | `plugin-sdk/approval-native-runtime` | Helpers de cible d’approbation native + de liaison de compte |
    | `plugin-sdk/approval-reply-runtime` | Helpers de payload de réponse d’approbation exec/plugin |
    | `plugin-sdk/command-auth-native` | Authentification de commande native + helpers natifs de cible de session |
    | `plugin-sdk/command-detection` | Helpers partagés de détection de commandes |
    | `plugin-sdk/command-surface` | Helpers de normalisation du corps de commande et de surface de commande |
    | `plugin-sdk/allow-from` | `formatAllowFromLowercase` |
    | `plugin-sdk/channel-secret-runtime` | Helpers ciblés de collecte de contrat de secret pour surfaces de secrets de canal/plugin |
    | `plugin-sdk/secret-ref-runtime` | Helpers ciblés `coerceSecretRef` et de typage SecretRef pour l’analyse des contrats de secret/de la configuration |
    | `plugin-sdk/security-runtime` | Helpers partagés de confiance, gate DM, contenu externe et collecte de secrets |
    | `plugin-sdk/ssrf-policy` | Helpers de politique SSRF de liste d’autorisation d’hôtes et de réseau privé |
    | `plugin-sdk/ssrf-dispatcher` | Helpers ciblés de dispatcher épinglé sans la large surface d’exécution infra |
    | `plugin-sdk/ssrf-runtime` | Helpers de dispatcher épinglé, fetch protégé contre SSRF et politique SSRF |
    | `plugin-sdk/secret-input` | Helpers d’analyse d’entrée de secret |
    | `plugin-sdk/webhook-ingress` | Helpers de requête/cible Webhook |
    | `plugin-sdk/webhook-request-guards` | Helpers de taille de corps de requête/délai d’expiration |
  </Accordion>

  <Accordion title="Sous-chemins d’exécution et de stockage">
    | Sous-chemin | Exportations principales |
    | --- | --- |
    | `plugin-sdk/runtime` | Helpers larges d’exécution/journalisation/sauvegarde/installation de plugin |
    | `plugin-sdk/runtime-env` | Helpers ciblés d’environnement d’exécution, logger, délai d’expiration, nouvelle tentative et backoff |
    | `plugin-sdk/channel-runtime-context` | Helpers génériques d’enregistrement et de recherche de contexte d’exécution de canal |
    | `plugin-sdk/runtime-store` | `createPluginRuntimeStore` |
    | `plugin-sdk/plugin-runtime` | Helpers partagés de commande/hook/http/interaction de plugin |
    | `plugin-sdk/hook-runtime` | Helpers partagés de pipeline de Webhook/hook interne |
    | `plugin-sdk/lazy-runtime` | Helpers d’import/liaison d’exécution paresseuse comme `createLazyRuntimeModule`, `createLazyRuntimeMethod` et `createLazyRuntimeSurface` |
    | `plugin-sdk/process-runtime` | Helpers d’exécution de processus |
    | `plugin-sdk/cli-runtime` | Helpers de formatage, d’attente et de version pour la CLI |
    | `plugin-sdk/gateway-runtime` | Helpers de client Gateway et de correctif d’état de canal |
    | `plugin-sdk/config-runtime` | Helpers de chargement/écriture de configuration |
    | `plugin-sdk/telegram-command-config` | Normalisation des noms/descriptions de commandes Telegram et vérifications des doublons/conflits, même lorsque la surface de contrat Telegram intégrée n’est pas disponible |
    | `plugin-sdk/text-autolink-runtime` | Détection d’autolien de référence de fichier sans le large barrel `text-runtime` |
    | `plugin-sdk/approval-runtime` | Helpers d’approbation exec/plugin, constructeurs de capacité d’approbation, helpers d’authentification/de profil, helpers de routage/d’exécution natifs |
    | `plugin-sdk/reply-runtime` | Helpers partagés d’exécution entrante/de réponse, découpage en blocs, distribution, Heartbeat, planificateur de réponse |
    | `plugin-sdk/reply-dispatch-runtime` | Helpers ciblés de distribution/finalisation de réponse |
    | `plugin-sdk/reply-history` | Helpers partagés d’historique de réponse sur courte fenêtre comme `buildHistoryContext`, `recordPendingHistoryEntry` et `clearHistoryEntriesIfEnabled` |
    | `plugin-sdk/reply-reference` | `createReplyReferencePlanner` |
    | `plugin-sdk/reply-chunking` | Helpers ciblés de découpage de texte/Markdown |
    | `plugin-sdk/session-store-runtime` | Helpers de chemin de stockage de session + `updated-at` |
    | `plugin-sdk/state-paths` | Helpers de chemins de répertoire State/OAuth |
    | `plugin-sdk/routing` | Helpers de route/clé de session/liaison de compte comme `resolveAgentRoute`, `buildAgentSessionKey` et `resolveDefaultAgentBoundAccountId` |
    | `plugin-sdk/status-helpers` | Helpers partagés de résumé d’état de canal/compte, valeurs par défaut d’état d’exécution et helpers de métadonnées de problème |
    | `plugin-sdk/target-resolver-runtime` | Helpers partagés de résolution de cible |
    | `plugin-sdk/string-normalization-runtime` | Helpers de normalisation de slug/chaîne |
    | `plugin-sdk/request-url` | Extrait des URL chaîne depuis des entrées de type fetch/request |
    | `plugin-sdk/run-command` | Exécuteur de commande temporisé avec résultats stdout/stderr normalisés |
    | `plugin-sdk/param-readers` | Lecteurs de paramètres communs d’outil/CLI |
    | `plugin-sdk/tool-payload` | Extrait des payloads normalisés depuis des objets de résultat d’outil |
    | `plugin-sdk/tool-send` | Extrait les champs de cible d’envoi canoniques depuis les arguments d’outil |
    | `plugin-sdk/temp-path` | Helpers partagés de chemin de téléchargement temporaire |
    | `plugin-sdk/logging-core` | Helpers de logger de sous-système et de masquage |
    | `plugin-sdk/markdown-table-runtime` | Helpers de mode de tableau Markdown |
    | `plugin-sdk/json-store` | Petits helpers de lecture/écriture d’état JSON |
    | `plugin-sdk/file-lock` | Helpers de verrouillage de fichier réentrants |
    | `plugin-sdk/persistent-dedupe` | Helpers de cache de déduplication adossé au disque |
    | `plugin-sdk/acp-runtime` | Helpers ACP d’exécution/session et de distribution de réponse |
    | `plugin-sdk/acp-binding-resolve-runtime` | Résolution en lecture seule des liaisons ACP sans imports de démarrage du cycle de vie |
    | `plugin-sdk/agent-config-primitives` | Primitives ciblées de schéma de configuration d’exécution d’agent |
    | `plugin-sdk/boolean-param` | Lecteur permissif de paramètre booléen |
    | `plugin-sdk/dangerous-name-runtime` | Helpers de résolution de correspondance de noms dangereux |
    | `plugin-sdk/device-bootstrap` | Helpers de bootstrap d’appareil et de jeton d’appairage |
    | `plugin-sdk/extension-shared` | Primitives helpers partagées de canal passif, statut et proxy ambiant |
    | `plugin-sdk/models-provider-runtime` | Helpers de réponse de commande `/models`/fournisseur |
    | `plugin-sdk/skill-commands-runtime` | Helpers de listage des commandes Skills |
    | `plugin-sdk/native-command-registry` | Helpers natifs de registre/construction/sérialisation de commandes |
    | `plugin-sdk/agent-harness` | Surface expérimentale de plugin de confiance pour harnais d’agent de bas niveau : types de harnais, helpers de pilotage/abandon d’exécution active, helpers de pont d’outil OpenClaw et utilitaires de résultat de tentative |
    | `plugin-sdk/provider-zai-endpoint` | Helpers de détection de point de terminaison Z.A.I |
    | `plugin-sdk/infra-runtime` | Helpers d’événement système/Heartbeat |
    | `plugin-sdk/collection-runtime` | Petits helpers de cache borné |
    | `plugin-sdk/diagnostic-runtime` | Helpers de drapeau et d’événement de diagnostic |
    | `plugin-sdk/error-runtime` | Graphe d’erreurs, formatage, helpers partagés de classification d’erreurs, `isApprovalNotFoundError` |
    | `plugin-sdk/fetch-runtime` | Helpers de fetch encapsulé, proxy et recherche épinglée |
    | `plugin-sdk/runtime-fetch` | Fetch d’exécution sensible au dispatcher sans imports proxy/fetch protégé |
    | `plugin-sdk/response-limit-runtime` | Lecteur borné de corps de réponse sans la large surface d’exécution média |
    | `plugin-sdk/session-binding-runtime` | État de liaison de conversation actuelle sans routage de liaison configurée ni magasins d’appairage |
    | `plugin-sdk/session-store-runtime` | Helpers de lecture du magasin de session sans imports larges d’écriture/de maintenance de configuration |
    | `plugin-sdk/context-visibility-runtime` | Résolution de visibilité du contexte et filtrage de contexte supplémentaire sans imports larges de configuration/sécurité |
    | `plugin-sdk/string-coerce-runtime` | Helpers ciblés de coercition et de normalisation de record/chaîne primitive sans imports Markdown/journalisation |
    | `plugin-sdk/host-runtime` | Helpers de normalisation de nom d’hôte et d’hôte SCP |
    | `plugin-sdk/retry-runtime` | Helpers de configuration de nouvelle tentative et d’exécuteur de nouvelle tentative |
    | `plugin-sdk/agent-runtime` | Helpers de répertoire/identité/espace de travail d’agent |
    | `plugin-sdk/directory-runtime` | Requête/déduplication de répertoire adossée à la configuration |
    | `plugin-sdk/keyed-async-queue` | `KeyedAsyncQueue` |
  </Accordion>

  <Accordion title="Sous-chemins de capacités et de test">
    | Sous-chemin | Exportations principales |
    | --- | --- |
    | `plugin-sdk/media-runtime` | Helpers partagés de récupération/transformation/stockage de médias, plus constructeurs de payload média |
    | `plugin-sdk/media-generation-runtime` | Helpers partagés de basculement en cas d’échec pour la génération de médias, sélection de candidats et messages de modèle manquant |
    | `plugin-sdk/media-understanding` | Types de fournisseur de compréhension de médias, plus exports helpers côté fournisseur pour image/audio |
    | `plugin-sdk/text-runtime` | Helpers partagés de texte/Markdown/journalisation comme la suppression de texte visible par l’assistant, les helpers de rendu/découpage/tableau Markdown, les helpers de masquage, les helpers de balises de directive et les utilitaires de texte sûr |
    | `plugin-sdk/text-chunking` | Helper de découpage de texte sortant |
    | `plugin-sdk/speech` | Types de fournisseur Speech, plus helpers côté fournisseur pour directives, registre et validation |
    | `plugin-sdk/speech-core` | Helpers partagés de types de fournisseur Speech, registre, directive et normalisation |
    | `plugin-sdk/realtime-transcription` | Types de fournisseur de transcription en temps réel et helpers de registre |
    | `plugin-sdk/realtime-voice` | Types de fournisseur de voix en temps réel et helpers de registre |
    | `plugin-sdk/image-generation` | Types de fournisseur de génération d’image |
    | `plugin-sdk/image-generation-core` | Helpers partagés de types, basculement en cas d’échec, authentification et registre pour la génération d’image |
    | `plugin-sdk/music-generation` | Types de fournisseur/requête/résultat pour la génération musicale |
    | `plugin-sdk/music-generation-core` | Helpers partagés de types, basculement en cas d’échec, recherche de fournisseur et analyse de `model-ref` pour la génération musicale |
    | `plugin-sdk/video-generation` | Types de fournisseur/requête/résultat pour la génération vidéo |
    | `plugin-sdk/video-generation-core` | Helpers partagés de types, basculement en cas d’échec, recherche de fournisseur et analyse de `model-ref` pour la génération vidéo |
    | `plugin-sdk/webhook-targets` | Registre de cibles Webhook et helpers d’installation de route |
    | `plugin-sdk/webhook-path` | Helpers de normalisation de chemin Webhook |
    | `plugin-sdk/web-media` | Helpers partagés de chargement de médias distants/locaux |
    | `plugin-sdk/zod` | `zod` réexporté pour les consommateurs du SDK Plugin |
    | `plugin-sdk/testing` | `installCommonResolveTargetErrorCases`, `shouldAckReaction` |
  </Accordion>

  <Accordion title="Sous-chemins de mémoire">
    | Sous-chemin | Exportations principales |
    | --- | --- |
    | `plugin-sdk/memory-core` | Surface helper `memory-core` intégrée pour les helpers de gestionnaire/configuration/fichier/CLI |
    | `plugin-sdk/memory-core-engine-runtime` | Façade d’exécution d’indexation/recherche mémoire |
    | `plugin-sdk/memory-core-host-engine-foundation` | Exports du moteur de fondation hôte mémoire |
    | `plugin-sdk/memory-core-host-engine-embeddings` | Contrats d’embeddings hôte mémoire, accès au registre, fournisseur local et helpers génériques de lot/distants |
    | `plugin-sdk/memory-core-host-engine-qmd` | Exports du moteur QMD hôte mémoire |
    | `plugin-sdk/memory-core-host-engine-storage` | Exports du moteur de stockage hôte mémoire |
    | `plugin-sdk/memory-core-host-multimodal` | Helpers multimodaux hôte mémoire |
    | `plugin-sdk/memory-core-host-query` | Helpers de requête hôte mémoire |
    | `plugin-sdk/memory-core-host-secret` | Helpers de secret hôte mémoire |
    | `plugin-sdk/memory-core-host-events` | Helpers de journal d’événements hôte mémoire |
    | `plugin-sdk/memory-core-host-status` | Helpers d’état hôte mémoire |
    | `plugin-sdk/memory-core-host-runtime-cli` | Helpers d’exécution CLI hôte mémoire |
    | `plugin-sdk/memory-core-host-runtime-core` | Helpers d’exécution cœur hôte mémoire |
    | `plugin-sdk/memory-core-host-runtime-files` | Helpers de fichier/d’exécution hôte mémoire |
    | `plugin-sdk/memory-host-core` | Alias neutre vis-à-vis du fournisseur pour les helpers d’exécution cœur hôte mémoire |
    | `plugin-sdk/memory-host-events` | Alias neutre vis-à-vis du fournisseur pour les helpers de journal d’événements hôte mémoire |
    | `plugin-sdk/memory-host-files` | Alias neutre vis-à-vis du fournisseur pour les helpers de fichier/d’exécution hôte mémoire |
    | `plugin-sdk/memory-host-markdown` | Helpers partagés de Markdown géré pour les plugins liés à la mémoire |
    | `plugin-sdk/memory-host-search` | Façade d’exécution Active Memory pour l’accès au gestionnaire de recherche |
    | `plugin-sdk/memory-host-status` | Alias neutre vis-à-vis du fournisseur pour les helpers d’état hôte mémoire |
    | `plugin-sdk/memory-lancedb` | Surface helper `memory-lancedb` intégrée |
  </Accordion>

  <Accordion title="Sous-chemins helpers intégrés réservés">
    | Famille | Sous-chemins actuels | Usage prévu |
    | --- | --- | --- |
    | Browser | `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support` | Helpers de prise en charge du Plugin browser intégré (`browser-support` reste le barrel de compatibilité) |
    | Matrix | `plugin-sdk/matrix`, `plugin-sdk/matrix-helper`, `plugin-sdk/matrix-runtime-heavy`, `plugin-sdk/matrix-runtime-shared`, `plugin-sdk/matrix-runtime-surface`, `plugin-sdk/matrix-surface`, `plugin-sdk/matrix-thread-bindings` | Surface helper/d’exécution Matrix intégrée |
    | Line | `plugin-sdk/line`, `plugin-sdk/line-core`, `plugin-sdk/line-runtime`, `plugin-sdk/line-surface` | Surface helper/d’exécution LINE intégrée |
    | IRC | `plugin-sdk/irc`, `plugin-sdk/irc-surface` | Surface helper IRC intégrée |
    | Helpers spécifiques aux canaux | `plugin-sdk/googlechat`, `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles`, `plugin-sdk/bluebubbles-policy`, `plugin-sdk/mattermost`, `plugin-sdk/mattermost-policy`, `plugin-sdk/feishu-conversation`, `plugin-sdk/msteams`, `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`, `plugin-sdk/twitch` | Points d’entrée de compatibilité/helper de canaux intégrés |
    | Helpers spécifiques à l’authentification/au plugin | `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`, `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`, `plugin-sdk/thread-ownership`, `plugin-sdk/voice-call` | Points d’entrée helper de fonctionnalité/plugin intégrés ; `plugin-sdk/github-copilot-token` exporte actuellement `DEFAULT_COPILOT_API_BASE_URL`, `deriveCopilotApiBaseUrlFromToken` et `resolveCopilotApiToken` |
  </Accordion>
</AccordionGroup>

## API d’enregistrement

Le callback `register(api)` reçoit un objet `OpenClawPluginApi` avec ces
méthodes :

### Enregistrement de capacités

| Méthode                                          | Ce qu’elle enregistre                  |
| ------------------------------------------------ | -------------------------------------- |
| `api.registerProvider(...)`                      | Inférence de texte (LLM)               |
| `api.registerAgentHarness(...)`                  | Exécuteur d’agent expérimental de bas niveau |
| `api.registerCliBackend(...)`                    | Backend local d’inférence CLI          |
| `api.registerChannel(...)`                       | Canal de messagerie                    |
| `api.registerSpeechProvider(...)`                | Synthèse texte-parole / STT            |
| `api.registerRealtimeTranscriptionProvider(...)` | Transcription temps réel en streaming  |
| `api.registerRealtimeVoiceProvider(...)`         | Sessions vocales temps réel duplex     |
| `api.registerMediaUnderstandingProvider(...)`    | Analyse d’images/audio/vidéo           |
| `api.registerImageGenerationProvider(...)`       | Génération d’image                     |
| `api.registerMusicGenerationProvider(...)`       | Génération musicale                    |
| `api.registerVideoGenerationProvider(...)`       | Génération vidéo                       |
| `api.registerWebFetchProvider(...)`              | Fournisseur de récupération / scraping web |
| `api.registerWebSearchProvider(...)`             | Recherche web                          |

### Outils et commandes

| Méthode                         | Ce qu’elle enregistre                         |
| ------------------------------- | --------------------------------------------- |
| `api.registerTool(tool, opts?)` | Outil d’agent (requis ou `{ optional: true }`) |
| `api.registerCommand(def)`      | Commande personnalisée (contourne le LLM)     |

### Infrastructure

| Méthode                                        | Ce qu’elle enregistre                 |
| ---------------------------------------------- | ------------------------------------- |
| `api.registerHook(events, handler, opts?)`     | Hook d’événement                      |
| `api.registerHttpRoute(params)`                | Point de terminaison HTTP Gateway     |
| `api.registerGatewayMethod(name, handler)`     | Méthode RPC Gateway                   |
| `api.registerCli(registrar, opts?)`            | Sous-commande CLI                     |
| `api.registerService(service)`                 | Service d’arrière-plan                |
| `api.registerInteractiveHandler(registration)` | Gestionnaire interactif               |
| `api.registerMemoryPromptSupplement(builder)`  | Section additive de prompt liée à la mémoire |
| `api.registerMemoryCorpusSupplement(adapter)`  | Corpus additif de recherche/lecture mémoire |

Les espaces de noms d’administration du cœur réservés (`config.*`, `exec.approvals.*`, `wizard.*`,
`update.*`) restent toujours `operator.admin`, même si un plugin tente
d’attribuer une portée de méthode Gateway plus étroite. Préférez des préfixes spécifiques au plugin pour les
méthodes possédées par le plugin.

### Métadonnées d’enregistrement CLI

`api.registerCli(registrar, opts?)` accepte deux types de métadonnées de premier niveau :

- `commands` : racines de commande explicites possédées par le registrar
- `descriptors` : descripteurs de commande au moment de l’analyse utilisés pour l’aide de la CLI racine,
  le routage et l’enregistrement paresseux de CLI de plugin

Si vous voulez qu’une commande de plugin reste chargée à la demande dans le chemin normal de la CLI racine,
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
        description: "Gérer les comptes Matrix, la vérification, les appareils et l’état du profil",
        hasSubcommands: true,
      },
    ],
  },
);
```

Utilisez `commands` seul uniquement lorsque vous n’avez pas besoin d’un enregistrement paresseux dans la CLI racine.
Ce chemin de compatibilité chargé de manière anticipée reste pris en charge, mais il n’installe pas
de placeholders adossés à des descripteurs pour le chargement paresseux au moment de l’analyse.

### Enregistrement de backend CLI

`api.registerCliBackend(...)` permet à un plugin de posséder la configuration par défaut pour un
backend CLI IA local tel que `codex-cli`.

- Le `id` du backend devient le préfixe du fournisseur dans des références de modèle comme `codex-cli/gpt-5`.
- Le `config` du backend utilise la même forme que `agents.defaults.cliBackends.<id>`.
- La configuration utilisateur garde la priorité. OpenClaw fusionne `agents.defaults.cliBackends.<id>` par-dessus la
  valeur par défaut du plugin avant d’exécuter la CLI.
- Utilisez `normalizeConfig` lorsqu’un backend a besoin de réécritures de compatibilité après fusion
  (par exemple pour normaliser d’anciennes formes de drapeaux).

### Slots exclusifs

| Méthode                                    | Ce qu’elle enregistre                                                                                                                                       |
| ------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `api.registerContextEngine(id, factory)`   | Moteur de contexte (un seul actif à la fois). Le callback `assemble()` reçoit `availableTools` et `citationsMode` afin que le moteur puisse adapter les ajouts au prompt. |
| `api.registerMemoryCapability(capability)` | Capacité mémoire unifiée                                                                                                                                    |
| `api.registerMemoryPromptSection(builder)` | Constructeur de section de prompt mémoire                                                                                                                   |
| `api.registerMemoryFlushPlan(resolver)`    | Résolveur de plan de vidage mémoire                                                                                                                         |
| `api.registerMemoryRuntime(runtime)`       | Adaptateur d’exécution mémoire                                                                                                                              |

### Adaptateurs d’embeddings mémoire

| Méthode                                        | Ce qu’elle enregistre                         |
| ---------------------------------------------- | --------------------------------------------- |
| `api.registerMemoryEmbeddingProvider(adapter)` | Adaptateur d’embeddings mémoire pour le plugin actif |

- `registerMemoryCapability` est l’API exclusive préférée pour les plugins mémoire.
- `registerMemoryCapability` peut aussi exposer `publicArtifacts.listArtifacts(...)`
  afin que des plugins compagnons puissent consommer des artefacts mémoire exportés via
  `openclaw/plugin-sdk/memory-host-core` au lieu d’atteindre la disposition privée
  d’un plugin mémoire spécifique.
- `registerMemoryPromptSection`, `registerMemoryFlushPlan` et
  `registerMemoryRuntime` sont des API exclusives de plugin mémoire compatibles avec l’héritage.
- `registerMemoryEmbeddingProvider` permet au plugin mémoire actif d’enregistrer un
  ou plusieurs identifiants d’adaptateur d’embeddings (par exemple `openai`, `gemini` ou un identifiant personnalisé
  défini par le plugin).
- La configuration utilisateur, comme `agents.defaults.memorySearch.provider` et
  `agents.defaults.memorySearch.fallback`, se résout par rapport à ces identifiants d’adaptateur enregistrés.

### Événements et cycle de vie

| Méthode                                      | Ce qu’elle fait              |
| -------------------------------------------- | ---------------------------- |
| `api.on(hookName, handler, opts?)`           | Hook de cycle de vie typé    |
| `api.onConversationBindingResolved(handler)` | Callback de liaison de conversation |

### Sémantique de décision des hooks

- `before_tool_call` : retourner `{ block: true }` est terminal. Dès qu’un gestionnaire le définit, les gestionnaires de priorité inférieure sont ignorés.
- `before_tool_call` : retourner `{ block: false }` est traité comme aucune décision (identique à l’omission de `block`), et non comme une substitution.
- `before_install` : retourner `{ block: true }` est terminal. Dès qu’un gestionnaire le définit, les gestionnaires de priorité inférieure sont ignorés.
- `before_install` : retourner `{ block: false }` est traité comme aucune décision (identique à l’omission de `block`), et non comme une substitution.
- `reply_dispatch` : retourner `{ handled: true, ... }` est terminal. Dès qu’un gestionnaire revendique la distribution, les gestionnaires de priorité inférieure et le chemin par défaut de distribution du modèle sont ignorés.
- `message_sending` : retourner `{ cancel: true }` est terminal. Dès qu’un gestionnaire le définit, les gestionnaires de priorité inférieure sont ignorés.
- `message_sending` : retourner `{ cancel: false }` est traité comme aucune décision (identique à l’omission de `cancel`), et non comme une substitution.

### Champs de l’objet API

| Champ                    | Type                      | Description                                                                                 |
| ------------------------ | ------------------------- | ------------------------------------------------------------------------------------------- |
| `api.id`                 | `string`                  | ID du plugin                                                                                |
| `api.name`               | `string`                  | Nom d’affichage                                                                             |
| `api.version`            | `string?`                 | Version du plugin (facultative)                                                             |
| `api.description`        | `string?`                 | Description du plugin (facultative)                                                         |
| `api.source`             | `string`                  | Chemin source du plugin                                                                     |
| `api.rootDir`            | `string?`                 | Répertoire racine du plugin (facultatif)                                                    |
| `api.config`             | `OpenClawConfig`          | Instantané de configuration actuel (instantané d’exécution en mémoire actif lorsqu’il est disponible) |
| `api.pluginConfig`       | `Record<string, unknown>` | Configuration spécifique au plugin provenant de `plugins.entries.<id>.config`               |
| `api.runtime`            | `PluginRuntime`           | [Helpers d’exécution](/fr/plugins/sdk-runtime)                                                 |
| `api.logger`             | `PluginLogger`            | Logger à portée limitée (`debug`, `info`, `warn`, `error`)                                  |
| `api.registrationMode`   | `PluginRegistrationMode`  | Mode de chargement actuel ; `"setup-runtime"` est la fenêtre légère de démarrage/configuration avant l’entrée complète |
| `api.resolvePath(input)` | `(string) => string`      | Résoudre un chemin relatif à la racine du plugin                                            |

## Convention de module interne

Dans votre plugin, utilisez des fichiers barrel locaux pour les imports internes :

```
my-plugin/
  api.ts            # Exports publics pour les consommateurs externes
  runtime-api.ts    # Exports d’exécution internes uniquement
  index.ts          # Point d’entrée du plugin
  setup-entry.ts    # Entrée légère pour configuration uniquement (facultatif)
```

<Warning>
  N’importez jamais votre propre plugin via `openclaw/plugin-sdk/<your-plugin>`
  depuis le code de production. Faites passer les imports internes via `./api.ts` ou
  `./runtime-api.ts`. Le chemin SDK est uniquement le contrat externe.
</Warning>

Les surfaces publiques de plugins intégrés chargées via façade (`api.ts`, `runtime-api.ts`,
`index.ts`, `setup-entry.ts` et fichiers d’entrée publics similaires) privilégient désormais l’instantané
de configuration d’exécution actif lorsque OpenClaw est déjà en cours d’exécution. Si aucun instantané
d’exécution n’existe encore, elles reviennent au fichier de configuration résolu sur disque.

Les plugins de fournisseur peuvent également exposer un barrel de contrat local au plugin et étroit lorsqu’un
helper est volontairement spécifique au fournisseur et n’a pas encore sa place dans un sous-chemin SDK
générique. Exemple intégré actuel : le fournisseur Anthropic conserve ses helpers de flux Claude
dans son propre point d’entrée public `api.ts` / `contract-api.ts` au lieu de
promouvoir la logique d’en-tête bêta Anthropic et `service_tier` dans un contrat
générique `plugin-sdk/*`.

Autres exemples intégrés actuels :

- `@openclaw/openai-provider` : `api.ts` exporte des constructeurs de fournisseur,
  des helpers de modèle par défaut et des constructeurs de fournisseur temps réel
- `@openclaw/openrouter-provider` : `api.ts` exporte le constructeur de fournisseur ainsi que
  des helpers d’intégration/de configuration

<Warning>
  Le code de production d’extension doit également éviter les imports `openclaw/plugin-sdk/<other-plugin>`.
  Si un helper est réellement partagé, promouvez-le vers un sous-chemin SDK neutre
  tel que `openclaw/plugin-sdk/speech`, `.../provider-model-shared` ou une autre
  surface orientée capacité au lieu de coupler deux plugins entre eux.
</Warning>

## Lié

- [Points d’entrée](/fr/plugins/sdk-entrypoints) — options `definePluginEntry` et `defineChannelPluginEntry`
- [Helpers d’exécution](/fr/plugins/sdk-runtime) — référence complète de l’espace de noms `api.runtime`
- [Configuration et config](/fr/plugins/sdk-setup) — empaquetage, manifestes, schémas de configuration
- [Tests](/fr/plugins/sdk-testing) — utilitaires de test et règles de lint
- [Migration du SDK](/fr/plugins/sdk-migration) — migration depuis des surfaces obsolètes
- [Internes des plugins](/fr/plugins/architecture) — architecture approfondie et modèle de capacité
