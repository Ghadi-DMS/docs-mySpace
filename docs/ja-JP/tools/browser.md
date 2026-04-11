---
read_when:
    - エージェント制御のブラウザ自動化の追加
    - OpenClaw が自分の Chrome に干渉している理由のデバッグ
    - macOS アプリでのブラウザ設定とライフサイクルの実装
summary: 統合ブラウザ制御サービス + アクションコマンド
title: ブラウザ（OpenClaw 管理）
x-i18n:
    generated_at: "2026-04-11T02:48:37Z"
    model: gpt-5.4
    provider: openai
    source_hash: da6fed36a6f40a50e825f90e5616778954545bd7e52397f7e088b85251ee024f
    source_path: tools/browser.md
    workflow: 15
---

# ブラウザ（openclaw 管理）

OpenClaw は、エージェントが制御する**専用の Chrome/Brave/Edge/Chromium プロファイル**を実行できます。
これは個人用ブラウザから分離されており、Gateway 内の小さなローカル
制御サービス（loopback のみ）を通じて管理されます。

初心者向けの見方:

- これは**エージェント専用の別ブラウザ**だと考えてください。
- `openclaw` プロファイルは個人用ブラウザプロファイルには**触れません**。
- エージェントは安全なレーン内で**タブを開き、ページを読み取り、クリックし、入力**できます。
- 組み込みの `user` プロファイルは、Chrome MCP を通じて、実際にサインイン済みの Chrome セッションに接続します。

## できること

- **openclaw** という名前の別ブラウザプロファイル（デフォルトでオレンジのアクセント）。
- 決定論的なタブ制御（一覧表示/開く/フォーカス/閉じる）。
- エージェントアクション（クリック/入力/ドラッグ/選択）、スナップショット、スクリーンショット、PDF。
- 任意のマルチプロファイル対応（`openclaw`、`work`、`remote`、...）。

このブラウザは日常使いのブラウザ**ではありません**。これは
エージェント自動化と検証のための、安全で分離されたサーフェスです。

## クイックスタート

```bash
openclaw browser --browser-profile openclaw status
openclaw browser --browser-profile openclaw start
openclaw browser --browser-profile openclaw open https://example.com
openclaw browser --browser-profile openclaw snapshot
```

「Browser disabled」と表示される場合は、config で有効にし（以下を参照）、
Gateway を再起動してください。

`openclaw browser` 自体が存在しない、またはエージェントが browser tool
を利用できないと言う場合は、[Missing browser command or tool](/ja-JP/tools/browser#missing-browser-command-or-tool) に進んでください。

## plugin 制御

デフォルトの `browser` tool は、現在デフォルトで有効な
バンドル plugin です。つまり、OpenClaw の残りの plugin システムを削除せずに、
これを無効化または置き換えできます:

```json5
{
  plugins: {
    entries: {
      browser: {
        enabled: false,
      },
    },
  },
}
```

同じ `browser` tool 名を提供する別の plugin をインストールする前に、
バンドル plugin を無効にしてください。デフォルトのブラウザ体験には次の両方が必要です:

- `plugins.entries.browser.enabled` が無効化されていないこと
- `browser.enabled=true`

plugin だけをオフにすると、バンドルされた browser CLI（`openclaw browser`）、
gateway メソッド（`browser.request`）、エージェント tool、およびデフォルトの browser 制御
サービスがすべて一緒に消えます。`browser.*` config はそのまま残るため、
置き換え plugin が再利用できます。

バンドル browser plugin は、現在 browser ランタイム実装も所有しています。
core には共有 Plugin SDK ヘルパーと、
古い内部 import パス向けの互換性 re-export だけが残ります。実際には、
browser plugin パッケージを削除または置き換えると、
2つ目の core 所有ランタイムが残るのではなく、browser 機能セット自体がなくなります。

browser config の変更は引き続き Gateway の再起動が必要です。これはバンドル plugin が
新しい設定で browser サービスを再登録するためです。

## browser コマンドまたは tool が見つからない

アップグレード後に `openclaw browser` が突然 unknown command になったり、
エージェントが browser tool がないと報告したりする場合、最も一般的な原因は、
`browser` を含まない制限付き `plugins.allow` リストです。

問題のある config の例:

```json5
{
  plugins: {
    allow: ["telegram"],
  },
}
```

修正するには、plugin allowlist に `browser` を追加します:

```json5
{
  plugins: {
    allow: ["telegram", "browser"],
  },
}
```

重要な注意:

- `plugins.allow` が設定されている場合、`browser.enabled=true` だけでは不十分です。
- `plugins.allow` が設定されている場合、`plugins.entries.browser.enabled=true` だけでも不十分です。
- `tools.alsoAllow: ["browser"]` はバンドルされた browser plugin を読み込み**ません**。これは plugin がすでに読み込まれた後に tool ポリシーを調整するだけです。
- 制限付き plugin allowlist が不要なら、`plugins.allow` を削除してもデフォルトのバンドル browser 動作に戻せます。

典型的な症状:

- `openclaw browser` が unknown command になる。
- `browser.request` が存在しない。
- エージェントが browser tool を利用不可または欠落と報告する。

## プロファイル: `openclaw` と `user`

- `openclaw`: 管理された分離ブラウザ（拡張機能不要）。
- `user`: 実際にサインイン済みの Chrome
  セッション向けの組み込み Chrome MCP 接続プロファイル。

エージェント browser tool 呼び出しでは:

- デフォルト: 分離された `openclaw` ブラウザを使います。
- 既存のログイン済みセッションが重要で、ユーザーが
  コンピューターの前にいて接続プロンプトをクリック/承認できる場合は、`profile="user"` を優先します。
- `profile` は、特定のブラウザモードを使いたいときの明示的な上書きです。

管理モードをデフォルトにしたい場合は `browser.defaultProfile: "openclaw"` を設定します。

## 設定

browser 設定は `~/.openclaw/openclaw.json` にあります。

```json5
{
  browser: {
    enabled: true, // デフォルト: true
    ssrfPolicy: {
      // dangerouslyAllowPrivateNetwork: true, // 信頼できるプライベートネットワークアクセスにのみ opt in
      // allowPrivateNetwork: true, // legacy alias
      // hostnameAllowlist: ["*.example.com", "example.com"],
      // allowedHostnames: ["localhost"],
    },
    // cdpUrl: "http://127.0.0.1:18792", // legacy な単一プロファイル上書き
    remoteCdpTimeoutMs: 1500, // リモート CDP HTTP タイムアウト（ms）
    remoteCdpHandshakeTimeoutMs: 3000, // リモート CDP WebSocket ハンドシェイクタイムアウト（ms）
    defaultProfile: "openclaw",
    color: "#FF4500",
    headless: false,
    noSandbox: false,
    attachOnly: false,
    executablePath: "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser",
    profiles: {
      openclaw: { cdpPort: 18800, color: "#FF4500" },
      work: { cdpPort: 18801, color: "#0066CC" },
      user: {
        driver: "existing-session",
        attachOnly: true,
        color: "#00AA00",
      },
      brave: {
        driver: "existing-session",
        attachOnly: true,
        userDataDir: "~/Library/Application Support/BraveSoftware/Brave-Browser",
        color: "#FB542B",
      },
      remote: { cdpUrl: "http://10.0.0.42:9222", color: "#00AA00" },
    },
  },
}
```

注意:

- browser 制御サービスは `gateway.port` から導出されるポートで loopback にバインドします
  （デフォルト: `18791`。gateway + 2）。
- Gateway ポート（`gateway.port` または `OPENCLAW_GATEWAY_PORT`）を上書きすると、
  派生 browser ポートも同じ「ファミリー」に収まるように移動します。
- `cdpUrl` が未設定の場合、デフォルトは管理されたローカル CDP ポートです。
- `remoteCdpTimeoutMs` は、リモート（非 loopback）CDP 到達性チェックに適用されます。
- `remoteCdpHandshakeTimeoutMs` は、リモート CDP WebSocket 到達性チェックに適用されます。
- browser の navigation/open-tab は、ナビゲーション前に SSRF ガードされ、
  ナビゲーション後の最終 `http(s)` URL に対してもベストエフォートで再チェックされます。
- strict SSRF モードでは、リモート CDP エンドポイントのディスカバリー/probe（`cdpUrl`。`/json/version` 参照を含む）もチェック対象です。
- `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork` はデフォルトで無効です。意図的にプライベートネットワーク browser アクセスを信頼する場合にのみ `true` に設定してください。
- `browser.ssrfPolicy.allowPrivateNetwork` は、互換性のための legacy alias として引き続きサポートされます。
- `attachOnly: true` は「ローカルブラウザを起動しない。すでに実行中なら接続のみ行う」という意味です。
- `color` とプロファイルごとの `color` は browser UI に色を付け、どのプロファイルがアクティブかを見分けやすくします。
- デフォルトプロファイルは `openclaw`（OpenClaw 管理のスタンドアロンブラウザ）です。サインイン済みユーザーブラウザに opt in するには `defaultProfile: "user"` を使います。
- 自動検出順序: システムデフォルトブラウザが Chromium ベースならそれを使用し、そうでなければ Chrome → Brave → Edge → Chromium → Chrome Canary の順です。
- ローカル `openclaw` プロファイルは `cdpPort`/`cdpUrl` を自動割り当てするため、これらを設定するのはリモート CDP の場合だけにしてください。
- `driver: "existing-session"` は、生の CDP ではなく Chrome DevTools MCP を使います。
  この driver には `cdpUrl` を設定しないでください。
- Brave や Edge のような非デフォルトの Chromium ユーザープロファイルに
  existing-session プロファイルを接続させるには、`browser.profiles.<name>.userDataDir` を設定してください。

## Brave（または別の Chromium ベースブラウザ）を使う

**システムデフォルト**ブラウザが Chromium ベース（Chrome/Brave/Edge など）の場合、
OpenClaw は自動的にそれを使います。自動検出を上書きするには `browser.executablePath` を設定します:

CLI の例:

```bash
openclaw config set browser.executablePath "/usr/bin/google-chrome"
```

```json5
// macOS
{
  browser: {
    executablePath: "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser"
  }
}

// Windows
{
  browser: {
    executablePath: "C:\\Program Files\\BraveSoftware\\Brave-Browser\\Application\\brave.exe"
  }
}

// Linux
{
  browser: {
    executablePath: "/usr/bin/brave-browser"
  }
}
```

## ローカル制御とリモート制御

- **ローカル制御（デフォルト）:** Gateway が loopback 制御サービスを起動し、ローカルブラウザを起動できます。
- **リモート制御（node host）:** ブラウザがあるマシン上で node host を実行し、Gateway が browser アクションをその node にプロキシします。
- **リモート CDP:** リモートの Chromium ベースブラウザに接続するには `browser.profiles.<name>.cdpUrl`（または `browser.cdpUrl`）を設定します。この場合、OpenClaw はローカルブラウザを起動しません。

停止動作はプロファイルモードによって異なります:

- ローカル管理プロファイル: `openclaw browser stop` は
  OpenClaw が起動した browser プロセスを停止します
- attach-only およびリモート CDP プロファイル: `openclaw browser stop` は、アクティブな
  制御セッションを閉じ、Playwright/CDP のエミュレーション上書き（viewport、
  color scheme、locale、timezone、offline mode などの状態）を解除します。
  OpenClaw 自体が browser プロセスを起動していない場合でも同様です

リモート CDP URL には auth を含めることができます:

- クエリトークン（例: `https://provider.example?token=<token>`）
- HTTP Basic auth（例: `https://user:pass@provider.example`）

OpenClaw は `/json/*` エンドポイント呼び出し時と
CDP WebSocket 接続時の両方で auth を保持します。
トークンは config ファイルにコミットするのではなく、環境変数やシークレットマネージャーを使うことを推奨します。

## Node browser プロキシ（ゼロ設定デフォルト）

browser があるマシンで **node host** を実行している場合、OpenClaw は
追加の browser 設定なしで browser tool 呼び出しをその node に自動ルーティングできます。
これはリモート Gateway のデフォルト経路です。

注意:

- node host は、そのローカル browser 制御サーバーを **proxy command** として公開します。
- プロファイルは node 側の `browser.profiles` 設定（ローカルと同じ）から取得されます。
- `nodeHost.browserProxy.allowProfiles` は任意です。空のままにすると legacy/デフォルト動作になり、設定済みのすべてのプロファイルがプロキシ経由で到達可能なままです。プロファイル作成/削除ルートも含まれます。
- `nodeHost.browserProxy.allowProfiles` を設定すると、OpenClaw はそれを最小権限境界として扱います: allowlist に含まれるプロファイルだけを対象にでき、永続プロファイル作成/削除ルートはプロキシサーフェス上でブロックされます。
- 不要なら無効化できます:
  - node 側: `nodeHost.browserProxy.enabled=false`
  - gateway 側: `gateway.nodes.browser.mode="off"`

## Browserless（ホスト型リモート CDP）

[Browserless](https://browserless.io) は、
HTTPS および WebSocket 経由で CDP 接続 URL を公開するホスト型 Chromium サービスです。OpenClaw はどちらの形式も使用できますが、
リモート browser プロファイルでは Browserless の接続ドキュメントにある
直接 WebSocket URL が最も簡単です。

例:

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "browserless",
    remoteCdpTimeoutMs: 2000,
    remoteCdpHandshakeTimeoutMs: 4000,
    profiles: {
      browserless: {
        cdpUrl: "wss://production-sfo.browserless.io?token=<BROWSERLESS_API_KEY>",
        color: "#00AA00",
      },
    },
  },
}
```

注意:

- `<BROWSERLESS_API_KEY>` は実際の Browserless token に置き換えてください。
- Browserless アカウントに対応するリージョン endpoint を選んでください（詳細は Browserless のドキュメントを参照）。
- Browserless から HTTPS ベース URL が提供される場合は、それを
  直接 CDP 接続用に `wss://` に変換することも、
  HTTPS URL のまま使って OpenClaw に `/json/version` をディスカバーさせることもできます。

## 直接 WebSocket CDP provider

一部のホスト型 browser サービスは、標準の HTTP ベース CDP ディスカバリー（`/json/version`）ではなく、
**直接 WebSocket** endpoint を公開しています。OpenClaw は両方をサポートします:

- **HTTP(S) エンドポイント** — OpenClaw は `/json/version` を呼び出して
  WebSocket debugger URL を検出してから接続します。
- **WebSocket エンドポイント**（`ws://` / `wss://`）— OpenClaw は `/json/version` をスキップして直接接続します。これは
  [Browserless](https://browserless.io)、
  [Browserbase](https://www.browserbase.com)、または
  WebSocket URL を提供する任意の provider で使用してください。

### Browserbase

[Browserbase](https://www.browserbase.com) は、
組み込み CAPTCHA 解決、stealth mode、および residential
proxy を備えた headless browser 実行用クラウドプラットフォームです。

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "browserbase",
    remoteCdpTimeoutMs: 3000,
    remoteCdpHandshakeTimeoutMs: 5000,
    profiles: {
      browserbase: {
        cdpUrl: "wss://connect.browserbase.com?apiKey=<BROWSERBASE_API_KEY>",
        color: "#F97316",
      },
    },
  },
}
```

注意:

- [Sign up](https://www.browserbase.com/sign-up) して、
  [Overview dashboard](https://www.browserbase.com/overview) から **API Key**
  をコピーしてください。
- `<BROWSERBASE_API_KEY>` は実際の Browserbase API key に置き換えてください。
- Browserbase は WebSocket 接続時に browser session を自動作成するため、
  手動のセッション作成手順は不要です。
- 無料プランでは、同時セッション 1 つ、月あたり browser 1 時間まで利用できます。
  有料プランの上限については [pricing](https://www.browserbase.com/pricing) を参照してください。
- 完全な API
  リファレンス、SDK ガイド、および統合例については [Browserbase docs](https://docs.browserbase.com) を参照してください。

## セキュリティ

重要な考え方:

- browser 制御は loopback 専用です。アクセスは Gateway の auth または node pairing を通じて流れます。
- スタンドアロンの loopback browser HTTP API は、**shared-secret auth のみ**を使用します:
  gateway token bearer auth、`x-openclaw-password`、または
  設定された gateway password を使う HTTP Basic auth です。
- Tailscale Serve の identity headers と `gateway.auth.mode: "trusted-proxy"` は、
  このスタンドアロン loopback browser API の認証には**なりません**。
- browser 制御が有効で、shared-secret auth が設定されていない場合、OpenClaw は
  起動時に `gateway.auth.token` を自動生成し、config に永続化します。
- `gateway.auth.mode` が
  すでに `password`、`none`、または `trusted-proxy` の場合、OpenClaw はその token を自動生成**しません**。
- Gateway とすべての node host はプライベートネットワーク（Tailscale）上に保ち、
  公開インターネットに露出させないでください。
- リモート CDP URL/token はシークレットとして扱い、環境変数やシークレットマネージャーを優先してください。

リモート CDP のヒント:

- 可能であれば暗号化されたエンドポイント（HTTPS または WSS）と短命トークンを使ってください。
- 長期トークンを config ファイルに直接埋め込むのは避けてください。

## プロファイル（マルチブラウザ）

OpenClaw は複数の名前付きプロファイル（ルーティング設定）をサポートします。プロファイルには次の種類があります:

- **openclaw-managed**: 専用の Chromium ベース browser インスタンス。独自の user data directory + CDP port を持ちます
- **remote**: 明示的な CDP URL（別の場所で動作している Chromium ベース browser）
- **existing session**: Chrome DevTools MCP 自動接続を通じた既存の Chrome プロファイル

デフォルト:

- `openclaw` プロファイルは、存在しない場合に自動作成されます。
- `user` プロファイルは、Chrome MCP の existing-session 接続用に組み込まれています。
- existing-session プロファイルは `user` 以外では opt-in です。`--driver existing-session` で作成してください。
- ローカル CDP ポートはデフォルトで **18800–18899** から割り当てられます。
- プロファイルを削除すると、そのローカルデータディレクトリは Trash に移動されます。

すべての制御エンドポイントは `?profile=<name>` を受け付けます。CLI では `--browser-profile` を使います。

## Chrome DevTools MCP 経由の existing-session

OpenClaw は、公式の Chrome DevTools MCP サーバーを通じて、
実行中の Chromium ベース browser プロファイルにも接続できます。これにより、その browser プロファイルですでに
開かれているタブとログイン状態を再利用できます。

公式の背景説明とセットアップ参照:

- [Chrome for Developers: Use Chrome DevTools MCP with your browser session](https://developer.chrome.com/blog/chrome-devtools-mcp-debug-your-browser-session)
- [Chrome DevTools MCP README](https://github.com/ChromeDevTools/chrome-devtools-mcp)

組み込みプロファイル:

- `user`

任意: 名前、色、または browser data directory を変えたい場合は、
独自の custom existing-session プロファイルを作成できます。

デフォルト動作:

- 組み込みの `user` プロファイルは Chrome MCP auto-connect を使い、
  デフォルトのローカル Google Chrome プロファイルを対象にします。

Brave、Edge、Chromium、またはデフォルト以外の Chrome プロファイルには `userDataDir` を使います:

```json5
{
  browser: {
    profiles: {
      brave: {
        driver: "existing-session",
        attachOnly: true,
        userDataDir: "~/Library/Application Support/BraveSoftware/Brave-Browser",
        color: "#FB542B",
      },
    },
  },
}
```

次に、対応する browser 側で次を行います:

1. その browser の inspect page for remote debugging を開きます。
2. remote debugging を有効にします。
3. browser を起動したままにし、OpenClaw が接続するときに接続プロンプトを承認します。

一般的な inspect page:

- Chrome: `chrome://inspect/#remote-debugging`
- Brave: `brave://inspect/#remote-debugging`
- Edge: `edge://inspect/#remote-debugging`

ライブ接続スモークテスト:

```bash
openclaw browser --browser-profile user start
openclaw browser --browser-profile user status
openclaw browser --browser-profile user tabs
openclaw browser --browser-profile user snapshot --format ai
```

成功時の見え方:

- `status` に `driver: existing-session` が表示される
- `status` に `transport: chrome-mcp` が表示される
- `status` に `running: true` が表示される
- `tabs` に、すでに開いている browser タブが一覧表示される
- `snapshot` が、選択されたライブタブから refs を返す

接続がうまくいかない場合の確認事項:

- 対象の Chromium ベース browser がバージョン `144+` である
- その browser の inspect page で remote debugging が有効である
- browser に接続同意プロンプトが表示され、それを承認した
- `openclaw doctor` は古い拡張機能ベースの browser config を移行し、
  デフォルト auto-connect プロファイル用に Chrome がローカルにインストールされているかを確認しますが、
  browser 側の remote debugging を代わりに有効化することはできません

エージェント利用:

- ユーザーのログイン済み browser 状態が必要な場合は `profile="user"` を使います。
- custom existing-session プロファイルを使う場合は、その明示的なプロファイル名を渡してください。
- このモードは、ユーザーがコンピューターの前にいて接続
  プロンプトを承認できる場合にのみ選んでください。
- Gateway または node host は `npx chrome-devtools-mcp@latest --autoConnect` を起動できます

注意:

- この経路は、サインイン済み browser セッション内で
  操作できるため、分離された `openclaw` プロファイルより高リスクです。
- OpenClaw はこの driver では browser を起動せず、
  既存セッションにのみ接続します。
- OpenClaw はここで公式の Chrome DevTools MCP `--autoConnect` フローを使用します。
  `userDataDir` が設定されている場合、OpenClaw はそれを渡して、その明示的な
  Chromium user data directory を対象にします。
- existing-session のスクリーンショットは、snapshot からのページキャプチャと `--ref` 要素
  キャプチャをサポートしますが、CSS `--element` セレクターはサポートしません。
- existing-session のページスクリーンショットは、Playwright なしで Chrome MCP 経由で動作します。
  ref ベースの要素スクリーンショット（`--ref`）もそこでは動作しますが、`--full-page`
  は `--ref` や `--element` と組み合わせられません。
- existing-session のアクションは、依然として managed browser
  経路より制限があります:
  - `click`、`type`、`hover`、`scrollIntoView`、`drag`、`select` には
    CSS セレクターではなく snapshot refs が必要です
  - `click` は左ボタンのみ対応です（ボタン上書きや modifier は不可）
  - `type` は `slowly=true` をサポートしません。`fill` または `press` を使ってください
  - `press` は `delayMs` をサポートしません
  - `hover`、`scrollIntoView`、`drag`、`select`、`fill`、`evaluate` は
    呼び出しごとの timeout 上書きをサポートしません
  - `select` は現在 1 つの値のみサポートします
- existing-session の `wait --url` は、他の browser driver と同様に完全一致、部分一致、および glob パターンをサポートします。`wait --load networkidle` はまだサポートされていません。
- existing-session の upload hook には `ref` または `inputRef` が必要で、
  一度に 1 ファイルのみサポートし、CSS `element` ターゲティングはサポートしません。
- existing-session の dialog hook は timeout 上書きをサポートしません。
- batch
  actions、PDF エクスポート、ダウンロードインターセプト、`responsebody` など、一部の機能は依然として managed browser 経路を必要とします。
- existing-session はホストローカルです。Chrome が別のマシン上、または
  別のネットワーク名前空間にある場合は、代わりに remote CDP または node host を使ってください。

## 分離保証

- **専用 user data dir**: 個人用 browser プロファイルには決して触れません。
- **専用ポート**: 開発ワークフローとの衝突を避けるため `9222` を使いません。
- **決定論的タブ制御**: 「最後のタブ」ではなく `targetId` でタブを対象にします。

## browser 選択

ローカル起動時、OpenClaw は利用可能なもののうち最初のものを選びます:

1. Chrome
2. Brave
3. Edge
4. Chromium
5. Chrome Canary

`browser.executablePath` で上書きできます。

プラットフォーム:

- macOS: `/Applications` と `~/Applications` を確認します。
- Linux: `google-chrome`、`brave`、`microsoft-edge`、`chromium` などを探します。
- Windows: 一般的なインストール場所を確認します。

## 制御 API（任意）

ローカル統合専用として、Gateway は小さな loopback HTTP API を公開します:

- ステータス/開始/停止: `GET /`、`POST /start`、`POST /stop`
- タブ: `GET /tabs`、`POST /tabs/open`、`POST /tabs/focus`、`DELETE /tabs/:targetId`
- スナップショット/スクリーンショット: `GET /snapshot`、`POST /screenshot`
- アクション: `POST /navigate`、`POST /act`
- フック: `POST /hooks/file-chooser`、`POST /hooks/dialog`
- ダウンロード: `POST /download`、`POST /wait/download`
- デバッグ: `GET /console`、`POST /pdf`
- デバッグ: `GET /errors`、`GET /requests`、`POST /trace/start`、`POST /trace/stop`、`POST /highlight`
- ネットワーク: `POST /response/body`
- 状態: `GET /cookies`、`POST /cookies/set`、`POST /cookies/clear`
- 状態: `GET /storage/:kind`、`POST /storage/:kind/set`、`POST /storage/:kind/clear`
- 設定: `POST /set/offline`、`POST /set/headers`、`POST /set/credentials`、`POST /set/geolocation`、`POST /set/media`、`POST /set/timezone`、`POST /set/locale`、`POST /set/device`

すべてのエンドポイントは `?profile=<name>` を受け付けます。

shared-secret gateway auth が設定されている場合、browser HTTP ルートにも auth が必要です:

- `Authorization: Bearer <gateway token>`
- `x-openclaw-password: <gateway password>` またはその password を使う HTTP Basic auth

注意:

- このスタンドアロン loopback browser API は、trusted-proxy や
  Tailscale Serve の identity headers を**利用しません**。
- `gateway.auth.mode` が `none` または `trusted-proxy` の場合、これらの loopback browser
  ルートはそれらの identity-bearing モードを継承しません。loopback 専用に保ってください。

### `/act` のエラー契約

`POST /act` は、ルートレベルの検証および
ポリシー失敗に対して構造化エラーレスポンスを使います:

```json
{ "error": "<message>", "code": "ACT_*" }
```

現在の `code` 値:

- `ACT_KIND_REQUIRED`（HTTP 400）: `kind` が欠落しているか認識されません。
- `ACT_INVALID_REQUEST`（HTTP 400）: アクションペイロードの正規化または検証に失敗しました。
- `ACT_SELECTOR_UNSUPPORTED`（HTTP 400）: 未対応のアクション種別で `selector` が使用されました。
- `ACT_EVALUATE_DISABLED`（HTTP 403）: config により `evaluate`（または `wait --fn`）が無効です。
- `ACT_TARGET_ID_MISMATCH`（HTTP 403）: トップレベルまたは batch の `targetId` がリクエスト対象と競合しています。
- `ACT_EXISTING_SESSION_UNSUPPORTED`（HTTP 501）: existing-session プロファイルではそのアクションはサポートされていません。

その他のランタイム失敗では、依然として `code`
フィールドなしの `{ "error": "<message>" }` が返る場合があります。

### Playwright 要件

一部の機能（navigate/act/AI snapshot/role snapshot、要素スクリーンショット、
PDF）には Playwright が必要です。Playwright がインストールされていない場合、それらのエンドポイントは
明確な 501 エラーを返します。

Playwright なしでも動作するもの:

- ARIA スナップショット
- タブごとの CDP
  WebSocket が利用可能な場合の managed `openclaw` browser 向けページスクリーンショット
- `existing-session` / Chrome MCP プロファイル向けページスクリーンショット
- snapshot 出力からの `existing-session` ref ベーススクリーンショット（`--ref`）

引き続き Playwright が必要なもの:

- `navigate`
- `act`
- AI snapshots / role snapshots
- CSS セレクター要素スクリーンショット（`--element`）
- 完全な browser PDF エクスポート

要素スクリーンショットでは `--full-page` も拒否されます。このルートは `fullPage is
not supported for element screenshots` を返します。

Playwright is not available in this gateway build` と表示された場合は、完全な
Playwright パッケージ（`playwright-core` ではなく）をインストールして Gateway を再起動するか、
browser サポート付きで OpenClaw を再インストールしてください。

#### Docker での Playwright インストール

Gateway が Docker で動作している場合は、`npx playwright` を避けてください（npm override の競合があります）。
代わりにバンドルされた CLI を使います:
__OC_I18N_900012__
browser ダウンロードを永続化するには、`PLAYWRIGHT_BROWSERS_PATH`（たとえば
`/home/node/.cache/ms-playwright`）を設定し、`/home/node` が
`OPENCLAW_HOME_VOLUME` または bind mount によって永続化されていることを確認してください。[Docker](/install/docker) を参照してください。

## 仕組み（内部）

高レベルの流れ:

- 小さな **control server** が HTTP リクエストを受け付けます。
- **CDP** を通じて Chromium ベース browser（Chrome/Brave/Edge/Chromium）に接続します。
- 高度なアクション（クリック/入力/スナップショット/PDF）には、
  CDP 上で **Playwright** を使います。
- Playwright がない場合は、Playwright 非依存の操作のみ利用できます。

この設計により、エージェントは安定した決定論的インターフェース上に保たれつつ、
ローカル/リモート browser やプロファイルを切り替えられます。

## CLI クイックリファレンス

すべてのコマンドは、特定プロファイルを対象にするため `--browser-profile <name>` を受け付けます。
すべてのコマンドは、機械可読な出力（安定したペイロード）のため `--json` も受け付けます。

基本:

- `openclaw browser status`
- `openclaw browser start`
- `openclaw browser stop`
- `openclaw browser tabs`
- `openclaw browser tab`
- `openclaw browser tab new`
- `openclaw browser tab select 2`
- `openclaw browser tab close 2`
- `openclaw browser open https://example.com`
- `openclaw browser focus abcd1234`
- `openclaw browser close abcd1234`

調査:

- `openclaw browser screenshot`
- `openclaw browser screenshot --full-page`
- `openclaw browser screenshot --ref 12`
- `openclaw browser screenshot --ref e12`
- `openclaw browser snapshot`
- `openclaw browser snapshot --format aria --limit 200`
- `openclaw browser snapshot --interactive --compact --depth 6`
- `openclaw browser snapshot --efficient`
- `openclaw browser snapshot --labels`
- `openclaw browser snapshot --selector "#main" --interactive`
- `openclaw browser snapshot --frame "iframe#main" --interactive`
- `openclaw browser console --level error`

ライフサイクルに関する注意:

- attach-only およびリモート CDP プロファイルでは、テスト後のクリーンアップコマンドとしても
  `openclaw browser stop` が正しい選択です。これは
  基盤となる browser を終了するのではなく、アクティブな制御セッションを閉じて
  一時的なエミュレーション上書きを解除します。
- `openclaw browser errors --clear`
- `openclaw browser requests --filter api --clear`
- `openclaw browser pdf`
- `openclaw browser responsebody "**/api" --max-chars 5000`

アクション:

- `openclaw browser navigate https://example.com`
- `openclaw browser resize 1280 720`
- `openclaw browser click 12 --double`
- `openclaw browser click e12 --double`
- `openclaw browser type 23 "hello" --submit`
- `openclaw browser press Enter`
- `openclaw browser hover 44`
- `openclaw browser scrollintoview e12`
- `openclaw browser drag 10 11`
- `openclaw browser select 9 OptionA OptionB`
- `openclaw browser download e12 report.pdf`
- `openclaw browser waitfordownload report.pdf`
- `openclaw browser upload /tmp/openclaw/uploads/file.pdf`
- `openclaw browser fill --fields '[{"ref":"1","type":"text","value":"Ada"}]'`
- `openclaw browser dialog --accept`
- `openclaw browser wait --text "Done"`
- `openclaw browser wait "#main" --url "**/dash" --load networkidle --fn "window.ready===true"`
- `openclaw browser evaluate --fn '(el) => el.textContent' --ref 7`
- `openclaw browser highlight e12`
- `openclaw browser trace start`
- `openclaw browser trace stop`

状態:

- `openclaw browser cookies`
- `openclaw browser cookies set session abc123 --url "https://example.com"`
- `openclaw browser cookies clear`
- `openclaw browser storage local get`
- `openclaw browser storage local set theme dark`
- `openclaw browser storage session clear`
- `openclaw browser set offline on`
- `openclaw browser set headers --headers-json '{"X-Debug":"1"}'`
- `openclaw browser set credentials user pass`
- `openclaw browser set credentials --clear`
- `openclaw browser set geo 37.7749 -122.4194 --origin "https://example.com"`
- `openclaw browser set geo --clear`
- `openclaw browser set media dark`
- `openclaw browser set timezone America/New_York`
- `openclaw browser set locale en-US`
- `openclaw browser set device "iPhone 14"`

注意:

- `upload` と `dialog` は **準備用** 呼び出しです。ファイル選択やダイアログを引き起こす click/press
  の前に実行してください。
- ダウンロードおよび trace の出力パスは OpenClaw の temp ルートに制限されます:
  - traces: `/tmp/openclaw`（フォールバック: `${os.tmpdir()}/openclaw`）
  - downloads: `/tmp/openclaw/downloads`（フォールバック: `${os.tmpdir()}/openclaw/downloads`）
- upload パスも OpenClaw の temp uploads ルートに制限されます:
  - uploads: `/tmp/openclaw/uploads`（フォールバック: `${os.tmpdir()}/openclaw/uploads`）
- `upload` は `--input-ref` または `--element` によってファイル入力を直接設定することもできます。
- `snapshot`:
  - `--format ai`（Playwright がインストールされている場合のデフォルト）: 数値 ref（`aria-ref="<n>"`）を含む AI スナップショットを返します。
  - `--format aria`: アクセシビリティツリーを返します（ref なし。調査専用）。
  - `--efficient`（または `--mode efficient`）: コンパクトな role snapshot プリセット（interactive + compact + depth + 低い maxChars）。
  - config デフォルト（tool/CLI のみ）: 呼び出し側が mode を渡さない場合に efficient snapshot を使うには `browser.snapshotDefaults.mode: "efficient"` を設定します（[Gateway configuration](/gateway/configuration-reference#browser) を参照）。
  - Role snapshot オプション（`--interactive`、`--compact`、`--depth`、`--selector`）は、`ref=e12` のような ref を持つ role ベースの snapshot を強制します。
  - `--frame "<iframe selector>"` は、role snapshot を iframe にスコープします（`e12` のような role ref と組み合わせます）。
  - `--interactive` は、対話可能要素の平坦で選びやすい一覧を出力します（アクション操作に最適）。
  - `--labels` は、重ね合わせた ref ラベル付きの viewport 限定スクリーンショットを追加します（`MEDIA:<path>` を出力）。

- `click`/`type` などには `snapshot` から取得した `ref`（数値 `12` または role ref `e12`）が必要です。
  アクションでは CSS セレクターは意図的にサポートされていません。

## スナップショットと ref

OpenClaw は 2 種類の「snapshot」スタイルをサポートします:

- **AI snapshot（数値 ref）**: `openclaw browser snapshot`（デフォルト。`--format ai`）
  - 出力: 数値 ref を含むテキスト snapshot。
  - アクション: `openclaw browser click 12`、`openclaw browser type 23 "hello"`。
  - 内部的には、ref は Playwright の `aria-ref` で解決されます。

- **Role snapshot（`e12` のような role ref）**: `openclaw browser snapshot --interactive`（または `--compact`、`--depth`、`--selector`、`--frame`）
  - 出力: `[ref=e12]`（および任意で `[nth=1]`）を含む role ベースのリスト/ツリー。
  - アクション: `openclaw browser click e12`、`openclaw browser highlight e12`。
  - 内部的には、ref は `getByRole(...)`（重複時は `nth()` を追加）で解決されます。
  - `--labels` を追加すると、重ね合わせた `e12` ラベル付き viewport スクリーンショットを含められます。

ref の挙動:

- ref は**ナビゲーションをまたいで安定しません**。何か失敗したら、`snapshot` を再実行して新しい ref を使ってください。
- role snapshot を `--frame` 付きで取得した場合、role ref は次の role snapshot までその iframe にスコープされます。

## wait の強化機能

時間やテキスト以外も待機できます:

- URL を待つ（Playwright 対応の glob をサポート）:
  - `openclaw browser wait --url "**/dash"`
- ロード状態を待つ:
  - `openclaw browser wait --load networkidle`
- JS predicate を待つ:
  - `openclaw browser wait --fn "window.ready===true"`
- セレクターが visible になるのを待つ:
  - `openclaw browser wait "#main"`

これらは組み合わせられます:
__OC_I18N_900013__
## デバッグワークフロー

アクションが失敗したとき（たとえば「not visible」「strict mode violation」「covered」）:

1. `openclaw browser snapshot --interactive`
2. `click <ref>` / `type <ref>` を使う（interactive mode では role ref を推奨）
3. まだ失敗する場合: `openclaw browser highlight <ref>` で Playwright が何を対象にしているか確認する
4. ページ挙動がおかしい場合:
   - `openclaw browser errors --clear`
   - `openclaw browser requests --filter api --clear`
5. 深いデバッグには trace を記録する:
   - `openclaw browser trace start`
   - 問題を再現する
   - `openclaw browser trace stop`（`TRACE:<path>` を出力）

## JSON 出力

`--json` はスクリプト処理と構造化ツール用です。

例:
__OC_I18N_900014__
JSON の role snapshot には `refs` に加えて小さな `stats` ブロック（lines/chars/refs/interactive）が含まれるため、ツール側でペイロードサイズと密度を判断できます。

## 状態と環境の調整項目

これらは「サイトを X のように振る舞わせる」ワークフローで役立ちます:

- Cookies: `cookies`、`cookies set`、`cookies clear`
- Storage: `storage local|session get|set|clear`
- Offline: `set offline on|off`
- Headers: `set headers --headers-json '{"X-Debug":"1"}'`（legacy の `set headers --json '{"X-Debug":"1"}'` も引き続きサポート）
- HTTP basic auth: `set credentials user pass`（または `--clear`）
- Geolocation: `set geo <lat> <lon> --origin "https://example.com"`（または `--clear`）
- Media: `set media dark|light|no-preference|none`
- Timezone / locale: `set timezone ...`、`set locale ...`
- Device / viewport:
  - `set device "iPhone 14"`（Playwright device preset）
  - `set viewport 1280 720`

## セキュリティとプライバシー

- openclaw browser プロファイルにはログイン済みセッションが含まれる場合があるため、機密として扱ってください。
- `browser act kind=evaluate` / `openclaw browser evaluate` と `wait --fn`
  は、ページコンテキストで任意の JavaScript を実行します。これは prompt injection によって誘導される可能性があります。
  必要ない場合は `browser.evaluateEnabled=false` で無効にしてください。
- ログインや anti-bot に関する注意（X/Twitter など）については、[Browser login + X/Twitter posting](/tools/browser-login) を参照してください。
- Gateway/node host はプライベート（loopback または tailnet のみ）に保ってください。
- リモート CDP エンドポイントは強力です。トンネリングし、保護してください。

strict-mode の例（デフォルトでは private/internal 宛先をブロック）:
__OC_I18N_900015__
## トラブルシューティング

Linux 固有の問題（特に snap Chromium）については、
[Browser troubleshooting](/tools/browser-linux-troubleshooting) を参照してください。

WSL2 Gateway + Windows Chrome の分離ホスト構成については、
[WSL2 + Windows + remote Chrome CDP troubleshooting](/tools/browser-wsl2-windows-remote-cdp-troubleshooting) を参照してください。

## エージェント tool と制御の仕組み

エージェントが受け取る browser 自動化用 tool は**1つだけ**です:

- `browser` — status/start/stop/tabs/open/focus/close/snapshot/screenshot/navigate/act

対応関係:

- `browser snapshot` は安定した UI ツリー（AI または ARIA）を返します。
- `browser act` は snapshot の `ref` ID を使って click/type/drag/select を行います。
- `browser screenshot` はピクセルをキャプチャします（full page または要素）。
- `browser` は次を受け付けます:
  - 名前付き browser プロファイル（openclaw、chrome、または remote CDP）を選ぶ `profile`。
  - browser の所在を選ぶ `target`（`sandbox` | `host` | `node`）。
  - サンドボックス化されたセッションでは、`target: "host"` に `agents.defaults.sandbox.browser.allowHostControl=true` が必要です。
  - `target` を省略した場合: サンドボックス化セッションではデフォルトが `sandbox`、非サンドボックスセッションではデフォルトが `host` です。
  - browser 対応 node が接続されている場合、`target="host"` または `target="node"` を固定しない限り、tool は自動的にそこへルーティングされることがあります。

これにより、エージェントは決定論的に保たれ、壊れやすいセレクターを避けられます。

## 関連

- [Tools Overview](/ja-JP/tools) — 利用可能なすべてのエージェント tool
- [Sandboxing](/ja-JP/gateway/sandboxing) — サンドボックス環境での browser 制御
- [Security](/ja-JP/gateway/security) — browser 制御のリスクとハードニング
