---
read_when:
    - Quieres modelos Xiaomi MiMo en OpenClaw
    - Necesitas configurar `XIAOMI_API_KEY`
summary: Usa modelos Xiaomi MiMo con OpenClaw
title: Xiaomi MiMo
x-i18n:
    generated_at: "2026-04-05T12:52:17Z"
    model: gpt-5.4
    provider: openai
    source_hash: a2533fa99b29070e26e0e1fbde924e1291c89b1fbc2537451bcc0eb677ea6949
    source_path: providers/xiaomi.md
    workflow: 15
---

# Xiaomi MiMo

Xiaomi MiMo es la plataforma API para modelos **MiMo**. OpenClaw usa el endpoint
compatible con OpenAI de Xiaomi con autenticación por clave de API. Crea tu clave de API en la
[consola de Xiaomi MiMo](https://platform.xiaomimimo.com/#/console/api-keys), luego configura el
proveedor incluido `xiaomi` con esa clave.

## Catálogo integrado

- URL base: `https://api.xiaomimimo.com/v1`
- API: `openai-completions`
- Autorización: `Bearer $XIAOMI_API_KEY`

| Referencia de modelo    | Entrada     | Contexto  | Salida máxima | Notas                        |
| ----------------------- | ----------- | --------- | ------------- | ---------------------------- |
| `xiaomi/mimo-v2-flash` | texto       | 262,144   | 8,192         | Modelo predeterminado        |
| `xiaomi/mimo-v2-pro`   | texto       | 1,048,576 | 32,000        | Reasoning habilitado         |
| `xiaomi/mimo-v2-omni`  | texto, imagen | 262,144 | 32,000        | Multimodal con reasoning habilitado |

## Configuración de CLI

```bash
openclaw onboard --auth-choice xiaomi-api-key
# or non-interactive
openclaw onboard --auth-choice xiaomi-api-key --xiaomi-api-key "$XIAOMI_API_KEY"
```

## Fragmento de configuración

```json5
{
  env: { XIAOMI_API_KEY: "your-key" },
  agents: { defaults: { model: { primary: "xiaomi/mimo-v2-flash" } } },
  models: {
    mode: "merge",
    providers: {
      xiaomi: {
        baseUrl: "https://api.xiaomimimo.com/v1",
        api: "openai-completions",
        apiKey: "XIAOMI_API_KEY",
        models: [
          {
            id: "mimo-v2-flash",
            name: "Xiaomi MiMo V2 Flash",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 262144,
            maxTokens: 8192,
          },
          {
            id: "mimo-v2-pro",
            name: "Xiaomi MiMo V2 Pro",
            reasoning: true,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 1048576,
            maxTokens: 32000,
          },
          {
            id: "mimo-v2-omni",
            name: "Xiaomi MiMo V2 Omni",
            reasoning: true,
            input: ["text", "image"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 262144,
            maxTokens: 32000,
          },
        ],
      },
    },
  },
}
```

## Notas

- Referencia del modelo predeterminado: `xiaomi/mimo-v2-flash`.
- Modelos integrados adicionales: `xiaomi/mimo-v2-pro`, `xiaomi/mimo-v2-omni`.
- El proveedor se inyecta automáticamente cuando `XIAOMI_API_KEY` está configurado (o existe un perfil de autenticación).
- Consulta [/concepts/model-providers](/concepts/model-providers) para las reglas de proveedores.
