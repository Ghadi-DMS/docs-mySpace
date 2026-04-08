---
read_when:
    - Você quer usar Z.AI / modelos GLM no OpenClaw
    - Você precisa de uma configuração simples com ZAI_API_KEY
summary: Use o Z.AI (modelos GLM) com o OpenClaw
title: Z.AI
x-i18n:
    generated_at: "2026-04-08T05:26:49Z"
    model: gpt-5.4
    provider: openai
    source_hash: 66cbd9813ee28d202dcae34debab1b0cf9927793acb00743c1c62b48d9e381f9
    source_path: providers/zai.md
    workflow: 15
---

# Z.AI

Z.AI é a plataforma de API para modelos **GLM**. Ela fornece APIs REST para GLM e usa chaves de API
para autenticação. Crie sua chave de API no console do Z.AI. O OpenClaw usa o provedor `zai`
com uma chave de API do Z.AI.

## Configuração da CLI

```bash
# Configuração genérica com chave de API e detecção automática de endpoint
openclaw onboard --auth-choice zai-api-key

# Coding Plan Global, recomendado para usuários do Coding Plan
openclaw onboard --auth-choice zai-coding-global

# Coding Plan CN (região da China), recomendado para usuários do Coding Plan
openclaw onboard --auth-choice zai-coding-cn

# API geral
openclaw onboard --auth-choice zai-global

# API geral CN (região da China)
openclaw onboard --auth-choice zai-cn
```

## Trecho de configuração

```json5
{
  env: { ZAI_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "zai/glm-5.1" } } },
}
```

`zai-api-key` permite que o OpenClaw detecte o endpoint correspondente do Z.AI a partir da chave
e aplique automaticamente a URL base correta. Use as opções regionais explícitas quando
quiser forçar uma superfície específica do Coding Plan ou da API geral.

## Catálogo GLM integrado

Atualmente, o OpenClaw inicializa o provedor integrado `zai` com:

- `glm-5.1`
- `glm-5`
- `glm-5-turbo`
- `glm-5v-turbo`
- `glm-4.7`
- `glm-4.7-flash`
- `glm-4.7-flashx`
- `glm-4.6`
- `glm-4.6v`
- `glm-4.5`
- `glm-4.5-air`
- `glm-4.5-flash`
- `glm-4.5v`

## Observações

- Os modelos GLM estão disponíveis como `zai/<model>` (exemplo: `zai/glm-5`).
- Referência de modelo integrado padrão: `zai/glm-5.1`
- IDs `glm-5*` desconhecidos ainda são resolvidos por encaminhamento no caminho do provedor integrado
  por meio da síntese de metadados pertencentes ao provedor a partir do modelo `glm-4.7` quando o id
  corresponde ao formato atual da família GLM-5.
- `tool_stream` vem ativado por padrão para streaming de chamadas de ferramenta do Z.AI. Defina
  `agents.defaults.models["zai/<model>"].params.tool_stream` como `false` para desativá-lo.
- Veja [/providers/glm](/pt-BR/providers/glm) para a visão geral da família de modelos.
- O Z.AI usa autenticação Bearer com sua chave de API.
