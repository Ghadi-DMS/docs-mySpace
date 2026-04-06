---
read_when:
    - OpenClaw plugin을 만들고 있을 때
    - plugin config schema를 배포해야 하거나 plugin 검증 오류를 디버깅해야 할 때
summary: Plugin manifest + JSON schema 요구사항(엄격한 config 검증)
title: Plugin Manifest
x-i18n:
    generated_at: "2026-04-06T03:10:06Z"
    model: gpt-5.4
    provider: openai
    source_hash: f6f915a761cdb5df77eba5d2ccd438c65445bd2ab41b0539d1200e63e8cf2c3a
    source_path: plugins/manifest.md
    workflow: 15
---

# Plugin manifest (`openclaw.plugin.json`)

이 페이지는 **네이티브 OpenClaw plugin manifest**만을 다룹니다.

호환되는 bundle 레이아웃은 [Plugin bundles](/ko/plugins/bundles)를 참조하세요.

호환 bundle 형식은 다른 manifest 파일을 사용합니다.

- Codex bundle: `.codex-plugin/plugin.json`
- Claude bundle: `.claude-plugin/plugin.json` 또는 manifest가 없는 기본 Claude component
  레이아웃
- Cursor bundle: `.cursor-plugin/plugin.json`

OpenClaw는 이러한 bundle 레이아웃도 자동 감지하지만, 여기서 설명하는
`openclaw.plugin.json` schema로 검증되지는 않습니다.

호환 bundle의 경우 OpenClaw는 현재 bundle metadata, 선언된
skill 루트, Claude command 루트, Claude bundle `settings.json` 기본값,
Claude bundle LSP 기본값, 그리고 레이아웃이 OpenClaw 런타임 기대치와
일치할 때 지원되는 hook pack을 읽습니다.

모든 네이티브 OpenClaw plugin은 **plugin 루트**에
`openclaw.plugin.json` 파일을 **반드시** 포함해야 합니다. OpenClaw는 이 manifest를 사용해
plugin 코드를 **실행하지 않고도** 설정을 검증합니다. Manifest가 없거나 유효하지 않으면
plugin 오류로 처리되며 config 검증이 차단됩니다.

전체 plugin 시스템 가이드는 [Plugins](/ko/tools/plugin)를 참조하세요.
네이티브 capability 모델과 현재 외부 호환성 가이드는
[Capability model](/ko/plugins/architecture#public-capability-model)을 참조하세요.

## 이 파일의 역할

`openclaw.plugin.json`은 OpenClaw가 plugin 코드를 로드하기 전에 읽는
metadata입니다.

다음 용도로 사용하세요.

- plugin 식별
- config 검증
- plugin 런타임을 부팅하지 않고도 사용할 수 있어야 하는 auth 및 onboarding metadata
- plugin 런타임이 로드되기 전에 해결되어야 하는 alias 및 auto-enable metadata
- 런타임이 로드되기 전에 plugin을 자동 활성화해야 하는 shorthand model-family
  ownership metadata
- 번들 compat wiring 및 contract coverage에 사용되는 정적 capability ownership 스냅샷
- 런타임을 로드하지 않고도 catalog 및 검증 표면에 병합되어야 하는
  채널별 config metadata
- config UI 힌트

다음 용도로 사용하지 마세요.

- 런타임 동작 등록
- 코드 entrypoint 선언
- npm install metadata

이런 항목은 plugin 코드와 `package.json`에 속합니다.

## 최소 예시

```json
{
  "id": "voice-call",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

## 확장 예시

```json
{
  "id": "openrouter",
  "name": "OpenRouter",
  "description": "OpenRouter provider plugin",
  "version": "1.0.0",
  "providers": ["openrouter"],
  "modelSupport": {
    "modelPrefixes": ["router-"]
  },
  "providerAuthEnvVars": {
    "openrouter": ["OPENROUTER_API_KEY"]
  },
  "providerAuthChoices": [
    {
      "provider": "openrouter",
      "method": "api-key",
      "choiceId": "openrouter-api-key",
      "choiceLabel": "OpenRouter API key",
      "groupId": "openrouter",
      "groupLabel": "OpenRouter",
      "optionKey": "openrouterApiKey",
      "cliFlag": "--openrouter-api-key",
      "cliOption": "--openrouter-api-key <key>",
      "cliDescription": "OpenRouter API key",
      "onboardingScopes": ["text-inference"]
    }
  ],
  "uiHints": {
    "apiKey": {
      "label": "API key",
      "placeholder": "sk-or-v1-...",
      "sensitive": true
    }
  },
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "apiKey": {
        "type": "string"
      }
    }
  }
}
```

## 최상위 필드 참고

| Field                               | Required | Type                             | 의미                                                                                                                                                                                                |
| ----------------------------------- | -------- | -------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`                                | Yes      | `string`                         | 정식 plugin id입니다. `plugins.entries.<id>`에서 사용하는 id입니다.                                                                                                                                  |
| `configSchema`                      | Yes      | `object`                         | 이 plugin config용 인라인 JSON Schema입니다.                                                                                                                                                        |
| `enabledByDefault`                  | No       | `true`                           | 번들 plugin을 기본 활성화로 표시합니다. 기본 비활성 상태로 두려면 생략하거나 `true`가 아닌 값을 설정하세요.                                                                                         |
| `legacyPluginIds`                   | No       | `string[]`                       | 이 정식 plugin id로 정규화되는 legacy id입니다.                                                                                                                                                     |
| `autoEnableWhenConfiguredProviders` | No       | `string[]`                       | auth, config, 또는 model ref에서 이 provider id가 언급되면 이 plugin을 자동 활성화해야 하는 provider id입니다.                                                                                     |
| `kind`                              | No       | `"memory"` \| `"context-engine"` | `plugins.slots.*`에서 사용하는 배타적 plugin 종류를 선언합니다.                                                                                                                                     |
| `channels`                          | No       | `string[]`                       | 이 plugin이 소유하는 channel id입니다. 탐지 및 config 검증에 사용됩니다.                                                                                                                            |
| `providers`                         | No       | `string[]`                       | 이 plugin이 소유하는 provider id입니다.                                                                                                                                                             |
| `modelSupport`                      | No       | `object`                         | 런타임 전에 plugin을 자동 로드하는 데 사용되는 manifest 소유 shorthand model-family metadata입니다.                                                                                                  |
| `providerAuthEnvVars`               | No       | `Record<string, string[]>`       | OpenClaw가 plugin 코드를 로드하지 않고도 검사할 수 있는 가벼운 provider-auth env metadata입니다.                                                                                                     |
| `providerAuthChoices`               | No       | `object[]`                       | onboarding picker, preferred-provider 해결, 단순 CLI 플래그 wiring을 위한 가벼운 auth-choice metadata입니다.                                                                                        |
| `contracts`                         | No       | `object`                         | speech, realtime transcription, realtime voice, media-understanding, image-generation, music-generation, video-generation, web-fetch, web search, tool ownership를 위한 정적 번들 capability 스냅샷입니다. |
| `channelConfigs`                    | No       | `Record<string, object>`         | 런타임 로드 전에 탐지 및 검증 표면에 병합되는 manifest 소유 채널 config metadata입니다.                                                                                                             |
| `skills`                            | No       | `string[]`                       | plugin 루트를 기준으로 한 skill 디렉터리입니다.                                                                                                                                                     |
| `name`                              | No       | `string`                         | 사람이 읽기 쉬운 plugin 이름입니다.                                                                                                                                                                 |
| `description`                       | No       | `string`                         | plugin 표면에 표시되는 짧은 요약입니다.                                                                                                                                                             |
| `version`                           | No       | `string`                         | 정보 제공용 plugin 버전입니다.                                                                                                                                                                      |
| `uiHints`                           | No       | `Record<string, object>`         | config 필드용 UI 라벨, placeholder, 민감도 힌트입니다.                                                                                                                                              |

## `providerAuthChoices` 참고

각 `providerAuthChoices` 항목은 하나의 onboarding 또는 auth choice를 설명합니다.
OpenClaw는 provider 런타임이 로드되기 전에 이를 읽습니다.

| Field                 | Required | Type                                            | 의미                                                                                     |
| --------------------- | -------- | ----------------------------------------------- | ---------------------------------------------------------------------------------------- |
| `provider`            | Yes      | `string`                                        | 이 choice가 속한 provider id입니다.                                                      |
| `method`              | Yes      | `string`                                        | 디스패치할 auth method id입니다.                                                         |
| `choiceId`            | Yes      | `string`                                        | onboarding 및 CLI 흐름에서 사용하는 안정적인 auth-choice id입니다.                       |
| `choiceLabel`         | No       | `string`                                        | 사용자 대상 라벨입니다. 생략하면 OpenClaw는 `choiceId`로 fallback합니다.                 |
| `choiceHint`          | No       | `string`                                        | picker용 짧은 도움말 텍스트입니다.                                                       |
| `assistantPriority`   | No       | `number`                                        | 어시스턴트 주도 대화형 picker에서 값이 낮을수록 먼저 정렬됩니다.                         |
| `assistantVisibility` | No       | `"visible"` \| `"manual-only"`                  | 어시스턴트 picker에서는 숨기되 수동 CLI 선택은 계속 허용합니다.                          |
| `deprecatedChoiceIds` | No       | `string[]`                                      | 사용자를 이 대체 choice로 리디렉션해야 하는 legacy choice id입니다.                      |
| `groupId`             | No       | `string`                                        | 관련 choice를 묶기 위한 선택적 그룹 id입니다.                                            |
| `groupLabel`          | No       | `string`                                        | 해당 그룹의 사용자 대상 라벨입니다.                                                      |
| `groupHint`           | No       | `string`                                        | 그룹용 짧은 도움말 텍스트입니다.                                                         |
| `optionKey`           | No       | `string`                                        | 단일 플래그 auth 흐름용 내부 옵션 키입니다.                                              |
| `cliFlag`             | No       | `string`                                        | `--openrouter-api-key` 같은 CLI 플래그 이름입니다.                                       |
| `cliOption`           | No       | `string`                                        | `--openrouter-api-key <key>` 같은 전체 CLI 옵션 형식입니다.                              |
| `cliDescription`      | No       | `string`                                        | CLI 도움말에 사용되는 설명입니다.                                                        |
| `onboardingScopes`    | No       | `Array<"text-inference" \| "image-generation">` | 이 choice가 표시되어야 하는 onboarding 표면입니다. 생략하면 기본값은 `["text-inference"]`입니다. |

## `uiHints` 참고

`uiHints`는 config 필드 이름에서 작은 렌더링 힌트로 매핑되는 맵입니다.

```json
{
  "uiHints": {
    "apiKey": {
      "label": "API key",
      "help": "Used for OpenRouter requests",
      "placeholder": "sk-or-v1-...",
      "sensitive": true
    }
  }
}
```

각 필드 힌트에는 다음이 포함될 수 있습니다.

| Field         | Type       | 의미                                 |
| ------------- | ---------- | ------------------------------------ |
| `label`       | `string`   | 사용자 대상 필드 라벨입니다.         |
| `help`        | `string`   | 짧은 도움말 텍스트입니다.            |
| `tags`        | `string[]` | 선택적 UI 태그입니다.                |
| `advanced`    | `boolean`  | 이 필드를 고급 항목으로 표시합니다.  |
| `sensitive`   | `boolean`  | 이 필드를 비밀값 또는 민감값으로 표시합니다. |
| `placeholder` | `string`   | 폼 입력용 placeholder 텍스트입니다.  |

## `contracts` 참고

`contracts`는 OpenClaw가 plugin 런타임을 import하지 않고도 읽을 수 있는
정적 capability ownership metadata에만 사용하세요.

```json
{
  "contracts": {
    "speechProviders": ["openai"],
    "realtimeTranscriptionProviders": ["openai"],
    "realtimeVoiceProviders": ["openai"],
    "mediaUnderstandingProviders": ["openai", "openai-codex"],
    "imageGenerationProviders": ["openai"],
    "videoGenerationProviders": ["qwen"],
    "webFetchProviders": ["firecrawl"],
    "webSearchProviders": ["gemini"],
    "tools": ["firecrawl_search", "firecrawl_scrape"]
  }
}
```

각 목록은 선택 사항입니다.

| Field                            | Type       | 의미                                                   |
| -------------------------------- | ---------- | ------------------------------------------------------ |
| `speechProviders`                | `string[]` | 이 plugin이 소유하는 speech provider id입니다.         |
| `realtimeTranscriptionProviders` | `string[]` | 이 plugin이 소유하는 realtime-transcription provider id입니다. |
| `realtimeVoiceProviders`         | `string[]` | 이 plugin이 소유하는 realtime-voice provider id입니다. |
| `mediaUnderstandingProviders`    | `string[]` | 이 plugin이 소유하는 media-understanding provider id입니다. |
| `imageGenerationProviders`       | `string[]` | 이 plugin이 소유하는 image-generation provider id입니다. |
| `videoGenerationProviders`       | `string[]` | 이 plugin이 소유하는 video-generation provider id입니다. |
| `webFetchProviders`              | `string[]` | 이 plugin이 소유하는 web-fetch provider id입니다.      |
| `webSearchProviders`             | `string[]` | 이 plugin이 소유하는 web-search provider id입니다.     |
| `tools`                          | `string[]` | 번들 contract 검사용으로 이 plugin이 소유하는 agent tool 이름입니다. |

## `channelConfigs` 참고

채널 plugin이 런타임 로드 전에 가벼운 config metadata를 필요로 할 때는
`channelConfigs`를 사용하세요.

```json
{
  "channelConfigs": {
    "matrix": {
      "schema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "homeserverUrl": { "type": "string" }
        }
      },
      "uiHints": {
        "homeserverUrl": {
          "label": "Homeserver URL",
          "placeholder": "https://matrix.example.com"
        }
      },
      "label": "Matrix",
      "description": "Matrix homeserver connection",
      "preferOver": ["matrix-legacy"]
    }
  }
}
```

각 채널 항목에는 다음이 포함될 수 있습니다.

| Field         | Type                     | 의미                                                                                 |
| ------------- | ------------------------ | ------------------------------------------------------------------------------------ |
| `schema`      | `object`                 | `channels.<id>`용 JSON Schema입니다. 선언된 각 채널 config 항목에 필수입니다.       |
| `uiHints`     | `Record<string, object>` | 해당 채널 config 섹션용 선택적 UI 라벨/placeholder/민감도 힌트입니다.              |
| `label`       | `string`                 | 런타임 metadata가 준비되지 않았을 때 picker 및 inspect 표면에 병합되는 채널 라벨입니다. |
| `description` | `string`                 | inspect 및 catalog 표면용 짧은 채널 설명입니다.                                    |
| `preferOver`  | `string[]`               | 선택 표면에서 이 채널이 우선해야 하는 legacy 또는 낮은 우선순위 plugin id입니다.   |

## `modelSupport` 참고

플러그인 런타임이 로드되기 전에 OpenClaw가 `gpt-5.4` 또는 `claude-sonnet-4.6` 같은
shorthand model id로부터 provider plugin을 추론해야 한다면
`modelSupport`를 사용하세요.

```json
{
  "modelSupport": {
    "modelPrefixes": ["gpt-", "o1", "o3", "o4"],
    "modelPatterns": ["^computer-use-preview"]
  }
}
```

OpenClaw는 다음 우선순위를 적용합니다.

- 명시적 `provider/model` ref는 소유 `providers` manifest metadata를 사용
- `modelPatterns`가 `modelPrefixes`보다 우선
- 번들되지 않은 plugin과 번들 plugin이 둘 다 일치하면, 번들되지 않은
  plugin이 우선
- 나머지 모호성은 사용자나 config가 provider를 지정할 때까지 무시

필드:

| Field           | Type       | 의미                                                                        |
| --------------- | ---------- | --------------------------------------------------------------------------- |
| `modelPrefixes` | `string[]` | shorthand model id에 대해 `startsWith`로 일치시키는 prefix입니다.          |
| `modelPatterns` | `string[]` | profile suffix 제거 후 shorthand model id에 일치시키는 regex source입니다. |

legacy 최상위 capability 키는 deprecated되었습니다. `openclaw doctor --fix`를 사용해
`speechProviders`, `realtimeTranscriptionProviders`,
`realtimeVoiceProviders`, `mediaUnderstandingProviders`,
`imageGenerationProviders`, `videoGenerationProviders`,
`webFetchProviders`, `webSearchProviders`를 `contracts` 아래로 옮기세요.
일반 manifest 로딩은 더 이상 이러한 최상위 필드를 capability
ownership으로 취급하지 않습니다.

## Manifest와 `package.json`의 차이

두 파일은 서로 다른 역할을 합니다.

| File                   | 용도                                                                                                                      |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.plugin.json` | plugin 코드가 실행되기 전에 반드시 있어야 하는 탐지, config 검증, auth-choice metadata, UI 힌트                         |
| `package.json`         | npm metadata, dependency 설치, entrypoint, install gating, setup, catalog metadata에 사용되는 `openclaw` 블록          |

어떤 metadata를 어디에 둬야 할지 확실하지 않다면 다음 규칙을 사용하세요.

- OpenClaw가 plugin 코드를 로드하기 전에 알아야 한다면 `openclaw.plugin.json`에 둡니다
- 패키징, 엔트리 파일, npm install 동작에 관한 것이면 `package.json`에 둡니다

### 탐지에 영향을 주는 `package.json` 필드

일부 사전 런타임 plugin metadata는 의도적으로 `openclaw.plugin.json`이 아니라
`package.json`의 `openclaw` 블록에 들어 있습니다.

중요한 예시:

| Field                                                             | 의미                                                                                                                                 |
| ----------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `openclaw.extensions`                                             | 네이티브 plugin entrypoint를 선언합니다.                                                                                            |
| `openclaw.setupEntry`                                             | onboarding 및 지연된 채널 시작 중에 사용되는 가벼운 setup 전용 entrypoint입니다.                                                   |
| `openclaw.channel`                                                | 라벨, 문서 경로, alias, 선택용 문구 같은 가벼운 채널 catalog metadata입니다.                                                       |
| `openclaw.channel.configuredState`                                | 전체 채널 런타임을 로드하지 않고도 "env 전용 설정이 이미 존재하는가?"에 답할 수 있는 가벼운 configured-state checker metadata입니다. |
| `openclaw.channel.persistedAuthState`                             | 전체 채널 런타임을 로드하지 않고도 "이미 로그인된 것이 있는가?"에 답할 수 있는 가벼운 persisted-auth checker metadata입니다.        |
| `openclaw.install.npmSpec` / `openclaw.install.localPath`         | 번들 plugin 및 외부 게시 plugin용 install/update 힌트입니다.                                                                       |
| `openclaw.install.defaultChoice`                                  | 여러 install 소스를 사용할 수 있을 때 선호되는 install 경로입니다.                                                                 |
| `openclaw.install.minHostVersion`                                 | `>=2026.3.22` 같은 semver 하한을 사용하는 최소 지원 OpenClaw 호스트 버전입니다.                                                   |
| `openclaw.install.allowInvalidConfigRecovery`                     | config가 유효하지 않을 때 제한된 번들 plugin 재설치 복구 경로를 허용합니다.                                                        |
| `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen` | 시작 중 전체 채널 plugin보다 먼저 setup 전용 채널 표면이 로드되도록 합니다.                                                        |

`openclaw.install.minHostVersion`은 install 중과 manifest
레지스트리 로딩 중에 강제됩니다. 유효하지 않은 값은 거부되며, 더 새로운 유효값은
이전 호스트에서 plugin을 건너뜁니다.

`openclaw.install.allowInvalidConfigRecovery`는 의도적으로 범위가 좁습니다.
임의의 깨진 config를 설치 가능하게 만들지는 않습니다. 현재는 누락된 번들 plugin 경로 또는
같은 번들 plugin에 대한 오래된 `channels.<id>` 항목 같은 특정한 오래된 번들 plugin 업그레이드 실패에서만
install 흐름 복구를 허용합니다. 관련 없는 config 오류는 여전히 install을 차단하고
운영자를 `openclaw doctor --fix`로 안내합니다.

`openclaw.channel.persistedAuthState`는 작은 checker 모듈을 위한
패키지 metadata입니다.

```json
{
  "openclaw": {
    "channel": {
      "id": "whatsapp",
      "persistedAuthState": {
        "specifier": "./auth-presence",
        "exportName": "hasAnyWhatsAppAuth"
      }
    }
  }
}
```

setup, doctor, 또는 configured-state 흐름이 전체 채널 plugin이 로드되기 전에
가벼운 예/아니오 auth probe를 필요로 할 때 사용하세요. 대상 export는
저장된 상태만 읽는 작은 함수여야 합니다. 전체 채널 런타임 barrel을 통해
연결하지 마세요.

`openclaw.channel.configuredState`도 가벼운 env 전용
configured check를 위해 동일한 형태를 따릅니다.

```json
{
  "openclaw": {
    "channel": {
      "id": "telegram",
      "configuredState": {
        "specifier": "./configured-state",
        "exportName": "hasTelegramConfiguredState"
      }
    }
  }
}
```

채널이 env 또는 다른 작은
비런타임 입력으로 configured-state에 답할 수 있을 때 사용하세요. Check에 전체 config 해결이나 실제
채널 런타임이 필요하다면, 그 로직은 대신 plugin `config.hasConfiguredState`
hook에 두세요.

## JSON Schema 요구사항

- **모든 plugin은 JSON Schema를 반드시 포함해야** 하며, config를 받지 않는 경우도 예외가 아닙니다.
- 빈 schema도 허용됩니다(예: `{ "type": "object", "additionalProperties": false }`).
- Schema는 런타임이 아니라 config 읽기/쓰기 시점에 검증됩니다.

## 검증 동작

- 알 수 없는 `channels.*` 키는 **오류**입니다. 단, 해당 channel id가
  plugin manifest에 선언된 경우는 예외입니다.
- `plugins.entries.<id>`, `plugins.allow`, `plugins.deny`, `plugins.slots.*`는
  **탐지 가능한** plugin id를 참조해야 합니다. 알 수 없는 id는 **오류**입니다.
- plugin이 설치되어 있지만 manifest 또는 schema가 깨졌거나 누락된 경우,
  검증이 실패하고 Doctor가 plugin 오류를 보고합니다.
- plugin config가 존재하지만 plugin이 **비활성화**된 경우, config는 유지되며
  Doctor + 로그에 **경고**가 표시됩니다.

전체 `plugins.*` schema는 [Configuration reference](/ko/gateway/configuration)를 참조하세요.

## 참고

- Manifest는 로컬 파일시스템 로드를 포함한 **네이티브 OpenClaw plugin에 필수**입니다.
- 런타임은 여전히 plugin 모듈을 별도로 로드하며, manifest는
  탐지 + 검증만을 위한 것입니다.
- 네이티브 manifest는 JSON5로 파싱되므로 주석, trailing comma,
  따옴표 없는 키도 최종 값이 객체인 한 허용됩니다.
- 문서화된 manifest 필드만 manifest loader가 읽습니다. 여기에
  custom 최상위 키를 추가하지 마세요.
- `providerAuthEnvVars`는 auth probe, env-marker
  검증, 그와 유사하게 env 이름만 검사하려고 plugin 런타임을 부팅해서는 안 되는
  provider-auth 표면을 위한 가벼운 metadata 경로입니다.
- `providerAuthChoices`는 auth-choice picker,
  `--auth-choice` 해결, preferred-provider 매핑, 단순 onboarding
  CLI 플래그 등록을 provider 런타임 로드 전에 수행하기 위한 가벼운 metadata 경로입니다.
  provider 코드가 필요한 런타임 wizard
  metadata는
  [Provider runtime hooks](/ko/plugins/architecture#provider-runtime-hooks)를 참조하세요.
- 배타적 plugin 종류는 `plugins.slots.*`를 통해 선택됩니다.
  - `kind: "memory"`는 `plugins.slots.memory`로 선택됩니다.
  - `kind: "context-engine"`는 `plugins.slots.contextEngine`으로 선택됩니다
    (기본값: 내장 `legacy`).
- plugin에 필요 없다면 `channels`, `providers`, `skills`는
  생략할 수 있습니다.
- plugin이 네이티브 모듈에 의존한다면, 빌드 단계와
  패키지 매니저 allowlist 요구사항(예: pnpm `allow-build-scripts`
  - `pnpm rebuild <package>`)을 문서화하세요.

## 관련 문서

- [Building Plugins](/ko/plugins/building-plugins) — plugin 시작하기
- [Plugin Architecture](/ko/plugins/architecture) — 내부 아키텍처
- [SDK Overview](/ko/plugins/sdk-overview) — Plugin SDK 참고 자료
