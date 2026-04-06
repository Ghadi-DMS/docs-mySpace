---
read_when:
    - 메모리가 어떻게 작동하는지 이해하고 싶습니다
    - 어떤 메모리 파일을 작성해야 하는지 알고 싶습니다
summary: OpenClaw가 세션 전반에 걸쳐 항목을 기억하는 방법
title: 메모리 개요
x-i18n:
    generated_at: "2026-04-06T03:06:41Z"
    model: gpt-5.4
    provider: openai
    source_hash: d19d4fa9c4b3232b7a97f7a382311d2a375b562040de15e9fe4a0b1990b825e7
    source_path: concepts/memory.md
    workflow: 15
---

# 메모리 개요

OpenClaw는 에이전트의
워크스페이스에 **일반 Markdown 파일**을 작성하여 항목을 기억합니다. 모델은 디스크에 저장된 것만
"기억"합니다 -- 숨겨진 상태는 없습니다.

## 작동 방식

에이전트에는 메모리 관련 파일이 세 개 있습니다:

- **`MEMORY.md`** -- 장기 메모리. 지속되는 사실, 선호도, 그리고
  결정 사항입니다. 모든 DM 세션 시작 시 로드됩니다.
- **`memory/YYYY-MM-DD.md`** -- 일일 노트. 누적되는 컨텍스트와 관찰 내용입니다.
  오늘과 어제의 노트가 자동으로 로드됩니다.
- **`DREAMS.md`** (실험적, 선택 사항) -- 사람이 검토할 수 있는 Dream Diary 및 dreaming 스윕
  요약입니다.

이 파일들은 에이전트 워크스페이스(기본값 `~/.openclaw/workspace`)에 있습니다.

<Tip>
에이전트가 무언가를 기억하길 원한다면 이렇게 요청하면 됩니다: "Remember that I
prefer TypeScript." 그러면 적절한 파일에 기록합니다.
</Tip>

## 메모리 도구

에이전트에는 메모리를 다루기 위한 두 가지 도구가 있습니다:

- **`memory_search`** -- 원래 표현과 문구가 달라도
  시맨틱 검색을 사용해 관련 노트를 찾습니다.
- **`memory_get`** -- 특정 메모리 파일이나 줄 범위를 읽습니다.

두 도구 모두 활성 메모리 plugin이 제공합니다(기본값: `memory-core`).

## 메모리 검색

임베딩 provider가 구성되어 있으면 `memory_search`는 **하이브리드
검색**을 사용합니다 -- 벡터 유사도(의미적 의미)와 키워드 일치
(ID 및 코드 심볼 같은 정확한 용어)를 결합합니다. 지원되는 provider의 API 키만 있으면
즉시 사용할 수 있습니다.

<Info>
OpenClaw는 사용 가능한 API 키를 바탕으로 임베딩 provider를 자동 감지합니다. OpenAI, Gemini, Voyage 또는 Mistral 키가
구성되어 있으면 메모리 검색이 자동으로 활성화됩니다.
</Info>

검색 동작 방식, 조정 옵션, provider 설정에 대한 자세한 내용은
[메모리 검색](/ko/concepts/memory-search)을 참조하세요.

## 메모리 백엔드

<CardGroup cols={3}>
<Card title="Builtin (기본값)" icon="database" href="/ko/concepts/memory-builtin">
SQLite 기반입니다. 키워드 검색, 벡터 유사도, 하이브리드 검색을 즉시 사용할 수 있습니다.
추가 종속성이 없습니다.
</Card>
<Card title="QMD" icon="search" href="/ko/concepts/memory-qmd">
리랭킹, 쿼리 확장, 워크스페이스 외부 디렉터리 인덱싱 기능을 제공하는 로컬 우선 사이드카입니다.
</Card>
<Card title="Honcho" icon="brain" href="/ko/concepts/memory-honcho">
사용자 모델링, 시맨틱 검색, 멀티 에이전트 인식을 갖춘 AI 네이티브 크로스 세션 메모리입니다.
Plugin 설치가 필요합니다.
</Card>
</CardGroup>

## 자동 메모리 플러시

[compaction](/ko/concepts/compaction)이 대화를 요약하기 전에 OpenClaw는
에이전트에게 중요한 컨텍스트를 메모리 파일에 저장하라고 상기시키는
조용한 턴을 실행합니다. 이것은 기본적으로 켜져 있으므로 아무것도 구성할 필요가 없습니다.

<Tip>
메모리 플러시는 compaction 중 컨텍스트 손실을 방지합니다. 대화에 중요한 사실이 있지만 아직 파일에
기록되지 않았다면, 요약이 일어나기 전에 자동으로 저장됩니다.
</Tip>

## dreaming (실험적)

dreaming은 메모리를 위한 선택적 백그라운드 통합 패스입니다. 단기 신호를 수집하고,
후보 항목의 점수를 매기며, 자격을 갖춘 항목만 장기 메모리(`MEMORY.md`)로 승격합니다.

장기 메모리의 신호 품질을 높게 유지하도록 설계되었습니다:

- **옵트인**: 기본적으로 비활성화되어 있습니다.
- **예약 실행**: 활성화되면 `memory-core`가 전체 dreaming 스윕을 위한
  반복 cron 작업 하나를 자동으로 관리합니다.
- **임곗값 적용**: 승격은 점수, 회상 빈도, 쿼리
  다양성 게이트를 통과해야 합니다.
- **검토 가능**: 단계 요약과 다이어리 항목이 `DREAMS.md`에 기록되어
  사람이 검토할 수 있습니다.

단계 동작, 점수화 신호, Dream Diary 세부 사항은
[Dreaming (experimental)](/concepts/dreaming)을 참조하세요.

## CLI

```bash
openclaw memory status          # 인덱스 상태와 provider 확인
openclaw memory search "query"  # 명령줄에서 검색
openclaw memory index --force   # 인덱스 재구축
```

## 추가 읽을거리

- [Builtin Memory Engine](/ko/concepts/memory-builtin) -- 기본 SQLite 백엔드
- [QMD Memory Engine](/ko/concepts/memory-qmd) -- 고급 로컬 우선 사이드카
- [Honcho Memory](/ko/concepts/memory-honcho) -- AI 네이티브 크로스 세션 메모리
- [Memory Search](/ko/concepts/memory-search) -- 검색 파이프라인, provider, 그리고
  조정
- [Dreaming (experimental)](/concepts/dreaming) -- 단기 회상에서 장기 메모리로의
  백그라운드 승격
- [Memory configuration reference](/ko/reference/memory-config) -- 모든 구성 설정
- [Compaction](/ko/concepts/compaction) -- compaction이 메모리와 상호작용하는 방식
