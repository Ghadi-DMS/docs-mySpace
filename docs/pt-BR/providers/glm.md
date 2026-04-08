---
read_when:
    - Você quer usar modelos GLM no OpenClaw
    - Você precisa da convenção de nomenclatura e da configuração dos modelos
summary: Visão geral da família de modelos GLM + como usá-la no OpenClaw
title: Modelos GLM
x-i18n:
    generated_at: "2026-04-08T05:26:43Z"
    model: gpt-5.4
    provider: openai
    source_hash: 79a55acfa139847b4b85dbc09f1068cbd2febb1e49f984a23ea9e3b43bc910eb
    source_path: providers/glm.md
    workflow: 15
---

# Modelos GLM

GLM é uma **família de modelos** (não uma empresa) disponível pela plataforma Z.AI. No OpenClaw, os
modelos GLM são acessados pelo provedor `zai` e por IDs de modelo como `zai/glm-5`.

## Configuração da CLI

```bash
# Configuração genérica com chave de API e detecção automática do endpoint
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

`zai-api-key` permite que o OpenClaw detecte o endpoint Z.AI correspondente a partir da chave e
aplique automaticamente a URL base correta. Use as opções regionais explícitas quando
quiser forçar um Coding Plan específico ou uma superfície específica da API geral.

## Modelos GLM atualmente incluídos no pacote

No momento, o OpenClaw predefine estes refs GLM no provedor `zai` incluído no pacote:

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

- As versões e a disponibilidade do GLM podem mudar; consulte a documentação da Z.AI para ver as informações mais recentes.
- O ref de modelo padrão incluído no pacote é `zai/glm-5.1`.
- Para detalhes do provedor, consulte [/providers/zai](/pt-BR/providers/zai).
