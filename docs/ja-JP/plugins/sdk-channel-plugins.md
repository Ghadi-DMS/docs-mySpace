---
read_when:
    - 新しいメッセージングチャンネルプラグインを構築している場合
    - OpenClawをメッセージングプラットフォームに接続したい場合
    - ChannelPluginアダプターサーフェスを理解する必要がある場合
sidebarTitle: Channel Plugins
summary: OpenClaw向けメッセージングチャンネルプラグインを構築するためのステップバイステップガイド
title: チャンネルプラグインの構築
x-i18n:
    generated_at: "2026-04-11T02:46:50Z"
    model: gpt-5.4
    provider: openai
    source_hash: 8a026e924f9ae8a3ddd46287674443bcfccb0247be504261522b078e1f440aef
    source_path: plugins/sdk-channel-plugins.md
    workflow: 15
---

# チャンネルプラグインの構築

このガイドでは、OpenClawをメッセージングプラットフォームに接続するチャンネルプラグインの構築方法を説明します。最後には、DMセキュリティ、pairing、返信スレッド化、アウトバウンドメッセージングを備えた動作するチャンネルが完成します。

<Info>
  まだOpenClawプラグインを一度も構築したことがない場合は、まず基本的なパッケージ構造とmanifest設定について[はじめに](/ja-JP/plugins/building-plugins)を読んでください。
</Info>

## チャンネルプラグインの仕組み

チャンネルプラグインには、独自のsend/edit/reactツールは不要です。OpenClawはコア内で1つの共有`message`ツールを維持します。プラグインが担当するのは次の領域です。

- **設定** — アカウント解決とセットアップウィザード
- **セキュリティ** — DMポリシーとallowlist
- **Pairing** — DM承認フロー
- **セッショングラマー** — プロバイダー固有の会話IDを、ベースチャット、スレッドID、親フォールバックへどう対応付けるか
- **アウトバウンド** — テキスト、メディア、pollをプラットフォームへ送信
- **スレッド化** — 返信をどうスレッド化するか

コアは、共有messageツール、プロンプト配線、外側のsession-key形状、汎用的な`:thread:`管理、およびディスパッチを担当します。

プラットフォームが会話IDの中に追加のscopeを保持する場合、その解析はプラグイン内で`messaging.resolveSessionConversation(...)`を使って維持してください。これは、`rawId`をベース会話ID、任意のスレッドID、明示的な`baseConversationId`、および任意の`parentConversationCandidates`に対応付けるための正式なフックです。`parentConversationCandidates`を返す場合は、最も狭い親から最も広い/ベース会話の順に並べてください。

同じ解析をチャンネルレジストリ起動前に必要とする同梱プラグインは、一致する`resolveSessionConversation(...)`エクスポートを持つトップレベルの`session-key-api.ts`ファイルも公開できます。コアは、ランタイムプラグインレジストリがまだ利用できないときにのみ、このbootstrap安全なサーフェスを使用します。

`messaging.resolveParentConversationCandidates(...)`は、プラグインが汎用/raw IDの上に親フォールバックだけを必要とする場合の、レガシー互換フォールバックとして引き続き利用できます。両方のフックが存在する場合、コアはまず`resolveSessionConversation(...).parentConversationCandidates`を使い、その正式なフックで省略された場合にのみ`resolveParentConversationCandidates(...)`へフォールバックします。

## 承認とチャンネルcapability

ほとんどのチャンネルプラグインでは、承認専用コードは不要です。

- コアは、同一チャット内の`/approve`、共有承認ボタンpayload、および汎用フォールバック配信を担当します。
- チャンネルに承認固有の動作が必要な場合は、チャンネルプラグイン上の1つの`approvalCapability`オブジェクトを優先してください。
- `ChannelPlugin.approvals`は削除されました。承認配信/native/render/authの情報は`approvalCapability`に置いてください。
- `plugin.auth`はlogin/logout専用です。コアはそのオブジェクトから承認authフックをもう読みません。
- `approvalCapability.authorizeActorAction`と`approvalCapability.getActionAvailabilityState`が正式な承認authの接合面です。
- 同一チャット内の承認auth可用性には`approvalCapability.getActionAvailabilityState`を使用してください。
- チャンネルがnative exec承認を公開する場合、開始元サーフェス/native client状態が同一チャット承認authと異なるときは`approvalCapability.getExecInitiatingSurfaceState`を使用してください。コアはこのexec専用フックを使って`enabled`と`disabled`を区別し、開始元チャンネルがnative exec承認をサポートしているかを判断し、native clientフォールバック案内にそのチャンネルを含めます。`createApproverRestrictedNativeApprovalCapability(...)`は一般的なケースでこれを補完します。
- 重複するローカル承認プロンプトの非表示や、配信前のtyping indicator送信のようなチャンネル固有のpayloadライフサイクル動作には、`outbound.shouldSuppressLocalPayloadPrompt`または`outbound.beforeDeliverPayload`を使用してください。
- `approvalCapability.delivery`は、native承認ルーティングまたはフォールバック抑制にのみ使用してください。
- `approvalCapability.nativeRuntime`は、チャンネル所有のnative承認情報に使用してください。ホットなチャンネルエントリポイントでは、`createLazyChannelApprovalNativeRuntimeAdapter(...)`で遅延化してください。これにより、コアが承認ライフサイクルを組み立てられる一方で、必要に応じてランタイムモジュールをオンデマンドでimportできます。
- `approvalCapability.render`は、チャンネルが共有レンダラーではなく本当にカスタム承認payloadを必要とする場合にのみ使用してください。
- チャンネルが無効時の返信で、native exec承認を有効化するために必要な正確な設定ノブを説明したい場合は`approvalCapability.describeExecApprovalSetup`を使用してください。このフックは`{ channel, channelLabel, accountId }`を受け取ります。名前付きアカウントのあるチャンネルは、トップレベルのデフォルトではなく、`channels.<channel>.accounts.<id>.execApprovals.*`のようなアカウントスコープのパスを描画するべきです。
- チャンネルが既存設定から安定したowner風のDM IDを推測できるなら、承認固有のコアロジックを追加せずに同一チャット内の`/approve`を制限するために、`openclaw/plugin-sdk/approval-runtime`の`createResolvedApproverActionAuthAdapter`を使用してください。
- チャンネルにnative承認配信が必要な場合、チャンネルコードはターゲット正規化とトランスポート/プレゼンテーション情報に集中させてください。`openclaw/plugin-sdk/approval-runtime`の`createChannelExecApprovalProfile`、`createChannelNativeOriginTargetResolver`、`createChannelApproverDmTargetResolver`、`createApproverRestrictedNativeApprovalCapability`を使用してください。チャンネル固有の情報は、理想的には`createChannelApprovalNativeRuntimeAdapter(...)`または`createLazyChannelApprovalNativeRuntimeAdapter(...)`を通じて`approvalCapability.nativeRuntime`の背後に置いてください。これにより、コアはハンドラーを組み立て、リクエストフィルタリング、ルーティング、重複排除、有効期限、Gatewayサブスクリプション、別経路通知を担当できます。`nativeRuntime`はいくつかのより小さい接合面に分割されています:
- `availability` — アカウントが設定されているか、およびリクエストを処理すべきかどうか
- `presentation` — 共有承認view modelを保留中/解決済み/期限切れのnative payloadまたは最終アクションに対応付ける
- `transport` — ターゲットを準備し、native承認メッセージを送信/更新/削除する
- `interactions` — nativeボタンまたはreaction向けの任意のbind/unbind/clear-actionフック
- `observe` — 任意の配信診断フック
- チャンネルがclient、token、Bolt app、webhook receiverのようなランタイム所有オブジェクトを必要とする場合は、`openclaw/plugin-sdk/channel-runtime-context`を通じて登録してください。汎用runtime-contextレジストリにより、コアは承認固有のラッパー接着コードを追加せずに、チャンネル起動状態からcapability駆動ハンドラーをbootstrapできます。
- capability駆動の接合面だけではまだ表現力が足りない場合にのみ、より低レベルな`createChannelApprovalHandler`または`createChannelNativeApprovalRuntime`を使用してください。
- native承認チャンネルは、`accountId`と`approvalKind`の両方をそれらのヘルパー経由でルーティングする必要があります。`accountId`はマルチアカウント承認ポリシーを正しいボットアカウントにスコープし、`approvalKind`はコアにハードコードされた分岐なしで、execとプラグイン承認の動作をチャンネルから利用可能に保ちます。
- コアは現在、承認の再ルーティング通知も担当します。チャンネルプラグインは、`createChannelNativeApprovalRuntime`から独自の「承認はDM/別チャンネルへ送られました」フォローアップメッセージを送るべきではありません。代わりに、共有承認capabilityヘルパーを通じて正確なorigin + approver-DMルーティングを公開し、開始元チャットへ通知を返す前にコアが実際の配信を集約できるようにしてください。
- 配信された承認IDの種類をエンドツーエンドで維持してください。native clientは、チャンネルローカル状態からexecかプラグインかの承認ルーティングを推測したり書き換えたりしてはいけません。
- 異なる承認種別は、意図的に異なるnativeサーフェスを公開できます。
  現在の同梱例:
  - Slackは、exec IDとプラグインIDの両方に対してnative承認ルーティングを利用可能なままにします。
  - Matrixは、exec承認とプラグイン承認で同じnative DM/チャンネルルーティングとreaction UXを維持しつつ、承認種別ごとにauthを異ならせることができます。
- `createApproverRestrictedNativeApprovalAdapter`は互換ラッパーとして依然存在しますが、新しいコードではcapability builderを優先し、プラグイン上に`approvalCapability`を公開してください。

ホットなチャンネルエントリポイントでは、そのファミリーの一部だけが必要な場合、より狭いランタイムsubpathを優先してください。

- `openclaw/plugin-sdk/approval-auth-runtime`
- `openclaw/plugin-sdk/approval-client-runtime`
- `openclaw/plugin-sdk/approval-delivery-runtime`
- `openclaw/plugin-sdk/approval-gateway-runtime`
- `openclaw/plugin-sdk/approval-handler-adapter-runtime`
- `openclaw/plugin-sdk/approval-handler-runtime`
- `openclaw/plugin-sdk/approval-native-runtime`
- `openclaw/plugin-sdk/approval-reply-runtime`
- `openclaw/plugin-sdk/channel-runtime-context`

同様に、より広い包括サーフェスが不要な場合は、`openclaw/plugin-sdk/setup-runtime`、`openclaw/plugin-sdk/setup-adapter-runtime`、`openclaw/plugin-sdk/reply-runtime`、`openclaw/plugin-sdk/reply-dispatch-runtime`、`openclaw/plugin-sdk/reply-reference`、`openclaw/plugin-sdk/reply-chunking`を優先してください。

セットアップについては特に:

- `openclaw/plugin-sdk/setup-runtime`は、ランタイム安全なセットアップヘルパーを扱います:
  import安全なセットアップpatchアダプター（`createPatchedAccountSetupAdapter`、`createEnvPatchedAccountSetupAdapter`、`createSetupInputPresenceValidator`）、lookup-note出力、`promptResolvedAllowFrom`、`splitSetupEntries`、および委譲セットアップproxy builder
- `openclaw/plugin-sdk/setup-adapter-runtime`は、`createEnvPatchedAccountSetupAdapter`向けの狭いenv対応アダプター接合面です
- `openclaw/plugin-sdk/channel-setup`は、optional-installセットアップbuilderといくつかのセットアップ安全プリミティブを扱います:
  `createOptionalChannelSetupSurface`、`createOptionalChannelSetupAdapter`、

チャンネルがenv駆動のセットアップやauthをサポートし、ランタイムがロードされる前に汎用の起動/設定フローがそれらのenv名を知る必要がある場合は、プラグインmanifestで`channelEnvVars`として宣言してください。チャンネルランタイムの`envVars`またはローカル定数は、operator向けコピー専用にしてください。
`createOptionalChannelSetupWizard`、`DEFAULT_ACCOUNT_ID`、`createTopLevelChannelDmPolicy`、`setSetupChannelEnabled`、および`splitSetupEntries`

- `moveSingleAccountChannelSectionToDefaultAccount(...)`のような、より重い共有セットアップ/設定ヘルパーも必要な場合にのみ、より広い`openclaw/plugin-sdk/setup`接合面を使用してください

チャンネルがセットアップサーフェス内で「まずこのプラグインをインストールしてください」と案内するだけでよい場合は、`createOptionalChannelSetupSurface(...)`を優先してください。生成されるadapter/wizardは、設定書き込みと最終化でfail closedし、検証・最終化・docsリンクのコピー全体で同じインストール必須メッセージを再利用します。

その他のホットなチャンネルパスでも、より広いレガシーサーフェスより狭いヘルパーを優先してください。

- `openclaw/plugin-sdk/account-core`、
  `openclaw/plugin-sdk/account-id`、
  `openclaw/plugin-sdk/account-resolution`、および
  `openclaw/plugin-sdk/account-helpers`は、マルチアカウント設定と
  デフォルトアカウントフォールバック向け
- `openclaw/plugin-sdk/inbound-envelope`と
  `openclaw/plugin-sdk/inbound-reply-dispatch`は、インバウンドのルート/envelopeと
  record-and-dispatch配線向け
- `openclaw/plugin-sdk/messaging-targets`は、ターゲット解析/照合向け
- `openclaw/plugin-sdk/outbound-media`と
  `openclaw/plugin-sdk/outbound-runtime`は、メディア読み込みと
  アウトバウンドID/送信delegate向け
- `openclaw/plugin-sdk/thread-bindings-runtime`は、スレッドbindingライフサイクル
  とadapter登録向け
- `openclaw/plugin-sdk/agent-media-payload`は、レガシーのagent/media
  payloadフィールドレイアウトがまだ必要な場合のみ
- `openclaw/plugin-sdk/telegram-command-config`は、Telegramカスタムコマンド
  正規化、重複/競合検証、およびフォールバック安定なコマンド
  設定契約向け

認証専用チャンネルは通常、デフォルトパスで十分です。コアが承認を処理し、プラグインはアウトバウンド/auth capabilityを公開するだけです。Matrix、Slack、Telegram、およびカスタムチャットトランスポートのようなnative承認チャンネルは、独自に承認ライフサイクルを実装するのではなく、共有nativeヘルパーを使用してください。

## インバウンドメンションポリシー

インバウンドメンション処理は、次の2層に分けて維持してください。

- プラグイン所有の証拠収集
- 共有ポリシー評価

共有レイヤーには`openclaw/plugin-sdk/channel-inbound`を使用してください。

プラグインローカルロジックに適しているもの:

- botへの返信検出
- bot引用の検出
- スレッド参加チェック
- service/systemメッセージの除外
- bot参加を証明するために必要なプラットフォームネイティブキャッシュ

共有ヘルパーに適しているもの:

- `requireMention`
- 明示的メンション結果
- 暗黙的メンションallowlist
- コマンドバイパス
- 最終的なスキップ判定

推奨フロー:

1. ローカルなメンション事実を計算します。
2. その事実を`resolveInboundMentionDecision({ facts, policy })`に渡します。
3. インバウンドゲートで`decision.effectiveWasMentioned`、`decision.shouldBypassMention`、`decision.shouldSkip`を使用します。

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

`api.runtime.channel.mentions`は、すでにランタイム注入に依存している同梱チャンネルプラグイン向けに、同じ共有メンションヘルパーを公開します。

- `buildMentionRegexes`
- `matchesMentionPatterns`
- `matchesMentionWithExplicit`
- `implicitMentionKindWhen`
- `resolveInboundMentionDecision`

古い`resolveMentionGating*`ヘルパーは、`openclaw/plugin-sdk/channel-inbound`上に互換エクスポートとしてのみ残っています。新しいコードでは`resolveInboundMentionDecision({ facts, policy })`を使用してください。

## ウォークスルー

<Steps>
  <a id="step-1-package-and-manifest"></a>
  <Step title="パッケージとmanifest">
    標準的なプラグインファイルを作成します。`package.json`内の`channel`フィールドが、これをチャンネルプラグインにします。完全なパッケージメタデータサーフェスについては、[Plugin Setup and Config](/ja-JP/plugins/sdk-setup#openclaw-channel)を参照してください。

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

  <Step title="チャンネルプラグインオブジェクトを構築する">
    `ChannelPlugin`インターフェースには、多くの省略可能なアダプターサーフェスがあります。最小構成の`id`と`setup`から始め、必要に応じてアダプターを追加してください。

    `src/channel.ts`を作成します:

    ```typescript src/channel.ts
    import {
      createChatChannelPlugin,
      createChannelPluginBase,
    } from "openclaw/plugin-sdk/channel-core";
    import type { OpenClawConfig } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatApi } from "./client.js"; // あなたのプラットフォームAPIクライアント

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

      // DMセキュリティ: botにメッセージを送れる相手
      security: {
        dm: {
          channelKey: "acme-chat",
          resolvePolicy: (account) => account.dmPolicy,
          resolveAllowFrom: (account) => account.allowFrom,
          defaultPolicy: "allowlist",
        },
      },

      // Pairing: 新しいDM連絡先向け承認フロー
      pairing: {
        text: {
          idLabel: "Acme Chat username",
          message: "本人確認のため、このコードを送信してください:",
          notify: async ({ target, code }) => {
            await acmeChatApi.sendDm(target, `Pairing code: ${code}`);
          },
        },
      },

      // スレッド化: 返信の配信方法
      threading: { topLevelReplyToMode: "reply" },

      // アウトバウンド: プラットフォームへメッセージを送信
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

    <Accordion title="createChatChannelPluginが担ってくれること">
      低レベルのアダプターインターフェースを手動で実装する代わりに、宣言的なオプションを渡すと、builderがそれらを組み合わせます。

      | オプション | 配線されるもの |
      | --- | --- |
      | `security.dm` | 設定フィールドからのスコープ付きDMセキュリティリゾルバー |
      | `pairing.text` | コード交換付きのテキストベースDM pairingフロー |
      | `threading` | reply-to-modeリゾルバー（固定、アカウントスコープ、またはカスタム） |
      | `outbound.attachedResults` | 結果メタデータ（メッセージID）を返す送信関数 |

      完全に制御したい場合は、宣言的オプションの代わりに生のアダプターオブジェクトを渡すこともできます。
    </Accordion>

  </Step>

  <Step title="エントリポイントを配線する">
    `index.ts`を作成します:

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

    チャンネル所有のCLI descriptorは`registerCliMetadata(...)`に置いてください。これにより、OpenClawは完全なチャンネルランタイムを有効化せずにルートヘルプへそれらを表示でき、通常の完全ロードでも実際のコマンド登録向けに同じdescriptorを取得できます。`registerFull(...)`はランタイム専用の処理に維持してください。
    `registerFull(...)`がGateway RPCメソッドを登録する場合は、プラグイン固有の接頭辞を使用してください。コア管理者名前空間（`config.*`、`exec.approvals.*`、`wizard.*`、`update.*`）は予約済みで、常に`operator.admin`に解決されます。
    `defineChannelPluginEntry`は、登録モードの分割を自動で処理します。すべてのオプションについては[Entry Points](/ja-JP/plugins/sdk-entrypoints#definechannelpluginentry)を参照してください。

  </Step>

  <Step title="セットアップエントリを追加する">
    オンボーディング中の軽量ロード用に`setup-entry.ts`を作成します:

    ```typescript setup-entry.ts
    import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineSetupPluginEntry(acmeChatPlugin);
    ```

    OpenClawは、チャンネルが無効または未設定のとき、完全なエントリの代わりにこれをロードします。これにより、セットアップフロー中に重いランタイムコードを引き込まずに済みます。詳細は[Setup and Config](/ja-JP/plugins/sdk-setup#setup-entry)を参照してください。

  </Step>

  <Step title="インバウンドメッセージを処理する">
    プラグインは、プラットフォームからメッセージを受信し、それをOpenClawへ転送する必要があります。一般的なパターンは、リクエストを検証し、チャンネルのインバウンドハンドラーを通じてディスパッチするWebhookです。

    ```typescript
    registerFull(api) {
      api.registerHttpRoute({
        path: "/acme-chat/webhook",
        auth: "plugin", // プラグイン管理認証（署名検証は自分で行う）
        handler: async (req, res) => {
          const event = parseWebhookPayload(req);

          // あなたのインバウンドハンドラーがメッセージをOpenClawへディスパッチします。
          // 正確な配線はプラットフォームSDKに依存します —
          // 実例は、同梱のMicrosoft TeamsまたはGoogle Chatプラグインパッケージを参照してください。
          await handleAcmeChatInbound(api, event);

          res.statusCode = 200;
          res.end("ok");
          return true;
        },
      });
    }
    ```

    <Note>
      インバウンドメッセージ処理はチャンネル固有です。各チャンネルプラグインが独自のインバウンドパイプラインを所有します。実際のパターンについては、同梱チャンネルプラグイン（たとえばMicrosoft TeamsまたはGoogle Chatプラグインパッケージ）を確認してください。
    </Note>

  </Step>

<a id="step-6-test"></a>
<Step title="テスト">
`src/channel.test.ts`に同居テストを書きます:

    ```typescript src/channel.test.ts
    import { describe, it, expect } from "vitest";
    import { acmeChatPlugin } from "./channel.js";

    describe("acme-chat plugin", () => {
      it("設定からアカウントを解決する", () => {
        const cfg = {
          channels: {
            "acme-chat": { token: "test-token", allowFrom: ["user1"] },
          },
        } as any;
        const account = acmeChatPlugin.setup!.resolveAccount(cfg, undefined);
        expect(account.token).toBe("test-token");
      });

      it("secretを実体化せずにアカウントを検査する", () => {
        const cfg = {
          channels: { "acme-chat": { token: "test-token" } },
        } as any;
        const result = acmeChatPlugin.setup!.inspectAccount!(cfg, undefined);
        expect(result.configured).toBe(true);
        expect(result.tokenStatus).toBe("available");
      });

      it("設定不足を報告する", () => {
        const cfg = { channels: {} } as any;
        const result = acmeChatPlugin.setup!.inspectAccount!(cfg, undefined);
        expect(result.configured).toBe(false);
      });
    });
    ```

    ```bash
    pnpm test -- <bundled-plugin-root>/acme-chat/
    ```

    共有テストヘルパーについては、[Testing](/ja-JP/plugins/sdk-testing)を参照してください。

  </Step>
</Steps>

## ファイル構成

```text
<bundled-plugin-root>/acme-chat/
├── package.json              # openclaw.channelメタデータ
├── openclaw.plugin.json      # 設定スキーマを含むmanifest
├── index.ts                  # defineChannelPluginEntry
├── setup-entry.ts            # defineSetupPluginEntry
├── api.ts                    # 公開エクスポート（任意）
├── runtime-api.ts            # 内部ランタイムエクスポート（任意）
└── src/
    ├── channel.ts            # createChatChannelPlugin経由のChannelPlugin
    ├── channel.test.ts       # テスト
    ├── client.ts             # プラットフォームAPIクライアント
    └── runtime.ts            # ランタイムストア（必要な場合）
```

## 高度なトピック

<CardGroup cols={2}>
  <Card title="スレッド化オプション" icon="git-branch" href="/ja-JP/plugins/sdk-entrypoints#registration-mode">
    固定、アカウントスコープ、またはカスタムの返信モード
  </Card>
  <Card title="messageツール統合" icon="puzzle" href="/ja-JP/plugins/architecture#channel-plugins-and-the-shared-message-tool">
    describeMessageToolとアクションディスカバリー
  </Card>
  <Card title="ターゲット解決" icon="crosshair" href="/ja-JP/plugins/architecture#channel-target-resolution">
    inferTargetChatType、looksLikeId、resolveTarget
  </Card>
  <Card title="ランタイムヘルパー" icon="settings" href="/ja-JP/plugins/sdk-runtime">
    api.runtime経由のTTS、STT、メディア、subagent
  </Card>
</CardGroup>

<Note>
一部の同梱ヘルパー接合面は、同梱プラグインの保守と互換性のために引き続き存在します。これらは新しいチャンネルプラグイン向けの推奨パターンではありません。その同梱プラグインファミリーを直接保守しているのでない限り、共通SDKサーフェスの汎用channel/setup/reply/runtime subpathを優先してください。
</Note>

## 次のステップ

- [プロバイダープラグイン](/ja-JP/plugins/sdk-provider-plugins) — プラグインがモデルも提供する場合
- [SDK概要](/ja-JP/plugins/sdk-overview) — 完全なsubpath importリファレンス
- [SDKテスト](/ja-JP/plugins/sdk-testing) — テストユーティリティと契約テスト
- [プラグインmanifest](/ja-JP/plugins/manifest) — 完全なmanifestスキーマ
