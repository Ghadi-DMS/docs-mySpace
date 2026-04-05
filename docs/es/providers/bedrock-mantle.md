---
read_when:
    - Quieres usar modelos OSS alojados en Bedrock Mantle con OpenClaw
    - Necesitas el endpoint compatible con OpenAI de Mantle para GPT-OSS, Qwen, Kimi o GLM
summary: Usar modelos Amazon Bedrock Mantle (compatibles con OpenAI) con OpenClaw
title: Amazon Bedrock Mantle
x-i18n:
    generated_at: "2026-04-05T12:50:47Z"
    model: gpt-5.4
    provider: openai
    source_hash: 2efe61261fbb430f63be9f5025c0654c44b191dbe96b3eb081d7ccbe78458907
    source_path: providers/bedrock-mantle.md
    workflow: 15
---

# Amazon Bedrock Mantle

OpenClaw incluye un proveedor integrado de **Amazon Bedrock Mantle** que se conecta al
endpoint compatible con OpenAI de Mantle. Mantle aloja modelos de código abierto y de
terceros (GPT-OSS, Qwen, Kimi, GLM y similares) mediante una superficie estándar
`/v1/chat/completions` respaldada por infraestructura Bedrock.

## Qué admite OpenClaw

- Proveedor: `amazon-bedrock-mantle`
- API: `openai-completions` (compatible con OpenAI)
- Autenticación: token bearer mediante `AWS_BEARER_TOKEN_BEDROCK`
- Región: `AWS_REGION` o `AWS_DEFAULT_REGION` (predeterminado: `us-east-1`)

## Descubrimiento automático de modelos

Cuando `AWS_BEARER_TOKEN_BEDROCK` está definido, OpenClaw descubre automáticamente
los modelos Mantle disponibles consultando el endpoint regional `/v1/models`.
Los resultados del descubrimiento se almacenan en caché durante 1 hora.

Regiones compatibles: `us-east-1`, `us-east-2`, `us-west-2`, `ap-northeast-1`,
`ap-south-1`, `ap-southeast-3`, `eu-central-1`, `eu-west-1`, `eu-west-2`,
`eu-south-1`, `eu-north-1`, `sa-east-1`.

## Onboarding

1. Establece el token bearer en el **host del gateway**:

```bash
export AWS_BEARER_TOKEN_BEDROCK="..."
# Optional (defaults to us-east-1):
export AWS_REGION="us-west-2"
```

2. Verifica que se descubren los modelos:

```bash
openclaw models list
```

Los modelos descubiertos aparecen bajo el proveedor `amazon-bedrock-mantle`. No
se requiere configuración adicional a menos que quieras anular los valores predeterminados.

## Configuración manual

Si prefieres una configuración explícita en lugar del descubrimiento automático:

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

## Notas

- Mantle requiere hoy un token bearer. Las credenciales IAM simples (roles de instancia,
  SSO, claves de acceso) no son suficientes sin un token.
- El token bearer es el mismo `AWS_BEARER_TOKEN_BEDROCK` usado por el proveedor estándar
  [Amazon Bedrock](/providers/bedrock).
- La compatibilidad con razonamiento se infiere a partir de ids de modelo que contienen patrones como
  `thinking`, `reasoner` o `gpt-oss-120b`.
- Si el endpoint de Mantle no está disponible o no devuelve modelos, el proveedor
  se omite silenciosamente.
