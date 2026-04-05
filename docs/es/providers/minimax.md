---
read_when:
    - Quieres modelos MiniMax en OpenClaw
    - Necesitas orientación para configurar MiniMax
summary: Usar modelos MiniMax en OpenClaw
title: MiniMax
x-i18n:
    generated_at: "2026-04-05T12:51:49Z"
    model: gpt-5.4
    provider: openai
    source_hash: 353e1d9ce1b48c90ccaba6cc0109e839c473ca3e65d0c5d8ba744e9011c2bf45
    source_path: providers/minimax.md
    workflow: 15
---

# MiniMax

El proveedor MiniMax de OpenClaw usa por defecto **MiniMax M2.7**.

MiniMax también proporciona:

- síntesis de voz integrada mediante T2A v2
- comprensión de imágenes integrada mediante `MiniMax-VL-01`
- `web_search` integrado mediante la API de búsqueda de MiniMax Coding Plan

División del proveedor:

- `minimax`: proveedor de texto con API key, además de generación de imágenes, comprensión de imágenes, voz y búsqueda web integradas
- `minimax-portal`: proveedor de texto con OAuth, además de generación de imágenes y comprensión de imágenes integradas

## Gama de modelos

- `MiniMax-M2.7`: modelo de razonamiento alojado predeterminado.
- `MiniMax-M2.7-highspeed`: nivel de razonamiento M2.7 más rápido.
- `image-01`: modelo de generación de imágenes (generación y edición image-to-image).

## Generación de imágenes

El plugin MiniMax registra el modelo `image-01` para la herramienta `image_generate`. Admite:

- **Generación de texto a imagen** con control de relación de aspecto.
- **Edición image-to-image** (referencia de sujeto) con control de relación de aspecto.
- Hasta **9 imágenes de salida** por solicitud.
- Hasta **1 imagen de referencia** por solicitud de edición.
- Relaciones de aspecto admitidas: `1:1`, `16:9`, `4:3`, `3:2`, `2:3`, `3:4`, `9:16`, `21:9`.

Para usar MiniMax en generación de imágenes, establécelo como proveedor de generación de imágenes:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: { primary: "minimax/image-01" },
    },
  },
}
```

El plugin usa la misma `MINIMAX_API_KEY` o autenticación OAuth que los modelos de texto. No se necesita configuración adicional si MiniMax ya está configurado.

Tanto `minimax` como `minimax-portal` registran `image_generate` con el mismo
modelo `image-01`. Las configuraciones con API key usan `MINIMAX_API_KEY`; las configuraciones con OAuth pueden usar
en su lugar la ruta de autenticación integrada `minimax-portal`.

Cuando el onboarding o la configuración con API key escriben entradas explícitas `models.providers.minimax`,
OpenClaw materializa `MiniMax-M2.7` y
`MiniMax-M2.7-highspeed` con `input: ["text", "image"]`.

El catálogo de texto MiniMax integrado en sí mismo sigue siendo metadatos solo de texto hasta
que exista esa configuración explícita del proveedor. La comprensión de imágenes se expone por separado
a través del proveedor multimedia `MiniMax-VL-01` propiedad del plugin.

## Comprensión de imágenes

El plugin MiniMax registra la comprensión de imágenes por separado del
catálogo de texto:

- `minimax`: modelo de imagen predeterminado `MiniMax-VL-01`
- `minimax-portal`: modelo de imagen predeterminado `MiniMax-VL-01`

Por eso el enrutamiento automático de medios puede usar la comprensión de imágenes de MiniMax incluso
cuando el catálogo del proveedor de texto integrado sigue mostrando referencias de chat M2.7 solo de texto.

## Búsqueda web

El plugin MiniMax también registra `web_search` mediante la API de búsqueda de MiniMax Coding Plan.

- ID del proveedor: `minimax`
- Resultados estructurados: títulos, URL, fragmentos, consultas relacionadas
- Variable de entorno preferida: `MINIMAX_CODE_PLAN_KEY`
- Alias de entorno aceptado: `MINIMAX_CODING_API_KEY`
- Fallback de compatibilidad: `MINIMAX_API_KEY` cuando ya apunta a un token de coding-plan
- Reutilización de región: `plugins.entries.minimax.config.webSearch.region`, luego `MINIMAX_API_HOST`, luego las URL base del proveedor MiniMax
- La búsqueda permanece en el ID de proveedor `minimax`; la configuración OAuth CN/global aún puede dirigir indirectamente la región mediante `models.providers.minimax-portal.baseUrl`

La configuración vive en `plugins.entries.minimax.config.webSearch.*`.
Consulta [MiniMax Search](/tools/minimax-search).

## Elegir una configuración

### MiniMax OAuth (Coding Plan) - recomendado

**Mejor para:** configuración rápida con MiniMax Coding Plan mediante OAuth, sin necesidad de API key.

Autentícate con la opción OAuth regional explícita:

```bash
openclaw onboard --auth-choice minimax-global-oauth
# o
openclaw onboard --auth-choice minimax-cn-oauth
```

Correspondencia de opciones:

- `minimax-global-oauth`: usuarios internacionales (`api.minimax.io`)
- `minimax-cn-oauth`: usuarios en China (`api.minimaxi.com`)

Consulta el README del paquete del plugin MiniMax en el repositorio de OpenClaw para más detalles.

### MiniMax M2.7 (API key)

**Mejor para:** MiniMax alojado con API compatible con Anthropic.

Configúralo mediante la CLI:

- Onboarding interactivo:

```bash
openclaw onboard --auth-choice minimax-global-api
# o
openclaw onboard --auth-choice minimax-cn-api
```

- `minimax-global-api`: usuarios internacionales (`api.minimax.io`)
- `minimax-cn-api`: usuarios en China (`api.minimaxi.com`)

```json5
{
  env: { MINIMAX_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "minimax/MiniMax-M2.7" } } },
  models: {
    mode: "merge",
    providers: {
      minimax: {
        baseUrl: "https://api.minimax.io/anthropic",
        apiKey: "${MINIMAX_API_KEY}",
        api: "anthropic-messages",
        models: [
          {
            id: "MiniMax-M2.7",
            name: "MiniMax M2.7",
            reasoning: true,
            input: ["text", "image"],
            cost: { input: 0.3, output: 1.2, cacheRead: 0.06, cacheWrite: 0.375 },
            contextWindow: 204800,
            maxTokens: 131072,
          },
          {
            id: "MiniMax-M2.7-highspeed",
            name: "MiniMax M2.7 Highspeed",
            reasoning: true,
            input: ["text", "image"],
            cost: { input: 0.6, output: 2.4, cacheRead: 0.06, cacheWrite: 0.375 },
            contextWindow: 204800,
            maxTokens: 131072,
          },
        ],
      },
    },
  },
}
```

En la ruta de streaming compatible con Anthropic, OpenClaw ahora desactiva el
thinking de MiniMax de forma predeterminada a menos que establezcas `thinking` explícitamente. El endpoint de
streaming de MiniMax emite `reasoning_content` en fragmentos delta de estilo OpenAI
en lugar de bloques nativos de thinking de Anthropic, lo que puede filtrar razonamiento interno
en la salida visible si se deja habilitado implícitamente.

### MiniMax M2.7 como fallback (ejemplo)

**Mejor para:** mantener tu modelo más potente de última generación como principal y usar MiniMax M2.7 como failover.
El ejemplo siguiente usa Opus como principal concreto; cámbialo por el modelo principal de última generación que prefieras.

```json5
{
  env: { MINIMAX_API_KEY: "sk-..." },
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": { alias: "primary" },
        "minimax/MiniMax-M2.7": { alias: "minimax" },
      },
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["minimax/MiniMax-M2.7"],
      },
    },
  },
}
```

## Configurar mediante `openclaw configure`

Usa el asistente interactivo de configuración para establecer MiniMax sin editar JSON:

1. Ejecuta `openclaw configure`.
2. Selecciona **Model/auth**.
3. Elige una opción de autenticación **MiniMax**.
4. Elige tu modelo predeterminado cuando se te solicite.

Opciones actuales de autenticación MiniMax en el asistente/CLI:

- `minimax-global-oauth`
- `minimax-cn-oauth`
- `minimax-global-api`
- `minimax-cn-api`

## Opciones de configuración

- `models.providers.minimax.baseUrl`: prefiere `https://api.minimax.io/anthropic` (compatible con Anthropic); `https://api.minimax.io/v1` es opcional para cargas compatibles con OpenAI.
- `models.providers.minimax.api`: prefiere `anthropic-messages`; `openai-completions` es opcional para cargas compatibles con OpenAI.
- `models.providers.minimax.apiKey`: API key de MiniMax (`MINIMAX_API_KEY`).
- `models.providers.minimax.models`: define `id`, `name`, `reasoning`, `contextWindow`, `maxTokens`, `cost`.
- `agents.defaults.models`: crea alias de los modelos que quieras en la lista de permitidos.
- `models.mode`: mantén `merge` si quieres añadir MiniMax junto con los integrados.

## Notas

- Las referencias de modelo siguen la ruta de autenticación:
  - Configuración con API key: `minimax/<model>`
  - Configuración con OAuth: `minimax-portal/<model>`
- Modelo de chat predeterminado: `MiniMax-M2.7`
- Modelo de chat alternativo: `MiniMax-M2.7-highspeed`
- En `api: "anthropic-messages"`, OpenClaw inyecta
  `thinking: { type: "disabled" }` a menos que thinking ya esté establecido explícitamente en
  params/config.
- `/fast on` o `params.fastMode: true` reescribe `MiniMax-M2.7` a
  `MiniMax-M2.7-highspeed` en la ruta de stream compatible con Anthropic.
- El onboarding y la configuración directa con API key escriben definiciones explícitas de modelos con
  `input: ["text", "image"]` para ambas variantes M2.7
- El catálogo integrado del proveedor expone actualmente las referencias de chat como metadatos
  solo de texto hasta que exista una configuración explícita del proveedor MiniMax
- API de uso de Coding Plan: `https://api.minimaxi.com/v1/api/openplatform/coding_plan/remains` (requiere una clave de coding plan).
- OpenClaw normaliza el uso del coding-plan de MiniMax al mismo formato de visualización de `% restante`
  usado por otros proveedores. Los campos raw `usage_percent` / `usagePercent` de MiniMax son cuota restante, no cuota consumida, por lo que OpenClaw los invierte.
  Los campos basados en conteo tienen prioridad cuando están presentes. Cuando la API devuelve `model_remains`,
  OpenClaw prefiere la entrada del modelo de chat, deriva la etiqueta de ventana a partir de
  `start_time` / `end_time` cuando es necesario e incluye el nombre del modelo seleccionado
  en la etiqueta del plan para que las ventanas de coding-plan sean más fáciles de distinguir.
- Las instantáneas de uso tratan `minimax`, `minimax-cn` y `minimax-portal` como la
  misma superficie de cuota de MiniMax, y prefieren el OAuth almacenado de MiniMax antes de
  volver a variables de entorno con claves de Coding Plan.
- Actualiza los valores de precios en `models.json` si necesitas un seguimiento exacto de costes.
- Enlace de referencia para MiniMax Coding Plan (10% de descuento): [https://platform.minimax.io/subscribe/coding-plan?code=DbXJTRClnb&source=link](https://platform.minimax.io/subscribe/coding-plan?code=DbXJTRClnb&source=link)
- Consulta [/concepts/model-providers](/concepts/model-providers) para ver las reglas de proveedores.
- Usa `openclaw models list` para confirmar el ID de proveedor actual y luego cambia con
  `openclaw models set minimax/MiniMax-M2.7` o
  `openclaw models set minimax-portal/MiniMax-M2.7`.

## Solución de problemas

### "Unknown model: minimax/MiniMax-M2.7"

Esto normalmente significa que **el proveedor MiniMax no está configurado** (no hay una
entrada de proveedor coincidente y no se encontró ninguna clave de entorno/perfil de autenticación MiniMax). Hay una corrección para esta
detección en **2026.1.12**. Soluciónalo con una de estas opciones:

- Actualiza a **2026.1.12** (o ejecuta desde el código fuente `main`), luego reinicia la gateway.
- Ejecuta `openclaw configure` y selecciona una opción de autenticación **MiniMax**, o
- Añade manualmente el bloque correspondiente `models.providers.minimax` o
  `models.providers.minimax-portal`, o
- Establece `MINIMAX_API_KEY`, `MINIMAX_OAUTH_TOKEN` o un perfil de autenticación MiniMax
  para que se pueda inyectar el proveedor correspondiente.

Asegúrate de que el ID del modelo distingue **mayúsculas y minúsculas**:

- Ruta con API key: `minimax/MiniMax-M2.7` o `minimax/MiniMax-M2.7-highspeed`
- Ruta con OAuth: `minimax-portal/MiniMax-M2.7` o
  `minimax-portal/MiniMax-M2.7-highspeed`

Luego vuelve a comprobar con:

```bash
openclaw models list
```
