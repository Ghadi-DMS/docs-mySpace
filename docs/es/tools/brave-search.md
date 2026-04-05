---
read_when:
    - Quieres usar Brave Search para `web_search`
    - Necesitas un `BRAVE_API_KEY` o detalles del plan
summary: Configuración de la API de Brave Search para `web_search`
title: Brave Search
x-i18n:
    generated_at: "2026-04-05T12:54:42Z"
    model: gpt-5.4
    provider: openai
    source_hash: bc026a69addf74375a0e407805b875ff527c77eb7298b2f5bb0e165197f77c0c
    source_path: tools/brave-search.md
    workflow: 15
---

# API de Brave Search

OpenClaw admite la API de Brave Search como proveedor de `web_search`.

## Obtener una API key

1. Crea una cuenta de la API de Brave Search en [https://brave.com/search/api/](https://brave.com/search/api/)
2. En el panel, elige el plan **Search** y genera una API key.
3. Guarda la clave en la configuración o establece `BRAVE_API_KEY` en el entorno del Gateway.

## Ejemplo de configuración

```json5
{
  plugins: {
    entries: {
      brave: {
        config: {
          webSearch: {
            apiKey: "BRAVE_API_KEY_HERE",
            mode: "web", // or "llm-context"
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "brave",
        maxResults: 5,
        timeoutSeconds: 30,
      },
    },
  },
}
```

Los ajustes de búsqueda de Brave específicos del proveedor ahora se encuentran en `plugins.entries.brave.config.webSearch.*`.
El antiguo `tools.web.search.apiKey` sigue cargándose mediante la capa de compatibilidad, pero ya no es la ruta de configuración canónica.

`webSearch.mode` controla el transporte de Brave:

- `web` (predeterminado): búsqueda web normal de Brave con títulos, URL y fragmentos
- `llm-context`: API LLM Context de Brave con fragmentos de texto extraídos previamente y fuentes para grounding

## Parámetros de la herramienta

| Parámetro     | Descripción                                                         |
| ------------- | ------------------------------------------------------------------- |
| `query`       | Consulta de búsqueda (obligatorio)                                  |
| `count`       | Número de resultados que se devolverán (1-10, predeterminado: 5)    |
| `country`     | Código de país ISO de 2 letras (p. ej., "US", "DE")                |
| `language`    | Código de idioma ISO 639-1 para los resultados de búsqueda (p. ej., "en", "de", "fr") |
| `search_lang` | Código de idioma de búsqueda de Brave (p. ej., `en`, `en-gb`, `zh-hans`) |
| `ui_lang`     | Código de idioma ISO para elementos de la UI                        |
| `freshness`   | Filtro temporal: `day` (24 h), `week`, `month` o `year`             |
| `date_after`  | Solo resultados publicados después de esta fecha (YYYY-MM-DD)       |
| `date_before` | Solo resultados publicados antes de esta fecha (YYYY-MM-DD)         |

**Ejemplos:**

```javascript
// Country and language-specific search
await web_search({
  query: "renewable energy",
  country: "DE",
  language: "de",
});

// Recent results (past week)
await web_search({
  query: "AI news",
  freshness: "week",
});

// Date range search
await web_search({
  query: "AI developments",
  date_after: "2024-01-01",
  date_before: "2024-06-30",
});
```

## Notas

- OpenClaw usa el plan **Search** de Brave. Si tienes una suscripción heredada (por ejemplo, el plan Free original con 2,000 consultas/mes), sigue siendo válida, pero no incluye funciones más nuevas como LLM Context ni límites de tasa más altos.
- Cada plan de Brave incluye **\$5/mes en crédito gratuito** (renovable). El plan Search cuesta \$5 por cada 1,000 solicitudes, por lo que el crédito cubre 1,000 consultas/mes. Establece tu límite de uso en el panel de Brave para evitar cargos inesperados. Consulta el [portal de la API de Brave](https://brave.com/search/api/) para ver los planes actuales.
- El plan Search incluye el endpoint LLM Context y derechos de inferencia de IA. Almacenar resultados para entrenar o ajustar modelos requiere un plan con derechos de almacenamiento explícitos. Consulta las [Condiciones del servicio](https://api-dashboard.search.brave.com/terms-of-service) de Brave.
- El modo `llm-context` devuelve entradas de fuentes con grounding en lugar del formato normal de fragmentos de búsqueda web.
- El modo `llm-context` no admite `ui_lang`, `freshness`, `date_after` ni `date_before`.
- `ui_lang` debe incluir una subetiqueta de región como `en-US`.
- Los resultados se almacenan en caché durante 15 minutos de forma predeterminada (configurable mediante `cacheTtlMinutes`).

## Relacionado

- [Resumen de Web Search](/tools/web) -- todos los proveedores y detección automática
- [Perplexity Search](/tools/perplexity-search) -- resultados estructurados con filtrado por dominio
- [Exa Search](/tools/exa-search) -- búsqueda neuronal con extracción de contenido
