---
read_when:
    - Ejecutar harnesses de programación mediante ACP
    - Configurar sesiones ACP vinculadas a conversaciones en canales de mensajería
    - Vincular una conversación de un canal de mensajes a una sesión ACP persistente
    - Solucionar problemas del backend ACP y del cableado de plugins
    - Operar comandos `/acp` desde el chat
summary: Usa sesiones de runtime ACP para Codex, Claude Code, Cursor, Gemini CLI, OpenClaw ACP y otros agentes de harness
title: Agentes ACP
x-i18n:
    generated_at: "2026-04-08T02:19:39Z"
    model: gpt-5.4
    provider: openai
    source_hash: 71c7c0cdae5247aefef17a0029360950a1c2987ddcee21a1bb7d78c67da52950
    source_path: tools/acp-agents.md
    workflow: 15
---

# Agentes ACP

Las sesiones de [Agent Client Protocol (ACP)](https://agentclientprotocol.com/) permiten que OpenClaw ejecute harnesses externos de programación (por ejemplo Pi, Claude Code, Codex, Cursor, Copilot, OpenClaw ACP, OpenCode, Gemini CLI y otros harnesses ACPX compatibles) mediante un plugin backend de ACP.

Si le pides a OpenClaw en lenguaje natural “ejecuta esto en Codex” o “inicia Claude Code en un hilo”, OpenClaw debería enrutar esa solicitud al runtime ACP (no al runtime nativo de subagentes). Cada creación de sesión ACP se rastrea como una [tarea en segundo plano](/es/automation/tasks).

Si quieres que Codex o Claude Code se conecten directamente como cliente MCP externo
a conversaciones de canal existentes de OpenClaw, usa [`openclaw mcp serve`](/cli/mcp)
en lugar de ACP.

## ¿Qué página quiero?

Hay tres superficies cercanas que es fácil confundir:

| Quieres...                                                                     | Usa esto                              | Notas                                                                                                       |
| ---------------------------------------------------------------------------------- | ------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| Ejecutar Codex, Claude Code, Gemini CLI u otro harness externo _a través de_ OpenClaw | Esta página: agentes ACP                 | Sesiones vinculadas al chat, `/acp spawn`, `sessions_spawn({ runtime: "acp" })`, tareas en segundo plano, controles de runtime |
| Exponer una sesión de OpenClaw Gateway _como_ servidor ACP para un editor o cliente      | [`openclaw acp`](/cli/acp)            | Modo puente. El IDE/cliente habla ACP con OpenClaw por stdio/WebSocket                                          |
| Reutilizar una CLI local de IA como modelo alternativo solo de texto                                 | [Backends CLI](/es/gateway/cli-backends) | No es ACP. Sin herramientas de OpenClaw, sin controles ACP, sin runtime de harness                                             |

## ¿Funciona de inmediato?

Normalmente, sí.

- Las instalaciones nuevas ahora incluyen el plugin de runtime empaquetado `acpx` habilitado de forma predeterminada.
- El plugin empaquetado `acpx` prefiere su binario `acpx` fijado localmente en el plugin.
- Al iniciarse, OpenClaw sondea ese binario y lo autorrepara si hace falta.
- Empieza con `/acp doctor` si quieres una comprobación rápida de preparación.

Lo que aún puede ocurrir en el primer uso:

- Es posible que un adaptador del harness objetivo se descargue bajo demanda con `npx` la primera vez que uses ese harness.
- La autenticación del proveedor aún tiene que existir en el host para ese harness.
- Si el host no tiene acceso a npm/red, las descargas iniciales del adaptador pueden fallar hasta que las cachés se precalienten o el adaptador se instale de otro modo.

Ejemplos:

- `/acp spawn codex`: OpenClaw debería estar listo para iniciar `acpx`, pero el adaptador ACP de Codex puede seguir necesitando una descarga inicial.
- `/acp spawn claude`: lo mismo con el adaptador ACP de Claude, además de la autenticación del lado de Claude en ese host.

## Flujo rápido para operadores

Úsalo cuando quieras una guía práctica de `/acp`:

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
5. Da una indicación a una sesión activa sin reemplazar el contexto:
   - `/acp steer tighten logging and continue`
6. Detén el trabajo:
   - `/acp cancel` (detener el turno actual), o
   - `/acp close` (cerrar sesión + eliminar vínculos)

## Inicio rápido para personas

Ejemplos de solicitudes naturales:

- “Vincula este canal de Discord a Codex.”
- “Inicia una sesión persistente de Codex en un hilo aquí y mantenla enfocada.”
- “Ejecuta esto como una sesión ACP puntual de Claude Code y resume el resultado.”
- “Vincula este chat de iMessage a Codex y mantén los seguimientos en el mismo espacio de trabajo.”
- “Usa Gemini CLI para esta tarea en un hilo y luego mantén los seguimientos en ese mismo hilo.”

Lo que OpenClaw debería hacer:

1. Elegir `runtime: "acp"`.
2. Resolver el harness objetivo solicitado (`agentId`, por ejemplo `codex`).
3. Si se solicita vinculación con la conversación actual y el canal activo la admite, vincular la sesión ACP a esa conversación.
4. En caso contrario, si se solicita vinculación con hilo y el canal actual la admite, vincular la sesión ACP al hilo.
5. Enrutar los mensajes de seguimiento vinculados a esa misma sesión ACP hasta que se desenfoque/cierre/caduca.

## ACP frente a subagentes

Usa ACP cuando quieras un runtime de harness externo. Usa subagentes cuando quieras ejecuciones delegadas nativas de OpenClaw.

| Área          | Sesión ACP                           | Ejecución de subagente                      |
| ------------- | ------------------------------------- | ---------------------------------- |
| Runtime       | Plugin backend ACP (por ejemplo acpx) | Runtime nativo de subagente de OpenClaw  |
| Clave de sesión   | `agent:<agentId>:acp:<uuid>`          | `agent:<agentId>:subagent:<uuid>`  |
| Comandos principales | `/acp ...`                            | `/subagents ...`                   |
| Herramienta de creación    | `sessions_spawn` con `runtime:"acp"` | `sessions_spawn` (runtime predeterminado) |

Consulta también [Subagentes](/es/tools/subagents).

## Cómo ejecuta ACP Claude Code

Para Claude Code mediante ACP, la pila es:

1. Plano de control de sesiones ACP de OpenClaw
2. plugin de runtime empaquetado `acpx`
3. adaptador ACP de Claude
4. maquinaria de runtime/sesión del lado de Claude

Distinción importante:

- Claude vía ACP es una sesión de harness con controles ACP, reanudación de sesión, seguimiento de tareas en segundo plano y vinculación opcional a conversación/hilo.
- Los backends CLI son runtimes alternativos locales y separados, solo de texto. Consulta [Backends CLI](/es/gateway/cli-backends).

Para operadores, la regla práctica es:

- si quieres `/acp spawn`, sesiones vinculables, controles de runtime o trabajo persistente del harness: usa ACP
- si quieres una alternativa local sencilla de texto mediante la CLI sin procesar: usa backends CLI

## Sesiones vinculadas

### Vinculaciones con la conversación actual

Usa `/acp spawn <harness> --bind here` cuando quieras que la conversación actual se convierta en un espacio de trabajo ACP duradero sin crear un hilo secundario.

Comportamiento:

- OpenClaw sigue gestionando el transporte del canal, la autenticación, la seguridad y la entrega.
- La conversación actual queda fijada a la clave de sesión ACP creada.
- Los mensajes de seguimiento en esa conversación se enrutan a la misma sesión ACP.
- `/new` y `/reset` reinician la misma sesión ACP vinculada en el mismo lugar.
- `/acp close` cierra la sesión y elimina la vinculación de la conversación actual.

Qué significa esto en la práctica:

- `--bind here` mantiene la misma superficie de chat. En Discord, el canal actual sigue siendo el actual.
- `--bind here` puede seguir creando una sesión ACP nueva si estás iniciando trabajo desde cero. La vinculación adjunta esa sesión a la conversación actual.
- `--bind here` no crea por sí sola un hilo secundario de Discord ni un tema de Telegram.
- El runtime ACP puede seguir teniendo su propio directorio de trabajo (`cwd`) o espacio de trabajo administrado por backend en disco. Ese espacio de trabajo del runtime está separado de la superficie de chat y no implica un nuevo hilo de mensajería.
- Si creas una sesión para un agente ACP diferente y no pasas `--cwd`, OpenClaw hereda por defecto el espacio de trabajo del **agente de destino**, no el del solicitante.
- Si esa ruta heredada del espacio de trabajo no existe (`ENOENT`/`ENOTDIR`), OpenClaw recurre al `cwd` predeterminado del backend en lugar de reutilizar silenciosamente el árbol equivocado.
- Si el espacio de trabajo heredado existe pero no se puede acceder a él (por ejemplo `EACCES`), la creación devuelve el error real de acceso en lugar de omitir `cwd`.

Modelo mental:

- superficie de chat: donde la gente sigue hablando (`canal de Discord`, `tema de Telegram`, `chat de iMessage`)
- sesión ACP: el estado duradero del runtime de Codex/Claude/Gemini al que OpenClaw enruta
- hilo/tema secundario: una superficie adicional opcional de mensajería creada solo por `--thread ...`
- espacio de trabajo del runtime: la ubicación del sistema de archivos donde se ejecuta el harness (`cwd`, checkout del repositorio, espacio de trabajo del backend)

Ejemplos:

- `/acp spawn codex --bind here`: mantener este chat, crear o adjuntar una sesión ACP de Codex y enrutarle aquí los mensajes futuros
- `/acp spawn codex --thread auto`: OpenClaw puede crear un hilo/tema secundario y vincular allí la sesión ACP
- `/acp spawn codex --bind here --cwd /workspace/repo`: misma vinculación de chat que arriba, pero Codex se ejecuta en `/workspace/repo`

Compatibilidad de vinculación con la conversación actual:

- Los canales de chat/mensajería que anuncian compatibilidad de vinculación con la conversación actual pueden usar `--bind here` mediante la ruta compartida de vinculación de conversaciones.
- Los canales con semánticas personalizadas de hilo/tema pueden seguir proporcionando canonicalización específica del canal detrás de la misma interfaz compartida.
- `--bind here` siempre significa “vincular la conversación actual en su lugar”.
- Las vinculaciones genéricas con la conversación actual usan el almacén compartido de vinculaciones de OpenClaw y sobreviven a los reinicios normales del gateway.

Notas:

- `--bind here` y `--thread ...` son mutuamente excluyentes en `/acp spawn`.
- En Discord, `--bind here` vincula el canal o hilo actual en su lugar. `spawnAcpSessions` solo es necesario cuando OpenClaw tiene que crear un hilo secundario para `--thread auto|here`.
- Si el canal activo no expone vinculaciones ACP con la conversación actual, OpenClaw devuelve un mensaje claro de no compatibilidad.
- `resume` y las preguntas de “sesión nueva” son cuestiones de sesión ACP, no del canal. Puedes reutilizar o reemplazar el estado del runtime sin cambiar la superficie de chat actual.

### Sesiones vinculadas a hilos

Cuando las vinculaciones de hilos están habilitadas para un adaptador de canal, las sesiones ACP pueden vincularse a hilos:

- OpenClaw vincula un hilo a una sesión ACP objetivo.
- Los mensajes de seguimiento en ese hilo se enrutan a la sesión ACP vinculada.
- La salida de ACP se entrega de vuelta al mismo hilo.
- Desenfocar/cerrar/archivar/caducidad por inactividad o caducidad por edad máxima elimina la vinculación.

La compatibilidad con la vinculación de hilos es específica del adaptador. Si el adaptador del canal activo no la admite, OpenClaw devuelve un mensaje claro de no compatibilidad o no disponibilidad.

Indicadores de función requeridos para ACP vinculado a hilos:

- `acp.enabled=true`
- `acp.dispatch.enabled` está activado de forma predeterminada (ponlo en `false` para pausar el despacho ACP)
- Indicador de creación ACP de hilo del adaptador de canal habilitado (específico del adaptador)
  - Discord: `channels.discord.threadBindings.spawnAcpSessions=true`
  - Telegram: `channels.telegram.threadBindings.spawnAcpSessions=true`

### Canales compatibles con hilos

- Cualquier adaptador de canal que exponga capacidad de vinculación de sesión/hilo.
- Compatibilidad integrada actual:
  - hilos/canales de Discord
  - temas de Telegram (temas de foro en grupos/supergrupos y temas de DM)
- Los canales de plugins pueden añadir compatibilidad mediante la misma interfaz de vinculación.

## Configuración específica por canal

Para flujos no efímeros, configura vinculaciones ACP persistentes en entradas de nivel superior `bindings[]`.

### Modelo de vinculación

- `bindings[].type="acp"` marca una vinculación persistente de conversación ACP.
- `bindings[].match` identifica la conversación objetivo:
  - canal o hilo de Discord: `match.channel="discord"` + `match.peer.id="<channelOrThreadId>"`
  - tema de foro de Telegram: `match.channel="telegram"` + `match.peer.id="<chatId>:topic:<topicId>"`
  - chat DM/de grupo de BlueBubbles: `match.channel="bluebubbles"` + `match.peer.id="<handle|chat_id:*|chat_guid:*|chat_identifier:*>"`
    Prefiere `chat_id:*` o `chat_identifier:*` para vinculaciones estables de grupos.
  - chat DM/de grupo de iMessage: `match.channel="imessage"` + `match.peer.id="<handle|chat_id:*|chat_guid:*|chat_identifier:*>"`
    Prefiere `chat_id:*` para vinculaciones estables de grupos.
- `bindings[].agentId` es el id del agente propietario de OpenClaw.
- Los overrides ACP opcionales viven en `bindings[].acp`:
  - `mode` (`persistent` o `oneshot`)
  - `label`
  - `cwd`
  - `backend`

### Valores predeterminados de runtime por agente

Usa `agents.list[].runtime` para definir valores predeterminados ACP una vez por agente:

- `agents.list[].runtime.type="acp"`
- `agents.list[].runtime.acp.agent` (id del harness, por ejemplo `codex` o `claude`)
- `agents.list[].runtime.acp.backend`
- `agents.list[].runtime.acp.mode`
- `agents.list[].runtime.acp.cwd`

Precedencia de overrides para sesiones ACP vinculadas:

1. `bindings[].acp.*`
2. `agents.list[].runtime.acp.*`
3. valores predeterminados globales de ACP (por ejemplo `acp.backend`)

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

- OpenClaw garantiza que la sesión ACP configurada exista antes de usarla.
- Los mensajes en ese canal o tema se enrutan a la sesión ACP configurada.
- En conversaciones vinculadas, `/new` y `/reset` reinician la misma clave de sesión ACP en el mismo lugar.
- Las vinculaciones temporales de runtime (por ejemplo creadas por flujos de enfoque de hilo) siguen aplicándose cuando están presentes.
- Para creaciones ACP entre agentes sin `cwd` explícito, OpenClaw hereda el espacio de trabajo del agente objetivo desde la configuración del agente.
- Las rutas heredadas del espacio de trabajo que no existen recurren al `cwd` predeterminado del backend; los fallos reales de acceso se muestran como errores de creación.

## Iniciar sesiones ACP (interfaces)

### Desde `sessions_spawn`

Usa `runtime: "acp"` para iniciar una sesión ACP desde un turno del agente o una llamada a herramienta.

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

- `runtime` usa `subagent` de forma predeterminada, así que establece `runtime: "acp"` explícitamente para sesiones ACP.
- Si se omite `agentId`, OpenClaw usa `acp.defaultAgent` cuando está configurado.
- `mode: "session"` requiere `thread: true` para mantener una conversación persistente vinculada.

Detalles de la interfaz:

- `task` (obligatorio): prompt inicial enviado a la sesión ACP.
- `runtime` (obligatorio para ACP): debe ser `"acp"`.
- `agentId` (opcional): id del harness ACP objetivo. Recurre a `acp.defaultAgent` si está definido.
- `thread` (opcional, predeterminado `false`): solicita un flujo de vinculación con hilo donde se admita.
- `mode` (opcional): `run` (puntual) o `session` (persistente).
  - el valor predeterminado es `run`
  - si `thread: true` y se omite mode, OpenClaw puede usar comportamiento persistente por defecto según la ruta de runtime
  - `mode: "session"` requiere `thread: true`
- `cwd` (opcional): directorio de trabajo solicitado para el runtime (validado por la política del backend/runtime). Si se omite, la creación ACP hereda el espacio de trabajo del agente objetivo cuando está configurado; las rutas heredadas que no existen recurren a los valores predeterminados del backend, mientras que los errores reales de acceso se devuelven.
- `label` (opcional): etiqueta visible para el operador usada en el texto de sesión/banner.
- `resumeSessionId` (opcional): reanuda una sesión ACP existente en lugar de crear una nueva. El agente vuelve a reproducir su historial de conversación mediante `session/load`. Requiere `runtime: "acp"`.
- `streamTo` (opcional): `"parent"` transmite resúmenes del progreso de la ejecución ACP inicial de vuelta a la sesión solicitante como eventos del sistema.
  - Cuando está disponible, las respuestas aceptadas incluyen `streamLogPath` que apunta a un registro JSONL con alcance de sesión (`<sessionId>.acp-stream.jsonl`) que puedes seguir para ver el historial completo del relay.

### Reanudar una sesión existente

Usa `resumeSessionId` para continuar una sesión ACP anterior en lugar de empezar desde cero. El agente vuelve a reproducir su historial de conversación mediante `session/load`, por lo que retoma el trabajo con el contexto completo de lo anterior.

```json
{
  "task": "Continue where we left off — fix the remaining test failures",
  "runtime": "acp",
  "agentId": "codex",
  "resumeSessionId": "<previous-session-id>"
}
```

Casos de uso habituales:

- Transferir una sesión de Codex desde tu portátil a tu teléfono: dile a tu agente que retome donde lo dejaste
- Continuar una sesión de programación que empezaste de forma interactiva en la CLI, ahora sin interfaz mediante tu agente
- Retomar trabajo interrumpido por un reinicio del gateway o una caducidad por inactividad

Notas:

- `resumeSessionId` requiere `runtime: "acp"`; devuelve un error si se usa con el runtime de subagente.
- `resumeSessionId` restaura el historial de conversación ACP ascendente; `thread` y `mode` siguen aplicándose normalmente a la nueva sesión de OpenClaw que estás creando, por lo que `mode: "session"` sigue requiriendo `thread: true`.
- El agente objetivo debe admitir `session/load` (Codex y Claude Code lo hacen).
- Si no se encuentra el id de sesión, la creación falla con un error claro; no hay retorno silencioso a una sesión nueva.

### Prueba rápida para operadores

Úsala después de desplegar un gateway cuando quieras una comprobación rápida en vivo de que la creación ACP
realmente funciona de extremo a extremo, no solo que pasen las pruebas unitarias.

Puerta recomendada:

1. Verifica la versión/commit del gateway desplegado en el host de destino.
2. Confirma que el código desplegado incluye la aceptación de linaje ACP en
   `src/gateway/sessions-patch.ts` (`subagent:* or acp:* sessions`).
3. Abre una sesión puente ACPX temporal hacia un agente en vivo (por ejemplo
   `razor(main)` en `jpclawhq`).
4. Pide a ese agente que llame a `sessions_spawn` con:
   - `runtime: "acp"`
   - `agentId: "codex"`
   - `mode: "run"`
   - task: `Reply with exactly LIVE-ACP-SPAWN-OK`
5. Verifica que el agente informe:
   - `accepted=yes`
   - una `childSessionKey` real
   - sin error de validación
6. Limpia la sesión puente ACPX temporal.

Ejemplo de prompt para el agente en vivo:

```text
Use the sessions_spawn tool now with runtime: "acp", agentId: "codex", and mode: "run".
Set the task to: "Reply with exactly LIVE-ACP-SPAWN-OK".
Then report only: accepted=<yes/no>; childSessionKey=<value or none>; error=<exact text or none>.
```

Notas:

- Mantén esta prueba rápida en `mode: "run"` a menos que estés probando intencionalmente
  sesiones ACP persistentes vinculadas a hilos.
- No exijas `streamTo: "parent"` para la puerta básica. Esa ruta depende de
  las capacidades de la sesión/solicitante y es una comprobación de integración aparte.
- Trata las pruebas de `mode: "session"` vinculadas a hilos como una segunda
  pasada de integración más rica desde un hilo real de Discord o un tema de Telegram.

## Compatibilidad con sandbox

Las sesiones ACP se ejecutan actualmente en el runtime del host, no dentro del sandbox de OpenClaw.

Limitaciones actuales:

- Si la sesión solicitante está en sandbox, las creaciones ACP se bloquean tanto para `sessions_spawn({ runtime: "acp" })` como para `/acp spawn`.
  - Error: `Sandboxed sessions cannot spawn ACP sessions because runtime="acp" runs on the host. Use runtime="subagent" from sandboxed sessions.`
- `sessions_spawn` con `runtime: "acp"` no admite `sandbox: "require"`.
  - Error: `sessions_spawn sandbox="require" is unsupported for runtime="acp" because ACP sessions run outside the sandbox. Use runtime="subagent" or sandbox="inherit".`

Usa `runtime: "subagent"` cuando necesites ejecución forzada por sandbox.

### Desde el comando `/acp`

Usa `/acp spawn` cuando necesites control explícito del operador desde el chat.

```text
/acp spawn codex --mode persistent --thread auto
/acp spawn codex --mode oneshot --thread off
/acp spawn codex --bind here
/acp spawn codex --thread here
```

Indicadores clave:

- `--mode persistent|oneshot`
- `--bind here|off`
- `--thread auto|here|off`
- `--cwd <absolute-path>`
- `--label <name>`

Consulta [Comandos slash](/es/tools/slash-commands).

## Resolución de objetivos de sesión

La mayoría de las acciones `/acp` aceptan un objetivo de sesión opcional (`session-key`, `session-id` o `session-label`).

Orden de resolución:

1. Argumento de objetivo explícito (o `--session` para `/acp steer`)
   - primero intenta con la clave
   - luego con un id de sesión con forma de UUID
   - luego con la etiqueta
2. Vinculación del hilo actual (si esta conversación/hilo está vinculada a una sesión ACP)
3. Respaldo de la sesión solicitante actual

Las vinculaciones con la conversación actual y las vinculaciones con hilos participan ambas en el paso 2.

Si no se resuelve ningún objetivo, OpenClaw devuelve un error claro (`Unable to resolve session target: ...`).

## Modos de vinculación al crear

`/acp spawn` admite `--bind here|off`.

| Modo   | Comportamiento                                                               |
| ------ | ---------------------------------------------------------------------- |
| `here` | Vincula la conversación activa actual en su lugar; falla si no hay ninguna activa. |
| `off`  | No crea una vinculación con la conversación actual.                          |

Notas:

- `--bind here` es la ruta más sencilla para el operador para “hacer que este canal o chat use Codex”.
- `--bind here` no crea un hilo secundario.
- `--bind here` solo está disponible en canales que exponen compatibilidad con vinculación de conversación actual.
- `--bind` y `--thread` no pueden combinarse en la misma llamada a `/acp spawn`.

## Modos de hilo al crear

`/acp spawn` admite `--thread auto|here|off`.

| Modo   | Comportamiento                                                                                            |
| ------ | --------------------------------------------------------------------------------------------------- |
| `auto` | En un hilo activo: vincula ese hilo. Fuera de un hilo: crea/vincula un hilo secundario cuando se admita. |
| `here` | Requiere un hilo activo actual; falla si no estás dentro de uno.                                                  |
| `off`  | Sin vinculación. La sesión se inicia sin vincular.                                                                 |

Notas:

- En superficies sin vinculación de hilos, el comportamiento predeterminado equivale en la práctica a `off`.
- La creación vinculada a hilos requiere compatibilidad de política del canal:
  - Discord: `channels.discord.threadBindings.spawnAcpSessions=true`
  - Telegram: `channels.telegram.threadBindings.spawnAcpSessions=true`
- Usa `--bind here` cuando quieras fijar la conversación actual sin crear un hilo secundario.

## Controles ACP

Familia de comandos disponibles:

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

`/acp status` muestra las opciones efectivas del runtime y, cuando están disponibles, tanto los identificadores de sesión a nivel de runtime como a nivel de backend.

Algunos controles dependen de las capacidades del backend. Si un backend no admite un control, OpenClaw devuelve un error claro de control no compatible.

## Recetario de comandos ACP

| Comando              | Qué hace                                              | Ejemplo                                                       |
| -------------------- | --------------------------------------------------------- | ------------------------------------------------------------- |
| `/acp spawn`         | Crea una sesión ACP; vinculación actual o con hilo opcional. | `/acp spawn codex --bind here --cwd /repo`                    |
| `/acp cancel`        | Cancela el turno en curso para la sesión objetivo.                 | `/acp cancel agent:codex:acp:<uuid>`                          |
| `/acp steer`         | Envía una instrucción de dirección a la sesión en ejecución.                | `/acp steer --session support inbox prioritize failing tests` |
| `/acp close`         | Cierra la sesión y desvincula objetivos de hilo.                  | `/acp close`                                                  |
| `/acp status`        | Muestra backend, modo, estado, opciones de runtime y capacidades. | `/acp status`                                                 |
| `/acp set-mode`      | Establece el modo de runtime para la sesión objetivo.                      | `/acp set-mode plan`                                          |
| `/acp set`           | Escritura genérica de opción de configuración del runtime.                      | `/acp set model openai/gpt-5.4`                               |
| `/acp cwd`           | Establece el override del directorio de trabajo del runtime.                   | `/acp cwd /Users/user/Projects/repo`                          |
| `/acp permissions`   | Establece el perfil de política de aprobación.                              | `/acp permissions strict`                                     |
| `/acp timeout`       | Establece el timeout del runtime (segundos).                            | `/acp timeout 120`                                            |
| `/acp model`         | Establece el override del modelo del runtime.                               | `/acp model anthropic/claude-opus-4-6`                        |
| `/acp reset-options` | Elimina los overrides de opciones de runtime de la sesión.                  | `/acp reset-options`                                          |
| `/acp sessions`      | Lista sesiones ACP recientes desde el almacén.                      | `/acp sessions`                                               |
| `/acp doctor`        | Estado del backend, capacidades y correcciones accionables.           | `/acp doctor`                                                 |
| `/acp install`       | Imprime pasos deterministas de instalación y habilitación.             | `/acp install`                                                |

`/acp sessions` lee el almacén para la sesión actual vinculada o solicitante. Los comandos que aceptan tokens `session-key`, `session-id` o `session-label` resuelven objetivos mediante el descubrimiento de sesiones del gateway, incluidas raíces personalizadas `session.store` por agente.

## Mapeo de opciones de runtime

`/acp` tiene comandos de conveniencia y un setter genérico.

Operaciones equivalentes:

- `/acp model <id>` se asigna a la clave de configuración de runtime `model`.
- `/acp permissions <profile>` se asigna a la clave de configuración de runtime `approval_policy`.
- `/acp timeout <seconds>` se asigna a la clave de configuración de runtime `timeout`.
- `/acp cwd <path>` actualiza directamente el override de `cwd` del runtime.
- `/acp set <key> <value>` es la ruta genérica.
  - Caso especial: `key=cwd` usa la ruta de override de `cwd`.
- `/acp reset-options` borra todos los overrides de runtime para la sesión objetivo.

## Compatibilidad actual de harnesses acpx

Alias integrados actuales de harnesses acpx:

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

Cuando OpenClaw usa el backend acpx, prefiere estos valores para `agentId`, salvo que tu configuración de acpx defina alias personalizados de agentes.
Si tu instalación local de Cursor aún expone ACP como `agent acp`, sobrescribe el comando del agente `cursor` en tu configuración de acpx en lugar de cambiar el valor predeterminado integrado.

El uso directo de la CLI de acpx también puede apuntar a adaptadores arbitrarios mediante `--agent <command>`, pero esa vía de escape sin procesar es una función de la CLI de acpx (no la ruta normal de `agentId` en OpenClaw).

## Configuración requerida

Línea base del núcleo ACP:

```json5
{
  acp: {
    enabled: true,
    // Opcional. El valor predeterminado es true; ponlo en false para pausar el despacho ACP y mantener los controles /acp.
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

La configuración de vinculación de hilos es específica del adaptador de canal. Ejemplo para Discord:

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

Si la creación ACP vinculada a hilos no funciona, primero verifica el indicador de función del adaptador:

- Discord: `channels.discord.threadBindings.spawnAcpSessions=true`

Las vinculaciones con la conversación actual no requieren crear hilos secundarios. Requieren un contexto activo de conversación y un adaptador de canal que exponga vinculaciones ACP de conversación.

Consulta [Referencia de configuración](/es/gateway/configuration-reference).

## Configuración del plugin para backend acpx

Las instalaciones nuevas incluyen el plugin de runtime empaquetado `acpx` habilitado de forma predeterminada, por lo que ACP
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

Instalación de espacio de trabajo local durante el desarrollo:

```bash
openclaw plugins install ./path/to/local/acpx-plugin
```

Luego verifica el estado del backend:

```text
/acp doctor
```

### Configuración de comando y versión de acpx

De forma predeterminada, el plugin backend empaquetado acpx (`acpx`) usa el binario fijado localmente dentro del plugin:

1. El comando usa por defecto el `node_modules/.bin/acpx` local al plugin dentro del paquete del plugin ACPX.
2. La versión esperada usa por defecto el pin de la extensión.
3. El inicio registra inmediatamente el backend ACP como no preparado.
4. Un trabajo en segundo plano verifica `acpx --version`.
5. Si el binario local al plugin falta o no coincide, ejecuta:
   `npm install --omit=dev --no-save acpx@<pinned>` y vuelve a verificar.

Puedes sobrescribir comando/versión en la configuración del plugin:

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

- `command` acepta una ruta absoluta, ruta relativa o nombre de comando (`acpx`).
- Las rutas relativas se resuelven desde el directorio del espacio de trabajo de OpenClaw.
- `expectedVersion: "any"` desactiva la coincidencia estricta de versión.
- Cuando `command` apunta a un binario/ruta personalizados, la instalación automática local al plugin se desactiva.
- El arranque de OpenClaw sigue siendo no bloqueante mientras se ejecuta la comprobación de estado del backend.

Consulta [Plugins](/es/tools/plugin).

### Instalación automática de dependencias

Cuando instalas OpenClaw globalmente con `npm install -g openclaw`, las
dependencias del runtime acpx (binarios específicos de plataforma) se instalan automáticamente
mediante un hook postinstall. Si la instalación automática falla, el gateway sigue iniciándose
con normalidad e informa de la dependencia faltante mediante `openclaw acp doctor`.

### Puente MCP de herramientas de plugins

De forma predeterminada, las sesiones ACPX **no** exponen al harness ACP las herramientas registradas por plugins de OpenClaw.

Si quieres que agentes ACP como Codex o Claude Code llamen a herramientas de plugins instalados de
OpenClaw, como recuperación/almacenamiento de memoria, habilita el puente dedicado:

```bash
openclaw config set plugins.entries.acpx.config.pluginToolsMcpBridge true
```

Qué hace esto:

- Inyecta un servidor MCP integrado llamado `openclaw-plugin-tools` en el arranque
  de la sesión ACPX.
- Expone herramientas de plugins ya registradas por plugins instalados y habilitados de OpenClaw.
- Mantiene la función explícita y desactivada por defecto.

Notas de seguridad y confianza:

- Esto amplía la superficie de herramientas del harness ACP.
- Los agentes ACP obtienen acceso solo a herramientas de plugins ya activas en el gateway.
- Trátalo como el mismo límite de confianza que permitir que esos plugins se ejecuten en
  OpenClaw.
- Revisa los plugins instalados antes de habilitarlo.

Los `mcpServers` personalizados siguen funcionando como antes. El puente integrado de herramientas de plugins es una comodidad adicional de adhesión voluntaria, no un sustituto de la configuración genérica de servidores MCP.

### Configuración del timeout del runtime

El plugin empaquetado `acpx` usa por defecto un
timeout de 120 segundos para turnos embebidos del runtime. Esto da a harnesses más lentos como Gemini CLI tiempo suficiente para completar
el arranque e inicialización de ACP. Sobrescríbelo si tu host necesita un límite de runtime distinto:

```bash
openclaw config set plugins.entries.acpx.config.timeoutSeconds 180
```

Reinicia el gateway después de cambiar este valor.

## Configuración de permisos

Las sesiones ACP se ejecutan sin interacción: no hay TTY para aprobar o denegar prompts de permisos de escritura de archivos y ejecución de shell. El plugin acpx proporciona dos claves de configuración que controlan cómo se gestionan los permisos:

Estos permisos de harness de ACPX son independientes de las aprobaciones de exec de OpenClaw y de indicadores de bypass del proveedor en backends CLI como Claude CLI `--permission-mode bypassPermissions`. ACPX `approve-all` es el interruptor de emergencia a nivel de harness para sesiones ACP.

### `permissionMode`

Controla qué operaciones puede realizar el agente del harness sin pedir confirmación.

| Valor           | Comportamiento                                                  |
| --------------- | --------------------------------------------------------- |
| `approve-all`   | Aprueba automáticamente todas las escrituras de archivos y comandos de shell.          |
| `approve-reads` | Aprueba automáticamente solo lecturas; las escrituras y exec requieren prompts. |
| `deny-all`      | Deniega todos los prompts de permisos.                              |

### `nonInteractivePermissions`

Controla qué ocurre cuando debería mostrarse un prompt de permisos pero no hay un TTY interactivo disponible (que es siempre el caso en las sesiones ACP).

| Valor  | Comportamiento                                                          |
| ------ | ----------------------------------------------------------------- |
| `fail` | Aborta la sesión con `AcpRuntimeError`. **(predeterminado)**           |
| `deny` | Deniega silenciosamente el permiso y continúa (degradación elegante). |

### Configuración

Configúralo mediante la configuración del plugin:

```bash
openclaw config set plugins.entries.acpx.config.permissionMode approve-all
openclaw config set plugins.entries.acpx.config.nonInteractivePermissions fail
```

Reinicia el gateway después de cambiar estos valores.

> **Importante:** OpenClaw usa actualmente por defecto `permissionMode=approve-reads` y `nonInteractivePermissions=fail`. En sesiones ACP no interactivas, cualquier escritura o exec que active un prompt de permisos puede fallar con `AcpRuntimeError: Permission prompt unavailable in non-interactive mode`.
>
> Si necesitas restringir permisos, establece `nonInteractivePermissions` en `deny` para que las sesiones se degraden de forma elegante en lugar de bloquearse.

## Solución de problemas

| Síntoma                                                                     | Causa probable                                                                    | Solución                                                                                                                                                               |
| --------------------------------------------------------------------------- | ------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ACP runtime backend is not configured`                                     | Falta el plugin backend o está deshabilitado.                                             | Instala y habilita el plugin backend y luego ejecuta `/acp doctor`.                                                                                                        |
| `ACP is disabled by policy (acp.enabled=false)`                             | ACP está deshabilitado globalmente.                                                          | Establece `acp.enabled=true`.                                                                                                                                           |
| `ACP dispatch is disabled by policy (acp.dispatch.enabled=false)`           | El despacho desde mensajes normales del hilo está deshabilitado.                                  | Establece `acp.dispatch.enabled=true`.                                                                                                                                  |
| `ACP agent "<id>" is not allowed by policy`                                 | El agente no está en la lista de permitidos.                                                         | Usa un `agentId` permitido o actualiza `acp.allowedAgents`.                                                                                                              |
| `Unable to resolve session target: ...`                                     | Token incorrecto de clave/id/etiqueta.                                                         | Ejecuta `/acp sessions`, copia la clave/etiqueta exacta e inténtalo de nuevo.                                                                                                                 |
| `--bind here requires running /acp spawn inside an active ... conversation` | `--bind here` se usó sin una conversación activa vinculable.                     | Muévete al chat/canal objetivo e inténtalo de nuevo, o usa creación sin vincular.                                                                                                  |
| `Conversation bindings are unavailable for <channel>.`                      | El adaptador carece de capacidad de vinculación ACP con la conversación actual.                      | Usa `/acp spawn ... --thread ...` donde se admita, configura `bindings[]` de nivel superior o muévete a un canal compatible.                                              |
| `--thread here requires running /acp spawn inside an active ... thread`     | `--thread here` se usó fuera de un contexto de hilo.                                  | Muévete al hilo objetivo o usa `--thread auto`/`off`.                                                                                                               |
| `Only <user-id> can rebind this channel/conversation/thread.`               | Otro usuario es propietario del objetivo de vinculación activo.                                    | Vuelve a vincular como propietario o usa otra conversación o hilo.                                                                                                        |
| `Thread bindings are unavailable for <channel>.`                            | El adaptador carece de capacidad de vinculación de hilos.                                        | Usa `--thread off` o muévete a un adaptador/canal compatible.                                                                                                          |
| `Sandboxed sessions cannot spawn ACP sessions ...`                          | El runtime ACP está en el host; la sesión solicitante está en sandbox.                       | Usa `runtime="subagent"` desde sesiones en sandbox, o ejecuta la creación ACP desde una sesión fuera de sandbox.                                                                  |
| `sessions_spawn sandbox="require" is unsupported for runtime="acp" ...`     | Se solicitó `sandbox="require"` para el runtime ACP.                                  | Usa `runtime="subagent"` para sandbox obligatorio, o usa ACP con `sandbox="inherit"` desde una sesión fuera de sandbox.                                               |
| Faltan metadatos ACP para la sesión vinculada                                      | Metadatos ACP obsoletos/eliminados.                                             | Vuelve a crear con `/acp spawn` y luego vuelve a vincular/enfocar el hilo.                                                                                                             |
| `AcpRuntimeError: Permission prompt unavailable in non-interactive mode`    | `permissionMode` bloquea escrituras/exec en sesión ACP no interactiva.             | Establece `plugins.entries.acpx.config.permissionMode` en `approve-all` y reinicia el gateway. Consulta [Configuración de permisos](#permission-configuration).                 |
| La sesión ACP falla pronto con poca salida                                  | Los prompts de permisos están bloqueados por `permissionMode`/`nonInteractivePermissions`. | Comprueba los registros del gateway para `AcpRuntimeError`. Para permisos completos, establece `permissionMode=approve-all`; para degradación elegante, establece `nonInteractivePermissions=deny`. |
| La sesión ACP se queda bloqueada indefinidamente tras completar el trabajo                       | El proceso del harness terminó pero la sesión ACP no informó la finalización.             | Supervísalo con `ps aux \| grep acpx`; mata manualmente los procesos obsoletos.                                                                                                |
