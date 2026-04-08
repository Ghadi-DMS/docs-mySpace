---
read_when:
    - Sie möchten verstehen, wie OpenClaw Modellkontext zusammenstellt
    - Sie wechseln zwischen der Legacy-Engine und einer Plugin-Engine
    - Sie erstellen ein Kontext-Engine-Plugin
summary: 'Kontext-Engine: steckbare Kontextzusammenstellung, Kompaktierung und Subagent-Lebenszyklus'
title: Kontext-Engine
x-i18n:
    generated_at: "2026-04-08T02:14:23Z"
    model: gpt-5.4
    provider: openai
    source_hash: e8290ac73272eee275bce8e481ac7959b65386752caa68044d0c6f3e450acfb1
    source_path: concepts/context-engine.md
    workflow: 15
---

# Kontext-Engine

Eine **Kontext-Engine** steuert, wie OpenClaw für jeden Lauf Modellkontext erstellt.
Sie entscheidet, welche Nachrichten einbezogen werden, wie ältere Verläufe zusammengefasst werden und wie
Kontext über Subagent-Grenzen hinweg verwaltet wird.

OpenClaw wird mit einer integrierten `legacy`-Engine ausgeliefert. Plugins können
alternative Engines registrieren, die den aktiven Lebenszyklus der Kontext-Engine ersetzen.

## Schnellstart

Prüfen Sie, welche Engine aktiv ist:

```bash
openclaw doctor
# or inspect config directly:
cat ~/.openclaw/openclaw.json | jq '.plugins.slots.contextEngine'
```

### Installieren eines Kontext-Engine-Plugins

Kontext-Engine-Plugins werden wie jedes andere OpenClaw-Plugin installiert. Installieren Sie es
zuerst und wählen Sie dann die Engine im Slot aus:

```bash
# Install from npm
openclaw plugins install @martian-engineering/lossless-claw

# Or install from a local path (for development)
openclaw plugins install -l ./my-context-engine
```

Aktivieren Sie dann das Plugin und wählen Sie es in Ihrer Konfiguration als aktive Engine aus:

```json5
// openclaw.json
{
  plugins: {
    slots: {
      contextEngine: "lossless-claw", // must match the plugin's registered engine id
    },
    entries: {
      "lossless-claw": {
        enabled: true,
        // Plugin-specific config goes here (see the plugin's docs)
      },
    },
  },
}
```

Starten Sie das Gateway nach der Installation und Konfiguration neu.

Um zurück zur integrierten Engine zu wechseln, setzen Sie `contextEngine` auf `"legacy"` (oder
entfernen Sie den Schlüssel vollständig — `"legacy"` ist der Standardwert).

## Funktionsweise

Jedes Mal, wenn OpenClaw einen Modell-Prompt ausführt, ist die Kontext-Engine an
vier Lebenszyklus-Punkten beteiligt:

1. **Ingest** — wird aufgerufen, wenn der Sitzung eine neue Nachricht hinzugefügt wird. Die Engine
   kann die Nachricht in ihrem eigenen Datenspeicher speichern oder indexieren.
2. **Assemble** — wird vor jedem Modelllauf aufgerufen. Die Engine gibt eine geordnete
   Menge von Nachrichten zurück (und eine optionale `systemPromptAddition`), die innerhalb
   des Token-Budgets bleibt.
3. **Compact** — wird aufgerufen, wenn das Kontextfenster voll ist oder wenn der Benutzer
   `/compact` ausführt. Die Engine fasst älteren Verlauf zusammen, um Speicherplatz freizugeben.
4. **After turn** — wird aufgerufen, nachdem ein Lauf abgeschlossen ist. Die Engine kann den Status persistieren,
   eine Hintergrund-Kompaktierung auslösen oder Indizes aktualisieren.

### Subagent-Lebenszyklus (optional)

OpenClaw ruft derzeit einen Hook für den Subagent-Lebenszyklus auf:

- **onSubagentEnded** — Aufräumen, wenn eine Subagent-Sitzung abgeschlossen wird oder bereinigt wird.

Der Hook `prepareSubagentSpawn` ist als Teil der Schnittstelle für die zukünftige Verwendung vorhanden,
wird aber von der Laufzeit derzeit noch nicht aufgerufen.

### Ergänzung des System-Prompts

Die Methode `assemble` kann einen String `systemPromptAddition` zurückgeben. OpenClaw
stellt diesen dem System-Prompt für den Lauf voran. Dadurch können Engines
dynamische Recall-Hinweise, Retrieval-Anweisungen oder kontextabhängige Hinweise einfügen,
ohne statische Workspace-Dateien zu benötigen.

## Die Legacy-Engine

Die integrierte `legacy`-Engine bewahrt das ursprüngliche Verhalten von OpenClaw:

- **Ingest**: no-op (der Sitzungsmanager übernimmt die Nachrichtenpersistenz direkt).
- **Assemble**: Durchreichen (die bestehende Pipeline sanitize → validate → limit
  in der Laufzeit übernimmt die Kontextzusammenstellung).
- **Compact**: delegiert an die integrierte Zusammenfassungs-Kompaktierung, die
  eine einzelne Zusammenfassung älterer Nachrichten erstellt und aktuelle Nachrichten unverändert beibehält.
- **After turn**: no-op.

Die Legacy-Engine registriert keine Tools und stellt keine `systemPromptAddition` bereit.

Wenn `plugins.slots.contextEngine` nicht gesetzt ist (oder auf `"legacy"` gesetzt ist), wird diese
Engine automatisch verwendet.

## Plugin-Engines

Ein Plugin kann über die Plugin-API eine Kontext-Engine registrieren:

```ts
import { buildMemorySystemPromptAddition } from "openclaw/plugin-sdk/core";

export default function register(api) {
  api.registerContextEngine("my-engine", () => ({
    info: {
      id: "my-engine",
      name: "My Context Engine",
      ownsCompaction: true,
    },

    async ingest({ sessionId, message, isHeartbeat }) {
      // Store the message in your data store
      return { ingested: true };
    },

    async assemble({ sessionId, messages, tokenBudget, availableTools, citationsMode }) {
      // Return messages that fit the budget
      return {
        messages: buildContext(messages, tokenBudget),
        estimatedTokens: countTokens(messages),
        systemPromptAddition: buildMemorySystemPromptAddition({
          availableTools: availableTools ?? new Set(),
          citationsMode,
        }),
      };
    },

    async compact({ sessionId, force }) {
      // Summarize older context
      return { ok: true, compacted: true };
    },
  }));
}
```

Aktivieren Sie es dann in der Konfiguration:

```json5
{
  plugins: {
    slots: {
      contextEngine: "my-engine",
    },
    entries: {
      "my-engine": {
        enabled: true,
      },
    },
  },
}
```

### Die ContextEngine-Schnittstelle

Erforderliche Mitglieder:

| Member             | Kind     | Purpose                                                  |
| ------------------ | -------- | -------------------------------------------------------- |
| `info`             | Property | Engine-ID, Name, Version und ob sie die Kompaktierung besitzt |
| `ingest(params)`   | Method   | Eine einzelne Nachricht speichern                        |
| `assemble(params)` | Method   | Kontext für einen Modelllauf erstellen (gibt `AssembleResult` zurück) |
| `compact(params)`  | Method   | Kontext zusammenfassen/reduzieren                        |

`assemble` gibt ein `AssembleResult` zurück mit:

- `messages` — die geordneten Nachrichten, die an das Modell gesendet werden.
- `estimatedTokens` (erforderlich, `number`) — die Schätzung der Engine für die Gesamtzahl
  der Tokens im zusammengestellten Kontext. OpenClaw verwendet dies für Entscheidungen über Kompaktierungsgrenzen
  und für Diagnoseberichte.
- `systemPromptAddition` (optional, `string`) — wird dem System-Prompt vorangestellt.

Optionale Mitglieder:

| Member                         | Kind   | Purpose                                                                                                         |
| ------------------------------ | ------ | --------------------------------------------------------------------------------------------------------------- |
| `bootstrap(params)`            | Method | Engine-Status für eine Sitzung initialisieren. Wird einmal aufgerufen, wenn die Engine eine Sitzung zum ersten Mal sieht (z. B. Verlauf importieren). |
| `ingestBatch(params)`          | Method | Einen abgeschlossenen Turn als Batch aufnehmen. Wird nach Abschluss eines Laufs aufgerufen, mit allen Nachrichten dieses Turns auf einmal. |
| `afterTurn(params)`            | Method | Lebenszyklusarbeit nach dem Lauf (Status persistieren, Hintergrund-Kompaktierung auslösen).                    |
| `prepareSubagentSpawn(params)` | Method | Gemeinsamen Status für eine Child-Sitzung einrichten.                                                          |
| `onSubagentEnded(params)`      | Method | Aufräumen, nachdem ein Subagent beendet wurde.                                                                 |
| `dispose()`                    | Method | Ressourcen freigeben. Wird während des Herunterfahrens des Gateways oder beim Neuladen des Plugins aufgerufen — nicht pro Sitzung. |

### ownsCompaction

`ownsCompaction` steuert, ob Pis integrierte automatische In-Attempt-Kompaktierung
für den Lauf aktiviert bleibt:

- `true` — die Engine besitzt das Kompaktierungsverhalten. OpenClaw deaktiviert Pis integrierte
  automatische Kompaktierung für diesen Lauf, und die Implementierung `compact()` der Engine ist
  verantwortlich für `/compact`, Kompaktierung zur Overflow-Wiederherstellung und jede proaktive
  Kompaktierung, die sie in `afterTurn()` ausführen möchte.
- `false` oder nicht gesetzt — Pis integrierte automatische Kompaktierung kann während der Prompt-
  Ausführung weiterhin laufen, aber die Methode `compact()` der aktiven Engine wird weiterhin für
  `/compact` und Overflow-Wiederherstellung aufgerufen.

`ownsCompaction: false` bedeutet **nicht**, dass OpenClaw automatisch auf
den Kompaktierungspfad der Legacy-Engine zurückfällt.

Das bedeutet, dass es zwei gültige Plugin-Muster gibt:

- **Owning-Modus** — implementieren Sie Ihren eigenen Kompaktierungsalgorithmus und setzen Sie
  `ownsCompaction: true`.
- **Delegating-Modus** — setzen Sie `ownsCompaction: false` und lassen Sie `compact()`
  `delegateCompactionToRuntime(...)` aus `openclaw/plugin-sdk/core` aufrufen, um
  das integrierte Kompaktierungsverhalten von OpenClaw zu verwenden.

Ein no-op-`compact()` ist für eine aktive nicht-besitzende Engine unsicher, weil es
den normalen Kompaktierungspfad für `/compact` und zur Overflow-Wiederherstellung für diesen
Engine-Slot deaktiviert.

## Konfigurationsreferenz

```json5
{
  plugins: {
    slots: {
      // Select the active context engine. Default: "legacy".
      // Set to a plugin id to use a plugin engine.
      contextEngine: "legacy",
    },
  },
}
```

Der Slot ist zur Laufzeit exklusiv — nur eine registrierte Kontext-Engine wird
für einen bestimmten Lauf oder Kompaktierungsvorgang aufgelöst. Andere aktivierte
Plugins mit `kind: "context-engine"` können weiterhin geladen werden und ihren Registrierungscode
ausführen; `plugins.slots.contextEngine` wählt nur aus, welche registrierte Engine-ID
OpenClaw auflöst, wenn eine Kontext-Engine benötigt wird.

## Beziehung zu Kompaktierung und Memory

- **Kompaktierung** ist eine Verantwortung der Kontext-Engine. Die Legacy-Engine
  delegiert an die integrierte Zusammenfassung von OpenClaw. Plugin-Engines können
  jede beliebige Kompaktierungsstrategie implementieren (DAG-Zusammenfassungen, Vektorretrieval usw.).
- **Memory-Plugins** (`plugins.slots.memory`) sind von Kontext-Engines getrennt.
  Memory-Plugins stellen Suche/Retrieval bereit; Kontext-Engines steuern, was das
  Modell sieht. Sie können zusammenarbeiten — eine Kontext-Engine könnte Memory-
  Plugin-Daten während der Zusammenstellung verwenden. Plugin-Engines, die den aktiven Memory-
  Prompt-Pfad verwenden möchten, sollten `buildMemorySystemPromptAddition(...)` aus
  `openclaw/plugin-sdk/core` bevorzugen; dies wandelt die aktiven Memory-Prompt-Abschnitte
  in eine sofort voranstellbare `systemPromptAddition` um. Wenn eine Engine eine Steuerung auf niedrigerer Ebene benötigt,
  kann sie weiterhin rohe Zeilen aus
  `openclaw/plugin-sdk/memory-host-core` über
  `buildActiveMemoryPromptSection(...)` abrufen.
- **Session-Pruning** (das Kürzen alter Tool-Ergebnisse im Arbeitsspeicher) läuft weiterhin,
  unabhängig davon, welche Kontext-Engine aktiv ist.

## Tipps

- Verwenden Sie `openclaw doctor`, um zu prüfen, ob Ihre Engine korrekt geladen wird.
- Wenn Sie Engines wechseln, werden bestehende Sitzungen mit ihrem aktuellen Verlauf fortgeführt.
  Die neue Engine übernimmt für zukünftige Läufe.
- Engine-Fehler werden protokolliert und in Diagnosen angezeigt. Wenn sich eine Plugin-Engine
  nicht registrieren lässt oder die ausgewählte Engine-ID nicht aufgelöst werden kann, fällt OpenClaw
  nicht automatisch zurück; Läufe schlagen fehl, bis Sie das Plugin reparieren oder
  `plugins.slots.contextEngine` wieder auf `"legacy"` setzen.
- Für die Entwicklung verwenden Sie `openclaw plugins install -l ./my-engine`, um ein
  lokales Plugin-Verzeichnis zu verknüpfen, ohne es zu kopieren.

Siehe auch: [Kompaktierung](/de/concepts/compaction), [Kontext](/de/concepts/context),
[Plugins](/de/tools/plugin), [Plugin-Manifest](/de/plugins/manifest).

## Verwandt

- [Kontext](/de/concepts/context) — wie Kontext für Agent-Turns erstellt wird
- [Plugin-Architektur](/de/plugins/architecture) — Registrieren von Kontext-Engine-Plugins
- [Kompaktierung](/de/concepts/compaction) — Zusammenfassen langer Konversationen
