---
read_when:
    - qa-lab 또는 qa-channel 확장 시
    - 리포지토리 기반 QA 시나리오 추가 시
    - Gateway 대시보드를 중심으로 더 높은 현실성의 QA 자동화 구축 시
summary: qa-lab, qa-channel, 시드된 시나리오, 프로토콜 보고서를 위한 비공개 QA 자동화 구성
title: QA E2E 자동화
x-i18n:
    generated_at: "2026-04-09T01:27:48Z"
    model: gpt-5.4
    provider: openai
    source_hash: c922607d67e0f3a2489ac82bc9f510f7294ced039c1014c15b676d826441d833
    source_path: concepts/qa-e2e-automation.md
    workflow: 15
---

# QA E2E 자동화

비공개 QA 스택은 단일 단위 테스트보다 더 현실적이고
채널 형태에 가까운 방식으로 OpenClaw를 검증하도록 설계되었습니다.

현재 구성 요소:

- `extensions/qa-channel`: DM, 채널, 스레드,
  리액션, 편집, 삭제 인터페이스를 갖춘 합성 메시지 채널
- `extensions/qa-lab`: 트랜스크립트를 관찰하고,
  인바운드 메시지를 주입하며, Markdown 보고서를 내보내기 위한 디버거 UI 및 QA 버스
- `qa/`: 시작 작업과 기본 QA
  시나리오를 위한 리포지토리 기반 시드 자산

현재 QA 운영자 흐름은 2패널 QA 사이트입니다:

- 왼쪽: 에이전트가 있는 Gateway 대시보드(Control UI)
- 오른쪽: Slack 스타일의 트랜스크립트와 시나리오 계획을 보여주는 QA Lab

다음으로 실행합니다:

```bash
pnpm qa:lab:up
```

이 명령은 QA 사이트를 빌드하고, Docker 기반 gateway 레인을 시작하며, 운영자나 자동화 루프가 에이전트에 QA
미션을 부여하고, 실제 채널 동작을 관찰하며, 무엇이 작동했고 실패했으며
또는 계속 막혀 있었는지를 기록할 수 있는 QA Lab 페이지를 노출합니다.

매번 Docker 이미지를 다시 빌드하지 않고 QA Lab UI를 더 빠르게 반복 개발하려면,
bind mount된 QA Lab 번들로 스택을 시작하세요:

```bash
pnpm openclaw qa docker-build-image
pnpm qa:lab:build
pnpm qa:lab:up:fast
pnpm qa:lab:watch
```

`qa:lab:up:fast`는 사전 빌드된 이미지에서 Docker 서비스를 유지하고
`extensions/qa-lab/web/dist`를 `qa-lab` 컨테이너에 bind mount합니다. `qa:lab:watch`는
변경 시 해당 번들을 다시 빌드하고, QA Lab 자산 해시가 변경되면 브라우저가 자동으로 다시 로드됩니다.

## 리포지토리 기반 시드

시드 자산은 `qa/`에 있습니다:

- `qa/scenarios/index.md`
- `qa/scenarios/*.md`

이 파일들은 QA 계획이 사람과
에이전트 모두에게 보이도록 의도적으로 git에 포함됩니다. 기본 목록은 다음을 포괄할 만큼 충분히 넓게 유지되어야 합니다:

- DM 및 채널 채팅
- 스레드 동작
- 메시지 액션 수명 주기
- cron 콜백
- 메모리 회상
- 모델 전환
- 서브에이전트 핸드오프
- 리포지토리 읽기 및 문서 읽기
- Lobster Invaders 같은 작은 빌드 작업 하나

## 보고

`qa-lab`은 관찰된 버스 타임라인에서 Markdown 프로토콜 보고서를 내보냅니다.
보고서는 다음에 답해야 합니다:

- 무엇이 작동했는가
- 무엇이 실패했는가
- 무엇이 계속 막혀 있었는가
- 어떤 후속 시나리오를 추가할 가치가 있는가

캐릭터 및 스타일 검사를 위해 동일한 시나리오를 여러 라이브 모델
ref에서 실행하고 판단된 Markdown 보고서를 작성하세요:

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

이 명령은 Docker가 아니라 로컬 QA gateway 자식 프로세스를 실행합니다. 캐릭터 평가
시나리오는 `SOUL.md`를 통해 페르소나를 설정한 다음, 채팅, 워크스페이스 도움말, 작은 파일 작업과 같은
일반 사용자 턴을 실행해야 합니다. 후보 모델에는 자신이 평가되고 있다는 사실을 알려서는 안 됩니다.
이 명령은 각 전체 트랜스크립트를 보존하고, 기본 실행 통계를 기록한 뒤, judge 모델에 fast 모드와
`xhigh` 추론을 사용해 자연스러움, 분위기, 유머를 기준으로 실행 결과의 순위를 매기도록 요청합니다.
제공자들을 비교할 때는 `--blind-judge-models`를 사용하세요. 이 경우 judge 프롬프트는 여전히
모든 트랜스크립트와 실행 상태를 받지만, 후보 ref는 `candidate-01` 같은 중립적인
레이블로 대체됩니다. 보고서는 파싱 후 순위를 실제 ref에 다시 매핑합니다.
후보 실행은 기본적으로 `high` thinking을 사용하며, 이를 지원하는 OpenAI 모델은 `xhigh`를 사용합니다.
특정 후보를 개별적으로 재정의하려면
`--model provider/model,thinking=<level>`을 사용하세요. `--thinking <level>`은 여전히 전역 기본값을 설정하며,
기존의 `--model-thinking <provider/model=level>` 형식도 호환성을 위해 유지됩니다.
OpenAI 후보 ref는 기본적으로 fast 모드를 사용하므로, 제공자가 이를 지원하는 경우 우선 처리됩니다.
단일 후보나 judge에 재정의가 필요하면 인라인으로 `,fast`, `,no-fast`, 또는 `,fast=false`를 추가하세요.
모든 후보 모델에 fast 모드를 강제로 적용하려는 경우에만 `--fast`를 전달하세요. 후보와 judge의 실행 시간은
벤치마크 분석을 위해 보고서에 기록되지만, judge 프롬프트에는 속도로 순위를 매기지 말라고 명시되어 있습니다.
후보 및 judge 모델 실행은 모두 기본적으로 동시성 16을 사용합니다.
제공자 제한이나 로컬 gateway 부하 때문에 실행이 너무 불안정해지면
`--concurrency` 또는 `--judge-concurrency`를 낮추세요.
후보 `--model`이 전달되지 않으면 character eval은 기본적으로
`openai/gpt-5.4`, `openai/gpt-5.2`, `openai/gpt-5`, `anthropic/claude-opus-4-6`,
`anthropic/claude-sonnet-4-6`, `zai/glm-5.1`,
`moonshot/kimi-k2.5`, 그리고
`google/gemini-3.1-pro-preview`를 사용합니다.
`--judge-model`이 전달되지 않으면 judge는 기본적으로
`openai/gpt-5.4,thinking=xhigh,fast` 및
`anthropic/claude-opus-4-6,thinking=high`를 사용합니다.

## 관련 문서

- [테스트](/ko/help/testing)
- [QA 채널](/ko/channels/qa-channel)
- [대시보드](/web/dashboard)
