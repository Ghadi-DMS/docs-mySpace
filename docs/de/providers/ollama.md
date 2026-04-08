---
read_when:
    - OpenClaw soll mit Cloud- oder lokalen Modellen über Ollama ausgeführt werden
    - Anleitung für Einrichtung und Konfiguration von Ollama wird benötigt
summary: OpenClaw mit Ollama ausführen (Cloud- und lokale Modelle)
title: Ollama
x-i18n:
    generated_at: "2026-04-08T02:18:26Z"
    model: gpt-5.4
    provider: openai
    source_hash: 222ec68f7d4bb29cc7796559ddef1d5059f5159e7a51e2baa3a271ddb3abb716
    source_path: providers/ollama.md
    workflow: 15
---

# Ollama

Ollama ist eine lokale LLM-Runtime, mit der sich Open-Source-Modelle einfach auf deinem Rechner ausführen lassen. OpenClaw integriert sich mit der nativen API von Ollama (`/api/chat`), unterstützt Streaming und Tool-Aufrufe und kann lokale Ollama-Modelle automatisch erkennen, wenn du dies mit `OLLAMA_API_KEY` (oder einem Auth-Profil) aktivierst und keinen expliziten Eintrag `models.providers.ollama` definierst.

<Warning>
**Remote-Ollama-Benutzer**: Verwende die OpenAI-kompatible URL `/v1` (`http://host:11434/v1`) nicht mit OpenClaw. Dadurch funktionieren Tool-Aufrufe nicht mehr korrekt, und Modelle können rohes Tool-JSON als Klartext ausgeben. Verwende stattdessen die native Ollama-API-URL: `baseUrl: "http://host:11434"` (ohne `/v1`).
</Warning>

## Schnellstart

### Onboarding (empfohlen)

Der schnellste Weg, Ollama einzurichten, ist über das Onboarding:

```bash
openclaw onboard
```

Wähle **Ollama** aus der Provider-Liste. Das Onboarding wird:

1. Nach der Ollama-Base-URL fragen, unter der deine Instanz erreichbar ist (Standard `http://127.0.0.1:11434`).
2. Zwischen **Cloud + Local** (Cloud-Modelle und lokale Modelle) oder **Local** (nur lokale Modelle) wählen lassen.
3. Einen Browser-Anmeldefluss öffnen, wenn du **Cloud + Local** auswählst und nicht bei ollama.com angemeldet bist.
4. Verfügbare Modelle erkennen und Standards vorschlagen.
5. Das ausgewählte Modell automatisch pullen, wenn es lokal nicht verfügbar ist.

Der nicht interaktive Modus wird ebenfalls unterstützt:

```bash
openclaw onboard --non-interactive \
  --auth-choice ollama \
  --accept-risk
```

Optional kann eine benutzerdefinierte Base-URL oder ein Modell angegeben werden:

```bash
openclaw onboard --non-interactive \
  --auth-choice ollama \
  --custom-base-url "http://ollama-host:11434" \
  --custom-model-id "qwen3.5:27b" \
  --accept-risk
```

### Manuelle Einrichtung

1. Ollama installieren: [https://ollama.com/download](https://ollama.com/download)

2. Ein lokales Modell pullen, wenn du lokale Inferenz verwenden möchtest:

```bash
ollama pull gemma4
# oder
ollama pull gpt-oss:20b
# oder
ollama pull llama3.3
```

3. Wenn du auch Cloud-Modelle möchtest, melde dich an:

```bash
ollama signin
```

4. Das Onboarding ausführen und `Ollama` wählen:

```bash
openclaw onboard
```

- `Local`: nur lokale Modelle
- `Cloud + Local`: lokale Modelle plus Cloud-Modelle
- Cloud-Modelle wie `kimi-k2.5:cloud`, `minimax-m2.7:cloud` und `glm-5.1:cloud` erfordern **kein** lokales `ollama pull`

OpenClaw schlägt derzeit vor:

- lokaler Standard: `gemma4`
- Cloud-Standards: `kimi-k2.5:cloud`, `minimax-m2.7:cloud`, `glm-5.1:cloud`

5. Wenn du die manuelle Einrichtung bevorzugst, aktiviere Ollama direkt für OpenClaw (jeder Wert funktioniert; Ollama benötigt keinen echten Schlüssel):

```bash
# Umgebungsvariable setzen
export OLLAMA_API_KEY="ollama-local"

# Oder in deiner Konfigurationsdatei konfigurieren
openclaw config set models.providers.ollama.apiKey "ollama-local"
```

6. Modelle prüfen oder wechseln:

```bash
openclaw models list
openclaw models set ollama/gemma4
```

7. Oder den Standard in der Konfiguration festlegen:

```json5
{
  agents: {
    defaults: {
      model: { primary: "ollama/gemma4" },
    },
  },
}
```

## Modellerkennung (impliziter Provider)

Wenn du `OLLAMA_API_KEY` (oder ein Auth-Profil) setzt und **nicht** `models.providers.ollama` definierst, erkennt OpenClaw Modelle von der lokalen Ollama-Instanz unter `http://127.0.0.1:11434`:

- Fragt `/api/tags` ab
- Verwendet best-effort-`/api/show`-Lookups, um `contextWindow` zu lesen, wenn verfügbar
- Markiert `reasoning` mit einer Heuristik anhand des Modellnamens (`r1`, `reasoning`, `think`)
- Setzt `maxTokens` auf das Standard-Max-Token-Limit von Ollama, das von OpenClaw verwendet wird
- Setzt alle Kosten auf `0`

Dadurch werden manuelle Modelleinträge vermieden, während der Katalog mit der lokalen Ollama-Instanz abgestimmt bleibt.

So siehst du, welche Modelle verfügbar sind:

```bash
ollama list
openclaw models list
```

Um ein neues Modell hinzuzufügen, pulle es einfach mit Ollama:

```bash
ollama pull mistral
```

Das neue Modell wird automatisch erkannt und kann verwendet werden.

Wenn du `models.providers.ollama` explizit setzt, wird die automatische Erkennung übersprungen, und du musst Modelle manuell definieren (siehe unten).

## Konfiguration

### Grundeinrichtung (implizite Erkennung)

Der einfachste Weg, Ollama zu aktivieren, ist per Umgebungsvariable:

```bash
export OLLAMA_API_KEY="ollama-local"
```

### Explizite Einrichtung (manuelle Modelle)

Verwende eine explizite Konfiguration, wenn:

- Ollama auf einem anderen Host/Port läuft.
- Du bestimmte Kontextfenster oder Modelllisten erzwingen möchtest.
- Du vollständig manuelle Modelldefinitionen möchtest.

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "http://ollama-host:11434",
        apiKey: "ollama-local",
        api: "ollama",
        models: [
          {
            id: "gpt-oss:20b",
            name: "GPT-OSS 20B",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 8192,
            maxTokens: 8192 * 10
          }
        ]
      }
    }
  }
}
```

Wenn `OLLAMA_API_KEY` gesetzt ist, kannst du `apiKey` im Provider-Eintrag weglassen, und OpenClaw füllt ihn für Verfügbarkeitsprüfungen aus.

### Benutzerdefinierte Base-URL (explizite Konfiguration)

Wenn Ollama auf einem anderen Host oder Port läuft (explizite Konfiguration deaktiviert die automatische Erkennung, daher Modelle manuell definieren):

```json5
{
  models: {
    providers: {
      ollama: {
        apiKey: "ollama-local",
        baseUrl: "http://ollama-host:11434", // No /v1 - use native Ollama API URL
        api: "ollama", // Set explicitly to guarantee native tool-calling behavior
      },
    },
  },
}
```

<Warning>
Füge der URL kein `/v1` hinzu. Der Pfad `/v1` verwendet den OpenAI-kompatiblen Modus, in dem Tool-Aufrufe nicht zuverlässig funktionieren. Verwende die Basis-URL von Ollama ohne Pfadsuffix.
</Warning>

### Modellauswahl

Sobald die Konfiguration eingerichtet ist, sind alle deine Ollama-Modelle verfügbar:

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "ollama/gpt-oss:20b",
        fallbacks: ["ollama/llama3.3", "ollama/qwen2.5-coder:32b"],
      },
    },
  },
}
```

## Cloud-Modelle

Cloud-Modelle ermöglichen es dir, in der Cloud gehostete Modelle (zum Beispiel `kimi-k2.5:cloud`, `minimax-m2.7:cloud`, `glm-5.1:cloud`) zusammen mit deinen lokalen Modellen auszuführen.

Um Cloud-Modelle zu verwenden, wähle während der Einrichtung den Modus **Cloud + Local**. Der Wizard prüft, ob du angemeldet bist, und öffnet bei Bedarf einen Browser-Anmeldefluss. Wenn die Authentifizierung nicht verifiziert werden kann, fällt der Wizard auf Standardwerte für lokale Modelle zurück.

Du kannst dich auch direkt unter [ollama.com/signin](https://ollama.com/signin) anmelden.

## Ollama Web Search

OpenClaw unterstützt auch **Ollama Web Search** als gebündelten `web_search`-
Provider.

- Verwendet deinen konfigurierten Ollama-Host (`models.providers.ollama.baseUrl`, wenn
  gesetzt, andernfalls `http://127.0.0.1:11434`).
- Benötigt keinen Schlüssel.
- Erfordert, dass Ollama läuft und du mit `ollama signin` angemeldet bist.

Wähle **Ollama Web Search** während `openclaw onboard` oder
`openclaw configure --section web`, oder setze:

```json5
{
  tools: {
    web: {
      search: {
        provider: "ollama",
      },
    },
  },
}
```

Die vollständigen Details zu Einrichtung und Verhalten findest du unter [Ollama Web Search](/de/tools/ollama-search).

## Erweitert

### Reasoning-Modelle

OpenClaw behandelt Modelle mit Namen wie `deepseek-r1`, `reasoning` oder `think` standardmäßig als reasoning-fähig:

```bash
ollama pull deepseek-r1:32b
```

### Modellkosten

Ollama ist kostenlos und läuft lokal, daher sind alle Modellkosten auf $0 gesetzt.

### Streaming-Konfiguration

Die Ollama-Integration von OpenClaw verwendet standardmäßig die **native Ollama-API** (`/api/chat`), die Streaming und Tool-Aufrufe gleichzeitig vollständig unterstützt. Es ist keine besondere Konfiguration erforderlich.

#### Legacy OpenAI-Compatible Mode

<Warning>
**Tool-Aufrufe sind im OpenAI-kompatiblen Modus nicht zuverlässig.** Verwende diesen Modus nur, wenn du das OpenAI-Format für einen Proxy benötigst und nicht auf natives Tool-Calling-Verhalten angewiesen bist.
</Warning>

Wenn du stattdessen den OpenAI-kompatiblen Endpunkt verwenden musst (z. B. hinter einem Proxy, der nur das OpenAI-Format unterstützt), setze explizit `api: "openai-completions"`:

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "http://ollama-host:11434/v1",
        api: "openai-completions",
        injectNumCtxForOpenAICompat: true, // default: true
        apiKey: "ollama-local",
        models: [...]
      }
    }
  }
}
```

Dieser Modus unterstützt möglicherweise Streaming + Tool-Aufrufe nicht gleichzeitig. Möglicherweise musst du Streaming mit `params: { streaming: false }` in der Modellkonfiguration deaktivieren.

Wenn `api: "openai-completions"` mit Ollama verwendet wird, injiziert OpenClaw standardmäßig `options.num_ctx`, damit Ollama nicht stillschweigend auf ein Kontextfenster von 4096 zurückfällt. Wenn dein Proxy/Upstream unbekannte `options`-Felder ablehnt, deaktiviere dieses Verhalten:

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "http://ollama-host:11434/v1",
        api: "openai-completions",
        injectNumCtxForOpenAICompat: false,
        apiKey: "ollama-local",
        models: [...]
      }
    }
  }
}
```

### Kontextfenster

Für automatisch erkannte Modelle verwendet OpenClaw das von Ollama gemeldete Kontextfenster, wenn verfügbar, andernfalls das von OpenClaw verwendete Standard-Kontextfenster für Ollama. Du kannst `contextWindow` und `maxTokens` in der expliziten Provider-Konfiguration überschreiben.

## Fehlerbehebung

### Ollama wird nicht erkannt

Stelle sicher, dass Ollama läuft, dass du `OLLAMA_API_KEY` (oder ein Auth-Profil) gesetzt hast und dass du **keinen** expliziten Eintrag `models.providers.ollama` definiert hast:

```bash
ollama serve
```

Und dass die API erreichbar ist:

```bash
curl http://localhost:11434/api/tags
```

### Keine Modelle verfügbar

Wenn dein Modell nicht aufgelistet ist, musst du entweder:

- das Modell lokal pullen oder
- das Modell explizit in `models.providers.ollama` definieren.

So fügst du Modelle hinzu:

```bash
ollama list  # See what's installed
ollama pull gemma4
ollama pull gpt-oss:20b
ollama pull llama3.3     # Or another model
```

### Verbindung abgelehnt

Prüfe, ob Ollama auf dem richtigen Port läuft:

```bash
# Check if Ollama is running
ps aux | grep ollama

# Or restart Ollama
ollama serve
```

## Siehe auch

- [Modell-Provider](/de/concepts/model-providers) - Überblick über alle Provider
- [Modellauswahl](/de/concepts/models) - So wählst du Modelle aus
- [Konfiguration](/de/gateway/configuration) - Vollständige Konfigurationsreferenz
