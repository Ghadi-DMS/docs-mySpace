---
read_when:
    - OpenClaw 업데이트 중
    - 업데이트 후 문제가 발생함
summary: OpenClaw를 안전하게 업데이트하는 방법(전역 설치 또는 소스), 그리고 롤백 전략
title: 업데이트
x-i18n:
    generated_at: "2026-04-06T03:08:24Z"
    model: gpt-5.4
    provider: openai
    source_hash: ca9fff0776b9f5977988b649e58a5d169e5fa3539261cb02779d724d4ca92877
    source_path: install/updating.md
    workflow: 15
---

# 업데이트

OpenClaw를 최신 상태로 유지하세요.

## 권장: `openclaw update`

가장 빠르게 업데이트하는 방법입니다. 설치 유형(npm 또는 git)을 감지하고, 최신 버전을 가져오며, `openclaw doctor`를 실행하고, gateway를 다시 시작합니다.

```bash
openclaw update
```

채널을 전환하거나 특정 버전을 대상으로 지정하려면 다음을 사용하세요.

```bash
openclaw update --channel beta
openclaw update --tag main
openclaw update --dry-run   # 적용 없이 미리 보기
```

`--channel beta`는 beta를 우선하지만, beta 태그가 없거나 최신 안정 릴리스보다 오래된 경우 런타임은 stable/latest로 대체됩니다. 일회성 패키지 업데이트에서 원래 npm beta dist-tag를 사용하려면 `--tag beta`를 사용하세요.

채널 의미는 [Development channels](/ko/install/development-channels)를 참고하세요.

## 대안: 설치 프로그램 다시 실행

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

온보딩을 건너뛰려면 `--no-onboard`를 추가하세요. 소스 설치의 경우 `--install-method git --no-onboard`를 전달하세요.

## 대안: 수동 npm, pnpm 또는 bun

```bash
npm i -g openclaw@latest
```

```bash
pnpm add -g openclaw@latest
```

```bash
bun add -g openclaw@latest
```

## 자동 업데이터

자동 업데이터는 기본적으로 꺼져 있습니다. `~/.openclaw/openclaw.json`에서 활성화하세요.

```json5
{
  update: {
    channel: "stable",
    auto: {
      enabled: true,
      stableDelayHours: 6,
      stableJitterHours: 12,
      betaCheckIntervalHours: 1,
    },
  },
}
```

| 채널 | 동작 |
| -------- | ------------------------------------------------------------------------------------------------------------- |
| `stable` | `stableDelayHours`만큼 기다린 뒤, `stableJitterHours` 전체에 걸쳐 결정론적 지터를 적용해 배포를 분산하며 적용합니다. |
| `beta`   | `betaCheckIntervalHours`마다(기본값: 1시간) 확인하고 즉시 적용합니다. |
| `dev`    | 자동 적용이 없습니다. 수동으로 `openclaw update`를 사용하세요. |

gateway는 시작 시 업데이트 힌트도 기록합니다(`update.checkOnStart: false`로 비활성화).

## 업데이트 후

<Steps>

### doctor 실행

```bash
openclaw doctor
```

config를 마이그레이션하고, DM 정책을 감사하며, gateway 상태를 확인합니다. 자세한 내용: [Doctor](/ko/gateway/doctor)

### gateway 다시 시작

```bash
openclaw gateway restart
```

### 확인

```bash
openclaw health
```

</Steps>

## 롤백

### 버전 고정(npm)

```bash
npm i -g openclaw@<version>
openclaw doctor
openclaw gateway restart
```

팁: `npm view openclaw version`은 현재 게시된 버전을 표시합니다.

### 커밋 고정(소스)

```bash
git fetch origin
git checkout "$(git rev-list -n 1 --before=\"2026-01-01\" origin/main)"
pnpm install && pnpm build
openclaw gateway restart
```

최신 상태로 돌아가려면: `git checkout main && git pull`.

## 문제가 해결되지 않을 때

- `openclaw doctor`를 다시 실행하고 출력을 주의 깊게 읽으세요.
- 소스 체크아웃에서 `openclaw update --channel dev`를 실행할 때, 업데이터는 필요하면 자동으로 `pnpm`을 부트스트랩합니다. `pnpm`/corepack 부트스트랩 오류가 보이면 `pnpm`을 수동으로 설치하거나(`corepack`을 다시 활성화한 뒤) 업데이트를 다시 실행하세요.
- 확인: [Troubleshooting](/ko/gateway/troubleshooting)
- Discord에서 문의: [https://discord.gg/clawd](https://discord.gg/clawd)

## 관련 문서

- [Install Overview](/ko/install) — 모든 설치 방법
- [Doctor](/ko/gateway/doctor) — 업데이트 후 상태 확인
- [Migrating](/ko/install/migrating) — 주요 버전 마이그레이션 가이드
