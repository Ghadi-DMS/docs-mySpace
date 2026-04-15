---
read_when:
    - Sie erstellen ein neues Messaging-Channel-Plugin.
    - Sie möchten OpenClaw mit einer Messaging-Plattform verbinden.
    - Sie müssen die Adapter-Oberfläche von ChannelPlugin verstehen.
sidebarTitle: Channel Plugins
summary: Schritt-für-Schritt-Anleitung zum Erstellen eines Messaging-Channel-Plugins für OpenClaw
title: Erstellen von Channel-Plugins
x-i18n:
    generated_at: "2026-04-15T06:21:28Z"
    model: gpt-5.4
    provider: openai
    source_hash: a7f4c746fe3163a8880e14c433f4db4a1475535d91716a53fb879551d8d62f65
    source_path: plugins/sdk-channel-plugins.md
    workflow: 15
---

# Erstellen von Channel-Plugins

Diese Anleitung führt Sie durch das Erstellen eines Channel-Plugins, das OpenClaw mit einer Messaging-Plattform verbindet. Am Ende haben Sie einen funktionierenden Channel mit DM-Sicherheit, Pairing, Antwort-Threading und ausgehender Nachrichtenübermittlung.

<Info>
  Wenn Sie bisher noch kein OpenClaw-Plugin erstellt haben, lesen Sie zuerst
  [Erste Schritte](/de/plugins/building-plugins) für die grundlegende
  Paketstruktur und die Manifest-Einrichtung.
</Info>

## So funktionieren Channel-Plugins

Channel-Plugins benötigen keine eigenen Send/Edit/React-Tools. OpenClaw behält ein
gemeinsam genutztes `message`-Tool im Core. Ihr Plugin ist zuständig für:

- **Konfiguration** — Account-Auflösung und Einrichtungsassistent
- **Sicherheit** — DM-Richtlinie und Zulassungslisten
- **Pairing** — DM-Genehmigungsablauf
- **Sitzungsgrammatik** — wie anbieterspezifische Konversations-IDs auf Basis-Chats, Thread-IDs und Parent-Fallbacks abgebildet werden
- **Ausgehend** — Senden von Text, Medien und Umfragen an die Plattform
- **Threading** — wie Antworten in Threads organisiert werden

Der Core ist zuständig für das gemeinsam genutzte Message-Tool, die Prompt-Verdrahtung, die äußere Sitzungs-Schlüsselstruktur, generische `:thread:`-Buchführung und Dispatch.

Wenn Ihr Channel `message`-Tool-Parameter hinzufügt, die Medienquellen transportieren, stellen Sie diese Parameternamen über `describeMessageTool(...).mediaSourceParams` bereit. Der Core verwendet diese explizite Liste für die Normalisierung von Sandbox-Pfaden und die Richtlinie für ausgehenden Medienzugriff, sodass Plugins keine Shared-Core-Sonderfälle für anbieterspezifische Avatar-, Anhangs- oder Titelbild-Parameter benötigen.
Geben Sie vorzugsweise eine aktionsbezogene Zuordnung zurück, etwa
`{ "set-profile": ["avatarUrl", "avatarPath"] }`, damit nicht zusammenhängende Aktionen nicht die Medienargumente einer anderen Aktion übernehmen. Ein flaches Array funktioniert weiterhin für Parameter, die absichtlich über alle bereitgestellten Aktionen hinweg gemeinsam genutzt werden.

Wenn Ihre Plattform zusätzlichen Geltungsbereich in Konversations-IDs speichert, belassen Sie diese Analyse im Plugin mit `messaging.resolveSessionConversation(...)`. Dies ist der kanonische Hook, um `rawId` auf die Basis-Konversations-ID, eine optionale Thread-ID, eine explizite `baseConversationId` und beliebige `parentConversationCandidates` abzubilden.
Wenn Sie `parentConversationCandidates` zurückgeben, halten Sie sie in der Reihenfolge vom spezifischsten Parent bis zur breitesten/Basis-Konversation.

Gebündelte Plugins, die dieselbe Analyse benötigen, bevor die Channel-Registry startet, können außerdem eine Top-Level-Datei `session-key-api.ts` mit einem passenden Export `resolveSessionConversation(...)` bereitstellen. Der Core verwendet diese Bootstrap-sichere Oberfläche nur dann, wenn die Runtime-Plugin-Registry noch nicht verfügbar ist.

`messaging.resolveParentConversationCandidates(...)` bleibt als Legacy-Kompatibilitäts-Fallback verfügbar, wenn ein Plugin nur Parent-Fallbacks zusätzlich zur generischen/rohen ID benötigt. Wenn beide Hooks vorhanden sind, verwendet der Core zuerst `resolveSessionConversation(...).parentConversationCandidates` und greift nur auf `resolveParentConversationCandidates(...)` zurück, wenn der kanonische Hook diese auslässt.

## Genehmigungen und Channel-Fähigkeiten

Die meisten Channel-Plugins benötigen keinen Genehmigungs-spezifischen Code.

- Der Core ist zuständig für `/approve` im selben Chat, gemeinsam genutzte Payloads für Genehmigungs-Buttons und generische Fallback-Zustellung.
- Bevorzugen Sie ein einzelnes `approvalCapability`-Objekt auf dem Channel-Plugin, wenn der Channel Genehmigungs-spezifisches Verhalten benötigt.
- `ChannelPlugin.approvals` wurde entfernt. Legen Sie Fakten zu Genehmigungszustellung/nativer Unterstützung/Rendering/Auth in `approvalCapability` ab.
- `plugin.auth` dient nur für Login/Logout; der Core liest aus diesem Objekt keine Genehmigungs-Auth-Hooks mehr.
- `approvalCapability.authorizeActorAction` und `approvalCapability.getActionAvailabilityState` sind die kanonische Schnittstelle für Genehmigungs-Auth.
- Verwenden Sie `approvalCapability.getActionAvailabilityState` für die Verfügbarkeit von Genehmigungs-Auth bei Genehmigungen im selben Chat.
- Wenn Ihr Channel native Exec-Genehmigungen bereitstellt, verwenden Sie `approvalCapability.getExecInitiatingSurfaceState` für den Zustand der auslösenden Oberfläche/des nativen Clients, wenn dieser sich von der Genehmigungs-Auth im selben Chat unterscheidet. Der Core verwendet diesen exec-spezifischen Hook, um zwischen `enabled` und `disabled` zu unterscheiden, zu entscheiden, ob der auslösende Channel native Exec-Genehmigungen unterstützt, und den Channel in Hinweisen für native Client-Fallbacks einzubeziehen. `createApproverRestrictedNativeApprovalCapability(...)` füllt dies für den häufigen Fall aus.
- Verwenden Sie `outbound.shouldSuppressLocalPayloadPrompt` oder `outbound.beforeDeliverPayload` für Channel-spezifisches Payload-Lifecycle-Verhalten, etwa das Ausblenden doppelter lokaler Genehmigungs-Prompts oder das Senden von Tippindikatoren vor der Zustellung.
- Verwenden Sie `approvalCapability.delivery` nur für natives Genehmigungs-Routing oder die Unterdrückung von Fallbacks.
- Verwenden Sie `approvalCapability.nativeRuntime` für Channel-eigene Fakten zu nativen Genehmigungen. Halten Sie diese auf Hot-Path-Channel-Entrypoints lazy mit `createLazyChannelApprovalNativeRuntimeAdapter(...)`, das Ihr Runtime-Modul bei Bedarf importieren kann, während der Core dennoch den Genehmigungs-Lifecycle zusammenstellen kann.
- Verwenden Sie `approvalCapability.render` nur dann, wenn ein Channel wirklich benutzerdefinierte Genehmigungs-Payloads statt des gemeinsam genutzten Renderers benötigt.
- Verwenden Sie `approvalCapability.describeExecApprovalSetup`, wenn der Channel in der Antwort für den deaktivierten Pfad die genauen Konfigurationsschalter erläutern soll, die zum Aktivieren nativer Exec-Genehmigungen erforderlich sind. Der Hook erhält `{ channel, channelLabel, accountId }`; Channels mit benannten Accounts sollten accountbezogene Pfade wie `channels.<channel>.accounts.<id>.execApprovals.*` statt Top-Level-Standards rendern.
- Wenn ein Channel aus vorhandener Konfiguration stabile owner-ähnliche DM-Identitäten ableiten kann, verwenden Sie `createResolvedApproverActionAuthAdapter` aus `openclaw/plugin-sdk/approval-runtime`, um `/approve` im selben Chat einzuschränken, ohne Genehmigungs-spezifische Core-Logik hinzuzufügen.
- Wenn ein Channel native Genehmigungszustellung benötigt, halten Sie den Channel-Code auf Zielnormalisierung plus Fakten zu Transport/Präsentation fokussiert. Verwenden Sie `createChannelExecApprovalProfile`, `createChannelNativeOriginTargetResolver`, `createChannelApproverDmTargetResolver` und `createApproverRestrictedNativeApprovalCapability` aus `openclaw/plugin-sdk/approval-runtime`. Legen Sie die Channel-spezifischen Fakten hinter `approvalCapability.nativeRuntime`, idealerweise über `createChannelApprovalNativeRuntimeAdapter(...)` oder `createLazyChannelApprovalNativeRuntimeAdapter(...)`, damit der Core den Handler zusammensetzen und Request-Filterung, Routing, Deduplizierung, Ablauf, Gateway-Abonnement und Hinweise zu anderweitiger Weiterleitung übernehmen kann. `nativeRuntime` ist in einige kleinere Schnittstellen aufgeteilt:
- `availability` — ob der Account konfiguriert ist und ob eine Anfrage behandelt werden sollte
- `presentation` — Abbildung des gemeinsam genutzten View-Models für Genehmigungen auf ausstehende/aufgelöste/abgelaufene native Payloads oder endgültige Aktionen
- `transport` — Vorbereitung von Zielen sowie Senden/Aktualisieren/Löschen nativer Genehmigungsnachrichten
- `interactions` — optionale Hooks zum Binden/Lösen/Löschen von Aktionen für native Buttons oder Reaktionen
- `observe` — optionale Hooks für Zustellungsdiagnostik
- Wenn der Channel Runtime-eigene Objekte wie einen Client, Token, eine Bolt-App oder einen Webhook-Empfänger benötigt, registrieren Sie diese über `openclaw/plugin-sdk/channel-runtime-context`. Die generische Runtime-Context-Registry ermöglicht dem Core, fähigkeitsgesteuerte Handler aus dem Startzustand des Channels zu bootstrappen, ohne Genehmigungs-spezifischen Wrapper-Klebstoff hinzuzufügen.
- Greifen Sie nur dann zu den Low-Level-Funktionen `createChannelApprovalHandler` oder `createChannelNativeApprovalRuntime`, wenn die fähigkeitsgesteuerte Schnittstelle noch nicht aussagekräftig genug ist.
- Native Genehmigungs-Channels müssen sowohl `accountId` als auch `approvalKind` durch diese Hilfsfunktionen leiten. `accountId` hält die Multi-Account-Genehmigungsrichtlinie auf den richtigen Bot-Account begrenzt, und `approvalKind` stellt Exec- vs. Plugin-Genehmigungsverhalten dem Channel zur Verfügung, ohne fest codierte Verzweigungen im Core.
- Der Core ist jetzt auch für Hinweise zur erneuten Weiterleitung von Genehmigungen zuständig. Channel-Plugins sollten daher keine eigenen Follow-up-Nachrichten wie „Genehmigung ging an DMs / einen anderen Channel“ aus `createChannelNativeApprovalRuntime` senden; stattdessen sollten sie präzises Routing für Ursprung + Approver-DM über die gemeinsam genutzten Hilfsfunktionen für Genehmigungs-Fähigkeiten bereitstellen und den Core die tatsächlichen Zustellungen aggregieren lassen, bevor irgendein Hinweis in den auslösenden Chat gepostet wird.
- Behalten Sie die Art der zugestellten Genehmigungs-ID Ende-zu-Ende bei. Native Clients sollten Exec- vs. Plugin-Genehmigungsrouting nicht anhand Channel-lokalen Zustands erraten oder umschreiben.
- Verschiedene Arten von Genehmigungen können absichtlich unterschiedliche native Oberflächen bereitstellen.
  Aktuelle gebündelte Beispiele:
  - Slack hält natives Genehmigungsrouting sowohl für Exec- als auch für Plugin-IDs verfügbar.
  - Matrix behält dasselbe native DM-/Channel-Routing und dieselbe Reaktions-UX für Exec- und Plugin-Genehmigungen bei, während sich Auth weiterhin je nach Genehmigungsart unterscheiden kann.
- `createApproverRestrictedNativeApprovalAdapter` existiert weiterhin als Kompatibilitäts-Wrapper, aber neuer Code sollte den Capability-Builder bevorzugen und `approvalCapability` im Plugin bereitstellen.

Für Hot-Path-Channel-Entrypoints sollten Sie die schmaleren Runtime-Unterpfade bevorzugen, wenn Sie nur einen Teil dieser Familie benötigen:

- `openclaw/plugin-sdk/approval-auth-runtime`
- `openclaw/plugin-sdk/approval-client-runtime`
- `openclaw/plugin-sdk/approval-delivery-runtime`
- `openclaw/plugin-sdk/approval-gateway-runtime`
- `openclaw/plugin-sdk/approval-handler-adapter-runtime`
- `openclaw/plugin-sdk/approval-handler-runtime`
- `openclaw/plugin-sdk/approval-native-runtime`
- `openclaw/plugin-sdk/approval-reply-runtime`
- `openclaw/plugin-sdk/channel-runtime-context`

Ebenso sollten Sie `openclaw/plugin-sdk/setup-runtime`,
`openclaw/plugin-sdk/setup-adapter-runtime`,
`openclaw/plugin-sdk/reply-runtime`,
`openclaw/plugin-sdk/reply-dispatch-runtime`,
`openclaw/plugin-sdk/reply-reference` und
`openclaw/plugin-sdk/reply-chunking` bevorzugen, wenn Sie die breitere übergreifende Oberfläche nicht benötigen.

Speziell für das Setup gilt:

- `openclaw/plugin-sdk/setup-runtime` deckt die Runtime-sicheren Setup-Hilfen ab:
  import-sichere Setup-Patch-Adapter (`createPatchedAccountSetupAdapter`,
  `createEnvPatchedAccountSetupAdapter`,
  `createSetupInputPresenceValidator`), Ausgabe von Lookup-Hinweisen,
  `promptResolvedAllowFrom`, `splitSetupEntries` und die delegierten
  Setup-Proxy-Builder
- `openclaw/plugin-sdk/setup-adapter-runtime` ist die schmale env-bewusste Adapter-Schnittstelle
  für `createEnvPatchedAccountSetupAdapter`
- `openclaw/plugin-sdk/channel-setup` deckt die Setup-Builder für optionale Installationen
  plus einige Setup-sichere Primitive ab:
  `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`,

Wenn Ihr Channel env-gesteuertes Setup oder Auth unterstützt und generische Startup-/Konfigurationsabläufe diese Env-Namen vor dem Laden der Runtime kennen sollen, deklarieren Sie sie im Plugin-Manifest mit `channelEnvVars`. Behalten Sie Runtime-`envVars` des Channels oder lokale Konstanten nur für operatorseitigen Text.
`createOptionalChannelSetupWizard`, `DEFAULT_ACCOUNT_ID`,
`createTopLevelChannelDmPolicy`, `setSetupChannelEnabled` und
`splitSetupEntries`

- verwenden Sie die breitere Schnittstelle `openclaw/plugin-sdk/setup` nur dann, wenn Sie auch die schwergewichtigeren gemeinsam genutzten Setup-/Konfigurationshilfen benötigen, wie etwa
  `moveSingleAccountChannelSectionToDefaultAccount(...)`

Wenn Ihr Channel in Setup-Oberflächen nur „installieren Sie zuerst dieses Plugin“ anzeigen möchte, bevorzugen Sie `createOptionalChannelSetupSurface(...)`. Der generierte Adapter/Assistent schlägt bei Konfigurationsschreibvorgängen und Finalisierung sicher fehl und verwendet dieselbe Meldung „Installation erforderlich“ für Validierung, Finalisierung und Text mit Docs-Link erneut.

Für andere Hot-Path-Channel-Pfade sollten Sie schmale Hilfsfunktionen gegenüber breiteren Legacy-Oberflächen bevorzugen:

- `openclaw/plugin-sdk/account-core`,
  `openclaw/plugin-sdk/account-id`,
  `openclaw/plugin-sdk/account-resolution` und
  `openclaw/plugin-sdk/account-helpers` für Multi-Account-Konfiguration und
  Default-Account-Fallback
- `openclaw/plugin-sdk/inbound-envelope` und
  `openclaw/plugin-sdk/inbound-reply-dispatch` für das Routing/die Envelope eingehender Nachrichten und
  Record-and-Dispatch-Verdrahtung
- `openclaw/plugin-sdk/messaging-targets` für Zielanalyse/-abgleich
- `openclaw/plugin-sdk/outbound-media` und
  `openclaw/plugin-sdk/outbound-runtime` für Medienladen plus ausgehende
  Identitäts-/Sende-Delegates
- `openclaw/plugin-sdk/thread-bindings-runtime` für den Lifecycle von Thread-Bindings
  und die Adapter-Registrierung
- `openclaw/plugin-sdk/agent-media-payload` nur dann, wenn weiterhin ein Legacy-Layout
  für Agent-/Medien-Payload-Felder erforderlich ist
- `openclaw/plugin-sdk/telegram-command-config` für die Normalisierung benutzerdefinierter Telegram-Befehle,
  Duplikat-/Konfliktvalidierung und einen fallback-stabilen Vertrag für die Befehlskonfiguration

Channels nur mit Auth können normalerweise beim Standardpfad bleiben: Der Core verarbeitet Genehmigungen, und das Plugin stellt nur Outbound-/Auth-Fähigkeiten bereit. Native Genehmigungs-Channels wie Matrix, Slack, Telegram und benutzerdefinierte Chat-Transporte sollten die gemeinsam genutzten nativen Hilfsfunktionen verwenden, statt ihren eigenen Genehmigungs-Lifecycle zu implementieren.

## Richtlinie für eingehende Erwähnungen

Halten Sie die Verarbeitung eingehender Erwähnungen in zwei Schichten getrennt:

- plugin-eigene Evidenzerfassung
- gemeinsam genutzte Richtlinienauswertung

Verwenden Sie `openclaw/plugin-sdk/channel-inbound` für die gemeinsam genutzte Schicht.

Gut geeignet für plugin-lokale Logik:

- Erkennung von Antworten an den Bot
- Erkennung von Zitaten des Bots
- Prüfungen zur Thread-Beteiligung
- Ausschlüsse von Service-/Systemnachrichten
- plattformnative Caches, die benötigt werden, um die Beteiligung des Bots nachzuweisen

Gut geeignet für die gemeinsam genutzte Hilfsfunktion:

- `requireMention`
- explizites Erwähnungsergebnis
- Zulassungsliste für implizite Erwähnungen
- Command-Bypass
- endgültige Skip-Entscheidung

Bevorzugter Ablauf:

1. Lokale Erwähnungsfakten berechnen.
2. Diese Fakten an `resolveInboundMentionDecision({ facts, policy })` übergeben.
3. `decision.effectiveWasMentioned`, `decision.shouldBypassMention` und `decision.shouldSkip` in Ihrem Inbound-Gate verwenden.

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

`api.runtime.channel.mentions` stellt dieselben gemeinsam genutzten Hilfsfunktionen für Erwähnungen für gebündelte Channel-Plugins bereit, die bereits von Runtime-Injection abhängen:

- `buildMentionRegexes`
- `matchesMentionPatterns`
- `matchesMentionWithExplicit`
- `implicitMentionKindWhen`
- `resolveInboundMentionDecision`

Die älteren Hilfsfunktionen `resolveMentionGating*` bleiben auf
`openclaw/plugin-sdk/channel-inbound` nur als Kompatibilitätsexporte erhalten. Neuer Code sollte `resolveInboundMentionDecision({ facts, policy })` verwenden.

## Schritt-für-Schritt-Anleitung

<Steps>
  <a id="step-1-package-and-manifest"></a>
  <Step title="Paket und Manifest">
    Erstellen Sie die Standarddateien des Plugins. Das Feld `channel` in `package.json`
    macht dies zu einem Channel-Plugin. Die vollständige Oberfläche der Paketmetadaten
    finden Sie unter [Plugin-Setup und Konfiguration](/de/plugins/sdk-setup#openclaw-channel):

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

  <Step title="Das Channel-Plugin-Objekt erstellen">
    Das Interface `ChannelPlugin` hat viele optionale Adapter-Oberflächen. Beginnen Sie mit
    dem Minimum — `id` und `setup` — und fügen Sie nach Bedarf Adapter hinzu.

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

    <Accordion title="Was `createChatChannelPlugin` für Sie erledigt">
      Statt Low-Level-Adapter-Interfaces manuell zu implementieren, übergeben Sie
      deklarative Optionen, und der Builder setzt sie zusammen:

      | Option | Was verdrahtet wird |
      | --- | --- |
      | `security.dm` | Bereichsbezogener DM-Sicherheits-Resolver aus Konfigurationsfeldern |
      | `pairing.text` | Textbasierter DM-Pairing-Ablauf mit Codeaustausch |
      | `threading` | Reply-to-Modus-Resolver (fest, accountbezogen oder benutzerdefiniert) |
      | `outbound.attachedResults` | Sendefunktionen, die Ergebnis-Metadaten zurückgeben (Nachrichten-IDs) |

      Sie können statt der deklarativen Optionen auch rohe Adapter-Objekte übergeben,
      wenn Sie die vollständige Kontrolle benötigen.
    </Accordion>

  </Step>

  <Step title="Den Entry-Point verdrahten">
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

    Legen Sie Channel-eigene CLI-Deskriptoren in `registerCliMetadata(...)` ab, damit OpenClaw
    sie in der Root-Hilfe anzeigen kann, ohne die vollständige Channel-Runtime zu aktivieren,
    während normale vollständige Ladevorgänge dieselben Deskriptoren weiterhin für die echte
    Befehlsregistrierung übernehmen. Behalten Sie `registerFull(...)` für reine Runtime-Arbeit.
    Wenn `registerFull(...)` Gateway-RPC-Methoden registriert, verwenden Sie ein
    pluginspezifisches Präfix. Core-Admin-Namespaces (`config.*`,
    `exec.approvals.*`, `wizard.*`, `update.*`) bleiben reserviert und werden immer
    zu `operator.admin` aufgelöst.
    `defineChannelPluginEntry` übernimmt die Aufteilung des Registrierungsmodus automatisch. Alle
    Optionen finden Sie unter [Entry-Points](/de/plugins/sdk-entrypoints#definechannelpluginentry).

  </Step>

  <Step title="Einen Setup-Entry hinzufügen">
    Erstellen Sie `setup-entry.ts` für leichtgewichtiges Laden während des Onboardings:

    ```typescript setup-entry.ts
    import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineSetupPluginEntry(acmeChatPlugin);
    ```

    OpenClaw lädt diesen statt des vollständigen Entry-Points, wenn der Channel deaktiviert
    oder nicht konfiguriert ist. So wird verhindert, dass während Setup-Abläufen schwergewichtiger Runtime-Code geladen wird.
    Details finden Sie unter [Setup und Konfiguration](/de/plugins/sdk-setup#setup-entry).

  </Step>

  <Step title="Eingehende Nachrichten verarbeiten">
    Ihr Plugin muss Nachrichten von der Plattform empfangen und an
    OpenClaw weiterleiten. Das typische Muster ist ein Webhook, der die Anfrage überprüft und
    sie über den Inbound-Handler Ihres Channels weiterleitet:

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
      Die Verarbeitung eingehender Nachrichten ist channelspezifisch. Jedes Channel-Plugin besitzt
      seine eigene Inbound-Pipeline. Sehen Sie sich gebündelte Channel-Plugins
      (zum Beispiel das Plugin-Paket für Microsoft Teams oder Google Chat) für reale Muster an.
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

    Für gemeinsam genutzte Test-Hilfsfunktionen siehe [Testing](/de/plugins/sdk-testing).

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
├── runtime-api.ts            # Interne Runtime-Exporte (optional)
└── src/
    ├── channel.ts            # ChannelPlugin über createChatChannelPlugin
    ├── channel.test.ts       # Tests
    ├── client.ts             # Plattform-API-Client
    └── runtime.ts            # Runtime-Store (falls erforderlich)
```

## Erweiterte Themen

<CardGroup cols={2}>
  <Card title="Threading-Optionen" icon="git-branch" href="/de/plugins/sdk-entrypoints#registration-mode">
    Feste, accountbezogene oder benutzerdefinierte Antwortmodi
  </Card>
  <Card title="Integration des Message-Tools" icon="puzzle" href="/de/plugins/architecture#channel-plugins-and-the-shared-message-tool">
    describeMessageTool und Action-Erkennung
  </Card>
  <Card title="Zielauflösung" icon="crosshair" href="/de/plugins/architecture#channel-target-resolution">
    inferTargetChatType, looksLikeId, resolveTarget
  </Card>
  <Card title="Runtime-Hilfsfunktionen" icon="settings" href="/de/plugins/sdk-runtime">
    TTS, STT, Medien, Subagent über api.runtime
  </Card>
</CardGroup>

<Note>
Einige gebündelte Helper-Schnittstellen existieren weiterhin für die Pflege gebündelter Plugins und
Kompatibilität. Sie sind nicht das empfohlene Muster für neue Channel-Plugins;
bevorzugen Sie die generischen Unterpfade channel/setup/reply/runtime aus der gemeinsamen SDK-
Oberfläche, es sei denn, Sie pflegen diese gebündelte Plugin-Familie direkt.
</Note>

## Nächste Schritte

- [Provider-Plugins](/de/plugins/sdk-provider-plugins) — wenn Ihr Plugin auch Modelle bereitstellt
- [SDK-Überblick](/de/plugins/sdk-overview) — vollständige Importreferenz für Unterpfade
- [SDK-Testing](/de/plugins/sdk-testing) — Test-Utilities und Vertragstests
- [Plugin-Manifest](/de/plugins/manifest) — vollständiges Manifestschema
