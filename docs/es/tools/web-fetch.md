---
read_when:
    - Quieres obtener una URL y extraer contenido legible
    - Necesitas configurar web_fetch o su alternativa de Firecrawl
    - Quieres entender los límites y el almacenamiento en caché de web_fetch
sidebarTitle: Web Fetch
summary: Herramienta web_fetch -- obtención HTTP con extracción de contenido legible
title: Web Fetch
x-i18n:
    generated_at: "2026-04-05T12:56:56Z"
    model: gpt-5.4
    provider: openai
    source_hash: 60c933a25d0f4511dc1683985988e115b836244c5eac4c6667b67c8eb15401e0
    source_path: tools/web-fetch.md
    workflow: 15
---

# Web Fetch

La herramienta `web_fetch` realiza un HTTP GET simple y extrae contenido legible
(HTML a markdown o texto). **No** ejecuta JavaScript.

Para sitios con mucho JS o páginas protegidas por inicio de sesión, usa el
[Web Browser](/tools/browser) en su lugar.

## Inicio rápido

`web_fetch` está **habilitado de forma predeterminada**; no se necesita configuración. El agente puede
llamarlo de inmediato:

```javascript
await web_fetch({ url: "https://example.com/article" });
```

## Parámetros de la herramienta

| Parameter     | Type     | Description                              |
| ------------- | -------- | ---------------------------------------- |
| `url`         | `string` | URL que se va a obtener (obligatoria, solo http/https) |
| `extractMode` | `string` | `"markdown"` (predeterminado) o `"text"` |
| `maxChars`    | `number` | Trunca la salida a esta cantidad de caracteres |

## Cómo funciona

<Steps>
  <Step title="Fetch">
    Envía un HTTP GET con un User-Agent similar a Chrome y el encabezado
    `Accept-Language`. Bloquea nombres de host privados/internos y vuelve a comprobar los redireccionamientos.
  </Step>
  <Step title="Extract">
    Ejecuta Readability (extracción del contenido principal) sobre la respuesta HTML.
  </Step>
  <Step title="Fallback (optional)">
    Si Readability falla y Firecrawl está configurado, vuelve a intentarlo mediante la
    API de Firecrawl con modo de evasión de bots.
  </Step>
  <Step title="Cache">
    Los resultados se almacenan en caché durante 15 minutos (configurable) para reducir
    las obtenciones repetidas de la misma URL.
  </Step>
</Steps>

## Configuración

```json5
{
  tools: {
    web: {
      fetch: {
        enabled: true, // predeterminado: true
        provider: "firecrawl", // opcional; omitir para detección automática
        maxChars: 50000, // máximo de caracteres de salida
        maxCharsCap: 50000, // límite máximo estricto para el parámetro maxChars
        maxResponseBytes: 2000000, // tamaño máximo de descarga antes del truncamiento
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
        maxRedirects: 3,
        readability: true, // usar extracción de Readability
        userAgent: "Mozilla/5.0 ...", // sobrescribir User-Agent
      },
    },
  },
}
```

## Alternativa de Firecrawl

Si falla la extracción de Readability, `web_fetch` puede recurrir a
[Firecrawl](/tools/firecrawl) para evasión de bots y una mejor extracción:

```json5
{
  tools: {
    web: {
      fetch: {
        provider: "firecrawl", // opcional; omitir para detección automática a partir de las credenciales disponibles
      },
    },
  },
  plugins: {
    entries: {
      firecrawl: {
        enabled: true,
        config: {
          webFetch: {
            apiKey: "fc-...", // opcional si FIRECRAWL_API_KEY está configurada
            baseUrl: "https://api.firecrawl.dev",
            onlyMainContent: true,
            maxAgeMs: 86400000, // duración de la caché (1 día)
            timeoutSeconds: 60,
          },
        },
      },
    },
  },
}
```

`plugins.entries.firecrawl.config.webFetch.apiKey` admite objetos SecretRef.
La configuración heredada `tools.web.fetch.firecrawl.*` se migra automáticamente mediante `openclaw doctor --fix`.

<Note>
  Si Firecrawl está habilitado y su SecretRef no se resuelve sin
  alternativa de variable de entorno `FIRECRAWL_API_KEY`, el inicio del gateway falla rápidamente.
</Note>

<Note>
  Las sobrescrituras de `baseUrl` de Firecrawl están restringidas: deben usar `https://` y
  el host oficial de Firecrawl (`api.firecrawl.dev`).
</Note>

Comportamiento actual en runtime:

- `tools.web.fetch.provider` selecciona explícitamente el proveedor alternativo de obtención.
- Si se omite `provider`, OpenClaw detecta automáticamente el primer proveedor de web-fetch
  listo a partir de las credenciales disponibles. Hoy, el proveedor incluido es Firecrawl.
- Si Readability está deshabilitado, `web_fetch` omite directamente la
  alternativa del proveedor seleccionado. Si no hay ningún proveedor disponible, falla de forma segura.

## Límites y seguridad

- `maxChars` se ajusta al límite de `tools.web.fetch.maxCharsCap`
- El cuerpo de la respuesta se limita a `maxResponseBytes` antes del análisis; las
  respuestas demasiado grandes se truncan con una advertencia
- Los nombres de host privados/internos están bloqueados
- Los redireccionamientos se comprueban y limitan mediante `maxRedirects`
- `web_fetch` es de mejor esfuerzo; algunos sitios necesitan [Web Browser](/tools/browser)

## Perfiles de herramientas

Si usas perfiles de herramientas o listas de permitidos, añade `web_fetch` o `group:web`:

```json5
{
  tools: {
    allow: ["web_fetch"],
    // o: allow: ["group:web"]  (incluye web_fetch, web_search y x_search)
  },
}
```

## Relacionado

- [Web Search](/tools/web) -- busca en la web con varios proveedores
- [Web Browser](/tools/browser) -- automatización completa del navegador para sitios con mucho JS
- [Firecrawl](/tools/firecrawl) -- herramientas de búsqueda y scraping de Firecrawl
