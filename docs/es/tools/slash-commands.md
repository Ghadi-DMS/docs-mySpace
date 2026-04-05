---
read_when:
    - Usar o configurar comandos de chat
    - Depurar el enrutamiento o los permisos de comandos
summary: 'Comandos con barra: texto frente a nativos, configuración y comandos compatibles'
title: Comandos con barra
x-i18n:
    generated_at: "2026-04-05T12:57:21Z"
    model: gpt-5.4
    provider: openai
    source_hash: 8c91437140732d9accca1094f07b9e05f861a75ac344531aa24cc2ffe000630f
    source_path: tools/slash-commands.md
    workflow: 15
---

# Comandos con barra

Los comandos son gestionados por el Gateway. La mayoría de los comandos deben enviarse como un mensaje **independiente** que empiece por `/`.
El comando de chat bash solo para host usa `! <cmd>` (con `/bash <cmd>` como alias).

Hay dos sistemas relacionados:

- **Comandos**: mensajes independientes `/...`.
- **Directivas**: `/think`, `/fast`, `/verbose`, `/reasoning`, `/elevated`, `/exec`, `/model`, `/queue`.
  - Las directivas se eliminan del mensaje antes de que el modelo lo vea.
  - En mensajes de chat normales (no solo directivas), se tratan como “pistas en línea” y **no** hacen persistir ajustes de sesión.
  - En mensajes solo de directivas (el mensaje contiene únicamente directivas), persisten en la sesión y responden con una confirmación.
  - Las directivas solo se aplican a **remitentes autorizados**. Si se establece `commands.allowFrom`, esa es la única
    lista de permitidos usada; en caso contrario, la autorización proviene de las listas de permitidos/emparejamiento del canal más `commands.useAccessGroups`.
    Los remitentes no autorizados ven las directivas tratadas como texto sin formato.

También hay algunos **atajos en línea** (solo para remitentes permitidos/autorizados): `/help`, `/commands`, `/status`, `/whoami` (`/id`).
Se ejecutan inmediatamente, se eliminan antes de que el modelo vea el mensaje y el texto restante continúa por el flujo normal.

## Configuración

```json5
{
  commands: {
    native: "auto",
    nativeSkills: "auto",
    text: true,
    bash: false,
    bashForegroundMs: 2000,
    config: false,
    mcp: false,
    plugins: false,
    debug: false,
    restart: false,
    allowFrom: {
      "*": ["user1"],
      discord: ["user:123"],
    },
    useAccessGroups: true,
  },
}
```

- `commands.text` (predeterminado `true`) habilita el análisis de `/...` en mensajes de chat.
  - En superficies sin comandos nativos (WhatsApp/WebChat/Signal/iMessage/Google Chat/Microsoft Teams), los comandos de texto siguen funcionando aunque lo establezcas en `false`.
- `commands.native` (predeterminado `"auto"`) registra comandos nativos.
  - Auto: activado para Discord/Telegram; desactivado para Slack (hasta que añadas comandos con barra); ignorado para proveedores sin soporte nativo.
  - Establece `channels.discord.commands.native`, `channels.telegram.commands.native` o `channels.slack.commands.native` para anularlo por proveedor (bool o `"auto"`).
  - `false` borra en el arranque los comandos registrados previamente en Discord/Telegram. Los comandos de Slack se gestionan en la app de Slack y no se eliminan automáticamente.
- `commands.nativeSkills` (predeterminado `"auto"`) registra comandos de **Skills** de forma nativa cuando hay soporte.
  - Auto: activado para Discord/Telegram; desactivado para Slack (Slack requiere crear un comando con barra por Skill).
  - Establece `channels.discord.commands.nativeSkills`, `channels.telegram.commands.nativeSkills` o `channels.slack.commands.nativeSkills` para anularlo por proveedor (bool o `"auto"`).
- `commands.bash` (predeterminado `false`) habilita `! <cmd>` para ejecutar comandos de shell en el host (`/bash <cmd>` es un alias; requiere listas de permitidos de `tools.elevated`).
- `commands.bashForegroundMs` (predeterminado `2000`) controla cuánto tiempo espera bash antes de cambiar a modo en segundo plano (`0` lo manda al fondo inmediatamente).
- `commands.config` (predeterminado `false`) habilita `/config` (lee/escribe `openclaw.json`).
- `commands.mcp` (predeterminado `false`) habilita `/mcp` (lee/escribe configuración MCP gestionada por OpenClaw en `mcp.servers`).
- `commands.plugins` (predeterminado `false`) habilita `/plugins` (detección/estado de plugins más controles de instalación + activación/desactivación).
- `commands.debug` (predeterminado `false`) habilita `/debug` (anulaciones solo de runtime).
- `commands.allowFrom` (opcional) establece una lista de permitidos por proveedor para la autorización de comandos. Cuando se configura, es la
  única fuente de autorización para comandos y directivas (se ignoran las listas de permitidos/emparejamiento del canal y `commands.useAccessGroups`).
  Usa `"*"` para un valor global predeterminado; las claves específicas por proveedor lo anulan.
- `commands.useAccessGroups` (predeterminado `true`) aplica listas de permitidos/políticas a los comandos cuando `commands.allowFrom` no está establecido.

## Lista de comandos

Texto + nativos (cuando están habilitados):

- `/help`
- `/commands`
- `/tools [compact|verbose]` (muestra lo que el agente actual puede usar ahora mismo; `verbose` añade descripciones)
- `/skill <name> [input]` (ejecuta una Skill por nombre)
- `/status` (muestra el estado actual; incluye uso/cuota del proveedor para el proveedor del modelo actual cuando está disponible)
- `/tasks` (lista tareas en segundo plano de la sesión actual; muestra detalles de tareas activas y recientes con recuentos de fallback locales del agente)
- `/allowlist` (listar/añadir/eliminar entradas de la lista de permitidos)
- `/approve <id> <decision>` (resuelve solicitudes de aprobación de exec; usa el mensaje de aprobación pendiente para ver las decisiones disponibles)
- `/context [list|detail|json]` (explica “context”; `detail` muestra tamaño por archivo + por herramienta + por Skill + del prompt del sistema)
- `/btw <question>` (hace una pregunta lateral efímera sobre la sesión actual sin cambiar el contexto futuro de la sesión; consulta [/tools/btw](/tools/btw))
- `/export-session [path]` (alias: `/export`) (exporta la sesión actual a HTML con el prompt del sistema completo)
- `/whoami` (muestra tu id de remitente; alias: `/id`)
- `/session idle <duration|off>` (gestiona el auto-unfocus por inactividad para enlaces de hilos enfocados)
- `/session max-age <duration|off>` (gestiona el auto-unfocus por edad máxima estricta para enlaces de hilos enfocados)
- `/subagents list|kill|log|info|send|steer|spawn` (inspecciona, controla o crea ejecuciones de subagentes para la sesión actual)
- `/acp spawn|cancel|steer|close|status|set-mode|set|cwd|permissions|timeout|model|reset-options|doctor|install|sessions` (inspecciona y controla sesiones de runtime ACP)
- `/agents` (lista agentes enlazados al hilo para esta sesión)
- `/focus <target>` (Discord: vincula este hilo, o un hilo nuevo, a un objetivo de sesión/subagente)
- `/unfocus` (Discord: elimina el enlace actual del hilo)
- `/kill <id|#|all>` (aborta inmediatamente uno o todos los subagentes en ejecución de esta sesión; sin mensaje de confirmación)
- `/steer <id|#> <message>` (redirige inmediatamente un subagente en ejecución: durante la ejecución cuando sea posible; si no, aborta el trabajo actual y reinicia con el mensaje de redirección)
- `/tell <id|#> <message>` (alias de `/steer`)
- `/config show|get|set|unset` (persiste configuración en disco, solo propietario; requiere `commands.config: true`)
- `/mcp show|get|set|unset` (gestiona la configuración de servidores MCP de OpenClaw, solo propietario; requiere `commands.mcp: true`)
- `/plugins list|show|get|install|enable|disable` (inspecciona plugins detectados, instala nuevos y alterna su activación; solo propietario para escrituras; requiere `commands.plugins: true`)
  - `/plugin` es un alias de `/plugins`.
  - `/plugin install <spec>` acepta las mismas especificaciones de plugin que `openclaw plugins install`: ruta/archivo local, paquete npm o `clawhub:<pkg>`.
  - Las escrituras de activación/desactivación siguen respondiendo con una pista de reinicio. En un gateway en primer plano observado, OpenClaw puede realizar ese reinicio automáticamente justo después de la escritura.
- `/debug show|set|unset|reset` (anulaciones de runtime, solo propietario; requiere `commands.debug: true`)
- `/usage off|tokens|full|cost` (pie de uso por respuesta o resumen local de costos)
- `/tts off|always|inbound|tagged|status|provider|limit|summary|audio` (controla TTS; consulta [/tts](/tools/tts))
  - Discord: el comando nativo es `/voice` (Discord reserva `/tts`); el texto `/tts` sigue funcionando.
- `/stop`
- `/restart`
- `/dock-telegram` (alias: `/dock_telegram`) (cambia las respuestas a Telegram)
- `/dock-discord` (alias: `/dock_discord`) (cambia las respuestas a Discord)
- `/dock-slack` (alias: `/dock_slack`) (cambia las respuestas a Slack)
- `/activation mention|always` (solo grupos)
- `/send on|off|inherit` (solo propietario)
- `/reset` o `/new [model]` (pista opcional de modelo; el resto se pasa tal cual)
- `/think <off|minimal|low|medium|high|xhigh>` (opciones dinámicas según modelo/proveedor; alias: `/thinking`, `/t`)
- `/fast status|on|off` (si omites el argumento, muestra el estado efectivo actual del modo fast)
- `/verbose on|full|off` (alias: `/v`)
- `/reasoning on|off|stream` (alias: `/reason`; cuando está activado, envía un mensaje aparte con el prefijo `Reasoning:`; `stream` = solo borrador de Telegram)
- `/elevated on|off|ask|full` (alias: `/elev`; `full` omite aprobaciones de exec)
- `/exec host=<auto|sandbox|gateway|node> security=<deny|allowlist|full> ask=<off|on-miss|always> node=<id>` (envía `/exec` para mostrar el actual)
- `/model <name>` (alias: `/models`; o `/<alias>` desde `agents.defaults.models.*.alias`)
- `/queue <mode>` (más opciones como `debounce:2s cap:25 drop:summarize`; envía `/queue` para ver la configuración actual)
- `/bash <command>` (solo host; alias de `! <command>`; requiere `commands.bash: true` + listas de permitidos de `tools.elevated`)
- `/dreaming [off|core|rem|deep|status|help]` (activa/desactiva el modo dreaming o muestra el estado; consulta [Dreaming](/es/concepts/memory-dreaming))

Solo texto:

- `/compact [instructions]` (consulta [/concepts/compaction](/es/concepts/compaction))
- `! <command>` (solo host; uno a la vez; usa `!poll` + `!stop` para trabajos de larga duración)
- `!poll` (comprueba salida/estado; acepta `sessionId` opcional; `/bash poll` también funciona)
- `!stop` (detiene el trabajo bash en ejecución; acepta `sessionId` opcional; `/bash stop` también funciona)

Notas:

- Los comandos aceptan `:` opcional entre el comando y los argumentos (por ejemplo, `/think: high`, `/send: on`, `/help:`).
- `/new <model>` acepta un alias de modelo, `provider/model` o un nombre de proveedor (coincidencia difusa); si no hay coincidencia, el texto se trata como cuerpo del mensaje.
- Para el desglose completo del uso del proveedor, usa `openclaw status --usage`.
- `/allowlist add|remove` requiere `commands.config=true` y respeta `configWrites` del canal.
- En canales con varias cuentas, `/allowlist --account <id>` orientado a configuración y `/config set channels.<provider>.accounts.<id>...` también respetan `configWrites` de la cuenta de destino.
- `/usage` controla el pie de uso por respuesta; `/usage cost` imprime un resumen local de costos a partir de los registros de sesión de OpenClaw.
- `/restart` está habilitado de forma predeterminada; establece `commands.restart: false` para deshabilitarlo.
- Comando nativo solo de Discord: `/vc join|leave|status` controla canales de voz (requiere `channels.discord.voice` y comandos nativos; no está disponible como texto).
- Los comandos de enlace de hilos de Discord (`/focus`, `/unfocus`, `/agents`, `/session idle`, `/session max-age`) requieren que los enlaces efectivos de hilos estén habilitados (`session.threadBindings.enabled` y/o `channels.discord.threadBindings.enabled`).
- Referencia de comandos ACP y comportamiento de runtime: [Agentes ACP](/tools/acp-agents).
- `/verbose` está pensado para depuración y visibilidad adicional; mantenlo **desactivado** en uso normal.
- `/fast on|off` hace persistir una anulación de sesión. Usa la opción `inherit` de la UI de sesiones para borrarla y volver a los valores predeterminados de la configuración.
- `/fast` es específico del proveedor: OpenAI/OpenAI Codex lo asignan a `service_tier=priority` en endpoints nativos de Responses, mientras que las solicitudes directas públicas a Anthropic, incluido el tráfico autenticado con OAuth enviado a `api.anthropic.com`, lo asignan a `service_tier=auto` o `standard_only`. Consulta [OpenAI](/es/providers/openai) y [Anthropic](/es/providers/anthropic).
- Los resúmenes de fallo de herramientas siguen mostrándose cuando corresponde, pero el texto detallado del fallo solo se incluye cuando `/verbose` está en `on` o `full`.
- `/reasoning` (y `/verbose`) son arriesgados en entornos de grupo: pueden revelar razonamiento interno o salida de herramientas que no pretendías exponer. Es preferible dejarlos desactivados, especialmente en chats grupales.
- `/model` hace persistir el nuevo modelo de la sesión inmediatamente.
- Si el agente está inactivo, la siguiente ejecución lo usa de inmediato.
- Si ya hay una ejecución activa, OpenClaw marca un cambio en vivo como pendiente y solo reinicia con el nuevo modelo en un punto limpio de reintento.
- Si la actividad de herramientas o la salida de respuesta ya ha comenzado, el cambio pendiente puede quedarse en cola hasta una oportunidad posterior de reintento o el siguiente turno del usuario.
- **Ruta rápida:** los mensajes que solo contienen comandos de remitentes permitidos se gestionan inmediatamente (omiten cola + modelo).
- **Control por mención en grupos:** los mensajes que solo contienen comandos de remitentes permitidos omiten los requisitos de mención.
- **Atajos en línea (solo remitentes permitidos):** ciertos comandos también funcionan cuando están incrustados en un mensaje normal y se eliminan antes de que el modelo vea el texto restante.
  - Ejemplo: `hey /status` activa una respuesta de estado, y el texto restante continúa por el flujo normal.
- Actualmente: `/help`, `/commands`, `/status`, `/whoami` (`/id`).
- Los mensajes solo de comandos no autorizados se ignoran silenciosamente, y los tokens `/...` en línea se tratan como texto sin formato.
- **Comandos de Skills:** las Skills `user-invocable` se exponen como comandos con barra. Los nombres se sanean a `a-z0-9_` (máx. 32 caracteres); las colisiones reciben sufijos numéricos (por ejemplo, `_2`).
  - `/skill <name> [input]` ejecuta una Skill por nombre (útil cuando los límites de comandos nativos impiden comandos por Skill).
  - De forma predeterminada, los comandos de Skills se reenvían al modelo como una solicitud normal.
  - Las Skills pueden declarar opcionalmente `command-dispatch: tool` para enrutar el comando directamente a una herramienta (determinista, sin modelo).
  - Ejemplo: `/prose` (plugin OpenProse) — consulta [OpenProse](/es/prose).
- **Argumentos de comandos nativos:** Discord usa autocompletado para opciones dinámicas (y menús de botones cuando omites argumentos obligatorios). Telegram y Slack muestran un menú de botones cuando un comando admite opciones y omites el argumento.

## `/tools`

`/tools` responde a una pregunta de runtime, no de configuración: **qué puede usar este agente ahora mismo en
esta conversación**.

- `/tools` predeterminado es compacto y está optimizado para un escaneo rápido.
- `/tools verbose` añade descripciones breves.
- Las superficies de comandos nativos que admiten argumentos exponen el mismo cambio de modo `compact|verbose`.
- Los resultados están acotados a la sesión, así que cambiar de agente, canal, hilo, autorización del remitente o modelo puede
  cambiar la salida.
- `/tools` incluye herramientas realmente accesibles en runtime, incluidas herramientas core, herramientas de plugins conectados y herramientas propiedad del canal.

Para editar perfiles y anulaciones, usa el panel de herramientas de la Control UI o las superficies de configuración/catálogo en lugar de
tratar `/tools` como un catálogo estático.

## Superficies de uso (qué se muestra y dónde)

- **Uso/cuota del proveedor** (ejemplo: “Claude 80% left”) aparece en `/status` para el proveedor del modelo actual cuando el seguimiento de uso está habilitado. OpenClaw normaliza las ventanas del proveedor a `% left`; para MiniMax, los campos porcentuales solo-de-restante se invierten antes de mostrarse, y las respuestas `model_remains` prefieren la entrada del modelo de chat junto con una etiqueta de plan etiquetada por modelo.
- **Líneas de tokens/caché** en `/status` pueden recurrir a la entrada de uso de transcripción más reciente cuando la instantánea activa de la sesión es escasa. Los valores activos distintos de cero siguen prevaleciendo, y el fallback de transcripción también puede recuperar la etiqueta del modelo activo de runtime más un total orientado al prompt mayor cuando los totales almacenados faltan o son menores.
- **Tokens/costo por respuesta** se controla con `/usage off|tokens|full` (se agrega a las respuestas normales).
- `/model status` trata sobre **modelos/auth/endpoints**, no sobre uso.

## Selección de modelo (`/model`)

`/model` está implementado como una directiva.

Ejemplos:

```
/model
/model list
/model 3
/model openai/gpt-5.4
/model opus@anthropic:default
/model status
```

Notas:

- `/model` y `/model list` muestran un selector compacto numerado (familia de modelos + proveedores disponibles).
- En Discord, `/model` y `/models` abren un selector interactivo con menús desplegables de proveedor y modelo más un paso de envío.
- `/model <#>` selecciona desde ese selector (y prefiere el proveedor actual cuando es posible).
- `/model status` muestra la vista detallada, incluido el endpoint configurado del proveedor (`baseUrl`) y el modo de API (`api`) cuando está disponible.

## Anulaciones de depuración

`/debug` te permite establecer anulaciones de configuración **solo de runtime** (memoria, no disco). Solo propietario. Deshabilitado de forma predeterminada; habilítalo con `commands.debug: true`.

Ejemplos:

```
/debug show
/debug set messages.responsePrefix="[openclaw]"
/debug set channels.whatsapp.allowFrom=["+1555","+4477"]
/debug unset messages.responsePrefix
/debug reset
```

Notas:

- Las anulaciones se aplican inmediatamente a las nuevas lecturas de configuración, pero **no** escriben en `openclaw.json`.
- Usa `/debug reset` para borrar todas las anulaciones y volver a la configuración en disco.

## Actualizaciones de configuración

`/config` escribe en tu configuración en disco (`openclaw.json`). Solo propietario. Deshabilitado de forma predeterminada; habilítalo con `commands.config: true`.

Ejemplos:

```
/config show
/config show messages.responsePrefix
/config get messages.responsePrefix
/config set messages.responsePrefix="[openclaw]"
/config unset messages.responsePrefix
```

Notas:

- La configuración se valida antes de escribir; los cambios no válidos se rechazan.
- Las actualizaciones de `/config` persisten tras reinicios.

## Actualizaciones de MCP

`/mcp` escribe definiciones de servidores MCP gestionadas por OpenClaw bajo `mcp.servers`. Solo propietario. Deshabilitado de forma predeterminada; habilítalo con `commands.mcp: true`.

Ejemplos:

```text
/mcp show
/mcp show context7
/mcp set context7={"command":"uvx","args":["context7-mcp"]}
/mcp unset context7
```

Notas:

- `/mcp` guarda la configuración en la configuración de OpenClaw, no en ajustes del proyecto propiedad de Pi.
- Los adaptadores de runtime deciden qué transportes son realmente ejecutables.

## Actualizaciones de plugins

`/plugins` permite a los operadores inspeccionar plugins detectados y alternar su activación en la configuración. Los flujos de solo lectura pueden usar `/plugin` como alias. Deshabilitado de forma predeterminada; habilítalo con `commands.plugins: true`.

Ejemplos:

```text
/plugins
/plugins list
/plugin show context7
/plugins enable context7
/plugins disable context7
```

Notas:

- `/plugins list` y `/plugins show` usan detección real de plugins frente al espacio de trabajo actual más la configuración en disco.
- `/plugins enable|disable` actualiza solo la configuración del plugin; no instala ni desinstala plugins.
- Después de cambios de activación/desactivación, reinicia el gateway para aplicarlos.

## Notas de superficie

- **Los comandos de texto** se ejecutan en la sesión de chat normal (los mensajes directos comparten `main`, los grupos tienen su propia sesión).
- **Los comandos nativos** usan sesiones aisladas:
  - Discord: `agent:<agentId>:discord:slash:<userId>`
  - Slack: `agent:<agentId>:slack:slash:<userId>` (prefijo configurable mediante `channels.slack.slashCommand.sessionPrefix`)
  - Telegram: `telegram:slash:<userId>` (apunta a la sesión del chat mediante `CommandTargetSessionKey`)
- **`/stop`** apunta a la sesión de chat activa para poder abortar la ejecución actual.
- **Slack:** `channels.slack.slashCommand` sigue siendo compatible para un único comando estilo `/openclaw`. Si habilitas `commands.native`, debes crear un comando con barra de Slack por cada comando integrado (los mismos nombres que `/help`). Los menús de argumentos de comandos para Slack se entregan como botones efímeros de Block Kit.
  - Excepción nativa de Slack: registra `/agentstatus` (no `/status`) porque Slack reserva `/status`. El texto `/status` sigue funcionando en mensajes de Slack.

## Preguntas laterales con BTW

`/btw` es una **pregunta lateral** rápida sobre la sesión actual.

A diferencia del chat normal:

- usa la sesión actual como contexto de fondo,
- se ejecuta como una llamada independiente **sin herramientas** y de una sola vez,
- no cambia el contexto futuro de la sesión,
- no se escribe en el historial de la transcripción,
- se entrega como un resultado lateral en vivo en lugar de como un mensaje normal del asistente.

Eso hace que `/btw` sea útil cuando quieres una aclaración temporal mientras la
tarea principal sigue en marcha.

Ejemplo:

```text
/btw what are we doing right now?
```

Consulta [Preguntas laterales con BTW](/tools/btw) para ver el comportamiento completo y los
detalles de UX del cliente.
