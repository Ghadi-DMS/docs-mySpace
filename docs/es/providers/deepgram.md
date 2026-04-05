---
read_when:
    - Quieres usar Deepgram speech-to-text para archivos adjuntos de audio
    - Necesitas un ejemplo rápido de configuración de Deepgram
summary: Transcripción de Deepgram para notas de voz entrantes
title: Deepgram
x-i18n:
    generated_at: "2026-04-05T12:50:57Z"
    model: gpt-5.4
    provider: openai
    source_hash: dabd1f6942c339fbd744fbf38040b6a663b06ddf4d9c9ee31e3ac034de9e79d9
    source_path: providers/deepgram.md
    workflow: 15
---

# Deepgram (transcripción de audio)

Deepgram es una API de speech-to-text. En OpenClaw se usa para la **transcripción
de audio/notas de voz entrantes** mediante `tools.media.audio`.

Cuando está habilitado, OpenClaw carga el archivo de audio en Deepgram e inyecta la transcripción
en la canalización de respuesta (`{{Transcript}}` + bloque `[Audio]`). Esto **no es streaming**;
usa el endpoint de transcripción de audio pregrabado.

Sitio web: [https://deepgram.com](https://deepgram.com)  
Documentación: [https://developers.deepgram.com](https://developers.deepgram.com)

## Inicio rápido

1. Configura tu clave de API:

```
DEEPGRAM_API_KEY=dg_...
```

2. Habilita el proveedor:

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        models: [{ provider: "deepgram", model: "nova-3" }],
      },
    },
  },
}
```

## Opciones

- `model`: id del modelo de Deepgram (predeterminado: `nova-3`)
- `language`: pista de idioma (opcional)
- `tools.media.audio.providerOptions.deepgram.detect_language`: habilita la detección de idioma (opcional)
- `tools.media.audio.providerOptions.deepgram.punctuate`: habilita la puntuación (opcional)
- `tools.media.audio.providerOptions.deepgram.smart_format`: habilita el formato inteligente (opcional)

Ejemplo con idioma:

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        models: [{ provider: "deepgram", model: "nova-3", language: "en" }],
      },
    },
  },
}
```

Ejemplo con opciones de Deepgram:

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        providerOptions: {
          deepgram: {
            detect_language: true,
            punctuate: true,
            smart_format: true,
          },
        },
        models: [{ provider: "deepgram", model: "nova-3" }],
      },
    },
  },
}
```

## Notas

- La autenticación sigue el orden estándar de autenticación del proveedor; `DEEPGRAM_API_KEY` es la ruta más simple.
- Anula endpoints o cabeceras con `tools.media.audio.baseUrl` y `tools.media.audio.headers` cuando uses un proxy.
- La salida sigue las mismas reglas de audio que otros proveedores (límites de tamaño, tiempos de espera, inyección de transcripción).
