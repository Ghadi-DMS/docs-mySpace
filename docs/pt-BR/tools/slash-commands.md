---
read_when:
    - Usar ou configurar comandos de chat
    - Depurar roteamento de comandos ou permissões
summary: 'Comandos slash: texto vs. nativo, configuração e comandos compatíveis'
title: Comandos Slash
x-i18n:
    generated_at: "2026-04-08T05:27:39Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4a7ee7f1a8012058279b9e632889b291d4e659e4ec81209ca8978afbb9ad4b96
    source_path: tools/slash-commands.md
    workflow: 15
---

# Comandos slash

Os comandos são tratados pelo Gateway. A maioria dos comandos deve ser enviada como uma mensagem **autônoma** que começa com `/`.
O comando de chat bash somente no host usa `! <cmd>` (com `/bash <cmd>` como alias).

Há dois sistemas relacionados:

- **Comandos**: mensagens autônomas `/...`.
- **Diretivas**: `/think`, `/fast`, `/verbose`, `/reasoning`, `/elevated`, `/exec`, `/model`, `/queue`.
  - As diretivas são removidas da mensagem antes que o modelo a veja.
  - Em mensagens normais de chat (não apenas com diretivas), elas são tratadas como “dicas inline” e **não** persistem as configurações da sessão.
  - Em mensagens somente com diretivas (a mensagem contém apenas diretivas), elas persistem na sessão e respondem com uma confirmação.
  - As diretivas só são aplicadas para **remetentes autorizados**. Se `commands.allowFrom` estiver definido, ele será a única
    lista de permissões usada; caso contrário, a autorização virá das listas de permissões/emparelhamento do canal mais `commands.useAccessGroups`.
    Remetentes não autorizados veem as diretivas tratadas como texto simples.

Há também alguns **atalhos inline** (apenas remetentes na allowlist/autorizados): `/help`, `/commands`, `/status`, `/whoami` (`/id`).
Eles são executados imediatamente, são removidos antes que o modelo veja a mensagem, e o texto restante continua pelo fluxo normal.

## Configuração

```json5
{
  commands: {
    native: "auto",
    nativeSkills: "auto",
    text: true,
    bash: false,
    bashForegroundMs: 2000,
    config: false,
    mcp: false,
    plugins: false,
    debug: false,
    restart: true,
    ownerAllowFrom: ["discord:123456789012345678"],
    ownerDisplay: "raw",
    ownerDisplaySecret: "${OWNER_ID_HASH_SECRET}",
    allowFrom: {
      "*": ["user1"],
      discord: ["user:123"],
    },
    useAccessGroups: true,
  },
}
```

- `commands.text` (padrão `true`) ativa a análise de `/...` em mensagens de chat.
  - Em superfícies sem comandos nativos (WhatsApp/WebChat/Signal/iMessage/Google Chat/Microsoft Teams), os comandos de texto ainda funcionam mesmo se você definir isso como `false`.
- `commands.native` (padrão `"auto"`) registra comandos nativos.
  - Auto: ativado para Discord/Telegram; desativado para Slack (até você adicionar comandos slash); ignorado para provedores sem suporte nativo.
  - Defina `channels.discord.commands.native`, `channels.telegram.commands.native` ou `channels.slack.commands.native` para sobrescrever por provedor (bool ou `"auto"`).
  - `false` limpa comandos registrados anteriormente no Discord/Telegram na inicialização. Os comandos do Slack são gerenciados no app do Slack e não são removidos automaticamente.
- `commands.nativeSkills` (padrão `"auto"`) registra comandos nativos de **skill** quando compatível.
  - Auto: ativado para Discord/Telegram; desativado para Slack (o Slack exige a criação de um comando slash por skill).
  - Defina `channels.discord.commands.nativeSkills`, `channels.telegram.commands.nativeSkills` ou `channels.slack.commands.nativeSkills` para sobrescrever por provedor (bool ou `"auto"`).
- `commands.bash` (padrão `false`) ativa `! <cmd>` para executar comandos do shell no host (`/bash <cmd>` é um alias; exige allowlists de `tools.elevated`).
- `commands.bashForegroundMs` (padrão `2000`) controla por quanto tempo o bash espera antes de alternar para o modo em segundo plano (`0` coloca em segundo plano imediatamente).
- `commands.config` (padrão `false`) ativa `/config` (lê/grava `openclaw.json`).
- `commands.mcp` (padrão `false`) ativa `/mcp` (lê/grava a configuração de MCP gerenciada pelo OpenClaw em `mcp.servers`).
- `commands.plugins` (padrão `false`) ativa `/plugins` (descoberta/status de plugins mais controles de instalação + ativação/desativação).
- `commands.debug` (padrão `false`) ativa `/debug` (sobrescritas somente em tempo de execução).
- `commands.restart` (padrão `true`) ativa `/restart` mais ações de ferramenta de reinicialização do gateway.
- `commands.ownerAllowFrom` (opcional) define a allowlist explícita do proprietário para superfícies de comando/ferramenta somente do proprietário. Isso é separado de `commands.allowFrom`.
- `commands.ownerDisplay` controla como os ids do proprietário aparecem no prompt do sistema: `raw` ou `hash`.
- `commands.ownerDisplaySecret` opcionalmente define o segredo HMAC usado quando `commands.ownerDisplay="hash"`.
- `commands.allowFrom` (opcional) define uma allowlist por provedor para autorização de comandos. Quando configurado, ela é a
  única fonte de autorização para comandos e diretivas (allowlists/emparelhamento do canal e `commands.useAccessGroups`
  são ignorados). Use `"*"` para um padrão global; chaves específicas de provedor têm prioridade.
- `commands.useAccessGroups` (padrão `true`) aplica allowlists/políticas a comandos quando `commands.allowFrom` não está definido.

## Lista de comandos

Fonte da verdade atual:

- os built-ins do core vêm de `src/auto-reply/commands-registry.shared.ts`
- os comandos dock gerados vêm de `src/auto-reply/commands-registry.data.ts`
- os comandos de plugin vêm de chamadas `registerCommand()` do plugin
- a disponibilidade real no seu gateway ainda depende de flags de configuração, da superfície do canal e de plugins instalados/ativados

### Comandos built-in do core

Comandos built-in disponíveis hoje:

- `/new [model]` inicia uma nova sessão; `/reset` é o alias de redefinição.
- `/compact [instructions]` compacta o contexto da sessão. Veja [/concepts/compaction](/pt-BR/concepts/compaction).
- `/stop` aborta a execução atual.
- `/session idle <duration|off>` e `/session max-age <duration|off>` gerenciam a expiração do vínculo com a thread.
- `/think <off|minimal|low|medium|high|xhigh>` define o nível de raciocínio. Aliases: `/thinking`, `/t`.
- `/verbose on|off|full` alterna a saída detalhada. Alias: `/v`.
- `/fast [status|on|off]` mostra ou define o modo rápido.
- `/reasoning [on|off|stream]` alterna a visibilidade do raciocínio. Alias: `/reason`.
- `/elevated [on|off|ask|full]` alterna o modo elevado. Alias: `/elev`.
- `/exec host=<auto|sandbox|gateway|node> security=<deny|allowlist|full> ask=<off|on-miss|always> node=<id>` mostra ou define os padrões de execução.
- `/model [name|#|status]` mostra ou define o modelo.
- `/models [provider] [page] [limit=<n>|size=<n>|all]` lista provedores ou modelos de um provedor.
- `/queue <mode>` gerencia o comportamento da fila (`steer`, `interrupt`, `followup`, `collect`, `steer-backlog`) mais opções como `debounce:2s cap:25 drop:summarize`.
- `/help` mostra o resumo curto de ajuda.
- `/commands` mostra o catálogo gerado de comandos.
- `/tools [compact|verbose]` mostra o que o agente atual pode usar neste momento.
- `/status` mostra o status em tempo de execução, incluindo uso/cota do provedor quando disponível.
- `/tasks` lista tarefas em segundo plano ativas/recentes da sessão atual.
- `/context [list|detail|json]` explica como o contexto é montado.
- `/export-session [path]` exporta a sessão atual para HTML. Alias: `/export`.
- `/whoami` mostra seu id de remetente. Alias: `/id`.
- `/skill <name> [input]` executa uma skill pelo nome.
- `/allowlist [list|add|remove] ...` gerencia entradas da allowlist. Somente texto.
- `/approve <id> <decision>` resolve prompts de aprovação de exec.
- `/btw <question>` faz uma pergunta paralela sem alterar o contexto futuro da sessão. Veja [/tools/btw](/pt-BR/tools/btw).
- `/subagents list|kill|log|info|send|steer|spawn` gerencia execuções de subagentes para a sessão atual.
- `/acp spawn|cancel|steer|close|sessions|status|set-mode|set|cwd|permissions|timeout|model|reset-options|doctor|install|help` gerencia sessões ACP e opções de tempo de execução.
- `/focus <target>` vincula a thread atual do Discord ou o tópico/conversa do Telegram a um alvo de sessão.
- `/unfocus` remove o vínculo atual.
- `/agents` lista agentes vinculados a threads da sessão atual.
- `/kill <id|#|all>` aborta um ou todos os subagentes em execução.
- `/steer <id|#> <message>` envia direcionamento para um subagente em execução. Alias: `/tell`.
- `/config show|get|set|unset` lê ou grava `openclaw.json`. Somente proprietário. Exige `commands.config: true`.
- `/mcp show|get|set|unset` lê ou grava a configuração do servidor MCP gerenciada pelo OpenClaw em `mcp.servers`. Somente proprietário. Exige `commands.mcp: true`.
- `/plugins list|inspect|show|get|install|enable|disable` inspeciona ou altera o estado de plugins. `/plugin` é um alias. Somente proprietário para gravações. Exige `commands.plugins: true`.
- `/debug show|set|unset|reset` gerencia sobrescritas de configuração somente em tempo de execução. Somente proprietário. Exige `commands.debug: true`.
- `/usage off|tokens|full|cost` controla o rodapé de uso por resposta ou imprime um resumo local de custos.
- `/tts on|off|status|provider|limit|summary|audio|help` controla TTS. Veja [/tools/tts](/pt-BR/tools/tts).
- `/restart` reinicia o OpenClaw quando ativado. Padrão: ativado; defina `commands.restart: false` para desativá-lo.
- `/activation mention|always` define o modo de ativação em grupo.
- `/send on|off|inherit` define a política de envio. Somente proprietário.
- `/bash <command>` executa um comando de shell no host. Somente texto. Alias: `! <command>`. Exige `commands.bash: true` mais allowlists de `tools.elevated`.
- `!poll [sessionId]` verifica um job bash em segundo plano.
- `!stop [sessionId]` interrompe um job bash em segundo plano.

### Comandos dock gerados

Os comandos dock são gerados a partir de plugins de canal com suporte a comando nativo. Conjunto empacotado atual:

- `/dock-discord` (alias: `/dock_discord`)
- `/dock-mattermost` (alias: `/dock_mattermost`)
- `/dock-slack` (alias: `/dock_slack`)
- `/dock-telegram` (alias: `/dock_telegram`)

### Comandos de plugins empacotados

Plugins empacotados podem adicionar mais comandos slash. Comandos empacotados atuais neste repositório:

- `/dreaming [on|off|status|help]` alterna o dreaming de memória. Veja [Dreaming](/pt-BR/concepts/dreaming).
- `/pair [qr|status|pending|approve|cleanup|notify]` gerencia o fluxo de emparelhamento/configuração do dispositivo. Veja [Pairing](/pt-BR/channels/pairing).
- `/phone status|arm <camera|screen|writes|all> [duration]|disarm` ativa temporariamente comandos de nó de telefone de alto risco.
- `/voice status|list [limit]|set <voiceId|name>` gerencia a configuração de voz do Talk. No Discord, o nome do comando nativo é `/talkvoice`.
- `/card ...` envia predefinições de rich card do LINE. Veja [LINE](/pt-BR/channels/line).
- Comandos exclusivos do QQBot:
  - `/bot-ping`
  - `/bot-version`
  - `/bot-help`
  - `/bot-upgrade`
  - `/bot-logs`

### Comandos dinâmicos de skill

Skills invocáveis pelo usuário também são expostas como comandos slash:

- `/skill <name> [input]` sempre funciona como ponto de entrada genérico.
- skills também podem aparecer como comandos diretos como `/prose` quando a skill/o plugin os registra.
- o registro de comandos nativos de skill é controlado por `commands.nativeSkills` e `channels.<provider>.commands.nativeSkills`.

Observações:

- Os comandos aceitam um `:` opcional entre o comando e os argumentos (por exemplo, `/think: high`, `/send: on`, `/help:`).
- `/new <model>` aceita um alias de modelo, `provider/model` ou um nome de provedor (correspondência aproximada); se não houver correspondência, o texto será tratado como corpo da mensagem.
- Para o detalhamento completo de uso por provedor, use `openclaw status --usage`.
- `/allowlist add|remove` exige `commands.config=true` e respeita `configWrites` do canal.
- Em canais com várias contas, `/allowlist --account <id>` direcionado à configuração e `/config set channels.<provider>.accounts.<id>...` também respeitam `configWrites` da conta de destino.
- `/usage` controla o rodapé de uso por resposta; `/usage cost` imprime um resumo local de custos a partir dos logs de sessão do OpenClaw.
- `/restart` é ativado por padrão; defina `commands.restart: false` para desativá-lo.
- `/plugins install <spec>` aceita as mesmas especificações de plugin que `openclaw plugins install`: caminho/arquivo local, pacote npm ou `clawhub:<pkg>`.
- `/plugins enable|disable` atualiza a configuração do plugin e pode solicitar uma reinicialização.
- Comando nativo exclusivo do Discord: `/vc join|leave|status` controla canais de voz (exige `channels.discord.voice` e comandos nativos; não está disponível como texto).
- Os comandos de vínculo com thread do Discord (`/focus`, `/unfocus`, `/agents`, `/session idle`, `/session max-age`) exigem que os vínculos efetivos com thread estejam ativados (`session.threadBindings.enabled` e/ou `channels.discord.threadBindings.enabled`).
- Referência de comandos ACP e comportamento em tempo de execução: [ACP Agents](/pt-BR/tools/acp-agents).
- `/verbose` é destinado a depuração e visibilidade extra; mantenha-o **desativado** no uso normal.
- `/fast on|off` persiste uma sobrescrita de sessão. Use a opção `inherit` da UI de Sessões para limpá-la e voltar aos padrões da configuração.
- `/fast` é específico do provedor: OpenAI/OpenAI Codex o mapeiam para `service_tier=priority` em endpoints nativos de Responses, enquanto solicitações públicas diretas ao Anthropic, incluindo tráfego autenticado por OAuth enviado para `api.anthropic.com`, o mapeiam para `service_tier=auto` ou `standard_only`. Veja [OpenAI](/pt-BR/providers/openai) e [Anthropic](/pt-BR/providers/anthropic).
- Resumos de falha de ferramenta ainda são mostrados quando relevantes, mas o texto detalhado da falha só é incluído quando `/verbose` está `on` ou `full`.
- `/reasoning` (e `/verbose`) são arriscados em ambientes de grupo: eles podem revelar raciocínio interno ou saída de ferramenta que você não pretendia expor. Prefira deixá-los desativados, especialmente em chats em grupo.
- `/model` persiste o novo modelo da sessão imediatamente.
- Se o agente estiver ocioso, a próxima execução o usará imediatamente.
- Se já houver uma execução ativa, o OpenClaw marca uma troca ao vivo como pendente e só reinicia no novo modelo em um ponto limpo de tentativa.
- Se a atividade de ferramenta ou a saída de resposta já tiver começado, a troca pendente pode ficar na fila até uma oportunidade posterior de tentativa ou até o próximo turno do usuário.
- **Caminho rápido:** mensagens somente de comando de remetentes na allowlist são tratadas imediatamente (ignoram fila + modelo).
- **Gate de menção em grupo:** mensagens somente de comando de remetentes na allowlist ignoram requisitos de menção.
- **Atalhos inline (apenas remetentes na allowlist):** certos comandos também funcionam quando embutidos em uma mensagem normal e são removidos antes que o modelo veja o texto restante.
  - Exemplo: `hey /status` dispara uma resposta de status, e o texto restante continua pelo fluxo normal.
- Atualmente: `/help`, `/commands`, `/status`, `/whoami` (`/id`).
- Mensagens somente de comando não autorizadas são ignoradas silenciosamente, e tokens inline `/...` são tratados como texto simples.
- **Comandos de skill:** skills `user-invocable` são expostas como comandos slash. Os nomes são sanitizados para `a-z0-9_` (máximo de 32 caracteres); colisões recebem sufixos numéricos (por exemplo, `_2`).
  - `/skill <name> [input]` executa uma skill pelo nome (útil quando limites de comandos nativos impedem comandos por skill).
  - Por padrão, os comandos de skill são encaminhados ao modelo como uma solicitação normal.
  - Skills podem opcionalmente declarar `command-dispatch: tool` para rotear o comando diretamente para uma ferramenta (determinístico, sem modelo).
  - Exemplo: `/prose` (plugin OpenProse) — veja [OpenProse](/pt-BR/prose).
- **Argumentos de comando nativo:** o Discord usa preenchimento automático para opções dinâmicas (e menus de botão quando você omite argumentos obrigatórios). Telegram e Slack mostram um menu de botões quando um comando aceita escolhas e você omite o argumento.

## `/tools`

`/tools` responde a uma pergunta de tempo de execução, não a uma pergunta de configuração: **o que este agente pode usar agora
nesta conversa**.

- O padrão de `/tools` é compacto e otimizado para leitura rápida.
- `/tools verbose` adiciona descrições curtas.
- Superfícies de comando nativo com suporte a argumentos expõem o mesmo seletor de modo `compact|verbose`.
- Os resultados são limitados ao escopo da sessão, então mudar agente, canal, thread, autorização do remetente ou modelo pode
  mudar a saída.
- `/tools` inclui ferramentas realmente acessíveis em tempo de execução, incluindo ferramentas do core, ferramentas
  de plugins conectados e ferramentas pertencentes ao canal.

Para editar perfil e sobrescritas, use o painel Tools da UI de Controle ou superfícies de configuração/catálogo em vez
de tratar `/tools` como um catálogo estático.

## Superfícies de uso (o que aparece onde)

- **Uso/cota do provedor** (exemplo: “Claude 80% restante”) aparece em `/status` para o provedor de modelo atual quando o rastreamento de uso está ativado. O OpenClaw normaliza janelas de provedor para `% restante`; para o MiniMax, campos percentuais apenas de restante são invertidos antes da exibição, e respostas `model_remains` preferem a entrada do modelo de chat mais um rótulo de plano com tag de modelo.
- **Linhas de token/cache** em `/status` podem recorrer à entrada de uso mais recente do transcript quando o snapshot da sessão ativa é escasso. Valores ativos não nulos ainda têm prioridade, e o fallback do transcript também pode recuperar o rótulo do modelo de tempo de execução ativo mais um total maior orientado a prompt quando os totais armazenados estão ausentes ou são menores.
- **Tokens/custo por resposta** é controlado por `/usage off|tokens|full` (anexado às respostas normais).
- `/model status` é sobre **modelos/auth/endpoints**, não sobre uso.

## Seleção de modelo (`/model`)

`/model` é implementado como uma diretiva.

Exemplos:

```
/model
/model list
/model 3
/model openai/gpt-5.4
/model opus@anthropic:default
/model status
```

Observações:

- `/model` e `/model list` mostram um seletor compacto e numerado (família do modelo + provedores disponíveis).
- No Discord, `/model` e `/models` abrem um seletor interativo com dropdowns de provedor e modelo mais uma etapa Submit.
- `/model <#>` seleciona a partir desse seletor (e prefere o provedor atual quando possível).
- `/model status` mostra a visualização detalhada, incluindo o endpoint configurado do provedor (`baseUrl`) e o modo de API (`api`) quando disponível.

## Sobrescritas de depuração

`/debug` permite definir sobrescritas de configuração **somente em tempo de execução** (memória, não disco). Somente proprietário. Desativado por padrão; ative com `commands.debug: true`.

Exemplos:

```
/debug show
/debug set messages.responsePrefix="[openclaw]"
/debug set channels.whatsapp.allowFrom=["+1555","+4477"]
/debug unset messages.responsePrefix
/debug reset
```

Observações:

- As sobrescritas se aplicam imediatamente a novas leituras de configuração, mas **não** gravam em `openclaw.json`.
- Use `/debug reset` para limpar todas as sobrescritas e retornar à configuração em disco.

## Atualizações de configuração

`/config` grava na sua configuração em disco (`openclaw.json`). Somente proprietário. Desativado por padrão; ative com `commands.config: true`.

Exemplos:

```
/config show
/config show messages.responsePrefix
/config get messages.responsePrefix
/config set messages.responsePrefix="[openclaw]"
/config unset messages.responsePrefix
```

Observações:

- A configuração é validada antes da gravação; alterações inválidas são rejeitadas.
- As atualizações de `/config` persistem entre reinicializações.

## Atualizações de MCP

`/mcp` grava definições de servidor MCP gerenciadas pelo OpenClaw em `mcp.servers`. Somente proprietário. Desativado por padrão; ative com `commands.mcp: true`.

Exemplos:

```text
/mcp show
/mcp show context7
/mcp set context7={"command":"uvx","args":["context7-mcp"]}
/mcp unset context7
```

Observações:

- `/mcp` armazena a configuração na configuração do OpenClaw, não nas configurações de projeto pertencentes ao Pi.
- Adaptadores de tempo de execução decidem quais transportes são realmente executáveis.

## Atualizações de plugin

`/plugins` permite que operadores inspecionem plugins descobertos e alternem a ativação na configuração. Fluxos somente leitura podem usar `/plugin` como alias. Desativado por padrão; ative com `commands.plugins: true`.

Exemplos:

```text
/plugins
/plugins list
/plugin show context7
/plugins enable context7
/plugins disable context7
```

Observações:

- `/plugins list` e `/plugins show` usam descoberta real de plugins no workspace atual mais a configuração em disco.
- `/plugins enable|disable` atualiza apenas a configuração do plugin; não instala nem desinstala plugins.
- Após alterações de ativação/desativação, reinicie o gateway para aplicá-las.

## Observações sobre superfícies

- **Comandos de texto** são executados na sessão normal de chat (DMs compartilham `main`, grupos têm sua própria sessão).
- **Comandos nativos** usam sessões isoladas:
  - Discord: `agent:<agentId>:discord:slash:<userId>`
  - Slack: `agent:<agentId>:slack:slash:<userId>` (prefixo configurável via `channels.slack.slashCommand.sessionPrefix`)
  - Telegram: `telegram:slash:<userId>` (direciona para a sessão do chat via `CommandTargetSessionKey`)
- **`/stop`** tem como alvo a sessão ativa de chat para poder abortar a execução atual.
- **Slack:** `channels.slack.slashCommand` ainda é compatível para um único comando no estilo `/openclaw`. Se você ativar `commands.native`, precisará criar um comando slash do Slack para cada comando built-in (mesmos nomes de `/help`). Menus de argumentos de comando para o Slack são entregues como botões Block Kit efêmeros.
  - Exceção nativa do Slack: registre `/agentstatus` (não `/status`) porque o Slack reserva `/status`. O texto `/status` ainda funciona em mensagens do Slack.

## Perguntas paralelas com BTW

`/btw` é uma **pergunta paralela** rápida sobre a sessão atual.

Diferentemente do chat normal:

- ele usa a sessão atual como contexto de fundo,
- ele é executado como uma chamada isolada **sem ferramentas**,
- ele não altera o contexto futuro da sessão,
- ele não é gravado no histórico do transcript,
- ele é entregue como um resultado paralelo ao vivo em vez de uma mensagem normal do assistente.

Isso torna `/btw` útil quando você quer um esclarecimento temporário enquanto a
tarefa principal continua.

Exemplo:

```text
/btw o que estamos fazendo agora?
```

Veja [BTW Side Questions](/pt-BR/tools/btw) para o comportamento completo e detalhes
da UX do cliente.
