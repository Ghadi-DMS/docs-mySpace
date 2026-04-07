---
read_when:
    - Vous voyez l'avertissement OPENCLAW_PLUGIN_SDK_COMPAT_DEPRECATED
    - Vous voyez l'avertissement OPENCLAW_EXTENSION_API_DEPRECATED
    - Vous mettez à jour un plugin vers l'architecture de plugin moderne
    - Vous maintenez un plugin OpenClaw externe
sidebarTitle: Migrate to SDK
summary: Migrez depuis la couche héritée de compatibilité ascendante vers le Plugin SDK moderne
title: Migration du Plugin SDK
x-i18n:
    generated_at: "2026-04-07T06:53:25Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3691060e9dc00ca8bee49240a047f0479398691bd14fb96e9204cc9243fdb32c
    source_path: plugins/sdk-migration.md
    workflow: 15
---

# Migration du Plugin SDK

OpenClaw est passé d'une large couche de compatibilité ascendante à une architecture
de plugin moderne avec des importations ciblées et documentées. Si votre plugin a été créé avant
la nouvelle architecture, ce guide vous aide à migrer.

## Ce qui change

L'ancien système de plugins fournissait deux surfaces très ouvertes qui permettaient aux plugins d'importer
tout ce dont ils avaient besoin depuis un seul point d'entrée :

- **`openclaw/plugin-sdk/compat`** — une importation unique qui réexportait des dizaines de
  helpers. Elle a été introduite pour garder les anciens plugins basés sur des hooks fonctionnels pendant que la
  nouvelle architecture de plugin était en cours de construction.
- **`openclaw/extension-api`** — un pont qui donnait aux plugins un accès direct à
  des helpers côté hôte comme l'exécuteur d'agent intégré.

Ces deux surfaces sont maintenant **obsolètes**. Elles fonctionnent toujours à l'exécution, mais les nouveaux
plugins ne doivent pas les utiliser, et les plugins existants doivent migrer avant que la prochaine
version majeure ne les supprime.

<Warning>
  La couche de compatibilité ascendante sera supprimée dans une future version majeure.
  Les plugins qui importent encore depuis ces surfaces cesseront de fonctionner lorsque cela arrivera.
</Warning>

## Pourquoi cela a changé

L'ancienne approche causait des problèmes :

- **Démarrage lent** — importer un helper chargeait des dizaines de modules sans rapport
- **Dépendances circulaires** — les larges réexportations facilitaient la création de cycles d'importation
- **Surface API peu claire** — aucun moyen de savoir quelles exportations étaient stables ou internes

Le Plugin SDK moderne corrige cela : chaque chemin d'importation (`openclaw/plugin-sdk/\<subpath\>`)
est un petit module autonome avec un objectif clair et un contrat documenté.

Les points d'extension de commodité hérités des fournisseurs pour les canaux intégrés ont également disparu. Les importations
telles que `openclaw/plugin-sdk/slack`, `openclaw/plugin-sdk/discord`,
`openclaw/plugin-sdk/signal`, `openclaw/plugin-sdk/whatsapp`,
les points d'extension helper associés à une marque de canal, et
`openclaw/plugin-sdk/telegram-core` étaient des raccourcis privés de mono-dépôt, pas
des contrats de plugin stables. Utilisez plutôt des sous-chemins SDK génériques étroits. À l'intérieur du
workspace de plugins intégrés, conservez les helpers appartenant au fournisseur dans le propre
`api.ts` ou `runtime-api.ts` de ce plugin.

Exemples actuels de fournisseurs intégrés :

- Anthropic conserve les helpers de flux spécifiques à Claude dans son propre point d'extension `api.ts` /
  `contract-api.ts`
- OpenAI conserve les constructeurs de fournisseurs, les helpers de modèle par défaut et les constructeurs de fournisseurs
  realtime dans son propre `api.ts`
- OpenRouter conserve le constructeur de fournisseur et les helpers d'onboarding/configuration dans son propre
  `api.ts`

## Comment migrer

<Steps>
  <Step title="Auditer le comportement de repli de l'enveloppe Windows">
    Si votre plugin utilise `openclaw/plugin-sdk/windows-spawn`, les enveloppes Windows
    `.cmd`/`.bat` non résolues échouent maintenant en mode fermé, sauf si vous passez explicitement
    `allowShellFallback: true`.

    ```typescript
    // Avant
    const program = applyWindowsSpawnProgramPolicy({ candidate });

    // Après
    const program = applyWindowsSpawnProgramPolicy({
      candidate,
      // Définissez ceci uniquement pour les appelants de compatibilité de confiance qui
      // acceptent intentionnellement le repli via le shell.
      allowShellFallback: true,
    });
    ```

    Si votre appelant ne dépend pas intentionnellement du repli shell, ne définissez pas
    `allowShellFallback` et gérez plutôt l'erreur levée.

  </Step>

  <Step title="Trouver les importations obsolètes">
    Recherchez dans votre plugin les importations provenant de l'une ou l'autre surface obsolète :

    ```bash
    grep -r "plugin-sdk/compat" my-plugin/
    grep -r "openclaw/extension-api" my-plugin/
    ```

  </Step>

  <Step title="Remplacer par des importations ciblées">
    Chaque exportation de l'ancienne surface correspond à un chemin d'importation moderne spécifique :

    ```typescript
    // Avant (couche de compatibilité ascendante obsolète)
    import {
      createChannelReplyPipeline,
      createPluginRuntimeStore,
      resolveControlCommandGate,
    } from "openclaw/plugin-sdk/compat";

    // Après (importations modernes ciblées)
    import { createChannelReplyPipeline } from "openclaw/plugin-sdk/channel-reply-pipeline";
    import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
    import { resolveControlCommandGate } from "openclaw/plugin-sdk/command-auth";
    ```

    Pour les helpers côté hôte, utilisez le runtime de plugin injecté au lieu d'importer
    directement :

    ```typescript
    // Avant (pont extension-api obsolète)
    import { runEmbeddedPiAgent } from "openclaw/extension-api";
    const result = await runEmbeddedPiAgent({ sessionId, prompt });

    // Après (runtime injecté)
    const result = await api.runtime.agent.runEmbeddedPiAgent({ sessionId, prompt });
    ```

    Le même modèle s'applique aux autres helpers de pont hérités :

    | Ancienne importation | Équivalent moderne |
    | --- | --- |
    | `resolveAgentDir` | `api.runtime.agent.resolveAgentDir` |
    | `resolveAgentWorkspaceDir` | `api.runtime.agent.resolveAgentWorkspaceDir` |
    | `resolveAgentIdentity` | `api.runtime.agent.resolveAgentIdentity` |
    | `resolveThinkingDefault` | `api.runtime.agent.resolveThinkingDefault` |
    | `resolveAgentTimeoutMs` | `api.runtime.agent.resolveAgentTimeoutMs` |
    | `ensureAgentWorkspace` | `api.runtime.agent.ensureAgentWorkspace` |
    | helpers du magasin de sessions | `api.runtime.agent.session.*` |

  </Step>

  <Step title="Construire et tester">
    ```bash
    pnpm build
    pnpm test -- my-plugin/
    ```
  </Step>
</Steps>

## Référence des chemins d'importation

<Accordion title="Tableau des chemins d'importation courants">
  | Chemin d'importation | Objectif | Exportations clés |
  | --- | --- | --- |
  | `plugin-sdk/plugin-entry` | Helper canonique de point d'entrée de plugin | `definePluginEntry` |
  | `plugin-sdk/core` | Réexportation parapluie héritée pour les définitions/constructeurs d'entrée de canal | `defineChannelPluginEntry`, `createChatChannelPlugin` |
  | `plugin-sdk/config-schema` | Exportation du schéma de configuration racine | `OpenClawSchema` |
  | `plugin-sdk/provider-entry` | Helper de point d'entrée de fournisseur unique | `defineSingleProviderPluginEntry` |
  | `plugin-sdk/channel-core` | Définitions et constructeurs ciblés d'entrée de canal | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
  | `plugin-sdk/setup` | Helpers partagés d'assistant de configuration | Invites de liste d'autorisation, constructeurs d'état de configuration |
  | `plugin-sdk/setup-runtime` | Helpers de runtime au moment de la configuration | Adaptateurs de patch de configuration sûrs à l'importation, helpers de notes de recherche, `promptResolvedAllowFrom`, `splitSetupEntries`, proxys de configuration délégués |
  | `plugin-sdk/setup-adapter-runtime` | Helpers d'adaptateur de configuration | `createEnvPatchedAccountSetupAdapter` |
  | `plugin-sdk/setup-tools` | Helpers d'outillage de configuration | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
  | `plugin-sdk/account-core` | Helpers multicomptes | Helpers de liste/configuration/action-gate de compte |
  | `plugin-sdk/account-id` | Helpers d'ID de compte | `DEFAULT_ACCOUNT_ID`, normalisation de l'ID de compte |
  | `plugin-sdk/account-resolution` | Helpers de recherche de compte | Helpers de recherche de compte + repli par défaut |
  | `plugin-sdk/account-helpers` | Helpers étroits de compte | Helpers de liste de comptes/action de compte |
  | `plugin-sdk/channel-setup` | Adaptateurs d'assistant de configuration | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, plus `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
  | `plugin-sdk/channel-pairing` | Primitives d'appairage DM | `createChannelPairingController` |
  | `plugin-sdk/channel-reply-pipeline` | Câblage du préfixe de réponse + indicateur de saisie | `createChannelReplyPipeline` |
  | `plugin-sdk/channel-config-helpers` | Fabriques d'adaptateurs de configuration | `createHybridChannelConfigAdapter` |
  | `plugin-sdk/channel-config-schema` | Constructeurs de schéma de configuration | Types de schéma de configuration de canal |
  | `plugin-sdk/telegram-command-config` | Helpers de configuration de commande Telegram | Normalisation des noms de commande, trim des descriptions, validation des doublons/conflits |
  | `plugin-sdk/channel-policy` | Résolution de la politique groupes/DM | `resolveChannelGroupRequireMention` |
  | `plugin-sdk/channel-lifecycle` | Suivi d'état des comptes | `createAccountStatusSink` |
  | `plugin-sdk/inbound-envelope` | Helpers d'enveloppe entrante | Helpers partagés de route + construction d'enveloppe |
  | `plugin-sdk/inbound-reply-dispatch` | Helpers de réponse entrante | Helpers partagés d'enregistrement et de distribution |
  | `plugin-sdk/messaging-targets` | Analyse des cibles de messagerie | Helpers d'analyse/mise en correspondance des cibles |
  | `plugin-sdk/outbound-media` | Helpers de médias sortants | Chargement partagé des médias sortants |
  | `plugin-sdk/outbound-runtime` | Helpers de runtime sortant | Helpers délégués d'identité/envoi sortants |
  | `plugin-sdk/thread-bindings-runtime` | Helpers de liaison de fil | Cycle de vie des liaisons de fil et helpers d'adaptateur |
  | `plugin-sdk/agent-media-payload` | Helpers hérités de charge utile média | Constructeur de charge utile média d'agent pour les anciennes dispositions de champs |
  | `plugin-sdk/channel-runtime` | Shim de compatibilité obsolète | Utilitaires de runtime de canal hérités uniquement |
  | `plugin-sdk/channel-send-result` | Types de résultat d'envoi | Types de résultat de réponse |
  | `plugin-sdk/runtime-store` | Stockage persistant de plugin | `createPluginRuntimeStore` |
  | `plugin-sdk/runtime` | Helpers de runtime larges | Helpers de runtime/journalisation/sauvegarde/installation de plugin |
  | `plugin-sdk/runtime-env` | Helpers étroits d'environnement de runtime | Logger/env de runtime, helpers de délai d'attente, de nouvelle tentative et de backoff |
  | `plugin-sdk/plugin-runtime` | Helpers partagés de runtime de plugin | Helpers de commandes/hooks/http/interactif de plugin |
  | `plugin-sdk/hook-runtime` | Helpers de pipeline de hooks | Helpers partagés de pipeline de hooks webhook/internes |
  | `plugin-sdk/lazy-runtime` | Helpers de runtime paresseux | `createLazyRuntimeModule`, `createLazyRuntimeMethod`, `createLazyRuntimeMethodBinder`, `createLazyRuntimeNamedExport`, `createLazyRuntimeSurface` |
  | `plugin-sdk/process-runtime` | Helpers de processus | Helpers partagés d'exécution |
  | `plugin-sdk/cli-runtime` | Helpers de runtime CLI | Formatage des commandes, attentes, helpers de version |
  | `plugin-sdk/gateway-runtime` | Helpers de passerelle | Client de passerelle et helpers de patch d'état de canal |
  | `plugin-sdk/config-runtime` | Helpers de configuration | Helpers de chargement/écriture de configuration |
  | `plugin-sdk/telegram-command-config` | Helpers de commande Telegram | Helpers de validation des commandes Telegram stables en repli lorsque la surface de contrat Telegram intégrée n'est pas disponible |
  | `plugin-sdk/approval-runtime` | Helpers d'invite d'approbation | Charge utile d'approbation exec/plugin, helpers de capacité/profil d'approbation, helpers natifs de routage/runtime d'approbation |
  | `plugin-sdk/approval-auth-runtime` | Helpers d'authentification d'approbation | Résolution de l'approbateur, auth d'action dans le même chat |
  | `plugin-sdk/approval-client-runtime` | Helpers client d'approbation | Helpers natifs de profil/filtre d'approbation exec |
  | `plugin-sdk/approval-delivery-runtime` | Helpers de distribution d'approbation | Adaptateurs natifs de capacité/distribution d'approbation |
  | `plugin-sdk/approval-native-runtime` | Helpers de cible d'approbation | Helpers natifs de cible d'approbation/liaison de compte |
  | `plugin-sdk/approval-reply-runtime` | Helpers de réponse d'approbation | Helpers de charge utile de réponse d'approbation exec/plugin |
  | `plugin-sdk/security-runtime` | Helpers de sécurité | Helpers partagés de confiance, de limitation DM, de contenu externe et de collecte de secrets |
  | `plugin-sdk/ssrf-policy` | Helpers de politique SSRF | Helpers de liste d'autorisation d'hôtes et de politique de réseau privé |
  | `plugin-sdk/ssrf-runtime` | Helpers de runtime SSRF | Helpers de pinned-dispatcher, fetch protégé, politique SSRF |
  | `plugin-sdk/collection-runtime` | Helpers de cache borné | `pruneMapToMaxSize` |
  | `plugin-sdk/diagnostic-runtime` | Helpers de contrôle de diagnostic | `isDiagnosticFlagEnabled`, `isDiagnosticsEnabled` |
  | `plugin-sdk/error-runtime` | Helpers de formatage d'erreurs | `formatUncaughtError`, `isApprovalNotFoundError`, helpers de graphe d'erreurs |
  | `plugin-sdk/fetch-runtime` | Helpers de fetch/proxy enveloppés | `resolveFetch`, helpers de proxy |
  | `plugin-sdk/host-runtime` | Helpers de normalisation d'hôte | `normalizeHostname`, `normalizeScpRemoteHost` |
  | `plugin-sdk/retry-runtime` | Helpers de nouvelle tentative | `RetryConfig`, `retryAsync`, exécuteurs de politiques |
  | `plugin-sdk/allow-from` | Formatage de liste d'autorisation | `formatAllowFromLowercase` |
  | `plugin-sdk/allowlist-resolution` | Mappage d'entrée de liste d'autorisation | `mapAllowlistResolutionInputs` |
  | `plugin-sdk/command-auth` | Helpers de contrôle de commandes et de surface de commande | `resolveControlCommandGate`, helpers d'autorisation de l'expéditeur, helpers de registre de commandes |
  | `plugin-sdk/secret-input` | Analyse d'entrée secrète | Helpers d'entrée secrète |
  | `plugin-sdk/webhook-ingress` | Helpers de requête webhook | Utilitaires de cible webhook |
  | `plugin-sdk/webhook-request-guards` | Helpers de garde de corps de requête webhook | Helpers de lecture/limite du corps de requête |
  | `plugin-sdk/reply-runtime` | Runtime partagé de réponse | Distribution entrante, heartbeat, planificateur de réponse, fragmentation |
  | `plugin-sdk/reply-dispatch-runtime` | Helpers étroits de distribution de réponse | Helpers de finalisation + distribution fournisseur |
  | `plugin-sdk/reply-history` | Helpers d'historique de réponse | `buildHistoryContext`, `buildPendingHistoryContextFromMap`, `recordPendingHistoryEntry`, `clearHistoryEntriesIfEnabled` |
  | `plugin-sdk/reply-reference` | Planification de référence de réponse | `createReplyReferencePlanner` |
  | `plugin-sdk/reply-chunking` | Helpers de fragmentation de réponse | Helpers de fragmentation texte/markdown |
  | `plugin-sdk/session-store-runtime` | Helpers de magasin de sessions | Helpers de chemin du magasin + updated-at |
  | `plugin-sdk/state-paths` | Helpers de chemins d'état | Helpers de répertoire d'état et OAuth |
  | `plugin-sdk/routing` | Helpers de routage/clé de session | `resolveAgentRoute`, `buildAgentSessionKey`, `resolveDefaultAgentBoundAccountId`, helpers de normalisation de clé de session |
  | `plugin-sdk/status-helpers` | Helpers d'état de canal | Constructeurs de résumés d'état de canal/compte, valeurs par défaut d'état runtime, helpers de métadonnées de problème |
  | `plugin-sdk/target-resolver-runtime` | Helpers de résolveur de cible | Helpers partagés de résolveur de cible |
  | `plugin-sdk/string-normalization-runtime` | Helpers de normalisation de chaîne | Helpers de normalisation slug/chaîne |
  | `plugin-sdk/request-url` | Helpers d'URL de requête | Extraire des URL chaîne depuis des entrées de type requête |
  | `plugin-sdk/run-command` | Helpers de commande temporisée | Exécuteur de commande temporisée avec stdout/stderr normalisés |
  | `plugin-sdk/param-readers` | Lecteurs de paramètres | Lecteurs de paramètres communs d'outil/CLI |
  | `plugin-sdk/tool-send` | Extraction d'envoi d'outil | Extraire les champs de cible d'envoi canoniques depuis les arguments d'outil |
  | `plugin-sdk/temp-path` | Helpers de chemin temporaire | Helpers partagés de chemin temporaire de téléchargement |
  | `plugin-sdk/logging-core` | Helpers de journalisation | Logger de sous-système et helpers de masquage |
  | `plugin-sdk/markdown-table-runtime` | Helpers de tableau Markdown | Helpers de mode tableau Markdown |
  | `plugin-sdk/reply-payload` | Types de réponse de message | Types de charge utile de réponse |
  | `plugin-sdk/provider-setup` | Helpers organisés de configuration de fournisseur local/autohébergé | Helpers de découverte/configuration de fournisseur autohébergé |
  | `plugin-sdk/self-hosted-provider-setup` | Helpers ciblés de configuration de fournisseur autohébergé compatible OpenAI | Les mêmes helpers de découverte/configuration de fournisseur autohébergé |
  | `plugin-sdk/provider-auth-runtime` | Helpers d'authentification runtime de fournisseur | Helpers de résolution de clé API à l'exécution |
  | `plugin-sdk/provider-auth-api-key` | Helpers de configuration de clé API de fournisseur | Helpers d'onboarding/écriture de profil de clé API |
  | `plugin-sdk/provider-auth-result` | Helpers de résultat d'authentification de fournisseur | Constructeur standard de résultat d'authentification OAuth |
  | `plugin-sdk/provider-auth-login` | Helpers de connexion interactive de fournisseur | Helpers partagés de connexion interactive |
  | `plugin-sdk/provider-env-vars` | Helpers de variables d'environnement de fournisseur | Helpers de recherche de variables d'environnement d'authentification de fournisseur |
  | `plugin-sdk/provider-model-shared` | Helpers partagés de modèle/rejeu de fournisseur | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, constructeurs partagés de politique de rejeu, helpers de point de terminaison de fournisseur et helpers de normalisation d'ID de modèle |
  | `plugin-sdk/provider-catalog-shared` | Helpers partagés de catalogue de fournisseur | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
  | `plugin-sdk/provider-onboard` | Patches d'onboarding de fournisseur | Helpers de configuration d'onboarding |
  | `plugin-sdk/provider-http` | Helpers HTTP de fournisseur | Helpers génériques HTTP/capacité de point de terminaison de fournisseur |
  | `plugin-sdk/provider-web-fetch` | Helpers web-fetch de fournisseur | Helpers d'enregistrement/cache de fournisseur web-fetch |
  | `plugin-sdk/provider-web-search-contract` | Helpers de contrat web-search de fournisseur | Helpers étroits de contrat de configuration/identifiants web-search tels que `enablePluginInConfig`, `resolveProviderWebSearchPluginConfig` et setters/getters d'identifiants limités |
  | `plugin-sdk/provider-web-search` | Helpers web-search de fournisseur | Helpers d'enregistrement/cache/runtime de fournisseur web-search |
  | `plugin-sdk/provider-tools` | Helpers de compatibilité d'outil/schéma de fournisseur | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, nettoyage de schéma Gemini + diagnostics, et helpers de compatibilité xAI tels que `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
  | `plugin-sdk/provider-usage` | Helpers d'usage de fournisseur | `fetchClaudeUsage`, `fetchGeminiUsage`, `fetchGithubCopilotUsage` et autres helpers d'usage de fournisseur |
  | `plugin-sdk/provider-stream` | Helpers d'enveloppe de flux de fournisseur | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, types d'enveloppe de flux, et helpers partagés d'enveloppe Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot |
  | `plugin-sdk/keyed-async-queue` | File asynchrone ordonnée | `KeyedAsyncQueue` |
  | `plugin-sdk/media-runtime` | Helpers partagés de médias | Helpers de fetch/transform/store de médias plus constructeurs de charge utile média |
  | `plugin-sdk/media-generation-runtime` | Helpers partagés de génération de médias | Helpers partagés de repli, sélection de candidats et messages de modèle manquant pour la génération d'image/vidéo/musique |
  | `plugin-sdk/media-understanding` | Helpers de compréhension des médias | Types de fournisseur de compréhension des médias plus exportations helper image/audio orientées fournisseur |
  | `plugin-sdk/text-runtime` | Helpers partagés de texte | Suppression de texte visible par l'assistant, helpers de rendu/fragmentation/tableau markdown, helpers de masquage, helpers de balises de directive, utilitaires de texte sûr et helpers connexes de texte/journalisation |
  | `plugin-sdk/text-chunking` | Helpers de fragmentation de texte | Helper de fragmentation de texte sortant |
  | `plugin-sdk/speech` | Helpers de parole | Types de fournisseur de parole plus helpers orientés fournisseur pour directives, registre et validation |
  | `plugin-sdk/speech-core` | Cœur partagé de parole | Types de fournisseur de parole, registre, directives, normalisation |
  | `plugin-sdk/realtime-transcription` | Helpers de transcription en temps réel | Types de fournisseur et helpers de registre |
  | `plugin-sdk/realtime-voice` | Helpers de voix en temps réel | Types de fournisseur et helpers de registre |
  | `plugin-sdk/image-generation-core` | Cœur partagé de génération d'image | Helpers de types, repli, auth et registre de génération d'image |
  | `plugin-sdk/music-generation` | Helpers de génération musicale | Types de fournisseur/requête/résultat de génération musicale |
  | `plugin-sdk/music-generation-core` | Cœur partagé de génération musicale | Helpers de types, repli, recherche de fournisseur et analyse de référence de modèle de génération musicale |
  | `plugin-sdk/video-generation` | Helpers de génération vidéo | Types de fournisseur/requête/résultat de génération vidéo |
  | `plugin-sdk/video-generation-core` | Cœur partagé de génération vidéo | Helpers de types, repli, recherche de fournisseur et analyse de référence de modèle de génération vidéo |
  | `plugin-sdk/interactive-runtime` | Helpers de réponse interactive | Normalisation/réduction de charge utile de réponse interactive |
  | `plugin-sdk/channel-config-primitives` | Primitives de configuration de canal | Primitives étroites de schéma de configuration de canal |
  | `plugin-sdk/channel-config-writes` | Helpers d'écriture de configuration de canal | Helpers d'autorisation d'écriture de configuration de canal |
  | `plugin-sdk/channel-plugin-common` | Prélude partagé de plugin de canal | Exportations partagées de prélude de plugin de canal |
  | `plugin-sdk/channel-status` | Helpers d'état de canal | Helpers partagés d'instantané/résumé d'état de canal |
  | `plugin-sdk/allowlist-config-edit` | Helpers de configuration de liste d'autorisation | Helpers de lecture/édition de configuration de liste d'autorisation |
  | `plugin-sdk/group-access` | Helpers d'accès aux groupes | Helpers partagés de décision d'accès aux groupes |
  | `plugin-sdk/direct-dm` | Helpers de DM direct | Helpers partagés d'authentification/garde DM direct |
  | `plugin-sdk/extension-shared` | Helpers partagés d'extension | Primitives de canal/état passif et helper proxy ambiant |
  | `plugin-sdk/webhook-targets` | Helpers de cible webhook | Registre de cibles webhook et helpers d'installation de route |
  | `plugin-sdk/webhook-path` | Helpers de chemin webhook | Helpers de normalisation de chemin webhook |
  | `plugin-sdk/web-media` | Helpers partagés de médias web | Helpers de chargement de médias distants/locaux |
  | `plugin-sdk/zod` | Réexportation Zod | `zod` réexporté pour les consommateurs du Plugin SDK |
  | `plugin-sdk/memory-core` | Helpers intégrés memory-core | Surface helper memory manager/config/file/CLI |
  | `plugin-sdk/memory-core-engine-runtime` | Façade runtime du moteur mémoire | Façade runtime d'indexation/recherche mémoire |
  | `plugin-sdk/memory-core-host-engine-foundation` | Moteur foundation de l'hôte mémoire | Exportations du moteur foundation de l'hôte mémoire |
  | `plugin-sdk/memory-core-host-engine-embeddings` | Moteur d'embeddings de l'hôte mémoire | Exportations du moteur d'embeddings de l'hôte mémoire |
  | `plugin-sdk/memory-core-host-engine-qmd` | Moteur QMD de l'hôte mémoire | Exportations du moteur QMD de l'hôte mémoire |
  | `plugin-sdk/memory-core-host-engine-storage` | Moteur de stockage de l'hôte mémoire | Exportations du moteur de stockage de l'hôte mémoire |
  | `plugin-sdk/memory-core-host-multimodal` | Helpers multimodaux de l'hôte mémoire | Helpers multimodaux de l'hôte mémoire |
  | `plugin-sdk/memory-core-host-query` | Helpers de requête de l'hôte mémoire | Helpers de requête de l'hôte mémoire |
  | `plugin-sdk/memory-core-host-secret` | Helpers secrets de l'hôte mémoire | Helpers secrets de l'hôte mémoire |
  | `plugin-sdk/memory-core-host-events` | Helpers de journal d'événements de l'hôte mémoire | Helpers de journal d'événements de l'hôte mémoire |
  | `plugin-sdk/memory-core-host-status` | Helpers d'état de l'hôte mémoire | Helpers d'état de l'hôte mémoire |
  | `plugin-sdk/memory-core-host-runtime-cli` | Runtime CLI de l'hôte mémoire | Helpers de runtime CLI de l'hôte mémoire |
  | `plugin-sdk/memory-core-host-runtime-core` | Runtime cœur de l'hôte mémoire | Helpers de runtime cœur de l'hôte mémoire |
  | `plugin-sdk/memory-core-host-runtime-files` | Helpers de fichiers/runtime de l'hôte mémoire | Helpers de fichiers/runtime de l'hôte mémoire |
  | `plugin-sdk/memory-host-core` | Alias runtime cœur de l'hôte mémoire | Alias neutre vis-à-vis du fournisseur pour les helpers de runtime cœur de l'hôte mémoire |
  | `plugin-sdk/memory-host-events` | Alias journal d'événements de l'hôte mémoire | Alias neutre vis-à-vis du fournisseur pour les helpers de journal d'événements de l'hôte mémoire |
  | `plugin-sdk/memory-host-files` | Alias fichier/runtime de l'hôte mémoire | Alias neutre vis-à-vis du fournisseur pour les helpers de fichier/runtime de l'hôte mémoire |
  | `plugin-sdk/memory-host-markdown` | Helpers markdown géré | Helpers partagés de markdown géré pour les plugins proches de la mémoire |
  | `plugin-sdk/memory-host-search` | Façade active de recherche mémoire | Façade runtime paresseuse du gestionnaire de recherche en mémoire active |
  | `plugin-sdk/memory-host-status` | Alias d'état de l'hôte mémoire | Alias neutre vis-à-vis du fournisseur pour les helpers d'état de l'hôte mémoire |
  | `plugin-sdk/memory-lancedb` | Helpers intégrés memory-lancedb | Surface helper memory-lancedb |
  | `plugin-sdk/testing` | Utilitaires de test | Helpers et mocks de test |
</Accordion>

Ce tableau est volontairement le sous-ensemble courant pour la migration, et non la surface complète du SDK.
La liste complète des plus de 200 points d'entrée se trouve dans
`scripts/lib/plugin-sdk-entrypoints.json`.

Cette liste comprend encore certains points d'extension helper de plugins intégrés tels que
`plugin-sdk/feishu`, `plugin-sdk/feishu-setup`, `plugin-sdk/zalo`,
`plugin-sdk/zalo-setup`, et `plugin-sdk/matrix*`. Ils restent exportés pour
la maintenance et la compatibilité des plugins intégrés, mais ils sont volontairement
omis du tableau courant de migration et ne constituent pas la cible recommandée pour
le nouveau code de plugin.

La même règle s'applique aux autres familles de helpers intégrés telles que :

- helpers de prise en charge du navigateur : `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support`
- Matrix : `plugin-sdk/matrix*`
- LINE : `plugin-sdk/line*`
- IRC : `plugin-sdk/irc*`
- surfaces de plugin/helper intégrées telles que `plugin-sdk/googlechat`,
  `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles*`,
  `plugin-sdk/mattermost*`, `plugin-sdk/msteams`,
  `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`,
  `plugin-sdk/twitch`,
  `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`,
  `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`,
  `plugin-sdk/thread-ownership`, et `plugin-sdk/voice-call`

`plugin-sdk/github-copilot-token` expose actuellement la surface étroite de helper de jeton
`DEFAULT_COPILOT_API_BASE_URL`,
`deriveCopilotApiBaseUrlFromToken`, et `resolveCopilotApiToken`.

Utilisez l'importation la plus étroite qui correspond au besoin. Si vous ne trouvez pas une exportation,
consultez le code source dans `src/plugin-sdk/` ou demandez sur Discord.

## Calendrier de suppression

| Quand | Ce qui se passe |
| ---------------------- | ----------------------------------------------------------------------- |
| **Maintenant** | Les surfaces obsolètes émettent des avertissements à l'exécution |
| **Prochaine version majeure** | Les surfaces obsolètes seront supprimées ; les plugins qui les utilisent encore échoueront |

Tous les plugins du cœur ont déjà été migrés. Les plugins externes doivent migrer
avant la prochaine version majeure.

## Supprimer temporairement les avertissements

Définissez ces variables d'environnement pendant que vous travaillez sur la migration :

```bash
OPENCLAW_SUPPRESS_PLUGIN_SDK_COMPAT_WARNING=1 openclaw gateway run
OPENCLAW_SUPPRESS_EXTENSION_API_WARNING=1 openclaw gateway run
```

Il s'agit d'une échappatoire temporaire, pas d'une solution permanente.

## Liens associés

- [Premiers pas](/fr/plugins/building-plugins) — créer votre premier plugin
- [Vue d'ensemble du SDK](/fr/plugins/sdk-overview) — référence complète des importations par sous-chemin
- [Plugins de canal](/fr/plugins/sdk-channel-plugins) — création de plugins de canal
- [Plugins de fournisseur](/fr/plugins/sdk-provider-plugins) — création de plugins de fournisseur
- [Internals des plugins](/fr/plugins/architecture) — analyse approfondie de l'architecture
- [Manifeste du plugin](/fr/plugins/manifest) — référence du schéma de manifeste
