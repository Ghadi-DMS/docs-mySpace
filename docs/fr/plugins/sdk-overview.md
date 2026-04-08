---
read_when:
    - Vous avez besoin de savoir depuis quel sous-chemin du SDK importer
    - Vous voulez une référence de toutes les méthodes d’enregistrement sur OpenClawPluginApi
    - Vous recherchez une exportation spécifique du SDK
sidebarTitle: SDK Overview
summary: Map des imports, référence de l’API d’enregistrement et architecture du SDK
title: Vue d’ensemble du SDK des plugins
x-i18n:
    generated_at: "2026-04-08T02:17:53Z"
    model: gpt-5.4
    provider: openai
    source_hash: c5a41bd82d165dfbb7fbd6e4528cf322e9133a51efe55fa8518a7a0a626d9d30
    source_path: plugins/sdk-overview.md
    workflow: 15
---

# Vue d’ensemble du SDK des plugins

Le SDK des plugins est le contrat typé entre les plugins et le cœur. Cette page
est la référence pour **quoi importer** et **ce que vous pouvez enregistrer**.

<Tip>
  **Vous cherchez un guide pratique ?**
  - Premier plugin ? Commencez par [Premiers pas](/fr/plugins/building-plugins)
  - Plugin de canal ? Consultez [Plugins de canal](/fr/plugins/sdk-channel-plugins)
  - Plugin de fournisseur ? Consultez [Plugins de fournisseur](/fr/plugins/sdk-provider-plugins)
</Tip>

## Convention d’import

Importez toujours depuis un sous-chemin spécifique :

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
```

Chaque sous-chemin est un petit module autonome. Cela permet de conserver un démarrage rapide et
d’éviter les problèmes de dépendances circulaires. Pour les helpers d’entrée/de build spécifiques aux canaux,
préférez `openclaw/plugin-sdk/channel-core` ; gardez `openclaw/plugin-sdk/core` pour
la surface d’ensemble plus large et les helpers partagés tels que
`buildChannelConfigSchema`.

N’ajoutez pas et ne dépendez pas de points d’entrée de commodité nommés d’après des fournisseurs tels que
`openclaw/plugin-sdk/slack`, `openclaw/plugin-sdk/discord`,
`openclaw/plugin-sdk/signal`, `openclaw/plugin-sdk/whatsapp`, ni de
points d’entrée de helpers marqués canal. Les plugins inclus doivent composer des sous-chemins
génériques du SDK dans leurs propres barils `api.ts` ou `runtime-api.ts`, et le cœur
doit soit utiliser ces barils locaux au plugin, soit ajouter un contrat SDK générique et ciblé
lorsque le besoin est réellement transversal aux canaux.

La map d’exports générée contient toujours un petit ensemble de points d’entrée de helpers de plugins inclus
tels que `plugin-sdk/feishu`, `plugin-sdk/feishu-setup`,
`plugin-sdk/zalo`, `plugin-sdk/zalo-setup` et `plugin-sdk/matrix*`. Ces
sous-chemins existent uniquement pour la maintenance et la compatibilité des plugins inclus ; ils sont
volontairement omis du tableau commun ci-dessous et ne constituent pas le
chemin d’import recommandé pour les nouveaux plugins tiers.

## Référence des sous-chemins

Les sous-chemins les plus couramment utilisés, regroupés par usage. La liste complète générée de
plus de 200 sous-chemins se trouve dans `scripts/lib/plugin-sdk-entrypoints.json`.

Les sous-chemins réservés de helpers de plugins inclus apparaissent toujours dans cette liste générée.
Considérez-les comme des surfaces de détail d’implémentation/de compatibilité sauf si une page de documentation
les présente explicitement comme publiques.

### Entrée de plugin

| Sous-chemin                 | Exports clés                                                                                                                          |
| --------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/plugin-entry`   | `definePluginEntry`                                                                                                                    |
| `plugin-sdk/core`           | `defineChannelPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase`, `defineSetupPluginEntry`, `buildChannelConfigSchema` |
| `plugin-sdk/config-schema`  | `OpenClawSchema`                                                                                                                       |
| `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry`                                                                                                      |

<AccordionGroup>
  <Accordion title="Sous-chemins de canal">
    | Sous-chemin | Exports clés |
    | --- | --- |
    | `plugin-sdk/channel-core` | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
    | `plugin-sdk/config-schema` | Export du schéma Zod racine `openclaw.json` (`OpenClawSchema`) |
    | `plugin-sdk/channel-setup` | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, ainsi que `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
    | `plugin-sdk/setup` | Helpers partagés d’assistant de configuration, invites d’allowlist, constructeurs d’état de configuration |
    | `plugin-sdk/setup-runtime` | `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
    | `plugin-sdk/setup-adapter-runtime` | `createEnvPatchedAccountSetupAdapter` |
    | `plugin-sdk/setup-tools` | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
    | `plugin-sdk/account-core` | Helpers de configuration multi-comptes / garde de portes d’action, helpers de repli sur le compte par défaut |
    | `plugin-sdk/account-id` | `DEFAULT_ACCOUNT_ID`, helpers de normalisation d’ID de compte |
    | `plugin-sdk/account-resolution` | Helpers de recherche de compte + repli par défaut |
    | `plugin-sdk/account-helpers` | Helpers ciblés de liste d’actions / d’actions de compte |
    | `plugin-sdk/channel-pairing` | `createChannelPairingController` |
    | `plugin-sdk/channel-reply-pipeline` | `createChannelReplyPipeline` |
    | `plugin-sdk/channel-config-helpers` | `createHybridChannelConfigAdapter` |
    | `plugin-sdk/channel-config-schema` | Types de schéma de configuration de canal |
    | `plugin-sdk/telegram-command-config` | Helpers de normalisation/validation des commandes personnalisées Telegram avec repli de contrat inclus |
    | `plugin-sdk/channel-policy` | `resolveChannelGroupRequireMention` |
    | `plugin-sdk/channel-lifecycle` | `createAccountStatusSink` |
    | `plugin-sdk/inbound-envelope` | Helpers partagés de routage entrant + construction d’enveloppe |
    | `plugin-sdk/inbound-reply-dispatch` | Helpers partagés d’enregistrement et de distribution entrante |
    | `plugin-sdk/messaging-targets` | Helpers d’analyse/correspondance de cible |
    | `plugin-sdk/outbound-media` | Helpers partagés de chargement de médias sortants |
    | `plugin-sdk/outbound-runtime` | Helpers d’identité sortante / délégation d’envoi |
    | `plugin-sdk/thread-bindings-runtime` | Helpers d’adaptateur et de cycle de vie des liaisons de fils |
    | `plugin-sdk/agent-media-payload` | Constructeur hérité de charge utile média d’agent |
    | `plugin-sdk/conversation-runtime` | Helpers de conversation/liaison de fil, d’appairage et de liaisons configurées |
    | `plugin-sdk/runtime-config-snapshot` | Helper d’instantané de configuration d’exécution |
    | `plugin-sdk/runtime-group-policy` | Helpers de résolution de politique de groupe à l’exécution |
    | `plugin-sdk/channel-status` | Helpers partagés d’instantané/résumé d’état de canal |
    | `plugin-sdk/channel-config-primitives` | Primitives ciblées de schéma de configuration de canal |
    | `plugin-sdk/channel-config-writes` | Helpers d’autorisation d’écriture de configuration de canal |
    | `plugin-sdk/channel-plugin-common` | Exports de prélude de plugin de canal partagés |
    | `plugin-sdk/allowlist-config-edit` | Helpers de lecture/édition de configuration d’allowlist |
    | `plugin-sdk/group-access` | Helpers partagés de décision d’accès de groupe |
    | `plugin-sdk/direct-dm` | Helpers partagés d’authentification/garde de MP directs |
    | `plugin-sdk/interactive-runtime` | Helpers de normalisation/réduction de charge utile de réponse interactive |
    | `plugin-sdk/channel-inbound` | Helpers de debounce entrant, de correspondance de mention, de politique de mention et d’enveloppe |
    | `plugin-sdk/channel-send-result` | Types de résultat de réponse |
    | `plugin-sdk/channel-actions` | `createMessageToolButtonsSchema`, `createMessageToolCardSchema` |
    | `plugin-sdk/channel-targets` | Helpers d’analyse/correspondance de cible |
    | `plugin-sdk/channel-contract` | Types de contrat de canal |
    | `plugin-sdk/channel-feedback` | Câblage de feedback/réaction |
    | `plugin-sdk/channel-secret-runtime` | Helpers ciblés de contrat de secrets tels que `collectSimpleChannelFieldAssignments`, `getChannelSurface`, `pushAssignment`, ainsi que types de cibles de secrets |
  </Accordion>

  <Accordion title="Sous-chemins de fournisseur">
    | Sous-chemin | Exports clés |
    | --- | --- |
    | `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry` |
    | `plugin-sdk/provider-setup` | Helpers soignés de configuration de fournisseurs locaux / auto-hébergés |
    | `plugin-sdk/self-hosted-provider-setup` | Helpers ciblés de configuration de fournisseurs auto-hébergés compatibles OpenAI |
    | `plugin-sdk/cli-backend` | Valeurs par défaut du backend CLI + constantes watchdog |
    | `plugin-sdk/provider-auth-runtime` | Helpers d’exécution de résolution de clé API pour plugins de fournisseur |
    | `plugin-sdk/provider-auth-api-key` | Helpers d’onboarding / d’écriture de profil de clé API tels que `upsertApiKeyProfile` |
    | `plugin-sdk/provider-auth-result` | Constructeur standard de résultat d’authentification OAuth |
    | `plugin-sdk/provider-auth-login` | Helpers partagés de connexion interactive pour plugins de fournisseur |
    | `plugin-sdk/provider-env-vars` | Helpers de recherche de variables d’environnement d’authentification de fournisseur |
    | `plugin-sdk/provider-auth` | `createProviderApiKeyAuthMethod`, `ensureApiKeyFromOptionEnvOrPrompt`, `upsertAuthProfile`, `upsertApiKeyProfile`, `writeOAuthCredentials` |
    | `plugin-sdk/provider-model-shared` | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, constructeurs partagés de politique de rejeu, helpers de point de terminaison de fournisseur et helpers de normalisation d’ID de modèle tels que `normalizeNativeXaiModelId` |
    | `plugin-sdk/provider-catalog-shared` | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
    | `plugin-sdk/provider-http` | Helpers génériques de capacité HTTP / point de terminaison de fournisseur |
    | `plugin-sdk/provider-web-fetch-contract` | Helpers ciblés de contrat de configuration / sélection web-fetch tels que `enablePluginInConfig` et `WebFetchProviderPlugin` |
    | `plugin-sdk/provider-web-fetch` | Helpers d’enregistrement / cache de fournisseur web-fetch |
    | `plugin-sdk/provider-web-search-contract` | Helpers ciblés de contrat de configuration / identifiants web-search tels que `enablePluginInConfig`, `resolveProviderWebSearchPluginConfig` et setters/getters d’identifiants limités |
    | `plugin-sdk/provider-web-search` | Helpers d’exécution / enregistrement / cache de fournisseur web-search |
    | `plugin-sdk/provider-tools` | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, nettoyage + diagnostics de schéma Gemini et helpers de compatibilité xAI tels que `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
    | `plugin-sdk/provider-usage` | `fetchClaudeUsage` et similaires |
    | `plugin-sdk/provider-stream` | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, types de wrappers de flux et helpers partagés de wrappers Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot |
    | `plugin-sdk/provider-onboard` | Helpers de patch de configuration d’onboarding |
    | `plugin-sdk/global-singleton` | Helpers de singleton / map / cache locaux au processus |
  </Accordion>

  <Accordion title="Sous-chemins d’authentification et de sécurité">
    | Sous-chemin | Exports clés |
    | --- | --- |
    | `plugin-sdk/command-auth` | `resolveControlCommandGate`, helpers de registre de commandes, helpers d’autorisation d’expéditeur |
    | `plugin-sdk/approval-auth-runtime` | Helpers de résolution d’approbateur et d’authentification d’action dans le même chat |
    | `plugin-sdk/approval-client-runtime` | Helpers natifs de profil / filtre d’approbation exec |
    | `plugin-sdk/approval-delivery-runtime` | Adaptateurs natifs de capacité / livraison d’approbation |
    | `plugin-sdk/approval-gateway-runtime` | Helper partagé de résolution de passerelle d’approbation |
    | `plugin-sdk/approval-handler-adapter-runtime` | Helpers légers de chargement d’adaptateur d’approbation native pour points d’entrée de canal à chaud |
    | `plugin-sdk/approval-handler-runtime` | Helpers d’exécution plus larges du gestionnaire d’approbation ; préférez les points d’entrée plus ciblés adaptateur/passerelle lorsqu’ils suffisent |
    | `plugin-sdk/approval-native-runtime` | Helpers natifs de cible d’approbation + liaison de compte |
    | `plugin-sdk/approval-reply-runtime` | Helpers de charge utile de réponse pour approbations exec/plugin |
    | `plugin-sdk/command-auth-native` | Helpers d’authentification native de commande + cible de session native |
    | `plugin-sdk/command-detection` | Helpers partagés de détection de commande |
    | `plugin-sdk/command-surface` | Helpers de normalisation du corps de commande et de surface de commande |
    | `plugin-sdk/allow-from` | `formatAllowFromLowercase` |
    | `plugin-sdk/channel-secret-runtime` | Helpers ciblés de collecte de contrat de secrets pour surfaces de secrets canal/plugin |
    | `plugin-sdk/secret-ref-runtime` | Helpers ciblés `coerceSecretRef` et de typage SecretRef pour l’analyse de contrat/config de secrets |
    | `plugin-sdk/security-runtime` | Helpers partagés de confiance, de contrôle d’accès MP, de contenu externe et de collecte de secrets |
    | `plugin-sdk/ssrf-policy` | Helpers de politique SSRF pour allowlist d’hôtes et réseau privé |
    | `plugin-sdk/ssrf-runtime` | Helpers de fetch protégé SSRF, de dispatcher épinglé et de politique SSRF |
    | `plugin-sdk/secret-input` | Helpers d’analyse d’entrée de secret |
    | `plugin-sdk/webhook-ingress` | Helpers de requête / cible de webhook |
    | `plugin-sdk/webhook-request-guards` | Helpers de taille de corps / délai d’attente de requête |
  </Accordion>

  <Accordion title="Sous-chemins d’exécution et de stockage">
    | Sous-chemin | Exports clés |
    | --- | --- |
    | `plugin-sdk/runtime` | Helpers larges d’exécution / journalisation / sauvegarde / installation de plugin |
    | `plugin-sdk/runtime-env` | Helpers ciblés d’environnement d’exécution, logger, délai, nouvelle tentative et backoff |
    | `plugin-sdk/channel-runtime-context` | Helpers génériques d’enregistrement et de recherche de contexte d’exécution de canal |
    | `plugin-sdk/runtime-store` | `createPluginRuntimeStore` |
    | `plugin-sdk/plugin-runtime` | Helpers partagés de commande / hook / HTTP / interaction de plugin |
    | `plugin-sdk/hook-runtime` | Helpers partagés de pipeline de hook webhook/interne |
    | `plugin-sdk/lazy-runtime` | Helpers de liaison/import paresseux d’exécution tels que `createLazyRuntimeModule`, `createLazyRuntimeMethod` et `createLazyRuntimeSurface` |
    | `plugin-sdk/process-runtime` | Helpers d’exécution de processus |
    | `plugin-sdk/cli-runtime` | Helpers CLI de formatage, d’attente et de version |
    | `plugin-sdk/gateway-runtime` | Helpers de client passerelle et de patch d’état de canal |
    | `plugin-sdk/config-runtime` | Helpers de chargement / écriture de configuration |
    | `plugin-sdk/telegram-command-config` | Normalisation du nom/de la description de commande Telegram et vérifications de doublons/conflits, même lorsque la surface de contrat Telegram incluse n’est pas disponible |
    | `plugin-sdk/approval-runtime` | Helpers d’approbation exec/plugin, constructeurs de capacité d’approbation, helpers d’authentification / profil, helpers natifs de routage / exécution |
    | `plugin-sdk/reply-runtime` | Helpers partagés d’exécution de réponse/entrée, découpage, distribution, heartbeat, planificateur de réponse |
    | `plugin-sdk/reply-dispatch-runtime` | Helpers ciblés de distribution / finalisation de réponse |
    | `plugin-sdk/reply-history` | Helpers partagés d’historique de réponse sur courte fenêtre tels que `buildHistoryContext`, `recordPendingHistoryEntry` et `clearHistoryEntriesIfEnabled` |
    | `plugin-sdk/reply-reference` | `createReplyReferencePlanner` |
    | `plugin-sdk/reply-chunking` | Helpers ciblés de découpage texte/Markdown |
    | `plugin-sdk/session-store-runtime` | Helpers de chemin de magasin de session + `updated-at` |
    | `plugin-sdk/state-paths` | Helpers de chemin de répertoire d’état / OAuth |
    | `plugin-sdk/routing` | Helpers de routage / clé de session / liaison de compte tels que `resolveAgentRoute`, `buildAgentSessionKey` et `resolveDefaultAgentBoundAccountId` |
    | `plugin-sdk/status-helpers` | Helpers partagés de résumé d’état canal/compte, valeurs par défaut d’état d’exécution et helpers de métadonnées de problème |
    | `plugin-sdk/target-resolver-runtime` | Helpers partagés de résolution de cible |
    | `plugin-sdk/string-normalization-runtime` | Helpers de normalisation de slug/chaîne |
    | `plugin-sdk/request-url` | Extraire des URL chaîne à partir d’entrées de type fetch/requête |
    | `plugin-sdk/run-command` | Exécuteur de commande chronométré avec résultats stdout/stderr normalisés |
    | `plugin-sdk/param-readers` | Lecteurs de paramètres communs d’outil/CLI |
    | `plugin-sdk/tool-send` | Extraire les champs de cible d’envoi canonique des arguments d’outil |
    | `plugin-sdk/temp-path` | Helpers partagés de chemin de téléchargement temporaire |
    | `plugin-sdk/logging-core` | Helpers de logger de sous-système et de masquage |
    | `plugin-sdk/markdown-table-runtime` | Helpers de mode tableau Markdown |
    | `plugin-sdk/json-store` | Petits helpers de lecture/écriture d’état JSON |
    | `plugin-sdk/file-lock` | Helpers de verrou de fichier réentrant |
    | `plugin-sdk/persistent-dedupe` | Helpers de cache de déduplication sauvegardé sur disque |
    | `plugin-sdk/acp-runtime` | Helpers d’exécution/session ACP et de distribution de réponse |
    | `plugin-sdk/agent-config-primitives` | Primitives ciblées de schéma de configuration d’exécution d’agent |
    | `plugin-sdk/boolean-param` | Lecteur permissif de paramètre booléen |
    | `plugin-sdk/dangerous-name-runtime` | Helpers de résolution de correspondance de noms dangereux |
    | `plugin-sdk/device-bootstrap` | Helpers de bootstrap d’appareil et de jeton d’appairage |
    | `plugin-sdk/extension-shared` | Primitives partagées de canal passif, d’état et de proxy ambiant |
    | `plugin-sdk/models-provider-runtime` | Helpers de réponse de fournisseur / commande `/models` |
    | `plugin-sdk/skill-commands-runtime` | Helpers de liste de commandes de Skills |
    | `plugin-sdk/native-command-registry` | Helpers de registre/construction/sérialisation de commandes natives |
    | `plugin-sdk/provider-zai-endpoint` | Helpers de détection de point de terminaison Z.AI |
    | `plugin-sdk/infra-runtime` | Helpers d’événement système / heartbeat |
    | `plugin-sdk/collection-runtime` | Petits helpers de cache borné |
    | `plugin-sdk/diagnostic-runtime` | Helpers de drapeau et d’événement de diagnostic |
    | `plugin-sdk/error-runtime` | Helpers de graphe d’erreur, de formatage, de classification partagée d’erreur, `isApprovalNotFoundError` |
    | `plugin-sdk/fetch-runtime` | Helpers de fetch encapsulé, proxy et recherche épinglée |
    | `plugin-sdk/host-runtime` | Helpers de normalisation de nom d’hôte et d’hôte SCP |
    | `plugin-sdk/retry-runtime` | Helpers de configuration et d’exécuteur de nouvelle tentative |
    | `plugin-sdk/agent-runtime` | Helpers de répertoire / identité / espace de travail d’agent |
    | `plugin-sdk/directory-runtime` | Requête/déduplication de répertoire appuyée sur la configuration |
    | `plugin-sdk/keyed-async-queue` | `KeyedAsyncQueue` |
  </Accordion>

  <Accordion title="Sous-chemins de capacité et de test">
    | Sous-chemin | Exports clés |
    | --- | --- |
    | `plugin-sdk/media-runtime` | Helpers partagés de récupération / transformation / stockage de médias ainsi que constructeurs de charge utile média |
    | `plugin-sdk/media-generation-runtime` | Helpers partagés de repli de génération de média, sélection de candidats et messages de modèle manquant |
    | `plugin-sdk/media-understanding` | Types de fournisseur de compréhension média ainsi qu’exports de helpers d’image/audio orientés fournisseur |
    | `plugin-sdk/text-runtime` | Helpers partagés de texte/Markdown/journalisation tels que suppression de texte visible par l’assistant, helpers de rendu/découpage/tableau Markdown, helpers de masquage, helpers de balise de directive et utilitaires de texte sûr |
    | `plugin-sdk/text-chunking` | Helper de découpage de texte sortant |
    | `plugin-sdk/speech` | Types de fournisseur de parole ainsi que helpers de directive, de registre et de validation orientés fournisseur |
    | `plugin-sdk/speech-core` | Types partagés de fournisseur de parole, helpers de registre, de directive et de normalisation |
    | `plugin-sdk/realtime-transcription` | Types de fournisseur de transcription temps réel et helpers de registre |
    | `plugin-sdk/realtime-voice` | Types de fournisseur de voix temps réel et helpers de registre |
    | `plugin-sdk/image-generation` | Types de fournisseur de génération d’image |
    | `plugin-sdk/image-generation-core` | Types partagés de génération d’image, helpers de repli, d’authentification et de registre |
    | `plugin-sdk/music-generation` | Types de fournisseur / requête / résultat de génération musicale |
    | `plugin-sdk/music-generation-core` | Types partagés de génération musicale, helpers de repli, recherche de fournisseur et analyse de model-ref |
    | `plugin-sdk/video-generation` | Types de fournisseur / requête / résultat de génération vidéo |
    | `plugin-sdk/video-generation-core` | Types partagés de génération vidéo, helpers de repli, recherche de fournisseur et analyse de model-ref |
    | `plugin-sdk/webhook-targets` | Registre de cibles webhook et helpers d’installation de route |
    | `plugin-sdk/webhook-path` | Helpers de normalisation de chemin de webhook |
    | `plugin-sdk/web-media` | Helpers partagés de chargement de médias distants/locaux |
    | `plugin-sdk/zod` | `zod` réexporté pour les consommateurs du SDK de plugin |
    | `plugin-sdk/testing` | `installCommonResolveTargetErrorCases`, `shouldAckReaction` |
  </Accordion>

  <Accordion title="Sous-chemins mémoire">
    | Sous-chemin | Exports clés |
    | --- | --- |
    | `plugin-sdk/memory-core` | Surface de helpers memory-core incluse pour les helpers de gestionnaire/config/fichier/CLI |
    | `plugin-sdk/memory-core-engine-runtime` | Façade d’exécution d’indexation/recherche mémoire |
    | `plugin-sdk/memory-core-host-engine-foundation` | Exports de moteur de fondation hôte mémoire |
    | `plugin-sdk/memory-core-host-engine-embeddings` | Exports de moteur d’embeddings hôte mémoire |
    | `plugin-sdk/memory-core-host-engine-qmd` | Exports de moteur QMD hôte mémoire |
    | `plugin-sdk/memory-core-host-engine-storage` | Exports de moteur de stockage hôte mémoire |
    | `plugin-sdk/memory-core-host-multimodal` | Helpers multimodaux d’hôte mémoire |
    | `plugin-sdk/memory-core-host-query` | Helpers de requête d’hôte mémoire |
    | `plugin-sdk/memory-core-host-secret` | Helpers de secret d’hôte mémoire |
    | `plugin-sdk/memory-core-host-events` | Helpers de journal d’événements d’hôte mémoire |
    | `plugin-sdk/memory-core-host-status` | Helpers d’état d’hôte mémoire |
    | `plugin-sdk/memory-core-host-runtime-cli` | Helpers d’exécution CLI d’hôte mémoire |
    | `plugin-sdk/memory-core-host-runtime-core` | Helpers d’exécution cœur d’hôte mémoire |
    | `plugin-sdk/memory-core-host-runtime-files` | Helpers de fichiers/exécution d’hôte mémoire |
    | `plugin-sdk/memory-host-core` | Alias neutre vis-à-vis du fournisseur pour les helpers d’exécution cœur d’hôte mémoire |
    | `plugin-sdk/memory-host-events` | Alias neutre vis-à-vis du fournisseur pour les helpers de journal d’événements d’hôte mémoire |
    | `plugin-sdk/memory-host-files` | Alias neutre vis-à-vis du fournisseur pour les helpers de fichiers/exécution d’hôte mémoire |
    | `plugin-sdk/memory-host-markdown` | Helpers partagés de Markdown géré pour plugins adjacents à la mémoire |
    | `plugin-sdk/memory-host-search` | Façade d’exécution mémoire active pour l’accès au gestionnaire de recherche |
    | `plugin-sdk/memory-host-status` | Alias neutre vis-à-vis du fournisseur pour les helpers d’état d’hôte mémoire |
    | `plugin-sdk/memory-lancedb` | Surface de helpers memory-lancedb incluse |
  </Accordion>

  <Accordion title="Sous-chemins réservés de helpers inclus">
    | Famille | Sous-chemins actuels | Usage prévu |
    | --- | --- | --- |
    | Browser | `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support` | Helpers de prise en charge du plugin Browser inclus (`browser-support` reste le baril de compatibilité) |
    | Matrix | `plugin-sdk/matrix`, `plugin-sdk/matrix-helper`, `plugin-sdk/matrix-runtime-heavy`, `plugin-sdk/matrix-runtime-shared`, `plugin-sdk/matrix-runtime-surface`, `plugin-sdk/matrix-surface`, `plugin-sdk/matrix-thread-bindings` | Surface d’exécution/helper Matrix incluse |
    | Line | `plugin-sdk/line`, `plugin-sdk/line-core`, `plugin-sdk/line-runtime`, `plugin-sdk/line-surface` | Surface d’exécution/helper LINE incluse |
    | IRC | `plugin-sdk/irc`, `plugin-sdk/irc-surface` | Surface de helper IRC incluse |
    | Helpers spécifiques à un canal | `plugin-sdk/googlechat`, `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles`, `plugin-sdk/bluebubbles-policy`, `plugin-sdk/mattermost`, `plugin-sdk/mattermost-policy`, `plugin-sdk/feishu-conversation`, `plugin-sdk/msteams`, `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`, `plugin-sdk/twitch` | Points d’entrée de compatibilité/helper de canaux inclus |
    | Helpers spécifiques à l’authentification/au plugin | `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`, `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`, `plugin-sdk/thread-ownership`, `plugin-sdk/voice-call` | Points d’entrée de helpers de fonctionnalité/plugin inclus ; `plugin-sdk/github-copilot-token` exporte actuellement `DEFAULT_COPILOT_API_BASE_URL`, `deriveCopilotApiBaseUrlFromToken` et `resolveCopilotApiToken` |
  </Accordion>
</AccordionGroup>

## API d’enregistrement

Le callback `register(api)` reçoit un objet `OpenClawPluginApi` avec ces
méthodes :

### Enregistrement de capacités

| Méthode                                          | Ce qu’elle enregistre            |
| ------------------------------------------------ | -------------------------------- |
| `api.registerProvider(...)`                      | Inférence de texte (LLM)         |
| `api.registerCliBackend(...)`                    | Backend d’inférence CLI local    |
| `api.registerChannel(...)`                       | Canal de messagerie              |
| `api.registerSpeechProvider(...)`                | Synthèse texte-vers-parole / STT |
| `api.registerRealtimeTranscriptionProvider(...)` | Transcription temps réel en flux |
| `api.registerRealtimeVoiceProvider(...)`         | Sessions vocales temps réel duplex |
| `api.registerMediaUnderstandingProvider(...)`    | Analyse d’image/audio/vidéo      |
| `api.registerImageGenerationProvider(...)`       | Génération d’image               |
| `api.registerMusicGenerationProvider(...)`       | Génération musicale              |
| `api.registerVideoGenerationProvider(...)`       | Génération vidéo                 |
| `api.registerWebFetchProvider(...)`              | Fournisseur de récupération / scraping web |
| `api.registerWebSearchProvider(...)`             | Recherche web                    |

### Outils et commandes

| Méthode                         | Ce qu’elle enregistre                        |
| ------------------------------ | -------------------------------------------- |
| `api.registerTool(tool, opts?)` | Outil d’agent (obligatoire ou `{ optional: true }`) |
| `api.registerCommand(def)`      | Commande personnalisée (contourne le LLM)    |

### Infrastructure

| Méthode                                        | Ce qu’elle enregistre                |
| --------------------------------------------- | ------------------------------------ |
| `api.registerHook(events, handler, opts?)`    | Hook d’événement                     |
| `api.registerHttpRoute(params)`               | Point de terminaison HTTP de passerelle |
| `api.registerGatewayMethod(name, handler)`    | Méthode RPC de passerelle            |
| `api.registerCli(registrar, opts?)`           | Sous-commande CLI                    |
| `api.registerService(service)`                | Service d’arrière-plan               |
| `api.registerInteractiveHandler(registration)`| Gestionnaire interactif              |
| `api.registerMemoryPromptSupplement(builder)` | Section additive de prompt adjacente à la mémoire |
| `api.registerMemoryCorpusSupplement(adapter)` | Corpus additif de recherche/lecture mémoire |

Les espaces de noms d’administration du cœur réservés (`config.*`, `exec.approvals.*`, `wizard.*`,
`update.*`) restent toujours `operator.admin`, même si un plugin essaie d’assigner une
portée de méthode de passerelle plus étroite. Préférez des préfixes spécifiques au plugin pour
les méthodes possédées par le plugin.

### Métadonnées d’enregistrement CLI

`api.registerCli(registrar, opts?)` accepte deux types de métadonnées de premier niveau :

- `commands` : racines de commande explicites possédées par le registrar
- `descriptors` : descripteurs de commande au moment de l’analyse, utilisés pour l’aide CLI racine,
  le routage et l’enregistrement paresseux de CLI de plugin

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
d’espaces réservés adossés à des descripteurs pour le chargement paresseux au moment de l’analyse.

### Enregistrement de backend CLI

`api.registerCliBackend(...)` permet à un plugin de posséder la configuration par défaut d’un
backend CLI IA local tel que `codex-cli`.

- Le backend `id` devient le préfixe de fournisseur dans des références de modèle comme `codex-cli/gpt-5`.
- Le backend `config` utilise la même forme que `agents.defaults.cliBackends.<id>`.
- La configuration utilisateur garde la priorité. OpenClaw fusionne `agents.defaults.cliBackends.<id>` sur la
  valeur par défaut du plugin avant d’exécuter la CLI.
- Utilisez `normalizeConfig` lorsqu’un backend a besoin de réécritures de compatibilité après la fusion
  (par exemple pour normaliser d’anciennes formes de drapeaux).

### Emplacements exclusifs

| Méthode                                    | Ce qu’elle enregistre               |
| ----------------------------------------- | ----------------------------------- |
| `api.registerContextEngine(id, factory)`   | Moteur de contexte (un seul actif à la fois) |
| `api.registerMemoryCapability(capability)` | Capacité mémoire unifiée            |
| `api.registerMemoryPromptSection(builder)` | Constructeur de section de prompt mémoire |
| `api.registerMemoryFlushPlan(resolver)`    | Résolveur de plan de vidage mémoire |
| `api.registerMemoryRuntime(runtime)`       | Adaptateur d’exécution mémoire      |

### Adaptateurs d’embeddings mémoire

| Méthode                                        | Ce qu’elle enregistre                               |
| --------------------------------------------- | --------------------------------------------------- |
| `api.registerMemoryEmbeddingProvider(adapter)` | Adaptateur d’embeddings mémoire pour le plugin actif |

- `registerMemoryCapability` est l’API préférée de plugin mémoire exclusif.
- `registerMemoryCapability` peut également exposer `publicArtifacts.listArtifacts(...)`
  afin que des plugins compagnons puissent consommer des artefacts mémoire exportés via
  `openclaw/plugin-sdk/memory-host-core` au lieu d’atteindre l’agencement privé
  d’un plugin mémoire spécifique.
- `registerMemoryPromptSection`, `registerMemoryFlushPlan` et
  `registerMemoryRuntime` sont des API héritées compatibles de plugin mémoire exclusif.
- `registerMemoryEmbeddingProvider` permet au plugin mémoire actif d’enregistrer un
  ou plusieurs identifiants d’adaptateur d’embeddings (par exemple `openai`, `gemini` ou un identifiant
  personnalisé défini par le plugin).
- La configuration utilisateur telle que `agents.defaults.memorySearch.provider` et
  `agents.defaults.memorySearch.fallback` se résout par rapport à ces identifiants
  d’adaptateur enregistrés.

### Événements et cycle de vie

| Méthode                                      | Ce qu’elle fait              |
| ------------------------------------------- | ---------------------------- |
| `api.on(hookName, handler, opts?)`          | Hook de cycle de vie typé    |
| `api.onConversationBindingResolved(handler)`| Callback de liaison de conversation |

### Sémantique des décisions de hook

- `before_tool_call` : renvoyer `{ block: true }` est terminal. Dès qu’un gestionnaire le définit, les gestionnaires de priorité inférieure sont ignorés.
- `before_tool_call` : renvoyer `{ block: false }` est traité comme aucune décision (comme si `block` était omis), pas comme une substitution.
- `before_install` : renvoyer `{ block: true }` est terminal. Dès qu’un gestionnaire le définit, les gestionnaires de priorité inférieure sont ignorés.
- `before_install` : renvoyer `{ block: false }` est traité comme aucune décision (comme si `block` était omis), pas comme une substitution.
- `reply_dispatch` : renvoyer `{ handled: true, ... }` est terminal. Dès qu’un gestionnaire revendique la distribution, les gestionnaires de priorité inférieure et le chemin de distribution du modèle par défaut sont ignorés.
- `message_sending` : renvoyer `{ cancel: true }` est terminal. Dès qu’un gestionnaire le définit, les gestionnaires de priorité inférieure sont ignorés.
- `message_sending` : renvoyer `{ cancel: false }` est traité comme aucune décision (comme si `cancel` était omis), pas comme une substitution.

### Champs de l’objet API

| Champ                    | Type                      | Description                                                                                 |
| ------------------------ | ------------------------- | ------------------------------------------------------------------------------------------- |
| `api.id`                 | `string`                  | ID du plugin                                                                                |
| `api.name`               | `string`                  | Nom d’affichage                                                                             |
| `api.version`            | `string?`                 | Version du plugin (facultative)                                                             |
| `api.description`        | `string?`                 | Description du plugin (facultative)                                                         |
| `api.source`             | `string`                  | Chemin source du plugin                                                                     |
| `api.rootDir`            | `string?`                 | Répertoire racine du plugin (facultatif)                                                    |
| `api.config`             | `OpenClawConfig`          | Instantané actuel de configuration (instantané actif en mémoire à l’exécution lorsqu’il est disponible) |
| `api.pluginConfig`       | `Record<string, unknown>` | Configuration spécifique au plugin depuis `plugins.entries.<id>.config`                     |
| `api.runtime`            | `PluginRuntime`           | [Helpers d’exécution](/fr/plugins/sdk-runtime)                                                 |
| `api.logger`             | `PluginLogger`            | Logger limité (`debug`, `info`, `warn`, `error`)                                            |
| `api.registrationMode`   | `PluginRegistrationMode`  | Mode de chargement actuel ; `"setup-runtime"` est la fenêtre légère de démarrage/configuration avant l’entrée complète |
| `api.resolvePath(input)` | `(string) => string`      | Résoudre un chemin relatif à la racine du plugin                                            |

## Convention de module interne

À l’intérieur de votre plugin, utilisez des fichiers barils locaux pour les imports internes :

```
my-plugin/
  api.ts            # Exports publics pour les consommateurs externes
  runtime-api.ts    # Exports d’exécution internes uniquement
  index.ts          # Point d’entrée du plugin
  setup-entry.ts    # Entrée légère de configuration uniquement (facultative)
```

<Warning>
  N’importez jamais votre propre plugin via `openclaw/plugin-sdk/<your-plugin>`
  dans le code de production. Faites passer les imports internes par `./api.ts` ou
  `./runtime-api.ts`. Le chemin SDK est uniquement le contrat externe.
</Warning>

Les surfaces publiques de plugins inclus chargées via façade (`api.ts`, `runtime-api.ts`,
`index.ts`, `setup-entry.ts` et fichiers d’entrée publics similaires) préfèrent désormais
l’instantané actif de configuration d’exécution lorsqu’OpenClaw fonctionne déjà. Si aucun instantané
d’exécution n’existe encore, elles se rabattent sur le fichier de configuration résolu sur disque.

Les plugins de fournisseur peuvent également exposer un baril de contrat local au plugin et ciblé lorsqu’un
helper est intentionnellement spécifique au fournisseur et n’a pas encore sa place dans un sous-chemin SDK
générique. Exemple inclus actuel : le fournisseur Anthropic conserve ses
helpers de flux Claude dans son propre point d’entrée public `api.ts` / `contract-api.ts` au lieu de
promouvoir la logique d’en-tête bêta Anthropic et `service_tier` dans un contrat
générique `plugin-sdk/*`.

Autres exemples inclus actuels :

- `@openclaw/openai-provider` : `api.ts` exporte des constructeurs de fournisseur,
  des helpers de modèle par défaut et des constructeurs de fournisseur temps réel
- `@openclaw/openrouter-provider` : `api.ts` exporte le constructeur de fournisseur ainsi que
  des helpers d’onboarding/configuration

<Warning>
  Le code de production d’extension doit également éviter les imports `openclaw/plugin-sdk/<other-plugin>`.
  Si un helper est réellement partagé, faites-le remonter vers un sous-chemin SDK neutre
  tel que `openclaw/plugin-sdk/speech`, `.../provider-model-shared` ou une autre
  surface orientée capacité au lieu de coupler deux plugins ensemble.
</Warning>

## Associé

- [Points d’entrée](/fr/plugins/sdk-entrypoints) — options `definePluginEntry` et `defineChannelPluginEntry`
- [Helpers d’exécution](/fr/plugins/sdk-runtime) — référence complète de l’espace de noms `api.runtime`
- [Configuration et config](/fr/plugins/sdk-setup) — packaging, manifestes, schémas de configuration
- [Tests](/fr/plugins/sdk-testing) — utilitaires de test et règles de lint
- [Migration SDK](/fr/plugins/sdk-migration) — migration depuis des surfaces obsolètes
- [Internes des plugins](/fr/plugins/architecture) — architecture détaillée et modèle de capacités
