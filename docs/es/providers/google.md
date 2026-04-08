---
read_when:
    - Quieres usar modelos Google Gemini con OpenClaw
    - Necesitas el flujo de autenticación con clave API u OAuth
summary: Configuración de Google Gemini (clave API + OAuth, generación de imágenes, comprensión de medios, búsqueda web)
title: Google (Gemini)
x-i18n:
    generated_at: "2026-04-08T02:17:24Z"
    model: gpt-5.4
    provider: openai
    source_hash: e9e558f5ce35c853e0240350be9a1890460c5f7f7fd30b05813a656497dee516
    source_path: providers/google.md
    workflow: 15
---

# Google (Gemini)

El plugin de Google proporciona acceso a modelos Gemini a través de Google AI Studio, además de
generación de imágenes, comprensión de medios (imagen/audio/video) y búsqueda web mediante
Gemini Grounding.

- Proveedor: `google`
- Autenticación: `GEMINI_API_KEY` o `GOOGLE_API_KEY`
- API: API de Google Gemini
- Proveedor alternativo: `google-gemini-cli` (OAuth)

## Inicio rápido

1. Define la clave API:

```bash
openclaw onboard --auth-choice gemini-api-key
```

2. Define un modelo predeterminado:

```json5
{
  agents: {
    defaults: {
      model: { primary: "google/gemini-3.1-pro-preview" },
    },
  },
}
```

## Ejemplo no interactivo

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice gemini-api-key \
  --gemini-api-key "$GEMINI_API_KEY"
```

## OAuth (Gemini CLI)

Un proveedor alternativo `google-gemini-cli` usa OAuth PKCE en lugar de una
clave API. Esta es una integración no oficial; algunos usuarios informan
restricciones en la cuenta. Úsala bajo tu propia responsabilidad.

- Modelo predeterminado: `google-gemini-cli/gemini-3-flash-preview`
- Alias: `gemini-cli`
- Requisito de instalación: Gemini CLI local disponible como `gemini`
  - Homebrew: `brew install gemini-cli`
  - npm: `npm install -g @google/gemini-cli`
- Inicio de sesión:

```bash
openclaw models auth login --provider google-gemini-cli --set-default
```

Variables de entorno:

- `OPENCLAW_GEMINI_OAUTH_CLIENT_ID`
- `OPENCLAW_GEMINI_OAUTH_CLIENT_SECRET`

(O las variantes `GEMINI_CLI_*`).

Si las solicitudes de OAuth de Gemini CLI fallan después de iniciar sesión, define
`GOOGLE_CLOUD_PROJECT` o `GOOGLE_CLOUD_PROJECT_ID` en el host de la gateway e
inténtalo de nuevo.

Si el inicio de sesión falla antes de que comience el flujo del navegador, asegúrate de que el comando local `gemini`
esté instalado y disponible en `PATH`. OpenClaw admite tanto instalaciones con Homebrew
como instalaciones globales de npm, incluidos diseños comunes de Windows/npm.

Notas de uso JSON de Gemini CLI:

- El texto de la respuesta proviene del campo JSON `response` de CLI.
- El uso recurre a `stats` cuando CLI deja `usage` vacío.
- `stats.cached` se normaliza a `cacheRead` de OpenClaw.
- Si falta `stats.input`, OpenClaw deriva los tokens de entrada de
  `stats.input_tokens - stats.cached`.

## Capacidades

| Capacidad              | Compatible        |
| ---------------------- | ----------------- |
| Finalizaciones de chat | Sí                |
| Generación de imágenes | Sí                |
| Generación de música   | Sí                |
| Comprensión de imagen  | Sí                |
| Transcripción de audio | Sí                |
| Comprensión de video   | Sí                |
| Búsqueda web (Grounding) | Sí              |
| Pensamiento/razonamiento | Sí (Gemini 3.1+) |

## Reutilización directa de caché de Gemini

Para ejecuciones directas de la API de Gemini (`api: "google-generative-ai"`), OpenClaw ahora
pasa un identificador configurado `cachedContent` a las solicitudes de Gemini.

- Configura parámetros por modelo o globales con
  `cachedContent` o el heredado `cached_content`
- Si ambos están presentes, `cachedContent` tiene prioridad
- Valor de ejemplo: `cachedContents/prebuilt-context`
- El uso por acierto de caché de Gemini se normaliza a `cacheRead` en OpenClaw a partir de
  `cachedContentTokenCount` ascendente

Ejemplo:

```json5
{
  agents: {
    defaults: {
      models: {
        "google/gemini-2.5-pro": {
          params: {
            cachedContent: "cachedContents/prebuilt-context",
          },
        },
      },
    },
  },
}
```

## Generación de imágenes

El proveedor integrado de generación de imágenes `google` usa de forma predeterminada
`google/gemini-3.1-flash-image-preview`.

- También admite `google/gemini-3-pro-image-preview`
- Generación: hasta 4 imágenes por solicitud
- Modo de edición: habilitado, hasta 5 imágenes de entrada
- Controles de geometría: `size`, `aspectRatio` y `resolution`

El proveedor `google-gemini-cli`, solo con OAuth, es una superficie separada
de inferencia de texto. La generación de imágenes, la comprensión de medios y Gemini Grounding permanecen en
el identificador de proveedor `google`.

Para usar Google como proveedor de imágenes predeterminado:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "google/gemini-3.1-flash-image-preview",
      },
    },
  },
}
```

Consulta [Image Generation](/es/tools/image-generation) para conocer los
parámetros compartidos de la herramienta, la selección de proveedor y el comportamiento de failover.

## Generación de video

El plugin integrado `google` también registra generación de video mediante la herramienta compartida
`video_generate`.

- Modelo de video predeterminado: `google/veo-3.1-fast-generate-preview`
- Modos: texto a video, imagen a video y flujos de referencia de un solo video
- Admite `aspectRatio`, `resolution` y `audio`
- Límite actual de duración: **4 a 8 segundos**

Para usar Google como proveedor de video predeterminado:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "google/veo-3.1-fast-generate-preview",
      },
    },
  },
}
```

Consulta [Video Generation](/es/tools/video-generation) para conocer los
parámetros compartidos de la herramienta, la selección de proveedor y el comportamiento de failover.

## Generación de música

El plugin integrado `google` también registra generación de música mediante la herramienta compartida
`music_generate`.

- Modelo de música predeterminado: `google/lyria-3-clip-preview`
- También admite `google/lyria-3-pro-preview`
- Controles de prompt: `lyrics` e `instrumental`
- Formato de salida: `mp3` de forma predeterminada, además de `wav` en `google/lyria-3-pro-preview`
- Entradas de referencia: hasta 10 imágenes
- Las ejecuciones respaldadas por sesión se desacoplan mediante el flujo compartido de tarea/estado, incluido `action: "status"`

Para usar Google como proveedor de música predeterminado:

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "google/lyria-3-clip-preview",
      },
    },
  },
}
```

Consulta [Music Generation](/es/tools/music-generation) para conocer los
parámetros compartidos de la herramienta, la selección de proveedor y el comportamiento de failover.

## Nota sobre el entorno

Si la gateway se ejecuta como daemon (launchd/systemd), asegúrate de que `GEMINI_API_KEY`
esté disponible para ese proceso (por ejemplo, en `~/.openclaw/.env` o mediante
`env.shellEnv`).
