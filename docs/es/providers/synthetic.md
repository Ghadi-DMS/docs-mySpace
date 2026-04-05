---
read_when:
    - Quieres usar Synthetic como proveedor de modelos
    - Necesitas una API key de Synthetic o configurar la URL base
summary: Usar la API compatible con Anthropic de Synthetic en OpenClaw
title: Synthetic
x-i18n:
    generated_at: "2026-04-05T12:52:03Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3495bca5cb134659cf6c54e31fa432989afe0cc04f53cf3e3146ce80a5e8af49
    source_path: providers/synthetic.md
    workflow: 15
---

# Synthetic

Synthetic expone endpoints compatibles con Anthropic. OpenClaw lo registra como el
proveedor `synthetic` y usa la API Anthropic Messages.

## Configuración rápida

1. Establece `SYNTHETIC_API_KEY` (o ejecuta el asistente siguiente).
2. Ejecuta onboarding:

```bash
openclaw onboard --auth-choice synthetic-api-key
```

El modelo predeterminado se establece en:

```
synthetic/hf:MiniMaxAI/MiniMax-M2.5
```

## Ejemplo de configuración

```json5
{
  env: { SYNTHETIC_API_KEY: "sk-..." },
  agents: {
    defaults: {
      model: { primary: "synthetic/hf:MiniMaxAI/MiniMax-M2.5" },
      models: { "synthetic/hf:MiniMaxAI/MiniMax-M2.5": { alias: "MiniMax M2.5" } },
    },
  },
  models: {
    mode: "merge",
    providers: {
      synthetic: {
        baseUrl: "https://api.synthetic.new/anthropic",
        apiKey: "${SYNTHETIC_API_KEY}",
        api: "anthropic-messages",
        models: [
          {
            id: "hf:MiniMaxAI/MiniMax-M2.5",
            name: "MiniMax M2.5",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 192000,
            maxTokens: 65536,
          },
        ],
      },
    },
  },
}
```

Nota: el cliente Anthropic de OpenClaw añade `/v1` a la URL base, así que usa
`https://api.synthetic.new/anthropic` (no `/anthropic/v1`). Si Synthetic cambia
su URL base, sobrescribe `models.providers.synthetic.baseUrl`.

## Catálogo de modelos

Todos los modelos de abajo usan coste `0` (entrada/salida/caché).

| ID de modelo                                           | Ventana de contexto | Máx. tokens | Razonamiento | Entrada       |
| ------------------------------------------------------ | ------------------- | ----------- | ------------ | ------------- |
| `hf:MiniMaxAI/MiniMax-M2.5`                            | 192000              | 65536       | false        | text          |
| `hf:moonshotai/Kimi-K2-Thinking`                       | 256000              | 8192        | true         | text          |
| `hf:zai-org/GLM-4.7`                                   | 198000              | 128000      | false        | text          |
| `hf:deepseek-ai/DeepSeek-R1-0528`                      | 128000              | 8192        | false        | text          |
| `hf:deepseek-ai/DeepSeek-V3-0324`                      | 128000              | 8192        | false        | text          |
| `hf:deepseek-ai/DeepSeek-V3.1`                         | 128000              | 8192        | false        | text          |
| `hf:deepseek-ai/DeepSeek-V3.1-Terminus`                | 128000              | 8192        | false        | text          |
| `hf:deepseek-ai/DeepSeek-V3.2`                         | 159000              | 8192        | false        | text          |
| `hf:meta-llama/Llama-3.3-70B-Instruct`                 | 128000              | 8192        | false        | text          |
| `hf:meta-llama/Llama-4-Maverick-17B-128E-Instruct-FP8` | 524000              | 8192        | false        | text          |
| `hf:moonshotai/Kimi-K2-Instruct-0905`                  | 256000              | 8192        | false        | text          |
| `hf:moonshotai/Kimi-K2.5`                              | 256000              | 8192        | true         | text + image  |
| `hf:openai/gpt-oss-120b`                               | 128000              | 8192        | false        | text          |
| `hf:Qwen/Qwen3-235B-A22B-Instruct-2507`                | 256000              | 8192        | false        | text          |
| `hf:Qwen/Qwen3-Coder-480B-A35B-Instruct`               | 256000              | 8192        | false        | text          |
| `hf:Qwen/Qwen3-VL-235B-A22B-Instruct`                  | 250000              | 8192        | false        | text + image  |
| `hf:zai-org/GLM-4.5`                                   | 128000              | 128000      | false        | text          |
| `hf:zai-org/GLM-4.6`                                   | 198000              | 128000      | false        | text          |
| `hf:zai-org/GLM-5`                                     | 256000              | 128000      | true         | text + image  |
| `hf:deepseek-ai/DeepSeek-V3`                           | 128000              | 8192        | false        | text          |
| `hf:Qwen/Qwen3-235B-A22B-Thinking-2507`                | 256000              | 8192        | true         | text          |

## Notas

- Las referencias de modelo usan `synthetic/<modelId>`.
- Si habilitas una lista de permitidos de modelos (`agents.defaults.models`), añade todos los modelos que
  planees usar.
- Consulta [Model providers](/concepts/model-providers) para ver las reglas de proveedores.
