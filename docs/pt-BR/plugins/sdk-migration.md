---
read_when:
    - Você vê o aviso OPENCLAW_PLUGIN_SDK_COMPAT_DEPRECATED
    - Você vê o aviso OPENCLAW_EXTENSION_API_DEPRECATED
    - Você está atualizando um plugin para a arquitetura moderna de plugins
    - Você mantém um plugin externo do OpenClaw
sidebarTitle: Migrate to SDK
summary: Migrar da camada legada de compatibilidade retroativa para o SDK moderno de plugins
title: Migração do SDK de plugins
x-i18n:
    generated_at: "2026-04-06T05:35:54Z"
    model: gpt-5.4
    provider: openai
    source_hash: 94f12d1376edd8184714cc4dbea4a88fa8ed652f65e9365ede6176f3bf441b33
    source_path: plugins/sdk-migration.md
    workflow: 15
---

# Migração do SDK de plugins

O OpenClaw passou de uma ampla camada de compatibilidade retroativa para uma
arquitetura moderna de plugins com imports específicos e documentados. Se o seu
plugin foi criado antes da nova arquitetura, este guia ajuda você a migrar.

## O que está mudando

O sistema antigo de plugins fornecia duas superfícies amplas que permitiam que
os plugins importassem qualquer coisa de que precisassem a partir de um único
ponto de entrada:

- **`openclaw/plugin-sdk/compat`** — um único import que reexportava dezenas de
  helpers. Ele foi introduzido para manter plugins antigos baseados em hooks
  funcionando enquanto a nova arquitetura de plugins estava sendo construída.
- **`openclaw/extension-api`** — uma ponte que dava aos plugins acesso direto a
  helpers do host, como o executor de agente incorporado.

Ambas as superfícies agora estão **obsoletas**. Elas ainda funcionam em tempo
de execução, mas novos plugins não devem usá-las, e plugins existentes devem
migrar antes que a próxima versão principal as remova.

<Warning>
  A camada de compatibilidade retroativa será removida em uma futura versão principal.
  Plugins que ainda importarem dessas superfícies deixarão de funcionar quando isso acontecer.
</Warning>

## Por que isso mudou

A abordagem antiga causava problemas:

- **Inicialização lenta** — importar um helper carregava dezenas de módulos não relacionados
- **Dependências circulares** — reexportações amplas facilitavam a criação de ciclos de import
- **Superfície de API pouco clara** — não havia como saber quais exports eram estáveis versus internos

O SDK moderno de plugins corrige isso: cada caminho de import (`openclaw/plugin-sdk/\<subpath\>`)
é um módulo pequeno e autocontido com um propósito claro e um contrato documentado.

As conveniências legadas para providers em canais empacotados também acabaram. Imports
como `openclaw/plugin-sdk/slack`, `openclaw/plugin-sdk/discord`,
`openclaw/plugin-sdk/signal`, `openclaw/plugin-sdk/whatsapp`,
superfícies helper com marca do canal e
`openclaw/plugin-sdk/telegram-core` eram atalhos privados do mono-repo, não
contratos estáveis de plugin. Use subcaminhos genéricos e específicos do SDK.
Dentro do workspace de plugins empacotados, mantenha helpers do provider no
próprio `api.ts` ou `runtime-api.ts` desse plugin.

Exemplos atuais de providers empacotados:

- Anthropic mantém helpers de stream específicos do Claude em sua própria superfície `api.ts` /
  `contract-api.ts`
- OpenAI mantém builders de provider, helpers de modelo padrão e builders de provider
  em tempo real em seu próprio `api.ts`
- OpenRouter mantém helper de builder de provider e helpers de onboarding/configuração em seu próprio
  `api.ts`

## Como migrar

<Steps>
  <Step title="Audite o comportamento de fallback do wrapper do Windows">
    Se o seu plugin usa `openclaw/plugin-sdk/windows-spawn`, wrappers `.cmd`/`.bat`
    não resolvidos no Windows agora falham de forma segura, a menos que você passe
    explicitamente `allowShellFallback: true`.

    ```typescript
    // Antes
    const program = applyWindowsSpawnProgramPolicy({ candidate });

    // Depois
    const program = applyWindowsSpawnProgramPolicy({
      candidate,
      // Defina isso apenas para chamadores de compatibilidade confiáveis que
      // intencionalmente aceitam fallback mediado por shell.
      allowShellFallback: true,
    });
    ```

    Se o seu chamador não depende intencionalmente de fallback por shell, não defina
    `allowShellFallback` e trate o erro lançado.

  </Step>

  <Step title="Encontre imports obsoletos">
    Procure no seu plugin por imports de qualquer uma das superfícies obsoletas:

    ```bash
    grep -r "plugin-sdk/compat" my-plugin/
    grep -r "openclaw/extension-api" my-plugin/
    ```

  </Step>

  <Step title="Substitua por imports específicos">
    Cada export da superfície antiga corresponde a um caminho de import moderno específico:

    ```typescript
    // Antes (camada obsoleta de compatibilidade retroativa)
    import {
      createChannelReplyPipeline,
      createPluginRuntimeStore,
      resolveControlCommandGate,
    } from "openclaw/plugin-sdk/compat";

    // Depois (imports modernos e específicos)
    import { createChannelReplyPipeline } from "openclaw/plugin-sdk/channel-reply-pipeline";
    import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
    import { resolveControlCommandGate } from "openclaw/plugin-sdk/command-auth";
    ```

    Para helpers do lado do host, use o runtime de plugin injetado em vez de importar
    diretamente:

    ```typescript
    // Antes (ponte obsoleta extension-api)
    import { runEmbeddedPiAgent } from "openclaw/extension-api";
    const result = await runEmbeddedPiAgent({ sessionId, prompt });

    // Depois (runtime injetado)
    const result = await api.runtime.agent.runEmbeddedPiAgent({ sessionId, prompt });
    ```

    O mesmo padrão se aplica a outros helpers legados da ponte:

    | Import antigo | Equivalente moderno |
    | --- | --- |
    | `resolveAgentDir` | `api.runtime.agent.resolveAgentDir` |
    | `resolveAgentWorkspaceDir` | `api.runtime.agent.resolveAgentWorkspaceDir` |
    | `resolveAgentIdentity` | `api.runtime.agent.resolveAgentIdentity` |
    | `resolveThinkingDefault` | `api.runtime.agent.resolveThinkingDefault` |
    | `resolveAgentTimeoutMs` | `api.runtime.agent.resolveAgentTimeoutMs` |
    | `ensureAgentWorkspace` | `api.runtime.agent.ensureAgentWorkspace` |
    | helpers do armazenamento de sessão | `api.runtime.agent.session.*` |

  </Step>

  <Step title="Compile e teste">
    ```bash
    pnpm build
    pnpm test -- my-plugin/
    ```
  </Step>
</Steps>

## Referência de caminhos de import

<Accordion title="Tabela comum de caminhos de import">
  | Caminho de import | Finalidade | Exports principais |
  | --- | --- | --- |
  | `plugin-sdk/plugin-entry` | Helper de entrada canônico de plugin | `definePluginEntry` |
  | `plugin-sdk/core` | Reexportação legada abrangente para definições/builders de entrada de canal | `defineChannelPluginEntry`, `createChatChannelPlugin` |
  | `plugin-sdk/config-schema` | Export da raiz do esquema de configuração | `OpenClawSchema` |
  | `plugin-sdk/provider-entry` | Helper de entrada para um único provider | `defineSingleProviderPluginEntry` |
  | `plugin-sdk/channel-core` | Definições e builders específicos de entrada de canal | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
  | `plugin-sdk/setup` | Helpers compartilhados do assistente de configuração | Prompts de lista de permissões, builders de status de configuração |
  | `plugin-sdk/setup-runtime` | Helpers de runtime em tempo de configuração | Adaptadores de patch de configuração seguros para import, helpers de nota de consulta, `promptResolvedAllowFrom`, `splitSetupEntries`, proxies de configuração delegada |
  | `plugin-sdk/setup-adapter-runtime` | Helpers de adaptador de configuração | `createEnvPatchedAccountSetupAdapter` |
  | `plugin-sdk/setup-tools` | Helpers de ferramentas de configuração | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
  | `plugin-sdk/account-core` | Helpers para múltiplas contas | Helpers de lista/configuração de contas/gate de ações |
  | `plugin-sdk/account-id` | Helpers de ID de conta | `DEFAULT_ACCOUNT_ID`, normalização de ID de conta |
  | `plugin-sdk/account-resolution` | Helpers de resolução de conta | Helpers de busca de conta + fallback padrão |
  | `plugin-sdk/account-helpers` | Helpers específicos de conta | Helpers de lista de contas/ação de conta |
  | `plugin-sdk/channel-setup` | Adaptadores do assistente de configuração | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, além de `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
  | `plugin-sdk/channel-pairing` | Primitivos de pareamento por DM | `createChannelPairingController` |
  | `plugin-sdk/channel-reply-pipeline` | Prefixo de resposta + integração de indicador de digitação | `createChannelReplyPipeline` |
  | `plugin-sdk/channel-config-helpers` | Fábricas de adaptadores de configuração | `createHybridChannelConfigAdapter` |
  | `plugin-sdk/channel-config-schema` | Builders de esquema de configuração | Tipos de esquema de configuração de canal |
  | `plugin-sdk/telegram-command-config` | Helpers de configuração de comando do Telegram | Normalização de nome de comando, trim de descrição, validação de duplicados/conflitos |
  | `plugin-sdk/channel-policy` | Resolução de políticas de grupo/DM | `resolveChannelGroupRequireMention` |
  | `plugin-sdk/channel-lifecycle` | Rastreamento de status de conta | `createAccountStatusSink` |
  | `plugin-sdk/inbound-envelope` | Helpers de envelope de entrada | Helpers compartilhados de rota + builder de envelope |
  | `plugin-sdk/inbound-reply-dispatch` | Helpers de resposta de entrada | Helpers compartilhados de registro e despacho |
  | `plugin-sdk/messaging-targets` | Parsing de alvos de mensagens | Helpers de parsing/correspondência de alvo |
  | `plugin-sdk/outbound-media` | Helpers de mídia de saída | Carregamento compartilhado de mídia de saída |
  | `plugin-sdk/outbound-runtime` | Helpers de runtime de saída | Helpers de identidade/delegação de envio de saída |
  | `plugin-sdk/thread-bindings-runtime` | Helpers de associação de threads | Ciclo de vida de associação de threads e helpers de adaptador |
  | `plugin-sdk/agent-media-payload` | Helpers legados de payload de mídia | Builder de payload de mídia do agente para layouts legados de campos |
  | `plugin-sdk/channel-runtime` | Shim de compatibilidade obsoleto | Apenas utilitários legados de runtime de canal |
  | `plugin-sdk/channel-send-result` | Tipos de resultado de envio | Tipos de resultado de resposta |
  | `plugin-sdk/runtime-store` | Armazenamento persistente do plugin | `createPluginRuntimeStore` |
  | `plugin-sdk/runtime` | Helpers amplos de runtime | Helpers de runtime/log/backup/instalação de plugin |
  | `plugin-sdk/runtime-env` | Helpers específicos de ambiente de runtime | Logger/ambiente de runtime, timeout, retry e helpers de backoff |
  | `plugin-sdk/plugin-runtime` | Helpers compartilhados de runtime de plugin | Helpers de comandos/hooks/http/interativos do plugin |
  | `plugin-sdk/hook-runtime` | Helpers de pipeline de hook | Helpers compartilhados de pipeline de webhook/hook interno |
  | `plugin-sdk/lazy-runtime` | Helpers de runtime lazy | `createLazyRuntimeModule`, `createLazyRuntimeMethod`, `createLazyRuntimeMethodBinder`, `createLazyRuntimeNamedExport`, `createLazyRuntimeSurface` |
  | `plugin-sdk/process-runtime` | Helpers de processo | Helpers compartilhados de execução |
  | `plugin-sdk/cli-runtime` | Helpers de runtime da CLI | Formatação de comando, esperas, helpers de versão |
  | `plugin-sdk/gateway-runtime` | Helpers de gateway | Cliente de gateway e helpers de patch de status de canal |
  | `plugin-sdk/config-runtime` | Helpers de configuração | Helpers de carregamento/gravação de configuração |
  | `plugin-sdk/telegram-command-config` | Helpers de comando do Telegram | Helpers de validação de comando do Telegram estáveis em fallback quando a superfície de contrato empacotada do Telegram não está disponível |
  | `plugin-sdk/approval-runtime` | Helpers de prompt de aprovação | Payload de aprovação de execução/plugin, helpers de capacidade/perfil de aprovação, helpers nativos de roteamento/runtime de aprovação |
  | `plugin-sdk/approval-auth-runtime` | Helpers de autenticação de aprovação | Resolução de aprovador, autenticação de ação no mesmo chat |
  | `plugin-sdk/approval-client-runtime` | Helpers de cliente de aprovação | Helpers nativos de perfil/filtro de aprovação de execução |
  | `plugin-sdk/approval-delivery-runtime` | Helpers de entrega de aprovação | Adaptadores nativos de capacidade/entrega de aprovação |
  | `plugin-sdk/approval-native-runtime` | Helpers de alvo de aprovação | Helpers nativos de associação de alvo/conta para aprovação |
  | `plugin-sdk/approval-reply-runtime` | Helpers de resposta de aprovação | Helpers de payload de resposta de aprovação de execução/plugin |
  | `plugin-sdk/security-runtime` | Helpers de segurança | Helpers compartilhados de confiança, controle de DM, conteúdo externo e coleta de segredos |
  | `plugin-sdk/ssrf-policy` | Helpers de política de SSRF | Helpers de lista de permissões de host e política de rede privada |
  | `plugin-sdk/ssrf-runtime` | Helpers de runtime de SSRF | Helpers de pinned-dispatcher, fetch protegido e política de SSRF |
  | `plugin-sdk/collection-runtime` | Helpers de cache limitado | `pruneMapToMaxSize` |
  | `plugin-sdk/diagnostic-runtime` | Helpers de controle de diagnósticos | `isDiagnosticFlagEnabled`, `isDiagnosticsEnabled` |
  | `plugin-sdk/error-runtime` | Helpers de formatação de erro | `formatUncaughtError`, `isApprovalNotFoundError`, helpers de grafo de erros |
  | `plugin-sdk/fetch-runtime` | Helpers de fetch/proxy encapsulado | `resolveFetch`, helpers de proxy |
  | `plugin-sdk/host-runtime` | Helpers de normalização de host | `normalizeHostname`, `normalizeScpRemoteHost` |
  | `plugin-sdk/retry-runtime` | Helpers de retry | `RetryConfig`, `retryAsync`, executores de política |
  | `plugin-sdk/allow-from` | Formatação de lista de permissões | `formatAllowFromLowercase` |
  | `plugin-sdk/allowlist-resolution` | Mapeamento de entradas da lista de permissões | `mapAllowlistResolutionInputs` |
  | `plugin-sdk/command-auth` | Controle de comandos e helpers de superfície de comando | `resolveControlCommandGate`, helpers de autorização do remetente, helpers de registro de comandos |
  | `plugin-sdk/secret-input` | Parsing de entrada secreta | Helpers de entrada secreta |
  | `plugin-sdk/webhook-ingress` | Helpers de requisição de webhook | Utilitários de alvo de webhook |
  | `plugin-sdk/webhook-request-guards` | Helpers de proteção de corpo de webhook | Helpers de leitura/limite do corpo da requisição |
  | `plugin-sdk/reply-runtime` | Runtime compartilhado de resposta | Despacho de entrada, heartbeat, planejador de resposta, fragmentação |
  | `plugin-sdk/reply-dispatch-runtime` | Helpers específicos de despacho de resposta | Helpers de finalização + despacho para provider |
  | `plugin-sdk/reply-history` | Helpers de histórico de respostas | `buildHistoryContext`, `buildPendingHistoryContextFromMap`, `recordPendingHistoryEntry`, `clearHistoryEntriesIfEnabled` |
  | `plugin-sdk/reply-reference` | Planejamento de referência de resposta | `createReplyReferencePlanner` |
  | `plugin-sdk/reply-chunking` | Helpers de fragmentação de resposta | Helpers de fragmentação de texto/markdown |
  | `plugin-sdk/session-store-runtime` | Helpers de armazenamento de sessão | Caminho do armazenamento + helpers de updated-at |
  | `plugin-sdk/state-paths` | Helpers de caminhos de estado | Helpers de diretório de estado e OAuth |
  | `plugin-sdk/routing` | Helpers de roteamento/chave de sessão | `resolveAgentRoute`, `buildAgentSessionKey`, `resolveDefaultAgentBoundAccountId`, helpers de normalização de chave de sessão |
  | `plugin-sdk/status-helpers` | Helpers de status de canal | Builders de resumo de status de canal/conta, padrões de estado de runtime, helpers de metadados de problema |
  | `plugin-sdk/target-resolver-runtime` | Helpers de resolvedor de alvo | Helpers compartilhados de resolvedor de alvo |
  | `plugin-sdk/string-normalization-runtime` | Helpers de normalização de string | Helpers de normalização de slug/string |
  | `plugin-sdk/request-url` | Helpers de URL de requisição | Extrai URLs em string de entradas semelhantes a requisição |
  | `plugin-sdk/run-command` | Helpers de comando com tempo controlado | Executor de comando com stdout/stderr normalizados |
  | `plugin-sdk/param-readers` | Leitores de parâmetros | Leitores comuns de parâmetros de ferramenta/CLI |
  | `plugin-sdk/tool-send` | Extração de envio de ferramenta | Extrai campos canônicos de alvo de envio dos argumentos da ferramenta |
  | `plugin-sdk/temp-path` | Helpers de caminho temporário | Helpers compartilhados de caminho temporário para download |
  | `plugin-sdk/logging-core` | Helpers de logging | Logger de subsistema e helpers de redação |
  | `plugin-sdk/markdown-table-runtime` | Helpers de tabela em Markdown | Helpers de modo de tabela em Markdown |
  | `plugin-sdk/reply-payload` | Tipos de resposta de mensagem | Tipos de payload de resposta |
  | `plugin-sdk/provider-setup` | Helpers selecionados de configuração de providers locais/self-hosted | Helpers de descoberta/configuração de provider self-hosted |
  | `plugin-sdk/self-hosted-provider-setup` | Helpers específicos de configuração de providers self-hosted compatíveis com OpenAI | Os mesmos helpers de descoberta/configuração de provider self-hosted |
  | `plugin-sdk/provider-auth-runtime` | Helpers de autenticação de runtime de provider | Helpers de resolução de chave de API em runtime |
  | `plugin-sdk/provider-auth-api-key` | Helpers de configuração de chave de API do provider | Helpers de onboarding/gravação de perfil de chave de API |
  | `plugin-sdk/provider-auth-result` | Helpers de resultado de autenticação do provider | Builder padrão de resultado de autenticação OAuth |
  | `plugin-sdk/provider-auth-login` | Helpers de login interativo do provider | Helpers compartilhados de login interativo |
  | `plugin-sdk/provider-env-vars` | Helpers de variáveis de ambiente do provider | Helpers de consulta de variáveis de ambiente de autenticação do provider |
  | `plugin-sdk/provider-model-shared` | Helpers compartilhados de modelo/replay do provider | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, builders compartilhados de política de replay, helpers de endpoint do provider e helpers de normalização de ID de modelo |
  | `plugin-sdk/provider-catalog-shared` | Helpers compartilhados de catálogo do provider | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
  | `plugin-sdk/provider-onboard` | Patches de onboarding do provider | Helpers de configuração de onboarding |
  | `plugin-sdk/provider-http` | Helpers HTTP do provider | Helpers genéricos de HTTP/capacidade de endpoint do provider |
  | `plugin-sdk/provider-web-fetch` | Helpers de web-fetch do provider | Helpers de registro/cache de provider web-fetch |
  | `plugin-sdk/provider-web-search` | Helpers de web-search do provider | Helpers de registro/cache/configuração de provider web-search |
  | `plugin-sdk/provider-tools` | Helpers de compatibilidade de ferramenta/esquema do provider | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, limpeza de esquema Gemini + diagnósticos e helpers de compatibilidade xAI como `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
  | `plugin-sdk/provider-usage` | Helpers de uso do provider | `fetchClaudeUsage`, `fetchGeminiUsage`, `fetchGithubCopilotUsage` e outros helpers de uso do provider |
  | `plugin-sdk/provider-stream` | Helpers de wrapper de stream do provider | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, tipos de wrapper de stream e helpers compartilhados de wrapper para Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot |
  | `plugin-sdk/keyed-async-queue` | Fila assíncrona ordenada | `KeyedAsyncQueue` |
  | `plugin-sdk/media-runtime` | Helpers compartilhados de mídia | Helpers de fetch/transformação/armazenamento de mídia, além de builders de payload de mídia |
  | `plugin-sdk/media-generation-runtime` | Helpers compartilhados de geração de mídia | Helpers compartilhados de failover, seleção de candidatos e mensagens de modelo ausente para geração de imagem/vídeo/música |
  | `plugin-sdk/media-understanding` | Helpers de media-understanding | Tipos de provider de media-understanding, além de exports de helpers de imagem/áudio voltados para provider |
  | `plugin-sdk/text-runtime` | Helpers compartilhados de texto | Remoção de texto visível ao assistente, helpers de renderização/fragmentação/tabela em markdown, helpers de redação, helpers de tag de diretiva, utilitários de texto seguro e helpers relacionados de texto/logging |
  | `plugin-sdk/text-chunking` | Helpers de fragmentação de texto | Helper de fragmentação de texto de saída |
  | `plugin-sdk/speech` | Helpers de fala | Tipos de provider de fala, além de helpers de diretiva, registro e validação voltados para provider |
  | `plugin-sdk/speech-core` | Núcleo compartilhado de fala | Tipos de provider de fala, registro, diretivas, normalização |
  | `plugin-sdk/realtime-transcription` | Helpers de transcrição em tempo real | Tipos de provider e helpers de registro |
  | `plugin-sdk/realtime-voice` | Helpers de voz em tempo real | Tipos de provider e helpers de registro |
  | `plugin-sdk/image-generation-core` | Núcleo compartilhado de geração de imagem | Tipos de geração de imagem, failover, autenticação e helpers de registro |
  | `plugin-sdk/music-generation` | Helpers de geração de música | Tipos de provider/requisição/resultado de geração de música |
  | `plugin-sdk/music-generation-core` | Núcleo compartilhado de geração de música | Tipos de geração de música, helpers de failover, busca de provider e parsing de referência de modelo |
  | `plugin-sdk/video-generation` | Helpers de geração de vídeo | Tipos de provider/requisição/resultado de geração de vídeo |
  | `plugin-sdk/video-generation-core` | Núcleo compartilhado de geração de vídeo | Tipos de geração de vídeo, helpers de failover, busca de provider e parsing de referência de modelo |
  | `plugin-sdk/interactive-runtime` | Helpers de resposta interativa | Normalização/redução de payload de resposta interativa |
  | `plugin-sdk/channel-config-primitives` | Primitivos de configuração de canal | Primitivos específicos de esquema de configuração de canal |
  | `plugin-sdk/channel-config-writes` | Helpers de gravação de configuração de canal | Helpers de autorização de gravação de configuração de canal |
  | `plugin-sdk/channel-plugin-common` | Prelúdio compartilhado de canal | Exports compartilhados de prelúdio de plugin de canal |
  | `plugin-sdk/channel-status` | Helpers de status de canal | Helpers compartilhados de snapshot/resumo de status de canal |
  | `plugin-sdk/allowlist-config-edit` | Helpers de configuração de lista de permissões | Helpers de edição/leitura de configuração de lista de permissões |
  | `plugin-sdk/group-access` | Helpers de acesso de grupo | Helpers compartilhados de decisão de acesso de grupo |
  | `plugin-sdk/direct-dm` | Helpers de DM direto | Helpers compartilhados de autenticação/proteção de DM direto |
  | `plugin-sdk/extension-shared` | Helpers compartilhados de extensão | Primitivos helper de canal passivo/status |
  | `plugin-sdk/webhook-targets` | Helpers de alvo de webhook | Helpers de registro de alvo de webhook e instalação de rota |
  | `plugin-sdk/webhook-path` | Helpers de caminho de webhook | Helpers de normalização de caminho de webhook |
  | `plugin-sdk/web-media` | Helpers compartilhados de mídia web | Helpers de carregamento de mídia remota/local |
  | `plugin-sdk/zod` | Reexportação de Zod | `zod` reexportado para consumidores do SDK de plugins |
  | `plugin-sdk/memory-core` | Helpers empacotados de memory-core | Superfície helper de gerenciador de memória/configuração/arquivo/CLI |
  | `plugin-sdk/memory-core-engine-runtime` | Fachada de runtime do mecanismo de memória | Fachada de runtime de índice/busca de memória |
  | `plugin-sdk/memory-core-host-engine-foundation` | Mecanismo foundation do host de memória | Exports do mecanismo foundation do host de memória |
  | `plugin-sdk/memory-core-host-engine-embeddings` | Mecanismo de embeddings do host de memória | Exports do mecanismo de embeddings do host de memória |
  | `plugin-sdk/memory-core-host-engine-qmd` | Mecanismo QMD do host de memória | Exports do mecanismo QMD do host de memória |
  | `plugin-sdk/memory-core-host-engine-storage` | Mecanismo de armazenamento do host de memória | Exports do mecanismo de armazenamento do host de memória |
  | `plugin-sdk/memory-core-host-multimodal` | Helpers multimodais do host de memória | Helpers multimodais do host de memória |
  | `plugin-sdk/memory-core-host-query` | Helpers de consulta do host de memória | Helpers de consulta do host de memória |
  | `plugin-sdk/memory-core-host-secret` | Helpers de segredo do host de memória | Helpers de segredo do host de memória |
  | `plugin-sdk/memory-core-host-events` | Helpers de diário de eventos do host de memória | Helpers de diário de eventos do host de memória |
  | `plugin-sdk/memory-core-host-status` | Helpers de status do host de memória | Helpers de status do host de memória |
  | `plugin-sdk/memory-core-host-runtime-cli` | Runtime de CLI do host de memória | Helpers de runtime de CLI do host de memória |
  | `plugin-sdk/memory-core-host-runtime-core` | Runtime central do host de memória | Helpers de runtime central do host de memória |
  | `plugin-sdk/memory-core-host-runtime-files` | Helpers de arquivos/runtime do host de memória | Helpers de arquivos/runtime do host de memória |
  | `plugin-sdk/memory-host-core` | Alias de runtime central do host de memória | Alias neutro em relação ao fornecedor para helpers de runtime central do host de memória |
  | `plugin-sdk/memory-host-events` | Alias de diário de eventos do host de memória | Alias neutro em relação ao fornecedor para helpers de diário de eventos do host de memória |
  | `plugin-sdk/memory-host-files` | Alias de arquivos/runtime do host de memória | Alias neutro em relação ao fornecedor para helpers de arquivos/runtime do host de memória |
  | `plugin-sdk/memory-host-markdown` | Helpers de markdown gerenciado | Helpers compartilhados de markdown gerenciado para plugins adjacentes à memória |
  | `plugin-sdk/memory-host-search` | Fachada de busca de memória ativa | Fachada lazy de runtime do gerenciador de busca de memória ativa |
  | `plugin-sdk/memory-host-status` | Alias de status do host de memória | Alias neutro em relação ao fornecedor para helpers de status do host de memória |
  | `plugin-sdk/memory-lancedb` | Helpers empacotados de memory-lancedb | Superfície helper de memory-lancedb |
  | `plugin-sdk/testing` | Utilitários de teste | Helpers e mocks de teste |
</Accordion>

Esta tabela é intencionalmente o subconjunto comum de migração, não a superfície
completa do SDK. A lista completa dos mais de 200 pontos de entrada está em
`scripts/lib/plugin-sdk-entrypoints.json`.

Essa lista ainda inclui algumas superfícies helper de plugins empacotados, como
`plugin-sdk/feishu`, `plugin-sdk/feishu-setup`, `plugin-sdk/zalo`,
`plugin-sdk/zalo-setup` e `plugin-sdk/matrix*`. Elas continuam exportadas para
manutenção e compatibilidade de plugins empacotados, mas são intencionalmente
omitidas da tabela comum de migração e não são o destino recomendado para
novo código de plugin.

A mesma regra se aplica a outras famílias de helpers empacotados, como:

- helpers de suporte a navegador: `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support`
- Matrix: `plugin-sdk/matrix*`
- LINE: `plugin-sdk/line*`
- IRC: `plugin-sdk/irc*`
- superfícies helper/plugin empacotadas como `plugin-sdk/googlechat`,
  `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles*`,
  `plugin-sdk/mattermost*`, `plugin-sdk/msteams`,
  `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`,
  `plugin-sdk/twitch`,
  `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`,
  `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`,
  `plugin-sdk/thread-ownership` e `plugin-sdk/voice-call`

`plugin-sdk/github-copilot-token` atualmente expõe a superfície específica de
helper de token `DEFAULT_COPILOT_API_BASE_URL`,
`deriveCopilotApiBaseUrlFromToken` e `resolveCopilotApiToken`.

Use o import mais específico que corresponda ao trabalho. Se você não encontrar
um export, verifique a fonte em `src/plugin-sdk/` ou pergunte no Discord.

## Cronograma de remoção

| Quando                 | O que acontece                                                         |
| ---------------------- | ---------------------------------------------------------------------- |
| **Agora**              | Superfícies obsoletas emitem avisos em tempo de execução               |
| **Próxima versão principal** | Superfícies obsoletas serão removidas; plugins que ainda as usam falharão |

Todos os plugins centrais já foram migrados. Plugins externos devem migrar
antes da próxima versão principal.

## Suprimindo os avisos temporariamente

Defina estas variáveis de ambiente enquanto trabalha na migração:

```bash
OPENCLAW_SUPPRESS_PLUGIN_SDK_COMPAT_WARNING=1 openclaw gateway run
OPENCLAW_SUPPRESS_EXTENSION_API_WARNING=1 openclaw gateway run
```

Isto é uma válvula de escape temporária, não uma solução permanente.

## Relacionado

- [Primeiros passos](/pt-BR/plugins/building-plugins) — crie seu primeiro plugin
- [Visão geral do SDK](/pt-BR/plugins/sdk-overview) — referência completa de imports por subcaminho
- [Plugins de canal](/pt-BR/plugins/sdk-channel-plugins) — criação de plugins de canal
- [Plugins de provider](/pt-BR/plugins/sdk-provider-plugins) — criação de plugins de provider
- [Internals de plugin](/pt-BR/plugins/architecture) — visão aprofundada da arquitetura
- [Manifesto do plugin](/pt-BR/plugins/manifest) — referência do esquema do manifesto
