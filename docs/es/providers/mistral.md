---
read_when:
    - Quieres usar modelos Mistral en OpenClaw
    - Necesitas onboarding con API key de Mistral y referencias de modelo
summary: Usa modelos Mistral y transcripción Voxtral con OpenClaw
title: Mistral
x-i18n:
    generated_at: "2026-04-05T12:51:24Z"
    model: gpt-5.4
    provider: openai
    source_hash: 8f61b9e0656dd7e0243861ddf14b1b41a07c38bff27cef9ad0815d14c8e34408
    source_path: providers/mistral.md
    workflow: 15
---

# Mistral

OpenClaw admite Mistral tanto para enrutamiento de modelos de texto/imagen (`mistral/...`) como para
transcripción de audio mediante Voxtral en media-understanding.
Mistral también puede usarse para embeddings de memory (`memorySearch.provider = "mistral"`).

## Configuración por CLI

```bash
openclaw onboard --auth-choice mistral-api-key
# or non-interactive
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

| Referencia de modelo              | Entrada     | Contexto | Salida máxima | Notas                    |
| --------------------------------- | ----------- | -------- | ------------- | ------------------------ |
| `mistral/mistral-large-latest`    | text, image | 262,144  | 16,384        | Modelo predeterminado    |
| `mistral/mistral-medium-2508`     | text, image | 262,144  | 8,192         | Mistral Medium 3.1       |
| `mistral/mistral-small-latest`    | text, image | 128,000  | 16,384        | Modelo multimodal más pequeño |
| `mistral/pixtral-large-latest`    | text, image | 128,000  | 32,768        | Pixtral                  |
| `mistral/codestral-latest`        | text        | 256,000  | 4,096         | Programación             |
| `mistral/devstral-medium-latest`  | text        | 262,144  | 32,768        | Devstral 2               |
| `mistral/magistral-small`         | text        | 128,000  | 40,000        | Con reasoning habilitado |

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

## Notas

- La autenticación de Mistral usa `MISTRAL_API_KEY`.
- La URL base del proveedor es `https://api.mistral.ai/v1` de forma predeterminada.
- El modelo predeterminado de onboarding es `mistral/mistral-large-latest`.
- El modelo de audio predeterminado de media-understanding para Mistral es `voxtral-mini-latest`.
- La ruta de transcripción multimedia usa `/v1/audio/transcriptions`.
- La ruta de embeddings de memory usa `/v1/embeddings` (modelo predeterminado: `mistral-embed`).
