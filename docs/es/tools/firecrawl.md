---
read_when:
    - Quieres extracción web con Firecrawl
    - Necesitas una API key de Firecrawl
    - Quieres Firecrawl como proveedor de `web_search`
    - Quieres extracción con evasión antibots para `web_fetch`
summary: Búsqueda, extracción y fallback de `web_fetch` con Firecrawl
title: Firecrawl
x-i18n:
    generated_at: "2026-04-05T12:55:33Z"
    model: gpt-5.4
    provider: openai
    source_hash: 45f17fc4b8e81e1bfe25f510b0a64ab0d50c4cc95bcf88d6ba7c62cece26162e
    source_path: tools/firecrawl.md
    workflow: 15
---

# Firecrawl

OpenClaw puede usar **Firecrawl** de tres maneras:

- como proveedor de `web_search`
- como herramientas explícitas del plugin: `firecrawl_search` y `firecrawl_scrape`
- como extractor de respaldo para `web_fetch`

Es un servicio alojado de extracción/búsqueda que admite evasión de bots y almacenamiento en caché,
lo que ayuda con sitios con mucho JS o páginas que bloquean las solicitudes HTTP simples.

## Obtener una API key

1. Crea una cuenta de Firecrawl y genera una API key.
2. Guárdala en la configuración o establece `FIRECRAWL_API_KEY` en el entorno del gateway.

## Configurar la búsqueda de Firecrawl

```json5
{
  tools: {
    web: {
      search: {
        provider: "firecrawl",
      },
    },
  },
  plugins: {
    entries: {
      firecrawl: {
        enabled: true,
        config: {
          webSearch: {
            apiKey: "FIRECRAWL_API_KEY_HERE",
            baseUrl: "https://api.firecrawl.dev",
          },
        },
      },
    },
  },
}
```

Notas:

- Elegir Firecrawl en onboarding o `openclaw configure --section web` habilita automáticamente el plugin Firecrawl incluido.
- `web_search` con Firecrawl admite `query` y `count`.
- Para controles específicos de Firecrawl como `sources`, `categories` o extracción de resultados, usa `firecrawl_search`.
- Las anulaciones de `baseUrl` deben permanecer en `https://api.firecrawl.dev`.
- `FIRECRAWL_BASE_URL` es el fallback compartido de env para las URL base de búsqueda y extracción de Firecrawl.

## Configurar la extracción de Firecrawl + fallback de `web_fetch`

```json5
{
  plugins: {
    entries: {
      firecrawl: {
        enabled: true,
        config: {
          webFetch: {
            apiKey: "FIRECRAWL_API_KEY_HERE",
            baseUrl: "https://api.firecrawl.dev",
            onlyMainContent: true,
            maxAgeMs: 172800000,
            timeoutSeconds: 60,
          },
        },
      },
    },
  },
}
```

Notas:

- Los intentos de fallback de Firecrawl solo se ejecutan cuando hay una API key disponible (`plugins.entries.firecrawl.config.webFetch.apiKey` o `FIRECRAWL_API_KEY`).
- `maxAgeMs` controla la antigüedad máxima permitida de los resultados en caché (ms). El valor predeterminado es 2 días.
- La configuración heredada `tools.web.fetch.firecrawl.*` se migra automáticamente mediante `openclaw doctor --fix`.
- Las anulaciones de URL base y extracción de Firecrawl están restringidas a `https://api.firecrawl.dev`.

`firecrawl_scrape` reutiliza la misma configuración `plugins.entries.firecrawl.config.webFetch.*` y las mismas variables env.

## Herramientas del plugin Firecrawl

### `firecrawl_search`

Usa esto cuando quieras controles de búsqueda específicos de Firecrawl en lugar de `web_search` genérico.

Parámetros principales:

- `query`
- `count`
- `sources`
- `categories`
- `scrapeResults`
- `timeoutSeconds`

### `firecrawl_scrape`

Usa esto para páginas con mucho JS o protegidas contra bots donde `web_fetch` simple es débil.

Parámetros principales:

- `url`
- `extractMode`
- `maxChars`
- `onlyMainContent`
- `maxAgeMs`
- `proxy`
- `storeInCache`
- `timeoutSeconds`

## Stealth / evasión de bots

Firecrawl expone un parámetro de **modo proxy** para evasión de bots (`basic`, `stealth` o `auto`).
OpenClaw siempre usa `proxy: "auto"` junto con `storeInCache: true` para las solicitudes de Firecrawl.
Si se omite `proxy`, Firecrawl usa `auto` de forma predeterminada. `auto` reintenta con proxies stealth si falla un intento básico, lo que puede consumir más créditos
que una extracción solo con modo básico.

## Cómo usa Firecrawl `web_fetch`

Orden de extracción de `web_fetch`:

1. Readability (local)
2. Firecrawl (si está seleccionado o se detecta automáticamente como el fallback activo de web-fetch)
3. Limpieza básica de HTML (último fallback)

El control de selección es `tools.web.fetch.provider`. Si lo omites, OpenClaw
detecta automáticamente el primer proveedor de web-fetch listo a partir de las credenciales disponibles.
Hoy, el proveedor incluido es Firecrawl.

## Relacionado

- [Resumen de Web Search](/tools/web) -- todos los proveedores y detección automática
- [Web Fetch](/tools/web-fetch) -- herramienta `web_fetch` con fallback de Firecrawl
- [Tavily](/tools/tavily) -- herramientas de búsqueda + extracción
