---
read_when:
    - Quieres usar modelos Grok en OpenClaw
    - Estás configurando la autenticación de xAI o los IDs de modelo
summary: Usar modelos Grok de xAI en OpenClaw
title: xAI
x-i18n:
    generated_at: "2026-04-05T12:52:16Z"
    model: gpt-5.4
    provider: openai
    source_hash: d11f27b48c69eed6324595977bca3506c7709424eef64cc73899f8d049148b82
    source_path: providers/xai.md
    workflow: 15
---

# xAI

OpenClaw incluye un plugin integrado del proveedor `xai` para modelos Grok.

## Configuración

1. Crea una API key en la consola de xAI.
2. Establece `XAI_API_KEY`, o ejecuta:

```bash
openclaw onboard --auth-choice xai-api-key
```

3. Elige un modelo como:

```json5
{
  agents: { defaults: { model: { primary: "xai/grok-4" } } },
}
```

OpenClaw ahora usa la API Responses de xAI como transporte integrado de xAI. La misma
`XAI_API_KEY` también puede alimentar `web_search` respaldado por Grok, `x_search` de primera clase
y `code_execution` remoto.
Si guardas una clave de xAI en `plugins.entries.xai.config.webSearch.apiKey`,
el proveedor integrado de modelos xAI ahora también reutiliza esa clave como fallback.
El ajuste de `code_execution` vive en `plugins.entries.xai.config.codeExecution`.

## Catálogo actual de modelos integrados

OpenClaw ahora incluye estas familias de modelos xAI listas para usar:

- `grok-3`, `grok-3-fast`, `grok-3-mini`, `grok-3-mini-fast`
- `grok-4`, `grok-4-0709`
- `grok-4-fast`, `grok-4-fast-non-reasoning`
- `grok-4-1-fast`, `grok-4-1-fast-non-reasoning`
- `grok-4.20-beta-latest-reasoning`, `grok-4.20-beta-latest-non-reasoning`
- `grok-code-fast-1`

El plugin también resuelve hacia adelante IDs más nuevos `grok-4*` y `grok-code-fast*` cuando
siguen la misma forma de API.

Notas sobre modelos rápidos:

- `grok-4-fast`, `grok-4-1-fast` y las variantes `grok-4.20-beta-*` son las
  referencias Grok actuales con capacidad de imagen en el catálogo integrado.
- `/fast on` o `agents.defaults.models["xai/<model>"].params.fastMode: true`
  reescriben las solicitudes nativas de xAI de la siguiente manera:
  - `grok-3` -> `grok-3-fast`
  - `grok-3-mini` -> `grok-3-mini-fast`
  - `grok-4` -> `grok-4-fast`
  - `grok-4-0709` -> `grok-4-fast`

Los alias heredados de compatibilidad siguen normalizándose a los IDs integrados canónicos. Por
ejemplo:

- `grok-4-fast-reasoning` -> `grok-4-fast`
- `grok-4-1-fast-reasoning` -> `grok-4-1-fast`
- `grok-4.20-reasoning` -> `grok-4.20-beta-latest-reasoning`
- `grok-4.20-non-reasoning` -> `grok-4.20-beta-latest-non-reasoning`

## Búsqueda web

El proveedor integrado de búsqueda web `grok` también usa `XAI_API_KEY`:

```bash
openclaw config set tools.web.search.provider grok
```

## Límites conocidos

- La autenticación hoy solo es por API key. OpenClaw todavía no tiene flujo OAuth/device-code para xAI.
- `grok-4.20-multi-agent-experimental-beta-0304` no es compatible en la ruta normal del proveedor xAI porque requiere una superficie de API upstream distinta del transporte estándar xAI de OpenClaw.

## Notas

- OpenClaw aplica automáticamente correcciones de compatibilidad específicas de xAI para esquemas de herramientas y llamadas de herramientas en la ruta compartida del ejecutor.
- Las solicitudes nativas de xAI usan por defecto `tool_stream: true`. Establece
  `agents.defaults.models["xai/<model>"].params.tool_stream` en `false` para
  desactivarlo.
- El wrapper integrado de xAI elimina las banderas de esquema estricto de herramientas no admitidas y
  las claves de carga de razonamiento antes de enviar solicitudes nativas de xAI.
- `web_search`, `x_search` y `code_execution` se exponen como herramientas de OpenClaw. OpenClaw habilita la función integrada específica de xAI que necesita dentro de cada solicitud de herramienta en lugar de adjuntar todas las herramientas nativas a cada turno de chat.
- `x_search` y `code_execution` son propiedad del plugin integrado xAI en lugar de estar codificados directamente en el runtime central del modelo.
- `code_execution` es ejecución remota en sandbox de xAI, no [`exec`](/tools/exec) local.
- Para ver la visión general más amplia del proveedor, consulta [Model providers](/providers/index).
