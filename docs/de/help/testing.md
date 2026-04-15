---
read_when:
    - Tests lokal oder in CI ausführen
    - Regressionstests für Modell-/Provider-Fehler hinzufügen
    - Gateway- und Agent-Verhalten debuggen
summary: 'Test-Kit: Unit-/E2E-/Live-Suites, Docker-Runner und was jeder Test abdeckt'
title: Testen
x-i18n:
    generated_at: "2026-04-15T06:21:26Z"
    model: gpt-5.4
    provider: openai
    source_hash: fbf647a5cf13b5861a3ba0cb367dc816c57f0e9c60d3cd6320da193bfadf5609
    source_path: help/testing.md
    workflow: 15
---

# Testen

OpenClaw hat drei Vitest-Suites (Unit/Integration, E2E, Live) und eine kleine Anzahl von Docker-Runnern.

Dieses Dokument ist ein Leitfaden dazu, „wie wir testen“:

- Was jede Suite abdeckt (und was sie bewusst _nicht_ abdeckt)
- Welche Befehle für gängige Workflows auszuführen sind (lokal, vor dem Push, Debugging)
- Wie Live-Tests Anmeldedaten erkennen und Modelle/Provider auswählen
- Wie Regressionstests für reale Modell-/Provider-Probleme hinzugefügt werden

## Schnellstart

An den meisten Tagen:

- Vollständiges Gate (vor dem Push erwartet): `pnpm build && pnpm check && pnpm test`
- Schnellere lokale Ausführung der vollständigen Suite auf einer leistungsstarken Maschine: `pnpm test:max`
- Direkte Vitest-Watch-Schleife: `pnpm test:watch`
- Direktes Datei-Targeting leitet jetzt auch Erweiterungs-/Kanalpfade weiter: `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts`
- Bevorzuge zuerst gezielte Ausführungen, wenn du an einem einzelnen Fehler arbeitest.
- Docker-gestützte QA-Site: `pnpm qa:lab:up`
- Linux-VM-gestützte QA-Lane: `pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline`

Wenn du Tests anfasst oder zusätzliche Sicherheit möchtest:

- Coverage-Gate: `pnpm test:coverage`
- E2E-Suite: `pnpm test:e2e`

Beim Debuggen realer Provider/Modelle (erfordert echte Anmeldedaten):

- Live-Suite (Modelle + Gateway-Tool-/Bild-Probes): `pnpm test:live`
- Eine Live-Datei gezielt und ohne viel Ausgabe ausführen: `pnpm test:live -- src/agents/models.profiles.live.test.ts`

Tipp: Wenn du nur einen einzelnen fehlschlagenden Fall brauchst, grenze Live-Tests bevorzugt über die unten beschriebenen Allowlist-Umgebungsvariablen ein.

## QA-spezifische Runner

Diese Befehle stehen neben den Haupt-Test-Suites zur Verfügung, wenn du die Realitätsnähe von qa-lab brauchst:

- `pnpm openclaw qa suite`
  - Führt repo-gestützte QA-Szenarien direkt auf dem Host aus.
  - Führt standardmäßig mehrere ausgewählte Szenarien parallel mit isolierten Gateway-Workern aus, bis zu 64 Worker oder die Anzahl der ausgewählten Szenarien. Verwende `--concurrency <count>`, um die Anzahl der Worker anzupassen, oder `--concurrency 1` für die ältere serielle Lane.
- `pnpm openclaw qa suite --runner multipass`
  - Führt dieselbe QA-Suite innerhalb einer flüchtigen Multipass-Linux-VM aus.
  - Behält dasselbe Verhalten zur Szenarioauswahl wie `qa suite` auf dem Host bei.
  - Verwendet dieselben Flags zur Provider-/Modellauswahl wie `qa suite`.
  - Live-Ausführungen leiten die unterstützten QA-Authentifizierungseingaben weiter, die für den Gast praktikabel sind:
    env-basierte Provider-Schlüssel, den Pfad zur QA-Live-Provider-Konfiguration und `CODEX_HOME`, wenn vorhanden.
  - Ausgabeverzeichnisse müssen unter dem Repo-Root bleiben, damit der Gast über den eingehängten Workspace zurückschreiben kann.
  - Schreibt den normalen QA-Bericht + die Zusammenfassung sowie Multipass-Logs unter
    `.artifacts/qa-e2e/...`.
- `pnpm qa:lab:up`
  - Startet die Docker-gestützte QA-Site für operatorähnliche QA-Arbeit.
- `pnpm openclaw qa matrix`
  - Führt die Matrix-Live-QA-Lane gegen einen flüchtigen, Docker-gestützten Tuwunel-Homeserver aus.
  - Dieser QA-Host ist derzeit nur für Repo/Entwicklung gedacht. Gepackte OpenClaw-Installationen enthalten kein `qa-lab`, daher stellen sie `openclaw qa` nicht bereit.
  - Repo-Checkouts laden den gebündelten Runner direkt; ein separater Installationsschritt für Plugins ist nicht erforderlich.
  - Stellt drei temporäre Matrix-Benutzer (`driver`, `sut`, `observer`) sowie einen privaten Raum bereit und startet dann ein QA-Gateway-Child mit dem echten Matrix-Plugin als SUT-Transport.
  - Verwendet standardmäßig das angeheftete stabile Tuwunel-Image `ghcr.io/matrix-construct/tuwunel:v1.5.1`. Überschreibe es mit `OPENCLAW_QA_MATRIX_TUWUNEL_IMAGE`, wenn du ein anderes Image testen musst.
  - Matrix stellt keine gemeinsamen Flags für Anmeldedatenquellen bereit, weil die Lane lokal flüchtige Benutzer bereitstellt.
  - Schreibt einen Matrix-QA-Bericht, eine Zusammenfassung und ein Artefakt mit beobachteten Ereignissen unter `.artifacts/qa-e2e/...`.
- `pnpm openclaw qa telegram`
  - Führt die Telegram-Live-QA-Lane gegen eine reale private Gruppe aus, wobei die Bot-Tokens von Driver und SUT aus der Umgebung verwendet werden.
  - Erfordert `OPENCLAW_QA_TELEGRAM_GROUP_ID`, `OPENCLAW_QA_TELEGRAM_DRIVER_BOT_TOKEN` und `OPENCLAW_QA_TELEGRAM_SUT_BOT_TOKEN`. Die Gruppen-ID muss die numerische Telegram-Chat-ID sein.
  - Unterstützt `--credential-source convex` für gemeinsam genutzte gepoolte Anmeldedaten. Verwende standardmäßig den env-Modus oder setze `OPENCLAW_QA_CREDENTIAL_SOURCE=convex`, um gepoolte Leases zu verwenden.
  - Erfordert zwei unterschiedliche Bots in derselben privaten Gruppe, wobei der SUT-Bot einen Telegram-Benutzernamen bereitstellen muss.
  - Für stabile Bot-zu-Bot-Beobachtung aktiviere in `@BotFather` den Bot-to-Bot Communication Mode für beide Bots und stelle sicher, dass der Driver-Bot Bot-Datenverkehr in der Gruppe beobachten kann.
  - Schreibt einen Telegram-QA-Bericht, eine Zusammenfassung und ein Artefakt mit beobachteten Nachrichten unter `.artifacts/qa-e2e/...`.

Live-Transport-Lanes teilen sich einen standardisierten Vertrag, damit neue Transporte nicht voneinander abweichen:

`qa-channel` bleibt die breite synthetische QA-Suite und ist nicht Teil der Live-Transport-Abdeckungsmatrix.

| Lane     | Canary | Mention-Gating | Allowlist-Block | Antwort auf oberster Ebene | Fortsetzen nach Neustart | Thread-Follow-up | Thread-Isolierung | Beobachtung von Reaktionen | Hilfe-Befehl |
| -------- | ------ | -------------- | --------------- | -------------------------- | ------------------------ | ---------------- | ----------------- | -------------------------- | ------------ |
| Matrix   | x      | x              | x               | x                          | x                        | x                | x                 | x                          |              |
| Telegram | x      |                |                 |                            |                          |                  |                   |                            | x            |

### Gemeinsame Telegram-Anmeldedaten über Convex (v1)

Wenn `--credential-source convex` (oder `OPENCLAW_QA_CREDENTIAL_SOURCE=convex`) für
`openclaw qa telegram` aktiviert ist, bezieht QA lab ein exklusives Lease aus einem Convex-gestützten Pool, sendet Heartbeat-Signale für dieses Lease, während die Lane läuft, und gibt das Lease beim Beenden frei.

Referenzgerüst für ein Convex-Projekt:

- `qa/convex-credential-broker/`

Erforderliche Umgebungsvariablen:

- `OPENCLAW_QA_CONVEX_SITE_URL` (zum Beispiel `https://your-deployment.convex.site`)
- Ein Secret für die ausgewählte Rolle:
  - `OPENCLAW_QA_CONVEX_SECRET_MAINTAINER` für `maintainer`
  - `OPENCLAW_QA_CONVEX_SECRET_CI` für `ci`
- Auswahl der Anmeldedatenrolle:
  - CLI: `--credential-role maintainer|ci`
  - Standard in der Umgebung: `OPENCLAW_QA_CREDENTIAL_ROLE` (Standard ist `maintainer`)

Optionale Umgebungsvariablen:

- `OPENCLAW_QA_CREDENTIAL_LEASE_TTL_MS` (Standard `1200000`)
- `OPENCLAW_QA_CREDENTIAL_HEARTBEAT_INTERVAL_MS` (Standard `30000`)
- `OPENCLAW_QA_CREDENTIAL_ACQUIRE_TIMEOUT_MS` (Standard `90000`)
- `OPENCLAW_QA_CREDENTIAL_HTTP_TIMEOUT_MS` (Standard `15000`)
- `OPENCLAW_QA_CONVEX_ENDPOINT_PREFIX` (Standard `/qa-credentials/v1`)
- `OPENCLAW_QA_CREDENTIAL_OWNER_ID` (optionale Trace-ID)
- `OPENCLAW_QA_ALLOW_INSECURE_HTTP=1` erlaubt loopback-`http://`-Convex-URLs für rein lokale Entwicklung.

`OPENCLAW_QA_CONVEX_SITE_URL` sollte im normalen Betrieb `https://` verwenden.

Maintainer-Admin-Befehle (Pool hinzufügen/entfernen/auflisten) erfordern
explizit `OPENCLAW_QA_CONVEX_SECRET_MAINTAINER`.

CLI-Hilfsbefehle für Maintainer:

```bash
pnpm openclaw qa credentials add --kind telegram --payload-file qa/telegram-credential.json
pnpm openclaw qa credentials list --kind telegram
pnpm openclaw qa credentials remove --credential-id <credential-id>
```

Verwende `--json` für maschinenlesbare Ausgabe in Skripten und CI-Hilfsprogrammen.

Standard-Endpoint-Vertrag (`OPENCLAW_QA_CONVEX_SITE_URL` + `/qa-credentials/v1`):

- `POST /acquire`
  - Anfrage: `{ kind, ownerId, actorRole, leaseTtlMs, heartbeatIntervalMs }`
  - Erfolg: `{ status: "ok", credentialId, leaseToken, payload, leaseTtlMs?, heartbeatIntervalMs? }`
  - Erschöpft/wiederholbar: `{ status: "error", code: "POOL_EXHAUSTED" | "NO_CREDENTIAL_AVAILABLE", ... }`
- `POST /heartbeat`
  - Anfrage: `{ kind, ownerId, actorRole, credentialId, leaseToken, leaseTtlMs }`
  - Erfolg: `{ status: "ok" }` (oder leeres `2xx`)
- `POST /release`
  - Anfrage: `{ kind, ownerId, actorRole, credentialId, leaseToken }`
  - Erfolg: `{ status: "ok" }` (oder leeres `2xx`)
- `POST /admin/add` (nur mit Maintainer-Secret)
  - Anfrage: `{ kind, actorId, payload, note?, status? }`
  - Erfolg: `{ status: "ok", credential }`
- `POST /admin/remove` (nur mit Maintainer-Secret)
  - Anfrage: `{ credentialId, actorId }`
  - Erfolg: `{ status: "ok", changed, credential }`
  - Schutz bei aktivem Lease: `{ status: "error", code: "LEASE_ACTIVE", ... }`
- `POST /admin/list` (nur mit Maintainer-Secret)
  - Anfrage: `{ kind?, status?, includePayload?, limit? }`
  - Erfolg: `{ status: "ok", credentials, count }`

Payload-Form für die Art Telegram:

- `{ groupId: string, driverToken: string, sutToken: string }`
- `groupId` muss eine numerische Telegram-Chat-ID als String sein.
- `admin/add` validiert diese Form für `kind: "telegram"` und weist fehlerhafte Payloads zurück.

### Einen Kanal zu QA hinzufügen

Das Hinzufügen eines Kanals zum Markdown-QA-System erfordert genau zwei Dinge:

1. Einen Transport-Adapter für den Kanal.
2. Ein Szenario-Paket, das den Kanalvertrag testet.

Füge keinen neuen Root-Befehl der obersten Ebene für QA hinzu, wenn der gemeinsame `qa-lab`-Host den Ablauf übernehmen kann.

`qa-lab` ist für die gemeinsamen Host-Mechaniken zuständig:

- den Root-Befehl `openclaw qa`
- Start und Beenden der Suite
- Worker-Konkurrenz
- Schreiben von Artefakten
- Berichtserstellung
- Szenarioausführung
- Kompatibilitäts-Aliasse für ältere `qa-channel`-Szenarien

Runner-Plugins sind für den Transportvertrag zuständig:

- wie `openclaw qa <runner>` unter dem gemeinsamen Root `qa` eingehängt wird
- wie das Gateway für diesen Transport konfiguriert wird
- wie die Bereitschaft geprüft wird
- wie eingehende Ereignisse injiziert werden
- wie ausgehende Nachrichten beobachtet werden
- wie Transkripte und normalisierter Transportzustand bereitgestellt werden
- wie transportgestützte Aktionen ausgeführt werden
- wie transport-spezifisches Zurücksetzen oder Aufräumen behandelt wird

Die Mindestanforderung für die Übernahme eines neuen Kanals ist:

1. Behalte `qa-lab` als Besitzer des gemeinsamen Roots `qa`.
2. Implementiere den Transport-Runner auf der gemeinsamen Host-Seam von `qa-lab`.
3. Behalte transport-spezifische Mechaniken im Runner-Plugin oder Kanal-Harness.
4. Hänge den Runner als `openclaw qa <runner>` ein, statt einen konkurrierenden Root-Befehl zu registrieren.
   Runner-Plugins sollten `qaRunners` in `openclaw.plugin.json` deklarieren und ein passendes Array `qaRunnerCliRegistrations` aus `runtime-api.ts` exportieren.
   Halte `runtime-api.ts` schlank; Lazy-CLI- und Runner-Ausführung sollten hinter separaten Entry-Points bleiben.
5. Erstelle oder passe Markdown-Szenarien unter `qa/scenarios/` an.
6. Verwende die generischen Szenario-Hilfsfunktionen für neue Szenarien.
7. Sorge dafür, dass bestehende Kompatibilitäts-Aliasse weiter funktionieren, außer das Repo führt bewusst eine Migration durch.

Die Entscheidungsregel ist strikt:

- Wenn Verhalten einmalig in `qa-lab` ausgedrückt werden kann, platziere es in `qa-lab`.
- Wenn Verhalten von einem Kanaltransport abhängt, behalte es in diesem Runner-Plugin oder Plugin-Harness.
- Wenn ein Szenario eine neue Fähigkeit benötigt, die mehr als ein Kanal verwenden kann, füge eine generische Hilfsfunktion hinzu, statt einen kanal-spezifischen Branch in `suite.ts`.
- Wenn ein Verhalten nur für einen Transport sinnvoll ist, halte das Szenario transport-spezifisch und mache das im Szenariovertrag explizit.

Bevorzugte Namen generischer Hilfsfunktionen für neue Szenarien sind:

- `waitForTransportReady`
- `waitForChannelReady`
- `injectInboundMessage`
- `injectOutboundMessage`
- `waitForTransportOutboundMessage`
- `waitForChannelOutboundMessage`
- `waitForNoTransportOutbound`
- `getTransportSnapshot`
- `readTransportMessage`
- `readTransportTranscript`
- `formatTransportTranscript`
- `resetTransport`

Kompatibilitäts-Aliasse bleiben für bestehende Szenarien verfügbar, darunter:

- `waitForQaChannelReady`
- `waitForOutboundMessage`
- `waitForNoOutbound`
- `formatConversationTranscript`
- `resetBus`

Neue Kanal-Arbeit sollte die generischen Namen der Hilfsfunktionen verwenden.
Kompatibilitäts-Aliasse existieren, um eine Migration mit Stichtag zu vermeiden, nicht als Modell für das Verfassen neuer Szenarien.

## Test-Suites (was wo ausgeführt wird)

Stelle dir die Suites als „zunehmenden Realismus“ vor (und zunehmende Flakiness/Kosten):

### Unit / Integration (Standard)

- Befehl: `pnpm test`
- Konfiguration: zehn sequentielle Shard-Läufe (`vitest.full-*.config.ts`) über die bestehenden bereichsspezifischen Vitest-Projekte
- Dateien: Core-/Unit-Inventare unter `src/**/*.test.ts`, `packages/**/*.test.ts`, `test/**/*.test.ts` und die per Allowlist freigegebenen `ui`-Node-Tests, die von `vitest.unit.config.ts` abgedeckt werden
- Umfang:
  - Reine Unit-Tests
  - In-Process-Integrationstests (Gateway-Authentifizierung, Routing, Tooling, Parsing, Konfiguration)
  - Deterministische Regressionstests für bekannte Fehler
- Erwartungen:
  - Läuft in CI
  - Keine echten Schlüssel erforderlich
  - Sollte schnell und stabil sein
- Hinweis zu Projekten:
  - Nicht gezieltes `pnpm test` führt jetzt elf kleinere Shard-Konfigurationen aus (`core-unit-src`, `core-unit-security`, `core-unit-ui`, `core-unit-support`, `core-support-boundary`, `core-contracts`, `core-bundled`, `core-runtime`, `agentic`, `auto-reply`, `extensions`) statt eines riesigen nativen Root-Projekt-Prozesses. Das reduziert die Spitzen-RSS auf ausgelasteten Maschinen und verhindert, dass `auto-reply`-/Erweiterungs-Arbeit nicht zusammenhängende Suites ausbremst.
  - `pnpm test --watch` verwendet weiterhin den nativen Root-`vitest.config.ts`-Projektgraphen, weil eine Watch-Schleife mit mehreren Shards nicht praktikabel ist.
  - `pnpm test`, `pnpm test:watch` und `pnpm test:perf:imports` leiten explizite Datei-/Verzeichnis-Targets jetzt zuerst durch bereichsspezifische Lanes, sodass `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts` nicht den gesamten Startaufwand des Root-Projekts zahlen muss.
  - `pnpm test:changed` erweitert geänderte Git-Pfade zu denselben bereichsspezifischen Lanes, wenn der Diff nur routbare Quell-/Testdateien berührt; Konfigurations-/Setup-Änderungen fallen weiterhin auf den breiten erneuten Lauf des Root-Projekts zurück.
  - Import-leichte Unit-Tests aus Agents, Commands, Plugins, `auto-reply`-Hilfsfunktionen, `plugin-sdk` und ähnlichen rein utilitären Bereichen werden über die `unit-fast`-Lane geleitet, die `test/setup-openclaw-runtime.ts` überspringt; zustandsbehaftete/runtime-schwere Dateien bleiben auf den bestehenden Lanes.
  - Ausgewählte `plugin-sdk`- und `commands`-Hilfsquellendateien ordnen Changed-Mode-Läufe ebenfalls expliziten benachbarten Tests in diesen leichten Lanes zu, sodass Hilfsänderungen nicht die vollständige schwere Suite für dieses Verzeichnis erneut ausführen.
  - `auto-reply` hat jetzt drei dedizierte Buckets: Core-Hilfsfunktionen auf oberster Ebene, `reply.*`-Integrationstests auf oberster Ebene und den Teilbaum `src/auto-reply/reply/**`. So bleibt die schwerste Reply-Harness-Arbeit von den günstigen Status-/Chunk-/Token-Tests getrennt.
- Hinweis zum eingebetteten Runner:
  - Wenn du Discovery-Eingaben für Message-Tools oder den Laufzeitkontext von Compaction änderst,
    behalte beide Ebenen der Abdeckung bei.
  - Füge fokussierte Hilfs-Regressionstests für reine Routing-/Normalisierungsgrenzen hinzu.
  - Halte außerdem die eingebetteten Runner-Integrations-Suites gesund:
    `src/agents/pi-embedded-runner/compact.hooks.test.ts`,
    `src/agents/pi-embedded-runner/run.overflow-compaction.test.ts` und
    `src/agents/pi-embedded-runner/run.overflow-compaction.loop.test.ts`.
  - Diese Suites prüfen, dass bereichsspezifische IDs und Compaction-Verhalten weiterhin
    durch die echten Pfade `run.ts` / `compact.ts` fließen; reine Hilfstests sind kein
    ausreichender Ersatz für diese Integrationspfade.
- Hinweis zum Pool:
  - Die Basis-Vitest-Konfiguration verwendet jetzt standardmäßig `threads`.
  - Die gemeinsame Vitest-Konfiguration setzt außerdem `isolate: false` fest und verwendet den nicht isolierten Runner für die Root-Projekte, E2E- und Live-Konfigurationen.
  - Die Root-UI-Lane behält ihr `jsdom`-Setup und ihren Optimizer, läuft jetzt aber ebenfalls auf dem gemeinsamen nicht isolierten Runner.
  - Jeder `pnpm test`-Shard erbt dieselben Standardwerte `threads` + `isolate: false` aus der gemeinsamen Vitest-Konfiguration.
  - Der gemeinsame Launcher `scripts/run-vitest.mjs` fügt jetzt standardmäßig auch `--no-maglev` für Vitest-Child-Node-Prozesse hinzu, um V8-Kompilierungs-Churn bei großen lokalen Läufen zu reduzieren. Setze `OPENCLAW_VITEST_ENABLE_MAGLEV=1`, wenn du mit dem Standardverhalten von V8 vergleichen musst.
- Hinweis zur schnellen lokalen Iteration:
  - `pnpm test:changed` wird über bereichsspezifische Lanes geroutet, wenn die geänderten Pfade sauber einer kleineren Suite zugeordnet werden können.
  - `pnpm test:max` und `pnpm test:changed:max` behalten dasselbe Routing-Verhalten bei, nur mit einer höheren Worker-Obergrenze.
  - Die automatische lokale Worker-Skalierung ist jetzt absichtlich konservativ und fährt ebenfalls zurück, wenn die Lastdurchschnittswerte des Hosts bereits hoch sind, sodass mehrere gleichzeitige Vitest-Läufe standardmäßig weniger Schaden anrichten.
  - Die Basis-Vitest-Konfiguration markiert die Projekte-/Konfigurationsdateien als `forceRerunTriggers`, damit erneute Läufe im Changed-Modus korrekt bleiben, wenn sich die Test-Verdrahtung ändert.
  - Die Konfiguration lässt `OPENCLAW_VITEST_FS_MODULE_CACHE` auf unterstützten Hosts aktiviert; setze `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/abs/path`, wenn du einen expliziten Cache-Speicherort für direktes Profiling möchtest.
- Hinweis zum Performance-Debugging:
  - `pnpm test:perf:imports` aktiviert die Berichterstattung zur Vitest-Importdauer sowie die Ausgabe der Import-Aufschlüsselung.
  - `pnpm test:perf:imports:changed` beschränkt dieselbe Profiling-Ansicht auf Dateien, die sich seit `origin/main` geändert haben.
- `pnpm test:perf:changed:bench -- --ref <git-ref>` vergleicht geroutetes `test:changed` mit dem nativen Root-Projekt-Pfad für diesen festgeschriebenen Diff und gibt Wall Time sowie das macOS-Maximum der RSS aus.
- `pnpm test:perf:changed:bench -- --worktree` benchmarkt den aktuellen Dirty Tree, indem die Liste geänderter Dateien durch `scripts/test-projects.mjs` und die Root-Vitest-Konfiguration geleitet wird.
  - `pnpm test:perf:profile:main` schreibt ein CPU-Profil des Main-Threads für den Start von Vitest/Vite und den Transformations-Overhead.
  - `pnpm test:perf:profile:runner` schreibt CPU-+Heap-Profile des Runners für die Unit-Suite bei deaktivierter Datei-Parallelität.

### E2E (Gateway-Smoke)

- Befehl: `pnpm test:e2e`
- Konfiguration: `vitest.e2e.config.ts`
- Dateien: `src/**/*.e2e.test.ts`, `test/**/*.e2e.test.ts`
- Laufzeit-Standards:
  - Verwendet Vitest-`threads` mit `isolate: false`, passend zum Rest des Repos.
  - Verwendet adaptive Worker (CI: bis zu 2, lokal: standardmäßig 1).
  - Läuft standardmäßig im stillen Modus, um den Overhead durch Konsolen-I/O zu reduzieren.
- Nützliche Overrides:
  - `OPENCLAW_E2E_WORKERS=<n>`, um die Anzahl der Worker zu erzwingen (begrenzt auf 16).
  - `OPENCLAW_E2E_VERBOSE=1`, um die ausführliche Konsolenausgabe wieder zu aktivieren.
- Umfang:
  - End-to-End-Verhalten von Multi-Instance-Gateways
  - WebSocket-/HTTP-Oberflächen, Node-Pairing und umfangreicheres Networking
- Erwartungen:
  - Läuft in CI (wenn in der Pipeline aktiviert)
  - Keine echten Schlüssel erforderlich
  - Mehr bewegliche Teile als Unit-Tests (kann langsamer sein)

### E2E: OpenShell-Backend-Smoke

- Befehl: `pnpm test:e2e:openshell`
- Datei: `test/openshell-sandbox.e2e.test.ts`
- Umfang:
  - Startet über Docker ein isoliertes OpenShell-Gateway auf dem Host
  - Erstellt eine Sandbox aus einer temporären lokalen Dockerfile
  - Testet OpenClaws OpenShell-Backend über echtes `sandbox ssh-config` + SSH-Exec
  - Verifiziert remote-kanonisches Dateisystemverhalten über die Sandbox-FS-Bridge
- Erwartungen:
  - Nur Opt-in; nicht Teil des standardmäßigen `pnpm test:e2e`-Laufs
  - Erfordert eine lokale `openshell`-CLI sowie einen funktionierenden Docker-Daemon
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
  - „Funktioniert dieser Provider/dieses Modell _heute_ tatsächlich mit echten Anmeldedaten?“
  - Erkennt Änderungen an Provider-Formaten, Eigenheiten bei Tool-Calling, Authentifizierungsprobleme und Rate-Limit-Verhalten
- Erwartungen:
  - Von Haus aus nicht CI-stabil (reale Netzwerke, reale Provider-Richtlinien, Quoten, Ausfälle)
  - Kostet Geld / verbraucht Rate Limits
  - Bevorzuge eingegrenzte Teilmengen statt „alles“
- Live-Läufe sourcen `~/.profile`, um fehlende API-Schlüssel aufzunehmen.
- Standardmäßig isolieren Live-Läufe weiterhin `HOME` und kopieren Konfigurations-/Auth-Material in ein temporäres Test-Home, damit Unit-Fixtures dein echtes `~/.openclaw` nicht verändern können.
- Setze `OPENCLAW_LIVE_USE_REAL_HOME=1` nur, wenn Live-Tests bewusst dein echtes Home-Verzeichnis verwenden sollen.
- `pnpm test:live` verwendet jetzt standardmäßig einen ruhigeren Modus: `[live] ...`-Fortschrittsausgaben bleiben sichtbar, aber der zusätzliche Hinweis zu `~/.profile` wird unterdrückt und Gateway-Bootstrap-Logs/Bonjour-Chatter werden stummgeschaltet. Setze `OPENCLAW_LIVE_TEST_QUIET=0`, wenn du die vollständigen Start-Logs wiederhaben möchtest.
- Rotation von API-Schlüsseln (providerspezifisch): setze `*_API_KEYS` im Komma-/Semikolon-Format oder `*_API_KEY_1`, `*_API_KEY_2` (zum Beispiel `OPENAI_API_KEYS`, `ANTHROPIC_API_KEYS`, `GEMINI_API_KEYS`) oder pro Live-Override über `OPENCLAW_LIVE_*_KEY`; Tests versuchen bei Rate-Limit-Antworten einen erneuten Lauf.
- Fortschritts-/Heartbeat-Ausgabe:
  - Live-Suites geben jetzt Fortschrittszeilen auf stderr aus, sodass bei langen Provider-Aufrufen sichtbar bleibt, dass Aktivität stattfindet, selbst wenn die Konsolenerfassung von Vitest ruhig ist.
  - `vitest.live.config.ts` deaktiviert das Abfangen der Konsole durch Vitest, sodass Fortschrittszeilen von Provider/Gateway während Live-Läufen sofort gestreamt werden.
  - Passe direkte Modell-Heartbeats mit `OPENCLAW_LIVE_HEARTBEAT_MS` an.
  - Passe Gateway-/Probe-Heartbeats mit `OPENCLAW_LIVE_GATEWAY_HEARTBEAT_MS` an.

## Welche Suite sollte ich ausführen?

Verwende diese Entscheidungstabelle:

- Logik/Tests bearbeiten: `pnpm test` ausführen (und `pnpm test:coverage`, wenn du viel geändert hast)
- Gateway-Networking / WS-Protokoll / Pairing berühren: zusätzlich `pnpm test:e2e` ausführen
- „Mein Bot ist down“ / providerspezifische Fehler / Tool-Calling debuggen: eingegrenztes `pnpm test:live` ausführen

## Live: Android-Node-Fähigkeiten-Sweep

- Test: `src/gateway/android-node.capabilities.live.test.ts`
- Skript: `pnpm android:test:integration`
- Ziel: **jeden aktuell beworbenen Befehl** eines verbundenen Android-Node aufrufen und das Vertragsverhalten des Befehls prüfen.
- Umfang:
  - Vorgegebene/manuelle Einrichtung (die Suite installiert/startet/paired die App nicht).
  - Gateway-`node.invoke`-Validierung Befehl für Befehl für den ausgewählten Android-Node.
- Erforderliche Voreinrichtung:
  - Android-App ist bereits verbunden und mit dem Gateway gepairt.
  - App bleibt im Vordergrund.
  - Berechtigungen/Aufnahmezustimmung wurden für die Fähigkeiten erteilt, bei denen du Erfolg erwartest.
- Optionale Ziel-Overrides:
  - `OPENCLAW_ANDROID_NODE_ID` oder `OPENCLAW_ANDROID_NODE_NAME`.
  - `OPENCLAW_ANDROID_GATEWAY_URL` / `OPENCLAW_ANDROID_GATEWAY_TOKEN` / `OPENCLAW_ANDROID_GATEWAY_PASSWORD`.
- Vollständige Android-Setup-Details: [Android-App](/de/platforms/android)

## Live: Modell-Smoke (Profilschlüssel)

Live-Tests sind in zwei Ebenen aufgeteilt, damit wir Fehler isolieren können:

- „Direktes Modell“ zeigt uns, dass der Provider/das Modell mit dem angegebenen Schlüssel überhaupt antworten kann.
- „Gateway-Smoke“ zeigt uns, dass die vollständige Gateway-+Agent-Pipeline für dieses Modell funktioniert (Sitzungen, Verlauf, Tools, Sandbox-Richtlinie usw.).

### Ebene 1: Direkte Modell-Completion (ohne Gateway)

- Test: `src/agents/models.profiles.live.test.ts`
- Ziel:
  - Erkannte Modelle aufzählen
  - `getApiKeyForModel` verwenden, um Modelle auszuwählen, für die du Anmeldedaten hast
  - Pro Modell eine kleine Completion ausführen (und gezielte Regressionstests, wo nötig)
- Aktivierung:
  - `pnpm test:live` (oder `OPENCLAW_LIVE_TEST=1`, wenn Vitest direkt aufgerufen wird)
- Setze `OPENCLAW_LIVE_MODELS=modern` (oder `all`, Alias für modern), um diese Suite tatsächlich auszuführen; andernfalls wird sie übersprungen, damit `pnpm test:live` auf Gateway-Smoke fokussiert bleibt
- Modellauswahl:
  - `OPENCLAW_LIVE_MODELS=modern`, um die moderne Allowlist auszuführen (Opus/Sonnet 4.6+, GPT-5.x + Codex, Gemini 3, GLM 4.7, MiniMax M2.7, Grok 4)
  - `OPENCLAW_LIVE_MODELS=all` ist ein Alias für die moderne Allowlist
  - oder `OPENCLAW_LIVE_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,..."` (Komma-Allowlist)
  - Moderne/All-Sweeps verwenden standardmäßig eine kuratierte Obergrenze mit hohem Signal; setze `OPENCLAW_LIVE_MAX_MODELS=0` für einen vollständigen modernen Sweep oder eine positive Zahl für eine kleinere Obergrenze.
- Providerauswahl:
  - `OPENCLAW_LIVE_PROVIDERS="google,google-antigravity,google-gemini-cli"` (Komma-Allowlist)
- Woher die Schlüssel kommen:
  - Standardmäßig: Profilspeicher und env-Fallbacks
  - Setze `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`, um **nur** den Profilspeicher zu erzwingen
- Warum das existiert:
  - Trennt „Provider-API ist kaputt / Schlüssel ist ungültig“ von „Gateway-Agent-Pipeline ist kaputt“
  - Enthält kleine, isolierte Regressionstests (Beispiel: OpenAI-Responses/Codex-Responses-Reasoning-Replay + Tool-Call-Flows)

### Ebene 2: Gateway + Dev-Agent-Smoke (was „@openclaw“ tatsächlich macht)

- Test: `src/gateway/gateway-models.profiles.live.test.ts`
- Ziel:
  - Ein In-Process-Gateway starten
  - Eine `agent:dev:*`-Sitzung erstellen/patchen (Modell-Override pro Lauf)
  - Modelle mit Schlüsseln durchlaufen und Folgendes prüfen:
    - „aussagekräftige“ Antwort (ohne Tools)
    - eine echte Tool-Invocation funktioniert (Read-Probe)
    - optionale zusätzliche Tool-Probes (Exec+Read-Probe)
    - OpenAI-Regressionspfade (nur Tool-Call → Follow-up) funktionieren weiterhin
- Probe-Details (damit du Fehler schnell erklären kannst):
  - `read`-Probe: Der Test schreibt eine Nonce-Datei in den Workspace und fordert den Agent auf, sie mit `read` zu lesen und die Nonce zurückzugeben.
  - `exec+read`-Probe: Der Test fordert den Agent auf, mit `exec` eine Nonce in eine temporäre Datei zu schreiben und sie dann mit `read` wieder zu lesen.
  - Image-Probe: Der Test hängt ein erzeugtes PNG an (Katze + zufälliger Code) und erwartet, dass das Modell `cat <CODE>` zurückgibt.
  - Implementierungsreferenz: `src/gateway/gateway-models.profiles.live.test.ts` und `src/gateway/live-image-probe.ts`.
- Aktivierung:
  - `pnpm test:live` (oder `OPENCLAW_LIVE_TEST=1`, wenn Vitest direkt aufgerufen wird)
- Modellauswahl:
  - Standard: moderne Allowlist (Opus/Sonnet 4.6+, GPT-5.x + Codex, Gemini 3, GLM 4.7, MiniMax M2.7, Grok 4)
  - `OPENCLAW_LIVE_GATEWAY_MODELS=all` ist ein Alias für die moderne Allowlist
  - Oder `OPENCLAW_LIVE_GATEWAY_MODELS="provider/model"` (oder Komma-Liste) setzen, um einzugrenzen
  - Moderne/All-Gateway-Sweeps verwenden standardmäßig eine kuratierte Obergrenze mit hohem Signal; setze `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=0` für einen vollständigen modernen Sweep oder eine positive Zahl für eine kleinere Obergrenze.
- Providerauswahl (vermeide „alles über OpenRouter“):
  - `OPENCLAW_LIVE_GATEWAY_PROVIDERS="google,google-antigravity,google-gemini-cli,openai,anthropic,zai,minimax"` (Komma-Allowlist)
- Tool- + Image-Probes sind in diesem Live-Test immer aktiviert:
  - `read`-Probe + `exec+read`-Probe (Tool-Stresstest)
  - Image-Probe läuft, wenn das Modell Unterstützung für Bildeingaben bewirbt
  - Ablauf (allgemein):
    - Der Test erzeugt ein kleines PNG mit „CAT“ + zufälligem Code (`src/gateway/live-image-probe.ts`)
    - Sendet es über `agent` mit `attachments: [{ mimeType: "image/png", content: "<base64>" }]`
    - Das Gateway parst Anhänge in `images[]` (`src/gateway/server-methods/agent.ts` + `src/gateway/chat-attachments.ts`)
    - Der eingebettete Agent leitet eine multimodale Benutzernachricht an das Modell weiter
    - Prüfung: Die Antwort enthält `cat` + den Code (OCR-Toleranz: kleinere Fehler sind erlaubt)

Tipp: Um zu sehen, was du auf deiner Maschine testen kannst (und die genauen `provider/model`-IDs), führe Folgendes aus:

```bash
openclaw models list
openclaw models list --json
```

## Live: CLI-Backend-Smoke (Claude, Codex, Gemini oder andere lokale CLIs)

- Test: `src/gateway/gateway-cli-backend.live.test.ts`
- Ziel: die Gateway-+Agent-Pipeline mit einem lokalen CLI-Backend validieren, ohne deine Standardkonfiguration anzufassen.
- Backend-spezifische Smoke-Standards liegen in der Definition `cli-backend.ts` der besitzenden Erweiterung.
- Aktivierung:
  - `pnpm test:live` (oder `OPENCLAW_LIVE_TEST=1`, wenn Vitest direkt aufgerufen wird)
  - `OPENCLAW_LIVE_CLI_BACKEND=1`
- Standards:
  - Standard-Provider/-Modell: `claude-cli/claude-sonnet-4-6`
  - Verhalten von Command/Args/Image kommt aus den Metadaten des besitzenden CLI-Backend-Plugins.
- Overrides (optional):
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL="codex-cli/gpt-5.4"`
  - `OPENCLAW_LIVE_CLI_BACKEND_COMMAND="/full/path/to/codex"`
  - `OPENCLAW_LIVE_CLI_BACKEND_ARGS='["exec","--json","--color","never","--sandbox","read-only","--skip-git-repo-check"]'`
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_PROBE=1`, um einen echten Bildanhang zu senden (Pfade werden in den Prompt injiziert).
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_ARG="--image"`, um Bilddateipfade als CLI-Argumente statt per Prompt-Injektion zu übergeben.
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_MODE="repeat"` (oder `"list"`), um zu steuern, wie Bildargumente übergeben werden, wenn `IMAGE_ARG` gesetzt ist.
  - `OPENCLAW_LIVE_CLI_BACKEND_RESUME_PROBE=1`, um einen zweiten Turn zu senden und den Resume-Flow zu validieren.
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL_SWITCH_PROBE=0`, um die standardmäßige Kontinuitätsprobe Claude Sonnet -> Opus in derselben Sitzung zu deaktivieren (auf `1` setzen, um sie zu erzwingen, wenn das ausgewählte Modell ein Switch-Ziel unterstützt).

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
pnpm test:docker:live-cli-backend:claude-subscription
pnpm test:docker:live-cli-backend:codex
pnpm test:docker:live-cli-backend:gemini
```

Hinweise:

- Der Docker-Runner liegt unter `scripts/test-live-cli-backend-docker.sh`.
- Er führt den Live-CLI-Backend-Smoke innerhalb des Repo-Docker-Images als Nicht-Root-Benutzer `node` aus.
- Er löst CLI-Smoke-Metadaten aus der besitzenden Erweiterung auf und installiert dann das passende Linux-CLI-Paket (`@anthropic-ai/claude-code`, `@openai/codex` oder `@google/gemini-cli`) in ein zwischengespeichertes beschreibbares Präfix unter `OPENCLAW_DOCKER_CLI_TOOLS_DIR` (Standard: `~/.cache/openclaw/docker-cli-tools`).
- `pnpm test:docker:live-cli-backend:claude-subscription` erfordert portables Claude-Code-Subscription-OAuth entweder über `~/.claude/.credentials.json` mit `claudeAiOauth.subscriptionType` oder `CLAUDE_CODE_OAUTH_TOKEN` aus `claude setup-token`. Zuerst wird direktes `claude -p` in Docker nachgewiesen, dann werden zwei Gateway-CLI-Backend-Turns ausgeführt, ohne Anthropic-API-Key-env-Variablen beizubehalten. Diese Subscription-Lane deaktiviert standardmäßig die Claude-MCP-/Tool- und Image-Probes, weil Claude die Nutzung durch Drittanbieter-Apps derzeit über Extra-Usage-Abrechnung statt über normale Grenzen des Subscription-Plans leitet.
- Der Live-CLI-Backend-Smoke testet jetzt denselben End-to-End-Flow für Claude, Codex und Gemini: Text-Turn, Bildklassifizierungs-Turn, dann MCP-`cron`-Tool-Call, verifiziert über die Gateway-CLI.
- Claudes standardmäßiger Smoke patcht außerdem die Sitzung von Sonnet auf Opus und prüft, dass sich die fortgesetzte Sitzung weiterhin eine frühere Notiz merkt.

## Live: ACP-Bind-Smoke (`/acp spawn ... --bind here`)

- Test: `src/gateway/gateway-acp-bind.live.test.ts`
- Ziel: den echten Conversation-Bind-Flow von ACP mit einem Live-ACP-Agent validieren:
  - `/acp spawn <agent> --bind here` senden
  - eine synthetische Message-Channel-Konversation direkt vor Ort binden
  - einen normalen Follow-up auf derselben Konversation senden
  - prüfen, dass der Follow-up im Transkript der gebundenen ACP-Sitzung landet
- Aktivierung:
  - `pnpm test:live src/gateway/gateway-acp-bind.live.test.ts`
  - `OPENCLAW_LIVE_ACP_BIND=1`
- Standards:
  - ACP-Agents in Docker: `claude,codex,gemini`
  - ACP-Agent für direktes `pnpm test:live ...`: `claude`
  - Synthetischer Kanal: Slack-DM-artiger Konversationskontext
  - ACP-Backend: `acpx`
- Overrides:
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=claude`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=codex`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude,codex,gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND='npx -y @agentclientprotocol/claude-agent-acp@<version>'`
- Hinweise:
  - Diese Lane verwendet die Gateway-Oberfläche `chat.send` mit rein administrativen synthetischen Feldern für die Herkunftsroute, damit Tests Message-Channel-Kontext anhängen können, ohne vorzugeben, extern zuzustellen.
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

Docker-Rezepte für einzelne Agents:

```bash
pnpm test:docker:live-acp-bind:claude
pnpm test:docker:live-acp-bind:codex
pnpm test:docker:live-acp-bind:gemini
```

Docker-Hinweise:

- Der Docker-Runner liegt unter `scripts/test-live-acp-bind-docker.sh`.
- Standardmäßig führt er den ACP-Bind-Smoke nacheinander gegen alle unterstützten Live-CLI-Agents aus: `claude`, `codex`, dann `gemini`.
- Verwende `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude`, `OPENCLAW_LIVE_ACP_BIND_AGENTS=codex` oder `OPENCLAW_LIVE_ACP_BIND_AGENTS=gemini`, um die Matrix einzugrenzen.
- Er sourct `~/.profile`, staged das passende CLI-Auth-Material in den Container, installiert `acpx` in ein beschreibbares npm-Präfix und installiert dann bei Bedarf die angeforderte Live-CLI (`@anthropic-ai/claude-code`, `@openai/codex` oder `@google/gemini-cli`).
- Innerhalb von Docker setzt der Runner `OPENCLAW_LIVE_ACP_BIND_ACPX_COMMAND=$HOME/.npm-global/bin/acpx`, damit `acpx` Provider-env-Variablen aus dem gesourcten Profil für die Child-Harness-CLI verfügbar hält.

## Live: Codex-App-Server-Harness-Smoke

- Ziel: das Plugin-eigene Codex-Harness über die normale Gateway-
  `agent`-Methode validieren:
  - das gebündelte `codex`-Plugin laden
  - `OPENCLAW_AGENT_RUNTIME=codex` auswählen
  - einen ersten Gateway-Agent-Turn an `codex/gpt-5.4` senden
  - einen zweiten Turn an dieselbe OpenClaw-Sitzung senden und prüfen, dass der App-Server-
    Thread fortgesetzt werden kann
  - `/codex status` und `/codex models` über denselben Gateway-Command-
    Pfad ausführen
- Test: `src/gateway/gateway-codex-harness.live.test.ts`
- Aktivierung: `OPENCLAW_LIVE_CODEX_HARNESS=1`
- Standardmodell: `codex/gpt-5.4`
- Optionale Image-Probe: `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=1`
- Optionale MCP-/Tool-Probe: `OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=1`
- Der Smoke setzt `OPENCLAW_AGENT_HARNESS_FALLBACK=none`, damit ein defektes Codex-
  Harness nicht bestehen kann, indem es stillschweigend auf PI zurückfällt.
- Auth: `OPENAI_API_KEY` aus der Shell/dem Profil sowie optional kopierte
  `~/.codex/auth.json` und `~/.codex/config.toml`

Lokales Rezept:

```bash
source ~/.profile
OPENCLAW_LIVE_CODEX_HARNESS=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_MODEL=codex/gpt-5.4 \
  pnpm test:live -- src/gateway/gateway-codex-harness.live.test.ts
```

Docker-Rezept:

```bash
source ~/.profile
pnpm test:docker:live-codex-harness
```

Docker-Hinweise:

- Der Docker-Runner liegt unter `scripts/test-live-codex-harness-docker.sh`.
- Er sourct das eingehängte `~/.profile`, übergibt `OPENAI_API_KEY`, kopiert Codex-CLI-
  Auth-Dateien, wenn vorhanden, installiert `@openai/codex` in ein beschreibbares eingehängtes npm-
  Präfix, staged den Quellbaum und führt dann nur den Live-Test des Codex-Harness aus.
- Docker aktiviert standardmäßig die Image- und MCP-/Tool-Probes. Setze
  `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=0` oder
  `OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=0`, wenn du einen engeren Debug-Lauf brauchst.
- Docker exportiert außerdem `OPENCLAW_AGENT_HARNESS_FALLBACK=none`, passend zur Live-
  Testkonfiguration, sodass `openai-codex/*`- oder PI-Fallback einen Codex-Harness-
  Regressionsfehler nicht verbergen kann.

### Empfohlene Live-Rezepte

Enge, explizite Allowlists sind am schnellsten und am wenigsten flaky:

- Einzelnes Modell, direkt (ohne Gateway):
  - `OPENCLAW_LIVE_MODELS="openai/gpt-5.4" pnpm test:live src/agents/models.profiles.live.test.ts`

- Einzelnes Modell, Gateway-Smoke:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Tool-Calling über mehrere Provider hinweg:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3-flash-preview,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Fokus auf Google (Gemini-API-Key + Antigravity):
  - Gemini (API-Key): `OPENCLAW_LIVE_GATEWAY_MODELS="google/gemini-3-flash-preview" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`
  - Antigravity (OAuth): `OPENCLAW_LIVE_GATEWAY_MODELS="google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-pro-high" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

Hinweise:

- `google/...` verwendet die Gemini-API (API-Key).
- `google-antigravity/...` verwendet die Antigravity-OAuth-Bridge (Agent-Endpoint im Stil von Cloud Code Assist).
- `google-gemini-cli/...` verwendet die lokale Gemini-CLI auf deiner Maschine (separate Authentifizierung + Eigenheiten beim Tooling).
- Gemini-API vs. Gemini-CLI:
  - API: OpenClaw ruft Googles gehostete Gemini-API über HTTP auf (API-Key / Profil-Authentifizierung); das ist, was die meisten Benutzer mit „Gemini“ meinen.
  - CLI: OpenClaw ruft ein lokales `gemini`-Binary über die Shell auf; es hat seine eigene Authentifizierung und kann sich anders verhalten (Streaming/Tool-Support/Versionsabweichungen).

## Live: Modellmatrix (was wir abdecken)

Es gibt keine feste „CI-Modellliste“ (Live ist Opt-in), aber dies sind die **empfohlenen** Modelle, die auf einer Entwickler-Maschine mit Schlüsseln regelmäßig abgedeckt werden sollten.

### Moderner Smoke-Satz (Tool-Calling + Bild)

Das ist der Lauf mit den „gängigen Modellen“, den wir funktionsfähig halten wollen:

- OpenAI (nicht Codex): `openai/gpt-5.4` (optional: `openai/gpt-5.4-mini`)
- OpenAI Codex: `openai-codex/gpt-5.4`
- Anthropic: `anthropic/claude-opus-4-6` (oder `anthropic/claude-sonnet-4-6`)
- Google (Gemini-API): `google/gemini-3.1-pro-preview` und `google/gemini-3-flash-preview` (ältere Gemini-2.x-Modelle vermeiden)
- Google (Antigravity): `google-antigravity/claude-opus-4-6-thinking` und `google-antigravity/gemini-3-flash`
- Z.AI (GLM): `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

Gateway-Smoke mit Tools + Bild ausführen:
`OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,openai-codex/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3.1-pro-preview,google/gemini-3-flash-preview,google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-flash,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

### Basislinie: Tool-Calling (Read + optional Exec)

Wähle mindestens eines pro Provider-Familie:

- OpenAI: `openai/gpt-5.4` (oder `openai/gpt-5.4-mini`)
- Anthropic: `anthropic/claude-opus-4-6` (oder `anthropic/claude-sonnet-4-6`)
- Google: `google/gemini-3-flash-preview` (oder `google/gemini-3.1-pro-preview`)
- Z.AI (GLM): `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

Optionale zusätzliche Abdeckung (nice to have):

- xAI: `xai/grok-4` (oder die neueste verfügbare Version)
- Mistral: `mistral/`… (wähle ein „tools“-fähiges Modell, das du aktiviert hast)
- Cerebras: `cerebras/`… (wenn du Zugriff hast)
- LM Studio: `lmstudio/`… (lokal; Tool-Calling hängt vom API-Modus ab)

### Vision: Bild senden (Anhang → multimodale Nachricht)

Nimm mindestens ein bildfähiges Modell in `OPENCLAW_LIVE_GATEWAY_MODELS` auf (Claude/Gemini/OpenAI-Bildvarianten usw.), um die Image-Probe zu testen.

### Aggregatoren / alternative Gateways

Wenn du aktivierte Schlüssel hast, unterstützen wir außerdem Tests über:

- OpenRouter: `openrouter/...` (Hunderte Modelle; verwende `openclaw models scan`, um Kandidaten mit Tool-+Bild-Fähigkeit zu finden)
- OpenCode: `opencode/...` für Zen und `opencode-go/...` für Go (Authentifizierung über `OPENCODE_API_KEY` / `OPENCODE_ZEN_API_KEY`)

Weitere Provider, die du in die Live-Matrix aufnehmen kannst (wenn du Anmeldedaten/Konfiguration hast):

- Integriert: `openai`, `openai-codex`, `anthropic`, `google`, `google-vertex`, `google-antigravity`, `google-gemini-cli`, `zai`, `openrouter`, `opencode`, `opencode-go`, `xai`, `groq`, `cerebras`, `mistral`, `github-copilot`
- Über `models.providers` (benutzerdefinierte Endpoints): `minimax` (Cloud/API) sowie jeder OpenAI-/Anthropic-kompatible Proxy (LM Studio, vLLM, LiteLLM usw.)

Tipp: Versuche nicht, „alle Modelle“ in den Docs fest zu codieren. Die maßgebliche Liste ist das, was `discoverModels(...)` auf deiner Maschine zurückgibt + welche Schlüssel verfügbar sind.

## Anmeldedaten (niemals committen)

Live-Tests erkennen Anmeldedaten auf dieselbe Weise wie die CLI. Praktische Auswirkungen:

- Wenn die CLI funktioniert, sollten Live-Tests dieselben Schlüssel finden.
- Wenn ein Live-Test „keine Anmeldedaten“ meldet, debugge auf dieselbe Weise wie bei `openclaw models list` / Modellauswahl.

- Auth-Profile pro Agent: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` (das ist mit „Profilschlüssel“ in den Live-Tests gemeint)
- Konfiguration: `~/.openclaw/openclaw.json` (oder `OPENCLAW_CONFIG_PATH`)
- Legacy-Statusverzeichnis: `~/.openclaw/credentials/` (wird bei Vorhandensein in das gestagte Live-Home kopiert, ist aber nicht der Hauptspeicher für Profilschlüssel)
- Lokale Live-Läufe kopieren standardmäßig die aktive Konfiguration, `auth-profiles.json`-Dateien pro Agent, Legacy-`credentials/` und unterstützte externe CLI-Auth-Verzeichnisse in ein temporäres Test-Home; gestagte Live-Homes überspringen `workspace/` und `sandboxes/`, und Pfad-Overrides für `agents.*.workspace` / `agentDir` werden entfernt, damit Probes nicht in deinem echten Host-Workspace landen.

Wenn du dich auf env-Schlüssel verlassen möchtest (z. B. aus deinem `~/.profile` exportiert), führe lokale Tests nach `source ~/.profile` aus oder verwende die Docker-Runner unten (sie können `~/.profile` in den Container einhängen).

## Deepgram Live (Audio-Transkription)

- Test: `src/media-understanding/providers/deepgram/audio.live.test.ts`
- Aktivierung: `DEEPGRAM_API_KEY=... DEEPGRAM_LIVE_TEST=1 pnpm test:live src/media-understanding/providers/deepgram/audio.live.test.ts`

## BytePlus Coding-Plan Live

- Test: `src/agents/byteplus.live.test.ts`
- Aktivierung: `BYTEPLUS_API_KEY=... BYTEPLUS_LIVE_TEST=1 pnpm test:live src/agents/byteplus.live.test.ts`
- Optionales Modell-Override: `BYTEPLUS_CODING_MODEL=ark-code-latest`

## ComfyUI-Workflow-Medien Live

- Test: `extensions/comfy/comfy.live.test.ts`
- Aktivierung: `OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts`
- Umfang:
  - Testet die gebündelten comfy-Pfade für Bild, Video und `music_generate`
  - Überspringt jede Fähigkeit, sofern `models.providers.comfy.<capability>` nicht konfiguriert ist
  - Nützlich nach Änderungen an comfy-Workflow-Submission, Polling, Downloads oder Plugin-Registrierung

## Bildgenerierung Live

- Test: `src/image-generation/runtime.live.test.ts`
- Befehl: `pnpm test:live src/image-generation/runtime.live.test.ts`
- Harness: `pnpm test:live:media image`
- Umfang:
  - Zählt jedes registrierte Plugin für Bildgenerierungs-Provider auf
  - Lädt fehlende Provider-env-Variablen vor dem Testen aus deiner Login-Shell (`~/.profile`)
  - Verwendet standardmäßig Live-/env-API-Schlüssel vor gespeicherten Auth-Profilen, damit veraltete Testschlüssel in `auth-profiles.json` echte Shell-Anmeldedaten nicht verdecken
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
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`, um Authentifizierung über den Profilspeicher zu erzwingen und reine env-Overrides zu ignorieren

## Musikgenerierung Live

- Test: `extensions/music-generation-providers.live.test.ts`
- Aktivierung: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts`
- Harness: `pnpm test:live:media music`
- Umfang:
  - Testet den gemeinsamen gebündelten Pfad für Musikgenerierungs-Provider
  - Deckt derzeit Google und MiniMax ab
  - Lädt Provider-env-Variablen vor dem Testen aus deiner Login-Shell (`~/.profile`)
  - Verwendet standardmäßig Live-/env-API-Schlüssel vor gespeicherten Auth-Profilen, damit veraltete Testschlüssel in `auth-profiles.json` echte Shell-Anmeldedaten nicht verdecken
  - Überspringt Provider ohne nutzbare Authentifizierung/Profil/Modell
  - Führt beide deklarierten Runtime-Modi aus, wenn verfügbar:
    - `generate` mit reiner Prompt-Eingabe
    - `edit`, wenn der Provider `capabilities.edit.enabled` deklariert
  - Aktuelle Abdeckung in der gemeinsamen Lane:
    - `google`: `generate`, `edit`
    - `minimax`: `generate`
    - `comfy`: separate Comfy-Live-Datei, nicht dieser gemeinsame Sweep
- Optionale Eingrenzung:
  - `OPENCLAW_LIVE_MUSIC_GENERATION_PROVIDERS="google,minimax"`
  - `OPENCLAW_LIVE_MUSIC_GENERATION_MODELS="google/lyria-3-clip-preview,minimax/music-2.5+"`
- Optionales Authentifizierungsverhalten:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`, um Authentifizierung über den Profilspeicher zu erzwingen und reine env-Overrides zu ignorieren

## Videogenerierung Live

- Test: `extensions/video-generation-providers.live.test.ts`
- Aktivierung: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/video-generation-providers.live.test.ts`
- Harness: `pnpm test:live:media video`
- Umfang:
  - Testet den gemeinsamen gebündelten Pfad für Videogenerierungs-Provider
  - Verwendet standardmäßig den release-sicheren Smoke-Pfad: Nicht-FAL-Provider, eine Text-zu-Video-Anfrage pro Provider, ein einsekündiger Lobster-Prompt und eine providerbezogene Obergrenze pro Operation aus `OPENCLAW_LIVE_VIDEO_GENERATION_TIMEOUT_MS` (standardmäßig `180000`)
  - Überspringt FAL standardmäßig, weil providerseitige Queue-Latenz die Release-Zeit dominieren kann; übergib `--video-providers fal` oder `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="fal"`, um ihn explizit auszuführen
  - Lädt Provider-env-Variablen vor dem Testen aus deiner Login-Shell (`~/.profile`)
  - Verwendet standardmäßig Live-/env-API-Schlüssel vor gespeicherten Auth-Profilen, damit veraltete Testschlüssel in `auth-profiles.json` echte Shell-Anmeldedaten nicht verdecken
  - Überspringt Provider ohne nutzbare Authentifizierung/Profil/Modell
  - Führt standardmäßig nur `generate` aus
  - Setze `OPENCLAW_LIVE_VIDEO_GENERATION_FULL_MODES=1`, um zusätzlich deklarierte Transformationsmodi auszuführen, wenn verfügbar:
    - `imageToVideo`, wenn der Provider `capabilities.imageToVideo.enabled` deklariert und das ausgewählte Provider-/Modellpaar im gemeinsamen Sweep buffer-gestützte lokale Bildeingabe akzeptiert
    - `videoToVideo`, wenn der Provider `capabilities.videoToVideo.enabled` deklariert und das ausgewählte Provider-/Modellpaar im gemeinsamen Sweep buffer-gestützte lokale Videoeingabe akzeptiert
  - Aktuell deklarierte, aber im gemeinsamen Sweep übersprungene `imageToVideo`-Provider:
    - `vydra`, weil das gebündelte `veo3` nur Text unterstützt und das gebündelte `kling` eine Remote-Bild-URL erfordert
  - Provider-spezifische Vydra-Abdeckung:
    - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_VYDRA_VIDEO=1 pnpm test:live -- extensions/vydra/vydra.live.test.ts`
    - diese Datei führt `veo3` Text-zu-Video sowie standardmäßig eine `kling`-Lane aus, die ein Fixture mit einer Remote-Bild-URL verwendet
  - Aktuelle `videoToVideo`-Live-Abdeckung:
    - nur `runway`, wenn das ausgewählte Modell `runway/gen4_aleph` ist
  - Aktuell deklarierte, aber im gemeinsamen Sweep übersprungene `videoToVideo`-Provider:
    - `alibaba`, `qwen`, `xai`, weil diese Pfade derzeit Referenz-URLs als Remote-`http(s)` / MP4 erfordern
    - `google`, weil die aktuelle gemeinsame Gemini-/Veo-Lane lokale buffer-gestützte Eingabe verwendet und dieser Pfad im gemeinsamen Sweep nicht akzeptiert wird
    - `openai`, weil der aktuelle gemeinsame Pfad keine Garantien für organisationsspezifischen Zugriff auf Video-Inpainting/Remix bietet
- Optionale Eingrenzung:
  - `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="google,openai,runway"`
  - `OPENCLAW_LIVE_VIDEO_GENERATION_MODELS="google/veo-3.1-fast-generate-preview,openai/sora-2,runway/gen4_aleph"`
  - `OPENCLAW_LIVE_VIDEO_GENERATION_SKIP_PROVIDERS=""`, um jeden Provider im Standardsweep einzuschließen, einschließlich FAL
  - `OPENCLAW_LIVE_VIDEO_GENERATION_TIMEOUT_MS=60000`, um die providerbezogene Obergrenze pro Operation für einen aggressiven Smoke-Lauf zu reduzieren
- Optionales Authentifizierungsverhalten:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`, um Authentifizierung über den Profilspeicher zu erzwingen und reine env-Overrides zu ignorieren

## Media-Live-Harness

- Befehl: `pnpm test:live:media`
- Zweck:
  - Führt die gemeinsamen Live-Suites für Bild, Musik und Video über einen repo-eigenen Entry-Point aus
  - Lädt fehlende Provider-env-Variablen automatisch aus `~/.profile`
  - Grenzt standardmäßig jede Suite automatisch auf Provider ein, die derzeit nutzbare Authentifizierung haben
  - Verwendet erneut `scripts/test-live.mjs`, sodass Heartbeat- und Quiet-Mode-Verhalten konsistent bleiben
- Beispiele:
  - `pnpm test:live:media`
  - `pnpm test:live:media image video --providers openai,google,minimax`
  - `pnpm test:live:media video --video-providers openai,runway --all-providers`
  - `pnpm test:live:media music --quiet`

## Docker-Runner (optionale „funktioniert unter Linux“-Prüfungen)

Diese Docker-Runner teilen sich in zwei Gruppen auf:

- Live-Modell-Runner: `test:docker:live-models` und `test:docker:live-gateway` führen nur ihre jeweils passende Live-Datei für Profilschlüssel innerhalb des Repo-Docker-Images aus (`src/agents/models.profiles.live.test.ts` und `src/gateway/gateway-models.profiles.live.test.ts`), wobei dein lokales Konfigurationsverzeichnis und dein Workspace eingehängt werden (und `~/.profile` gesourct wird, wenn es eingehängt ist). Die passenden lokalen Entry-Points sind `test:live:models-profiles` und `test:live:gateway-profiles`.
- Docker-Live-Runner verwenden standardmäßig eine kleinere Smoke-Obergrenze, damit ein vollständiger Docker-Sweep praktikabel bleibt:
  `test:docker:live-models` verwendet standardmäßig `OPENCLAW_LIVE_MAX_MODELS=12`, und
  `test:docker:live-gateway` verwendet standardmäßig `OPENCLAW_LIVE_GATEWAY_SMOKE=1`,
  `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=8`,
  `OPENCLAW_LIVE_GATEWAY_STEP_TIMEOUT_MS=45000` und
  `OPENCLAW_LIVE_GATEWAY_MODEL_TIMEOUT_MS=90000`. Überschreibe diese env-Variablen, wenn du
  ausdrücklich den größeren vollständigen Scan möchtest.
- `test:docker:all` baut das Live-Docker-Image einmal über `test:docker:live-build` und verwendet es dann für die beiden Docker-Live-Lanes erneut.
- Container-Smoke-Runner: `test:docker:openwebui`, `test:docker:onboard`, `test:docker:gateway-network`, `test:docker:mcp-channels` und `test:docker:plugins` starten einen oder mehrere echte Container und verifizieren Integrationspfade auf höherer Ebene.

Die Docker-Runner für Live-Modelle binden außerdem nur die benötigten CLI-Auth-Homes ein (oder alle unterstützten, wenn der Lauf nicht eingegrenzt ist) und kopieren sie dann vor dem Lauf in das Home-Verzeichnis des Containers, damit OAuth externer CLIs Tokens aktualisieren kann, ohne den Auth-Speicher des Hosts zu verändern:

- Direkte Modelle: `pnpm test:docker:live-models` (Skript: `scripts/test-live-models-docker.sh`)
- ACP-Bind-Smoke: `pnpm test:docker:live-acp-bind` (Skript: `scripts/test-live-acp-bind-docker.sh`)
- CLI-Backend-Smoke: `pnpm test:docker:live-cli-backend` (Skript: `scripts/test-live-cli-backend-docker.sh`)
- Codex-App-Server-Harness-Smoke: `pnpm test:docker:live-codex-harness` (Skript: `scripts/test-live-codex-harness-docker.sh`)
- Gateway + Dev-Agent: `pnpm test:docker:live-gateway` (Skript: `scripts/test-live-gateway-models-docker.sh`)
- Open-WebUI-Live-Smoke: `pnpm test:docker:openwebui` (Skript: `scripts/e2e/openwebui-docker.sh`)
- Onboarding-Assistent (TTY, vollständiges Scaffolding): `pnpm test:docker:onboard` (Skript: `scripts/e2e/onboard-docker.sh`)
- Gateway-Networking (zwei Container, WS-Auth + Health): `pnpm test:docker:gateway-network` (Skript: `scripts/e2e/gateway-network-docker.sh`)
- MCP-Kanal-Bridge (vorgefülltes Gateway + stdio-Bridge + roher Claude-Benachrichtigungsframe-Smoke): `pnpm test:docker:mcp-channels` (Skript: `scripts/e2e/mcp-channels-docker.sh`)
- Plugins (Installations-Smoke + `/plugin`-Alias + Neustartsemantik des Claude-Bundles): `pnpm test:docker:plugins` (Skript: `scripts/e2e/plugins-docker.sh`)

Die Docker-Runner für Live-Modelle binden außerdem den aktuellen Checkout schreibgeschützt ein und
stagen ihn in ein temporäres Workdir innerhalb des Containers. Dadurch bleibt das Runtime-
Image schlank, während Vitest trotzdem gegen genau deinen lokalen Quellcode/deine lokale Konfiguration ausgeführt wird.
Der Staging-Schritt überspringt große nur lokale Caches und App-Build-Ausgaben wie
`.pnpm-store`, `.worktrees`, `__openclaw_vitest__` und app-lokale `.build`- oder
Gradle-Ausgabeverzeichnisse, damit Docker-Live-Läufe nicht minutenlang
maschinenspezifische Artefakte kopieren.
Sie setzen außerdem `OPENCLAW_SKIP_CHANNELS=1`, damit Gateway-Live-Probes nicht
echte Kanal-Worker für Telegram/Discord usw. innerhalb des Containers starten.
`test:docker:live-models` führt weiterhin `pnpm test:live` aus, daher gib auch
`OPENCLAW_LIVE_GATEWAY_*` weiter, wenn du die Gateway-
Live-Abdeckung in dieser Docker-Lane eingrenzen oder ausschließen musst.
`test:docker:openwebui` ist ein höherstufiger Kompatibilitäts-Smoke: Er startet einen
OpenClaw-Gateway-Container mit aktivierten OpenAI-kompatiblen HTTP-Endpoints,
startet einen angehefteten Open-WebUI-Container gegen dieses Gateway, meldet sich über
Open WebUI an, prüft, dass `/api/models` `openclaw/default` bereitstellt, und sendet dann eine
echte Chat-Anfrage über den Proxy `/api/chat/completions` von Open WebUI.
Der erste Lauf kann deutlich langsamer sein, weil Docker möglicherweise das
Open-WebUI-Image ziehen muss und Open WebUI möglicherweise seine eigene Kaltstart-Einrichtung abschließen muss.
Diese Lane erwartet einen nutzbaren Live-Modell-Schlüssel, und `OPENCLAW_PROFILE_FILE`
(`~/.profile` standardmäßig) ist der primäre Weg, ihn in Docker-Läufen bereitzustellen.
Erfolgreiche Läufe geben eine kleine JSON-Payload aus wie `{ "ok": true, "model":
"openclaw/default", ... }`.
`test:docker:mcp-channels` ist absichtlich deterministisch und benötigt kein
echtes Telegram-, Discord- oder iMessage-Konto. Es startet ein vorgefülltes Gateway-
Container, startet einen zweiten Container, der `openclaw mcp serve` ausführt, und
prüft dann geroutete Konversationserkennung, Transkript-Lesezugriffe, Anhang-Metadaten,
Verhalten der Live-Ereigniswarteschlange, Routing ausgehender Sendungen und Benachrichtigungen
zu Kanal + Berechtigungen im Claude-Stil über die echte stdio-MCP-Bridge. Die Benachrichtigungsprüfung
untersucht die rohen stdio-MCP-Frames direkt, sodass der Smoke das validiert, was die
Bridge tatsächlich ausgibt, nicht nur das, was ein bestimmtes Client-SDK zufällig bereitstellt.

Manueller ACP-Smoke für Threads in natürlicher Sprache (nicht CI):

- `bun scripts/dev/discord-acp-plain-language-smoke.ts --channel <discord-channel-id> ...`
- Behalte dieses Skript für Regressions-/Debug-Workflows. Es könnte für die Validierung des ACP-Thread-Routings erneut benötigt werden, also nicht löschen.

Nützliche env-Variablen:

- `OPENCLAW_CONFIG_DIR=...` (Standard: `~/.openclaw`), eingehängt nach `/home/node/.openclaw`
- `OPENCLAW_WORKSPACE_DIR=...` (Standard: `~/.openclaw/workspace`), eingehängt nach `/home/node/.openclaw/workspace`
- `OPENCLAW_PROFILE_FILE=...` (Standard: `~/.profile`), eingehängt nach `/home/node/.profile` und vor dem Ausführen der Tests gesourct
- `OPENCLAW_DOCKER_CLI_TOOLS_DIR=...` (Standard: `~/.cache/openclaw/docker-cli-tools`), eingehängt nach `/home/node/.npm-global` für zwischengespeicherte CLI-Installationen innerhalb von Docker
- Externe CLI-Auth-Verzeichnisse/-Dateien unter `$HOME` werden schreibgeschützt unter `/host-auth...` eingehängt und dann vor dem Start der Tests nach `/home/node/...` kopiert
  - Standardverzeichnisse: `.minimax`
  - Standarddateien: `~/.codex/auth.json`, `~/.codex/config.toml`, `.claude.json`, `~/.claude/.credentials.json`, `~/.claude/settings.json`, `~/.claude/settings.local.json`
  - Eingegrenzte Provider-Läufe hängen nur die benötigten Verzeichnisse/Dateien ein, die aus `OPENCLAW_LIVE_PROVIDERS` / `OPENCLAW_LIVE_GATEWAY_PROVIDERS` abgeleitet werden
  - Manuelles Override mit `OPENCLAW_DOCKER_AUTH_DIRS=all`, `OPENCLAW_DOCKER_AUTH_DIRS=none` oder einer Komma-Liste wie `OPENCLAW_DOCKER_AUTH_DIRS=.claude,.codex`
- `OPENCLAW_LIVE_GATEWAY_MODELS=...` / `OPENCLAW_LIVE_MODELS=...`, um den Lauf einzugrenzen
- `OPENCLAW_LIVE_GATEWAY_PROVIDERS=...` / `OPENCLAW_LIVE_PROVIDERS=...`, um Provider im Container zu filtern
- `OPENCLAW_SKIP_DOCKER_BUILD=1`, um ein vorhandenes Image `openclaw:local-live` für erneute Läufe wiederzuverwenden, die keinen Neubau benötigen
- `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`, um sicherzustellen, dass Anmeldedaten aus dem Profilspeicher kommen (nicht aus env)
- `OPENCLAW_OPENWEBUI_MODEL=...`, um das vom Gateway für den Open-WebUI-Smoke bereitgestellte Modell auszuwählen
- `OPENCLAW_OPENWEBUI_PROMPT=...`, um den für den Open-WebUI-Smoke verwendeten Nonce-Prüf-Prompt zu überschreiben
- `OPENWEBUI_IMAGE=...`, um das angeheftete Open-WebUI-Image-Tag zu überschreiben

## Docs-Sanity

Führe nach Änderungen an der Dokumentation die Docs-Prüfungen aus: `pnpm check:docs`.
Führe die vollständige Mintlify-Anchor-Validierung aus, wenn du auch Prüfungen für In-Page-Überschriften benötigst: `pnpm docs:check-links:anchors`.

## Offline-Regression (CI-sicher)

Das sind Regressionstests für die „reale Pipeline“ ohne echte Provider:

- Gateway-Tool-Calling (Mock-OpenAI, echtes Gateway + Agent-Schleife): `src/gateway/gateway.test.ts` (Fall: "runs a mock OpenAI tool call end-to-end via gateway agent loop")
- Gateway-Assistent (WS `wizard.start`/`wizard.next`, schreibt Konfiguration + erzwungene Authentifizierung): `src/gateway/gateway.test.ts` (Fall: "runs wizard over ws and writes auth token config")

## Agent-Zuverlässigkeits-Evals (Skills)

Wir haben bereits einige CI-sichere Tests, die sich wie „Agent-Zuverlässigkeits-Evals“ verhalten:

- Mock-Tool-Calling über die echte Gateway- + Agent-Schleife (`src/gateway/gateway.test.ts`).
- End-to-End-Assistenten-Flows, die Sitzungsverdrahtung und Konfigurationseffekte validieren (`src/gateway/gateway.test.ts`).

Was für Skills noch fehlt (siehe [Skills](/de/tools/skills)):

- **Decisioning:** Wenn Skills im Prompt aufgeführt sind, wählt der Agent den richtigen Skill aus (oder vermeidet irrelevante)?
- **Compliance:** Liest der Agent vor der Nutzung `SKILL.md` und folgt den erforderlichen Schritten/Argumenten?
- **Workflow-Verträge:** Multi-Turn-Szenarien, die Tool-Reihenfolge, Übernahme des Sitzungsverlaufs und Sandbox-Grenzen prüfen.

Zukünftige Evals sollten zuerst deterministisch bleiben:

- Ein Szenario-Runner, der Mock-Provider verwendet, um Tool-Calls + Reihenfolge, Skill-Datei-Lesezugriffe und Sitzungsverdrahtung zu prüfen.
- Eine kleine Suite skill-fokussierter Szenarien (verwenden vs. vermeiden, Gating, Prompt-Injection).
- Optionale Live-Evals (Opt-in, env-gesteuert) erst, nachdem die CI-sichere Suite vorhanden ist.

## Vertragstests (Plugin- und Kanalform)

Vertragstests verifizieren, dass jedes registrierte Plugin und jeder Kanal seinem
Schnittstellenvertrag entspricht. Sie iterieren über alle entdeckten Plugins und führen eine Suite aus
Form- und Verhaltensprüfungen aus. Die standardmäßige Unit-Lane `pnpm test`
überspringt diese gemeinsam genutzten Seam- und Smoke-Dateien absichtlich; führe die Vertragsbefehle explizit aus,
wenn du gemeinsam genutzte Kanal- oder Provider-Oberflächen änderst.

### Befehle

- Alle Verträge: `pnpm test:contracts`
- Nur Kanalverträge: `pnpm test:contracts:channels`
- Nur Provider-Verträge: `pnpm test:contracts:plugins`

### Kanalverträge

Zu finden unter `src/channels/plugins/contracts/*.contract.test.ts`:

- **plugin** - Grundform des Plugins (ID, Name, Fähigkeiten)
- **setup** - Vertragsprüfung des Setup-Assistenten
- **session-binding** - Verhalten beim Sitzungs-Binding
- **outbound-payload** - Struktur der Nachrichten-Payload
- **inbound** - Verarbeitung eingehender Nachrichten
- **actions** - Handler für Kanalaktionen
- **threading** - Verarbeitung von Thread-IDs
- **directory** - API für Verzeichnis/Roster
- **group-policy** - Durchsetzung von Gruppenrichtlinien

### Provider-Statusverträge

Zu finden unter `src/plugins/contracts/*.contract.test.ts`.

- **status** - Kanal-Status-Probes
- **registry** - Form der Plugin-Registry

### Provider-Verträge

Zu finden unter `src/plugins/contracts/*.contract.test.ts`:

- **auth** - Vertrag des Authentifizierungsablaufs
- **auth-choice** - Authentifizierungswahl/-auswahl
- **catalog** - API des Modellkatalogs
- **discovery** - Plugin-Erkennung
- **loader** - Plugin-Laden
- **runtime** - Provider-Runtime
- **shape** - Plugin-Form/Schnittstelle
- **wizard** - Setup-Assistent

### Wann ausführen

- Nach Änderungen an `plugin-sdk`-Exports oder Subpfaden
- Nach dem Hinzufügen oder Ändern eines Kanal- oder Provider-Plugins
- Nach Refactorings der Plugin-Registrierung oder -Erkennung

Vertragstests laufen in CI und erfordern keine echten API-Schlüssel.

## Regressionen hinzufügen (Leitlinien)

Wenn du ein Provider-/Modellproblem behebst, das in Live entdeckt wurde:

- Füge nach Möglichkeit eine CI-sichere Regression hinzu (Mock-/Stub-Provider oder Erfassung der exakten Transformation der Request-Form)
- Wenn es von Natur aus nur live prüfbar ist (Rate Limits, Authentifizierungsrichtlinien), halte den Live-Test eng und als Opt-in über env-Variablen
- Bevorzuge die kleinste Ebene, die den Fehler erkennt:
  - Fehler bei Provider-Request-Konvertierung/-Replay → Test direkter Modelle
  - Fehler in Gateway-Sitzung/Verlauf/Tool-Pipeline → Gateway-Live-Smoke oder CI-sicherer Gateway-Mock-Test
- Schutzregel für SecretRef-Traversierung:
  - `src/secrets/exec-secret-ref-id-parity.test.ts` leitet aus Registry-Metadaten (`listSecretTargetRegistryEntries()`) ein Stichprobenziel pro SecretRef-Klasse ab und prüft dann, dass Exec-IDs von Traversierungssegmenten zurückgewiesen werden.
  - Wenn du in `src/secrets/target-registry-data.ts` eine neue SecretRef-Zielfamilie mit `includeInPlan` hinzufügst, aktualisiere `classifyTargetClass` in diesem Test. Der Test schlägt absichtlich bei nicht klassifizierten Ziel-IDs fehl, damit neue Klassen nicht stillschweigend übersprungen werden können.
