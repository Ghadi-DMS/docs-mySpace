---
read_when:
    - Ejecutas harnesses de programación mediante ACP
    - Configuras sesiones ACP vinculadas a conversaciones en canales de mensajería
    - Vinculas una conversación de un canal de mensajes a una sesión ACP persistente
    - Solucionas problemas del backend ACP y del cableado de plugins
    - Operas comandos `/acp` desde el chat
summary: Usa sesiones de runtime ACP para Codex, Claude Code, Cursor, Gemini CLI, OpenClaw ACP y otros agentes de harness
title: Agentes ACP
x-i18n:
    generated_at: "2026-04-05T12:56:03Z"
    model: gpt-5.4
    provider: openai
    source_hash: 47063abc8170129cd22808d9a4b23160d0f340f6dc789907589d349f68c12e3e
    source_path: tools/acp-agents.md
    workflow: 15
---

# Agentes ACP

Las sesiones de [Agent Client Protocol (ACP)](https://agentclientprotocol.com/) permiten que OpenClaw ejecute harnesses de programación externos (por ejemplo Pi, Claude Code, Codex, Cursor, Copilot, OpenClaw ACP, OpenCode, Gemini CLI y otros harnesses ACPX compatibles) mediante un plugin de backend ACP.

Si le pides a OpenClaw en lenguaje natural que "ejecute esto en Codex" o que "inicie Claude Code en un hilo", OpenClaw debe enrutar esa solicitud al runtime ACP (no al runtime nativo de subagentes). Cada creación de sesión ACP se rastrea como una [tarea en segundo plano](/es/automation/tasks).

Si quieres que Codex o Claude Code se conecten como un cliente MCP externo directamente
a conversaciones de canales existentes de OpenClaw, usa [`openclaw mcp serve`](/cli/mcp)
en lugar de ACP.

## ¿Qué página necesito?

Hay tres superficies cercanas que es fácil confundir:

| Quieres...                                                                     | Usa esto                              | Notas                                                                                                       |
| ---------------------------------------------------------------------------------- | ------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| Ejecutar Codex, Claude Code, Gemini CLI u otro harness externo _a través de_ OpenClaw | Esta página: agentes ACP                 | Sesiones vinculadas al chat, `/acp spawn`, `sessions_spawn({ runtime: "acp" })`, tareas en segundo plano, controles de runtime |
| Exponer una sesión de OpenClaw Gateway _como_ un servidor ACP para un editor o cliente      | [`openclaw acp`](/cli/acp)            | Modo puente. El IDE/cliente habla ACP con OpenClaw por stdio/WebSocket                                          |
| Reutilizar una CLI de IA local como modelo de respaldo solo de texto                                 | [Backends de CLI](/es/gateway/cli-backends) | No es ACP. Sin herramientas de OpenClaw, sin controles ACP, sin runtime de harness                                             |

## ¿Esto funciona listo para usar?

Normalmente, sí.

- Las instalaciones nuevas ahora incluyen el plugin de runtime `acpx` integrado habilitado de forma predeterminada.
- El plugin `acpx` integrado prefiere su binario `acpx` fijado localmente en el plugin.
- Al iniciar, OpenClaw sondea ese binario y lo autorrepara si es necesario.
- Empieza con `/acp doctor` si quieres una comprobación rápida de disponibilidad.

Lo que todavía puede ocurrir en el primer uso:

- Un adaptador de harness de destino puede descargarse bajo demanda con `npx` la primera vez que uses ese harness.
- La autenticación del proveedor aún debe existir en el host para ese harness.
- Si el host no tiene acceso a npm/red, las descargas iniciales del adaptador pueden fallar hasta que la caché se precaliente o el adaptador se instale de otra manera.

Ejemplos:

- `/acp spawn codex`: OpenClaw debería estar listo para inicializar `acpx`, pero el adaptador ACP de Codex aún podría necesitar una descarga en el primer uso.
- `/acp spawn claude`: la misma situación para el adaptador ACP de Claude, además de la autenticación del lado de Claude en ese host.

## Flujo rápido para operadores

Usa esto cuando quieras un runbook práctico de `/acp`:

1. Crea una sesión:
   - `/acp spawn codex --bind here`
   - `/acp spawn codex --mode persistent --thread auto`
2. Trabaja en la conversación o hilo vinculado (o apunta explícitamente a esa clave de sesión).
3. Comprueba el estado del runtime:
   - `/acp status`
4. Ajusta las opciones del runtime según sea necesario:
   - `/acp model <provider/model>`
   - `/acp permissions <profile>`
   - `/acp timeout <seconds>`
5. Redirige una sesión activa sin reemplazar el contexto:
   - `/acp steer tighten logging and continue`
6. Detén el trabajo:
   - `/acp cancel` (detiene el turno actual), o
   - `/acp close` (cierra la sesión y elimina los vínculos)

## Inicio rápido para personas

Ejemplos de solicitudes naturales:

- "Vincula este canal de Discord a Codex."
- "Inicia una sesión persistente de Codex en un hilo aquí y mantenla enfocada."
- "Ejecuta esto como una sesión ACP de Claude Code de una sola vez y resume el resultado."
- "Vincula este chat de iMessage a Codex y mantén los seguimientos en el mismo espacio de trabajo."
- "Usa Gemini CLI para esta tarea en un hilo y luego mantén los seguimientos en ese mismo hilo."

Lo que OpenClaw debe hacer:

1. Elegir `runtime: "acp"`.
2. Resolver el objetivo del harness solicitado (`agentId`, por ejemplo `codex`).
3. Si se solicita una vinculación a la conversación actual y el canal activo la admite, vincular la sesión ACP a esa conversación.
4. En caso contrario, si se solicita una vinculación a un hilo y el canal actual la admite, vincular la sesión ACP al hilo.
5. Enrutar los mensajes de seguimiento vinculados a esa misma sesión ACP hasta que se desenfoque, se cierre o expire.

## ACP frente a subagentes

Usa ACP cuando quieras un runtime de harness externo. Usa subagentes cuando quieras ejecuciones delegadas nativas de OpenClaw.

| Área          | Sesión ACP                           | Ejecución de subagente                      |
| ------------- | ------------------------------------- | ---------------------------------- |
| Runtime       | Plugin de backend ACP (por ejemplo acpx) | Runtime nativo de subagentes de OpenClaw  |
| Clave de sesión   | `agent:<agentId>:acp:<uuid>`          | `agent:<agentId>:subagent:<uuid>`  |
| Comandos principales | `/acp ...`                            | `/subagents ...`                   |
| Herramienta de creación    | `sessions_spawn` con `runtime:"acp"` | `sessions_spawn` (runtime predeterminado) |

Consulta también [Subagentes](/tools/subagents).

## Cómo ACP ejecuta Claude Code

Para Claude Code mediante ACP, la pila es:

1. Plano de control de sesiones ACP de OpenClaw
2. plugin de runtime `acpx` integrado
3. Adaptador ACP de Claude
4. Maquinaria de runtime/sesión del lado de Claude

Distinción importante:

- Claude sobre ACP no es lo mismo que el runtime de respaldo directo `claude-cli/...`.
- Claude sobre ACP es una sesión de harness con controles ACP, reanudación de sesión, seguimiento de tareas en segundo plano y vinculación opcional a conversación/hilo.
- `claude-cli/...` es un backend CLI local solo de texto. Consulta [Backends de CLI](/es/gateway/cli-backends).

Para operadores, la regla práctica es:

- si quieres `/acp spawn`, sesiones vinculables, controles de runtime o trabajo persistente del harness: usa ACP
- si quieres un respaldo local simple de texto mediante la CLI sin procesar: usa backends de CLI

## Sesiones vinculadas

### Vinculaciones a la conversación actual

Usa `/acp spawn <harness> --bind here` cuando quieras que la conversación actual se convierta en un espacio de trabajo ACP duradero sin crear un hilo secundario.

Comportamiento:

- OpenClaw sigue siendo responsable del transporte del canal, autenticación, seguridad y entrega.
- La conversación actual se fija a la clave de sesión ACP creada.
- Los mensajes de seguimiento en esa conversación se enrutan a la misma sesión ACP.
- `/new` y `/reset` restablecen la misma sesión ACP vinculada en su lugar.
- `/acp close` cierra la sesión y elimina la vinculación de la conversación actual.

Lo que esto significa en la práctica:

- `--bind here` mantiene la misma superficie de chat. En Discord, el canal actual sigue siendo el canal actual.
- `--bind here` aún puede crear una nueva sesión ACP si estás iniciando trabajo nuevo. La vinculación adjunta esa sesión a la conversación actual.
- `--bind here` no crea por sí solo un hilo secundario de Discord ni un tema de Telegram.
- El runtime ACP aún puede tener su propio directorio de trabajo (`cwd`) o espacio de trabajo administrado por el backend en disco. Ese espacio de trabajo del runtime es independiente de la superficie de chat y no implica un nuevo hilo de mensajería.
- Si creas una sesión para otro agente ACP y no pasas `--cwd`, OpenClaw hereda de forma predeterminada el espacio de trabajo del **agente de destino**, no del solicitante.
- Si falta esa ruta de espacio de trabajo heredada (`ENOENT`/`ENOTDIR`), OpenClaw recurre al `cwd` predeterminado del backend en lugar de reutilizar silenciosamente el árbol incorrecto.
- Si el espacio de trabajo heredado existe pero no se puede acceder a él (por ejemplo `EACCES`), la creación devuelve el error real de acceso en lugar de omitir `cwd`.

Modelo mental:

- superficie de chat: donde las personas siguen hablando (`canal de Discord`, `tema de Telegram`, `chat de iMessage`)
- sesión ACP: el estado duradero del runtime de Codex/Claude/Gemini al que OpenClaw enruta
- hilo/tema secundario: una superficie adicional opcional de mensajería creada solo por `--thread ...`
- espacio de trabajo del runtime: la ubicación del sistema de archivos donde se ejecuta el harness (`cwd`, checkout del repositorio, espacio de trabajo del backend)

Ejemplos:

- `/acp spawn codex --bind here`: mantener este chat, crear o adjuntar una sesión ACP de Codex y enrutarle aquí los mensajes futuros
- `/acp spawn codex --thread auto`: OpenClaw puede crear un hilo/tema secundario y vincular allí la sesión ACP
- `/acp spawn codex --bind here --cwd /workspace/repo`: misma vinculación al chat que arriba, pero Codex se ejecuta en `/workspace/repo`

Compatibilidad con vinculación a la conversación actual:

- Los canales de chat/mensajes que anuncian compatibilidad con vinculación a la conversación actual pueden usar `--bind here` mediante la ruta compartida de vinculación de conversaciones.
- Los canales con semántica personalizada de hilos/temas aún pueden proporcionar canonicalización específica del canal detrás de la misma interfaz compartida.
- `--bind here` siempre significa "vincular la conversación actual en su lugar".
- Las vinculaciones genéricas a la conversación actual usan el almacén compartido de vinculaciones de OpenClaw y sobreviven a reinicios normales del gateway.

Notas:

- `--bind here` y `--thread ...` son mutuamente excluyentes en `/acp spawn`.
- En Discord, `--bind here` vincula el canal o hilo actual en su lugar. `spawnAcpSessions` solo es necesario cuando OpenClaw necesita crear un hilo secundario para `--thread auto|here`.
- Si el canal activo no expone vinculaciones ACP a la conversación actual, OpenClaw devuelve un mensaje claro de no compatible.
- `resume` y las preguntas de "sesión nueva" son preguntas sobre la sesión ACP, no sobre el canal. Puedes reutilizar o reemplazar el estado del runtime sin cambiar la superficie de chat actual.

### Sesiones vinculadas a hilos

Cuando las vinculaciones a hilos están habilitadas para un adaptador de canal, las sesiones ACP pueden vincularse a hilos:

- OpenClaw vincula un hilo a una sesión ACP de destino.
- Los mensajes de seguimiento en ese hilo se enrutan a la sesión ACP vinculada.
- La salida de ACP se entrega de vuelta al mismo hilo.
- Desenfocar/cerrar/archivar/expiración por tiempo de inactividad o por antigüedad máxima elimina la vinculación.

La compatibilidad con vinculación a hilos depende del adaptador. Si el adaptador de canal activo no admite vinculaciones a hilos, OpenClaw devuelve un mensaje claro de no compatible/no disponible.

Flags de función obligatorios para ACP vinculado a hilos:

- `acp.enabled=true`
- `acp.dispatch.enabled` está activado de forma predeterminada (configúralo en `false` para pausar el despacho ACP)
- Flag de creación de hilos ACP del adaptador de canal habilitado (específico del adaptador)
  - Discord: `channels.discord.threadBindings.spawnAcpSessions=true`
  - Telegram: `channels.telegram.threadBindings.spawnAcpSessions=true`

### Canales compatibles con hilos

- Cualquier adaptador de canal que exponga capacidad de vinculación de sesión/hilo.
- Compatibilidad integrada actual:
  - Hilos/canales de Discord
  - Temas de Telegram (temas de foro en grupos/supergrupos y temas de DM)
- Los canales de plugin pueden añadir compatibilidad mediante la misma interfaz de vinculación.

## Ajustes específicos de canal

Para flujos no efímeros, configura vinculaciones ACP persistentes en entradas `bindings[]` de nivel superior.

### Modelo de vinculación

- `bindings[].type="acp"` marca una vinculación persistente de conversación ACP.
- `bindings[].match` identifica la conversación de destino:
  - Canal o hilo de Discord: `match.channel="discord"` + `match.peer.id="<channelOrThreadId>"`
  - Tema de foro de Telegram: `match.channel="telegram"` + `match.peer.id="<chatId>:topic:<topicId>"`
  - Chat DM/grupal de BlueBubbles: `match.channel="bluebubbles"` + `match.peer.id="<handle|chat_id:*|chat_guid:*|chat_identifier:*>"`
    Prefiere `chat_id:*` o `chat_identifier:*` para vinculaciones de grupo estables.
  - Chat DM/grupal de iMessage: `match.channel="imessage"` + `match.peer.id="<handle|chat_id:*|chat_guid:*|chat_identifier:*>"`
    Prefiere `chat_id:*` para vinculaciones de grupo estables.
- `bindings[].agentId` es el id del agente de OpenClaw propietario.
- Los reemplazos opcionales de ACP se encuentran en `bindings[].acp`:
  - `mode` (`persistent` o `oneshot`)
  - `label`
  - `cwd`
  - `backend`

### Valores predeterminados del runtime por agente

Usa `agents.list[].runtime` para definir una vez los valores predeterminados de ACP por agente:

- `agents.list[].runtime.type="acp"`
- `agents.list[].runtime.acp.agent` (id del harness, por ejemplo `codex` o `claude`)
- `agents.list[].runtime.acp.backend`
- `agents.list[].runtime.acp.mode`
- `agents.list[].runtime.acp.cwd`

Precedencia de reemplazos para sesiones ACP vinculadas:

1. `bindings[].acp.*`
2. `agents.list[].runtime.acp.*`
3. Valores predeterminados globales de ACP (por ejemplo `acp.backend`)

Ejemplo:

```json5
{
  agents: {
    list: [
      {
        id: "codex",
        runtime: {
          type: "acp",
          acp: {
            agent: "codex",
            backend: "acpx",
            mode: "persistent",
            cwd: "/workspace/openclaw",
          },
        },
      },
      {
        id: "claude",
        runtime: {
          type: "acp",
          acp: { agent: "claude", backend: "acpx", mode: "persistent" },
        },
      },
    ],
  },
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "discord",
        accountId: "default",
        peer: { kind: "channel", id: "222222222222222222" },
      },
      acp: { label: "codex-main" },
    },
    {
      type: "acp",
      agentId: "claude",
      match: {
        channel: "telegram",
        accountId: "default",
        peer: { kind: "group", id: "-1001234567890:topic:42" },
      },
      acp: { cwd: "/workspace/repo-b" },
    },
    {
      type: "route",
      agentId: "main",
      match: { channel: "discord", accountId: "default" },
    },
    {
      type: "route",
      agentId: "main",
      match: { channel: "telegram", accountId: "default" },
    },
  ],
  channels: {
    discord: {
      guilds: {
        "111111111111111111": {
          channels: {
            "222222222222222222": { requireMention: false },
          },
        },
      },
    },
    telegram: {
      groups: {
        "-1001234567890": {
          topics: { "42": { requireMention: false } },
        },
      },
    },
  },
}
```

Comportamiento:

- OpenClaw se asegura de que la sesión ACP configurada exista antes de usarla.
- Los mensajes en ese canal o tema se enrutan a la sesión ACP configurada.
- En conversaciones vinculadas, `/new` y `/reset` restablecen en su lugar la misma clave de sesión ACP.
- Las vinculaciones temporales del runtime (por ejemplo las creadas por flujos de enfoque de hilo) siguen aplicándose donde estén presentes.
- Para creaciones ACP entre agentes sin un `cwd` explícito, OpenClaw hereda el espacio de trabajo del agente de destino desde la configuración del agente.
- Las rutas heredadas de espacio de trabajo que faltan recurren al `cwd` predeterminado del backend; los fallos reales de acceso en rutas existentes se muestran como errores de creación.

## Iniciar sesiones ACP (interfaces)

### Desde `sessions_spawn`

Usa `runtime: "acp"` para iniciar una sesión ACP desde un turno de agente o una llamada de herramienta.

```json
{
  "task": "Open the repo and summarize failing tests",
  "runtime": "acp",
  "agentId": "codex",
  "thread": true,
  "mode": "session"
}
```

Notas:

- `runtime` usa por defecto `subagent`, así que configura explícitamente `runtime: "acp"` para sesiones ACP.
- Si se omite `agentId`, OpenClaw usa `acp.defaultAgent` cuando está configurado.
- `mode: "session"` requiere `thread: true` para mantener una conversación vinculada persistente.

Detalles de la interfaz:

- `task` (obligatorio): prompt inicial enviado a la sesión ACP.
- `runtime` (obligatorio para ACP): debe ser `"acp"`.
- `agentId` (opcional): id del harness ACP de destino. Usa `acp.defaultAgent` como respaldo si está configurado.
- `thread` (opcional, predeterminado `false`): solicita flujo de vinculación a hilo donde sea compatible.
- `mode` (opcional): `run` (una sola vez) o `session` (persistente).
  - el valor predeterminado es `run`
  - si `thread: true` y se omite el modo, OpenClaw puede usar un comportamiento persistente predeterminado según la ruta del runtime
  - `mode: "session"` requiere `thread: true`
- `cwd` (opcional): directorio de trabajo solicitado para el runtime (validado por la política del backend/runtime). Si se omite, la creación ACP hereda el espacio de trabajo del agente de destino cuando esté configurado; las rutas heredadas ausentes recurren a los valores predeterminados del backend, mientras que los errores reales de acceso se devuelven.
- `label` (opcional): etiqueta orientada al operador usada en el texto de sesión/banner.
- `resumeSessionId` (opcional): reanuda una sesión ACP existente en lugar de crear una nueva. El agente reproduce su historial de conversación mediante `session/load`. Requiere `runtime: "acp"`.
- `streamTo` (opcional): `"parent"` transmite resúmenes del progreso de la ejecución ACP inicial de vuelta a la sesión solicitante como eventos del sistema.
  - Cuando está disponible, las respuestas aceptadas incluyen `streamLogPath` que apunta a un registro JSONL con alcance de sesión (`<sessionId>.acp-stream.jsonl`) que puedes seguir para ver el historial completo del relay.

### Reanudar una sesión existente

Usa `resumeSessionId` para continuar una sesión ACP anterior en lugar de empezar desde cero. El agente reproduce su historial de conversación mediante `session/load`, por lo que retoma el trabajo con el contexto completo de lo anterior.

```json
{
  "task": "Continue where we left off — fix the remaining test failures",
  "runtime": "acp",
  "agentId": "codex",
  "resumeSessionId": "<previous-session-id>"
}
```

Casos de uso habituales:

- Transferir una sesión de Codex de tu portátil a tu teléfono: indica a tu agente que retome donde lo dejaste
- Continuar una sesión de programación que comenzaste de forma interactiva en la CLI, ahora de forma headless mediante tu agente
- Retomar trabajo que fue interrumpido por un reinicio del gateway o por expiración por inactividad

Notas:

- `resumeSessionId` requiere `runtime: "acp"`; devuelve un error si se usa con el runtime de subagente.
- `resumeSessionId` restaura el historial de conversación ACP upstream; `thread` y `mode` siguen aplicándose normalmente a la nueva sesión de OpenClaw que estás creando, así que `mode: "session"` sigue requiriendo `thread: true`.
- El agente de destino debe admitir `session/load` (Codex y Claude Code lo hacen).
- Si no se encuentra el ID de sesión, la creación falla con un error claro; no hay respaldo silencioso a una sesión nueva.

### Prueba de humo para operadores

Usa esto después de desplegar un gateway cuando quieras una comprobación rápida en vivo de que la creación ACP
realmente funciona de extremo a extremo, no solo que pasa pruebas unitarias.

Puerta recomendada:

1. Verifica la versión/commit del gateway desplegado en el host de destino.
2. Confirma que el código fuente desplegado incluye la aceptación de linaje ACP en
   `src/gateway/sessions-patch.ts` (`subagent:* or acp:* sessions`).
3. Abre una sesión temporal de puente ACPX hacia un agente en vivo (por ejemplo
   `razor(main)` en `jpclawhq`).
4. Pide a ese agente que llame a `sessions_spawn` con:
   - `runtime: "acp"`
   - `agentId: "codex"`
   - `mode: "run"`
   - tarea: `Reply with exactly LIVE-ACP-SPAWN-OK`
5. Verifica que el agente informe:
   - `accepted=yes`
   - una `childSessionKey` real
   - ningún error de validador
6. Limpia la sesión temporal de puente ACPX.

Prompt de ejemplo para el agente en vivo:

```text
Use the sessions_spawn tool now with runtime: "acp", agentId: "codex", and mode: "run".
Set the task to: "Reply with exactly LIVE-ACP-SPAWN-OK".
Then report only: accepted=<yes/no>; childSessionKey=<value or none>; error=<exact text or none>.
```

Notas:

- Mantén esta prueba de humo en `mode: "run"` a menos que estés probando
  intencionalmente sesiones ACP persistentes vinculadas a hilos.
- No exijas `streamTo: "parent"` para la puerta básica. Esa ruta depende de
  las capacidades del solicitante/la sesión y es una comprobación de integración aparte.
- Trata las pruebas vinculadas a hilos con `mode: "session"` como una segunda
  pasada de integración más completa desde un hilo real de Discord o un tema de Telegram.

## Compatibilidad con sandbox

Actualmente, las sesiones ACP se ejecutan en el runtime del host, no dentro del sandbox de OpenClaw.

Limitaciones actuales:

- Si la sesión solicitante está en sandbox, las creaciones ACP se bloquean tanto para `sessions_spawn({ runtime: "acp" })` como para `/acp spawn`.
  - Error: `Sandboxed sessions cannot spawn ACP sessions because runtime="acp" runs on the host. Use runtime="subagent" from sandboxed sessions.`
- `sessions_spawn` con `runtime: "acp"` no admite `sandbox: "require"`.
  - Error: `sessions_spawn sandbox="require" is unsupported for runtime="acp" because ACP sessions run outside the sandbox. Use runtime="subagent" or sandbox="inherit".`

Usa `runtime: "subagent"` cuando necesites ejecución forzada por sandbox.

### Desde el comando `/acp`

Usa `/acp spawn` para un control explícito del operador desde el chat cuando sea necesario.

```text
/acp spawn codex --mode persistent --thread auto
/acp spawn codex --mode oneshot --thread off
/acp spawn codex --bind here
/acp spawn codex --thread here
```

Flags principales:

- `--mode persistent|oneshot`
- `--bind here|off`
- `--thread auto|here|off`
- `--cwd <absolute-path>`
- `--label <name>`

Consulta [Comandos slash](/tools/slash-commands).

## Resolución de destino de sesión

La mayoría de las acciones `/acp` aceptan un destino de sesión opcional (`session-key`, `session-id` o `session-label`).

Orden de resolución:

1. Argumento de destino explícito (o `--session` para `/acp steer`)
   - primero intenta clave
   - luego id de sesión con forma de UUID
   - luego etiqueta
2. Vinculación del hilo actual (si esta conversación/hilo está vinculado a una sesión ACP)
3. Respaldo a la sesión solicitante actual

Las vinculaciones a la conversación actual y a hilos participan ambas en el paso 2.

Si no se resuelve ningún destino, OpenClaw devuelve un error claro (`Unable to resolve session target: ...`).

## Modos de vinculación al crear

`/acp spawn` admite `--bind here|off`.

| Modo   | Comportamiento                                                               |
| ------ | ---------------------------------------------------------------------- |
| `here` | Vincula en su lugar la conversación activa actual; falla si no hay ninguna activa. |
| `off`  | No crea una vinculación a la conversación actual.                          |

Notas:

- `--bind here` es la ruta más simple para operadores para "hacer que este canal o chat use Codex".
- `--bind here` no crea un hilo secundario.
- `--bind here` solo está disponible en canales que exponen compatibilidad con vinculación a la conversación actual.
- `--bind` y `--thread` no pueden combinarse en la misma llamada a `/acp spawn`.

## Modos de hilo al crear

`/acp spawn` admite `--thread auto|here|off`.

| Modo   | Comportamiento                                                                                            |
| ------ | --------------------------------------------------------------------------------------------------- |
| `auto` | En un hilo activo: vincula ese hilo. Fuera de un hilo: crea/vincula un hilo secundario cuando sea compatible. |
| `here` | Requiere un hilo activo actual; falla si no estás dentro de uno.                                                  |
| `off`  | Sin vinculación. La sesión se inicia sin vincular.                                                                 |

Notas:

- En superficies sin vinculación a hilos, el comportamiento predeterminado es efectivamente `off`.
- La creación vinculada a hilos requiere compatibilidad de la política del canal:
  - Discord: `channels.discord.threadBindings.spawnAcpSessions=true`
  - Telegram: `channels.telegram.threadBindings.spawnAcpSessions=true`
- Usa `--bind here` cuando quieras fijar la conversación actual sin crear un hilo secundario.

## Controles ACP

Familia de comandos disponible:

- `/acp spawn`
- `/acp cancel`
- `/acp steer`
- `/acp close`
- `/acp status`
- `/acp set-mode`
- `/acp set`
- `/acp cwd`
- `/acp permissions`
- `/acp timeout`
- `/acp model`
- `/acp reset-options`
- `/acp sessions`
- `/acp doctor`
- `/acp install`

`/acp status` muestra las opciones efectivas del runtime y, cuando están disponibles, los identificadores de sesión tanto a nivel de runtime como de backend.

Algunos controles dependen de las capacidades del backend. Si un backend no admite un control, OpenClaw devuelve un error claro de control no compatible.

## Recetario de comandos ACP

| Command              | What it does                                              | Example                                                       |
| -------------------- | --------------------------------------------------------- | ------------------------------------------------------------- |
| `/acp spawn`         | Crea una sesión ACP; vinculación opcional a conversación actual o a hilo. | `/acp spawn codex --bind here --cwd /repo`                    |
| `/acp cancel`        | Cancela el turno en curso para la sesión de destino.                 | `/acp cancel agent:codex:acp:<uuid>`                          |
| `/acp steer`         | Envía una instrucción de dirección a la sesión en ejecución.                | `/acp steer --session support inbox prioritize failing tests` |
| `/acp close`         | Cierra la sesión y desvincula los destinos de hilo.                  | `/acp close`                                                  |
| `/acp status`        | Muestra backend, modo, estado, opciones de runtime y capacidades. | `/acp status`                                                 |
| `/acp set-mode`      | Configura el modo de runtime para la sesión de destino.                      | `/acp set-mode plan`                                          |
| `/acp set`           | Escritura genérica de opción de configuración del runtime.                      | `/acp set model openai/gpt-5.4`                               |
| `/acp cwd`           | Configura el reemplazo del directorio de trabajo del runtime.                   | `/acp cwd /Users/user/Projects/repo`                          |
| `/acp permissions`   | Configura el perfil de política de aprobación.                              | `/acp permissions strict`                                     |
| `/acp timeout`       | Configura el tiempo de espera del runtime (segundos).                            | `/acp timeout 120`                                            |
| `/acp model`         | Configura el reemplazo del modelo del runtime.                               | `/acp model anthropic/claude-opus-4-6`                        |
| `/acp reset-options` | Elimina los reemplazos de opciones de runtime de la sesión.                  | `/acp reset-options`                                          |
| `/acp sessions`      | Enumera las sesiones ACP recientes desde el almacén.                      | `/acp sessions`                                               |
| `/acp doctor`        | Estado del backend, capacidades y correcciones accionables.           | `/acp doctor`                                                 |
| `/acp install`       | Imprime pasos deterministas de instalación y habilitación.             | `/acp install`                                                |

`/acp sessions` lee el almacén para la sesión actual vinculada o solicitante. Los comandos que aceptan tokens `session-key`, `session-id` o `session-label` resuelven destinos mediante el descubrimiento de sesiones del gateway, incluidos los roots personalizados de `session.store` por agente.

## Mapeo de opciones de runtime

`/acp` tiene comandos de conveniencia y un configurador genérico.

Operaciones equivalentes:

- `/acp model <id>` se asigna a la clave de configuración de runtime `model`.
- `/acp permissions <profile>` se asigna a la clave de configuración de runtime `approval_policy`.
- `/acp timeout <seconds>` se asigna a la clave de configuración de runtime `timeout`.
- `/acp cwd <path>` actualiza directamente el reemplazo de `cwd` del runtime.
- `/acp set <key> <value>` es la ruta genérica.
  - Caso especial: `key=cwd` usa la ruta de reemplazo de `cwd`.
- `/acp reset-options` borra todos los reemplazos de runtime para la sesión de destino.

## Compatibilidad actual de harnesses acpx

Alias de harness integrados actuales de acpx:

- `claude`
- `codex`
- `copilot`
- `cursor` (Cursor CLI: `cursor-agent acp`)
- `droid`
- `gemini`
- `iflow`
- `kilocode`
- `kimi`
- `kiro`
- `openclaw`
- `opencode`
- `pi`
- `qwen`

Cuando OpenClaw usa el backend acpx, prefiere estos valores para `agentId` a menos que tu configuración de acpx defina alias personalizados de agente.
Si tu instalación local de Cursor todavía expone ACP como `agent acp`, reemplaza el comando del agente `cursor` en tu configuración de acpx en lugar de cambiar el valor predeterminado integrado.

El uso directo de la CLI de acpx también puede apuntar a adaptadores arbitrarios mediante `--agent <command>`, pero esa vía de escape sin procesar es una función de la CLI de acpx (no la ruta normal de `agentId` de OpenClaw).

## Configuración obligatoria

Base central de ACP:

```json5
{
  acp: {
    enabled: true,
    // Opcional. El valor predeterminado es true; configúralo en false para pausar el despacho ACP manteniendo los controles /acp.
    dispatch: { enabled: true },
    backend: "acpx",
    defaultAgent: "codex",
    allowedAgents: [
      "claude",
      "codex",
      "copilot",
      "cursor",
      "droid",
      "gemini",
      "iflow",
      "kilocode",
      "kimi",
      "kiro",
      "openclaw",
      "opencode",
      "pi",
      "qwen",
    ],
    maxConcurrentSessions: 8,
    stream: {
      coalesceIdleMs: 300,
      maxChunkChars: 1200,
    },
    runtime: {
      ttlMinutes: 120,
    },
  },
}
```

La configuración de vinculación a hilos es específica del adaptador de canal. Ejemplo para Discord:

```json5
{
  session: {
    threadBindings: {
      enabled: true,
      idleHours: 24,
      maxAgeHours: 0,
    },
  },
  channels: {
    discord: {
      threadBindings: {
        enabled: true,
        spawnAcpSessions: true,
      },
    },
  },
}
```

Si la creación ACP vinculada a hilos no funciona, primero verifica el flag de función del adaptador:

- Discord: `channels.discord.threadBindings.spawnAcpSessions=true`

Las vinculaciones a la conversación actual no requieren crear un hilo secundario. Requieren un contexto de conversación activo y un adaptador de canal que exponga vinculaciones de conversación ACP.

Consulta la [Referencia de configuración](/es/gateway/configuration-reference).

## Configuración del plugin para el backend acpx

Las instalaciones nuevas incluyen el plugin de runtime `acpx` integrado habilitado de forma predeterminada, así que ACP
normalmente funciona sin un paso manual de instalación del plugin.

Empieza con:

```text
/acp doctor
```

Si deshabilitaste `acpx`, lo denegaste mediante `plugins.allow` / `plugins.deny`, o quieres
cambiar a un checkout local de desarrollo, usa la ruta explícita del plugin:

```bash
openclaw plugins install acpx
openclaw config set plugins.entries.acpx.enabled true
```

Instalación desde un espacio de trabajo local durante el desarrollo:

```bash
openclaw plugins install ./path/to/local/acpx-plugin
```

Luego verifica el estado del backend:

```text
/acp doctor
```

### Configuración de comando y versión de acpx

De forma predeterminada, el plugin de backend acpx integrado (`acpx`) usa el binario fijado localmente en el plugin:

1. El comando usa por defecto `node_modules/.bin/acpx` local del plugin dentro del paquete del plugin ACPX.
2. La versión esperada usa por defecto el pin de la extensión.
3. El inicio registra inmediatamente el backend ACP como no listo.
4. Un trabajo de ensure en segundo plano verifica `acpx --version`.
5. Si el binario local del plugin falta o no coincide, ejecuta:
   `npm install --omit=dev --no-save acpx@<pinned>` y vuelve a verificar.

Puedes reemplazar comando/versión en la configuración del plugin:

```json
{
  "plugins": {
    "entries": {
      "acpx": {
        "enabled": true,
        "config": {
          "command": "../acpx/dist/cli.js",
          "expectedVersion": "any"
        }
      }
    }
  }
}
```

Notas:

- `command` acepta una ruta absoluta, una ruta relativa o un nombre de comando (`acpx`).
- Las rutas relativas se resuelven desde el directorio del espacio de trabajo de OpenClaw.
- `expectedVersion: "any"` desactiva la comprobación estricta de versión.
- Cuando `command` apunta a un binario/ruta personalizado, se desactiva la instalación automática local del plugin.
- El inicio de OpenClaw sigue sin bloquear mientras se ejecuta la comprobación de estado del backend.

Consulta [Plugins](/tools/plugin).

### Instalación automática de dependencias

Cuando instalas OpenClaw globalmente con `npm install -g openclaw`, las
dependencias de runtime de acpx (binarios específicos de la plataforma) se instalan automáticamente
mediante un hook de postinstall. Si la instalación automática falla, el gateway sigue iniciándose
con normalidad e informa de la dependencia faltante mediante `openclaw acp doctor`.

### Puente MCP de herramientas de plugins

De forma predeterminada, las sesiones ACPX **no** exponen al
harness ACP las herramientas registradas por plugins de OpenClaw.

Si quieres que agentes ACP como Codex o Claude Code llamen a herramientas de plugins de OpenClaw instalados
como memory recall/store, habilita el puente dedicado:

```bash
openclaw config set plugins.entries.acpx.config.pluginToolsMcpBridge true
```

Lo que esto hace:

- Inyecta un servidor MCP integrado llamado `openclaw-plugin-tools` en el bootstrap
  de la sesión ACPX.
- Expone herramientas de plugins ya registradas por plugins de OpenClaw instalados y habilitados.
- Mantiene la función explícita y desactivada de forma predeterminada.

Notas de seguridad y confianza:

- Esto amplía la superficie de herramientas del harness ACP.
- Los agentes ACP obtienen acceso solo a las herramientas de plugins ya activas en el gateway.
- Trátalo como el mismo límite de confianza que permitir que esos plugins se ejecuten
  en el propio OpenClaw.
- Revisa los plugins instalados antes de habilitarlo.

Los `mcpServers` personalizados siguen funcionando como antes. El puente integrado de herramientas de plugins es
una comodidad adicional opcional, no un reemplazo de la configuración genérica de servidores MCP.

## Configuración de permisos

Las sesiones ACP se ejecutan de forma no interactiva; no hay TTY para aprobar o denegar solicitudes de permisos de escritura de archivos y ejecución de shell. El plugin acpx proporciona dos claves de configuración que controlan cómo se gestionan los permisos:

Estos permisos de harness ACPX son independientes de las aprobaciones de ejecución de OpenClaw y de flags de omisión específicos del proveedor en backends CLI, como `--permission-mode bypassPermissions` de Claude CLI. ACPX `approve-all` es el interruptor de emergencia a nivel de harness para sesiones ACP.

### `permissionMode`

Controla qué operaciones puede realizar el agente de harness sin solicitar confirmación.

| Value           | Behavior                                                  |
| --------------- | --------------------------------------------------------- |
| `approve-all`   | Aprueba automáticamente todas las escrituras de archivos y comandos de shell.          |
| `approve-reads` | Aprueba automáticamente solo lecturas; las escrituras y ejecuciones requieren confirmación. |
| `deny-all`      | Deniega todas las solicitudes de permisos.                              |

### `nonInteractivePermissions`

Controla qué ocurre cuando se mostraría una solicitud de permisos pero no hay un TTY interactivo disponible (que siempre es el caso en las sesiones ACP).

| Value  | Behavior                                                          |
| ------ | ----------------------------------------------------------------- |
| `fail` | Aborta la sesión con `AcpRuntimeError`. **(predeterminado)**           |
| `deny` | Deniega silenciosamente el permiso y continúa (degradación controlada). |

### Configuración

Configura mediante la configuración del plugin:

```bash
openclaw config set plugins.entries.acpx.config.permissionMode approve-all
openclaw config set plugins.entries.acpx.config.nonInteractivePermissions fail
```

Reinicia el gateway después de cambiar estos valores.

> **Importante:** Actualmente, OpenClaw usa por defecto `permissionMode=approve-reads` y `nonInteractivePermissions=fail`. En sesiones ACP no interactivas, cualquier escritura o ejecución que active una solicitud de permisos puede fallar con `AcpRuntimeError: Permission prompt unavailable in non-interactive mode`.
>
> Si necesitas restringir permisos, configura `nonInteractivePermissions` en `deny` para que las sesiones se degraden de forma controlada en lugar de fallar.

## Solución de problemas

| Symptom                                                                     | Likely cause                                                                    | Fix                                                                                                                                                               |
| --------------------------------------------------------------------------- | ------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ACP runtime backend is not configured`                                     | Falta el plugin de backend o está deshabilitado.                                             | Instala y habilita el plugin de backend, y luego ejecuta `/acp doctor`.                                                                                                        |
| `ACP is disabled by policy (acp.enabled=false)`                             | ACP está deshabilitado globalmente.                                                          | Configura `acp.enabled=true`.                                                                                                                                           |
| `ACP dispatch is disabled by policy (acp.dispatch.enabled=false)`           | El despacho desde mensajes normales del hilo está deshabilitado.                                  | Configura `acp.dispatch.enabled=true`.                                                                                                                                  |
| `ACP agent "<id>" is not allowed by policy`                                 | El agente no está en la lista de permitidos.                                                         | Usa un `agentId` permitido o actualiza `acp.allowedAgents`.                                                                                                              |
| `Unable to resolve session target: ...`                                     | Token de clave/id/etiqueta incorrecto.                                                         | Ejecuta `/acp sessions`, copia la clave/etiqueta exacta y vuelve a intentarlo.                                                                                                                 |
| `--bind here requires running /acp spawn inside an active ... conversation` | `--bind here` se usó sin una conversación vinculable activa.                     | Ve al chat/canal de destino y vuelve a intentarlo, o usa una creación sin vinculación.                                                                                                  |
| `Conversation bindings are unavailable for <channel>.`                      | Al adaptador le falta capacidad de vinculación ACP a la conversación actual.                      | Usa `/acp spawn ... --thread ...` cuando sea compatible, configura `bindings[]` de nivel superior, o cambia a un canal compatible.                                              |
| `--thread here requires running /acp spawn inside an active ... thread`     | `--thread here` se usó fuera del contexto de un hilo.                                  | Ve al hilo de destino o usa `--thread auto`/`off`.                                                                                                               |
| `Only <user-id> can rebind this channel/conversation/thread.`               | Otro usuario es propietario del destino de vinculación activo.                                    | Vuelve a vincular como propietario o usa otra conversación o hilo.                                                                                                        |
| `Thread bindings are unavailable for <channel>.`                            | Al adaptador le falta capacidad de vinculación a hilos.                                        | Usa `--thread off` o cambia a un adaptador/canal compatible.                                                                                                          |
| `Sandboxed sessions cannot spawn ACP sessions ...`                          | El runtime ACP se ejecuta en el host; la sesión solicitante está en sandbox.                       | Usa `runtime="subagent"` desde sesiones en sandbox, o ejecuta la creación ACP desde una sesión sin sandbox.                                                                  |
| `sessions_spawn sandbox="require" is unsupported for runtime="acp" ...`     | Se solicitó `sandbox="require"` para el runtime ACP.                                  | Usa `runtime="subagent"` para sandbox obligatorio, o usa ACP con `sandbox="inherit"` desde una sesión sin sandbox.                                               |
| Missing ACP metadata for bound session                                      | Metadatos ACP obsoletos/eliminados para la sesión vinculada.                                             | Vuelve a crear con `/acp spawn` y luego vuelve a vincular/enfocar el hilo.                                                                                                             |
| `AcpRuntimeError: Permission prompt unavailable in non-interactive mode`    | `permissionMode` bloquea escrituras/ejecuciones en la sesión ACP no interactiva.             | Configura `plugins.entries.acpx.config.permissionMode` en `approve-all` y reinicia el gateway. Consulta [Configuración de permisos](#permission-configuration).                 |
| La sesión ACP falla temprano con poca salida                                  | Las solicitudes de permisos están bloqueadas por `permissionMode`/`nonInteractivePermissions`. | Comprueba los logs del gateway para `AcpRuntimeError`. Para permisos completos, configura `permissionMode=approve-all`; para degradación controlada, configura `nonInteractivePermissions=deny`. |
| La sesión ACP se queda bloqueada indefinidamente después de completar el trabajo                       | El proceso del harness terminó pero la sesión ACP no informó la finalización.             | Supervísalo con `ps aux \| grep acpx`; mata manualmente los procesos obsoletos.                                                                                                |
