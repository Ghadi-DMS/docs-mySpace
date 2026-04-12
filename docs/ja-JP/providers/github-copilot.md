---
read_when:
    - モデル provider として GitHub Copilot を使いたい場合
    - '`openclaw models auth login-github-copilot` フローが必要な場合'
summary: デバイスフローを使って OpenClaw から GitHub Copilot にサインインする
title: GitHub Copilot
x-i18n:
    generated_at: "2026-04-12T23:31:12Z"
    model: gpt-5.4
    provider: openai
    source_hash: 51fee006e7d4e78e37b0c29356b0090b132de727d99b603441767d3fb642140b
    source_path: providers/github-copilot.md
    workflow: 15
---

# GitHub Copilot

GitHub Copilot は GitHub の AI コーディングアシスタントです。GitHub アカウントとプランに対して Copilot モデルへのアクセスを提供します。OpenClaw では、Copilot をモデル provider として 2 つの異なる方法で使用できます。

## OpenClaw で Copilot を使う 2 つの方法

<Tabs>
  <Tab title="組み込み provider (`github-copilot`)">
    ネイティブのデバイスログインフローを使って GitHub トークンを取得し、その後 OpenClaw 実行時にそれを Copilot API トークンへ交換します。これは **デフォルト** かつ最も簡単な方法で、VS Code を必要としません。

    <Steps>
      <Step title="ログインコマンドを実行する">
        ```bash
        openclaw models auth login-github-copilot
        ```

        URL にアクセスしてワンタイムコードを入力するよう求められます。完了するまで
        ターミナルを開いたままにしてください。
      </Step>
      <Step title="デフォルトモデルを設定する">
        ```bash
        openclaw models set github-copilot/gpt-4o
        ```

        または設定で:

        ```json5
        {
          agents: { defaults: { model: { primary: "github-copilot/gpt-4o" } } },
        }
        ```
      </Step>
    </Steps>

  </Tab>

  <Tab title="Copilot Proxy Plugin (`copilot-proxy`)">
    **Copilot Proxy** VS Code 拡張機能をローカルブリッジとして使用します。OpenClaw は
    プロキシの `/v1` エンドポイントと通信し、そこで設定したモデル一覧を使用します。

    <Note>
    すでに VS Code で Copilot Proxy を実行している場合、またはそれ経由でルーティングする必要がある場合は、こちらを選んでください。Plugin を有効にし、VS Code 拡張機能を実行し続ける必要があります。
    </Note>

  </Tab>
</Tabs>

## 任意のフラグ

| フラグ | 説明 |
| --------------- | --------------------------------------------------- |
| `--yes` | 確認プロンプトをスキップします |
| `--set-default` | provider の推奨デフォルトモデルも適用します |

```bash
# 確認をスキップ
openclaw models auth login-github-copilot --yes

# ログインしてデフォルトモデルも 1 ステップで設定
openclaw models auth login --provider github-copilot --method device --set-default
```

<AccordionGroup>
  <Accordion title="対話型 TTY が必要">
    デバイスログインフローには対話型 TTY が必要です。非対話スクリプトや CI
    パイプラインではなく、ターミナルで直接実行してください。
  </Accordion>

  <Accordion title="モデルの利用可否はプランに依存">
    Copilot のモデル利用可否は GitHub プランに依存します。モデルが
    拒否された場合は、別の ID（たとえば `github-copilot/gpt-4.1`）を試してください。
  </Accordion>

  <Accordion title="トランスポートの選択">
    Claude モデル ID は自動的に Anthropic Messages トランスポートを使用します。GPT、
    o-series、および Gemini モデルは OpenAI Responses トランスポートを維持します。OpenClaw
    はモデル参照に基づいて正しいトランスポートを選択します。
  </Accordion>

  <Accordion title="環境変数の解決順序">
    OpenClaw は、次の優先順位で環境変数から Copilot 認証を解決します。

    | 優先順位 | 変数 | 注記 |
    | -------- | --------------------- | -------------------------------- |
    | 1 | `COPILOT_GITHUB_TOKEN` | 最優先、Copilot 専用 |
    | 2 | `GH_TOKEN` | GitHub CLI トークン（フォールバック） |
    | 3 | `GITHUB_TOKEN` | 標準 GitHub トークン（最低優先） |

    複数の変数が設定されている場合、OpenClaw は最優先のものを使用します。
    デバイスログインフロー（`openclaw models auth login-github-copilot`）は
    auth プロファイルストアにトークンを保存し、すべての環境変数よりも優先されます。

  </Accordion>

  <Accordion title="トークン保存">
    このログインでは GitHub トークンを auth プロファイルストアに保存し、OpenClaw 実行時にそれを
    Copilot API トークンへ交換します。トークンを手動で管理する必要はありません。
  </Accordion>
</AccordionGroup>

<Warning>
対話型 TTY が必要です。ログインコマンドはヘッドレススクリプトや CI ジョブの中ではなく、
ターミナルで直接実行してください。
</Warning>

## 関連

<CardGroup cols={2}>
  <Card title="Model selection" href="/ja-JP/concepts/model-providers" icon="layers">
    provider、モデル参照、フェイルオーバー動作の選び方。
  </Card>
  <Card title="OAuth and auth" href="/ja-JP/gateway/authentication" icon="key">
    認証の詳細と資格情報再利用ルール。
  </Card>
</CardGroup>
