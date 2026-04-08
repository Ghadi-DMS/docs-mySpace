---
read_when:
    - Ändern des Verhaltens von Gruppenchats oder der Erwähnungssteuerung
summary: Verhalten von Gruppenchats über Oberflächen hinweg (Discord/iMessage/Matrix/Microsoft Teams/Signal/Slack/Telegram/WhatsApp/Zalo)
title: Gruppen
x-i18n:
    generated_at: "2026-04-08T02:14:34Z"
    model: gpt-5.4
    provider: openai
    source_hash: f5045badbba30587c8f1bf27f6b940c7c471a95c57093c9adb142374413ac81e
    source_path: channels/groups.md
    workflow: 15
---

# Gruppen

OpenClaw behandelt Gruppenchats über Oberflächen hinweg konsistent: Discord, iMessage, Matrix, Microsoft Teams, Signal, Slack, Telegram, WhatsApp, Zalo.

## Einführung für Einsteiger (2 Minuten)

OpenClaw „lebt“ in deinen eigenen Messaging-Konten. Es gibt keinen separaten WhatsApp-Bot-Benutzer.
Wenn **du** in einer Gruppe bist, kann OpenClaw diese Gruppe sehen und dort antworten.

Standardverhalten:

- Gruppen sind eingeschränkt (`groupPolicy: "allowlist"`).
- Antworten erfordern eine Erwähnung, sofern du die Erwähnungssteuerung nicht ausdrücklich deaktivierst.

Übersetzt bedeutet das: Sender auf der Zulassungsliste können OpenClaw durch eine Erwähnung auslösen.

> Kurzfassung
>
> - **DM-Zugriff** wird über `*.allowFrom` gesteuert.
> - **Gruppenzugriff** wird über `*.groupPolicy` + Zulassungslisten (`*.groups`, `*.groupAllowFrom`) gesteuert.
> - **Antwortauslösung** wird über Erwähnungssteuerung (`requireMention`, `/activation`) gesteuert.

Schneller Ablauf (was mit einer Gruppennachricht passiert):

```
groupPolicy? disabled -> drop
groupPolicy? allowlist -> group allowed? no -> drop
requireMention? yes -> mentioned? no -> store for context only
otherwise -> reply
```

## Kontextsichtigkeit und Zulassungslisten

An der Sicherheit von Gruppen sind zwei unterschiedliche Steuerungen beteiligt:

- **Trigger-Autorisierung**: wer den Agenten auslösen kann (`groupPolicy`, `groups`, `groupAllowFrom`, kanalspezifische Zulassungslisten).
- **Kontextsichtigkeit**: welcher ergänzende Kontext in das Modell eingespeist wird (Antworttext, Zitate, Thread-Verlauf, weitergeleitete Metadaten).

Standardmäßig priorisiert OpenClaw normales Chatverhalten und behält den Kontext größtenteils so bei, wie er empfangen wurde. Das bedeutet, dass Zulassungslisten in erster Linie darüber entscheiden, wer Aktionen auslösen kann, und keine universelle Schwärzungsgrenze für jedes zitierte oder historische Snippet sind.

Das aktuelle Verhalten ist kanalspezifisch:

- Einige Kanäle wenden in bestimmten Pfaden bereits senderbasierte Filterung für ergänzenden Kontext an (zum Beispiel Slack-Thread-Seeding, Matrix-Antwort-/Thread-Lookups).
- Andere Kanäle reichen Zitat-/Antwort-/Weiterleitungskontext weiterhin so weiter, wie er empfangen wurde.

Richtung zur Härtung (geplant):

- `contextVisibility: "all"` (Standard) behält das aktuelle Verhalten wie empfangen bei.
- `contextVisibility: "allowlist"` filtert ergänzenden Kontext auf Sender in der Zulassungsliste.
- `contextVisibility: "allowlist_quote"` ist `allowlist` plus eine explizite Ausnahme für ein Zitat/eine Antwort.

Bis dieses Härtungsmodell konsistent über alle Kanäle hinweg implementiert ist, sind Unterschiede je nach Oberfläche zu erwarten.

![Ablauf von Gruppennachrichten](/images/groups-flow.svg)

Wenn du Folgendes möchtest...

| Ziel                                         | Was festgelegt werden soll                                 |
| -------------------------------------------- | ---------------------------------------------------------- |
| Alle Gruppen erlauben, aber nur auf @mentions antworten | `groups: { "*": { requireMention: true } }`                |
| Alle Gruppenantworten deaktivieren           | `groupPolicy: "disabled"`                                  |
| Nur bestimmte Gruppen                        | `groups: { "<group-id>": { ... } }` (kein `"*"`-Schlüssel) |
| Nur du kannst in Gruppen auslösen            | `groupPolicy: "allowlist"`, `groupAllowFrom: ["+1555..."]` |

## Sitzungsschlüssel

- Gruppensitzungen verwenden Sitzungsschlüssel vom Typ `agent:<agentId>:<channel>:group:<id>` (Räume/Kanäle verwenden `agent:<agentId>:<channel>:channel:<id>`).
- Telegram-Forenthemen fügen `:topic:<threadId>` zur Gruppen-ID hinzu, sodass jedes Thema eine eigene Sitzung hat.
- Direktchats verwenden die Hauptsitzung (oder bei entsprechender Konfiguration pro Sender).
- Heartbeats werden für Gruppensitzungen übersprungen.

<a id="pattern-personal-dms-public-groups-single-agent"></a>

## Muster: persönliche DMs + öffentliche Gruppen (einzelner Agent)

Ja — das funktioniert gut, wenn dein „persönlicher“ Verkehr **DMs** und dein „öffentlicher“ Verkehr **Gruppen** sind.

Warum: Im Einzelagentenmodus landen DMs typischerweise im **Haupt**-Sitzungsschlüssel (`agent:main:main`), während Gruppen immer **Nicht-Haupt**-Sitzungsschlüssel verwenden (`agent:main:<channel>:group:<id>`). Wenn du Sandboxing mit `mode: "non-main"` aktivierst, laufen diese Gruppensitzungen in Docker, während deine Haupt-DM-Sitzung auf dem Host bleibt.

Das gibt dir ein Agent-„Gehirn“ (gemeinsamer Workspace + Speicher), aber zwei Ausführungsmodi:

- **DMs**: vollständige Tools (Host)
- **Gruppen**: Sandbox + eingeschränkte Tools (Docker)

> Wenn du wirklich getrennte Workspaces/Personas brauchst („persönlich“ und „öffentlich“ dürfen sich nie vermischen), verwende einen zweiten Agenten + Bindings. Siehe [Multi-Agent Routing](/de/concepts/multi-agent).

Beispiel (DMs auf dem Host, Gruppen in der Sandbox + nur Messaging-Tools):

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main", // groups/channels are non-main -> sandboxed
        scope: "session", // strongest isolation (one container per group/channel)
        workspaceAccess: "none",
      },
    },
  },
  tools: {
    sandbox: {
      tools: {
        // If allow is non-empty, everything else is blocked (deny still wins).
        allow: ["group:messaging", "group:sessions"],
        deny: ["group:runtime", "group:fs", "group:ui", "nodes", "cron", "gateway"],
      },
    },
  },
}
```

Möchtest du statt „Gruppen können nur Ordner X sehen“ lieber „kein Host-Zugriff“? Behalte `workspaceAccess: "none"` bei und mounte nur Pfade aus der Zulassungsliste in die Sandbox:

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",
        scope: "session",
        workspaceAccess: "none",
        docker: {
          binds: [
            // hostPath:containerPath:mode
            "/home/user/FriendsShared:/data:ro",
          ],
        },
      },
    },
  },
}
```

Verwandt:

- Konfigurationsschlüssel und Standardwerte: [Gateway-Konfiguration](/de/gateway/configuration-reference#agentsdefaultssandbox)
- Debugging, warum ein Tool blockiert ist: [Sandbox vs Tool Policy vs Elevated](/de/gateway/sandbox-vs-tool-policy-vs-elevated)
- Details zu Bind-Mounts: [Sandboxing](/de/gateway/sandboxing#custom-bind-mounts)

## Anzeige-Labels

- UI-Labels verwenden `displayName`, wenn verfügbar, formatiert als `<channel>:<token>`.
- `#room` ist für Räume/Kanäle reserviert; Gruppenchats verwenden `g-<slug>` (Kleinbuchstaben, Leerzeichen -> `-`, `#@+._-` beibehalten).

## Gruppenrichtlinie

Steuere, wie Gruppen-/Raumnachrichten pro Kanal behandelt werden:

```json5
{
  channels: {
    whatsapp: {
      groupPolicy: "disabled", // "open" | "disabled" | "allowlist"
      groupAllowFrom: ["+15551234567"],
    },
    telegram: {
      groupPolicy: "disabled",
      groupAllowFrom: ["123456789"], // numeric Telegram user id (wizard can resolve @username)
    },
    signal: {
      groupPolicy: "disabled",
      groupAllowFrom: ["+15551234567"],
    },
    imessage: {
      groupPolicy: "disabled",
      groupAllowFrom: ["chat_id:123"],
    },
    msteams: {
      groupPolicy: "disabled",
      groupAllowFrom: ["user@org.com"],
    },
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        GUILD_ID: { channels: { help: { allow: true } } },
      },
    },
    slack: {
      groupPolicy: "allowlist",
      channels: { "#general": { allow: true } },
    },
    matrix: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["@owner:example.org"],
      groups: {
        "!roomId:example.org": { enabled: true },
        "#alias:example.org": { enabled: true },
      },
    },
  },
}
```

| Richtlinie    | Verhalten                                                    |
| ------------- | ------------------------------------------------------------ |
| `"open"`      | Gruppen umgehen Zulassungslisten; Erwähnungssteuerung gilt weiterhin. |
| `"disabled"`  | Blockiert alle Gruppennachrichten vollständig.               |
| `"allowlist"` | Erlaubt nur Gruppen/Räume, die der konfigurierten Zulassungsliste entsprechen. |

Hinweise:

- `groupPolicy` ist getrennt von der Erwähnungssteuerung (die @mentions erfordert).
- WhatsApp/Telegram/Signal/iMessage/Microsoft Teams/Zalo: verwenden `groupAllowFrom` (Fallback: explizites `allowFrom`).
- DM-Pairing-Freigaben (`*-allowFrom`-Store-Einträge) gelten nur für DM-Zugriff; die Autorisierung von Gruppensendern bleibt ausdrücklich an Gruppen-Zulassungslisten gebunden.
- Discord: Die Zulassungsliste verwendet `channels.discord.guilds.<id>.channels`.
- Slack: Die Zulassungsliste verwendet `channels.slack.channels`.
- Matrix: Die Zulassungsliste verwendet `channels.matrix.groups`. Bevorzuge Raum-IDs oder Aliasse; die Namensauflösung beigetretener Räume erfolgt nach bestem Bemühen, und nicht aufgelöste Namen werden zur Laufzeit ignoriert. Verwende `channels.matrix.groupAllowFrom`, um Sender einzuschränken; pro Raum werden auch `users`-Zulassungslisten unterstützt.
- Gruppen-DMs werden separat gesteuert (`channels.discord.dm.*`, `channels.slack.dm.*`).
- Die Telegram-Zulassungsliste kann Benutzer-IDs (`"123456789"`, `"telegram:123456789"`, `"tg:123456789"`) oder Benutzernamen (`"@alice"` oder `"alice"`) abgleichen; Präfixe sind nicht case-sensitive.
- Der Standard ist `groupPolicy: "allowlist"`; wenn deine Gruppen-Zulassungsliste leer ist, werden Gruppennachrichten blockiert.
- Laufzeitsicherheit: Wenn ein Provider-Block vollständig fehlt (`channels.<provider>` fehlt), fällt die Gruppenrichtlinie auf einen Fail-Closed-Modus zurück (typischerweise `allowlist`), statt `channels.defaults.groupPolicy` zu übernehmen.

Schnelles mentales Modell (Auswertungsreihenfolge für Gruppennachrichten):

1. `groupPolicy` (open/disabled/allowlist)
2. Gruppen-Zulassungslisten (`*.groups`, `*.groupAllowFrom`, kanalspezifische Zulassungsliste)
3. Erwähnungssteuerung (`requireMention`, `/activation`)

## Erwähnungssteuerung (Standard)

Gruppennachrichten erfordern eine Erwähnung, sofern dies nicht pro Gruppe überschrieben wird. Standardwerte liegen pro Subsystem unter `*.groups."*"`.

Eine Antwort auf eine Bot-Nachricht zählt als implizite Erwähnung, wenn der Kanal
Antwort-Metadaten unterstützt. Das Zitieren einer Bot-Nachricht kann ebenfalls als implizite
Erwähnung zählen, auf Kanälen, die Zitat-Metadaten bereitstellen. Zu den aktuellen integrierten Fällen gehören
Telegram, WhatsApp, Slack, Discord, Microsoft Teams und ZaloUser.

```json5
{
  channels: {
    whatsapp: {
      groups: {
        "*": { requireMention: true },
        "123@g.us": { requireMention: false },
      },
    },
    telegram: {
      groups: {
        "*": { requireMention: true },
        "123456789": { requireMention: false },
      },
    },
    imessage: {
      groups: {
        "*": { requireMention: true },
        "123": { requireMention: false },
      },
    },
  },
  agents: {
    list: [
      {
        id: "main",
        groupChat: {
          mentionPatterns: ["@openclaw", "openclaw", "\\+15555550123"],
          historyLimit: 50,
        },
      },
    ],
  },
}
```

Hinweise:

- `mentionPatterns` sind case-insensitive sichere Regex-Muster; ungültige Muster und unsichere Formen verschachtelter Wiederholungen werden ignoriert.
- Oberflächen, die explizite Erwähnungen bereitstellen, funktionieren weiterhin; Muster sind ein Fallback.
- Überschreibung pro Agent: `agents.list[].groupChat.mentionPatterns` (nützlich, wenn mehrere Agenten sich eine Gruppe teilen).
- Erwähnungssteuerung wird nur erzwungen, wenn Erwähnungserkennung möglich ist (native Erwähnungen oder konfigurierte `mentionPatterns`).
- Discord-Standards liegen in `channels.discord.guilds."*"` (überschreibbar pro Guild/Kanal).
- Gruppenverlaufs-Kontext wird kanalübergreifend einheitlich verpackt und ist **nur ausstehend** (Nachrichten, die wegen Erwähnungssteuerung übersprungen wurden); verwende `messages.groupChat.historyLimit` für den globalen Standard und `channels.<channel>.historyLimit` (oder `channels.<channel>.accounts.*.historyLimit`) für Überschreibungen. Setze `0`, um dies zu deaktivieren.

## Tool-Einschränkungen für Gruppen/Kanäle (optional)

Einige Kanalkonfigurationen unterstützen Einschränkungen, welche Tools **innerhalb einer bestimmten Gruppe/eines bestimmten Raums/eines bestimmten Kanals** verfügbar sind.

- `tools`: erlauben/verbieten Tools für die gesamte Gruppe.
- `toolsBySender`: Überschreibungen pro Sender innerhalb der Gruppe.
  Verwende explizite Schlüsselpräfixe:
  `id:<senderId>`, `e164:<phone>`, `username:<handle>`, `name:<displayName>` und das Platzhalterzeichen `"*"`.
  Alte Schlüssel ohne Präfix werden weiterhin akzeptiert und nur als `id:` abgeglichen.

Auflösungsreihenfolge (das Spezifischste gewinnt):

1. Treffer in `toolsBySender` der Gruppe/des Kanals
2. `tools` der Gruppe/des Kanals
3. Treffer in `toolsBySender` des Standards (`"*"` )
4. `tools` des Standards (`"*"`)

Beispiel (Telegram):

```json5
{
  channels: {
    telegram: {
      groups: {
        "*": { tools: { deny: ["exec"] } },
        "-1001234567890": {
          tools: { deny: ["exec", "read", "write"] },
          toolsBySender: {
            "id:123456789": { alsoAllow: ["exec"] },
          },
        },
      },
    },
  },
}
```

Hinweise:

- Tool-Einschränkungen für Gruppen/Kanäle werden zusätzlich zur globalen/agentenspezifischen Tool-Richtlinie angewendet (deny hat weiterhin Vorrang).
- Einige Kanäle verwenden eine andere Verschachtelung für Räume/Kanäle (z. B. Discord `guilds.*.channels.*`, Slack `channels.*`, Microsoft Teams `teams.*.channels.*`).

## Gruppen-Zulassungslisten

Wenn `channels.whatsapp.groups`, `channels.telegram.groups` oder `channels.imessage.groups` konfiguriert ist, fungieren die Schlüssel als Gruppen-Zulassungsliste. Verwende `"*"` , um alle Gruppen zuzulassen und gleichzeitig das Standardverhalten für Erwähnungen festzulegen.

Häufige Verwirrung: Eine DM-Pairing-Freigabe ist nicht dasselbe wie Gruppenautorisierung.
Bei Kanälen, die DM-Pairing unterstützen, schaltet der Pairing-Speicher nur DMs frei. Gruppenbefehle erfordern weiterhin eine explizite Autorisierung von Gruppensendern aus Konfigurations-Zulassungslisten wie `groupAllowFrom` oder dem dokumentierten Konfigurations-Fallback für diesen Kanal.

Häufige Absichten (kopieren/einfügen):

1. Alle Gruppenantworten deaktivieren

```json5
{
  channels: { whatsapp: { groupPolicy: "disabled" } },
}
```

2. Nur bestimmte Gruppen zulassen (WhatsApp)

```json5
{
  channels: {
    whatsapp: {
      groups: {
        "123@g.us": { requireMention: true },
        "456@g.us": { requireMention: false },
      },
    },
  },
}
```

3. Alle Gruppen zulassen, aber Erwähnung verlangen (explizit)

```json5
{
  channels: {
    whatsapp: {
      groups: { "*": { requireMention: true } },
    },
  },
}
```

4. Nur der Eigentümer kann in Gruppen auslösen (WhatsApp)

```json5
{
  channels: {
    whatsapp: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15551234567"],
      groups: { "*": { requireMention: true } },
    },
  },
}
```

## Aktivierung (nur Eigentümer)

Gruppeneigentümer können die Aktivierung pro Gruppe umschalten:

- `/activation mention`
- `/activation always`

Der Eigentümer wird über `channels.whatsapp.allowFrom` bestimmt (oder die eigene E.164 des Bots, wenn nicht gesetzt). Sende den Befehl als eigenständige Nachricht. Andere Oberflächen ignorieren `/activation` derzeit.

## Kontextfelder

Eingehende Gruppennutzlasten setzen:

- `ChatType=group`
- `GroupSubject` (falls bekannt)
- `GroupMembers` (falls bekannt)
- `WasMentioned` (Ergebnis der Erwähnungssteuerung)
- Telegram-Forenthemen enthalten außerdem `MessageThreadId` und `IsForum`.

Kanalspezifische Hinweise:

- BlueBubbles kann optional unbenannte macOS-Gruppenteilnehmer aus der lokalen Kontakte-Datenbank anreichern, bevor `GroupMembers` befüllt wird. Das ist standardmäßig deaktiviert und läuft nur, nachdem die normale Gruppensteuerung erfolgreich passiert wurde.

Der Agent-System-Prompt enthält im ersten Zug einer neuen Gruppensitzung eine Gruppeneinführung. Sie erinnert das Modell daran, wie ein Mensch zu antworten, Markdown-Tabellen zu vermeiden, leere Zeilen zu minimieren, normalem Chat-Abstand zu folgen und keine literalen `\n`-Sequenzen zu tippen.

## iMessage-spezifisches

- Bevorzuge `chat_id:<id>` beim Routing oder für Zulassungslisten.
- Chats auflisten: `imsg chats --limit 20`.
- Gruppenantworten gehen immer an dieselbe `chat_id` zurück.

## WhatsApp-spezifisches

Siehe [Gruppennachrichten](/de/channels/group-messages) für WhatsApp-spezifisches Verhalten (Verlaufsinjektion, Details zur Erwähnungsbehandlung).
