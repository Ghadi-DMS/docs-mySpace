---
read_when:
    - plugin에 설정 마법사를 추가하고 있을 때
    - setup-entry.ts와 index.ts의 차이를 이해해야 할 때
    - plugin config 스키마 또는 package.json의 openclaw 메타데이터를 정의하고 있을 때
sidebarTitle: Setup and Config
summary: 설정 마법사, setup-entry.ts, config 스키마, package.json 메타데이터
title: Plugin 설정 및 Config
x-i18n:
    generated_at: "2026-04-06T03:10:57Z"
    model: gpt-5.4
    provider: openai
    source_hash: eac2586516d27bcd94cc4c259fe6274c792b3f9938c7ddd6dbf04a6dbb988dc9
    source_path: plugins/sdk-setup.md
    workflow: 15
---

# Plugin 설정 및 Config

plugin 패키징(`package.json` 메타데이터), 매니페스트
(`openclaw.plugin.json`), setup entry, config 스키마에 대한 참조입니다.

<Tip>
  **단계별 가이드를 찾고 있나요?** how-to 가이드는 컨텍스트 안에서 패키징을 다룹니다.
  [Channel Plugins](/ko/plugins/sdk-channel-plugins#step-1-package-and-manifest) 및
  [Provider Plugins](/ko/plugins/sdk-provider-plugins#step-1-package-and-manifest)을 참고하세요.
</Tip>

## 패키지 메타데이터

`package.json`에는 plugin 시스템에
plugin이 무엇을 제공하는지 알려주는 `openclaw` 필드가 필요합니다.

**채널 plugin:**

```json
{
  "name": "@myorg/openclaw-my-channel",
  "version": "1.0.0",
  "type": "module",
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "channel": {
      "id": "my-channel",
      "label": "My Channel",
      "blurb": "Short description of the channel."
    }
  }
}
```

**프로바이더 plugin / ClawHub 게시 기준:**

```json openclaw-clawhub-package.json
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

plugin을 ClawHub에 외부 게시하는 경우 해당 `compat` 및 `build`
필드는 필수입니다. 정식 게시 스니펫은
`docs/snippets/plugin-publish/`에 있습니다.

### `openclaw` 필드

| 필드         | 타입       | 설명                                                                                                     |
| ------------ | ---------- | -------------------------------------------------------------------------------------------------------- |
| `extensions` | `string[]` | entry point 파일(패키지 루트 기준 상대 경로)                                                            |
| `setupEntry` | `string`   | 가벼운 setup 전용 entry(선택 사항)                                                                       |
| `channel`    | `object`   | setup, picker, quickstart, 상태 표면을 위한 채널 카탈로그 메타데이터                                     |
| `providers`  | `string[]` | 이 plugin이 등록하는 provider ID                                                                         |
| `install`    | `object`   | 설치 힌트: `npmSpec`, `localPath`, `defaultChoice`, `minHostVersion`, `allowInvalidConfigRecovery`      |
| `startup`    | `object`   | 시작 동작 플래그                                                                                         |

### `openclaw.channel`

`openclaw.channel`은 런타임이 로드되기 전에 채널 검색 및 setup
표면을 위한 가벼운 패키지 메타데이터입니다.

| 필드                                   | 타입       | 의미                                                                          |
| -------------------------------------- | ---------- | ----------------------------------------------------------------------------- |
| `id`                                   | `string`   | 정식 채널 ID                                                                  |
| `label`                                | `string`   | 기본 채널 레이블                                                              |
| `selectionLabel`                       | `string`   | `label`과 달라야 할 때의 picker/setup 레이블                                 |
| `detailLabel`                          | `string`   | 더 풍부한 채널 카탈로그 및 상태 표면을 위한 보조 상세 레이블                 |
| `docsPath`                             | `string`   | setup 및 선택 링크용 문서 경로                                                |
| `docsLabel`                            | `string`   | 문서 링크 레이블이 채널 ID와 달라야 할 때 사용하는 재정의 레이블             |
| `blurb`                                | `string`   | 짧은 온보딩/카탈로그 설명                                                     |
| `order`                                | `number`   | 채널 카탈로그에서의 정렬 순서                                                 |
| `aliases`                              | `string[]` | 채널 선택을 위한 추가 조회 별칭                                               |
| `preferOver`                           | `string[]` | 이 채널이 더 높은 우선순위를 가져야 하는 낮은 우선순위 plugin/channel ID      |
| `systemImage`                          | `string`   | 채널 UI 카탈로그용 선택적 아이콘/system-image 이름                            |
| `selectionDocsPrefix`                  | `string`   | 선택 표면에서 문서 링크 앞에 붙는 접두 텍스트                                 |
| `selectionDocsOmitLabel`               | `boolean`  | 선택 문구에서 레이블이 붙은 문서 링크 대신 문서 경로를 직접 표시              |
| `selectionExtras`                      | `string[]` | 선택 문구에 추가되는 짧은 문자열                                              |
| `markdownCapable`                      | `boolean`  | 아웃바운드 포맷 결정에서 이 채널이 markdown 가능함을 표시                     |
| `exposure`                             | `object`   | setup, 구성 목록, 문서 표면용 채널 가시성 제어                                |
| `quickstartAllowFrom`                  | `boolean`  | 이 채널을 표준 quickstart `allowFrom` setup 흐름에 포함                       |
| `forceAccountBinding`                  | `boolean`  | 계정이 하나뿐이어도 명시적 account binding을 요구                             |
| `preferSessionLookupForAnnounceTarget` | `boolean`  | 이 채널의 announce 대상 확인 시 session lookup을 우선                         |

예시:

```json
{
  "openclaw": {
    "channel": {
      "id": "my-channel",
      "label": "My Channel",
      "selectionLabel": "My Channel (self-hosted)",
      "detailLabel": "My Channel Bot",
      "docsPath": "/channels/my-channel",
      "docsLabel": "my-channel",
      "blurb": "Webhook-based self-hosted chat integration.",
      "order": 80,
      "aliases": ["mc"],
      "preferOver": ["my-channel-legacy"],
      "selectionDocsPrefix": "Guide:",
      "selectionExtras": ["Markdown"],
      "markdownCapable": true,
      "exposure": {
        "configured": true,
        "setup": true,
        "docs": true
      },
      "quickstartAllowFrom": true
    }
  }
}
```

`exposure`는 다음을 지원합니다.

- `configured`: 구성됨/상태 스타일 목록 표면에 채널 포함
- `setup`: 대화형 setup/configure picker에 채널 포함
- `docs`: 문서/내비게이션 표면에서 채널을 공개 대상으로 표시

`showConfigured`와 `showInSetup`도 레거시 별칭으로 계속 지원됩니다. 가능하면
`exposure`를 우선하세요.

### `openclaw.install`

`openclaw.install`은 매니페스트 메타데이터가 아니라 패키지 메타데이터입니다.

| 필드                         | 타입                 | 의미                                                                                     |
| ---------------------------- | -------------------- | ---------------------------------------------------------------------------------------- |
| `npmSpec`                    | `string`             | 설치/업데이트 흐름을 위한 정식 npm spec                                                  |
| `localPath`                  | `string`             | 로컬 개발 또는 번들 설치 경로                                                            |
| `defaultChoice`              | `"npm"` \| `"local"` | 둘 다 가능할 때 선호되는 설치 소스                                                       |
| `minHostVersion`             | `string`             | `>=x.y.z` 형식의 최소 지원 OpenClaw 버전                                                 |
| `allowInvalidConfigRecovery` | `boolean`            | 번들 plugin 재설치 흐름이 특정 오래된 config 실패에서 복구하도록 허용                    |

`minHostVersion`이 설정되면 설치와 매니페스트 레지스트리 로딩 모두에서
이를 강제합니다. 더 오래된 호스트는 plugin을 건너뛰며, 잘못된 버전 문자열은 거부됩니다.

`allowInvalidConfigRecovery`는 망가진 config를 위한 일반적인 우회 수단이 아닙니다.
이는 범위가 좁은 번들 plugin 복구 전용으로,
재설치/setup이 누락된 번들 plugin 경로나 같은 plugin에 대한 오래된 `channels.<id>`
항목 같은 알려진 업그레이드 잔재를 복구하도록 하기 위한 것입니다.
관련 없는 이유로 config가 망가진 경우 설치는 여전히 fail closed로 처리되고
운영자에게 `openclaw doctor --fix`를 실행하라고 안내합니다.

### 지연된 전체 로드

채널 plugins는 다음과 같이 지연 로드를 선택할 수 있습니다.

```json
{
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "startup": {
      "deferConfiguredChannelFullLoadUntilAfterListen": true
    }
  }
}
```

활성화하면 OpenClaw는 이미 구성된 채널에 대해서도
pre-listen 시작 단계에서는 `setupEntry`만 로드합니다.
전체 entry는 gateway가 listen을 시작한 뒤에 로드됩니다.

<Warning>
  지연 로드는 `setupEntry`가 gateway가 listen을 시작하기 전에 필요한 모든 것을
  등록하는 경우에만 활성화하세요(채널 등록, HTTP 경로, gateway 메서드).
  전체 entry가 필수 시작 기능을 소유한다면 기본 동작을 유지하세요.
</Warning>

setup/full entry가 gateway RPC 메서드를 등록한다면
plugin 전용 접두사를 유지하세요. 예약된 코어 관리 네임스페이스(`config.*`,
`exec.approvals.*`, `wizard.*`, `update.*`)는 계속 코어 소유이며 항상
`operator.admin`으로 확인됩니다.

## Plugin 매니페스트

모든 네이티브 plugin은 패키지 루트에 `openclaw.plugin.json`을 포함해야 합니다.
OpenClaw는 이를 사용해 plugin 코드를 실행하지 않고도 config를 검증합니다.

```json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "description": "Adds My Plugin capabilities to OpenClaw",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "webhookSecret": {
        "type": "string",
        "description": "Webhook verification secret"
      }
    }
  }
}
```

채널 plugin의 경우 `kind`와 `channels`를 추가하세요.

```json
{
  "id": "my-channel",
  "kind": "channel",
  "channels": ["my-channel"],
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

config가 없는 plugin도 스키마를 반드시 포함해야 합니다.
빈 스키마도 유효합니다.

```json
{
  "id": "my-plugin",
  "configSchema": {
    "type": "object",
    "additionalProperties": false
  }
}
```

전체 스키마 참조는 [Plugin Manifest](/ko/plugins/manifest)를 참고하세요.

## ClawHub 게시

plugin 패키지에는 패키지 전용 ClawHub 명령을 사용하세요.

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

레거시 skill 전용 게시 별칭은 Skills용입니다. Plugin 패키지는
항상 `clawhub package publish`를 사용해야 합니다.

## Setup entry

`setup-entry.ts` 파일은 OpenClaw가 setup 표면만 필요할 때
로드하는 `index.ts`의 가벼운 대안입니다(온보딩, config 복구,
비활성화된 채널 검사).

```typescript
// setup-entry.ts
import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
import { myChannelPlugin } from "./src/channel.js";

export default defineSetupPluginEntry(myChannelPlugin);
```

이렇게 하면 setup 흐름 중에 무거운 런타임 코드(암호화 라이브러리, CLI 등록,
백그라운드 서비스)를 로드하지 않아도 됩니다.

**OpenClaw가 전체 entry 대신 `setupEntry`를 사용하는 경우:**

- 채널이 비활성화되었지만 setup/온보딩 표면이 필요한 경우
- 채널이 활성화되었지만 아직 구성되지 않은 경우
- 지연 로드가 활성화된 경우(`deferConfiguredChannelFullLoadUntilAfterListen`)

**`setupEntry`가 등록해야 하는 것:**

- 채널 plugin 객체(`defineSetupPluginEntry`를 통해)
- gateway listen 전에 필요한 모든 HTTP 경로
- 시작 중 필요한 모든 gateway 메서드

이러한 시작용 gateway 메서드도 `config.*`나 `update.*` 같은
예약된 코어 관리 네임스페이스는 피해야 합니다.

**`setupEntry`에 포함하면 안 되는 것:**

- CLI 등록
- 백그라운드 서비스
- 무거운 런타임 import(crypto, SDK)
- 시작 후에만 필요한 gateway 메서드

### 좁은 setup helper import

핫 setup 전용 경로에서는 setup 표면의 일부만 필요할 때
더 넓은 `plugin-sdk/setup` umbrella 대신 좁은 setup helper 연결점을 우선하세요.

| import 경로                        | 사용 목적                                                                             | 주요 export                                                                                                                                                                                                                                                                                  |
| ---------------------------------- | ------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/setup-runtime`         | `setupEntry` / 지연 채널 시작에서도 계속 사용할 수 있는 setup 시점 런타임 helper      | `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
| `plugin-sdk/setup-adapter-runtime` | 환경 인지형 account setup 어댑터                                                      | `createEnvPatchedAccountSetupAdapter`                                                                                                                                                                                                                                                        |
| `plugin-sdk/setup-tools`           | setup/install CLI/archive/docs helper                                                 | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR`                                                                                                                                                                               |

config 패치 helper인
`moveSingleAccountChannelSectionToDefaultAccount(...)`를 포함한 전체 공유 setup
도구 상자가 필요하다면 더 넓은 `plugin-sdk/setup` 연결점을 사용하세요.

setup 패치 어댑터는 import 시에도 핫 경로 안전성을 유지합니다. 이들의 번들
single-account 승격 contract-surface 조회는 지연 실행되므로,
`plugin-sdk/setup-runtime`을 import해도 어댑터를 실제로 사용하기 전에
번들 contract-surface 검색을 즉시 로드하지는 않습니다.

### 채널 소유 single-account 승격

채널이 단일 account 최상위 config에서
`channels.<id>.accounts.*`로 업그레이드될 때, 기본 공유 동작은
승격된 account 범위 값을 `accounts.default`로 이동하는 것입니다.

번들 채널은 setup
contract surface를 통해 이 승격을 좁히거나 재정의할 수 있습니다.

- `singleAccountKeysToMove`: 승격된 account로 이동해야 하는 추가 최상위 키
- `namedAccountPromotionKeys`: 이름 있는 account가 이미 존재할 때는 이
  키만 승격된 account로 이동하고, 공유 정책/전달 키는 채널 루트에 유지
- `resolveSingleAccountPromotionTarget(...)`: 승격된 값을 받을 기존 account 선택

현재 번들 예시는 Matrix입니다. 정확히 하나의 이름 있는 Matrix account가
이미 존재하거나 `defaultAccount`가 `Ops` 같은 기존의 비정규 키를 가리키는 경우,
승격은 새 `accounts.default` 항목을 만드는 대신 해당 account를 유지합니다.

## Config 스키마

plugin config는 매니페스트의 JSON Schema에 대해 검증됩니다. 사용자는 다음과 같이
plugin을 구성합니다.

```json5
{
  plugins: {
    entries: {
      "my-plugin": {
        config: {
          webhookSecret: "abc123",
        },
      },
    },
  },
}
```

등록 중 plugin은 이 config를 `api.pluginConfig`로 받습니다.

채널 전용 config에는 대신 채널 config 섹션을 사용하세요.

```json5
{
  channels: {
    "my-channel": {
      token: "bot-token",
      allowFrom: ["user1", "user2"],
    },
  },
}
```

### 채널 config 스키마 만들기

`openclaw/plugin-sdk/core`의 `buildChannelConfigSchema`를 사용해
Zod 스키마를 OpenClaw가 검증하는 `ChannelConfigSchema` 래퍼로 변환하세요.

```typescript
import { z } from "zod";
import { buildChannelConfigSchema } from "openclaw/plugin-sdk/core";

const accountSchema = z.object({
  token: z.string().optional(),
  allowFrom: z.array(z.string()).optional(),
  accounts: z.object({}).catchall(z.any()).optional(),
  defaultAccount: z.string().optional(),
});

const configSchema = buildChannelConfigSchema(accountSchema);
```

## 설정 마법사

채널 plugins는 `openclaw onboard`용 대화형 설정 마법사를 제공할 수 있습니다.
마법사는 `ChannelPlugin`의 `ChannelSetupWizard` 객체입니다.

```typescript
import type { ChannelSetupWizard } from "openclaw/plugin-sdk/channel-setup";

const setupWizard: ChannelSetupWizard = {
  channel: "my-channel",
  status: {
    configuredLabel: "Connected",
    unconfiguredLabel: "Not configured",
    resolveConfigured: ({ cfg }) => Boolean((cfg.channels as any)?.["my-channel"]?.token),
  },
  credentials: [
    {
      inputKey: "token",
      providerHint: "my-channel",
      credentialLabel: "Bot token",
      preferredEnvVar: "MY_CHANNEL_BOT_TOKEN",
      envPrompt: "Use MY_CHANNEL_BOT_TOKEN from environment?",
      keepPrompt: "Keep current token?",
      inputPrompt: "Enter your bot token:",
      inspect: ({ cfg, accountId }) => {
        const token = (cfg.channels as any)?.["my-channel"]?.token;
        return {
          accountConfigured: Boolean(token),
          hasConfiguredValue: Boolean(token),
        };
      },
    },
  ],
};
```

`ChannelSetupWizard` 타입은 `credentials`, `textInputs`,
`dmPolicy`, `allowFrom`, `groupAccess`, `prepare`, `finalize` 등을 지원합니다.
전체 예시는 번들 plugin 패키지(예: Discord plugin의 `src/channel.setup.ts`)를 참고하세요.

표준
`note -> prompt -> parse -> merge -> patch` 흐름만 필요한 DM 허용 목록 프롬프트에는
`openclaw/plugin-sdk/setup`의 공유 setup
helper인 `createPromptParsedAllowFromForAccount(...)`,
`createTopLevelChannelParsedAllowFromPrompt(...)`,
`createNestedChannelParsedAllowFromPrompt(...)`를 우선 사용하세요.

레이블, 점수, 선택적 추가 줄만 다른 채널 setup 상태 블록에는
각 plugin에서 동일한 `status` 객체를 직접 구현하는 대신
`openclaw/plugin-sdk/setup`의 `createStandardChannelSetupStatus(...)`를 우선 사용하세요.

특정 컨텍스트에서만 나타나야 하는 선택적 setup 표면에는
`openclaw/plugin-sdk/channel-setup`의 `createOptionalChannelSetupSurface`를 사용하세요.

```typescript
import { createOptionalChannelSetupSurface } from "openclaw/plugin-sdk/channel-setup";

const setupSurface = createOptionalChannelSetupSurface({
  channel: "my-channel",
  label: "My Channel",
  npmSpec: "@myorg/openclaw-my-channel",
  docsPath: "/channels/my-channel",
});
// Returns { setupAdapter, setupWizard }
```

`plugin-sdk/channel-setup`은 선택적 설치 표면의 절반만 필요할 때를 위한
더 저수준의 `createOptionalChannelSetupAdapter(...)` 및
`createOptionalChannelSetupWizard(...)` builder도 제공합니다.

생성된 선택적 adapter/wizard는 실제 config 쓰기에서는 fail closed로 동작합니다.
이들은 `validateInput`,
`applyAccountConfig`, `finalize` 전반에 걸쳐 하나의 설치 필요 메시지를 재사용하고,
`docsPath`가 설정된 경우 문서 링크를 덧붙입니다.

바이너리 기반 setup UI에는 각 채널에 같은 바이너리/상태 연결 코드를
복사하는 대신 공유 위임 helper를 우선 사용하세요.

- 레이블, 힌트, 점수, 바이너리 탐지만 달라지는 상태 블록에는 `createDetectedBinaryStatus(...)`
- 경로 기반 텍스트 입력에는 `createCliPathTextInput(...)`
- `setupEntry`가 더 무거운 전체 마법사로 지연 위임해야 할 때는
  `createDelegatedSetupWizardStatusResolvers(...)`,
  `createDelegatedPrepare(...)`, `createDelegatedFinalize(...)`,
  `createDelegatedResolveConfigured(...)`
- `setupEntry`가 `textInputs[*].shouldPrompt` 결정만 위임하면 될 때는
  `createDelegatedTextInputShouldPrompt(...)`

## 게시 및 설치

**외부 plugins:** [ClawHub](/ko/tools/clawhub) 또는 npm에 게시한 뒤 설치하세요.

```bash
openclaw plugins install @myorg/openclaw-my-plugin
```

OpenClaw는 먼저 ClawHub를 시도하고 자동으로 npm으로 대체합니다.
ClawHub만 명시적으로 강제할 수도 있습니다.

```bash
openclaw plugins install clawhub:@myorg/openclaw-my-plugin   # ClawHub only
```

이에 대응하는 `npm:` 재정의는 없습니다. ClawHub 대체 이후 npm 경로를 원한다면
일반 npm 패키지 spec을 사용하세요.

```bash
openclaw plugins install @myorg/openclaw-my-plugin
```

**리포지토리 내 plugins:** 번들 plugin 워크스페이스 트리 아래에 두면 빌드 중 자동으로
검색됩니다.

**사용자 설치 명령:**

```bash
openclaw plugins install <package-name>
```

<Info>
  npm 소스 설치의 경우 `openclaw plugins install`은
  `npm install --ignore-scripts`를 실행합니다(수명 주기 스크립트 없음).
  plugin 의존성 트리는 순수 JS/TS로 유지하고 `postinstall` 빌드가 필요한 패키지는 피하세요.
</Info>

## 관련 문서

- [SDK Entry Points](/ko/plugins/sdk-entrypoints) -- `definePluginEntry` 및 `defineChannelPluginEntry`
- [Plugin Manifest](/ko/plugins/manifest) -- 전체 매니페스트 스키마 참조
- [Building Plugins](/ko/plugins/building-plugins) -- 단계별 시작 가이드
