---
read_when:
    - Quieres usar modelos Google Gemini con OpenClaw
    - Necesitas el flujo de autenticación por clave de API u OAuth
summary: Configuración de Google Gemini (clave de API + OAuth, generación de imágenes, comprensión de medios, búsqueda web)
title: Google (Gemini)
x-i18n:
    generated_at: "2026-04-05T12:51:17Z"
    model: gpt-5.4
    provider: openai
    source_hash: fa3c4326e83fad277ae4c2cb9501b6e89457afcfa7e3e1d57ae01c9c0c6846e2
    source_path: providers/google.md
    workflow: 15
---

# Google (Gemini)

El plugin de Google proporciona acceso a modelos Gemini a través de Google AI Studio, además de
generación de imágenes, comprensión de medios (imagen/audio/video) y búsqueda web mediante
Gemini Grounding.

- Proveedor: `google`
- Autenticación: `GEMINI_API_KEY` o `GOOGLE_API_KEY`
- API: Google Gemini API
- Proveedor alternativo: `google-gemini-cli` (OAuth)

## Inicio rápido

1. Configura la clave de API:

```bash
openclaw onboard --auth-choice gemini-api-key
```

2. Configura un modelo predeterminado:

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

Un proveedor alternativo, `google-gemini-cli`, usa PKCE OAuth en lugar de una
clave de API. Esta es una integración no oficial; algunos usuarios informan de
restricciones de cuenta. Úsala bajo tu propia responsabilidad.

- Modelo predeterminado: `google-gemini-cli/gemini-3.1-pro-preview`
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

(O las variantes `GEMINI_CLI_*`.)

Si las solicitudes de Gemini CLI OAuth fallan después del inicio de sesión, configura
`GOOGLE_CLOUD_PROJECT` o `GOOGLE_CLOUD_PROJECT_ID` en el host del gateway y
vuelve a intentarlo.

Si el inicio de sesión falla antes de que comience el flujo del navegador, asegúrate de que el comando local `gemini`
esté instalado y en `PATH`. OpenClaw admite tanto instalaciones con Homebrew
como instalaciones globales con npm, incluidos los diseños habituales de Windows/npm.

Notas de uso de JSON de Gemini CLI:

- El texto de respuesta proviene del campo JSON `response` de la CLI.
- El uso recurre a `stats` cuando la CLI deja `usage` vacío.
- `stats.cached` se normaliza a `cacheRead` de OpenClaw.
- Si falta `stats.input`, OpenClaw deriva los tokens de entrada a partir de
  `stats.input_tokens - stats.cached`.

## Capacidades

| Capacidad                | Compatible        |
| ------------------------ | ----------------- |
| Finalizaciones de chat   | Sí                |
| Generación de imágenes   | Sí                |
| Comprensión de imágenes  | Sí                |
| Transcripción de audio   | Sí                |
| Comprensión de video     | Sí                |
| Búsqueda web (Grounding) | Sí                |
| Thinking/razonamiento    | Sí (Gemini 3.1+)  |

## Reutilización directa de caché de Gemini

Para ejecuciones directas de la API de Gemini (`api: "google-generative-ai"`), OpenClaw ahora
pasa un identificador `cachedContent` configurado a las solicitudes de Gemini.

- Configura parámetros por modelo o globales usando
  `cachedContent` o el heredado `cached_content`
- Si ambos están presentes, `cachedContent` tiene prioridad
- Valor de ejemplo: `cachedContents/prebuilt-context`
- El uso de aciertos de caché de Gemini se normaliza a `cacheRead` de OpenClaw a partir de
  `cachedContentTokenCount` del upstream

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

El proveedor incluido de generación de imágenes `google` usa de forma predeterminada
`google/gemini-3.1-flash-image-preview`.

- También admite `google/gemini-3-pro-image-preview`
- Generación: hasta 4 imágenes por solicitud
- Modo edición: habilitado, hasta 5 imágenes de entrada
- Controles de geometría: `size`, `aspectRatio` y `resolution`

El proveedor `google-gemini-cli`, solo OAuth, es una superficie independiente
de inferencia de texto. La generación de imágenes, la comprensión de medios y Gemini Grounding permanecen en
el id de proveedor `google`.

## Nota sobre el entorno

Si el Gateway se ejecuta como daemon (launchd/systemd), asegúrate de que `GEMINI_API_KEY`
esté disponible para ese proceso (por ejemplo, en `~/.openclaw/.env` o mediante
`env.shellEnv`).
