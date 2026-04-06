---
read_when:
    - 메모리 승격이 자동으로 실행되게 하려는 경우
    - 각 dreaming 단계가 무엇을 하는지 이해하려는 경우
    - MEMORY.md를 오염시키지 않고 통합을 조정하려는 경우
summary: 가벼운 수면, 깊은 수면, REM 단계와 Dream Diary를 포함한 백그라운드 메모리 통합
title: Dreaming(실험적)
x-i18n:
    generated_at: "2026-04-06T03:06:14Z"
    model: gpt-5.4
    provider: openai
    source_hash: f27da718176bebf59fe8a80fddd4fb5b6d814ac5647f6c1e8344bcfb328db9de
    source_path: concepts/dreaming.md
    workflow: 15
---

# Dreaming(실험적)

Dreaming은 `memory-core`의 백그라운드 메모리 통합 시스템입니다.
이 기능은 OpenClaw가 강한 단기 신호를 오래 유지되는 메모리로 이동하도록 도와주면서,
그 과정을 설명 가능하고 검토 가능하게 유지합니다.

Dreaming은 **옵트인** 기능이며 기본적으로 비활성화되어 있습니다.

## dreaming이 기록하는 내용

Dreaming은 두 종류의 출력을 유지합니다.

- `memory/.dreams/`의 **기계 상태**(recall 저장소, 단계 신호, 수집 체크포인트, 잠금)
- `DREAMS.md`(또는 기존 `dreams.md`)의 **사람이 읽을 수 있는 출력** 및 `memory/dreaming/<phase>/YYYY-MM-DD.md` 아래의 선택적 단계 보고서 파일

장기 승격은 여전히 `MEMORY.md`에만 기록됩니다.

## 단계 모델

Dreaming은 세 가지 협력 단계로 동작합니다.

| 단계 | 목적 | 영속적 기록 |
| ----- | ---- | ----------- |
| Light | 최근 단기 자료를 정렬하고 준비 | 아니요 |
| Deep  | 오래 유지할 후보를 점수화하고 승격 | 예 (`MEMORY.md`) |
| REM   | 주제와 반복되는 아이디어를 성찰 | 아니요 |

이 단계들은 내부 구현 세부 사항이며, 사용자별로 따로 구성하는
"모드"가 아닙니다.

### Light 단계

Light 단계는 최근 일일 메모리 신호와 recall 추적을 수집하고, 중복을 제거하며,
후보 줄을 준비합니다.

- 단기 recall 상태와 최근 일일 메모리 파일에서 읽습니다.
- 저장소에 인라인 출력이 포함된 경우 관리되는 `## Light Sleep` 블록을 기록합니다.
- 이후 Deep 순위 산정에 사용할 강화 신호를 기록합니다.
- `MEMORY.md`에는 절대 기록하지 않습니다.

### Deep 단계

Deep 단계는 무엇이 장기 메모리가 될지를 결정합니다.

- 가중치 점수와 임곗값 게이트를 사용해 후보 순위를 매깁니다.
- 통과하려면 `minScore`, `minRecallCount`, `minUniqueQueries`가 필요합니다.
- 기록 전에 라이브 일일 파일에서 스니펫을 다시 복원하므로, 오래되었거나 삭제된 스니펫은 건너뜁니다.
- 승격된 항목을 `MEMORY.md`에 추가합니다.
- `DREAMS.md`에 `## Deep Sleep` 요약을 기록하고, 선택적으로 `memory/dreaming/deep/YYYY-MM-DD.md`를 기록합니다.

### REM 단계

REM 단계는 패턴과 성찰 신호를 추출합니다.

- 최근 단기 추적에서 주제 및 성찰 요약을 생성합니다.
- 저장소에 인라인 출력이 포함된 경우 관리되는 `## REM Sleep` 블록을 기록합니다.
- Deep 순위 산정에 사용되는 REM 강화 신호를 기록합니다.
- `MEMORY.md`에는 절대 기록하지 않습니다.

## Dream Diary

Dreaming은 또한 `DREAMS.md`에 서사형 **Dream Diary**를 유지합니다.
각 단계에 충분한 자료가 쌓이면, `memory-core`는 최선의 노력 방식의 백그라운드
subagent 턴(기본 런타임 모델 사용)을 실행하고 짧은 일기 항목을 추가합니다.

이 일기는 Dreams UI에서 사람이 읽기 위한 것이며, 승격 소스는 아닙니다.

## Deep 순위 산정 신호

Deep 순위 산정은 여섯 개의 가중치 기반 기본 신호와 단계 강화 신호를 사용합니다.

| 신호 | 가중치 | 설명 |
| ---- | ------ | ---- |
| 빈도 | 0.24   | 항목이 누적한 단기 신호 수 |
| 관련성 | 0.30   | 항목의 평균 검색 품질 |
| 쿼리 다양성 | 0.15   | 해당 항목이 드러난 개별 쿼리/일별 컨텍스트 |
| 최신성 | 0.15   | 시간 감쇠 기반 신선도 점수 |
| 통합도 | 0.10   | 여러 날에 걸친 반복 강도 |
| 개념적 풍부함 | 0.06   | 스니펫/경로의 개념 태그 밀도 |

Light 및 REM 단계 적중은
`memory/.dreams/phase-signals.json`에서 작은 최신성 감쇠 기반 부스트를 추가합니다.

## 스케줄링

활성화되면 `memory-core`는 전체 dreaming 스윕을 위한 하나의 cron 작업을 자동 관리합니다.
각 스윕은 순서대로 단계를 실행합니다: light -> REM -> deep.

기본 주기 동작:

| 설정 | 기본값 |
| ---- | ------ |
| `dreaming.frequency` | `0 3 * * *` |

## 빠른 시작

dreaming 활성화:

```json
{
  "plugins": {
    "entries": {
      "memory-core": {
        "config": {
          "dreaming": {
            "enabled": true
          }
        }
      }
    }
  }
}
```

사용자 지정 스윕 주기로 dreaming 활성화:

```json
{
  "plugins": {
    "entries": {
      "memory-core": {
        "config": {
          "dreaming": {
            "enabled": true,
            "timezone": "America/Los_Angeles",
            "frequency": "0 */6 * * *"
          }
        }
      }
    }
  }
}
```

## 슬래시 명령어

```
/dreaming status
/dreaming on
/dreaming off
/dreaming help
```

## CLI 워크플로

미리 보기 또는 수동 적용에는 CLI 승격을 사용합니다.

```bash
openclaw memory promote
openclaw memory promote --apply
openclaw memory promote --limit 5
openclaw memory status --deep
```

수동 `memory promote`는 CLI 플래그로 재정의하지 않는 한 기본적으로 Deep 단계 임곗값을 사용합니다.

## 주요 기본값

모든 설정은 `plugins.entries.memory-core.config.dreaming` 아래에 있습니다.

| 키 | 기본값 |
| --- | ------ |
| `enabled`   | `false`     |
| `frequency` | `0 3 * * *` |

단계 정책, 임곗값, 저장소 동작은 내부 구현 세부 사항입니다
(사용자 대상 구성 아님).

전체 키 목록은 [메모리 구성 참조](/ko/reference/memory-config#dreaming-experimental)를 참조하세요.

## Dreams UI

활성화되면 Gateway의 **Dreams** 탭에 다음이 표시됩니다.

- 현재 dreaming 활성화 상태
- 단계별 상태 및 관리형 스윕 존재 여부
- 단기, 장기, 오늘 승격된 항목 수
- 다음 예약 실행 시점
- `doctor.memory.dreamDiary`를 기반으로 하는 확장 가능한 Dream Diary 리더

## 관련 문서

- [메모리](/ko/concepts/memory)
- [메모리 검색](/ko/concepts/memory-search)
- [memory CLI](/cli/memory)
- [메모리 구성 참조](/ko/reference/memory-config)
