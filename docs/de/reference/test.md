---
read_when:
    - Tests ausführen oder beheben
summary: Wie Tests lokal ausgeführt werden (vitest) und wann force-/coverage-Modi verwendet werden sollten
title: Tests
x-i18n:
    generated_at: "2026-04-08T02:18:40Z"
    model: gpt-5.4
    provider: openai
    source_hash: f7c19390f7577b3a29796c67514c96fe4c86c9fa0c7686cd4e377c6e31dcd085
    source_path: reference/test.md
    workflow: 15
---

# Tests

- Vollständiges Test-Kit (Suites, Live, Docker): [Testing](/de/help/testing)

- `pnpm test:force`: Beendet alle verbliebenen Gateway-Prozesse, die den Standard-Control-Port belegen, und führt dann die vollständige Vitest-Suite mit einem isolierten Gateway-Port aus, damit Server-Tests nicht mit einer laufenden Instanz kollidieren. Verwenden Sie dies, wenn ein vorheriger Gateway-Lauf Port 18789 belegt gelassen hat.
- `pnpm test:coverage`: Führt die Unit-Suite mit V8-Coverage aus (über `vitest.unit.config.ts`). Globale Schwellenwerte sind 70 % für Zeilen/Branches/Funktionen/Statements. Coverage schließt integrationslastige Entry-Points aus (CLI-Verdrahtung, Gateway-/Telegram-Bridges, statischer Webchat-Server), damit das Ziel auf unit-testbare Logik fokussiert bleibt.
- `pnpm test:coverage:changed`: Führt Unit-Coverage nur für Dateien aus, die sich seit `origin/main` geändert haben.
- `pnpm test:changed`: erweitert geänderte Git-Pfade zu gezielten Vitest-Lanes, wenn der Diff nur routingfähige Quell-/Testdateien berührt. Änderungen an Konfiguration/Setup fallen weiterhin auf den nativen Root-Projects-Lauf zurück, damit Verdrahtungsänderungen bei Bedarf breit erneut ausgeführt werden.
- `pnpm test`: leitet explizite Datei-/Verzeichnisziele durch gezielte Vitest-Lanes. Nicht zielgerichtete Läufe führen jetzt elf sequentielle Shard-Konfigurationen aus (`vitest.full-core-unit-src.config.ts`, `vitest.full-core-unit-security.config.ts`, `vitest.full-core-unit-ui.config.ts`, `vitest.full-core-unit-support.config.ts`, `vitest.full-core-support-boundary.config.ts`, `vitest.full-core-contracts.config.ts`, `vitest.full-core-bundled.config.ts`, `vitest.full-core-runtime.config.ts`, `vitest.full-agentic.config.ts`, `vitest.full-auto-reply.config.ts`, `vitest.full-extensions.config.ts`) statt eines einzigen riesigen Root-Project-Prozesses.
- Ausgewählte Testdateien in `plugin-sdk` und `commands` werden jetzt durch dedizierte leichte Lanes geleitet, die nur `test/setup.ts` beibehalten, während laufzeitintensive Fälle auf ihren bestehenden Lanes bleiben.
- Ausgewählte Hilfs-Quelldateien in `plugin-sdk` und `commands` ordnen `pnpm test:changed` ebenfalls expliziten benachbarten Tests in diesen leichten Lanes zu, sodass kleine Hilfsänderungen das erneute Ausführen der schweren, laufzeitgestützten Suites vermeiden.
- `auto-reply` ist jetzt ebenfalls in drei dedizierte Konfigurationen aufgeteilt (`core`, `top-level`, `reply`), damit das Reply-Harness die leichteren Top-Level-Tests für Status/Token/Helper nicht dominiert.
- Die Basis-Vitest-Konfiguration verwendet jetzt standardmäßig `pool: "threads"` und `isolate: false`, wobei der gemeinsam genutzte nicht isolierte Runner in den Repo-Konfigurationen aktiviert ist.
- `pnpm test:channels` führt `vitest.channels.config.ts` aus.
- `pnpm test:extensions` führt `vitest.extensions.config.ts` aus.
- `pnpm test:extensions`: führt Extension-/Plugin-Suites aus.
- `pnpm test:perf:imports`: aktiviert die Berichterstattung zu Vitest-Importdauer und Importaufschlüsselung und verwendet dabei weiterhin gezieltes Lane-Routing für explizite Datei-/Verzeichnisziele.
- `pnpm test:perf:imports:changed`: dasselbe Import-Profiling, aber nur für Dateien, die sich seit `origin/main` geändert haben.
- `pnpm test:perf:changed:bench -- --ref <git-ref>` benchmarkt den gerouteten Changed-Mode-Pfad gegen den nativen Root-Project-Lauf für denselben committeten Git-Diff.
- `pnpm test:perf:changed:bench -- --worktree` benchmarkt das aktuelle Worktree-Änderungsset, ohne vorher zu committen.
- `pnpm test:perf:profile:main`: schreibt ein CPU-Profil für den Vitest-Hauptthread (`.artifacts/vitest-main-profile`).
- `pnpm test:perf:profile:runner`: schreibt CPU- und Heap-Profile für den Unit-Runner (`.artifacts/vitest-runner-profile`).
- Gateway-Integration: Opt-in über `OPENCLAW_TEST_INCLUDE_GATEWAY=1 pnpm test` oder `pnpm test:gateway`.
- `pnpm test:e2e`: Führt Gateway-End-to-End-Smoke-Tests aus (Pairing mit mehreren Instanzen über WS/HTTP/Node). Verwendet standardmäßig `threads` + `isolate: false` mit adaptiven Workern in `vitest.e2e.config.ts`; passen Sie dies mit `OPENCLAW_E2E_WORKERS=<n>` an und setzen Sie `OPENCLAW_E2E_VERBOSE=1` für ausführliche Logs.
- `pnpm test:live`: Führt Provider-Live-Tests aus (minimax/zai). Erfordert API-Schlüssel und `LIVE=1` (oder providerspezifisch `*_LIVE_TEST=1`), damit Tests nicht übersprungen werden.
- `pnpm test:docker:openwebui`: Startet ein Docker-basiertes OpenClaw + Open WebUI, meldet sich über Open WebUI an, prüft `/api/models` und führt dann einen echten proxied Chat über `/api/chat/completions` aus. Erfordert einen verwendbaren Live-Modellschlüssel (zum Beispiel OpenAI in `~/.profile`), zieht ein externes Open-WebUI-Image und ist nicht dafür gedacht, wie normale Unit-/E2E-Suites CI-stabil zu sein.
- `pnpm test:docker:mcp-channels`: Startet einen vorbereiteten Gateway-Container und einen zweiten Client-Container, der `openclaw mcp serve` startet, und verifiziert dann geroutete Konversationserkennung, Transkript-Lesevorgänge, Anhangsmetadaten, Verhalten der Live-Ereigniswarteschlange, ausgehendes Send-Routing und Claude-artige Kanal- und Berechtigungsbenachrichtigungen über die echte stdio-Bridge. Die Claude-Benachrichtigungs-Assertion liest die rohen stdio-MCP-Frames direkt, sodass der Smoke-Test widerspiegelt, was die Bridge tatsächlich ausgibt.

## Lokales PR-Gate

Führen Sie für lokale PR-Land-/Gate-Prüfungen Folgendes aus:

- `pnpm check`
- `pnpm build`
- `pnpm test`
- `pnpm check:docs`

Wenn `pnpm test` auf einem stark ausgelasteten Host flakey ist, führen Sie es einmal erneut aus, bevor Sie es als Regression behandeln, und isolieren Sie es dann mit `pnpm test <path/to/test>`. Für Hosts mit eingeschränktem Arbeitsspeicher verwenden Sie:

- `OPENCLAW_VITEST_MAX_WORKERS=1 pnpm test`
- `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/tmp/openclaw-vitest-cache pnpm test:changed`

## Modell-Latenz-Benchmark (lokale Schlüssel)

Skript: [`scripts/bench-model.ts`](https://github.com/openclaw/openclaw/blob/main/scripts/bench-model.ts)

Verwendung:

- `source ~/.profile && pnpm tsx scripts/bench-model.ts --runs 10`
- Optionale Umgebungsvariablen: `MINIMAX_API_KEY`, `MINIMAX_BASE_URL`, `MINIMAX_MODEL`, `ANTHROPIC_API_KEY`
- Standard-Prompt: „Reply with a single word: ok. No punctuation or extra text.“

Letzter Lauf (2025-12-31, 20 Läufe):

- minimax Median 1279 ms (min. 1114, max. 2431)
- opus Median 2454 ms (min. 1224, max. 3170)

## CLI-Start-Benchmark

Skript: [`scripts/bench-cli-startup.ts`](https://github.com/openclaw/openclaw/blob/main/scripts/bench-cli-startup.ts)

Verwendung:

- `pnpm test:startup:bench`
- `pnpm test:startup:bench:smoke`
- `pnpm test:startup:bench:save`
- `pnpm test:startup:bench:update`
- `pnpm test:startup:bench:check`
- `pnpm tsx scripts/bench-cli-startup.ts`
- `pnpm tsx scripts/bench-cli-startup.ts --runs 12`
- `pnpm tsx scripts/bench-cli-startup.ts --preset real`
- `pnpm tsx scripts/bench-cli-startup.ts --preset real --case status --case gatewayStatus --runs 3`
- `pnpm tsx scripts/bench-cli-startup.ts --entry openclaw.mjs --entry-secondary dist/entry.js --preset all`
- `pnpm tsx scripts/bench-cli-startup.ts --preset all --output .artifacts/cli-startup-bench-all.json`
- `pnpm tsx scripts/bench-cli-startup.ts --preset real --case gatewayStatusJson --output .artifacts/cli-startup-bench-smoke.json`
- `pnpm tsx scripts/bench-cli-startup.ts --preset real --cpu-prof-dir .artifacts/cli-cpu`
- `pnpm tsx scripts/bench-cli-startup.ts --json`

Voreinstellungen:

- `startup`: `--version`, `--help`, `health`, `health --json`, `status --json`, `status`
- `real`: `health`, `status`, `status --json`, `sessions`, `sessions --json`, `agents list --json`, `gateway status`, `gateway status --json`, `gateway health --json`, `config get gateway.port`
- `all`: beide Voreinstellungen

Die Ausgabe enthält `sampleCount`, Durchschnitt, p50, p95, Min./Max., Verteilung von Exit-Code/Signal und Zusammenfassungen des maximalen RSS für jeden Befehl. Mit optionalem `--cpu-prof-dir` / `--heap-prof-dir` werden V8-Profile pro Lauf geschrieben, sodass Zeitmessung und Profilerfassung dasselbe Harness verwenden.

Konventionen für gespeicherte Ausgaben:

- `pnpm test:startup:bench:smoke` schreibt das gezielte Smoke-Artefakt nach `.artifacts/cli-startup-bench-smoke.json`
- `pnpm test:startup:bench:save` schreibt das Artefakt der vollständigen Suite nach `.artifacts/cli-startup-bench-all.json` mit `runs=5` und `warmup=1`
- `pnpm test:startup:bench:update` aktualisiert die eingecheckte Baseline-Fixture in `test/fixtures/cli-startup-bench.json` mit `runs=5` und `warmup=1`

Eingecheckte Fixture:

- `test/fixtures/cli-startup-bench.json`
- Aktualisieren mit `pnpm test:startup:bench:update`
- Aktuelle Ergebnisse mit der Fixture vergleichen mit `pnpm test:startup:bench:check`

## Onboarding-E2E (Docker)

Docker ist optional; dies wird nur für containerisierte Onboarding-Smoke-Tests benötigt.

Vollständiger Kaltstart-Ablauf in einem sauberen Linux-Container:

```bash
scripts/e2e/onboard-docker.sh
```

Dieses Skript steuert den interaktiven Assistenten über ein Pseudo-TTY, verifiziert Konfigurations-/Workspace-/Sitzungsdateien, startet dann das Gateway und führt `openclaw health` aus.

## QR-Import-Smoke (Docker)

Stellt sicher, dass `qrcode-terminal` unter den unterstützten Docker-Node-Laufzeiten geladen wird (standardmäßig Node 24, kompatibel mit Node 22):

```bash
pnpm test:docker:qr
```
