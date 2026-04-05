---
read_when:
    - Diseñar o refactorizar la comprensión multimedia
    - Ajustar el preprocesamiento entrante de audio/video/imagen
summary: Comprensión entrante de imagen/audio/video (opcional) con respaldos de proveedor + CLI
title: Comprensión multimedia
x-i18n:
    generated_at: "2026-04-05T12:47:51Z"
    model: gpt-5.4
    provider: openai
    source_hash: fe36bd42250d48d12f4ff549e8644afa7be8e42ee51f8aff4f21f81b7ff060f4
    source_path: nodes/media-understanding.md
    workflow: 15
---

# Comprensión multimedia - Entrante (2026-01-17)

OpenClaw puede **resumir contenido multimedia entrante** (imagen/audio/video) antes de que se ejecute el pipeline de respuesta. Detecta automáticamente cuándo hay herramientas locales o claves de proveedor disponibles, y puede deshabilitarse o personalizarse. Si la comprensión está desactivada, los modelos siguen recibiendo los archivos/URLs originales como de costumbre.

El comportamiento multimedia específico de cada proveedor se registra mediante plugins del proveedor, mientras que el núcleo de OpenClaw
posee la configuración compartida `tools.media`, el orden de respaldo y la
integración con el pipeline de respuesta.

## Objetivos

- Opcional: resumir previamente el contenido multimedia entrante en texto corto para un enrutamiento más rápido y un mejor análisis de comandos.
- Conservar siempre la entrega del contenido multimedia original al modelo.
- Compatibilidad con **APIs de proveedores** y **respaldos de CLI**.
- Permitir varios modelos con respaldo ordenado (error/tamaño/timeout).

## Comportamiento de alto nivel

1. Recopilar archivos adjuntos entrantes (`MediaPaths`, `MediaUrls`, `MediaTypes`).
2. Para cada capacidad habilitada (imagen/audio/video), seleccionar archivos adjuntos según la política (predeterminado: **primero**).
3. Elegir la primera entrada de modelo elegible (tamaño + capacidad + autenticación).
4. Si un modelo falla o el contenido multimedia es demasiado grande, **recurrir a la siguiente entrada**.
5. En caso de éxito:
   - `Body` se convierte en un bloque `[Image]`, `[Audio]` o `[Video]`.
   - El audio establece `{{Transcript}}`; el análisis de comandos usa el texto del subtítulo cuando está presente,
     y en caso contrario la transcripción.
   - Los subtítulos se conservan como `User text:` dentro del bloque.

Si la comprensión falla o está deshabilitada, **el flujo de respuesta continúa** con el cuerpo original + archivos adjuntos.

## Resumen de configuración

`tools.media` admite **modelos compartidos** más reemplazos por capacidad:

- `tools.media.models`: lista de modelos compartidos (usa `capabilities` para controlarlos).
- `tools.media.image` / `tools.media.audio` / `tools.media.video`:
  - valores predeterminados (`prompt`, `maxChars`, `maxBytes`, `timeoutSeconds`, `language`)
  - reemplazos de proveedor (`baseUrl`, `headers`, `providerOptions`)
  - opciones de audio Deepgram mediante `tools.media.audio.providerOptions.deepgram`
  - controles de eco de transcripción de audio (`echoTranscript`, predeterminado `false`; `echoFormat`)
  - **lista opcional de `models` por capacidad** (tiene prioridad sobre los modelos compartidos)
  - política de `attachments` (`mode`, `maxAttachments`, `prefer`)
  - `scope` (control opcional por canal/chatType/clave de sesión)
- `tools.media.concurrency`: máximo de ejecuciones simultáneas por capacidad (predeterminado **2**).

```json5
{
  tools: {
    media: {
      models: [
        /* lista compartida */
      ],
      image: {
        /* reemplazos opcionales */
      },
      audio: {
        /* reemplazos opcionales */
        echoTranscript: true,
        echoFormat: '📝 "{transcript}"',
      },
      video: {
        /* reemplazos opcionales */
      },
    },
  },
}
```

### Entradas de modelo

Cada entrada de `models[]` puede ser de **proveedor** o de **CLI**:

```json5
{
  type: "provider", // predeterminado si se omite
  provider: "openai",
  model: "gpt-5.4-mini",
  prompt: "Describe la imagen en <= 500 caracteres.",
  maxChars: 500,
  maxBytes: 10485760,
  timeoutSeconds: 60,
  capabilities: ["image"], // opcional, se usa para entradas multimodales
  profile: "vision-profile",
  preferredProfile: "vision-fallback",
}
```

```json5
{
  type: "cli",
  command: "gemini",
  args: [
    "-m",
    "gemini-3-flash",
    "--allowed-tools",
    "read_file",
    "Lee el contenido multimedia en {{MediaPath}} y descríbelo en <= {{MaxChars}} caracteres.",
  ],
  maxChars: 500,
  maxBytes: 52428800,
  timeoutSeconds: 120,
  capabilities: ["video", "image"],
}
```

Las plantillas de CLI también pueden usar:

- `{{MediaDir}}` (directorio que contiene el archivo multimedia)
- `{{OutputDir}}` (directorio temporal creado para esta ejecución)
- `{{OutputBase}}` (ruta base del archivo temporal, sin extensión)

## Valores predeterminados y límites

Valores predeterminados recomendados:

- `maxChars`: **500** para imagen/video (corto y apto para comandos)
- `maxChars`: **sin definir** para audio (transcripción completa a menos que definas un límite)
- `maxBytes`:
  - imagen: **10MB**
  - audio: **20MB**
  - video: **50MB**

Reglas:

- Si el contenido multimedia supera `maxBytes`, ese modelo se omite y **se prueba el siguiente modelo**.
- Los archivos de audio menores de **1024 bytes** se tratan como vacíos/corruptos y se omiten antes de la transcripción por proveedor/CLI.
- Si el modelo devuelve más de `maxChars`, la salida se recorta.
- `prompt` usa por defecto algo simple como “Describe el/la {media}.” más la guía de `maxChars` (solo imagen/video).
- Si el modelo principal de imagen activo ya admite visión de forma nativa, OpenClaw
  omite el bloque de resumen `[Image]` y en su lugar pasa la imagen original al
  modelo.
- Si `<capability>.enabled: true` pero no hay modelos configurados, OpenClaw prueba el
  **modelo de respuesta activo** cuando su proveedor admite esa capacidad.

### Detección automática de comprensión multimedia (predeterminada)

Si `tools.media.<capability>.enabled` **no** está definido en `false` y no has
configurado modelos, OpenClaw detecta automáticamente en este orden y **se detiene en la primera
opción que funciona**:

1. **Modelo de respuesta activo** cuando su proveedor admite la capacidad.
2. Referencias principales/de respaldo de **`agents.defaults.imageModel`** (solo imagen).
3. **CLI locales** (solo audio; si están instaladas)
   - `sherpa-onnx-offline` (requiere `SHERPA_ONNX_MODEL_DIR` con encoder/decoder/joiner/tokens)
   - `whisper-cli` (`whisper-cpp`; usa `WHISPER_CPP_MODEL` o el modelo tiny incluido)
   - `whisper` (CLI de Python; descarga modelos automáticamente)
4. **Gemini CLI** (`gemini`) usando `read_many_files`
5. **Autenticación de proveedor**
   - Las entradas configuradas en `models.providers.*` que admiten la capacidad se
     prueban antes que el orden de respaldo incluido.
   - Los proveedores configurados solo de imagen con un modelo compatible con imagen se autorregistran para
     comprensión multimedia incluso cuando no son un plugin de proveedor incluido.
   - Orden de respaldo incluido:
     - Audio: OpenAI → Groq → Deepgram → Google → Mistral
     - Imagen: OpenAI → Anthropic → Google → MiniMax → MiniMax Portal → Z.AI
     - Video: Google → Qwen → Moonshot

Para deshabilitar la detección automática, define:

```json5
{
  tools: {
    media: {
      audio: {
        enabled: false,
      },
    },
  },
}
```

Nota: la detección de binarios es best-effort en macOS/Linux/Windows; asegúrate de que la CLI esté en `PATH` (expandimos `~`), o define un modelo CLI explícito con una ruta completa del comando.

### Compatibilidad con entorno proxy (modelos de proveedor)

Cuando la comprensión multimedia **de audio** y **video** basada en proveedor está habilitada, OpenClaw
respeta las variables de entorno estándar de proxy saliente para llamadas HTTP del proveedor:

- `HTTPS_PROXY`
- `HTTP_PROXY`
- `https_proxy`
- `http_proxy`

Si no hay variables de entorno de proxy definidas, la comprensión multimedia usa salida directa.
Si el valor del proxy está mal formado, OpenClaw registra una advertencia y recurre a la
obtención directa.

## Capacidades (opcional)

Si defines `capabilities`, la entrada solo se ejecuta para esos tipos multimedia. Para listas compartidas,
OpenClaw puede inferir valores predeterminados:

- `openai`, `anthropic`, `minimax`: **image**
- `minimax-portal`: **image**
- `moonshot`: **image + video**
- `openrouter`: **image**
- `google` (Gemini API): **image + audio + video**
- `qwen`: **image + video**
- `mistral`: **audio**
- `zai`: **image**
- `groq`: **audio**
- `deepgram`: **audio**
- Cualquier catálogo `models.providers.<id>.models[]` con un modelo compatible con imagen:
  **image**

Para entradas CLI, **define `capabilities` explícitamente** para evitar coincidencias inesperadas.
Si omites `capabilities`, la entrada es elegible para la lista en la que aparece.

## Matriz de compatibilidad de proveedores (integraciones de OpenClaw)

| Capacidad | Integración de proveedor                                                            | Notas                                                                                                                                    |
| --------- | ------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------- |
| Imagen    | OpenAI, OpenRouter, Anthropic, Google, MiniMax, Moonshot, Qwen, Z.AI, proveedores configurados | Los plugins de proveedor registran la compatibilidad con imagen; MiniMax y MiniMax OAuth usan `MiniMax-VL-01`; los proveedores configurados compatibles con imagen se autorregistran. |
| Audio     | OpenAI, Groq, Deepgram, Google, Mistral                                              | Transcripción por proveedor (Whisper/Deepgram/Gemini/Voxtral).                                                                           |
| Video     | Google, Qwen, Moonshot                                                               | Comprensión de video por proveedor mediante plugins del proveedor; la comprensión de video de Qwen usa los endpoints Standard DashScope. |

Nota sobre MiniMax:

- La comprensión de imagen de `minimax` y `minimax-portal` proviene del proveedor multimedia
  `MiniMax-VL-01`, propiedad del plugin.
- El catálogo de texto incluido de MiniMax sigue empezando solo con texto; las entradas explícitas
  `models.providers.minimax` materializan referencias M2.7 de chat compatibles con imagen.

## Guía de selección de modelos

- Prefiere el modelo más potente y de generación más reciente disponible para cada capacidad multimedia cuando la calidad y la seguridad sean importantes.
- Para agentes con herramientas habilitadas que manejan entradas no confiables, evita modelos multimedia más antiguos o débiles.
- Mantén al menos un respaldo por capacidad para disponibilidad (modelo de calidad + modelo más rápido/barato).
- Los respaldos de CLI (`whisper-cli`, `whisper`, `gemini`) son útiles cuando las APIs del proveedor no están disponibles.
- Nota sobre `parakeet-mlx`: con `--output-dir`, OpenClaw lee `<output-dir>/<media-basename>.txt` cuando el formato de salida es `txt` (o no se especifica); los formatos distintos de `txt` recurren a stdout.

## Política de archivos adjuntos

`attachments` por capacidad controla qué archivos adjuntos se procesan:

- `mode`: `first` (predeterminado) o `all`
- `maxAttachments`: límite del número procesado (predeterminado **1**)
- `prefer`: `first`, `last`, `path`, `url`

Cuando `mode: "all"`, las salidas se etiquetan como `[Image 1/2]`, `[Audio 2/2]`, etc.

Comportamiento de extracción de archivos adjuntos:

- El texto extraído del archivo se envuelve como **contenido externo no confiable** antes de
  agregarse al prompt multimedia.
- El bloque inyectado usa marcadores de límite explícitos como
  `<<<EXTERNAL_UNTRUSTED_CONTENT id="...">>>` /
  `<<<END_EXTERNAL_UNTRUSTED_CONTENT id="...">>>` e incluye una línea de metadatos
  `Source: External`.
- Esta ruta de extracción de archivos adjuntos omite intencionalmente el banner largo
  `SECURITY NOTICE:` para evitar inflar el prompt multimedia; los marcadores de límite
  y los metadatos se mantienen de todos modos.
- Si un archivo no tiene texto extraíble, OpenClaw inyecta `[No extractable text]`.
- Si un PDF recurre a imágenes de páginas renderizadas en esta ruta, el prompt multimedia mantiene
  el marcador `[PDF content rendered to images; images not forwarded to model]`
  porque este paso de extracción de adjuntos reenvía bloques de texto, no las imágenes renderizadas del PDF.

## Ejemplos de configuración

### 1) Lista compartida de modelos + reemplazos

```json5
{
  tools: {
    media: {
      models: [
        { provider: "openai", model: "gpt-5.4-mini", capabilities: ["image"] },
        {
          provider: "google",
          model: "gemini-3-flash-preview",
          capabilities: ["image", "audio", "video"],
        },
        {
          type: "cli",
          command: "gemini",
          args: [
            "-m",
            "gemini-3-flash",
            "--allowed-tools",
            "read_file",
            "Lee el contenido multimedia en {{MediaPath}} y descríbelo en <= {{MaxChars}} caracteres.",
          ],
          capabilities: ["image", "video"],
        },
      ],
      audio: {
        attachments: { mode: "all", maxAttachments: 2 },
      },
      video: {
        maxChars: 500,
      },
    },
  },
}
```

### 2) Solo audio + video (imagen desactivada)

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        models: [
          { provider: "openai", model: "gpt-4o-mini-transcribe" },
          {
            type: "cli",
            command: "whisper",
            args: ["--model", "base", "{{MediaPath}}"],
          },
        ],
      },
      video: {
        enabled: true,
        maxChars: 500,
        models: [
          { provider: "google", model: "gemini-3-flash-preview" },
          {
            type: "cli",
            command: "gemini",
            args: [
              "-m",
              "gemini-3-flash",
              "--allowed-tools",
              "read_file",
              "Lee el contenido multimedia en {{MediaPath}} y descríbelo en <= {{MaxChars}} caracteres.",
            ],
          },
        ],
      },
    },
  },
}
```

### 3) Comprensión opcional de imagen

```json5
{
  tools: {
    media: {
      image: {
        enabled: true,
        maxBytes: 10485760,
        maxChars: 500,
        models: [
          { provider: "openai", model: "gpt-5.4-mini" },
          { provider: "anthropic", model: "claude-opus-4-6" },
          {
            type: "cli",
            command: "gemini",
            args: [
              "-m",
              "gemini-3-flash",
              "--allowed-tools",
              "read_file",
              "Lee el contenido multimedia en {{MediaPath}} y descríbelo en <= {{MaxChars}} caracteres.",
            ],
          },
        ],
      },
    },
  },
}
```

### 4) Entrada única multimodal (capacidades explícitas)

```json5
{
  tools: {
    media: {
      image: {
        models: [
          {
            provider: "google",
            model: "gemini-3.1-pro-preview",
            capabilities: ["image", "video", "audio"],
          },
        ],
      },
      audio: {
        models: [
          {
            provider: "google",
            model: "gemini-3.1-pro-preview",
            capabilities: ["image", "video", "audio"],
          },
        ],
      },
      video: {
        models: [
          {
            provider: "google",
            model: "gemini-3.1-pro-preview",
            capabilities: ["image", "video", "audio"],
          },
        ],
      },
    },
  },
}
```

## Salida de estado

Cuando se ejecuta la comprensión multimedia, `/status` incluye una línea de resumen breve:

```
📎 Media: image ok (openai/gpt-5.4-mini) · audio skipped (maxBytes)
```

Esto muestra resultados por capacidad y el proveedor/modelo elegido cuando corresponde.

## Notas

- La comprensión es **best-effort**. Los errores no bloquean las respuestas.
- Los archivos adjuntos siguen pasándose a los modelos incluso cuando la comprensión está deshabilitada.
- Usa `scope` para limitar dónde se ejecuta la comprensión (por ejemplo, solo en DMs).

## Documentación relacionada

- [Configuración](/gateway/configuration)
- [Compatibilidad con imágenes y contenido multimedia](/nodes/images)
