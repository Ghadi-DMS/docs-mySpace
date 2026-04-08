---
read_when:
    - Devi sapere da quale sottopercorso dell'SDK importare
    - Vuoi un riferimento per tutti i metodi di registrazione su OpenClawPluginApi
    - Stai cercando una specifica esportazione dell'SDK
sidebarTitle: SDK Overview
summary: Mappa degli import, riferimento dell'API di registrazione e architettura dell'SDK
title: Panoramica del Plugin SDK
x-i18n:
    generated_at: "2026-04-08T08:13:33Z"
    model: gpt-5.4
    provider: openai
    source_hash: 6e7b420eb0f3faa8916357d52df949f6c9a46f1c843a1e6a0c0b8bb26db6cbff
    source_path: plugins/sdk-overview.md
    workflow: 15
---

# Panoramica del Plugin SDK

Il plugin SDK è il contratto tipizzato tra i plugin e il core. Questa pagina è il
riferimento per **cosa importare** e **cosa puoi registrare**.

<Tip>
  **Cerchi una guida pratica?**
  - Primo plugin? Inizia con [Getting Started](/it/plugins/building-plugins)
  - Plugin di canale? Vedi [Channel Plugins](/it/plugins/sdk-channel-plugins)
  - Plugin provider? Vedi [Provider Plugins](/it/plugins/sdk-provider-plugins)
</Tip>

## Convenzione di importazione

Importa sempre da un sottopercorso specifico:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
```

Ogni sottopercorso è un modulo piccolo e autosufficiente. Questo mantiene
l'avvio veloce e previene problemi di dipendenze circolari. Per gli helper di
entry/build specifici dei canali, preferisci `openclaw/plugin-sdk/channel-core`;
mantieni `openclaw/plugin-sdk/core` per la superficie ombrello più ampia e gli
helper condivisi come `buildChannelConfigSchema`.

Non aggiungere né dipendere da seam di convenienza con nomi di provider come
`openclaw/plugin-sdk/slack`, `openclaw/plugin-sdk/discord`,
`openclaw/plugin-sdk/signal`, `openclaw/plugin-sdk/whatsapp`, o
seam di helper brandizzati per canale. I plugin inclusi dovrebbero comporre i
sottopercorsi generici dell'SDK all'interno dei propri barrel `api.ts` o
`runtime-api.ts`, e il core dovrebbe usare quei barrel locali al plugin oppure
aggiungere un contratto SDK generico ristretto quando l'esigenza è davvero
trasversale ai canali.

La mappa di esportazione generata contiene ancora un piccolo insieme di seam di
helper per plugin inclusi come `plugin-sdk/feishu`, `plugin-sdk/feishu-setup`,
`plugin-sdk/zalo`, `plugin-sdk/zalo-setup` e `plugin-sdk/matrix*`. Quei
sottopercorsi esistono solo per la manutenzione e la compatibilità dei plugin
inclusi; sono intenzionalmente omessi dalla tabella comune qui sotto e non sono
il percorso di importazione consigliato per nuovi plugin di terze parti.

## Riferimento dei sottopercorsi

I sottopercorsi usati più comunemente, raggruppati per scopo. L'elenco completo
generato di oltre 200 sottopercorsi si trova in `scripts/lib/plugin-sdk-entrypoints.json`.

I sottopercorsi helper riservati per plugin inclusi compaiono ancora in
quell'elenco generato. Trattali come superfici di dettaglio implementativo o di
compatibilità, a meno che una pagina della documentazione non ne promuova
esplicitamente uno come pubblico.

### Entry del plugin

| Sottopercorso               | Esportazioni chiave                                                                                                                  |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `plugin-sdk/plugin-entry`   | `definePluginEntry`                                                                                                                  |
| `plugin-sdk/core`           | `defineChannelPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase`, `defineSetupPluginEntry`, `buildChannelConfigSchema` |
| `plugin-sdk/config-schema`  | `OpenClawSchema`                                                                                                                     |
| `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry`                                                                                                    |

<AccordionGroup>
  <Accordion title="Sottopercorsi dei canali">
    | Sottopercorso | Esportazioni chiave |
    | --- | --- |
    | `plugin-sdk/channel-core` | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
    | `plugin-sdk/config-schema` | Esportazione dello schema Zod radice di `openclaw.json` (`OpenClawSchema`) |
    | `plugin-sdk/channel-setup` | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, più `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
    | `plugin-sdk/setup` | Helper condivisi per il wizard di setup, prompt allowlist, builder dello stato di setup |
    | `plugin-sdk/setup-runtime` | `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
    | `plugin-sdk/setup-adapter-runtime` | `createEnvPatchedAccountSetupAdapter` |
    | `plugin-sdk/setup-tools` | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
    | `plugin-sdk/account-core` | Helper per config/multi-account/action-gate, helper di fallback per account predefinito |
    | `plugin-sdk/account-id` | `DEFAULT_ACCOUNT_ID`, helper di normalizzazione degli ID account |
    | `plugin-sdk/account-resolution` | Helper per lookup degli account + fallback predefinito |
    | `plugin-sdk/account-helpers` | Helper ristretti per elenco account/azioni account |
    | `plugin-sdk/channel-pairing` | `createChannelPairingController` |
    | `plugin-sdk/channel-reply-pipeline` | `createChannelReplyPipeline` |
    | `plugin-sdk/channel-config-helpers` | `createHybridChannelConfigAdapter` |
    | `plugin-sdk/channel-config-schema` | Tipi di schema della configurazione del canale |
    | `plugin-sdk/telegram-command-config` | Helper di normalizzazione/validazione dei comandi personalizzati di Telegram con fallback del contratto incluso |
    | `plugin-sdk/channel-policy` | `resolveChannelGroupRequireMention` |
    | `plugin-sdk/channel-lifecycle` | `createAccountStatusSink` |
    | `plugin-sdk/inbound-envelope` | Helper condivisi per route inbound + builder di envelope |
    | `plugin-sdk/inbound-reply-dispatch` | Helper condivisi per registrazione e dispatch inbound |
    | `plugin-sdk/messaging-targets` | Helper di parsing/corrispondenza dei target |
    | `plugin-sdk/outbound-media` | Helper condivisi per il caricamento dei media outbound |
    | `plugin-sdk/outbound-runtime` | Helper per identità outbound/delega di invio |
    | `plugin-sdk/thread-bindings-runtime` | Ciclo di vita thread-binding e helper di adapter |
    | `plugin-sdk/agent-media-payload` | Builder legacy del payload media dell'agente |
    | `plugin-sdk/conversation-runtime` | Helper per binding conversazione/thread, pairing e binding configurato |
    | `plugin-sdk/runtime-config-snapshot` | Helper per snapshot della configurazione runtime |
    | `plugin-sdk/runtime-group-policy` | Helper per la risoluzione della group-policy runtime |
    | `plugin-sdk/channel-status` | Helper condivisi per snapshot/riepilogo dello stato del canale |
    | `plugin-sdk/channel-config-primitives` | Primitive ristrette dello schema di configurazione del canale |
    | `plugin-sdk/channel-config-writes` | Helper di autorizzazione per scrittura della configurazione del canale |
    | `plugin-sdk/channel-plugin-common` | Esportazioni di preambolo condivise per plugin di canale |
    | `plugin-sdk/allowlist-config-edit` | Helper di modifica/lettura della configurazione allowlist |
    | `plugin-sdk/group-access` | Helper condivisi per decisioni di accesso ai gruppi |
    | `plugin-sdk/direct-dm` | Helper condivisi per auth/guard delle DM dirette |
    | `plugin-sdk/interactive-runtime` | Helper di normalizzazione/riduzione del payload di risposta interattiva |
    | `plugin-sdk/channel-inbound` | Helper per debounce inbound, corrispondenza mention, mention-policy ed envelope |
    | `plugin-sdk/channel-send-result` | Tipi di risultato della risposta |
    | `plugin-sdk/channel-actions` | `createMessageToolButtonsSchema`, `createMessageToolCardSchema` |
    | `plugin-sdk/channel-targets` | Helper di parsing/corrispondenza dei target |
    | `plugin-sdk/channel-contract` | Tipi del contratto del canale |
    | `plugin-sdk/channel-feedback` | Collegamento di feedback/reazioni |
    | `plugin-sdk/channel-secret-runtime` | Helper ristretti del contratto dei secret come `collectSimpleChannelFieldAssignments`, `getChannelSurface`, `pushAssignment` e tipi target dei secret |
  </Accordion>

  <Accordion title="Sottopercorsi dei provider">
    | Sottopercorso | Esportazioni chiave |
    | --- | --- |
    | `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry` |
    | `plugin-sdk/provider-setup` | Helper selezionati di setup per provider locali/self-hosted |
    | `plugin-sdk/self-hosted-provider-setup` | Helper mirati di setup per provider self-hosted compatibili con OpenAI |
    | `plugin-sdk/cli-backend` | Valori predefiniti del backend CLI + costanti watchdog |
    | `plugin-sdk/provider-auth-runtime` | Helper runtime per la risoluzione delle API key per plugin provider |
    | `plugin-sdk/provider-auth-api-key` | Helper per onboarding/scrittura profili API key come `upsertApiKeyProfile` |
    | `plugin-sdk/provider-auth-result` | Builder standard del risultato auth OAuth |
    | `plugin-sdk/provider-auth-login` | Helper condivisi di login interattivo per plugin provider |
    | `plugin-sdk/provider-env-vars` | Helper di lookup delle variabili d'ambiente di auth del provider |
    | `plugin-sdk/provider-auth` | `createProviderApiKeyAuthMethod`, `ensureApiKeyFromOptionEnvOrPrompt`, `upsertAuthProfile`, `upsertApiKeyProfile`, `writeOAuthCredentials` |
    | `plugin-sdk/provider-model-shared` | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, builder condivisi di replay-policy, helper per endpoint del provider e helper di normalizzazione degli ID modello come `normalizeNativeXaiModelId` |
    | `plugin-sdk/provider-catalog-shared` | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
    | `plugin-sdk/provider-http` | Helper generici per capacità HTTP/endpoint del provider |
    | `plugin-sdk/provider-web-fetch-contract` | Helper ristretti del contratto di configurazione/selezione web-fetch come `enablePluginInConfig` e `WebFetchProviderPlugin` |
    | `plugin-sdk/provider-web-fetch` | Helper di registrazione/cache del provider web-fetch |
    | `plugin-sdk/provider-web-search-config-contract` | Helper ristretti di configurazione/credenziali web-search per provider che non richiedono il wiring di abilitazione del plugin |
    | `plugin-sdk/provider-web-search-contract` | Helper ristretti del contratto di configurazione/credenziali web-search come `createWebSearchProviderContractFields`, `enablePluginInConfig`, `resolveProviderWebSearchPluginConfig` e setter/getter di credenziali con ambito |
    | `plugin-sdk/provider-web-search` | Helper di registrazione/cache/runtime del provider web-search |
    | `plugin-sdk/provider-tools` | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, cleanup + diagnostica dello schema Gemini e helper di compatibilità xAI come `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
    | `plugin-sdk/provider-usage` | `fetchClaudeUsage` e simili |
    | `plugin-sdk/provider-stream` | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, tipi di wrapper stream e helper wrapper condivisi per Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot |
    | `plugin-sdk/provider-onboard` | Helper di patch della configurazione di onboarding |
    | `plugin-sdk/global-singleton` | Helper per singleton/map/cache locali al processo |
  </Accordion>

  <Accordion title="Sottopercorsi di auth e sicurezza">
    | Sottopercorso | Esportazioni chiave |
    | --- | --- |
    | `plugin-sdk/command-auth` | `resolveControlCommandGate`, helper del registry dei comandi, helper di autorizzazione del mittente |
    | `plugin-sdk/approval-auth-runtime` | Helper per la risoluzione degli approvatori e action-auth nella stessa chat |
    | `plugin-sdk/approval-client-runtime` | Helper di profilo/filtro per approvazioni native exec |
    | `plugin-sdk/approval-delivery-runtime` | Adapter condivisi per capacità/consegna delle approvazioni native |
    | `plugin-sdk/approval-gateway-runtime` | Helper condiviso per la risoluzione del gateway delle approvazioni |
    | `plugin-sdk/approval-handler-adapter-runtime` | Helper leggeri di caricamento dell'adapter di approvazione nativa per entrypoint di canale ad alta frequenza |
    | `plugin-sdk/approval-handler-runtime` | Helper runtime più ampi per il gestore di approvazioni; preferisci i seam più ristretti adapter/gateway quando sono sufficienti |
    | `plugin-sdk/approval-native-runtime` | Helper per target di approvazione nativa + binding account |
    | `plugin-sdk/approval-reply-runtime` | Helper per il payload di risposta delle approvazioni exec/plugin |
    | `plugin-sdk/command-auth-native` | Auth dei comandi nativi + helper per target di sessione nativa |
    | `plugin-sdk/command-detection` | Helper condivisi di rilevamento dei comandi |
    | `plugin-sdk/command-surface` | Helper per normalizzazione del body del comando e command-surface |
    | `plugin-sdk/allow-from` | `formatAllowFromLowercase` |
    | `plugin-sdk/channel-secret-runtime` | Helper ristretti di raccolta del contratto dei secret per superfici di secret di canale/plugin |
    | `plugin-sdk/secret-ref-runtime` | Helper ristretti `coerceSecretRef` e tipizzazione SecretRef per parsing del contratto dei secret/configurazione |
    | `plugin-sdk/security-runtime` | Helper condivisi per trust, gating DM, contenuti esterni e raccolta dei secret |
    | `plugin-sdk/ssrf-policy` | Helper di allowlist host e policy SSRF per reti private |
    | `plugin-sdk/ssrf-runtime` | Helper per dispatcher bloccato, fetch protetto da SSRF e policy SSRF |
    | `plugin-sdk/secret-input` | Helper di parsing per input secret |
    | `plugin-sdk/webhook-ingress` | Helper per richieste/target webhook |
    | `plugin-sdk/webhook-request-guards` | Helper per dimensione del body/timeout delle richieste |
  </Accordion>

  <Accordion title="Sottopercorsi runtime e storage">
    | Sottopercorso | Esportazioni chiave |
    | --- | --- |
    | `plugin-sdk/runtime` | Helper ampi per runtime/logging/backup/installazione plugin |
    | `plugin-sdk/runtime-env` | Helper ristretti per env runtime, logger, timeout, retry e backoff |
    | `plugin-sdk/channel-runtime-context` | Helper generici di registrazione e lookup del contesto runtime del canale |
    | `plugin-sdk/runtime-store` | `createPluginRuntimeStore` |
    | `plugin-sdk/plugin-runtime` | Helper condivisi per comandi/hook/http/interattivi dei plugin |
    | `plugin-sdk/hook-runtime` | Helper condivisi per pipeline di webhook/internal hook |
    | `plugin-sdk/lazy-runtime` | Helper per import/binding runtime lazy come `createLazyRuntimeModule`, `createLazyRuntimeMethod` e `createLazyRuntimeSurface` |
    | `plugin-sdk/process-runtime` | Helper per esecuzione di processi |
    | `plugin-sdk/cli-runtime` | Helper per formattazione CLI, attesa e versione |
    | `plugin-sdk/gateway-runtime` | Helper per client gateway e patch dello stato del canale |
    | `plugin-sdk/config-runtime` | Helper di caricamento/scrittura della configurazione |
    | `plugin-sdk/telegram-command-config` | Normalizzazione di nome/descrizione dei comandi Telegram e controlli di duplicati/conflitti, anche quando la superficie del contratto Telegram incluso non è disponibile |
    | `plugin-sdk/approval-runtime` | Helper per approvazioni exec/plugin, builder di capacità di approvazione, helper auth/profili, helper di routing/runtime nativi |
    | `plugin-sdk/reply-runtime` | Helper condivisi per runtime inbound/risposte, chunking, dispatch, heartbeat, pianificatore delle risposte |
    | `plugin-sdk/reply-dispatch-runtime` | Helper ristretti per dispatch/finalizzazione delle risposte |
    | `plugin-sdk/reply-history` | Helper condivisi per la cronologia delle risposte a finestra breve come `buildHistoryContext`, `recordPendingHistoryEntry` e `clearHistoryEntriesIfEnabled` |
    | `plugin-sdk/reply-reference` | `createReplyReferencePlanner` |
    | `plugin-sdk/reply-chunking` | Helper ristretti per chunking di testo/markdown |
    | `plugin-sdk/session-store-runtime` | Helper per path dello store di sessione + updated-at |
    | `plugin-sdk/state-paths` | Helper di path per directory state/OAuth |
    | `plugin-sdk/routing` | Helper per route/session-key/binding account come `resolveAgentRoute`, `buildAgentSessionKey` e `resolveDefaultAgentBoundAccountId` |
    | `plugin-sdk/status-helpers` | Helper condivisi per il riepilogo dello stato di canale/account, valori predefiniti dello stato runtime e helper per metadati dei problemi |
    | `plugin-sdk/target-resolver-runtime` | Helper condivisi per il resolver dei target |
    | `plugin-sdk/string-normalization-runtime` | Helper di normalizzazione slug/stringhe |
    | `plugin-sdk/request-url` | Estrai URL stringa da input simili a fetch/request |
    | `plugin-sdk/run-command` | Esecutore di comandi temporizzato con risultati stdout/stderr normalizzati |
    | `plugin-sdk/param-readers` | Lettori comuni di parametri tool/CLI |
    | `plugin-sdk/tool-payload` | Estrai payload normalizzati da oggetti risultato dei tool |
    | `plugin-sdk/tool-send` | Estrai i campi target di invio canonici dagli argomenti del tool |
    | `plugin-sdk/temp-path` | Helper condivisi per path di download temporanei |
    | `plugin-sdk/logging-core` | Helper per logger di sottosistema e redazione |
    | `plugin-sdk/markdown-table-runtime` | Helper per modalità tabelle Markdown |
    | `plugin-sdk/json-store` | Piccoli helper di lettura/scrittura dello stato JSON |
    | `plugin-sdk/file-lock` | Helper di file-lock rientrante |
    | `plugin-sdk/persistent-dedupe` | Helper per cache di deduplica persistente su disco |
    | `plugin-sdk/acp-runtime` | Helper ACP per runtime/sessione e reply-dispatch |
    | `plugin-sdk/agent-config-primitives` | Primitive ristrette dello schema di configurazione runtime dell'agente |
    | `plugin-sdk/boolean-param` | Lettore permissivo di parametri booleani |
    | `plugin-sdk/dangerous-name-runtime` | Helper di risoluzione della corrispondenza di nomi pericolosi |
    | `plugin-sdk/device-bootstrap` | Helper per bootstrap del dispositivo e token di pairing |
    | `plugin-sdk/extension-shared` | Primitive helper condivise per canale passivo, stato e proxy ambientale |
    | `plugin-sdk/models-provider-runtime` | Helper per il comando `/models` e risposte del provider |
    | `plugin-sdk/skill-commands-runtime` | Helper di elenco dei comandi Skills |
    | `plugin-sdk/native-command-registry` | Helper per registry/build/serializzazione dei comandi nativi |
    | `plugin-sdk/provider-zai-endpoint` | Helper di rilevamento endpoint Z.A.I |
    | `plugin-sdk/infra-runtime` | Helper per eventi di sistema/heartbeat |
    | `plugin-sdk/collection-runtime` | Piccoli helper per cache limitate |
    | `plugin-sdk/diagnostic-runtime` | Helper per flag ed eventi diagnostici |
    | `plugin-sdk/error-runtime` | Grafo degli errori, formattazione, helper condivisi per classificazione degli errori, `isApprovalNotFoundError` |
    | `plugin-sdk/fetch-runtime` | Helper per fetch incapsulato, proxy e lookup bloccato |
    | `plugin-sdk/host-runtime` | Helper di normalizzazione per hostname e host SCP |
    | `plugin-sdk/retry-runtime` | Helper per configurazione dei retry ed esecuzione con retry |
    | `plugin-sdk/agent-runtime` | Helper per directory/identità/workspace dell'agente |
    | `plugin-sdk/directory-runtime` | Query/dedup di directory basate sulla configurazione |
    | `plugin-sdk/keyed-async-queue` | `KeyedAsyncQueue` |
  </Accordion>

  <Accordion title="Sottopercorsi di capability e testing">
    | Sottopercorso | Esportazioni chiave |
    | --- | --- |
    | `plugin-sdk/media-runtime` | Helper condivisi per fetch/trasformazione/storage dei media più builder di payload media |
    | `plugin-sdk/media-generation-runtime` | Helper condivisi per failover della generazione media, selezione dei candidati e messaggistica per modello mancante |
    | `plugin-sdk/media-understanding` | Tipi di provider per comprensione dei media più esportazioni di helper image/audio rivolte ai provider |
    | `plugin-sdk/text-runtime` | Helper condivisi per testo/markdown/logging come rimozione del testo visibile all'assistente, helper per rendering/chunking/tabelle Markdown, helper di redazione, helper per directive-tag e utility per testo sicuro |
    | `plugin-sdk/text-chunking` | Helper per chunking del testo outbound |
    | `plugin-sdk/speech` | Tipi di provider speech più helper rivolti ai provider per directive, registry e validazione |
    | `plugin-sdk/speech-core` | Tipi condivisi di provider speech, registry, directive e helper di normalizzazione |
    | `plugin-sdk/realtime-transcription` | Tipi di provider di trascrizione realtime e helper del registry |
    | `plugin-sdk/realtime-voice` | Tipi di provider voce realtime e helper del registry |
    | `plugin-sdk/image-generation` | Tipi di provider per generazione di immagini |
    | `plugin-sdk/image-generation-core` | Tipi condivisi di generazione immagini, failover, auth e helper del registry |
    | `plugin-sdk/music-generation` | Tipi di provider/richieste/risultati per generazione musicale |
    | `plugin-sdk/music-generation-core` | Tipi condivisi di generazione musicale, helper di failover, lookup provider e parsing model-ref |
    | `plugin-sdk/video-generation` | Tipi di provider/richieste/risultati per generazione video |
    | `plugin-sdk/video-generation-core` | Tipi condivisi di generazione video, helper di failover, lookup provider e parsing model-ref |
    | `plugin-sdk/webhook-targets` | Registry dei target webhook e helper per installazione delle route |
    | `plugin-sdk/webhook-path` | Helper di normalizzazione dei path webhook |
    | `plugin-sdk/web-media` | Helper condivisi per caricamento di media remoti/locali |
    | `plugin-sdk/zod` | `zod` riesportato per i consumatori del plugin SDK |
    | `plugin-sdk/testing` | `installCommonResolveTargetErrorCases`, `shouldAckReaction` |
  </Accordion>

  <Accordion title="Sottopercorsi memory">
    | Sottopercorso | Esportazioni chiave |
    | --- | --- |
    | `plugin-sdk/memory-core` | Superficie helper memory-core inclusa per helper manager/config/file/CLI |
    | `plugin-sdk/memory-core-engine-runtime` | Facade runtime dell'indice/ricerca memory |
    | `plugin-sdk/memory-core-host-engine-foundation` | Esportazioni del motore foundation dell'host memory |
    | `plugin-sdk/memory-core-host-engine-embeddings` | Esportazioni del motore embeddings dell'host memory |
    | `plugin-sdk/memory-core-host-engine-qmd` | Esportazioni del motore QMD dell'host memory |
    | `plugin-sdk/memory-core-host-engine-storage` | Esportazioni del motore storage dell'host memory |
    | `plugin-sdk/memory-core-host-multimodal` | Helper multimodali dell'host memory |
    | `plugin-sdk/memory-core-host-query` | Helper di query dell'host memory |
    | `plugin-sdk/memory-core-host-secret` | Helper dei secret dell'host memory |
    | `plugin-sdk/memory-core-host-events` | Helper del journal eventi dell'host memory |
    | `plugin-sdk/memory-core-host-status` | Helper di stato dell'host memory |
    | `plugin-sdk/memory-core-host-runtime-cli` | Helper runtime CLI dell'host memory |
    | `plugin-sdk/memory-core-host-runtime-core` | Helper runtime core dell'host memory |
    | `plugin-sdk/memory-core-host-runtime-files` | Helper file/runtime dell'host memory |
    | `plugin-sdk/memory-host-core` | Alias vendor-neutral per helper runtime core dell'host memory |
    | `plugin-sdk/memory-host-events` | Alias vendor-neutral per helper del journal eventi dell'host memory |
    | `plugin-sdk/memory-host-files` | Alias vendor-neutral per helper file/runtime dell'host memory |
    | `plugin-sdk/memory-host-markdown` | Helper condivisi per markdown gestito per plugin adiacenti a memory |
    | `plugin-sdk/memory-host-search` | Facade runtime memory attiva per accesso al search-manager |
    | `plugin-sdk/memory-host-status` | Alias vendor-neutral per helper di stato dell'host memory |
    | `plugin-sdk/memory-lancedb` | Superficie helper memory-lancedb inclusa |
  </Accordion>

  <Accordion title="Sottopercorsi helper inclusi riservati">
    | Famiglia | Sottopercorsi correnti | Uso previsto |
    | --- | --- | --- |
    | Browser | `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support` | Helper di supporto per il plugin browser incluso (`browser-support` resta il barrel di compatibilità) |
    | Matrix | `plugin-sdk/matrix`, `plugin-sdk/matrix-helper`, `plugin-sdk/matrix-runtime-heavy`, `plugin-sdk/matrix-runtime-shared`, `plugin-sdk/matrix-runtime-surface`, `plugin-sdk/matrix-surface`, `plugin-sdk/matrix-thread-bindings` | Superficie helper/runtime Matrix inclusa |
    | Line | `plugin-sdk/line`, `plugin-sdk/line-core`, `plugin-sdk/line-runtime`, `plugin-sdk/line-surface` | Superficie helper/runtime LINE inclusa |
    | IRC | `plugin-sdk/irc`, `plugin-sdk/irc-surface` | Superficie helper IRC inclusa |
    | Helper specifici del canale | `plugin-sdk/googlechat`, `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles`, `plugin-sdk/bluebubbles-policy`, `plugin-sdk/mattermost`, `plugin-sdk/mattermost-policy`, `plugin-sdk/feishu-conversation`, `plugin-sdk/msteams`, `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`, `plugin-sdk/twitch` | Seam di compatibilità/helper di canale inclusi |
    | Helper specifici di auth/plugin | `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`, `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`, `plugin-sdk/thread-ownership`, `plugin-sdk/voice-call` | Seam helper di funzionalità/plugin inclusi; `plugin-sdk/github-copilot-token` esporta attualmente `DEFAULT_COPILOT_API_BASE_URL`, `deriveCopilotApiBaseUrlFromToken` e `resolveCopilotApiToken` |
  </Accordion>
</AccordionGroup>

## API di registrazione

La callback `register(api)` riceve un oggetto `OpenClawPluginApi` con questi
metodi:

### Registrazione delle capability

| Metodo                                           | Cosa registra                  |
| ------------------------------------------------ | ------------------------------ |
| `api.registerProvider(...)`                      | Inferenza di testo (LLM)       |
| `api.registerCliBackend(...)`                    | Backend di inferenza CLI locale |
| `api.registerChannel(...)`                       | Canale di messaggistica        |
| `api.registerSpeechProvider(...)`                | Sintesi text-to-speech / STT   |
| `api.registerRealtimeTranscriptionProvider(...)` | Trascrizione realtime in streaming |
| `api.registerRealtimeVoiceProvider(...)`         | Sessioni vocali realtime duplex |
| `api.registerMediaUnderstandingProvider(...)`    | Analisi di immagini/audio/video |
| `api.registerImageGenerationProvider(...)`       | Generazione di immagini        |
| `api.registerMusicGenerationProvider(...)`       | Generazione musicale           |
| `api.registerVideoGenerationProvider(...)`       | Generazione video              |
| `api.registerWebFetchProvider(...)`              | Provider di web fetch / scraping |
| `api.registerWebSearchProvider(...)`             | Ricerca web                    |

### Tool e comandi

| Metodo                          | Cosa registra                                |
| ------------------------------- | -------------------------------------------- |
| `api.registerTool(tool, opts?)` | Tool dell'agente (obbligatorio o `{ optional: true }`) |
| `api.registerCommand(def)`      | Comando personalizzato (bypassa l'LLM)       |

### Infrastruttura

| Metodo                                         | Cosa registra                         |
| ---------------------------------------------- | ------------------------------------- |
| `api.registerHook(events, handler, opts?)`     | Hook di evento                        |
| `api.registerHttpRoute(params)`                | Endpoint HTTP del gateway             |
| `api.registerGatewayMethod(name, handler)`     | Metodo RPC del gateway                |
| `api.registerCli(registrar, opts?)`            | Sottocomando CLI                      |
| `api.registerService(service)`                 | Servizio in background                |
| `api.registerInteractiveHandler(registration)` | Gestore interattivo                   |
| `api.registerMemoryPromptSupplement(builder)`  | Sezione di prompt additiva adiacente a memory |
| `api.registerMemoryCorpusSupplement(adapter)`  | Corpus additivo per ricerca/lettura memory |

I namespace amministrativi core riservati (`config.*`, `exec.approvals.*`, `wizard.*`,
`update.*`) restano sempre `operator.admin`, anche se un plugin prova ad assegnare
uno scope più ristretto al metodo del gateway. Preferisci prefissi specifici del
plugin per i metodi di proprietà del plugin.

### Metadati di registrazione CLI

`api.registerCli(registrar, opts?)` accetta due tipi di metadati di primo livello:

- `commands`: root di comando espliciti posseduti dal registrar
- `descriptors`: descrittori di comando in fase di parsing usati per l'help della
  root CLI, il routing e la registrazione lazy della CLI del plugin

Se vuoi che un comando del plugin resti lazy-loaded nel normale percorso della
root CLI, fornisci `descriptors` che coprano ogni root di comando di primo
livello esposta da quel registrar.

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
        description: "Gestisci account Matrix, verifica, dispositivi e stato del profilo",
        hasSubcommands: true,
      },
    ],
  },
);
```

Usa `commands` da solo solo quando non hai bisogno della registrazione lazy
della root CLI. Quel percorso di compatibilità eager resta supportato, ma non
installa segnaposto basati su descriptor per il lazy loading in fase di parsing.

### Registrazione del backend CLI

`api.registerCliBackend(...)` consente a un plugin di possedere la
configurazione predefinita per un backend CLI AI locale come `codex-cli`.

- L'`id` del backend diventa il prefisso del provider nei model ref come `codex-cli/gpt-5`.
- La `config` del backend usa la stessa forma di `agents.defaults.cliBackends.<id>`.
- La configurazione utente ha comunque la precedenza. OpenClaw unisce `agents.defaults.cliBackends.<id>` sopra il valore predefinito del
  plugin prima di eseguire la CLI.
- Usa `normalizeConfig` quando un backend richiede riscritture di compatibilità dopo l'unione
  (ad esempio normalizzare vecchie forme di flag).

### Slot esclusivi

| Metodo                                     | Cosa registra                                                                                                                                              |
| ------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `api.registerContextEngine(id, factory)`   | Context engine (uno attivo alla volta). La callback `assemble()` riceve `availableTools` e `citationsMode` così il motore può adattare le aggiunte al prompt. |
| `api.registerMemoryCapability(capability)` | Capability memory unificata                                                                                                                                 |
| `api.registerMemoryPromptSection(builder)` | Builder della sezione prompt memory                                                                                                                        |
| `api.registerMemoryFlushPlan(resolver)`    | Resolver del piano di flush memory                                                                                                                         |
| `api.registerMemoryRuntime(runtime)`       | Adapter runtime memory                                                                                                                                     |

### Adapter di embedding memory

| Metodo                                         | Cosa registra                                  |
| ---------------------------------------------- | ---------------------------------------------- |
| `api.registerMemoryEmbeddingProvider(adapter)` | Adapter di embedding memory per il plugin attivo |

- `registerMemoryCapability` è l'API preferita per plugin memory esclusivi.
- `registerMemoryCapability` può anche esporre `publicArtifacts.listArtifacts(...)`
  così i plugin companion possono consumare gli artifact memory esportati tramite
  `openclaw/plugin-sdk/memory-host-core` invece di entrare nel layout privato di
  uno specifico plugin memory.
- `registerMemoryPromptSection`, `registerMemoryFlushPlan` e
  `registerMemoryRuntime` sono API legacy-compatibili per plugin memory esclusivi.
- `registerMemoryEmbeddingProvider` consente al plugin memory attivo di registrare
  uno o più ID di adapter di embedding (ad esempio `openai`, `gemini` o un ID
  personalizzato definito dal plugin).
- La configurazione utente come `agents.defaults.memorySearch.provider` e
  `agents.defaults.memorySearch.fallback` viene risolta rispetto a quegli ID di
  adapter registrati.

### Eventi e ciclo di vita

| Metodo                                       | Cosa fa                     |
| -------------------------------------------- | --------------------------- |
| `api.on(hookName, handler, opts?)`           | Hook di ciclo di vita tipizzato |
| `api.onConversationBindingResolved(handler)` | Callback di binding della conversazione |

### Semantica delle decisioni degli hook

- `before_tool_call`: restituire `{ block: true }` è terminale. Una volta che un gestore lo imposta, i gestori a priorità più bassa vengono saltati.
- `before_tool_call`: restituire `{ block: false }` è trattato come nessuna decisione (come omettere `block`), non come override.
- `before_install`: restituire `{ block: true }` è terminale. Una volta che un gestore lo imposta, i gestori a priorità più bassa vengono saltati.
- `before_install`: restituire `{ block: false }` è trattato come nessuna decisione (come omettere `block`), non come override.
- `reply_dispatch`: restituire `{ handled: true, ... }` è terminale. Una volta che un gestore rivendica il dispatch, i gestori a priorità più bassa e il percorso predefinito di dispatch del modello vengono saltati.
- `message_sending`: restituire `{ cancel: true }` è terminale. Una volta che un gestore lo imposta, i gestori a priorità più bassa vengono saltati.
- `message_sending`: restituire `{ cancel: false }` è trattato come nessuna decisione (come omettere `cancel`), non come override.

### Campi dell'oggetto API

| Campo                    | Tipo                      | Descrizione                                                                                   |
| ------------------------ | ------------------------- | --------------------------------------------------------------------------------------------- |
| `api.id`                 | `string`                  | ID del plugin                                                                                 |
| `api.name`               | `string`                  | Nome visualizzato                                                                             |
| `api.version`            | `string?`                 | Versione del plugin (facoltativa)                                                             |
| `api.description`        | `string?`                 | Descrizione del plugin (facoltativa)                                                          |
| `api.source`             | `string`                  | Percorso sorgente del plugin                                                                  |
| `api.rootDir`            | `string?`                 | Directory root del plugin (facoltativa)                                                       |
| `api.config`             | `OpenClawConfig`          | Snapshot della configurazione corrente (snapshot runtime in memoria attivo quando disponibile) |
| `api.pluginConfig`       | `Record<string, unknown>` | Configurazione specifica del plugin da `plugins.entries.<id>.config`                          |
| `api.runtime`            | `PluginRuntime`           | [Helper runtime](/it/plugins/sdk-runtime)                                                        |
| `api.logger`             | `PluginLogger`            | Logger con scope (`debug`, `info`, `warn`, `error`)                                           |
| `api.registrationMode`   | `PluginRegistrationMode`  | Modalità di caricamento corrente; `"setup-runtime"` è la finestra leggera di avvio/setup pre-entry completa |
| `api.resolvePath(input)` | `(string) => string`      | Risolve il percorso relativo alla root del plugin                                             |

## Convenzione per i moduli interni

All'interno del tuo plugin, usa file barrel locali per gli import interni:

```
my-plugin/
  api.ts            # Esportazioni pubbliche per consumatori esterni
  runtime-api.ts    # Esportazioni runtime solo interne
  index.ts          # Punto di ingresso del plugin
  setup-entry.ts    # Entry leggera solo per setup (facoltativa)
```

<Warning>
  Non importare mai il tuo stesso plugin tramite `openclaw/plugin-sdk/<your-plugin>`
  dal codice di produzione. Instrada gli import interni tramite `./api.ts` o
  `./runtime-api.ts`. Il percorso SDK è solo il contratto esterno.
</Warning>

Le superfici pubbliche dei plugin inclusi caricate tramite facade (`api.ts`, `runtime-api.ts`,
`index.ts`, `setup-entry.ts` e file entry pubblici simili) ora preferiscono lo
snapshot della configurazione runtime attiva quando OpenClaw è già in esecuzione. Se non
esiste ancora uno snapshot runtime, usano come fallback il file di configurazione
risolto su disco.

I plugin provider possono anche esporre un barrel di contratto locale al plugin e
ristretto quando un helper è intenzionalmente specifico del provider e non appartiene
ancora a un sottopercorso generico dell'SDK. Esempio incluso attuale: il provider
Anthropic mantiene i propri helper di stream Claude nella propria seam pubblica
`api.ts` / `contract-api.ts` invece di promuovere la logica di header beta Anthropic
e `service_tier` in un contratto generico `plugin-sdk/*`.

Altri esempi inclusi attuali:

- `@openclaw/openai-provider`: `api.ts` esporta builder del provider,
  helper per modelli predefiniti e builder di provider realtime
- `@openclaw/openrouter-provider`: `api.ts` esporta il builder del provider più
  helper di onboarding/configurazione

<Warning>
  Il codice di produzione delle estensioni dovrebbe anche evitare gli import
  `openclaw/plugin-sdk/<other-plugin>`. Se un helper è davvero condiviso,
  promuovilo a un sottopercorso SDK neutrale come `openclaw/plugin-sdk/speech`,
  `.../provider-model-shared` o un'altra superficie orientata alle capability
  invece di accoppiare due plugin tra loro.
</Warning>

## Correlati

- [Entry Points](/it/plugins/sdk-entrypoints) — opzioni di `definePluginEntry` e `defineChannelPluginEntry`
- [Runtime Helpers](/it/plugins/sdk-runtime) — riferimento completo del namespace `api.runtime`
- [Setup and Config](/it/plugins/sdk-setup) — packaging, manifest e schemi di configurazione
- [Testing](/it/plugins/sdk-testing) — utility di test e regole lint
- [SDK Migration](/it/plugins/sdk-migration) — migrazione da superfici deprecate
- [Plugin Internals](/it/plugins/architecture) — architettura approfondita e modello delle capability
