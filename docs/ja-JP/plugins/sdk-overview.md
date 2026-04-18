---
read_when:
    - どのSDKサブパスからインポートすべきかを知る必要がある
    - OpenClawPluginApi のすべての登録メソッドのリファレンスが欲しい
    - 特定のSDKエクスポートを調べている
sidebarTitle: SDK Overview
summary: インポートマップ、登録APIリファレンス、SDKアーキテクチャ
title: Plugin SDK の概要
x-i18n:
    generated_at: "2026-04-18T04:40:48Z"
    model: gpt-5.4
    provider: openai
    source_hash: 05d3d0022cca32d29c76f6cea01cdf4f88ac69ef0ef3d7fb8a60fbf9a6b9b331
    source_path: plugins/sdk-overview.md
    workflow: 15
---

# Plugin SDK の概要

plugin SDKは、pluginとコアの間にある型付き契約です。このページは、**何をインポートするか** と **何を登録できるか** のリファレンスです。

<Tip>
  **ハウツーガイドを探していますか？**
  - 初めてのpluginですか？ [はじめに](/ja-JP/plugins/building-plugins) から始めてください
  - Channel pluginですか？ [Channel Plugins](/ja-JP/plugins/sdk-channel-plugins) を参照してください
  - Provider pluginですか？ [Provider Plugins](/ja-JP/plugins/sdk-provider-plugins) を参照してください
</Tip>

## インポート規約

必ず特定のサブパスからインポートしてください：

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
```

各サブパスは、小さく自己完結したモジュールです。これにより起動を高速に保ち、
循環依存の問題を防ぎます。channel固有のエントリ/ビルドヘルパーについては、
`openclaw/plugin-sdk/channel-core` を優先し、より広いアンブレラサーフェスや
`buildChannelConfigSchema` のような共有ヘルパーには
`openclaw/plugin-sdk/core` を使ってください。

`openclaw/plugin-sdk/slack`、`openclaw/plugin-sdk/discord`、
`openclaw/plugin-sdk/signal`、`openclaw/plugin-sdk/whatsapp` のような
provider名付きの便利用シームや、channelブランド付きのヘルパーシームを
追加したり依存したりしないでください。バンドルされたpluginは、汎用的な
SDKサブパスを自身の `api.ts` または `runtime-api.ts` バレル内で組み合わせるべきであり、コアはそれらのpluginローカルバレルを使うか、本当に
チャネル横断の必要がある場合に限って狭い汎用SDK契約を追加すべきです。

生成されたエクスポートマップには、`plugin-sdk/feishu`、
`plugin-sdk/feishu-setup`、`plugin-sdk/zalo`、
`plugin-sdk/zalo-setup`、`plugin-sdk/matrix*` のような、
少数のバンドルplugin用ヘルパーシームが依然として含まれています。これらの
サブパスは、バンドルpluginの保守と互換性のためだけに存在します。意図的に
下の共通テーブルからは除外されており、新しいサードパーティpluginに推奨される
インポートパスではありません。

## サブパスリファレンス

目的別にまとめた、最も一般的に使われるサブパスです。200を超えるサブパスの
生成済み完全一覧は `scripts/lib/plugin-sdk-entrypoints.json` にあります。

予約済みのバンドルplugin用ヘルパーサブパスは、その生成済み一覧には依然として
表示されます。ドキュメントページで明示的に公開として推奨されていない限り、
それらは実装詳細/互換性サーフェスとして扱ってください。

### Plugin entry

| Subpath | 主なエクスポート |
| --------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/plugin-entry`   | `definePluginEntry`                                                                                                                    |
| `plugin-sdk/core`           | `defineChannelPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase`, `defineSetupPluginEntry`, `buildChannelConfigSchema` |
| `plugin-sdk/config-schema`  | `OpenClawSchema`                                                                                                                       |
| `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry`                                                                                                      |

<AccordionGroup>
  <Accordion title="Channel subpaths">
    | Subpath | 主なエクスポート |
    | --- | --- |
    | `plugin-sdk/channel-core` | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
    | `plugin-sdk/config-schema` | ルート `openclaw.json` Zodスキーマエクスポート (`OpenClawSchema`) |
    | `plugin-sdk/channel-setup` | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, および `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
    | `plugin-sdk/setup` | 共有セットアップウィザードヘルパー、allowlistプロンプト、セットアップステータスビルダー |
    | `plugin-sdk/setup-runtime` | `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
    | `plugin-sdk/setup-adapter-runtime` | `createEnvPatchedAccountSetupAdapter` |
    | `plugin-sdk/setup-tools` | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
    | `plugin-sdk/account-core` | マルチアカウント設定/アクションゲートヘルパー、デフォルトアカウントフォールバックヘルパー |
    | `plugin-sdk/account-id` | `DEFAULT_ACCOUNT_ID`、account-id正規化ヘルパー |
    | `plugin-sdk/account-resolution` | アカウント検索 + デフォルトフォールバックヘルパー |
    | `plugin-sdk/account-helpers` | 狭いaccount-list/account-actionヘルパー |
    | `plugin-sdk/channel-pairing` | `createChannelPairingController` |
    | `plugin-sdk/channel-reply-pipeline` | `createChannelReplyPipeline` |
    | `plugin-sdk/channel-config-helpers` | `createHybridChannelConfigAdapter` |
    | `plugin-sdk/channel-config-schema` | Channel設定スキーマ型 |
    | `plugin-sdk/telegram-command-config` | バンドル契約フォールバック付きのTelegramカスタムコマンド正規化/検証ヘルパー |
    | `plugin-sdk/command-gating` | 狭いコマンド認可ゲートヘルパー |
    | `plugin-sdk/channel-policy` | `resolveChannelGroupRequireMention` |
    | `plugin-sdk/channel-lifecycle` | `createAccountStatusSink` |
    | `plugin-sdk/inbound-envelope` | 共有の受信ルート + エンベロープビルダーヘルパー |
    | `plugin-sdk/inbound-reply-dispatch` | 共有の受信record-and-dispatchヘルパー |
    | `plugin-sdk/messaging-targets` | ターゲット解析/マッチングヘルパー |
    | `plugin-sdk/outbound-media` | 共有の送信メディア読み込みヘルパー |
    | `plugin-sdk/outbound-runtime` | 送信元identity/sendデリゲートヘルパー |
    | `plugin-sdk/poll-runtime` | 狭いpoll正規化ヘルパー |
    | `plugin-sdk/thread-bindings-runtime` | スレッドバインディングのライフサイクルおよびアダプタヘルパー |
    | `plugin-sdk/agent-media-payload` | 旧来のagentメディアペイロードビルダー |
    | `plugin-sdk/conversation-runtime` | 会話/スレッドバインディング、pairing、configured-bindingヘルパー |
    | `plugin-sdk/runtime-config-snapshot` | ランタイム設定スナップショットヘルパー |
    | `plugin-sdk/runtime-group-policy` | ランタイムgroup-policy解決ヘルパー |
    | `plugin-sdk/channel-status` | 共有channelステータススナップショット/サマリーヘルパー |
    | `plugin-sdk/channel-config-primitives` | 狭いchannel設定スキーマプリミティブ |
    | `plugin-sdk/channel-config-writes` | channel設定書き込み認可ヘルパー |
    | `plugin-sdk/channel-plugin-common` | 共有channel pluginプレリュードエクスポート |
    | `plugin-sdk/allowlist-config-edit` | allowlist設定の編集/読み取りヘルパー |
    | `plugin-sdk/group-access` | 共有group-access判定ヘルパー |
    | `plugin-sdk/direct-dm` | 共有direct-DM認証/ガードヘルパー |
    | `plugin-sdk/interactive-runtime` | 対話型返信ペイロードの正規化/削減ヘルパー |
    | `plugin-sdk/channel-inbound` | 受信デバウンス、メンションマッチング、mention-policyヘルパー、およびエンベロープヘルパーの互換性バレル |
    | `plugin-sdk/channel-mention-gating` | より広い受信ランタイムサーフェスを含まない、狭いmention-policyヘルパー |
    | `plugin-sdk/channel-location` | channel位置コンテキストおよびフォーマットヘルパー |
    | `plugin-sdk/channel-logging` | 受信ドロップおよびtyping/ack失敗のためのchannelロギングヘルパー |
    | `plugin-sdk/channel-send-result` | 返信結果型 |
    | `plugin-sdk/channel-actions` | `createMessageToolButtonsSchema`, `createMessageToolCardSchema` |
    | `plugin-sdk/channel-targets` | ターゲット解析/マッチングヘルパー |
    | `plugin-sdk/channel-contract` | Channel契約型 |
    | `plugin-sdk/channel-feedback` | フィードバック/リアクション接続 |
    | `plugin-sdk/channel-secret-runtime` | `collectSimpleChannelFieldAssignments`, `getChannelSurface`, `pushAssignment`、およびsecret target型のような狭いsecret-contractヘルパー |
  </Accordion>

  <Accordion title="Provider subpaths">
    | Subpath | 主なエクスポート |
    | --- | --- |
    | `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry` |
    | `plugin-sdk/provider-setup` | 厳選されたローカル/セルフホストproviderセットアップヘルパー |
    | `plugin-sdk/self-hosted-provider-setup` | OpenAI互換セルフホストprovider向けの特化セットアップヘルパー |
    | `plugin-sdk/cli-backend` | CLIバックエンドのデフォルト + watchdog定数 |
    | `plugin-sdk/provider-auth-runtime` | provider plugin向けのランタイムAPIキー解決ヘルパー |
    | `plugin-sdk/provider-auth-api-key` | `upsertApiKeyProfile` などのAPIキーオンボーディング/プロファイル書き込みヘルパー |
    | `plugin-sdk/provider-auth-result` | 標準OAuth認証結果ビルダー |
    | `plugin-sdk/provider-auth-login` | provider plugin向けの共有対話型ログインヘルパー |
    | `plugin-sdk/provider-env-vars` | provider認証env-var検索ヘルパー |
    | `plugin-sdk/provider-auth` | `createProviderApiKeyAuthMethod`, `ensureApiKeyFromOptionEnvOrPrompt`, `upsertAuthProfile`, `upsertApiKeyProfile`, `writeOAuthCredentials` |
    | `plugin-sdk/provider-model-shared` | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, 共有replay-policyビルダー、providerエンドポイントヘルパー、および `normalizeNativeXaiModelId` のようなmodel-id正規化ヘルパー |
    | `plugin-sdk/provider-catalog-shared` | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
    | `plugin-sdk/provider-http` | 汎用provider HTTP/エンドポイント機能ヘルパー |
    | `plugin-sdk/provider-web-fetch-contract` | `enablePluginInConfig` や `WebFetchProviderPlugin` のような、狭いweb-fetch設定/選択契約ヘルパー |
    | `plugin-sdk/provider-web-fetch` | web-fetch provider登録/キャッシュヘルパー |
    | `plugin-sdk/provider-web-search-config-contract` | plugin有効化接続を必要としないprovider向けの、狭いweb-search設定/認証情報ヘルパー |
    | `plugin-sdk/provider-web-search-contract` | `createWebSearchProviderContractFields`, `enablePluginInConfig`, `resolveProviderWebSearchPluginConfig`、およびスコープ付き認証情報setter/getterのような、狭いweb-search設定/認証情報契約ヘルパー |
    | `plugin-sdk/provider-web-search` | web-search provider登録/キャッシュ/ランタイムヘルパー |
    | `plugin-sdk/provider-tools` | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, Geminiスキーマクリーンアップ + 診断、および `resolveXaiModelCompatPatch` / `applyXaiModelCompat` のようなxAI互換ヘルパー |
    | `plugin-sdk/provider-usage` | `fetchClaudeUsage` など |
    | `plugin-sdk/provider-stream` | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, ストリームラッパー型、および共有のAnthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilotラッパーヘルパー |
    | `plugin-sdk/provider-onboard` | オンボーディング設定パッチヘルパー |
    | `plugin-sdk/global-singleton` | プロセスローカルsingleton/map/cacheヘルパー |
  </Accordion>

  <Accordion title="認証とセキュリティのサブパス">
    | Subpath | 主なエクスポート |
    | --- | --- |
    | `plugin-sdk/command-auth` | `resolveControlCommandGate`、コマンドレジストリヘルパー、送信者認可ヘルパー |
    | `plugin-sdk/command-status` | `buildCommandsMessagePaginated` や `buildHelpMessage` のようなコマンド/ヘルプメッセージビルダー |
    | `plugin-sdk/approval-auth-runtime` | 承認者解決および同一チャットのアクション認可ヘルパー |
    | `plugin-sdk/approval-client-runtime` | ネイティブexec承認プロファイル/フィルタヘルパー |
    | `plugin-sdk/approval-delivery-runtime` | ネイティブ承認機能/配信アダプタ |
    | `plugin-sdk/approval-gateway-runtime` | 共有承認Gateway解決ヘルパー |
    | `plugin-sdk/approval-handler-adapter-runtime` | ホットなchannelエントリポイント向けの軽量ネイティブ承認アダプタ読み込みヘルパー |
    | `plugin-sdk/approval-handler-runtime` | より広い承認ハンドラーランタイムヘルパー。狭いadapter/gatewayシームで十分な場合はそちらを優先してください |
    | `plugin-sdk/approval-native-runtime` | ネイティブ承認ターゲット + アカウントバインディングヘルパー |
    | `plugin-sdk/approval-reply-runtime` | exec/plugin承認返信ペイロードヘルパー |
    | `plugin-sdk/command-auth-native` | ネイティブコマンド認可 + ネイティブセッションターゲットヘルパー |
    | `plugin-sdk/command-detection` | 共有コマンド検出ヘルパー |
    | `plugin-sdk/command-surface` | コマンド本文の正規化およびコマンドサーフェスヘルパー |
    | `plugin-sdk/allow-from` | `formatAllowFromLowercase` |
    | `plugin-sdk/channel-secret-runtime` | channel/plugin secretサーフェス向けの狭いsecret-contract収集ヘルパー |
    | `plugin-sdk/secret-ref-runtime` | secret-contract/設定解析向けの、狭い `coerceSecretRef` および SecretRef 型ヘルパー |
    | `plugin-sdk/security-runtime` | 共有の信頼、DMゲーティング、外部コンテンツ、およびsecret収集ヘルパー |
    | `plugin-sdk/ssrf-policy` | ホストallowlistおよびプライベートネットワークSSRFポリシーヘルパー |
    | `plugin-sdk/ssrf-dispatcher` | 広いinfraランタイムサーフェスを含まない、狭いpinned-dispatcherヘルパー |
    | `plugin-sdk/ssrf-runtime` | pinned-dispatcher、SSRFガード付きfetch、およびSSRFポリシーヘルパー |
    | `plugin-sdk/secret-input` | secret入力解析ヘルパー |
    | `plugin-sdk/webhook-ingress` | Webhookリクエスト/ターゲットヘルパー |
    | `plugin-sdk/webhook-request-guards` | リクエスト本文サイズ/タイムアウトヘルパー |
  </Accordion>

  <Accordion title="ランタイムとストレージのサブパス">
    | Subpath | 主なエクスポート |
    | --- | --- |
    | `plugin-sdk/runtime` | 広範なランタイム/ロギング/バックアップ/pluginインストールヘルパー |
    | `plugin-sdk/runtime-env` | 狭いランタイムenv、logger、timeout、retry、およびbackoffヘルパー |
    | `plugin-sdk/channel-runtime-context` | 汎用channelランタイムコンテキスト登録および検索ヘルパー |
    | `plugin-sdk/runtime-store` | `createPluginRuntimeStore` |
    | `plugin-sdk/plugin-runtime` | 共有pluginコマンド/hook/http/interactiveヘルパー |
    | `plugin-sdk/hook-runtime` | 共有Webhook/internal hookパイプラインヘルパー |
    | `plugin-sdk/lazy-runtime` | `createLazyRuntimeModule`、`createLazyRuntimeMethod`、`createLazyRuntimeSurface` のような遅延ランタイムimport/バインディングヘルパー |
    | `plugin-sdk/process-runtime` | プロセスexecヘルパー |
    | `plugin-sdk/cli-runtime` | CLI整形、待機、およびバージョンヘルパー |
    | `plugin-sdk/gateway-runtime` | Gatewayクライアントおよびchannel-statusパッチヘルパー |
    | `plugin-sdk/config-runtime` | 設定の読み込み/書き込みヘルパー |
    | `plugin-sdk/telegram-command-config` | バンドルされたTelegram契約サーフェスが利用できない場合でも、Telegramコマンド名/説明の正規化および重複/競合チェック |
    | `plugin-sdk/text-autolink-runtime` | 広いtext-runtimeバレルを含まない、ファイル参照自動リンク検出 |
    | `plugin-sdk/approval-runtime` | exec/plugin承認ヘルパー、承認機能ビルダー、認可/プロファイルヘルパー、ネイティブルーティング/ランタイムヘルパー |
    | `plugin-sdk/reply-runtime` | 共有の受信/返信ランタイムヘルパー、チャンク化、ディスパッチ、Heartbeat、返信プランナー |
    | `plugin-sdk/reply-dispatch-runtime` | 狭い返信ディスパッチ/完了ヘルパー |
    | `plugin-sdk/reply-history` | `buildHistoryContext`、`recordPendingHistoryEntry`、`clearHistoryEntriesIfEnabled` のような共有短期ウィンドウ返信履歴ヘルパー |
    | `plugin-sdk/reply-reference` | `createReplyReferencePlanner` |
    | `plugin-sdk/reply-chunking` | 狭いテキスト/Markdownチャンク化ヘルパー |
    | `plugin-sdk/session-store-runtime` | セッションストアパス + updated-atヘルパー |
    | `plugin-sdk/state-paths` | state/OAuthディレクトリパスヘルパー |
    | `plugin-sdk/routing` | `resolveAgentRoute`、`buildAgentSessionKey`、`resolveDefaultAgentBoundAccountId` のようなルート/セッションキー/アカウントバインディングヘルパー |
    | `plugin-sdk/status-helpers` | 共有channel/accountステータスサマリーヘルパー、ランタイムstateデフォルト、およびissueメタデータヘルパー |
    | `plugin-sdk/target-resolver-runtime` | 共有ターゲットリゾルバヘルパー |
    | `plugin-sdk/string-normalization-runtime` | slug/文字列正規化ヘルパー |
    | `plugin-sdk/request-url` | fetch/request風入力から文字列URLを抽出 |
    | `plugin-sdk/run-command` | 正規化されたstdout/stderr結果を返す時間制限付きコマンドランナー |
    | `plugin-sdk/param-readers` | 共通tool/CLIパラメータリーダー |
    | `plugin-sdk/tool-payload` | tool結果オブジェクトから正規化済みペイロードを抽出 |
    | `plugin-sdk/tool-send` | tool引数から正規の送信ターゲットフィールドを抽出 |
    | `plugin-sdk/temp-path` | 共有一時ダウンロードパスヘルパー |
    | `plugin-sdk/logging-core` | サブシステムloggerおよびredactionヘルパー |
    | `plugin-sdk/markdown-table-runtime` | Markdown表モードヘルパー |
    | `plugin-sdk/json-store` | 小規模JSON state読み書きヘルパー |
    | `plugin-sdk/file-lock` | 再入可能ファイルロックヘルパー |
    | `plugin-sdk/persistent-dedupe` | ディスクバックdedupeキャッシュヘルパー |
    | `plugin-sdk/acp-runtime` | ACPランタイム/セッションおよびreply-dispatchヘルパー |
    | `plugin-sdk/acp-binding-resolve-runtime` | ライフサイクル起動importなしの読み取り専用ACPバインディング解決 |
    | `plugin-sdk/agent-config-primitives` | 狭いagentランタイム設定スキーマプリミティブ |
    | `plugin-sdk/boolean-param` | 緩いbooleanパラメータリーダー |
    | `plugin-sdk/dangerous-name-runtime` | 危険な名前のマッチング解決ヘルパー |
    | `plugin-sdk/device-bootstrap` | デバイスbootstrapおよびpairing tokenヘルパー |
    | `plugin-sdk/extension-shared` | 共有passive-channel、status、およびambient proxyヘルパープリミティブ |
    | `plugin-sdk/models-provider-runtime` | `/models` コマンド/provider返信ヘルパー |
    | `plugin-sdk/skill-commands-runtime` | skillコマンド一覧ヘルパー |
    | `plugin-sdk/native-command-registry` | ネイティブコマンドレジストリの構築/シリアライズヘルパー |
    | `plugin-sdk/agent-harness` | 低レベルagent harness向けの実験的trusted-pluginサーフェス: harness型、アクティブ実行のsteer/abortヘルパー、OpenClaw tool bridgeヘルパー、およびattempt resultユーティリティ |
    | `plugin-sdk/provider-zai-endpoint` | Z.A.Iエンドポイント検出ヘルパー |
    | `plugin-sdk/infra-runtime` | システムイベント/Heartbeatヘルパー |
    | `plugin-sdk/collection-runtime` | 小規模な有界キャッシュヘルパー |
    | `plugin-sdk/diagnostic-runtime` | 診断フラグおよびイベントヘルパー |
    | `plugin-sdk/error-runtime` | エラーグラフ、整形、共有エラー分類ヘルパー、`isApprovalNotFoundError` |
    | `plugin-sdk/fetch-runtime` | ラップされたfetch、proxy、およびpinned lookupヘルパー |
    | `plugin-sdk/runtime-fetch` | proxy/guarded-fetch importなしのdispatcher対応ランタイムfetch |
    | `plugin-sdk/response-limit-runtime` | 広いmediaランタイムサーフェスを含まない有界レスポンス本文リーダー |
    | `plugin-sdk/session-binding-runtime` | 設定済みbindingルーティングやpairingストアを含まない現在の会話binding state |
    | `plugin-sdk/session-store-runtime` | 広い設定書き込み/保守importを含まないセッションストア読み取りヘルパー |
    | `plugin-sdk/context-visibility-runtime` | 広い設定/セキュリティimportを含まないコンテキスト可視性解決および補助コンテキストフィルタリング |
    | `plugin-sdk/string-coerce-runtime` | Markdown/ロギングimportを含まない、狭いプリミティブrecord/文字列の強制変換および正規化ヘルパー |
    | `plugin-sdk/host-runtime` | ホスト名およびSCPホスト正規化ヘルパー |
    | `plugin-sdk/retry-runtime` | retry設定およびretry実行ヘルパー |
    | `plugin-sdk/agent-runtime` | agentディレクトリ/identity/workspaceヘルパー |
    | `plugin-sdk/directory-runtime` | 設定バックのディレクトリ照会/dedupe |
    | `plugin-sdk/keyed-async-queue` | `KeyedAsyncQueue` |
  </Accordion>

  <Accordion title="機能とテストのサブパス">
    | Subpath | 主なエクスポート |
    | --- | --- |
    | `plugin-sdk/media-runtime` | メディアペイロードビルダーに加え、共有メディアfetch/変換/保存ヘルパー |
    | `plugin-sdk/media-generation-runtime` | 共有メディア生成failoverヘルパー、候補選択、および不足モデルメッセージ |
    | `plugin-sdk/media-understanding` | メディア理解provider型と、provider向け画像/音声ヘルパーエクスポート |
    | `plugin-sdk/text-runtime` | アシスタント可視テキストの除去、Markdownレンダリング/チャンク化/表ヘルパー、redactionヘルパー、directive-tagヘルパー、安全なテキストユーティリティなどの共有テキスト/Markdown/ロギングヘルパー |
    | `plugin-sdk/text-chunking` | 送信テキストチャンク化ヘルパー |
    | `plugin-sdk/speech` | 音声provider型と、provider向けdirective、registry、および検証ヘルパー |
    | `plugin-sdk/speech-core` | 共有音声provider型、registry、directive、および正規化ヘルパー |
    | `plugin-sdk/realtime-transcription` | リアルタイム文字起こしprovider型とregistryヘルパー |
    | `plugin-sdk/realtime-voice` | リアルタイム音声provider型とregistryヘルパー |
    | `plugin-sdk/image-generation` | 画像生成provider型 |
    | `plugin-sdk/image-generation-core` | 共有画像生成型、failover、認証、およびregistryヘルパー |
    | `plugin-sdk/music-generation` | 音楽生成provider/request/result型 |
    | `plugin-sdk/music-generation-core` | 共有音楽生成型、failoverヘルパー、provider検索、およびmodel-ref解析 |
    | `plugin-sdk/video-generation` | 動画生成provider/request/result型 |
    | `plugin-sdk/video-generation-core` | 共有動画生成型、failoverヘルパー、provider検索、およびmodel-ref解析 |
    | `plugin-sdk/webhook-targets` | Webhookターゲットregistryおよびroute-installヘルパー |
    | `plugin-sdk/webhook-path` | Webhookパス正規化ヘルパー |
    | `plugin-sdk/web-media` | 共有リモート/ローカルメディア読み込みヘルパー |
    | `plugin-sdk/zod` | plugin SDK利用者向けに再エクスポートされた `zod` |
    | `plugin-sdk/testing` | `installCommonResolveTargetErrorCases`、`shouldAckReaction` |
  </Accordion>

  <Accordion title="メモリのサブパス">
    | Subpath | 主なエクスポート |
    | --- | --- |
    | `plugin-sdk/memory-core` | manager/config/file/CLIヘルパー向けのバンドルされたmemory-coreヘルパーサーフェス |
    | `plugin-sdk/memory-core-engine-runtime` | メモリインデックス/検索ランタイムファサード |
    | `plugin-sdk/memory-core-host-engine-foundation` | メモリホストfoundationエンジンエクスポート |
    | `plugin-sdk/memory-core-host-engine-embeddings` | メモリホスト埋め込み契約、registryアクセス、ローカルprovider、および汎用バッチ/リモートヘルパー |
    | `plugin-sdk/memory-core-host-engine-qmd` | メモリホストQMDエンジンエクスポート |
    | `plugin-sdk/memory-core-host-engine-storage` | メモリホストストレージエンジンエクスポート |
    | `plugin-sdk/memory-core-host-multimodal` | メモリホストマルチモーダルヘルパー |
    | `plugin-sdk/memory-core-host-query` | メモリホストクエリヘルパー |
    | `plugin-sdk/memory-core-host-secret` | メモリホストsecretヘルパー |
    | `plugin-sdk/memory-core-host-events` | メモリホストイベントジャーナルヘルパー |
    | `plugin-sdk/memory-core-host-status` | メモリホストステータスヘルパー |
    | `plugin-sdk/memory-core-host-runtime-cli` | メモリホストCLIランタイムヘルパー |
    | `plugin-sdk/memory-core-host-runtime-core` | メモリホストコアランタイムヘルパー |
    | `plugin-sdk/memory-core-host-runtime-files` | メモリホストファイル/ランタイムヘルパー |
    | `plugin-sdk/memory-host-core` | メモリホストコアランタイムヘルパーのベンダー中立エイリアス |
    | `plugin-sdk/memory-host-events` | メモリホストイベントジャーナルヘルパーのベンダー中立エイリアス |
    | `plugin-sdk/memory-host-files` | メモリホストファイル/ランタイムヘルパーのベンダー中立エイリアス |
    | `plugin-sdk/memory-host-markdown` | メモリ隣接plugin向けの共有managed-markdownヘルパー |
    | `plugin-sdk/memory-host-search` | 検索managerアクセス向けのActive Memoryランタイムファサード |
    | `plugin-sdk/memory-host-status` | メモリホストステータスヘルパーのベンダー中立エイリアス |
    | `plugin-sdk/memory-lancedb` | バンドルされたmemory-lancedbヘルパーサーフェス |
  </Accordion>

  <Accordion title="予約済みのバンドルhelperサブパス">
    | Family | 現在のサブパス | 想定用途 |
    | --- | --- | --- |
    | Browser | `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support` | バンドルされたbrowser pluginサポートヘルパー（`browser-support` は互換性バレルのままです） |
    | Matrix | `plugin-sdk/matrix`, `plugin-sdk/matrix-helper`, `plugin-sdk/matrix-runtime-heavy`, `plugin-sdk/matrix-runtime-shared`, `plugin-sdk/matrix-runtime-surface`, `plugin-sdk/matrix-surface`, `plugin-sdk/matrix-thread-bindings` | バンドルされたMatrixヘルパー/ランタイムサーフェス |
    | Line | `plugin-sdk/line`, `plugin-sdk/line-core`, `plugin-sdk/line-runtime`, `plugin-sdk/line-surface` | バンドルされたLINEヘルパー/ランタイムサーフェス |
    | IRC | `plugin-sdk/irc`, `plugin-sdk/irc-surface` | バンドルされたIRCヘルパーサーフェス |
    | Channel固有ヘルパー | `plugin-sdk/googlechat`, `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles`, `plugin-sdk/bluebubbles-policy`, `plugin-sdk/mattermost`, `plugin-sdk/mattermost-policy`, `plugin-sdk/feishu-conversation`, `plugin-sdk/msteams`, `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`, `plugin-sdk/twitch` | バンドルされたchannel互換性/helperシーム |
    | 認証/plugin固有ヘルパー | `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`, `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`, `plugin-sdk/thread-ownership`, `plugin-sdk/voice-call` | バンドルされた機能/pluginヘルパーシーム。`plugin-sdk/github-copilot-token` は現在 `DEFAULT_COPILOT_API_BASE_URL`、`deriveCopilotApiBaseUrlFromToken`、`resolveCopilotApiToken` をエクスポートします |
  </Accordion>
</AccordionGroup>

## 登録API

`register(api)` コールバックは、次のメソッドを持つ `OpenClawPluginApi` オブジェクトを受け取ります。

### 機能登録

| メソッド                                           | 登録するもの                          |
| -------------------------------------------------- | ------------------------------------- |
| `api.registerProvider(...)`                        | テキスト推論（LLM）                   |
| `api.registerAgentHarness(...)`                    | 実験的な低レベルagent実行器           |
| `api.registerCliBackend(...)`                      | ローカルCLI推論バックエンド           |
| `api.registerChannel(...)`                         | メッセージングchannel                 |
| `api.registerSpeechProvider(...)`                  | テキスト読み上げ / STT合成            |
| `api.registerRealtimeTranscriptionProvider(...)`   | ストリーミングのリアルタイム文字起こし |
| `api.registerRealtimeVoiceProvider(...)`           | 双方向リアルタイム音声セッション      |
| `api.registerMediaUnderstandingProvider(...)`      | 画像/音声/動画解析                    |
| `api.registerImageGenerationProvider(...)`         | 画像生成                              |
| `api.registerMusicGenerationProvider(...)`         | 音楽生成                              |
| `api.registerVideoGenerationProvider(...)`         | 動画生成                              |
| `api.registerWebFetchProvider(...)`                | Web fetch / スクレイプprovider        |
| `api.registerWebSearchProvider(...)`               | Web検索                               |

### ツールとコマンド

| メソッド                          | 登録するもの                                 |
| --------------------------------- | -------------------------------------------- |
| `api.registerTool(tool, opts?)`   | agentツール（必須、または `{ optional: true }`） |
| `api.registerCommand(def)`        | カスタムコマンド（LLMをバイパスする）        |

### インフラストラクチャ

| メソッド                                         | 登録するもの                          |
| ------------------------------------------------ | ------------------------------------- |
| `api.registerHook(events, handler, opts?)`       | イベントhook                          |
| `api.registerHttpRoute(params)`                  | Gateway HTTPエンドポイント            |
| `api.registerGatewayMethod(name, handler)`       | Gateway RPCメソッド                   |
| `api.registerCli(registrar, opts?)`              | CLIサブコマンド                       |
| `api.registerService(service)`                   | バックグラウンドサービス              |
| `api.registerInteractiveHandler(registration)`   | 対話型ハンドラー                      |
| `api.registerMemoryPromptSupplement(builder)`    | 追加的なメモリ隣接プロンプトセクション |
| `api.registerMemoryCorpusSupplement(adapter)`    | 追加的なメモリ検索/読み取りコーパス   |

予約済みのコア管理namespace（`config.*`、`exec.approvals.*`、`wizard.*`、
`update.*`）は、pluginがより狭いGatewayメソッドスコープを割り当てようとしても、
常に `operator.admin` のままです。plugin所有メソッドには、plugin固有のprefixを
優先してください。

### CLI登録メタデータ

`api.registerCli(registrar, opts?)` は、2種類のトップレベルメタデータを受け取ります：

- `commands`: registrarが所有する明示的なコマンドルート
- `descriptors`: ルートCLIヘルプ、ルーティング、および遅延plugin CLI登録に使われる、解析時コマンド記述子

通常のルートCLIパスでpluginコマンドを遅延ロードのままにしたい場合は、
そのregistrarが公開するすべてのトップレベルコマンドルートをカバーする
`descriptors` を指定してください。

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
        description: "Matrixアカウント、検証、デバイス、およびプロファイル状態を管理する",
        hasSubcommands: true,
      },
    ],
  },
);
```

ルートCLIの遅延登録が不要な場合にのみ、`commands` を単独で使ってください。
この即時互換パスは引き続きサポートされますが、解析時の遅延ロード用に
descriptorベースのプレースホルダーはインストールされません。

### CLIバックエンド登録

`api.registerCliBackend(...)` により、pluginは `codex-cli` のようなローカル
AI CLIバックエンドのデフォルト設定を所有できます。

- バックエンドの `id` は、`codex-cli/gpt-5` のようなmodel refにおけるprovider prefixになります。
- バックエンドの `config` は `agents.defaults.cliBackends.<id>` と同じ形を使います。
- ユーザー設定が引き続き優先されます。OpenClawはCLIを実行する前に、
  pluginデフォルトの上に `agents.defaults.cliBackends.<id>` をマージします。
- マージ後にバックエンドで互換性書き換えが必要な場合
  （たとえば古いフラグ形状の正規化など）は、`normalizeConfig` を使ってください。

### 排他的スロット

| メソッド                                     | 登録するもの                                                                                                                                                 |
| -------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `api.registerContextEngine(id, factory)`     | コンテキストエンジン（一度に1つだけアクティブ）。`assemble()` コールバックは `availableTools` と `citationsMode` を受け取り、エンジンがプロンプト追加内容を調整できるようにします。 |
| `api.registerMemoryCapability(capability)`   | 統一メモリ機能                                                                                                                                               |
| `api.registerMemoryPromptSection(builder)`   | メモリプロンプトセクションビルダー                                                                                                                           |
| `api.registerMemoryFlushPlan(resolver)`      | メモリフラッシュプランリゾルバ                                                                                                                               |
| `api.registerMemoryRuntime(runtime)`         | メモリランタイムアダプタ                                                                                                                                     |

### メモリ埋め込みアダプタ

| メソッド                                         | 登録するもの                                   |
| ------------------------------------------------ | ---------------------------------------------- |
| `api.registerMemoryEmbeddingProvider(adapter)`   | アクティブplugin向けメモリ埋め込みアダプタ     |

- `registerMemoryCapability` は、推奨される排他的メモリplugin APIです。
- `registerMemoryCapability` は `publicArtifacts.listArtifacts(...)` も公開でき、
  companion pluginはそれにより、特定のメモリpluginの非公開レイアウトに入り込まずに
  `openclaw/plugin-sdk/memory-host-core` 経由でエクスポートされたメモリアーティファクトを利用できます。
- `registerMemoryPromptSection`、`registerMemoryFlushPlan`、および
  `registerMemoryRuntime` は、旧来互換の排他的メモリplugin APIです。
- `registerMemoryEmbeddingProvider` により、アクティブなメモリpluginは1つ以上の
  埋め込みアダプタid（たとえば `openai`、`gemini`、またはplugin定義のカスタムid）を登録できます。
- `agents.defaults.memorySearch.provider` や
  `agents.defaults.memorySearch.fallback` のようなユーザー設定は、
  それらの登録済みアダプタidに対して解決されます。

### イベントとライフサイクル

| メソッド                                       | 動作 |
| ---------------------------------------------- | ---- |
| `api.on(hookName, handler, opts?)`             | 型付きライフサイクルhook |
| `api.onConversationBindingResolved(handler)`   | 会話bindingコールバック |

### Hook判定セマンティクス

- `before_tool_call`: `{ block: true }` を返すと終端になります。いずれかのハンドラーがこれを設定すると、それより低優先度のハンドラーはスキップされます。
- `before_tool_call`: `{ block: false }` を返しても未決定として扱われます（`block` を省略した場合と同じ）ので、上書きにはなりません。
- `before_install`: `{ block: true }` を返すと終端になります。いずれかのハンドラーがこれを設定すると、それより低優先度のハンドラーはスキップされます。
- `before_install`: `{ block: false }` を返しても未決定として扱われます（`block` を省略した場合と同じ）ので、上書きにはなりません。
- `reply_dispatch`: `{ handled: true, ... }` を返すと終端になります。いずれかのハンドラーがディスパッチを引き受けると、それより低優先度のハンドラーとデフォルトのモデルディスパッチ経路はスキップされます。
- `message_sending`: `{ cancel: true }` を返すと終端になります。いずれかのハンドラーがこれを設定すると、それより低優先度のハンドラーはスキップされます。
- `message_sending`: `{ cancel: false }` を返しても未決定として扱われます（`cancel` を省略した場合と同じ）ので、上書きにはなりません。

### APIオブジェクトのフィールド

| Field | Type | 説明 |
| ------------------------ | ------------------------- | ------------------------------------------------------------------------------------------- |
| `api.id`                 | `string`                  | Plugin id |
| `api.name`               | `string`                  | 表示名 |
| `api.version`            | `string?`                 | Pluginバージョン（任意） |
| `api.description`        | `string?`                 | Plugin説明（任意） |
| `api.source`             | `string`                  | Pluginソースパス |
| `api.rootDir`            | `string?`                 | Pluginルートディレクトリ（任意） |
| `api.config`             | `OpenClawConfig`          | 現在の設定スナップショット（利用可能な場合は、アクティブなメモリ内ランタイムスナップショット） |
| `api.pluginConfig`       | `Record<string, unknown>` | `plugins.entries.<id>.config` にあるplugin固有設定 |
| `api.runtime`            | `PluginRuntime`           | [ランタイムヘルパー](/ja-JP/plugins/sdk-runtime) |
| `api.logger`             | `PluginLogger`            | スコープ付きlogger（`debug`、`info`、`warn`、`error`） |
| `api.registrationMode`   | `PluginRegistrationMode`  | 現在のロードモード。`"setup-runtime"` は、完全なエントリ起動/セットアップ前の軽量ウィンドウです |
| `api.resolvePath(input)` | `(string) => string`      | pluginルート基準でパスを解決する |

## 内部モジュール規約

plugin内では、内部インポートにローカルバレルファイルを使ってください：

```
my-plugin/
  api.ts            # 外部利用者向けの公開エクスポート
  runtime-api.ts    # 内部専用ランタイムエクスポート
  index.ts          # Pluginエントリポイント
  setup-entry.ts    # 軽量なセットアップ専用エントリ（任意）
```

<Warning>
  本番コード内で、自分自身のpluginを `openclaw/plugin-sdk/<your-plugin>`
  経由でインポートしてはいけません。内部インポートは `./api.ts` または
  `./runtime-api.ts` を通してください。SDKパスは外部契約専用です。
</Warning>

ファサードロードされるバンドルplugin公開サーフェス（`api.ts`、`runtime-api.ts`、
`index.ts`、`setup-entry.ts`、および同様の公開エントリファイル）は、OpenClawが
すでに実行中であれば、現在のランタイム設定スナップショットを優先するように
なりました。まだランタイムスナップショットが存在しない場合は、ディスク上の
解決済み設定ファイルにフォールバックします。

Provider pluginは、helperが意図的にprovider固有で、まだ汎用SDKサブパスに属さない場合、狭いpluginローカル契約バレルを公開することもできます。現在のバンドル例として、Anthropic providerはClaudeストリームヘルパーを、自身の公開 `api.ts` / `contract-api.ts` シームに保持しており、Anthropicのベータヘッダーや `service_tier` ロジックを汎用の `plugin-sdk/*` 契約へ昇格させていません。

現在のその他のバンドル例：

- `@openclaw/openai-provider`: `api.ts` はproviderビルダー、
  デフォルトモデルヘルパー、およびリアルタイムproviderビルダーをエクスポートします
- `@openclaw/openrouter-provider`: `api.ts` はproviderビルダーに加えて
  オンボーディング/設定ヘルパーをエクスポートします

<Warning>
  extensionの本番コードでも、`openclaw/plugin-sdk/<other-plugin>` の
  インポートは避けるべきです。helperが本当に共有されるべきものなら、2つのpluginを
  結合させるのではなく、`openclaw/plugin-sdk/speech`、
  `.../provider-model-shared`、または別の機能指向サーフェスのような
  中立的なSDKサブパスへ昇格させてください。
</Warning>

## 関連

- [Entry Points](/ja-JP/plugins/sdk-entrypoints) — `definePluginEntry` と `defineChannelPluginEntry` のオプション
- [ランタイムヘルパー](/ja-JP/plugins/sdk-runtime) — 完全な `api.runtime` namespaceリファレンス
- [セットアップと設定](/ja-JP/plugins/sdk-setup) — パッケージング、manifest、設定スキーマ
- [Testing](/ja-JP/plugins/sdk-testing) — テストユーティリティとlintルール
- [SDK Migration](/ja-JP/plugins/sdk-migration) — 非推奨サーフェスからの移行
- [Plugin Internals](/ja-JP/plugins/architecture) — 詳細なアーキテクチャと機能モデル
