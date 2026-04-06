---
read_when:
    - Stai creando un nuovo plugin di canale di messaggistica
    - Vuoi collegare OpenClaw a una piattaforma di messaggistica
    - Hai bisogno di capire la superficie dell'adattatore ChannelPlugin
sidebarTitle: Channel Plugins
summary: Guida passo passo per creare un plugin di canale di messaggistica per OpenClaw
title: Creazione di plugin di canale
x-i18n:
    generated_at: "2026-04-06T03:10:09Z"
    model: gpt-5.4
    provider: openai
    source_hash: 66b52c10945a8243d803af3bf7e1ea0051869ee92eda2af5718d9bb24fbb8552
    source_path: plugins/sdk-channel-plugins.md
    workflow: 15
---

# Creazione di plugin di canale

Questa guida ti accompagna nella creazione di un plugin di canale che collega OpenClaw a una
piattaforma di messaggistica. Alla fine avrai un canale funzionante con sicurezza DM,
abbinamento, threading delle risposte e messaggistica in uscita.

<Info>
  Se non hai mai creato un plugin OpenClaw prima d'ora, leggi prima
  [Getting Started](/it/plugins/building-plugins) per la struttura di base del
  pacchetto e la configurazione del manifest.
</Info>

## Come funzionano i plugin di canale

I plugin di canale non hanno bisogno dei propri strumenti per inviare/modificare/reagire. OpenClaw mantiene un
unico strumento `message` condiviso nel core. Il tuo plugin gestisce:

- **Configurazione** — risoluzione dell'account e procedura guidata di configurazione
- **Sicurezza** — policy DM e allowlist
- **Abbinamento** — flusso di approvazione DM
- **Grammatica della sessione** — come gli id di conversazione specifici del provider vengono mappati a chat di base, id di thread e fallback parent
- **Uscita** — invio di testo, media e sondaggi alla piattaforma
- **Threading** — come vengono organizzate le risposte

Il core gestisce lo strumento message condiviso, il wiring del prompt, la forma esterna della chiave di sessione,
la gestione generica di `:thread:` e il dispatch.

Se la tua piattaforma memorizza uno scope aggiuntivo negli id di conversazione, mantieni quel parsing
nel plugin con `messaging.resolveSessionConversation(...)`. Questo è l'hook
canonico per mappare `rawId` all'id di conversazione di base, all'eventuale id di thread,
all'esplicito `baseConversationId` e a qualsiasi `parentConversationCandidates`.
Quando restituisci `parentConversationCandidates`, mantienili ordinati dal
parent più specifico alla conversazione più ampia/di base.

I plugin bundled che necessitano dello stesso parsing prima che il registro dei canali si avvii
possono anche esporre un file `session-key-api.ts` di primo livello con una
export `resolveSessionConversation(...)` corrispondente. Il core usa questa superficie
sicura per il bootstrap solo quando il registro runtime dei plugin non è ancora disponibile.

`messaging.resolveParentConversationCandidates(...)` resta disponibile come
fallback legacy di compatibilità quando un plugin ha bisogno solo di fallback parent
sopra l'id generico/raw. Se entrambi gli hook esistono, il core usa
prima `resolveSessionConversation(...).parentConversationCandidates` e passa a
`resolveParentConversationCandidates(...)` solo quando l'hook canonico
li omette.

## Approvazioni e capability del canale

La maggior parte dei plugin di canale non necessita di codice specifico per le approvazioni.

- Il core gestisce `/approve` nella stessa chat, i payload condivisi dei pulsanti di approvazione e il recapito generico di fallback.
- Preferisci un singolo oggetto `approvalCapability` nel plugin di canale quando il canale necessita di comportamento specifico per le approvazioni.
- `approvalCapability.authorizeActorAction` e `approvalCapability.getActionAvailabilityState` sono la seam canonica per l'autorizzazione delle approvazioni.
- Se il tuo canale espone approvazioni exec native, implementa `approvalCapability.getActionAvailabilityState` anche quando il transport nativo vive interamente sotto `approvalCapability.native`. Il core usa quell'hook di disponibilità per distinguere `enabled` da `disabled`, decidere se il canale che ha avviato l'azione supporta approvazioni native e includere il canale nella guida di fallback per i client nativi.
- Usa `outbound.shouldSuppressLocalPayloadPrompt` o `outbound.beforeDeliverPayload` per comportamenti specifici del canale nel ciclo di vita del payload, come nascondere prompt locali di approvazione duplicati o inviare indicatori di digitazione prima del recapito.
- Usa `approvalCapability.delivery` solo per instradamento di approvazioni native o soppressione del fallback.
- Usa `approvalCapability.render` solo quando un canale ha davvero bisogno di payload di approvazione personalizzati invece del renderer condiviso.
- Usa `approvalCapability.describeExecApprovalSetup` quando il canale vuole che la risposta del percorso disabilitato spieghi gli esatti knob di configurazione necessari per abilitare approvazioni exec native. L'hook riceve `{ channel, channelLabel, accountId }`; i canali con account nominali dovrebbero renderizzare percorsi con scope per account come `channels.<channel>.accounts.<id>.execApprovals.*` invece dei valori predefiniti di primo livello.
- Se un canale può dedurre identità DM stabili simili a owner dalla configurazione esistente, usa `createResolvedApproverActionAuthAdapter` da `openclaw/plugin-sdk/approval-runtime` per limitare `/approve` nella stessa chat senza aggiungere logica core specifica per le approvazioni.
- Se un canale ha bisogno del recapito nativo delle approvazioni, mantieni il codice del canale focalizzato sulla normalizzazione del target e sugli hook di transport. Usa `createChannelExecApprovalProfile`, `createChannelNativeOriginTargetResolver`, `createChannelApproverDmTargetResolver`, `createApproverRestrictedNativeApprovalCapability` e `createChannelNativeApprovalRuntime` da `openclaw/plugin-sdk/approval-runtime` così il core gestisce filtro delle richieste, instradamento, deduplica, scadenza e sottoscrizione gateway.
- I canali con approvazioni native devono instradare sia `accountId` sia `approvalKind` tramite questi helper. `accountId` mantiene la policy di approvazione multi-account limitata al corretto account bot, e `approvalKind` mantiene disponibile al canale il comportamento di approvazione exec vs plugin senza branch hardcoded nel core.
- Preserva end-to-end il tipo di id di approvazione recapitato. I client nativi non dovrebbero
  dedurre o riscrivere l'instradamento delle approvazioni exec vs plugin dallo stato locale del canale.
- Diversi tipi di approvazione possono esporre intenzionalmente superfici native diverse.
  Esempi bundled attuali:
  - Slack mantiene disponibile l'instradamento nativo delle approvazioni sia per gli id exec sia per quelli plugin.
  - Matrix mantiene l'instradamento DM/canale nativo solo per le approvazioni exec e lascia
    le approvazioni plugin sul percorso condiviso `/approve` nella stessa chat.
- `createApproverRestrictedNativeApprovalAdapter` esiste ancora come wrapper di compatibilità, ma il nuovo codice dovrebbe preferire il builder di capability ed esporre `approvalCapability` nel plugin.

Per entrypoint di canale sensibili al tempo di caricamento, preferisci i sottopercorsi runtime più stretti quando ti
serve solo una parte di quella famiglia:

- `openclaw/plugin-sdk/approval-auth-runtime`
- `openclaw/plugin-sdk/approval-client-runtime`
- `openclaw/plugin-sdk/approval-delivery-runtime`
- `openclaw/plugin-sdk/approval-native-runtime`
- `openclaw/plugin-sdk/approval-reply-runtime`

Allo stesso modo, preferisci `openclaw/plugin-sdk/setup-runtime`,
`openclaw/plugin-sdk/setup-adapter-runtime`,
`openclaw/plugin-sdk/reply-runtime`,
`openclaw/plugin-sdk/reply-dispatch-runtime`,
`openclaw/plugin-sdk/reply-reference` e
`openclaw/plugin-sdk/reply-chunking` quando non ti serve la più ampia
superficie umbrella.

Per la configurazione in particolare:

- `openclaw/plugin-sdk/setup-runtime` copre gli helper di configurazione sicuri per il runtime:
  adattatori patch per setup sicuri all'import (`createPatchedAccountSetupAdapter`,
  `createEnvPatchedAccountSetupAdapter`,
  `createSetupInputPresenceValidator`), output delle note di lookup,
  `promptResolvedAllowFrom`, `splitSetupEntries` e i builder
  del setup-proxy delegato
- `openclaw/plugin-sdk/setup-adapter-runtime` è la seam stretta dell'adattatore
  consapevole dell'env per `createEnvPatchedAccountSetupAdapter`
- `openclaw/plugin-sdk/channel-setup` copre i builder di configurazione per installazione facoltativa
  più alcuni primitivi sicuri per la configurazione:
  `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`,
  `createOptionalChannelSetupWizard`, `DEFAULT_ACCOUNT_ID`,
  `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled` e
  `splitSetupEntries`
- usa la seam più ampia `openclaw/plugin-sdk/setup` solo quando ti servono anche gli
  helper condivisi più pesanti per configurazione/setup come
  `moveSingleAccountChannelSectionToDefaultAccount(...)`

Se il tuo canale vuole solo pubblicizzare "installa prima questo plugin" nelle superfici di configurazione,
preferisci `createOptionalChannelSetupSurface(...)`. L'adattatore/wizard generato
fallisce in modo chiuso sulle scritture di configurazione e sulla finalizzazione, e riutilizza
lo stesso messaggio di installazione richiesta in validazione, finalize e testo con link alla documentazione.

Per altri percorsi di canale sensibili al caricamento, preferisci gli helper stretti alle superfici legacy più ampie:

- `openclaw/plugin-sdk/account-core`,
  `openclaw/plugin-sdk/account-id`,
  `openclaw/plugin-sdk/account-resolution` e
  `openclaw/plugin-sdk/account-helpers` per configurazione multi-account e
  fallback dell'account predefinito
- `openclaw/plugin-sdk/inbound-envelope` e
  `openclaw/plugin-sdk/inbound-reply-dispatch` per instradamento/envelope inbound e
  wiring di record-and-dispatch
- `openclaw/plugin-sdk/messaging-targets` per parsing/matching del target
- `openclaw/plugin-sdk/outbound-media` e
  `openclaw/plugin-sdk/outbound-runtime` per caricamento dei media più delegati di identità/invio outbound
- `openclaw/plugin-sdk/thread-bindings-runtime` per il ciclo di vita del thread-binding
  e la registrazione dell'adattatore
- `openclaw/plugin-sdk/agent-media-payload` solo quando è ancora richiesto un layout legacy dei campi del payload agent/media
- `openclaw/plugin-sdk/telegram-command-config` per normalizzazione dei comandi personalizzati di Telegram, validazione di duplicati/conflitti e un contratto di configurazione dei comandi stabile rispetto al fallback

I canali solo-auth di solito possono fermarsi al percorso predefinito: il core gestisce le approvazioni e il plugin espone solo capability outbound/auth. I canali con approvazioni native come Matrix, Slack, Telegram e transport chat personalizzati dovrebbero usare gli helper nativi condivisi invece di implementare autonomamente il ciclo di vita delle approvazioni.

## Procedura guidata

<Steps>
  <a id="step-1-package-and-manifest"></a>
  <Step title="Pacchetto e manifest">
    Crea i file standard del plugin. Il campo `channel` in `package.json` è
    ciò che rende questo un plugin di canale. Per la superficie completa dei metadati del pacchetto,
    vedi [Plugin Setup and Config](/it/plugins/sdk-setup#openclawchannel):

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

  <Step title="Crea l'oggetto plugin di canale">
    L'interfaccia `ChannelPlugin` ha molte superfici adattatore facoltative. Inizia con
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

    <Accordion title="Cosa fa per te createChatChannelPlugin">
      Invece di implementare manualmente interfacce adattatore di basso livello, passi
      opzioni dichiarative e il builder le compone:

      | Opzione | Cosa collega |
      | --- | --- |
      | `security.dm` | Resolver della sicurezza DM con scope dai campi di configurazione |
      | `pairing.text` | Flusso di abbinamento DM basato su testo con scambio di codice |
      | `threading` | Resolver della modalità reply-to (fisso, con scope account o personalizzato) |
      | `outbound.attachedResults` | Funzioni di invio che restituiscono metadati del risultato (id messaggio) |

      Puoi anche passare oggetti adattatore raw invece delle opzioni dichiarative
      se hai bisogno del pieno controllo.
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

    Inserisci i descrittori CLI gestiti dal canale in `registerCliMetadata(...)` così OpenClaw
    può mostrarli nell'help root senza attivare il runtime completo del canale,
    mentre i normali caricamenti completi continuano a usare gli stessi descrittori per la vera registrazione dei comandi.
    Mantieni `registerFull(...)` per il lavoro solo runtime.
    Se `registerFull(...)` registra metodi gateway RPC, usa un
    prefisso specifico del plugin. I namespace admin del core (`config.*`,
    `exec.approvals.*`, `wizard.*`, `update.*`) restano riservati e
    risolvono sempre a `operator.admin`.
    `defineChannelPluginEntry` gestisce automaticamente la separazione della modalità di registrazione. Vedi
    [Entry Points](/it/plugins/sdk-entrypoints#definechannelpluginentry) per tutte le
    opzioni.

  </Step>

  <Step title="Aggiungi una voce di configurazione">
    Crea `setup-entry.ts` per un caricamento leggero durante l'onboarding:

    ```typescript setup-entry.ts
    import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineSetupPluginEntry(acmeChatPlugin);
    ```

    OpenClaw carica questo invece dell'entry completa quando il canale è disabilitato
    o non configurato. Questo evita di trascinare codice runtime pesante durante i flussi di configurazione.
    Vedi [Setup and Config](/it/plugins/sdk-setup#setup-entry) per i dettagli.

  </Step>

  <Step title="Gestisci i messaggi in ingresso">
    Il tuo plugin deve ricevere messaggi dalla piattaforma e inoltrarli a
    OpenClaw. Il pattern tipico è un webhook che verifica la richiesta e
    la inoltra tramite l'handler inbound del tuo canale:

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
      La gestione dei messaggi in ingresso è specifica del canale. Ogni plugin di canale gestisce
      la propria pipeline inbound. Guarda i plugin di canale bundled
      (per esempio il pacchetto plugin Microsoft Teams o Google Chat) per pattern reali.
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

    Per helper di test condivisi, vedi [Testing](/it/plugins/sdk-testing).

  </Step>
</Steps>

## Struttura dei file

```
<bundled-plugin-root>/acme-chat/
├── package.json              # metadata openclaw.channel
├── openclaw.plugin.json      # Manifest con schema di configurazione
├── index.ts                  # defineChannelPluginEntry
├── setup-entry.ts            # defineSetupPluginEntry
├── api.ts                    # Export pubblici (facoltativo)
├── runtime-api.ts            # Export runtime interni (facoltativo)
└── src/
    ├── channel.ts            # ChannelPlugin tramite createChatChannelPlugin
    ├── channel.test.ts       # Test
    ├── client.ts             # Client API della piattaforma
    └── runtime.ts            # Store runtime (se necessario)
```

## Argomenti avanzati

<CardGroup cols={2}>
  <Card title="Opzioni di threading" icon="git-branch" href="/it/plugins/sdk-entrypoints#registration-mode">
    Modalità di risposta fisse, con scope account o personalizzate
  </Card>
  <Card title="Integrazione dello strumento message" icon="puzzle" href="/it/plugins/architecture#channel-plugins-and-the-shared-message-tool">
    describeMessageTool e discovery delle azioni
  </Card>
  <Card title="Risoluzione del target" icon="crosshair" href="/it/plugins/architecture#channel-target-resolution">
    inferTargetChatType, looksLikeId, resolveTarget
  </Card>
  <Card title="Helper runtime" icon="settings" href="/it/plugins/sdk-runtime">
    TTS, STT, media, sottoagente tramite api.runtime
  </Card>
</CardGroup>

<Note>
Esistono ancora alcune seam helper bundled per manutenzione e
compatibilità dei plugin bundled. Non sono il pattern consigliato per i nuovi plugin di canale;
preferisci i sottopercorsi generici channel/setup/reply/runtime dalla comune
superficie SDK a meno che tu non stia mantenendo direttamente quella famiglia di plugin bundled.
</Note>

## Passaggi successivi

- [Provider Plugins](/it/plugins/sdk-provider-plugins) — se il tuo plugin fornisce anche modelli
- [SDK Overview](/it/plugins/sdk-overview) — riferimento completo agli import dei sottopercorsi
- [SDK Testing](/it/plugins/sdk-testing) — utility di test e test di contratto
- [Plugin Manifest](/it/plugins/manifest) — schema completo del manifest
