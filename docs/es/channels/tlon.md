---
read_when:
    - Trabajas en funcionalidades del canal Tlon/Urbit
summary: Estado de compatibilidad, capacidades y configuración de Tlon/Urbit
title: Tlon
x-i18n:
    generated_at: "2026-04-05T12:36:40Z"
    model: gpt-5.4
    provider: openai
    source_hash: 289cffb3c1b2d450a5f41e0d67117dfb5c192cec956d82039caac9df9f07496d
    source_path: channels/tlon.md
    workflow: 15
---

# Tlon

Tlon es un mensajero descentralizado basado en Urbit. OpenClaw se conecta a tu nave de Urbit y puede
responder a mensajes directos y mensajes de chat grupal. Las respuestas en grupos requieren una mención con @ de forma predeterminada y pueden
restringirse aún más mediante listas de permitidos.

Estado: plugin incluido. Se admiten mensajes directos, menciones en grupos, respuestas en hilos, formato de texto enriquecido y
carga de imágenes. Las reacciones y las encuestas aún no son compatibles.

## Plugin incluido

Tlon se incluye como plugin integrado en las versiones actuales de OpenClaw, por lo que las
compilaciones empaquetadas normales no necesitan una instalación independiente.

Si usas una compilación antigua o una instalación personalizada que excluye Tlon, instálalo
manualmente:

Instalar mediante CLI (registro npm):

```bash
openclaw plugins install @openclaw/tlon
```

Checkout local (al ejecutar desde un repositorio git):

```bash
openclaw plugins install ./path/to/local/tlon-plugin
```

Detalles: [Plugins](/tools/plugin)

## Configuración

1. Asegúrate de que el plugin de Tlon esté disponible.
   - Las versiones empaquetadas actuales de OpenClaw ya lo incluyen.
   - Las instalaciones antiguas o personalizadas pueden añadirlo manualmente con los comandos anteriores.
2. Reúne la URL de tu nave y el código de inicio de sesión.
3. Configura `channels.tlon`.
4. Reinicia el gateway.
5. Envía un mensaje directo al bot o menciónalo en un canal grupal.

Configuración mínima (una sola cuenta):

```json5
{
  channels: {
    tlon: {
      enabled: true,
      ship: "~sampel-palnet",
      url: "https://your-ship-host",
      code: "lidlut-tabwed-pillex-ridrup",
      ownerShip: "~your-main-ship", // recommended: your ship, always allowed
    },
  },
}
```

## Naves privadas/LAN

De forma predeterminada, OpenClaw bloquea nombres de host privados o internos y rangos de IP para protección contra SSRF.
Si tu nave se ejecuta en una red privada (localhost, IP LAN o nombre de host interno),
debes habilitarlo explícitamente:

```json5
{
  channels: {
    tlon: {
      url: "http://localhost:8080",
      allowPrivateNetwork: true,
    },
  },
}
```

Esto se aplica a URL como:

- `http://localhost:8080`
- `http://192.168.x.x:8080`
- `http://my-ship.local:8080`

⚠️ Habilítalo solo si confías en tu red local. Esta configuración desactiva las protecciones SSRF
para las solicitudes a la URL de tu nave.

## Canales de grupo

La detección automática está habilitada de forma predeterminada. También puedes fijar canales manualmente:

```json5
{
  channels: {
    tlon: {
      groupChannels: ["chat/~host-ship/general", "chat/~host-ship/support"],
    },
  },
}
```

Desactivar la detección automática:

```json5
{
  channels: {
    tlon: {
      autoDiscoverChannels: false,
    },
  },
}
```

## Control de acceso

Lista de permitidos de mensajes directos (vacía = no se permiten mensajes directos; usa `ownerShip` para el flujo de aprobación):

```json5
{
  channels: {
    tlon: {
      dmAllowlist: ["~zod", "~nec"],
    },
  },
}
```

Autorización de grupos (restringida de forma predeterminada):

```json5
{
  channels: {
    tlon: {
      defaultAuthorizedShips: ["~zod"],
      authorization: {
        channelRules: {
          "chat/~host-ship/general": {
            mode: "restricted",
            allowedShips: ["~zod", "~nec"],
          },
          "chat/~host-ship/announcements": {
            mode: "open",
          },
        },
      },
    },
  },
}
```

## Propietario y sistema de aprobación

Configura una nave propietaria para recibir solicitudes de aprobación cuando usuarios no autorizados intenten interactuar:

```json5
{
  channels: {
    tlon: {
      ownerShip: "~your-main-ship",
    },
  },
}
```

La nave propietaria está **autorizada automáticamente en todas partes**: las invitaciones a mensajes directos se aceptan automáticamente y
los mensajes de canal siempre están permitidos. No necesitas añadir al propietario a `dmAllowlist` ni a
`defaultAuthorizedShips`.

Cuando está configurada, el propietario recibe notificaciones por mensaje directo para:

- Solicitudes de mensajes directos de naves que no están en la lista de permitidos
- Menciones en canales sin autorización
- Solicitudes de invitación a grupos

## Configuración de aceptación automática

Aceptar automáticamente invitaciones a mensajes directos (para naves en `dmAllowlist`):

```json5
{
  channels: {
    tlon: {
      autoAcceptDmInvites: true,
    },
  },
}
```

Aceptar automáticamente invitaciones a grupos:

```json5
{
  channels: {
    tlon: {
      autoAcceptGroupInvites: true,
    },
  },
}
```

## Destinos de entrega (CLI/cron)

Usa estos con `openclaw message send` o entrega por cron:

- Mensaje directo: `~sampel-palnet` o `dm/~sampel-palnet`
- Grupo: `chat/~host-ship/channel` o `group:~host-ship/channel`

## Skills incluido

El plugin de Tlon incluye un Skills integrado ([`@tloncorp/tlon-skill`](https://github.com/tloncorp/tlon-skill))
que proporciona acceso por CLI a operaciones de Tlon:

- **Contactos**: obtener o actualizar perfiles, listar contactos
- **Canales**: listar, crear, publicar mensajes, recuperar historial
- **Grupos**: listar, crear, gestionar miembros
- **Mensajes directos**: enviar mensajes, reaccionar a mensajes
- **Reacciones**: añadir o eliminar reacciones emoji a publicaciones y mensajes directos
- **Configuración**: gestionar permisos del plugin mediante comandos de barra

El Skills está disponible automáticamente cuando el plugin está instalado.

## Capacidades

| Función         | Estado                                  |
| --------------- | --------------------------------------- |
| Mensajes directos | ✅ Compatible                          |
| Grupos/canales  | ✅ Compatible (con restricción por mención de forma predeterminada) |
| Hilos           | ✅ Compatible (respuestas automáticas en el hilo) |
| Texto enriquecido | ✅ Markdown convertido al formato de Tlon |
| Imágenes        | ✅ Cargadas al almacenamiento de Tlon   |
| Reacciones      | ✅ Mediante el [Skills integrado](#skills-integrado) |
| Encuestas       | ❌ Aún no compatible                    |
| Comandos nativos | ✅ Compatibles (solo propietario de forma predeterminada) |

## Solución de problemas

Ejecuta primero esta secuencia:

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
```

Fallos comunes:

- **Mensajes directos ignorados**: el remitente no está en `dmAllowlist` y no hay `ownerShip` configurado para el flujo de aprobación.
- **Mensajes de grupo ignorados**: el canal no se detectó o el remitente no está autorizado.
- **Errores de conexión**: comprueba que se pueda acceder a la URL de la nave; habilita `allowPrivateNetwork` para naves locales.
- **Errores de autenticación**: verifica que el código de inicio de sesión esté vigente (los códigos rotan).

## Referencia de configuración

Configuración completa: [Configuration](/gateway/configuration)

Opciones del proveedor:

- `channels.tlon.enabled`: habilitar o deshabilitar el inicio del canal.
- `channels.tlon.ship`: nombre de la nave Urbit del bot (por ejemplo `~sampel-palnet`).
- `channels.tlon.url`: URL de la nave (por ejemplo `https://sampel-palnet.tlon.network`).
- `channels.tlon.code`: código de inicio de sesión de la nave.
- `channels.tlon.allowPrivateNetwork`: permitir URL de localhost/LAN (omitir SSRF).
- `channels.tlon.ownerShip`: nave propietaria para el sistema de aprobación (siempre autorizada).
- `channels.tlon.dmAllowlist`: naves autorizadas para mensajes directos (vacío = ninguna).
- `channels.tlon.autoAcceptDmInvites`: aceptar automáticamente mensajes directos de naves permitidas.
- `channels.tlon.autoAcceptGroupInvites`: aceptar automáticamente todas las invitaciones a grupos.
- `channels.tlon.autoDiscoverChannels`: detectar automáticamente canales de grupo (predeterminado: true).
- `channels.tlon.groupChannels`: nidos de canales fijados manualmente.
- `channels.tlon.defaultAuthorizedShips`: naves autorizadas para todos los canales.
- `channels.tlon.authorization.channelRules`: reglas de autorización por canal.
- `channels.tlon.showModelSignature`: anexar el nombre del modelo a los mensajes.

## Notas

- Las respuestas en grupos requieren una mención (por ejemplo `~your-bot-ship`) para responder.
- Respuestas en hilos: si el mensaje entrante está en un hilo, OpenClaw responde dentro del hilo.
- Texto enriquecido: el formato Markdown (negrita, cursiva, código, encabezados, listas) se convierte al formato nativo de Tlon.
- Imágenes: las URL se cargan al almacenamiento de Tlon y se incrustan como bloques de imagen.

## Relacionado

- [Resumen de canales](/channels) — todos los canales compatibles
- [Emparejamiento](/channels/pairing) — autenticación de mensajes directos y flujo de emparejamiento
- [Grupos](/channels/groups) — comportamiento del chat grupal y restricción por mención
- [Enrutamiento de canales](/channels/channel-routing) — enrutamiento de sesiones para mensajes
- [Seguridad](/gateway/security) — modelo de acceso y refuerzo
