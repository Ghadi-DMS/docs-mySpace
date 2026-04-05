---
read_when:
    - Quieres usar Cloudflare AI Gateway con OpenClaw
    - Necesitas el ID de cuenta, el ID del gateway o la variable de entorno de la clave de API
summary: Configuración de Cloudflare AI Gateway (autenticación + selección de modelo)
title: Cloudflare AI Gateway
x-i18n:
    generated_at: "2026-04-05T12:50:56Z"
    model: gpt-5.4
    provider: openai
    source_hash: db77652c37652ca20f7c50f32382dbaeaeb50ea5bdeaf1d4fd17dc394e58950c
    source_path: providers/cloudflare-ai-gateway.md
    workflow: 15
---

# Cloudflare AI Gateway

Cloudflare AI Gateway se sitúa delante de las API de los proveedores y te permite añadir analíticas, caché y controles. Para Anthropic, OpenClaw usa la Anthropic Messages API a través del endpoint de tu Gateway.

- Proveedor: `cloudflare-ai-gateway`
- URL base: `https://gateway.ai.cloudflare.com/v1/<account_id>/<gateway_id>/anthropic`
- Modelo predeterminado: `cloudflare-ai-gateway/claude-sonnet-4-5`
- Clave de API: `CLOUDFLARE_AI_GATEWAY_API_KEY` (tu clave de API del proveedor para solicitudes a través del Gateway)

Para modelos de Anthropic, usa tu clave de API de Anthropic.

## Inicio rápido

1. Configura la clave de API del proveedor y los detalles del Gateway:

```bash
openclaw onboard --auth-choice cloudflare-ai-gateway-api-key
```

2. Configura un modelo predeterminado:

```json5
{
  agents: {
    defaults: {
      model: { primary: "cloudflare-ai-gateway/claude-sonnet-4-5" },
    },
  },
}
```

## Ejemplo no interactivo

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice cloudflare-ai-gateway-api-key \
  --cloudflare-ai-gateway-account-id "your-account-id" \
  --cloudflare-ai-gateway-gateway-id "your-gateway-id" \
  --cloudflare-ai-gateway-api-key "$CLOUDFLARE_AI_GATEWAY_API_KEY"
```

## Gateways autenticados

Si habilitaste la autenticación del Gateway en Cloudflare, añade la cabecera `cf-aig-authorization` (esto es adicional a tu clave de API del proveedor).

```json5
{
  models: {
    providers: {
      "cloudflare-ai-gateway": {
        headers: {
          "cf-aig-authorization": "Bearer <cloudflare-ai-gateway-token>",
        },
      },
    },
  },
}
```

## Nota sobre el entorno

Si el Gateway se ejecuta como daemon (launchd/systemd), asegúrate de que `CLOUDFLARE_AI_GATEWAY_API_KEY` esté disponible para ese proceso (por ejemplo, en `~/.openclaw/.env` o mediante `env.shellEnv`).
