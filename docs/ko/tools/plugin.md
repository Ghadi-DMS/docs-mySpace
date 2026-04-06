---
read_when:
    - 플러그인 설치 또는 구성 중
    - 플러그인 discovery 및 로드 규칙 이해 중
    - Codex/Claude 호환 플러그인 bundle 작업 중
sidebarTitle: Install and Configure
summary: OpenClaw 플러그인 설치, 구성 및 관리
title: 플러그인
x-i18n:
    generated_at: "2026-04-06T03:13:51Z"
    model: gpt-5.4
    provider: openai
    source_hash: 9e2472a3023f3c1c6ee05b0cdc228f6b713cc226a08695b327de8a3ad6973c83
    source_path: tools/plugin.md
    workflow: 15
---

# 플러그인

플러그인은 OpenClaw를 새로운 capability로 확장합니다: channels, model providers,
tools, skills, speech, realtime transcription, realtime voice,
media-understanding, image generation, video generation, web fetch, web
search 등입니다. 일부 플러그인은 **core**(OpenClaw와 함께 제공)이고, 다른 일부는
**external**(커뮤니티가 npm에 게시)입니다.

## 빠른 시작

<Steps>
  <Step title="로드된 항목 보기">
    ```bash
    openclaw plugins list
    ```
  </Step>

  <Step title="플러그인 설치">
    ```bash
    # npm에서
    openclaw plugins install @openclaw/voice-call

    # 로컬 디렉터리 또는 아카이브에서
    openclaw plugins install ./my-plugin
    openclaw plugins install ./my-plugin.tgz
    ```

  </Step>

  <Step title="Gateway 다시 시작">
    ```bash
    openclaw gateway restart
    ```

    그런 다음 config 파일의 `plugins.entries.\<id\>.config` 아래에서 구성하세요.

  </Step>
</Steps>

채팅 기본 제어를 선호한다면 `commands.plugins: true`를 활성화하고 다음을 사용하세요.

```text
/plugin install clawhub:@openclaw/voice-call
/plugin show voice-call
/plugin enable voice-call
```

설치 경로는 CLI와 동일한 resolver를 사용합니다: 로컬 경로/아카이브, 명시적
`clawhub:<pkg>`, 또는 일반 패키지 명세(먼저 ClawHub, 그다음 npm fallback).

config가 잘못되면 설치는 일반적으로 fail closed로 실패하고
`openclaw doctor --fix`를 안내합니다. 유일한 복구 예외는
`openclaw.install.allowInvalidConfigRecovery`에 옵트인한 플러그인을 위한
좁은 번들 플러그인 재설치 경로입니다.

## 플러그인 유형

OpenClaw는 두 가지 플러그인 형식을 인식합니다.

| 형식       | 작동 방식                                                        | 예시                                                   |
| ---------- | ---------------------------------------------------------------- | ------------------------------------------------------ |
| **Native** | `openclaw.plugin.json` + 런타임 모듈; 프로세스 내에서 실행됨     | 공식 플러그인, 커뮤니티 npm 패키지                     |
| **Bundle** | Codex/Claude/Cursor 호환 레이아웃; OpenClaw 기능에 매핑됨        | `.codex-plugin/`, `.claude-plugin/`, `.cursor-plugin/` |

두 형식 모두 `openclaw plugins list`에 표시됩니다. bundle에 대한 자세한 내용은 [Plugin Bundles](/ko/plugins/bundles)를 참고하세요.

기본 플러그인을 작성하는 경우 [Building Plugins](/ko/plugins/building-plugins)
및 [Plugin SDK Overview](/ko/plugins/sdk-overview)에서 시작하세요.

## 공식 플러그인

### 설치 가능(npm)

| 플러그인         | 패키지                | 문서                                 |
| ---------------- | --------------------- | ------------------------------------ |
| Matrix           | `@openclaw/matrix`    | [Matrix](/ko/channels/matrix)           |
| Microsoft Teams  | `@openclaw/msteams`   | [Microsoft Teams](/ko/channels/msteams) |
| Nostr            | `@openclaw/nostr`     | [Nostr](/ko/channels/nostr)             |
| Voice Call       | `@openclaw/voice-call`| [Voice Call](/ko/plugins/voice-call)    |
| Zalo             | `@openclaw/zalo`      | [Zalo](/ko/channels/zalo)               |
| Zalo Personal    | `@openclaw/zalouser`  | [Zalo Personal](/ko/plugins/zalouser)   |

### Core(OpenClaw와 함께 제공)

<AccordionGroup>
  <Accordion title="모델 provider(기본 활성화)">
    `anthropic`, `byteplus`, `cloudflare-ai-gateway`, `github-copilot`, `google`,
    `huggingface`, `kilocode`, `kimi-coding`, `minimax`, `mistral`, `qwen`,
    `moonshot`, `nvidia`, `openai`, `opencode`, `opencode-go`, `openrouter`,
    `qianfan`, `synthetic`, `together`, `venice`,
    `vercel-ai-gateway`, `volcengine`, `xiaomi`, `zai`
  </Accordion>

  <Accordion title="메모리 플러그인">
    - `memory-core` — 번들 메모리 검색(`plugins.slots.memory`를 통한 기본값)
    - `memory-lancedb` — install-on-demand 장기 메모리, auto-recall/capture 포함(`plugins.slots.memory = "memory-lancedb"`로 설정)
  </Accordion>

  <Accordion title="음성 provider(기본 활성화)">
    `elevenlabs`, `microsoft`
  </Accordion>

  <Accordion title="기타">
    - `browser` — browser tool, `openclaw browser` CLI, `browser.request` gateway method, browser runtime, 기본 browser control service용 번들 browser 플러그인(기본 활성화; 교체 전에 비활성화 필요)
    - `copilot-proxy` — VS Code Copilot Proxy bridge(기본 비활성화)
  </Accordion>
</AccordionGroup>

서드파티 플러그인을 찾고 있나요? [Community Plugins](/ko/plugins/community)를 참고하세요.

## 구성

```json5
{
  plugins: {
    enabled: true,
    allow: ["voice-call"],
    deny: ["untrusted-plugin"],
    load: { paths: ["~/Projects/oss/voice-call-extension"] },
    entries: {
      "voice-call": { enabled: true, config: { provider: "twilio" } },
    },
  },
}
```

| 필드             | 설명                                                     |
| ---------------- | -------------------------------------------------------- |
| `enabled`        | 마스터 토글(기본값: `true`)                              |
| `allow`          | 플러그인 allowlist(선택 사항)                            |
| `deny`           | 플러그인 denylist(선택 사항; deny가 우선)                |
| `load.paths`     | 추가 플러그인 파일/디렉터리                              |
| `slots`          | 독점 슬롯 선택자(예: `memory`, `contextEngine`)          |
| `entries.\<id\>` | 플러그인별 토글 + config                                 |

config 변경은 **gateway 재시작이 필요합니다**. Gateway가 config
watch + in-process restart가 활성화된 상태로 실행 중이면(기본 `openclaw gateway` 경로),
그 재시작은 일반적으로 config 쓰기가 완료된 직후 자동으로 수행됩니다.

<Accordion title="플러그인 상태: disabled vs missing vs invalid">
  - **Disabled**: 플러그인은 존재하지만 활성화 규칙에 따라 꺼져 있습니다. Config는 유지됩니다.
  - **Missing**: config가 discovery에서 찾지 못한 플러그인 id를 참조합니다.
  - **Invalid**: 플러그인은 존재하지만 config가 선언된 schema와 일치하지 않습니다.
</Accordion>

## Discovery 및 우선순위

OpenClaw는 다음 순서로 플러그인을 스캔합니다(먼저 일치한 항목이 우선).

<Steps>
  <Step title="Config 경로">
    `plugins.load.paths` — 명시적 파일 또는 디렉터리 경로.
  </Step>

  <Step title="Workspace extensions">
    `\<workspace\>/.openclaw/<plugin-root>/*.ts` 및 `\<workspace\>/.openclaw/<plugin-root>/*/index.ts`.
  </Step>

  <Step title="전역 extensions">
    `~/.openclaw/<plugin-root>/*.ts` 및 `~/.openclaw/<plugin-root>/*/index.ts`.
  </Step>

  <Step title="번들 플러그인">
    OpenClaw와 함께 제공됩니다. 많은 항목이 기본적으로 활성화됩니다(모델 providers, speech).
    다른 항목은 명시적으로 활성화해야 합니다.
  </Step>
</Steps>

### 활성화 규칙

- `plugins.enabled: false`는 모든 플러그인을 비활성화합니다
- `plugins.deny`는 항상 allow보다 우선합니다
- `plugins.entries.\<id\>.enabled: false`는 해당 플러그인을 비활성화합니다
- workspace 출처 플러그인은 **기본적으로 비활성화**됩니다(명시적으로 활성화해야 함)
- 번들 플러그인은 재정의되지 않는 한 기본 활성화 집합을 따릅니다
- 독점 슬롯은 해당 슬롯에 선택된 플러그인을 강제로 활성화할 수 있습니다

## 플러그인 슬롯(독점 카테고리)

일부 카테고리는 독점적입니다(한 번에 하나만 활성화 가능).

```json5
{
  plugins: {
    slots: {
      memory: "memory-core", // 또는 비활성화하려면 "none"
      contextEngine: "legacy", // 또는 플러그인 id
    },
  },
}
```

| 슬롯             | 제어 대상              | 기본값             |
| ---------------- | ---------------------- | ------------------ |
| `memory`         | 활성 메모리 플러그인   | `memory-core`      |
| `contextEngine`  | 활성 컨텍스트 엔진     | `legacy` (내장)    |

## CLI 참조

```bash
openclaw plugins list                       # 간단한 인벤토리
openclaw plugins list --enabled            # 로드된 플러그인만
openclaw plugins list --verbose            # 플러그인별 상세 줄
openclaw plugins list --json               # 기계 판독용 인벤토리
openclaw plugins inspect <id>              # 상세 정보
openclaw plugins inspect <id> --json       # 기계 판독용
openclaw plugins inspect --all             # 전체 테이블
openclaw plugins info <id>                 # inspect 별칭
openclaw plugins doctor                    # 진단

openclaw plugins install <package>         # 설치(먼저 ClawHub, 그다음 npm)
openclaw plugins install clawhub:<pkg>     # ClawHub에서만 설치
openclaw plugins install <spec> --force    # 기존 설치 덮어쓰기
openclaw plugins install <path>            # 로컬 경로에서 설치
openclaw plugins install -l <path>         # 개발용 링크(복사 안 함)
openclaw plugins install <plugin> --marketplace <source>
openclaw plugins install <plugin> --marketplace https://github.com/<owner>/<repo>
openclaw plugins install <spec> --pin      # 정확히 해결된 npm spec 기록
openclaw plugins install <spec> --dangerously-force-unsafe-install
openclaw plugins update <id>             # 플러그인 하나 업데이트
openclaw plugins update <id> --dangerously-force-unsafe-install
openclaw plugins update --all            # 모두 업데이트
openclaw plugins uninstall <id>          # config/설치 기록 제거
openclaw plugins uninstall <id> --keep-files
openclaw plugins marketplace list <source>
openclaw plugins marketplace list <source> --json

openclaw plugins enable <id>
openclaw plugins disable <id>
```

번들 플러그인은 OpenClaw와 함께 제공됩니다. 많은 항목이 기본적으로 활성화됩니다(예:
번들 모델 providers, 번들 speech providers, 번들 browser
plugin). 다른 번들 플러그인은 여전히 `openclaw plugins enable <id>`가 필요합니다.

`--force`는 기존에 설치된 플러그인 또는 hook pack을 같은 위치에 덮어씁니다.
이 옵션은 소스 경로를 재사용하고 관리되는 설치 대상을 덮어쓰지 않는 `--link`와 함께는 지원되지 않습니다.

`--pin`은 npm 전용입니다. marketplace 설치는
npm spec 대신 marketplace source metadata를 유지하므로 `--marketplace`와 함께는 지원되지 않습니다.

`--dangerously-force-unsafe-install`은 내장 위험 코드 스캐너의 false
positive를 위한 비상용 재정의입니다. 플러그인 설치 및 플러그인 업데이트가 내장
`critical` 결과를 넘어 계속 진행되도록 허용하지만,
플러그인 `before_install` 정책 차단이나 scan-failure 차단까지 우회하지는 않습니다.

이 CLI 플래그는 플러그인 설치/업데이트 흐름에만 적용됩니다. Gateway 기반 skill
dependency 설치는 대신 대응하는 `dangerouslyForceUnsafeInstall` 요청 재정의를 사용하며,
`openclaw skills install`은 별도의 ClawHub
skill 다운로드/설치 흐름으로 유지됩니다.

호환 bundle은 같은 플러그인 목록/inspect/enable/disable
흐름에 참여합니다. 현재 런타임 지원에는 bundle skills, Claude command-skills,
Claude `settings.json` 기본값, Claude `.lsp.json` 및 manifest에 선언된
`lspServers` 기본값, Cursor command-skills, 호환되는 Codex hook
디렉터리가 포함됩니다.

`openclaw plugins inspect <id>`는 bundle 기반 플러그인에 대해 감지된 bundle capability와
지원되거나 지원되지 않는 MCP 및 LSP server 항목도 보고합니다.

Marketplace source는
`~/.claude/plugins/known_marketplaces.json`의 Claude 알려진 marketplace 이름,
로컬 marketplace 루트 또는 `marketplace.json` 경로, `owner/repo` 같은 GitHub 축약형,
GitHub repo URL 또는 git URL이 될 수 있습니다. 원격 marketplace의 경우,
플러그인 항목은 복제된 marketplace repo 내부에 있어야 하며
상대 경로 source만 사용해야 합니다.

전체 내용은 [`openclaw plugins` CLI reference](/cli/plugins)를 참고하세요.

## 플러그인 API 개요

기본 플러그인은 `register(api)`를 노출하는 엔트리 객체를 export합니다. 오래된
플러그인은 여전히 레거시 별칭인 `activate(api)`를 사용할 수 있지만, 새 플러그인은
`register`를 사용해야 합니다.

```typescript
export default definePluginEntry({
  id: "my-plugin",
  name: "My Plugin",
  register(api) {
    api.registerProvider({
      /* ... */
    });
    api.registerTool({
      /* ... */
    });
    api.registerChannel({
      /* ... */
    });
  },
});
```

OpenClaw는 엔트리 객체를 로드하고 플러그인
활성화 중에 `register(api)`를 호출합니다. 로더는 여전히 오래된 플러그인을 위해
`activate(api)`로 fallback하지만, 번들 플러그인과 새 외부 플러그인은
`register`를 공개 계약으로 취급해야 합니다.

일반적인 등록 메서드:

| 메서드                                  | 등록 대상                     |
| --------------------------------------- | ----------------------------- |
| `registerProvider`                      | 모델 provider (LLM)           |
| `registerChannel`                       | 채팅 채널                    |
| `registerTool`                          | 에이전트 도구                |
| `registerHook` / `on(...)`              | lifecycle hooks              |
| `registerSpeechProvider`                | 텍스트 음성 변환 / STT        |
| `registerRealtimeTranscriptionProvider` | 스트리밍 STT                 |
| `registerRealtimeVoiceProvider`         | 양방향 realtime voice        |
| `registerMediaUnderstandingProvider`    | 이미지/오디오 분석           |
| `registerImageGenerationProvider`       | 이미지 생성                  |
| `registerMusicGenerationProvider`       | 음악 생성                    |
| `registerVideoGenerationProvider`       | 비디오 생성                  |
| `registerWebFetchProvider`              | 웹 fetch / scrape provider   |
| `registerWebSearchProvider`             | 웹 검색                      |
| `registerHttpRoute`                     | HTTP endpoint                |
| `registerCommand` / `registerCli`       | CLI commands                 |
| `registerContextEngine`                 | 컨텍스트 엔진                |
| `registerService`                       | 백그라운드 service           |

typed lifecycle hook의 hook guard 동작:

- `before_tool_call`: `{ block: true }`는 종료 동작이며, 더 낮은 우선순위 handler는 건너뜁니다.
- `before_tool_call`: `{ block: false }`는 no-op이며 이전 block을 해제하지 않습니다.
- `before_install`: `{ block: true }`는 종료 동작이며, 더 낮은 우선순위 handler는 건너뜁니다.
- `before_install`: `{ block: false }`는 no-op이며 이전 block을 해제하지 않습니다.
- `message_sending`: `{ cancel: true }`는 종료 동작이며, 더 낮은 우선순위 handler는 건너뜁니다.
- `message_sending`: `{ cancel: false }`는 no-op이며 이전 cancel을 해제하지 않습니다.

typed hook 전체 동작은 [SDK Overview](/ko/plugins/sdk-overview#hook-decision-semantics)를 참고하세요.

## 관련 문서

- [Building Plugins](/ko/plugins/building-plugins) — 직접 플러그인 만들기
- [Plugin Bundles](/ko/plugins/bundles) — Codex/Claude/Cursor bundle 호환성
- [Plugin Manifest](/ko/plugins/manifest) — manifest schema
- [Registering Tools](/ko/plugins/building-plugins#registering-agent-tools) — 플러그인에 에이전트 도구 추가
- [Plugin Internals](/ko/plugins/architecture) — capability 모델 및 로드 파이프라인
- [Community Plugins](/ko/plugins/community) — 서드파티 목록
