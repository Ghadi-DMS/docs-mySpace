---
read_when:
    - Quieres modelos GLM en OpenClaw
    - Necesitas la convención de nombres de modelos y la configuración
summary: Resumen de la familia de modelos GLM + cómo usarla en OpenClaw
title: Modelos GLM
x-i18n:
    generated_at: "2026-04-05T12:51:04Z"
    model: gpt-5.4
    provider: openai
    source_hash: 59622edab5094d991987f9788fbf08b33325e737e7ff88632b0c3ac89412d4c7
    source_path: providers/glm.md
    workflow: 15
---

# Modelos GLM

GLM es una **familia de modelos** (no una empresa) disponible a través de la plataforma Z.AI. En OpenClaw, los modelos GLM
se acceden mediante el proveedor `zai` y IDs de modelo como `zai/glm-5`.

## Configuración de CLI

```bash
# Configuración genérica con API key y autodetección del endpoint
openclaw onboard --auth-choice zai-api-key

# Coding Plan Global, recomendado para usuarios de Coding Plan
openclaw onboard --auth-choice zai-coding-global

# Coding Plan CN (región de China), recomendado para usuarios de Coding Plan
openclaw onboard --auth-choice zai-coding-cn

# API general
openclaw onboard --auth-choice zai-global

# API general CN (región de China)
openclaw onboard --auth-choice zai-cn
```

## Fragmento de configuración

```json5
{
  env: { ZAI_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "zai/glm-5" } } },
}
```

`zai-api-key` permite que OpenClaw detecte el endpoint Z.AI correspondiente a partir de la clave y
aplique automáticamente la URL base correcta. Usa las opciones regionales explícitas cuando
quieras forzar una superficie concreta de Coding Plan o de la API general.

## Modelos GLM integrados actuales

OpenClaw actualmente inicializa el proveedor integrado `zai` con estas referencias GLM:

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

- Las versiones y la disponibilidad de GLM pueden cambiar; consulta la documentación de Z.AI para ver las últimas novedades.
- La referencia de modelo integrada predeterminada es `zai/glm-5`.
- Para detalles del proveedor, consulta [/providers/zai](/providers/zai).
