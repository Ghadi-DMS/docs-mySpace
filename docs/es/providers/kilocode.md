---
read_when:
    - Quieres una sola clave de API para muchos LLM
    - Quieres ejecutar modelos mediante Kilo Gateway en OpenClaw
summary: Usa la API unificada de Kilo Gateway para acceder a muchos modelos en OpenClaw
title: Kilo Gateway
x-i18n:
    generated_at: "2026-04-05T12:51:18Z"
    model: gpt-5.4
    provider: openai
    source_hash: 857266967b4a7553d501990631df2bae0f849d061521dc9f34e29687ecb94884
    source_path: providers/kilocode.md
    workflow: 15
---

# Kilo Gateway

Kilo Gateway proporciona una **API unificada** que enruta solicitudes a muchos modelos detrás de un único
endpoint y una única clave de API. Es compatible con OpenAI, por lo que la mayoría de los SDK de OpenAI funcionan cambiando la URL base.

## Obtener una clave de API

1. Ve a [app.kilo.ai](https://app.kilo.ai)
2. Inicia sesión o crea una cuenta
3. Ve a API Keys y genera una clave nueva

## Configuración por CLI

```bash
openclaw onboard --auth-choice kilocode-api-key
```

O configura la variable de entorno:

```bash
export KILOCODE_API_KEY="<your-kilocode-api-key>" # pragma: allowlist secret
```

## Fragmento de configuración

```json5
{
  env: { KILOCODE_API_KEY: "<your-kilocode-api-key>" }, // pragma: allowlist secret
  agents: {
    defaults: {
      model: { primary: "kilocode/kilo/auto" },
    },
  },
}
```

## Modelo predeterminado

El modelo predeterminado es `kilocode/kilo/auto`, un modelo de enrutamiento inteligente
propiedad del proveedor y gestionado por Kilo Gateway.

OpenClaw trata `kilocode/kilo/auto` como la referencia predeterminada estable, pero no
publica una asignación respaldada por la fuente entre tarea y modelo upstream para esa ruta.

## Modelos disponibles

OpenClaw detecta dinámicamente los modelos disponibles desde Kilo Gateway al iniciarse. Usa
`/models kilocode` para ver la lista completa de modelos disponibles con tu cuenta.

Cualquier modelo disponible en el gateway puede usarse con el prefijo `kilocode/`:

```
kilocode/kilo/auto              (predeterminado - enrutamiento inteligente)
kilocode/anthropic/claude-sonnet-4
kilocode/openai/gpt-5.4
kilocode/google/gemini-3-pro-preview
...y muchos más
```

## Notas

- Las referencias de modelo son `kilocode/<model-id>` (por ejemplo, `kilocode/anthropic/claude-sonnet-4`).
- Modelo predeterminado: `kilocode/kilo/auto`
- URL base: `https://api.kilo.ai/api/gateway/`
- El catálogo integrado de respaldo siempre incluye `kilocode/kilo/auto` (`Kilo Auto`) con
  `input: ["text", "image"]`, `reasoning: true`, `contextWindow: 1000000`
  y `maxTokens: 128000`
- Al iniciarse, OpenClaw intenta `GET https://api.kilo.ai/api/gateway/models` y
  combina los modelos detectados antes del catálogo estático de respaldo
- El enrutamiento exacto upstream detrás de `kilocode/kilo/auto` pertenece a Kilo Gateway,
  no está codificado de forma rígida en OpenClaw
- Kilo Gateway está documentado en el código fuente como compatible con OpenRouter, por lo que permanece en
  la ruta de proxy compatible con OpenAI en lugar del modelado nativo de solicitudes de OpenAI
- Las referencias Kilo respaldadas por Gemini permanecen en la ruta de proxy-Gemini, por lo que OpenClaw conserva
  allí la sanitización de la firma de pensamiento de Gemini sin habilitar la
  validación nativa de replay de Gemini ni reescrituras de bootstrap.
- El envoltorio compartido de stream de Kilo añade la cabecera de aplicación del proveedor y normaliza
  payloads de reasoning de proxy para referencias concretas de modelos compatibles. `kilocode/kilo/auto`
  y otras pistas sin compatibilidad con reasoning de proxy omiten esa inyección de reasoning.
- Para más opciones de modelos/proveedores, consulta [/concepts/model-providers](/concepts/model-providers).
- Kilo Gateway usa internamente un token Bearer con tu clave de API.
