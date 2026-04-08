---
read_when:
    - Erweitern von qa-lab oder qa-channel
    - Hinzufügen repo-gestützter QA-Szenarien
    - Aufbauen realitätsnäherer QA-Automatisierung rund um das Gateway-Dashboard
summary: Private QA-Automatisierungsstruktur für qa-lab, qa-channel, Seed-Szenarien und Protokollberichte
title: QA-E2E-Automatisierung
x-i18n:
    generated_at: "2026-04-08T02:13:58Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3b4aa5acc8e77303f4045d4f04372494cae21b89d2fdaba856dbb4855ced9d27
    source_path: concepts/qa-e2e-automation.md
    workflow: 15
---

# QA-E2E-Automatisierung

Der private QA-Stack soll OpenClaw auf eine realistischere,
kanalartige Weise testen, als es ein einzelner Unit-Test leisten kann.

Aktuelle Bestandteile:

- `extensions/qa-channel`: synthetischer Nachrichtenkanal mit Oberflächen für DM, Kanal, Thread,
  Reaktion, Bearbeiten und Löschen.
- `extensions/qa-lab`: Debugger-UI und QA-Bus zum Beobachten des Transkripts,
  Einspeisen eingehender Nachrichten und Exportieren eines Markdown-Berichts.
- `qa/`: repo-gestützte Seed-Assets für die Startaufgabe und grundlegende QA-
  Szenarien.

Der aktuelle QA-Operator-Workflow ist eine QA-Site mit zwei Bereichen:

- Links: Gateway-Dashboard (Control UI) mit dem Agenten.
- Rechts: QA Lab, das das Slack-ähnliche Transkript und den Szenarioplan anzeigt.

Ausführen mit:

```bash
pnpm qa:lab:up
```

Dadurch wird die QA-Site gebaut, die Docker-gestützte Gateway-Lane gestartet und die
QA-Lab-Seite bereitgestellt, auf der ein Operator oder eine Automatisierungsschleife dem Agenten eine QA-
Mission geben, echtes Kanalverhalten beobachten und festhalten kann, was funktioniert hat, fehlgeschlagen ist oder
blockiert geblieben ist.

Für schnellere Iteration an der QA-Lab-UI, ohne das Docker-Image jedes Mal neu zu bauen,
starten Sie den Stack mit einem per Bind-Mount eingebundenen QA-Lab-Bundle:

```bash
pnpm openclaw qa docker-build-image
pnpm qa:lab:build
pnpm qa:lab:up:fast
pnpm qa:lab:watch
```

`qa:lab:up:fast` hält die Docker-Dienste auf einem vorgebauten Image und bind-mountet
`extensions/qa-lab/web/dist` in den `qa-lab`-Container ein. `qa:lab:watch`
baut dieses Bundle bei Änderungen neu, und der Browser lädt automatisch neu, wenn sich der QA-Lab-
Asset-Hash ändert.

## Repo-gestützte Seeds

Seed-Assets befinden sich in `qa/`:

- `qa/scenarios.md`

Diese sind absichtlich in Git enthalten, damit der QA-Plan sowohl für Menschen als auch für den
Agenten sichtbar ist. Die grundlegende Liste sollte breit genug bleiben, um Folgendes abzudecken:

- DM- und Kanal-Chat
- Thread-Verhalten
- Lebenszyklus von Nachrichtenaktionen
- Cron-Callbacks
- Memory-Abruf
- Modellwechsel
- Übergabe an Subagenten
- Lesen des Repos und der Dokumentation
- eine kleine Build-Aufgabe wie Lobster Invaders

## Berichterstellung

`qa-lab` exportiert einen Markdown-Protokollbericht aus der beobachteten Bus-Zeitleiste.
Der Bericht sollte beantworten:

- Was funktioniert hat
- Was fehlgeschlagen ist
- Was blockiert geblieben ist
- Welche Folge-Szenarien sich hinzuzufügen lohnen

## Zugehörige Dokumentation

- [Testing](/de/help/testing)
- [QA Channel](/de/channels/qa-channel)
- [Dashboard](/web/dashboard)
