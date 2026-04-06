---
read_when:
    - Vuoi usare Qwen con OpenClaw
    - In precedenza usavi Qwen OAuth
summary: Usa Qwen Cloud tramite il provider qwen integrato di OpenClaw
title: Qwen
x-i18n:
    generated_at: "2026-04-06T03:11:28Z"
    model: gpt-5.4
    provider: openai
    source_hash: f175793693ab6a4c3f1f4d42040e673c15faf7603a500757423e9e06977c989d
    source_path: providers/qwen.md
    workflow: 15
---

# Qwen

<Warning>

**Qwen OAuth è stato rimosso.** L'integrazione OAuth del piano gratuito
(`qwen-portal`) che usava gli endpoint `portal.qwen.ai` non è più disponibile.
Vedi [Issue #49557](https://github.com/openclaw/openclaw/issues/49557) per il
contesto.

</Warning>

## Consigliato: Qwen Cloud

OpenClaw ora tratta Qwen come provider integrato di prima classe con id canonico
`qwen`. Il provider integrato punta agli endpoint Qwen Cloud / Alibaba DashScope e
Coding Plan e mantiene gli id legacy `modelstudio` funzionanti come alias di
compatibilità.

- Provider: `qwen`
- Variabile env preferita: `QWEN_API_KEY`
- Accettate anche per compatibilità: `MODELSTUDIO_API_KEY`, `DASHSCOPE_API_KEY`
- Stile API: compatibile con OpenAI

Se vuoi `qwen3.6-plus`, preferisci l'endpoint **Standard (pay-as-you-go)**.
Il supporto Coding Plan può essere in ritardo rispetto al catalogo pubblico.

```bash
# Endpoint globale Coding Plan
openclaw onboard --auth-choice qwen-api-key

# Endpoint Cina Coding Plan
openclaw onboard --auth-choice qwen-api-key-cn

# Endpoint globale Standard (pay-as-you-go)
openclaw onboard --auth-choice qwen-standard-api-key

# Endpoint Cina Standard (pay-as-you-go)
openclaw onboard --auth-choice qwen-standard-api-key-cn
```

Gli id auth-choice legacy `modelstudio-*` e i model ref `modelstudio/...` continuano
a funzionare come alias di compatibilità, ma i nuovi flussi di configurazione dovrebbero preferire gli id
auth-choice canonici `qwen-*` e i model ref `qwen/...`.

Dopo l'onboarding, imposta un modello predefinito:

```json5
{
  agents: {
    defaults: {
      model: { primary: "qwen/qwen3.5-plus" },
    },
  },
}
```

## Tipi di piano ed endpoint

| Piano                      | Regione | Auth choice                | Endpoint                                         |
| -------------------------- | ------- | -------------------------- | ------------------------------------------------ |
| Standard (pay-as-you-go)   | Cina    | `qwen-standard-api-key-cn` | `dashscope.aliyuncs.com/compatible-mode/v1`      |
| Standard (pay-as-you-go)   | Globale | `qwen-standard-api-key`    | `dashscope-intl.aliyuncs.com/compatible-mode/v1` |
| Coding Plan (abbonamento)  | Cina    | `qwen-api-key-cn`          | `coding.dashscope.aliyuncs.com/v1`               |
| Coding Plan (abbonamento)  | Globale | `qwen-api-key`             | `coding-intl.dashscope.aliyuncs.com/v1`          |

Il provider seleziona automaticamente l'endpoint in base al tuo auth choice. Le scelte
canoniche usano la famiglia `qwen-*`; `modelstudio-*` resta solo per compatibilità.
Puoi sovrascrivere con un `baseUrl` personalizzato nella configurazione.

Gli endpoint nativi Model Studio pubblicizzano la compatibilità con l'utilizzo in streaming sul
trasporto condiviso `openai-completions`. OpenClaw ora la ricava dalle capability
dell'endpoint, così gli id provider personalizzati compatibili con DashScope che puntano agli
stessi host nativi ereditano lo stesso comportamento di streaming-usage invece di
richiedere specificamente l'id provider integrato `qwen`.

## Ottieni la tua API key

- **Gestisci le chiavi**: [home.qwencloud.com/api-keys](https://home.qwencloud.com/api-keys)
- **Documentazione**: [docs.qwencloud.com](https://docs.qwencloud.com/developer-guides/getting-started/introduction)

## Catalogo integrato

OpenClaw attualmente include questo catalogo Qwen integrato:

| Model ref                   | Input       | Contesto  | Note                                               |
| --------------------------- | ----------- | --------- | -------------------------------------------------- |
| `qwen/qwen3.5-plus`         | text, image | 1,000,000 | Modello predefinito                                |
| `qwen/qwen3.6-plus`         | text, image | 1,000,000 | Preferisci gli endpoint Standard quando ti serve questo modello |
| `qwen/qwen3-max-2026-01-23` | text        | 262,144   | Linea Qwen Max                                     |
| `qwen/qwen3-coder-next`     | text        | 262,144   | Coding                                             |
| `qwen/qwen3-coder-plus`     | text        | 1,000,000 | Coding                                             |
| `qwen/MiniMax-M2.5`         | text        | 1,000,000 | Reasoning abilitato                                |
| `qwen/glm-5`                | text        | 202,752   | GLM                                                |
| `qwen/glm-4.7`              | text        | 202,752   | GLM                                                |
| `qwen/kimi-k2.5`            | text, image | 262,144   | Moonshot AI tramite Alibaba                        |

La disponibilità può comunque variare in base all'endpoint e al piano di fatturazione anche quando un modello è
presente nel catalogo integrato.

La compatibilità native-streaming usage si applica sia agli host Coding Plan sia
agli host Standard compatibili con DashScope:

- `https://coding.dashscope.aliyuncs.com/v1`
- `https://coding-intl.dashscope.aliyuncs.com/v1`
- `https://dashscope.aliyuncs.com/compatible-mode/v1`
- `https://dashscope-intl.aliyuncs.com/compatible-mode/v1`

## Disponibilità di Qwen 3.6 Plus

`qwen3.6-plus` è disponibile sugli endpoint Model Studio Standard (pay-as-you-go):

- Cina: `dashscope.aliyuncs.com/compatible-mode/v1`
- Globale: `dashscope-intl.aliyuncs.com/compatible-mode/v1`

Se gli endpoint Coding Plan restituiscono un errore "unsupported model" per
`qwen3.6-plus`, passa a Standard (pay-as-you-go) invece della coppia
endpoint/chiave Coding Plan.

## Piano delle capability

L'estensione `qwen` viene posizionata come la sede vendor per l'intera superficie
Qwen Cloud, non solo per i modelli coding/text.

- Modelli text/chat: integrati ora
- Tool calling, output strutturato, thinking: ereditati dal trasporto compatibile con OpenAI
- Image generation: pianificata a livello di provider plugin
- Comprensione di immagini/video: integrata ora sull'endpoint Standard
- Speech/audio: pianificata a livello di provider plugin
- Embedding/reranking per memory: pianificati tramite la superficie embedding adapter
- Video generation: integrata ora tramite la capability condivisa di video-generation

## Componenti aggiuntivi multimodali

L'estensione `qwen` ora espone anche:

- Comprensione video tramite `qwen-vl-max-latest`
- Generazione video Wan tramite:
  - `wan2.6-t2v` (predefinito)
  - `wan2.6-i2v`
  - `wan2.6-r2v`
  - `wan2.6-r2v-flash`
  - `wan2.7-r2v`

Queste superfici multimodali usano gli endpoint DashScope **Standard**, non gli
endpoint Coding Plan.

- Base URL Standard globale/intl: `https://dashscope-intl.aliyuncs.com/compatible-mode/v1`
- Base URL Standard Cina: `https://dashscope.aliyuncs.com/compatible-mode/v1`

Per la generazione video, OpenClaw mappa la regione Qwen configurata all'host
video DashScope AIGC corrispondente prima di inviare il job:

- Globale/Intl: `https://dashscope-intl.aliyuncs.com`
- Cina: `https://dashscope.aliyuncs.com`

Questo significa che un normale `models.providers.qwen.baseUrl` che punta a uno
degli host Qwen Coding Plan o Standard continua comunque a mantenere la generazione video sul corretto
endpoint video DashScope regionale.

Per la generazione video, imposta esplicitamente un modello predefinito:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: { primary: "qwen/wan2.6-t2v" },
    },
  },
}
```

Limiti attuali della generazione video Qwen integrata:

- Fino a **1** video in uscita per richiesta
- Fino a **1** immagine di input
- Fino a **4** video di input
- Fino a **10 secondi** di durata
- Supporta `size`, `aspectRatio`, `resolution`, `audio` e `watermark`
- La modalità immagine/video di riferimento attualmente richiede **URL http(s) remoti**. I
  percorsi di file locali vengono rifiutati in anticipo perché l'endpoint video DashScope non
  accetta buffer locali caricati per questi riferimenti.

Vedi [Video Generation](/tools/video-generation) per i parametri condivisi dello
strumento, la selezione del provider e il comportamento di failover.

## Nota sull'ambiente

Se il Gateway viene eseguito come demone (launchd/systemd), assicurati che `QWEN_API_KEY` sia
disponibile per quel processo (ad esempio in `~/.openclaw/.env` o tramite
`env.shellEnv`).
