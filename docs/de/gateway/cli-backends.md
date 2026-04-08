---
read_when:
    - Sie möchten einen zuverlässigen Fallback, wenn API-Provider ausfallen
    - Sie verwenden Codex CLI oder andere lokale AI-CLIs und möchten sie wiederverwenden
    - Sie möchten die MCP-loopback-Bridge für den Tool-Zugriff von CLI-Backends verstehen
summary: 'CLI-Backends: lokaler AI-CLI-Fallback mit optionaler MCP-Tool-Bridge'
title: CLI-Backends
x-i18n:
    generated_at: "2026-04-08T02:14:35Z"
    model: gpt-5.4
    provider: openai
    source_hash: b0e8c41f5f5a8e34466f6b765e5c08585ef1788fa9e9d953257324bcc6cbc414
    source_path: gateway/cli-backends.md
    workflow: 15
---

# CLI-Backends (Fallback-Laufzeit)

OpenClaw kann **lokale AI-CLIs** als **reinen Text-Fallback** ausführen, wenn API-Provider nicht verfügbar sind,
Rate-Limits greifen oder sich vorübergehend fehlerhaft verhalten. Das ist bewusst konservativ gehalten:

- **OpenClaw-Tools werden nicht direkt injiziert**, aber Backends mit `bundleMcp: true`
  können Gateway-Tools über eine loopback-MCP-Bridge erhalten.
- **JSONL-Streaming** für CLIs, die es unterstützen.
- **Sitzungen werden unterstützt** (damit Folge-Turns kohärent bleiben).
- **Bilder können durchgereicht werden**, wenn die CLI Bildpfade akzeptiert.

Das ist als **Sicherheitsnetz** und nicht als primärer Pfad gedacht. Verwenden Sie es, wenn Sie
Textantworten möchten, die „immer funktionieren“, ohne auf externe APIs angewiesen zu sein.

Wenn Sie stattdessen eine vollständige Harness-Laufzeit mit ACP-Sitzungssteuerung, Hintergrundaufgaben,
Thread-/Konversationsbindung und persistenten externen Coding-Sitzungen möchten, verwenden Sie
[ACP Agents](/de/tools/acp-agents). CLI-Backends sind nicht ACP.

## Einsteigerfreundlicher Schnellstart

Sie können Codex CLI **ohne Konfiguration** verwenden (das gebündelte OpenAI-Plugin
registriert ein Standard-Backend):

```bash
openclaw agent --message "hi" --model codex-cli/gpt-5.4
```

Wenn Ihr Gateway unter launchd/systemd läuft und PATH minimal ist, fügen Sie nur den
Befehlspfad hinzu:

```json5
{
  agents: {
    defaults: {
      cliBackends: {
        "codex-cli": {
          command: "/opt/homebrew/bin/codex",
        },
      },
    },
  },
}
```

Das ist alles. Keine Schlüssel, keine zusätzliche Auth-Konfiguration über die CLI selbst hinaus erforderlich.

Wenn Sie ein gebündeltes CLI-Backend als **primären Nachrichten-Provider** auf einem
Gateway-Host verwenden, lädt OpenClaw jetzt automatisch das zugehörige gebündelte Plugin, wenn Ihre Konfiguration
dieses Backend explizit in einer Modellreferenz oder unter
`agents.defaults.cliBackends` referenziert.

## Verwendung als Fallback

Fügen Sie Ihrer Fallback-Liste ein CLI-Backend hinzu, damit es nur ausgeführt wird, wenn primäre Modelle fehlschlagen:

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["codex-cli/gpt-5.4"],
      },
      models: {
        "anthropic/claude-opus-4-6": { alias: "Opus" },
        "codex-cli/gpt-5.4": {},
      },
    },
  },
}
```

Hinweise:

- Wenn Sie `agents.defaults.models` (Allowlist) verwenden, müssen Sie dort auch Ihre CLI-Backend-Modelle einschließen.
- Wenn der primäre Provider fehlschlägt (Auth, Rate-Limits, Timeouts), versucht OpenClaw
  als Nächstes das CLI-Backend.

## Konfigurationsübersicht

Alle CLI-Backends befinden sich unter:

```
agents.defaults.cliBackends
```

Jeder Eintrag ist mit einer **Provider-ID** verschlüsselt (z. B. `codex-cli`, `my-cli`).
Die Provider-ID wird zur linken Seite Ihrer Modellreferenz:

```
<provider>/<model>
```

### Beispielkonfiguration

```json5
{
  agents: {
    defaults: {
      cliBackends: {
        "codex-cli": {
          command: "/opt/homebrew/bin/codex",
        },
        "my-cli": {
          command: "my-cli",
          args: ["--json"],
          output: "json",
          input: "arg",
          modelArg: "--model",
          modelAliases: {
            "claude-opus-4-6": "opus",
            "claude-sonnet-4-6": "sonnet",
          },
          sessionArg: "--session",
          sessionMode: "existing",
          sessionIdFields: ["session_id", "conversation_id"],
          systemPromptArg: "--system",
          systemPromptWhen: "first",
          imageArg: "--image",
          imageMode: "repeat",
          serialize: true,
        },
      },
    },
  },
}
```

## Funktionsweise

1. **Wählt ein Backend aus** anhand des Provider-Präfixes (`codex-cli/...`).
2. **Erstellt einen System-Prompt** mit demselben OpenClaw-Prompt und Workspace-Kontext.
3. **Führt die CLI** mit einer Sitzungs-ID aus (falls unterstützt), damit der Verlauf konsistent bleibt.
4. **Parst die Ausgabe** (JSON oder Klartext) und gibt den endgültigen Text zurück.
5. **Persistiert Sitzungs-IDs** pro Backend, damit Folgeanfragen dieselbe CLI-Sitzung wiederverwenden.

<Note>
Das gebündelte Anthropic-`claude-cli`-Backend wird wieder unterstützt. Mitarbeiter von Anthropic
haben uns mitgeteilt, dass die Nutzung von Claude CLI im OpenClaw-Stil wieder erlaubt ist, daher behandelt OpenClaw
die Nutzung von `claude -p` für diese Integration als genehmigt, solange Anthropic
keine neue Richtlinie veröffentlicht.
</Note>

## Sitzungen

- Wenn die CLI Sitzungen unterstützt, setzen Sie `sessionArg` (z. B. `--session-id`) oder
  `sessionArgs` (Platzhalter `{sessionId}`), wenn die ID in mehrere Flags
  eingefügt werden muss.
- Wenn die CLI einen **Resume-Subcommand** mit anderen Flags verwendet, setzen Sie
  `resumeArgs` (ersetzt `args` beim Fortsetzen) und optional `resumeOutput`
  (für nicht-JSON-Resumes).
- `sessionMode`:
  - `always`: sendet immer eine Sitzungs-ID (neue UUID, wenn keine gespeichert ist).
  - `existing`: sendet eine Sitzungs-ID nur, wenn zuvor eine gespeichert wurde.
  - `none`: sendet niemals eine Sitzungs-ID.

Hinweise zur Serialisierung:

- `serialize: true` hält Ausführungen auf derselben Lane geordnet.
- Die meisten CLIs serialisieren auf einer Provider-Lane.
- OpenClaw verwirft die Wiederverwendung gespeicherter CLI-Sitzungen, wenn sich der Auth-Status des Backends ändert, einschließlich erneuter Anmeldung, Token-Rotation oder geänderter Auth-Profil-Credentials.

## Bilder (Durchreichung)

Wenn Ihre CLI Bildpfade akzeptiert, setzen Sie `imageArg`:

```json5
imageArg: "--image",
imageMode: "repeat"
```

OpenClaw schreibt Base64-Bilder in temporäre Dateien. Wenn `imageArg` gesetzt ist, werden diese
Pfade als CLI-Argumente übergeben. Wenn `imageArg` fehlt, hängt OpenClaw die
Dateipfade an den Prompt an (Pfadinjektion), was für CLIs ausreicht, die lokale
Dateien automatisch aus einfachen Pfaden laden.

## Ein- / Ausgaben

- `output: "json"` (Standard) versucht, JSON zu parsen und Text + Sitzungs-ID zu extrahieren.
- Für Gemini CLI-JSON-Ausgabe liest OpenClaw den Antworttext aus `response` und
  die Nutzung aus `stats`, wenn `usage` fehlt oder leer ist.
- `output: "jsonl"` parst JSONL-Streams (zum Beispiel Codex CLI `--json`) und extrahiert die finale Agentennachricht sowie Sitzungs-
  Kennungen, wenn vorhanden.
- `output: "text"` behandelt stdout als endgültige Antwort.

Eingabemodi:

- `input: "arg"` (Standard) übergibt den Prompt als letztes CLI-Argument.
- `input: "stdin"` sendet den Prompt über stdin.
- Wenn der Prompt sehr lang ist und `maxPromptArgChars` gesetzt ist, wird stdin verwendet.

## Standardwerte (plugin-eigen)

Das gebündelte OpenAI-Plugin registriert außerdem einen Standard für `codex-cli`:

- `command: "codex"`
- `args: ["exec","--json","--color","never","--sandbox","workspace-write","--skip-git-repo-check"]`
- `resumeArgs: ["exec","resume","{sessionId}","--color","never","--sandbox","workspace-write","--skip-git-repo-check"]`
- `output: "jsonl"`
- `resumeOutput: "text"`
- `modelArg: "--model"`
- `imageArg: "--image"`
- `sessionMode: "existing"`

Das gebündelte Google-Plugin registriert außerdem einen Standard für `google-gemini-cli`:

- `command: "gemini"`
- `args: ["--output-format", "json", "--prompt", "{prompt}"]`
- `resumeArgs: ["--resume", "{sessionId}", "--output-format", "json", "--prompt", "{prompt}"]`
- `imageArg: "@"`
- `imagePathScope: "workspace"`
- `modelArg: "--model"`
- `sessionMode: "existing"`
- `sessionIdFields: ["session_id", "sessionId"]`

Voraussetzung: Die lokale Gemini CLI muss installiert und als
`gemini` auf `PATH` verfügbar sein (`brew install gemini-cli` oder
`npm install -g @google/gemini-cli`).

Hinweise zu Gemini-CLI-JSON:

- Der Antworttext wird aus dem JSON-Feld `response` gelesen.
- Die Nutzung fällt auf `stats` zurück, wenn `usage` fehlt oder leer ist.
- `stats.cached` wird in OpenClaw-`cacheRead` normalisiert.
- Wenn `stats.input` fehlt, leitet OpenClaw Eingabetokens aus
  `stats.input_tokens - stats.cached` ab.

Nur bei Bedarf überschreiben (häufig: absoluter `command`-Pfad).

## Plugin-eigene Standardwerte

CLI-Backend-Standards sind jetzt Teil der Plugin-Oberfläche:

- Plugins registrieren sie mit `api.registerCliBackend(...)`.
- Die Backend-`id` wird zum Provider-Präfix in Modellreferenzen.
- Benutzerkonfiguration unter `agents.defaults.cliBackends.<id>` überschreibt weiterhin den Plugin-Standard.
- Backend-spezifische Konfigurationsbereinigung bleibt über den optionalen
  Hook `normalizeConfig` plugin-eigen.

## Bundle-MCP-Overlays

CLI-Backends erhalten **keine** OpenClaw-Tool-Aufrufe direkt, aber ein Backend kann
sich mit `bundleMcp: true` für ein generiertes MCP-Konfigurations-Overlay anmelden.

Aktuelles gebündeltes Verhalten:

- `claude-cli`: generierte strikte MCP-Konfigurationsdatei
- `codex-cli`: Inline-Konfigurations-Overrides für `mcp_servers`
- `google-gemini-cli`: generierte Gemini-Systemeinstellungsdatei

Wenn Bundle MCP aktiviert ist, führt OpenClaw Folgendes aus:

- Es startet einen loopback-HTTP-MCP-Server, der Gateway-Tools für den CLI-Prozess bereitstellt.
- Es authentifiziert die Bridge mit einem sitzungsspezifischen Token (`OPENCLAW_MCP_TOKEN`).
- Es begrenzt den Tool-Zugriff auf den aktuellen Sitzungs-, Konto- und Kanal-Kontext.
- Es lädt aktivierte Bundle-MCP-Server für den aktuellen Workspace.
- Es führt sie mit jeder vorhandenen Backend-MCP-Konfigurations-/Einstellungsstruktur zusammen.
- Es schreibt die Startkonfiguration unter Verwendung des Backend-eigenen Integrationsmodus aus der besitzenden Erweiterung um.

Wenn keine MCP-Server aktiviert sind, injiziert OpenClaw dennoch eine strikte Konfiguration, wenn sich ein
Backend für Bundle MCP anmeldet, damit Hintergrundläufe isoliert bleiben.

## Einschränkungen

- **Keine direkten OpenClaw-Tool-Aufrufe.** OpenClaw injiziert keine Tool-Aufrufe in
  das CLI-Backend-Protokoll. Backends sehen Gateway-Tools nur, wenn sie sich für
  `bundleMcp: true` anmelden.
- **Streaming ist backend-spezifisch.** Einige Backends streamen JSONL; andere puffern
  bis zum Beenden.
- **Strukturierte Ausgaben** hängen vom JSON-Format der CLI ab.
- **Codex-CLI-Sitzungen** werden über Textausgabe fortgesetzt (kein JSONL), was weniger
  strukturiert ist als der anfängliche `--json`-Lauf. OpenClaw-Sitzungen funktionieren weiterhin
  normal.

## Fehlerbehebung

- **CLI nicht gefunden**: Setzen Sie `command` auf einen vollständigen Pfad.
- **Falscher Modellname**: Verwenden Sie `modelAliases`, um `provider/model` → CLI-Modell zuzuordnen.
- **Keine Sitzungskontinuität**: Stellen Sie sicher, dass `sessionArg` gesetzt ist und `sessionMode` nicht
  `none` ist (Codex CLI kann derzeit nicht mit JSON-Ausgabe fortsetzen).
- **Bilder werden ignoriert**: Setzen Sie `imageArg` (und prüfen Sie, ob die CLI Dateipfade unterstützt).
