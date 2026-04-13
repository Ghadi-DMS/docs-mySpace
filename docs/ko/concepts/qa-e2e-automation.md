---
read_when:
    - qa-lab 또는 qa-channel 확장하기
    - 리포지토리 기반 QA 시나리오 추가하기
    - Gateway 대시보드를 중심으로 더 높은 현실성의 QA 자동화 구축하기
summary: qa-lab, qa-channel, 시드된 시나리오, 그리고 프로토콜 보고서를 위한 비공개 QA 자동화 구조
title: QA E2E 자동화
x-i18n:
    generated_at: "2026-04-13T06:05:37Z"
    model: gpt-5.4
    provider: openai
    source_hash: a4a4f5c765163565c95c2a071f201775fd9d8d60cad4ff25d71e4710559c1570
    source_path: concepts/qa-e2e-automation.md
    workflow: 15
---

# QA E2E 자동화

비공개 QA 스택은 단일 단위 테스트로는 할 수 없는 방식으로, OpenClaw를 더 현실적이고
채널 형태에 가깝게 검증하기 위한 것입니다.

현재 구성 요소:

- `extensions/qa-channel`: DM, 채널, 스레드, 리액션, 수정, 삭제 표면을 갖춘 합성 메시지 채널
- `extensions/qa-lab`: 대화 내용을 관찰하고, 인바운드 메시지를 주입하며, Markdown 보고서를 내보내기 위한 디버거 UI 및 QA 버스
- `qa/`: 시작 작업과 기준 QA 시나리오를 위한 리포지토리 기반 시드 자산

현재 QA 운영자 흐름은 2개 패널로 구성된 QA 사이트입니다:

- 왼쪽: 에이전트가 있는 Gateway 대시보드(Control UI)
- 오른쪽: Slack 비슷한 대화 내용과 시나리오 계획을 보여주는 QA Lab

다음 명령으로 실행합니다:

```bash
pnpm qa:lab:up
```

이 명령은 QA 사이트를 빌드하고, Docker 기반 gateway 레인을 시작하며, 운영자나 자동화 루프가
에이전트에게 QA 미션을 부여하고, 실제 채널 동작을 관찰하며, 무엇이 동작했고 실패했는지,
혹은 막힌 상태로 남았는지를 기록할 수 있는 QA Lab 페이지를 노출합니다.

매번 Docker 이미지를 다시 빌드하지 않고 더 빠르게 QA Lab UI를 반복 개발하려면,
바인드 마운트된 QA Lab 번들과 함께 스택을 시작하세요:

```bash
pnpm openclaw qa docker-build-image
pnpm qa:lab:build
pnpm qa:lab:up:fast
pnpm qa:lab:watch
```

`qa:lab:up:fast`는 Docker 서비스를 사전 빌드된 이미지로 유지하고,
`extensions/qa-lab/web/dist`를 `qa-lab` 컨테이너에 바인드 마운트합니다. `qa:lab:watch`는
변경 시 해당 번들을 다시 빌드하며, QA Lab 자산 해시가 바뀌면 브라우저가 자동으로 새로고침됩니다.

전송 계층이 실제인 Matrix 스모크 레인을 실행하려면 다음을 사용하세요:

```bash
pnpm openclaw qa matrix
```

이 레인은 Docker에서 일회용 Tuwunel 홈서버를 프로비저닝하고, 임시 드라이버, SUT, 관찰자 사용자를 등록하며,
비공개 룸 하나를 만든 뒤, 실제 Matrix Plugin을 QA gateway child 안에서 실행합니다. 라이브 전송 레인은
테스트 중인 전송 계층으로 child config 범위를 제한하므로, Matrix는 child config에서 `qa-channel`
없이 실행됩니다.

전송 계층이 실제인 Telegram 스모크 레인을 실행하려면 다음을 사용하세요:

```bash
pnpm openclaw qa telegram
```

이 레인은 일회용 서버를 프로비저닝하는 대신 실제 비공개 Telegram 그룹 하나를 대상으로 합니다.
이를 위해 `OPENCLAW_QA_TELEGRAM_GROUP_ID`,
`OPENCLAW_QA_TELEGRAM_DRIVER_BOT_TOKEN`,
`OPENCLAW_QA_TELEGRAM_SUT_BOT_TOKEN`이 필요하며,
같은 비공개 그룹에 있는 서로 다른 두 개의 봇도 필요합니다. SUT 봇은 Telegram 사용자 이름이 있어야 하며,
봇 간 관찰은 두 봇 모두 `@BotFather`에서 Bot-to-Bot Communication Mode를 활성화했을 때 가장 잘 동작합니다.

이제 라이브 전송 레인들은 각자 자체 시나리오 목록 형태를 만드는 대신 더 작은 하나의 공통 계약을 공유합니다:

`qa-channel`은 여전히 폭넓은 합성 제품 동작 스위트이며, 라이브 전송 커버리지 매트릭스에는 포함되지 않습니다.

| 레인     | 카나리 | 멘션 게이팅 | 허용 목록 차단 | 최상위 답장 | 재시작 후 재개 | 스레드 후속 응답 | 스레드 격리 | 리액션 관찰 | 도움말 명령 |
| -------- | ------ | ----------- | -------------- | ----------- | -------------- | ---------------- | ----------- | ----------- | ------------ |
| Matrix   | x      | x           | x              | x           | x              | x                | x           | x           |              |
| Telegram | x      |             |                |             |                |                  |             |             | x            |

이를 통해 `qa-channel`은 폭넓은 제품 동작 스위트로 유지되고, Matrix, Telegram, 그리고 향후 라이브 전송 계층은
하나의 명시적인 전송 계약 체크리스트를 공유하게 됩니다.

Docker를 QA 경로에 포함하지 않는 일회용 Linux VM 레인을 실행하려면 다음을 사용하세요:

```bash
pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline
```

이 명령은 새 Multipass 게스트를 부팅하고, 의존성을 설치하고, 게스트 안에서 OpenClaw를 빌드하고,
`qa suite`를 실행한 뒤, 일반 QA 보고서와 요약을 호스트의 `.artifacts/qa-e2e/...`로 다시 복사합니다.
시나리오 선택 동작은 호스트에서의 `qa suite`와 동일하게 재사용합니다.
호스트와 Multipass suite 실행은 기본적으로 선택된 여러 시나리오를 격리된 gateway worker와 함께 병렬 실행하며,
최대 64개 worker 또는 선택된 시나리오 수까지 사용합니다. worker 수를 조정하려면 `--concurrency <count>`를,
직렬 실행하려면 `--concurrency 1`을 사용하세요.
라이브 실행은 게스트에서 실용적인 범위의 지원되는 QA 인증 입력을 전달합니다. 여기에는 env 기반 provider 키,
QA 라이브 provider config 경로, 그리고 존재할 경우 `CODEX_HOME`이 포함됩니다. 게스트가 마운트된 워크스페이스를 통해
다시 쓸 수 있도록 `--output-dir`은 리포지토리 루트 아래로 유지하세요.

## 리포지토리 기반 시드

시드 자산은 `qa/`에 있습니다:

- `qa/scenarios/index.md`
- `qa/scenarios/*.md`

이 파일들은 QA 계획이 사람과 에이전트 모두에게 보이도록 의도적으로 git에 포함되어 있습니다.

`qa-lab`은 범용 Markdown 러너로 유지되어야 합니다. 각 시나리오 Markdown 파일은 하나의 테스트 실행에 대한
source of truth여야 하며, 다음을 정의해야 합니다:

- 시나리오 메타데이터
- 문서 및 코드 참조
- 선택적 Plugin 요구 사항
- 선택적 gateway config 패치
- 실행 가능한 `qa-flow`

`qa-flow`를 뒷받침하는 재사용 가능한 런타임 표면은 범용적이고 교차 기능적으로 유지되어도 됩니다. 예를 들어,
Markdown 시나리오는 transport별 QA 러너를 추가하지 않고도, Gateway `browser.request` seam을 통해 내장된
Control UI를 구동하는 브라우저 측 헬퍼와 전송 계층 측 헬퍼를 조합할 수 있습니다.

기준 목록은 다음을 포괄할 만큼 충분히 넓어야 합니다:

- DM 및 채널 채팅
- 스레드 동작
- 메시지 액션 수명 주기
- Cron 콜백
- 메모리 회상
- 모델 전환
- 서브에이전트 핸드오프
- 리포지토리 읽기 및 문서 읽기
- Lobster Invaders 같은 작은 빌드 작업 하나

## 전송 어댑터

`qa-lab`은 Markdown QA 시나리오를 위한 범용 전송 seam을 소유합니다.
`qa-channel`은 그 seam 위의 첫 번째 어댑터이지만, 설계 목표는 더 넓습니다.
향후 실제 또는 합성 채널은 전송별 QA 러너를 추가하는 대신 동일한 suite runner에 연결되어야 합니다.

아키텍처 수준에서의 분리는 다음과 같습니다:

- `qa-lab`은 범용 시나리오 실행, worker 동시성, 아티팩트 기록, 보고를 소유합니다.
- 전송 어댑터는 gateway config, 준비 상태, 인바운드 및 아웃바운드 관찰, 전송 액션, 정규화된 전송 상태를 소유합니다.
- `qa/scenarios/` 아래의 Markdown 시나리오 파일이 테스트 실행을 정의하며, `qa-lab`은 이를 실행하는 재사용 가능한 런타임 표면을 제공합니다.

새 채널 어댑터에 대한 유지관리자용 도입 가이드는
[Testing](/ko/help/testing#adding-a-channel-to-qa)에 있습니다.

## 보고

`qa-lab`은 관찰된 버스 타임라인으로부터 Markdown 프로토콜 보고서를 내보냅니다.
이 보고서는 다음 질문에 답해야 합니다:

- 무엇이 동작했는가
- 무엇이 실패했는가
- 무엇이 막힌 상태로 남았는가
- 어떤 후속 시나리오를 추가할 가치가 있는가

캐릭터 및 스타일 검사를 위해서는, 동일한 시나리오를 여러 라이브 모델 ref로 실행하고
판정된 Markdown 보고서를 작성하세요:

```bash
pnpm openclaw qa character-eval \
  --model openai/gpt-5.4,thinking=xhigh \
  --model openai/gpt-5.2,thinking=xhigh \
  --model openai/gpt-5,thinking=xhigh \
  --model anthropic/claude-opus-4-6,thinking=high \
  --model anthropic/claude-sonnet-4-6,thinking=high \
  --model zai/glm-5.1,thinking=high \
  --model moonshot/kimi-k2.5,thinking=high \
  --model google/gemini-3.1-pro-preview,thinking=high \
  --judge-model openai/gpt-5.4,thinking=xhigh,fast \
  --judge-model anthropic/claude-opus-4-6,thinking=high \
  --blind-judge-models \
  --concurrency 16 \
  --judge-concurrency 16
```

이 명령은 Docker가 아니라 로컬 QA gateway child 프로세스를 실행합니다. Character eval
시나리오는 `SOUL.md`를 통해 페르소나를 설정한 다음, 채팅, 워크스페이스 도움말, 작은 파일 작업 같은
일반 사용자 턴을 실행해야 합니다. 후보 모델에게는 자신이 평가되고 있다는 사실을 알려서는 안 됩니다.
이 명령은 각 전체 대화 내용을 보존하고, 기본 실행 통계를 기록한 뒤, 판정 모델에 fast mode와
`xhigh` 추론을 사용해 자연스러움, 분위기, 유머 기준으로 실행 결과의 순위를 매기도록 요청합니다.
provider를 비교할 때는 `--blind-judge-models`를 사용하세요. 이 경우 판정 프롬프트는 여전히 모든 대화 내용과
실행 상태를 받지만, 후보 ref는 `candidate-01` 같은 중립 라벨로 대체되며, 보고서는 파싱 후 순위를 실제 ref에 다시 매핑합니다.
후보 실행은 기본적으로 `high` thinking을 사용하며, 이를 지원하는 OpenAI 모델은 `xhigh`를 사용합니다.
특정 후보를 개별적으로 재정의하려면
`--model provider/model,thinking=<level>`을 사용하세요. `--thinking <level>`은 여전히
전역 폴백을 설정하며, 기존의 `--model-thinking <provider/model=level>` 형식도 호환성을 위해 유지됩니다.
OpenAI 후보 ref는 provider가 지원하는 경우 우선 처리에 fast mode를 기본 사용합니다.
특정 후보나 판정 모델에 재정의가 필요하면 `,fast`, `,no-fast`, 또는 `,fast=false`를 개별적으로 추가하세요.
모든 후보 모델에 대해 fast mode를 강제로 켜고 싶을 때만 `--fast`를 전달하세요. 후보 및 판정 실행 시간은
벤치마크 분석을 위해 보고서에 기록되지만, 판정 프롬프트에는 속도로 순위를 매기지 말라고 명시되어 있습니다.
후보와 판정 모델 실행은 둘 다 기본적으로 동시성 16을 사용합니다. provider 제한이나 로컬 gateway 부하 때문에
실행이 너무 불안정해지면 `--concurrency` 또는 `--judge-concurrency`를 낮추세요.
후보 `--model`을 전달하지 않으면 character eval은 기본적으로
`openai/gpt-5.4`, `openai/gpt-5.2`, `openai/gpt-5`, `anthropic/claude-opus-4-6`,
`anthropic/claude-sonnet-4-6`, `zai/glm-5.1`,
`moonshot/kimi-k2.5`, 그리고
`google/gemini-3.1-pro-preview`를 사용합니다.
판정용 `--judge-model`을 전달하지 않으면 판정 모델 기본값은
`openai/gpt-5.4,thinking=xhigh,fast`와
`anthropic/claude-opus-4-6,thinking=high`입니다.

## 관련 문서

- [Testing](/ko/help/testing)
- [QA Channel](/ko/channels/qa-channel)
- [Dashboard](/web/dashboard)
