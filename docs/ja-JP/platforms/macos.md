---
read_when:
    - macOSアプリ機能の実装
    - macOSでのGatewayライフサイクルまたはNodeブリッジの変更
summary: OpenClaw macOSコンパニオンアプリ（メニューバー + Gatewayブローカー）
title: macOSアプリ
x-i18n:
    generated_at: "2026-04-18T04:40:13Z"
    model: gpt-5.4
    provider: openai
    source_hash: d637df2f73ced110223c48ea3c934045d782e150a46495f434cf924a6a00baf0
    source_path: platforms/macos.md
    workflow: 15
---

# OpenClaw macOS Companion（メニューバー + Gatewayブローカー）

macOSアプリは、OpenClawの**メニューバーコンパニオン**です。権限を管理し、ローカルでGatewayを管理または接続し（launchdまたは手動）、macOSの機能をノードとしてエージェントに公開します。

## できること

- ネイティブ通知とステータスをメニューバーに表示します。
- TCCプロンプト（通知、アクセシビリティ、画面収録、マイク、音声認識、Automation/AppleScript）を管理します。
- Gatewayを実行または接続します（ローカルまたはリモート）。
- macOS専用ツール（Canvas、Camera、Screen Recording、`system.run`）を公開します。
- **remote**モードではローカルのノードホストサービスを起動し（launchd）、**local**モードでは停止します。
- 必要に応じて、UIオートメーション用の**PeekabooBridge**をホストします。
- リクエストに応じて、グローバルCLI（`openclaw`）をnpm、pnpm、またはbun経由でインストールします（アプリはnpm、次にpnpm、最後にbunを優先します。Nodeは引き続き推奨されるGatewayランタイムです）。

## localモードとremoteモード

- **Local**（デフォルト）: 実行中のローカルGatewayがあればアプリはそれに接続します。ない場合は、`openclaw gateway install` を使ってlaunchdサービスを有効にします。
- **Remote**: アプリはSSH/Tailscale経由でGatewayに接続し、ローカルプロセスを起動しません。
  アプリはローカルの**ノードホストサービス**を起動して、リモートGatewayがこのMacに到達できるようにします。
  アプリはGatewayを子プロセスとして起動しません。
  Gateway検出は現在、生のtailnet IPよりもTailscale MagicDNS名を優先するため、tailnet IPが変わった場合でもMacアプリはより確実に復旧できます。

## Launchd制御

アプリは、ユーザーごとのLaunchAgent `ai.openclaw.gateway` を管理します（`--profile`/`OPENCLAW_PROFILE`使用時は `ai.openclaw.<profile>`、従来の `com.openclaw.*` も引き続きアンロードされます）。

```bash
launchctl kickstart -k gui/$UID/ai.openclaw.gateway
launchctl bootout gui/$UID/ai.openclaw.gateway
```

名前付きプロファイルを実行する場合は、ラベルを `ai.openclaw.<profile>` に置き換えてください。

LaunchAgentがインストールされていない場合は、アプリから有効にするか、`openclaw gateway install` を実行してください。

## Nodeの機能（mac）

macOSアプリは自身をNodeとして提示します。よく使うコマンド:

- Canvas: `canvas.present`, `canvas.navigate`, `canvas.eval`, `canvas.snapshot`, `canvas.a2ui.*`
- Camera: `camera.snap`, `camera.clip`
- Screen: `screen.snapshot`, `screen.record`
- System: `system.run`, `system.notify`

Nodeは、エージェントが何を許可するか判断できるように `permissions` マップを報告します。

Nodeサービス + アプリIPC:

- ヘッドレスのノードホストサービスが実行中の場合（remoteモード）、NodeとしてGateway WSに接続します。
- `system.run` はローカルUnixソケット経由でmacOSアプリ内（UI/TCCコンテキスト）で実行されます。プロンプトと出力はアプリ内に留まります。

図（SCI）:

```
Gateway -> Node Service (WS)
                 |  IPC (UDS + token + HMAC + TTL)
                 v
             Mac App (UI + TCC + system.run)
```

## 実行承認（system.run）

`system.run` は、macOSアプリ内の**Exec approvals**（Settings → Exec approvals）で制御されます。
セキュリティ + 確認 + 許可リストは、Mac上の以下にローカル保存されます:

```
~/.openclaw/exec-approvals.json
```

例:

```json
{
  "version": 1,
  "defaults": {
    "security": "deny",
    "ask": "on-miss"
  },
  "agents": {
    "main": {
      "security": "allowlist",
      "ask": "on-miss",
      "allowlist": [{ "pattern": "/opt/homebrew/bin/rg" }]
    }
  }
}
```

注意:

- `allowlist` エントリーは、解決済みバイナリパスに対するglobパターンです。
- シェル制御または展開構文（`&&`, `||`, `;`, `|`, `` ` ``, `$`, `<`, `>`, `(`, `)`）を含む生のシェルコマンドテキストは、allowlistミスとして扱われ、明示的な承認が必要です（またはシェルバイナリをallowlistに追加する必要があります）。
- プロンプトで「Always Allow」を選ぶと、そのコマンドがallowlistに追加されます。
- `system.run` の環境変数オーバーライドはフィルタリングされ（`PATH`, `DYLD_*`, `LD_*`, `NODE_OPTIONS`, `PYTHON*`, `PERL*`, `RUBYOPT`, `SHELLOPTS`, `PS4` を除外）、その後アプリの環境変数とマージされます。
- シェルラッパー（`bash|sh|zsh ... -c/-lc`）では、リクエスト単位の環境変数オーバーライドは、小さな明示的allowlist（`TERM`, `LANG`, `LC_*`, `COLORTERM`, `NO_COLOR`, `FORCE_COLOR`）に縮小されます。
- allowlistモードで「常に許可」を選択した場合、既知のディスパッチラッパー（`env`, `nice`, `nohup`, `stdbuf`, `timeout`）では、ラッパーパスではなく内部の実行ファイルパスが保存されます。安全にアンラップできない場合、allowlistエントリーは自動保存されません。

## ディープリンク

アプリはローカルアクション用に `openclaw://` URLスキームを登録します。

### `openclaw://agent`

Gatewayの `agent` リクエストをトリガーします。
__OC_I18N_900004__
クエリパラメータ:

- `message`（必須）
- `sessionKey`（任意）
- `thinking`（任意）
- `deliver` / `to` / `channel`（任意）
- `timeoutSeconds`（任意）
- `key`（任意の無人モードキー）

安全性:

- `key` がない場合、アプリは確認を求めます。
- `key` がない場合、アプリは確認プロンプト用に短いメッセージ長制限を適用し、`deliver` / `to` / `channel` を無視します。
- 有効な `key` がある場合、実行は無人で行われます（個人用オートメーション向け）。

## オンボーディングフロー（一般的な流れ）

1. **OpenClaw.app**をインストールして起動します。
2. 権限チェックリスト（TCCプロンプト）を完了します。
3. **Local**モードが有効で、Gatewayが実行中であることを確認します。
4. ターミナルアクセスが必要ならCLIをインストールします。

## 状態ディレクトリの配置（macOS）

OpenClawの状態ディレクトリは、iCloudやその他のクラウド同期フォルダに置かないでください。
同期対応パスでは遅延が増える可能性があり、セッションや認証情報でファイルロックや同期競合が発生することがあります。

次のような、同期されないローカル状態パスを推奨します:
__OC_I18N_900005__
`openclaw doctor` が以下の場所に状態を検出した場合:

- `~/Library/Mobile Documents/com~apple~CloudDocs/...`
- `~/Library/CloudStorage/...`

警告を表示し、ローカルパスへの移動を推奨します。

## ビルドと開発ワークフロー（ネイティブ）

- `cd apps/macos && swift build`
- `swift run OpenClaw`（またはXcode）
- アプリをパッケージ化: `scripts/package-mac-app.sh`

## Gateway接続のデバッグ（macOS CLI）

アプリを起動せずに、macOSアプリと同じGateway WebSocketハンドシェイクと検出ロジックをデバッグCLIで試せます。
__OC_I18N_900006__
接続オプション:

- `--url <ws://host:port>`: 設定を上書き
- `--mode <local|remote>`: 設定から解決（デフォルト: 設定またはlocal）
- `--probe`: 新しいヘルスプローブを強制
- `--timeout <ms>`: リクエストタイムアウト（デフォルト: `15000`）
- `--json`: diff比較用の構造化出力

検出オプション:

- `--include-local`: 「local」として除外されるGatewayも含める
- `--timeout <ms>`: 全体の検出ウィンドウ（デフォルト: `2000`）
- `--json`: diff比較用の構造化出力

ヒント: `openclaw gateway discover --json` と比較して、macOSアプリの検出パイプライン（`local.` に加えて、設定済みの広域ドメイン、そして広域およびTailscale Serveのフォールバック）が、Node CLIの `dns-sd` ベースの検出と異なるか確認してください。

## リモート接続の配管（SSHトンネル）

macOSアプリが**Remote**モードで実行されると、ローカルUIコンポーネントがリモートGatewayをlocalhost上にあるかのように扱えるよう、SSHトンネルを開きます。

### 制御トンネル（Gateway WebSocketポート）

- **目的:** ヘルスチェック、ステータス、Web Chat、設定、その他のコントロールプレーン呼び出し。
- **ローカルポート:** Gatewayポート（デフォルト `18789`）、常に固定。
- **リモートポート:** リモートホスト上の同じGatewayポート。
- **動作:** ランダムなローカルポートは使いません。アプリは既存の正常なトンネルを再利用し、必要であれば再起動します。
- **SSHの形:** `ssh -N -L <local>:127.0.0.1:<remote>` に、BatchMode + ExitOnForwardFailure + keepaliveオプションを付加。
- **IPレポート:** SSHトンネルはloopbackを使うため、gatewayから見るノードIPは `127.0.0.1` になります。実際のクライアントIPを表示したい場合は、**Direct (ws/wss)** トランスポートを使用してください（[macOS remote access](/ja-JP/platforms/mac/remote)を参照）。

セットアップ手順については、[macOS remote access](/ja-JP/platforms/mac/remote)を参照してください。プロトコルの詳細については、[Gateway protocol](/ja-JP/gateway/protocol)を参照してください。

## 関連ドキュメント

- [Gateway runbook](/ja-JP/gateway)
- [Gateway（macOS）](/ja-JP/platforms/mac/bundled-gateway)
- [macOS permissions](/ja-JP/platforms/mac/permissions)
- [Canvas](/ja-JP/platforms/mac/canvas)
