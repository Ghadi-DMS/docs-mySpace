---
read_when:
    - Einrichten von Matrix in OpenClaw
    - Konfigurieren von Matrix-E2EE und Verifizierung
summary: Matrix-Supportstatus, Einrichtung und Konfigurationsbeispiele
title: Matrix
x-i18n:
    generated_at: "2026-04-08T02:16:54Z"
    model: gpt-5.4
    provider: openai
    source_hash: ec926df79a41fa296d63f0ec7219d0f32e075628d76df9ea490e93e4c5030f83
    source_path: channels/matrix.md
    workflow: 15
---

# Matrix

Matrix ist das gebündelte Matrix-Kanal-Plugin für OpenClaw.
Es verwendet das offizielle `matrix-js-sdk` und unterstützt DMs, Räume, Threads, Medien, Reaktionen, Umfragen, Standort und E2EE.

## Gebündeltes Plugin

Matrix wird in aktuellen OpenClaw-Releases als gebündeltes Plugin ausgeliefert, daher benötigen normale
paketierte Builds keine separate Installation.

Wenn Sie eine ältere Version oder eine benutzerdefinierte Installation verwenden, die Matrix ausschließt, installieren Sie
es manuell:

Von npm installieren:

```bash
openclaw plugins install @openclaw/matrix
```

Von einem lokalen Checkout installieren:

```bash
openclaw plugins install ./path/to/local/matrix-plugin
```

Siehe [Plugins](/de/tools/plugin) für das Plugin-Verhalten und die Installationsregeln.

## Einrichtung

1. Stellen Sie sicher, dass das Matrix-Plugin verfügbar ist.
   - Aktuelle paketierte OpenClaw-Releases enthalten es bereits.
   - Ältere/benutzerdefinierte Installationen können es manuell mit den obigen Befehlen hinzufügen.
2. Erstellen Sie ein Matrix-Konto auf Ihrem Homeserver.
3. Konfigurieren Sie `channels.matrix` mit entweder:
   - `homeserver` + `accessToken`, oder
   - `homeserver` + `userId` + `password`.
4. Starten Sie das Gateway neu.
5. Starten Sie eine DM mit dem Bot oder laden Sie ihn in einen Raum ein.
   - Neue Matrix-Einladungen funktionieren nur, wenn `channels.matrix.autoJoin` sie zulässt.

Interaktive Einrichtungswege:

```bash
openclaw channels add
openclaw configure --section channels
```

Wonach der Matrix-Assistent tatsächlich fragt:

- Homeserver-URL
- Authentifizierungsmethode: Zugriffstoken oder Passwort
- Benutzer-ID nur, wenn Sie Passwortauthentifizierung auswählen
- optionaler Gerätename
- ob E2EE aktiviert werden soll
- ob der Matrix-Raumzugriff jetzt konfiguriert werden soll
- ob der automatische Beitritt zu Matrix-Einladungen jetzt konfiguriert werden soll
- wenn der automatische Beitritt zu Einladungen aktiviert ist, ob er `allowlist`, `always` oder `off` sein soll

Wichtiges Verhalten des Assistenten:

- Wenn für das ausgewählte Konto bereits Matrix-Auth-Umgebungsvariablen vorhanden sind und für dieses Konto noch keine Authentifizierung in der Konfiguration gespeichert ist, bietet der Assistent eine Umgebungsvariablen-Verknüpfung an, damit die Einrichtung die Authentifizierung in Umgebungsvariablen behalten kann, anstatt Geheimnisse in die Konfiguration zu kopieren.
- Wenn Sie interaktiv ein weiteres Matrix-Konto hinzufügen, wird der eingegebene Kontoname in die Konto-ID normalisiert, die in Konfiguration und Umgebungsvariablen verwendet wird. Beispielsweise wird aus `Ops Bot` `ops-bot`.
- Abfragen für DM-Zulassungslisten akzeptieren sofort vollständige `@user:server`-Werte. Anzeigenamen funktionieren nur, wenn die Live-Verzeichnissuche genau einen Treffer findet; andernfalls fordert der Assistent Sie auf, es mit einer vollständigen Matrix-ID erneut zu versuchen.
- Abfragen für Raum-Zulassungslisten akzeptieren Raum-IDs und Aliase direkt. Sie können auch Namen beigetretener Räume live auflösen, aber nicht aufgelöste Namen werden bei der Einrichtung nur wie eingegeben beibehalten und später von der Laufzeitauflösung der Zulassungsliste ignoriert. Bevorzugen Sie `!room:server` oder `#alias:server`.
- Der Assistent zeigt jetzt vor dem Schritt zum automatischen Beitritt bei Einladungen eine ausdrückliche Warnung an, da `channels.matrix.autoJoin` standardmäßig auf `off` steht; Agents treten eingeladenen Räumen oder neuen Einladungen im DM-Stil nicht bei, wenn Sie dies nicht festlegen.
- Im Zulassungslistenmodus für automatischen Beitritt bei Einladungen verwenden Sie nur stabile Einladungsziele: `!roomId:server`, `#alias:server` oder `*`. Einfache Raumnamen werden abgelehnt.
- Die Laufzeitidentität von Raum/Sitzung verwendet die stabile Matrix-Raum-ID. Im Raum deklarierte Aliase werden nur als Suchparameter verwendet, nicht als langfristiger Sitzungsschlüssel oder stabile Gruppenidentität.
- Um Raumnamen vor dem Speichern aufzulösen, verwenden Sie `openclaw channels resolve --channel matrix "Project Room"`.

<Warning>
`channels.matrix.autoJoin` ist standardmäßig `off`.

Wenn Sie es nicht setzen, tritt der Bot eingeladenen Räumen oder neuen Einladungen im DM-Stil nicht bei, sodass er nicht in neuen Gruppen oder eingeladenen DMs erscheint, sofern Sie nicht zuerst manuell beitreten.

Setzen Sie `autoJoin: "allowlist"` zusammen mit `autoJoinAllowlist`, um einzuschränken, welche Einladungen akzeptiert werden, oder setzen Sie `autoJoin: "always"`, wenn er jeder Einladung beitreten soll.

Im Modus `allowlist` akzeptiert `autoJoinAllowlist` nur `!roomId:server`, `#alias:server` oder `*`.
</Warning>

Beispiel für eine Zulassungsliste:

```json5
{
  channels: {
    matrix: {
      autoJoin: "allowlist",
      autoJoinAllowlist: ["!ops:example.org", "#support:example.org"],
      groups: {
        "!ops:example.org": {
          requireMention: true,
        },
      },
    },
  },
}
```

Jeder Einladung beitreten:

```json5
{
  channels: {
    matrix: {
      autoJoin: "always",
    },
  },
}
```

Minimale tokenbasierte Einrichtung:

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      dm: { policy: "pairing" },
    },
  },
}
```

Passwortbasierte Einrichtung (Token wird nach der Anmeldung zwischengespeichert):

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      userId: "@bot:example.org",
      password: "replace-me", // pragma: allowlist secret
      deviceName: "OpenClaw Gateway",
    },
  },
}
```

Matrix speichert zwischengespeicherte Anmeldedaten in `~/.openclaw/credentials/matrix/`.
Das Standardkonto verwendet `credentials.json`; benannte Konten verwenden `credentials-<account>.json`.
Wenn dort zwischengespeicherte Anmeldedaten vorhanden sind, betrachtet OpenClaw Matrix für Einrichtung, Doctor und die Erkennung des Kanalstatus als konfiguriert, auch wenn die aktuelle Authentifizierung nicht direkt in der Konfiguration gesetzt ist.

Äquivalente Umgebungsvariablen (werden verwendet, wenn der Konfigurationsschlüssel nicht gesetzt ist):

- `MATRIX_HOMESERVER`
- `MATRIX_ACCESS_TOKEN`
- `MATRIX_USER_ID`
- `MATRIX_PASSWORD`
- `MATRIX_DEVICE_ID`
- `MATRIX_DEVICE_NAME`

Für Nicht-Standardkonten verwenden Sie kontospezifische Umgebungsvariablen:

- `MATRIX_<ACCOUNT_ID>_HOMESERVER`
- `MATRIX_<ACCOUNT_ID>_ACCESS_TOKEN`
- `MATRIX_<ACCOUNT_ID>_USER_ID`
- `MATRIX_<ACCOUNT_ID>_PASSWORD`
- `MATRIX_<ACCOUNT_ID>_DEVICE_ID`
- `MATRIX_<ACCOUNT_ID>_DEVICE_NAME`

Beispiel für Konto `ops`:

- `MATRIX_OPS_HOMESERVER`
- `MATRIX_OPS_ACCESS_TOKEN`

Für die normalisierte Konto-ID `ops-bot` verwenden Sie:

- `MATRIX_OPS_X2D_BOT_HOMESERVER`
- `MATRIX_OPS_X2D_BOT_ACCESS_TOKEN`

Matrix maskiert Satzzeichen in Konto-IDs, damit kontobezogene Umgebungsvariablen kollisionsfrei bleiben.
Zum Beispiel wird `-` zu `_X2D_`, sodass `ops-prod` auf `MATRIX_OPS_X2D_PROD_*` abgebildet wird.

Der interaktive Assistent bietet die Umgebungsvariablen-Verknüpfung nur an, wenn diese Auth-Umgebungsvariablen bereits vorhanden sind und für das ausgewählte Konto noch keine Matrix-Authentifizierung in der Konfiguration gespeichert ist.

## Konfigurationsbeispiel

Dies ist eine praktische Basiskonfiguration mit DM-Pairing, Raum-Zulassungsliste und aktiviertem E2EE:

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      encryption: true,

      dm: {
        policy: "pairing",
        sessionScope: "per-room",
        threadReplies: "off",
      },

      groupPolicy: "allowlist",
      groupAllowFrom: ["@admin:example.org"],
      groups: {
        "!roomid:example.org": {
          requireMention: true,
        },
      },

      autoJoin: "allowlist",
      autoJoinAllowlist: ["!roomid:example.org"],
      threadReplies: "inbound",
      replyToMode: "off",
      streaming: "partial",
    },
  },
}
```

`autoJoin` gilt allgemein für Matrix-Einladungen, nicht nur für Raum-/Gruppeneinladungen.
Dazu gehören auch neue Einladungen im DM-Stil. Zum Zeitpunkt der Einladung weiß OpenClaw nicht zuverlässig, ob der
eingeladene Raum letztlich als DM oder als Gruppe behandelt wird, daher durchlaufen alle Einladungen zuerst dieselbe
`autoJoin`-Entscheidung. `dm.policy` gilt weiterhin, nachdem der Bot dem Raum beigetreten ist und der Raum
als DM klassifiziert wurde, sodass `autoJoin` das Beitrittsverhalten steuert, während `dm.policy` das Antwort-/Zugriffs-
verhalten steuert.

## Streaming-Vorschauen

Antwort-Streaming in Matrix ist Opt-in.

Setzen Sie `channels.matrix.streaming` auf `"partial"`, wenn OpenClaw eine einzelne Live-Vorschau-
Antwort senden, diese Vorschau während der Textgenerierung durch das Modell direkt bearbeiten und sie
abschließend finalisieren soll, sobald die Antwort fertig ist:

```json5
{
  channels: {
    matrix: {
      streaming: "partial",
    },
  },
}
```

- `streaming: "off"` ist die Standardeinstellung. OpenClaw wartet auf die endgültige Antwort und sendet sie einmalig.
- `streaming: "partial"` erstellt eine bearbeitbare Vorschau-Nachricht für den aktuellen Assistant-Block unter Verwendung normaler Matrix-Textnachrichten. Dadurch bleibt das bisherige Benachrichtigungsverhalten von Matrix mit Vorschau zuerst erhalten, sodass Standard-Clients möglicherweise auf den ersten gestreamten Vorschautext statt auf den fertigen Block benachrichtigen.
- `streaming: "quiet"` erstellt einen bearbeitbaren stillen Vorschauhinweis für den aktuellen Assistant-Block. Verwenden Sie dies nur, wenn Sie zusätzlich Empfänger-Push-Regeln für finalisierte Vorschau-Bearbeitungen konfigurieren.
- `blockStreaming: true` aktiviert separate Matrix-Fortschrittsnachrichten. Wenn Vorschau-Streaming aktiviert ist, behält Matrix den Live-Entwurf für den aktuellen Block bei und bewahrt abgeschlossene Blöcke als separate Nachrichten.
- Wenn Vorschau-Streaming aktiviert ist und `blockStreaming` deaktiviert ist, bearbeitet Matrix den Live-Entwurf direkt und finalisiert dasselbe Ereignis, wenn der Block oder Zug beendet ist.
- Wenn die Vorschau nicht mehr in ein einzelnes Matrix-Ereignis passt, beendet OpenClaw das Vorschau-Streaming und greift auf die normale endgültige Zustellung zurück.
- Medienantworten senden Anhänge weiterhin normal. Wenn eine veraltete Vorschau nicht mehr sicher wiederverwendet werden kann, redigiert OpenClaw sie, bevor die endgültige Medienantwort gesendet wird.
- Vorschau-Bearbeitungen verursachen zusätzliche Matrix-API-Aufrufe. Lassen Sie Streaming deaktiviert, wenn Sie das konservativste Rate-Limit-Verhalten wünschen.

`blockStreaming` aktiviert Entwurfsvorschauen nicht von selbst.
Verwenden Sie `streaming: "partial"` oder `streaming: "quiet"` für Vorschau-Bearbeitungen; fügen Sie dann `blockStreaming: true` nur hinzu, wenn auch abgeschlossene Assistant-Blöcke als separate Fortschrittsnachrichten sichtbar bleiben sollen.

Wenn Sie Matrix-Standardbenachrichtigungen ohne benutzerdefinierte Push-Regeln benötigen, verwenden Sie `streaming: "partial"` für Vorschau-zuerst-Verhalten oder lassen Sie `streaming` für nur endgültige Zustellung deaktiviert. Mit `streaming: "off"`:

- `blockStreaming: true` sendet jeden fertigen Block als normale benachrichtigende Matrix-Nachricht.
- `blockStreaming: false` sendet nur die endgültige vollständige Antwort als normale benachrichtigende Matrix-Nachricht.

### Selbstgehostete Push-Regeln für stille finalisierte Vorschauen

Wenn Sie Ihre eigene Matrix-Infrastruktur betreiben und stille Vorschauen nur benachrichtigen sollen, wenn ein Block oder
eine endgültige Antwort fertig ist, setzen Sie `streaming: "quiet"` und fügen Sie eine benutzerspezifische Push-Regel für finalisierte Vorschau-Bearbeitungen hinzu.

Dies ist in der Regel eine Empfänger-Benutzereinrichtung, keine globale Homeserver-Konfigurationsänderung:

Kurzübersicht, bevor Sie beginnen:

- Empfängerbenutzer = die Person, die die Benachrichtigung erhalten soll
- Bot-Benutzer = das OpenClaw-Matrix-Konto, das die Antwort sendet
- verwenden Sie für die folgenden API-Aufrufe das Zugriffstoken des Empfängerbenutzers
- gleichen Sie `sender` in der Push-Regel mit der vollständigen MXID des Bot-Benutzers ab

1. Konfigurieren Sie OpenClaw für stille Vorschauen:

```json5
{
  channels: {
    matrix: {
      streaming: "quiet",
    },
  },
}
```

2. Stellen Sie sicher, dass das Empfängerkonto bereits normale Matrix-Push-Benachrichtigungen erhält. Regeln für stille Vorschauen
   funktionieren nur, wenn dieser Benutzer bereits funktionierende Pusher/Geräte hat.

3. Beschaffen Sie das Zugriffstoken des Empfängerbenutzers.
   - Verwenden Sie das Token des empfangenden Benutzers, nicht das Token des Bots.
   - Ein vorhandenes Client-Sitzungstoken wiederzuverwenden ist normalerweise am einfachsten.
   - Wenn Sie ein neues Token ausstellen müssen, können Sie sich über die standardmäßige Matrix-Client-Server-API anmelden:

```bash
curl -sS -X POST \
  "https://matrix.example.org/_matrix/client/v3/login" \
  -H "Content-Type: application/json" \
  --data '{
    "type": "m.login.password",
    "identifier": {
      "type": "m.id.user",
      "user": "@alice:example.org"
    },
    "password": "REDACTED"
  }'
```

4. Verifizieren Sie, dass das Empfängerkonto bereits Pusher hat:

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushers"
```

Wenn dies keine aktiven Pusher/Geräte zurückgibt, beheben Sie zuerst die normalen Matrix-Benachrichtigungen, bevor Sie die
unten stehende OpenClaw-Regel hinzufügen.

OpenClaw kennzeichnet finalisierte Bearbeitungen von Nur-Text-Vorschauen mit:

```json
{
  "com.openclaw.finalized_preview": true
}
```

5. Erstellen Sie für jedes Empfängerkonto, das diese Benachrichtigungen erhalten soll, eine Override-Push-Regel:

```bash
curl -sS -X PUT \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname" \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  --data '{
    "conditions": [
      { "kind": "event_match", "key": "type", "pattern": "m.room.message" },
      {
        "kind": "event_property_is",
        "key": "content.m\\.relates_to.rel_type",
        "value": "m.replace"
      },
      {
        "kind": "event_property_is",
        "key": "content.com\\.openclaw\\.finalized_preview",
        "value": true
      },
      { "kind": "event_match", "key": "sender", "pattern": "@bot:example.org" }
    ],
    "actions": [
      "notify",
      { "set_tweak": "sound", "value": "default" },
      { "set_tweak": "highlight", "value": false }
    ]
  }'
```

Ersetzen Sie diese Werte, bevor Sie den Befehl ausführen:

- `https://matrix.example.org`: Ihre Homeserver-Basis-URL
- `$USER_ACCESS_TOKEN`: das Zugriffstoken des empfangenden Benutzers
- `openclaw-finalized-preview-botname`: eine für diesen Bot und diesen empfangenden Benutzer eindeutige Regel-ID
- `@bot:example.org`: Ihre OpenClaw-Matrix-Bot-MXID, nicht die MXID des empfangenden Benutzers

Wichtig für Setups mit mehreren Bots:

- Push-Regeln werden über `ruleId` identifiziert. Wenn Sie `PUT` erneut gegen dieselbe Regel-ID ausführen, wird diese eine Regel aktualisiert.
- Wenn ein empfangender Benutzer für mehrere OpenClaw-Matrix-Bot-Konten benachrichtigt werden soll, erstellen Sie pro Bot eine Regel mit einer eindeutigen Regel-ID für jede Sender-Übereinstimmung.
- Ein einfaches Muster ist `openclaw-finalized-preview-<botname>`, etwa `openclaw-finalized-preview-ops` oder `openclaw-finalized-preview-support`.

Die Regel wird gegen den Ereignisabsender ausgewertet:

- authentifizieren Sie sich mit dem Token des empfangenden Benutzers
- gleichen Sie `sender` mit der OpenClaw-Bot-MXID ab

6. Verifizieren Sie, dass die Regel vorhanden ist:

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname"
```

7. Testen Sie eine gestreamte Antwort. Im stillen Modus sollte der Raum eine stille Entwurfsvorschau anzeigen, und die endgültige
   direkte Bearbeitung sollte benachrichtigen, sobald der Block oder Zug abgeschlossen ist.

Wenn Sie die Regel später entfernen müssen, löschen Sie dieselbe Regel-ID mit dem Token des empfangenden Benutzers:

```bash
curl -sS -X DELETE \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname"
```

Hinweise:

- Erstellen Sie die Regel mit dem Zugriffstoken des empfangenden Benutzers, nicht mit dem Token des Bots.
- Neue benutzerdefinierte `override`-Regeln werden vor den standardmäßigen Unterdrückungsregeln eingefügt, daher ist kein zusätzlicher Ordnungsparameter erforderlich.
- Dies betrifft nur Bearbeitungen von Nur-Text-Vorschauen, die OpenClaw sicher direkt finalisieren kann. Medien-Fallbacks und Fallbacks bei veralteten Vorschauen verwenden weiterhin die normale Matrix-Zustellung.
- Wenn `GET /_matrix/client/v3/pushers` keine Pusher anzeigt, hat der Benutzer für dieses Konto/Gerät noch keine funktionierende Matrix-Push-Zustellung.

#### Synapse

Für Synapse reicht die obige Einrichtung normalerweise bereits aus:

- Für finalisierte OpenClaw-Vorschau-Benachrichtigungen ist keine spezielle Änderung in `homeserver.yaml` erforderlich.
- Wenn Ihre Synapse-Bereitstellung bereits normale Matrix-Push-Benachrichtigungen sendet, ist der wichtigste Einrichtungsschritt das obige Benutzer-Token plus der Aufruf von `pushrules`.
- Wenn Sie Synapse hinter einem Reverse-Proxy oder Workern betreiben, stellen Sie sicher, dass `/_matrix/client/.../pushrules/` Synapse korrekt erreicht.
- Wenn Sie Synapse-Worker betreiben, stellen Sie sicher, dass Pusher fehlerfrei arbeiten. Die Push-Zustellung wird vom Hauptprozess oder von `synapse.app.pusher` / konfigurierten Pusher-Workern verarbeitet.

#### Tuwunel

Für Tuwunel verwenden Sie denselben Einrichtungsablauf und denselben oben gezeigten API-Aufruf für Push-Regeln:

- Für die Kennzeichnung finalisierter Vorschauen selbst ist keine Tuwunel-spezifische Konfiguration erforderlich.
- Wenn normale Matrix-Benachrichtigungen für diesen Benutzer bereits funktionieren, ist der wichtigste Einrichtungsschritt das obige Benutzer-Token plus der Aufruf von `pushrules`.
- Wenn Benachrichtigungen zu verschwinden scheinen, während der Benutzer auf einem anderen Gerät aktiv ist, prüfen Sie, ob `suppress_push_when_active` aktiviert ist. Tuwunel hat diese Option in Tuwunel 1.4.2 am 12. September 2025 hinzugefügt, und sie kann Pushes an andere Geräte absichtlich unterdrücken, während ein Gerät aktiv ist.

## Verschlüsselung und Verifizierung

In verschlüsselten (E2EE-)Räumen verwenden ausgehende Bildereignisse `thumbnail_file`, sodass Bildvorschauen zusammen mit dem vollständigen Anhang verschlüsselt werden. Unverschlüsselte Räume verwenden weiterhin einfaches `thumbnail_url`. Es ist keine Konfiguration erforderlich — das Plugin erkennt den E2EE-Status automatisch.

### Bot-zu-Bot-Räume

Standardmäßig werden Matrix-Nachrichten anderer konfigurierter OpenClaw-Matrix-Konten ignoriert.

Verwenden Sie `allowBots`, wenn Sie absichtlich Matrix-Datenverkehr zwischen Agents wünschen:

```json5
{
  channels: {
    matrix: {
      allowBots: "mentions", // true | "mentions"
      groups: {
        "!roomid:example.org": {
          requireMention: true,
        },
      },
    },
  },
}
```

- `allowBots: true` akzeptiert Nachrichten anderer konfigurierter Matrix-Bot-Konten in zulässigen Räumen und DMs.
- `allowBots: "mentions"` akzeptiert diese Nachrichten in Räumen nur dann, wenn sie diesen Bot sichtbar erwähnen. DMs sind weiterhin erlaubt.
- `groups.<room>.allowBots` überschreibt die Einstellung auf Kontoebene für einen Raum.
- OpenClaw ignoriert weiterhin Nachrichten derselben Matrix-Benutzer-ID, um Selbstantwort-Schleifen zu vermeiden.
- Matrix stellt hier kein natives Bot-Flag bereit; OpenClaw behandelt „von einem Bot verfasst“ als „von einem anderen konfigurierten Matrix-Konto auf diesem OpenClaw-Gateway gesendet“.

Verwenden Sie strikte Raum-Zulassungslisten und Erwähnungsanforderungen, wenn Sie Bot-zu-Bot-Datenverkehr in gemeinsam genutzten Räumen aktivieren.

Verschlüsselung aktivieren:

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      encryption: true,
      dm: { policy: "pairing" },
    },
  },
}
```

Verifizierungsstatus prüfen:

```bash
openclaw matrix verify status
```

Ausführlicher Status (vollständige Diagnose):

```bash
openclaw matrix verify status --verbose
```

Den gespeicherten Wiederherstellungsschlüssel in maschinenlesbarer Ausgabe einbeziehen:

```bash
openclaw matrix verify status --include-recovery-key --json
```

Cross-Signing- und Verifizierungsstatus bootstrapen:

```bash
openclaw matrix verify bootstrap
```

Unterstützung für mehrere Konten: Verwenden Sie `channels.matrix.accounts` mit kontoabhängigen Anmeldedaten und optionalem `name`. Das gemeinsame Muster finden Sie unter [Konfigurationsreferenz](/de/gateway/configuration-reference#multi-account-all-channels).

Ausführliche Bootstrap-Diagnose:

```bash
openclaw matrix verify bootstrap --verbose
```

Ein Zurücksetzen auf eine frische Cross-Signing-Identität vor dem Bootstrap erzwingen:

```bash
openclaw matrix verify bootstrap --force-reset-cross-signing
```

Dieses Gerät mit einem Wiederherstellungsschlüssel verifizieren:

```bash
openclaw matrix verify device "<your-recovery-key>"
```

Ausführliche Details zur Geräteverifizierung:

```bash
openclaw matrix verify device "<your-recovery-key>" --verbose
```

Integrität der Raumschlüssel-Sicherung prüfen:

```bash
openclaw matrix verify backup status
```

Ausführliche Diagnose zur Integrität der Sicherung:

```bash
openclaw matrix verify backup status --verbose
```

Raumschlüssel aus der Server-Sicherung wiederherstellen:

```bash
openclaw matrix verify backup restore
```

Ausführliche Diagnose zur Wiederherstellung:

```bash
openclaw matrix verify backup restore --verbose
```

Löschen Sie die aktuelle Server-Sicherung und erstellen Sie eine neue Sicherungsbasislinie. Wenn der gespeicherte
Sicherungsschlüssel nicht sauber geladen werden kann, kann dieses Zurücksetzen auch den geheimen Speicher neu erstellen, sodass
zukünftige Kaltstarts den neuen Sicherungsschlüssel laden können:

```bash
openclaw matrix verify backup reset --yes
```

Alle `verify`-Befehle sind standardmäßig kompakt (einschließlich stiller interner SDK-Protokollierung) und zeigen detaillierte Diagnose nur mit `--verbose`.
Verwenden Sie `--json` für vollständige maschinenlesbare Ausgabe bei der Skripterstellung.

In Setups mit mehreren Konten verwenden Matrix-CLI-Befehle das implizite Matrix-Standardkonto, sofern Sie nicht `--account <id>` übergeben.
Wenn Sie mehrere benannte Konten konfigurieren, setzen Sie zuerst `channels.matrix.defaultAccount`, andernfalls werden diese impliziten CLI-Operationen angehalten und fordern Sie auf, explizit ein Konto auszuwählen.
Verwenden Sie `--account` immer dann, wenn Verifizierungs- oder Geräteoperationen explizit ein benanntes Konto ansprechen sollen:

```bash
openclaw matrix verify status --account assistant
openclaw matrix verify backup restore --account assistant
openclaw matrix devices list --account assistant
```

Wenn die Verschlüsselung für ein benanntes Konto deaktiviert oder nicht verfügbar ist, verweisen Matrix-Warnungen und Verifizierungsfehler auf den Konfigurationsschlüssel dieses Kontos, zum Beispiel `channels.matrix.accounts.assistant.encryption`.

### Was „verifiziert“ bedeutet

OpenClaw behandelt dieses Matrix-Gerät nur dann als verifiziert, wenn es durch Ihre eigene Cross-Signing-Identität verifiziert ist.
In der Praxis macht `openclaw matrix verify status --verbose` drei Vertrauenssignale sichtbar:

- `Locally trusted`: Dieses Gerät wird nur vom aktuellen Client als vertrauenswürdig eingestuft
- `Cross-signing verified`: Das SDK meldet das Gerät als durch Cross-Signing verifiziert
- `Signed by owner`: Das Gerät ist mit Ihrem eigenen Self-Signing-Schlüssel signiert

`Verified by owner` wird nur dann zu `yes`, wenn Cross-Signing-Verifizierung oder Eigentümer-Signierung vorhanden ist.
Lokales Vertrauen allein reicht nicht aus, damit OpenClaw das Gerät als vollständig verifiziert behandelt.

### Was Bootstrap macht

`openclaw matrix verify bootstrap` ist der Reparatur- und Einrichtungsbefehl für verschlüsselte Matrix-Konten.
Er führt der Reihe nach Folgendes aus:

- bootstrapt den geheimen Speicher und verwendet nach Möglichkeit einen vorhandenen Wiederherstellungsschlüssel wieder
- bootstrapt Cross-Signing und lädt fehlende öffentliche Cross-Signing-Schlüssel hoch
- versucht, das aktuelle Gerät zu markieren und per Cross-Signing zu signieren
- erstellt eine neue serverseitige Raumschlüssel-Sicherung, falls noch keine existiert

Wenn der Homeserver interaktive Authentifizierung zum Hochladen von Cross-Signing-Schlüsseln verlangt, versucht OpenClaw das Hochladen zuerst ohne Authentifizierung, dann mit `m.login.dummy`, dann mit `m.login.password`, wenn `channels.matrix.password` konfiguriert ist.

Verwenden Sie `--force-reset-cross-signing` nur, wenn Sie absichtlich die aktuelle Cross-Signing-Identität verwerfen und eine neue erstellen möchten.

Wenn Sie absichtlich die aktuelle Raumschlüssel-Sicherung verwerfen und eine neue
Sicherungsbasislinie für zukünftige Nachrichten starten möchten, verwenden Sie `openclaw matrix verify backup reset --yes`.
Tun Sie dies nur, wenn Sie akzeptieren, dass nicht wiederherstellbarer alter verschlüsselter Verlauf
nicht verfügbar bleibt und OpenClaw den geheimen Speicher möglicherweise neu erstellt, wenn das aktuelle Sicherungs-
geheimnis nicht sicher geladen werden kann.

### Frische Sicherungsbasislinie

Wenn Sie möchten, dass zukünftige verschlüsselte Nachrichten weiterhin funktionieren, und akzeptieren, dass nicht wiederherstellbarer alter Verlauf verloren geht, führen Sie diese Befehle in dieser Reihenfolge aus:

```bash
openclaw matrix verify backup reset --yes
openclaw matrix verify backup status --verbose
openclaw matrix verify status
```

Fügen Sie zu jedem Befehl `--account <id>` hinzu, wenn Sie explizit ein benanntes Matrix-Konto ansprechen möchten.

### Startverhalten

Wenn `encryption: true` gesetzt ist, verwendet Matrix standardmäßig `startupVerification` mit dem Wert `"if-unverified"`.
Beim Start fordert Matrix, falls dieses Gerät noch nicht verifiziert ist, in einem anderen Matrix-Client eine Selbstverifizierung an,
überspringt doppelte Anfragen, solange bereits eine aussteht, und wendet vor erneuten Versuchen nach Neustarts eine lokale Abkühlzeit an.
Fehlgeschlagene Anforderungsversuche werden standardmäßig früher erneut versucht als erfolgreiche Erstellungen von Anforderungen.
Setzen Sie `startupVerification: "off"`, um automatische Anfragen beim Start zu deaktivieren, oder passen Sie `startupVerificationCooldownHours`
an, wenn Sie ein kürzeres oder längeres Wiederholungsfenster wünschen.

Beim Start wird außerdem automatisch ein konservativer Crypto-Bootstrap-Durchlauf ausgeführt.
Dieser Durchlauf versucht zuerst, den aktuellen geheimen Speicher und die aktuelle Cross-Signing-Identität wiederzuverwenden, und vermeidet das Zurücksetzen von Cross-Signing, sofern Sie nicht einen expliziten Bootstrap-Reparaturablauf ausführen.

Wenn beim Start ein fehlerhafter Bootstrap-Zustand gefunden wird und `channels.matrix.password` konfiguriert ist, kann OpenClaw einen strengeren Reparaturpfad versuchen.
Wenn das aktuelle Gerät bereits vom Eigentümer signiert ist, bewahrt OpenClaw diese Identität, anstatt sie automatisch zurückzusetzen.

Upgrade vom vorherigen öffentlichen Matrix-Plugin:

- OpenClaw verwendet nach Möglichkeit automatisch dasselbe Matrix-Konto, dasselbe Zugriffstoken und dieselbe Geräteidentität wieder.
- Bevor umsetzbare Matrix-Migrationsänderungen ausgeführt werden, erstellt oder verwendet OpenClaw einen Wiederherstellungs-Snapshot unter `~/Backups/openclaw-migrations/`.
- Wenn Sie mehrere Matrix-Konten verwenden, setzen Sie `channels.matrix.defaultAccount`, bevor Sie vom alten Flat-Store-Layout aktualisieren, damit OpenClaw weiß, welches Konto diesen gemeinsam genutzten Legacy-Status erhalten soll.
- Wenn das vorherige Plugin lokal einen Entschlüsselungsschlüssel für Matrix-Raumschlüssel-Sicherungen gespeichert hat, importiert der Start oder `openclaw doctor --fix` ihn automatisch in den neuen Wiederherstellungsschlüssel-Ablauf.
- Wenn sich das Matrix-Zugriffstoken geändert hat, nachdem die Migration vorbereitet wurde, durchsucht der Start jetzt benachbarte token-hash-Speicherwurzeln nach ausstehendem Legacy-Wiederherstellungsstatus, bevor die automatische Sicherungswiederherstellung aufgegeben wird.
- Wenn sich das Matrix-Zugriffstoken später für dasselbe Konto, denselben Homeserver und denselben Benutzer ändert, bevorzugt OpenClaw jetzt die Wiederverwendung der vollständigsten vorhandenen token-hash-Speicherwurzel, anstatt mit einem leeren Matrix-Statusverzeichnis zu beginnen.
- Beim nächsten Gateway-Start werden gesicherte Raumschlüssel automatisch in den neuen Crypto-Speicher wiederhergestellt.
- Wenn das alte Plugin nur lokale Raumschlüssel hatte, die nie gesichert wurden, warnt OpenClaw deutlich. Diese Schlüssel können nicht automatisch aus dem vorherigen Rust-Crypto-Speicher exportiert werden, daher kann ein Teil des alten verschlüsselten Verlaufs bis zur manuellen Wiederherstellung nicht verfügbar bleiben.
- Siehe [Matrix-Migration](/de/install/migrating-matrix) für den vollständigen Upgrade-Ablauf, Einschränkungen, Wiederherstellungsbefehle und häufige Migrationsmeldungen.

Der verschlüsselte Laufzeitstatus ist unter konto-, benutzer- und token-hash-spezifischen Wurzeln organisiert in
`~/.openclaw/matrix/accounts/<account>/<homeserver>__<user>/<token-hash>/`.
Dieses Verzeichnis enthält den Sync-Speicher (`bot-storage.json`), den Crypto-Speicher (`crypto/`),
die Wiederherstellungsschlüsseldatei (`recovery-key.json`), den IndexedDB-Snapshot (`crypto-idb-snapshot.json`),
Thread-Bindings (`thread-bindings.json`) und den Status der Startverifizierung (`startup-verification.json`),
wenn diese Funktionen verwendet werden.
Wenn sich das Token ändert, die Kontoidentität aber gleich bleibt, verwendet OpenClaw die beste vorhandene
Wurzel für dieses Konto/Homeserver/Benutzer-Tupel wieder, damit vorheriger Sync-Status, Crypto-Status, Thread-Bindings
und Startverifizierungsstatus sichtbar bleiben.

### Node-Crypto-Speichermodell

Matrix-E2EE in diesem Plugin verwendet den offiziellen Rust-Crypto-Pfad von `matrix-js-sdk` in Node.
Dieser Pfad erwartet IndexedDB-gestützte Persistenz, wenn der Crypto-Status Neustarts überdauern soll.

OpenClaw stellt dies derzeit in Node bereit, indem es:

- `fake-indexeddb` als den vom SDK erwarteten IndexedDB-API-Shim verwendet
- die Inhalte der Rust-Crypto-IndexedDB aus `crypto-idb-snapshot.json` vor `initRustCrypto` wiederherstellt
- die aktualisierten IndexedDB-Inhalte nach der Initialisierung und während der Laufzeit zurück in `crypto-idb-snapshot.json` persistiert
- die Snapshot-Wiederherstellung und Persistierung gegenüber `crypto-idb-snapshot.json` mit einer beratenden Dateisperre serialisiert, damit Laufzeitpersistenz des Gateways und CLI-Wartung nicht um dieselbe Snapshot-Datei konkurrieren

Dies ist Kompatibilitäts-/Speicher-Infrastruktur, keine benutzerdefinierte Crypto-Implementierung.
Die Snapshot-Datei ist sensibler Laufzeitstatus und wird mit restriktiven Dateiberechtigungen gespeichert.
Unter dem Sicherheitsmodell von OpenClaw befinden sich der Gateway-Host und das lokale OpenClaw-Statusverzeichnis bereits innerhalb der vertrauenswürdigen Betreibergrenze, daher ist dies primär ein operatives Haltbarkeitsproblem und keine separate Remote-Vertrauensgrenze.

Geplante Verbesserung:

- SecretRef-Unterstützung für persistentes Matrix-Schlüsselmaterial hinzufügen, damit Wiederherstellungsschlüssel und verwandte Speicher-Verschlüsselungsgeheimnisse aus OpenClaw-Geheimnisanbietern statt nur aus lokalen Dateien bezogen werden können

## Profilverwaltung

Aktualisieren Sie das Matrix-Selbstprofil für das ausgewählte Konto mit:

```bash
openclaw matrix profile set --name "OpenClaw Assistant"
openclaw matrix profile set --avatar-url https://cdn.example.org/avatar.png
```

Fügen Sie `--account <id>` hinzu, wenn Sie explizit ein benanntes Matrix-Konto ansprechen möchten.

Matrix akzeptiert `mxc://`-Avatar-URLs direkt. Wenn Sie eine `http://`- oder `https://`-Avatar-URL übergeben, lädt OpenClaw sie zuerst zu Matrix hoch und speichert die aufgelöste `mxc://`-URL zurück in `channels.matrix.avatarUrl` (oder die ausgewählte Kontoüberschreibung).

## Automatische Verifizierungshinweise

Matrix veröffentlicht jetzt Hinweise zum Verifizierungslebenszyklus direkt als `m.notice`-Nachrichten im strikten DM-Verifizierungsraum.
Dazu gehören:

- Hinweise auf Verifizierungsanfragen
- Hinweise auf Verifizierungsbereitschaft (mit expliziter Anleitung „Per Emoji verifizieren“)
- Hinweise auf Verifizierungsstart und -abschluss
- SAS-Details (Emoji und Dezimalwerte), sofern verfügbar

Eingehende Verifizierungsanfragen von einem anderen Matrix-Client werden von OpenClaw nachverfolgt und automatisch akzeptiert.
Bei Selbstverifizierungsabläufen startet OpenClaw den SAS-Ablauf auch automatisch, sobald Emoji-Verifizierung verfügbar wird, und bestätigt seine eigene Seite.
Bei Verifizierungsanfragen von einem anderen Matrix-Benutzer/Gerät akzeptiert OpenClaw die Anfrage automatisch und wartet dann, bis der SAS-Ablauf normal fortgesetzt wird.
Sie müssen weiterhin die Emoji- oder Dezimal-SAS in Ihrem Matrix-Client vergleichen und dort „Sie stimmen überein“ bestätigen, um die Verifizierung abzuschließen.

OpenClaw akzeptiert selbstinitiierte doppelte Abläufe nicht blind automatisch. Beim Start wird keine neue Anfrage erstellt, wenn bereits eine Selbstverifizierungsanfrage aussteht.

Verifizierungs-/Protokoll-/Systemhinweise werden nicht an die Agent-Chat-Pipeline weitergeleitet und erzeugen daher kein `NO_REPLY`.

### Gerätehygiene

Alte von OpenClaw verwaltete Matrix-Geräte können sich im Konto ansammeln und das Vertrauen in verschlüsselten Räumen schwerer nachvollziehbar machen.
Listen Sie sie auf mit:

```bash
openclaw matrix devices list
```

Entfernen Sie veraltete von OpenClaw verwaltete Geräte mit:

```bash
openclaw matrix devices prune-stale
```

### Reparatur direkter Räume

Wenn der Direktnachrichtenstatus nicht mehr synchron ist, kann OpenClaw veraltete `m.direct`-Zuordnungen haben, die auf alte Einzelräume statt auf die aktive DM zeigen. Überprüfen Sie die aktuelle Zuordnung für einen Peer mit:

```bash
openclaw matrix direct inspect --user-id @alice:example.org
```

Reparieren Sie sie mit:

```bash
openclaw matrix direct repair --user-id @alice:example.org
```

Die Reparatur hält die Matrix-spezifische Logik innerhalb des Plugins:

- sie bevorzugt eine strikte 1:1-DM, die bereits in `m.direct` zugeordnet ist
- andernfalls greift sie auf jede aktuell beigetretene strikte 1:1-DM mit diesem Benutzer zurück
- wenn keine fehlerfreie DM existiert, erstellt sie einen neuen direkten Raum und schreibt `m.direct` so um, dass er auf diesen zeigt

Der Reparaturablauf löscht alte Räume nicht automatisch. Er wählt nur die fehlerfreie DM aus und aktualisiert die Zuordnung, damit neue Matrix-Sendungen, Verifizierungshinweise und andere Direktnachrichtenabläufe wieder den richtigen Raum ansprechen.

## Threads

Matrix unterstützt native Matrix-Threads sowohl für automatische Antworten als auch für Sendungen durch Message-Tools.

- `dm.sessionScope: "per-user"` (Standard) hält das Matrix-DM-Routing absenderbezogen, sodass mehrere DM-Räume eine Sitzung teilen können, wenn sie demselben Peer zugeordnet werden.
- `dm.sessionScope: "per-room"` isoliert jeden Matrix-DM-Raum in seinen eigenen Sitzungsschlüssel und verwendet dabei weiterhin normale DM-Authentifizierungs- und Zulassungslistenprüfungen.
- Explizite Matrix-Konversationsbindungen haben weiterhin Vorrang vor `dm.sessionScope`, sodass gebundene Räume und Threads ihr gewähltes Zielsitzung beibehalten.
- `threadReplies: "off"` hält Antworten auf oberster Ebene und hält eingehende gethreadete Nachrichten in der übergeordneten Sitzung.
- `threadReplies: "inbound"` antwortet in einem Thread nur dann innerhalb dieses Threads, wenn die eingehende Nachricht bereits in diesem Thread war.
- `threadReplies: "always"` hält Raumantworten in einem Thread, der an der auslösenden Nachricht verankert ist, und leitet diese Konversation ab der ersten auslösenden Nachricht durch die passende threadbezogene Sitzung.
- `dm.threadReplies` überschreibt die Einstellung auf oberster Ebene nur für DMs. So können Sie zum Beispiel Raum-Threads isoliert halten und DMs flach lassen.
- Eingehende gethreadete Nachrichten enthalten die Thread-Root-Nachricht als zusätzlichen Agent-Kontext.
- Sendungen durch Message-Tools übernehmen jetzt automatisch den aktuellen Matrix-Thread, wenn das Ziel derselbe Raum oder dasselbe DM-Benutzerziel ist, es sei denn, es wird explizit eine `threadId` angegeben.
- Die Wiederverwendung desselben DM-Benutzerziels in derselben Sitzung greift nur, wenn die aktuellen Sitzungsmetadaten denselben DM-Peer auf demselben Matrix-Konto belegen; andernfalls greift OpenClaw auf normales benutzerbezogenes Routing zurück.
- Wenn OpenClaw erkennt, dass ein Matrix-DM-Raum mit einem anderen DM-Raum auf derselben gemeinsamen Matrix-DM-Sitzung kollidiert, veröffentlicht es in diesem Raum einen einmaligen `m.notice` mit der `/focus`-Ausweichmöglichkeit, wenn Thread-Bindings aktiviert sind, sowie dem Hinweis auf `dm.sessionScope`.
- Laufzeit-Thread-Bindings werden für Matrix unterstützt. `/focus`, `/unfocus`, `/agents`, `/session idle`, `/session max-age` und threadgebundenes `/acp spawn` funktionieren jetzt in Matrix-Räumen und DMs.
- `/focus` auf oberster Ebene in einem Matrix-Raum/DM erstellt einen neuen Matrix-Thread und bindet ihn an die Zielsitzung, wenn `threadBindings.spawnSubagentSessions=true`.
- Das Ausführen von `/focus` oder `/acp spawn --thread here` innerhalb eines vorhandenen Matrix-Threads bindet stattdessen diesen aktuellen Thread.

## ACP-Konversationsbindungen

Matrix-Räume, DMs und vorhandene Matrix-Threads können in dauerhafte ACP-Arbeitsbereiche verwandelt werden, ohne die Chat-Oberfläche zu ändern.

Schneller Operator-Ablauf:

- Führen Sie `/acp spawn codex --bind here` innerhalb der Matrix-DM, des Raums oder des vorhandenen Threads aus, den Sie weiterverwenden möchten.
- In einer Matrix-DM oder einem Raum auf oberster Ebene bleibt die aktuelle DM bzw. der aktuelle Raum die Chat-Oberfläche, und zukünftige Nachrichten werden an die erzeugte ACP-Sitzung geleitet.
- Innerhalb eines vorhandenen Matrix-Threads bindet `--bind here` diesen aktuellen Thread direkt.
- `/new` und `/reset` setzen dieselbe gebundene ACP-Sitzung direkt zurück.
- `/acp close` schließt die ACP-Sitzung und entfernt die Bindung.

Hinweise:

- `--bind here` erstellt keinen untergeordneten Matrix-Thread.
- `threadBindings.spawnAcpSessions` ist nur für `/acp spawn --thread auto|here` erforderlich, wenn OpenClaw einen untergeordneten Matrix-Thread erstellen oder binden muss.

### Thread-Binding-Konfiguration

Matrix übernimmt globale Standardwerte aus `session.threadBindings` und unterstützt außerdem kanalbezogene Überschreibungen:

- `threadBindings.enabled`
- `threadBindings.idleHours`
- `threadBindings.maxAgeHours`
- `threadBindings.spawnSubagentSessions`
- `threadBindings.spawnAcpSessions`

Threadgebundene Spawn-Flags für Matrix sind Opt-in:

- Setzen Sie `threadBindings.spawnSubagentSessions: true`, um zuzulassen, dass `/focus` auf oberster Ebene neue Matrix-Threads erstellt und bindet.
- Setzen Sie `threadBindings.spawnAcpSessions: true`, um zuzulassen, dass `/acp spawn --thread auto|here` ACP-Sitzungen an Matrix-Threads bindet.

## Reaktionen

Matrix unterstützt ausgehende Reaktionsaktionen, eingehende Reaktionsbenachrichtigungen und eingehende Bestätigungsreaktionen.

- Werkzeuge für ausgehende Reaktionen werden durch `channels["matrix"].actions.reactions` gesteuert.
- `react` fügt einem bestimmten Matrix-Ereignis eine Reaktion hinzu.
- `reactions` listet die aktuelle Reaktionszusammenfassung für ein bestimmtes Matrix-Ereignis auf.
- `emoji=""` entfernt die eigenen Reaktionen des Bot-Kontos auf diesem Ereignis.
- `remove: true` entfernt nur die angegebene Emoji-Reaktion vom Bot-Konto.

Der Geltungsbereich von Bestätigungsreaktionen wird in dieser Standardreihenfolge von OpenClaw aufgelöst:

- `channels["matrix"].accounts.<accountId>.ackReaction`
- `channels["matrix"].ackReaction`
- `messages.ackReaction`
- Agent-Identity-Emoji-Fallback

Der Geltungsbereich von Bestätigungsreaktionen wird in dieser Reihenfolge aufgelöst:

- `channels["matrix"].accounts.<accountId>.ackReactionScope`
- `channels["matrix"].ackReactionScope`
- `messages.ackReactionScope`

Der Modus für Reaktionsbenachrichtigungen wird in dieser Reihenfolge aufgelöst:

- `channels["matrix"].accounts.<accountId>.reactionNotifications`
- `channels["matrix"].reactionNotifications`
- Standard: `own`

Aktuelles Verhalten:

- `reactionNotifications: "own"` leitet hinzugefügte `m.reaction`-Ereignisse weiter, wenn sie auf vom Bot verfasste Matrix-Nachrichten abzielen.
- `reactionNotifications: "off"` deaktiviert Reaktionssystemereignisse.
- Das Entfernen von Reaktionen wird weiterhin nicht in Systemereignisse umgewandelt, da Matrix diese als Redaktionen und nicht als eigenständige `m.reaction`-Entfernungen darstellt.

## Verlaufs-Kontext

- `channels.matrix.historyLimit` steuert, wie viele aktuelle Raumnachrichten als `InboundHistory` einbezogen werden, wenn eine Matrix-Raumnachricht den Agent auslöst.
- Es greift auf `messages.groupChat.historyLimit` zurück. Wenn beide nicht gesetzt sind, ist der effektive Standardwert `0`, sodass erwähnungsgesteuerte Raumnachrichten nicht gepuffert werden. Setzen Sie `0`, um zu deaktivieren.
- Der Matrix-Raumverlauf ist nur raumbezogen. DMs verwenden weiterhin den normalen Sitzungsverlauf.
- Der Matrix-Raumverlauf ist nur für ausstehende Nachrichten: OpenClaw puffert Raumnachrichten, die noch keine Antwort ausgelöst haben, und erstellt dann einen Snapshot dieses Fensters, wenn eine Erwähnung oder ein anderer Auslöser eintrifft.
- Die aktuelle Auslösernachricht wird nicht in `InboundHistory` einbezogen; sie verbleibt für diesen Zug im Haupttext der eingehenden Nachricht.
- Wiederholungen desselben Matrix-Ereignisses verwenden den ursprünglichen Verlaufs-Snapshot erneut, anstatt zu neueren Raumnachrichten weiterzuwandern.

## Kontextsichtigkeit

Matrix unterstützt die gemeinsame Steuerung `contextVisibility` für ergänzenden Raumkontext wie abgerufenen Antworttext, Thread-Wurzeln und ausstehenden Verlauf.

- `contextVisibility: "all"` ist die Standardeinstellung. Ergänzender Kontext wird wie empfangen beibehalten.
- `contextVisibility: "allowlist"` filtert ergänzenden Kontext nach Absendern, die durch die aktiven Raum-/Benutzer-Zulassungslistenprüfungen erlaubt sind.
- `contextVisibility: "allowlist_quote"` verhält sich wie `allowlist`, behält aber weiterhin eine explizit zitierte Antwort bei.

Diese Einstellung betrifft die Sichtbarkeit ergänzenden Kontexts, nicht die Frage, ob die eingehende Nachricht selbst eine Antwort auslösen kann.
Die Berechtigung zum Auslösen stammt weiterhin aus `groupPolicy`, `groups`, `groupAllowFrom` und den DM-Richtlinieneinstellungen.

## Beispiel für DM- und Raumrichtlinien

```json5
{
  channels: {
    matrix: {
      dm: {
        policy: "allowlist",
        allowFrom: ["@admin:example.org"],
        threadReplies: "off",
      },
      groupPolicy: "allowlist",
      groupAllowFrom: ["@admin:example.org"],
      groups: {
        "!roomid:example.org": {
          requireMention: true,
        },
      },
    },
  },
}
```

Siehe [Groups](/de/channels/groups) für Verhalten bei Erwähnungsgating und Zulassungslisten.

Beispiel für Pairing in Matrix-DMs:

```bash
openclaw pairing list matrix
openclaw pairing approve matrix <CODE>
```

Wenn ein nicht genehmigter Matrix-Benutzer Sie vor der Genehmigung weiterhin anschreibt, verwendet OpenClaw denselben ausstehenden Pairing-Code wieder und kann nach einer kurzen Abkühlzeit erneut eine Erinnerungsantwort senden, anstatt einen neuen Code zu erzeugen.

Siehe [Pairing](/de/channels/pairing) für den gemeinsamen DM-Pairing-Ablauf und das Speicherlayout.

## Exec-Genehmigungen

Matrix kann als nativer Genehmigungsclient für ein Matrix-Konto dienen. Die nativen
DM-/Kanal-Routing-Regler befinden sich weiterhin unter der Exec-Genehmigungskonfiguration:

- `channels.matrix.execApprovals.enabled`
- `channels.matrix.execApprovals.approvers` (optional; greift auf `channels.matrix.dm.allowFrom` zurück)
- `channels.matrix.execApprovals.target` (`dm` | `channel` | `both`, Standard: `dm`)
- `channels.matrix.execApprovals.agentFilter`
- `channels.matrix.execApprovals.sessionFilter`

Genehmiger müssen Matrix-Benutzer-IDs wie `@owner:example.org` sein. Matrix aktiviert native Genehmigungen automatisch, wenn `enabled` nicht gesetzt oder `"auto"` ist und mindestens ein Genehmiger aufgelöst werden kann. Exec-Genehmigungen verwenden zuerst `execApprovals.approvers` und können auf `channels.matrix.dm.allowFrom` zurückgreifen. Plugin-Genehmigungen autorisieren über `channels.matrix.dm.allowFrom`. Setzen Sie `enabled: false`, um Matrix explizit als nativen Genehmigungsclient zu deaktivieren. Genehmigungsanfragen greifen andernfalls auf andere konfigurierte Genehmigungsrouten oder die Fallback-Richtlinie für Genehmigungen zurück.

Das native Matrix-Routing unterstützt jetzt beide Genehmigungsarten:

- `channels.matrix.execApprovals.*` steuert den nativen DM-/Kanal-Fanout-Modus für Matrix-Genehmigungsaufforderungen.
- Exec-Genehmigungen verwenden die Genehmigermenge für Exec aus `execApprovals.approvers` oder `channels.matrix.dm.allowFrom`.
- Plugin-Genehmigungen verwenden die Matrix-DM-Zulassungsliste aus `channels.matrix.dm.allowFrom`.
- Matrix-Reaktionskürzel und Nachrichtenaktualisierungen gelten für Exec- und Plugin-Genehmigungen.

Zustellregeln:

- `target: "dm"` sendet Genehmigungsaufforderungen an Genehmiger-DMs
- `target: "channel"` sendet die Aufforderung zurück an den ursprünglichen Matrix-Raum oder die ursprüngliche DM
- `target: "both"` sendet an Genehmiger-DMs und an den ursprünglichen Matrix-Raum oder die ursprüngliche DM

Matrix-Genehmigungsaufforderungen initialisieren Reaktionskürzel auf der primären Genehmigungsnachricht:

- `✅` = einmal erlauben
- `❌` = ablehnen
- `♾️` = immer erlauben, wenn diese Entscheidung durch die effektive Exec-Richtlinie zulässig ist

Genehmiger können auf diese Nachricht reagieren oder die Fallback-Slash-Befehle verwenden: `/approve <id> allow-once`, `/approve <id> allow-always` oder `/approve <id> deny`.

Nur aufgelöste Genehmiger können genehmigen oder ablehnen. Bei Exec-Genehmigungen enthält die Kanalzustellung den Befehlstext, daher sollten `channel` oder `both` nur in vertrauenswürdigen Räumen aktiviert werden.

Matrix-Genehmigungsaufforderungen verwenden den gemeinsamen Kern-Genehmigungsplaner erneut. Die Matrix-spezifische native Oberfläche verarbeitet Raum-/DM-Routing, Reaktionen sowie das Senden/Aktualisieren/Löschen von Nachrichten für Exec- und Plugin-Genehmigungen.

Kontoabhängige Überschreibung:

- `channels.matrix.accounts.<account>.execApprovals`

Verwandte Dokumentation: [Exec-Genehmigungen](/de/tools/exec-approvals)

## Beispiel für mehrere Konten

```json5
{
  channels: {
    matrix: {
      enabled: true,
      defaultAccount: "assistant",
      dm: { policy: "pairing" },
      accounts: {
        assistant: {
          homeserver: "https://matrix.example.org",
          accessToken: "syt_assistant_xxx",
          encryption: true,
        },
        alerts: {
          homeserver: "https://matrix.example.org",
          accessToken: "syt_alerts_xxx",
          dm: {
            policy: "allowlist",
            allowFrom: ["@ops:example.org"],
            threadReplies: "off",
          },
        },
      },
    },
  },
}
```

Werte auf oberster Ebene in `channels.matrix` fungieren als Standardwerte für benannte Konten, sofern ein Konto sie nicht überschreibt.
Sie können geerbte Raumeinträge mit `groups.<room>.account` (oder dem Legacy-Schlüssel `rooms.<room>.account`) auf ein Matrix-Konto beschränken.
Einträge ohne `account` bleiben für alle Matrix-Konten gemeinsam, und Einträge mit `account: "default"` funktionieren weiterhin, wenn das Standardkonto direkt auf oberster Ebene unter `channels.matrix.*` konfiguriert ist.
Partielle gemeinsam genutzte Auth-Standardwerte erzeugen für sich genommen kein separates implizites Standardkonto. OpenClaw synthetisiert das oberste `default`-Konto nur dann, wenn dieses Standardkonto frische Authentifizierung hat (`homeserver` plus `accessToken` oder `homeserver` plus `userId` und `password`); benannte Konten können weiterhin über `homeserver` plus `userId` erkennbar bleiben, wenn zwischengespeicherte Anmeldedaten die Authentifizierung später erfüllen.
Wenn Matrix bereits genau ein benanntes Konto hat oder `defaultAccount` auf einen vorhandenen benannten Kontoschlüssel zeigt, bewahrt die Reparatur/Einrichtungs-Promotion von Einzelkonto zu Mehrkonto dieses Konto, anstatt einen neuen `accounts.default`-Eintrag zu erstellen. Nur Matrix-Auth-/Bootstrap-Schlüssel werden in dieses hochgestufte Konto verschoben; gemeinsam genutzte Richtlinienschlüssel für die Zustellung bleiben auf oberster Ebene.
Setzen Sie `defaultAccount`, wenn OpenClaw ein benanntes Matrix-Konto für implizites Routing, Sondierungen und CLI-Operationen bevorzugen soll.
Wenn Sie mehrere benannte Konten konfigurieren, setzen Sie `defaultAccount` oder übergeben Sie `--account <id>` für CLI-Befehle, die auf impliziter Kontoauswahl beruhen.
Übergeben Sie `--account <id>` an `openclaw matrix verify ...` und `openclaw matrix devices ...`, wenn Sie diese implizite Auswahl für einen Befehl überschreiben möchten.

## Private/LAN-Homeserver

Standardmäßig blockiert OpenClaw private/interne Matrix-Homeserver zum Schutz vor SSRF, sofern Sie
nicht pro Konto explizit optieren.

Wenn Ihr Homeserver auf localhost, einer LAN-/Tailscale-IP oder einem internen Hostnamen läuft, aktivieren Sie
`network.dangerouslyAllowPrivateNetwork` für dieses Matrix-Konto:

```json5
{
  channels: {
    matrix: {
      homeserver: "http://matrix-synapse:8008",
      network: {
        dangerouslyAllowPrivateNetwork: true,
      },
      accessToken: "syt_internal_xxx",
    },
  },
}
```

Beispiel für die CLI-Einrichtung:

```bash
openclaw matrix account add \
  --account ops \
  --homeserver http://matrix-synapse:8008 \
  --allow-private-network \
  --access-token syt_ops_xxx
```

Dieses Opt-in erlaubt nur vertrauenswürdige private/interne Ziele. Öffentliche unverschlüsselte Homeserver wie
`http://matrix.example.org:8008` bleiben blockiert. Bevorzugen Sie wann immer möglich `https://`.

## Proxying von Matrix-Datenverkehr

Wenn Ihre Matrix-Bereitstellung einen expliziten ausgehenden HTTP(S)-Proxy benötigt, setzen Sie `channels.matrix.proxy`:

```json5
{
  channels: {
    matrix: {
      homeserver: "https://matrix.example.org",
      accessToken: "syt_bot_xxx",
      proxy: "http://127.0.0.1:7890",
    },
  },
}
```

Benannte Konten können den Standardwert auf oberster Ebene mit `channels.matrix.accounts.<id>.proxy` überschreiben.
OpenClaw verwendet dieselbe Proxy-Einstellung für den Matrix-Laufzeitverkehr und für Sondierungen des Kontostatus.

## Zielauflösung

Matrix akzeptiert diese Zielformen überall dort, wo OpenClaw nach einem Raum- oder Benutzerziel fragt:

- Benutzer: `@user:server`, `user:@user:server` oder `matrix:user:@user:server`
- Räume: `!room:server`, `room:!room:server` oder `matrix:room:!room:server`
- Aliase: `#alias:server`, `channel:#alias:server` oder `matrix:channel:#alias:server`

Die Live-Verzeichnissuche verwendet das angemeldete Matrix-Konto:

- Benutzersuchen fragen das Matrix-Benutzerverzeichnis auf diesem Homeserver ab.
- Raumsuchen akzeptieren explizite Raum-IDs und Aliase direkt und greifen dann auf die Suche nach Namen beigetretener Räume für dieses Konto zurück.
- Die Suche nach Namen beigetretener Räume ist Best-Effort. Wenn ein Raumname nicht zu einer ID oder einem Alias aufgelöst werden kann, wird er bei der Laufzeitauflösung der Zulassungsliste ignoriert.

## Konfigurationsreferenz

- `enabled`: den Kanal aktivieren oder deaktivieren.
- `name`: optionales Label für das Konto.
- `defaultAccount`: bevorzugte Konto-ID, wenn mehrere Matrix-Konten konfiguriert sind.
- `homeserver`: Homeserver-URL, zum Beispiel `https://matrix.example.org`.
- `network.dangerouslyAllowPrivateNetwork`: diesem Matrix-Konto erlauben, sich mit privaten/internen Homeservern zu verbinden. Aktivieren Sie dies, wenn der Homeserver auf `localhost`, eine LAN-/Tailscale-IP oder einen internen Host wie `matrix-synapse` aufgelöst wird.
- `proxy`: optionale HTTP(S)-Proxy-URL für Matrix-Datenverkehr. Benannte Konten können den Standardwert auf oberster Ebene mit ihrem eigenen `proxy` überschreiben.
- `userId`: vollständige Matrix-Benutzer-ID, zum Beispiel `@bot:example.org`.
- `accessToken`: Zugriffstoken für tokenbasierte Authentifizierung. Klartextwerte und SecretRef-Werte werden für `channels.matrix.accessToken` und `channels.matrix.accounts.<id>.accessToken` über env/file/exec-Anbieter unterstützt. Siehe [Secrets Management](/de/gateway/secrets).
- `password`: Passwort für passwortbasierte Anmeldung. Klartextwerte und SecretRef-Werte werden unterstützt.
- `deviceId`: explizite Matrix-Geräte-ID.
- `deviceName`: Anzeigename des Geräts für Passwortanmeldung.
- `avatarUrl`: gespeicherte Selbst-Avatar-URL für Profilsynchronisierung und `set-profile`-Aktualisierungen.
- `initialSyncLimit`: Ereignislimit für den Start-Sync.
- `encryption`: E2EE aktivieren.
- `allowlistOnly`: Verhalten nur mit Zulassungsliste für DMs und Räume erzwingen.
- `allowBots`: Nachrichten anderer konfigurierter OpenClaw-Matrix-Konten zulassen (`true` oder `"mentions"`).
- `groupPolicy`: `open`, `allowlist` oder `disabled`.
- `contextVisibility`: Sichtbarkeitsmodus für ergänzenden Raumkontext (`all`, `allowlist`, `allowlist_quote`).
- `groupAllowFrom`: Zulassungsliste von Benutzer-IDs für Raumverkehr.
- `groupAllowFrom`-Einträge sollten vollständige Matrix-Benutzer-IDs sein. Nicht aufgelöste Namen werden zur Laufzeit ignoriert.
- `historyLimit`: maximale Anzahl von Raumnachrichten, die als Gruppenverlaufs-Kontext einbezogen werden. Greift auf `messages.groupChat.historyLimit` zurück; wenn beide nicht gesetzt sind, ist der effektive Standardwert `0`. Setzen Sie `0`, um zu deaktivieren.
- `replyToMode`: `off`, `first`, `all` oder `batched`.
- `markdown`: optionale Markdown-Rendering-Konfiguration für ausgehenden Matrix-Text.
- `streaming`: `off` (Standard), `partial`, `quiet`, `true` oder `false`. `partial` und `true` aktivieren Vorschau-zuerst-Entwurfsaktualisierungen mit normalen Matrix-Textnachrichten. `quiet` verwendet nicht benachrichtigende Vorschauhinweise für selbstgehostete Push-Regel-Setups.
- `blockStreaming`: `true` aktiviert separate Fortschrittsnachrichten für abgeschlossene Assistant-Blöcke, während Entwurfsvorschau-Streaming aktiv ist.
- `threadReplies`: `off`, `inbound` oder `always`.
- `threadBindings`: kanalbezogene Überschreibungen für threadgebundenes Sitzungsrouting und Lebenszyklus.
- `startupVerification`: automatischer Selbstverifizierungs-Anfragemodus beim Start (`if-unverified`, `off`).
- `startupVerificationCooldownHours`: Abkühlzeit vor einem erneuten Versuch automatischer Startverifizierungsanfragen.
- `textChunkLimit`: Chunk-Größe für ausgehende Nachrichten.
- `chunkMode`: `length` oder `newline`.
- `responsePrefix`: optionales Nachrichtenpräfix für ausgehende Antworten.
- `ackReaction`: optionale Überschreibung der Bestätigungsreaktion für diesen Kanal/dieses Konto.
- `ackReactionScope`: optionale Überschreibung des Geltungsbereichs der Bestätigungsreaktion (`group-mentions`, `group-all`, `direct`, `all`, `none`, `off`).
- `reactionNotifications`: Modus für eingehende Reaktionsbenachrichtigungen (`own`, `off`).
- `mediaMaxMb`: Mediengrößenlimit in MB für Matrix-Medienverarbeitung. Es gilt für ausgehende Sendungen und eingehende Medienverarbeitung.
- `autoJoin`: Richtlinie für automatischen Beitritt bei Einladungen (`always`, `allowlist`, `off`). Standard: `off`. Dies gilt allgemein für Matrix-Einladungen, einschließlich Einladungen im DM-Stil, nicht nur für Raum-/Gruppeneinladungen. OpenClaw trifft diese Entscheidung zum Zeitpunkt der Einladung, bevor zuverlässig klassifiziert werden kann, ob der beigetretene Raum eine DM oder eine Gruppe ist.
- `autoJoinAllowlist`: Räume/Aliase, die erlaubt sind, wenn `autoJoin` den Wert `allowlist` hat. Alias-Einträge werden während der Behandlung der Einladung in Raum-IDs aufgelöst; OpenClaw vertraut nicht auf den vom eingeladenen Raum behaupteten Alias-Status.
- `dm`: DM-Richtlinienblock (`enabled`, `policy`, `allowFrom`, `sessionScope`, `threadReplies`).
- `dm.policy`: steuert den DM-Zugriff, nachdem OpenClaw dem Raum beigetreten ist und ihn als DM klassifiziert hat. Es ändert nicht, ob einer Einladung automatisch beigetreten wird.
- `dm.allowFrom`-Einträge sollten vollständige Matrix-Benutzer-IDs sein, es sei denn, Sie haben sie bereits über eine Live-Verzeichnissuche aufgelöst.
- `dm.sessionScope`: `per-user` (Standard) oder `per-room`. Verwenden Sie `per-room`, wenn jeder Matrix-DM-Raum einen separaten Kontext behalten soll, selbst wenn der Peer derselbe ist.
- `dm.threadReplies`: DM-spezifische Überschreibung der Thread-Richtlinie (`off`, `inbound`, `always`). Sie überschreibt die Einstellung `threadReplies` auf oberster Ebene sowohl für die Platzierung von Antworten als auch für die Sitzungsisolierung in DMs.
- `execApprovals`: Matrix-native Zustellung von Exec-Genehmigungen (`enabled`, `approvers`, `target`, `agentFilter`, `sessionFilter`).
- `execApprovals.approvers`: Matrix-Benutzer-IDs, die Exec-Anfragen genehmigen dürfen. Optional, wenn `dm.allowFrom` die Genehmiger bereits identifiziert.
- `execApprovals.target`: `dm | channel | both` (Standard: `dm`).
- `accounts`: benannte kontoabhängige Überschreibungen. Werte auf oberster Ebene in `channels.matrix` fungieren als Standardwerte für diese Einträge.
- `groups`: richtlinienbezogene Zuordnung pro Raum. Bevorzugen Sie Raum-IDs oder Aliase; nicht aufgelöste Raumnamen werden zur Laufzeit ignoriert. Die Sitzungs-/Gruppenidentität verwendet nach der Auflösung die stabile Raum-ID, während menschenlesbare Labels weiterhin von Raumnamen stammen.
- `groups.<room>.account`: einen geerbten Raumeintrag in Setups mit mehreren Konten auf ein bestimmtes Matrix-Konto beschränken.
- `groups.<room>.allowBots`: Überschreibung auf Raumebene für Absender aus konfigurierten Bot-Konten (`true` oder `"mentions"`).
- `groups.<room>.users`: absenderbezogene Zulassungsliste pro Raum.
- `groups.<room>.tools`: raumbezogene Tool-Allow/Deny-Überschreibungen.
- `groups.<room>.autoReply`: Überschreibung des Erwähnungsgatings auf Raumebene. `true` deaktiviert Erwähnungsanforderungen für diesen Raum; `false` erzwingt sie wieder.
- `groups.<room>.skills`: optionaler Skill-Filter auf Raumebene.
- `groups.<room>.systemPrompt`: optionales systemPrompt-Snippet auf Raumebene.
- `rooms`: Legacy-Alias für `groups`.
- `actions`: Tool-Gating pro Aktion (`messages`, `reactions`, `pins`, `profile`, `memberInfo`, `channelInfo`, `verification`).

## Verwandt

- [Channels Overview](/de/channels) — alle unterstützten Kanäle
- [Pairing](/de/channels/pairing) — DM-Authentifizierung und Pairing-Ablauf
- [Groups](/de/channels/groups) — Verhalten in Gruppenchats und Erwähnungsgating
- [Channel Routing](/de/channels/channel-routing) — Sitzungsrouting für Nachrichten
- [Security](/de/gateway/security) — Zugriffsmodell und Härtung
