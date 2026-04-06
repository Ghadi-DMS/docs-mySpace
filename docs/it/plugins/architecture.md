---
read_when:
    - Creazione o debug di plugin OpenClaw nativi
    - Comprendere il modello delle capacità dei plugin o i confini di ownership
    - Lavorare sulla pipeline di caricamento dei plugin o sul registry
    - Implementare hook runtime dei provider o plugin di canale
sidebarTitle: Internals
summary: 'Elementi interni dei plugin: modello delle capacità, ownership, contratti, pipeline di caricamento e helper runtime'
title: Elementi interni dei plugin
x-i18n:
    generated_at: "2026-04-06T03:12:00Z"
    model: gpt-5.4
    provider: openai
    source_hash: d39158455701dedfb75f6c20b8c69fd36ed9841f1d92bed1915f448df57fd47b
    source_path: plugins/architecture.md
    workflow: 15
---

# Elementi interni dei plugin

<Info>
  Questo è il **riferimento architetturale approfondito**. Per guide pratiche, vedi:
  - [Install and use plugins](/it/tools/plugin) — guida per gli utenti
  - [Getting Started](/it/plugins/building-plugins) — primo tutorial sui plugin
  - [Channel Plugins](/it/plugins/sdk-channel-plugins) — crea un canale di messaggistica
  - [Provider Plugins](/it/plugins/sdk-provider-plugins) — crea un provider di modelli
  - [SDK Overview](/it/plugins/sdk-overview) — mappa di importazione e API di registrazione
</Info>

Questa pagina descrive l'architettura interna del sistema di plugin OpenClaw.

## Modello pubblico delle capacità

Le capacità sono il modello pubblico dei **plugin nativi** all'interno di OpenClaw. Ogni
plugin OpenClaw nativo si registra rispetto a uno o più tipi di capacità:

| Capability             | Metodo di registrazione                         | Plugin di esempio                    |
| ---------------------- | ----------------------------------------------- | ------------------------------------ |
| Inferenza testuale     | `api.registerProvider(...)`                     | `openai`, `anthropic`                |
| Sintesi vocale         | `api.registerSpeechProvider(...)`               | `elevenlabs`, `microsoft`            |
| Trascrizione realtime  | `api.registerRealtimeTranscriptionProvider(...)` | `openai`                             |
| Voce realtime          | `api.registerRealtimeVoiceProvider(...)`        | `openai`                             |
| Comprensione media     | `api.registerMediaUnderstandingProvider(...)`   | `openai`, `google`                   |
| Generazione immagini   | `api.registerImageGenerationProvider(...)`      | `openai`, `google`, `fal`, `minimax` |
| Generazione musicale   | `api.registerMusicGenerationProvider(...)`      | `google`, `minimax`                  |
| Generazione video      | `api.registerVideoGenerationProvider(...)`      | `qwen`                               |
| Recupero web           | `api.registerWebFetchProvider(...)`             | `firecrawl`                          |
| Ricerca web            | `api.registerWebSearchProvider(...)`            | `google`                             |
| Canale / messaggistica | `api.registerChannel(...)`                      | `msteams`, `matrix`                  |

Un plugin che registra zero capacità ma fornisce hook, strumenti o
servizi è un plugin **legacy solo hook**. Questo pattern è ancora pienamente supportato.

### Posizionamento sulla compatibilità esterna

Il modello delle capacità è già integrato nel core e oggi è usato dai plugin
nativi/inclusi, ma la compatibilità dei plugin esterni richiede ancora un criterio più rigoroso di “è
esportato, quindi è congelato”.

Indicazioni attuali:

- **plugin esterni esistenti:** mantieni funzionanti le integrazioni basate su hook; trattale
  come base di compatibilità
- **nuovi plugin nativi/inclusi:** preferisci la registrazione esplicita delle capacità rispetto a
  accessi specifici al vendor o nuovi design solo hook
- **plugin esterni che adottano la registrazione delle capacità:** consentito, ma tratta le
  superfici helper specifiche per capacità come in evoluzione a meno che la documentazione non
  indichi esplicitamente un contratto stabile

Regola pratica:

- le API di registrazione delle capacità sono la direzione prevista
- gli hook legacy restano il percorso più sicuro per evitare rotture per i plugin esterni durante
  la transizione
- i sottopercorsi helper esportati non sono tutti equivalenti; preferisci il contratto ristretto documentato,
  non esportazioni helper incidentali

### Forme dei plugin

OpenClaw classifica ogni plugin caricato in una forma in base al suo comportamento
di registrazione effettivo (non solo ai metadati statici):

- **plain-capability** -- registra esattamente un tipo di capacità (per esempio un
  plugin solo provider come `mistral`)
- **hybrid-capability** -- registra più tipi di capacità (per esempio
  `openai` possiede inferenza testuale, sintesi vocale, comprensione media e generazione
  immagini)
- **hook-only** -- registra solo hook (tipizzati o personalizzati), nessuna capacità,
  nessuno strumento, comando o servizio
- **non-capability** -- registra strumenti, comandi, servizi o route ma nessuna
  capacità

Usa `openclaw plugins inspect <id>` per vedere la forma e la
suddivisione delle capacità di un plugin. Vedi [CLI reference](/cli/plugins#inspect) per i dettagli.

### Hook legacy

L'hook `before_agent_start` resta supportato come percorso di compatibilità per
i plugin solo hook. Plugin legacy reali dipendono ancora da esso.

Direzione:

- mantienilo funzionante
- documentalo come legacy
- preferisci `before_model_resolve` per il lavoro di override del modello/provider
- preferisci `before_prompt_build` per il lavoro di mutazione del prompt
- rimuovilo solo quando l'uso reale cala e la copertura delle fixture dimostra la sicurezza della migrazione

### Segnali di compatibilità

Quando esegui `openclaw doctor` o `openclaw plugins inspect <id>`, potresti vedere
una di queste etichette:

| Segnale                   | Significato                                                  |
| ------------------------- | ------------------------------------------------------------ |
| **config valid**          | La configurazione viene analizzata correttamente e i plugin vengono risolti |
| **compatibility advisory** | Il plugin usa un pattern supportato ma più vecchio (es. `hook-only`) |
| **legacy warning**        | Il plugin usa `before_agent_start`, che è deprecato         |
| **hard error**            | La configurazione non è valida o il plugin non è stato caricato |

Né `hook-only` né `before_agent_start` romperanno oggi il tuo plugin --
`hook-only` è un avviso, e `before_agent_start` genera solo un warning. Questi
segnali compaiono anche in `openclaw status --all` e `openclaw plugins doctor`.

## Panoramica dell'architettura

Il sistema di plugin di OpenClaw ha quattro livelli:

1. **Manifest + discovery**
   OpenClaw trova i plugin candidati dai percorsi configurati, dalle root del workspace,
   dalle root globali delle estensioni e dalle estensioni incluse. La discovery legge prima i
   manifest nativi `openclaw.plugin.json` più i manifest dei bundle supportati.
2. **Abilitazione + validazione**
   Il core decide se un plugin individuato è abilitato, disabilitato, bloccato o
   selezionato per uno slot esclusivo come la memoria.
3. **Caricamento runtime**
   I plugin OpenClaw nativi vengono caricati in-process tramite jiti e registrano
   capacità in un registry centrale. I bundle compatibili vengono normalizzati in
   record del registry senza importare codice runtime.
4. **Consumo delle superfici**
   Il resto di OpenClaw legge il registry per esporre strumenti, canali, configurazione
   provider, hook, route HTTP, comandi CLI e servizi.

Per la CLI dei plugin in particolare, la discovery dei comandi root è divisa in due fasi:

- i metadati al momento del parsing provengono da `registerCli(..., { descriptors: [...] })`
- il vero modulo CLI del plugin può restare lazy e registrarsi alla prima invocazione

Questo mantiene il codice CLI del plugin all'interno del plugin, pur consentendo a OpenClaw
di riservare i nomi dei comandi root prima del parsing.

Il confine progettuale importante:

- discovery + validazione della configurazione dovrebbero funzionare a partire dai **metadati del manifest/schema**
  senza eseguire codice del plugin
- il comportamento runtime nativo deriva dal percorso `register(api)` del modulo plugin

Questa separazione permette a OpenClaw di validare la configurazione, spiegare plugin mancanti/disabilitati e
costruire suggerimenti UI/schema prima che il runtime completo sia attivo.

### Plugin di canale e strumento di messaggistica condiviso

I plugin di canale non devono registrare uno strumento separato di invio/modifica/reazione per
le normali azioni di chat. OpenClaw mantiene un unico strumento `message` condiviso nel core, e
i plugin di canale possiedono dietro di esso la discovery e l'esecuzione specifiche del canale.

Il confine attuale è:

- il core possiede l'host condiviso dello strumento `message`, il wiring del prompt, la
  gestione di sessioni/thread e il dispatch dell'esecuzione
- i plugin di canale possiedono la discovery di azioni con ambito, la discovery delle
  capacità e qualsiasi frammento di schema specifico del canale
- i plugin di canale possiedono la grammatica della conversazione di sessione specifica del provider, ad esempio
  come gli ID conversazione codificano gli ID thread o vengono ereditati dalle conversazioni parent
- i plugin di canale eseguono l'azione finale tramite il loro action adapter

Per i plugin di canale, la superficie SDK è
`ChannelMessageActionAdapter.describeMessageTool(...)`. Questa chiamata di discovery unificata consente a un plugin di restituire insieme
le sue azioni visibili, le capacità e i contributi allo schema così che queste parti non vadano fuori allineamento.

Il core passa l'ambito runtime a questo passaggio di discovery. I campi importanti includono:

- `accountId`
- `currentChannelId`
- `currentThreadTs`
- `currentMessageId`
- `sessionKey`
- `sessionId`
- `agentId`
- `requesterSenderId` attendibile in ingresso

Questo è importante per i plugin sensibili al contesto. Un canale può nascondere o esporre
azioni di messaggio in base all'account attivo, alla stanza/thread/messaggio corrente o
all'identità attendibile del richiedente, senza codificare rami specifici del canale nello
strumento `message` del core.

Per questo le modifiche di instradamento dell'embedded runner restano lavoro del plugin: il runner è
responsabile di inoltrare l'identità corrente di chat/sessione nel confine di
discovery del plugin così che lo strumento `message` condiviso esponga la corretta
superficie posseduta dal canale per il turno corrente.

Per gli helper di esecuzione posseduti dal canale, i plugin inclusi dovrebbero mantenere il runtime di esecuzione
dentro i propri moduli di estensione. Il core non possiede più i runtime di azione di messaggio di Discord,
Slack, Telegram o WhatsApp in `src/agents/tools`.
Non pubblichiamo sottopercorsi separati `plugin-sdk/*-action-runtime`, e i plugin inclusi
dovrebbero importare direttamente il proprio codice runtime locale dai propri
moduli posseduti dall'estensione.

Lo stesso confine si applica in generale ai seam SDK con nome del provider: il core non
dovrebbe importare barrel di convenienza specifici del canale per Slack, Discord, Signal,
WhatsApp o estensioni simili. Se il core ha bisogno di un comportamento, deve consumare il
barrel `api.ts` / `runtime-api.ts` del plugin incluso oppure promuovere tale esigenza
in una capacità generica ristretta nel SDK condiviso.

Per i sondaggi in particolare, ci sono due percorsi di esecuzione:

- `outbound.sendPoll` è la baseline condivisa per i canali che si adattano al modello
  comune dei sondaggi
- `actions.handleAction("poll")` è il percorso preferito per semantiche di sondaggio specifiche del canale o parametri extra dei sondaggi

Il core ora rinvia il parsing condiviso dei sondaggi finché il dispatch del sondaggio del plugin non rifiuta
l'azione, così i gestori di sondaggi posseduti dal plugin possono accettare campi di sondaggio specifici del canale senza essere bloccati
prima dal parser generico dei sondaggi.

Vedi [Load pipeline](#load-pipeline) per la sequenza completa di avvio.

## Modello di ownership delle capacità

OpenClaw tratta un plugin nativo come confine di ownership per un'**azienda** o una
**funzionalità**, non come un contenitore di integrazioni non correlate.

Questo significa:

- un plugin aziendale dovrebbe di norma possedere tutte le superfici OpenClaw rivolte a quell'azienda
- un plugin di funzionalità dovrebbe di norma possedere l'intera superficie della funzionalità che introduce
- i canali dovrebbero consumare capacità condivise del core invece di reimplementare ad hoc il comportamento del provider

Esempi:

- il plugin incluso `openai` possiede il comportamento del provider di modelli OpenAI e il comportamento OpenAI
  per sintesi vocale + voce realtime + comprensione media + generazione di immagini
- il plugin incluso `elevenlabs` possiede il comportamento di sintesi vocale ElevenLabs
- il plugin incluso `microsoft` possiede il comportamento di sintesi vocale Microsoft
- il plugin incluso `google` possiede il comportamento del provider di modelli Google più
  il comportamento Google per comprensione media + generazione di immagini + ricerca web
- il plugin incluso `firecrawl` possiede il comportamento web-fetch di Firecrawl
- i plugin inclusi `minimax`, `mistral`, `moonshot` e `zai` possiedono i loro
  backend di comprensione media
- il plugin `voice-call` è un plugin di funzionalità: possiede trasporto di chiamata, strumenti,
  CLI, route e bridging dei media-stream Twilio, ma consuma capacità condivise di sintesi vocale
  più trascrizione realtime e voce realtime invece di importare direttamente plugin dei vendor

Lo stato finale previsto è:

- OpenAI vive in un unico plugin anche se copre modelli di testo, voce, immagini e
  futuro video
- un altro vendor può fare lo stesso per la propria area funzionale
- i canali non si preoccupano di quale plugin vendor possiede il provider; consumano il
  contratto di capacità condivisa esposto dal core

Questa è la distinzione chiave:

- **plugin** = confine di ownership
- **capability** = contratto core che più plugin possono implementare o consumare

Quindi, se OpenClaw aggiunge un nuovo dominio come il video, la prima domanda non è
“quale provider dovrebbe codificare in modo rigido la gestione video?” La prima domanda è “qual è
il contratto core della capacità video?” Una volta che quel contratto esiste, i plugin vendor
possono registrarsi rispetto ad esso e i plugin di canale/funzionalità possono consumarlo.

Se la capacità non esiste ancora, la mossa corretta di solito è:

1. definire la capacità mancante nel core
2. esporla tramite l'API/runtime del plugin in modo tipizzato
3. collegare canali/funzionalità a quella capacità
4. lasciare che i plugin vendor registrino le implementazioni

Questo mantiene esplicita l'ownership evitando al tempo stesso comportamenti del core che dipendono da un
singolo vendor o da un percorso di codice specifico di un plugin una tantum.

### Stratificazione delle capacità

Usa questo modello mentale quando decidi dove deve stare il codice:

- **livello core delle capacità**: orchestrazione condivisa, policy, fallback, regole di merge della config,
  semantica di consegna e contratti tipizzati
- **livello plugin vendor**: API specifiche del vendor, autenticazione, cataloghi modelli, sintesi vocale,
  generazione di immagini, futuri backend video, endpoint di utilizzo
- **livello plugin di canale/funzionalità**: integrazioni Slack/Discord/voice-call/ecc.
  che consumano capacità core e le presentano su una superficie

Per esempio, TTS segue questa forma:

- il core possiede policy TTS al momento della risposta, ordine di fallback, preferenze e consegna al canale
- `openai`, `elevenlabs` e `microsoft` possiedono le implementazioni di sintesi
- `voice-call` consuma l'helper runtime TTS per telefonia

Lo stesso pattern dovrebbe essere preferito per le capacità future.

### Esempio di plugin aziendale multi-capacità

Un plugin aziendale dovrebbe risultare coeso dall'esterno. Se OpenClaw ha contratti condivisi
per modelli, sintesi vocale, trascrizione realtime, voce realtime, comprensione media,
generazione di immagini, generazione video, recupero web e ricerca web,
un vendor può possedere tutte le sue superfici in un unico posto:

```ts
import type { OpenClawPluginDefinition } from "openclaw/plugin-sdk/plugin-entry";
import {
  describeImageWithModel,
  transcribeOpenAiCompatibleAudio,
} from "openclaw/plugin-sdk/media-understanding";

const plugin: OpenClawPluginDefinition = {
  id: "exampleai",
  name: "ExampleAI",
  register(api) {
    api.registerProvider({
      id: "exampleai",
      // hook di auth/catalogo modelli/runtime
    });

    api.registerSpeechProvider({
      id: "exampleai",
      // config vocale del vendor — implementa direttamente l'interfaccia SpeechProviderPlugin
    });

    api.registerMediaUnderstandingProvider({
      id: "exampleai",
      capabilities: ["image", "audio", "video"],
      async describeImage(req) {
        return describeImageWithModel({
          provider: "exampleai",
          model: req.model,
          input: req.input,
        });
      },
      async transcribeAudio(req) {
        return transcribeOpenAiCompatibleAudio({
          provider: "exampleai",
          model: req.model,
          input: req.input,
        });
      },
    });

    api.registerWebSearchProvider(
      createPluginBackedWebSearchProvider({
        id: "exampleai-search",
        // logica di credenziali + fetch
      }),
    );
  },
};

export default plugin;
```

Ciò che conta non sono i nomi esatti degli helper. Conta la forma:

- un plugin possiede la superficie del vendor
- il core continua a possedere i contratti di capacità
- i canali e i plugin di funzionalità consumano helper `api.runtime.*`, non codice vendor
- i test di contratto possono verificare che il plugin abbia registrato le capacità che
  dichiara di possedere

### Esempio di capacità: comprensione video

OpenClaw tratta già la comprensione di immagini/audio/video come un'unica
capacità condivisa. Qui si applica lo stesso modello di ownership:

1. il core definisce il contratto di media-understanding
2. i plugin vendor registrano `describeImage`, `transcribeAudio` e
   `describeVideo` a seconda dei casi
3. i canali e i plugin di funzionalità consumano il comportamento condiviso del core invece di
   collegarsi direttamente al codice del vendor

Questo evita di incorporare nel core le assunzioni video di un provider. Il plugin possiede
la superficie vendor; il core possiede il contratto di capacità e il comportamento di fallback.

La generazione video segue già la stessa sequenza: il core possiede il contratto di capacità tipizzato
e l'helper runtime, e i plugin vendor registrano
implementazioni `api.registerVideoGenerationProvider(...)` rispetto ad esso.

Hai bisogno di una checklist concreta di rollout? Vedi
[Capability Cookbook](/it/plugins/architecture).

## Contratti e applicazione

La superficie API dei plugin è intenzionalmente tipizzata e centralizzata in
`OpenClawPluginApi`. Questo contratto definisce i punti di registrazione supportati e
gli helper runtime su cui un plugin può fare affidamento.

Perché è importante:

- gli autori di plugin ottengono uno standard interno stabile
- il core può rifiutare ownership duplicate, ad esempio due plugin che registrano lo stesso
  provider id
- l'avvio può mostrare diagnostica utilizzabile per registrazioni malformate
- i test di contratto possono imporre l'ownership dei plugin inclusi e prevenire derive silenziose

Ci sono due livelli di applicazione:

1. **applicazione della registrazione runtime**
   Il registry dei plugin valida le registrazioni mentre i plugin vengono caricati. Esempi:
   provider id duplicati, speech provider id duplicati e registrazioni
   malformate producono diagnostica del plugin invece di comportamento indefinito.
2. **test di contratto**
   I plugin inclusi vengono catturati nei registry di contratto durante i test così
   OpenClaw può affermare esplicitamente l'ownership. Oggi questo è usato per i model
   provider, gli speech provider, i web search provider e l'ownership della registrazione inclusa.

L'effetto pratico è che OpenClaw sa in anticipo quale plugin possiede quale
superficie. Questo consente al core e ai canali di comporsi senza attriti perché l'ownership è
dichiarata, tipizzata e verificabile invece che implicita.

### Cosa appartiene a un contratto

Buoni contratti di plugin sono:

- tipizzati
- piccoli
- specifici per capacità
- posseduti dal core
- riutilizzabili da più plugin
- consumabili da canali/funzionalità senza conoscenza del vendor

Cattivi contratti di plugin sono:

- policy specifiche del vendor nascoste nel core
- vie di fuga una tantum di un plugin che bypassano il registry
- codice di canale che entra direttamente in un'implementazione vendor
- oggetti runtime ad hoc che non fanno parte di `OpenClawPluginApi` o
  `api.runtime`

In caso di dubbio, alza il livello di astrazione: definisci prima la capacità, poi
lascia che i plugin si colleghino ad essa.

## Modello di esecuzione

I plugin OpenClaw nativi vengono eseguiti **in-process** con il Gateway. Non sono
sandboxed. Un plugin nativo caricato ha lo stesso confine di fiducia a livello di processo del
codice core.

Implicazioni:

- un plugin nativo può registrare strumenti, gestori di rete, hook e servizi
- un bug in un plugin nativo può causare crash o destabilizzare il gateway
- un plugin nativo malevolo equivale a esecuzione di codice arbitrario all'interno
  del processo OpenClaw

I bundle compatibili sono più sicuri per impostazione predefinita perché OpenClaw attualmente li tratta
come pacchetti di metadati/contenuti. Nelle versioni attuali, questo significa soprattutto
Skills incluse.

Usa allowlist e percorsi espliciti di installazione/caricamento per i plugin non inclusi. Tratta
i plugin del workspace come codice di sviluppo, non come impostazioni predefinite di produzione.

Per i nomi dei package del workspace inclusi, mantieni l'id del plugin ancorato nel nome npm:
`@openclaw/<id>` per impostazione predefinita, oppure un suffisso tipizzato approvato come
`-provider`, `-plugin`, `-speech`, `-sandbox` o `-media-understanding` quando
il package espone intenzionalmente un ruolo di plugin più ristretto.

Nota importante sulla fiducia:

- `plugins.allow` si fida degli **id dei plugin**, non della provenienza della sorgente.
- Un plugin del workspace con lo stesso id di un plugin incluso oscura intenzionalmente
  la copia inclusa quando quel plugin del workspace è abilitato/in allowlist.
- Questo è normale e utile per sviluppo locale, test di patch e hotfix.

## Confine di esportazione

OpenClaw esporta capacità, non comodità implementative.

Mantieni pubblica la registrazione delle capacità. Riduci le esportazioni helper non contrattuali:

- sottopercorsi helper specifici dei plugin inclusi
- sottopercorsi di plumbing runtime non pensati come API pubbliche
- helper di convenienza specifici del vendor
- helper di setup/onboarding che sono dettagli implementativi

Alcuni sottopercorsi helper di plugin inclusi rimangono ancora nella mappa
di esportazione SDK generata per compatibilità e manutenzione dei plugin inclusi. Esempi attuali includono
`plugin-sdk/feishu`, `plugin-sdk/feishu-setup`, `plugin-sdk/zalo`,
`plugin-sdk/zalo-setup` e vari seam `plugin-sdk/matrix*`. Trattali come
esportazioni riservate di dettaglio implementativo, non come il pattern SDK raccomandato per
nuovi plugin di terze parti.

## Pipeline di caricamento

All'avvio, OpenClaw esegue più o meno questa sequenza:

1. individua le root dei plugin candidati
2. legge manifest nativi o di bundle compatibili e metadati dei package
3. rifiuta i candidati non sicuri
4. normalizza la configurazione dei plugin (`plugins.enabled`, `allow`, `deny`, `entries`,
   `slots`, `load.paths`)
5. decide l'abilitazione per ogni candidato
6. carica i moduli nativi abilitati tramite jiti
7. chiama gli hook nativi `register(api)` (o `activate(api)` — alias legacy) e raccoglie le registrazioni nel plugin registry
8. espone il registry alle superfici di comandi/runtime

<Note>
`activate` è un alias legacy di `register` — il loader risolve quello che trova (`def.register ?? def.activate`) e lo chiama nello stesso punto. Tutti i plugin inclusi usano `register`; per i nuovi plugin preferisci `register`.
</Note>

I gate di sicurezza avvengono **prima** dell'esecuzione runtime. I candidati vengono bloccati
quando l'entry esce dalla root del plugin, il percorso è scrivibile da tutti, o l'ownership del percorso
sembra sospetta per i plugin non inclusi.

### Comportamento manifest-first

Il manifest è la fonte di verità del control plane. OpenClaw lo usa per:

- identificare il plugin
- scoprire canali/Skills/schema di configurazione dichiarati o capacità del bundle
- validare `plugins.entries.<id>.config`
- arricchire etichette/segnaposto della Control UI
- mostrare metadati di installazione/catalogo

Per i plugin nativi, il modulo runtime è la parte data plane. Registra il
comportamento effettivo come hook, strumenti, comandi o flussi provider.

### Cosa mette in cache il loader

OpenClaw mantiene brevi cache in-process per:

- risultati della discovery
- dati del registry dei manifest
- registry dei plugin caricati

Queste cache riducono i burst all'avvio e l'overhead dei comandi ripetuti. È corretto
pensarle come cache prestazionali di breve durata, non come persistenza.

Nota sulle prestazioni:

- Imposta `OPENCLAW_DISABLE_PLUGIN_DISCOVERY_CACHE=1` oppure
  `OPENCLAW_DISABLE_PLUGIN_MANIFEST_CACHE=1` per disabilitare queste cache.
- Regola le finestre di cache con `OPENCLAW_PLUGIN_DISCOVERY_CACHE_MS` e
  `OPENCLAW_PLUGIN_MANIFEST_CACHE_MS`.

## Modello del registry

I plugin caricati non mutano direttamente variabili globali casuali del core. Si registrano in un
plugin registry centrale.

Il registry tiene traccia di:

- record dei plugin (identità, sorgente, origine, stato, diagnostica)
- strumenti
- hook legacy e hook tipizzati
- canali
- provider
- gestori Gateway RPC
- route HTTP
- registrar CLI
- servizi in background
- comandi posseduti dal plugin

Le funzionalità core leggono poi da questo registry invece di parlare direttamente con i moduli dei plugin.
Questo mantiene il caricamento unidirezionale:

- modulo plugin -> registrazione nel registry
- runtime core -> consumo del registry

Questa separazione è importante per la manutenibilità. Significa che la maggior parte delle superfici core deve
avere un solo punto di integrazione: “leggi il registry”, non “gestisci casi speciali per ogni modulo plugin”.

## Callback di binding delle conversazioni

I plugin che associano una conversazione possono reagire quando un'approvazione viene risolta.

Usa `api.onConversationBindingResolved(...)` per ricevere una callback dopo che una
richiesta di binding è stata approvata o negata:

```ts
export default {
  id: "my-plugin",
  register(api) {
    api.onConversationBindingResolved(async (event) => {
      if (event.status === "approved") {
        // Ora esiste un binding per questo plugin + conversazione.
        console.log(event.binding?.conversationId);
        return;
      }

      // La richiesta è stata negata; cancella qualsiasi stato locale in sospeso.
      console.log(event.request.conversation.conversationId);
    });
  },
};
```

Campi del payload della callback:

- `status`: `"approved"` o `"denied"`
- `decision`: `"allow-once"`, `"allow-always"` o `"deny"`
- `binding`: il binding risolto per le richieste approvate
- `request`: il riepilogo della richiesta originale, suggerimento di detach, sender id e
  metadati della conversazione

Questa callback è solo di notifica. Non cambia chi è autorizzato ad associare una
conversazione, e viene eseguita dopo il completamento della gestione dell'approvazione nel core.

## Hook runtime del provider

I plugin provider ora hanno due livelli:

- metadati del manifest: `providerAuthEnvVars` per ricerca economica dell'auth env prima del
  caricamento runtime, più `providerAuthChoices` per etichette economiche di onboarding/scelta auth
  e metadati dei flag CLI prima del caricamento runtime
- hook in fase di configurazione: `catalog` / `discovery` legacy più `applyConfigDefaults`
- hook runtime: `normalizeModelId`, `normalizeTransport`,
  `normalizeConfig`,
  `applyNativeStreamingUsageCompat`, `resolveConfigApiKey`,
  `resolveSyntheticAuth`, `shouldDeferSyntheticProfileAuth`,
  `resolveDynamicModel`, `prepareDynamicModel`, `normalizeResolvedModel`,
  `contributeResolvedModelCompat`, `capabilities`,
  `normalizeToolSchemas`, `inspectToolSchemas`,
  `resolveReasoningOutputMode`, `prepareExtraParams`, `createStreamFn`,
  `wrapStreamFn`, `resolveTransportTurnState`,
  `resolveWebSocketSessionPolicy`, `formatApiKey`, `refreshOAuth`,
  `buildAuthDoctorHint`, `matchesContextOverflowError`,
  `classifyFailoverReason`, `isCacheTtlEligible`,
  `buildMissingAuthMessage`, `suppressBuiltInModel`, `augmentModelCatalog`,
  `isBinaryThinking`, `supportsXHighThinking`,
  `resolveDefaultThinkingLevel`, `isModernModelRef`, `prepareRuntimeAuth`,
  `resolveUsageAuth`, `fetchUsageSnapshot`, `createEmbeddingProvider`,
  `buildReplayPolicy`,
  `sanitizeReplayHistory`, `validateReplayTurns`, `onModelSelected`

OpenClaw continua a possedere il loop generico dell'agente, il failover, la gestione della trascrizione e
la policy sugli strumenti. Questi hook sono la superficie di estensione per il comportamento specifico del provider senza
richiedere un trasporto di inferenza completamente personalizzato.

Usa il manifest `providerAuthEnvVars` quando il provider ha credenziali basate su env
che percorsi generici di auth/status/model-picker devono vedere senza caricare il runtime del plugin.
Usa il manifest `providerAuthChoices` quando le superfici CLI di onboarding/scelta auth
devono conoscere l'id della scelta del provider, le etichette di gruppo e il semplice wiring
auth con un solo flag senza caricare il runtime del provider. Mantieni `envVars` del runtime provider
per suggerimenti rivolti agli operatori come etichette di onboarding o variabili di setup per
client-id/client-secret OAuth.

### Ordine e utilizzo degli hook

Per i plugin modello/provider, OpenClaw chiama gli hook in questo ordine approssimativo.
La colonna “Quando usarlo” è la guida rapida alla decisione.

| #   | Hook                              | Cosa fa                                                                                | Quando usarlo                                                                                                                               |
| --- | --------------------------------- | -------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `catalog`                         | Pubblica la config del provider in `models.providers` durante la generazione di `models.json` | Il provider possiede un catalogo o valori predefiniti di base URL                                                                           |
| 2   | `applyConfigDefaults`             | Applica valori predefiniti globali di config posseduti dal provider durante la materializzazione della config | I valori predefiniti dipendono dalla modalità auth, dall'env o dalla semantica della famiglia di modelli del provider                      |
| --  | _(ricerca modello built-in)_      | OpenClaw prova prima il normale percorso registry/catalog                              | _(non è un hook del plugin)_                                                                                                                |
| 3   | `normalizeModelId`                | Normalizza alias legacy o di anteprima degli ID modello prima della ricerca            | Il provider possiede la pulizia degli alias prima della risoluzione del modello canonico                                                   |
| 4   | `normalizeTransport`              | Normalizza `api` / `baseUrl` della famiglia di provider prima dell'assemblaggio generico del modello | Il provider possiede la pulizia del transport per provider id personalizzati nella stessa famiglia di transport                            |
| 5   | `normalizeConfig`                 | Normalizza `models.providers.<id>` prima della risoluzione runtime/provider            | Il provider necessita di pulizia della config che dovrebbe vivere con il plugin; gli helper Google-family inclusi supportano anche le voci supportate di config Google |
| 6   | `applyNativeStreamingUsageCompat` | Applica riscritture di compatibilità native per l'uso dello streaming ai provider della config | Il provider richiede correzioni dei metadati di uso dello streaming basate su endpoint                                                     |
| 7   | `resolveConfigApiKey`             | Risolve l'auth tramite marcatore env per i provider della config prima del caricamento dell'auth runtime | Il provider possiede la risoluzione dell'API key tramite marcatore env; `amazon-bedrock` ha anche qui un resolver built-in per AWS env-marker |
| 8   | `resolveSyntheticAuth`            | Espone auth locale/self-hosted o basata su config senza persistere plaintext           | Il provider può operare con un marcatore credenziale sintetico/locale                                                                      |
| 9   | `shouldDeferSyntheticProfileAuth` | Declassa placeholder di profilo sintetico memorizzati dietro auth basata su env/config | Il provider memorizza profili placeholder sintetici che non dovrebbero avere la precedenza                                                |
| 10  | `resolveDynamicModel`             | Fallback sincrono per ID modello posseduti dal provider non ancora presenti nel registry locale | Il provider accetta ID modello arbitrari upstream                                                                                           |
| 11  | `prepareDynamicModel`             | Warm-up asincrono, poi `resolveDynamicModel` viene eseguito di nuovo                   | Il provider ha bisogno di metadati di rete prima di risolvere ID sconosciuti                                                              |
| 12  | `normalizeResolvedModel`          | Riscrittura finale prima che l'embedded runner usi il modello risolto                  | Il provider necessita di riscritture del transport ma continua a usare un transport core                                                  |
| 13  | `contributeResolvedModelCompat`   | Contribuisce flag di compatibilità per modelli vendor dietro un altro transport compatibile | Il provider riconosce i propri modelli su transport proxy senza prendere il controllo del provider                                        |
| 14  | `capabilities`                    | Metadati di transcript/tooling posseduti dal provider usati dalla logica condivisa del core | Il provider necessita di particolarità di transcript/famiglia provider                                                                    |
| 15  | `normalizeToolSchemas`            | Normalizza gli schema degli strumenti prima che l'embedded runner li veda              | Il provider necessita di pulizia degli schema per famiglia di transport                                                                    |
| 16  | `inspectToolSchemas`              | Espone diagnostica di schema posseduta dal provider dopo la normalizzazione            | Il provider vuole warning su parole chiave senza insegnare al core regole specifiche del provider                                         |
| 17  | `resolveReasoningOutputMode`      | Seleziona il contratto di output del reasoning nativo o tagged                         | Il provider necessita di reasoning/output finale tagged invece di campi nativi                                                             |
| 18  | `prepareExtraParams`              | Normalizzazione dei parametri della richiesta prima dei wrapper generici di opzioni di stream | Il provider necessita di parametri di richiesta predefiniti o pulizia di parametri per provider                                           |
| 19  | `createStreamFn`                  | Sostituisce completamente il normale percorso di stream con un transport personalizzato | Il provider necessita di un protocollo wire personalizzato, non solo di un wrapper                                                        |
| 20  | `wrapStreamFn`                    | Wrapper dello stream dopo l'applicazione dei wrapper generici                          | Il provider necessita di wrapper di compatibilità per header/body/model della richiesta senza un transport personalizzato                 |
| 21  | `resolveTransportTurnState`       | Collega header o metadati nativi per turno del transport                               | Il provider vuole che i transport generici inviino identità di turno native del provider                                                  |
| 22  | `resolveWebSocketSessionPolicy`   | Collega header WebSocket nativi o policy di cool-down della sessione                   | Il provider vuole che i transport generici WS regolino header di sessione o policy di fallback                                            |
| 23  | `formatApiKey`                    | Formatter del profilo auth: il profilo memorizzato diventa la stringa `apiKey` runtime | Il provider memorizza metadati auth extra e necessita di una forma di token runtime personalizzata                                        |
| 24  | `refreshOAuth`                    | Override del refresh OAuth per endpoint personalizzati o policy in caso di errore di refresh | Il provider non si adatta ai refresher condivisi `pi-ai`                                                                                  |
| 25  | `buildAuthDoctorHint`             | Suggerimento di riparazione aggiunto quando il refresh OAuth fallisce                  | Il provider necessita di indicazioni di riparazione auth possedute dal provider dopo un errore di refresh                                |
| 26  | `matchesContextOverflowError`     | Matcher posseduto dal provider per overflow della context window                        | Il provider ha errori raw di overflow che le euristiche generiche non intercetterebbero                                                   |
| 27  | `classifyFailoverReason`          | Classificazione dei motivi di failover posseduta dal provider                          | Il provider può mappare errori raw API/transport in rate-limit/overload/ecc.                                                              |
| 28  | `isCacheTtlEligible`              | Policy prompt-cache per provider proxy/backhaul                                        | Il provider necessita di gating cache TTL specifico del proxy                                                                              |
| 29  | `buildMissingAuthMessage`         | Sostituzione del messaggio generico di recupero per auth mancante                      | Il provider necessita di un suggerimento di recupero auth mancante specifico del provider                                                  |
| 30  | `suppressBuiltInModel`            | Soppressione di modelli upstream obsoleti più eventuale suggerimento di errore rivolto all'utente | Il provider deve nascondere righe upstream obsolete o sostituirle con un suggerimento vendor                                              |
| 31  | `augmentModelCatalog`             | Righe sintetiche/finali del catalogo aggiunte dopo la discovery                        | Il provider necessita di righe sintetiche forward-compat in `models list` e nei picker                                                     |
| 32  | `isBinaryThinking`                | Toggle reasoning on/off per provider con thinking binario                              | Il provider espone solo thinking binario on/off                                                                                            |
| 33  | `supportsXHighThinking`           | Supporto reasoning `xhigh` per modelli selezionati                                     | Il provider vuole `xhigh` solo su un sottoinsieme di modelli                                                                               |
| 34  | `resolveDefaultThinkingLevel`     | Livello `/think` predefinito per una specifica famiglia di modelli                     | Il provider possiede la policy `/think` predefinita per una famiglia di modelli                                                            |
| 35  | `isModernModelRef`                | Matcher di modello moderno per filtri di profili live e selezione smoke                | Il provider possiede il matching del modello preferito per live/smoke                                                                      |
| 36  | `prepareRuntimeAuth`              | Scambia una credenziale configurata con il token/chiave runtime effettivo appena prima dell'inferenza | Il provider necessita di uno scambio token o di credenziali di richiesta a breve durata                                                   |
| 37  | `resolveUsageAuth`                | Risolve credenziali di usage/billing per `/usage` e superfici di stato correlate       | Il provider necessita di parsing personalizzato del token usage/quota o di una credenziale di usage diversa                               |
| 38  | `fetchUsageSnapshot`              | Recupera e normalizza snapshot di usage/quota specifici del provider dopo la risoluzione dell'auth | Il provider necessita di un endpoint o parser di payload specifico per usage                                                              |
| 39  | `createEmbeddingProvider`         | Costruisce un adapter embedding posseduto dal provider per memoria/ricerca             | Il comportamento embedding della memoria appartiene al plugin provider                                                                     |
| 40  | `buildReplayPolicy`               | Restituisce una replay policy che controlla la gestione della transcript per il provider | Il provider necessita di una policy transcript personalizzata (ad esempio stripping dei blocchi di thinking)                              |
| 41  | `sanitizeReplayHistory`           | Riscrive la cronologia di replay dopo la pulizia generica della transcript             | Il provider necessita di riscritture di replay specifiche oltre agli helper condivisi di compattazione                                   |
| 42  | `validateReplayTurns`             | Validazione o ristrutturazione finale dei turni di replay prima dell'embedded runner   | Il transport del provider richiede validazione più rigida dei turni dopo la sanitizzazione generica                                      |
| 43  | `onModelSelected`                 | Esegue effetti collaterali post-selezione posseduti dal provider                       | Il provider necessita di telemetria o stato posseduto dal provider quando un modello diventa attivo                                       |

`normalizeModelId`, `normalizeTransport` e `normalizeConfig` verificano prima il
plugin provider corrispondente, poi passano ad altri plugin provider compatibili con hook
finché uno non modifica realmente il model id o il transport/config. Questo mantiene
funzionanti gli shim alias/compat provider senza richiedere al chiamante di sapere quale
plugin incluso possiede la riscrittura. Se nessun hook provider riscrive una voce supportata
di config della famiglia Google, il normalizer di config Google incluso applica comunque
quella pulizia di compatibilità.

Se il provider necessita di un protocollo wire completamente personalizzato o di un esecutore di richieste personalizzato,
si tratta di una classe diversa di estensione. Questi hook servono per comportamento del provider
che continua a funzionare nel normale loop di inferenza di OpenClaw.

### Esempio di provider

```ts
api.registerProvider({
  id: "example-proxy",
  label: "Example Proxy",
  auth: [],
  catalog: {
    order: "simple",
    run: async (ctx) => {
      const apiKey = ctx.resolveProviderApiKey("example-proxy").apiKey;
      if (!apiKey) {
        return null;
      }
      return {
        provider: {
          baseUrl: "https://proxy.example.com/v1",
          apiKey,
          api: "openai-completions",
          models: [{ id: "auto", name: "Auto" }],
        },
      };
    },
  },
  resolveDynamicModel: (ctx) => ({
    id: ctx.modelId,
    name: ctx.modelId,
    provider: "example-proxy",
    api: "openai-completions",
    baseUrl: "https://proxy.example.com/v1",
    reasoning: false,
    input: ["text"],
    cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
    contextWindow: 128000,
    maxTokens: 8192,
  }),
  prepareRuntimeAuth: async (ctx) => {
    const exchanged = await exchangeToken(ctx.apiKey);
    return {
      apiKey: exchanged.token,
      baseUrl: exchanged.baseUrl,
      expiresAt: exchanged.expiresAt,
    };
  },
  resolveUsageAuth: async (ctx) => {
    const auth = await ctx.resolveOAuthToken();
    return auth ? { token: auth.token } : null;
  },
  fetchUsageSnapshot: async (ctx) => {
    return await fetchExampleProxyUsage(ctx.token, ctx.timeoutMs, ctx.fetchFn);
  },
});
```

### Esempi built-in

- Anthropic usa `resolveDynamicModel`, `capabilities`, `buildAuthDoctorHint`,
  `resolveUsageAuth`, `fetchUsageSnapshot`, `isCacheTtlEligible`,
  `resolveDefaultThinkingLevel`, `applyConfigDefaults`, `isModernModelRef`,
  e `wrapStreamFn` perché possiede forward-compat per Claude 4.6,
  suggerimenti per la famiglia provider, indicazioni di riparazione auth, integrazione
  dell'endpoint usage, idoneità della prompt-cache, valori predefiniti di config consapevoli dell'auth,
  policy di thinking predefinita/adattiva di Claude, e shaping dello stream specifico di Anthropic per
  beta header, `/fast` / `serviceTier` e `context1m`.
- Gli helper di stream specifici di Claude di Anthropic rimangono per ora nel seam pubblico `api.ts` / `contract-api.ts`
  del plugin incluso. La superficie di quel package
  esporta `wrapAnthropicProviderStream`, `resolveAnthropicBetas`,
  `resolveAnthropicFastMode`, `resolveAnthropicServiceTier` e i builder wrapper Anthropic di livello più basso invece di ampliare l'SDK generico intorno alle regole di beta-header di un singolo provider.
- OpenAI usa `resolveDynamicModel`, `normalizeResolvedModel` e
  `capabilities` più `buildMissingAuthMessage`, `suppressBuiltInModel`,
  `augmentModelCatalog`, `supportsXHighThinking` e `isModernModelRef`
  perché possiede forward-compat per GPT-5.4, la normalizzazione diretta OpenAI
  `openai-completions` -> `openai-responses`, suggerimenti auth consapevoli di Codex,
  soppressione di Spark, righe sintetiche della lista OpenAI e policy di thinking /
  live-model di GPT-5; la famiglia di stream `openai-responses-defaults` possiede i
  wrapper condivisi nativi di OpenAI Responses per header di attribuzione,
  `/fast`/`serviceTier`, verbosity del testo, ricerca web Codex nativa,
  shaping del payload di reasoning-compat e gestione del contesto Responses.
- OpenRouter usa `catalog` più `resolveDynamicModel` e
  `prepareDynamicModel` perché il provider è pass-through e può esporre nuovi
  model id prima dell'aggiornamento del catalogo statico di OpenClaw; usa anche
  `capabilities`, `wrapStreamFn` e `isCacheTtlEligible` per mantenere fuori dal core
  header di richiesta specifici del provider, metadati di instradamento, patch di reasoning e
  policy della prompt-cache. La sua replay policy proviene dalla famiglia
  `passthrough-gemini`, mentre la famiglia di stream `openrouter-thinking`
  possiede l'iniezione del reasoning del proxy e gli skip per modelli non supportati / `auto`.
- GitHub Copilot usa `catalog`, `auth`, `resolveDynamicModel` e
  `capabilities` più `prepareRuntimeAuth` e `fetchUsageSnapshot` perché ha
  bisogno di device login posseduto dal provider, comportamento di fallback del modello, particolarità
  della transcript Claude, scambio di token GitHub -> token Copilot e endpoint usage posseduto dal provider.
- OpenAI Codex usa `catalog`, `resolveDynamicModel`,
  `normalizeResolvedModel`, `refreshOAuth` e `augmentModelCatalog` più
  `prepareExtraParams`, `resolveUsageAuth` e `fetchUsageSnapshot` perché
  continua a funzionare sui transport core di OpenAI ma possiede la propria
  normalizzazione di transport/base URL, policy di fallback del refresh OAuth, scelta predefinita del transport,
  righe sintetiche del catalogo Codex e integrazione dell'endpoint usage di ChatGPT; condivide
  la stessa famiglia di stream `openai-responses-defaults` di OpenAI diretto.
- Google AI Studio e Gemini CLI OAuth usano `resolveDynamicModel`,
  `buildReplayPolicy`, `sanitizeReplayHistory`,
  `resolveReasoningOutputMode`, `wrapStreamFn` e `isModernModelRef` perché la
  famiglia di replay `google-gemini` possiede fallback forward-compat di Gemini 3.1,
  validazione nativa del replay Gemini, sanitizzazione del replay bootstrap, modalità
  di output reasoning tagged e matching di modelli moderni, mentre la
  famiglia di stream `google-thinking` possiede la normalizzazione del payload di thinking Gemini;
  Gemini CLI OAuth usa anche `formatApiKey`, `resolveUsageAuth` e
  `fetchUsageSnapshot` per formattazione del token, parsing del token e wiring dell'endpoint quota.
- Anthropic Vertex usa `buildReplayPolicy` tramite la
  famiglia di replay `anthropic-by-model` così che la pulizia del replay specifica di Claude resti
  limitata agli id Claude invece che a ogni transport `anthropic-messages`.
- Amazon Bedrock usa `buildReplayPolicy`, `matchesContextOverflowError`,
  `classifyFailoverReason` e `resolveDefaultThinkingLevel` perché possiede la classificazione
  specifica di Bedrock per errori di throttle/not-ready/context-overflow
  per il traffico Anthropic-on-Bedrock; la sua replay policy condivide comunque lo stesso
  guard `anthropic-by-model` solo per Claude.
- OpenRouter, Kilocode, Opencode e Opencode Go usano `buildReplayPolicy`
  tramite la famiglia di replay `passthrough-gemini` perché fanno da proxy a modelli Gemini
  attraverso transport compatibili con OpenAI e necessitano della
  sanitizzazione della thought-signature di Gemini senza validazione nativa del replay Gemini o riscritture bootstrap.
- MiniMax usa `buildReplayPolicy` tramite la
  famiglia di replay `hybrid-anthropic-openai` perché un unico provider possiede sia semantiche
  di messaggi Anthropic sia compatibili con OpenAI; mantiene la rimozione dei blocchi di thinking
  solo per Claude sul lato Anthropic, riportando però la modalità di output reasoning a quella nativa, e la famiglia di stream `minimax-fast-mode` possiede le riscritture del modello in fast-mode sul percorso di stream condiviso.
- Moonshot usa `catalog` più `wrapStreamFn` perché continua a usare il transport
  OpenAI condiviso ma necessita della normalizzazione del payload di thinking posseduta dal provider; la
  famiglia di stream `moonshot-thinking` mappa config più stato `/think` sul suo
  payload nativo di thinking binario.
- Kilocode usa `catalog`, `capabilities`, `wrapStreamFn` e
  `isCacheTtlEligible` perché necessita di header di richiesta posseduti dal provider,
  normalizzazione del payload di reasoning, suggerimenti transcript Gemini e gating Anthropic
  del cache-TTL; la famiglia di stream `kilocode-thinking` mantiene l'iniezione del Kilo thinking
  sul percorso di stream proxy condiviso saltando `kilo/auto` e altri proxy model id
  che non supportano payload di reasoning espliciti.
- Z.AI usa `resolveDynamicModel`, `prepareExtraParams`, `wrapStreamFn`,
  `isCacheTtlEligible`, `isBinaryThinking`, `isModernModelRef`,
  `resolveUsageAuth` e `fetchUsageSnapshot` perché possiede fallback GLM-5,
  valori predefiniti `tool_stream`, UX di thinking binario, matching di modelli moderni e sia auth usage sia recupero quota; la famiglia di stream `tool-stream-default-on` mantiene il wrapper `tool_stream` attivo per default fuori dal codice glue scritto a mano per provider.
- xAI usa `normalizeResolvedModel`, `normalizeTransport`,
  `contributeResolvedModelCompat`, `prepareExtraParams`, `wrapStreamFn`,
  `resolveSyntheticAuth`, `resolveDynamicModel` e `isModernModelRef`
  perché possiede normalizzazione nativa del transport xAI Responses, riscritture
  degli alias fast-mode di Grok, `tool_stream` predefinito, pulizia dei payload
  strict-tool / reasoning, riuso dell'auth di fallback per strumenti posseduti dal plugin, risoluzione
  forward-compat dei modelli Grok e patch di compatibilità possedute dal provider come profilo schema strumenti xAI, keyword di schema non supportate, `web_search` nativo e decodifica di argomenti di tool-call con entità HTML.
- Mistral, OpenCode Zen e OpenCode Go usano solo `capabilities` per mantenere
  fuori dal core le particolarità di transcript/tooling.
- I provider inclusi solo catalogo come `byteplus`, `cloudflare-ai-gateway`,
  `huggingface`, `kimi-coding`, `nvidia`, `qianfan`,
  `synthetic`, `together`, `venice`, `vercel-ai-gateway` e `volcengine` usano
  solo `catalog`.
- Qwen usa `catalog` per il suo provider testuale più le registrazioni condivise
  di media-understanding e video-generation per le sue superfici multimodali.
- MiniMax e Xiaomi usano `catalog` più hook di usage perché il loro comportamento `/usage`
  è posseduto dal plugin anche se l'inferenza continua a passare attraverso i transport condivisi.

## Helper runtime

I plugin possono accedere a helper core selezionati tramite `api.runtime`. Per TTS:

```ts
const clip = await api.runtime.tts.textToSpeech({
  text: "Hello from OpenClaw",
  cfg: api.config,
});

const result = await api.runtime.tts.textToSpeechTelephony({
  text: "Hello from OpenClaw",
  cfg: api.config,
});

const voices = await api.runtime.tts.listVoices({
  provider: "elevenlabs",
  cfg: api.config,
});
```

Note:

- `textToSpeech` restituisce il normale payload di output TTS del core per superfici file/messaggi vocali.
- Usa la configurazione core `messages.tts` e la selezione del provider.
- Restituisce buffer audio PCM + sample rate. I plugin devono effettuare resampling/encoding per i provider.
- `listVoices` è facoltativo per provider. Usalo per picker vocali o flussi di setup posseduti dal vendor.
- Gli elenchi di voci possono includere metadati più ricchi come locale, genere e tag di personalità per picker consapevoli del provider.
- Oggi OpenAI ed ElevenLabs supportano la telefonia. Microsoft no.

I plugin possono anche registrare speech provider tramite `api.registerSpeechProvider(...)`.

```ts
api.registerSpeechProvider({
  id: "acme-speech",
  label: "Acme Speech",
  isConfigured: ({ config }) => Boolean(config.messages?.tts),
  synthesize: async (req) => {
    return {
      audioBuffer: Buffer.from([]),
      outputFormat: "mp3",
      fileExtension: ".mp3",
      voiceCompatible: false,
    };
  },
});
```

Note:

- Mantieni nel core la policy TTS, il fallback e la consegna della risposta.
- Usa speech provider per il comportamento di sintesi posseduto dal vendor.
- L'input legacy Microsoft `edge` viene normalizzato nell'id provider `microsoft`.
- Il modello di ownership preferito è orientato all'azienda: un singolo plugin vendor può possedere
  provider di testo, voce, immagini e futuri media man mano che OpenClaw aggiunge questi
  contratti di capacità.

Per la comprensione di immagini/audio/video, i plugin registrano un provider tipizzato
di media-understanding invece di una generica bag chiave/valore:

```ts
api.registerMediaUnderstandingProvider({
  id: "google",
  capabilities: ["image", "audio", "video"],
  describeImage: async (req) => ({ text: "..." }),
  transcribeAudio: async (req) => ({ text: "..." }),
  describeVideo: async (req) => ({ text: "..." }),
});
```

Note:

- Mantieni nel core orchestrazione, fallback, configurazione e wiring del canale.
- Mantieni il comportamento vendor nel plugin provider.
- L'espansione additiva deve restare tipizzata: nuovi metodi facoltativi, nuovi campi risultato facoltativi, nuove capacità facoltative.
- La generazione video segue già lo stesso pattern:
  - il core possiede il contratto di capacità e l'helper runtime
  - i plugin vendor registrano `api.registerVideoGenerationProvider(...)`
  - i plugin di funzionalità/canale consumano `api.runtime.videoGeneration.*`

Per gli helper runtime di media-understanding, i plugin possono chiamare:

```ts
const image = await api.runtime.mediaUnderstanding.describeImageFile({
  filePath: "/tmp/inbound-photo.jpg",
  cfg: api.config,
  agentDir: "/tmp/agent",
});

const video = await api.runtime.mediaUnderstanding.describeVideoFile({
  filePath: "/tmp/inbound-video.mp4",
  cfg: api.config,
});
```

Per la trascrizione audio, i plugin possono usare il runtime media-understanding
o il vecchio alias STT:

```ts
const { text } = await api.runtime.mediaUnderstanding.transcribeAudioFile({
  filePath: "/tmp/inbound-audio.ogg",
  cfg: api.config,
  // Facoltativo quando il MIME non può essere dedotto in modo affidabile:
  mime: "audio/ogg",
});
```

Note:

- `api.runtime.mediaUnderstanding.*` è la superficie condivisa preferita per
  la comprensione di immagini/audio/video.
- Usa la configurazione audio core di media-understanding (`tools.media.audio`) e l'ordine di fallback del provider.
- Restituisce `{ text: undefined }` quando non viene prodotto alcun output di trascrizione (ad esempio input saltato/non supportato).
- `api.runtime.stt.transcribeAudioFile(...)` resta come alias di compatibilità.

I plugin possono anche avviare esecuzioni in background di subagent tramite `api.runtime.subagent`:

```ts
const result = await api.runtime.subagent.run({
  sessionKey: "agent:main:subagent:search-helper",
  message: "Expand this query into focused follow-up searches.",
  provider: "openai",
  model: "gpt-4.1-mini",
  deliver: false,
});
```

Note:

- `provider` e `model` sono override facoltativi per singola esecuzione, non modifiche persistenti della sessione.
- OpenClaw onora questi campi override solo per chiamanti attendibili.
- Per esecuzioni di fallback possedute dal plugin, gli operatori devono attivare esplicitamente `plugins.entries.<id>.subagent.allowModelOverride: true`.
- Usa `plugins.entries.<id>.subagent.allowedModels` per limitare i plugin attendibili a specifici target canonici `provider/model`, oppure `"*"` per consentire esplicitamente qualsiasi target.
- Le esecuzioni subagent di plugin non attendibili continuano a funzionare, ma le richieste di override vengono rifiutate invece di tornare silenziosamente al fallback.

Per la ricerca web, i plugin possono consumare l'helper runtime condiviso invece di
entrare nel wiring dello strumento dell'agente:

```ts
const providers = api.runtime.webSearch.listProviders({
  config: api.config,
});

const result = await api.runtime.webSearch.search({
  config: api.config,
  args: {
    query: "OpenClaw plugin runtime helpers",
    count: 5,
  },
});
```

I plugin possono anche registrare provider di ricerca web tramite
`api.registerWebSearchProvider(...)`.

Note:

- Mantieni nel core la selezione del provider, la risoluzione delle credenziali e la semantica condivisa della richiesta.
- Usa provider di ricerca web per transport di ricerca specifici del vendor.
- `api.runtime.webSearch.*` è la superficie condivisa preferita per i plugin di funzionalità/canale che necessitano di comportamento di ricerca senza dipendere dal wrapper dello strumento agente.

### `api.runtime.imageGeneration`

```ts
const result = await api.runtime.imageGeneration.generate({
  config: api.config,
  args: { prompt: "A friendly lobster mascot", size: "1024x1024" },
});

const providers = api.runtime.imageGeneration.listProviders({
  config: api.config,
});
```

- `generate(...)`: genera un'immagine usando la catena di provider di generazione immagini configurata.
- `listProviders(...)`: elenca i provider di generazione immagini disponibili e le loro capacità.

## Route HTTP del Gateway

I plugin possono esporre endpoint HTTP con `api.registerHttpRoute(...)`.

```ts
api.registerHttpRoute({
  path: "/acme/webhook",
  auth: "plugin",
  match: "exact",
  handler: async (_req, res) => {
    res.statusCode = 200;
    res.end("ok");
    return true;
  },
});
```

Campi della route:

- `path`: percorso della route sotto il server HTTP del gateway.
- `auth`: obbligatorio. Usa `"gateway"` per richiedere la normale auth del gateway, oppure `"plugin"` per auth/validazione webhook gestite dal plugin.
- `match`: facoltativo. `"exact"` (predefinito) o `"prefix"`.
- `replaceExisting`: facoltativo. Consente allo stesso plugin di sostituire la propria registrazione di route esistente.
- `handler`: restituisce `true` quando la route ha gestito la richiesta.

Note:

- `api.registerHttpHandler(...)` è stato rimosso e causerà un errore di caricamento del plugin. Usa invece `api.registerHttpRoute(...)`.
- Le route dei plugin devono dichiarare esplicitamente `auth`.
- I conflitti esatti `path + match` vengono rifiutati a meno che `replaceExisting: true`, e un plugin non può sostituire la route di un altro plugin.
- Route sovrapposte con livelli `auth` diversi vengono rifiutate. Mantieni catene di fallthrough `exact`/`prefix` solo sullo stesso livello auth.
- Le route `auth: "plugin"` **non** ricevono automaticamente gli scope runtime dell'operatore. Sono destinate a webhook/validazione di firma gestiti dal plugin, non a chiamate helper privilegiate del Gateway.
- Le route `auth: "gateway"` vengono eseguite dentro uno scope runtime di richiesta Gateway, ma quello scope è intenzionalmente conservativo:
  - l'auth bearer con secret condiviso (`gateway.auth.mode = "token"` / `"password"`) mantiene gli scope runtime delle route plugin fissati a `operator.write`, anche se il chiamante invia `x-openclaw-scopes`
  - le modalità HTTP attendibili con identità (ad esempio `trusted-proxy` o `gateway.auth.mode = "none"` su ingress privato) onorano `x-openclaw-scopes` solo quando l'header è esplicitamente presente
  - se `x-openclaw-scopes` è assente in quelle richieste di route plugin con identità, lo scope runtime torna a `operator.write`
- Regola pratica: non presumere che una route plugin con gateway-auth sia implicitamente una superficie admin. Se la tua route necessita di comportamento solo admin, richiedi una modalità auth con identità e documenta il contratto esplicito dell'header `x-openclaw-scopes`.

## Percorsi di importazione del Plugin SDK

Usa sottopercorsi SDK invece dell'import monolitico `openclaw/plugin-sdk` quando
scrivi plugin:

- `openclaw/plugin-sdk/plugin-entry` per le primitive di registrazione dei plugin.
- `openclaw/plugin-sdk/core` per il contratto generico condiviso rivolto ai plugin.
- `openclaw/plugin-sdk/config-schema` per l'export dello schema Zod root `openclaw.json`
  (`OpenClawSchema`).
- Primitive di canale stabili come `openclaw/plugin-sdk/channel-setup`,
  `openclaw/plugin-sdk/setup-runtime`,
  `openclaw/plugin-sdk/setup-adapter-runtime`,
  `openclaw/plugin-sdk/setup-tools`,
  `openclaw/plugin-sdk/channel-pairing`,
  `openclaw/plugin-sdk/channel-contract`,
  `openclaw/plugin-sdk/channel-feedback`,
  `openclaw/plugin-sdk/channel-inbound`,
  `openclaw/plugin-sdk/channel-lifecycle`,
  `openclaw/plugin-sdk/channel-reply-pipeline`,
  `openclaw/plugin-sdk/command-auth`,
  `openclaw/plugin-sdk/secret-input` e
  `openclaw/plugin-sdk/webhook-ingress` per wiring condiviso di setup/auth/reply/webhook.
  `channel-inbound` è la home condivisa per debounce, matching delle menzioni,
  formattazione dell'envelope e helper del contesto dell'envelope in ingresso.
  `channel-setup` è il seam ristretto per setup a installazione facoltativa.
  `setup-runtime` è la superficie runtime-safe di setup usata da `setupEntry` /
  avvio differito, inclusi gli adapter di patch di setup import-safe.
  `setup-adapter-runtime` è il seam dell'adapter di setup account consapevole dell'env.
  `setup-tools` è il piccolo seam helper per CLI/archivio/documentazione (`formatCliCommand`,
  `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`,
  `CONFIG_DIR`).
- Sottopercorsi di dominio come `openclaw/plugin-sdk/channel-config-helpers`,
  `openclaw/plugin-sdk/allow-from`,
  `openclaw/plugin-sdk/channel-config-schema`,
  `openclaw/plugin-sdk/telegram-command-config`,
  `openclaw/plugin-sdk/channel-policy`,
  `openclaw/plugin-sdk/approval-runtime`,
  `openclaw/plugin-sdk/config-runtime`,
  `openclaw/plugin-sdk/infra-runtime`,
  `openclaw/plugin-sdk/agent-runtime`,
  `openclaw/plugin-sdk/lazy-runtime`,
  `openclaw/plugin-sdk/reply-history`,
  `openclaw/plugin-sdk/routing`,
  `openclaw/plugin-sdk/status-helpers`,
  `openclaw/plugin-sdk/text-runtime`,
  `openclaw/plugin-sdk/runtime-store` e
  `openclaw/plugin-sdk/directory-runtime` per helper condivisi di runtime/config.
  `telegram-command-config` è il seam pubblico ristretto per normalizzazione/validazione
  dei comandi personalizzati Telegram e resta disponibile anche se la superficie contrattuale inclusa di Telegram è temporaneamente non disponibile.
  `text-runtime` è il seam condiviso per testo/markdown/logging, inclusi
  stripping del testo visibile all'assistente, helper di render/chunking markdown, helper di redazione, helper di directive-tag e utility di testo sicuro.
- I seam di canale specifici per approvazione dovrebbero preferire un singolo contratto `approvalCapability`
  sul plugin. Il core legge poi auth, consegna, rendering e comportamento di instradamento nativo delle approvazioni attraverso questa unica capacità invece di mescolare il comportamento di approvazione in campi plugin non correlati.
- `openclaw/plugin-sdk/channel-runtime` è deprecato e rimane solo come
  shim di compatibilità per plugin più vecchi. Il nuovo codice dovrebbe importare invece primitive generiche più ristrette, e il codice del repo non dovrebbe aggiungere nuovi import dello shim.
- Gli elementi interni delle estensioni incluse restano privati. I plugin esterni dovrebbero usare solo sottopercorsi `openclaw/plugin-sdk/*`. Il codice core/test di OpenClaw può usare i punti di ingresso pubblici del repo sotto la root di un package plugin come `index.js`, `api.js`, `runtime-api.js`, `setup-entry.js` e file a ambito ristretto come `login-qr-api.js`. Non importare mai `src/*` di un package plugin dal core o da un'altra estensione.
- Suddivisione del punto di ingresso del repo:
  `<plugin-package-root>/api.js` è il barrel helper/tipi,
  `<plugin-package-root>/runtime-api.js` è il barrel solo runtime,
  `<plugin-package-root>/index.js` è l'entry del plugin incluso,
  e `<plugin-package-root>/setup-entry.js` è l'entry del plugin di setup.
- Esempi attuali di provider inclusi:
  - Anthropic usa `api.js` / `contract-api.js` per helper di stream Claude come
    `wrapAnthropicProviderStream`, helper per beta-header e parsing di `service_tier`.
  - OpenAI usa `api.js` per builder provider, helper dei modelli predefiniti e builder provider realtime.
  - OpenRouter usa `api.js` per il proprio builder provider più helper di onboarding/config,
    mentre `register.runtime.js` può ancora riesportare helper generici
    `plugin-sdk/provider-stream` per uso locale nel repo.
- I punti di ingresso pubblici caricati tramite facade preferiscono lo snapshot della config runtime attiva
  quando esiste, poi ricorrono alla config risolta su disco quando
  OpenClaw non sta ancora servendo uno snapshot runtime.
- Le primitive generiche condivise restano il contratto SDK pubblico preferito. Esiste ancora un piccolo set di compatibilità riservato di seam helper brandizzati per canale incluso. Trattali come seam per manutenzione/compatibilità del pacchetto incluso, non come nuovi target di importazione di terze parti; i nuovi contratti cross-channel dovrebbero continuare a finire in sottopercorsi generici `plugin-sdk/*` o nei barrel locali del plugin `api.js` / `runtime-api.js`.

Nota di compatibilità:

- Evita il barrel root `openclaw/plugin-sdk` per il nuovo codice.
- Preferisci prima le primitive stabili ristrette. I sottopercorsi più recenti setup/pairing/reply/
  feedback/contract/inbound/threading/command/secret-input/webhook/infra/
  allowlist/status/message-tool sono il contratto previsto per nuovo lavoro su plugin inclusi ed esterni.
  Il parsing/matching del target appartiene a `openclaw/plugin-sdk/channel-targets`.
  I gate delle azioni di messaggio e gli helper per message-id delle reazioni appartengono a
  `openclaw/plugin-sdk/channel-actions`.
- I barrel helper specifici delle estensioni incluse non sono stabili per impostazione predefinita. Se un
  helper è necessario solo a un'estensione inclusa, mantienilo dietro il seam locale `api.js` o `runtime-api.js` dell'estensione invece di promuoverlo in `openclaw/plugin-sdk/<extension>`.
- I nuovi seam helper condivisi dovrebbero essere generici, non brandizzati per canale. Il parsing condiviso dei target appartiene a `openclaw/plugin-sdk/channel-targets`; gli elementi interni specifici del canale restano dietro il seam locale `api.js` o `runtime-api.js` del plugin proprietario.
- Sottopercorsi specifici per capacità come `image-generation`,
  `media-understanding` e `speech` esistono perché i plugin nativi/inclusi li usano oggi. La loro presenza non significa di per sé che ogni helper esportato sia un contratto esterno congelato a lungo termine.

## Schema dello strumento message

I plugin dovrebbero possedere i contributi allo schema `describeMessageTool(...)` specifici del canale.
Mantieni i campi specifici del provider nel plugin, non nel core condiviso.

Per frammenti di schema condivisi e portabili, riusa gli helper generici esportati tramite
`openclaw/plugin-sdk/channel-actions`:

- `createMessageToolButtonsSchema()` per payload in stile griglia di pulsanti
- `createMessageToolCardSchema()` per payload di card strutturate

Se una forma di schema ha senso solo per un provider, definiscila nel codice di quel plugin
invece di promuoverla nel SDK condiviso.

## Risoluzione del target del canale

I plugin di canale dovrebbero possedere la semantica del target specifica del canale. Mantieni
generico l'host outbound condiviso e usa la superficie dell'adapter di messaggistica per le regole del provider:

- `messaging.inferTargetChatType({ to })` decide se un target normalizzato
  debba essere trattato come `direct`, `group` o `channel` prima della directory lookup.
- `messaging.targetResolver.looksLikeId(raw, normalized)` dice al core se un
  input dovrebbe saltare direttamente alla risoluzione tipo-id invece della ricerca nella directory.
- `messaging.targetResolver.resolveTarget(...)` è il fallback del plugin quando
  il core necessita di una risoluzione finale posseduta dal provider dopo la normalizzazione o dopo un miss della directory.
- `messaging.resolveOutboundSessionRoute(...)` possiede la costruzione della route di sessione specifica del provider una volta risolto un target.

Suddivisione consigliata:

- Usa `inferTargetChatType` per decisioni di categoria che dovrebbero avvenire prima della
  ricerca di peer/gruppi.
- Usa `looksLikeId` per controlli del tipo “tratta questo come id target esplicito/nativo”.
- Usa `resolveTarget` per fallback di normalizzazione specifico del provider, non per
  una ricerca ampia nella directory.
- Mantieni gli id nativi del provider come chat id, thread id, JID, handle e room
  id dentro i valori `target` o nei parametri specifici del provider, non in campi generici dell'SDK.

## Directory basate sulla configurazione

I plugin che derivano voci di directory dalla configurazione dovrebbero mantenere quella logica nel
plugin e riutilizzare gli helper condivisi di
`openclaw/plugin-sdk/directory-runtime`.

Usa questo quando un canale necessita di peer/gruppi basati sulla config come:

- peer DM guidati da allowlist
- mappe canale/gruppo configurate
- fallback statici di directory con ambito account

Gli helper condivisi in `directory-runtime` gestiscono solo operazioni generiche:

- filtro delle query
- applicazione dei limiti
- helper di deduplica/normalizzazione
- costruzione di `ChannelDirectoryEntry[]`

L'ispezione account specifica del canale e la normalizzazione degli id dovrebbero restare nell'implementazione del plugin.

## Cataloghi provider

I plugin provider possono definire cataloghi di modelli per l'inferenza con
`registerProvider({ catalog: { run(...) { ... } } })`.

`catalog.run(...)` restituisce la stessa forma che OpenClaw scrive in
`models.providers`:

- `{ provider }` per una voce provider
- `{ providers }` per più voci provider

Usa `catalog` quando il plugin possiede model id specifici del provider, valori
predefiniti di base URL o metadati dei modelli condizionati dall'auth.

`catalog.order` controlla quando il catalogo di un plugin viene unito rispetto ai
provider impliciti built-in di OpenClaw:

- `simple`: provider plain con API key o guidati da env
- `profile`: provider che compaiono quando esistono profili auth
- `paired`: provider che sintetizzano più voci provider correlate
- `late`: ultimo passaggio, dopo gli altri provider impliciti

I provider successivi vincono in caso di collisione di chiave, così i plugin possono intenzionalmente
sovrascrivere una voce provider built-in con lo stesso provider id.

Compatibilità:

- `discovery` continua a funzionare come alias legacy
- se sono registrati sia `catalog` sia `discovery`, OpenClaw usa `catalog`

## Ispezione del canale in sola lettura

Se il tuo plugin registra un canale, preferisci implementare
`plugin.config.inspectAccount(cfg, accountId)` insieme a `resolveAccount(...)`.

Perché:

- `resolveAccount(...)` è il percorso runtime. Può presumere che le credenziali
  siano completamente materializzate e può fallire rapidamente quando mancano secret richiesti.
- I percorsi di comando in sola lettura come `openclaw status`, `openclaw status --all`,
  `openclaw channels status`, `openclaw channels resolve` e i flussi di repair doctor/config
  non dovrebbero aver bisogno di materializzare credenziali runtime solo per
  descrivere la configurazione.

Comportamento consigliato di `inspectAccount(...)`:

- Restituisce solo uno stato descrittivo dell'account.
- Preserva `enabled` e `configured`.
- Includi campi di sorgente/stato delle credenziali quando rilevanti, come:
  - `tokenSource`, `tokenStatus`
  - `botTokenSource`, `botTokenStatus`
  - `appTokenSource`, `appTokenStatus`
  - `signingSecretSource`, `signingSecretStatus`
- Non è necessario restituire i valori raw dei token solo per riportare la disponibilità in sola lettura. Restituire `tokenStatus: "available"` (e il campo sorgente corrispondente) è sufficiente per i comandi di stato.
- Usa `configured_unavailable` quando una credenziale è configurata tramite SecretRef ma non disponibile nel percorso di comando corrente.

Questo consente ai comandi in sola lettura di riportare “configurato ma non disponibile in questo percorso di comando” invece di andare in crash o segnalare erroneamente l'account come non configurato.

## Package pack

Una directory plugin può includere un `package.json` con `openclaw.extensions`:

```json
{
  "name": "my-pack",
  "openclaw": {
    "extensions": ["./src/safety.ts", "./src/tools.ts"],
    "setupEntry": "./src/setup-entry.ts"
  }
}
```

Ogni entry diventa un plugin. Se il pack elenca più estensioni, l'id del plugin
diventa `name/<fileBase>`.

Se il tuo plugin importa dipendenze npm, installale in quella directory così
`node_modules` sia disponibile (`npm install` / `pnpm install`).

Guardrail di sicurezza: ogni entry `openclaw.extensions` deve restare dentro la directory del plugin
dopo la risoluzione dei symlink. Le entry che escono dalla directory del package vengono
rifiutate.

Nota di sicurezza: `openclaw plugins install` installa le dipendenze del plugin con
`npm install --omit=dev --ignore-scripts` (nessuno script di lifecycle, nessuna dipendenza dev a runtime). Mantieni gli alberi di dipendenze del plugin “pure JS/TS” ed evita package che richiedono build `postinstall`.

Facoltativo: `openclaw.setupEntry` può puntare a un modulo leggero solo setup.
Quando OpenClaw necessita di superfici di setup per un plugin di canale disabilitato, oppure
quando un plugin di canale è abilitato ma ancora non configurato, carica `setupEntry`
invece dell'entry completa del plugin. Questo rende più leggeri avvio e setup
quando l'entry principale del plugin collega anche strumenti, hook o altro codice solo runtime.

Facoltativo: `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen`
può scegliere per un plugin di canale lo stesso percorso `setupEntry` durante la fase
di pre-listen del gateway, anche quando il canale è già configurato.

Usa questo solo quando `setupEntry` copre completamente la superficie di avvio che deve esistere
prima che il gateway inizi ad ascoltare. In pratica, questo significa che l'entry di setup
deve registrare ogni capacità posseduta dal canale da cui l'avvio dipende, come:

- la registrazione del canale stesso
- qualsiasi route HTTP che deve essere disponibile prima che il gateway inizi ad ascoltare
- qualsiasi metodo, strumento o servizio gateway che deve esistere durante quella stessa finestra

Se la tua entry completa possiede ancora qualche capacità di avvio richiesta, non abilitare
questo flag. Mantieni il comportamento predefinito del plugin e lascia che OpenClaw carichi
l'entry completa durante l'avvio.

I canali inclusi possono anche pubblicare helper di superficie contrattuale solo setup che il core
può consultare prima che venga caricato il runtime completo del canale. L'attuale superficie di promozione setup è:

- `singleAccountKeysToMove`
- `namedAccountPromotionKeys`
- `resolveSingleAccountPromotionTarget(...)`

Il core usa questa superficie quando deve promuovere una configurazione legacy di canale a singolo account in
`channels.<id>.accounts.*` senza caricare l'entry completa del plugin.
Matrix è l'esempio incluso attuale: sposta in un
account nominato promosso solo le chiavi auth/bootstrap quando esistono già account nominati e può preservare una chiave configurata di account predefinito non canonica invece di creare sempre `accounts.default`.

Questi setup patch adapter mantengono lazy la discovery della superficie contrattuale inclusa.
Il tempo di importazione rimane leggero; la superficie di promozione viene caricata solo al primo utilizzo invece di rientrare nell'avvio del canale incluso durante l'import del modulo.

Quando queste superfici di avvio includono metodi Gateway RPC, mantienili su un
prefisso specifico del plugin. Gli namespace admin del core (`config.*`,
`exec.approvals.*`, `wizard.*`, `update.*`) restano riservati e si risolvono sempre
in `operator.admin`, anche se un plugin richiede uno scope più ristretto.

Esempio:

```json
{
  "name": "@scope/my-channel",
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "startup": {
      "deferConfiguredChannelFullLoadUntilAfterListen": true
    }
  }
}
```

### Metadati del catalogo dei canali

I plugin di canale possono pubblicizzare metadati di setup/discovery tramite `openclaw.channel` e
suggerimenti di installazione tramite `openclaw.install`. Questo mantiene il catalogo del core privo di dati.

Esempio:

```json
{
  "name": "@openclaw/nextcloud-talk",
  "openclaw": {
    "extensions": ["./index.ts"],
    "channel": {
      "id": "nextcloud-talk",
      "label": "Nextcloud Talk",
      "selectionLabel": "Nextcloud Talk (self-hosted)",
      "docsPath": "/channels/nextcloud-talk",
      "docsLabel": "nextcloud-talk",
      "blurb": "Chat self-hosted tramite bot webhook Nextcloud Talk.",
      "order": 65,
      "aliases": ["nc-talk", "nc"]
    },
    "install": {
      "npmSpec": "@openclaw/nextcloud-talk",
      "localPath": "<bundled-plugin-local-path>",
      "defaultChoice": "npm"
    }
  }
}
```

Campi utili di `openclaw.channel` oltre all'esempio minimo:

- `detailLabel`: etichetta secondaria per superfici di catalogo/stato più ricche
- `docsLabel`: override del testo del link della documentazione
- `preferOver`: id di plugin/canale a priorità inferiore che questa voce di catalogo dovrebbe superare
- `selectionDocsPrefix`, `selectionDocsOmitLabel`, `selectionExtras`: controlli di copia per la superficie di selezione
- `markdownCapable`: contrassegna il canale come compatibile con Markdown per decisioni sulla formattazione in uscita
- `exposure.configured`: nasconde il canale dalle superfici di elenco dei canali configurati quando impostato a `false`
- `exposure.setup`: nasconde il canale dai picker interattivi di setup/configurazione quando impostato a `false`
- `exposure.docs`: contrassegna il canale come interno/privato per le superfici di navigazione della documentazione
- `showConfigured` / `showInSetup`: alias legacy ancora accettati per compatibilità; preferisci `exposure`
- `quickstartAllowFrom`: abilita per il canale il flusso standard quickstart `allowFrom`
- `forceAccountBinding`: richiede un binding esplicito dell'account anche quando esiste un solo account
- `preferSessionLookupForAnnounceTarget`: preferisce la lookup della sessione quando risolve target di announce

OpenClaw può anche unire **cataloghi di canali esterni** (ad esempio un export
di registry MPM). Inserisci un file JSON in uno dei seguenti percorsi:

- `~/.openclaw/mpm/plugins.json`
- `~/.openclaw/mpm/catalog.json`
- `~/.openclaw/plugins/catalog.json`

Oppure punta `OPENCLAW_PLUGIN_CATALOG_PATHS` (o `OPENCLAW_MPM_CATALOG_PATHS`) a
uno o più file JSON (delimitati da virgole/punto e virgola/`PATH`). Ogni file dovrebbe
contenere `{ "entries": [ { "name": "@scope/pkg", "openclaw": { "channel": {...}, "install": {...} } } ] }`. Il parser accetta anche `"packages"` o `"plugins"` come alias legacy della chiave `"entries"`.

## Plugin del motore di contesto

I plugin del motore di contesto possiedono l'orchestrazione del contesto di sessione per ingest, assemblaggio
e compattazione. Registrali dal tuo plugin con
`api.registerContextEngine(id, factory)`, quindi seleziona il motore attivo con
`plugins.slots.contextEngine`.

Usa questo quando il tuo plugin deve sostituire o estendere la pipeline di contesto
predefinita invece di aggiungere solo ricerca in memoria o hook.

```ts
export default function (api) {
  api.registerContextEngine("lossless-claw", () => ({
    info: { id: "lossless-claw", name: "Lossless Claw", ownsCompaction: true },
    async ingest() {
      return { ingested: true };
    },
    async assemble({ messages }) {
      return { messages, estimatedTokens: 0 };
    },
    async compact() {
      return { ok: true, compacted: false };
    },
  }));
}
```

Se il tuo motore **non** possiede l'algoritmo di compattazione, mantieni `compact()`
implementato e delegalo esplicitamente:

```ts
import { delegateCompactionToRuntime } from "openclaw/plugin-sdk/core";

export default function (api) {
  api.registerContextEngine("my-memory-engine", () => ({
    info: {
      id: "my-memory-engine",
      name: "My Memory Engine",
      ownsCompaction: false,
    },
    async ingest() {
      return { ingested: true };
    },
    async assemble({ messages }) {
      return { messages, estimatedTokens: 0 };
    },
    async compact(params) {
      return await delegateCompactionToRuntime(params);
    },
  }));
}
```

## Aggiungere una nuova capacità

Quando un plugin necessita di un comportamento che non si adatta all'API corrente, non bypassare
il sistema dei plugin con un accesso privato. Aggiungi la capacità mancante.

Sequenza consigliata:

1. definire il contratto core
   Decidi quale comportamento condiviso il core dovrebbe possedere: policy, fallback, merge della config,
   ciclo di vita, semantica rivolta al canale e forma dell'helper runtime.
2. aggiungere superfici tipizzate di registrazione/runtime del plugin
   Estendi `OpenClawPluginApi` e/o `api.runtime` con la più piccola
   superficie di capacità tipizzata utile.
3. collegare i consumer core + canale/funzionalità
   I canali e i plugin di funzionalità dovrebbero consumare la nuova capacità tramite il core,
   non importando direttamente un'implementazione vendor.
4. registrare implementazioni vendor
   I plugin vendor registrano poi i loro backend rispetto alla capacità.
5. aggiungere copertura di contratto
   Aggiungi test così ownership e forma della registrazione restino esplicite nel tempo.

È così che OpenClaw resta opinato senza diventare rigidamente legato alla visione del mondo di un
singolo provider. Vedi il [Capability Cookbook](/it/plugins/architecture)
per una checklist concreta dei file e un esempio completo.

### Checklist delle capacità

Quando aggiungi una nuova capacità, l'implementazione di solito dovrebbe toccare insieme
queste superfici:

- tipi del contratto core in `src/<capability>/types.ts`
- runner/helper runtime del core in `src/<capability>/runtime.ts`
- superficie di registrazione dell'API plugin in `src/plugins/types.ts`
- wiring del plugin registry in `src/plugins/registry.ts`
- esposizione runtime del plugin in `src/plugins/runtime/*` quando i plugin di funzionalità/canale devono consumarla
- helper di capture/test in `src/test-utils/plugin-registration.ts`
- asserzioni di ownership/contratto in `src/plugins/contracts/registry.ts`
- documentazione per operatori/plugin in `docs/`

Se una di queste superfici manca, di solito è un segnale che la capacità non è
ancora completamente integrata.

### Template di capacità

Pattern minimo:

```ts
// contratto core
export type VideoGenerationProviderPlugin = {
  id: string;
  label: string;
  generateVideo: (req: VideoGenerationRequest) => Promise<VideoGenerationResult>;
};

// API plugin
api.registerVideoGenerationProvider({
  id: "openai",
  label: "OpenAI",
  async generateVideo(req) {
    return await generateOpenAiVideo(req);
  },
});

// helper runtime condiviso per plugin di funzionalità/canale
const clip = await api.runtime.videoGeneration.generate({
  prompt: "Show the robot walking through the lab.",
  cfg,
});
```

Pattern di test del contratto:

```ts
expect(findVideoGenerationProviderIdsForPlugin("openai")).toEqual(["openai"]);
```

Questo mantiene semplice la regola:

- il core possiede il contratto di capacità + l'orchestrazione
- i plugin vendor possiedono le implementazioni vendor
- i plugin di funzionalità/canale consumano helper runtime
- i test di contratto mantengono esplicita l'ownership
