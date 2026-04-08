---
read_when:
    - Quieres ejecutar OpenClaw contra un servidor local de inferrs
    - Estás sirviendo Gemma u otro modelo mediante inferrs
    - Necesitas las marcas de compatibilidad exactas de OpenClaw para inferrs
summary: Ejecuta OpenClaw mediante inferrs (servidor local compatible con OpenAI)
title: inferrs
x-i18n:
    generated_at: "2026-04-08T02:17:25Z"
    model: gpt-5.4
    provider: openai
    source_hash: d84f660d49a682d0c0878707eebe1bc1e83dd115850687076ea3938b9f9c86c6
    source_path: providers/inferrs.md
    workflow: 15
---

# inferrs

[inferrs](https://github.com/ericcurtin/inferrs) puede servir modelos locales detrás de una
API `/v1` compatible con OpenAI. OpenClaw funciona con `inferrs` mediante la ruta genérica
`openai-completions`.

Actualmente, lo mejor es tratar `inferrs` como un backend personalizado autoalojado
compatible con OpenAI, no como un plugin de proveedor dedicado de OpenClaw.

## Inicio rápido

1. Inicia `inferrs` con un modelo.

Ejemplo:

```bash
inferrs serve gg-hf-gg/gemma-4-E2B-it \
  --host 127.0.0.1 \
  --port 8080 \
  --device metal
```

2. Verifica que se pueda acceder al servidor.

```bash
curl http://127.0.0.1:8080/health
curl http://127.0.0.1:8080/v1/models
```

3. Añade una entrada explícita de proveedor de OpenClaw y apunta tu modelo predeterminado a ella.

## Ejemplo completo de configuración

Este ejemplo usa Gemma 4 en un servidor local de `inferrs`.

```json5
{
  agents: {
    defaults: {
      model: { primary: "inferrs/gg-hf-gg/gemma-4-E2B-it" },
      models: {
        "inferrs/gg-hf-gg/gemma-4-E2B-it": {
          alias: "Gemma 4 (inferrs)",
        },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      inferrs: {
        baseUrl: "http://127.0.0.1:8080/v1",
        apiKey: "inferrs-local",
        api: "openai-completions",
        models: [
          {
            id: "gg-hf-gg/gemma-4-E2B-it",
            name: "Gemma 4 E2B (inferrs)",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 131072,
            maxTokens: 4096,
            compat: {
              requiresStringContent: true,
            },
          },
        ],
      },
    },
  },
}
```

## Por qué importa `requiresStringContent`

Algunas rutas Chat Completions de `inferrs` aceptan solo
`messages[].content` de tipo cadena, no matrices estructuradas de partes de contenido.

Si las ejecuciones de OpenClaw fallan con un error como:

```text
messages[1].content: invalid type: sequence, expected a string
```

establece:

```json5
compat: {
  requiresStringContent: true
}
```

OpenClaw aplanará las partes de contenido de texto puro en cadenas simples antes de enviar
la solicitud.

## Advertencia sobre Gemma y el esquema de herramientas

Algunas combinaciones actuales de `inferrs` + Gemma aceptan solicitudes directas pequeñas a
`/v1/chat/completions`, pero aun así fallan en turnos completos del entorno de ejecución de agentes de OpenClaw.

Si eso ocurre, prueba primero esto:

```json5
compat: {
  requiresStringContent: true,
  supportsTools: false
}
```

Eso desactiva la superficie de esquema de herramientas de OpenClaw para el modelo y puede reducir la
presión del prompt en backends locales más estrictos.

Si las solicitudes directas pequeñas siguen funcionando, pero los turnos normales de agentes de OpenClaw continúan
fallando dentro de `inferrs`, el problema restante suele estar en el comportamiento ascendente del modelo/servidor
y no en la capa de transporte de OpenClaw.

## Prueba manual rápida

Una vez configurado, prueba ambas capas:

```bash
curl http://127.0.0.1:8080/v1/chat/completions \
  -H 'content-type: application/json' \
  -d '{"model":"gg-hf-gg/gemma-4-E2B-it","messages":[{"role":"user","content":"What is 2 + 2?"}],"stream":false}'

openclaw infer model run \
  --model inferrs/gg-hf-gg/gemma-4-E2B-it \
  --prompt "What is 2 + 2? Reply with one short sentence." \
  --json
```

Si el primer comando funciona pero el segundo falla, usa las notas de solución de problemas
de abajo.

## Solución de problemas

- `curl /v1/models` falla: `inferrs` no está en ejecución, no es accesible o no
  está enlazado al host/puerto esperado.
- `messages[].content ... expected a string`: establece
  `compat.requiresStringContent: true`.
- Las llamadas directas pequeñas a `/v1/chat/completions` funcionan, pero `openclaw infer model run`
  falla: prueba `compat.supportsTools: false`.
- OpenClaw ya no recibe errores de esquema, pero `inferrs` sigue fallando en turnos de
  agentes más grandes: trátalo como una limitación ascendente de `inferrs` o del modelo y reduce la
  presión del prompt o cambia de backend/modelo local.

## Comportamiento de estilo proxy

`inferrs` se trata como un backend `/v1` compatible con OpenAI de estilo proxy, no como un
endpoint nativo de OpenAI.

- aquí no se aplica el modelado de solicitudes exclusivo de OpenAI nativo
- no hay `service_tier`, ni `store` de Responses, ni sugerencias de caché de prompts, ni
  modelado de payload de compatibilidad de razonamiento de OpenAI
- los encabezados ocultos de atribución de OpenClaw (`originator`, `version`, `User-Agent`)
  no se inyectan en URLs base personalizadas de `inferrs`

## Ver también

- [Modelos locales](/es/gateway/local-models)
- [Solución de problemas del gateway](/es/gateway/troubleshooting#local-openai-compatible-backend-passes-direct-probes-but-agent-runs-fail)
- [Proveedores de modelos](/es/concepts/model-providers)
