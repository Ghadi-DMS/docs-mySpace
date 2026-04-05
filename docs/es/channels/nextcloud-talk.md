---
read_when:
    - Trabajando en funciones del canal de Nextcloud Talk
summary: Estado de compatibilidad, capacidades y configuración de Nextcloud Talk
title: Nextcloud Talk
x-i18n:
    generated_at: "2026-04-05T12:35:47Z"
    model: gpt-5.4
    provider: openai
    source_hash: 900402afe67cf3ce96103d55158eb28cffb29c9845b77248e70d7653b12ae810
    source_path: channels/nextcloud-talk.md
    workflow: 15
---

# Nextcloud Talk

Estado: plugin incluido (bot de webhook). Se admiten mensajes directos, salas, reacciones y mensajes con markdown.

## Plugin incluido

Nextcloud Talk se incluye como plugin integrado en las versiones actuales de OpenClaw, por lo que las compilaciones empaquetadas normales no necesitan una instalación separada.

Si estás usando una compilación antigua o una instalación personalizada que excluye Nextcloud Talk, instálalo manualmente:

Instalar mediante la CLI (registro npm):

```bash
openclaw plugins install @openclaw/nextcloud-talk
```

Checkout local (al ejecutarlo desde un repositorio git):

```bash
openclaw plugins install ./path/to/local/nextcloud-talk-plugin
```

Detalles: [Plugins](/tools/plugin)

## Configuración rápida (principiante)

1. Asegúrate de que el plugin de Nextcloud Talk esté disponible.
   - Las versiones empaquetadas actuales de OpenClaw ya lo incluyen.
   - Las instalaciones antiguas/personalizadas pueden agregarlo manualmente con los comandos anteriores.
2. En tu servidor de Nextcloud, crea un bot:

   ```bash
   ./occ talk:bot:install "OpenClaw" "<shared-secret>" "<webhook-url>" --feature reaction
   ```

3. Habilita el bot en la configuración de la sala de destino.
4. Configura OpenClaw:
   - Configuración: `channels.nextcloud-talk.baseUrl` + `channels.nextcloud-talk.botSecret`
   - O variable de entorno: `NEXTCLOUD_TALK_BOT_SECRET` (solo cuenta predeterminada)
5. Reinicia el gateway (o termina la configuración).

Configuración mínima:

```json5
{
  channels: {
    "nextcloud-talk": {
      enabled: true,
      baseUrl: "https://cloud.example.com",
      botSecret: "shared-secret",
      dmPolicy: "pairing",
    },
  },
}
```

## Notas

- Los bots no pueden iniciar mensajes directos. El usuario debe enviar primero un mensaje al bot.
- La URL del webhook debe ser accesible desde el Gateway; configura `webhookPublicUrl` si estás detrás de un proxy.
- La API del bot no admite cargas de medios; los medios se envían como URL.
- La carga útil del webhook no distingue entre mensajes directos y salas; configura `apiUser` + `apiPassword` para habilitar búsquedas de tipo de sala (de lo contrario, los mensajes directos se tratan como salas).

## Control de acceso (mensajes directos)

- Predeterminado: `channels.nextcloud-talk.dmPolicy = "pairing"`. Los remitentes desconocidos reciben un código de pairing.
- Aprobar mediante:
  - `openclaw pairing list nextcloud-talk`
  - `openclaw pairing approve nextcloud-talk <CODE>`
- Mensajes directos públicos: `channels.nextcloud-talk.dmPolicy="open"` más `channels.nextcloud-talk.allowFrom=["*"]`.
- `allowFrom` coincide solo con ID de usuario de Nextcloud; los nombres para mostrar se ignoran.

## Salas (grupos)

- Predeterminado: `channels.nextcloud-talk.groupPolicy = "allowlist"` (controlado por menciones).
- Incluye salas en la allowlist con `channels.nextcloud-talk.rooms`:

```json5
{
  channels: {
    "nextcloud-talk": {
      rooms: {
        "room-token": { requireMention: true },
      },
    },
  },
}
```

- Para no permitir ninguna sala, deja vacía la allowlist o establece `channels.nextcloud-talk.groupPolicy="disabled"`.

## Capacidades

| Función          | Estado         |
| ---------------- | -------------- |
| Mensajes directos | Compatible     |
| Salas            | Compatible     |
| Hilos            | No compatible  |
| Medios           | Solo URL       |
| Reacciones       | Compatible     |
| Comandos nativos | No compatible  |

## Referencia de configuración (Nextcloud Talk)

Configuración completa: [Configuration](/gateway/configuration)

Opciones del proveedor:

- `channels.nextcloud-talk.enabled`: habilitar/deshabilitar el inicio del canal.
- `channels.nextcloud-talk.baseUrl`: URL de la instancia de Nextcloud.
- `channels.nextcloud-talk.botSecret`: secreto compartido del bot.
- `channels.nextcloud-talk.botSecretFile`: ruta del secreto en archivo regular. Se rechazan enlaces simbólicos.
- `channels.nextcloud-talk.apiUser`: usuario de API para búsquedas de salas (detección de mensajes directos).
- `channels.nextcloud-talk.apiPassword`: contraseña de API/app para búsquedas de salas.
- `channels.nextcloud-talk.apiPasswordFile`: ruta del archivo de contraseña de API.
- `channels.nextcloud-talk.webhookPort`: puerto del listener de webhook (predeterminado: 8788).
- `channels.nextcloud-talk.webhookHost`: host del webhook (predeterminado: 0.0.0.0).
- `channels.nextcloud-talk.webhookPath`: ruta del webhook (predeterminado: /nextcloud-talk-webhook).
- `channels.nextcloud-talk.webhookPublicUrl`: URL del webhook accesible externamente.
- `channels.nextcloud-talk.dmPolicy`: `pairing | allowlist | open | disabled`.
- `channels.nextcloud-talk.allowFrom`: allowlist de mensajes directos (ID de usuario). `open` requiere `"*"`.
- `channels.nextcloud-talk.groupPolicy`: `allowlist | open | disabled`.
- `channels.nextcloud-talk.groupAllowFrom`: allowlist de grupos (ID de usuario).
- `channels.nextcloud-talk.rooms`: configuración y allowlist por sala.
- `channels.nextcloud-talk.historyLimit`: límite de historial de grupos (0 desactiva).
- `channels.nextcloud-talk.dmHistoryLimit`: límite de historial de mensajes directos (0 desactiva).
- `channels.nextcloud-talk.dms`: sobrescrituras por mensaje directo (`historyLimit`).
- `channels.nextcloud-talk.textChunkLimit`: tamaño de fragmento del texto saliente (caracteres).
- `channels.nextcloud-talk.chunkMode`: `length` (predeterminado) o `newline` para dividir en líneas en blanco (límites de párrafo) antes de fragmentar por longitud.
- `channels.nextcloud-talk.blockStreaming`: deshabilitar el streaming por bloques para este canal.
- `channels.nextcloud-talk.blockStreamingCoalesce`: ajuste de coalescencia del streaming por bloques.
- `channels.nextcloud-talk.mediaMaxMb`: límite de medios entrantes (MB).

## Relacionado

- [Channels Overview](/channels) — todos los canales compatibles
- [Pairing](/channels/pairing) — autenticación de mensajes directos y flujo de pairing
- [Groups](/channels/groups) — comportamiento de chats grupales y control por menciones
- [Channel Routing](/channels/channel-routing) — enrutamiento de sesiones para mensajes
- [Security](/gateway/security) — modelo de acceso y endurecimiento
