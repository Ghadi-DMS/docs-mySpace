---
read_when:
    - Quieres entender qué significa “contexto” en OpenClaw
    - Estás depurando por qué el modelo “sabe” algo (o lo olvidó)
    - Quieres reducir la sobrecarga de contexto (`/context`, `/status`, `/compact`)
summary: 'Contexto: qué ve el modelo, cómo se construye y cómo inspeccionarlo'
title: Contexto
x-i18n:
    generated_at: "2026-04-05T12:40:02Z"
    model: gpt-5.4
    provider: openai
    source_hash: a75b4cd65bf6385d46265b9ce1643310bc99d220e35ec4b4924096bed3ca4aa0
    source_path: concepts/context.md
    workflow: 15
---

# Contexto

“Contexto” es **todo lo que OpenClaw envía al modelo para una ejecución**. Está limitado por la **ventana de contexto** del modelo (límite de tokens).

Modelo mental para principiantes:

- **Prompt del sistema** (construido por OpenClaw): reglas, herramientas, lista de Skills, hora/entorno de ejecución y archivos del espacio de trabajo inyectados.
- **Historial de conversación**: tus mensajes + los mensajes del asistente para esta sesión.
- **Llamadas/resultados de herramientas + adjuntos**: salida de comandos, lecturas de archivos, imágenes/audio, etc.

El contexto _no es lo mismo_ que la “memoria”: la memoria puede almacenarse en disco y volver a cargarse después; el contexto es lo que está dentro de la ventana actual del modelo.

## Inicio rápido (inspeccionar contexto)

- `/status` → vista rápida de “¿cuánto de mi ventana está lleno?” + configuración de la sesión.
- `/context list` → qué está inyectado + tamaños aproximados (por archivo + totales).
- `/context detail` → desglose más profundo: por archivo, tamaños de esquemas de herramientas, tamaños de entradas de Skills y tamaño del prompt del sistema.
- `/usage tokens` → añade un pie de uso por respuesta a las respuestas normales.
- `/compact` → resume el historial antiguo en una entrada compacta para liberar espacio en la ventana.

Consulta también: [Comandos de barra](/tools/slash-commands), [Uso de tokens y costos](/reference/token-use), [Compactación](/concepts/compaction).

## Ejemplo de salida

Los valores varían según el modelo, el proveedor, la política de herramientas y lo que haya en tu espacio de trabajo.

### `/context list`

```
🧠 Desglose de contexto
Espacio de trabajo: <workspaceDir>
Máximo de bootstrap/archivo: 20,000 caracteres
Sandbox: mode=non-main sandboxed=false
Prompt del sistema (ejecución): 38,412 caracteres (~9,603 tok) (Contexto del proyecto 23,901 caracteres (~5,976 tok))

Archivos inyectados del espacio de trabajo:
- AGENTS.md: OK | raw 1,742 chars (~436 tok) | injected 1,742 chars (~436 tok)
- SOUL.md: OK | raw 912 chars (~228 tok) | injected 912 chars (~228 tok)
- TOOLS.md: TRUNCATED | raw 54,210 chars (~13,553 tok) | injected 20,962 chars (~5,241 tok)
- IDENTITY.md: OK | raw 211 chars (~53 tok) | injected 211 chars (~53 tok)
- USER.md: OK | raw 388 chars (~97 tok) | injected 388 chars (~97 tok)
- HEARTBEAT.md: MISSING | raw 0 | injected 0
- BOOTSTRAP.md: OK | raw 0 chars (~0 tok) | injected 0 chars (~0 tok)

Lista de Skills (texto del prompt del sistema): 2,184 caracteres (~546 tok) (12 Skills)
Herramientas: read, edit, write, exec, process, browser, message, sessions_send, …
Lista de herramientas (texto del prompt del sistema): 1,032 caracteres (~258 tok)
Esquemas de herramientas (JSON): 31,988 caracteres (~7,997 tok) (cuentan para el contexto; no se muestran como texto)
Herramientas: (igual que arriba)

Tokens de sesión (en caché): 14,250 total / ctx=32,000
```

### `/context detail`

```
🧠 Desglose de contexto (detallado)
…
Skills principales (tamaño de entrada en el prompt):
- frontend-design: 412 caracteres (~103 tok)
- oracle: 401 caracteres (~101 tok)
… (+10 más Skills)

Herramientas principales (tamaño del esquema):
- browser: 9,812 caracteres (~2,453 tok)
- exec: 6,240 caracteres (~1,560 tok)
… (+N más herramientas)
```

## Qué cuenta para la ventana de contexto

Cuenta todo lo que recibe el modelo, incluidos:

- Prompt del sistema (todas las secciones).
- Historial de conversación.
- Llamadas a herramientas + resultados de herramientas.
- Adjuntos/transcripciones (imágenes/audio/archivos).
- Resúmenes de compactación y artefactos de poda.
- “Wrappers” del proveedor o cabeceras ocultas (no visibles, pero siguen contando).

## Cómo construye OpenClaw el prompt del sistema

El prompt del sistema **pertenece a OpenClaw** y se reconstruye en cada ejecución. Incluye:

- Lista de herramientas + descripciones breves.
- Lista de Skills (solo metadatos; ver más abajo).
- Ubicación del espacio de trabajo.
- Hora (UTC + hora del usuario convertida si está configurada).
- Metadatos del entorno de ejecución (host/SO/modelo/thinking).
- Archivos bootstrap inyectados del espacio de trabajo bajo **Contexto del proyecto**.

Desglose completo: [Prompt del sistema](/concepts/system-prompt).

## Archivos inyectados del espacio de trabajo (Contexto del proyecto)

De forma predeterminada, OpenClaw inyecta un conjunto fijo de archivos del espacio de trabajo (si existen):

- `AGENTS.md`
- `SOUL.md`
- `TOOLS.md`
- `IDENTITY.md`
- `USER.md`
- `HEARTBEAT.md`
- `BOOTSTRAP.md` (solo en la primera ejecución)

Los archivos grandes se truncan por archivo usando `agents.defaults.bootstrapMaxChars` (predeterminado `20000` caracteres). OpenClaw también aplica un límite total de inyección bootstrap entre archivos con `agents.defaults.bootstrapTotalMaxChars` (predeterminado `150000` caracteres). `/context` muestra tamaños **sin procesar frente a inyectados** y si hubo truncamiento.

Cuando ocurre truncamiento, el entorno de ejecución puede inyectar un bloque de advertencia dentro del prompt en Contexto del proyecto. Configúralo con `agents.defaults.bootstrapPromptTruncationWarning` (`off`, `once`, `always`; predeterminado `once`).

## Skills: inyectadas vs cargadas bajo demanda

El prompt del sistema incluye una **lista compacta de Skills** (nombre + descripción + ubicación). Esta lista tiene una sobrecarga real.

Las instrucciones de las Skills _no_ se incluyen de forma predeterminada. Se espera que el modelo haga `read` del `SKILL.md` de la Skill **solo cuando sea necesario**.

## Herramientas: hay dos costos

Las herramientas afectan al contexto de dos maneras:

1. **Texto de la lista de herramientas** en el prompt del sistema (lo que ves como “Tooling”).
2. **Esquemas de herramientas** (JSON). Se envían al modelo para que pueda llamar herramientas. Cuentan para el contexto aunque no los veas como texto sin formato.

`/context detail` desglosa los esquemas de herramientas más grandes para que puedas ver qué domina.

## Comandos, directivas y "atajos en línea"

Los comandos de barra los gestiona el Gateway. Hay varios comportamientos diferentes:

- **Comandos independientes**: un mensaje que es solo `/...` se ejecuta como comando.
- **Directivas**: `/think`, `/verbose`, `/reasoning`, `/elevated`, `/model`, `/queue` se eliminan antes de que el modelo vea el mensaje.
  - Los mensajes que contienen solo directivas conservan la configuración de la sesión.
  - Las directivas en línea dentro de un mensaje normal actúan como sugerencias por mensaje.
- **Atajos en línea** (solo remitentes en lista de permitidos): ciertos tokens `/...` dentro de un mensaje normal pueden ejecutarse inmediatamente (ejemplo: “hey /status”), y se eliminan antes de que el modelo vea el texto restante.

Detalles: [Comandos de barra](/tools/slash-commands).

## Sesiones, compactación y poda (qué persiste)

Lo que persiste entre mensajes depende del mecanismo:

- **El historial normal** persiste en la transcripción de la sesión hasta que la política lo compacte/pode.
- **La compactación** conserva un resumen en la transcripción y mantiene intactos los mensajes recientes.
- **La poda** elimina resultados antiguos de herramientas del prompt _en memoria_ para una ejecución, pero no reescribe la transcripción.

Documentación: [Sesión](/concepts/session), [Compactación](/concepts/compaction), [Poda de sesión](/concepts/session-pruning).

De forma predeterminada, OpenClaw usa el motor de contexto integrado `legacy` para el ensamblaje y la
compactación. Si instalas un plugin que proporcione `kind: "context-engine"` y lo
seleccionas con `plugins.slots.contextEngine`, OpenClaw delega el ensamblaje del contexto,
`/compact` y los hooks relacionados del ciclo de vida del contexto de subagentes a ese
motor. `ownsCompaction: false` no vuelve automáticamente al motor
heredado; el motor activo debe seguir implementando `compact()` correctamente. Consulta
[Motor de contexto](/concepts/context-engine) para la interfaz conectable completa,
los hooks de ciclo de vida y la configuración.

## Qué informa realmente `/context`

`/context` prefiere el informe más reciente del prompt del sistema **construido en ejecución** cuando está disponible:

- `System prompt (run)` = capturado de la última ejecución integrada (capaz de usar herramientas) y conservado en el almacén de sesiones.
- `System prompt (estimate)` = calculado sobre la marcha cuando no existe informe de ejecución (o al ejecutar mediante un backend de CLI que no genera el informe).

En cualquier caso, informa tamaños y los principales contribuyentes; **no** vuelca el prompt completo del sistema ni los esquemas de herramientas.

## Relacionado

- [Motor de contexto](/concepts/context-engine) — inyección de contexto personalizada mediante plugins
- [Compactación](/concepts/compaction) — resumir conversaciones largas
- [Prompt del sistema](/concepts/system-prompt) — cómo se construye el prompt del sistema
- [Bucle de agente](/concepts/agent-loop) — el ciclo completo de ejecución del agente
