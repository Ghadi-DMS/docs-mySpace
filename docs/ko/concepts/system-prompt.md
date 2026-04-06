---
read_when:
    - 시스템 프롬프트 텍스트, 도구 목록 또는 시간/heartbeat 섹션을 편집할 때
    - 워크스페이스 부트스트랩 또는 Skills 주입 동작을 변경할 때
summary: OpenClaw 시스템 프롬프트에 무엇이 포함되며 어떻게 조립되는지
title: 시스템 프롬프트
x-i18n:
    generated_at: "2026-04-06T03:07:25Z"
    model: gpt-5.4
    provider: openai
    source_hash: f14ba7f16dda81ac973d72be05931fa246bdfa0e1068df1a84d040ebd551c236
    source_path: concepts/system-prompt.md
    workflow: 15
---

# 시스템 프롬프트

OpenClaw는 모든 에이전트 실행마다 사용자 지정 시스템 프롬프트를 구성합니다. 이 프롬프트는 **OpenClaw 소유**이며 pi-coding-agent 기본 프롬프트를 사용하지 않습니다.

프롬프트는 OpenClaw가 조립하여 각 에이전트 실행에 주입합니다.

프로바이더 플러그인은 전체 OpenClaw 소유 프롬프트를 대체하지 않고도 캐시 인지형 프롬프트 안내를 추가할 수 있습니다. 프로바이더 런타임은 다음을 수행할 수 있습니다.

- 이름이 지정된 소수의 핵심 섹션(`interaction_style`,
  `tool_call_style`, `execution_bias`) 교체
- 프롬프트 캐시 경계 위에 **안정적인 접두사** 주입
- 프롬프트 캐시 경계 아래에 **동적 접미사** 주입

모델 계열별 튜닝에는 프로바이더 소유 기여를 사용하세요. 레거시
`before_prompt_build` 프롬프트 변경은 호환성 목적이나 진정으로 전역적인 프롬프트
변경에만 사용하고, 일반적인 프로바이더 동작에는 사용하지 마세요.

## 구조

프롬프트는 의도적으로 간결하며 고정된 섹션을 사용합니다.

- **Tooling**: 구조화된 도구의 단일 진실 공급원 알림과 런타임 도구 사용 안내.
- **Safety**: 권력 추구 행동이나 감독 우회를 피하기 위한 짧은 가드레일 알림.
- **Skills**(사용 가능한 경우): 필요할 때 skill 지침을 로드하는 방법을 모델에 알려줍니다.
- **OpenClaw Self-Update**: `config.schema.lookup`로 안전하게 구성을 검사하고,
  `config.patch`로 구성을 패치하고, `config.apply`로 전체 구성을
  교체하며, 명시적인 사용자 요청이 있을 때만 `update.run`을 실행하는 방법입니다.
  소유자 전용 `gateway` 도구는 `tools.exec.ask` / `tools.exec.security`의
  재작성도 거부하며, 여기에는 해당 보호된 exec 경로로 정규화되는 레거시 `tools.bash.*`
  별칭도 포함됩니다.
- **Workspace**: 작업 디렉터리(`agents.defaults.workspace`).
- **Documentation**: OpenClaw 문서의 로컬 경로(리포지토리 또는 npm 패키지)와 읽어야 하는 시점.
- **Workspace Files (injected)**: 부트스트랩 파일이 아래에 포함되어 있음을 나타냅니다.
- **Sandbox**(활성화된 경우): 샌드박스 런타임, 샌드박스 경로, 권한 상승 exec 사용 가능 여부를 나타냅니다.
- **Current Date & Time**: 사용자 로컬 시간, 시간대 및 시간 형식.
- **Reply Tags**: 지원되는 프로바이더용 선택적 답장 태그 구문.
- **Heartbeats**: heartbeat 프롬프트 및 ack 동작.
- **Runtime**: 호스트, OS, node, 리포지토리 루트(감지된 경우), 사고 수준(한 줄).
- **Reasoning**: 현재 가시성 수준 + /reasoning 전환 힌트.

Tooling 섹션에는 장시간 실행 작업에 대한 런타임 안내도 포함됩니다.

- 미래의 후속 작업(`나중에 다시 확인`, 리마인더, 반복 작업)에는 `exec` sleep 루프, `yieldMs` 지연 기법, 반복적인 `process`
  폴링 대신 cron을 사용
- 지금 시작해서 백그라운드에서 계속 실행되는 명령에만 `exec` / `process` 사용
- 자동 완료 깨우기가 활성화되어 있으면, 명령은 한 번만 시작하고
  출력이 나오거나 실패할 때 푸시 기반 깨우기 경로에 의존
- 실행 중인 명령을 확인해야 할 때는 로그, 상태, 입력 또는 개입을 위해 `process` 사용
- 작업 규모가 더 크다면 `sessions_spawn`을 우선 사용. 하위 에이전트 완료는
  푸시 기반이며 요청자에게 자동으로 공지됨
- 완료만 기다리기 위해 `subagents list` / `sessions_list`를 반복해서 폴링하지 않기

실험적 `update_plan` 도구가 활성화되어 있으면 Tooling은 또한
비사소한 다단계 작업에만 이를 사용하고, 정확히 하나의 `in_progress`
단계를 유지하며, 매번 업데이트 후 전체 계획을 반복하지 않도록 모델에 지시합니다.

시스템 프롬프트의 Safety 가드레일은 권고적입니다. 이는 모델 동작을 안내하지만 정책을 강제하지는 않습니다. 강제 적용에는 도구 정책, exec 승인, 샌드박싱, 채널 허용 목록을 사용하세요. 운영자는 설계상 이를 비활성화할 수 있습니다.

기본 승인 카드/버튼을 지원하는 채널에서는 이제 런타임 프롬프트가
에이전트에게 먼저 그 기본 승인 UI에 의존하라고 알려줍니다. 수동
`/approve` 명령은 도구 결과가 채팅 승인을 사용할 수 없다고 알려주거나
수동 승인이 유일한 경로일 때만 포함해야 합니다.

## 프롬프트 모드

OpenClaw는 하위 에이전트를 위해 더 작은 시스템 프롬프트를 렌더링할 수 있습니다. 런타임은 각 실행에 대해
`promptMode`를 설정합니다(사용자 대상 구성은 아님).

- `full`(기본값): 위의 모든 섹션을 포함합니다.
- `minimal`: 하위 에이전트에 사용되며 **Skills**, **Memory Recall**, **OpenClaw
  Self-Update**, **Model Aliases**, **User Identity**, **Reply Tags**,
  **Messaging**, **Silent Replies**, **Heartbeats**를 생략합니다. Tooling, **Safety**,
  Workspace, Sandbox, Current Date & Time(알려진 경우), Runtime 및 주입된
  컨텍스트는 계속 사용할 수 있습니다.
- `none`: 기본 식별 한 줄만 반환합니다.

`promptMode=minimal`일 때 추가 주입 프롬프트는 **Group Chat Context** 대신
**Subagent Context**로 표시됩니다.

## Workspace 부트스트랩 주입

부트스트랩 파일은 잘려서 **Project Context** 아래에 추가되므로, 모델이 명시적인 읽기 없이도 정체성과 프로필 컨텍스트를 볼 수 있습니다.

- `AGENTS.md`
- `SOUL.md`
- `TOOLS.md`
- `IDENTITY.md`
- `USER.md`
- `HEARTBEAT.md`
- `BOOTSTRAP.md`(완전히 새로운 워크스페이스에서만)
- `MEMORY.md`가 있으면 그것을, 없으면 소문자 대체 파일인 `memory.md`

이 모든 파일은 매 턴마다 **컨텍스트 창에 주입**되므로
토큰을 소비합니다. 간결하게 유지하세요. 특히 `MEMORY.md`는 시간이 지나며
커질 수 있고, 예상보다 높은 컨텍스트 사용량과 더 잦은
압축으로 이어질 수 있습니다.

> **참고:** `memory/*.md` 일일 파일은 자동으로 주입되지 않습니다.
> 대신 `memory_search` 및 `memory_get` 도구를 통해 필요 시 접근하므로
> 모델이 명시적으로 읽지 않는 한 컨텍스트 창을 차지하지 않습니다.

큰 파일은 마커와 함께 잘립니다. 파일별 최대 크기는
`agents.defaults.bootstrapMaxChars`(기본값: 20000)로 제어됩니다. 파일 전체에 걸친 총 주입 부트스트랩
콘텐츠는 `agents.defaults.bootstrapTotalMaxChars`
(기본값: 150000)로 제한됩니다. 누락된 파일은 짧은 누락 파일 마커를 주입합니다. 잘림이
발생하면 OpenClaw는 Project Context에 경고 블록을 주입할 수 있습니다. 이는
`agents.defaults.bootstrapPromptTruncationWarning` (`off`, `once`, `always`;
기본값: `once`)로 제어합니다.

하위 에이전트 세션은 `AGENTS.md`와 `TOOLS.md`만 주입합니다(다른 부트스트랩 파일은
하위 에이전트 컨텍스트를 작게 유지하기 위해 필터링됩니다).

내부 훅은 `agent:bootstrap`을 통해 이 단계를 가로채 주입된
부트스트랩 파일을 변경하거나 교체할 수 있습니다(예: `SOUL.md`를 다른 persona로 교체).

에이전트가 덜 일반적으로 들리게 하고 싶다면
[SOUL.md Personality Guide](/ko/concepts/soul)부터 시작하세요.

각 주입 파일이 얼마나 기여하는지(원본 대비 주입본, 잘림 여부, 그리고 도구 스키마 오버헤드)를 확인하려면 `/context list` 또는 `/context detail`을 사용하세요. [Context](/ko/concepts/context)를 참고하세요.

## 시간 처리

사용자 시간대가 알려진 경우 시스템 프롬프트에는 전용 **Current Date & Time** 섹션이 포함됩니다. 프롬프트 캐시 안정성을 유지하기 위해 이제 여기에는
**시간대**만 포함되고(동적인 시계나 시간 형식은 포함되지 않음) 있습니다.

에이전트에 현재 시간이 필요하면 `session_status`를 사용하세요. 상태 카드에는 타임스탬프 줄이 포함됩니다. 같은 도구로 세션별 모델
재정의도 설정할 수 있습니다(`model=default`로 해제).

구성 항목:

- `agents.defaults.userTimezone`
- `agents.defaults.timeFormat` (`auto` | `12` | `24`)

전체 동작 세부 정보는 [Date & Time](/ko/date-time)을 참고하세요.

## Skills

적격한 skill이 있으면 OpenClaw는 각 skill의 **파일 경로**를 포함하는 간결한 **사용 가능한 skills 목록**
(`formatSkillsForPrompt`)을 주입합니다. 프롬프트는 모델에게 나열된
위치(워크스페이스, 관리형 또는 번들)에 있는 SKILL.md를 로드하기 위해 `read`를 사용하라고 지시합니다. 적격한 skill이 없으면
Skills 섹션은 생략됩니다.

적격성에는 skill 메타데이터 게이트, 런타임 환경/구성 검사,
그리고 `agents.defaults.skills` 또는
`agents.list[].skills`가 구성된 경우 유효한 에이전트 skill 허용 목록이 포함됩니다.

```
<available_skills>
  <skill>
    <name>...</name>
    <description>...</description>
    <location>...</location>
  </skill>
</available_skills>
```

이렇게 하면 기본 프롬프트를 작게 유지하면서도 필요한 skill 사용은 가능하게 합니다.

## Documentation

사용 가능한 경우, 시스템 프롬프트에는 로컬
OpenClaw 문서 디렉터리(리포지토리 워크스페이스의 `docs/` 또는 번들된 npm
패키지 문서)를 가리키는 **Documentation** 섹션이 포함되며, 공개 미러, 소스 리포지토리, 커뮤니티 Discord,
그리고 Skills 발견을 위한 ClawHub([https://clawhub.ai](https://clawhub.ai))도 함께 언급합니다. 프롬프트는 모델에게 OpenClaw 동작, 명령, 구성 또는 아키텍처에 대해서는 먼저 로컬 문서를 참고하고,
가능하면 직접 `openclaw status`를 실행하며(접근 권한이 없을 때만 사용자에게 묻도록) 지시합니다.
