---
read_when:
    - Quieres entender qué significa “contexto” en OpenClaw
    - Estás depurando por qué el modelo “sabe” algo (o lo olvidó)
    - Quieres reducir la sobrecarga de contexto (`/context`, `/status`, `/compact`)
summary: 'Contexto: qué ve el modelo, cómo se construye y cómo inspeccionarlo'
title: Contexto
x-i18n:
    generated_at: "2026-04-18T04:59:12Z"
    model: gpt-5.4
    provider: openai
    source_hash: 477ccb1d9654968d0e904b6846b32b8c14db6b6c0d3d2ec2b7409639175629f9
    source_path: concepts/context.md
    workflow: 15
---

# Contexto

“Contexto” es **todo lo que OpenClaw envía al modelo para una ejecución**. Está limitado por la **ventana de contexto** del modelo (límite de tokens).

Modelo mental para principiantes:

- **Prompt del sistema** (construido por OpenClaw): reglas, herramientas, lista de Skills, tiempo/runtime y archivos del espacio de trabajo inyectados.
- **Historial de la conversación**: tus mensajes + los mensajes del asistente para esta sesión.
- **Llamadas/resultados de herramientas + adjuntos**: salida de comandos, lecturas de archivos, imágenes/audio, etc.

El contexto _no es lo mismo_ que la “memoria”: la memoria puede almacenarse en disco y volver a cargarse más tarde; el contexto es lo que está dentro de la ventana actual del modelo.

## Inicio rápido (inspeccionar el contexto)

- `/status` → vista rápida de “¿qué tan llena está mi ventana?” + configuración de la sesión.
- `/context list` → qué está inyectado + tamaños aproximados (por archivo + totales).
- `/context detail` → desglose más profundo: por archivo, tamaños de esquema por herramienta, tamaños de entrada por skill y tamaño del prompt del sistema.
- `/usage tokens` → añade un pie de uso por respuesta a las respuestas normales.
- `/compact` → resume el historial anterior en una entrada compacta para liberar espacio en la ventana.

Consulta también: [Comandos de barra](/es/tools/slash-commands), [Uso de tokens y costos](/es/reference/token-use), [Compaction](/es/concepts/compaction).

## Salida de ejemplo

Los valores varían según el modelo, el proveedor, la política de herramientas y lo que haya en tu espacio de trabajo.

### `/context list`

```
🧠 Context breakdown
Workspace: <workspaceDir>
Bootstrap max/file: 12,000 chars
Sandbox: mode=non-main sandboxed=false
System prompt (run): 38,412 chars (~9,603 tok) (Project Context 23,901 chars (~5,976 tok))

Injected workspace files:
- AGENTS.md: OK | raw 1,742 chars (~436 tok) | injected 1,742 chars (~436 tok)
- SOUL.md: OK | raw 912 chars (~228 tok) | injected 912 chars (~228 tok)
- TOOLS.md: TRUNCATED | raw 54,210 chars (~13,553 tok) | injected 20,962 chars (~5,241 tok)
- IDENTITY.md: OK | raw 211 chars (~53 tok) | injected 211 chars (~53 tok)
- USER.md: OK | raw 388 chars (~97 tok) | injected 388 chars (~97 tok)
- HEARTBEAT.md: MISSING | raw 0 | injected 0
- BOOTSTRAP.md: OK | raw 0 chars (~0 tok) | injected 0 chars (~0 tok)

Skills list (system prompt text): 2,184 chars (~546 tok) (12 skills)
Tools: read, edit, write, exec, process, browser, message, sessions_send, …
Tool list (system prompt text): 1,032 chars (~258 tok)
Tool schemas (JSON): 31,988 chars (~7,997 tok) (counts toward context; not shown as text)
Tools: (same as above)

Session tokens (cached): 14,250 total / ctx=32,000
```

### `/context detail`

```
🧠 Context breakdown (detailed)
…
Top skills (prompt entry size):
- frontend-design: 412 chars (~103 tok)
- oracle: 401 chars (~101 tok)
… (+10 more skills)

Top tools (schema size):
- browser: 9,812 chars (~2,453 tok)
- exec: 6,240 chars (~1,560 tok)
… (+N more tools)
```

## Qué cuenta para la ventana de contexto

Todo lo que recibe el modelo cuenta, incluido:

- Prompt del sistema (todas las secciones).
- Historial de la conversación.
- Llamadas de herramientas + resultados de herramientas.
- Adjuntos/transcripciones (imágenes/audio/archivos).
- Resúmenes de Compaction y artefactos de pruning.
- “Wrappers” del proveedor o encabezados ocultos (no visibles, pero aun así cuentan).

## Cómo OpenClaw construye el prompt del sistema

El prompt del sistema **pertenece a OpenClaw** y se reconstruye en cada ejecución. Incluye:

- Lista de herramientas + descripciones breves.
- Lista de Skills (solo metadatos; ver abajo).
- Ubicación del espacio de trabajo.
- Hora (UTC + hora del usuario convertida si está configurada).
- Metadatos del runtime (host/OS/modelo/thinking).
- Archivos bootstrap del espacio de trabajo inyectados bajo **Project Context**.

Desglose completo: [Prompt del sistema](/es/concepts/system-prompt).

## Archivos del espacio de trabajo inyectados (Project Context)

De forma predeterminada, OpenClaw inyecta un conjunto fijo de archivos del espacio de trabajo (si están presentes):

- `AGENTS.md`
- `SOUL.md`
- `TOOLS.md`
- `IDENTITY.md`
- `USER.md`
- `HEARTBEAT.md`
- `BOOTSTRAP.md` (solo la primera ejecución)

Los archivos grandes se truncan por archivo usando `agents.defaults.bootstrapMaxChars` (predeterminado `12000` caracteres). OpenClaw también aplica un límite total de inyección bootstrap entre archivos con `agents.defaults.bootstrapTotalMaxChars` (predeterminado `60000` caracteres). `/context` muestra los tamaños **sin procesar vs inyectados** y si ocurrió truncamiento.

Cuando ocurre truncamiento, el runtime puede inyectar un bloque de advertencia dentro del prompt en Project Context. Configura esto con `agents.defaults.bootstrapPromptTruncationWarning` (`off`, `once`, `always`; predeterminado `once`).

## Skills: inyectadas vs cargadas bajo demanda

El prompt del sistema incluye una **lista de Skills** compacta (nombre + descripción + ubicación). Esta lista tiene una sobrecarga real.

Las instrucciones de Skill _no_ se incluyen de forma predeterminada. Se espera que el modelo haga `read` del `SKILL.md` de la skill **solo cuando sea necesario**.

## Herramientas: hay dos costos

Las herramientas afectan al contexto de dos maneras:

1. **Texto de la lista de herramientas** en el prompt del sistema (lo que ves como “Tooling”).
2. **Esquemas de herramientas** (JSON). Se envían al modelo para que pueda llamar herramientas. Cuentan para el contexto aunque no los veas como texto plano.

`/context detail` desglosa los esquemas de herramientas más grandes para que puedas ver qué domina.

## Comandos, directivas y "atajos en línea"

Los comandos de barra son manejados por el Gateway. Hay algunos comportamientos diferentes:

- **Comandos independientes**: un mensaje que es solo `/...` se ejecuta como comando.
- **Directivas**: `/think`, `/verbose`, `/trace`, `/reasoning`, `/elevated`, `/model`, `/queue` se eliminan antes de que el modelo vea el mensaje.
  - Los mensajes compuestos solo por directivas conservan la configuración de la sesión.
  - Las directivas en línea en un mensaje normal actúan como sugerencias por mensaje.
- **Atajos en línea** (solo remitentes permitidos): ciertos tokens `/...` dentro de un mensaje normal pueden ejecutarse inmediatamente (ejemplo: “hey /status”), y se eliminan antes de que el modelo vea el texto restante.

Detalles: [Comandos de barra](/es/tools/slash-commands).

## Sesiones, Compaction y pruning (qué persiste)

Lo que persiste entre mensajes depende del mecanismo:

- **El historial normal** persiste en la transcripción de la sesión hasta que la política lo compacta/poda.
- **Compaction** conserva un resumen en la transcripción y mantiene intactos los mensajes recientes.
- **Pruning** elimina resultados antiguos de herramientas del prompt _en memoria_ para una ejecución, pero no reescribe la transcripción.

Documentación: [Sesión](/es/concepts/session), [Compaction](/es/concepts/compaction), [Poda de sesión](/es/concepts/session-pruning).

De forma predeterminada, OpenClaw usa el motor de contexto `legacy` integrado para el ensamblaje y la compacción. Si instalas un Plugin que proporciona `kind: "context-engine"` y lo seleccionas con `plugins.slots.contextEngine`, OpenClaw delega en ese motor el ensamblaje del contexto, `/compact` y los hooks relacionados del ciclo de vida del contexto del subagente. `ownsCompaction: false` no vuelve automáticamente al motor legacy; el motor activo debe seguir implementando `compact()` correctamente. Consulta
[Context Engine](/es/concepts/context-engine) para ver la interfaz conectable completa, los hooks del ciclo de vida y la configuración.

## Qué informa realmente `/context`

`/context` prefiere el informe más reciente del prompt del sistema **construido durante la ejecución** cuando está disponible:

- `System prompt (run)` = capturado de la última ejecución embebida (con capacidad de herramientas) y persistido en el almacenamiento de la sesión.
- `System prompt (estimate)` = calculado sobre la marcha cuando no existe un informe de ejecución (o al ejecutarse mediante un backend de CLI que no genera el informe).

En cualquier caso, informa tamaños y principales contribuyentes; **no** muestra el prompt completo del sistema ni los esquemas de herramientas.

## Relacionado

- [Context Engine](/es/concepts/context-engine) — inyección de contexto personalizada mediante plugins
- [Compaction](/es/concepts/compaction) — resumir conversaciones largas
- [Prompt del sistema](/es/concepts/system-prompt) — cómo se construye el prompt del sistema
- [Bucle del agente](/es/concepts/agent-loop) — el ciclo completo de ejecución del agente
