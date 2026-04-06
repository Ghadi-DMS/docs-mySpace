---
read_when:
    - Gerando vídeos por meio do agente
    - Configurando provedores e modelos de geração de vídeo
    - Entendendo os parâmetros da ferramenta video_generate
summary: Gere vídeos a partir de texto, imagens ou vídeos existentes usando 12 backends de provedores
title: Geração de Vídeo
x-i18n:
    generated_at: "2026-04-06T05:34:56Z"
    model: gpt-5.4
    provider: openai
    source_hash: 90d8a392b35adbd899232b02c55c10895b9d7ffc9858d6ca448f2e4e4a57f12f
    source_path: tools/video-generation.md
    workflow: 15
---

# Geração de Vídeo

Os agentes do OpenClaw podem gerar vídeos a partir de prompts de texto, imagens de referência ou vídeos existentes. Há suporte para doze backends de provedores, cada um com diferentes opções de modelo, modos de entrada e conjuntos de recursos. O agente escolhe automaticamente o provedor certo com base na sua configuração e nas chaves de API disponíveis.

<Note>
A ferramenta `video_generate` só aparece quando pelo menos um provedor de geração de vídeo está disponível. Se você não a encontrar nas ferramentas do seu agente, defina uma chave de API de provedor ou configure `agents.defaults.videoGenerationModel`.
</Note>

## Início rápido

1. Defina uma chave de API para qualquer provedor compatível:

```bash
export GEMINI_API_KEY="your-key"
```

2. Opcionalmente, fixe um modelo padrão:

```bash
openclaw config set agents.defaults.videoGenerationModel.primary "google/veo-3.1-fast-generate-preview"
```

3. Peça ao agente:

> Gere um vídeo cinematográfico de 5 segundos de uma lagosta amigável surfando ao pôr do sol.

O agente chama `video_generate` automaticamente. Nenhuma allowlist de ferramentas é necessária.

## O que acontece quando você gera um vídeo

A geração de vídeo é assíncrona. Quando o agente chama `video_generate` em uma sessão:

1. O OpenClaw envia a solicitação ao provedor e retorna imediatamente um ID de tarefa.
2. O provedor processa o trabalho em segundo plano (normalmente de 30 segundos a 5 minutos, dependendo do provedor e da resolução).
3. Quando o vídeo fica pronto, o OpenClaw reativa a mesma sessão com um evento interno de conclusão.
4. O agente publica o vídeo finalizado de volta na conversa original.

Enquanto um trabalho está em andamento, chamadas duplicadas de `video_generate` na mesma sessão retornam o status atual da tarefa em vez de iniciar outra geração. Use `openclaw tasks list` ou `openclaw tasks show <taskId>` para verificar o progresso pela CLI.

Fora de execuções de agente com suporte de sessão (por exemplo, invocações diretas da ferramenta), a ferramenta recorre à geração inline e retorna o caminho final da mídia no mesmo turno.

## Provedores compatíveis

| Provedor | Modelo padrão                  | Texto | Imagem de ref.     | Vídeo de ref.    | Chave de API                              |
| -------- | ------------------------------ | ----- | ------------------ | ---------------- | ----------------------------------------- |
| Alibaba  | `wan2.6-t2v`                   | Sim   | Sim (URL remota)   | Sim (URL remota) | `MODELSTUDIO_API_KEY`                     |
| BytePlus | `seedance-1-0-lite-t2v-250428` | Sim   | 1 imagem           | Não              | `BYTEPLUS_API_KEY`                        |
| ComfyUI  | `workflow`                     | Sim   | 1 imagem           | Não              | `COMFY_API_KEY` ou `COMFY_CLOUD_API_KEY`  |
| fal      | `fal-ai/minimax/video-01-live` | Sim   | 1 imagem           | Não              | `FAL_KEY`                                 |
| Google   | `veo-3.1-fast-generate-preview`| Sim   | 1 imagem           | 1 vídeo          | `GEMINI_API_KEY`                          |
| MiniMax  | `MiniMax-Hailuo-2.3`           | Sim   | 1 imagem           | Não              | `MINIMAX_API_KEY`                         |
| OpenAI   | `sora-2`                       | Sim   | 1 imagem           | 1 vídeo          | `OPENAI_API_KEY`                          |
| Qwen     | `wan2.6-t2v`                   | Sim   | Sim (URL remota)   | Sim (URL remota) | `QWEN_API_KEY`                            |
| Runway   | `gen4.5`                       | Sim   | 1 imagem           | 1 vídeo          | `RUNWAYML_API_SECRET`                     |
| Together | `Wan-AI/Wan2.2-T2V-A14B`       | Sim   | 1 imagem           | Não              | `TOGETHER_API_KEY`                        |
| Vydra    | `veo3`                         | Sim   | 1 imagem (`kling`) | Não              | `VYDRA_API_KEY`                           |
| xAI      | `grok-imagine-video`           | Sim   | 1 imagem           | 1 vídeo          | `XAI_API_KEY`                             |

Alguns provedores aceitam variáveis de ambiente adicionais ou alternativas para chaves de API. Consulte as [páginas individuais dos provedores](#related) para mais detalhes.

Execute `video_generate action=list` para inspecionar os provedores e modelos disponíveis em tempo de execução.

## Parâmetros da ferramenta

### Obrigatórios

| Parâmetro | Tipo   | Descrição                                                                    |
| --------- | ------ | ---------------------------------------------------------------------------- |
| `prompt`  | string | Descrição em texto do vídeo a ser gerado (obrigatória para `action: "generate"`) |

### Entradas de conteúdo

| Parâmetro | Tipo     | Descrição                               |
| --------- | -------- | --------------------------------------- |
| `image`   | string   | Imagem de referência única (caminho ou URL) |
| `images`  | string[] | Várias imagens de referência (até 5)    |
| `video`   | string   | Vídeo de referência único (caminho ou URL) |
| `videos`  | string[] | Vários vídeos de referência (até 4)     |

### Controles de estilo

| Parâmetro        | Tipo    | Descrição                                                                |
| ---------------- | ------- | ------------------------------------------------------------------------ |
| `aspectRatio`    | string  | `1:1`, `2:3`, `3:2`, `3:4`, `4:3`, `4:5`, `5:4`, `9:16`, `16:9`, `21:9` |
| `resolution`     | string  | `480P`, `720P` ou `1080P`                                                |
| `durationSeconds`| number  | Duração alvo em segundos (arredondada para o valor compatível mais próximo do provedor) |
| `size`           | string  | Indicação de tamanho quando o provedor oferece suporte                   |
| `audio`          | boolean | Ativa áudio gerado quando compatível                                     |
| `watermark`      | boolean | Alterna a marca d’água do provedor quando compatível                     |

### Avançado

| Parâmetro | Tipo   | Descrição                                      |
| --------- | ------ | ---------------------------------------------- |
| `action`  | string | `"generate"` (padrão), `"status"` ou `"list"`  |
| `model`   | string | Substituição de provedor/modelo (por ex. `runway/gen4.5`) |
| `filename`| string | Indicação de nome de arquivo de saída          |

Nem todos os provedores oferecem suporte a todos os parâmetros. Substituições não compatíveis são ignoradas em regime de melhor esforço e relatadas como avisos no resultado da ferramenta. Limites rígidos de capacidade (como entradas de referência em excesso) falham antes do envio.

## Ações

- **generate** (padrão) -- cria um vídeo a partir do prompt fornecido e de entradas de referência opcionais.
- **status** -- verifica o estado da tarefa de vídeo em andamento para a sessão atual sem iniciar outra geração.
- **list** -- mostra os provedores, modelos e seus recursos disponíveis.

## Seleção de modelo

Ao gerar um vídeo, o OpenClaw resolve o modelo nesta ordem:

1. **Parâmetro de ferramenta `model`** -- se o agente especificar um na chamada.
2. **`videoGenerationModel.primary`** -- da configuração.
3. **`videoGenerationModel.fallbacks`** -- tentados em ordem.
4. **Detecção automática** -- usa provedores com autenticação válida, começando pelo provedor padrão atual e depois os demais provedores em ordem alfabética.

Se um provedor falhar, o próximo candidato será tentado automaticamente. Se todos os candidatos falharem, o erro incluirá detalhes de cada tentativa.

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "google/veo-3.1-fast-generate-preview",
        fallbacks: ["runway/gen4.5", "qwen/wan2.6-t2v"],
      },
    },
  },
}
```

## Observações sobre provedores

| Provedor | Observações                                                                                                                                                 |
| -------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Alibaba  | Usa o endpoint assíncrono do DashScope/Model Studio. Imagens e vídeos de referência devem ser URLs `http(s)` remotas.                                    |
| BytePlus | Apenas uma imagem de referência.                                                                                                                            |
| ComfyUI  | Execução local ou em nuvem orientada por workflow. Oferece suporte a texto para vídeo e imagem para vídeo por meio do grafo configurado.                 |
| fal      | Usa um fluxo baseado em fila para trabalhos longos. Apenas uma imagem de referência.                                                                       |
| Google   | Usa Gemini/Veo. Oferece suporte a uma imagem ou um vídeo de referência.                                                                                    |
| MiniMax  | Apenas uma imagem de referência.                                                                                                                            |
| OpenAI   | Apenas a substituição `size` é encaminhada. Outras substituições de estilo (`aspectRatio`, `resolution`, `audio`, `watermark`) são ignoradas com aviso. |
| Qwen     | Mesmo backend DashScope do Alibaba. As entradas de referência devem ser URLs `http(s)` remotas; arquivos locais são rejeitados antecipadamente.          |
| Runway   | Oferece suporte a arquivos locais por meio de URIs de dados. Vídeo para vídeo exige `runway/gen4_aleph`. Execuções somente com texto expõem proporções `16:9` e `9:16`. |
| Together | Apenas uma imagem de referência.                                                                                                                            |
| Vydra    | Usa `https://www.vydra.ai/api/v1` diretamente para evitar redirecionamentos que descartam a autenticação. `veo3` vem incluído apenas como texto para vídeo; `kling` exige uma URL remota de imagem. |
| xAI      | Oferece suporte a fluxos de texto para vídeo, imagem para vídeo e edição/extensão de vídeo remoto.                                                        |

## Configuração

Defina o modelo padrão de geração de vídeo na configuração do OpenClaw:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "qwen/wan2.6-t2v",
        fallbacks: ["qwen/wan2.6-r2v-flash"],
      },
    },
  },
}
```

Ou via CLI:

```bash
openclaw config set agents.defaults.videoGenerationModel.primary "qwen/wan2.6-t2v"
```

## Relacionado

- [Visão geral das ferramentas](/pt-BR/tools)
- [Tarefas em segundo plano](/pt-BR/automation/tasks) -- rastreamento de tarefas para geração assíncrona de vídeo
- [Alibaba Model Studio](/pt-BR/providers/alibaba)
- [BytePlus](/pt-BR/concepts/model-providers#byteplus-international)
- [ComfyUI](/pt-BR/providers/comfy)
- [fal](/pt-BR/providers/fal)
- [Google (Gemini)](/pt-BR/providers/google)
- [MiniMax](/pt-BR/providers/minimax)
- [OpenAI](/pt-BR/providers/openai)
- [Qwen](/pt-BR/providers/qwen)
- [Runway](/pt-BR/providers/runway)
- [Together AI](/pt-BR/providers/together)
- [Vydra](/pt-BR/providers/vydra)
- [xAI](/pt-BR/providers/xai)
- [Referência de configuração](/pt-BR/gateway/configuration-reference#agent-defaults)
- [Modelos](/pt-BR/concepts/models)
