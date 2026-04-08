---
read_when:
    - Beantwortung häufiger Supportfragen zu Einrichtung, Installation, Onboarding oder Laufzeit
    - Triage von gemeldeten Benutzerproblemen vor einer tieferen Fehlersuche
summary: Häufig gestellte Fragen zu OpenClaw-Einrichtung, Konfiguration und Verwendung
title: FAQ
x-i18n:
    generated_at: "2026-04-08T02:21:39Z"
    model: gpt-5.4
    provider: openai
    source_hash: 001b4605966b45b08108606f76ae937ec348c2179b04cf6fb34fef94833705e6
    source_path: help/faq.md
    workflow: 15
---

# FAQ

Kurze Antworten plus tiefere Fehlersuche für reale Setups (lokale Entwicklung, VPS, Multi-Agent, OAuth/API-Schlüssel, Modell-Failover). Für Laufzeitdiagnosen siehe [Fehlerbehebung](/de/gateway/troubleshooting). Die vollständige Konfigurationsreferenz finden Sie unter [Konfiguration](/de/gateway/configuration).

## Die ersten 60 Sekunden, wenn etwas kaputt ist

1. **Schnellstatus (erste Prüfung)**

   ```bash
   openclaw status
   ```

   Schnelle lokale Zusammenfassung: Betriebssystem + Update, Erreichbarkeit von Gateway/Dienst, Agents/Sitzungen, Provider-Konfiguration + Laufzeitprobleme (wenn das Gateway erreichbar ist).

2. **Teilbarer Bericht (sicher zum Weitergeben)**

   ```bash
   openclaw status --all
   ```

   Schreibgeschützte Diagnose mit Log-Ende (Tokens unkenntlich gemacht).

3. **Daemon- + Port-Status**

   ```bash
   openclaw gateway status
   ```

   Zeigt Supervisor-Laufzeit gegenüber RPC-Erreichbarkeit, die Ziel-URL der Prüfung und welche Konfiguration der Dienst wahrscheinlich verwendet hat.

4. **Tiefe Prüfungen**

   ```bash
   openclaw status --deep
   ```

   Führt eine Live-Gesundheitsprüfung des Gateways aus, einschließlich Kanalprüfungen, wenn unterstützt
   (erfordert ein erreichbares Gateway). Siehe [Health](/de/gateway/health).

5. **Das neueste Log verfolgen**

   ```bash
   openclaw logs --follow
   ```

   Falls RPC nicht verfügbar ist, verwenden Sie stattdessen:

   ```bash
   tail -f "$(ls -t /tmp/openclaw/openclaw-*.log | head -1)"
   ```

   Dateilogs sind von Dienstlogs getrennt; siehe [Logging](/de/logging) und [Fehlerbehebung](/de/gateway/troubleshooting).

6. **Doctor ausführen (Reparaturen)**

   ```bash
   openclaw doctor
   ```

   Repariert/migriert Konfiguration/Zustand + führt Gesundheitsprüfungen aus. Siehe [Doctor](/de/gateway/doctor).

7. **Gateway-Schnappschuss**

   ```bash
   openclaw health --json
   openclaw health --verbose   # shows the target URL + config path on errors
   ```

   Fragt das laufende Gateway nach einem vollständigen Schnappschuss (nur WS). Siehe [Health](/de/gateway/health).

## Schnellstart und Ersteinrichtung

<AccordionGroup>
  <Accordion title="Ich stecke fest, was ist der schnellste Weg wieder heraus?">
    Verwenden Sie einen lokalen KI-Agenten, der **Ihre Maschine sehen** kann. Das ist viel effektiver, als
    in Discord zu fragen, denn die meisten Fälle von „Ich stecke fest“ sind **lokale Konfigurations- oder Umgebungsprobleme**,
    die Helfer aus der Ferne nicht prüfen können.

    - **Claude Code**: [https://www.anthropic.com/claude-code/](https://www.anthropic.com/claude-code/)
    - **OpenAI Codex**: [https://openai.com/codex/](https://openai.com/codex/)

    Diese Tools können das Repo lesen, Befehle ausführen, Logs untersuchen und beim Beheben
    von Problemen auf Maschinenebene helfen (PATH, Dienste, Berechtigungen, Auth-Dateien). Geben Sie ihnen
    den **vollständigen Source-Checkout** über die hackbare (git-)Installation:

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Dadurch wird OpenClaw **aus einem Git-Checkout** installiert, sodass der Agent Code + Docs lesen
    und über die genaue Version nachdenken kann, die Sie ausführen. Sie können jederzeit später wieder auf stable
    zurückwechseln, indem Sie das Installationsprogramm ohne `--install-method git` erneut ausführen.

    Tipp: Bitten Sie den Agenten, die Reparatur **zu planen und zu überwachen** (Schritt für Schritt) und dann nur die
    notwendigen Befehle auszuführen. So bleiben Änderungen klein und leichter prüfbar.

    Wenn Sie einen echten Bug oder eine Korrektur entdecken, reichen Sie bitte ein GitHub-Issue ein oder senden Sie einen PR:
    [https://github.com/openclaw/openclaw/issues](https://github.com/openclaw/openclaw/issues)
    [https://github.com/openclaw/openclaw/pulls](https://github.com/openclaw/openclaw/pulls)

    Beginnen Sie mit diesen Befehlen (teilen Sie die Ausgaben, wenn Sie um Hilfe bitten):

    ```bash
    openclaw status
    openclaw models status
    openclaw doctor
    ```

    Was sie tun:

    - `openclaw status`: schneller Schnappschuss der Gateway-/Agent-Gesundheit + grundlegender Konfiguration.
    - `openclaw models status`: prüft Provider-Authentifizierung + Modellverfügbarkeit.
    - `openclaw doctor`: validiert und repariert häufige Konfigurations-/Statusprobleme.

    Weitere nützliche CLI-Prüfungen: `openclaw status --all`, `openclaw logs --follow`,
    `openclaw gateway status`, `openclaw health --verbose`.

    Schnelle Debug-Schleife: [Die ersten 60 Sekunden, wenn etwas kaputt ist](#first-60-seconds-if-something-is-broken).
    Installationsdokumentation: [Installieren](/de/install), [Installer-Flags](/de/install/installer), [Aktualisieren](/de/install/updating).

  </Accordion>

  <Accordion title="Heartbeat wird ständig übersprungen. Was bedeuten die Gründe dafür?">
    Häufige Gründe, warum Heartbeat übersprungen wird:

    - `quiet-hours`: außerhalb des konfigurierten Aktivitätsfensters
    - `empty-heartbeat-file`: `HEARTBEAT.md` existiert, enthält aber nur leere/header-only-Struktur
    - `no-tasks-due`: Der Aufgabenmodus von `HEARTBEAT.md` ist aktiv, aber keines der Aufgabenintervalle ist bereits fällig
    - `alerts-disabled`: sämtliche Heartbeat-Sichtbarkeit ist deaktiviert (`showOk`, `showAlerts` und `useIndicator` sind alle ausgeschaltet)

    Im Aufgabenmodus werden Fälligkeitszeitstempel erst nach Abschluss
    eines echten Heartbeat-Laufs weitergesetzt. Übersprungene Läufe markieren Aufgaben nicht als abgeschlossen.

    Dokumentation: [Heartbeat](/de/gateway/heartbeat), [Automatisierung & Aufgaben](/de/automation).

  </Accordion>

  <Accordion title="Empfohlene Methode zur Installation und Einrichtung von OpenClaw">
    Das Repo empfiehlt, aus dem Quellcode zu laufen und Onboarding zu verwenden:

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash
    openclaw onboard --install-daemon
    ```

    Der Assistent kann UI-Assets auch automatisch bauen. Nach dem Onboarding führen Sie das Gateway typischerweise auf Port **18789** aus.

    Aus dem Quellcode (Mitwirkende/Entwicklung):

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    pnpm ui:build # auto-installs UI deps on first run
    openclaw onboard
    ```

    Wenn Sie noch keine globale Installation haben, führen Sie es über `pnpm openclaw onboard` aus.

  </Accordion>

  <Accordion title="Wie öffne ich das Dashboard nach dem Onboarding?">
    Der Assistent öffnet Ihren Browser direkt nach dem Onboarding mit einer sauberen (nicht tokenisierten) Dashboard-URL und druckt den Link außerdem in der Zusammenfassung aus. Lassen Sie diesen Tab offen; wenn er nicht gestartet wurde, kopieren Sie die ausgegebene URL auf derselben Maschine und fügen Sie sie ein.
  </Accordion>

  <Accordion title="Wie authentifiziere ich das Dashboard auf localhost im Vergleich zu remote?">
    **Localhost (dieselbe Maschine):**

    - Öffnen Sie `http://127.0.0.1:18789/`.
    - Wenn nach Shared-Secret-Authentifizierung gefragt wird, fügen Sie das konfigurierte Token oder Passwort in die Einstellungen der Control UI ein.
    - Token-Quelle: `gateway.auth.token` (oder `OPENCLAW_GATEWAY_TOKEN`).
    - Passwort-Quelle: `gateway.auth.password` (oder `OPENCLAW_GATEWAY_PASSWORD`).
    - Wenn noch kein Shared Secret konfiguriert ist, erzeugen Sie ein Token mit `openclaw doctor --generate-gateway-token`.

    **Nicht auf localhost:**

    - **Tailscale Serve** (empfohlen): Behalten Sie Bind an loopback bei, führen Sie `openclaw gateway --tailscale serve` aus und öffnen Sie `https://<magicdns>/`. Wenn `gateway.auth.allowTailscale` den Wert `true` hat, erfüllen Identitäts-Header die Authentifizierung für Control UI/WebSocket (kein eingefügtes Shared Secret, setzt vertrauenswürdigen Gateway-Host voraus); HTTP-APIs erfordern weiterhin Shared-Secret-Authentifizierung, es sei denn, Sie verwenden absichtlich `none` für privaten Ingress oder HTTP-Authentifizierung über einen vertrauenswürdigen Proxy.
      Schlechte gleichzeitige Serve-Authentifizierungsversuche vom selben Client werden serialisiert, bevor der Failed-Auth-Limiter sie erfasst, sodass der zweite schlechte Wiederholungsversuch bereits `retry later` anzeigen kann.
    - **Tailnet-Bind**: Führen Sie `openclaw gateway --bind tailnet --token "<token>"` aus (oder konfigurieren Sie Passwortauthentifizierung), öffnen Sie `http://<tailscale-ip>:18789/` und fügen Sie dann das passende Shared Secret in die Dashboard-Einstellungen ein.
    - **Identitätsbewusster Reverse Proxy**: Lassen Sie das Gateway hinter einem nicht-loopback vertrauenswürdigen Proxy, konfigurieren Sie `gateway.auth.mode: "trusted-proxy"` und öffnen Sie dann die Proxy-URL.
    - **SSH-Tunnel**: `ssh -N -L 18789:127.0.0.1:18789 user@host` und dann `http://127.0.0.1:18789/` öffnen. Shared-Secret-Authentifizierung gilt weiterhin über den Tunnel; fügen Sie bei Aufforderung das konfigurierte Token oder Passwort ein.

    Siehe [Dashboard](/web/dashboard) und [Web surfaces](/web) für Bind-Modi und Authentifizierungsdetails.

  </Accordion>

  <Accordion title="Warum gibt es zwei Exec-Genehmigungskonfigurationen für Chat-Genehmigungen?">
    Sie steuern unterschiedliche Ebenen:

    - `approvals.exec`: leitet Genehmigungsaufforderungen an Chat-Ziele weiter
    - `channels.<channel>.execApprovals`: lässt diesen Kanal als nativen Genehmigungs-Client für Exec-Genehmigungen fungieren

    Die Host-Exec-Richtlinie ist weiterhin die eigentliche Genehmigungsschranke. Die Chat-Konfiguration steuert nur, wo Genehmigungsaufforderungen
    erscheinen und wie Personen darauf antworten können.

    In den meisten Setups benötigen Sie **nicht** beides:

    - Wenn der Chat bereits Befehle und Antworten unterstützt, funktioniert `/approve` im selben Chat über den gemeinsamen Pfad.
    - Wenn ein unterstützter nativer Kanal Genehmigende sicher erkennen kann, aktiviert OpenClaw jetzt standardmäßig DM-first-native Genehmigungen automatisch, wenn `channels.<channel>.execApprovals.enabled` nicht gesetzt oder `"auto"` ist.
    - Wenn native Genehmigungskarten/-schaltflächen verfügbar sind, ist diese native UI der primäre Pfad; der Agent sollte nur dann einen manuellen `/approve`-Befehl einfügen, wenn das Tool-Ergebnis angibt, dass Chat-Genehmigungen nicht verfügbar sind oder eine manuelle Genehmigung der einzige Weg ist.
    - Verwenden Sie `approvals.exec` nur, wenn Aufforderungen zusätzlich an andere Chats oder explizite Ops-Räume weitergeleitet werden müssen.
    - Verwenden Sie `channels.<channel>.execApprovals.target: "channel"` oder `"both"` nur, wenn Sie ausdrücklich möchten, dass Genehmigungsaufforderungen in den ursprünglichen Raum/das ursprüngliche Thema zurückgepostet werden.
    - Plugin-Genehmigungen sind wiederum getrennt: Sie verwenden standardmäßig `/approve` im selben Chat, optionales `approvals.plugin`-Forwarding, und nur einige native Kanäle halten zusätzlich native Behandlung für Plugin-Genehmigungen aktiv.

    Kurzfassung: Weiterleitung ist für Routing, native Client-Konfiguration für umfangreichere kanalspezifische UX.
    Siehe [Exec Approvals](/de/tools/exec-approvals).

  </Accordion>

  <Accordion title="Welche Laufzeitumgebung benötige ich?">
    Node **>= 22** ist erforderlich. `pnpm` wird empfohlen. Bun wird für das Gateway **nicht empfohlen**.
  </Accordion>

  <Accordion title="Läuft es auf Raspberry Pi?">
    Ja. Das Gateway ist leichtgewichtig – die Dokumentation nennt **512MB-1GB RAM**, **1 Kern** und etwa **500MB**
    Speicherplatz als ausreichend für den persönlichen Gebrauch und weist darauf hin, dass ein **Raspberry Pi 4 es ausführen kann**.

    Wenn Sie etwas mehr Spielraum möchten (Logs, Medien, andere Dienste), werden **2GB empfohlen**, das ist
    aber kein hartes Minimum.

    Tipp: Ein kleiner Pi/VPS kann das Gateway hosten, und Sie können **Nodes** auf Ihrem Laptop/Telefon koppeln für
    lokalen Bildschirm/Kamera/Canvas oder Befehlsausführung. Siehe [Nodes](/de/nodes).

  </Accordion>

  <Accordion title="Gibt es Tipps für Raspberry-Pi-Installationen?">
    Kurzfassung: Es funktioniert, aber rechnen Sie mit Ecken und Kanten.

    - Verwenden Sie ein **64-Bit**-Betriebssystem und halten Sie Node auf >= 22.
    - Bevorzugen Sie die **hackbare (git-)Installation**, damit Sie Logs sehen und schnell aktualisieren können.
    - Starten Sie ohne Kanäle/Skills und fügen Sie sie dann einzeln hinzu.
    - Wenn Sie auf seltsame Binärprobleme stoßen, ist es meist ein **ARM-Kompatibilitätsproblem**.

    Dokumentation: [Linux](/de/platforms/linux), [Installieren](/de/install).

  </Accordion>

  <Accordion title="Es hängt bei wake up my friend / onboarding will not hatch. Was nun?">
    Dieser Bildschirm hängt davon ab, dass das Gateway erreichbar und authentifiziert ist. Die TUI sendet
    beim ersten Hatch außerdem automatisch „Wake up, my friend!“. Wenn Sie diese Zeile **ohne Antwort**
    sehen und die Tokens bei 0 bleiben, wurde der Agent nie ausgeführt.

    1. Starten Sie das Gateway neu:

    ```bash
    openclaw gateway restart
    ```

    2. Prüfen Sie Status + Auth:

    ```bash
    openclaw status
    openclaw models status
    openclaw logs --follow
    ```

    3. Wenn es weiterhin hängt, führen Sie aus:

    ```bash
    openclaw doctor
    ```

    Wenn das Gateway remote ist, stellen Sie sicher, dass die Tunnel-/Tailscale-Verbindung aktiv ist und die UI
    auf das richtige Gateway zeigt. Siehe [Remote-Zugriff](/de/gateway/remote).

  </Accordion>

  <Accordion title="Kann ich mein Setup auf eine neue Maschine (Mac mini) migrieren, ohne Onboarding erneut zu machen?">
    Ja. Kopieren Sie das **State-Verzeichnis** und den **Workspace**, und führen Sie dann einmal Doctor aus. Dadurch
    bleibt Ihr Bot „genau derselbe“ (Memory, Sitzungsverlauf, Auth und Kanalstatus),
    solange Sie **beide** Orte kopieren:

    1. Installieren Sie OpenClaw auf der neuen Maschine.
    2. Kopieren Sie `$OPENCLAW_STATE_DIR` (Standard: `~/.openclaw`) von der alten Maschine.
    3. Kopieren Sie Ihren Workspace (Standard: `~/.openclaw/workspace`).
    4. Führen Sie `openclaw doctor` aus und starten Sie den Gateway-Dienst neu.

    Dadurch bleiben Konfiguration, Auth-Profile, WhatsApp-Zugangsdaten, Sitzungen und Memory erhalten. Wenn Sie im
    Remote-Modus arbeiten, denken Sie daran, dass der Gateway-Host Sitzungs-Store und Workspace besitzt.

    **Wichtig:** Wenn Sie nur Ihren Workspace zu GitHub committen/pushen, sichern
    Sie **Memory + Bootstrap-Dateien**, aber **nicht** den Sitzungsverlauf oder Auth. Diese befinden sich
    unter `~/.openclaw/` (zum Beispiel `~/.openclaw/agents/<agentId>/sessions/`).

    Verwandt: [Migrieren](/de/install/migrating), [Wo Dinge auf der Festplatte liegen](#where-things-live-on-disk),
    [Agent-Workspace](/de/concepts/agent-workspace), [Doctor](/de/gateway/doctor),
    [Remote-Modus](/de/gateway/remote).

  </Accordion>

  <Accordion title="Wo sehe ich, was in der neuesten Version neu ist?">
    Prüfen Sie das GitHub-Änderungsprotokoll:
    [https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md)

    Die neuesten Einträge stehen oben. Wenn der oberste Abschnitt als **Unreleased** markiert ist, ist der nächste datierte
    Abschnitt die zuletzt ausgelieferte Version. Einträge sind nach **Highlights**, **Changes** und
    **Fixes** gruppiert (plus Docs/andere Abschnitte bei Bedarf).

  </Accordion>

  <Accordion title="Kein Zugriff auf docs.openclaw.ai (SSL-Fehler)">
    Einige Comcast/Xfinity-Verbindungen blockieren `docs.openclaw.ai` fälschlicherweise über Xfinity
    Advanced Security. Deaktivieren Sie dies oder setzen Sie `docs.openclaw.ai` auf die Allowlist und versuchen Sie es erneut.
    Bitte helfen Sie uns beim Entsperren, indem Sie hier melden: [https://spa.xfinity.com/check_url_status](https://spa.xfinity.com/check_url_status).

    Wenn Sie die Website weiterhin nicht erreichen, sind die Docs auf GitHub gespiegelt:
    [https://github.com/openclaw/openclaw/tree/main/docs](https://github.com/openclaw/openclaw/tree/main/docs)

  </Accordion>

  <Accordion title="Unterschied zwischen stable und beta">
    **Stable** und **beta** sind **npm dist-tags**, keine separaten Code-Linien:

    - `latest` = stable
    - `beta` = frühe Build-Version zum Testen

    Üblicherweise landet eine Stable-Veröffentlichung zuerst auf **beta**, dann verschiebt ein expliziter
    Promotionsschritt dieselbe Version auf `latest`. Maintainer können bei Bedarf auch
    direkt auf `latest` veröffentlichen. Deshalb können beta und stable nach der Promotion auf **dieselbe Version** zeigen.

    Was sich geändert hat:
    [https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md)

    Installations-Einzeiler und den Unterschied zwischen beta und dev finden Sie im Akkordeon unten.

  </Accordion>

  <Accordion title="Wie installiere ich die Beta-Version und was ist der Unterschied zwischen beta und dev?">
    **Beta** ist das npm dist-tag `beta` (kann nach der Promotion `latest` entsprechen).
    **Dev** ist der fortlaufende Stand von `main` (git); wenn veröffentlicht, verwendet er das npm dist-tag `dev`.

    Einzeiler (macOS/Linux):

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --beta
    ```

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Windows-Installer (PowerShell):
    [https://openclaw.ai/install.ps1](https://openclaw.ai/install.ps1)

    Mehr Details: [Entwicklungskanäle](/de/install/development-channels) und [Installer-Flags](/de/install/installer).

  </Accordion>

  <Accordion title="Wie probiere ich die neuesten Bits aus?">
    Zwei Möglichkeiten:

    1. **Dev-Kanal (git-Checkout):**

    ```bash
    openclaw update --channel dev
    ```

    Dadurch wechseln Sie zum Branch `main` und aktualisieren aus dem Quellcode.

    2. **Hackbare Installation (von der Installer-Seite):**

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Dadurch erhalten Sie ein lokales Repo, das Sie bearbeiten und dann über git aktualisieren können.

    Wenn Sie lieber manuell sauber klonen möchten, verwenden Sie:

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    ```

    Dokumentation: [Update](/cli/update), [Entwicklungskanäle](/de/install/development-channels),
    [Installieren](/de/install).

  </Accordion>

  <Accordion title="Wie lange dauern Installation und Onboarding normalerweise?">
    Grobe Orientierung:

    - **Installation:** 2-5 Minuten
    - **Onboarding:** 5-15 Minuten, abhängig davon, wie viele Kanäle/Modelle Sie konfigurieren

    Wenn es hängt, verwenden Sie [Installer hängt](#quick-start-and-first-run-setup)
    und die schnelle Debug-Schleife unter [Ich stecke fest](#quick-start-and-first-run-setup).

  </Accordion>

  <Accordion title="Installer hängt? Wie bekomme ich mehr Rückmeldung?">
    Führen Sie den Installer mit **ausführlicher Ausgabe** erneut aus:

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --verbose
    ```

    Beta-Installation mit ausführlicher Ausgabe:

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --beta --verbose
    ```

    Für eine hackbare (git-)Installation:

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git --verbose
    ```

    Windows-Entsprechung (PowerShell):

    ```powershell
    # install.ps1 has no dedicated -Verbose flag yet.
    Set-PSDebug -Trace 1
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -NoOnboard
    Set-PSDebug -Trace 0
    ```

    Weitere Optionen: [Installer-Flags](/de/install/installer).

  </Accordion>

  <Accordion title="Bei der Windows-Installation steht git not found oder openclaw not recognized">
    Zwei häufige Windows-Probleme:

    **1) npm-Fehler spawn git / git not found**

    - Installieren Sie **Git for Windows** und stellen Sie sicher, dass `git` im PATH liegt.
    - Schließen und öffnen Sie PowerShell erneut und führen Sie dann den Installer erneut aus.

    **2) openclaw is not recognized nach der Installation**

    - Ihr globaler npm-bin-Ordner befindet sich nicht im PATH.
    - Prüfen Sie den Pfad:

      ```powershell
      npm config get prefix
      ```

    - Fügen Sie dieses Verzeichnis Ihrem Benutzer-PATH hinzu (unter Windows ist kein `\bin`-Suffix nötig; auf den meisten Systemen ist es `%AppData%\npm`).
    - Schließen und öffnen Sie PowerShell erneut, nachdem Sie den PATH aktualisiert haben.

    Wenn Sie das reibungsloseste Windows-Setup möchten, verwenden Sie **WSL2** statt nativem Windows.
    Dokumentation: [Windows](/de/platforms/windows).

  </Accordion>

  <Accordion title="Windows-Exec-Ausgabe zeigt verstümmelten chinesischen Text - was soll ich tun?">
    Das ist normalerweise eine Codepage-Abweichung der Konsole in nativen Windows-Shells.

    Symptome:

    - Ausgabe von `system.run`/`exec` stellt Chinesisch als Mojibake dar
    - Derselbe Befehl sieht in einem anderen Terminal-Profil korrekt aus

    Schneller Workaround in PowerShell:

    ```powershell
    chcp 65001
    [Console]::InputEncoding = [System.Text.UTF8Encoding]::new($false)
    [Console]::OutputEncoding = [System.Text.UTF8Encoding]::new($false)
    $OutputEncoding = [System.Text.UTF8Encoding]::new($false)
    ```

    Starten Sie dann das Gateway neu und versuchen Sie Ihren Befehl erneut:

    ```powershell
    openclaw gateway restart
    ```

    Wenn Sie dies auf aktuellem OpenClaw weiterhin reproduzieren, verfolgen/melden Sie es unter:

    - [Issue #30640](https://github.com/openclaw/openclaw/issues/30640)

  </Accordion>

  <Accordion title="Die Docs haben meine Frage nicht beantwortet - wie bekomme ich eine bessere Antwort?">
    Verwenden Sie die **hackbare (git-)Installation**, damit Sie den vollständigen Quellcode und die Docs lokal haben, und fragen Sie dann
    Ihren Bot (oder Claude/Codex) _aus diesem Ordner heraus_, damit er das Repo lesen und präzise antworten kann.

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Mehr Details: [Installieren](/de/install) und [Installer-Flags](/de/install/installer).

  </Accordion>

  <Accordion title="Wie installiere ich OpenClaw unter Linux?">
    Kurzantwort: Folgen Sie dem Linux-Leitfaden und führen Sie dann das Onboarding aus.

    - Schneller Linux-Pfad + Dienstinstallation: [Linux](/de/platforms/linux).
    - Vollständige Schritt-für-Schritt-Anleitung: [Getting Started](/de/start/getting-started).
    - Installation + Updates: [Install & updates](/de/install/updating).

  </Accordion>

  <Accordion title="Wie installiere ich OpenClaw auf einem VPS?">
    Jeder Linux-VPS funktioniert. Installieren Sie auf dem Server und verwenden Sie dann SSH/Tailscale, um das Gateway zu erreichen.

    Leitfäden: [exe.dev](/de/install/exe-dev), [Hetzner](/de/install/hetzner), [Fly.io](/de/install/fly).
    Remote-Zugriff: [Gateway remote](/de/gateway/remote).

  </Accordion>

  <Accordion title="Wo sind die Cloud-/VPS-Installationsanleitungen?">
    Wir haben einen **Hosting-Hub** mit den gängigen Providern. Wählen Sie einen aus und folgen Sie der Anleitung:

    - [VPS hosting](/de/vps) (alle Provider an einem Ort)
    - [Fly.io](/de/install/fly)
    - [Hetzner](/de/install/hetzner)
    - [exe.dev](/de/install/exe-dev)

    So funktioniert es in der Cloud: Das **Gateway läuft auf dem Server**, und Sie greifen
    von Ihrem Laptop/Telefon aus über die Control UI (oder Tailscale/SSH) darauf zu. Ihr Zustand + Workspace
    liegen auf dem Server, behandeln Sie den Host also als Quelle der Wahrheit und sichern Sie ihn.

    Sie können **Nodes** (Mac/iOS/Android/headless) mit diesem Cloud-Gateway koppeln, um auf
    lokalen Bildschirm/Kamera/Canvas zuzugreifen oder Befehle auf Ihrem Laptop auszuführen, während
    das Gateway in der Cloud bleibt.

    Hub: [Plattformen](/de/platforms). Remote-Zugriff: [Gateway remote](/de/gateway/remote).
    Nodes: [Nodes](/de/nodes), [Nodes CLI](/cli/nodes).

  </Accordion>

  <Accordion title="Kann ich OpenClaw bitten, sich selbst zu aktualisieren?">
    Kurzantwort: **möglich, aber nicht empfohlen**. Der Update-Ablauf kann das
    Gateway neu starten (wodurch die aktive Sitzung unterbrochen wird), benötigt möglicherweise einen sauberen git-Checkout und
    kann eine Bestätigung verlangen. Sicherer: Führen Sie Updates als Betreiber aus einer Shell aus.

    Verwenden Sie die CLI:

    ```bash
    openclaw update
    openclaw update status
    openclaw update --channel stable|beta|dev
    openclaw update --tag <dist-tag|version>
    openclaw update --no-restart
    ```

    Wenn Sie von einem Agenten aus automatisieren müssen:

    ```bash
    openclaw update --yes --no-restart
    openclaw gateway restart
    ```

    Dokumentation: [Update](/cli/update), [Aktualisieren](/de/install/updating).

  </Accordion>

  <Accordion title="Was macht Onboarding eigentlich?">
    `openclaw onboard` ist der empfohlene Einrichtungsweg. Im **lokalen Modus** führt es Sie durch:

    - **Modell-/Auth-Einrichtung** (Provider-OAuth, API-Schlüssel, Anthropic-Setup-Token sowie lokale Modelloptionen wie LM Studio)
    - **Workspace**-Ort + Bootstrap-Dateien
    - **Gateway-Einstellungen** (Bind/Port/Auth/Tailscale)
    - **Kanäle** (WhatsApp, Telegram, Discord, Mattermost, Signal, iMessage sowie gebündelte Kanal-Plugins wie QQ Bot)
    - **Daemon-Installation** (LaunchAgent auf macOS; systemd-Benutzereinheit auf Linux/WSL2)
    - **Gesundheitsprüfungen** und Auswahl von **Skills**

    Es warnt außerdem, wenn Ihr konfiguriertes Modell unbekannt ist oder Auth fehlt.

  </Accordion>

  <Accordion title="Brauche ich ein Claude- oder OpenAI-Abonnement, um das zu betreiben?">
    Nein. Sie können OpenClaw mit **API-Schlüsseln** (Anthropic/OpenAI/andere) oder mit
    **rein lokalen Modellen** ausführen, damit Ihre Daten auf Ihrem Gerät bleiben. Abonnements (Claude
    Pro/Max oder OpenAI Codex) sind optionale Möglichkeiten, diese Provider zu authentifizieren.

    Für Anthropic in OpenClaw ist die praktische Trennung:

    - **Anthropic-API-Schlüssel**: normale Anthropic-API-Abrechnung
    - **Claude-CLI- / Claude-Abonnement-Authentifizierung in OpenClaw**: Anthropic-Mitarbeiter
      haben uns gesagt, dass diese Nutzung wieder erlaubt ist, und OpenClaw behandelt die Verwendung von `claude -p`
      für diese Integration als genehmigt, sofern Anthropic keine neue
      Richtlinie veröffentlicht

    Für langlebige Gateway-Hosts sind Anthropic-API-Schlüssel weiterhin die besser
    vorhersagbare Einrichtung. OpenAI Codex OAuth wird für externe
    Tools wie OpenClaw ausdrücklich unterstützt.

    OpenClaw unterstützt außerdem andere gehostete abonnementähnliche Optionen, darunter
    **Qwen Cloud Coding Plan**, **MiniMax Coding Plan** und
    **Z.AI / GLM Coding Plan**.

    Dokumentation: [Anthropic](/de/providers/anthropic), [OpenAI](/de/providers/openai),
    [Qwen Cloud](/de/providers/qwen),
    [MiniMax](/de/providers/minimax), [GLM Models](/de/providers/glm),
    [Lokale Modelle](/de/gateway/local-models), [Modelle](/de/concepts/models).

  </Accordion>

  <Accordion title="Kann ich Claude Max ohne API-Schlüssel verwenden?">
    Ja.

    Anthropic-Mitarbeiter haben uns gesagt, dass die Nutzung im Stil von OpenClaw mit Claude CLI wieder erlaubt ist, daher behandelt
    OpenClaw Claude-Abonnement-Authentifizierung und die Nutzung von `claude -p` für diese Integration als genehmigt,
    sofern Anthropic keine neue Richtlinie veröffentlicht. Wenn Sie die am besten vorhersagbare serverseitige Einrichtung möchten,
    verwenden Sie stattdessen einen Anthropic-API-Schlüssel.

  </Accordion>

  <Accordion title="Unterstützen Sie Claude-Abonnement-Authentifizierung (Claude Pro oder Max)?">
    Ja.

    Anthropic-Mitarbeiter haben uns gesagt, dass diese Nutzung wieder erlaubt ist, daher behandelt
    OpenClaw die Wiederverwendung von Claude CLI und die Nutzung von `claude -p` für diese Integration als genehmigt,
    sofern Anthropic keine neue Richtlinie veröffentlicht.

    Anthropic-Setup-Token ist weiterhin als unterstützter OpenClaw-Token-Pfad verfügbar, aber OpenClaw bevorzugt jetzt die Wiederverwendung von Claude CLI und `claude -p`, wenn verfügbar.
    Für Produktions- oder Multi-User-Workloads bleibt Anthropic-API-Key-Authentifizierung die
    sicherere und besser vorhersagbare Wahl. Wenn Sie andere abonnementartige gehostete
    Optionen in OpenClaw möchten, siehe [OpenAI](/de/providers/openai), [Qwen / Model
    Cloud](/de/providers/qwen), [MiniMax](/de/providers/minimax) und [GLM
    Models](/de/providers/glm).

  </Accordion>

<a id="why-am-i-seeing-http-429-ratelimiterror-from-anthropic"></a>
<Accordion title="Warum sehe ich HTTP 429 rate_limit_error von Anthropic?">
Das bedeutet, dass Ihr **Anthropic-Kontingent/Ratelimit** für das aktuelle Zeitfenster erschöpft ist. Wenn Sie
**Claude CLI** verwenden, warten Sie, bis das Fenster zurückgesetzt wird, oder upgraden Sie Ihren Tarif. Wenn Sie
einen **Anthropic-API-Schlüssel** verwenden, prüfen Sie die Anthropic Console
auf Nutzung/Abrechnung und erhöhen Sie bei Bedarf die Limits.

    Wenn die Nachricht konkret lautet:
    `Extra usage is required for long context requests`, versucht die Anfrage,
    die 1M-Context-Beta von Anthropic zu verwenden (`context1m: true`). Das funktioniert nur, wenn Ihre
    Zugangsdaten für Long-Context-Abrechnung berechtigt sind (API-Key-Abrechnung oder der
    Claude-Login-Pfad von OpenClaw mit aktivierter Extra Usage).

    Tipp: Legen Sie ein **Fallback-Modell** fest, damit OpenClaw weiter antworten kann, wenn ein Provider ratelimitiert ist.
    Siehe [Modelle](/cli/models), [OAuth](/de/concepts/oauth) und
    [/gateway/troubleshooting#anthropic-429-extra-usage-required-for-long-context](/de/gateway/troubleshooting#anthropic-429-extra-usage-required-for-long-context).

  </Accordion>

  <Accordion title="Wird AWS Bedrock unterstützt?">
    Ja. OpenClaw hat einen gebündelten **Amazon Bedrock (Converse)**-Provider. Wenn AWS-Umgebungsmarker vorhanden sind, kann OpenClaw den streaming-/textfähigen Bedrock-Katalog automatisch erkennen und ihn als impliziten `amazon-bedrock`-Provider zusammenführen; andernfalls können Sie `plugins.entries.amazon-bedrock.config.discovery.enabled` explizit aktivieren oder einen manuellen Provider-Eintrag hinzufügen. Siehe [Amazon Bedrock](/de/providers/bedrock) und [Model providers](/de/providers/models). Wenn Sie lieber einen verwalteten Schlüsselablauf möchten, ist ein OpenAI-kompatibler Proxy vor Bedrock weiterhin eine gültige Option.
  </Accordion>

  <Accordion title="Wie funktioniert Codex-Authentifizierung?">
    OpenClaw unterstützt **OpenAI Code (Codex)** über OAuth (ChatGPT-Anmeldung). Onboarding kann den OAuth-Ablauf ausführen und setzt das Standardmodell gegebenenfalls auf `openai-codex/gpt-5.4`. Siehe [Model providers](/de/concepts/model-providers) und [Onboarding (CLI)](/de/start/wizard).
  </Accordion>

  <Accordion title="Warum schaltet ChatGPT GPT-5.4 `openai/gpt-5.4` in OpenClaw nicht frei?">
    OpenClaw behandelt die beiden Pfade getrennt:

    - `openai-codex/gpt-5.4` = ChatGPT/Codex OAuth
    - `openai/gpt-5.4` = direkte OpenAI-Platform-API

    In OpenClaw ist die ChatGPT/Codex-Anmeldung an den Pfad `openai-codex/*` gebunden,
    nicht an den direkten Pfad `openai/*`. Wenn Sie den direkten API-Pfad in
    OpenClaw möchten, setzen Sie `OPENAI_API_KEY` (oder die entsprechende OpenAI-Provider-Konfiguration).
    Wenn Sie ChatGPT/Codex-Anmeldung in OpenClaw möchten, verwenden Sie `openai-codex/*`.

  </Accordion>

  <Accordion title="Warum können sich Codex-OAuth-Limits von ChatGPT Web unterscheiden?">
    `openai-codex/*` verwendet den Codex-OAuth-Pfad, und seine nutzbaren Kontingentfenster werden
    von OpenAI verwaltet und hängen vom Tarif ab. In der Praxis können sich diese Limits von
    der Nutzung auf der ChatGPT-Website/-App unterscheiden, selbst wenn beide mit demselben Konto verknüpft sind.

    OpenClaw kann die aktuell sichtbaren Provider-Nutzungs-/Kontingentfenster in
    `openclaw models status` anzeigen, erfindet oder normalisiert jedoch keine ChatGPT-Web-
    Berechtigungen in direkten API-Zugriff. Wenn Sie den direkten OpenAI-Platform-
    Abrechnungs-/Limitpfad möchten, verwenden Sie `openai/*` mit einem API-Schlüssel.

  </Accordion>

  <Accordion title="Unterstützen Sie OpenAI-Abonnement-Authentifizierung (Codex OAuth)?">
    Ja. OpenClaw unterstützt **OpenAI Code (Codex) Subscription OAuth** vollständig.
    OpenAI erlaubt die Verwendung von Subscription OAuth in externen Tools/Workflows
    wie OpenClaw ausdrücklich. Das Onboarding kann den OAuth-Ablauf für Sie ausführen.

    Siehe [OAuth](/de/concepts/oauth), [Model providers](/de/concepts/model-providers) und [Onboarding (CLI)](/de/start/wizard).

  </Accordion>

  <Accordion title="Wie richte ich Gemini CLI OAuth ein?">
    Gemini CLI verwendet einen **Plugin-Auth-Flow**, keine Client-ID und kein Secret in `openclaw.json`.

    Schritte:

    1. Installieren Sie Gemini CLI lokal, sodass `gemini` im `PATH` liegt
       - Homebrew: `brew install gemini-cli`
       - npm: `npm install -g @google/gemini-cli`
    2. Aktivieren Sie das Plugin: `openclaw plugins enable google`
    3. Login: `openclaw models auth login --provider google-gemini-cli --set-default`
    4. Standardmodell nach dem Login: `google-gemini-cli/gemini-3-flash-preview`
    5. Wenn Anfragen fehlschlagen, setzen Sie `GOOGLE_CLOUD_PROJECT` oder `GOOGLE_CLOUD_PROJECT_ID` auf dem Gateway-Host

    Dadurch werden OAuth-Tokens in Auth-Profilen auf dem Gateway-Host gespeichert. Details: [Model providers](/de/concepts/model-providers).

  </Accordion>

  <Accordion title="Ist ein lokales Modell für lockere Chats okay?">
    Normalerweise nein. OpenClaw benötigt großen Kontext und starke Sicherheit; kleine Karten kürzen und lecken. Wenn Sie es unbedingt müssen, verwenden Sie lokal (LM Studio) den **größten** Modell-Build, den Sie ausführen können, und siehe [/gateway/local-models](/de/gateway/local-models). Kleinere/quantisierte Modelle erhöhen das Risiko von Prompt Injection – siehe [Sicherheit](/de/gateway/security).
  </Accordion>

  <Accordion title="Wie halte ich gehosteten Modellverkehr in einer bestimmten Region?">
    Wählen Sie regional gebundene Endpunkte. OpenRouter bietet in den USA gehostete Optionen für MiniMax, Kimi und GLM; wählen Sie die in den USA gehostete Variante, um Daten in der Region zu halten. Sie können weiterhin Anthropic/OpenAI daneben aufführen, indem Sie `models.mode: "merge"` verwenden, damit Fallbacks verfügbar bleiben und gleichzeitig der von Ihnen gewählte regionale Provider respektiert wird.
  </Accordion>

  <Accordion title="Muss ich einen Mac Mini kaufen, um das zu installieren?">
    Nein. OpenClaw läuft auf macOS oder Linux (Windows über WSL2). Ein Mac mini ist optional – manche Leute
    kaufen einen als Always-on-Host, aber auch ein kleiner VPS, Heimserver oder eine Raspberry-Pi-Klasse-Box funktioniert.

    Sie benötigen einen Mac nur **für macOS-exklusive Tools**. Für iMessage verwenden Sie [BlueBubbles](/de/channels/bluebubbles) (empfohlen) – der BlueBubbles-Server läuft auf jedem Mac, und das Gateway kann unter Linux oder anderswo laufen. Wenn Sie andere nur unter macOS verfügbare Tools möchten, führen Sie das Gateway auf einem Mac aus oder koppeln Sie einen macOS-Node.

    Dokumentation: [BlueBubbles](/de/channels/bluebubbles), [Nodes](/de/nodes), [Mac remote mode](/de/platforms/mac/remote).

  </Accordion>

  <Accordion title="Brauche ich einen Mac mini für iMessage-Unterstützung?">
    Sie benötigen **irgendein macOS-Gerät**, das bei Messages angemeldet ist. Es muss **kein** Mac mini sein –
    jeder Mac funktioniert. Verwenden Sie **[BlueBubbles](/de/channels/bluebubbles)** (empfohlen) für iMessage – der BlueBubbles-Server läuft unter macOS, während das Gateway unter Linux oder anderswo laufen kann.

    Häufige Setups:

    - Führen Sie das Gateway auf Linux/VPS aus und den BlueBubbles-Server auf irgendeinem Mac, der bei Messages angemeldet ist.
    - Führen Sie alles auf dem Mac aus, wenn Sie das einfachste Setup mit nur einer Maschine möchten.

    Dokumentation: [BlueBubbles](/de/channels/bluebubbles), [Nodes](/de/nodes),
    [Mac remote mode](/de/platforms/mac/remote).

  </Accordion>

  <Accordion title="Wenn ich einen Mac mini kaufe, um OpenClaw zu betreiben, kann ich ihn mit meinem MacBook Pro verbinden?">
    Ja. Der **Mac mini kann das Gateway ausführen**, und Ihr MacBook Pro kann sich als
    **Node** (Begleitgerät) verbinden. Nodes führen nicht das Gateway aus – sie stellen zusätzliche
    Fähigkeiten wie Bildschirm/Kamera/Canvas und `system.run` auf diesem Gerät bereit.

    Häufiges Muster:

    - Gateway auf dem Mac mini (immer an).
    - MacBook Pro führt die macOS-App oder einen Node-Host aus und koppelt sich mit dem Gateway.
    - Verwenden Sie `openclaw nodes status` / `openclaw nodes list`, um es zu sehen.

    Dokumentation: [Nodes](/de/nodes), [Nodes CLI](/cli/nodes).

  </Accordion>

  <Accordion title="Kann ich Bun verwenden?">
    Bun wird **nicht empfohlen**. Wir sehen Laufzeitprobleme, besonders mit WhatsApp und Telegram.
    Verwenden Sie **Node** für stabile Gateways.

    Wenn Sie trotzdem mit Bun experimentieren möchten, tun Sie dies auf einem Nicht-Produktions-Gateway
    ohne WhatsApp/Telegram.

  </Accordion>

  <Accordion title="Telegram: Was gehört in allowFrom?">
    `channels.telegram.allowFrom` ist **die Telegram-Benutzer-ID des menschlichen Absenders** (numerisch). Es ist nicht der Bot-Benutzername.

    Das Onboarding akzeptiert `@username` als Eingabe und löst ihn in eine numerische ID auf, aber die OpenClaw-Autorisierung verwendet ausschließlich numerische IDs.

    Sicherer (kein Drittanbieter-Bot):

    - Schreiben Sie Ihrem Bot per DM und führen Sie dann `openclaw logs --follow` aus und lesen Sie `from.id`.

    Offizielle Bot API:

    - Schreiben Sie Ihrem Bot per DM und rufen Sie dann `https://api.telegram.org/bot<bot_token>/getUpdates` auf und lesen Sie `message.from.id`.

    Drittanbieter (weniger privat):

    - Schreiben Sie `@userinfobot` oder `@getidsbot` per DM.

    Siehe [/channels/telegram](/de/channels/telegram#access-control-and-activation).

  </Accordion>

  <Accordion title="Können mehrere Personen dieselbe WhatsApp-Nummer mit unterschiedlichen OpenClaw-Instanzen verwenden?">
    Ja, über **Multi-Agent-Routing**. Binden Sie die WhatsApp-**DM** jedes Absenders (Peer `kind: "direct"`, Absender-E.164 wie `+15551234567`) an eine andere `agentId`, damit jede Person ihren eigenen Workspace und Sitzungs-Store erhält. Antworten kommen weiterhin vom **gleichen WhatsApp-Konto**, und die DM-Zugriffskontrolle (`channels.whatsapp.dmPolicy` / `channels.whatsapp.allowFrom`) ist global pro WhatsApp-Konto. Siehe [Multi-Agent Routing](/de/concepts/multi-agent) und [WhatsApp](/de/channels/whatsapp).
  </Accordion>

  <Accordion title='Kann ich einen "Fast-Chat"-Agenten und einen "Opus zum Coden"-Agenten betreiben?'>
    Ja. Verwenden Sie Multi-Agent-Routing: Geben Sie jedem Agenten sein eigenes Standardmodell und binden Sie dann eingehende Routen (Provider-Konto oder bestimmte Peers) an jeden Agenten. Beispielkonfigurationen finden Sie unter [Multi-Agent Routing](/de/concepts/multi-agent). Siehe auch [Modelle](/de/concepts/models) und [Konfiguration](/de/gateway/configuration).
  </Accordion>

  <Accordion title="Funktioniert Homebrew unter Linux?">
    Ja. Homebrew unterstützt Linux (Linuxbrew). Schnelle Einrichtung:

    ```bash
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
    echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"' >> ~/.profile
    eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
    brew install <formula>
    ```

    Wenn Sie OpenClaw über systemd ausführen, stellen Sie sicher, dass der PATH des Dienstes `/home/linuxbrew/.linuxbrew/bin` (oder Ihr brew-Präfix) enthält, damit über `brew` installierte Tools auch in Nicht-Login-Shells aufgelöst werden.
    Neuere Builds stellen Linux-systemd-Diensten außerdem gängige Benutzer-bin-Verzeichnisse voran (z. B. `~/.local/bin`, `~/.npm-global/bin`, `~/.local/share/pnpm`, `~/.bun/bin`) und berücksichtigen `PNPM_HOME`, `NPM_CONFIG_PREFIX`, `BUN_INSTALL`, `VOLTA_HOME`, `ASDF_DATA_DIR`, `NVM_DIR` und `FNM_DIR`, sofern gesetzt.

  </Accordion>

  <Accordion title="Unterschied zwischen der hackbaren git-Installation und npm install">
    - **Hackbare (git-)Installation:** vollständiger Source-Checkout, bearbeitbar, am besten für Mitwirkende.
      Sie bauen lokal und können Code/Docs patchen.
    - **npm install:** globale CLI-Installation, kein Repo, am besten für „einfach ausführen“.
      Updates kommen über npm dist-tags.

    Dokumentation: [Getting started](/de/start/getting-started), [Aktualisieren](/de/install/updating).

  </Accordion>

  <Accordion title="Kann ich später zwischen npm- und git-Installation wechseln?">
    Ja. Installieren Sie die andere Variante und führen Sie dann Doctor aus, damit der Gateway-Dienst auf den neuen Entry-Point zeigt.
    Dadurch werden **Ihre Daten nicht gelöscht** – es wird nur die OpenClaw-Codeinstallation geändert. Ihr Zustand
    (`~/.openclaw`) und Workspace (`~/.openclaw/workspace`) bleiben unberührt.

    Von npm zu git:

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    openclaw doctor
    openclaw gateway restart
    ```

    Von git zu npm:

    ```bash
    npm install -g openclaw@latest
    openclaw doctor
    openclaw gateway restart
    ```

    Doctor erkennt eine Abweichung beim Gateway-Dienst-Entrypoint und bietet an, die Dienstkonfiguration an die aktuelle Installation anzupassen (in der Automatisierung mit `--repair`).

    Tipps zur Sicherung: siehe [Backup-Strategie](#where-things-live-on-disk).

  </Accordion>

  <Accordion title="Soll ich das Gateway auf meinem Laptop oder auf einem VPS ausführen?">
    Kurzantwort: **Wenn Sie 24/7-Zuverlässigkeit möchten, verwenden Sie einen VPS**. Wenn Sie den
    geringsten Aufwand möchten und mit Schlafmodus/Neustarts einverstanden sind, führen Sie es lokal aus.

    **Laptop (lokales Gateway)**

    - **Vorteile:** keine Serverkosten, direkter Zugriff auf lokale Dateien, sichtbares Browserfenster.
    - **Nachteile:** Schlafmodus/Netzwerkunterbrechungen = Verbindungsabbrüche, OS-Updates/Neustarts unterbrechen, muss wach bleiben.

    **VPS / Cloud**

    - **Vorteile:** immer an, stabiles Netzwerk, keine Laptop-Schlafprobleme, leichter am Laufen zu halten.
    - **Nachteile:** oft headless (verwenden Sie Screenshots), nur Remote-Dateizugriff, für Updates ist SSH nötig.

    **OpenClaw-spezifischer Hinweis:** WhatsApp/Telegram/Slack/Mattermost/Discord funktionieren alle gut auf einem VPS. Der einzige echte Kompromiss ist **headless browser** vs. sichtbares Fenster. Siehe [Browser](/de/tools/browser).

    **Empfohlener Standard:** VPS, wenn Sie früher Gateway-Verbindungsabbrüche hatten. Lokal ist großartig, wenn Sie den Mac aktiv verwenden und lokalen Dateizugriff oder UI-Automatisierung mit sichtbarem Browser wünschen.

  </Accordion>

  <Accordion title="Wie wichtig ist es, OpenClaw auf einer dedizierten Maschine auszuführen?">
    Nicht erforderlich, aber **für Zuverlässigkeit und Isolation empfohlen**.

    - **Dedizierter Host (VPS/Mac mini/Pi):** immer an, weniger Unterbrechungen durch Schlafmodus/Neustarts, sauberere Berechtigungen, leichter am Laufen zu halten.
    - **Geteilter Laptop/Desktop:** völlig in Ordnung zum Testen und für aktive Nutzung, aber rechnen Sie mit Pausen, wenn die Maschine schläft oder Updates einspielt.

    Wenn Sie das Beste aus beiden Welten möchten, lassen Sie das Gateway auf einem dedizierten Host und koppeln Sie Ihren Laptop als **Node** für lokale Bildschirm-/Kamera-/Exec-Tools. Siehe [Nodes](/de/nodes).
    Für Sicherheitshinweise lesen Sie [Sicherheit](/de/gateway/security).

  </Accordion>

  <Accordion title="Was sind die Mindestanforderungen an einen VPS und welches Betriebssystem wird empfohlen?">
    OpenClaw ist leichtgewichtig. Für ein einfaches Gateway + einen Chat-Kanal:

    - **Absolutes Minimum:** 1 vCPU, 1GB RAM, ~500MB Speicherplatz.
    - **Empfohlen:** 1-2 vCPU, 2GB RAM oder mehr für Spielraum (Logs, Medien, mehrere Kanäle). Node-Tools und Browser-Automatisierung können ressourcenhungrig sein.

    Betriebssystem: Verwenden Sie **Ubuntu LTS** (oder ein anderes modernes Debian/Ubuntu). Der Linux-Installationspfad ist dort am besten getestet.

    Dokumentation: [Linux](/de/platforms/linux), [VPS hosting](/de/vps).

  </Accordion>

  <Accordion title="Kann ich OpenClaw in einer VM ausführen und was sind die Anforderungen?">
    Ja. Behandeln Sie eine VM wie einen VPS: Sie muss immer eingeschaltet, erreichbar und
    mit genügend RAM für das Gateway und alle aktivierten Kanäle ausgestattet sein.

    Grundlegende Orientierung:

    - **Absolutes Minimum:** 1 vCPU, 1GB RAM.
    - **Empfohlen:** 2GB RAM oder mehr, wenn Sie mehrere Kanäle, Browser-Automatisierung oder Medientools ausführen.
    - **OS:** Ubuntu LTS oder ein anderes modernes Debian/Ubuntu.

    Wenn Sie unter Windows arbeiten, ist **WSL2 die einfachste VM-ähnliche Einrichtung** und bietet die beste Tooling-
    Kompatibilität. Siehe [Windows](/de/platforms/windows), [VPS hosting](/de/vps).
    Wenn Sie macOS in einer VM ausführen, siehe [macOS VM](/de/install/macos-vm).

  </Accordion>
</AccordionGroup>

## Was ist OpenClaw?

<AccordionGroup>
  <Accordion title="Was ist OpenClaw in einem Absatz?">
    OpenClaw ist ein persönlicher KI-Assistent, den Sie auf Ihren eigenen Geräten ausführen. Er antwortet auf den Messaging-Oberflächen, die Sie bereits verwenden (WhatsApp, Telegram, Slack, Mattermost, Discord, Google Chat, Signal, iMessage, WebChat und gebündelte Kanal-Plugins wie QQ Bot) und kann auf unterstützten Plattformen auch Sprache + ein Live-Canvas bereitstellen. Das **Gateway** ist die immer aktive Kontrollebene; der Assistent ist das Produkt.
  </Accordion>

  <Accordion title="Wertversprechen">
    OpenClaw ist nicht „nur ein Claude-Wrapper“. Es ist eine **lokal-first Kontrollebene**, mit der Sie einen
    leistungsfähigen Assistenten auf **Ihrer eigenen Hardware** betreiben können, erreichbar über die Chat-Apps, die Sie bereits nutzen, mit
    zustandsbehafteten Sitzungen, Memory und Tools – ohne die Kontrolle über Ihre Workflows an ein gehostetes
    SaaS abzugeben.

    Highlights:

    - **Ihre Geräte, Ihre Daten:** Führen Sie das Gateway dort aus, wo Sie möchten (Mac, Linux, VPS) und behalten Sie
      Workspace + Sitzungsverlauf lokal.
    - **Echte Kanäle, keine Web-Sandbox:** WhatsApp/Telegram/Slack/Discord/Signal/iMessage/etc,
      plus mobile Sprache und Canvas auf unterstützten Plattformen.
    - **Modellagnostisch:** Nutzen Sie Anthropic, OpenAI, MiniMax, OpenRouter usw. mit Routing pro Agent
      und Failover.
    - **Lokale Option:** Führen Sie lokale Modelle aus, damit **alle Daten auf Ihrem Gerät bleiben können**, wenn Sie möchten.
    - **Multi-Agent-Routing:** Trennen Sie Agenten pro Kanal, Konto oder Aufgabe, jeweils mit eigenem
      Workspace und Standardwerten.
    - **Open Source und hackbar:** Inspizieren, erweitern und selbst hosten ohne Vendor Lock-in.

    Dokumentation: [Gateway](/de/gateway), [Kanäle](/de/channels), [Multi-Agent](/de/concepts/multi-agent),
    [Memory](/de/concepts/memory).

  </Accordion>

  <Accordion title="Ich habe es gerade eingerichtet - was sollte ich zuerst tun?">
    Gute erste Projekte:

    - Eine Website erstellen (WordPress, Shopify oder eine einfache statische Website).
    - Eine mobile App prototypen (Überblick, Screens, API-Plan).
    - Dateien und Ordner organisieren (Aufräumen, Benennen, Tagging).
    - Gmail verbinden und Zusammenfassungen oder Nachfassaktionen automatisieren.

    Es kann große Aufgaben bewältigen, funktioniert aber am besten, wenn Sie sie in Phasen aufteilen und
    Sub agents für parallele Arbeit verwenden.

  </Accordion>

  <Accordion title="Was sind die fünf wichtigsten alltäglichen Anwendungsfälle für OpenClaw?">
    Alltägliche Erfolge sehen typischerweise so aus:

    - **Persönliche Briefings:** Zusammenfassungen von Posteingang, Kalender und Nachrichten, die Sie interessieren.
    - **Recherche und Entwürfe:** schnelle Recherche, Zusammenfassungen und erste Entwürfe für E-Mails oder Dokumente.
    - **Erinnerungen und Nachfassaktionen:** durch Cron oder Heartbeat gesteuerte Anstöße und Checklisten.
    - **Browser-Automatisierung:** Formulare ausfüllen, Daten sammeln und Web-Aufgaben wiederholen.
    - **Geräteübergreifende Koordination:** Senden Sie eine Aufgabe von Ihrem Telefon, lassen Sie das Gateway sie auf einem Server ausführen und erhalten Sie das Ergebnis im Chat zurück.

  </Accordion>

  <Accordion title="Kann OpenClaw bei Lead-Generierung, Outreach, Anzeigen und Blogs für ein SaaS helfen?">
    Ja, für **Recherche, Qualifizierung und Entwürfe**. Es kann Websites scannen, Shortlists erstellen,
    Interessenten zusammenfassen und Entwürfe für Outreach oder Anzeigentexte schreiben.

    Für **Outreach oder Anzeigenläufe** sollte ein Mensch eingebunden bleiben. Vermeiden Sie Spam, halten Sie lokale Gesetze und
    Plattformrichtlinien ein und prüfen Sie alles, bevor es gesendet wird. Das sicherste Muster ist,
    OpenClaw entwerfen zu lassen und Sie genehmigen.

    Dokumentation: [Sicherheit](/de/gateway/security).

  </Accordion>

  <Accordion title="Welche Vorteile hat es gegenüber Claude Code für Webentwicklung?">
    OpenClaw ist ein **persönlicher Assistent** und eine Koordinationsschicht, kein IDE-Ersatz. Verwenden Sie
    Claude Code oder Codex für die schnellste direkte Coding-Schleife innerhalb eines Repos. Verwenden Sie OpenClaw, wenn Sie
    dauerhaftes Memory, geräteübergreifenden Zugriff und Tool-Orchestrierung möchten.

    Vorteile:

    - **Persistentes Memory + Workspace** über Sitzungen hinweg
    - **Plattformübergreifender Zugriff** (WhatsApp, Telegram, TUI, WebChat)
    - **Tool-Orchestrierung** (Browser, Dateien, Planung, Hooks)
    - **Immer aktives Gateway** (auf einem VPS ausführen, von überall interagieren)
    - **Nodes** für lokalen Browser/Bildschirm/Kamera/Exec

    Showcase: [https://openclaw.ai/showcase](https://openclaw.ai/showcase)

  </Accordion>
</AccordionGroup>

## Skills und Automatisierung

<AccordionGroup>
  <Accordion title="Wie passe ich Skills an, ohne das Repo „dirty“ zu halten?">
    Verwenden Sie verwaltete Overrides, statt die Repo-Kopie zu bearbeiten. Legen Sie Ihre Änderungen unter `~/.openclaw/skills/<name>/SKILL.md` ab (oder fügen Sie einen Ordner über `skills.load.extraDirs` in `~/.openclaw/openclaw.json` hinzu). Die Priorität ist `<workspace>/skills` → `<workspace>/.agents/skills` → `~/.agents/skills` → `~/.openclaw/skills` → gebündelt → `skills.load.extraDirs`, sodass verwaltete Overrides weiterhin gegenüber gebündelten Skills gewinnen, ohne git anzufassen. Wenn der Skill global installiert sein soll, aber nur für bestimmte Agenten sichtbar sein darf, behalten Sie die gemeinsame Kopie in `~/.openclaw/skills` und steuern die Sichtbarkeit mit `agents.defaults.skills` und `agents.list[].skills`. Nur Änderungen, die für Upstream geeignet sind, sollten im Repo liegen und als PRs herausgehen.
  </Accordion>

  <Accordion title="Kann ich Skills aus einem benutzerdefinierten Ordner laden?">
    Ja. Fügen Sie zusätzliche Verzeichnisse über `skills.load.extraDirs` in `~/.openclaw/openclaw.json` hinzu (niedrigste Priorität). Die Standardpriorität ist `<workspace>/skills` → `<workspace>/.agents/skills` → `~/.agents/skills` → `~/.openclaw/skills` → gebündelt → `skills.load.extraDirs`. `clawhub` installiert standardmäßig nach `./skills`, was OpenClaw in der nächsten Sitzung als `<workspace>/skills` behandelt. Wenn der Skill nur für bestimmte Agenten sichtbar sein soll, kombinieren Sie das mit `agents.defaults.skills` oder `agents.list[].skills`.
  </Accordion>

  <Accordion title="Wie kann ich unterschiedliche Modelle für unterschiedliche Aufgaben verwenden?">
    Heute werden diese Muster unterstützt:

    - **Cron-Jobs**: Isolierte Jobs können pro Job einen `model`-Override setzen.
    - **Sub-agents**: Leiten Sie Aufgaben an separate Agenten mit unterschiedlichen Standardmodellen weiter.
    - **Ad-hoc-Wechsel**: Verwenden Sie `/model`, um das aktuelle Sitzungsmodell jederzeit zu wechseln.

    Siehe [Cron jobs](/de/automation/cron-jobs), [Multi-Agent Routing](/de/concepts/multi-agent) und [Slash commands](/de/tools/slash-commands).

  </Accordion>

  <Accordion title="Der Bot friert bei schwerer Arbeit ein. Wie lagere ich das aus?">
    Verwenden Sie **Sub-agents** für lange oder parallele Aufgaben. Sub-agents laufen in ihrer eigenen Sitzung,
    geben eine Zusammenfassung zurück und halten Ihren Hauptchat reaktionsfähig.

    Bitten Sie Ihren Bot, „einen Subagenten für diese Aufgabe zu starten“ oder verwenden Sie `/subagents`.
    Verwenden Sie `/status` im Chat, um zu sehen, was das Gateway gerade tut (und ob es beschäftigt ist).

    Token-Tipp: Lange Aufgaben und Sub-agents verbrauchen beide Tokens. Wenn Kosten ein Thema sind, setzen Sie ein
    günstigeres Modell für Sub-agents über `agents.defaults.subagents.model`.

    Dokumentation: [Sub-agents](/de/tools/subagents), [Background Tasks](/de/automation/tasks).

  </Accordion>

  <Accordion title="Wie funktionieren thread-gebundene Subagent-Sitzungen auf Discord?">
    Verwenden Sie Thread-Bindungen. Sie können einen Discord-Thread an ein Subagent- oder Sitzungsziel binden, sodass Folgemeldungen in diesem Thread bei der gebundenen Sitzung bleiben.

    Grundlegender Ablauf:

    - Starten mit `sessions_spawn` unter Verwendung von `thread: true` (und optional `mode: "session"` für persistente Folgeinteraktionen).
    - Oder manuell mit `/focus <target>` binden.
    - Verwenden Sie `/agents`, um den Bindungsstatus zu prüfen.
    - Verwenden Sie `/session idle <duration|off>` und `/session max-age <duration|off>`, um automatisches Unfocus zu steuern.
    - Verwenden Sie `/unfocus`, um den Thread zu lösen.

    Erforderliche Konfiguration:

    - Globale Standardwerte: `session.threadBindings.enabled`, `session.threadBindings.idleHours`, `session.threadBindings.maxAgeHours`.
    - Discord-Overrides: `channels.discord.threadBindings.enabled`, `channels.discord.threadBindings.idleHours`, `channels.discord.threadBindings.maxAgeHours`.
    - Automatische Bindung beim Start: Setzen Sie `channels.discord.threadBindings.spawnSubagentSessions: true`.

    Dokumentation: [Sub-agents](/de/tools/subagents), [Discord](/de/channels/discord), [Configuration Reference](/de/gateway/configuration-reference), [Slash commands](/de/tools/slash-commands).

  </Accordion>

  <Accordion title="Ein Subagent ist fertig geworden, aber das Abschluss-Update ging an die falsche Stelle oder wurde nie gepostet. Was sollte ich prüfen?">
    Prüfen Sie zuerst die aufgelöste Anforderer-Route:

    - Die Zustellung von Subagent-Abschlüssen im Completion-Modus bevorzugt jede gebundene Thread- oder Konversationsroute, wenn eine existiert.
    - Wenn der Completion-Ursprung nur einen Kanal enthält, fällt OpenClaw auf die gespeicherte Route der Anforderer-Sitzung (`lastChannel` / `lastTo` / `lastAccountId`) zurück, damit direkte Zustellung weiterhin funktionieren kann.
    - Wenn weder eine gebundene Route noch eine verwendbare gespeicherte Route existiert, kann direkte Zustellung fehlschlagen und das Ergebnis fällt auf Warteschlangen-Zustellung der Sitzung zurück, statt sofort im Chat gepostet zu werden.
    - Ungültige oder veraltete Ziele können weiterhin Queue-Fallback oder endgültiges Zustellungsversagen erzwingen.
    - Wenn die letzte sichtbare Assistentenantwort des Childs exakt das stille Token `NO_REPLY` / `no_reply` oder exakt `ANNOUNCE_SKIP` ist, unterdrückt OpenClaw die Ankündigung absichtlich, statt veralteten früheren Fortschritt zu posten.
    - Wenn das Child nach nur Tool-Aufrufen ein Timeout hatte, kann die Ankündigung dies in eine kurze Zusammenfassung des Teilfortschritts einklappen, statt rohe Tool-Ausgabe zu wiederholen.

    Debug:

    ```bash
    openclaw tasks show <runId-or-sessionKey>
    ```

    Dokumentation: [Sub-agents](/de/tools/subagents), [Background Tasks](/de/automation/tasks), [Session Tools](/de/concepts/session-tool).

  </Accordion>

  <Accordion title="Cron oder Erinnerungen werden nicht ausgelöst. Was sollte ich prüfen?">
    Cron läuft innerhalb des Gateway-Prozesses. Wenn das Gateway nicht dauerhaft läuft,
    werden geplante Jobs nicht ausgeführt.

    Checkliste:

    - Bestätigen Sie, dass Cron aktiviert ist (`cron.enabled`) und `OPENCLAW_SKIP_CRON` nicht gesetzt ist.
    - Prüfen Sie, ob das Gateway 24/7 läuft (kein Schlafmodus/keine Neustarts).
    - Verifizieren Sie die Zeitzoneneinstellungen für den Job (`--tz` vs. Host-Zeitzone).

    Debug:

    ```bash
    openclaw cron run <jobId>
    openclaw cron runs --id <jobId> --limit 50
    ```

    Dokumentation: [Cron jobs](/de/automation/cron-jobs), [Automatisierung & Aufgaben](/de/automation).

  </Accordion>

  <Accordion title="Cron wurde ausgelöst, aber nichts wurde an den Kanal gesendet. Warum?">
    Prüfen Sie zuerst den Zustellmodus:

    - `--no-deliver` / `delivery.mode: "none"` bedeutet, dass keine externe Nachricht erwartet wird.
    - Fehlendes oder ungültiges Ankündigungsziel (`channel` / `to`) bedeutet, dass der Runner die ausgehende Zustellung übersprungen hat.
    - Fehler bei der Kanalauthentifizierung (`unauthorized`, `Forbidden`) bedeuten, dass der Runner versucht hat zuzustellen, aber die Zugangsdaten dies verhindert haben.
    - Ein stilles isoliertes Ergebnis (`NO_REPLY` / `no_reply` allein) wird als absichtlich nicht zustellbar behandelt, daher unterdrückt der Runner auch die Queue-Fallback-Zustellung.

    Bei isolierten Cron-Jobs besitzt der Runner die endgültige Zustellung. Vom Agenten wird erwartet,
    eine Klartext-Zusammenfassung zurückzugeben, die der Runner versendet. `--no-deliver` hält
    dieses Ergebnis intern; es erlaubt dem Agenten nicht, stattdessen direkt mit dem
    Message-Tool zu senden.

    Debug:

    ```bash
    openclaw cron runs --id <jobId> --limit 50
    openclaw tasks show <runId-or-sessionKey>
    ```

    Dokumentation: [Cron jobs](/de/automation/cron-jobs), [Background Tasks](/de/automation/tasks).

  </Accordion>

  <Accordion title="Warum hat ein isolierter Cron-Lauf Modelle gewechselt oder einmal erneut versucht?">
    Das ist normalerweise der Live-Model-Switch-Pfad, nicht doppelte Planung.

    Isolierter Cron kann einen Laufzeit-Modell-Handoff persistieren und erneut versuchen, wenn der aktive
    Lauf `LiveSessionModelSwitchError` auslöst. Der Wiederholungsversuch behält das umgeschaltete
    Provider-/Modell bei, und wenn der Wechsel einen neuen Auth-Profil-Override mit sich brachte, persistiert Cron
    auch diesen vor dem erneuten Versuch.

    Verwandte Auswahlregeln:

    - Override des Gmail-Hook-Modells gewinnt zuerst, wenn anwendbar.
    - Dann das pro-Job gesetzte `model`.
    - Dann ein eventuell gespeicherter Cron-Sitzungs-Modell-Override.
    - Dann die normale Auswahl von Agent-/Standardmodell.

    Die Wiederholungsschleife ist begrenzt. Nach dem ersten Versuch plus 2 Wechsel-Wiederholungen
    bricht Cron ab, statt endlos zu schleifen.

    Debug:

    ```bash
    openclaw cron runs --id <jobId> --limit 50
    openclaw tasks show <runId-or-sessionKey>
    ```

    Dokumentation: [Cron jobs](/de/automation/cron-jobs), [cron CLI](/cli/cron).

  </Accordion>

  <Accordion title="Wie installiere ich Skills unter Linux?">
    Verwenden Sie native `openclaw skills`-Befehle oder legen Sie Skills in Ihrem Workspace ab. Die macOS-Skills-UI ist unter Linux nicht verfügbar.
    Sie können Skills auf [https://clawhub.ai](https://clawhub.ai) durchsuchen.

    ```bash
    openclaw skills search "calendar"
    openclaw skills search --limit 20
    openclaw skills install <skill-slug>
    openclaw skills install <skill-slug> --version <version>
    openclaw skills install <skill-slug> --force
    openclaw skills update --all
    openclaw skills list --eligible
    openclaw skills check
    ```

    Die native `openclaw skills install` schreibt in das aktive `skills/`-
    Verzeichnis des Workspace. Installieren Sie die separate `clawhub`-CLI nur, wenn Sie eigene Skills veröffentlichen oder
    synchronisieren möchten. Für gemeinsame Installationen über Agenten hinweg legen Sie den Skill unter
    `~/.openclaw/skills` ab und verwenden `agents.defaults.skills` oder
    `agents.list[].skills`, wenn Sie einschränken möchten, welche Agenten ihn sehen dürfen.

  </Accordion>

  <Accordion title="Kann OpenClaw Aufgaben zeitgesteuert oder kontinuierlich im Hintergrund ausführen?">
    Ja. Verwenden Sie den Gateway-Scheduler:

    - **Cron-Jobs** für geplante oder wiederkehrende Aufgaben (bleiben über Neustarts hinweg erhalten).
    - **Heartbeat** für periodische Prüfungen der „Hauptsitzung“.
    - **Isolierte Jobs** für autonome Agenten, die Zusammenfassungen posten oder in Chats zustellen.

    Dokumentation: [Cron jobs](/de/automation/cron-jobs), [Automatisierung & Aufgaben](/de/automation),
    [Heartbeat](/de/gateway/heartbeat).

  </Accordion>

  <Accordion title="Kann ich Apple-macOS-exklusive Skills von Linux aus ausführen?">
    Nicht direkt. macOS-Skills werden durch `metadata.openclaw.os` plus erforderliche Binärdateien begrenzt, und Skills erscheinen nur dann im System-Prompt, wenn sie auf dem **Gateway-Host** zulässig sind. Unter Linux werden nur für `darwin` bestimmte Skills (wie `apple-notes`, `apple-reminders`, `things-mac`) nicht geladen, sofern Sie das Gating nicht überschreiben.

    Es gibt drei unterstützte Muster:

    **Option A - Führen Sie das Gateway auf einem Mac aus (am einfachsten).**
    Führen Sie das Gateway dort aus, wo die macOS-Binärdateien existieren, und verbinden Sie sich dann von Linux aus im [Remote-Modus](#gateway-ports-already-running-and-remote-mode) oder über Tailscale. Die Skills laden normal, weil der Gateway-Host macOS ist.

    **Option B - Verwenden Sie einen macOS-Node (ohne SSH).**
    Führen Sie das Gateway auf Linux aus, koppeln Sie einen macOS-Node (Menüleisten-App) und setzen Sie **Node Run Commands** auf dem Mac auf „Always Ask“ oder „Always Allow“. OpenClaw kann macOS-exklusive Skills als zulässig behandeln, wenn die erforderlichen Binärdateien auf dem Node vorhanden sind. Der Agent führt diese Skills über das Tool `nodes` aus. Wenn Sie „Always Ask“ wählen, fügt die Genehmigung „Always Allow“ im Prompt diesen Befehl der Allowlist hinzu.

    **Option C - macOS-Binärdateien per SSH proxyen (fortgeschritten).**
    Belassen Sie das Gateway auf Linux, aber sorgen Sie dafür, dass die erforderlichen CLI-Binärdateien in SSH-Wrapper aufgelöst werden, die auf einem Mac laufen. Überschreiben Sie dann den Skill so, dass Linux erlaubt ist und er zulässig bleibt.

    1. Erstellen Sie einen SSH-Wrapper für die Binärdatei (Beispiel: `memo` für Apple Notes):

       ```bash
       #!/usr/bin/env bash
       set -euo pipefail
       exec ssh -T user@mac-host /opt/homebrew/bin/memo "$@"
       ```

    2. Legen Sie den Wrapper in den `PATH` auf dem Linux-Host (z. B. `~/bin/memo`).
    3. Überschreiben Sie die Skill-Metadaten (Workspace oder `~/.openclaw/skills`), um Linux zu erlauben:

       ```markdown
       ---
       name: apple-notes
       description: Manage Apple Notes via the memo CLI on macOS.
       metadata: { "openclaw": { "os": ["darwin", "linux"], "requires": { "bins": ["memo"] } } }
       ---
       ```

    4. Starten Sie eine neue Sitzung, damit der Skills-Schnappschuss aktualisiert wird.

  </Accordion>

  <Accordion title="Gibt es eine Notion- oder HeyGen-Integration?">
    Derzeit nicht integriert.

    Optionen:

    - **Benutzerdefinierter Skill / Plugin:** am besten für zuverlässigen API-Zugriff (Notion/HeyGen haben beide APIs).
    - **Browser-Automatisierung:** funktioniert ohne Code, ist aber langsamer und fragiler.

    Wenn Sie den Kontext pro Kunde beibehalten möchten (Agentur-Workflows), ist ein einfaches Muster:

    - Eine Notion-Seite pro Kunde (Kontext + Präferenzen + aktive Arbeit).
    - Bitten Sie den Agenten, diese Seite zu Beginn einer Sitzung abzurufen.

    Wenn Sie eine native Integration möchten, öffnen Sie einen Feature-Request oder bauen Sie einen Skill,
    der auf diese APIs abzielt.

    Skills installieren:

    ```bash
    openclaw skills install <skill-slug>
    openclaw skills update --all
    ```

    Native Installationen landen im aktiven `skills/`-Verzeichnis des Workspace. Für gemeinsame Skills über Agenten hinweg platzieren Sie sie in `~/.openclaw/skills/<name>/SKILL.md`. Wenn nur einige Agenten eine gemeinsame Installation sehen sollen, konfigurieren Sie `agents.defaults.skills` oder `agents.list[].skills`. Einige Skills erwarten über Homebrew installierte Binärdateien; unter Linux bedeutet das Linuxbrew (siehe den Linux-FAQ-Eintrag zu Homebrew oben). Siehe [Skills](/de/tools/skills), [Skills config](/de/tools/skills-config) und [ClawHub](/de/tools/clawhub).

  </Accordion>

  <Accordion title="Wie nutze ich mein bereits angemeldetes Chrome mit OpenClaw?">
    Verwenden Sie das integrierte Browser-Profil `user`, das sich über Chrome DevTools MCP verbindet:

    ```bash
    openclaw browser --browser-profile user tabs
    openclaw browser --browser-profile user snapshot
    ```

    Wenn Sie einen benutzerdefinierten Namen möchten, erstellen Sie ein explizites MCP-Profil:

    ```bash
    openclaw browser create-profile --name chrome-live --driver existing-session
    openclaw browser --browser-profile chrome-live tabs
    ```

    Dieser Pfad ist host-lokal. Wenn das Gateway anderswo läuft, führen Sie entweder einen Node-Host auf der Browser-Maschine aus oder verwenden Sie stattdessen Remote-CDP.

    Aktuelle Einschränkungen von `existing-session` / `user`:

    - Aktionen sind ref-basiert, nicht CSS-Selector-basiert
    - Uploads erfordern `ref` / `inputRef` und unterstützen derzeit jeweils nur eine Datei
    - `responsebody`, PDF-Export, Download-Abfangen und Batch-Aktionen benötigen weiterhin einen verwalteten Browser oder ein rohes CDP-Profil

  </Accordion>
</AccordionGroup>

## Sandboxing und Memory

<AccordionGroup>
  <Accordion title="Gibt es eine eigene Dokumentation zum Sandboxing?">
    Ja. Siehe [Sandboxing](/de/gateway/sandboxing). Für Docker-spezifische Einrichtung (vollständiges Gateway in Docker oder Sandbox-Images) siehe [Docker](/de/install/docker).
  </Accordion>

  <Accordion title="Docker fühlt sich eingeschränkt an - wie aktiviere ich die vollständigen Funktionen?">
    Das Standard-Image ist auf Sicherheit ausgelegt und läuft als Benutzer `node`, daher
    enthält es keine Systempakete, kein Homebrew und keine gebündelten Browser. Für ein vollständigeres Setup:

    - Persistieren Sie `/home/node` mit `OPENCLAW_HOME_VOLUME`, damit Caches erhalten bleiben.
    - Backen Sie Systemabhängigkeiten mit `OPENCLAW_DOCKER_APT_PACKAGES` in das Image ein.
    - Installieren Sie Playwright-Browser über die gebündelte CLI:
      `node /app/node_modules/playwright-core/cli.js install chromium`
    - Setzen Sie `PLAYWRIGHT_BROWSERS_PATH` und stellen Sie sicher, dass der Pfad persistiert wird.

    Dokumentation: [Docker](/de/install/docker), [Browser](/de/tools/browser).

  </Accordion>

  <Accordion title="Kann ich DMs persönlich halten, aber Gruppen öffentlich/in einer Sandbox mit einem Agenten machen?">
    Ja – wenn Ihr privater Verkehr **DMs** sind und Ihr öffentlicher Verkehr **Gruppen**.

    Verwenden Sie `agents.defaults.sandbox.mode: "non-main"`, sodass Gruppen-/Kanalsitzungen (Nicht-Hauptschlüssel) in Docker laufen, während die Haupt-DM-Sitzung auf dem Host bleibt. Beschränken Sie dann über `tools.sandbox.tools`, welche Tools in Sandbox-Sitzungen verfügbar sind.

    Schritt-für-Schritt-Einrichtung + Beispielkonfiguration: [Gruppen: persönliche DMs + öffentliche Gruppen](/de/channels/groups#pattern-personal-dms-public-groups-single-agent)

    Wichtige Konfigurationsreferenz: [Gateway configuration](/de/gateway/configuration-reference#agentsdefaultssandbox)

  </Accordion>

  <Accordion title="Wie binde ich einen Host-Ordner in die Sandbox ein?">
    Setzen Sie `agents.defaults.sandbox.docker.binds` auf `["host:path:mode"]` (z. B. `"/home/user/src:/src:ro"`). Globale + agent-spezifische Binds werden zusammengeführt; agent-spezifische Binds werden ignoriert, wenn `scope: "shared"` gesetzt ist. Verwenden Sie `:ro` für alles Sensible und denken Sie daran, dass Binds die Dateisystemgrenzen der Sandbox umgehen.

    OpenClaw validiert Bind-Quellen sowohl gegen den normalisierten Pfad als auch gegen den kanonischen Pfad, der über den tiefsten existierenden Vorfahren aufgelöst wird. Das bedeutet, dass Ausbrüche über Symlink-Parents weiterhin fehlschlagen, selbst wenn das letzte Pfadsegment noch nicht existiert, und Allowed-Root-Prüfungen auch nach der Symlink-Auflösung weiterhin gelten.

    Siehe [Sandboxing](/de/gateway/sandboxing#custom-bind-mounts) und [Sandbox vs Tool Policy vs Elevated](/de/gateway/sandbox-vs-tool-policy-vs-elevated#bind-mounts-security-quick-check) für Beispiele und Sicherheitshinweise.

  </Accordion>

  <Accordion title="Wie funktioniert Memory?">
    OpenClaw-Memory besteht einfach aus Markdown-Dateien im Agent-Workspace:

    - Tagesnotizen in `memory/YYYY-MM-DD.md`
    - Kuratierte Langzeitnotizen in `MEMORY.md` (nur Haupt-/private Sitzungen)

    OpenClaw führt außerdem einen **stillen Memory-Flush vor der Kompaktierung** aus, um das Modell
    daran zu erinnern, dauerhafte Notizen zu schreiben, bevor automatisch kompaktiert wird. Dies läuft nur, wenn der Workspace
    beschreibbar ist (schreibgeschützte Sandboxes überspringen es). Siehe [Memory](/de/concepts/memory).

  </Accordion>

  <Accordion title="Memory vergisst ständig Dinge. Wie sorge ich dafür, dass sie bleiben?">
    Bitten Sie den Bot, **die Tatsache ins Memory zu schreiben**. Langfristige Notizen gehören in `MEMORY.md`,
    kurzfristiger Kontext in `memory/YYYY-MM-DD.md`.

    Dieser Bereich wird noch verbessert. Es hilft, das Modell daran zu erinnern, Memories zu speichern;
    es weiß dann, was zu tun ist. Wenn es weiterhin vergisst, prüfen Sie, ob das Gateway bei jedem Lauf denselben
    Workspace verwendet.

    Dokumentation: [Memory](/de/concepts/memory), [Agent workspace](/de/concepts/agent-workspace).

  </Accordion>

  <Accordion title="Bleibt Memory für immer bestehen? Was sind die Grenzen?">
    Memory-Dateien liegen auf der Festplatte und bleiben bestehen, bis Sie sie löschen. Die Grenze ist Ihr
    Speicherplatz, nicht das Modell. Der **Sitzungskontext** ist weiterhin durch das Modell-
    Kontextfenster begrenzt, daher können lange Unterhaltungen kompakt werden oder abgeschnitten werden. Deshalb
    gibt es die Memory-Suche – sie holt nur die relevanten Teile zurück in den Kontext.

    Dokumentation: [Memory](/de/concepts/memory), [Kontext](/de/concepts/context).

  </Accordion>

  <Accordion title="Benötigt die semantische Memory-Suche einen OpenAI-API-Schlüssel?">
    Nur wenn Sie **OpenAI-Embeddings** verwenden. Codex OAuth deckt Chat/Completions ab und
    gewährt **keinen** Zugriff auf Embeddings, daher hilft **die Anmeldung mit Codex (OAuth oder dem
    Codex-CLI-Login)** nicht bei der semantischen Memory-Suche. OpenAI-Embeddings
    benötigen weiterhin einen echten API-Schlüssel (`OPENAI_API_KEY` oder `models.providers.openai.apiKey`).

    Wenn Sie keinen Provider explizit setzen, wählt OpenClaw automatisch einen Provider aus, sobald es
    einen API-Schlüssel auflösen kann (Auth-Profile, `models.providers.*.apiKey` oder Umgebungsvariablen).
    Es bevorzugt OpenAI, wenn ein OpenAI-Schlüssel aufgelöst wird, andernfalls Gemini, wenn ein Gemini-Schlüssel
    aufgelöst wird, dann Voyage, dann Mistral. Wenn kein Remote-Schlüssel verfügbar ist, bleibt die Memory-
    Suche deaktiviert, bis Sie sie konfigurieren. Wenn Sie einen lokalen Modellpfad
    konfiguriert haben und dieser vorhanden ist, bevorzugt OpenClaw
    `local`. Ollama wird unterstützt, wenn Sie explizit
    `memorySearch.provider = "ollama"` setzen.

    Wenn Sie lieber lokal bleiben möchten, setzen Sie `memorySearch.provider = "local"` (und optional
    `memorySearch.fallback = "none"`). Wenn Sie Gemini-Embeddings möchten, setzen Sie
    `memorySearch.provider = "gemini"` und stellen Sie `GEMINI_API_KEY` (oder
    `memorySearch.remote.apiKey`) bereit. Wir unterstützen **OpenAI, Gemini, Voyage, Mistral, Ollama oder lokale** Embedding-
    Modelle – siehe [Memory](/de/concepts/memory) für Einrichtungsdetails.

  </Accordion>
</AccordionGroup>

## Wo Dinge auf der Festplatte liegen

<AccordionGroup>
  <Accordion title="Werden alle mit OpenClaw verwendeten Daten lokal gespeichert?">
    Nein – **der Zustand von OpenClaw ist lokal**, aber **externe Dienste sehen weiterhin, was Sie an sie senden**.

    - **Standardmäßig lokal:** Sitzungen, Memory-Dateien, Konfiguration und Workspace liegen auf dem Gateway-Host
      (`~/.openclaw` + Ihr Workspace-Verzeichnis).
    - **Notwendigerweise remote:** Nachrichten, die Sie an Modellanbieter senden (Anthropic/OpenAI/etc.), gehen an
      deren APIs, und Chat-Plattformen (WhatsApp/Telegram/Slack/etc.) speichern Nachrichtendaten auf ihren
      Servern.
    - **Sie kontrollieren den Umfang:** Durch lokale Modelle bleiben Prompts auf Ihrer Maschine, aber Kanal-
      Verkehr läuft weiterhin über die Server des jeweiligen Kanals.

    Verwandt: [Agent workspace](/de/concepts/agent-workspace), [Memory](/de/concepts/memory).

  </Accordion>

  <Accordion title="Wo speichert OpenClaw seine Daten?">
    Alles liegt unter `$OPENCLAW_STATE_DIR` (Standard: `~/.openclaw`):

    | Path                                                            | Purpose                                                            |
    | --------------------------------------------------------------- | ------------------------------------------------------------------ |
    | `$OPENCLAW_STATE_DIR/openclaw.json`                             | Hauptkonfiguration (JSON5)                                         |
    | `$OPENCLAW_STATE_DIR/credentials/oauth.json`                    | Legacy-OAuth-Import (bei erster Nutzung in Auth-Profile kopiert)   |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/auth-profiles.json` | Auth-Profile (OAuth, API-Schlüssel und optional `keyRef`/`tokenRef`) |
    | `$OPENCLAW_STATE_DIR/secrets.json`                              | Optional dateigestützte Secret-Payload für `file`-SecretRef-Provider |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/auth.json`          | Legacy-Kompatibilitätsdatei (statische `api_key`-Einträge bereinigt) |
    | `$OPENCLAW_STATE_DIR/credentials/`                              | Provider-Zustand (z. B. `whatsapp/<accountId>/creds.json`)         |
    | `$OPENCLAW_STATE_DIR/agents/`                                   | Zustand pro Agent (`agentDir` + Sitzungen)                         |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/sessions/`                | Unterhaltungsverlauf & Zustand (pro Agent)                         |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/sessions/sessions.json`   | Sitzungsmetadaten (pro Agent)                                      |

    Legacy-Single-Agent-Pfad: `~/.openclaw/agent/*` (migriert durch `openclaw doctor`).

    Ihr **Workspace** (`AGENTS.md`, Memory-Dateien, Skills usw.) ist getrennt und wird über `agents.defaults.workspace` konfiguriert (Standard: `~/.openclaw/workspace`).

  </Accordion>

  <Accordion title="Wo sollten AGENTS.md / SOUL.md / USER.md / MEMORY.md liegen?">
    Diese Dateien liegen im **Agent-Workspace**, nicht in `~/.openclaw`.

    - **Workspace (pro Agent)**: `AGENTS.md`, `SOUL.md`, `IDENTITY.md`, `USER.md`,
      `MEMORY.md` (oder Legacy-Fallback `memory.md`, wenn `MEMORY.md` fehlt),
      `memory/YYYY-MM-DD.md`, optional `HEARTBEAT.md`.
    - **State-Dir (`~/.openclaw`)**: Konfiguration, Kanal-/Provider-Zustand, Auth-Profile, Sitzungen, Logs
      und gemeinsame Skills (`~/.openclaw/skills`).

    Standard-Workspace ist `~/.openclaw/workspace`, konfigurierbar über:

    ```json5
    {
      agents: { defaults: { workspace: "~/.openclaw/workspace" } },
    }
    ```

    Wenn der Bot nach einem Neustart „vergisst“, prüfen Sie, ob das Gateway bei jedem Start denselben
    Workspace verwendet (und denken Sie daran: Im Remote-Modus wird der Workspace des **Gateway-Hosts**
    verwendet, nicht der Ihres lokalen Laptops).

    Tipp: Wenn Sie ein dauerhaftes Verhalten oder eine Präferenz möchten, bitten Sie den Bot, es **in
    AGENTS.md oder MEMORY.md zu schreiben**, statt sich auf den Chat-Verlauf zu verlassen.

    Siehe [Agent workspace](/de/concepts/agent-workspace) und [Memory](/de/concepts/memory).

  </Accordion>

  <Accordion title="Empfohlene Backup-Strategie">
    Legen Sie Ihren **Agent-Workspace** in ein **privates** Git-Repo und sichern Sie ihn an einem
    privaten Ort (zum Beispiel GitHub privat). Damit erfassen Sie Memory + AGENTS/SOUL/USER-
    Dateien und können den „Geist“ des Assistenten später wiederherstellen.

    Committen Sie **nichts** unter `~/.openclaw` (Zugangsdaten, Sitzungen, Tokens oder verschlüsselte Secret-Payloads).
    Wenn Sie eine vollständige Wiederherstellung benötigen, sichern Sie sowohl den Workspace als auch das State-Verzeichnis
    separat (siehe die Migrationsfrage oben).

    Dokumentation: [Agent workspace](/de/concepts/agent-workspace).

  </Accordion>

  <Accordion title="Wie deinstalliere ich OpenClaw vollständig?">
    Siehe den eigenen Leitfaden: [Uninstall](/de/install/uninstall).
  </Accordion>

  <Accordion title="Können Agents außerhalb des Workspace arbeiten?">
    Ja. Der Workspace ist das **standardmäßige cwd** und der Memory-Anker, keine harte Sandbox.
    Relative Pfade werden innerhalb des Workspace aufgelöst, aber absolute Pfade können andere
    Host-Orte erreichen, sofern Sandboxing nicht aktiviert ist. Wenn Sie Isolation benötigen, verwenden Sie
    [`agents.defaults.sandbox`](/de/gateway/sandboxing) oder agent-spezifische Sandbox-Einstellungen. Wenn Sie
    ein Repo als Standard-Arbeitsverzeichnis möchten, zeigen Sie den `workspace` dieses Agenten
    auf die Repo-Wurzel. Das OpenClaw-Repo ist nur Quellcode; halten Sie den
    Workspace getrennt, sofern Sie nicht absichtlich möchten, dass der Agent darin arbeitet.

    Beispiel (Repo als Standard-cwd):

    ```json5
    {
      agents: {
        defaults: {
          workspace: "~/Projects/my-repo",
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Remote-Modus: Wo befindet sich der Sitzungs-Store?">
    Der Sitzungszustand gehört dem **Gateway-Host**. Wenn Sie im Remote-Modus arbeiten, befindet sich der relevante Sitzungs-Store auf der Remote-Maschine, nicht auf Ihrem lokalen Laptop. Siehe [Session management](/de/concepts/session).
  </Accordion>
</AccordionGroup>

## Konfigurationsgrundlagen

<AccordionGroup>
  <Accordion title="Welches Format hat die Konfiguration? Wo ist sie?">
    OpenClaw liest eine optionale **JSON5**-Konfiguration von `$OPENCLAW_CONFIG_PATH` (Standard: `~/.openclaw/openclaw.json`):

    ```
    $OPENCLAW_CONFIG_PATH
    ```

    Wenn die Datei fehlt, werden relativ sichere Standardwerte verwendet (einschließlich eines Standard-Workspace von `~/.openclaw/workspace`).

  </Accordion>

  <Accordion title='Ich habe `gateway.bind: "lan"` (oder `"tailnet"`) gesetzt und jetzt lauscht nichts / die UI sagt unauthorized'>
    Nicht-loopback-Binds **erfordern einen gültigen Gateway-Auth-Pfad**. In der Praxis bedeutet das:

    - Shared-Secret-Authentifizierung: Token oder Passwort
    - `gateway.auth.mode: "trusted-proxy"` hinter einem korrekt konfigurierten identitätsbewussten Reverse Proxy ohne loopback

    ```json5
    {
      gateway: {
        bind: "lan",
        auth: {
          mode: "token",
          token: "replace-me",
        },
      },
    }
    ```

    Hinweise:

    - `gateway.remote.token` / `.password` aktivieren lokale Gateway-Authentifizierung **nicht** von selbst.
    - Lokale Aufrufpfade können `gateway.remote.*` nur als Fallback verwenden, wenn `gateway.auth.*` nicht gesetzt ist.
    - Für Passwortauthentifizierung setzen Sie stattdessen `gateway.auth.mode: "password"` plus `gateway.auth.password` (oder `OPENCLAW_GATEWAY_PASSWORD`).
    - Wenn `gateway.auth.token` / `gateway.auth.password` explizit über SecretRef konfiguriert und nicht aufgelöst werden kann, schlägt die Auflösung fail-closed fehl (kein Remote-Fallback, das dies maskiert).
    - Shared-Secret-Control-UI-Setups authentifizieren sich über `connect.params.auth.token` oder `connect.params.auth.password` (gespeichert in App-/UI-Einstellungen). Identitätsbasierte Modi wie Tailscale Serve oder `trusted-proxy` verwenden stattdessen Request-Header. Vermeiden Sie Shared Secrets in URLs.
    - Mit `gateway.auth.mode: "trusted-proxy"` erfüllen Reverse Proxys auf demselben Host über loopback die Trusted-Proxy-Authentifizierung weiterhin **nicht**. Der vertrauenswürdige Proxy muss eine konfigurierte Quelle ohne loopback sein.

  </Accordion>

  <Accordion title="Warum brauche ich jetzt auf localhost ein Token?">
    OpenClaw erzwingt standardmäßig Gateway-Authentifizierung, auch für loopback. Im normalen Standardpfad bedeutet das Token-Authentifizierung: Wenn kein expliziter Auth-Pfad konfiguriert ist, löst sich der Gateway-Start in den Token-Modus auf und erzeugt automatisch eines, das unter `gateway.auth.token` gespeichert wird. Daher **müssen lokale WS-Clients sich authentifizieren**. Das blockiert andere lokale Prozesse beim Aufruf des Gateways.

    Wenn Sie einen anderen Auth-Pfad bevorzugen, können Sie explizit den Passwortmodus wählen (oder für identitätsbewusste Reverse Proxys ohne loopback `trusted-proxy`). Wenn Sie **wirklich** offenen loopback möchten, setzen Sie `gateway.auth.mode: "none"` explizit in Ihrer Konfiguration. Doctor kann jederzeit ein Token für Sie erzeugen: `openclaw doctor --generate-gateway-token`.

  </Accordion>

  <Accordion title="Muss ich nach Konfigurationsänderungen neu starten?">
    Das Gateway überwacht die Konfiguration und unterstützt Hot-Reload:

    - `gateway.reload.mode: "hybrid"` (Standard): sichere Änderungen hot anwenden, für kritische neu starten
    - `hot`, `restart`, `off` werden ebenfalls unterstützt

  </Accordion>

  <Accordion title="Wie deaktiviere ich lustige CLI-Taglines?">
    Setzen Sie `cli.banner.taglineMode` in der Konfiguration:

    ```json5
    {
      cli: {
        banner: {
          taglineMode: "off", // random | default | off
        },
      },
    }
    ```

    - `off`: blendet den Tagline-Text aus, behält aber Banner-Titel-/Versionszeile bei.
    - `default`: verwendet jedes Mal `All your chats, one OpenClaw.`.
    - `random`: rotierende lustige/saisonale Taglines (Standardverhalten).
    - Wenn Sie überhaupt kein Banner möchten, setzen Sie die Umgebungsvariable `OPENCLAW_HIDE_BANNER=1`.

  </Accordion>

  <Accordion title="Wie aktiviere ich Websuche (und Web-Abruf)?">
    `web_fetch` funktioniert ohne API-Schlüssel. `web_search` hängt von Ihrem gewählten
    Provider ab:

    - API-gestützte Provider wie Brave, Exa, Firecrawl, Gemini, Grok, Kimi, MiniMax Search, Perplexity und Tavily erfordern ihre normale API-Schlüssel-Einrichtung.
    - Ollama Web Search benötigt keinen Schlüssel, verwendet aber Ihren konfigurierten Ollama-Host und erfordert `ollama signin`.
    - DuckDuckGo benötigt keinen Schlüssel, ist aber eine inoffizielle HTML-basierte Integration.
    - SearXNG ist schlüsselfrei/self-hosted; konfigurieren Sie `SEARXNG_BASE_URL` oder `plugins.entries.searxng.config.webSearch.baseUrl`.

    **Empfohlen:** Führen Sie `openclaw configure --section web` aus und wählen Sie einen Provider.
    Alternativen per Umgebungsvariable:

    - Brave: `BRAVE_API_KEY`
    - Exa: `EXA_API_KEY`
    - Firecrawl: `FIRECRAWL_API_KEY`
    - Gemini: `GEMINI_API_KEY`
    - Grok: `XAI_API_KEY`
    - Kimi: `KIMI_API_KEY` oder `MOONSHOT_API_KEY`
    - MiniMax Search: `MINIMAX_CODE_PLAN_KEY`, `MINIMAX_CODING_API_KEY` oder `MINIMAX_API_KEY`
    - Perplexity: `PERPLEXITY_API_KEY` oder `OPENROUTER_API_KEY`
    - SearXNG: `SEARXNG_BASE_URL`
    - Tavily: `TAVILY_API_KEY`

    ```json5
    {
      plugins: {
        entries: {
          brave: {
            config: {
              webSearch: {
                apiKey: "BRAVE_API_KEY_HERE",
              },
            },
          },
        },
        },
        tools: {
          web: {
            search: {
              enabled: true,
              provider: "brave",
              maxResults: 5,
            },
            fetch: {
              enabled: true,
              provider: "firecrawl", // optional; omit for auto-detect
            },
          },
        },
    }
    ```

    Provider-spezifische Web-Suchkonfiguration befindet sich jetzt unter `plugins.entries.<plugin>.config.webSearch.*`.
    Legacy-Provider-Pfade unter `tools.web.search.*` werden vorübergehend aus Kompatibilitätsgründen noch geladen, sollten aber für neue Konfigurationen nicht verwendet werden.
    Firecrawl-Web-Fetch-Fallback-Konfiguration befindet sich unter `plugins.entries.firecrawl.config.webFetch.*`.

    Hinweise:

    - Wenn Sie Allowlists verwenden, fügen Sie `web_search`/`web_fetch`/`x_search` oder `group:web` hinzu.
    - `web_fetch` ist standardmäßig aktiviert (sofern nicht explizit deaktiviert).
    - Wenn `tools.web.fetch.provider` weggelassen wird, erkennt OpenClaw den ersten bereiten Fetch-Fallback-Provider automatisch aus den verfügbaren Zugangsdaten. Derzeit ist der gebündelte Provider Firecrawl.
    - Daemons lesen Umgebungsvariablen aus `~/.openclaw/.env` (oder aus der Dienstumgebung).

    Dokumentation: [Web tools](/de/tools/web).

  </Accordion>

  <Accordion title="config.apply hat meine Konfiguration gelöscht. Wie stelle ich sie wieder her und vermeide das?">
    `config.apply` ersetzt die **gesamte Konfiguration**. Wenn Sie ein Teilobjekt senden, wird alles
    andere entfernt.

    Wiederherstellung:

    - Stellen Sie aus einem Backup wieder her (git oder eine kopierte `~/.openclaw/openclaw.json`).
    - Wenn Sie kein Backup haben, führen Sie `openclaw doctor` erneut aus und konfigurieren Sie Kanäle/Modelle neu.
    - Wenn dies unerwartet war, reichen Sie einen Bug ein und fügen Sie Ihre zuletzt bekannte Konfiguration oder ein Backup bei.
    - Ein lokaler Coding-Agent kann eine funktionierende Konfiguration oft aus Logs oder dem Verlauf rekonstruieren.

    Vermeidung:

    - Verwenden Sie `openclaw config set` für kleine Änderungen.
    - Verwenden Sie `openclaw configure` für interaktive Bearbeitung.
    - Verwenden Sie zuerst `config.schema.lookup`, wenn Sie sich bei einem genauen Pfad oder der Feldform nicht sicher sind; es gibt einen flachen Schema-Knoten plus Zusammenfassungen der unmittelbaren Kinder zum weiteren Drill-down zurück.
    - Verwenden Sie `config.patch` für partielle RPC-Bearbeitungen; reservieren Sie `config.apply` für vollständiges Ersetzen der Konfiguration.
    - Wenn Sie das nur für den Eigentümer verfügbare `gateway`-Tool aus einem Agent-Lauf verwenden, lehnt es weiterhin Schreibzugriffe auf `tools.exec.ask` / `tools.exec.security` ab (einschließlich Legacy-Aliasse `tools.bash.*`, die auf dieselben geschützten Exec-Pfade normalisiert werden).

    Dokumentation: [Config](/cli/config), [Configure](/cli/configure), [Doctor](/de/gateway/doctor).

  </Accordion>

  <Accordion title="Wie betreibe ich ein zentrales Gateway mit spezialisierten Workern auf mehreren Geräten?">
    Das übliche Muster ist **ein Gateway** (z. B. Raspberry Pi) plus **Nodes** und **Agents**:

    - **Gateway (zentral):** besitzt Kanäle (Signal/WhatsApp), Routing und Sitzungen.
    - **Nodes (Geräte):** Macs/iOS/Android verbinden sich als Peripheriegeräte und stellen lokale Tools (`system.run`, `canvas`, `camera`) bereit.
    - **Agents (Worker):** separate Gehirne/Workspaces für spezielle Rollen (z. B. „Hetzner ops“, „Personal data“).
    - **Sub-agents:** starten Hintergrundarbeit aus einem Hauptagenten heraus, wenn Sie Parallelität möchten.
    - **TUI:** Verbindung zum Gateway herstellen und Agents/Sitzungen wechseln.

    Dokumentation: [Nodes](/de/nodes), [Remote access](/de/gateway/remote), [Multi-Agent Routing](/de/concepts/multi-agent), [Sub-agents](/de/tools/subagents), [TUI](/web/tui).

  </Accordion>

  <Accordion title="Kann der OpenClaw-Browser headless laufen?">
    Ja. Es ist eine Konfigurationsoption:

    ```json5
    {
      browser: { headless: true },
      agents: {
        defaults: {
          sandbox: { browser: { headless: true } },
        },
      },
    }
    ```

    Standard ist `false` (mit sichtbarem Browser). Headless löst auf manchen Websites eher Anti-Bot-Prüfungen aus. Siehe [Browser](/de/tools/browser).

    Headless verwendet **dieselbe Chromium-Engine** und funktioniert für die meisten Automatisierungen (Formulare, Klicks, Scraping, Logins). Die Hauptunterschiede:

    - Kein sichtbares Browserfenster (verwenden Sie Screenshots, wenn Sie etwas visuell brauchen).
    - Einige Seiten sind im Headless-Modus strenger bei Automatisierung (CAPTCHAs, Anti-Bot).
      Zum Beispiel blockiert X/Twitter Headless-Sitzungen häufig.

  </Accordion>

  <Accordion title="Wie verwende ich Brave für Browser-Steuerung?">
    Setzen Sie `browser.executablePath` auf Ihre Brave-Binärdatei (oder einen anderen Chromium-basierten Browser) und starten Sie das Gateway neu.
    Die vollständigen Konfigurationsbeispiele finden Sie unter [Browser](/de/tools/browser#use-brave-or-another-chromium-based-browser).
  </Accordion>
</AccordionGroup>

## Remote-Gateways und Nodes

<AccordionGroup>
  <Accordion title="Wie propagieren Befehle zwischen Telegram, dem Gateway und Nodes?">
    Telegram-Nachrichten werden vom **Gateway** verarbeitet. Das Gateway führt den Agenten aus und
    ruft erst dann Nodes über den **Gateway WebSocket** auf, wenn ein Node-Tool benötigt wird:

    Telegram → Gateway → Agent → `node.*` → Node → Gateway → Telegram

    Nodes sehen keinen eingehenden Provider-Verkehr; sie empfangen nur Node-RPC-Aufrufe.

  </Accordion>

  <Accordion title="Wie kann mein Agent auf meinen Computer zugreifen, wenn das Gateway remote gehostet wird?">
    Kurzantwort: **Koppeln Sie Ihren Computer als Node**. Das Gateway läuft anderswo, kann aber
    `node.*`-Tools (Bildschirm, Kamera, System) auf Ihrer lokalen Maschine über den Gateway-WebSocket aufrufen.

    Typisches Setup:

    1. Führen Sie das Gateway auf dem Always-on-Host aus (VPS/Home-Server).
    2. Bringen Sie den Gateway-Host + Ihren Computer ins gleiche Tailnet.
    3. Stellen Sie sicher, dass der Gateway-WS erreichbar ist (Tailnet-Bind oder SSH-Tunnel).
    4. Öffnen Sie die macOS-App lokal und verbinden Sie sich im Modus **Remote over SSH** (oder direkt per Tailnet),
       damit sie sich als Node registrieren kann.
    5. Genehmigen Sie den Node auf dem Gateway:

       ```bash
       openclaw devices list
       openclaw devices approve <requestId>
       ```

    Es ist keine separate TCP-Bridge erforderlich; Nodes verbinden sich über den Gateway WebSocket.

    Sicherheitshinweis: Das Koppeln eines macOS-Nodes erlaubt `system.run` auf dieser Maschine. Koppeln
    Sie nur Geräte, denen Sie vertrauen, und lesen Sie [Sicherheit](/de/gateway/security).

    Dokumentation: [Nodes](/de/nodes), [Gateway protocol](/de/gateway/protocol), [macOS remote mode](/de/platforms/mac/remote), [Sicherheit](/de/gateway/security).

  </Accordion>

  <Accordion title="Tailscale ist verbunden, aber ich bekomme keine Antworten. Was nun?">
    Prüfen Sie die Grundlagen:

    - Gateway läuft: `openclaw gateway status`
    - Gateway-Gesundheit: `openclaw status`
    - Kanal-Gesundheit: `openclaw channels status`

    Verifizieren Sie dann Authentifizierung und Routing:

    - Wenn Sie Tailscale Serve verwenden, stellen Sie sicher, dass `gateway.auth.allowTailscale` korrekt gesetzt ist.
    - Wenn Sie sich über SSH-Tunnel verbinden, prüfen Sie, ob der lokale Tunnel aktiv ist und auf den richtigen Port zeigt.
    - Bestätigen Sie, dass Ihre Allowlists (DM oder Gruppe) Ihr Konto enthalten.

    Dokumentation: [Tailscale](/de/gateway/tailscale), [Remote access](/de/gateway/remote), [Kanäle](/de/channels).

  </Accordion>

  <Accordion title="Können zwei OpenClaw-Instanzen miteinander sprechen (lokal + VPS)?">
    Ja. Es gibt keine eingebaute „Bot-zu-Bot“-Bridge, aber Sie können sie auf einige
    zuverlässige Arten verdrahten:

    **Am einfachsten:** Verwenden Sie einen normalen Chat-Kanal, auf den beide Bots zugreifen können (Telegram/Slack/WhatsApp).
    Lassen Sie Bot A eine Nachricht an Bot B senden und Bot B dann wie gewohnt antworten.

    **CLI-Bridge (generisch):** Führen Sie ein Skript aus, das das andere Gateway mit
    `openclaw agent --message ... --deliver` aufruft und auf einen Chat zielt, auf dem der andere Bot
    lauscht. Wenn ein Bot auf einem Remote-VPS läuft, richten Sie Ihre CLI auf dieses Remote-Gateway
    über SSH/Tailscale aus (siehe [Remote access](/de/gateway/remote)).

    Beispielmuster (auf einer Maschine ausführen, die das Ziel-Gateway erreichen kann):

    ```bash
    openclaw agent --message "Hello from local bot" --deliver --channel telegram --reply-to <chat-id>
    ```

    Tipp: Fügen Sie eine Leitplanke hinzu, damit die beiden Bots nicht endlos schleifen (nur Erwähnungen,
    Kanal-Allowlists oder eine Regel „nicht auf Bot-Nachrichten antworten“).

    Dokumentation: [Remote access](/de/gateway/remote), [Agent CLI](/cli/agent), [Agent send](/de/tools/agent-send).

  </Accordion>

  <Accordion title="Brauche ich separate VPSes für mehrere Agents?">
    Nein. Ein Gateway kann mehrere Agents hosten, jeweils mit eigenem Workspace, Modell-Standardwerten
    und Routing. Das ist die normale Einrichtung und viel günstiger und einfacher, als
    pro Agent einen eigenen VPS zu betreiben.

    Verwenden Sie separate VPSes nur, wenn Sie harte Isolation (Sicherheitsgrenzen) oder sehr
    unterschiedliche Konfigurationen benötigen, die Sie nicht teilen möchten. Andernfalls behalten Sie ein Gateway und
    verwenden mehrere Agents oder Sub-agents.

  </Accordion>

  <Accordion title="Gibt es einen Vorteil, einen Node auf meinem persönlichen Laptop statt SSH von einem VPS zu verwenden?">
    Ja – Nodes sind die erstklassige Methode, Ihren Laptop von einem Remote-Gateway aus zu erreichen, und sie
    ermöglichen mehr als Shell-Zugriff. Das Gateway läuft auf macOS/Linux (Windows über WSL2) und ist
    leichtgewichtig (ein kleiner VPS oder eine Raspberry-Pi-Klasse-Box reicht; 4 GB RAM sind mehr als genug), daher ist ein häufiges
    Setup ein Always-on-Host plus Ihr Laptop als Node.

    - **Kein eingehendes SSH erforderlich.** Nodes verbinden sich zum Gateway WebSocket und verwenden Geräte-Pairing.
    - **Sicherere Ausführungskontrollen.** `system.run` wird auf diesem Laptop durch Node-Allowlists/Genehmigungen begrenzt.
    - **Mehr Gerätetools.** Nodes stellen zusätzlich zu `system.run` auch `canvas`, `camera` und `screen` bereit.
    - **Lokale Browser-Automatisierung.** Lassen Sie das Gateway auf einem VPS, aber führen Sie Chrome lokal über einen Node-Host auf dem Laptop aus oder hängen Sie sich über Chrome MCP an lokales Chrome auf dem Host.

    SSH ist für ad-hoc-Shell-Zugriff in Ordnung, aber Nodes sind für fortlaufende Agent-Workflows und
    Geräteautomatisierung einfacher.

    Dokumentation: [Nodes](/de/nodes), [Nodes CLI](/cli/nodes), [Browser](/de/tools/browser).

  </Accordion>

  <Accordion title="Führen Nodes einen Gateway-Dienst aus?">
    Nein. Es sollte nur **ein Gateway** pro Host laufen, es sei denn, Sie betreiben absichtlich isolierte Profile (siehe [Multiple gateways](/de/gateway/multiple-gateways)). Nodes sind Peripheriegeräte, die sich
    mit dem Gateway verbinden (iOS-/Android-Nodes oder macOS-„Node-Modus“ in der Menüleisten-App). Für headless Node-
    Hosts und CLI-Steuerung siehe [Node host CLI](/cli/node).

    Ein vollständiger Neustart ist für Änderungen an `gateway`, `discovery` und `canvasHost` erforderlich.

  </Accordion>

  <Accordion title="Gibt es einen API-/RPC-Weg, Konfiguration anzuwenden?">
    Ja.

    - `config.schema.lookup`: inspiziert einen Konfigurations-Teilbaum mit seinem flachen Schema-Knoten, passendem UI-Hinweis und Zusammenfassungen der unmittelbaren Kinder vor dem Schreiben
    - `config.get`: holt den aktuellen Schnappschuss + Hash
    - `config.patch`: sicheres partielles Update (für die meisten RPC-Bearbeitungen bevorzugt); lädt neu, wenn möglich, und startet neu, wenn nötig
    - `config.apply`: validiert + ersetzt die gesamte Konfiguration; lädt neu, wenn möglich, und startet neu, wenn nötig
    - Das nur für den Eigentümer verfügbare Laufzeit-Tool `gateway` verweigert weiterhin Umschreibungen von `tools.exec.ask` / `tools.exec.security`; Legacy-Aliasse `tools.bash.*` werden auf dieselben geschützten Exec-Pfade normalisiert

  </Accordion>

  <Accordion title="Minimal sinnvolle Konfiguration für eine Erstinstallation">
    ```json5
    {
      agents: { defaults: { workspace: "~/.openclaw/workspace" } },
      channels: { whatsapp: { allowFrom: ["+15555550123"] } },
    }
    ```

    Damit setzen Sie Ihren Workspace und beschränken, wer den Bot auslösen kann.

  </Accordion>

  <Accordion title="Wie richte ich Tailscale auf einem VPS ein und verbinde mich von meinem Mac aus?">
    Minimale Schritte:

    1. **Auf dem VPS installieren + anmelden**

       ```bash
       curl -fsSL https://tailscale.com/install.sh | sh
       sudo tailscale up
       ```

    2. **Auf Ihrem Mac installieren + anmelden**
       - Verwenden Sie die Tailscale-App und melden Sie sich im selben Tailnet an.
    3. **MagicDNS aktivieren (empfohlen)**
       - Aktivieren Sie in der Tailscale-Admin-Konsole MagicDNS, damit der VPS einen stabilen Namen erhält.
    4. **Den Tailnet-Hostnamen verwenden**
       - SSH: `ssh user@your-vps.tailnet-xxxx.ts.net`
       - Gateway WS: `ws://your-vps.tailnet-xxxx.ts.net:18789`

    Wenn Sie die Control UI ohne SSH möchten, verwenden Sie Tailscale Serve auf dem VPS:

    ```bash
    openclaw gateway --tailscale serve
    ```

    Dadurch bleibt das Gateway an loopback gebunden und wird über Tailscale per HTTPS bereitgestellt. Siehe [Tailscale](/de/gateway/tailscale).

  </Accordion>

  <Accordion title="Wie verbinde ich einen Mac-Node mit einem Remote-Gateway (Tailscale Serve)?">
    Serve stellt die **Gateway Control UI + WS** bereit. Nodes verbinden sich über denselben Gateway-WS-Endpunkt.

    Empfohlene Einrichtung:

    1. **Stellen Sie sicher, dass VPS + Mac im selben Tailnet sind**.
    2. **Verwenden Sie die macOS-App im Remote-Modus** (das SSH-Ziel kann der Tailnet-Hostname sein).
       Die App tunnelt den Gateway-Port und verbindet sich als Node.
    3. **Genehmigen Sie den Node** auf dem Gateway:

       ```bash
       openclaw devices list
       openclaw devices approve <requestId>
       ```

    Dokumentation: [Gateway protocol](/de/gateway/protocol), [Discovery](/de/gateway/discovery), [macOS remote mode](/de/platforms/mac/remote).

  </Accordion>

  <Accordion title="Soll ich auf einem zweiten Laptop installieren oder nur einen Node hinzufügen?">
    Wenn Sie auf dem zweiten Laptop nur **lokale Tools** (Bildschirm/Kamera/Exec) benötigen, fügen Sie ihn als
    **Node** hinzu. So bleibt es bei einem einzigen Gateway und Sie vermeiden doppelte Konfiguration. Lokale Node-Tools sind
    derzeit nur für macOS verfügbar, wir planen aber, sie auf andere Betriebssysteme auszuweiten.

    Installieren Sie ein zweites Gateway nur, wenn Sie **harte Isolation** oder zwei vollständig getrennte Bots brauchen.

    Dokumentation: [Nodes](/de/nodes), [Nodes CLI](/cli/nodes), [Multiple gateways](/de/gateway/multiple-gateways).

  </Accordion>
</AccordionGroup>

## Umgebungsvariablen und `.env`-Laden

<AccordionGroup>
  <Accordion title="Wie lädt OpenClaw Umgebungsvariablen?">
    OpenClaw liest Umgebungsvariablen aus dem Elternprozess (Shell, launchd/systemd, CI usw.) und lädt zusätzlich:

    - `.env` aus dem aktuellen Arbeitsverzeichnis
    - ein globales Fallback-`.env` aus `~/.openclaw/.env` (alias `$OPENCLAW_STATE_DIR/.env`)

    Keines der `.env`-Dateien überschreibt bestehende Umgebungsvariablen.

    Sie können auch Inline-Umgebungsvariablen in der Konfiguration definieren (werden nur angewendet, wenn sie im Prozess-Env fehlen):

    ```json5
    {
      env: {
        OPENROUTER_API_KEY: "sk-or-...",
        vars: { GROQ_API_KEY: "gsk-..." },
      },
    }
    ```

    Siehe [/environment](/de/help/environment) für vollständige Prioritäten und Quellen.

  </Accordion>

  <Accordion title="Ich habe das Gateway über den Dienst gestartet und meine Umgebungsvariablen sind verschwunden. Was nun?">
    Zwei häufige Lösungen:

    1. Legen Sie die fehlenden Schlüssel in `~/.openclaw/.env` ab, damit sie erfasst werden, auch wenn der Dienst Ihr Shell-Env nicht übernimmt.
    2. Aktivieren Sie Shell-Import (Opt-in-Komfortfunktion):

    ```json5
    {
      env: {
        shellEnv: {
          enabled: true,
          timeoutMs: 15000,
        },
      },
    }
    ```

    Dadurch wird Ihre Login-Shell ausgeführt und es werden nur fehlende erwartete Schlüssel importiert (niemals überschrieben). Entsprechende Umgebungsvariablen:
    `OPENCLAW_LOAD_SHELL_ENV=1`, `OPENCLAW_SHELL_ENV_TIMEOUT_MS=15000`.

  </Accordion>

  <Accordion title='Ich habe `COPILOT_GITHUB_TOKEN` gesetzt, aber models status zeigt "Shell env: off." Warum?'>
    `openclaw models status` meldet, ob **Shell-Env-Import** aktiviert ist. „Shell env: off“
    bedeutet **nicht**, dass Ihre Umgebungsvariablen fehlen – es bedeutet nur, dass OpenClaw Ihre
    Login-Shell nicht automatisch lädt.

    Wenn das Gateway als Dienst läuft (launchd/systemd), übernimmt es Ihre Shell-
    Umgebung nicht. Beheben Sie das auf eine dieser Arten:

    1. Legen Sie das Token in `~/.openclaw/.env` ab:

       ```
       COPILOT_GITHUB_TOKEN=...
       ```

    2. Oder aktivieren Sie den Shell-Import (`env.shellEnv.enabled: true`).
    3. Oder fügen Sie es Ihrem Konfigurations-Block `env` hinzu (wird nur angewendet, wenn es fehlt).

    Starten Sie dann das Gateway neu und prüfen Sie erneut:

    ```bash
    openclaw models status
    ```

    Copilot-Tokens werden aus `COPILOT_GITHUB_TOKEN` gelesen (auch `GH_TOKEN` / `GITHUB_TOKEN`).
    Siehe [/concepts/model-providers](/de/concepts/model-providers) und [/environment](/de/help/environment).

  </Accordion>
</AccordionGroup>

## Sitzungen und mehrere Chats

<AccordionGroup>
  <Accordion title="Wie starte ich eine neue Unterhaltung?">
    Senden Sie `/new` oder `/reset` als eigenständige Nachricht. Siehe [Session management](/de/concepts/session).
  </Accordion>

  <Accordion title="Werden Sitzungen automatisch zurückgesetzt, wenn ich nie /new sende?">
    Sitzungen können nach `session.idleMinutes` ablaufen, aber dies ist **standardmäßig deaktiviert** (Standard **0**).
    Setzen Sie einen positiven Wert, um das Idle-Ablaufen zu aktivieren. Wenn aktiviert, startet die **nächste**
    Nachricht nach der Leerlaufzeit eine neue Sitzungs-ID für diesen Chat-Schlüssel.
    Dadurch werden Transkripte nicht gelöscht – es beginnt nur eine neue Sitzung.

    ```json5
    {
      session: {
        idleMinutes: 240,
      },
    }
    ```

  </Accordion>

  <Accordion title="Gibt es eine Möglichkeit, ein Team aus OpenClaw-Instanzen zu erstellen (ein CEO und viele Agents)?">
    Ja, über **Multi-Agent-Routing** und **Sub-agents**. Sie können einen Koordinator-
    Agenten und mehrere Worker-Agents mit eigenen Workspaces und Modellen erstellen.

    Allerdings sollte dies eher als **unterhaltsames Experiment** betrachtet werden. Es ist tokenintensiv und oft
    weniger effizient als ein Bot mit getrennten Sitzungen. Das typische Modell, das wir
    uns vorstellen, ist ein Bot, mit dem Sie sprechen, mit unterschiedlichen Sitzungen für parallele Arbeit. Dieser
    Bot kann bei Bedarf auch Sub-agents starten.

    Dokumentation: [Multi-agent routing](/de/concepts/multi-agent), [Sub-agents](/de/tools/subagents), [Agents CLI](/cli/agents).

  </Accordion>

  <Accordion title="Warum wurde der Kontext mitten in einer Aufgabe abgeschnitten? Wie verhindere ich das?">
    Der Sitzungskontext ist durch das Modellfenster begrenzt. Lange Chats, große Tool-Ausgaben oder viele
    Dateien können Kompaktierung oder Abschneidung auslösen.

    Was hilft:

    - Bitten Sie den Bot, den aktuellen Zustand zusammenzufassen und in eine Datei zu schreiben.
    - Verwenden Sie `/compact` vor langen Aufgaben und `/new`, wenn Sie das Thema wechseln.
    - Halten Sie wichtigen Kontext im Workspace und bitten Sie den Bot, ihn erneut zu lesen.
    - Verwenden Sie Sub-agents für lange oder parallele Arbeit, damit der Hauptchat kleiner bleibt.
    - Wählen Sie ein Modell mit einem größeren Kontextfenster, wenn dies häufig passiert.

  </Accordion>

  <Accordion title="Wie setze ich OpenClaw vollständig zurück, aber behalte es installiert?">
    Verwenden Sie den Reset-Befehl:

    ```bash
    openclaw reset
    ```

    Nicht-interaktiver vollständiger Reset:

    ```bash
    openclaw reset --scope full --yes --non-interactive
    ```

    Führen Sie dann die Einrichtung erneut aus:

    ```bash
    openclaw onboard --install-daemon
    ```

    Hinweise:

    - Das Onboarding bietet auch **Reset** an, wenn es eine bestehende Konfiguration erkennt. Siehe [Onboarding (CLI)](/de/start/wizard).
    - Wenn Sie Profile verwendet haben (`--profile` / `OPENCLAW_PROFILE`), setzen Sie jedes State-Dir zurück (Standard ist `~/.openclaw-<profile>`).
    - Dev-Reset: `openclaw gateway --dev --reset` (nur Entwicklung; löscht Dev-Konfiguration + Zugangsdaten + Sitzungen + Workspace).

  </Accordion>

  <Accordion title='Ich erhalte Fehler wie "context too large" - wie setze ich zurück oder kompaktiere?'>
    Verwenden Sie eine dieser Möglichkeiten:

    - **Compact** (behält die Unterhaltung, fasst aber ältere Turns zusammen):

      ```
      /compact
      ```

      oder `/compact <instructions>`, um die Zusammenfassung zu steuern.

    - **Reset** (frische Sitzungs-ID für denselben Chat-Schlüssel):

      ```
      /new
      /reset
      ```

    Wenn das weiterhin passiert:

    - Aktivieren oder optimieren Sie **Session pruning** (`agents.defaults.contextPruning`), um alte Tool-Ausgaben zu kürzen.
    - Verwenden Sie ein Modell mit einem größeren Kontextfenster.

    Dokumentation: [Compaction](/de/concepts/compaction), [Session pruning](/de/concepts/session-pruning), [Session management](/de/concepts/session).

  </Accordion>

  <Accordion title='Warum sehe ich "LLM request rejected: messages.content.tool_use.input field required"?'>
    Das ist ein Provider-Validierungsfehler: Das Modell hat einen `tool_use`-Block ohne das erforderliche
    `input` ausgegeben. Das bedeutet normalerweise, dass der Sitzungsverlauf veraltet oder beschädigt ist (oft nach langen Threads
    oder einer Tool-/Schema-Änderung).

    Lösung: Starten Sie mit `/new` (eigenständige Nachricht) eine neue Sitzung.

  </Accordion>

  <Accordion title="Warum bekomme ich alle 30 Minuten Heartbeat-Nachrichten?">
    Heartbeats laufen standardmäßig alle **30m** (**1h** bei OAuth-Authentifizierung). Passen Sie sie an oder deaktivieren Sie sie:

    ```json5
    {
      agents: {
        defaults: {
          heartbeat: {
            every: "2h", // or "0m" to disable
          },
        },
      },
    }
    ```

    Wenn `HEARTBEAT.md` existiert, aber faktisch leer ist (nur Leerzeilen und Markdown-
    Überschriften wie `# Heading`), überspringt OpenClaw den Heartbeat-Lauf, um API-Aufrufe zu sparen.
    Wenn die Datei fehlt, läuft der Heartbeat trotzdem und das Modell entscheidet, was zu tun ist.

    Pro-Agent-Overrides verwenden `agents.list[].heartbeat`. Dokumentation: [Heartbeat](/de/gateway/heartbeat).

  </Accordion>

  <Accordion title='Muss ich einem WhatsApp-Gruppenchat ein "Bot-Konto" hinzufügen?'>
    Nein. OpenClaw läuft auf **Ihrem eigenen Konto**, also kann OpenClaw die Gruppe sehen, wenn Sie darin sind.
    Standardmäßig sind Gruppenantworten blockiert, bis Sie Absender erlauben (`groupPolicy: "allowlist"`).

    Wenn Sie möchten, dass nur **Sie** Gruppenantworten auslösen können:

    ```json5
    {
      channels: {
        whatsapp: {
          groupPolicy: "allowlist",
          groupAllowFrom: ["+15551234567"],
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Wie erhalte ich die JID einer WhatsApp-Gruppe?">
    Option 1 (am schnellsten): Logs verfolgen und eine Testnachricht in die Gruppe senden:

    ```bash
    openclaw logs --follow --json
    ```

    Suchen Sie nach `chatId` (oder `from`) mit Endung `@g.us`, zum Beispiel:
    `1234567890-1234567890@g.us`.

    Option 2 (wenn bereits konfiguriert/allowlistet): Gruppen aus der Konfiguration auflisten:

    ```bash
    openclaw directory groups list --channel whatsapp
    ```

    Dokumentation: [WhatsApp](/de/channels/whatsapp), [Directory](/cli/directory), [Logs](/cli/logs).

  </Accordion>

  <Accordion title="Warum antwortet OpenClaw in einer Gruppe nicht?">
    Zwei häufige Ursachen:

    - Mention-Gating ist aktiv (Standard). Sie müssen den Bot mit @ erwähnen (oder `mentionPatterns` treffen).
    - Sie haben `channels.whatsapp.groups` ohne `"*"` konfiguriert und die Gruppe steht nicht auf der Allowlist.

    Siehe [Groups](/de/channels/groups) und [Group messages](/de/channels/group-messages).

  </Accordion>

  <Accordion title="Teilen Gruppen/Threads den Kontext mit DMs?">
    Direkte Chats kollabieren standardmäßig in die Hauptsitzung. Gruppen/Kanäle haben eigene Sitzungsschlüssel, und Telegram-Themen / Discord-Threads sind separate Sitzungen. Siehe [Groups](/de/channels/groups) und [Group messages](/de/channels/group-messages).
  </Accordion>

  <Accordion title="Wie viele Workspaces und Agents kann ich erstellen?">
    Keine festen Grenzen. Dutzende (sogar Hunderte) sind in Ordnung, aber achten Sie auf:

    - **Wachsender Speicherplatz:** Sitzungen + Transkripte liegen unter `~/.openclaw/agents/<agentId>/sessions/`.
    - **Token-Kosten:** Mehr Agents bedeuten mehr gleichzeitige Modellnutzung.
    - **Betriebsaufwand:** Auth-Profile, Workspaces und Kanalrouting pro Agent.

    Tipps:

    - Halten Sie pro Agent einen **aktiven** Workspace (`agents.defaults.workspace`).
    - Räumen Sie alte Sitzungen auf (JSONL oder Store-Einträge löschen), wenn der Speicherverbrauch wächst.
    - Verwenden Sie `openclaw doctor`, um verwaiste Workspaces und Profilabweichungen zu erkennen.

  </Accordion>

  <Accordion title="Kann ich mehrere Bots oder Chats gleichzeitig betreiben (Slack), und wie richte ich das ein?">
    Ja. Verwenden Sie **Multi-Agent Routing**, um mehrere isolierte Agents auszuführen und eingehende Nachrichten nach
    Kanal/Konto/Peer zu routen. Slack wird als Kanal unterstützt und kann an bestimmte Agents gebunden werden.

    Browser-Zugriff ist leistungsfähig, aber nicht „alles, was ein Mensch kann“ – Anti-Bot-Maßnahmen, CAPTCHAs und MFA können
    Automatisierung weiterhin blockieren. Für die zuverlässigste Browser-Steuerung verwenden Sie lokales Chrome MCP auf dem Host,
    oder nutzen Sie CDP auf der Maschine, die den Browser tatsächlich ausführt.

    Best-Practice-Setup:

    - Always-on-Gateway-Host (VPS/Mac mini).
    - Ein Agent pro Rolle (Bindings).
    - Slack-Kanal/Kanäle an diese Agents gebunden.
    - Lokaler Browser über Chrome MCP oder bei Bedarf einen Node.

    Dokumentation: [Multi-Agent Routing](/de/concepts/multi-agent), [Slack](/de/channels/slack),
    [Browser](/de/tools/browser), [Nodes](/de/nodes).

  </Accordion>
</AccordionGroup>

## Modelle: Standardwerte, Auswahl, Aliasse, Wechsel

<AccordionGroup>
  <Accordion title='Was ist das "Standardmodell"?'>
    Das Standardmodell von OpenClaw ist das, was Sie festlegen unter:

    ```
    agents.defaults.model.primary
    ```

    Modelle werden als `provider/model` referenziert (Beispiel: `openai/gpt-5.4`). Wenn Sie den Provider weglassen, versucht OpenClaw zuerst einen Alias, dann eine eindeutige configured-provider-Zuordnung für diese exakte Modell-ID und fällt erst danach auf den konfigurierten Standard-Provider als veralteten Kompatibilitätspfad zurück. Wenn dieser Provider das konfigurierte Standardmodell nicht mehr anbietet, fällt OpenClaw auf das erste konfigurierte Provider-/Modellpaar zurück, anstatt einen veralteten entfernten Provider-Standard anzuzeigen. Sie sollten weiterhin **explizit** `provider/model` setzen.

  </Accordion>

  <Accordion title="Welches Modell empfehlen Sie?">
    **Empfohlener Standard:** Verwenden Sie das stärkste Modell der neuesten Generation, das in Ihrem Provider-Stack verfügbar ist.
    **Für tool-aktivierte oder Agents mit nicht vertrauenswürdiger Eingabe:** Priorisieren Sie Modellstärke vor Kosten.
    **Für routinemäßigen/risikoarmen Chat:** Verwenden Sie günstigere Fallback-Modelle und routen Sie nach Agent-Rolle.

    MiniMax hat eigene Dokumentation: [MiniMax](/de/providers/minimax) und
    [Lokale Modelle](/de/gateway/local-models).

    Faustregel: Verwenden Sie für wichtige Arbeit das **beste Modell, das Sie sich leisten können**, und ein günstigeres
    Modell für routinemäßigen Chat oder Zusammenfassungen. Sie können Modelle pro Agent routen und Sub-agents verwenden, um
    lange Aufgaben zu parallelisieren (jeder Sub-agent verbraucht Tokens). Siehe [Modelle](/de/concepts/models) und
    [Sub-agents](/de/tools/subagents).

    Starke Warnung: Schwächere/überquantisierte Modelle sind anfälliger für Prompt
    Injection und unsicheres Verhalten. Siehe [Sicherheit](/de/gateway/security).

    Mehr Kontext: [Modelle](/de/concepts/models).

  </Accordion>

  <Accordion title="Wie wechsle ich Modelle, ohne meine Konfiguration zu löschen?">
    Verwenden Sie **Modell-Befehle** oder bearbeiten Sie nur die **Modell**-Felder. Vermeiden Sie vollständiges Ersetzen der Konfiguration.

    Sichere Optionen:

    - `/model` im Chat (schnell, pro Sitzung)
    - `openclaw models set ...` (aktualisiert nur die Modellkonfiguration)
    - `openclaw configure --section model` (interaktiv)
    - Bearbeiten Sie `agents.defaults.model` in `~/.openclaw/openclaw.json`

    Vermeiden Sie `config.apply` mit einem Teilobjekt, es sei denn, Sie möchten bewusst die ganze Konfiguration ersetzen.
    Bei RPC-Bearbeitungen prüfen Sie zuerst mit `config.schema.lookup` und bevorzugen `config.patch`. Die Lookup-Payload gibt Ihnen den normalisierten Pfad, flache Schema-Dokumentation/Einschränkungen und Zusammenfassungen der unmittelbaren Kinder
    für partielle Updates.
    Wenn Sie die Konfiguration überschrieben haben, stellen Sie sie aus einem Backup wieder her oder führen Sie `openclaw doctor` erneut aus, um zu reparieren.

    Dokumentation: [Modelle](/de/concepts/models), [Configure](/cli/configure), [Config](/cli/config), [Doctor](/de/gateway/doctor).

  </Accordion>

  <Accordion title="Kann ich selbst gehostete Modelle verwenden (llama.cpp, vLLM, Ollama)?">
    Ja. Ollama ist der einfachste Weg für lokale Modelle.

    Schnellstes Setup:

    1. Installieren Sie Ollama von `https://ollama.com/download`
    2. Pullen Sie ein lokales Modell wie `ollama pull gemma4`
    3. Wenn Sie auch Cloud-Modelle möchten, führen Sie `ollama signin` aus
    4. Führen Sie `openclaw onboard` aus und wählen Sie `Ollama`
    5. Wählen Sie `Local` oder `Cloud + Local`

    Hinweise:

    - `Cloud + Local` liefert Ihnen Cloud-Modelle plus Ihre lokalen Ollama-Modelle
    - Cloud-Modelle wie `kimi-k2.5:cloud` benötigen keinen lokalen Pull
    - Für manuelles Wechseln verwenden Sie `openclaw models list` und `openclaw models set ollama/<model>`

    Sicherheitshinweis: Kleinere oder stark quantisierte Modelle sind anfälliger für Prompt
    Injection. Für jeden Bot, der Tools verwenden kann, empfehlen wir dringend **große Modelle**.
    Wenn Sie trotzdem kleine Modelle möchten, aktivieren Sie Sandboxing und strikte Tool-Allowlists.

    Dokumentation: [Ollama](/de/providers/ollama), [Lokale Modelle](/de/gateway/local-models),
    [Model providers](/de/concepts/model-providers), [Sicherheit](/de/gateway/security),
    [Sandboxing](/de/gateway/sandboxing).

  </Accordion>

  <Accordion title="Was verwenden OpenClaw, Flawd und Krill für Modelle?">
    - Diese Deployments können sich unterscheiden und sich im Laufe der Zeit ändern; es gibt keine feste Provider-Empfehlung.
    - Prüfen Sie die aktuelle Laufzeiteinstellung auf jedem Gateway mit `openclaw models status`.
    - Für sicherheitssensible/tool-aktivierte Agents verwenden Sie das stärkste Modell der neuesten Generation, das verfügbar ist.
  </Accordion>

  <Accordion title="Wie wechsle ich Modelle on the fly (ohne Neustart)?">
    Verwenden Sie den Befehl `/model` als eigenständige Nachricht:

    ```
    /model sonnet
    /model opus
    /model gpt
    /model gpt-mini
    /model gemini
    /model gemini-flash
    /model gemini-flash-lite
    ```

    Das sind die eingebauten Aliasse. Eigene Aliasse können über `agents.defaults.models` hinzugefügt werden.

    Verfügbare Modelle können Sie mit `/model`, `/model list` oder `/model status` auflisten.

    `/model` (und `/model list`) zeigt eine kompakte, nummerierte Auswahl. Wählen Sie per Nummer:

    ```
    /model 3
    ```

    Sie können auch ein bestimmtes Auth-Profil für den Provider erzwingen (pro Sitzung):

    ```
    /model opus@anthropic:default
    /model opus@anthropic:work
    ```

    Tipp: `/model status` zeigt, welcher Agent aktiv ist, welche Datei `auth-profiles.json` verwendet wird und welches Auth-Profil als Nächstes versucht wird.
    Außerdem werden der konfigurierte Provider-Endpunkt (`baseUrl`) und der API-Modus (`api`) angezeigt, wenn verfügbar.

    **Wie löse ich die Fixierung eines mit @profile gesetzten Profils?**

    Führen Sie `/model` erneut **ohne** das Suffix `@profile` aus:

    ```
    /model anthropic/claude-opus-4-6
    ```

    Wenn Sie zum Standard zurückkehren möchten, wählen Sie ihn in `/model` aus (oder senden Sie `/model <default provider/model>`).
    Verwenden Sie `/model status`, um zu bestätigen, welches Auth-Profil aktiv ist.

  </Accordion>

  <Accordion title="Kann ich GPT 5.2 für tägliche Aufgaben und Codex 5.3 zum Coden verwenden?">
    Ja. Setzen Sie eines als Standard und wechseln Sie nach Bedarf:

    - **Schneller Wechsel (pro Sitzung):** `/model gpt-5.4` für tägliche Aufgaben, `/model openai-codex/gpt-5.4` zum Coden mit Codex OAuth.
    - **Standard + Wechsel:** Setzen Sie `agents.defaults.model.primary` auf `openai/gpt-5.4` und wechseln Sie dann beim Coden zu `openai-codex/gpt-5.4` (oder umgekehrt).
    - **Sub-agents:** Leiten Sie Coding-Aufgaben an Sub-agents mit einem anderen Standardmodell weiter.

    Siehe [Modelle](/de/concepts/models) und [Slash commands](/de/tools/slash-commands).

  </Accordion>

  <Accordion title="Wie konfiguriere ich den Fast-Modus für GPT 5.4?">
    Verwenden Sie entweder einen Sitzungsumschalter oder einen Standardwert in der Konfiguration:

    - **Pro Sitzung:** Senden Sie `/fast on`, während die Sitzung `openai/gpt-5.4` oder `openai-codex/gpt-5.4` verwendet.
    - **Standard pro Modell:** Setzen Sie `agents.defaults.models["openai/gpt-5.4"].params.fastMode` auf `true`.
    - **Auch für Codex OAuth:** Wenn Sie zusätzlich `openai-codex/gpt-5.4` verwenden, setzen Sie dort denselben Schalter.

    Beispiel:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openai/gpt-5.4": {
              params: {
                fastMode: true,
              },
            },
            "openai-codex/gpt-5.4": {
              params: {
                fastMode: true,
              },
            },
          },
        },
      },
    }
    ```

    Für OpenAI wird der Fast-Modus auf unterstützten nativen Responses-Anfragen auf `service_tier = "priority"` abgebildet. Sitzungs-Overrides durch `/fast` schlagen Konfigurations-Standardwerte.

    Siehe [Thinking and fast mode](/de/tools/thinking) und [OpenAI fast mode](/de/providers/openai#openai-fast-mode).

  </Accordion>

  <Accordion title='Warum sehe ich "Model ... is not allowed" und dann keine Antwort?'>
    Wenn `agents.defaults.models` gesetzt ist, wird es zur **Allowlist** für `/model` und alle
    Sitzungs-Overrides. Die Wahl eines Modells, das nicht in dieser Liste steht, gibt zurück:

    ```
    Model "provider/model" is not allowed. Use /model to list available models.
    ```

    Dieser Fehler wird **anstelle** einer normalen Antwort zurückgegeben. Behebung: Fügen Sie das Modell zu
    `agents.defaults.models` hinzu, entfernen Sie die Allowlist oder wählen Sie ein Modell aus `/model list`.

  </Accordion>

  <Accordion title='Warum sehe ich "Unknown model: minimax/MiniMax-M2.7"?'>
    Das bedeutet, dass der **Provider nicht konfiguriert** ist (es wurde keine MiniMax-Provider-Konfiguration oder kein Auth-
    Profil gefunden), sodass das Modell nicht aufgelöst werden kann.

    Checkliste zur Behebung:

    1. Aktualisieren Sie auf eine aktuelle OpenClaw-Version (oder führen Sie den Source-Branch `main` aus) und starten Sie das Gateway neu.
    2. Stellen Sie sicher, dass MiniMax konfiguriert ist (Assistent oder JSON) oder dass MiniMax-Authentifizierung
       in env/Auth-Profilen vorhanden ist, damit der passende Provider injiziert werden kann
       (`MINIMAX_API_KEY` für `minimax`, `MINIMAX_OAUTH_TOKEN` oder gespeichertes MiniMax-
       OAuth für `minimax-portal`).
    3. Verwenden Sie für Ihren Auth-Pfad die exakte Modell-ID (Groß-/Kleinschreibung beachten):
       `minimax/MiniMax-M2.7` oder `minimax/MiniMax-M2.7-highspeed` für API-Key-
       Setup oder `minimax-portal/MiniMax-M2.7` /
       `minimax-portal/MiniMax-M2.7-highspeed` für OAuth-Setup.
    4. Führen Sie aus:

       ```bash
       openclaw models list
       ```

       und wählen Sie aus der Liste (oder `/model list` im Chat).

    Siehe [MiniMax](/de/providers/minimax) und [Modelle](/de/concepts/models).

  </Accordion>

  <Accordion title="Kann ich MiniMax als Standard und OpenAI für komplexe Aufgaben verwenden?">
    Ja. Verwenden Sie **MiniMax als Standard** und wechseln Sie **pro Sitzung** das Modell, wenn nötig.
    Fallbacks sind für **Fehler**, nicht für „schwere Aufgaben“, daher verwenden Sie `/model` oder einen separaten Agenten.

    **Option A: pro Sitzung wechseln**

    ```json5
    {
      env: { MINIMAX_API_KEY: "sk-...", OPENAI_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "minimax/MiniMax-M2.7" },
          models: {
            "minimax/MiniMax-M2.7": { alias: "minimax" },
            "openai/gpt-5.4": { alias: "gpt" },
          },
        },
      },
    }
    ```

    Dann:

    ```
    /model gpt
    ```

    **Option B: getrennte Agents**

    - Agent A Standard: MiniMax
    - Agent B Standard: OpenAI
    - Nach Agent routen oder mit `/agent` wechseln

    Dokumentation: [Modelle](/de/concepts/models), [Multi-Agent Routing](/de/concepts/multi-agent), [MiniMax](/de/providers/minimax), [OpenAI](/de/providers/openai).

  </Accordion>

  <Accordion title="Sind opus / sonnet / gpt eingebaute Shortcuts?">
    Ja. OpenClaw liefert einige Standard-Shortcuts mit (werden nur angewendet, wenn das Modell in `agents.defaults.models` existiert):

    - `opus` → `anthropic/claude-opus-4-6`
    - `sonnet` → `anthropic/claude-sonnet-4-6`
    - `gpt` → `openai/gpt-5.4`
    - `gpt-mini` → `openai/gpt-5.4-mini`
    - `gpt-nano` → `openai/gpt-5.4-nano`
    - `gemini` → `google/gemini-3.1-pro-preview`
    - `gemini-flash` → `google/gemini-3-flash-preview`
    - `gemini-flash-lite` → `google/gemini-3.1-flash-lite-preview`

    Wenn Sie einen eigenen Alias mit demselben Namen setzen, gewinnt Ihr Wert.

  </Accordion>

  <Accordion title="Wie definiere/überschreibe ich Modell-Shortcuts (Aliasse)?">
    Aliasse kommen aus `agents.defaults.models.<modelId>.alias`. Beispiel:

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "anthropic/claude-opus-4-6" },
          models: {
            "anthropic/claude-opus-4-6": { alias: "opus" },
            "anthropic/claude-sonnet-4-6": { alias: "sonnet" },
            "anthropic/claude-haiku-4-5": { alias: "haiku" },
          },
        },
      },
    }
    ```

    Dann löst `/model sonnet` (oder `/<alias>`, wenn unterstützt) zu dieser Modell-ID auf.

  </Accordion>

  <Accordion title="Wie füge ich Modelle anderer Provider wie OpenRouter oder Z.AI hinzu?">
    OpenRouter (Pay-per-Token; viele Modelle):

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "openrouter/anthropic/claude-sonnet-4-6" },
          models: { "openrouter/anthropic/claude-sonnet-4-6": {} },
        },
      },
      env: { OPENROUTER_API_KEY: "sk-or-..." },
    }
    ```

    Z.AI (GLM-Modelle):

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "zai/glm-5" },
          models: { "zai/glm-5": {} },
        },
      },
      env: { ZAI_API_KEY: "..." },
    }
    ```

    Wenn Sie auf ein Provider-/Modellpaar verweisen, aber der erforderliche Providerschlüssel fehlt, erhalten Sie einen Laufzeit-Auth-Fehler (z. B. `No API key found for provider "zai"`).

    **No API key found for provider nach dem Hinzufügen eines neuen Agenten**

    Das bedeutet gewöhnlich, dass der **neue Agent** einen leeren Auth-Store hat. Auth ist pro Agent und
    wird gespeichert unter:

    ```
    ~/.openclaw/agents/<agentId>/agent/auth-profiles.json
    ```

    Mögliche Lösungen:

    - Führen Sie `openclaw agents add <id>` aus und konfigurieren Sie Auth im Assistenten.
    - Oder kopieren Sie `auth-profiles.json` aus dem `agentDir` des Hauptagenten in das `agentDir` des neuen Agenten.

    Verwenden Sie **nicht** dasselbe `agentDir` für mehrere Agents; das führt zu Auth-/Sitzungskollisionen.

  </Accordion>
</AccordionGroup>

## Modell-Failover und "All models failed"

<AccordionGroup>
  <Accordion title="Wie funktioniert Failover?">
    Failover erfolgt in zwei Stufen:

    1. **Rotation von Auth-Profilen** innerhalb desselben Providers.
    2. **Modell-Fallback** zum nächsten Modell in `agents.defaults.model.fallbacks`.

    Cooldowns gelten für fehlschlagende Profile (exponentielles Backoff), sodass OpenClaw weiter antworten kann, auch wenn ein Provider ratelimitiert ist oder vorübergehend ausfällt.

    Der Rate-Limit-Bucket umfasst mehr als einfache `429`-Antworten. OpenClaw
    behandelt auch Meldungen wie `Too many concurrent requests`,
    `ThrottlingException`, `concurrency limit reached`,
    `workers_ai ... quota limit exceeded`, `resource exhausted` und periodische
    Nutzungsfenster-Limits (`weekly/monthly limit