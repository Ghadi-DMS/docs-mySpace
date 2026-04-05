---
read_when:
    - Quieres usar Búsqueda de Perplexity para búsqueda web
    - Necesitas configurar `PERPLEXITY_API_KEY` o `OPENROUTER_API_KEY`
summary: API de búsqueda de Perplexity y compatibilidad de Sonar/OpenRouter para `web_search`
title: Búsqueda de Perplexity (ruta heredada)
x-i18n:
    generated_at: "2026-04-05T12:47:38Z"
    model: gpt-5.4
    provider: openai
    source_hash: ba91e63e7412f3b6f889ee11f4a66563014932a1dc7be8593fe2262a4877b89b
    source_path: perplexity.md
    workflow: 15
---

# API de búsqueda de Perplexity

OpenClaw admite la API de búsqueda de Perplexity como proveedor `web_search`.
Devuelve resultados estructurados con los campos `title`, `url` y `snippet`.

Por compatibilidad, OpenClaw también admite configuraciones heredadas de Perplexity Sonar/OpenRouter.
Si usas `OPENROUTER_API_KEY`, una clave `sk-or-...` en `plugins.entries.perplexity.config.webSearch.apiKey`, o estableces `plugins.entries.perplexity.config.webSearch.baseUrl` / `model`, el proveedor cambia a la ruta de chat-completions y devuelve respuestas sintetizadas por IA con citas en lugar de resultados estructurados de la API de búsqueda.

## Obtener una clave API de Perplexity

1. Crea una cuenta de Perplexity en [perplexity.ai/settings/api](https://www.perplexity.ai/settings/api)
2. Genera una clave API en el panel
3. Guarda la clave en la configuración o establece `PERPLEXITY_API_KEY` en el entorno del Gateway.

## Compatibilidad con OpenRouter

Si ya estabas usando OpenRouter para Perplexity Sonar, mantén `provider: "perplexity"` y establece `OPENROUTER_API_KEY` en el entorno del Gateway, o guarda una clave `sk-or-...` en `plugins.entries.perplexity.config.webSearch.apiKey`.

Controles de compatibilidad opcionales:

- `plugins.entries.perplexity.config.webSearch.baseUrl`
- `plugins.entries.perplexity.config.webSearch.model`

## Ejemplos de configuración

### API de búsqueda nativa de Perplexity

```json5
{
  plugins: {
    entries: {
      perplexity: {
        config: {
          webSearch: {
            apiKey: "pplx-...",
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "perplexity",
      },
    },
  },
}
```

### Compatibilidad con OpenRouter / Sonar

```json5
{
  plugins: {
    entries: {
      perplexity: {
        config: {
          webSearch: {
            apiKey: "<openrouter-api-key>",
            baseUrl: "https://openrouter.ai/api/v1",
            model: "perplexity/sonar-pro",
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "perplexity",
      },
    },
  },
}
```

## Dónde establecer la clave

**Mediante configuración:** ejecuta `openclaw configure --section web`. Guarda la clave en
`~/.openclaw/openclaw.json` bajo `plugins.entries.perplexity.config.webSearch.apiKey`.
Ese campo también acepta objetos SecretRef.

**Mediante entorno:** establece `PERPLEXITY_API_KEY` o `OPENROUTER_API_KEY`
en el entorno del proceso Gateway. Para una instalación de gateway, colócalo en
`~/.openclaw/.env` (o en el entorno de tu servicio). Consulta [Variables de entorno](/help/faq#env-vars-and-env-loading).

Si `provider: "perplexity"` está configurado y el SecretRef de la clave de Perplexity no se resuelve sin respaldo de entorno, el inicio/recarga falla rápidamente.

## Parámetros de la herramienta

Estos parámetros se aplican a la ruta nativa de la API de búsqueda de Perplexity.

| Parámetro             | Descripción                                          |
| --------------------- | ---------------------------------------------------- |
| `query`               | Consulta de búsqueda (obligatoria)                   |
| `count`               | Número de resultados que se devolverán (1-10, predeterminado: 5) |
| `country`             | Código de país ISO de 2 letras (por ejemplo, `"US"`, `"DE"`) |
| `language`            | Código de idioma ISO 639-1 (por ejemplo, `"en"`, `"de"`, `"fr"`) |
| `freshness`           | Filtro temporal: `day` (24 h), `week`, `month` o `year` |
| `date_after`          | Solo resultados publicados después de esta fecha (YYYY-MM-DD) |
| `date_before`         | Solo resultados publicados antes de esta fecha (YYYY-MM-DD) |
| `domain_filter`       | Matriz de lista de permitidos/denegados de dominios (máx. 20) |
| `max_tokens`          | Presupuesto total de contenido (predeterminado: 25000, máx.: 1000000) |
| `max_tokens_per_page` | Límite de tokens por página (predeterminado: 2048)   |

Para la ruta heredada de compatibilidad Sonar/OpenRouter:

- se aceptan `query`, `count` y `freshness`
- `count` es solo de compatibilidad ahí; la respuesta sigue siendo una sola
  respuesta sintetizada con citas en lugar de una lista de N resultados
- los filtros exclusivos de la API de búsqueda como `country`, `language`, `date_after`,
  `date_before`, `domain_filter`, `max_tokens` y `max_tokens_per_page`
  devuelven errores explícitos

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

// Domain filtering (allowlist)
await web_search({
  query: "climate research",
  domain_filter: ["nature.com", "science.org", ".edu"],
});

// Domain filtering (denylist - prefix with -)
await web_search({
  query: "product reviews",
  domain_filter: ["-reddit.com", "-pinterest.com"],
});

// More content extraction
await web_search({
  query: "detailed AI research",
  max_tokens: 50000,
  max_tokens_per_page: 4096,
});
```

### Reglas de filtro de dominios

- Máximo 20 dominios por filtro
- No se puede mezclar lista de permitidos y lista de denegados en la misma solicitud
- Usa el prefijo `-` para entradas de lista de denegados (por ejemplo, `["-reddit.com"]`)

## Notas

- La API de búsqueda de Perplexity devuelve resultados estructurados de búsqueda web (`title`, `url`, `snippet`)
- OpenRouter o `plugins.entries.perplexity.config.webSearch.baseUrl` / `model` explícitos hacen que Perplexity vuelva a Sonar chat completions por compatibilidad
- La compatibilidad con Sonar/OpenRouter devuelve una única respuesta sintetizada con citas, no filas estructuradas de resultados
- Los resultados se almacenan en caché durante 15 minutos de forma predeterminada (configurable mediante `cacheTtlMinutes`)

Consulta [Herramientas web](/tools/web) para ver la configuración completa de `web_search`.
Consulta [documentación de la API de búsqueda de Perplexity](https://docs.perplexity.ai/docs/search/quickstart) para más detalles.
