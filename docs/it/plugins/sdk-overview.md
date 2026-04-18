---
read_when:
    - Devi sapere da quale sottopercorso dell'SDK importare
    - Vuoi un riferimento per tutti i metodi di registrazione su OpenClawPluginApi
    - Stai cercando una specifica esportazione dell'SDK
sidebarTitle: SDK Overview
summary: Mappa di importazione, riferimento dell'API di registrazione e architettura dell'SDK
title: Panoramica dell'SDK del Plugin
x-i18n:
    generated_at: "2026-04-18T08:05:09Z"
    model: gpt-5.4
    provider: openai
    source_hash: 05d3d0022cca32d29c76f6cea01cdf4f88ac69ef0ef3d7fb8a60fbf9a6b9b331
    source_path: plugins/sdk-overview.md
    workflow: 15
---

# Panoramica dell'SDK del Plugin

L'SDK del plugin è il contratto tipizzato tra i plugin e il core. Questa pagina è il
riferimento per **cosa importare** e **cosa è possibile registrare**.

<Tip>
  **Cerchi una guida pratica?**
  - Primo plugin? Inizia con [Guida introduttiva](/it/plugins/building-plugins)
  - Plugin di canale? Vedi [Plugin di canale](/it/plugins/sdk-channel-plugins)
  - Plugin provider? Vedi [Plugin provider](/it/plugins/sdk-provider-plugins)
</Tip>

## Convenzione di importazione

Importa sempre da un sottopercorso specifico:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
```

Ogni sottopercorso è un modulo piccolo e autosufficiente. Questo mantiene l'avvio rapido e
previene i problemi di dipendenze circolari. Per gli helper di entry/build specifici per canale,
preferisci `openclaw/plugin-sdk/channel-core`; mantieni `openclaw/plugin-sdk/core` per
la superficie ombrello più ampia e gli helper condivisi come
`buildChannelConfigSchema`.

Non aggiungere né dipendere da seam di convenienza con nomi di provider come
`openclaw/plugin-sdk/slack`, `openclaw/plugin-sdk/discord`,
`openclaw/plugin-sdk/signal`, `openclaw/plugin-sdk/whatsapp`, o
seam di helper con branding del canale. I plugin inclusi dovrebbero comporre sottopercorsi
generici dell'SDK all'interno dei propri barrel `api.ts` o `runtime-api.ts`, e il core
dovrebbe usare quei barrel locali al plugin oppure aggiungere un contratto SDK generico e ristretto
quando la necessità è davvero trasversale ai canali.

La mappa di esportazione generata contiene ancora un piccolo insieme di seam helper per plugin inclusi
come `plugin-sdk/feishu`, `plugin-sdk/feishu-setup`,
`plugin-sdk/zalo`, `plugin-sdk/zalo-setup` e `plugin-sdk/matrix*`. Quei
sottopercorsi esistono solo per la manutenzione e la compatibilità dei plugin inclusi; sono
intenzionalmente omessi dalla tabella comune qui sotto e non sono il percorso di importazione
consigliato per i nuovi plugin di terze parti.

## Riferimento dei sottopercorsi

I sottopercorsi usati più comunemente, raggruppati per scopo. L'elenco completo generato di
oltre 200 sottopercorsi si trova in `scripts/lib/plugin-sdk-entrypoints.json`.

I sottopercorsi helper riservati ai plugin inclusi compaiono ancora in quell'elenco generato.
Considerali come superfici di dettaglio implementativo/compatibilità, a meno che una pagina della documentazione
non ne promuova esplicitamente uno come pubblico.

### Entry del plugin

| Sottopercorso              | Esportazioni chiave                                                                                                                   |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/plugin-entry`  | `definePluginEntry`                                                                                                                   |
| `plugin-sdk/core`          | `defineChannelPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase`, `defineSetupPluginEntry`, `buildChannelConfigSchema` |
| `plugin-sdk/config-schema` | `OpenClawSchema`                                                                                                                      |
| `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry`                                                                                                     |

<AccordionGroup>
  <Accordion title="Sottopercorsi del canale">
    | Sottopercorso | Esportazioni chiave |
    | --- | --- |
    | `plugin-sdk/channel-core` | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
    | `plugin-sdk/config-schema` | Esportazione dello schema Zod radice `openclaw.json` (`OpenClawSchema`) |
    | `plugin-sdk/channel-setup` | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, più `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
    | `plugin-sdk/setup` | Helper condivisi per la procedura guidata di configurazione, prompt della allowlist, builder dello stato di configurazione |
    | `plugin-sdk/setup-runtime` | `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
    | `plugin-sdk/setup-adapter-runtime` | `createEnvPatchedAccountSetupAdapter` |
    | `plugin-sdk/setup-tools` | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
    | `plugin-sdk/account-core` | Helper per configurazione multi-account/action-gate, helper di fallback dell'account predefinito |
    | `plugin-sdk/account-id` | `DEFAULT_ACCOUNT_ID`, helper di normalizzazione dell'ID account |
    | `plugin-sdk/account-resolution` | Helper per ricerca account + fallback predefinito |
    | `plugin-sdk/account-helpers` | Helper mirati per elenco account/azioni account |
    | `plugin-sdk/channel-pairing` | `createChannelPairingController` |
    | `plugin-sdk/channel-reply-pipeline` | `createChannelReplyPipeline` |
    | `plugin-sdk/channel-config-helpers` | `createHybridChannelConfigAdapter` |
    | `plugin-sdk/channel-config-schema` | Tipi dello schema di configurazione del canale |
    | `plugin-sdk/telegram-command-config` | Helper di normalizzazione/validazione dei comandi personalizzati di Telegram con fallback al contratto incluso |
    | `plugin-sdk/command-gating` | Helper mirati per i gate di autorizzazione dei comandi |
    | `plugin-sdk/channel-policy` | `resolveChannelGroupRequireMention` |
    | `plugin-sdk/channel-lifecycle` | `createAccountStatusSink` |
    | `plugin-sdk/inbound-envelope` | Helper condivisi per instradamento inbound + builder di envelope |
    | `plugin-sdk/inbound-reply-dispatch` | Helper condivisi per registrazione e dispatch inbound |
    | `plugin-sdk/messaging-targets` | Helper per parsing/matching dei target |
    | `plugin-sdk/outbound-media` | Helper condivisi per il caricamento dei media outbound |
    | `plugin-sdk/outbound-runtime` | Helper per identità outbound/delega di invio |
    | `plugin-sdk/poll-runtime` | Helper mirati per la normalizzazione dei sondaggi |
    | `plugin-sdk/thread-bindings-runtime` | Helper per ciclo di vita dei binding dei thread e adattatori |
    | `plugin-sdk/agent-media-payload` | Builder legacy del payload media dell'agente |
    | `plugin-sdk/conversation-runtime` | Helper per binding di conversazioni/thread, pairing e binding configurati |
    | `plugin-sdk/runtime-config-snapshot` | Helper per snapshot della configurazione runtime |
    | `plugin-sdk/runtime-group-policy` | Helper per la risoluzione delle policy di gruppo runtime |
    | `plugin-sdk/channel-status` | Helper condivisi per snapshot/riepilogo dello stato del canale |
    | `plugin-sdk/channel-config-primitives` | Primitive mirate dello schema di configurazione del canale |
    | `plugin-sdk/channel-config-writes` | Helper di autorizzazione per scritture della configurazione del canale |
    | `plugin-sdk/channel-plugin-common` | Esportazioni prelude condivise del plugin di canale |
    | `plugin-sdk/allowlist-config-edit` | Helper di lettura/modifica della configurazione della allowlist |
    | `plugin-sdk/group-access` | Helper condivisi per le decisioni di accesso ai gruppi |
    | `plugin-sdk/direct-dm` | Helper condivisi per autenticazione/guard delle DM dirette |
    | `plugin-sdk/interactive-runtime` | Helper per normalizzazione/riduzione del payload di risposta interattiva |
    | `plugin-sdk/channel-inbound` | Barrel di compatibilità per debounce inbound, matching delle menzioni, helper per policy delle menzioni e helper envelope |
    | `plugin-sdk/channel-mention-gating` | Helper mirati per la policy delle menzioni senza la superficie runtime inbound più ampia |
    | `plugin-sdk/channel-location` | Helper per contesto e formattazione della posizione del canale |
    | `plugin-sdk/channel-logging` | Helper di logging del canale per scarti inbound e errori di typing/ack |
    | `plugin-sdk/channel-send-result` | Tipi del risultato della risposta |
    | `plugin-sdk/channel-actions` | `createMessageToolButtonsSchema`, `createMessageToolCardSchema` |
    | `plugin-sdk/channel-targets` | Helper per parsing/matching dei target |
    | `plugin-sdk/channel-contract` | Tipi del contratto del canale |
    | `plugin-sdk/channel-feedback` | Wiring di feedback/reazioni |
    | `plugin-sdk/channel-secret-runtime` | Helper mirati del contratto dei secret come `collectSimpleChannelFieldAssignments`, `getChannelSurface`, `pushAssignment` e tipi target dei secret |
  </Accordion>

  <Accordion title="Sottopercorsi del provider">
    | Sottopercorso | Esportazioni chiave |
    | --- | --- |
    | `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry` |
    | `plugin-sdk/provider-setup` | Helper curati per la configurazione di provider locali/self-hosted |
    | `plugin-sdk/self-hosted-provider-setup` | Helper mirati per la configurazione di provider self-hosted compatibili con OpenAI |
    | `plugin-sdk/cli-backend` | Valori predefiniti del backend CLI + costanti watchdog |
    | `plugin-sdk/provider-auth-runtime` | Helper runtime per la risoluzione delle API key per i plugin provider |
    | `plugin-sdk/provider-auth-api-key` | Helper per onboarding/scrittura del profilo API key come `upsertApiKeyProfile` |
    | `plugin-sdk/provider-auth-result` | Builder standard del risultato di autenticazione OAuth |
    | `plugin-sdk/provider-auth-login` | Helper condivisi per login interattivo dei plugin provider |
    | `plugin-sdk/provider-env-vars` | Helper di ricerca delle variabili d'ambiente di autenticazione del provider |
    | `plugin-sdk/provider-auth` | `createProviderApiKeyAuthMethod`, `ensureApiKeyFromOptionEnvOrPrompt`, `upsertAuthProfile`, `upsertApiKeyProfile`, `writeOAuthCredentials` |
    | `plugin-sdk/provider-model-shared` | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, builder condivisi delle policy di replay, helper per endpoint del provider e helper di normalizzazione dell'ID modello come `normalizeNativeXaiModelId` |
    | `plugin-sdk/provider-catalog-shared` | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
    | `plugin-sdk/provider-http` | Helper generici per capacità HTTP/endpoint del provider |
    | `plugin-sdk/provider-web-fetch-contract` | Helper mirati del contratto di configurazione/selezione web-fetch come `enablePluginInConfig` e `WebFetchProviderPlugin` |
    | `plugin-sdk/provider-web-fetch` | Helper di registrazione/cache del provider web-fetch |
    | `plugin-sdk/provider-web-search-config-contract` | Helper mirati di configurazione/credenziali web-search per provider che non richiedono wiring di abilitazione del plugin |
    | `plugin-sdk/provider-web-search-contract` | Helper mirati del contratto di configurazione/credenziali web-search come `createWebSearchProviderContractFields`, `enablePluginInConfig`, `resolveProviderWebSearchPluginConfig` e setter/getter di credenziali con ambito |
    | `plugin-sdk/provider-web-search` | Helper di registrazione/cache/runtime del provider web-search |
    | `plugin-sdk/provider-tools` | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, cleanup + diagnostica dello schema Gemini e helper di compatibilità xAI come `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
    | `plugin-sdk/provider-usage` | `fetchClaudeUsage` e simili |
    | `plugin-sdk/provider-stream` | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, tipi di stream wrapper e helper wrapper condivisi per Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot |
    | `plugin-sdk/provider-onboard` | Helper per patch della configurazione di onboarding |
    | `plugin-sdk/global-singleton` | Helper per singleton/map/cache locali al processo |
  </Accordion>

  <Accordion title="Sottopercorsi di autenticazione e sicurezza">
    | Sottopercorso | Esportazioni chiave |
    | --- | --- |
    | `plugin-sdk/command-auth` | `resolveControlCommandGate`, helper del registro dei comandi, helper di autorizzazione del mittente |
    | `plugin-sdk/command-status` | Builder di messaggi di comando/aiuto come `buildCommandsMessagePaginated` e `buildHelpMessage` |
    | `plugin-sdk/approval-auth-runtime` | Risoluzione degli approvatori e helper di autenticazione delle azioni nella stessa chat |
    | `plugin-sdk/approval-client-runtime` | Helper di profilo/filtro di approvazione `exec` nativa |
    | `plugin-sdk/approval-delivery-runtime` | Adattatori nativi di capacità/consegna delle approvazioni |
    | `plugin-sdk/approval-gateway-runtime` | Helper condiviso di risoluzione del Gateway di approvazione |
    | `plugin-sdk/approval-handler-adapter-runtime` | Helper leggeri di caricamento degli adattatori di approvazione nativa per entrypoint di canale ad alta frequenza |
    | `plugin-sdk/approval-handler-runtime` | Helper runtime più ampi per il gestore delle approvazioni; preferisci i seam più ristretti adapter/gateway quando sono sufficienti |
    | `plugin-sdk/approval-native-runtime` | Helper per target di approvazione nativa + binding dell'account |
    | `plugin-sdk/approval-reply-runtime` | Helper del payload di risposta per approvazioni `exec`/plugin |
    | `plugin-sdk/command-auth-native` | Autenticazione dei comandi nativi + helper nativi per target di sessione |
    | `plugin-sdk/command-detection` | Helper condivisi per il rilevamento dei comandi |
    | `plugin-sdk/command-surface` | Normalizzazione del corpo del comando e helper della superficie di comando |
    | `plugin-sdk/allow-from` | `formatAllowFromLowercase` |
    | `plugin-sdk/channel-secret-runtime` | Helper mirati di raccolta del contratto dei secret per superfici secret di canale/plugin |
    | `plugin-sdk/secret-ref-runtime` | Helper mirati `coerceSecretRef` e di tipizzazione di SecretRef per il parsing di configurazione/contratto dei secret |
    | `plugin-sdk/security-runtime` | Helper condivisi per trust, gating delle DM, contenuti esterni e raccolta dei secret |
    | `plugin-sdk/ssrf-policy` | Helper della policy SSRF per allowlist degli host e reti private |
    | `plugin-sdk/ssrf-dispatcher` | Helper mirati del dispatcher pinned senza l'ampia superficie runtime dell'infrastruttura |
    | `plugin-sdk/ssrf-runtime` | Helper per dispatcher pinned, fetch protetto da SSRF e policy SSRF |
    | `plugin-sdk/secret-input` | Helper di parsing degli input secret |
    | `plugin-sdk/webhook-ingress` | Helper per richieste/target Webhook |
    | `plugin-sdk/webhook-request-guards` | Helper per dimensione del body della richiesta/timeout |
  </Accordion>

  <Accordion title="Sottopercorsi di runtime e archiviazione">
    | Sottopercorso | Esportazioni chiave |
    | --- | --- |
    | `plugin-sdk/runtime` | Ampi helper per runtime/logging/backup/installazione plugin |
    | `plugin-sdk/runtime-env` | Helper mirati per env runtime, logger, timeout, retry e backoff |
    | `plugin-sdk/channel-runtime-context` | Helper generici per registrazione e lookup del contesto runtime del canale |
    | `plugin-sdk/runtime-store` | `createPluginRuntimeStore` |
    | `plugin-sdk/plugin-runtime` | Helper condivisi per comandi/hook/http/interattivi del plugin |
    | `plugin-sdk/hook-runtime` | Helper condivisi per la pipeline di hook Webhook/interna |
    | `plugin-sdk/lazy-runtime` | Helper per import/binding runtime lazy come `createLazyRuntimeModule`, `createLazyRuntimeMethod` e `createLazyRuntimeSurface` |
    | `plugin-sdk/process-runtime` | Helper per esecuzione di processi |
    | `plugin-sdk/cli-runtime` | Helper per formattazione CLI, attesa e versione |
    | `plugin-sdk/gateway-runtime` | Helper per client Gateway e patch dello stato del canale |
    | `plugin-sdk/config-runtime` | Helper per caricamento/scrittura della configurazione |
    | `plugin-sdk/telegram-command-config` | Normalizzazione di nome/descrizione dei comandi Telegram e controlli di duplicati/conflitti, anche quando la superficie del contratto Telegram incluso non è disponibile |
    | `plugin-sdk/text-autolink-runtime` | Rilevamento di autolink di riferimenti a file senza l'ampio barrel `text-runtime` |
    | `plugin-sdk/approval-runtime` | Helper per approvazioni `exec`/plugin, builder di capacità di approvazione, helper auth/profilo, helper nativi di routing/runtime |
    | `plugin-sdk/reply-runtime` | Helper condivisi per runtime inbound/risposta, chunking, dispatch, Heartbeat, pianificatore delle risposte |
    | `plugin-sdk/reply-dispatch-runtime` | Helper mirati di dispatch/finalizzazione delle risposte |
    | `plugin-sdk/reply-history` | Helper condivisi per la cronologia delle risposte a breve finestra come `buildHistoryContext`, `recordPendingHistoryEntry` e `clearHistoryEntriesIfEnabled` |
    | `plugin-sdk/reply-reference` | `createReplyReferencePlanner` |
    | `plugin-sdk/reply-chunking` | Helper mirati per chunking di testo/Markdown |
    | `plugin-sdk/session-store-runtime` | Helper per percorso del session store + updated-at |
    | `plugin-sdk/state-paths` | Helper per i percorsi delle directory state/OAuth |
    | `plugin-sdk/routing` | Helper per route/session-key/binding dell'account come `resolveAgentRoute`, `buildAgentSessionKey` e `resolveDefaultAgentBoundAccountId` |
    | `plugin-sdk/status-helpers` | Helper condivisi per riepiloghi di stato di canale/account, valori predefiniti dello stato runtime e helper per metadati dei problemi |
    | `plugin-sdk/target-resolver-runtime` | Helper condivisi per il resolver dei target |
    | `plugin-sdk/string-normalization-runtime` | Helper per normalizzazione di slug/stringhe |
    | `plugin-sdk/request-url` | Estrai URL stringa da input simili a fetch/richiesta |
    | `plugin-sdk/run-command` | Esecutore di comandi temporizzato con risultati stdout/stderr normalizzati |
    | `plugin-sdk/param-readers` | Lettori comuni di parametri per tool/CLI |
    | `plugin-sdk/tool-payload` | Estrai payload normalizzati dagli oggetti risultato dei tool |
    | `plugin-sdk/tool-send` | Estrai campi canonici del target di invio dagli argomenti del tool |
    | `plugin-sdk/temp-path` | Helper condivisi per percorsi di download temporanei |
    | `plugin-sdk/logging-core` | Helper per logger di sottosistema e redazione |
    | `plugin-sdk/markdown-table-runtime` | Helper per modalità tabella Markdown |
    | `plugin-sdk/json-store` | Piccoli helper per lettura/scrittura dello stato JSON |
    | `plugin-sdk/file-lock` | Helper re-entrant per file lock |
    | `plugin-sdk/persistent-dedupe` | Helper per cache di deduplicazione persistente su disco |
    | `plugin-sdk/acp-runtime` | Helper ACP per runtime/sessione e reply-dispatch |
    | `plugin-sdk/acp-binding-resolve-runtime` | Risoluzione di binding ACP in sola lettura senza import di startup del ciclo di vita |
    | `plugin-sdk/agent-config-primitives` | Primitive mirate dello schema di configurazione runtime dell'agente |
    | `plugin-sdk/boolean-param` | Lettore permissivo di parametri booleani |
    | `plugin-sdk/dangerous-name-runtime` | Helper di risoluzione per matching di nomi pericolosi |
    | `plugin-sdk/device-bootstrap` | Helper per bootstrap del dispositivo e token di pairing |
    | `plugin-sdk/extension-shared` | Primitive helper condivise per canali passivi, stato e proxy ambientale |
    | `plugin-sdk/models-provider-runtime` | Helper per il comando `/models` e risposte dei provider |
    | `plugin-sdk/skill-commands-runtime` | Helper per l'elenco dei comandi Skills |
    | `plugin-sdk/native-command-registry` | Helper per registro/build/serializzazione dei comandi nativi |
    | `plugin-sdk/agent-harness` | Superficie sperimentale per plugin trusted per harness di agente a basso livello: tipi di harness, helper di steer/abort dell'esecuzione attiva, helper di bridge degli strumenti OpenClaw e utility per i risultati dei tentativi |
    | `plugin-sdk/provider-zai-endpoint` | Helper per il rilevamento degli endpoint Z.A.I |
    | `plugin-sdk/infra-runtime` | Helper per eventi di sistema/Heartbeat |
    | `plugin-sdk/collection-runtime` | Piccoli helper per cache limitate |
    | `plugin-sdk/diagnostic-runtime` | Helper per flag ed eventi diagnostici |
    | `plugin-sdk/error-runtime` | Grafo degli errori, formattazione, helper condivisi di classificazione degli errori, `isApprovalNotFoundError` |
    | `plugin-sdk/fetch-runtime` | Helper per fetch wrapperizzato, proxy e lookup pinned |
    | `plugin-sdk/runtime-fetch` | Fetch runtime consapevole del dispatcher senza import di proxy/guarded-fetch |
    | `plugin-sdk/response-limit-runtime` | Lettore limitato del body della risposta senza l'ampia superficie media runtime |
    | `plugin-sdk/session-binding-runtime` | Stato corrente del binding della conversazione senza routing dei binding configurati o store di pairing |
    | `plugin-sdk/session-store-runtime` | Helper di lettura del session store senza ampi import di scritture/manutenzione della configurazione |
    | `plugin-sdk/context-visibility-runtime` | Risoluzione della visibilità del contesto e filtro del contesto supplementare senza ampi import di configurazione/sicurezza |
    | `plugin-sdk/string-coerce-runtime` | Helper mirati per coercizione e normalizzazione di record/stringhe primitive senza import di Markdown/logging |
    | `plugin-sdk/host-runtime` | Helper per normalizzazione di hostname e host SCP |
    | `plugin-sdk/retry-runtime` | Helper per configurazione del retry ed esecutore del retry |
    | `plugin-sdk/agent-runtime` | Helper per directory/identità/workspace dell'agente |
    | `plugin-sdk/directory-runtime` | Query/deduplicazione delle directory basata sulla configurazione |
    | `plugin-sdk/keyed-async-queue` | `KeyedAsyncQueue` |
  </Accordion>

  <Accordion title="Sottopercorsi di capacità e test">
    | Sottopercorso | Esportazioni chiave |
    | --- | --- |
    | `plugin-sdk/media-runtime` | Helper condivisi per fetch/trasformazione/archiviazione dei media più builder di payload media |
    | `plugin-sdk/media-generation-runtime` | Helper condivisi per failover della generazione media, selezione dei candidati e messaggistica per modello mancante |
    | `plugin-sdk/media-understanding` | Tipi di provider per comprensione dei media più esportazioni helper rivolte ai provider per immagini/audio |
    | `plugin-sdk/text-runtime` | Helper condivisi per testo/Markdown/logging come rimozione del testo visibile all'assistente, helper per render/chunking/tabelle Markdown, helper di redazione, helper per tag di direttiva e utility per testo sicuro |
    | `plugin-sdk/text-chunking` | Helper per chunking del testo outbound |
    | `plugin-sdk/speech` | Tipi di provider speech più helper rivolti ai provider per direttive, registro e validazione |
    | `plugin-sdk/speech-core` | Tipi condivisi di provider speech, helper per registro, direttive e normalizzazione |
    | `plugin-sdk/realtime-transcription` | Tipi di provider per trascrizione in tempo reale e helper di registro |
    | `plugin-sdk/realtime-voice` | Tipi di provider per voce in tempo reale e helper di registro |
    | `plugin-sdk/image-generation` | Tipi di provider per generazione di immagini |
    | `plugin-sdk/image-generation-core` | Tipi condivisi per generazione di immagini, helper per failover, autenticazione e registro |
    | `plugin-sdk/music-generation` | Tipi di provider/richiesta/risultato per generazione musicale |
    | `plugin-sdk/music-generation-core` | Tipi condivisi per generazione musicale, helper per failover, lookup del provider e parsing di model-ref |
    | `plugin-sdk/video-generation` | Tipi di provider/richiesta/risultato per generazione video |
    | `plugin-sdk/video-generation-core` | Tipi condivisi per generazione video, helper per failover, lookup del provider e parsing di model-ref |
    | `plugin-sdk/webhook-targets` | Registro dei target Webhook e helper per installazione delle route |
    | `plugin-sdk/webhook-path` | Helper per normalizzazione del percorso Webhook |
    | `plugin-sdk/web-media` | Helper condivisi per il caricamento di media remoti/locali |
    | `plugin-sdk/zod` | `zod` riesportato per i consumatori dell'SDK del plugin |
    | `plugin-sdk/testing` | `installCommonResolveTargetErrorCases`, `shouldAckReaction` |
  </Accordion>

  <Accordion title="Sottopercorsi della memoria">
    | Sottopercorso | Esportazioni chiave |
    | --- | --- |
    | `plugin-sdk/memory-core` | Superficie helper `memory-core` inclusa per helper di manager/configurazione/file/CLI |
    | `plugin-sdk/memory-core-engine-runtime` | Facciata runtime per indice/ricerca della memoria |
    | `plugin-sdk/memory-core-host-engine-foundation` | Esportazioni del motore foundation dell'host della memoria |
    | `plugin-sdk/memory-core-host-engine-embeddings` | Contratti di embedding dell'host della memoria, accesso al registro, provider locale e helper generici batch/remoti |
    | `plugin-sdk/memory-core-host-engine-qmd` | Esportazioni del motore QMD dell'host della memoria |
    | `plugin-sdk/memory-core-host-engine-storage` | Esportazioni del motore di archiviazione dell'host della memoria |
    | `plugin-sdk/memory-core-host-multimodal` | Helper multimodali dell'host della memoria |
    | `plugin-sdk/memory-core-host-query` | Helper di query dell'host della memoria |
    | `plugin-sdk/memory-core-host-secret` | Helper secret dell'host della memoria |
    | `plugin-sdk/memory-core-host-events` | Helper del journal degli eventi dell'host della memoria |
    | `plugin-sdk/memory-core-host-status` | Helper di stato dell'host della memoria |
    | `plugin-sdk/memory-core-host-runtime-cli` | Helper runtime CLI dell'host della memoria |
    | `plugin-sdk/memory-core-host-runtime-core` | Helper runtime core dell'host della memoria |
    | `plugin-sdk/memory-core-host-runtime-files` | Helper file/runtime dell'host della memoria |
    | `plugin-sdk/memory-host-core` | Alias neutrale rispetto al vendor per gli helper runtime core dell'host della memoria |
    | `plugin-sdk/memory-host-events` | Alias neutrale rispetto al vendor per gli helper del journal degli eventi dell'host della memoria |
    | `plugin-sdk/memory-host-files` | Alias neutrale rispetto al vendor per gli helper file/runtime dell'host della memoria |
    | `plugin-sdk/memory-host-markdown` | Helper condivisi per Markdown gestito per plugin adiacenti alla memoria |
    | `plugin-sdk/memory-host-search` | Facciata runtime di Active Memory per l'accesso al gestore di ricerca |
    | `plugin-sdk/memory-host-status` | Alias neutrale rispetto al vendor per gli helper di stato dell'host della memoria |
    | `plugin-sdk/memory-lancedb` | Superficie helper `memory-lancedb` inclusa |
  </Accordion>

  <Accordion title="Sottopercorsi helper inclusi riservati">
    | Famiglia | Sottopercorsi attuali | Uso previsto |
    | --- | --- | --- |
    | Browser | `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support` | Helper di supporto del plugin browser incluso (`browser-support` rimane il barrel di compatibilità) |
    | Matrix | `plugin-sdk/matrix`, `plugin-sdk/matrix-helper`, `plugin-sdk/matrix-runtime-heavy`, `plugin-sdk/matrix-runtime-shared`, `plugin-sdk/matrix-runtime-surface`, `plugin-sdk/matrix-surface`, `plugin-sdk/matrix-thread-bindings` | Superficie helper/runtime Matrix inclusa |
    | Line | `plugin-sdk/line`, `plugin-sdk/line-core`, `plugin-sdk/line-runtime`, `plugin-sdk/line-surface` | Superficie helper/runtime LINE inclusa |
    | IRC | `plugin-sdk/irc`, `plugin-sdk/irc-surface` | Superficie helper IRC inclusa |
    | Helper specifici del canale | `plugin-sdk/googlechat`, `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles`, `plugin-sdk/bluebubbles-policy`, `plugin-sdk/mattermost`, `plugin-sdk/mattermost-policy`, `plugin-sdk/feishu-conversation`, `plugin-sdk/msteams`, `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`, `plugin-sdk/twitch` | Seam di compatibilità/helper di canale inclusi |
    | Helper specifici di auth/plugin | `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`, `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`, `plugin-sdk/thread-ownership`, `plugin-sdk/voice-call` | Seam helper di funzionalità/plugin inclusi; `plugin-sdk/github-copilot-token` attualmente esporta `DEFAULT_COPILOT_API_BASE_URL`, `deriveCopilotApiBaseUrlFromToken` e `resolveCopilotApiToken` |
  </Accordion>
</AccordionGroup>

## API di registrazione

La callback `register(api)` riceve un oggetto `OpenClawPluginApi` con questi
metodi:

### Registrazione delle capacità

| Metodo                                           | Cosa registra                         |
| ------------------------------------------------ | ------------------------------------- |
| `api.registerProvider(...)`                      | Inferenza di testo (LLM)              |
| `api.registerAgentHarness(...)`                  | Esecutore sperimentale di agenti a basso livello |
| `api.registerCliBackend(...)`                    | Backend CLI locale per l'inferenza    |
| `api.registerChannel(...)`                       | Canale di messaggistica               |
| `api.registerSpeechProvider(...)`                | Sintesi text-to-speech / STT          |
| `api.registerRealtimeTranscriptionProvider(...)` | Trascrizione realtime in streaming    |
| `api.registerRealtimeVoiceProvider(...)`         | Sessioni vocali realtime duplex       |
| `api.registerMediaUnderstandingProvider(...)`    | Analisi di immagini/audio/video       |
| `api.registerImageGenerationProvider(...)`       | Generazione di immagini               |
| `api.registerMusicGenerationProvider(...)`       | Generazione musicale                  |
| `api.registerVideoGenerationProvider(...)`       | Generazione video                     |
| `api.registerWebFetchProvider(...)`              | Provider di fetch / scraping web      |
| `api.registerWebSearchProvider(...)`             | Ricerca web                           |

### Tool e comandi

| Metodo                          | Cosa registra                                |
| ------------------------------- | -------------------------------------------- |
| `api.registerTool(tool, opts?)` | Tool dell'agente (obbligatorio o `{ optional: true }`) |
| `api.registerCommand(def)`      | Comando personalizzato (bypassa l'LLM)       |

### Infrastruttura

| Metodo                                         | Cosa registra                           |
| ---------------------------------------------- | --------------------------------------- |
| `api.registerHook(events, handler, opts?)`     | Hook di evento                          |
| `api.registerHttpRoute(params)`                | Endpoint HTTP del Gateway               |
| `api.registerGatewayMethod(name, handler)`     | Metodo RPC del Gateway                  |
| `api.registerCli(registrar, opts?)`            | Sottocomando CLI                        |
| `api.registerService(service)`                 | Servizio in background                  |
| `api.registerInteractiveHandler(registration)` | Gestore interattivo                     |
| `api.registerMemoryPromptSupplement(builder)`  | Sezione aggiuntiva del prompt adiacente alla memoria |
| `api.registerMemoryCorpusSupplement(adapter)`  | Corpus aggiuntivo di ricerca/lettura della memoria |

Gli spazi dei nomi amministrativi core riservati (`config.*`, `exec.approvals.*`, `wizard.*`,
`update.*`) restano sempre `operator.admin`, anche se un plugin prova ad assegnare
un ambito di metodo Gateway più ristretto. Preferisci prefissi specifici del plugin per
i metodi posseduti dal plugin.

### Metadati di registrazione CLI

`api.registerCli(registrar, opts?)` accetta due tipi di metadati di primo livello:

- `commands`: radici di comando esplicite possedute dal registrar
- `descriptors`: descrittori di comando al momento del parsing usati per l'help della CLI root,
  il routing e la registrazione lazy della CLI del plugin

Se vuoi che un comando del plugin resti caricato lazy nel normale percorso della CLI root,
fornisci `descriptors` che coprano ogni radice di comando di primo livello esposta da quel
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
        description: "Gestisci account Matrix, verifica, dispositivi e stato del profilo",
        hasSubcommands: true,
      },
    ],
  },
);
```

Usa `commands` da solo solo quando non hai bisogno della registrazione lazy della CLI root.
Quel percorso di compatibilità eager resta supportato, ma non installa
placeholder supportati da descriptor per il caricamento lazy al momento del parsing.

### Registrazione del backend CLI

`api.registerCliBackend(...)` consente a un plugin di gestire la configurazione predefinita di un
backend CLI AI locale come `codex-cli`.

- Il `id` del backend diventa il prefisso del provider nei model ref come `codex-cli/gpt-5`.
- La `config` del backend usa la stessa forma di `agents.defaults.cliBackends.<id>`.
- La configurazione utente ha comunque la precedenza. OpenClaw unisce `agents.defaults.cliBackends.<id>` sopra il
  valore predefinito del plugin prima di eseguire la CLI.
- Usa `normalizeConfig` quando un backend ha bisogno di riscritture di compatibilità dopo il merge
  (per esempio normalizzando vecchie forme di flag).

### Slot esclusivi

| Metodo                                     | Cosa registra                                                                                                                                         |
| ------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| `api.registerContextEngine(id, factory)`   | Motore di contesto (uno attivo alla volta). La callback `assemble()` riceve `availableTools` e `citationsMode` così il motore può adattare le aggiunte al prompt. |
| `api.registerMemoryCapability(capability)` | Capacità di memoria unificata                                                                                                                         |
| `api.registerMemoryPromptSection(builder)` | Builder della sezione prompt della memoria                                                                                                            |
| `api.registerMemoryFlushPlan(resolver)`    | Resolver del piano di flush della memoria                                                                                                             |
| `api.registerMemoryRuntime(runtime)`       | Adattatore runtime della memoria                                                                                                                      |

### Adattatori di embedding della memoria

| Metodo                                         | Cosa registra                                  |
| ---------------------------------------------- | ---------------------------------------------- |
| `api.registerMemoryEmbeddingProvider(adapter)` | Adattatore di embedding della memoria per il plugin attivo |

- `registerMemoryCapability` è l'API preferita del plugin di memoria esclusivo.
- `registerMemoryCapability` può anche esporre `publicArtifacts.listArtifacts(...)`
  così i plugin companion possono consumare artefatti di memoria esportati tramite
  `openclaw/plugin-sdk/memory-host-core` invece di entrare nel layout privato
  di uno specifico plugin di memoria.
- `registerMemoryPromptSection`, `registerMemoryFlushPlan` e
  `registerMemoryRuntime` sono API legacy-compatibili del plugin di memoria esclusivo.
- `registerMemoryEmbeddingProvider` consente al plugin di memoria attivo di registrare uno
  o più ID di adattatore di embedding (per esempio `openai`, `gemini` o un ID personalizzato definito dal plugin).
- La configurazione utente come `agents.defaults.memorySearch.provider` e
  `agents.defaults.memorySearch.fallback` viene risolta rispetto a quegli ID di adattatore
  registrati.

### Eventi e ciclo di vita

| Metodo                                       | Cosa fa                     |
| -------------------------------------------- | --------------------------- |
| `api.on(hookName, handler, opts?)`           | Hook del ciclo di vita tipizzato |
| `api.onConversationBindingResolved(handler)` | Callback di binding della conversazione |

### Semantica delle decisioni degli hook

- `before_tool_call`: restituire `{ block: true }` è terminale. Non appena un gestore lo imposta, i gestori con priorità inferiore vengono saltati.
- `before_tool_call`: restituire `{ block: false }` viene trattato come nessuna decisione (uguale a omettere `block`), non come un override.
- `before_install`: restituire `{ block: true }` è terminale. Non appena un gestore lo imposta, i gestori con priorità inferiore vengono saltati.
- `before_install`: restituire `{ block: false }` viene trattato come nessuna decisione (uguale a omettere `block`), non come un override.
- `reply_dispatch`: restituire `{ handled: true, ... }` è terminale. Non appena un gestore rivendica il dispatch, i gestori con priorità inferiore e il percorso di dispatch predefinito del modello vengono saltati.
- `message_sending`: restituire `{ cancel: true }` è terminale. Non appena un gestore lo imposta, i gestori con priorità inferiore vengono saltati.
- `message_sending`: restituire `{ cancel: false }` viene trattato come nessuna decisione (uguale a omettere `cancel`), non come un override.

### Campi dell'oggetto API

| Campo                    | Tipo                      | Descrizione                                                                                 |
| ------------------------ | ------------------------- | ------------------------------------------------------------------------------------------- |
| `api.id`                 | `string`                  | ID del plugin                                                                               |
| `api.name`               | `string`                  | Nome visualizzato                                                                           |
| `api.version`            | `string?`                 | Versione del plugin (opzionale)                                                             |
| `api.description`        | `string?`                 | Descrizione del plugin (opzionale)                                                          |
| `api.source`             | `string`                  | Percorso sorgente del plugin                                                                |
| `api.rootDir`            | `string?`                 | Directory root del plugin (opzionale)                                                       |
| `api.config`             | `OpenClawConfig`          | Snapshot corrente della configurazione (snapshot runtime in-memory attivo quando disponibile) |
| `api.pluginConfig`       | `Record<string, unknown>` | Configurazione specifica del plugin da `plugins.entries.<id>.config`                        |
| `api.runtime`            | `PluginRuntime`           | [Helper runtime](/it/plugins/sdk-runtime)                                                      |
| `api.logger`             | `PluginLogger`            | Logger con scope (`debug`, `info`, `warn`, `error`)                                         |
| `api.registrationMode`   | `PluginRegistrationMode`  | Modalità di caricamento corrente; `"setup-runtime"` è la finestra leggera di avvio/configurazione prima dell'entry completa |
| `api.resolvePath(input)` | `(string) => string`      | Risolve il percorso relativo alla root del plugin                                           |

## Convenzione dei moduli interni

All'interno del tuo plugin, usa file barrel locali per le importazioni interne:

```
my-plugin/
  api.ts            # Esportazioni pubbliche per consumatori esterni
  runtime-api.ts    # Esportazioni runtime solo interne
  index.ts          # Entry point del plugin
  setup-entry.ts    # Entry leggera solo per la configurazione (opzionale)
```

<Warning>
  Non importare mai il tuo plugin tramite `openclaw/plugin-sdk/<your-plugin>`
  dal codice di produzione. Indirizza le importazioni interne tramite `./api.ts` o
  `./runtime-api.ts`. Il percorso SDK è solo il contratto esterno.
</Warning>

Le superfici pubbliche dei plugin inclusi caricate tramite facade (`api.ts`, `runtime-api.ts`,
`index.ts`, `setup-entry.ts` e file entry pubblici simili) ora preferiscono lo
snapshot attivo della configurazione runtime quando OpenClaw è già in esecuzione. Se non esiste ancora
uno snapshot runtime, usano come fallback il file di configurazione risolto su disco.

I plugin provider possono anche esporre un barrel di contratto locale al plugin e ristretto quando un
helper è intenzionalmente specifico del provider e non appartiene ancora a un sottopercorso
SDK generico. Esempio incluso attuale: il provider Anthropic mantiene i suoi helper
di stream Claude nel proprio seam pubblico `api.ts` / `contract-api.ts` invece di
promuovere la logica dell'header beta Anthropic e `service_tier` in un contratto
generico `plugin-sdk/*`.

Altri esempi inclusi attuali:

- `@openclaw/openai-provider`: `api.ts` esporta builder di provider,
  helper per modelli predefiniti e builder di provider realtime
- `@openclaw/openrouter-provider`: `api.ts` esporta il builder del provider più
  helper di onboarding/configurazione

<Warning>
  Il codice di produzione delle estensioni dovrebbe anche evitare importazioni `openclaw/plugin-sdk/<other-plugin>`.
  Se un helper è davvero condiviso, promuovilo a un sottopercorso SDK neutrale
  come `openclaw/plugin-sdk/speech`, `.../provider-model-shared` o un'altra
  superficie orientata alle capacità invece di accoppiare due plugin tra loro.
</Warning>

## Correlati

- [Punti di ingresso](/it/plugins/sdk-entrypoints) — opzioni di `definePluginEntry` e `defineChannelPluginEntry`
- [Helper runtime](/it/plugins/sdk-runtime) — riferimento completo dello spazio dei nomi `api.runtime`
- [Configurazione e setup](/it/plugins/sdk-setup) — packaging, manifest, schemi di configurazione
- [Testing](/it/plugins/sdk-testing) — utility di test e regole di lint
- [Migrazione dell'SDK](/it/plugins/sdk-migration) — migrazione da superfici deprecate
- [Elementi interni del plugin](/it/plugins/architecture) — architettura approfondita e modello di capacità
