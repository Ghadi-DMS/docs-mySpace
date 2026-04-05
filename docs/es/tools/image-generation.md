---
read_when:
    - Generar imágenes mediante el agente
    - Configurar proveedores y modelos de generación de imágenes
    - Comprender los parámetros de la herramienta `image_generate`
summary: Genera y edita imágenes usando proveedores configurados (OpenAI, Google Gemini, fal, MiniMax)
title: Generación de imágenes
x-i18n:
    generated_at: "2026-04-05T12:55:46Z"
    model: gpt-5.4
    provider: openai
    source_hash: d38a8a583997ceff6523ce4f51808c97a2b59fe4e5a34cf79cdcb70d7e83aec2
    source_path: tools/image-generation.md
    workflow: 15
---

# Generación de imágenes

La herramienta `image_generate` permite al agente crear y editar imágenes usando tus proveedores configurados. Las imágenes generadas se entregan automáticamente como archivos multimedia adjuntos en la respuesta del agente.

<Note>
La herramienta solo aparece cuando hay disponible al menos un proveedor de generación de imágenes. Si no ves `image_generate` en las herramientas de tu agente, configura `agents.defaults.imageGenerationModel` o establece una clave API de proveedor.
</Note>

## Inicio rápido

1. Establece una clave API para al menos un proveedor (por ejemplo `OPENAI_API_KEY` o `GEMINI_API_KEY`).
2. Opcionalmente, establece tu modelo preferido:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: "openai/gpt-image-1",
    },
  },
}
```

3. Pídele al agente: _"Genera una imagen de una amigable mascota langosta."_

El agente llama automáticamente a `image_generate`. No se necesita lista de permitidos de herramientas: está habilitada por defecto cuando hay un proveedor disponible.

## Proveedores compatibles

| Proveedor | Modelo predeterminado             | Compatibilidad con edición | Clave API                                              |
| --------- | --------------------------------- | -------------------------- | ------------------------------------------------------ |
| OpenAI    | `gpt-image-1`                     | Sí (hasta 5 imágenes)      | `OPENAI_API_KEY`                                       |
| Google    | `gemini-3.1-flash-image-preview`  | Sí                         | `GEMINI_API_KEY` o `GOOGLE_API_KEY`                    |
| fal       | `fal-ai/flux/dev`                 | Sí                         | `FAL_KEY`                                              |
| MiniMax   | `image-01`                        | Sí (referencia de sujeto)  | `MINIMAX_API_KEY` o OAuth de MiniMax (`minimax-portal`) |

Usa `action: "list"` para inspeccionar los proveedores y modelos disponibles en tiempo de ejecución:

```
/tool image_generate action=list
```

## Parámetros de la herramienta

| Parámetro     | Tipo      | Descripción                                                                           |
| ------------- | --------- | ------------------------------------------------------------------------------------- |
| `prompt`      | string    | Prompt de generación de imagen (obligatorio para `action: "generate"`)               |
| `action`      | string    | `"generate"` (predeterminado) o `"list"` para inspeccionar proveedores                |
| `model`       | string    | Sobrescritura de proveedor/modelo, por ejemplo `openai/gpt-image-1`                   |
| `image`       | string    | Ruta o URL de una sola imagen de referencia para el modo de edición                   |
| `images`      | string[]  | Varias rutas de imágenes de referencia para el modo de edición (hasta 5)              |
| `size`        | string    | Sugerencia de tamaño: `1024x1024`, `1536x1024`, `1024x1536`, `1024x1792`, `1792x1024` |
| `aspectRatio` | string    | Relación de aspecto: `1:1`, `2:3`, `3:2`, `3:4`, `4:3`, `4:5`, `5:4`, `9:16`, `16:9`, `21:9` |
| `resolution`  | string    | Sugerencia de resolución: `1K`, `2K` o `4K`                                           |
| `count`       | number    | Número de imágenes a generar (1–4)                                                    |
| `filename`    | string    | Sugerencia de nombre de archivo de salida                                             |

No todos los proveedores admiten todos los parámetros. La herramienta pasa lo que admite cada proveedor e ignora el resto.

## Configuración

### Selección de modelo

```json5
{
  agents: {
    defaults: {
      // String form: primary model only
      imageGenerationModel: "google/gemini-3.1-flash-image-preview",

      // Object form: primary + ordered fallbacks
      imageGenerationModel: {
        primary: "openai/gpt-image-1",
        fallbacks: ["google/gemini-3.1-flash-image-preview", "fal/fal-ai/flux/dev"],
      },
    },
  },
}
```

### Orden de selección de proveedor

Al generar una imagen, OpenClaw prueba los proveedores en este orden:

1. **Parámetro `model`** de la llamada a la herramienta (si el agente especifica uno)
2. **`imageGenerationModel.primary`** de la configuración
3. **`imageGenerationModel.fallbacks`** en orden
4. **Detección automática**: usa solo los valores predeterminados de proveedores respaldados por autenticación:
   - primero el proveedor predeterminado actual
   - después los demás proveedores registrados de generación de imágenes en orden de id de proveedor

Si un proveedor falla (error de autenticación, límite de tasa, etc.), se prueba automáticamente el siguiente candidato. Si todos fallan, el error incluye detalles de cada intento.

Notas:

- La detección automática tiene en cuenta la autenticación. Un valor predeterminado de proveedor solo entra en la lista de candidatos cuando OpenClaw realmente puede autenticar ese proveedor.
- Usa `action: "list"` para inspeccionar los proveedores registrados actualmente, sus modelos predeterminados y las sugerencias de variables de entorno de autenticación.

### Edición de imágenes

OpenAI, Google, fal y MiniMax admiten edición de imágenes de referencia. Pasa una ruta o URL de imagen de referencia:

```
"Genera una versión en acuarela de esta foto" + image: "/path/to/photo.jpg"
```

OpenAI y Google admiten hasta 5 imágenes de referencia mediante el parámetro `images`. fal y MiniMax admiten 1.

La generación de imágenes de MiniMax está disponible mediante ambas rutas de autenticación incluidas de MiniMax:

- `minimax/image-01` para configuraciones con clave API
- `minimax-portal/image-01` para configuraciones con OAuth

## Capacidades del proveedor

| Capacidad             | OpenAI               | Google               | fal                 | MiniMax                      |
| --------------------- | -------------------- | -------------------- | ------------------- | ---------------------------- |
| Generar               | Sí (hasta 4)         | Sí (hasta 4)         | Sí (hasta 4)        | Sí (hasta 9)                 |
| Edición/referencia    | Sí (hasta 5 imágenes) | Sí (hasta 5 imágenes) | Sí (1 imagen)       | Sí (1 imagen, referencia de sujeto) |
| Control de tamaño     | Sí                   | Sí                   | Sí                  | No                           |
| Relación de aspecto   | No                   | Sí                   | Sí (solo generación) | Sí                          |
| Resolución (1K/2K/4K) | No                   | Sí                   | Sí                  | No                           |

## Relacionado

- [Descripción general de herramientas](/tools) — todas las herramientas de agente disponibles
- [Referencia de configuración](/es/gateway/configuration-reference#agent-defaults) — configuración de `imageGenerationModel`
- [Modelos](/es/concepts/models) — configuración de modelos y failover
