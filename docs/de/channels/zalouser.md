---
read_when:
    - Einrichten von Zalo Personal für OpenClaw
    - Debuggen von Zalo Personal-Login oder Nachrichtenfluss
summary: Unterstützung für persönliche Zalo-Konten über natives zca-js (QR-Login), Funktionen und Konfiguration
title: Zalo Personal
x-i18n:
    generated_at: "2026-04-08T02:14:10Z"
    model: gpt-5.4
    provider: openai
    source_hash: 08f50edb2f4c6fe24972efe5e321f5fd0572c7d29af5c1db808151c7c943dc66
    source_path: channels/zalouser.md
    workflow: 15
---

# Zalo Personal (inoffiziell)

Status: experimentell. Diese Integration automatisiert ein **persönliches Zalo-Konto** über natives `zca-js` in OpenClaw.

> **Warnung:** Dies ist eine inoffizielle Integration und kann zur Sperrung/Blockierung des Kontos führen. Verwendung auf eigenes Risiko.

## Gebündeltes Plugin

Zalo Personal wird in aktuellen OpenClaw-Releases als gebündeltes Plugin ausgeliefert, daher benötigen normale
paketierte Builds keine separate Installation.

Wenn Sie eine ältere Build-Version oder eine benutzerdefinierte Installation ohne Zalo Personal verwenden,
installieren Sie es manuell:

- Über die CLI installieren: `openclaw plugins install @openclaw/zalouser`
- Oder aus einem Quellcode-Checkout: `openclaw plugins install ./path/to/local/zalouser-plugin`
- Details: [Plugins](/de/tools/plugin)

Es ist keine externe `zca`-/`openzca`-CLI-Binärdatei erforderlich.

## Schnelleinrichtung (Anfänger)

1. Stellen Sie sicher, dass das Zalo Personal-Plugin verfügbar ist.
   - Aktuelle paketierte OpenClaw-Releases enthalten es bereits gebündelt.
   - Ältere/benutzerdefinierte Installationen können es mit den obigen Befehlen manuell hinzufügen.
2. Anmelden (QR, auf dem Gateway-Rechner):
   - `openclaw channels login --channel zalouser`
   - Scannen Sie den QR-Code mit der mobilen Zalo-App.
3. Aktivieren Sie den Kanal:

```json5
{
  channels: {
    zalouser: {
      enabled: true,
      dmPolicy: "pairing",
    },
  },
}
```

4. Starten Sie das Gateway neu (oder schließen Sie die Einrichtung ab).
5. Der DM-Zugriff verwendet standardmäßig Pairing; genehmigen Sie beim ersten Kontakt den Pairing-Code.

## Was es ist

- Läuft vollständig im Prozess über `zca-js`.
- Verwendet native Event-Listener zum Empfangen eingehender Nachrichten.
- Sendet Antworten direkt über die JS-API (Text/Medien/Link).
- Entwickelt für Anwendungsfälle mit „persönlichem Konto“, in denen die Zalo Bot API nicht verfügbar ist.

## Benennung

Die Kanal-ID ist `zalouser`, um deutlich zu machen, dass damit ein **persönliches Zalo-Benutzerkonto** automatisiert wird (inoffiziell). `zalo` bleibt für eine mögliche zukünftige offizielle Zalo-API-Integration reserviert.

## IDs finden (Verzeichnis)

Verwenden Sie die Verzeichnis-CLI, um Kontakte/Gruppen und deren IDs zu finden:

```bash
openclaw directory self --channel zalouser
openclaw directory peers list --channel zalouser --query "name"
openclaw directory groups list --channel zalouser --query "work"
```

## Einschränkungen

- Ausgehender Text wird in Blöcke von etwa 2000 Zeichen aufgeteilt (Zalo-Client-Limits).
- Streaming ist standardmäßig blockiert.

## Zugriffskontrolle (DMs)

`channels.zalouser.dmPolicy` unterstützt: `pairing | allowlist | open | disabled` (Standard: `pairing`).

`channels.zalouser.allowFrom` akzeptiert Benutzer-IDs oder Namen. Während der Einrichtung werden Namen über die prozessinterne Kontaktsuche des Plugins in IDs aufgelöst.

Genehmigen über:

- `openclaw pairing list zalouser`
- `openclaw pairing approve zalouser <code>`

## Gruppenzugriff (optional)

- Standard: `channels.zalouser.groupPolicy = "open"` (Gruppen erlaubt). Verwenden Sie `channels.defaults.groupPolicy`, um den Standard zu überschreiben, wenn nichts gesetzt ist.
- Auf eine Allowlist beschränken mit:
  - `channels.zalouser.groupPolicy = "allowlist"`
  - `channels.zalouser.groups` (Schlüssel sollten stabile Gruppen-IDs sein; Namen werden beim Start nach Möglichkeit in IDs aufgelöst)
  - `channels.zalouser.groupAllowFrom` (steuert, welche Absender in erlaubten Gruppen den Bot auslösen können)
- Alle Gruppen blockieren: `channels.zalouser.groupPolicy = "disabled"`.
- Der Konfigurationsassistent kann nach Gruppen-Allowlists fragen.
- Beim Start löst OpenClaw Gruppen-/Benutzernamen in Allowlists in IDs auf und protokolliert die Zuordnung.
- Die Zuordnung der Gruppen-Allowlist erfolgt standardmäßig nur anhand der ID. Nicht aufgelöste Namen werden für die Authentifizierung ignoriert, es sei denn, `channels.zalouser.dangerouslyAllowNameMatching: true` ist aktiviert.
- `channels.zalouser.dangerouslyAllowNameMatching: true` ist ein Break-Glass-Kompatibilitätsmodus, der die Zuordnung anhand veränderlicher Gruppennamen wieder aktiviert.
- Wenn `groupAllowFrom` nicht gesetzt ist, verwendet die Laufzeitumgebung für Absenderprüfungen in Gruppen `allowFrom` als Fallback.
- Absenderprüfungen gelten sowohl für normale Gruppennachrichten als auch für Steuerbefehle (zum Beispiel `/new`, `/reset`).

Beispiel:

```json5
{
  channels: {
    zalouser: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["1471383327500481391"],
      groups: {
        "123456789": { allow: true },
        "Work Chat": { allow: true },
      },
    },
  },
}
```

### Erwähnungsanforderung für Gruppen

- `channels.zalouser.groups.<group>.requireMention` steuert, ob Antworten in Gruppen eine Erwähnung erfordern.
- Auflösungsreihenfolge: exakte Gruppen-ID/-Name -> normalisierter Gruppen-Slug -> `*` -> Standard (`true`).
- Dies gilt sowohl für Gruppen auf der Allowlist als auch für den offenen Gruppenmodus.
- Das Zitieren einer Bot-Nachricht zählt als implizite Erwähnung für die Aktivierung in Gruppen.
- Autorisierte Steuerbefehle (zum Beispiel `/new`) können die Erwähnungsanforderung umgehen.
- Wenn eine Gruppennachricht übersprungen wird, weil eine Erwähnung erforderlich ist, speichert OpenClaw sie als ausstehende Gruppenhistorie und schließt sie in die nächste verarbeitete Gruppennachricht ein.
- Das Standardlimit der Gruppenhistorie ist `messages.groupChat.historyLimit` (Fallback `50`). Sie können es pro Konto mit `channels.zalouser.historyLimit` überschreiben.

Beispiel:

```json5
{
  channels: {
    zalouser: {
      groupPolicy: "allowlist",
      groups: {
        "*": { allow: true, requireMention: true },
        "Work Chat": { allow: true, requireMention: false },
      },
    },
  },
}
```

## Mehrere Konten

Konten werden auf `zalouser`-Profile im OpenClaw-Status abgebildet. Beispiel:

```json5
{
  channels: {
    zalouser: {
      enabled: true,
      defaultAccount: "default",
      accounts: {
        work: { enabled: true, profile: "work" },
      },
    },
  },
}
```

## Tippen, Reaktionen und Zustellbestätigungen

- OpenClaw sendet vor dem Versenden einer Antwort ein Tippen-Ereignis (nach bestem Bemühen).
- Die Nachrichtenreaktionsaktion `react` wird für `zalouser` in Kanalaktionen unterstützt.
  - Verwenden Sie `remove: true`, um ein bestimmtes Reaktions-Emoji aus einer Nachricht zu entfernen.
  - Semantik von Reaktionen: [Reaktionen](/de/tools/reactions)
- Für eingehende Nachrichten, die Ereignismetadaten enthalten, sendet OpenClaw Zustellungs- und Gesehen-Bestätigungen (nach bestem Bemühen).

## Fehlerbehebung

**Login bleibt nicht bestehen:**

- `openclaw channels status --probe`
- Erneut anmelden: `openclaw channels logout --channel zalouser && openclaw channels login --channel zalouser`

**Allowlist-/Gruppenname wurde nicht aufgelöst:**

- Verwenden Sie numerische IDs in `allowFrom`/`groupAllowFrom`/`groups` oder exakte Freundes-/Gruppennamen.

**Von alter CLI-basierter Einrichtung aktualisiert:**

- Entfernen Sie alle Annahmen über alte externe `zca`-Prozesse.
- Der Kanal läuft jetzt vollständig in OpenClaw ohne externe CLI-Binärdateien.

## Verwandt

- [Kanäle im Überblick](/de/channels) — alle unterstützten Kanäle
- [Pairing](/de/channels/pairing) — DM-Authentifizierung und Pairing-Ablauf
- [Gruppen](/de/channels/groups) — Verhalten in Gruppenchats und Erwähnungsanforderung
- [Kanal-Routing](/de/channels/channel-routing) — Sitzungsrouting für Nachrichten
- [Sicherheit](/de/gateway/security) — Zugriffsmodell und Härtung
