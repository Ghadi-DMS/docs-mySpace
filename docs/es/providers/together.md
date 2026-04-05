---
read_when:
    - Quieres usar Together AI con OpenClaw
    - Necesitas la variable de entorno de la clave de API o la opción de autenticación de CLI
summary: Configuración de Together AI (autenticación + selección de modelo)
title: Together AI
x-i18n:
    generated_at: "2026-04-05T12:52:05Z"
    model: gpt-5.4
    provider: openai
    source_hash: 22aacbaadf860ce8245bba921dcc5ede9da8fd6fa1bc3cc912551aecc1ba0d71
    source_path: providers/together.md
    workflow: 15
---

# Together AI

[Together AI](https://together.ai) proporciona acceso a modelos líderes de código abierto, incluidos Llama, DeepSeek, Kimi y más, a través de una API unificada.

- Proveedor: `together`
- Autenticación: `TOGETHER_API_KEY`
- API: compatible con OpenAI
- URL base: `https://api.together.xyz/v1`

## Inicio rápido

1. Configura la clave de API (recomendado: almacénala para el Gateway):

```bash
openclaw onboard --auth-choice together-api-key
```

2. Configura un modelo predeterminado:

```json5
{
  agents: {
    defaults: {
      model: { primary: "together/moonshotai/Kimi-K2.5" },
    },
  },
}
```

## Ejemplo no interactivo

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice together-api-key \
  --together-api-key "$TOGETHER_API_KEY"
```

Esto configurará `together/moonshotai/Kimi-K2.5` como modelo predeterminado.

## Nota sobre el entorno

Si el Gateway se ejecuta como daemon (launchd/systemd), asegúrate de que `TOGETHER_API_KEY`
esté disponible para ese proceso (por ejemplo, en `~/.openclaw/.env` o mediante
`env.shellEnv`).

## Catálogo integrado

OpenClaw incluye actualmente este catálogo integrado de Together:

| Referencia de modelo                                          | Nombre                                 | Entrada     | Contexto   | Notas                            |
| ------------------------------------------------------------- | -------------------------------------- | ----------- | ---------- | -------------------------------- |
| `together/moonshotai/Kimi-K2.5`                              | Kimi K2.5                              | texto, imagen | 262,144  | Modelo predeterminado; reasoning habilitado |
| `together/zai-org/GLM-4.7`                                   | GLM 4.7 Fp8                            | texto       | 202,752    | Modelo de texto de propósito general |
| `together/meta-llama/Llama-3.3-70B-Instruct-Turbo`           | Llama 3.3 70B Instruct Turbo           | texto       | 131,072    | Modelo rápido de instrucciones   |
| `together/meta-llama/Llama-4-Scout-17B-16E-Instruct`         | Llama 4 Scout 17B 16E Instruct         | texto, imagen | 10,000,000 | Multimodal                     |
| `together/meta-llama/Llama-4-Maverick-17B-128E-Instruct-FP8` | Llama 4 Maverick 17B 128E Instruct FP8 | texto, imagen | 20,000,000 | Multimodal                     |
| `together/deepseek-ai/DeepSeek-V3.1`                         | DeepSeek V3.1                          | texto       | 131,072    | Modelo general de texto          |
| `together/deepseek-ai/DeepSeek-R1`                           | DeepSeek R1                            | texto       | 131,072    | Modelo de razonamiento           |
| `together/moonshotai/Kimi-K2-Instruct-0905`                  | Kimi K2-Instruct 0905                  | texto       | 262,144    | Modelo secundario de texto Kimi  |

El ajuste preestablecido de onboarding configura `together/moonshotai/Kimi-K2.5` como modelo predeterminado.
