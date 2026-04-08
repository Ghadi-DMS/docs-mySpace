---
read_when:
    - Sie erstellen ein neues Messaging-Kanal-Plugin
    - Sie möchten OpenClaw mit einer Messaging-Plattform verbinden
    - Sie müssen die Adapter-Oberfläche von ChannelPlugin verstehen
sidebarTitle: Channel Plugins
summary: Schritt-für-Schritt-Anleitung zum Erstellen eines Messaging-Kanal-Plugins für OpenClaw
title: Kanal-Plugins erstellen
x-i18n:
    generated_at: "2026-04-08T02:17:14Z"
    model: gpt-5.4
    provider: openai
    source_hash: d23365b6d92006b30e671f9f0afdba40a2b88c845c5d2299d71c52a52985672f
    source_path: plugins/sdk-channel-plugins.md
    workflow: 15
---

# Kanal-Plugins erstellen

Diese Anleitung führt Sie durch das Erstellen eines Kanal-Plugins, das OpenClaw mit einer
Messaging-Plattform verbindet. Am Ende haben Sie einen funktionierenden Kanal mit DM-Sicherheit,
Kopplung, Reply-Threading und ausgehenden Nachrichten.

<Info>
  Wenn Sie noch nie ein OpenClaw-Plugin erstellt haben, lesen Sie zuerst
  [Erste Schritte](/de/plugins/building-plugins) für die grundlegende Paket-
  Struktur und Manifest-Einrichtung.
</Info>

## So funktionieren Kanal-Plugins

Kanal-Plugins benötigen keine eigenen Sende-/Bearbeiten-/Reagieren-Tools. OpenClaw behält ein
gemeinsam genutztes `message`-Tool im Core. Ihr Plugin besitzt:

- **Konfiguration** — Kontenauflösung und Einrichtungsassistent
- **Sicherheit** — DM-Richtlinie und Allowlists
- **Kopplung** — DM-Genehmigungsablauf
- **Sitzungsgrammatik** — wie anbieterspezifische Gesprächs-IDs auf Basis-Chats, Thread-IDs und Parent-Fallbacks abgebildet werden
- **Ausgehend** — Senden von Text, Medien und Umfragen an die Plattform
- **Threading** — wie Antworten in Threads organisiert werden

Der Core besitzt das gemeinsam genutzte Message-Tool, die Prompt-Verkabelung, die äußere Form des Sitzungsschlüssels,
die generische `:thread:`-Buchführung und den Versand.

Wenn Ihre Plattform zusätzlichen Geltungsbereich in Gesprächs-IDs speichert, behalten Sie dieses Parsing
im Plugin mit `messaging.resolveSessionConversation(...)`. Das ist der
kanonische Hook für die Zuordnung von `rawId` zur Basis-Gesprächs-ID, einer optionalen Thread-
ID, einer expliziten `baseConversationId` und allen `parentConversationCandidates`.
Wenn Sie `parentConversationCandidates` zurückgeben, halten Sie deren Reihenfolge von dem
engsten Parent bis zum breitesten/Basis-Gespräch ein.

Gebündelte Plugins, die dasselbe Parsing benötigen, bevor die Kanal-Registry startet,
können auch eine Datei `session-key-api.ts` auf oberster Ebene mit einem passenden
Export `resolveSessionConversation(...)` bereitstellen. Der Core verwendet diese bootstrap-sichere Oberfläche
nur dann, wenn die Laufzeit-Plugin-Registry noch nicht verfügbar ist.

`messaging.resolveParentConversationCandidates(...)` bleibt als
Legacy-Kompatibilitäts-Fallback verfügbar, wenn ein Plugin nur Parent-Fallbacks zusätzlich
zur generischen/rohen ID benötigt. Wenn beide Hooks existieren, verwendet der Core
zuerst `resolveSessionConversation(...).parentConversationCandidates` und fällt nur
auf `resolveParentConversationCandidates(...)` zurück, wenn der kanonische Hook
sie auslässt.

## Genehmigungen und Kanalfähigkeiten

Die meisten Kanal-Plugins benötigen keinen genehmigungsspezifischen Code.

- Der Core besitzt `/approve` im selben Chat, gemeinsam genutzte Payloads für Genehmigungsschaltflächen und generische Fallback-Zustellung.
- Bevorzugen Sie ein einzelnes `approvalCapability`-Objekt auf dem Kanal-Plugin, wenn der Kanal genehmigungsspezifisches Verhalten benötigt.
- `ChannelPlugin.approvals` wurde entfernt. Legen Sie Fakten zu Genehmigungszustellung/nativem Verhalten/Rendering/Auth in `approvalCapability` ab.
- `plugin.auth` ist nur für Login/Logout; der Core liest keine Genehmigungs-Auth-Hooks mehr aus diesem Objekt.
- `approvalCapability.authorizeActorAction` und `approvalCapability.getActionAvailabilityState` sind die kanonische Nahtstelle für Genehmigungs-Auth.
- Verwenden Sie `approvalCapability.getActionAvailabilityState` für die Verfügbarkeit von Genehmigungs-Auth im selben Chat.
- Wenn Ihr Kanal native Exec-Genehmigungen bereitstellt, verwenden Sie `approvalCapability.getExecInitiatingSurfaceState` für den initiierenden Oberflächen-/Native-Client-Status, wenn dieser sich von der Genehmigungs-Auth im selben Chat unterscheidet. Der Core verwendet diesen exec-spezifischen Hook, um `enabled` und `disabled` zu unterscheiden, zu entscheiden, ob der initiierende Kanal native Exec-Genehmigungen unterstützt, und den Kanal in die Fallback-Hinweise für Native Clients aufzunehmen. `createApproverRestrictedNativeApprovalCapability(...)` füllt dies für den häufigen Fall aus.
- Verwenden Sie `outbound.shouldSuppressLocalPayloadPrompt` oder `outbound.beforeDeliverPayload` für kanalspezifisches Payload-Lifecycle-Verhalten, etwa das Ausblenden doppelter lokaler Genehmigungs-Prompts oder das Senden von Tippindikatoren vor der Zustellung.
- Verwenden Sie `approvalCapability.delivery` nur für natives Genehmigungs-Routing oder Unterdrückung von Fallbacks.
- Verwenden Sie `approvalCapability.nativeRuntime` für kanalinterne native Fakten zu Genehmigungen. Halten Sie es auf Hot-Channel-Entrypoints lazy mit `createLazyChannelApprovalNativeRuntimeAdapter(...)`, das Ihr Laufzeitmodul bei Bedarf importieren kann, während der Core weiterhin den Genehmigungs-Lifecycle zusammensetzt.
- Verwenden Sie `approvalCapability.render` nur, wenn ein Kanal wirklich benutzerdefinierte Genehmigungs-Payloads statt des gemeinsam genutzten Renderers benötigt.
- Verwenden Sie `approvalCapability.describeExecApprovalSetup`, wenn der Kanal in der Disabled-Antwort die exakten Konfigurationsschalter erläutern soll, die zum Aktivieren nativer Exec-Genehmigungen nötig sind. Der Hook erhält `{ channel, channelLabel, accountId }`; Kanäle mit benannten Konten sollten kontobezogene Pfade wie `channels.<channel>.accounts.<id>.execApprovals.*` statt Standardwerten auf oberster Ebene ausgeben.
- Wenn ein Kanal stabile DM-Identitäten mit Owner-Charakter aus vorhandener Konfiguration ableiten kann, verwenden Sie `createResolvedApproverActionAuthAdapter` aus `openclaw/plugin-sdk/approval-runtime`, um `/approve` im selben Chat einzuschränken, ohne genehmigungsspezifische Core-Logik hinzuzufügen.
- Wenn ein Kanal native Genehmigungszustellung benötigt, konzentrieren Sie den Kanalcode auf Zielnormalisierung sowie Fakten zu Transport/Darstellung. Verwenden Sie `createChannelExecApprovalProfile`, `createChannelNativeOriginTargetResolver`, `createChannelApproverDmTargetResolver` und `createApproverRestrictedNativeApprovalCapability` aus `openclaw/plugin-sdk/approval-runtime`. Platzieren Sie die kanalspezifischen Fakten hinter `approvalCapability.nativeRuntime`, idealerweise über `createChannelApprovalNativeRuntimeAdapter(...)` oder `createLazyChannelApprovalNativeRuntimeAdapter(...)`, damit der Core den Handler zusammensetzen und Anforderungsfilterung, Routing, Deduplizierung, Ablauf, Gateway-Abonnement und Hinweise auf anderweitig geroutete Anfragen übernehmen kann. `nativeRuntime` ist in einige kleinere Schnittstellen aufgeteilt:
- `availability` — ob das Konto konfiguriert ist und ob eine Anfrage verarbeitet werden soll
- `presentation` — Zuordnung des gemeinsam genutzten Genehmigungs-View-Models zu nativen Payloads für ausstehend/aufgelöst/abgelaufen oder zu finalen Aktionen
- `transport` — Ziele vorbereiten sowie native Genehmigungsnachrichten senden/aktualisieren/löschen
- `interactions` — optionale Hooks zum Binden/Lösen/Löschen von Aktionen für native Buttons oder Reaktionen
- `observe` — optionale Hooks für Zustellungsdiagnostik
- Wenn der Kanal laufzeitinterne Objekte wie einen Client, Token, eine Bolt-App oder einen Webhook-Empfänger benötigt, registrieren Sie sie über `openclaw/plugin-sdk/channel-runtime-context`. Die generische Laufzeitkontext-Registry ermöglicht es dem Core, capability-gesteuerte Handler aus dem Kanal-Startup-Zustand zu bootstrappen, ohne genehmigungsspezifischen Wrapper-Kleber hinzuzufügen.
- Greifen Sie nur dann zu den Low-Level-Funktionen `createChannelApprovalHandler` oder `createChannelNativeApprovalRuntime`, wenn die capability-gesteuerte Schnittstelle noch nicht ausdrucksstark genug ist.
- Native Genehmigungskanäle müssen sowohl `accountId` als auch `approvalKind` durch diese Helfer leiten. `accountId` hält Richtlinien für Multi-Account-Genehmigungen auf das richtige Bot-Konto beschränkt, und `approvalKind` hält Exec- gegenüber Plugin-Genehmigungsverhalten für den Kanal verfügbar, ohne hart codierte Verzweigungen im Core.
- Der Core besitzt jetzt auch Hinweise zur Umleitung von Genehmigungen. Kanal-Plugins sollten daher keine eigenen Folgenachrichten wie „Genehmigung ging an DMs / einen anderen Kanal“ aus `createChannelNativeApprovalRuntime` senden; stattdessen sollten sie korrektes Routing von Ursprung + Genehmiger-DM über die gemeinsam genutzten Helfer für Genehmigungs-Capabilities bereitstellen, damit der Core tatsächliche Zustellungen aggregieren kann, bevor er einen Hinweis zurück in den initiierenden Chat sendet.
- Bewahren Sie die Art der zugestellten Genehmigungs-ID durchgängig. Native Clients sollten
  das Routing von Exec- gegenüber Plugin-Genehmigungen nicht aus kanal-lokalem Zustand erraten oder umschreiben.
- Verschiedene Genehmigungsarten können absichtlich unterschiedliche native Oberflächen bereitstellen.
  Aktuelle gebündelte Beispiele:
  - Slack hält natives Genehmigungs-Routing sowohl für Exec- als auch für Plugin-IDs verfügbar.
  - Matrix behält dasselbe native DM-/Kanal-Routing und dieselbe Reaktions-UX für Exec-
    und Plugin-Genehmigungen bei, lässt Auth jedoch weiterhin je nach Genehmigungsart unterschiedlich sein.
- `createApproverRestrictedNativeApprovalAdapter` existiert weiterhin als Kompatibilitäts-Wrapper, neuer Code sollte jedoch den Capability-Builder bevorzugen und `approvalCapability` auf dem Plugin bereitstellen.

Für Hot-Channel-Entrypoints sollten Sie die schmaleren Runtime-Unterpfade bevorzugen, wenn Sie nur
einen Teil dieser Familie benötigen:

- `openclaw/plugin-sdk/approval-auth-runtime`
- `openclaw/plugin-sdk/approval-client-runtime`
- `openclaw/plugin-sdk/approval-delivery-runtime`
- `openclaw/plugin-sdk/approval-gateway-runtime`
- `openclaw/plugin-sdk/approval-handler-adapter-runtime`
- `openclaw/plugin-sdk/approval-handler-runtime`
- `openclaw/plugin-sdk/approval-native-runtime`
- `openclaw/plugin-sdk/approval-reply-runtime`
- `openclaw/plugin-sdk/channel-runtime-context`

Bevorzugen Sie ebenso `openclaw/plugin-sdk/setup-runtime`,
`openclaw/plugin-sdk/setup-adapter-runtime`,
`openclaw/plugin-sdk/reply-runtime`,
`openclaw/plugin-sdk/reply-dispatch-runtime`,
`openclaw/plugin-sdk/reply-reference` und
`openclaw/plugin-sdk/reply-chunking`, wenn Sie die breitere
Oberfläche nicht benötigen.

Speziell für Setup:

- `openclaw/plugin-sdk/setup-runtime` deckt die laufzeitsicheren Setup-Helfer ab:
  import-sichere Setup-Patch-Adapter (`createPatchedAccountSetupAdapter`,
  `createEnvPatchedAccountSetupAdapter`,
  `createSetupInputPresenceValidator`), Ausgabe von Lookup-Hinweisen,
  `promptResolvedAllowFrom`, `splitSetupEntries` und die delegierten
  Setup-Proxy-Builder
- `openclaw/plugin-sdk/setup-adapter-runtime` ist die schmale, env-bewusste Adapter-
  Schnittstelle für `createEnvPatchedAccountSetupAdapter`
- `openclaw/plugin-sdk/channel-setup` deckt die Setup-Builder für optionale Installationen
  sowie einige setup-sichere Primitive ab:
  `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`,

Wenn Ihr Kanal env-gesteuertes Setup oder Auth unterstützt und generische Startup-/Konfigurations-
Abläufe diese Env-Namen kennen sollen, bevor die Laufzeit geladen wird, deklarieren Sie sie im
Plugin-Manifest mit `channelEnvVars`. Behalten Sie Runtime-`envVars` des Kanals oder lokale
Konstanten nur für operatorseitige Texte bei.
`createOptionalChannelSetupWizard`, `DEFAULT_ACCOUNT_ID`,
`createTopLevelChannelDmPolicy`, `setSetupChannelEnabled` und
`splitSetupEntries`

- verwenden Sie die breitere Schnittstelle `openclaw/plugin-sdk/setup` nur dann, wenn Sie auch die
  schwergewichtigeren gemeinsam genutzten Setup-/Konfigurationshelfer benötigen, etwa
  `moveSingleAccountChannelSectionToDefaultAccount(...)`

Wenn Ihr Kanal in Setup-Oberflächen nur „installieren Sie dieses Plugin zuerst“ bewerben möchte, bevorzugen Sie `createOptionalChannelSetupSurface(...)`. Der generierte
Adapter/Assistent schlägt bei Konfigurationsschreibvorgängen und Finalisierung fail-closed fehl und verwendet
dieselbe Meldung „Installation erforderlich“ für Validierung, Finalisierung und Docs-Link-Text.

Für andere Hot-Channel-Pfade sollten Sie die schmalen Helfer gegenüber breiteren Legacy-
Oberflächen bevorzugen:

- `openclaw/plugin-sdk/account-core`,
  `openclaw/plugin-sdk/account-id`,
  `openclaw/plugin-sdk/account-resolution` und
  `openclaw/plugin-sdk/account-helpers` für Multi-Account-Konfiguration und
  Fallback auf das Standardkonto
- `openclaw/plugin-sdk/inbound-envelope` und
  `openclaw/plugin-sdk/inbound-reply-dispatch` für die Verkabelung von eingehender Route/Envelope
  sowie Record-and-Dispatch
- `openclaw/plugin-sdk/messaging-targets` für Ziel-Parsing/-Matching
- `openclaw/plugin-sdk/outbound-media` und
  `openclaw/plugin-sdk/outbound-runtime` für Medienladen sowie Delegates für ausgehende
  Identität/Senden
- `openclaw/plugin-sdk/thread-bindings-runtime` für den Thread-Binding-Lifecycle
  und die Adapter-Registrierung
- `openclaw/plugin-sdk/agent-media-payload` nur dann, wenn weiterhin ein Legacy-Agent-/Media-
  Payload-Feldlayout erforderlich ist
- `openclaw/plugin-sdk/telegram-command-config` für die Normalisierung benutzerdefinierter Telegram-Befehle,
  Validierung von Duplikaten/Konflikten und einen fallback-stabilen Vertrag für Befehls-
  Konfiguration

Kanäle nur mit Auth können meist beim Standardpfad bleiben: Der Core übernimmt Genehmigungen und das Plugin stellt nur Outbound-/Auth-Capabilities bereit. Kanäle mit nativen Genehmigungen wie Matrix, Slack, Telegram und benutzerdefinierte Chat-Transporte sollten die gemeinsam genutzten nativen Helfer verwenden, statt ihren eigenen Genehmigungs-Lifecycle zu bauen.

## Richtlinie für eingehende Erwähnungen

Halten Sie die Behandlung eingehender Erwähnungen in zwei Schichten getrennt:

- plugin-eigene Sammlung von Belegen
- gemeinsam genutzte Richtlinienauswertung

Verwenden Sie `openclaw/plugin-sdk/channel-inbound` für die gemeinsam genutzte Schicht.

Geeignet für plugin-lokale Logik:

- Erkennung von Antworten an den Bot
- Erkennung von Zitaten des Bots
- Prüfungen auf Thread-Beteiligung
- Ausschlüsse von Service-/Systemnachrichten
- plattformspezifische Caches, die nötig sind, um die Beteiligung des Bots nachzuweisen

Geeignet für den gemeinsam genutzten Helfer:

- `requireMention`
- explizites Erwähnungsergebnis
- Allowlist für implizite Erwähnungen
- Umgehung für Befehle
- endgültige Überspringen-Entscheidung

Bevorzugter Ablauf:

1. Lokale Erwähnungsfakten berechnen.
2. Diese Fakten an `resolveInboundMentionDecision({ facts, policy })` übergeben.
3. `decision.effectiveWasMentioned`, `decision.shouldBypassMention` und `decision.shouldSkip` in Ihrer Inbound-Gate-Logik verwenden.

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

`api.runtime.channel.mentions` stellt dieselben gemeinsam genutzten Erwähnungshelfer für
gebündelte Kanal-Plugins bereit, die bereits von Laufzeit-Injektion abhängen:

- `buildMentionRegexes`
- `matchesMentionPatterns`
- `matchesMentionWithExplicit`
- `implicitMentionKindWhen`
- `resolveInboundMentionDecision`

Die älteren Helfer `resolveMentionGating*` bleiben auf
`openclaw/plugin-sdk/channel-inbound` nur als Kompatibilitäts-Exporte erhalten. Neuer Code
sollte `resolveInboundMentionDecision({ facts, policy })` verwenden.

## Schritt-für-Schritt-Anleitung

<Steps>
  <a id="step-1-package-and-manifest"></a>
  <Step title="Paket und Manifest">
    Erstellen Sie die Standarddateien des Plugins. Das Feld `channel` in `package.json`
    macht dies zu einem Kanal-Plugin. Die vollständige Oberfläche der Paketmetadaten finden Sie
    unter [Plugin Setup and Config](/de/plugins/sdk-setup#openclawchannel):

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

  <Step title="Das Kanal-Plugin-Objekt erstellen">
    Die Schnittstelle `ChannelPlugin` hat viele optionale Adapter-Oberflächen. Beginnen Sie mit
    dem Minimum — `id` und `setup` — und fügen Sie Adapter nach Bedarf hinzu.

    Erstellen Sie `src/channel.ts`:

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

    <Accordion title="Was createChatChannelPlugin für Sie übernimmt">
      Statt Low-Level-Adapter-Schnittstellen manuell zu implementieren, übergeben Sie
      deklarative Optionen, und der Builder setzt sie zusammen:

      | Option | Was verkabelt wird |
      | --- | --- |
      | `security.dm` | Auf Konfigurationsfeldern basierender, bereichsbezogener DM-Sicherheits-Resolver |
      | `pairing.text` | Textbasierter DM-Kopplungsablauf mit Code-Austausch |
      | `threading` | Resolver für Reply-Modus (fest, kontoabhängig oder benutzerdefiniert) |
      | `outbound.attachedResults` | Sendefunktionen, die Ergebnis-Metadaten zurückgeben (Nachrichten-IDs) |

      Sie können auch rohe Adapter-Objekte statt der deklarativen Optionen übergeben,
      wenn Sie vollständige Kontrolle benötigen.
    </Accordion>

  </Step>

  <Step title="Den Einstiegspunkt verdrahten">
    Erstellen Sie `index.ts`:

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

    Legen Sie kanalinterne CLI-Deskriptoren in `registerCliMetadata(...)` ab, damit OpenClaw
    sie in der Root-Hilfe anzeigen kann, ohne die vollständige Kanal-Laufzeit zu aktivieren,
    während normale vollständige Ladevorgänge dieselben Deskriptoren weiterhin für die echte Befehls-
    Registrierung übernehmen. Behalten Sie `registerFull(...)` für reine Laufzeitarbeit bei.
    Wenn `registerFull(...)` Gateway-RPC-Methoden registriert, verwenden Sie ein
    pluginspezifisches Präfix. Core-Admin-Namespaces (`config.*`,
    `exec.approvals.*`, `wizard.*`, `update.*`) bleiben reserviert und werden immer
    zu `operator.admin` aufgelöst.
    `defineChannelPluginEntry` übernimmt die Aufteilung nach Registrierungsmodus automatisch. Alle
    Optionen finden Sie unter [Entry Points](/de/plugins/sdk-entrypoints#definechannelpluginentry).

  </Step>

  <Step title="Einen Setup-Eintrag hinzufügen">
    Erstellen Sie `setup-entry.ts` für leichtgewichtiges Laden während des Onboardings:

    ```typescript setup-entry.ts
    import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineSetupPluginEntry(acmeChatPlugin);
    ```

    OpenClaw lädt dies statt des vollständigen Einstiegspunkts, wenn der Kanal deaktiviert
    oder nicht konfiguriert ist. Dadurch wird vermieden, während Setup-Abläufen
    schwergewichtigen Laufzeitcode zu laden. Einzelheiten finden Sie unter [Setup and Config](/de/plugins/sdk-setup#setup-entry).

  </Step>

  <Step title="Eingehende Nachrichten verarbeiten">
    Ihr Plugin muss Nachrichten von der Plattform empfangen und an
    OpenClaw weiterleiten. Das typische Muster ist ein Webhook, der die Anfrage verifiziert und
    sie durch den Inbound-Handler Ihres Kanals weiterleitet:

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
      Die Verarbeitung eingehender Nachrichten ist kanalspezifisch. Jedes Kanal-Plugin besitzt
      seine eigene Inbound-Pipeline. Sehen Sie sich gebündelte Kanal-Plugins an
      (zum Beispiel das Plugin-Paket für Microsoft Teams oder Google Chat), um reale Muster zu sehen.
    </Note>

  </Step>

<a id="step-6-test"></a>
<Step title="Testen">
Schreiben Sie colocated Tests in `src/channel.test.ts`:

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

    Gemeinsam genutzte Testhelfer finden Sie unter [Testing](/de/plugins/sdk-testing).

  </Step>
</Steps>

## Dateistruktur

```
<bundled-plugin-root>/acme-chat/
├── package.json              # openclaw.channel-Metadaten
├── openclaw.plugin.json      # Manifest mit Konfigurationsschema
├── index.ts                  # defineChannelPluginEntry
├── setup-entry.ts            # defineSetupPluginEntry
├── api.ts                    # Öffentliche Exporte (optional)
├── runtime-api.ts            # Interne Laufzeit-Exporte (optional)
└── src/
    ├── channel.ts            # ChannelPlugin über createChatChannelPlugin
    ├── channel.test.ts       # Tests
    ├── client.ts             # Plattform-API-Client
    └── runtime.ts            # Laufzeitspeicher (falls erforderlich)
```

## Fortgeschrittene Themen

<CardGroup cols={2}>
  <Card title="Threading-Optionen" icon="git-branch" href="/de/plugins/sdk-entrypoints#registration-mode">
    Feste, kontoabhängige oder benutzerdefinierte Reply-Modi
  </Card>
  <Card title="Integration des Message-Tools" icon="puzzle" href="/de/plugins/architecture#channel-plugins-and-the-shared-message-tool">
    describeMessageTool und Action Discovery
  </Card>
  <Card title="Zielauflösung" icon="crosshair" href="/de/plugins/architecture#channel-target-resolution">
    inferTargetChatType, looksLikeId, resolveTarget
  </Card>
  <Card title="Laufzeithelfer" icon="settings" href="/de/plugins/sdk-runtime">
    TTS, STT, Medien, Subagent über api.runtime
  </Card>
</CardGroup>

<Note>
Einige gebündelte Hilfs-Schnittstellen existieren weiterhin für die Wartung gebündelter Plugins und
für Kompatibilität. Sie sind nicht das empfohlene Muster für neue Kanal-Plugins;
bevorzugen Sie die generischen Unterpfade channel/setup/reply/runtime aus der gemeinsamen SDK-
Oberfläche, sofern Sie nicht direkt diese gebündelte Plugin-Familie warten.
</Note>

## Nächste Schritte

- [Provider Plugins](/de/plugins/sdk-provider-plugins) — wenn Ihr Plugin auch Modelle bereitstellt
- [SDK Overview](/de/plugins/sdk-overview) — vollständige Referenz für Subpath-Importe
- [SDK Testing](/de/plugins/sdk-testing) — Test-Utilities und Vertragstests
- [Plugin Manifest](/de/plugins/manifest) — vollständiges Manifest-Schema
