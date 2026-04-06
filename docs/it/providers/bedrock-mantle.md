---
read_when:
    - Vuoi usare i modelli OSS ospitati da Bedrock Mantle con OpenClaw
    - Hai bisogno dell'endpoint Mantle compatibile con OpenAI per GPT-OSS, Qwen, Kimi o GLM
summary: Usa i modelli Mantle di Amazon Bedrock (compatibili con OpenAI) con OpenClaw
title: Amazon Bedrock Mantle
x-i18n:
    generated_at: "2026-04-06T03:10:20Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4e5b33ede4067fb7de02a046f3e375cbd2af4bf68e7751c8dd687447f1a78c86
    source_path: providers/bedrock-mantle.md
    workflow: 15
---

# Amazon Bedrock Mantle

OpenClaw include un provider bundled **Amazon Bedrock Mantle** che si collega
all'endpoint Mantle compatibile con OpenAI. Mantle ospita modelli open source e
di terze parti (GPT-OSS, Qwen, Kimi, GLM e simili) tramite una superficie standard
`/v1/chat/completions` supportata dall'infrastruttura Bedrock.

## Cosa supporta OpenClaw

- Provider: `amazon-bedrock-mantle`
- API: `openai-completions` (compatibile con OpenAI)
- Autenticazione: `AWS_BEARER_TOKEN_BEDROCK` esplicito o generazione del bearer token tramite la catena di credenziali IAM
- Regione: `AWS_REGION` o `AWS_DEFAULT_REGION` (predefinita: `us-east-1`)

## Rilevamento automatico dei modelli

Quando `AWS_BEARER_TOKEN_BEDROCK` è impostato, OpenClaw lo usa direttamente. In caso contrario,
OpenClaw prova a generare un bearer token Mantle dalla catena di credenziali predefinita
AWS, inclusi profili condivisi di credenziali/configurazione, SSO, web
identity e ruoli di istanza o task. Quindi rileva i modelli Mantle
disponibili interrogando l'endpoint `/v1/models` della regione. I risultati del rilevamento vengono
memorizzati nella cache per 1 ora e i bearer token derivati da IAM vengono aggiornati ogni ora.

Regioni supportate: `us-east-1`, `us-east-2`, `us-west-2`, `ap-northeast-1`,
`ap-south-1`, `ap-southeast-3`, `eu-central-1`, `eu-west-1`, `eu-west-2`,
`eu-south-1`, `eu-north-1`, `sa-east-1`.

## Onboarding

1. Scegli un percorso di autenticazione sull'**host gateway**:

Bearer token esplicito:

```bash
export AWS_BEARER_TOKEN_BEDROCK="..."
# Facoltativo (predefinito: us-east-1):
export AWS_REGION="us-west-2"
```

Credenziali IAM:

```bash
# Qui funziona qualsiasi sorgente di autenticazione compatibile con AWS SDK, ad esempio:
export AWS_PROFILE="default"
export AWS_REGION="us-west-2"
```

2. Verifica che i modelli vengano rilevati:

```bash
openclaw models list
```

I modelli rilevati appaiono sotto il provider `amazon-bedrock-mantle`. Non è
necessaria alcuna configurazione aggiuntiva, a meno che tu non voglia sovrascrivere i valori predefiniti.

## Configurazione manuale

Se preferisci una configurazione esplicita invece del rilevamento automatico:

```json5
{
  models: {
    providers: {
      "amazon-bedrock-mantle": {
        baseUrl: "https://bedrock-mantle.us-east-1.api.aws/v1",
        api: "openai-completions",
        auth: "api-key",
        apiKey: "env:AWS_BEARER_TOKEN_BEDROCK",
        models: [
          {
            id: "gpt-oss-120b",
            name: "GPT-OSS 120B",
            reasoning: true,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 32000,
            maxTokens: 4096,
          },
        ],
      },
    },
  },
}
```

## Note

- OpenClaw può generare per te il bearer token Mantle a partire da credenziali IAM
  compatibili con AWS SDK quando `AWS_BEARER_TOKEN_BEDROCK` non è impostato.
- Il bearer token è lo stesso `AWS_BEARER_TOKEN_BEDROCK` usato dal provider standard
  [Amazon Bedrock](/it/providers/bedrock).
- Il supporto al reasoning viene dedotto dagli ID modello che contengono pattern come
  `thinking`, `reasoner` o `gpt-oss-120b`.
- Se l'endpoint Mantle non è disponibile o non restituisce modelli, il provider viene
  saltato silenziosamente.
