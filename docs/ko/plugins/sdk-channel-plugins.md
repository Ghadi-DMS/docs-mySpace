---
read_when:
    - 새 메시징 채널 plugin을 만들고 있을 때
    - OpenClaw를 메시징 플랫폼에 연결하려고 할 때
    - ChannelPlugin 어댑터 표면을 이해해야 할 때
sidebarTitle: Channel Plugins
summary: OpenClaw용 메시징 채널 plugin을 구축하는 단계별 가이드
title: 채널 Plugins 구축하기
x-i18n:
    generated_at: "2026-04-06T03:09:51Z"
    model: gpt-5.4
    provider: openai
    source_hash: 66b52c10945a8243d803af3bf7e1ea0051869ee92eda2af5718d9bb24fbb8552
    source_path: plugins/sdk-channel-plugins.md
    workflow: 15
---

# 채널 Plugins 구축하기

이 가이드는 OpenClaw를 메시징 플랫폼에 연결하는 채널 plugin 구축 과정을 안내합니다. 이 가이드를 마치면 DM 보안, pairing, 답장 스레딩, 아웃바운드 메시징을 갖춘 작동하는 채널을 만들 수 있습니다.

<Info>
  아직 OpenClaw plugin을 한 번도 만들어본 적이 없다면 먼저
  기본 패키지 구조와 매니페스트 설정을 위해
  [Getting Started](/ko/plugins/building-plugins)를 읽어보세요.
</Info>

## 채널 plugins가 동작하는 방식

채널 plugins에는 자체 send/edit/react 도구가 필요하지 않습니다. OpenClaw는 코어에 하나의
공유 `message` 도구를 유지합니다. 여러분의 plugin이 담당하는 부분은 다음과 같습니다.

- **Config** — 계정 확인 및 설정 마법사
- **Security** — DM 정책 및 허용 목록
- **Pairing** — DM 승인 흐름
- **Session grammar** — 프로바이더별 대화 ID가 기본 채팅, 스레드 ID, 부모 대체값에 어떻게 매핑되는지
- **Outbound** — 플랫폼으로 텍스트, 미디어, poll 보내기
- **Threading** — 답장을 어떻게 스레드로 연결할지

코어는 공유 메시지 도구, 프롬프트 연결, 바깥쪽 session-key 형태,
일반적인 `:thread:` bookkeeping, 그리고 디스패치를 담당합니다.

플랫폼이 대화 ID 안에 추가 범위를 저장한다면, 그 파싱은
plugin 안의 `messaging.resolveSessionConversation(...)`에 유지하세요. 이것이
`rawId`를 기본 대화 ID, 선택적 스레드 ID,
명시적 `baseConversationId`, 그리고 모든 `parentConversationCandidates`에 매핑하는
정식 훅입니다.
`parentConversationCandidates`를 반환할 때는 가장 좁은 부모부터 가장 넓은/기본 대화 순으로
정렬해 유지하세요.

채널 레지스트리가 부팅되기 전에 동일한 파싱이 필요한 번들 plugin은
일치하는 `resolveSessionConversation(...)` 내보내기를 가진 최상위
`session-key-api.ts` 파일을 노출할 수도 있습니다. 코어는 런타임 plugin 레지스트리를
아직 사용할 수 없을 때만 이 부트스트랩 안전 표면을 사용합니다.

`messaging.resolveParentConversationCandidates(...)`는
plugin이 일반/raw ID 위에 부모 대체값만 필요로 하는 경우를 위한
레거시 호환성 대체 경로로 계속 사용할 수 있습니다. 두 훅이 모두 있으면 코어는
먼저 `resolveSessionConversation(...).parentConversationCandidates`를 사용하고,
정식 훅이 이를 생략한 경우에만 `resolveParentConversationCandidates(...)`로
대체합니다.

## 승인 및 채널 기능

대부분의 채널 plugins에는 승인 전용 코드가 필요하지 않습니다.

- 코어는 같은 채팅의 `/approve`, 공유 승인 버튼 payload, 일반적인 대체 전달을 담당합니다.
- 채널에 승인 전용 동작이 필요할 때는 채널 plugin에 하나의 `approvalCapability` 객체를 두는 방식을 우선하세요.
- `approvalCapability.authorizeActorAction`과 `approvalCapability.getActionAvailabilityState`가 정식 승인 인증 연결점입니다.
- 채널이 네이티브 exec 승인을 노출한다면, 네이티브 전송이 전적으로 `approvalCapability.native` 아래에 있더라도 `approvalCapability.getActionAvailabilityState`를 구현하세요. 코어는 이 가용성 훅을 사용해 `enabled`와 `disabled`를 구분하고, 시작 채널이 네이티브 승인을 지원하는지 판단하며, 채널을 네이티브 클라이언트 대체 안내에 포함합니다.
- 중복 로컬 승인 프롬프트 숨김이나 전달 전 typing indicator 전송 같은 채널별 payload 수명 주기 동작에는 `outbound.shouldSuppressLocalPayloadPrompt` 또는 `outbound.beforeDeliverPayload`를 사용하세요.
- `approvalCapability.delivery`는 네이티브 승인 라우팅이나 대체 전달 억제에만 사용하세요.
- 채널이 공유 렌더러 대신 진정으로 사용자 지정 승인 payload가 필요할 때만 `approvalCapability.render`를 사용하세요.
- 채널이 비활성화 경로 응답에서 네이티브 exec 승인을 활성화하는 데 필요한 정확한 config knob를 설명하고 싶다면 `approvalCapability.describeExecApprovalSetup`를 사용하세요. 이 훅은 `{ channel, channelLabel, accountId }`를 받습니다. 이름 있는 계정 채널은 최상위 기본값 대신 `channels.<channel>.accounts.<id>.execApprovals.*` 같은 계정 범위 경로를 렌더링해야 합니다.
- 채널이 기존 config에서 안정적인 소유자 유사 DM 정체성을 추론할 수 있다면, 승인 전용 코어 로직을 추가하지 않고 같은 채팅 `/approve`를 제한하기 위해 `openclaw/plugin-sdk/approval-runtime`의 `createResolvedApproverActionAuthAdapter`를 사용하세요.
- 채널에 네이티브 승인 전달이 필요하다면, 채널 코드는 대상 정규화와 전송 훅에 집중되도록 유지하세요. `openclaw/plugin-sdk/approval-runtime`의 `createChannelExecApprovalProfile`, `createChannelNativeOriginTargetResolver`, `createChannelApproverDmTargetResolver`, `createApproverRestrictedNativeApprovalCapability`, `createChannelNativeApprovalRuntime`을 사용해 요청 필터링, 라우팅, 중복 제거, 만료, gateway 구독은 코어가 맡도록 하세요.
- 네이티브 승인 채널은 `accountId`와 `approvalKind`를 모두 이 도우미를 통해 라우팅해야 합니다. `accountId`는 다중 계정 승인 정책을 올바른 봇 계정 범위로 유지하고, `approvalKind`는 코어의 하드코딩 분기 없이 exec 대 plugin 승인 동작을 채널에서 사용할 수 있게 합니다.
- 전달된 승인 ID 종류는 처음부터 끝까지 그대로 보존하세요. 네이티브 클라이언트는 채널 로컬 상태를 바탕으로 exec 대 plugin 승인 라우팅을 추측하거나 다시 쓰면 안 됩니다.
- 서로 다른 승인 종류는 의도적으로 서로 다른 네이티브 표면을 노출할 수 있습니다.
  현재 번들 예시는 다음과 같습니다.
  - Slack은 exec와 plugin ID 모두에 대해 네이티브 승인 라우팅을 유지합니다.
  - Matrix는 exec 승인에만 네이티브 DM/채널 라우팅을 유지하고,
    plugin 승인은 공유 같은 채팅 `/approve` 경로에 둡니다.
- `createApproverRestrictedNativeApprovalAdapter`도 여전히 호환성 래퍼로 존재하지만, 새 코드는 capability builder를 우선 사용하고 plugin에 `approvalCapability`를 노출해야 합니다.

핫 채널 진입점에서는 해당 계열의 한 부분만 필요하다면
더 좁은 런타임 하위 경로를 우선 사용하세요.

- `openclaw/plugin-sdk/approval-auth-runtime`
- `openclaw/plugin-sdk/approval-client-runtime`
- `openclaw/plugin-sdk/approval-delivery-runtime`
- `openclaw/plugin-sdk/approval-native-runtime`
- `openclaw/plugin-sdk/approval-reply-runtime`

마찬가지로 더 넓은 umbrella
표면이 필요하지 않다면 `openclaw/plugin-sdk/setup-runtime`,
`openclaw/plugin-sdk/setup-adapter-runtime`,
`openclaw/plugin-sdk/reply-runtime`,
`openclaw/plugin-sdk/reply-dispatch-runtime`,
`openclaw/plugin-sdk/reply-reference`,
`openclaw/plugin-sdk/reply-chunking`를 우선하세요.

설정의 경우 구체적으로 다음을 따르세요.

- `openclaw/plugin-sdk/setup-runtime`은 런타임 안전 설정 도우미를 다룹니다.
  여기에는 import-safe 설정 패치 어댑터(`createPatchedAccountSetupAdapter`,
  `createEnvPatchedAccountSetupAdapter`,
  `createSetupInputPresenceValidator`), lookup-note 출력,
  `promptResolvedAllowFrom`, `splitSetupEntries`, 위임된
  setup-proxy builder가 포함됩니다.
- `openclaw/plugin-sdk/setup-adapter-runtime`은 `createEnvPatchedAccountSetupAdapter`를 위한
  좁은 env 인지형 어댑터 연결점입니다.
- `openclaw/plugin-sdk/channel-setup`은 선택적 설치 setup
  builder와 몇 가지 setup-safe 기본 요소를 다룹니다.
  `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`,
  `createOptionalChannelSetupWizard`, `DEFAULT_ACCOUNT_ID`,
  `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, 그리고
  `splitSetupEntries`가 포함됩니다.
- 더 무거운 공유 설정/config 도우미인
  `moveSingleAccountChannelSectionToDefaultAccount(...)`도 필요한 경우에만
  더 넓은 `openclaw/plugin-sdk/setup` 연결점을 사용하세요.

채널이 setup 표면에서 "먼저 이 plugin을 설치하세요"만 알리고 싶다면,
`createOptionalChannelSetupSurface(...)`를 우선 사용하세요. 생성되는
adapter/wizard는 config 쓰기와 최종화에서 fail closed 방식으로 동작하며,
검증, finalize, docs-link 문구 전반에 걸쳐 동일한 설치 필요 메시지를 재사용합니다.

다른 핫 채널 경로에서도 넓은 레거시 표면보다 좁은 도우미를 우선하세요.

- 다중 계정 config 및 기본 계정 대체를 위한
  `openclaw/plugin-sdk/account-core`,
  `openclaw/plugin-sdk/account-id`,
  `openclaw/plugin-sdk/account-resolution`,
  `openclaw/plugin-sdk/account-helpers`
- 인바운드 경로/envelope 및 기록-디스패치 연결을 위한
  `openclaw/plugin-sdk/inbound-envelope`,
  `openclaw/plugin-sdk/inbound-reply-dispatch`
- 대상 파싱/매칭을 위한 `openclaw/plugin-sdk/messaging-targets`
- 미디어 로드 및 아웃바운드 정체성/전송 위임을 위한
  `openclaw/plugin-sdk/outbound-media`,
  `openclaw/plugin-sdk/outbound-runtime`
- 스레드 바인딩 수명 주기 및 어댑터 등록을 위한
  `openclaw/plugin-sdk/thread-bindings-runtime`
- 레거시 agent/media payload 필드 레이아웃이 여전히 필요한 경우에만
  `openclaw/plugin-sdk/agent-media-payload`
- Telegram 사용자 지정 명령 정규화, 중복/충돌 검증,
  그리고 fallback-stable 명령 config 계약을 위한
  `openclaw/plugin-sdk/telegram-command-config`

인증 전용 채널은 보통 기본 경로에서 멈춰도 됩니다. 코어가 승인을 처리하고 plugin은 아웃바운드/인증 기능만 노출하면 됩니다. Matrix, Slack, Telegram, 사용자 지정 채팅 전송 같은 네이티브 승인 채널은 자체 승인 수명 주기를 만들지 말고 공유 네이티브 도우미를 사용해야 합니다.

## 단계별 안내

<Steps>
  <a id="step-1-package-and-manifest"></a>
  <Step title="패키지 및 매니페스트">
    표준 plugin 파일을 만드세요. `package.json`의 `channel` 필드는
    이것이 채널 plugin임을 나타냅니다. 전체 패키지 메타데이터 표면은
    [Plugin Setup and Config](/ko/plugins/sdk-setup#openclawchannel)을 참고하세요.

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

  <Step title="채널 plugin 객체 만들기">
    `ChannelPlugin` 인터페이스에는 많은 선택적 어댑터 표면이 있습니다. 우선
    최소 구성인 `id`와 `setup`부터 시작하고, 필요에 따라 어댑터를 추가하세요.

    `src/channel.ts`를 만드세요.

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

    <Accordion title="createChatChannelPlugin이 대신 처리해 주는 것">
      저수준 어댑터 인터페이스를 직접 구현하는 대신,
      선언적 옵션을 전달하면 builder가 이를 조합합니다.

      | 옵션 | 연결되는 항목 |
      | --- | --- |
      | `security.dm` | config 필드로부터 범위가 지정된 DM 보안 확인자 |
      | `pairing.text` | 코드 교환을 사용하는 텍스트 기반 DM pairing 흐름 |
      | `threading` | reply-to 모드 확인자(고정, 계정 범위 지정, 또는 사용자 지정) |
      | `outbound.attachedResults` | 결과 메타데이터(message ID)를 반환하는 전송 함수 |

      완전한 제어가 필요하다면 선언적 옵션 대신 원시 어댑터 객체를 전달할 수도 있습니다.
    </Accordion>

  </Step>

  <Step title="진입점 연결">
    `index.ts`를 만드세요.

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

    채널 소유 CLI descriptor는 `registerCliMetadata(...)`에 두세요. 그래야 OpenClaw가
    전체 채널 런타임을 활성화하지 않고도 루트 도움말에 이를 표시할 수 있으며,
    일반적인 전체 로드도 실제 명령 등록을 위해 같은 descriptor를 가져올 수 있습니다.
    `registerFull(...)`은 런타임 전용 작업에 유지하세요.
    `registerFull(...)`이 gateway RPC 메서드를 등록한다면
    plugin 전용 접두사를 사용하세요. 코어 관리 네임스페이스(`config.*`,
    `exec.approvals.*`, `wizard.*`, `update.*`)는 예약되어 있으며 항상
    `operator.admin`으로 확인됩니다.
    `defineChannelPluginEntry`는 등록 모드 분리를 자동으로 처리합니다. 모든
    옵션은 [Entry Points](/ko/plugins/sdk-entrypoints#definechannelpluginentry)를 참고하세요.

  </Step>

  <Step title="setup entry 추가">
    온보딩 중 경량 로드를 위해 `setup-entry.ts`를 만드세요.

    ```typescript setup-entry.ts
    import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineSetupPluginEntry(acmeChatPlugin);
    ```

    OpenClaw는 채널이 비활성화되었거나 아직 구성되지 않았을 때 전체 entry 대신 이를 로드합니다.
    이렇게 하면 설정 흐름 중에 무거운 런타임 코드를 끌어오지 않아도 됩니다.
    자세한 내용은 [Setup and Config](/ko/plugins/sdk-setup#setup-entry)를 참고하세요.

  </Step>

  <Step title="인바운드 메시지 처리">
    plugin은 플랫폼으로부터 메시지를 받아 OpenClaw로 전달해야 합니다.
    일반적인 패턴은 요청을 검증하고
    채널의 인바운드 핸들러를 통해 디스패치하는 webhook입니다.

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
      인바운드 메시지 처리는 채널별로 다릅니다. 각 채널 plugin은
      자체 인바운드 파이프라인을 소유합니다. 실제 패턴은 번들 채널 plugins
      (예: Microsoft Teams 또는 Google Chat plugin 패키지)을 참고하세요.
    </Note>

  </Step>

<a id="step-6-test"></a>
<Step title="테스트">
`src/channel.test.ts`에 코로케이션 테스트를 작성하세요.

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

    공유 테스트 도우미는 [Testing](/ko/plugins/sdk-testing)을 참고하세요.

  </Step>
</Steps>

## 파일 구조

```
<bundled-plugin-root>/acme-chat/
├── package.json              # openclaw.channel metadata
├── openclaw.plugin.json      # config schema가 포함된 매니페스트
├── index.ts                  # defineChannelPluginEntry
├── setup-entry.ts            # defineSetupPluginEntry
├── api.ts                    # 공개 export(선택 사항)
├── runtime-api.ts            # 내부 런타임 export(선택 사항)
└── src/
    ├── channel.ts            # createChatChannelPlugin을 통한 ChannelPlugin
    ├── channel.test.ts       # 테스트
    ├── client.ts             # 플랫폼 API client
    └── runtime.ts            # 런타임 저장소(필요한 경우)
```

## 고급 주제

<CardGroup cols={2}>
  <Card title="스레딩 옵션" icon="git-branch" href="/ko/plugins/sdk-entrypoints#registration-mode">
    고정, 계정 범위 지정, 또는 사용자 지정 답장 모드
  </Card>
  <Card title="메시지 도구 통합" icon="puzzle" href="/ko/plugins/architecture#channel-plugins-and-the-shared-message-tool">
    describeMessageTool 및 action discovery
  </Card>
  <Card title="대상 확인" icon="crosshair" href="/ko/plugins/architecture#channel-target-resolution">
    inferTargetChatType, looksLikeId, resolveTarget
  </Card>
  <Card title="런타임 도우미" icon="settings" href="/ko/plugins/sdk-runtime">
    TTS, STT, media, subagent via api.runtime
  </Card>
</CardGroup>

<Note>
일부 번들 도우미 연결점은 번들 plugin 유지 관리와
호환성을 위해 여전히 존재합니다. 새 채널 plugins에는 권장되는 패턴이 아니며,
그 번들 plugin 계열을 직접 유지 관리하는 경우가 아니라면
번들된 공통 SDK 표면의 일반적인 channel/setup/reply/runtime 하위 경로를 우선 사용하세요.
</Note>

## 다음 단계

- [Provider Plugins](/ko/plugins/sdk-provider-plugins) — plugin이 모델도 제공하는 경우
- [SDK Overview](/ko/plugins/sdk-overview) — 전체 하위 경로 import 참조
- [SDK Testing](/ko/plugins/sdk-testing) — 테스트 유틸리티 및 계약 테스트
- [Plugin Manifest](/ko/plugins/manifest) — 전체 매니페스트 스키마
