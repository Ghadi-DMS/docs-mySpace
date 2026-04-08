---
read_when:
    - Quieres usar modelos de Mistral en OpenClaw
    - Necesitas onboarding con API key de Mistral y referencias de modelos
summary: Usa modelos de Mistral y transcripción Voxtral con OpenClaw
title: Mistral
x-i18n:
    generated_at: "2026-04-08T02:17:30Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4e32a0eb2a37dba6383ba338b06a8d0be600e7443aa916225794ccb0fdf46aee
    source_path: providers/mistral.md
    workflow: 15
---

# Mistral

OpenClaw es compatible con Mistral tanto para el enrutamiento de modelos de texto/imagen (`mistral/...`) como para
la transcripción de audio mediante Voxtral en la comprensión de medios.
Mistral también puede usarse para embeddings de memoria (`memorySearch.provider = "mistral"`).

## Configuración de la CLI

```bash
openclaw onboard --auth-choice mistral-api-key
# o de forma no interactiva
openclaw onboard --mistral-api-key "$MISTRAL_API_KEY"
```

## Fragmento de configuración (proveedor LLM)

```json5
{
  env: { MISTRAL_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "mistral/mistral-large-latest" } } },
}
```

## Catálogo LLM integrado

OpenClaw incluye actualmente este catálogo empaquetado de Mistral:

| Referencia de modelo             | Entrada     | Contexto | Salida máxima | Notas                                                            |
| -------------------------------- | ----------- | -------- | ------------- | ---------------------------------------------------------------- |
| `mistral/mistral-large-latest`   | text, image | 262,144  | 16,384        | Modelo predeterminado                                            |
| `mistral/mistral-medium-2508`    | text, image | 262,144  | 8,192         | Mistral Medium 3.1                                               |
| `mistral/mistral-small-latest`   | text, image | 128,000  | 16,384        | Mistral Small 4; razonamiento ajustable mediante API `reasoning_effort` |
| `mistral/pixtral-large-latest`   | text, image | 128,000  | 32,768        | Pixtral                                                          |
| `mistral/codestral-latest`       | text        | 256,000  | 4,096         | Programación                                                     |
| `mistral/devstral-medium-latest` | text        | 262,144  | 32,768        | Devstral 2                                                       |
| `mistral/magistral-small`        | text        | 128,000  | 40,000        | Razonamiento habilitado                                          |

## Fragmento de configuración (transcripción de audio con Voxtral)

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

## Razonamiento ajustable (`mistral-small-latest`)

`mistral/mistral-small-latest` se corresponde con Mistral Small 4 y admite [razonamiento ajustable](https://docs.mistral.ai/capabilities/reasoning/adjustable) en la API de Chat Completions mediante `reasoning_effort` (`none` minimiza el razonamiento adicional en la salida; `high` muestra trazas completas de razonamiento antes de la respuesta final).

OpenClaw asigna el nivel de **thinking** de la sesión a la API de Mistral:

- **off** / **minimal** → `none`
- **low** / **medium** / **high** / **xhigh** / **adaptive** → `high`

Los demás modelos del catálogo empaquetado de Mistral no usan este parámetro; sigue usando los modelos `magistral-*` cuando quieras el comportamiento nativo de Mistral orientado primero al razonamiento.

## Notas

- La autenticación de Mistral usa `MISTRAL_API_KEY`.
- La URL base del proveedor es `https://api.mistral.ai/v1` de forma predeterminada.
- El modelo predeterminado del onboarding es `mistral/mistral-large-latest`.
- El modelo de audio predeterminado de comprensión de medios para Mistral es `voxtral-mini-latest`.
- La ruta de transcripción de medios usa `/v1/audio/transcriptions`.
- La ruta de embeddings de memoria usa `/v1/embeddings` (modelo predeterminado: `mistral-embed`).
