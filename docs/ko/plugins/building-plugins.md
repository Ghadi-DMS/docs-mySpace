---
read_when:
    - 새 OpenClaw plugin을 만들고 싶습니다
    - plugin 개발을 위한 빠른 시작 가이드가 필요합니다
    - OpenClaw에 새 채널, provider, 도구 또는 기타 capability를 추가하고 있습니다
sidebarTitle: Getting Started
summary: 몇 분 만에 첫 OpenClaw plugin 만들기
title: Plugin 만들기
x-i18n:
    generated_at: "2026-04-06T03:09:27Z"
    model: gpt-5.4
    provider: openai
    source_hash: 9be344cb300ecbcba08e593a95bcc93ab16c14b28a0ff0c29b26b79d8249146c
    source_path: plugins/building-plugins.md
    workflow: 15
---

# Plugin 만들기

Plugins는 새 capability로 OpenClaw를 확장합니다: 채널, 모델 providers,
speech, realtime transcription, realtime voice, media understanding, image
generation, video generation, web fetch, web search, agent tools 또는
이들의 조합입니다.

plugin을 OpenClaw 리포지토리에 추가할 필요는 없습니다.
[ClawHub](/ko/tools/clawhub) 또는 npm에 게시하면 사용자는
`openclaw plugins install <package-name>`으로 설치합니다. OpenClaw는 먼저 ClawHub를 시도하고
자동으로 npm으로 폴백합니다.

## 사전 요구 사항

- Node >= 22 및 패키지 관리자(npm 또는 pnpm)
- TypeScript(ESM)에 대한 익숙함
- 리포지토리 내 plugins의 경우: 리포지토리 클론 및 `pnpm install` 완료

## 어떤 종류의 plugin인가요?

<CardGroup cols={3}>
  <Card title="Channel plugin" icon="messages-square" href="/ko/plugins/sdk-channel-plugins">
    OpenClaw를 메시징 플랫폼(Discord, IRC 등)에 연결
  </Card>
  <Card title="Provider plugin" icon="cpu" href="/ko/plugins/sdk-provider-plugins">
    모델 provider(LLM, 프록시 또는 사용자 지정 엔드포인트) 추가
  </Card>
  <Card title="Tool / hook plugin" icon="wrench">
    agent 도구, 이벤트 hooks 또는 서비스 등록 — 아래 계속
  </Card>
</CardGroup>

채널 plugin이 선택 사항이며 onboarding/setup 실행 시 설치되어 있지 않을 수도 있다면
`openclaw/plugin-sdk/channel-setup`의
`createOptionalChannelSetupSurface(...)`를 사용하세요. 이것은 설치 요구 사항을 알리고
plugin이 설치될 때까지 실제 config 쓰기에서 안전하게 닫히도록 하는
setup adapter + wizard 쌍을 생성합니다.

## 빠른 시작: 도구 plugin

이 안내에서는 agent 도구를 등록하는 최소 plugin을 만듭니다. 채널
및 provider plugins에는 위에 링크된 전용 가이드가 있습니다.

<Steps>
  <Step title="패키지와 manifest 만들기">
    <CodeGroup>
    ```json package.json
    {
      "name": "@myorg/openclaw-my-plugin",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "compat": {
          "pluginApi": ">=2026.3.24-beta.2",
          "minGatewayVersion": "2026.3.24-beta.2"
        },
        "build": {
          "openclawVersion": "2026.3.24-beta.2",
          "pluginSdkVersion": "2026.3.24-beta.2"
        }
      }
    }
    ```

    ```json openclaw.plugin.json
    {
      "id": "my-plugin",
      "name": "My Plugin",
      "description": "Adds a custom tool to OpenClaw",
      "configSchema": {
        "type": "object",
        "additionalProperties": false
      }
    }
    ```
    </CodeGroup>

    모든 plugin에는 config가 없어도 manifest가 필요합니다.
    전체 스키마는 [Manifest](/ko/plugins/manifest)를 참조하세요. 정식 ClawHub
    게시 스니펫은 `docs/snippets/plugin-publish/`에 있습니다.

  </Step>

  <Step title="entry point 작성">

    ```typescript
    // index.ts
    import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
    import { Type } from "@sinclair/typebox";

    export default definePluginEntry({
      id: "my-plugin",
      name: "My Plugin",
      description: "Adds a custom tool to OpenClaw",
      register(api) {
        api.registerTool({
          name: "my_tool",
          description: "Do a thing",
          parameters: Type.Object({ input: Type.String() }),
          async execute(_id, params) {
            return { content: [{ type: "text", text: `Got: ${params.input}` }] };
          },
        });
      },
    });
    ```

    `definePluginEntry`는 채널이 아닌 plugins용입니다. 채널의 경우
    `defineChannelPluginEntry`를 사용하세요 — [Channel Plugins](/ko/plugins/sdk-channel-plugins)를 참조하세요.
    전체 entry point 옵션은 [Entry Points](/ko/plugins/sdk-entrypoints)를 참조하세요.

  </Step>

  <Step title="테스트 및 게시">

    **외부 plugins:** ClawHub로 검증 및 게시한 다음 설치합니다:

    ```bash
    clawhub package publish your-org/your-plugin --dry-run
    clawhub package publish your-org/your-plugin
    openclaw plugins install clawhub:@myorg/openclaw-my-plugin
    ```

    OpenClaw는 `@myorg/openclaw-my-plugin` 같은
    기본 package spec에 대해서도 npm보다 먼저 ClawHub를 확인합니다.

    **리포지토리 내 plugins:** 번들 plugin workspace 트리 아래에 두면 자동으로 검색됩니다.

    ```bash
    pnpm test -- <bundled-plugin-root>/my-plugin/
    ```

  </Step>
</Steps>

## Plugin capability

단일 plugin은 `api` 객체를 통해 원하는 수의 capability를 등록할 수 있습니다:

| Capability             | 등록 메서드                                      | 상세 가이드                                                                   |
| ---------------------- | ------------------------------------------------ | ----------------------------------------------------------------------------- |
| 텍스트 추론(LLM)       | `api.registerProvider(...)`                      | [Provider Plugins](/ko/plugins/sdk-provider-plugins)                             |
| 채널 / 메시징          | `api.registerChannel(...)`                       | [Channel Plugins](/ko/plugins/sdk-channel-plugins)                               |
| Speech (TTS/STT)       | `api.registerSpeechProvider(...)`                | [Provider Plugins](/ko/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| Realtime transcription | `api.registerRealtimeTranscriptionProvider(...)` | [Provider Plugins](/ko/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| Realtime voice         | `api.registerRealtimeVoiceProvider(...)`         | [Provider Plugins](/ko/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| Media understanding    | `api.registerMediaUnderstandingProvider(...)`    | [Provider Plugins](/ko/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| Image generation       | `api.registerImageGenerationProvider(...)`       | [Provider Plugins](/ko/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| Music generation       | `api.registerMusicGenerationProvider(...)`       | [Provider Plugins](/ko/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| Video generation       | `api.registerVideoGenerationProvider(...)`       | [Provider Plugins](/ko/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| Web fetch              | `api.registerWebFetchProvider(...)`              | [Provider Plugins](/ko/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| Web search             | `api.registerWebSearchProvider(...)`             | [Provider Plugins](/ko/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| Agent tools            | `api.registerTool(...)`                          | 아래                                                                          |
| 사용자 지정 commands   | `api.registerCommand(...)`                       | [Entry Points](/ko/plugins/sdk-entrypoints)                                      |
| 이벤트 hooks           | `api.registerHook(...)`                          | [Entry Points](/ko/plugins/sdk-entrypoints)                                      |
| HTTP routes            | `api.registerHttpRoute(...)`                     | [Internals](/ko/plugins/architecture#gateway-http-routes)                        |
| CLI 하위 명령          | `api.registerCli(...)`                           | [Entry Points](/ko/plugins/sdk-entrypoints)                                      |

전체 등록 API는 [SDK 개요](/ko/plugins/sdk-overview#registration-api)를 참조하세요.

plugin이 사용자 지정 gateway RPC 메서드를 등록하는 경우
plugin별 접두사를 유지하세요. 코어 admin 네임스페이스(`config.*`,
`exec.approvals.*`, `wizard.*`, `update.*`)는 예약되어 있으며
plugin이 더 좁은 범위를 요청하더라도 항상 `operator.admin`으로 해석됩니다.

기억해야 할 hook guard 의미론:

- `before_tool_call`: `{ block: true }`는 종결 결정이며 더 낮은 우선순위 핸들러를 중지합니다.
- `before_tool_call`: `{ block: false }`는 결정 없음으로 처리됩니다.
- `before_tool_call`: `{ requireApproval: true }`는 agent 실행을 일시 중지하고 exec approval 오버레이, Telegram 버튼, Discord 상호작용 또는 모든 채널의 `/approve` command를 통해 사용자 승인을 요청합니다.
- `before_install`: `{ block: true }`는 종결 결정이며 더 낮은 우선순위 핸들러를 중지합니다.
- `before_install`: `{ block: false }`는 결정 없음으로 처리됩니다.
- `message_sending`: `{ cancel: true }`는 종결 결정이며 더 낮은 우선순위 핸들러를 중지합니다.
- `message_sending`: `{ cancel: false }`는 결정 없음으로 처리됩니다.

`/approve` command는 bounded fallback으로 exec approval과 plugin approval을 모두 처리합니다. exec approval id를 찾지 못하면 OpenClaw는 같은 id로 plugin approvals를 다시 시도합니다. Plugin approval 전달은 config의 `approvals.plugin`을 통해 독립적으로 구성할 수 있습니다.

사용자 지정 approval 배선에서 동일한 bounded fallback 사례를 감지해야 한다면,
approval-expiry 문자열을 수동으로 비교하는 대신
`openclaw/plugin-sdk/error-runtime`의 `isApprovalNotFoundError`를 사용하세요.

자세한 내용은 [SDK 개요 hook 결정 의미론](/ko/plugins/sdk-overview#hook-decision-semantics)을 참조하세요.

## agent 도구 등록

도구는 LLM이 호출할 수 있는 타입 지정 함수입니다. 필수(항상 사용 가능)일 수도 있고
선택적(사용자 옵트인)일 수도 있습니다:

```typescript
register(api) {
  // 필수 도구 — 항상 사용 가능
  api.registerTool({
    name: "my_tool",
    description: "Do a thing",
    parameters: Type.Object({ input: Type.String() }),
    async execute(_id, params) {
      return { content: [{ type: "text", text: params.input }] };
    },
  });

  // 선택적 도구 — 사용자가 allowlist에 추가해야 함
  api.registerTool(
    {
      name: "workflow_tool",
      description: "Run a workflow",
      parameters: Type.Object({ pipeline: Type.String() }),
      async execute(_id, params) {
        return { content: [{ type: "text", text: params.pipeline }] };
      },
    },
    { optional: true },
  );
}
```

사용자는 config에서 선택적 도구를 활성화합니다:

```json5
{
  tools: { allow: ["workflow_tool"] },
}
```

- 도구 이름은 코어 도구와 충돌하면 안 됩니다(충돌 시 건너뜀)
- 부작용이나 추가 바이너리 요구 사항이 있는 도구에는 `optional: true`를 사용하세요
- 사용자는 `tools.allow`에 plugin id를 추가해 plugin의 모든 도구를 활성화할 수 있습니다

## import 규칙

항상 집중된 `openclaw/plugin-sdk/<subpath>` 경로에서 import하세요:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";

// 잘못된 예: 단일 루트(monolithic root) (deprecated, 제거 예정)
import { ... } from "openclaw/plugin-sdk";
```

전체 하위 경로 참조는 [SDK 개요](/ko/plugins/sdk-overview)를 참조하세요.

plugin 내부에서는 내부 import에 로컬 배럴 파일(`api.ts`, `runtime-api.ts`)을 사용하세요 —
자신의 plugin을 SDK 경로를 통해 import하지 마세요.

provider plugins의 경우 provider별 helper는 진짜로 일반적인 seam이 아닌 한
그 패키지 루트 배럴에 두세요. 현재 번들 예시는 다음과 같습니다:

- Anthropic: Claude stream 래퍼와 `service_tier` / beta helpers
- OpenAI: provider builders, 기본 모델 helpers, realtime providers
- OpenRouter: provider builder와 onboarding/config helpers

helper가 하나의 번들 provider 패키지 안에서만 유용하다면
`openclaw/plugin-sdk/*`로 승격하는 대신 해당 패키지 루트 seam에 두세요.

일부 생성된 `openclaw/plugin-sdk/<bundled-id>` helper seam은 번들 plugin 유지 관리와 호환성을 위해 여전히 존재합니다. 예를 들어
`plugin-sdk/feishu-setup` 또는 `plugin-sdk/zalo-setup`이 있습니다. 이것들은
새 서드파티 plugins의 기본 패턴이 아니라 예약된 표면으로 취급하세요.

## 제출 전 체크리스트

<Check>**package.json**에 올바른 `openclaw` 메타데이터가 있음</Check>
<Check>**openclaw.plugin.json** manifest가 존재하고 유효함</Check>
<Check>entry point가 `defineChannelPluginEntry` 또는 `definePluginEntry`를 사용함</Check>
<Check>모든 imports가 집중된 `plugin-sdk/<subpath>` 경로를 사용함</Check>
<Check>내부 imports가 SDK 자기 import가 아닌 로컬 모듈을 사용함</Check>
<Check>테스트 통과 (`pnpm test -- <bundled-plugin-root>/my-plugin/`)</Check>
<Check>`pnpm check` 통과 (리포지토리 내 plugins)</Check>

## Beta 릴리스 테스트

1. [openclaw/openclaw](https://github.com/openclaw/openclaw/releases)의 GitHub 릴리스 태그를 주시하고 `Watch` > `Releases`를 통해 구독하세요. Beta 태그는 `v2026.3.N-beta.1`처럼 보입니다. 릴리스 공지를 위해 공식 OpenClaw X 계정 [@openclaw](https://x.com/openclaw)의 알림을 켤 수도 있습니다.
2. beta 태그가 나타나는 즉시 plugin을 테스트하세요. stable 전까지의 창은 보통 몇 시간밖에 되지 않습니다.
3. 테스트 후 `plugin-forum` Discord 채널의 plugin 스레드에 `all good` 또는 문제가 무엇이었는지 게시하세요. 아직 스레드가 없다면 하나 만드세요.
4. 문제가 생기면 `Beta blocker: <plugin-name> - <summary>` 제목의 이슈를 열거나 업데이트하고 `beta-blocker` label을 적용하세요. 해당 이슈 링크를 스레드에 넣으세요.
5. `main`으로 `fix(<plugin-id>): beta blocker - <summary>` 제목의 PR을 열고 PR과 Discord 스레드 둘 다에 이슈를 링크하세요. 기여자는 PR에 label을 붙일 수 없으므로 제목이 관리자와 자동화를 위한 PR 측 신호입니다. PR이 있는 blocker는 병합되고, 없는 blocker는 그대로 배포될 수도 있습니다. 관리자들은 beta 테스트 중 이 스레드를 지켜봅니다.
6. 침묵은 녹색 신호입니다. 이 창을 놓치면 수정은 다음 주기에 반영될 가능성이 높습니다.

## 다음 단계

<CardGroup cols={2}>
  <Card title="Channel Plugins" icon="messages-square" href="/ko/plugins/sdk-channel-plugins">
    메시징 채널 plugin 만들기
  </Card>
  <Card title="Provider Plugins" icon="cpu" href="/ko/plugins/sdk-provider-plugins">
    모델 provider plugin 만들기
  </Card>
  <Card title="SDK Overview" icon="book-open" href="/ko/plugins/sdk-overview">
    import 맵 및 등록 API 참조
  </Card>
  <Card title="Runtime Helpers" icon="settings" href="/ko/plugins/sdk-runtime">
    api.runtime를 통한 TTS, 검색, subagent
  </Card>
  <Card title="Testing" icon="test-tubes" href="/ko/plugins/sdk-testing">
    테스트 유틸리티 및 패턴
  </Card>
  <Card title="Plugin Manifest" icon="file-json" href="/ko/plugins/manifest">
    전체 manifest 스키마 참조
  </Card>
</CardGroup>

## 관련

- [Plugin Architecture](/ko/plugins/architecture) — 내부 아키텍처 심화 설명
- [SDK Overview](/ko/plugins/sdk-overview) — Plugin SDK 참조
- [Manifest](/ko/plugins/manifest) — plugin manifest 형식
- [Channel Plugins](/ko/plugins/sdk-channel-plugins) — 채널 plugins 만들기
- [Provider Plugins](/ko/plugins/sdk-provider-plugins) — provider plugins 만들기
