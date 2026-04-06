---
read_when:
    - 메모리 백엔드로 QMD를 설정하려고 합니다
    - 리랭킹이나 추가 인덱싱 경로 같은 고급 메모리 기능이 필요합니다
summary: BM25, 벡터, 리랭킹, 쿼리 확장을 갖춘 로컬 우선 검색 사이드카
title: QMD 메모리 엔진
x-i18n:
    generated_at: "2026-04-06T03:06:18Z"
    model: gpt-5.4
    provider: openai
    source_hash: 36642c7df94b88f562745dd2270334379f2aeeef4b363a8c13ef6be42dadbe5c
    source_path: concepts/memory-qmd.md
    workflow: 15
---

# QMD 메모리 엔진

[QMD](https://github.com/tobi/qmd)는 OpenClaw와 함께 실행되는 로컬 우선 검색 사이드카입니다. 단일
바이너리에서 BM25, 벡터 검색, 리랭킹을 결합하며, 워크스페이스 메모리 파일을
넘어서는 콘텐츠도 인덱싱할 수 있습니다.

## builtin 대비 추가되는 점

- **리랭킹 및 쿼리 확장**으로 더 나은 재현율을 제공합니다.
- **추가 디렉터리 인덱싱** -- 프로젝트 문서, 팀 노트, 디스크 상의 모든 항목.
- **세션 transcript 인덱싱** -- 이전 대화를 회상할 수 있습니다.
- **완전한 로컬 실행** -- Bun + node-llama-cpp를 통해 실행되며, GGUF 모델을 자동으로 다운로드합니다.
- **자동 폴백** -- QMD를 사용할 수 없으면 OpenClaw가
  builtin 엔진으로 원활하게 폴백합니다.

## 시작하기

### 사전 요구 사항

- QMD 설치: `npm install -g @tobilu/qmd` 또는 `bun install -g @tobilu/qmd`
- 확장을 허용하는 SQLite 빌드(macOS에서는 `brew install sqlite`).
- 게이트웨이의 `PATH`에 QMD가 있어야 합니다.
- macOS와 Linux는 바로 사용할 수 있습니다. Windows는 WSL2를 통해 가장 잘 지원됩니다.

### 활성화

```json5
{
  memory: {
    backend: "qmd",
  },
}
```

OpenClaw는 `~/.openclaw/agents/<agentId>/qmd/` 아래에 독립형 QMD 홈을 생성하고
사이드카 수명 주기를 자동으로 관리합니다 -- 컬렉션, 업데이트, 임베딩 실행이 자동으로 처리됩니다.
현재 QMD 컬렉션 및 MCP 쿼리 형태를 우선 사용하지만, 필요할 때는
레거시 `--mask` 컬렉션 플래그와 이전 MCP 도구 이름으로도 폴백합니다.

## 사이드카 동작 방식

- OpenClaw는 워크스페이스 메모리 파일과 구성된
  `memory.qmd.paths`로부터 컬렉션을 생성한 다음, 부팅 시와 주기적으로
  (기본값: 5분마다) `qmd update` + `qmd embed`를 실행합니다.
- 부팅 새로 고침은 채팅 시작이 차단되지 않도록 백그라운드에서 실행됩니다.
- 검색은 구성된 `searchMode`를 사용합니다(기본값: `search`; `vsearch`와 `query`도 지원).
  모드가 실패하면 OpenClaw는 `qmd query`로 재시도합니다.
- QMD가 완전히 실패하면 OpenClaw는 builtin SQLite 엔진으로 폴백합니다.

<Info>
첫 번째 검색은 느릴 수 있습니다 -- 첫 `qmd query` 실행 시 리랭킹과
쿼리 확장을 위해 QMD가 GGUF 모델(~2 GB)을 자동으로 다운로드합니다.
</Info>

## 모델 재정의

QMD 모델 환경 변수는 게이트웨이 프로세스에서
변경 없이 그대로 전달되므로, 새로운 OpenClaw 구성을 추가하지 않고도 전역적으로 QMD를 조정할 수 있습니다.

```bash
export QMD_EMBED_MODEL="hf:Qwen/Qwen3-Embedding-0.6B-GGUF/Qwen3-Embedding-0.6B-Q8_0.gguf"
export QMD_RERANK_MODEL="/absolute/path/to/reranker.gguf"
export QMD_GENERATE_MODEL="/absolute/path/to/generator.gguf"
```

임베딩 모델을 변경한 후에는 인덱스가 새 벡터 공간과 일치하도록
임베딩을 다시 실행하세요.

## 추가 경로 인덱싱

추가 디렉터리를 검색 가능하게 하려면 QMD가 해당 디렉터리를 가리키도록 설정하세요:

```json5
{
  memory: {
    backend: "qmd",
    qmd: {
      paths: [{ name: "docs", path: "~/notes", pattern: "**/*.md" }],
    },
  },
}
```

추가 경로의 스니펫은 검색 결과에서 `qmd/<collection>/<relative-path>`로 표시됩니다.
`memory_get`은 이 접두사를 이해하고 올바른 컬렉션 루트에서 읽습니다.

## 세션 transcript 인덱싱

이전 대화를 회상할 수 있도록 세션 인덱싱을 활성화합니다:

```json5
{
  memory: {
    backend: "qmd",
    qmd: {
      sessions: { enabled: true },
    },
  },
}
```

transcript는 정제된 User/Assistant 턴으로 내보내져
`~/.openclaw/agents/<id>/qmd/sessions/` 아래의 전용 QMD 컬렉션에 저장됩니다.

## 검색 범위

기본적으로 QMD 검색 결과는 DM 세션에서만 표시되며(그룹이나
채널은 아님), `memory.qmd.scope`를 구성해 이를 변경할 수 있습니다:

```json5
{
  memory: {
    qmd: {
      scope: {
        default: "deny",
        rules: [{ action: "allow", match: { chatType: "direct" } }],
      },
    },
  },
}
```

범위가 검색을 거부하면 OpenClaw는 파생된 채널과
채팅 유형을 포함한 경고를 기록하므로 빈 결과를 더 쉽게 디버그할 수 있습니다.

## 인용

`memory.citations`가 `auto` 또는 `on`이면 검색 스니펫에
`Source: <path#line>` 바닥글이 포함됩니다. 바닥글은 생략하면서도 경로를 내부적으로 에이전트에 계속 전달하려면
`memory.citations = "off"`로 설정하세요.

## 사용해야 하는 경우

다음이 필요하다면 QMD를 선택하세요:

- 더 높은 품질의 결과를 위한 리랭킹.
- 워크스페이스 외부의 프로젝트 문서나 노트 검색.
- 과거 세션 대화 회상.
- API 키가 필요 없는 완전한 로컬 검색.

더 단순한 설정에서는 추가 종속성 없이도
[builtin 엔진](/ko/concepts/memory-builtin)이 잘 작동합니다.

## 문제 해결

**QMD를 찾을 수 없나요?** 바이너리가 게이트웨이의 `PATH`에 있는지 확인하세요. OpenClaw가
서비스로 실행된다면 다음과 같이 심볼릭 링크를 만드세요:
`sudo ln -s ~/.bun/bin/qmd /usr/local/bin/qmd`.

**첫 검색이 매우 느린가요?** QMD는 첫 사용 시 GGUF 모델을 다운로드합니다. OpenClaw가 사용하는 동일한 XDG 디렉터리로
`qmd query "test"`를 사용해 미리 준비하세요.

**검색 시간이 초과되나요?** `memory.qmd.limits.timeoutMs`(기본값: 4000ms)를 늘리세요.
더 느린 하드웨어에서는 `120000`으로 설정하세요.

**그룹 채팅에서 결과가 비어 있나요?** `memory.qmd.scope`를 확인하세요 -- 기본값은
DM 세션만 허용합니다.

**워크스페이스에 보이는 임시 리포지토리 때문에 `ENAMETOOLONG` 또는 인덱싱 손상이 발생하나요?**
QMD 순회는 현재 OpenClaw의 builtin 심볼릭 링크 규칙이 아니라
기반 QMD 스캐너 동작을 따릅니다. QMD가
순환 안전 순회나 명시적 제외 제어를 제공할 때까지 임시 모노리포 체크아웃은
`.tmp/` 같은 숨김 디렉터리 아래나 인덱싱된 QMD 루트 밖에 두세요.

## 구성

전체 구성 표면(`memory.qmd.*`), 검색 모드, 업데이트 간격,
범위 규칙 및 기타 모든 설정은
[메모리 구성 참조](/ko/reference/memory-config)를 참조하세요.
