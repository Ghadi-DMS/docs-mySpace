---
read_when:
    - Necesitas depurar ids de sesión, JSONL de transcripciones o campos de sessions.json
    - Estás cambiando el comportamiento de la compactación automática o agregando tareas de mantenimiento "previas a la compactación"
    - Quieres implementar vaciados de memoria o turnos silenciosos del sistema
summary: 'Análisis detallado: almacén de sesiones + transcripciones, ciclo de vida e internals de la compactación (automática)'
title: Análisis detallado de la gestión de sesiones
x-i18n:
    generated_at: "2026-04-05T12:53:36Z"
    model: gpt-5.4
    provider: openai
    source_hash: e379d624dd7808d3af25ed011079268ce6a9da64bb3f301598884ad4c46ab091
    source_path: reference/session-management-compaction.md
    workflow: 15
---

# Gestión de sesiones y compactación (análisis detallado)

Este documento explica cómo OpenClaw gestiona las sesiones de extremo a extremo:

- **Enrutamiento de sesiones** (cómo los mensajes entrantes se asignan a una `sessionKey`)
- **Almacén de sesiones** (`sessions.json`) y qué rastrea
- **Persistencia de transcripciones** (`*.jsonl`) y su estructura
- **Higiene de transcripciones** (ajustes específicos del proveedor antes de las ejecuciones)
- **Límites de contexto** (ventana de contexto frente a tokens rastreados)
- **Compactación** (compactación manual + automática) y dónde enganchar trabajo previo a la compactación
- **Mantenimiento silencioso** (por ejemplo, escrituras de memoria que no deberían producir salida visible para el usuario)

Si primero quieres una visión general de nivel superior, empieza por:

- [/concepts/session](/es/concepts/session)
- [/concepts/compaction](/es/concepts/compaction)
- [/concepts/memory](/es/concepts/memory)
- [/concepts/memory-search](/es/concepts/memory-search)
- [/concepts/session-pruning](/es/concepts/session-pruning)
- [/reference/transcript-hygiene](/reference/transcript-hygiene)

---

## Fuente de verdad: el Gateway

OpenClaw está diseñado en torno a un único **proceso Gateway** que posee el estado de la sesión.

- Las interfaces de usuario (app de macOS, UI de control web, TUI) deben consultar al Gateway para obtener listas de sesiones y recuentos de tokens.
- En modo remoto, los archivos de sesión están en el host remoto; "comprobar tus archivos locales del Mac" no reflejará lo que está usando el Gateway.

---

## Dos capas de persistencia

OpenClaw persiste las sesiones en dos capas:

1. **Almacén de sesiones (`sessions.json`)**
   - Mapa clave/valor: `sessionKey -> SessionEntry`
   - Pequeño, mutable, seguro de editar (o de borrar entradas)
   - Rastrea metadatos de la sesión (id de sesión actual, última actividad, alternadores, contadores de tokens, etc.)

2. **Transcripción (`<sessionId>.jsonl`)**
   - Transcripción append-only con estructura de árbol (las entradas tienen `id` + `parentId`)
   - Almacena la conversación real + llamadas a herramientas + resúmenes de compactación
   - Se usa para reconstruir el contexto del modelo para turnos futuros

---

## Ubicaciones en disco

Por agente, en el host del Gateway:

- Almacén: `~/.openclaw/agents/<agentId>/sessions/sessions.json`
- Transcripciones: `~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl`
  - Sesiones de temas de Telegram: `.../<sessionId>-topic-<threadId>.jsonl`

OpenClaw resuelve esto mediante `src/config/sessions.ts`.

---

## Mantenimiento del almacén y controles de disco

La persistencia de sesiones tiene controles automáticos de mantenimiento (`session.maintenance`) para `sessions.json` y los artefactos de transcripción:

- `mode`: `warn` (predeterminado) o `enforce`
- `pruneAfter`: umbral de antigüedad para entradas obsoletas (predeterminado `30d`)
- `maxEntries`: límite de entradas en `sessions.json` (predeterminado `500`)
- `rotateBytes`: rota `sessions.json` cuando es demasiado grande (predeterminado `10mb`)
- `resetArchiveRetention`: retención para archivos de archivado de transcripciones `*.reset.<timestamp>` (predeterminado: igual que `pruneAfter`; `false` desactiva la limpieza)
- `maxDiskBytes`: presupuesto opcional para el directorio de sesiones
- `highWaterBytes`: objetivo opcional después de la limpieza (predeterminado `80%` de `maxDiskBytes`)

Orden de aplicación para la limpieza del presupuesto de disco (`mode: "enforce"`):

1. Elimina primero los artefactos de transcripción archivados u huérfanos más antiguos.
2. Si todavía está por encima del objetivo, expulsa las entradas de sesión más antiguas y sus archivos de transcripción.
3. Sigue hasta que el uso esté en o por debajo de `highWaterBytes`.

En `mode: "warn"`, OpenClaw informa sobre posibles expulsiones pero no modifica el almacén ni los archivos.

Ejecuta el mantenimiento bajo demanda:

```bash
openclaw sessions cleanup --dry-run
openclaw sessions cleanup --enforce
```

---

## Sesiones cron y registros de ejecución

Las ejecuciones cron aisladas también crean entradas/transcripciones de sesión, y tienen controles de retención dedicados:

- `cron.sessionRetention` (predeterminado `24h`) depura las sesiones antiguas de ejecuciones cron aisladas del almacén de sesiones (`false` desactiva esta opción).
- `cron.runLog.maxBytes` + `cron.runLog.keepLines` depuran archivos `~/.openclaw/cron/runs/<jobId>.jsonl` (valores predeterminados: `2_000_000` bytes y `2000` líneas).

---

## Claves de sesión (`sessionKey`)

Una `sessionKey` identifica _en qué contenedor de conversación_ estás (enrutamiento + aislamiento).

Patrones comunes:

- Chat principal/directo (por agente): `agent:<agentId>:<mainKey>` (predeterminado `main`)
- Grupo: `agent:<agentId>:<channel>:group:<id>`
- Sala/canal (Discord/Slack): `agent:<agentId>:<channel>:channel:<id>` o `...:room:<id>`
- Cron: `cron:<job.id>`
- Webhook: `hook:<uuid>` (a menos que se sobrescriba)

Las reglas canónicas están documentadas en [/concepts/session](/es/concepts/session).

---

## Ids de sesión (`sessionId`)

Cada `sessionKey` apunta a un `sessionId` actual (el archivo de transcripción que continúa la conversación).

Reglas generales:

- **Restablecer** (`/new`, `/reset`) crea un nuevo `sessionId` para esa `sessionKey`.
- **Restablecimiento diario** (predeterminado a las 4:00 AM hora local en el host del gateway) crea un nuevo `sessionId` en el siguiente mensaje después del límite de restablecimiento.
- **Caducidad por inactividad** (`session.reset.idleMinutes` o el heredado `session.idleMinutes`) crea un nuevo `sessionId` cuando llega un mensaje después de la ventana de inactividad. Cuando diario + inactividad están ambos configurados, gana el que caduque primero.
- **Protección de bifurcación del padre del hilo** (`session.parentForkMaxTokens`, predeterminado `100000`) omite la bifurcación de la transcripción padre cuando la sesión padre ya es demasiado grande; el nuevo hilo empieza desde cero. Establece `0` para desactivarlo.

Detalle de implementación: la decisión ocurre en `initSessionState()` en `src/auto-reply/reply/session.ts`.

---

## Esquema del almacén de sesiones (`sessions.json`)

El tipo de valor del almacén es `SessionEntry` en `src/config/sessions.ts`.

Campos clave (no exhaustivo):

- `sessionId`: id de la transcripción actual (el nombre del archivo se deriva de esto a menos que se establezca `sessionFile`)
- `updatedAt`: marca de tiempo de la última actividad
- `sessionFile`: sobrescritura opcional explícita de la ruta de la transcripción
- `chatType`: `direct | group | room` (ayuda a las interfaces y a la política de envío)
- `provider`, `subject`, `room`, `space`, `displayName`: metadatos para etiquetado de grupos/canales
- Alternadores:
  - `thinkingLevel`, `verboseLevel`, `reasoningLevel`, `elevatedLevel`
  - `sendPolicy` (sobrescritura por sesión)
- Selección de modelo:
  - `providerOverride`, `modelOverride`, `authProfileOverride`
- Contadores de tokens (best-effort / dependientes del proveedor):
  - `inputTokens`, `outputTokens`, `totalTokens`, `contextTokens`
- `compactionCount`: con qué frecuencia se completó la compactación automática para esta clave de sesión
- `memoryFlushAt`: marca de tiempo del último vaciado de memoria previo a la compactación
- `memoryFlushCompactionCount`: recuento de compactaciones cuando se ejecutó el último vaciado

El almacén es seguro de editar, pero el Gateway es la autoridad: puede reescribir o rehidratar entradas a medida que las sesiones se ejecutan.

---

## Estructura de la transcripción (`*.jsonl`)

Las transcripciones las gestiona el `SessionManager` de `@mariozechner/pi-coding-agent`.

El archivo es JSONL:

- Primera línea: cabecera de sesión (`type: "session"`, incluye `id`, `cwd`, `timestamp`, `parentSession` opcional)
- Luego: entradas de sesión con `id` + `parentId` (árbol)

Tipos de entrada destacados:

- `message`: mensajes de usuario/asistente/toolResult
- `custom_message`: mensajes inyectados por extensiones que _sí_ entran en el contexto del modelo (pueden ocultarse de la UI)
- `custom`: estado de la extensión que _no_ entra en el contexto del modelo
- `compaction`: resumen de compactación persistido con `firstKeptEntryId` y `tokensBefore`
- `branch_summary`: resumen persistido al navegar por una rama del árbol

OpenClaw intencionalmente **no** "corrige" transcripciones; el Gateway usa `SessionManager` para leerlas/escribirlas.

---

## Ventanas de contexto frente a tokens rastreados

Importan dos conceptos distintos:

1. **Ventana de contexto del modelo**: límite estricto por modelo (tokens visibles para el modelo)
2. **Contadores del almacén de sesiones**: estadísticas acumuladas escritas en `sessions.json` (usadas para `/status` y paneles)

Si estás ajustando límites:

- La ventana de contexto proviene del catálogo de modelos (y puede sobrescribirse mediante configuración).
- `contextTokens` en el almacén es un valor estimado/de informes en tiempo de ejecución; no lo trates como una garantía estricta.

Para más información, consulta [/token-use](/reference/token-use).

---

## Compactación: qué es

La compactación resume la conversación antigua en una entrada persistida `compaction` en la transcripción y mantiene intactos los mensajes recientes.

Después de la compactación, los turnos futuros ven:

- El resumen de compactación
- Los mensajes posteriores a `firstKeptEntryId`

La compactación es **persistente** (a diferencia de la depuración de sesión). Consulta [/concepts/session-pruning](/es/concepts/session-pruning).

## Límites de fragmentos de compactación y emparejamiento de herramientas

Cuando OpenClaw divide una transcripción larga en fragmentos de compactación, mantiene
las llamadas a herramientas del asistente emparejadas con sus entradas `toolResult`
correspondientes.

- Si la división por proporción de tokens cae entre una llamada a herramienta y su resultado, OpenClaw
  desplaza el límite al mensaje de llamada a herramienta del asistente en lugar de separar
  el par.
- Si un bloque final de resultado de herramienta, de otro modo, hiciera que el fragmento superara el objetivo,
  OpenClaw preserva ese bloque de herramienta pendiente y mantiene intacta la cola no resumida.
- Los bloques de llamada a herramienta abortados/con error no mantienen abierta una división pendiente.

---

## Cuándo ocurre la compactación automática (runtime de Pi)

En el agente Pi integrado, la compactación automática se activa en dos casos:

1. **Recuperación por desbordamiento**: el modelo devuelve un error de desbordamiento de contexto
   (`request_too_large`, `context length exceeded`, `input exceeds the maximum
number of tokens`, `input token count exceeds the maximum number of input
tokens`, `input is too long for the model`, `ollama error: context length
exceeded`, y variantes similares con forma de proveedor) → compactar → reintentar.
2. **Mantenimiento por umbral**: después de un turno correcto, cuando:

`contextTokens > contextWindow - reserveTokens`

Donde:

- `contextWindow` es la ventana de contexto del modelo
- `reserveTokens` es el margen reservado para prompts + la siguiente salida del modelo

Estas son semánticas del runtime de Pi (OpenClaw consume los eventos, pero Pi decide cuándo compactar).

---

## Ajustes de compactación (`reserveTokens`, `keepRecentTokens`)

Los ajustes de compactación de Pi viven en la configuración de Pi:

```json5
{
  compaction: {
    enabled: true,
    reserveTokens: 16384,
    keepRecentTokens: 20000,
  },
}
```

OpenClaw también aplica un suelo de seguridad para ejecuciones integradas:

- Si `compaction.reserveTokens < reserveTokensFloor`, OpenClaw lo aumenta.
- El suelo predeterminado es `20000` tokens.
- Establece `agents.defaults.compaction.reserveTokensFloor: 0` para desactivar el suelo.
- Si ya es mayor, OpenClaw lo deja como está.

Por qué: dejar suficiente margen para "mantenimiento" de varios turnos (como escrituras de memoria) antes de que la compactación sea inevitable.

Implementación: `ensurePiCompactionReserveTokens()` en `src/agents/pi-settings.ts`
(llamado desde `src/agents/pi-embedded-runner.ts`).

---

## Superficies visibles para el usuario

Puedes observar la compactación y el estado de la sesión mediante:

- `/status` (en cualquier sesión de chat)
- `openclaw status` (CLI)
- `openclaw sessions` / `sessions --json`
- Modo detallado: `🧹 Compactación automática completada` + recuento de compactaciones

---

## Mantenimiento silencioso (`NO_REPLY`)

OpenClaw admite turnos "silenciosos" para tareas en segundo plano donde el usuario no debería ver salida intermedia.

Convención:

- El asistente empieza su salida con el token silencioso exacto `NO_REPLY` /
  `no_reply` para indicar "no entregar una respuesta al usuario".
- OpenClaw elimina/suprime esto en la capa de entrega.
- La supresión exacta del token silencioso no distingue mayúsculas de minúsculas, así que `NO_REPLY` y
  `no_reply` cuentan cuando toda la carga útil es solo el token silencioso.
- Esto es solo para turnos verdaderamente en segundo plano/sin entrega; no es un atajo para
  solicitudes ordinarias procesables del usuario.

A partir de `2026.1.10`, OpenClaw también suprime el **streaming de borrador/escritura** cuando un
fragmento parcial empieza con `NO_REPLY`, para que las operaciones silenciosas no filtren salida
parcial a mitad del turno.

---

## "Vaciado de memoria" previo a la compactación (implementado)

Objetivo: antes de que ocurra la compactación automática, ejecutar un turno agéntico silencioso que escriba estado
duradero en disco (por ejemplo, `memory/YYYY-MM-DD.md` en el espacio de trabajo del agente) para que la compactación no pueda
borrar contexto crítico.

OpenClaw usa el enfoque de **vaciado previo al umbral**:

1. Supervisar el uso de contexto de la sesión.
2. Cuando cruza un "umbral suave" (por debajo del umbral de compactación de Pi), ejecutar una directiva silenciosa
   de "escribe memoria ahora" al agente.
3. Usar el token silencioso exacto `NO_REPLY` / `no_reply` para que el usuario no vea
   nada.

Configuración (`agents.defaults.compaction.memoryFlush`):

- `enabled` (predeterminado: `true`)
- `softThresholdTokens` (predeterminado: `4000`)
- `prompt` (mensaje de usuario para el turno de vaciado)
- `systemPrompt` (prompt del sistema adicional añadido para el turno de vaciado)

Notas:

- El prompt/prompt del sistema predeterminados incluyen una pista `NO_REPLY` para suprimir
  la entrega.
- El vaciado se ejecuta una vez por ciclo de compactación (rastreado en `sessions.json`).
- El vaciado se ejecuta solo para sesiones Pi integradas (los backends de CLI lo omiten).
- El vaciado se omite cuando el espacio de trabajo de la sesión es de solo lectura (`workspaceAccess: "ro"` o `"none"`).
- Consulta [Memory](/es/concepts/memory) para el diseño de archivos del espacio de trabajo y los patrones de escritura.

Pi también expone un hook `session_before_compact` en la API de extensiones, pero la lógica de vaciado
de OpenClaw vive actualmente del lado del Gateway.

---

## Lista de verificación para solución de problemas

- ¿La clave de sesión es incorrecta? Empieza con [/concepts/session](/es/concepts/session) y confirma la `sessionKey` en `/status`.
- ¿No coinciden el almacén y la transcripción? Confirma el host del Gateway y la ruta del almacén desde `openclaw status`.
- ¿Compactación excesiva? Comprueba:
  - ventana de contexto del modelo (demasiado pequeña)
  - ajustes de compactación (`reserveTokens` demasiado alto para la ventana del modelo puede causar una compactación más temprana)
  - exceso de resultados de herramientas: habilita/ajusta la depuración de sesión
- ¿Se filtran turnos silenciosos? Confirma que la respuesta empieza con `NO_REPLY` (token exacto sin distinguir mayúsculas de minúsculas) y que estás en una compilación que incluye la corrección de supresión de streaming.
