---
read_when:
    - Hinzufügen oder Ändern von Doctor-Migrationen
    - Einführung von inkompatiblen Konfigurationsänderungen
summary: 'Doctor-Befehl: Zustandsprüfungen, Konfigurationsmigrationen und Reparaturschritte'
title: Doctor
x-i18n:
    generated_at: "2026-04-08T02:15:50Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3761a222d9db7088f78215575fa84e5896794ad701aa716e8bf9039a4424dca6
    source_path: gateway/doctor.md
    workflow: 15
---

# Doctor

`openclaw doctor` ist das Reparatur- und Migrationstool für OpenClaw. Es behebt veraltete
Konfigurationen/Zustände, prüft den Zustand und bietet umsetzbare Reparaturschritte.

## Schnellstart

```bash
openclaw doctor
```

### Headless / Automatisierung

```bash
openclaw doctor --yes
```

Akzeptiert Standardwerte ohne Rückfrage (einschließlich Neustart-/Dienst-/Sandbox-Reparaturschritten, wenn zutreffend).

```bash
openclaw doctor --repair
```

Wendet empfohlene Reparaturen ohne Rückfrage an (Reparaturen + Neustarts, wenn sicher).

```bash
openclaw doctor --repair --force
```

Wendet auch aggressive Reparaturen an (überschreibt benutzerdefinierte Supervisor-Konfigurationen).

```bash
openclaw doctor --non-interactive
```

Wird ohne Rückfragen ausgeführt und wendet nur sichere Migrationen an (Konfigurationsnormalisierung + Zustandsverschiebungen auf dem Datenträger). Überspringt Neustart-/Dienst-/Sandbox-Aktionen, die menschliche Bestätigung erfordern.
Migrationen von Altzuständen werden bei Erkennung automatisch ausgeführt.

```bash
openclaw doctor --deep
```

Prüft Systemdienste auf zusätzliche Gateway-Installationen (launchd/systemd/schtasks).

Wenn Sie Änderungen vor dem Schreiben prüfen möchten, öffnen Sie zuerst die Konfigurationsdatei:

```bash
cat ~/.openclaw/openclaw.json
```

## Was es tut (Zusammenfassung)

- Optionales Pre-Flight-Update für Git-Installationen (nur interaktiv).
- Frischeprüfung des UI-Protokolls (baut die Control UI neu, wenn das Protokollschema neuer ist).
- Zustandsprüfung + Neustartabfrage.
- Skills-Statusübersicht (geeignet/fehlend/blockiert) und Plugin-Status.
- Konfigurationsnormalisierung für Altwerte.
- Migration der Talk-Konfiguration von den alten flachen `talk.*`-Feldern in `talk.provider` + `talk.providers.<provider>`.
- Browser-Migrationsprüfungen für alte Chrome-Erweiterungskonfigurationen und Chrome-MCP-Bereitschaft.
- OpenCode-Anbieter-Override-Warnungen (`models.providers.opencode` / `models.providers.opencode-go`).
- Codex-OAuth-Shadowing-Warnungen (`models.providers.openai-codex`).
- Prüfung der OAuth-TLS-Voraussetzungen für OpenAI-Codex-OAuth-Profile.
- Migration von Altzuständen auf dem Datenträger (Sitzungen/Agent-Verzeichnis/WhatsApp-Authentifizierung).
- Migration veralteter Vertragsschlüssel in Plugin-Manifests (`speechProviders`, `realtimeTranscriptionProviders`, `realtimeVoiceProviders`, `mediaUnderstandingProviders`, `imageGenerationProviders`, `videoGenerationProviders`, `webFetchProviders`, `webSearchProviders` → `contracts`).
- Migration des Legacy-Cron-Speichers (`jobId`, `schedule.cron`, Delivery-/Payload-Felder auf oberster Ebene, Payload-`provider`, einfache Fallback-Jobs für Webhooks mit `notify: true`).
- Überprüfung von Sitzungs-Sperrdateien und Bereinigung veralteter Sperren.
- Integritäts- und Berechtigungsprüfungen des Zustands (Sitzungen, Transkripte, Zustandsverzeichnis).
- Berechtigungsprüfungen der Konfigurationsdatei (`chmod 600`) bei lokaler Ausführung.
- Zustand der Modellauthentifizierung: prüft OAuth-Ablauf, kann ablaufende Tokens aktualisieren und meldet Cooldown-/Deaktivierungszustände von Auth-Profilen.
- Erkennung zusätzlicher Workspace-Verzeichnisse (`~/openclaw`).
- Reparatur des Sandbox-Images, wenn Sandboxing aktiviert ist.
- Migration alter Dienste und Erkennung zusätzlicher Gateways.
- Migration alter Matrix-Kanalzustände (im Modus `--fix` / `--repair`).
- Prüfungen der Gateway-Laufzeit (Dienst installiert, aber nicht ausgeführt; zwischengespeichertes launchd-Label).
- Kanalstatuswarnungen (vom laufenden Gateway abgefragt).
- Audit der Supervisor-Konfiguration (launchd/systemd/schtasks) mit optionaler Reparatur.
- Best-Practice-Prüfungen für die Gateway-Laufzeit (Node vs. Bun, Pfade von Versionsmanagern).
- Diagnose von Gateway-Portkollisionen (Standard `18789`).
- Sicherheitswarnungen für offene DM-Richtlinien.
- Prüfungen der Gateway-Authentifizierung für den lokalen Token-Modus (bietet Tokenerzeugung an, wenn keine Tokenquelle vorhanden ist; überschreibt keine Token-SecretRef-Konfigurationen).
- Prüfung von systemd linger unter Linux.
- Prüfung der Dateigröße von Workspace-Bootstrap-Dateien (Warnungen bei Abschneidung/nahe am Limit für Kontextdateien).
- Prüfung des Status der Shell-Vervollständigung sowie automatische Installation/Aktualisierung.
- Berechtigkeitsprüfung für den Embedding-Anbieter der Memory-Suche (lokales Modell, Remote-API-Schlüssel oder QMD-Binärdatei).
- Prüfungen der Quellinstallation (pnpm-Workspace-Abweichung, fehlende UI-Assets, fehlende tsx-Binärdatei).
- Schreibt aktualisierte Konfiguration + Wizard-Metadaten.

## Detailliertes Verhalten und Begründung

### 0) Optionales Update (Git-Installationen)

Wenn dies ein Git-Checkout ist und Doctor interaktiv ausgeführt wird, bietet es an,
vor dem Ausführen von Doctor zu aktualisieren (Fetch/Rebase/Build).

### 1) Konfigurationsnormalisierung

Wenn die Konfiguration veraltete Wertformen enthält (zum Beispiel `messages.ackReaction`
ohne kanalspezifische Überschreibung), normalisiert Doctor sie in das aktuelle
Schema.

Dazu gehören auch alte flache Talk-Felder. Die aktuelle öffentliche Talk-Konfiguration ist
`talk.provider` + `talk.providers.<provider>`. Doctor schreibt alte
Formen von `talk.voiceId` / `talk.voiceAliases` / `talk.modelId` / `talk.outputFormat` /
`talk.apiKey` in die Anbieterzuordnung um.

### 2) Migrationen veralteter Konfigurationsschlüssel

Wenn die Konfiguration veraltete Schlüssel enthält, verweigern andere Befehle die Ausführung
und fordern Sie auf, `openclaw doctor` auszuführen.

Doctor wird:

- Erklären, welche veralteten Schlüssel gefunden wurden.
- Die angewendete Migration anzeigen.
- `~/.openclaw/openclaw.json` mit dem aktualisierten Schema neu schreiben.

Das Gateway führt Doctor-Migrationen auch beim Start automatisch aus, wenn es ein
veraltetes Konfigurationsformat erkennt, sodass veraltete Konfigurationen ohne manuelles Eingreifen repariert werden.
Migrationen des Cron-Job-Speichers werden von `openclaw doctor --fix` verarbeitet.

Aktuelle Migrationen:

- `routing.allowFrom` → `channels.whatsapp.allowFrom`
- `routing.groupChat.requireMention` → `channels.whatsapp/telegram/imessage.groups."*".requireMention`
- `routing.groupChat.historyLimit` → `messages.groupChat.historyLimit`
- `routing.groupChat.mentionPatterns` → `messages.groupChat.mentionPatterns`
- `routing.queue` → `messages.queue`
- `routing.bindings` → `bindings` auf oberster Ebene
- `routing.agents`/`routing.defaultAgentId` → `agents.list` + `agents.list[].default`
- veraltete `talk.voiceId`/`talk.voiceAliases`/`talk.modelId`/`talk.outputFormat`/`talk.apiKey` → `talk.provider` + `talk.providers.<provider>`
- `routing.agentToAgent` → `tools.agentToAgent`
- `routing.transcribeAudio` → `tools.media.audio.models`
- `messages.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `messages.tts.providers.<provider>`
- `channels.discord.voice.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `channels.discord.voice.tts.providers.<provider>`
- `channels.discord.accounts.<id>.voice.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `channels.discord.accounts.<id>.voice.tts.providers.<provider>`
- `plugins.entries.voice-call.config.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `plugins.entries.voice-call.config.tts.providers.<provider>`
- `plugins.entries.voice-call.config.provider: "log"` → `"mock"`
- `plugins.entries.voice-call.config.twilio.from` → `plugins.entries.voice-call.config.fromNumber`
- `plugins.entries.voice-call.config.streaming.sttProvider` → `plugins.entries.voice-call.config.streaming.provider`
- `plugins.entries.voice-call.config.streaming.openaiApiKey|sttModel|silenceDurationMs|vadThreshold`
  → `plugins.entries.voice-call.config.streaming.providers.openai.*`
- `bindings[].match.accountID` → `bindings[].match.accountId`
- Bei Kanälen mit benannten `accounts`, aber verbliebenen Kanalwerten auf oberster Ebene für ein Einzelkonto, werden diese kontobezogenen Werte in das für diesen Kanal ausgewählte hochgestufte Konto verschoben (`accounts.default` für die meisten Kanäle; Matrix kann ein vorhandenes passendes benanntes/Standardziel beibehalten)
- `identity` → `agents.list[].identity`
- `agent.*` → `agents.defaults` + `tools.*` (tools/elevated/exec/sandbox/subagents)
- `agent.model`/`allowedModels`/`modelAliases`/`modelFallbacks`/`imageModelFallbacks`
  → `agents.defaults.models` + `agents.defaults.model.primary/fallbacks` + `agents.defaults.imageModel.primary/fallbacks`
- `browser.ssrfPolicy.allowPrivateNetwork` → `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork`
- `browser.profiles.*.driver: "extension"` → `"existing-session"`
- `browser.relayBindHost` entfernen (alte Relay-Einstellung der Erweiterung)

Doctor-Warnungen enthalten auch Hinweise zu Standardkonten für Mehrkonten-Kanäle:

- Wenn zwei oder mehr Einträge in `channels.<channel>.accounts` konfiguriert sind, ohne `channels.<channel>.defaultAccount` oder `accounts.default`, warnt Doctor, dass Fallback-Routing ein unerwartetes Konto auswählen kann.
- Wenn `channels.<channel>.defaultAccount` auf eine unbekannte Konto-ID gesetzt ist, warnt Doctor und listet die konfigurierten Konto-IDs auf.

### 2b) OpenCode-Anbieter-Overrides

Wenn Sie `models.providers.opencode`, `opencode-zen` oder `opencode-go`
manuell hinzugefügt haben, überschreibt das den integrierten OpenCode-Katalog aus `@mariozechner/pi-ai`.
Dadurch können Modelle auf die falsche API gezwungen oder Kosten auf null gesetzt werden. Doctor warnt, damit Sie die Überschreibung entfernen und Routing + Kosten pro Modell wiederherstellen können.

### 2c) Browser-Migration und Chrome-MCP-Bereitschaft

Wenn Ihre Browser-Konfiguration noch auf den entfernten Pfad der Chrome-Erweiterung verweist, normalisiert Doctor
sie auf das aktuelle hostlokale Chrome-MCP-Anbindungsmodell:

- `browser.profiles.*.driver: "extension"` wird zu `"existing-session"`
- `browser.relayBindHost` wird entfernt

Doctor prüft außerdem den hostlokalen Chrome-MCP-Pfad, wenn Sie `defaultProfile:
"user"` oder ein konfiguriertes Profil `existing-session` verwenden:

- prüft, ob Google Chrome auf demselben Host für Standardprofile mit
  automatischer Verbindung installiert ist
- prüft die erkannte Chrome-Version und warnt, wenn sie unter Chrome 144 liegt
- erinnert daran, Remote-Debugging auf der Browser-Inspektionsseite zu aktivieren (zum
  Beispiel `chrome://inspect/#remote-debugging`, `brave://inspect/#remote-debugging`
  oder `edge://inspect/#remote-debugging`)

Doctor kann die browserseitige Einstellung nicht für Sie aktivieren. Hostlokales Chrome MCP
erfordert weiterhin:

- einen Chromium-basierten Browser 144+ auf dem Gateway-/Node-Host
- einen lokal laufenden Browser
- aktiviertes Remote-Debugging in diesem Browser
- das Bestätigen der ersten Zustimmungsabfrage zum Anbinden im Browser

Die Bereitschaft bezieht sich hier nur auf lokale Voraussetzungen für das Anbinden. Existing-session behält
die aktuellen Routenbeschränkungen von Chrome MCP bei; erweiterte Routen wie `responsebody`, PDF-
Export, Download-Interception und Stapelaktionen erfordern weiterhin einen verwalteten
Browser oder ein rohes CDP-Profil.

Diese Prüfung gilt **nicht** für Docker-, Sandbox-, Remote-Browser- oder andere
Headless-Abläufe. Diese verwenden weiterhin rohes CDP.

### 2d) OAuth-TLS-Voraussetzungen

Wenn ein OpenAI-Codex-OAuth-Profil konfiguriert ist, prüft Doctor den OpenAI-
Autorisierungsendpunkt, um zu verifizieren, dass der lokale Node/OpenSSL-TLS-Stack die Zertifikatskette
validieren kann. Wenn die Prüfung mit einem Zertifikatsfehler fehlschlägt (zum
Beispiel `UNABLE_TO_GET_ISSUER_CERT_LOCALLY`, abgelaufenes Zertifikat oder selbstsigniertes Zertifikat),
gibt Doctor plattformspezifische Hinweise zur Behebung aus. Auf macOS mit einem Homebrew-Node ist die
Behebung normalerweise `brew postinstall ca-certificates`. Mit `--deep` wird die Prüfung auch dann ausgeführt,
wenn das Gateway gesund ist.

### 2c) Codex-OAuth-Anbieter-Overrides

Wenn Sie zuvor alte OpenAI-Transporteinstellungen unter
`models.providers.openai-codex` hinzugefügt haben, können diese den integrierten Pfad des Codex-OAuth-
Anbieters verdecken, den neuere Releases automatisch verwenden. Doctor warnt, wenn es diese
alten Transporteinstellungen zusammen mit Codex OAuth sieht, damit Sie die veraltete
Transportüberschreibung entfernen oder umschreiben und das integrierte Routing-/Fallback-Verhalten
wiederherstellen können. Benutzerdefinierte Proxys und reine Header-Overrides werden weiterhin unterstützt und lösen diese Warnung nicht aus.

### 3) Migrationen von Altzuständen (Datenträgerlayout)

Doctor kann ältere Layouts auf dem Datenträger in die aktuelle Struktur migrieren:

- Sitzungsspeicher + Transkripte:
  - von `~/.openclaw/sessions/` nach `~/.openclaw/agents/<agentId>/sessions/`
- Agent-Verzeichnis:
  - von `~/.openclaw/agent/` nach `~/.openclaw/agents/<agentId>/agent/`
- WhatsApp-Authentifizierungszustand (Baileys):
  - aus altem `~/.openclaw/credentials/*.json` (außer `oauth.json`)
  - nach `~/.openclaw/credentials/whatsapp/<accountId>/...` (Standard-Konto-ID: `default`)

Diese Migrationen erfolgen nach bestem Bemühen und sind idempotent; Doctor gibt Warnungen aus, wenn
es alte Ordner als Backups zurücklässt. Das Gateway/die CLI migriert die alten Sitzungen + das Agent-Verzeichnis
beim Start ebenfalls automatisch, sodass Verlauf/Auth/Modelle ohne manuellen Doctor-Lauf im
pfad pro Agent landen. Die WhatsApp-Authentifizierung wird bewusst nur über `openclaw doctor` migriert. Die Normalisierung von Talk-Anbieter/Anbieterzuordnung
vergleicht jetzt nach struktureller Gleichheit, sodass Unterschiede nur in der Schlüsselreihenfolge keine
wiederholten, wirkungslosen Änderungen durch `doctor --fix` mehr auslösen.

### 3a) Migrationen alter Plugin-Manifests

Doctor scannt alle installierten Plugin-Manifeste auf veraltete Capability-Schlüssel
auf oberster Ebene (`speechProviders`, `realtimeTranscriptionProviders`,
`realtimeVoiceProviders`, `mediaUnderstandingProviders`,
`imageGenerationProviders`, `videoGenerationProviders`, `webFetchProviders`,
`webSearchProviders`). Wenn sie gefunden werden, bietet es an, sie in das Objekt `contracts`
zu verschieben und die Manifestdatei direkt neu zu schreiben. Diese Migration ist idempotent;
wenn der Schlüssel `contracts` bereits dieselben Werte enthält, wird der alte Schlüssel entfernt,
ohne die Daten zu duplizieren.

### 3b) Migrationen des Legacy-Cron-Speichers

Doctor prüft auch den Cron-Job-Speicher (`~/.openclaw/cron/jobs.json` standardmäßig,
oder `cron.store`, wenn überschrieben) auf alte Jobformen, die der Scheduler aus Kompatibilitätsgründen
weiterhin akzeptiert.

Aktuelle Bereinigungen für Cron umfassen:

- `jobId` → `id`
- `schedule.cron` → `schedule.expr`
- Payload-Felder auf oberster Ebene (`message`, `model`, `thinking`, ...) → `payload`
- Delivery-Felder auf oberster Ebene (`deliver`, `channel`, `to`, `provider`, ...) → `delivery`
- Delivery-Aliasse für Payload-`provider` → explizites `delivery.channel`
- einfache alte Fallback-Jobs für Webhooks mit `notify: true` → explizites `delivery.mode="webhook"` mit `delivery.to=cron.webhook`

Doctor migriert Jobs mit `notify: true` nur dann automatisch, wenn dies ohne
Verhaltensänderung möglich ist. Wenn ein Job den alten Notify-Fallback mit einem vorhandenen
Nicht-Webhook-Delivery-Modus kombiniert, warnt Doctor und überlässt diesen Job der manuellen Prüfung.

### 3c) Bereinigung von Sitzungssperren

Doctor scannt jedes Sitzungverzeichnis jedes Agenten nach veralteten Schreib-Sperrdateien — Dateien, die
zurückbleiben, wenn eine Sitzung abnormal beendet wurde. Für jede gefundene Sperrdatei meldet es:
den Pfad, die PID, ob die PID noch aktiv ist, das Alter der Sperre und ob sie als
veraltet gilt (tote PID oder älter als 30 Minuten). Im Modus `--fix` / `--repair`
entfernt es veraltete Sperrdateien automatisch; andernfalls gibt es einen Hinweis aus und
weist Sie an, den Befehl mit `--fix` erneut auszuführen.

### 4) Integritätsprüfungen des Zustands (Sitzungspersistenz, Routing und Sicherheit)

Das Zustandsverzeichnis ist der operative Hirnstamm. Wenn es verschwindet, verlieren Sie
Sitzungen, Anmeldedaten, Protokolle und Konfigurationen (es sei denn, Sie haben anderswo Backups).

Doctor prüft:

- **Zustandsverzeichnis fehlt**: warnt vor katastrophalem Zustandsverlust, fragt nach, ob
  das Verzeichnis neu erstellt werden soll, und erinnert Sie daran, dass fehlende Daten nicht wiederhergestellt werden können.
- **Berechtigungen des Zustandsverzeichnisses**: prüft Schreibbarkeit; bietet an, Berechtigungen zu reparieren
  (und gibt einen `chown`-Hinweis aus, wenn ein Eigentümer-/Gruppenmismatch erkannt wird).
- **macOS-Zustandsverzeichnis in Cloud-Sync**: warnt, wenn der Zustand unter iCloud Drive
  (`~/Library/Mobile Documents/com~apple~CloudDocs/...`) oder
  `~/Library/CloudStorage/...` aufgelöst wird, weil Sync-basierte Pfade langsamere I/O
  und Sperr-/Synchronisationsrennen verursachen können.
- **Linux-Zustandsverzeichnis auf SD oder eMMC**: warnt, wenn der Zustand auf eine `mmcblk*`-
  Mount-Quelle aufgelöst wird, weil zufällige I/O auf SD- oder eMMC-Basis langsamer sein und bei
  Sitzungs- und Credential-Schreibvorgängen schneller verschleißen kann.
- **Sitzungsverzeichnisse fehlen**: `sessions/` und das Sitzungs-Speicherverzeichnis sind
  erforderlich, um den Verlauf zu speichern und `ENOENT`-Abstürze zu vermeiden.
- **Transkript-Mismatch**: warnt, wenn bei aktuellen Sitzungseinträgen
  Transkriptdateien fehlen.
- **Hauptsitzung „1-zeiliges JSONL“**: meldet, wenn das Haupttranskript nur eine
  Zeile hat (der Verlauf sammelt sich nicht an).
- **Mehrere Zustandsverzeichnisse**: warnt, wenn mehrere `~/.openclaw`-Ordner über
  Home-Verzeichnisse hinweg vorhanden sind oder wenn `OPENCLAW_STATE_DIR` auf etwas anderes zeigt (der Verlauf kann
  sich zwischen Installationen aufteilen).
- **Erinnerung an den Remote-Modus**: wenn `gateway.mode=remote`, erinnert Doctor Sie daran,
  es auf dem Remote-Host auszuführen (dort liegt der Zustand).
- **Berechtigungen der Konfigurationsdatei**: warnt, wenn `~/.openclaw/openclaw.json`
  für Gruppe/Welt lesbar ist, und bietet an, sie auf `600` zu verschärfen.

### 5) Zustand der Modellauthentifizierung (OAuth-Ablauf)

Doctor prüft OAuth-Profile im Auth-Speicher, warnt, wenn Tokens bald
ablaufen/abgelaufen sind, und kann sie aktualisieren, wenn dies sicher ist. Wenn das Anthropic-
OAuth-/Token-Profil veraltet ist, schlägt es einen Anthropic-API-Schlüssel oder den
Anthropic-Setup-Token-Pfad vor.
Aufforderungen zur Aktualisierung erscheinen nur bei interaktiver Ausführung (TTY); `--non-interactive`
überspringt Aktualisierungsversuche.

Doctor meldet auch Auth-Profile, die vorübergehend nicht nutzbar sind aufgrund von:

- kurzen Cooldowns (Ratenbegrenzungen/Timeouts/Auth-Fehler)
- längeren Deaktivierungen (Abrechnungs-/Guthabenfehler)

### 6) Modellvalidierung für Hooks

Wenn `hooks.gmail.model` gesetzt ist, validiert Doctor die Modellreferenz gegen den
Katalog und die Allowlist und warnt, wenn sie nicht aufgelöst wird oder nicht erlaubt ist.

### 7) Reparatur des Sandbox-Images

Wenn Sandboxing aktiviert ist, prüft Doctor Docker-Images und bietet an, sie zu bauen oder
auf alte Namen zu wechseln, wenn das aktuelle Image fehlt.

### 7b) Laufzeitabhängigkeiten gebündelter Plugins

Doctor verifiziert, dass Laufzeitabhängigkeiten gebündelter Plugins (zum Beispiel die
Laufzeitpakete des Discord-Plugins) im OpenClaw-Installationsstamm vorhanden sind.
Wenn welche fehlen, meldet Doctor die Pakete und installiert sie im Modus
`openclaw doctor --fix` / `openclaw doctor --repair`.

### 8) Migrationen von Gateway-Diensten und Hinweise zur Bereinigung

Doctor erkennt alte Gateway-Dienste (launchd/systemd/schtasks) und
bietet an, sie zu entfernen und den OpenClaw-Dienst mit dem aktuellen Gateway-Port zu installieren.
Es kann außerdem nach zusätzlichen Gateway-ähnlichen Diensten suchen und Hinweise zur Bereinigung ausgeben.
Profilbenannte OpenClaw-Gateway-Dienste gelten als erstklassig und werden nicht als „zusätzlich“
markiert.

### 8b) Matrix-Migration beim Start

Wenn für ein Matrix-Kanalkonto eine ausstehende oder umsetzbare Migration von Altzuständen vorliegt,
erstellt Doctor (im Modus `--fix` / `--repair`) einen Snapshot vor der Migration und
führt dann die Migrationsschritte nach bestem Bemühen aus: Migration des alten Matrix-Zustands und Vorbereitung des alten
verschlüsselten Zustands. Beide Schritte sind nicht fatal; Fehler werden protokolliert und
der Start wird fortgesetzt. Im Nur-Lese-Modus (`openclaw doctor` ohne `--fix`) wird diese Prüfung
vollständig übersprungen.

### 9) Sicherheitswarnungen

Doctor gibt Warnungen aus, wenn ein Anbieter für DMs ohne Allowlist offen ist oder
wenn eine Richtlinie auf gefährliche Weise konfiguriert ist.

### 10) systemd linger (Linux)

Wenn er als systemd-Benutzerdienst läuft, stellt Doctor sicher, dass Linger aktiviert ist, damit das
Gateway nach dem Abmelden aktiv bleibt.

### 11) Workspace-Status (Skills, Plugins und alte Verzeichnisse)

Doctor gibt eine Zusammenfassung des Workspace-Zustands für den Standard-Agenten aus:

- **Skills-Status**: zählt geeignete, an Anforderungen scheiternde und durch Allowlists blockierte Skills.
- **Alte Workspace-Verzeichnisse**: warnt, wenn `~/openclaw` oder andere alte Workspace-Verzeichnisse
  neben dem aktuellen Workspace vorhanden sind.
- **Plugin-Status**: zählt geladene/deaktivierte/fehlerhafte Plugins; listet Plugin-IDs für alle
  Fehler auf; meldet Capabilitys gebündelter Plugins.
- **Plugin-Kompatibilitätswarnungen**: markiert Plugins mit Kompatibilitätsproblemen mit
  der aktuellen Laufzeit.
- **Plugin-Diagnosen**: zeigt beim Laden entstandene Warnungen oder Fehler an, die von der
  Plugin-Registry ausgegeben wurden.

### 11b) Größe von Bootstrap-Dateien

Doctor prüft, ob Bootstrap-Dateien des Workspace (zum Beispiel `AGENTS.md`,
`CLAUDE.md` oder andere injizierte Kontextdateien) nahe am konfigurierten
Zeichenbudget liegen oder es überschreiten. Es meldet pro Datei rohe vs. injizierte Zeichenzahlen,
den Prozentsatz der Abschneidung, die Ursache der Abschneidung (`max/file` oder `max/total`) und die insgesamt injizierten
Zeichen als Anteil am Gesamtbudget. Wenn Dateien abgeschnitten werden oder nahe am
Limit liegen, gibt Doctor Tipps zur Abstimmung von `agents.defaults.bootstrapMaxChars`
und `agents.defaults.bootstrapTotalMaxChars`.

### 11c) Shell-Vervollständigung

Doctor prüft, ob die Tab-Vervollständigung für die aktuelle Shell
(zsh, bash, fish oder PowerShell) installiert ist:

- Wenn das Shell-Profil ein langsames dynamisches Vervollständigungsmuster verwendet
  (`source <(openclaw completion ...)`), aktualisiert Doctor es auf die schnellere
  Variante mit zwischengespeicherter Datei.
- Wenn die Vervollständigung im Profil konfiguriert ist, aber die Cache-Datei fehlt,
  erzeugt Doctor den Cache automatisch neu.
- Wenn überhaupt keine Vervollständigung konfiguriert ist, fordert Doctor zur Installation auf
  (nur im interaktiven Modus; mit `--non-interactive` übersprungen).

Führen Sie `openclaw completion --write-state` aus, um den Cache manuell neu zu erzeugen.

### 12) Prüfungen der Gateway-Authentifizierung (lokaler Token)

Doctor prüft die Bereitschaft der lokalen Gateway-Token-Authentifizierung.

- Wenn der Token-Modus einen Token benötigt und keine Tokenquelle vorhanden ist, bietet Doctor an, einen zu generieren.
- Wenn `gateway.auth.token` durch SecretRef verwaltet wird, aber nicht verfügbar ist, warnt Doctor und überschreibt ihn nicht mit Klartext.
- `openclaw doctor --generate-gateway-token` erzwingt die Generierung nur dann, wenn kein Token-SecretRef konfiguriert ist.

### 12b) SecretRef-bewusste Reparaturen im Nur-Lese-Modus

Einige Reparaturabläufe müssen konfigurierte Anmeldedaten prüfen, ohne das Fail-Fast-Verhalten der Laufzeit zu schwächen.

- `openclaw doctor --fix` verwendet jetzt dasselbe SecretRef-Zusammenfassungsmodell im Nur-Lese-Modus wie Befehle der Statusfamilie für gezielte Konfigurationsreparaturen.
- Beispiel: Die Reparatur von Telegram-`allowFrom` / `groupAllowFrom` mit `@username` versucht, wenn verfügbar, konfigurierte Bot-Anmeldedaten zu verwenden.
- Wenn das Telegram-Bot-Token über SecretRef konfiguriert ist, aber im aktuellen Befehlsablauf nicht verfügbar ist, meldet Doctor, dass die Anmeldedaten konfiguriert, aber nicht verfügbar sind, und überspringt die automatische Auflösung, statt abzustürzen oder das Token fälschlich als fehlend zu melden.

### 13) Zustandsprüfung des Gateways + Neustart

Doctor führt eine Zustandsprüfung durch und bietet an, das Gateway neu zu starten, wenn es
ungesund wirkt.

### 13b) Bereitschaft der Memory-Suche

Doctor prüft, ob der konfigurierte Embedding-Anbieter der Memory-Suche für den
Standard-Agenten bereit ist. Das Verhalten hängt vom konfigurierten Backend und Anbieter ab:

- **QMD-Backend**: prüft, ob die Binärdatei `qmd` verfügbar und startbar ist.
  Wenn nicht, werden Hinweise zur Behebung ausgegeben, einschließlich des npm-Pakets und einer manuellen Option für den Binärpfad.
- **Expliziter lokaler Anbieter**: prüft auf eine lokale Modelldatei oder eine erkannte
  Remote-/herunterladbare Modell-URL. Wenn sie fehlt, wird vorgeschlagen, zu einem Remote-Anbieter zu wechseln.
- **Expliziter Remote-Anbieter** (`openai`, `voyage` usw.): verifiziert, dass ein API-Schlüssel
  in der Umgebung oder im Auth-Speicher vorhanden ist. Gibt umsetzbare Hinweise aus, wenn er fehlt.
- **Automatischer Anbieter**: prüft zuerst die Verfügbarkeit lokaler Modelle und versucht dann jeden Remote-
  Anbieter in der Reihenfolge der automatischen Auswahl.

Wenn ein Gateway-Prüfergebnis verfügbar ist (das Gateway war zum Zeitpunkt der
Prüfung gesund), gleicht Doctor dessen Ergebnis mit der in der CLI sichtbaren Konfiguration ab und vermerkt
jede Abweichung.

Verwenden Sie `openclaw memory status --deep`, um die Embedding-Bereitschaft zur Laufzeit zu verifizieren.

### 14) Kanalstatuswarnungen

Wenn das Gateway gesund ist, führt Doctor eine Abfrage des Kanalstatus aus und meldet
Warnungen mit vorgeschlagenen Behebungen.

### 15) Audit der Supervisor-Konfiguration + Reparatur

Doctor prüft die installierte Supervisor-Konfiguration (launchd/systemd/schtasks) auf
fehlende oder veraltete Standardwerte (z. B. network-online-Abhängigkeiten und
Neustartverzögerung bei systemd). Wenn es eine Abweichung findet, empfiehlt es ein Update und kann
die Dienstdatei/Aufgabe mit den aktuellen Standardwerten neu schreiben.

Hinweise:

- `openclaw doctor` fragt vor dem Neuschreiben der Supervisor-Konfiguration nach.
- `openclaw doctor --yes` akzeptiert die Standard-Reparaturabfragen.
- `openclaw doctor --repair` wendet empfohlene Behebungen ohne Rückfrage an.
- `openclaw doctor --repair --force` überschreibt benutzerdefinierte Supervisor-Konfigurationen.
- Wenn die Token-Authentifizierung ein Token erfordert und `gateway.auth.token` durch SecretRef verwaltet wird, validiert die Installation/Reparatur des Doctor-Dienstes den SecretRef, persistiert aber keine aufgelösten Klartext-Tokenwerte in die Umgebungsmetadaten des Supervisor-Dienstes.
- Wenn die Token-Authentifizierung ein Token erfordert und der konfigurierte Token-SecretRef nicht aufgelöst ist, blockiert Doctor den Installations-/Reparaturpfad mit umsetzbaren Hinweisen.
- Wenn sowohl `gateway.auth.token` als auch `gateway.auth.password` konfiguriert sind und `gateway.auth.mode` nicht gesetzt ist, blockiert Doctor Installation/Reparatur, bis der Modus explizit gesetzt wird.
- Für Linux-User-systemd-Units umfassen Token-Drift-Prüfungen von Doctor jetzt sowohl Quellen aus `Environment=` als auch aus `EnvironmentFile=`, wenn Dienst-Auth-Metadaten verglichen werden.
- Sie können jederzeit eine vollständige Neuschreibung mit `openclaw gateway install --force` erzwingen.

### 16) Gateway-Laufzeit + Portdiagnosen

Doctor prüft die Laufzeit des Dienstes (PID, letzter Exit-Status) und warnt, wenn der
Dienst installiert, aber tatsächlich nicht ausgeführt wird. Es prüft auch auf Portkollisionen
am Gateway-Port (Standard `18789`) und meldet wahrscheinliche Ursachen (Gateway läuft bereits,
SSH-Tunnel).

### 17) Best Practices für die Gateway-Laufzeit

Doctor warnt, wenn der Gateway-Dienst auf Bun oder einem Node-Pfad mit Versionsmanager
(`nvm`, `fnm`, `volta`, `asdf` usw.) läuft. WhatsApp- und Telegram-Kanäle erfordern Node,
und Pfade von Versionsmanagern können nach Upgrades brechen, weil der Dienst Ihre Shell-Initialisierung nicht
lädt. Doctor bietet an, zu einer System-Node-Installation zu migrieren, wenn
verfügbar (Homebrew/apt/choco).

### 18) Konfigurationsschreiben + Wizard-Metadaten

Doctor persistiert alle Konfigurationsänderungen und versieht die
Wizard-Metadaten mit einem Zeitstempel, um den Doctor-Lauf aufzuzeichnen.

### 19) Workspace-Tipps (Backup + Memory-System)

Doctor schlägt ein Workspace-Memory-System vor, wenn keines vorhanden ist, und gibt einen Backup-Hinweis aus,
wenn der Workspace noch nicht unter Git steht.

Siehe [/concepts/agent-workspace](/de/concepts/agent-workspace) für eine vollständige Anleitung zur
Workspace-Struktur und Git-Backups (empfohlen: privates GitHub oder GitLab).
