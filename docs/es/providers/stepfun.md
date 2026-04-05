---
read_when:
    - Quieres modelos StepFun en OpenClaw
    - Necesitas orientación de configuración para StepFun
summary: Usar modelos StepFun con OpenClaw
title: StepFun
x-i18n:
    generated_at: "2026-04-05T12:52:03Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3154852556577b4cfb387a2de281559f2b173c774bfbcaea996abe5379ae684a
    source_path: providers/stepfun.md
    workflow: 15
---

# StepFun

OpenClaw incluye un plugin de proveedor StepFun integrado con dos id de proveedor:

- `stepfun` para el endpoint estándar
- `stepfun-plan` para el endpoint Step Plan

Los catálogos integrados actualmente difieren por superficie:

- Estándar: `step-3.5-flash`
- Step Plan: `step-3.5-flash`, `step-3.5-flash-2603`

## Resumen de región y endpoint

- Endpoint estándar de China: `https://api.stepfun.com/v1`
- Endpoint estándar global: `https://api.stepfun.ai/v1`
- Endpoint Step Plan de China: `https://api.stepfun.com/step_plan/v1`
- Endpoint Step Plan global: `https://api.stepfun.ai/step_plan/v1`
- Variable de entorno de autenticación: `STEPFUN_API_KEY`

Usa una clave de China con los endpoints `.com` y una clave global con los
endpoints `.ai`.

## Configuración por CLI

Configuración interactiva:

```bash
openclaw onboard
```

Elige una de estas opciones de autenticación:

- `stepfun-standard-api-key-cn`
- `stepfun-standard-api-key-intl`
- `stepfun-plan-api-key-cn`
- `stepfun-plan-api-key-intl`

Ejemplos no interactivos:

```bash
openclaw onboard --auth-choice stepfun-standard-api-key-intl --stepfun-api-key "$STEPFUN_API_KEY"
openclaw onboard --auth-choice stepfun-plan-api-key-intl --stepfun-api-key "$STEPFUN_API_KEY"
```

## Referencias de modelo

- Modelo estándar predeterminado: `stepfun/step-3.5-flash`
- Modelo predeterminado de Step Plan: `stepfun-plan/step-3.5-flash`
- Modelo alternativo de Step Plan: `stepfun-plan/step-3.5-flash-2603`

## Catálogos integrados

Estándar (`stepfun`):

| Referencia de modelo      | Contexto | Salida máxima | Notas                     |
| ------------------------- | -------- | ------------- | ------------------------- |
| `stepfun/step-3.5-flash`  | 262,144  | 65,536        | Modelo estándar predeterminado |

Step Plan (`stepfun-plan`):

| Referencia de modelo                | Contexto | Salida máxima | Notas                         |
| ----------------------------------- | -------- | ------------- | ----------------------------- |
| `stepfun-plan/step-3.5-flash`       | 262,144  | 65,536        | Modelo predeterminado de Step Plan |
| `stepfun-plan/step-3.5-flash-2603`  | 262,144  | 65,536        | Modelo adicional de Step Plan |

## Fragmentos de configuración

Proveedor estándar:

```json5
{
  env: { STEPFUN_API_KEY: "your-key" },
  agents: { defaults: { model: { primary: "stepfun/step-3.5-flash" } } },
  models: {
    mode: "merge",
    providers: {
      stepfun: {
        baseUrl: "https://api.stepfun.ai/v1",
        api: "openai-completions",
        apiKey: "${STEPFUN_API_KEY}",
        models: [
          {
            id: "step-3.5-flash",
            name: "Step 3.5 Flash",
            reasoning: true,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 262144,
            maxTokens: 65536,
          },
        ],
      },
    },
  },
}
```

Proveedor Step Plan:

```json5
{
  env: { STEPFUN_API_KEY: "your-key" },
  agents: { defaults: { model: { primary: "stepfun-plan/step-3.5-flash" } } },
  models: {
    mode: "merge",
    providers: {
      "stepfun-plan": {
        baseUrl: "https://api.stepfun.ai/step_plan/v1",
        api: "openai-completions",
        apiKey: "${STEPFUN_API_KEY}",
        models: [
          {
            id: "step-3.5-flash",
            name: "Step 3.5 Flash",
            reasoning: true,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 262144,
            maxTokens: 65536,
          },
          {
            id: "step-3.5-flash-2603",
            name: "Step 3.5 Flash 2603",
            reasoning: true,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 262144,
            maxTokens: 65536,
          },
        ],
      },
    },
  },
}
```

## Notas

- El proveedor está incluido con OpenClaw, por lo que no hay un paso separado de instalación del plugin.
- `step-3.5-flash-2603` actualmente solo se expone en `stepfun-plan`.
- Un único flujo de autenticación escribe perfiles ajustados a la región para `stepfun` y `stepfun-plan`, de modo que ambas superficies puedan descubrirse juntas.
- Usa `openclaw models list` y `openclaw models set <provider/model>` para inspeccionar o cambiar modelos.
- Para una vista general más amplia de los proveedores, consulta [Proveedores de modelos](/concepts/model-providers).
