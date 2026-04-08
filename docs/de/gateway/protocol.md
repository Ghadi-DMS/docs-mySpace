---
read_when:
    - Implementierung oder Aktualisierung von Gateway-WS-Clients
    - Debugging von Protokollabweichungen oder Verbindungsfehlern
    - Neugenerieren von Protokoll-Schema/-Modellen
summary: 'Gateway-WebSocket-Protokoll: Handshake, Frames, Versionierung'
title: Gateway-Protokoll
x-i18n:
    generated_at: "2026-04-08T02:16:02Z"
    model: gpt-5.4
    provider: openai
    source_hash: 8635e3ac1dd311dbd3a770b088868aa1495a8d53b3ebc1eae0dfda3b2bf4694a
    source_path: gateway/protocol.md
    workflow: 15
---

# Gateway-Protokoll (WebSocket)

Das Gateway-WS-Protokoll ist die **einzige Control Plane + der einzige Node-Transport** für
OpenClaw. Alle Clients (CLI, Web-UI, macOS-App, iOS/Android-Nodes, headless
Nodes) verbinden sich über WebSocket und deklarieren beim
Handshake ihre **Rolle** + ihren **Scope**.

## Transport

- WebSocket, Text-Frames mit JSON-Nutzdaten.
- Der erste Frame **muss** eine `connect`-Anfrage sein.

## Handshake (connect)

Gateway → Client (Pre-Connect-Challenge):

```json
{
  "type": "event",
  "event": "connect.challenge",
  "payload": { "nonce": "…", "ts": 1737264000000 }
}
```

Client → Gateway:

```json
{
  "type": "req",
  "id": "…",
  "method": "connect",
  "params": {
    "minProtocol": 3,
    "maxProtocol": 3,
    "client": {
      "id": "cli",
      "version": "1.2.3",
      "platform": "macos",
      "mode": "operator"
    },
    "role": "operator",
    "scopes": ["operator.read", "operator.write"],
    "caps": [],
    "commands": [],
    "permissions": {},
    "auth": { "token": "…" },
    "locale": "en-US",
    "userAgent": "openclaw-cli/1.2.3",
    "device": {
      "id": "device_fingerprint",
      "publicKey": "…",
      "signature": "…",
      "signedAt": 1737264000000,
      "nonce": "…"
    }
  }
}
```

Gateway → Client:

```json
{
  "type": "res",
  "id": "…",
  "ok": true,
  "payload": { "type": "hello-ok", "protocol": 3, "policy": { "tickIntervalMs": 15000 } }
}
```

Wenn ein Device-Token ausgegeben wird, enthält `hello-ok` außerdem:

```json
{
  "auth": {
    "deviceToken": "…",
    "role": "operator",
    "scopes": ["operator.read", "operator.write"]
  }
}
```

Während eines vertrauten Bootstrap-Handoffs kann `hello-ok.auth` zusätzlich weitere
begrenzte Rolleneinträge in `deviceTokens` enthalten:

```json
{
  "auth": {
    "deviceToken": "…",
    "role": "node",
    "scopes": [],
    "deviceTokens": [
      {
        "deviceToken": "…",
        "role": "operator",
        "scopes": ["operator.approvals", "operator.read", "operator.talk.secrets", "operator.write"]
      }
    ]
  }
}
```

Für den integrierten Node/Operator-Bootstrap-Ablauf bleibt das primäre Node-Token bei
`scopes: []`, und jedes übergebene Operator-Token bleibt auf die Bootstrap-
Operator-Zulassungsliste begrenzt (`operator.approvals`, `operator.read`,
`operator.talk.secrets`, `operator.write`). Bootstrap-Scope-Prüfungen bleiben
rollenpräfixbasiert: Operator-Einträge erfüllen nur Operator-Anfragen, und Nicht-Operator-
Rollen benötigen weiterhin Scopes unter ihrem eigenen Rollenpräfix.

### Node-Beispiel

```json
{
  "type": "req",
  "id": "…",
  "method": "connect",
  "params": {
    "minProtocol": 3,
    "maxProtocol": 3,
    "client": {
      "id": "ios-node",
      "version": "1.2.3",
      "platform": "ios",
      "mode": "node"
    },
    "role": "node",
    "scopes": [],
    "caps": ["camera", "canvas", "screen", "location", "voice"],
    "commands": ["camera.snap", "canvas.navigate", "screen.record", "location.get"],
    "permissions": { "camera.capture": true, "screen.record": false },
    "auth": { "token": "…" },
    "locale": "en-US",
    "userAgent": "openclaw-ios/1.2.3",
    "device": {
      "id": "device_fingerprint",
      "publicKey": "…",
      "signature": "…",
      "signedAt": 1737264000000,
      "nonce": "…"
    }
  }
}
```

## Framing

- **Anfrage**: `{type:"req", id, method, params}`
- **Antwort**: `{type:"res", id, ok, payload|error}`
- **Ereignis**: `{type:"event", event, payload, seq?, stateVersion?}`

Methoden mit Seiteneffekten erfordern **Idempotenzschlüssel** (siehe Schema).

## Rollen + Scopes

### Rollen

- `operator` = Control-Plane-Client (CLI/UI/Automatisierung).
- `node` = Capability-Host (camera/screen/canvas/system.run).

### Scopes (operator)

Häufige Scopes:

- `operator.read`
- `operator.write`
- `operator.admin`
- `operator.approvals`
- `operator.pairing`
- `operator.talk.secrets`

`talk.config` mit `includeSecrets: true` erfordert `operator.talk.secrets`
(oder `operator.admin`).

Plugin-registrierte Gateway-RPC-Methoden können ihren eigenen Operator-Scope anfordern, aber
reservierte Core-Admin-Präfixe (`config.*`, `exec.approvals.*`, `wizard.*`,
`update.*`) werden immer zu `operator.admin` aufgelöst.

Der Methodenscope ist nur die erste Hürde. Einige Slash-Befehle, die über
`chat.send` erreicht werden, wenden zusätzlich strengere Prüfungen auf Befehlsebene an. Zum Beispiel
erfordern persistente Schreibvorgänge mit `/config set` und `/config unset` `operator.admin`.

`node.pair.approve` hat zusätzlich zur
Basis-Methodenscope-Prüfung noch eine weitere Scope-Prüfung zur Freigabezeit:

- Anfragen ohne Befehl: `operator.pairing`
- Anfragen mit Nicht-Exec-Node-Befehlen: `operator.pairing` + `operator.write`
- Anfragen, die `system.run`, `system.run.prepare` oder `system.which` enthalten:
  `operator.pairing` + `operator.admin`

### Caps/commands/permissions (node)

Nodes deklarieren beim Verbindungsaufbau Capability-Claims:

- `caps`: allgemeine Capability-Kategorien.
- `commands`: Zulassungsliste für Befehle bei `invoke`.
- `permissions`: granulare Umschalter (z. B. `screen.record`, `camera.capture`).

Das Gateway behandelt diese als **Claims** und erzwingt serverseitige Zulassungslisten.

## Presence

- `system-presence` gibt Einträge zurück, die nach Device-Identität verschlüsselt sind.
- Presence-Einträge enthalten `deviceId`, `roles` und `scopes`, damit UIs pro Device eine einzelne Zeile anzeigen können,
  selbst wenn es sowohl als **operator** als auch als **node** verbunden ist.

## Häufige RPC-Methodenfamilien

Diese Seite ist kein generierter vollständiger Dump, aber die öffentliche WS-Oberfläche ist
breiter als die obigen Handshake-/Auth-Beispiele. Dies sind die wichtigsten Methodenfamilien, die
das Gateway derzeit bereitstellt.

`hello-ok.features.methods` ist eine konservative Discovery-Liste, die aus
`src/gateway/server-methods-list.ts` plus geladenen Plugin-/Kanal-Methodenexporten erstellt wird.
Behandle sie als Feature Discovery, nicht als generierten Dump aller aufrufbaren Hilfsfunktionen,
die in `src/gateway/server-methods/*.ts` implementiert sind.

### System und Identität

- `health` gibt den zwischengespeicherten oder frisch geprüften Gateway-Health-Snapshot zurück.
- `status` gibt die Gateway-Zusammenfassung im Stil von `/status` zurück; sensible Felder werden
  nur für Operator-Clients mit Admin-Scope einbezogen.
- `gateway.identity.get` gibt die Gateway-Device-Identität zurück, die von Relay- und
  Pairing-Abläufen verwendet wird.
- `system-presence` gibt den aktuellen Presence-Snapshot für verbundene
  Operator-/Node-Devices zurück.
- `system-event` hängt ein Systemereignis an und kann den Presence-Kontext
  aktualisieren/übertragen.
- `last-heartbeat` gibt das zuletzt persistent gespeicherte Heartbeat-Ereignis zurück.
- `set-heartbeats` schaltet die Heartbeat-Verarbeitung auf dem Gateway um.

### Modelle und Nutzung

- `models.list` gibt den zur Laufzeit zulässigen Modellkatalog zurück.
- `usage.status` gibt Zusammenfassungen zu Nutzungsfenstern/verbleibendem Kontingent des Providers zurück.
- `usage.cost` gibt aggregierte Zusammenfassungen der Kostennutzung für einen Datumsbereich zurück.
- `doctor.memory.status` gibt die Bereitschaft von Vektorspeicher/Embeddings für den
  aktiven Standard-Agent-Workspace zurück.
- `sessions.usage` gibt Nutzungszusammenfassungen pro Sitzung zurück.
- `sessions.usage.timeseries` gibt Zeitreihennutzung für eine Sitzung zurück.
- `sessions.usage.logs` gibt Nutzungseinträge für eine Sitzung zurück.

### Kanäle und Login-Hilfen

- `channels.status` gibt Statuszusammenfassungen für integrierte und gebündelte Kanäle/Plugins zurück.
- `channels.logout` meldet einen bestimmten Kanal/Account ab, sofern der Kanal
  Logout unterstützt.
- `web.login.start` startet einen QR-/Web-Login-Ablauf für den aktuellen QR-fähigen Web-
  Kanal-Provider.
- `web.login.wait` wartet darauf, dass dieser QR-/Web-Login-Ablauf abgeschlossen wird, und startet bei Erfolg den
  Kanal.
- `push.test` sendet einen APNs-Test-Push an einen registrierten iOS-Node.
- `voicewake.get` gibt die gespeicherten Wake-Word-Trigger zurück.
- `voicewake.set` aktualisiert Wake-Word-Trigger und überträgt die Änderung.

### Messaging und Logs

- `send` ist die direkte RPC für ausgehende Zustellung für kanal-/account-/threadbezogene
  Sendungen außerhalb des Chat-Runners.
- `logs.tail` gibt den konfigurierten Tail des Gateway-Dateilogs mit Cursor/Limit und
  Max-Byte-Steuerung zurück.

### Talk und TTS

- `talk.config` gibt die effektive Talk-Konfigurationsnutzlast zurück; `includeSecrets`
  erfordert `operator.talk.secrets` (oder `operator.admin`).
- `talk.mode` setzt/überträgt den aktuellen Talk-Modus-Status für WebChat-/Control-UI-
  Clients.
- `talk.speak` synthetisiert Sprache über den aktiven Talk-Speech-Provider.
- `tts.status` gibt den TTS-Aktivierungsstatus, den aktiven Provider, Fallback-Provider
  und den Konfigurationsstatus des Providers zurück.
- `tts.providers` gibt das sichtbare TTS-Provider-Inventar zurück.
- `tts.enable` und `tts.disable` schalten den TTS-Präferenzstatus um.
- `tts.setProvider` aktualisiert den bevorzugten TTS-Provider.
- `tts.convert` führt eine einmalige Text-zu-Sprache-Konvertierung aus.

### Secrets, Konfiguration, Update und Wizard

- `secrets.reload` löst aktive SecretRefs erneut auf und tauscht den Laufzeitstatus der Secrets
  nur bei vollständigem Erfolg aus.
- `secrets.resolve` löst Secret-Zuweisungen für Befehlsziele für ein bestimmtes
  Befehls-/Zielset auf.
- `config.get` gibt den aktuellen Konfigurations-Snapshot und Hash zurück.
- `config.set` schreibt eine validierte Konfigurationsnutzlast.
- `config.patch` führt ein Merge einer partiellen Konfigurationsaktualisierung aus.
- `config.apply` validiert + ersetzt die vollständige Konfigurationsnutzlast.
- `config.schema` gibt die Live-Konfigurations-Schema-Nutzlast zurück, die von Control UI und
  CLI-Tooling verwendet wird: Schema, `uiHints`, Version und Generierungsmetadaten, einschließlich
  Plugin- + Kanal-Schema-Metadaten, wenn die Laufzeit sie laden kann. Das Schema
  enthält die Feldmetadaten `title` / `description`, die aus denselben Labels
  und Hilfetexten abgeleitet sind, die von der UI verwendet werden, einschließlich verschachtelter Objekte,
  Wildcards, Array-Elementen und `anyOf` / `oneOf` / `allOf`-Kompositionszweigen, wenn passende
  Felddokumentation vorhanden ist.
- `config.schema.lookup` gibt eine pfadbezogene Lookup-Nutzlast für einen Konfigurationspfad zurück:
  normalisierter Pfad, ein flacher Schema-Knoten, passender Hint + `hintPath` und
  Zusammenfassungen direkter Kindknoten für UI-/CLI-Drill-down.
  - Lookup-Schema-Knoten behalten die benutzerseitige Dokumentation und allgemeine Validierungsfelder:
    `title`, `description`, `type`, `enum`, `const`, `format`, `pattern`,
    numerische/String-/Array-/Objektgrenzen sowie boolesche Flags wie
    `additionalProperties`, `deprecated`, `readOnly`, `writeOnly`.
  - Zusammenfassungen von Kindknoten stellen `key`, normalisierten `path`, `type`, `required`,
    `hasChildren` sowie den passenden `hint` / `hintPath` bereit.
- `update.run` führt den Gateway-Update-Ablauf aus und plant nur dann einen Neustart,
  wenn das Update selbst erfolgreich war.
- `wizard.start`, `wizard.next`, `wizard.status` und `wizard.cancel` stellen den
  Onboarding-Wizard über WS-RPC bereit.

### Bestehende große Familien

#### Agent- und Workspace-Hilfen

- `agents.list` gibt konfigurierte Agent-Einträge zurück.
- `agents.create`, `agents.update` und `agents.delete` verwalten Agent-Datensätze und
  Workspace-Verdrahtung.
- `agents.files.list`, `agents.files.get` und `agents.files.set` verwalten die
  exponierten Bootstrap-Workspace-Dateien für einen Agenten.
- `agent.identity.get` gibt die effektive Assistant-Identität für einen Agenten oder eine
  Sitzung zurück.
- `agent.wait` wartet, bis ein Lauf abgeschlossen ist, und gibt, wenn verfügbar, den terminalen Snapshot zurück.

#### Sitzungssteuerung

- `sessions.list` gibt den aktuellen Sitzungsindex zurück.
- `sessions.subscribe` und `sessions.unsubscribe` schalten Abonnements für Sitzungsänderungsereignisse
  für den aktuellen WS-Client um.
- `sessions.messages.subscribe` und `sessions.messages.unsubscribe` schalten
  Transcript-/Nachrichtenereignis-Abonnements für eine Sitzung um.
- `sessions.preview` gibt begrenzte Transcript-Vorschauen für bestimmte Sitzungsschlüssel zurück.
- `sessions.resolve` löst ein Sitzungsziel auf oder kanonisiert es.
- `sessions.create` erstellt einen neuen Sitzungseintrag.
- `sessions.send` sendet eine Nachricht in eine bestehende Sitzung.
- `sessions.steer` ist die Interrupt-and-Steer-Variante für eine aktive Sitzung.
- `sessions.abort` bricht aktive Arbeit für eine Sitzung ab.
- `sessions.patch` aktualisiert Sitzungsmetadaten/Überschreibungen.
- `sessions.reset`, `sessions.delete` und `sessions.compact` führen Sitzungswartung durch.
- `sessions.get` gibt die vollständige gespeicherte Sitzungszeile zurück.
- Die Chat-Ausführung verwendet weiterhin `chat.history`, `chat.send`, `chat.abort` und
  `chat.inject`.
- `chat.history` ist für UI-Clients anzeige-normalisiert: Inline-Direktiv-Tags werden aus sichtbarem Text entfernt,
  XML-Nutzlasten von Tool-Aufrufen im Klartext (einschließlich
  `<tool_call>...</tool_call>`, `<function_call>...</function_call>`,
  `<tool_calls>...</tool_calls>`, `<function_calls>...</function_calls>` sowie
  abgeschnittener Tool-Call-Blöcke) und geleakte ASCII-/vollbreite Modell-Steuertokens
  werden entfernt, reine Assistant-Zeilen mit Silent-Token wie exakt `NO_REPLY` /
  `no_reply` werden ausgelassen, und übergroße Zeilen können durch Platzhalter ersetzt werden.

#### Device-Pairing und Device-Tokens

- `device.pair.list` gibt ausstehende und genehmigte gekoppelte Devices zurück.
- `device.pair.approve`, `device.pair.reject` und `device.pair.remove` verwalten
  Device-Pairing-Einträge.
- `device.token.rotate` rotiert ein Token eines gekoppelten Devices innerhalb der genehmigten Rollen-
  und Scope-Grenzen.
- `device.token.revoke` widerruft ein Device-Token.

#### Node-Pairing, Invoke und ausstehende Arbeit

- `node.pair.request`, `node.pair.list`, `node.pair.approve`,
  `node.pair.reject` und `node.pair.verify` decken Node-Pairing und Bootstrap-
  Verifikation ab.
- `node.list` und `node.describe` geben bekannte/verbundene Node-Zustände zurück.
- `node.rename` aktualisiert das Label eines gekoppelten Node.
- `node.invoke` leitet einen Befehl an einen verbundenen Node weiter.
- `node.invoke.result` gibt das Ergebnis für eine Invoke-Anfrage zurück.
- `node.event` überträgt Node-originierte Ereignisse zurück in das Gateway.
- `node.canvas.capability.refresh` aktualisiert eingegrenzte Canvas-Capability-Tokens.
- `node.pending.pull` und `node.pending.ack` sind die Queue-APIs für verbundene Nodes.
- `node.pending.enqueue` und `node.pending.drain` verwalten langlebige ausstehende Arbeit
  für Offline-/getrennte Nodes.

#### Freigabefamilien

- `exec.approval.request`, `exec.approval.get`, `exec.approval.list` und
  `exec.approval.resolve` decken einmalige Exec-Freigabeanfragen plus Nachschlagen/Wiederholen
  ausstehender Freigaben ab.
- `exec.approval.waitDecision` wartet auf eine ausstehende Exec-Freigabe und gibt
  die endgültige Entscheidung zurück (oder `null` bei Timeout).
- `exec.approvals.get` und `exec.approvals.set` verwalten Gateway-Exec-Freigabe-
  Richtlinien-Snapshots.
- `exec.approvals.node.get` und `exec.approvals.node.set` verwalten Node-lokale Exec-
  Freigaberichtlinien über Node-Relay-Befehle.
- `plugin.approval.request`, `plugin.approval.list`,
  `plugin.approval.waitDecision` und `plugin.approval.resolve` decken
  plugindefinierte Freigabeabläufe ab.

#### Andere große Familien

- Automatisierung:
  - `wake` plant eine sofortige oder Heartbeat-basierte Wake-Text-Injektion
  - `cron.list`, `cron.status`, `cron.add`, `cron.update`, `cron.remove`,
    `cron.run`, `cron.runs`
- Skills/Tools: `skills.*`, `tools.catalog`, `tools.effective`

### Häufige Ereignisfamilien

- `chat`: UI-Chat-Aktualisierungen wie `chat.inject` und andere reine Transcript-
  Chat-Ereignisse.
- `session.message` und `session.tool`: Transcript-/Event-Stream-Aktualisierungen für eine
  abonnierte Sitzung.
- `sessions.changed`: Sitzungsindex oder -metadaten wurden geändert.
- `presence`: Aktualisierungen des System-Presence-Snapshots.
- `tick`: periodisches Keepalive-/Liveness-Ereignis.
- `health`: Aktualisierung des Gateway-Health-Snapshots.
- `heartbeat`: Aktualisierung des Heartbeat-Ereignisstreams.
- `cron`: Änderungsereignis für Cron-Lauf/Job.
- `shutdown`: Gateway-Shutdown-Benachrichtigung.
- `node.pair.requested` / `node.pair.resolved`: Lebenszyklus des Node-Pairings.
- `node.invoke.request`: Broadcast einer Node-Invoke-Anfrage.
- `device.pair.requested` / `device.pair.resolved`: Lebenszyklus des gekoppelten Devices.
- `voicewake.changed`: Wake-Word-Trigger-Konfiguration wurde geändert.
- `exec.approval.requested` / `exec.approval.resolved`: Lebenszyklus der Exec-
  Freigabe.
- `plugin.approval.requested` / `plugin.approval.resolved`: Lebenszyklus der Plugin-Freigabe.

### Node-Hilfsmethoden

- Nodes können `skills.bins` aufrufen, um die aktuelle Liste der Skill-Executables
  für Auto-Allow-Prüfungen abzurufen.

### Operator-Hilfsmethoden

- Operatoren können `tools.catalog` (`operator.read`) aufrufen, um den Laufzeit-Toolkatalog für einen
  Agenten abzurufen. Die Antwort enthält gruppierte Tools und Provenance-Metadaten:
  - `source`: `core` oder `plugin`
  - `pluginId`: Plugin-Eigentümer, wenn `source="plugin"`
  - `optional`: ob ein Plugin-Tool optional ist
- Operatoren können `tools.effective` (`operator.read`) aufrufen, um das zur Laufzeit effektiv geltende Tool-
  Inventar für eine Sitzung abzurufen.
  - `sessionKey` ist erforderlich.
  - Das Gateway leitet den vertrauenswürdigen Laufzeitkontext serverseitig aus der Sitzung ab, statt
    vom Aufrufer bereitgestellte Auth- oder Delivery-Kontexte zu akzeptieren.
  - Die Antwort ist sitzungsbezogen und spiegelt wider, was die aktive Unterhaltung gerade verwenden kann,
    einschließlich Core-, Plugin- und Kanal-Tools.
- Operatoren können `skills.status` (`operator.read`) aufrufen, um das sichtbare
  Skill-Inventar für einen Agenten abzurufen.
  - `agentId` ist optional; lasse es weg, um den Standard-Agent-Workspace zu lesen.
  - Die Antwort enthält Eignung, fehlende Voraussetzungen, Konfigurationsprüfungen und
    bereinigte Installationsoptionen, ohne rohe Secret-Werte offenzulegen.
- Operatoren können `skills.search` und `skills.detail` (`operator.read`) für
  ClawHub-Discovery-Metadaten aufrufen.
- Operatoren können `skills.install` (`operator.admin`) in zwei Modi aufrufen:
  - ClawHub-Modus: `{ source: "clawhub", slug, version?, force? }` installiert einen
    Skill-Ordner in das `skills/`-Verzeichnis des Standard-Agent-Workspace.
  - Gateway-Installer-Modus: `{ name, installId, dangerouslyForceUnsafeInstall?, timeoutMs? }`
    führt eine deklarierte Aktion `metadata.openclaw.install` auf dem Gateway-Host aus.
- Operatoren können `skills.update` (`operator.admin`) in zwei Modi aufrufen:
  - Der ClawHub-Modus aktualisiert einen verfolgten Slug oder alle verfolgten ClawHub-Installationen im
    Standard-Agent-Workspace.
  - Der Konfigurationsmodus patched Werte unter `skills.entries.<skillKey>` wie `enabled`,
    `apiKey` und `env`.

## Exec-Freigaben

- Wenn eine Exec-Anfrage Freigabe benötigt, überträgt das Gateway `exec.approval.requested`.
- Operator-Clients lösen dies durch Aufruf von `exec.approval.resolve` auf (erfordert `operator.approvals`-Scope).
- Für `host=node` muss `exec.approval.request` `systemRunPlan` enthalten (kanonische `argv`/`cwd`/`rawCommand`/Sitzungsmetadaten). Anfragen ohne `systemRunPlan` werden abgelehnt.
- Nach der Freigabe verwenden weitergeleitete `node.invoke system.run`-Aufrufe diesen kanonischen
  `systemRunPlan` als maßgeblichen Befehl-/cwd-/Sitzungskontext.
- Wenn ein Aufrufer `command`, `rawCommand`, `cwd`, `agentId` oder
  `sessionKey` zwischen `prepare` und dem endgültig freigegebenen `system.run`-Forward verändert, lehnt das
  Gateway den Lauf ab, statt der veränderten Nutzlast zu vertrauen.

## Agent-Delivery-Fallback

- `agent`-Anfragen können `deliver=true` enthalten, um eine ausgehende Zustellung anzufordern.
- `bestEffortDeliver=false` behält das strikte Verhalten bei: nicht auflösbare oder nur intern verfügbare Delivery-Ziele geben `INVALID_REQUEST` zurück.
- `bestEffortDeliver=true` erlaubt Fallback auf reine Sitzungsausführung, wenn keine extern zustellbare Route aufgelöst werden kann (zum Beispiel interne/WebChat-Sitzungen oder mehrdeutige Multi-Kanal-Konfigurationen).

## Versionierung

- `PROTOCOL_VERSION` befindet sich in `src/gateway/protocol/schema.ts`.
- Clients senden `minProtocol` + `maxProtocol`; der Server lehnt Abweichungen ab.
- Schemata + Modelle werden aus TypeBox-Definitionen generiert:
  - `pnpm protocol:gen`
  - `pnpm protocol:gen:swift`
  - `pnpm protocol:check`

## Auth

- Gateway-Authentifizierung mit Shared Secret verwendet `connect.params.auth.token` oder
  `connect.params.auth.password`, abhängig vom konfigurierten Auth-Modus.
- Identitätstragende Modi wie Tailscale Serve
  (`gateway.auth.allowTailscale: true`) oder nicht-loopback
  `gateway.auth.mode: "trusted-proxy"` erfüllen die Connect-Auth-Prüfung über
  Request-Header statt über `connect.params.auth.*`.
- Private-Ingress `gateway.auth.mode: "none"` überspringt Shared-Secret-Connect-Auth
  vollständig; diesen Modus nicht über öffentlichen/nicht vertrauenswürdigen Ingress exponieren.
- Nach dem Pairing gibt das Gateway ein **Device-Token** aus, das auf Verbindungs-
  Rolle + Scopes begrenzt ist. Es wird in `hello-ok.auth.deviceToken` zurückgegeben und sollte vom Client
  für zukünftige Verbindungen persistent gespeichert werden.
- Clients sollten das primäre `hello-ok.auth.deviceToken` nach jeder
  erfolgreichen Verbindung persistent speichern.
- Beim Wiederverbinden mit diesem **gespeicherten** Device-Token sollte auch der gespeicherte
  genehmigte Scope-Satz für dieses Token wiederverwendet werden. Dadurch bleibt bereits gewährter
  read/probe/status-Zugriff erhalten und es wird vermieden, dass Wiederverbindungen stillschweigend auf einen
  engeren impliziten Admin-only-Scope zurückfallen.
- Die normale Reihenfolge für Connect-Auth ist zuerst explizites Shared Token/Passwort, dann
  explizites `deviceToken`, dann gespeichertes pro-Device-Token und dann Bootstrap-Token.
- Zusätzliche Einträge in `hello-ok.auth.deviceTokens` sind Bootstrap-Handoff-Tokens.
  Speichere sie nur dann persistent, wenn die Verbindung Bootstrap-Auth auf einem vertrauenswürdigen Transport wie
  `wss://` oder Loopback/lokalem Pairing verwendet hat.
- Wenn ein Client ein **explizites** `deviceToken` oder explizite `scopes` bereitstellt, bleibt dieser
  vom Aufrufer angeforderte Scope-Satz maßgeblich; zwischengespeicherte Scopes werden nur
  wiederverwendet, wenn der Client das gespeicherte pro-Device-Token wiederverwendet.
- Device-Tokens können über `device.token.rotate` und
  `device.token.revoke` rotiert/widerrufen werden (erfordert `operator.pairing`-Scope).
- Die Ausgabe/Rotation von Tokens bleibt auf den genehmigten Rollensatz begrenzt, der im
  Pairing-Eintrag dieses Devices gespeichert ist; durch Rotation eines Tokens kann das Device nicht auf eine
  Rolle erweitert werden, die durch die Pairing-Freigabe nie gewährt wurde.
- Für Sitzungen mit Token gekoppelter Devices ist die Device-Verwaltung selbstbegrenzt, sofern der
  Aufrufer nicht zusätzlich `operator.admin` hat: Nicht-Admin-Aufrufer können nur ihren **eigenen**
  Device-Eintrag entfernen/widerrufen/rotieren.
- `device.token.rotate` prüft außerdem den angeforderten Operator-Scope-Satz gegen die
  aktuellen Sitzungsscopes des Aufrufers. Nicht-Admin-Aufrufer können ein Token nicht in einen
  breiteren Operator-Scope-Satz rotieren, als sie bereits besitzen.
- Auth-Fehler enthalten `error.details.code` plus Wiederherstellungshinweise:
  - `error.details.canRetryWithDeviceToken` (Boolean)
  - `error.details.recommendedNextStep` (`retry_with_device_token`, `update_auth_configuration`, `update_auth_credentials`, `wait_then_retry`, `review_auth_configuration`)
- Client-Verhalten für `AUTH_TOKEN_MISMATCH`:
  - Vertrauenswürdige Clients können einen begrenzten Wiederholungsversuch mit einem zwischengespeicherten pro-Device-Token unternehmen.
  - Wenn dieser Wiederholungsversuch fehlschlägt, sollten Clients automatische Reconnect-Schleifen stoppen und Hinweise für notwendige Operator-Aktionen anzeigen.

## Device-Identität + Pairing

- Nodes sollten eine stabile Device-Identität (`device.id`) einschließen, die aus dem
  Fingerabdruck eines Schlüsselpaares abgeleitet wird.
- Gateways geben Tokens pro Device + Rolle aus.
- Pairing-Freigaben sind für neue Device-IDs erforderlich, sofern lokale Auto-Freigabe
  nicht aktiviert ist.
- Die Auto-Freigabe für Pairing ist auf direkte lokale Loopback-Verbindungen zentriert.
- OpenClaw hat außerdem einen engen backend-/containerlokalen Self-Connect-Pfad für
  vertrauenswürdige Shared-Secret-Hilfsabläufe.
- Verbindungen aus derselben Tailnet- oder LAN-Hostumgebung werden für Pairing weiterhin als remote behandelt und
  erfordern Freigabe.
- Alle WS-Clients müssen beim `connect` die `device`-Identität einbeziehen (operator + node).
  Control UI kann sie nur in diesen Modi weglassen:
  - `gateway.controlUi.allowInsecureAuth=true` für localhost-only-Kompatibilität mit unsicherem HTTP.
  - erfolgreiche Operator-Control-UI-Authentifizierung mit `gateway.auth.mode: "trusted-proxy"`.
  - `gateway.controlUi.dangerouslyDisableDeviceAuth=true` (Break-Glass, starke Sicherheitsabsenkung).
- Alle Verbindungen müssen die serverseitig bereitgestellte `connect.challenge`-Nonce signieren.

### Migrationsdiagnostik für Device-Auth

Für Legacy-Clients, die noch das Signaturverhalten vor der Challenge verwenden, liefert `connect` jetzt
`DEVICE_AUTH_*`-Detailcodes unter `error.details.code` mit einem stabilen `error.details.reason`.

Häufige Migrationsfehler:

| Meldung                     | details.code                     | details.reason           | Bedeutung                                          |
| --------------------------- | -------------------------------- | ------------------------ | -------------------------------------------------- |
| `device nonce required`     | `DEVICE_AUTH_NONCE_REQUIRED`     | `device-nonce-missing`   | Client hat `device.nonce` ausgelassen (oder leer gesendet). |
| `device nonce mismatch`     | `DEVICE_AUTH_NONCE_MISMATCH`     | `device-nonce-mismatch`  | Client hat mit einer veralteten/falschen Nonce signiert. |
| `device signature invalid`  | `DEVICE_AUTH_SIGNATURE_INVALID`  | `device-signature`       | Die Signaturnutzlast stimmt nicht mit der v2-Nutzlast überein. |
| `device signature expired`  | `DEVICE_AUTH_SIGNATURE_EXPIRED`  | `device-signature-stale` | Der signierte Zeitstempel liegt außerhalb der zulässigen Abweichung. |
| `device identity mismatch`  | `DEVICE_AUTH_DEVICE_ID_MISMATCH` | `device-id-mismatch`     | `device.id` stimmt nicht mit dem Fingerabdruck des Public Key überein. |
| `device public key invalid` | `DEVICE_AUTH_PUBLIC_KEY_INVALID` | `device-public-key`      | Format/Kanonisierung des Public Key ist fehlgeschlagen. |

Migrationsziel:

- Immer auf `connect.challenge` warten.
- Die v2-Nutzlast signieren, die die Server-Nonce enthält.
- Dieselbe Nonce in `connect.params.device.nonce` senden.
- Die bevorzugte Signaturnutzlast ist `v3`, die zusätzlich zu den Feldern für Device/Client/Rolle/Scopes/Token/Nonce auch `platform` und `deviceFamily` bindet.
- Legacy-`v2`-Signaturen bleiben aus Kompatibilitätsgründen akzeptiert, aber das Pinning gepaarter Device-
  Metadaten steuert weiterhin die Befehlsrichtlinie bei Wiederverbindungen.

## TLS + Pinning

- TLS wird für WS-Verbindungen unterstützt.
- Clients können optional den Fingerabdruck des Gateway-Zertifikats pinnen (siehe `gateway.tls`-
  Konfiguration plus `gateway.remote.tlsFingerprint` oder CLI `--tls-fingerprint`).

## Scope

Dieses Protokoll stellt die **vollständige Gateway-API** bereit (Status, Kanäle, Modelle, Chat,
Agent, Sitzungen, Nodes, Freigaben usw.). Die genaue Oberfläche wird durch die
TypeBox-Schemata in `src/gateway/protocol/schema.ts` definiert.
