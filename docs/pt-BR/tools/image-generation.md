---
read_when:
    - Ao gerar imagens por meio do agente
    - Ao configurar provedores e modelos de geração de imagens
    - Ao entender os parâmetros da ferramenta `image_generate`
summary: Gere e edite imagens usando provedores configurados (OpenAI, Google Gemini, fal, MiniMax, ComfyUI, Vydra)
title: Geração de Imagens
x-i18n:
    generated_at: "2026-04-06T05:34:58Z"
    model: gpt-5.4
    provider: openai
    source_hash: 903cc522c283a8da2cbd449ae3e25f349a74d00ecfdaf0f323fd8aa3f2107aea
    source_path: tools/image-generation.md
    workflow: 15
---

# Geração de Imagens

A ferramenta `image_generate` permite que o agente crie e edite imagens usando seus provedores configurados. As imagens geradas são entregues automaticamente como anexos de mídia na resposta do agente.

<Note>
A ferramenta só aparece quando pelo menos um provedor de geração de imagens está disponível. Se você não vê `image_generate` nas ferramentas do seu agente, configure `agents.defaults.imageGenerationModel` ou defina uma chave de API de provedor.
</Note>

## Início rápido

1. Defina uma chave de API para pelo menos um provedor (por exemplo, `OPENAI_API_KEY` ou `GEMINI_API_KEY`).
2. Opcionalmente, defina seu modelo preferido:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "openai/gpt-image-1",
      },
    },
  },
}
```

3. Peça ao agente: _"Gere uma imagem de um mascote lagosta amigável."_

O agente chama `image_generate` automaticamente. Não é necessário permitir a ferramenta em uma lista — ela é ativada por padrão quando um provedor está disponível.

## Provedores compatíveis

| Provedor | Modelo padrão                    | Suporte a edição                   | Chave de API                                           |
| -------- | -------------------------------- | ---------------------------------- | ------------------------------------------------------ |
| OpenAI   | `gpt-image-1`                    | Sim (até 5 imagens)                | `OPENAI_API_KEY`                                       |
| Google   | `gemini-3.1-flash-image-preview` | Sim                                | `GEMINI_API_KEY` ou `GOOGLE_API_KEY`                   |
| fal      | `fal-ai/flux/dev`                | Sim                                | `FAL_KEY`                                              |
| MiniMax  | `image-01`                       | Sim (referência de sujeito)        | `MINIMAX_API_KEY` ou OAuth do MiniMax (`minimax-portal`) |
| ComfyUI  | `workflow`                       | Sim (1 imagem, configurada no workflow) | `COMFY_API_KEY` ou `COMFY_CLOUD_API_KEY` para nuvem    |
| Vydra    | `grok-imagine`                   | Não                                | `VYDRA_API_KEY`                                        |

Use `action: "list"` para inspecionar os provedores e modelos disponíveis em tempo de execução:

```
/tool image_generate action=list
```

## Parâmetros da ferramenta

| Parâmetro    | Tipo     | Descrição                                                                            |
| ------------ | -------- | ------------------------------------------------------------------------------------ |
| `prompt`     | string   | Prompt de geração de imagem (obrigatório para `action: "generate"`)                  |
| `action`     | string   | `"generate"` (padrão) ou `"list"` para inspecionar provedores                        |
| `model`      | string   | Sobrescrita de provedor/modelo, por exemplo `openai/gpt-image-1`                     |
| `image`      | string   | Caminho ou URL de uma única imagem de referência para o modo de edição               |
| `images`     | string[] | Múltiplas imagens de referência para o modo de edição (até 5)                        |
| `size`       | string   | Dica de tamanho: `1024x1024`, `1536x1024`, `1024x1536`, `1024x1792`, `1792x1024`     |
| `aspectRatio`| string   | Proporção da imagem: `1:1`, `2:3`, `3:2`, `3:4`, `4:3`, `4:5`, `5:4`, `9:16`, `16:9`, `21:9` |
| `resolution` | string   | Dica de resolução: `1K`, `2K` ou `4K`                                                |
| `count`      | number   | Número de imagens a gerar (1–4)                                                      |
| `filename`   | string   | Dica de nome de arquivo de saída                                                     |

Nem todos os provedores oferecem suporte a todos os parâmetros. A ferramenta encaminha o que cada provedor suporta, ignora o restante e informa as sobrescritas descartadas no resultado da ferramenta.

## Configuração

### Seleção de modelo

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "openai/gpt-image-1",
        fallbacks: ["google/gemini-3.1-flash-image-preview", "fal/fal-ai/flux/dev"],
      },
    },
  },
}
```

### Ordem de seleção de provedores

Ao gerar uma imagem, o OpenClaw tenta os provedores nesta ordem:

1. Parâmetro **`model`** da chamada da ferramenta (se o agente especificar um)
2. **`imageGenerationModel.primary`** da configuração
3. **`imageGenerationModel.fallbacks`** em ordem
4. **Detecção automática** — usa apenas padrões de provedor com autenticação disponível:
   - provedor padrão atual primeiro
   - provedores de geração de imagem registrados restantes em ordem de ID do provedor

Se um provedor falhar (erro de autenticação, limite de taxa etc.), o próximo candidato será tentado automaticamente. Se todos falharem, o erro incluirá detalhes de cada tentativa.

Observações:

- A detecção automática considera a autenticação. Um padrão de provedor só entra na lista de candidatos
  quando o OpenClaw realmente consegue autenticar esse provedor.
- Use `action: "list"` para inspecionar os provedores atualmente registrados, seus
  modelos padrão e dicas de variáveis de ambiente para autenticação.

### Edição de imagem

OpenAI, Google, fal, MiniMax e ComfyUI oferecem suporte à edição de imagens de referência. Passe um caminho ou URL de imagem de referência:

```
"Gerar uma versão em aquarela desta foto" + image: "/path/to/photo.jpg"
```

OpenAI e Google oferecem suporte a até 5 imagens de referência por meio do parâmetro `images`. fal, MiniMax e ComfyUI oferecem suporte a 1.

A geração de imagens do MiniMax está disponível por ambos os caminhos de autenticação empacotados do MiniMax:

- `minimax/image-01` para configurações com chave de API
- `minimax-portal/image-01` para configurações com OAuth

## Capacidades do provedor

| Capacidade            | OpenAI               | Google               | fal                 | MiniMax                    | ComfyUI                            | Vydra   |
| --------------------- | -------------------- | -------------------- | ------------------- | -------------------------- | ---------------------------------- | ------- |
| Gerar                 | Sim (até 4)          | Sim (até 4)          | Sim (até 4)         | Sim (até 9)                | Sim (saídas definidas pelo workflow) | Sim (1) |
| Edição/referência     | Sim (até 5 imagens)  | Sim (até 5 imagens)  | Sim (1 imagem)      | Sim (1 imagem, ref. de sujeito) | Sim (1 imagem, configurada no workflow) | Não     |
| Controle de tamanho   | Sim                  | Sim                  | Sim                 | Não                        | Não                                 | Não      |
| Proporção da imagem   | Não                  | Sim                  | Sim (apenas geração) | Sim                       | Não                                 | Não      |
| Resolução (1K/2K/4K)  | Não                  | Sim                  | Sim                 | Não                        | Não                                 | Não      |

## Relacionado

- [Visão geral das ferramentas](/pt-BR/tools) — todas as ferramentas disponíveis do agente
- [fal](/pt-BR/providers/fal) — configuração do provedor de imagem e vídeo fal
- [ComfyUI](/pt-BR/providers/comfy) — configuração de workflow do ComfyUI local e do Comfy Cloud
- [Google (Gemini)](/pt-BR/providers/google) — configuração do provedor de imagens Gemini
- [MiniMax](/pt-BR/providers/minimax) — configuração do provedor de imagens MiniMax
- [OpenAI](/pt-BR/providers/openai) — configuração do provedor OpenAI Images
- [Vydra](/pt-BR/providers/vydra) — configuração de imagem, vídeo e fala do Vydra
- [Referência de configuração](/pt-BR/gateway/configuration-reference#agent-defaults) — configuração `imageGenerationModel`
- [Modelos](/pt-BR/concepts/models) — configuração de modelos e failover
