---
read_when:
    - 채팅 명령을 사용하거나 구성할 때
    - 명령 라우팅 또는 권한을 디버깅할 때
summary: 'Slash commands: 텍스트 vs 네이티브, 구성, 지원 명령'
title: Slash Commands
x-i18n:
    generated_at: "2026-04-06T03:14:22Z"
    model: gpt-5.4
    provider: openai
    source_hash: 417e35b9ddd87f25f6c019111b55b741046ea11039dde89210948185ced5696d
    source_path: tools/slash-commands.md
    workflow: 15
---

# Slash Commands

명령은 Gateway에서 처리됩니다. 대부분의 명령은 `/`로 시작하는 **독립 실행형** 메시지로 보내야 합니다.
host 전용 bash 채팅 명령은 `! <cmd>`를 사용합니다(` /bash <cmd>`는 별칭).

서로 관련된 두 가지 시스템이 있습니다.

- **Commands**: 독립 실행형 `/...` 메시지
- **Directives**: `/think`, `/fast`, `/verbose`, `/reasoning`, `/elevated`, `/exec`, `/model`, `/queue`
  - Directives는 모델이 메시지를 보기 전에 제거됩니다.
  - 일반 채팅 메시지(지시어만 있는 메시지가 아님)에서는 “인라인 힌트”로 처리되며 세션 설정을 유지하지 않습니다.
  - 지시어만 있는 메시지(메시지에 지시어만 포함됨)에서는 세션에 유지되며 확인 응답을 보냅니다.
  - Directives는 **권한 있는 발신자**에게만 적용됩니다. `commands.allowFrom`이 설정되어 있으면 그것만
    directives와 commands에 사용되는 허용 목록이 되며, 그렇지 않으면 권한은 채널 허용 목록/페어링 + `commands.useAccessGroups`에서 옵니다.
    권한이 없는 발신자에게는 directives가 일반 텍스트로 처리됩니다.

또한 몇 가지 **인라인 단축키**도 있습니다(allowlist에 있거나 권한 있는 발신자만): `/help`, `/commands`, `/status`, `/whoami` (`/id`).
이들은 즉시 실행되고, 모델이 메시지를 보기 전에 제거되며, 남은 텍스트는 정상 흐름으로 계속 진행됩니다.

## 구성

```json5
{
  commands: {
    native: "auto",
    nativeSkills: "auto",
    text: true,
    bash: false,
    bashForegroundMs: 2000,
    config: false,
    mcp: false,
    plugins: false,
    debug: false,
    restart: false,
    allowFrom: {
      "*": ["user1"],
      discord: ["user:123"],
    },
    useAccessGroups: true,
  },
}
```

- `commands.text`(기본값 `true`)는 채팅 메시지에서 `/...` 파싱을 활성화합니다.
  - 네이티브 명령이 없는 표면(WhatsApp/WebChat/Signal/iMessage/Google Chat/Microsoft Teams)에서는 이를 `false`로 설정해도 텍스트 명령은 계속 동작합니다.
- `commands.native`(기본값 `"auto"`)는 네이티브 명령을 등록합니다.
  - 자동: Discord/Telegram에서는 켜짐, Slack에서는 꺼짐(Slack slash command를 추가하기 전까지), 네이티브 지원이 없는 공급자에서는 무시됨.
  - 공급자별 재정의를 위해 `channels.discord.commands.native`, `channels.telegram.commands.native`, 또는 `channels.slack.commands.native`를 설정하세요(bool 또는 `"auto"`).
  - `false`는 시작 시 Discord/Telegram에서 이전에 등록된 명령을 지웁니다. Slack 명령은 Slack 앱에서 관리되며 자동으로 제거되지 않습니다.
- `commands.nativeSkills`(기본값 `"auto"`)는 지원되는 경우 **skill** 명령을 네이티브로 등록합니다.
  - 자동: Discord/Telegram에서는 켜짐, Slack에서는 꺼짐(Slack은 skill별로 slash command를 하나씩 만들어야 함).
  - 공급자별 재정의를 위해 `channels.discord.commands.nativeSkills`, `channels.telegram.commands.nativeSkills`, 또는 `channels.slack.commands.nativeSkills`를 설정하세요(bool 또는 `"auto"`).
- `commands.bash`(기본값 `false`)는 host 셸 명령을 실행하는 `! <cmd>`를 활성화합니다(` /bash <cmd>`는 별칭이며 `tools.elevated` allowlist가 필요).
- `commands.bashForegroundMs`(기본값 `2000`)는 bash가 백그라운드 모드로 전환되기 전에 얼마나 기다릴지 제어합니다(`0`이면 즉시 백그라운드 전환).
- `commands.config`(기본값 `false`)는 `/config`를 활성화합니다(`openclaw.json` 읽기/쓰기).
- `commands.mcp`(기본값 `false`)는 `/mcp`를 활성화합니다(`mcp.servers` 아래의 OpenClaw 관리 MCP config 읽기/쓰기).
- `commands.plugins`(기본값 `false`)는 `/plugins`를 활성화합니다(plugin 탐색/상태와 설치 + 활성화/비활성화 제어).
- `commands.debug`(기본값 `false`)는 `/debug`를 활성화합니다(런타임 전용 재정의).
- `commands.allowFrom`(선택 사항)은 명령 권한 부여를 위한 공급자별 허용 목록을 설정합니다. 구성되면 이것이
  commands와 directives의 유일한 권한 부여 소스가 되며(채널 허용 목록/페어링과 `commands.useAccessGroups`
  는 무시됨). 전역 기본값에는 `"*"`를 사용하고, 공급자별 키가 있으면 그것이 우선합니다.
- `commands.useAccessGroups`(기본값 `true`)는 `commands.allowFrom`이 설정되지 않았을 때 commands에 대해 허용 목록/정책을 적용합니다.

## 명령 목록

텍스트 + 네이티브(활성화된 경우):

- `/help`
- `/commands`
- `/tools [compact|verbose]` (현재 에이전트가 지금 바로 사용할 수 있는 것을 표시; `verbose`는 설명 추가)
- `/skill <name> [input]` (이름으로 skill 실행)
- `/status` (현재 상태 표시; 사용 가능한 경우 현재 모델 공급자의 공급자 사용량/할당량 포함)
- `/tasks` (현재 세션의 백그라운드 작업 나열; agent-local 폴백 횟수와 함께 활성/최근 작업 세부 정보 표시)
- `/allowlist` (허용 목록 항목 나열/추가/제거)
- `/approve <id> <decision>` (exec 승인 프롬프트 해결; 사용 가능한 결정은 대기 중 승인 메시지를 사용)
- `/context [list|detail|json]` (“context” 설명; `detail`은 파일별 + 도구별 + skill별 + 시스템 프롬프트 크기 표시)
- `/btw <question>` (현재 세션에 대한 일회성 사이드 질문을 미래 세션 컨텍스트를 바꾸지 않고 수행; [/tools/btw](/ko/tools/btw) 참고)
- `/export-session [path]` (별칭: `/export`) (전체 시스템 프롬프트와 함께 현재 세션을 HTML로 내보내기)
- `/whoami` (발신자 id 표시; 별칭: `/id`)
- `/session idle <duration|off>` (포커스된 스레드 바인딩의 비활성 자동 언포커스 관리)
- `/session max-age <duration|off>` (포커스된 스레드 바인딩의 하드 max-age 자동 언포커스 관리)
- `/subagents list|kill|log|info|send|steer|spawn` (현재 세션에 대한 서브에이전트 실행 검사, 제어, 생성)
- `/acp spawn|cancel|steer|close|status|set-mode|set|cwd|permissions|timeout|model|reset-options|doctor|install|sessions` (ACP 런타임 세션 검사 및 제어)
- `/agents` (이 세션에 스레드 바인딩된 에이전트 나열)
- `/focus <target>` (Discord: 이 스레드 또는 새 스레드를 세션/서브에이전트 대상에 바인딩)
- `/unfocus` (Discord: 현재 스레드 바인딩 제거)
- `/kill <id|#|all>` (이 세션의 실행 중인 하나 또는 모든 서브에이전트를 즉시 중단, 확인 메시지 없음)
- `/steer <id|#> <message>` (실행 중인 서브에이전트를 즉시 조향: 가능하면 실행 중에, 아니면 현재 작업을 중단하고 조향 메시지로 재시작)
- `/tell <id|#> <message>` (`/steer`의 별칭)
- `/config show|get|set|unset` (config를 디스크에 지속 저장, 소유자 전용; `commands.config: true` 필요)
- `/mcp show|get|set|unset` (OpenClaw MCP server config 관리, 소유자 전용; `commands.mcp: true` 필요)
- `/plugins list|show|get|install|enable|disable` (탐색된 plugin 검사, 새 plugin 설치, 활성화 전환; 쓰기는 소유자 전용; `commands.plugins: true` 필요)
  - `/plugin`은 `/plugins`의 별칭입니다.
  - `/plugin install <spec>`은 `openclaw plugins install`과 동일한 plugin spec을 받습니다: 로컬 경로/아카이브, npm 패키지, 또는 `clawhub:<pkg>`.
  - 활성화/비활성화 쓰기 후에도 재시작 힌트로 응답합니다. watched foreground gateway에서는 OpenClaw가 쓰기 직후 자동으로 재시작할 수 있습니다.
- `/debug show|set|unset|reset` (런타임 재정의, 소유자 전용; `commands.debug: true` 필요)
- `/usage off|tokens|full|cost` (응답별 사용량 푸터 또는 로컬 비용 요약)
- `/tts off|always|inbound|tagged|status|provider|limit|summary|audio` (TTS 제어; [/tts](/ko/tools/tts) 참고)
  - Discord: 네이티브 명령은 `/voice`입니다(Discord는 `/tts` 예약). 텍스트 `/tts`는 계속 동작합니다.
- `/stop`
- `/restart`
- `/dock-telegram` (별칭: `/dock_telegram`) (응답을 Telegram으로 전환)
- `/dock-discord` (별칭: `/dock_discord`) (응답을 Discord로 전환)
- `/dock-slack` (별칭: `/dock_slack`) (응답을 Slack으로 전환)
- `/activation mention|always` (그룹 전용)
- `/send on|off|inherit` (소유자 전용)
- `/reset` 또는 `/new [model]` (선택적 모델 힌트, 나머지는 그대로 전달됨)
- `/think <off|minimal|low|medium|high|xhigh>` (모델/공급자별 동적 선택지; 별칭: `/thinking`, `/t`)
- `/fast status|on|off` (인수를 생략하면 현재 유효한 fast-mode 상태 표시)
- `/verbose on|full|off` (별칭: `/v`)
- `/reasoning on|off|stream` (별칭: `/reason`; on이면 `Reasoning:`으로 시작하는 별도 메시지를 보냄, `stream` = Telegram draft 전용)
- `/elevated on|off|ask|full` (별칭: `/elev`; `full`은 exec 승인을 건너뜀)
- `/exec host=<auto|sandbox|gateway|node> security=<deny|allowlist|full> ask=<off|on-miss|always> node=<id>` (현재 값을 보려면 `/exec` 전송)
- `/model <name>` (별칭: `/models`; 또는 `agents.defaults.models.*.alias`의 `/<alias>`)
- `/queue <mode>` (`debounce:2s cap:25 drop:summarize` 같은 옵션 포함; 현재 설정을 보려면 `/queue` 전송)
- `/bash <command>` (host 전용; `! <command>`의 별칭; `commands.bash: true` + `tools.elevated` allowlist 필요)
- `/dreaming [on|off|status|help]` (전역 dreaming 전환 또는 상태 표시; [Dreaming](/concepts/dreaming) 참고)

텍스트 전용:

- `/compact [instructions]` ([/concepts/compaction](/ko/concepts/compaction) 참고)
- `! <command>` (host 전용; 한 번에 하나씩; 장기 실행 작업에는 `!poll` + `!stop` 사용)
- `!poll` (출력/상태 확인; 선택적으로 `sessionId` 허용; `/bash poll`도 동작)
- `!stop` (실행 중인 bash 작업 중지; 선택적으로 `sessionId` 허용; `/bash stop`도 동작)

참고:

- 명령은 명령과 인수 사이에 선택적으로 `:`를 받을 수 있습니다(예: `/think: high`, `/send: on`, `/help:`).
- `/new <model>`은 모델 별칭, `provider/model`, 또는 공급자 이름(퍼지 매칭)을 받습니다. 일치하지 않으면 텍스트는 메시지 본문으로 처리됩니다.
- 전체 공급자 사용량 분석은 `openclaw status --usage`를 사용하세요.
- `/allowlist add|remove`는 `commands.config=true`가 필요하며 채널 `configWrites`를 따릅니다.
- 멀티 계정 채널에서는 config 대상 `/allowlist --account <id>`와 `/config set channels.<provider>.accounts.<id>...`도 대상 계정의 `configWrites`를 따릅니다.
- `/usage`는 응답별 사용량 푸터를 제어합니다. `/usage cost`는 OpenClaw 세션 로그에서 로컬 비용 요약을 출력합니다.
- `/restart`는 기본적으로 활성화됩니다. 비활성화하려면 `commands.restart: false`를 설정하세요.
- Discord 전용 네이티브 명령: `/vc join|leave|status`는 음성 채널을 제어합니다(`channels.discord.voice`와 네이티브 명령 필요, 텍스트로는 사용 불가).
- Discord 스레드 바인딩 명령(`/focus`, `/unfocus`, `/agents`, `/session idle`, `/session max-age`)은 유효한 스레드 바인딩이 활성화되어 있어야 합니다(`session.threadBindings.enabled` 및/또는 `channels.discord.threadBindings.enabled`).
- ACP 명령 참조와 런타임 동작: [ACP Agents](/ko/tools/acp-agents).
- `/verbose`는 디버깅과 추가 가시성을 위한 것입니다. 일반 사용에서는 **꺼둔 상태**를 유지하세요.
- `/fast on|off`는 세션 재정의를 유지합니다. 이를 지우고 config 기본값으로 되돌리려면 Sessions UI의 `inherit` 옵션을 사용하세요.
- `/fast`는 공급자별입니다. OpenAI/OpenAI Codex는 이를 네이티브 Responses 엔드포인트에서 `service_tier=priority`로 매핑하는 반면, `api.anthropic.com`으로 보내는 OAuth 인증 트래픽을 포함한 직접 공개 Anthropic 요청은 이를 `service_tier=auto` 또는 `standard_only`로 매핑합니다. [OpenAI](/ko/providers/openai) 및 [Anthropic](/ko/providers/anthropic)을 참고하세요.
- 도구 실패 요약은 관련이 있을 때 계속 표시되지만, 자세한 실패 텍스트는 `/verbose`가 `on` 또는 `full`일 때만 포함됩니다.
- `/reasoning`(및 `/verbose`)은 그룹 설정에서 위험할 수 있습니다. 노출 의도가 없던 내부 reasoning 또는 도구 출력을 드러낼 수 있습니다. 특히 그룹 채팅에서는 꺼 두는 것이 좋습니다.
- `/model`은 새로운 세션 모델을 즉시 유지합니다.
- 에이전트가 유휴 상태이면 다음 실행에서 즉시 사용됩니다.
- 실행이 이미 진행 중이면 OpenClaw는 live 전환을 보류로 표시하고 깔끔한 재시도 지점에서만 새 모델로 재시작합니다.
- 도구 활동이나 응답 출력이 이미 시작되었다면, 보류 중인 전환은 나중 재시도 기회나 다음 사용자 턴까지 대기할 수 있습니다.
- **빠른 경로:** allowlist에 있는 발신자의 명령 전용 메시지는 즉시 처리됩니다(큐 + 모델 우회).
- **그룹 멘션 게이팅:** allowlist에 있는 발신자의 명령 전용 메시지는 멘션 요구 사항을 우회합니다.
- **인라인 단축키(allowlist에 있는 발신자만):** 일부 명령은 일반 메시지 안에 포함되어도 동작하며 모델이 남은 텍스트를 보기 전에 제거됩니다.
  - 예: `hey /status`는 상태 응답을 트리거하고, 남은 텍스트는 정상 흐름으로 계속 진행됩니다.
- 현재: `/help`, `/commands`, `/status`, `/whoami` (`/id`)
- 권한 없는 명령 전용 메시지는 조용히 무시되며, 인라인 `/...` 토큰은 일반 텍스트로 처리됩니다.
- **Skill commands:** `user-invocable` skills는 slash command로 노출됩니다. 이름은 `a-z0-9_`로 정리되며(최대 32자), 충돌 시 숫자 접미사가 붙습니다(예: `_2`).
  - `/skill <name> [input]`은 이름으로 skill을 실행합니다(skill별 명령에 대한 네이티브 명령 제한이 있을 때 유용).
  - 기본적으로 skill 명령은 정상 요청으로 모델에 전달됩니다.
  - skills는 선택적으로 `command-dispatch: tool`을 선언하여 명령을 도구로 직접 라우팅할 수 있습니다(결정적, 모델 없음).
  - 예: `/prose` (OpenProse plugin) — [OpenProse](/ko/prose) 참고
- **네이티브 명령 인수:** Discord는 동적 옵션에 autocomplete를 사용합니다(필수 인수를 생략하면 버튼 메뉴도 사용). Telegram과 Slack은 명령이 선택지를 지원하고 인수를 생략하면 버튼 메뉴를 표시합니다.

## `/tools`

`/tools`는 config 질문이 아니라 런타임 질문에 답합니다. 즉, **이 에이전트가 지금
이 대화에서 실제로 무엇을 사용할 수 있는가**입니다.

- 기본 `/tools`는 compact하며 빠르게 훑어보기에 최적화되어 있습니다.
- `/tools verbose`는 짧은 설명을 추가합니다.
- 인수를 지원하는 네이티브 명령 표면은 `compact|verbose`와 같은 모드 전환을 노출합니다.
- 결과는 세션 범위이므로 에이전트, 채널, 스레드, 발신자 권한, 모델을 변경하면 출력이
  달라질 수 있습니다.
- `/tools`에는 core 도구, 연결된
  plugin 도구, 채널 소유 도구를 포함하여 런타임에서 실제 도달 가능한 도구가 포함됩니다.

프로필 및 재정의 편집은 `/tools`를 정적 카탈로그처럼 다루기보다
Control UI Tools 패널이나 config/catalog 표면을 사용하세요.

## 사용량 표면(어디에 무엇이 표시되는지)

- **공급자 사용량/할당량**(예: “Claude 80% left”)은 사용량 추적이 활성화되어 있을 때 현재 모델 공급자에 대해 `/status`에 표시됩니다. OpenClaw는 공급자 기간을 `% left`로 정규화합니다. MiniMax의 경우 남은 양만 나타내는 퍼센트 필드는 표시 전에 반전되며, `model_remains` 응답은 채팅 모델 항목과 모델 태그가 붙은 플랜 레이블을 우선 사용합니다.
- `/status`의 **토큰/캐시 줄**은 live 세션 스냅샷이 빈약할 때 최신 transcript 사용량 항목으로 폴백할 수 있습니다. 기존의 0이 아닌 live 값은 여전히 우선하며, transcript 폴백은 저장된 총계가 없거나 더 작을 때 활성 런타임 모델 레이블과 더 큰 프롬프트 지향 총량도 복구할 수 있습니다.
- **응답별 토큰/비용**은 `/usage off|tokens|full`로 제어됩니다(일반 응답에 덧붙여짐).
- `/model status`는 사용량이 아니라 **모델/인증/엔드포인트**에 관한 것입니다.

## 모델 선택(`/model`)

`/model`은 directive로 구현됩니다.

예시:

```
/model
/model list
/model 3
/model openai/gpt-5.4
/model opus@anthropic:default
/model status
```

참고:

- `/model`과 `/model list`는 compact하고 번호가 매겨진 선택기(모델 패밀리 + 사용 가능한 공급자)를 표시합니다.
- Discord에서는 `/model`과 `/models`가 공급자 및 모델 드롭다운과 Submit 단계가 있는 대화형 선택기를 엽니다.
- `/model <#>`는 해당 선택기에서 선택합니다(가능하면 현재 공급자를 우선함).
- `/model status`는 자세한 보기를 표시하며, 사용 가능한 경우 구성된 공급자 엔드포인트(`baseUrl`)와 API 모드(`api`)를 포함합니다.

## 디버그 재정의

`/debug`를 사용하면 **런타임 전용** config 재정의(메모리, 디스크 아님)를 설정할 수 있습니다. 소유자 전용입니다. 기본적으로 비활성화되어 있으며 `commands.debug: true`로 활성화하세요.

예시:

```
/debug show
/debug set messages.responsePrefix="[openclaw]"
/debug set channels.whatsapp.allowFrom=["+1555","+4477"]
/debug unset messages.responsePrefix
/debug reset
```

참고:

- 재정의는 새 config 읽기에 즉시 적용되지만 `openclaw.json`에 기록되지는 않습니다.
- 모든 재정의를 지우고 디스크의 config로 돌아가려면 `/debug reset`을 사용하세요.

## Config 업데이트

`/config`는 디스크의 config(`openclaw.json`)에 기록합니다. 소유자 전용입니다. 기본적으로 비활성화되어 있으며 `commands.config: true`로 활성화하세요.

예시:

```
/config show
/config show messages.responsePrefix
/config get messages.responsePrefix
/config set messages.responsePrefix="[openclaw]"
/config unset messages.responsePrefix
```

참고:

- config는 기록 전에 검증되며, 유효하지 않은 변경은 거부됩니다.
- `/config` 업데이트는 재시작 후에도 유지됩니다.

## MCP 업데이트

`/mcp`는 `mcp.servers` 아래의 OpenClaw 관리 MCP server 정의를 기록합니다. 소유자 전용입니다. 기본적으로 비활성화되어 있으며 `commands.mcp: true`로 활성화하세요.

예시:

```text
/mcp show
/mcp show context7
/mcp set context7={"command":"uvx","args":["context7-mcp"]}
/mcp unset context7
```

참고:

- `/mcp`는 Pi 소유 프로젝트 설정이 아니라 OpenClaw config에 저장합니다.
- 실제로 어떤 전송이 실행 가능한지는 런타임 어댑터가 결정합니다.

## Plugin 업데이트

`/plugins`를 통해 운영자는 탐색된 plugin을 검사하고 config에서 활성화를 전환할 수 있습니다. 읽기 전용 흐름에서는 `/plugin`을 별칭으로 사용할 수 있습니다. 기본적으로 비활성화되어 있으며 `commands.plugins: true`로 활성화하세요.

예시:

```text
/plugins
/plugins list
/plugin show context7
/plugins enable context7
/plugins disable context7
```

참고:

- `/plugins list`와 `/plugins show`는 현재 작업공간과 디스크상의 config를 기준으로 실제 plugin 탐색을 사용합니다.
- `/plugins enable|disable`는 plugin config만 업데이트하며 plugin을 설치하거나 제거하지는 않습니다.
- 활성화/비활성화 변경 후 적용하려면 gateway를 재시작하세요.

## 표면 참고

- **텍스트 명령**은 정상 채팅 세션에서 실행됩니다(DM은 `main`을 공유하고, 그룹은 자체 세션을 가짐).
- **네이티브 명령**은 격리된 세션을 사용합니다.
  - Discord: `agent:<agentId>:discord:slash:<userId>`
  - Slack: `agent:<agentId>:slack:slash:<userId>` (접두사는 `channels.slack.slashCommand.sessionPrefix`로 구성 가능)
  - Telegram: `telegram:slash:<userId>` (`CommandTargetSessionKey`를 통해 채팅 세션을 대상으로 지정)
- **`/stop`**은 활성 채팅 세션을 대상으로 하므로 현재 실행을 중단할 수 있습니다.
- **Slack:** `channels.slack.slashCommand`는 단일 `/openclaw` 스타일 명령용으로 여전히 지원됩니다. `commands.native`를 활성화하면 내장 명령마다 Slack slash command를 하나씩 만들어야 합니다(`/help`와 같은 이름). Slack용 명령 인수 메뉴는 ephemeral Block Kit 버튼으로 전달됩니다.
  - Slack 네이티브 예외: Slack이 `/status`를 예약했기 때문에 `/status`가 아니라 `/agentstatus`를 등록하세요. 텍스트 `/status`는 Slack 메시지에서 계속 동작합니다.

## BTW 사이드 질문

`/btw`는 현재 세션에 대한 빠른 **사이드 질문**입니다.

일반 채팅과 달리 다음과 같습니다.

- 현재 세션을 배경 컨텍스트로 사용하고,
- 별도의 **도구 없는** 원샷 호출로 실행되며,
- 미래 세션 컨텍스트를 변경하지 않고,
- transcript 기록에 쓰이지 않으며,
- 일반 assistant 메시지 대신 live 사이드 결과로 전달됩니다.

따라서 `/btw`는 주요
작업은 계속 진행하면서 일시적인 설명이 필요할 때 유용합니다.

예시:

```text
/btw what are we doing right now?
```

전체 동작과 클라이언트 UX 세부 사항은 [BTW Side Questions](/ko/tools/btw)를 참고하세요.
