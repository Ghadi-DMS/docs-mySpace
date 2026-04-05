---
read_when:
    - Quieres una única clave de API para muchos LLM
    - Quieres ejecutar modelos mediante OpenRouter en OpenClaw
summary: Usa la API unificada de OpenRouter para acceder a muchos modelos en OpenClaw
title: OpenRouter
x-i18n:
    generated_at: "2026-04-05T12:51:42Z"
    model: gpt-5.4
    provider: openai
    source_hash: 8dd354ba060bcb47724c89ae17c8e2af8caecac4bd996fcddb584716c1840b87
    source_path: providers/openrouter.md
    workflow: 15
---

# OpenRouter

OpenRouter proporciona una **API unificada** que enruta solicitudes a muchos modelos detrás de un único
endpoint y una única clave de API. Es compatible con OpenAI, así que la mayoría de los SDK de OpenAI funcionan cambiando la URL base.

## Configuración de la CLI

```bash
openclaw onboard --auth-choice openrouter-api-key
```

## Fragmento de configuración

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      model: { primary: "openrouter/auto" },
    },
  },
}
```

## Notas

- Las referencias de modelo son `openrouter/<provider>/<model>`.
- Onboarding usa `openrouter/auto` como valor predeterminado. Cambia después a un modelo concreto con
  `openclaw models set openrouter/<provider>/<model>`.
- Para ver más opciones de modelos/proveedores, consulta [/concepts/model-providers](/concepts/model-providers).
- OpenRouter usa internamente un token Bearer con tu clave de API.
- En solicitudes reales a OpenRouter (`https://openrouter.ai/api/v1`), OpenClaw también
  añade los encabezados documentados de atribución de aplicación de OpenRouter:
  `HTTP-Referer: https://openclaw.ai`, `X-OpenRouter-Title: OpenClaw` y
  `X-OpenRouter-Categories: cli-agent`.
- En rutas OpenRouter verificadas, las referencias de modelo de Anthropic también conservan los
  marcadores `cache_control` específicos de OpenRouter que OpenClaw usa para
  una mejor reutilización de la caché de prompts en bloques de prompt del sistema/desarrollador.
- Si rediriges el proveedor OpenRouter a algún otro proxy/URL base, OpenClaw
  no inyecta esos encabezados específicos de OpenRouter ni los marcadores de caché de Anthropic.
- OpenRouter sigue funcionando a través de la ruta compatible con OpenAI de estilo proxy, así que
  el modelado de solicitudes nativo exclusivo de OpenAI, como `serviceTier`, `store` de Responses,
  las cargas útiles de compatibilidad de razonamiento de OpenAI y las pistas de caché de prompts, no se reenvía.
- Las referencias de OpenRouter respaldadas por Gemini permanecen en la ruta proxy-Gemini: OpenClaw mantiene
  allí el saneamiento de firmas de pensamiento de Gemini, pero no habilita la validación de repetición nativa de Gemini ni las reescrituras bootstrap.
- En rutas compatibles que no sean `auto`, OpenClaw asigna el nivel de pensamiento seleccionado a cargas útiles de razonamiento del proxy de OpenRouter. Las pistas de modelos no compatibles y
  `openrouter/auto` omiten esa inyección de razonamiento.
- Si pasas enrutamiento del proveedor OpenRouter en los parámetros del modelo, OpenClaw lo reenvía
  como metadatos de enrutamiento de OpenRouter antes de que se ejecuten los envoltorios de flujo compartidos.
