---
read_when:
    - Configurando la integración del chat de Twitch para OpenClaw
summary: Configuración y puesta en marcha del bot de chat de Twitch
title: Twitch
x-i18n:
    generated_at: "2026-04-05T12:36:58Z"
    model: gpt-5.4
    provider: openai
    source_hash: 47af9fb6edb1f462c5919850ee9d05e500a1914ddd0d64a41608fbe960e77cd6
    source_path: channels/twitch.md
    workflow: 15
---

# Twitch

Compatibilidad con el chat de Twitch mediante conexión IRC. OpenClaw se conecta como un usuario de Twitch (cuenta de bot) para recibir y enviar mensajes en canales.

## Plugin incluido

Twitch se incluye como plugin integrado en las versiones actuales de OpenClaw, por lo que las compilaciones empaquetadas normales no necesitan una instalación separada.

Si estás usando una compilación antigua o una instalación personalizada que excluye Twitch, instálalo manualmente:

Instalar mediante la CLI (registro npm):

```bash
openclaw plugins install @openclaw/twitch
```

Checkout local (al ejecutarlo desde un repositorio git):

```bash
openclaw plugins install ./path/to/local/twitch-plugin
```

Detalles: [Plugins](/tools/plugin)

## Configuración rápida (principiante)

1. Asegúrate de que el plugin de Twitch esté disponible.
   - Las versiones empaquetadas actuales de OpenClaw ya lo incluyen.
   - Las instalaciones antiguas/personalizadas pueden agregarlo manualmente con los comandos anteriores.
2. Crea una cuenta de Twitch dedicada para el bot (o usa una cuenta existente).
3. Genera credenciales: [Twitch Token Generator](https://twitchtokengenerator.com/)
   - Selecciona **Bot Token**
   - Verifica que estén seleccionados los permisos `chat:read` y `chat:write`
   - Copia el **Client ID** y el **Access Token**
4. Encuentra tu ID de usuario de Twitch: [https://www.streamweasels.com/tools/convert-twitch-username-to-user-id/](https://www.streamweasels.com/tools/convert-twitch-username-to-user-id/)
5. Configura el token:
   - Variable de entorno: `OPENCLAW_TWITCH_ACCESS_TOKEN=...` (solo cuenta predeterminada)
   - O configuración: `channels.twitch.accessToken`
   - Si ambos están configurados, la configuración tiene prioridad (la variable de entorno como respaldo es solo para la cuenta predeterminada).
6. Inicia el gateway.

**⚠️ Importante:** agrega control de acceso (`allowFrom` o `allowedRoles`) para impedir que usuarios no autorizados activen el bot. `requireMention` tiene como valor predeterminado `true`.

Configuración mínima:

```json5
{
  channels: {
    twitch: {
      enabled: true,
      username: "openclaw", // Cuenta de Twitch del bot
      accessToken: "oauth:abc123...", // OAuth Access Token (o usa la variable de entorno OPENCLAW_TWITCH_ACCESS_TOKEN)
      clientId: "xyz789...", // Client ID de Token Generator
      channel: "vevisk", // A qué canal de chat de Twitch unirse (obligatorio)
      allowFrom: ["123456789"], // (recomendado) Solo tu ID de usuario de Twitch - obténlo en https://www.streamweasels.com/tools/convert-twitch-username-to-user-id/
    },
  },
}
```

## Qué es

- Un canal de Twitch gestionado por el Gateway.
- Enrutamiento determinista: las respuestas siempre vuelven a Twitch.
- Cada cuenta se asigna a una clave de sesión aislada `agent:<agentId>:twitch:<accountName>`.
- `username` es la cuenta del bot (la que se autentica), `channel` es la sala de chat a la que se une.

## Configuración (detallada)

### Generar credenciales

Usa [Twitch Token Generator](https://twitchtokengenerator.com/):

- Selecciona **Bot Token**
- Verifica que estén seleccionados los permisos `chat:read` y `chat:write`
- Copia el **Client ID** y el **Access Token**

No se requiere registro manual de la app. Los tokens caducan después de varias horas.

### Configurar el bot

**Variable de entorno (solo cuenta predeterminada):**

```bash
OPENCLAW_TWITCH_ACCESS_TOKEN=oauth:abc123...
```

**O configuración:**

```json5
{
  channels: {
    twitch: {
      enabled: true,
      username: "openclaw",
      accessToken: "oauth:abc123...",
      clientId: "xyz789...",
      channel: "vevisk",
    },
  },
}
```

Si tanto la variable de entorno como la configuración están definidas, la configuración tiene prioridad.

### Control de acceso (recomendado)

```json5
{
  channels: {
    twitch: {
      allowFrom: ["123456789"], // (recomendado) Solo tu ID de usuario de Twitch
    },
  },
}
```

Prefiere `allowFrom` para una allowlist estricta. Usa `allowedRoles` en su lugar si quieres control de acceso basado en roles.

**Roles disponibles:** `"moderator"`, `"owner"`, `"vip"`, `"subscriber"`, `"all"`.

**¿Por qué ID de usuario?** Los nombres de usuario pueden cambiar, lo que permite la suplantación. Los ID de usuario son permanentes.

Encuentra tu ID de usuario de Twitch: [https://www.streamweasels.com/tools/convert-twitch-username-to-user-id/](https://www.streamweasels.com/tools/convert-twitch-username-to-user-id/) (Convierte tu nombre de usuario de Twitch en ID)

## Renovación de token (opcional)

Los tokens de [Twitch Token Generator](https://twitchtokengenerator.com/) no se pueden renovar automáticamente; regénéralos cuando caduquen.

Para la renovación automática de tokens, crea tu propia aplicación de Twitch en [Twitch Developer Console](https://dev.twitch.tv/console) y agrega a la configuración:

```json5
{
  channels: {
    twitch: {
      clientSecret: "your_client_secret",
      refreshToken: "your_refresh_token",
    },
  },
}
```

El bot renueva automáticamente los tokens antes de que caduquen y registra los eventos de renovación.

## Compatibilidad con varias cuentas

Usa `channels.twitch.accounts` con tokens por cuenta. Consulta [`gateway/configuration`](/gateway/configuration) para ver el patrón compartido.

Ejemplo (una cuenta de bot en dos canales):

```json5
{
  channels: {
    twitch: {
      accounts: {
        channel1: {
          username: "openclaw",
          accessToken: "oauth:abc123...",
          clientId: "xyz789...",
          channel: "vevisk",
        },
        channel2: {
          username: "openclaw",
          accessToken: "oauth:def456...",
          clientId: "uvw012...",
          channel: "secondchannel",
        },
      },
    },
  },
}
```

**Nota:** cada cuenta necesita su propio token (un token por canal).

## Control de acceso

### Restricciones basadas en roles

```json5
{
  channels: {
    twitch: {
      accounts: {
        default: {
          allowedRoles: ["moderator", "vip"],
        },
      },
    },
  },
}
```

### Allowlist por ID de usuario (más segura)

```json5
{
  channels: {
    twitch: {
      accounts: {
        default: {
          allowFrom: ["123456789", "987654321"],
        },
      },
    },
  },
}
```

### Acceso basado en roles (alternativa)

`allowFrom` es una allowlist estricta. Cuando está configurada, solo se permiten esos ID de usuario.
Si quieres acceso basado en roles, deja `allowFrom` sin configurar y usa `allowedRoles` en su lugar:

```json5
{
  channels: {
    twitch: {
      accounts: {
        default: {
          allowedRoles: ["moderator"],
        },
      },
    },
  },
}
```

### Deshabilitar el requisito de @mention

De forma predeterminada, `requireMention` es `true`. Para deshabilitarlo y responder a todos los mensajes:

```json5
{
  channels: {
    twitch: {
      accounts: {
        default: {
          requireMention: false,
        },
      },
    },
  },
}
```

## Solución de problemas

Primero, ejecuta los comandos de diagnóstico:

```bash
openclaw doctor
openclaw channels status --probe
```

### El bot no responde a los mensajes

**Comprueba el control de acceso:** asegúrate de que tu ID de usuario esté en `allowFrom`, o elimina temporalmente `allowFrom` y configura `allowedRoles: ["all"]` para hacer una prueba.

**Comprueba que el bot esté en el canal:** el bot debe unirse al canal especificado en `channel`.

### Problemas con el token

**"Failed to connect" o errores de autenticación:**

- Verifica que `accessToken` sea el valor del token de acceso OAuth (normalmente comienza con el prefijo `oauth:`)
- Comprueba que el token tenga los permisos `chat:read` y `chat:write`
- Si usas renovación de token, verifica que `clientSecret` y `refreshToken` estén configurados

### La renovación del token no funciona

**Comprueba los logs para ver eventos de renovación:**

```
Using env token source for mybot
Access token refreshed for user 123456 (expires in 14400s)
```

Si ves "token refresh disabled (no refresh token)":

- Asegúrate de que `clientSecret` esté proporcionado
- Asegúrate de que `refreshToken` esté proporcionado

## Configuración

**Configuración de cuenta:**

- `username` - Nombre de usuario del bot
- `accessToken` - Token de acceso OAuth con `chat:read` y `chat:write`
- `clientId` - Twitch Client ID (de Token Generator o de tu app)
- `channel` - Canal al que unirse (obligatorio)
- `enabled` - Habilitar esta cuenta (predeterminado: `true`)
- `clientSecret` - Opcional: para la renovación automática del token
- `refreshToken` - Opcional: para la renovación automática del token
- `expiresIn` - Caducidad del token en segundos
- `obtainmentTimestamp` - Marca de tiempo de obtención del token
- `allowFrom` - Allowlist de ID de usuario
- `allowedRoles` - Control de acceso basado en roles (`"moderator" | "owner" | "vip" | "subscriber" | "all"`)
- `requireMention` - Requerir @mention (predeterminado: `true`)

**Opciones del proveedor:**

- `channels.twitch.enabled` - Habilitar/deshabilitar el inicio del canal
- `channels.twitch.username` - Nombre de usuario del bot (configuración simplificada de cuenta única)
- `channels.twitch.accessToken` - Token de acceso OAuth (configuración simplificada de cuenta única)
- `channels.twitch.clientId` - Twitch Client ID (configuración simplificada de cuenta única)
- `channels.twitch.channel` - Canal al que unirse (configuración simplificada de cuenta única)
- `channels.twitch.accounts.<accountName>` - Configuración multicuenta (todos los campos de cuenta anteriores)

Ejemplo completo:

```json5
{
  channels: {
    twitch: {
      enabled: true,
      username: "openclaw",
      accessToken: "oauth:abc123...",
      clientId: "xyz789...",
      channel: "vevisk",
      clientSecret: "secret123...",
      refreshToken: "refresh456...",
      allowFrom: ["123456789"],
      allowedRoles: ["moderator", "vip"],
      accounts: {
        default: {
          username: "mybot",
          accessToken: "oauth:abc123...",
          clientId: "xyz789...",
          channel: "your_channel",
          enabled: true,
          clientSecret: "secret123...",
          refreshToken: "refresh456...",
          expiresIn: 14400,
          obtainmentTimestamp: 1706092800000,
          allowFrom: ["123456789", "987654321"],
          allowedRoles: ["moderator"],
        },
      },
    },
  },
}
```

## Acciones de herramientas

El agente puede llamar a `twitch` con la acción:

- `send` - Enviar un mensaje a un canal

Ejemplo:

```json5
{
  action: "twitch",
  params: {
    message: "Hello Twitch!",
    to: "#mychannel",
  },
}
```

## Seguridad y operaciones

- **Trata los tokens como contraseñas** - Nunca hagas commit de tokens en git
- **Usa renovación automática de tokens** para bots de larga duración
- **Usa allowlists de ID de usuario** en lugar de nombres de usuario para el control de acceso
- **Supervisa los logs** para ver eventos de renovación de token y estado de conexión
- **Limita al mínimo el alcance de los tokens** - Solicita solo `chat:read` y `chat:write`
- **Si te bloqueas**: reinicia el gateway después de confirmar que ningún otro proceso posee la sesión

## Límites

- **500 caracteres** por mensaje (fragmentación automática en límites de palabras)
- El markdown se elimina antes de fragmentar
- Sin limitación de velocidad (usa los límites de velocidad integrados de Twitch)

## Relacionado

- [Channels Overview](/channels) — todos los canales compatibles
- [Pairing](/channels/pairing) — autenticación de mensajes directos y flujo de pairing
- [Groups](/channels/groups) — comportamiento de chats grupales y control por menciones
- [Channel Routing](/channels/channel-routing) — enrutamiento de sesiones para mensajes
- [Security](/gateway/security) — modelo de acceso y endurecimiento
