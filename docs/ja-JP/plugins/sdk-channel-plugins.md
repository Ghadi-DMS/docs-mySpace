---
read_when:
    - 新しいメッセージングチャネル Plugin を構築しています
    - OpenClaw をメッセージングプラットフォームに接続したいと考えています
    - ChannelPlugin アダプターの公開インターフェースを理解する必要があります
sidebarTitle: Channel Plugins
summary: OpenClaw 向けメッセージングチャネル Plugin を構築するためのステップバイステップガイド
title: チャネル Plugin の構築
x-i18n:
    generated_at: "2026-04-18T04:40:46Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3dda53c969bc7356a450c2a5bf49fb82bf1283c23e301dec832d8724b11e724b
    source_path: plugins/sdk-channel-plugins.md
    workflow: 15
---

# チャネル Plugin の構築

このガイドでは、OpenClaw をメッセージングプラットフォームに接続するチャネル Plugin の構築手順を説明します。最後には、DM セキュリティ、ペアリング、返信スレッド化、送信メッセージングを備えた動作するチャネルが完成します。

<Info>
  OpenClaw Plugin をまだ一度も作成したことがない場合は、まず基本的なパッケージ構造とマニフェスト設定について [はじめに](/ja-JP/plugins/building-plugins) を読んでください。
</Info>

## チャネル Plugin の仕組み

チャネル Plugin には、独自の send/edit/react ツールは不要です。OpenClaw は、コアに1つの共有 `message` ツールを保持します。あなたの Plugin が担うのは次の部分です。

- **設定** — アカウント解決とセットアップウィザード
- **セキュリティ** — DM ポリシーと許可リスト
- **ペアリング** — DM 承認フロー
- **セッショングラマー** — プロバイダー固有の会話 ID を、どのようにベースチャット、スレッド ID、親フォールバックへ対応付けるか
- **送信** — テキスト、メディア、投票をプラットフォームへ送信する処理
- **スレッド化** — 返信をどのようにスレッド化するか

コアは、共有 message ツール、プロンプト配線、外側のセッションキー形状、汎用的な `:thread:` の管理、およびディスパッチを担います。

チャネルでメディアソースを運ぶ message ツールのパラメーターを追加する場合は、それらのパラメーター名を `describeMessageTool(...).mediaSourceParams` を通じて公開してください。コアはその明示的な一覧を、サンドボックス内のパス正規化および送信時のメディアアクセスポリシーに使用するため、Plugin 側でプロバイダー固有の avatar、attachment、cover-image パラメーター向けの共有コア特例は不要です。
できれば、`{ "set-profile": ["avatarUrl", "avatarPath"] }` のようなアクションキー付きマップを返してください。そうすることで、無関係なアクションが別のアクションのメディア引数を引き継がずに済みます。意図的にすべての公開アクションで共有するパラメーターであれば、フラットな配列でも動作します。

プラットフォームが会話 ID 内に追加スコープを保存する場合は、その解析を `messaging.resolveSessionConversation(...)` とともに Plugin 内に保持してください。これは、`rawId` をベース会話 ID、任意のスレッド ID、明示的な `baseConversationId`、および `parentConversationCandidates` へ対応付けるための正規フックです。
`parentConversationCandidates` を返す場合は、最も狭い親から最も広い親/ベース会話の順に並べてください。

チャネルレジストリの起動前に同じ解析が必要な同梱 Plugin は、対応する `resolveSessionConversation(...)` エクスポートを持つトップレベルの `session-key-api.ts` ファイルも公開できます。コアは、ランタイム Plugin レジストリがまだ利用できないときに限り、このブートストラップ安全な公開インターフェースを使用します。

`messaging.resolveParentConversationCandidates(...)` は、Plugin が汎用/raw ID の上に親フォールバックだけを必要とする場合の、レガシー互換フォールバックとして引き続き利用できます。両方のフックが存在する場合、コアはまず `resolveSessionConversation(...).parentConversationCandidates` を使用し、その正規フックがそれらを省略したときのみ `resolveParentConversationCandidates(...)` にフォールバックします。

## 承認とチャネル機能

ほとんどのチャネル Plugin では、承認専用コードは不要です。

- コアは、同一チャット内の `/approve`、共有承認ボタンのペイロード、汎用フォールバック配信を担います。
- チャネルで承認固有の挙動が必要な場合は、チャネル Plugin 上に1つの `approvalCapability` オブジェクトを置くのが望ましいです。
- `ChannelPlugin.approvals` は削除されました。承認の配信/ネイティブ/レンダリング/認証に関する情報は `approvalCapability` に置いてください。
- `plugin.auth` は login/logout 専用です。コアはそのオブジェクトから承認認証フックをもう読みません。
- `approvalCapability.authorizeActorAction` と `approvalCapability.getActionAvailabilityState` が、承認認証の正規の公開インターフェースです。
- 同一チャット内の承認認証の可用性には `approvalCapability.getActionAvailabilityState` を使用してください。
- チャネルがネイティブ exec 承認を公開する場合は、開始サーフェス/ネイティブクライアントの状態が同一チャット承認認証と異なるときに `approvalCapability.getExecInitiatingSurfaceState` を使用してください。コアはこの exec 専用フックを使用して、`enabled` と `disabled` を区別し、開始チャネルがネイティブ exec 承認をサポートするかどうかを判断し、ネイティブクライアントのフォールバック案内にそのチャネルを含めます。`createApproverRestrictedNativeApprovalCapability(...)` は、この一般的なケースを埋めます。
- 重複したローカル承認プロンプトを隠す、配信前に typing indicator を送る、といったチャネル固有のペイロードライフサイクル挙動には、`outbound.shouldSuppressLocalPayloadPrompt` または `outbound.beforeDeliverPayload` を使用してください。
- `approvalCapability.delivery` は、ネイティブ承認のルーティングまたはフォールバック抑制にのみ使用してください。
- `approvalCapability.nativeRuntime` は、チャネル所有のネイティブ承認情報に使用してください。ホットなチャネルエントリーポイントでは `createLazyChannelApprovalNativeRuntimeAdapter(...)` を使って遅延化し、必要時にランタイムモジュールを import できるようにしつつ、コアが承認ライフサイクルを組み立てられるようにしてください。
- `approvalCapability.render` は、チャネルが共有レンダラーではなく本当にカスタム承認ペイロードを必要とする場合にのみ使用してください。
- チャネルが、ネイティブ exec 承認を有効にするために必要な正確な設定項目を disabled 経路の返信で説明したい場合は、`approvalCapability.describeExecApprovalSetup` を使用してください。このフックは `{ channel, channelLabel, accountId }` を受け取ります。名前付きアカウントチャネルは、トップレベルのデフォルトではなく、`channels.<channel>.accounts.<id>.execApprovals.*` のようなアカウントスコープのパスをレンダリングしてください。
- チャネルが既存設定から安定した owner 類似の DM ID を推測できる場合は、承認固有のコアロジックを追加せずに、同一チャット内の `/approve` を制限するために `openclaw/plugin-sdk/approval-runtime` の `createResolvedApproverActionAuthAdapter` を使用してください。
- チャネルがネイティブ承認配信を必要とする場合は、チャネルコードをターゲット正規化とトランスポート/表示情報に集中させてください。`openclaw/plugin-sdk/approval-runtime` の `createChannelExecApprovalProfile`、`createChannelNativeOriginTargetResolver`、`createChannelApproverDmTargetResolver`、`createApproverRestrictedNativeApprovalCapability` を使用してください。チャネル固有の情報は `approvalCapability.nativeRuntime` の背後に置き、理想的には `createChannelApprovalNativeRuntimeAdapter(...)` または `createLazyChannelApprovalNativeRuntimeAdapter(...)` を介してください。そうすることで、コアがハンドラーを組み立て、リクエストのフィルタリング、ルーティング、重複排除、有効期限、Gateway 購読、別経路ルーティング通知を担えます。`nativeRuntime` は、いくつかの小さな公開インターフェースに分かれています。
- `availability` — アカウントが設定済みかどうか、およびリクエストを処理すべきかどうか
- `presentation` — 共有承認ビュー・モデルを pending/resolved/expired のネイティブペイロードや最終アクションへ変換
- `transport` — ターゲットを準備し、ネイティブ承認メッセージを送信/更新/削除
- `interactions` — ネイティブボタンやリアクション向けの任意の bind/unbind/clear-action フック
- `observe` — 任意の配信診断フック
- チャネルが client、token、Bolt app、Webhook レシーバーのようなランタイム所有オブジェクトを必要とする場合は、`openclaw/plugin-sdk/channel-runtime-context` を通じて登録してください。汎用 runtime-context レジストリにより、コアは承認固有のラッパー glue を追加せずに、チャネル起動状態から capability 駆動ハンドラーをブートストラップできます。
- capability 駆動の公開インターフェースではまだ表現力が足りない場合にのみ、より低レベルの `createChannelApprovalHandler` または `createChannelNativeApprovalRuntime` を使ってください。
- ネイティブ承認チャネルでは、`accountId` と `approvalKind` の両方をそれらのヘルパー経由でルーティングする必要があります。`accountId` は複数アカウントの承認ポリシーを正しい bot アカウントにスコープし、`approvalKind` は exec と Plugin 承認の挙動を、コアにハードコードされた分岐なしでチャネル側で扱えるようにします。
- コアは現在、承認の再ルーティング通知も担います。チャネル Plugin は `createChannelNativeApprovalRuntime` から独自の「承認は DM / 別チャネルへ送られました」というフォローアップメッセージを送るべきではありません。代わりに、共有承認 capability ヘルパーを通じて正確な送信元 + approver-DM ルーティングを公開し、実際の配信を集約したうえで、必要な通知を開始チャットへ投稿するかどうかはコアに任せてください。
- 配信された承認 ID の種類は、最初から最後まで保持してください。ネイティブクライアントは、チャネルローカルの状態から exec と Plugin 承認のルーティングを推測したり書き換えたりしてはいけません。
- 異なる承認種別は、意図的に異なるネイティブサーフェスを公開してよい場合があります。
  現在の同梱例:
  - Slack は exec と Plugin の両方の ID について、ネイティブ承認ルーティングを利用可能なままにしています。
  - Matrix は exec と Plugin 承認の両方で同じネイティブ DM/チャネルルーティングと reaction UX を維持しつつ、承認種別ごとに auth だけは異なるようにできます。
- `createApproverRestrictedNativeApprovalAdapter` は互換ラッパーとしてまだ存在しますが、新しいコードでは capability builder を優先し、Plugin 上で `approvalCapability` を公開してください。

ホットなチャネルエントリーポイントでは、そのファミリーの一部だけが必要な場合、より狭いランタイム subpath を優先してください。

- `openclaw/plugin-sdk/approval-auth-runtime`
- `openclaw/plugin-sdk/approval-client-runtime`
- `openclaw/plugin-sdk/approval-delivery-runtime`
- `openclaw/plugin-sdk/approval-gateway-runtime`
- `openclaw/plugin-sdk/approval-handler-adapter-runtime`
- `openclaw/plugin-sdk/approval-handler-runtime`
- `openclaw/plugin-sdk/approval-native-runtime`
- `openclaw/plugin-sdk/approval-reply-runtime`
- `openclaw/plugin-sdk/channel-runtime-context`

同様に、より広い包括的な公開インターフェースが不要な場合は、`openclaw/plugin-sdk/setup-runtime`、`openclaw/plugin-sdk/setup-adapter-runtime`、`openclaw/plugin-sdk/reply-runtime`、`openclaw/plugin-sdk/reply-dispatch-runtime`、`openclaw/plugin-sdk/reply-reference`、`openclaw/plugin-sdk/reply-chunking` を優先してください。

セットアップについては特に以下の通りです。

- `openclaw/plugin-sdk/setup-runtime` は、ランタイム安全なセットアップヘルパーをカバーします:
  import-safe なセットアップ patch アダプター（`createPatchedAccountSetupAdapter`、`createEnvPatchedAccountSetupAdapter`、`createSetupInputPresenceValidator`）、lookup-note 出力、`promptResolvedAllowFrom`、`splitSetupEntries`、および委譲セットアッププロキシ builder
- `openclaw/plugin-sdk/setup-adapter-runtime` は、`createEnvPatchedAccountSetupAdapter` のための、より狭い env 対応アダプター公開インターフェースです
- `openclaw/plugin-sdk/channel-setup` は、任意インストールのセットアップ builder と、いくつかのセットアップ安全な基本要素をカバーします:
  `createOptionalChannelSetupSurface`、`createOptionalChannelSetupAdapter`、

チャネルが env 駆動のセットアップや auth をサポートし、汎用的な起動/設定フローがランタイム読み込み前にその env 名を知る必要がある場合は、Plugin マニフェストで `channelEnvVars` に宣言してください。チャネルランタイムの `envVars` やローカル定数は、運用者向け説明文だけに使ってください。
`createOptionalChannelSetupWizard`、`DEFAULT_ACCOUNT_ID`、`createTopLevelChannelDmPolicy`、`setSetupChannelEnabled`、および `splitSetupEntries`

- `moveSingleAccountChannelSectionToDefaultAccount(...)` のような、より重い共有セットアップ/設定ヘルパーも必要な場合にのみ、より広い `openclaw/plugin-sdk/setup` 公開インターフェースを使用してください

チャネルがセットアップ画面で「まずこの Plugin をインストールしてください」と案内したいだけであれば、`createOptionalChannelSetupSurface(...)` を優先してください。生成される adapter/wizard は、設定書き込みと最終確定に対して fail closed し、検証・finalize・docs リンク文面で同じインストール必須メッセージを再利用します。

そのほかのホットなチャネルパスでも、より広いレガシー公開インターフェースではなく、より狭いヘルパーを優先してください。

- `openclaw/plugin-sdk/account-core`
- `openclaw/plugin-sdk/account-id`
- `openclaw/plugin-sdk/account-resolution`
- `openclaw/plugin-sdk/account-helpers` は、複数アカウント設定とデフォルトアカウントへのフォールバック向け
- `openclaw/plugin-sdk/inbound-envelope` と
  `openclaw/plugin-sdk/inbound-reply-dispatch` は、受信ルート/エンベロープおよび record-and-dispatch 配線向け
- `openclaw/plugin-sdk/messaging-targets` は、ターゲットの解析/マッチング向け
- `openclaw/plugin-sdk/outbound-media` と
  `openclaw/plugin-sdk/outbound-runtime` は、メディア読み込みと送信 identity/send delegate 向け
- `openclaw/plugin-sdk/thread-bindings-runtime` は、スレッドバインディングのライフサイクルと adapter 登録向け
- `openclaw/plugin-sdk/agent-media-payload` は、レガシー agent/media ペイロードのフィールドレイアウトがまだ必要な場合のみ
- `openclaw/plugin-sdk/telegram-command-config` は、Telegram カスタムコマンドの正規化、重複/競合検証、およびフォールバック安定なコマンド設定契約向け

認証専用チャネルは通常、デフォルト経路で十分です。コアが承認を処理し、Plugin は送信/auth capability を公開するだけで済みます。Matrix、Slack、Telegram、およびカスタムチャット転送のようなネイティブ承認チャネルは、独自に承認ライフサイクルを実装するのではなく、共有ネイティブヘルパーを使用してください。

## 受信メンションポリシー

受信メンション処理は、次の2層に分けてください。

- Plugin 所有の証拠収集
- 共有ポリシー評価

メンションポリシーの判定には `openclaw/plugin-sdk/channel-mention-gating` を使用してください。
より広い受信ヘルパーバレルが必要な場合にのみ `openclaw/plugin-sdk/channel-inbound` を使用してください。

Plugin ローカルロジックに向いているもの:

- bot への返信検出
- bot の引用検出
- スレッド参加チェック
- service/system メッセージの除外
- bot 参加を証明するために必要なプラットフォームネイティブのキャッシュ

共有ヘルパーに向いているもの:

- `requireMention`
- 明示的メンション結果
- 暗黙的メンション許可リスト
- コマンドバイパス
- 最終的なスキップ判定

推奨フロー:

1. ローカルのメンション事実を計算する。
2. その事実を `resolveInboundMentionDecision({ facts, policy })` に渡す。
3. `decision.effectiveWasMentioned`、`decision.shouldBypassMention`、`decision.shouldSkip` を受信ゲートで使用する。

```typescript
import {
  implicitMentionKindWhen,
  matchesMentionWithExplicit,
  resolveInboundMentionDecision,
} from "openclaw/plugin-sdk/channel-inbound";

const mentionMatch = matchesMentionWithExplicit(text, {
  mentionRegexes,
  mentionPatterns,
});

const facts = {
  canDetectMention: true,
  wasMentioned: mentionMatch.matched,
  hasAnyMention: mentionMatch.hasExplicitMention,
  implicitMentionKinds: [
    ...implicitMentionKindWhen("reply_to_bot", isReplyToBot),
    ...implicitMentionKindWhen("quoted_bot", isQuoteOfBot),
  ],
};

const decision = resolveInboundMentionDecision({
  facts,
  policy: {
    isGroup,
    requireMention,
    allowedImplicitMentionKinds: requireExplicitMention ? [] : ["reply_to_bot", "quoted_bot"],
    allowTextCommands,
    hasControlCommand,
    commandAuthorized,
  },
});

if (decision.shouldSkip) return;
```

`api.runtime.channel.mentions` は、すでにランタイム注入に依存している同梱チャネル Plugin 向けに、同じ共有メンションヘルパーを公開します。

- `buildMentionRegexes`
- `matchesMentionPatterns`
- `matchesMentionWithExplicit`
- `implicitMentionKindWhen`
- `resolveInboundMentionDecision`

`implicitMentionKindWhen` と `resolveInboundMentionDecision` だけが必要な場合は、無関係な受信ランタイムヘルパーの読み込みを避けるため、`openclaw/plugin-sdk/channel-mention-gating` から import してください。

古い `resolveMentionGating*` ヘルパーは、互換エクスポートとしてのみ `openclaw/plugin-sdk/channel-inbound` に残っています。新しいコードでは `resolveInboundMentionDecision({ facts, policy })` を使用してください。

## ウォークスルー

<Steps>
  <a id="step-1-package-and-manifest"></a>
  <Step title="パッケージとマニフェスト">
    標準の Plugin ファイルを作成します。`package.json` の `channel` フィールドによって、これがチャネル Plugin になります。完全なパッケージメタデータの公開インターフェースについては、[Plugin Setup and Config](/ja-JP/plugins/sdk-setup#openclaw-channel) を参照してください。

    <CodeGroup>
    ```json package.json
    {
      "name": "@myorg/openclaw-acme-chat",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "setupEntry": "./setup-entry.ts",
        "channel": {
          "id": "acme-chat",
          "label": "Acme Chat",
          "blurb": "Connect OpenClaw to Acme Chat."
        }
      }
    }
    ```

    ```json openclaw.plugin.json
    {
      "id": "acme-chat",
      "kind": "channel",
      "channels": ["acme-chat"],
      "name": "Acme Chat",
      "description": "Acme Chat channel plugin",
      "configSchema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "acme-chat": {
            "type": "object",
            "properties": {
              "token": { "type": "string" },
              "allowFrom": {
                "type": "array",
                "items": { "type": "string" }
              }
            }
          }
        }
      }
    }
    ```
    </CodeGroup>

  </Step>

  <Step title="チャネル Plugin オブジェクトを構築する">
    `ChannelPlugin` インターフェースには、多くの任意アダプター公開インターフェースがあります。最小構成である `id` と `setup` から始め、必要に応じてアダプターを追加してください。

    `src/channel.ts` を作成します:

    ```typescript src/channel.ts
    import {
      createChatChannelPlugin,
      createChannelPluginBase,
    } from "openclaw/plugin-sdk/channel-core";
    import type { OpenClawConfig } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatApi } from "./client.js"; // your platform API client

    type ResolvedAccount = {
      accountId: string | null;
      token: string;
      allowFrom: string[];
      dmPolicy: string | undefined;
    };

    function resolveAccount(
      cfg: OpenClawConfig,
      accountId?: string | null,
    ): ResolvedAccount {
      const section = (cfg.channels as Record<string, any>)?.["acme-chat"];
      const token = section?.token;
      if (!token) throw new Error("acme-chat: token is required");
      return {
        accountId: accountId ?? null,
        token,
        allowFrom: section?.allowFrom ?? [],
        dmPolicy: section?.dmSecurity,
      };
    }

    export const acmeChatPlugin = createChatChannelPlugin<ResolvedAccount>({
      base: createChannelPluginBase({
        id: "acme-chat",
        setup: {
          resolveAccount,
          inspectAccount(cfg, accountId) {
            const section =
              (cfg.channels as Record<string, any>)?.["acme-chat"];
            return {
              enabled: Boolean(section?.token),
              configured: Boolean(section?.token),
              tokenStatus: section?.token ? "available" : "missing",
            };
          },
        },
      }),

      // DM security: who can message the bot
      security: {
        dm: {
          channelKey: "acme-chat",
          resolvePolicy: (account) => account.dmPolicy,
          resolveAllowFrom: (account) => account.allowFrom,
          defaultPolicy: "allowlist",
        },
      },

      // Pairing: approval flow for new DM contacts
      pairing: {
        text: {
          idLabel: "Acme Chat username",
          message: "Send this code to verify your identity:",
          notify: async ({ target, code }) => {
            await acmeChatApi.sendDm(target, `Pairing code: ${code}`);
          },
        },
      },

      // Threading: how replies are delivered
      threading: { topLevelReplyToMode: "reply" },

      // Outbound: send messages to the platform
      outbound: {
        attachedResults: {
          sendText: async (params) => {
            const result = await acmeChatApi.sendMessage(
              params.to,
              params.text,
            );
            return { messageId: result.id };
          },
        },
        base: {
          sendMedia: async (params) => {
            await acmeChatApi.sendFile(params.to, params.filePath);
          },
        },
      },
    });
    ```

    <Accordion title="createChatChannelPlugin が代わりに行ってくれること">
      低レベルのアダプターインターフェースを手作業で実装する代わりに、宣言的なオプションを渡すことで、builder がそれらを組み立てます。

      | Option | 配線されるもの |
      | --- | --- |
      | `security.dm` | 設定フィールドからのスコープ付き DM セキュリティリゾルバー |
      | `pairing.text` | コード交換を伴うテキストベースの DM ペアリングフロー |
      | `threading` | reply-to モードのリゾルバー（固定、アカウントスコープ、またはカスタム） |
      | `outbound.attachedResults` | 結果メタデータ（message ID）を返す送信関数 |

      完全な制御が必要な場合は、宣言的オプションの代わりに生のアダプターオブジェクトを渡すこともできます。
    </Accordion>

  </Step>

  <Step title="エントリーポイントを配線する">
    `index.ts` を作成します:

    ```typescript index.ts
    import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineChannelPluginEntry({
      id: "acme-chat",
      name: "Acme Chat",
      description: "Acme Chat channel plugin",
      plugin: acmeChatPlugin,
      registerCliMetadata(api) {
        api.registerCli(
          ({ program }) => {
            program
              .command("acme-chat")
              .description("Acme Chat management");
          },
          {
            descriptors: [
              {
                name: "acme-chat",
                description: "Acme Chat management",
                hasSubcommands: false,
              },
            ],
          },
        );
      },
      registerFull(api) {
        api.registerGatewayMethod(/* ... */);
      },
    });
    ```

    チャネル所有の CLI descriptor は `registerCliMetadata(...)` に置いてください。これにより、OpenClaw は完全なチャネルランタイムを有効化せずにそれらをルートヘルプに表示でき、通常の完全ロードでも実際のコマンド登録のために同じ descriptor を取得できます。`registerFull(...)` はランタイム専用の処理に保持してください。
    `registerFull(...)` が Gateway RPC メソッドを登録する場合は、Plugin 固有のプレフィックスを使用してください。コア管理 namespace（`config.*`、`exec.approvals.*`、`wizard.*`、`update.*`）は予約済みで、常に `operator.admin` に解決されます。
    `defineChannelPluginEntry` は登録モードの分割を自動的に処理します。すべてのオプションについては [Entry Points](/ja-JP/plugins/sdk-entrypoints#definechannelpluginentry) を参照してください。

  </Step>

  <Step title="セットアップエントリーを追加する">
    オンボーディング中の軽量ロード用に `setup-entry.ts` を作成します:

    ```typescript setup-entry.ts
    import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineSetupPluginEntry(acmeChatPlugin);
    ```

    OpenClaw は、チャネルが無効または未設定の場合に、完全なエントリーではなくこちらを読み込みます。これにより、セットアップフロー中に重いランタイムコードを引き込まずに済みます。詳細は [Setup and Config](/ja-JP/plugins/sdk-setup#setup-entry) を参照してください。

    セットアップ安全なエクスポートを sidecar モジュールへ分離する同梱ワークスペースチャネルは、明示的なセットアップ時ランタイム setter も必要とする場合、`openclaw/plugin-sdk/channel-entry-contract` の `defineBundledChannelSetupEntry(...)` を使用できます。

  </Step>

  <Step title="受信メッセージを処理する">
    Plugin は、プラットフォームからメッセージを受信し、それを OpenClaw に転送する必要があります。典型的なパターンは、リクエストを検証し、チャネルの受信ハンドラー経由でディスパッチする Webhook です。

    ```typescript
    registerFull(api) {
      api.registerHttpRoute({
        path: "/acme-chat/webhook",
        auth: "plugin", // plugin-managed auth (verify signatures yourself)
        handler: async (req, res) => {
          const event = parseWebhookPayload(req);

          // Your inbound handler dispatches the message to OpenClaw.
          // The exact wiring depends on your platform SDK —
          // see a real example in the bundled Microsoft Teams or Google Chat plugin package.
          await handleAcmeChatInbound(api, event);

          res.statusCode = 200;
          res.end("ok");
          return true;
        },
      });
    }
    ```

    <Note>
      受信メッセージ処理はチャネル固有です。各チャネル Plugin がそれぞれの受信パイプラインを所有します。実際のパターンについては、同梱チャネル Plugin（たとえば Microsoft Teams または Google Chat の Plugin パッケージ）を参照してください。
    </Note>

  </Step>

<a id="step-6-test"></a>
<Step title="テスト">
`src/channel.test.ts` に同居テストを書いてください:

    ```typescript src/channel.test.ts
    import { describe, it, expect } from "vitest";
    import { acmeChatPlugin } from "./channel.js";

    describe("acme-chat plugin", () => {
      it("resolves account from config", () => {
        const cfg = {
          channels: {
            "acme-chat": { token: "test-token", allowFrom: ["user1"] },
          },
        } as any;
        const account = acmeChatPlugin.setup!.resolveAccount(cfg, undefined);
        expect(account.token).toBe("test-token");
      });

      it("inspects account without materializing secrets", () => {
        const cfg = {
          channels: { "acme-chat": { token: "test-token" } },
        } as any;
        const result = acmeChatPlugin.setup!.inspectAccount!(cfg, undefined);
        expect(result.configured).toBe(true);
        expect(result.tokenStatus).toBe("available");
      });

      it("reports missing config", () => {
        const cfg = { channels: {} } as any;
        const result = acmeChatPlugin.setup!.inspectAccount!(cfg, undefined);
        expect(result.configured).toBe(false);
      });
    });
    ```

    ```bash
    pnpm test -- <bundled-plugin-root>/acme-chat/
    ```

    共有テストヘルパーについては、[Testing](/ja-JP/plugins/sdk-testing) を参照してください。

  </Step>
</Steps>

## ファイル構成

```
<bundled-plugin-root>/acme-chat/
├── package.json              # openclaw.channel メタデータ
├── openclaw.plugin.json      # 設定スキーマを含むマニフェスト
├── index.ts                  # defineChannelPluginEntry
├── setup-entry.ts            # defineSetupPluginEntry
├── api.ts                    # 公開エクスポート（任意）
├── runtime-api.ts            # 内部ランタイムエクスポート（任意）
└── src/
    ├── channel.ts            # createChatChannelPlugin による ChannelPlugin
    ├── channel.test.ts       # テスト
    ├── client.ts             # プラットフォーム API クライアント
    └── runtime.ts            # ランタイムストア（必要な場合）
```

## 高度なトピック

<CardGroup cols={2}>
  <Card title="スレッド化オプション" icon="git-branch" href="/ja-JP/plugins/sdk-entrypoints#registration-mode">
    固定、アカウントスコープ、またはカスタムの reply モード
  </Card>
  <Card title="message ツール統合" icon="puzzle" href="/ja-JP/plugins/architecture#channel-plugins-and-the-shared-message-tool">
    describeMessageTool とアクション検出
  </Card>
  <Card title="ターゲット解決" icon="crosshair" href="/ja-JP/plugins/architecture#channel-target-resolution">
    inferTargetChatType、looksLikeId、resolveTarget
  </Card>
  <Card title="ランタイムヘルパー" icon="settings" href="/ja-JP/plugins/sdk-runtime">
    api.runtime 経由の TTS、STT、メディア、subagent
  </Card>
</CardGroup>

<Note>
一部の同梱ヘルパー公開インターフェースは、同梱 Plugin の保守と互換性のためにまだ存在しています。これらは新しいチャネル Plugin に推奨されるパターンではありません。その同梱 Plugin ファミリーを直接保守しているのでない限り、共通 SDK 公開インターフェースの汎用 channel/setup/reply/runtime subpath を優先してください。
</Note>

## 次のステップ

- [Provider Plugins](/ja-JP/plugins/sdk-provider-plugins) — Plugin がモデルも提供する場合
- [SDK Overview](/ja-JP/plugins/sdk-overview) — 完全な subpath import リファレンス
- [SDK Testing](/ja-JP/plugins/sdk-testing) — テストユーティリティとコントラクトテスト
- [Plugin Manifest](/ja-JP/plugins/manifest) — 完全なマニフェストスキーマ
