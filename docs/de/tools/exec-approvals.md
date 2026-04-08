---
read_when:
    - Konfigurieren von Exec-Freigaben oder Zulassungslisten
    - Implementierung der Exec-Freigabe-UX in der macOS-App
    - Prüfung von Prompts für Sandbox-Escapes und deren Auswirkungen
summary: Exec-Freigaben, Zulassungslisten und Prompts für Sandbox-Escapes
title: Exec-Freigaben
x-i18n:
    generated_at: "2026-04-08T02:20:02Z"
    model: gpt-5.4
    provider: openai
    source_hash: 6041929185bab051ad873cc4822288cb7d6f0470e19e7ae7a16b70f76dfc2cd9
    source_path: tools/exec-approvals.md
    workflow: 15
---

# Exec-Freigaben

Exec-Freigaben sind die **Schutzmaßnahme der Begleit-App / des Node-Hosts**, damit ein sandboxed Agent
Befehle auf einem echten Host (`gateway` oder `node`) ausführen kann. Betrachte das wie eine Sicherheitsverriegelung:
Befehle sind nur erlaubt, wenn Richtlinie + Zulassungsliste + (optionale) Benutzerfreigabe alle zustimmen.
Exec-Freigaben gelten **zusätzlich** zu Tool-Richtlinien und Elevated-Gating (außer Elevated ist auf `full` gesetzt; dann werden Freigaben übersprungen).
Die wirksame Richtlinie ist die **strengere** von `tools.exec.*` und den Freigabe-Standards. Wenn ein Freigabefeld fehlt, wird der Wert aus `tools.exec` verwendet.
Host-Exec verwendet auch den lokalen Freigabestatus auf diesem Rechner. Ein hostlokales
`ask: "always"` in `~/.openclaw/exec-approvals.json` sorgt weiterhin für Prompts, selbst wenn
Sitzungs- oder Konfigurationsstandards `ask: "on-miss"` anfordern.
Verwende `openclaw approvals get`, `openclaw approvals get --gateway` oder
`openclaw approvals get --node <id|name|ip>`, um die angeforderte Richtlinie,
die Quellen der Host-Richtlinie und das wirksame Ergebnis zu prüfen.

Wenn die UI der Begleit-App **nicht verfügbar** ist, wird jede Anfrage, die einen Prompt erfordert,
durch den **Ask-Fallback** aufgelöst (Standard: deny).

Native Chat-Freigabeclients können auf der ausstehenden Freigabenachricht auch kanalspezifische Bedienelemente bereitstellen. Zum Beispiel kann Matrix Reaktions-Shortcuts auf dem
Freigabe-Prompt setzen (`✅` einmal erlauben, `❌` verweigern und `♾️` immer erlauben, wenn verfügbar),
während die `/approve ...`-Befehle in der Nachricht als Fallback erhalten bleiben.

## Wo es gilt

Exec-Freigaben werden lokal auf dem Ausführungshost erzwungen:

- **gateway host** → `openclaw`-Prozess auf dem Gateway-Rechner
- **node host** → Node-Runner (macOS-Begleit-App oder headless Node-Host)

Hinweis zum Vertrauensmodell:

- Gateway-authentifizierte Aufrufer sind vertrauenswürdige Operatoren für dieses Gateway.
- Gepaarte Nodes erweitern diese vertrauenswürdige Operator-Fähigkeit auf den Node-Host.
- Exec-Freigaben verringern das Risiko versehentlicher Ausführung, sind aber keine Per-User-Authentifizierungsgrenze.
- Freigegebene Node-Host-Läufe binden den kanonischen Ausführungskontext: kanonisches cwd, exaktes argv, env-
  Binding, wenn vorhanden, und gepinnter Pfad zur ausführbaren Datei, wenn anwendbar.
- Bei Shell-Skripten und direkten Interpreter-/Runtime-Dateiaufrufen versucht OpenClaw außerdem,
  genau einen konkreten lokalen Dateiparameter zu binden. Wenn sich diese gebundene Datei nach der Freigabe,
  aber vor der Ausführung ändert, wird der Lauf verweigert, statt abweichenden Inhalt auszuführen.
- Dieses Datei-Binding ist absichtlich best-effort und kein vollständiges semantisches Modell jedes
  Interpreter-/Runtime-Loader-Pfads. Wenn der Freigabemodus nicht genau eine konkrete lokale
  Datei identifizieren kann, die gebunden werden kann, verweigert er die Erstellung eines freigabegestützten Laufs,
  statt vollständige Abdeckung nur vorzutäuschen.

macOS-Aufteilung:

- **node host service** leitet `system.run` über lokales IPC an die **macOS-App** weiter.
- **macOS-App** erzwingt Freigaben + führt den Befehl im UI-Kontext aus.

## Einstellungen und Speicherung

Freigaben liegen in einer lokalen JSON-Datei auf dem Ausführungshost:

`~/.openclaw/exec-approvals.json`

Beispielschema:

```json
{
  "version": 1,
  "socket": {
    "path": "~/.openclaw/exec-approvals.sock",
    "token": "base64url-token"
  },
  "defaults": {
    "security": "deny",
    "ask": "on-miss",
    "askFallback": "deny",
    "autoAllowSkills": false
  },
  "agents": {
    "main": {
      "security": "allowlist",
      "ask": "on-miss",
      "askFallback": "deny",
      "autoAllowSkills": true,
      "allowlist": [
        {
          "id": "B0C8C0B3-2C2D-4F8A-9A3C-5A4B3C2D1E0F",
          "pattern": "~/Projects/**/bin/rg",
          "lastUsedAt": 1737150000000,
          "lastUsedCommand": "rg -n TODO",
          "lastResolvedPath": "/Users/user/Projects/.../bin/rg"
        }
      ]
    }
  }
}
```

## Modus ohne Freigaben „YOLO“

Wenn Host-Exec ohne Freigabe-Prompts laufen soll, musst du **beide** Richtlinienebenen öffnen:

- angeforderte Exec-Richtlinie in der OpenClaw-Konfiguration (`tools.exec.*`)
- hostlokale Freigaberichtlinie in `~/.openclaw/exec-approvals.json`

Das ist jetzt das Standardverhalten des Hosts, sofern du es nicht ausdrücklich verschärfst:

- `tools.exec.security`: `full` auf `gateway`/`node`
- `tools.exec.ask`: `off`
- Host-`askFallback`: `full`

Wichtiger Unterschied:

- `tools.exec.host=auto` wählt aus, wo Exec ausgeführt wird: in der Sandbox, wenn verfügbar, sonst auf dem Gateway.
- YOLO wählt, wie Host-Exec freigegeben wird: `security=full` plus `ask=off`.
- Im YOLO-Modus fügt OpenClaw keine separate heuristische Freigabehürde für Befehlsverschleierung zusätzlich zur konfigurierten Host-Exec-Richtlinie hinzu.
- `auto` macht Gateway-Routing nicht zu einer freien Überschreibung aus einer sandboxed Sitzung heraus. Eine pro Aufruf gesetzte Anfrage `host=node` ist von `auto` aus erlaubt, und `host=gateway` ist von `auto` aus nur erlaubt, wenn keine Sandbox-Runtime aktiv ist. Wenn du einen stabilen Standard ohne `auto` möchtest, setze `tools.exec.host` oder verwende `/exec host=...` ausdrücklich.

Wenn du eine konservativere Konfiguration möchtest, ziehe eine der Ebenen wieder auf `allowlist` / `on-miss`
oder `deny` an.

Persistente Einrichtung „nie nachfragen“ für den Gateway-Host:

```bash
openclaw config set tools.exec.host gateway
openclaw config set tools.exec.security full
openclaw config set tools.exec.ask off
openclaw gateway restart
```

Dann die Freigabedatei des Hosts passend setzen:

```bash
openclaw approvals set --stdin <<'EOF'
{
  version: 1,
  defaults: {
    security: "full",
    ask: "off",
    askFallback: "full"
  }
}
EOF
```

Für einen Node-Host dieselbe Freigabedatei stattdessen auf diesem Node anwenden:

```bash
openclaw approvals set --node <id|name|ip> --stdin <<'EOF'
{
  version: 1,
  defaults: {
    security: "full",
    ask: "off",
    askFallback: "full"
  }
}
EOF
```

Nur-Sitzung-Abkürzung:

- `/exec security=full ask=off` ändert nur die aktuelle Sitzung.
- `/elevated full` ist eine Break-Glass-Abkürzung, die für diese Sitzung auch Exec-Freigaben überspringt.

Wenn die Freigabedatei des Hosts strenger bleibt als die Konfiguration, gewinnt weiterhin die strengere Host-Richtlinie.

## Richtlinienoptionen

### Security (`exec.security`)

- **deny**: alle Host-Exec-Anfragen blockieren.
- **allowlist**: nur Befehle aus der Zulassungsliste erlauben.
- **full**: alles erlauben (entspricht elevated).

### Ask (`exec.ask`)

- **off**: niemals nachfragen.
- **on-miss**: nur nachfragen, wenn keine Übereinstimmung in der Zulassungsliste vorliegt.
- **always**: bei jedem Befehl nachfragen.
- Dauerhaftes Vertrauen per `allow-always` unterdrückt Prompts nicht, wenn der wirksame Ask-Modus `always` ist

### Ask-Fallback (`askFallback`)

Wenn ein Prompt erforderlich ist, aber keine UI erreichbar ist, entscheidet der Fallback:

- **deny**: blockieren.
- **allowlist**: nur erlauben, wenn die Zulassungsliste übereinstimmt.
- **full**: erlauben.

### Härtung für Inline-Interpreter-Eval (`tools.exec.strictInlineEval`)

Wenn `tools.exec.strictInlineEval=true`, behandelt OpenClaw Formen für Inline-Codeauswertung als nur-freigabefähig, selbst wenn die Interpreter-Binärdatei selbst auf der Zulassungsliste steht.

Beispiele:

- `python -c`
- `node -e`, `node --eval`, `node -p`
- `ruby -e`
- `perl -e`, `perl -E`
- `php -r`
- `lua -e`
- `osascript -e`

Das ist Defense-in-Depth für Interpreter-Loader, die sich nicht sauber auf einen stabilen Dateiparameter abbilden lassen. Im strikten Modus gilt:

- diese Befehle benötigen weiterhin eine explizite Freigabe;
- `allow-always` speichert für sie nicht automatisch neue Zulassungslisteneinträge.

## Zulassungsliste (pro Agent)

Zulassungslisten sind **pro Agent**. Wenn mehrere Agenten existieren, wechselst du in der macOS-App,
welchen Agenten du bearbeitest. Muster sind **case-insensitive Glob-Matches**.
Muster sollten sich zu **Pfaden von Binärdateien** auflösen (Einträge nur mit Basename werden ignoriert).
Legacy-Einträge unter `agents.default` werden beim Laden nach `agents.main` migriert.
Shell-Ketten wie `echo ok && pwd` erfordern weiterhin, dass jedes Segment auf oberster Ebene die Regeln der Zulassungsliste erfüllt.

Beispiele:

- `~/Projects/**/bin/peekaboo`
- `~/.local/bin/*`
- `/opt/homebrew/bin/rg`

Jeder Eintrag in der Zulassungsliste verfolgt:

- **id** stabile UUID für die UI-Identität (optional)
- **last used** Zeitstempel
- **last used command**
- **last resolved path**

## Skill-CLIs automatisch zulassen

Wenn **Auto-allow skill CLIs** aktiviert ist, werden ausführbare Dateien, auf die bekannte Skills verweisen,
auf Nodes (macOS-Node oder headless Node-Host) so behandelt, als stünden sie auf der Zulassungsliste. Dazu wird
`skills.bins` über Gateway-RPC verwendet, um die Liste der Skill-Binaries abzurufen. Deaktiviere dies, wenn du strikte manuelle Zulassungslisten willst.

Wichtige Hinweise zum Vertrauensmodell:

- Dies ist eine **implizite Convenience-Zulassungsliste**, getrennt von manuellen Pfad-Einträgen in der Zulassungsliste.
- Sie ist für vertrauenswürdige Operator-Umgebungen gedacht, in denen Gateway und Node in derselben Vertrauensgrenze liegen.
- Wenn du strikt explizites Vertrauen benötigst, lasse `autoAllowSkills: false` gesetzt und verwende nur manuelle Pfad-Einträge in der Zulassungsliste.

## Safe Bins (nur stdin)

`tools.exec.safeBins` definiert eine kleine Liste von **nur-stdin**-Binärdateien (zum Beispiel `cut`),
die im Allowlist-Modus **ohne** explizite Einträge in der Zulassungsliste laufen dürfen. Safe Bins lehnen
positionale Dateiparameter und pfadartige Tokens ab, sodass sie nur auf dem eingehenden Stream arbeiten können.
Betrachte dies als schmalen Fast Path für Stream-Filter, nicht als allgemeine Vertrauensliste.
Füge **keine** Interpreter- oder Runtime-Binärdateien (zum Beispiel `python3`, `node`, `ruby`, `bash`, `sh`, `zsh`) zu `safeBins` hinzu.
Wenn ein Befehl Code auswerten, Unterbefehle ausführen oder absichtlich Dateien lesen kann, bevorzuge explizite Einträge in der Zulassungsliste und lasse Freigabe-Prompts aktiviert.
Benutzerdefinierte Safe Bins müssen ein explizites Profil unter `tools.exec.safeBinProfiles.<bin>` definieren.
Die Validierung erfolgt deterministisch nur anhand der Form von argv (keine Host-Dateisystem-Existenzprüfungen), wodurch
Oracle-Verhalten durch Unterschiede bei Allow/Deny für vorhandene Dateien verhindert wird.
Dateiorientierte Optionen werden für Standard-Safe-Bins abgelehnt (zum Beispiel `sort -o`, `sort --output`,
`sort --files0-from`, `sort --compress-program`, `sort --random-source`,
`sort --temporary-directory`/`-T`, `wc --files0-from`, `jq -f/--from-file`,
`grep -f/--file`).
Safe Bins erzwingen außerdem explizite Flag-Richtlinien pro Binärdatei für Optionen, die das Nur-stdin-
Verhalten aufbrechen (zum Beispiel `sort -o/--output/--compress-program` und rekursive Flags bei grep).
Long-Options werden im Safe-Bin-Modus fail-closed validiert: unbekannte Flags und mehrdeutige
Abkürzungen werden abgelehnt.
Abgelehnte Flags nach Safe-Bin-Profil:

[//]: # "SAFE_BIN_DENIED_FLAGS:START"

- `grep`: `--dereference-recursive`, `--directories`, `--exclude-from`, `--file`, `--recursive`, `-R`, `-d`, `-f`, `-r`
- `jq`: `--argfile`, `--from-file`, `--library-path`, `--rawfile`, `--slurpfile`, `-L`, `-f`
- `sort`: `--compress-program`, `--files0-from`, `--output`, `--random-source`, `--temporary-directory`, `-T`, `-o`
- `wc`: `--files0-from`

[//]: # "SAFE_BIN_DENIED_FLAGS:END"

Safe Bins erzwingen außerdem, dass argv-Tokens zur Ausführungszeit als **literaler Text** behandelt werden (kein Globbing
und keine Expansion von `$VARS`) für Nur-stdin-Segmente, sodass Muster wie `*` oder `$HOME/...` nicht
zum Einschmuggeln von Dateilesen verwendet werden können.
Safe Bins müssen außerdem aus vertrauenswürdigen Verzeichnissen für Binärdateien aufgelöst werden (System-Standards plus optionale
`tools.exec.safeBinTrustedDirs`). `PATH`-Einträge werden nie automatisch als vertrauenswürdig behandelt.
Die Standardverzeichnisse für vertrauenswürdige Safe Bins sind absichtlich minimal: `/bin`, `/usr/bin`.
Wenn sich deine Safe-Bin-Binärdatei in Verzeichnissen von Paketmanagern oder Benutzern befindet (zum Beispiel
`/opt/homebrew/bin`, `/usr/local/bin`, `/opt/local/bin`, `/snap/bin`), füge sie ausdrücklich
zu `tools.exec.safeBinTrustedDirs` hinzu.
Shell-Ketten und Umleitungen werden im Allowlist-Modus nicht automatisch erlaubt.

Shell-Ketten (`&&`, `||`, `;`) sind erlaubt, wenn jedes Segment auf oberster Ebene die Zulassungsliste erfüllt
(einschließlich Safe Bins oder automatischer Skill-Zulassung). Umleitungen bleiben im Allowlist-Modus nicht unterstützt.
Command Substitution (`$()` / Backticks) wird während des Parsens der Zulassungsliste abgelehnt, auch innerhalb
doppelter Anführungszeichen; verwende einfache Anführungszeichen, wenn du wörtlichen `$()`-Text brauchst.
Bei Freigaben über die macOS-Begleit-App wird roher Shell-Text mit Shell-Steuer- oder Expansionssyntax
(`&&`, `||`, `;`, `|`, `` ` ``, `$`, `<`, `>`, `(`, `)`) als Miss der Zulassungsliste behandelt, sofern
die Shell-Binärdatei selbst nicht auf der Zulassungsliste steht.
Für Shell-Wrapper (`bash|sh|zsh ... -c/-lc`) werden anfragebezogene env-Überschreibungen auf eine kleine explizite
Zulassungsliste reduziert (`TERM`, `LANG`, `LC_*`, `COLORTERM`, `NO_COLOR`, `FORCE_COLOR`).
Für `allow-always`-Entscheidungen im Allowlist-Modus
werden bei bekannten Dispatch-Wrappern
(`env`, `nice`, `nohup`, `stdbuf`, `timeout`) die Pfade der inneren ausführbaren Dateien statt der Wrapper-
Pfade persistent gespeichert. Shell-Multiplexer (`busybox`, `toybox`) werden auch für Shell-Applets (`sh`, `ash`,
usw.) entpackt, sodass die inneren ausführbaren Dateien statt der Multiplexer-Binärdateien gespeichert werden. Wenn ein Wrapper oder
Multiplexer nicht sicher entpackt werden kann, wird automatisch kein Eintrag in der Zulassungsliste gespeichert.
Wenn du Interpreter wie `python3` oder `node` auf die Zulassungsliste setzt, bevorzuge `tools.exec.strictInlineEval=true`, damit Inline-Eval weiterhin eine explizite Freigabe erfordert. Im strikten Modus kann `allow-always` weiterhin harmlose Interpreter-/Skriptaufrufe speichern, aber Träger für Inline-Eval werden nicht automatisch gespeichert.

Standard-Safe-Bins:

[//]: # "SAFE_BIN_DEFAULTS:START"

`cut`, `uniq`, `head`, `tail`, `tr`, `wc`

[//]: # "SAFE_BIN_DEFAULTS:END"

`grep` und `sort` stehen nicht in der Standardliste. Wenn du sie aktivierst, behalte explizite Einträge in der Zulassungsliste für
ihre Nicht-stdin-Workflows.
Für `grep` im Safe-Bin-Modus gib das Muster mit `-e`/`--regexp` an; die positionale Form für Muster wird
abgelehnt, damit Dateiparameter nicht als mehrdeutige positionale Argumente eingeschmuggelt werden können.

### Safe Bins versus Zulassungsliste

| Thema            | `tools.exec.safeBins`                                  | Zulassungsliste (`exec-approvals.json`)                      |
| ---------------- | ------------------------------------------------------ | ------------------------------------------------------------ |
| Ziel             | Schmale stdin-Filter automatisch erlauben              | Bestimmten ausführbaren Dateien explizit vertrauen           |
| Match-Typ        | Name der ausführbaren Datei + argv-Richtlinie für Safe Bins | Glob-Muster für den aufgelösten Pfad der ausführbaren Datei  |
| Umfang der Argumente | Eingeschränkt durch Safe-Bin-Profil und Regeln für literale Tokens | Nur Pfad-Match; für Argumente bist du ansonsten selbst verantwortlich |
| Typische Beispiele | `head`, `tail`, `tr`, `wc`                           | `jq`, `python3`, `node`, `ffmpeg`, benutzerdefinierte CLIs   |
| Am besten geeignet | Texttransformationen mit geringem Risiko in Pipelines | Jedes Tool mit breiterem Verhalten oder Nebeneffekten        |

Ort der Konfiguration:

- `safeBins` stammt aus der Konfiguration (`tools.exec.safeBins` oder pro Agent `agents.list[].tools.exec.safeBins`).
- `safeBinTrustedDirs` stammt aus der Konfiguration (`tools.exec.safeBinTrustedDirs` oder pro Agent `agents.list[].tools.exec.safeBinTrustedDirs`).
- `safeBinProfiles` stammt aus der Konfiguration (`tools.exec.safeBinProfiles` oder pro Agent `agents.list[].tools.exec.safeBinProfiles`). Profilschlüssel pro Agent überschreiben globale Schlüssel.
- Einträge in der Zulassungsliste liegen hostlokal in `~/.openclaw/exec-approvals.json` unter `agents.<id>.allowlist` (oder über Control UI / `openclaw approvals allowlist ...`).
- `openclaw security audit` warnt mit `tools.exec.safe_bins_interpreter_unprofiled`, wenn Interpreter-/Runtime-Binaries in `safeBins` ohne explizite Profile vorkommen.
- `openclaw doctor --fix` kann fehlende benutzerdefinierte Einträge `safeBinProfiles.<bin>` als `{}` erzeugen (danach prüfen und verschärfen). Interpreter-/Runtime-Binaries werden nicht automatisch erzeugt.

Beispiel für ein benutzerdefiniertes Profil:
__OC_I18N_900004__
Wenn du `jq` ausdrücklich in `safeBins` aufnimmst, lehnt OpenClaw das Builtin `env` im Safe-Bin-
Modus weiterhin ab, sodass `jq -n env` die Umgebungsvariablen des Host-Prozesses nicht ohne expliziten Pfad in der Zulassungsliste
oder Freigabe-Prompt ausgeben kann.

## Bearbeiten in Control UI

Verwende die Karte **Control UI → Nodes → Exec approvals**, um Standards, agentbezogene
Überschreibungen und Zulassungslisten zu bearbeiten. Wähle einen Scope (Standards oder einen Agenten),
ändere die Richtlinie, füge Muster für die Zulassungsliste hinzu oder entferne sie und klicke dann auf **Save**. Die UI zeigt Metadaten zu **last used**
pro Muster an, damit du die Liste sauber halten kannst.

Der Zielwähler wählt **Gateway** (lokale Freigaben) oder einen **Node**. Nodes
müssen `system.execApprovals.get/set` bekanntgeben (macOS-App oder headless Node-Host).
Wenn ein Node noch keine Exec-Freigaben bekanntgibt, bearbeite dessen lokale
`~/.openclaw/exec-approvals.json` direkt.

CLI: `openclaw approvals` unterstützt Bearbeiten für Gateway oder Node (siehe [Approvals CLI](/cli/approvals)).

## Freigabeablauf

Wenn ein Prompt erforderlich ist, überträgt das Gateway `exec.approval.requested` an Operator-Clients.
Control UI und die macOS-App lösen dies über `exec.approval.resolve`, dann leitet das Gateway die
freigegebene Anfrage an den Node-Host weiter.

Für `host=node` enthalten Freigabeanfragen eine kanonische Nutzlast `systemRunPlan`. Das Gateway verwendet
diesen Plan als maßgeblichen Befehl-/cwd-/Sitzungskontext beim Weiterleiten freigegebener `system.run`-
Anfragen.

Das ist wichtig für asynchrone Latenz bei Freigaben:

- der Node-Exec-Pfad bereitet im Voraus einen kanonischen Plan vor
- der Freigabedatensatz speichert diesen Plan und seine Binding-Metadaten
- nach der Freigabe verwendet der endgültig weitergeleitete `system.run`-Aufruf den gespeicherten Plan erneut,
  statt späteren Änderungen des Aufrufers zu vertrauen
- wenn der Aufrufer `command`, `rawCommand`, `cwd`, `agentId` oder
  `sessionKey` ändert, nachdem die Freigabeanfrage erstellt wurde, lehnt das Gateway den
  weitergeleiteten Lauf wegen eines Freigabe-Mismatch ab

## Interpreter-/Runtime-Befehle

Freigabegestützte Interpreter-/Runtime-Läufe sind absichtlich konservativ:

- Exakter Kontext von argv/cwd/env wird immer gebunden.
- Direkte Shell-Skripte und direkte Runtime-Dateiformen werden best-effort an einen konkreten lokalen
  Dateisnapshot gebunden.
- Häufige Formen mit Wrappern von Paketmanagern, die sich weiterhin zu genau einer direkten lokalen Datei auflösen lassen (zum Beispiel
  `pnpm exec`, `pnpm node`, `npm exec`, `npx`), werden vor dem Binding entpackt.
- Wenn OpenClaw für einen Interpreter-/Runtime-Befehl nicht genau eine konkrete lokale Datei identifizieren kann
  (zum Beispiel Package-Skripte, Eval-Formen, Runtime-spezifische Loader-Ketten oder mehrdeutige Formen mit mehreren Dateien),
  wird die freigabegestützte Ausführung verweigert, statt semantische Abdeckung zu behaupten, die nicht
  tatsächlich vorhanden ist.
- Für solche Workflows solltest du Sandboxing, eine separate Host-Grenze oder einen expliziten
  vertrauenswürdigen Workflow mit Zulassungsliste/`full` bevorzugen, bei dem der Operator die breitere Runtime-Semantik akzeptiert.

Wenn Freigaben erforderlich sind, gibt das Exec-Tool sofort mit einer Freigabe-ID zurück. Verwende diese ID, um
spätere Systemereignisse zu korrelieren (`Exec finished` / `Exec denied`). Wenn vor dem Timeout keine Entscheidung eingeht, wird
die Anfrage als Timeout bei der Freigabe behandelt und als Verweigerungsgrund ausgegeben.

### Verhalten bei Follow-up-Zustellung

Nach Abschluss eines freigegebenen asynchronen Exec sendet OpenClaw einen Follow-up-`agent`-Turn an dieselbe Sitzung.

- Wenn ein gültiges externes Zustellungsziel existiert (zustellbarer Kanal plus Ziel `to`), verwendet die Follow-up-Zustellung diesen Kanal.
- Bei Webchat-only- oder internen Sitzungsabläufen ohne externes Ziel bleibt die Follow-up-Zustellung nur sitzungsintern (`deliver: false`).
- Wenn ein Aufrufer ausdrücklich eine strikte externe Zustellung anfordert, aber kein auflösbarer externer Kanal vorhanden ist, schlägt die Anfrage mit `INVALID_REQUEST` fehl.
- Wenn `bestEffortDeliver` aktiviert ist und kein externer Kanal aufgelöst werden kann, wird die Zustellung auf nur sitzungsintern herabgestuft, statt fehlzuschlagen.

Der Bestätigungsdialog enthält:

- Befehl + Argumente
- cwd
- Agent-ID
- aufgelösten Pfad der ausführbaren Datei
- Host- + Richtlinienmetadaten

Aktionen:

- **Allow once** → jetzt ausführen
- **Always allow** → zur Zulassungsliste hinzufügen + ausführen
- **Deny** → blockieren

## Freigaben an Chat-Kanäle weiterleiten

Du kannst Prompts für Exec-Freigaben an jeden Chat-Kanal weiterleiten (einschließlich Plugin-Kanälen) und
sie mit `/approve` freigeben. Dabei wird die normale Pipeline für ausgehende Zustellung verwendet.

Konfiguration:
__OC_I18N_900005__
Im Chat antworten:
__OC_I18N_900006__
Der Befehl `/approve` verarbeitet sowohl Exec-Freigaben als auch Plugin-Freigaben. Wenn die ID nicht zu einer ausstehenden Exec-Freigabe passt, prüft er automatisch stattdessen Plugin-Freigaben.

### Weiterleitung von Plugin-Freigaben

Die Weiterleitung von Plugin-Freigaben verwendet dieselbe Zustellungspipeline wie Exec-Freigaben, hat aber eine eigene
unabhängige Konfiguration unter `approvals.plugin`. Das Aktivieren oder Deaktivieren der einen wirkt sich nicht auf die andere aus.
__OC_I18N_900007__
Die Form der Konfiguration ist identisch zu `approvals.exec`: `enabled`, `mode`, `agentFilter`,
`sessionFilter` und `targets` funktionieren genauso.

Kanäle, die gemeinsame interaktive Antworten unterstützen, rendern dieselben Freigabe-Buttons sowohl für Exec- als auch für
Plugin-Freigaben. Kanäle ohne gemeinsame interaktive UI fallen auf Klartext mit `/approve`-
Anweisungen zurück.

### Freigaben im selben Chat auf jedem Kanal

Wenn eine Exec- oder Plugin-Freigabeanfrage von einer zustellbaren Chat-Oberfläche stammt, kann derselbe Chat
sie jetzt standardmäßig mit `/approve` freigeben. Dies gilt für Kanäle wie Slack, Matrix und
Microsoft Teams zusätzlich zu den bestehenden Abläufen über Web UI und Terminal UI.

Dieser gemeinsame Pfad über Textbefehle verwendet das normale Kanal-Auth-Modell für diese Unterhaltung. Wenn der
ursprüngliche Chat bereits Befehle senden und Antworten empfangen kann, benötigen Freigabeanfragen keinen
separaten nativen Zustellungsadapter mehr, nur um ausstehend zu bleiben.

Discord und Telegram unterstützen ebenfalls `/approve` im selben Chat, aber diese Kanäle verwenden für die Autorisierung weiterhin
ihre aufgelöste Liste von Approvern, selbst wenn native Zustellung für Freigaben deaktiviert ist.

Für Telegram und andere native Freigabeclients, die das Gateway direkt aufrufen,
ist dieser Fallback absichtlich auf Fehler vom Typ „approval not found“ begrenzt. Eine echte
Verweigerung/ein echter Fehler einer Exec-Freigabe versucht nicht stillschweigend, als Plugin-Freigabe erneut zu laufen.

### Native Zustellung von Freigaben

Einige Kanäle können auch als native Freigabeclients fungieren. Native Clients ergänzen Approver-DMs, Fanout in den Ursprungs-Chat
und kanalspezifische interaktive Freigabe-UX zusätzlich zum gemeinsamen `/approve`-
Ablauf im selben Chat.

Wenn native Freigabekarten/-Buttons verfügbar sind, ist diese native UI der primäre
agentseitige Pfad. Der Agent sollte nicht zusätzlich einen doppelten Klartext-
Befehl `/approve` ausgeben, es sei denn, das Tool-Ergebnis sagt, dass Chat-Freigaben nicht verfügbar sind oder
manuelle Freigabe der einzige verbleibende Weg ist.

Generisches Modell:

- die Host-Exec-Richtlinie entscheidet weiterhin, ob eine Exec-Freigabe erforderlich ist
- `approvals.exec` steuert das Weiterleiten von Freigabe-Prompts an andere Chat-Ziele
- `channels.<channel>.execApprovals` steuert, ob dieser Kanal als nativer Freigabeclient fungiert

Native Freigabeclients aktivieren standardmäßig automatisch DM-first-Zustellung, wenn alle folgenden Bedingungen erfüllt sind:

- der Kanal unterstützt native Zustellung von Freigaben
- Approver können aus expliziten `execApprovals.approvers` oder den
  dokumentierten Fallback-Quellen dieses Kanals aufgelöst werden
- `channels.<channel>.execApprovals.enabled` ist nicht gesetzt oder `"auto"`

Setze `enabled: false`, um einen nativen Freigabeclient ausdrücklich zu deaktivieren. Setze `enabled: true`, um
ihn zu erzwingen, wenn Approver aufgelöst werden. Öffentliche Zustellung in den Ursprungs-Chat bleibt
über `channels.<channel>.execApprovals.target` ausdrücklich gesteuert.

FAQ: [Warum gibt es zwei Konfigurationen für Exec-Freigaben bei Chat-Freigaben?](/help/faq#why-are-there-two-exec-approval-configs-for-chat-approvals)

- Discord: `channels.discord.execApprovals.*`
- Slack: `channels.slack.execApprovals.*`
- Telegram: `channels.telegram.execApprovals.*`

Diese nativen Freigabeclients ergänzen DM-Routing und optionales Fanout in Kanäle zusätzlich zum gemeinsamen
Ablauf `/approve` im selben Chat und gemeinsamen Freigabe-Buttons.

Gemeinsames Verhalten:

- Slack, Matrix, Microsoft Teams und ähnliche zustellbare Chats verwenden das normale Kanal-Auth-Modell
  für `/approve` im selben Chat
- wenn ein nativer Freigabeclient sich automatisch aktiviert, ist das standardmäßige native Zustellungsziel die DM des Approvers
- für Discord und Telegram können nur aufgelöste Approver freigeben oder verweigern
- Discord-Approver können explizit (`execApprovals.approvers`) oder aus `commands.ownerAllowFrom` abgeleitet sein
- Telegram-Approver können explizit (`execApprovals.approvers`) oder aus bestehender Eigentümerkonfiguration abgeleitet sein (`allowFrom`, plus `defaultTo` für Direktnachrichten, sofern unterstützt)
- Slack-Approver können explizit (`execApprovals.approvers`) oder aus `commands.ownerAllowFrom` abgeleitet sein
- native Slack-Buttons behalten die Art der Freigabe-ID bei, sodass IDs mit `plugin:` Plugin-Freigaben auflösen können,
  ohne eine zweite Slack-lokale Fallback-Schicht
- natives Matrix-DM-/Kanal-Routing und Reaktions-Shortcuts verarbeiten sowohl Exec- als auch Plugin-Freigaben;
  die Autorisierung für Plugins stammt weiterhin aus `channels.matrix.dm.allowFrom`
- der Anfragende muss kein Approver sein
- der Ursprungs-Chat kann direkt mit `/approve` freigeben, wenn dieser Chat bereits Befehle und Antworten unterstützt
- native Discord-Freigabe-Buttons routen nach Art der Freigabe-ID: IDs mit `plugin:` gehen
  direkt zu Plugin-Freigaben, alles andere zu Exec-Freigaben
- native Telegram-Freigabe-Buttons folgen demselben begrenzten Fallback von Exec zu Plugin wie `/approve`
- wenn natives `target` Zustellung in den Ursprungs-Chat aktiviert, enthalten Freigabe-Prompts den Befehlstext
- ausstehende Exec-Freigaben laufen standardmäßig nach 30 Minuten ab
- wenn keine Operator-UI oder kein konfigurierter Freigabeclient die Anfrage annehmen kann, fällt der Prompt auf `askFallback` zurück

Telegram verwendet standardmäßig Approver-DMs (`target: "dm"`). Du kannst auf `channel` oder `both` umstellen, wenn
Freigabe-Prompts auch im ursprünglichen Telegram-Chat/-Thema erscheinen sollen. Bei Telegram-Forenthemen
bewahrt OpenClaw das Thema sowohl für den Freigabe-Prompt als auch für das Follow-up nach der Freigabe.

Siehe:

- [Discord](/channels/discord)
- [Telegram](/channels/telegram)

### macOS-IPC-Ablauf
__OC_I18N_900008__
Sicherheitshinweise:

- Unix-Socket-Modus `0600`, Token gespeichert in `exec-approvals.json`.
- Peer-Check mit derselben UID.
- Challenge/Response (Nonce + HMAC-Token + Request-Hash) + kurze TTL.

## Systemereignisse

Der Exec-Lebenszyklus wird als Systemnachrichten ausgegeben:

- `Exec running` (nur wenn der Befehl den Schwellenwert für Laufmeldungen überschreitet)
- `Exec finished`
- `Exec denied`

Diese werden in der Sitzung des Agenten gepostet, nachdem der Node das Ereignis gemeldet hat.
Exec-Freigaben auf dem Gateway-Host geben dieselben Lebenszyklus-Ereignisse aus, wenn der Befehl abgeschlossen ist (und optional bei längerer Laufzeit als dem Schwellenwert).
Durch Freigaben geschützte Execs verwenden die Freigabe-ID in diesen Nachrichten erneut als `runId`, damit sie leicht korreliert werden können.

## Verhalten bei verweigerter Freigabe

Wenn eine asynchrone Exec-Freigabe verweigert wird, verhindert OpenClaw, dass der Agent
Ausgaben aus einem früheren Lauf desselben Befehls in der Sitzung wiederverwendet. Der Verweigerungsgrund
wird mit explizitem Hinweis übergeben, dass keine Befehlsausgabe verfügbar ist. Dadurch wird verhindert,
dass der Agent behauptet, es gebe neue Ausgabe, oder den verweigerten Befehl mit
veralteten Ergebnissen aus einem früheren erfolgreichen Lauf wiederholt.

## Auswirkungen

- **full** ist mächtig; wenn möglich, Zulassungslisten bevorzugen.
- **ask** hält dich im Ablauf, ermöglicht aber trotzdem schnelle Freigaben.
- Zulassungslisten pro Agent verhindern, dass Freigaben eines Agenten in andere auslaufen.
- Freigaben gelten nur für Host-Exec-Anfragen von **autorisierten Sendern**. Nicht autorisierte Sender können kein `/exec` ausführen.
- `/exec security=full` ist eine bequeme Option auf Sitzungsebene für autorisierte Operatoren und überspringt Freigaben absichtlich.
  Um Host-Exec hart zu blockieren, setze die Freigabe-Security auf `deny` oder verweigere das Tool `exec` über Tool-Richtlinien.

Verwandt:

- [Exec-Tool](/de/tools/exec)
- [Elevated mode](/de/tools/elevated)
- [Skills](/de/tools/skills)

## Verwandt

- [Exec](/de/tools/exec) — Tool zur Ausführung von Shell-Befehlen
- [Sandboxing](/de/gateway/sandboxing) — Sandbox-Modi und Workspace-Zugriff
- [Sicherheit](/de/gateway/security) — Sicherheitsmodell und Härtung
- [Sandbox vs Tool Policy vs Elevated](/de/gateway/sandbox-vs-tool-policy-vs-elevated) — wann was verwendet werden sollte
