---
read_when:
    - Quieres usar Perplexity Search para búsqueda web
    - Necesitas configurar `PERPLEXITY_API_KEY` o `OPENROUTER_API_KEY`
summary: API de Perplexity Search y compatibilidad con Sonar/OpenRouter para `web_search`
title: Búsqueda Perplexity
x-i18n:
    generated_at: "2026-04-05T12:56:20Z"
    model: gpt-5.4
    provider: openai
    source_hash: 06d97498e26e5570364e1486cb75584ed53b40a0091bf0210e1ea62f62d562ea
    source_path: tools/perplexity-search.md
    workflow: 15
---

# API de Perplexity Search

OpenClaw admite la API de Perplexity Search como proveedor de `web_search`.
Devuelve resultados estructurados con los campos `title`, `url` y `snippet`.

Por compatibilidad, OpenClaw también admite configuraciones heredadas de Perplexity Sonar/OpenRouter.
Si usas `OPENROUTER_API_KEY`, una clave `sk-or-...` en `plugins.entries.perplexity.config.webSearch.apiKey`, o configuras `plugins.entries.perplexity.config.webSearch.baseUrl` / `model`, el proveedor cambia a la ruta de chat completions y devuelve respuestas sintetizadas por IA con citas en lugar de resultados estructurados de la API de Search.

## Obtener una clave de API de Perplexity

1. Crea una cuenta de Perplexity en [perplexity.ai/settings/api](https://www.perplexity.ai/settings/api)
2. Genera una clave de API en el panel
3. Guarda la clave en la configuración o establece `PERPLEXITY_API_KEY` en el entorno del Gateway.

## Compatibilidad con OpenRouter

Si ya estabas usando OpenRouter para Perplexity Sonar, mantén `provider: "perplexity"` y establece `OPENROUTER_API_KEY` en el entorno del Gateway, o guarda una clave `sk-or-...` en `plugins.entries.perplexity.config.webSearch.apiKey`.

Controles opcionales de compatibilidad:

- `plugins.entries.perplexity.config.webSearch.baseUrl`
- `plugins.entries.perplexity.config.webSearch.model`

## Ejemplos de configuración

### API nativa de Perplexity Search

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

## Dónde configurar la clave

**Mediante configuración:** ejecuta `openclaw configure --section web`. Guarda la clave en
`~/.openclaw/openclaw.json` en `plugins.entries.perplexity.config.webSearch.apiKey`.
Ese campo también acepta objetos SecretRef.

**Mediante entorno:** establece `PERPLEXITY_API_KEY` o `OPENROUTER_API_KEY`
en el entorno del proceso Gateway. Para una instalación del gateway, colócalo en
`~/.openclaw/.env` (o en el entorno de tu servicio). Consulta [Variables de entorno](/es/help/faq#env-vars-and-env-loading).

Si `provider: "perplexity"` está configurado y la SecretRef de la clave de Perplexity no se resuelve sin alternativa en el entorno, el inicio/la recarga falla de inmediato.

## Parámetros de la herramienta

Estos parámetros se aplican a la ruta nativa de la API de Perplexity Search.

| Parámetro             | Descripción                                              |
| --------------------- | -------------------------------------------------------- |
| `query`               | Consulta de búsqueda (obligatoria)                       |
| `count`               | Número de resultados que se devolverán (1-10, predeterminado: 5) |
| `country`             | Código ISO de país de 2 letras (por ejemplo, `"US"`, `"DE"`) |
| `language`            | Código de idioma ISO 639-1 (por ejemplo, `"en"`, `"de"`, `"fr"`) |
| `freshness`           | Filtro temporal: `day` (24 h), `week`, `month` o `year` |
| `date_after`          | Solo resultados publicados después de esta fecha (YYYY-MM-DD) |
| `date_before`         | Solo resultados publicados antes de esta fecha (YYYY-MM-DD) |
| `domain_filter`       | Array de allowlist/denylist de dominios (máx. 20)        |
| `max_tokens`          | Presupuesto total de contenido (predeterminado: 25000, máx.: 1000000) |
| `max_tokens_per_page` | Límite de tokens por página (predeterminado: 2048)      |

Para la ruta heredada de compatibilidad con Sonar/OpenRouter:

- Se aceptan `query`, `count` y `freshness`
- `count` es solo de compatibilidad allí; la respuesta sigue siendo una única
  respuesta sintetizada con citas en lugar de una lista de N resultados
- Los filtros exclusivos de la API de Search como `country`, `language`, `date_after`,
  `date_before`, `domain_filter`, `max_tokens` y `max_tokens_per_page`
  devuelven errores explícitos

**Ejemplos:**

```javascript
// Búsqueda específica por país e idioma
await web_search({
  query: "renewable energy",
  country: "DE",
  language: "de",
});

// Resultados recientes (última semana)
await web_search({
  query: "AI news",
  freshness: "week",
});

// Búsqueda por intervalo de fechas
await web_search({
  query: "AI developments",
  date_after: "2024-01-01",
  date_before: "2024-06-30",
});

// Filtrado de dominios (allowlist)
await web_search({
  query: "climate research",
  domain_filter: ["nature.com", "science.org", ".edu"],
});

// Filtrado de dominios (denylist - prefijo con -)
await web_search({
  query: "product reviews",
  domain_filter: ["-reddit.com", "-pinterest.com"],
});

// Más extracción de contenido
await web_search({
  query: "detailed AI research",
  max_tokens: 50000,
  max_tokens_per_page: 4096,
});
```

### Reglas del filtro de dominios

- Máximo de 20 dominios por filtro
- No se puede mezclar allowlist y denylist en la misma solicitud
- Usa el prefijo `-` para entradas de denylist (por ejemplo, `["-reddit.com"]`)

## Notas

- La API de Perplexity Search devuelve resultados estructurados de búsqueda web (`title`, `url`, `snippet`)
- OpenRouter o `plugins.entries.perplexity.config.webSearch.baseUrl` / `model` explícitos hacen que Perplexity vuelva a usar Sonar chat completions por compatibilidad
- La compatibilidad con Sonar/OpenRouter devuelve una respuesta sintetizada con citas, no filas de resultados estructurados
- Los resultados se almacenan en caché durante 15 minutos de forma predeterminada (configurable mediante `cacheTtlMinutes`)

## Relacionado

- [Descripción general de Web Search](/tools/web) -- todos los proveedores y detección automática
- [Documentación de la API de Perplexity Search](https://docs.perplexity.ai/docs/search/quickstart) -- documentación oficial de Perplexity
- [Brave Search](/tools/brave-search) -- resultados estructurados con filtros de país/idioma
- [Búsqueda Exa](/tools/exa-search) -- búsqueda neural con extracción de contenido
