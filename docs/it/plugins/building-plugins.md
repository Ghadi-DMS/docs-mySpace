---
read_when:
    - Vuoi creare un nuovo plugin OpenClaw
    - Hai bisogno di una guida rapida per lo sviluppo di plugin
    - Stai aggiungendo un nuovo canale, provider, strumento o un'altra capability a OpenClaw
sidebarTitle: Getting Started
summary: Crea il tuo primo plugin OpenClaw in pochi minuti
title: Creazione di plugin
x-i18n:
    generated_at: "2026-04-06T03:08:54Z"
    model: gpt-5.4
    provider: openai
    source_hash: 9be344cb300ecbcba08e593a95bcc93ab16c14b28a0ff0c29b26b79d8249146c
    source_path: plugins/building-plugins.md
    workflow: 15
---

# Creazione di plugin

I plugin estendono OpenClaw con nuove capability: canali, provider di modelli,
speech, trascrizione realtime, voce realtime, comprensione dei media, generazione
di immagini, generazione di video, web fetch, web search, strumenti agente, o
qualsiasi combinazione.

Non è necessario aggiungere il tuo plugin al repository OpenClaw. Pubblicalo su
[ClawHub](/it/tools/clawhub) o npm e gli utenti lo installano con
`openclaw plugins install <package-name>`. OpenClaw prova prima ClawHub e
ripiega automaticamente su npm.

## Prerequisiti

- Node >= 22 e un gestore di pacchetti (npm o pnpm)
- Familiarità con TypeScript (ESM)
- Per i plugin nel repository: repository clonato e `pnpm install` eseguito

## Che tipo di plugin?

<CardGroup cols={3}>
  <Card title="Plugin canale" icon="messages-square" href="/it/plugins/sdk-channel-plugins">
    Collega OpenClaw a una piattaforma di messaggistica (Discord, IRC, ecc.)
  </Card>
  <Card title="Plugin provider" icon="cpu" href="/it/plugins/sdk-provider-plugins">
    Aggiungi un provider di modelli (LLM, proxy o endpoint personalizzato)
  </Card>
  <Card title="Plugin tool / hook" icon="wrench">
    Registra strumenti agente, hook di eventi o servizi — continua sotto
  </Card>
</CardGroup>

Se un plugin canale è facoltativo e potrebbe non essere installato quando viene
eseguito onboarding/setup, usa `createOptionalChannelSetupSurface(...)` da
`openclaw/plugin-sdk/channel-setup`. Produce una coppia adattatore di setup + wizard
che pubblicizza il requisito di installazione e fallisce in modalità fail-closed sulle scritture reali della configurazione
finché il plugin non è installato.

## Avvio rapido: plugin tool

Questa guida crea un plugin minimo che registra uno strumento agente. I plugin
canale e provider hanno guide dedicate collegate sopra.

<Steps>
  <Step title="Crea il pacchetto e il manifesto">
    <CodeGroup>
    ```json package.json
    {
      "name": "@myorg/openclaw-my-plugin",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "compat": {
          "pluginApi": ">=2026.3.24-beta.2",
          "minGatewayVersion": "2026.3.24-beta.2"
        },
        "build": {
          "openclawVersion": "2026.3.24-beta.2",
          "pluginSdkVersion": "2026.3.24-beta.2"
        }
      }
    }
    ```

    ```json openclaw.plugin.json
    {
      "id": "my-plugin",
      "name": "My Plugin",
      "description": "Adds a custom tool to OpenClaw",
      "configSchema": {
        "type": "object",
        "additionalProperties": false
      }
    }
    ```
    </CodeGroup>

    Ogni plugin ha bisogno di un manifesto, anche senza configurazione. Vedi
    [Manifest](/it/plugins/manifest) per lo schema completo. Gli snippet canonici per la pubblicazione su ClawHub
    si trovano in `docs/snippets/plugin-publish/`.

  </Step>

  <Step title="Scrivi il punto di ingresso">

    ```typescript
    // index.ts
    import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
    import { Type } from "@sinclair/typebox";

    export default definePluginEntry({
      id: "my-plugin",
      name: "My Plugin",
      description: "Adds a custom tool to OpenClaw",
      register(api) {
        api.registerTool({
          name: "my_tool",
          description: "Do a thing",
          parameters: Type.Object({ input: Type.String() }),
          async execute(_id, params) {
            return { content: [{ type: "text", text: `Got: ${params.input}` }] };
          },
        });
      },
    });
    ```

    `definePluginEntry` è per i plugin non-canale. Per i canali, usa
    `defineChannelPluginEntry` — vedi [Plugin canale](/it/plugins/sdk-channel-plugins).
    Per le opzioni complete del punto di ingresso, vedi [Punti di ingresso](/it/plugins/sdk-entrypoints).

  </Step>

  <Step title="Testa e pubblica">

    **Plugin esterni:** convalida e pubblica con ClawHub, poi installa:

    ```bash
    clawhub package publish your-org/your-plugin --dry-run
    clawhub package publish your-org/your-plugin
    openclaw plugins install clawhub:@myorg/openclaw-my-plugin
    ```

    OpenClaw controlla anche ClawHub prima di npm per specifiche di pacchetto bare come
    `@myorg/openclaw-my-plugin`.

    **Plugin nel repository:** inseriscili nell'albero workspace dei plugin bundled — vengono rilevati automaticamente.

    ```bash
    pnpm test -- <bundled-plugin-root>/my-plugin/
    ```

  </Step>
</Steps>

## Capability dei plugin

Un singolo plugin può registrare qualsiasi numero di capability tramite l'oggetto `api`:

| Capability             | Metodo di registrazione                         | Guida dettagliata                                                               |
| ---------------------- | ----------------------------------------------- | ------------------------------------------------------------------------------- |
| Inferenza testo (LLM)  | `api.registerProvider(...)`                     | [Plugin provider](/it/plugins/sdk-provider-plugins)                                |
| Canale / messaggistica | `api.registerChannel(...)`                      | [Plugin canale](/it/plugins/sdk-channel-plugins)                                   |
| Speech (TTS/STT)       | `api.registerSpeechProvider(...)`               | [Plugin provider](/it/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| Trascrizione realtime  | `api.registerRealtimeTranscriptionProvider(...)` | [Plugin provider](/it/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| Voce realtime          | `api.registerRealtimeVoiceProvider(...)`        | [Plugin provider](/it/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| Comprensione dei media | `api.registerMediaUnderstandingProvider(...)`   | [Plugin provider](/it/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| Generazione di immagini | `api.registerImageGenerationProvider(...)`     | [Plugin provider](/it/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| Generazione musicale   | `api.registerMusicGenerationProvider(...)`      | [Plugin provider](/it/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| Generazione di video   | `api.registerVideoGenerationProvider(...)`      | [Plugin provider](/it/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| Web fetch              | `api.registerWebFetchProvider(...)`             | [Plugin provider](/it/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| Web search             | `api.registerWebSearchProvider(...)`            | [Plugin provider](/it/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| Strumenti agente       | `api.registerTool(...)`                         | Sotto                                                                           |
| Comandi personalizzati | `api.registerCommand(...)`                      | [Punti di ingresso](/it/plugins/sdk-entrypoints)                                   |
| Hook di eventi         | `api.registerHook(...)`                         | [Punti di ingresso](/it/plugins/sdk-entrypoints)                                   |
| Route HTTP             | `api.registerHttpRoute(...)`                    | [Interni](/it/plugins/architecture#gateway-http-routes)                            |
| Sottocomandi CLI       | `api.registerCli(...)`                          | [Punti di ingresso](/it/plugins/sdk-entrypoints)                                   |

Per l'API completa di registrazione, vedi [Panoramica SDK](/it/plugins/sdk-overview#registration-api).

Se il tuo plugin registra metodi RPC gateway personalizzati, mantienili su un
prefisso specifico del plugin. Gli spazi dei nomi di amministrazione core (`config.*`,
`exec.approvals.*`, `wizard.*`, `update.*`) restano riservati e vengono sempre risolti a
`operator.admin`, anche se un plugin richiede uno scope più ristretto.

Semantica dei guard hook da tenere presente:

- `before_tool_call`: `{ block: true }` è terminale e ferma gli handler a priorità inferiore.
- `before_tool_call`: `{ block: false }` viene trattato come nessuna decisione.
- `before_tool_call`: `{ requireApproval: true }` mette in pausa l'esecuzione dell'agente e chiede all'utente l'approvazione tramite overlay di approvazione exec, pulsanti Telegram, interazioni Discord o il comando `/approve` su qualsiasi canale.
- `before_install`: `{ block: true }` è terminale e ferma gli handler a priorità inferiore.
- `before_install`: `{ block: false }` viene trattato come nessuna decisione.
- `message_sending`: `{ cancel: true }` è terminale e ferma gli handler a priorità inferiore.
- `message_sending`: `{ cancel: false }` viene trattato come nessuna decisione.

Il comando `/approve` gestisce sia le approvazioni exec sia quelle dei plugin con fallback limitato: quando un ID di approvazione exec non viene trovato, OpenClaw riprova lo stesso ID tramite le approvazioni del plugin. L'inoltro delle approvazioni del plugin può essere configurato in modo indipendente tramite `approvals.plugin` nella configurazione.

Se il plumbing di approvazione personalizzato deve rilevare quel medesimo caso di fallback limitato,
preferisci `isApprovalNotFoundError` da `openclaw/plugin-sdk/error-runtime`
invece di confrontare manualmente le stringhe di scadenza dell'approvazione.

Vedi [Semantica delle decisioni degli hook nella panoramica SDK](/it/plugins/sdk-overview#hook-decision-semantics) per i dettagli.

## Registrazione degli strumenti agente

Gli strumenti sono funzioni tipizzate che l'LLM può chiamare. Possono essere obbligatori (sempre
disponibili) o facoltativi (opt-in dell'utente):

```typescript
register(api) {
  // Required tool — always available
  api.registerTool({
    name: "my_tool",
    description: "Do a thing",
    parameters: Type.Object({ input: Type.String() }),
    async execute(_id, params) {
      return { content: [{ type: "text", text: params.input }] };
    },
  });

  // Optional tool — user must add to allowlist
  api.registerTool(
    {
      name: "workflow_tool",
      description: "Run a workflow",
      parameters: Type.Object({ pipeline: Type.String() }),
      async execute(_id, params) {
        return { content: [{ type: "text", text: params.pipeline }] };
      },
    },
    { optional: true },
  );
}
```

Gli utenti abilitano gli strumenti facoltativi nella configurazione:

```json5
{
  tools: { allow: ["workflow_tool"] },
}
```

- I nomi degli strumenti non devono entrare in conflitto con gli strumenti core (i conflitti vengono saltati)
- Usa `optional: true` per strumenti con effetti collaterali o requisiti binari aggiuntivi
- Gli utenti possono abilitare tutti gli strumenti di un plugin aggiungendo l'ID del plugin a `tools.allow`

## Convenzioni di importazione

Importa sempre da percorsi mirati `openclaw/plugin-sdk/<subpath>`:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";

// Wrong: monolithic root (deprecated, will be removed)
import { ... } from "openclaw/plugin-sdk";
```

Per il riferimento completo dei sottopercorsi, vedi [Panoramica SDK](/it/plugins/sdk-overview).

All'interno del tuo plugin, usa file barrel locali (`api.ts`, `runtime-api.ts`) per
le importazioni interne — non importare mai il tuo stesso plugin tramite il suo percorso SDK.

Per i plugin provider, mantieni gli helper specifici del provider in quei barrel
alla root del pacchetto, a meno che il punto di estensione non sia davvero generico. Esempi bundled attuali:

- Anthropic: wrapper di stream Claude e helper `service_tier` / beta
- OpenAI: builder provider, helper per i modelli predefiniti, provider realtime
- OpenRouter: builder provider più helper di onboarding/configurazione

Se un helper è utile solo all'interno di un singolo pacchetto provider bundled, mantienilo su quel
punto di estensione alla root del pacchetto invece di promuoverlo in `openclaw/plugin-sdk/*`.

Esistono ancora alcuni punti di estensione helper generati `openclaw/plugin-sdk/<bundled-id>` per
manutenzione e compatibilità dei plugin bundled, per esempio
`plugin-sdk/feishu-setup` o `plugin-sdk/zalo-setup`. Trattali come superfici
riservate, non come il modello predefinito per nuovi plugin di terze parti.

## Checklist pre-invio

<Check>**package.json** ha i metadati `openclaw` corretti</Check>
<Check>Il manifesto **openclaw.plugin.json** è presente e valido</Check>
<Check>Il punto di ingresso usa `defineChannelPluginEntry` o `definePluginEntry`</Check>
<Check>Tutte le importazioni usano percorsi mirati `plugin-sdk/<subpath>`</Check>
<Check>Le importazioni interne usano moduli locali, non auto-importazioni SDK</Check>
<Check>I test passano (`pnpm test -- <bundled-plugin-root>/my-plugin/`)</Check>
<Check>`pnpm check` passa (plugin nel repository)</Check>

## Test delle beta release

1. Tieni d'occhio i tag di release GitHub su [openclaw/openclaw](https://github.com/openclaw/openclaw/releases) e iscriviti tramite `Watch` > `Releases`. I tag beta hanno un aspetto simile a `v2026.3.N-beta.1`. Puoi anche attivare le notifiche per l'account X ufficiale di OpenClaw [@openclaw](https://x.com/openclaw) per gli annunci di release.
2. Testa il tuo plugin contro il tag beta non appena appare. La finestra prima della stable è in genere di poche ore.
3. Pubblica nel thread del tuo plugin nel canale Discord `plugin-forum` dopo il test con `all good` oppure indicando cosa si è rotto. Se non hai ancora un thread, creane uno.
4. Se qualcosa si rompe, apri o aggiorna un issue intitolato `Beta blocker: <plugin-name> - <summary>` e applica l'etichetta `beta-blocker`. Inserisci il link dell'issue nel tuo thread.
5. Apri una PR verso `main` intitolata `fix(<plugin-id>): beta blocker - <summary>` e collega l'issue sia nella PR sia nel tuo thread Discord. I contributor non possono etichettare le PR, quindi il titolo è il segnale lato PR per maintainer e automazione. I blocker con una PR vengono uniti; i blocker senza PR potrebbero comunque essere rilasciati. I maintainer monitorano questi thread durante i test beta.
6. Il silenzio significa verde. Se perdi la finestra, probabilmente la tua correzione entrerà nel ciclo successivo.

## Passaggi successivi

<CardGroup cols={2}>
  <Card title="Plugin canale" icon="messages-square" href="/it/plugins/sdk-channel-plugins">
    Crea un plugin per canali di messaggistica
  </Card>
  <Card title="Plugin provider" icon="cpu" href="/it/plugins/sdk-provider-plugins">
    Crea un plugin provider di modelli
  </Card>
  <Card title="Panoramica SDK" icon="book-open" href="/it/plugins/sdk-overview">
    Riferimento della mappa di importazione e dell'API di registrazione
  </Card>
  <Card title="Helper runtime" icon="settings" href="/it/plugins/sdk-runtime">
    TTS, search, subagent tramite api.runtime
  </Card>
  <Card title="Testing" icon="test-tubes" href="/it/plugins/sdk-testing">
    Utilità e pattern di test
  </Card>
  <Card title="Manifesto plugin" icon="file-json" href="/it/plugins/manifest">
    Riferimento completo dello schema del manifesto
  </Card>
</CardGroup>

## Correlati

- [Architettura dei plugin](/it/plugins/architecture) — approfondimento sull'architettura interna
- [Panoramica SDK](/it/plugins/sdk-overview) — riferimento del Plugin SDK
- [Manifest](/it/plugins/manifest) — formato del manifesto plugin
- [Plugin canale](/it/plugins/sdk-channel-plugins) — creazione di plugin canale
- [Plugin provider](/it/plugins/sdk-provider-plugins) — creazione di plugin provider
