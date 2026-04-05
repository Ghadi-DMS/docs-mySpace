---
read_when:
    - Quieres usar Exa para `web_search`
    - Necesitas un `EXA_API_KEY`
    - Quieres búsqueda neural o extracción de contenido
summary: 'Búsqueda con Exa AI: búsqueda neural y por palabras clave con extracción de contenido'
title: Búsqueda Exa
x-i18n:
    generated_at: "2026-04-05T12:55:22Z"
    model: gpt-5.4
    provider: openai
    source_hash: 307b727b4fb88756cac51c17ffd73468ca695c4481692e03d0b4a9969982a2a8
    source_path: tools/exa-search.md
    workflow: 15
---

# Búsqueda Exa

OpenClaw admite [Exa AI](https://exa.ai/) como proveedor de `web_search`. Exa
ofrece modos de búsqueda neural, por palabras clave e híbrida con
extracción de contenido integrada (resaltados, texto, resúmenes).

## Obtener una clave de API

<Steps>
  <Step title="Crea una cuenta">
    Regístrate en [exa.ai](https://exa.ai/) y genera una clave de API desde tu
    panel.
  </Step>
  <Step title="Guarda la clave">
    Establece `EXA_API_KEY` en el entorno del Gateway, o configúralo mediante:

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
      exa: {
        config: {
          webSearch: {
            apiKey: "exa-...", // opcional si EXA_API_KEY está establecido
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "exa",
      },
    },
  },
}
```

**Alternativa de entorno:** establece `EXA_API_KEY` en el entorno del Gateway.
Para una instalación del gateway, colócalo en `~/.openclaw/.env`.

## Parámetros de la herramienta

| Parámetro     | Descripción                                                                  |
| ------------- | ---------------------------------------------------------------------------- |
| `query`       | Consulta de búsqueda (obligatoria)                                           |
| `count`       | Resultados que se devolverán (1-100)                                         |
| `type`        | Modo de búsqueda: `auto`, `neural`, `fast`, `deep`, `deep-reasoning` o `instant` |
| `freshness`   | Filtro temporal: `day`, `week`, `month` o `year`                             |
| `date_after`  | Resultados posteriores a esta fecha (YYYY-MM-DD)                             |
| `date_before` | Resultados anteriores a esta fecha (YYYY-MM-DD)                              |
| `contents`    | Opciones de extracción de contenido (consulta abajo)                         |

### Extracción de contenido

Exa puede devolver contenido extraído junto con los resultados de búsqueda. Pasa un
objeto `contents` para habilitarlo:

```javascript
await web_search({
  query: "transformer architecture explained",
  type: "neural",
  contents: {
    text: true, // texto completo de la página
    highlights: { numSentences: 3 }, // frases clave
    summary: true, // resumen de IA
  },
});
```

| Opción de `contents` | Tipo                                                                  | Descripción                 |
| -------------------- | --------------------------------------------------------------------- | --------------------------- |
| `text`               | `boolean \| { maxCharacters }`                                        | Extrae el texto completo de la página |
| `highlights`         | `boolean \| { maxCharacters, query, numSentences, highlightsPerUrl }` | Extrae frases clave         |
| `summary`            | `boolean \| { query }`                                                | Resumen generado por IA     |

### Modos de búsqueda

| Modo             | Descripción                            |
| ---------------- | -------------------------------------- |
| `auto`           | Exa elige el mejor modo (predeterminado) |
| `neural`         | Búsqueda semántica/basada en significado |
| `fast`           | Búsqueda rápida por palabras clave     |
| `deep`           | Búsqueda profunda exhaustiva           |
| `deep-reasoning` | Búsqueda profunda con razonamiento     |
| `instant`        | Resultados más rápidos                 |

## Notas

- Si no se proporciona ninguna opción `contents`, Exa usa de forma predeterminada `{ highlights: true }`
  para que los resultados incluyan extractos de frases clave
- Los resultados conservan los campos `highlightScores` y `summary` de la
  respuesta de la API de Exa cuando están disponibles
- Las descripciones de los resultados se resuelven primero a partir de `highlights`, luego de `summary`, y después del
  texto completo; se usa lo que esté disponible
- `freshness` y `date_after`/`date_before` no pueden combinarse; usa un único
  modo de filtro temporal
- Se pueden devolver hasta 100 resultados por consulta (sujeto a los límites
  del tipo de búsqueda de Exa)
- Los resultados se almacenan en caché durante 15 minutos de forma predeterminada (configurable mediante
  `cacheTtlMinutes`)
- Exa es una integración oficial de API con respuestas JSON estructuradas

## Relacionado

- [Descripción general de Web Search](/tools/web) -- todos los proveedores y detección automática
- [Brave Search](/tools/brave-search) -- resultados estructurados con filtros de país/idioma
- [Perplexity Search](/tools/perplexity-search) -- resultados estructurados con filtrado por dominio
