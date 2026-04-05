---
read_when:
    - Quieres usar MiniMax para `web_search`
    - Necesitas una clave de MiniMax Coding Plan
    - Quieres orientación sobre los hosts de búsqueda de MiniMax CN/global
summary: MiniMax Search mediante la API de búsqueda de Coding Plan
title: MiniMax Search
x-i18n:
    generated_at: "2026-04-05T12:56:01Z"
    model: gpt-5.4
    provider: openai
    source_hash: b8c3767790f428fc7e239590a97e9dbee0d3bd6550ca3299ae22da0f5a57231a
    source_path: tools/minimax-search.md
    workflow: 15
---

# MiniMax Search

OpenClaw admite MiniMax como proveedor de `web_search` mediante la API de búsqueda de MiniMax
Coding Plan. Devuelve resultados de búsqueda estructurados con títulos, URL,
fragmentos y consultas relacionadas.

## Obtener una clave de Coding Plan

<Steps>
  <Step title="Crear una clave">
    Crea o copia una clave de MiniMax Coding Plan desde
    [MiniMax Platform](https://platform.minimax.io/user-center/basic-information/interface-key).
  </Step>
  <Step title="Guardar la clave">
    Configura `MINIMAX_CODE_PLAN_KEY` en el entorno del Gateway, o configúralo mediante:

    ```bash
    openclaw configure --section web
    ```

  </Step>
</Steps>

OpenClaw también acepta `MINIMAX_CODING_API_KEY` como alias de variable de entorno. `MINIMAX_API_KEY`
se sigue leyendo como respaldo de compatibilidad cuando ya apunta a un token de coding plan.

## Configuración

```json5
{
  plugins: {
    entries: {
      minimax: {
        config: {
          webSearch: {
            apiKey: "sk-cp-...", // opcional si MINIMAX_CODE_PLAN_KEY está configurada
            region: "global", // o "cn"
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "minimax",
      },
    },
  },
}
```

**Alternativa con variable de entorno:** configura `MINIMAX_CODE_PLAN_KEY` en el entorno del Gateway.
Para una instalación del gateway, colócala en `~/.openclaw/.env`.

## Selección de región

MiniMax Search usa estos endpoints:

- Global: `https://api.minimax.io/v1/coding_plan/search`
- CN: `https://api.minimaxi.com/v1/coding_plan/search`

Si `plugins.entries.minimax.config.webSearch.region` no está configurado, OpenClaw resuelve
la región en este orden:

1. `tools.web.search.minimax.region` / `webSearch.region` propiedad del plugin
2. `MINIMAX_API_HOST`
3. `models.providers.minimax.baseUrl`
4. `models.providers.minimax-portal.baseUrl`

Eso significa que la configuración inicial de CN o `MINIMAX_API_HOST=https://api.minimaxi.com/...`
mantiene automáticamente MiniMax Search también en el host CN.

Incluso cuando autenticaste MiniMax mediante la ruta OAuth `minimax-portal`,
la búsqueda web sigue registrándose como id de proveedor `minimax`; la URL base del proveedor OAuth
solo se usa como sugerencia de región para la selección del host CN/global.

## Parámetros admitidos

MiniMax Search admite:

- `query`
- `count` (OpenClaw recorta la lista de resultados devuelta al recuento solicitado)

Los filtros específicos del proveedor no se admiten actualmente.

## Relacionado

- [Resumen de Web Search](/tools/web) -- todos los proveedores y la detección automática
- [MiniMax](/es/providers/minimax) -- configuración de modelo, imagen, voz y autenticación
