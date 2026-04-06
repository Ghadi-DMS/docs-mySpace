---
read_when:
    - 로컬 또는 CI 테스트 실행에 합성 QA 전송을 연결하고 있습니다
    - 번들된 qa-channel 구성 표면이 필요합니다
    - 엔드 투 엔드 QA 자동화를 반복 개선하고 있습니다
summary: 결정론적인 OpenClaw QA 시나리오를 위한 합성 Slack급 채널 plugin
title: QA 채널
x-i18n:
    generated_at: "2026-04-06T03:05:54Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3b88cd73df2f61b34ad1eb83c3450f8fe15a51ac69fbb5a9eca0097564d67a06
    source_path: channels/qa-channel.md
    workflow: 15
---

# QA 채널

`qa-channel`은 자동화된 OpenClaw QA를 위한 번들된 합성 메시지 전송입니다.

이것은 프로덕션 채널이 아닙니다. 상태를 결정론적이고 완전히
검사 가능하게 유지하면서 실제 전송이 사용하는 동일한 채널 plugin
경계를 시험하기 위해 존재합니다.

## 현재 수행하는 작업

- Slack급 대상 문법:
  - `dm:<user>`
  - `channel:<room>`
  - `thread:<room>/<thread>`
- 다음을 위한 HTTP 기반 합성 버스:
  - 인바운드 메시지 주입
  - 아웃바운드 transcript 캡처
  - 스레드 생성
  - 반응
  - 편집
  - 삭제
  - 검색 및 읽기 작업
- Markdown 보고서를 작성하는 번들된 호스트 측 자체 점검 실행기

## 구성

```json
{
  "channels": {
    "qa-channel": {
      "baseUrl": "http://127.0.0.1:43123",
      "botUserId": "openclaw",
      "botDisplayName": "OpenClaw QA",
      "allowFrom": ["*"],
      "pollTimeoutMs": 1000
    }
  }
}
```

지원되는 계정 키:

- `baseUrl`
- `botUserId`
- `botDisplayName`
- `pollTimeoutMs`
- `allowFrom`
- `defaultTo`
- `actions.messages`
- `actions.reactions`
- `actions.search`
- `actions.threads`

## 실행기

현재 수직 슬라이스:

```bash
pnpm qa:e2e
```

이제 이것은 번들된 `qa-lab` extension을 통해 라우팅됩니다. 리포지토리 내
QA 버스를 시작하고, 번들된 `qa-channel` 런타임 슬라이스를 부팅하고, 결정론적
자체 점검을 실행하며, `.artifacts/qa-e2e/` 아래에 Markdown 보고서를 작성합니다.

비공개 디버거 UI:

```bash
pnpm qa:lab:build
pnpm openclaw qa ui
```

전체 리포지토리 기반 QA 모음:

```bash
pnpm openclaw qa suite
```

이것은 배포된 Control UI 번들과 별도로 로컬 URL에서
비공개 QA 디버거를 실행합니다.

## 범위

현재 범위는 의도적으로 좁습니다:

- 버스 + plugin 전송
- 스레드 라우팅 문법
- 채널 소유 메시지 작업
- Markdown 보고

후속 작업에서 다음이 추가될 예정입니다:

- Dockerized OpenClaw 오케스트레이션
- provider/model 매트릭스 실행
- 더 풍부한 시나리오 검색
- 이후 OpenClaw 네이티브 오케스트레이션
