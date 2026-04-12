---
read_when:
    - 返信向けに text-to-speech を有効にする
    - TTS provider または制限を設定するრებაassistant to=final
    - '`/tts` コマンドの使用'
summary: 送信返信向けの text-to-speech（TTS）
title: Text-to-Speech
x-i18n:
    generated_at: "2026-04-12T23:34:35Z"
    model: gpt-5.4
    provider: openai
    source_hash: ad79a6be34879347dc73fdab1bd219823cd7c6aa8504e3e4c73e1a0554c837c5
    source_path: tools/tts.md
    workflow: 15
---

# Text-to-speech（TTS）

OpenClaw は、ElevenLabs、Microsoft、MiniMax、または OpenAI を使用して送信返信を音声に変換できます。
これは、OpenClaw が音声を送信できる場所ならどこでも動作します。

## サポートされるサービス

- **ElevenLabs**（primary または fallback provider）
- **Microsoft**（primary または fallback provider。現在のバンドル実装は `node-edge-tts` を使用）
- **MiniMax**（primary または fallback provider。T2A v2 API を使用）
- **OpenAI**（primary または fallback provider。summary にも使用）

### Microsoft speech の注意事項

バンドルされた Microsoft speech provider は現在、`node-edge-tts` ライブラリを通じて Microsoft Edge のオンライン neural TTS サービスを使用します。これはホスト型サービスであり（ローカルではありません）、Microsoft のエンドポイントを使用し、API キーは不要です。
`node-edge-tts` は speech の設定オプションと出力形式を公開していますが、すべてのオプションがサービスでサポートされているわけではありません。`edge` を使う従来の config や directive 入力は引き続き動作し、`microsoft` に正規化されます。

この経路は、公開された SLA やクォータのない公開 Web サービスであるため、best-effort として扱ってください。保証された制限やサポートが必要な場合は、OpenAI または ElevenLabs を使用してください。

## 任意のキー

OpenAI、ElevenLabs、または MiniMax を使用する場合:

- `ELEVENLABS_API_KEY`（または `XI_API_KEY`）
- `MINIMAX_API_KEY`
- `OPENAI_API_KEY`

Microsoft speech には API キーは**不要**です。

複数の provider が設定されている場合、選択された provider が最初に使用され、他は fallback option になります。
自動 summary は設定された `summaryModel`（または `agents.defaults.model.primary`）を使用するため、summary を有効にする場合は、その provider も認証済みである必要があります。

## サービスリンク

- [OpenAI Text-to-Speech guide](https://platform.openai.com/docs/guides/text-to-speech)
- [OpenAI Audio API reference](https://platform.openai.com/docs/api-reference/audio)
- [ElevenLabs Text to Speech](https://elevenlabs.io/docs/api-reference/text-to-speech)
- [ElevenLabs Authentication](https://elevenlabs.io/docs/api-reference/authentication)
- [MiniMax T2A v2 API](https://platform.minimaxi.com/document/T2A%20V2)
- [node-edge-tts](https://github.com/SchneeHertz/node-edge-tts)
- [Microsoft Speech output formats](https://learn.microsoft.com/azure/ai-services/speech-service/rest-text-to-speech#audio-outputs)

## デフォルトで有効ですか？

いいえ。自動 TTS はデフォルトで **off** です。config の
`messages.tts.auto` で有効にするか、`/tts on` でローカルに有効化してください。

`messages.tts.provider` が未設定の場合、OpenClaw は registry の auto-select 順序で最初に設定された
speech provider を選びます。

## 設定

TTS の設定は `openclaw.json` の `messages.tts` 配下にあります。
完全な schema は [Gateway configuration](/ja-JP/gateway/configuration) にあります。

### 最小設定（有効化 + provider）

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "elevenlabs",
    },
  },
}
```

### OpenAI を primary、ElevenLabs を fallback にする

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "openai",
      summaryModel: "openai/gpt-4.1-mini",
      modelOverrides: {
        enabled: true,
      },
      providers: {
        openai: {
          apiKey: "openai_api_key",
          baseUrl: "https://api.openai.com/v1",
          model: "gpt-4o-mini-tts",
          voice: "alloy",
        },
        elevenlabs: {
          apiKey: "elevenlabs_api_key",
          baseUrl: "https://api.elevenlabs.io",
          voiceId: "voice_id",
          modelId: "eleven_multilingual_v2",
          seed: 42,
          applyTextNormalization: "auto",
          languageCode: "en",
          voiceSettings: {
            stability: 0.5,
            similarityBoost: 0.75,
            style: 0.0,
            useSpeakerBoost: true,
            speed: 1.0,
          },
        },
      },
    },
  },
}
```

### Microsoft を primary にする（API キー不要）

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "microsoft",
      providers: {
        microsoft: {
          enabled: true,
          voice: "en-US-MichelleNeural",
          lang: "en-US",
          outputFormat: "audio-24khz-48kbitrate-mono-mp3",
          rate: "+10%",
          pitch: "-5%",
        },
      },
    },
  },
}
```

### MiniMax を primary にする

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "minimax",
      providers: {
        minimax: {
          apiKey: "minimax_api_key",
          baseUrl: "https://api.minimax.io",
          model: "speech-2.8-hd",
          voiceId: "English_expressive_narrator",
          speed: 1.0,
          vol: 1.0,
          pitch: 0,
        },
      },
    },
  },
}
```

### Microsoft speech を無効にする

```json5
{
  messages: {
    tts: {
      providers: {
        microsoft: {
          enabled: false,
        },
      },
    },
  },
}
```

### カスタム制限 + prefs path

```json5
{
  messages: {
    tts: {
      auto: "always",
      maxTextLength: 4000,
      timeoutMs: 30000,
      prefsPath: "~/.openclaw/settings/tts.json",
    },
  },
}
```

### 受信音声メッセージの後だけ音声で返信する

```json5
{
  messages: {
    tts: {
      auto: "inbound",
    },
  },
}
```

### 長い返信の自動 summary を無効にする

```json5
{
  messages: {
    tts: {
      auto: "always",
    },
  },
}
```

その後、次を実行します:

```
/tts summary off
```

### フィールドに関する注意

- `auto`: 自動 TTS モード（`off`、`always`、`inbound`、`tagged`）。
  - `inbound` は、受信音声メッセージの後にのみ音声を送信します。
  - `tagged` は、返信に `[[tts:key=value]]` ディレクティブまたは `[[tts:text]]...[[/tts:text]]` ブロックが含まれる場合にのみ音声を送信します。
- `enabled`: 従来の切り替え項目（doctor はこれを `auto` に移行します）。
- `mode`: `"final"`（デフォルト）または `"all"`（tool/block 返信を含む）。
- `provider`: `"elevenlabs"`、`"microsoft"`、`"minimax"`、`"openai"` のような speech provider ID（fallback は自動）。
- `provider` が**未設定**の場合、OpenClaw は registry の auto-select 順序で最初に設定された speech provider を使用します。
- 従来の `provider: "edge"` は引き続き動作し、`microsoft` に正規化されます。
- `summaryModel`: 自動 summary 用の任意の安価なモデル。デフォルトは `agents.defaults.model.primary` です。
  - `provider/model` または設定済み model alias を受け付けます。
- `modelOverrides`: モデルが TTS ディレクティブを出力できるようにします（デフォルトで有効）。
  - `allowProvider` のデフォルトは `false` です（provider 切り替えはオプトイン）。
- `providers.<id>`: speech provider ID をキーにした provider 所有設定。
- 従来の直接 provider ブロック（`messages.tts.openai`、`messages.tts.elevenlabs`、`messages.tts.microsoft`、`messages.tts.edge`）は、読み込み時に自動で `messages.tts.providers.<id>` に移行されます。
- `maxTextLength`: TTS 入力のハード上限（文字数）。これを超えると `/tts audio` は失敗します。
- `timeoutMs`: リクエストタイムアウト（ms）。
- `prefsPath`: ローカルの prefs JSON path を上書きします（provider/limit/summary）。
- `apiKey` の値は env var（`ELEVENLABS_API_KEY`/`XI_API_KEY`、`MINIMAX_API_KEY`、`OPENAI_API_KEY`）にフォールバックします。
- `providers.elevenlabs.baseUrl`: ElevenLabs API base URL を上書きします。
- `providers.openai.baseUrl`: OpenAI TTS エンドポイントを上書きします。
  - 解決順序: `messages.tts.providers.openai.baseUrl` -> `OPENAI_TTS_BASE_URL` -> `https://api.openai.com/v1`
  - デフォルト以外の値は OpenAI 互換 TTS エンドポイントとして扱われるため、カスタム model 名と voice 名を使用できます。
- `providers.elevenlabs.voiceSettings`:
  - `stability`、`similarityBoost`、`style`: `0..1`
  - `useSpeakerBoost`: `true|false`
  - `speed`: `0.5..2.0`（1.0 = 通常）
- `providers.elevenlabs.applyTextNormalization`: `auto|on|off`
- `providers.elevenlabs.languageCode`: 2 文字の ISO 639-1（例: `en`、`de`）
- `providers.elevenlabs.seed`: 整数 `0..4294967295`（best-effort の決定性）
- `providers.minimax.baseUrl`: MiniMax API base URL を上書きします（デフォルト `https://api.minimax.io`、env: `MINIMAX_API_HOST`）。
- `providers.minimax.model`: TTS モデル（デフォルト `speech-2.8-hd`、env: `MINIMAX_TTS_MODEL`）。
- `providers.minimax.voiceId`: voice 識別子（デフォルト `English_expressive_narrator`、env: `MINIMAX_TTS_VOICE_ID`）。
- `providers.minimax.speed`: 再生速度 `0.5..2.0`（デフォルト 1.0）。
- `providers.minimax.vol`: 音量 `(0, 10]`（デフォルト 1.0、0 より大きい必要があります）。
- `providers.minimax.pitch`: ピッチシフト `-12..12`（デフォルト 0）。
- `providers.microsoft.enabled`: Microsoft speech の使用を許可します（デフォルト `true`、API キー不要）。
- `providers.microsoft.voice`: Microsoft neural voice 名（例: `en-US-MichelleNeural`）。
- `providers.microsoft.lang`: 言語コード（例: `en-US`）。
- `providers.microsoft.outputFormat`: Microsoft 出力形式（例: `audio-24khz-48kbitrate-mono-mp3`）。
  - 有効な値については Microsoft Speech output formats を参照してください。すべての形式がバンドルされた Edge ベース transport でサポートされるわけではありません。
- `providers.microsoft.rate` / `providers.microsoft.pitch` / `providers.microsoft.volume`: パーセント文字列（例: `+10%`、`-5%`）。
- `providers.microsoft.saveSubtitles`: 音声ファイルと並べて JSON 字幕を書き出します。
- `providers.microsoft.proxy`: Microsoft speech リクエスト用のプロキシ URL。
- `providers.microsoft.timeoutMs`: リクエストタイムアウト上書き（ms）。
- `edge.*`: 同じ Microsoft 設定に対する従来の alias。

## モデル駆動の上書き（デフォルトで有効）

デフォルトでは、モデルは単一の返信に対して TTS ディレクティブを出力**できます**。
`messages.tts.auto` が `tagged` の場合、これらのディレクティブが音声をトリガーするために必要です。

有効な場合、モデルは `[[tts:...]]` ディレクティブを出力して単一の返信の voice を上書きできます。さらに、音声内にのみ現れるべき表現タグ（笑い、歌唱キューなど）を提供するために、任意の `[[tts:text]]...[[/tts:text]]` ブロックも使用できます。

`provider=...` ディレクティブは、`modelOverrides.allowProvider: true` でない限り無視されます。

返信 payload の例:

```
Here you go.

[[tts:voiceId=pMsXgVXv3BLzUgSXRplE model=eleven_v3 speed=1.1]]
[[tts:text]](laughs) Read the song once more.[[/tts:text]]
```

利用可能なディレクティブキー（有効時）:

- `provider`（登録済み speech provider ID。例: `openai`、`elevenlabs`、`minimax`、`microsoft`。`allowProvider: true` が必要）
- `voice`（OpenAI voice）または `voiceId`（ElevenLabs / MiniMax）
- `model`（OpenAI TTS model、ElevenLabs model ID、または MiniMax model）
- `stability`、`similarityBoost`、`style`、`speed`、`useSpeakerBoost`
- `vol` / `volume`（MiniMax 音量、0-10）
- `pitch`（MiniMax pitch、-12 から 12）
- `applyTextNormalization`（`auto|on|off`）
- `languageCode`（ISO 639-1）
- `seed`

すべてのモデル上書きを無効にする:

```json5
{
  messages: {
    tts: {
      modelOverrides: {
        enabled: false,
      },
    },
  },
}
```

任意の allowlist（他の調整項目を設定可能にしたまま provider 切り替えを有効化）:

```json5
{
  messages: {
    tts: {
      modelOverrides: {
        enabled: true,
        allowProvider: true,
        allowSeed: false,
      },
    },
  },
}
```

## ユーザーごとの設定

スラッシュコマンドは、ローカル上書きを `prefsPath` に書き込みます（デフォルト:
`~/.openclaw/settings/tts.json`、`OPENCLAW_TTS_PREFS` または
`messages.tts.prefsPath` で上書き可能）。

保存されるフィールド:

- `enabled`
- `provider`
- `maxLength`（summary しきい値。デフォルト 1500 文字）
- `summarize`（デフォルト `true`）

これらは、その host 上で `messages.tts.*` を上書きします。

## 出力形式（固定）

- **Feishu / Matrix / Telegram / WhatsApp**: Opus 音声メッセージ（ElevenLabs の `opus_48000_64`、OpenAI の `opus`）。
  - 48kHz / 64kbps は、音声メッセージとして良いトレードオフです。
- **その他のチャネル**: MP3（ElevenLabs の `mp3_44100_128`、OpenAI の `mp3`）。
  - 44.1kHz / 128kbps は、音声明瞭性のデフォルトバランスです。
- **MiniMax**: MP3（`speech-2.8-hd` モデル、32kHz サンプルレート）。voice-note 形式はネイティブにはサポートされません。Opus 音声メッセージを確実に使いたい場合は OpenAI または ElevenLabs を使用してください。
- **Microsoft**: `microsoft.outputFormat` を使用します（デフォルト `audio-24khz-48kbitrate-mono-mp3`）。
  - バンドルされた transport は `outputFormat` を受け付けますが、すべての形式がサービスで利用できるわけではありません。
  - 出力形式の値は Microsoft Speech output formats に従います（Ogg/WebM Opus を含む）。
  - Telegram の `sendVoice` は OGG/MP3/M4A を受け付けます。Opus 音声メッセージを確実に使いたい場合は OpenAI/ElevenLabs を使用してください。
  - 設定された Microsoft 出力形式が失敗した場合、OpenClaw は MP3 で再試行します。

OpenAI/ElevenLabs の出力形式はチャネルごとに固定です（上記参照）。

## 自動 TTS の動作

有効な場合、OpenClaw は次のように動作します:

- 返信にすでに media または `MEDIA:` ディレクティブが含まれている場合は TTS をスキップします。
- 非常に短い返信（10 文字未満）はスキップします。
- 有効な場合、長い返信を `agents.defaults.model.primary`（または `summaryModel`）を使って要約します。
- 生成された音声を返信に添付します。

返信が `maxLength` を超えており、summary が off の場合（または
summary model 用の API キーがない場合）、音声はスキップされ、
通常のテキスト返信が送信されます。

## フローダイアグラム

```
Reply -> TTS enabled?
  no  -> テキストを送信
  yes -> media / MEDIA: / 短文あり?
          yes -> テキストを送信
          no  -> 長さ > 上限?
                   no  -> TTS -> 音声を添付
                   yes -> summary 有効?
                            no  -> テキストを送信
                            yes -> 要約（summaryModel または agents.defaults.model.primary）
                                      -> TTS -> 音声を添付
```

## スラッシュコマンドの使用

コマンドは 1 つだけです: `/tts`。
有効化の詳細は [Slash commands](/ja-JP/tools/slash-commands) を参照してください。

Discord の注意: `/tts` は Discord 組み込みコマンドのため、OpenClaw は
そこでネイティブコマンドとして `/voice` を登録します。テキストの `/tts ...` は引き続き動作します。

```
/tts off
/tts on
/tts status
/tts provider openai
/tts limit 2000
/tts summary off
/tts audio Hello from OpenClaw
```

注意:

- コマンドには認可済み送信者が必要です（allowlist/owner ルールは引き続き適用されます）。
- `commands.text` またはネイティブコマンド登録を有効にする必要があります。
- config の `messages.tts.auto` は `off|always|inbound|tagged` を受け付けます。
- `/tts on` はローカル TTS 設定を `always` に書き込み、`/tts off` は `off` に書き込みます。
- `inbound` または `tagged` をデフォルトにしたい場合は config を使用してください。
- `limit` と `summary` はメイン config ではなくローカル prefs に保存されます。
- `/tts audio` は 1 回限りの音声返信を生成します（TTS を on に切り替えるわけではありません）。
- `/tts status` には最新試行の fallback 可視性が含まれます:
  - fallback 成功: `Fallback: <primary> -> <used>` と `Attempts: ...`
  - 失敗: `Error: ...` と `Attempts: ...`
  - 詳細診断: `Attempt details: provider:outcome(reasonCode) latency`
- OpenAI と ElevenLabs の API 失敗には、解析済みの provider エラー詳細と request id（provider が返した場合）が含まれるようになっており、TTS エラー/ログに表示されます。

## agent ツール

`tts` ツールはテキストを音声に変換し、返信配信用の音声添付を返します。チャネルが Feishu、Matrix、Telegram、または WhatsApp の場合、音声はファイル添付ではなく voice message として配信されます。

## Gateway RPC

Gateway メソッド:

- `tts.status`
- `tts.enable`
- `tts.disable`
- `tts.convert`
- `tts.setProvider`
- `tts.providers`
