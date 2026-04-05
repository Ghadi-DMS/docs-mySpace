---
read_when:
    - Quieres realizar una llamada de voz saliente desde OpenClaw
    - Estás configurando o desarrollando el plugin de llamadas de voz
summary: 'Plugin Voice Call: llamadas salientes + entrantes mediante Twilio/Telnyx/Plivo (instalación del plugin + configuración + CLI)'
title: Plugin Voice Call
x-i18n:
    generated_at: "2026-04-05T12:50:50Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4e6d10c9fde6ce1f51637af285edc0c710e9cb7702231c0a91b527b721eaddc1
    source_path: plugins/voice-call.md
    workflow: 15
---

# Voice Call (plugin)

Llamadas de voz para OpenClaw mediante un plugin. Admite notificaciones salientes y
conversaciones de varios turnos con políticas de entrada.

Proveedores actuales:

- `twilio` (Programmable Voice + Media Streams)
- `telnyx` (Call Control v2)
- `plivo` (Voice API + transferencia XML + GetInput speech)
- `mock` (desarrollo/sin red)

Modelo mental rápido:

- Instala el plugin
- Reinicia la Gateway
- Configura en `plugins.entries.voice-call.config`
- Usa `openclaw voicecall ...` o la herramienta `voice_call`

## Dónde se ejecuta (local vs remoto)

El plugin Voice Call se ejecuta **dentro del proceso de Gateway**.

Si usas una Gateway remota, instala/configura el plugin en la **máquina que ejecuta la Gateway**, y luego reinicia la Gateway para cargarlo.

## Instalación

### Opción A: instalar desde npm (recomendado)

```bash
openclaw plugins install @openclaw/voice-call
```

Reinicia la Gateway después.

### Opción B: instalar desde una carpeta local (desarrollo, sin copiar)

```bash
PLUGIN_SRC=./path/to/local/voice-call-plugin
openclaw plugins install "$PLUGIN_SRC"
cd "$PLUGIN_SRC" && pnpm install
```

Reinicia la Gateway después.

## Configuración

Establece la configuración en `plugins.entries.voice-call.config`:

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        enabled: true,
        config: {
          provider: "twilio", // o "telnyx" | "plivo" | "mock"
          fromNumber: "+15550001234",
          toNumber: "+15550005678",

          twilio: {
            accountSid: "ACxxxxxxxx",
            authToken: "...",
          },

          telnyx: {
            apiKey: "...",
            connectionId: "...",
            // Clave pública del webhook de Telnyx desde el Telnyx Mission Control Portal
            // (cadena Base64; también se puede establecer mediante TELNYX_PUBLIC_KEY).
            publicKey: "...",
          },

          plivo: {
            authId: "MAxxxxxxxxxxxxxxxxxxxx",
            authToken: "...",
          },

          // Servidor webhook
          serve: {
            port: 3334,
            path: "/voice/webhook",
          },

          // Seguridad de webhook (recomendado para túneles/proxies)
          webhookSecurity: {
            allowedHosts: ["voice.example.com"],
            trustedProxyIPs: ["100.64.0.1"],
          },

          // Exposición pública (elige una)
          // publicUrl: "https://example.ngrok.app/voice/webhook",
          // tunnel: { provider: "ngrok" },
          // tailscale: { mode: "funnel", path: "/voice/webhook" }

          outbound: {
            defaultMode: "notify", // notify | conversation
          },

          streaming: {
            enabled: true,
            provider: "openai", // opcional; primer proveedor registrado de transcripción en tiempo real cuando no se establece
            streamPath: "/voice/stream",
            providers: {
              openai: {
                apiKey: "sk-...", // opcional si OPENAI_API_KEY está establecido
                model: "gpt-4o-transcribe",
                silenceDurationMs: 800,
                vadThreshold: 0.5,
              },
            },
            preStartTimeoutMs: 5000,
            maxPendingConnections: 32,
            maxPendingConnectionsPerIp: 4,
            maxConnections: 128,
          },
        },
      },
    },
  },
}
```

Notas:

- Twilio/Telnyx requieren una URL de webhook **accesible públicamente**.
- Plivo requiere una URL de webhook **accesible públicamente**.
- `mock` es un proveedor local para desarrollo (sin llamadas de red).
- Si las configuraciones antiguas siguen usando `provider: "log"`, `twilio.from` o claves heredadas `streaming.*` de OpenAI, ejecuta `openclaw doctor --fix` para reescribirlas.
- Telnyx requiere `telnyx.publicKey` (o `TELNYX_PUBLIC_KEY`) a menos que `skipSignatureVerification` sea true.
- `skipSignatureVerification` es solo para pruebas locales.
- Si usas la capa gratuita de ngrok, establece `publicUrl` en la URL exacta de ngrok; la verificación de firma siempre se aplica.
- `tunnel.allowNgrokFreeTierLoopbackBypass: true` permite webhooks de Twilio con firmas no válidas **solo** cuando `tunnel.provider="ngrok"` y `serve.bind` es loopback (agente local de ngrok). Úsalo solo para desarrollo local.
- Las URL de la capa gratuita de ngrok pueden cambiar o añadir comportamiento intersticial; si `publicUrl` cambia, las firmas de Twilio fallarán. Para producción, prefiere un dominio estable o Tailscale funnel.
- Valores predeterminados de seguridad para streaming:
  - `streaming.preStartTimeoutMs` cierra sockets que nunca envían un frame `start` válido.
- `streaming.maxPendingConnections` limita el total de sockets pre-start no autenticados.
- `streaming.maxPendingConnectionsPerIp` limita los sockets pre-start no autenticados por IP de origen.
- `streaming.maxConnections` limita el total de sockets de stream multimedia abiertos (pendientes + activos).
- El fallback en runtime sigue aceptando por ahora esas claves antiguas de voice-call, pero la ruta de reescritura es `openclaw doctor --fix` y la capa de compatibilidad es temporal.

## Transcripción en streaming

`streaming` selecciona un proveedor de transcripción en tiempo real para el audio en vivo de la llamada.

Comportamiento actual en runtime:

- `streaming.provider` es opcional. Si no se establece, Voice Call usa el primer
  proveedor registrado de transcripción en tiempo real.
- Hoy el proveedor integrado es OpenAI, registrado por el plugin integrado `openai`.
- La configuración raw propiedad del proveedor vive en `streaming.providers.<providerId>`.
- Si `streaming.provider` apunta a un proveedor no registrado, o no hay ningún proveedor
  de transcripción en tiempo real registrado, Voice Call registra una advertencia y
  omite el streaming multimedia en lugar de hacer fallar todo el plugin.

Valores predeterminados de transcripción en streaming de OpenAI:

- API key: `streaming.providers.openai.apiKey` o `OPENAI_API_KEY`
- model: `gpt-4o-transcribe`
- `silenceDurationMs`: `800`
- `vadThreshold`: `0.5`

Ejemplo:

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          streaming: {
            enabled: true,
            provider: "openai",
            streamPath: "/voice/stream",
            providers: {
              openai: {
                apiKey: "sk-...", // opcional si OPENAI_API_KEY está establecido
                model: "gpt-4o-transcribe",
                silenceDurationMs: 800,
                vadThreshold: 0.5,
              },
            },
          },
        },
      },
    },
  },
}
```

Las claves heredadas siguen migrándose automáticamente con `openclaw doctor --fix`:

- `streaming.sttProvider` → `streaming.provider`
- `streaming.openaiApiKey` → `streaming.providers.openai.apiKey`
- `streaming.sttModel` → `streaming.providers.openai.model`
- `streaming.silenceDurationMs` → `streaming.providers.openai.silenceDurationMs`
- `streaming.vadThreshold` → `streaming.providers.openai.vadThreshold`

## Recolector de llamadas obsoletas

Usa `staleCallReaperSeconds` para finalizar llamadas que nunca reciben un webhook terminal
(por ejemplo, llamadas en modo notify que nunca se completan). El valor predeterminado es `0`
(deshabilitado).

Rangos recomendados:

- **Producción:** `120`–`300` segundos para flujos de estilo notify.
- Mantén este valor **por encima de `maxDurationSeconds`** para que las llamadas normales puedan
  terminar. Un buen punto de partida es `maxDurationSeconds + 30–60` segundos.

Ejemplo:

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          maxDurationSeconds: 300,
          staleCallReaperSeconds: 360,
        },
      },
    },
  },
}
```

## Seguridad de webhook

Cuando un proxy o túnel se sitúa delante de la Gateway, el plugin reconstruye la
URL pública para la verificación de firmas. Estas opciones controlan qué encabezados reenviados son de confianza.

`webhookSecurity.allowedHosts` crea una lista de permitidos de hosts a partir de encabezados reenviados.

`webhookSecurity.trustForwardingHeaders` confía en los encabezados reenviados sin una lista de permitidos.

`webhookSecurity.trustedProxyIPs` solo confía en los encabezados reenviados cuando la IP remota
de la solicitud coincide con la lista.

La protección contra repetición de webhooks está habilitada para Twilio y Plivo. Las solicitudes de webhook
válidas repetidas se reconocen, pero se omiten sus efectos secundarios.

Los turnos de conversación de Twilio incluyen un token por turno en los callbacks de `<Gather>`, por lo que
los callbacks de voz obsoletos/repetidos no pueden satisfacer un turno de transcripción pendiente más reciente.

Las solicitudes webhook no autenticadas se rechazan antes de leer el cuerpo cuando faltan los
encabezados de firma requeridos por el proveedor.

El webhook de voice-call usa el perfil compartido de cuerpo previo a la autenticación (64 KB / 5 segundos)
más un límite por IP de solicitudes en curso antes de la verificación de firma.

Ejemplo con un host público estable:

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          publicUrl: "https://voice.example.com/voice/webhook",
          webhookSecurity: {
            allowedHosts: ["voice.example.com"],
          },
        },
      },
    },
  },
}
```

## TTS para llamadas

Voice Call usa la configuración principal `messages.tts` para
audio sintetizado en streaming en llamadas. Puedes sobrescribirla en la configuración del plugin con la
**misma forma**: se fusiona en profundidad con `messages.tts`.

```json5
{
  tts: {
    provider: "elevenlabs",
    providers: {
      elevenlabs: {
        voiceId: "pMsXgVXv3BLzUgSXRplE",
        modelId: "eleven_multilingual_v2",
      },
    },
  },
}
```

Notas:

- Las claves heredadas `tts.<provider>` dentro de la configuración del plugin (`openai`, `elevenlabs`, `microsoft`, `edge`) se migran automáticamente a `tts.providers.<provider>` al cargarse. Prefiere la forma `providers` en la configuración confirmada.
- **Microsoft speech se ignora para las llamadas de voz** (el audio de telefonía necesita PCM; el transporte actual de Microsoft no expone salida PCM para telefonía).
- El TTS del núcleo se usa cuando el media streaming de Twilio está habilitado; en caso contrario, las llamadas usan las voces nativas del proveedor.
- Si ya hay un media stream activo de Twilio, Voice Call no vuelve a TwiML `<Say>`. Si el TTS de telefonía no está disponible en ese estado, la solicitud de reproducción falla en lugar de mezclar dos rutas de reproducción.
- Cuando el TTS de telefonía vuelve a un proveedor secundario, Voice Call registra una advertencia con la cadena de proveedores (`from`, `to`, `attempts`) para depuración.

### Más ejemplos

Usar solo el TTS del núcleo (sin sobrescritura):

```json5
{
  messages: {
    tts: {
      provider: "openai",
      providers: {
        openai: { voice: "alloy" },
      },
    },
  },
}
```

Sobrescribir a ElevenLabs solo para llamadas (mantener el valor predeterminado del núcleo en otros sitios):

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          tts: {
            provider: "elevenlabs",
            providers: {
              elevenlabs: {
                apiKey: "elevenlabs_key",
                voiceId: "pMsXgVXv3BLzUgSXRplE",
                modelId: "eleven_multilingual_v2",
              },
            },
          },
        },
      },
    },
  },
}
```

Sobrescribir solo el modelo de OpenAI para llamadas (ejemplo de deep-merge):

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          tts: {
            providers: {
              openai: {
                model: "gpt-4o-mini-tts",
                voice: "marin",
              },
            },
          },
        },
      },
    },
  },
}
```

## Llamadas entrantes

La política de entrada usa `disabled` de forma predeterminada. Para habilitar llamadas entrantes, establece:

```json5
{
  inboundPolicy: "allowlist",
  allowFrom: ["+15550001234"],
  inboundGreeting: "Hello! How can I help?",
}
```

`inboundPolicy: "allowlist"` es un filtro de caller-ID de baja garantía. El plugin
normaliza el valor `From` proporcionado por el proveedor y lo compara con `allowFrom`.
La verificación de webhooks autentica la entrega del proveedor y la integridad de la carga, pero
no demuestra la propiedad del número llamante PSTN/VoIP. Trata `allowFrom` como
filtrado de caller-ID, no como identidad fuerte del llamante.

Las respuestas automáticas usan el sistema de agentes. Ajústalo con:

- `responseModel`
- `responseSystemPrompt`
- `responseTimeoutMs`

### Contrato de salida hablada

Para respuestas automáticas, Voice Call añade un contrato estricto de salida hablada al system prompt:

- `{"spoken":"..."}`

Luego Voice Call extrae el texto hablado de forma defensiva:

- Ignora cargas marcadas como contenido de razonamiento/error.
- Analiza JSON directo, JSON cercado o claves `"spoken"` inline.
- Vuelve a texto sin formato y elimina párrafos iniciales probables de planificación/meta.

Esto mantiene la reproducción hablada centrada en el texto dirigido a quien llama y evita filtrar texto de planificación al audio.

### Comportamiento al iniciar la conversación

Para llamadas salientes `conversation`, el manejo del primer mensaje está vinculado al estado de reproducción en vivo:

- La limpieza de cola por barge-in y la respuesta automática se suprimen solo mientras el saludo inicial se está reproduciendo activamente.
- Si falla la reproducción inicial, la llamada vuelve a `listening` y el mensaje inicial queda en cola para reintento.
- La reproducción inicial para streaming de Twilio empieza al conectarse el stream sin retraso adicional.

### Margen de desconexión del stream de Twilio

Cuando un media stream de Twilio se desconecta, Voice Call espera `2000ms` antes de finalizar automáticamente la llamada:

- Si el stream se reconecta durante esa ventana, se cancela el final automático.
- Si no se vuelve a registrar ningún stream después del margen, la llamada se finaliza para evitar llamadas activas atascadas.

## CLI

```bash
openclaw voicecall call --to "+15555550123" --message "Hello from OpenClaw"
openclaw voicecall start --to "+15555550123"   # alias de call
openclaw voicecall continue --call-id <id> --message "Any questions?"
openclaw voicecall speak --call-id <id> --message "One moment"
openclaw voicecall end --call-id <id>
openclaw voicecall status --call-id <id>
openclaw voicecall tail
openclaw voicecall latency                     # resumir la latencia de turnos a partir de registros
openclaw voicecall expose --mode funnel
```

`latency` lee `calls.jsonl` desde la ruta de almacenamiento predeterminada de voice-call. Usa
`--file <path>` para apuntar a otro registro y `--last <n>` para limitar el análisis
a los últimos N registros (predeterminado 200). La salida incluye p50/p90/p99 para la latencia
de turnos y tiempos de espera de escucha.

## Herramienta del agente

Nombre de la herramienta: `voice_call`

Acciones:

- `initiate_call` (message, to?, mode?)
- `continue_call` (callId, message)
- `speak_to_user` (callId, message)
- `end_call` (callId)
- `get_status` (callId)

Este repositorio incluye una Skill document correspondiente en `skills/voice-call/SKILL.md`.

## RPC de Gateway

- `voicecall.initiate` (`to?`, `message`, `mode?`)
- `voicecall.continue` (`callId`, `message`)
- `voicecall.speak` (`callId`, `message`)
- `voicecall.end` (`callId`)
- `voicecall.status` (`callId`)
