---
read_when:
    - Erweitern von qa-lab oder qa-channel
    - Hinzufügen von repository-gestützten QA-Szenarien
    - Erstellen von QA-Automatisierung mit höherem Realismus rund um das Gateway-Dashboard
summary: Private QA-Automatisierungsstruktur für qa-lab, qa-channel, vorab definierte Szenarien und Protokollberichte
title: QA-E2E-Automatisierung
x-i18n:
    generated_at: "2026-04-10T06:21:16Z"
    model: gpt-5.4
    provider: openai
    source_hash: 357d6698304ff7a8c4aa8a7be97f684d50f72b524740050aa761ac0ee68266de
    source_path: concepts/qa-e2e-automation.md
    workflow: 15
---

# QA-E2E-Automatisierung

Der private QA-Stack ist dafür gedacht, OpenClaw auf eine realistischere,
kanalähnliche Weise zu prüfen, als es ein einzelner Unit-Test kann.

Aktuelle Bestandteile:

- `extensions/qa-channel`: synthetischer Nachrichtenkanal mit Oberflächen für Direktnachrichten, Kanäle, Threads,
  Reaktionen, Bearbeitungen und Löschungen.
- `extensions/qa-lab`: Debugger-UI und QA-Bus zum Beobachten des Transkripts,
  Einspeisen eingehender Nachrichten und Exportieren eines Markdown-Berichts.
- `qa/`: repository-gestützte Seed-Assets für die Kickoff-Aufgabe und grundlegende QA-
  Szenarien.

Der aktuelle QA-Operator-Workflow ist eine QA-Site mit zwei Bereichen:

- Links: Gateway-Dashboard (Control UI) mit dem Agenten.
- Rechts: QA Lab mit dem Slack-ähnlichen Transkript und dem Szenarioplan.

Starten Sie sie mit:

```bash
pnpm qa:lab:up
```

Dadurch wird die QA-Site gebaut, die Docker-gestützte Gateway-Lane gestartet und die
QA-Lab-Seite bereitgestellt, auf der ein Operator oder eine Automatisierungsschleife dem Agenten eine QA-
Mission geben, echtes Kanalverhalten beobachten und erfassen kann, was funktioniert hat, fehlgeschlagen ist oder blockiert geblieben ist.

Für schnellere Iteration an der QA-Lab-UI, ohne das Docker-Image jedes Mal neu zu bauen,
starten Sie den Stack mit einem bind-gemounteten QA-Lab-Bundle:

```bash
pnpm openclaw qa docker-build-image
pnpm qa:lab:build
pnpm qa:lab:up:fast
pnpm qa:lab:watch
```

`qa:lab:up:fast` hält die Docker-Services auf einem vorgebauten Image und bind-mountet
`extensions/qa-lab/web/dist` in den `qa-lab`-Container. `qa:lab:watch`
baut dieses Bundle bei Änderungen neu, und der Browser lädt automatisch neu, wenn sich der Asset-Hash von QA Lab ändert.

Für eine kurzlebige Linux-VM-Lane, ohne Docker in den QA-Pfad einzubinden, führen Sie aus:

```bash
pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline
```

Dadurch wird ein frischer Multipass-Gast gestartet, Abhängigkeiten werden installiert, OpenClaw
innerhalb des Gasts gebaut, `qa suite` ausgeführt und anschließend der normale QA-Bericht sowie die
Zusammenfassung zurück nach `.artifacts/qa-e2e/...` auf dem Host kopiert.
Dabei wird dasselbe Verhalten zur Szenarioauswahl wie bei `qa suite` auf dem Host verwendet.
Live-Ausführungen leiten die unterstützten QA-Authentifizierungseingaben weiter, die für den
Gast praktikabel sind: provider-basierte Schlüssel über Umgebungsvariablen, der Konfigurationspfad des QA-Live-Providers und
`CODEX_HOME`, falls vorhanden. Halten Sie `--output-dir` unter dem Repository-Root, damit der Gast
über den gemounteten Workspace zurückschreiben kann.

## Repository-gestützte Seeds

Seed-Assets liegen in `qa/`:

- `qa/scenarios/index.md`
- `qa/scenarios/*.md`

Diese sind bewusst in Git enthalten, damit der QA-Plan sowohl für Menschen als auch für den
Agenten sichtbar ist. Die grundlegende Liste sollte breit genug bleiben, um Folgendes abzudecken:

- Direktnachrichten- und Kanal-Chat
- Thread-Verhalten
- Lebenszyklus von Nachrichtenaktionen
- Cron-Callbacks
- Speicherabruf
- Modellwechsel
- Subagent-Übergabe
- Lesen des Repositorys und Lesen der Dokumentation
- eine kleine Build-Aufgabe wie Lobster Invaders

## Berichterstellung

`qa-lab` exportiert einen Markdown-Protokollbericht aus der beobachteten Bus-Zeitleiste.
Der Bericht sollte beantworten:

- Was funktioniert hat
- Was fehlgeschlagen ist
- Was blockiert geblieben ist
- Welche Folge-Szenarien sich hinzuzufügen lohnen

Für Prüfungen von Charakter und Stil führen Sie dasselbe Szenario über mehrere Live-Modell-
Referenzen hinweg aus und schreiben einen bewerteten Markdown-Bericht:

```bash
pnpm openclaw qa character-eval \
  --model openai/gpt-5.4,thinking=xhigh \
  --model openai/gpt-5.2,thinking=xhigh \
  --model openai/gpt-5,thinking=xhigh \
  --model anthropic/claude-opus-4-6,thinking=high \
  --model anthropic/claude-sonnet-4-6,thinking=high \
  --model zai/glm-5.1,thinking=high \
  --model moonshot/kimi-k2.5,thinking=high \
  --model google/gemini-3.1-pro-preview,thinking=high \
  --judge-model openai/gpt-5.4,thinking=xhigh,fast \
  --judge-model anthropic/claude-opus-4-6,thinking=high \
  --blind-judge-models \
  --concurrency 16 \
  --judge-concurrency 16
```

Der Befehl führt lokale QA-Gateway-Child-Prozesse aus, nicht Docker. Szenarien zur Charakterbewertung
sollten die Persona über `SOUL.md` festlegen und dann gewöhnliche Benutzerinteraktionen
wie Chat, Hilfe zum Workspace und kleine Dateiaufgaben ausführen. Dem Kandidatenmodell
soll nicht mitgeteilt werden, dass es bewertet wird. Der Befehl bewahrt jedes vollständige
Transkript auf, zeichnet grundlegende Ausführungsstatistiken auf und bittet dann die
Bewertungsmodelle im schnellen Modus mit `xhigh`-Reasoning, die Ausführungen nach
Natürlichkeit, Stimmung und Humor zu bewerten.
Verwenden Sie `--blind-judge-models`, wenn Sie Provider vergleichen: Der Bewertungs-Prompt erhält weiterhin
jedes Transkript und jeden Ausführungsstatus, aber Kandidaten-Referenzen werden durch neutrale
Bezeichnungen wie `candidate-01` ersetzt; der Bericht ordnet die Rankings nach dem Parsen wieder den echten Referenzen zu.
Kandidatenausführungen verwenden standardmäßig `high` Thinking, mit `xhigh` für OpenAI-Modelle, die dies
unterstützen. Überschreiben Sie einen bestimmten Kandidaten inline mit
`--model provider/model,thinking=<level>`. `--thinking <level>` setzt weiterhin einen
globalen Fallback, und die ältere Form `--model-thinking <provider/model=level>` bleibt
aus Kompatibilitätsgründen erhalten.
OpenAI-Kandidatenreferenzen verwenden standardmäßig den schnellen Modus, sodass dort Prioritätsverarbeitung genutzt wird,
wo der Provider dies unterstützt. Fügen Sie inline `,fast`, `,no-fast` oder `,fast=false` hinzu, wenn für einen
einzelnen Kandidaten oder Bewerter eine Überschreibung nötig ist. Übergeben Sie `--fast` nur, wenn Sie
den schnellen Modus für jedes Kandidatenmodell erzwingen möchten. Dauerwerte für Kandidaten und Bewerter werden
im Bericht für die Benchmark-Analyse erfasst, aber in den Bewertungs-Prompts wird ausdrücklich darauf hingewiesen,
nicht nach Geschwindigkeit zu bewerten.
Sowohl Kandidaten- als auch Bewertungsmodell-Ausführungen verwenden standardmäßig eine Parallelität von 16. Senken Sie
`--concurrency` oder `--judge-concurrency`, wenn Provider-Limits oder lokaler Gateway-
Druck eine Ausführung zu unruhig machen.
Wenn kein Kandidat-`--model` übergeben wird, verwendet die Charakterbewertung standardmäßig
`openai/gpt-5.4`, `openai/gpt-5.2`, `openai/gpt-5`, `anthropic/claude-opus-4-6`,
`anthropic/claude-sonnet-4-6`, `zai/glm-5.1`,
`moonshot/kimi-k2.5` und
`google/gemini-3.1-pro-preview`, wenn kein `--model` übergeben wird.
Wenn kein `--judge-model` übergeben wird, verwenden die Bewerter standardmäßig
`openai/gpt-5.4,thinking=xhigh,fast` und
`anthropic/claude-opus-4-6,thinking=high`.

## Verwandte Dokumentation

- [Testing](/de/help/testing)
- [QA Channel](/de/channels/qa-channel)
- [Dashboard](/web/dashboard)
