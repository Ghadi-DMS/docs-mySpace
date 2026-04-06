---
read_when:
    - Vedi l'avviso OPENCLAW_PLUGIN_SDK_COMPAT_DEPRECATED
    - Vedi l'avviso OPENCLAW_EXTENSION_API_DEPRECATED
    - Stai aggiornando un plugin alla moderna architettura dei plugin
    - Mantieni un plugin OpenClaw esterno
sidebarTitle: Migrate to SDK
summary: Migra dal layer legacy di retrocompatibilità al moderno Plugin SDK
title: Migrazione del Plugin SDK
x-i18n:
    generated_at: "2026-04-06T03:10:03Z"
    model: gpt-5.4
    provider: openai
    source_hash: b71ce69b30c3bb02da1b263b1d11dc3214deae5f6fc708515e23b5a1c7bb7c8f
    source_path: plugins/sdk-migration.md
    workflow: 15
---

# Migrazione del Plugin SDK

OpenClaw è passato da un ampio layer di retrocompatibilità a una moderna
architettura dei plugin con importazioni mirate e documentate. Se il tuo plugin è stato creato prima
della nuova architettura, questa guida ti aiuta a migrare.

## Cosa sta cambiando

Il vecchio sistema di plugin forniva due superfici molto ampie che permettevano ai plugin di importare
qualsiasi cosa servisse da un unico punto di ingresso:

- **`openclaw/plugin-sdk/compat`** — una singola importazione che riesportava decine di
  helper. È stata introdotta per mantenere funzionanti i vecchi plugin basati su hook mentre la
  nuova architettura dei plugin veniva costruita.
- **`openclaw/extension-api`** — un bridge che dava ai plugin accesso diretto agli
  helper lato host come l'embedded agent runner.

Entrambe le superfici sono ora **deprecate**. Funzionano ancora a runtime, ma i nuovi
plugin non devono usarle e i plugin esistenti dovrebbero migrare prima che la prossima
major release le rimuova.

<Warning>
  Il layer di retrocompatibilità verrà rimosso in una futura major release.
  I plugin che continuano a importare da queste superfici smetteranno di funzionare quando ciò accadrà.
</Warning>

## Perché questo è cambiato

Il vecchio approccio causava problemi:

- **Avvio lento** — importare un helper caricava decine di moduli non correlati
- **Dipendenze circolari** — le riesportazioni ampie rendevano facile creare cicli di importazione
- **Superficie API poco chiara** — non c'era modo di capire quali esportazioni fossero stabili rispetto a quelle interne

Il moderno Plugin SDK risolve questo problema: ogni percorso di importazione (`openclaw/plugin-sdk/\<subpath\>`)
è un modulo piccolo e autosufficiente con uno scopo chiaro e un contratto documentato.

Sono state rimosse anche le vecchie superfici di comodo dei provider per i canali bundled. Le importazioni
come `openclaw/plugin-sdk/slack`, `openclaw/plugin-sdk/discord`,
`openclaw/plugin-sdk/signal`, `openclaw/plugin-sdk/whatsapp`,
superfici helper con marchio del canale e
`openclaw/plugin-sdk/telegram-core` erano scorciatoie private del mono-repo, non
contratti plugin stabili. Usa invece sottopercorsi SDK generici e stretti. All'interno del
workspace dei plugin bundled, mantieni gli helper di proprietà del provider nel proprio
`api.ts` o `runtime-api.ts` di quel plugin.

Esempi attuali di provider bundled:

- Anthropic mantiene gli helper di stream specifici di Claude nella propria superficie `api.ts` /
  `contract-api.ts`
- OpenAI mantiene builder provider, helper per i modelli predefiniti e builder di provider
  realtime nel proprio `api.ts`
- OpenRouter mantiene nel proprio `api.ts` il builder del provider e gli helper di onboarding/configurazione

## Come migrare

<Steps>
  <Step title="Verifica il comportamento di fallback del wrapper Windows">
    Se il tuo plugin usa `openclaw/plugin-sdk/windows-spawn`, i wrapper Windows
    `.cmd`/`.bat` non risolti ora falliscono in modalità fail-closed a meno che tu non passi esplicitamente
    `allowShellFallback: true`.

    ```typescript
    // Before
    const program = applyWindowsSpawnProgramPolicy({ candidate });

    // After
    const program = applyWindowsSpawnProgramPolicy({
      candidate,
      // Only set this for trusted compatibility callers that intentionally
      // accept shell-mediated fallback.
      allowShellFallback: true,
    });
    ```

    Se il tuo chiamante non dipende intenzionalmente dal fallback della shell, non impostare
    `allowShellFallback` e gestisci invece l'errore lanciato.

  </Step>

  <Step title="Trova le importazioni deprecate">
    Cerca nel tuo plugin le importazioni da una delle due superfici deprecate:

    ```bash
    grep -r "plugin-sdk/compat" my-plugin/
    grep -r "openclaw/extension-api" my-plugin/
    ```

  </Step>

  <Step title="Sostituisci con importazioni mirate">
    Ogni esportazione della vecchia superficie corrisponde a uno specifico percorso di importazione moderno:

    ```typescript
    // Before (deprecated backwards-compatibility layer)
    import {
      createChannelReplyPipeline,
      createPluginRuntimeStore,
      resolveControlCommandGate,
    } from "openclaw/plugin-sdk/compat";

    // After (modern focused imports)
    import { createChannelReplyPipeline } from "openclaw/plugin-sdk/channel-reply-pipeline";
    import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
    import { resolveControlCommandGate } from "openclaw/plugin-sdk/command-auth";
    ```

    Per gli helper lato host, usa il runtime del plugin iniettato invece di importare
    direttamente:

    ```typescript
    // Before (deprecated extension-api bridge)
    import { runEmbeddedPiAgent } from "openclaw/extension-api";
    const result = await runEmbeddedPiAgent({ sessionId, prompt });

    // After (injected runtime)
    const result = await api.runtime.agent.runEmbeddedPiAgent({ sessionId, prompt });
    ```

    Lo stesso schema si applica ad altri helper legacy del bridge:

    | Vecchia importazione | Equivalente moderno |
    | --- | --- |
    | `resolveAgentDir` | `api.runtime.agent.resolveAgentDir` |
    | `resolveAgentWorkspaceDir` | `api.runtime.agent.resolveAgentWorkspaceDir` |
    | `resolveAgentIdentity` | `api.runtime.agent.resolveAgentIdentity` |
    | `resolveThinkingDefault` | `api.runtime.agent.resolveThinkingDefault` |
    | `resolveAgentTimeoutMs` | `api.runtime.agent.resolveAgentTimeoutMs` |
    | `ensureAgentWorkspace` | `api.runtime.agent.ensureAgentWorkspace` |
    | helper dell'archivio sessioni | `api.runtime.agent.session.*` |

  </Step>

  <Step title="Compila e testa">
    ```bash
    pnpm build
    pnpm test -- my-plugin/
    ```
  </Step>
</Steps>

## Riferimento dei percorsi di importazione

<Accordion title="Tabella dei percorsi di importazione comuni">
  | Percorso di importazione | Scopo | Esportazioni principali |
  | --- | --- | --- |
  | `plugin-sdk/plugin-entry` | Helper canonico del punto di ingresso plugin | `definePluginEntry` |
  | `plugin-sdk/core` | Riesportazione legacy ombrello per definizioni/builder di entry dei canali | `defineChannelPluginEntry`, `createChatChannelPlugin` |
  | `plugin-sdk/config-schema` | Esportazione dello schema di configurazione root | `OpenClawSchema` |
  | `plugin-sdk/provider-entry` | Helper di entry per singolo provider | `defineSingleProviderPluginEntry` |
  | `plugin-sdk/channel-core` | Definizioni e builder focalizzati per le entry dei canali | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
  | `plugin-sdk/setup` | Helper condivisi del wizard di setup | Prompt di allowlist, builder dello stato di setup |
  | `plugin-sdk/setup-runtime` | Helper runtime per il setup | Adattatori di patch setup import-safe, helper per note di lookup, `promptResolvedAllowFrom`, `splitSetupEntries`, proxy di setup delegati |
  | `plugin-sdk/setup-adapter-runtime` | Helper per l'adattatore di setup | `createEnvPatchedAccountSetupAdapter` |
  | `plugin-sdk/setup-tools` | Helper degli strumenti di setup | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
  | `plugin-sdk/account-core` | Helper multi-account | Helper per elenco account/config/action-gate |
  | `plugin-sdk/account-id` | Helper per l'ID account | `DEFAULT_ACCOUNT_ID`, normalizzazione dell'ID account |
  | `plugin-sdk/account-resolution` | Helper per la lookup degli account | Helper di lookup account + fallback predefinito |
  | `plugin-sdk/account-helpers` | Helper ristretti per gli account | Helper per elenco account/azioni account |
  | `plugin-sdk/channel-setup` | Adattatori del wizard di setup | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, più `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
  | `plugin-sdk/channel-pairing` | Primitive di pairing DM | `createChannelPairingController` |
  | `plugin-sdk/channel-reply-pipeline` | Wiring di prefisso risposta + digitazione | `createChannelReplyPipeline` |
  | `plugin-sdk/channel-config-helpers` | Factory di adattatori config | `createHybridChannelConfigAdapter` |
  | `plugin-sdk/channel-config-schema` | Builder di schema config | Tipi di schema della configurazione del canale |
  | `plugin-sdk/telegram-command-config` | Helper della configurazione dei comandi Telegram | Normalizzazione del nome del comando, trim della descrizione, validazione di duplicati/conflitti |
  | `plugin-sdk/channel-policy` | Risoluzione delle policy gruppo/DM | `resolveChannelGroupRequireMention` |
  | `plugin-sdk/channel-lifecycle` | Tracciamento dello stato dell'account | `createAccountStatusSink` |
  | `plugin-sdk/inbound-envelope` | Helper per l'envelope inbound | Helper condivisi per route + builder dell'envelope |
  | `plugin-sdk/inbound-reply-dispatch` | Helper per le risposte inbound | Helper condivisi per record-and-dispatch |
  | `plugin-sdk/messaging-targets` | Parsing dei target di messaggistica | Helper per parsing/matching dei target |
  | `plugin-sdk/outbound-media` | Helper per i media outbound | Caricamento condiviso dei media outbound |
  | `plugin-sdk/outbound-runtime` | Helper runtime outbound | Helper delegati per identità/send outbound |
  | `plugin-sdk/thread-bindings-runtime` | Helper per i thread binding | Lifecycle dei thread binding e helper degli adattatori |
  | `plugin-sdk/agent-media-payload` | Helper legacy per il payload media | Builder del payload media agente per layout di campi legacy |
  | `plugin-sdk/channel-runtime` | Shim di compatibilità deprecato | Solo utility runtime legacy del canale |
  | `plugin-sdk/channel-send-result` | Tipi del risultato di invio | Tipi del risultato della risposta |
  | `plugin-sdk/runtime-store` | Storage persistente del plugin | `createPluginRuntimeStore` |
  | `plugin-sdk/runtime` | Helper runtime ampi | Helper per runtime/logging/backup/installazione plugin |
  | `plugin-sdk/runtime-env` | Helper ristretti dell'ambiente runtime | Logger/runtime env, helper per timeout, retry e backoff |
  | `plugin-sdk/plugin-runtime` | Helper runtime condivisi del plugin | Helper per comandi/hook/http/interattivi del plugin |
  | `plugin-sdk/hook-runtime` | Helper della pipeline hook | Helper condivisi per pipeline di webhook/hook interni |
  | `plugin-sdk/lazy-runtime` | Helper runtime lazy | `createLazyRuntimeModule`, `createLazyRuntimeMethod`, `createLazyRuntimeMethodBinder`, `createLazyRuntimeNamedExport`, `createLazyRuntimeSurface` |
  | `plugin-sdk/process-runtime` | Helper di processo | Helper exec condivisi |
  | `plugin-sdk/cli-runtime` | Helper runtime CLI | Formattazione dei comandi, attese, helper di versione |
  | `plugin-sdk/gateway-runtime` | Helper del gateway | Client gateway e helper di patch dello stato dei canali |
  | `plugin-sdk/config-runtime` | Helper di configurazione | Helper di caricamento/scrittura della configurazione |
  | `plugin-sdk/telegram-command-config` | Helper dei comandi Telegram | Helper di validazione dei comandi Telegram con fallback stabile quando la superficie contrattuale Telegram bundled non è disponibile |
  | `plugin-sdk/approval-runtime` | Helper per i prompt di approvazione | Payload di approvazione exec/plugin, helper di capability/profilo di approvazione, helper runtime/instradamento di approvazione nativa |
  | `plugin-sdk/approval-auth-runtime` | Helper auth delle approvazioni | Risoluzione dell'approvatore, auth per azione nella stessa chat |
  | `plugin-sdk/approval-client-runtime` | Helper client delle approvazioni | Helper nativi di profilo/filtro per approvazione exec |
  | `plugin-sdk/approval-delivery-runtime` | Helper delivery delle approvazioni | Adattatori di capability/delivery per approvazione nativa |
  | `plugin-sdk/approval-native-runtime` | Helper per il target di approvazione | Helper di binding target/account per approvazione nativa |
  | `plugin-sdk/approval-reply-runtime` | Helper per le risposte di approvazione | Helper di payload di risposta per approvazione exec/plugin |
  | `plugin-sdk/security-runtime` | Helper di sicurezza | Helper condivisi per trust, controllo DM, contenuti esterni e raccolta dei segreti |
  | `plugin-sdk/ssrf-policy` | Helper della policy SSRF | Helper di allowlist host e policy di rete privata |
  | `plugin-sdk/ssrf-runtime` | Helper runtime SSRF | Pinned-dispatcher, fetch protetto, helper di policy SSRF |
  | `plugin-sdk/collection-runtime` | Helper di cache limitata | `pruneMapToMaxSize` |
  | `plugin-sdk/diagnostic-runtime` | Helper di gating diagnostico | `isDiagnosticFlagEnabled`, `isDiagnosticsEnabled` |
  | `plugin-sdk/error-runtime` | Helper di formattazione degli errori | `formatUncaughtError`, `isApprovalNotFoundError`, helper del grafo degli errori |
  | `plugin-sdk/fetch-runtime` | Helper di fetch/proxy incapsulati | `resolveFetch`, helper proxy |
  | `plugin-sdk/host-runtime` | Helper di normalizzazione host | `normalizeHostname`, `normalizeScpRemoteHost` |
  | `plugin-sdk/retry-runtime` | Helper di retry | `RetryConfig`, `retryAsync`, esecutori di policy |
  | `plugin-sdk/allow-from` | Formattazione della allowlist | `formatAllowFromLowercase` |
  | `plugin-sdk/allowlist-resolution` | Mappatura dell'input della allowlist | `mapAllowlistResolutionInputs` |
  | `plugin-sdk/command-auth` | Gating dei comandi e helper della superficie dei comandi | `resolveControlCommandGate`, helper di autorizzazione del mittente, helper del registro comandi |
  | `plugin-sdk/secret-input` | Parsing dell'input dei segreti | Helper per l'input dei segreti |
  | `plugin-sdk/webhook-ingress` | Helper per richieste webhook | Utility per i target webhook |
  | `plugin-sdk/webhook-request-guards` | Helper delle guardie per il body del webhook | Helper per lettura/limite del body della richiesta |
  | `plugin-sdk/reply-runtime` | Runtime condiviso della risposta | Dispatch inbound, heartbeat, pianificatore di risposte, chunking |
  | `plugin-sdk/reply-dispatch-runtime` | Helper ristretti per il dispatch delle risposte | Helper di finalize + provider dispatch |
  | `plugin-sdk/reply-history` | Helper della cronologia delle risposte | `buildHistoryContext`, `buildPendingHistoryContextFromMap`, `recordPendingHistoryEntry`, `clearHistoryEntriesIfEnabled` |
  | `plugin-sdk/reply-reference` | Pianificazione del riferimento della risposta | `createReplyReferencePlanner` |
  | `plugin-sdk/reply-chunking` | Helper dei chunk di risposta | Helper di chunking del testo/markdown |
  | `plugin-sdk/session-store-runtime` | Helper dell'archivio sessioni | Helper per percorso dell'archivio + updated-at |
  | `plugin-sdk/state-paths` | Helper dei percorsi di stato | Helper per directory di stato e OAuth |
  | `plugin-sdk/routing` | Helper di routing/chiave di sessione | `resolveAgentRoute`, `buildAgentSessionKey`, `resolveDefaultAgentBoundAccountId`, helper di normalizzazione della chiave di sessione |
  | `plugin-sdk/status-helpers` | Helper dello stato del canale | Builder del riepilogo di stato canale/account, valori predefiniti dello stato runtime, helper per i metadati dei problemi |
  | `plugin-sdk/target-resolver-runtime` | Helper del risolutore del target | Helper condivisi del risolutore del target |
  | `plugin-sdk/string-normalization-runtime` | Helper di normalizzazione delle stringhe | Helper per normalizzazione di slug/stringhe |
  | `plugin-sdk/request-url` | Helper per URL della richiesta | Estrai URL stringa da input simili a request |
  | `plugin-sdk/run-command` | Helper per comandi temporizzati | Runner di comandi temporizzati con stdout/stderr normalizzati |
  | `plugin-sdk/param-readers` | Lettori di parametri | Lettori comuni di parametri tool/CLI |
  | `plugin-sdk/tool-send` | Estrazione del send del tool | Estrai campi target di invio canonici dagli argomenti del tool |
  | `plugin-sdk/temp-path` | Helper per i percorsi temporanei | Helper condivisi per i percorsi di download temporaneo |
  | `plugin-sdk/logging-core` | Helper di logging | Logger del sottosistema e helper di redazione |
  | `plugin-sdk/markdown-table-runtime` | Helper per tabelle Markdown | Helper per la modalità tabelle Markdown |
  | `plugin-sdk/reply-payload` | Tipi della risposta del messaggio | Tipi del payload della risposta |
  | `plugin-sdk/provider-setup` | Helper curati di setup del provider locale/self-hosted | Helper di discovery/configurazione del provider self-hosted |
  | `plugin-sdk/self-hosted-provider-setup` | Helper focalizzati di setup del provider self-hosted compatibile con OpenAI | Gli stessi helper di discovery/configurazione del provider self-hosted |
  | `plugin-sdk/provider-auth-runtime` | Helper runtime auth del provider | Helper runtime per la risoluzione della chiave API |
  | `plugin-sdk/provider-auth-api-key` | Helper di setup della chiave API del provider | Helper di onboarding/scrittura profilo per chiave API |
  | `plugin-sdk/provider-auth-result` | Helper del risultato auth del provider | Builder standard del risultato auth OAuth |
  | `plugin-sdk/provider-auth-login` | Helper di login interattivo del provider | Helper condivisi di login interattivo |
  | `plugin-sdk/provider-env-vars` | Helper per env var del provider | Helper di lookup delle env var auth del provider |
  | `plugin-sdk/provider-model-shared` | Helper condivisi per modello/replay del provider | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, builder condivisi della policy di replay, helper degli endpoint del provider e helper di normalizzazione dell'ID del modello |
  | `plugin-sdk/provider-catalog-shared` | Helper condivisi per il catalogo del provider | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
  | `plugin-sdk/provider-onboard` | Patch di onboarding del provider | Helper di configurazione dell'onboarding |
  | `plugin-sdk/provider-http` | Helper HTTP del provider | Helper generici per capability HTTP/endpoint del provider |
  | `plugin-sdk/provider-web-fetch` | Helper web-fetch del provider | Helper di registrazione/cache del provider web-fetch |
  | `plugin-sdk/provider-web-search` | Helper web-search del provider | Helper di registrazione/cache/configurazione del provider web-search |
  | `plugin-sdk/provider-tools` | Helper di compatibilità tool/schema del provider | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, pulizia dello schema Gemini + diagnostica, e helper di compatibilità xAI come `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
  | `plugin-sdk/provider-usage` | Helper di utilizzo del provider | `fetchClaudeUsage`, `fetchGeminiUsage`, `fetchGithubCopilotUsage` e altri helper di utilizzo del provider |
  | `plugin-sdk/provider-stream` | Helper wrapper degli stream del provider | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, tipi di wrapper degli stream, e helper wrapper condivisi per Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot |
  | `plugin-sdk/keyed-async-queue` | Coda asincrona ordinata | `KeyedAsyncQueue` |
  | `plugin-sdk/media-runtime` | Helper media condivisi | Helper di fetch/transform/store dei media più builder del payload media |
  | `plugin-sdk/media-understanding` | Helper di media-understanding | Tipi del provider di media understanding più esportazioni helper lato provider per immagini/audio |
  | `plugin-sdk/text-runtime` | Helper testuali condivisi | Rimozione del testo visibile all'assistant, helper di render/chunking/tabella markdown, helper di redazione, helper di directive-tag, utility di testo sicuro e helper correlati di testo/logging |
  | `plugin-sdk/text-chunking` | Helper di chunking del testo | Helper di chunking del testo outbound |
  | `plugin-sdk/speech` | Helper speech | Tipi del provider speech più helper lato provider per direttive, registro e validazione |
  | `plugin-sdk/speech-core` | Core speech condiviso | Tipi del provider speech, registro, direttive, normalizzazione |
  | `plugin-sdk/realtime-transcription` | Helper per trascrizione realtime | Tipi del provider e helper del registro |
  | `plugin-sdk/realtime-voice` | Helper per voce realtime | Tipi del provider e helper del registro |
  | `plugin-sdk/image-generation-core` | Core condiviso per generazione immagini | Tipi per generazione immagini, failover, auth e helper del registro |
  | `plugin-sdk/music-generation` | Helper per generazione musicale | Tipi provider/richiesta/risultato per generazione musicale |
  | `plugin-sdk/music-generation-core` | Core condiviso per generazione musicale | Tipi per generazione musicale, helper di failover, lookup provider e parsing dei riferimenti modello |
  | `plugin-sdk/video-generation` | Helper per generazione video | Tipi provider/richiesta/risultato per generazione video |
  | `plugin-sdk/video-generation-core` | Core condiviso per generazione video | Tipi per generazione video, helper di failover, lookup provider e parsing dei riferimenti modello |
  | `plugin-sdk/interactive-runtime` | Helper di risposta interattiva | Normalizzazione/riduzione del payload di risposta interattiva |
  | `plugin-sdk/channel-config-primitives` | Primitive della configurazione del canale | Primitive ristrette dello schema config del canale |
  | `plugin-sdk/channel-config-writes` | Helper di scrittura della configurazione del canale | Helper di autorizzazione per la scrittura della configurazione del canale |
  | `plugin-sdk/channel-plugin-common` | Prelude condivisa del canale | Esportazioni della prelude condivisa del plugin canale |
  | `plugin-sdk/channel-status` | Helper dello stato del canale | Helper condivisi per snapshot/riepilogo dello stato del canale |
  | `plugin-sdk/allowlist-config-edit` | Helper della configurazione allowlist | Helper per modifica/lettura della configurazione allowlist |
  | `plugin-sdk/group-access` | Helper di accesso al gruppo | Helper condivisi per le decisioni di accesso al gruppo |
  | `plugin-sdk/direct-dm` | Helper direct-DM | Helper condivisi per auth/guard dei direct-DM |
  | `plugin-sdk/extension-shared` | Helper condivisi delle estensioni | Primitive helper per canali passivi/stato |
  | `plugin-sdk/webhook-targets` | Helper per i target webhook | Registro dei target webhook e helper di installazione delle route |
  | `plugin-sdk/webhook-path` | Helper del percorso webhook | Helper di normalizzazione del percorso webhook |
  | `plugin-sdk/web-media` | Helper condivisi per i media web | Helper di caricamento dei media remoti/locali |
  | `plugin-sdk/zod` | Riesportazione Zod | Riesportazione di `zod` per i consumatori del Plugin SDK |
  | `plugin-sdk/memory-core` | Helper bundled memory-core | Superficie helper di memory manager/config/file/CLI |
  | `plugin-sdk/memory-core-engine-runtime` | Facade runtime del motore memory | Facade runtime di indicizzazione/ricerca memory |
  | `plugin-sdk/memory-core-host-engine-foundation` | Motore foundation host memory | Esportazioni del motore foundation host memory |
  | `plugin-sdk/memory-core-host-engine-embeddings` | Motore embeddings host memory | Esportazioni del motore embeddings host memory |
  | `plugin-sdk/memory-core-host-engine-qmd` | Motore QMD host memory | Esportazioni del motore QMD host memory |
  | `plugin-sdk/memory-core-host-engine-storage` | Motore storage host memory | Esportazioni del motore storage host memory |
  | `plugin-sdk/memory-core-host-multimodal` | Helper multimodali host memory | Helper multimodali host memory |
  | `plugin-sdk/memory-core-host-query` | Helper query host memory | Helper query host memory |
  | `plugin-sdk/memory-core-host-secret` | Helper secret host memory | Helper secret host memory |
  | `plugin-sdk/memory-core-host-status` | Helper stato host memory | Helper stato host memory |
  | `plugin-sdk/memory-core-host-runtime-cli` | Runtime CLI host memory | Helper runtime CLI host memory |
  | `plugin-sdk/memory-core-host-runtime-core` | Runtime core host memory | Helper runtime core host memory |
  | `plugin-sdk/memory-core-host-runtime-files` | Helper file/runtime host memory | Helper file/runtime host memory |
  | `plugin-sdk/memory-lancedb` | Helper bundled memory-lancedb | Superficie helper memory-lancedb |
  | `plugin-sdk/testing` | Utility di test | Helper e mock di test |
</Accordion>

Questa tabella è intenzionalmente il sottoinsieme comune per la migrazione, non l'intera
superficie dell'SDK. L'elenco completo degli oltre 200 entrypoint si trova in
`scripts/lib/plugin-sdk-entrypoints.json`.

Questo elenco include ancora alcune superfici helper dei plugin bundled come
`plugin-sdk/feishu`, `plugin-sdk/feishu-setup`, `plugin-sdk/zalo`,
`plugin-sdk/zalo-setup` e `plugin-sdk/matrix*`. Restano esportate per
manutenzione e compatibilità dei plugin bundled, ma sono intenzionalmente
omesse dalla tabella comune di migrazione e non sono la destinazione consigliata per
nuovo codice plugin.

La stessa regola si applica ad altre famiglie di helper bundled come:

- helper di supporto browser: `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support`
- Matrix: `plugin-sdk/matrix*`
- LINE: `plugin-sdk/line*`
- IRC: `plugin-sdk/irc*`
- superfici helper/plugin bundled come `plugin-sdk/googlechat`,
  `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles*`,
  `plugin-sdk/mattermost*`, `plugin-sdk/msteams`,
  `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`,
  `plugin-sdk/twitch`,
  `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`,
  `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`,
  `plugin-sdk/thread-ownership` e `plugin-sdk/voice-call`

`plugin-sdk/github-copilot-token` espone attualmente la ristretta
superficie helper dei token `DEFAULT_COPILOT_API_BASE_URL`,
`deriveCopilotApiBaseUrlFromToken` e `resolveCopilotApiToken`.

Usa l'importazione più stretta che corrisponde al lavoro da svolgere. Se non riesci a trovare un'esportazione,
controlla il sorgente in `src/plugin-sdk/` o chiedi su Discord.

## Timeline di rimozione

| Quando                 | Cosa succede                                                           |
| ---------------------- | ---------------------------------------------------------------------- |
| **Adesso**             | Le superfici deprecate emettono avvisi a runtime                       |
| **Prossima major release** | Le superfici deprecate verranno rimosse; i plugin che le usano ancora smetteranno di funzionare |

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
- [Panoramica SDK](/it/plugins/sdk-overview) — riferimento completo delle importazioni per sottopercorso
- [Plugin canale](/it/plugins/sdk-channel-plugins) — creazione di plugin canale
- [Plugin provider](/it/plugins/sdk-provider-plugins) — creazione di plugin provider
- [Interni dei plugin](/it/plugins/architecture) — approfondimento sull'architettura
- [Manifesto plugin](/it/plugins/manifest) — riferimento dello schema del manifesto
