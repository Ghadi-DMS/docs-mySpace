---
read_when:
    - Devi sapere da quale sottopercorso SDK importare
    - Vuoi un riferimento per tutti i metodi di registrazione su OpenClawPluginApi
    - Stai cercando una specifica esportazione dell'SDK
sidebarTitle: SDK Overview
summary: Riferimento della mappa di importazione, dell'API di registrazione e dell'architettura SDK
title: Panoramica del Plugin SDK
x-i18n:
    generated_at: "2026-04-06T03:10:57Z"
    model: gpt-5.4
    provider: openai
    source_hash: d801641f26f39dc21490d2a69a337ff1affb147141360916b8b58a267e9f822a
    source_path: plugins/sdk-overview.md
    workflow: 15
---

# Panoramica del Plugin SDK

Il plugin SDK è il contratto tipizzato tra plugin e core. Questa pagina è il
riferimento per **cosa importare** e **cosa puoi registrare**.

<Tip>
  **Cerchi una guida pratica?**
  - Primo plugin? Inizia con [Getting Started](/it/plugins/building-plugins)
  - Plugin canale? Vedi [Channel Plugins](/it/plugins/sdk-channel-plugins)
  - Plugin provider? Vedi [Provider Plugins](/it/plugins/sdk-provider-plugins)
</Tip>

## Convenzione di importazione

Importa sempre da un sottopercorso specifico:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
```

Ogni sottopercorso è un modulo piccolo e autosufficiente. Questo mantiene
l'avvio rapido e previene problemi di dipendenze circolari. Per gli helper di
entry/build specifici del canale, preferisci `openclaw/plugin-sdk/channel-core`;
mantieni `openclaw/plugin-sdk/core` per la superficie ombrello più ampia e per
gli helper condivisi come `buildChannelConfigSchema`.

Non aggiungere né dipendere da seam di convenienza con nome del provider come
`openclaw/plugin-sdk/slack`, `openclaw/plugin-sdk/discord`,
`openclaw/plugin-sdk/signal`, `openclaw/plugin-sdk/whatsapp` o
seam helper con branding del canale. I plugin bundled dovrebbero comporre
sottopercorsi SDK generici all'interno dei propri barrel `api.ts` o `runtime-api.ts`, e il core
dovrebbe usare quei barrel locali al plugin oppure aggiungere un contratto SDK
generico e ristretto quando l'esigenza è davvero cross-channel.

La mappa di export generata contiene ancora un piccolo insieme di seam helper di plugin bundled
come `plugin-sdk/feishu`, `plugin-sdk/feishu-setup`,
`plugin-sdk/zalo`, `plugin-sdk/zalo-setup` e `plugin-sdk/matrix*`. Questi
sottopercorsi esistono solo per manutenzione e compatibilità dei plugin bundled;
sono volutamente omessi dalla tabella comune qui sotto e non sono il percorso di
importazione consigliato per nuovi plugin di terze parti.

## Riferimento dei sottopercorsi

I sottopercorsi usati più comunemente, raggruppati per scopo. L'elenco completo generato di
oltre 200 sottopercorsi si trova in `scripts/lib/plugin-sdk-entrypoints.json`.

I sottopercorsi helper riservati ai plugin bundled compaiono ancora in questo elenco generato.
Trattali come superfici di dettaglio implementativo/compatibilità a meno che una pagina della documentazione
non ne promuova esplicitamente una come pubblica.

### Entry del plugin

| Sottopercorso              | Esportazioni chiave                                                                                                                    |
| -------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/plugin-entry`   | `definePluginEntry`                                                                                                                    |
| `plugin-sdk/core`           | `defineChannelPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase`, `defineSetupPluginEntry`, `buildChannelConfigSchema` |
| `plugin-sdk/config-schema`  | `OpenClawSchema`                                                                                                                       |
| `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry`                                                                                                      |

<AccordionGroup>
  <Accordion title="Sottopercorsi dei canali">
    | Sottopercorso | Esportazioni chiave |
    | --- | --- |
    | `plugin-sdk/channel-core` | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
    | `plugin-sdk/config-schema` | Export dello schema Zod radice di `openclaw.json` (`OpenClawSchema`) |
    | `plugin-sdk/channel-setup` | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, più `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
    | `plugin-sdk/setup` | Helper condivisi per setup wizard, prompt allowlist, builder dello stato di setup |
    | `plugin-sdk/setup-runtime` | `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
    | `plugin-sdk/setup-adapter-runtime` | `createEnvPatchedAccountSetupAdapter` |
    | `plugin-sdk/setup-tools` | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
    | `plugin-sdk/account-core` | Helper per config/action-gate multi-account, helper di fallback per account predefinito |
    | `plugin-sdk/account-id` | `DEFAULT_ACCOUNT_ID`, helper di normalizzazione dell'ID account |
    | `plugin-sdk/account-resolution` | Ricerca account + helper di fallback al predefinito |
    | `plugin-sdk/account-helpers` | Helper ristretti per elenco account/azioni account |
    | `plugin-sdk/channel-pairing` | `createChannelPairingController` |
    | `plugin-sdk/channel-reply-pipeline` | `createChannelReplyPipeline` |
    | `plugin-sdk/channel-config-helpers` | `createHybridChannelConfigAdapter` |
    | `plugin-sdk/channel-config-schema` | Tipi di schema config del canale |
    | `plugin-sdk/telegram-command-config` | Helper di normalizzazione/validazione dei comandi personalizzati Telegram con fallback al contratto bundled |
    | `plugin-sdk/channel-policy` | `resolveChannelGroupRequireMention` |
    | `plugin-sdk/channel-lifecycle` | `createAccountStatusSink` |
    | `plugin-sdk/inbound-envelope` | Helper condivisi per route inbound + builder dell'envelope |
    | `plugin-sdk/inbound-reply-dispatch` | Helper condivisi per registrazione e dispatch inbound |
    | `plugin-sdk/messaging-targets` | Helper per parsing/corrispondenza dei target |
    | `plugin-sdk/outbound-media` | Helper condivisi per il caricamento dei media in uscita |
    | `plugin-sdk/outbound-runtime` | Helper per identità/delegati di invio in uscita |
    | `plugin-sdk/thread-bindings-runtime` | Helper per ciclo di vita e adapter dei thread binding |
    | `plugin-sdk/agent-media-payload` | Builder legacy del payload media dell'agente |
    | `plugin-sdk/conversation-runtime` | Helper per binding di conversazione/thread, pairing e binding configurati |
    | `plugin-sdk/runtime-config-snapshot` | Helper per snapshot di config runtime |
    | `plugin-sdk/runtime-group-policy` | Helper per la risoluzione della group policy runtime |
    | `plugin-sdk/channel-status` | Helper condivisi per snapshot/riepilogo dello stato del canale |
    | `plugin-sdk/channel-config-primitives` | Primitive ristrette di schema config del canale |
    | `plugin-sdk/channel-config-writes` | Helper di autorizzazione alle scritture config del canale |
    | `plugin-sdk/channel-plugin-common` | Export di preambolo condivisi per i plugin canale |
    | `plugin-sdk/allowlist-config-edit` | Helper di lettura/modifica config allowlist |
    | `plugin-sdk/group-access` | Helper condivisi per le decisioni di accesso ai gruppi |
    | `plugin-sdk/direct-dm` | Helper condivisi di auth/guard per DM diretti |
    | `plugin-sdk/interactive-runtime` | Helper di normalizzazione/riduzione del payload delle risposte interattive |
    | `plugin-sdk/channel-inbound` | Helper per debounce, corrispondenza menzioni, envelope |
    | `plugin-sdk/channel-send-result` | Tipi del risultato di risposta |
    | `plugin-sdk/channel-actions` | `createMessageToolButtonsSchema`, `createMessageToolCardSchema` |
    | `plugin-sdk/channel-targets` | Helper per parsing/corrispondenza dei target |
    | `plugin-sdk/channel-contract` | Tipi del contratto del canale |
    | `plugin-sdk/channel-feedback` | Wiring di feedback/reazioni |
  </Accordion>

  <Accordion title="Sottopercorsi dei provider">
    | Sottopercorso | Esportazioni chiave |
    | --- | --- |
    | `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry` |
    | `plugin-sdk/provider-setup` | Helper di setup curati per provider locali/self-hosted |
    | `plugin-sdk/self-hosted-provider-setup` | Helper di setup focalizzati per provider self-hosted compatibili OpenAI |
    | `plugin-sdk/provider-auth-runtime` | Helper di risoluzione runtime della chiave API per i plugin provider |
    | `plugin-sdk/provider-auth-api-key` | Helper di onboarding/scrittura profilo per chiavi API |
    | `plugin-sdk/provider-auth-result` | Builder standard del risultato auth OAuth |
    | `plugin-sdk/provider-auth-login` | Helper condivisi di login interattivo per i plugin provider |
    | `plugin-sdk/provider-env-vars` | Helper di lookup delle variabili env auth dei provider |
    | `plugin-sdk/provider-auth` | `createProviderApiKeyAuthMethod`, `ensureApiKeyFromOptionEnvOrPrompt`, `upsertAuthProfile` |
    | `plugin-sdk/provider-model-shared` | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, builder condivisi di replay-policy, helper per endpoint provider e helper di normalizzazione model-id come `normalizeNativeXaiModelId` |
    | `plugin-sdk/provider-catalog-shared` | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
    | `plugin-sdk/provider-http` | Helper generici di capacità HTTP/endpoint provider |
    | `plugin-sdk/provider-web-fetch` | Helper di registrazione/cache per provider web-fetch |
    | `plugin-sdk/provider-web-search` | Helper di registrazione/cache/config per provider web-search |
    | `plugin-sdk/provider-tools` | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, pulizia schema Gemini + diagnostica e helper di compatibilità xAI come `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
    | `plugin-sdk/provider-usage` | `fetchClaudeUsage` e simili |
    | `plugin-sdk/provider-stream` | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, tipi di stream wrapper e helper wrapper condivisi per Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot |
    | `plugin-sdk/provider-onboard` | Helper di patch config per onboarding |
    | `plugin-sdk/global-singleton` | Helper per singleton/mappe/cache locali al processo |
  </Accordion>

  <Accordion title="Sottopercorsi auth e sicurezza">
    | Sottopercorso | Esportazioni chiave |
    | --- | --- |
    | `plugin-sdk/command-auth` | `resolveControlCommandGate`, helper per registro comandi, helper di autorizzazione del mittente |
    | `plugin-sdk/approval-auth-runtime` | Helper per risoluzione degli approvatori e auth delle azioni nella stessa chat |
    | `plugin-sdk/approval-client-runtime` | Helper per profili/filtri di approvazione exec nativi |
    | `plugin-sdk/approval-delivery-runtime` | Adapter per capacità/consegna delle approvazioni native |
    | `plugin-sdk/approval-native-runtime` | Helper per target di approvazione nativa + binding account |
    | `plugin-sdk/approval-reply-runtime` | Helper per payload di risposta di approvazione exec/plugin |
    | `plugin-sdk/command-auth-native` | Auth dei comandi nativi + helper per target sessione nativa |
    | `plugin-sdk/command-detection` | Helper condivisi di rilevamento dei comandi |
    | `plugin-sdk/command-surface` | Helper per normalizzazione del corpo del comando e command-surface |
    | `plugin-sdk/allow-from` | `formatAllowFromLowercase` |
    | `plugin-sdk/security-runtime` | Helper condivisi per trust, gating DM, contenuti esterni e raccolta segreti |
    | `plugin-sdk/ssrf-policy` | Helper per policy SSRF con allowlist host e rete privata |
    | `plugin-sdk/ssrf-runtime` | Helper per pinned-dispatcher, fetch protetto da SSRF e policy SSRF |
    | `plugin-sdk/secret-input` | Helper per parsing dell'input segreto |
    | `plugin-sdk/webhook-ingress` | Helper per richiesta/target webhook |
    | `plugin-sdk/webhook-request-guards` | Helper per dimensione body richiesta/timeout |
  </Accordion>

  <Accordion title="Sottopercorsi runtime e storage">
    | Sottopercorso | Esportazioni chiave |
    | --- | --- |
    | `plugin-sdk/runtime` | Helper ampi per runtime/logging/backup/installazione plugin |
    | `plugin-sdk/runtime-env` | Helper ristretti per env runtime, logger, timeout, retry e backoff |
    | `plugin-sdk/runtime-store` | `createPluginRuntimeStore` |
    | `plugin-sdk/plugin-runtime` | Helper condivisi per comando/hook/http/interattivi dei plugin |
    | `plugin-sdk/hook-runtime` | Helper condivisi per la pipeline di hook webhook/interni |
    | `plugin-sdk/lazy-runtime` | Helper di importazione/binding runtime lazy come `createLazyRuntimeModule`, `createLazyRuntimeMethod` e `createLazyRuntimeSurface` |
    | `plugin-sdk/process-runtime` | Helper di esecuzione processi |
    | `plugin-sdk/cli-runtime` | Helper per formattazione CLI, attesa e versione |
    | `plugin-sdk/gateway-runtime` | Helper per client gateway e patch dello stato del canale |
    | `plugin-sdk/config-runtime` | Helper per caricamento/scrittura della config |
    | `plugin-sdk/telegram-command-config` | Normalizzazione di nome/descrizione dei comandi Telegram e controlli di duplicati/conflitti, anche quando la superficie di contratto Telegram bundled non è disponibile |
    | `plugin-sdk/approval-runtime` | Helper per approvazione exec/plugin, builder di approval-capability, helper auth/profilo, helper di routing/runtime nativi |
    | `plugin-sdk/reply-runtime` | Helper condivisi per runtime inbound/risposta, chunking, dispatch, heartbeat, pianificatore delle risposte |
    | `plugin-sdk/reply-dispatch-runtime` | Helper ristretti per dispatch/finalizzazione della risposta |
    | `plugin-sdk/reply-history` | Helper condivisi per la cronologia delle risposte a breve finestra come `buildHistoryContext`, `recordPendingHistoryEntry` e `clearHistoryEntriesIfEnabled` |
    | `plugin-sdk/reply-reference` | `createReplyReferencePlanner` |
    | `plugin-sdk/reply-chunking` | Helper ristretti per chunking di testo/markdown |
    | `plugin-sdk/session-store-runtime` | Helper per percorso store sessione + updated-at |
    | `plugin-sdk/state-paths` | Helper per percorsi state/OAuth dir |
    | `plugin-sdk/routing` | Helper per route/session-key/account binding come `resolveAgentRoute`, `buildAgentSessionKey` e `resolveDefaultAgentBoundAccountId` |
    | `plugin-sdk/status-helpers` | Helper condivisi per riepilogo stato canale/account, valori predefiniti dello stato runtime e metadati dei problemi |
    | `plugin-sdk/target-resolver-runtime` | Helper condivisi per resolver dei target |
    | `plugin-sdk/string-normalization-runtime` | Helper di normalizzazione slug/stringhe |
    | `plugin-sdk/request-url` | Estrae URL stringa da input simili a fetch/request |
    | `plugin-sdk/run-command` | Runner di comandi temporizzato con risultati stdout/stderr normalizzati |
    | `plugin-sdk/param-readers` | Reader comuni dei parametri tool/CLI |
    | `plugin-sdk/tool-send` | Estrae campi canonicali del target di invio dagli argomenti dello strumento |
    | `plugin-sdk/temp-path` | Helper condivisi per percorsi temporanei di download |
    | `plugin-sdk/logging-core` | Helper per logger del sottosistema e redazione |
    | `plugin-sdk/markdown-table-runtime` | Helper per la modalità tabella Markdown |
    | `plugin-sdk/json-store` | Piccoli helper di lettura/scrittura stato JSON |
    | `plugin-sdk/file-lock` | Helper di file-lock rientrante |
    | `plugin-sdk/persistent-dedupe` | Helper per cache dedupe persistente su disco |
    | `plugin-sdk/acp-runtime` | Helper per runtime/sessione ACP e reply-dispatch |
    | `plugin-sdk/agent-config-primitives` | Primitive ristrette di schema config runtime dell'agente |
    | `plugin-sdk/boolean-param` | Reader flessibile di parametri booleani |
    | `plugin-sdk/dangerous-name-runtime` | Helper di risoluzione per matching di nomi pericolosi |
    | `plugin-sdk/device-bootstrap` | Helper per bootstrap del dispositivo e token di pairing |
    | `plugin-sdk/extension-shared` | Primitive condivise per canali passivi e helper di stato |
    | `plugin-sdk/models-provider-runtime` | Helper di risposta del comando `/models`/provider |
    | `plugin-sdk/skill-commands-runtime` | Helper di elenco dei comandi Skills |
    | `plugin-sdk/native-command-registry` | Helper per registro/build/serializzazione dei comandi nativi |
    | `plugin-sdk/provider-zai-endpoint` | Helper di rilevamento endpoint Z.AI |
    | `plugin-sdk/infra-runtime` | Helper per eventi di sistema/heartbeat |
    | `plugin-sdk/collection-runtime` | Piccoli helper per cache limitate |
    | `plugin-sdk/diagnostic-runtime` | Helper per flag ed eventi diagnostici |
    | `plugin-sdk/error-runtime` | Grafo degli errori, formattazione, helper condivisi di classificazione errori, `isApprovalNotFoundError` |
    | `plugin-sdk/fetch-runtime` | Helper per fetch incapsulato, proxy e lookup fissati |
    | `plugin-sdk/host-runtime` | Helper di normalizzazione per hostname e host SCP |
    | `plugin-sdk/retry-runtime` | Helper per config retry e runner retry |
    | `plugin-sdk/agent-runtime` | Helper per agent dir/identity/workspace |
    | `plugin-sdk/directory-runtime` | Query/dedup delle directory supportata dalla config |
    | `plugin-sdk/keyed-async-queue` | `KeyedAsyncQueue` |
  </Accordion>

  <Accordion title="Sottopercorsi di capacità e test">
    | Sottopercorso | Esportazioni chiave |
    | --- | --- |
    | `plugin-sdk/media-runtime` | Helper condivisi per fetch/transform/store dei media più builder del payload media |
    | `plugin-sdk/media-understanding` | Tipi del provider media understanding più export helper lato provider per immagini/audio |
    | `plugin-sdk/text-runtime` | Helper condivisi per testo/markdown/logging come stripping del testo visibile all'assistente, helper di rendering/chunking/tabella markdown, helper di redazione, helper per tag direttiva e utility di testo sicuro |
    | `plugin-sdk/text-chunking` | Helper per chunking del testo in uscita |
    | `plugin-sdk/speech` | Tipi del provider speech più export helper lato provider per directive, registro e validazione |
    | `plugin-sdk/speech-core` | Tipi condivisi del provider speech, helper per registro, directive e normalizzazione |
    | `plugin-sdk/realtime-transcription` | Tipi del provider di trascrizione realtime e helper del registro |
    | `plugin-sdk/realtime-voice` | Tipi del provider voce realtime e helper del registro |
    | `plugin-sdk/image-generation` | Tipi del provider per generazione immagini |
    | `plugin-sdk/image-generation-core` | Tipi condivisi di generazione immagini, helper per failover, auth e registro |
    | `plugin-sdk/music-generation` | Tipi di provider/richiesta/risultato per generazione musicale |
    | `plugin-sdk/music-generation-core` | Tipi condivisi di generazione musicale, helper per failover, ricerca provider e parsing model-ref |
    | `plugin-sdk/video-generation` | Tipi di provider/richiesta/risultato per generazione video |
    | `plugin-sdk/video-generation-core` | Tipi condivisi di generazione video, helper per failover, ricerca provider e parsing model-ref |
    | `plugin-sdk/webhook-targets` | Registro dei target webhook e helper per installazione delle route |
    | `plugin-sdk/webhook-path` | Helper di normalizzazione del percorso webhook |
    | `plugin-sdk/web-media` | Helper condivisi per il caricamento di media remoti/locali |
    | `plugin-sdk/zod` | `zod` riesportato per i consumer del plugin SDK |
    | `plugin-sdk/testing` | `installCommonResolveTargetErrorCases`, `shouldAckReaction` |
  </Accordion>

  <Accordion title="Sottopercorsi memory">
    | Sottopercorso | Esportazioni chiave |
    | --- | --- |
    | `plugin-sdk/memory-core` | Superficie helper bundled memory-core per helper di manager/config/file/CLI |
    | `plugin-sdk/memory-core-engine-runtime` | Facciata runtime per indice/ricerca memory |
    | `plugin-sdk/memory-core-host-engine-foundation` | Export del motore foundation dell'host memory |
    | `plugin-sdk/memory-core-host-engine-embeddings` | Export del motore embeddings dell'host memory |
    | `plugin-sdk/memory-core-host-engine-qmd` | Export del motore QMD dell'host memory |
    | `plugin-sdk/memory-core-host-engine-storage` | Export del motore storage dell'host memory |
    | `plugin-sdk/memory-core-host-multimodal` | Helper multimodali dell'host memory |
    | `plugin-sdk/memory-core-host-query` | Helper di query dell'host memory |
    | `plugin-sdk/memory-core-host-secret` | Helper dei segreti dell'host memory |
    | `plugin-sdk/memory-core-host-status` | Helper di stato dell'host memory |
    | `plugin-sdk/memory-core-host-runtime-cli` | Helper runtime CLI dell'host memory |
    | `plugin-sdk/memory-core-host-runtime-core` | Helper runtime core dell'host memory |
    | `plugin-sdk/memory-core-host-runtime-files` | Helper file/runtime dell'host memory |
    | `plugin-sdk/memory-lancedb` | Superficie helper bundled memory-lancedb |
  </Accordion>

  <Accordion title="Sottopercorsi helper bundled riservati">
    | Famiglia | Sottopercorsi correnti | Utilizzo previsto |
    | --- | --- | --- |
    | Browser | `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support` | Helper di supporto del plugin browser bundled (`browser-support` resta il barrel di compatibilità) |
    | Matrix | `plugin-sdk/matrix`, `plugin-sdk/matrix-helper`, `plugin-sdk/matrix-runtime-heavy`, `plugin-sdk/matrix-runtime-shared`, `plugin-sdk/matrix-runtime-surface`, `plugin-sdk/matrix-surface`, `plugin-sdk/matrix-thread-bindings` | Superficie helper/runtime Matrix bundled |
    | Line | `plugin-sdk/line`, `plugin-sdk/line-core`, `plugin-sdk/line-runtime`, `plugin-sdk/line-surface` | Superficie helper/runtime LINE bundled |
    | IRC | `plugin-sdk/irc`, `plugin-sdk/irc-surface` | Superficie helper IRC bundled |
    | Helper specifici del canale | `plugin-sdk/googlechat`, `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles`, `plugin-sdk/bluebubbles-policy`, `plugin-sdk/mattermost`, `plugin-sdk/mattermost-policy`, `plugin-sdk/feishu-conversation`, `plugin-sdk/msteams`, `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`, `plugin-sdk/twitch` | Seam helper/compatibilità dei canali bundled |
    | Helper auth/plugin-specifici | `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`, `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`, `plugin-sdk/thread-ownership`, `plugin-sdk/voice-call` | Seam helper per funzionalità/plugin bundled; `plugin-sdk/github-copilot-token` attualmente esporta `DEFAULT_COPILOT_API_BASE_URL`, `deriveCopilotApiBaseUrlFromToken` e `resolveCopilotApiToken` |
  </Accordion>
</AccordionGroup>

## API di registrazione

La callback `register(api)` riceve un oggetto `OpenClawPluginApi` con questi
metodi:

### Registrazione delle capacità

| Metodo                                           | Cosa registra                    |
| ------------------------------------------------ | -------------------------------- |
| `api.registerProvider(...)`                      | Inferenza testuale (LLM)         |
| `api.registerChannel(...)`                       | Canale di messaggistica          |
| `api.registerSpeechProvider(...)`                | Sintesi text-to-speech / STT     |
| `api.registerRealtimeTranscriptionProvider(...)` | Trascrizione realtime in streaming |
| `api.registerRealtimeVoiceProvider(...)`         | Sessioni vocali realtime duplex  |
| `api.registerMediaUnderstandingProvider(...)`    | Analisi di immagini/audio/video  |
| `api.registerImageGenerationProvider(...)`       | Generazione di immagini          |
| `api.registerMusicGenerationProvider(...)`       | Generazione musicale             |
| `api.registerVideoGenerationProvider(...)`       | Generazione video                |
| `api.registerWebFetchProvider(...)`              | Provider di web fetch / scraping |
| `api.registerWebSearchProvider(...)`             | Ricerca web                      |

### Strumenti e comandi

| Metodo                          | Cosa registra                                 |
| ------------------------------- | --------------------------------------------- |
| `api.registerTool(tool, opts?)` | Strumento dell'agente (obbligatorio o `{ optional: true }`) |
| `api.registerCommand(def)`      | Comando personalizzato (bypassa l'LLM)        |

### Infrastruttura

| Metodo                                         | Cosa registra         |
| ---------------------------------------------- | --------------------- |
| `api.registerHook(events, handler, opts?)`     | Hook evento           |
| `api.registerHttpRoute(params)`                | Endpoint HTTP gateway |
| `api.registerGatewayMethod(name, handler)`     | Metodo RPC gateway    |
| `api.registerCli(registrar, opts?)`            | Sottocomando CLI      |
| `api.registerService(service)`                 | Servizio in background |
| `api.registerInteractiveHandler(registration)` | Gestore interattivo   |

I namespace amministrativi core riservati (`config.*`, `exec.approvals.*`, `wizard.*`,
`update.*`) rimangono sempre `operator.admin`, anche se un plugin prova ad assegnare
uno scope del metodo gateway più ristretto. Preferisci prefissi specifici del plugin per
i metodi posseduti dal plugin.

### Metadati di registrazione CLI

`api.registerCli(registrar, opts?)` accetta due tipi di metadati di primo livello:

- `commands`: root di comando esplicite possedute dal registrar
- `descriptors`: descrittori di comando in fase di parsing usati per help della CLI root,
  routing e registrazione CLI lazy del plugin

Se vuoi che un comando del plugin resti caricato lazy nel normale percorso della CLI root,
fornisci `descriptors` che coprano ogni root di comando di primo livello esposta da quel
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
        description: "Manage Matrix accounts, verification, devices, and profile state",
        hasSubcommands: true,
      },
    ],
  },
);
```

Usa `commands` da solo solo quando non hai bisogno della registrazione lazy della CLI root.
Quel percorso di compatibilità eager resta supportato, ma non installa
placeholder supportati da descriptor per il caricamento lazy in fase di parsing.

### Slot esclusivi

| Metodo                                     | Cosa registra                          |
| ------------------------------------------ | -------------------------------------- |
| `api.registerContextEngine(id, factory)`   | Motore di contesto (uno attivo alla volta) |
| `api.registerMemoryPromptSection(builder)` | Builder di sezione del prompt memory   |
| `api.registerMemoryFlushPlan(resolver)`    | Resolver del piano di flush memory     |
| `api.registerMemoryRuntime(runtime)`       | Adapter runtime memory                 |

### Adapter di embedding memory

| Metodo                                         | Cosa registra                                      |
| ---------------------------------------------- | -------------------------------------------------- |
| `api.registerMemoryEmbeddingProvider(adapter)` | Adapter di embedding memory per il plugin attivo   |

- `registerMemoryPromptSection`, `registerMemoryFlushPlan` e
  `registerMemoryRuntime` sono esclusivi dei plugin memory.
- `registerMemoryEmbeddingProvider` consente al plugin memory attivo di registrare uno
  o più ID di adapter embedding (per esempio `openai`, `gemini` o un ID personalizzato definito dal plugin).
- La config utente come `agents.defaults.memorySearch.provider` e
  `agents.defaults.memorySearch.fallback` viene risolta rispetto a quegli ID di adapter
  registrati.

### Eventi e ciclo di vita

| Metodo                                       | Cosa fa                     |
| -------------------------------------------- | --------------------------- |
| `api.on(hookName, handler, opts?)`           | Hook del ciclo di vita tipizzato |
| `api.onConversationBindingResolved(handler)` | Callback di binding della conversazione |

### Semantica delle decisioni degli hook

- `before_tool_call`: restituire `{ block: true }` è terminale. Una volta che un gestore lo imposta, i gestori a priorità più bassa vengono saltati.
- `before_tool_call`: restituire `{ block: false }` è trattato come nessuna decisione (uguale a omettere `block`), non come override.
- `before_install`: restituire `{ block: true }` è terminale. Una volta che un gestore lo imposta, i gestori a priorità più bassa vengono saltati.
- `before_install`: restituire `{ block: false }` è trattato come nessuna decisione (uguale a omettere `block`), non come override.
- `reply_dispatch`: restituire `{ handled: true, ... }` è terminale. Una volta che un gestore rivendica il dispatch, i gestori a priorità più bassa e il percorso predefinito di dispatch del modello vengono saltati.
- `message_sending`: restituire `{ cancel: true }` è terminale. Una volta che un gestore lo imposta, i gestori a priorità più bassa vengono saltati.
- `message_sending`: restituire `{ cancel: false }` è trattato come nessuna decisione (uguale a omettere `cancel`), non come override.

### Campi dell'oggetto API

| Campo                    | Tipo                      | Descrizione                                                                                  |
| ------------------------ | ------------------------- | -------------------------------------------------------------------------------------------- |
| `api.id`                 | `string`                  | ID del plugin                                                                                |
| `api.name`               | `string`                  | Nome visualizzato                                                                            |
| `api.version`            | `string?`                 | Versione del plugin (facoltativa)                                                            |
| `api.description`        | `string?`                 | Descrizione del plugin (facoltativa)                                                         |
| `api.source`             | `string`                  | Percorso sorgente del plugin                                                                 |
| `api.rootDir`            | `string?`                 | Directory root del plugin (facoltativa)                                                      |
| `api.config`             | `OpenClawConfig`          | Snapshot della config corrente (snapshot runtime in-memory attivo quando disponibile)        |
| `api.pluginConfig`       | `Record<string, unknown>` | Config specifica del plugin da `plugins.entries.<id>.config`                                 |
| `api.runtime`            | `PluginRuntime`           | [Helper runtime](/it/plugins/sdk-runtime)                                                       |
| `api.logger`             | `PluginLogger`            | Logger con scope (`debug`, `info`, `warn`, `error`)                                          |
| `api.registrationMode`   | `PluginRegistrationMode`  | Modalità di caricamento corrente; `"setup-runtime"` è la finestra leggera di avvio/setup pre-full-entry |
| `api.resolvePath(input)` | `(string) => string`      | Risolve il percorso relativo alla root del plugin                                            |

## Convenzione dei moduli interni

All'interno del tuo plugin, usa file barrel locali per le importazioni interne:

```
my-plugin/
  api.ts            # Export pubblici per consumer esterni
  runtime-api.ts    # Export runtime solo interni
  index.ts          # Punto di ingresso del plugin
  setup-entry.ts    # Entry leggera solo setup (facoltativa)
```

<Warning>
  Non importare mai il tuo stesso plugin tramite `openclaw/plugin-sdk/<your-plugin>`
  dal codice di produzione. Instrada le importazioni interne tramite `./api.ts` o
  `./runtime-api.ts`. Il percorso SDK è solo il contratto esterno.
</Warning>

Le superfici pubbliche dei plugin bundled caricate tramite facade (`api.ts`, `runtime-api.ts`,
`index.ts`, `setup-entry.ts` e file di entry pubblici simili) ora preferiscono lo
snapshot attivo della config runtime quando OpenClaw è già in esecuzione. Se non esiste ancora
uno snapshot runtime, usano come fallback il file di config risolto su disco.

I plugin provider possono anche esporre un barrel di contratto locale al plugin quando un
helper è intenzionalmente specifico del provider e non appartiene ancora a un sottopercorso SDK
generico. Esempio bundled attuale: il provider Anthropic mantiene i propri helper
stream Claude nel proprio seam pubblico `api.ts` / `contract-api.ts` invece di
promuovere la logica degli header beta Anthropic e `service_tier` in un contratto
generico `plugin-sdk/*`.

Altri esempi bundled attuali:

- `@openclaw/openai-provider`: `api.ts` esporta builder dei provider,
  helper dei modelli predefiniti e builder dei provider realtime
- `@openclaw/openrouter-provider`: `api.ts` esporta il builder del provider più
  helper di onboarding/config

<Warning>
  Anche il codice di produzione delle estensioni dovrebbe evitare importazioni
  `openclaw/plugin-sdk/<other-plugin>`. Se un helper è davvero condiviso, promuovilo a un sottopercorso SDK neutrale
  come `openclaw/plugin-sdk/speech`, `.../provider-model-shared` o un'altra
  superficie orientata alla capacità invece di accoppiare due plugin tra loro.
</Warning>

## Correlati

- [Entry Points](/it/plugins/sdk-entrypoints) — opzioni di `definePluginEntry` e `defineChannelPluginEntry`
- [Runtime Helpers](/it/plugins/sdk-runtime) — riferimento completo del namespace `api.runtime`
- [Setup and Config](/it/plugins/sdk-setup) — packaging, manifest, schemi di config
- [Testing](/it/plugins/sdk-testing) — utility di test e regole lint
- [SDK Migration](/it/plugins/sdk-migration) — migrazione dalle superfici deprecate
- [Plugin Internals](/it/plugins/architecture) — architettura approfondita e modello delle capacità
