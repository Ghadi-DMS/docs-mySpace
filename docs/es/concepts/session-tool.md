---
read_when:
    - Quieres entender qué herramientas de sesión tiene el agente
    - Quieres configurar el acceso entre sesiones o la creación de subagentes
    - Quieres inspeccionar el estado o controlar subagentes creados
summary: Herramientas del agente para estado entre sesiones, recuerdo, mensajería y orquestación de subagentes
title: Herramientas de sesión
x-i18n:
    generated_at: "2026-04-05T12:41:01Z"
    model: gpt-5.4
    provider: openai
    source_hash: 77fab7cbf9d1a5cccaf316b69fefe212bbf9370876c8b92e988d3175f5545a4d
    source_path: concepts/session-tool.md
    workflow: 15
---

# Herramientas de sesión

OpenClaw proporciona a los agentes herramientas para trabajar entre sesiones, inspeccionar el estado y
orquestar subagentes.

## Herramientas disponibles

| Herramienta         | Qué hace                                                                    |
| ------------------- | --------------------------------------------------------------------------- |
| `sessions_list`    | Enumera sesiones con filtros opcionales (tipo, antigüedad reciente)         |
| `sessions_history` | Lee la transcripción de una sesión específica                               |
| `sessions_send`    | Envía un mensaje a otra sesión y opcionalmente espera                       |
| `sessions_spawn`   | Crea una sesión aislada de subagente para trabajo en segundo plano          |
| `sessions_yield`   | Finaliza el turno actual y espera resultados de seguimiento de subagentes   |
| `subagents`        | Enumera, dirige o elimina subagentes creados para esta sesión               |
| `session_status`   | Muestra una tarjeta tipo `/status` y opcionalmente establece una anulación de modelo por sesión |

## Enumerar y leer sesiones

`sessions_list` devuelve sesiones con su clave, tipo, canal, modelo, conteos de
tokens y marcas de tiempo. Filtra por tipo (`main`, `group`, `cron`, `hook`,
`node`) o por antigüedad reciente (`activeMinutes`).

`sessions_history` obtiene la transcripción de la conversación para una sesión específica.
De forma predeterminada, se excluyen los resultados de herramientas; pasa `includeTools: true` para verlos.
La vista devuelta está intencionalmente limitada y filtrada por seguridad:

- el texto del asistente se normaliza antes del recuerdo:
  - se eliminan las etiquetas de pensamiento
  - se eliminan los bloques de andamiaje `<relevant-memories>` / `<relevant_memories>`
  - se eliminan bloques XML de carga útil de llamada a herramienta en texto sin formato como `<tool_call>...</tool_call>`,
    `<function_call>...</function_call>`, `<tool_calls>...</tool_calls>` y
    `<function_calls>...</function_calls>`, incluidas las cargas útiles truncadas
    que nunca se cierran correctamente
  - se elimina el andamiaje degradado de llamada/resultado de herramienta como `[Tool Call: ...]`,
    `[Tool Result ...]` y `[Historical context ...]`
  - se eliminan los tokens de control del modelo filtrados como `<|assistant|>`, otros tokens ASCII
    `<|...|>` y variantes de ancho completo `<｜...｜>`
  - se elimina XML malformado de llamada a herramienta de MiniMax como `<invoke ...>` /
    `</minimax:tool_call>`
- el texto con apariencia de credencial/token se redacta antes de devolverse
- los bloques de texto largos se truncan
- los historiales muy grandes pueden descartar filas antiguas o reemplazar una fila sobredimensionada por
  `[sessions_history omitted: message too large]`
- la herramienta informa indicadores de resumen como `truncated`, `droppedMessages`,
  `contentTruncated`, `contentRedacted` y `bytes`

Ambas herramientas aceptan una **clave de sesión** (como `"main"`) o un **ID de sesión**
de una llamada previa a list.

Si necesitas la transcripción exacta byte por byte, inspecciona el archivo de transcripción en
disco en lugar de tratar `sessions_history` como un volcado sin procesar.

## Enviar mensajes entre sesiones

`sessions_send` entrega un mensaje a otra sesión y opcionalmente espera
la respuesta:

- **Enviar y continuar:** establece `timeoutSeconds: 0` para encolar y devolver
  inmediatamente.
- **Esperar respuesta:** establece un tiempo de espera y obtén la respuesta en línea.

Después de que el destino responda, OpenClaw puede ejecutar un **bucle de respuesta**
en el que los agentes alternan mensajes (hasta 5 turnos). El agente de destino puede responder
`REPLY_SKIP` para detenerse antes.

## Ayudas de estado y orquestación

`session_status` es la herramienta ligera equivalente a `/status` para la sesión actual
u otra sesión visible. Informa uso, tiempo, estado del modelo/runtime y
contexto vinculado de tarea en segundo plano cuando está presente. Al igual que `/status`, puede completar
contadores dispersos de tokens/caché desde la última entrada de uso de la transcripción, y
`model=default` borra una anulación por sesión.

`sessions_yield` finaliza intencionalmente el turno actual para que el siguiente mensaje pueda ser
el evento de seguimiento que estás esperando. Úsalo después de crear subagentes cuando
quieras que los resultados de finalización lleguen como el siguiente mensaje en lugar de construir
bucles de sondeo.

`subagents` es la herramienta de plano de control para subagentes de OpenClaw ya
creados. Admite:

- `action: "list"` para inspeccionar ejecuciones activas/recientes
- `action: "steer"` para enviar orientación de seguimiento a un hijo en ejecución
- `action: "kill"` para detener un hijo o `all`

## Crear subagentes

`sessions_spawn` crea una sesión aislada para una tarea en segundo plano. Siempre es
no bloqueante: devuelve inmediatamente con un `runId` y `childSessionKey`.

Opciones clave:

- `runtime: "subagent"` (predeterminado) o `"acp"` para agentes de harness externos.
- Anulaciones de `model` y `thinking` para la sesión hija.
- `thread: true` para vincular la creación a un hilo de chat (Discord, Slack, etc.).
- `sandbox: "require"` para exigir sandboxing en el hijo.

Los subagentes hoja predeterminados no reciben herramientas de sesión. Cuando
`maxSpawnDepth >= 2`, los subagentes orquestadores de profundidad 1 reciben además
`sessions_spawn`, `subagents`, `sessions_list` y `sessions_history` para que
puedan gestionar sus propios hijos. Las ejecuciones hoja siguen sin recibir
herramientas de orquestación recursiva.

Tras completarse, un paso de anuncio publica el resultado en el canal del solicitante.
La entrega de finalización conserva el enrutamiento vinculado de hilo/tema cuando está disponible, y si
el origen de la finalización solo identifica un canal, OpenClaw aún puede reutilizar la ruta
almacenada de la sesión del solicitante (`lastChannel` / `lastTo`) para entrega
directa.

Para el comportamiento específico de ACP, consulta [Agentes ACP](/tools/acp-agents).

## Visibilidad

Las herramientas de sesión tienen alcance limitado para restringir lo que el agente puede ver:

| Nivel   | Alcance                                    |
| ------- | ------------------------------------------ |
| `self`  | Solo la sesión actual                      |
| `tree`  | Sesión actual + subagentes creados         |
| `agent` | Todas las sesiones de este agente          |
| `all`   | Todas las sesiones (entre agentes si está configurado) |

El valor predeterminado es `tree`. Las sesiones en sandbox quedan limitadas a `tree` independientemente de la
configuración.

## Lecturas adicionales

- [Gestión de sesiones](/concepts/session) -- enrutamiento, ciclo de vida, mantenimiento
- [Agentes ACP](/tools/acp-agents) -- creación con harness externo
- [Multi-agent](/concepts/multi-agent) -- arquitectura multiagente
- [Configuración del Gateway](/gateway/configuration) -- parámetros de configuración de herramientas de sesión
