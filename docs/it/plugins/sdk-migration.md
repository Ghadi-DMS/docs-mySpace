---
read_when:
    - Visualizzi l'avviso OPENCLAW_PLUGIN_SDK_COMPAT_DEPRECATED
    - Visualizzi l'avviso OPENCLAW_EXTENSION_API_DEPRECATED
    - Stai aggiornando un plugin alla moderna architettura dei plugin
    - Mantieni un plugin OpenClaw esterno
sidebarTitle: Migrate to SDK
summary: Migra dal livello legacy di retrocompatibilità al moderno plugin SDK
title: Migrazione del Plugin SDK
x-i18n:
    generated_at: "2026-04-08T08:13:26Z"
    model: gpt-5.4
    provider: openai
    source_hash: d9a2ce7f5553563516a549ca87e776a6a71e8dd8533a773c5ddbecfae43e7b77
    source_path: plugins/sdk-migration.md
    workflow: 15
---

# Migrazione del Plugin SDK

OpenClaw è passato da un ampio livello di retrocompatibilità a una moderna
architettura dei plugin con import mirati e documentati. Se il tuo plugin è
stato creato prima della nuova architettura, questa guida ti aiuta a migrare.

## Cosa sta cambiando

Il vecchio sistema di plugin forniva due superfici molto ampie che permettevano
ai plugin di importare qualsiasi cosa servisse da un unico punto di ingresso:

- **`openclaw/plugin-sdk/compat`** — un singolo import che riesportava decine di
  helper. È stato introdotto per mantenere funzionanti i plugin più vecchi
  basati su hook mentre veniva sviluppata la nuova architettura dei plugin.
- **`openclaw/extension-api`** — un bridge che forniva ai plugin accesso diretto
  agli helper lato host, come l'embedded agent runner.

Entrambe le superfici sono ora **deprecate**. Continuano a funzionare a runtime,
ma i nuovi plugin non devono usarle e i plugin esistenti dovrebbero migrare
prima che la prossima major release le rimuova.

<Warning>
  Il livello di retrocompatibilità verrà rimosso in una futura major release.
  I plugin che importano ancora da queste superfici smetteranno di funzionare quando ciò accadrà.
</Warning>

## Perché questo è cambiato

Il vecchio approccio causava problemi:

- **Avvio lento** — importare un helper caricava decine di moduli non correlati
- **Dipendenze circolari** — riesportazioni ampie rendevano facile creare cicli di import
- **Superficie API poco chiara** — non c'era modo di capire quali export fossero stabili e quali interni

Il moderno plugin SDK risolve questo problema: ogni percorso di import (`openclaw/plugin-sdk/\<subpath\>`)
è un modulo piccolo e autonomo con uno scopo chiaro e un contratto documentato.

Sono state rimosse anche le comode seam legacy dei provider per i canali bundle.
Import come `openclaw/plugin-sdk/slack`, `openclaw/plugin-sdk/discord`,
`openclaw/plugin-sdk/signal`, `openclaw/plugin-sdk/whatsapp`,
le seam helper con brand del canale e
`openclaw/plugin-sdk/telegram-core` erano scorciatoie private del mono-repo, non
contratti plugin stabili. Usa invece subpath SDK generici e mirati. All'interno
del workspace dei plugin bundle, mantieni gli helper di proprietà del provider
nel `api.ts` o `runtime-api.ts` del plugin stesso.

Esempi attuali di provider bundle:

- Anthropic mantiene gli helper di stream specifici per Claude nel proprio `api.ts` /
  seam `contract-api.ts`
- OpenAI mantiene i builder dei provider, gli helper per i modelli predefiniti e i builder dei provider realtime
  nel proprio `api.ts`
- OpenRouter mantiene nel proprio `api.ts` il builder del provider e gli helper di onboarding/config

## Come migrare

<Steps>
  <Step title="Migra gli handler approval-native ai capability facts">
    I plugin di canale con capacità di approvazione ora espongono il comportamento di approvazione nativo tramite
    `approvalCapability.nativeRuntime` più il registro condiviso del contesto runtime.

    Modifiche principali:

    - Sostituisci `approvalCapability.handler.loadRuntime(...)` con
      `approvalCapability.nativeRuntime`
    - Sposta auth/delivery specifici dell'approvazione dal wiring legacy `plugin.auth` /
      `plugin.approvals` a `approvalCapability`
    - `ChannelPlugin.approvals` è stato rimosso dal contratto pubblico dei
      plugin di canale; sposta i campi delivery/native/render in `approvalCapability`
    - `plugin.auth` rimane solo per i flussi di login/logout del canale; gli hook
      auth di approvazione lì presenti non vengono più letti dal core
    - Registra oggetti runtime di proprietà del canale, come client, token o app
      Bolt, tramite `openclaw/plugin-sdk/channel-runtime-context`
    - Non inviare avvisi di reroute di proprietà del plugin dagli handler di approvazione nativi;
      il core ora gestisce gli avvisi di instradamento altrove a partire dai risultati di delivery reali
    - Quando passi `channelRuntime` a `createChannelManager(...)`, fornisci una
      vera superficie `createPluginRuntime().channel`. Gli stub parziali vengono rifiutati.

    Vedi `/plugins/sdk-channel-plugins` per l'attuale layout della capability di approvazione.

  </Step>

  <Step title="Controlla il comportamento di fallback del wrapper Windows">
    Se il tuo plugin usa `openclaw/plugin-sdk/windows-spawn`, i wrapper Windows
    `.cmd`/`.bat` non risolti ora falliscono in modalità chiusa a meno che tu non passi esplicitamente
    `allowShellFallback: true`.

    ```typescript
    // Prima
    const program = applyWindowsSpawnProgramPolicy({ candidate });

    // Dopo
    const program = applyWindowsSpawnProgramPolicy({
      candidate,
      // Imposta questo solo per chiamanti di compatibilità fidati che
      // accettano intenzionalmente il fallback mediato dalla shell.
      allowShellFallback: true,
    });
    ```

    Se il chiamante non dipende intenzionalmente dal fallback della shell, non impostare
    `allowShellFallback` e gestisci invece l'errore generato.

  </Step>

  <Step title="Trova gli import deprecati">
    Cerca nel tuo plugin gli import provenienti da una delle due superfici deprecate:

    ```bash
    grep -r "plugin-sdk/compat" my-plugin/
    grep -r "openclaw/extension-api" my-plugin/
    ```

  </Step>

  <Step title="Sostituisci con import mirati">
    Ogni export della vecchia superficie corrisponde a uno specifico percorso di import moderno:

    ```typescript
    // Prima (livello di retrocompatibilità deprecato)
    import {
      createChannelReplyPipeline,
      createPluginRuntimeStore,
      resolveControlCommandGate,
    } from "openclaw/plugin-sdk/compat";

    // Dopo (import moderni e mirati)
    import { createChannelReplyPipeline } from "openclaw/plugin-sdk/channel-reply-pipeline";
    import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
    import { resolveControlCommandGate } from "openclaw/plugin-sdk/command-auth";
    ```

    Per gli helper lato host, usa il runtime del plugin iniettato invece di importare
    direttamente:

    ```typescript
    // Prima (bridge extension-api deprecato)
    import { runEmbeddedPiAgent } from "openclaw/extension-api";
    const result = await runEmbeddedPiAgent({ sessionId, prompt });

    // Dopo (runtime iniettato)
    const result = await api.runtime.agent.runEmbeddedPiAgent({ sessionId, prompt });
    ```

    Lo stesso schema si applica ad altri helper legacy del bridge:

    | Vecchio import | Equivalente moderno |
    | --- | --- |
    | `resolveAgentDir` | `api.runtime.agent.resolveAgentDir` |
    | `resolveAgentWorkspaceDir` | `api.runtime.agent.resolveAgentWorkspaceDir` |
    | `resolveAgentIdentity` | `api.runtime.agent.resolveAgentIdentity` |
    | `resolveThinkingDefault` | `api.runtime.agent.resolveThinkingDefault` |
    | `resolveAgentTimeoutMs` | `api.runtime.agent.resolveAgentTimeoutMs` |
    | `ensureAgentWorkspace` | `api.runtime.agent.ensureAgentWorkspace` |
    | helper del session store | `api.runtime.agent.session.*` |

  </Step>

  <Step title="Build e test">
    ```bash
    pnpm build
    pnpm test -- my-plugin/
    ```
  </Step>
</Steps>

## Riferimento ai percorsi di import

<Accordion title="Tabella comune dei percorsi di import">
  | Percorso di import | Scopo | Export principali |
  | --- | --- | --- |
  | `plugin-sdk/plugin-entry` | Helper canonico per il punto di ingresso del plugin | `definePluginEntry` |
  | `plugin-sdk/core` | Riesportazione legacy ombrello per definizioni/builder del punto di ingresso del canale | `defineChannelPluginEntry`, `createChatChannelPlugin` |
  | `plugin-sdk/config-schema` | Export dello schema di configurazione radice | `OpenClawSchema` |
  | `plugin-sdk/provider-entry` | Helper per il punto di ingresso a provider singolo | `defineSingleProviderPluginEntry` |
  | `plugin-sdk/channel-core` | Definizioni e builder mirati per il punto di ingresso del canale | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
  | `plugin-sdk/setup` | Helper condivisi per il wizard di configurazione | Prompt allowlist, builder dello stato di configurazione |
  | `plugin-sdk/setup-runtime` | Helper runtime per la configurazione | Adapter di patch setup import-safe, helper per note di lookup, `promptResolvedAllowFrom`, `splitSetupEntries`, proxy di setup delegati |
  | `plugin-sdk/setup-adapter-runtime` | Helper per adapter di setup | `createEnvPatchedAccountSetupAdapter` |
  | `plugin-sdk/setup-tools` | Helper per gli strumenti di setup | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
  | `plugin-sdk/account-core` | Helper multi-account | Helper per elenco account/config/action-gate |
  | `plugin-sdk/account-id` | Helper per account-id | `DEFAULT_ACCOUNT_ID`, normalizzazione di account-id |
  | `plugin-sdk/account-resolution` | Helper per lookup account | Helper per lookup account + fallback predefinito |
  | `plugin-sdk/account-helpers` | Helper account mirati | Helper per elenco account/azioni account |
  | `plugin-sdk/channel-setup` | Adapter del wizard di configurazione | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, più `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
  | `plugin-sdk/channel-pairing` | Primitive di pairing DM | `createChannelPairingController` |
  | `plugin-sdk/channel-reply-pipeline` | Wiring di prefisso risposta + typing | `createChannelReplyPipeline` |
  | `plugin-sdk/channel-config-helpers` | Factory per adapter di configurazione | `createHybridChannelConfigAdapter` |
  | `plugin-sdk/channel-config-schema` | Builder dello schema di configurazione | Tipi di schema per la configurazione del canale |
  | `plugin-sdk/telegram-command-config` | Helper per la configurazione dei comandi Telegram | Normalizzazione del nome comando, trimming della descrizione, validazione di duplicati/conflitti |
  | `plugin-sdk/channel-policy` | Risoluzione delle policy gruppo/DM | `resolveChannelGroupRequireMention` |
  | `plugin-sdk/channel-lifecycle` | Tracciamento dello stato account | `createAccountStatusSink` |
  | `plugin-sdk/inbound-envelope` | Helper per inbound envelope | Helper condivisi per route + builder envelope |
  | `plugin-sdk/inbound-reply-dispatch` | Helper per risposte in ingresso | Helper condivisi per record-and-dispatch |
  | `plugin-sdk/messaging-targets` | Parsing dei target di messaggistica | Helper per parsing/matching dei target |
  | `plugin-sdk/outbound-media` | Helper per media in uscita | Caricamento condiviso dei media in uscita |
  | `plugin-sdk/outbound-runtime` | Helper runtime per l'uscita | Helper per identità/delega di invio in uscita |
  | `plugin-sdk/thread-bindings-runtime` | Helper per thread-binding | Helper per lifecycle e adapter dei thread-binding |
  | `plugin-sdk/agent-media-payload` | Helper legacy per media payload | Builder di media payload dell'agente per layout di campi legacy |
  | `plugin-sdk/channel-runtime` | Shim di compatibilità deprecato | Solo utility legacy per il runtime del canale |
  | `plugin-sdk/channel-send-result` | Tipi di risultato di invio | Tipi di risultato della risposta |
  | `plugin-sdk/runtime-store` | Storage persistente del plugin | `createPluginRuntimeStore` |
  | `plugin-sdk/runtime` | Helper runtime estesi | Helper per runtime/logging/backup/installazione plugin |
  | `plugin-sdk/runtime-env` | Helper runtime-env mirati | Helper per logger/runtime env, timeout, retry e backoff |
  | `plugin-sdk/plugin-runtime` | Helper runtime condivisi del plugin | Helper per comandi/hook/http/interattività del plugin |
  | `plugin-sdk/hook-runtime` | Helper per la pipeline degli hook | Helper condivisi per pipeline di webhook/internal hook |
  | `plugin-sdk/lazy-runtime` | Helper runtime lazy | `createLazyRuntimeModule`, `createLazyRuntimeMethod`, `createLazyRuntimeMethodBinder`, `createLazyRuntimeNamedExport`, `createLazyRuntimeSurface` |
  | `plugin-sdk/process-runtime` | Helper di processo | Helper condivisi per exec |
  | `plugin-sdk/cli-runtime` | Helper runtime della CLI | Formattazione comandi, attese, helper di versione |
  | `plugin-sdk/gateway-runtime` | Helper del gateway | Client gateway e helper di patch dello stato del canale |
  | `plugin-sdk/config-runtime` | Helper di configurazione | Helper per caricamento/scrittura della configurazione |
  | `plugin-sdk/telegram-command-config` | Helper per i comandi Telegram | Helper di validazione dei comandi Telegram con fallback stabile quando la superficie di contratto del Telegram bundle non è disponibile |
  | `plugin-sdk/approval-runtime` | Helper per i prompt di approvazione | Payload di approvazione exec/plugin, helper per capability/profile di approvazione, helper per instradamento/runtime di approvazione nativa |
  | `plugin-sdk/approval-auth-runtime` | Helper auth per approvazioni | Risoluzione dell'approvatore, auth per azioni nella stessa chat |
  | `plugin-sdk/approval-client-runtime` | Helper client per approvazioni | Helper per profilo/filtro di approvazione exec nativa |
  | `plugin-sdk/approval-delivery-runtime` | Helper delivery per approvazioni | Adapter di capability/delivery per approvazioni native |
  | `plugin-sdk/approval-gateway-runtime` | Helper gateway per approvazioni | Helper condiviso per la risoluzione del gateway di approvazione |
  | `plugin-sdk/approval-handler-adapter-runtime` | Helper adapter per approvazioni | Helper leggeri per il caricamento di adapter di approvazione nativa per entrypoint di canale hot |
  | `plugin-sdk/approval-handler-runtime` | Helper handler per approvazioni | Helper runtime più estesi per handler di approvazione; preferisci seam adapter/gateway più mirate quando sono sufficienti |
  | `plugin-sdk/approval-native-runtime` | Helper per target di approvazione | Helper nativi per target/account binding di approvazione |
  | `plugin-sdk/approval-reply-runtime` | Helper per risposte di approvazione | Helper per payload di risposta di approvazione exec/plugin |
  | `plugin-sdk/channel-runtime-context` | Helper per il contesto runtime del canale | Helper generici register/get/watch per il contesto runtime del canale |
  | `plugin-sdk/security-runtime` | Helper di sicurezza | Helper condivisi per trust, gating DM, contenuti esterni e raccolta dei secret |
  | `plugin-sdk/ssrf-policy` | Helper per policy SSRF | Helper per allowlist host e policy di rete privata |
  | `plugin-sdk/ssrf-runtime` | Helper runtime SSRF | Dispatcher fissato, fetch protetto, helper per policy SSRF |
  | `plugin-sdk/collection-runtime` | Helper per cache limitata | `pruneMapToMaxSize` |
  | `plugin-sdk/diagnostic-runtime` | Helper per gating diagnostico | `isDiagnosticFlagEnabled`, `isDiagnosticsEnabled` |
  | `plugin-sdk/error-runtime` | Helper per la formattazione degli errori | `formatUncaughtError`, `isApprovalNotFoundError`, helper per grafi di errore |
  | `plugin-sdk/fetch-runtime` | Helper per fetch/proxy wrapped | `resolveFetch`, helper per proxy |
  | `plugin-sdk/host-runtime` | Helper per la normalizzazione dell'host | `normalizeHostname`, `normalizeScpRemoteHost` |
  | `plugin-sdk/retry-runtime` | Helper di retry | `RetryConfig`, `retryAsync`, runner di policy |
  | `plugin-sdk/allow-from` | Formattazione della allowlist | `formatAllowFromLowercase` |
  | `plugin-sdk/allowlist-resolution` | Mappatura degli input della allowlist | `mapAllowlistResolutionInputs` |
  | `plugin-sdk/command-auth` | Gating dei comandi e helper per la superficie dei comandi | `resolveControlCommandGate`, helper di autorizzazione del mittente, helper per registro dei comandi |
  | `plugin-sdk/secret-input` | Parsing dell'input secret | Helper per input secret |
  | `plugin-sdk/webhook-ingress` | Helper per richieste webhook | Utility per target webhook |
  | `plugin-sdk/webhook-request-guards` | Helper guard per body webhook | Helper per lettura/limite del body della richiesta |
  | `plugin-sdk/reply-runtime` | Runtime condiviso delle risposte | Dispatch in ingresso, heartbeat, pianificatore delle risposte, chunking |
  | `plugin-sdk/reply-dispatch-runtime` | Helper mirati per il dispatch delle risposte | Helper per finalize + provider dispatch |
  | `plugin-sdk/reply-history` | Helper per la cronologia delle risposte | `buildHistoryContext`, `buildPendingHistoryContextFromMap`, `recordPendingHistoryEntry`, `clearHistoryEntriesIfEnabled` |
  | `plugin-sdk/reply-reference` | Pianificazione dei riferimenti di risposta | `createReplyReferencePlanner` |
  | `plugin-sdk/reply-chunking` | Helper per chunk delle risposte | Helper per chunking di testo/markdown |
  | `plugin-sdk/session-store-runtime` | Helper per il session store | Helper per percorso dello store + updated-at |
  | `plugin-sdk/state-paths` | Helper per i percorsi di stato | Helper per directory di stato e OAuth |
  | `plugin-sdk/routing` | Helper per routing/session-key | `resolveAgentRoute`, `buildAgentSessionKey`, `resolveDefaultAgentBoundAccountId`, helper di normalizzazione della session-key |
  | `plugin-sdk/status-helpers` | Helper per lo stato del canale | Builder di riepilogo dello stato canale/account, valori predefiniti dello stato runtime, helper per metadata dei problemi |
  | `plugin-sdk/target-resolver-runtime` | Helper per il resolver dei target | Helper condivisi per il resolver dei target |
  | `plugin-sdk/string-normalization-runtime` | Helper per la normalizzazione delle stringhe | Helper per la normalizzazione di slug/stringhe |
  | `plugin-sdk/request-url` | Helper per URL di richiesta | Estrae URL stringa da input simili a richieste |
  | `plugin-sdk/run-command` | Helper per comandi temporizzati | Runner di comandi temporizzati con stdout/stderr normalizzati |
  | `plugin-sdk/param-readers` | Lettori di parametri | Lettori comuni di parametri per tool/CLI |
  | `plugin-sdk/tool-payload` | Estrazione del payload dei tool | Estrae payload normalizzati dagli oggetti risultato dei tool |
  | `plugin-sdk/tool-send` | Estrazione dell'invio dei tool | Estrae campi target di invio canonici dagli argomenti del tool |
  | `plugin-sdk/temp-path` | Helper per percorsi temporanei | Helper condivisi per percorsi temporanei di download |
  | `plugin-sdk/logging-core` | Helper di logging | Logger di sottosistema e helper di redazione |
  | `plugin-sdk/markdown-table-runtime` | Helper per tabelle Markdown | Helper per modalità tabella Markdown |
  | `plugin-sdk/reply-payload` | Tipi di risposta del messaggio | Tipi di payload della risposta |
  | `plugin-sdk/provider-setup` | Helper curati per la configurazione di provider locali/self-hosted | Helper per discovery/config di provider self-hosted |
  | `plugin-sdk/self-hosted-provider-setup` | Helper mirati per la configurazione di provider self-hosted compatibili con OpenAI | Gli stessi helper per discovery/config di provider self-hosted |
  | `plugin-sdk/provider-auth-runtime` | Helper auth runtime del provider | Helper runtime per la risoluzione delle API key |
  | `plugin-sdk/provider-auth-api-key` | Helper di configurazione API key del provider | Helper per onboarding/scrittura profilo con API key |
  | `plugin-sdk/provider-auth-result` | Helper per auth-result del provider | Builder standard del risultato auth OAuth |
  | `plugin-sdk/provider-auth-login` | Helper di login interattivo del provider | Helper condivisi per login interattivo |
  | `plugin-sdk/provider-env-vars` | Helper per env var del provider | Helper per il lookup delle env var auth del provider |
  | `plugin-sdk/provider-model-shared` | Helper condivisi per modelli/replay del provider | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, builder condivisi di replay-policy, helper per endpoint del provider e helper di normalizzazione del model-id |
  | `plugin-sdk/provider-catalog-shared` | Helper condivisi per catalogo del provider | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
  | `plugin-sdk/provider-onboard` | Patch di onboarding del provider | Helper di configurazione per onboarding |
  | `plugin-sdk/provider-http` | Helper HTTP del provider | Helper generici per HTTP/capability endpoint del provider |
  | `plugin-sdk/provider-web-fetch` | Helper web-fetch del provider | Helper per registrazione/cache del provider web-fetch |
  | `plugin-sdk/provider-web-search-config-contract` | Helper di configurazione per web-search del provider | Helper mirati di configurazione/credenziali web-search per provider che non richiedono wiring di abilitazione plugin |
  | `plugin-sdk/provider-web-search-contract` | Helper di contratto per web-search del provider | Helper mirati di contratto per configurazione/credenziali web-search come `createWebSearchProviderContractFields`, `enablePluginInConfig`, `resolveProviderWebSearchPluginConfig` e setter/getter di credenziali con scope |
  | `plugin-sdk/provider-web-search` | Helper di web-search del provider | Helper per registrazione/cache/runtime del provider web-search |
  | `plugin-sdk/provider-tools` | Helper di compatibilità provider tool/schema | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, pulizia schema Gemini + diagnostica e helper compat per xAI come `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
  | `plugin-sdk/provider-usage` | Helper di utilizzo del provider | `fetchClaudeUsage`, `fetchGeminiUsage`, `fetchGithubCopilotUsage` e altri helper di utilizzo del provider |
  | `plugin-sdk/provider-stream` | Helper wrapper di stream del provider | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, tipi di stream wrapper e helper wrapper condivisi per Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot |
  | `plugin-sdk/keyed-async-queue` | Coda asincrona ordinata | `KeyedAsyncQueue` |
  | `plugin-sdk/media-runtime` | Helper media condivisi | Helper per fetch/transform/store dei media più builder di media payload |
  | `plugin-sdk/media-generation-runtime` | Helper condivisi per media-generation | Helper condivisi per failover, selezione dei candidati e messaggi di modello mancante per generazione di immagini/video/musica |
  | `plugin-sdk/media-understanding` | Helper per media-understanding | Tipi di provider di media understanding più export di helper per immagini/audio rivolti ai provider |
  | `plugin-sdk/text-runtime` | Helper di testo condivisi | Rimozione di testo visibile all'assistente, helper per render/chunking/tabella markdown, helper di redazione, helper per directive-tag, utility per testo sicuro e relativi helper di testo/logging |
  | `plugin-sdk/text-chunking` | Helper per chunking del testo | Helper per chunking del testo in uscita |
  | `plugin-sdk/speech` | Helper speech | Tipi di provider speech più export di helper per direttive, registro e validazione rivolti ai provider |
  | `plugin-sdk/speech-core` | Core speech condiviso | Tipi di provider speech, registro, direttive, normalizzazione |
  | `plugin-sdk/realtime-transcription` | Helper per trascrizione realtime | Tipi di provider e helper di registro |
  | `plugin-sdk/realtime-voice` | Helper per voce realtime | Tipi di provider e helper di registro |
  | `plugin-sdk/image-generation-core` | Core condiviso per image-generation | Tipi, failover, auth e helper di registro per image-generation |
  | `plugin-sdk/music-generation` | Helper per music-generation | Tipi di provider/richiesta/risultato per music-generation |
  | `plugin-sdk/music-generation-core` | Core condiviso per music-generation | Tipi per music-generation, helper di failover, lookup del provider e parsing del model-ref |
  | `plugin-sdk/video-generation` | Helper per video-generation | Tipi di provider/richiesta/risultato per video-generation |
  | `plugin-sdk/video-generation-core` | Core condiviso per video-generation | Tipi per video-generation, helper di failover, lookup del provider e parsing del model-ref |
  | `plugin-sdk/interactive-runtime` | Helper per risposte interattive | Normalizzazione/riduzione del payload delle risposte interattive |
  | `plugin-sdk/channel-config-primitives` | Primitive per la configurazione del canale | Primitive mirate di channel config-schema |
  | `plugin-sdk/channel-config-writes` | Helper per scrittura della configurazione del canale | Helper di autorizzazione per la scrittura della configurazione del canale |
  | `plugin-sdk/channel-plugin-common` | Preludio condiviso del canale | Export del preludio condiviso del plugin di canale |
  | `plugin-sdk/channel-status` | Helper per lo stato del canale | Helper condivisi per snapshot/riepilogo dello stato del canale |
  | `plugin-sdk/allowlist-config-edit` | Helper di configurazione della allowlist | Helper per modifica/lettura della configurazione della allowlist |
  | `plugin-sdk/group-access` | Helper per accesso ai gruppi | Helper condivisi per decisioni di accesso ai gruppi |
  | `plugin-sdk/direct-dm` | Helper per DM diretti | Helper condivisi per auth/guard dei DM diretti |
  | `plugin-sdk/extension-shared` | Helper condivisi per estensioni | Primitive helper per canale/stato passivi e proxy ambient |
  | `plugin-sdk/webhook-targets` | Helper per target webhook | Registro dei target webhook e helper per installazione delle route |
  | `plugin-sdk/webhook-path` | Helper per percorsi webhook | Helper per normalizzazione dei percorsi webhook |
  | `plugin-sdk/web-media` | Helper web media condivisi | Helper per caricamento di media remoti/locali |
  | `plugin-sdk/zod` | Riesportazione Zod | `zod` riesportato per i consumer del plugin SDK |
  | `plugin-sdk/memory-core` | Helper bundle di memory-core | Superficie helper per gestore/config/file/CLI della memoria |
  | `plugin-sdk/memory-core-engine-runtime` | Facade runtime del motore di memoria | Facade runtime per indice/ricerca della memoria |
  | `plugin-sdk/memory-core-host-engine-foundation` | Motore foundation host della memoria | Export del motore foundation host della memoria |
  | `plugin-sdk/memory-core-host-engine-embeddings` | Motore embedding host della memoria | Export del motore embedding host della memoria |
  | `plugin-sdk/memory-core-host-engine-qmd` | Motore QMD host della memoria | Export del motore QMD host della memoria |
  | `plugin-sdk/memory-core-host-engine-storage` | Motore storage host della memoria | Export del motore storage host della memoria |
  | `plugin-sdk/memory-core-host-multimodal` | Helper multimodali host della memoria | Helper multimodali host della memoria |
  | `plugin-sdk/memory-core-host-query` | Helper query host della memoria | Helper query host della memoria |
  | `plugin-sdk/memory-core-host-secret` | Helper secret host della memoria | Helper secret host della memoria |
  | `plugin-sdk/memory-core-host-events` | Helper journal eventi host della memoria | Helper journal eventi host della memoria |
  | `plugin-sdk/memory-core-host-status` | Helper stato host della memoria | Helper stato host della memoria |
  | `plugin-sdk/memory-core-host-runtime-cli` | Runtime CLI host della memoria | Helper runtime CLI host della memoria |
  | `plugin-sdk/memory-core-host-runtime-core` | Runtime core host della memoria | Helper runtime core host della memoria |
  | `plugin-sdk/memory-core-host-runtime-files` | Helper file/runtime host della memoria | Helper file/runtime host della memoria |
  | `plugin-sdk/memory-host-core` | Alias runtime core host della memoria | Alias vendor-neutral per helper runtime core host della memoria |
  | `plugin-sdk/memory-host-events` | Alias journal eventi host della memoria | Alias vendor-neutral per helper journal eventi host della memoria |
  | `plugin-sdk/memory-host-files` | Alias file/runtime host della memoria | Alias vendor-neutral per helper file/runtime host della memoria |
  | `plugin-sdk/memory-host-markdown` | Helper markdown gestito | Helper condivisi di managed-markdown per plugin adiacenti alla memoria |
  | `plugin-sdk/memory-host-search` | Facade di ricerca della memoria attiva | Facade runtime lazy del gestore di ricerca della memoria attiva |
  | `plugin-sdk/memory-host-status` | Alias stato host della memoria | Alias vendor-neutral per helper stato host della memoria |
  | `plugin-sdk/memory-lancedb` | Helper bundle di memory-lancedb | Superficie helper di memory-lancedb |
  | `plugin-sdk/testing` | Utility di test | Helper e mock di test |
</Accordion>

Questa tabella è intenzionalmente il sottoinsieme comune per la migrazione, non l'intera
superficie dell'SDK. L'elenco completo di oltre 200 entrypoint si trova in
`scripts/lib/plugin-sdk-entrypoints.json`.

Questo elenco include ancora alcune seam helper di plugin bundle come
`plugin-sdk/feishu`, `plugin-sdk/feishu-setup`, `plugin-sdk/zalo`,
`plugin-sdk/zalo-setup` e `plugin-sdk/matrix*`. Restano esportate per la
manutenzione e la compatibilità dei plugin bundle, ma sono intenzionalmente
omesse dalla tabella comune di migrazione e non sono il target consigliato per
il nuovo codice plugin.

La stessa regola si applica ad altre famiglie di helper bundle, come:

- helper di supporto browser: `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support`
- Matrix: `plugin-sdk/matrix*`
- LINE: `plugin-sdk/line*`
- IRC: `plugin-sdk/irc*`
- superfici di helper/plugin bundle come `plugin-sdk/googlechat`,
  `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles*`,
  `plugin-sdk/mattermost*`, `plugin-sdk/msteams`,
  `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`,
  `plugin-sdk/twitch`,
  `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`,
  `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`,
  `plugin-sdk/thread-ownership` e `plugin-sdk/voice-call`

`plugin-sdk/github-copilot-token` espone attualmente la superficie ristretta di helper per token
`DEFAULT_COPILOT_API_BASE_URL`,
`deriveCopilotApiBaseUrlFromToken` e `resolveCopilotApiToken`.

Usa l'import più ristretto che corrisponde al lavoro da svolgere. Se non trovi un export,
controlla il sorgente in `src/plugin-sdk/` oppure chiedi in Discord.

## Timeline di rimozione

| Quando | Cosa accade |
| ---------------------- | ----------------------------------------------------------------------- |
| **Ora** | Le superfici deprecate emettono avvisi a runtime |
| **Prossima major release** | Le superfici deprecate verranno rimosse; i plugin che le usano ancora falliranno |

Tutti i plugin core sono già stati migrati. I plugin esterni dovrebbero migrare
prima della prossima major release.

## Sopprimere temporaneamente gli avvisi

Imposta queste variabili d'ambiente mentre lavori alla migrazione:

```bash
OPENCLAW_SUPPRESS_PLUGIN_SDK_COMPAT_WARNING=1 openclaw gateway run
OPENCLAW_SUPPRESS_EXTENSION_API_WARNING=1 openclaw gateway run
```

Questa è una via di fuga temporanea, non una soluzione permanente.

## Correlati

- [Per iniziare](/it/plugins/building-plugins) — crea il tuo primo plugin
- [Panoramica dell'SDK](/it/plugins/sdk-overview) — riferimento completo agli import per subpath
- [Plugin di canale](/it/plugins/sdk-channel-plugins) — creazione di plugin di canale
- [Plugin provider](/it/plugins/sdk-provider-plugins) — creazione di plugin provider
- [Componenti interni dei plugin](/it/plugins/architecture) — approfondimento sull'architettura
- [Manifest del plugin](/it/plugins/manifest) — riferimento dello schema del manifest
