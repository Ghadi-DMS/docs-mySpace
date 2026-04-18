---
read_when:
    - Stai creando un nuovo Plugin di canale di messaggistica
    - Vuoi collegare OpenClaw a una piattaforma di messaggistica
    - Devi comprendere la superficie dell'adattatore ChannelPlugin
sidebarTitle: Channel Plugins
summary: Guida passo passo per creare un Plugin di canale di messaggistica per OpenClaw
title: Creazione di Plugin di canale
x-i18n:
    generated_at: "2026-04-18T08:05:10Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3dda53c969bc7356a450c2a5bf49fb82bf1283c23e301dec832d8724b11e724b
    source_path: plugins/sdk-channel-plugins.md
    workflow: 15
---

# Creazione di Plugin di canale

Questa guida illustra come creare un Plugin di canale che collega OpenClaw a una
piattaforma di messaggistica. Alla fine avrai un canale funzionante con sicurezza
dei DM, pairing, threading delle risposte e messaggistica in uscita.

<Info>
  Se non hai mai creato prima alcun Plugin OpenClaw, leggi prima
  [Getting Started](/it/plugins/building-plugins) per la struttura di base del
  pacchetto e la configurazione del manifest.
</Info>

## Come funzionano i Plugin di canale

I Plugin di canale non hanno bisogno di strumenti propri per inviare/modificare/reagire. OpenClaw mantiene un unico strumento `message` condiviso nel core. Il tuo Plugin gestisce:

- **Config** — risoluzione dell'account e procedura guidata di configurazione
- **Security** — policy DM e allowlist
- **Pairing** — flusso di approvazione DM
- **Session grammar** — come gli id di conversazione specifici del provider vengono mappati a chat di base, id di thread e fallback dei parent
- **Outbound** — invio di testo, media e sondaggi alla piattaforma
- **Threading** — come vengono inserite nei thread le risposte

Il core gestisce lo strumento message condiviso, il wiring del prompt, la forma
esterna della chiave di sessione, la gestione generica di `:thread:` e il dispatch.

Se il tuo canale aggiunge parametri dello strumento message che trasportano sorgenti media, esponi quei
nomi di parametro tramite `describeMessageTool(...).mediaSourceParams`. Il core usa
quell'elenco esplicito per la normalizzazione dei percorsi sandbox e la policy
di accesso ai media in uscita, quindi i Plugin non hanno bisogno di casi speciali
nel core condiviso per parametri specifici del provider come avatar, allegati o immagini di copertina.
Preferisci restituire una mappa indicizzata per azione come
`{ "set-profile": ["avatarUrl", "avatarPath"] }` così azioni non correlate non
ereditano gli argomenti media di un'altra azione. Un array piatto continua a funzionare per i parametri che
sono intenzionalmente condivisi tra tutte le azioni esposte.

Se la tua piattaforma memorizza scope aggiuntivo dentro gli id di conversazione, mantieni quel parsing
nel Plugin con `messaging.resolveSessionConversation(...)`. Questo è l'hook
canonico per mappare `rawId` all'id di conversazione di base, all'eventuale thread
id, a `baseConversationId` esplicito e a eventuali `parentConversationCandidates`.
Quando restituisci `parentConversationCandidates`, mantienili ordinati dal
parent più specifico a quello più ampio/conversazione di base.

I Plugin bundled che hanno bisogno dello stesso parsing prima dell'avvio del registro
dei canali possono anche esporre un file `session-key-api.ts` di primo livello con un'esportazione
`resolveSessionConversation(...)` corrispondente. Il core usa questa superficie
sicura per il bootstrap solo quando il registro runtime dei Plugin non è ancora disponibile.

`messaging.resolveParentConversationCandidates(...)` resta disponibile come fallback di compatibilità legacy quando un Plugin ha bisogno solo di fallback dei parent sopra
l'id generico/raw. Se esistono entrambi gli hook, il core usa prima
`resolveSessionConversation(...).parentConversationCandidates` e ricorre a
`resolveParentConversationCandidates(...)` solo quando l'hook canonico li
omette.

## Approvazioni e capacità del canale

La maggior parte dei Plugin di canale non ha bisogno di codice specifico per le approvazioni.

- Il core gestisce `/approve` nella stessa chat, i payload condivisi dei pulsanti di approvazione e la consegna generica di fallback.
- Preferisci un unico oggetto `approvalCapability` sul Plugin di canale quando il canale richiede un comportamento specifico per le approvazioni.
- `ChannelPlugin.approvals` è stato rimosso. Inserisci fatti di consegna/native/render/auth relativi all'approvazione in `approvalCapability`.
- `plugin.auth` è solo per login/logout; il core non legge più hook auth di approvazione da quell'oggetto.
- `approvalCapability.authorizeActorAction` e `approvalCapability.getActionAvailabilityState` sono la seam canonica per l'auth delle approvazioni.
- Usa `approvalCapability.getActionAvailabilityState` per la disponibilità auth delle approvazioni nella stessa chat.
- Se il tuo canale espone approvazioni exec native, usa `approvalCapability.getExecInitiatingSurfaceState` per lo stato della superficie di avvio/client nativo quando differisce dall'auth di approvazione nella stessa chat. Il core usa quell'hook specifico per exec per distinguere `enabled` da `disabled`, decidere se il canale di avvio supporta approvazioni exec native e includere il canale nelle indicazioni di fallback del client nativo. `createApproverRestrictedNativeApprovalCapability(...)` lo compila per il caso comune.
- Usa `outbound.shouldSuppressLocalPayloadPrompt` o `outbound.beforeDeliverPayload` per il comportamento del ciclo di vita del payload specifico del canale, come nascondere prompt locali di approvazione duplicati o inviare indicatori di digitazione prima della consegna.
- Usa `approvalCapability.delivery` solo per il routing nativo delle approvazioni o la soppressione del fallback.
- Usa `approvalCapability.nativeRuntime` per i fatti di approvazione nativa posseduti dal canale. Mantienilo lazy sugli entrypoint hot del canale con `createLazyChannelApprovalNativeRuntimeAdapter(...)`, che può importare il tuo modulo runtime on demand pur permettendo al core di assemblare il ciclo di vita dell'approvazione.
- Usa `approvalCapability.render` solo quando un canale ha davvero bisogno di payload di approvazione personalizzati invece del renderer condiviso.
- Usa `approvalCapability.describeExecApprovalSetup` quando il canale vuole che la risposta del percorso disabilitato spieghi esattamente quali manopole di config servono per abilitare approvazioni exec native. L'hook riceve `{ channel, channelLabel, accountId }`; i canali con account nominati dovrebbero renderizzare percorsi con scope di account come `channels.<channel>.accounts.<id>.execApprovals.*` invece di valori predefiniti di primo livello.
- Se un canale può dedurre identità DM stabili simili al proprietario dalla config esistente, usa `createResolvedApproverActionAuthAdapter` da `openclaw/plugin-sdk/approval-runtime` per limitare `/approve` nella stessa chat senza aggiungere logica core specifica per l'approvazione.
- Se un canale ha bisogno della consegna di approvazioni native, mantieni il codice del canale focalizzato sulla normalizzazione della destinazione più i fatti di trasporto/presentazione. Usa `createChannelExecApprovalProfile`, `createChannelNativeOriginTargetResolver`, `createChannelApproverDmTargetResolver` e `createApproverRestrictedNativeApprovalCapability` da `openclaw/plugin-sdk/approval-runtime`. Inserisci i fatti specifici del canale dietro `approvalCapability.nativeRuntime`, idealmente tramite `createChannelApprovalNativeRuntimeAdapter(...)` o `createLazyChannelApprovalNativeRuntimeAdapter(...)`, così il core può assemblare l'handler e gestire filtraggio delle richieste, routing, deduplica, scadenza, sottoscrizione Gateway e avvisi di instradamento altrove. `nativeRuntime` è suddiviso in alcune seam più piccole:
- `availability` — se l'account è configurato e se una richiesta deve essere gestita
- `presentation` — mappa il view model di approvazione condiviso in payload nativi pending/resolved/expired o azioni finali
- `transport` — prepara le destinazioni più invio/aggiornamento/eliminazione dei messaggi di approvazione nativi
- `interactions` — hook opzionali bind/unbind/clear-action per pulsanti o reazioni native
- `observe` — hook opzionali per diagnostica della consegna
- Se il canale ha bisogno di oggetti posseduti dal runtime come un client, token, app Bolt o ricevitore Webhook, registrali tramite `openclaw/plugin-sdk/channel-runtime-context`. Il registro generico del contesto runtime permette al core di eseguire il bootstrap di handler guidati dalle capacità dallo stato di avvio del canale senza aggiungere glue wrapper specifico per l'approvazione.
- Ricorri a `createChannelApprovalHandler` o `createChannelNativeApprovalRuntime` di livello inferiore solo quando la seam guidata dalle capacità non è ancora sufficientemente espressiva.
- I canali di approvazione nativa devono instradare sia `accountId` sia `approvalKind` tramite questi helper. `accountId` mantiene la policy di approvazione multi-account nello scope del giusto account bot, e `approvalKind` mantiene disponibile al canale il comportamento di approvazione exec vs Plugin senza branch hardcoded nel core.
- Il core ora gestisce anche gli avvisi di reinstradamento delle approvazioni. I Plugin di canale non dovrebbero inviare propri messaggi di follow-up del tipo "l'approvazione è andata nei DM / in un altro canale" da `createChannelNativeApprovalRuntime`; invece, esponi un routing accurato origin + DM dell'approvatore tramite gli helper condivisi della capacità di approvazione e lascia che il core aggreghi le consegne effettive prima di pubblicare qualunque avviso nella chat di avvio.
- Preserva end-to-end il tipo di id di approvazione consegnato. I client nativi non dovrebbero
  indovinare o riscrivere il routing di approvazione exec vs Plugin dallo stato locale del canale.
- Diversi tipi di approvazione possono intenzionalmente esporre superfici native differenti.
  Esempi bundled attuali:
  - Slack mantiene disponibile il routing di approvazione nativa sia per gli id exec sia per quelli Plugin.
  - Matrix mantiene lo stesso routing DM/canale nativo e la stessa UX a reazioni per approvazioni exec
    e Plugin, continuando comunque a permettere che l'auth differisca in base al tipo di approvazione.
- `createApproverRestrictedNativeApprovalAdapter` esiste ancora come wrapper di compatibilità, ma il nuovo codice dovrebbe preferire il builder di capacità ed esporre `approvalCapability` sul Plugin.

Per gli entrypoint hot del canale, preferisci i sottopercorsi runtime più ristretti quando ti serve solo
una parte di quella famiglia:

- `openclaw/plugin-sdk/approval-auth-runtime`
- `openclaw/plugin-sdk/approval-client-runtime`
- `openclaw/plugin-sdk/approval-delivery-runtime`
- `openclaw/plugin-sdk/approval-gateway-runtime`
- `openclaw/plugin-sdk/approval-handler-adapter-runtime`
- `openclaw/plugin-sdk/approval-handler-runtime`
- `openclaw/plugin-sdk/approval-native-runtime`
- `openclaw/plugin-sdk/approval-reply-runtime`
- `openclaw/plugin-sdk/channel-runtime-context`

Allo stesso modo, preferisci `openclaw/plugin-sdk/setup-runtime`,
`openclaw/plugin-sdk/setup-adapter-runtime`,
`openclaw/plugin-sdk/reply-runtime`,
`openclaw/plugin-sdk/reply-dispatch-runtime`,
`openclaw/plugin-sdk/reply-reference` e
`openclaw/plugin-sdk/reply-chunking` quando non hai bisogno della superficie
ombrello più ampia.

Per la configurazione in particolare:

- `openclaw/plugin-sdk/setup-runtime` copre gli helper di setup sicuri per il runtime:
  adapter di patch setup import-safe (`createPatchedAccountSetupAdapter`,
  `createEnvPatchedAccountSetupAdapter`,
  `createSetupInputPresenceValidator`), output delle note di lookup,
  `promptResolvedAllowFrom`, `splitSetupEntries` e i builder
  delegated setup-proxy
- `openclaw/plugin-sdk/setup-adapter-runtime` è la seam ristretta di adapter env-aware
  per `createEnvPatchedAccountSetupAdapter`
- `openclaw/plugin-sdk/channel-setup` copre i builder di setup a installazione opzionale
  più alcune primitive setup-safe:
  `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`,

Se il tuo canale supporta setup o auth guidati da env e i flussi generici di avvio/config
devono conoscere quei nomi env prima del caricamento del runtime, dichiarali nel
manifest del Plugin con `channelEnvVars`. Mantieni `envVars` del runtime del canale o costanti locali
solo per testo rivolto agli operatori.
`createOptionalChannelSetupWizard`, `DEFAULT_ACCOUNT_ID`,
`createTopLevelChannelDmPolicy`, `setSetupChannelEnabled` e
`splitSetupEntries`

- usa la seam più ampia `openclaw/plugin-sdk/setup` solo quando hai bisogno anche degli
  helper condivisi di setup/config più pesanti, come
  `moveSingleAccountChannelSectionToDefaultAccount(...)`

Se il tuo canale vuole solo pubblicizzare "installa prima questo Plugin" nelle
superfici di setup, preferisci `createOptionalChannelSetupSurface(...)`. L'adapter/la procedura guidata generati falliscono in modo chiuso sulle scritture di config e sulla finalizzazione, e riutilizzano
lo stesso messaggio "installazione richiesta" in validazione, finalize e testo del link
alla documentazione.

Per altri percorsi hot del canale, preferisci gli helper ristretti alle superfici
legacy più ampie:

- `openclaw/plugin-sdk/account-core`,
  `openclaw/plugin-sdk/account-id`,
  `openclaw/plugin-sdk/account-resolution` e
  `openclaw/plugin-sdk/account-helpers` per config multi-account e
  fallback dell'account predefinito
- `openclaw/plugin-sdk/inbound-envelope` e
  `openclaw/plugin-sdk/inbound-reply-dispatch` per route/envelope inbound e
  wiring di record-and-dispatch
- `openclaw/plugin-sdk/messaging-targets` per parsing/matching della destinazione
- `openclaw/plugin-sdk/outbound-media` e
  `openclaw/plugin-sdk/outbound-runtime` per caricamento dei media più delegati
  di identità/invio outbound
- `openclaw/plugin-sdk/thread-bindings-runtime` per il ciclo di vita dei thread-binding
  e la registrazione degli adapter
- `openclaw/plugin-sdk/agent-media-payload` solo quando è ancora richiesto un layout di campi legacy agent/media
  payload
- `openclaw/plugin-sdk/telegram-command-config` per normalizzazione dei comandi personalizzati di Telegram, validazione di duplicati/conflitti e un contratto di config dei comandi stabile in fallback

I canali solo auth di solito possono fermarsi al percorso predefinito: il core gestisce le approvazioni e il Plugin espone solo le capacità outbound/auth. I canali di approvazione nativa come Matrix, Slack, Telegram e trasporti chat personalizzati dovrebbero usare gli helper nativi condivisi invece di implementare da soli il proprio ciclo di vita delle approvazioni.

## Policy per le menzioni in entrata

Mantieni la gestione delle menzioni in entrata suddivisa in due livelli:

- raccolta delle evidenze gestita dal Plugin
- valutazione della policy condivisa

Usa `openclaw/plugin-sdk/channel-mention-gating` per le decisioni sulla policy delle menzioni.
Usa `openclaw/plugin-sdk/channel-inbound` solo quando ti serve il barrel di helper
inbound più ampio.

Adatto alla logica locale del Plugin:

- rilevamento della risposta al bot
- rilevamento della citazione del bot
- controlli di partecipazione al thread
- esclusioni di messaggi di servizio/sistema
- cache native della piattaforma necessarie per dimostrare la partecipazione del bot

Adatto all'helper condiviso:

- `requireMention`
- risultato di menzione esplicita
- allowlist di menzione implicita
- bypass dei comandi
- decisione finale di salto

Flusso consigliato:

1. Calcola i fatti locali sulle menzioni.
2. Passa questi fatti a `resolveInboundMentionDecision({ facts, policy })`.
3. Usa `decision.effectiveWasMentioned`, `decision.shouldBypassMention` e `decision.shouldSkip` nel tuo gate inbound.

```typescript
import {
  implicitMentionKindWhen,
  matchesMentionWithExplicit,
  resolveInboundMentionDecision,
} from "openclaw/plugin-sdk/channel-inbound";

const mentionMatch = matchesMentionWithExplicit(text, {
  mentionRegexes,
  mentionPatterns,
});

const facts = {
  canDetectMention: true,
  wasMentioned: mentionMatch.matched,
  hasAnyMention: mentionMatch.hasExplicitMention,
  implicitMentionKinds: [
    ...implicitMentionKindWhen("reply_to_bot", isReplyToBot),
    ...implicitMentionKindWhen("quoted_bot", isQuoteOfBot),
  ],
};

const decision = resolveInboundMentionDecision({
  facts,
  policy: {
    isGroup,
    requireMention,
    allowedImplicitMentionKinds: requireExplicitMention ? [] : ["reply_to_bot", "quoted_bot"],
    allowTextCommands,
    hasControlCommand,
    commandAuthorized,
  },
});

if (decision.shouldSkip) return;
```

`api.runtime.channel.mentions` espone gli stessi helper condivisi per le menzioni per
i Plugin di canale bundled che dipendono già dall'iniezione runtime:

- `buildMentionRegexes`
- `matchesMentionPatterns`
- `matchesMentionWithExplicit`
- `implicitMentionKindWhen`
- `resolveInboundMentionDecision`

Se ti servono solo `implicitMentionKindWhen` e
`resolveInboundMentionDecision`, importa da
`openclaw/plugin-sdk/channel-mention-gating` per evitare di caricare helper
runtime inbound non correlati.

I vecchi helper `resolveMentionGating*` restano su
`openclaw/plugin-sdk/channel-inbound` solo come esportazioni di compatibilità. Il nuovo codice
dovrebbe usare `resolveInboundMentionDecision({ facts, policy })`.

## Procedura dettagliata

<Steps>
  <a id="step-1-package-and-manifest"></a>
  <Step title="Pacchetto e manifest">
    Crea i file standard del Plugin. Il campo `channel` in `package.json` è
    ciò che rende questo un Plugin di canale. Per la superficie completa dei
    metadati del pacchetto, vedi [Plugin Setup and Config](/it/plugins/sdk-setup#openclaw-channel):

    <CodeGroup>
    ```json package.json
    {
      "name": "@myorg/openclaw-acme-chat",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "setupEntry": "./setup-entry.ts",
        "channel": {
          "id": "acme-chat",
          "label": "Acme Chat",
          "blurb": "Connect OpenClaw to Acme Chat."
        }
      }
    }
    ```

    ```json openclaw.plugin.json
    {
      "id": "acme-chat",
      "kind": "channel",
      "channels": ["acme-chat"],
      "name": "Acme Chat",
      "description": "Acme Chat channel plugin",
      "configSchema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "acme-chat": {
            "type": "object",
            "properties": {
              "token": { "type": "string" },
              "allowFrom": {
                "type": "array",
                "items": { "type": "string" }
              }
            }
          }
        }
      }
    }
    ```
    </CodeGroup>

  </Step>

  <Step title="Crea l'oggetto Plugin di canale">
    L'interfaccia `ChannelPlugin` ha molte superfici di adattatore opzionali. Inizia con
    il minimo indispensabile — `id` e `setup` — e aggiungi adattatori secondo necessità.

    Crea `src/channel.ts`:

    ```typescript src/channel.ts
    import {
      createChatChannelPlugin,
      createChannelPluginBase,
    } from "openclaw/plugin-sdk/channel-core";
    import type { OpenClawConfig } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatApi } from "./client.js"; // your platform API client

    type ResolvedAccount = {
      accountId: string | null;
      token: string;
      allowFrom: string[];
      dmPolicy: string | undefined;
    };

    function resolveAccount(
      cfg: OpenClawConfig,
      accountId?: string | null,
    ): ResolvedAccount {
      const section = (cfg.channels as Record<string, any>)?.["acme-chat"];
      const token = section?.token;
      if (!token) throw new Error("acme-chat: token is required");
      return {
        accountId: accountId ?? null,
        token,
        allowFrom: section?.allowFrom ?? [],
        dmPolicy: section?.dmSecurity,
      };
    }

    export const acmeChatPlugin = createChatChannelPlugin<ResolvedAccount>({
      base: createChannelPluginBase({
        id: "acme-chat",
        setup: {
          resolveAccount,
          inspectAccount(cfg, accountId) {
            const section =
              (cfg.channels as Record<string, any>)?.["acme-chat"];
            return {
              enabled: Boolean(section?.token),
              configured: Boolean(section?.token),
              tokenStatus: section?.token ? "available" : "missing",
            };
          },
        },
      }),

      // DM security: who can message the bot
      security: {
        dm: {
          channelKey: "acme-chat",
          resolvePolicy: (account) => account.dmPolicy,
          resolveAllowFrom: (account) => account.allowFrom,
          defaultPolicy: "allowlist",
        },
      },

      // Pairing: approval flow for new DM contacts
      pairing: {
        text: {
          idLabel: "Acme Chat username",
          message: "Send this code to verify your identity:",
          notify: async ({ target, code }) => {
            await acmeChatApi.sendDm(target, `Pairing code: ${code}`);
          },
        },
      },

      // Threading: how replies are delivered
      threading: { topLevelReplyToMode: "reply" },

      // Outbound: send messages to the platform
      outbound: {
        attachedResults: {
          sendText: async (params) => {
            const result = await acmeChatApi.sendMessage(
              params.to,
              params.text,
            );
            return { messageId: result.id };
          },
        },
        base: {
          sendMedia: async (params) => {
            await acmeChatApi.sendFile(params.to, params.filePath);
          },
        },
      },
    });
    ```

    <Accordion title="Che cosa fa per te createChatChannelPlugin">
      Invece di implementare manualmente interfacce di adattatore di basso livello, passi
      opzioni dichiarative e il builder le compone:

      | Option | What it wires |
      | --- | --- |
      | `security.dm` | Resolver di sicurezza DM con scope dai campi di config |
      | `pairing.text` | Flusso di pairing DM basato su testo con scambio di codice |
      | `threading` | Resolver della modalità reply-to (fissa, con scope di account o personalizzata) |
      | `outbound.attachedResults` | Funzioni di invio che restituiscono metadati del risultato (ID dei messaggi) |

      Puoi anche passare oggetti di adattatore raw invece delle opzioni dichiarative
      se hai bisogno di pieno controllo.
    </Accordion>

  </Step>

  <Step title="Collega l'entry point">
    Crea `index.ts`:

    ```typescript index.ts
    import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineChannelPluginEntry({
      id: "acme-chat",
      name: "Acme Chat",
      description: "Acme Chat channel plugin",
      plugin: acmeChatPlugin,
      registerCliMetadata(api) {
        api.registerCli(
          ({ program }) => {
            program
              .command("acme-chat")
              .description("Acme Chat management");
          },
          {
            descriptors: [
              {
                name: "acme-chat",
                description: "Acme Chat management",
                hasSubcommands: false,
              },
            ],
          },
        );
      },
      registerFull(api) {
        api.registerGatewayMethod(/* ... */);
      },
    });
    ```

    Inserisci i descrittori CLI posseduti dal canale in `registerCliMetadata(...)` così OpenClaw
    può mostrarli nell'help root senza attivare il runtime completo del canale,
    mentre i normali caricamenti completi continuano a rilevare gli stessi descrittori per la registrazione
    reale dei comandi. Mantieni `registerFull(...)` per il lavoro solo runtime.
    Se `registerFull(...)` registra metodi RPC Gateway, usa un
    prefisso specifico del Plugin. I namespace admin del core (`config.*`,
    `exec.approvals.*`, `wizard.*`, `update.*`) restano riservati e si
    risolvono sempre a `operator.admin`.
    `defineChannelPluginEntry` gestisce automaticamente la suddivisione della modalità di registrazione. Vedi
    [Entry Points](/it/plugins/sdk-entrypoints#definechannelpluginentry) per tutte le
    opzioni.

  </Step>

  <Step title="Aggiungi un'entry di setup">
    Crea `setup-entry.ts` per un caricamento leggero durante l'onboarding:

    ```typescript setup-entry.ts
    import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineSetupPluginEntry(acmeChatPlugin);
    ```

    OpenClaw carica questo invece dell'entry completa quando il canale è disabilitato
    o non configurato. Evita di trascinare codice runtime pesante durante i flussi di setup.
    Vedi [Setup and Config](/it/plugins/sdk-setup#setup-entry) per i dettagli.

    I canali workspace bundled che suddividono le esportazioni sicure per il setup in moduli
    sidecar possono usare `defineBundledChannelSetupEntry(...)` da
    `openclaw/plugin-sdk/channel-entry-contract` quando hanno bisogno anche di un
    setter runtime esplicito al momento del setup.

  </Step>

  <Step title="Gestisci i messaggi in entrata">
    Il tuo Plugin deve ricevere i messaggi dalla piattaforma e inoltrarli a
    OpenClaw. Il pattern tipico è un Webhook che verifica la richiesta e
    la inoltra attraverso l'handler inbound del tuo canale:

    ```typescript
    registerFull(api) {
      api.registerHttpRoute({
        path: "/acme-chat/webhook",
        auth: "plugin", // plugin-managed auth (verify signatures yourself)
        handler: async (req, res) => {
          const event = parseWebhookPayload(req);

          // Your inbound handler dispatches the message to OpenClaw.
          // The exact wiring depends on your platform SDK —
          // see a real example in the bundled Microsoft Teams or Google Chat plugin package.
          await handleAcmeChatInbound(api, event);

          res.statusCode = 200;
          res.end("ok");
          return true;
        },
      });
    }
    ```

    <Note>
      La gestione dei messaggi in entrata è specifica del canale. Ogni Plugin di canale gestisce
      la propria pipeline inbound. Guarda i Plugin di canale bundled
      (per esempio il pacchetto Plugin Microsoft Teams o Google Chat) per pattern reali.
    </Note>

  </Step>

<a id="step-6-test"></a>
<Step title="Test">
Scrivi test colocati in `src/channel.test.ts`:

    ```typescript src/channel.test.ts
    import { describe, it, expect } from "vitest";
    import { acmeChatPlugin } from "./channel.js";

    describe("acme-chat plugin", () => {
      it("resolves account from config", () => {
        const cfg = {
          channels: {
            "acme-chat": { token: "test-token", allowFrom: ["user1"] },
          },
        } as any;
        const account = acmeChatPlugin.setup!.resolveAccount(cfg, undefined);
        expect(account.token).toBe("test-token");
      });

      it("inspects account without materializing secrets", () => {
        const cfg = {
          channels: { "acme-chat": { token: "test-token" } },
        } as any;
        const result = acmeChatPlugin.setup!.inspectAccount!(cfg, undefined);
        expect(result.configured).toBe(true);
        expect(result.tokenStatus).toBe("available");
      });

      it("reports missing config", () => {
        const cfg = { channels: {} } as any;
        const result = acmeChatPlugin.setup!.inspectAccount!(cfg, undefined);
        expect(result.configured).toBe(false);
      });
    });
    ```

    ```bash
    pnpm test -- <bundled-plugin-root>/acme-chat/
    ```

    Per gli helper di test condivisi, vedi [Testing](/it/plugins/sdk-testing).

  </Step>
</Steps>

## Struttura dei file

```
<bundled-plugin-root>/acme-chat/
├── package.json              # metadati openclaw.channel
├── openclaw.plugin.json      # Manifest con schema di config
├── index.ts                  # defineChannelPluginEntry
├── setup-entry.ts            # defineSetupPluginEntry
├── api.ts                    # Esportazioni pubbliche (opzionale)
├── runtime-api.ts            # Esportazioni runtime interne (opzionale)
└── src/
    ├── channel.ts            # ChannelPlugin tramite createChatChannelPlugin
    ├── channel.test.ts       # Test
    ├── client.ts             # client API della piattaforma
    └── runtime.ts            # store runtime (se necessario)
```

## Argomenti avanzati

<CardGroup cols={2}>
  <Card title="Opzioni di threading" icon="git-branch" href="/it/plugins/sdk-entrypoints#registration-mode">
    Modalità di risposta fisse, con scope di account o personalizzate
  </Card>
  <Card title="Integrazione dello strumento message" icon="puzzle" href="/it/plugins/architecture#channel-plugins-and-the-shared-message-tool">
    describeMessageTool e discovery delle azioni
  </Card>
  <Card title="Risoluzione della destinazione" icon="crosshair" href="/it/plugins/architecture#channel-target-resolution">
    inferTargetChatType, looksLikeId, resolveTarget
  </Card>
  <Card title="Helper runtime" icon="settings" href="/it/plugins/sdk-runtime">
    TTS, STT, media, subagent tramite api.runtime
  </Card>
</CardGroup>

<Note>
Alcune seam helper bundled esistono ancora per la manutenzione dei Plugin bundled e la
compatibilità. Non sono il pattern consigliato per i nuovi Plugin di canale;
preferisci i sottopercorsi generici channel/setup/reply/runtime dalla superficie
SDK comune, a meno che tu non stia mantenendo direttamente quella famiglia di Plugin bundled.
</Note>

## Passaggi successivi

- [Provider Plugins](/it/plugins/sdk-provider-plugins) — se il tuo Plugin fornisce anche modelli
- [SDK Overview](/it/plugins/sdk-overview) — riferimento completo agli import dei sottopercorsi
- [SDK Testing](/it/plugins/sdk-testing) — utility di test e test di contratto
- [Plugin Manifest](/it/plugins/manifest) — schema completo del manifest
