---
read_when:
    - Conectar Codex, Claude Code u otro cliente MCP a canales respaldados por OpenClaw
    - Ejecutar `openclaw mcp serve`
    - Administrar definiciones guardadas de servidores MCP de OpenClaw
summary: Exponer conversaciones de canales de OpenClaw mediante MCP y administrar definiciones guardadas de servidores MCP
title: mcp
x-i18n:
    generated_at: "2026-04-05T12:38:43Z"
    model: gpt-5.4
    provider: openai
    source_hash: b35de9e14f96666eeca2f93c06cb214e691152f911d45ee778efe9cf5bf96cc2
    source_path: cli/mcp.md
    workflow: 15
---

# mcp

`openclaw mcp` tiene dos funciones:

- ejecutar OpenClaw como servidor MCP con `openclaw mcp serve`
- administrar las definiciones salientes de servidores MCP propiedad de OpenClaw con `list`, `show`,
  `set` y `unset`

En otras palabras:

- `serve` es OpenClaw actuando como servidor MCP
- `list` / `show` / `set` / `unset` es OpenClaw actuando como un registro del lado cliente de MCP
  para otros servidores MCP que sus tiempos de ejecución puedan consumir más adelante

Usa [`openclaw acp`](/cli/acp) cuando OpenClaw deba alojar una
sesión de arnés de programación por sí mismo y enrutar ese tiempo de ejecución a través de ACP.

## OpenClaw como servidor MCP

Esta es la ruta de `openclaw mcp serve`.

## Cuándo usar `serve`

Usa `openclaw mcp serve` cuando:

- Codex, Claude Code u otro cliente MCP deba hablar directamente con
  conversaciones de canales respaldadas por OpenClaw
- ya tengas un Gateway de OpenClaw local o remoto con sesiones enrutadas
- quieras un único servidor MCP que funcione en todos los backends de canales de OpenClaw en lugar
  de ejecutar puentes separados por canal

Usa [`openclaw acp`](/cli/acp) en su lugar cuando OpenClaw deba alojar el tiempo de ejecución
de programación por sí mismo y mantener la sesión del agente dentro de OpenClaw.

## Cómo funciona

`openclaw mcp serve` inicia un servidor MCP por stdio. El cliente MCP es propietario de ese
proceso. Mientras el cliente mantenga abierta la sesión stdio, el puente se conecta a un
Gateway de OpenClaw local o remoto mediante WebSocket y expone conversaciones de canales enrutadas
a través de MCP.

Ciclo de vida:

1. el cliente MCP lanza `openclaw mcp serve`
2. el puente se conecta al Gateway
3. las sesiones enrutadas se convierten en conversaciones MCP y herramientas de transcripción/historial
4. los eventos en vivo se ponen en cola en memoria mientras el puente está conectado
5. si el modo de canal de Claude está habilitado, la misma sesión también puede recibir
   notificaciones push específicas de Claude

Comportamiento importante:

- el estado de la cola en vivo comienza cuando el puente se conecta
- el historial de transcripción anterior se lee con `messages_read`
- las notificaciones push de Claude solo existen mientras la sesión MCP está activa
- cuando el cliente se desconecta, el puente sale y la cola en vivo desaparece

## Elegir un modo de cliente

Usa el mismo puente de dos formas distintas:

- Clientes MCP genéricos: solo herramientas MCP estándar. Usa `conversations_list`,
  `messages_read`, `events_poll`, `events_wait`, `messages_send` y las
  herramientas de aprobación.
- Claude Code: herramientas MCP estándar más el adaptador de canal específico de Claude.
  Habilita `--claude-channel-mode on` o deja el valor predeterminado `auto`.

Hoy, `auto` se comporta igual que `on`. Aún no hay detección de capacidades del cliente.

## Qué expone `serve`

El puente usa los metadatos de ruta de sesión existentes del Gateway para exponer conversaciones
respaldadas por canales. Una conversación aparece cuando OpenClaw ya tiene estado de sesión
con una ruta conocida como:

- `channel`
- metadatos de destinatario o destino
- `accountId` opcional
- `threadId` opcional

Esto ofrece a los clientes MCP un lugar único para:

- enumerar conversaciones enrutadas recientes
- leer historial de transcripción reciente
- esperar nuevos eventos entrantes
- enviar una respuesta de vuelta a través de la misma ruta
- ver solicitudes de aprobación que lleguen mientras el puente esté conectado

## Uso

```bash
# Local Gateway
openclaw mcp serve

# Remote Gateway
openclaw mcp serve --url wss://gateway-host:18789 --token-file ~/.openclaw/gateway.token

# Remote Gateway with password auth
openclaw mcp serve --url wss://gateway-host:18789 --password-file ~/.openclaw/gateway.password

# Enable verbose bridge logs
openclaw mcp serve --verbose

# Disable Claude-specific push notifications
openclaw mcp serve --claude-channel-mode off
```

## Herramientas del puente

El puente actual expone estas herramientas MCP:

- `conversations_list`
- `conversation_get`
- `messages_read`
- `attachments_fetch`
- `events_poll`
- `events_wait`
- `messages_send`
- `permissions_list_open`
- `permissions_respond`

### `conversations_list`

Enumera conversaciones recientes respaldadas por sesión que ya tienen metadatos de ruta en
el estado de sesión del Gateway.

Filtros útiles:

- `limit`
- `search`
- `channel`
- `includeDerivedTitles`
- `includeLastMessage`

### `conversation_get`

Devuelve una conversación por `session_key`.

### `messages_read`

Lee mensajes recientes de la transcripción para una conversación respaldada por sesión.

### `attachments_fetch`

Extrae bloques de contenido no textual de un mensaje de transcripción. Esta es una
vista de metadatos sobre el contenido de la transcripción, no un almacén independiente y duradero
de blobs adjuntos.

### `events_poll`

Lee eventos en vivo en cola desde un cursor numérico.

### `events_wait`

Hace long-polling hasta que llega el siguiente evento en cola que coincida o hasta que expire el tiempo de espera.

Usa esto cuando un cliente MCP genérico necesite entrega casi en tiempo real sin un
protocolo push específico de Claude.

### `messages_send`

Envía texto de vuelta a través de la misma ruta ya registrada en la sesión.

Comportamiento actual:

- requiere una ruta de conversación existente
- usa el canal, destinatario, id de cuenta e id de hilo de la sesión
- envía solo texto

### `permissions_list_open`

Enumera las solicitudes pendientes de aprobación de exec/plugin que el puente ha observado desde que
se conectó al Gateway.

### `permissions_respond`

Resuelve una solicitud pendiente de aprobación de exec/plugin con:

- `allow-once`
- `allow-always`
- `deny`

## Modelo de eventos

El puente mantiene una cola de eventos en memoria mientras está conectado.

Tipos de eventos actuales:

- `message`
- `exec_approval_requested`
- `exec_approval_resolved`
- `plugin_approval_requested`
- `plugin_approval_resolved`
- `claude_permission_request`

Límites importantes:

- la cola es solo en vivo; comienza cuando se inicia el puente MCP
- `events_poll` y `events_wait` no reproducen por sí solos el historial anterior del Gateway
- el historial duradero debe leerse con `messages_read`

## Notificaciones de canal de Claude

El puente también puede exponer notificaciones de canal específicas de Claude. Este es el
equivalente en OpenClaw de un adaptador de canal de Claude Code: las herramientas MCP estándar siguen
disponibles, pero los mensajes entrantes en vivo también pueden llegar como notificaciones MCP específicas de Claude.

Indicadores:

- `--claude-channel-mode off`: solo herramientas MCP estándar
- `--claude-channel-mode on`: habilita notificaciones de canal de Claude
- `--claude-channel-mode auto`: valor predeterminado actual; mismo comportamiento del puente que `on`

Cuando el modo de canal de Claude está habilitado, el servidor anuncia capacidades experimentales
de Claude y puede emitir:

- `notifications/claude/channel`
- `notifications/claude/channel/permission`

Comportamiento actual del puente:

- los mensajes de transcripción entrantes de `user` se reenvían como
  `notifications/claude/channel`
- las solicitudes de permisos de Claude recibidas mediante MCP se rastrean en memoria
- si la conversación vinculada luego envía `yes abcde` o `no abcde`, el puente
  convierte eso en `notifications/claude/channel/permission`
- estas notificaciones son solo de la sesión en vivo; si el cliente MCP se desconecta,
  no hay destino push

Esto es intencionalmente específico del cliente. Los clientes MCP genéricos deben basarse en las
herramientas estándar de polling.

## Configuración del cliente MCP

Ejemplo de configuración de cliente stdio:

```json
{
  "mcpServers": {
    "openclaw": {
      "command": "openclaw",
      "args": [
        "mcp",
        "serve",
        "--url",
        "wss://gateway-host:18789",
        "--token-file",
        "/path/to/gateway.token"
      ]
    }
  }
}
```

Para la mayoría de los clientes MCP genéricos, comienza con la superficie estándar de herramientas e ignora
el modo Claude. Activa el modo Claude solo para clientes que realmente entiendan los
métodos de notificación específicos de Claude.

## Opciones

`openclaw mcp serve` admite:

- `--url <url>`: URL WebSocket del Gateway
- `--token <token>`: token del Gateway
- `--token-file <path>`: leer el token desde un archivo
- `--password <password>`: contraseña del Gateway
- `--password-file <path>`: leer la contraseña desde un archivo
- `--claude-channel-mode <auto|on|off>`: modo de notificación de Claude
- `-v`, `--verbose`: registros detallados en stderr

Prefiere `--token-file` o `--password-file` en lugar de secretos en línea cuando sea posible.

## Seguridad y límite de confianza

El puente no inventa el enrutamiento. Solo expone conversaciones que el Gateway
ya sabe cómo enrutar.

Eso significa:

- las listas de permitidos de remitentes, el emparejamiento y la confianza a nivel de canal siguen perteneciendo a la
  configuración subyacente del canal de OpenClaw
- `messages_send` solo puede responder a través de una ruta almacenada existente
- el estado de aprobación es solo en vivo/en memoria para la sesión actual del puente
- la autenticación del puente debe usar los mismos controles de token o contraseña del Gateway en los que
  confiarías para cualquier otro cliente remoto del Gateway

Si una conversación falta en `conversations_list`, la causa habitual no es la
configuración de MCP. Son metadatos de ruta ausentes o incompletos en la sesión
subyacente del Gateway.

## Pruebas

OpenClaw incluye una prueba smoke determinista de Docker para este puente:

```bash
pnpm test:docker:mcp-channels
```

Esa prueba smoke:

- inicia un contenedor Gateway precargado
- inicia un segundo contenedor que lanza `openclaw mcp serve`
- verifica el descubrimiento de conversaciones, lecturas de transcripción, lecturas de metadatos de adjuntos,
  comportamiento de la cola de eventos en vivo y enrutamiento de envíos salientes
- valida notificaciones de canal y permisos al estilo Claude sobre el puente stdio MCP real

Esta es la forma más rápida de demostrar que el puente funciona sin conectar una cuenta real
de Telegram, Discord o iMessage en la ejecución de prueba.

Para un contexto de pruebas más amplio, consulta [Testing](/help/testing).

## Solución de problemas

### No se devuelven conversaciones

Normalmente significa que la sesión del Gateway aún no es enrutable. Confirma que la
sesión subyacente tenga almacenados los metadatos de ruta opcionales de canal/proveedor, destinatario y
cuenta/hilo.

### `events_poll` o `events_wait` omite mensajes antiguos

Es lo esperado. La cola en vivo comienza cuando el puente se conecta. Lee el historial anterior de la transcripción
con `messages_read`.

### Las notificaciones de Claude no aparecen

Comprueba todo lo siguiente:

- el cliente mantuvo abierta la sesión stdio MCP
- `--claude-channel-mode` es `on` o `auto`
- el cliente realmente entiende los métodos de notificación específicos de Claude
- el mensaje entrante ocurrió después de que el puente se conectara

### Faltan aprobaciones

`permissions_list_open` solo muestra solicitudes de aprobación observadas mientras el puente
estuvo conectado. No es una API de historial duradero de aprobaciones.

## OpenClaw como registro de cliente MCP

Esta es la ruta de `openclaw mcp list`, `show`, `set` y `unset`.

Estos comandos no exponen OpenClaw mediante MCP. Administran definiciones MCP propiedad de OpenClaw
bajo `mcp.servers` en la configuración de OpenClaw.

Esas definiciones guardadas son para tiempos de ejecución que OpenClaw lanza o configura
más adelante, como Pi integrado y otros adaptadores de tiempo de ejecución. OpenClaw almacena las
definiciones de forma centralizada para que esos tiempos de ejecución no necesiten mantener sus propias listas duplicadas
de servidores MCP.

Comportamiento importante:

- estos comandos solo leen o escriben la configuración de OpenClaw
- no se conectan al servidor MCP de destino
- no validan si el comando, la URL o el transporte remoto son
  accesibles en ese momento
- los adaptadores de tiempo de ejecución deciden qué formas de transporte admiten realmente en
  tiempo de ejecución

## Definiciones guardadas de servidores MCP

OpenClaw también almacena un registro ligero de servidores MCP en la configuración para las superficies
que quieran definiciones MCP administradas por OpenClaw.

Comandos:

- `openclaw mcp list`
- `openclaw mcp show [name]`
- `openclaw mcp set <name> <json>`
- `openclaw mcp unset <name>`

Notas:

- `list` ordena los nombres de los servidores.
- `show` sin nombre imprime el objeto completo configurado del servidor MCP.
- `set` espera un valor de objeto JSON en una sola línea de comandos.
- `unset` falla si el servidor indicado no existe.

Ejemplos:

```bash
openclaw mcp list
openclaw mcp show context7 --json
openclaw mcp set context7 '{"command":"uvx","args":["context7-mcp"]}'
openclaw mcp set docs '{"url":"https://mcp.example.com"}'
openclaw mcp unset context7
```

Ejemplo de forma de configuración:

```json
{
  "mcp": {
    "servers": {
      "context7": {
        "command": "uvx",
        "args": ["context7-mcp"]
      },
      "docs": {
        "url": "https://mcp.example.com"
      }
    }
  }
}
```

### Transporte stdio

Lanza un proceso hijo local y se comunica por stdin/stdout.

| Campo                      | Descripción                            |
| -------------------------- | -------------------------------------- |
| `command`                  | Ejecutable que se debe lanzar (obligatorio) |
| `args`                     | Matriz de argumentos de línea de comandos |
| `env`                      | Variables de entorno adicionales       |
| `cwd` / `workingDirectory` | Directorio de trabajo del proceso      |

### Transporte SSE / HTTP

Se conecta a un servidor MCP remoto mediante HTTP Server-Sent Events.

| Campo                 | Descripción                                                      |
| --------------------- | ---------------------------------------------------------------- |
| `url`                 | URL HTTP o HTTPS del servidor remoto (obligatorio)               |
| `headers`             | Mapa opcional clave-valor de encabezados HTTP (por ejemplo, tokens de autenticación) |
| `connectionTimeoutMs` | Tiempo de espera de conexión por servidor en ms (opcional)       |

Ejemplo:

```json
{
  "mcp": {
    "servers": {
      "remote-tools": {
        "url": "https://mcp.example.com",
        "headers": {
          "Authorization": "Bearer <token>"
        }
      }
    }
  }
}
```

Los valores sensibles en `url` (userinfo) y `headers` se redactan en los registros y
la salida de estado.

### Transporte HTTP transmitible

`streamable-http` es una opción de transporte adicional junto con `sse` y `stdio`. Usa transmisión HTTP para la comunicación bidireccional con servidores MCP remotos.

| Campo                 | Descripción                                                                            |
| --------------------- | -------------------------------------------------------------------------------------- |
| `url`                 | URL HTTP o HTTPS del servidor remoto (obligatorio)                                     |
| `transport`           | Establécelo en `"streamable-http"` para seleccionar este transporte; si se omite, OpenClaw usa `sse` |
| `headers`             | Mapa opcional clave-valor de encabezados HTTP (por ejemplo, tokens de autenticación)   |
| `connectionTimeoutMs` | Tiempo de espera de conexión por servidor en ms (opcional)                             |

Ejemplo:

```json
{
  "mcp": {
    "servers": {
      "streaming-tools": {
        "url": "https://mcp.example.com/stream",
        "transport": "streamable-http",
        "connectionTimeoutMs": 10000,
        "headers": {
          "Authorization": "Bearer <token>"
        }
      }
    }
  }
}
```

Estos comandos solo administran la configuración guardada. No inician el puente de canales,
no abren una sesión activa de cliente MCP ni demuestran que el servidor de destino sea accesible.

## Límites actuales

Esta página documenta el puente tal como se distribuye hoy.

Límites actuales:

- el descubrimiento de conversaciones depende de los metadatos de ruta de sesión existentes del Gateway
- no hay un protocolo push genérico más allá del adaptador específico de Claude
- todavía no hay herramientas para editar mensajes ni reaccionar
- el transporte HTTP/SSE/streamable-http se conecta a un único servidor remoto; aún no hay upstream multiplexado
- `permissions_list_open` solo incluye aprobaciones observadas mientras el puente está
  conectado
