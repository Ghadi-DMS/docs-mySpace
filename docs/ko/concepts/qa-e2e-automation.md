---
read_when:
    - qa-lab 또는 qa-channel을 확장하려는 경우
    - 저장소 기반 QA 시나리오를 추가하려는 경우
    - Gateway 대시보드를 중심으로 더 현실감 있는 QA 자동화를 구축하려는 경우
summary: qa-lab, qa-channel, 시드 시나리오, 프로토콜 보고서를 위한 비공개 QA 자동화 구조
title: QA E2E 자동화
x-i18n:
    generated_at: "2026-04-06T03:06:52Z"
    model: gpt-5.4
    provider: openai
    source_hash: df35f353d5ab0e0432e6a828c82772f9a88edb41c20ec5037315b7ba310b28e6
    source_path: concepts/qa-e2e-automation.md
    workflow: 15
---

# QA E2E 자동화

비공개 QA 스택은 단일 단위 테스트보다 더 현실적이고,
채널 형태에 가까운 방식으로 OpenClaw를 검증하도록 설계되었습니다.

현재 구성 요소:

- `extensions/qa-channel`: DM, 채널, 스레드, 리액션, 수정, 삭제 기능을 갖춘 합성 메시지 채널
- `extensions/qa-lab`: 대화 내용을 관찰하고, 인바운드 메시지를 주입하며, Markdown 보고서를 내보내기 위한 디버거 UI 및 QA 버스
- `qa/`: 시작 작업과 기본 QA 시나리오를 위한 저장소 기반 시드 자산

장기 목표는 두 개의 패널로 구성된 QA 사이트입니다.

- 왼쪽: 에이전트가 있는 Gateway 대시보드(Control UI)
- 오른쪽: Slack과 유사한 대화 내용과 시나리오 계획을 보여주는 QA Lab

이렇게 하면 운영자나 자동화 루프가 에이전트에 QA 임무를 부여하고,
실제 채널 동작을 관찰하며, 무엇이 작동했고 실패했으며 계속 막혀 있었는지를 기록할 수 있습니다.

## 저장소 기반 시드

시드 자산은 `qa/`에 있습니다.

- `qa/QA_KICKOFF_TASK.md`
- `qa/seed-scenarios.json`

이 파일들은 QA 계획이 사람과 에이전트 모두에게 보이도록 의도적으로 git에 포함됩니다.
기본 목록은 다음을 포괄할 수 있을 만큼 충분히 넓어야 합니다.

- DM 및 채널 채팅
- 스레드 동작
- 메시지 액션 수명 주기
- cron 콜백
- 메모리 회상
- 모델 전환
- subagent 핸드오프
- 저장소 읽기 및 문서 읽기
- Lobster Invaders와 같은 작은 빌드 작업 하나

## 보고

`qa-lab`은 관찰된 버스 타임라인에서 Markdown 프로토콜 보고서를 내보냅니다.
이 보고서는 다음 질문에 답해야 합니다.

- 무엇이 작동했는가
- 무엇이 실패했는가
- 무엇이 계속 막혀 있었는가
- 어떤 후속 시나리오를 추가할 가치가 있는가

## 관련 문서

- [테스트](/ko/help/testing)
- [QA Channel](/channels/qa-channel)
- [대시보드](/web/dashboard)
