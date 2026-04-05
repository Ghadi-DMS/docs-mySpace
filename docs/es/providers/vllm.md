---
read_when:
    - Quieres ejecutar OpenClaw contra un servidor local vLLM
    - Quieres endpoints `/v1` compatibles con OpenAI con tus propios modelos
summary: Ejecutar OpenClaw con vLLM (servidor local compatible con OpenAI)
title: vLLM
x-i18n:
    generated_at: "2026-04-05T12:52:13Z"
    model: gpt-5.4
    provider: openai
    source_hash: ebde34d0453586d10340680b8d51465fdc98bd28e8a96acfaeb24606886b50f4
    source_path: providers/vllm.md
    workflow: 15
---

# vLLM

vLLM puede servir modelos de código abierto (y algunos personalizados) mediante una API HTTP **compatible con OpenAI**. OpenClaw puede conectarse a vLLM usando la API `openai-completions`.

OpenClaw también puede **descubrir automáticamente** los modelos disponibles de vLLM cuando optas por ello con `VLLM_API_KEY` (cualquier valor sirve si tu servidor no impone autenticación) y no defines una entrada explícita `models.providers.vllm`.

## Inicio rápido

1. Inicia vLLM con un servidor compatible con OpenAI.

Tu URL base debe exponer endpoints `/v1` (por ejemplo `/v1/models`, `/v1/chat/completions`). vLLM suele ejecutarse en:

- `http://127.0.0.1:8000/v1`

2. Activa la opción (cualquier valor sirve si no hay autenticación configurada):

```bash
export VLLM_API_KEY="vllm-local"
```

3. Selecciona un modelo (sustitúyelo por uno de los ids de modelo de tu vLLM):

```json5
{
  agents: {
    defaults: {
      model: { primary: "vllm/your-model-id" },
    },
  },
}
```

## Descubrimiento de modelos (proveedor implícito)

Cuando `VLLM_API_KEY` está definido (o existe un perfil de autenticación) y **no** defines `models.providers.vllm`, OpenClaw consultará:

- `GET http://127.0.0.1:8000/v1/models`

...y convertirá los ids devueltos en entradas de modelo.

Si defines `models.providers.vllm` explícitamente, se omite el descubrimiento automático y debes definir los modelos manualmente.

## Configuración explícita (modelos manuales)

Usa configuración explícita cuando:

- vLLM se ejecuta en otro host/puerto.
- Quieres fijar valores `contextWindow`/`maxTokens`.
- Tu servidor requiere una clave API real (o quieres controlar los encabezados).

```json5
{
  models: {
    providers: {
      vllm: {
        baseUrl: "http://127.0.0.1:8000/v1",
        apiKey: "${VLLM_API_KEY}",
        api: "openai-completions",
        models: [
          {
            id: "your-model-id",
            name: "Local vLLM Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 128000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

## Solución de problemas

- Comprueba que el servidor sea accesible:

```bash
curl http://127.0.0.1:8000/v1/models
```

- Si las solicitudes fallan con errores de autenticación, establece un `VLLM_API_KEY` real que coincida con la configuración de tu servidor, o configura el proveedor explícitamente en `models.providers.vllm`.

## Comportamiento estilo proxy

vLLM se trata como un backend `/v1` compatible con OpenAI de estilo proxy, no como un endpoint nativo
de OpenAI.

- la conformación de solicitudes nativa solo de OpenAI no se aplica aquí
- no hay `service_tier`, ni `store` de Responses, ni sugerencias de caché de prompts, ni
  conformación de carga útil de compatibilidad de razonamiento de OpenAI
- los encabezados ocultos de atribución de OpenClaw (`originator`, `version`, `User-Agent`)
  no se inyectan en URLs base personalizadas de vLLM
