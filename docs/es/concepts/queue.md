---
read_when:
    - Cambiar la ejecución o concurrencia de respuestas automáticas
summary: Diseño de cola de comandos que serializa ejecuciones de respuesta automática entrantes
title: Cola de comandos
x-i18n:
    generated_at: "2026-04-05T12:40:20Z"
    model: gpt-5.4
    provider: openai
    source_hash: 36e1d004e9a2c21ad1470517a249285216114dd4cf876681cc860e992c73914f
    source_path: concepts/queue.md
    workflow: 15
---

# Cola de comandos (2026-01-16)

Serializamos las ejecuciones de respuesta automática entrantes (todos los canales) mediante una pequeña cola en proceso para evitar que varias ejecuciones del agente colisionen, mientras seguimos permitiendo paralelismo seguro entre sesiones.

## Por qué

- Las ejecuciones de respuesta automática pueden ser costosas (llamadas LLM) y pueden colisionar cuando llegan varios mensajes entrantes con poca diferencia de tiempo.
- La serialización evita competir por recursos compartidos (archivos de sesión, registros, stdin de CLI) y reduce la probabilidad de límites de tasa aguas arriba.

## Cómo funciona

- Una cola FIFO con reconocimiento de carriles vacía cada carril con un límite de concurrencia configurable (predeterminado 1 para carriles no configurados; `main` usa 4 de forma predeterminada y `subagent`, 8).
- `runEmbeddedPiAgent` pone en cola por **clave de sesión** (carril `session:<key>`) para garantizar que solo haya una ejecución activa por sesión.
- Luego, cada ejecución de sesión se pone en cola en un **carril global** (`main` de forma predeterminada) para que el paralelismo total quede limitado por `agents.defaults.maxConcurrent`.
- Cuando el registro detallado está habilitado, las ejecuciones en cola emiten un aviso corto si esperaron más de ~2 s antes de comenzar.
- Los indicadores de escritura siguen activándose inmediatamente al ponerse en cola (cuando el canal lo admite), por lo que la experiencia del usuario no cambia mientras espera su turno.

## Modos de cola (por canal)

Los mensajes entrantes pueden dirigir la ejecución actual, esperar a un turno de seguimiento o hacer ambas cosas:

- `steer`: inyecta inmediatamente en la ejecución actual (cancela las llamadas de herramientas pendientes después del siguiente límite de herramienta). Si no hay streaming, vuelve a `followup`.
- `followup`: se pone en cola para el siguiente turno del agente después de que termine la ejecución actual.
- `collect`: fusiona todos los mensajes en cola en un **único** turno de seguimiento (predeterminado). Si los mensajes apuntan a distintos canales/hilos, se vacían individualmente para preservar el enrutamiento.
- `steer-backlog` (también `steer+backlog`): dirige ahora **y** preserva el mensaje para un turno de seguimiento.
- `interrupt` (heredado): aborta la ejecución activa de esa sesión y luego ejecuta el mensaje más reciente.
- `queue` (alias heredado): igual que `steer`.

Steer-backlog significa que puedes recibir una respuesta de seguimiento después de la ejecución dirigida, por lo que
las superficies con streaming pueden parecer duplicados. Prefiere `collect`/`steer` si quieres
una respuesta por mensaje entrante.
Envía `/queue collect` como comando independiente (por sesión) o establece `messages.queue.byChannel.discord: "collect"`.

Valores predeterminados (cuando no están definidos en la configuración):

- Todas las superficies → `collect`

Configúralo globalmente o por canal mediante `messages.queue`:

```json5
{
  messages: {
    queue: {
      mode: "collect",
      debounceMs: 1000,
      cap: 20,
      drop: "summarize",
      byChannel: { discord: "collect" },
    },
  },
}
```

## Opciones de cola

Las opciones se aplican a `followup`, `collect` y `steer-backlog` (y a `steer` cuando vuelve a `followup`):

- `debounceMs`: espera a que haya silencio antes de iniciar un turno de seguimiento (evita “continúa, continúa”).
- `cap`: máximo de mensajes en cola por sesión.
- `drop`: política de desbordamiento (`old`, `new`, `summarize`).

Summarize conserva una lista breve con viñetas de los mensajes descartados y la inyecta como un prompt sintético de seguimiento.
Valores predeterminados: `debounceMs: 1000`, `cap: 20`, `drop: summarize`.

## Sobrescrituras por sesión

- Envía `/queue <mode>` como comando independiente para almacenar el modo de la sesión actual.
- Las opciones se pueden combinar: `/queue collect debounce:2s cap:25 drop:summarize`
- `/queue default` o `/queue reset` borra la sobrescritura de la sesión.

## Alcance y garantías

- Se aplica a ejecuciones de agentes de respuesta automática en todos los canales entrantes que usan la canalización de respuesta del gateway (WhatsApp web, Telegram, Slack, Discord, Signal, iMessage, webchat, etc.).
- El carril predeterminado (`main`) es para todo el proceso en entradas + heartbeats principales; establece `agents.defaults.maxConcurrent` para permitir varias sesiones en paralelo.
- Pueden existir carriles adicionales (por ejemplo, `cron`, `subagent`) para que los trabajos en segundo plano puedan ejecutarse en paralelo sin bloquear las respuestas entrantes. Estas ejecuciones desacopladas se registran como [tareas en segundo plano](/automation/tasks).
- Los carriles por sesión garantizan que solo una ejecución del agente toque una sesión dada a la vez.
- Sin dependencias externas ni hilos de trabajo en segundo plano; TypeScript puro + promesas.

## Resolución de problemas

- Si parece que los comandos están atascados, habilita los registros detallados y busca líneas “queued for …ms” para confirmar que la cola se está vaciando.
- Si necesitas la profundidad de la cola, habilita los registros detallados y observa las líneas de temporización de la cola.
