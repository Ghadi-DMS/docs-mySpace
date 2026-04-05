---
read_when:
    - Ajustar el análisis de directivas o los valores predeterminados de thinking, fast-mode o verbose
summary: Sintaxis de directivas para /think, /fast, /verbose y visibilidad del razonamiento
title: Niveles de razonamiento
x-i18n:
    generated_at: "2026-04-05T12:56:55Z"
    model: gpt-5.4
    provider: openai
    source_hash: f60aeb6ab4c7ce858f725f589f54184b29d8c91994d18c8deafa75179b9a62cb
    source_path: tools/thinking.md
    workflow: 15
---

# Niveles de razonamiento (directivas `/think`)

## Qué hace

- Directiva en línea en cualquier cuerpo entrante: `/t <level>`, `/think:<level>` o `/thinking <level>`.
- Niveles (alias): `off | minimal | low | medium | high | xhigh | adaptive`
  - minimal → “think”
  - low → “think hard”
  - medium → “think harder”
  - high → “ultrathink” (presupuesto máximo)
  - xhigh → “ultrathink+” (solo modelos GPT-5.2 + Codex)
  - adaptive → presupuesto de razonamiento adaptativo gestionado por el proveedor (compatible con la familia de modelos Anthropic Claude 4.6)
  - `x-high`, `x_high`, `extra-high`, `extra high` y `extra_high` se asignan a `xhigh`.
  - `highest`, `max` se asignan a `high`.
- Notas sobre proveedores:
  - Los modelos Anthropic Claude 4.6 usan `adaptive` por defecto cuando no se establece un nivel de razonamiento explícito.
  - MiniMax (`minimax/*`) en la ruta de streaming compatible con Anthropic usa por defecto `thinking: { type: "disabled" }` a menos que establezcas explícitamente el razonamiento en los parámetros del modelo o en los parámetros de la solicitud. Esto evita fugas de deltas `reasoning_content` del formato de stream Anthropic no nativo de MiniMax.
  - Z.AI (`zai/*`) solo admite razonamiento binario (`on`/`off`). Cualquier nivel distinto de `off` se trata como `on` (asignado a `low`).
  - Moonshot (`moonshot/*`) asigna `/think off` a `thinking: { type: "disabled" }` y cualquier nivel distinto de `off` a `thinking: { type: "enabled" }`. Cuando el razonamiento está habilitado, Moonshot solo acepta `tool_choice` `auto|none`; OpenClaw normaliza los valores incompatibles a `auto`.

## Orden de resolución

1. Directiva en línea en el mensaje (se aplica solo a ese mensaje).
2. Sobrescritura de sesión (establecida al enviar un mensaje que contiene solo la directiva).
3. Predeterminado por agente (`agents.list[].thinkingDefault` en la configuración).
4. Predeterminado global (`agents.defaults.thinkingDefault` en la configuración).
5. Respaldo: `adaptive` para modelos Anthropic Claude 4.6, `low` para otros modelos con capacidad de razonamiento y `off` en caso contrario.

## Establecer un valor predeterminado de sesión

- Envía un mensaje que sea **solo** la directiva (se permiten espacios), por ejemplo `/think:medium` o `/t high`.
- Esto se mantiene para la sesión actual (por defecto por remitente); se borra con `/think:off` o al restablecer la inactividad de la sesión.
- Se envía una respuesta de confirmación (`Thinking level set to high.` / `Thinking disabled.`). Si el nivel no es válido (por ejemplo `/thinking big`), el comando se rechaza con una pista y el estado de la sesión permanece sin cambios.
- Envía `/think` (o `/think:`) sin argumento para ver el nivel de razonamiento actual.

## Aplicación por agente

- **Pi integrado**: el nivel resuelto se pasa al runtime del agente Pi en proceso.

## Modo rápido (/fast)

- Niveles: `on|off`.
- Un mensaje que contenga solo la directiva activa o desactiva una sobrescritura de modo rápido de sesión y responde `Fast mode enabled.` / `Fast mode disabled.`.
- Envía `/fast` (o `/fast status`) sin modo para ver el estado efectivo actual del modo rápido.
- OpenClaw resuelve el modo rápido en este orden:
  1. `/fast on|off` en línea/solo directiva
  2. Sobrescritura de sesión
  3. Predeterminado por agente (`agents.list[].fastModeDefault`)
  4. Configuración por modelo: `agents.defaults.models["<provider>/<model>"].params.fastMode`
  5. Respaldo: `off`
- Para `openai/*`, el modo rápido se asigna al procesamiento prioritario de OpenAI enviando `service_tier=priority` en las solicitudes Responses compatibles.
- Para `openai-codex/*`, el modo rápido envía la misma marca `service_tier=priority` en Codex Responses. OpenClaw mantiene un único interruptor compartido de `/fast` en ambas rutas de autenticación.
- Para solicitudes públicas directas de `anthropic/*`, incluido el tráfico autenticado por OAuth enviado a `api.anthropic.com`, el modo rápido se asigna a niveles de servicio de Anthropic: `/fast on` establece `service_tier=auto`, `/fast off` establece `service_tier=standard_only`.
- Para `minimax/*` en la ruta compatible con Anthropic, `/fast on` (o `params.fastMode: true`) reescribe `MiniMax-M2.7` a `MiniMax-M2.7-highspeed`.
- Los parámetros explícitos del modelo Anthropic `serviceTier` / `service_tier` sobrescriben el valor predeterminado del modo rápido cuando ambos están establecidos. OpenClaw sigue omitiendo la inyección de nivel de servicio de Anthropic para URL base proxy que no sean de Anthropic.

## Directivas de detalle (/verbose o /v)

- Niveles: `on` (mínimo) | `full` | `off` (predeterminado).
- Un mensaje que contenga solo la directiva activa el modo detallado de sesión y responde `Verbose logging enabled.` / `Verbose logging disabled.`; los niveles no válidos devuelven una pista sin cambiar el estado.
- `/verbose off` almacena una sobrescritura explícita de sesión; bórrala mediante la interfaz de sesiones eligiendo `inherit`.
- La directiva en línea afecta solo a ese mensaje; en caso contrario se aplican los valores predeterminados de sesión/globales.
- Envía `/verbose` (o `/verbose:`) sin argumento para ver el nivel de detalle actual.
- Cuando el modo detallado está activado, los agentes que emiten resultados estructurados de herramientas (Pi, otros agentes JSON) devuelven cada llamada de herramienta como su propio mensaje solo de metadatos, con el prefijo `<emoji> <tool-name>: <arg>` cuando está disponible (ruta/comando). Estos resúmenes de herramientas se envían tan pronto como se inicia cada herramienta (burbujas separadas), no como deltas de streaming.
- Los resúmenes de fallo de herramientas siguen siendo visibles en modo normal, pero los sufijos de detalle de error sin procesar se ocultan a menos que verbose sea `on` o `full`.
- Cuando verbose es `full`, las salidas de las herramientas también se reenvían tras completarse (burbuja separada, truncada a una longitud segura). Si cambias `/verbose on|full|off` mientras una ejecución está en curso, las burbujas de herramientas posteriores respetarán la nueva configuración.

## Visibilidad del razonamiento (/reasoning)

- Niveles: `on|off|stream`.
- Un mensaje que contenga solo la directiva activa o desactiva si se muestran bloques de razonamiento en las respuestas.
- Cuando está habilitado, el razonamiento se envía como un **mensaje separado** con el prefijo `Reasoning:`.
- `stream` (solo Telegram): transmite el razonamiento a la burbuja de borrador de Telegram mientras se genera la respuesta y luego envía la respuesta final sin razonamiento.
- Alias: `/reason`.
- Envía `/reasoning` (o `/reasoning:`) sin argumento para ver el nivel de razonamiento actual.
- Orden de resolución: directiva en línea, luego sobrescritura de sesión, luego predeterminado por agente (`agents.list[].reasoningDefault`) y luego respaldo (`off`).

## Relacionado

- La documentación del modo elevado está en [Modo elevado](/tools/elevated).

## Heartbeats

- El cuerpo de la sonda de heartbeat es el prompt de heartbeat configurado (predeterminado: `Read HEARTBEAT.md if it exists (workspace context). Follow it strictly. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.`). Las directivas en línea en un mensaje de heartbeat se aplican como de costumbre (pero evita cambiar los valores predeterminados de sesión desde los heartbeats).
- La entrega del heartbeat usa por defecto solo la carga útil final. Para enviar también el mensaje separado `Reasoning:` (cuando esté disponible), establece `agents.defaults.heartbeat.includeReasoning: true` o por agente `agents.list[].heartbeat.includeReasoning: true`.

## IU de chat web

- El selector de razonamiento del chat web refleja el nivel almacenado de la sesión desde el almacén/configuración de sesión entrante cuando se carga la página.
- Elegir otro nivel escribe la sobrescritura de sesión inmediatamente mediante `sessions.patch`; no espera al siguiente envío y no es una sobrescritura de un solo uso `thinkingOnce`.
- La primera opción es siempre `Default (<resolved level>)`, donde el valor predeterminado resuelto proviene del modelo activo de la sesión: `adaptive` para Claude 4.6 en Anthropic/Bedrock, `low` para otros modelos con capacidad de razonamiento y `off` en caso contrario.
- El selector sigue teniendo en cuenta el proveedor:
  - la mayoría de los proveedores muestran `off | minimal | low | medium | high | adaptive`
  - Z.AI muestra `off | on` binario
- `/think:<level>` sigue funcionando y actualiza el mismo nivel de sesión almacenado, por lo que las directivas de chat y el selector permanecen sincronizados.
