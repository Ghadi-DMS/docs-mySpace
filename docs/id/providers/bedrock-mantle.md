---
read_when:
    - Anda ingin menggunakan model OSS yang dihosting Bedrock Mantle dengan OpenClaw
    - Anda memerlukan endpoint Mantle yang kompatibel OpenAI untuk GPT-OSS, Qwen, Kimi, atau GLM
summary: Gunakan model Amazon Bedrock Mantle (kompatibel OpenAI) dengan OpenClaw
title: Amazon Bedrock Mantle
x-i18n:
    generated_at: "2026-04-06T03:09:57Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4e5b33ede4067fb7de02a046f3e375cbd2af4bf68e7751c8dd687447f1a78c86
    source_path: providers/bedrock-mantle.md
    workflow: 15
---

# Amazon Bedrock Mantle

OpenClaw menyertakan provider **Amazon Bedrock Mantle** bawaan yang terhubung ke
endpoint Mantle yang kompatibel OpenAI. Mantle menghosting model open-source dan
pihak ketiga (GPT-OSS, Qwen, Kimi, GLM, dan sejenisnya) melalui permukaan standar
`/v1/chat/completions` yang didukung oleh infrastruktur Bedrock.

## Yang didukung OpenClaw

- Provider: `amazon-bedrock-mantle`
- API: `openai-completions` (kompatibel OpenAI)
- Auth: `AWS_BEARER_TOKEN_BEDROCK` eksplisit atau pembuatan bearer token rantai kredensial IAM
- Region: `AWS_REGION` atau `AWS_DEFAULT_REGION` (default: `us-east-1`)

## Penemuan model otomatis

Saat `AWS_BEARER_TOKEN_BEDROCK` disetel, OpenClaw menggunakannya secara langsung. Jika tidak,
OpenClaw mencoba membuat bearer token Mantle dari rantai kredensial default AWS,
termasuk profil shared credentials/config, SSO, identitas web, serta peran instance
atau task. Kemudian OpenClaw menemukan model Mantle yang tersedia dengan mengueri endpoint
`/v1/models` region tersebut. Hasil penemuan di-cache selama 1 jam, dan bearer token
turunan IAM diperbarui setiap jam.

Region yang didukung: `us-east-1`, `us-east-2`, `us-west-2`, `ap-northeast-1`,
`ap-south-1`, `ap-southeast-3`, `eu-central-1`, `eu-west-1`, `eu-west-2`,
`eu-south-1`, `eu-north-1`, `sa-east-1`.

## Onboarding

1. Pilih satu jalur auth di **host gateway**:

Bearer token eksplisit:

```bash
export AWS_BEARER_TOKEN_BEDROCK="..."
# Opsional (default ke us-east-1):
export AWS_REGION="us-west-2"
```

Kredensial IAM:

```bash
# Sumber auth yang kompatibel dengan AWS SDK apa pun dapat digunakan di sini, misalnya:
export AWS_PROFILE="default"
export AWS_REGION="us-west-2"
```

2. Verifikasi bahwa model ditemukan:

```bash
openclaw models list
```

Model yang ditemukan akan muncul di bawah provider `amazon-bedrock-mantle`. Tidak
diperlukan config tambahan kecuali Anda ingin mengoverride default.

## Konfigurasi manual

Jika Anda lebih memilih config eksplisit alih-alih penemuan otomatis:

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

## Catatan

- OpenClaw dapat membuat bearer token Mantle untuk Anda dari kredensial IAM yang
  kompatibel dengan AWS SDK saat `AWS_BEARER_TOKEN_BEDROCK` tidak disetel.
- Bearer token tersebut sama dengan `AWS_BEARER_TOKEN_BEDROCK` yang digunakan oleh provider
  [Amazon Bedrock](/id/providers/bedrock) standar.
- Dukungan reasoning disimpulkan dari ID model yang mengandung pola seperti
  `thinking`, `reasoner`, atau `gpt-oss-120b`.
- Jika endpoint Mantle tidak tersedia atau tidak mengembalikan model apa pun, provider ini
  dilewati secara diam-diam.
