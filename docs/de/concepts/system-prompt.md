---
read_when:
    - Beim Bearbeiten von System-Prompt-Text, der Tool-Liste oder der Abschnitte zu Zeit/Heartbeat
    - Beim Ändern des Workspace-Bootstraps oder des Verhaltens der Skills-Injektion
summary: Was der OpenClaw-System-Prompt enthält und wie er zusammengesetzt wird
title: System-Prompt
x-i18n:
    generated_at: "2026-04-08T02:14:16Z"
    model: gpt-5.4
    provider: openai
    source_hash: e55fc886bc8ec47584d07c9e60dfacd964dc69c7db976ea373877dc4fe09a79a
    source_path: concepts/system-prompt.md
    workflow: 15
---

# System-Prompt

OpenClaw erstellt für jeden Agent-Lauf einen benutzerdefinierten System-Prompt. Der Prompt ist **OpenClaw-eigen** und verwendet nicht den Standard-Prompt von pi-coding-agent.

Der Prompt wird von OpenClaw zusammengesetzt und in jeden Agent-Lauf eingefügt.

Provider-Plugins können cache-fähige Prompt-Hinweise beitragen, ohne den
vollständigen OpenClaw-eigenen Prompt zu ersetzen. Die Provider-Laufzeit kann:

- eine kleine Menge benannter Kernabschnitte ersetzen (`interaction_style`,
  `tool_call_style`, `execution_bias`)
- ein **stabiles Präfix** oberhalb der Prompt-Cache-Grenze einfügen
- ein **dynamisches Suffix** unterhalb der Prompt-Cache-Grenze einfügen

Verwenden Sie provider-eigene Beiträge für modellfamilien-spezifische
Abstimmung. Behalten Sie die veraltete Prompt-Mutation
`before_prompt_build` für Kompatibilität oder wirklich globale
Prompt-Änderungen bei, nicht für normales Provider-Verhalten.

## Struktur

Der Prompt ist absichtlich kompakt und verwendet feste Abschnitte:

- **Tooling**: Erinnerung an die Single Source of Truth für strukturierte Tools sowie Laufzeit-Hinweise zur Tool-Nutzung.
- **Safety**: kurze Erinnerung an Schutzleitplanken, um machtsuchendes Verhalten oder das Umgehen von Aufsicht zu vermeiden.
- **Skills** (wenn verfügbar): erklärt dem Modell, wie es Skill-Anweisungen bei Bedarf lädt.
- **OpenClaw Self-Update**: wie die Konfiguration sicher mit
  `config.schema.lookup` geprüft, mit `config.patch` gepatcht, die vollständige
  Konfiguration mit `config.apply` ersetzt und `update.run` nur auf ausdrückliche
  Benutzeranfrage ausgeführt wird. Das nur für Eigentümer verfügbare Tool `gateway`
  verweigert außerdem das Umschreiben von
  `tools.exec.ask` / `tools.exec.security`, einschließlich veralteter Aliasse
  `tools.bash.*`, die auf diese geschützten exec-Pfade normalisiert werden.
- **Workspace**: Arbeitsverzeichnis (`agents.defaults.workspace`).
- **Documentation**: lokaler Pfad zur OpenClaw-Dokumentation (Repository oder npm-Paket) und wann sie gelesen werden soll.
- **Workspace Files (injected)**: zeigt an, dass Bootstrap-Dateien unten eingefügt sind.
- **Sandbox** (wenn aktiviert): zeigt sandboxierte Laufzeit, Sandbox-Pfade und an, ob exec mit erhöhten Rechten verfügbar ist.
- **Current Date & Time**: benutzerlokale Zeit, Zeitzone und Zeitformat.
- **Reply Tags**: optionale Reply-Tag-Syntax für unterstützte Provider.
- **Heartbeats**: Heartbeat-Prompt und Bestätigungsverhalten, wenn Heartbeats für den Standard-Agent aktiviert sind.
- **Runtime**: Host, OS, Node, Modell, Repo-Root (wenn erkannt), Thinking-Level (eine Zeile).
- **Reasoning**: aktuelle Sichtbarkeitsstufe + Hinweis zum Umschalten mit /reasoning.

Der Abschnitt Tooling enthält außerdem Laufzeit-Hinweise für langlaufende Arbeit:

- verwenden Sie cron für spätere Nachverfolgung (`check back later`, Erinnerungen, wiederkehrende Arbeit)
  statt `exec`-Sleep-Schleifen, `yieldMs`-Verzögerungstricks oder wiederholtem
  `process`-Polling
- verwenden Sie `exec` / `process` nur für Befehle, die jetzt starten und im
  Hintergrund weiterlaufen
- wenn das automatische Abschluss-Aufwecken aktiviert ist, starten Sie den
  Befehl einmal und verlassen Sie sich auf den Push-basierten Aufweckpfad, wenn
  er Ausgabe erzeugt oder fehlschlägt
- verwenden Sie `process` für Logs, Status, Eingaben oder Eingriffe, wenn Sie
  einen laufenden Befehl prüfen müssen
- wenn die Aufgabe größer ist, bevorzugen Sie `sessions_spawn`; der Abschluss
  von Unter-Agenten ist Push-basiert und wird automatisch an den Anfragenden
  zurückgemeldet
- pollen Sie `subagents list` / `sessions_list` nicht in einer Schleife, nur um
  auf den Abschluss zu warten

Wenn das experimentelle Tool `update_plan` aktiviert ist, weist Tooling das
Modell außerdem an, es nur für nicht triviale Arbeit mit mehreren Schritten zu
verwenden, genau einen Schritt mit `in_progress` beizubehalten und nicht nach
jeder Aktualisierung den gesamten Plan zu wiederholen.

Die Safety-Leitplanken im System-Prompt sind beratend. Sie steuern das Modellverhalten, setzen jedoch keine Richtlinien durch. Verwenden Sie Tool-Richtlinien, exec-Genehmigungen, Sandboxing und Kanal-Allowlists für harte Durchsetzung; Betreiber können diese absichtlich deaktivieren.

Auf Kanälen mit nativen Genehmigungskarten/-schaltflächen weist der Laufzeit-Prompt den
Agenten jetzt an, sich zuerst auf diese native Genehmigungs-UI zu verlassen. Er
soll nur dann einen manuellen Befehl `/approve` einfügen, wenn das Tool-Ergebnis
angibt, dass Chat-Genehmigungen nicht verfügbar sind oder eine manuelle
Genehmigung der einzige Weg ist.

## Prompt-Modi

OpenClaw kann für Unter-Agenten kleinere System-Prompts rendern. Die Laufzeit setzt für jeden
Lauf einen `promptMode` (keine benutzerseitige Konfiguration):

- `full` (Standard): enthält alle obigen Abschnitte.
- `minimal`: wird für Unter-Agenten verwendet; lässt **Skills**, **Memory Recall**, **OpenClaw
  Self-Update**, **Model Aliases**, **User Identity**, **Reply Tags**,
  **Messaging**, **Silent Replies** und **Heartbeats** weg. Tooling, **Safety**,
  Workspace, Sandbox, Current Date & Time (wenn bekannt), Runtime und eingefügter
  Kontext bleiben verfügbar.
- `none`: gibt nur die grundlegende Identitätszeile zurück.

Wenn `promptMode=minimal`, werden zusätzlich eingefügte Prompts als **Subagent
Context** statt als **Group Chat Context** bezeichnet.

## Workspace-Bootstrap-Injektion

Bootstrap-Dateien werden gekürzt und unter **Project Context** angehängt, damit das Modell Identitäts- und Profilkontext sieht, ohne sie explizit lesen zu müssen:

- `AGENTS.md`
- `SOUL.md`
- `TOOLS.md`
- `IDENTITY.md`
- `USER.md`
- `HEARTBEAT.md`
- `BOOTSTRAP.md` (nur in ganz neuen Workspaces)
- `MEMORY.md`, falls vorhanden, andernfalls `memory.md` als Fallback in Kleinbuchstaben

Alle diese Dateien werden bei jeder Runde **in das Kontextfenster eingefügt**, sofern
keine dateispezifische Bedingung gilt. `HEARTBEAT.md` wird bei normalen Läufen
weggelassen, wenn Heartbeats für den Standard-Agent deaktiviert sind oder
`agents.defaults.heartbeat.includeSystemPromptSection` false ist. Halten Sie
eingefügte Dateien kurz — insbesondere `MEMORY.md`, das im Laufe der Zeit wachsen
kann und zu unerwartet hoher Kontextnutzung und häufigerer Kompaktierung führen kann.

> **Hinweis:** tägliche Dateien `memory/*.md` werden **nicht** automatisch eingefügt. Sie
> werden bei Bedarf über die Tools `memory_search` und `memory_get` aufgerufen, sodass sie
> nicht auf das Kontextfenster angerechnet werden, es sei denn, das Modell liest sie ausdrücklich.

Große Dateien werden mit einer Markierung gekürzt. Die maximale Größe pro Datei wird durch
`agents.defaults.bootstrapMaxChars` gesteuert (Standard: 20000). Der gesamte eingefügte Bootstrap-
Inhalt über alle Dateien hinweg ist durch `agents.defaults.bootstrapTotalMaxChars`
begrenzt (Standard: 150000). Fehlende Dateien fügen eine kurze Markierung für fehlende Dateien ein. Wenn Kürzung
auftritt, kann OpenClaw einen Warnblock in Project Context einfügen; steuern Sie dies mit
`agents.defaults.bootstrapPromptTruncationWarning` (`off`, `once`, `always`;
Standard: `once`).

Unter-Agent-Sitzungen fügen nur `AGENTS.md` und `TOOLS.md` ein (andere Bootstrap-Dateien
werden herausgefiltert, um den Unter-Agent-Kontext klein zu halten).

Interne Hooks können diesen Schritt über `agent:bootstrap` abfangen, um die
eingefügten Bootstrap-Dateien zu verändern oder zu ersetzen (zum Beispiel `SOUL.md` durch
eine alternative Persona auszutauschen).

Wenn Sie möchten, dass der Agent weniger generisch klingt, beginnen Sie mit dem
[SOUL.md Personality Guide](/de/concepts/soul).

Um zu prüfen, wie viel jede eingefügte Datei beiträgt (roh vs. eingefügt, Kürzung sowie Tool-Schema-Overhead), verwenden Sie `/context list` oder `/context detail`. Siehe [Context](/de/concepts/context).

## Umgang mit Zeit

Der System-Prompt enthält einen eigenen Abschnitt **Current Date & Time**, wenn die
Benutzerzeitzone bekannt ist. Um den Prompt cache-stabil zu halten, enthält er jetzt nur noch
die **Zeitzone** (keine dynamische Uhrzeit oder kein Zeitformat).

Verwenden Sie `session_status`, wenn der Agent die aktuelle Uhrzeit benötigt; die Statuskarte
enthält eine Zeitstempelzeile. Dasselbe Tool kann optional ein modellspezifisches
Override pro Sitzung setzen (`model=default` löscht es).

Konfiguration mit:

- `agents.defaults.userTimezone`
- `agents.defaults.timeFormat` (`auto` | `12` | `24`)

Siehe [Date & Time](/de/date-time) für vollständige Verhaltensdetails.

## Skills

Wenn geeignete Skills vorhanden sind, fügt OpenClaw eine kompakte **Liste verfügbarer Skills**
(`formatSkillsForPrompt`) ein, die den **Dateipfad** für jeden Skill enthält. Der
Prompt weist das Modell an, `read` zu verwenden, um die SKILL.md am aufgeführten
Speicherort zu laden (Workspace, verwaltet oder gebündelt). Wenn keine geeigneten Skills vorhanden sind, wird der
Skills-Abschnitt weggelassen.

Die Eignung umfasst Skill-Metadaten-Gates, Laufzeit-Umgebungs-/Konfigurationsprüfungen
und die effektive Allowlist für Agent-Skills, wenn `agents.defaults.skills` oder
`agents.list[].skills` konfiguriert ist.

```
<available_skills>
  <skill>
    <name>...</name>
    <description>...</description>
    <location>...</location>
  </skill>
</available_skills>
```

Dadurch bleibt der Basis-Prompt klein und ermöglicht dennoch die gezielte Nutzung von Skills.

## Documentation

Wenn verfügbar, enthält der System-Prompt einen Abschnitt **Documentation**, der auf das
lokale OpenClaw-Dokumentationsverzeichnis verweist (entweder `docs/` im Repo-Workspace oder die gebündelte npm-
Paketdokumentation) und außerdem auf den öffentlichen Mirror, das Quell-Repository, die Community auf Discord und
ClawHub ([https://clawhub.ai](https://clawhub.ai)) zur Skills-Entdeckung hinweist. Der Prompt weist das Modell an, zuerst die lokale Dokumentation zu konsultieren
für OpenClaw-Verhalten, Befehle, Konfiguration oder Architektur und
`openclaw status` nach Möglichkeit selbst auszuführen (den Benutzer nur dann zu fragen, wenn kein Zugriff besteht).
