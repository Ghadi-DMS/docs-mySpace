---
read_when:
    - Quieres usar Gemini para `web_search`
    - Necesitas una `GEMINI_API_KEY`
    - Quieres grounding de Google Search
summary: Búsqueda web de Gemini con grounding de Google Search
title: Gemini Search
x-i18n:
    generated_at: "2026-04-05T12:55:29Z"
    model: gpt-5.4
    provider: openai
    source_hash: 42644176baca6b4b041142541618f6f68361d410d6f425cc4104cd88d9f7c480
    source_path: tools/gemini-search.md
    workflow: 15
---

# Gemini Search

OpenClaw es compatible con modelos Gemini con
[grounding integrado de Google Search](https://ai.google.dev/gemini-api/docs/grounding),
que devuelve respuestas sintetizadas por IA respaldadas por resultados en vivo
de Google Search con citas.

## Obtener una clave API

<Steps>
  <Step title="Crear una clave">
    Ve a [Google AI Studio](https://aistudio.google.com/apikey) y crea una
    clave API.
  </Step>
  <Step title="Guardar la clave">
    Establece `GEMINI_API_KEY` en el entorno del Gateway, o configúralo mediante:

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
      google: {
        config: {
          webSearch: {
            apiKey: "AIza...", // optional if GEMINI_API_KEY is set
            model: "gemini-2.5-flash", // default
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "gemini",
      },
    },
  },
}
```

**Alternativa con variable de entorno:** establece `GEMINI_API_KEY` en el entorno del Gateway.
Para una instalación del gateway, colócala en `~/.openclaw/.env`.

## Cómo funciona

A diferencia de los proveedores de búsqueda tradicionales que devuelven una
lista de enlaces y fragmentos, Gemini usa grounding de Google Search para
producir respuestas sintetizadas por IA con citas en línea. Los resultados
incluyen tanto la respuesta sintetizada como las URL de origen.

- Las URL de citas del grounding de Gemini se resuelven automáticamente desde
  las URL de redirección de Google a URL directas.
- La resolución de redirecciones usa la ruta de protección SSRF (comprobaciones
  HEAD + de redirección + validación de http/https) antes de devolver la URL de
  cita final.
- La resolución de redirecciones usa valores predeterminados estrictos de SSRF,
  por lo que las redirecciones a destinos privados/internos se bloquean.

## Parámetros compatibles

La búsqueda de Gemini es compatible con `query`.

Se acepta `count` para compatibilidad compartida con `web_search`, pero el grounding
de Gemini sigue devolviendo una única respuesta sintetizada con citas en lugar
de una lista de N resultados.

No se admiten filtros específicos del proveedor como `country`, `language`,
`freshness` y `domain_filter`.

## Selección de modelo

El modelo predeterminado es `gemini-2.5-flash` (rápido y rentable). Puede
usarse cualquier modelo Gemini compatible con grounding mediante
`plugins.entries.google.config.webSearch.model`.

## Relacionado

- [Descripción general de Web Search](/tools/web) -- todos los proveedores y detección automática
- [Brave Search](/tools/brave-search) -- resultados estructurados con fragmentos
- [Perplexity Search](/tools/perplexity-search) -- resultados estructurados + extracción de contenido
