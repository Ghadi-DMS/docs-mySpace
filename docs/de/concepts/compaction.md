---
read_when:
    - Sie möchten die automatische Kompaktierung und `/compact` verstehen
    - Sie debuggen lange Sitzungen, die an Kontextgrenzen stoßen
summary: Wie OpenClaw lange Unterhaltungen zusammenfasst, um innerhalb der Modellgrenzen zu bleiben
title: Kompaktierung
x-i18n:
    generated_at: "2026-04-08T02:14:15Z"
    model: gpt-5.4
    provider: openai
    source_hash: e6590b82a8c3a9c310998d653459ca4d8612495703ca0a8d8d306d7643142fd1
    source_path: concepts/compaction.md
    workflow: 15
---

# Kompaktierung

Jedes Modell hat ein Kontextfenster -- die maximale Anzahl an Tokens, die es verarbeiten kann.
Wenn sich eine Unterhaltung dieser Grenze nähert, **kompaktiert** OpenClaw ältere Nachrichten
zu einer Zusammenfassung, damit der Chat fortgesetzt werden kann.

## So funktioniert es

1. Ältere Gesprächsverläufe werden zu einem kompakten Eintrag zusammengefasst.
2. Die Zusammenfassung wird im Sitzungsprotokoll gespeichert.
3. Neuere Nachrichten bleiben unverändert erhalten.

Wenn OpenClaw den Verlauf in Kompaktierungsblöcke aufteilt, hält es Tool-Aufrufe
des Assistenten zusammen mit den zugehörigen `toolResult`-Einträgen gepaart. Wenn ein Teilungspunkt
innerhalb eines Tool-Blocks landet, verschiebt OpenClaw die Grenze, damit das Paar zusammenbleibt und
das aktuelle, nicht zusammengefasste Ende erhalten bleibt.

Der vollständige Unterhaltungsverlauf bleibt auf dem Datenträger gespeichert. Die Kompaktierung ändert nur, was das
Modell im nächsten Zug sieht.

## Automatische Kompaktierung

Die automatische Kompaktierung ist standardmäßig aktiviert. Sie wird ausgeführt, wenn sich die Sitzung der Kontextgrenze
nähert oder wenn das Modell einen Kontextüberlauffehler zurückgibt (in diesem Fall
kompaktiert OpenClaw und versucht es erneut). Typische Überlauf-Signaturen sind
`request_too_large`, `context length exceeded`, `input exceeds the maximum
number of tokens`, `input token count exceeds the maximum number of input
tokens`, `input is too long for the model` und `ollama error: context length
exceeded`.

<Info>
Bevor kompaktieret wird, erinnert OpenClaw den Agenten automatisch daran, wichtige
Notizen in [memory](/de/concepts/memory)-Dateien zu speichern. Das verhindert Kontextverlust.
</Info>

Verwenden Sie die Einstellung `agents.defaults.compaction` in Ihrer `openclaw.json`, um das Verhalten der Kompaktierung zu konfigurieren (Modus, Ziel-Tokens usw.).
Die Zusammenfassung bei der Kompaktierung bewahrt standardmäßig intransparente Bezeichner (`identifierPolicy: "strict"`). Sie können dies mit `identifierPolicy: "off"` überschreiben oder mit `identifierPolicy: "custom"` und `identifierInstructions` eigenen Text bereitstellen.

Optional können Sie über `agents.defaults.compaction.model` ein anderes Modell für die Kompaktierungszusammenfassung angeben. Das ist nützlich, wenn Ihr primäres Modell ein lokales oder kleines Modell ist und Sie möchten, dass die Kompaktierungszusammenfassungen von einem leistungsfähigeren Modell erstellt werden. Die Überschreibung akzeptiert jede Zeichenfolge im Format `provider/model-id`:

```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "model": "openrouter/anthropic/claude-sonnet-4-6"
      }
    }
  }
}
```

Das funktioniert auch mit lokalen Modellen, zum Beispiel mit einem zweiten Ollama-Modell, das für Zusammenfassungen vorgesehen ist, oder mit einem feinabgestimmten Kompaktierungsspezialisten:

```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "model": "ollama/llama3.1:8b"
      }
    }
  }
}
```

Wenn kein Wert gesetzt ist, verwendet die Kompaktierung das primäre Modell des Agenten.

## Austauschbare Kompaktierungsanbieter

Plugins können über `registerCompactionProvider()` in der Plugin-API einen benutzerdefinierten Kompaktierungsanbieter registrieren. Wenn ein Anbieter registriert und konfiguriert ist, delegiert OpenClaw die Zusammenfassung an ihn statt an die integrierte LLM-Pipeline.

Um einen registrierten Anbieter zu verwenden, setzen Sie die Anbieter-ID in Ihrer Konfiguration:

```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "provider": "my-provider"
      }
    }
  }
}
```

Das Setzen von `provider` erzwingt automatisch `mode: "safeguard"`. Anbieter erhalten dieselben Kompaktierungsanweisungen und dieselbe Richtlinie zur Bewahrung von Bezeichnern wie der integrierte Pfad, und OpenClaw bewahrt nach der Anbieterausgabe weiterhin den Suffix-Kontext von aktuellen und geteilten Gesprächszügen. Wenn der Anbieter fehlschlägt oder ein leeres Ergebnis zurückgibt, greift OpenClaw auf die integrierte LLM-Zusammenfassung zurück.

## Automatische Kompaktierung (standardmäßig aktiviert)

Wenn sich eine Sitzung dem Kontextfenster des Modells nähert oder es überschreitet, löst OpenClaw die automatische Kompaktierung aus und versucht die ursprüngliche Anfrage möglicherweise mit dem kompaktierten Kontext erneut.

Sie sehen dann:

- `🧹 Auto-compaction complete` im ausführlichen Modus
- `/status`, das `🧹 Compactions: <count>` anzeigt

Vor der Kompaktierung kann OpenClaw einen **stillen Memory-Flush**-Zug ausführen, um
dauerhafte Notizen auf dem Datenträger zu speichern. Weitere Details und Konfigurationsmöglichkeiten finden Sie unter [Memory](/de/concepts/memory).

## Manuelle Kompaktierung

Geben Sie `/compact` in einem beliebigen Chat ein, um eine Kompaktierung zu erzwingen. Fügen Sie Anweisungen hinzu, um
die Zusammenfassung zu steuern:

```
/compact Focus on the API design decisions
```

## Verwendung eines anderen Modells

Standardmäßig verwendet die Kompaktierung das primäre Modell Ihres Agenten. Sie können ein leistungsfähigeres
Modell für bessere Zusammenfassungen verwenden:

```json5
{
  agents: {
    defaults: {
      compaction: {
        model: "openrouter/anthropic/claude-sonnet-4-6",
      },
    },
  },
}
```

## Hinweis zum Start der Kompaktierung

Standardmäßig läuft die Kompaktierung still im Hintergrund. Um einen kurzen Hinweis anzuzeigen, wenn die Kompaktierung
beginnt, aktivieren Sie `notifyUser`:

```json5
{
  agents: {
    defaults: {
      compaction: {
        notifyUser: true,
      },
    },
  },
}
```

Wenn aktiviert, sieht der Benutzer zu Beginn jedes Kompaktierungslaufs eine kurze Nachricht (zum Beispiel „Kontext wird kompaktieret...“).

## Kompaktierung vs. Pruning

|                  | Kompaktierung                 | Pruning                          |
| ---------------- | ----------------------------- | -------------------------------- |
| **Was sie tut**  | Fasst ältere Unterhaltungen zusammen | Kürzt alte Tool-Ergebnisse       |
| **Gespeichert?** | Ja (im Sitzungsprotokoll)     | Nein (nur im Speicher, pro Anfrage) |
| **Umfang**       | Gesamte Unterhaltung          | Nur Tool-Ergebnisse              |

[Session pruning](/de/concepts/session-pruning) ist eine leichtgewichtigere Ergänzung, die
Tool-Ausgaben kürzt, ohne sie zusammenzufassen.

## Fehlerbehebung

**Zu häufige Kompaktierung?** Das Kontextfenster des Modells könnte klein sein, oder Tool-
Ausgaben könnten groß sein. Versuchen Sie,
[session pruning](/de/concepts/session-pruning) zu aktivieren.

**Der Kontext wirkt nach der Kompaktierung veraltet?** Verwenden Sie `/compact Focus on <topic>`, um
die Zusammenfassung zu steuern, oder aktivieren Sie den [memory flush](/de/concepts/memory), damit Notizen
erhalten bleiben.

**Sie brauchen einen Neustart?** `/new` startet eine neue Sitzung ohne Kompaktierung.

Für die erweiterte Konfiguration (reservierte Tokens, Bezeichnerbewahrung, benutzerdefinierte
Kontext-Engines, serverseitige OpenAI-Kompaktierung) siehe den
[Session Management Deep Dive](/de/reference/session-management-compaction).

## Verwandt

- [Session](/de/concepts/session) — Sitzungsverwaltung und Lebenszyklus
- [Session Pruning](/de/concepts/session-pruning) — Tool-Ergebnisse kürzen
- [Context](/de/concepts/context) — wie Kontext für Agentenzüge aufgebaut wird
- [Hooks](/de/automation/hooks) — Hooks für den Kompaktierungslebenszyklus (before_compaction, after_compaction)
