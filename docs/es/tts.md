---
read_when:
    - Habilitar text-to-speech para respuestas
    - Configurar proveedores o límites de TTS
    - Usar comandos /tts
summary: Text-to-speech (TTS) para respuestas salientes
title: Text-to-Speech (ruta heredada)
x-i18n:
    generated_at: "2026-04-05T12:57:39Z"
    model: gpt-5.4
    provider: openai
    source_hash: acca61773996299a582ab88e5a5db12d8f22ce8a28292ce97cc5dd5fdc2d3b83
    source_path: tts.md
    workflow: 15
---

# Text-to-speech (TTS)

OpenClaw puede convertir respuestas salientes en audio usando ElevenLabs, Microsoft, MiniMax u OpenAI.
Funciona en cualquier lugar donde OpenClaw pueda enviar audio.

## Servicios compatibles

- **ElevenLabs** (proveedor principal o de respaldo)
- **Microsoft** (proveedor principal o de respaldo; la implementación integrada actual usa `node-edge-tts`)
- **MiniMax** (proveedor principal o de respaldo; usa la API T2A v2)
- **OpenAI** (proveedor principal o de respaldo; también se usa para resúmenes)

### Notas sobre Microsoft speech

El proveedor integrado de speech de Microsoft usa actualmente el servicio
neural TTS en línea de Microsoft Edge mediante la librería `node-edge-tts`. Es un servicio alojado (no
local), usa endpoints de Microsoft y no requiere API key.
`node-edge-tts` expone opciones de configuración de voz y formatos de salida, pero
no todas las opciones son compatibles con el servicio. La configuración heredada y la entrada de directivas
que usan `edge` siguen funcionando y se normalizan a `microsoft`.

Como esta ruta es un servicio web público sin un SLA o cuota publicados,
trátala como best-effort. Si necesitas límites y soporte garantizados, usa OpenAI
o ElevenLabs.

## Claves opcionales

Si quieres OpenAI, ElevenLabs o MiniMax:

- `ELEVENLABS_API_KEY` (o `XI_API_KEY`)
- `MINIMAX_API_KEY`
- `OPENAI_API_KEY`

Microsoft speech **no** requiere API key.

Si se configuran varios proveedores, primero se usa el proveedor seleccionado y los demás actúan como opciones de respaldo.
El resumen automático usa el `summaryModel` configurado (o `agents.defaults.model.primary`),
por lo que ese proveedor también debe estar autenticado si habilitas los resúmenes.

## Enlaces de servicios

- [Guía de Text-to-Speech de OpenAI](https://platform.openai.com/docs/guides/text-to-speech)
- [Referencia de la API de Audio de OpenAI](https://platform.openai.com/docs/api-reference/audio)
- [Text to Speech de ElevenLabs](https://elevenlabs.io/docs/api-reference/text-to-speech)
- [Autenticación de ElevenLabs](https://elevenlabs.io/docs/api-reference/authentication)
- [API MiniMax T2A v2](https://platform.minimaxi.com/document/T2A%20V2)
- [node-edge-tts](https://github.com/SchneeHertz/node-edge-tts)
- [Formatos de salida de Microsoft Speech](https://learn.microsoft.com/azure/ai-services/speech-service/rest-text-to-speech#audio-outputs)

## ¿Está habilitado de forma predeterminada?

No. El auto‑TTS está **desactivado** de forma predeterminada. Actívalo en la configuración con
`messages.tts.auto` o por sesión con `/tts always` (alias: `/tts on`).

Cuando `messages.tts.provider` no está definido, OpenClaw elige el primer
proveedor de speech configurado según el orden de selección automática del registro.

## Configuración

La configuración de TTS se encuentra en `messages.tts` en `openclaw.json`.
El esquema completo está en [Configuración de Gateway](/es/gateway/configuration).

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

### Microsoft principal (sin API key)

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

### Desactivar Microsoft speech

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

### Desactivar el resumen automático para respuestas largas

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

- `auto`: modo auto‑TTS (`off`, `always`, `inbound`, `tagged`).
  - `inbound` solo envía audio después de un mensaje de voz entrante.
  - `tagged` solo envía audio cuando la respuesta incluye etiquetas `[[tts]]`.
- `enabled`: interruptor heredado (doctor lo migra a `auto`).
- `mode`: `"final"` (predeterminado) o `"all"` (incluye respuestas de herramientas/bloques).
- `provider`: id del proveedor de speech, como `"elevenlabs"`, `"microsoft"`, `"minimax"` u `"openai"` (el respaldo es automático).
- Si `provider` **no está definido**, OpenClaw usa el primer proveedor de speech configurado según el orden de selección automática del registro.
- El valor heredado `provider: "edge"` sigue funcionando y se normaliza a `microsoft`.
- `summaryModel`: modelo económico opcional para el resumen automático; por defecto es `agents.defaults.model.primary`.
  - Acepta `provider/model` o un alias de modelo configurado.
- `modelOverrides`: permite que el modelo emita directivas TTS (activado de forma predeterminada).
  - `allowProvider` tiene como valor predeterminado `false` (el cambio de proveedor es opt-in).
- `providers.<id>`: configuraciones propiedad del proveedor indexadas por id del proveedor de speech.
- Los bloques heredados directos del proveedor (`messages.tts.openai`, `messages.tts.elevenlabs`, `messages.tts.microsoft`, `messages.tts.edge`) se migran automáticamente a `messages.tts.providers.<id>` al cargar.
- `maxTextLength`: límite estricto para la entrada TTS (caracteres). `/tts audio` falla si se supera.
- `timeoutMs`: tiempo de espera de la solicitud (ms).
- `prefsPath`: sobrescribe la ruta local del JSON de preferencias (proveedor/límite/resumen).
- Los valores `apiKey` recurren a variables de entorno (`ELEVENLABS_API_KEY`/`XI_API_KEY`, `MINIMAX_API_KEY`, `OPENAI_API_KEY`).
- `providers.elevenlabs.baseUrl`: sobrescribe la URL base de la API de ElevenLabs.
- `providers.openai.baseUrl`: sobrescribe el endpoint TTS de OpenAI.
  - Orden de resolución: `messages.tts.providers.openai.baseUrl` -> `OPENAI_TTS_BASE_URL` -> `https://api.openai.com/v1`
  - Los valores no predeterminados se tratan como endpoints TTS compatibles con OpenAI, por lo que se aceptan nombres personalizados de modelo y voz.
- `providers.elevenlabs.voiceSettings`:
  - `stability`, `similarityBoost`, `style`: `0..1`
  - `useSpeakerBoost`: `true|false`
  - `speed`: `0.5..2.0` (1.0 = normal)
- `providers.elevenlabs.applyTextNormalization`: `auto|on|off`
- `providers.elevenlabs.languageCode`: ISO 639-1 de 2 letras (por ejemplo `en`, `de`)
- `providers.elevenlabs.seed`: entero `0..4294967295` (determinismo best-effort)
- `providers.minimax.baseUrl`: sobrescribe la URL base de la API de MiniMax (predeterminada `https://api.minimax.io`, env: `MINIMAX_API_HOST`).
- `providers.minimax.model`: modelo TTS (predeterminado `speech-2.8-hd`, env: `MINIMAX_TTS_MODEL`).
- `providers.minimax.voiceId`: identificador de voz (predeterminado `English_expressive_narrator`, env: `MINIMAX_TTS_VOICE_ID`).
- `providers.minimax.speed`: velocidad de reproducción `0.5..2.0` (predeterminada 1.0).
- `providers.minimax.vol`: volumen `(0, 10]` (predeterminado 1.0; debe ser mayor que 0).
- `providers.minimax.pitch`: desplazamiento de tono `-12..12` (predeterminado 0).
- `providers.microsoft.enabled`: permite el uso de Microsoft speech (predeterminado `true`; sin API key).
- `providers.microsoft.voice`: nombre de voz neural de Microsoft (por ejemplo `en-US-MichelleNeural`).
- `providers.microsoft.lang`: código de idioma (por ejemplo `en-US`).
- `providers.microsoft.outputFormat`: formato de salida de Microsoft (por ejemplo `audio-24khz-48kbitrate-mono-mp3`).
  - Consulta los formatos de salida de Microsoft Speech para ver los valores válidos; no todos los formatos son compatibles con el transporte integrado respaldado por Edge.
- `providers.microsoft.rate` / `providers.microsoft.pitch` / `providers.microsoft.volume`: cadenas de porcentaje (por ejemplo `+10%`, `-5%`).
- `providers.microsoft.saveSubtitles`: escribe subtítulos JSON junto al archivo de audio.
- `providers.microsoft.proxy`: URL proxy para solicitudes de Microsoft speech.
- `providers.microsoft.timeoutMs`: sobrescritura del tiempo de espera de la solicitud (ms).
- `edge.*`: alias heredado para la misma configuración de Microsoft.

## Sobrescrituras controladas por el modelo (activadas de forma predeterminada)

De forma predeterminada, el modelo **puede** emitir directivas TTS para una sola respuesta.
Cuando `messages.tts.auto` es `tagged`, estas directivas son necesarias para activar el audio.

Cuando está habilitado, el modelo puede emitir directivas `[[tts:...]]` para sobrescribir la voz
en una sola respuesta, además de un bloque opcional `[[tts:text]]...[[/tts:text]]` para
proporcionar etiquetas expresivas (risa, indicaciones para cantar, etc.) que solo deben aparecer en
el audio.

Las directivas `provider=...` se ignoran a menos que `modelOverrides.allowProvider: true`.

Ejemplo de carga útil de respuesta:

```
Here you go.

[[tts:voiceId=pMsXgVXv3BLzUgSXRplE model=eleven_v3 speed=1.1]]
[[tts:text]](laughs) Read the song once more.[[/tts:text]]
```

Claves de directiva disponibles (cuando están habilitadas):

- `provider` (id de proveedor de speech registrado, por ejemplo `openai`, `elevenlabs`, `minimax` o `microsoft`; requiere `allowProvider: true`)
- `voice` (voz de OpenAI) o `voiceId` (ElevenLabs / MiniMax)
- `model` (modelo TTS de OpenAI, id de modelo de ElevenLabs o modelo de MiniMax)
- `stability`, `similarityBoost`, `style`, `speed`, `useSpeakerBoost`
- `vol` / `volume` (volumen de MiniMax, 0-10)
- `pitch` (tono de MiniMax, -12 a 12)
- `applyTextNormalization` (`auto|on|off`)
- `languageCode` (ISO 639-1)
- `seed`

Desactiva todas las sobrescrituras del modelo:

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

Lista permitida opcional (habilita el cambio de proveedor mientras mantiene configurables los demás controles):

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

Los comandos slash escriben sobrescrituras locales en `prefsPath` (predeterminado:
`~/.openclaw/settings/tts.json`, sobrescribible con `OPENCLAW_TTS_PREFS` o
`messages.tts.prefsPath`).

Campos almacenados:

- `enabled`
- `provider`
- `maxLength` (umbral de resumen; predeterminado 1500 caracteres)
- `summarize` (predeterminado `true`)

Estos sobrescriben `messages.tts.*` para ese host.

## Formatos de salida (fijos)

- **Feishu / Matrix / Telegram / WhatsApp**: mensaje de voz Opus (`opus_48000_64` de ElevenLabs, `opus` de OpenAI).
  - 48kHz / 64kbps es una buena compensación para mensajes de voz.
- **Otros canales**: MP3 (`mp3_44100_128` de ElevenLabs, `mp3` de OpenAI).
  - 44.1kHz / 128kbps es el equilibrio predeterminado para la claridad del habla.
- **MiniMax**: MP3 (modelo `speech-2.8-hd`, frecuencia de muestreo de 32kHz). El formato de nota de voz no es compatible de forma nativa; usa OpenAI o ElevenLabs si necesitas mensajes de voz Opus garantizados.
- **Microsoft**: usa `microsoft.outputFormat` (predeterminado `audio-24khz-48kbitrate-mono-mp3`).
  - El transporte integrado acepta `outputFormat`, pero no todos los formatos están disponibles en el servicio.
  - Los valores de formato de salida siguen los formatos de salida de Microsoft Speech (incluidos Ogg/WebM Opus).
  - Telegram `sendVoice` acepta OGG/MP3/M4A; usa OpenAI/ElevenLabs si necesitas
    mensajes de voz Opus garantizados.
  - Si falla el formato de salida de Microsoft configurado, OpenClaw vuelve a intentar con MP3.

Los formatos de salida de OpenAI/ElevenLabs son fijos por canal (ver arriba).

## Comportamiento de auto-TTS

Cuando está habilitado, OpenClaw:

- omite TTS si la respuesta ya contiene medios o una directiva `MEDIA:`.
- omite respuestas muy cortas (< 10 caracteres).
- resume respuestas largas cuando está habilitado usando `agents.defaults.model.primary` (o `summaryModel`).
- adjunta el audio generado a la respuesta.

Si la respuesta supera `maxLength` y el resumen está desactivado (o no hay API key para el
modelo de resumen), se
omite el audio y se envía la respuesta de texto normal.

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

## Uso del comando slash

Hay un solo comando: `/tts`.
Consulta [Comandos slash](/tools/slash-commands) para ver los detalles de habilitación.

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

- Los comandos requieren un remitente autorizado (siguen aplicándose las reglas de lista permitida/propietario).
- `commands.text` o el registro de comandos nativos deben estar habilitados.
- `off|always|inbound|tagged` son interruptores por sesión (`/tts on` es un alias para `/tts always`).
- `limit` y `summary` se almacenan en preferencias locales, no en la configuración principal.
- `/tts audio` genera una respuesta de audio puntual (no activa TTS).
- `/tts status` incluye visibilidad de respaldo del intento más reciente:
  - respaldo exitoso: `Fallback: <primary> -> <used>` más `Attempts: ...`
  - error: `Error: ...` más `Attempts: ...`
  - diagnóstico detallado: `Attempt details: provider:outcome(reasonCode) latency`
- Los fallos de API de OpenAI y ElevenLabs ahora incluyen detalle analizado del error del proveedor e id de solicitud (cuando el proveedor lo devuelve), lo que se refleja en errores/registros de TTS.

## Herramienta del agente

La herramienta `tts` convierte texto en voz y devuelve un adjunto de audio para
la entrega de la respuesta. Cuando el canal es Feishu, Matrix, Telegram o WhatsApp,
el audio se entrega como un mensaje de voz en lugar de un archivo adjunto.

## RPC de Gateway

Métodos de Gateway:

- `tts.status`
- `tts.enable`
- `tts.disable`
- `tts.convert`
- `tts.setProvider`
- `tts.providers`
