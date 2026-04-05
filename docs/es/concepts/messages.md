---
read_when:
    - Explicar cómo los mensajes entrantes se convierten en respuestas
    - Aclarar sesiones, modos de puesta en cola o comportamiento de streaming
    - Documentar la visibilidad del razonamiento y las implicaciones de uso
summary: Flujo de mensajes, sesiones, puesta en cola y visibilidad del razonamiento
title: Mensajes
x-i18n:
    generated_at: "2026-04-05T12:40:05Z"
    model: gpt-5.4
    provider: openai
    source_hash: 475f892bd534fdb10a2ee5d3c57a3d4a7fb8e1ab68d695189ba186004713f6f3
    source_path: concepts/messages.md
    workflow: 15
---

# Mensajes

Esta página reúne cómo OpenClaw gestiona los mensajes entrantes, las sesiones, la puesta en cola,
el streaming y la visibilidad del razonamiento.

## Flujo de mensajes (alto nivel)

```
Inbound message
  -> routing/bindings -> session key
  -> queue (if a run is active)
  -> agent run (streaming + tools)
  -> outbound replies (channel limits + chunking)
```

Las opciones clave viven en la configuración:

- `messages.*` para prefijos, puesta en cola y comportamiento de grupos.
- `agents.defaults.*` para valores predeterminados de block streaming y fragmentación.
- Sobrescrituras por canal (`channels.whatsapp.*`, `channels.telegram.*`, etc.) para límites y cambios de streaming.

Consulta [Configuración](/gateway/configuration) para ver el esquema completo.

## Deduplicación de entrada

Los canales pueden volver a entregar el mismo mensaje después de reconexiones. OpenClaw mantiene una
caché de corta duración indexada por canal/cuenta/par/sesión/id de mensaje para que las entregas duplicadas
no activen otra ejecución del agente.

## Debouncing de entrada

Los mensajes consecutivos rápidos del **mismo remitente** pueden agruparse en un único
turno del agente mediante `messages.inbound`. El debouncing se limita por canal + conversación
y usa el mensaje más reciente para el threading/los IDs de respuesta.

Configuración (valor predeterminado global + sobrescrituras por canal):

```json5
{
  messages: {
    inbound: {
      debounceMs: 2000,
      byChannel: {
        whatsapp: 5000,
        slack: 1500,
        discord: 1500,
      },
    },
  },
}
```

Notas:

- El debounce se aplica a mensajes **solo de texto**; multimedia/archivos adjuntos se vacían inmediatamente.
- Los comandos de control omiten el debouncing para que sigan siendo independientes.

## Sesiones y dispositivos

Las sesiones son propiedad del gateway, no de los clientes.

- Los chats directos se consolidan en la clave de sesión principal del agente.
- Los grupos/canales obtienen sus propias claves de sesión.
- El almacén de sesiones y las transcripciones viven en el host del gateway.

Varios dispositivos/canales pueden asignarse a la misma sesión, pero el historial no se
sincroniza completamente de vuelta a todos los clientes. Recomendación: usa un único dispositivo principal para conversaciones largas para evitar divergencias de contexto. La interfaz de Control y la TUI siempre muestran la transcripción de la sesión respaldada por el gateway, por lo que son la fuente de verdad.

Detalles: [Gestión de sesiones](/concepts/session).

## Cuerpos de entrada e historial de contexto

OpenClaw separa el **cuerpo del prompt** del **cuerpo del comando**:

- `Body`: texto del prompt enviado al agente. Esto puede incluir envolturas del canal y
  envolturas opcionales del historial.
- `CommandBody`: texto sin procesar del usuario para analizar directivas/comandos.
- `RawBody`: alias heredado de `CommandBody` (se conserva por compatibilidad).

Cuando un canal proporciona historial, usa una envoltura compartida:

- `[Chat messages since your last reply - for context]`
- `[Current message - respond to this]`

Para **chats no directos** (grupos/canales/salas), el **cuerpo del mensaje actual** lleva como prefijo la
etiqueta del remitente (el mismo estilo que se usa para las entradas del historial). Esto mantiene
coherentes los mensajes en tiempo real y los puestos en cola/en historial dentro del prompt del agente.

Los búferes de historial son **solo pendientes**: incluyen mensajes de grupo que _no_
activaron una ejecución (por ejemplo, mensajes bloqueados por control de menciones) y **excluyen** mensajes
que ya están en la transcripción de la sesión.

La eliminación de directivas solo se aplica a la sección del **mensaje actual** para que el historial
permanezca intacto. Los canales que encapsulan historial deben establecer `CommandBody` (o
`RawBody`) con el texto original del mensaje y mantener `Body` como el prompt combinado.
Los búferes de historial se configuran mediante `messages.groupChat.historyLimit` (valor
predeterminado global) y sobrescrituras por canal como `channels.slack.historyLimit` o
`channels.telegram.accounts.<id>.historyLimit` (establece `0` para desactivar).

## Puesta en cola y seguimientos

Si una ejecución ya está activa, los mensajes entrantes pueden ponerse en cola, dirigirse a la
ejecución actual o recopilarse para un turno de seguimiento.

- Configura mediante `messages.queue` (y `messages.queue.byChannel`).
- Modos: `interrupt`, `steer`, `followup`, `collect`, más variantes de backlog.

Detalles: [Puesta en cola](/concepts/queue).

## Streaming, fragmentación y agrupación

El block streaming envía respuestas parciales a medida que el modelo produce bloques de texto.
La fragmentación respeta los límites de texto del canal y evita dividir código delimitado.

Configuraciones clave:

- `agents.defaults.blockStreamingDefault` (`on|off`, desactivado de forma predeterminada)
- `agents.defaults.blockStreamingBreak` (`text_end|message_end`)
- `agents.defaults.blockStreamingChunk` (`minChars|maxChars|breakPreference`)
- `agents.defaults.blockStreamingCoalesce` (agrupación basada en inactividad)
- `agents.defaults.humanDelay` (pausa similar a la humana entre respuestas por bloques)
- Sobrescrituras por canal: `*.blockStreaming` y `*.blockStreamingCoalesce` (los canales que no son Telegram requieren `*.blockStreaming: true` explícito)

Detalles: [Streaming + fragmentación](/concepts/streaming).

## Visibilidad del razonamiento y tokens

OpenClaw puede exponer u ocultar el razonamiento del modelo:

- `/reasoning on|off|stream` controla la visibilidad.
- El contenido del razonamiento sigue contando para el uso de tokens cuando lo produce el modelo.
- Telegram admite el streaming del razonamiento en la burbuja de borrador.

Detalles: [Directivas de pensamiento + razonamiento](/tools/thinking) y [Uso de tokens](/reference/token-use).

## Prefijos, threading y respuestas

El formato de los mensajes salientes está centralizado en `messages`:

- `messages.responsePrefix`, `channels.<channel>.responsePrefix` y `channels.<channel>.accounts.<id>.responsePrefix` (cascada de prefijos salientes), además de `channels.whatsapp.messagePrefix` (prefijo entrante de WhatsApp)
- Threading de respuestas mediante `replyToMode` y valores predeterminados por canal

Detalles: [Configuración](/gateway/configuration-reference#messages) y documentación de canales.

## Relacionado

- [Streaming](/concepts/streaming) — entrega de mensajes en tiempo real
- [Retry](/concepts/retry) — comportamiento de reintento en la entrega de mensajes
- [Queue](/concepts/queue) — cola de procesamiento de mensajes
- [Canales](/channels) — integraciones con plataformas de mensajería
