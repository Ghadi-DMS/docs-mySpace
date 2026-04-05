---
read_when:
    - Necesitas un recorrido exacto del bucle del agente o de los eventos del ciclo de vida
summary: Ciclo de vida del bucle del agente, streams y semántica de espera
title: Bucle del agente
x-i18n:
    generated_at: "2026-04-05T12:39:34Z"
    model: gpt-5.4
    provider: openai
    source_hash: 8e562e63c494881e9c345efcb93c5f972d69aaec61445afc3d4ad026b2d26883
    source_path: concepts/agent-loop.md
    workflow: 15
---

# Bucle del agente (OpenClaw)

Un bucle agéntico es la ejecución completa y “real” de un agente: entrada → ensamblaje de contexto → inferencia del modelo →
ejecución de herramientas → respuestas en streaming → persistencia. Es la ruta autoritativa que convierte un mensaje
en acciones y una respuesta final, mientras mantiene coherente el estado de la sesión.

En OpenClaw, un bucle es una única ejecución serializada por sesión que emite eventos de ciclo de vida y de stream
mientras el modelo piensa, llama herramientas y transmite salida. Este documento explica cómo está
conectado de extremo a extremo ese bucle auténtico.

## Puntos de entrada

- RPC del Gateway: `agent` y `agent.wait`.
- CLI: comando `agent`.

## Cómo funciona (alto nivel)

1. El RPC `agent` valida parámetros, resuelve la sesión (`sessionKey`/`sessionId`), conserva los metadatos de la sesión y devuelve `{ runId, acceptedAt }` de inmediato.
2. `agentCommand` ejecuta el agente:
   - resuelve el modelo y los valores predeterminados de thinking/verbose
   - carga la instantánea de Skills
   - llama a `runEmbeddedPiAgent` (tiempo de ejecución de pi-agent-core)
   - emite **lifecycle end/error** si el bucle integrado no emite uno
3. `runEmbeddedPiAgent`:
   - serializa ejecuciones mediante colas por sesión y una cola global
   - resuelve el perfil de modelo y autenticación y construye la sesión de pi
   - se suscribe a eventos de pi y transmite deltas del asistente y de herramientas
   - aplica el tiempo de espera -> aborta la ejecución si se supera
   - devuelve cargas útiles y metadatos de uso
4. `subscribeEmbeddedPiSession` conecta los eventos de pi-agent-core al stream `agent` de OpenClaw:
   - eventos de herramienta => `stream: "tool"`
   - deltas del asistente => `stream: "assistant"`
   - eventos de ciclo de vida => `stream: "lifecycle"` (`phase: "start" | "end" | "error"`)
5. `agent.wait` usa `waitForAgentRun`:
   - espera **lifecycle end/error** para `runId`
   - devuelve `{ status: ok|error|timeout, startedAt, endedAt, error? }`

## Colas y concurrencia

- Las ejecuciones se serializan por clave de sesión (carril de sesión) y, opcionalmente, mediante un carril global.
- Esto evita condiciones de carrera de herramientas o sesiones y mantiene coherente el historial de la sesión.
- Los canales de mensajería pueden elegir modos de cola (collect/steer/followup) que alimentan este sistema de carriles.
  Consulta [Command Queue](/concepts/queue).

## Preparación de sesión y espacio de trabajo

- El espacio de trabajo se resuelve y se crea; las ejecuciones en sandbox pueden redirigirse a una raíz de espacio de trabajo aislada.
- Se cargan Skills (o se reutilizan desde una instantánea) y se inyectan en el entorno y en el prompt.
- Los archivos de bootstrap/contexto se resuelven y se inyectan en el informe del prompt del sistema.
- Se adquiere un bloqueo de escritura de sesión; `SessionManager` se abre y se prepara antes del streaming.

## Ensamblaje del prompt y prompt del sistema

- El prompt del sistema se construye a partir del prompt base de OpenClaw, el prompt de Skills, el contexto de bootstrap y las anulaciones por ejecución.
- Se aplican límites específicos del modelo y reservas de tokens para compactación.
- Consulta [System prompt](/concepts/system-prompt) para ver lo que ve el modelo.

## Puntos de hook (donde puedes interceptar)

OpenClaw tiene dos sistemas de hooks:

- **Hooks internos** (hooks del Gateway): scripts controlados por eventos para comandos y eventos del ciclo de vida.
- **Hooks de plugin**: puntos de extensión dentro del ciclo de vida del agente o herramienta y de la canalización del gateway.

### Hooks internos (hooks del Gateway)

- **`agent:bootstrap`**: se ejecuta mientras se construyen archivos de bootstrap antes de que el prompt del sistema se finalice.
  Úsalo para añadir o eliminar archivos de contexto de bootstrap.
- **Hooks de comandos**: `/new`, `/reset`, `/stop` y otros eventos de comandos (consulta la documentación de Hooks).

Consulta [Hooks](/automation/hooks) para ver configuración y ejemplos.

### Hooks de plugin (ciclo de vida del agente y del gateway)

Se ejecutan dentro del bucle del agente o de la canalización del gateway:

- **`before_model_resolve`**: se ejecuta antes de la sesión (sin `messages`) para anular de forma determinista el proveedor o modelo antes de la resolución del modelo.
- **`before_prompt_build`**: se ejecuta después de cargar la sesión (con `messages`) para inyectar `prependContext`, `systemPrompt`, `prependSystemContext` o `appendSystemContext` antes del envío del prompt. Usa `prependContext` para texto dinámico por turno y campos de contexto del sistema para guía estable que deba situarse en el espacio del prompt del sistema.
- **`before_agent_start`**: hook heredado de compatibilidad que puede ejecutarse en cualquiera de las fases; prefiere los hooks explícitos anteriores.
- **`before_agent_reply`**: se ejecuta después de las acciones en línea y antes de la llamada al LLM, permitiendo que un plugin reclame el turno y devuelva una respuesta sintética o silencie completamente el turno.
- **`agent_end`**: inspecciona la lista final de mensajes y los metadatos de la ejecución tras completarse.
- **`before_compaction` / `after_compaction`**: observan o anotan ciclos de compactación.
- **`before_tool_call` / `after_tool_call`**: interceptan parámetros y resultados de herramientas.
- **`before_install`**: inspecciona hallazgos del análisis integrado y, opcionalmente, bloquea instalaciones de Skills o plugins.
- **`tool_result_persist`**: transforma sincrónicamente resultados de herramientas antes de que se escriban en la transcripción de la sesión.
- **`message_received` / `message_sending` / `message_sent`**: hooks de mensajes entrantes y salientes.
- **`session_start` / `session_end`**: límites del ciclo de vida de la sesión.
- **`gateway_start` / `gateway_stop`**: eventos del ciclo de vida del gateway.

Reglas de decisión de hooks para controles de salida y herramientas:

- `before_tool_call`: `{ block: true }` es terminal y detiene controladores de menor prioridad.
- `before_tool_call`: `{ block: false }` es una operación nula y no elimina un bloqueo previo.
- `before_install`: `{ block: true }` es terminal y detiene controladores de menor prioridad.
- `before_install`: `{ block: false }` es una operación nula y no elimina un bloqueo previo.
- `message_sending`: `{ cancel: true }` es terminal y detiene controladores de menor prioridad.
- `message_sending`: `{ cancel: false }` es una operación nula y no elimina una cancelación previa.

Consulta [Plugin hooks](/plugins/architecture#provider-runtime-hooks) para ver la API de hooks y los detalles de registro.

## Streaming y respuestas parciales

- Los deltas del asistente se transmiten desde pi-agent-core y se emiten como eventos `assistant`.
- El streaming por bloques puede emitir respuestas parciales en `text_end` o `message_end`.
- El streaming de razonamiento puede emitirse como un stream separado o como respuestas por bloques.
- Consulta [Streaming](/concepts/streaming) para ver el comportamiento de fragmentación y respuestas por bloques.

## Ejecución de herramientas y herramientas de mensajería

- Los eventos de inicio, actualización y fin de herramientas se emiten en el stream `tool`.
- Los resultados de herramientas se sanean en cuanto a tamaño y cargas útiles de imágenes antes de registrarse o emitirse.
- Los envíos de herramientas de mensajería se rastrean para suprimir confirmaciones duplicadas del asistente.

## Modelado de la respuesta y supresión

- Las cargas útiles finales se ensamblan a partir de:
  - texto del asistente (y razonamiento opcional)
  - resúmenes de herramientas en línea (cuando verbose + allowed)
  - texto de error del asistente cuando el modelo falla
- El token silencioso exacto `NO_REPLY` / `no_reply` se filtra de las
  cargas útiles salientes.
- Los duplicados de herramientas de mensajería se eliminan de la lista final de cargas útiles.
- Si no quedan cargas útiles representables y una herramienta falló, se emite
  una respuesta alternativa de error de herramienta (a menos que una herramienta de mensajería ya haya enviado una respuesta visible para el usuario).

## Compactación y reintentos

- La compactación automática emite eventos de stream `compaction` y puede activar un reintento.
- En el reintento, se restablecen los búferes en memoria y los resúmenes de herramientas para evitar salida duplicada.
- Consulta [Compaction](/concepts/compaction) para ver la canalización de compactación.

## Streams de eventos (actualmente)

- `lifecycle`: emitido por `subscribeEmbeddedPiSession` (y como respaldo por `agentCommand`)
- `assistant`: deltas transmitidos desde pi-agent-core
- `tool`: eventos de herramientas transmitidos desde pi-agent-core

## Manejo del canal de chat

- Los deltas del asistente se almacenan en búfer en mensajes `delta` del chat.
- Se emite un `final` del chat en **lifecycle end/error**.

## Tiempos de espera

- Predeterminado de `agent.wait`: 30 s (solo la espera). El parámetro `timeoutMs` lo sustituye.
- Tiempo de ejecución del agente: valor predeterminado de `agents.defaults.timeoutSeconds` 172800 s (48 horas); se aplica en el temporizador de aborto de `runEmbeddedPiAgent`.

## Dónde puede terminar antes de tiempo

- Tiempo de espera del agente (aborto)
- AbortSignal (cancelación)
- Desconexión del gateway o tiempo de espera de RPC
- Tiempo de espera de `agent.wait` (solo espera, no detiene al agente)

## Relacionado

- [Tools](/tools) — herramientas disponibles del agente
- [Hooks](/automation/hooks) — scripts controlados por eventos activados por eventos del ciclo de vida del agente
- [Compaction](/concepts/compaction) — cómo se resumen las conversaciones largas
- [Exec Approvals](/tools/exec-approvals) — puertas de aprobación para comandos de shell
- [Thinking](/tools/thinking) — configuración del nivel de thinking/razonamiento
