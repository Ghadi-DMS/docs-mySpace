---
read_when:
    - Quieres trabajo en segundo plano o en paralelo mediante el agente
    - Estás cambiando `sessions_spawn` o la política de herramientas de subagentes
    - Estás implementando o solucionando problemas de sesiones de subagentes vinculadas a hilos
summary: 'Subagentes: ejecución de agentes aislados que anuncian los resultados de vuelta al chat solicitante'
title: Subagentes
x-i18n:
    generated_at: "2026-04-05T12:57:23Z"
    model: gpt-5.4
    provider: openai
    source_hash: 9df7cc35a3069ce4eb9c92a95df3ce5365a00a3fae92ff73def75461b58fec3f
    source_path: tools/subagents.md
    workflow: 15
---

# Subagentes

Los subagentes son ejecuciones de agentes en segundo plano generadas a partir de una ejecución de agente existente. Se ejecutan en su propia sesión (`agent:<agentId>:subagent:<uuid>`) y, cuando terminan, **anuncian** su resultado de vuelta al canal de chat solicitante. Cada ejecución de subagente se registra como una [tarea en segundo plano](/es/automation/tasks).

## Comando slash

Usa `/subagents` para inspeccionar o controlar ejecuciones de subagentes para la **sesión actual**:

- `/subagents list`
- `/subagents kill <id|#|all>`
- `/subagents log <id|#> [limit] [tools]`
- `/subagents info <id|#>`
- `/subagents send <id|#> <message>`
- `/subagents steer <id|#> <message>`
- `/subagents spawn <agentId> <task> [--model <model>] [--thinking <level>]`

Controles de vinculación a hilos:

Estos comandos funcionan en canales que admiten vinculaciones persistentes a hilos. Consulta **Canales compatibles con hilos** más abajo.

- `/focus <subagent-label|session-key|session-id|session-label>`
- `/unfocus`
- `/agents`
- `/session idle <duration|off>`
- `/session max-age <duration|off>`

`/subagents info` muestra metadatos de la ejecución (estado, marcas de tiempo, id de sesión, ruta de la transcripción, limpieza).
Usa `sessions_history` para una vista de recuperación acotada y filtrada por seguridad; inspecciona la
ruta de la transcripción en disco cuando necesites la transcripción completa sin procesar.

### Comportamiento de spawn

`/subagents spawn` inicia un subagente en segundo plano como un comando de usuario, no como un relé interno, y envía una actualización final de finalización de vuelta al chat solicitante cuando la ejecución termina.

- El comando de spawn no bloquea; devuelve inmediatamente un id de ejecución.
- Al completarse, el subagente anuncia un mensaje de resumen/resultado de vuelta al canal de chat solicitante.
- La entrega de finalización está basada en envío. Una vez generado, no hagas sondeos en bucle a `/subagents list`,
  `sessions_list` o `sessions_history` solo para esperar a que
  termine; inspecciona el estado solo bajo demanda para depuración o intervención.
- Al completarse, OpenClaw cierra en la medida de lo posible las pestañas/procesos de navegador rastreados abiertos por esa sesión de subagente antes de que continúe el flujo de limpieza del anuncio.
- Para spawns manuales, la entrega es resistente:
  - OpenClaw primero intenta la entrega directa con `agent` usando una clave de idempotencia estable.
  - Si la entrega directa falla, recurre al enrutamiento por cola.
  - Si el enrutamiento por cola sigue sin estar disponible, el anuncio se reintenta con un retroceso exponencial corto antes del abandono final.
- La entrega de finalización conserva la ruta del solicitante resuelta:
  - las rutas de finalización vinculadas a hilos o conversaciones tienen prioridad cuando están disponibles
  - si el origen de finalización solo proporciona un canal, OpenClaw completa el destino/cuenta que falta usando la ruta resuelta de la sesión solicitante (`lastChannel` / `lastTo` / `lastAccountId`) para que la entrega directa siga funcionando
- La transferencia de finalización a la sesión solicitante es contexto interno generado en runtime (no texto redactado por el usuario) e incluye:
  - `Result` (el texto de respuesta `assistant` visible más reciente, o en su defecto el texto más reciente saneado de tool/toolResult)
  - `Status` (`completed successfully` / `failed` / `timed out` / `unknown`)
  - estadísticas compactas de runtime/tokens
  - una instrucción de entrega que indica al agente solicitante que reescriba en voz normal de asistente (no reenviar metadatos internos sin procesar)
- `--model` y `--thinking` anulan los valores predeterminados para esa ejecución específica.
- Usa `info`/`log` para inspeccionar detalles y salida después de la finalización.
- `/subagents spawn` es modo de una sola ejecución (`mode: "run"`). Para sesiones persistentes vinculadas a hilos, usa `sessions_spawn` con `thread: true` y `mode: "session"`.
- Para sesiones del arnés ACP (Codex, Claude Code, Gemini CLI), usa `sessions_spawn` con `runtime: "acp"` y consulta [Agentes ACP](/tools/acp-agents).

Objetivos principales:

- Paralelizar trabajo de “investigación / tarea larga / herramienta lenta” sin bloquear la ejecución principal.
- Mantener a los subagentes aislados de forma predeterminada (separación de sesiones + sandboxing opcional).
- Hacer que la superficie de herramientas sea difícil de usar incorrectamente: los subagentes **no** reciben herramientas de sesión de forma predeterminada.
- Admitir profundidad de anidamiento configurable para patrones de orquestación.

Nota sobre costo: cada subagente tiene su **propio** contexto y uso de tokens. Para tareas pesadas o repetitivas,
establece un modelo más barato para los subagentes y mantén tu agente principal en un modelo de mayor calidad.
Puedes configurarlo mediante `agents.defaults.subagents.model` o anulaciones por agente.

## Herramienta

Usa `sessions_spawn`:

- Inicia una ejecución de subagente (`deliver: false`, carril global: `subagent`)
- Luego ejecuta un paso de anuncio y publica la respuesta de anuncio en el canal de chat solicitante
- Modelo predeterminado: hereda del llamador salvo que establezcas `agents.defaults.subagents.model` (o `agents.list[].subagents.model` por agente); un `sessions_spawn.model` explícito sigue teniendo prioridad.
- Thinking predeterminado: hereda del llamador salvo que establezcas `agents.defaults.subagents.thinking` (o `agents.list[].subagents.thinking` por agente); un `sessions_spawn.thinking` explícito sigue teniendo prioridad.
- Tiempo de espera predeterminado de la ejecución: si se omite `sessions_spawn.runTimeoutSeconds`, OpenClaw usa `agents.defaults.subagents.runTimeoutSeconds` cuando está configurado; en caso contrario recurre a `0` (sin tiempo de espera).

Parámetros de la herramienta:

- `task` (obligatorio)
- `label?` (opcional)
- `agentId?` (opcional; generar bajo otro id de agente si está permitido)
- `model?` (opcional; anula el modelo del subagente; los valores no válidos se omiten y el subagente se ejecuta con el modelo predeterminado con una advertencia en el resultado de la herramienta)
- `thinking?` (opcional; anula el nivel de thinking para la ejecución del subagente)
- `runTimeoutSeconds?` (usa por defecto `agents.defaults.subagents.runTimeoutSeconds` cuando está configurado; en caso contrario `0`; cuando se establece, la ejecución del subagente se aborta tras N segundos)
- `thread?` (predeterminado `false`; cuando es `true`, solicita vinculación del hilo del canal para esta sesión de subagente)
- `mode?` (`run|session`)
  - el valor predeterminado es `run`
  - si `thread: true` y se omite `mode`, el valor predeterminado pasa a ser `session`
  - `mode: "session"` requiere `thread: true`
- `cleanup?` (`delete|keep`, predeterminado `keep`)
- `sandbox?` (`inherit|require`, predeterminado `inherit`; `require` rechaza el spawn salvo que el runtime hijo de destino esté en sandbox)
- `sessions_spawn` **no** acepta parámetros de entrega por canal (`target`, `channel`, `to`, `threadId`, `replyTo`, `transport`). Para la entrega, usa `message`/`sessions_send` desde la ejecución generada.

## Sesiones vinculadas a hilos

Cuando las vinculaciones a hilos están habilitadas para un canal, un subagente puede permanecer vinculado a un hilo para que los mensajes de usuario de seguimiento en ese hilo sigan enrutándose a la misma sesión de subagente.

### Canales compatibles con hilos

- Discord (actualmente el único canal compatible): admite sesiones persistentes de subagentes vinculadas a hilos (`sessions_spawn` con `thread: true`), controles manuales de hilos (`/focus`, `/unfocus`, `/agents`, `/session idle`, `/session max-age`) y claves de adaptador `channels.discord.threadBindings.enabled`, `channels.discord.threadBindings.idleHours`, `channels.discord.threadBindings.maxAgeHours` y `channels.discord.threadBindings.spawnSubagentSessions`.

Flujo rápido:

1. Genera con `sessions_spawn` usando `thread: true` (y opcionalmente `mode: "session"`).
2. OpenClaw crea o vincula un hilo a ese destino de sesión en el canal activo.
3. Las respuestas y mensajes de seguimiento en ese hilo se enrutan a la sesión vinculada.
4. Usa `/session idle` para inspeccionar/actualizar el desenfoque automático por inactividad y `/session max-age` para controlar el límite estricto.
5. Usa `/unfocus` para desvincular manualmente.

Controles manuales:

- `/focus <target>` vincula el hilo actual (o crea uno) a un destino de subagente/sesión.
- `/unfocus` elimina la vinculación del hilo actualmente vinculado.
- `/agents` muestra las ejecuciones activas y el estado de la vinculación (`thread:<id>` o `unbound`).
- `/session idle` y `/session max-age` solo funcionan para hilos vinculados enfocados.

Interruptores de configuración:

- Valor predeterminado global: `session.threadBindings.enabled`, `session.threadBindings.idleHours`, `session.threadBindings.maxAgeHours`
- Las claves de anulación por canal y de vinculación automática de spawn son específicas del adaptador. Consulta **Canales compatibles con hilos** más arriba.

Consulta [Referencia de configuración](/es/gateway/configuration-reference) y [Comandos slash](/tools/slash-commands) para obtener detalles actuales del adaptador.

Allowlist:

- `agents.list[].subagents.allowAgents`: lista de ids de agentes a los que se puede apuntar mediante `agentId` (`["*"]` para permitir cualquiera). Valor predeterminado: solo el agente solicitante.
- `agents.defaults.subagents.allowAgents`: allowlist predeterminada de agentes de destino usada cuando el agente solicitante no establece su propia `subagents.allowAgents`.
- Protección de herencia de sandbox: si la sesión solicitante está en sandbox, `sessions_spawn` rechaza destinos que se ejecutarían sin sandbox.
- `agents.defaults.subagents.requireAgentId` / `agents.list[].subagents.requireAgentId`: cuando es true, bloquea llamadas `sessions_spawn` que omiten `agentId` (fuerza selección explícita de perfil). Valor predeterminado: false.

Descubrimiento:

- Usa `agents_list` para ver qué ids de agentes están permitidos actualmente para `sessions_spawn`.

Archivado automático:

- Las sesiones de subagentes se archivan automáticamente tras `agents.defaults.subagents.archiveAfterMinutes` (predeterminado: 60).
- El archivado usa `sessions.delete` y renombra la transcripción a `*.deleted.<timestamp>` (misma carpeta).
- `cleanup: "delete"` archiva inmediatamente después del anuncio (pero conserva la transcripción mediante cambio de nombre).
- El archivado automático es una operación en la medida de lo posible; los temporizadores pendientes se pierden si el gateway se reinicia.
- `runTimeoutSeconds` **no** archiva automáticamente; solo detiene la ejecución. La sesión permanece hasta el archivado automático.
- El archivado automático se aplica por igual a sesiones de profundidad 1 y profundidad 2.
- La limpieza del navegador es independiente de la limpieza de archivo: las pestañas/procesos del navegador rastreados se cierran en la medida de lo posible cuando termina la ejecución, incluso si se conserva el registro de la transcripción/sesión.

## Subagentes anidados

De forma predeterminada, los subagentes no pueden generar sus propios subagentes (`maxSpawnDepth: 1`). Puedes habilitar un nivel de anidamiento estableciendo `maxSpawnDepth: 2`, lo que permite el **patrón de orquestación**: principal → subagente orquestador → sub-subagentes trabajadores.

### Cómo habilitarlo

```json5
{
  agents: {
    defaults: {
      subagents: {
        maxSpawnDepth: 2, // allow sub-agents to spawn children (default: 1)
        maxChildrenPerAgent: 5, // max active children per agent session (default: 5)
        maxConcurrent: 8, // global concurrency lane cap (default: 8)
        runTimeoutSeconds: 900, // default timeout for sessions_spawn when omitted (0 = no timeout)
      },
    },
  },
}
```

### Niveles de profundidad

| Profundidad | Forma de la clave de sesión                  | Rol                                           | ¿Puede generar?              |
| ----------- | -------------------------------------------- | --------------------------------------------- | ---------------------------- |
| 0           | `agent:<id>:main`                            | Agente principal                              | Siempre                      |
| 1           | `agent:<id>:subagent:<uuid>`                 | Subagente (orquestador cuando se permite profundidad 2) | Solo si `maxSpawnDepth >= 2` |
| 2           | `agent:<id>:subagent:<uuid>:subagent:<uuid>` | Sub-subagente (trabajador hoja)               | Nunca                        |

### Cadena de anuncios

Los resultados fluyen de vuelta por la cadena:

1. El trabajador de profundidad 2 termina → anuncia a su padre (orquestador de profundidad 1)
2. El orquestador de profundidad 1 recibe el anuncio, sintetiza resultados, termina → anuncia al principal
3. El agente principal recibe el anuncio y lo entrega al usuario

Cada nivel solo ve anuncios de sus hijos directos.

Guía operativa:

- Inicia el trabajo hijo una sola vez y espera eventos de finalización en lugar de crear bucles de sondeo
  alrededor de `sessions_list`, `sessions_history`, `/subagents list` o
  comandos `exec` con sleep.
- Si llega un evento de finalización hijo después de que ya enviaste la respuesta final,
  el seguimiento correcto es el token silencioso exacto `NO_REPLY` / `no_reply`.

### Política de herramientas por profundidad

- El rol y el alcance de control se escriben en los metadatos de la sesión en el momento del spawn. Eso evita que las claves de sesión planas o restauradas recuperen accidentalmente privilegios de orquestador.
- **Profundidad 1 (orquestador, cuando `maxSpawnDepth >= 2`)**: obtiene `sessions_spawn`, `subagents`, `sessions_list`, `sessions_history` para poder gestionar sus hijos. Otras herramientas de sesión/sistema siguen denegadas.
- **Profundidad 1 (hoja, cuando `maxSpawnDepth == 1`)**: sin herramientas de sesión (comportamiento predeterminado actual).
- **Profundidad 2 (trabajador hoja)**: sin herramientas de sesión — `sessions_spawn` siempre está denegado en profundidad 2. No puede generar más hijos.

### Límite de spawn por agente

Cada sesión de agente (a cualquier profundidad) puede tener como máximo `maxChildrenPerAgent` (predeterminado: 5) hijos activos al mismo tiempo. Esto evita el fan-out descontrolado de un único orquestador.

### Detención en cascada

Detener un orquestador de profundidad 1 detiene automáticamente todos sus hijos de profundidad 2:

- `/stop` en el chat principal detiene todos los agentes de profundidad 1 y se propaga a sus hijos de profundidad 2.
- `/subagents kill <id>` detiene un subagente específico y se propaga a sus hijos.
- `/subagents kill all` detiene todos los subagentes del solicitante y se propaga.

## Autenticación

La autenticación del subagente se resuelve por **id de agente**, no por tipo de sesión:

- La clave de sesión del subagente es `agent:<agentId>:subagent:<uuid>`.
- El almacén de autenticación se carga desde el `agentDir` de ese agente.
- Los perfiles de autenticación del agente principal se fusionan como **respaldo**; los perfiles del agente anulan a los del principal en caso de conflicto.

Nota: la fusión es aditiva, por lo que los perfiles del principal siempre están disponibles como respaldo. La autenticación totalmente aislada por agente todavía no se admite.

## Anuncio

Los subagentes informan de vuelta mediante un paso de anuncio:

- El paso de anuncio se ejecuta dentro de la sesión del subagente (no en la sesión del solicitante).
- Si el subagente responde exactamente `ANNOUNCE_SKIP`, no se publica nada.
- Si el texto más reciente del asistente es el token silencioso exacto `NO_REPLY` / `no_reply`,
  la salida del anuncio se suprime aunque existiera progreso visible previo.
- En caso contrario, la entrega depende de la profundidad del solicitante:
  - las sesiones solicitantes de nivel superior usan una llamada de seguimiento `agent` con entrega externa (`deliver=true`)
  - las sesiones solicitantes de subagentes anidados reciben una inyección interna de seguimiento (`deliver=false`) para que el orquestador pueda sintetizar resultados de hijos dentro de la sesión
  - si una sesión solicitante de subagente anidado ya no existe, OpenClaw recurre al solicitante de esa sesión cuando está disponible
- Para sesiones solicitantes de nivel superior, la entrega directa en modo finalización primero resuelve cualquier ruta vinculada de conversación/hilo y anulación de hook, luego completa los campos de destino de canal que faltan desde la ruta almacenada de la sesión solicitante. Eso mantiene las finalizaciones en el chat/tema correcto incluso cuando el origen de finalización solo identifica el canal.
- La agregación de finalización de hijos se limita a la ejecución solicitante actual al construir hallazgos de finalización anidados, lo que evita que salidas antiguas de hijos de ejecuciones previas se filtren a la finalización actual.
- Las respuestas de anuncio conservan el enrutamiento de hilo/tema cuando está disponible en los adaptadores de canal.
- El contexto de anuncio se normaliza en un bloque de evento interno estable:
  - origen (`subagent` o `cron`)
  - clave/id de la sesión hija
  - tipo de anuncio + etiqueta de tarea
  - línea de estado derivada del resultado del runtime (`success`, `error`, `timeout` o `unknown`)
  - contenido del resultado seleccionado del último texto visible del asistente, o en su defecto texto saneado del último tool/toolResult
  - una instrucción de seguimiento que describe cuándo responder frente a permanecer en silencio
- `Status` no se infiere a partir de la salida del modelo; proviene de señales del resultado del runtime.
- En caso de tiempo de espera, si el hijo solo llegó a completar llamadas de herramientas, el anuncio puede condensar ese historial en un breve resumen de progreso parcial en lugar de reproducir salida sin procesar de herramientas.

Las cargas útiles del anuncio incluyen una línea de estadísticas al final (incluso cuando van envueltas):

- Runtime (por ejemplo, `runtime 5m12s`)
- Uso de tokens (entrada/salida/total)
- Costo estimado cuando el precio del modelo está configurado (`models.providers.*.models[].cost`)
- `sessionKey`, `sessionId` y ruta de la transcripción (para que el agente principal pueda recuperar el historial mediante `sessions_history` o inspeccionar el archivo en disco)
- Los metadatos internos están pensados solo para orquestación; las respuestas orientadas al usuario deben reescribirse en voz normal de asistente.

`sessions_history` es la ruta de orquestación más segura:

- la recuperación del asistente se normaliza primero:
  - se eliminan las etiquetas de thinking
  - se eliminan los bloques de andamiaje `<relevant-memories>` / `<relevant_memories>`
  - se eliminan bloques de carga útil XML de llamada a herramientas en texto plano, como `<tool_call>...</tool_call>`,
    `<function_call>...</function_call>`, `<tool_calls>...</tool_calls>` y
    `<function_calls>...</function_calls>`, incluidas las
    cargas truncadas que nunca se cierran limpiamente
  - se elimina el andamiaje degradado de llamada/resultado de herramientas y los marcadores de contexto histórico
  - se eliminan tokens filtrados de control del modelo como `<|assistant|>`, otros tokens ASCII
    `<|...|>` y variantes de ancho completo `<｜...｜>`
  - se elimina XML malformado de llamada a herramientas de MiniMax
- el texto parecido a credenciales/tokens se redacta
- los bloques largos pueden truncarse
- los historiales muy grandes pueden eliminar filas antiguas o sustituir una fila sobredimensionada por
  `[sessions_history omitted: message too large]`
- la inspección de la transcripción sin procesar en disco es la alternativa cuando necesitas la transcripción completa byte por byte

## Política de herramientas (herramientas de subagente)

De forma predeterminada, los subagentes obtienen **todas las herramientas excepto las herramientas de sesión** y las herramientas del sistema:

- `sessions_list`
- `sessions_history`
- `sessions_send`
- `sessions_spawn`

Aquí también `sessions_history` sigue siendo una vista de recuperación acotada y saneada; no es
un volcado de transcripción sin procesar.

Cuando `maxSpawnDepth >= 2`, los subagentes orquestadores de profundidad 1 reciben además `sessions_spawn`, `subagents`, `sessions_list` y `sessions_history` para poder gestionar sus hijos.

Anúlalo mediante configuración:

```json5
{
  agents: {
    defaults: {
      subagents: {
        maxConcurrent: 1,
      },
    },
  },
  tools: {
    subagents: {
      tools: {
        // deny wins
        deny: ["gateway", "cron"],
        // if allow is set, it becomes allow-only (deny still wins)
        // allow: ["read", "exec", "process"]
      },
    },
  },
}
```

## Concurrencia

Los subagentes usan un carril de cola dedicado dentro del proceso:

- Nombre del carril: `subagent`
- Concurrencia: `agents.defaults.subagents.maxConcurrent` (predeterminado `8`)

## Detención

- Enviar `/stop` en el chat solicitante aborta la sesión solicitante y detiene cualquier ejecución activa de subagente generada a partir de ella, propagándose a hijos anidados.
- `/subagents kill <id>` detiene un subagente específico y se propaga a sus hijos.

## Limitaciones

- El anuncio del subagente es **best-effort**. Si el gateway se reinicia, se pierde el trabajo pendiente de “anunciar de vuelta”.
- Los subagentes siguen compartiendo los mismos recursos del proceso del gateway; trata `maxConcurrent` como una válvula de seguridad.
- `sessions_spawn` siempre es no bloqueante: devuelve `{ status: "accepted", runId, childSessionKey }` inmediatamente.
- El contexto del subagente solo inyecta `AGENTS.md` + `TOOLS.md` (no `SOUL.md`, `IDENTITY.md`, `USER.md`, `HEARTBEAT.md` ni `BOOTSTRAP.md`).
- La profundidad máxima de anidamiento es 5 (rango de `maxSpawnDepth`: 1–5). Se recomienda profundidad 2 para la mayoría de los casos de uso.
- `maxChildrenPerAgent` limita los hijos activos por sesión (predeterminado: 5, rango: 1–20).
