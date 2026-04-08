---
read_when:
    - Sie müssen Session-IDs, Transcript-JSONL oder Felder in sessions.json debuggen
    - Sie ändern das Verhalten der automatischen Kompaktierung oder fügen Housekeeping vor der Kompaktierung hinzu
    - Sie möchten Memory-Flushing oder stille System-Turns implementieren
summary: 'Tiefgehender Einblick: Session-Store + Transcripts, Lifecycle und Interna der (automatischen) Kompaktierung'
title: Tiefgehender Einblick in das Session-Management
x-i18n:
    generated_at: "2026-04-08T02:18:49Z"
    model: gpt-5.4
    provider: openai
    source_hash: cb1a4048646486693db8943a9e9c6c5bcb205f0ed532b34842de3d0346077454
    source_path: reference/session-management-compaction.md
    workflow: 15
---

# Session-Management und Kompaktierung (Deep Dive)

Dieses Dokument erklärt, wie OpenClaw Sessions Ende-zu-Ende verwaltet:

- **Session-Routing** (wie eingehende Nachrichten einer `sessionKey` zugeordnet werden)
- **Session-Store** (`sessions.json`) und was er erfasst
- **Persistenz von Transcripts** (`*.jsonl`) und ihre Struktur
- **Transcript-Hygiene** (anbieterspezifische Korrekturen vor Ausführungen)
- **Kontextgrenzen** (Kontextfenster vs. erfasste Tokens)
- **Kompaktierung** (manuelle + automatische Kompaktierung) und wo Arbeiten vor der Kompaktierung eingehängt werden
- **Stilles Housekeeping** (z. B. Memory-Schreibvorgänge, die keine für Nutzer sichtbare Ausgabe erzeugen sollen)

Wenn Sie zuerst einen Überblick auf höherer Ebene möchten, beginnen Sie mit:

- [/concepts/session](/de/concepts/session)
- [/concepts/compaction](/de/concepts/compaction)
- [/concepts/memory](/de/concepts/memory)
- [/concepts/memory-search](/de/concepts/memory-search)
- [/concepts/session-pruning](/de/concepts/session-pruning)
- [/reference/transcript-hygiene](/de/reference/transcript-hygiene)

---

## Quelle der Wahrheit: das Gateway

OpenClaw ist auf einen einzelnen **Gateway-Prozess** ausgelegt, der den Session-Status besitzt.

- UIs (macOS-App, web Control UI, TUI) sollten das Gateway nach Session-Listen und Token-Zählwerten abfragen.
- Im Remote-Modus liegen Session-Dateien auf dem Remote-Host; ein „Prüfen Ihrer lokalen Mac-Dateien“ spiegelt nicht wider, was das Gateway verwendet.

---

## Zwei Persistenzschichten

OpenClaw persistiert Sessions in zwei Schichten:

1. **Session-Store (`sessions.json`)**
   - Schlüssel/Wert-Zuordnung: `sessionKey -> SessionEntry`
   - Klein, veränderbar, sicher bearbeitbar (oder Einträge löschbar)
   - Erfasst Session-Metadaten (aktuelle Session-ID, letzte Aktivität, Umschalter, Token-Zählwerte usw.)

2. **Transcript (`<sessionId>.jsonl`)**
   - Append-only-Transcript mit Baumstruktur (Einträge haben `id` + `parentId`)
   - Speichert die eigentliche Unterhaltung + Tool-Aufrufe + Kompaktierungszusammenfassungen
   - Wird verwendet, um den Modellkontext für zukünftige Turns wiederaufzubauen

---

## Speicherorte auf dem Datenträger

Pro Agent auf dem Gateway-Host:

- Store: `~/.openclaw/agents/<agentId>/sessions/sessions.json`
- Transcripts: `~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl`
  - Telegram-Topic-Sessions: `.../<sessionId>-topic-<threadId>.jsonl`

OpenClaw löst diese über `src/config/sessions.ts` auf.

---

## Store-Wartung und Datenträgerkontrollen

Die Session-Persistenz verfügt über automatische Wartungssteuerungen (`session.maintenance`) für `sessions.json` und Transcript-Artefakte:

- `mode`: `warn` (Standard) oder `enforce`
- `pruneAfter`: Altersgrenze für veraltete Einträge (Standard `30d`)
- `maxEntries`: Obergrenze für Einträge in `sessions.json` (Standard `500`)
- `rotateBytes`: rotiert `sessions.json`, wenn es zu groß ist (Standard `10mb`)
- `resetArchiveRetention`: Aufbewahrung für Transcript-Archive `*.reset.<timestamp>` (Standard: identisch zu `pruneAfter`; `false` deaktiviert die Bereinigung)
- `maxDiskBytes`: optionales Budget für das Sessions-Verzeichnis
- `highWaterBytes`: optionales Ziel nach der Bereinigung (Standard `80%` von `maxDiskBytes`)

Durchsetzungsreihenfolge für Bereinigung bei Datenträgerbudget (`mode: "enforce"`):

1. Entfernen Sie zuerst die ältesten archivierten oder verwaisten Transcript-Artefakte.
2. Wenn das Ziel weiterhin überschritten wird, entfernen Sie die ältesten Session-Einträge und ihre Transcript-Dateien.
3. Fahren Sie fort, bis die Nutzung bei oder unter `highWaterBytes` liegt.

In `mode: "warn"` meldet OpenClaw mögliche Entfernungen, verändert aber den Store/die Dateien nicht.

Wartung bei Bedarf ausführen:

```bash
openclaw sessions cleanup --dry-run
openclaw sessions cleanup --enforce
```

---

## Cron-Sessions und Run-Logs

Isolierte Cron-Ausführungen erzeugen ebenfalls Session-Einträge/Transcripts, und dafür gibt es eigene Aufbewahrungssteuerungen:

- `cron.sessionRetention` (Standard `24h`) entfernt alte isolierte Cron-Run-Sessions aus dem Session-Store (`false` deaktiviert dies).
- `cron.runLog.maxBytes` + `cron.runLog.keepLines` kürzen Dateien `~/.openclaw/cron/runs/<jobId>.jsonl` (Standards: `2_000_000` Bytes und `2000` Zeilen).

---

## Session-Schlüssel (`sessionKey`)

Ein `sessionKey` identifiziert, _in welchem Gesprächs-Bucket_ Sie sich befinden (Routing + Isolation).

Häufige Muster:

- Haupt-/Direkt-Chat (pro Agent): `agent:<agentId>:<mainKey>` (Standard `main`)
- Gruppe: `agent:<agentId>:<channel>:group:<id>`
- Raum/Kanal (Discord/Slack): `agent:<agentId>:<channel>:channel:<id>` oder `...:room:<id>`
- Cron: `cron:<job.id>`
- Webhook: `hook:<uuid>` (sofern nicht überschrieben)

Die kanonischen Regeln sind unter [/concepts/session](/de/concepts/session) dokumentiert.

---

## Session-IDs (`sessionId`)

Jeder `sessionKey` verweist auf eine aktuelle `sessionId` (die Transcript-Datei, die die Unterhaltung fortsetzt).

Faustregeln:

- **Zurücksetzen** (`/new`, `/reset`) erstellt eine neue `sessionId` für diesen `sessionKey`.
- **Tägliches Zurücksetzen** (standardmäßig 4:00 Uhr lokale Zeit auf dem Gateway-Host) erstellt bei der nächsten Nachricht nach der Reset-Grenze eine neue `sessionId`.
- **Idle-Ablauf** (`session.reset.idleMinutes` oder Legacy-`session.idleMinutes`) erstellt eine neue `sessionId`, wenn nach dem Inaktivitätsfenster eine Nachricht eintrifft. Wenn täglich + idle beide konfiguriert sind, gilt, was zuerst abläuft.
- **Fork-Schutz für Thread-Parent** (`session.parentForkMaxTokens`, Standard `100000`) überspringt das Forken des Parent-Transcripts, wenn die Parent-Session bereits zu groß ist; der neue Thread beginnt frisch. Setzen Sie `0`, um dies zu deaktivieren.

Implementierungsdetail: Die Entscheidung erfolgt in `initSessionState()` in `src/auto-reply/reply/session.ts`.

---

## Schema des Session-Stores (`sessions.json`)

Der Werttyp des Stores ist `SessionEntry` in `src/config/sessions.ts`.

Wichtige Felder (nicht vollständig):

- `sessionId`: aktuelle Transcript-ID (Dateiname wird daraus abgeleitet, sofern `sessionFile` nicht gesetzt ist)
- `updatedAt`: Zeitstempel der letzten Aktivität
- `sessionFile`: optionale explizite Überschreibung des Transcript-Pfads
- `chatType`: `direct | group | room` (hilft UIs und Sende-Richtlinien)
- `provider`, `subject`, `room`, `space`, `displayName`: Metadaten für Gruppen-/Kanallabels
- Umschalter:
  - `thinkingLevel`, `verboseLevel`, `reasoningLevel`, `elevatedLevel`
  - `sendPolicy` (Überschreibung pro Session)
- Modellauswahl:
  - `providerOverride`, `modelOverride`, `authProfileOverride`
- Token-Zählwerte (Best-Effort / anbieterabhängig):
  - `inputTokens`, `outputTokens`, `totalTokens`, `contextTokens`
- `compactionCount`: wie oft die automatische Kompaktierung für diesen Session-Schlüssel abgeschlossen wurde
- `memoryFlushAt`: Zeitstempel des letzten Memory-Flushs vor der Kompaktierung
- `memoryFlushCompactionCount`: Kompaktierungszähler, als der letzte Flush ausgeführt wurde

Der Store kann sicher bearbeitet werden, aber das Gateway ist die Autorität: Es kann Einträge neu schreiben oder rehydrieren, während Sessions laufen.

---

## Transcript-Struktur (`*.jsonl`)

Transcripts werden vom `SessionManager` von `@mariozechner/pi-coding-agent` verwaltet.

Die Datei ist JSONL:

- Erste Zeile: Session-Header (`type: "session"`, enthält `id`, `cwd`, `timestamp`, optional `parentSession`)
- Danach: Session-Einträge mit `id` + `parentId` (Baum)

Bemerkenswerte Eintragstypen:

- `message`: Nutzer-/Assistant-/`toolResult`-Nachrichten
- `custom_message`: von Erweiterungen eingefügte Nachrichten, die _in_ den Modellkontext eingehen (können in der UI verborgen sein)
- `custom`: Erweiterungsstatus, der _nicht_ in den Modellkontext eingeht
- `compaction`: persistierte Kompaktierungszusammenfassung mit `firstKeptEntryId` und `tokensBefore`
- `branch_summary`: persistierte Zusammenfassung beim Navigieren in einem Baumzweig

OpenClaw „korrigiert“ Transcripts absichtlich **nicht**; das Gateway verwendet `SessionManager`, um sie zu lesen/zu schreiben.

---

## Kontextfenster vs. erfasste Tokens

Zwei verschiedene Konzepte sind wichtig:

1. **Kontextfenster des Modells**: harte Obergrenze pro Modell (Tokens, die für das Modell sichtbar sind)
2. **Zählwerte im Session-Store**: rollierende Statistiken, die in `sessions.json` geschrieben werden (verwendet für /status und Dashboards)

Wenn Sie Grenzwerte abstimmen:

- Das Kontextfenster kommt aus dem Modellkatalog (und kann per Konfiguration überschrieben werden).
- `contextTokens` im Store ist ein Laufzeitschätzwert/Berichtswert; behandeln Sie es nicht als strikte Garantie.

Mehr dazu unter [/token-use](/de/reference/token-use).

---

## Kompaktierung: was das ist

Kompaktierung fasst ältere Unterhaltung in einem persistenten `compaction`-Eintrag im Transcript zusammen und lässt aktuelle Nachrichten intakt.

Nach der Kompaktierung sehen zukünftige Turns:

- Die Kompaktierungszusammenfassung
- Nachrichten nach `firstKeptEntryId`

Kompaktierung ist **persistent** (anders als Session-Pruning). Siehe [/concepts/session-pruning](/de/concepts/session-pruning).

## Chunk-Grenzen bei der Kompaktierung und Tool-Paarung

Wenn OpenClaw ein langes Transcript in Kompaktierungs-Chunks aufteilt, hält es
Assistant-Tool-Aufrufe zusammen mit ihren passenden `toolResult`-Einträgen gepaart.

- Wenn die tokenbasierte Aufteilung zwischen einem Tool-Aufruf und seinem Ergebnis landet, verschiebt OpenClaw
  die Grenze auf die Assistant-Nachricht mit dem Tool-Aufruf, statt das
  Paar zu trennen.
- Wenn ein nachlaufender `toolResult`-Block den Chunk sonst über das Ziel hinausschieben würde,
  bewahrt OpenClaw diesen ausstehenden Tool-Block und lässt den nicht zusammengefassten Tail
  intakt.
- Abgebrochene/fehlerhafte Tool-Aufruf-Blöcke halten eine ausstehende Aufteilung nicht offen.

---

## Wann automatische Kompaktierung erfolgt (Pi-Laufzeit)

Im eingebetteten Pi-Agenten wird automatische Kompaktierung in zwei Fällen ausgelöst:

1. **Overflow-Wiederherstellung**: Das Modell gibt einen Kontext-Overflow-Fehler zurück
   (`request_too_large`, `context length exceeded`, `input exceeds the maximum
number of tokens`, `input token count exceeds the maximum number of input
tokens`, `input is too long for the model`, `ollama error: context length
exceeded` und ähnliche anbietergeformte Varianten) → kompaktieren → erneut versuchen.
2. **Schwellwert-Wartung**: nach einem erfolgreichen Turn, wenn:

`contextTokens > contextWindow - reserveTokens`

Dabei gilt:

- `contextWindow` ist das Kontextfenster des Modells
- `reserveTokens` ist der reservierte Puffer für Prompts + die nächste Modellausgabe

Dies sind Semantiken der Pi-Laufzeit (OpenClaw konsumiert die Ereignisse, aber Pi entscheidet, wann kompaktiert wird).

---

## Einstellungen für Kompaktierung (`reserveTokens`, `keepRecentTokens`)

Die Kompaktierungseinstellungen von Pi befinden sich in den Pi-Einstellungen:

```json5
{
  compaction: {
    enabled: true,
    reserveTokens: 16384,
    keepRecentTokens: 20000,
  },
}
```

OpenClaw erzwingt außerdem eine Sicherheitsuntergrenze für eingebettete Ausführungen:

- Wenn `compaction.reserveTokens < reserveTokensFloor` ist, erhöht OpenClaw diesen Wert.
- Die Standarduntergrenze ist `20000` Tokens.
- Setzen Sie `agents.defaults.compaction.reserveTokensFloor: 0`, um die Untergrenze zu deaktivieren.
- Wenn der Wert bereits höher ist, lässt OpenClaw ihn unverändert.

Warum: Es soll genug Puffer für mehrturniges „Housekeeping“ (wie Memory-Schreibvorgänge) bleiben, bevor Kompaktierung unvermeidlich wird.

Implementierung: `ensurePiCompactionReserveTokens()` in `src/agents/pi-settings.ts`
(aufgerufen aus `src/agents/pi-embedded-runner.ts`).

---

## Austauschbare Kompaktierungsanbieter

Plugins können über `registerCompactionProvider()` auf der Plugin-API einen Kompaktierungsanbieter registrieren. Wenn `agents.defaults.compaction.provider` auf eine registrierte Anbieter-ID gesetzt ist, delegiert die Safeguard-Erweiterung die Zusammenfassung an diesen Anbieter statt an die integrierte Pipeline `summarizeInStages`.

- `provider`: ID eines registrierten Kompaktierungsanbieter-Plugins. Nicht setzen für Standard-LLM-Zusammenfassung.
- Das Setzen von `provider` erzwingt `mode: "safeguard"`.
- Anbieter erhalten dieselben Kompaktierungsanweisungen und dieselbe Richtlinie zur Beibehaltung von Identifikatoren wie der integrierte Pfad.
- Die Safeguard-Funktion bewahrt weiterhin Kontext des Suffixes aktueller Turns und getrennter Turns nach der Anbieterausgabe.
- Wenn der Anbieter fehlschlägt oder ein leeres Ergebnis zurückgibt, fällt OpenClaw automatisch auf die integrierte LLM-Zusammenfassung zurück.
- Abort-/Timeout-Signale werden erneut ausgelöst (nicht unterdrückt), um die Abbruchanforderung des Aufrufers zu respektieren.

Quelle: `src/plugins/compaction-provider.ts`, `src/agents/pi-hooks/compaction-safeguard.ts`.

---

## Für Nutzer sichtbare Oberflächen

Sie können Kompaktierung und Session-Status über Folgendes beobachten:

- `/status` (in jeder Chat-Session)
- `openclaw status` (CLI)
- `openclaw sessions` / `sessions --json`
- Verbose-Modus: `🧹 Auto-compaction complete` + Kompaktierungszähler

---

## Stilles Housekeeping (`NO_REPLY`)

OpenClaw unterstützt „stille“ Turns für Hintergrundaufgaben, bei denen der Nutzer keine Zwischenausgabe sehen soll.

Konvention:

- Der Assistant beginnt seine Ausgabe mit dem exakten stillen Token `NO_REPLY` /
  `no_reply`, um „keine Antwort an den Nutzer ausliefern“ anzuzeigen.
- OpenClaw entfernt/unterdrückt dies in der Zustellungsschicht.
- Die Unterdrückung des exakten stillen Tokens ist nicht case-sensitive, daher zählen `NO_REPLY` und
  `no_reply` beide, wenn die gesamte Payload nur aus dem stillen Token besteht.
- Dies ist nur für echte Hintergrund-/Nicht-Zustellungs-Turns gedacht; es ist keine Abkürzung für
  gewöhnliche, umsetzbare Nutzeranfragen.

Seit `2026.1.10` unterdrückt OpenClaw außerdem **Entwurfs-/Typing-Streaming**, wenn ein
partieller Chunk mit `NO_REPLY` beginnt, damit stille Operationen nicht mitten im Turn
partielle Ausgabe durchsickern lassen.

---

## „Memory Flush“ vor der Kompaktierung (implementiert)

Ziel: Bevor automatische Kompaktierung erfolgt, einen stillen agentischen Turn ausführen, der dauerhaften
Status auf den Datenträger schreibt (z. B. `memory/YYYY-MM-DD.md` im Agent-Workspace), damit die Kompaktierung
kritischen Kontext nicht löschen kann.

OpenClaw verwendet den Ansatz **Flush vor dem Schwellwert**:

1. Überwachen Sie die Nutzung des Session-Kontexts.
2. Wenn sie einen „weichen Schwellwert“ überschreitet (unterhalb des Kompaktierungsschwellwerts von Pi), führen Sie eine stille
   Direktive „write memory now“ für den Agenten aus.
3. Verwenden Sie das exakte stille Token `NO_REPLY` / `no_reply`, damit der Nutzer
   nichts sieht.

Konfiguration (`agents.defaults.compaction.memoryFlush`):

- `enabled` (Standard: `true`)
- `softThresholdTokens` (Standard: `4000`)
- `prompt` (Nutzernachricht für den Flush-Turn)
- `systemPrompt` (zusätzlicher System-Prompt, der für den Flush-Turn angehängt wird)

Hinweise:

- Der Standard-Prompt/System-Prompt enthält einen `NO_REPLY`-Hinweis, um die
  Zustellung zu unterdrücken.
- Der Flush wird einmal pro Kompaktierungszyklus ausgeführt (erfasst in `sessions.json`).
- Der Flush wird nur für eingebettete Pi-Sessions ausgeführt (CLI-Backends überspringen ihn).
- Der Flush wird übersprungen, wenn der Session-Workspace schreibgeschützt ist (`workspaceAccess: "ro"` oder `"none"`).
- Unter [Memory](/de/concepts/memory) finden Sie das Dateilayout des Workspace und die Schreibmuster.

Pi stellt außerdem einen Hook `session_before_compact` in der Erweiterungs-API bereit, aber die
Flush-Logik von OpenClaw lebt heute auf der Gateway-Seite.

---

## Checkliste zur Fehlerbehebung

- Session-Schlüssel falsch? Beginnen Sie mit [/concepts/session](/de/concepts/session) und prüfen Sie den `sessionKey` in `/status`.
- Store stimmt nicht mit Transcript überein? Prüfen Sie den Gateway-Host und den Store-Pfad aus `openclaw status`.
- Kompaktierungs-Spam? Prüfen Sie:
  - Kontextfenster des Modells (zu klein)
  - Kompaktierungseinstellungen (`reserveTokens` zu hoch für das Modellfenster kann frühere Kompaktierung verursachen)
  - Aufblähung durch Tool-Ergebnisse: Aktivieren/optimieren Sie Session-Pruning
- Stille Turns lecken? Prüfen Sie, ob die Antwort mit `NO_REPLY` beginnt (case-insensitive, exaktes Token) und ob Sie eine Build-Version mit dem Fix zur Streaming-Unterdrückung verwenden.
