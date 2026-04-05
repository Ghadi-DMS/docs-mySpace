---
read_when:
    - Configurando el control de acceso a MD
    - Emparejando un nuevo nodo iOS/Android
    - Revisando la postura de seguridad de OpenClaw
summary: 'Resumen de pairing: aprobar quién puede enviarte MD y qué nodos pueden unirse'
title: Pairing
x-i18n:
    generated_at: "2026-04-05T12:36:01Z"
    model: gpt-5.4
    provider: openai
    source_hash: 2bd99240b3530def23c05a26915d07cf8b730565c2822c6338437f8fb3f285c9
    source_path: channels/pairing.md
    workflow: 15
---

# Pairing

“Pairing” es el paso explícito de **aprobación del propietario** en OpenClaw.
Se utiliza en dos lugares:

1. **Pairing de MD** (quién tiene permiso para hablar con el bot)
2. **Pairing de nodos** (qué dispositivos/nodos tienen permiso para unirse a la red del gateway)

Contexto de seguridad: [Seguridad](/gateway/security)

## 1) Pairing de MD (acceso al chat entrante)

Cuando un canal está configurado con la política de MD `pairing`, los remitentes desconocidos reciben un código corto y su mensaje **no se procesa** hasta que lo apruebes.

Las políticas predeterminadas de MD están documentadas en: [Seguridad](/gateway/security)

Códigos de pairing:

- 8 caracteres, en mayúsculas, sin caracteres ambiguos (`0O1I`).
- **Caducan después de 1 hora**. El bot solo envía el mensaje de pairing cuando se crea una nueva solicitud (aproximadamente una vez por hora por remitente).
- Las solicitudes pendientes de pairing de MD están limitadas de forma predeterminada a **3 por canal**; las solicitudes adicionales se ignoran hasta que una caduque o se apruebe.

### Aprobar un remitente

```bash
openclaw pairing list telegram
openclaw pairing approve telegram <CODE>
```

Canales compatibles: `bluebubbles`, `discord`, `feishu`, `googlechat`, `imessage`, `irc`, `line`, `matrix`, `mattermost`, `msteams`, `nextcloud-talk`, `nostr`, `openclaw-weixin`, `signal`, `slack`, `synology-chat`, `telegram`, `twitch`, `whatsapp`, `zalo`, `zalouser`.

### Dónde se almacena el estado

Se almacena en `~/.openclaw/credentials/`:

- Solicitudes pendientes: `<channel>-pairing.json`
- Almacén de listas de permitidos aprobadas:
  - Cuenta predeterminada: `<channel>-allowFrom.json`
  - Cuenta no predeterminada: `<channel>-<accountId>-allowFrom.json`

Comportamiento del alcance por cuenta:

- Las cuentas no predeterminadas solo leen/escriben su archivo de lista de permitidos con alcance propio.
- La cuenta predeterminada usa el archivo de lista de permitidos sin alcance específico del canal.

Trata estos archivos como sensibles (controlan el acceso a tu asistente).

Importante: este almacén es para acceso por MD. La autorización de grupos es independiente.
Aprobar un código de pairing de MD no permite automáticamente que ese remitente ejecute comandos de grupo o controle el bot en grupos. Para el acceso a grupos, configura las listas de permitidos explícitas del canal para grupos (por ejemplo, `groupAllowFrom`, `groups` o anulaciones por grupo/por tema según el canal).

## 2) Pairing de dispositivos nodo (nodos iOS/Android/macOS/sin interfaz)

Los nodos se conectan al Gateway como **dispositivos** con `role: node`. El Gateway
crea una solicitud de pairing de dispositivo que debe aprobarse.

### Emparejar mediante Telegram (recomendado para iOS)

Si usas el plugin `device-pair`, puedes hacer el pairing inicial del dispositivo completamente desde Telegram:

1. En Telegram, envía a tu bot: `/pair`
2. El bot responde con dos mensajes: un mensaje de instrucciones y un mensaje separado con el **código de configuración** (fácil de copiar/pegar en Telegram).
3. En tu teléfono, abre la app de OpenClaw para iOS → Ajustes → Gateway.
4. Pega el código de configuración y conéctate.
5. De vuelta en Telegram: `/pair pending` (revisa los ID de solicitud, el rol y los alcances), luego aprueba.

El código de configuración es una carga JSON codificada en base64 que contiene:

- `url`: la URL WebSocket del Gateway (`ws://...` o `wss://...`)
- `bootstrapToken`: un token de bootstrap de corta duración para un solo dispositivo usado en el protocolo inicial de pairing

Ese token de bootstrap lleva el perfil de bootstrap de pairing integrado:

- el token `node` principal transferido permanece con `scopes: []`
- cualquier token `operator` transferido permanece limitado a la lista de permitidos de bootstrap:
  `operator.approvals`, `operator.read`, `operator.talk.secrets`, `operator.write`
- las comprobaciones de alcance del bootstrap usan prefijos por rol, no un único conjunto plano de alcances:
  las entradas de alcance de operator solo satisfacen solicitudes de operator, y los roles que no son operator
  deben seguir solicitando alcances bajo su propio prefijo de rol

Trata el código de configuración como una contraseña mientras sea válido.

### Aprobar un dispositivo nodo

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw devices reject <requestId>
```

Si el mismo dispositivo vuelve a intentarlo con distintos detalles de autenticación (por ejemplo, distinto
rol/alcances/clave pública), la solicitud pendiente anterior es reemplazada y se crea un nuevo
`requestId`.

### Almacenamiento del estado de pairing de nodos

Se almacena en `~/.openclaw/devices/`:

- `pending.json` (de corta duración; las solicitudes pendientes caducan)
- `paired.json` (dispositivos emparejados + tokens)

### Notas

- La API heredada `node.pair.*` (CLI: `openclaw nodes pending|approve|reject|rename`) es un
  almacén de pairing independiente propiedad del gateway. Los nodos WS siguen requiriendo pairing de dispositivo.
- El registro de pairing es la fuente de verdad duradera para los roles aprobados. Los tokens de dispositivo activos
  permanecen limitados a ese conjunto de roles aprobados; una entrada de token aislada
  fuera de los roles aprobados no crea acceso nuevo.

## Documentos relacionados

- Modelo de seguridad + inyección de prompts: [Seguridad](/gateway/security)
- Actualizar de forma segura (ejecuta doctor): [Actualización](/install/updating)
- Configuraciones de canales:
  - Telegram: [Telegram](/channels/telegram)
  - WhatsApp: [WhatsApp](/channels/whatsapp)
  - Signal: [Signal](/channels/signal)
  - BlueBubbles (iMessage): [BlueBubbles](/channels/bluebubbles)
  - iMessage (heredado): [iMessage](/channels/imessage)
  - Discord: [Discord](/channels/discord)
  - Slack: [Slack](/channels/slack)
