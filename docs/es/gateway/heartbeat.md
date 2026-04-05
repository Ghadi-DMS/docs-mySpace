---
read_when:
    - Ajustar la cadencia o los mensajes de heartbeat
    - Decidir entre heartbeat y cron para tareas programadas
summary: Mensajes de sondeo de heartbeat y reglas de notificación
title: Heartbeat
x-i18n:
    generated_at: "2026-04-05T12:42:38Z"
    model: gpt-5.4
    provider: openai
    source_hash: f417b0d4453bed9022144d364521a59dec919d44cca8f00f0def005cd38b146f
    source_path: gateway/heartbeat.md
    workflow: 15
---

# Heartbeat (Gateway)

> **¿Heartbeat o cron?** Consulta [Automatización y tareas](/automation) para orientarte sobre cuándo usar cada uno.

Heartbeat ejecuta **turnos periódicos del agente** en la sesión principal para que el modelo pueda
mostrar cualquier cosa que requiera atención sin enviarte spam.

Heartbeat es un turno programado de la sesión principal: **no** crea registros de [tareas en segundo plano](/automation/tasks).
Los registros de tareas son para trabajo desacoplado (ejecuciones de ACP, subagentes, trabajos cron aislados).

Solución de problemas: [Tareas programadas](/automation/cron-jobs#troubleshooting)

## Inicio rápido (principiante)

1. Deja los heartbeat habilitados (el valor predeterminado es `30m`, o `1h` para autenticación Anthropic OAuth/token, incluida la reutilización de Claude CLI) o establece tu propia cadencia.
2. Crea una pequeña lista de verificación `HEARTBEAT.md` o un bloque `tasks:` en el espacio de trabajo del agente (opcional, pero recomendado).
3. Decide adónde deben ir los mensajes de heartbeat (`target: "none"` es el valor predeterminado; establece `target: "last"` para enrutar al último contacto).
4. Opcional: habilita la entrega del razonamiento de heartbeat para mayor transparencia.
5. Opcional: usa contexto de bootstrap ligero si las ejecuciones de heartbeat solo necesitan `HEARTBEAT.md`.
6. Opcional: habilita sesiones aisladas para evitar enviar el historial completo de la conversación en cada heartbeat.
7. Opcional: restringe heartbeat a las horas activas (hora local).

Ejemplo de configuración:

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // entrega explícita al último contacto (el valor predeterminado es "none")
        directPolicy: "allow", // predeterminado: permite destinos directos/DM; establece "block" para suprimir
        lightContext: true, // opcional: solo inyecta HEARTBEAT.md de los archivos de bootstrap
        isolatedSession: true, // opcional: sesión nueva en cada ejecución (sin historial de conversación)
        // activeHours: { start: "08:00", end: "24:00" },
        // includeReasoning: true, // opcional: envía también un mensaje separado `Reasoning:`
      },
    },
  },
}
```

## Valores predeterminados

- Intervalo: `30m` (o `1h` cuando Anthropic OAuth/token es el modo de autenticación detectado, incluida la reutilización de Claude CLI). Establece `agents.defaults.heartbeat.every` o `agents.list[].heartbeat.every`; usa `0m` para desactivar.
- Cuerpo del prompt (configurable mediante `agents.defaults.heartbeat.prompt`):
  `Read HEARTBEAT.md if it exists (workspace context). Follow it strictly. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.`
- El prompt de heartbeat se envía **literalmente** como mensaje del usuario. El prompt del sistema
  incluye una sección “Heartbeat” y la ejecución se marca internamente.
- Las horas activas (`heartbeat.activeHours`) se comprueban en la zona horaria configurada.
  Fuera de la ventana, los heartbeat se omiten hasta el siguiente tick dentro de la ventana.

## Para qué sirve el prompt de heartbeat

El prompt predeterminado es intencionalmente amplio:

- **Tareas en segundo plano**: “Consider outstanding tasks” empuja al agente a revisar
  seguimientos (bandeja de entrada, calendario, recordatorios, trabajo en cola) y mostrar cualquier cosa urgente.
- **Comprobación con el humano**: “Checkup sometimes on your human during day time” impulsa una
  comprobación ligera ocasional de “¿necesitas algo?”, pero evita spam nocturno
  usando tu zona horaria local configurada (consulta [/concepts/timezone](/concepts/timezone)).

Heartbeat puede reaccionar a [tareas en segundo plano](/automation/tasks) completadas, pero una ejecución de heartbeat no crea por sí misma un registro de tarea.

Si quieres que un heartbeat haga algo muy específico (por ejemplo, “comprobar estadísticas de Gmail PubSub”
o “verificar el estado del gateway”), establece `agents.defaults.heartbeat.prompt` (o
`agents.list[].heartbeat.prompt`) con un cuerpo personalizado (enviado literalmente).

## Contrato de respuesta

- Si nada requiere atención, responde con **`HEARTBEAT_OK`**.
- Durante las ejecuciones de heartbeat, OpenClaw trata `HEARTBEAT_OK` como confirmación cuando aparece
  al **inicio o al final** de la respuesta. El token se elimina y la respuesta se
  descarta si el contenido restante es **≤ `ackMaxChars`** (predeterminado: 300).
- Si `HEARTBEAT_OK` aparece en la **mitad** de una respuesta, no se trata
  de forma especial.
- Para alertas, **no** incluyas `HEARTBEAT_OK`; devuelve solo el texto de la alerta.

Fuera de heartbeat, cualquier `HEARTBEAT_OK` suelto al inicio/final de un mensaje se elimina
y se registra; un mensaje que sea solo `HEARTBEAT_OK` se descarta.

## Configuración

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m", // predeterminado: 30m (0m desactiva)
        model: "anthropic/claude-opus-4-6",
        includeReasoning: false, // predeterminado: false (entrega un mensaje `Reasoning:` separado cuando está disponible)
        lightContext: false, // predeterminado: false; true conserva solo HEARTBEAT.md de los archivos de bootstrap del espacio de trabajo
        isolatedSession: false, // predeterminado: false; true ejecuta cada heartbeat en una sesión nueva (sin historial de conversación)
        target: "last", // predeterminado: none | opciones: last | none | <id de canal> (núcleo o plugin, por ejemplo "bluebubbles")
        to: "+15551234567", // anulación opcional específica del canal
        accountId: "ops-bot", // id de canal opcional para varias cuentas
        prompt: "Read HEARTBEAT.md if it exists (workspace context). Follow it strictly. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.",
        ackMaxChars: 300, // número máximo de caracteres permitidos después de HEARTBEAT_OK
      },
    },
  },
}
```

### Alcance y precedencia

- `agents.defaults.heartbeat` establece el comportamiento global de heartbeat.
- `agents.list[].heartbeat` se fusiona por encima; si algún agente tiene un bloque `heartbeat`, **solo esos agentes** ejecutan heartbeat.
- `channels.defaults.heartbeat` establece valores predeterminados de visibilidad para todos los canales.
- `channels.<channel>.heartbeat` anula los valores predeterminados del canal.
- `channels.<channel>.accounts.<id>.heartbeat` (canales con varias cuentas) anula la configuración por canal.

### Heartbeats por agente

Si alguna entrada de `agents.list[]` incluye un bloque `heartbeat`, **solo esos agentes**
ejecutan heartbeat. El bloque por agente se fusiona sobre `agents.defaults.heartbeat`
(así puedes establecer valores compartidos una sola vez y anularlos por agente).

Ejemplo: dos agentes; solo el segundo ejecuta heartbeat.

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // entrega explícita al último contacto (el valor predeterminado es "none")
      },
    },
    list: [
      { id: "main", default: true },
      {
        id: "ops",
        heartbeat: {
          every: "1h",
          target: "whatsapp",
          to: "+15551234567",
          prompt: "Read HEARTBEAT.md if it exists (workspace context). Follow it strictly. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.",
        },
      },
    ],
  },
}
```

### Ejemplo de horas activas

Restringe heartbeat al horario laboral en una zona horaria específica:

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // entrega explícita al último contacto (el valor predeterminado es "none")
        activeHours: {
          start: "09:00",
          end: "22:00",
          timezone: "America/New_York", // opcional; usa tu userTimezone si está definida; de lo contrario, la zona horaria del host
        },
      },
    },
  },
}
```

Fuera de esta ventana (antes de las 9 a. m. o después de las 10 p. m. hora del Este), los heartbeat se omiten. El siguiente tick programado dentro de la ventana se ejecutará con normalidad.

### Configuración 24/7

Si quieres que heartbeat se ejecute todo el día, usa uno de estos patrones:

- Omite `activeHours` por completo (sin restricción de ventana horaria; este es el comportamiento predeterminado).
- Establece una ventana de día completo: `activeHours: { start: "00:00", end: "24:00" }`.

No establezcas la misma hora en `start` y `end` (por ejemplo `08:00` a `08:00`).
Eso se trata como una ventana de anchura cero, por lo que heartbeat siempre se omite.

### Ejemplo de varias cuentas

Usa `accountId` para apuntar a una cuenta específica en canales con varias cuentas como Telegram:

```json5
{
  agents: {
    list: [
      {
        id: "ops",
        heartbeat: {
          every: "1h",
          target: "telegram",
          to: "12345678:topic:42", // opcional: enrutar a un tema/hilo específico
          accountId: "ops-bot",
        },
      },
    ],
  },
  channels: {
    telegram: {
      accounts: {
        "ops-bot": { botToken: "YOUR_TELEGRAM_BOT_TOKEN" },
      },
    },
  },
}
```

### Notas sobre los campos

- `every`: intervalo de heartbeat (cadena de duración; unidad predeterminada = minutos).
- `model`: anulación opcional de modelo para ejecuciones de heartbeat (`provider/model`).
- `includeReasoning`: cuando está habilitado, también entrega el mensaje `Reasoning:` separado cuando está disponible (con la misma forma que `/reasoning on`).
- `lightContext`: cuando es true, las ejecuciones de heartbeat usan contexto de bootstrap ligero y conservan solo `HEARTBEAT.md` de los archivos de bootstrap del espacio de trabajo.
- `isolatedSession`: cuando es true, cada heartbeat se ejecuta en una sesión nueva sin historial previo de conversación. Usa el mismo patrón de aislamiento que cron `sessionTarget: "isolated"`. Reduce drásticamente el costo en tokens por heartbeat. Combínalo con `lightContext: true` para obtener el máximo ahorro. El enrutamiento de entrega sigue usando el contexto de la sesión principal.
- `session`: clave de sesión opcional para ejecuciones de heartbeat.
  - `main` (predeterminado): sesión principal del agente.
  - Clave de sesión explícita (cópiala de `openclaw sessions --json` o de la [CLI de sesiones](/cli/sessions)).
  - Formatos de clave de sesión: consulta [Sesiones](/concepts/session) y [Grupos](/channels/groups).
- `target`:
  - `last`: entrega al último canal externo usado.
  - canal explícito: cualquier canal configurado o id de plugin, por ejemplo `discord`, `matrix`, `telegram` o `whatsapp`.
  - `none` (predeterminado): ejecuta heartbeat pero **no entrega** nada externamente.
- `directPolicy`: controla el comportamiento de entrega directa/DM:
  - `allow` (predeterminado): permite la entrega directa/DM de heartbeat.
  - `block`: suprime la entrega directa/DM (`reason=dm-blocked`).
- `to`: anulación opcional del destinatario (id específico del canal, por ejemplo E.164 para WhatsApp o un id de chat de Telegram). Para temas/hilos de Telegram, usa `<chatId>:topic:<messageThreadId>`.
- `accountId`: id de cuenta opcional para canales con varias cuentas. Cuando `target: "last"`, el id de cuenta se aplica al último canal resuelto si admite cuentas; de lo contrario, se ignora. Si el id de cuenta no coincide con una cuenta configurada para el canal resuelto, la entrega se omite.
- `prompt`: anula el cuerpo de prompt predeterminado (no se fusiona).
- `ackMaxChars`: número máximo de caracteres permitidos después de `HEARTBEAT_OK` antes de la entrega.
- `suppressToolErrorWarnings`: cuando es true, suprime las cargas útiles de advertencia de error de herramientas durante las ejecuciones de heartbeat.
- `activeHours`: restringe las ejecuciones de heartbeat a una ventana horaria. Objeto con `start` (HH:MM, inclusivo; usa `00:00` para el inicio del día), `end` (HH:MM exclusivo; `24:00` permitido para el fin del día) y `timezone` opcional.
  - Omitido o `"user"`: usa tu `agents.defaults.userTimezone` si está definida; de lo contrario, recurre a la zona horaria del sistema host.
  - `"local"`: siempre usa la zona horaria del sistema host.
  - Cualquier identificador IANA (por ejemplo `America/New_York`): se usa directamente; si no es válido, recurre al comportamiento de `"user"` descrito arriba.
  - `start` y `end` no deben ser iguales para una ventana activa; valores iguales se tratan como anchura cero (siempre fuera de la ventana).
  - Fuera de la ventana activa, los heartbeat se omiten hasta el siguiente tick dentro de la ventana.

## Comportamiento de entrega

- Heartbeat se ejecuta en la sesión principal del agente por defecto (`agent:<id>:<mainKey>`),
  o en `global` cuando `session.scope = "global"`. Establece `session` para anularlo a una
  sesión de canal específica (Discord/WhatsApp/etc.).
- `session` solo afecta el contexto de ejecución; la entrega está controlada por `target` y `to`.
- Para entregar a un canal/destinatario específico, establece `target` + `to`. Con
  `target: "last"`, la entrega usa el último canal externo de esa sesión.
- Las entregas de heartbeat permiten destinos directos/DM de forma predeterminada. Establece `directPolicy: "block"` para suprimir envíos a destinos directos sin dejar de ejecutar el turno de heartbeat.
- Si la cola principal está ocupada, el heartbeat se omite y se reintenta más tarde.
- Si `target` no se resuelve a ningún destino externo, la ejecución sigue ocurriendo pero no
  se envía ningún mensaje saliente.
- Si `showOk`, `showAlerts` y `useIndicator` están todos desactivados, la ejecución se omite desde el principio como `reason=alerts-disabled`.
- Si solo está desactivada la entrega de alertas, OpenClaw aún puede ejecutar heartbeat, actualizar las marcas de tiempo de tareas vencidas, restaurar la marca de tiempo de inactividad de la sesión y suprimir la carga útil externa de la alerta.
- Las respuestas solo de heartbeat **no** mantienen viva la sesión; el último `updatedAt`
  se restaura para que la caducidad por inactividad se comporte con normalidad.
- Las [tareas en segundo plano](/automation/tasks) desacopladas pueden poner en cola un evento del sistema y activar heartbeat cuando la sesión principal debe notar algo rápidamente. Esa activación no hace que heartbeat se convierta en una tarea en segundo plano.

## Controles de visibilidad

De forma predeterminada, las confirmaciones `HEARTBEAT_OK` se suprimen mientras que el contenido de alerta
se entrega. Puedes ajustar esto por canal o por cuenta:

```yaml
channels:
  defaults:
    heartbeat:
      showOk: false # Oculta HEARTBEAT_OK (predeterminado)
      showAlerts: true # Muestra mensajes de alerta (predeterminado)
      useIndicator: true # Emite eventos de indicador (predeterminado)
  telegram:
    heartbeat:
      showOk: true # Muestra confirmaciones OK en Telegram
  whatsapp:
    accounts:
      work:
        heartbeat:
          showAlerts: false # Suprime la entrega de alertas para esta cuenta
```

Precedencia: por cuenta → por canal → valores predeterminados del canal → valores predeterminados integrados.

### Qué hace cada indicador

- `showOk`: envía una confirmación `HEARTBEAT_OK` cuando el modelo devuelve una respuesta solo de OK.
- `showAlerts`: envía el contenido de la alerta cuando el modelo devuelve una respuesta distinta de OK.
- `useIndicator`: emite eventos de indicador para superficies de estado de la IU.

Si **los tres** son false, OpenClaw omite por completo la ejecución de heartbeat (sin llamada al modelo).

### Ejemplos por canal frente a por cuenta

```yaml
channels:
  defaults:
    heartbeat:
      showOk: false
      showAlerts: true
      useIndicator: true
  slack:
    heartbeat:
      showOk: true # todas las cuentas de Slack
    accounts:
      ops:
        heartbeat:
          showAlerts: false # suprime alertas solo para la cuenta ops
  telegram:
    heartbeat:
      showOk: true
```

### Patrones comunes

| Objetivo                                  | Configuración                                                                             |
| ----------------------------------------- | ----------------------------------------------------------------------------------------- |
| Comportamiento predeterminado (OK silenciosos, alertas activadas) | _(no se necesita configuración)_                                            |
| Completamente silencioso (sin mensajes, sin indicador) | `channels.defaults.heartbeat: { showOk: false, showAlerts: false, useIndicator: false }` |
| Solo indicador (sin mensajes)             | `channels.defaults.heartbeat: { showOk: false, showAlerts: false, useIndicator: true }`  |
| OK solo en un canal                       | `channels.telegram.heartbeat: { showOk: true }`                                           |

## `HEARTBEAT.md` (opcional)

Si existe un archivo `HEARTBEAT.md` en el espacio de trabajo, el prompt predeterminado le indica al
agente que lo lea. Piensa en él como tu “lista de verificación de heartbeat”: pequeño, estable y
seguro para incluir cada 30 minutos.

Si `HEARTBEAT.md` existe pero está efectivamente vacío (solo líneas en blanco y encabezados de Markdown
como `# Heading`), OpenClaw omite la ejecución de heartbeat para ahorrar llamadas API.
Esa omisión se informa como `reason=empty-heartbeat-file`.
Si el archivo no existe, heartbeat sigue ejecutándose y el modelo decide qué hacer.

Mantenlo pequeño (lista de verificación o recordatorios breves) para evitar inflar el prompt.

Ejemplo de `HEARTBEAT.md`:

```md
# Heartbeat checklist

- Quick scan: anything urgent in inboxes?
- If it’s daytime, do a lightweight check-in if nothing else is pending.
- If a task is blocked, write down _what is missing_ and ask Peter next time.
```

### Bloques `tasks:`

`HEARTBEAT.md` también admite un pequeño bloque estructurado `tasks:` para comprobaciones
basadas en intervalos dentro del propio heartbeat.

Ejemplo:

```md
tasks:

- name: inbox-triage
  interval: 30m
  prompt: "Check for urgent unread emails and flag anything time sensitive."
- name: calendar-scan
  interval: 2h
  prompt: "Check for upcoming meetings that need prep or follow-up."

# Additional instructions

- Keep alerts short.
- If nothing needs attention after all due tasks, reply HEARTBEAT_OK.
```

Comportamiento:

- OpenClaw analiza el bloque `tasks:` y comprueba cada tarea con su propio `interval`.
- Solo las tareas **vencidas** se incluyen en el prompt de heartbeat de ese tick.
- Si no hay tareas vencidas, heartbeat se omite por completo (`reason=no-tasks-due`) para evitar una llamada inútil al modelo.
- El contenido que no pertenece a tareas en `HEARTBEAT.md` se conserva y se añade como contexto adicional después de la lista de tareas vencidas.
- Las marcas de tiempo de la última ejecución de las tareas se almacenan en el estado de sesión (`heartbeatTaskState`), por lo que los intervalos sobreviven a reinicios normales.
- Las marcas de tiempo de las tareas solo avanzan después de que una ejecución de heartbeat completa su ruta normal de respuesta. Las ejecuciones omitidas `empty-heartbeat-file` / `no-tasks-due` no marcan tareas como completadas.

El modo de tareas es útil cuando quieres que un solo archivo heartbeat contenga varias comprobaciones periódicas sin pagar por todas ellas en cada tick.

### ¿Puede el agente actualizar `HEARTBEAT.md`?

Sí, si se lo pides.

`HEARTBEAT.md` es simplemente un archivo normal en el espacio de trabajo del agente, así que puedes decirle al
agente (en un chat normal) algo como:

- “Actualiza `HEARTBEAT.md` para añadir una comprobación diaria del calendario.”
- “Reescribe `HEARTBEAT.md` para que sea más corto y esté centrado en seguimientos de la bandeja de entrada.”

Si quieres que esto ocurra de forma proactiva, también puedes incluir una línea explícita en
tu prompt de heartbeat como: “Si la lista de verificación queda obsoleta, actualiza HEARTBEAT.md
con una mejor.”

Nota de seguridad: no pongas secretos (claves API, números de teléfono, tokens privados) en
`HEARTBEAT.md`, ya que pasa a formar parte del contexto del prompt.

## Activación manual (bajo demanda)

Puedes poner en cola un evento del sistema y activar un heartbeat inmediato con:

```bash
openclaw system event --text "Check for urgent follow-ups" --mode now
```

Si varios agentes tienen `heartbeat` configurado, una activación manual ejecuta inmediatamente
cada uno de esos heartbeat de agente.

Usa `--mode next-heartbeat` para esperar al siguiente tick programado.

## Entrega de razonamiento (opcional)

De forma predeterminada, los heartbeat entregan solo la carga útil final de “respuesta”.

Si quieres transparencia, habilita:

- `agents.defaults.heartbeat.includeReasoning: true`

Cuando está habilitado, los heartbeat también entregan un mensaje separado con prefijo
`Reasoning:` (con la misma forma que `/reasoning on`). Esto puede ser útil cuando el agente
está gestionando varias sesiones/codexes y quieres ver por qué decidió enviarte un mensaje,
pero también puede filtrar más detalles internos de los que quieres. Es preferible mantenerlo
desactivado en chats de grupo.

## Consideraciones de costo

Los heartbeat ejecutan turnos completos del agente. Los intervalos más cortos consumen más tokens. Para reducir costo:

- Usa `isolatedSession: true` para evitar enviar todo el historial de conversación (~100K tokens bajan a ~2-5K por ejecución).
- Usa `lightContext: true` para limitar los archivos de bootstrap solo a `HEARTBEAT.md`.
- Establece un `model` más barato (por ejemplo `ollama/llama3.2:1b`).
- Mantén `HEARTBEAT.md` pequeño.
- Usa `target: "none"` si solo quieres actualizaciones de estado internas.

## Relacionado

- [Automatización y tareas](/automation) — todos los mecanismos de automatización de un vistazo
- [Tareas en segundo plano](/automation/tasks) — cómo se rastrea el trabajo desacoplado
- [Zona horaria](/concepts/timezone) — cómo afecta la zona horaria a la programación de heartbeat
- [Solución de problemas](/automation/cron-jobs#troubleshooting) — depuración de problemas de automatización
