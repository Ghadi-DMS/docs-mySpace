---
read_when:
    - Tests lokal oder in CI ausführen
    - Regressionstests für Modell-/Provider-Fehler hinzufügen
    - Gateway- und Agent-Verhalten debuggen
summary: 'Test-Kit: Unit-/E2E-/Live-Suites, Docker-Runner und was jeder Test abdeckt'
title: Tests
x-i18n:
    generated_at: "2026-04-15T14:40:31Z"
    model: gpt-5.4
    provider: openai
    source_hash: ec3632cafa1f38b27510372391b84af744266df96c58f7fac98aa03763465db8
    source_path: help/testing.md
    workflow: 15
---

# Tests

OpenClaw hat drei Vitest-Suites (Unit/Integration, E2E, Live) und eine kleine Anzahl an Docker-Runnern.

Dieses Dokument ist ein Leitfaden dazu, **wie wir testen**:

- Was jede Suite abdeckt (und was sie bewusst _nicht_ abdeckt)
- Welche Befehle für gängige Workflows auszuführen sind (lokal, vor dem Push, Debugging)
- Wie Live-Tests Anmeldedaten erkennen und Modelle/Provider auswählen
- Wie man Regressionen für reale Modell-/Provider-Probleme hinzufügt

## Schnellstart

An den meisten Tagen:

- Vollständiges Gate (vor dem Push erwartet): `pnpm build && pnpm check && pnpm test`
- Schnellere lokale Ausführung der vollständigen Suite auf einer leistungsfähigen Maschine: `pnpm test:max`
- Direkte Vitest-Watch-Schleife: `pnpm test:watch`
- Direktes Dateitargeting leitet jetzt auch Pfade für Erweiterungen/Channels weiter: `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts`
- Bevorzuge zuerst gezielte Ausführungen, wenn du an einem einzelnen Fehler iterierst.
- Docker-gestützte QA-Site: `pnpm qa:lab:up`
- Linux-VM-gestützte QA-Strecke: `pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline`

Wenn du Tests anfasst oder zusätzliche Sicherheit möchtest:

- Coverage-Gate: `pnpm test:coverage`
- E2E-Suite: `pnpm test:e2e`

Beim Debuggen realer Provider/Modelle (erfordert echte Anmeldedaten):

- Live-Suite (Modelle + Gateway-Tool-/Image-Prüfungen): `pnpm test:live`
- Eine einzelne Live-Datei leise ausführen: `pnpm test:live -- src/agents/models.profiles.live.test.ts`

Tipp: Wenn du nur einen einzelnen fehlschlagenden Fall brauchst, grenze Live-Tests bevorzugt über die unten beschriebenen Allowlist-Umgebungsvariablen ein.

## QA-spezifische Runner

Diese Befehle stehen neben den Haupttest-Suites zur Verfügung, wenn du mehr QA-lab-Realismus brauchst:

- `pnpm openclaw qa suite`
  - Führt repo-gestützte QA-Szenarien direkt auf dem Host aus.
  - Führt standardmäßig mehrere ausgewählte Szenarien parallel mit isolierten Gateway-Workern aus, bis zu 64 Workern oder der Anzahl der ausgewählten Szenarien. Verwende `--concurrency <count>`, um die Anzahl der Worker anzupassen, oder `--concurrency 1` für die ältere serielle Strecke.
- `pnpm openclaw qa suite --runner multipass`
  - Führt dieselbe QA-Suite in einer kurzlebigen Multipass-Linux-VM aus.
  - Behält dasselbe Verhalten bei der Szenarioauswahl wie `qa suite` auf dem Host bei.
  - Verwendet dieselben Flags zur Provider-/Modellauswahl wie `qa suite`.
  - Live-Ausführungen reichen die unterstützten QA-Auth-Eingaben weiter, die für den Gast praktikabel sind:
    env-basierte Provider-Schlüssel, den QA-Live-Provider-Konfigurationspfad und `CODEX_HOME`, falls vorhanden.
  - Ausgabeordner müssen unterhalb der Repo-Wurzel bleiben, damit der Gast über den eingebundenen Workspace zurückschreiben kann.
  - Schreibt den normalen QA-Bericht + die Zusammenfassung sowie Multipass-Logs unter `.artifacts/qa-e2e/...`.
- `pnpm qa:lab:up`
  - Startet die Docker-gestützte QA-Site für operatorartige QA-Arbeit.
- `pnpm openclaw qa matrix`
  - Führt die Matrix-Live-QA-Strecke gegen einen kurzlebigen Docker-gestützten Tuwunel-Homeserver aus.
  - Dieser QA-Host ist aktuell nur für Repo/Entwicklung gedacht. Gepackte OpenClaw-Installationen liefern `qa-lab` nicht mit aus und stellen daher `openclaw qa` nicht bereit.
  - Repo-Checkouts laden den gebündelten Runner direkt; kein separater Schritt zur Plugin-Installation ist erforderlich.
  - Richtet drei temporäre Matrix-Benutzer (`driver`, `sut`, `observer`) sowie einen privaten Raum ein und startet dann ein QA-Gateway-Unterprozess mit dem echten Matrix-Plugin als SUT-Transport.
  - Verwendet standardmäßig das festgepinnte stabile Tuwunel-Image `ghcr.io/matrix-construct/tuwunel:v1.5.1`. Mit `OPENCLAW_QA_MATRIX_TUWUNEL_IMAGE` kannst du ein anderes Image zum Testen überschreiben.
  - Matrix stellt keine gemeinsamen Flags für Anmeldedatenquellen bereit, da die Strecke lokal kurzlebige Benutzer bereitstellt.
  - Schreibt einen Matrix-QA-Bericht, eine Zusammenfassung und ein Artifact mit beobachteten Ereignissen unter `.artifacts/qa-e2e/...`.
- `pnpm openclaw qa telegram`
  - Führt die Telegram-Live-QA-Strecke gegen eine echte private Gruppe aus und verwendet dabei die Bot-Token für Driver und SUT aus der Umgebung.
  - Erfordert `OPENCLAW_QA_TELEGRAM_GROUP_ID`, `OPENCLAW_QA_TELEGRAM_DRIVER_BOT_TOKEN` und `OPENCLAW_QA_TELEGRAM_SUT_BOT_TOKEN`. Die Gruppen-ID muss die numerische Telegram-Chat-ID sein.
  - Unterstützt `--credential-source convex` für gemeinsam genutzte gepoolte Anmeldedaten. Verwende standardmäßig den env-Modus oder setze `OPENCLAW_QA_CREDENTIAL_SOURCE=convex`, um gepoolte Leases zu nutzen.
  - Erfordert zwei unterschiedliche Bots in derselben privaten Gruppe, wobei der SUT-Bot einen Telegram-Benutzernamen bereitstellen muss.
  - Für eine stabile Bot-zu-Bot-Beobachtung aktiviere in `@BotFather` für beide Bots den Bot-to-Bot Communication Mode und stelle sicher, dass der Driver-Bot Bot-Verkehr in der Gruppe beobachten kann.
  - Schreibt einen Telegram-QA-Bericht, eine Zusammenfassung und ein Artifact mit beobachteten Nachrichten unter `.artifacts/qa-e2e/...`.

Live-Transport-Strecken teilen sich einen gemeinsamen Standardvertrag, damit neue Transporte nicht auseinanderdriften:

`qa-channel` bleibt die breit angelegte synthetische QA-Suite und ist nicht Teil der Live-Transport-Abdeckungsmatrix.

| Strecke | Canary | Mention-Gating | Allowlist-Block | Antwort auf oberster Ebene | Fortsetzung nach Neustart | Thread-Follow-up | Thread-Isolation | Reaktionsbeobachtung | Help-Befehl |
| -------- | ------ | -------------- | --------------- | -------------------------- | ------------------------- | ---------------- | ---------------- | -------------------- | ----------- |
| Matrix   | x      | x              | x               | x                          | x                         | x                | x                | x                    |             |
| Telegram | x      |                |                 |                            |                           |                  |                  |                      | x           |

### Gemeinsame Telegram-Anmeldedaten über Convex (v1)

Wenn `--credential-source convex` (oder `OPENCLAW_QA_CREDENTIAL_SOURCE=convex`) für
`openclaw qa telegram` aktiviert ist, bezieht QA lab ein exklusives Lease aus einem Convex-gestützten Pool, sendet Heartbeat-Signale für dieses Lease, solange die Strecke läuft, und gibt das Lease beim Beenden wieder frei.

Referenzgerüst für ein Convex-Projekt:

- `qa/convex-credential-broker/`

Erforderliche env vars:

- `OPENCLAW_QA_CONVEX_SITE_URL` (zum Beispiel `https://your-deployment.convex.site`)
- Ein Secret für die ausgewählte Rolle:
  - `OPENCLAW_QA_CONVEX_SECRET_MAINTAINER` für `maintainer`
  - `OPENCLAW_QA_CONVEX_SECRET_CI` für `ci`
- Auswahl der Rolle für Anmeldedaten:
  - CLI: `--credential-role maintainer|ci`
  - Env-Standard: `OPENCLAW_QA_CREDENTIAL_ROLE` (Standard ist `maintainer`)

Optionale env vars:

- `OPENCLAW_QA_CREDENTIAL_LEASE_TTL_MS` (Standard `1200000`)
- `OPENCLAW_QA_CREDENTIAL_HEARTBEAT_INTERVAL_MS` (Standard `30000`)
- `OPENCLAW_QA_CREDENTIAL_ACQUIRE_TIMEOUT_MS` (Standard `90000`)
- `OPENCLAW_QA_CREDENTIAL_HTTP_TIMEOUT_MS` (Standard `15000`)
- `OPENCLAW_QA_CONVEX_ENDPOINT_PREFIX` (Standard `/qa-credentials/v1`)
- `OPENCLAW_QA_CREDENTIAL_OWNER_ID` (optionale Trace-ID)
- `OPENCLAW_QA_ALLOW_INSECURE_HTTP=1` erlaubt Loopback-`http://`-Convex-URLs nur für lokale Entwicklung.

`OPENCLAW_QA_CONVEX_SITE_URL` sollte im Normalbetrieb `https://` verwenden.

Admin-Befehle für Maintainer (Pool hinzufügen/entfernen/auflisten) erfordern
explizit `OPENCLAW_QA_CONVEX_SECRET_MAINTAINER`.

CLI-Hilfsbefehle für Maintainer:

```bash
pnpm openclaw qa credentials add --kind telegram --payload-file qa/telegram-credential.json
pnpm openclaw qa credentials list --kind telegram
pnpm openclaw qa credentials remove --credential-id <credential-id>
```

Verwende `--json` für maschinenlesbare Ausgabe in Skripten und CI-Hilfsprogrammen.

Standard-Endpunktvertrag (`OPENCLAW_QA_CONVEX_SITE_URL` + `/qa-credentials/v1`):

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

Payload-Form für Telegram-Typ:

- `{ groupId: string, driverToken: string, sutToken: string }`
- `groupId` muss eine numerische Telegram-Chat-ID als String sein.
- `admin/add` validiert diese Form für `kind: "telegram"` und weist fehlerhafte Payloads zurück.

### Einen Channel zu QA hinzufügen

Das Hinzufügen eines Channels zum Markdown-QA-System erfordert genau zwei Dinge:

1. Einen Transport-Adapter für den Channel.
2. Ein Szenariopaket, das den Channel-Vertrag testet.

Füge keinen neuen QA-Befehlsstamm auf oberster Ebene hinzu, wenn der gemeinsame `qa-lab`-Host
den Ablauf besitzen kann.

`qa-lab` besitzt die gemeinsamen Host-Mechaniken:

- den `openclaw qa`-Befehlsstamm
- Start und Beenden der Suite
- Worker-Konkurrenz
- Schreiben von Artifacts
- Berichtserstellung
- Szenarioausführung
- Kompatibilitätsaliasse für ältere `qa-channel`-Szenarien

Runner-Plugins besitzen den Transportvertrag:

- wie `openclaw qa <runner>` unter dem gemeinsamen `qa`-Stamm eingehängt wird
- wie das Gateway für diesen Transport konfiguriert wird
- wie Bereitschaft geprüft wird
- wie eingehende Ereignisse injiziert werden
- wie ausgehende Nachrichten beobachtet werden
- wie Transkripte und normalisierter Transportzustand bereitgestellt werden
- wie transportgestützte Aktionen ausgeführt werden
- wie transport-spezifisches Zurücksetzen oder Aufräumen behandelt wird

Die minimale Voraussetzung für die Einführung eines neuen Channels ist:

1. `qa-lab` als Besitzer des gemeinsamen `qa`-Stamms beibehalten.
2. Den Transport-Runner auf der gemeinsamen `qa-lab`-Host-Seam implementieren.
3. Transport-spezifische Mechaniken im Runner-Plugin oder Channel-Harness belassen.
4. Den Runner als `openclaw qa <runner>` einhängen, statt einen konkurrierenden Root-Befehl zu registrieren.
   Runner-Plugins sollten `qaRunners` in `openclaw.plugin.json` deklarieren und ein passendes `qaRunnerCliRegistrations`-Array aus `runtime-api.ts` exportieren.
   Halte `runtime-api.ts` schlank; lazy CLI- und Runner-Ausführung sollten hinter separaten Entrypoints bleiben.
5. Markdown-Szenarien unter `qa/scenarios/` verfassen oder anpassen.
6. Die generischen Szenario-Helfer für neue Szenarien verwenden.
7. Bestehende Kompatibilitätsaliasse funktionsfähig halten, außer das Repo führt bewusst eine Migration durch.

Die Entscheidungsregel ist strikt:

- Wenn Verhalten einmalig in `qa-lab` ausgedrückt werden kann, gehört es in `qa-lab`.
- Wenn Verhalten von einem Channel-Transport abhängt, bleibt es in diesem Runner-Plugin oder Plugin-Harness.
- Wenn ein Szenario eine neue Fähigkeit benötigt, die mehr als ein Channel nutzen kann, füge einen generischen Helfer hinzu statt eines Channel-spezifischen Zweigs in `suite.ts`.
- Wenn ein Verhalten nur für einen Transport sinnvoll ist, halte das Szenario transport-spezifisch und mache das im Szenariovertrag explizit.

Bevorzugte generische Helfernamen für neue Szenarien sind:

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

Kompatibilitätsaliasse bleiben für bestehende Szenarien verfügbar, darunter:

- `waitForQaChannelReady`
- `waitForOutboundMessage`
- `waitForNoOutbound`
- `formatConversationTranscript`
- `resetBus`

Neue Channel-Arbeit sollte die generischen Helfernamen verwenden.
Kompatibilitätsaliasse existieren, um eine Migration an einem Stichtag zu vermeiden, nicht als Modell für
neue Szenarioerstellung.

## Test-Suites (was wo läuft)

Stelle dir die Suites als „zunehmenden Realismus“ vor (und zunehmende Instabilität/Kosten):

### Unit / Integration (Standard)

- Befehl: `pnpm test`
- Konfiguration: zehn sequenzielle Shard-Läufe (`vitest.full-*.config.ts`) über die bestehenden abgegrenzten Vitest-Projekte
- Dateien: Core-/Unit-Inventare unter `src/**/*.test.ts`, `packages/**/*.test.ts`, `test/**/*.test.ts` sowie die per Whitelist freigegebenen `ui`-Node-Tests, die von `vitest.unit.config.ts` abgedeckt werden
- Umfang:
  - Reine Unit-Tests
  - In-Process-Integrationstests (Gateway-Auth, Routing, Tooling, Parsing, Konfiguration)
  - Deterministische Regressionen für bekannte Fehler
- Erwartungen:
  - Läuft in CI
  - Keine echten Schlüssel erforderlich
  - Sollte schnell und stabil sein
- Hinweis zu Projekten:
  - Nicht gezieltes `pnpm test` führt jetzt elf kleinere Shard-Konfigurationen aus (`core-unit-src`, `core-unit-security`, `core-unit-ui`, `core-unit-support`, `core-support-boundary`, `core-contracts`, `core-bundled`, `core-runtime`, `agentic`, `auto-reply`, `extensions`) statt eines einzigen großen nativen Root-Projekt-Prozesses. Das senkt den Spitzen-RSS auf ausgelasteten Maschinen und verhindert, dass Auto-Reply-/Erweiterungsarbeit andere Suites ausbremst.
  - `pnpm test --watch` verwendet weiterhin den nativen Root-`vitest.config.ts`-Projektgraphen, weil eine Multi-Shard-Watch-Schleife nicht praktikabel ist.
  - `pnpm test`, `pnpm test:watch` und `pnpm test:perf:imports` leiten explizite Datei-/Verzeichnisziele jetzt zuerst durch abgegrenzte Strecken, sodass `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts` nicht den Startaufwand des vollständigen Root-Projekts zahlen muss.
  - `pnpm test:changed` erweitert geänderte Git-Pfade in dieselben abgegrenzten Strecken, wenn der Diff nur routbare Source-/Test-Dateien berührt; Konfigurations-/Setup-Änderungen fallen weiterhin auf den breiten erneuten Root-Projekt-Lauf zurück.
  - Import-leichte Unit-Tests aus Agents, Commands, Plugins, Auto-Reply-Helfern, `plugin-sdk` und ähnlichen reinen Utility-Bereichen laufen über die `unit-fast`-Strecke, die `test/setup-openclaw-runtime.ts` überspringt; zustandsbehaftete/laufzeitintensive Dateien bleiben auf den bestehenden Strecken.
  - Ausgewählte `plugin-sdk`- und `commands`-Helper-Quelldateien ordnen Läufe im Changed-Modus ebenfalls expliziten benachbarten Tests in diesen leichten Strecken zu, sodass Helper-Änderungen kein erneutes Ausführen der vollständigen schweren Suite für dieses Verzeichnis auslösen.
  - `auto-reply` hat jetzt drei dedizierte Buckets: Core-Helper auf oberster Ebene, `reply.*`-Integrationstests auf oberster Ebene und den Teilbaum `src/auto-reply/reply/**`. Dadurch bleibt die schwerste Reply-Harness-Arbeit von den günstigen Status-/Chunk-/Token-Tests getrennt.
- Hinweis zum eingebetteten Runner:
  - Wenn du Eingaben für die Discovery von Message-Tools oder den Laufzeitkontext von Compaction änderst,
    halte beide Abdeckungsebenen intakt.
  - Füge fokussierte Helper-Regressionen für reine Routing-/Normalisierungsgrenzen hinzu.
  - Halte außerdem die Integrations-Suites des eingebetteten Runners gesund:
    `src/agents/pi-embedded-runner/compact.hooks.test.ts`,
    `src/agents/pi-embedded-runner/run.overflow-compaction.test.ts` und
    `src/agents/pi-embedded-runner/run.overflow-compaction.loop.test.ts`.
  - Diese Suites verifizieren, dass abgegrenzte IDs und Compaction-Verhalten weiterhin
    durch die echten Pfade `run.ts` / `compact.ts` fließen; reine Helper-Tests sind kein
    ausreichender Ersatz für diese Integrationspfade.
- Hinweis zum Pool:
  - Die Basis-Vitest-Konfiguration verwendet jetzt standardmäßig `threads`.
  - Die gemeinsame Vitest-Konfiguration setzt außerdem `isolate: false` fest und verwendet den nicht isolierten Runner über die Root-Projekte, E2E- und Live-Konfigurationen hinweg.
  - Die Root-UI-Strecke behält ihr `jsdom`-Setup und ihren Optimizer bei, läuft jetzt aber ebenfalls auf dem gemeinsamen nicht isolierten Runner.
  - Jeder `pnpm test`-Shard übernimmt dieselben Standards `threads` + `isolate: false` aus der gemeinsamen Vitest-Konfiguration.
  - Der gemeinsame Launcher `scripts/run-vitest.mjs` fügt für Vitest-Child-Node-Prozesse jetzt standardmäßig ebenfalls `--no-maglev` hinzu, um den V8-Kompilierungs-Overhead bei großen lokalen Läufen zu reduzieren. Setze `OPENCLAW_VITEST_ENABLE_MAGLEV=1`, wenn du mit dem Standardverhalten von V8 vergleichen musst.
- Hinweis zur schnellen lokalen Iteration:
  - `pnpm test:changed` leitet durch abgegrenzte Strecken, wenn die geänderten Pfade eindeutig einer kleineren Suite zugeordnet werden können.
  - `pnpm test:max` und `pnpm test:changed:max` behalten dasselbe Routing-Verhalten bei, nur mit einer höheren Worker-Obergrenze.
  - Die automatische lokale Worker-Skalierung ist jetzt absichtlich konservativ und fährt ebenfalls zurück, wenn die Host-Load-Average bereits hoch ist, sodass mehrere gleichzeitige Vitest-Läufe standardmäßig weniger Schaden anrichten.
  - Die Basis-Vitest-Konfiguration markiert die Projekte/Konfigurationsdateien als `forceRerunTriggers`, damit erneute Läufe im Changed-Modus korrekt bleiben, wenn sich das Test-Wiring ändert.
  - Die Konfiguration lässt `OPENCLAW_VITEST_FS_MODULE_CACHE` auf unterstützten Hosts aktiviert; setze `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/abs/path`, wenn du einen expliziten Cache-Ort für direktes Profiling möchtest.
- Hinweis zum Performance-Debugging:
  - `pnpm test:perf:imports` aktiviert die Vitest-Berichterstattung zur Importdauer sowie die Ausgabe der Importaufschlüsselung.
  - `pnpm test:perf:imports:changed` grenzt dieselbe Profiling-Ansicht auf Dateien ein, die sich seit `origin/main` geändert haben.
- `pnpm test:perf:changed:bench -- --ref <git-ref>` vergleicht das geroutete `test:changed` mit dem nativen Root-Projekt-Pfad für diesen festgeschriebenen Diff und gibt Laufzeit sowie den maximalen RSS unter macOS aus.
- `pnpm test:perf:changed:bench -- --worktree` benchmarkt den aktuellen Dirty-Tree, indem die Liste geänderter Dateien durch `scripts/test-projects.mjs` und die Root-Vitest-Konfiguration geroutet wird.
  - `pnpm test:perf:profile:main` schreibt ein CPU-Profil des Main-Threads für den Vitest-/Vite-Start und den Transform-Overhead.
  - `pnpm test:perf:profile:runner` schreibt CPU- und Heap-Profile des Runners für die Unit-Suite bei deaktivierter Dateiparallelität.

### E2E (Gateway-Smoke)

- Befehl: `pnpm test:e2e`
- Konfiguration: `vitest.e2e.config.ts`
- Dateien: `src/**/*.e2e.test.ts`, `test/**/*.e2e.test.ts`
- Laufzeitstandards:
  - Verwendet Vitest-`threads` mit `isolate: false` und entspricht damit dem Rest des Repos.
  - Verwendet adaptive Worker (CI: bis zu 2, lokal: standardmäßig 1).
  - Läuft standardmäßig im Silent-Modus, um den Konsolen-I/O-Overhead zu reduzieren.
- Nützliche Überschreibungen:
  - `OPENCLAW_E2E_WORKERS=<n>`, um die Anzahl der Worker zu erzwingen (begrenzt auf 16).
  - `OPENCLAW_E2E_VERBOSE=1`, um ausführliche Konsolenausgabe wieder zu aktivieren.
- Umfang:
  - End-to-End-Verhalten von Multi-Instance-Gateways
  - WebSocket-/HTTP-Oberflächen, Node-Pairing und schwergewichtigere Netzwerkarbeit
- Erwartungen:
  - Läuft in CI (wenn in der Pipeline aktiviert)
  - Keine echten Schlüssel erforderlich
  - Mehr bewegliche Teile als Unit-Tests (kann langsamer sein)

### E2E: OpenShell-Backend-Smoke

- Befehl: `pnpm test:e2e:openshell`
- Datei: `test/openshell-sandbox.e2e.test.ts`
- Umfang:
  - Startet über Docker ein isoliertes OpenShell-Gateway auf dem Host
  - Erstellt aus einem temporären lokalen Dockerfile eine Sandbox
  - Testet OpenClaws OpenShell-Backend über echtes `sandbox ssh-config` + SSH-Exec
  - Verifiziert kanonisches Remote-Dateisystemverhalten über die Sandbox-FS-Bridge
- Erwartungen:
  - Nur per Opt-in; nicht Teil des standardmäßigen `pnpm test:e2e`-Laufs
  - Erfordert eine lokale `openshell`-CLI plus einen funktionierenden Docker-Daemon
  - Verwendet isoliertes `HOME` / `XDG_CONFIG_HOME` und zerstört dann Test-Gateway und Sandbox
- Nützliche Überschreibungen:
  - `OPENCLAW_E2E_OPENSHELL=1`, um den Test zu aktivieren, wenn die breitere E2E-Suite manuell ausgeführt wird
  - `OPENCLAW_E2E_OPENSHELL_COMMAND=/path/to/openshell`, um auf ein nicht standardmäßiges CLI-Binary oder Wrapper-Skript zu verweisen

### Live (echte Provider + echte Modelle)

- Befehl: `pnpm test:live`
- Konfiguration: `vitest.live.config.ts`
- Dateien: `src/**/*.live.test.ts`
- Standard: durch `pnpm test:live` **aktiviert** (setzt `OPENCLAW_LIVE_TEST=1`)
- Umfang:
  - „Funktioniert dieser Provider/dieses Modell _heute_ tatsächlich mit echten Anmeldedaten?“
  - Erkennt Änderungen an Provider-Formaten, Tool-Calling-Eigenheiten, Auth-Probleme und Rate-Limit-Verhalten
- Erwartungen:
  - Absichtlich nicht CI-stabil (echte Netzwerke, echte Provider-Richtlinien, Quoten, Ausfälle)
  - Kostet Geld / verbraucht Rate Limits
  - Statt „alles“ möglichst eingegrenzte Teilmengen ausführen
- Live-Läufe sourcen `~/.profile`, um fehlende API-Schlüssel zu übernehmen.
- Standardmäßig isolieren Live-Läufe `HOME` weiterhin und kopieren Konfigurations-/Auth-Material in ein temporäres Test-Home, damit Unit-Fixtures dein echtes `~/.openclaw` nicht verändern können.
- Setze `OPENCLAW_LIVE_USE_REAL_HOME=1` nur dann, wenn Live-Tests absichtlich dein echtes Home-Verzeichnis verwenden sollen.
- `pnpm test:live` verwendet jetzt standardmäßig einen ruhigeren Modus: `[live] ...`-Fortschrittsausgabe bleibt erhalten, aber der zusätzliche Hinweis zu `~/.profile` wird unterdrückt und Gateway-Bootstrap-Logs/Bonjour-Chatter werden stummgeschaltet. Setze `OPENCLAW_LIVE_TEST_QUIET=0`, wenn du wieder die vollständigen Start-Logs sehen möchtest.
- API-Schlüsselrotation (provider-spezifisch): setze `*_API_KEYS` im Komma-/Semikolonformat oder `*_API_KEY_1`, `*_API_KEY_2` (zum Beispiel `OPENAI_API_KEYS`, `ANTHROPIC_API_KEYS`, `GEMINI_API_KEYS`) oder pro Live-Überschreibung `OPENCLAW_LIVE_*_KEY`; Tests versuchen bei Rate-Limit-Antworten erneut.
- Fortschritts-/Heartbeat-Ausgabe:
  - Live-Suites geben jetzt Fortschrittszeilen auf stderr aus, sodass lange Provider-Aufrufe sichtbar aktiv bleiben, auch wenn die Vitest-Konsolenerfassung ruhig ist.
  - `vitest.live.config.ts` deaktiviert die Vitest-Konsolenabfangung, sodass Provider-/Gateway-Fortschrittszeilen bei Live-Läufen sofort gestreamt werden.
  - Passe Heartbeats für direkte Modelle mit `OPENCLAW_LIVE_HEARTBEAT_MS` an.
  - Passe Heartbeats für Gateway/Probes mit `OPENCLAW_LIVE_GATEWAY_HEARTBEAT_MS` an.

## Welche Suite sollte ich ausführen?

Verwende diese Entscheidungstabelle:

- Logik/Tests bearbeiten: `pnpm test` ausführen (und `pnpm test:coverage`, wenn du viel geändert hast)
- Gateway-Networking / WS-Protokoll / Pairing anfassen: `pnpm test:e2e` ergänzen
- „Mein Bot ist down“ / provider-spezifische Fehler / Tool-Calling debuggen: ein eingegrenztes `pnpm test:live` ausführen

## Live: Android-Node-Capability-Sweep

- Test: `src/gateway/android-node.capabilities.live.test.ts`
- Skript: `pnpm android:test:integration`
- Ziel: **jeden aktuell beworbenen Befehl** eines verbundenen Android-Nodes aufrufen und das Befehlsvertragsverhalten prüfen.
- Umfang:
  - Vorgegebene/manuelle Vorbereitung (die Suite installiert/startet/pairt die App nicht).
  - Befehlsweise `node.invoke`-Validierung des Gateways für den ausgewählten Android-Node.
- Erforderliche Vorbereitung:
  - Android-App bereits verbunden und mit dem Gateway gepairt.
  - App im Vordergrund halten.
  - Berechtigungen/Capture-Zustimmung für Capabilities erteilt, von denen du erwartest, dass sie bestehen.
- Optionale Zielüberschreibungen:
  - `OPENCLAW_ANDROID_NODE_ID` oder `OPENCLAW_ANDROID_NODE_NAME`.
  - `OPENCLAW_ANDROID_GATEWAY_URL` / `OPENCLAW_ANDROID_GATEWAY_TOKEN` / `OPENCLAW_ANDROID_GATEWAY_PASSWORD`.
- Vollständige Android-Setup-Details: [Android-App](/de/platforms/android)

## Live: Modell-Smoke (Profile-Keys)

Live-Tests sind in zwei Ebenen aufgeteilt, damit wir Fehler isolieren können:

- „Direktes Modell“ sagt uns, ob der Provider/das Modell mit dem angegebenen Schlüssel überhaupt antworten kann.
- „Gateway-Smoke“ sagt uns, ob die vollständige Gateway+Agent-Pipeline für dieses Modell funktioniert (Sitzungen, Verlauf, Tools, Sandbox-Policy usw.).

### Ebene 1: Direkte Modell-Completion (ohne Gateway)

- Test: `src/agents/models.profiles.live.test.ts`
- Ziel:
  - Erkannte Modelle auflisten
  - `getApiKeyForModel` verwenden, um Modelle auszuwählen, für die du Anmeldedaten hast
  - Pro Modell eine kleine Completion ausführen (und gezielte Regressionen, wo nötig)
- Aktivierung:
  - `pnpm test:live` (oder `OPENCLAW_LIVE_TEST=1`, wenn Vitest direkt aufgerufen wird)
- Setze `OPENCLAW_LIVE_MODELS=modern` (oder `all`, Alias für modern), damit diese Suite tatsächlich ausgeführt wird; andernfalls wird sie übersprungen, damit `pnpm test:live` auf Gateway-Smoke fokussiert bleibt
- Modellauswahl:
  - `OPENCLAW_LIVE_MODELS=modern`, um die moderne Allowlist auszuführen (Opus/Sonnet 4.6+, GPT-5.x + Codex, Gemini 3, GLM 4.7, MiniMax M2.7, Grok 4)
  - `OPENCLAW_LIVE_MODELS=all` ist ein Alias für die moderne Allowlist
  - oder `OPENCLAW_LIVE_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,..."` (Komma-Allowlist)
  - Moderne/alle Sweeps verwenden standardmäßig eine kuratierte Obergrenze mit hohem Signal; setze `OPENCLAW_LIVE_MAX_MODELS=0` für einen vollständigen modernen Sweep oder einen positiven Wert für eine kleinere Obergrenze.
- Providerauswahl:
  - `OPENCLAW_LIVE_PROVIDERS="google,google-antigravity,google-gemini-cli"` (Komma-Allowlist)
- Woher die Schlüssel kommen:
  - Standardmäßig: Profile-Store und env-Fallbacks
  - Setze `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`, um **nur den Profile-Store** zu erzwingen
- Warum es das gibt:
  - Trennt „Provider-API ist defekt / Schlüssel ist ungültig“ von „Gateway-Agent-Pipeline ist defekt“
  - Enthält kleine, isolierte Regressionen (Beispiel: OpenAI-Responses/Codex-Responses-Reasoning-Replay- und Tool-Call-Flows)

### Ebene 2: Gateway + Dev-Agent-Smoke (was `@openclaw` tatsächlich tut)

- Test: `src/gateway/gateway-models.profiles.live.test.ts`
- Ziel:
  - Ein In-Process-Gateway starten
  - Eine `agent:dev:*`-Sitzung erstellen/patchen (Modell-Override pro Lauf)
  - Modelle mit Schlüsseln durchlaufen und Folgendes prüfen:
    - „sinnvolle“ Antwort (ohne Tools)
    - ein echter Tool-Aufruf funktioniert (`read`-Probe)
    - optionale zusätzliche Tool-Probes (`exec+read`-Probe)
    - OpenAI-Regressionspfade (nur Tool-Call → Follow-up) funktionieren weiterhin
- Probe-Details (damit du Fehler schnell erklären kannst):
  - `read`-Probe: Der Test schreibt eine Nonce-Datei in den Workspace und fordert den Agenten auf, sie zu `read`en und die Nonce zurückzugeben.
  - `exec+read`-Probe: Der Test fordert den Agenten auf, per `exec` eine Nonce in eine temporäre Datei zu schreiben und sie dann per `read` wieder auszulesen.
  - Image-Probe: Der Test hängt ein generiertes PNG an (Katze + zufälliger Code) und erwartet, dass das Modell `cat <CODE>` zurückgibt.
  - Implementierungsreferenz: `src/gateway/gateway-models.profiles.live.test.ts` und `src/gateway/live-image-probe.ts`.
- Aktivierung:
  - `pnpm test:live` (oder `OPENCLAW_LIVE_TEST=1`, wenn Vitest direkt aufgerufen wird)
- Modellauswahl:
  - Standard: moderne Allowlist (Opus/Sonnet 4.6+, GPT-5.x + Codex, Gemini 3, GLM 4.7, MiniMax M2.7, Grok 4)
  - `OPENCLAW_LIVE_GATEWAY_MODELS=all` ist ein Alias für die moderne Allowlist
  - Oder `OPENCLAW_LIVE_GATEWAY_MODELS="provider/model"` setzen (oder Komma-Liste), um einzugrenzen
  - Moderne/alle Gateway-Sweeps verwenden standardmäßig eine kuratierte Obergrenze mit hohem Signal; setze `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=0` für einen vollständigen modernen Sweep oder einen positiven Wert für eine kleinere Obergrenze.
- Providerauswahl (vermeide „alles von OpenRouter“):
  - `OPENCLAW_LIVE_GATEWAY_PROVIDERS="google,google-antigravity,google-gemini-cli,openai,anthropic,zai,minimax"` (Komma-Allowlist)
- Tool- + Image-Probes sind in diesem Live-Test immer aktiviert:
  - `read`-Probe + `exec+read`-Probe (Tool-Stresstest)
  - Die Image-Probe läuft, wenn das Modell Unterstützung für Bildeingaben bewirbt
  - Ablauf (grobe Übersicht):
    - Der Test erzeugt ein kleines PNG mit „CAT“ + zufälligem Code (`src/gateway/live-image-probe.ts`)
    - Sendet es über `agent` mit `attachments: [{ mimeType: "image/png", content: "<base64>" }]`
    - Das Gateway parst Attachments in `images[]` (`src/gateway/server-methods/agent.ts` + `src/gateway/chat-attachments.ts`)
    - Der eingebettete Agent leitet eine multimodale Benutzernachricht an das Modell weiter
    - Prüfung: Die Antwort enthält `cat` + den Code (OCR-Toleranz: kleine Fehler sind erlaubt)

Tipp: Um zu sehen, was du auf deiner Maschine testen kannst (und die genauen `provider/model`-IDs), führe Folgendes aus:

```bash
openclaw models list
openclaw models list --json
```

## Live: CLI-Backend-Smoke (Claude, Codex, Gemini oder andere lokale CLIs)

- Test: `src/gateway/gateway-cli-backend.live.test.ts`
- Ziel: die Gateway- + Agent-Pipeline mit einem lokalen CLI-Backend validieren, ohne deine Standardkonfiguration anzufassen.
- Backend-spezifische Smoke-Standards liegen in der Definition `cli-backend.ts` der jeweiligen besitzenden Erweiterung.
- Aktivierung:
  - `pnpm test:live` (oder `OPENCLAW_LIVE_TEST=1`, wenn Vitest direkt aufgerufen wird)
  - `OPENCLAW_LIVE_CLI_BACKEND=1`
- Standards:
  - Standard-Provider/-Modell: `claude-cli/claude-sonnet-4-6`
  - Verhalten für Command/Args/Image kommt aus den Metadaten des jeweiligen CLI-Backend-Plugins.
- Überschreibungen (optional):
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL="codex-cli/gpt-5.4"`
  - `OPENCLAW_LIVE_CLI_BACKEND_COMMAND="/full/path/to/codex"`
  - `OPENCLAW_LIVE_CLI_BACKEND_ARGS='["exec","--json","--color","never","--sandbox","read-only","--skip-git-repo-check"]'`
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_PROBE=1`, um ein echtes Bild-Attachment zu senden (Pfade werden in den Prompt injiziert).
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_ARG="--image"`, um Bilddateipfade als CLI-Args statt per Prompt-Injektion zu übergeben.
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_MODE="repeat"` (oder `"list"`), um zu steuern, wie Bild-Args übergeben werden, wenn `IMAGE_ARG` gesetzt ist.
  - `OPENCLAW_LIVE_CLI_BACKEND_RESUME_PROBE=1`, um einen zweiten Turn zu senden und den Resume-Flow zu validieren.
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL_SWITCH_PROBE=0`, um die standardmäßige Kontinuitätsprobe Claude Sonnet -> Opus in derselben Sitzung zu deaktivieren (auf `1` setzen, um sie zu erzwingen, wenn das ausgewählte Modell ein Umschaltziel unterstützt).

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
- Er führt den Live-CLI-Backend-Smoke im Repo-Docker-Image als Nicht-Root-Benutzer `node` aus.
- Er löst CLI-Smoke-Metadaten aus der jeweiligen besitzenden Erweiterung auf und installiert dann das passende Linux-CLI-Paket (`@anthropic-ai/claude-code`, `@openai/codex` oder `@google/gemini-cli`) in ein gecachtes beschreibbares Präfix unter `OPENCLAW_DOCKER_CLI_TOOLS_DIR` (Standard: `~/.cache/openclaw/docker-cli-tools`).
- `pnpm test:docker:live-cli-backend:claude-subscription` erfordert portables Claude-Code-Subscription-OAuth entweder über `~/.claude/.credentials.json` mit `claudeAiOauth.subscriptionType` oder `CLAUDE_CODE_OAUTH_TOKEN` aus `claude setup-token`. Es prüft zuerst direktes `claude -p` in Docker und führt dann zwei Gateway-CLI-Backend-Turns aus, ohne Anhtropic-API-Key-env vars beizubehalten. Diese Subscription-Strecke deaktiviert standardmäßig die Claude-MCP-/Tool- und Image-Probes, weil Claude die Nutzung von Drittanbieter-Apps derzeit über Zusatznutzungsabrechnung statt über normale Limits des Subscription-Tarifs leitet.
- Der Live-CLI-Backend-Smoke testet jetzt denselben End-to-End-Ablauf für Claude, Codex und Gemini: Text-Turn, Bildklassifizierungs-Turn, dann MCP-Tool-Call `cron`, verifiziert über die Gateway-CLI.
- Der standardmäßige Claude-Smoke patcht die Sitzung außerdem von Sonnet auf Opus und verifiziert, dass die fortgesetzte Sitzung sich weiterhin an eine frühere Notiz erinnert.

## Live: ACP-Bind-Smoke (`/acp spawn ... --bind here`)

- Test: `src/gateway/gateway-acp-bind.live.test.ts`
- Ziel: den echten ACP-Conversation-Bind-Flow mit einem Live-ACP-Agenten validieren:
  - `/acp spawn <agent> --bind here` senden
  - eine synthetische Message-Channel-Konversation an Ort und Stelle binden
  - ein normales Follow-up in derselben Konversation senden
  - verifizieren, dass das Follow-up im Transkript der gebundenen ACP-Sitzung landet
- Aktivierung:
  - `pnpm test:live src/gateway/gateway-acp-bind.live.test.ts`
  - `OPENCLAW_LIVE_ACP_BIND=1`
- Standards:
  - ACP-Agenten in Docker: `claude,codex,gemini`
  - ACP-Agent für direktes `pnpm test:live ...`: `claude`
  - Synthetischer Channel: Konversationskontext im Stil einer Slack-DM
  - ACP-Backend: `acpx`
- Überschreibungen:
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=claude`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=codex`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude,codex,gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND='npx -y @agentclientprotocol/claude-agent-acp@<version>'`
- Hinweise:
  - Diese Strecke verwendet die Gateway-Oberfläche `chat.send` mit Admin-only-Feldern für synthetische Ausgangsrouten, damit Tests Message-Channel-Kontext anhängen können, ohne vorzugeben, extern zuzustellen.
  - Wenn `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND` nicht gesetzt ist, verwendet der Test die eingebaute Agent-Registry des eingebetteten `acpx`-Plugins für den ausgewählten ACP-Harness-Agenten.

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
- Er sourct `~/.profile`, stellt passendes CLI-Auth-Material in den Container bereit, installiert `acpx` in ein beschreibbares npm-Präfix und installiert dann die angeforderte Live-CLI (`@anthropic-ai/claude-code`, `@openai/codex` oder `@google/gemini-cli`), falls sie fehlt.
- Innerhalb von Docker setzt der Runner `OPENCLAW_LIVE_ACP_BIND_ACPX_COMMAND=$HOME/.npm-global/bin/acpx`, damit acpx Provider-env vars aus dem gesourcten Profil für die untergeordnete Harness-CLI verfügbar hält.

## Live: Codex-App-Server-Harness-Smoke

- Ziel: die Plugin-eigene Codex-Harness über die normale Gateway-
  `agent`-Methode validieren:
  - das gebündelte `codex`-Plugin laden
  - `OPENCLAW_AGENT_RUNTIME=codex` auswählen
  - einen ersten Gateway-Agent-Turn an `codex/gpt-5.4` senden
  - einen zweiten Turn an dieselbe OpenClaw-Sitzung senden und verifizieren, dass der App-Server-Thread wiederaufgenommen werden kann
  - `/codex status` und `/codex models` über denselben Gateway-Command-
    Pfad ausführen
- Test: `src/gateway/gateway-codex-harness.live.test.ts`
- Aktivierung: `OPENCLAW_LIVE_CODEX_HARNESS=1`
- Standardmodell: `codex/gpt-5.4`
- Optionale Image-Probe: `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=1`
- Optionale MCP-/Tool-Probe: `OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=1`
- Der Smoke setzt `OPENCLAW_AGENT_HARNESS_FALLBACK=none`, damit eine defekte Codex-
  Harness nicht unbemerkt durch stillen Fallback auf PI bestehen kann.
- Auth: `OPENAI_API_KEY` aus Shell/Profil sowie optional kopierte Dateien
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
- Er sourct das eingebundene `~/.profile`, übergibt `OPENAI_API_KEY`, kopiert Codex-CLI-
  Auth-Dateien, sofern vorhanden, installiert `@openai/codex` in ein beschreibbares eingebundenes npm-
  Präfix, stellt den Source-Tree bereit und führt dann nur den Live-Test der Codex-Harness aus.
- Docker aktiviert standardmäßig die Image- und MCP-/Tool-Probes. Setze
  `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=0` oder
  `OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=0`, wenn du einen engeren Debug-Lauf brauchst.
- Docker exportiert außerdem `OPENCLAW_AGENT_HARNESS_FALLBACK=none`, passend zur Live-
  Testkonfiguration, damit `openai-codex/*`- oder PI-Fallback eine Codex-Harness-
  Regression nicht verbergen kann.

### Empfohlene Live-Rezepte

Enge, explizite Allowlists sind am schnellsten und am wenigsten instabil:

- Einzelnes Modell, direkt (ohne Gateway):
  - `OPENCLAW_LIVE_MODELS="openai/gpt-5.4" pnpm test:live src/agents/models.profiles.live.test.ts`

- Einzelnes Modell, Gateway-Smoke:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Tool-Calling über mehrere Provider hinweg:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3-flash-preview,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Google-Fokus (Gemini-API-Schlüssel + Antigravity):
  - Gemini (API-Schlüssel): `OPENCLAW_LIVE_GATEWAY_MODELS="google/gemini-3-flash-preview" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`
  - Antigravity (OAuth): `OPENCLAW_LIVE_GATEWAY_MODELS="google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-pro-high" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

Hinweise:

- `google/...` verwendet die Gemini-API (API-Schlüssel).
- `google-antigravity/...` verwendet die Antigravity-OAuth-Bridge (Agent-Endpunkt im Stil von Cloud Code Assist).
- `google-gemini-cli/...` verwendet die lokale Gemini-CLI auf deiner Maschine (separate Auth + eigene Tooling-Eigenheiten).
- Gemini-API vs. Gemini-CLI:
  - API: OpenClaw ruft Googles gehostete Gemini-API über HTTP auf (API-Schlüssel / Profile-Auth); das ist, was die meisten Benutzer mit „Gemini“ meinen.
  - CLI: OpenClaw führt ein lokales `gemini`-Binary per Shell aus; es hat seine eigene Authentifizierung und kann sich anders verhalten (Streaming/Tool-Support/Versionsabweichungen).

## Live: Modellmatrix (was wir abdecken)

Es gibt keine feste „CI-Modellliste“ (Live ist Opt-in), aber dies sind die **empfohlenen** Modelle, die regelmäßig auf einer Entwickler-Maschine mit Schlüsseln abgedeckt werden sollten.

### Modernes Smoke-Set (Tool-Calling + Image)

Das ist der Lauf für „gängige Modelle“, den wir funktionsfähig halten wollen:

- OpenAI (nicht Codex): `openai/gpt-5.4` (optional: `openai/gpt-5.4-mini`)
- OpenAI Codex: `openai-codex/gpt-5.4`
- Anthropic: `anthropic/claude-opus-4-6` (oder `anthropic/claude-sonnet-4-6`)
- Google (Gemini-API): `google/gemini-3.1-pro-preview` und `google/gemini-3-flash-preview` (ältere Gemini-2.x-Modelle vermeiden)
- Google (Antigravity): `google-antigravity/claude-opus-4-6-thinking` und `google-antigravity/gemini-3-flash`
- Z.AI (GLM): `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

Gateway-Smoke mit Tools + Image ausführen:
`OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,openai-codex/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3.1-pro-preview,google/gemini-3-flash-preview,google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-flash,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

### Baseline: Tool-Calling (Read + optional Exec)

Wähle mindestens ein Modell pro Provider-Familie:

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

### Vision: Bild senden (Attachment → multimodale Nachricht)

Nimm mindestens ein bildfähiges Modell in `OPENCLAW_LIVE_GATEWAY_MODELS` auf (Claude-/Gemini-/OpenAI-Varianten mit Vision-Unterstützung usw.), um die Image-Probe zu testen.

### Aggregatoren / alternative Gateways

Wenn du aktivierte Schlüssel hast, unterstützen wir Tests auch über:

- OpenRouter: `openrouter/...` (Hunderte von Modellen; verwende `openclaw models scan`, um Kandidaten mit Tool- und Image-Fähigkeiten zu finden)
- OpenCode: `opencode/...` für Zen und `opencode-go/...` für Go (Auth über `OPENCODE_API_KEY` / `OPENCODE_ZEN_API_KEY`)

Weitere Provider, die du in die Live-Matrix aufnehmen kannst (falls du Anmeldedaten/Konfiguration hast):

- Integriert: `openai`, `openai-codex`, `anthropic`, `google`, `google-vertex`, `google-antigravity`, `google-gemini-cli`, `zai`, `openrouter`, `opencode`, `opencode-go`, `xai`, `groq`, `cerebras`, `mistral`, `github-copilot`
- Über `models.providers` (benutzerdefinierte Endpunkte): `minimax` (Cloud/API) sowie jeder OpenAI-/Anthropic-kompatible Proxy (LM Studio, vLLM, LiteLLM usw.)

Tipp: Versuche nicht, in der Dokumentation „alle Modelle“ fest zu codieren. Die maßgebliche Liste ist, was `discoverModels(...)` auf deiner Maschine zurückgibt + welche Schlüssel verfügbar sind.

## Anmeldedaten (niemals committen)

Live-Tests erkennen Anmeldedaten auf dieselbe Weise wie die CLI. Praktische Auswirkungen:

- Wenn die CLI funktioniert, sollten Live-Tests dieselben Schlüssel finden.
- Wenn ein Live-Test „no creds“ meldet, debugge auf dieselbe Weise wie bei `openclaw models list` / der Modellauswahl.

- Auth-Profile pro Agent: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` (das ist, was mit „Profile-Keys“ in den Live-Tests gemeint ist)
- Konfiguration: `~/.openclaw/openclaw.json` (oder `OPENCLAW_CONFIG_PATH`)
- Legacy-State-Verzeichnis: `~/.openclaw/credentials/` (wird bei Vorhandensein in das vorbereitete Live-Home kopiert, ist aber nicht der Hauptspeicher für Profile-Keys)
- Lokale Live-Läufe kopieren standardmäßig die aktive Konfiguration, die Dateien `auth-profiles.json` pro Agent, das Legacy-Verzeichnis `credentials/` und unterstützte externe CLI-Auth-Verzeichnisse in ein temporäres Test-Home; vorbereitete Live-Homes überspringen `workspace/` und `sandboxes/`, und Pfad-Overrides für `agents.*.workspace` / `agentDir` werden entfernt, damit Probes nicht in deinem echten Host-Workspace landen.

Wenn du dich auf env-Schlüssel verlassen willst (z. B. in deinem `~/.profile` exportiert), führe lokale Tests nach `source ~/.profile` aus oder verwende die Docker-Runner unten (sie können `~/.profile` in den Container einbinden).

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
  - Testet die gebündelten Comfy-Pfade für Bild, Video und `music_generate`
  - Überspringt jede Capability, sofern `models.providers.comfy.<capability>` nicht konfiguriert ist
  - Nützlich nach Änderungen an Comfy-Workflow-Übermittlung, Polling, Downloads oder Plugin-Registrierung

## Bildgenerierung live

- Test: `src/image-generation/runtime.live.test.ts`
- Befehl: `pnpm test:live src/image-generation/runtime.live.test.ts`
- Harness: `pnpm test:live:media image`
- Umfang:
  - Listet jedes registrierte Plugin für Bildgenerierungs-Provider auf
  - Lädt fehlende Provider-env vars vor dem Testen aus deiner Login-Shell (`~/.profile`)
  - Verwendet standardmäßig Live-/env-API-Schlüssel vor gespeicherten Auth-Profilen, damit veraltete Testschlüssel in `auth-profiles.json` echte Shell-Anmeldedaten nicht überdecken
  - Überspringt Provider ohne nutzbare Auth/Profile/Modelle
  - Führt die Standardvarianten der Bildgenerierung über die gemeinsame Runtime-Capability aus:
    - `google:flash-generate`
    - `google:pro-generate`
    - `google:pro-edit`
    - `openai:default-generate`
- Aktuell abgedeckte gebündelte Provider:
  - `openai`
  - `google`
- Optionale Eingrenzung:
  - `OPENCLAW_LIVE_IMAGE_GENERATION_PROVIDERS="openai,google"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_MODELS="openai/gpt-image-1,google/gemini-3.1-flash-image-preview"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_CASES="google:flash-generate,google:pro-edit"`
- Optionales Auth-Verhalten:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`, um Auth aus dem Profile-Store zu erzwingen und env-only-Overrides zu ignorieren

## Musikgenerierung live

- Test: `extensions/music-generation-providers.live.test.ts`
- Aktivierung: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts`
- Harness: `pnpm test:live:media music`
- Umfang:
  - Testet den gemeinsamen gebündelten Provider-Pfad für Musikgenerierung
  - Deckt derzeit Google und MiniMax ab
  - Lädt Provider-env vars vor dem Testen aus deiner Login-Shell (`~/.profile`)
  - Verwendet standardmäßig Live-/env-API-Schlüssel vor gespeicherten Auth-Profilen, damit veraltete Testschlüssel in `auth-profiles.json` echte Shell-Anmeldedaten nicht überdecken
  - Überspringt Provider ohne nutzbare Auth/Profile/Modelle
  - Führt beide deklarierten Runtime-Modi aus, wenn verfügbar:
    - `generate` mit rein promptbasierter Eingabe
    - `edit`, wenn der Provider `capabilities.edit.enabled` deklariert
  - Aktuelle Abdeckung der gemeinsamen Strecke:
    - `google`: `generate`, `edit`
    - `minimax`: `generate`
    - `comfy`: separate Comfy-Live-Datei, nicht Teil dieses gemeinsamen Sweeps
- Optionale Eingrenzung:
  - `OPENCLAW_LIVE_MUSIC_GENERATION_PROVIDERS="google,minimax"`
  - `OPENCLAW_LIVE_MUSIC_GENERATION_MODELS="google/lyria-3-clip-preview,minimax/music-2.5+"`
- Optionales Auth-Verhalten:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`, um Auth aus dem Profile-Store zu erzwingen und env-only-Overrides zu ignorieren

## Videogenerierung live

- Test: `extensions/video-generation-providers.live.test.ts`
- Aktivierung: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/video-generation-providers.live.test.ts`
- Harness: `pnpm test:live:media video`
- Umfang:
  - Testet den gemeinsamen gebündelten Provider-Pfad für Videogenerierung
  - Verwendet standardmäßig den release-sicheren Smoke-Pfad: keine FAL-Provider, eine Text-zu-Video-Anfrage pro Provider, einen einsekündigen Lobster-Prompt und eine Provider-spezifische Operationsgrenze aus `OPENCLAW_LIVE_VIDEO_GENERATION_TIMEOUT_MS` (standardmäßig `180000`)
  - Überspringt FAL standardmäßig, weil providerseitige Queue-Latenz die Release-Zeit dominieren kann; übergib `--video-providers fal` oder `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="fal"`, um ihn explizit auszuführen
  - Lädt Provider-env vars vor dem Testen aus deiner Login-Shell (`~/.profile`)
  - Verwendet standardmäßig Live-/env-API-Schlüssel vor gespeicherten Auth-Profilen, damit veraltete Testschlüssel in `auth-profiles.json` echte Shell-Anmeldedaten nicht überdecken
  - Überspringt Provider ohne nutzbare Auth/Profile/Modelle
  - Führt standardmäßig nur `generate` aus
  - Setze `OPENCLAW_LIVE_VIDEO_GENERATION_FULL_MODES=1`, um bei Verfügbarkeit auch deklarierte Transform-Modi auszuführen:
    - `imageToVideo`, wenn der Provider `capabilities.imageToVideo.enabled` deklariert und der ausgewählte Provider/das ausgewählte Modell im gemeinsamen Sweep bufferbasierte lokale Bildeingaben akzeptiert
    - `videoToVideo`, wenn der Provider `capabilities.videoToVideo.enabled` deklariert und der ausgewählte Provider/das ausgewählte Modell im gemeinsamen Sweep bufferbasierte lokale Videoeingaben akzeptiert
  - Aktuelle in der gemeinsamen Sweep deklarierte, aber übersprungene `imageToVideo`-Provider:
    - `vydra`, weil das gebündelte `veo3` nur Text unterstützt und das gebündelte `kling` eine Remote-Bild-URL erfordert
  - Provider-spezifische Vydra-Abdeckung:
    - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_VYDRA_VIDEO=1 pnpm test:live -- extensions/vydra/vydra.live.test.ts`
    - diese Datei führt standardmäßig `veo3` Text-zu-Video plus eine `kling`-Strecke aus, die eine Fixture mit Remote-Bild-URL verwendet
  - Aktuelle `videoToVideo`-Live-Abdeckung:
    - nur `runway`, wenn das ausgewählte Modell `runway/gen4_aleph` ist
  - Aktuelle in der gemeinsamen Sweep deklarierte, aber übersprungene `videoToVideo`-Provider:
    - `alibaba`, `qwen`, `xai`, weil diese Pfade derzeit Remote-Referenz-URLs mit `http(s)` / MP4 erfordern
    - `google`, weil die aktuelle gemeinsame Gemini-/Veo-Strecke lokale bufferbasierte Eingaben verwendet und dieser Pfad im gemeinsamen Sweep nicht akzeptiert wird
    - `openai`, weil die aktuelle gemeinsame Strecke keine Garantien für organisationsspezifischen Zugriff auf Video-Inpaint/Remix bietet
- Optionale Eingrenzung:
  - `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="google,openai,runway"`
  - `OPENCLAW_LIVE_VIDEO_GENERATION_MODELS="google/veo-3.1-fast-generate-preview,openai/sora-2,runway/gen4_aleph"`
  - `OPENCLAW_LIVE_VIDEO_GENERATION_SKIP_PROVIDERS=""`, um jeden Provider im Standard-Sweep einzubeziehen, einschließlich FAL
  - `OPENCLAW_LIVE_VIDEO_GENERATION_TIMEOUT_MS=60000`, um die Operationsgrenze pro Provider für einen aggressiven Smoke-Lauf zu senken
- Optionales Auth-Verhalten:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`, um Auth aus dem Profile-Store zu erzwingen und env-only-Overrides zu ignorieren

## Media-Live-Harness

- Befehl: `pnpm test:live:media`
- Zweck:
  - Führt die gemeinsamen Live-Suites für Bild, Musik und Video über einen repo-nativen Entrypoint aus
  - Lädt fehlende Provider-env vars automatisch aus `~/.profile`
  - Grenzt jede Suite standardmäßig automatisch auf Provider ein, die aktuell nutzbare Auth haben
  - Verwendet erneut `scripts/test-live.mjs`, sodass Heartbeat- und Quiet-Mode-Verhalten konsistent bleiben
- Beispiele:
  - `pnpm test:live:media`
  - `pnpm test:live:media image video --providers openai,google,minimax`
  - `pnpm test:live:media video --video-providers openai,runway --all-providers`
  - `pnpm test:live:media music --quiet`

## Docker-Runner (optionale „funktioniert unter Linux“-Prüfungen)

Diese Docker-Runner sind in zwei Gruppen aufgeteilt:

- Live-Modell-Runner: `test:docker:live-models` und `test:docker:live-gateway` führen nur ihre jeweils passende Live-Datei für Profile-Keys im Repo-Docker-Image aus (`src/agents/models.profiles.live.test.ts` und `src/gateway/gateway-models.profiles.live.test.ts`), wobei dein lokales Konfigurationsverzeichnis und dein Workspace eingebunden werden (und `~/.profile` gesourct wird, falls eingebunden). Die passenden lokalen Entrypoints sind `test:live:models-profiles` und `test:live:gateway-profiles`.
- Docker-Live-Runner verwenden standardmäßig eine kleinere Smoke-Obergrenze, damit ein vollständiger Docker-Sweep praktikabel bleibt:
  `test:docker:live-models` verwendet standardmäßig `OPENCLAW_LIVE_MAX_MODELS=12`, und
  `test:docker:live-gateway` verwendet standardmäßig `OPENCLAW_LIVE_GATEWAY_SMOKE=1`,
  `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=8`,
  `OPENCLAW_LIVE_GATEWAY_STEP_TIMEOUT_MS=45000` und
  `OPENCLAW_LIVE_GATEWAY_MODEL_TIMEOUT_MS=90000`. Überschreibe diese env vars, wenn du
  ausdrücklich den größeren vollständigen Scan möchtest.
- `test:docker:all` baut das Live-Docker-Image einmal über `test:docker:live-build` und verwendet es dann für die beiden Live-Docker-Strecken erneut.
- Container-Smoke-Runner: `test:docker:openwebui`, `test:docker:onboard`, `test:docker:gateway-network`, `test:docker:mcp-channels` und `test:docker:plugins` starten einen oder mehrere echte Container und verifizieren Integrationspfade auf höherer Ebene.

Die Docker-Runner für Live-Modelle binden außerdem nur die benötigten CLI-Auth-Homes ein (oder alle unterstützten, wenn der Lauf nicht eingegrenzt ist) und kopieren sie dann vor dem Lauf in das Container-Home, damit externes CLI-OAuth Tokens aktualisieren kann, ohne den Auth-Store des Hosts zu verändern:

- Direkte Modelle: `pnpm test:docker:live-models` (Skript: `scripts/test-live-models-docker.sh`)
- ACP-Bind-Smoke: `pnpm test:docker:live-acp-bind` (Skript: `scripts/test-live-acp-bind-docker.sh`)
- CLI-Backend-Smoke: `pnpm test:docker:live-cli-backend` (Skript: `scripts/test-live-cli-backend-docker.sh`)
- Codex-App-Server-Harness-Smoke: `pnpm test:docker:live-codex-harness` (Skript: `scripts/test-live-codex-harness-docker.sh`)
- Gateway + Dev-Agent: `pnpm test:docker:live-gateway` (Skript: `scripts/test-live-gateway-models-docker.sh`)
- Open-WebUI-Live-Smoke: `pnpm test:docker:openwebui` (Skript: `scripts/e2e/openwebui-docker.sh`)
- Onboarding-Assistent (TTY, vollständiges Scaffolding): `pnpm test:docker:onboard` (Skript: `scripts/e2e/onboard-docker.sh`)
- Gateway-Networking (zwei Container, WS-Auth + Health): `pnpm test:docker:gateway-network` (Skript: `scripts/e2e/gateway-network-docker.sh`)
- MCP-Channel-Bridge (vorbereiteter Gateway + stdio-Bridge + roher Claude-Benachrichtigungsframe-Smoke): `pnpm test:docker:mcp-channels` (Skript: `scripts/e2e/mcp-channels-docker.sh`)
- Plugins (Installations-Smoke + `/plugin`-Alias + Neustartsemantik des Claude-Bundles): `pnpm test:docker:plugins` (Skript: `scripts/e2e/plugins-docker.sh`)

Die Docker-Runner für Live-Modelle binden außerdem den aktuellen Checkout schreibgeschützt ein und
stellen ihn in ein temporäres Workdir im Container bereit. So bleibt das Runtime-
Image schlank, während Vitest trotzdem gegen deinen exakten lokalen Source-/Konfigurationsstand läuft.
Der Bereitstellungsschritt überspringt große lokale Caches und App-Build-Ausgaben wie
`.pnpm-store`, `.worktrees`, `__openclaw_vitest__` und app-lokale `.build`- oder
Gradle-Ausgabeordner, damit Docker-Live-Läufe nicht Minuten mit dem Kopieren
maschinenspezifischer Artifacts verbringen.
Sie setzen außerdem `OPENCLAW_SKIP_CHANNELS=1`, damit Gateway-Live-Probes keine
echten Telegram-/Discord-/usw.-Channel-Worker im Container starten.
`test:docker:live-models` führt weiterhin `pnpm test:live` aus; reiche daher auch
`OPENCLAW_LIVE_GATEWAY_*` durch, wenn du die Gateway-Live-Abdeckung in dieser Docker-Strecke
eingrenzen oder ausschließen musst.
`test:docker:openwebui` ist ein höherstufiger Kompatibilitäts-Smoke: Er startet einen
OpenClaw-Gateway-Container mit aktivierten OpenAI-kompatiblen HTTP-Endpunkten,
startet einen festgepinnte Open-WebUI-Container gegen dieses Gateway, meldet sich über
Open WebUI an, verifiziert, dass `/api/models` `openclaw/default` bereitstellt, und sendet dann
eine echte Chat-Anfrage über den Proxy `/api/chat/completions` von Open WebUI.
Der erste Lauf kann spürbar langsamer sein, weil Docker möglicherweise zuerst das
Open-WebUI-Image ziehen muss und Open WebUI sein eigenes Cold-Start-Setup abschließen muss.
Diese Strecke erwartet einen verwendbaren Live-Modellschlüssel, und `OPENCLAW_PROFILE_FILE`
(`~/.profile` standardmäßig) ist der primäre Weg, ihn in Docker-Läufen bereitzustellen.
Erfolgreiche Läufe geben eine kleine JSON-Payload wie `{ "ok": true, "model":
"openclaw/default", ... }` aus.
`test:docker:mcp-channels` ist absichtlich deterministisch und benötigt kein
echtes Telegram-, Discord- oder iMessage-Konto. Es startet einen vorbereiteten Gateway-
Container, startet einen zweiten Container, der `openclaw mcp serve` ausführt, und
verifiziert dann geroutete Konversationserkennung, Transkriptlesevorgänge, Attachment-Metadaten,
Verhalten der Live-Event-Queue, Routing ausgehender Sends sowie Claude-artige Channel- +
Berechtigungsbenachrichtigungen über die echte stdio-MCP-Bridge. Die Benachrichtigungsprüfung
untersucht direkt die rohen stdio-MCP-Frames, sodass der Smoke validiert, was die
Bridge tatsächlich ausgibt, und nicht nur, was ein bestimmtes Client-SDK gerade sichtbar macht.

Manueller ACP-Smoketest für Threads in natürlicher Sprache (nicht CI):

- `bun scripts/dev/discord-acp-plain-language-smoke.ts --channel <discord-channel-id> ...`
- Behalte dieses Skript für Regressions-/Debug-Workflows bei. Es kann für die Validierung des ACP-Thread-Routings erneut benötigt werden, also nicht löschen.

Nützliche env vars:

- `OPENCLAW_CONFIG_DIR=...` (Standard: `~/.openclaw`), eingebunden nach `/home/node/.openclaw`
- `OPENCLAW_WORKSPACE_DIR=...` (Standard: `~/.openclaw/workspace`), eingebunden nach `/home/node/.openclaw/workspace`
- `OPENCLAW_PROFILE_FILE=...` (Standard: `~/.profile`), eingebunden nach `/home/node/.profile` und vor dem Ausführen der Tests gesourct
- `OPENCLAW_DOCKER_PROFILE_ENV_ONLY=1`, um nur env vars zu prüfen, die aus `OPENCLAW_PROFILE_FILE` gesourct wurden, unter Verwendung temporärer Konfigurations-/Workspace-Verzeichnisse und ohne externe CLI-Auth-Mounts
- `OPENCLAW_DOCKER_CLI_TOOLS_DIR=...` (Standard: `~/.cache/openclaw/docker-cli-tools`), eingebunden nach `/home/node/.npm-global` für gecachte CLI-Installationen in Docker
- Externe CLI-Auth-Verzeichnisse/-Dateien unter `$HOME` werden schreibgeschützt unter `/host-auth...` eingebunden und dann nach `/home/node/...` kopiert, bevor die Tests starten
  - Standardverzeichnisse: `.minimax`
  - Standarddateien: `~/.codex/auth.json`, `~/.codex/config.toml`, `.claude.json`, `~/.claude/.credentials.json`, `~/.claude/settings.json`, `~/.claude/settings.local.json`
  - Eingegrenzte Provider-Läufe binden nur die benötigten Verzeichnisse/Dateien ein, die aus `OPENCLAW_LIVE_PROVIDERS` / `OPENCLAW_LIVE_GATEWAY_PROVIDERS` abgeleitet werden
  - Manuelle Überschreibung mit `OPENCLAW_DOCKER_AUTH_DIRS=all`, `OPENCLAW_DOCKER_AUTH_DIRS=none` oder einer Komma-Liste wie `OPENCLAW_DOCKER_AUTH_DIRS=.claude,.codex`
- `OPENCLAW_LIVE_GATEWAY_MODELS=...` / `OPENCLAW_LIVE_MODELS=...`, um den Lauf einzugrenzen
- `OPENCLAW_LIVE_GATEWAY_PROVIDERS=...` / `OPENCLAW_LIVE_PROVIDERS=...`, um Provider im Container zu filtern
- `OPENCLAW_SKIP_DOCKER_BUILD=1`, um ein vorhandenes Image `openclaw:local-live` für erneute Läufe zu verwenden, die keinen Neubau benötigen
- `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`, um sicherzustellen, dass Anmeldedaten aus dem Profile-Store kommen (nicht aus env)
- `OPENCLAW_OPENWEBUI_MODEL=...`, um das Modell auszuwählen, das das Gateway für den Open-WebUI-Smoke bereitstellt
- `OPENCLAW_OPENWEBUI_PROMPT=...`, um den für den Open-WebUI-Smoke verwendeten Nonce-Prüfprompt zu überschreiben
- `OPENWEBUI_IMAGE=...`, um das festgepinnte Open-WebUI-Image-Tag zu überschreiben

## Doku-Sanity

Führe nach Doku-Bearbeitungen Doku-Prüfungen aus: `pnpm check:docs`.
Führe die vollständige Mintlify-Anchor-Validierung aus, wenn du auch Heading-Prüfungen innerhalb der Seite brauchst: `pnpm docs:check-links:anchors`.

## Offline-Regression (CI-sicher)

Das sind Regressionen mit „echter Pipeline“, aber ohne echte Provider:

- Gateway-Tool-Calling (Mock-OpenAI, echtes Gateway + Agent-Loop): `src/gateway/gateway.test.ts` (Fall: "runs a mock OpenAI tool call end-to-end via gateway agent loop")
- Gateway-Assistent (WS `wizard.start`/`wizard.next`, schreibt Konfiguration + Auth erzwungen): `src/gateway/gateway.test.ts` (Fall: "runs wizard over ws and writes auth token config")

## Evals zur Agent-Zuverlässigkeit (Skills)

Wir haben bereits einige CI-sichere Tests, die sich wie „Evals zur Agent-Zuverlässigkeit“ verhalten:

- Mock-Tool-Calling über den echten Gateway- + Agent-Loop (`src/gateway/gateway.test.ts`).
- End-to-End-Assistenten-Flows, die Sitzungs-Wiring und Konfigurationseffekte validieren (`src/gateway/gateway.test.ts`).

Was für Skills noch fehlt (siehe [Skills](/de/tools/skills)):

- **Entscheidungsfindung:** Wenn Skills im Prompt aufgeführt sind, wählt der Agent dann den richtigen Skill aus (oder vermeidet irrelevante)?
- **Compliance:** Liest der Agent vor der Verwendung `SKILL.md` und befolgt die erforderlichen Schritte/Args?
- **Workflow-Verträge:** Multi-Turn-Szenarien, die Tool-Reihenfolge, Sitzungsverlauf über mehrere Turns und Sandbox-Grenzen prüfen.

Zukünftige Evals sollten zuerst deterministisch bleiben:

- Ein Szenario-Runner mit Mock-Providern, der Tool-Calls + Reihenfolge, Skill-Dateilesen und Sitzungs-Wiring prüft.
- Eine kleine Suite mit Skill-fokussierten Szenarien (verwenden vs. vermeiden, Gating, Prompt-Injection).
- Optionale Live-Evals (Opt-in, env-gated) erst dann, wenn die CI-sichere Suite vorhanden ist.

## Contract-Tests (Plugin- und Channel-Form)

Contract-Tests verifizieren, dass jedes registrierte Plugin und jeder registrierte Channel seinem
Schnittstellenvertrag entspricht. Sie iterieren über alle entdeckten Plugins und führen eine Suite von
Prüfungen für Form und Verhalten aus. Die standardmäßige Unit-Strecke `pnpm test`
überspringt diese gemeinsamen Seam- und Smoke-Dateien absichtlich; führe die Contract-Befehle explizit aus,
wenn du gemeinsame Channel- oder Provider-Oberflächen anfasst.

### Befehle

- Alle Contracts: `pnpm test:contracts`
- Nur Channel-Contracts: `pnpm test:contracts:channels`
- Nur Provider-Contracts: `pnpm test:contracts:plugins`

### Channel-Contracts

Liegen unter `src/channels/plugins/contracts/*.contract.test.ts`:

- **plugin** - Grundlegende Plugin-Form (ID, Name, Capabilities)
- **setup** - Vertrag für den Setup-Assistenten
- **session-binding** - Verhalten der Sitzungsbindung
- **outbound-payload** - Struktur der Message-Payload
- **inbound** - Verarbeitung eingehender Nachrichten
- **actions** - Channel-Action-Handler
- **threading** - Verarbeitung von Thread-IDs
- **directory** - API für Verzeichnis/Roster
- **group-policy** - Durchsetzung der Gruppenrichtlinie

### Provider-Status-Contracts

Liegen unter `src/plugins/contracts/*.contract.test.ts`.

- **status** - Channel-Status-Probes
- **registry** - Form der Plugin-Registry

### Provider-Contracts

Liegen unter `src/plugins/contracts/*.contract.test.ts`:

- **auth** - Auth-Flow-Vertrag
- **auth-choice** - Auth-Auswahl
- **catalog** - API des Modellkatalogs
- **discovery** - Plugin-Discovery
- **loader** - Plugin-Laden
- **runtime** - Provider-Runtime
- **shape** - Plugin-Form/Schnittstelle
- **wizard** - Setup-Assistent

### Wann ausführen

- Nach Änderungen an `plugin-sdk`-Exports oder Subpfaden
- Nach dem Hinzufügen oder Ändern eines Channel- oder Provider-Plugins
- Nach Refactorings an Plugin-Registrierung oder -Discovery

Contract-Tests laufen in CI und benötigen keine echten API-Schlüssel.

## Regressionen hinzufügen (Leitfaden)

Wenn du ein in Live entdecktes Provider-/Modellproblem behebst:

- Füge möglichst eine CI-sichere Regression hinzu (Mock-/Stub-Provider oder erfasse die exakte Transformation der Request-Form)
- Wenn das Problem von Natur aus nur live testbar ist (Rate Limits, Auth-Richtlinien), halte den Live-Test eng und aktiviere ihn per Opt-in über env vars
- Bevorzuge die kleinste Ebene, die den Fehler erkennt:
  - Fehler bei Provider-Request-Konvertierung/Replay → Test für direkte Modelle
  - Fehler bei Gateway-Sitzung/Verlauf/Tool-Pipeline → Gateway-Live-Smoke oder CI-sicherer Gateway-Mock-Test
- Guardrail für SecretRef-Traversierung:
  - `src/secrets/exec-secret-ref-id-parity.test.ts` leitet aus den Registry-Metadaten (`listSecretTargetRegistryEntries()`) ein gesampeltes Ziel pro SecretRef-Klasse ab und prüft dann, dass Exec-IDs von Traversierungssegmenten zurückgewiesen werden.
  - Wenn du in `src/secrets/target-registry-data.ts` eine neue `includeInPlan`-SecretRef-Zielfamilie hinzufügst, aktualisiere `classifyTargetClass` in diesem Test. Der Test schlägt absichtlich bei nicht klassifizierten Ziel-IDs fehl, damit neue Klassen nicht stillschweigend ausgelassen werden können.
