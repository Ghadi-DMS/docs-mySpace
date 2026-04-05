---
read_when:
    - Quieres la configuración de Moonshot K2 (Moonshot Open Platform) frente a Kimi Coding
    - Necesitas entender los endpoints, claves y referencias de modelo separados
    - Quieres una configuración de copiar y pegar para cualquiera de los dos proveedores
summary: Configura Moonshot K2 frente a Kimi Coding (proveedores y claves separados)
title: Moonshot AI
x-i18n:
    generated_at: "2026-04-05T12:51:42Z"
    model: gpt-5.4
    provider: openai
    source_hash: a80c71ef432b778e296bd60b7d9ec7c72d025d13fd9bdae474b3d58436d15695
    source_path: providers/moonshot.md
    workflow: 15
---

# Moonshot AI (Kimi)

Moonshot proporciona la API de Kimi con endpoints compatibles con OpenAI. Configura el
proveedor y establece el modelo predeterminado en `moonshot/kimi-k2.5`, o usa
Kimi Coding con `kimi/kimi-code`.

ID actuales de modelos Kimi K2:

[//]: # "moonshot-kimi-k2-ids:start"

- `kimi-k2.5`
- `kimi-k2-thinking`
- `kimi-k2-thinking-turbo`
- `kimi-k2-turbo`

[//]: # "moonshot-kimi-k2-ids:end"

```bash
openclaw onboard --auth-choice moonshot-api-key
# or
openclaw onboard --auth-choice moonshot-api-key-cn
```

Kimi Coding:

```bash
openclaw onboard --auth-choice kimi-code-api-key
```

Nota: Moonshot y Kimi Coding son proveedores separados. Las claves no son intercambiables, los endpoints son distintos y las referencias de modelo también lo son (Moonshot usa `moonshot/...`, Kimi Coding usa `kimi/...`).

La búsqueda web de Kimi también usa el plugin Moonshot:

```bash
openclaw configure --section web
```

Elige **Kimi** en la sección de búsqueda web para almacenar
`plugins.entries.moonshot.config.webSearch.*`.

## Fragmento de configuración (API de Moonshot)

```json5
{
  env: { MOONSHOT_API_KEY: "sk-..." },
  agents: {
    defaults: {
      model: { primary: "moonshot/kimi-k2.5" },
      models: {
        // moonshot-kimi-k2-aliases:start
        "moonshot/kimi-k2.5": { alias: "Kimi K2.5" },
        "moonshot/kimi-k2-thinking": { alias: "Kimi K2 Thinking" },
        "moonshot/kimi-k2-thinking-turbo": { alias: "Kimi K2 Thinking Turbo" },
        "moonshot/kimi-k2-turbo": { alias: "Kimi K2 Turbo" },
        // moonshot-kimi-k2-aliases:end
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      moonshot: {
        baseUrl: "https://api.moonshot.ai/v1",
        apiKey: "${MOONSHOT_API_KEY}",
        api: "openai-completions",
        models: [
          // moonshot-kimi-k2-models:start
          {
            id: "kimi-k2.5",
            name: "Kimi K2.5",
            reasoning: false,
            input: ["text", "image"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 262144,
            maxTokens: 262144,
          },
          {
            id: "kimi-k2-thinking",
            name: "Kimi K2 Thinking",
            reasoning: true,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 262144,
            maxTokens: 262144,
          },
          {
            id: "kimi-k2-thinking-turbo",
            name: "Kimi K2 Thinking Turbo",
            reasoning: true,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 262144,
            maxTokens: 262144,
          },
          {
            id: "kimi-k2-turbo",
            name: "Kimi K2 Turbo",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 256000,
            maxTokens: 16384,
          },
          // moonshot-kimi-k2-models:end
        ],
      },
    },
  },
}
```

## Kimi Coding

```json5
{
  env: { KIMI_API_KEY: "sk-..." },
  agents: {
    defaults: {
      model: { primary: "kimi/kimi-code" },
      models: {
        "kimi/kimi-code": { alias: "Kimi" },
      },
    },
  },
}
```

## Búsqueda web de Kimi

OpenClaw también incluye **Kimi** como proveedor de `web_search`, respaldado por la
búsqueda web de Moonshot.

La configuración interactiva puede solicitar:

- la región de la API de Moonshot:
  - `https://api.moonshot.ai/v1`
  - `https://api.moonshot.cn/v1`
- el modelo predeterminado de búsqueda web de Kimi (usa `kimi-k2.5` por defecto)

La configuración se guarda en `plugins.entries.moonshot.config.webSearch`:

```json5
{
  plugins: {
    entries: {
      moonshot: {
        config: {
          webSearch: {
            apiKey: "sk-...", // or use KIMI_API_KEY / MOONSHOT_API_KEY
            baseUrl: "https://api.moonshot.ai/v1",
            model: "kimi-k2.5",
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "kimi",
      },
    },
  },
}
```

## Notas

- Las referencias de modelo de Moonshot usan `moonshot/<modelId>`. Las referencias de modelo de Kimi Coding usan `kimi/<modelId>`.
- La referencia actual del modelo predeterminado de Kimi Coding es `kimi/kimi-code`. El heredado `kimi/k2p5` sigue aceptándose como ID de modelo por compatibilidad.
- La búsqueda web de Kimi usa `KIMI_API_KEY` o `MOONSHOT_API_KEY`, y usa por defecto `https://api.moonshot.ai/v1` con el modelo `kimi-k2.5`.
- Los endpoints nativos de Moonshot (`https://api.moonshot.ai/v1` y
  `https://api.moonshot.cn/v1`) anuncian compatibilidad de uso de streaming en el
  transporte compartido `openai-completions`. OpenClaw ahora se basa en las capacidades
  del endpoint, por lo que los ID de proveedor personalizados compatibles que apuntan a los mismos hosts
  nativos de Moonshot heredan el mismo comportamiento de uso de streaming.
- Sobrescribe el precio y los metadatos de contexto en `models.providers` si es necesario.
- Si Moonshot publica límites de contexto distintos para un modelo, ajusta
  `contextWindow` en consecuencia.
- Usa `https://api.moonshot.ai/v1` para el endpoint internacional y `https://api.moonshot.cn/v1` para el endpoint de China.
- Opciones de onboarding:
  - `moonshot-api-key` para `https://api.moonshot.ai/v1`
  - `moonshot-api-key-cn` para `https://api.moonshot.cn/v1`

## Modo thinking nativo (Moonshot)

Moonshot Kimi admite thinking nativo binario:

- `thinking: { type: "enabled" }`
- `thinking: { type: "disabled" }`

Configúralo por modelo mediante `agents.defaults.models.<provider/model>.params`:

```json5
{
  agents: {
    defaults: {
      models: {
        "moonshot/kimi-k2.5": {
          params: {
            thinking: { type: "disabled" },
          },
        },
      },
    },
  },
}
```

OpenClaw también asigna niveles de ejecución `/think` para Moonshot:

- `/think off` -> `thinking.type=disabled`
- cualquier nivel de thinking distinto de off -> `thinking.type=enabled`

Cuando el thinking de Moonshot está habilitado, `tool_choice` debe ser `auto` o `none`. OpenClaw normaliza los valores incompatibles de `tool_choice` a `auto` por compatibilidad.
