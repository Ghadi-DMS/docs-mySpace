---
x-i18n:
    generated_at: "2026-04-08T02:18:30Z"
    model: gpt-5.4
    provider: openai
    source_hash: 0e156cc8e2fe946a0423862f937754a7caa1fe7e6863b50a80bff49a1c86e1e8
    source_path: refactor/qa.md
    workflow: 15
---

# QA-Refactor

Status: grundlegende Migration abgeschlossen.

## Ziel

OpenClaw-QA von einem Modell mit geteilten Definitionen auf eine Single Source of Truth umstellen:

- Szenario-Metadaten
- an das Modell gesendete Prompts
- Setup und Teardown
- Harness-Logik
- Assertions und Erfolgskriterien
- Artefakte und Hinweise für Berichte

Der gewünschte Endzustand ist ein generisches QA-Harness, das leistungsfähige Szenario-Definitionsdateien lädt, statt den Großteil des Verhaltens in TypeScript fest zu codieren.

## Aktueller Stand

Die primäre Single Source of Truth liegt jetzt in `qa/scenarios.md`.

Implementiert:

- `qa/scenarios.md`
  - kanonisches QA-Paket
  - Operator-Identität
  - Startmission
  - Szenario-Metadaten
  - Handler-Bindings
- `extensions/qa-lab/src/scenario-catalog.ts`
  - Markdown-Paketparser + zod-Validierung
- `extensions/qa-lab/src/qa-agent-bootstrap.ts`
  - Plan-Rendering aus dem Markdown-Paket
- `extensions/qa-lab/src/qa-agent-workspace.ts`
  - stellt generierte Kompatibilitätsdateien plus `QA_SCENARIOS.md` bereit
- `extensions/qa-lab/src/suite.ts`
  - wählt ausführbare Szenarien über in Markdown definierte Handler-Bindings aus
- QA-Bus-Protokoll + UI
  - generische Inline-Anhänge für Bild-/Video-/Audio-/Datei-Rendering

Verbleibende geteilte Oberflächen:

- `extensions/qa-lab/src/suite.ts`
  - enthält weiterhin den Großteil der ausführbaren benutzerdefinierten Handler-Logik
- `extensions/qa-lab/src/report.ts`
  - leitet die Berichtsstruktur weiterhin aus Laufzeitausgaben ab

Die Aufteilung der Single Source of Truth ist damit behoben, aber die Ausführung ist weiterhin größtenteils handlergestützt statt vollständig deklarativ.

## Wie die tatsächliche Szenario-Oberfläche aussieht

Beim Lesen der aktuellen Suite zeigen sich einige unterschiedliche Szenarioklassen.

### Einfache Interaktion

- Kanal-Baseline
- DM-Baseline
- Threaded-Follow-up
- Modellwechsel
- Weiterverfolgung von Genehmigungen
- Reaktion/Bearbeiten/Löschen

### Konfigurations- und Laufzeitmutation

- config patch beim Deaktivieren von Skills
- config apply bei Neustart-Aufwecken
- config restart bei Fähigkeitsumschaltung
- Laufzeit-Inventar-Drift-Prüfung

### Dateisystem- und Repo-Assertions

- Quell-/Dokumentations-Erkennungsbericht
- Lobster Invaders bauen
- Suche nach generierten Bildartefakten

### Speicher-Orchestrierung

- Speicherabruf
- Speicher-Tools im Kanalkontext
- Fallback bei Speicherfehlern
- Ranking von Sitzungsspeicher
- Isolierung von Thread-Speicher
- Sweep für Memory Dreaming

### Tool- und Plugin-Integration

- MCP plugin-tools-Aufruf
- Skill-Sichtbarkeit
- Skill-Hot-Install
- native Bildgenerierung
- Bild-Roundtrip
- Bildverständnis aus Anhang

### Mehrere Runden und mehrere Akteure

- Unter-Agent-Übergabe
- Unter-Agent-Fanout-Synthese
- Flows zur Wiederherstellung nach Neustart

Diese Kategorien sind wichtig, weil sie die DSL-Anforderungen bestimmen. Eine flache Liste aus Prompt + erwartetem Text reicht nicht aus.

## Richtung

### Single Source of Truth

Verwenden Sie `qa/scenarios.md` als verfasste Single Source of Truth.

Das Paket sollte bleiben:

- für Reviews menschenlesbar
- maschinenlesbar
- reichhaltig genug, um Folgendes zu steuern:
  - Suite-Ausführung
  - Bootstrap des QA-Workspace
  - Metadaten der QA-Lab-UI
  - Doku-/Discovery-Prompts
  - Berichtsgenerierung

### Bevorzugtes Authoring-Format

Markdown als Top-Level-Format mit strukturierter YAML darin verwenden.

Empfohlene Form:

- YAML-Frontmatter
  - id
  - title
  - surface
  - tags
  - docs refs
  - code refs
  - model/provider overrides
  - prerequisites
- Prosa-Abschnitte
  - objective
  - notes
  - debugging hints
- YAML-Blöcke in Fences
  - setup
  - steps
  - assertions
  - cleanup

Das bietet:

- bessere PR-Lesbarkeit als riesiges JSON
- reichhaltigeren Kontext als reines YAML
- striktes Parsing und zod-Validierung

Rohes JSON ist nur als generierte Zwischenform akzeptabel.

## Vorgeschlagene Form einer Szenariodatei

Beispiel:

````md
---
id: image-generation-roundtrip
title: Image generation roundtrip
surface: image
tags: [media, image, roundtrip]
models:
  primary: openai/gpt-5.4
requires:
  tools: [image_generate]
  plugins: [openai, qa-channel]
docsRefs:
  - docs/help/testing.md
  - docs/concepts/model-providers.md
codeRefs:
  - extensions/qa-lab/src/suite.ts
  - src/gateway/chat-attachments.ts
---

# Ziel

Prüfen, dass generierte Medien in der Follow-up-Runde wieder angehängt werden.

# Setup

```yaml scenario.setup
- action: config.patch
  patch:
    agents:
      defaults:
        imageGenerationModel:
          primary: openai/gpt-image-1
- action: session.create
  key: agent:qa:image-roundtrip
```

# Schritte

```yaml scenario.steps
- action: agent.send
  session: agent:qa:image-roundtrip
  message: |
    Bildgenerierungsprüfung: Generiere ein QA-Leuchtturmbild und fasse es in einem kurzen Satz zusammen.
- action: artifact.capture
  kind: generated-image
  promptSnippet: Bildgenerierungsprüfung
  saveAs: lighthouseImage
- action: agent.send
  session: agent:qa:image-roundtrip
  message: |
    Prüfung der Bildinspektion im Roundtrip: Beschreibe den generierten Leuchtturm-Anhang in einem kurzen Satz.
  attachments:
    - fromArtifact: lighthouseImage
```

# Erwartung

```yaml scenario.expect
- assert: outbound.textIncludes
  value: lighthouse
- assert: requestLog.matches
  where:
    promptIncludes: Prüfung der Bildinspektion im Roundtrip
  imageInputCountGte: 1
- assert: artifact.exists
  ref: lighthouseImage
```
````

## Runner-Fähigkeiten, die die DSL abdecken muss

Ausgehend von der aktuellen Suite braucht der generische Runner mehr als nur Prompt-Ausführung.

### Umgebungs- und Setup-Aktionen

- `bus.reset`
- `gateway.waitHealthy`
- `channel.waitReady`
- `session.create`
- `thread.create`
- `workspace.writeSkill`

### Agent-Runden-Aktionen

- `agent.send`
- `agent.wait`
- `bus.injectInbound`
- `bus.injectOutbound`

### Konfigurations- und Laufzeitaktionen

- `config.get`
- `config.patch`
- `config.apply`
- `gateway.restart`
- `tools.effective`
- `skills.status`

### Datei- und Artefaktaktionen

- `file.write`
- `file.read`
- `file.delete`
- `file.touchTime`
- `artifact.captureGeneratedImage`
- `artifact.capturePath`

### Speicher- und cron-Aktionen

- `memory.indexForce`
- `memory.searchCli`
- `doctor.memory.status`
- `cron.list`
- `cron.run`
- `cron.waitCompletion`
- `sessionTranscript.write`

### MCP-Aktionen

- `mcp.callTool`

### Assertions

- `outbound.textIncludes`
- `outbound.inThread`
- `outbound.notInRoot`
- `tool.called`
- `tool.notPresent`
- `skill.visible`
- `skill.disabled`
- `file.contains`
- `memory.contains`
- `requestLog.matches`
- `sessionStore.matches`
- `cron.managedPresent`
- `artifact.exists`

## Variablen und Artefakt-Referenzen

Die DSL muss gespeicherte Ausgaben und spätere Referenzen unterstützen.

Beispiele aus der aktuellen Suite:

- einen Thread erstellen und dann `threadId` wiederverwenden
- eine Sitzung erstellen und dann `sessionKey` wiederverwenden
- ein Bild generieren und dann die Datei in der nächsten Runde anhängen
- einen Wake-Marker-String generieren und dann prüfen, dass er später erscheint

Benötigte Fähigkeiten:

- `saveAs`
- `${vars.name}`
- `${artifacts.name}`
- typisierte Referenzen für Pfade, Sitzungsschlüssel, Thread-IDs, Marker, Tool-Ausgaben

Ohne Variablenunterstützung wird die Harness weiterhin Szenariologik zurück in TypeScript auslaufen lassen.

## Was als Escape Hatches bleiben sollte

Ein vollständig rein deklarativer Runner ist in Phase 1 nicht realistisch.

Einige Szenarien sind von Natur aus stark orchestrierungsintensiv:

- Sweep für Memory Dreaming
- config apply bei Neustart-Aufwecken
- config restart bei Fähigkeitsumschaltung
- Auflösung generierter Bildartefakte nach Zeitstempel/Pfad
- Auswertung von Discovery-Reports

Diese sollten vorerst explizite benutzerdefinierte Handler verwenden.

Empfohlene Regel:

- 85-90 % deklarativ
- explizite `customHandler`-Schritte für den schwierigen Rest
- nur benannte und dokumentierte benutzerdefinierte Handler
- kein anonymer Inline-Code in der Szenariodatei

So bleibt die generische Engine sauber und Fortschritt ist trotzdem möglich.

## Architekturänderung

### Aktuell

Szenario-Markdown ist bereits die Single Source of Truth für:

- Suite-Ausführung
- Bootstrap-Dateien des Workspace
- Szenariokatalog der QA-Lab-UI
- Berichtsmetadaten
- Discovery-Prompts

Generierte Kompatibilität:

- der bereitgestellte Workspace enthält weiterhin `QA_KICKOFF_TASK.md`
- der bereitgestellte Workspace enthält weiterhin `QA_SCENARIO_PLAN.md`
- der bereitgestellte Workspace enthält jetzt außerdem `QA_SCENARIOS.md`

## Refactor-Plan

### Phase 1: Loader und Schema

Erledigt.

- `qa/scenarios.md` hinzugefügt
- Parser für benannten Markdown-YAML-Paketinhalt hinzugefügt
- mit zod validiert
- Consumer auf das geparste Paket umgestellt
- `qa/seed-scenarios.json` und `qa/QA_KICKOFF_TASK.md` auf Repo-Ebene entfernt

### Phase 2: generische Engine

- `extensions/qa-lab/src/suite.ts` aufteilen in:
  - Loader
  - Engine
  - Action-Registry
  - Assertion-Registry
  - benutzerdefinierte Handler
- bestehende Hilfsfunktionen als Engine-Operationen beibehalten

Ergebnis:

- Engine führt einfache deklarative Szenarien aus

Mit Szenarien beginnen, die größtenteils aus Prompt + Warten + Assertion bestehen:

- Threaded-Follow-up
- Bildverständnis aus Anhang
- Skill-Sichtbarkeit und -Aufruf
- Kanal-Baseline

Ergebnis:

- erste echte in Markdown definierte Szenarien, die über die generische Engine ausgeliefert werden

### Phase 4: mittlere Szenarien migrieren

- Bildgenerierungs-Roundtrip
- Speicher-Tools im Kanalkontext
- Ranking von Sitzungsspeicher
- Unter-Agent-Übergabe
- Unter-Agent-Fanout-Synthese

Ergebnis:

- Variablen, Artefakte, Tool-Assertions und Request-Log-Assertions praxiserprobt

### Phase 5: schwierige Szenarien auf benutzerdefinierten Handlern belassen

- Sweep für Memory Dreaming
- config apply bei Neustart-Aufwecken
- config restart bei Fähigkeitsumschaltung
- Laufzeit-Inventar-Drift

Ergebnis:

- gleiches Authoring-Format, aber mit expliziten Custom-Step-Blöcken, wo nötig

### Phase 6: hart codierte Szenario-Map löschen

Sobald die Paketabdeckung gut genug ist:

- den Großteil des szenariospezifischen TypeScript-Branchings aus `extensions/qa-lab/src/suite.ts` entfernen

## Fake Slack / Unterstützung für Rich Media

Der aktuelle QA-Bus ist textzentriert.

Relevante Dateien:

- `extensions/qa-channel/src/protocol.ts`
- `extensions/qa-lab/src/bus-state.ts`
- `extensions/qa-lab/src/bus-queries.ts`
- `extensions/qa-lab/src/bus-server.ts`
- `extensions/qa-lab/web/src/ui-render.ts`

Heute unterstützt der QA-Bus:

- Text
- Reaktionen
- Threads

Inline-Medienanhänge werden noch nicht modelliert.

### Benötigter Transportvertrag

Ein generisches Modell für QA-Bus-Anhänge hinzufügen:

```ts
type QaBusAttachment = {
  id: string;
  kind: "image" | "video" | "audio" | "file";
  mimeType: string;
  fileName?: string;
  inline?: boolean;
  url?: string;
  contentBase64?: string;
  width?: number;
  height?: number;
  durationMs?: number;
  altText?: string;
  transcript?: string;
};
```

Dann `attachments?: QaBusAttachment[]` hinzufügen zu:

- `QaBusMessage`
- `QaBusInboundMessageInput`
- `QaBusOutboundMessageInput`

### Warum zuerst generisch

Kein nur auf Slack zugeschnittenes Medienmodell bauen.

Stattdessen:

- ein generisches QA-Transportmodell
- mehrere Renderer darauf aufbauend
  - aktueller QA-Lab-Chat
  - zukünftiges Fake-Slack-Web
  - beliebige andere Fake-Transport-Ansichten

Das verhindert doppelte Logik und sorgt dafür, dass Medienszenarien transportagnostisch bleiben.

### Erforderliche UI-Arbeit

Die QA-UI aktualisieren, um Folgendes zu rendern:

- Inline-Bildvorschau
- Inline-Audioplayer
- Inline-Videoplayer
- Chip für Dateianhänge

Die aktuelle UI kann bereits Threads und Reaktionen rendern, daher sollte das Rendern von Anhängen auf demselben Nachrichtenkartenmodell aufbauen können.

### Szenarioarbeit, die durch Medientransport ermöglicht wird

Sobald Anhänge durch den QA-Bus fließen, können wir reichhaltigere Fake-Chat-Szenarien hinzufügen:

- Inline-Bildantwort in Fake Slack
- Verständnis von Audioanhängen
- Verständnis von Videoanhängen
- gemischte Anhangsreihenfolge
- Thread-Antwort mit beibehaltenen Medien

## Empfehlung

Der nächste Implementierungsblock sollte sein:

1. Markdown-Szenario-Loader + zod-Schema hinzufügen
2. den aktuellen Katalog aus Markdown generieren
3. zuerst einige einfache Szenarien migrieren
4. generische Unterstützung für QA-Bus-Anhänge hinzufügen
5. Inline-Bilder in der QA-UI rendern
6. dann auf Audio und Video erweitern

Das ist der kleinste Weg, der beide Ziele belegt:

- generische in Markdown definierte QA
- reichhaltigere Fake-Messaging-Oberflächen

## Offene Fragen

- ob Szenariodateien eingebettete Markdown-Prompt-Templates mit Variableninterpolation erlauben sollten
- ob Setup/Cleanup benannte Abschnitte oder einfach geordnete Aktionslisten sein sollten
- ob Artefakt-Referenzen im Schema stark typisiert oder stringbasiert sein sollten
- ob benutzerdefinierte Handler in einer Registry oder in Registries pro Oberfläche leben sollten
- ob die generierte JSON-Kompatibilitätsdatei während der Migration eingecheckt bleiben sollte
