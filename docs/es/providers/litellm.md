---
read_when:
    - Quieres enrutar OpenClaw a través de un proxy de LiteLLM
    - Necesitas seguimiento de costes, logging o enrutamiento de modelos mediante LiteLLM
summary: Ejecuta OpenClaw mediante LiteLLM Proxy para acceso unificado a modelos y seguimiento de costes
title: LiteLLM
x-i18n:
    generated_at: "2026-04-05T12:51:27Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4e8ca73458186285bc06967b397b8a008791dc58eea1159d6c358e1a794982d1
    source_path: providers/litellm.md
    workflow: 15
---

# LiteLLM

[LiteLLM](https://litellm.ai) es un gateway de LLM de código abierto que proporciona una API unificada para más de 100 proveedores de modelos. Enruta OpenClaw a través de LiteLLM para obtener seguimiento centralizado de costes, logging y la flexibilidad de cambiar backends sin modificar tu configuración de OpenClaw.

## ¿Por qué usar LiteLLM con OpenClaw?

- **Seguimiento de costes** — Ve exactamente cuánto gasta OpenClaw en todos los modelos
- **Enrutamiento de modelos** — Cambia entre Claude, GPT-4, Gemini, Bedrock sin cambios de configuración
- **Claves virtuales** — Crea claves con límites de gasto para OpenClaw
- **Logging** — Logs completos de solicitudes/respuestas para depuración
- **Fallbacks** — Failover automático si tu proveedor principal no está disponible

## Inicio rápido

### Mediante onboarding

```bash
openclaw onboard --auth-choice litellm-api-key
```

### Configuración manual

1. Inicia LiteLLM Proxy:

```bash
pip install 'litellm[proxy]'
litellm --model claude-opus-4-6
```

2. Apunta OpenClaw a LiteLLM:

```bash
export LITELLM_API_KEY="your-litellm-key"

openclaw
```

Eso es todo. OpenClaw ahora enruta a través de LiteLLM.

## Configuración

### Variables de entorno

```bash
export LITELLM_API_KEY="sk-litellm-key"
```

### Archivo de configuración

```json5
{
  models: {
    providers: {
      litellm: {
        baseUrl: "http://localhost:4000",
        apiKey: "${LITELLM_API_KEY}",
        api: "openai-completions",
        models: [
          {
            id: "claude-opus-4-6",
            name: "Claude Opus 4.6",
            reasoning: true,
            input: ["text", "image"],
            contextWindow: 200000,
            maxTokens: 64000,
          },
          {
            id: "gpt-4o",
            name: "GPT-4o",
            reasoning: false,
            input: ["text", "image"],
            contextWindow: 128000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
  agents: {
    defaults: {
      model: { primary: "litellm/claude-opus-4-6" },
    },
  },
}
```

## Claves virtuales

Crea una clave dedicada para OpenClaw con límites de gasto:

```bash
curl -X POST "http://localhost:4000/key/generate" \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "key_alias": "openclaw",
    "max_budget": 50.00,
    "budget_duration": "monthly"
  }'
```

Usa la clave generada como `LITELLM_API_KEY`.

## Enrutamiento de modelos

LiteLLM puede enrutar solicitudes de modelos a distintos backends. Configúralo en tu `config.yaml` de LiteLLM:

```yaml
model_list:
  - model_name: claude-opus-4-6
    litellm_params:
      model: claude-opus-4-6
      api_key: os.environ/ANTHROPIC_API_KEY

  - model_name: gpt-4o
    litellm_params:
      model: gpt-4o
      api_key: os.environ/OPENAI_API_KEY
```

OpenClaw sigue solicitando `claude-opus-4-6`; LiteLLM se encarga del enrutamiento.

## Ver el uso

Consulta el panel de LiteLLM o la API:

```bash
# Información de la clave
curl "http://localhost:4000/key/info" \
  -H "Authorization: Bearer sk-litellm-key"

# Logs de gasto
curl "http://localhost:4000/spend/logs" \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY"
```

## Notas

- LiteLLM se ejecuta en `http://localhost:4000` por defecto
- OpenClaw se conecta a través del endpoint `/v1` compatible con OpenAI de estilo proxy de LiteLLM
- El modelado de solicitudes exclusivo de OpenAI nativo no se aplica a través de LiteLLM:
  sin `service_tier`, sin `store` de Responses, sin pistas de caché de prompts y sin
  modelado de carga útil de compatibilidad de razonamiento de OpenAI
- Las cabeceras ocultas de atribución de OpenClaw (`originator`, `version`, `User-Agent`)
  no se inyectan en URLs base personalizadas de LiteLLM

## Ver también

- [LiteLLM Docs](https://docs.litellm.ai)
- [Model Providers](/concepts/model-providers)
