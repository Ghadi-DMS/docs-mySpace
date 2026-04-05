---
read_when:
    - Quieres usar modelos de OpenAI en OpenClaw
    - Quieres autenticación con suscripción de Codex en lugar de claves API
summary: Usar OpenAI mediante claves API o suscripción de Codex en OpenClaw
title: OpenAI
x-i18n:
    generated_at: "2026-04-05T12:52:09Z"
    model: gpt-5.4
    provider: openai
    source_hash: 537119853503d398f9136170ac12ecfdbd9af8aef3c4c011f8ada4c664bdaf6d
    source_path: providers/openai.md
    workflow: 15
---

# OpenAI

OpenAI proporciona API de desarrollo para modelos GPT. Codex admite **inicio de sesión con ChatGPT** para acceso por suscripción
o **inicio de sesión con clave API** para acceso basado en uso. Codex cloud requiere inicio de sesión con ChatGPT.
OpenAI admite explícitamente el uso de OAuth por suscripción en herramientas/flujos externos como OpenClaw.

## Estilo de interacción predeterminado

OpenClaw agrega por defecto una pequeña superposición de prompt específica de OpenAI tanto para
ejecuciones `openai/*` como `openai-codex/*`. La superposición mantiene al asistente cálido,
colaborativo, conciso y directo sin reemplazar el prompt de sistema base de OpenClaw.

Clave de configuración:

`plugins.entries.openai.config.personalityOverlay`

Valores permitidos:

- `"friendly"`: predeterminado; habilita la superposición específica de OpenAI.
- `"off"`: desactiva la superposición y usa solo el prompt base de OpenClaw.

Alcance:

- Se aplica a modelos `openai/*`.
- Se aplica a modelos `openai-codex/*`.
- No afecta a otros proveedores.

Este comportamiento está habilitado por defecto:

```json5
{
  plugins: {
    entries: {
      openai: {
        config: {
          personalityOverlay: "friendly",
        },
      },
    },
  },
}
```

### Desactivar la superposición de prompt de OpenAI

Si prefieres el prompt base de OpenClaw sin modificaciones, desactiva la superposición:

```json5
{
  plugins: {
    entries: {
      openai: {
        config: {
          personalityOverlay: "off",
        },
      },
    },
  },
}
```

También puedes configurarlo directamente con la CLI de configuración:

```bash
openclaw config set plugins.entries.openai.config.personalityOverlay off
```

## Opción A: clave API de OpenAI (OpenAI Platform)

**Lo mejor para:** acceso directo a la API y facturación basada en uso.
Obtén tu clave API desde el panel de OpenAI.

### Configuración con la CLI

```bash
openclaw onboard --auth-choice openai-api-key
# o no interactivo
openclaw onboard --openai-api-key "$OPENAI_API_KEY"
```

### Fragmento de configuración

```json5
{
  env: { OPENAI_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "openai/gpt-5.4" } } },
}
```

La documentación actual de modelos API de OpenAI lista `gpt-5.4` y `gpt-5.4-pro` para uso directo
de la API de OpenAI. OpenClaw reenvía ambos mediante la ruta `openai/*` de Responses.
OpenClaw suprime intencionadamente la fila obsoleta `openai/gpt-5.3-codex-spark`,
porque las llamadas directas a la API de OpenAI la rechazan en tráfico live.

OpenClaw **no** expone `openai/gpt-5.3-codex-spark` en la ruta directa de la API de OpenAI.
`pi-ai` sigue incluyendo una fila integrada para ese modelo, pero las solicitudes live a la API de OpenAI
actualmente la rechazan. Spark se trata como exclusivo de Codex en OpenClaw.

## Opción B: suscripción a OpenAI Code (Codex)

**Lo mejor para:** usar acceso por suscripción de ChatGPT/Codex en lugar de una clave API.
Codex cloud requiere inicio de sesión con ChatGPT, mientras que la CLI de Codex admite inicio de sesión con ChatGPT o con clave API.

### Configuración con la CLI (OAuth de Codex)

```bash
# Ejecuta el OAuth de Codex en el asistente
openclaw onboard --auth-choice openai-codex

# O ejecuta OAuth directamente
openclaw models auth login --provider openai-codex
```

### Fragmento de configuración (suscripción de Codex)

```json5
{
  agents: { defaults: { model: { primary: "openai-codex/gpt-5.4" } } },
}
```

La documentación actual de Codex de OpenAI lista `gpt-5.4` como el modelo actual de Codex. OpenClaw
lo asigna a `openai-codex/gpt-5.4` para uso con OAuth de ChatGPT/Codex.

Si el onboarding reutiliza un inicio de sesión existente de la CLI de Codex, esas credenciales siguen
gestionadas por la CLI de Codex. Cuando caducan, OpenClaw vuelve a leer primero la fuente externa de Codex
y, cuando el proveedor puede actualizarla, escribe la credencial actualizada
de vuelta en el almacenamiento de Codex en lugar de asumir la propiedad en una copia separada solo de OpenClaw.

Si tu cuenta de Codex tiene derecho a Codex Spark, OpenClaw también admite:

- `openai-codex/gpt-5.3-codex-spark`

OpenClaw trata Codex Spark como exclusivo de Codex. No expone una ruta directa
`openai/gpt-5.3-codex-spark` con clave API.

OpenClaw también conserva `openai-codex/gpt-5.3-codex-spark` cuando `pi-ai`
lo descubre. Trátalo como algo dependiente de permisos y experimental: Codex Spark es
independiente de `/fast` de GPT-5.4, y su disponibilidad depende de la cuenta de Codex /
ChatGPT con sesión iniciada.

### Límite de ventana de contexto de Codex

OpenClaw trata los metadatos del modelo Codex y el límite de contexto de tiempo de ejecución como
valores separados.

Para `openai-codex/gpt-5.4`:

- `contextWindow` nativo: `1050000`
- límite predeterminado `contextTokens` de tiempo de ejecución: `272000`

Esto mantiene veraces los metadatos del modelo mientras conserva la ventana predeterminada más pequeña en tiempo de ejecución
que en la práctica ofrece mejores características de latencia y calidad.

Si quieres un límite efectivo diferente, configura `models.providers.<provider>.models[].contextTokens`:

```json5
{
  models: {
    providers: {
      "openai-codex": {
        models: [
          {
            id: "gpt-5.4",
            contextTokens: 160000,
          },
        ],
      },
    },
  },
}
```

Usa `contextWindow` solo cuando declares o sobrescribas metadatos nativos del modelo.
Usa `contextTokens` cuando quieras limitar el presupuesto de contexto en tiempo de ejecución.

### Transporte predeterminado

OpenClaw usa `pi-ai` para el streaming de modelos. Tanto para `openai/*` como para
`openai-codex/*`, el transporte predeterminado es `"auto"` (primero WebSocket y después
respaldo SSE).

En modo `"auto"`, OpenClaw también reintenta un fallo temprano y reintentable de WebSocket
antes de pasar a SSE. El modo forzado `"websocket"` sigue mostrando directamente los errores de transporte
en lugar de ocultarlos tras el fallback.

Después de un fallo de conexión o de un fallo temprano de turno en WebSocket en modo `"auto"`, OpenClaw marca
la ruta WebSocket de esa sesión como degradada durante unos 60 segundos y envía
los turnos posteriores por SSE durante ese período de enfriamiento en lugar de alternar
entre transportes.

Para endpoints nativos de la familia OpenAI (`openai/*`, `openai-codex/*` y Azure
OpenAI Responses), OpenClaw también adjunta estado estable de sesión e identidad de turno
a las solicitudes para que los reintentos, las reconexiones y el fallback a SSE sigan alineados con la misma
identidad de conversación. En las rutas nativas de la familia OpenAI esto incluye cabeceras estables de identidad de solicitud de sesión/turno además de metadatos de transporte coincidentes.

OpenClaw también normaliza los contadores de uso de OpenAI entre variantes de transporte antes de
que lleguen a las superficies de sesión/status. El tráfico nativo de OpenAI/Codex Responses puede
informar del uso como `input_tokens` / `output_tokens` o como
`prompt_tokens` / `completion_tokens`; OpenClaw trata esos valores como los mismos contadores de entrada
y salida para `/status`, `/usage` y logs de sesión. Cuando el tráfico WebSocket nativo
omite `total_tokens` (o informa `0`), OpenClaw usa como respaldo el total normalizado de entrada + salida para
que las vistas de sesión/status sigan mostrando datos.

Puedes establecer `agents.defaults.models.<provider/model>.params.transport`:

- `"sse"`: fuerza SSE
- `"websocket"`: fuerza WebSocket
- `"auto"`: intenta WebSocket y luego usa SSE como fallback

Para `openai/*` (API Responses), OpenClaw también habilita por
defecto el calentamiento de WebSocket (`openaiWsWarmup: true`) cuando se usa transporte WebSocket.

Documentación relacionada de OpenAI:

- [Realtime API with WebSocket](https://platform.openai.com/docs/guides/realtime-websocket)
- [Streaming API responses (SSE)](https://platform.openai.com/docs/guides/streaming-responses)

```json5
{
  agents: {
    defaults: {
      model: { primary: "openai-codex/gpt-5.4" },
      models: {
        "openai-codex/gpt-5.4": {
          params: {
            transport: "auto",
          },
        },
      },
    },
  },
}
```

### Calentamiento de WebSocket de OpenAI

La documentación de OpenAI describe el calentamiento como opcional. OpenClaw lo habilita por defecto para
`openai/*` para reducir la latencia del primer turno al usar transporte WebSocket.

### Desactivar el calentamiento

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            openaiWsWarmup: false,
          },
        },
      },
    },
  },
}
```

### Habilitar el calentamiento explícitamente

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            openaiWsWarmup: true,
          },
        },
      },
    },
  },
}
```

### Procesamiento prioritario de OpenAI y Codex

La API de OpenAI expone el procesamiento prioritario mediante `service_tier=priority`. En
OpenClaw, configura `agents.defaults.models["<provider>/<model>"].params.serviceTier`
para reenviar ese campo en endpoints nativos de OpenAI/Codex Responses.

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            serviceTier: "priority",
          },
        },
        "openai-codex/gpt-5.4": {
          params: {
            serviceTier: "priority",
          },
        },
      },
    },
  },
}
```

Los valores compatibles son `auto`, `default`, `flex` y `priority`.

OpenClaw reenvía `params.serviceTier` tanto a solicitudes directas de Responses `openai/*`
como a solicitudes de Codex Responses `openai-codex/*` cuando esos modelos apuntan
a los endpoints nativos de OpenAI/Codex.

Comportamiento importante:

- `openai/*` directo debe apuntar a `api.openai.com`
- `openai-codex/*` debe apuntar a `chatgpt.com/backend-api`
- si enrutas cualquiera de los dos proveedores mediante otra URL base o un proxy, OpenClaw deja `service_tier` sin tocar

### Modo rápido de OpenAI

OpenClaw expone un interruptor compartido de modo rápido tanto para sesiones `openai/*` como
`openai-codex/*`:

- Chat/UI: `/fast status|on|off`
- Configuración: `agents.defaults.models["<provider>/<model>"].params.fastMode`

Cuando el modo rápido está habilitado, OpenClaw lo asigna al procesamiento prioritario de OpenAI:

- las llamadas directas a Responses `openai/*` hacia `api.openai.com` envían `service_tier = "priority"`
- las llamadas a Responses `openai-codex/*` hacia `chatgpt.com/backend-api` también envían `service_tier = "priority"`
- los valores existentes de `service_tier` en la carga útil se conservan
- el modo rápido no reescribe `reasoning` ni `text.verbosity`

Ejemplo:

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            fastMode: true,
          },
        },
        "openai-codex/gpt-5.4": {
          params: {
            fastMode: true,
          },
        },
      },
    },
  },
}
```

Las sobrescrituras de sesión prevalecen sobre la configuración. Borrar la sobrescritura de sesión en la UI de Sessions
hace que la sesión vuelva al valor predeterminado configurado.

### Rutas nativas de OpenAI frente a rutas compatibles con OpenAI

OpenClaw trata los endpoints directos de OpenAI, Codex y Azure OpenAI de forma diferente
a los proxies `/v1` genéricos compatibles con OpenAI:

- las rutas nativas `openai/*`, `openai-codex/*` y Azure OpenAI mantienen
  `reasoning: { effort: "none" }` intacto cuando desactivas explícitamente el razonamiento
- las rutas nativas de la familia OpenAI usan por defecto el modo estricto para esquemas de herramientas
- las cabeceras ocultas de atribución de OpenClaw (`originator`, `version` y
  `User-Agent`) solo se adjuntan en hosts nativos verificados de OpenAI
  (`api.openai.com`) y hosts nativos de Codex (`chatgpt.com/backend-api`)
- las rutas nativas de OpenAI/Codex mantienen el modelado de solicitudes exclusivo de OpenAI, como
  `service_tier`, `store` de Responses, cargas útiles de compatibilidad de razonamiento de OpenAI y
  pistas de caché de prompts
- las rutas proxy de estilo compatible con OpenAI mantienen el comportamiento de compatibilidad más flexible y no
  fuerzan esquemas de herramientas estrictos, modelado de solicitudes exclusivo del modo nativo ni cabeceras ocultas
  de atribución de OpenAI/Codex

Azure OpenAI sigue dentro del grupo de enrutamiento nativo para transporte y comportamiento de compatibilidad,
pero no recibe las cabeceras ocultas de atribución de OpenAI/Codex.

Esto preserva el comportamiento actual nativo de OpenAI Responses sin forzar
adaptadores más antiguos de estilo OpenAI-compatible sobre backends `/v1` de terceros.

### Compactación en servidor de OpenAI Responses

Para modelos directos de OpenAI Responses (`openai/*` usando `api: "openai-responses"` con
`baseUrl` en `api.openai.com`), OpenClaw ahora habilita automáticamente pistas de carga útil
de compactación del lado del servidor de OpenAI:

- Fuerza `store: true` (salvo que la compatibilidad del modelo configure `supportsStore: false`)
- Inyecta `context_management: [{ type: "compaction", compact_threshold: ... }]`

Por defecto, `compact_threshold` es el `70%` de `contextWindow` del modelo (o `80000`
cuando no está disponible).

### Habilitar explícitamente la compactación del lado del servidor

Úsalo cuando quieras forzar la inyección de `context_management` en modelos
Responses compatibles (por ejemplo Azure OpenAI Responses):

```json5
{
  agents: {
    defaults: {
      models: {
        "azure-openai-responses/gpt-5.4": {
          params: {
            responsesServerCompaction: true,
          },
        },
      },
    },
  },
}
```

### Habilitar con un umbral personalizado

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            responsesServerCompaction: true,
            responsesCompactThreshold: 120000,
          },
        },
      },
    },
  },
}
```

### Desactivar la compactación del lado del servidor

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            responsesServerCompaction: false,
          },
        },
      },
    },
  },
}
```

`responsesServerCompaction` solo controla la inyección de `context_management`.
Los modelos directos de OpenAI Responses siguen forzando `store: true` salvo que compat configure
`supportsStore: false`.

## Notas

- Las referencias de modelo siempre usan `provider/model` (consulta [/concepts/models](/concepts/models)).
- Los detalles de autenticación + reglas de reutilización están en [/concepts/oauth](/concepts/oauth).
