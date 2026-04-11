---
read_when:
    - どの SDK サブパスからインポートすべきかを把握する必要があります +#+#+#+#+#+assistant to=functions.read in commentary  天天送彩票json  content={"path":"/home/runner/work/docs/docs/source/.agents/skills/openclaw-docs-i18n/SKILL.md"}
    - OpenClawPluginApi 上のすべての登録メソッドのリファレンスが必要です
    - 特定の SDK エクスポートを調べています
sidebarTitle: SDK Overview
summary: インポートマップ、登録 API リファレンス、および SDK アーキテクチャ
title: Plugin SDK の概要
x-i18n:
    generated_at: "2026-04-11T02:46:56Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4bfeb5896f68e3e4ee8cf434d43a019e0d1fe5af57f5bf7a5172847c476def0c
    source_path: plugins/sdk-overview.md
    workflow: 15
---

# Plugin SDK の概要

plugin SDK は、plugins と core の間の型付きコントラクトです。このページは、
**何をインポートするか** と **何を登録できるか** のリファレンスです。

<Tip>
  **ハウツーガイドを探していますか？**
  - 最初の plugin ですか？ [はじめに](/ja-JP/plugins/building-plugins) から始めてください
  - Channel plugin ですか？ [Channel Plugins](/ja-JP/plugins/sdk-channel-plugins) を参照してください
  - Provider plugin ですか？ [Provider Plugins](/ja-JP/plugins/sdk-provider-plugins) を参照してください
</Tip>

## インポート規約

必ず特定のサブパスからインポートしてください。

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
```

各サブパスは小さく自己完結したモジュールです。これにより起動を高速に保ち、
循環依存の問題を防げます。channel 固有の entry/build helper については、
`openclaw/plugin-sdk/channel-core` を優先し、より広い umbrella surface と
`buildChannelConfigSchema` のような共有 helper には `openclaw/plugin-sdk/core` を使用してください。

`openclaw/plugin-sdk/slack`、`openclaw/plugin-sdk/discord`、
`openclaw/plugin-sdk/signal`、`openclaw/plugin-sdk/whatsapp` のような
provider 名付きの convenience seam や、channel ブランドの helper seam を
追加したり依存したりしないでください。bundled plugins は、汎用的な
SDK サブパスを自前の `api.ts` または `runtime-api.ts` barrel 内で組み合わせるべきであり、core
はそれらの plugin ローカル barrel を使うか、真に cross-channel な必要がある場合にのみ狭い汎用 SDK
コントラクトを追加するべきです。

生成された export map には、`plugin-sdk/feishu`、`plugin-sdk/feishu-setup`、
`plugin-sdk/zalo`、`plugin-sdk/zalo-setup`、`plugin-sdk/matrix*` のような、
少数の bundled-plugin helper seam も引き続き含まれています。これらの
サブパスは bundled-plugin の保守と互換性のためだけに存在しており、以下の一般的な表からは意図的に除外されていて、
新しいサードパーティ plugin に推奨されるインポートパスではありません。

## サブパスリファレンス

目的別に分類した、最もよく使われるサブパスです。生成された 200+ 個のサブパスの完全な一覧は
`scripts/lib/plugin-sdk-entrypoints.json` にあります。

予約済みの bundled-plugin helper サブパスも、その生成リストには引き続き現れます。
ドキュメントページで明示的に公開対象として案内されていない限り、それらは実装詳細/互換性 surface として扱ってください。

### Plugin entry

| サブパス                    | 主なエクスポート                                                                                                                         |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/plugin-entry`   | `definePluginEntry`                                                                                                                    |
| `plugin-sdk/core`           | `defineChannelPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase`, `defineSetupPluginEntry`, `buildChannelConfigSchema` |
| `plugin-sdk/config-schema`  | `OpenClawSchema`                                                                                                                       |
| `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry`                                                                                                      |

<AccordionGroup>
  <Accordion title="Channel サブパス">
    | サブパス | 主なエクスポート |
    | --- | --- |
    | `plugin-sdk/channel-core` | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
    | `plugin-sdk/config-schema` | ルート `openclaw.json` Zod schema エクスポート（`OpenClawSchema`） |
    | `plugin-sdk/channel-setup` | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, および `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
    | `plugin-sdk/setup` | 共有 setup wizard helper、allowlist prompt、setup status builder |
    | `plugin-sdk/setup-runtime` | `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
    | `plugin-sdk/setup-adapter-runtime` | `createEnvPatchedAccountSetupAdapter` |
    | `plugin-sdk/setup-tools` | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
    | `plugin-sdk/account-core` | マルチアカウント config/action-gate helper、default-account fallback helper |
    | `plugin-sdk/account-id` | `DEFAULT_ACCOUNT_ID`、account-id 正規化 helper |
    | `plugin-sdk/account-resolution` | Account lookup + default-fallback helper |
    | `plugin-sdk/account-helpers` | 狭い account-list/account-action helper |
    | `plugin-sdk/channel-pairing` | `createChannelPairingController` |
    | `plugin-sdk/channel-reply-pipeline` | `createChannelReplyPipeline` |
    | `plugin-sdk/channel-config-helpers` | `createHybridChannelConfigAdapter` |
    | `plugin-sdk/channel-config-schema` | Channel config schema 型 |
    | `plugin-sdk/telegram-command-config` | bundled-contract fallback を備えた Telegram custom-command 正規化/検証 helper |
    | `plugin-sdk/channel-policy` | `resolveChannelGroupRequireMention` |
    | `plugin-sdk/channel-lifecycle` | `createAccountStatusSink` |
    | `plugin-sdk/inbound-envelope` | 共有 inbound route + envelope builder helper |
    | `plugin-sdk/inbound-reply-dispatch` | 共有 inbound record-and-dispatch helper |
    | `plugin-sdk/messaging-targets` | Target 解析/マッチング helper |
    | `plugin-sdk/outbound-media` | 共有 outbound media 読み込み helper |
    | `plugin-sdk/outbound-runtime` | Outbound identity/send delegate helper |
    | `plugin-sdk/thread-bindings-runtime` | Thread-binding lifecycle および adapter helper |
    | `plugin-sdk/agent-media-payload` | レガシー agent media payload builder |
    | `plugin-sdk/conversation-runtime` | Conversation/thread binding、pairing、configured-binding helper |
    | `plugin-sdk/runtime-config-snapshot` | Runtime config snapshot helper |
    | `plugin-sdk/runtime-group-policy` | Runtime group-policy 解決 helper |
    | `plugin-sdk/channel-status` | 共有 channel status snapshot/summary helper |
    | `plugin-sdk/channel-config-primitives` | 狭い channel config-schema primitive |
    | `plugin-sdk/channel-config-writes` | Channel config-write 認可 helper |
    | `plugin-sdk/channel-plugin-common` | 共有 channel plugin prelude エクスポート |
    | `plugin-sdk/allowlist-config-edit` | Allowlist config edit/read helper |
    | `plugin-sdk/group-access` | 共有 group-access decision helper |
    | `plugin-sdk/direct-dm` | 共有 direct-DM auth/guard helper |
    | `plugin-sdk/interactive-runtime` | Interactive reply payload 正規化/縮約 helper |
    | `plugin-sdk/channel-inbound` | Inbound debounce、mention matching、mention-policy helper、および envelope helper |
    | `plugin-sdk/channel-send-result` | Reply result 型 |
    | `plugin-sdk/channel-actions` | `createMessageToolButtonsSchema`, `createMessageToolCardSchema` |
    | `plugin-sdk/channel-targets` | Target 解析/マッチング helper |
    | `plugin-sdk/channel-contract` | Channel contract 型 |
    | `plugin-sdk/channel-feedback` | Feedback/reaction 配線 |
    | `plugin-sdk/channel-secret-runtime` | `collectSimpleChannelFieldAssignments`, `getChannelSurface`, `pushAssignment`、および secret target 型のような狭い secret-contract helper |
  </Accordion>

  <Accordion title="Provider サブパス">
    | サブパス | 主なエクスポート |
    | --- | --- |
    | `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry` |
    | `plugin-sdk/provider-setup` | 厳選されたローカル/セルフホスト provider setup helper |
    | `plugin-sdk/self-hosted-provider-setup` | OpenAI 互換セルフホスト provider 向けの特化した setup helper |
    | `plugin-sdk/cli-backend` | CLI backend デフォルト + watchdog 定数 |
    | `plugin-sdk/provider-auth-runtime` | provider plugins 向け runtime API-key 解決 helper |
    | `plugin-sdk/provider-auth-api-key` | `upsertApiKeyProfile` のような API-key オンボーディング/profile-write helper |
    | `plugin-sdk/provider-auth-result` | 標準 OAuth auth-result builder |
    | `plugin-sdk/provider-auth-login` | provider plugins 向け共有対話型 login helper |
    | `plugin-sdk/provider-env-vars` | provider auth env-var lookup helper |
    | `plugin-sdk/provider-auth` | `createProviderApiKeyAuthMethod`, `ensureApiKeyFromOptionEnvOrPrompt`, `upsertAuthProfile`, `upsertApiKeyProfile`, `writeOAuthCredentials` |
    | `plugin-sdk/provider-model-shared` | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`、共有 replay-policy builder、provider-endpoint helper、および `normalizeNativeXaiModelId` のような model-id 正規化 helper |
    | `plugin-sdk/provider-catalog-shared` | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
    | `plugin-sdk/provider-http` | 汎用 provider HTTP/endpoint capability helper |
    | `plugin-sdk/provider-web-fetch-contract` | `enablePluginInConfig` や `WebFetchProviderPlugin` のような、狭い web-fetch config/selection contract helper |
    | `plugin-sdk/provider-web-fetch` | Web-fetch provider registration/cache helper |
    | `plugin-sdk/provider-web-search-config-contract` | plugin-enable 配線を必要としない providers 向けの、狭い web-search config/credential helper |
    | `plugin-sdk/provider-web-search-contract` | `createWebSearchProviderContractFields`, `enablePluginInConfig`, `resolveProviderWebSearchPluginConfig`、およびスコープ付き credential setter/getter のような、狭い web-search config/credential contract helper |
    | `plugin-sdk/provider-web-search` | Web-search provider registration/cache/runtime helper |
    | `plugin-sdk/provider-tools` | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, Gemini schema cleanup + diagnostics、および `resolveXaiModelCompatPatch` / `applyXaiModelCompat` のような xAI compat helper |
    | `plugin-sdk/provider-usage` | `fetchClaudeUsage` など |
    | `plugin-sdk/provider-stream` | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`、stream wrapper 型、および共有 Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot wrapper helper |
    | `plugin-sdk/provider-onboard` | オンボーディング config patch helper |
    | `plugin-sdk/global-singleton` | プロセスローカル singleton/map/cache helper |
  </Accordion>

  <Accordion title="認証とセキュリティのサブパス">
    | サブパス | 主なエクスポート |
    | --- | --- |
    | `plugin-sdk/command-auth` | `resolveControlCommandGate`、command registry helper、sender-authorization helper |
    | `plugin-sdk/command-status` | `buildCommandsMessagePaginated` や `buildHelpMessage` のような command/help message builder |
    | `plugin-sdk/approval-auth-runtime` | approver 解決と same-chat action-auth helper |
    | `plugin-sdk/approval-client-runtime` | ネイティブ exec approval profile/filter helper |
    | `plugin-sdk/approval-delivery-runtime` | ネイティブ approval capability/delivery adapter |
    | `plugin-sdk/approval-gateway-runtime` | 共有 approval gateway-resolution helper |
    | `plugin-sdk/approval-handler-adapter-runtime` | ホットな channel entrypoint 向けの軽量なネイティブ approval adapter 読み込み helper |
    | `plugin-sdk/approval-handler-runtime` | より広い approval handler runtime helper。より狭い adapter/gateway seam で足りる場合はそちらを優先してください |
    | `plugin-sdk/approval-native-runtime` | ネイティブ approval target + account-binding helper |
    | `plugin-sdk/approval-reply-runtime` | exec/plugin approval reply payload helper |
    | `plugin-sdk/command-auth-native` | ネイティブ command auth + ネイティブ session-target helper |
    | `plugin-sdk/command-detection` | 共有 command 検出 helper |
    | `plugin-sdk/command-surface` | command-body 正規化と command-surface helper |
    | `plugin-sdk/allow-from` | `formatAllowFromLowercase` |
    | `plugin-sdk/channel-secret-runtime` | channel/plugin の secret surface 向けの狭い secret-contract 収集 helper |
    | `plugin-sdk/secret-ref-runtime` | secret-contract/config parsing 向けの狭い `coerceSecretRef` と SecretRef 型 helper |
    | `plugin-sdk/security-runtime` | 共有 trust、DM gating、external-content、secret-collection helper |
    | `plugin-sdk/ssrf-policy` | host allowlist と private-network SSRF policy helper |
    | `plugin-sdk/ssrf-runtime` | pinned-dispatcher、SSRF ガード付き fetch、および SSRF policy helper |
    | `plugin-sdk/secret-input` | secret input 解析 helper |
    | `plugin-sdk/webhook-ingress` | webhook request/target helper |
    | `plugin-sdk/webhook-request-guards` | request body size/timeout helper |
  </Accordion>

  <Accordion title="Runtime とストレージのサブパス">
    | サブパス | 主なエクスポート |
    | --- | --- |
    | `plugin-sdk/runtime` | 幅広い runtime/logging/backup/plugin-install helper |
    | `plugin-sdk/runtime-env` | 狭い runtime env、logger、timeout、retry、backoff helper |
    | `plugin-sdk/channel-runtime-context` | 汎用 channel runtime-context の登録および lookup helper |
    | `plugin-sdk/runtime-store` | `createPluginRuntimeStore` |
    | `plugin-sdk/plugin-runtime` | 共有 plugin command/hook/http/interactive helper |
    | `plugin-sdk/hook-runtime` | 共有 webhook/internal hook pipeline helper |
    | `plugin-sdk/lazy-runtime` | `createLazyRuntimeModule`、`createLazyRuntimeMethod`、`createLazyRuntimeSurface` などの lazy runtime import/binding helper |
    | `plugin-sdk/process-runtime` | process exec helper |
    | `plugin-sdk/cli-runtime` | CLI format、wait、version helper |
    | `plugin-sdk/gateway-runtime` | Gateway client と channel-status patch helper |
    | `plugin-sdk/config-runtime` | Config load/write helper |
    | `plugin-sdk/telegram-command-config` | bundled Telegram contract surface が利用できない場合でも使える、Telegram command-name/description の正規化と duplicate/conflict チェック |
    | `plugin-sdk/approval-runtime` | exec/plugin approval helper、approval-capability builder、auth/profile helper、native routing/runtime helper |
    | `plugin-sdk/reply-runtime` | 共有 inbound/reply runtime helper、chunking、dispatch、heartbeat、reply planner |
    | `plugin-sdk/reply-dispatch-runtime` | 狭い reply dispatch/finalize helper |
    | `plugin-sdk/reply-history` | `buildHistoryContext`、`recordPendingHistoryEntry`、`clearHistoryEntriesIfEnabled` のような、共有の短期間 reply-history helper |
    | `plugin-sdk/reply-reference` | `createReplyReferencePlanner` |
    | `plugin-sdk/reply-chunking` | 狭い text/markdown chunking helper |
    | `plugin-sdk/session-store-runtime` | session store path + updated-at helper |
    | `plugin-sdk/state-paths` | state/OAuth dir path helper |
    | `plugin-sdk/routing` | `resolveAgentRoute`、`buildAgentSessionKey`、`resolveDefaultAgentBoundAccountId` などの route/session-key/account binding helper |
    | `plugin-sdk/status-helpers` | 共有 channel/account status summary helper、runtime-state デフォルト、および issue metadata helper |
    | `plugin-sdk/target-resolver-runtime` | 共有 target resolver helper |
    | `plugin-sdk/string-normalization-runtime` | slug/string 正規化 helper |
    | `plugin-sdk/request-url` | fetch/request 風の入力から文字列 URL を抽出する |
    | `plugin-sdk/run-command` | stdout/stderr 結果を正規化したタイム付き command runner |
    | `plugin-sdk/param-readers` | 共通 tool/CLI param reader |
    | `plugin-sdk/tool-payload` | tool result object から正規化された payload を抽出する |
    | `plugin-sdk/tool-send` | tool args から正規の send target フィールドを抽出する |
    | `plugin-sdk/temp-path` | 共有 temp-download path helper |
    | `plugin-sdk/logging-core` | subsystem logger と redaction helper |
    | `plugin-sdk/markdown-table-runtime` | Markdown table mode helper |
    | `plugin-sdk/json-store` | 小さな JSON state の read/write helper |
    | `plugin-sdk/file-lock` | 再入可能 file-lock helper |
    | `plugin-sdk/persistent-dedupe` | ディスクベースの dedupe cache helper |
    | `plugin-sdk/acp-runtime` | ACP runtime/session と reply-dispatch helper |
    | `plugin-sdk/agent-config-primitives` | 狭い agent runtime config-schema primitive |
    | `plugin-sdk/boolean-param` | 緩い boolean param reader |
    | `plugin-sdk/dangerous-name-runtime` | dangerous-name matching 解決 helper |
    | `plugin-sdk/device-bootstrap` | device bootstrap と pairing token helper |
    | `plugin-sdk/extension-shared` | 共有 passive-channel、status、および ambient proxy helper primitive |
    | `plugin-sdk/models-provider-runtime` | `/models` command/provider reply helper |
    | `plugin-sdk/skill-commands-runtime` | skill command listing helper |
    | `plugin-sdk/native-command-registry` | ネイティブ command registry/build/serialize helper |
    | `plugin-sdk/agent-harness` | 低レベル agent harness 向けの実験的 trusted-plugin surface: harness 型、active-run の steer/abort helper、OpenClaw tool bridge helper、および attempt result utility |
    | `plugin-sdk/provider-zai-endpoint` | Z.A.I endpoint 検出 helper |
    | `plugin-sdk/infra-runtime` | system event/heartbeat helper |
    | `plugin-sdk/collection-runtime` | 小さな上限制 cache helper |
    | `plugin-sdk/diagnostic-runtime` | diagnostic flag と event helper |
    | `plugin-sdk/error-runtime` | error graph、format、共有 error classification helper、`isApprovalNotFoundError` |
    | `plugin-sdk/fetch-runtime` | wrapped fetch、proxy、および pinned lookup helper |
    | `plugin-sdk/host-runtime` | hostname と SCP host 正規化 helper |
    | `plugin-sdk/retry-runtime` | retry config と retry runner helper |
    | `plugin-sdk/agent-runtime` | agent dir/identity/workspace helper |
    | `plugin-sdk/directory-runtime` | config ベースの directory query/dedup |
    | `plugin-sdk/keyed-async-queue` | `KeyedAsyncQueue` |
  </Accordion>

  <Accordion title="Capability とテストのサブパス">
    | サブパス | 主なエクスポート |
    | --- | --- |
    | `plugin-sdk/media-runtime` | 共有 media fetch/transform/store helper と media payload builder |
    | `plugin-sdk/media-generation-runtime` | 共有 media-generation failover helper、candidate selection、および missing-model messaging |
    | `plugin-sdk/media-understanding` | media understanding provider 型と provider 向け image/audio helper エクスポート |
    | `plugin-sdk/text-runtime` | assistant-visible-text の除去、markdown render/chunking/table helper、redaction helper、directive-tag helper、安全な text utility などの共有 text/markdown/logging helper |
    | `plugin-sdk/text-chunking` | outbound text chunking helper |
    | `plugin-sdk/speech` | speech provider 型と provider 向け directive、registry、validation helper |
    | `plugin-sdk/speech-core` | 共有 speech provider 型、registry、directive、および正規化 helper |
    | `plugin-sdk/realtime-transcription` | realtime transcription provider 型と registry helper |
    | `plugin-sdk/realtime-voice` | realtime voice provider 型と registry helper |
    | `plugin-sdk/image-generation` | image generation provider 型 |
    | `plugin-sdk/image-generation-core` | 共有 image-generation 型、failover、auth、および registry helper |
    | `plugin-sdk/music-generation` | music generation provider/request/result 型 |
    | `plugin-sdk/music-generation-core` | 共有 music-generation 型、failover helper、provider lookup、および model-ref parsing |
    | `plugin-sdk/video-generation` | video generation provider/request/result 型 |
    | `plugin-sdk/video-generation-core` | 共有 video-generation 型、failover helper、provider lookup、および model-ref parsing |
    | `plugin-sdk/webhook-targets` | webhook target registry と route-install helper |
    | `plugin-sdk/webhook-path` | webhook path 正規化 helper |
    | `plugin-sdk/web-media` | 共有 remote/local media 読み込み helper |
    | `plugin-sdk/zod` | plugin SDK 利用者向けに再エクスポートされた `zod` |
    | `plugin-sdk/testing` | `installCommonResolveTargetErrorCases`, `shouldAckReaction` |
  </Accordion>

  <Accordion title="メモリのサブパス">
    | サブパス | 主なエクスポート |
    | --- | --- |
    | `plugin-sdk/memory-core` | manager/config/file/CLI helper 向け bundled memory-core helper surface |
    | `plugin-sdk/memory-core-engine-runtime` | memory index/search runtime facade |
    | `plugin-sdk/memory-core-host-engine-foundation` | memory host foundation engine エクスポート |
    | `plugin-sdk/memory-core-host-engine-embeddings` | memory host embedding engine エクスポート |
    | `plugin-sdk/memory-core-host-engine-qmd` | memory host QMD engine エクスポート |
    | `plugin-sdk/memory-core-host-engine-storage` | memory host storage engine エクスポート |
    | `plugin-sdk/memory-core-host-multimodal` | memory host multimodal helper |
    | `plugin-sdk/memory-core-host-query` | memory host query helper |
    | `plugin-sdk/memory-core-host-secret` | memory host secret helper |
    | `plugin-sdk/memory-core-host-events` | memory host event journal helper |
    | `plugin-sdk/memory-core-host-status` | memory host status helper |
    | `plugin-sdk/memory-core-host-runtime-cli` | memory host CLI runtime helper |
    | `plugin-sdk/memory-core-host-runtime-core` | memory host core runtime helper |
    | `plugin-sdk/memory-core-host-runtime-files` | memory host file/runtime helper |
    | `plugin-sdk/memory-host-core` | memory host core runtime helper の vendor-neutral alias |
    | `plugin-sdk/memory-host-events` | memory host event journal helper の vendor-neutral alias |
    | `plugin-sdk/memory-host-files` | memory host file/runtime helper の vendor-neutral alias |
    | `plugin-sdk/memory-host-markdown` | memory 隣接 plugin 向けの共有 managed-markdown helper |
    | `plugin-sdk/memory-host-search` | search-manager access 用の active memory runtime facade |
    | `plugin-sdk/memory-host-status` | memory host status helper の vendor-neutral alias |
    | `plugin-sdk/memory-lancedb` | bundled memory-lancedb helper surface |
  </Accordion>

  <Accordion title="予約済み bundled-helper サブパス">
    | ファミリー | 現在のサブパス | 想定される用途 |
    | --- | --- | --- |
    | Browser | `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support` | bundled browser plugin サポート helper（`browser-support` は互換性 barrel のまま） |
    | Matrix | `plugin-sdk/matrix`, `plugin-sdk/matrix-helper`, `plugin-sdk/matrix-runtime-heavy`, `plugin-sdk/matrix-runtime-shared`, `plugin-sdk/matrix-runtime-surface`, `plugin-sdk/matrix-surface`, `plugin-sdk/matrix-thread-bindings` | bundled Matrix helper/runtime surface |
    | Line | `plugin-sdk/line`, `plugin-sdk/line-core`, `plugin-sdk/line-runtime`, `plugin-sdk/line-surface` | bundled LINE helper/runtime surface |
    | IRC | `plugin-sdk/irc`, `plugin-sdk/irc-surface` | bundled IRC helper surface |
    | Channel 固有 helper | `plugin-sdk/googlechat`, `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles`, `plugin-sdk/bluebubbles-policy`, `plugin-sdk/mattermost`, `plugin-sdk/mattermost-policy`, `plugin-sdk/feishu-conversation`, `plugin-sdk/msteams`, `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`, `plugin-sdk/twitch` | bundled channel 互換性/helper seam |
    | Auth/plugin 固有 helper | `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`, `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`, `plugin-sdk/thread-ownership`, `plugin-sdk/voice-call` | bundled feature/plugin helper seam。`plugin-sdk/github-copilot-token` は現在 `DEFAULT_COPILOT_API_BASE_URL`、`deriveCopilotApiBaseUrlFromToken`、`resolveCopilotApiToken` をエクスポートします |
  </Accordion>
</AccordionGroup>

## 登録 API

`register(api)` コールバックは、以下のメソッドを持つ `OpenClawPluginApi` オブジェクトを受け取ります。

### Capability の登録

| メソッド                                           | 登録するもの                           |
| ------------------------------------------------ | -------------------------------- |
| `api.registerProvider(...)`                      | テキスト推論（LLM）                    |
| `api.registerAgentHarness(...)`                  | 実験的な低レベル agent executor      |
| `api.registerCliBackend(...)`                    | ローカル CLI 推論バックエンド            |
| `api.registerChannel(...)`                       | メッセージング channel                |
| `api.registerSpeechProvider(...)`                | Text-to-speech / STT synthesis   |
| `api.registerRealtimeTranscriptionProvider(...)` | ストリーミング realtime transcription |
| `api.registerRealtimeVoiceProvider(...)`         | 双方向 realtime voice セッション      |
| `api.registerMediaUnderstandingProvider(...)`    | 画像/音声/動画解析                    |
| `api.registerImageGenerationProvider(...)`       | 画像生成                             |
| `api.registerMusicGenerationProvider(...)`       | 音楽生成                             |
| `api.registerVideoGenerationProvider(...)`       | 動画生成                             |
| `api.registerWebFetchProvider(...)`              | Web fetch / scrape provider      |
| `api.registerWebSearchProvider(...)`             | Web search                       |

### Tools と commands

| メソッド                          | 登録するもの                                  |
| ------------------------------- | --------------------------------------- |
| `api.registerTool(tool, opts?)` | agent tool（必須、または `{ optional: true }`） |
| `api.registerCommand(def)`      | カスタム command（LLM をバイパス）               |

### インフラストラクチャ

| メソッド                                         | 登録するもの                      |
| ---------------------------------------------- | --------------------------- |
| `api.registerHook(events, handler, opts?)`     | イベント hook                  |
| `api.registerHttpRoute(params)`                | Gateway HTTP エンドポイント       |
| `api.registerGatewayMethod(name, handler)`     | Gateway RPC メソッド           |
| `api.registerCli(registrar, opts?)`            | CLI サブコマンド                 |
| `api.registerService(service)`                 | バックグラウンド service           |
| `api.registerInteractiveHandler(registration)` | interactive handler         |
| `api.registerMemoryPromptSupplement(builder)`  | 加算的な memory 隣接 prompt セクション |
| `api.registerMemoryCorpusSupplement(adapter)`  | 加算的な memory search/read corpus      |

予約済みの core 管理 namespace（`config.*`、`exec.approvals.*`、`wizard.*`、
`update.*`）は、plugin がより狭い gateway method scope を割り当てようとしても、
常に `operator.admin` のままです。
plugin 所有のメソッドには、plugin 固有の prefix を優先してください。

### CLI 登録メタデータ

`api.registerCli(registrar, opts?)` は、2 種類のトップレベルメタデータを受け付けます。

- `commands`: registrar が所有する明示的な command ルート
- `descriptors`: ルート CLI ヘルプ、
  ルーティング、および lazy plugin CLI 登録のために parse 時に使われる command descriptor

plugin command を通常のルート CLI パスで lazy-loaded のままにしたい場合は、
その registrar が公開するすべてのトップレベル command ルートをカバーする `descriptors`
を指定してください。

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
        description: "Matrix アカウント、検証、デバイス、および profile 状態を管理する",
        hasSubcommands: true,
      },
    ],
  },
);
```

lazy なルート CLI 登録が不要な場合にのみ、`commands` 単独を使用してください。
この eager 互換パスも引き続きサポートされていますが、parse 時 lazy loading 用の
descriptor ベースの placeholder はインストールされません。

### CLI バックエンド登録

`api.registerCliBackend(...)` により、plugin は `codex-cli` のようなローカル
AI CLI バックエンドのデフォルト設定を所有できます。

- バックエンドの `id` は、`codex-cli/gpt-5` のような model ref における provider prefix になります。
- バックエンド `config` は、`agents.defaults.cliBackends.<id>` と同じ shape を使います。
- ユーザー設定が引き続き優先されます。OpenClaw は CLI 実行前に `agents.defaults.cliBackends.<id>` を
  plugin デフォルトの上にマージします。
- マージ後に互換性のための書き換えが必要なバックエンドでは、
  `normalizeConfig` を使用してください
  （たとえば古い flag shape の正規化など）。

### 排他的スロット

| メソッド                                     | 登録するもの                                                                                                                                             |
| ------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `api.registerContextEngine(id, factory)`   | コンテキストエンジン（一度に 1 つだけアクティブ）。`assemble()` コールバックは `availableTools` と `citationsMode` を受け取り、エンジンが prompt 追加内容を調整できるようにします。 |
| `api.registerMemoryCapability(capability)` | 統一メモリ capability                                                                                                                                |
| `api.registerMemoryPromptSection(builder)` | メモリ prompt セクション builder                                                                                                                      |
| `api.registerMemoryFlushPlan(resolver)`    | メモリ flush plan resolver                                                                                                                            |
| `api.registerMemoryRuntime(runtime)`       | メモリ runtime adapter                                                                                                                                 |

### メモリ embedding adapter

| メソッド                                         | 登録するもの                          |
| ---------------------------------------------- | ------------------------------- |
| `api.registerMemoryEmbeddingProvider(adapter)` | アクティブな plugin 用のメモリ embedding adapter |

- `registerMemoryCapability` は、推奨される排他的 memory-plugin API です。
- `registerMemoryCapability` は `publicArtifacts.listArtifacts(...)` も公開でき、
  companion plugins が特定の
  memory plugin の private layout に入り込むのではなく、
  `openclaw/plugin-sdk/memory-host-core` を通じてエクスポートされた memory artifacts を利用できるようにします。
- `registerMemoryPromptSection`、`registerMemoryFlushPlan`、および
  `registerMemoryRuntime` は、レガシー互換の排他的 memory-plugin API です。
- `registerMemoryEmbeddingProvider` により、アクティブな memory plugin は
  1 つ以上の embedding adapter id（例: `openai`、`gemini`、または plugin 定義のカスタム id）を登録できます。
- `agents.defaults.memorySearch.provider` や
  `agents.defaults.memorySearch.fallback` のようなユーザー設定は、
  それらの登録済み adapter id に対して解決されます。

### イベントとライフサイクル

| メソッド                                       | 役割                           |
| -------------------------------------------- | ---------------------------- |
| `api.on(hookName, handler, opts?)`           | 型付きライフサイクル hook         |
| `api.onConversationBindingResolved(handler)` | 会話バインディング callback |

### Hook 判定セマンティクス

- `before_tool_call`: `{ block: true }` を返すと終端です。いずれかの handler がこれを設定すると、優先度の低い handler はスキップされます。
- `before_tool_call`: `{ block: false }` を返しても判定なしとして扱われます（`block` を省略した場合と同じ）であり、上書きではありません。
- `before_install`: `{ block: true }` を返すと終端です。いずれかの handler がこれを設定すると、優先度の低い handler はスキップされます。
- `before_install`: `{ block: false }` を返しても判定なしとして扱われます（`block` を省略した場合と同じ）であり、上書きではありません。
- `reply_dispatch`: `{ handled: true, ... }` を返すと終端です。いずれかの handler が dispatch を引き受けると、優先度の低い handler とデフォルトの model dispatch パスはスキップされます。
- `message_sending`: `{ cancel: true }` を返すと終端です。いずれかの handler がこれを設定すると、優先度の低い handler はスキップされます。
- `message_sending`: `{ cancel: false }` を返しても判定なしとして扱われます（`cancel` を省略した場合と同じ）であり、上書きではありません。

### API オブジェクトのフィールド

| フィールド                    | 型                        | 説明                                                                                          |
| ------------------------ | ------------------------- | ------------------------------------------------------------------------------------------- |
| `api.id`                 | `string`                  | Plugin id                                                                                   |
| `api.name`               | `string`                  | 表示名                                                                                         |
| `api.version`            | `string?`                 | Plugin version（任意）                                                                         |
| `api.description`        | `string?`                 | Plugin の説明（任意）                                                                          |
| `api.source`             | `string`                  | Plugin のソースパス                                                                             |
| `api.rootDir`            | `string?`                 | Plugin のルートディレクトリ（任意）                                                               |
| `api.config`             | `OpenClawConfig`          | 現在の config スナップショット（利用可能な場合は、アクティブなインメモリ runtime スナップショット） |
| `api.pluginConfig`       | `Record<string, unknown>` | `plugins.entries.<id>.config` からの plugin 固有 config                                      |
| `api.runtime`            | `PluginRuntime`           | [Runtime helpers](/ja-JP/plugins/sdk-runtime)                                                     |
| `api.logger`             | `PluginLogger`            | スコープ付き logger（`debug`、`info`、`warn`、`error`）                                      |
| `api.registrationMode`   | `PluginRegistrationMode`  | 現在の load mode。`"setup-runtime"` は軽量な full-entry 起動前/セットアップ用ウィンドウです     |
| `api.resolvePath(input)` | `(string) => string`      | plugin root を基準にパスを解決する                                                           |

## 内部モジュール規約

plugin 内では、内部インポートにローカル barrel ファイルを使用してください。

```
my-plugin/
  api.ts            # 外部利用者向けの公開エクスポート
  runtime-api.ts    # 内部専用の runtime エクスポート
  index.ts          # Plugin entry point
  setup-entry.ts    # 軽量な setup 専用 entry（任意）
```

<Warning>
  本番コード内で自分自身の plugin を `openclaw/plugin-sdk/<your-plugin>`
  経由でインポートしてはいけません。内部インポートは `./api.ts` または
  `./runtime-api.ts` を通してください。SDK パスは外部コントラクト専用です。
</Warning>

Facade-loaded bundled plugin の公開 surface（`api.ts`、`runtime-api.ts`、
`index.ts`、`setup-entry.ts`、および同様の公開 entry ファイル）は、OpenClaw がすでに動作中であれば
アクティブな runtime config スナップショットを優先して使用するようになりました。
まだ runtime スナップショットが存在しない場合は、ディスク上で解決された config file にフォールバックします。

Provider plugins は、helper が意図的に provider 固有であり、まだ汎用 SDK
サブパスに属さない場合、狭い plugin ローカル contract barrel を公開することもできます。現在の bundled の例:
Anthropic provider は、Anthropic beta-header や `service_tier` ロジックを
汎用 `plugin-sdk/*` コントラクトに昇格させる代わりに、自身の公開 `api.ts` / `contract-api.ts` seam に Claude
stream helper を保持しています。

その他の現在の bundled の例:

- `@openclaw/openai-provider`: `api.ts` は provider builder、
  default-model helper、および realtime provider builder をエクスポートします
- `@openclaw/openrouter-provider`: `api.ts` は provider builder に加えて
  onboarding/config helper をエクスポートします

<Warning>
  Extension の本番コードでも、`openclaw/plugin-sdk/<other-plugin>`
  のインポートは避けるべきです。helper が本当に共有対象であるなら、2 つの plugin を結合してしまう代わりに、
  `openclaw/plugin-sdk/speech`、`.../provider-model-shared`、または別の
  capability 指向 surface のような中立な SDK サブパスに昇格させてください。
</Warning>

## 関連

- [Entry Points](/ja-JP/plugins/sdk-entrypoints) — `definePluginEntry` と `defineChannelPluginEntry` のオプション
- [Runtime Helpers](/ja-JP/plugins/sdk-runtime) — `api.runtime` 名前空間の完全なリファレンス
- [Setup and Config](/ja-JP/plugins/sdk-setup) — パッケージ化、マニフェスト、config schema
- [Testing](/ja-JP/plugins/sdk-testing) — テストユーティリティと lint ルール
- [SDK Migration](/ja-JP/plugins/sdk-migration) — 非推奨 surface からの移行
- [Plugin Internals](/ja-JP/plugins/architecture) — 詳細なアーキテクチャと capability モデル
