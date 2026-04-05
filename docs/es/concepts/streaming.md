---
read_when:
    - Explicas cómo funciona el streaming o la fragmentación en canales
    - Cambias el comportamiento del streaming por bloques o de la fragmentación en canales
    - Depuras respuestas por bloques duplicadas o tempranas, o el streaming de vista previa en canales
summary: Comportamiento de streaming y fragmentación (respuestas por bloques, streaming de vista previa en canales, asignación de modos)
title: Streaming y fragmentación
x-i18n:
    generated_at: "2026-04-05T12:40:59Z"
    model: gpt-5.4
    provider: openai
    source_hash: 44b0d08c7eafcb32030ef7c8d5719c2ea2d34e4bac5fdad8cc8b3f4e9e9fad97
    source_path: concepts/streaming.md
    workflow: 15
---

# Streaming y fragmentación

OpenClaw tiene dos capas de streaming separadas:

- **Streaming por bloques (canales):** emite **bloques** completados mientras el asistente escribe. Son mensajes normales del canal (no deltas de tokens).
- **Streaming de vista previa (Telegram/Discord/Slack):** actualiza un **mensaje de vista previa** temporal mientras se genera.

Hoy no existe **streaming real de deltas de tokens** hacia los mensajes de canal. El streaming de vista previa se basa en mensajes (enviar + editar/anexar).

## Streaming por bloques (mensajes de canal)

El streaming por bloques envía la salida del asistente en fragmentos gruesos a medida que está disponible.

```
Model output
  └─ text_delta/events
       ├─ (blockStreamingBreak=text_end)
       │    └─ chunker emits blocks as buffer grows
       └─ (blockStreamingBreak=message_end)
            └─ chunker flushes at message_end
                   └─ channel send (block replies)
```

Leyenda:

- `text_delta/events`: eventos del stream del modelo (pueden ser escasos en modelos sin streaming).
- `chunker`: `EmbeddedBlockChunker` que aplica límites mínimos y máximos + preferencia de corte.
- `channel send`: mensajes salientes reales (respuestas por bloques).

**Controles:**

- `agents.defaults.blockStreamingDefault`: `"on"`/`"off"` (predeterminado: desactivado).
- Anulaciones por canal: `*.blockStreaming` (y variantes por cuenta) para forzar `"on"`/`"off"` por canal.
- `agents.defaults.blockStreamingBreak`: `"text_end"` o `"message_end"`.
- `agents.defaults.blockStreamingChunk`: `{ minChars, maxChars, breakPreference? }`.
- `agents.defaults.blockStreamingCoalesce`: `{ minChars?, maxChars?, idleMs? }` (fusiona bloques transmitidos antes del envío).
- Límite rígido del canal: `*.textChunkLimit` (por ejemplo, `channels.whatsapp.textChunkLimit`).
- Modo de fragmentación del canal: `*.chunkMode` (`length` predeterminado, `newline` divide en líneas en blanco (límites de párrafo) antes de fragmentar por longitud).
- Límite flexible de Discord: `channels.discord.maxLinesPerMessage` (predeterminado: 17) divide respuestas altas para evitar recortes de la interfaz.

**Semántica de límites:**

- `text_end`: transmite bloques tan pronto como los emite el chunker; vacía en cada `text_end`.
- `message_end`: espera a que termine el mensaje del asistente y luego vacía la salida en búfer.

`message_end` sigue usando el chunker si el texto en búfer supera `maxChars`, por lo que puede emitir varios fragmentos al final.

## Algoritmo de fragmentación (límites bajos/altos)

La fragmentación por bloques se implementa con `EmbeddedBlockChunker`:

- **Límite bajo:** no emite hasta que el búfer sea >= `minChars` (salvo que se fuerce).
- **Límite alto:** prefiere cortes antes de `maxChars`; si se fuerza, corta en `maxChars`.
- **Preferencia de corte:** `paragraph` → `newline` → `sentence` → `whitespace` → corte rígido.
- **Bloques de código:** nunca se dividen dentro de bloques delimitados; cuando se fuerza en `maxChars`, cierra y vuelve a abrir el bloque para mantener Markdown válido.

`maxChars` se ajusta al `textChunkLimit` del canal, por lo que no puedes superar los límites por canal.

## Coalescencia (fusionar bloques transmitidos)

Cuando el streaming por bloques está habilitado, OpenClaw puede **fusionar fragmentos consecutivos de bloques**
antes de enviarlos. Esto reduce el “spam de una sola línea” mientras sigue proporcionando
salida progresiva.

- La coalescencia espera **intervalos de inactividad** (`idleMs`) antes de vaciar.
- Los búferes están limitados por `maxChars` y se vacían si lo superan.
- `minChars` evita enviar fragmentos muy pequeños hasta que se acumule suficiente texto
  (el vaciado final siempre envía el texto restante).
- El separador se deriva de `blockStreamingChunk.breakPreference`
  (`paragraph` → `\n\n`, `newline` → `\n`, `sentence` → espacio).
- Las anulaciones por canal están disponibles mediante `*.blockStreamingCoalesce` (incluidas configuraciones por cuenta).
- El valor predeterminado de coalescencia `minChars` aumenta a 1500 para Signal/Slack/Discord salvo que se anule.

## Ritmo humano entre bloques

Cuando el streaming por bloques está habilitado, puedes añadir una **pausa aleatoria** entre
respuestas por bloques (después del primer bloque). Esto hace que las respuestas de varias burbujas parezcan
más naturales.

- Configuración: `agents.defaults.humanDelay` (anular por agente mediante `agents.list[].humanDelay`).
- Modos: `off` (predeterminado), `natural` (800–2500 ms), `custom` (`minMs`/`maxMs`).
- Se aplica solo a **respuestas por bloques**, no a respuestas finales ni a resúmenes de herramientas.

## "Transmitir fragmentos o todo"

Esto corresponde a:

- **Transmitir fragmentos:** `blockStreamingDefault: "on"` + `blockStreamingBreak: "text_end"` (emitir sobre la marcha). Los canales que no son Telegram también necesitan `*.blockStreaming: true`.
- **Transmitir todo al final:** `blockStreamingBreak: "message_end"` (vaciar una vez, posiblemente en varios fragmentos si es muy largo).
- **Sin streaming por bloques:** `blockStreamingDefault: "off"` (solo respuesta final).

**Nota del canal:** El streaming por bloques está **desactivado a menos que**
`*.blockStreaming` se configure explícitamente como `true`. Los canales pueden transmitir una vista previa en vivo
(`channels.<channel>.streaming`) sin respuestas por bloques.

Recordatorio de ubicación de configuración: los valores predeterminados `blockStreaming*` están en
`agents.defaults`, no en la configuración raíz.

## Modos de streaming de vista previa

Clave canónica: `channels.<channel>.streaming`

Modos:

- `off`: desactiva el streaming de vista previa.
- `partial`: una sola vista previa que se reemplaza por el texto más reciente.
- `block`: la vista previa se actualiza en pasos fragmentados o anexados.
- `progress`: vista previa de progreso/estado durante la generación, respuesta final al completarse.

### Asignación por canal

| Canal    | `off` | `partial` | `block` | `progress`        |
| -------- | ----- | --------- | ------- | ----------------- |
| Telegram | ✅    | ✅        | ✅      | se asigna a `partial` |
| Discord  | ✅    | ✅        | ✅      | se asigna a `partial` |
| Slack    | ✅    | ✅        | ✅      | ✅                |

Solo Slack:

- `channels.slack.nativeStreaming` alterna llamadas a la API nativa de streaming de Slack cuando `streaming=partial` (predeterminado: `true`).

Migración de claves heredadas:

- Telegram: `streamMode` + booleano `streaming` migran automáticamente al enum `streaming`.
- Discord: `streamMode` + booleano `streaming` migran automáticamente al enum `streaming`.
- Slack: `streamMode` migra automáticamente al enum `streaming`; el booleano `streaming` migra automáticamente a `nativeStreaming`.

### Comportamiento en tiempo de ejecución

Telegram:

- Usa actualizaciones de vista previa con `sendMessage` + `editMessageText` en mensajes directos y grupos/temas.
- El streaming de vista previa se omite cuando el streaming por bloques de Telegram está habilitado explícitamente (para evitar doble streaming).
- `/reasoning stream` puede escribir razonamiento en la vista previa.

Discord:

- Usa mensajes de vista previa con envío + edición.
- El modo `block` usa fragmentación de borrador (`draftChunk`).
- El streaming de vista previa se omite cuando el streaming por bloques de Discord está habilitado explícitamente.

Slack:

- `partial` puede usar la API nativa de streaming de Slack (`chat.startStream`/`append`/`stop`) cuando está disponible.
- `block` usa vistas previas de borrador con estilo de anexado.
- `progress` usa texto de vista previa de estado y luego la respuesta final.

## Relacionado

- [Messages](/concepts/messages) — ciclo de vida y entrega de mensajes
- [Retry](/concepts/retry) — comportamiento de reintento en fallos de entrega
- [Channels](/channels) — compatibilidad de streaming por canal
