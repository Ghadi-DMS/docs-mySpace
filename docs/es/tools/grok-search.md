---
read_when:
    - Quieres usar Grok para `web_search`
    - Necesitas una `XAI_API_KEY` para búsqueda web
summary: Búsqueda web de Grok mediante respuestas fundamentadas en la web de xAI
title: Búsqueda Grok
x-i18n:
    generated_at: "2026-04-05T12:55:39Z"
    model: gpt-5.4
    provider: openai
    source_hash: ae2343012eebbe75d3ecdde3cb4470415c3275b694d0339bc26c46675a652054
    source_path: tools/grok-search.md
    workflow: 15
---

# Búsqueda Grok

OpenClaw admite Grok como proveedor de `web_search`, usando respuestas fundamentadas
en la web de xAI para producir respuestas sintetizadas por IA respaldadas por resultados
de búsqueda en vivo con citas.

La misma `XAI_API_KEY` también puede alimentar la herramienta integrada `x_search` para la
búsqueda de publicaciones en X (antes Twitter). Si guardas la clave en
`plugins.entries.xai.config.webSearch.apiKey`, OpenClaw ahora la reutiliza como
respaldo también para el proveedor de modelos xAI incluido.

Para métricas de publicaciones de X, como reposts, respuestas, marcadores o visualizaciones, es preferible
usar `x_search` con la URL exacta de la publicación o el ID del estado en lugar de una consulta de búsqueda amplia.

## Incorporación y configuración

Si eliges **Grok** durante:

- `openclaw onboard`
- `openclaw configure --section web`

OpenClaw puede mostrar un paso de seguimiento independiente para habilitar `x_search` con la misma
`XAI_API_KEY`. Ese seguimiento:

- solo aparece después de elegir Grok para `web_search`
- no es una opción independiente de nivel superior de proveedor de búsqueda web
- opcionalmente puede establecer el modelo de `x_search` durante el mismo flujo

Si lo omites, puedes habilitar o cambiar `x_search` más tarde en la configuración.

## Obtener una clave de API

<Steps>
  <Step title="Crea una clave">
    Obtén una clave de API de [xAI](https://console.x.ai/).
  </Step>
  <Step title="Guarda la clave">
    Establece `XAI_API_KEY` en el entorno del Gateway, o configúralo mediante:

    ```bash
    openclaw configure --section web
    ```

  </Step>
</Steps>

## Configuración

```json5
{
  plugins: {
    entries: {
      xai: {
        config: {
          webSearch: {
            apiKey: "xai-...", // opcional si XAI_API_KEY está establecido
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "grok",
      },
    },
  },
}
```

**Alternativa de entorno:** establece `XAI_API_KEY` en el entorno del Gateway.
Para una instalación del gateway, colócalo en `~/.openclaw/.env`.

## Cómo funciona

Grok usa respuestas fundamentadas en la web de xAI para sintetizar respuestas con
citas integradas, de forma similar al enfoque de fundamentación con Google Search de Gemini.

## Parámetros compatibles

La búsqueda de Grok admite `query`.

Se acepta `count` para compatibilidad compartida con `web_search`, pero Grok sigue
devolviendo una respuesta sintetizada con citas en lugar de una lista de N resultados.

Actualmente no se admiten filtros específicos del proveedor.

## Relacionado

- [Descripción general de Web Search](/tools/web) -- todos los proveedores y detección automática
- [x_search en Web Search](/tools/web#x_search) -- búsqueda de X de primera clase mediante xAI
- [Búsqueda Gemini](/tools/gemini-search) -- respuestas sintetizadas por IA mediante fundamentación de Google
