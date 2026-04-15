---
read_when:
    - Matrix in OpenClaw einrichten
    - Matrix-E2EE und Verifizierung konfigurieren
summary: Status des Matrix-Supports, Einrichtung und Konfigurationsbeispiele
title: Matrix
x-i18n:
    generated_at: "2026-04-15T06:21:25Z"
    model: gpt-5.4
    provider: openai
    source_hash: 631f6fdcfebc23136c1a66b04851a25c047535d13cceba5650b8b421bc3afcf8
    source_path: channels/matrix.md
    workflow: 15
---

# Matrix

Matrix ist ein gebündeltes Kanal-Plugin für OpenClaw.
Es verwendet das offizielle `matrix-js-sdk` und unterstützt Direktnachrichten, Räume, Threads, Medien, Reaktionen, Umfragen, Standort und E2EE.

## Gebündeltes Plugin

Matrix wird in aktuellen OpenClaw-Releases als gebündeltes Plugin mitgeliefert, daher
benötigen normale paketierte Builds keine separate Installation.

Wenn du einen älteren Build oder eine benutzerdefinierte Installation verwendest, die Matrix ausschließt, installiere
es manuell:

Von npm installieren:

```bash
openclaw plugins install @openclaw/matrix
```

Aus einem lokalen Checkout installieren:

```bash
openclaw plugins install ./path/to/local/matrix-plugin
```

Siehe [Plugins](/de/tools/plugin) für Plugin-Verhalten und Installationsregeln.

## Einrichtung

1. Stelle sicher, dass das Matrix-Plugin verfügbar ist.
   - Aktuelle paketierte OpenClaw-Releases enthalten es bereits.
   - Ältere/benutzerdefinierte Installationen können es mit den obigen Befehlen manuell hinzufügen.
2. Erstelle ein Matrix-Konto auf deinem Homeserver.
3. Konfiguriere `channels.matrix` mit entweder:
   - `homeserver` + `accessToken`, oder
   - `homeserver` + `userId` + `password`.
4. Starte das Gateway neu.
5. Starte eine Direktnachricht mit dem Bot oder lade ihn in einen Raum ein.
   - Neue Matrix-Einladungen funktionieren nur, wenn `channels.matrix.autoJoin` sie zulässt.

Interaktive Einrichtungswege:

```bash
openclaw channels add
openclaw configure --section channels
```

Der Matrix-Assistent fragt nach:

- Homeserver-URL
- Authentifizierungsmethode: Access Token oder Passwort
- Benutzer-ID (nur Passwort-Authentifizierung)
- optionalem Gerätenamen
- ob E2EE aktiviert werden soll
- ob Raumzugriff und automatisches Beitreten bei Einladungen konfiguriert werden sollen

Wichtige Verhaltensweisen des Assistenten:

- Wenn Matrix-Authentifizierungs-Umgebungsvariablen bereits vorhanden sind und für dieses Konto noch keine Authentifizierung in der Konfiguration gespeichert ist, bietet der Assistent eine Umgebungsvariablen-Verknüpfung an, damit die Authentifizierung in Umgebungsvariablen bleiben kann.
- Kontonamen werden zur Konto-ID normalisiert. Zum Beispiel wird aus `Ops Bot` `ops-bot`.
- Einträge in der DM-Allowlist akzeptieren `@user:server` direkt; Anzeigenamen funktionieren nur, wenn die Live-Verzeichnissuche genau eine Übereinstimmung findet.
- Einträge in der Raum-Allowlist akzeptieren Raum-IDs und Aliasse direkt. Bevorzuge `!room:server` oder `#alias:server`; nicht aufgelöste Namen werden zur Laufzeit von der Allowlist-Auflösung ignoriert.
- Im Allowlist-Modus für automatisches Beitreten bei Einladungen nur stabile Einladungsziele verwenden: `!roomId:server`, `#alias:server` oder `*`. Einfache Raumnamen werden abgelehnt.
- Um Raumnamen vor dem Speichern aufzulösen, verwende `openclaw channels resolve --channel matrix "Project Room"`.

<Warning>
`channels.matrix.autoJoin` ist standardmäßig auf `off` gesetzt.

Wenn du es nicht setzt, tritt der Bot eingeladenen Räumen oder neuen DM-artigen Einladungen nicht bei, daher erscheint er nicht in neuen Gruppen oder eingeladenen DMs, außer du trittst zuerst manuell bei.

Setze `autoJoin: "allowlist"` zusammen mit `autoJoinAllowlist`, um einzuschränken, welche Einladungen akzeptiert werden, oder setze `autoJoin: "always"`, wenn er jeder Einladung beitreten soll.

Im Modus `allowlist` akzeptiert `autoJoinAllowlist` nur `!roomId:server`, `#alias:server` oder `*`.
</Warning>

Allowlist-Beispiel:

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
Wenn dort zwischengespeicherte Anmeldedaten vorhanden sind, behandelt OpenClaw Matrix für Einrichtung, Doctor und Kanalstatus-Erkennung als konfiguriert, auch wenn die aktuelle Authentifizierung nicht direkt in der Konfiguration gesetzt ist.

Entsprechende Umgebungsvariablen (werden verwendet, wenn der Konfigurationsschlüssel nicht gesetzt ist):

- `MATRIX_HOMESERVER`
- `MATRIX_ACCESS_TOKEN`
- `MATRIX_USER_ID`
- `MATRIX_PASSWORD`
- `MATRIX_DEVICE_ID`
- `MATRIX_DEVICE_NAME`

Für Nicht-Standardkonten verwende kontobezogene Umgebungsvariablen:

- `MATRIX_<ACCOUNT_ID>_HOMESERVER`
- `MATRIX_<ACCOUNT_ID>_ACCESS_TOKEN`
- `MATRIX_<ACCOUNT_ID>_USER_ID`
- `MATRIX_<ACCOUNT_ID>_PASSWORD`
- `MATRIX_<ACCOUNT_ID>_DEVICE_ID`
- `MATRIX_<ACCOUNT_ID>_DEVICE_NAME`

Beispiel für das Konto `ops`:

- `MATRIX_OPS_HOMESERVER`
- `MATRIX_OPS_ACCESS_TOKEN`

Für die normalisierte Konto-ID `ops-bot` verwende:

- `MATRIX_OPS_X2D_BOT_HOMESERVER`
- `MATRIX_OPS_X2D_BOT_ACCESS_TOKEN`

Matrix maskiert Satzzeichen in Konto-IDs, damit kontobezogene Umgebungsvariablen kollisionsfrei bleiben.
Zum Beispiel wird `-` zu `_X2D_`, sodass `ops-prod` auf `MATRIX_OPS_X2D_PROD_*` abgebildet wird.

Der interaktive Assistent bietet die Umgebungsvariablen-Verknüpfung nur an, wenn diese Authentifizierungs-Umgebungsvariablen bereits vorhanden sind und für das ausgewählte Konto noch keine Matrix-Authentifizierung in der Konfiguration gespeichert ist.

## Konfigurationsbeispiel

Dies ist eine praktische Basiskonfiguration mit DM-Pairing, Raum-Allowlist und aktiviertem E2EE:

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

`autoJoin` gilt für alle Matrix-Einladungen, einschließlich DM-artiger Einladungen. OpenClaw kann einen eingeladenen
Raum zum Zeitpunkt der Einladung nicht zuverlässig als DM oder Gruppe klassifizieren, daher laufen alle Einladungen zuerst über `autoJoin`.
`dm.policy` gilt, nachdem der Bot beigetreten ist und der Raum als DM klassifiziert wurde.

## Streaming-Vorschauen

Reply-Streaming für Matrix ist optional.

Setze `channels.matrix.streaming` auf `"partial"`, wenn OpenClaw eine einzelne Live-Vorschau
der Antwort senden, diese Vorschau während der Textgenerierung durch das Modell direkt bearbeiten und sie dann
abschließen soll, sobald die Antwort fertig ist:

```json5
{
  channels: {
    matrix: {
      streaming: "partial",
    },
  },
}
```

- `streaming: "off"` ist die Standardeinstellung. OpenClaw wartet auf die endgültige Antwort und sendet sie einmal.
- `streaming: "partial"` erstellt eine bearbeitbare Vorschau-Nachricht für den aktuellen Assistant-Block unter Verwendung normaler Matrix-Textnachrichten. Dadurch bleibt das ältere Benachrichtigungsverhalten von Matrix mit Vorschau zuerst erhalten, sodass Standard-Clients möglicherweise beim ersten gestreamten Vorschautext statt beim fertigen Block benachrichtigen.
- `streaming: "quiet"` erstellt eine bearbeitbare stille Vorschau-Mitteilung für den aktuellen Assistant-Block. Verwende dies nur, wenn du zusätzlich Empfänger-Push-Regeln für abgeschlossene Vorschau-Bearbeitungen konfigurierst.
- `blockStreaming: true` aktiviert separate Matrix-Fortschrittsnachrichten. Wenn Vorschau-Streaming aktiviert ist, behält Matrix den Live-Entwurf für den aktuellen Block bei und belässt abgeschlossene Blöcke als separate Nachrichten.
- Wenn Vorschau-Streaming aktiviert ist und `blockStreaming` deaktiviert ist, bearbeitet Matrix den Live-Entwurf direkt und schließt dasselbe Ereignis ab, wenn der Block oder Turn beendet ist.
- Wenn die Vorschau nicht mehr in ein einzelnes Matrix-Ereignis passt, beendet OpenClaw das Vorschau-Streaming und greift auf normale endgültige Zustellung zurück.
- Medienantworten senden Anhänge weiterhin normal. Wenn eine veraltete Vorschau nicht mehr sicher wiederverwendet werden kann, entfernt OpenClaw sie vor dem Senden der endgültigen Medienantwort.
- Vorschau-Bearbeitungen verursachen zusätzliche Matrix-API-Aufrufe. Lasse Streaming deaktiviert, wenn du das konservativste Verhalten bei Ratenbegrenzungen möchtest.

`blockStreaming` aktiviert Entwurfsvorschauen nicht von selbst.
Verwende `streaming: "partial"` oder `streaming: "quiet"` für Vorschau-Bearbeitungen; füge dann `blockStreaming: true` nur hinzu, wenn abgeschlossene Assistant-Blöcke zusätzlich als separate Fortschrittsnachrichten sichtbar bleiben sollen.

Wenn du Standard-Matrix-Benachrichtigungen ohne benutzerdefinierte Push-Regeln benötigst, verwende `streaming: "partial"` für Vorschau-zuerst-Verhalten oder lasse `streaming` für reine Endzustellung deaktiviert. Mit `streaming: "off"`:

- `blockStreaming: true` sendet jeden abgeschlossenen Block als normale benachrichtigende Matrix-Nachricht.
- `blockStreaming: false` sendet nur die endgültige abgeschlossene Antwort als normale benachrichtigende Matrix-Nachricht.

### Selbst gehostete Push-Regeln für stille abgeschlossene Vorschauen

Wenn du deine eigene Matrix-Infrastruktur betreibst und stille Vorschauen erst dann benachrichtigen sollen, wenn ein Block oder die
endgültige Antwort abgeschlossen ist, setze `streaming: "quiet"` und füge eine benutzerspezifische Push-Regel für abgeschlossene Vorschau-Bearbeitungen hinzu.

Dies ist normalerweise eine Einrichtung pro Empfängerbenutzer, keine globale Konfigurationsänderung am Homeserver:

Kurzübersicht vor dem Start:

- Empfängerbenutzer = die Person, die die Benachrichtigung erhalten soll
- Bot-Benutzer = das OpenClaw-Matrix-Konto, das die Antwort sendet
- verwende für die folgenden API-Aufrufe das Access Token des Empfängerbenutzers
- gleiche `sender` in der Push-Regel mit der vollständigen MXID des Bot-Benutzers ab

1. Konfiguriere OpenClaw für stille Vorschauen:

```json5
{
  channels: {
    matrix: {
      streaming: "quiet",
    },
  },
}
```

2. Stelle sicher, dass das Empfängerkonto bereits normale Matrix-Push-Benachrichtigungen erhält. Regeln für stille Vorschauen
   funktionieren nur, wenn dieser Benutzer bereits funktionierende Pusher/Geräte hat.

3. Besorge das Access Token des Empfängerbenutzers.
   - Verwende das Token des empfangenden Benutzers, nicht das Token des Bots.
   - Ein vorhandenes Client-Sitzungstoken wiederzuverwenden ist normalerweise am einfachsten.
   - Wenn du ein neues Token erstellen musst, kannst du dich über die standardmäßige Matrix Client-Server-API anmelden:

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

4. Prüfe, ob das Empfängerkonto bereits Pusher hat:

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushers"
```

Wenn dies keine aktiven Pusher/Geräte zurückgibt, behebe zuerst die normalen Matrix-Benachrichtigungen, bevor du die
unten stehende OpenClaw-Regel hinzufügst.

OpenClaw markiert abgeschlossene textbasierte Vorschau-Bearbeitungen mit:

```json
{
  "com.openclaw.finalized_preview": true
}
```

5. Erstelle für jedes Empfängerkonto, das diese Benachrichtigungen erhalten soll, eine Override-Push-Regel:

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

Ersetze diese Werte, bevor du den Befehl ausführst:

- `https://matrix.example.org`: deine Homeserver-Basis-URL
- `$USER_ACCESS_TOKEN`: das Access Token des empfangenden Benutzers
- `openclaw-finalized-preview-botname`: eine für diesen Bot bei diesem empfangenden Benutzer eindeutige Regel-ID
- `@bot:example.org`: die MXID deines OpenClaw-Matrix-Bots, nicht die MXID des empfangenden Benutzers

Wichtig für Setups mit mehreren Bots:

- Push-Regeln werden über `ruleId` identifiziert. Wenn `PUT` erneut mit derselben Regel-ID ausgeführt wird, wird genau diese Regel aktualisiert.
- Wenn ein empfangender Benutzer für mehrere OpenClaw-Matrix-Bot-Konten Benachrichtigungen erhalten soll, erstelle eine Regel pro Bot mit jeweils einer eindeutigen Regel-ID für jede `sender`-Übereinstimmung.
- Ein einfaches Muster ist `openclaw-finalized-preview-<botname>`, zum Beispiel `openclaw-finalized-preview-ops` oder `openclaw-finalized-preview-support`.

Die Regel wird anhand des Ereignis-Absenders ausgewertet:

- authentifiziere dich mit dem Token des empfangenden Benutzers
- gleiche `sender` mit der MXID des OpenClaw-Bots ab

6. Prüfe, ob die Regel vorhanden ist:

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname"
```

7. Teste eine gestreamte Antwort. Im stillen Modus sollte der Raum eine stille Entwurfs-Vorschau anzeigen, und die endgültige
   direkte Bearbeitung sollte benachrichtigen, sobald der Block oder Turn abgeschlossen ist.

Wenn du die Regel später entfernen musst, lösche dieselbe Regel-ID mit dem Token des empfangenden Benutzers:

```bash
curl -sS -X DELETE \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname"
```

Hinweise:

- Erstelle die Regel mit dem Access Token des empfangenden Benutzers, nicht mit dem des Bots.
- Neue benutzerdefinierte `override`-Regeln werden vor den standardmäßigen Unterdrückungsregeln eingefügt, daher ist kein zusätzlicher Ordnungsparameter erforderlich.
- Dies betrifft nur reine Text-Vorschau-Bearbeitungen, die OpenClaw sicher direkt abschließen kann. Medien-Fallbacks und Fallbacks für veraltete Vorschauen verwenden weiterhin die normale Matrix-Zustellung.
- Wenn `GET /_matrix/client/v3/pushers` keine Pusher anzeigt, hat der Benutzer für dieses Konto/Gerät noch keine funktionierende Matrix-Push-Zustellung.

#### Synapse

Für Synapse reicht die obige Einrichtung normalerweise bereits aus:

- Es ist keine spezielle Änderung an `homeserver.yaml` für abgeschlossene OpenClaw-Vorschau-Benachrichtigungen erforderlich.
- Wenn deine Synapse-Bereitstellung bereits normale Matrix-Push-Benachrichtigungen sendet, sind das Benutzertoken und der obige `pushrules`-Aufruf der wichtigste Einrichtungsschritt.
- Wenn du Synapse hinter einem Reverse Proxy oder mit Workern betreibst, stelle sicher, dass `/_matrix/client/.../pushrules/` Synapse korrekt erreicht.
- Wenn du Synapse-Worker verwendest, stelle sicher, dass die Pusher fehlerfrei laufen. Die Push-Zustellung wird vom Hauptprozess oder von `synapse.app.pusher` / konfigurierten Pusher-Workern übernommen.

#### Tuwunel

Für Tuwunel verwende denselben Einrichtungsablauf und denselben `push-rule`-API-Aufruf wie oben gezeigt:

- Für den Marker der abgeschlossenen Vorschau selbst ist keine Tuwunel-spezifische Konfiguration erforderlich.
- Wenn normale Matrix-Benachrichtigungen für diesen Benutzer bereits funktionieren, sind das Benutzertoken und der obige `pushrules`-Aufruf der wichtigste Einrichtungsschritt.
- Wenn Benachrichtigungen zu verschwinden scheinen, während der Benutzer auf einem anderen Gerät aktiv ist, prüfe, ob `suppress_push_when_active` aktiviert ist. Tuwunel hat diese Option in Tuwunel 1.4.2 am 12. September 2025 hinzugefügt, und sie kann Pushes an andere Geräte absichtlich unterdrücken, während ein Gerät aktiv ist.

## Bot-zu-Bot-Räume

Standardmäßig werden Matrix-Nachrichten von anderen konfigurierten OpenClaw-Matrix-Konten ignoriert.

Verwende `allowBots`, wenn du absichtlich Matrix-Verkehr zwischen Agents erlauben möchtest:

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

- `allowBots: true` akzeptiert Nachrichten von anderen konfigurierten Matrix-Bot-Konten in erlaubten Räumen und DMs.
- `allowBots: "mentions"` akzeptiert diese Nachrichten in Räumen nur dann, wenn dieser Bot darin sichtbar erwähnt wird. DMs sind weiterhin erlaubt.
- `groups.<room>.allowBots` überschreibt die Einstellung auf Kontoebene für einen einzelnen Raum.
- OpenClaw ignoriert weiterhin Nachrichten von derselben Matrix-Benutzer-ID, um Selbstantwort-Schleifen zu vermeiden.
- Matrix stellt hier kein natives Bot-Flag bereit; OpenClaw behandelt „von Bot verfasst“ als „von einem anderen konfigurierten Matrix-Konto auf diesem OpenClaw-Gateway gesendet“.

Verwende strikte Raum-Allowlists und Erwähnungsanforderungen, wenn du Bot-zu-Bot-Verkehr in gemeinsam genutzten Räumen aktivierst.

## Verschlüsselung und Verifizierung

In verschlüsselten (E2EE-)Räumen verwenden ausgehende Bildereignisse `thumbnail_file`, sodass Bildvorschauen zusammen mit dem vollständigen Anhang verschlüsselt werden. Unverschlüsselte Räume verwenden weiterhin einfaches `thumbnail_url`. Es ist keine Konfiguration erforderlich — das Plugin erkennt den E2EE-Status automatisch.

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

Den gespeicherten Recovery Key in maschinenlesbarer Ausgabe einschließen:

```bash
openclaw matrix verify status --include-recovery-key --json
```

Cross-Signing- und Verifizierungsstatus initialisieren:

```bash
openclaw matrix verify bootstrap
```

Ausführliche Bootstrap-Diagnose:

```bash
openclaw matrix verify bootstrap --verbose
```

Vor dem Bootstrap eine neue Cross-Signing-Identität erzwingen:

```bash
openclaw matrix verify bootstrap --force-reset-cross-signing
```

Dieses Gerät mit einem Recovery Key verifizieren:

```bash
openclaw matrix verify device "<your-recovery-key>"
```

Ausführliche Details zur Geräteverifizierung:

```bash
openclaw matrix verify device "<your-recovery-key>" --verbose
```

Zustand des Raumschlüssel-Backups prüfen:

```bash
openclaw matrix verify backup status
```

Ausführliche Diagnose zum Zustand des Backups:

```bash
openclaw matrix verify backup status --verbose
```

Raumschlüssel aus einem Server-Backup wiederherstellen:

```bash
openclaw matrix verify backup restore
```

Ausführliche Diagnose zur Wiederherstellung:

```bash
openclaw matrix verify backup restore --verbose
```

Das aktuelle Server-Backup löschen und eine neue Backup-Basis erstellen. Wenn der gespeicherte
Backup-Schlüssel nicht sauber geladen werden kann, kann dieses Zurücksetzen auch den Secret Storage neu erstellen, damit
zukünftige Kaltstarts den neuen Backup-Schlüssel laden können:

```bash
openclaw matrix verify backup reset --yes
```

Alle `verify`-Befehle sind standardmäßig kompakt (einschließlich stiller interner SDK-Protokollierung) und zeigen nur mit `--verbose` detaillierte Diagnosen an.
Verwende `--json` für vollständige maschinenlesbare Ausgabe bei der Skripterstellung.

In Setups mit mehreren Konten verwenden Matrix-CLI-Befehle implizit das standardmäßige Matrix-Konto, sofern du nicht `--account <id>` übergibst.
Wenn du mehrere benannte Konten konfigurierst, setze zuerst `channels.matrix.defaultAccount`, andernfalls werden diese impliziten CLI-Operationen angehalten und verlangen, dass du ein Konto explizit auswählst.
Verwende `--account`, wenn Verifizierungs- oder Geräteoperationen gezielt auf ein benanntes Konto angewendet werden sollen:

```bash
openclaw matrix verify status --account assistant
openclaw matrix verify backup restore --account assistant
openclaw matrix devices list --account assistant
```

Wenn Verschlüsselung für ein benanntes Konto deaktiviert oder nicht verfügbar ist, zeigen Matrix-Warnungen und Verifizierungsfehler auf den Konfigurationsschlüssel dieses Kontos, zum Beispiel `channels.matrix.accounts.assistant.encryption`.

### Was „verifiziert“ bedeutet

OpenClaw behandelt dieses Matrix-Gerät nur dann als verifiziert, wenn es durch deine eigene Cross-Signing-Identität verifiziert wurde.
In der Praxis zeigt `openclaw matrix verify status --verbose` drei Vertrauenssignale an:

- `Locally trusted`: Dieses Gerät wird nur vom aktuellen Client als vertrauenswürdig eingestuft
- `Cross-signing verified`: Das SDK meldet das Gerät als durch Cross-Signing verifiziert
- `Signed by owner`: Das Gerät ist mit deinem eigenen Self-Signing-Schlüssel signiert

`Verified by owner` wird nur dann zu `yes`, wenn Cross-Signing-Verifizierung oder Signierung durch den Eigentümer vorhanden ist.
Lokales Vertrauen allein reicht nicht aus, damit OpenClaw das Gerät als vollständig verifiziert behandelt.

### Was Bootstrap macht

`openclaw matrix verify bootstrap` ist der Reparatur- und Einrichtungsbefehl für verschlüsselte Matrix-Konten.
Er führt der Reihe nach Folgendes aus:

- initialisiert Secret Storage und verwendet nach Möglichkeit einen vorhandenen Recovery Key erneut
- initialisiert Cross-Signing und lädt fehlende öffentliche Cross-Signing-Schlüssel hoch
- versucht, das aktuelle Gerät zu markieren und per Cross-Signing zu signieren
- erstellt ein neues serverseitiges Raumschlüssel-Backup, falls noch keines vorhanden ist

Wenn der Homeserver interaktive Authentifizierung zum Hochladen von Cross-Signing-Schlüsseln verlangt, versucht OpenClaw das Hochladen zuerst ohne Authentifizierung, dann mit `m.login.dummy` und anschließend mit `m.login.password`, wenn `channels.matrix.password` konfiguriert ist.

Verwende `--force-reset-cross-signing` nur, wenn du die aktuelle Cross-Signing-Identität absichtlich verwerfen und eine neue erstellen möchtest.

Wenn du das aktuelle Raumschlüssel-Backup absichtlich verwerfen und eine neue
Backup-Basis für zukünftige Nachrichten erstellen möchtest, verwende `openclaw matrix verify backup reset --yes`.
Tu dies nur, wenn du akzeptierst, dass nicht wiederherstellbarer alter verschlüsselter Verlauf
nicht verfügbar bleibt und dass OpenClaw den Secret Storage möglicherweise neu erstellt, wenn das aktuelle Backup-Geheimnis
nicht sicher geladen werden kann.

### Neue Backup-Basis

Wenn du die Funktion für zukünftige verschlüsselte Nachrichten erhalten und den Verlust nicht wiederherstellbaren alten Verlaufs akzeptieren möchtest, führe diese Befehle in dieser Reihenfolge aus:

```bash
openclaw matrix verify backup reset --yes
openclaw matrix verify backup status --verbose
openclaw matrix verify status
```

Füge zu jedem Befehl `--account <id>` hinzu, wenn du gezielt ein benanntes Matrix-Konto ansprechen möchtest.

### Verhalten beim Start

Wenn `encryption: true` gesetzt ist, verwendet Matrix standardmäßig `startupVerification` mit dem Wert `"if-unverified"`.
Beim Start fordert Matrix, wenn dieses Gerät noch nicht verifiziert ist, die Selbstverifizierung in einem anderen Matrix-Client an,
überspringt doppelte Anfragen, solange bereits eine aussteht, und wendet vor einem erneuten Versuch nach Neustarts eine lokale Abkühlzeit an.
Fehlgeschlagene Anfrageversuche werden standardmäßig früher erneut versucht als erfolgreich erstellte Anfragen.
Setze `startupVerification: "off"`, um automatische Anfragen beim Start zu deaktivieren, oder passe `startupVerificationCooldownHours`
an, wenn du ein kürzeres oder längeres Wiederholungsfenster möchtest.

Beim Start wird außerdem automatisch ein konservativer Crypto-Bootstrap-Durchlauf ausgeführt.
Dieser Durchlauf versucht zuerst, den aktuellen Secret Storage und die aktuelle Cross-Signing-Identität wiederzuverwenden, und vermeidet ein Zurücksetzen von Cross-Signing, es sei denn, du führst explizit einen Bootstrap-Reparaturablauf aus.

Wenn beim Start ein defekter Bootstrap-Zustand gefunden wird und `channels.matrix.password` konfiguriert ist, kann OpenClaw einen strengeren Reparaturpfad versuchen.
Wenn das aktuelle Gerät bereits vom Eigentümer signiert ist, bewahrt OpenClaw diese Identität, anstatt sie automatisch zurückzusetzen.

Siehe [Matrix migration](/de/install/migrating-matrix) für den vollständigen Upgrade-Ablauf, Einschränkungen, Recovery-Befehle und häufige Migrationsmeldungen.

### Verifizierungshinweise

Matrix veröffentlicht Hinweise zum Lebenszyklus der Verifizierung direkt im strikten DM-Verifizierungsraum als `m.notice`-Nachrichten.
Dazu gehören:

- Hinweise zu Verifizierungsanfragen
- Hinweise darauf, dass die Verifizierung bereit ist (mit explizitem Hinweis „Per Emoji verifizieren“)
- Hinweise zum Start und Abschluss der Verifizierung
- SAS-Details (Emoji und Dezimalwerte), wenn verfügbar

Eingehende Verifizierungsanfragen von einem anderen Matrix-Client werden von OpenClaw nachverfolgt und automatisch akzeptiert.
Bei Selbstverifizierungsabläufen startet OpenClaw außerdem automatisch den SAS-Ablauf, sobald die Emoji-Verifizierung verfügbar wird, und bestätigt die eigene Seite.
Bei Verifizierungsanfragen von einem anderen Matrix-Benutzer/Gerät akzeptiert OpenClaw die Anfrage automatisch und wartet dann darauf, dass der SAS-Ablauf normal fortgesetzt wird.
Du musst die Emoji- oder dezimalen SAS-Werte in deinem Matrix-Client weiterhin vergleichen und dort „Sie stimmen überein“ bestätigen, um die Verifizierung abzuschließen.

OpenClaw akzeptiert selbst initiierte doppelte Abläufe nicht blind automatisch. Beim Start wird das Erstellen einer neuen Anfrage übersprungen, wenn bereits eine Selbstverifizierungsanfrage aussteht.

Hinweise des Verifizierungsprotokolls/Systems werden nicht an die Agent-Chat-Pipeline weitergeleitet und erzeugen daher kein `NO_REPLY`.

### Gerätehygiene

Alte von OpenClaw verwaltete Matrix-Geräte können sich im Konto ansammeln und die Vertrauensbewertung in verschlüsselten Räumen schwerer nachvollziehbar machen.
Liste sie auf mit:

```bash
openclaw matrix devices list
```

Entferne veraltete von OpenClaw verwaltete Geräte mit:

```bash
openclaw matrix devices prune-stale
```

### Krypto-Speicher

Matrix-E2EE verwendet den offiziellen Rust-Kryptopfad von `matrix-js-sdk` in Node, mit `fake-indexeddb` als IndexedDB-Shim. Der Kryptozustand wird in einer Snapshot-Datei (`crypto-idb-snapshot.json`) gespeichert und beim Start wiederhergestellt. Die Snapshot-Datei ist sensibler Laufzeitzustand und wird mit restriktiven Dateiberechtigungen gespeichert.

Der verschlüsselte Laufzeitzustand befindet sich unter kontospezifischen, benutzerspezifischen Token-Hash-Wurzeln in
`~/.openclaw/matrix/accounts/<account>/<homeserver>__<user>/<token-hash>/`.
Dieses Verzeichnis enthält den Sync-Speicher (`bot-storage.json`), den Krypto-Speicher (`crypto/`),
die Recovery-Key-Datei (`recovery-key.json`), den IndexedDB-Snapshot (`crypto-idb-snapshot.json`),
Thread-Bindungen (`thread-bindings.json`) und den Startverifizierungszustand (`startup-verification.json`).
Wenn sich das Token ändert, die Kontoidentität aber gleich bleibt, verwendet OpenClaw die beste vorhandene
Wurzel für dieses Tupel aus Konto/Homeserver/Benutzer erneut, sodass der bisherige Sync-Zustand, Krypto-Zustand, Thread-Bindungen
und Startverifizierungszustand weiterhin sichtbar bleiben.

## Profilverwaltung

Aktualisiere das Matrix-Selbstprofil für das ausgewählte Konto mit:

```bash
openclaw matrix profile set --name "OpenClaw Assistant"
openclaw matrix profile set --avatar-url https://cdn.example.org/avatar.png
```

Füge `--account <id>` hinzu, wenn du gezielt ein benanntes Matrix-Konto ansprechen möchtest.

Matrix akzeptiert `mxc://`-Avatar-URLs direkt. Wenn du eine `http://`- oder `https://`-Avatar-URL übergibst, lädt OpenClaw sie zuerst zu Matrix hoch und speichert die aufgelöste `mxc://`-URL zurück in `channels.matrix.avatarUrl` (oder in die ausgewählte Kontoüberschreibung).

## Threads

Matrix unterstützt native Matrix-Threads sowohl für automatische Antworten als auch für Sendevorgänge von Message-Tools.

- `dm.sessionScope: "per-user"` (Standard) hält das Routing von Matrix-DMs absenderbezogen, sodass mehrere DM-Räume eine Sitzung gemeinsam nutzen können, wenn sie zum selben Gegenüber aufgelöst werden.
- `dm.sessionScope: "per-room"` isoliert jeden Matrix-DM-Raum in seinen eigenen Sitzungsschlüssel, verwendet aber weiterhin normale DM-Authentifizierung und Allowlist-Prüfungen.
- Explizite Matrix-Konversationsbindungen haben weiterhin Vorrang vor `dm.sessionScope`, sodass gebundene Räume und Threads ihre gewählte Zielsitzung beibehalten.
- `threadReplies: "off"` hält Antworten auf oberster Ebene und belässt eingehende Thread-Nachrichten in der übergeordneten Sitzung.
- `threadReplies: "inbound"` antwortet innerhalb eines Threads nur dann, wenn die eingehende Nachricht bereits in diesem Thread war.
- `threadReplies: "always"` hält Raumantworten in einem Thread mit Wurzel in der auslösenden Nachricht und leitet diese Konversation ab der ersten auslösenden Nachricht über die passende threadspezifische Sitzung.
- `dm.threadReplies` überschreibt die Einstellung der obersten Ebene nur für DMs. Du kannst zum Beispiel Raum-Threads isoliert halten und DMs flach belassen.
- Eingehende Thread-Nachrichten enthalten die Thread-Wurzel-Nachricht als zusätzlichen Agent-Kontext.
- Sendevorgänge von Message-Tools übernehmen automatisch den aktuellen Matrix-Thread, wenn das Ziel derselbe Raum oder dasselbe DM-Benutzerziel ist, es sei denn, eine explizite `threadId` wird angegeben.
- Dieselbe sitzungsbezogene Wiederverwendung eines DM-Benutzerziels greift nur, wenn die aktuellen Sitzungsmetadaten dasselbe DM-Gegenüber auf demselben Matrix-Konto belegen; andernfalls greift OpenClaw auf normales benutzerbezogenes Routing zurück.
- Wenn OpenClaw erkennt, dass ein Matrix-DM-Raum mit einem anderen DM-Raum in derselben gemeinsamen Matrix-DM-Sitzung kollidiert, veröffentlicht es einmalig ein `m.notice` in diesem Raum mit dem `/focus`-Ausweg, wenn Thread-Bindungen aktiviert sind und dem Hinweis `dm.sessionScope`.
- Laufzeit-Thread-Bindungen werden für Matrix unterstützt. `/focus`, `/unfocus`, `/agents`, `/session idle`, `/session max-age` und threadgebundenes `/acp spawn` funktionieren in Matrix-Räumen und DMs.
- `/focus` auf oberster Ebene in einem Matrix-Raum/DM erstellt einen neuen Matrix-Thread und bindet ihn an die Zielsitzung, wenn `threadBindings.spawnSubagentSessions=true`.
- Wenn `/focus` oder `/acp spawn --thread here` innerhalb eines vorhandenen Matrix-Threads ausgeführt wird, bindet es stattdessen diesen aktuellen Thread.

## ACP-Konversationsbindungen

Matrix-Räume, DMs und vorhandene Matrix-Threads können in dauerhafte ACP-Arbeitsbereiche umgewandelt werden, ohne die Chat-Oberfläche zu ändern.

Schneller Operator-Ablauf:

- Führe `/acp spawn codex --bind here` innerhalb der Matrix-DM, des Raums oder des vorhandenen Threads aus, den du weiterverwenden möchtest.
- In einer Matrix-DM oder einem Raum auf oberster Ebene bleibt die aktuelle DM bzw. der aktuelle Raum die Chat-Oberfläche und zukünftige Nachrichten werden an die erzeugte ACP-Sitzung geleitet.
- Innerhalb eines vorhandenen Matrix-Threads bindet `--bind here` diesen aktuellen Thread an Ort und Stelle.
- `/new` und `/reset` setzen dieselbe gebundene ACP-Sitzung an Ort und Stelle zurück.
- `/acp close` schließt die ACP-Sitzung und entfernt die Bindung.

Hinweise:

- `--bind here` erstellt keinen untergeordneten Matrix-Thread.
- `threadBindings.spawnAcpSessions` ist nur für `/acp spawn --thread auto|here` erforderlich, wenn OpenClaw einen untergeordneten Matrix-Thread erstellen oder binden muss.

### Konfiguration von Thread-Bindungen

Matrix übernimmt globale Standardwerte aus `session.threadBindings` und unterstützt außerdem kanalbezogene Überschreibungen:

- `threadBindings.enabled`
- `threadBindings.idleHours`
- `threadBindings.maxAgeHours`
- `threadBindings.spawnSubagentSessions`
- `threadBindings.spawnAcpSessions`

Threadgebundene Spawn-Flags für Matrix sind optional:

- Setze `threadBindings.spawnSubagentSessions: true`, um zu erlauben, dass `/focus` auf oberster Ebene neue Matrix-Threads erstellt und bindet.
- Setze `threadBindings.spawnAcpSessions: true`, um zu erlauben, dass `/acp spawn --thread auto|here` ACP-Sitzungen an Matrix-Threads bindet.

## Reaktionen

Matrix unterstützt ausgehende Reaktionsaktionen, eingehende Reaktionsbenachrichtigungen und eingehende Bestätigungsreaktionen.

- Ausgehende Reaktions-Tools werden durch `channels["matrix"].actions.reactions` gesteuert.
- `react` fügt eine Reaktion zu einem bestimmten Matrix-Ereignis hinzu.
- `reactions` listet die aktuelle Reaktionszusammenfassung für ein bestimmtes Matrix-Ereignis auf.
- `emoji=""` entfernt die eigenen Reaktionen des Bot-Kontos auf dieses Ereignis.
- `remove: true` entfernt nur die angegebene Emoji-Reaktion des Bot-Kontos.

Der Geltungsbereich von Bestätigungsreaktionen wird in dieser Reihenfolge aufgelöst:

- `channels["matrix"].accounts.<accountId>.ackReaction`
- `channels["matrix"].ackReaction`
- `messages.ackReaction`
- Emoji-Fallback der Agent-Identität

Der Geltungsbereich der Bestätigungsreaktion wird in dieser Reihenfolge aufgelöst:

- `channels["matrix"].accounts.<accountId>.ackReactionScope`
- `channels["matrix"].ackReactionScope`
- `messages.ackReactionScope`

Der Modus für Reaktionsbenachrichtigungen wird in dieser Reihenfolge aufgelöst:

- `channels["matrix"].accounts.<accountId>.reactionNotifications`
- `channels["matrix"].reactionNotifications`
- Standard: `own`

Verhalten:

- `reactionNotifications: "own"` leitet hinzugefügte `m.reaction`-Ereignisse weiter, wenn sie auf von Bots verfasste Matrix-Nachrichten zielen.
- `reactionNotifications: "off"` deaktiviert Reaktionssystemereignisse.
- Das Entfernen von Reaktionen wird nicht in Systemereignisse umgewandelt, weil Matrix diese als Redactions und nicht als eigenständige `m.reaction`-Entfernungen darstellt.

## Verlaufskontext

- `channels.matrix.historyLimit` steuert, wie viele aktuelle Raumnachrichten als `InboundHistory` einbezogen werden, wenn eine Matrix-Raumnachricht den Agent auslöst. Fällt auf `messages.groupChat.historyLimit` zurück; wenn beide nicht gesetzt sind, ist der effektive Standardwert `0`. Setze `0`, um dies zu deaktivieren.
- Der Verlauf von Matrix-Räumen gilt nur für Räume. DMs verwenden weiterhin den normalen Sitzungsverlauf.
- Der Verlauf von Matrix-Räumen ist nur für ausstehende Nachrichten: OpenClaw puffert Raumnachrichten, die noch keine Antwort ausgelöst haben, und erstellt dann einen Snapshot dieses Fensters, wenn eine Erwähnung oder ein anderer Auslöser eintrifft.
- Die aktuelle Auslösernachricht wird nicht in `InboundHistory` aufgenommen; sie bleibt für diesen Turn im Hauptteil der eingehenden Nachricht.
- Wiederholungen desselben Matrix-Ereignisses verwenden den ursprünglichen Verlaufs-Snapshot wieder, anstatt zu neueren Raumnachrichten weiterzuwandern.

## Kontextsichtigkeit

Matrix unterstützt die gemeinsame Steuerung `contextVisibility` für ergänzenden Raumkontext wie abgerufenen Antworttext, Thread-Wurzeln und ausstehenden Verlauf.

- `contextVisibility: "all"` ist die Standardeinstellung. Ergänzender Kontext bleibt wie empfangen erhalten.
- `contextVisibility: "allowlist"` filtert ergänzenden Kontext auf Absender, die durch die aktiven Allowlist-Prüfungen für Raum/Benutzer erlaubt sind.
- `contextVisibility: "allowlist_quote"` verhält sich wie `allowlist`, behält aber dennoch ein explizites zitiertes Reply bei.

Diese Einstellung beeinflusst die Sichtbarkeit des ergänzenden Kontexts, nicht, ob die eingehende Nachricht selbst eine Antwort auslösen darf.
Die Autorisierung des Auslösers stammt weiterhin aus `groupPolicy`, `groups`, `groupAllowFrom` und den Einstellungen für DM-Richtlinien.

## DM- und Raumrichtlinie

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

Siehe [Groups](/de/channels/groups) für Verhalten bei Erwähnungsgating und Allowlists.

Pairing-Beispiel für Matrix-DMs:

```bash
openclaw pairing list matrix
openclaw pairing approve matrix <CODE>
```

Wenn ein nicht genehmigter Matrix-Benutzer dir vor der Freigabe weiterhin Nachrichten sendet, verwendet OpenClaw denselben ausstehenden Pairing-Code erneut und kann nach einer kurzen Abkühlzeit erneut eine Erinnerungsantwort senden, anstatt einen neuen Code zu erzeugen.

Siehe [Pairing](/de/channels/pairing) für den gemeinsamen DM-Pairing-Ablauf und das Speicherlayout.

## Reparatur direkter Räume

Wenn der Direktnachrichtenstatus nicht mehr synchron ist, kann OpenClaw veraltete `m.direct`-Zuordnungen haben, die auf alte Einzelräume statt auf die aktuelle DM zeigen. Prüfe die aktuelle Zuordnung für ein Gegenüber mit:

```bash
openclaw matrix direct inspect --user-id @alice:example.org
```

Repariere sie mit:

```bash
openclaw matrix direct repair --user-id @alice:example.org
```

Der Reparaturablauf:

- bevorzugt eine strikte 1:1-DM, die bereits in `m.direct` zugeordnet ist
- greift andernfalls auf eine aktuell beigetretene strikte 1:1-DM mit diesem Benutzer zurück
- erstellt einen neuen Direkt-Raum und schreibt `m.direct` neu, wenn keine funktionsfähige DM existiert

Der Reparaturablauf löscht alte Räume nicht automatisch. Er wählt nur die funktionsfähige DM aus und aktualisiert die Zuordnung, sodass neue Matrix-Sendevorgänge, Verifizierungshinweise und andere Direktnachrichtenabläufe wieder den richtigen Raum ansteuern.

## Exec-Genehmigungen

Matrix kann als nativer Genehmigungs-Client für ein Matrix-Konto fungieren. Die nativen
Routing-Regler für DMs/Kanäle befinden sich weiterhin unter der Konfiguration für Exec-Genehmigungen:

- `channels.matrix.execApprovals.enabled`
- `channels.matrix.execApprovals.approvers` (optional; fällt auf `channels.matrix.dm.allowFrom` zurück)
- `channels.matrix.execApprovals.target` (`dm` | `channel` | `both`, Standard: `dm`)
- `channels.matrix.execApprovals.agentFilter`
- `channels.matrix.execApprovals.sessionFilter`

Genehmigende Personen müssen Matrix-Benutzer-IDs wie `@owner:example.org` sein. Matrix aktiviert native Genehmigungen automatisch, wenn `enabled` nicht gesetzt oder `"auto"` ist und mindestens eine genehmigende Person aufgelöst werden kann. Exec-Genehmigungen verwenden zuerst `execApprovals.approvers` und können auf `channels.matrix.dm.allowFrom` zurückfallen. Plugin-Genehmigungen autorisieren über `channels.matrix.dm.allowFrom`. Setze `enabled: false`, um Matrix explizit als nativen Genehmigungs-Client zu deaktivieren. Genehmigungsanfragen fallen andernfalls auf andere konfigurierte Genehmigungswege oder die Fallback-Richtlinie für Genehmigungen zurück.

Das native Matrix-Routing unterstützt beide Genehmigungsarten:

- `channels.matrix.execApprovals.*` steuert den nativen DM-/Kanal-Fanout-Modus für Matrix-Genehmigungsaufforderungen.
- Exec-Genehmigungen verwenden die Menge genehmigender Personen aus `execApprovals.approvers` oder `channels.matrix.dm.allowFrom`.
- Plugin-Genehmigungen verwenden die DM-Allowlist von Matrix aus `channels.matrix.dm.allowFrom`.
- Matrix-Reaktionskürzel und Nachrichtenaktualisierungen gelten sowohl für Exec- als auch für Plugin-Genehmigungen.

Zustellungsregeln:

- `target: "dm"` sendet Genehmigungsaufforderungen an DMs der genehmigenden Personen
- `target: "channel"` sendet die Aufforderung zurück an den auslösenden Matrix-Raum oder die auslösende DM
- `target: "both"` sendet an DMs der genehmigenden Personen und an den auslösenden Matrix-Raum oder die auslösende DM

Matrix-Genehmigungsaufforderungen setzen Reaktionskürzel auf der primären Genehmigungsnachricht:

- `✅` = einmal erlauben
- `❌` = ablehnen
- `♾️` = immer erlauben, wenn diese Entscheidung durch die effektive Exec-Richtlinie erlaubt ist

Genehmigende Personen können auf diese Nachricht reagieren oder die Fallback-Slash-Befehle verwenden: `/approve <id> allow-once`, `/approve <id> allow-always` oder `/approve <id> deny`.

Nur aufgelöste genehmigende Personen können genehmigen oder ablehnen. Bei Exec-Genehmigungen enthält die Kanalzustellung den Befehlstext, aktiviere also `channel` oder `both` nur in vertrauenswürdigen Räumen.

Kontobezogene Überschreibung:

- `channels.matrix.accounts.<account>.execApprovals`

Verwandte Dokumentation: [Exec approvals](/de/tools/exec-approvals)

## Mehrere Konten

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

Werte auf oberster Ebene unter `channels.matrix` dienen als Standardwerte für benannte Konten, sofern ein Konto sie nicht überschreibt.
Du kannst geerbte Raumeinträge mit `groups.<room>.account` auf ein bestimmtes Matrix-Konto beschränken.
Einträge ohne `account` bleiben für alle Matrix-Konten gemeinsam, und Einträge mit `account: "default"` funktionieren weiterhin, wenn das Standardkonto direkt auf oberster Ebene unter `channels.matrix.*` konfiguriert ist.
Teilweise gemeinsam genutzte Authentifizierungs-Standardwerte erzeugen für sich genommen kein separates implizites Standardkonto. OpenClaw synthetisiert das oberste Konto `default` nur dann, wenn dieses Standardkonto aktuelle Authentifizierung hat (`homeserver` plus `accessToken` oder `homeserver` plus `userId` und `password`); benannte Konten können weiterhin auffindbar bleiben, wenn `homeserver` plus `userId` später durch zwischengespeicherte Anmeldedaten zur Authentifizierung ausreichen.
Wenn Matrix bereits genau ein benanntes Konto hat oder `defaultAccount` auf einen vorhandenen Schlüssel eines benannten Kontos verweist, bewahrt die Reparatur-/Einrichtungs-Hochstufung von Einzelkonto zu Mehrkonten dieses Konto, anstatt einen neuen `accounts.default`-Eintrag zu erstellen. Nur Matrix-Authentifizierungs-/Bootstrap-Schlüssel werden in dieses hochgestufte Konto verschoben; gemeinsam genutzte Zustellungsrichtlinien-Schlüssel bleiben auf oberster Ebene.
Setze `defaultAccount`, wenn OpenClaw für implizites Routing, Probing und CLI-Operationen ein benanntes Matrix-Konto bevorzugen soll.
Wenn mehrere Matrix-Konten konfiguriert sind und eine Konto-ID `default` ist, verwendet OpenClaw dieses Konto implizit, auch wenn `defaultAccount` nicht gesetzt ist.
Wenn du mehrere benannte Konten konfigurierst, setze `defaultAccount` oder übergib `--account <id>` für CLI-Befehle, die auf impliziter Kontoauswahl basieren.
Übergib `--account <id>` an `openclaw matrix verify ...` und `openclaw matrix devices ...`, wenn du diese implizite Auswahl für einen einzelnen Befehl überschreiben möchtest.

Siehe [Configuration reference](/de/gateway/configuration-reference#multi-account-all-channels) für das gemeinsame Muster für mehrere Konten.

## Private/LAN-Homeserver

Standardmäßig blockiert OpenClaw private/interne Matrix-Homeserver zum Schutz vor SSRF, sofern du
nicht explizit pro Konto zustimmst.

Wenn dein Homeserver auf localhost, einer LAN-/Tailscale-IP oder einem internen Hostnamen läuft, aktiviere
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

CLI-Einrichtungsbeispiel:

```bash
openclaw matrix account add \
  --account ops \
  --homeserver http://matrix-synapse:8008 \
  --allow-private-network \
  --access-token syt_ops_xxx
```

Diese Zustimmung erlaubt nur vertrauenswürdige private/interne Ziele. Öffentliche Homeserver im Klartext wie
`http://matrix.example.org:8008` bleiben blockiert. Bevorzuge nach Möglichkeit `https://`.

## Matrix-Datenverkehr über einen Proxy leiten

Wenn deine Matrix-Bereitstellung einen expliziten ausgehenden HTTP(S)-Proxy benötigt, setze `channels.matrix.proxy`:

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
OpenClaw verwendet dieselbe Proxy-Einstellung für Matrix-Laufzeitdatenverkehr und Prüfungen des Kontostatus.

## Zielauflösung

Matrix akzeptiert überall dort diese Zielformen, wo OpenClaw nach einem Raum- oder Benutzerziel fragt:

- Benutzer: `@user:server`, `user:@user:server` oder `matrix:user:@user:server`
- Räume: `!room:server`, `room:!room:server` oder `matrix:room:!room:server`
- Aliasse: `#alias:server`, `channel:#alias:server` oder `matrix:channel:#alias:server`

Die Live-Verzeichnissuche verwendet das angemeldete Matrix-Konto:

- Benutzersuchen fragen das Matrix-Benutzerverzeichnis auf diesem Homeserver ab.
- Raumsuchen akzeptieren explizite Raum-IDs und Aliasse direkt und greifen dann auf die Suche in beigetretenen Raumnamen für dieses Konto zurück.
- Die Suche nach Namen beigetretener Räume erfolgt nach bestem Bemühen. Wenn ein Raumname nicht zu einer ID oder einem Alias aufgelöst werden kann, wird er bei der Laufzeit-Auflösung von Allowlists ignoriert.

## Konfigurationsreferenz

- `enabled`: den Kanal aktivieren oder deaktivieren.
- `name`: optionales Label für das Konto.
- `defaultAccount`: bevorzugte Konto-ID, wenn mehrere Matrix-Konten konfiguriert sind.
- `homeserver`: Homeserver-URL, zum Beispiel `https://matrix.example.org`.
- `network.dangerouslyAllowPrivateNetwork`: diesem Matrix-Konto erlauben, sich mit privaten/internen Homeservern zu verbinden. Aktiviere dies, wenn der Homeserver zu `localhost`, einer LAN-/Tailscale-IP oder einem internen Host wie `matrix-synapse` aufgelöst wird.
- `proxy`: optionale HTTP(S)-Proxy-URL für Matrix-Datenverkehr. Benannte Konten können den Standardwert auf oberster Ebene mit ihrem eigenen `proxy` überschreiben.
- `userId`: vollständige Matrix-Benutzer-ID, zum Beispiel `@bot:example.org`.
- `accessToken`: Access Token für tokenbasierte Authentifizierung. Klartextwerte und SecretRef-Werte werden für `channels.matrix.accessToken` und `channels.matrix.accounts.<id>.accessToken` über env-/file-/exec-Provider unterstützt. Siehe [Secrets Management](/de/gateway/secrets).
- `password`: Passwort für passwortbasierte Anmeldung. Klartextwerte und SecretRef-Werte werden unterstützt.
- `deviceId`: explizite Matrix-Geräte-ID.
- `deviceName`: Anzeigename des Geräts für die Passwortanmeldung.
- `avatarUrl`: gespeicherte Self-Avatar-URL für Profilsynchronisierung und Aktualisierungen über `profile set`.
- `initialSyncLimit`: maximale Anzahl an Ereignissen, die während der Startsynchronisierung abgerufen werden.
- `encryption`: E2EE aktivieren.
- `allowlistOnly`: wenn `true`, wird die Raumrichtlinie `open` auf `allowlist` hochgestuft und alle aktiven DM-Richtlinien außer `disabled` (einschließlich `pairing` und `open`) werden auf `allowlist` erzwungen. Hat keinen Einfluss auf `disabled`-Richtlinien.
- `allowBots`: Nachrichten von anderen konfigurierten OpenClaw-Matrix-Konten zulassen (`true` oder `"mentions"`).
- `groupPolicy`: `open`, `allowlist` oder `disabled`.
- `contextVisibility`: Sichtbarkeitsmodus für ergänzenden Raumkontext (`all`, `allowlist`, `allowlist_quote`).
- `groupAllowFrom`: Allowlist von Benutzer-IDs für Raumdatenverkehr. Einträge sollten vollständige Matrix-Benutzer-IDs sein; nicht aufgelöste Namen werden zur Laufzeit ignoriert.
- `historyLimit`: maximale Anzahl an Raumnachrichten, die als Verlaufskontext für Gruppen einbezogen werden. Fällt auf `messages.groupChat.historyLimit` zurück; wenn beide nicht gesetzt sind, ist der effektive Standardwert `0`. Setze `0`, um dies zu deaktivieren.
- `replyToMode`: `off`, `first`, `all` oder `batched`.
- `markdown`: optionale Markdown-Rendering-Konfiguration für ausgehenden Matrix-Text.
- `streaming`: `off` (Standard), `"partial"`, `"quiet"`, `true` oder `false`. `"partial"` und `true` aktivieren Vorschau-zuerst-Entwurfsaktualisierungen mit normalen Matrix-Textnachrichten. `"quiet"` verwendet nicht benachrichtigende Vorschau-Hinweise für selbstgehostete Push-Regel-Setups. `false` ist gleichbedeutend mit `"off"`.
- `blockStreaming`: `true` aktiviert separate Fortschrittsnachrichten für abgeschlossene Assistant-Blöcke, während das Streaming von Entwurfsvorschauen aktiv ist.
- `threadReplies`: `off`, `inbound` oder `always`.
- `threadBindings`: kanalbezogene Überschreibungen für threadgebundenes Sitzungsrouting und dessen Lebenszyklus.
- `startupVerification`: automatischer Modus für Selbstverifizierungsanfragen beim Start (`if-unverified`, `off`).
- `startupVerificationCooldownHours`: Abkühlzeit vor dem erneuten Versuch automatischer Selbstverifizierungsanfragen beim Start.
- `textChunkLimit`: Chunk-Größe für ausgehende Nachrichten in Zeichen (gilt, wenn `chunkMode` auf `length` gesetzt ist).
- `chunkMode`: `length` teilt Nachrichten nach Zeichenanzahl; `newline` teilt an Zeilengrenzen.
- `responsePrefix`: optionale Zeichenfolge, die allen ausgehenden Antworten für diesen Kanal vorangestellt wird.
- `ackReaction`: optionale Überschreibung der Bestätigungsreaktion für diesen Kanal/dieses Konto.
- `ackReactionScope`: optionale Überschreibung des Geltungsbereichs der Bestätigungsreaktion (`group-mentions`, `group-all`, `direct`, `all`, `none`, `off`).
- `reactionNotifications`: Modus für eingehende Reaktionsbenachrichtigungen (`own`, `off`).
- `mediaMaxMb`: Obergrenze für Mediengröße in MB für ausgehende Sendungen und eingehende Medienverarbeitung.
- `autoJoin`: Richtlinie für automatisches Beitreten bei Einladungen (`always`, `allowlist`, `off`). Standard: `off`. Gilt für alle Matrix-Einladungen, einschließlich DM-artiger Einladungen.
- `autoJoinAllowlist`: Räume/Aliasse, die erlaubt sind, wenn `autoJoin` auf `allowlist` gesetzt ist. Alias-Einträge werden während der Einladungsverarbeitung in Raum-IDs aufgelöst; OpenClaw vertraut nicht auf Alias-Zustände, die vom eingeladenen Raum behauptet werden.
- `dm`: DM-Richtlinienblock (`enabled`, `policy`, `allowFrom`, `sessionScope`, `threadReplies`).
- `dm.policy`: steuert den DM-Zugriff, nachdem OpenClaw dem Raum beigetreten ist und ihn als DM klassifiziert hat. Dies ändert nicht, ob einer Einladung automatisch beigetreten wird.
- `dm.allowFrom`: Einträge sollten vollständige Matrix-Benutzer-IDs sein, außer du hast sie bereits über eine Live-Verzeichnissuche aufgelöst.
- `dm.sessionScope`: `per-user` (Standard) oder `per-room`. Verwende `per-room`, wenn jeder Matrix-DM-Raum einen getrennten Kontext behalten soll, auch wenn das Gegenüber gleich ist.
- `dm.threadReplies`: Überschreibung der Thread-Richtlinie nur für DMs (`off`, `inbound`, `always`). Überschreibt die Einstellung `threadReplies` auf oberster Ebene sowohl für die Platzierung von Antworten als auch für die Sitzungsisolation in DMs.
- `execApprovals`: Matrix-native Zustellung von Exec-Genehmigungen (`enabled`, `approvers`, `target`, `agentFilter`, `sessionFilter`).
- `execApprovals.approvers`: Matrix-Benutzer-IDs, die Exec-Anfragen genehmigen dürfen. Optional, wenn `dm.allowFrom` die genehmigenden Personen bereits identifiziert.
- `execApprovals.target`: `dm | channel | both` (Standard: `dm`).
- `accounts`: benannte kontobezogene Überschreibungen. Werte auf oberster Ebene unter `channels.matrix` dienen als Standardwerte für diese Einträge.
- `groups`: richtlinienbezogene Zuordnung pro Raum. Bevorzuge Raum-IDs oder Aliasse; nicht aufgelöste Raumnamen werden zur Laufzeit ignoriert. Sitzungs-/Gruppenidentität verwendet nach der Auflösung die stabile Raum-ID.
- `groups.<room>.account`: beschränkt einen geerbten Raumeintrag in Setups mit mehreren Konten auf ein bestimmtes Matrix-Konto.
- `groups.<room>.allowBots`: Überschreibung auf Raumebene für von konfigurierten Bots gesendete Nachrichten (`true` oder `"mentions"`).
- `groups.<room>.users`: absenderbezogene Allowlist pro Raum.
- `groups.<room>.tools`: Überschreibungen zum Zulassen/Ablehnen von Tools pro Raum.
- `groups.<room>.autoReply`: Überschreibung des Erwähnungsgatings auf Raumebene. `true` deaktiviert Erwähnungsanforderungen für diesen Raum; `false` erzwingt sie wieder.
- `groups.<room>.skills`: optionaler Skills-Filter auf Raumebene.
- `groups.<room>.systemPrompt`: optionales Snippet für den System Prompt auf Raumebene.
- `rooms`: veralteter Alias für `groups`.
- `actions`: Tool-Gating pro Aktion (`messages`, `reactions`, `pins`, `profile`, `memberInfo`, `channelInfo`, `verification`).

## Verwandt

- [Channels Overview](/de/channels) — alle unterstützten Kanäle
- [Pairing](/de/channels/pairing) — DM-Authentifizierung und Pairing-Ablauf
- [Groups](/de/channels/groups) — Verhalten in Gruppenchats und Erwähnungsgating
- [Channel Routing](/de/channels/channel-routing) — Sitzungsrouting für Nachrichten
- [Security](/de/gateway/security) — Zugriffsmodell und Härtung
