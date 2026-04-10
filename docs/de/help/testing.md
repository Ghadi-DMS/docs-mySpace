---
read_when:
    - Tests lokal oder in CI ausführen
    - Regressionstests für Modell-/Provider-Fehler hinzufügen
    - Gateway- und Agent-Verhalten debuggen
summary: 'Test-Kit: Unit-/E2E-/Live-Suites, Docker-Runner und was jeder Test abdeckt'
title: Testen
x-i18n:
    generated_at: "2026-04-10T06:21:15Z"
    model: gpt-5.4
    provider: openai
    source_hash: 21b78e59a5189f4e8e6e1b490d350f4735c0395da31d21fc5d10b825313026b4
    source_path: help/testing.md
    workflow: 15
---

# Testen

OpenClaw hat drei Vitest-Suites (Unit/Integration, E2E, Live) und eine kleine Gruppe von Docker-Runnern.

Dieses Dokument ist ein Leitfaden dazu, „wie wir testen“:

- Was jede Suite abdeckt (und was sie bewusst _nicht_ abdeckt)
- Welche Befehle für gängige Abläufe ausgeführt werden sollten (lokal, vor dem Push, Debugging)
- Wie Live-Tests Anmeldedaten erkennen und Modelle/Provider auswählen
- Wie Regressionstests für reale Modell-/Provider-Probleme hinzugefügt werden

## Schnellstart

An den meisten Tagen:

- Vollständiges Gate (vor dem Push erwartet): `pnpm build && pnpm check && pnpm test`
- Schnellere lokale Ausführung der vollständigen Suite auf einem leistungsfähigen Rechner: `pnpm test:max`
- Direkte Vitest-Watch-Schleife: `pnpm test:watch`
- Direktes Targeting von Dateien leitet jetzt auch Extension-/Channel-Pfade weiter: `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts`
- Bevorzuge zuerst gezielte Ausführungen, wenn du an einem einzelnen Fehler arbeitest.
- Docker-gestützte QA-Site: `pnpm qa:lab:up`
- Linux-VM-gestützte QA-Bahn: `pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline`

Wenn du Tests anfasst oder zusätzliche Sicherheit möchtest:

- Coverage-Gate: `pnpm test:coverage`
- E2E-Suite: `pnpm test:e2e`

Beim Debuggen realer Provider/Modelle (erfordert echte Anmeldedaten):

- Live-Suite (Modelle + Gateway-Tool-/Image-Sonden): `pnpm test:live`
- Eine einzelne Live-Datei leise ausführen: `pnpm test:live -- src/agents/models.profiles.live.test.ts`

Tipp: Wenn du nur einen einzelnen fehlschlagenden Fall brauchst, grenze Live-Tests lieber über die unten beschriebenen Allowlist-Umgebungsvariablen ein.

## QA-spezifische Runner

Diese Befehle stehen neben den Haupt-Testsuites bereit, wenn du die Realität von QA-lab brauchst:

- `pnpm openclaw qa suite`
  - Führt repo-gestützte QA-Szenarien direkt auf dem Host aus.
- `pnpm openclaw qa suite --runner multipass`
  - Führt dieselbe QA-Suite in einer temporären Multipass-Linux-VM aus.
  - Behält dasselbe Verhalten bei der Szenarioauswahl wie `qa suite` auf dem Host bei.
  - Verwendet dieselben Flags zur Provider-/Modellauswahl wie `qa suite`.
  - Live-Ausführungen leiten die unterstützten QA-Authentifizierungs-Eingaben weiter, die für den Gast praktikabel sind:
    umgebungsbasierte Provider-Schlüssel, den Pfad zur QA-Live-Provider-Konfiguration und `CODEX_HOME`,
    falls vorhanden.
  - Ausgabeverzeichnisse müssen unter dem Repo-Root bleiben, damit der Gast über
    den eingehängten Workspace zurückschreiben kann.
  - Schreibt den normalen QA-Bericht + die Zusammenfassung sowie Multipass-Logs unter
    `.artifacts/qa-e2e/...`.
- `pnpm qa:lab:up`
  - Startet die Docker-gestützte QA-Site für operatorähnliche QA-Arbeit.

## Test-Suites (was wo läuft)

Betrachte die Suites als „zunehmend realistisch“ (und zunehmend fehleranfällig/kostenintensiv):

### Unit / Integration (Standard)

- Befehl: `pnpm test`
- Konfiguration: zehn sequentielle Shard-Läufe (`vitest.full-*.config.ts`) über die vorhandenen begrenzten Vitest-Projekte
- Dateien: Core-/Unit-Inventare unter `src/**/*.test.ts`, `packages/**/*.test.ts`, `test/**/*.test.ts` und die auf der Allowlist stehenden `ui`-Node-Tests, die von `vitest.unit.config.ts` abgedeckt werden
- Umfang:
  - Reine Unit-Tests
  - In-Process-Integrationstests (Gateway-Auth, Routing, Tooling, Parsing, Konfiguration)
  - Deterministische Regressionstests für bekannte Fehler
- Erwartungen:
  - Läuft in CI
  - Keine echten Schlüssel erforderlich
  - Sollte schnell und stabil sein
- Hinweis zu Projekten:
  - Nicht gezieltes `pnpm test` führt jetzt elf kleinere Shard-Konfigurationen aus (`core-unit-src`, `core-unit-security`, `core-unit-ui`, `core-unit-support`, `core-support-boundary`, `core-contracts`, `core-bundled`, `core-runtime`, `agentic`, `auto-reply`, `extensions`) statt eines einzigen riesigen nativen Root-Projekt-Prozesses. Das senkt den Spitzen-RSS auf ausgelasteten Rechnern und verhindert, dass auto-reply-/Extension-Arbeit nicht zusammenhängende Suites aushungert.
  - `pnpm test --watch` verwendet weiterhin den nativen Root-`vitest.config.ts`-Projektgraphen, weil eine Watch-Schleife mit mehreren Shards nicht praktikabel ist.
  - `pnpm test`, `pnpm test:watch` und `pnpm test:perf:imports` leiten explizite Datei-/Verzeichnis-Targets zuerst durch begrenzte Bahnen, sodass `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts` nicht die Startkosten des vollständigen Root-Projekts zahlen muss.
  - `pnpm test:changed` erweitert geänderte Git-Pfade in dieselben begrenzten Bahnen, wenn der Diff nur weiterleitbare Quell-/Testdateien betrifft; Änderungen an Konfiguration/Setup fallen weiterhin auf den breiten Root-Projekt-Neustart zurück.
  - Ausgewählte `plugin-sdk`- und `commands`-Tests werden ebenfalls durch dedizierte leichte Bahnen geleitet, die `test/setup-openclaw-runtime.ts` überspringen; zustandsbehaftete/runtime-schwere Dateien bleiben auf den vorhandenen Bahnen.
  - Ausgewählte `plugin-sdk`- und `commands`-Hilfsquelldateien ordnen Runs im Changed-Modus auch expliziten benachbarten Tests in diesen leichten Bahnen zu, sodass Änderungen an Hilfsdateien nicht die vollständige schwere Suite für dieses Verzeichnis neu ausführen.
  - `auto-reply` hat jetzt drei dedizierte Buckets: Hilfsfunktionen der obersten Core-Ebene, Integrations-Tests der obersten Ebene `reply.*` und den Teilbaum `src/auto-reply/reply/**`. Dadurch bleibt die schwerste Reply-Harness-Arbeit von den günstigen Status-/Chunk-/Token-Tests getrennt.
- Hinweis zum eingebetteten Runner:
  - Wenn du Eingaben für die Message-Tool-Erkennung oder den Laufzeitkontext der Kompaktierung änderst,
    halte beide Ebenen der Abdeckung aufrecht.
  - Füge gezielte Hilfsregressionstests für reine Routing-/Normalisierungs-Grenzen hinzu.
  - Halte außerdem die Integrations-Suites des eingebetteten Runners gesund:
    `src/agents/pi-embedded-runner/compact.hooks.test.ts`,
    `src/agents/pi-embedded-runner/run.overflow-compaction.test.ts` und
    `src/agents/pi-embedded-runner/run.overflow-compaction.loop.test.ts`.
  - Diese Suites verifizieren, dass begrenzte IDs und das Kompaktierungsverhalten weiterhin
    durch die echten Pfade `run.ts` / `compact.ts` fließen; reine Hilfstests sind
    kein ausreichender Ersatz für diese Integrationspfade.
- Hinweis zum Pool:
  - Die Basis-Vitest-Konfiguration verwendet jetzt standardmäßig `threads`.
  - Die gemeinsame Vitest-Konfiguration setzt außerdem `isolate: false` fest und verwendet den nicht isolierten Runner über die Root-Projekte, E2E- und Live-Konfigurationen hinweg.
  - Die Root-UI-Bahn behält ihr `jsdom`-Setup und ihren Optimizer, läuft jetzt aber ebenfalls auf dem gemeinsamen nicht isolierten Runner.
  - Jeder `pnpm test`-Shard übernimmt dieselben Standardwerte `threads` + `isolate: false` aus der gemeinsamen Vitest-Konfiguration.
  - Der gemeinsame Launcher `scripts/run-vitest.mjs` fügt standardmäßig jetzt auch `--no-maglev` für Vitest-Child-Node-Prozesse hinzu, um V8-Kompilieraufwand bei großen lokalen Läufen zu reduzieren. Setze `OPENCLAW_VITEST_ENABLE_MAGLEV=1`, wenn du mit dem Standardverhalten von V8 vergleichen musst.
- Hinweis zur schnellen lokalen Iteration:
  - `pnpm test:changed` leitet durch begrenzte Bahnen, wenn die geänderten Pfade sauber auf eine kleinere Suite abgebildet werden.
  - `pnpm test:max` und `pnpm test:changed:max` behalten dasselbe Weiterleitungsverhalten bei, nur mit einer höheren Worker-Obergrenze.
  - Die automatische Skalierung lokaler Worker ist jetzt bewusst konservativ und fährt auch zurück, wenn die Host-Load-Average bereits hoch ist, sodass mehrere gleichzeitige Vitest-Läufe standardmäßig weniger Schaden anrichten.
  - Die Basis-Vitest-Konfiguration markiert die Projekte/Konfigurationsdateien als `forceRerunTriggers`, damit Reruns im Changed-Modus korrekt bleiben, wenn sich das Test-Wiring ändert.
  - Die Konfiguration lässt `OPENCLAW_VITEST_FS_MODULE_CACHE` auf unterstützten Hosts aktiviert; setze `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/abs/path`, wenn du einen expliziten Cache-Speicherort für direktes Profiling verwenden möchtest.
- Hinweis zum Performance-Debugging:
  - `pnpm test:perf:imports` aktiviert Import-Dauer-Reporting von Vitest sowie eine Ausgabe zur Import-Aufschlüsselung.
  - `pnpm test:perf:imports:changed` beschränkt dieselbe Profiling-Ansicht auf Dateien, die seit `origin/main` geändert wurden.
- `pnpm test:perf:changed:bench -- --ref <git-ref>` vergleicht geroutetes `test:changed` mit dem nativen Root-Projekt-Pfad für diesen festgeschriebenen Diff und gibt Wandzeit sowie macOS-Max-RSS aus.
- `pnpm test:perf:changed:bench -- --worktree` benchmarkt den aktuellen Dirty-Tree, indem die Liste geänderter Dateien durch `scripts/test-projects.mjs` und die Root-Vitest-Konfiguration geroutet wird.
  - `pnpm test:perf:profile:main` schreibt ein CPU-Profil des Main-Threads für Vitest-/Vite-Start- und Transform-Overhead.
  - `pnpm test:perf:profile:runner` schreibt CPU-+Heap-Profile des Runners für die Unit-Suite mit deaktivierter Datei-Parallelisierung.

### E2E (Gateway-Smoke)

- Befehl: `pnpm test:e2e`
- Konfiguration: `vitest.e2e.config.ts`
- Dateien: `src/**/*.e2e.test.ts`, `test/**/*.e2e.test.ts`
- Laufzeit-Standards:
  - Verwendet Vitest-`threads` mit `isolate: false`, passend zum Rest des Repos.
  - Verwendet adaptive Worker (CI: bis zu 2, lokal: standardmäßig 1).
  - Läuft standardmäßig im Silent-Modus, um den Overhead durch Konsolen-I/O zu verringern.
- Nützliche Overrides:
  - `OPENCLAW_E2E_WORKERS=<n>`, um die Worker-Anzahl zu erzwingen (begrenzt auf 16).
  - `OPENCLAW_E2E_VERBOSE=1`, um ausführliche Konsolenausgabe wieder zu aktivieren.
- Umfang:
  - End-to-End-Verhalten des Gateways mit mehreren Instanzen
  - WebSocket-/HTTP-Oberflächen, Node-Pairing und schwerere Netzwerkarbeit
- Erwartungen:
  - Läuft in CI (wenn in der Pipeline aktiviert)
  - Keine echten Schlüssel erforderlich
  - Mehr bewegliche Teile als Unit-Tests (kann langsamer sein)

### E2E: OpenShell-Backend-Smoke

- Befehl: `pnpm test:e2e:openshell`
- Datei: `test/openshell-sandbox.e2e.test.ts`
- Umfang:
  - Startet über Docker ein isoliertes OpenShell-Gateway auf dem Host
  - Erstellt eine Sandbox aus einem temporären lokalen Dockerfile
  - Testet das OpenShell-Backend von OpenClaw über echtes `sandbox ssh-config` + SSH-Ausführung
  - Verifiziert remote-kanonisches Dateisystemverhalten über die Sandbox-fs-Bridge
- Erwartungen:
  - Nur Opt-in; nicht Teil des standardmäßigen Laufs `pnpm test:e2e`
  - Erfordert eine lokale `openshell`-CLI sowie einen funktionierenden Docker-Daemon
  - Verwendet isolierte `HOME` / `XDG_CONFIG_HOME` und zerstört anschließend das Test-Gateway und die Sandbox
- Nützliche Overrides:
  - `OPENCLAW_E2E_OPENSHELL=1`, um den Test zu aktivieren, wenn die breitere E2E-Suite manuell ausgeführt wird
  - `OPENCLAW_E2E_OPENSHELL_COMMAND=/path/to/openshell`, um auf ein nicht standardmäßiges CLI-Binary oder Wrapper-Skript zu verweisen

### Live (reale Provider + reale Modelle)

- Befehl: `pnpm test:live`
- Konfiguration: `vitest.live.config.ts`
- Dateien: `src/**/*.live.test.ts`
- Standard: **aktiviert** durch `pnpm test:live` (setzt `OPENCLAW_LIVE_TEST=1`)
- Umfang:
  - „Funktioniert dieser Provider/dieses Modell _heute_ tatsächlich mit echten Anmeldedaten?“
  - Erkennt Änderungen an Provider-Formaten, Tool-Calling-Eigenheiten, Auth-Probleme und Verhalten bei Rate-Limits
- Erwartungen:
  - Von Natur aus nicht CI-stabil (reale Netzwerke, echte Provider-Richtlinien, Quoten, Ausfälle)
  - Kostet Geld / verbraucht Rate-Limits
  - Bevorzuge eingegrenzte Teilmengen statt „alles“
- Live-Läufe sourcen `~/.profile`, um fehlende API-Schlüssel aufzunehmen.
- Standardmäßig isolieren Live-Läufe weiterhin `HOME` und kopieren Konfigurations-/Authentifizierungsmaterial in ein temporäres Test-Home, damit Unit-Fixtures dein echtes `~/.openclaw` nicht verändern können.
- Setze `OPENCLAW_LIVE_USE_REAL_HOME=1` nur, wenn du bewusst möchtest, dass Live-Tests dein echtes Home-Verzeichnis verwenden.
- `pnpm test:live` verwendet jetzt standardmäßig einen leiseren Modus: Es behält die Fortschrittsausgabe `[live] ...` bei, unterdrückt aber den zusätzlichen Hinweis zu `~/.profile` und schaltet Gateway-Bootstrap-Logs/Bonjour-Chatter stumm. Setze `OPENCLAW_LIVE_TEST_QUIET=0`, wenn du die vollständigen Start-Logs wieder sehen möchtest.
- API-Schlüssel-Rotation (providerspezifisch): setze `*_API_KEYS` im Komma-/Semikolon-Format oder `*_API_KEY_1`, `*_API_KEY_2` (zum Beispiel `OPENAI_API_KEYS`, `ANTHROPIC_API_KEYS`, `GEMINI_API_KEYS`) oder ein Override pro Live-Lauf über `OPENCLAW_LIVE_*_KEY`; Tests versuchen es bei Rate-Limit-Antworten erneut.
- Fortschritts-/Heartbeat-Ausgabe:
  - Live-Suites geben jetzt Fortschrittszeilen an stderr aus, damit lange Provider-Aufrufe auch dann sichtbar aktiv sind, wenn die Konsolenerfassung von Vitest leise ist.
  - `vitest.live.config.ts` deaktiviert die Konsolenabfangung von Vitest, sodass Fortschrittszeilen von Provider/Gateway bei Live-Läufen sofort gestreamt werden.
  - Passe Heartbeats für direkte Modelle mit `OPENCLAW_LIVE_HEARTBEAT_MS` an.
  - Passe Heartbeats für Gateway/Sonden mit `OPENCLAW_LIVE_GATEWAY_HEARTBEAT_MS` an.

## Welche Suite sollte ich ausführen?

Verwende diese Entscheidungstabelle:

- Logik/Tests bearbeiten: führe `pnpm test` aus (und `pnpm test:coverage`, wenn du viel geändert hast)
- Gateway-Netzwerk / WS-Protokoll / Pairing anfassen: ergänze `pnpm test:e2e`
- „Mein Bot ist down“ debuggen / providerspezifische Fehler / Tool Calling: führe ein eingegrenztes `pnpm test:live` aus

## Live: Android-Node-Fähigkeits-Sweep

- Test: `src/gateway/android-node.capabilities.live.test.ts`
- Skript: `pnpm android:test:integration`
- Ziel: **jeden derzeit beworbenen Befehl** eines verbundenen Android-Node aufrufen und das Vertragsverhalten des Befehls prüfen.
- Umfang:
  - Vorkonditioniertes/manuelles Setup (die Suite installiert/startet/pairt die App nicht).
  - Befehl-für-Befehl-Validierung von Gateway-`node.invoke` für den ausgewählten Android-Node.
- Erforderliches Vorab-Setup:
  - Android-App ist bereits verbunden und mit dem Gateway gepairt.
  - App bleibt im Vordergrund.
  - Berechtigungen/Aufnahmezustimmung sind für die Fähigkeiten erteilt, die erfolgreich sein sollen.
- Optionale Ziel-Overrides:
  - `OPENCLAW_ANDROID_NODE_ID` oder `OPENCLAW_ANDROID_NODE_NAME`.
  - `OPENCLAW_ANDROID_GATEWAY_URL` / `OPENCLAW_ANDROID_GATEWAY_TOKEN` / `OPENCLAW_ANDROID_GATEWAY_PASSWORD`.
- Vollständige Android-Setup-Details: [Android App](/de/platforms/android)

## Live: Modell-Smoke (Profilschlüssel)

Live-Tests sind in zwei Ebenen aufgeteilt, damit wir Fehler isolieren können:

- „Direktes Modell“ sagt uns, ob der Provider/das Modell mit dem angegebenen Schlüssel überhaupt antworten kann.
- „Gateway-Smoke“ sagt uns, ob die vollständige Gateway-+Agent-Pipeline für dieses Modell funktioniert (Sessions, Verlauf, Tools, Sandbox-Richtlinie usw.).

### Ebene 1: Direkte Modell-Completion (kein Gateway)

- Test: `src/agents/models.profiles.live.test.ts`
- Ziel:
  - Erkannte Modelle aufzählen
  - `getApiKeyForModel` verwenden, um Modelle auszuwählen, für die du Anmeldedaten hast
  - Pro Modell eine kleine Completion ausführen (und bei Bedarf gezielte Regressionstests)
- Aktivierung:
  - `pnpm test:live` (oder `OPENCLAW_LIVE_TEST=1`, wenn Vitest direkt aufgerufen wird)
- Setze `OPENCLAW_LIVE_MODELS=modern` (oder `all`, Alias für modern), um diese Suite tatsächlich auszuführen; andernfalls wird sie übersprungen, damit `pnpm test:live` auf Gateway-Smoke fokussiert bleibt
- Modellauswahl:
  - `OPENCLAW_LIVE_MODELS=modern`, um die moderne Allowlist auszuführen (Opus/Sonnet 4.6+, GPT-5.x + Codex, Gemini 3, GLM 4.7, MiniMax M2.7, Grok 4)
  - `OPENCLAW_LIVE_MODELS=all` ist ein Alias für die moderne Allowlist
  - oder `OPENCLAW_LIVE_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,..."` (Komma-Allowlist)
  - Moderne/alle Sweeps verwenden standardmäßig ein kuratiertes Limit mit hoher Signalkraft; setze `OPENCLAW_LIVE_MAX_MODELS=0` für einen vollständigen modernen Sweep oder eine positive Zahl für ein kleineres Limit.
- Providerauswahl:
  - `OPENCLAW_LIVE_PROVIDERS="google,google-antigravity,google-gemini-cli"` (Komma-Allowlist)
- Woher die Schlüssel kommen:
  - Standardmäßig: Profilspeicher und Env-Fallbacks
  - Setze `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`, um **nur den Profilspeicher** zu erzwingen
- Warum das existiert:
  - Trennt „Provider-API ist kaputt / Schlüssel ist ungültig“ von „Gateway-Agent-Pipeline ist kaputt“
  - Enthält kleine, isolierte Regressionstests (Beispiel: OpenAI Responses/Codex Responses Reasoning-Replay + Tool-Call-Flows)

### Ebene 2: Gateway + Dev-Agent-Smoke (was `@openclaw` tatsächlich tut)

- Test: `src/gateway/gateway-models.profiles.live.test.ts`
- Ziel:
  - Ein In-Process-Gateway starten
  - Eine `agent:dev:*`-Session erstellen/patchen (Modell-Override pro Lauf)
  - Modelle mit Schlüsseln durchlaufen und Folgendes prüfen:
    - „sinnvolle“ Antwort (ohne Tools)
    - ein echter Tool-Aufruf funktioniert (Read-Probe)
    - optionale zusätzliche Tool-Sonden funktionieren (Exec+Read-Probe)
    - OpenAI-Regressionspfade (nur Tool-Call → Follow-up) funktionieren weiterhin
- Sondendetails (damit du Fehler schnell erklären kannst):
  - `read`-Probe: Der Test schreibt eine Nonce-Datei in den Workspace und fordert den Agent auf, sie zu `read`en und die Nonce zurückzugeben.
  - `exec+read`-Probe: Der Test fordert den Agent auf, per `exec` eine Nonce in eine temporäre Datei zu schreiben und sie dann per `read` zurückzulesen.
  - Image-Probe: Der Test hängt ein generiertes PNG an (Katze + zufälliger Code) und erwartet, dass das Modell `cat <CODE>` zurückgibt.
  - Implementierungsreferenz: `src/gateway/gateway-models.profiles.live.test.ts` und `src/gateway/live-image-probe.ts`.
- Aktivierung:
  - `pnpm test:live` (oder `OPENCLAW_LIVE_TEST=1`, wenn Vitest direkt aufgerufen wird)
- Modellauswahl:
  - Standard: moderne Allowlist (Opus/Sonnet 4.6+, GPT-5.x + Codex, Gemini 3, GLM 4.7, MiniMax M2.7, Grok 4)
  - `OPENCLAW_LIVE_GATEWAY_MODELS=all` ist ein Alias für die moderne Allowlist
  - Oder setze `OPENCLAW_LIVE_GATEWAY_MODELS="provider/model"` (oder eine Komma-Liste), um einzugrenzen
  - Moderne/alle Gateway-Sweeps verwenden standardmäßig ein kuratiertes Limit mit hoher Signalkraft; setze `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=0` für einen vollständigen modernen Sweep oder eine positive Zahl für ein kleineres Limit.
- Providerauswahl (vermeide „alles von OpenRouter“):
  - `OPENCLAW_LIVE_GATEWAY_PROVIDERS="google,google-antigravity,google-gemini-cli,openai,anthropic,zai,minimax"` (Komma-Allowlist)
- Tool- und Image-Sonden sind in diesem Live-Test immer aktiv:
  - `read`-Probe + `exec+read`-Probe (Tool-Stress)
  - Image-Probe läuft, wenn das Modell Unterstützung für Bildeingaben bewirbt
  - Ablauf (auf hoher Ebene):
    - Test generiert ein winziges PNG mit „CAT“ + zufälligem Code (`src/gateway/live-image-probe.ts`)
    - Sendet es über `agent` `attachments: [{ mimeType: "image/png", content: "<base64>" }]`
    - Gateway parst Anhänge in `images[]` (`src/gateway/server-methods/agent.ts` + `src/gateway/chat-attachments.ts`)
    - Eingebetteter Agent leitet eine multimodale Nutzernachricht an das Modell weiter
    - Prüfung: Antwort enthält `cat` + den Code (OCR-Toleranz: kleine Fehler sind erlaubt)

Tipp: Um zu sehen, was du auf deinem Rechner testen kannst (und die genauen IDs `provider/model`), führe Folgendes aus:

```bash
openclaw models list
openclaw models list --json
```

## Live: CLI-Backend-Smoke (Claude, Codex, Gemini oder andere lokale CLIs)

- Test: `src/gateway/gateway-cli-backend.live.test.ts`
- Ziel: die Gateway-+Agent-Pipeline mit einem lokalen CLI-Backend validieren, ohne deine Standardkonfiguration anzufassen.
- Backend-spezifische Smoke-Standards liegen in der Definition `cli-backend.ts` der besitzenden Extension.
- Aktivierung:
  - `pnpm test:live` (oder `OPENCLAW_LIVE_TEST=1`, wenn Vitest direkt aufgerufen wird)
  - `OPENCLAW_LIVE_CLI_BACKEND=1`
- Standards:
  - Standard-Provider/-Modell: `claude-cli/claude-sonnet-4-6`
  - Verhalten von Befehl/Argumenten/Bildern kommt aus den Metadaten des besitzenden CLI-Backend-Plugins.
- Overrides (optional):
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL="codex-cli/gpt-5.4"`
  - `OPENCLAW_LIVE_CLI_BACKEND_COMMAND="/full/path/to/codex"`
  - `OPENCLAW_LIVE_CLI_BACKEND_ARGS='["exec","--json","--color","never","--sandbox","read-only","--skip-git-repo-check"]'`
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_PROBE=1`, um einen echten Bildanhang zu senden (Pfade werden in den Prompt injiziert).
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_ARG="--image"`, um Bilddateipfade als CLI-Argumente statt per Prompt-Injektion zu übergeben.
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_MODE="repeat"` (oder `"list"`), um zu steuern, wie Bildargumente übergeben werden, wenn `IMAGE_ARG` gesetzt ist.
  - `OPENCLAW_LIVE_CLI_BACKEND_RESUME_PROBE=1`, um einen zweiten Turn zu senden und den Resume-Flow zu validieren.
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL_SWITCH_PROBE=0`, um die standardmäßige Kontinuitätsprobe derselben Session Claude Sonnet -> Opus zu deaktivieren (setze auf `1`, um sie zu erzwingen, wenn das ausgewählte Modell ein Wechselziel unterstützt).

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

- Der Docker-Runner liegt unter `scripts/test-live-cli-backend-docker.sh`.
- Er führt den Live-CLI-Backend-Smoke im Docker-Image des Repos als nicht-root-`node`-Nutzer aus.
- Er löst Metadaten für den CLI-Smoke aus der besitzenden Extension auf und installiert dann das passende Linux-CLI-Paket (`@anthropic-ai/claude-code`, `@openai/codex` oder `@google/gemini-cli`) in ein zwischengespeichertes beschreibbares Präfix unter `OPENCLAW_DOCKER_CLI_TOOLS_DIR` (Standard: `~/.cache/openclaw/docker-cli-tools`).
- Der Live-CLI-Backend-Smoke testet jetzt denselben End-to-End-Flow für Claude, Codex und Gemini: Text-Turn, Bildklassifizierungs-Turn, dann MCP-`cron`-Tool-Call, verifiziert über die Gateway-CLI.
- Claudes standardmäßiger Smoke patcht außerdem die Session von Sonnet auf Opus und prüft, dass sich die fortgesetzte Session weiterhin an eine frühere Notiz erinnert.

## Live: ACP-Bind-Smoke (`/acp spawn ... --bind here`)

- Test: `src/gateway/gateway-acp-bind.live.test.ts`
- Ziel: den echten ACP-Conversational-Bind-Flow mit einem Live-ACP-Agent validieren:
  - `/acp spawn <agent> --bind here` senden
  - einen synthetischen Message-Channel-Conversation-Kontext an Ort und Stelle binden
  - auf derselben Conversation ein normales Follow-up senden
  - prüfen, dass das Follow-up im gebundenen ACP-Session-Transkript landet
- Aktivierung:
  - `pnpm test:live src/gateway/gateway-acp-bind.live.test.ts`
  - `OPENCLAW_LIVE_ACP_BIND=1`
- Standards:
  - ACP-Agenten in Docker: `claude,codex,gemini`
  - ACP-Agent für direktes `pnpm test:live ...`: `claude`
  - Synthetischer Channel: Conversation-Kontext im Stil einer Slack-DM
  - ACP-Backend: `acpx`
- Overrides:
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=claude`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=codex`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude,codex,gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND='npx -y @agentclientprotocol/claude-agent-acp@<version>'`
- Hinweise:
  - Diese Bahn verwendet die Gateway-Oberfläche `chat.send` mit admin-only-synthetischen Feldern für die Ursprungsroute, damit Tests Message-Channel-Kontext anhängen können, ohne vorzugeben, extern zuzustellen.
  - Wenn `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND` nicht gesetzt ist, verwendet der Test die integrierte Agent-Registry des eingebetteten `acpx`-Plugins für den ausgewählten ACP-Harness-Agent.

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

- Der Docker-Runner liegt unter `scripts/test-live-acp-bind-docker.sh`.
- Standardmäßig führt er den ACP-Bind-Smoke nacheinander gegen alle unterstützten Live-CLI-Agenten aus: `claude`, `codex`, dann `gemini`.
- Verwende `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude`, `OPENCLAW_LIVE_ACP_BIND_AGENTS=codex` oder `OPENCLAW_LIVE_ACP_BIND_AGENTS=gemini`, um die Matrix einzugrenzen.
- Er sourct `~/.profile`, stellt das passende CLI-Authentifizierungsmaterial im Container bereit, installiert `acpx` in ein beschreibbares npm-Präfix und installiert dann die angeforderte Live-CLI (`@anthropic-ai/claude-code`, `@openai/codex` oder `@google/gemini-cli`), falls sie fehlt.
- Innerhalb von Docker setzt der Runner `OPENCLAW_LIVE_ACP_BIND_ACPX_COMMAND=$HOME/.npm-global/bin/acpx`, damit acpx die Provider-Umgebungsvariablen aus dem gesourcten Profil für die untergeordnete Harness-CLI verfügbar hält.

### Empfohlene Live-Rezepte

Enge, explizite Allowlists sind am schnellsten und am wenigsten fehleranfällig:

- Einzelnes Modell, direkt (ohne Gateway):
  - `OPENCLAW_LIVE_MODELS="openai/gpt-5.4" pnpm test:live src/agents/models.profiles.live.test.ts`

- Einzelnes Modell, Gateway-Smoke:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Tool Calling über mehrere Provider hinweg:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3-flash-preview,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Google-Fokus (Gemini-API-Schlüssel + Antigravity):
  - Gemini (API-Schlüssel): `OPENCLAW_LIVE_GATEWAY_MODELS="google/gemini-3-flash-preview" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`
  - Antigravity (OAuth): `OPENCLAW_LIVE_GATEWAY_MODELS="google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-pro-high" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

Hinweise:

- `google/...` verwendet die Gemini API (API-Schlüssel).
- `google-antigravity/...` verwendet die Antigravity-OAuth-Bridge (agent endpoint im Stil von Cloud Code Assist).
- `google-gemini-cli/...` verwendet die lokale Gemini-CLI auf deinem Rechner (separate Authentifizierung + eigene Tooling-Eigenheiten).
- Gemini API vs. Gemini CLI:
  - API: OpenClaw ruft Googles gehostete Gemini API über HTTP auf (API-Schlüssel / Profil-Auth); das ist das, was die meisten Nutzer mit „Gemini“ meinen.
  - CLI: OpenClaw shellt zu einem lokalen `gemini`-Binary aus; dieses hat seine eigene Authentifizierung und kann sich anders verhalten (Streaming/Tool-Unterstützung/Versionsabweichung).

## Live: Modellmatrix (was wir abdecken)

Es gibt keine feste „CI-Modellliste“ (Live ist Opt-in), aber dies sind die **empfohlenen** Modelle, die auf einem Entwicklerrechner mit Schlüsseln regelmäßig abgedeckt werden sollten.

### Modernes Smoke-Set (Tool Calling + Bild)

Dies ist der Lauf für die „gängigen Modelle“, den wir funktionsfähig halten wollen:

- OpenAI (nicht Codex): `openai/gpt-5.4` (optional: `openai/gpt-5.4-mini`)
- OpenAI Codex: `openai-codex/gpt-5.4`
- Anthropic: `anthropic/claude-opus-4-6` (oder `anthropic/claude-sonnet-4-6`)
- Google (Gemini API): `google/gemini-3.1-pro-preview` und `google/gemini-3-flash-preview` (vermeide ältere Gemini-2.x-Modelle)
- Google (Antigravity): `google-antigravity/claude-opus-4-6-thinking` und `google-antigravity/gemini-3-flash`
- Z.AI (GLM): `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

Gateway-Smoke mit Tools + Bild ausführen:
`OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,openai-codex/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3.1-pro-preview,google/gemini-3-flash-preview,google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-flash,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

### Baseline: Tool Calling (`Read` + optional `Exec`)

Wähle mindestens eines pro Provider-Familie:

- OpenAI: `openai/gpt-5.4` (oder `openai/gpt-5.4-mini`)
- Anthropic: `anthropic/claude-opus-4-6` (oder `anthropic/claude-sonnet-4-6`)
- Google: `google/gemini-3-flash-preview` (oder `google/gemini-3.1-pro-preview`)
- Z.AI (GLM): `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

Optionale zusätzliche Abdeckung (nice to have):

- xAI: `xai/grok-4` (oder die neueste verfügbare Version)
- Mistral: `mistral/`… (wähle ein „tools“-fähiges Modell, das du aktiviert hast)
- Cerebras: `cerebras/`… (falls du Zugriff hast)
- LM Studio: `lmstudio/`… (lokal; Tool Calling hängt vom API-Modus ab)

### Vision: Bild senden (Anhang → multimodale Nachricht)

Nimm mindestens ein bildfähiges Modell in `OPENCLAW_LIVE_GATEWAY_MODELS` auf (Claude/Gemini/OpenAI-vision-fähige Varianten usw.), um die Image-Probe auszuführen.

### Aggregatoren / alternative Gateways

Wenn du passende Schlüssel aktiviert hast, unterstützen wir auch Tests über:

- OpenRouter: `openrouter/...` (Hunderte Modelle; verwende `openclaw models scan`, um Kandidaten mit Tool- und Bild-Unterstützung zu finden)
- OpenCode: `opencode/...` für Zen und `opencode-go/...` für Go (Auth über `OPENCODE_API_KEY` / `OPENCODE_ZEN_API_KEY`)

Weitere Provider, die du in die Live-Matrix aufnehmen kannst (wenn du Anmeldedaten/Konfiguration hast):

- Integriert: `openai`, `openai-codex`, `anthropic`, `google`, `google-vertex`, `google-antigravity`, `google-gemini-cli`, `zai`, `openrouter`, `opencode`, `opencode-go`, `xai`, `groq`, `cerebras`, `mistral`, `github-copilot`
- Über `models.providers` (benutzerdefinierte Endpoints): `minimax` (Cloud/API) sowie jeder OpenAI-/Anthropic-kompatible Proxy (LM Studio, vLLM, LiteLLM usw.)

Tipp: Versuche nicht, „alle Modelle“ in der Dokumentation fest zu codieren. Die maßgebliche Liste ist das, was `discoverModels(...)` auf deinem Rechner zurückgibt + welche Schlüssel verfügbar sind.

## Anmeldedaten (niemals committen)

Live-Tests erkennen Anmeldedaten auf dieselbe Weise wie die CLI. Praktische Folgen:

- Wenn die CLI funktioniert, sollten Live-Tests dieselben Schlüssel finden.
- Wenn ein Live-Test „keine Anmeldedaten“ meldet, debugge es genauso, wie du `openclaw models list` / Modellauswahl debuggen würdest.

- Auth-Profile pro Agent: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` (das ist es, was „Profilschlüssel“ in den Live-Tests bedeutet)
- Konfiguration: `~/.openclaw/openclaw.json` (oder `OPENCLAW_CONFIG_PATH`)
- Legacy-Statusverzeichnis: `~/.openclaw/credentials/` (wird in das vorbereitete Live-Home kopiert, wenn vorhanden, ist aber nicht der Hauptspeicher für Profilschlüssel)
- Lokale Live-Läufe kopieren standardmäßig die aktive Konfiguration, `auth-profiles.json`-Dateien pro Agent, das Legacy-`credentials/`-Verzeichnis und unterstützte externe CLI-Auth-Verzeichnisse in ein temporäres Test-Home; vorbereitete Live-Homes überspringen `workspace/` und `sandboxes/`, und `agents.*.workspace`- / `agentDir`-Pfad-Overrides werden entfernt, damit Sonden nicht in deinem echten Host-Workspace landen.

Wenn du auf Env-Schlüssel setzen willst (z. B. in deinem `~/.profile` exportiert), führe lokale Tests nach `source ~/.profile` aus oder verwende die Docker-Runner unten (sie können `~/.profile` in den Container einhängen).

## Deepgram Live (Audiotranskription)

- Test: `src/media-understanding/providers/deepgram/audio.live.test.ts`
- Aktivierung: `DEEPGRAM_API_KEY=... DEEPGRAM_LIVE_TEST=1 pnpm test:live src/media-understanding/providers/deepgram/audio.live.test.ts`

## BytePlus Coding-Plan Live

- Test: `src/agents/byteplus.live.test.ts`
- Aktivierung: `BYTEPLUS_API_KEY=... BYTEPLUS_LIVE_TEST=1 pnpm test:live src/agents/byteplus.live.test.ts`
- Optionales Modell-Override: `BYTEPLUS_CODING_MODEL=ark-code-latest`

## ComfyUI-Workflow-Medien-Live

- Test: `extensions/comfy/comfy.live.test.ts`
- Aktivierung: `OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts`
- Umfang:
  - Testet die gebündelten Comfy-Pfade für Bild, Video und `music_generate`
  - Überspringt jede Fähigkeit, sofern `models.providers.comfy.<capability>` nicht konfiguriert ist
  - Nützlich nach Änderungen an der Übermittlung von Comfy-Workflows, Polling, Downloads oder Plugin-Registrierung

## Bildgenerierung Live

- Test: `src/image-generation/runtime.live.test.ts`
- Befehl: `pnpm test:live src/image-generation/runtime.live.test.ts`
- Harness: `pnpm test:live:media image`
- Umfang:
  - Zählt jedes registrierte Plugin für Bildgenerierungs-Provider auf
  - Lädt fehlende Provider-Umgebungsvariablen vor dem Sondieren aus deiner Login-Shell (`~/.profile`)
  - Verwendet standardmäßig Live-/Env-API-Schlüssel vor gespeicherten Auth-Profilen, damit veraltete Testschlüssel in `auth-profiles.json` keine echten Shell-Anmeldedaten verdecken
  - Überspringt Provider ohne nutzbare Auth/Profile/Modelle
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
- Optionales Auth-Verhalten:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`, um Auth aus dem Profilspeicher zu erzwingen und nur über Env vorhandene Overrides zu ignorieren

## Musikgenerierung Live

- Test: `extensions/music-generation-providers.live.test.ts`
- Aktivierung: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts`
- Harness: `pnpm test:live:media music`
- Umfang:
  - Testet den gemeinsamen gebündelten Pfad für Musikgenerierungs-Provider
  - Deckt derzeit Google und MiniMax ab
  - Lädt Provider-Umgebungsvariablen vor dem Sondieren aus deiner Login-Shell (`~/.profile`)
  - Verwendet standardmäßig Live-/Env-API-Schlüssel vor gespeicherten Auth-Profilen, damit veraltete Testschlüssel in `auth-profiles.json` keine echten Shell-Anmeldedaten verdecken
  - Überspringt Provider ohne nutzbare Auth/Profile/Modelle
  - Führt beide deklarierten Runtime-Modi aus, wenn verfügbar:
    - `generate` mit reiner Prompt-Eingabe
    - `edit`, wenn der Provider `capabilities.edit.enabled` deklariert
  - Derzeitige Abdeckung der gemeinsamen Bahn:
    - `google`: `generate`, `edit`
    - `minimax`: `generate`
    - `comfy`: separate Comfy-Live-Datei, nicht dieser gemeinsame Sweep
- Optionale Eingrenzung:
  - `OPENCLAW_LIVE_MUSIC_GENERATION_PROVIDERS="google,minimax"`
  - `OPENCLAW_LIVE_MUSIC_GENERATION_MODELS="google/lyria-3-clip-preview,minimax/music-2.5+"`
- Optionales Auth-Verhalten:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`, um Auth aus dem Profilspeicher zu erzwingen und nur über Env vorhandene Overrides zu ignorieren

## Videogenerierung Live

- Test: `extensions/video-generation-providers.live.test.ts`
- Aktivierung: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/video-generation-providers.live.test.ts`
- Harness: `pnpm test:live:media video`
- Umfang:
  - Testet den gemeinsamen gebündelten Pfad für Videogenerierungs-Provider
  - Lädt Provider-Umgebungsvariablen vor dem Sondieren aus deiner Login-Shell (`~/.profile`)
  - Verwendet standardmäßig Live-/Env-API-Schlüssel vor gespeicherten Auth-Profilen, damit veraltete Testschlüssel in `auth-profiles.json` keine echten Shell-Anmeldedaten verdecken
  - Überspringt Provider ohne nutzbare Auth/Profile/Modelle
  - Führt beide deklarierten Runtime-Modi aus, wenn verfügbar:
    - `generate` mit reiner Prompt-Eingabe
    - `imageToVideo`, wenn der Provider `capabilities.imageToVideo.enabled` deklariert und der ausgewählte Provider/das ausgewählte Modell im gemeinsamen Sweep buffer-gestützte lokale Bildeingaben akzeptiert
    - `videoToVideo`, wenn der Provider `capabilities.videoToVideo.enabled` deklariert und der ausgewählte Provider/das ausgewählte Modell im gemeinsamen Sweep buffer-gestützte lokale Videoeingaben akzeptiert
  - Derzeit deklarierte, aber im gemeinsamen Sweep übersprungene `imageToVideo`-Provider:
    - `vydra`, weil das gebündelte `veo3` nur Text unterstützt und das gebündelte `kling` eine entfernte Bild-URL erfordert
  - Providerspezifische Vydra-Abdeckung:
    - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_VYDRA_VIDEO=1 pnpm test:live -- extensions/vydra/vydra.live.test.ts`
    - diese Datei führt `veo3` für Text-zu-Video sowie standardmäßig eine `kling`-Bahn aus, die eine Fixture mit entfernter Bild-URL verwendet
  - Derzeitige `videoToVideo`-Live-Abdeckung:
    - `runway` nur, wenn das ausgewählte Modell `runway/gen4_aleph` ist
  - Derzeit deklarierte, aber im gemeinsamen Sweep übersprungene `videoToVideo`-Provider:
    - `alibaba`, `qwen`, `xai`, weil diese Pfade derzeit entfernte Referenz-URLs `http(s)` / MP4 erfordern
    - `google`, weil die aktuelle gemeinsame Gemini-/Veo-Bahn lokal buffer-gestützte Eingaben verwendet und dieser Pfad im gemeinsamen Sweep nicht akzeptiert wird
    - `openai`, weil der aktuellen gemeinsamen Bahn Garantien für organisationsspezifischen Video-Inpaint-/Remix-Zugriff fehlen
- Optionale Eingrenzung:
  - `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="google,openai,runway"`
  - `OPENCLAW_LIVE_VIDEO_GENERATION_MODELS="google/veo-3.1-fast-generate-preview,openai/sora-2,runway/gen4_aleph"`
- Optionales Auth-Verhalten:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`, um Auth aus dem Profilspeicher zu erzwingen und nur über Env vorhandene Overrides zu ignorieren

## Medien-Live-Harness

- Befehl: `pnpm test:live:media`
- Zweck:
  - Führt die gemeinsamen Live-Suites für Bild, Musik und Video über einen repo-nativen Entry Point aus
  - Lädt fehlende Provider-Umgebungsvariablen automatisch aus `~/.profile`
  - Grenzt standardmäßig jede Suite automatisch auf Provider ein, die derzeit nutzbare Auth haben
  - Verwendet `scripts/test-live.mjs` erneut, sodass Heartbeat- und Quiet-Mode-Verhalten konsistent bleiben
- Beispiele:
  - `pnpm test:live:media`
  - `pnpm test:live:media image video --providers openai,google,minimax`
  - `pnpm test:live:media video --video-providers openai,runway --all-providers`
  - `pnpm test:live:media music --quiet`

## Docker-Runner (optionale „funktioniert unter Linux“-Prüfungen)

Diese Docker-Runner teilen sich in zwei Gruppen auf:

- Live-Modell-Runner: `test:docker:live-models` und `test:docker:live-gateway` führen nur ihre jeweils passende Live-Datei mit Profilschlüsseln im Docker-Image des Repos aus (`src/agents/models.profiles.live.test.ts` und `src/gateway/gateway-models.profiles.live.test.ts`), wobei dein lokales Konfigurationsverzeichnis und dein Workspace eingehängt werden (und `~/.profile` gesourct wird, falls eingehängt). Die passenden lokalen Entry Points sind `test:live:models-profiles` und `test:live:gateway-profiles`.
- Docker-Live-Runner verwenden standardmäßig ein kleineres Smoke-Limit, damit ein vollständiger Docker-Sweep praktikabel bleibt:
  `test:docker:live-models` verwendet standardmäßig `OPENCLAW_LIVE_MAX_MODELS=12`, und
  `test:docker:live-gateway` verwendet standardmäßig `OPENCLAW_LIVE_GATEWAY_SMOKE=1`,
  `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=8`,
  `OPENCLAW_LIVE_GATEWAY_STEP_TIMEOUT_MS=45000` und
  `OPENCLAW_LIVE_GATEWAY_MODEL_TIMEOUT_MS=90000`. Überschreibe diese Env-Variablen, wenn du
  bewusst den größeren vollständigen Scan möchtest.
- `test:docker:all` baut das Live-Docker-Image einmal über `test:docker:live-build` und verwendet es dann für die beiden Docker-Live-Bahnen erneut.
- Container-Smoke-Runner: `test:docker:openwebui`, `test:docker:onboard`, `test:docker:gateway-network`, `test:docker:mcp-channels` und `test:docker:plugins` starten einen oder mehrere echte Container und verifizieren Integrationspfade auf höherer Ebene.

Die Docker-Runner für Live-Modelle binden außerdem nur die benötigten CLI-Auth-Homes ein (oder alle unterstützten, wenn der Lauf nicht eingegrenzt ist) und kopieren sie dann vor dem Lauf in das Home im Container, damit externe CLI-OAuth Token aktualisieren kann, ohne den Auth-Speicher des Hosts zu verändern:

- Direkte Modelle: `pnpm test:docker:live-models` (Skript: `scripts/test-live-models-docker.sh`)
- ACP-Bind-Smoke: `pnpm test:docker:live-acp-bind` (Skript: `scripts/test-live-acp-bind-docker.sh`)
- CLI-Backend-Smoke: `pnpm test:docker:live-cli-backend` (Skript: `scripts/test-live-cli-backend-docker.sh`)
- Gateway + Dev-Agent: `pnpm test:docker:live-gateway` (Skript: `scripts/test-live-gateway-models-docker.sh`)
- Open-WebUI-Live-Smoke: `pnpm test:docker:openwebui` (Skript: `scripts/e2e/openwebui-docker.sh`)
- Onboarding-Assistent (TTY, vollständiges Scaffolding): `pnpm test:docker:onboard` (Skript: `scripts/e2e/onboard-docker.sh`)
- Gateway-Netzwerk (zwei Container, WS-Auth + Health): `pnpm test:docker:gateway-network` (Skript: `scripts/e2e/gateway-network-docker.sh`)
- MCP-Channel-Bridge (mit Seed versehenes Gateway + stdio-Bridge + roher Claude-Benachrichtigungsframe-Smoke): `pnpm test:docker:mcp-channels` (Skript: `scripts/e2e/mcp-channels-docker.sh`)
- Plugins (Install-Smoke + `/plugin`-Alias + Neustartsemantik des Claude-Bundles): `pnpm test:docker:plugins` (Skript: `scripts/e2e/plugins-docker.sh`)

Die Docker-Runner für Live-Modelle binden außerdem den aktuellen Checkout schreibgeschützt ein und
stellen ihn in einem temporären Arbeitsverzeichnis im Container bereit. Dadurch bleibt das Runtime-
Image schlank, während Vitest trotzdem gegen deinen exakten lokalen Quellcode/deine Konfiguration ausgeführt wird.
Der Bereitstellungsschritt überspringt große lokale Caches und App-Build-Ausgaben wie
`.pnpm-store`, `.worktrees`, `__openclaw_vitest__` und app-lokale `.build`- oder
Gradle-Ausgabeverzeichnisse, damit Docker-Live-Läufe nicht minutenlang maschinenspezifische
Artefakte kopieren.
Sie setzen außerdem `OPENCLAW_SKIP_CHANNELS=1`, damit Gateway-Live-Sonden im Container
keine echten Worker für Telegram/Discord/usw. starten.
`test:docker:live-models` führt weiterhin `pnpm test:live` aus, gib also
auch `OPENCLAW_LIVE_GATEWAY_*` weiter, wenn du Gateway-Live-Abdeckung in dieser Docker-Bahn
eingrenzen oder ausschließen musst.
`test:docker:openwebui` ist ein Kompatibilitäts-Smoke auf höherer Ebene: Er startet einen
OpenClaw-Gateway-Container mit aktivierten OpenAI-kompatiblen HTTP-Endpoints, startet einen
angepinnten Open-WebUI-Container gegen dieses Gateway, meldet sich über
Open WebUI an, verifiziert, dass `/api/models` `openclaw/default` bereitstellt, und sendet dann
eine echte Chat-Anfrage über den Proxy `/api/chat/completions` von Open WebUI.
Der erste Lauf kann deutlich langsamer sein, weil Docker möglicherweise erst das
Open-WebUI-Image ziehen muss und Open WebUI möglicherweise seinen eigenen Kaltstart abschließen muss.
Diese Bahn erwartet einen nutzbaren Live-Modellschlüssel, und `OPENCLAW_PROFILE_FILE`
(`~/.profile` standardmäßig) ist der primäre Weg, ihn in Docker-Läufen bereitzustellen.
Erfolgreiche Läufe geben ein kleines JSON-Payload wie `{ "ok": true, "model":
"openclaw/default", ... }` aus.
`test:docker:mcp-channels` ist bewusst deterministisch und benötigt
keinen echten Telegram-, Discord- oder iMessage-Account. Es startet ein mit Seed versehenes Gateway-
Container, startet einen zweiten Container, der `openclaw mcp serve` startet, und
verifiziert dann geroutete Conversation-Erkennung, Transkript-Lesezugriffe, Metadaten von Anhängen,
Verhalten der Live-Event-Queue, ausgehendes Send-Routing sowie Benachrichtigungen im Claude-Stil zu Channel +
Berechtigungen über die echte stdio-MCP-Bridge. Die Benachrichtigungsprüfung
untersucht die rohen stdio-MCP-Frames direkt, sodass der Smoke validiert, was die
Bridge tatsächlich ausgibt, und nicht nur das, was ein bestimmtes Client-SDK zufällig sichtbar macht.

Manueller ACP-Plain-Language-Thread-Smoke (nicht CI):

- `bun scripts/dev/discord-acp-plain-language-smoke.ts --channel <discord-channel-id> ...`
- Behalte dieses Skript für Regressions-/Debug-Workflows. Es könnte für die Validierung des ACP-Thread-Routings wieder benötigt werden, also nicht löschen.

Nützliche Env-Variablen:

- `OPENCLAW_CONFIG_DIR=...` (Standard: `~/.openclaw`), eingehängt nach `/home/node/.openclaw`
- `OPENCLAW_WORKSPACE_DIR=...` (Standard: `~/.openclaw/workspace`), eingehängt nach `/home/node/.openclaw/workspace`
- `OPENCLAW_PROFILE_FILE=...` (Standard: `~/.profile`), eingehängt nach `/home/node/.profile` und vor dem Ausführen der Tests gesourct
- `OPENCLAW_DOCKER_CLI_TOOLS_DIR=...` (Standard: `~/.cache/openclaw/docker-cli-tools`), eingehängt nach `/home/node/.npm-global` für zwischengespeicherte CLI-Installationen in Docker
- Externe CLI-Auth-Verzeichnisse/-Dateien unter `$HOME` werden schreibgeschützt unter `/host-auth...` eingehängt und dann vor Testbeginn nach `/home/node/...` kopiert
  - Standardverzeichnisse: `.minimax`
  - Standarddateien: `~/.codex/auth.json`, `~/.codex/config.toml`, `.claude.json`, `~/.claude/.credentials.json`, `~/.claude/settings.json`, `~/.claude/settings.local.json`
  - Eingegrenzte Provider-Läufe hängen nur die benötigten Verzeichnisse/Dateien ein, die aus `OPENCLAW_LIVE_PROVIDERS` / `OPENCLAW_LIVE_GATEWAY_PROVIDERS` abgeleitet werden
  - Manuelles Override mit `OPENCLAW_DOCKER_AUTH_DIRS=all`, `OPENCLAW_DOCKER_AUTH_DIRS=none` oder einer Komma-Liste wie `OPENCLAW_DOCKER_AUTH_DIRS=.claude,.codex`
- `OPENCLAW_LIVE_GATEWAY_MODELS=...` / `OPENCLAW_LIVE_MODELS=...`, um den Lauf einzugrenzen
- `OPENCLAW_LIVE_GATEWAY_PROVIDERS=...` / `OPENCLAW_LIVE_PROVIDERS=...`, um Provider im Container zu filtern
- `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`, um sicherzustellen, dass Anmeldedaten aus dem Profilspeicher kommen (nicht aus Env)
- `OPENCLAW_OPENWEBUI_MODEL=...`, um das Modell auszuwählen, das vom Gateway für den Open-WebUI-Smoke bereitgestellt wird
- `OPENCLAW_OPENWEBUI_PROMPT=...`, um den für den Open-WebUI-Smoke verwendeten Nonce-Check-Prompt zu überschreiben
- `OPENWEBUI_IMAGE=...`, um den angepinnten Open-WebUI-Image-Tag zu überschreiben

## Docs-Sanity-Check

Führe nach Änderungen an der Dokumentation die Docs-Checks aus: `pnpm check:docs`.
Führe die vollständige Mintlify-Anchor-Validierung aus, wenn du auch Heading-Prüfungen innerhalb einer Seite brauchst: `pnpm docs:check-links:anchors`.

## Offline-Regression (CI-sicher)

Dies sind Regressionstests für „echte Pipelines“ ohne echte Provider:

- Gateway-Tool Calling (gemocktes OpenAI, echtes Gateway + Agent-Schleife): `src/gateway/gateway.test.ts` (Fall: "runs a mock OpenAI tool call end-to-end via gateway agent loop")
- Gateway-Assistent (WS `wizard.start`/`wizard.next`, schreibt Konfiguration + Auth erzwungen): `src/gateway/gateway.test.ts` (Fall: "runs wizard over ws and writes auth token config")

## Agenten-Zuverlässigkeits-Evals (Skills)

Wir haben bereits einige CI-sichere Tests, die sich wie „Agenten-Zuverlässigkeits-Evals“ verhalten:

- Gemocktes Tool Calling über die echte Gateway- + Agent-Schleife (`src/gateway/gateway.test.ts`).
- End-to-End-Assistenten-Flows, die Session-Wiring und Konfigurationseffekte validieren (`src/gateway/gateway.test.ts`).

Was für Skills noch fehlt (siehe [Skills](/de/tools/skills)):

- **Entscheidungsfindung:** Wenn Skills im Prompt aufgeführt sind, wählt der Agent den richtigen Skill (oder vermeidet irrelevante)?
- **Compliance:** Liest der Agent vor der Verwendung `SKILL.md` und befolgt die erforderlichen Schritte/Argumente?
- **Workflow-Verträge:** Multi-Turn-Szenarien, die Tool-Reihenfolge, Übernahme des Session-Verlaufs und Sandbox-Grenzen prüfen.

Zukünftige Evals sollten zunächst deterministisch bleiben:

- Ein Szenario-Runner, der Mock-Provider verwendet, um Tool-Aufrufe + Reihenfolge, Skill-Datei-Lesevorgänge und Session-Wiring zu prüfen.
- Eine kleine Suite von skillfokussierten Szenarien (verwenden vs. vermeiden, Gating, Prompt Injection).
- Optionale Live-Evals (Opt-in, Env-gated) erst, nachdem die CI-sichere Suite vorhanden ist.

## Vertragstests (Plugin- und Channel-Form)

Vertragstests verifizieren, dass jedes registrierte Plugin und jeder Channel seinem
Schnittstellenvertrag entspricht. Sie iterieren über alle gefundenen Plugins und führen
eine Suite von Form- und Verhaltensprüfungen aus. Die standardmäßige Unit-Bahn `pnpm test`
überspringt diese gemeinsamen Seam- und Smoke-Dateien bewusst; führe die Vertragsbefehle explizit
aus, wenn du gemeinsame Channel- oder Provider-Oberflächen anfasst.

### Befehle

- Alle Verträge: `pnpm test:contracts`
- Nur Channel-Verträge: `pnpm test:contracts:channels`
- Nur Provider-Verträge: `pnpm test:contracts:plugins`

### Channel-Verträge

Unter `src/channels/plugins/contracts/*.contract.test.ts`:

- **plugin** - Grundlegende Plugin-Form (ID, Name, Fähigkeiten)
- **setup** - Vertrag des Setup-Assistenten
- **session-binding** - Verhalten der Session-Bindung
- **outbound-payload** - Struktur des Message-Payloads
- **inbound** - Behandlung eingehender Nachrichten
- **actions** - Handler für Channel-Aktionen
- **threading** - Behandlung von Thread-IDs
- **directory** - API für Verzeichnis/Roster
- **group-policy** - Durchsetzung von Gruppenrichtlinien

### Provider-Status-Verträge

Unter `src/plugins/contracts/*.contract.test.ts`.

- **status** - Channel-Status-Sonden
- **registry** - Form der Plugin-Registry

### Provider-Verträge

Unter `src/plugins/contracts/*.contract.test.ts`:

- **auth** - Vertrag des Auth-Flows
- **auth-choice** - Auth-Auswahl/-Selektion
- **catalog** - API des Modellkatalogs
- **discovery** - Plugin-Erkennung
- **loader** - Plugin-Laden
- **runtime** - Provider-Runtime
- **shape** - Plugin-Form/-Schnittstelle
- **wizard** - Setup-Assistent

### Wann ausführen

- Nach Änderungen an `plugin-sdk`-Exports oder Subpaths
- Nach dem Hinzufügen oder Ändern eines Channel- oder Provider-Plugins
- Nach Refactorings an Plugin-Registrierung oder -Erkennung

Vertragstests laufen in CI und benötigen keine echten API-Schlüssel.

## Regressionen hinzufügen (Leitfaden)

Wenn du ein in Live entdecktes Provider-/Modellproblem behebst:

- Füge möglichst einen CI-sicheren Regressionstest hinzu (Mock/Stub-Provider oder erfasse die exakte Transformation der Request-Form)
- Wenn es von Natur aus nur live testbar ist (Rate-Limits, Auth-Richtlinien), halte den Live-Test eng eingegrenzt und per Env-Variablen opt-in
- Bevorzuge die kleinste Ebene, die den Fehler erfasst:
  - Fehler bei Provider-Request-Konvertierung/-Replay → Test für direkte Modelle
  - Fehler in Gateway-Session-/Verlaufs-/Tool-Pipeline → Gateway-Live-Smoke oder CI-sicherer Gateway-Mock-Test
- Guardrail für SecretRef-Traversal:
  - `src/secrets/exec-secret-ref-id-parity.test.ts` leitet aus Registry-Metadaten (`listSecretTargetRegistryEntries()`) ein Beispielziel pro SecretRef-Klasse ab und prüft dann, dass Exec-IDs für Traversal-Segmente abgelehnt werden.
  - Wenn du in `src/secrets/target-registry-data.ts` eine neue SecretRef-Zielfamilie `includeInPlan` hinzufügst, aktualisiere `classifyTargetClass` in diesem Test. Der Test schlägt absichtlich bei nicht klassifizierten Ziel-IDs fehl, damit neue Klassen nicht stillschweigend übersprungen werden.
