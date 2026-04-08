---
read_when:
    - Vous voyez l'avertissement OPENCLAW_PLUGIN_SDK_COMPAT_DEPRECATED
    - Vous voyez l'avertissement OPENCLAW_EXTENSION_API_DEPRECATED
    - Vous mettez à jour un plugin vers l'architecture de plugin moderne
    - Vous maintenez un plugin OpenClaw externe
sidebarTitle: Migrate to SDK
summary: Migrer de la couche héritée de compatibilité descendante vers le Plugin SDK moderne
title: Migration du Plugin SDK
x-i18n:
    generated_at: "2026-04-08T02:17:31Z"
    model: gpt-5.4
    provider: openai
    source_hash: 155a8b14bc345319c8516ebdb8a0ccdea2c5f7fa07dad343442996daee21ecad
    source_path: plugins/sdk-migration.md
    workflow: 15
---

# Migration du Plugin SDK

OpenClaw est passé d'une large couche de compatibilité descendante à une architecture de plugin
moderne avec des imports ciblés et documentés. Si votre plugin a été créé avant
la nouvelle architecture, ce guide vous aidera à migrer.

## Ce qui change

L'ancien système de plugins fournissait deux surfaces très ouvertes qui permettaient aux plugins d'importer
tout ce dont ils avaient besoin à partir d'un point d'entrée unique :

- **`openclaw/plugin-sdk/compat`** — un import unique qui réexportait des dizaines de
  helpers. Il a été introduit pour maintenir le fonctionnement des anciens plugins basés sur des hooks pendant la
  construction de la nouvelle architecture de plugin.
- **`openclaw/extension-api`** — un pont qui donnait aux plugins un accès direct à
  des helpers côté hôte comme le runner d'agent embarqué.

Ces deux surfaces sont désormais **obsolètes**. Elles fonctionnent toujours au runtime, mais les nouveaux
plugins ne doivent pas les utiliser, et les plugins existants doivent migrer avant que la prochaine
version majeure ne les supprime.

<Warning>
  La couche de compatibilité descendante sera supprimée dans une future version majeure.
  Les plugins qui importent encore depuis ces surfaces cesseront de fonctionner à ce moment-là.
</Warning>

## Pourquoi cela a changé

L'ancienne approche causait des problèmes :

- **Démarrage lent** — importer un helper chargeait des dizaines de modules sans rapport
- **Dépendances circulaires** — les réexportations larges facilitaient la création de cycles d'import
- **Surface d'API floue** — aucun moyen de distinguer les exports stables des exports internes

Le Plugin SDK moderne corrige cela : chaque chemin d'import (`openclaw/plugin-sdk/\<subpath\>`)
est un module petit et autonome, avec un objectif clair et un contrat documenté.

Les points d'extension de commodité hérités pour les fournisseurs des canaux groupés ont également disparu. Les imports
tels que `openclaw/plugin-sdk/slack`, `openclaw/plugin-sdk/discord`,
`openclaw/plugin-sdk/signal`, `openclaw/plugin-sdk/whatsapp`,
les points d'extension de helpers propres à un canal, et
`openclaw/plugin-sdk/telegram-core` étaient des raccourcis privés du mono-repo, pas
des contrats de plugin stables. Utilisez à la place des sous-chemins SDK génériques étroits. À l'intérieur de
l'espace de travail des plugins groupés, conservez les helpers gérés par le fournisseur dans le
propre `api.ts` ou `runtime-api.ts` de ce plugin.

Exemples actuels de fournisseurs groupés :

- Anthropic conserve les helpers de flux spécifiques à Claude dans son propre point d'extension `api.ts` /
  `contract-api.ts`
- OpenAI conserve les builders de fournisseur, les helpers de modèle par défaut et les builders de fournisseur
  temps réel dans son propre `api.ts`
- OpenRouter conserve les helpers de builder de fournisseur et d'onboarding/configuration dans son propre
  `api.ts`

## Comment migrer

<Steps>
  <Step title="Migrer les handlers natifs d'approbation vers des faits de capacité">
    Les plugins de canal capables de gérer les approbations exposent désormais le comportement natif d'approbation via
    `approvalCapability.nativeRuntime` plus le registre partagé de contexte runtime.

    Changements clés :

    - Remplacez `approvalCapability.handler.loadRuntime(...)` par
      `approvalCapability.nativeRuntime`
    - Déplacez l'authentification/la livraison spécifiques aux approbations hors du câblage hérité `plugin.auth` /
      `plugin.approvals` et vers `approvalCapability`
    - `ChannelPlugin.approvals` a été supprimé du contrat public des plugins de canal ;
      déplacez les champs de livraison/natif/rendu vers `approvalCapability`
    - `plugin.auth` reste utilisé uniquement pour les flux de connexion/déconnexion de canal ; les hooks
      d'authentification d'approbation présents à cet endroit ne sont plus lus par le cœur
    - Enregistrez les objets runtime possédés par le canal comme les clients, jetons ou applications
      Bolt via `openclaw/plugin-sdk/channel-runtime-context`
    - N'envoyez pas d'avis de redirection gérés par le plugin depuis les handlers natifs d'approbation ;
      le cœur gère désormais les avis « routé ailleurs » à partir des résultats réels de livraison
    - Lors du passage de `channelRuntime` à `createChannelManager(...)`, fournissez une
      vraie surface `createPluginRuntime().channel`. Les stubs partiels sont rejetés.

    Voir `/plugins/sdk-channel-plugins` pour la disposition actuelle de la
    capacité d'approbation.

  </Step>

  <Step title="Auditer le comportement de repli des wrappers Windows">
    Si votre plugin utilise `openclaw/plugin-sdk/windows-spawn`, les wrappers Windows
    `.cmd`/`.bat` non résolus échouent désormais de manière fermée sauf si vous passez explicitement
    `allowShellFallback: true`.

    ```typescript
    // Avant
    const program = applyWindowsSpawnProgramPolicy({ candidate });

    // Après
    const program = applyWindowsSpawnProgramPolicy({
      candidate,
      // Définissez ceci uniquement pour les appelants de compatibilité de confiance qui
      // acceptent intentionnellement le repli médié par le shell.
      allowShellFallback: true,
    });
    ```

    Si votre appelant ne dépend pas intentionnellement du repli shell, ne définissez pas
    `allowShellFallback` et gérez plutôt l'erreur levée.

  </Step>

  <Step title="Trouver les imports obsolètes">
    Recherchez dans votre plugin les imports depuis l'une ou l'autre surface obsolète :

    ```bash
    grep -r "plugin-sdk/compat" my-plugin/
    grep -r "openclaw/extension-api" my-plugin/
    ```

  </Step>

  <Step title="Remplacer par des imports ciblés">
    Chaque export de l'ancienne surface correspond à un chemin d'import moderne spécifique :

    ```typescript
    // Avant (couche obsolète de compatibilité descendante)
    import {
      createChannelReplyPipeline,
      createPluginRuntimeStore,
      resolveControlCommandGate,
    } from "openclaw/plugin-sdk/compat";

    // Après (imports modernes ciblés)
    import { createChannelReplyPipeline } from "openclaw/plugin-sdk/channel-reply-pipeline";
    import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
    import { resolveControlCommandGate } from "openclaw/plugin-sdk/command-auth";
    ```

    Pour les helpers côté hôte, utilisez le runtime de plugin injecté au lieu d'importer
    directement :

    ```typescript
    // Avant (pont extension-api obsolète)
    import { runEmbeddedPiAgent } from "openclaw/extension-api";
    const result = await runEmbeddedPiAgent({ sessionId, prompt });

    // Après (runtime injecté)
    const result = await api.runtime.agent.runEmbeddedPiAgent({ sessionId, prompt });
    ```

    Le même modèle s'applique aux autres helpers hérités du pont :

    | Ancien import | Équivalent moderne |
    | --- | --- |
    | `resolveAgentDir` | `api.runtime.agent.resolveAgentDir` |
    | `resolveAgentWorkspaceDir` | `api.runtime.agent.resolveAgentWorkspaceDir` |
    | `resolveAgentIdentity` | `api.runtime.agent.resolveAgentIdentity` |
    | `resolveThinkingDefault` | `api.runtime.agent.resolveThinkingDefault` |
    | `resolveAgentTimeoutMs` | `api.runtime.agent.resolveAgentTimeoutMs` |
    | `ensureAgentWorkspace` | `api.runtime.agent.ensureAgentWorkspace` |
    | helpers de magasin de session | `api.runtime.agent.session.*` |

  </Step>

  <Step title="Compiler et tester">
    ```bash
    pnpm build
    pnpm test -- my-plugin/
    ```
  </Step>
</Steps>

## Référence des chemins d'import

<Accordion title="Tableau des chemins d'import courants">
  | Chemin d'import | Usage | Exports clés |
  | --- | --- | --- |
  | `plugin-sdk/plugin-entry` | Helper canonique de point d'entrée de plugin | `definePluginEntry` |
  | `plugin-sdk/core` | Réexportation parapluie héritée pour les définitions/builders d'entrée de canal | `defineChannelPluginEntry`, `createChatChannelPlugin` |
  | `plugin-sdk/config-schema` | Export du schéma de configuration racine | `OpenClawSchema` |
  | `plugin-sdk/provider-entry` | Helper de point d'entrée à fournisseur unique | `defineSingleProviderPluginEntry` |
  | `plugin-sdk/channel-core` | Définitions et builders ciblés d'entrée de canal | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
  | `plugin-sdk/setup` | Helpers partagés de l'assistant de configuration | Prompts de liste d'autorisation, builders d'état de configuration |
  | `plugin-sdk/setup-runtime` | Helpers runtime au moment de la configuration | Adaptateurs de patch de configuration sûrs à importer, helpers de notes de recherche, `promptResolvedAllowFrom`, `splitSetupEntries`, proxys de configuration délégués |
  | `plugin-sdk/setup-adapter-runtime` | Helpers d'adaptateur de configuration | `createEnvPatchedAccountSetupAdapter` |
  | `plugin-sdk/setup-tools` | Helpers d'outillage de configuration | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
  | `plugin-sdk/account-core` | Helpers multi-comptes | Helpers de liste de comptes/configuration/contrôle des actions |
  | `plugin-sdk/account-id` | Helpers d'identifiant de compte | `DEFAULT_ACCOUNT_ID`, normalisation des identifiants de compte |
  | `plugin-sdk/account-resolution` | Helpers de recherche de compte | Helpers de recherche de compte + repli par défaut |
  | `plugin-sdk/account-helpers` | Helpers étroits de compte | Helpers de liste de comptes/actions sur les comptes |
  | `plugin-sdk/channel-setup` | Adaptateurs de l'assistant de configuration | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, plus `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
  | `plugin-sdk/channel-pairing` | Primitives d'appairage DM | `createChannelPairingController` |
  | `plugin-sdk/channel-reply-pipeline` | Câblage du préfixe de réponse + indicateur de frappe | `createChannelReplyPipeline` |
  | `plugin-sdk/channel-config-helpers` | Fabriques d'adaptateurs de configuration | `createHybridChannelConfigAdapter` |
  | `plugin-sdk/channel-config-schema` | Builders de schéma de configuration | Types de schéma de configuration de canal |
  | `plugin-sdk/telegram-command-config` | Helpers de configuration des commandes Telegram | Normalisation des noms de commandes, nettoyage des descriptions, validation des doublons/conflits |
  | `plugin-sdk/channel-policy` | Résolution des politiques de groupe/DM | `resolveChannelGroupRequireMention` |
  | `plugin-sdk/channel-lifecycle` | Suivi d'état des comptes | `createAccountStatusSink` |
  | `plugin-sdk/inbound-envelope` | Helpers d'enveloppe entrante | Helpers partagés de route + construction d'enveloppe |
  | `plugin-sdk/inbound-reply-dispatch` | Helpers de réponse entrante | Helpers partagés d'enregistrement et d'envoi |
  | `plugin-sdk/messaging-targets` | Analyse des cibles de messagerie | Helpers d'analyse/correspondance des cibles |
  | `plugin-sdk/outbound-media` | Helpers de média sortant | Chargement partagé des médias sortants |
  | `plugin-sdk/outbound-runtime` | Helpers runtime sortants | Helpers de délégation d'identité/envoi sortants |
  | `plugin-sdk/thread-bindings-runtime` | Helpers de liaison de threads | Helpers de cycle de vie et d'adaptateur des liaisons de threads |
  | `plugin-sdk/agent-media-payload` | Helpers hérités de charge utile média | Builder de charge utile média d'agent pour les dispositions de champs héritées |
  | `plugin-sdk/channel-runtime` | Shim de compatibilité obsolète | Uniquement des utilitaires hérités de runtime de canal |
  | `plugin-sdk/channel-send-result` | Types de résultat d'envoi | Types de résultat de réponse |
  | `plugin-sdk/runtime-store` | Stockage persistant de plugin | `createPluginRuntimeStore` |
  | `plugin-sdk/runtime` | Helpers runtime larges | Helpers de runtime/journalisation/sauvegarde/installation de plugin |
  | `plugin-sdk/runtime-env` | Helpers étroits d'environnement runtime | Journalisation/env runtime, délai d'attente, retry et backoff |
  | `plugin-sdk/plugin-runtime` | Helpers runtime partagés de plugin | Helpers de commandes/hooks/http/interactif pour plugins |
  | `plugin-sdk/hook-runtime` | Helpers de pipeline de hooks | Helpers partagés de pipeline de hook webhook/interne |
  | `plugin-sdk/lazy-runtime` | Helpers runtime paresseux | `createLazyRuntimeModule`, `createLazyRuntimeMethod`, `createLazyRuntimeMethodBinder`, `createLazyRuntimeNamedExport`, `createLazyRuntimeSurface` |
  | `plugin-sdk/process-runtime` | Helpers de processus | Helpers partagés d'exécution |
  | `plugin-sdk/cli-runtime` | Helpers runtime CLI | Formatage des commandes, attentes, helpers de version |
  | `plugin-sdk/gateway-runtime` | Helpers de passerelle | Client de passerelle et helpers de patch de statut de canal |
  | `plugin-sdk/config-runtime` | Helpers de configuration | Helpers de chargement/écriture de configuration |
  | `plugin-sdk/telegram-command-config` | Helpers de commandes Telegram | Helpers de validation de commandes Telegram stables en repli quand la surface de contrat Telegram groupée n'est pas disponible |
  | `plugin-sdk/approval-runtime` | Helpers de prompts d'approbation | Charge utile d'approbation exec/plugin, helpers de profil/capacité d'approbation, helpers natifs de routage/runtime d'approbation |
  | `plugin-sdk/approval-auth-runtime` | Helpers d'authentification d'approbation | Résolution des approbateurs, authentification d'action dans le même chat |
  | `plugin-sdk/approval-client-runtime` | Helpers de client d'approbation | Helpers natifs de profil/filtre d'approbation exec |
  | `plugin-sdk/approval-delivery-runtime` | Helpers de livraison d'approbation | Adaptateurs natifs de capacité/livraison d'approbation |
  | `plugin-sdk/approval-gateway-runtime` | Helpers de passerelle d'approbation | Helper partagé de résolution de passerelle d'approbation |
  | `plugin-sdk/approval-handler-adapter-runtime` | Helpers d'adaptateur d'approbation | Helpers légers de chargement d'adaptateurs natifs d'approbation pour les points d'entrée de canal sensibles au temps de chargement |
  | `plugin-sdk/approval-handler-runtime` | Helpers de handler d'approbation | Helpers runtime plus larges pour les handlers d'approbation ; préférez les points d'extension plus étroits adaptateur/passerelle lorsqu'ils suffisent |
  | `plugin-sdk/approval-native-runtime` | Helpers de cible d'approbation | Helpers natifs de liaison cible/compte pour approbation |
  | `plugin-sdk/approval-reply-runtime` | Helpers de réponse d'approbation | Helpers de charge utile de réponse d'approbation exec/plugin |
  | `plugin-sdk/channel-runtime-context` | Helpers de contexte runtime de canal | Helpers génériques d'enregistrement/lecture/observation du contexte runtime de canal |
  | `plugin-sdk/security-runtime` | Helpers de sécurité | Helpers partagés de confiance, contrôle DM, contenu externe et collecte de secrets |
  | `plugin-sdk/ssrf-policy` | Helpers de politique SSRF | Helpers de liste d'autorisation d'hôtes et de politique de réseau privé |
  | `plugin-sdk/ssrf-runtime` | Helpers runtime SSRF | Helpers de dispatcher épinglé, fetch protégé, politique SSRF |
  | `plugin-sdk/collection-runtime` | Helpers de cache borné | `pruneMapToMaxSize` |
  | `plugin-sdk/diagnostic-runtime` | Helpers de contrôle des diagnostics | `isDiagnosticFlagEnabled`, `isDiagnosticsEnabled` |
  | `plugin-sdk/error-runtime` | Helpers de formatage des erreurs | `formatUncaughtError`, `isApprovalNotFoundError`, helpers de graphe d'erreurs |
  | `plugin-sdk/fetch-runtime` | Helpers de fetch/proxy encapsulés | `resolveFetch`, helpers de proxy |
  | `plugin-sdk/host-runtime` | Helpers de normalisation d'hôte | `normalizeHostname`, `normalizeScpRemoteHost` |
  | `plugin-sdk/retry-runtime` | Helpers de retry | `RetryConfig`, `retryAsync`, exécuteurs de politique |
  | `plugin-sdk/allow-from` | Formatage de liste d'autorisation | `formatAllowFromLowercase` |
  | `plugin-sdk/allowlist-resolution` | Mappage des entrées de liste d'autorisation | `mapAllowlistResolutionInputs` |
  | `plugin-sdk/command-auth` | Contrôle des commandes et helpers de surface de commande | `resolveControlCommandGate`, helpers d'autorisation d'expéditeur, helpers de registre de commandes |
  | `plugin-sdk/secret-input` | Analyse des entrées secrètes | Helpers d'entrée secrète |
  | `plugin-sdk/webhook-ingress` | Helpers de requête webhook | Utilitaires de cible webhook |
  | `plugin-sdk/webhook-request-guards` | Helpers de garde de corps webhook | Helpers de lecture/limitation du corps de requête |
  | `plugin-sdk/reply-runtime` | Runtime partagé de réponse | Envoi entrant, heartbeat, planificateur de réponse, fragmentation |
  | `plugin-sdk/reply-dispatch-runtime` | Helpers étroits d'envoi de réponse | Helpers de finalisation + envoi fournisseur |
  | `plugin-sdk/reply-history` | Helpers d'historique des réponses | `buildHistoryContext`, `buildPendingHistoryContextFromMap`, `recordPendingHistoryEntry`, `clearHistoryEntriesIfEnabled` |
  | `plugin-sdk/reply-reference` | Planification des références de réponse | `createReplyReferencePlanner` |
  | `plugin-sdk/reply-chunking` | Helpers de fragmentation des réponses | Helpers de fragmentation texte/markdown |
  | `plugin-sdk/session-store-runtime` | Helpers de magasin de session | Helpers de chemin de magasin + date de mise à jour |
  | `plugin-sdk/state-paths` | Helpers de chemins d'état | Helpers de répertoire d'état et OAuth |
  | `plugin-sdk/routing` | Helpers de routage/clé de session | `resolveAgentRoute`, `buildAgentSessionKey`, `resolveDefaultAgentBoundAccountId`, helpers de normalisation de clé de session |
  | `plugin-sdk/status-helpers` | Helpers de statut de canal | Builders de résumé de statut de canal/compte, valeurs par défaut de l'état runtime, helpers de métadonnées de problèmes |
  | `plugin-sdk/target-resolver-runtime` | Helpers de résolution de cible | Helpers partagés de résolution de cible |
  | `plugin-sdk/string-normalization-runtime` | Helpers de normalisation de chaînes | Helpers de normalisation de slug/chaînes |
  | `plugin-sdk/request-url` | Helpers d'URL de requête | Extraire des URL chaîne depuis des entrées de type requête |
  | `plugin-sdk/run-command` | Helpers de commande temporisée | Exécuteur de commande temporisé avec stdout/stderr normalisés |
  | `plugin-sdk/param-readers` | Lecteurs de paramètres | Lecteurs communs de paramètres outil/CLI |
  | `plugin-sdk/tool-send` | Extraction des envois d'outil | Extraire les champs canoniques de cible d'envoi à partir des arguments d'outil |
  | `plugin-sdk/temp-path` | Helpers de chemin temporaire | Helpers partagés de chemin de téléchargement temporaire |
  | `plugin-sdk/logging-core` | Helpers de journalisation | Helpers de logger de sous-système et de masquage |
  | `plugin-sdk/markdown-table-runtime` | Helpers de tableau Markdown | Helpers de mode de tableau Markdown |
  | `plugin-sdk/reply-payload` | Types de réponse de message | Types de charge utile de réponse |
  | `plugin-sdk/provider-setup` | Helpers sélectionnés de configuration de fournisseur local/autohébergé | Helpers de découverte/configuration de fournisseur autohébergé |
  | `plugin-sdk/self-hosted-provider-setup` | Helpers ciblés de configuration de fournisseur autohébergé compatible OpenAI | Les mêmes helpers de découverte/configuration de fournisseur autohébergé |
  | `plugin-sdk/provider-auth-runtime` | Helpers d'authentification runtime de fournisseur | Helpers de résolution runtime de clé API |
  | `plugin-sdk/provider-auth-api-key` | Helpers de configuration de clé API fournisseur | Helpers d'onboarding/écriture de profil de clé API |
  | `plugin-sdk/provider-auth-result` | Helpers de résultat d'authentification fournisseur | Builder standard de résultat d'authentification OAuth |
  | `plugin-sdk/provider-auth-login` | Helpers de connexion interactive fournisseur | Helpers partagés de connexion interactive |
  | `plugin-sdk/provider-env-vars` | Helpers de variables d'environnement fournisseur | Helpers de recherche des variables d'environnement d'authentification fournisseur |
  | `plugin-sdk/provider-model-shared` | Helpers partagés de modèle/relecture fournisseur | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, builders partagés de politique de relecture, helpers d'endpoint fournisseur et helpers de normalisation d'identifiant de modèle |
  | `plugin-sdk/provider-catalog-shared` | Helpers partagés de catalogue fournisseur | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
  | `plugin-sdk/provider-onboard` | Patches d'onboarding fournisseur | Helpers de configuration d'onboarding |
  | `plugin-sdk/provider-http` | Helpers HTTP fournisseur | Helpers génériques HTTP/capacité d'endpoint fournisseur |
  | `plugin-sdk/provider-web-fetch` | Helpers de web-fetch fournisseur | Helpers d'enregistrement/cache du fournisseur web-fetch |
  | `plugin-sdk/provider-web-search-contract` | Helpers de contrat de recherche web fournisseur | Helpers étroits de contrat config/identifiants de recherche web comme `enablePluginInConfig`, `resolveProviderWebSearchPluginConfig`, et setters/getters d'identifiants à portée limitée |
  | `plugin-sdk/provider-web-search` | Helpers de recherche web fournisseur | Helpers d'enregistrement/cache/runtime du fournisseur de recherche web |
  | `plugin-sdk/provider-tools` | Helpers de compatibilité outil/schéma fournisseur | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, nettoyage de schéma Gemini + diagnostics, et helpers de compatibilité xAI comme `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
  | `plugin-sdk/provider-usage` | Helpers d'usage fournisseur | `fetchClaudeUsage`, `fetchGeminiUsage`, `fetchGithubCopilotUsage`, et autres helpers d'usage fournisseur |
  | `plugin-sdk/provider-stream` | Helpers de wrapper de flux fournisseur | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, types de wrapper de flux, et helpers partagés de wrappers Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot |
  | `plugin-sdk/keyed-async-queue` | File async ordonnée | `KeyedAsyncQueue` |
  | `plugin-sdk/media-runtime` | Helpers partagés de média | Helpers de récupération/transformation/stockage de média plus builders de charge utile média |
  | `plugin-sdk/media-generation-runtime` | Helpers partagés de génération média | Helpers partagés de bascule, sélection de candidats et messages de modèle manquant pour la génération d'image/vidéo/musique |
  | `plugin-sdk/media-understanding` | Helpers de compréhension média | Types de fournisseur media-understanding plus exports de helpers image/audio côté fournisseur |
  | `plugin-sdk/text-runtime` | Helpers partagés de texte | Suppression du texte visible par l'assistant, helpers de rendu/fragmentation/tableau Markdown, helpers de masquage, helpers de balises de directive, utilitaires de texte sûr, et helpers liés au texte/à la journalisation |
  | `plugin-sdk/text-chunking` | Helpers de fragmentation de texte | Helper de fragmentation de texte sortant |
  | `plugin-sdk/speech` | Helpers de parole | Types de fournisseur de parole plus helpers côté fournisseur pour directives, registre et validation |
  | `plugin-sdk/speech-core` | Cœur partagé de la parole | Types de fournisseur de parole, registre, directives, normalisation |
  | `plugin-sdk/realtime-transcription` | Helpers de transcription en temps réel | Types de fournisseur et helpers de registre |
  | `plugin-sdk/realtime-voice` | Helpers de voix en temps réel | Types de fournisseur et helpers de registre |
  | `plugin-sdk/image-generation-core` | Cœur partagé de génération d'image | Types, bascule, authentification et helpers de registre de génération d'image |
  | `plugin-sdk/music-generation` | Helpers de génération musicale | Types de fournisseur/requête/résultat de génération musicale |
  | `plugin-sdk/music-generation-core` | Cœur partagé de génération musicale | Types de génération musicale, helpers de bascule, recherche de fournisseur et analyse de référence de modèle |
  | `plugin-sdk/video-generation` | Helpers de génération vidéo | Types de fournisseur/requête/résultat de génération vidéo |
  | `plugin-sdk/video-generation-core` | Cœur partagé de génération vidéo | Types de génération vidéo, helpers de bascule, recherche de fournisseur et analyse de référence de modèle |
  | `plugin-sdk/interactive-runtime` | Helpers de réponse interactive | Normalisation/réduction des charges utiles de réponse interactive |
  | `plugin-sdk/channel-config-primitives` | Primitives de configuration de canal | Primitives étroites de schéma de configuration de canal |
  | `plugin-sdk/channel-config-writes` | Helpers d'écriture de configuration de canal | Helpers d'autorisation d'écriture de configuration de canal |
  | `plugin-sdk/channel-plugin-common` | Prélude partagé de canal | Exports de prélude partagés de plugin de canal |
  | `plugin-sdk/channel-status` | Helpers de statut de canal | Helpers partagés de cliché/résumé de statut de canal |
  | `plugin-sdk/allowlist-config-edit` | Helpers de configuration de liste d'autorisation | Helpers de lecture/édition de configuration de liste d'autorisation |
  | `plugin-sdk/group-access` | Helpers d'accès groupe | Helpers partagés de décision d'accès groupe |
  | `plugin-sdk/direct-dm` | Helpers de DM direct | Helpers partagés d'authentification/garde pour DM direct |
  | `plugin-sdk/extension-shared` | Helpers partagés d'extension | Primitives de canal/statut passif et de proxy ambiant |
  | `plugin-sdk/webhook-targets` | Helpers de cible webhook | Helpers de registre de cibles webhook et d'installation de routes |
  | `plugin-sdk/webhook-path` | Helpers de chemin webhook | Helpers de normalisation de chemin webhook |
  | `plugin-sdk/web-media` | Helpers partagés de média web | Helpers de chargement de média distant/local |
  | `plugin-sdk/zod` | Réexport Zod | Réexport de `zod` pour les consommateurs du Plugin SDK |
  | `plugin-sdk/memory-core` | Helpers groupés memory-core | Surface de helpers pour gestionnaire/configuration/fichier/CLI mémoire |
  | `plugin-sdk/memory-core-engine-runtime` | Façade runtime du moteur mémoire | Façade runtime d'indexation/recherche mémoire |
  | `plugin-sdk/memory-core-host-engine-foundation` | Moteur de fondation hôte mémoire | Exports du moteur de fondation hôte mémoire |
  | `plugin-sdk/memory-core-host-engine-embeddings` | Moteur d'embedding hôte mémoire | Exports du moteur d'embedding hôte mémoire |
  | `plugin-sdk/memory-core-host-engine-qmd` | Moteur QMD hôte mémoire | Exports du moteur QMD hôte mémoire |
  | `plugin-sdk/memory-core-host-engine-storage` | Moteur de stockage hôte mémoire | Exports du moteur de stockage hôte mémoire |
  | `plugin-sdk/memory-core-host-multimodal` | Helpers multimodaux hôte mémoire | Helpers multimodaux hôte mémoire |
  | `plugin-sdk/memory-core-host-query` | Helpers de requête hôte mémoire | Helpers de requête hôte mémoire |
  | `plugin-sdk/memory-core-host-secret` | Helpers de secret hôte mémoire | Helpers de secret hôte mémoire |
  | `plugin-sdk/memory-core-host-events` | Helpers de journal d'événements hôte mémoire | Helpers de journal d'événements hôte mémoire |
  | `plugin-sdk/memory-core-host-status` | Helpers de statut hôte mémoire | Helpers de statut hôte mémoire |
  | `plugin-sdk/memory-core-host-runtime-cli` | Runtime CLI hôte mémoire | Helpers runtime CLI hôte mémoire |
  | `plugin-sdk/memory-core-host-runtime-core` | Runtime cœur hôte mémoire | Helpers runtime cœur hôte mémoire |
  | `plugin-sdk/memory-core-host-runtime-files` | Helpers de fichiers/runtime hôte mémoire | Helpers de fichiers/runtime hôte mémoire |
  | `plugin-sdk/memory-host-core` | Alias runtime cœur hôte mémoire | Alias neutre vis-à-vis du fournisseur pour les helpers runtime cœur hôte mémoire |
  | `plugin-sdk/memory-host-events` | Alias journal d'événements hôte mémoire | Alias neutre vis-à-vis du fournisseur pour les helpers de journal d'événements hôte mémoire |
  | `plugin-sdk/memory-host-files` | Alias fichier/runtime hôte mémoire | Alias neutre vis-à-vis du fournisseur pour les helpers de fichier/runtime hôte mémoire |
  | `plugin-sdk/memory-host-markdown` | Helpers Markdown gérés | Helpers partagés de Markdown géré pour les plugins adjacents à la mémoire |
  | `plugin-sdk/memory-host-search` | Façade de recherche mémoire active | Façade runtime paresseuse du gestionnaire de recherche de mémoire active |
  | `plugin-sdk/memory-host-status` | Alias statut hôte mémoire | Alias neutre vis-à-vis du fournisseur pour les helpers de statut hôte mémoire |
  | `plugin-sdk/memory-lancedb` | Helpers groupés memory-lancedb | Surface de helpers memory-lancedb |
  | `plugin-sdk/testing` | Utilitaires de test | Helpers et mocks de test |
</Accordion>

Ce tableau couvre intentionnellement le sous-ensemble commun de migration, et non la surface
complète du SDK. La liste complète des plus de 200 points d'entrée se trouve dans
`scripts/lib/plugin-sdk-entrypoints.json`.

Cette liste inclut encore certains points d'extension de helpers de plugins groupés comme
`plugin-sdk/feishu`, `plugin-sdk/feishu-setup`, `plugin-sdk/zalo`,
`plugin-sdk/zalo-setup`, et `plugin-sdk/matrix*`. Ils restent exportés pour
la maintenance et la compatibilité des plugins groupés, mais sont volontairement
omis du tableau commun de migration et ne constituent pas la cible recommandée pour le
nouveau code de plugin.

La même règle s'applique à d'autres familles de helpers groupés telles que :

- helpers de prise en charge du navigateur : `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support`
- Matrix : `plugin-sdk/matrix*`
- LINE : `plugin-sdk/line*`
- IRC : `plugin-sdk/irc*`
- surfaces de helper/plugin groupées comme `plugin-sdk/googlechat`,
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

Utilisez l'import le plus étroit qui correspond à la tâche. Si vous ne trouvez pas un export,
vérifiez la source dans `src/plugin-sdk/` ou demandez sur Discord.

## Calendrier de suppression

| Quand | Ce qui se passe |
| ---------------------- | ----------------------------------------------------------------------- |
| **Maintenant** | Les surfaces obsolètes émettent des avertissements au runtime |
| **Prochaine version majeure** | Les surfaces obsolètes seront supprimées ; les plugins qui les utilisent encore échoueront |

Tous les plugins du cœur ont déjà été migrés. Les plugins externes doivent migrer
avant la prochaine version majeure.

## Supprimer temporairement les avertissements

Définissez ces variables d'environnement pendant que vous travaillez à la migration :

```bash
OPENCLAW_SUPPRESS_PLUGIN_SDK_COMPAT_WARNING=1 openclaw gateway run
OPENCLAW_SUPPRESS_EXTENSION_API_WARNING=1 openclaw gateway run
```

Il s'agit d'une échappatoire temporaire, pas d'une solution permanente.

## Lié

- [Getting Started](/fr/plugins/building-plugins) — créez votre premier plugin
- [SDK Overview](/fr/plugins/sdk-overview) — référence complète des imports par sous-chemin
- [Channel Plugins](/fr/plugins/sdk-channel-plugins) — créer des plugins de canal
- [Provider Plugins](/fr/plugins/sdk-provider-plugins) — créer des plugins de fournisseur
- [Plugin Internals](/fr/plugins/architecture) — analyse approfondie de l'architecture
- [Plugin Manifest](/fr/plugins/manifest) — référence du schéma de manifeste
