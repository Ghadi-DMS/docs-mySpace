---
read_when:
    - Quieres una lista completa de todo lo que admite OpenClaw
summary: Capacidades de OpenClaw en canales, enrutamiento, contenido multimedia y experiencia de usuario.
title: Funciones
x-i18n:
    generated_at: "2026-04-05T12:39:32Z"
    model: gpt-5.4
    provider: openai
    source_hash: 43eae89d9af44ea786dd0221d8d602ebcea15da9d5064396ac9920c0345e2ad3
    source_path: concepts/features.md
    workflow: 15
---

# Funciones

## Aspectos destacados

<Columns>
  <Card title="Canales" icon="message-square">
    Discord, iMessage, Signal, Slack, Telegram, WhatsApp, WebChat y más con un único Gateway.
  </Card>
  <Card title="Plugins" icon="plug">
    Los plugins empaquetados añaden Matrix, Nextcloud Talk, Nostr, Twitch, Zalo y más sin instalaciones separadas en las versiones actuales normales.
  </Card>
  <Card title="Enrutamiento" icon="route">
    Enrutamiento multiagente con sesiones aisladas.
  </Card>
  <Card title="Contenido multimedia" icon="image">
    Imágenes, audio, video, documentos y generación de imágenes/video.
  </Card>
  <Card title="Aplicaciones e interfaz" icon="monitor">
    Interfaz web de Control y aplicación complementaria para macOS.
  </Card>
  <Card title="Nodos móviles" icon="smartphone">
    Nodos de iOS y Android con emparejamiento, voz/chat y comandos avanzados del dispositivo.
  </Card>
</Columns>

## Lista completa

**Canales:**

- Los canales integrados incluyen Discord, Google Chat, iMessage (heredado), IRC, Signal, Slack, Telegram, WebChat y WhatsApp
- Los canales de plugins empaquetados incluyen BlueBubbles para iMessage, Feishu, LINE, Matrix, Mattermost, Microsoft Teams, Nextcloud Talk, Nostr, QQ Bot, Synology Chat, Tlon, Twitch, Zalo y Zalo Personal
- Los plugins de canal opcionales instalados por separado incluyen Voice Call y paquetes de terceros como WeChat
- Los plugins de canal de terceros pueden ampliar aún más el Gateway, como WeChat
- Compatibilidad con chats grupales con activación basada en menciones
- Seguridad en mensajes directos con listas de permitidos y emparejamiento

**Agente:**

- Entorno de ejecución de agente integrado con streaming de herramientas
- Enrutamiento multiagente con sesiones aisladas por espacio de trabajo o remitente
- Sesiones: los chats directos se consolidan en `main`; los grupos están aislados
- Streaming y fragmentación para respuestas largas

**Autenticación y proveedores:**

- Más de 35 proveedores de modelos (Anthropic, OpenAI, Google y más)
- Autenticación de suscripción mediante OAuth (por ejemplo, OpenAI Codex)
- Compatibilidad con proveedores personalizados y autoalojados (vLLM, SGLang, Ollama y cualquier endpoint compatible con OpenAI o Anthropic)

**Contenido multimedia:**

- Imágenes, audio, video y documentos de entrada y salida
- Superficies de capacidades compartidas para generación de imágenes y generación de video
- Transcripción de notas de voz
- Conversión de texto a voz con varios proveedores

**Aplicaciones e interfaces:**

- WebChat e interfaz web de Control
- Aplicación complementaria de barra de menús para macOS
- Nodo de iOS con emparejamiento, Canvas, cámara, grabación de pantalla, ubicación y voz
- Nodo de Android con emparejamiento, chat, voz, Canvas, cámara y comandos del dispositivo

**Herramientas y automatización:**

- Automatización de navegador, exec, sandboxing
- Búsqueda web (Brave, DuckDuckGo, Exa, Firecrawl, Gemini, Grok, Kimi, MiniMax Search, Ollama Web Search, Perplexity, SearXNG, Tavily)
- Trabajos cron y programación de heartbeat
- Skills, plugins y canalizaciones de flujo de trabajo (Lobster)
