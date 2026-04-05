---
read_when:
    - Quieres usar Kimi para `web_search`
    - Necesitas una `KIMI_API_KEY` o `MOONSHOT_API_KEY`
summary: Búsqueda web de Kimi mediante Moonshot web search
title: Búsqueda Kimi
x-i18n:
    generated_at: "2026-04-05T12:55:51Z"
    model: gpt-5.4
    provider: openai
    source_hash: 753757a5497a683c35b4509ed3709b9514dc14a45612675d0f729ae6668c82a5
    source_path: tools/kimi-search.md
    workflow: 15
---

# Búsqueda Kimi

OpenClaw admite Kimi como proveedor de `web_search`, usando la búsqueda web de Moonshot
para producir respuestas sintetizadas por IA con citas.

## Obtener una clave de API

<Steps>
  <Step title="Crea una clave">
    Obtén una clave de API de [Moonshot AI](https://platform.moonshot.cn/).
  </Step>
  <Step title="Guarda la clave">
    Establece `KIMI_API_KEY` o `MOONSHOT_API_KEY` en el entorno del Gateway, o
    configúralo mediante:

    ```bash
    openclaw configure --section web
    ```

  </Step>
</Steps>

Cuando eliges **Kimi** durante `openclaw onboard` o
`openclaw configure --section web`, OpenClaw también puede pedirte:

- la región de la API de Moonshot:
  - `https://api.moonshot.ai/v1`
  - `https://api.moonshot.cn/v1`
- el modelo predeterminado de búsqueda web de Kimi (el valor predeterminado es `kimi-k2.5`)

## Configuración

```json5
{
  plugins: {
    entries: {
      moonshot: {
        config: {
          webSearch: {
            apiKey: "sk-...", // opcional si KIMI_API_KEY o MOONSHOT_API_KEY está establecido
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

Si usas el host de la API de China para chat (`models.providers.moonshot.baseUrl`:
`https://api.moonshot.cn/v1`), OpenClaw reutiliza ese mismo host para Kimi
`web_search` cuando se omite `tools.web.search.kimi.baseUrl`, para que las claves de
[platform.moonshot.cn](https://platform.moonshot.cn/) no lleguen por error al
endpoint internacional (que a menudo devuelve HTTP 401). Sobrescribe esto
con `tools.web.search.kimi.baseUrl` cuando necesites una URL base de búsqueda distinta.

**Alternativa de entorno:** establece `KIMI_API_KEY` o `MOONSHOT_API_KEY` en el
entorno del Gateway. Para una instalación del gateway, colócalo en `~/.openclaw/.env`.

Si omites `baseUrl`, OpenClaw usa de forma predeterminada `https://api.moonshot.ai/v1`.
Si omites `model`, OpenClaw usa de forma predeterminada `kimi-k2.5`.

## Cómo funciona

Kimi usa la búsqueda web de Moonshot para sintetizar respuestas con citas integradas,
de forma similar al enfoque de respuestas fundamentadas de Gemini y Grok.

## Parámetros compatibles

La búsqueda de Kimi admite `query`.

Se acepta `count` para compatibilidad compartida con `web_search`, pero Kimi sigue
devolviendo una respuesta sintetizada con citas en lugar de una lista de N resultados.

Actualmente no se admiten filtros específicos del proveedor.

## Relacionado

- [Descripción general de Web Search](/tools/web) -- todos los proveedores y detección automática
- [Moonshot AI](/es/providers/moonshot) -- documentación del proveedor de modelos Moonshot + Kimi Coding
- [Búsqueda Gemini](/tools/gemini-search) -- respuestas sintetizadas por IA mediante fundamentación de Google
- [Búsqueda Grok](/tools/grok-search) -- respuestas sintetizadas por IA mediante fundamentación de xAI
