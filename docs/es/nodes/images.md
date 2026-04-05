---
read_when:
    - Modificar el pipeline multimedia o los archivos adjuntos
summary: Reglas de manejo de imágenes y contenido multimedia para send, gateway y respuestas del agente
title: Compatibilidad con imágenes y contenido multimedia
x-i18n:
    generated_at: "2026-04-05T12:47:05Z"
    model: gpt-5.4
    provider: openai
    source_hash: c3bb372b45a3bae51eae03b41cb22c4cde144675a54ddfd12e01a96132e48a8a
    source_path: nodes/images.md
    workflow: 15
---

# Compatibilidad con imágenes y contenido multimedia (2025-12-05)

El canal de WhatsApp se ejecuta mediante **Baileys Web**. Este documento recoge las reglas actuales de manejo multimedia para send, gateway y respuestas del agente.

## Objetivos

- Enviar contenido multimedia con subtítulos opcionales mediante `openclaw message send --media`.
- Permitir que las respuestas automáticas de la bandeja de entrada web incluyan contenido multimedia junto con texto.
- Mantener límites razonables y predecibles por tipo.

## Superficie de la CLI

- `openclaw message send --media <path-or-url> [--message <caption>]`
  - `--media` es opcional; el subtítulo puede estar vacío para envíos solo con contenido multimedia.
  - `--dry-run` imprime la carga útil resuelta; `--json` emite `{ channel, to, messageId, mediaUrl, caption }`.

## Comportamiento del canal WhatsApp Web

- Entrada: ruta de archivo local **o** URL HTTP(S).
- Flujo: cargar en un Buffer, detectar el tipo de contenido multimedia y crear la carga útil correcta:
  - **Imágenes:** redimensionar y recomprimir a JPEG (lado máximo 2048 px) apuntando a `channels.whatsapp.mediaMaxMb` (predeterminado: 50 MB).
  - **Audio/Voz/Video:** paso directo hasta 16 MB; el audio se envía como nota de voz (`ptt: true`).
  - **Documentos:** cualquier otra cosa, hasta 100 MB, conservando el nombre del archivo cuando esté disponible.
- Reproducción estilo GIF de WhatsApp: envía un MP4 con `gifPlayback: true` (CLI: `--gif-playback`) para que los clientes móviles lo reproduzcan en bucle en línea.
- La detección de MIME prefiere magic bytes, luego encabezados y después la extensión del archivo.
- El subtítulo proviene de `--message` o `reply.text`; se permite subtítulo vacío.
- Registro: en modo no detallado muestra `↩️`/`✅`; en modo detallado incluye tamaño y ruta/URL de origen.

## Pipeline de respuesta automática

- `getReplyFromConfig` devuelve `{ text?, mediaUrl?, mediaUrls? }`.
- Cuando hay contenido multimedia, el remitente web resuelve rutas locales o URLs usando el mismo pipeline que `openclaw message send`.
- Si se proporcionan varias entradas multimedia, se envían secuencialmente.

## Contenido multimedia entrante a comandos (Pi)

- Cuando los mensajes web entrantes incluyen contenido multimedia, OpenClaw lo descarga en un archivo temporal y expone variables de plantilla:
  - `{{MediaUrl}}` pseudo-URL para el contenido multimedia entrante.
  - `{{MediaPath}}` ruta temporal local escrita antes de ejecutar el comando.
- Cuando un sandbox Docker por sesión está habilitado, el contenido multimedia entrante se copia al espacio de trabajo del sandbox y `MediaPath`/`MediaUrl` se reescriben a una ruta relativa como `media/inbound/<filename>`.
- La comprensión multimedia (si se configura mediante `tools.media.*` o `tools.media.models` compartido) se ejecuta antes de la creación de plantillas y puede insertar bloques `[Image]`, `[Audio]` y `[Video]` en `Body`.
  - El audio establece `{{Transcript}}` y usa la transcripción para el análisis del comando, de modo que los comandos slash sigan funcionando.
  - Las descripciones de video e imagen conservan cualquier texto de subtítulo para el análisis del comando.
  - Si el modelo principal de imagen activo ya admite visión de forma nativa, OpenClaw omite el bloque de resumen `[Image]` y pasa la imagen original al modelo en su lugar.
- De forma predeterminada, solo se procesa el primer archivo adjunto coincidente de imagen/audio/video; establece `tools.media.<cap>.attachments` para procesar varios archivos adjuntos.

## Límites y errores

**Límites de envío saliente (envío web de WhatsApp)**

- Imágenes: hasta `channels.whatsapp.mediaMaxMb` (predeterminado: 50 MB) después de la recompresión.
- Audio/voz/video: límite de 16 MB; documentos: 100 MB.
- Contenido multimedia demasiado grande o ilegible → error claro en los registros y se omite la respuesta.

**Límites de comprensión multimedia (transcripción/descripción)**

- Valor predeterminado de imagen: 10 MB (`tools.media.image.maxBytes`).
- Valor predeterminado de audio: 20 MB (`tools.media.audio.maxBytes`).
- Valor predeterminado de video: 50 MB (`tools.media.video.maxBytes`).
- El contenido multimedia demasiado grande omite la comprensión, pero las respuestas siguen procesándose con el cuerpo original.

## Notas para pruebas

- Cubre flujos de envío + respuesta para casos de imagen/audio/documento.
- Valida la recompresión de imágenes (límite de tamaño) y el indicador de nota de voz para audio.
- Asegúrate de que las respuestas con varios elementos multimedia se distribuyan como envíos secuenciales.
