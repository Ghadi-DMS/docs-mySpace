---
read_when:
    - Quieres usar modelos Google Gemini con OpenClaw
    - Necesitas el flujo de autenticación con clave de API u OAuth
summary: Configuración de Google Gemini (clave de API + OAuth, generación de imágenes, comprensión de medios, búsqueda web)
title: Google (Gemini)
x-i18n:
    generated_at: "2026-04-09T01:29:49Z"
    model: gpt-5.4
    provider: openai
    source_hash: fad2ff68987301bd86145fa6e10de8c7b38d5bd5dbcd13db9c883f7f5b9a4e01
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

1. Establece la clave de API:

```bash
openclaw onboard --auth-choice gemini-api-key
```

2. Establece un modelo predeterminado:

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

Un proveedor alternativo `google-gemini-cli` usa OAuth con PKCE en lugar de una clave de
API. Esta es una integración no oficial; algunos usuarios informan
restricciones de cuenta. Úsalo bajo tu propia responsabilidad.

- Modelo predeterminado: `google-gemini-cli/gemini-3-flash-preview`
- Alias: `gemini-cli`
- Requisito previo de instalación: Gemini CLI local disponible como `gemini`
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

Si las solicitudes OAuth de Gemini CLI fallan después del inicio de sesión, establece
`GOOGLE_CLOUD_PROJECT` o `GOOGLE_CLOUD_PROJECT_ID` en el host de la gateway y
vuelve a intentarlo.

Si el inicio de sesión falla antes de que comience el flujo del navegador, asegúrate de que el comando local `gemini`
esté instalado y en `PATH`. OpenClaw admite tanto instalaciones con Homebrew
como instalaciones globales con npm, incluidos diseños comunes de Windows/npm.

Notas sobre el uso de JSON de Gemini CLI:

- El texto de respuesta proviene del campo `response` del JSON de CLI.
- El uso vuelve a `stats` cuando la CLI deja `usage` vacío.
- `stats.cached` se normaliza en `cacheRead` de OpenClaw.
- Si falta `stats.input`, OpenClaw deriva los tokens de entrada a partir de
  `stats.input_tokens - stats.cached`.

## Capacidades

| Capacidad              | Compatible        |
| ---------------------- | ----------------- |
| Finalización de chat   | Sí                |
| Generación de imágenes | Sí                |
| Generación de música   | Sí                |
| Comprensión de imágenes| Sí                |
| Transcripción de audio | Sí                |
| Comprensión de video   | Sí                |
| Búsqueda web (Grounding) | Sí              |
| Thinking/reasoning     | Sí (Gemini 3.1+) |
| Modelos Gemma 4        | Sí                |

Los modelos Gemma 4 (por ejemplo `gemma-4-26b-a4b-it`) admiten el modo thinking. OpenClaw reescribe `thinkingBudget` a un `thinkingLevel` de Google compatible para Gemma 4. Establecer thinking en `off` mantiene thinking desactivado en lugar de asignarlo a `MINIMAL`.

## Reutilización directa de caché de Gemini

Para ejecuciones directas con la API de Gemini (`api: "google-generative-ai"`), OpenClaw ahora
pasa un identificador `cachedContent` configurado a las solicitudes de Gemini.

- Configura parámetros por modelo o globales con
  `cachedContent` o el heredado `cached_content`
- Si ambos están presentes, `cachedContent` tiene prioridad
- Valor de ejemplo: `cachedContents/prebuilt-context`
- El uso con aciertos de caché de Gemini se normaliza en `cacheRead` de OpenClaw a partir de
  `cachedContentTokenCount` upstream

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

El proveedor empaquetado de generación de imágenes `google` usa de forma predeterminada
`google/gemini-3.1-flash-image-preview`.

- También admite `google/gemini-3-pro-image-preview`
- Generación: hasta 4 imágenes por solicitud
- Modo edición: habilitado, hasta 5 imágenes de entrada
- Controles de geometría: `size`, `aspectRatio` y `resolution`

El proveedor `google-gemini-cli`, solo con OAuth, es una superficie independiente
de inferencia de texto. La generación de imágenes, la comprensión de medios y Gemini Grounding siguen en
el ID de proveedor `google`.

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

Consulta [Image Generation](/es/tools/image-generation) para ver los parámetros
compartidos de la herramienta, la selección de proveedor y el comportamiento de conmutación por error.

## Generación de video

El plugin empaquetado `google` también registra la generación de video mediante la herramienta compartida
`video_generate`.

- Modelo de video predeterminado: `google/veo-3.1-fast-generate-preview`
- Modos: texto a video, imagen a video y flujos de referencia de un solo video
- Admite `aspectRatio`, `resolution` y `audio`
- Límite actual de duración: **de 4 a 8 segundos**

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

Consulta [Video Generation](/es/tools/video-generation) para ver los parámetros
compartidos de la herramienta, la selección de proveedor y el comportamiento de conmutación por error.

## Generación de música

El plugin empaquetado `google` también registra la generación de música mediante la herramienta compartida
`music_generate`.

- Modelo de música predeterminado: `google/lyria-3-clip-preview`
- También admite `google/lyria-3-pro-preview`
- Controles del prompt: `lyrics` e `instrumental`
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

Consulta [Music Generation](/es/tools/music-generation) para ver los parámetros
compartidos de la herramienta, la selección de proveedor y el comportamiento de conmutación por error.

## Nota sobre el entorno

Si la Gateway se ejecuta como daemon (launchd/systemd), asegúrate de que `GEMINI_API_KEY`
esté disponible para ese proceso (por ejemplo, en `~/.openclaw/.env` o mediante
`env.shellEnv`).
