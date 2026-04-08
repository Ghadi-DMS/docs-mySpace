---
read_when:
    - Explicar como o streaming ou a fragmentação funcionam nos canais
    - Alterar o streaming em blocos ou o comportamento de fragmentação do canal
    - Depurar respostas em bloco duplicadas/antecipadas ou streaming de prévia do canal
summary: Comportamento de streaming + fragmentação (respostas em blocos, streaming de prévia do canal, mapeamento de modos)
title: Streaming e Fragmentação
x-i18n:
    generated_at: "2026-04-08T05:26:46Z"
    model: gpt-5.4
    provider: openai
    source_hash: a8e847bb7da890818cd79dec7777f6ae488e6d6c0468e948e56b6b6c598e0000
    source_path: concepts/streaming.md
    workflow: 15
---

# Streaming + fragmentação

O OpenClaw tem duas camadas de streaming separadas:

- **Streaming em blocos (canais):** emite **blocos** concluídos conforme o assistente escreve. Essas são mensagens normais do canal (não deltas de token).
- **Streaming de prévia (Telegram/Discord/Slack):** atualiza uma **mensagem de prévia** temporária durante a geração.

Atualmente, **não há streaming real de deltas de token** para mensagens do canal. O streaming de prévia é baseado em mensagens (envio + edições/anexos).

## Streaming em blocos (mensagens do canal)

O streaming em blocos envia a saída do assistente em fragmentos maiores conforme ela fica disponível.

```
Model output
  └─ text_delta/events
       ├─ (blockStreamingBreak=text_end)
       │    └─ chunker emits blocks as buffer grows
       └─ (blockStreamingBreak=message_end)
            └─ chunker flushes at message_end
                   └─ channel send (block replies)
```

Legenda:

- `text_delta/events`: eventos de stream do modelo (podem ser esparsos para modelos sem streaming).
- `chunker`: `EmbeddedBlockChunker` aplicando limites mínimos/máximos + preferência de quebra.
- `channel send`: mensagens de saída reais (respostas em bloco).

**Controles:**

- `agents.defaults.blockStreamingDefault`: `"on"`/`"off"` (padrão: desativado).
- Sobrescritas por canal: `*.blockStreaming` (e variantes por conta) para forçar `"on"`/`"off"` por canal.
- `agents.defaults.blockStreamingBreak`: `"text_end"` ou `"message_end"`.
- `agents.defaults.blockStreamingChunk`: `{ minChars, maxChars, breakPreference? }`.
- `agents.defaults.blockStreamingCoalesce`: `{ minChars?, maxChars?, idleMs? }` (mescla blocos transmitidos antes do envio).
- Limite rígido do canal: `*.textChunkLimit` (por exemplo, `channels.whatsapp.textChunkLimit`).
- Modo de fragmentação do canal: `*.chunkMode` (`length` por padrão, `newline` divide em linhas em branco [limites de parágrafo] antes da fragmentação por comprimento).
- Limite flexível do Discord: `channels.discord.maxLinesPerMessage` (padrão: 17) divide respostas altas para evitar corte na UI.

**Semântica de limites:**

- `text_end`: transmite blocos assim que o chunker os emite; descarrega em cada `text_end`.
- `message_end`: espera até que a mensagem do assistente termine e, então, descarrega a saída em buffer.

`message_end` ainda usa o chunker se o texto em buffer exceder `maxChars`, então ele pode emitir vários fragmentos no final.

## Algoritmo de fragmentação (limites baixo/alto)

A fragmentação em blocos é implementada por `EmbeddedBlockChunker`:

- **Limite baixo:** não emite até que o buffer >= `minChars` (a menos que seja forçado).
- **Limite alto:** prefere divisões antes de `maxChars`; se forçado, divide em `maxChars`.
- **Preferência de quebra:** `paragraph` → `newline` → `sentence` → `whitespace` → quebra rígida.
- **Blocos de código:** nunca divide dentro de blocos; quando é forçado em `maxChars`, fecha + reabre o bloco para manter o Markdown válido.

`maxChars` é limitado a `textChunkLimit` do canal, então você não pode exceder os limites por canal.

## Coalescência (mesclar blocos transmitidos)

Quando o streaming em blocos está ativado, o OpenClaw pode **mesclar fragmentos consecutivos de blocos**
antes de enviá-los. Isso reduz o “spam de linha única” enquanto ainda fornece
saída progressiva.

- A coalescência espera por **intervalos de inatividade** (`idleMs`) antes de descarregar.
- Os buffers são limitados por `maxChars` e serão descarregados se o excederem.
- `minChars` evita que fragmentos muito pequenos sejam enviados até que texto suficiente se acumule
  (o descarregamento final sempre envia o texto restante).
- O joiner é derivado de `blockStreamingChunk.breakPreference`
  (`paragraph` → `\n\n`, `newline` → `\n`, `sentence` → espaço).
- Sobrescritas por canal estão disponíveis via `*.blockStreamingCoalesce` (incluindo configs por conta).
- O valor padrão de coalescência `minChars` é elevado para 1500 em Signal/Slack/Discord, a menos que seja sobrescrito.

## Ritmo humano entre blocos

Quando o streaming em blocos está ativado, você pode adicionar uma **pausa aleatória** entre
respostas em bloco (após o primeiro bloco). Isso faz com que respostas em várias bolhas pareçam
mais naturais.

- Configuração: `agents.defaults.humanDelay` (sobrescreva por agente via `agents.list[].humanDelay`).
- Modos: `off` (padrão), `natural` (800–2500ms), `custom` (`minMs`/`maxMs`).
- Aplica-se apenas a **respostas em bloco**, não a respostas finais nem a resumos de ferramentas.

## "Transmitir fragmentos ou tudo"

Isso corresponde a:

- **Transmitir fragmentos:** `blockStreamingDefault: "on"` + `blockStreamingBreak: "text_end"` (emite conforme avança). Canais que não sejam Telegram também precisam de `*.blockStreaming: true`.
- **Transmitir tudo no final:** `blockStreamingBreak: "message_end"` (descarrega uma vez, possivelmente em vários fragmentos se for muito longo).
- **Sem streaming em blocos:** `blockStreamingDefault: "off"` (apenas resposta final).

**Observação sobre canais:** O streaming em blocos fica **desativado, a menos que**
`*.blockStreaming` seja explicitamente definido como `true`. Os canais podem transmitir uma prévia ao vivo
(`channels.<channel>.streaming`) sem respostas em bloco.

Lembrete de localização da configuração: os padrões `blockStreaming*` ficam em
`agents.defaults`, não na configuração raiz.

## Modos de streaming de prévia

Chave canônica: `channels.<channel>.streaming`

Modos:

- `off`: desativa o streaming de prévia.
- `partial`: uma única prévia que é substituída pelo texto mais recente.
- `block`: atualizações de prévia em etapas fragmentadas/anexadas.
- `progress`: prévia de progresso/status durante a geração, resposta final ao concluir.

### Mapeamento por canal

| Canal    | `off` | `partial` | `block` | `progress`        |
| -------- | ----- | --------- | ------- | ----------------- |
| Telegram | ✅    | ✅        | ✅      | mapeia para `partial` |
| Discord  | ✅    | ✅        | ✅      | mapeia para `partial` |
| Slack    | ✅    | ✅        | ✅      | ✅                |

Somente no Slack:

- `channels.slack.streaming.nativeTransport` alterna chamadas da API nativa de streaming do Slack quando `channels.slack.streaming.mode="partial"` (padrão: `true`).
- O streaming nativo do Slack e o status de thread do assistente no Slack exigem um destino de thread de resposta; DMs de nível superior não mostram essa prévia no estilo de thread.

Migração de chave legada:

- Telegram: `streamMode` + booleano `streaming` são migrados automaticamente para o enum `streaming`.
- Discord: `streamMode` + booleano `streaming` são migrados automaticamente para o enum `streaming`.
- Slack: `streamMode` é migrado automaticamente para `streaming.mode`; o booleano `streaming` é migrado automaticamente para `streaming.mode` mais `streaming.nativeTransport`; o legado `nativeStreaming` é migrado automaticamente para `streaming.nativeTransport`.

### Comportamento em tempo de execução

Telegram:

- Usa atualizações de prévia com `sendMessage` + `editMessageText` em DMs e grupos/tópicos.
- O streaming de prévia é ignorado quando o streaming em blocos do Telegram está explicitamente ativado (para evitar streaming duplo).
- `/reasoning stream` pode gravar o raciocínio na prévia.

Discord:

- Usa mensagens de prévia com envio + edição.
- O modo `block` usa fragmentação de rascunho (`draftChunk`).
- O streaming de prévia é ignorado quando o streaming em blocos do Discord está explicitamente ativado.

Slack:

- `partial` pode usar o streaming nativo do Slack (`chat.startStream`/`append`/`stop`) quando disponível.
- `block` usa prévias de rascunho no estilo append.
- `progress` usa texto de prévia de status e, depois, a resposta final.

## Relacionado

- [Mensagens](/pt-BR/concepts/messages) — ciclo de vida e entrega de mensagens
- [Retry](/pt-BR/concepts/retry) — comportamento de repetição em falhas de entrega
- [Canais](/pt-BR/channels) — suporte a streaming por canal
