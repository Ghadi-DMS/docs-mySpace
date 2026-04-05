---
read_when:
    - Quieres usar modelos de NVIDIA en OpenClaw
    - Necesitas configurar NVIDIA_API_KEY
summary: Usa la API compatible con OpenAI de NVIDIA en OpenClaw
title: NVIDIA
x-i18n:
    generated_at: "2026-04-05T12:51:31Z"
    model: gpt-5.4
    provider: openai
    source_hash: a24c5e46c0cf0fbc63bf09c772b486dd7f8f4b52e687d3b835bb54a1176b28da
    source_path: providers/nvidia.md
    workflow: 15
---

# NVIDIA

NVIDIA proporciona una API compatible con OpenAI en `https://integrate.api.nvidia.com/v1` para modelos Nemotron y NeMo. Autentícate con una clave de API de [NVIDIA NGC](https://catalog.ngc.nvidia.com/).

## Configuración de la CLI

Exporta la clave una vez, luego ejecuta el onboarding y establece un modelo de NVIDIA:

```bash
export NVIDIA_API_KEY="nvapi-..."
openclaw onboard --auth-choice skip
openclaw models set nvidia/nvidia/llama-3.1-nemotron-70b-instruct
```

Si sigues pasando `--token`, recuerda que termina en el historial del shell y en la salida de `ps`; prefiere la variable de entorno cuando sea posible.

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
      model: { primary: "nvidia/nvidia/llama-3.1-nemotron-70b-instruct" },
    },
  },
}
```

## IDs de modelo

| Referencia de modelo                                  | Nombre                                   | Contexto | Salida máxima |
| ----------------------------------------------------- | ---------------------------------------- | -------- | ------------- |
| `nvidia/nvidia/llama-3.1-nemotron-70b-instruct`      | NVIDIA Llama 3.1 Nemotron 70B Instruct   | 131,072  | 4,096         |
| `nvidia/meta/llama-3.3-70b-instruct`                 | Meta Llama 3.3 70B Instruct              | 131,072  | 4,096         |
| `nvidia/nvidia/mistral-nemo-minitron-8b-8k-instruct` | NVIDIA Mistral NeMo Minitron 8B Instruct | 8,192    | 2,048         |

## Notas

- Endpoint `/v1` compatible con OpenAI; usa una clave de API de NVIDIA NGC.
- El proveedor se habilita automáticamente cuando `NVIDIA_API_KEY` está configurada.
- El catálogo integrado es estático; los costos están predeterminados a `0` en el código fuente.
