---
read_when:
    - Quieres configurar Perplexity como proveedor de búsqueda web
    - Necesitas la clave de API de Perplexity o la configuración de proxy de OpenRouter
summary: Configuración del proveedor Perplexity para búsqueda web (clave de API, modos de búsqueda, filtrado)
title: Perplexity (proveedor)
x-i18n:
    generated_at: "2026-04-05T12:51:43Z"
    model: gpt-5.4
    provider: openai
    source_hash: df9082d15d6a36a096e21efe8cee78e4b8643252225520f5b96a0b99cf5a7a4b
    source_path: providers/perplexity-provider.md
    workflow: 15
---

# Perplexity (proveedor de búsqueda web)

El plugin de Perplexity proporciona capacidades de búsqueda web a través de la
Search API de Perplexity o Perplexity Sonar mediante OpenRouter.

<Note>
Esta página cubre la configuración del **proveedor** de Perplexity. Para la
**herramienta** de Perplexity (cómo la usa el agente), consulta [Perplexity tool](/tools/perplexity-search).
</Note>

- Tipo: proveedor de búsqueda web (no proveedor de modelos)
- Autenticación: `PERPLEXITY_API_KEY` (directo) o `OPENROUTER_API_KEY` (mediante OpenRouter)
- Ruta de configuración: `plugins.entries.perplexity.config.webSearch.apiKey`

## Inicio rápido

1. Configura la clave de API:

```bash
openclaw configure --section web
```

O configúrala directamente:

```bash
openclaw config set plugins.entries.perplexity.config.webSearch.apiKey "pplx-xxxxxxxxxxxx"
```

2. El agente usará automáticamente Perplexity para búsquedas web cuando esté configurado.

## Modos de búsqueda

El plugin selecciona automáticamente el transporte según el prefijo de la clave de API:

| Prefijo de clave | Transporte                     | Funciones                                        |
| ---------------- | ------------------------------ | ------------------------------------------------ |
| `pplx-`          | Search API nativa de Perplexity | Resultados estructurados, filtros de dominio/idioma/fecha |
| `sk-or-`         | OpenRouter (Sonar)             | Respuestas sintetizadas por IA con citas         |

## Filtrado de la API nativa

Al usar la API nativa de Perplexity (clave `pplx-`), las búsquedas admiten:

- **País**: código de país de 2 letras
- **Idioma**: código de idioma ISO 639-1
- **Intervalo de fechas**: day, week, month, year
- **Filtros de dominio**: lista de permitidos/lista de denegados (máximo 20 dominios)
- **Presupuesto de contenido**: `max_tokens`, `max_tokens_per_page`

## Nota sobre el entorno

Si el Gateway se ejecuta como daemon (launchd/systemd), asegúrate de que
`PERPLEXITY_API_KEY` esté disponible para ese proceso (por ejemplo, en
`~/.openclaw/.env` o mediante `env.shellEnv`).
