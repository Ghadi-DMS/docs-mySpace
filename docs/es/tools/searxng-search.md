---
read_when:
    - Quieres un proveedor de búsqueda web autoalojado
    - Quieres usar SearXNG para `web_search`
    - Necesitas una opción de búsqueda centrada en la privacidad o aislada de la red
summary: Búsqueda web con SearXNG -- proveedor de metabúsqueda autoalojado y sin claves
title: Búsqueda con SearXNG
x-i18n:
    generated_at: "2026-04-05T12:56:32Z"
    model: gpt-5.4
    provider: openai
    source_hash: 0a8fc7f890b7595d17c5ef8aede9b84bb2459f30a53d5d87c4e7423e1ac83ca5
    source_path: tools/searxng-search.md
    workflow: 15
---

# Búsqueda con SearXNG

OpenClaw es compatible con [SearXNG](https://docs.searxng.org/) como proveedor
de `web_search` **autoalojado y sin claves**. SearXNG es un motor de
metabúsqueda de código abierto que agrega resultados de Google, Bing,
DuckDuckGo y otras fuentes.

Ventajas:

- **Gratis e ilimitado** -- no requiere clave API ni suscripción comercial
- **Privacidad / entorno aislado** -- las consultas nunca salen de tu red
- **Funciona en cualquier lugar** -- sin restricciones regionales de las API de búsqueda comerciales

## Configuración

<Steps>
  <Step title="Ejecuta una instancia de SearXNG">
    ```bash
    docker run -d -p 8888:8080 searxng/searxng
    ```

    O usa cualquier implementación existente de SearXNG a la que tengas acceso.
    Consulta la [documentación de SearXNG](https://docs.searxng.org/) para una
    configuración de producción.

  </Step>
  <Step title="Configura">
    ```bash
    openclaw configure --section web
    # Select "searxng" as the provider
    ```

    O establece la variable de entorno y deja que la detección automática la encuentre:

    ```bash
    export SEARXNG_BASE_URL="http://localhost:8888"
    ```

  </Step>
</Steps>

## Configuración

```json5
{
  tools: {
    web: {
      search: {
        provider: "searxng",
      },
    },
  },
}
```

Ajustes a nivel de plugin para la instancia de SearXNG:

```json5
{
  plugins: {
    entries: {
      searxng: {
        config: {
          webSearch: {
            baseUrl: "http://localhost:8888",
            categories: "general,news", // optional
            language: "en", // optional
          },
        },
      },
    },
  },
}
```

El campo `baseUrl` también acepta objetos SecretRef.

Reglas de transporte:

- `https://` funciona para hosts de SearXNG públicos o privados
- `http://` solo se acepta para hosts de red privada de confianza o loopback
- los hosts públicos de SearXNG deben usar `https://`

## Variable de entorno

Establece `SEARXNG_BASE_URL` como alternativa a la configuración:

```bash
export SEARXNG_BASE_URL="http://localhost:8888"
```

Cuando `SEARXNG_BASE_URL` está establecida y no hay ningún proveedor configurado explícitamente, la detección automática selecciona SearXNG automáticamente (con la prioridad más baja -- cualquier proveedor respaldado por API con clave gana primero).

## Referencia de configuración del plugin

| Campo        | Descripción                                                         |
| ------------ | ------------------------------------------------------------------- |
| `baseUrl`    | URL base de tu instancia de SearXNG (obligatorio)                   |
| `categories` | Categorías separadas por comas como `general`, `news` o `science`   |
| `language`   | Código de idioma para los resultados, como `en`, `de` o `fr`        |

## Notas

- **API JSON** -- usa el endpoint nativo `format=json` de SearXNG, no scraping HTML
- **Sin clave API** -- funciona con cualquier instancia de SearXNG desde el primer momento
- **Validación de URL base** -- `baseUrl` debe ser una URL válida `http://` o `https://`; los hosts públicos deben usar `https://`
- **Orden de detección automática** -- SearXNG se comprueba al final (orden 200) en la detección automática. Los proveedores respaldados por API con claves configuradas se ejecutan primero, luego DuckDuckGo (orden 100) y después Ollama Web Search (orden 110)
- **Autoalojado** -- tú controlas la instancia, las consultas y los motores de búsqueda ascendentes
- **`categories`** usa `general` por defecto cuando no está configurado

<Tip>
  Para que la API JSON de SearXNG funcione, asegúrate de que tu instancia de SearXNG tenga habilitado el formato `json` en `settings.yml` bajo `search.formats`.
</Tip>

## Relacionado

- [Descripción general de Web Search](/tools/web) -- todos los proveedores y detección automática
- [DuckDuckGo Search](/tools/duckduckgo-search) -- otra alternativa sin claves
- [Brave Search](/tools/brave-search) -- resultados estructurados con nivel gratuito
