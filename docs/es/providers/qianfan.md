---
read_when:
    - Quieres una sola clave API para muchos LLM
    - Necesitas orientación de configuración para Baidu Qianfan
summary: Usar la API unificada de Qianfan para acceder a muchos modelos en OpenClaw
title: Qianfan
x-i18n:
    generated_at: "2026-04-05T12:51:47Z"
    model: gpt-5.4
    provider: openai
    source_hash: 965d83dd968563447ce3571a73bd71c6876275caff8664311a852b2f9827e55b
    source_path: providers/qianfan.md
    workflow: 15
---

# Guía del proveedor Qianfan

Qianfan es la plataforma MaaS de Baidu y ofrece una **API unificada** que enruta solicitudes a muchos modelos detrás de un único
endpoint y una única clave API. Es compatible con OpenAI, por lo que la mayoría de SDK de OpenAI funcionan cambiando la URL base.

## Requisitos previos

1. Una cuenta de Baidu Cloud con acceso a la API de Qianfan
2. Una clave API de la consola de Qianfan
3. OpenClaw instalado en tu sistema

## Obtener tu clave API

1. Visita la [Consola de Qianfan](https://console.bce.baidu.com/qianfan/ais/console/apiKey)
2. Crea una nueva aplicación o selecciona una existente
3. Genera una clave API (formato: `bce-v3/ALTAK-...`)
4. Copia la clave API para usarla con OpenClaw

## Configuración por CLI

```bash
openclaw onboard --auth-choice qianfan-api-key
```

## Fragmento de configuración

```json5
{
  env: { QIANFAN_API_KEY: "bce-v3/ALTAK-..." },
  agents: {
    defaults: {
      model: { primary: "qianfan/deepseek-v3.2" },
      models: {
        "qianfan/deepseek-v3.2": { alias: "QIANFAN" },
      },
    },
  },
  models: {
    providers: {
      qianfan: {
        baseUrl: "https://qianfan.baidubce.com/v2",
        api: "openai-completions",
        models: [
          {
            id: "deepseek-v3.2",
            name: "DEEPSEEK V3.2",
            reasoning: true,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 98304,
            maxTokens: 32768,
          },
          {
            id: "ernie-5.0-thinking-preview",
            name: "ERNIE-5.0-Thinking-Preview",
            reasoning: true,
            input: ["text", "image"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 119000,
            maxTokens: 64000,
          },
        ],
      },
    },
  },
}
```

## Notas

- Referencia de modelo integrada predeterminada: `qianfan/deepseek-v3.2`
- URL base predeterminada: `https://qianfan.baidubce.com/v2`
- El catálogo integrado actualmente incluye `deepseek-v3.2` y `ernie-5.0-thinking-preview`
- Añade o anula `models.providers.qianfan` solo cuando necesites una URL base personalizada o metadatos de modelo personalizados
- Qianfan se ejecuta mediante la ruta de transporte compatible con OpenAI, no con la conformación de solicitudes nativa de OpenAI

## Documentación relacionada

- [Configuración de OpenClaw](/gateway/configuration)
- [Proveedores de modelos](/concepts/model-providers)
- [Configuración del agente](/concepts/agent)
- [Documentación de la API de Qianfan](https://cloud.baidu.com/doc/qianfan-api/s/3m7of64lb)
