---
read_when:
    - Configurar Mattermost
    - Depurar el enrutamiento de Mattermost
summary: Configuración del bot de Mattermost y de OpenClaw
title: Mattermost
x-i18n:
    generated_at: "2026-04-05T12:36:14Z"
    model: gpt-5.4
    provider: openai
    source_hash: f21dc7543176fda0b38b00fab60f0daae38dffcf68fa1cf7930a9f14ec57cb5a
    source_path: channels/mattermost.md
    workflow: 15
---

# Mattermost

Estado: plugin incluido (token de bot + eventos de WebSocket). Se admiten canales, grupos y mensajes directos.
Mattermost es una plataforma de mensajería para equipos que puede alojarse por cuenta propia; consulta el sitio oficial en
[mattermost.com](https://mattermost.com) para obtener detalles del producto y descargas.

## Plugin incluido

Mattermost se incluye como plugin integrado en las versiones actuales de OpenClaw, por lo que las compilaciones empaquetadas normales no requieren una instalación independiente.

Si usas una compilación antigua o una instalación personalizada que excluye Mattermost,
instálalo manualmente:

Instalar mediante CLI (registro npm):

```bash
openclaw plugins install @openclaw/mattermost
```

Checkout local (al ejecutar desde un repositorio git):

```bash
openclaw plugins install ./path/to/local/mattermost-plugin
```

Detalles: [Plugins](/tools/plugin)

## Configuración rápida

1. Asegúrate de que el plugin de Mattermost esté disponible.
   - Las versiones empaquetadas actuales de OpenClaw ya lo incluyen.
   - Las instalaciones antiguas o personalizadas pueden añadirlo manualmente con los comandos anteriores.
2. Crea una cuenta de bot de Mattermost y copia el **token de bot**.
3. Copia la **URL base** de Mattermost (por ejemplo, `https://chat.example.com`).
4. Configura OpenClaw e inicia el gateway.

Configuración mínima:

```json5
{
  channels: {
    mattermost: {
      enabled: true,
      botToken: "mm-token",
      baseUrl: "https://chat.example.com",
      dmPolicy: "pairing",
    },
  },
}
```

## Comandos de barra nativos

Los comandos de barra nativos son opcionales. Cuando se habilitan, OpenClaw registra comandos de barra `oc_*` mediante
la API de Mattermost y recibe POST de devolución de llamada en el servidor HTTP del gateway.

```json5
{
  channels: {
    mattermost: {
      commands: {
        native: true,
        nativeSkills: true,
        callbackPath: "/api/channels/mattermost/command",
        // Use when Mattermost cannot reach the gateway directly (reverse proxy/public URL).
        callbackUrl: "https://gateway.example.com/api/channels/mattermost/command",
      },
    },
  },
}
```

Notas:

- `native: "auto"` está deshabilitado de forma predeterminada para Mattermost. Configura `native: true` para habilitarlo.
- Si se omite `callbackUrl`, OpenClaw deriva una a partir del host/puerto del gateway y `callbackPath`.
- En configuraciones con varias cuentas, `commands` puede definirse en el nivel superior o en
  `channels.mattermost.accounts.<id>.commands` (los valores de la cuenta prevalecen sobre los campos de nivel superior).
- Las devoluciones de llamada de comandos se validan con los tokens por comando devueltos por
  Mattermost cuando OpenClaw registra comandos `oc_*`.
- Las devoluciones de llamada de comandos de barra fallan de forma segura cuando falla el registro, el inicio es parcial o
  el token de devolución de llamada no coincide con uno de los comandos registrados.
- Requisito de accesibilidad: el endpoint de devolución de llamada debe ser accesible desde el servidor de Mattermost.
  - No configures `callbackUrl` como `localhost` a menos que Mattermost se ejecute en el mismo host o espacio de nombres de red que OpenClaw.
  - No configures `callbackUrl` con tu URL base de Mattermost a menos que esa URL haga proxy inverso de `/api/channels/mattermost/command` hacia OpenClaw.
  - Una comprobación rápida es `curl https://<gateway-host>/api/channels/mattermost/command`; un GET debería devolver `405 Method Not Allowed` desde OpenClaw, no `404`.
- Requisito de lista de permitidos de salida de Mattermost:
  - Si tu devolución de llamada apunta a direcciones privadas, de tailnet o internas, configura
    `ServiceSettings.AllowedUntrustedInternalConnections` de Mattermost para incluir el host/dominio de la devolución de llamada.
  - Usa entradas de host/dominio, no URL completas.
    - Correcto: `gateway.tailnet-name.ts.net`
    - Incorrecto: `https://gateway.tailnet-name.ts.net`

## Variables de entorno (cuenta predeterminada)

Configúralas en el host del gateway si prefieres usar variables de entorno:

- `MATTERMOST_BOT_TOKEN=...`
- `MATTERMOST_URL=https://chat.example.com`

Las variables de entorno solo se aplican a la cuenta **predeterminada** (`default`). Las demás cuentas deben usar valores de configuración.

## Modos de chat

Mattermost responde automáticamente a los mensajes directos. El comportamiento en canales se controla con `chatmode`:

- `oncall` (predeterminado): responde solo cuando se le menciona con @ en canales.
- `onmessage`: responde a cada mensaje del canal.
- `onchar`: responde cuando un mensaje comienza con un prefijo de activación.

Ejemplo de configuración:

```json5
{
  channels: {
    mattermost: {
      chatmode: "onchar",
      oncharPrefixes: [">", "!"],
    },
  },
}
```

Notas:

- `onchar` sigue respondiendo a menciones explícitas con @.
- `channels.mattermost.requireMention` se respeta para configuraciones heredadas, pero se prefiere `chatmode`.

## Hilos y sesiones

Usa `channels.mattermost.replyToMode` para controlar si las respuestas de canales y grupos permanecen en el
canal principal o inician un hilo bajo la publicación que las activó.

- `off` (predeterminado): solo responde en un hilo cuando la publicación entrante ya está en uno.
- `first`: para publicaciones de canal o grupo de nivel superior, inicia un hilo bajo esa publicación y enruta la
  conversación a una sesión con ámbito de hilo.
- `all`: mismo comportamiento que `first` en Mattermost actualmente.
- Los mensajes directos ignoran esta configuración y siguen sin hilos.

Ejemplo de configuración:

```json5
{
  channels: {
    mattermost: {
      replyToMode: "all",
    },
  },
}
```

Notas:

- Las sesiones con ámbito de hilo usan el id de la publicación activadora como raíz del hilo.
- `first` y `all` son actualmente equivalentes porque, una vez que Mattermost tiene una raíz de hilo,
  los fragmentos y medios de seguimiento continúan en ese mismo hilo.

## Control de acceso (mensajes directos)

- Predeterminado: `channels.mattermost.dmPolicy = "pairing"` (los remitentes desconocidos reciben un código de emparejamiento).
- Aprobar mediante:
  - `openclaw pairing list mattermost`
  - `openclaw pairing approve mattermost <CODE>`
- Mensajes directos públicos: `channels.mattermost.dmPolicy="open"` más `channels.mattermost.allowFrom=["*"]`.

## Canales (grupos)

- Predeterminado: `channels.mattermost.groupPolicy = "allowlist"` (con restricción por mención).
- Define remitentes permitidos con `channels.mattermost.groupAllowFrom` (se recomiendan IDs de usuario).
- Las anulaciones de mención por canal están en `channels.mattermost.groups.<channelId>.requireMention`
  o `channels.mattermost.groups["*"].requireMention` como valor predeterminado.
- La coincidencia `@username` es mutable y solo se habilita cuando `channels.mattermost.dangerouslyAllowNameMatching: true`.
- Canales abiertos: `channels.mattermost.groupPolicy="open"` (con restricción por mención).
- Nota de tiempo de ejecución: si falta por completo `channels.mattermost`, el tiempo de ejecución usa `groupPolicy="allowlist"` para las comprobaciones de grupo (incluso si `channels.defaults.groupPolicy` está configurado).

Ejemplo:

```json5
{
  channels: {
    mattermost: {
      groupPolicy: "open",
      groups: {
        "*": { requireMention: true },
        "team-channel-id": { requireMention: false },
      },
    },
  },
}
```

## Destinos para entrega saliente

Usa estos formatos de destino con `openclaw message send` o cron/webhooks:

- `channel:<id>` para un canal
- `user:<id>` para un mensaje directo
- `@username` para un mensaje directo (resuelto mediante la API de Mattermost)

Los IDs opacos simples (como `64ifufp...`) son **ambiguos** en Mattermost (ID de usuario frente a ID de canal).

OpenClaw los resuelve **priorizando el usuario**:

- Si el ID existe como usuario (`GET /api/v4/users/<id>` devuelve éxito), OpenClaw envía un **mensaje directo** resolviendo el canal directo mediante `/api/v4/channels/direct`.
- En caso contrario, el ID se trata como un **ID de canal**.

Si necesitas un comportamiento determinista, usa siempre los prefijos explícitos (`user:<id>` / `channel:<id>`).

## Reintento de canal de mensaje directo

Cuando OpenClaw envía a un destino de mensaje directo de Mattermost y necesita resolver primero el canal directo,
reintenta por defecto los fallos transitorios al crear ese canal directo.

Usa `channels.mattermost.dmChannelRetry` para ajustar ese comportamiento globalmente para el plugin de Mattermost,
o `channels.mattermost.accounts.<id>.dmChannelRetry` para una cuenta concreta.

```json5
{
  channels: {
    mattermost: {
      dmChannelRetry: {
        maxRetries: 3,
        initialDelayMs: 1000,
        maxDelayMs: 10000,
        timeoutMs: 30000,
      },
    },
  },
}
```

Notas:

- Esto solo se aplica a la creación de canales de mensajes directos (`/api/v4/channels/direct`), no a todas las llamadas a la API de Mattermost.
- Los reintentos se aplican a fallos transitorios como límites de tasa, respuestas 5xx y errores de red o tiempo de espera.
- Los errores de cliente 4xx distintos de `429` se consideran permanentes y no se reintentan.

## Reacciones (herramienta de mensajes)

- Usa `message action=react` con `channel=mattermost`.
- `messageId` es el id de la publicación de Mattermost.
- `emoji` acepta nombres como `thumbsup` o `:+1:` (los dos puntos son opcionales).
- Configura `remove=true` (booleano) para eliminar una reacción.
- Los eventos de añadir o quitar reacciones se reenvían como eventos del sistema a la sesión del agente enrutada.

Ejemplos:

```
message action=react channel=mattermost target=channel:<channelId> messageId=<postId> emoji=thumbsup
message action=react channel=mattermost target=channel:<channelId> messageId=<postId> emoji=thumbsup remove=true
```

Configuración:

- `channels.mattermost.actions.reactions`: habilita o deshabilita acciones de reacción (predeterminado: true).
- Anulación por cuenta: `channels.mattermost.accounts.<id>.actions.reactions`.

## Botones interactivos (herramienta de mensajes)

Envía mensajes con botones en los que se puede hacer clic. Cuando un usuario hace clic en un botón, el agente recibe la
selección y puede responder.

Habilita los botones añadiendo `inlineButtons` a las capacidades del canal:

```json5
{
  channels: {
    mattermost: {
      capabilities: ["inlineButtons"],
    },
  },
}
```

Usa `message action=send` con un parámetro `buttons`. Los botones son un arreglo 2D (filas de botones):

```
message action=send channel=mattermost target=channel:<channelId> buttons=[[{"text":"Yes","callback_data":"yes"},{"text":"No","callback_data":"no"}]]
```

Campos de los botones:

- `text` (obligatorio): etiqueta mostrada.
- `callback_data` (obligatorio): valor que se devuelve al hacer clic (se usa como ID de acción).
- `style` (opcional): `"default"`, `"primary"` o `"danger"`.

Cuando un usuario hace clic en un botón:

1. Todos los botones se sustituyen por una línea de confirmación (por ejemplo, "✓ **Yes** selected by @user").
2. El agente recibe la selección como mensaje entrante y responde.

Notas:

- Las devoluciones de llamada de botones usan verificación HMAC-SHA256 (automática, sin configuración necesaria).
- Mattermost elimina los datos de devolución de llamada de sus respuestas de la API (función de seguridad), por lo que todos los botones
  se eliminan al hacer clic; no es posible una eliminación parcial.
- Los IDs de acción que contienen guiones o guiones bajos se sanean automáticamente
  (limitación del enrutamiento de Mattermost).

Configuración:

- `channels.mattermost.capabilities`: arreglo de cadenas de capacidad. Añade `"inlineButtons"` para
  habilitar la descripción de la herramienta de botones en el prompt del sistema del agente.
- `channels.mattermost.interactions.callbackBaseUrl`: URL base externa opcional para las devoluciones de llamada
  de botones (por ejemplo `https://gateway.example.com`). Úsala cuando Mattermost no pueda
  llegar al gateway directamente en su host de enlace.
- En configuraciones con varias cuentas, también puedes definir el mismo campo en
  `channels.mattermost.accounts.<id>.interactions.callbackBaseUrl`.
- Si se omite `interactions.callbackBaseUrl`, OpenClaw deriva la URL de devolución de llamada de
  `gateway.customBindHost` + `gateway.port`, y luego usa `http://localhost:<port>` como reserva.
- Regla de accesibilidad: la URL de devolución de llamada del botón debe ser accesible desde el servidor de Mattermost.
  `localhost` solo funciona cuando Mattermost y OpenClaw se ejecutan en el mismo host o espacio de nombres de red.
- Si tu destino de devolución de llamada es privado, de tailnet o interno, añade su host o dominio a
  `ServiceSettings.AllowedUntrustedInternalConnections` de Mattermost.

### Integración directa con la API (scripts externos)

Los scripts externos y webhooks pueden publicar botones directamente mediante la API REST de Mattermost
en lugar de pasar por la herramienta `message` del agente. Usa `buildButtonAttachments()` de
la extensión siempre que sea posible; si publicas JSON sin procesar, sigue estas reglas:

**Estructura de la carga útil:**

```json5
{
  channel_id: "<channelId>",
  message: "Choose an option:",
  props: {
    attachments: [
      {
        actions: [
          {
            id: "mybutton01", // alphanumeric only — see below
            type: "button", // required, or clicks are silently ignored
            name: "Approve", // display label
            style: "primary", // optional: "default", "primary", "danger"
            integration: {
              url: "https://gateway.example.com/mattermost/interactions/default",
              context: {
                action_id: "mybutton01", // must match button id (for name lookup)
                action: "approve",
                // ... any custom fields ...
                _token: "<hmac>", // see HMAC section below
              },
            },
          },
        ],
      },
    ],
  },
}
```

**Reglas críticas:**

1. Los adjuntos van en `props.attachments`, no en `attachments` de nivel superior (se ignoran silenciosamente).
2. Cada acción necesita `type: "button"`; sin ello, los clics se descartan silenciosamente.
3. Cada acción necesita un campo `id`; Mattermost ignora las acciones sin ID.
4. El `id` de la acción debe ser **solo alfanumérico** (`[a-zA-Z0-9]`). Los guiones y guiones bajos rompen
   el enrutamiento de acciones del lado del servidor de Mattermost (devuelve 404). Elimínalos antes de usarlo.
5. `context.action_id` debe coincidir con el `id` del botón para que el mensaje de confirmación muestre el
   nombre del botón (por ejemplo, "Approve") en lugar de un ID sin procesar.
6. `context.action_id` es obligatorio; el controlador de interacciones devuelve 400 sin él.

**Generación del token HMAC:**

El gateway verifica los clics de botones con HMAC-SHA256. Los scripts externos deben generar tokens
que coincidan con la lógica de verificación del gateway:

1. Deriva el secreto a partir del token del bot:
   `HMAC-SHA256(key="openclaw-mattermost-interactions", data=botToken)`
2. Construye el objeto de contexto con todos los campos **excepto** `_token`.
3. Serializa con **claves ordenadas** y **sin espacios** (el gateway usa `JSON.stringify`
   con claves ordenadas, lo que produce una salida compacta).
4. Firma: `HMAC-SHA256(key=secret, data=serializedContext)`
5. Añade el resumen hexadecimal resultante como `_token` en el contexto.

Ejemplo en Python:

```python
import hmac, hashlib, json

secret = hmac.new(
    b"openclaw-mattermost-interactions",
    bot_token.encode(), hashlib.sha256
).hexdigest()

ctx = {"action_id": "mybutton01", "action": "approve"}
payload = json.dumps(ctx, sort_keys=True, separators=(",", ":"))
token = hmac.new(secret.encode(), payload.encode(), hashlib.sha256).hexdigest()

context = {**ctx, "_token": token}
```

Errores habituales con HMAC:

- `json.dumps` de Python añade espacios de forma predeterminada (`{"key": "val"}`). Usa
  `separators=(",", ":")` para coincidir con la salida compacta de JavaScript (`{"key":"val"}`).
- Firma siempre **todos** los campos del contexto (menos `_token`). El gateway elimina `_token` y luego
  firma todo lo restante. Firmar un subconjunto provoca un fallo silencioso de la verificación.
- Usa `sort_keys=True`; el gateway ordena las claves antes de firmar, y Mattermost puede
  reordenar los campos del contexto al almacenar la carga útil.
- Deriva el secreto a partir del token del bot (determinista), no de bytes aleatorios. El secreto
  debe ser el mismo en el proceso que crea los botones y en el gateway que verifica.

## Adaptador de directorio

El plugin de Mattermost incluye un adaptador de directorio que resuelve nombres de canales y usuarios
mediante la API de Mattermost. Esto permite usar destinos `#channel-name` y `@username` en
`openclaw message send` y entregas por cron/webhook.

No se necesita configuración: el adaptador usa el token del bot de la configuración de la cuenta.

## Varias cuentas

Mattermost admite varias cuentas en `channels.mattermost.accounts`:

```json5
{
  channels: {
    mattermost: {
      accounts: {
        default: { name: "Primary", botToken: "mm-token", baseUrl: "https://chat.example.com" },
        alerts: { name: "Alerts", botToken: "mm-token-2", baseUrl: "https://alerts.example.com" },
      },
    },
  },
}
```

## Solución de problemas

- No hay respuestas en canales: asegúrate de que el bot esté en el canal y menciónalo (oncall), usa un prefijo de activación (onchar) o configura `chatmode: "onmessage"`.
- Errores de autenticación: comprueba el token del bot, la URL base y si la cuenta está habilitada.
- Problemas con varias cuentas: las variables de entorno solo se aplican a la cuenta `default`.
- Los comandos de barra nativos devuelven `Unauthorized: invalid command token.`: OpenClaw
  no aceptó el token de devolución de llamada. Las causas típicas son:
  - el registro del comando de barra falló o solo se completó parcialmente al iniciar
  - la devolución de llamada está llegando al gateway o a la cuenta equivocados
  - Mattermost todavía tiene comandos antiguos que apuntan a un destino de devolución de llamada anterior
  - el gateway se reinició sin reactivar los comandos de barra
- Si los comandos de barra nativos dejan de funcionar, revisa los registros en busca de
  `mattermost: failed to register slash commands` o
  `mattermost: native slash commands enabled but no commands could be registered`.
- Si se omite `callbackUrl` y los registros advierten que la devolución de llamada se resolvió en
  `http://127.0.0.1:18789/...`, esa URL probablemente solo sea accesible cuando
  Mattermost se ejecuta en el mismo host o espacio de nombres de red que OpenClaw. Configura en su lugar una
  `commands.callbackUrl` explícita y accesible externamente.
- Los botones aparecen como cuadros blancos: es posible que el agente esté enviando datos de botones mal formados. Comprueba que cada botón tenga los campos `text` y `callback_data`.
- Los botones se muestran pero los clics no hacen nada: verifica que `AllowedUntrustedInternalConnections` en la configuración del servidor Mattermost incluya `127.0.0.1 localhost`, y que `EnablePostActionIntegration` sea `true` en `ServiceSettings`.
- Los botones devuelven 404 al hacer clic: el `id` del botón probablemente contiene guiones o guiones bajos. El enrutador de acciones de Mattermost se rompe con IDs no alfanuméricos. Usa solo `[a-zA-Z0-9]`.
- Los registros del gateway muestran `invalid _token`: discrepancia de HMAC. Comprueba que firmas todos los campos del contexto (no un subconjunto), que usas claves ordenadas y JSON compacto (sin espacios). Consulta la sección de HMAC anterior.
- Los registros del gateway muestran `missing _token in context`: el campo `_token` no está en el contexto del botón. Asegúrate de incluirlo al construir la carga útil de integración.
- La confirmación muestra el ID sin procesar en lugar del nombre del botón: `context.action_id` no coincide con el `id` del botón. Configura ambos con el mismo valor saneado.
- El agente no conoce los botones: añade `capabilities: ["inlineButtons"]` a la configuración del canal de Mattermost.

## Relacionado

- [Resumen de canales](/channels) — todos los canales compatibles
- [Emparejamiento](/channels/pairing) — autenticación de mensajes directos y flujo de emparejamiento
- [Grupos](/channels/groups) — comportamiento del chat grupal y restricción por mención
- [Enrutamiento de canales](/channels/channel-routing) — enrutamiento de sesiones para mensajes
- [Seguridad](/gateway/security) — modelo de acceso y refuerzo
