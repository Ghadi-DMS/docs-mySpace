---
read_when:
    - プラグインからコアヘルパー（TTS、STT、画像生成、Web検索、サブエージェント）を呼び出す必要があります
    - '`api.runtime`が何を公開しているかを理解したいです'
    - プラグインコードから設定、エージェント、またはメディアヘルパーにアクセスしています
sidebarTitle: Runtime Helpers
summary: api.runtime -- プラグインで利用できる注入済みランタイムヘルパー
title: プラグインランタイムヘルパー
x-i18n:
    generated_at: "2026-04-11T02:47:20Z"
    model: gpt-5.4
    provider: openai
    source_hash: fbf8a6ecd970300f784b8aca20eed40ba12c83107abd27385bfdc3347d2544be
    source_path: plugins/sdk-runtime.md
    workflow: 15
---

# プラグインランタイムヘルパー

登録時にすべてのプラグインへ注入される`api.runtime`オブジェクトのリファレンスです。ホスト内部を直接インポートする代わりに、これらのヘルパーを使用してください。

<Tip>
  **ウォークスルーを探していますか？** [Channel Plugins](/ja-JP/plugins/sdk-channel-plugins)
  または[Provider Plugins](/ja-JP/plugins/sdk-provider-plugins)で、これらのヘルパーが実際の流れの中でどう使われるかを段階的に確認できます。
</Tip>

```typescript
register(api) {
  const runtime = api.runtime;
}
```

## ランタイム名前空間

### `api.runtime.agent`

エージェントID、ディレクトリ、セッション管理。

```typescript
// エージェントの作業ディレクトリを解決
const agentDir = api.runtime.agent.resolveAgentDir(cfg);

// エージェントワークスペースを解決
const workspaceDir = api.runtime.agent.resolveAgentWorkspaceDir(cfg);

// エージェントIDを取得
const identity = api.runtime.agent.resolveAgentIdentity(cfg);

// デフォルトthinkingレベルを取得
const thinking = api.runtime.agent.resolveThinkingDefault(cfg, provider, model);

// エージェントタイムアウトを取得
const timeoutMs = api.runtime.agent.resolveAgentTimeoutMs(cfg);

// ワークスペースの存在を保証
await api.runtime.agent.ensureAgentWorkspace(cfg);

// 埋め込みエージェントターンを実行
const agentDir = api.runtime.agent.resolveAgentDir(cfg);
const result = await api.runtime.agent.runEmbeddedAgent({
  sessionId: "my-plugin:task-1",
  runId: crypto.randomUUID(),
  sessionFile: path.join(agentDir, "sessions", "my-plugin-task-1.jsonl"),
  workspaceDir: api.runtime.agent.resolveAgentWorkspaceDir(cfg),
  prompt: "Summarize the latest changes",
  timeoutMs: api.runtime.agent.resolveAgentTimeoutMs(cfg),
});
```

`runEmbeddedAgent(...)`は、プラグインコードから通常のOpenClaw
エージェントターンを開始するための中立的なヘルパーです。これは、チャネル起点の返信と同じプロバイダー/モデル解決および
エージェントハーネス選択を使用します。

`runEmbeddedPiAgent(...)`は、互換性エイリアスとして引き続き利用できます。

**セッションストアヘルパー**は`api.runtime.agent.session`配下にあります:

```typescript
const storePath = api.runtime.agent.session.resolveStorePath(cfg);
const store = api.runtime.agent.session.loadSessionStore(cfg);
await api.runtime.agent.session.saveSessionStore(cfg, store);
const filePath = api.runtime.agent.session.resolveSessionFilePath(cfg, sessionId);
```

### `api.runtime.agent.defaults`

デフォルトのモデル定数とプロバイダー定数:

```typescript
const model = api.runtime.agent.defaults.model; // 例: "anthropic/claude-sonnet-4-6"
const provider = api.runtime.agent.defaults.provider; // 例: "anthropic"
```

### `api.runtime.subagent`

バックグラウンドサブエージェント実行の開始と管理。

```typescript
// サブエージェント実行を開始
const { runId } = await api.runtime.subagent.run({
  sessionKey: "agent:main:subagent:search-helper",
  message: "Expand this query into focused follow-up searches.",
  provider: "openai", // 任意の上書き
  model: "gpt-4.1-mini", // 任意の上書き
  deliver: false,
});

// 完了を待機
const result = await api.runtime.subagent.waitForRun({ runId, timeoutMs: 30000 });

// セッションメッセージを読み取り
const { messages } = await api.runtime.subagent.getSessionMessages({
  sessionKey: "agent:main:subagent:search-helper",
  limit: 10,
});

// セッションを削除
await api.runtime.subagent.deleteSession({
  sessionKey: "agent:main:subagent:search-helper",
});
```

<Warning>
  モデル上書き（`provider`/`model`）には、設定内の
  `plugins.entries.<id>.subagent.allowModelOverride: true`によるオペレーターの明示的オプトインが必要です。
  信頼されていないプラグインでもサブエージェントは実行できますが、上書き要求は拒否されます。
</Warning>

### `api.runtime.taskFlow`

Task Flowランタイムを既存のOpenClawセッションキーまたは信頼済みツール
コンテキストにバインドし、毎回ownerを渡さずにTask Flowを作成・管理します。

```typescript
const taskFlow = api.runtime.taskFlow.fromToolContext(ctx);

const created = taskFlow.createManaged({
  controllerId: "my-plugin/review-batch",
  goal: "Review new pull requests",
});

const child = taskFlow.runTask({
  flowId: created.flowId,
  runtime: "acp",
  childSessionKey: "agent:main:subagent:reviewer",
  task: "Review PR #123",
  status: "running",
  startedAt: Date.now(),
});

const waiting = taskFlow.setWaiting({
  flowId: created.flowId,
  expectedRevision: created.revision,
  currentStep: "await-human-reply",
  waitJson: { kind: "reply", channel: "telegram" },
});
```

独自のバインディングレイヤーから信頼済みのOpenClawセッションキーをすでに持っている場合は、
`bindSession({ sessionKey, requesterOrigin })`を使用してください。生のユーザー入力からバインドしてはいけません。

### `api.runtime.tts`

テキスト読み上げ。

```typescript
// 標準TTS
const clip = await api.runtime.tts.textToSpeech({
  text: "Hello from OpenClaw",
  cfg: api.config,
});

// 電話向け最適化TTS
const telephonyClip = await api.runtime.tts.textToSpeechTelephony({
  text: "Hello from OpenClaw",
  cfg: api.config,
});

// 利用可能な音声を一覧表示
const voices = await api.runtime.tts.listVoices({
  provider: "elevenlabs",
  cfg: api.config,
});
```

コアの`messages.tts`設定とプロバイダー選択を使用します。PCMオーディオ
バッファ + サンプルレートを返します。

### `api.runtime.mediaUnderstanding`

画像、音声、動画の解析。

```typescript
// 画像を説明
const image = await api.runtime.mediaUnderstanding.describeImageFile({
  filePath: "/tmp/inbound-photo.jpg",
  cfg: api.config,
  agentDir: "/tmp/agent",
});

// 音声を文字起こし
const { text } = await api.runtime.mediaUnderstanding.transcribeAudioFile({
  filePath: "/tmp/inbound-audio.ogg",
  cfg: api.config,
  mime: "audio/ogg", // 任意。MIMEを推定できない場合に使用
});

// 動画を説明
const video = await api.runtime.mediaUnderstanding.describeVideoFile({
  filePath: "/tmp/inbound-video.mp4",
  cfg: api.config,
});

// 汎用ファイル解析
const result = await api.runtime.mediaUnderstanding.runFile({
  filePath: "/tmp/inbound-file.pdf",
  cfg: api.config,
});
```

出力が生成されない場合（例: 入力がスキップされた場合）は`{ text: undefined }`を返します。

<Info>
  `api.runtime.stt.transcribeAudioFile(...)`は、
  `api.runtime.mediaUnderstanding.transcribeAudioFile(...)`の互換性エイリアスとして引き続き利用できます。
</Info>

### `api.runtime.imageGeneration`

画像生成。

```typescript
const result = await api.runtime.imageGeneration.generate({
  prompt: "A robot painting a sunset",
  cfg: api.config,
});

const providers = api.runtime.imageGeneration.listProviders({ cfg: api.config });
```

### `api.runtime.webSearch`

Web検索。

```typescript
const providers = api.runtime.webSearch.listProviders({ config: api.config });

const result = await api.runtime.webSearch.search({
  config: api.config,
  args: { query: "OpenClaw plugin SDK", count: 5 },
});
```

### `api.runtime.media`

低レベルメディアユーティリティ。

```typescript
const webMedia = await api.runtime.media.loadWebMedia(url);
const mime = await api.runtime.media.detectMime(buffer);
const kind = api.runtime.media.mediaKindFromMime("image/jpeg"); // "image"
const isVoice = api.runtime.media.isVoiceCompatibleAudio(filePath);
const metadata = await api.runtime.media.getImageMetadata(filePath);
const resized = await api.runtime.media.resizeToJpeg(buffer, { maxWidth: 800 });
```

### `api.runtime.config`

設定の読み込みと書き込み。

```typescript
const cfg = await api.runtime.config.loadConfig();
await api.runtime.config.writeConfigFile(cfg);
```

### `api.runtime.system`

システムレベルユーティリティ。

```typescript
await api.runtime.system.enqueueSystemEvent(event);
api.runtime.system.requestHeartbeatNow();
const output = await api.runtime.system.runCommandWithTimeout(cmd, args, opts);
const hint = api.runtime.system.formatNativeDependencyHint(pkg);
```

### `api.runtime.events`

イベント購読。

```typescript
api.runtime.events.onAgentEvent((event) => {
  /* ... */
});
api.runtime.events.onSessionTranscriptUpdate((update) => {
  /* ... */
});
```

### `api.runtime.logging`

ロギング。

```typescript
const verbose = api.runtime.logging.shouldLogVerbose();
const childLogger = api.runtime.logging.getChildLogger({ plugin: "my-plugin" }, { level: "debug" });
```

### `api.runtime.modelAuth`

モデルおよびプロバイダー認証の解決。

```typescript
const auth = await api.runtime.modelAuth.getApiKeyForModel({ model, cfg });
const providerAuth = await api.runtime.modelAuth.resolveApiKeyForProvider({
  provider: "openai",
  cfg,
});
```

### `api.runtime.state`

状態ディレクトリ解決。

```typescript
const stateDir = api.runtime.state.resolveStateDir();
```

### `api.runtime.tools`

メモリツールファクトリーとCLI。

```typescript
const getTool = api.runtime.tools.createMemoryGetTool(/* ... */);
const searchTool = api.runtime.tools.createMemorySearchTool(/* ... */);
api.runtime.tools.registerMemoryCli(/* ... */);
```

### `api.runtime.channel`

チャネル固有のランタイムヘルパー（チャネルプラグインが読み込まれている場合に利用可能）。

`api.runtime.channel.mentions`は、ランタイム注入を使用する
バンドル版チャネルプラグイン向けの共有受信メンションポリシーサーフェスです:

```typescript
const mentionMatch = api.runtime.channel.mentions.matchesMentionWithExplicit(text, {
  mentionRegexes,
  mentionPatterns,
});

const decision = api.runtime.channel.mentions.resolveInboundMentionDecision({
  facts: {
    canDetectMention: true,
    wasMentioned: mentionMatch.matched,
    implicitMentionKinds: api.runtime.channel.mentions.implicitMentionKindWhen(
      "reply_to_bot",
      isReplyToBot,
    ),
  },
  policy: {
    isGroup,
    requireMention,
    allowTextCommands,
    hasControlCommand,
    commandAuthorized,
  },
});
```

利用可能なメンションヘルパー:

- `buildMentionRegexes`
- `matchesMentionPatterns`
- `matchesMentionWithExplicit`
- `implicitMentionKindWhen`
- `resolveInboundMentionDecision`

`api.runtime.channel.mentions`は、古い
`resolveMentionGating*`互換ヘルパーを意図的に公開していません。正規化された
`{ facts, policy }`経路を優先してください。

## ランタイム参照の保存

`register`コールバックの外で使用するためにランタイム参照を保存するには、
`createPluginRuntimeStore`を使用してください:

```typescript
import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
import type { PluginRuntime } from "openclaw/plugin-sdk/runtime-store";

const store = createPluginRuntimeStore<PluginRuntime>("my-plugin runtime not initialized");

// エントリーポイント内
export default defineChannelPluginEntry({
  id: "my-plugin",
  name: "My Plugin",
  description: "Example",
  plugin: myPlugin,
  setRuntime: store.setRuntime,
});

// 他のファイル内
export function getRuntime() {
  return store.getRuntime(); // 初期化されていない場合はthrow
}

export function tryGetRuntime() {
  return store.tryGetRuntime(); // 初期化されていない場合はnullを返す
}
```

## その他のトップレベル`api`フィールド

`api.runtime`に加えて、APIオブジェクトは以下も提供します:

| フィールド               | 型                        | 説明                                                                                     |
| ------------------------ | ------------------------- | ---------------------------------------------------------------------------------------- |
| `api.id`                 | `string`                  | プラグインID                                                                             |
| `api.name`               | `string`                  | プラグイン表示名                                                                         |
| `api.config`             | `OpenClawConfig`          | 現在の設定スナップショット（利用可能な場合は、アクティブなインメモリ実行時スナップショット） |
| `api.pluginConfig`       | `Record<string, unknown>` | `plugins.entries.<id>.config`から取得したプラグイン固有設定                             |
| `api.logger`             | `PluginLogger`            | スコープ付きロガー（`debug`、`info`、`warn`、`error`）                                  |
| `api.registrationMode`   | `PluginRegistrationMode`  | 現在の読み込みモード。`"setup-runtime"`は、完全なエントリー起動/設定前の軽量ウィンドウです |
| `api.resolvePath(input)` | `(string) => string`      | プラグインルートからの相対パスを解決します                                               |

## 関連

- [SDK Overview](/ja-JP/plugins/sdk-overview) -- サブパスリファレンス
- [SDK Entry Points](/ja-JP/plugins/sdk-entrypoints) -- `definePluginEntry`オプション
- [Plugin Internals](/ja-JP/plugins/architecture) -- capability modelとレジストリ
