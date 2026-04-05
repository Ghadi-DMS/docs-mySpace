---
read_when:
    - Quieres elegir un canal de chat para OpenClaw
    - Necesitas una vista general rápida de las plataformas de mensajería compatibles
summary: Plataformas de mensajería a las que OpenClaw puede conectarse
title: Canales de chat
x-i18n:
    generated_at: "2026-04-05T12:35:12Z"
    model: gpt-5.4
    provider: openai
    source_hash: 246ee6f16aebe751241f00102bb435978ed21f6158385aff5d8e222e30567416
    source_path: channels/index.md
    workflow: 15
---

# Canales de chat

OpenClaw puede hablar contigo en cualquier aplicación de chat que ya uses. Cada canal se conecta a través del Gateway.
El texto es compatible en todas partes; los archivos multimedia y las reacciones varían según el canal.

## Canales compatibles

- [BlueBubbles](/channels/bluebubbles) — **Recomendado para iMessage**; usa la API REST del servidor BlueBubbles en macOS con compatibilidad completa de funciones (plugin incluido; editar, deshacer envío, efectos, reacciones, administración de grupos — la edición actualmente está rota en macOS 26 Tahoe).
- [Discord](/channels/discord) — API de bots de Discord + Gateway; admite servidores, canales y mensajes directos.
- [Feishu](/channels/feishu) — bot de Feishu/Lark mediante WebSocket (plugin incluido).
- [Google Chat](/channels/googlechat) — aplicación de la API de Google Chat mediante webhook HTTP.
- [iMessage (legacy)](/channels/imessage) — integración heredada de macOS mediante CLI imsg (obsoleta; usa BlueBubbles para configuraciones nuevas).
- [IRC](/channels/irc) — servidores IRC clásicos; canales y mensajes directos con controles de emparejamiento/lista de permitidos.
- [LINE](/channels/line) — bot de la API de mensajería LINE (plugin incluido).
- [Matrix](/channels/matrix) — protocolo Matrix (plugin incluido).
- [Mattermost](/channels/mattermost) — API de bots + WebSocket; canales, grupos y mensajes directos (plugin incluido).
- [Microsoft Teams](/channels/msteams) — Bot Framework; compatibilidad empresarial (plugin incluido).
- [Nextcloud Talk](/channels/nextcloud-talk) — chat autohospedado mediante Nextcloud Talk (plugin incluido).
- [Nostr](/channels/nostr) — mensajes directos descentralizados mediante NIP-04 (plugin incluido).
- [QQ Bot](/channels/qqbot) — API de QQ Bot; chat privado, chat grupal y multimedia enriquecido (plugin incluido).
- [Signal](/channels/signal) — signal-cli; centrado en la privacidad.
- [Slack](/channels/slack) — SDK de Bolt; aplicaciones de espacio de trabajo.
- [Synology Chat](/channels/synology-chat) — chat de Synology NAS mediante webhooks salientes+entrantes (plugin incluido).
- [Telegram](/channels/telegram) — API de bots mediante grammY; admite grupos.
- [Tlon](/channels/tlon) — mensajero basado en Urbit (plugin incluido).
- [Twitch](/channels/twitch) — chat de Twitch mediante conexión IRC (plugin incluido).
- [Voice Call](/plugins/voice-call) — telefonía mediante Plivo o Twilio (plugin, se instala por separado).
- [WebChat](/web/webchat) — interfaz de usuario WebChat del Gateway mediante WebSocket.
- [WeChat](https://www.npmjs.com/package/@tencent-weixin/openclaw-weixin) — plugin de bot Tencent iLink mediante inicio de sesión con QR; solo chats privados.
- [WhatsApp](/channels/whatsapp) — el más popular; usa Baileys y requiere emparejamiento con QR.
- [Zalo](/channels/zalo) — API de Zalo Bot; el mensajero popular de Vietnam (plugin incluido).
- [Zalo Personal](/channels/zalouser) — cuenta personal de Zalo mediante inicio de sesión con QR (plugin incluido).

## Notas

- Los canales pueden ejecutarse simultáneamente; configura varios y OpenClaw enrutará por chat.
- La configuración más rápida suele ser **Telegram** (token de bot simple). WhatsApp requiere emparejamiento con QR y
  almacena más estado en disco.
- El comportamiento de los grupos varía según el canal; consulta [Grupos](/channels/groups).
- El emparejamiento de mensajes directos y las listas de permitidos se aplican por seguridad; consulta [Seguridad](/gateway/security).
- Solución de problemas: [Solución de problemas de canales](/channels/troubleshooting).
- Los proveedores de modelos se documentan por separado; consulta [Proveedores de modelos](/providers/models).
