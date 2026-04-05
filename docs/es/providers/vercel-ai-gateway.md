---
read_when:
    - Quieres usar Vercel AI Gateway con OpenClaw
    - Necesitas la variable de entorno de API key o la opción de autenticación de la CLI
summary: Configuración de Vercel AI Gateway (autenticación + selección de modelo)
title: Vercel AI Gateway
x-i18n:
    generated_at: "2026-04-05T12:52:06Z"
    model: gpt-5.4
    provider: openai
    source_hash: f30768dc3db49708b25042d317906f7ad9a2c72b0fa03263bc04f5eefbf7a507
    source_path: providers/vercel-ai-gateway.md
    workflow: 15
---

# Vercel AI Gateway

[Vercel AI Gateway](https://vercel.com/ai-gateway) proporciona una API unificada para acceder a cientos de modelos mediante un único endpoint.

- Proveedor: `vercel-ai-gateway`
- Autenticación: `AI_GATEWAY_API_KEY`
- API: compatible con Anthropic Messages
- OpenClaw detecta automáticamente el catálogo `/v1/models` del Gateway, por lo que `/models vercel-ai-gateway`
  incluye referencias de modelo actuales como `vercel-ai-gateway/openai/gpt-5.4`.

## Inicio rápido

1. Establece la API key (recomendado: guárdala para el Gateway):

```bash
openclaw onboard --auth-choice ai-gateway-api-key
```

2. Establece un modelo predeterminado:

```json5
{
  agents: {
    defaults: {
      model: { primary: "vercel-ai-gateway/anthropic/claude-opus-4.6" },
    },
  },
}
```

## Ejemplo no interactivo

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice ai-gateway-api-key \
  --ai-gateway-api-key "$AI_GATEWAY_API_KEY"
```

## Nota sobre el entorno

Si el Gateway se ejecuta como daemon (launchd/systemd), asegúrate de que `AI_GATEWAY_API_KEY`
esté disponible para ese proceso (por ejemplo, en `~/.openclaw/.env` o mediante
`env.shellEnv`).

## Forma abreviada del ID del modelo

OpenClaw acepta referencias abreviadas de modelos Claude de Vercel y las normaliza en
runtime:

- `vercel-ai-gateway/claude-opus-4.6` -> `vercel-ai-gateway/anthropic/claude-opus-4.6`
- `vercel-ai-gateway/opus-4.6` -> `vercel-ai-gateway/anthropic/claude-opus-4-6`
