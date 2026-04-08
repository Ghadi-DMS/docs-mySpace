---
read_when:
    - Sie möchten Mistral-Modelle in OpenClaw verwenden
    - Sie benötigen Mistral-API-Schlüssel-Onboarding und Modell-Refs
summary: Mistral-Modelle und Voxtral-Transkription mit OpenClaw verwenden
title: Mistral
x-i18n:
    generated_at: "2026-04-08T02:17:45Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4e32a0eb2a37dba6383ba338b06a8d0be600e7443aa916225794ccb0fdf46aee
    source_path: providers/mistral.md
    workflow: 15
---

# Mistral

OpenClaw unterstützt Mistral sowohl für Text-/Bild-Modellrouting (`mistral/...`) als auch
für Audiotranskription über Voxtral im Medienverständnis.
Mistral kann auch für Memory-Embeddings verwendet werden (`memorySearch.provider = "mistral"`).

## CLI-Einrichtung

```bash
openclaw onboard --auth-choice mistral-api-key
# oder nicht interaktiv
openclaw onboard --mistral-api-key "$MISTRAL_API_KEY"
```

## Konfigurations-Snippet (LLM-Anbieter)

```json5
{
  env: { MISTRAL_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "mistral/mistral-large-latest" } } },
}
```

## Integrierter LLM-Katalog

OpenClaw liefert derzeit diesen gebündelten Mistral-Katalog aus:

| Modell-Ref                       | Eingabe     | Kontext | Maximale Ausgabe | Hinweise                                                        |
| -------------------------------- | ----------- | ------- | ---------------- | ---------------------------------------------------------------- |
| `mistral/mistral-large-latest`   | text, image | 262,144 | 16,384           | Standardmodell                                                   |
| `mistral/mistral-medium-2508`    | text, image | 262,144 | 8,192            | Mistral Medium 3.1                                               |
| `mistral/mistral-small-latest`   | text, image | 128,000 | 16,384           | Mistral Small 4; anpassbares Reasoning über API-`reasoning_effort` |
| `mistral/pixtral-large-latest`   | text, image | 128,000 | 32,768           | Pixtral                                                          |
| `mistral/codestral-latest`       | text        | 256,000 | 4,096            | Coding                                                           |
| `mistral/devstral-medium-latest` | text        | 262,144 | 32,768           | Devstral 2                                                       |
| `mistral/magistral-small`        | text        | 128,000 | 40,000           | Reasoning aktiviert                                              |

## Konfigurations-Snippet (Audiotranskription mit Voxtral)

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        models: [{ provider: "mistral", model: "voxtral-mini-latest" }],
      },
    },
  },
}
```

## Anpassbares Reasoning (`mistral-small-latest`)

`mistral/mistral-small-latest` entspricht Mistral Small 4 und unterstützt [anpassbares Reasoning](https://docs.mistral.ai/capabilities/reasoning/adjustable) in der Chat-Completions-API über `reasoning_effort` (`none` minimiert zusätzliches Thinking in der Ausgabe; `high` zeigt vollständige Thinking-Traces vor der finalen Antwort an).

OpenClaw ordnet die **Thinking**-Stufe der Sitzung der API von Mistral zu:

- **off** / **minimal** → `none`
- **low** / **medium** / **high** / **xhigh** / **adaptive** → `high`

Andere Modelle im gebündelten Mistral-Katalog verwenden diesen Parameter nicht; verwenden Sie weiterhin `magistral-*`-Modelle, wenn Sie das native, auf Reasoning ausgerichtete Verhalten von Mistral möchten.

## Hinweise

- Mistral-Auth verwendet `MISTRAL_API_KEY`.
- Die Standard-Basis-URL des Anbieters ist `https://api.mistral.ai/v1`.
- Das Standardmodell im Onboarding ist `mistral/mistral-large-latest`.
- Das Standard-Audiomodell für Medienverständnis bei Mistral ist `voxtral-mini-latest`.
- Der Pfad für Medientranskription verwendet `/v1/audio/transcriptions`.
- Der Pfad für Memory-Embeddings verwendet `/v1/embeddings` (Standardmodell: `mistral-embed`).
