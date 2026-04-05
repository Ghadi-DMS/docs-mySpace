---
read_when:
    - Habilitar texto a voz para respuestas
    - Configurar proveedores o límites de TTS
    - Usar comandos /tts
summary: Texto a voz (TTS) para respuestas salientes
title: Texto a voz
x-i18n:
    generated_at: "2026-04-05T12:57:20Z"
    model: gpt-5.4
    provider: openai
    source_hash: 8487c8acef7585bd4eb5e3b39e2a063ebc6b5f0103524abdcbadd3a7781ffc46
    source_path: tools/tts.md
    workflow: 15
---

# Texto a voz (TTS)

OpenClaw puede convertir respuestas salientes en audio usando ElevenLabs, Microsoft, MiniMax u OpenAI.
Funciona en cualquier lugar donde OpenClaw pueda enviar audio.

## Servicios compatibles

- **ElevenLabs** (proveedor principal o de respaldo)
- **Microsoft** (proveedor principal o de respaldo; la implementación empaquetada actual usa `node-edge-tts`)
- **MiniMax** (proveedor principal o de respaldo; usa la API T2A v2)
- **OpenAI** (proveedor principal o de respaldo; también se usa para resúmenes)

### Notas sobre Microsoft speech

El proveedor de voz de Microsoft empaquetado actualmente usa el servicio
neural de TTS en línea de Microsoft Edge mediante la biblioteca `node-edge-tts`. Es un servicio alojado (no
local), usa endpoints de Microsoft y no requiere una clave de API.
`node-edge-tts` expone opciones de configuración de voz y formatos de salida, pero
no todas las opciones son compatibles con el servicio. La configuración heredada y la entrada de directivas
que usan `edge` siguen funcionando y se normalizan a `microsoft`.

Dado que esta ruta es un servicio web público sin un SLA ni cuota publicados,
considérala como de mejor esfuerzo. Si necesitas límites garantizados y soporte, usa OpenAI
o ElevenLabs.

## Claves opcionales

Si quieres OpenAI, ElevenLabs o MiniMax:

- `ELEVENLABS_API_KEY` (o `XI_API_KEY`)
- `MINIMAX_API_KEY`
- `OPENAI_API_KEY`

Microsoft speech **no** requiere una clave de API.

Si hay varios proveedores configurados, se usa primero el proveedor seleccionado y los demás son opciones de respaldo.
El resumen automático usa el `summaryModel` configurado (o `agents.defaults.model.primary`),
por lo que ese proveedor también debe estar autenticado si habilitas los resúmenes.

## Enlaces de servicios

- [Guía de texto a voz de OpenAI](https://platform.openai.com/docs/guides/text-to-speech)
- [Referencia de la API de Audio de OpenAI](https://platform.openai.com/docs/api-reference/audio)
- [Texto a voz de ElevenLabs](https://elevenlabs.io/docs/api-reference/text-to-speech)
- [Autenticación de ElevenLabs](https://elevenlabs.io/docs/api-reference/authentication)
- [API T2A v2 de MiniMax](https://platform.minimaxi.com/document/T2A%20V2)
- [node-edge-tts](https://github.com/SchneeHertz/node-edge-tts)
- [Formatos de salida de Microsoft Speech](https://learn.microsoft.com/azure/ai-services/speech-service/rest-text-to-speech#audio-outputs)

## ¿Está habilitado de forma predeterminada?

No. El TTS automático está **desactivado** de forma predeterminada. Habilítalo en la configuración con
`messages.tts.auto` o por sesión con `/tts always` (alias: `/tts on`).

Cuando `messages.tts.provider` no está configurado, OpenClaw elige el primer
proveedor de voz configurado en el orden de selección automática del registro.

## Configuración

La configuración de TTS vive en `messages.tts` en `openclaw.json`.
El esquema completo está en [Configuración del Gateway](/es/gateway/configuration).

### Configuración mínima (habilitar + proveedor)

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "elevenlabs",
    },
  },
}
```

### OpenAI principal con ElevenLabs como respaldo

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "openai",
      summaryModel: "openai/gpt-4.1-mini",
      modelOverrides: {
        enabled: true,
      },
      providers: {
        openai: {
          apiKey: "openai_api_key",
          baseUrl: "https://api.openai.com/v1",
          model: "gpt-4o-mini-tts",
          voice: "alloy",
        },
        elevenlabs: {
          apiKey: "elevenlabs_api_key",
          baseUrl: "https://api.elevenlabs.io",
          voiceId: "voice_id",
          modelId: "eleven_multilingual_v2",
          seed: 42,
          applyTextNormalization: "auto",
          languageCode: "en",
          voiceSettings: {
            stability: 0.5,
            similarityBoost: 0.75,
            style: 0.0,
            useSpeakerBoost: true,
            speed: 1.0,
          },
        },
      },
    },
  },
}
```

### Microsoft principal (sin clave de API)

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "microsoft",
      providers: {
        microsoft: {
          enabled: true,
          voice: "en-US-MichelleNeural",
          lang: "en-US",
          outputFormat: "audio-24khz-48kbitrate-mono-mp3",
          rate: "+10%",
          pitch: "-5%",
        },
      },
    },
  },
}
```

### MiniMax principal

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "minimax",
      providers: {
        minimax: {
          apiKey: "minimax_api_key",
          baseUrl: "https://api.minimax.io",
          model: "speech-2.8-hd",
          voiceId: "English_expressive_narrator",
          speed: 1.0,
          vol: 1.0,
          pitch: 0,
        },
      },
    },
  },
}
```

### Deshabilitar Microsoft speech

```json5
{
  messages: {
    tts: {
      providers: {
        microsoft: {
          enabled: false,
        },
      },
    },
  },
}
```

### Límites personalizados + ruta de preferencias

```json5
{
  messages: {
    tts: {
      auto: "always",
      maxTextLength: 4000,
      timeoutMs: 30000,
      prefsPath: "~/.openclaw/settings/tts.json",
    },
  },
}
```

### Responder con audio solo después de un mensaje de voz entrante

```json5
{
  messages: {
    tts: {
      auto: "inbound",
    },
  },
}
```

### Deshabilitar el resumen automático para respuestas largas

```json5
{
  messages: {
    tts: {
      auto: "always",
    },
  },
}
```

Luego ejecuta:

```
/tts summary off
```

### Notas sobre los campos

- `auto`: modo de TTS automático (`off`, `always`, `inbound`, `tagged`).
  - `inbound` solo envía audio después de un mensaje de voz entrante.
  - `tagged` solo envía audio cuando la respuesta incluye etiquetas `[[tts]]`.
- `enabled`: interruptor heredado (doctor migra esto a `auto`).
- `mode`: `"final"` (predeterminado) o `"all"` (incluye respuestas de herramientas/bloques).
- `provider`: id del proveedor de voz, como `"elevenlabs"`, `"microsoft"`, `"minimax"` u `"openai"` (el respaldo es automático).
- Si `provider` **no** está configurado, OpenClaw usa el primer proveedor de voz configurado en el orden de selección automática del registro.
- El heredado `provider: "edge"` sigue funcionando y se normaliza a `microsoft`.
- `summaryModel`: modelo económico opcional para resumen automático; usa por defecto `agents.defaults.model.primary`.
  - Acepta `provider/model` o un alias de modelo configurado.
- `modelOverrides`: permite que el modelo emita directivas de TTS (activado de forma predeterminada).
  - `allowProvider` usa `false` de forma predeterminada (el cambio de proveedor es opcional).
- `providers.<id>`: configuración propiedad del proveedor, indexada por id del proveedor de voz.
- Los bloques heredados directos de proveedor (`messages.tts.openai`, `messages.tts.elevenlabs`, `messages.tts.microsoft`, `messages.tts.edge`) se migran automáticamente a `messages.tts.providers.<id>` al cargar.
- `maxTextLength`: límite estricto para la entrada de TTS (caracteres). `/tts audio` falla si se supera.
- `timeoutMs`: tiempo de espera de la solicitud (ms).
- `prefsPath`: anula la ruta local del JSON de preferencias (proveedor/límite/resumen).
- Los valores `apiKey` recurren a variables de entorno (`ELEVENLABS_API_KEY`/`XI_API_KEY`, `MINIMAX_API_KEY`, `OPENAI_API_KEY`).
- `providers.elevenlabs.baseUrl`: anula la URL base de la API de ElevenLabs.
- `providers.openai.baseUrl`: anula el endpoint de TTS de OpenAI.
  - Orden de resolución: `messages.tts.providers.openai.baseUrl` -> `OPENAI_TTS_BASE_URL` -> `https://api.openai.com/v1`
  - Los valores no predeterminados se tratan como endpoints de TTS compatibles con OpenAI, por lo que se aceptan nombres personalizados de modelo y voz.
- `providers.elevenlabs.voiceSettings`:
  - `stability`, `similarityBoost`, `style`: `0..1`
  - `useSpeakerBoost`: `true|false`
  - `speed`: `0.5..2.0` (1.0 = normal)
- `providers.elevenlabs.applyTextNormalization`: `auto|on|off`
- `providers.elevenlabs.languageCode`: ISO 639-1 de 2 letras (por ejemplo `en`, `de`)
- `providers.elevenlabs.seed`: entero `0..4294967295` (determinismo de mejor esfuerzo)
- `providers.minimax.baseUrl`: anula la URL base de la API de MiniMax (predeterminada `https://api.minimax.io`, env: `MINIMAX_API_HOST`).
- `providers.minimax.model`: modelo de TTS (predeterminado `speech-2.8-hd`, env: `MINIMAX_TTS_MODEL`).
- `providers.minimax.voiceId`: identificador de voz (predeterminado `English_expressive_narrator`, env: `MINIMAX_TTS_VOICE_ID`).
- `providers.minimax.speed`: velocidad de reproducción `0.5..2.0` (predeterminada 1.0).
- `providers.minimax.vol`: volumen `(0, 10]` (predeterminado 1.0; debe ser mayor que 0).
- `providers.minimax.pitch`: desplazamiento de tono `-12..12` (predeterminado 0).
- `providers.microsoft.enabled`: permite el uso de Microsoft speech (predeterminado `true`; sin clave de API).
- `providers.microsoft.voice`: nombre de voz neural de Microsoft (por ejemplo `en-US-MichelleNeural`).
- `providers.microsoft.lang`: código de idioma (por ejemplo `en-US`).
- `providers.microsoft.outputFormat`: formato de salida de Microsoft (por ejemplo `audio-24khz-48kbitrate-mono-mp3`).
  - Consulta los formatos de salida de Microsoft Speech para ver valores válidos; no todos los formatos son compatibles con el transporte empaquetado respaldado por Edge.
- `providers.microsoft.rate` / `providers.microsoft.pitch` / `providers.microsoft.volume`: cadenas de porcentaje (por ejemplo `+10%`, `-5%`).
- `providers.microsoft.saveSubtitles`: escribe subtítulos JSON junto al archivo de audio.
- `providers.microsoft.proxy`: URL de proxy para solicitudes de Microsoft speech.
- `providers.microsoft.timeoutMs`: anulación del tiempo de espera de la solicitud (ms).
- `edge.*`: alias heredado para la misma configuración de Microsoft.

## Anulaciones controladas por el modelo (activadas de forma predeterminada)

De forma predeterminada, el modelo **puede** emitir directivas de TTS para una sola respuesta.
Cuando `messages.tts.auto` es `tagged`, estas directivas son obligatorias para activar el audio.

Cuando está habilitado, el modelo puede emitir directivas `[[tts:...]]` para anular la voz
de una sola respuesta, además de un bloque opcional `[[tts:text]]...[[/tts:text]]` para
proporcionar etiquetas expresivas (risa, indicaciones de canto, etc.) que solo deben aparecer en
el audio.

Las directivas `provider=...` se ignoran a menos que `modelOverrides.allowProvider: true`.

Ejemplo de carga útil de respuesta:

```
Here you go.

[[tts:voiceId=pMsXgVXv3BLzUgSXRplE model=eleven_v3 speed=1.1]]
[[tts:text]](laughs) Read the song once more.[[/tts:text]]
```

Claves de directiva disponibles (cuando están habilitadas):

- `provider` (id del proveedor de voz registrado, por ejemplo `openai`, `elevenlabs`, `minimax` o `microsoft`; requiere `allowProvider: true`)
- `voice` (voz de OpenAI) o `voiceId` (ElevenLabs / MiniMax)
- `model` (modelo TTS de OpenAI, id de modelo de ElevenLabs o modelo de MiniMax)
- `stability`, `similarityBoost`, `style`, `speed`, `useSpeakerBoost`
- `vol` / `volume` (volumen de MiniMax, 0-10)
- `pitch` (tono de MiniMax, -12 a 12)
- `applyTextNormalization` (`auto|on|off`)
- `languageCode` (ISO 639-1)
- `seed`

Deshabilitar todas las anulaciones del modelo:

```json5
{
  messages: {
    tts: {
      modelOverrides: {
        enabled: false,
      },
    },
  },
}
```

Allowlist opcional (habilita el cambio de proveedor mientras mantiene configurables otros parámetros):

```json5
{
  messages: {
    tts: {
      modelOverrides: {
        enabled: true,
        allowProvider: true,
        allowSeed: false,
      },
    },
  },
}
```

## Preferencias por usuario

Los comandos slash escriben anulaciones locales en `prefsPath` (predeterminado:
`~/.openclaw/settings/tts.json`, anulable con `OPENCLAW_TTS_PREFS` o
`messages.tts.prefsPath`).

Campos almacenados:

- `enabled`
- `provider`
- `maxLength` (umbral de resumen; predeterminado 1500 caracteres)
- `summarize` (predeterminado `true`)

Estos anulan `messages.tts.*` para ese host.

## Formatos de salida (fijos)

- **Feishu / Matrix / Telegram / WhatsApp**: mensaje de voz Opus (`opus_48000_64` de ElevenLabs, `opus` de OpenAI).
  - 48 kHz / 64 kbps es un buen equilibrio para mensajes de voz.
- **Otros canales**: MP3 (`mp3_44100_128` de ElevenLabs, `mp3` de OpenAI).
  - 44.1 kHz / 128 kbps es el equilibrio predeterminado para claridad de voz.
- **MiniMax**: MP3 (modelo `speech-2.8-hd`, frecuencia de muestreo de 32 kHz). El formato de nota de voz no es compatible de forma nativa; usa OpenAI o ElevenLabs para mensajes de voz Opus garantizados.
- **Microsoft**: usa `microsoft.outputFormat` (predeterminado `audio-24khz-48kbitrate-mono-mp3`).
  - El transporte empaquetado acepta un `outputFormat`, pero no todos los formatos están disponibles en el servicio.
  - Los valores de formato de salida siguen los formatos de salida de Microsoft Speech (incluido Ogg/WebM Opus).
  - Telegram `sendVoice` acepta OGG/MP3/M4A; usa OpenAI/ElevenLabs si necesitas
    mensajes de voz Opus garantizados.
  - Si falla el formato de salida configurado de Microsoft, OpenClaw reintenta con MP3.

Los formatos de salida de OpenAI/ElevenLabs son fijos por canal (consulta arriba).

## Comportamiento del TTS automático

Cuando está habilitado, OpenClaw:

- omite TTS si la respuesta ya contiene medios o una directiva `MEDIA:`.
- omite respuestas muy cortas (< 10 caracteres).
- resume respuestas largas cuando está habilitado usando `agents.defaults.model.primary` (o `summaryModel`).
- adjunta el audio generado a la respuesta.

Si la respuesta supera `maxLength` y el resumen está desactivado (o no hay clave de API para el
modelo de resumen), se omite el audio y se envía la respuesta de texto normal.

## Diagrama de flujo

```
Reply -> TTS enabled?
  no  -> send text
  yes -> has media / MEDIA: / short?
          yes -> send text
          no  -> length > limit?
                   no  -> TTS -> attach audio
                   yes -> summary enabled?
                            no  -> send text
                            yes -> summarize (summaryModel or agents.defaults.model.primary)
                                      -> TTS -> attach audio
```

## Uso de comandos slash

Hay un único comando: `/tts`.
Consulta [Comandos slash](/tools/slash-commands) para ver detalles de habilitación.

Nota sobre Discord: `/tts` es un comando integrado de Discord, por lo que OpenClaw registra
`/voice` como comando nativo allí. El texto `/tts ...` sigue funcionando.

```
/tts off
/tts always
/tts inbound
/tts tagged
/tts status
/tts provider openai
/tts limit 2000
/tts summary off
/tts audio Hello from OpenClaw
```

Notas:

- Los comandos requieren un remitente autorizado (siguen aplicándose las reglas de allowlist/propietario).
- `commands.text` o el registro de comandos nativos deben estar habilitados.
- `off|always|inbound|tagged` son interruptores por sesión (`/tts on` es un alias de `/tts always`).
- `limit` y `summary` se almacenan en preferencias locales, no en la configuración principal.
- `/tts audio` genera una respuesta de audio única (no activa TTS).
- `/tts status` incluye visibilidad del respaldo para el intento más reciente:
  - respaldo exitoso: `Fallback: <primary> -> <used>` más `Attempts: ...`
  - fallo: `Error: ...` más `Attempts: ...`
  - diagnósticos detallados: `Attempt details: provider:outcome(reasonCode) latency`
- Los fallos de API de OpenAI y ElevenLabs ahora incluyen el detalle del error del proveedor analizado y el id de solicitud (cuando el proveedor lo devuelve), lo cual aparece en errores/registros de TTS.

## Herramienta del agente

La herramienta `tts` convierte texto a voz y devuelve un adjunto de audio para
la entrega de respuestas. Cuando el canal es Feishu, Matrix, Telegram o WhatsApp,
el audio se entrega como mensaje de voz en lugar de como archivo adjunto.

## RPC del Gateway

Métodos del Gateway:

- `tts.status`
- `tts.enable`
- `tts.disable`
- `tts.convert`
- `tts.setProvider`
- `tts.providers`
