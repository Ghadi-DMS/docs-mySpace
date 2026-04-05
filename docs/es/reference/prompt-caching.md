---
read_when:
    - Quieres reducir los costos de tokens de prompt con la retención de caché
    - Necesitas un comportamiento de caché por agente en configuraciones con varios agentes
    - Estás ajustando juntos el heartbeat y la poda de `cache-ttl`
summary: Controles de almacenamiento en caché de prompts, orden de combinación, comportamiento del proveedor y patrones de ajuste
title: Almacenamiento en caché de prompts
x-i18n:
    generated_at: "2026-04-05T12:53:42Z"
    model: gpt-5.4
    provider: openai
    source_hash: 13d5f3153b6593ae22cd04a6c2540e074cf15df9f1990fc5b7184fe803f4a1bd
    source_path: reference/prompt-caching.md
    workflow: 15
---

# Almacenamiento en caché de prompts

El almacenamiento en caché de prompts significa que el proveedor del modelo puede reutilizar prefijos de prompt sin cambios (normalmente instrucciones de sistema/desarrollador y otro contexto estable) entre turnos en lugar de volver a procesarlos cada vez. OpenClaw normaliza el uso del proveedor en `cacheRead` y `cacheWrite` cuando la API upstream expone esos contadores directamente.

Las superficies de estado también pueden recuperar contadores de caché del registro
de uso de la transcripción más reciente cuando la instantánea de la sesión en vivo no los tiene, para que `/status` pueda seguir
mostrando una línea de caché después de una pérdida parcial de metadatos de sesión. Los valores de caché en vivo existentes que no son cero siguen teniendo prioridad sobre los valores de respaldo de la transcripción.

Por qué esto importa: menor costo de tokens, respuestas más rápidas y un rendimiento más predecible para sesiones de larga duración. Sin almacenamiento en caché, los prompts repetidos pagan el costo completo del prompt en cada turno incluso cuando la mayor parte de la entrada no cambió.

Esta página cubre todos los controles relacionados con la caché que afectan la reutilización de prompts y el costo de tokens.

Referencias de proveedores:

- Almacenamiento en caché de prompts de Anthropic: [https://platform.claude.com/docs/en/build-with-claude/prompt-caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- Almacenamiento en caché de prompts de OpenAI: [https://developers.openai.com/api/docs/guides/prompt-caching](https://developers.openai.com/api/docs/guides/prompt-caching)
- Encabezados de API e ID de solicitud de OpenAI: [https://developers.openai.com/api/reference/overview](https://developers.openai.com/api/reference/overview)
- ID de solicitud y errores de Anthropic: [https://platform.claude.com/docs/en/api/errors](https://platform.claude.com/docs/en/api/errors)

## Controles principales

### `cacheRetention` (valor predeterminado global, modelo y por agente)

Establece la retención de caché como valor predeterminado global para todos los modelos:

```yaml
agents:
  defaults:
    params:
      cacheRetention: "long" # none | short | long
```

Sobrescribe por modelo:

```yaml
agents:
  defaults:
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "short" # none | short | long
```

Sobrescritura por agente:

```yaml
agents:
  list:
    - id: "alerts"
      params:
        cacheRetention: "none"
```

Orden de combinación de configuración:

1. `agents.defaults.params` (valor predeterminado global — se aplica a todos los modelos)
2. `agents.defaults.models["provider/model"].params` (sobrescritura por modelo)
3. `agents.list[].params` (id de agente coincidente; sobrescribe por clave)

### `contextPruning.mode: "cache-ttl"`

Poda el contexto antiguo de resultados de herramientas después de las ventanas TTL de caché para que las solicitudes posteriores a inactividad no vuelvan a almacenar en caché un historial sobredimensionado.

```yaml
agents:
  defaults:
    contextPruning:
      mode: "cache-ttl"
      ttl: "1h"
```

Consulta [Poda de sesiones](/es/concepts/session-pruning) para ver el comportamiento completo.

### Heartbeat keep-warm

El heartbeat puede mantener activas las ventanas de caché y reducir las escrituras repetidas en caché después de períodos de inactividad.

```yaml
agents:
  defaults:
    heartbeat:
      every: "55m"
```

El heartbeat por agente es compatible en `agents.list[].heartbeat`.

## Comportamiento del proveedor

### Anthropic (API directa)

- `cacheRetention` es compatible.
- Con perfiles de autenticación mediante API key de Anthropic, OpenClaw inicializa `cacheRetention: "short"` para referencias de modelos de Anthropic cuando no está definido.
- Las respuestas nativas de Messages de Anthropic exponen tanto `cache_read_input_tokens` como `cache_creation_input_tokens`, por lo que OpenClaw puede mostrar tanto `cacheRead` como `cacheWrite`.
- Para solicitudes nativas de Anthropic, `cacheRetention: "short"` se asigna a la caché efímera predeterminada de 5 minutos, y `cacheRetention: "long"` amplía al TTL de 1 hora solo en hosts directos `api.anthropic.com`.

### OpenAI (API directa)

- El almacenamiento en caché de prompts es automático en los modelos recientes compatibles. OpenClaw no necesita inyectar marcadores de caché a nivel de bloque.
- OpenClaw usa `prompt_cache_key` para mantener estable el enrutamiento de caché entre turnos y usa `prompt_cache_retention: "24h"` solo cuando `cacheRetention: "long"` está seleccionado en hosts directos de OpenAI.
- Las respuestas de OpenAI exponen los tokens de prompt almacenados en caché mediante `usage.prompt_tokens_details.cached_tokens` (o `input_tokens_details.cached_tokens` en eventos de la Responses API). OpenClaw asigna eso a `cacheRead`.
- OpenAI no expone un contador separado de tokens de escritura en caché, por lo que `cacheWrite` permanece en `0` en rutas de OpenAI incluso cuando el proveedor está calentando una caché.
- OpenAI devuelve encabezados útiles de trazabilidad y límites de tasa como `x-request-id`, `openai-processing-ms` y `x-ratelimit-*`, pero el cálculo de aciertos de caché debe provenir de la carga útil de uso, no de los encabezados.
- En la práctica, OpenAI suele comportarse como una caché de prefijo inicial en lugar de una reutilización móvil de historial completo al estilo Anthropic. Los turnos de texto largo con prefijo estable pueden llegar a una meseta cercana a `4864` tokens almacenados en caché en las pruebas en vivo actuales, mientras que las transcripciones con muchas herramientas o de estilo MCP suelen estabilizarse cerca de `4608` tokens almacenados en caché incluso en repeticiones exactas.

### Anthropic Vertex

- Los modelos de Anthropic en Vertex AI (`anthropic-vertex/*`) admiten `cacheRetention` de la misma manera que Anthropic directo.
- `cacheRetention: "long"` se asigna al TTL real de caché de prompts de 1 hora en endpoints de Vertex AI.
- La retención de caché predeterminada para `anthropic-vertex` coincide con los valores predeterminados directos de Anthropic.
- Las solicitudes de Vertex se enrutan mediante modelado de caché consciente de límites para que la reutilización de caché se mantenga alineada con lo que los proveedores realmente reciben.

### Amazon Bedrock

- Las referencias de modelos Anthropic Claude (`amazon-bedrock/*anthropic.claude*`) admiten el paso explícito de `cacheRetention`.
- Los modelos de Bedrock que no son de Anthropic se fuerzan a `cacheRetention: "none"` en tiempo de ejecución.

### Modelos Anthropic en OpenRouter

Para referencias de modelos `openrouter/anthropic/*`, OpenClaw inyecta
`cache_control` de Anthropic en bloques de prompt de sistema/desarrollador para mejorar la reutilización de la caché de prompts
solo cuando la solicitud sigue apuntando a una ruta verificada de OpenRouter
(`openrouter` en su endpoint predeterminado, o cualquier proveedor/base URL que resuelva
a `openrouter.ai`).

Si rediriges el modelo a una URL proxy arbitraria compatible con OpenAI, OpenClaw
deja de inyectar esos marcadores de caché específicos de Anthropic para OpenRouter.

### Otros proveedores

Si el proveedor no admite este modo de caché, `cacheRetention` no tiene efecto.

### API directa de Google Gemini

- El transporte directo de Gemini (`api: "google-generative-ai"`) informa los aciertos de caché
  mediante `cachedContentTokenCount` upstream; OpenClaw lo asigna a `cacheRead`.
- Cuando `cacheRetention` está establecido en un modelo directo de Gemini, OpenClaw
  crea, reutiliza y actualiza automáticamente recursos `cachedContents` para prompts del sistema
  en ejecuciones de Google AI Studio. Esto significa que ya no necesitas crear previamente
  un identificador de contenido en caché manualmente.
- Aun así puedes pasar un identificador existente de contenido en caché de Gemini como
  `params.cachedContent` (o el heredado `params.cached_content`) en el modelo
  configurado.
- Esto es independiente del almacenamiento en caché de prefijos de prompt de Anthropic/OpenAI. Para Gemini,
  OpenClaw administra un recurso `cachedContents` nativo del proveedor en lugar de
  inyectar marcadores de caché en la solicitud.

### Uso JSON de CLI de Gemini

- La salida JSON de la CLI de Gemini también puede mostrar aciertos de caché mediante `stats.cached`;
  OpenClaw lo asigna a `cacheRead`.
- Si la CLI omite un valor directo `stats.input`, OpenClaw deriva los tokens de entrada
  a partir de `stats.input_tokens - stats.cached`.
- Esto es solo normalización de uso. No significa que OpenClaw esté creando
  marcadores de caché de prompts al estilo Anthropic/OpenAI para la CLI de Gemini.

## Límite de caché del prompt del sistema

OpenClaw divide el prompt del sistema en un **prefijo estable** y un **sufijo volátil**
separados por un límite interno de prefijo de caché. El contenido por encima del
límite (definiciones de herramientas, metadatos de Skills, archivos del espacio de trabajo y otro
contexto relativamente estático) se ordena para que permanezca idéntico a nivel de bytes entre turnos.
El contenido por debajo del límite (por ejemplo `HEARTBEAT.md`, marcas de tiempo de ejecución y
otros metadatos por turno) puede cambiar sin invalidar el prefijo
almacenado en caché.

Decisiones clave de diseño:

- Los archivos estables de contexto del proyecto del espacio de trabajo se ordenan antes de `HEARTBEAT.md` para que
  el cambio frecuente de heartbeat no invalide el prefijo estable.
- El límite se aplica en el modelado de caché de la familia Anthropic, la familia OpenAI, Google y el transporte
  de CLI, para que todos los proveedores compatibles se beneficien de la misma estabilidad
  de prefijo.
- Las solicitudes de Codex Responses y Anthropic Vertex se enrutan mediante
  modelado de caché consciente de límites para que la reutilización de caché se mantenga alineada con lo que los proveedores
  realmente reciben.
- Las huellas del prompt del sistema se normalizan (espacios en blanco, finales de línea,
  contexto añadido por hooks, orden de capacidades en tiempo de ejecución) para que los prompts
  semánticamente sin cambios compartan KV/caché entre turnos.

Si ves picos inesperados de `cacheWrite` después de un cambio de configuración o del espacio de trabajo,
comprueba si el cambio cae por encima o por debajo del límite de caché. Mover
contenido volátil por debajo del límite (o estabilizarlo) suele resolver el
problema.

## Protecciones de estabilidad de caché de OpenClaw

OpenClaw también mantiene deterministas varias formas de carga útil sensibles a la caché antes de
que la solicitud llegue al proveedor:

- Los catálogos agrupados de herramientas MCP se ordenan de forma determinista antes del
  registro de herramientas, para que los cambios en el orden de `listTools()` no alteren el bloque de herramientas ni
  invaliden los prefijos de caché de prompts.
- Las sesiones heredadas con bloques de imágenes persistidos mantienen intactos los **3 turnos completados más recientes**;
  los bloques de imágenes antiguos ya procesados pueden
  sustituirse por un marcador para que los seguimientos con muchas imágenes no sigan reenviando cargas útiles grandes
  obsoletas.

## Patrones de ajuste

### Tráfico mixto (valor predeterminado recomendado)

Mantén una base de larga duración en tu agente principal y desactiva el almacenamiento en caché en agentes notificadores con ráfagas:

```yaml
agents:
  defaults:
    model:
      primary: "anthropic/claude-opus-4-6"
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "long"
  list:
    - id: "research"
      default: true
      heartbeat:
        every: "55m"
    - id: "alerts"
      params:
        cacheRetention: "none"
```

### Base orientada al costo

- Establece una base con `cacheRetention: "short"`.
- Activa `contextPruning.mode: "cache-ttl"`.
- Mantén el heartbeat por debajo de tu TTL solo para los agentes que se benefician de cachés activas.

## Diagnóstico de caché

OpenClaw expone diagnósticos dedicados de trazas de caché para ejecuciones de agentes integrados.

Para los diagnósticos normales orientados al usuario, `/status` y otros resúmenes de uso pueden usar
la entrada de uso más reciente de la transcripción como fuente de respaldo para `cacheRead` /
`cacheWrite` cuando la entrada de la sesión en vivo no tiene esos contadores.

## Pruebas de regresión en vivo

OpenClaw mantiene una única compuerta combinada de regresión de caché en vivo para prefijos repetidos, turnos con herramientas, turnos con imágenes, transcripciones de herramientas de estilo MCP y un control sin caché de Anthropic.

- `src/agents/live-cache-regression.live.test.ts`
- `src/agents/live-cache-regression-baseline.ts`

Ejecuta la compuerta en vivo reducida con:

```sh
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_CACHE_TEST=1 pnpm test:live:cache
```

El archivo de referencia almacena los números observados en vivo más recientes junto con los umbrales mínimos de regresión específicos del proveedor usados por la prueba.
El ejecutor también usa IDs de sesión nuevos por ejecución y espacios de nombres de prompt para que el estado previo de la caché no contamine la muestra de regresión actual.

Estas pruebas intencionalmente no usan criterios de éxito idénticos en todos los proveedores.

### Expectativas en vivo de Anthropic

- Espera escrituras de calentamiento explícitas mediante `cacheWrite`.
- Espera una reutilización de casi todo el historial en turnos repetidos porque el control de caché de Anthropic adelanta el punto de corte de caché a lo largo de la conversación.
- Las aserciones en vivo actuales siguen usando umbrales altos de tasa de aciertos para rutas estables, con herramientas e imágenes.

### Expectativas en vivo de OpenAI

- Espera solo `cacheRead`. `cacheWrite` permanece en `0`.
- Trata la reutilización de caché en turnos repetidos como una meseta específica del proveedor, no como una reutilización móvil de historial al estilo Anthropic.
- Las aserciones en vivo actuales usan comprobaciones de umbral mínimo conservadoras derivadas del comportamiento observado en vivo en `gpt-5.4-mini`:
  - prefijo estable: `cacheRead >= 4608`, tasa de aciertos `>= 0.90`
  - transcripción de herramientas: `cacheRead >= 4096`, tasa de aciertos `>= 0.85`
  - transcripción de imágenes: `cacheRead >= 3840`, tasa de aciertos `>= 0.82`
  - transcripción de estilo MCP: `cacheRead >= 4096`, tasa de aciertos `>= 0.85`

La verificación en vivo combinada más reciente del 2026-04-04 obtuvo:

- prefijo estable: `cacheRead=4864`, tasa de aciertos `0.966`
- transcripción de herramientas: `cacheRead=4608`, tasa de aciertos `0.896`
- transcripción de imágenes: `cacheRead=4864`, tasa de aciertos `0.954`
- transcripción de estilo MCP: `cacheRead=4608`, tasa de aciertos `0.891`

El tiempo de reloj local reciente para la compuerta combinada fue de aproximadamente `88s`.

Por qué las aserciones difieren:

- Anthropic expone puntos de corte de caché explícitos y reutilización móvil del historial de conversación.
- El almacenamiento en caché de prompts de OpenAI sigue siendo sensible al prefijo exacto, pero el prefijo reutilizable efectivo en tráfico vivo de Responses puede estabilizarse antes que el prompt completo.
- Debido a eso, comparar Anthropic y OpenAI con un único umbral porcentual entre proveedores genera regresiones falsas.

### Configuración de `diagnostics.cacheTrace`

```yaml
diagnostics:
  cacheTrace:
    enabled: true
    filePath: "~/.openclaw/logs/cache-trace.jsonl" # opcional
    includeMessages: false # predeterminado true
    includePrompt: false # predeterminado true
    includeSystem: false # predeterminado true
```

Valores predeterminados:

- `filePath`: `$OPENCLAW_STATE_DIR/logs/cache-trace.jsonl`
- `includeMessages`: `true`
- `includePrompt`: `true`
- `includeSystem`: `true`

### Variables de entorno (depuración puntual)

- `OPENCLAW_CACHE_TRACE=1` activa el rastreo de caché.
- `OPENCLAW_CACHE_TRACE_FILE=/path/to/cache-trace.jsonl` sobrescribe la ruta de salida.
- `OPENCLAW_CACHE_TRACE_MESSAGES=0|1` activa o desactiva la captura de la carga útil completa de mensajes.
- `OPENCLAW_CACHE_TRACE_PROMPT=0|1` activa o desactiva la captura del texto del prompt.
- `OPENCLAW_CACHE_TRACE_SYSTEM=0|1` activa o desactiva la captura del prompt del sistema.

### Qué inspeccionar

- Los eventos de rastreo de caché están en JSONL e incluyen instantáneas por etapas como `session:loaded`, `prompt:before`, `stream:context` y `session:after`.
- El impacto por turno de tokens de caché es visible en las superficies normales de uso mediante `cacheRead` y `cacheWrite` (por ejemplo `/usage full` y resúmenes de uso de sesión).
- Para Anthropic, espera tanto `cacheRead` como `cacheWrite` cuando el almacenamiento en caché esté activo.
- Para OpenAI, espera `cacheRead` en aciertos de caché y que `cacheWrite` permanezca en `0`; OpenAI no publica un campo separado de tokens de escritura en caché.
- Si necesitas trazabilidad de solicitudes, registra por separado los ID de solicitud y los encabezados de límite de tasa de las métricas de caché. La salida actual de rastreo de caché de OpenClaw se centra en la forma del prompt/sesión y en el uso normalizado de tokens, en lugar de en encabezados sin procesar de respuestas del proveedor.

## Solución rápida de problemas

- `cacheWrite` alto en la mayoría de los turnos: comprueba si hay entradas volátiles en el prompt del sistema y verifica que el modelo/proveedor admita tu configuración de caché.
- `cacheWrite` alto en Anthropic: a menudo significa que el punto de corte de caché está cayendo sobre contenido que cambia en cada solicitud.
- `cacheRead` bajo en OpenAI: verifica que el prefijo estable esté al principio, que el prefijo repetido tenga al menos 1024 tokens y que se reutilice la misma `prompt_cache_key` en los turnos que deberían compartir una caché.
- Sin efecto de `cacheRetention`: confirma que la clave del modelo coincida con `agents.defaults.models["provider/model"]`.
- Solicitudes de Bedrock Nova/Mistral con configuración de caché: es esperado que en tiempo de ejecución se fuerce a `none`.

Documentos relacionados:

- [Anthropic](/es/providers/anthropic)
- [Uso de tokens y costos](/reference/token-use)
- [Poda de sesiones](/es/concepts/session-pruning)
- [Referencia de configuración de Gateway](/es/gateway/configuration-reference)
