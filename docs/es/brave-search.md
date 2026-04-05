---
read_when:
    - Quieres usar Brave Search para `web_search`
    - Necesitas un `BRAVE_API_KEY` o detalles del plan
summary: Configuración de Brave Search API para `web_search`
title: Brave Search (ruta heredada)
x-i18n:
    generated_at: "2026-04-05T12:34:29Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7788e4cee7dc460819e55095c87df8cea29ba3a8bd3cef4c0e98ac601b45b651
    source_path: brave-search.md
    workflow: 15
---

# Brave Search API

OpenClaw admite Brave Search API como proveedor de `web_search`.

## Obtener una clave de API

1. Crea una cuenta de Brave Search API en [https://brave.com/search/api/](https://brave.com/search/api/)
2. En el panel, elige el plan **Search** y genera una clave de API.
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

La configuración de búsqueda de Brave específica del proveedor ahora se encuentra en `plugins.entries.brave.config.webSearch.*`.
La configuración heredada `tools.web.search.apiKey` sigue cargándose mediante la capa de compatibilidad, pero ya no es la ruta de configuración canónica.

`webSearch.mode` controla el transporte de Brave:

- `web` (predeterminado): búsqueda web normal de Brave con títulos, URL y fragmentos
- `llm-context`: Brave LLM Context API con fragmentos de texto y fuentes extraídos previamente para fundamentación

## Parámetros de la herramienta

| Parámetro     | Descripción                                                         |
| ------------- | ------------------------------------------------------------------- |
| `query`       | Consulta de búsqueda (obligatoria)                                  |
| `count`       | Número de resultados que se devolverán (1-10, predeterminado: 5)    |
| `country`     | Código de país ISO de 2 letras (p. ej., "US", "DE")                 |
| `language`    | Código de idioma ISO 639-1 para los resultados de búsqueda (p. ej., "en", "de", "fr") |
| `search_lang` | Código de idioma de búsqueda de Brave (p. ej., `en`, `en-gb`, `zh-hans`) |
| `ui_lang`     | Código de idioma ISO para elementos de la interfaz                  |
| `freshness`   | Filtro de tiempo: `day` (24 h), `week`, `month` o `year`            |
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

- OpenClaw usa el plan **Search** de Brave. Si tienes una suscripción heredada (por ejemplo, el plan Free original con 2.000 consultas al mes), sigue siendo válida, pero no incluye funciones más recientes como LLM Context ni límites de tasa más altos.
- Cada plan de Brave incluye **\$5 al mes en crédito gratis** (renovable). El plan Search cuesta \$5 por cada 1.000 solicitudes, por lo que el crédito cubre 1.000 consultas al mes. Configura tu límite de uso en el panel de Brave para evitar cargos inesperados. Consulta el [portal de Brave API](https://brave.com/search/api/) para ver los planes actuales.
- El plan Search incluye el endpoint de LLM Context y derechos de inferencia de IA. Almacenar resultados para entrenar o ajustar modelos requiere un plan con derechos explícitos de almacenamiento. Consulta las [Condiciones del servicio](https://api-dashboard.search.brave.com/terms-of-service) de Brave.
- El modo `llm-context` devuelve entradas de fuentes fundamentadas en lugar del formato normal de fragmentos de búsqueda web.
- El modo `llm-context` no admite `ui_lang`, `freshness`, `date_after` ni `date_before`.
- `ui_lang` debe incluir una subetiqueta de región como `en-US`.
- Los resultados se almacenan en caché durante 15 minutos de forma predeterminada (configurable mediante `cacheTtlMinutes`).

Consulta [Herramientas web](/tools/web) para ver la configuración completa de `web_search`.
