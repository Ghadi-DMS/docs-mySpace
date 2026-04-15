---
read_when:
    - Sie möchten, dass die Speicherhochstufung automatisch ausgeführt wird.
    - Sie möchten verstehen, was jede Dreaming-Phase bewirkt.
    - Sie möchten die Konsolidierung abstimmen, ohne `MEMORY.md` zu überladen.
summary: Hintergrund-Konsolidierung des Gedächtnisses mit leichten, tiefen und REM-Phasen sowie einem Traumtagebuch
title: Dreaming (experimentell)
x-i18n:
    generated_at: "2026-04-15T06:21:28Z"
    model: gpt-5.4
    provider: openai
    source_hash: 5882a5068f2eabe54ca9893184e5385330a432b921870c38626399ce11c31e25
    source_path: concepts/dreaming.md
    workflow: 15
---

# Dreaming (experimentell)

Dreaming ist das Hintergrundsystem zur Gedächtniskonsolidierung in `memory-core`.
Es hilft OpenClaw dabei, starke kurzfristige Signale in dauerhaftes Gedächtnis zu überführen und
den Prozess dabei nachvollziehbar und überprüfbar zu halten.

Dreaming ist **optional** und standardmäßig deaktiviert.

## Was Dreaming schreibt

Dreaming verwaltet zwei Arten von Ausgaben:

- **Maschinenzustand** in `memory/.dreams/` (Recall-Speicher, Phasensignale, Ingestion-Prüfpunkte, Sperren).
- **Menschenlesbare Ausgabe** in `DREAMS.md` (oder vorhandener `dreams.md`) und optionale Phasenberichtdateien unter `memory/dreaming/<phase>/YYYY-MM-DD.md`.

Die langfristige Hochstufung schreibt weiterhin nur nach `MEMORY.md`.

## Phasenmodell

Dreaming verwendet drei zusammenarbeitende Phasen:

| Phase | Zweck                                      | Dauerhafter Schreibvorgang |
| ----- | ------------------------------------------ | -------------------------- |
| Light | Kürzliches kurzfristiges Material sortieren und vorbereiten | Nein                       |
| Deep  | Dauerhafte Kandidaten bewerten und hochstufen | Ja (`MEMORY.md`)           |
| REM   | Über Themen und wiederkehrende Ideen reflektieren | Nein                       |

Diese Phasen sind interne Implementierungsdetails, keine separaten vom Benutzer konfigurierten
„Modi“.

### Light-Phase

Die Light-Phase nimmt aktuelle tägliche Gedächtnissignale und Recall-Spuren auf, dedupliziert sie
und bereitet Kandidatenzeilen vor.

- Liest aus dem kurzfristigen Recall-Zustand, aktuellen täglichen Gedächtnisdateien und redigierten Sitzungs-Transkripten, sofern verfügbar.
- Schreibt einen verwalteten Block `## Light Sleep`, wenn der Speicher Inline-Ausgabe enthält.
- Zeichnet Verstärkungssignale für das spätere Deep-Ranking auf.
- Schreibt niemals nach `MEMORY.md`.

### Deep-Phase

Die Deep-Phase entscheidet, was Teil des Langzeitgedächtnisses wird.

- Ordnet Kandidaten mithilfe gewichteter Bewertung und Schwellenwert-Gates.
- Erfordert, dass `minScore`, `minRecallCount` und `minUniqueQueries` erfüllt werden.
- Hydriert Snippets vor dem Schreiben aus Live-Tagesdateien erneut, sodass veraltete oder gelöschte Snippets übersprungen werden.
- Hängt hochgestufte Einträge an `MEMORY.md` an.
- Schreibt eine Zusammenfassung `## Deep Sleep` in `DREAMS.md` und schreibt optional `memory/dreaming/deep/YYYY-MM-DD.md`.

### REM-Phase

Die REM-Phase extrahiert Muster und reflektierende Signale.

- Erstellt Themen- und Reflexionszusammenfassungen aus aktuellen kurzfristigen Spuren.
- Schreibt einen verwalteten Block `## REM Sleep`, wenn der Speicher Inline-Ausgabe enthält.
- Zeichnet REM-Verstärkungssignale auf, die vom Deep-Ranking verwendet werden.
- Schreibt niemals nach `MEMORY.md`.

## Ingestion von Sitzungs-Transkripten

Dreaming kann redigierte Sitzungs-Transkripte in den Dreaming-Korpus aufnehmen. Wenn
Transkripte verfügbar sind, werden sie zusammen mit täglichen
Gedächtnissignalen und Recall-Spuren in die Light-Phase eingespeist. Persönliche und sensible Inhalte werden
vor der Ingestion redigiert.

## Traumtagebuch

Dreaming führt außerdem ein erzählerisches **Traumtagebuch** in `DREAMS.md`.
Sobald nach jeder Phase genügend Material vorhanden ist, führt `memory-core` im Hintergrund nach bestem Bemühen
einen Subagent-Durchlauf aus (unter Verwendung des Standard-Runtime-Modells) und fügt einen kurzen Tagebucheintrag an.

Dieses Tagebuch ist für Menschen in der Dreams-UI gedacht, nicht als Quelle für Hochstufungen.
Von Dreaming erzeugte Tagebuch-/Berichtsartefakte sind von der kurzfristigen
Hochstufung ausgeschlossen. Nur fundierte Gedächtnis-Snippets kommen für die Hochstufung nach
`MEMORY.md` infrage.

Es gibt außerdem einen fundierten historischen Backfill-Pfad für Überprüfungs- und Wiederherstellungsarbeiten:

- `memory rem-harness --path ... --grounded` zeigt eine Vorschau fundierter Tagebuchausgabe aus historischen `YYYY-MM-DD.md`-Notizen.
- `memory rem-backfill --path ...` schreibt reversible fundierte Tagebucheinträge in `DREAMS.md`.
- `memory rem-backfill --path ... --stage-short-term` stellt fundierte dauerhafte Kandidaten in denselben kurzfristigen Evidenzspeicher bereit, den die normale Deep-Phase bereits verwendet.
- `memory rem-backfill --rollback` und `--rollback-short-term` entfernen diese bereitgestellten Backfill-Artefakte, ohne gewöhnliche Tagebucheinträge oder den aktiven kurzfristigen Recall zu berühren.

Die Control UI stellt denselben Tagebuch-Backfill-/Reset-Ablauf bereit, sodass Sie die
Ergebnisse in der Dreams-Szene prüfen können, bevor Sie entscheiden, ob die fundierten Kandidaten
eine Hochstufung verdienen. Die Szene zeigt außerdem einen eigenen fundierten Pfad, damit Sie sehen können,
welche bereitgestellten kurzfristigen Einträge aus historischem Replay stammen, welche hochgestuften
Elemente fundiert gesteuert waren, und nur fundiert bereitgestellte Einträge löschen können, ohne
den gewöhnlichen aktiven kurzfristigen Zustand zu berühren.

## Deep-Ranking-Signale

Das Deep-Ranking verwendet sechs gewichtete Basissignale plus Phasenverstärkung:

| Signal              | Gewicht | Beschreibung                                     |
| ------------------- | ------ | ------------------------------------------------ |
| Frequency           | 0.24   | Wie viele kurzfristige Signale der Eintrag angesammelt hat |
| Relevance           | 0.30   | Durchschnittliche Abrufqualität für den Eintrag  |
| Query diversity     | 0.15   | Unterschiedliche Abfrage-/Tageskontexte, in denen er aufgetaucht ist |
| Recency             | 0.15   | Zeitlich abklingender Aktualitätswert            |
| Consolidation       | 0.10   | Stärke des mehrtägigen Wiederauftretens          |
| Conceptual richness | 0.06   | Dichte der Konzept-Tags aus Snippet/Pfad         |

Treffer aus der Light- und REM-Phase fügen einen kleinen, zeitlich abklingenden Boost aus
`memory/.dreams/phase-signals.json` hinzu.

## Zeitplanung

Wenn aktiviert, verwaltet `memory-core` automatisch einen Cron-Job für einen vollständigen Dreaming-Durchlauf.
Jeder Durchlauf führt die Phasen der Reihe nach aus: light -> REM -> deep.

Standardverhalten für die Ausführungsfrequenz:

| Einstellung         | Standard    |
| ------------------- | ----------- |
| `dreaming.frequency` | `0 3 * * *` |

## Schnellstart

Dreaming aktivieren:

```json
{
  "plugins": {
    "entries": {
      "memory-core": {
        "config": {
          "dreaming": {
            "enabled": true
          }
        }
      }
    }
  }
}
```

Dreaming mit benutzerdefinierter Durchlauf-Frequenz aktivieren:

```json
{
  "plugins": {
    "entries": {
      "memory-core": {
        "config": {
          "dreaming": {
            "enabled": true,
            "timezone": "America/Los_Angeles",
            "frequency": "0 */6 * * *"
          }
        }
      }
    }
  }
}
```

## Slash-Befehl

```
/dreaming status
/dreaming on
/dreaming off
/dreaming help
```

## CLI-Ablauf

Verwenden Sie die CLI-Hochstufung für Vorschau oder manuelles Anwenden:

```bash
openclaw memory promote
openclaw memory promote --apply
openclaw memory promote --limit 5
openclaw memory status --deep
```

Das manuelle `memory promote` verwendet standardmäßig die Schwellenwerte der Deep-Phase, sofern sie
nicht mit CLI-Flags überschrieben werden.

Erklären, warum ein bestimmter Kandidat hochgestuft würde oder nicht:

```bash
openclaw memory promote-explain "router vlan"
openclaw memory promote-explain "router vlan" --json
```

REM-Reflexionen, Kandidatenwahrheiten und Deep-Hochstufungsausgabe in der Vorschau anzeigen, ohne
etwas zu schreiben:

```bash
openclaw memory rem-harness
openclaw memory rem-harness --json
```

## Wichtige Standardwerte

Alle Einstellungen befinden sich unter `plugins.entries.memory-core.config.dreaming`.

| Schlüssel   | Standard    |
| ----------- | ----------- |
| `enabled`   | `false`     |
| `frequency` | `0 3 * * *` |

Phasenrichtlinie, Schwellenwerte und Speicherverhalten sind interne Implementierungsdetails
(keine benutzerseitige Konfiguration).

Siehe [Referenz zur Memory-Konfiguration](/de/reference/memory-config#dreaming-experimental)
für die vollständige Schlüsselliste.

## Dreams-UI

Wenn aktiviert, zeigt der Gateway-Tab **Dreams** Folgendes an:

- aktuellen Aktivierungsstatus von Dreaming
- Status auf Phasenebene und Vorhandensein verwalteter Durchläufe
- Zählwerte für kurzfristige, fundierte, Signal- und heute hochgestufte Einträge
- Zeitpunkt des nächsten geplanten Durchlaufs
- einen eigenen fundierten Szenenpfad für bereitgestellte historische Replay-Einträge
- einen ausklappbaren Leser für das Traumtagebuch, gestützt durch `doctor.memory.dreamDiary`

## Verwandt

- [Memory](/de/concepts/memory)
- [Memory Search](/de/concepts/memory-search)
- [memory CLI](/cli/memory)
- [Referenz zur Memory-Konfiguration](/de/reference/memory-config)
