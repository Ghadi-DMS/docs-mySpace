---
read_when:
    - アクセスや自動化を拡大する機能の追加
summary: シェルアクセスを持つAI Gatewayを実行する際のセキュリティ上の考慮事項と脅威モデル
title: セキュリティ
x-i18n:
    generated_at: "2026-04-11T02:44:59Z"
    model: gpt-5.4
    provider: openai
    source_hash: 770407f64b2ce27221ebd9756b2f8490a249c416064186e64edb663526f9d6b5
    source_path: gateway/security/index.md
    workflow: 15
---

# セキュリティ

<Warning>
**パーソナルアシスタントの信頼モデル:** このガイダンスは、Gatewayごとに1つの信頼された運用者境界があることを前提としています（単一ユーザー／パーソナルアシスタントモデル）。
OpenClawは、1つのエージェント／Gatewayを複数の敵対的ユーザーが共有するような、敵対的マルチテナントのセキュリティ境界では**ありません**。
混在した信頼や敵対的ユーザーでの運用が必要な場合は、信頼境界を分割してください（Gatewayと認証情報を分離し、可能であればOSユーザーやホストも分離してください）。
</Warning>

**このページの内容:** [信頼モデル](#scope-first-personal-assistant-security-model) | [クイック監査](#quick-check-openclaw-security-audit) | [強化済みベースライン](#hardened-baseline-in-60-seconds) | [DMアクセスモデル](#dm-access-model-pairing-allowlist-open-disabled) | [設定の強化](#configuration-hardening-examples) | [インシデント対応](#incident-response)

## まず範囲を明確にする: パーソナルアシスタントのセキュリティモデル

OpenClawのセキュリティガイダンスは、**パーソナルアシスタント**としてのデプロイを前提としています。つまり、信頼された運用者境界は1つで、そこに複数のエージェントが存在する可能性があるということです。

- サポートされるセキュリティ体制: Gatewayごとに1つのユーザー／信頼境界（境界ごとに1つのOSユーザー／ホスト／VPSが望ましい）。
- サポートされないセキュリティ境界: 相互に信頼していない、または敵対的なユーザー同士が共有する1つのGateway／エージェント。
- 敵対的ユーザー間の分離が必要な場合は、信頼境界ごとに分割してください（Gatewayと認証情報を分離し、理想的にはOSユーザーやホストも分離します）。
- 複数の信頼していないユーザーが、ツール有効化済みの1つのエージェントにメッセージできる場合、そのユーザーたちはそのエージェントに委任された同じツール権限を共有しているものとして扱ってください。

このページでは、**そのモデルの範囲内での**強化方法を説明します。1つの共有Gateway上で敵対的マルチテナント分離を実現できるとは主張していません。

## クイックチェック: `openclaw security audit`

参照: [Formal Verification (Security Models)](/ja-JP/security/formal-verification)

これは定期的に実行してください。特に、設定を変更した後やネットワークサーフェスを公開した後は重要です。

```bash
openclaw security audit
openclaw security audit --deep
openclaw security audit --fix
openclaw security audit --json
```

`security audit --fix` は、意図的に対象を絞っています。一般的なオープングループポリシーを許可リストへ切り替え、`logging.redactSensitive: "tools"` を復元し、state/config/include-file の権限を強化し、Windows上で実行される場合はPOSIXの `chmod` ではなくWindows ACLのリセットを使用します。

この監査は、よくある危険な設定（Gateway認証の露出、ブラウザ制御の露出、昇格済み許可リスト、ファイルシステム権限、緩いexec承認、オープンチャネルからのツール露出）を検出します。

OpenClawは製品であると同時に実験でもあります。つまり、最先端モデルの振る舞いを、実際のメッセージングサーフェスや実際のツールへ接続しています。**「完全に安全な」セットアップは存在しません。** 重要なのは、次の点について意識的であることです。

- 誰があなたのボットに話しかけられるのか
- ボットがどこで動作してよいのか
- ボットが何に触れられるのか

まずは機能する最小限のアクセスから始め、確信が持てるようになってから少しずつ広げてください。

### デプロイとホストの信頼

OpenClawは、ホストと設定の境界が信頼されていることを前提としています。

- 誰かがGatewayホストの状態や設定（`openclaw.json` を含む `~/.openclaw`）を変更できるなら、その人は信頼された運用者として扱ってください。
- 相互に信頼していない、または敵対的な複数の運用者のために1つのGatewayを動かすのは、**推奨される構成ではありません**。
- 信頼が混在するチームでは、別々のGatewayで信頼境界を分離してください（少なくともOSユーザーやホストは分けてください）。
- 推奨されるデフォルトは、マシン／ホスト（またはVPS）ごとに1ユーザー、そのユーザー用に1つのGateway、そしてそのGateway内に1つ以上のエージェント、という構成です。
- 1つのGatewayインスタンス内では、認証済みの運用者アクセスは、ユーザー単位のテナント役割ではなく、信頼されたコントロールプレーン役割です。
- セッション識別子（`sessionKey`、セッションID、ラベル）はルーティング選択子であり、認可トークンではありません。
- 複数人が1つのツール有効化済みエージェントにメッセージできる場合、その全員が同じ権限セットを操作できます。ユーザーごとのセッション／メモリー分離はプライバシーには役立ちますが、共有エージェントをユーザー単位のホスト認可へ変えるものではありません。

### 共有Slackワークスペース: 実際のリスク

「Slackの誰でもボットにメッセージできる」場合、中核となるリスクは委任されたツール権限です。

- 許可された送信者は誰でも、エージェントのポリシー範囲内でツール呼び出し（`exec`、ブラウザ、ネットワーク／ファイルツール）を誘発できます。
- ある送信者からのプロンプト／コンテンツインジェクションにより、共有状態、デバイス、または出力へ影響するアクションが引き起こされる可能性があります。
- 1つの共有エージェントが機密な認証情報やファイルを持っている場合、許可された送信者は誰でも、ツール利用を通じて情報流出を引き起こせる可能性があります。

チーム向けワークフローには、ツールを最小限にした別々のエージェント／Gatewayを使用し、個人データを扱うエージェントは非公開に保ってください。

### 会社で共有するエージェント: 許容されるパターン

そのエージェントを使う全員が同じ信頼境界に属し（たとえば同じ会社の1チーム）、かつそのエージェントが厳密に業務範囲に限定されているなら、これは許容されます。

- 専用のマシン／VM／コンテナ上で実行する。
- そのランタイム専用のOSユーザー、専用のブラウザ／プロファイル／アカウントを使う。
- そのランタイムで、個人のApple／Googleアカウントや個人のパスワードマネージャー／ブラウザプロファイルにサインインしない。

同じランタイム上で個人用と会社用のアイデンティティを混在させると、その分離は崩れ、個人データの露出リスクが高まります。

## Gatewayとnodeの信頼概念

Gatewayとnodeは、役割の異なる1つの運用者信頼ドメインとして扱ってください。

- **Gateway** はコントロールプレーンであり、ポリシーサーフェスです（`gateway.auth`、ツールポリシー、ルーティング）。
- **Node** はそのGatewayとペアリングされたリモート実行サーフェスです（コマンド、デバイス操作、ホストローカル機能）。
- Gatewayに認証された呼び出し元は、Gatewayスコープで信頼されます。ペアリング後のnodeアクションは、そのnode上での信頼された運用者アクションです。
- `sessionKey` はルーティング／コンテキスト選択であり、ユーザー単位の認証ではありません。
- Exec承認（許可リスト + ask）は、運用者の意図に対するガードレールであり、敵対的マルチテナント分離ではありません。
- 信頼された単一運用者セットアップにおけるOpenClawの製品デフォルトでは、`gateway`／`node` 上のホストexecは承認プロンプトなしで許可されます（`security="full"`、それを厳しくしない限り `ask="off"`）。このデフォルトは意図的なUXであり、それ自体は脆弱性ではありません。
- Exec承認は、正確なリクエストコンテキストと、ベストエフォートの直接的なローカルファイルオペランドに結び付きます。あらゆるランタイム／インタープリターのローダーパスを意味的にモデル化するものではありません。強い境界が必要なら、サンドボックス化とホスト分離を使用してください。

敵対的ユーザー分離が必要な場合は、OSユーザー／ホスト単位で信頼境界を分割し、別々のGatewayを実行してください。

## 信頼境界マトリクス

リスクをトリアージするときの簡易モデルとして、これを使ってください。

| 境界または制御 | それが意味すること | よくある誤解 |
| -------------- | ------------------ | ------------ |
| `gateway.auth` (token/password/trusted-proxy/device auth) | Gateway APIへの呼び出し元を認証する | 「安全にするには、すべてのフレームにメッセージ単位の署名が必要」 |
| `sessionKey` | コンテキスト／セッション選択のためのルーティングキー | 「セッションキーはユーザー認証境界である」 |
| プロンプト／コンテンツのガードレール | モデル悪用リスクを低減する | 「プロンプトインジェクションだけで認証回避が証明される」 |
| `canvas.eval` / browser evaluate | 有効化されている場合の意図的な運用者機能 | 「どんなJS evalプリミティブでも、この信頼モデルでは自動的に脆弱性になる」 |
| ローカルTUIの `!` shell | 明示的に運用者が起動するローカル実行 | 「ローカルのシェル利便機能コマンドはリモートインジェクションである」 |
| Nodeペアリングとnodeコマンド | ペアリング済みデバイス上での運用者レベルのリモート実行 | 「リモートデバイス制御は、デフォルトで信頼されていないユーザーアクセスとして扱うべき」 |

## 設計上、脆弱性ではないもの

以下のパターンはよく報告されますが、実際の境界回避が示されない限り、通常は対応不要としてクローズされます。

- ポリシー／認証／サンドボックス回避を伴わない、プロンプトインジェクションのみの連鎖。
- 1つの共有ホスト／設定上での敵対的マルチテナント運用を前提にした主張。
- 共有Gateway構成で、通常の運用者の読み取り経路アクセス（たとえば `sessions.list`／`sessions.preview`／`chat.history`）をIDORとして分類する主張。
- localhost限定デプロイに対する指摘（たとえば、loopback専用GatewayでのHSTS）。
- このリポジトリに存在しない受信パスに対する、Discord受信webhook署名の指摘。
- `system.run` に対して、nodeペアリングメタデータを隠れた第2のコマンド単位承認レイヤーとして扱う報告。ただし実際の実行境界は依然としてGatewayのグローバルnodeコマンドポリシーとnode自身のexec承認です。
- `sessionKey` を認証トークンとして扱う「ユーザー単位認可の欠如」の指摘。

## 研究者向けの事前チェックリスト

GHSAを開く前に、次のすべてを確認してください。

1. 再現手順が最新の `main` または最新リリースでも成立する。
2. 報告に、正確なコードパス（`file`、関数、行範囲）と、テストしたバージョン／コミットが含まれている。
3. 影響が、文書化された信頼境界をまたいでいる（単なるプロンプトインジェクションではない）。
4. 主張が [Out of Scope](https://github.com/openclaw/openclaw/blob/main/SECURITY.md#out-of-scope) に記載されていない。
5. 既存アドバイザリに重複がないか確認済みである（該当する場合は正規のGHSAを再利用する）。
6. デプロイ前提（loopback/local か公開済みか、信頼された運用者か信頼していない運用者か）が明示されている。

## 60秒でできる強化済みベースライン

まずこのベースラインを使い、その後、信頼するエージェントごとに必要なツールだけを再有効化してください。

```json5
{
  gateway: {
    mode: "local",
    bind: "loopback",
    auth: { mode: "token", token: "replace-with-long-random-token" },
  },
  session: {
    dmScope: "per-channel-peer",
  },
  tools: {
    profile: "messaging",
    deny: ["group:automation", "group:runtime", "group:fs", "sessions_spawn", "sessions_send"],
    fs: { workspaceOnly: true },
    exec: { security: "deny", ask: "always" },
    elevated: { enabled: false },
  },
  channels: {
    whatsapp: { dmPolicy: "pairing", groups: { "*": { requireMention: true } } },
  },
}
```

これにより、Gatewayはローカル専用のままとなり、DMは分離され、コントロールプレーン／ランタイムツールはデフォルトで無効化されます。

## 共有受信箱の簡易ルール

複数人があなたのボットへDMできる場合:

- `session.dmScope: "per-channel-peer"` を設定してください（マルチアカウントチャネルでは `"per-account-channel-peer"`）。
- `dmPolicy: "pairing"` または厳格な許可リストを維持してください。
- 共有DMと広範なツールアクセスを絶対に組み合わせないでください。
- これにより協調型／共有受信箱は強化されますが、ユーザーがホスト／設定への書き込みアクセスを共有している場合の、敵対的な共同テナント分離を目的とした設計ではありません。

## コンテキスト可視性モデル

OpenClawは、次の2つの概念を分離しています。

- **トリガー認可**: 誰がエージェントを起動できるか（`dmPolicy`、`groupPolicy`、許可リスト、メンションゲート）。
- **コンテキスト可視性**: どの補助コンテキストがモデル入力へ注入されるか（返信本文、引用テキスト、スレッド履歴、転送メタデータ）。

許可リストは、トリガーとコマンド認可を制御します。`contextVisibility` 設定は、補助コンテキスト（引用返信、スレッドルート、取得済み履歴）をどのようにフィルタリングするかを制御します。

- `contextVisibility: "all"`（デフォルト）は、補助コンテキストを受信したまま保持します。
- `contextVisibility: "allowlist"` は、補助コンテキストを現在有効な許可リストチェックで許可された送信者に限定してフィルタリングします。
- `contextVisibility: "allowlist_quote"` は `allowlist` と同様に動作しますが、明示的な引用返信を1つだけ保持します。

`contextVisibility` は、チャネル単位またはルーム／会話単位で設定してください。設定の詳細は [Group Chats](/ja-JP/channels/groups#context-visibility-and-allowlists) を参照してください。

アドバイザリのトリアージ指針:

- 「モデルが、許可リストにない送信者からの引用文や履歴テキストを見られる」ことだけを示す主張は、`contextVisibility` で対処できる強化上の指摘であり、それ自体では認証やサンドボックス境界の回避ではありません。
- セキュリティ上の影響を持つには、報告は依然として、信頼境界（認証、ポリシー、サンドボックス、承認、または他の文書化された境界）の回避を実証する必要があります。

## 監査でチェックされるもの（概要）

- **受信アクセス**（DMポリシー、グループポリシー、許可リスト）: 見知らぬ相手がボットを起動できるか？
- **ツールの影響範囲**（昇格ツール + オープンなルーム）: プロンプトインジェクションがシェル／ファイル／ネットワーク操作に発展しうるか？
- **Exec承認のずれ**（`security=full`、`autoAllowSkills`、`strictInlineEval` なしのインタープリター許可リスト）: ホストexecのガードレールは、まだ意図したとおりに機能しているか？
  - `security="full"` は広範な姿勢に関する警告であり、バグの証明ではありません。これは、信頼されたパーソナルアシスタント構成向けに選ばれたデフォルトです。脅威モデル上、承認や許可リストのガードレールが必要な場合にのみ、これを厳しくしてください。
- **ネットワーク露出**（Gatewayのbind/auth、Tailscale Serve/Funnel、弱い／短い認証トークン）。
- **ブラウザ制御の露出**（リモートnode、リレーポート、リモートCDPエンドポイント）。
- **ローカルディスク衛生**（権限、symlink、設定include、`「同期フォルダー」` のパス）。
- **プラグイン**（明示的な許可リストなしで拡張機能が存在している）。
- **ポリシーのずれ／誤設定**（sandbox docker設定はあるのにsandboxモードがオフ、`gateway.nodes.denyCommands` パターンが無効になっているのは、マッチングが正確なコマンド名のみを対象とし、シェルテキストを検査しないためであることへの注意。たとえば `system.run`。危険な `gateway.nodes.allowCommands` エントリー。グローバルな `tools.profile="minimal"` がエージェント単位プロファイルで上書きされている。緩いツールポリシー下で拡張プラグインのツールへ到達可能）。
- **ランタイム期待値のずれ**（たとえば、`tools.exec.host` のデフォルトが `auto` になった後でも、暗黙のexecが `sandbox` を意味すると想定している場合や、sandboxモードがオフなのに明示的に `tools.exec.host="sandbox"` を設定している場合）。
- **モデル衛生**（設定されたモデルがレガシーに見える場合に警告。ハードブロックではありません）。

`--deep` を実行すると、OpenClawはベストエフォートでライブGatewayプローブも試みます。

## 認証情報ストレージの対応表

アクセス監査やバックアップ対象の判断時には、これを使ってください。

- **WhatsApp**: `~/.openclaw/credentials/whatsapp/<accountId>/creds.json`
- **Telegram bot token**: config/env または `channels.telegram.tokenFile`（通常ファイルのみ。symlinkは拒否）
- **Discord bot token**: config/env または SecretRef（env/file/execプロバイダー）
- **Slackトークン**: config/env (`channels.slack.*`)
- **ペアリング許可リスト**:
  - `~/.openclaw/credentials/<channel>-allowFrom.json`（デフォルトアカウント）
  - `~/.openclaw/credentials/<channel>-<accountId>-allowFrom.json`（デフォルト以外のアカウント）
- **モデル認証プロファイル**: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
- **ファイルベースのシークレットペイロード（任意）**: `~/.openclaw/secrets.json`
- **レガシーOAuthインポート**: `~/.openclaw/credentials/oauth.json`

## セキュリティ監査チェックリスト

監査で検出結果が表示された場合は、次の優先順位で扱ってください。

1. **`「open」` な設定 + ツール有効化**: まずDM／グループをロックダウンします（ペアリング／許可リスト）。その後でツールポリシーやsandbox化を強化してください。
2. **公開ネットワーク露出**（LAN bind、Funnel、auth欠如）: 直ちに修正してください。
3. **ブラウザ制御のリモート露出**: 運用者アクセスと同等に扱ってください（tailnet限定、意図的なnodeペアリング、公開露出を避ける）。
4. **権限**: state／config／credentials／auth がグループまたは全員に読み取り可能になっていないことを確認してください。
5. **プラグイン／拡張機能**: 明示的に信頼するものだけを読み込んでください。
6. **モデル選択**: ツールを持つボットには、現代的で命令耐性の高いモデルを優先してください。

## セキュリティ監査用語集

実運用環境で特によく見かける、シグナルの強い `checkId` 値は次のとおりです（網羅的ではありません）:

| `checkId`                                                     | 重大度        | 重要な理由                                                                             | 主な修正キー／パス                                                                                   | 自動修正 |
| ------------------------------------------------------------- | ------------- | -------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- | -------- |
| `fs.state_dir.perms_world_writable`                           | critical      | 他のユーザー／プロセスがOpenClawの状態全体を変更できる                                | `~/.openclaw` のファイルシステム権限                                                                 | あり     |
| `fs.state_dir.perms_group_writable`                           | warn          | 同じグループのユーザーがOpenClawの状態全体を変更できる                                | `~/.openclaw` のファイルシステム権限                                                                 | あり     |
| `fs.state_dir.perms_readable`                                 | warn          | stateディレクトリを他者が読み取れる                                                   | `~/.openclaw` のファイルシステム権限                                                                 | あり     |
| `fs.state_dir.symlink`                                        | warn          | stateディレクトリの参照先が別の信頼境界になる                                          | stateディレクトリのファイルシステムレイアウト                                                       | なし     |
| `fs.config.perms_writable`                                    | critical      | 他者がauth／ツールポリシー／設定を変更できる                                           | `~/.openclaw/openclaw.json` のファイルシステム権限                                                  | あり     |
| `fs.config.symlink`                                           | warn          | configの参照先が別の信頼境界になる                                                     | configファイルのファイルシステムレイアウト                                                           | なし     |
| `fs.config.perms_group_readable`                              | warn          | 同じグループのユーザーがconfig内のトークン／設定を読める                              | configファイルのファイルシステム権限                                                                 | あり     |
| `fs.config.perms_world_readable`                              | critical      | configからトークン／設定が露出する可能性がある                                         | configファイルのファイルシステム権限                                                                 | あり     |
| `fs.config_include.perms_writable`                            | critical      | config includeファイルを他者が変更できる                                               | `openclaw.json` から参照されるincludeファイルの権限                                                 | あり     |
| `fs.config_include.perms_group_readable`                      | warn          | 同じグループのユーザーがincludeされたシークレット／設定を読める                        | `openclaw.json` から参照されるincludeファイルの権限                                                 | あり     |
| `fs.config_include.perms_world_readable`                      | critical      | includeされたシークレット／設定が誰でも読める                                          | `openclaw.json` から参照されるincludeファイルの権限                                                 | あり     |
| `fs.auth_profiles.perms_writable`                             | critical      | 他者が保存済みモデル認証情報を注入または置換できる                                     | `agents/<agentId>/agent/auth-profiles.json` の権限                                                  | あり     |
| `fs.auth_profiles.perms_readable`                             | warn          | 他者がAPIキーやOAuthトークンを読める                                                   | `agents/<agentId>/agent/auth-profiles.json` の権限                                                  | あり     |
| `fs.credentials_dir.perms_writable`                           | critical      | 他者がチャネルのペアリング／認証情報状態を変更できる                                   | `~/.openclaw/credentials` のファイルシステム権限                                                    | あり     |
| `fs.credentials_dir.perms_readable`                           | warn          | 他者がチャネルの認証情報状態を読める                                                   | `~/.openclaw/credentials` のファイルシステム権限                                                    | あり     |
| `fs.sessions_store.perms_readable`                            | warn          | 他者がセッショントランスクリプト／メタデータを読める                                   | セッションストアの権限                                                                               | あり     |
| `fs.log_file.perms_readable`                                  | warn          | 他者が、マスク済みではあるが依然として機微なログを読める                               | Gatewayログファイルの権限                                                                            | あり     |
| `fs.synced_dir`                                               | warn          | iCloud／Dropbox／Drive内のstate／configは、トークン／トランスクリプト露出範囲を広げる | config／stateを同期フォルダーから移動する                                                           | なし     |
| `gateway.bind_no_auth`                                        | critical      | 共有シークレットなしでリモートbindしている                                             | `gateway.bind`、`gateway.auth.*`                                                                     | なし     |
| `gateway.loopback_no_auth`                                    | critical      | リバースプロキシ経由のloopbackが未認証になる可能性がある                               | `gateway.auth.*`、プロキシ設定                                                                       | なし     |
| `gateway.trusted_proxies_missing`                             | warn          | リバースプロキシヘッダーが存在するのに信頼されていない                                 | `gateway.trustedProxies`                                                                             | なし     |
| `gateway.http.no_auth`                                        | warn/critical | `auth.mode="none"` の状態でGateway HTTP APIに到達できる                                | `gateway.auth.mode`、`gateway.http.endpoints.*`                                                      | なし     |
| `gateway.http.session_key_override_enabled`                   | info          | HTTP API呼び出し元が `sessionKey` を上書きできる                                       | `gateway.http.allowSessionKeyOverride`                                                               | なし     |
| `gateway.tools_invoke_http.dangerous_allow`                   | warn/critical | HTTP API経由で危険なツールを再有効化する                                               | `gateway.tools.allow`                                                                                | なし     |
| `gateway.nodes.allow_commands_dangerous`                      | warn/critical | 影響の大きいnodeコマンド（camera／screen／contacts／calendar／SMS）を有効にする        | `gateway.nodes.allowCommands`                                                                        | なし     |
| `gateway.nodes.deny_commands_ineffective`                     | warn          | denyエントリーのパターン風指定は、シェルテキストやグループにはマッチしない             | `gateway.nodes.denyCommands`                                                                         | なし     |
| `gateway.tailscale_funnel`                                    | critical      | 公開インターネットへの露出                                                              | `gateway.tailscale.mode`                                                                             | なし     |
| `gateway.tailscale_serve`                                     | info          | Serve経由のtailnet露出が有効になっている                                               | `gateway.tailscale.mode`                                                                             | なし     |
| `gateway.control_ui.allowed_origins_required`                 | critical      | loopback以外のControl UIで、明示的なブラウザオリジン許可リストがない                   | `gateway.controlUi.allowedOrigins`                                                                   | なし     |
| `gateway.control_ui.allowed_origins_wildcard`                 | warn/critical | `allowedOrigins=["*"]` によりブラウザオリジン許可リストが無効化される                  | `gateway.controlUi.allowedOrigins`                                                                   | なし     |
| `gateway.control_ui.host_header_origin_fallback`              | warn/critical | Hostヘッダー由来のオリジンフォールバックを有効にし、DNS rebinding対策が弱まる          | `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback`                                         | なし     |
| `gateway.control_ui.insecure_auth`                            | warn          | 互換性のための安全でないauthトグルが有効になっている                                   | `gateway.controlUi.allowInsecureAuth`                                                                | なし     |
| `gateway.control_ui.device_auth_disabled`                     | critical      | デバイスアイデンティティチェックを無効化する                                           | `gateway.controlUi.dangerouslyDisableDeviceAuth`                                                     | なし     |
| `gateway.real_ip_fallback_enabled`                            | warn/critical | `X-Real-IP` フォールバックを信頼すると、プロキシ誤設定経由で送信元IP偽装が可能になる    | `gateway.allowRealIpFallback`、`gateway.trustedProxies`                                              | なし     |
| `gateway.token_too_short`                                     | warn          | 短い共有トークンは総当たりしやすい                                                     | `gateway.auth.token`                                                                                 | なし     |
| `gateway.auth_no_rate_limit`                                  | warn          | 認証付きの公開面にレート制限がないと総当たりリスクが高まる                             | `gateway.auth.rateLimit`                                                                             | なし     |
| `gateway.trusted_proxy_auth`                                  | critical      | プロキシのアイデンティティがauth境界になる                                             | `gateway.auth.mode="trusted-proxy"`                                                                  | なし     |
| `gateway.trusted_proxy_no_proxies`                            | critical      | trusted-proxy authなのに信頼するプロキシIPがないのは安全ではない                       | `gateway.trustedProxies`                                                                             | なし     |
| `gateway.trusted_proxy_no_user_header`                        | critical      | trusted-proxy authでユーザーアイデンティティを安全に解決できない                       | `gateway.auth.trustedProxy.userHeader`                                                               | なし     |
| `gateway.trusted_proxy_no_allowlist`                          | warn          | trusted-proxy authで、認証済みの任意の上流ユーザーを受け入れてしまう                   | `gateway.auth.trustedProxy.allowUsers`                                                               | なし     |
| `checkId`                                                     | 重大度        | 重要な理由                                                                             | 主な修正キー／パス                                                                                   | 自動修正 |
| ------------------------------------------------------------- | ------------- | -------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- | -------- |
| `gateway.probe_auth_secretref_unavailable`                    | warn          | このコマンド経路ではディーププローブがauth SecretRefを解決できなかった                | ディーププローブのauthソース／SecretRefの可用性                                                     | なし     |
| `gateway.probe_failed`                                        | warn/critical | ライブGatewayプローブが失敗した                                                       | Gatewayの到達性／auth                                                                                | なし     |
| `discovery.mdns_full_mode`                                    | warn/critical | mDNSのfullモードはローカルネットワーク上に `cliPath`／`sshPort` メタデータを広告する  | `discovery.mdns.mode`、`gateway.bind`                                                                | なし     |
| `config.insecure_or_dangerous_flags`                          | warn          | 安全でない／危険なデバッグフラグが有効になっている                                    | 複数のキー（検出詳細を参照）                                                                         | なし     |
| `config.secrets.gateway_password_in_config`                   | warn          | Gatewayパスワードがconfig内に直接保存されている                                       | `gateway.auth.password`                                                                              | なし     |
| `config.secrets.hooks_token_in_config`                        | warn          | Hook bearer tokenがconfig内に直接保存されている                                       | `hooks.token`                                                                                        | なし     |
| `hooks.token_reuse_gateway_token`                             | critical      | Hook受信トークンがGateway authの解除にも使われる                                      | `hooks.token`、`gateway.auth.token`                                                                  | なし     |
| `hooks.token_too_short`                                       | warn          | Hook受信で総当たりが容易になる                                                        | `hooks.token`                                                                                        | なし     |
| `hooks.default_session_key_unset`                             | warn          | Hookエージェントの実行が、リクエストごとに生成されるセッションへファンアウトする       | `hooks.defaultSessionKey`                                                                            | なし     |
| `hooks.allowed_agent_ids_unrestricted`                        | warn/critical | 認証済みHook呼び出し元が、設定済みの任意のエージェントへルーティングできる             | `hooks.allowedAgentIds`                                                                              | なし     |
| `hooks.request_session_key_enabled`                           | warn/critical | 外部呼び出し元が `sessionKey` を選べる                                                 | `hooks.allowRequestSessionKey`                                                                       | なし     |
| `hooks.request_session_key_prefixes_missing`                  | warn/critical | 外部セッションキーの形状に制限がない                                                   | `hooks.allowedSessionKeyPrefixes`                                                                    | なし     |
| `hooks.path_root`                                             | critical      | Hookパスが `/` のため、受信が衝突または誤ルーティングしやすい                          | `hooks.path`                                                                                         | なし     |
| `hooks.installs_unpinned_npm_specs`                           | warn          | Hookインストール記録が不変のnpm specに固定されていない                                | Hookインストールメタデータ                                                                           | なし     |
| `hooks.installs_missing_integrity`                            | warn          | Hookインストール記録に整合性メタデータがない                                           | Hookインストールメタデータ                                                                           | なし     |
| `hooks.installs_version_drift`                                | warn          | Hookインストール記録がインストール済みパッケージとずれている                           | Hookインストールメタデータ                                                                           | なし     |
| `logging.redact_off`                                          | warn          | 機密値がログ／statusへ漏れる                                                           | `logging.redactSensitive`                                                                            | あり     |
| `browser.control_invalid_config`                              | warn          | ランタイム前の時点でブラウザ制御設定が不正                                             | `browser.*`                                                                                          | なし     |
| `browser.control_no_auth`                                     | critical      | トークン／パスワードauthなしでブラウザ制御が公開されている                             | `gateway.auth.*`                                                                                     | なし     |
| `browser.remote_cdp_http`                                     | warn          | 平文HTTP経由のリモートCDPには通信の暗号化がない                                        | ブラウザプロファイルの `cdpUrl`                                                                      | なし     |
| `browser.remote_cdp_private_host`                             | warn          | リモートCDPの対象がプライベート／内部ホストである                                      | ブラウザプロファイルの `cdpUrl`、`browser.ssrfPolicy.*`                                              | なし     |
| `sandbox.docker_config_mode_off`                              | warn          | Sandbox Docker設定はあるが有効化されていない                                           | `agents.*.sandbox.mode`                                                                              | なし     |
| `sandbox.bind_mount_non_absolute`                             | warn          | 相対bind mountは予測しにくい形で解決される可能性がある                                 | `agents.*.sandbox.docker.binds[]`                                                                    | なし     |
| `sandbox.dangerous_bind_mount`                                | critical      | Sandboxのbind mount先が、ブロック対象のシステム／認証情報／Dockerソケットのパスである  | `agents.*.sandbox.docker.binds[]`                                                                    | なし     |
| `sandbox.dangerous_network_mode`                              | critical      | Sandbox Dockerネットワークが `host` または `container:*` の名前空間共有モードを使う    | `agents.*.sandbox.docker.network`                                                                    | なし     |
| `sandbox.dangerous_seccomp_profile`                           | critical      | Sandbox seccompプロファイルがコンテナ分離を弱める                                      | `agents.*.sandbox.docker.securityOpt`                                                                | なし     |
| `sandbox.dangerous_apparmor_profile`                          | critical      | Sandbox AppArmorプロファイルがコンテナ分離を弱める                                     | `agents.*.sandbox.docker.securityOpt`                                                                | なし     |
| `sandbox.browser_cdp_bridge_unrestricted`                     | warn          | Sandboxブラウザブリッジが送信元範囲制限なしで公開されている                            | `sandbox.browser.cdpSourceRange`                                                                     | なし     |
| `sandbox.browser_container.non_loopback_publish`              | critical      | 既存のブラウザコンテナが、loopback以外のインターフェースでCDPを公開している            | ブラウザsandboxコンテナの公開設定                                                                    | なし     |
| `sandbox.browser_container.hash_label_missing`                | warn          | 既存のブラウザコンテナは、現在のconfig-hashラベル導入前のものである                    | `openclaw sandbox recreate --browser --all`                                                          | なし     |
| `sandbox.browser_container.hash_epoch_stale`                  | warn          | 既存のブラウザコンテナは、現在のブラウザ設定エポックより前のものである                 | `openclaw sandbox recreate --browser --all`                                                          | なし     |
| `tools.exec.host_sandbox_no_sandbox_defaults`                 | warn          | sandboxがオフのとき、`exec host=sandbox` はフェイルクローズする                         | `tools.exec.host`、`agents.defaults.sandbox.mode`                                                    | なし     |
| `tools.exec.host_sandbox_no_sandbox_agents`                   | warn          | エージェント単位の `exec host=sandbox` は、sandboxがオフのときフェイルクローズする      | `agents.list[].tools.exec.host`、`agents.list[].sandbox.mode`                                        | なし     |
| `tools.exec.security_full_configured`                         | warn/critical | ホストexecが `security="full"` で実行されている                                        | `tools.exec.security`、`agents.list[].tools.exec.security`                                           | なし     |
| `tools.exec.auto_allow_skills_enabled`                        | warn          | Exec承認がskill binを暗黙に信頼する                                                    | `~/.openclaw/exec-approvals.json`                                                                    | なし     |
| `tools.exec.allowlist_interpreter_without_strict_inline_eval` | warn          | インタープリター許可リストが、再承認を強制せずにインラインevalを許可する                | `tools.exec.strictInlineEval`、`agents.list[].tools.exec.strictInlineEval`、exec approvals allowlist | なし     |
| `tools.exec.safe_bins_interpreter_unprofiled`                 | warn          | `safeBins` にあるインタープリター／ランタイムbinが明示プロファイルなしでexecリスクを広げる | `tools.exec.safeBins`、`tools.exec.safeBinProfiles`、`agents.list[].tools.exec.*`                 | なし     |
| `tools.exec.safe_bins_broad_behavior`                         | warn          | `safeBins` にある広範動作ツールが、低リスクstdinフィルタ信頼モデルを弱める              | `tools.exec.safeBins`、`agents.list[].tools.exec.safeBins`                                           | なし     |
| `tools.exec.safe_bin_trusted_dirs_risky`                      | warn          | `safeBinTrustedDirs` に可変または危険なディレクトリが含まれている                       | `tools.exec.safeBinTrustedDirs`、`agents.list[].tools.exec.safeBinTrustedDirs`                       | なし     |
| `skills.workspace.symlink_escape`                             | warn          | ワークスペースの `skills/**/SKILL.md` がワークスペースルート外を解決する（symlink連鎖のずれ） | ワークスペース `skills/**` のファイルシステム状態                                                | なし     |
| `plugins.extensions_no_allowlist`                             | warn          | 明示的なプラグイン許可リストなしで拡張機能がインストールされている                      | `plugins.allowlist`                                                                                  | なし     |
| `plugins.installs_unpinned_npm_specs`                         | warn          | プラグインインストール記録が不変のnpm specに固定されていない                           | プラグインインストールメタデータ                                                                     | なし     |
| `checkId`                                                     | 重大度        | 重要な理由                                                                             | 主な修正キー／パス                                                                                   | 自動修正 |
| ------------------------------------------------------------- | ------------- | -------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- | -------- |
| `plugins.installs_missing_integrity`                          | warn          | プラグインインストール記録に整合性メタデータがない                                    | プラグインインストールメタデータ                                                                     | なし     |
| `plugins.installs_version_drift`                              | warn          | プラグインインストール記録がインストール済みパッケージとずれている                     | プラグインインストールメタデータ                                                                     | なし     |
| `plugins.code_safety`                                         | warn/critical | プラグインコードスキャンで疑わしいまたは危険なパターンが見つかった                     | プラグインコード／インストール元                                                                     | なし     |
| `plugins.code_safety.entry_path`                              | warn          | プラグインのエントリーパスが隠し場所または `node_modules` 配下を指している             | プラグインマニフェストの `entry`                                                                     | なし     |
| `plugins.code_safety.entry_escape`                            | critical      | プラグインのエントリーがプラグインディレクトリ外へ抜けている                           | プラグインマニフェストの `entry`                                                                     | なし     |
| `plugins.code_safety.scan_failed`                             | warn          | プラグインコードスキャンを完了できなかった                                             | プラグイン拡張パス／スキャン環境                                                                     | なし     |
| `skills.code_safety`                                          | warn/critical | Skillインストーラーメタデータ／コードに疑わしいまたは危険なパターンが含まれる          | Skillインストール元                                                                                  | なし     |
| `skills.code_safety.scan_failed`                              | warn          | Skillコードスキャンを完了できなかった                                                  | Skillスキャン環境                                                                                    | なし     |
| `security.exposure.open_channels_with_exec`                   | warn/critical | 共有／公開ルームからexec有効化済みエージェントへ到達できる                             | `channels.*.dmPolicy`、`channels.*.groupPolicy`、`tools.exec.*`、`agents.list[].tools.exec.*`      | なし     |
| `security.exposure.open_groups_with_elevated`                 | critical      | オープングループ + 昇格ツールは、影響の大きいプロンプトインジェクション経路を作る       | `channels.*.groupPolicy`、`tools.elevated.*`                                                         | なし     |
| `security.exposure.open_groups_with_runtime_or_fs`            | critical/warn | オープングループから、sandbox／workspaceガードなしでコマンド／ファイルツールへ到達できる | `channels.*.groupPolicy`、`tools.profile/deny`、`tools.fs.workspaceOnly`、`agents.*.sandbox.mode` | なし     |
| `security.trust_model.multi_user_heuristic`                   | warn          | Gatewayの信頼モデルはパーソナルアシスタントなのに、configがマルチユーザーに見える       | 信頼境界を分離する、または共有ユーザー向け強化（`sandbox.mode`、tool deny／workspaceスコープ化）   | なし     |
| `tools.profile_minimal_overridden`                            | warn          | エージェントの上書きがグローバルなminimalプロファイルを迂回している                     | `agents.list[].tools.profile`                                                                        | なし     |
| `plugins.tools_reachable_permissive_policy`                   | warn          | 緩いポリシー文脈で拡張ツールへ到達可能                                                  | `tools.profile` + tool allow/deny                                                                    | なし     |
| `models.legacy`                                               | warn          | レガシーモデルファミリーがまだ設定されている                                            | モデル選択                                                                                           | なし     |
| `models.weak_tier`                                            | warn          | 設定されたモデルが現在の推奨ティアを下回っている                                        | モデル選択                                                                                           | なし     |
| `models.small_params`                                         | critical/info | 小規模モデル + 安全でないツールサーフェスはインジェクションリスクを高める               | モデル選択 + sandbox／ツールポリシー                                                                 | なし     |
| `summary.attack_surface`                                      | info          | auth、チャネル、ツール、露出姿勢の要約                                                  | 複数のキー（検出詳細を参照）                                                                         | なし     |

## HTTP経由のControl UI

Control UIがデバイスアイデンティティを生成するには、**安全なコンテキスト**（HTTPSまたはlocalhost）が必要です。`gateway.controlUi.allowInsecureAuth` はローカル互換用のトグルです。

- localhost上では、ページが安全でないHTTPで読み込まれた場合でも、デバイスアイデンティティなしでControl UI認証を許可します。
- これはペアリングチェックをバイパスしません。
- リモート（localhost以外）のデバイスアイデンティティ要件は緩和しません。

HTTPS（Tailscale Serve）を優先するか、UIを `127.0.0.1` 上で開いてください。

緊急時専用の手段として、`gateway.controlUi.dangerouslyDisableDeviceAuth` はデバイスアイデンティティチェックを完全に無効化します。これは重大なセキュリティ低下です。積極的にデバッグしていて、すぐに元へ戻せる場合を除き、無効のままにしてください。

これらの危険なフラグとは別に、`gateway.auth.mode: "trusted-proxy"` が成功すると、デバイスアイデンティティなしで**運用者**のControl UIセッションを受け入れることがあります。これは意図されたauthモードの挙動であり、`allowInsecureAuth` の近道ではありません。また、nodeロールのControl UIセッションには適用されません。

`openclaw security audit` は、この設定が有効な場合に警告します。

## 安全でない／危険なフラグの概要

`openclaw security audit` は、既知の安全でない／危険なデバッグスイッチが有効な場合、`config.insecure_or_dangerous_flags` を含めます。現在このチェックで集約されるのは次のとおりです。

- `gateway.controlUi.allowInsecureAuth=true`
- `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback=true`
- `gateway.controlUi.dangerouslyDisableDeviceAuth=true`
- `hooks.gmail.allowUnsafeExternalContent=true`
- `hooks.mappings[<index>].allowUnsafeExternalContent=true`
- `tools.exec.applyPatch.workspaceOnly=false`
- `plugins.entries.acpx.config.permissionMode=approve-all`

OpenClawのconfigスキーマで定義されている、完全な `dangerous*` / `dangerously*` configキーは次のとおりです。

- `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback`
- `gateway.controlUi.dangerouslyDisableDeviceAuth`
- `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork`
- `channels.discord.dangerouslyAllowNameMatching`
- `channels.discord.accounts.<accountId>.dangerouslyAllowNameMatching`
- `channels.slack.dangerouslyAllowNameMatching`
- `channels.slack.accounts.<accountId>.dangerouslyAllowNameMatching`
- `channels.googlechat.dangerouslyAllowNameMatching`
- `channels.googlechat.accounts.<accountId>.dangerouslyAllowNameMatching`
- `channels.msteams.dangerouslyAllowNameMatching`
- `channels.synology-chat.dangerouslyAllowNameMatching`（拡張チャネル）
- `channels.synology-chat.accounts.<accountId>.dangerouslyAllowNameMatching`（拡張チャネル）
- `channels.synology-chat.dangerouslyAllowInheritedWebhookPath`（拡張チャネル）
- `channels.zalouser.dangerouslyAllowNameMatching`（拡張チャネル）
- `channels.zalouser.accounts.<accountId>.dangerouslyAllowNameMatching`（拡張チャネル）
- `channels.irc.dangerouslyAllowNameMatching`（拡張チャネル）
- `channels.irc.accounts.<accountId>.dangerouslyAllowNameMatching`（拡張チャネル）
- `channels.mattermost.dangerouslyAllowNameMatching`（拡張チャネル）
- `channels.mattermost.accounts.<accountId>.dangerouslyAllowNameMatching`（拡張チャネル）
- `channels.telegram.network.dangerouslyAllowPrivateNetwork`
- `channels.telegram.accounts.<accountId>.network.dangerouslyAllowPrivateNetwork`
- `agents.defaults.sandbox.docker.dangerouslyAllowReservedContainerTargets`
- `agents.defaults.sandbox.docker.dangerouslyAllowExternalBindSources`
- `agents.defaults.sandbox.docker.dangerouslyAllowContainerNamespaceJoin`
- `agents.list[<index>].sandbox.docker.dangerouslyAllowReservedContainerTargets`
- `agents.list[<index>].sandbox.docker.dangerouslyAllowExternalBindSources`
- `agents.list[<index>].sandbox.docker.dangerouslyAllowContainerNamespaceJoin`

## リバースプロキシ設定

Gatewayをリバースプロキシ（nginx、Caddy、Traefikなど）の背後で動かす場合は、転送されたクライアントIPを正しく扱うために `gateway.trustedProxies` を設定してください。

Gatewayが、`trustedProxies` に**含まれていない**アドレスからのプロキシヘッダーを検出した場合、その接続をローカルクライアントとしては扱いません。Gateway authが無効なら、その接続は拒否されます。これにより、プロキシ経由の接続がlocalhostから来たように見えて自動的に信頼される、という認証バイパスを防げます。

`gateway.trustedProxies` は `gateway.auth.mode: "trusted-proxy"` にも使われますが、このauthモードはさらに厳格です。

- trusted-proxy authは**loopback送信元のプロキシに対してフェイルクローズ**します。
- 同一ホスト上のloopbackリバースプロキシは、ローカルクライアント判定と転送IP処理のために `gateway.trustedProxies` を引き続き利用できます。
- 同一ホスト上のloopbackリバースプロキシでは、`gateway.auth.mode: "trusted-proxy"` ではなく、token/password authを使用してください。

```yaml
gateway:
  trustedProxies:
    - "10.0.0.1" # reverse proxy IP
  # 任意。デフォルトはfalse。
  # プロキシが X-Forwarded-For を提供できない場合にのみ有効化してください。
  allowRealIpFallback: false
  auth:
    mode: password
    password: ${OPENCLAW_GATEWAY_PASSWORD}
```

`trustedProxies` が設定されている場合、GatewayはクライアントIPの判定に `X-Forwarded-For` を使います。`X-Real-IP` は、`gateway.allowRealIpFallback: true` が明示的に設定されていない限り、デフォルトでは無視されます。

良いリバースプロキシの挙動（受信した転送ヘッダーを上書きする）:

```nginx
proxy_set_header X-Forwarded-For $remote_addr;
proxy_set_header X-Real-IP $remote_addr;
```

悪いリバースプロキシの挙動（信頼されていない転送ヘッダーを追記／保持する）:

```nginx
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

## HSTSとオリジンに関する注意

- OpenClaw Gatewayは、まずlocal／loopbackを前提としています。TLSをリバースプロキシで終端する場合は、そのプロキシ側のHTTPSドメインでHSTSを設定してください。
- Gateway自身がHTTPSを終端する場合は、`gateway.http.securityHeaders.strictTransportSecurity` を設定して、OpenClawのレスポンスからHSTSヘッダーを送出できます。
- 詳細なデプロイ指針は [Trusted Proxy Auth](/ja-JP/gateway/trusted-proxy-auth#tls-termination-and-hsts) にあります。
- loopback以外でControl UIをデプロイする場合、`gateway.controlUi.allowedOrigins` はデフォルトで必須です。
- `gateway.controlUi.allowedOrigins: ["*"]` は、明示的な「すべて許可」のブラウザオリジンポリシーであり、強化されたデフォルトではありません。厳密に管理されたローカルテスト以外では避けてください。
- loopback上で一般的なloopback免除が有効な場合でも、ブラウザオリジン認証の失敗には引き続きレート制限がかかります。ただし、ロックアウトキーは共有のlocalhostバケット1つではなく、正規化された `Origin` 値ごとにスコープされます。
- `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback=true` は、Hostヘッダー由来のオリジンフォールバックモードを有効にします。運用者が選択した危険なポリシーとして扱ってください。
- DNS rebindingやプロキシのHostヘッダー挙動は、デプロイ強化上の懸念として扱ってください。`trustedProxies` は厳密に保ち、Gatewayを公開インターネットへ直接露出しないでください。

## ローカルセッションログはディスク上に保存される

OpenClawは、セッショントランスクリプトを `~/.openclaw/agents/<agentId>/sessions/*.jsonl` に保存します。  
これはセッション継続性と、必要に応じてセッションメモリーのインデックス化に必要ですが、同時に**ファイルシステムへアクセスできる任意のプロセス／ユーザーがそのログを読める**ことも意味します。信頼境界はディスクアクセスだと考え、`~/.openclaw` の権限を厳しくしてください（下の監査セクションを参照）。エージェント間でより強い分離が必要なら、別々のOSユーザーまたは別々のホストで実行してください。

## Node実行（system.run）

macOS nodeがペアリングされている場合、Gatewayはそのnode上で `system.run` を呼び出せます。これはMac上での**リモートコード実行**です。

- nodeのペアリング（承認 + token）が必要です。
- Gatewayのnodeペアリングは、コマンド単位の承認サーフェスではありません。これはnodeのアイデンティティ／信頼とtoken発行を確立するものです。
- Gatewayは、`gateway.nodes.allowCommands` / `denyCommands` を通じて、大まかなグローバルnodeコマンドポリシーを適用します。
- Mac側では **Settings → Exec approvals** で制御します（security + ask + allowlist）。
- node単位の `system.run` ポリシーは、そのnode自身のexec approvalsファイル（`exec.approvals.node.*`）であり、GatewayのグローバルなコマンドIDポリシーより厳しい場合も緩い場合もあります。
- `security="full"` かつ `ask="off"` で動作するnodeは、デフォルトの信頼された運用者モデルに従っています。デプロイでより厳密な承認または許可リスト方針が明示的に必要でない限り、これは想定内の挙動として扱ってください。
- 承認モードは、正確なリクエストコンテキストと、可能な場合は1つの具体的なローカルスクリプト／ファイルオペランドに結び付きます。インタープリター／ランタイムコマンドに対して、OpenClawが正確に1つの直接ローカルファイルを特定できない場合、完全な意味的カバレッジを約束するのではなく、承認ベース実行は拒否されます。
- `host=node` の場合、承認ベース実行では、整形済みの正規 `systemRunPlan` も保存されます。その後の承認済み転送ではその保存済みプランが再利用され、承認リクエスト作成後に呼び出し元がコマンド／cwd／セッションコンテキストを編集しようとすると、Gateway側の検証で拒否されます。
- リモート実行が不要なら、securityを **deny** に設定し、そのMacのnodeペアリングを削除してください。

この区別はトリアージで重要です。

- 再接続したペアリング済みnodeが別のコマンド一覧を広告していても、それだけでは脆弱性ではありません。Gatewayのグローバルポリシーとnodeのローカルexec approvalsが、実際の実行境界を引き続き強制している限りは問題ではありません。
- nodeペアリングメタデータを、隠れた第2のコマンド単位承認レイヤーとして扱う報告は、通常はポリシー／UX上の混乱であり、セキュリティ境界の回避ではありません。

## 動的Skills（watcher / remote nodes）

OpenClawはセッションの途中でSkills一覧を更新できます。

- **Skills watcher**: `SKILL.md` の変更により、次のエージェントターンでSkillsスナップショットが更新されることがあります。
- **Remote nodes**: macOS nodeを接続すると、macOS専用Skillsが対象になることがあります（binのプローブに基づく）。

Skillフォルダーは**信頼されたコード**として扱い、誰が変更できるかを制限してください。

## 脅威モデル

あなたのAIアシスタントは次のことができます。

- 任意のシェルコマンドを実行する
- ファイルを読み書きする
- ネットワークサービスへアクセスする
- 誰にでもメッセージを送る（WhatsAppアクセスを与えた場合）

あなたにメッセージを送れる人は次のことができます。

- AIをだまして悪いことをさせようとする
- あなたのデータへのアクセスをソーシャルエンジニアリングで引き出そうとする
- インフラの詳細を探ろうとする

## 中核概念: 知能より先にアクセス制御

ここで起きる失敗の多くは、高度な攻撃ではありません。単に「誰かがボットにメッセージし、ボットがその依頼どおりに動いた」というものです。

OpenClawの立場は次のとおりです。

- **まずアイデンティティ:** 誰がボットに話しかけられるかを決める（DMペアリング／許可リスト／明示的な `「open」`）。
- **次にスコープ:** ボットがどこで動作してよいかを決める（グループ許可リスト + メンションゲーティング、ツール、sandbox化、デバイス権限）。
- **最後にモデル:** モデルは操作されうるものだと想定し、操作されても影響範囲が限定されるように設計する。

## コマンド認可モデル

スラッシュコマンドとディレクティブは、**認可された送信者**に対してのみ処理されます。認可は、チャネルの許可リスト／ペアリングと `commands.useAccessGroups` から導出されます（[Configuration](/ja-JP/gateway/configuration) および [Slash commands](/ja-JP/tools/slash-commands) を参照）。チャネル許可リストが空、または `"*"` を含む場合、そのチャネルのコマンドは事実上オープンです。

`/exec` は、認可された運用者向けのセッション限定の利便機能です。これはconfigへ書き込んだり、他のセッションを変更したりはしません。

## コントロールプレーンツールのリスク

2つの組み込みツールは、永続的なコントロールプレーン変更を行えます。

- `gateway` は `config.schema.lookup` / `config.get` でconfigを調べられ、`config.apply`、`config.patch`、`update.run` で永続変更を行えます。
- `cron` は、元のチャット／タスク終了後も動き続けるスケジュールジョブを作成できます。

オーナー専用の `gateway` ランタイムツールは、現在も `tools.exec.ask` または `tools.exec.security` の書き換えを拒否します。レガシーな `tools.bash.*` エイリアスは、書き込み前に同じ保護されたexecパスへ正規化されます。

信頼していないコンテンツを扱うエージェント／サーフェスでは、デフォルトでこれらを拒否してください。

```json5
{
  tools: {
    deny: ["gateway", "cron", "sessions_spawn", "sessions_send"],
  },
}
```

`commands.restart=false` は再起動アクションをブロックするだけです。`gateway` のconfig／updateアクションは無効化しません。

## プラグイン／拡張機能

プラグインはGateway内で**インプロセス**で実行されます。信頼されたコードとして扱ってください。

- 信頼できる提供元からのプラグインだけをインストールしてください。
- 明示的な `plugins.allow` 許可リストを優先してください。
- 有効化する前にプラグイン設定を確認してください。
- プラグイン変更後はGatewayを再起動してください。
- プラグインをインストールまたは更新する場合（`openclaw plugins install <package>`、`openclaw plugins update <id>`）、これは信頼していないコードを実行するのと同じだと考えてください。
  - インストール先は、現在有効なプラグインインストールルート配下にある、プラグインごとのディレクトリです。
  - OpenClawは、インストール／更新前に組み込みの危険コードスキャンを実行します。`critical` 検出はデフォルトでブロックされます。
  - OpenClawは `npm pack` を使用し、その後そのディレクトリ内で `npm install --omit=dev` を実行します（npmのライフサイクルスクリプトは、インストール中にコードを実行できます）。
  - 固定された厳密バージョン（`@scope/pkg@1.2.3`）を優先し、有効化前にディスク上へ展開されたコードを確認してください。
  - `--dangerously-force-unsafe-install` は、プラグインのインストール／更新フローにおける組み込みスキャンの誤検知に対する緊急手段専用です。これはプラグインの `before_install` フックによるポリシーブロックを回避せず、スキャン失敗も回避しません。
  - Gateway連動のSkill依存関係インストールも、同じ危険／疑わしいの区別に従います。組み込みの `critical` 検出は、呼び出し元が明示的に `dangerouslyForceUnsafeInstall` を設定しない限りブロックされ、疑わしい検出は引き続き警告のみです。`openclaw skills install` は、引き続き別個のClawHub Skillダウンロード／インストールフローです。

詳細: [Plugins](/ja-JP/tools/plugin)

<a id="dm-access-model-pairing-allowlist-open-disabled"></a>

## DMアクセスモデル（pairing / allowlist / open / disabled）

現在DMに対応しているすべてのチャネルは、メッセージが処理される**前に**受信DMを制御するDMポリシー（`dmPolicy` または `*.dm.policy`）をサポートしています。

- `pairing`（デフォルト）: 未知の送信者には短いペアリングコードが送られ、承認されるまでボットはそのメッセージを無視します。コードの有効期限は1時間です。新しいリクエストが作成されるまでは、繰り返しDMしてもコードは再送されません。保留中リクエストは、デフォルトで**チャネルごとに3件**までに制限されます。
- `allowlist`: 未知の送信者はブロックされます（ペアリングのハンドシェイクなし）。
- `open`: 誰でもDM可能にします（公開）。チャネル許可リストに `"*"` を含めることが**必須**です（明示的なオプトイン）。
- `disabled`: 受信DMを完全に無視します。

CLIで承認するには:

```bash
openclaw pairing list <channel>
openclaw pairing approve <channel> <code>
```

詳細とディスク上のファイル: [Pairing](/ja-JP/channels/pairing)

## DMセッション分離（マルチユーザーモード）

デフォルトでは、OpenClawは**すべてのDMをメインセッションへルーティング**するため、あなたのアシスタントはデバイスやチャネルをまたいで継続性を保てます。**複数人**がボットへDMできる場合（オープンDMや複数人の許可リスト）、DMセッションを分離することを検討してください。

```json5
{
  session: { dmScope: "per-channel-peer" },
}
```

これにより、グループチャットの分離を維持しつつ、ユーザー間のコンテキスト漏えいを防げます。

これはメッセージングコンテキストの境界であり、ホスト管理者境界ではありません。ユーザー同士が敵対的で、同じGatewayホスト／configを共有している場合は、信頼境界ごとに別々のGatewayを実行してください。

### セキュアDMモード（推奨）

上記のスニペットは、**セキュアDMモード**として扱ってください。

- デフォルト: `session.dmScope: "main"`（継続性のため、すべてのDMが1つのセッションを共有）
- ローカルCLIオンボーディングのデフォルト: 未設定時は `session.dmScope: "per-channel-peer"` を書き込む（既存の明示値は保持）
- セキュアDMモード: `session.dmScope: "per-channel-peer"`（各チャネル + 送信者の組み合わせごとに独立したDMコンテキスト）
- チャネル横断の送信者分離: `session.dmScope: "per-peer"`（同じタイプの全チャネルを通して、送信者ごとに1つのセッション）

同じチャネル上で複数アカウントを動かす場合は、代わりに `per-account-channel-peer` を使ってください。同じ人物が複数チャネルから連絡してくる場合は、`session.identityLinks` を使ってそれらのDMセッションを1つの正規アイデンティティへ統合します。詳細は [Session Management](/ja-JP/concepts/session) と [Configuration](/ja-JP/gateway/configuration) を参照してください。

## 許可リスト（DM + グループ）- 用語

OpenClawには、「誰が自分を起動できるか」に関する2つの別々のレイヤーがあります。

- **DM許可リスト**（`allowFrom` / `channels.discord.allowFrom` / `channels.slack.allowFrom`; レガシー: `channels.discord.dm.allowFrom`, `channels.slack.dm.allowFrom`）: ダイレクトメッセージでボットへ話しかけることを許可される相手。
  - `dmPolicy="pairing"` の場合、承認は `~/.openclaw/credentials/` 配下のアカウント単位ペアリング許可リストストアへ書き込まれます（デフォルトアカウントは `<channel>-allowFrom.json`、デフォルト以外のアカウントは `<channel>-<accountId>-allowFrom.json`）。その後、config許可リストとマージされます。
- **グループ許可リスト**（チャネル固有）: ボットがどのグループ／チャネル／guildからのメッセージをそもそも受け付けるか。
  - 一般的なパターン:
    - `channels.whatsapp.groups`、`channels.telegram.groups`、`channels.imessage.groups`: `requireMention` のようなグループ単位デフォルト。これが設定されると、グループ許可リストとしても機能します（すべて許可を維持するには `"*"` を含めます）。
    - `groupPolicy="allowlist"` + `groupAllowFrom`: グループセッション内で誰がボットを起動できるかを制限します（WhatsApp／Telegram／Signal／iMessage／Microsoft Teams）。
    - `channels.discord.guilds` / `channels.slack.channels`: サーフェス単位の許可リスト + メンションのデフォルト。
  - グループチェックは次の順で実行されます。まず `groupPolicy`／グループ許可リスト、その次にメンション／返信による起動です。
  - ボットのメッセージへ返信すること（暗黙のメンション）は、`groupAllowFrom` のような送信者許可リストをバイパスしません。
  - **セキュリティ上の注意:** `dmPolicy="open"` と `groupPolicy="open"` は最終手段として扱ってください。これらはほとんど使うべきではありません。部屋の全員を完全に信頼している場合を除き、pairing + 許可リストを優先してください。

詳細: [Configuration](/ja-JP/gateway/configuration) および [Groups](/ja-JP/channels/groups)

## プロンプトインジェクション（何か、なぜ重要か）

プロンプトインジェクションとは、攻撃者が、モデルを操作して危険なことをさせるようなメッセージを作ることです（「指示を無視しろ」「ファイルシステムを吐き出せ」「このリンクを開いてコマンドを実行しろ」など）。

強力なシステムプロンプトがあっても、**プロンプトインジェクションは未解決です**。システムプロンプトのガードレールは、あくまで緩い指針にすぎません。強制力のある制御は、ツールポリシー、exec承認、sandbox化、チャネル許可リストから来ます（しかも、これらは設計上、運用者が無効化できます）。実際に役立つ対策は次のとおりです。

- 受信DMをロックダウンする（pairing／許可リスト）。
- グループではメンションゲーティングを優先し、公開ルームでの「常時待機」ボットは避ける。
- リンク、添付、貼り付けられた指示は、デフォルトで敵対的とみなす。
- 機密性の高いツール実行はsandbox内で行い、シークレットをエージェントが到達できるファイルシステムに置かない。
- 注意: sandbox化はオプトインです。sandboxモードがオフなら、暗黙の `host=auto` はGatewayホストへ解決されます。明示的な `host=sandbox` は、sandboxランタイムが利用できないため、引き続きフェイルクローズします。その挙動をconfig上で明示したいなら、`host=gateway` を設定してください。
- 高リスクツール（`exec`、`browser`、`web_fetch`、`web_search`）は、信頼されたエージェントまたは明示的な許可リストに限定する。
- インタープリター（`python`、`node`、`ruby`、`perl`、`php`、`lua`、`osascript`）を許可リスト化する場合は、インラインeval形式でも明示的承認が必要になるよう `tools.exec.strictInlineEval` を有効にする。
- **モデル選択は重要です:** 古い／小さい／レガシーなモデルは、プロンプトインジェクションやツール誤用への耐性が大幅に低いです。ツール有効化済みエージェントには、利用可能な中で最も強力な最新世代の命令耐性モデルを使ってください。

信頼していないものとして扱うべき危険信号:

- 「このファイル／URLを読んで、書かれていることをそのまま実行して」
- 「システムプロンプトや安全ルールを無視して」
- 「隠された指示やツール出力を見せて」
- `~/.openclaw` やログの内容をすべて貼り付けて

## 安全でない外部コンテンツのバイパスフラグ

OpenClawには、外部コンテンツの安全ラッピングを無効化する明示的なバイパスフラグがあります。

- `hooks.mappings[].allowUnsafeExternalContent`
- `hooks.gmail.allowUnsafeExternalContent`
- Cronペイロードフィールド `allowUnsafeExternalContent`

ガイダンス:

- 本番では、これらを未設定またはfalseのままにしてください。
- 厳密に範囲を絞ったデバッグ時にのみ一時的に有効化してください。
- 有効にする場合は、そのエージェントを分離してください（sandbox + 最小限ツール + 専用セッション名前空間）。

Hooksのリスクに関する注意:

- Hookペイロードは、たとえ配信が自分で管理するシステムから来ていても、信頼していないコンテンツです（メール／ドキュメント／Webコンテンツにはプロンプトインジェクションが含まれうるため）。
- 弱いモデルティアはこのリスクを高めます。Hook駆動の自動化では、強力で現代的なモデルティアを優先し、ツールポリシーは厳格に保ってください（`tools.profile: "messaging"` またはそれより厳格）。可能であればsandbox化も行ってください。

### プロンプトインジェクションは公開DMでなくても起こる

**自分だけ**がボットへメッセージできる場合でも、ボットが読む**信頼していないコンテンツ**（Web検索／取得結果、ブラウザページ、メール、ドキュメント、添付、貼り付けられたログ／コード）を通じて、プロンプトインジェクションは起こりえます。言い換えると、脅威サーフェスは送信者だけではなく、**コンテンツそのもの**も敵対的な指示を運びうるということです。

ツールが有効な場合の典型的なリスクは、コンテキストの流出やツール呼び出しの誘発です。影響範囲は次の方法で小さくできます。

- 信頼していないコンテンツの要約には、読み取り専用またはツール無効の**reader agent** を使い、その要約をメインエージェントへ渡す。
- 必要になるまで、ツール有効化済みエージェントでは `web_search` / `web_fetch` / `browser` をオフにしておく。
- OpenResponsesのURL入力（`input_file` / `input_image`）では、`gateway.http.endpoints.responses.files.urlAllowlist` と `gateway.http.endpoints.responses.images.urlAllowlist` を厳しく設定し、`maxUrlParts` は低く保つ。
  空の許可リストは未設定として扱われます。URL取得を完全に無効化したいなら、`files.allowUrl: false` / `images.allowUrl: false` を使ってください。
- OpenResponsesのファイル入力では、デコードされた `input_file` のテキストも引き続き**信頼していない外部コンテンツ**として注入されます。Gatewayがローカルでデコードしたからといって、そのファイルテキストを信頼してはいけません。注入されるブロックには、長い `SECURITY NOTICE:` バナーはありませんが、明示的な `<<<EXTERNAL_UNTRUSTED_CONTENT ...>>>` 境界マーカーと `Source: External` メタデータが引き続き付きます。
- 添付ドキュメントからテキストを抽出し、それをメディアプロンプトへ追加する前にも、同じマーカーベースのラッピングが適用されます。
- 信頼していない入力に触れるエージェントでは、sandbox化と厳格なツール許可リストを有効にする。
- シークレットをプロンプト内に置かず、Gatewayホスト上のenv/config経由で渡す。

### モデルの強さ（セキュリティ上の注意）

プロンプトインジェクション耐性は、モデルティア間で**均一ではありません**。小型で安価なモデルほど、特に敵対的プロンプト下では、ツール誤用や命令ハイジャックに弱い傾向があります。

<Warning>
ツール有効化済みエージェントや、信頼していないコンテンツを読むエージェントでは、古い／小さいモデルによるプロンプトインジェクションリスクは、しばしば高すぎます。そのようなワークロードを弱いモデルティアで実行しないでください。
</Warning>

推奨事項:

- ツールを実行できる、またはファイル／ネットワークへ触れられるボットには、**最新世代かつ最上位ティアのモデル**を使ってください。
- ツール有効化済みエージェントや、信頼していない受信箱には、**古い／弱い／小さいティアを使わないでください**。プロンプトインジェクションリスクが高すぎます。
- やむを得ず小さいモデルを使う場合は、**影響範囲を縮小**してください（読み取り専用ツール、強力なsandbox化、最小限のファイルシステムアクセス、厳格な許可リスト）。
- 小型モデルを実行する場合は、**すべてのセッションでsandbox化を有効にし**、入力が厳密に制御されていない限り **web_search/web_fetch/browser を無効化**してください。
- 信頼された入力のみでツールなしのチャット専用パーソナルアシスタントであれば、小型モデルでも通常は問題ありません。

<a id="reasoning-verbose-output-in-groups"></a>

## グループでのReasoningと詳細出力

`/reasoning` と `/verbose` は、公開チャネル向けではない内部推論やツール出力を露出することがあります。グループ設定では、これらは**デバッグ専用**として扱い、明確に必要な場合を除いて無効のままにしてください。

ガイダンス:

- 公開ルームでは `/reasoning` と `/verbose` を無効のままにしてください。
- 有効にする場合は、信頼されたDMまたは厳密に管理されたルームでのみ行ってください。
- 忘れないでください: 詳細出力には、ツール引数、URL、モデルが見たデータが含まれる可能性があります。

## 設定の強化（例）

### 0) ファイル権限

Gatewayホスト上では、config + stateを非公開に保ってください。

- `~/.openclaw/openclaw.json`: `600`（ユーザーのみ読み書き）
- `~/.openclaw`: `700`（ユーザーのみ）

`openclaw doctor` は、これらの権限について警告し、強化を提案できます。

### 0.4) ネットワーク露出（bind + port + firewall）

Gatewayは、単一ポート上で **WebSocket + HTTP** を多重化します。

- デフォルト: `18789`
- config／フラグ／env: `gateway.port`、`--port`、`OPENCLAW_GATEWAY_PORT`

このHTTPサーフェスには、Control UIとcanvas hostが含まれます。

- Control UI（SPAアセット）（デフォルトのベースパス `/`）
- Canvas host: `/__openclaw__/canvas/` および `/__openclaw__/a2ui/`（任意のHTML/JS。信頼していないコンテンツとして扱ってください）

通常のブラウザでcanvasコンテンツを読み込む場合は、他の信頼していないWebページと同様に扱ってください。

- 信頼していないネットワーク／ユーザーへcanvas hostを公開しないでください。
- 影響を十分に理解していない限り、canvasコンテンツを特権的なWebサーフェスと同一オリジンで共有させないでください。

bindモードは、Gatewayがどこで待ち受けるかを制御します。

- `gateway.bind: "loopback"`（デフォルト）: ローカルクライアントだけが接続できます。
- loopback以外のbind（`"lan"`、`"tailnet"`、`"custom"`）は攻撃面を広げます。これらは、gateway auth（共有token／password、または正しく設定された非loopback trusted proxy）と実際のファイアウォールがある場合にのみ使ってください。

経験則:

- LAN bindよりTailscale Serveを優先してください（ServeはGatewayをloopback上に保ち、アクセスはTailscaleが処理します）。
- どうしてもLANへbindする必要がある場合は、ポートへのアクセスを送信元IPの厳格な許可リストでファイアウォールしてください。広くポートフォワードしないでください。
- 認証なしのGatewayを `0.0.0.0` へ絶対に公開しないでください。

### 0.4.1) Dockerのポート公開 + UFW（`DOCKER-USER`）

VPS上でDockerを使ってOpenClawを動かす場合は、公開されたコンテナポート（`-p HOST:CONTAINER` またはComposeの `ports:`）が、ホストの `INPUT` ルールだけでなく、Dockerの転送チェーンを通ってルーティングされることを忘れないでください。

Dockerのトラフィックをファイアウォールポリシーに合わせるには、`DOCKER-USER` でルールを強制してください（このチェーンはDocker自身のacceptルールより前に評価されます）。最近の多くのディストリビューションでは、`iptables`／`ip6tables` は `iptables-nft` フロントエンドを使っており、それでもこれらのルールはnftablesバックエンドに適用されます。

最小限の許可リスト例（IPv4）:

```bash
# /etc/ufw/after.rules（独立した *filter セクションとして追記）
*filter
:DOCKER-USER - [0:0]
-A DOCKER-USER -m conntrack --ctstate ESTABLISHED,RELATED -j RETURN
-A DOCKER-USER -s 127.0.0.0/8 -j RETURN
-A DOCKER-USER -s 10.0.0.0/8 -j RETURN
-A DOCKER-USER -s 172.16.0.0/12 -j RETURN
-A DOCKER-USER -s 192.168.0.0/16 -j RETURN
-A DOCKER-USER -s 100.64.0.0/10 -j RETURN
-A DOCKER-USER -p tcp --dport 80 -j RETURN
-A DOCKER-USER -p tcp --dport 443 -j RETURN
-A DOCKER-USER -m conntrack --ctstate NEW -j DROP
-A DOCKER-USER -j RETURN
COMMIT
```

IPv6には別のテーブルがあります。Docker IPv6が有効なら、`/etc/ufw/after6.rules` に対応するポリシーも追加してください。

ドキュメントのスニペットに `eth0` のようなインターフェース名をハードコードしないでください。インターフェース名はVPSイメージごとに異なり（`ens3`、`enp*` など）、一致しないとdenyルールが意図せずスキップされることがあります。

再読み込み後の簡易確認:

```bash
ufw reload
iptables -S DOCKER-USER
ip6tables -S DOCKER-USER
nmap -sT -p 1-65535 <public-ip> --open
```

外部から見える想定ポートは、意図的に公開したものだけであるべきです（多くの構成ではSSH + リバースプロキシ用ポート）。

### 0.4.2) mDNS/Bonjour discovery（情報漏えい）

Gatewayは、ローカルデバイス探索のためにmDNS（ポート5353上の `_openclaw-gw._tcp`）で自身の存在をブロードキャストします。fullモードでは、運用上の詳細を露出しうるTXTレコードが含まれます。

- `cliPath`: CLIバイナリへの完全なファイルシステムパス（ユーザー名とインストール場所が分かる）
- `sshPort`: ホスト上でのSSH利用可否を広告する
- `displayName`、`lanHost`: ホスト名に関する情報

**運用セキュリティ上の考慮:** インフラの詳細をブロードキャストすると、ローカルネットワーク上の誰にとっても偵察が容易になります。ファイルシステムパスやSSH可用性のような「無害に見える」情報でも、攻撃者が環境を把握する助けになります。

**推奨事項:**

1. **Minimalモード**（デフォルト。公開Gateway向けに推奨）: mDNSブロードキャストから機微なフィールドを省略します。

   ```json5
   {
     discovery: {
       mdns: { mode: "minimal" },
     },
   }
   ```

2. ローカルデバイス探索が不要なら、**完全に無効化**してください。

   ```json5
   {
     discovery: {
       mdns: { mode: "off" },
     },
   }
   ```

3. **Fullモード**（オプトイン）: TXTレコードに `cliPath` + `sshPort` を含めます。

   ```json5
   {
     discovery: {
       mdns: { mode: "full" },
     },
   }
   ```

4. **環境変数**（代替手段）: configを変更せずにmDNSを無効化するには、`OPENCLAW_DISABLE_BONJOUR=1` を設定してください。

minimalモードでも、Gatewayはデバイス探索に十分な情報（`role`、`gatewayPort`、`transport`）は引き続きブロードキャストしますが、`cliPath` と `sshPort` は省略します。CLIパス情報が必要なアプリは、認証済みWebSocket接続経由で取得できます。

### 0.5) Gateway WebSocketをロックダウンする（ローカルauth）

Gateway authはデフォルトで**必須**です。有効なgateway auth経路が設定されていない場合、GatewayはWebSocket接続を拒否します（フェイルクローズ）。

オンボーディングはデフォルトでtokenを生成するため（loopbackであっても）、ローカルクライアントも認証が必要です。

**すべての**WSクライアントに認証を要求するには、tokenを設定してください。

```json5
{
  gateway: {
    auth: { mode: "token", token: "your-token" },
  },
}
```

Doctorはこれを生成できます: `openclaw doctor --generate-gateway-token`

注意: `gateway.remote.token` / `.password` はクライアント認証情報の取得元です。これら**単体では**ローカルWSアクセスを保護しません。  
ローカル呼び出し経路では、`gateway.auth.*` が未設定のときに限り、フォールバックとして `gateway.remote.*` を使用できます。  
`gateway.auth.token` / `gateway.auth.password` がSecretRef経由で明示的に設定されていて未解決の場合、解決はフェイルクローズし、リモートフォールバックで隠されることはありません。  
任意: `wss://` を使う場合は、`gateway.remote.tlsFingerprint` でリモートTLSをピン留めできます。  
平文の `ws://` はデフォルトでloopback専用です。信頼されたプライベートネットワーク経路での緊急手段として、クライアントプロセスに `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1` を設定してください。

ローカルデバイスのペアリング:

- 同一ホスト上のクライアント接続を円滑にするため、直接のローカルloopback接続ではデバイスペアリングは自動承認されます。
- OpenClawには、信頼された共有シークレットのヘルパーフロー向けに、限定的なバックエンド／コンテナローカルのセルフ接続経路もあります。
- Tailnet接続とLAN接続は、同一ホスト上のtailnet bindを含め、ペアリング上はリモートとして扱われ、引き続き承認が必要です。

authモード:

- `gateway.auth.mode: "token"`: 共有bearer token（ほとんどの構成で推奨）。
- `gateway.auth.mode: "password"`: password auth（envでの設定推奨: `OPENCLAW_GATEWAY_PASSWORD`）。
- `gateway.auth.mode: "trusted-proxy"`: アイデンティティ対応のリバースプロキシを信頼し、ヘッダー経由で渡されるアイデンティティでユーザーを認証します（[Trusted Proxy Auth](/ja-JP/gateway/trusted-proxy-auth) を参照）。

ローテーションチェックリスト（token／password）:

1. 新しいシークレットを生成／設定する（`gateway.auth.token` または `OPENCLAW_GATEWAY_PASSWORD`）。
2. Gatewayを再起動する（または、macOSアプリがGatewayを監督している場合は、そのアプリを再起動する）。
3. リモートクライアントを更新する（Gatewayへ接続する各マシン上の `gateway.remote.token` / `.password`）。
4. 古い認証情報では接続できなくなったことを確認する。

### 0.6) Tailscale Serveのアイデンティティヘッダー

`gateway.auth.allowTailscale` が `true` の場合（Serveではデフォルト）、OpenClawはControl UI／WebSocket認証にTailscale Serveのアイデンティティヘッダー（`tailscale-user-login`）を受け入れます。OpenClawは、`x-forwarded-for` アドレスをローカルのTailscaleデーモン（`tailscale whois`）で解決し、その結果をヘッダーと照合することでアイデンティティを検証します。これは、Tailscaleによって注入される `x-forwarded-for`、`x-forwarded-proto`、`x-forwarded-host` を含み、かつloopbackへ到達するリクエストに対してのみ発動します。  
この非同期アイデンティティチェック経路では、同じ `{scope, ip}` に対する失敗試行は、リミッターが失敗を記録する前に直列化されます。そのため、同一Serveクライアントからの並行した不正リトライでは、2回目の試行が単なる並走ミスマッチになる代わりに、即座にロックアウトされることがあります。  
HTTP APIエンドポイント（たとえば `/v1/*`、`/tools/invoke`、`/api/channels/*`）は、Tailscaleのアイデンティティヘッダーauthを**使用しません**。これらは引き続き、Gatewayに設定されたHTTP authモードに従います。

重要な境界上の注意:

- Gateway HTTP bearer authは、事実上、全面的な運用者アクセスです。
- `/v1/chat/completions`、`/v1/responses`、または `/api/channels/*` を呼び出せる認証情報は、そのGatewayに対するフルアクセスの運用者シークレットとして扱ってください。
- OpenAI互換HTTPサーフェスでは、共有シークレットbearer authは、デフォルトの完全な運用者スコープ（`operator.admin`、`operator.approvals`、`operator.pairing`、`operator.read`、`operator.talk.secrets`、`operator.write`）と、エージェントターンに対するownerセマンティクスを復元します。より狭い `x-openclaw-scopes` 値を送っても、この共有シークレット経路では権限は縮小されません。
- HTTPでのリクエスト単位スコープセマンティクスが適用されるのは、trusted proxy authや、プライベート受信面での `gateway.auth.mode="none"` のような、アイデンティティを持つモードから来るリクエストだけです。
- それらのアイデンティティ付きモードでは、`x-openclaw-scopes` を省略すると通常の運用者デフォルトスコープセットへフォールバックします。より狭いスコープにしたい場合は、ヘッダーを明示的に送ってください。
- `/tools/invoke` も同じ共有シークレットルールに従います。つまり、そこでのtoken／password bearer authもフル運用者アクセスとして扱われます。一方、アイデンティティ付きモードでは、宣言されたスコープが引き続き尊重されます。
- これらの認証情報を信頼していない呼び出し元と共有しないでください。信頼境界ごとに別々のGatewayを使うことを優先してください。

**信頼前提:** tokenなしのServe authは、gateway hostが信頼されていることを前提としています。これを、敵対的な同一ホストプロセスに対する保護だと考えないでください。信頼していないローカルコードがgateway host上で動作する可能性があるなら、`gateway.auth.allowTailscale` を無効にし、`gateway.auth.mode: "token"` または `"password"` による明示的な共有シークレットauthを要求してください。

**セキュリティルール:** これらのヘッダーを、自分のリバースプロキシから転送しないでください。Gatewayの手前でTLSを終端またはプロキシする場合は、`gateway.auth.allowTailscale` を無効にし、代わりに共有シークレットauth（`gateway.auth.mode: "token"` または `"password"`）か [Trusted Proxy Auth](/ja-JP/gateway/trusted-proxy-auth) を使用してください。

信頼するプロキシ:

- Gatewayの手前でTLSを終端する場合は、`gateway.trustedProxies` にプロキシIPを設定してください。
- OpenClawは、それらのIPからの `x-forwarded-for`（または `x-real-ip`）を信頼し、ローカルペアリングチェックやHTTP auth／ローカルチェックのためのクライアントIP判定に使います。
- プロキシが `x-forwarded-for` を**上書き**し、Gatewayポートへの直接アクセスをブロックしていることを確認してください。

参照: [Tailscale](/ja-JP/gateway/tailscale) および [Web overview](/web)

### 0.6.1) node host経由のブラウザ制御（推奨）

Gatewayがリモートにあり、ブラウザが別のマシンで動作している場合は、そのブラウザマシン上で **node host** を実行し、Gatewayにブラウザ操作をプロキシさせてください（[Browser tool](/ja-JP/tools/browser) を参照）。nodeペアリングは管理者アクセスとして扱ってください。

推奨パターン:

- Gatewayとnode hostは同じtailnet（Tailscale）上に保つ。
- nodeは意図的にペアリングし、ブラウザプロキシルーティングが不要なら無効化する。

避けるべきこと:

- リレーポートや制御ポートをLANや公開インターネットへ露出すること。
- ブラウザ制御エンドポイントにTailscale Funnelを使うこと（公開露出）。

### 0.7) ディスク上のシークレット（機微データ）

`~/.openclaw/`（または `$OPENCLAW_STATE_DIR/`）配下のものは、シークレットまたは非公開データを含みうると考えてください。

- `openclaw.json`: configには、token（gateway、remote gateway）、プロバイダー設定、許可リストが含まれる可能性があります。
- `credentials/**`: チャネル認証情報（例: WhatsApp creds）、ペアリング許可リスト、レガシーOAuthインポート。
- `agents/<agentId>/agent/auth-profiles.json`: APIキー、tokenプロファイル、OAuthトークン、および任意の `keyRef`／`tokenRef`。
- `secrets.json`（任意）: `file` SecretRefプロバイダー（`secrets.providers`）で使われるファイルベースのシークレットペイロード。
- `agents/<agentId>/agent/auth.json`: レガシー互換ファイル。静的な `api_key` エントリーは、発見時に削除されます。
- `agents/<agentId>/sessions/**`: セッショントランスクリプト（`*.jsonl`）+ ルーティングメタデータ（`sessions.json`）。プライベートメッセージやツール出力を含むことがあります。
- バンドル済みプラグインパッケージ: インストール済みプラグイン（およびその `node_modules/`）。
- `sandboxes/**`: ツールsandboxワークスペース。sandbox内で読み書きしたファイルのコピーが蓄積されることがあります。

強化のヒント:

- 権限を厳しく保つ（ディレクトリは `700`、ファイルは `600`）。
- Gatewayホストではフルディスク暗号化を使用する。
- ホストを共有する場合は、Gateway専用のOSユーザーアカウントを優先する。

### 0.8) ログ + トランスクリプト（マスキング + 保持）

アクセス制御が正しくても、ログやトランスクリプトから機微情報が漏れることがあります。

- Gatewayログには、ツールの要約、エラー、URLが含まれることがあります。
- セッショントランスクリプトには、貼り付けられたシークレット、ファイル内容、コマンド出力、リンクが含まれることがあります。

推奨事項:

- ツール要約のマスキングを有効のままにしてください（`logging.redactSensitive: "tools"`。デフォルト）。
- 自分の環境向けに `logging.redactPatterns` でカスタムパターンを追加してください（token、ホスト名、内部URLなど）。
- 診断情報を共有する場合は、生ログではなく、`openclaw status --all` を優先してください（貼り付けやすく、シークレットはマスク済み）。
- 長期保持が不要なら、古いセッショントランスクリプトやログファイルを削除してください。

詳細: [Logging](/ja-JP/gateway/logging)

### 1) DM: デフォルトでpairing

```json5
{
  channels: { whatsapp: { dmPolicy: "pairing" } },
}
```

### 2) グループ: すべてでメンション必須

```json
{
  "channels": {
    "whatsapp": {
      "groups": {
        "*": { "requireMention": true }
      }
    }
  },
  "agents": {
    "list": [
      {
        "id": "main",
        "groupChat": { "mentionPatterns": ["@openclaw", "@mybot"] }
      }
    ]
  }
}
```

グループチャットでは、明示的にメンションされたときだけ応答してください。

### 3) 番号を分ける（WhatsApp、Signal、Telegram）

電話番号ベースのチャネルでは、AI用に個人用とは別の電話番号で運用することを検討してください。

- 個人用番号: あなたの会話は非公開のまま
- ボット用番号: AIが適切な境界付きで対応する

### 4) 読み取り専用モード（sandbox + tools経由）

次を組み合わせることで、読み取り専用プロファイルを構築できます。

- `agents.defaults.sandbox.workspaceAccess: "ro"`（またはワークスペースアクセスなしなら `"none"`）
- `write`、`edit`、`apply_patch`、`exec`、`process` などをブロックするツールallow／denyリスト

追加の強化オプション:

- `tools.exec.applyPatch.workspaceOnly: true`（デフォルト）: sandbox化がオフでも、`apply_patch` がワークスペースディレクトリ外へ書き込み／削除できないようにします。`apply_patch` でワークスペース外のファイルへ触れさせたい意図がある場合にのみ `false` に設定してください。
- `tools.fs.workspaceOnly: true`（任意）: `read`／`write`／`edit`／`apply_patch` のパスと、ネイティブプロンプト画像の自動読み込みパスをワークスペースディレクトリに制限します（現在絶対パスを許可していて、単一のガードレールを追加したい場合に有用です）。
- ファイルシステムルートは狭く保ってください。エージェントワークスペースやsandboxワークスペースに、ホームディレクトリのような広いルートを使わないでください。広いルートは、機微なローカルファイル（たとえば `~/.openclaw` 配下のstate／config）をファイルシステムツールへ露出する可能性があります。

### 5) セキュアベースライン（コピー＆ペースト用）

Gatewayを非公開に保ち、DMにpairingを要求し、グループボットの常時応答を避ける、1つの `「安全なデフォルト」` configです。

```json5
{
  gateway: {
    mode: "local",
    bind: "loopback",
    port: 18789,
    auth: { mode: "token", token: "your-long-random-token" },
  },
  channels: {
    whatsapp: {
      dmPolicy: "pairing",
      groups: { "*": { requireMention: true } },
    },
  },
}
```

ツール実行も `「より安全なデフォルト」` にしたい場合は、sandboxを追加し、オーナー以外のエージェントでは危険なツールをdenyしてください（下の `「エージェントごとのアクセスプロファイル」` の例を参照）。

チャット駆動のエージェントターン向けの組み込みベースラインでは、オーナー以外の送信者は `cron` または `gateway` ツールを使用できません。

## Sandboxing（推奨）

専用ドキュメント: [Sandboxing](/ja-JP/gateway/sandboxing)

補完し合う2つの方法があります。

- **Gateway全体をDockerで実行する**（コンテナ境界）: [Docker](/ja-JP/install/docker)
- **ツールsandbox**（`agents.defaults.sandbox`、Gateway本体はホスト上 + ツールはDocker分離）: [Sandboxing](/ja-JP/gateway/sandboxing)

注意: エージェント間アクセスを防ぐには、`agents.defaults.sandbox.scope` を `"agent"`（デフォルト）のままにするか、より厳密なセッション単位分離が必要なら `"session"` を使ってください。`scope: "shared"` は単一のコンテナ／ワークスペースを使います。

また、sandbox内でのエージェントワークスペースアクセスも検討してください。

- `agents.defaults.sandbox.workspaceAccess: "none"`（デフォルト）では、エージェントワークスペースはアクセス不可のままで、ツールは `~/.openclaw/sandboxes` 配下のsandboxワークスペースに対して実行されます。
- `agents.defaults.sandbox.workspaceAccess: "ro"` は、エージェントワークスペースを `/agent` に読み取り専用でマウントします（`write`／`edit`／`apply_patch` を無効化）。
- `agents.defaults.sandbox.workspaceAccess: "rw"` は、エージェントワークスペースを `/workspace` に読み書き可能でマウントします。
- 追加の `sandbox.docker.binds` は、正規化およびcanonical化されたソースパスに対して検証されます。親symlinkのトリックやcanonical home aliasを使っても、`/etc`、`/var/run`、またはOSホーム配下の認証情報ディレクトリのようなブロック対象ルートへ解決される場合は、引き続きフェイルクローズします。

重要: `tools.elevated` は、sandbox外でexecを実行するグローバルなベースラインのエスケープハッチです。有効なhostはデフォルトで `gateway`、exec対象が `node` に設定されている場合は `node` です。`tools.elevated.allowFrom` は厳格に保ち、見知らぬ相手には有効化しないでください。さらに、エージェント単位で `agents.list[].tools.elevated` により制限できます。参照: [Elevated Mode](/ja-JP/tools/elevated)

### サブエージェント委任のガードレール

セッションツールを許可する場合、委任されたサブエージェント実行も別の境界判断として扱ってください。

- エージェントが本当に委任を必要としない限り、`sessions_spawn` をdenyしてください。
- `agents.defaults.subagents.allowAgents` と、エージェント単位の `agents.list[].subagents.allowAgents` 上書きは、既知の安全な対象エージェントに限定してください。
- sandbox化を維持すべきワークフローでは、`sessions_spawn` を `sandbox: "require"` で呼び出してください（デフォルトは `inherit`）。
- `sandbox: "require"` は、対象の子ランタイムがsandbox化されていない場合に即座に失敗します。

## ブラウザ制御のリスク

ブラウザ制御を有効にすると、モデルに実際のブラウザを操作する能力を与えることになります。  
そのブラウザプロファイルにログイン済みセッションが含まれている場合、モデルはそれらのアカウントやデータへアクセスできます。ブラウザプロファイルは**機微な状態**として扱ってください。

- エージェント専用のプロファイルを優先してください（デフォルトの `openclaw` プロファイル）。
- 個人で普段使っているプロファイルをエージェントに向けないでください。
- sandbox化されたエージェントでは、信頼していない限りホストのブラウザ制御を無効にしてください。
- スタンドアロンのloopbackブラウザ制御APIは、共有シークレットauth（gateway token bearer authまたはgateway password）だけを受け付けます。trusted-proxyやTailscale Serveのアイデンティティヘッダーは使用しません。
- ブラウザのダウンロードは信頼していない入力として扱い、分離されたダウンロードディレクトリを優先してください。
- 可能なら、エージェントプロファイルではブラウザ同期／パスワードマネージャーを無効にしてください（影響範囲を小さくするため）。
- リモートGatewayでは、`「ブラウザ制御」` は、そのプロファイルが到達できる範囲に対する `「運用者アクセス」` と同等だと考えてください。
- Gatewayとnode hostはtailnet限定に保ち、ブラウザ制御ポートをLANや公開インターネットへ露出しないでください。
- ブラウザプロキシルーティングが不要なら無効化してください（`gateway.nodes.browser.mode="off"`）。
- Chrome MCPの既存セッションモードは**より安全ではありません**。そのホストのChromeプロファイルが到達できる範囲に対して、あなたとして振る舞えます。

### ブラウザSSRFポリシー（デフォルトで厳格）

OpenClawのブラウザナビゲーションポリシーは、デフォルトで厳格です。プライベート／内部宛先は、明示的にオプトインしない限りブロックされたままです。

- デフォルト: `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork` は未設定であるため、ブラウザナビゲーションはプライベート／内部／特殊用途の宛先を引き続きブロックします。
- レガシーエイリアス: `browser.ssrfPolicy.allowPrivateNetwork` も互換性のため引き続き受け付けられます。
- オプトインモード: プライベート／内部／特殊用途の宛先を許可するには、`browser.ssrfPolicy.dangerouslyAllowPrivateNetwork: true` を設定します。
- 厳格モードでは、明示的な例外のために `hostnameAllowlist`（`*.example.com` のようなパターン）と `allowedHostnames`（`localhost` のようなブロック対象名を含む正確なホスト例外）を使用します。
- リダイレクト経由のピボットを減らすため、ナビゲーションはリクエスト前にチェックされ、ナビゲーション後の最終 `http(s)` URLに対してもベストエフォートで再チェックされます。

厳格ポリシーの例:

```json5
{
  browser: {
    ssrfPolicy: {
      dangerouslyAllowPrivateNetwork: false,
      hostnameAllowlist: ["*.example.com", "example.com"],
      allowedHostnames: ["localhost"],
    },
  },
}
```

## エージェントごとのアクセスプロファイル（マルチエージェント）

マルチエージェントルーティングでは、各エージェントに独自のsandbox + ツールポリシーを持たせられます。これを使って、エージェントごとに**フルアクセス**、**読み取り専用**、**アクセスなし**を与えてください。詳細と優先順位ルールは [Multi-Agent Sandbox & Tools](/ja-JP/tools/multi-agent-sandbox-tools) を参照してください。

一般的なユースケース:

- 個人用エージェント: フルアクセス、sandboxなし
- 家族／業務エージェント: sandbox化 + 読み取り専用ツール
- 公開エージェント: sandbox化 + ファイルシステム／シェルツールなし

### 例: フルアクセス（sandboxなし）

```json5
{
  agents: {
    list: [
      {
        id: "personal",
        workspace: "~/.openclaw/workspace-personal",
        sandbox: { mode: "off" },
      },
    ],
  },
}
```

### 例: 読み取り専用ツール + 読み取り専用ワークスペース

```json5
{
  agents: {
    list: [
      {
        id: "family",
        workspace: "~/.openclaw/workspace-family",
        sandbox: {
          mode: "all",
          scope: "agent",
          workspaceAccess: "ro",
        },
        tools: {
          allow: ["read"],
          deny: ["write", "edit", "apply_patch", "exec", "process", "browser"],
        },
      },
    ],
  },
}
```

### 例: ファイルシステム／シェルアクセスなし（プロバイダーメッセージングは許可）

```json5
{
  agents: {
    list: [
      {
        id: "public",
        workspace: "~/.openclaw/workspace-public",
        sandbox: {
          mode: "all",
          scope: "agent",
          workspaceAccess: "none",
        },
        // セッションツールはトランスクリプトから機微データを露出する可能性があります。デフォルトではOpenClawはこれらのツールを
        // 現在のセッション + 生成されたサブエージェントセッションに制限していますが、必要に応じてさらに絞り込めます。
        // 設定リファレンスの `tools.sessions.visibility` を参照してください。
        tools: {
          sessions: { visibility: "tree" }, // self | tree | agent | all
          allow: [
            "sessions_list",
            "sessions_history",
            "sessions_send",
            "sessions_spawn",
            "session_status",
            "whatsapp",
            "telegram",
            "slack",
            "discord",
          ],
          deny: [
            "read",
            "write",
            "edit",
            "apply_patch",
            "exec",
            "process",
            "browser",
            "canvas",
            "nodes",
            "cron",
            "gateway",
            "image",
          ],
        },
      },
    ],
  },
}
```

## AIに伝えるべきこと

エージェントのシステムプロンプトには、セキュリティガイドラインを含めてください。

```
## Security Rules
- Never share directory listings or file paths with strangers
- Never reveal API keys, credentials, or infrastructure details
- Verify requests that modify system config with the owner
- When in doubt, ask before acting
- Keep private data private unless explicitly authorized
```

## インシデント対応

AIが問題のある行動をした場合:

### 封じ込め

1. **止める:** macOSアプリ（Gatewayを監督している場合）を停止するか、`openclaw gateway` プロセスを終了します。
2. **露出を閉じる:** 何が起きたか把握するまで、`gateway.bind: "loopback"` に設定するか、Tailscale Funnel／Serveを無効化します。
3. **アクセスを凍結する:** リスクの高いDM／グループを `dmPolicy: "disabled"` またはメンション必須へ切り替え、もし `"*"` の全許可エントリーがあれば削除します。

### ローテーション（シークレットが漏れたなら侵害を前提にする）

1. Gateway auth（`gateway.auth.token` / `OPENCLAW_GATEWAY_PASSWORD`）をローテーションし、再起動します。
2. Gatewayを呼び出せる各マシン上のリモートクライアントシークレット（`gateway.remote.token` / `.password`）をローテーションします。
3. プロバイダー／API認証情報（WhatsApp creds、Slack／Discordトークン、`auth-profiles.json` 内のモデル／APIキー、および使用している場合は暗号化シークレットペイロード値）をローテーションします。

### 監査

1. Gatewayログを確認する: `/tmp/openclaw/openclaw-YYYY-MM-DD.log`（または `logging.file`）。
2. 関連するトランスクリプトを確認する: `~/.openclaw/agents/<agentId>/sessions/*.jsonl`。
3. 最近のconfig変更を確認する（アクセス拡大につながりうるもの: `gateway.bind`、`gateway.auth`、DM／グループポリシー、`tools.elevated`、プラグイン変更）。
4. `openclaw security audit --deep` を再実行し、criticalな検出結果が解消されたことを確認する。

### レポート用に収集する้อมูล

- タイムスタンプ、GatewayホストのOS + OpenClawバージョン
- セッショントランスクリプト + 短いログ末尾（マスク後）
- 攻撃者が送った内容 + エージェントが実行した内容
- Gatewayがloopback以外へ露出していたか（LAN／Tailscale Funnel／Serve）

## シークレットスキャン（detect-secrets）

CIは、`secrets` ジョブで `detect-secrets` のpre-commitフックを実行します。  
`main` へのpushでは常に全ファイルスキャンが実行されます。pull requestでは、ベースコミットが利用可能なら変更ファイルだけの高速経路を使い、そうでなければ全ファイルスキャンへフォールバックします。失敗した場合、ベースラインにまだ含まれていない新しい候補があることを意味します。

### CIが失敗した場合

1. ローカルで再現します:

   ```bash
   pre-commit run --all-files detect-secrets
   ```

2. ツールの挙動を理解します:
   - pre-commit内の `detect-secrets` は、リポジトリのベースラインと除外設定を使って `detect-secrets-hook` を実行します。
   - `detect-secrets audit` は対話式レビューを開き、各ベースライン項目を実際のシークレットか誤検知かとしてマークできます。
3. 実際のシークレットであれば、ローテーションまたは削除し、その後スキャンを再実行してベースラインを更新します。
4. 誤検知であれば、対話式監査を実行して false としてマークします:

   ```bash
   detect-secrets audit .secrets.baseline
   ```

5. 新しい除外が必要な場合は、それを `.detect-secrets.cfg` に追加し、対応する `--exclude-files` / `--exclude-lines` フラグでベースラインを再生成してください（configファイルは参照専用であり、detect-secretsはこれを自動では読み込みません）。

意図した状態を反映した `.secrets.baseline` をコミットしてください。

## セキュリティ問題の報告

OpenClawに脆弱性を見つけた場合は、責任ある形で報告してください。

1. メール: [security@openclaw.ai](mailto:security@openclaw.ai)
2. 修正されるまで公開投稿しない
3. 希望する場合は匿名で、そうでなければクレジットを掲載します
