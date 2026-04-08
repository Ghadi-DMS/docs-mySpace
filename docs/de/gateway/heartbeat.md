---
read_when:
    - Beim Anpassen der Heartbeat-Frequenz oder -Nachrichten
    - Bei der Entscheidung zwischen Heartbeat und cron für geplante Aufgaben
summary: Heartbeat-Abfragenachrichten und Benachrichtigungsregeln
title: Heartbeat
x-i18n:
    generated_at: "2026-04-08T02:15:16Z"
    model: gpt-5.4
    provider: openai
    source_hash: a8021d747637060eacb91ec5f75904368a08790c19f4fca32acda8c8c0a25e41
    source_path: gateway/heartbeat.md
    workflow: 15
---

# Heartbeat (Gateway)

> **Heartbeat oder Cron?** Siehe [Automation & Tasks](/de/automation) für Hinweise dazu, wann was verwendet werden sollte.

Heartbeat führt **periodische Agent-Runden** in der Hauptsitzung aus, damit das Modell alles hervorheben kann, was Aufmerksamkeit braucht, ohne Sie zuzuspammen.

Heartbeat ist eine geplante Hauptsitzungsrunde — es erstellt **keine** Einträge für [Hintergrundaufgaben](/de/automation/tasks).
Aufgabeneinträge sind für entkoppelte Arbeit gedacht (ACP-Läufe, Unter-Agenten, isolierte cron-Jobs).

Fehlerbehebung: [Scheduled Tasks](/de/automation/cron-jobs#troubleshooting)

## Schnellstart (für Einsteiger)

1. Lassen Sie Heartbeats aktiviert (Standard ist `30m`, oder `1h` bei Anthropic-OAuth-/Token-Authentifizierung, einschließlich Claude-CLI-Wiederverwendung) oder legen Sie Ihre eigene Frequenz fest.
2. Erstellen Sie eine kleine Checkliste in `HEARTBEAT.md` oder einen `tasks:`-Block im Agent-Workspace (optional, aber empfohlen).
3. Entscheiden Sie, wohin Heartbeat-Nachrichten gesendet werden sollen (`target: "none"` ist der Standard; setzen Sie `target: "last"`, um an den letzten Kontakt zu senden).
4. Optional: Aktivieren Sie die Zustellung von Heartbeat-Reasoning für mehr Transparenz.
5. Optional: Verwenden Sie leichten Bootstrap-Kontext, wenn Heartbeat-Läufe nur `HEARTBEAT.md` benötigen.
6. Optional: Aktivieren Sie isolierte Sitzungen, um zu vermeiden, dass bei jedem Heartbeat der vollständige Gesprächsverlauf gesendet wird.
7. Optional: Beschränken Sie Heartbeats auf aktive Stunden (Ortszeit).

Beispielkonfiguration:

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // explizite Zustellung an den letzten Kontakt (Standard ist "none")
        directPolicy: "allow", // Standard: direkte/DM-Ziele zulassen; auf "block" setzen, um zu unterdrücken
        lightContext: true, // optional: nur HEARTBEAT.md aus Bootstrap-Dateien einfügen
        isolatedSession: true, // optional: frische Sitzung bei jedem Lauf (kein Gesprächsverlauf)
        // activeHours: { start: "08:00", end: "24:00" },
        // includeReasoning: true, // optional: zusätzlich separate `Reasoning:`-Nachricht senden
      },
    },
  },
}
```

## Standardwerte

- Intervall: `30m` (oder `1h`, wenn Anthropic-OAuth-/Token-Authentifizierung der erkannte Authentifizierungsmodus ist, einschließlich Claude-CLI-Wiederverwendung). Setzen Sie `agents.defaults.heartbeat.every` oder pro Agent `agents.list[].heartbeat.every`; verwenden Sie `0m`, um zu deaktivieren.
- Prompt-Textkörper (konfigurierbar über `agents.defaults.heartbeat.prompt`):
  `Read HEARTBEAT.md if it exists (workspace context). Follow it strictly. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.`
- Der Heartbeat-Prompt wird **wortwörtlich** als Benutzernachricht gesendet. Der System-
  Prompt enthält nur dann einen Abschnitt „Heartbeat“, wenn Heartbeats für den
  Standard-Agent aktiviert sind und der Lauf intern entsprechend markiert ist.
- Wenn Heartbeats mit `0m` deaktiviert sind, lassen normale Läufe auch `HEARTBEAT.md`
  aus dem Bootstrap-Kontext weg, sodass das Modell keine Anweisungen sieht, die nur für Heartbeats gedacht sind.
- Aktive Stunden (`heartbeat.activeHours`) werden in der konfigurierten Zeitzone geprüft.
  Außerhalb des Fensters werden Heartbeats bis zum nächsten Tick innerhalb des Fensters übersprungen.

## Wofür der Heartbeat-Prompt da ist

Der Standard-Prompt ist absichtlich breit gehalten:

- **Hintergrundaufgaben**: „Consider outstanding tasks“ weist den Agenten an, ausstehende
  Nachverfolgungen (Posteingang, Kalender, Erinnerungen, anstehende Arbeit) zu prüfen und alles Dringende hervorzuheben.
- **Check-in beim Menschen**: „Checkup sometimes on your human during day time“ weist auf eine
  gelegentliche leichte „Brauchst du etwas?“-Nachricht hin, vermeidet aber nächtlichen Spam,
  indem Ihre konfigurierte lokale Zeitzone verwendet wird (siehe [/concepts/timezone](/de/concepts/timezone)).

Heartbeat kann auf abgeschlossene [Hintergrundaufgaben](/de/automation/tasks) reagieren, aber ein Heartbeat-Lauf selbst erstellt keinen Aufgabeneintrag.

Wenn ein Heartbeat etwas sehr Spezifisches tun soll (z. B. „Gmail-PubSub-
Statistiken prüfen“ oder „Gateway-Zustand verifizieren“), setzen Sie `agents.defaults.heartbeat.prompt` (oder
`agents.list[].heartbeat.prompt`) auf einen benutzerdefinierten Textkörper (wird wortwörtlich gesendet).

## Antwortvertrag

- Wenn nichts Aufmerksamkeit braucht, antworten Sie mit **`HEARTBEAT_OK`**.
- Während Heartbeat-Läufen behandelt OpenClaw `HEARTBEAT_OK` als Bestätigung, wenn es
  am **Anfang oder Ende** der Antwort erscheint. Das Token wird entfernt und die Antwort
  verworfen, wenn der verbleibende Inhalt **≤ `ackMaxChars`** ist (Standard: 300).
- Wenn `HEARTBEAT_OK` in der **Mitte** einer Antwort erscheint, wird es nicht speziell behandelt.
- Für Warnungen **kein** `HEARTBEAT_OK` einfügen; geben Sie nur den Warntext zurück.

Außerhalb von Heartbeats wird ein verirrtes `HEARTBEAT_OK` am Anfang/Ende einer Nachricht entfernt
und protokolliert; eine Nachricht, die nur aus `HEARTBEAT_OK` besteht, wird verworfen.

## Konfiguration

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m", // Standard: 30m (0m deaktiviert)
        model: "anthropic/claude-opus-4-6",
        includeReasoning: false, // Standard: false (separate Reasoning:-Nachricht senden, wenn verfügbar)
        lightContext: false, // Standard: false; true behält nur HEARTBEAT.md aus den Workspace-Bootstrap-Dateien
        isolatedSession: false, // Standard: false; true führt jeden Heartbeat in einer frischen Sitzung aus (kein Gesprächsverlauf)
        target: "last", // Standard: none | Optionen: last | none | <channel id> (Core oder Plugin, z. B. "bluebubbles")
        to: "+15551234567", // optionaler kanalspezifischer Override
        accountId: "ops-bot", // optionale Kanal-ID für Mehrkonten-Kanäle
        prompt: "Read HEARTBEAT.md if it exists (workspace context). Follow it strictly. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.",
        ackMaxChars: 300, // maximal erlaubte Zeichen nach HEARTBEAT_OK
      },
    },
  },
}
```

### Geltungsbereich und Priorität

- `agents.defaults.heartbeat` legt globales Heartbeat-Verhalten fest.
- `agents.list[].heartbeat` wird darüber zusammengeführt; wenn irgendein Agent einen `heartbeat`-Block hat, führen **nur diese Agenten** Heartbeats aus.
- `channels.defaults.heartbeat` legt Sichtbarkeitsstandards für alle Kanäle fest.
- `channels.<channel>.heartbeat` überschreibt die Kanalstandards.
- `channels.<channel>.accounts.<id>.heartbeat` (Mehrkonten-Kanäle) überschreibt pro Kanal-Einstellung.

### Heartbeats pro Agent

Wenn irgendein Eintrag in `agents.list[]` einen `heartbeat`-Block enthält, führen **nur diese Agenten**
Heartbeats aus. Der Block pro Agent wird auf `agents.defaults.heartbeat`
zusammengeführt (damit Sie gemeinsame Standards einmal setzen und pro Agent überschreiben können).

Beispiel: zwei Agenten, nur der zweite Agent führt Heartbeats aus.

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // explizite Zustellung an den letzten Kontakt (Standard ist "none")
      },
    },
    list: [
      { id: "main", default: true },
      {
        id: "ops",
        heartbeat: {
          every: "1h",
          target: "whatsapp",
          to: "+15551234567",
          prompt: "Read HEARTBEAT.md if it exists (workspace context). Follow it strictly. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.",
        },
      },
    ],
  },
}
```

### Beispiel für aktive Stunden

Beschränken Sie Heartbeats auf Geschäftszeiten in einer bestimmten Zeitzone:

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // explizite Zustellung an den letzten Kontakt (Standard ist "none")
        activeHours: {
          start: "09:00",
          end: "22:00",
          timezone: "America/New_York", // optional; verwendet Ihre userTimezone, falls gesetzt, sonst die Host-Zeitzone
        },
      },
    },
  },
}
```

Außerhalb dieses Fensters (vor 9 Uhr oder nach 22 Uhr Eastern Time) werden Heartbeats übersprungen. Der nächste geplante Tick innerhalb des Fensters läuft normal.

### 24/7-Einrichtung

Wenn Sie möchten, dass Heartbeats den ganzen Tag laufen, verwenden Sie eines dieser Muster:

- Lassen Sie `activeHours` vollständig weg (keine Beschränkung auf ein Zeitfenster; das ist das Standardverhalten).
- Setzen Sie ein ganztägiges Fenster: `activeHours: { start: "00:00", end: "24:00" }`.

Setzen Sie nicht dieselbe Zeit für `start` und `end` (zum Beispiel `08:00` bis `08:00`).
Das wird als Fenster mit Breite null behandelt, sodass Heartbeats immer übersprungen werden.

### Beispiel mit mehreren Konten

Verwenden Sie `accountId`, um ein bestimmtes Konto auf Mehrkonten-Kanälen wie Telegram anzusprechen:

```json5
{
  agents: {
    list: [
      {
        id: "ops",
        heartbeat: {
          every: "1h",
          target: "telegram",
          to: "12345678:topic:42", // optional: an ein bestimmtes Thema/einen bestimmten Thread senden
          accountId: "ops-bot",
        },
      },
    ],
  },
  channels: {
    telegram: {
      accounts: {
        "ops-bot": { botToken: "YOUR_TELEGRAM_BOT_TOKEN" },
      },
    },
  },
}
```

### Hinweise zu Feldern

- `every`: Heartbeat-Intervall (Dauerzeichenfolge; Standardeinheit = Minuten).
- `model`: optionales Modell-Override für Heartbeat-Läufe (`provider/model`).
- `includeReasoning`: wenn aktiviert, zusätzlich die separate Nachricht `Reasoning:` zustellen, wenn verfügbar (gleiche Form wie bei `/reasoning on`).
- `lightContext`: wenn true, verwenden Heartbeat-Läufe einen leichten Bootstrap-Kontext und behalten nur `HEARTBEAT.md` aus den Workspace-Bootstrap-Dateien.
- `isolatedSession`: wenn true, läuft jeder Heartbeat in einer frischen Sitzung ohne vorherigen Gesprächsverlauf. Verwendet dasselbe Isolationsmuster wie cron `sessionTarget: "isolated"`. Reduziert die Token-Kosten pro Heartbeat drastisch. Kombinieren Sie es mit `lightContext: true` für maximale Einsparungen. Das Zustellungsrouting nutzt weiterhin den Kontext der Hauptsitzung.
- `session`: optionaler Sitzungsschlüssel für Heartbeat-Läufe.
  - `main` (Standard): Agent-Hauptsitzung.
  - Expliziter Sitzungsschlüssel (aus `openclaw sessions --json` oder der [sessions CLI](/cli/sessions) kopieren).
  - Formate für Sitzungsschlüssel: siehe [Sessions](/de/concepts/session) und [Groups](/de/channels/groups).
- `target`:
  - `last`: an den zuletzt verwendeten externen Kanal zustellen.
  - expliziter Kanal: jede konfigurierte Kanal- oder Plugin-ID, zum Beispiel `discord`, `matrix`, `telegram` oder `whatsapp`.
  - `none` (Standard): den Heartbeat ausführen, aber **nicht extern zustellen**.
- `directPolicy`: steuert das Zustellungsverhalten für direkte/DM-Ziele:
  - `allow` (Standard): direkte/DM-Heartbeat-Zustellung zulassen.
  - `block`: direkte/DM-Zustellung unterdrücken (`reason=dm-blocked`).
- `to`: optionaler Empfänger-Override (kanalspezifische ID, z. B. E.164 für WhatsApp oder eine Telegram-Chat-ID). Für Telegram-Themen/Threads verwenden Sie `<chatId>:topic:<messageThreadId>`.
- `accountId`: optionale Konto-ID für Mehrkonten-Kanäle. Bei `target: "last"` gilt die Konto-ID für den aufgelösten letzten Kanal, falls dieser Konten unterstützt; andernfalls wird sie ignoriert. Wenn die Konto-ID keinem konfigurierten Konto für den aufgelösten Kanal entspricht, wird die Zustellung übersprungen.
- `prompt`: überschreibt den Standard-Prompt-Textkörper (wird nicht zusammengeführt).
- `ackMaxChars`: maximal erlaubte Zeichen nach `HEARTBEAT_OK` vor der Zustellung.
- `suppressToolErrorWarnings`: wenn true, werden Warn-Payloads für Tool-Fehler während Heartbeat-Läufen unterdrückt.
- `activeHours`: beschränkt Heartbeat-Läufe auf ein Zeitfenster. Objekt mit `start` (HH:MM, inklusiv; `00:00` für Tagesbeginn), `end` (HH:MM exklusiv; `24:00` für Tagesende erlaubt) und optional `timezone`.
  - Weggelassen oder `"user"`: verwendet Ihre `agents.defaults.userTimezone`, falls gesetzt, andernfalls die Zeitzone des Host-Systems.
  - `"local"`: verwendet immer die Zeitzone des Host-Systems.
  - Jeder IANA-Bezeichner (z. B. `America/New_York`): wird direkt verwendet; falls ungültig, wird auf das oben beschriebene Verhalten von `"user"` zurückgefallen.
  - `start` und `end` dürfen für ein aktives Fenster nicht gleich sein; gleiche Werte werden als Fenster mit Breite null behandelt (immer außerhalb des Fensters).
  - Außerhalb des aktiven Fensters werden Heartbeats bis zum nächsten Tick innerhalb des Fensters übersprungen.

## Zustellungsverhalten

- Heartbeats laufen standardmäßig in der Hauptsitzung des Agenten (`agent:<id>:<mainKey>`),
  oder in `global`, wenn `session.scope = "global"` ist. Setzen Sie `session`, um auf eine
  bestimmte Kanalsitzung (Discord/WhatsApp/etc.) zu überschreiben.
- `session` wirkt sich nur auf den Laufkontext aus; die Zustellung wird durch `target` und `to` gesteuert.
- Um an einen bestimmten Kanal/Empfänger zuzustellen, setzen Sie `target` + `to`. Mit
  `target: "last"` verwendet die Zustellung den letzten externen Kanal für diese Sitzung.
- Heartbeat-Zustellungen erlauben direkte/DM-Ziele standardmäßig. Setzen Sie `directPolicy: "block"`, um direkte Zielsendungen zu unterdrücken und trotzdem die Heartbeat-Runde auszuführen.
- Wenn die Hauptwarteschlange beschäftigt ist, wird der Heartbeat übersprungen und später erneut versucht.
- Wenn `target` zu keinem externen Ziel aufgelöst wird, findet der Lauf trotzdem statt, aber es
  wird keine ausgehende Nachricht gesendet.
- Wenn `showOk`, `showAlerts` und `useIndicator` alle deaktiviert sind, wird der Lauf vorab mit `reason=alerts-disabled` übersprungen.
- Wenn nur die Warnzustellung deaktiviert ist, kann OpenClaw den Heartbeat trotzdem ausführen, Zeitstempel fälliger Aufgaben aktualisieren, den Idle-Zeitstempel der Sitzung wiederherstellen und die nach außen gerichtete Warn-Payload unterdrücken.
- Heartbeat-only-Antworten halten die Sitzung **nicht** aktiv; das letzte `updatedAt`
  wird wiederhergestellt, sodass der Idle-Ablauf normal funktioniert.
- Entkoppelte [Hintergrundaufgaben](/de/automation/tasks) können ein Systemereignis in die Warteschlange stellen und den Heartbeat aufwecken, wenn die Hauptsitzung etwas schnell bemerken soll. Dieses Aufwecken macht den Heartbeat-Lauf nicht zu einer Hintergrundaufgabe.

## Sichtbarkeitssteuerung

Standardmäßig werden Bestätigungen mit `HEARTBEAT_OK` unterdrückt, während Warninhalte
zugestellt werden. Sie können dies pro Kanal oder pro Konto anpassen:

```yaml
channels:
  defaults:
    heartbeat:
      showOk: false # HEARTBEAT_OK ausblenden (Standard)
      showAlerts: true # Warnmeldungen anzeigen (Standard)
      useIndicator: true # Indikatorereignisse ausgeben (Standard)
  telegram:
    heartbeat:
      showOk: true # OK-Bestätigungen auf Telegram anzeigen
  whatsapp:
    accounts:
      work:
        heartbeat:
          showAlerts: false # Warnzustellung für dieses Konto unterdrücken
```

Priorität: pro Konto → pro Kanal → Kanalstandards → integrierte Standardwerte.

### Was jede Kennzeichnung bewirkt

- `showOk`: sendet eine Bestätigung `HEARTBEAT_OK`, wenn das Modell eine reine OK-Antwort zurückgibt.
- `showAlerts`: sendet den Warninhalt, wenn das Modell eine Nicht-OK-Antwort zurückgibt.
- `useIndicator`: gibt Indikatorereignisse für UI-Statusoberflächen aus.

Wenn **alle drei** false sind, überspringt OpenClaw den Heartbeat-Lauf vollständig (kein Modellaufruf).

### Beispiele pro Kanal vs. pro Konto

```yaml
channels:
  defaults:
    heartbeat:
      showOk: false
      showAlerts: true
      useIndicator: true
  slack:
    heartbeat:
      showOk: true # alle Slack-Konten
    accounts:
      ops:
        heartbeat:
          showAlerts: false # Warnungen nur für das ops-Konto unterdrücken
  telegram:
    heartbeat:
      showOk: true
```

### Häufige Muster

| Ziel                                     | Konfiguration                                                                            |
| ---------------------------------------- | ---------------------------------------------------------------------------------------- |
| Standardverhalten (stille OKs, Warnungen an) | _(keine Konfiguration erforderlich)_                                                     |
| Vollständig still (keine Nachrichten, kein Indikator) | `channels.defaults.heartbeat: { showOk: false, showAlerts: false, useIndicator: false }` |
| Nur Indikator (keine Nachrichten)        | `channels.defaults.heartbeat: { showOk: false, showAlerts: false, useIndicator: true }`  |
| OKs nur in einem Kanal                   | `channels.telegram.heartbeat: { showOk: true }`                                          |

## HEARTBEAT.md (optional)

Wenn im Workspace eine Datei `HEARTBEAT.md` vorhanden ist, weist der Standard-Prompt den
Agenten an, sie zu lesen. Betrachten Sie sie als Ihre „Heartbeat-Checkliste“: klein, stabil und
sicher, um sie alle 30 Minuten einzubinden.

Bei normalen Läufen wird `HEARTBEAT.md` nur eingefügt, wenn Heartbeat-Hinweise
für den Standard-Agent aktiviert sind. Wenn die Heartbeat-Frequenz mit `0m` deaktiviert wird oder
`includeSystemPromptSection: false` gesetzt ist, wird sie aus dem normalen Bootstrap-
Kontext weggelassen.

Wenn `HEARTBEAT.md` vorhanden, aber praktisch leer ist (nur leere Zeilen und Markdown-
Überschriften wie `# Heading`), überspringt OpenClaw den Heartbeat-Lauf, um API-Aufrufe zu sparen.
Dieses Überspringen wird als `reason=empty-heartbeat-file` gemeldet.
Wenn die Datei fehlt, läuft der Heartbeat trotzdem und das Modell entscheidet, was zu tun ist.

Halten Sie sie klein (kurze Checkliste oder Erinnerungen), um Prompt-Aufblähung zu vermeiden.

Beispiel für `HEARTBEAT.md`:

```md
# Heartbeat checklist

- Quick scan: anything urgent in inboxes?
- If it’s daytime, do a lightweight check-in if nothing else is pending.
- If a task is blocked, write down _what is missing_ and ask Peter next time.
```

### `tasks:`-Blöcke

`HEARTBEAT.md` unterstützt außerdem einen kleinen strukturierten `tasks:`-Block für
intervallbasierte Prüfungen innerhalb des Heartbeats selbst.

Beispiel:

```md
tasks:

- name: inbox-triage
  interval: 30m
  prompt: "Check for urgent unread emails and flag anything time sensitive."
- name: calendar-scan
  interval: 2h
  prompt: "Check for upcoming meetings that need prep or follow-up."

# Additional instructions

- Keep alerts short.
- If nothing needs attention after all due tasks, reply HEARTBEAT_OK.
```

Verhalten:

- OpenClaw parst den `tasks:`-Block und prüft jede Aufgabe anhand ihres eigenen `interval`.
- Nur **fällige** Aufgaben werden für diesen Tick in den Heartbeat-Prompt aufgenommen.
- Wenn keine Aufgaben fällig sind, wird der Heartbeat vollständig übersprungen (`reason=no-tasks-due`), um einen verschwendeten Modellaufruf zu vermeiden.
- Nicht-Aufgaben-Inhalt in `HEARTBEAT.md` bleibt erhalten und wird nach der Liste fälliger Aufgaben als zusätzlicher Kontext angehängt.
- Zeitstempel der letzten Ausführung von Aufgaben werden im Sitzungsstatus (`heartbeatTaskState`) gespeichert, sodass Intervalle normale Neustarts überdauern.
- Aufgabenzeitstempel werden erst fortgeschrieben, nachdem ein Heartbeat-Lauf seinen normalen Antwortpfad abgeschlossen hat. Übersprungene Läufe wegen `empty-heartbeat-file` / `no-tasks-due` markieren Aufgaben nicht als abgeschlossen.

Der Aufgabenmodus ist nützlich, wenn Sie eine Heartbeat-Datei für mehrere periodische Prüfungen verwenden möchten, ohne bei jedem Tick für alle zu bezahlen.

### Kann der Agent `HEARTBEAT.md` aktualisieren?

Ja — wenn Sie ihn darum bitten.

`HEARTBEAT.md` ist einfach eine normale Datei im Agent-Workspace, sodass Sie dem
Agenten (in einem normalen Chat) zum Beispiel Folgendes sagen können:

- „Aktualisiere `HEARTBEAT.md`, um eine tägliche Kalenderprüfung hinzuzufügen.“
- „Schreibe `HEARTBEAT.md` neu, damit sie kürzer ist und sich auf Nachverfolgungen im Posteingang konzentriert.“

Wenn Sie möchten, dass dies proaktiv geschieht, können Sie auch eine explizite Zeile in
Ihren Heartbeat-Prompt aufnehmen, zum Beispiel: „If the checklist becomes stale, update HEARTBEAT.md
with a better one.“

Sicherheitshinweis: Legen Sie keine Geheimnisse (API-Schlüssel, Telefonnummern, private Tokens) in
`HEARTBEAT.md` ab — sie wird Teil des Prompt-Kontexts.

## Manuelles Aufwecken (bei Bedarf)

Sie können ein Systemereignis in die Warteschlange stellen und einen sofortigen Heartbeat auslösen mit:

```bash
openclaw system event --text "Check for urgent follow-ups" --mode now
```

Wenn mehrere Agenten `heartbeat` konfiguriert haben, führt ein manuelles Aufwecken jeden dieser
Agent-Heartbeats sofort aus.

Verwenden Sie `--mode next-heartbeat`, um bis zum nächsten geplanten Tick zu warten.

## Reasoning-Zustellung (optional)

Standardmäßig stellen Heartbeats nur die endgültige „Antwort“-Payload zu.

Wenn Sie Transparenz möchten, aktivieren Sie:

- `agents.defaults.heartbeat.includeReasoning: true`

Wenn aktiviert, stellen Heartbeats zusätzlich eine separate Nachricht mit dem Präfix
`Reasoning:` zu (gleiche Form wie bei `/reasoning on`). Das kann nützlich sein, wenn der Agent
mehrere Sitzungen/Codexe verwaltet und Sie sehen möchten, warum er entschieden hat, Sie anzupingen
— es kann aber auch mehr interne Details preisgeben, als Sie möchten. In Gruppenchats sollten Sie dies besser ausgeschaltet lassen.

## Kostenbewusstsein

Heartbeat-Läufe führen vollständige Agent-Runden aus. Kürzere Intervalle verbrauchen mehr Tokens. Zur Kostensenkung:

- Verwenden Sie `isolatedSession: true`, um zu vermeiden, dass der vollständige Gesprächsverlauf gesendet wird (~100K Tokens auf ~2-5K pro Lauf).
- Verwenden Sie `lightContext: true`, um Bootstrap-Dateien auf nur `HEARTBEAT.md` zu beschränken.
- Setzen Sie ein günstigeres `model` (z. B. `ollama/llama3.2:1b`).
- Halten Sie `HEARTBEAT.md` klein.
- Verwenden Sie `target: "none"`, wenn Sie nur interne Statusaktualisierungen möchten.

## Verwandt

- [Automation & Tasks](/de/automation) — alle Automatisierungsmechanismen auf einen Blick
- [Background Tasks](/de/automation/tasks) — wie entkoppelte Arbeit nachverfolgt wird
- [Timezone](/de/concepts/timezone) — wie die Zeitzone die Heartbeat-Planung beeinflusst
- [Troubleshooting](/de/automation/cron-jobs#troubleshooting) — Fehlerbehebung bei Automatisierungsproblemen
