---
read_when:
    - APIプロバイダーが失敗したときに、信頼できるフォールバックが必要です
    - Codex CLIやその他のローカルAI CLIを実行していて、それらを再利用したいと考えています
    - CLIバックエンドのツールアクセスのためのMCP loopbackブリッジを理解したいと考えています
summary: 'CLIバックエンド: オプションのMCPツールブリッジを備えたローカルAI CLIフォールバック'
title: CLIバックエンド
x-i18n:
    generated_at: "2026-04-11T02:44:18Z"
    model: gpt-5.4
    provider: openai
    source_hash: d108dbea043c260a80d15497639298f71a6b4d800f68d7b39bc129f7667ca608
    source_path: gateway/cli-backends.md
    workflow: 15
---

# CLIバックエンド（フォールバックランタイム）

OpenClawは、APIプロバイダーが停止している、レート制限されている、または一時的に不安定なときに、**テキスト専用のフォールバック**として**ローカルAI CLI**を実行できます。これは意図的に保守的な設計です。

- **OpenClawツールは直接注入されません**が、`bundleMcp: true` を持つバックエンドは、loopback MCPブリッジ経由でGatewayツールを受け取れます。
- それをサポートするCLI向けの**JSONLストリーミング**。
- **セッションをサポート**しているため、後続のターンでも一貫性が保たれます。
- CLIが画像パスを受け付ける場合は、**画像をそのまま渡す**ことができます。

これは主要な経路というより、**セーフティネット**として設計されています。外部APIに依存せず、「常に動作する」テキスト応答が必要な場合に使用してください。

ACPセッション制御、バックグラウンドタスク、スレッド/会話のバインディング、永続的な外部コーディングセッションを備えた完全なハーネスランタイムが必要な場合は、代わりに[ACP Agents](/ja-JP/tools/acp-agents)を使用してください。CLIバックエンドはACPではありません。

## 初心者向けクイックスタート

設定なしでもCodex CLIを使用できます（バンドルされたOpenAIプラグインがデフォルトのバックエンドを登録します）。

```bash
openclaw agent --message "hi" --model codex-cli/gpt-5.4
```

Gatewayがlaunchd/systemd配下で実行され、PATHが最小限の場合は、コマンドパスだけを追加してください。

```json5
{
  agents: {
    defaults: {
      cliBackends: {
        "codex-cli": {
          command: "/opt/homebrew/bin/codex",
        },
      },
    },
  },
}
```

これで完了です。CLI自体に必要なもの以外、キーも追加の認証設定も不要です。

バンドルされたCLIバックエンドをGatewayホスト上の**主要メッセージプロバイダー**として使用する場合、設定でモデル参照または`agents.defaults.cliBackends`の下にそのバックエンドを明示的に参照すると、OpenClawはそのバックエンドを所有するバンドルプラグインを自動で読み込みます。

## フォールバックとして使う

CLIバックエンドをフォールバックリストに追加すると、主要モデルが失敗したときだけ実行されます。

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["codex-cli/gpt-5.4"],
      },
      models: {
        "anthropic/claude-opus-4-6": { alias: "Opus" },
        "codex-cli/gpt-5.4": {},
      },
    },
  },
}
```

注意点:

- `agents.defaults.models`（許可リスト）を使う場合は、CLIバックエンドのモデルもそこに含める必要があります。
- 主要プロバイダーが失敗した場合（認証、レート制限、タイムアウト）、OpenClawは次にCLIバックエンドを試します。

## 設定の概要

すべてのCLIバックエンドは次の場所にあります。

```
agents.defaults.cliBackends
```

各エントリーは**provider id**（例: `codex-cli`、`my-cli`）をキーとして持ちます。  
provider idはモデル参照の左側になります。

```
<provider>/<model>
```

### 設定例

```json5
{
  agents: {
    defaults: {
      cliBackends: {
        "codex-cli": {
          command: "/opt/homebrew/bin/codex",
        },
        "my-cli": {
          command: "my-cli",
          args: ["--json"],
          output: "json",
          input: "arg",
          modelArg: "--model",
          modelAliases: {
            "claude-opus-4-6": "opus",
            "claude-sonnet-4-6": "sonnet",
          },
          sessionArg: "--session",
          sessionMode: "existing",
          sessionIdFields: ["session_id", "conversation_id"],
          systemPromptArg: "--system",
          // CodexスタイルのCLIは代わりにプロンプトファイルを指定できます:
          // systemPromptFileConfigArg: "-c",
          // systemPromptFileConfigKey: "model_instructions_file",
          systemPromptWhen: "first",
          imageArg: "--image",
          imageMode: "repeat",
          serialize: true,
        },
      },
    },
  },
}
```

## 仕組み

1. providerプレフィックス（`codex-cli/...`）に基づいて**バックエンドを選択**します。
2. 同じOpenClawプロンプトとワークスペースコンテキストを使って**システムプロンプトを構築**します。
3. 履歴の一貫性を保つため、対応している場合はセッションID付きで**CLIを実行**します。
4. **出力を解析**し（JSONまたはプレーンテキスト）、最終的なテキストを返します。
5. バックエンドごとに**セッションIDを永続化**し、後続のやり取りで同じCLIセッションを再利用します。

<Note>
バンドルされたAnthropicの`claude-cli`バックエンドが再びサポートされました。Anthropicのスタッフから、OpenClawスタイルのClaude CLI使用は再び許可されていると案内されたため、Anthropicが新しいポリシーを公開しない限り、OpenClawはこの統合における`claude -p`の使用を認可済みとして扱います。
</Note>

バンドルされたOpenAIの`codex-cli`バックエンドは、Codexの`model_instructions_file`設定オーバーライド（`-c model_instructions_file="..."`）を通じてOpenClawのシステムプロンプトを渡します。CodexはClaudeスタイルの`--append-system-prompt`フラグを公開していないため、OpenClawは新しいCodex CLIセッションごとに組み立てたプロンプトを一時ファイルに書き込みます。

バンドルされたAnthropicの`claude-cli`バックエンドは、OpenClawのSkillsスナップショットを2つの方法で受け取ります。1つは追加されたシステムプロンプト内のコンパクトなOpenClaw Skillsカタログ、もう1つは`--plugin-dir`で渡される一時的なClaude Codeプラグインです。このプラグインには、そのエージェント/セッションに対して適格なSkillsのみが含まれるため、Claude Codeネイティブのスキルリゾルバーは、OpenClawが通常プロンプトで提示するのと同じフィルタ済みセットを見ることになります。Skillのenv/APIキー上書きは、実行時に子プロセス環境へOpenClawから引き続き適用されます。

## セッション

- CLIがセッションをサポートしている場合は、`sessionArg`（例: `--session-id`）または、IDを複数のフラグに挿入する必要があるときは`sessionArgs`（プレースホルダー`{sessionId}`）を設定してください。
- CLIが異なるフラグを使う**resumeサブコマンド**を使用する場合は、`resumeArgs`（再開時に`args`を置き換える）と、必要に応じて`resumeOutput`（JSON以外の再開向け）を設定してください。
- `sessionMode`:
  - `always`: 常にセッションIDを送信します（保存済みがなければ新しいUUID）。
  - `existing`: 以前に保存されていた場合のみセッションIDを送信します。
  - `none`: セッションIDを送信しません。

シリアライズに関する注意:

- `serialize: true` は同じレーンの実行順を維持します。
- ほとんどのCLIは1つのproviderレーン上でシリアライズされます。
- OpenClawは、再ログイン、トークンローテーション、または認証プロファイル資格情報の変更を含め、バックエンドの認証状態が変わると、保存済みCLIセッションの再利用を破棄します。

## 画像（パススルー）

CLIが画像パスを受け付ける場合は、`imageArg`を設定します。

```json5
imageArg: "--image",
imageMode: "repeat"
```

OpenClawはbase64画像を一時ファイルに書き込みます。`imageArg`が設定されている場合、それらのパスはCLI引数として渡されます。`imageArg`がない場合、OpenClawはファイルパスをプロンプトに追記します（パス注入）。これは、プレーンなパスからローカルファイルを自動読み込みするCLIには十分です。

## 入力 / 出力

- `output: "json"`（デフォルト）はJSONを解析し、テキストとセッションIDの抽出を試みます。
- Gemini CLIのJSON出力では、`usage`がない、または空の場合、OpenClawは`response`から返信テキストを、`stats`から使用量を読み取ります。
- `output: "jsonl"` はJSONLストリーム（例: Codex CLI `--json`）を解析し、存在する場合は最終的なエージェントメッセージとセッション識別子を抽出します。
- `output: "text"` はstdoutを最終応答として扱います。

入力モード:

- `input: "arg"`（デフォルト）は、プロンプトを最後のCLI引数として渡します。
- `input: "stdin"` は、プロンプトをstdin経由で送信します。
- プロンプトが非常に長く、`maxPromptArgChars`が設定されている場合は、stdinが使用されます。

## デフォルト（プラグイン所有）

バンドルされたOpenAIプラグインは、`codex-cli`用のデフォルトも登録します。

- `command: "codex"`
- `args: ["exec","--json","--color","never","--sandbox","workspace-write","--skip-git-repo-check"]`
- `resumeArgs: ["exec","resume","{sessionId}","--color","never","--sandbox","workspace-write","--skip-git-repo-check"]`
- `output: "jsonl"`
- `resumeOutput: "text"`
- `modelArg: "--model"`
- `imageArg: "--image"`
- `sessionMode: "existing"`

バンドルされたGoogleプラグインも、`google-gemini-cli`用のデフォルトを登録します。

- `command: "gemini"`
- `args: ["--output-format", "json", "--prompt", "{prompt}"]`
- `resumeArgs: ["--resume", "{sessionId}", "--output-format", "json", "--prompt", "{prompt}"]`
- `imageArg: "@"`
- `imagePathScope: "workspace"`
- `modelArg: "--model"`
- `sessionMode: "existing"`
- `sessionIdFields: ["session_id", "sessionId"]`

前提条件: ローカルのGemini CLIがインストールされており、`PATH`上で`gemini`として利用できる必要があります（`brew install gemini-cli` または `npm install -g @google/gemini-cli`）。

Gemini CLI JSONに関する注意:

- 返信テキストはJSONの`response`フィールドから読み取られます。
- `usage`が存在しない、または空の場合、使用量は`stats`にフォールバックします。
- `stats.cached`はOpenClawの`cacheRead`へ正規化されます。
- `stats.input`がない場合、OpenClawは`stats.input_tokens - stats.cached`から入力トークンを導出します。

必要な場合のみ上書きしてください（一般的なのは絶対`command`パスです）。

## プラグイン所有のデフォルト

CLIバックエンドのデフォルトは、現在ではプラグインサーフェスの一部です。

- プラグインは`api.registerCliBackend(...)`でそれらを登録します。
- バックエンドの`id`は、モデル参照内のproviderプレフィックスになります。
- `agents.defaults.cliBackends.<id>`内のユーザー設定は、引き続きプラグインのデフォルトを上書きします。
- バックエンド固有の設定クリーンアップは、オプションの`normalizeConfig`フックを通じて引き続きプラグイン所有です。

小さなプロンプト/メッセージ互換シムが必要なプラグインは、プロバイダーやCLIバックエンドを置き換えずに、双方向のテキスト変換を宣言できます。

```typescript
api.registerTextTransforms({
  input: [
    { from: /red basket/g, to: "blue basket" },
    { from: /paper ticket/g, to: "digital ticket" },
    { from: /left shelf/g, to: "right shelf" },
  ],
  output: [
    { from: /blue basket/g, to: "red basket" },
    { from: /digital ticket/g, to: "paper ticket" },
    { from: /right shelf/g, to: "left shelf" },
  ],
});
```

`input`は、CLIに渡されるシステムプロンプトとユーザープロンプトを書き換えます。`output`は、ストリーミングされたassistantデルタと、解析済みの最終テキストを、OpenClaw自身のコントロールマーカー処理とチャネル配信の前に書き換えます。

Claude Codeのstream-json互換JSONLを出力するCLIでは、そのバックエンドの設定に`jsonlDialect: "claude-stream-json"`を設定してください。

## bundle MCPオーバーレイ

CLIバックエンドは**OpenClawツール呼び出しを直接受け取りません**が、バックエンドは`bundleMcp: true`で生成されたMCP設定オーバーレイにオプトインできます。

現在のバンドル動作:

- `claude-cli`: 生成されたstrict MCP設定ファイル
- `codex-cli`: `mcp_servers`用のインライン設定オーバーライド
- `google-gemini-cli`: 生成されたGeminiシステム設定ファイル

bundle MCPが有効な場合、OpenClawは次を行います。

- GatewayツールをCLIプロセスに公開するloopback HTTP MCPサーバーを起動する
- セッションごとのトークン（`OPENCLAW_MCP_TOKEN`）でブリッジを認証する
- ツールアクセスを現在のセッション、アカウント、チャネルコンテキストにスコープする
- 現在のワークスペースで有効なbundle-MCPサーバーを読み込む
- それらを既存のバックエンドMCP設定/設定形状とマージする
- 起動設定を、所有拡張のバックエンド所有統合モードを使って書き換える

MCPサーバーが1つも有効でない場合でも、バックエンドがbundle MCPにオプトインしていれば、バックグラウンド実行を分離したままにするため、OpenClawはstrict設定を引き続き注入します。

## 制限事項

- **直接のOpenClawツール呼び出しはありません。** OpenClawはCLIバックエンドプロトコルにツール呼び出しを注入しません。バックエンドが`bundleMcp: true`にオプトインした場合のみ、Gatewayツールを見ることができます。
- **ストリーミングはバックエンド依存です。** JSONLをストリーミングするバックエンドもあれば、終了までバッファするバックエンドもあります。
- **構造化出力**はCLIのJSON形式に依存します。
- **Codex CLIセッション**はテキスト出力経由で再開されます（JSONLではありません）。そのため、初回の`--json`実行より構造化が弱くなります。OpenClawセッション自体は通常どおり機能します。

## トラブルシューティング

- **CLIが見つからない**: `command`をフルパスに設定してください。
- **モデル名が間違っている**: `modelAliases`を使って`provider/model` → CLIモデルにマッピングしてください。
- **セッションの継続性がない**: `sessionArg`が設定され、`sessionMode`が`none`でないことを確認してください（Codex CLIは現在JSON出力で再開できません）。
- **画像が無視される**: `imageArg`を設定し（CLIがファイルパスをサポートしていることも確認してください）。
