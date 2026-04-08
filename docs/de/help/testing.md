---
read_when:
    - Beim lokalen Ausführen von Tests oder in CI
    - Beim Hinzufügen von Regressionstests für Modell-/Provider-Fehler
    - Beim Debuggen von Gateway- und Agent-Verhalten
summary: 'Test-Kit: Unit-/E2E-/Live-Suiten, Docker-Runner und was jeder Test abdeckt'
title: Tests
x-i18n:
    generated_at: "2026-04-08T02:17:53Z"
    model: gpt-5.4
    provider: openai
    source_hash: ace2c19bfc350780475f4348264a4b55be2b4ccbb26f0d33b4a6af34510943b5
    source_path: help/testing.md
    workflow: 15
---

# Tests

OpenClaw hat drei Vitest-Suiten (Unit/Integration, E2E, Live) und eine kleine Gruppe von Docker-Runnern.

Diese Dokumentation ist ein Leitfaden dazu, **wie wir testen**:

- Was jede Suite abdeckt (und was sie bewusst _nicht_ abdeckt)
- Welche Befehle für gängige Workflows ausgeführt werden sollten (lokal, vor dem Push, Debugging)
- Wie Live-Tests Anmeldedaten erkennen und Modelle/Provider auswählen
- Wie Regressionstests für reale Modell-/Provider-Probleme hinzugefügt werden

## Schnellstart

An den meisten Tagen:

- Vollständiges Gate (vor einem Push erwartet): `pnpm build && pnpm check && pnpm test`
- Schnellere vollständige lokale Ausführung auf einem leistungsfähigen Rechner: `pnpm test:max`
- Direkte Vitest-Watch-Schleife: `pnpm test:watch`
- Direktes Targeting einzelner Dateien routet jetzt auch Extension-/Kanalpfade: `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts`
- Docker-gestützte QA-Site: `pnpm qa:lab:up`

Wenn Sie Tests ändern oder zusätzliche Sicherheit möchten:

- Coverage-Gate: `pnpm test:coverage`
- E2E-Suite: `pnpm test:e2e`

Beim Debuggen realer Provider/Modelle (erfordert echte Zugangsdaten):

- Live-Suite (Modelle + Gateway-Tool-/Bild-Probes): `pnpm test:live`
- Eine einzelne Live-Datei leise ausführen: `pnpm test:live -- src/agents/models.profiles.live.test.ts`

Tipp: Wenn Sie nur einen fehlschlagenden Fall brauchen, grenzen Sie Live-Tests bevorzugt über die unten beschriebenen Allowlist-Umgebungsvariablen ein.

## Test-Suiten (was wo läuft)

Betrachten Sie die Suiten als „zunehmend realistisch“ (und zunehmend anfällig/kostenintensiv):

### Unit / Integration (Standard)

- Befehl: `pnpm test`
- Konfiguration: zehn sequenzielle Shard-Läufe (`vitest.full-*.config.ts`) über die vorhandenen scoped Vitest-Projekte
- Dateien: Core-/Unit-Inventare unter `src/**/*.test.ts`, `packages/**/*.test.ts`, `test/**/*.test.ts` und die per `vitest.unit.config.ts` freigegebenen `ui`-Node-Tests
- Umfang:
  - Reine Unit-Tests
  - In-Process-Integrationstests (Gateway-Authentifizierung, Routing, Tooling, Parsing, Konfiguration)
  - Deterministische Regressionstests für bekannte Fehler
- Erwartungen:
  - Läuft in CI
  - Keine echten Schlüssel erforderlich
  - Sollte schnell und stabil sein
- Hinweis zu Projekten:
  - Nicht gezieltes `pnpm test` führt jetzt elf kleinere Shard-Konfigurationen aus (`core-unit-src`, `core-unit-security`, `core-unit-ui`, `core-unit-support`, `core-support-boundary`, `core-contracts`, `core-bundled`, `core-runtime`, `agentic`, `auto-reply`, `extensions`) statt eines einzigen riesigen nativen Root-Project-Prozesses. Das senkt den Spitzen-RSS auf ausgelasteten Rechnern und verhindert, dass auto-reply-/Extension-Arbeit andere Suiten ausbremst.
  - `pnpm test --watch` verwendet weiterhin den nativen Root-`vitest.config.ts`-Projektgraphen, weil eine Watch-Schleife über mehrere Shards nicht praktikabel ist.
  - `pnpm test`, `pnpm test:watch` und `pnpm test:perf:imports` routen explizite Datei-/Verzeichnisziele zuerst durch scoped Lanes, sodass `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts` nicht die vollen Startup-Kosten des Root-Projekts zahlen muss.
  - `pnpm test:changed` erweitert geänderte Git-Pfade in dieselben scoped Lanes, wenn der Diff nur routbare Quell-/Testdateien berührt; Änderungen an Konfiguration/Setup fallen weiterhin auf die breitere Wiederholung des Root-Projekts zurück.
  - Ausgewählte `plugin-sdk`- und `commands`-Tests werden ebenfalls durch dedizierte leichte Lanes geroutet, die `test/setup-openclaw-runtime.ts` überspringen; zustandsbehaftete/laufzeitintensive Dateien bleiben auf den bestehenden Lanes.
  - Ausgewählte `plugin-sdk`- und `commands`-Hilfsquelldateien mappen Läufe im Changed-Modus ebenfalls auf explizite benachbarte Tests in diesen leichten Lanes, sodass Änderungen an Helfern nicht die gesamte schwere Suite für dieses Verzeichnis neu starten.
  - `auto-reply` hat jetzt drei dedizierte Buckets: Core-Helfer auf oberster Ebene, Integrations-Tests `reply.*` auf oberster Ebene und den Unterbaum `src/auto-reply/reply/**`. Dadurch bleibt die schwerste Reply-Harness-Arbeit von den günstigen Status-/Chunk-/Token-Tests getrennt.
- Hinweis zum eingebetteten Runner:
  - Wenn Sie Eingaben zur Message-Tool-Erkennung oder den Laufzeitkontext der Kompaktierung ändern,
    halten Sie beide Abdeckungsebenen intakt.
  - Fügen Sie gezielte Helfer-Regressionen für reine Routing-/Normalisierungsgrenzen hinzu.
  - Halten Sie außerdem die Integrationssuiten des eingebetteten Runners funktionsfähig:
    `src/agents/pi-embedded-runner/compact.hooks.test.ts`,
    `src/agents/pi-embedded-runner/run.overflow-compaction.test.ts` und
    `src/agents/pi-embedded-runner/run.overflow-compaction.loop.test.ts`.
  - Diese Suiten prüfen, dass scoped IDs und Kompaktierungsverhalten weiterhin
    durch die realen Pfade `run.ts` / `compact.ts` fließen; reine Helfer-Tests
    sind kein ausreichender Ersatz für diese Integrationspfade.
- Hinweis zum Pool:
  - Die Basis-Vitest-Konfiguration verwendet jetzt standardmäßig `threads`.
  - Die gemeinsame Vitest-Konfiguration fixiert außerdem `isolate: false` und verwendet den nicht isolierten Runner über Root-Projekte, E2E- und Live-Konfigurationen hinweg.
  - Die Root-UI-Lane behält ihr `jsdom`-Setup und ihren Optimizer, läuft jetzt aber ebenfalls auf dem gemeinsamen nicht isolierten Runner.
  - Jeder `pnpm test`-Shard erbt dieselben Standardwerte `threads` + `isolate: false` aus der gemeinsamen Vitest-Konfiguration.
  - Der gemeinsame Launcher `scripts/run-vitest.mjs` fügt standardmäßig jetzt auch `--no-maglev` für Vitest-Kind-Node-Prozesse hinzu, um V8-Kompilieraufwand bei großen lokalen Läufen zu reduzieren. Setzen Sie `OPENCLAW_VITEST_ENABLE_MAGLEV=1`, wenn Sie gegen das Standardverhalten von V8 vergleichen müssen.
- Hinweis zur schnellen lokalen Iteration:
  - `pnpm test:changed` routet durch scoped Lanes, wenn sich die geänderten Pfade sauber auf eine kleinere Suite abbilden lassen.
  - `pnpm test:max` und `pnpm test:changed:max` behalten dasselbe Routing-Verhalten bei, nur mit höherem Worker-Limit.
  - Die automatische Skalierung lokaler Worker ist jetzt bewusst konservativ und reduziert sich auch dann, wenn die Host-Load-Average bereits hoch ist, sodass mehrere gleichzeitige Vitest-Läufe standardmäßig weniger Schaden anrichten.
  - Die Basis-Vitest-Konfiguration markiert die Projekte/Konfigurationsdateien als `forceRerunTriggers`, damit Wiederholungen im Changed-Modus korrekt bleiben, wenn sich die Test-Verdrahtung ändert.
  - Die Konfiguration lässt `OPENCLAW_VITEST_FS_MODULE_CACHE` auf unterstützten Hosts aktiviert; setzen Sie `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/abs/path`, wenn Sie einen expliziten Cache-Pfad für direktes Profiling möchten.
- Hinweis zum Performance-Debugging:
  - `pnpm test:perf:imports` aktiviert Importdauer-Berichte von Vitest plus Ausgabe zur Import-Aufschlüsselung.
  - `pnpm test:perf:imports:changed` beschränkt dieselbe Profiling-Ansicht auf Dateien, die seit `origin/main` geändert wurden.
- `pnpm test:perf:changed:bench -- --ref <git-ref>` vergleicht das geroutete `test:changed` für diesen festgeschriebenen Diff mit dem nativen Root-Project-Pfad und gibt Laufzeit plus macOS-Max-RSS aus.
- `pnpm test:perf:changed:bench -- --worktree` benchmarkt den aktuellen nicht festgeschriebenen Baum, indem die Liste geänderter Dateien durch `scripts/test-projects.mjs` und die Root-Vitest-Konfiguration geroutet wird.
  - `pnpm test:perf:profile:main` schreibt ein CPU-Profil des Hauptthreads für Vitest-/Vite-Startup- und Transform-Overhead.
  - `pnpm test:perf:profile:runner` schreibt Runner-CPU- und Heap-Profile für die Unit-Suite bei deaktivierter Datei-Parallelität.

### E2E (Gateway-Smoke)

- Befehl: `pnpm test:e2e`
- Konfiguration: `vitest.e2e.config.ts`
- Dateien: `src/**/*.e2e.test.ts`, `test/**/*.e2e.test.ts`
- Laufzeit-Standards:
  - Verwendet Vitest `threads` mit `isolate: false`, passend zum Rest des Repositories.
  - Verwendet adaptive Worker (CI: bis zu 2, lokal: standardmäßig 1).
  - Läuft standardmäßig im Silent-Modus, um Console-I/O-Overhead zu reduzieren.
- Nützliche Overrides:
  - `OPENCLAW_E2E_WORKERS=<n>`, um die Worker-Anzahl zu erzwingen (maximal 16).
  - `OPENCLAW_E2E_VERBOSE=1`, um ausführliche Console-Ausgabe wieder zu aktivieren.
- Umfang:
  - End-to-End-Verhalten von Gateway-Instanzen mit mehreren Instanzen
  - WebSocket-/HTTP-Oberflächen, Node-Pairing und aufwendigere Netzwerkpfade
- Erwartungen:
  - Läuft in CI (wenn in der Pipeline aktiviert)
  - Keine echten Schlüssel erforderlich
  - Mehr bewegliche Teile als Unit-Tests (kann langsamer sein)

### E2E: OpenShell-Backend-Smoke

- Befehl: `pnpm test:e2e:openshell`
- Datei: `test/openshell-sandbox.e2e.test.ts`
- Umfang:
  - Startet ein isoliertes OpenShell-Gateway auf dem Host über Docker
  - Erstellt eine Sandbox aus einer temporären lokalen Dockerfile
  - Testet OpenClaws OpenShell-Backend über echtes `sandbox ssh-config` + SSH-exec
  - Prüft remote-kanonisches Dateisystemverhalten über die Sandbox-FS-Bridge
- Erwartungen:
  - Nur opt-in; nicht Teil des Standardlaufs `pnpm test:e2e`
  - Erfordert eine lokale `openshell`-CLI plus einen funktionierenden Docker-Daemon
  - Verwendet isoliertes `HOME` / `XDG_CONFIG_HOME` und zerstört anschließend Test-Gateway und Sandbox
- Nützliche Overrides:
  - `OPENCLAW_E2E_OPENSHELL=1`, um den Test zu aktivieren, wenn die breitere E2E-Suite manuell ausgeführt wird
  - `OPENCLAW_E2E_OPENSHELL_COMMAND=/path/to/openshell`, um auf ein nicht standardmäßiges CLI-Binary oder Wrapper-Skript zu verweisen

### Live (reale Provider + reale Modelle)

- Befehl: `pnpm test:live`
- Konfiguration: `vitest.live.config.ts`
- Dateien: `src/**/*.live.test.ts`
- Standard: **aktiviert** durch `pnpm test:live` (setzt `OPENCLAW_LIVE_TEST=1`)
- Umfang:
  - „Funktioniert dieser Provider/dieses Modell _heute_ tatsächlich mit echten Zugangsdaten?“
  - Erkennt Provider-Formatänderungen, Eigenheiten beim Tool-Calling, Authentifizierungsprobleme und Rate-Limit-Verhalten
- Erwartungen:
  - Bewusst nicht CI-stabil (reale Netzwerke, reale Provider-Richtlinien, Quoten, Ausfälle)
  - Kostet Geld / verbraucht Rate Limits
  - Bevorzugt eingegrenzte Teilmengen statt „alles“
- Live-Läufe laden `~/.profile`, um fehlende API-Schlüssel aufzunehmen.
- Standardmäßig isolieren Live-Läufe trotzdem `HOME` und kopieren Konfigurations-/Authentifizierungsmaterial in ein temporäres Test-Home, damit Unit-Fixtures Ihr echtes `~/.openclaw` nicht verändern können.
- Setzen Sie `OPENCLAW_LIVE_USE_REAL_HOME=1` nur dann, wenn Live-Tests absichtlich Ihr echtes Home-Verzeichnis verwenden sollen.
- `pnpm test:live` verwendet jetzt standardmäßig einen leiseren Modus: Der `[live] ...`-Fortschritt bleibt sichtbar, die zusätzliche `~/.profile`-Meldung wird jedoch unterdrückt und Gateway-Bootstrap-Logs/Bonjour-Rauschen werden stummgeschaltet. Setzen Sie `OPENCLAW_LIVE_TEST_QUIET=0`, wenn Sie die vollständigen Start-Logs wieder sehen möchten.
- Rotation von API-Schlüsseln (provider-spezifisch): Setzen Sie `*_API_KEYS` im Komma-/Semikolon-Format oder `*_API_KEY_1`, `*_API_KEY_2` (zum Beispiel `OPENAI_API_KEYS`, `ANTHROPIC_API_KEYS`, `GEMINI_API_KEYS`) oder pro Live-Override `OPENCLAW_LIVE_*_KEY`; Tests wiederholen bei Rate-Limit-Antworten.
- Fortschritts-/Heartbeat-Ausgabe:
  - Live-Suiten geben jetzt Fortschrittszeilen nach stderr aus, damit lange Provider-Aufrufe sichtbar aktiv bleiben, auch wenn die Console-Erfassung von Vitest still ist.
  - `vitest.live.config.ts` deaktiviert die Console-Abfangung von Vitest, sodass Fortschrittszeilen von Provider/Gateway während Live-Läufen sofort gestreamt werden.
  - Steuern Sie Heartbeats für direkte Modelle mit `OPENCLAW_LIVE_HEARTBEAT_MS`.
  - Steuern Sie Heartbeats für Gateway/Probe mit `OPENCLAW_LIVE_GATEWAY_HEARTBEAT_MS`.

## Welche Suite sollte ich ausführen?

Verwenden Sie diese Entscheidungshilfe:

- Logik/Tests bearbeiten: `pnpm test` ausführen (und `pnpm test:coverage`, wenn Sie viel geändert haben)
- Gateway-Netzwerk / WS-Protokoll / Pairing anfassen: zusätzlich `pnpm test:e2e`
- „Mein Bot ist down“ / provider-spezifische Ausfälle / Tool-Calling debuggen: ein eingegrenztes `pnpm test:live` ausführen

## Live: Android-Node-Fähigkeiten-Sweep

- Test: `src/gateway/android-node.capabilities.live.test.ts`
- Skript: `pnpm android:test:integration`
- Ziel: **jeden aktuell beworbenen Befehl** eines verbundenen Android-Nodes aufrufen und das Vertragsverhalten des Befehls prüfen.
- Umfang:
  - Vorausgesetztes/manuelles Setup (die Suite installiert/startet/pairt die App nicht).
  - Gateway-Validierung `node.invoke` pro Befehl für den ausgewählten Android-Node.
- Erforderliches Vorab-Setup:
  - Android-App bereits mit dem Gateway verbunden und gepairt.
  - App im Vordergrund halten.
  - Berechtigungen/Aufzeichnungseinwilligung für Fähigkeiten erteilen, die erfolgreich sein sollen.
- Optionale Target-Overrides:
  - `OPENCLAW_ANDROID_NODE_ID` oder `OPENCLAW_ANDROID_NODE_NAME`.
  - `OPENCLAW_ANDROID_GATEWAY_URL` / `OPENCLAW_ANDROID_GATEWAY_TOKEN` / `OPENCLAW_ANDROID_GATEWAY_PASSWORD`.
- Vollständige Details zur Android-Einrichtung: [Android App](/de/platforms/android)

## Live: Modell-Smoke (Profilschlüssel)

Live-Tests sind in zwei Ebenen aufgeteilt, damit Fehler isoliert werden können:

- „Direktes Modell“ sagt uns, ob der Provider/das Modell mit dem angegebenen Schlüssel überhaupt antworten kann.
- „Gateway-Smoke“ sagt uns, ob die vollständige Gateway+Agent-Pipeline für dieses Modell funktioniert (Sitzungen, Verlauf, Tools, Sandbox-Richtlinie usw.).

### Ebene 1: Direkte Modell-Completion (ohne Gateway)

- Test: `src/agents/models.profiles.live.test.ts`
- Ziel:
  - Erkannte Modelle aufzählen
  - `getApiKeyForModel` verwenden, um Modelle auszuwählen, für die Sie Zugangsdaten haben
  - Eine kleine Completion pro Modell ausführen (und gezielte Regressionen, wenn nötig)
- Aktivierung:
  - `pnpm test:live` (oder `OPENCLAW_LIVE_TEST=1`, wenn Vitest direkt aufgerufen wird)
- Setzen Sie `OPENCLAW_LIVE_MODELS=modern` (oder `all`, Alias für modern), um diese Suite tatsächlich auszuführen; andernfalls wird sie übersprungen, damit `pnpm test:live` auf Gateway-Smoke fokussiert bleibt
- Modellauswahl:
  - `OPENCLAW_LIVE_MODELS=modern`, um die moderne Allowlist auszuführen (Opus/Sonnet 4.6+, GPT-5.x + Codex, Gemini 3, GLM 4.7, MiniMax M2.7, Grok 4)
  - `OPENCLAW_LIVE_MODELS=all` ist ein Alias für die moderne Allowlist
  - oder `OPENCLAW_LIVE_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,..."` (Komma-Allowlist)
- Providerauswahl:
  - `OPENCLAW_LIVE_PROVIDERS="google,google-antigravity,google-gemini-cli"` (Komma-Allowlist)
- Herkunft der Schlüssel:
  - Standardmäßig: Profilspeicher und Env-Fallbacks
  - Setzen Sie `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`, um **nur** den Profilspeicher zu erzwingen
- Warum das existiert:
  - Trennt „Provider-API ist kaputt / Schlüssel ist ungültig“ von „Gateway-Agent-Pipeline ist kaputt“
  - Enthält kleine, isolierte Regressionen (Beispiel: OpenAI-Responses/Codex-Responses-Reasoning-Replay + Tool-Call-Flows)

### Ebene 2: Gateway + Dev-Agent-Smoke (was „@openclaw“ tatsächlich macht)

- Test: `src/gateway/gateway-models.profiles.live.test.ts`
- Ziel:
  - Ein In-Process-Gateway starten
  - Eine Sitzung `agent:dev:*` erstellen/patchen (Modell-Override pro Lauf)
  - Über Modelle mit Schlüsseln iterieren und prüfen:
    - „sinnvolle“ Antwort (ohne Tools)
    - ein echter Tool-Aufruf funktioniert (`read`-Probe)
    - optionale zusätzliche Tool-Probes funktionieren (`exec+read`-Probe)
    - OpenAI-Regressionspfade (nur Tool-Call → Follow-up) funktionieren weiter
- Details zu den Probes (damit Fehler schnell erklärt werden können):
  - `read`-Probe: Der Test schreibt eine Nonce-Datei in den Workspace und fordert den Agenten auf, sie mit `read` zu lesen und die Nonce zurückzugeben.
  - `exec+read`-Probe: Der Test fordert den Agenten auf, per `exec` eine Nonce in eine temporäre Datei zu schreiben und sie dann mit `read` zurückzulesen.
  - Bild-Probe: Der Test hängt eine generierte PNG-Datei an (Katze + randomisierter Code) und erwartet, dass das Modell `cat <CODE>` zurückgibt.
  - Implementierungsreferenz: `src/gateway/gateway-models.profiles.live.test.ts` und `src/gateway/live-image-probe.ts`.
- Aktivierung:
  - `pnpm test:live` (oder `OPENCLAW_LIVE_TEST=1`, wenn Vitest direkt aufgerufen wird)
- Modellauswahl:
  - Standard: moderne Allowlist (Opus/Sonnet 4.6+, GPT-5.x + Codex, Gemini 3, GLM 4.7, MiniMax M2.7, Grok 4)
  - `OPENCLAW_LIVE_GATEWAY_MODELS=all` ist ein Alias für die moderne Allowlist
  - Oder `OPENCLAW_LIVE_GATEWAY_MODELS="provider/model"` (oder Komma-Liste) setzen, um einzugrenzen
- Providerauswahl (vermeidet „OpenRouter alles“):
  - `OPENCLAW_LIVE_GATEWAY_PROVIDERS="google,google-antigravity,google-gemini-cli,openai,anthropic,zai,minimax"` (Komma-Allowlist)
- Tool- + Bild-Probes sind in diesem Live-Test immer aktiv:
  - `read`-Probe + `exec+read`-Probe (Tool-Stresstest)
  - Bild-Probe läuft, wenn das Modell Unterstützung für Bildeingabe meldet
  - Ablauf (hohe Ebene):
    - Test generiert eine winzige PNG mit „CAT“ + Zufallscode (`src/gateway/live-image-probe.ts`)
    - Sendet sie über `agent` `attachments: [{ mimeType: "image/png", content: "<base64>" }]`
    - Gateway parst Anhänge in `images[]` (`src/gateway/server-methods/agent.ts` + `src/gateway/chat-attachments.ts`)
    - Eingebetteter Agent leitet eine multimodale Benutzernachricht an das Modell weiter
    - Prüfung: Antwort enthält `cat` + den Code (OCR-Toleranz: kleinere Fehler sind erlaubt)

Tipp: Um zu sehen, was Sie auf Ihrem Rechner testen können (und die exakten IDs `provider/model`), führen Sie aus:

```bash
openclaw models list
openclaw models list --json
```

## Live: CLI-Backend-Smoke (Claude, Codex, Gemini oder andere lokale CLIs)

- Test: `src/gateway/gateway-cli-backend.live.test.ts`
- Ziel: die Gateway+Agent-Pipeline mit einem lokalen CLI-Backend validieren, ohne Ihre Standardkonfiguration anzufassen.
- Standards für backend-spezifischen Smoke liegen zusammen mit der `cli-backend.ts`-Definition der besitzenden Extension.
- Aktivierung:
  - `pnpm test:live` (oder `OPENCLAW_LIVE_TEST=1`, wenn Vitest direkt aufgerufen wird)
  - `OPENCLAW_LIVE_CLI_BACKEND=1`
- Standardwerte:
  - Standard-Provider/-Modell: `claude-cli/claude-sonnet-4-6`
  - Befehls-/Arg-/Bildverhalten stammt aus den Metadaten des besitzenden CLI-Backend-Plugins.
- Overrides (optional):
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL="codex-cli/gpt-5.4"`
  - `OPENCLAW_LIVE_CLI_BACKEND_COMMAND="/full/path/to/codex"`
  - `OPENCLAW_LIVE_CLI_BACKEND_ARGS='["exec","--json","--color","never","--sandbox","read-only","--skip-git-repo-check"]'`
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_PROBE=1`, um einen echten Bildanhang zu senden (Pfade werden in den Prompt eingefügt).
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_ARG="--image"`, um Bilddateipfade als CLI-Argumente statt per Prompt-Injektion zu übergeben.
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_MODE="repeat"` (oder `"list"`), um zu steuern, wie Bildargumente übergeben werden, wenn `IMAGE_ARG` gesetzt ist.
  - `OPENCLAW_LIVE_CLI_BACKEND_RESUME_PROBE=1`, um eine zweite Runde zu senden und den Resume-Flow zu validieren.
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL_SWITCH_PROBE=0`, um die standardmäßige Kontinuitätsprobe derselben Sitzung beim Wechsel Claude Sonnet -> Opus zu deaktivieren (auf `1` setzen, um sie zu erzwingen, wenn das gewählte Modell ein Wechselziel unterstützt).

Beispiel:

```bash
OPENCLAW_LIVE_CLI_BACKEND=1 \
  OPENCLAW_LIVE_CLI_BACKEND_MODEL="codex-cli/gpt-5.4" \
  pnpm test:live src/gateway/gateway-cli-backend.live.test.ts
```

Docker-Rezept:

```bash
pnpm test:docker:live-cli-backend
```

Docker-Rezepte für einzelne Provider:

```bash
pnpm test:docker:live-cli-backend:claude
pnpm test:docker:live-cli-backend:codex
pnpm test:docker:live-cli-backend:gemini
```

Hinweise:

- Der Docker-Runner befindet sich unter `scripts/test-live-cli-backend-docker.sh`.
- Er führt den Live-CLI-Backend-Smoke innerhalb des Repo-Docker-Images als nicht-root Benutzer `node` aus.
- Er löst CLI-Smoke-Metadaten aus der besitzenden Extension auf und installiert dann das passende Linux-CLI-Paket (`@anthropic-ai/claude-code`, `@openai/codex` oder `@google/gemini-cli`) in ein gecachtes beschreibbares Präfix unter `OPENCLAW_DOCKER_CLI_TOOLS_DIR` (Standard: `~/.cache/openclaw/docker-cli-tools`).
- Der Live-CLI-Backend-Smoke testet jetzt denselben End-to-End-Flow für Claude, Codex und Gemini: Text-Runde, Bildklassifizierungsrunde, dann MCP-Tool-Aufruf `cron`, geprüft über die Gateway-CLI.
- Claudes Standard-Smoke patcht außerdem die Sitzung von Sonnet auf Opus und prüft, dass sich die fortgesetzte Sitzung weiterhin eine frühere Notiz merkt.

## Live: ACP-Bind-Smoke (`/acp spawn ... --bind here`)

- Test: `src/gateway/gateway-acp-bind.live.test.ts`
- Ziel: den realen ACP-Konversations-Bind-Flow mit einem Live-ACP-Agenten validieren:
  - `/acp spawn <agent> --bind here` senden
  - eine synthetische Nachrichtenkanal-Konversation direkt binden
  - ein normales Follow-up in genau derselben Konversation senden
  - prüfen, dass das Follow-up im gebundenen ACP-Sitzungstranskript landet
- Aktivierung:
  - `pnpm test:live src/gateway/gateway-acp-bind.live.test.ts`
  - `OPENCLAW_LIVE_ACP_BIND=1`
- Standardwerte:
  - ACP-Agenten in Docker: `claude,codex,gemini`
  - ACP-Agent für direktes `pnpm test:live ...`: `claude`
  - Synthetischer Kanal: Slack-DM-ähnlicher Konversationskontext
  - ACP-Backend: `acpx`
- Overrides:
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=claude`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=codex`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude,codex,gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND='npx -y @agentclientprotocol/claude-agent-acp@<version>'`
- Hinweise:
  - Diese Lane verwendet die Gateway-Oberfläche `chat.send` mit admin-only Feldern für synthetische Ursprungspfade, damit Tests Nachrichtenkanal-Kontext anhängen können, ohne vorzugeben, extern zuzustellen.
  - Wenn `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND` nicht gesetzt ist, verwendet der Test das eingebaute Agent-Registry des eingebetteten `acpx`-Plugins für den ausgewählten ACP-Harness-Agenten.

Beispiel:

```bash
OPENCLAW_LIVE_ACP_BIND=1 \
  OPENCLAW_LIVE_ACP_BIND_AGENT=claude \
  pnpm test:live src/gateway/gateway-acp-bind.live.test.ts
```

Docker-Rezept:

```bash
pnpm test:docker:live-acp-bind
```

Docker-Rezepte für einzelne Agenten:

```bash
pnpm test:docker:live-acp-bind:claude
pnpm test:docker:live-acp-bind:codex
pnpm test:docker:live-acp-bind:gemini
```

Docker-Hinweise:

- Der Docker-Runner befindet sich unter `scripts/test-live-acp-bind-docker.sh`.
- Standardmäßig führt er den ACP-Bind-Smoke nacheinander gegen alle unterstützten Live-CLI-Agenten aus: `claude`, `codex`, dann `gemini`.
- Verwenden Sie `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude`, `OPENCLAW_LIVE_ACP_BIND_AGENTS=codex` oder `OPENCLAW_LIVE_ACP_BIND_AGENTS=gemini`, um die Matrix einzugrenzen.
- Er lädt `~/.profile`, stellt das passende CLI-Authentifizierungsmaterial in den Container bereit, installiert `acpx` in ein beschreibbares npm-Präfix und installiert dann die angeforderte Live-CLI (`@anthropic-ai/claude-code`, `@openai/codex` oder `@google/gemini-cli`), falls sie fehlt.
- Innerhalb von Docker setzt der Runner `OPENCLAW_LIVE_ACP_BIND_ACPX_COMMAND=$HOME/.npm-global/bin/acpx`, damit `acpx` Provider-Umgebungsvariablen aus dem geladenen Profil für die untergeordnete Harness-CLI verfügbar hält.

### Empfohlene Live-Rezepte

Schmale, explizite Allowlists sind am schnellsten und am wenigsten fehleranfällig:

- Einzelnes Modell, direkt (ohne Gateway):
  - `OPENCLAW_LIVE_MODELS="openai/gpt-5.4" pnpm test:live src/agents/models.profiles.live.test.ts`

- Einzelnes Modell, Gateway-Smoke:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Tool-Calling über mehrere Provider hinweg:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3-flash-preview,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Fokus auf Google (Gemini-API-Schlüssel + Antigravity):
  - Gemini (API-Schlüssel): `OPENCLAW_LIVE_GATEWAY_MODELS="google/gemini-3-flash-preview" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`
  - Antigravity (OAuth): `OPENCLAW_LIVE_GATEWAY_MODELS="google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-pro-high" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

Hinweise:

- `google/...` verwendet die Gemini-API (API-Schlüssel).
- `google-antigravity/...` verwendet die Antigravity-OAuth-Bridge (Cloud-Code-Assist-ähnlicher Agent-Endpunkt).
- `google-gemini-cli/...` verwendet die lokale Gemini-CLI auf Ihrem Rechner (separate Authentifizierung + Tooling-Eigenheiten).
- Gemini-API vs. Gemini-CLI:
  - API: OpenClaw ruft Googles gehostete Gemini-API über HTTP auf (API-Schlüssel / Profil-Authentifizierung); das ist in der Regel das, was Nutzer mit „Gemini“ meinen.
  - CLI: OpenClaw führt lokal ein `gemini`-Binary aus; es hat eine eigene Authentifizierung und kann sich anders verhalten (Streaming/Tool-Support/Versionsabweichungen).

## Live: Modellmatrix (was wir abdecken)

Es gibt keine feste „CI-Modellliste“ (Live ist opt-in), aber dies sind die **empfohlenen** Modelle, die regelmäßig auf einem Entwicklerrechner mit Schlüsseln abgedeckt werden sollten.

### Moderner Smoke-Satz (Tool-Calling + Bild)

Das ist der Lauf für die „gängigen Modelle“, der funktionsfähig bleiben soll:

- OpenAI (nicht Codex): `openai/gpt-5.4` (optional: `openai/gpt-5.4-mini`)
- OpenAI Codex: `openai-codex/gpt-5.4`
- Anthropic: `anthropic/claude-opus-4-6` (oder `anthropic/claude-sonnet-4-6`)
- Google (Gemini-API): `google/gemini-3.1-pro-preview` und `google/gemini-3-flash-preview` (ältere Gemini-2.x-Modelle vermeiden)
- Google (Antigravity): `google-antigravity/claude-opus-4-6-thinking` und `google-antigravity/gemini-3-flash`
- Z.AI (GLM): `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

Gateway-Smoke mit Tools + Bild ausführen:
`OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,openai-codex/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3.1-pro-preview,google/gemini-3-flash-preview,google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-flash,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

### Baseline: Tool-Calling (Read + optional Exec)

Wählen Sie mindestens eines pro Provider-Familie:

- OpenAI: `openai/gpt-5.4` (oder `openai/gpt-5.4-mini`)
- Anthropic: `anthropic/claude-opus-4-6` (oder `anthropic/claude-sonnet-4-6`)
- Google: `google/gemini-3-flash-preview` (oder `google/gemini-3.1-pro-preview`)
- Z.AI (GLM): `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

Optionale zusätzliche Abdeckung (nice to have):

- xAI: `xai/grok-4` (oder die neueste verfügbare Version)
- Mistral: `mistral/`… (wählen Sie ein aktiviertes Modell mit „tools“-Fähigkeit)
- Cerebras: `cerebras/`… (wenn Sie Zugriff haben)
- LM Studio: `lmstudio/`… (lokal; Tool-Calling hängt vom API-Modus ab)

### Vision: Bild senden (Anhang → multimodale Nachricht)

Nehmen Sie mindestens ein bildfähiges Modell in `OPENCLAW_LIVE_GATEWAY_MODELS` auf (Claude/Gemini/OpenAI-Varianten mit Vision-Unterstützung usw.), um die Bild-Probe auszuführen.

### Aggregatoren / alternative Gateways

Wenn Sie Schlüssel aktiviert haben, unterstützen wir auch Tests über:

- OpenRouter: `openrouter/...` (Hunderte Modelle; verwenden Sie `openclaw models scan`, um Kandidaten mit Tool- + Bild-Fähigkeit zu finden)
- OpenCode: `opencode/...` für Zen und `opencode-go/...` für Go (Authentifizierung über `OPENCODE_API_KEY` / `OPENCODE_ZEN_API_KEY`)

Weitere Provider, die Sie in die Live-Matrix aufnehmen können (wenn Sie Zugangsdaten/Konfiguration haben):

- Integriert: `openai`, `openai-codex`, `anthropic`, `google`, `google-vertex`, `google-antigravity`, `google-gemini-cli`, `zai`, `openrouter`, `opencode`, `opencode-go`, `xai`, `groq`, `cerebras`, `mistral`, `github-copilot`
- Über `models.providers` (benutzerdefinierte Endpunkte): `minimax` (Cloud/API) sowie jeder OpenAI-/Anthropic-kompatible Proxy (LM Studio, vLLM, LiteLLM usw.)

Tipp: Versuchen Sie nicht, „alle Modelle“ in der Dokumentation fest zu codieren. Die maßgebliche Liste ist alles, was `discoverModels(...)` auf Ihrem Rechner zurückgibt + alle verfügbaren Schlüssel.

## Zugangsdaten (niemals committen)

Live-Tests finden Zugangsdaten genauso wie die CLI. Praktische Konsequenzen:

- Wenn die CLI funktioniert, sollten Live-Tests dieselben Schlüssel finden.
- Wenn ein Live-Test „keine Zugangsdaten“ meldet, debuggen Sie genauso, wie Sie `openclaw models list` / Modellauswahl debuggen würden.

- Authentifizierungsprofile pro Agent: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` (das bedeutet in den Live-Tests „Profilschlüssel“)
- Konfiguration: `~/.openclaw/openclaw.json` (oder `OPENCLAW_CONFIG_PATH`)
- Veraltetes State-Verzeichnis: `~/.openclaw/credentials/` (wird bei lokalen Live-Homes in das vorbereitete Test-Home kopiert, wenn vorhanden, ist aber nicht der Hauptspeicher für Profilschlüssel)
- Lokale Live-Läufe kopieren standardmäßig die aktive Konfiguration, pro Agent die `auth-profiles.json`-Dateien, veraltete `credentials/` und unterstützte externe CLI-Auth-Verzeichnisse in ein temporäres Test-Home; vorbereitete Live-Homes überspringen `workspace/` und `sandboxes/`, und Pfad-Overrides für `agents.*.workspace` / `agentDir` werden entfernt, damit Probes nicht in Ihrem echten Host-Workspace landen.

Wenn Sie sich auf Env-Schlüssel verlassen möchten (z. B. aus Ihrer `~/.profile` exportiert), führen Sie lokale Tests nach `source ~/.profile` aus oder verwenden Sie die Docker-Runner unten (diese können `~/.profile` in den Container mounten).

## Deepgram live (Audiotranskription)

- Test: `src/media-understanding/providers/deepgram/audio.live.test.ts`
- Aktivierung: `DEEPGRAM_API_KEY=... DEEPGRAM_LIVE_TEST=1 pnpm test:live src/media-understanding/providers/deepgram/audio.live.test.ts`

## BytePlus Coding-Plan live

- Test: `src/agents/byteplus.live.test.ts`
- Aktivierung: `BYTEPLUS_API_KEY=... BYTEPLUS_LIVE_TEST=1 pnpm test:live src/agents/byteplus.live.test.ts`
- Optionales Modell-Override: `BYTEPLUS_CODING_MODEL=ark-code-latest`

## ComfyUI-Workflow-Medien live

- Test: `extensions/comfy/comfy.live.test.ts`
- Aktivierung: `OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts`
- Umfang:
  - Testet die gebündelten comfy-Pfade für Bild, Video und `music_generate`
  - Überspringt jede Fähigkeit, sofern `models.providers.comfy.<capability>` nicht konfiguriert ist
  - Nützlich nach Änderungen an comfy-Workflow-Übermittlung, Polling, Downloads oder Plugin-Registrierung

## Bildgenerierung live

- Test: `src/image-generation/runtime.live.test.ts`
- Befehl: `pnpm test:live src/image-generation/runtime.live.test.ts`
- Harness: `pnpm test:live:media image`
- Umfang:
  - Zählt jedes registrierte Provider-Plugin für Bildgenerierung auf
  - Lädt fehlende Provider-Umgebungsvariablen aus Ihrer Login-Shell (`~/.profile`), bevor Probes ausgeführt werden
  - Verwendet standardmäßig Live-/Env-API-Schlüssel vor gespeicherten Authentifizierungsprofilen, sodass veraltete Testschlüssel in `auth-profiles.json` echte Shell-Zugangsdaten nicht maskieren
  - Überspringt Provider ohne nutzbare Authentifizierung/Profil/Modell
  - Führt die Standardvarianten der Bildgenerierung über die gemeinsame Runtime-Fähigkeit aus:
    - `google:flash-generate`
    - `google:pro-generate`
    - `google:pro-edit`
    - `openai:default-generate`
- Derzeit abgedeckte gebündelte Provider:
  - `openai`
  - `google`
- Optionale Eingrenzung:
  - `OPENCLAW_LIVE_IMAGE_GENERATION_PROVIDERS="openai,google"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_MODELS="openai/gpt-image-1,google/gemini-3.1-flash-image-preview"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_CASES="google:flash-generate,google:pro-edit"`
- Optionales Authentifizierungsverhalten:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`, um Authentifizierung aus dem Profilspeicher zu erzwingen und nur-env-Overrides zu ignorieren

## Musikgenerierung live

- Test: `extensions/music-generation-providers.live.test.ts`
- Aktivierung: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts`
- Harness: `pnpm test:live:media music`
- Umfang:
  - Testet den gemeinsamen gebündelten Provider-Pfad für Musikgenerierung
  - Deckt derzeit Google und MiniMax ab
  - Lädt Provider-Umgebungsvariablen aus Ihrer Login-Shell (`~/.profile`), bevor Probes ausgeführt werden
  - Verwendet standardmäßig Live-/Env-API-Schlüssel vor gespeicherten Authentifizierungsprofilen, sodass veraltete Testschlüssel in `auth-profiles.json` echte Shell-Zugangsdaten nicht maskieren
  - Überspringt Provider ohne nutzbare Authentifizierung/Profil/Modell
  - Führt beide deklarierten Runtime-Modi aus, sofern verfügbar:
    - `generate` mit Eingabe nur per Prompt
    - `edit`, wenn der Provider `capabilities.edit.enabled` deklariert
  - Aktuelle Abdeckung der gemeinsamen Lane:
    - `google`: `generate`, `edit`
    - `minimax`: `generate`
    - `comfy`: separate Comfy-Live-Datei, nicht dieser gemeinsame Sweep
- Optionale Eingrenzung:
  - `OPENCLAW_LIVE_MUSIC_GENERATION_PROVIDERS="google,minimax"`
  - `OPENCLAW_LIVE_MUSIC_GENERATION_MODELS="google/lyria-3-clip-preview,minimax/music-2.5+"`
- Optionales Authentifizierungsverhalten:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`, um Authentifizierung aus dem Profilspeicher zu erzwingen und nur-env-Overrides zu ignorieren

## Videogenerierung live

- Test: `extensions/video-generation-providers.live.test.ts`
- Aktivierung: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/video-generation-providers.live.test.ts`
- Harness: `pnpm test:live:media video`
- Umfang:
  - Testet den gemeinsamen gebündelten Provider-Pfad für Videogenerierung
  - Lädt Provider-Umgebungsvariablen aus Ihrer Login-Shell (`~/.profile`), bevor Probes ausgeführt werden
  - Verwendet standardmäßig Live-/Env-API-Schlüssel vor gespeicherten Authentifizierungsprofilen, sodass veraltete Testschlüssel in `auth-profiles.json` echte Shell-Zugangsdaten nicht maskieren
  - Überspringt Provider ohne nutzbare Authentifizierung/Profil/Modell
  - Führt beide deklarierten Runtime-Modi aus, sofern verfügbar:
    - `generate` mit Eingabe nur per Prompt
    - `imageToVideo`, wenn der Provider `capabilities.imageToVideo.enabled` deklariert und der ausgewählte Provider/das ausgewählte Modell im gemeinsamen Sweep Puffer-gestützte lokale Bildeingabe akzeptiert
    - `videoToVideo`, wenn der Provider `capabilities.videoToVideo.enabled` deklariert und der ausgewählte Provider/das ausgewählte Modell im gemeinsamen Sweep Puffer-gestützte lokale Videoeingabe akzeptiert
  - Aktuell deklarierte, aber im gemeinsamen Sweep übersprungene `imageToVideo`-Provider:
    - `vydra`, weil das gebündelte `veo3` nur Text unterstützt und das gebündelte `kling` eine entfernte Bild-URL erfordert
  - Provider-spezifische Vydra-Abdeckung:
    - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_VYDRA_VIDEO=1 pnpm test:live -- extensions/vydra/vydra.live.test.ts`
    - diese Datei führt `veo3` Text-zu-Video plus eine `kling`-Lane aus, die standardmäßig eine Fixture mit entfernter Bild-URL verwendet
  - Aktuelle `videoToVideo`-Live-Abdeckung:
    - nur `runway`, wenn das ausgewählte Modell `runway/gen4_aleph` ist
  - Aktuell deklarierte, aber im gemeinsamen Sweep übersprungene `videoToVideo`-Provider:
    - `alibaba`, `qwen`, `xai`, weil diese Pfade derzeit entfernte Referenz-URLs `http(s)` / MP4 erfordern
    - `google`, weil die aktuelle gemeinsame Gemini-/Veo-Lane lokale puffergestützte Eingabe verwendet und dieser Pfad im gemeinsamen Sweep nicht akzeptiert wird
    - `openai`, weil die aktuelle gemeinsame Lane keine Garantien für org-spezifischen Zugriff auf Video-Inpaint/Remix hat
- Optionale Eingrenzung:
  - `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="google,openai,runway"`
  - `OPENCLAW_LIVE_VIDEO_GENERATION_MODELS="google/veo-3.1-fast-generate-preview,openai/sora-2,runway/gen4_aleph"`
- Optionales Authentifizierungsverhalten:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`, um Authentifizierung aus dem Profilspeicher zu erzwingen und nur-env-Overrides zu ignorieren

## Media-Live-Harness

- Befehl: `pnpm test:live:media`
- Zweck:
  - Führt die gemeinsamen Live-Suiten für Bild, Musik und Video über einen repo-nativen Entry-Point aus
  - Lädt fehlende Provider-Umgebungsvariablen automatisch aus `~/.profile`
  - Grenzt standardmäßig jede Suite automatisch auf Provider ein, die aktuell nutzbare Authentifizierung haben
  - Verwendet `scripts/test-live.mjs` wieder, sodass Heartbeat- und Quiet-Mode-Verhalten konsistent bleiben
- Beispiele:
  - `pnpm test:live:media`
  - `pnpm test:live:media image video --providers openai,google,minimax`
  - `pnpm test:live:media video --video-providers openai,runway --all-providers`
  - `pnpm test:live:media music --quiet`

## Docker-Runner (optionale „funktioniert unter Linux“-Prüfungen)

Diese Docker-Runner teilen sich in zwei Gruppen:

- Live-Modell-Runner: `test:docker:live-models` und `test:docker:live-gateway` führen nur ihre jeweilige Live-Datei für Profilschlüssel im Repo-Docker-Image aus (`src/agents/models.profiles.live.test.ts` und `src/gateway/gateway-models.profiles.live.test.ts`), mounten dabei Ihr lokales Konfigurationsverzeichnis und Ihren Workspace (und laden `~/.profile`, falls gemountet). Die passenden lokalen Entry-Points sind `test:live:models-profiles` und `test:live:gateway-profiles`.
- Docker-Live-Runner verwenden standardmäßig ein kleineres Smoke-Limit, damit ein vollständiger Docker-Sweep praktikabel bleibt:
  `test:docker:live-models` verwendet standardmäßig `OPENCLAW_LIVE_MAX_MODELS=12`, und
  `test:docker:live-gateway` verwendet standardmäßig `OPENCLAW_LIVE_GATEWAY_SMOKE=1`,
  `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=8`,
  `OPENCLAW_LIVE_GATEWAY_STEP_TIMEOUT_MS=45000` und
  `OPENCLAW_LIVE_GATEWAY_MODEL_TIMEOUT_MS=90000`. Überschreiben Sie diese Env-Variablen, wenn Sie
  ausdrücklich den größeren vollständigen Scan möchten.
- `test:docker:all` baut das Live-Docker-Image einmal über `test:docker:live-build` und verwendet es dann für die beiden Docker-Lanes für Live erneut.
- Container-Smoke-Runner: `test:docker:openwebui`, `test:docker:onboard`, `test:docker:gateway-network`, `test:docker:mcp-channels` und `test:docker:plugins` starten einen oder mehrere echte Container und prüfen Integrationspfade höherer Ebene.

Die Docker-Runner für Live-Modelle mounten außerdem nur die benötigten CLI-Auth-Homes (oder alle unterstützten, wenn der Lauf nicht eingegrenzt ist) und kopieren sie dann vor dem Lauf in das Container-Home, damit externe CLI-OAuth Tokens aktualisieren kann, ohne den Auth-Store auf dem Host zu verändern:

- Direkte Modelle: `pnpm test:docker:live-models` (Skript: `scripts/test-live-models-docker.sh`)
- ACP-Bind-Smoke: `pnpm test:docker:live-acp-bind` (Skript: `scripts/test-live-acp-bind-docker.sh`)
- CLI-Backend-Smoke: `pnpm test:docker:live-cli-backend` (Skript: `scripts/test-live-cli-backend-docker.sh`)
- Gateway + Dev-Agent: `pnpm test:docker:live-gateway` (Skript: `scripts/test-live-gateway-models-docker.sh`)
- Open-WebUI-Live-Smoke: `pnpm test:docker:openwebui` (Skript: `scripts/e2e/openwebui-docker.sh`)
- Onboarding-Assistent (TTY, vollständiges Scaffolding): `pnpm test:docker:onboard` (Skript: `scripts/e2e/onboard-docker.sh`)
- Gateway-Netzwerk (zwei Container, WS-Auth + Health): `pnpm test:docker:gateway-network` (Skript: `scripts/e2e/gateway-network-docker.sh`)
- MCP-Kanal-Bridge (vorbereiteter Gateway + stdio-Bridge + roher Claude-Benachrichtigungsframe-Smoke): `pnpm test:docker:mcp-channels` (Skript: `scripts/e2e/mcp-channels-docker.sh`)
- Plugins (Installations-Smoke + Alias `/plugin` + Claude-Bundle-Neustart-Semantik): `pnpm test:docker:plugins` (Skript: `scripts/e2e/plugins-docker.sh`)

Die Docker-Runner für Live-Modelle mounten außerdem den aktuellen Checkout schreibgeschützt und
stellen ihn in einem temporären Arbeitsverzeichnis innerhalb des Containers bereit. Dadurch bleibt das Runtime-
Image schlank, während Vitest weiterhin gegen Ihren exakten lokalen Quellstand/Ihre Konfiguration läuft.
Der Bereitstellungsschritt überspringt große lokale Caches und App-Build-Ausgaben wie
`.pnpm-store`, `.worktrees`, `__openclaw_vitest__` und app-lokale `.build`- oder
Gradle-Ausgabeverzeichnisse, damit Docker-Live-Läufe nicht minutenlang
maschinenabhängige Artefakte kopieren.
Sie setzen außerdem `OPENCLAW_SKIP_CHANNELS=1`, damit Gateway-Live-Probes im Container
keine echten Kanal-Worker für Telegram/Discord/etc. starten.
`test:docker:live-models` führt weiterhin `pnpm test:live` aus; geben Sie daher
auch `OPENCLAW_LIVE_GATEWAY_*` weiter, wenn Sie Gateway-Live-Abdeckung in dieser Docker-Lane
eingrenzen oder ausschließen möchten.
`test:docker:openwebui` ist ein Kompatibilitäts-Smoke auf höherer Ebene: Es startet einen
OpenClaw-Gateway-Container mit aktivierten OpenAI-kompatiblen HTTP-Endpunkten,
startet einen fest angepinnten Open-WebUI-Container gegen dieses Gateway, meldet sich über
Open WebUI an, prüft, dass `/api/models` `openclaw/default` bereitstellt, und sendet dann eine
echte Chat-Anfrage über den Proxy `/api/chat/completions` von Open WebUI.
Der erste Lauf kann merklich langsamer sein, weil Docker möglicherweise erst das
Open-WebUI-Image ziehen muss und Open WebUI sein eigenes Cold-Start-Setup abschließen muss.
Diese Lane erwartet einen nutzbaren Live-Modellschlüssel, und `OPENCLAW_PROFILE_FILE`
(`~/.profile` standardmäßig) ist der primäre Weg, ihn in Docker-Läufen bereitzustellen.
Erfolgreiche Läufe geben eine kleine JSON-Payload aus wie `{ "ok": true, "model":
"openclaw/default", ... }`.
`test:docker:mcp-channels` ist bewusst deterministisch und benötigt kein
echtes Telegram-, Discord- oder iMessage-Konto. Es startet einen vorbereiteten Gateway-
Container, startet einen zweiten Container, der `openclaw mcp serve` ausführt, und
prüft dann geroutete Konversationserkennung, Transkript-Lesezugriffe, Anhangsmetadaten,
das Verhalten der Live-Event-Queue, ausgehendes Send-Routing sowie Claude-ähnliche Kanal- +
Berechtigungsbenachrichtigungen über die echte stdio-MCP-Bridge. Die Benachrichtigungsprüfung
untersucht die rohen stdio-MCP-Frames direkt, sodass der Smoke validiert, was die
Bridge tatsächlich ausgibt, und nicht nur das, was ein bestimmtes Client-SDK zufällig sichtbar macht.

Manueller ACP-Smoke für Plain-Language-Threads (nicht CI):

- `bun scripts/dev/discord-acp-plain-language-smoke.ts --channel <discord-channel-id> ...`
- Behalten Sie dieses Skript für Regressions-/Debugging-Workflows bei. Es könnte für die Validierung des ACP-Thread-Routings erneut gebraucht werden, also nicht löschen.

Nützliche Env-Variablen:

- `OPENCLAW_CONFIG_DIR=...` (Standard: `~/.openclaw`), gemountet nach `/home/node/.openclaw`
- `OPENCLAW_WORKSPACE_DIR=...` (Standard: `~/.openclaw/workspace`), gemountet nach `/home/node/.openclaw/workspace`
- `OPENCLAW_PROFILE_FILE=...` (Standard: `~/.profile`), gemountet nach `/home/node/.profile` und vor dem Ausführen der Tests geladen
- `OPENCLAW_DOCKER_CLI_TOOLS_DIR=...` (Standard: `~/.cache/openclaw/docker-cli-tools`), gemountet nach `/home/node/.npm-global` für gecachte CLI-Installationen innerhalb von Docker
- Externe CLI-Auth-Verzeichnisse/-Dateien unter `$HOME` werden schreibgeschützt unter `/host-auth...` gemountet und dann vor dem Start der Tests nach `/home/node/...` kopiert
  - Standardverzeichnisse: `.minimax`
  - Standarddateien: `~/.codex/auth.json`, `~/.codex/config.toml`, `.claude.json`, `~/.claude/.credentials.json`, `~/.claude/settings.json`, `~/.claude/settings.local.json`
  - Eingegrenzte Provider-Läufe mounten nur die benötigten Verzeichnisse/Dateien, abgeleitet aus `OPENCLAW_LIVE_PROVIDERS` / `OPENCLAW_LIVE_GATEWAY_PROVIDERS`
  - Manuell überschreiben mit `OPENCLAW_DOCKER_AUTH_DIRS=all`, `OPENCLAW_DOCKER_AUTH_DIRS=none` oder einer Komma-Liste wie `OPENCLAW_DOCKER_AUTH_DIRS=.claude,.codex`
- `OPENCLAW_LIVE_GATEWAY_MODELS=...` / `OPENCLAW_LIVE_MODELS=...`, um den Lauf einzugrenzen
- `OPENCLAW_LIVE_GATEWAY_PROVIDERS=...` / `OPENCLAW_LIVE_PROVIDERS=...`, um Provider im Container zu filtern
- `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`, um sicherzustellen, dass Zugangsdaten aus dem Profilspeicher kommen (nicht aus Env)
- `OPENCLAW_OPENWEBUI_MODEL=...`, um das vom Gateway für den Open-WebUI-Smoke bereitgestellte Modell auszuwählen
- `OPENCLAW_OPENWEBUI_PROMPT=...`, um den für den Open-WebUI-Smoke verwendeten Nonce-Prüf-Prompt zu überschreiben
- `OPENWEBUI_IMAGE=...`, um den angepinnten Open-WebUI-Image-Tag zu überschreiben

## Dokumentations-Sanity

Führen Sie nach Dokumentationsänderungen Dokumentationsprüfungen aus: `pnpm check:docs`.
Führen Sie die vollständige Mintlify-Anchor-Validierung aus, wenn Sie auch In-Page-Heading-Prüfungen brauchen: `pnpm docs:check-links:anchors`.

## Offline-Regression (CI-sicher)

Dies sind Regressionen für die „reale Pipeline“ ohne reale Provider:

- Gateway-Tool-Calling (Mock-OpenAI, echtes Gateway + Agent-Schleife): `src/gateway/gateway.test.ts` (Fall: "runs a mock OpenAI tool call end-to-end via gateway agent loop")
- Gateway-Assistent (WS `wizard.start`/`wizard.next`, schreibt Konfiguration + Auth erzwungen): `src/gateway/gateway.test.ts` (Fall: "runs wizard over ws and writes auth token config")

## Evals zur Agent-Zuverlässigkeit (Skills)

Wir haben bereits einige CI-sichere Tests, die sich wie „Evals zur Agent-Zuverlässigkeit“ verhalten:

- Mock-Tool-Calling durch das reale Gateway + die Agent-Schleife (`src/gateway/gateway.test.ts`).
- End-to-End-Assistenten-Flows, die Sitzungsverdrahtung und Konfigurationseffekte validieren (`src/gateway/gateway.test.ts`).

Was für Skills noch fehlt (siehe [Skills](/de/tools/skills)):

- **Entscheidungsverhalten:** Wenn Skills im Prompt aufgeführt sind, wählt der Agent dann den richtigen Skill (oder vermeidet irrelevante)?
- **Konformität:** Liest der Agent `SKILL.md` vor der Nutzung und befolgt erforderliche Schritte/Argumente?
- **Workflow-Verträge:** Mehrere Runden umfassende Szenarien, die Tool-Reihenfolge, Übernahme des Sitzungsverlaufs und Sandbox-Grenzen prüfen.

Zukünftige Evals sollten zuerst deterministisch bleiben:

- Ein Szenario-Runner mit Mock-Providern, um Tool-Aufrufe + Reihenfolge, Skill-Datei-Lesevorgänge und Sitzungsverdrahtung zu prüfen.
- Eine kleine Suite mit auf Skills fokussierten Szenarien (verwenden vs. vermeiden, Gating, Prompt-Injektion).
- Optionale Live-Evals (opt-in, env-gesteuert) erst, nachdem die CI-sichere Suite vorhanden ist.

## Vertragstests (Plugin- und Kanal-Shape)

Vertragstests prüfen, dass jedes registrierte Plugin und jeder registrierte Kanal seinem
Schnittstellenvertrag entspricht. Sie iterieren über alle entdeckten Plugins und führen eine Suite aus
Form- und Verhaltensprüfungen aus. Die Standard-Unit-Lane `pnpm test`
überspringt diese gemeinsamen Nahtstellen- und Smoke-Dateien absichtlich; führen Sie die Vertragsbefehle explizit
aus, wenn Sie gemeinsame Kanal- oder Provider-Oberflächen ändern.

### Befehle

- Alle Verträge: `pnpm test:contracts`
- Nur Kanalverträge: `pnpm test:contracts:channels`
- Nur Provider-Verträge: `pnpm test:contracts:plugins`

### Kanalverträge

Zu finden in `src/channels/plugins/contracts/*.contract.test.ts`:

- **plugin** - Grundlegende Plugin-Form (id, name, capabilities)
- **setup** - Vertrag des Setup-Assistenten
- **session-binding** - Verhalten bei Sitzungsbindung
- **outbound-payload** - Struktur der Nachrichten-Payload
- **inbound** - Verarbeitung eingehender Nachrichten
- **actions** - Kanal-Aktions-Handler
- **threading** - Behandlung von Thread-IDs
- **directory** - API für Verzeichnis/Roster
- **group-policy** - Durchsetzung von Gruppenrichtlinien

### Provider-Status-Verträge

Zu finden in `src/plugins/contracts/*.contract.test.ts`.

- **status** - Kanal-Status-Probes
- **registry** - Form der Plugin-Registry

### Provider-Verträge

Zu finden in `src/plugins/contracts/*.contract.test.ts`:

- **auth** - Vertrag des Authentifizierungsflusses
- **auth-choice** - Authentifizierungswahl/-auswahl
- **catalog** - API des Modellkatalogs
- **discovery** - Plugin-Erkennung
- **loader** - Plugin-Laden
- **runtime** - Provider-Laufzeit
- **shape** - Plugin-Form/Schnittstelle
- **wizard** - Setup-Assistent

### Wann ausführen

- Nach Änderungen an `plugin-sdk`-Exports oder Subpaths
- Nach dem Hinzufügen oder Ändern eines Kanal- oder Provider-Plugins
- Nach dem Refactoring von Plugin-Registrierung oder -Erkennung

Vertragstests laufen in CI und erfordern keine echten API-Schlüssel.

## Regressionen hinzufügen (Hinweise)

Wenn Sie ein in Live entdecktes Provider-/Modellproblem beheben:

- Fügen Sie nach Möglichkeit eine CI-sichere Regression hinzu (Mock-/Stub-Provider oder exakte Erfassung der Request-Shape-Transformation)
- Wenn das Problem inhärent nur live auftritt (Rate Limits, Authentifizierungsrichtlinien), halten Sie den Live-Test schmal und opt-in über Env-Variablen
- Zielen Sie bevorzugt auf die kleinste Ebene, die den Fehler erkennt:
  - Bug bei Provider-Request-Konvertierung/-Replay → Test für direkte Modelle
  - Bug in Gateway-Sitzung/Verlauf/Tool-Pipeline → Gateway-Live-Smoke oder CI-sicherer Gateway-Mock-Test
- Guardrail für SecretRef-Traversal:
  - `src/secrets/exec-secret-ref-id-parity.test.ts` leitet pro SecretRef-Klasse genau ein Beispielziel aus den Registry-Metadaten ab (`listSecretTargetRegistryEntries()`) und prüft dann, dass Traversal-Segment-Exec-IDs abgelehnt werden.
  - Wenn Sie in `src/secrets/target-registry-data.ts` eine neue SecretRef-Zielfamilie mit `includeInPlan` hinzufügen, aktualisieren Sie `classifyTargetClass` in diesem Test. Der Test schlägt absichtlich bei nicht klassifizierten Ziel-IDs fehl, damit neue Klassen nicht stillschweigend übersprungen werden.
