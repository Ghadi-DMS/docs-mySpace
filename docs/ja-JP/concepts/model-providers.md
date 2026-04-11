---
read_when:
    - プロバイダーごとのモデル設定リファレンスが必要です
    - モデルプロバイダー向けの設定例やCLIオンボーディングコマンドが必要です
summary: モデルプロバイダーの概要と設定例 + CLIフロー
title: モデルプロバイダー
x-i18n:
    generated_at: "2026-04-11T02:44:15Z"
    model: gpt-5.4
    provider: openai
    source_hash: 910ea7895e74c03910757d9d3e02825754b779b204eca7275b28422647ed0151
    source_path: concepts/model-providers.md
    workflow: 15
---

# モデルプロバイダー

このページでは、**LLM/モデルプロバイダー**（WhatsApp/Telegramのようなチャットチャネルではありません）を扱います。
モデル選択ルールについては、[/concepts/models](/ja-JP/concepts/models)を参照してください。

## クイックルール

- モデル参照は`provider/model`を使用します（例: `opencode/claude-opus-4-6`）。
- `agents.defaults.models`を設定すると、それが許可リストになります。
- CLIヘルパー: `openclaw onboard`、`openclaw models list`、`openclaw models set <provider/model>`。
- フォールバックの実行時ルール、クールダウンプローブ、セッション上書きの永続化は、[/concepts/model-failover](/ja-JP/concepts/model-failover)に記載されています。
- `models.providers.*.models[].contextWindow`はネイティブなモデルメタデータです。
  `models.providers.*.models[].contextTokens`は実行時に有効な上限です。
- プロバイダープラグインは`registerProvider({ catalog })`を通じてモデルカタログを注入できます。
  OpenClawはその出力を`models.providers`にマージしてから
  `models.json`を書き込みます。
- プロバイダーマニフェストは`providerAuthEnvVars`と
  `providerAuthAliases`を宣言できるため、汎用の環境変数ベース認証プローブやプロバイダーバリアントで
  プラグインランタイムを読み込む必要がありません。残っているコアの環境変数マップは現在、
  非プラグイン/コアプロバイダーと、AnthropicのAPIキー優先オンボーディングのような
  いくつかの汎用優先順位ケースのためだけに使われています。
- プロバイダープラグインは、以下を通じてプロバイダーの実行時動作も所有できます:
  `normalizeModelId`, `normalizeTransport`, `normalizeConfig`,
  `applyNativeStreamingUsageCompat`, `resolveConfigApiKey`,
  `resolveSyntheticAuth`, `shouldDeferSyntheticProfileAuth`,
  `resolveDynamicModel`, `prepareDynamicModel`,
  `normalizeResolvedModel`, `contributeResolvedModelCompat`,
  `capabilities`, `normalizeToolSchemas`,
  `inspectToolSchemas`, `resolveReasoningOutputMode`,
  `prepareExtraParams`, `createStreamFn`, `wrapStreamFn`,
  `resolveTransportTurnState`, `resolveWebSocketSessionPolicy`,
  `createEmbeddingProvider`, `formatApiKey`, `refreshOAuth`,
  `buildAuthDoctorHint`,
  `matchesContextOverflowError`, `classifyFailoverReason`,
  `isCacheTtlEligible`, `buildMissingAuthMessage`, `suppressBuiltInModel`,
  `augmentModelCatalog`, `isBinaryThinking`, `supportsXHighThinking`,
  `resolveDefaultThinkingLevel`, `applyConfigDefaults`, `isModernModelRef`,
  `prepareRuntimeAuth`, `resolveUsageAuth`, `fetchUsageSnapshot`, and
  `onModelSelected`。
- 注: プロバイダー実行時の`capabilities`は共有ランナーのメタデータです（プロバイダーファミリー、文字起こし/ツールの癖、トランスポート/キャッシュのヒント）。
  これは、プラグインが何を登録するか（テキスト推論、音声など）を説明する
  [公開 capability model](/ja-JP/plugins/architecture#public-capability-model)
  とは同じではありません。
- バンドル版の`codex`プロバイダーは、バンドル版Codexエージェントハーネスと組みになっています。
  Codexが所有するログイン、モデル検出、ネイティブなスレッド再開、
  アプリサーバー実行を使いたい場合は、`codex/gpt-*`を使用してください。通常の`openai/gpt-*`参照は引き続き
  OpenAIプロバイダーと通常のOpenClawプロバイダートランスポートを使用します。
  Codex専用デプロイでは、自動PIフォールバックを
  `agents.defaults.embeddedHarness.fallback: "none"`で無効にできます。詳細は
  [Codex Harness](/ja-JP/plugins/codex-harness)を参照してください。

## プラグイン所有のプロバイダー動作

OpenClawが汎用推論ループを維持しつつ、
プロバイダープラグインがプロバイダー固有ロジックの大半を所有できるようになりました。

一般的な分担:

- `auth[].run` / `auth[].runNonInteractive`: プロバイダーが`openclaw onboard`、`openclaw models auth`、ヘッドレス設定向けのオンボーディング/ログイン
  フローを所有
- `wizard.setup` / `wizard.modelPicker`: プロバイダーが認証選択ラベル、
  レガシーエイリアス、オンボーディング許可リストのヒント、オンボーディング/モデルピッカー内のセットアップ項目を所有
- `catalog`: プロバイダーが`models.providers`に表示される
- `normalizeModelId`: プロバイダーが、検索または正規化の前に
  レガシー/プレビューモデルIDを正規化する
- `normalizeTransport`: プロバイダーが、汎用モデル組み立ての前にトランスポートファミリーの`api` / `baseUrl`を正規化する。
  OpenClawはまず一致したプロバイダーを確認し、
  その後、実際にトランスポートを変更するものが見つかるまで、他のフック対応プロバイダープラグインを確認します
- `normalizeConfig`: プロバイダーが、ランタイムで使用する前に`models.providers.<id>`設定を正規化する。
  OpenClawはまず一致したプロバイダーを確認し、その後、
  実際に設定を変更するものが見つかるまで、他のフック対応プロバイダープラグインを確認します。どの
  プロバイダーフックも設定を書き換えない場合でも、バンドル版のGoogleファミリーヘルパーは引き続き
  サポート対象のGoogleプロバイダーエントリーを正規化します。
- `applyNativeStreamingUsageCompat`: プロバイダーが、設定プロバイダー向けにエンドポイント主導のネイティブストリーミング使用量互換リライトを適用する
- `resolveConfigApiKey`: プロバイダーが、完全なランタイム認証の読み込みを強制せずに
  設定プロバイダー向けの環境マーカー認証を解決する。
  `amazon-bedrock`にはここにAWS環境マーカー用の組み込みリゾルバーもありますが、
  Bedrockランタイム認証自体はAWS SDKのデフォルトチェーンを使用します。
- `resolveSyntheticAuth`: プロバイダーが、平文のシークレットを永続化せずに
  ローカル/セルフホスト型またはその他の設定ベース認証の可用性を公開できる
- `shouldDeferSyntheticProfileAuth`: プロバイダーが、保存された合成プロファイル
  プレースホルダーを、環境変数/設定ベース認証より低い優先順位としてマークできる
- `resolveDynamicModel`: プロバイダーが、まだローカルの
  静的カタログに存在しないモデルIDを受け入れる
- `prepareDynamicModel`: プロバイダーが、動的解決の再試行前に
  メタデータ更新を必要とする
- `normalizeResolvedModel`: プロバイダーが、トランスポートまたはベースURLの書き換えを必要とする
- `contributeResolvedModelCompat`: プロバイダーが、別の互換トランスポート経由で届いた場合でも、
  自社ベンダーモデル向けの互換フラグを提供する
- `capabilities`: プロバイダーが文字起こし/ツール/プロバイダーファミリーの癖を公開する
- `normalizeToolSchemas`: プロバイダーが、埋め込みランナーが参照する前に
  ツールスキーマを整える
- `inspectToolSchemas`: プロバイダーが、正規化後に
  トランスポート固有のスキーマ警告を提示する
- `resolveReasoningOutputMode`: プロバイダーが、ネイティブとタグ付きの
  reasoning出力契約を選択する
- `prepareExtraParams`: プロバイダーが、モデルごとのリクエストパラメータをデフォルト設定または正規化する
- `createStreamFn`: プロバイダーが通常のストリーム経路を
  完全なカスタムトランスポートに置き換える
- `wrapStreamFn`: プロバイダーがリクエストヘッダー/ボディ/モデル互換ラッパーを適用する
- `resolveTransportTurnState`: プロバイダーが、ターンごとのネイティブトランスポート
  ヘッダーまたはメタデータを提供する
- `resolveWebSocketSessionPolicy`: プロバイダーが、ネイティブWebSocketセッション
  ヘッダーまたはセッションクールダウンポリシーを提供する
- `createEmbeddingProvider`: プロバイダーが、コアの埋め込みスイッチボードではなく
  プロバイダープラグインに属するメモリ埋め込み動作を所有する
- `formatApiKey`: プロバイダーが、保存された認証プロファイルを
  トランスポートが期待する実行時`apiKey`文字列へ整形する
- `refreshOAuth`: プロバイダーが、共有の`pi-ai`
  リフレッシャーでは不十分な場合にOAuth更新を所有する
- `buildAuthDoctorHint`: プロバイダーが、OAuth更新に失敗したときに
  修復ガイダンスを追加する
- `matchesContextOverflowError`: プロバイダーが、汎用ヒューリスティクスでは見逃される
  プロバイダー固有のコンテキストウィンドウ超過エラーを認識する
- `classifyFailoverReason`: プロバイダーが、プロバイダー固有の生のトランスポート/API
  エラーを、レート制限や過負荷などのフェイルオーバー理由に対応付ける
- `isCacheTtlEligible`: プロバイダーが、どの上流モデルIDがプロンプトキャッシュTTLをサポートするかを判断する
- `buildMissingAuthMessage`: プロバイダーが、汎用認証ストアエラーを
  プロバイダー固有の復旧ヒントに置き換える
- `suppressBuiltInModel`: プロバイダーが、古くなった上流行を非表示にし、
  直接解決の失敗時にはベンダー所有のエラーを返せる
- `augmentModelCatalog`: プロバイダーが、検出と設定マージの後に
  合成/最終カタログ行を追加する
- `isBinaryThinking`: プロバイダーが、二値のオン/オフthinking UXを所有する
- `supportsXHighThinking`: プロバイダーが、選択したモデルで`xhigh`を有効にする
- `resolveDefaultThinkingLevel`: プロバイダーが、モデルファミリー向けの
  デフォルト`/think`ポリシーを所有する
- `applyConfigDefaults`: プロバイダーが、認証モード、環境変数、またはモデルファミリーに基づいて、
  設定具体化中にプロバイダー固有のグローバルデフォルトを適用する
- `isModernModelRef`: プロバイダーが、ライブ/スモーク向けの推奨モデル照合を所有する
- `prepareRuntimeAuth`: プロバイダーが、設定済み資格情報を
  短命な実行時トークンに変換する
- `resolveUsageAuth`: プロバイダーが、`/usage`
  や関連するステータス/レポート画面向けの使用量/クォータ資格情報を解決する
- `fetchUsageSnapshot`: プロバイダーが使用量エンドポイントの取得/解析を所有し、
  コアは引き続き概要の外枠と整形を所有する
- `onModelSelected`: プロバイダーが、テレメトリやプロバイダー所有セッション管理のような
  モデル選択後の副作用を実行する

現在のバンドル例:

- `anthropic`: Claude 4.6の前方互換フォールバック、認証修復ヒント、使用量
  エンドポイントの取得、キャッシュTTL/プロバイダーファミリーのメタデータ、および認証を考慮したグローバル
  設定デフォルト
- `amazon-bedrock`: Bedrock固有のスロットリング/未準備エラー向けに、プロバイダー所有のコンテキスト超過一致と
  フェイルオーバー理由分類を提供し、さらに
  Claude専用のリプレイポリシー保護のための共有`anthropic-by-model`リプレイファミリーを
  Anthropicトラフィックに適用
- `anthropic-vertex`: Anthropicメッセージ
  トラフィック向けのClaude専用リプレイポリシー保護
- `openrouter`: モデルIDのパススルー、リクエストラッパー、プロバイダー capability
  ヒント、プロキシGeminiトラフィック上でのGemini thought-signatureサニタイズ、
  `openrouter-thinking`ストリームファミリー経由のプロキシreasoning注入、ルーティング
  メタデータの転送、およびキャッシュTTLポリシー
- `github-copilot`: オンボーディング/デバイスログイン、前方互換モデルフォールバック、
  Claude-thinking文字起こしヒント、ランタイムトークン交換、および使用量エンドポイント
  の取得
- `openai`: GPT-5.4の前方互換フォールバック、直接OpenAIトランスポート
  正規化、Codex対応の認証不足ヒント、Sparkの抑制、合成
  OpenAI/Codexカタログ行、thinking/ライブモデルポリシー、使用量トークンエイリアス
  正規化（`input` / `output`および`prompt` / `completion`ファミリー）、ネイティブOpenAI/Codex
  ラッパー向けの共有`openai-responses-defaults`ストリームファミリー、
  プロバイダーファミリーのメタデータ、`gpt-image-1`向けのバンドル版画像生成プロバイダー
  登録、および`sora-2`向けのバンドル版動画生成プロバイダー
  登録
- `google`および`google-gemini-cli`: Gemini 3.1の前方互換フォールバック、
  ネイティブGeminiリプレイ検証、ブートストラップリプレイサニタイズ、タグ付き
  reasoning出力モード、モダンモデル照合、Gemini image-previewモデル向けのバンドル版画像生成
  プロバイダー登録、およびVeoモデル向けのバンドル版
  動画生成プロバイダー登録。Gemini CLI OAuthはさらに、
  使用量画面向けの認証プロファイルトークン整形、使用量トークン解析、クォータエンドポイント
  取得も所有します
- `moonshot`: 共有トランスポート、プラグイン所有のthinkingペイロード正規化
- `kilocode`: 共有トランスポート、プラグイン所有のリクエストヘッダー、reasoningペイロード
  正規化、プロキシGemini thought-signatureサニタイズ、およびキャッシュTTL
  ポリシー
- `zai`: GLM-5の前方互換フォールバック、`tool_stream`デフォルト、キャッシュTTL
  ポリシー、二値thinking/ライブモデルポリシー、および使用量認証 + クォータ取得。
  不明な`glm-5*` IDは、バンドル版`glm-4.7`テンプレートから合成されます
- `xai`: ネイティブResponsesトランスポート正規化、Grok高速バリアント向けの
  `/fast`エイリアス書き換え、デフォルト`tool_stream`、xAI固有のツールスキーマ /
  reasoningペイロード整理、および`grok-imagine-video`向けのバンドル版動画生成プロバイダー
  登録
- `mistral`: プラグイン所有の capabilityメタデータ
- `opencode`および`opencode-go`: プラグイン所有の capabilityメタデータに加え、
  プロキシGemini thought-signatureサニタイズ
- `alibaba`: `alibaba/wan2.6-t2v`のような直接Wanモデル参照向けの
  プラグイン所有動画生成カタログ
- `byteplus`: プラグイン所有カタログに加え、Seedanceのテキストから動画/画像から動画モデル向けの
  バンドル版動画生成プロバイダー登録
- `fal`: ホスト型サードパーティ動画モデル向けの
  バンドル版動画生成プロバイダー登録、およびFLUX画像モデル向けのホスト型サードパーティ画像生成プロバイダー
  登録に加え、ホスト型サードパーティ動画モデル向けのバンドル版
  動画生成プロバイダー登録
- `cloudflare-ai-gateway`, `huggingface`, `kimi`, `nvidia`, `qianfan`,
  `stepfun`, `synthetic`, `venice`, `vercel-ai-gateway`, および`volcengine`:
  プラグイン所有カタログのみ
- `qwen`: テキストモデル向けのプラグイン所有カタログに加え、その
  マルチモーダル画面向けの共有media-understandingおよび動画生成プロバイダー登録。
  Qwen動画生成は、`wan2.6-t2v`や`wan2.7-r2v`のような
  バンドル版Wanモデルを伴う標準DashScope動画エンドポイントを使用します
- `runway`: `gen4.5`のようなネイティブ
  Runwayタスクベースモデル向けのプラグイン所有動画生成プロバイダー登録
- `minimax`: プラグイン所有カタログ、Hailuo動画モデル向けのバンドル版動画生成プロバイダー
  登録、`image-01`向けのバンドル版画像生成プロバイダー
  登録、ハイブリッドAnthropic/OpenAIリプレイポリシー
  選択、および使用量認証/スナップショットロジック
- `together`: プラグイン所有カタログに加え、Wan動画モデル向けのバンドル版動画生成プロバイダー
  登録
- `xiaomi`: プラグイン所有カタログに加え、使用量認証/スナップショットロジック

バンドル版`openai`プラグインは現在、両方のプロバイダーIDである`openai`と
`openai-codex`を所有します。

これで、OpenClawの通常トランスポートにまだ適合するプロバイダーを網羅しています。完全にカスタムのリクエスト実行子を必要とするプロバイダーは、
別の、より深い拡張サーフェスになります。

## APIキーのローテーション

- 選択されたプロバイダー向けの汎用プロバイダーローテーションをサポートします。
- 複数キーは以下で設定します:
  - `OPENCLAW_LIVE_<PROVIDER>_KEY`（単一のライブ上書き、最優先）
  - `<PROVIDER>_API_KEYS`（カンマまたはセミコロン区切りのリスト）
  - `<PROVIDER>_API_KEY`（プライマリキー）
  - `<PROVIDER>_API_KEY_*`（番号付きリスト、例: `<PROVIDER>_API_KEY_1`）
- Googleプロバイダーでは、`GOOGLE_API_KEY`もフォールバックとして含まれます。
- キーの選択順序は優先順位を維持し、値の重複を排除します。
- リクエストは、レート制限レスポンスのときだけ次のキーで再試行されます（例:
  `429`, `rate_limit`, `quota`, `resource exhausted`, `Too many
concurrent requests`, `ThrottlingException`, `concurrency limit reached`,
  `workers_ai ... quota limit exceeded`、または定期的な使用量制限メッセージ）。
- レート制限以外の失敗は即座に失敗し、キーのローテーションは試行されません。
- すべての候補キーが失敗した場合、最後の試行の最終エラーが返されます。

## 組み込みプロバイダー（pi-aiカタログ）

OpenClawにはpi‑aiカタログが同梱されています。これらのプロバイダーでは
`models.providers`設定は**不要**です。認証を設定してモデルを選ぶだけです。

### OpenAI

- プロバイダー: `openai`
- 認証: `OPENAI_API_KEY`
- 任意のローテーション: `OPENAI_API_KEYS`, `OPENAI_API_KEY_1`, `OPENAI_API_KEY_2`, に加えて`OPENCLAW_LIVE_OPENAI_KEY`（単一上書き）
- モデル例: `openai/gpt-5.4`, `openai/gpt-5.4-pro`
- CLI: `openclaw onboard --auth-choice openai-api-key`
- デフォルトトランスポートは`auto`です（WebSocket優先、SSEフォールバック）
- モデルごとの上書きは`agents.defaults.models["openai/<model>"].params.transport`（`"sse"`、`"websocket"`、または`"auto"`）で行います
- OpenAI Responses WebSocketウォームアップは、`params.openaiWsWarmup`（`true`/`false`）によりデフォルトで有効です
- OpenAI優先処理は`agents.defaults.models["openai/<model>"].params.serviceTier`で有効にできます
- `/fast`および`params.fastMode`は、直接の`openai/*` Responsesリクエストを`api.openai.com`上の`service_tier=priority`にマップします
- 共有の`/fast`トグルではなく明示的なティアを使いたい場合は`params.serviceTier`を使用してください
- 非表示のOpenClawアトリビューションヘッダー（`originator`, `version`,
  `User-Agent`）は、汎用OpenAI互換プロキシではなく、`api.openai.com`へのネイティブOpenAIトラフィックにのみ適用されます
- ネイティブOpenAIルートは、Responsesの`store`、プロンプトキャッシュヒント、
  OpenAI reasoning互換ペイロード整形も維持します。プロキシルートでは維持されません
- `openai/gpt-5.3-codex-spark`は、ライブのOpenAI APIが拒否するため、OpenClawでは意図的に抑制されています。SparkはCodex専用として扱われます

```json5
{
  agents: { defaults: { model: { primary: "openai/gpt-5.4" } } },
}
```

### Anthropic

- プロバイダー: `anthropic`
- 認証: `ANTHROPIC_API_KEY`
- 任意のローテーション: `ANTHROPIC_API_KEYS`, `ANTHROPIC_API_KEY_1`, `ANTHROPIC_API_KEY_2`, に加えて`OPENCLAW_LIVE_ANTHROPIC_KEY`（単一上書き）
- モデル例: `anthropic/claude-opus-4-6`
- CLI: `openclaw onboard --auth-choice apiKey`
- 直接の公開Anthropicリクエストは、共有の`/fast`トグルと`params.fastMode`もサポートし、`api.anthropic.com`へ送られるAPIキー認証およびOAuth認証トラフィックを含みます。OpenClawはこれをAnthropicの`service_tier`（`auto`対`standard_only`）にマップします
- Anthropicに関する注記: Anthropicスタッフから、OpenClawスタイルのClaude CLI使用は再び許可されていると伝えられたため、Anthropicが新しいポリシーを公開しない限り、OpenClawはClaude CLIの再利用と`claude -p`の使用をこの統合で認可済みとして扱います。
- Anthropic setup-tokenは、引き続きサポートされたOpenClawトークン経路として利用可能ですが、OpenClawは現在、利用可能であればClaude CLIの再利用と`claude -p`を優先します。

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

### OpenAI Code（Codex）

- プロバイダー: `openai-codex`
- 認証: OAuth（ChatGPT）
- モデル例: `openai-codex/gpt-5.4`
- CLI: `openclaw onboard --auth-choice openai-codex`または`openclaw models auth login --provider openai-codex`
- デフォルトトランスポートは`auto`です（WebSocket優先、SSEフォールバック）
- モデルごとの上書きは`agents.defaults.models["openai-codex/<model>"].params.transport`（`"sse"`、`"websocket"`、または`"auto"`）で行います
- `params.serviceTier`は、ネイティブCodex Responsesリクエスト（`chatgpt.com/backend-api`）でも転送されます
- 非表示のOpenClawアトリビューションヘッダー（`originator`, `version`,
  `User-Agent`）は、汎用OpenAI互換プロキシではなく、
  `chatgpt.com/backend-api`へのネイティブCodexトラフィックにのみ付与されます
- 直接の`openai/*`と同じ`/fast`トグルおよび`params.fastMode`設定を共有し、OpenClawはこれを`service_tier=priority`にマップします
- `openai-codex/gpt-5.3-codex-spark`は、Codex OAuthカタログがそれを公開している場合は引き続き利用可能です。利用権に依存します
- `openai-codex/gpt-5.4`は、ネイティブの`contextWindow = 1050000`とデフォルトの実行時`contextTokens = 272000`を維持します。実行時上限は`models.providers.openai-codex.models[].contextTokens`で上書きできます
- ポリシー注記: OpenAI Codex OAuthは、OpenClawのような外部ツール/ワークフロー向けに明示的にサポートされています。

```json5
{
  agents: { defaults: { model: { primary: "openai-codex/gpt-5.4" } } },
}
```

```json5
{
  models: {
    providers: {
      "openai-codex": {
        models: [{ id: "gpt-5.4", contextTokens: 160000 }],
      },
    },
  },
}
```

### その他のサブスクリプション型ホストオプション

- [Qwen Cloud](/ja-JP/providers/qwen): Qwen Cloudプロバイダーサーフェスに加え、Alibaba DashScopeおよびCoding Planエンドポイントのマッピング
- [MiniMax](/ja-JP/providers/minimax): MiniMax Coding Plan OAuthまたはAPIキーアクセス
- [GLM Models](/ja-JP/providers/glm): Z.AI Coding Planまたは汎用APIエンドポイント

### OpenCode

- 認証: `OPENCODE_API_KEY`（または`OPENCODE_ZEN_API_KEY`）
- Zenランタイムプロバイダー: `opencode`
- Goランタイムプロバイダー: `opencode-go`
- モデル例: `opencode/claude-opus-4-6`, `opencode-go/kimi-k2.5`
- CLI: `openclaw onboard --auth-choice opencode-zen`または`openclaw onboard --auth-choice opencode-go`

```json5
{
  agents: { defaults: { model: { primary: "opencode/claude-opus-4-6" } } },
}
```

### Google Gemini（APIキー）

- プロバイダー: `google`
- 認証: `GEMINI_API_KEY`
- 任意のローテーション: `GEMINI_API_KEYS`, `GEMINI_API_KEY_1`, `GEMINI_API_KEY_2`, `GOOGLE_API_KEY`フォールバック、および`OPENCLAW_LIVE_GEMINI_KEY`（単一上書き）
- モデル例: `google/gemini-3.1-pro-preview`, `google/gemini-3-flash-preview`
- 互換性: `google/gemini-3.1-flash-preview`を使うレガシーOpenClaw設定は`google/gemini-3-flash-preview`へ正規化されます
- CLI: `openclaw onboard --auth-choice gemini-api-key`
- 直接Gemini実行では、`agents.defaults.models["google/<model>"].params.cachedContent`
  （またはレガシーの`cached_content`）も受け付け、
  プロバイダーネイティブの`cachedContents/...`ハンドルを転送します。GeminiのキャッシュヒットはOpenClawの`cacheRead`として表示されます

### Google VertexとGemini CLI

- プロバイダー: `google-vertex`, `google-gemini-cli`
- 認証: Vertexはgcloud ADCを使用し、Gemini CLIは独自のOAuthフローを使用します
- 注意: OpenClawでのGemini CLI OAuthは非公式な統合です。サードパーティクライアントの使用後にGoogleアカウントの制限が発生したと報告しているユーザーもいます。利用を続行する場合は、Googleの利用規約を確認し、重要ではないアカウントを使用してください。
- Gemini CLI OAuthは、バンドル版`google`プラグインの一部として提供されます。
  - まずGemini CLIをインストールします:
    - `brew install gemini-cli`
    - または`npm install -g @google/gemini-cli`
  - 有効化: `openclaw plugins enable google`
  - ログイン: `openclaw models auth login --provider google-gemini-cli --set-default`
  - デフォルトモデル: `google-gemini-cli/gemini-3-flash-preview`
  - 注: `openclaw.json`にクライアントIDやシークレットを貼り付けることは**ありません**。CLIログインフローは
    Gatewayホスト上の認証プロファイルにトークンを保存します。
  - ログイン後にリクエストが失敗する場合は、Gatewayホストで`GOOGLE_CLOUD_PROJECT`または`GOOGLE_CLOUD_PROJECT_ID`を設定してください。
  - Gemini CLIのJSON返信は`response`から解析され、使用量は
    `stats`にフォールバックし、`stats.cached`はOpenClawの`cacheRead`に正規化されます。

### Z.AI（GLM）

- プロバイダー: `zai`
- 認証: `ZAI_API_KEY`
- モデル例: `zai/glm-5.1`
- CLI: `openclaw onboard --auth-choice zai-api-key`
  - エイリアス: `z.ai/*`および`z-ai/*`は`zai/*`に正規化されます
  - `zai-api-key`は一致するZ.AIエンドポイントを自動検出します。`zai-coding-global`、`zai-coding-cn`、`zai-global`、および`zai-cn`は特定のサーフェスを強制します

### Vercel AI Gateway

- プロバイダー: `vercel-ai-gateway`
- 認証: `AI_GATEWAY_API_KEY`
- モデル例: `vercel-ai-gateway/anthropic/claude-opus-4.6`
- CLI: `openclaw onboard --auth-choice ai-gateway-api-key`

### Kilo Gateway

- プロバイダー: `kilocode`
- 認証: `KILOCODE_API_KEY`
- モデル例: `kilocode/kilo/auto`
- CLI: `openclaw onboard --auth-choice kilocode-api-key`
- ベースURL: `https://api.kilo.ai/api/gateway/`
- 静的フォールバックカタログには`kilocode/kilo/auto`が同梱されており、
  ライブの`https://api.kilo.ai/api/gateway/models`検出によって実行時
  カタログがさらに拡張される場合があります。
- `kilocode/kilo/auto`の背後にある正確な上流ルーティングはKilo Gatewayが所有しており、
  OpenClawにハードコードされていません。

設定の詳細は[/providers/kilocode](/ja-JP/providers/kilocode)を参照してください。

### その他のバンドル版プロバイダープラグイン

- OpenRouter: `openrouter`（`OPENROUTER_API_KEY`）
- モデル例: `openrouter/auto`
- OpenClawは、リクエストの実際の宛先が`openrouter.ai`である場合にのみ、
  OpenRouterが文書化しているアプリ属性ヘッダーを適用します
- OpenRouter固有のAnthropic `cache_control`マーカーも同様に、
  任意のプロキシURLではなく、検証済みのOpenRouterルートにのみ適用されます
- OpenRouterは引き続きプロキシ形式のOpenAI互換経路上にあるため、ネイティブ
  OpenAI専用のリクエスト整形（`serviceTier`、Responsesの`store`、
  プロンプトキャッシュヒント、OpenAI reasoning互換ペイロード）は転送されません
- GeminiベースのOpenRouter参照では、プロキシGeminiのthought-signatureサニタイズ
  のみが維持されます。ネイティブGeminiのリプレイ検証およびブートストラップ書き換えは無効のままです
- Kilo Gateway: `kilocode`（`KILOCODE_API_KEY`）
- モデル例: `kilocode/kilo/auto`
- GeminiベースのKilo参照では、同じプロキシGemini thought-signature
  サニタイズ経路が維持されます。`kilocode/kilo/auto`やその他のプロキシでreasoning非対応
  のヒントでは、プロキシreasoning注入をスキップします
- MiniMax: `minimax`（APIキー）および`minimax-portal`（OAuth）
- 認証: `minimax`には`MINIMAX_API_KEY`、`minimax-portal`には`MINIMAX_OAUTH_TOKEN`または`MINIMAX_API_KEY`
- モデル例: `minimax/MiniMax-M2.7`または`minimax-portal/MiniMax-M2.7`
- MiniMaxのオンボーディング/APIキー設定では、
  `input: ["text", "image"]`を持つ明示的なM2.7モデル定義が書き込まれます。バンドル版プロバイダーカタログでは、そのプロバイダー設定が具体化されるまでは
  チャット参照をテキスト専用のまま維持します
- Moonshot: `moonshot`（`MOONSHOT_API_KEY`）
- モデル例: `moonshot/kimi-k2.5`
- Kimi Coding: `kimi`（`KIMI_API_KEY`または`KIMICODE_API_KEY`）
- モデル例: `kimi/kimi-code`
- Qianfan: `qianfan`（`QIANFAN_API_KEY`）
- モデル例: `qianfan/deepseek-v3.2`
- Qwen Cloud: `qwen`（`QWEN_API_KEY`、`MODELSTUDIO_API_KEY`、または`DASHSCOPE_API_KEY`）
- モデル例: `qwen/qwen3.5-plus`
- NVIDIA: `nvidia`（`NVIDIA_API_KEY`）
- モデル例: `nvidia/nvidia/llama-3.1-nemotron-70b-instruct`
- StepFun: `stepfun` / `stepfun-plan`（`STEPFUN_API_KEY`）
- モデル例: `stepfun/step-3.5-flash`, `stepfun-plan/step-3.5-flash-2603`
- Together: `together`（`TOGETHER_API_KEY`）
- モデル例: `together/moonshotai/Kimi-K2.5`
- Venice: `venice`（`VENICE_API_KEY`）
- Xiaomi: `xiaomi`（`XIAOMI_API_KEY`）
- モデル例: `xiaomi/mimo-v2-flash`
- Vercel AI Gateway: `vercel-ai-gateway`（`AI_GATEWAY_API_KEY`）
- Hugging Face Inference: `huggingface`（`HUGGINGFACE_HUB_TOKEN`または`HF_TOKEN`）
- Cloudflare AI Gateway: `cloudflare-ai-gateway`（`CLOUDFLARE_AI_GATEWAY_API_KEY`）
- Volcengine: `volcengine`（`VOLCANO_ENGINE_API_KEY`）
- モデル例: `volcengine-plan/ark-code-latest`
- BytePlus: `byteplus`（`BYTEPLUS_API_KEY`）
- モデル例: `byteplus-plan/ark-code-latest`
- xAI: `xai`（`XAI_API_KEY`）
  - ネイティブのバンドル版xAIリクエストはxAI Responses経路を使用します
  - `/fast`または`params.fastMode: true`は、`grok-3`、`grok-3-mini`、
    `grok-4`、および`grok-4-0709`をそれぞれの`*-fast`バリアントに書き換えます
  - `tool_stream`はデフォルトで有効です。無効にするには、
    `agents.defaults.models["xai/<model>"].params.tool_stream`を`false`に
    設定してください
- Mistral: `mistral`（`MISTRAL_API_KEY`）
- モデル例: `mistral/mistral-large-latest`
- CLI: `openclaw onboard --auth-choice mistral-api-key`
- Groq: `groq`（`GROQ_API_KEY`）
- Cerebras: `cerebras`（`CEREBRAS_API_KEY`）
  - Cerebras上のGLMモデルは`zai-glm-4.7`および`zai-glm-4.6`というIDを使用します。
  - OpenAI互換ベースURL: `https://api.cerebras.ai/v1`。
- GitHub Copilot: `github-copilot`（`COPILOT_GITHUB_TOKEN` / `GH_TOKEN` / `GITHUB_TOKEN`）
- Hugging Face Inferenceのモデル例: `huggingface/deepseek-ai/DeepSeek-R1`。CLI: `openclaw onboard --auth-choice huggingface-api-key`。詳細は[Hugging Face (Inference)](/ja-JP/providers/huggingface)を参照してください。

## `models.providers`経由のプロバイダー（カスタム/ベースURL）

**カスタム**プロバイダーまたは
OpenAI/Anthropic互換プロキシを追加するには、`models.providers`（または`models.json`）を使用します。

以下のバンドル版プロバイダープラグインの多くは、すでにデフォルトカタログを公開しています。
デフォルトのベースURL、ヘッダー、またはモデル一覧を上書きしたい場合にのみ、
明示的な`models.providers.<id>`エントリーを使用してください。

### Moonshot AI（Kimi）

Moonshotはバンドル版プロバイダープラグインとして提供されています。通常は組み込みプロバイダーを使用し、
ベースURLまたはモデルメタデータを上書きする必要がある場合にのみ、明示的な`models.providers.moonshot`エントリーを追加してください。

- プロバイダー: `moonshot`
- 認証: `MOONSHOT_API_KEY`
- モデル例: `moonshot/kimi-k2.5`
- CLI: `openclaw onboard --auth-choice moonshot-api-key`または`openclaw onboard --auth-choice moonshot-api-key-cn`

Kimi K2モデルID:

[//]: # "moonshot-kimi-k2-model-refs:start"

- `moonshot/kimi-k2.5`
- `moonshot/kimi-k2-thinking`
- `moonshot/kimi-k2-thinking-turbo`
- `moonshot/kimi-k2-turbo`

[//]: # "moonshot-kimi-k2-model-refs:end"

```json5
{
  agents: {
    defaults: { model: { primary: "moonshot/kimi-k2.5" } },
  },
  models: {
    mode: "merge",
    providers: {
      moonshot: {
        baseUrl: "https://api.moonshot.ai/v1",
        apiKey: "${MOONSHOT_API_KEY}",
        api: "openai-completions",
        models: [{ id: "kimi-k2.5", name: "Kimi K2.5" }],
      },
    },
  },
}
```

### Kimi Coding

Kimi CodingはMoonshot AIのAnthropic互換エンドポイントを使用します。

- プロバイダー: `kimi`
- 認証: `KIMI_API_KEY`
- モデル例: `kimi/kimi-code`

```json5
{
  env: { KIMI_API_KEY: "sk-..." },
  agents: {
    defaults: { model: { primary: "kimi/kimi-code" } },
  },
}
```

レガシーな`kimi/k2p5`も互換モデルIDとして引き続き受け付けられます。

### Volcano Engine（Doubao）

Volcano Engine（火山引擎）は、中国でDoubaoやその他のモデルへのアクセスを提供します。

- プロバイダー: `volcengine`（coding: `volcengine-plan`）
- 認証: `VOLCANO_ENGINE_API_KEY`
- モデル例: `volcengine-plan/ark-code-latest`
- CLI: `openclaw onboard --auth-choice volcengine-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "volcengine-plan/ark-code-latest" } },
  },
}
```

オンボーディングではデフォルトでcodingサーフェスが選ばれますが、一般的な`volcengine/*`
カタログも同時に登録されます。

オンボーディング/モデル設定ピッカーでは、Volcengineの認証選択は
`volcengine/*`行と`volcengine-plan/*`行の両方を優先します。これらのモデルがまだ読み込まれていない場合、
OpenClawは空のプロバイダースコープ付きピッカーを表示する代わりに、
フィルターなしカタログへフォールバックします。

利用可能なモデル:

- `volcengine/doubao-seed-1-8-251228`（Doubao Seed 1.8）
- `volcengine/doubao-seed-code-preview-251028`
- `volcengine/kimi-k2-5-260127`（Kimi K2.5）
- `volcengine/glm-4-7-251222`（GLM 4.7）
- `volcengine/deepseek-v3-2-251201`（DeepSeek V3.2 128K）

コーディングモデル（`volcengine-plan`）:

- `volcengine-plan/ark-code-latest`
- `volcengine-plan/doubao-seed-code`
- `volcengine-plan/kimi-k2.5`
- `volcengine-plan/kimi-k2-thinking`
- `volcengine-plan/glm-4.7`

### BytePlus（国際）

BytePlus ARKは、国際ユーザー向けにVolcano Engineと同じモデルへのアクセスを提供します。

- プロバイダー: `byteplus`（coding: `byteplus-plan`）
- 認証: `BYTEPLUS_API_KEY`
- モデル例: `byteplus-plan/ark-code-latest`
- CLI: `openclaw onboard --auth-choice byteplus-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "byteplus-plan/ark-code-latest" } },
  },
}
```

オンボーディングではデフォルトでcodingサーフェスが選ばれますが、一般的な`byteplus/*`
カタログも同時に登録されます。

オンボーディング/モデル設定ピッカーでは、BytePlusの認証選択は
`byteplus/*`行と`byteplus-plan/*`行の両方を優先します。これらのモデルがまだ読み込まれていない場合、
OpenClawは空のプロバイダースコープ付きピッカーを表示する代わりに、
フィルターなしカタログへフォールバックします。

利用可能なモデル:

- `byteplus/seed-1-8-251228`（Seed 1.8）
- `byteplus/kimi-k2-5-260127`（Kimi K2.5）
- `byteplus/glm-4-7-251222`（GLM 4.7）

コーディングモデル（`byteplus-plan`）:

- `byteplus-plan/ark-code-latest`
- `byteplus-plan/doubao-seed-code`
- `byteplus-plan/kimi-k2.5`
- `byteplus-plan/kimi-k2-thinking`
- `byteplus-plan/glm-4.7`

### Synthetic

Syntheticは、`synthetic`プロバイダーの背後でAnthropic互換モデルを提供します。

- プロバイダー: `synthetic`
- 認証: `SYNTHETIC_API_KEY`
- モデル例: `synthetic/hf:MiniMaxAI/MiniMax-M2.5`
- CLI: `openclaw onboard --auth-choice synthetic-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "synthetic/hf:MiniMaxAI/MiniMax-M2.5" } },
  },
  models: {
    mode: "merge",
    providers: {
      synthetic: {
        baseUrl: "https://api.synthetic.new/anthropic",
        apiKey: "${SYNTHETIC_API_KEY}",
        api: "anthropic-messages",
        models: [{ id: "hf:MiniMaxAI/MiniMax-M2.5", name: "MiniMax M2.5" }],
      },
    },
  },
}
```

### MiniMax

MiniMaxはカスタムエンドポイントを使用するため、`models.providers`経由で設定します。

- MiniMax OAuth（Global）: `--auth-choice minimax-global-oauth`
- MiniMax OAuth（CN）: `--auth-choice minimax-cn-oauth`
- MiniMax APIキー（Global）: `--auth-choice minimax-global-api`
- MiniMax APIキー（CN）: `--auth-choice minimax-cn-api`
- 認証: `minimax`には`MINIMAX_API_KEY`、`minimax-portal`には`MINIMAX_OAUTH_TOKEN`または
  `MINIMAX_API_KEY`

設定の詳細、モデルオプション、設定スニペットについては[/providers/minimax](/ja-JP/providers/minimax)を参照してください。

MiniMaxのAnthropic互換ストリーミング経路では、OpenClawは
明示的に設定しない限りthinkingをデフォルトで無効にし、`/fast on`は
`MiniMax-M2.7`を`MiniMax-M2.7-highspeed`に書き換えます。

プラグイン所有のcapability分割:

- テキスト/チャットのデフォルトは引き続き`minimax/MiniMax-M2.7`
- 画像生成は`minimax/image-01`または`minimax-portal/image-01`
- 画像理解は、両方のMiniMax認証経路でプラグイン所有の`MiniMax-VL-01`
- Web検索はプロバイダーID `minimax`のまま

### Ollama

Ollamaはバンドル版プロバイダープラグインとして提供され、OllamaのネイティブAPIを使用します。

- プロバイダー: `ollama`
- 認証: 不要（ローカルサーバー）
- モデル例: `ollama/llama3.3`
- インストール: [https://ollama.com/download](https://ollama.com/download)

```bash
# Ollamaをインストールし、その後モデルをpullします:
ollama pull llama3.3
```

```json5
{
  agents: {
    defaults: { model: { primary: "ollama/llama3.3" } },
  },
}
```

`OLLAMA_API_KEY`でオプトインすると、Ollamaはローカルの`http://127.0.0.1:11434`で検出され、
バンドル版プロバイダープラグインがOllamaを`openclaw onboard`と
モデルピッカーに直接追加します。オンボーディング、クラウド/ローカルモード、
カスタム設定については[/providers/ollama](/ja-JP/providers/ollama)を参照してください。

### vLLM

vLLMは、ローカル/セルフホストのOpenAI互換
サーバー向けバンドル版プロバイダープラグインとして提供されます。

- プロバイダー: `vllm`
- 認証: 任意（サーバーによる）
- デフォルトベースURL: `http://127.0.0.1:8000/v1`

ローカルで自動検出にオプトインするには（サーバーで認証を強制しない場合は任意の値で動作します）:

```bash
export VLLM_API_KEY="vllm-local"
```

その後、モデルを設定します（`/v1/models`が返すIDのいずれかに置き換えてください）:

```json5
{
  agents: {
    defaults: { model: { primary: "vllm/your-model-id" } },
  },
}
```

詳細は[/providers/vllm](/ja-JP/providers/vllm)を参照してください。

### SGLang

SGLangは、高速なセルフホスト
OpenAI互換サーバー向けバンドル版プロバイダープラグインとして提供されます。

- プロバイダー: `sglang`
- 認証: 任意（サーバーによる）
- デフォルトベースURL: `http://127.0.0.1:30000/v1`

ローカルで自動検出にオプトインするには（サーバーが認証を
強制しない場合は任意の値で動作します）:

```bash
export SGLANG_API_KEY="sglang-local"
```

その後、モデルを設定します（`/v1/models`が返すIDのいずれかに置き換えてください）:

```json5
{
  agents: {
    defaults: { model: { primary: "sglang/your-model-id" } },
  },
}
```

詳細は[/providers/sglang](/ja-JP/providers/sglang)を参照してください。

### ローカルプロキシ（LM Studio、vLLM、LiteLLMなど）

例（OpenAI互換）:

```json5
{
  agents: {
    defaults: {
      model: { primary: "lmstudio/my-local-model" },
      models: { "lmstudio/my-local-model": { alias: "ローカル" } },
    },
  },
  models: {
    providers: {
      lmstudio: {
        baseUrl: "http://localhost:1234/v1",
        apiKey: "LMSTUDIO_KEY",
        api: "openai-completions",
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 200000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

注記:

- カスタムプロバイダーでは、`reasoning`、`input`、`cost`、`contextWindow`、および`maxTokens`は任意です。
  省略した場合、OpenClawのデフォルトは以下になります:
  - `reasoning: false`
  - `input: ["text"]`
  - `cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 }`
  - `contextWindow: 200000`
  - `maxTokens: 8192`
- 推奨: プロキシ/モデルの制限に一致する明示的な値を設定してください。
- 非ネイティブエンドポイント（ホストが`api.openai.com`ではない非空の`baseUrl`）で`api: "openai-completions"`を使用する場合、OpenClawは、未対応の`developer`ロールによるプロバイダー400エラーを避けるため、`compat.supportsDeveloperRole: false`を強制します。
- プロキシ形式のOpenAI互換ルートでは、ネイティブOpenAI専用のリクエスト
  整形もスキップされます。`service_tier`、Responsesの`store`、プロンプトキャッシュヒント、
  OpenAI reasoning互換ペイロード整形、非表示のOpenClawアトリビューション
  ヘッダーは送信されません。
- `baseUrl`が空または省略されている場合、OpenClawはデフォルトのOpenAI動作（`api.openai.com`に解決される）を維持します。
- 安全のため、非ネイティブの`openai-completions`エンドポイントでは、明示的な`compat.supportsDeveloperRole: true`も引き続き上書きされます。

## CLIの例

```bash
openclaw onboard --auth-choice opencode-zen
openclaw models set opencode/claude-opus-4-6
openclaw models list
```

関連項目: 完全な設定例については[/gateway/configuration](/ja-JP/gateway/configuration)を参照してください。

## 関連

- [Models](/ja-JP/concepts/models) — モデル設定とエイリアス
- [Model Failover](/ja-JP/concepts/model-failover) — フォールバックチェーンと再試行動作
- [Configuration Reference](/ja-JP/gateway/configuration-reference#agent-defaults) — モデル設定キー
- [Providers](/ja-JP/providers) — プロバイダーごとの設定ガイド
