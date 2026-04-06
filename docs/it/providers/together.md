---
read_when:
    - Vuoi usare Together AI con OpenClaw
    - Hai bisogno della variabile env della chiave API o della scelta di autenticazione CLI
summary: Configurazione di Together AI (autenticazione + selezione del modello)
title: Together AI
x-i18n:
    generated_at: "2026-04-06T03:11:20Z"
    model: gpt-5.4
    provider: openai
    source_hash: b68fdc15bfcac8d59e3e0c06a39162abd48d9d41a9a64a0ac622cd8e3f80a595
    source_path: providers/together.md
    workflow: 15
---

# Together AI

[Together AI](https://together.ai) fornisce accesso ai principali modelli open-source, tra cui Llama, DeepSeek, Kimi e altri, tramite un'API unificata.

- Provider: `together`
- Autenticazione: `TOGETHER_API_KEY`
- API: compatibile con OpenAI
- URL di base: `https://api.together.xyz/v1`

## Avvio rapido

1. Imposta la chiave API (consigliato: archiviarla per il Gateway):

```bash
openclaw onboard --auth-choice together-api-key
```

2. Imposta un modello predefinito:

```json5
{
  agents: {
    defaults: {
      model: { primary: "together/moonshotai/Kimi-K2.5" },
    },
  },
}
```

## Esempio non interattivo

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice together-api-key \
  --together-api-key "$TOGETHER_API_KEY"
```

Questo imposterà `together/moonshotai/Kimi-K2.5` come modello predefinito.

## Nota sull'ambiente

Se il Gateway viene eseguito come demone (launchd/systemd), assicurati che `TOGETHER_API_KEY`
sia disponibile per quel processo (ad esempio in `~/.openclaw/.env` o tramite
`env.shellEnv`).

## Catalogo integrato

OpenClaw attualmente include questo catalogo Together bundled:

| Riferimento modello                                         | Nome                                   | Input       | Contesto   | Note                             |
| ----------------------------------------------------------- | -------------------------------------- | ----------- | ---------- | -------------------------------- |
| `together/moonshotai/Kimi-K2.5`                             | Kimi K2.5                              | text, image | 262,144    | Modello predefinito; reasoning abilitato |
| `together/zai-org/GLM-4.7`                                  | GLM 4.7 Fp8                            | text        | 202,752    | Modello di testo generico        |
| `together/meta-llama/Llama-3.3-70B-Instruct-Turbo`          | Llama 3.3 70B Instruct Turbo           | text        | 131,072    | Modello istruzionale veloce      |
| `together/meta-llama/Llama-4-Scout-17B-16E-Instruct`        | Llama 4 Scout 17B 16E Instruct         | text, image | 10,000,000 | Multimodale                      |
| `together/meta-llama/Llama-4-Maverick-17B-128E-Instruct-FP8`| Llama 4 Maverick 17B 128E Instruct FP8 | text, image | 20,000,000 | Multimodale                      |
| `together/deepseek-ai/DeepSeek-V3.1`                        | DeepSeek V3.1                          | text        | 131,072    | Modello di testo generico        |
| `together/deepseek-ai/DeepSeek-R1`                          | DeepSeek R1                            | text        | 131,072    | Modello di reasoning             |
| `together/moonshotai/Kimi-K2-Instruct-0905`                 | Kimi K2-Instruct 0905                  | text        | 262,144    | Modello di testo Kimi secondario |

Il preset di onboarding imposta `together/moonshotai/Kimi-K2.5` come modello predefinito.

## Generazione video

Il plugin bundled `together` registra anche la generazione video tramite lo
strumento condiviso `video_generate`.

- Modello video predefinito: `together/Wan-AI/Wan2.2-T2V-A14B`
- Modalità: flussi text-to-video e con singola immagine di riferimento
- Supporta `aspectRatio` e `resolution`

Per usare Together come provider video predefinito:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "together/Wan-AI/Wan2.2-T2V-A14B",
      },
    },
  },
}
```

Consulta [Generazione video](/tools/video-generation) per i parametri dello
strumento condiviso, la selezione del provider e il comportamento di failover.
