---
read_when:
    - Necesitas una guía exacta del bucle del agente o de los eventos del ciclo de vida
summary: Ciclo de vida del bucle del agente, flujos y semántica de espera
title: Bucle del agente
x-i18n:
    generated_at: "2026-04-09T01:27:48Z"
    model: gpt-5.4
    provider: openai
    source_hash: 32d3a73df8dabf449211a6183a70dcfd2a9b6f584dc76d0c4c9147582b2ca6a1
    source_path: concepts/agent-loop.md
    workflow: 15
---

# Bucle del agente (OpenClaw)

Un bucle agéntico es la ejecución “real” completa de un agente: entrada → ensamblaje de contexto → inferencia del modelo →
ejecución de herramientas → respuestas en streaming → persistencia. Es la ruta autoritativa que convierte un mensaje
en acciones y una respuesta final, mientras mantiene coherente el estado de la sesión.

En OpenClaw, un bucle es una sola ejecución serializada por sesión que emite eventos de ciclo de vida y de flujo
mientras el modelo piensa, llama herramientas y transmite la salida. Este documento explica cómo se conecta ese bucle auténtico de extremo a extremo.

## Puntos de entrada

- RPC del gateway: `agent` y `agent.wait`.
- CLI: comando `agent`.

## Cómo funciona (visión general)

1. El RPC `agent` valida los parámetros, resuelve la sesión (sessionKey/sessionId), persiste los metadatos de la sesión y devuelve `{ runId, acceptedAt }` de inmediato.
2. `agentCommand` ejecuta el agente:
   - resuelve los valores predeterminados de modelo + thinking/verbose
   - carga la instantánea de Skills
   - llama a `runEmbeddedPiAgent` (runtime de pi-agent-core)
   - emite **lifecycle end/error** si el bucle integrado no emite uno
3. `runEmbeddedPiAgent`:
   - serializa las ejecuciones mediante colas por sesión y una cola global
   - resuelve el perfil de modelo + autenticación y construye la sesión de Pi
   - se suscribe a los eventos de Pi y transmite los deltas del asistente/herramientas
   - aplica el tiempo de espera -> aborta la ejecución si se supera
   - devuelve payloads + metadatos de uso
4. `subscribeEmbeddedPiSession` conecta los eventos de pi-agent-core con el flujo `agent` de OpenClaw:
   - eventos de herramientas => `stream: "tool"`
   - deltas del asistente => `stream: "assistant"`
   - eventos de ciclo de vida => `stream: "lifecycle"` (`phase: "start" | "end" | "error"`)
5. `agent.wait` usa `waitForAgentRun`:
   - espera **lifecycle end/error** para `runId`
   - devuelve `{ status: ok|error|timeout, startedAt, endedAt, error? }`

## Cola + concurrencia

- Las ejecuciones se serializan por clave de sesión (carril de sesión) y, opcionalmente, a través de un carril global.
- Esto evita condiciones de carrera de herramientas/sesión y mantiene coherente el historial de la sesión.
- Los canales de mensajería pueden elegir modos de cola (collect/steer/followup) que alimentan este sistema de carriles.
  Consulta [Cola de comandos](/es/concepts/queue).

## Preparación de sesión + espacio de trabajo

- El espacio de trabajo se resuelve y se crea; las ejecuciones en sandbox pueden redirigirse a una raíz de espacio de trabajo en sandbox.
- Las Skills se cargan (o se reutilizan desde una instantánea) y se inyectan en el entorno y en el prompt.
- Los archivos de bootstrap/contexto se resuelven y se inyectan en el informe del system prompt.
- Se adquiere un bloqueo de escritura de sesión; `SessionManager` se abre y se prepara antes del streaming.

## Ensamblaje del prompt + system prompt

- El system prompt se construye a partir del prompt base de OpenClaw, el prompt de Skills, el contexto de bootstrap y las anulaciones por ejecución.
- Se aplican los límites específicos del modelo y los tokens reservados para compactación.
- Consulta [System prompt](/es/concepts/system-prompt) para ver lo que ve el modelo.

## Puntos de hook (donde puedes interceptar)

OpenClaw tiene dos sistemas de hooks:

- **Hooks internos** (hooks del gateway): scripts impulsados por eventos para comandos y eventos del ciclo de vida.
- **Hooks de plugin**: puntos de extensión dentro del ciclo de vida del agente/herramientas y del pipeline del gateway.

### Hooks internos (hooks del gateway)

- **`agent:bootstrap`**: se ejecuta mientras se construyen los archivos de bootstrap antes de que se finalice el system prompt.
  Úsalo para agregar o eliminar archivos de contexto de bootstrap.
- **Hooks de comandos**: `/new`, `/reset`, `/stop` y otros eventos de comandos (consulta el documento de Hooks).

Consulta [Hooks](/es/automation/hooks) para la configuración y ejemplos.

### Hooks de plugin (ciclo de vida del agente + gateway)

Estos se ejecutan dentro del bucle del agente o del pipeline del gateway:

- **`before_model_resolve`**: se ejecuta antes de la sesión (sin `messages`) para anular de forma determinista el proveedor/modelo antes de la resolución del modelo.
- **`before_prompt_build`**: se ejecuta después de cargar la sesión (con `messages`) para inyectar `prependContext`, `systemPrompt`, `prependSystemContext` o `appendSystemContext` antes de enviar el prompt. Usa `prependContext` para texto dinámico por turno y los campos de contexto del sistema para guía estable que deba ubicarse en el espacio del system prompt.
- **`before_agent_start`**: hook heredado de compatibilidad que puede ejecutarse en cualquiera de las dos fases; prefiere los hooks explícitos anteriores.
- **`before_agent_reply`**: se ejecuta después de las acciones en línea y antes de la llamada al LLM, permitiendo que un plugin reclame el turno y devuelva una respuesta sintética o silencie el turno por completo.
- **`agent_end`**: inspecciona la lista final de mensajes y los metadatos de ejecución tras completarse.
- **`before_compaction` / `after_compaction`**: observan o anotan los ciclos de compactación.
- **`before_tool_call` / `after_tool_call`**: interceptan los parámetros/resultados de las herramientas.
- **`before_install`**: inspecciona los resultados del análisis integrado y, opcionalmente, bloquea instalaciones de Skills o plugins.
- **`tool_result_persist`**: transforma de forma síncrona los resultados de herramientas antes de que se escriban en la transcripción de la sesión.
- **`message_received` / `message_sending` / `message_sent`**: hooks de mensajes entrantes + salientes.
- **`session_start` / `session_end`**: límites del ciclo de vida de la sesión.
- **`gateway_start` / `gateway_stop`**: eventos del ciclo de vida del gateway.

Reglas de decisión de hooks para guardas de salida/herramientas:

- `before_tool_call`: `{ block: true }` es terminal y detiene los manejadores de menor prioridad.
- `before_tool_call`: `{ block: false }` no hace nada y no elimina un bloqueo previo.
- `before_install`: `{ block: true }` es terminal y detiene los manejadores de menor prioridad.
- `before_install`: `{ block: false }` no hace nada y no elimina un bloqueo previo.
- `message_sending`: `{ cancel: true }` es terminal y detiene los manejadores de menor prioridad.
- `message_sending`: `{ cancel: false }` no hace nada y no elimina una cancelación previa.

Consulta [Hooks de plugin](/es/plugins/architecture#provider-runtime-hooks) para la API de hooks y los detalles de registro.

## Streaming + respuestas parciales

- Los deltas del asistente se transmiten desde pi-agent-core y se emiten como eventos `assistant`.
- El streaming por bloques puede emitir respuestas parciales en `text_end` o en `message_end`.
- El streaming de razonamiento puede emitirse como un flujo independiente o como respuestas por bloques.
- Consulta [Streaming](/es/concepts/streaming) para el comportamiento de fragmentación y respuestas por bloques.

## Ejecución de herramientas + herramientas de mensajería

- Los eventos de inicio/actualización/fin de herramientas se emiten en el flujo `tool`.
- Los resultados de herramientas se sanean por tamaño y payloads de imagen antes de registrarlos/emitirlos.
- Los envíos de herramientas de mensajería se rastrean para suprimir confirmaciones duplicadas del asistente.

## Modelado de respuestas + supresión

- Los payloads finales se ensamblan a partir de:
  - texto del asistente (y razonamiento opcional)
  - resúmenes de herramientas en línea (cuando verbose + permitido)
  - texto de error del asistente cuando falla el modelo
- El token silencioso exacto `NO_REPLY` / `no_reply` se filtra de los
  payloads salientes.
- Los duplicados de herramientas de mensajería se eliminan de la lista final de payloads.
- Si no quedan payloads renderizables y una herramienta falló, se emite
  una respuesta de error de herramienta de respaldo
  (a menos que una herramienta de mensajería ya haya enviado una respuesta visible para el usuario).

## Compactación + reintentos

- La compactación automática emite eventos de flujo `compaction` y puede activar un reintento.
- En un reintento, los búferes en memoria y los resúmenes de herramientas se restablecen para evitar salida duplicada.
- Consulta [Compactación](/es/concepts/compaction) para el pipeline de compactación.

## Flujos de eventos (hoy)

- `lifecycle`: emitido por `subscribeEmbeddedPiSession` (y como respaldo por `agentCommand`)
- `assistant`: deltas en streaming desde pi-agent-core
- `tool`: eventos de herramientas en streaming desde pi-agent-core

## Manejo del canal de chat

- Los deltas del asistente se almacenan en búfer en mensajes `delta` del chat.
- Se emite un `final` de chat en **lifecycle end/error**.

## Tiempos de espera

- Valor predeterminado de `agent.wait`: 30 s (solo la espera). El parámetro `timeoutMs` lo reemplaza.
- Runtime del agente: `agents.defaults.timeoutSeconds`, valor predeterminado 172800 s (48 horas); se aplica en el temporizador de aborto de `runEmbeddedPiAgent`.
- Tiempo de espera inactivo del LLM: `agents.defaults.llm.idleTimeoutSeconds` aborta una solicitud al modelo cuando no llegan fragmentos de respuesta antes de la ventana de inactividad. Establécelo explícitamente para modelos locales lentos o proveedores de razonamiento/llamadas a herramientas; establécelo en 0 para desactivarlo. Si no se establece, OpenClaw usa `agents.defaults.timeoutSeconds` cuando está configurado; de lo contrario, 60 s. Las ejecuciones activadas por cron sin tiempo de espera explícito del LLM o del agente desactivan el watchdog de inactividad y dependen del tiempo de espera externo del cron.

## Dónde las cosas pueden terminar antes de tiempo

- Tiempo de espera del agente (aborto)
- AbortSignal (cancelación)
- Desconexión del gateway o tiempo de espera de RPC
- Tiempo de espera de `agent.wait` (solo de espera, no detiene al agente)

## Relacionado

- [Herramientas](/es/tools) — herramientas disponibles del agente
- [Hooks](/es/automation/hooks) — scripts impulsados por eventos activados por eventos del ciclo de vida del agente
- [Compactación](/es/concepts/compaction) — cómo se resumen las conversaciones largas
- [Aprobaciones de Exec](/es/tools/exec-approvals) — puertas de aprobación para comandos de shell
- [Thinking](/es/tools/thinking) — configuración del nivel de thinking/razonamiento
