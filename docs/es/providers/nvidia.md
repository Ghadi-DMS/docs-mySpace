---
read_when:
    - Quieres usar modelos abiertos en OpenClaw gratis
    - Necesitas configurar NVIDIA_API_KEY
summary: Usa la API compatible con OpenAI de NVIDIA en OpenClaw
title: NVIDIA
x-i18n:
    generated_at: "2026-04-08T02:17:34Z"
    model: gpt-5.4
    provider: openai
    source_hash: b00f8cedaf223a33ba9f6a6dd8cf066d88cebeea52d391b871e435026182228a
    source_path: providers/nvidia.md
    workflow: 15
---

# NVIDIA

NVIDIA ofrece una API compatible con OpenAI en `https://integrate.api.nvidia.com/v1` para modelos abiertos de forma gratuita. Autentícate con una API key de [build.nvidia.com](https://build.nvidia.com/settings/api-keys).

## Configuración de CLI

Exporta la clave una vez, luego ejecuta la incorporación y configura un modelo de NVIDIA:

```bash
export NVIDIA_API_KEY="nvapi-..."
openclaw onboard --auth-choice skip
openclaw models set nvidia/nvidia/nemotron-3-super-120b-a12b
```

Si todavía pasas `--token`, recuerda que quedará en el historial del shell y en la salida de `ps`; prefiere la variable de entorno cuando sea posible.

## Fragmento de configuración

```json5
{
  env: { NVIDIA_API_KEY: "nvapi-..." },
  models: {
    providers: {
      nvidia: {
        baseUrl: "https://integrate.api.nvidia.com/v1",
        api: "openai-completions",
      },
    },
  },
  agents: {
    defaults: {
      model: { primary: "nvidia/nvidia/nemotron-3-super-120b-a12b" },
    },
  },
}
```

## ID de modelos

| Model ref                                  | Name                         | Context | Max output |
| ------------------------------------------ | ---------------------------- | ------- | ---------- |
| `nvidia/nvidia/nemotron-3-super-120b-a12b` | NVIDIA Nemotron 3 Super 120B | 262,144 | 8,192      |
| `nvidia/moonshotai/kimi-k2.5`              | Kimi K2.5                    | 262,144 | 8,192      |
| `nvidia/minimaxai/minimax-m2.5`            | Minimax M2.5                 | 196,608 | 8,192      |
| `nvidia/z-ai/glm5`                         | GLM 5                        | 202,752 | 8,192      |

## Notas

- Endpoint `/v1` compatible con OpenAI; usa una API key de [build.nvidia.com](https://build.nvidia.com/).
- El proveedor se habilita automáticamente cuando `NVIDIA_API_KEY` está configurada.
- El catálogo empaquetado es estático; los costos usan `0` de forma predeterminada en el código fuente.
