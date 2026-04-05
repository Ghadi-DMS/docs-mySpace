---
read_when:
    - Quieres modelos Z.AI / GLM en OpenClaw
    - Necesitas una configuración sencilla con ZAI_API_KEY
summary: Usa Z.AI (modelos GLM) con OpenClaw
title: Z.AI
x-i18n:
    generated_at: "2026-04-05T12:52:16Z"
    model: gpt-5.4
    provider: openai
    source_hash: 48006cdd580484f0c62e2877b27a6a68d7bc44795b3e97a28213d95182d9acf9
    source_path: providers/zai.md
    workflow: 15
---

# Z.AI

Z.AI es la plataforma de API para modelos **GLM**. Proporciona API REST para GLM y usa API keys
para autenticación. Crea tu API key en la consola de Z.AI. OpenClaw usa el proveedor `zai`
con una API key de Z.AI.

## Configuración por CLI

```bash
# Generic API-key setup with endpoint auto-detection
openclaw onboard --auth-choice zai-api-key

# Coding Plan Global, recommended for Coding Plan users
openclaw onboard --auth-choice zai-coding-global

# Coding Plan CN (China region), recommended for Coding Plan users
openclaw onboard --auth-choice zai-coding-cn

# General API
openclaw onboard --auth-choice zai-global

# General API CN (China region)
openclaw onboard --auth-choice zai-cn
```

## Fragmento de configuración

```json5
{
  env: { ZAI_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "zai/glm-5" } } },
}
```

`zai-api-key` permite que OpenClaw detecte el endpoint correspondiente de Z.AI a partir de la clave y
aplique automáticamente la URL base correcta. Usa las opciones regionales explícitas cuando
quieras forzar una superficie concreta de Coding Plan o de API general.

## Catálogo GLM empaquetado

OpenClaw actualmente inicializa el proveedor empaquetado `zai` con:

- `glm-5.1`
- `glm-5`
- `glm-5-turbo`
- `glm-5v-turbo`
- `glm-4.7`
- `glm-4.7-flash`
- `glm-4.7-flashx`
- `glm-4.6`
- `glm-4.6v`
- `glm-4.5`
- `glm-4.5-air`
- `glm-4.5-flash`
- `glm-4.5v`

## Notas

- Los modelos GLM están disponibles como `zai/<model>` (ejemplo: `zai/glm-5`).
- Referencia del modelo empaquetado predeterminado: `zai/glm-5`
- Los ids `glm-5*` desconocidos siguen resolviéndose por reenvío en la ruta del proveedor empaquetado
  sintetizando metadatos propiedad del proveedor a partir de la plantilla `glm-4.7` cuando el id
  coincide con la forma actual de la familia GLM-5.
- `tool_stream` está habilitado de forma predeterminada para el streaming de llamadas a herramientas de Z.AI. Establece
  `agents.defaults.models["zai/<model>"].params.tool_stream` en `false` para desactivarlo.
- Consulta [/providers/glm](/providers/glm) para ver el resumen de la familia de modelos.
- Z.AI usa autenticación Bearer con tu API key.
