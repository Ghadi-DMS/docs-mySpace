---
read_when:
    - Beim Ausführen von Skripten aus dem Repository
    - Beim Hinzufügen oder Ändern von Skripten unter ./scripts
summary: 'Repository-Skripte: Zweck, Umfang und Sicherheitshinweise'
title: Skripte
x-i18n:
    generated_at: "2026-04-08T02:15:24Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3ecf1e9327929948fb75f80e306963af49b353c0aa8d3b6fa532ca964ff8b975
    source_path: help/scripts.md
    workflow: 15
---

# Skripte

Das Verzeichnis `scripts/` enthält Hilfsskripte für lokale Workflows und Betriebsaufgaben.
Verwenden Sie diese, wenn eine Aufgabe eindeutig an ein Skript gebunden ist; andernfalls bevorzugen Sie die CLI.

## Konventionen

- Skripte sind **optional**, sofern sie nicht in der Dokumentation oder in Release-Checklisten referenziert werden.
- Bevorzugen Sie CLI-Oberflächen, wenn sie vorhanden sind (Beispiel: Authentifizierungsüberwachung verwendet `openclaw models status --check`).
- Gehen Sie davon aus, dass Skripte hostspezifisch sind; lesen Sie sie vor der Ausführung auf einem neuen Rechner.

## Skripte zur Authentifizierungsüberwachung

Die Authentifizierungsüberwachung wird in [Authentication](/de/gateway/authentication) behandelt. Die Skripte unter `scripts/` sind optionale Extras für systemd-/Termux-Telefon-Workflows.

## GitHub-Lesehilfe

Verwenden Sie `scripts/gh-read`, wenn `gh` für Repository-bezogene Leseaufrufe ein GitHub-App-Installationstoken verwenden soll, während normales `gh` für Schreibaktionen mit Ihrem persönlichen Login weiterläuft.

Erforderliche Umgebungsvariablen:

- `OPENCLAW_GH_READ_APP_ID`
- `OPENCLAW_GH_READ_PRIVATE_KEY_FILE`

Optionale Umgebungsvariablen:

- `OPENCLAW_GH_READ_INSTALLATION_ID`, wenn Sie die installationsbasierte Ermittlung über das Repository überspringen möchten
- `OPENCLAW_GH_READ_PERMISSIONS` als kommagetrenntes Override für die anzufordernde Teilmenge an Leseberechtigungen

Reihenfolge der Repository-Auflösung:

- `gh ... -R owner/repo`
- `GH_REPO`
- `git remote origin`

Beispiele:

- `scripts/gh-read pr view 123`
- `scripts/gh-read run list -R openclaw/openclaw`
- `scripts/gh-read api repos/openclaw/openclaw/pulls/123`

## Beim Hinzufügen von Skripten

- Halten Sie Skripte fokussiert und dokumentiert.
- Fügen Sie einen kurzen Eintrag in der relevanten Dokumentation hinzu (oder erstellen Sie eine, falls sie fehlt).
