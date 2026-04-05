---
read_when:
    - Quieres ejecutar OpenClaw contra un servidor SGLang local
    - Quieres endpoints `/v1` compatibles con OpenAI con tus propios modelos
summary: Ejecuta OpenClaw con SGLang (servidor autoalojado compatible con OpenAI)
title: SGLang
x-i18n:
    generated_at: "2026-04-05T12:51:54Z"
    model: gpt-5.4
    provider: openai
    source_hash: 9850277c6c5e318e60237688b4d8a5b1387d4e9586534ae2eb6ad953abba8948
    source_path: providers/sglang.md
    workflow: 15
---

# SGLang

SGLang puede servir modelos de código abierto mediante una API HTTP **compatible con OpenAI**.
OpenClaw puede conectarse a SGLang usando la API `openai-completions`.

OpenClaw también puede **detectar automáticamente** los modelos disponibles de SGLang cuando optas
por ello con `SGLANG_API_KEY` (cualquier valor funciona si tu servidor no exige autenticación)
y no defines una entrada explícita `models.providers.sglang`.

## Inicio rápido

1. Inicia SGLang con un servidor compatible con OpenAI.

Tu URL base debe exponer endpoints `/v1` (por ejemplo `/v1/models`,
`/v1/chat/completions`). SGLang suele ejecutarse en:

- `http://127.0.0.1:30000/v1`

2. Habilítalo (cualquier valor funciona si no hay autenticación configurada):

```bash
export SGLANG_API_KEY="sglang-local"
```

3. Ejecuta onboarding y elige `SGLang`, o configura un modelo directamente:

```bash
openclaw onboard
```

```json5
{
  agents: {
    defaults: {
      model: { primary: "sglang/your-model-id" },
    },
  },
}
```

## Detección de modelos (proveedor implícito)

Cuando `SGLANG_API_KEY` está configurado (o existe un perfil de autenticación) y **no**
defines `models.providers.sglang`, OpenClaw consultará:

- `GET http://127.0.0.1:30000/v1/models`

y convertirá los IDs devueltos en entradas de modelo.

Si configuras `models.providers.sglang` explícitamente, la detección automática se omite y
debes definir los modelos manualmente.

## Configuración explícita (modelos manuales)

Usa configuración explícita cuando:

- SGLang se ejecuta en un host o puerto diferente.
- Quieres fijar valores de `contextWindow`/`maxTokens`.
- Tu servidor requiere una clave de API real (o quieres controlar las cabeceras).

```json5
{
  models: {
    providers: {
      sglang: {
        baseUrl: "http://127.0.0.1:30000/v1",
        apiKey: "${SGLANG_API_KEY}",
        api: "openai-completions",
        models: [
          {
            id: "your-model-id",
            name: "Modelo SGLang local",
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

- Comprueba que se pueda acceder al servidor:

```bash
curl http://127.0.0.1:30000/v1/models
```

- Si las solicitudes fallan con errores de autenticación, configura un `SGLANG_API_KEY` real que coincida
  con la configuración de tu servidor, o configura el proveedor explícitamente en
  `models.providers.sglang`.

## Comportamiento de estilo proxy

SGLang se trata como un backend `/v1` compatible con OpenAI de estilo proxy, no como un
endpoint nativo de OpenAI.

- aquí no se aplica el modelado de solicitudes exclusivo de OpenAI nativo
- no hay `service_tier`, ni `store` de Responses, ni pistas de caché de prompt, ni
  modelado de cargas útiles de compatibilidad de razonamiento de OpenAI
- las cabeceras ocultas de atribución de OpenClaw (`originator`, `version`, `User-Agent`)
  no se inyectan en URLs base personalizadas de SGLang
