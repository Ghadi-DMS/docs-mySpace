---
read_when:
    - 埋め込みエージェントランタイムまたはハーネスレジストリを変更しています
    - バンドル版または信頼済みプラグインからエージェントハーネスを登録しています
    - Codexプラグインがモデルプロバイダーとどのように関係するかを理解する必要があります
sidebarTitle: Agent Harness
summary: 低レベルの埋め込みエージェント実行子を置き換えるプラグイン向けの実験的SDKサーフェス
title: エージェントハーネスプラグイン
x-i18n:
    generated_at: "2026-04-11T02:46:40Z"
    model: gpt-5.4
    provider: openai
    source_hash: 43c1f2c087230398b0162ed98449f239c8db1e822e51c7dcd40c54fa6c3374e1
    source_path: plugins/sdk-agent-harness.md
    workflow: 15
---

# エージェントハーネスプラグイン

**エージェントハーネス**は、準備済みのOpenClawエージェントターン1回分に対する低レベル実行子です。これはモデルプロバイダーでも、チャネルでも、ツールレジストリでもありません。

このサーフェスは、バンドル版または信頼済みのネイティブプラグインでのみ使用してください。パラメーター型が意図的に現在の埋め込みランナーを反映しているため、この契約はまだ実験的です。

## ハーネスを使うべき場合

モデルファミリーが独自のネイティブセッションランタイムを持ち、通常のOpenClawプロバイダートランスポートでは抽象化として不適切な場合に、エージェントハーネスを登録します。

例:

- スレッドとコンパクションを管理するネイティブのコーディングエージェントサーバー
- ネイティブのプラン/reasoning/ツールイベントをストリーミングしなければならないローカルCLIまたはデーモン
- OpenClawセッションの文字起こしに加えて独自の再開IDを必要とするモデルランタイム

新しいLLM APIを追加するだけの目的でハーネスを登録してはいけません。通常のHTTPまたはWebSocketモデルAPIであれば、[provider plugin](/ja-JP/plugins/sdk-provider-plugins)を構築してください。

## コアが引き続き所有するもの

ハーネスが選択される前に、OpenClawはすでに以下を解決しています:

- プロバイダーとモデル
- ランタイム認証状態
- thinkingレベルとコンテキスト予算
- OpenClawの文字起こし/セッションファイル
- ワークスペース、サンドボックス、ツールポリシー
- チャネル返信コールバックとストリーミングコールバック
- モデルフォールバックとライブモデル切り替えポリシー

この分割は意図的なものです。ハーネスは準備済みの試行を実行しますが、プロバイダーを選択したり、チャネル配信を置き換えたり、黙ってモデルを切り替えたりはしません。

## ハーネスを登録する

**インポート:** `openclaw/plugin-sdk/agent-harness`

```typescript
import type { AgentHarness } from "openclaw/plugin-sdk/agent-harness";
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

const myHarness: AgentHarness = {
  id: "my-harness",
  label: "My native agent harness",

  supports(ctx) {
    return ctx.provider === "my-provider"
      ? { supported: true, priority: 100 }
      : { supported: false };
  },

  async runAttempt(params) {
    // ネイティブスレッドを開始または再開します。
    // params.prompt、params.tools、params.images、params.onPartialReply、
    // params.onAgentEvent、およびその他の準備済み試行フィールドを使用します。
    return await runMyNativeTurn(params);
  },
};

export default definePluginEntry({
  id: "my-native-agent",
  name: "My Native Agent",
  description: "Runs selected models through a native agent daemon.",
  register(api) {
    api.registerAgentHarness(myHarness);
  },
});
```

## 選択ポリシー

OpenClawは、プロバイダー/モデル解決後にハーネスを選択します:

1. `OPENCLAW_AGENT_RUNTIME=<id>`は、そのIDを持つ登録済みハーネスを強制します。
2. `OPENCLAW_AGENT_RUNTIME=pi`は、組み込みPIハーネスを強制します。
3. `OPENCLAW_AGENT_RUNTIME=auto`は、登録済みハーネスに、解決済みのプロバイダー/モデルをサポートしているか問い合わせます。
4. 一致する登録済みハーネスがない場合、PIフォールバックが無効でなければOpenClawはPIを使用します。

強制されたプラグインハーネスの失敗は、実行失敗として表面化します。`auto`モードでは、
選択されたプラグインハーネスがターンの副作用を生成する前に失敗した場合、
OpenClawはPIにフォールバックすることがあります。代わりにそのフォールバックを確定的な失敗にしたい場合は、`OPENCLAW_AGENT_HARNESS_FALLBACK=none`または
`embeddedHarness.fallback: "none"`を設定してください。

バンドル版Codexプラグインは、ハーネスIDとして`codex`を登録します。コアはこれを
通常のプラグインハーネスIDとして扱います。Codex固有のエイリアスは共有ランタイムセレクターではなく、
プラグインまたはオペレーター設定に属します。

## プロバイダーとハーネスの組み合わせ

ほとんどのハーネスは、プロバイダーもあわせて登録するべきです。プロバイダーは、
モデル参照、認証状態、モデルメタデータ、および`/model`選択をOpenClawの他の部分から見えるようにします。
その後、ハーネスは`supports(...)`内でそのプロバイダーを要求します。

バンドル版Codexプラグインはこのパターンに従っています:

- プロバイダーID: `codex`
- ユーザーモデル参照: `codex/gpt-5.4`、`codex/gpt-5.2`、またはCodexアプリサーバーが返すその他のモデル
- ハーネスID: `codex`
- 認証: 合成プロバイダー可用性。CodexハーネスがネイティブのCodexログイン/セッションを所有するため
- アプリサーバーリクエスト: OpenClawは生のモデルIDをCodexに送信し、
  ハーネスがネイティブのアプリサーバープロトコルと通信します

Codexプラグインは追加的なものです。通常の`openai/gpt-*`参照は引き続きOpenAIプロバイダー参照であり、
通常のOpenClawプロバイダー経路を使用し続けます。Codex管理の認証、
Codexモデル検出、ネイティブスレッド、およびCodexアプリサーバー実行が必要な場合は`codex/gpt-*`
を選択してください。`/model`は、OpenAIプロバイダー資格情報を必要とせずに、
Codexアプリサーバーが返すCodexモデル間を切り替えられます。

オペレーター設定、モデルプレフィックス例、Codex専用設定については、
[Codex Harness](/ja-JP/plugins/codex-harness)を参照してください。

OpenClawはCodexアプリサーバー`0.118.0`以降を必要とします。Codexプラグインは
アプリサーバーの初期化ハンドシェイクを確認し、古いまたはバージョン未設定のサーバーをブロックすることで、
OpenClawがテスト済みのプロトコルサーフェスに対してのみ実行されるようにします。

## PIフォールバックを無効にする

デフォルトでは、OpenClawは埋め込みエージェントを`agents.defaults.embeddedHarness`
が`{ runtime: "auto", fallback: "pi" }`に設定された状態で実行します。`auto`モードでは、登録済みプラグイン
ハーネスがプロバイダー/モデルの組み合わせを要求できます。一致するものがない場合、または自動選択された
プラグインハーネスが出力生成前に失敗した場合、OpenClawはPIにフォールバックします。

プラグインハーネスだけが実際に使用されていることを証明する必要がある場合は、`fallback: "none"`を設定してください。これにより自動PIフォールバックは無効になりますが、
明示的な`runtime: "pi"`や`OPENCLAW_AGENT_RUNTIME=pi`はブロックされません。

Codex専用の埋め込み実行の場合:

```json
{
  "agents": {
    "defaults": {
      "model": "codex/gpt-5.4",
      "embeddedHarness": {
        "runtime": "codex",
        "fallback": "none"
      }
    }
  }
}
```

一致するモデルを任意の登録済みプラグインハーネスに要求させつつ、OpenClawが黙ってPIにフォールバックすることは避けたい場合は、
`runtime: "auto"`のままにしてフォールバックを無効にしてください:

```json
{
  "agents": {
    "defaults": {
      "embeddedHarness": {
        "runtime": "auto",
        "fallback": "none"
      }
    }
  }
}
```

エージェント単位の上書きも同じ形を使います:

```json
{
  "agents": {
    "defaults": {
      "embeddedHarness": {
        "runtime": "auto",
        "fallback": "pi"
      }
    },
    "list": [
      {
        "id": "codex-only",
        "model": "codex/gpt-5.4",
        "embeddedHarness": {
          "runtime": "codex",
          "fallback": "none"
        }
      }
    ]
  }
}
```

`OPENCLAW_AGENT_RUNTIME`は引き続き設定済みランタイムを上書きします。環境から
PIフォールバックを無効にするには`OPENCLAW_AGENT_HARNESS_FALLBACK=none`を使用してください。

```bash
OPENCLAW_AGENT_RUNTIME=codex \
OPENCLAW_AGENT_HARNESS_FALLBACK=none \
openclaw gateway run
```

フォールバックを無効にすると、要求されたハーネスが
登録されていない、解決済みのプロバイダー/モデルをサポートしていない、または
ターンの副作用を生成する前に失敗した場合、セッションは早い段階で失敗します。これは
Codex専用デプロイや、Codexアプリサーバー経路が実際に使われていることを証明しなければならない
ライブテストでは意図された動作です。

この設定が制御するのは埋め込みエージェントハーネスだけです。画像、
動画、音楽、TTS、PDF、その他のプロバイダー固有のモデルルーティングは無効化されません。

## ネイティブセッションと文字起こしミラー

ハーネスは、ネイティブセッションID、スレッドID、またはデーモン側の再開トークンを保持する場合があります。
その対応付けはOpenClawセッションに明示的に関連付けたままにし、
ユーザーに見えるassistant/tool出力をOpenClawの文字起こしへミラーし続けてください。

OpenClawの文字起こしは、以下の互換レイヤーのままです:

- チャネルから見えるセッション履歴
- 文字起こし検索とインデックス作成
- 後続ターンで組み込みPIハーネスに戻すこと
- 汎用の`/new`、`/reset`、およびセッション削除動作

ハーネスがサイドカーの対応情報を保存する場合は、所有するOpenClawセッションがリセットされたときに
OpenClawがそれをクリアできるよう、`reset(...)`を実装してください。

## ツールおよびメディア結果

コアはOpenClawのツール一覧を構築し、それを準備済み試行に渡します。
ハーネスが動的ツール呼び出しを実行する場合は、チャネルメディアを自分で送信するのではなく、
ハーネス結果の形を通じてツール結果を返してください。

これにより、テキスト、画像、動画、音楽、TTS、承認、メッセージングツール出力が、
PIベース実行と同じ配信経路に保たれます。

## 現在の制限

- 公開インポートパスは汎用ですが、一部の試行/結果型エイリアスには互換性のためにまだ
  `Pi`名が残っています。
- サードパーティ製ハーネスのインストールは実験的です。ネイティブセッションランタイムが必要になるまでは
  provider pluginを優先してください。
- ターンをまたいだハーネス切り替えはサポートされています。ネイティブツール、承認、assistantテキスト、またはメッセージ送信が始まった後、
  ターンの途中でハーネスを切り替えないでください。

## 関連

- [SDK Overview](/ja-JP/plugins/sdk-overview)
- [Runtime Helpers](/ja-JP/plugins/sdk-runtime)
- [Provider Plugins](/ja-JP/plugins/sdk-provider-plugins)
- [Codex Harness](/ja-JP/plugins/codex-harness)
- [Model Providers](/ja-JP/concepts/model-providers)
