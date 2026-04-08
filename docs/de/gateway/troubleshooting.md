---
read_when:
    - Das Hub zur Fehlerbehebung hat Sie für eine tiefere Diagnose hierher verwiesen
    - Sie benötigen stabile symptomorientierte Runbook-Abschnitte mit exakten Befehlen
summary: Detailliertes Runbook zur Fehlerbehebung für Gateway, Channels, Automatisierung, Nodes und Browser
title: Fehlerbehebung
x-i18n:
    generated_at: "2026-04-08T02:15:47Z"
    model: gpt-5.4
    provider: openai
    source_hash: 02c9537845248db0c9d315bf581338a93215fe6fe3688ed96c7105cbb19fe6ba
    source_path: gateway/troubleshooting.md
    workflow: 15
---

# Gateway-Fehlerbehebung

Diese Seite ist das detaillierte Runbook.
Beginnen Sie unter [/help/troubleshooting](/de/help/troubleshooting), wenn Sie zuerst den schnellen Triage-Ablauf möchten.

## Befehlsabfolge

Führen Sie diese Befehle zuerst in dieser Reihenfolge aus:

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

Erwartete Anzeichen für einen gesunden Zustand:

- `openclaw gateway status` zeigt `Runtime: running` und `RPC probe: ok`.
- `openclaw doctor` meldet keine blockierenden Konfigurations-/Serviceprobleme.
- `openclaw channels status --probe` zeigt den Live-Transportstatus pro Konto und,
  wo unterstützt, Probe-/Audit-Ergebnisse wie `works` oder `audit ok`.

## Anthropic 429 zusätzlicher Verbrauch für langen Kontext erforderlich

Verwenden Sie diesen Abschnitt, wenn Logs/Fehler Folgendes enthalten:
`HTTP 429: rate_limit_error: Extra usage is required for long context requests`.

```bash
openclaw logs --follow
openclaw models status
openclaw config get agents.defaults.models
```

Achten Sie auf Folgendes:

- Das ausgewählte Anthropic-Opus-/Sonnet-Modell hat `params.context1m: true`.
- Die aktuelle Anthropic-Anmeldung ist nicht für die Nutzung mit langem Kontext berechtigt.
- Anfragen schlagen nur bei langen Sitzungen/Modelläufen fehl, die den 1M-Beta-Pfad benötigen.

Optionen zur Behebung:

1. Deaktivieren Sie `context1m` für dieses Modell, um auf das normale Kontextfenster zurückzufallen.
2. Verwenden Sie eine Anthropic-Anmeldung, die für Anfragen mit langem Kontext berechtigt ist, oder wechseln Sie zu einem Anthropic-API-Schlüssel.
3. Konfigurieren Sie Fallback-Modelle, damit Läufe fortgesetzt werden, wenn Anthropic-Anfragen mit langem Kontext abgelehnt werden.

Zugehörig:

- [/providers/anthropic](/de/providers/anthropic)
- [/reference/token-use](/de/reference/token-use)
- [/help/faq#why-am-i-seeing-http-429-ratelimiterror-from-anthropic](/de/help/faq#why-am-i-seeing-http-429-ratelimiterror-from-anthropic)

## Lokales OpenAI-kompatibles Backend besteht direkte Probes, aber Agent-Läufe schlagen fehl

Verwenden Sie diesen Abschnitt, wenn:

- `curl ... /v1/models` funktioniert
- kleine direkte `/v1/chat/completions`-Aufrufe funktionieren
- OpenClaw-Modelläufe nur bei normalen Agent-Turns fehlschlagen

```bash
curl http://127.0.0.1:1234/v1/models
curl http://127.0.0.1:1234/v1/chat/completions \
  -H 'content-type: application/json' \
  -d '{"model":"<id>","messages":[{"role":"user","content":"hi"}],"stream":false}'
openclaw infer model run --model <provider/model> --prompt "hi" --json
openclaw logs --follow
```

Achten Sie auf Folgendes:

- Direkte kleine Aufrufe sind erfolgreich, aber OpenClaw-Läufe schlagen nur bei größeren Prompts fehl
- Backend-Fehler, bei denen `messages[].content` einen String erwartet
- Backend-Abstürze, die nur bei größeren Prompt-Token-Zahlen oder vollständigen Agent-
  Laufzeit-Prompts auftreten

Häufige Anzeichen:

- `messages[...].content: invalid type: sequence, expected a string` → das Backend
  lehnt strukturierte Chat-Completions-Inhaltsteile ab. Behebung: setzen Sie
  `models.providers.<provider>.models[].compat.requiresStringContent: true`.
- Direkte kleine Anfragen sind erfolgreich, aber OpenClaw-Agent-Läufe schlagen mit Backend-/Modell-
  Abstürzen fehl (zum Beispiel Gemma auf einigen `inferrs`-Builds) → der OpenClaw-Transport ist
  wahrscheinlich bereits korrekt; das Backend scheitert an der größeren Agent-Laufzeit-
  Prompt-Form.
- Die Fehler nehmen ab, nachdem Tools deaktiviert wurden, verschwinden aber nicht → Tool-Schemas waren
  Teil des Drucks, aber das verbleibende Problem ist weiterhin eine vorgelagerte Modell-/Server-Kapazität
  oder ein Backend-Fehler.

Optionen zur Behebung:

1. Setzen Sie `compat.requiresStringContent: true` für Chat-Completions-Backends, die nur Strings unterstützen.
2. Setzen Sie `compat.supportsTools: false` für Modelle/Backends, die die
   OpenClaw-Tool-Schema-Oberfläche nicht zuverlässig verarbeiten können.
3. Reduzieren Sie den Prompt-Druck, wo möglich: kleinerer Workspace-Bootstrap, kürzere
   Sitzungshistorie, leichteres lokales Modell oder ein Backend mit stärkerer Unterstützung für langen Kontext.
4. Wenn direkte kleine Anfragen weiterhin erfolgreich sind, während OpenClaw-Agent-Turns im Backend
   trotzdem abstürzen, behandeln Sie dies als vorgelagerte Server-/Modellbeschränkung und reichen Sie dort
   eine Reproduktion mit der akzeptierten Payload-Form ein.

Zugehörig:

- [/gateway/local-models](/de/gateway/local-models)
- [/gateway/configuration#models](/de/gateway/configuration#models)
- [/gateway/configuration-reference#openai-compatible-endpoints](/de/gateway/configuration-reference#openai-compatible-endpoints)

## Keine Antworten

Wenn Channels aktiv sind, aber nichts antwortet, prüfen Sie Routing und Richtlinien, bevor Sie irgendetwas neu verbinden.

```bash
openclaw status
openclaw channels status --probe
openclaw pairing list --channel <channel> [--account <id>]
openclaw config get channels
openclaw logs --follow
```

Achten Sie auf Folgendes:

- Pairing steht für DM-Absender aus.
- Gating für Gruppenerwähnungen (`requireMention`, `mentionPatterns`).
- Nichtübereinstimmungen bei der Channel-/Gruppen-Allowlist.

Häufige Anzeichen:

- `drop guild message (mention required` → Gruppennachricht wird ignoriert, bis eine Erwähnung erfolgt.
- `pairing request` → der Absender benötigt eine Genehmigung.
- `blocked` / `allowlist` → Absender/Channel wurde durch eine Richtlinie gefiltert.

Zugehörig:

- [/channels/troubleshooting](/de/channels/troubleshooting)
- [/channels/pairing](/de/channels/pairing)
- [/channels/groups](/de/channels/groups)

## Dashboard-Control-UI-Konnektivität

Wenn Dashboard/Control UI keine Verbindung herstellt, prüfen Sie URL, Auth-Modus und Annahmen zum sicheren Kontext.

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
openclaw doctor
openclaw gateway status --json
```

Achten Sie auf Folgendes:

- Korrekte Probe-URL und Dashboard-URL.
- Nichtübereinstimmung bei Auth-Modus/Token zwischen Client und Gateway.
- HTTP-Nutzung, wo Geräteidentität erforderlich ist.

Häufige Anzeichen:

- `device identity required` → unsicherer Kontext oder fehlende Geräteauthentifizierung.
- `origin not allowed` → Browser-`Origin` ist nicht in `gateway.controlUi.allowedOrigins`
  enthalten (oder Sie verbinden sich von einer Browser-Origin, die nicht loopback ist, ohne explizite
  Allowlist).
- `device nonce required` / `device nonce mismatch` → der Client schließt den
  herausforderungsbasierten Geräteauthentifizierungsablauf nicht ab (`connect.challenge` + `device.nonce`).
- `device signature invalid` / `device signature expired` → der Client hat die falsche
  Payload (oder einen veralteten Zeitstempel) für den aktuellen Handshake signiert.
- `AUTH_TOKEN_MISMATCH` mit `canRetryWithDeviceToken=true` → der Client kann einen vertrauenswürdigen Wiederholungsversuch mit zwischengespeichertem Gerätetoken ausführen.
- Dieser Wiederholungsversuch mit zwischengespeichertem Token verwendet erneut die zwischengespeicherte Scope-Menge, die mit dem gekoppelten
  Gerätetoken gespeichert wurde. Aufrufer mit explizitem `deviceToken` / expliziten `scopes` behalten stattdessen ihre
  angeforderte Scope-Menge bei.
- Außerhalb dieses Wiederholungswegs ist die Priorität für die Verbindungsauthentifizierung explizit
  gemeinsames Token/Passwort zuerst, dann explizites `deviceToken`, dann gespeichertes Gerätetoken,
  dann Bootstrap-Token.
- Auf dem asynchronen Tailscale-Serve-Control-UI-Pfad werden fehlgeschlagene Versuche für dieselbe
  `{scope, ip}` serialisiert, bevor der Limiter den Fehler erfasst. Zwei fehlerhafte gleichzeitige Wiederholungsversuche desselben Clients können daher beim zweiten Versuch
  `retry later` statt zweier einfacher Nichtübereinstimmungen auslösen.
- `too many failed authentication attempts (retry later)` von einem loopback-Client mit Browser-Origin
  → wiederholte Fehler von derselben normalisierten `Origin` werden vorübergehend ausgesperrt; eine andere localhost-Origin verwendet einen separaten Bucket.
- Wiederholtes `unauthorized` nach diesem Wiederholungsversuch → Drift zwischen gemeinsamem Token/Gerätetoken; aktualisieren Sie die Token-Konfiguration und genehmigen/rotieren Sie das Gerätetoken bei Bedarf erneut.
- `gateway connect failed:` → falscher Host/Port/falsches URL-Ziel.

### Schnelle Zuordnung von Auth-Detailcodes

Verwenden Sie `error.details.code` aus der fehlgeschlagenen `connect`-Antwort, um die nächste Maßnahme zu bestimmen:

| Detail code                  | Bedeutung                                               | Empfohlene Maßnahme                                                                                                                                                                                                                                                                          |
| ---------------------------- | ------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `AUTH_TOKEN_MISSING`         | Der Client hat kein erforderliches gemeinsames Token gesendet. | Fügen Sie das Token im Client ein/setzen Sie es und versuchen Sie es erneut. Für Dashboard-Pfade: `openclaw config get gateway.auth.token` und dann in die Control-UI-Einstellungen einfügen.                                                                                              |
| `AUTH_TOKEN_MISMATCH`        | Das gemeinsame Token stimmte nicht mit dem Gateway-Auth-Token überein. | Wenn `canRetryWithDeviceToken=true`, erlauben Sie einen vertrauenswürdigen Wiederholungsversuch. Wiederholungsversuche mit zwischengespeichertem Token verwenden erneut gespeicherte genehmigte Scopes; Aufrufer mit explizitem `deviceToken` / `scopes` behalten die angeforderten Scopes. Wenn es weiterhin fehlschlägt, führen Sie die [Checkliste zur Behebung von Token-Drift](/cli/devices#token-drift-recovery-checklist) aus. |
| `AUTH_DEVICE_TOKEN_MISMATCH` | Das zwischengespeicherte gerätespezifische Token ist veraltet oder widerrufen. | Rotieren/genehmigen Sie das Gerätetoken mit [devices CLI](/cli/devices) erneut und stellen Sie die Verbindung dann erneut her.                                                                                                                                                               |
| `PAIRING_REQUIRED`           | Die Geräteidentität ist bekannt, aber nicht für diese Rolle genehmigt. | Genehmigen Sie die ausstehende Anfrage: `openclaw devices list` und dann `openclaw devices approve <requestId>`.                                                                                                                                                                              |

Prüfung der Migration zu Device Auth v2:

```bash
openclaw --version
openclaw doctor
openclaw gateway status
```

Wenn die Logs Nonce-/Signaturfehler zeigen, aktualisieren Sie den verbindenden Client und prüfen Sie, dass er:

1. auf `connect.challenge` wartet
2. die challenge-gebundene Payload signiert
3. `connect.params.device.nonce` mit derselben Challenge-Nonce sendet

Wenn `openclaw devices rotate` / `revoke` / `remove` unerwartet verweigert wird:

- Sitzungen mit gekoppeltem Gerätetoken können nur **ihr eigenes** Gerät verwalten, es sei denn, der
  Aufrufer hat zusätzlich `operator.admin`
- `openclaw devices rotate --scope ...` kann nur Operator-Scopes anfordern, die
  die Aufrufer-Sitzung bereits besitzt

Zugehörig:

- [/web/control-ui](/web/control-ui)
- [/gateway/configuration](/de/gateway/configuration) (Gateway-Auth-Modi)
- [/gateway/trusted-proxy-auth](/de/gateway/trusted-proxy-auth)
- [/gateway/remote](/de/gateway/remote)
- [/cli/devices](/cli/devices)

## Gateway-Service läuft nicht

Verwenden Sie diesen Abschnitt, wenn der Service installiert ist, der Prozess aber nicht dauerhaft läuft.

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
openclaw doctor
openclaw gateway status --deep   # scannt auch Dienste auf Systemebene
```

Achten Sie auf Folgendes:

- `Runtime: stopped` mit Hinweisen zum Beenden.
- Nichtübereinstimmung bei der Service-Konfiguration (`Config (cli)` vs `Config (service)`).
- Port-/Listener-Konflikte.
- Zusätzliche launchd-/systemd-/schtasks-Installationen bei Verwendung von `--deep`.
- Bereinigungshinweise für `Other gateway-like services detected (best effort)`.

Häufige Anzeichen:

- `Gateway start blocked: set gateway.mode=local` oder `existing config is missing gateway.mode` → der lokale Gateway-Modus ist nicht aktiviert, oder die Konfigurationsdatei wurde überschrieben und hat `gateway.mode` verloren. Behebung: setzen Sie `gateway.mode="local"` in Ihrer Konfiguration oder führen Sie `openclaw onboard --mode local` / `openclaw setup` erneut aus, um die erwartete Konfiguration für den lokalen Modus wiederherzustellen. Wenn Sie OpenClaw über Podman ausführen, ist der Standard-Konfigurationspfad `~/.openclaw/openclaw.json`.
- `refusing to bind gateway ... without auth` → Nicht-loopback-Bind ohne gültigen Gateway-Auth-Pfad (Token/Passwort oder Trusted-Proxy, sofern konfiguriert).
- `another gateway instance is already listening` / `EADDRINUSE` → Portkonflikt.
- `Other gateway-like services detected (best effort)` → veraltete oder parallele launchd-/systemd-/schtasks-Units existieren. Die meisten Setups sollten ein Gateway pro Maschine beibehalten; wenn Sie tatsächlich mehr als eines benötigen, isolieren Sie Ports + Konfiguration/Status/Workspace. Siehe [/gateway#multiple-gateways-same-host](/de/gateway#multiple-gateways-same-host).

Zugehörig:

- [/gateway/background-process](/de/gateway/background-process)
- [/gateway/configuration](/de/gateway/configuration)
- [/gateway/doctor](/de/gateway/doctor)

## Gateway-Probe-Warnungen

Verwenden Sie diesen Abschnitt, wenn `openclaw gateway probe` etwas erreicht, aber trotzdem einen Warnblock ausgibt.

```bash
openclaw gateway probe
openclaw gateway probe --json
openclaw gateway probe --ssh user@gateway-host
```

Achten Sie auf Folgendes:

- `warnings[].code` und `primaryTargetId` in der JSON-Ausgabe.
- Ob sich die Warnung auf SSH-Fallback, mehrere Gateways, fehlende Scopes oder nicht aufgelöste Auth-Refs bezieht.

Häufige Anzeichen:

- `SSH tunnel failed to start; falling back to direct probes.` → die SSH-Einrichtung ist fehlgeschlagen, aber der Befehl hat trotzdem direkte konfigurierte/loopback-Ziele versucht.
- `multiple reachable gateways detected` → mehr als ein Ziel hat geantwortet. Das bedeutet normalerweise ein beabsichtigtes Multi-Gateway-Setup oder veraltete/duplizierte Listener.
- `Probe diagnostics are limited by gateway scopes (missing operator.read)` → die Verbindung hat funktioniert, aber die Detail-RPC ist durch Scopes eingeschränkt; koppeln Sie die Geräteidentität oder verwenden Sie Anmeldedaten mit `operator.read`.
- nicht aufgelöster SecretRef-Warntext zu `gateway.auth.*` / `gateway.remote.*` → Auth-Material war in diesem Befehlspfad für das fehlgeschlagene Ziel nicht verfügbar.

Zugehörig:

- [/cli/gateway](/cli/gateway)
- [/gateway#multiple-gateways-same-host](/de/gateway#multiple-gateways-same-host)
- [/gateway/remote](/de/gateway/remote)

## Channel verbunden, aber Nachrichten fließen nicht

Wenn der Channel-Status verbunden ist, der Nachrichtenfluss aber tot ist, konzentrieren Sie sich auf Richtlinien, Berechtigungen und channelspezifische Zustellregeln.

```bash
openclaw channels status --probe
openclaw pairing list --channel <channel> [--account <id>]
openclaw status --deep
openclaw logs --follow
openclaw config get channels
```

Achten Sie auf Folgendes:

- DM-Richtlinie (`pairing`, `allowlist`, `open`, `disabled`).
- Gruppen-Allowlist und Anforderungen an Erwähnungen.
- Fehlende Channel-API-Berechtigungen/-Scopes.

Häufige Anzeichen:

- `mention required` → Nachricht wird durch die Gruppen-Erwähnungsrichtlinie ignoriert.
- `pairing` / ausstehende Genehmigungsspuren → der Absender ist nicht genehmigt.
- `missing_scope`, `not_in_channel`, `Forbidden`, `401/403` → Problem mit Channel-Auth/Berechtigungen.

Zugehörig:

- [/channels/troubleshooting](/de/channels/troubleshooting)
- [/channels/whatsapp](/de/channels/whatsapp)
- [/channels/telegram](/de/channels/telegram)
- [/channels/discord](/de/channels/discord)

## Cron- und Heartbeat-Zustellung

Wenn Cron oder Heartbeat nicht ausgeführt wurde oder nicht zugestellt wurde, prüfen Sie zuerst den Scheduler-Status und dann das Zustellziel.

```bash
openclaw cron status
openclaw cron list
openclaw cron runs --id <jobId> --limit 20
openclaw system heartbeat last
openclaw logs --follow
```

Achten Sie auf Folgendes:

- Cron ist aktiviert und die nächste Aufweckzeit ist vorhanden.
- Status des Job-Ausführungsverlaufs (`ok`, `skipped`, `error`).
- Heartbeat-Überspringgründe (`quiet-hours`, `requests-in-flight`, `alerts-disabled`, `empty-heartbeat-file`, `no-tasks-due`).

Häufige Anzeichen:

- `cron: scheduler disabled; jobs will not run automatically` → Cron ist deaktiviert.
- `cron: timer tick failed` → Scheduler-Tick ist fehlgeschlagen; prüfen Sie Datei-/Log-/Laufzeitfehler.
- `heartbeat skipped` mit `reason=quiet-hours` → außerhalb des Zeitfensters für aktive Stunden.
- `heartbeat skipped` mit `reason=empty-heartbeat-file` → `HEARTBEAT.md` existiert, enthält aber nur Leerzeilen / Markdown-Überschriften, daher überspringt OpenClaw den Modellaufruf.
- `heartbeat skipped` mit `reason=no-tasks-due` → `HEARTBEAT.md` enthält einen `tasks:`-Block, aber keine der Aufgaben ist bei diesem Tick fällig.
- `heartbeat: unknown accountId` → ungültige Konto-ID für das Heartbeat-Zustellziel.
- `heartbeat skipped` mit `reason=dm-blocked` → das Heartbeat-Ziel wurde in ein Ziel im DM-Stil aufgelöst, während `agents.defaults.heartbeat.directPolicy` (oder die agentenspezifische Überschreibung) auf `block` gesetzt ist.

Zugehörig:

- [/automation/cron-jobs#troubleshooting](/de/automation/cron-jobs#troubleshooting)
- [/automation/cron-jobs](/de/automation/cron-jobs)
- [/gateway/heartbeat](/de/gateway/heartbeat)

## Gekoppeltes Node-Tool schlägt fehl

Wenn ein Node gekoppelt ist, Tools aber fehlschlagen, isolieren Sie Vordergrund-, Berechtigungs- und Genehmigungsstatus.

```bash
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
openclaw approvals get --node <idOrNameOrIp>
openclaw logs --follow
openclaw status
```

Achten Sie auf Folgendes:

- Node ist online und hat die erwarteten Fähigkeiten.
- OS-Berechtigungen für Kamera/Mikrofon/Standort/Bildschirm.
- Exec-Genehmigungen und Allowlist-Status.

Häufige Anzeichen:

- `NODE_BACKGROUND_UNAVAILABLE` → die Node-App muss im Vordergrund sein.
- `*_PERMISSION_REQUIRED` / `LOCATION_PERMISSION_REQUIRED` → fehlende OS-Berechtigung.
- `SYSTEM_RUN_DENIED: approval required` → Exec-Genehmigung steht aus.
- `SYSTEM_RUN_DENIED: allowlist miss` → Befehl wurde durch die Allowlist blockiert.

Zugehörig:

- [/nodes/troubleshooting](/de/nodes/troubleshooting)
- [/nodes/index](/de/nodes/index)
- [/tools/exec-approvals](/de/tools/exec-approvals)

## Browser-Tool schlägt fehl

Verwenden Sie diesen Abschnitt, wenn Browser-Tool-Aktionen fehlschlagen, obwohl das Gateway selbst gesund ist.

```bash
openclaw browser status
openclaw browser start --browser-profile openclaw
openclaw browser profiles
openclaw logs --follow
openclaw doctor
```

Achten Sie auf Folgendes:

- Ob `plugins.allow` gesetzt ist und `browser` enthält.
- Gültiger Pfad zur Browser-Executable.
- Erreichbarkeit des CDP-Profils.
- Verfügbarkeit von lokalem Chrome für `existing-session`-/`user`-Profile.

Häufige Anzeichen:

- `unknown command "browser"` oder `unknown command 'browser'` → das gebündelte Browser-Plugin ist durch `plugins.allow` ausgeschlossen.
- Browser-Tool fehlt / ist nicht verfügbar, obwohl `browser.enabled=true` → `plugins.allow` schließt `browser` aus, daher wurde das Plugin nie geladen.
- `Failed to start Chrome CDP on port` → Browser-Prozess konnte nicht gestartet werden.
- `browser.executablePath not found` → der konfigurierte Pfad ist ungültig.
- `browser.cdpUrl must be http(s) or ws(s)` → die konfigurierte CDP-URL verwendet ein nicht unterstütztes Schema wie `file:` oder `ftp:`.
- `browser.cdpUrl has invalid port` → die konfigurierte CDP-URL hat einen fehlerhaften oder ungültigen Port.
- `No Chrome tabs found for profile="user"` → das Chrome-MCP-Attach-Profil hat keine geöffneten lokalen Chrome-Tabs.
- `Remote CDP for profile "<name>" is not reachable` → der konfigurierte Remote-CDP-Endpunkt ist vom Gateway-Host aus nicht erreichbar.
- `Browser attachOnly is enabled ... not reachable` oder `Browser attachOnly is enabled and CDP websocket ... is not reachable` → das Attach-only-Profil hat kein erreichbares Ziel, oder der HTTP-Endpunkt hat geantwortet, aber der CDP-WebSocket konnte trotzdem nicht geöffnet werden.
- `Playwright is not available in this gateway build; '<feature>' is unsupported.` → der aktuelle Gateway-Install enthält nicht das vollständige Playwright-Paket; ARIA-Snapshots und einfache Seitenscreenshots können weiterhin funktionieren, aber Navigation, AI-Snapshots, Element-Screenshots per CSS-Selektor und PDF-Export bleiben nicht verfügbar.
- `fullPage is not supported for element screenshots` → die Screenshot-Anfrage hat `--full-page` mit `--ref` oder `--element` kombiniert.
- `element screenshots are not supported for existing-session profiles; use ref from snapshot.` → Screenshot-Aufrufe von Chrome MCP / `existing-session` müssen Seitenerfassung oder einen Snapshot-`--ref` verwenden, nicht CSS-`--element`.
- `existing-session file uploads do not support element selectors; use ref/inputRef.` → Upload-Hooks von Chrome MCP benötigen Snapshot-Refs, keine CSS-Selektoren.
- `existing-session file uploads currently support one file at a time.` → senden Sie bei Chrome-MCP-Profilen einen Upload pro Aufruf.
- `existing-session dialog handling does not support timeoutMs.` → Dialog-Hooks auf Chrome-MCP-Profilen unterstützen keine Timeout-Überschreibungen.
- `response body is not supported for existing-session profiles yet.` → `responsebody` erfordert weiterhin einen verwalteten Browser oder ein rohes CDP-Profil.
- veraltete Überschreibungen für Viewport / Dark Mode / Gebietsschema / Offline-Modus auf Attach-only- oder Remote-CDP-Profilen → führen Sie `openclaw browser stop --browser-profile <name>` aus, um die aktive Steuersitzung zu schließen und den Playwright-/CDP-Emulationsstatus freizugeben, ohne das gesamte Gateway neu zu starten.

Zugehörig:

- [/tools/browser-linux-troubleshooting](/de/tools/browser-linux-troubleshooting)
- [/tools/browser](/de/tools/browser)

## Wenn Sie ein Upgrade durchgeführt haben und plötzlich etwas nicht mehr funktioniert

Die meisten Probleme nach einem Upgrade sind Konfigurations-Drift oder jetzt erzwungene strengere Standardwerte.

### 1) Verhalten von Auth- und URL-Überschreibungen hat sich geändert

```bash
openclaw gateway status
openclaw config get gateway.mode
openclaw config get gateway.remote.url
openclaw config get gateway.auth.mode
```

Was zu prüfen ist:

- Wenn `gateway.mode=remote`, können CLI-Aufrufe auf remote zielen, während Ihr lokaler Service in Ordnung ist.
- Explizite `--url`-Aufrufe greifen nicht auf gespeicherte Anmeldedaten zurück.

Häufige Anzeichen:

- `gateway connect failed:` → falsches URL-Ziel.
- `unauthorized` → Endpunkt erreichbar, aber falsche Authentifizierung.

### 2) Bind- und Auth-Schutzmechanismen sind strenger

```bash
openclaw config get gateway.bind
openclaw config get gateway.auth.mode
openclaw config get gateway.auth.token
openclaw gateway status
openclaw logs --follow
```

Was zu prüfen ist:

- Nicht-loopback-Binds (`lan`, `tailnet`, `custom`) benötigen einen gültigen Gateway-Auth-Pfad: gemeinsame Token-/Passwort-Authentifizierung oder eine korrekt konfigurierte nicht-loopback-`trusted-proxy`-Bereitstellung.
- Alte Schlüssel wie `gateway.token` ersetzen `gateway.auth.token` nicht.

Häufige Anzeichen:

- `refusing to bind gateway ... without auth` → Nicht-loopback-Bind ohne gültigen Gateway-Auth-Pfad.
- `RPC probe: failed` während die Runtime läuft → Gateway lebt, ist aber mit der aktuellen Auth/URL nicht zugänglich.

### 3) Pairing- und Geräteidentitätsstatus haben sich geändert

```bash
openclaw devices list
openclaw pairing list --channel <channel> [--account <id>]
openclaw logs --follow
openclaw doctor
```

Was zu prüfen ist:

- Ausstehende Gerätegenehmigungen für Dashboard/Nodes.
- Ausstehende DM-Pairing-Genehmigungen nach Richtlinien- oder Identitätsänderungen.

Häufige Anzeichen:

- `device identity required` → Geräteauthentifizierung ist nicht erfüllt.
- `pairing required` → Absender/Gerät muss genehmigt werden.

Wenn Service-Konfiguration und Runtime nach den Prüfungen weiterhin nicht übereinstimmen, installieren Sie die Service-Metadaten aus demselben Profil-/Statusverzeichnis neu:

```bash
openclaw gateway install --force
openclaw gateway restart
```

Zugehörig:

- [/gateway/pairing](/de/gateway/pairing)
- [/gateway/authentication](/de/gateway/authentication)
- [/gateway/background-process](/de/gateway/background-process)
