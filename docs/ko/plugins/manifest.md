---
read_when:
    - OpenClaw plugin을 빌드하고 있습니다
    - plugin 구성 스키마를 배포하거나 plugin 검증 오류를 디버그해야 합니다
summary: Plugin 매니페스트 + JSON Schema 요구 사항(엄격한 구성 검증)
title: Plugin 매니페스트
x-i18n:
    generated_at: "2026-04-09T01:29:23Z"
    model: gpt-5.4
    provider: openai
    source_hash: 9a7ee4b621a801d2a8f32f8976b0e1d9433c7810eb360aca466031fc0ffb286a
    source_path: plugins/manifest.md
    workflow: 15
---

# Plugin 매니페스트(openclaw.plugin.json)

이 페이지는 **네이티브 OpenClaw plugin 매니페스트** 전용입니다.

호환되는 번들 레이아웃은 [Plugin bundles](/ko/plugins/bundles)를 참조하세요.

호환되는 번들 형식은 서로 다른 매니페스트 파일을 사용합니다.

- Codex 번들: `.codex-plugin/plugin.json`
- Claude 번들: `.claude-plugin/plugin.json` 또는 매니페스트가 없는 기본 Claude component
  레이아웃
- Cursor 번들: `.cursor-plugin/plugin.json`

OpenClaw는 이러한 번들 레이아웃도 자동 감지하지만, 여기서 설명하는 `openclaw.plugin.json` 스키마에 대해서는 검증하지 않습니다.

호환 번들의 경우, OpenClaw는 현재 레이아웃이 OpenClaw 런타임 기대 사항과 일치할 때 번들 메타데이터와 선언된 skill 루트, Claude command 루트, Claude 번들 `settings.json` 기본값, Claude 번들 LSP 기본값, 지원되는 hook pack을 읽습니다.

모든 네이티브 OpenClaw plugin은 **plugin 루트**에 `openclaw.plugin.json` 파일을 **반드시** 포함해야 합니다. OpenClaw는 이 매니페스트를 사용해 **plugin 코드를 실행하지 않고도** 구성을 검증합니다. 매니페스트가 없거나 유효하지 않으면 plugin 오류로 처리되며 구성 검증이 차단됩니다.

전체 plugin 시스템 가이드는 [Plugins](/ko/tools/plugin)를 참조하세요.
네이티브 capability 모델과 현재 외부 호환성 지침은 [Capability model](/ko/plugins/architecture#public-capability-model)을 참조하세요.

## 이 파일의 역할

`openclaw.plugin.json`은 OpenClaw가 plugin 코드를 로드하기 전에 읽는 메타데이터입니다.

다음 용도로 사용하세요.

- plugin 식별자
- 구성 검증
- plugin 런타임을 부팅하지 않고도 사용할 수 있어야 하는 auth 및 onboarding 메타데이터
- plugin 런타임 로드 전에 해석되어야 하는 별칭 및 자동 활성화 메타데이터
- 런타임 로드 전에 plugin을 자동 활성화해야 하는 shorthand model-family 소유 메타데이터
- 번들 compat wiring 및 계약 커버리지에 사용되는 정적 capability 소유 스냅샷
- 런타임을 로드하지 않고 catalog 및 검증 표면에 병합되어야 하는 채널별 구성 메타데이터
- config UI 힌트

다음 용도로는 사용하지 마세요.

- 런타임 동작 등록
- 코드 entrypoint 선언
- npm 설치 메타데이터

이들은 plugin 코드와 `package.json`에 속합니다.

## 최소 예제

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

## 자세한 예제

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
  "cliBackends": ["openrouter-cli"],
  "providerAuthEnvVars": {
    "openrouter": ["OPENROUTER_API_KEY"]
  },
  "providerAuthAliases": {
    "openrouter-coding": "openrouter"
  },
  "channelEnvVars": {
    "openrouter-chatops": ["OPENROUTER_CHATOPS_TOKEN"]
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

## 최상위 필드 참조

| Field                               | Required | Type                             | 의미                                                                                                                                                                                                        |
| ----------------------------------- | -------- | -------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`                                | Yes      | `string`                         | 정식 plugin id입니다. 이 id는 `plugins.entries.<id>`에서 사용됩니다.                                                                                                                                       |
| `configSchema`                      | Yes      | `object`                         | 이 plugin 구성에 대한 인라인 JSON Schema입니다.                                                                                                                                                            |
| `enabledByDefault`                  | No       | `true`                           | 번들 plugin을 기본적으로 활성화된 것으로 표시합니다. 기본적으로 비활성화 상태로 두려면 생략하거나 `true`가 아닌 값을 설정하세요.                                                                          |
| `legacyPluginIds`                   | No       | `string[]`                       | 이 정식 plugin id로 정규화되는 레거시 id입니다.                                                                                                                                                            |
| `autoEnableWhenConfiguredProviders` | No       | `string[]`                       | auth, config 또는 model ref에서 해당 provider id를 언급할 때 이 plugin을 자동 활성화해야 하는 provider id입니다.                                                                                          |
| `kind`                              | No       | `"memory"` \| `"context-engine"` | `plugins.slots.*`에서 사용하는 배타적 plugin kind를 선언합니다.                                                                                                                                            |
| `channels`                          | No       | `string[]`                       | 이 plugin이 소유한 channel id입니다. 검색 및 구성 검증에 사용됩니다.                                                                                                                                       |
| `providers`                         | No       | `string[]`                       | 이 plugin이 소유한 provider id입니다.                                                                                                                                                                       |
| `modelSupport`                      | No       | `object`                         | 런타임 전에 plugin을 자동 로드하는 데 사용되는 매니페스트 소유 shorthand model-family 메타데이터입니다.                                                                                                    |
| `cliBackends`                       | No       | `string[]`                       | 이 plugin이 소유한 CLI inference backend id입니다. 명시적 config ref를 기준으로 시작 시 자동 활성화하는 데 사용됩니다.                                                                                    |
| `providerAuthEnvVars`               | No       | `Record<string, string[]>`       | OpenClaw가 plugin 코드를 로드하지 않고도 검사할 수 있는 저비용 provider-auth env 메타데이터입니다.                                                                                                         |
| `providerAuthAliases`               | No       | `Record<string, string>`         | auth 조회를 위해 다른 provider id를 재사용해야 하는 provider id입니다. 예를 들어 기본 provider API 키와 auth 프로필을 공유하는 코딩 provider가 해당됩니다.                                               |
| `channelEnvVars`                    | No       | `Record<string, string[]>`       | OpenClaw가 plugin 코드를 로드하지 않고도 검사할 수 있는 저비용 channel env 메타데이터입니다. env 기반 channel 설정 또는 범용 startup/config helper가 확인해야 하는 auth 표면에 사용하세요.               |
| `providerAuthChoices`               | No       | `object[]`                       | onboarding 선택기, 선호 provider 결정, 간단한 CLI 플래그 wiring을 위한 저비용 auth-choice 메타데이터입니다.                                                                                               |
| `contracts`                         | No       | `object`                         | speech, realtime transcription, realtime voice, media-understanding, image-generation, music-generation, video-generation, web-fetch, web search 및 tool 소유권에 대한 정적 번들 capability 스냅샷입니다. |
| `channelConfigs`                    | No       | `Record<string, object>`         | 런타임이 로드되기 전에 검색 및 검증 표면에 병합되는 매니페스트 소유 channel config 메타데이터입니다.                                                                                                      |
| `skills`                            | No       | `string[]`                       | plugin 루트를 기준으로 로드할 skill 디렉터리입니다.                                                                                                                                                        |
| `name`                              | No       | `string`                         | 사람이 읽을 수 있는 plugin 이름입니다.                                                                                                                                                                      |
| `description`                       | No       | `string`                         | plugin 표면에 표시되는 짧은 요약입니다.                                                                                                                                                                     |
| `version`                           | No       | `string`                         | 정보 제공용 plugin 버전입니다.                                                                                                                                                                              |
| `uiHints`                           | No       | `Record<string, object>`         | 구성 필드에 대한 UI 레이블, placeholder, 민감도 힌트입니다.                                                                                                                                                 |

## providerAuthChoices 참조

각 `providerAuthChoices` 항목은 하나의 onboarding 또는 auth 선택지를 설명합니다.
OpenClaw는 provider 런타임이 로드되기 전에 이를 읽습니다.

| Field                 | Required | Type                                            | 의미                                                                                             |
| --------------------- | -------- | ----------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| `provider`            | Yes      | `string`                                        | 이 선택지가 속한 provider id입니다.                                                               |
| `method`              | Yes      | `string`                                        | 전달할 auth method id입니다.                                                                     |
| `choiceId`            | Yes      | `string`                                        | onboarding 및 CLI 흐름에서 사용하는 안정적인 auth-choice id입니다.                               |
| `choiceLabel`         | No       | `string`                                        | 사용자용 레이블입니다. 생략하면 OpenClaw는 `choiceId`로 폴백합니다.                               |
| `choiceHint`          | No       | `string`                                        | 선택기에 표시할 짧은 도움말 텍스트입니다.                                                         |
| `assistantPriority`   | No       | `number`                                        | assistant 기반 대화형 선택기에서 더 낮은 값이 먼저 정렬됩니다.                                   |
| `assistantVisibility` | No       | `"visible"` \| `"manual-only"`                  | assistant 선택기에서는 숨기되 수동 CLI 선택은 계속 허용합니다.                                   |
| `deprecatedChoiceIds` | No       | `string[]`                                      | 사용자를 이 대체 선택지로 리디렉션해야 하는 레거시 choice id입니다.                               |
| `groupId`             | No       | `string`                                        | 관련 선택지를 그룹화하기 위한 선택적 group id입니다.                                             |
| `groupLabel`          | No       | `string`                                        | 해당 그룹의 사용자용 레이블입니다.                                                                |
| `groupHint`           | No       | `string`                                        | 그룹에 대한 짧은 도움말 텍스트입니다.                                                             |
| `optionKey`           | No       | `string`                                        | 간단한 단일 플래그 auth 흐름을 위한 내부 option key입니다.                                       |
| `cliFlag`             | No       | `string`                                        | `--openrouter-api-key` 같은 CLI 플래그 이름입니다.                                                |
| `cliOption`           | No       | `string`                                        | `--openrouter-api-key <key>` 같은 전체 CLI option 형태입니다.                                    |
| `cliDescription`      | No       | `string`                                        | CLI help에 사용되는 설명입니다.                                                                   |
| `onboardingScopes`    | No       | `Array<"text-inference" \| "image-generation">` | 이 선택지를 어떤 onboarding 표면에 표시할지 지정합니다. 생략하면 기본값은 `["text-inference"]`입니다. |

## uiHints 참조

`uiHints`는 구성 필드 이름에서 작은 렌더링 힌트로 매핑되는 맵입니다.

```json
{
  "uiHints": {
    "apiKey": {
      "label": "API key",
      "help": "OpenRouter 요청에 사용됨",
      "placeholder": "sk-or-v1-...",
      "sensitive": true
    }
  }
}
```

각 필드 힌트는 다음을 포함할 수 있습니다.

| Field         | Type       | 의미                                  |
| ------------- | ---------- | ------------------------------------- |
| `label`       | `string`   | 사용자용 필드 레이블입니다.            |
| `help`        | `string`   | 짧은 도움말 텍스트입니다.             |
| `tags`        | `string[]` | 선택적 UI 태그입니다.                 |
| `advanced`    | `boolean`  | 이 필드를 고급 항목으로 표시합니다.   |
| `sensitive`   | `boolean`  | 이 필드를 비밀 또는 민감 정보로 표시합니다. |
| `placeholder` | `string`   | 양식 입력의 placeholder 텍스트입니다. |

## contracts 참조

`contracts`는 OpenClaw가 plugin 런타임을 import하지 않고도 읽을 수 있는 정적 capability 소유 메타데이터에만 사용하세요.

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

| Field                            | Type       | 의미                                                        |
| -------------------------------- | ---------- | ----------------------------------------------------------- |
| `speechProviders`                | `string[]` | 이 plugin이 소유한 speech provider id입니다.                |
| `realtimeTranscriptionProviders` | `string[]` | 이 plugin이 소유한 realtime-transcription provider id입니다. |
| `realtimeVoiceProviders`         | `string[]` | 이 plugin이 소유한 realtime-voice provider id입니다.        |
| `mediaUnderstandingProviders`    | `string[]` | 이 plugin이 소유한 media-understanding provider id입니다.   |
| `imageGenerationProviders`       | `string[]` | 이 plugin이 소유한 image-generation provider id입니다.      |
| `videoGenerationProviders`       | `string[]` | 이 plugin이 소유한 video-generation provider id입니다.      |
| `webFetchProviders`              | `string[]` | 이 plugin이 소유한 web-fetch provider id입니다.             |
| `webSearchProviders`             | `string[]` | 이 plugin이 소유한 web-search provider id입니다.            |
| `tools`                          | `string[]` | 번들 계약 검사에 사용하는 이 plugin 소유의 agent tool 이름입니다. |

## channelConfigs 참조

channel plugin이 런타임 로드 전에 저비용 config 메타데이터를 필요로 하는 경우 `channelConfigs`를 사용하세요.

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
      "description": "Matrix homeserver 연결",
      "preferOver": ["matrix-legacy"]
    }
  }
}
```

각 channel 항목은 다음을 포함할 수 있습니다.

| Field         | Type                     | 의미                                                                                   |
| ------------- | ------------------------ | -------------------------------------------------------------------------------------- |
| `schema`      | `object`                 | `channels.<id>`용 JSON Schema입니다. 선언된 각 channel config 항목에 필수입니다.      |
| `uiHints`     | `Record<string, object>` | 해당 channel config 섹션에 대한 선택적 UI 레이블/placeholder/민감도 힌트입니다.       |
| `label`       | `string`                 | 런타임 메타데이터가 준비되지 않았을 때 선택기 및 검사 표면에 병합되는 channel 레이블입니다. |
| `description` | `string`                 | 검사 및 catalog 표면용 짧은 channel 설명입니다.                                       |
| `preferOver`  | `string[]`               | 선택 표면에서 이 channel이 우선해야 하는 레거시 또는 낮은 우선순위 plugin id입니다.    |

## modelSupport 참조

plugin 런타임이 로드되기 전에 OpenClaw가 `gpt-5.4` 또는 `claude-sonnet-4.6` 같은 shorthand model id로부터 provider plugin을 추론해야 한다면 `modelSupport`를 사용하세요.

```json
{
  "modelSupport": {
    "modelPrefixes": ["gpt-", "o1", "o3", "o4"],
    "modelPatterns": ["^computer-use-preview"]
  }
}
```

OpenClaw는 다음 우선순위를 적용합니다.

- 명시적인 `provider/model` ref는 소유 `providers` 매니페스트 메타데이터를 사용합니다
- `modelPatterns`가 `modelPrefixes`보다 우선합니다
- 비번들 plugin 하나와 번들 plugin 하나가 모두 일치하면 비번들 plugin이 우선합니다
- 그 외의 모호성은 사용자 또는 config가 provider를 지정할 때까지 무시됩니다

필드:

| Field           | Type       | 의미                                                                 |
| --------------- | ---------- | -------------------------------------------------------------------- |
| `modelPrefixes` | `string[]` | shorthand model id에 대해 `startsWith`로 일치시키는 접두사입니다.    |
| `modelPatterns` | `string[]` | 프로필 접미사를 제거한 뒤 shorthand model id에 대해 일치시키는 정규식 소스입니다. |

레거시 최상위 capability 키는 더 이상 권장되지 않습니다. `openclaw doctor --fix`를 사용해 `speechProviders`, `realtimeTranscriptionProviders`, `realtimeVoiceProviders`, `mediaUnderstandingProviders`, `imageGenerationProviders`, `videoGenerationProviders`, `webFetchProviders`, `webSearchProviders`를 `contracts` 아래로 이동하세요. 일반 매니페스트 로딩은 더 이상 이러한 최상위 필드를 capability 소유권으로 취급하지 않습니다.

## 매니페스트와 package.json의 차이

두 파일은 서로 다른 역할을 합니다.

| File                   | 용도                                                                                                                                 |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `openclaw.plugin.json` | plugin 코드가 실행되기 전에 반드시 존재해야 하는 검색, 구성 검증, auth-choice 메타데이터, UI 힌트                                 |
| `package.json`         | npm 메타데이터, 의존성 설치, 그리고 entrypoint, 설치 게이팅, 설정, catalog 메타데이터에 사용되는 `openclaw` 블록                |

어떤 메타데이터를 어디에 둘지 확신이 없다면 이 규칙을 사용하세요.

- OpenClaw가 plugin 코드를 로드하기 전에 알아야 한다면 `openclaw.plugin.json`에 넣으세요
- 패키징, entry 파일, 또는 npm 설치 동작과 관련 있다면 `package.json`에 넣으세요

### 검색에 영향을 주는 package.json 필드

일부 사전 런타임 plugin 메타데이터는 의도적으로 `openclaw.plugin.json`이 아니라 `package.json`의 `openclaw` 블록에 있습니다.

중요한 예:

| Field                                                             | 의미                                                                                                                                         |
| ----------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.extensions`                                             | 네이티브 plugin entrypoint를 선언합니다.                                                                                                     |
| `openclaw.setupEntry`                                             | onboarding 및 지연된 channel 시작 중에 사용되는 경량 setup 전용 entrypoint입니다.                                                           |
| `openclaw.channel`                                                | 레이블, 문서 경로, 별칭, 선택용 문구 같은 저비용 channel catalog 메타데이터입니다.                                                          |
| `openclaw.channel.configuredState`                                | 전체 channel 런타임을 로드하지 않고도 "env만으로 된 설정이 이미 존재하는가?"에 답할 수 있는 경량 configured-state 검사기 메타데이터입니다. |
| `openclaw.channel.persistedAuthState`                             | 전체 channel 런타임을 로드하지 않고도 "이미 로그인된 항목이 있는가?"에 답할 수 있는 경량 persisted-auth 검사기 메타데이터입니다.           |
| `openclaw.install.npmSpec` / `openclaw.install.localPath`         | 번들 및 외부 게시 plugin용 설치/업데이트 힌트입니다.                                                                                         |
| `openclaw.install.defaultChoice`                                  | 여러 설치 소스를 사용할 수 있을 때 선호되는 설치 경로입니다.                                                                                |
| `openclaw.install.minHostVersion`                                 | `>=2026.3.22` 같은 semver 하한을 사용하는 최소 지원 OpenClaw 호스트 버전입니다.                                                             |
| `openclaw.install.allowInvalidConfigRecovery`                     | 구성이 유효하지 않을 때 제한된 번들 plugin 재설치 복구 경로를 허용합니다.                                                                   |
| `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen` | 시작 중 전체 channel plugin보다 setup 전용 channel 표면을 먼저 로드할 수 있게 합니다.                                                      |

`openclaw.install.minHostVersion`은 설치 중과 매니페스트 레지스트리 로딩 중에 적용됩니다. 유효하지 않은 값은 거부되며, 유효하지만 더 새로운 값은 오래된 호스트에서 plugin을 건너뜁니다.

`openclaw.install.allowInvalidConfigRecovery`는 의도적으로 범위가 좁습니다. 임의의 깨진 구성을 설치 가능하게 만들지는 않습니다. 현재는 누락된 번들 plugin 경로 또는 동일 번들 plugin에 대한 오래된 `channels.<id>` 항목 같은 특정한 낡은 번들 plugin 업그레이드 실패에서만 설치 흐름이 복구되도록 허용합니다. 관련 없는 구성 오류는 여전히 설치를 차단하고 운영자를 `openclaw doctor --fix`로 안내합니다.

`openclaw.channel.persistedAuthState`는 작은 검사기 모듈을 위한 package 메타데이터입니다.

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

설정, doctor 또는 configured-state 흐름이 전체 channel plugin이 로드되기 전에 저비용 예/아니오 auth 프로브를 필요로 할 때 사용하세요. 대상 export는 저장된 상태만 읽는 작은 함수여야 하며, 전체 channel 런타임 barrel을 통해 연결해서는 안 됩니다.

`openclaw.channel.configuredState`는 저비용 env 전용 configured 검사를 위해 같은 형태를 따릅니다.

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

channel이 env 또는 기타 작은 비런타임 입력만으로 configured-state에 답할 수 있을 때 사용하세요. 검사가 전체 config 해석이나 실제 channel 런타임을 필요로 한다면, 대신 그 로직을 plugin `config.hasConfiguredState` hook에 두세요.

## JSON Schema 요구 사항

- **모든 plugin은 JSON Schema를 반드시 포함해야 하며**, 구성을 전혀 받지 않더라도 예외는 없습니다.
- 빈 스키마도 허용됩니다(예: `{ "type": "object", "additionalProperties": false }`).
- 스키마는 런타임이 아니라 구성 읽기/쓰기 시점에 검증됩니다.

## 검증 동작

- 알 수 없는 `channels.*` 키는 **오류**입니다. 단, 해당 channel id가 plugin 매니페스트에 선언된 경우는 예외입니다.
- `plugins.entries.<id>`, `plugins.allow`, `plugins.deny`, `plugins.slots.*`는 **검색 가능한** plugin id를 참조해야 합니다. 알 수 없는 id는 **오류**입니다.
- plugin이 설치되어 있지만 매니페스트나 스키마가 깨졌거나 누락된 경우 검증이 실패하고 Doctor가 plugin 오류를 보고합니다.
- plugin 구성이 존재하지만 plugin이 **비활성화**되어 있으면 구성은 유지되며 Doctor + 로그에 **경고**가 표시됩니다.

전체 `plugins.*` 스키마는 [Configuration reference](/ko/gateway/configuration)를 참조하세요.

## 참고

- 매니페스트는 로컬 파일 시스템 로드를 포함해 **네이티브 OpenClaw plugin에 필수**입니다.
- 런타임은 여전히 plugin 모듈을 별도로 로드합니다. 매니페스트는 검색 + 검증 전용입니다.
- 네이티브 매니페스트는 JSON5로 파싱되므로 최종 값이 여전히 객체이기만 하면 주석, 후행 쉼표, 따옴표 없는 키를 허용합니다.
- 매니페스트 로더는 문서화된 매니페스트 필드만 읽습니다. 여기에 사용자 정의 최상위 키를 추가하지 마세요.
- `providerAuthEnvVars`는 env 이름을 확인하기 위해 plugin 런타임을 부팅해서는 안 되는 auth 프로브, env-marker 검증 및 유사한 provider-auth 표면을 위한 저비용 메타데이터 경로입니다.
- `providerAuthAliases`는 core에 관계를 하드코딩하지 않고도 provider 변형이 다른 provider의 auth env vars, auth 프로필, config 기반 auth, API 키 onboarding 선택지를 재사용할 수 있게 합니다.
- `channelEnvVars`는 env 이름을 확인하기 위해 plugin 런타임을 부팅해서는 안 되는 shell-env 폴백, setup 프롬프트 및 유사한 channel 표면을 위한 저비용 메타데이터 경로입니다.
- `providerAuthChoices`는 provider 런타임이 로드되기 전에 auth-choice 선택기, `--auth-choice` 해석, 선호 provider 매핑, 간단한 onboarding CLI 플래그 등록을 위한 저비용 메타데이터 경로입니다. provider 코드가 필요한 런타임 wizard 메타데이터는 [Provider runtime hooks](/ko/plugins/architecture#provider-runtime-hooks)를 참조하세요.
- 배타적 plugin kind는 `plugins.slots.*`를 통해 선택됩니다.
  - `kind: "memory"`는 `plugins.slots.memory`로 선택됩니다.
  - `kind: "context-engine"`는 `plugins.slots.contextEngine`으로 선택됩니다
    (기본값: 내장 `legacy`).
- `channels`, `providers`, `cliBackends`, `skills`는 plugin에 필요하지 않다면 생략할 수 있습니다.
- plugin이 네이티브 모듈에 의존하는 경우, 빌드 단계와 package-manager 허용 목록 요구 사항(예: pnpm `allow-build-scripts`
  - `pnpm rebuild <package>`)을 문서화하세요.

## 관련 문서

- [Building Plugins](/ko/plugins/building-plugins) — plugin 시작 가이드
- [Plugin Architecture](/ko/plugins/architecture) — 내부 아키텍처
- [SDK Overview](/ko/plugins/sdk-overview) — Plugin SDK 참조
