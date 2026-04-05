---
read_when:
    - Quieres un proveedor de búsqueda web que no requiera API key
    - Quieres usar DuckDuckGo para `web_search`
    - Necesitas una alternativa de búsqueda sin configuración
summary: 'Búsqueda web con DuckDuckGo: proveedor alternativo sin clave (experimental, basado en HTML)'
title: Búsqueda con DuckDuckGo
x-i18n:
    generated_at: "2026-04-05T12:55:12Z"
    model: gpt-5.4
    provider: openai
    source_hash: 31f8e3883584534396c247c3d8069ea4c5b6399e0ff13a9dd0c8ee0c3da02096
    source_path: tools/duckduckgo-search.md
    workflow: 15
---

# Búsqueda con DuckDuckGo

OpenClaw admite DuckDuckGo como proveedor `web_search` **sin clave**. No se requiere
ninguna API key ni cuenta.

<Warning>
  DuckDuckGo es una integración **experimental, no oficial** que extrae resultados
  de las páginas de búsqueda sin JavaScript de DuckDuckGo, no de una API oficial. Es de esperar
  fallos ocasionales por páginas de verificación anti-bots o cambios en el HTML.
</Warning>

## Configuración

No se necesita API key; solo establece DuckDuckGo como tu proveedor:

<Steps>
  <Step title="Configurar">
    ```bash
    openclaw configure --section web
    # Selecciona "duckduckgo" como proveedor
    ```
  </Step>
</Steps>

## Configuración

```json5
{
  tools: {
    web: {
      search: {
        provider: "duckduckgo",
      },
    },
  },
}
```

Configuraciones opcionales a nivel de plugin para región y SafeSearch:

```json5
{
  plugins: {
    entries: {
      duckduckgo: {
        config: {
          webSearch: {
            region: "us-en", // Código de región de DuckDuckGo
            safeSearch: "moderate", // "strict", "moderate" o "off"
          },
        },
      },
    },
  },
}
```

## Parámetros de la herramienta

| Parámetro    | Descripción                                                      |
| ------------ | ---------------------------------------------------------------- |
| `query`      | Consulta de búsqueda (obligatoria)                               |
| `count`      | Resultados a devolver (1-10, predeterminado: 5)                  |
| `region`     | Código de región de DuckDuckGo (por ejemplo, `us-en`, `uk-en`, `de-de`) |
| `safeSearch` | Nivel de SafeSearch: `strict`, `moderate` (predeterminado) o `off` |

La región y SafeSearch también pueden configurarse en la configuración del plugin (ver arriba): los
parámetros de la herramienta sobrescriben los valores de configuración en cada consulta.

## Notas

- **Sin API key**: funciona de inmediato, sin configuración
- **Experimental**: recopila resultados de las páginas de búsqueda HTML sin JavaScript
  de DuckDuckGo, no de una API o SDK oficial
- **Riesgo de verificación anti-bots**: DuckDuckGo puede mostrar CAPTCHA o bloquear solicitudes
  con uso intensivo o automatizado
- **Análisis de HTML**: los resultados dependen de la estructura de la página, que puede cambiar sin
  previo aviso
- **Orden de detección automática**: DuckDuckGo es la primera alternativa sin clave
  (orden 100) en la detección automática. Los proveedores con API y claves configuradas se ejecutan
  primero, luego Ollama Web Search (orden 110) y después SearXNG (orden 200)
- **SafeSearch usa `moderate` de forma predeterminada** cuando no está configurado

<Tip>
  Para uso en producción, considera [Brave Search](/tools/brave-search) (con nivel gratuito
  disponible) u otro proveedor respaldado por API.
</Tip>

## Relacionado

- [Resumen de Web Search](/tools/web) -- todos los proveedores y detección automática
- [Brave Search](/tools/brave-search) -- resultados estructurados con nivel gratuito
- [Exa Search](/tools/exa-search) -- búsqueda neuronal con extracción de contenido
