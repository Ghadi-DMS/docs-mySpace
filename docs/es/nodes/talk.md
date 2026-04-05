---
read_when:
    - Implementar el modo Talk en macOS/iOS/Android
    - Cambiar el comportamiento de voz/TTS/interrupción
summary: 'Modo Talk: conversaciones de voz continuas con ElevenLabs TTS'
title: Modo Talk
x-i18n:
    generated_at: "2026-04-05T12:47:20Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3f10a3e9ee8fc2b4f7a89771d6e7b7373166a51ef9e9aa2d8c5ea67fc0729f9d
    source_path: nodes/talk.md
    workflow: 15
---

# Modo Talk

El modo Talk es un bucle continuo de conversación por voz:

1. Escuchar el habla
2. Enviar la transcripción al modelo (sesión principal, `chat.send`)
3. Esperar la respuesta
4. Reproducirla mediante el proveedor Talk configurado (`talk.speak`)

## Comportamiento (macOS)

- **Superposición siempre activa** mientras el modo Talk está habilitado.
- Transiciones de fase **Escuchando → Pensando → Hablando**.
- En una **pausa breve** (ventana de silencio), se envía la transcripción actual.
- Las respuestas se **escriben en WebChat** (igual que al escribir).
- **Interrumpir al hablar** (activado de forma predeterminada): si la persona usuaria empieza a hablar mientras el asistente está hablando, detenemos la reproducción y anotamos la marca de tiempo de la interrupción para el siguiente prompt.

## Directivas de voz en las respuestas

El asistente puede anteponer a su respuesta una **única línea JSON** para controlar la voz:

```json
{ "voice": "<voice-id>", "once": true }
```

Reglas:

- Solo la primera línea no vacía.
- Las claves desconocidas se ignoran.
- `once: true` se aplica solo a la respuesta actual.
- Sin `once`, la voz se convierte en el nuevo valor predeterminado del modo Talk.
- La línea JSON se elimina antes de la reproducción TTS.

Claves compatibles:

- `voice` / `voice_id` / `voiceId`
- `model` / `model_id` / `modelId`
- `speed`, `rate` (WPM), `stability`, `similarity`, `style`, `speakerBoost`
- `seed`, `normalize`, `lang`, `output_format`, `latency_tier`
- `once`

## Configuración (`~/.openclaw/openclaw.json`)

```json5
{
  talk: {
    voiceId: "elevenlabs_voice_id",
    modelId: "eleven_v3",
    outputFormat: "mp3_44100_128",
    apiKey: "elevenlabs_api_key",
    silenceTimeoutMs: 1500,
    interruptOnSpeech: true,
  },
}
```

Valores predeterminados:

- `interruptOnSpeech`: true
- `silenceTimeoutMs`: cuando no está definido, Talk mantiene la ventana de pausa predeterminada de la plataforma antes de enviar la transcripción (`700 ms en macOS y Android, 900 ms en iOS`)
- `voiceId`: recurre a `ELEVENLABS_VOICE_ID` / `SAG_VOICE_ID` (o a la primera voz de ElevenLabs cuando la clave API está disponible)
- `modelId`: el valor predeterminado es `eleven_v3` cuando no está definido
- `apiKey`: recurre a `ELEVENLABS_API_KEY` (o al perfil de shell del gateway si está disponible)
- `outputFormat`: el valor predeterminado es `pcm_44100` en macOS/iOS y `pcm_24000` en Android (establece `mp3_*` para forzar streaming MP3)

## IU de macOS

- Conmutador de barra de menús: **Talk**
- Pestaña de configuración: grupo **Talk Mode** (id de voz + conmutador de interrupción)
- Superposición:
  - **Escuchando**: la nube pulsa con el nivel del micrófono
  - **Pensando**: animación de hundimiento
  - **Hablando**: anillos radiantes
  - Hacer clic en la nube: detener la reproducción
  - Hacer clic en X: salir del modo Talk

## Notas

- Requiere permisos de Speech + Microphone.
- Usa `chat.send` contra la clave de sesión `main`.
- El gateway resuelve la reproducción de Talk mediante `talk.speak` usando el proveedor Talk activo. Android recurre al TTS local del sistema solo cuando ese RPC no está disponible.
- `stability` para `eleven_v3` se valida como `0.0`, `0.5` o `1.0`; otros modelos aceptan `0..1`.
- `latency_tier` se valida como `0..4` cuando está definido.
- Android admite formatos de salida `pcm_16000`, `pcm_22050`, `pcm_24000` y `pcm_44100` para streaming AudioTrack de baja latencia.
