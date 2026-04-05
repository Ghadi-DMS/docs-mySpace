---
read_when:
    - Configuración de integraciones de IDE basadas en ACP
    - Depuración del enrutamiento de sesiones ACP hacia el Gateway
summary: Ejecuta el puente ACP para integraciones de IDE
title: acp
x-i18n:
    generated_at: "2026-04-05T12:37:43Z"
    model: gpt-5.4
    provider: openai
    source_hash: 2461b181e4a97dd84580581e9436ca1947a224decce8044132dbcf7fb2b7502c
    source_path: cli/acp.md
    workflow: 15
---

# acp

Ejecuta el puente de [Agent Client Protocol (ACP)](https://agentclientprotocol.com/) que se comunica con un Gateway de OpenClaw.

Este comando habla ACP sobre stdio para IDEs y reenvía prompts al Gateway
mediante WebSocket. Mantiene las sesiones de ACP asignadas a claves de sesión del Gateway.

`openclaw acp` es un puente ACP respaldado por Gateway, no un entorno de
ejecución de editor totalmente nativo de ACP. Se centra en el enrutamiento de
sesiones, la entrega de prompts y actualizaciones básicas de streaming.

Si quieres que un cliente MCP externo se comunique directamente con las
conversaciones de canal de OpenClaw en lugar de alojar una sesión de arnés ACP,
usa [`openclaw mcp serve`](/cli/mcp) en su lugar.

## Qué no es esto

Esta página suele confundirse con las sesiones de arnés ACP.

`openclaw acp` significa:

- OpenClaw actúa como servidor ACP
- un IDE o cliente ACP se conecta a OpenClaw
- OpenClaw reenvía ese trabajo a una sesión del Gateway

Esto es diferente de [ACP Agents](/tools/acp-agents), donde OpenClaw ejecuta un
arnés externo como Codex o Claude Code mediante `acpx`.

Regla rápida:

- si el editor/cliente quiere hablar ACP con OpenClaw: usa `openclaw acp`
- si OpenClaw debe iniciar Codex/Claude/Gemini como arnés ACP: usa `/acp spawn` y [ACP Agents](/tools/acp-agents)

## Matriz de compatibilidad

| Área de ACP                                                           | Estado      | Notas                                                                                                                                                                                                                                               |
| --------------------------------------------------------------------- | ----------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `initialize`, `newSession`, `prompt`, `cancel`                        | Implementado | Flujo principal del puente sobre stdio hacia chat/send + cancelación del Gateway.                                                                                                                                                                  |
| `listSessions`, comandos slash                                        | Implementado | La lista de sesiones funciona con el estado de sesiones del Gateway; los comandos se anuncian mediante `available_commands_update`.                                                                                                                |
| `loadSession`                                                         | Parcial     | Vuelve a vincular la sesión ACP a una clave de sesión del Gateway y reproduce el historial almacenado de texto de usuario/asistente. El historial de herramientas/sistema todavía no se reconstruye.                                             |
| Contenido del prompt (`text`, `resource` incrustado, imágenes)        | Parcial     | El texto y los recursos se aplanan en la entrada de chat; las imágenes se convierten en adjuntos del Gateway.                                                                                                                                     |
| Modos de sesión                                                       | Parcial     | `session/set_mode` es compatible y el puente expone controles iniciales de sesión respaldados por Gateway para nivel de pensamiento, verbosidad de herramientas, razonamiento, detalle de uso y acciones elevadas. Las superficies más amplias de modo/configuración nativas de ACP siguen fuera de alcance. |
| Información de sesión y actualizaciones de uso                        | Parcial     | El puente emite notificaciones `session_info_update` y `usage_update` de mejor esfuerzo a partir de instantáneas almacenadas en caché de la sesión del Gateway. El uso es aproximado y solo se envía cuando los totales de tokens del Gateway se marcan como recientes. |
| Streaming de herramientas                                             | Parcial     | Los eventos `tool_call` / `tool_call_update` incluyen E/S sin procesar, contenido de texto y ubicaciones de archivos de mejor esfuerzo cuando los argumentos/resultados de herramientas del Gateway los exponen. Los terminales incrustados y la salida nativa de diffs más rica aún no se exponen. |
| Servidores MCP por sesión (`mcpServers`)                              | No compatible | El modo puente rechaza solicitudes de servidores MCP por sesión. Configura MCP en el gateway o el agente de OpenClaw en su lugar.                                                                                                                |
| Métodos de sistema de archivos del cliente (`fs/read_text_file`, `fs/write_text_file`) | No compatible | El puente no llama a métodos del sistema de archivos del cliente ACP.                                                                                                                                                                              |
| Métodos de terminal del cliente (`terminal/*`)                        | No compatible | El puente no crea terminales de cliente ACP ni transmite ids de terminal a través de llamadas de herramientas.                                                                                                                                     |
| Planes de sesión / streaming de pensamiento                           | No compatible | Actualmente, el puente emite texto de salida y estado de herramientas, no planes ACP ni actualizaciones de pensamiento.                                                                                                                           |

## Limitaciones conocidas

- `loadSession` reproduce el historial almacenado de texto de usuario y
  asistente, pero no reconstruye llamadas históricas de herramientas, avisos
  del sistema ni tipos de eventos nativos de ACP más ricos.
- Si varios clientes ACP comparten la misma clave de sesión del Gateway, el
  enrutamiento de eventos y cancelaciones es de mejor esfuerzo en lugar de
  estar estrictamente aislado por cliente. Prefiere las sesiones aisladas
  predeterminadas `acp:<uuid>` cuando necesites turnos locales del editor
  limpios.
- Los estados de detención del Gateway se traducen a motivos de detención de
  ACP, pero esa asignación es menos expresiva que un entorno de ejecución
  totalmente nativo de ACP.
- Los controles iniciales de sesión muestran actualmente un subconjunto
  enfocado de opciones del Gateway: nivel de pensamiento, verbosidad de
  herramientas, razonamiento, detalle de uso y acciones elevadas. La selección
  de modelo y los controles del host de ejecución todavía no se exponen como
  opciones de configuración de ACP.
- `session_info_update` y `usage_update` se derivan de instantáneas de sesión
  del Gateway, no de una contabilidad en vivo nativa de ACP. El uso es
  aproximado, no incluye datos de costo y solo se emite cuando el Gateway marca
  como recientes los datos totales de tokens.
- Los datos de seguimiento de herramientas son de mejor esfuerzo. El puente
  puede mostrar rutas de archivos que aparezcan en argumentos/resultados de
  herramientas conocidos, pero todavía no emite terminales ACP ni diffs de
  archivos estructurados.

## Uso

```bash
openclaw acp

# Remote Gateway
openclaw acp --url wss://gateway-host:18789 --token <token>

# Remote Gateway (token from file)
openclaw acp --url wss://gateway-host:18789 --token-file ~/.openclaw/gateway.token

# Attach to an existing session key
openclaw acp --session agent:main:main

# Attach by label (must already exist)
openclaw acp --session-label "support inbox"

# Reset the session key before the first prompt
openclaw acp --session agent:main:main --reset-session
```

## Cliente ACP (depuración)

Usa el cliente ACP integrado para verificar rápidamente el puente sin un IDE.
Inicia el puente ACP y te permite escribir prompts de forma interactiva.

```bash
openclaw acp client

# Point the spawned bridge at a remote Gateway
openclaw acp client --server-args --url wss://gateway-host:18789 --token-file ~/.openclaw/gateway.token

# Override the server command (default: openclaw)
openclaw acp client --server "node" --server-args openclaw.mjs acp --url ws://127.0.0.1:19001
```

Modelo de permisos (modo de depuración del cliente):

- La aprobación automática se basa en lista de permitidos y solo se aplica a ids de herramientas principales de confianza.
- La aprobación automática de `read` se limita al directorio de trabajo actual (`--cwd` cuando está definido).
- ACP solo aprueba automáticamente clases de solo lectura limitadas: llamadas `read` dentro del cwd activo más herramientas de búsqueda de solo lectura (`search`, `web_search`, `memory_search`). Las herramientas desconocidas/no principales, lecturas fuera de alcance, herramientas con capacidad de ejecución, herramientas del plano de control, herramientas mutables y flujos interactivos siempre requieren aprobación explícita mediante prompt.
- `toolCall.kind` proporcionado por el servidor se trata como metadatos no confiables (no como fuente de autorización).
- Esta política del puente ACP es independiente de los permisos del arnés ACPX. Si ejecutas OpenClaw mediante el backend `acpx`, `plugins.entries.acpx.config.permissionMode=approve-all` es el interruptor de emergencia “yolo” para esa sesión de arnés.

## Cómo usar esto

Usa ACP cuando un IDE (u otro cliente) habla Agent Client Protocol y quieres que
controle una sesión del Gateway de OpenClaw.

1. Asegúrate de que el Gateway esté en ejecución (local o remoto).
2. Configura el destino del Gateway (configuración o flags).
3. Haz que tu IDE ejecute `openclaw acp` sobre stdio.

Ejemplo de configuración (persistente):

```bash
openclaw config set gateway.remote.url wss://gateway-host:18789
openclaw config set gateway.remote.token <token>
```

Ejemplo de ejecución directa (sin escribir configuración):

```bash
openclaw acp --url wss://gateway-host:18789 --token <token>
# preferred for local process safety
openclaw acp --url wss://gateway-host:18789 --token-file ~/.openclaw/gateway.token
```

## Selección de agentes

ACP no selecciona agentes directamente. Enruta por la clave de sesión del Gateway.

Usa claves de sesión con ámbito de agente para dirigirte a un agente específico:

```bash
openclaw acp --session agent:main:main
openclaw acp --session agent:design:main
openclaw acp --session agent:qa:bug-123
```

Cada sesión ACP se asigna a una única clave de sesión del Gateway. Un agente
puede tener muchas sesiones; ACP usa por defecto una sesión aislada `acp:<uuid>`
a menos que sobrescribas la clave o la etiqueta.

Los `mcpServers` por sesión no son compatibles en modo puente. Si un cliente ACP
los envía durante `newSession` o `loadSession`, el puente devuelve un error
claro en lugar de ignorarlos en silencio.

Si quieres que las sesiones respaldadas por ACPX vean herramientas de plugin de
OpenClaw, habilita el puente de plugins ACPX del lado del gateway en lugar de
intentar pasar `mcpServers` por sesión. Consulta [ACP Agents](/tools/acp-agents#plugin-tools-mcp-bridge).

## Uso desde `acpx` (Codex, Claude, otros clientes ACP)

Si quieres que un agente de programación como Codex o Claude Code hable con tu
bot de OpenClaw mediante ACP, usa `acpx` con su destino integrado `openclaw`.

Flujo típico:

1. Ejecuta el Gateway y asegúrate de que el puente ACP pueda alcanzarlo.
2. Haz que `acpx openclaw` apunte a `openclaw acp`.
3. Dirige la comunicación a la clave de sesión de OpenClaw que quieres que use el agente de programación.

Ejemplos:

```bash
# One-shot request into your default OpenClaw ACP session
acpx openclaw exec "Summarize the active OpenClaw session state."

# Persistent named session for follow-up turns
acpx openclaw sessions ensure --name codex-bridge
acpx openclaw -s codex-bridge --cwd /path/to/repo \
  "Ask my OpenClaw work agent for recent context relevant to this repo."
```

Si quieres que `acpx openclaw` apunte siempre a un Gateway y una clave de sesión
específicos, sobrescribe el comando del agente `openclaw` en `~/.acpx/config.json`:

```json
{
  "agents": {
    "openclaw": {
      "command": "env OPENCLAW_HIDE_BANNER=1 OPENCLAW_SUPPRESS_NOTES=1 openclaw acp --url ws://127.0.0.1:18789 --token-file ~/.openclaw/gateway.token --session agent:main:main"
    }
  }
}
```

Para una copia local de OpenClaw del repositorio, usa el punto de entrada directo
de la CLI en lugar del ejecutor de desarrollo para que el flujo ACP permanezca limpio.
Por ejemplo:

```bash
env OPENCLAW_HIDE_BANNER=1 OPENCLAW_SUPPRESS_NOTES=1 node openclaw.mjs acp ...
```

Esta es la forma más sencilla de permitir que Codex, Claude Code u otro cliente
compatible con ACP obtenga información contextual de un agente de OpenClaw sin
raspar un terminal.

## Configuración del editor Zed

Añade un agente ACP personalizado en `~/.config/zed/settings.json` (o usa la interfaz de configuración de Zed):

```json
{
  "agent_servers": {
    "OpenClaw ACP": {
      "type": "custom",
      "command": "openclaw",
      "args": ["acp"],
      "env": {}
    }
  }
}
```

Para dirigirte a un Gateway o agente específico:

```json
{
  "agent_servers": {
    "OpenClaw ACP": {
      "type": "custom",
      "command": "openclaw",
      "args": [
        "acp",
        "--url",
        "wss://gateway-host:18789",
        "--token",
        "<token>",
        "--session",
        "agent:design:main"
      ],
      "env": {}
    }
  }
}
```

En Zed, abre el panel Agent y selecciona “OpenClaw ACP” para iniciar un hilo.

## Asignación de sesiones

De forma predeterminada, las sesiones ACP reciben una clave de sesión aislada del Gateway con el prefijo `acp:`.
Para reutilizar una sesión conocida, pasa una clave de sesión o etiqueta:

- `--session <key>`: usa una clave de sesión específica del Gateway.
- `--session-label <label>`: resuelve una sesión existente por etiqueta.
- `--reset-session`: genera un id de sesión nuevo para esa clave (misma clave, nueva transcripción).

Si tu cliente ACP admite metadatos, puedes sobrescribirlo por sesión:

```json
{
  "_meta": {
    "sessionKey": "agent:main:main",
    "sessionLabel": "support inbox",
    "resetSession": true
  }
}
```

Más información sobre claves de sesión en [/concepts/session](/concepts/session).

## Opciones

- `--url <url>`: URL WebSocket del Gateway (usa por defecto `gateway.remote.url` cuando está configurado).
- `--token <token>`: token de autenticación del Gateway.
- `--token-file <path>`: lee el token de autenticación del Gateway desde un archivo.
- `--password <password>`: contraseña de autenticación del Gateway.
- `--password-file <path>`: lee la contraseña de autenticación del Gateway desde un archivo.
- `--session <key>`: clave de sesión predeterminada.
- `--session-label <label>`: etiqueta de sesión predeterminada a resolver.
- `--require-existing`: falla si la clave/etiqueta de sesión no existe.
- `--reset-session`: restablece la clave de sesión antes del primer uso.
- `--no-prefix-cwd`: no antepone el directorio de trabajo a los prompts.
- `--provenance <off|meta|meta+receipt>`: incluye metadatos o comprobantes de procedencia de ACP.
- `--verbose, -v`: registro detallado en stderr.

Nota de seguridad:

- `--token` y `--password` pueden ser visibles en listados locales de procesos en algunos sistemas.
- Prefiere `--token-file`/`--password-file` o variables de entorno (`OPENCLAW_GATEWAY_TOKEN`, `OPENCLAW_GATEWAY_PASSWORD`).
- La resolución de autenticación del Gateway sigue el contrato compartido que usan otros clientes del Gateway:
  - modo local: variables de entorno (`OPENCLAW_GATEWAY_*`) -> `gateway.auth.*` -> respaldo en `gateway.remote.*` solo cuando `gateway.auth.*` no está definido (los SecretRef locales configurados pero no resueltos fallan en modo cerrado)
  - modo remoto: `gateway.remote.*` con respaldo de variables de entorno/configuración según las reglas de precedencia remota
  - `--url` es seguro para sobrescritura y no reutiliza credenciales implícitas de configuración/entorno; pasa `--token`/`--password` explícitos (o sus variantes con archivo)
- Los procesos hijos del backend de ejecución ACP reciben `OPENCLAW_SHELL=acp`, que puede usarse para reglas específicas de contexto de shell/perfil.
- `openclaw acp client` establece `OPENCLAW_SHELL=acp-client` en el proceso del puente generado.

### Opciones de `acp client`

- `--cwd <dir>`: directorio de trabajo para la sesión ACP.
- `--server <command>`: comando del servidor ACP (predeterminado: `openclaw`).
- `--server-args <args...>`: argumentos adicionales pasados al servidor ACP.
- `--server-verbose`: habilita el registro detallado en el servidor ACP.
- `--verbose, -v`: registro detallado del cliente.
