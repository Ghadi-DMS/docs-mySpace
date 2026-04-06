---
read_when:
    - Você precisa saber de qual subcaminho do SDK importar
    - Você quer uma referência para todos os métodos de registro em OpenClawPluginApi
    - Você está procurando uma exportação específica do SDK
sidebarTitle: SDK Overview
summary: Mapa de importação, referência da API de registro e arquitetura do SDK
title: Visão geral do SDK de plugins
x-i18n:
    generated_at: "2026-04-06T05:35:59Z"
    model: gpt-5.4
    provider: openai
    source_hash: acd2887ef52c66b2f234858d812bb04197ecd0bfb3e4f7bf3622f8fdc765acad
    source_path: plugins/sdk-overview.md
    workflow: 15
---

# Visão geral do SDK de plugins

O SDK de plugins é o contrato tipado entre plugins e o núcleo. Esta página é a
referência para **o que importar** e **o que você pode registrar**.

<Tip>
  **Está procurando um guia prático?**
  - Primeiro plugin? Comece com [Primeiros passos](/pt-BR/plugins/building-plugins)
  - Plugin de canal? Consulte [Plugins de canal](/pt-BR/plugins/sdk-channel-plugins)
  - Plugin de provedor? Consulte [Plugins de provedor](/pt-BR/plugins/sdk-provider-plugins)
</Tip>

## Convenção de importação

Sempre importe de um subcaminho específico:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
```

Cada subcaminho é um módulo pequeno e autocontido. Isso mantém a inicialização
rápida e evita problemas de dependência circular. Para auxiliares de
entrada/build específicos de canal, prefira `openclaw/plugin-sdk/channel-core`;
mantenha `openclaw/plugin-sdk/core` para a superfície guarda-chuva mais ampla e
auxiliares compartilhados como `buildChannelConfigSchema`.

Não adicione nem dependa de superfícies de conveniência nomeadas por provedor,
como `openclaw/plugin-sdk/slack`, `openclaw/plugin-sdk/discord`,
`openclaw/plugin-sdk/signal`, `openclaw/plugin-sdk/whatsapp` ou
superfícies auxiliares com marca de canal. Plugins empacotados devem compor
subcaminhos genéricos do SDK dentro de seus próprios barrels `api.ts` ou
`runtime-api.ts`, e o núcleo deve usar esses barrels locais do plugin ou
adicionar um contrato de SDK genérico e estreito quando a necessidade for
realmente entre canais.

O mapa de exportação gerado ainda contém um pequeno conjunto de
superfícies auxiliares de plugins empacotados, como `plugin-sdk/feishu`,
`plugin-sdk/feishu-setup`, `plugin-sdk/zalo`, `plugin-sdk/zalo-setup` e
`plugin-sdk/matrix*`. Esses subcaminhos existem apenas para manutenção e
compatibilidade de plugins empacotados; eles são intencionalmente omitidos da
tabela comum abaixo e não são o caminho de importação recomendado para novos
plugins de terceiros.

## Referência de subcaminhos

Os subcaminhos mais usados, agrupados por finalidade. A lista completa gerada
de mais de 200 subcaminhos fica em `scripts/lib/plugin-sdk-entrypoints.json`.

Os subcaminhos auxiliares reservados para plugins empacotados ainda aparecem
nessa lista gerada. Trate-os como detalhes de implementação/superfícies de
compatibilidade, a menos que uma página da documentação promova explicitamente
algum deles como público.

### Entrada do plugin

| Subcaminho                 | Principais exportações                                                                                                                |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/plugin-entry`  | `definePluginEntry`                                                                                                                   |
| `plugin-sdk/core`          | `defineChannelPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase`, `defineSetupPluginEntry`, `buildChannelConfigSchema` |
| `plugin-sdk/config-schema` | `OpenClawSchema`                                                                                                                      |
| `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry`                                                                                                     |

<AccordionGroup>
  <Accordion title="Subcaminhos de canal">
    | Subcaminho | Principais exportações |
    | --- | --- |
    | `plugin-sdk/channel-core` | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
    | `plugin-sdk/config-schema` | Exportação do schema Zod raiz de `openclaw.json` (`OpenClawSchema`) |
    | `plugin-sdk/channel-setup` | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, além de `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
    | `plugin-sdk/setup` | Auxiliares compartilhados do assistente de configuração, prompts de allowlist, construtores de status de configuração |
    | `plugin-sdk/setup-runtime` | `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
    | `plugin-sdk/setup-adapter-runtime` | `createEnvPatchedAccountSetupAdapter` |
    | `plugin-sdk/setup-tools` | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
    | `plugin-sdk/account-core` | Auxiliares de configuração/gatilho de ação para múltiplas contas, auxiliares de fallback para conta padrão |
    | `plugin-sdk/account-id` | `DEFAULT_ACCOUNT_ID`, auxiliares de normalização de id de conta |
    | `plugin-sdk/account-resolution` | Auxiliares de busca de conta + fallback para padrão |
    | `plugin-sdk/account-helpers` | Auxiliares estreitos de lista de contas/ação em conta |
    | `plugin-sdk/channel-pairing` | `createChannelPairingController` |
    | `plugin-sdk/channel-reply-pipeline` | `createChannelReplyPipeline` |
    | `plugin-sdk/channel-config-helpers` | `createHybridChannelConfigAdapter` |
    | `plugin-sdk/channel-config-schema` | Tipos de schema de configuração de canal |
    | `plugin-sdk/telegram-command-config` | Auxiliares de normalização/validação de comandos personalizados do Telegram com fallback de contrato empacotado |
    | `plugin-sdk/channel-policy` | `resolveChannelGroupRequireMention` |
    | `plugin-sdk/channel-lifecycle` | `createAccountStatusSink` |
    | `plugin-sdk/inbound-envelope` | Auxiliares compartilhados de rota de entrada + construtor de envelope |
    | `plugin-sdk/inbound-reply-dispatch` | Auxiliares compartilhados para registrar e despachar entrada |
    | `plugin-sdk/messaging-targets` | Auxiliares de análise/correspondência de alvos |
    | `plugin-sdk/outbound-media` | Auxiliares compartilhados de carregamento de mídia de saída |
    | `plugin-sdk/outbound-runtime` | Auxiliares de identidade de saída/delegação de envio |
    | `plugin-sdk/thread-bindings-runtime` | Auxiliares de ciclo de vida e adaptador de vinculações de thread |
    | `plugin-sdk/agent-media-payload` | Construtor legado de payload de mídia do agente |
    | `plugin-sdk/conversation-runtime` | Auxiliares de vinculação de conversa/thread, pareamento e vinculação configurada |
    | `plugin-sdk/runtime-config-snapshot` | Auxiliar de snapshot da configuração de runtime |
    | `plugin-sdk/runtime-group-policy` | Auxiliares de resolução de política de grupo em runtime |
    | `plugin-sdk/channel-status` | Auxiliares compartilhados de snapshot/resumo de status de canal |
    | `plugin-sdk/channel-config-primitives` | Primitivas estreitas de schema de configuração de canal |
    | `plugin-sdk/channel-config-writes` | Auxiliares de autorização de gravação de configuração de canal |
    | `plugin-sdk/channel-plugin-common` | Exportações de prelúdio compartilhadas para plugins de canal |
    | `plugin-sdk/allowlist-config-edit` | Auxiliares de edição/leitura de configuração de allowlist |
    | `plugin-sdk/group-access` | Auxiliares compartilhados de decisão de acesso a grupos |
    | `plugin-sdk/direct-dm` | Auxiliares compartilhados de autenticação/proteção para DM direta |
    | `plugin-sdk/interactive-runtime` | Auxiliares de normalização/redução de payload de resposta interativa |
    | `plugin-sdk/channel-inbound` | Auxiliares de debounce, correspondência de menção e envelope |
    | `plugin-sdk/channel-send-result` | Tipos de resultado de resposta |
    | `plugin-sdk/channel-actions` | `createMessageToolButtonsSchema`, `createMessageToolCardSchema` |
    | `plugin-sdk/channel-targets` | Auxiliares de análise/correspondência de alvos |
    | `plugin-sdk/channel-contract` | Tipos de contrato de canal |
    | `plugin-sdk/channel-feedback` | Integração de feedback/reação |
  </Accordion>

  <Accordion title="Subcaminhos de provedor">
    | Subcaminho | Principais exportações |
    | --- | --- |
    | `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry` |
    | `plugin-sdk/provider-setup` | Auxiliares curados de configuração para provedores locais/self-hosted |
    | `plugin-sdk/self-hosted-provider-setup` | Auxiliares focados de configuração para provedor self-hosted compatível com OpenAI |
    | `plugin-sdk/provider-auth-runtime` | Auxiliares de resolução de chave de API em runtime para plugins de provedor |
    | `plugin-sdk/provider-auth-api-key` | Auxiliares de onboarding/gravação de perfil de chave de API |
    | `plugin-sdk/provider-auth-result` | Construtor padrão de resultado de autenticação OAuth |
    | `plugin-sdk/provider-auth-login` | Auxiliares compartilhados de login interativo para plugins de provedor |
    | `plugin-sdk/provider-env-vars` | Auxiliares de busca de variáveis de ambiente de autenticação de provedor |
    | `plugin-sdk/provider-auth` | `createProviderApiKeyAuthMethod`, `ensureApiKeyFromOptionEnvOrPrompt`, `upsertAuthProfile` |
    | `plugin-sdk/provider-model-shared` | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, construtores compartilhados de política de replay, auxiliares de endpoint de provedor e auxiliares de normalização de id de modelo, como `normalizeNativeXaiModelId` |
    | `plugin-sdk/provider-catalog-shared` | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
    | `plugin-sdk/provider-http` | Auxiliares genéricos de HTTP/capacidade de endpoint de provedor |
    | `plugin-sdk/provider-web-fetch` | Auxiliares de registro/cache de provedor de web fetch |
    | `plugin-sdk/provider-web-search` | Auxiliares de registro/cache/configuração de provedor de web search |
    | `plugin-sdk/provider-tools` | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, limpeza + diagnósticos de schema do Gemini e auxiliares de compatibilidade xAI, como `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
    | `plugin-sdk/provider-usage` | `fetchClaudeUsage` e similares |
    | `plugin-sdk/provider-stream` | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, tipos de wrapper de stream e auxiliares compartilhados de wrapper para Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot |
    | `plugin-sdk/provider-onboard` | Auxiliares de patch de configuração de onboarding |
    | `plugin-sdk/global-singleton` | Auxiliares de singleton/mapa/cache local ao processo |
  </Accordion>

  <Accordion title="Subcaminhos de autenticação e segurança">
    | Subcaminho | Principais exportações |
    | --- | --- |
    | `plugin-sdk/command-auth` | `resolveControlCommandGate`, auxiliares de registro de comandos, auxiliares de autorização de remetente |
    | `plugin-sdk/approval-auth-runtime` | Auxiliares de resolução de aprovador e autenticação de ação no mesmo chat |
    | `plugin-sdk/approval-client-runtime` | Auxiliares de perfil/filtro de aprovação para execução nativa |
    | `plugin-sdk/approval-delivery-runtime` | Adaptadores de capacidade/entrega de aprovação nativa |
    | `plugin-sdk/approval-native-runtime` | Auxiliares de alvo de aprovação nativa + vinculação de conta |
    | `plugin-sdk/approval-reply-runtime` | Auxiliares de payload de resposta para aprovação de execução/plugin |
    | `plugin-sdk/command-auth-native` | Auxiliares de autenticação de comando nativo + alvo de sessão nativa |
    | `plugin-sdk/command-detection` | Auxiliares compartilhados de detecção de comando |
    | `plugin-sdk/command-surface` | Auxiliares de normalização do corpo do comando e superfície de comando |
    | `plugin-sdk/allow-from` | `formatAllowFromLowercase` |
    | `plugin-sdk/security-runtime` | Auxiliares compartilhados de confiança, restrição de DM, conteúdo externo e coleta de segredo |
    | `plugin-sdk/ssrf-policy` | Auxiliares de allowlist de host e política SSRF para rede privada |
    | `plugin-sdk/ssrf-runtime` | Auxiliares de dispatcher fixado, fetch protegido por SSRF e política SSRF |
    | `plugin-sdk/secret-input` | Auxiliares de análise de entrada de segredo |
    | `plugin-sdk/webhook-ingress` | Auxiliares de requisição/alvo de webhook |
    | `plugin-sdk/webhook-request-guards` | Auxiliares de tamanho do corpo/timeout de requisição |
  </Accordion>

  <Accordion title="Subcaminhos de runtime e armazenamento">
    | Subcaminho | Principais exportações |
    | --- | --- |
    | `plugin-sdk/runtime` | Auxiliares amplos de runtime/log/backup/instalação de plugin |
    | `plugin-sdk/runtime-env` | Auxiliares estreitos de ambiente de runtime, logger, timeout, retry e backoff |
    | `plugin-sdk/runtime-store` | `createPluginRuntimeStore` |
    | `plugin-sdk/plugin-runtime` | Auxiliares compartilhados de comando/hook/http/interação de plugin |
    | `plugin-sdk/hook-runtime` | Auxiliares compartilhados de pipeline para webhooks/hooks internos |
    | `plugin-sdk/lazy-runtime` | Auxiliares de importação/vinculação lazy em runtime, como `createLazyRuntimeModule`, `createLazyRuntimeMethod` e `createLazyRuntimeSurface` |
    | `plugin-sdk/process-runtime` | Auxiliares de execução de processo |
    | `plugin-sdk/cli-runtime` | Auxiliares de formatação, espera e versão da CLI |
    | `plugin-sdk/gateway-runtime` | Auxiliares de cliente do gateway e patch de status de canal |
    | `plugin-sdk/config-runtime` | Auxiliares de carregamento/gravação de configuração |
    | `plugin-sdk/telegram-command-config` | Normalização de nome/descrição de comando do Telegram e verificações de duplicidade/conflito, mesmo quando a superfície de contrato empacotada do Telegram não está disponível |
    | `plugin-sdk/approval-runtime` | Auxiliares de aprovação de execução/plugin, construtores de capacidade de aprovação, auxiliares de autenticação/perfil, auxiliares nativos de roteamento/runtime |
    | `plugin-sdk/reply-runtime` | Auxiliares compartilhados de runtime para entrada/resposta, fragmentação, despacho, heartbeat e planejador de resposta |
    | `plugin-sdk/reply-dispatch-runtime` | Auxiliares estreitos de despacho/finalização de resposta |
    | `plugin-sdk/reply-history` | Auxiliares compartilhados de histórico de resposta em janela curta, como `buildHistoryContext`, `recordPendingHistoryEntry` e `clearHistoryEntriesIfEnabled` |
    | `plugin-sdk/reply-reference` | `createReplyReferencePlanner` |
    | `plugin-sdk/reply-chunking` | Auxiliares estreitos de fragmentação de texto/markdown |
    | `plugin-sdk/session-store-runtime` | Auxiliares de caminho da store de sessão + updated-at |
    | `plugin-sdk/state-paths` | Auxiliares de caminho de diretório de estado/OAuth |
    | `plugin-sdk/routing` | Auxiliares de roteamento/chave de sessão/vinculação de conta, como `resolveAgentRoute`, `buildAgentSessionKey` e `resolveDefaultAgentBoundAccountId` |
    | `plugin-sdk/status-helpers` | Auxiliares compartilhados de resumo de status de canal/conta, padrões de estado de runtime e auxiliares de metadados de problema |
    | `plugin-sdk/target-resolver-runtime` | Auxiliares compartilhados de resolução de alvo |
    | `plugin-sdk/string-normalization-runtime` | Auxiliares de normalização de slug/string |
    | `plugin-sdk/request-url` | Extrair URLs em string de entradas do tipo fetch/request |
    | `plugin-sdk/run-command` | Executor de comando com tempo medido e resultados stdout/stderr normalizados |
    | `plugin-sdk/param-readers` | Leitores comuns de parâmetros de ferramenta/CLI |
    | `plugin-sdk/tool-send` | Extrair campos canônicos de alvo de envio de argumentos de ferramenta |
    | `plugin-sdk/temp-path` | Auxiliares compartilhados de caminho temporário para download |
    | `plugin-sdk/logging-core` | Logger de subsistema e auxiliares de redação |
    | `plugin-sdk/markdown-table-runtime` | Auxiliares de modo de tabela Markdown |
    | `plugin-sdk/json-store` | Pequenos auxiliares de leitura/gravação de estado JSON |
    | `plugin-sdk/file-lock` | Auxiliares de bloqueio de arquivo reentrante |
    | `plugin-sdk/persistent-dedupe` | Auxiliares de cache de deduplicação em disco |
    | `plugin-sdk/acp-runtime` | Auxiliares de runtime/sessão ACP e despacho de resposta |
    | `plugin-sdk/agent-config-primitives` | Primitivas estreitas de schema de configuração de runtime do agente |
    | `plugin-sdk/boolean-param` | Leitor flexível de parâmetro booleano |
    | `plugin-sdk/dangerous-name-runtime` | Auxiliares de resolução de correspondência de nomes perigosos |
    | `plugin-sdk/device-bootstrap` | Auxiliares de bootstrap de dispositivo e token de pareamento |
    | `plugin-sdk/extension-shared` | Primitivas compartilhadas de canal passivo e auxiliares de status |
    | `plugin-sdk/models-provider-runtime` | Auxiliares de resposta para comando `/models`/provedor |
    | `plugin-sdk/skill-commands-runtime` | Auxiliares de listagem de comandos de Skills |
    | `plugin-sdk/native-command-registry` | Auxiliares de registro/build/serialização de comando nativo |
    | `plugin-sdk/provider-zai-endpoint` | Auxiliares de detecção de endpoint Z.AI |
    | `plugin-sdk/infra-runtime` | Auxiliares de evento do sistema/heartbeat |
    | `plugin-sdk/collection-runtime` | Pequenos auxiliares de cache limitado |
    | `plugin-sdk/diagnostic-runtime` | Auxiliares de flag e evento de diagnóstico |
    | `plugin-sdk/error-runtime` | Grafo de erro, formatação, auxiliares compartilhados de classificação de erro, `isApprovalNotFoundError` |
    | `plugin-sdk/fetch-runtime` | Auxiliares de fetch encapsulado, proxy e busca fixada |
    | `plugin-sdk/host-runtime` | Auxiliares de normalização de hostname e host SCP |
    | `plugin-sdk/retry-runtime` | Auxiliares de configuração e executor de retry |
    | `plugin-sdk/agent-runtime` | Auxiliares de diretório/identidade/workspace do agente |
    | `plugin-sdk/directory-runtime` | Consulta/deduplicação de diretório baseada em configuração |
    | `plugin-sdk/keyed-async-queue` | `KeyedAsyncQueue` |
  </Accordion>

  <Accordion title="Subcaminhos de capacidade e teste">
    | Subcaminho | Principais exportações |
    | --- | --- |
    | `plugin-sdk/media-runtime` | Auxiliares compartilhados de busca/transformação/armazenamento de mídia, além de construtores de payload de mídia |
    | `plugin-sdk/media-generation-runtime` | Auxiliares compartilhados de failover para geração de mídia, seleção de candidatos e mensagens de modelo ausente |
    | `plugin-sdk/media-understanding` | Tipos de provedor de entendimento de mídia, além de exportações de auxiliares de imagem/áudio voltadas a provedores |
    | `plugin-sdk/text-runtime` | Auxiliares compartilhados de texto/markdown/log, como remoção de texto visível ao assistente, auxiliares de renderização/fragmentação/tabela em markdown, auxiliares de redação, auxiliares de tag de diretiva e utilitários de texto seguro |
    | `plugin-sdk/text-chunking` | Auxiliar de fragmentação de texto de saída |
    | `plugin-sdk/speech` | Tipos de provedor de fala, além de auxiliares de diretiva, registro e validação voltados a provedores |
    | `plugin-sdk/speech-core` | Tipos compartilhados de provedor de fala, registro, diretiva e auxiliares de normalização |
    | `plugin-sdk/realtime-transcription` | Tipos de provedor de transcrição em tempo real e auxiliares de registro |
    | `plugin-sdk/realtime-voice` | Tipos de provedor de voz em tempo real e auxiliares de registro |
    | `plugin-sdk/image-generation` | Tipos de provedor de geração de imagem |
    | `plugin-sdk/image-generation-core` | Tipos compartilhados de geração de imagem, failover, autenticação e auxiliares de registro |
    | `plugin-sdk/music-generation` | Tipos de provedor/requisição/resultado para geração de música |
    | `plugin-sdk/music-generation-core` | Tipos compartilhados de geração de música, auxiliares de failover, busca de provedor e análise de model-ref |
    | `plugin-sdk/video-generation` | Tipos de provedor/requisição/resultado para geração de vídeo |
    | `plugin-sdk/video-generation-core` | Tipos compartilhados de geração de vídeo, auxiliares de failover, busca de provedor e análise de model-ref |
    | `plugin-sdk/webhook-targets` | Registro de alvos de webhook e auxiliares de instalação de rota |
    | `plugin-sdk/webhook-path` | Auxiliares de normalização de caminho de webhook |
    | `plugin-sdk/web-media` | Auxiliares compartilhados de carregamento de mídia remota/local |
    | `plugin-sdk/zod` | `zod` reexportado para consumidores do SDK de plugins |
    | `plugin-sdk/testing` | `installCommonResolveTargetErrorCases`, `shouldAckReaction` |
  </Accordion>

  <Accordion title="Subcaminhos de memória">
    | Subcaminho | Principais exportações |
    | --- | --- |
    | `plugin-sdk/memory-core` | Superfície auxiliar empacotada de memory-core para auxiliares de gerente/configuração/arquivo/CLI |
    | `plugin-sdk/memory-core-engine-runtime` | Fachada de runtime para índice/busca de memória |
    | `plugin-sdk/memory-core-host-engine-foundation` | Exportações do engine foundation do host de memória |
    | `plugin-sdk/memory-core-host-engine-embeddings` | Exportações do engine de embeddings do host de memória |
    | `plugin-sdk/memory-core-host-engine-qmd` | Exportações do engine QMD do host de memória |
    | `plugin-sdk/memory-core-host-engine-storage` | Exportações do engine de armazenamento do host de memória |
    | `plugin-sdk/memory-core-host-multimodal` | Auxiliares multimodais do host de memória |
    | `plugin-sdk/memory-core-host-query` | Auxiliares de consulta do host de memória |
    | `plugin-sdk/memory-core-host-secret` | Auxiliares de segredo do host de memória |
    | `plugin-sdk/memory-core-host-events` | Auxiliares de diário de eventos do host de memória |
    | `plugin-sdk/memory-core-host-status` | Auxiliares de status do host de memória |
    | `plugin-sdk/memory-core-host-runtime-cli` | Auxiliares de runtime CLI do host de memória |
    | `plugin-sdk/memory-core-host-runtime-core` | Auxiliares centrais de runtime do host de memória |
    | `plugin-sdk/memory-core-host-runtime-files` | Auxiliares de arquivo/runtime do host de memória |
    | `plugin-sdk/memory-host-core` | Alias neutro em relação ao fornecedor para auxiliares centrais de runtime do host de memória |
    | `plugin-sdk/memory-host-events` | Alias neutro em relação ao fornecedor para auxiliares do diário de eventos do host de memória |
    | `plugin-sdk/memory-host-files` | Alias neutro em relação ao fornecedor para auxiliares de arquivo/runtime do host de memória |
    | `plugin-sdk/memory-host-markdown` | Auxiliares compartilhados de markdown gerenciado para plugins adjacentes à memória |
    | `plugin-sdk/memory-host-search` | Fachada de runtime de memória ativa para acesso ao search-manager |
    | `plugin-sdk/memory-host-status` | Alias neutro em relação ao fornecedor para auxiliares de status do host de memória |
    | `plugin-sdk/memory-lancedb` | Superfície auxiliar empacotada de memory-lancedb |
  </Accordion>

  <Accordion title="Subcaminhos auxiliares empacotados reservados">
    | Família | Subcaminhos atuais | Uso pretendido |
    | --- | --- | --- |
    | Browser | `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support` | Auxiliares de suporte para o plugin empacotado de browser (`browser-support` continua sendo o barrel de compatibilidade) |
    | Matrix | `plugin-sdk/matrix`, `plugin-sdk/matrix-helper`, `plugin-sdk/matrix-runtime-heavy`, `plugin-sdk/matrix-runtime-shared`, `plugin-sdk/matrix-runtime-surface`, `plugin-sdk/matrix-surface`, `plugin-sdk/matrix-thread-bindings` | Superfície auxiliar/runtime empacotada do Matrix |
    | Line | `plugin-sdk/line`, `plugin-sdk/line-core`, `plugin-sdk/line-runtime`, `plugin-sdk/line-surface` | Superfície auxiliar/runtime empacotada do LINE |
    | IRC | `plugin-sdk/irc`, `plugin-sdk/irc-surface` | Superfície auxiliar empacotada do IRC |
    | Auxiliares específicos de canal | `plugin-sdk/googlechat`, `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles`, `plugin-sdk/bluebubbles-policy`, `plugin-sdk/mattermost`, `plugin-sdk/mattermost-policy`, `plugin-sdk/feishu-conversation`, `plugin-sdk/msteams`, `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`, `plugin-sdk/twitch` | Superfícies de compatibilidade/auxiliares de canais empacotados |
    | Auxiliares específicos de autenticação/plugin | `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`, `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`, `plugin-sdk/thread-ownership`, `plugin-sdk/voice-call` | Superfícies auxiliares de recursos/plugins empacotados; `plugin-sdk/github-copilot-token` atualmente exporta `DEFAULT_COPILOT_API_BASE_URL`, `deriveCopilotApiBaseUrlFromToken` e `resolveCopilotApiToken` |
  </Accordion>
</AccordionGroup>

## API de registro

O callback `register(api)` recebe um objeto `OpenClawPluginApi` com estes
métodos:

### Registro de capacidades

| Método                                           | O que registra                    |
| ------------------------------------------------ | --------------------------------- |
| `api.registerProvider(...)`                      | Inferência de texto (LLM)         |
| `api.registerChannel(...)`                       | Canal de mensagens                |
| `api.registerSpeechProvider(...)`                | Síntese de texto para fala / STT  |
| `api.registerRealtimeTranscriptionProvider(...)` | Transcrição em tempo real por streaming |
| `api.registerRealtimeVoiceProvider(...)`         | Sessões de voz em tempo real duplex |
| `api.registerMediaUnderstandingProvider(...)`    | Análise de imagem/áudio/vídeo     |
| `api.registerImageGenerationProvider(...)`       | Geração de imagem                 |
| `api.registerMusicGenerationProvider(...)`       | Geração de música                 |
| `api.registerVideoGenerationProvider(...)`       | Geração de vídeo                  |
| `api.registerWebFetchProvider(...)`              | Provedor de web fetch / scrape    |
| `api.registerWebSearchProvider(...)`             | Web search                        |

### Ferramentas e comandos

| Método                          | O que registra                               |
| ------------------------------- | -------------------------------------------- |
| `api.registerTool(tool, opts?)` | Ferramenta do agente (obrigatória ou `{ optional: true }`) |
| `api.registerCommand(def)`      | Comando personalizado (contorna o LLM)       |

### Infraestrutura

| Método                                         | O que registra                          |
| ---------------------------------------------- | --------------------------------------- |
| `api.registerHook(events, handler, opts?)`     | Hook de evento                          |
| `api.registerHttpRoute(params)`                | Endpoint HTTP do gateway                |
| `api.registerGatewayMethod(name, handler)`     | Método RPC do gateway                   |
| `api.registerCli(registrar, opts?)`            | Subcomando da CLI                       |
| `api.registerService(service)`                 | Serviço em segundo plano                |
| `api.registerInteractiveHandler(registration)` | Manipulador interativo                  |
| `api.registerMemoryPromptSupplement(builder)`  | Seção adicional de prompt adjacente à memória |
| `api.registerMemoryCorpusSupplement(adapter)`  | Corpus adicional de busca/leitura de memória |

Os namespaces administrativos reservados do núcleo (`config.*`,
`exec.approvals.*`, `wizard.*`, `update.*`) sempre permanecem como `operator.admin`,
mesmo se um plugin tentar atribuir um escopo mais restrito ao método do
gateway. Prefira prefixos específicos do plugin para métodos pertencentes ao
plugin.

### Metadados de registro da CLI

`api.registerCli(registrar, opts?)` aceita dois tipos de metadados de nível superior:

- `commands`: raízes de comando explícitas pertencentes ao registrador
- `descriptors`: descritores de comando em tempo de parsing usados para ajuda
  da CLI raiz, roteamento e registro lazy da CLI do plugin

Se você quiser que um comando do plugin permaneça carregado de forma lazy no
caminho normal da CLI raiz, forneça `descriptors` que cubram toda raiz de
comando de nível superior exposta por esse registrador.

```typescript
api.registerCli(
  async ({ program }) => {
    const { registerMatrixCli } = await import("./src/cli.js");
    registerMatrixCli({ program });
  },
  {
    descriptors: [
      {
        name: "matrix",
        description: "Gerencie contas Matrix, verificação, dispositivos e estado de perfil",
        hasSubcommands: true,
      },
    ],
  },
);
```

Use `commands` sozinho apenas quando você não precisar de registro lazy na CLI
raiz. Esse caminho de compatibilidade eager continua com suporte, mas não
instala placeholders com suporte a descritores para carregamento lazy em tempo
de parsing.

### Slots exclusivos

| Método                                     | O que registra                       |
| ------------------------------------------ | ------------------------------------ |
| `api.registerContextEngine(id, factory)`   | Engine de contexto (um ativo por vez) |
| `api.registerMemoryPromptSection(builder)` | Construtor de seção de prompt de memória |
| `api.registerMemoryFlushPlan(resolver)`    | Resolver de plano de flush de memória |
| `api.registerMemoryRuntime(runtime)`       | Adaptador de runtime de memória      |

### Adaptadores de embedding de memória

| Método                                         | O que registra                                   |
| ---------------------------------------------- | ------------------------------------------------ |
| `api.registerMemoryEmbeddingProvider(adapter)` | Adaptador de embedding de memória para o plugin ativo |

- `registerMemoryPromptSection`, `registerMemoryFlushPlan` e
  `registerMemoryRuntime` são exclusivos de plugins de memória.
- `registerMemoryEmbeddingProvider` permite que o plugin de memória ativo
  registre um ou mais ids de adaptador de embedding (por exemplo `openai`,
  `gemini` ou um id personalizado definido pelo plugin).
- A configuração do usuário, como `agents.defaults.memorySearch.provider` e
  `agents.defaults.memorySearch.fallback`, é resolvida com base nesses ids de
  adaptador registrados.

### Eventos e ciclo de vida

| Método                                       | O que faz                   |
| -------------------------------------------- | --------------------------- |
| `api.on(hookName, handler, opts?)`           | Hook tipado de ciclo de vida |
| `api.onConversationBindingResolved(handler)` | Callback de vinculação de conversa |

### Semântica de decisão de hooks

- `before_tool_call`: retornar `{ block: true }` é terminal. Assim que qualquer manipulador o definir, os manipuladores de menor prioridade serão ignorados.
- `before_tool_call`: retornar `{ block: false }` é tratado como nenhuma decisão (o mesmo que omitir `block`), não como substituição.
- `before_install`: retornar `{ block: true }` é terminal. Assim que qualquer manipulador o definir, os manipuladores de menor prioridade serão ignorados.
- `before_install`: retornar `{ block: false }` é tratado como nenhuma decisão (o mesmo que omitir `block`), não como substituição.
- `reply_dispatch`: retornar `{ handled: true, ... }` é terminal. Assim que qualquer manipulador reivindicar o despacho, os manipuladores de menor prioridade e o caminho padrão de despacho do modelo serão ignorados.
- `message_sending`: retornar `{ cancel: true }` é terminal. Assim que qualquer manipulador o definir, os manipuladores de menor prioridade serão ignorados.
- `message_sending`: retornar `{ cancel: false }` é tratado como nenhuma decisão (o mesmo que omitir `cancel`), não como substituição.

### Campos do objeto API

| Campo                    | Tipo                      | Descrição                                                                                   |
| ------------------------ | ------------------------- | ------------------------------------------------------------------------------------------- |
| `api.id`                 | `string`                  | Id do plugin                                                                                |
| `api.name`               | `string`                  | Nome de exibição                                                                            |
| `api.version`            | `string?`                 | Versão do plugin (opcional)                                                                 |
| `api.description`        | `string?`                 | Descrição do plugin (opcional)                                                              |
| `api.source`             | `string`                  | Caminho de origem do plugin                                                                 |
| `api.rootDir`            | `string?`                 | Diretório raiz do plugin (opcional)                                                         |
| `api.config`             | `OpenClawConfig`          | Snapshot atual da configuração (snapshot ativo em memória do runtime quando disponível)     |
| `api.pluginConfig`       | `Record<string, unknown>` | Configuração específica do plugin de `plugins.entries.<id>.config`                          |
| `api.runtime`            | `PluginRuntime`           | [Auxiliares de runtime](/pt-BR/plugins/sdk-runtime)                                               |
| `api.logger`             | `PluginLogger`            | Logger com escopo (`debug`, `info`, `warn`, `error`)                                        |
| `api.registrationMode`   | `PluginRegistrationMode`  | Modo de carregamento atual; `"setup-runtime"` é a janela leve de inicialização/configuração antes da entrada completa |
| `api.resolvePath(input)` | `(string) => string`      | Resolver caminho relativo à raiz do plugin                                                  |

## Convenção de módulos internos

Dentro do seu plugin, use arquivos barrel locais para importações internas:

```
my-plugin/
  api.ts            # Exportações públicas para consumidores externos
  runtime-api.ts    # Exportações internas apenas para runtime
  index.ts          # Ponto de entrada do plugin
  setup-entry.ts    # Entrada leve apenas para configuração (opcional)
```

<Warning>
  Nunca importe seu próprio plugin por meio de `openclaw/plugin-sdk/<your-plugin>`
  no código de produção. Encaminhe as importações internas por `./api.ts` ou
  `./runtime-api.ts`. O caminho do SDK é apenas o contrato externo.
</Warning>

As superfícies públicas de plugins empacotados carregadas por fachada (`api.ts`,
`runtime-api.ts`, `index.ts`, `setup-entry.ts` e arquivos de entrada pública
semelhantes) agora preferem o snapshot ativo da configuração de runtime quando
o OpenClaw já está em execução. Se ainda não existir um snapshot de runtime,
elas usam como fallback o arquivo de configuração resolvido em disco.

Plugins de provedor também podem expor um barrel de contrato local ao plugin e
mais estreito quando um auxiliar for intencionalmente específico do provedor e
ainda não pertencer a um subcaminho genérico do SDK. Exemplo empacotado atual:
o provedor Anthropic mantém seus auxiliares de stream Claude em sua própria
superfície pública `api.ts` / `contract-api.ts`, em vez de promover a lógica de
cabeçalho beta do Anthropic e `service_tier` a um contrato genérico
`plugin-sdk/*`.

Outros exemplos empacotados atuais:

- `@openclaw/openai-provider`: `api.ts` exporta construtores de provedor,
  auxiliares de modelo padrão e construtores de provedor em tempo real
- `@openclaw/openrouter-provider`: `api.ts` exporta o construtor de provedor e
  auxiliares de onboarding/configuração

<Warning>
  O código de produção de extensões também deve evitar importações de
  `openclaw/plugin-sdk/<other-plugin>`. Se um auxiliar for realmente
  compartilhado, promova-o para um subcaminho neutro do SDK, como
  `openclaw/plugin-sdk/speech`, `.../provider-model-shared` ou outra
  superfície orientada por capacidade, em vez de acoplar dois plugins.
</Warning>

## Relacionado

- [Pontos de entrada](/pt-BR/plugins/sdk-entrypoints) — opções de `definePluginEntry` e `defineChannelPluginEntry`
- [Auxiliares de runtime](/pt-BR/plugins/sdk-runtime) — referência completa do namespace `api.runtime`
- [Configuração e schema](/pt-BR/plugins/sdk-setup) — empacotamento, manifests, schemas de configuração
- [Testes](/pt-BR/plugins/sdk-testing) — utilitários de teste e regras de lint
- [Migração do SDK](/pt-BR/plugins/sdk-migration) — migração de superfícies descontinuadas
- [Internos de plugins](/pt-BR/plugins/architecture) — arquitetura profunda e modelo de capacidades
