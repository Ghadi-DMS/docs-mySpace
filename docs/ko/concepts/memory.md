---
read_when:
    - 메모리가 어떻게 작동하는지 이해하고 싶을 때
    - 어떤 메모리 파일에 작성해야 하는지 알고 싶을 때
summary: OpenClaw가 세션 전반에 걸쳐 정보를 기억하는 방법
title: 메모리 개요
x-i18n:
    generated_at: "2026-04-09T01:27:57Z"
    model: gpt-5.4
    provider: openai
    source_hash: 2fe47910f5bf1c44be379e971c605f1cb3a29befcf2a7ee11fb3833cbe3b9059
    source_path: concepts/memory.md
    workflow: 15
---

# 메모리 개요

OpenClaw는 에이전트의 워크스페이스에 **일반 Markdown 파일**을 작성하여 정보를 기억합니다. 모델이 "기억하는" 것은 디스크에 저장된 내용뿐이며, 숨겨진 상태는 없습니다.

## 작동 방식

에이전트에는 메모리와 관련된 파일이 세 가지 있습니다.

- **`MEMORY.md`** -- 장기 메모리입니다. 지속되는 사실, 선호 사항, 결정 사항을 담습니다. 모든 DM 세션이 시작될 때 로드됩니다.
- **`memory/YYYY-MM-DD.md`** -- 일일 노트입니다. 진행 중인 컨텍스트와 관찰 내용을 담습니다. 오늘과 어제의 노트는 자동으로 로드됩니다.
- **`DREAMS.md`** (실험적, 선택 사항) -- 사람의 검토를 위한 Dream Diary 및 dreaming sweep 요약을 담으며, 근거 기반의 과거 백필 항목도 포함합니다.

이 파일들은 에이전트 워크스페이스에 있습니다(기본값 `~/.openclaw/workspace`).

<Tip>
에이전트가 무언가를 기억하길 원한다면, 그냥 이렇게 요청하면 됩니다: "내가 TypeScript를 선호한다고 기억해." 그러면 적절한 파일에 기록합니다.
</Tip>

## 메모리 도구

에이전트에는 메모리를 다루기 위한 두 가지 도구가 있습니다.

- **`memory_search`** -- 원문과 표현이 달라도 시맨틱 검색을 사용해 관련 노트를 찾습니다.
- **`memory_get`** -- 특정 메모리 파일 또는 줄 범위를 읽습니다.

두 도구 모두 활성 메모리 플러그인(기본값: `memory-core`)에서 제공합니다.

## Memory Wiki 컴패니언 플러그인

지속 메모리가 단순한 원시 노트가 아니라 관리되는 지식 베이스처럼 동작하길 원한다면, 번들된 `memory-wiki` 플러그인을 사용하세요.

`memory-wiki`는 지속 지식을 위키 볼트로 컴파일하며, 다음을 제공합니다.

- 결정론적인 페이지 구조
- 구조화된 주장과 증거
- 모순 및 최신성 추적
- 생성된 대시보드
- 에이전트/런타임 소비자를 위한 컴파일된 다이제스트
- `wiki_search`, `wiki_get`, `wiki_apply`, `wiki_lint` 같은 위키 네이티브 도구

이 플러그인은 활성 메모리 플러그인을 대체하지 않습니다. 활성 메모리 플러그인은 여전히 회상, 승격, dreaming을 담당합니다. `memory-wiki`는 그 옆에 출처가 풍부한 지식 계층을 추가합니다.

자세한 내용은 [Memory Wiki](/ko/plugins/memory-wiki)를 참조하세요.

## 메모리 검색

임베딩 제공자가 구성되어 있으면 `memory_search`는 **하이브리드 검색**을 사용합니다. 즉, 벡터 유사도(의미적 의미)와 키워드 매칭(ID나 코드 심볼 같은 정확한 용어)을 결합합니다. 지원되는 제공자 중 하나의 API 키만 있으면 바로 사용할 수 있습니다.

<Info>
OpenClaw는 사용 가능한 API 키를 바탕으로 임베딩 제공자를 자동 감지합니다. OpenAI, Gemini, Voyage 또는 Mistral 키가 구성되어 있으면 메모리 검색이 자동으로 활성화됩니다.
</Info>

검색 작동 방식, 튜닝 옵션, 제공자 설정에 대한 자세한 내용은 [Memory Search](/ko/concepts/memory-search)를 참조하세요.

## 메모리 백엔드

<CardGroup cols={3}>
<Card title="내장형 (기본값)" icon="database" href="/ko/concepts/memory-builtin">
SQLite 기반입니다. 키워드 검색, 벡터 유사도, 하이브리드 검색을 즉시 사용할 수 있습니다. 추가 의존성이 필요 없습니다.
</Card>
<Card title="QMD" icon="search" href="/ko/concepts/memory-qmd">
재순위 지정, 쿼리 확장, 워크스페이스 외부 디렉터리 인덱싱 기능을 갖춘 로컬 우선 사이드카입니다.
</Card>
<Card title="Honcho" icon="brain" href="/ko/concepts/memory-honcho">
사용자 모델링, 시맨틱 검색, 멀티 에이전트 인식을 갖춘 AI 네이티브 크로스세션 메모리입니다. 플러그인을 설치해야 합니다.
</Card>
</CardGroup>

## 지식 위키 계층

<CardGroup cols={1}>
<Card title="Memory Wiki" icon="book" href="/ko/plugins/memory-wiki">
지속 메모리를 주장, 대시보드, 브리지 모드, Obsidian 친화적 워크플로를 갖춘 출처 풍부한 위키 볼트로 컴파일합니다.
</Card>
</CardGroup>

## 자동 메모리 플러시

[compaction](/ko/concepts/compaction)이 대화를 요약하기 전에, OpenClaw는 에이전트에게 중요한 컨텍스트를 메모리 파일에 저장하라고 상기시키는 조용한 턴을 실행합니다. 이 기능은 기본적으로 켜져 있으므로 별도로 설정할 필요가 없습니다.

<Tip>
메모리 플러시는 compaction 중 컨텍스트 손실을 방지합니다. 대화에 중요한 사실이 아직 파일에 기록되지 않았다면, 요약이 이루어지기 전에 자동으로 저장됩니다.
</Tip>

## Dreaming (실험적)

Dreaming은 메모리를 위한 선택적 백그라운드 통합 패스입니다. 단기 신호를 수집하고, 후보를 점수화하며, 자격을 충족한 항목만 장기 메모리(`MEMORY.md`)로 승격합니다.

이 기능은 장기 메모리의 신호 대 잡음비를 높게 유지하도록 설계되었습니다.

- **Opt-in**: 기본적으로 비활성화되어 있습니다.
- **예약 실행**: 활성화되면 `memory-core`가 전체 dreaming sweep을 위한 반복 cron 작업 하나를 자동으로 관리합니다.
- **임곗값 적용**: 승격은 점수, 회상 빈도, 쿼리 다양성 게이트를 통과해야 합니다.
- **검토 가능**: 단계 요약과 다이어리 항목이 사람이 검토할 수 있도록 `DREAMS.md`에 기록됩니다.

단계별 동작, 점수 신호, Dream Diary 세부 정보는 [Dreaming (experimental)](/ko/concepts/dreaming)을 참조하세요.

## 근거 기반 백필과 라이브 승격

dreaming 시스템에는 이제 밀접하게 연관된 두 가지 검토 경로가 있습니다.

- **라이브 dreaming**은 `memory/.dreams/` 아래의 단기 dreaming 저장소를 기반으로 작동하며, 일반적인 딥 단계가 무엇을 `MEMORY.md`로 승격할 수 있는지 결정할 때 사용하는 방식입니다.
- **근거 기반 백필**은 과거의 `memory/YYYY-MM-DD.md` 노트를 독립적인 일일 파일로 읽고, 구조화된 검토 출력을 `DREAMS.md`에 기록합니다.

근거 기반 백필은 `MEMORY.md`를 수동으로 편집하지 않고도 오래된 노트를 다시 처리하고 시스템이 무엇을 지속적인 정보로 판단하는지 확인하고 싶을 때 유용합니다.

다음 명령을 사용하면:

```bash
openclaw memory rem-backfill --path ./memory --stage-short-term
```

근거 기반의 지속 후보는 직접 승격되지 않습니다. 대신 일반적인 딥 단계가 이미 사용하는 것과 동일한 단기 dreaming 저장소에 스테이징됩니다. 즉, 다음을 의미합니다.

- `DREAMS.md`는 계속 사람 검토용 표면으로 남습니다.
- 단기 저장소는 계속 머신 대상 순위화 표면으로 남습니다.
- `MEMORY.md`는 여전히 딥 승격에 의해서만 기록됩니다.

재처리가 유용하지 않았다고 판단되면, 일반 다이어리 항목이나 정상적인 회상 상태를 건드리지 않고 스테이징된 아티팩트를 제거할 수 있습니다.

```bash
openclaw memory rem-backfill --rollback
openclaw memory rem-backfill --rollback-short-term
```

## CLI

```bash
openclaw memory status          # 인덱스 상태와 제공자 확인
openclaw memory search "query"  # 명령줄에서 검색
openclaw memory index --force   # 인덱스 다시 빌드
```

## 추가 읽을거리

- [Builtin Memory Engine](/ko/concepts/memory-builtin) -- 기본 SQLite 백엔드
- [QMD Memory Engine](/ko/concepts/memory-qmd) -- 고급 로컬 우선 사이드카
- [Honcho Memory](/ko/concepts/memory-honcho) -- AI 네이티브 크로스세션 메모리
- [Memory Wiki](/ko/plugins/memory-wiki) -- 컴파일된 지식 볼트 및 위키 네이티브 도구
- [Memory Search](/ko/concepts/memory-search) -- 검색 파이프라인, 제공자 및 튜닝
- [Dreaming (experimental)](/ko/concepts/dreaming) -- 단기 회상에서 장기 메모리로의 백그라운드 승격
- [Memory configuration reference](/ko/reference/memory-config) -- 모든 구성 옵션
- [Compaction](/ko/concepts/compaction) -- compaction이 메모리와 상호작용하는 방식
