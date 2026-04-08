---
read_when:
    - Sie möchten kostenlose offene Modelle in OpenClaw verwenden
    - Sie benötigen die Einrichtung von NVIDIA_API_KEY
summary: Die OpenAI-kompatible API von NVIDIA in OpenClaw verwenden
title: NVIDIA
x-i18n:
    generated_at: "2026-04-08T02:17:56Z"
    model: gpt-5.4
    provider: openai
    source_hash: b00f8cedaf223a33ba9f6a6dd8cf066d88cebeea52d391b871e435026182228a
    source_path: providers/nvidia.md
    workflow: 15
---

# NVIDIA

NVIDIA bietet unter `https://integrate.api.nvidia.com/v1` eine OpenAI-kompatible API für offene Modelle kostenlos an. Authentifizieren Sie sich mit einem API-Schlüssel von [build.nvidia.com](https://build.nvidia.com/settings/api-keys).

## CLI-Einrichtung

Exportieren Sie den Schlüssel einmal und führen Sie dann das Onboarding aus und setzen Sie ein NVIDIA-Modell:

```bash
export NVIDIA_API_KEY="nvapi-..."
openclaw onboard --auth-choice skip
openclaw models set nvidia/nvidia/nemotron-3-super-120b-a12b
```

Wenn Sie weiterhin `--token` übergeben, denken Sie daran, dass es im Shell-Verlauf und in der `ps`-Ausgabe landet; verwenden Sie nach Möglichkeit bevorzugt die Umgebungsvariable.

## Konfigurations-Snippet

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

## Modell-IDs

| Model ref                                  | Name                         | Kontext | Maximale Ausgabe |
| ------------------------------------------ | ---------------------------- | ------- | ---------------- |
| `nvidia/nvidia/nemotron-3-super-120b-a12b` | NVIDIA Nemotron 3 Super 120B | 262,144 | 8,192            |
| `nvidia/moonshotai/kimi-k2.5`              | Kimi K2.5                    | 262,144 | 8,192            |
| `nvidia/minimaxai/minimax-m2.5`            | Minimax M2.5                 | 196,608 | 8,192            |
| `nvidia/z-ai/glm5`                         | GLM 5                        | 202,752 | 8,192            |

## Hinweise

- OpenAI-kompatibler `/v1`-Endpunkt; verwenden Sie einen API-Schlüssel von [build.nvidia.com](https://build.nvidia.com/).
- Der Anbieter wird automatisch aktiviert, wenn `NVIDIA_API_KEY` gesetzt ist.
- Der gebündelte Katalog ist statisch; Kosten sind im Quellcode standardmäßig auf `0` gesetzt.
