---
read_when:
    - OpenClaw에서 Matrix 설정하기
    - Matrix E2EE 및 검증 구성하기
summary: Matrix 지원 상태, 설정, 및 구성 예제
title: Matrix
x-i18n:
    generated_at: "2026-04-09T01:30:07Z"
    model: gpt-5.4
    provider: openai
    source_hash: 28fc13c7620c1152200315ae69c94205da6de3180c53c814dd8ce03b5cb1758f
    source_path: channels/matrix.md
    workflow: 15
---

# Matrix

Matrix는 OpenClaw에 번들로 포함된 채널 plugin입니다.
공식 `matrix-js-sdk`를 사용하며 DM, room, 스레드, 미디어, 반응, 투표, 위치, E2EE를 지원합니다.

## 번들 plugin

Matrix는 현재 OpenClaw 릴리스에 번들 plugin으로 포함되어 제공되므로 일반적인 패키지 빌드에서는 별도 설치가 필요하지 않습니다.

이전 빌드나 Matrix가 제외된 사용자 지정 설치를 사용하는 경우 수동으로 설치하세요:

npm에서 설치:

```bash
openclaw plugins install @openclaw/matrix
```

로컬 체크아웃에서 설치:

```bash
openclaw plugins install ./path/to/local/matrix-plugin
```

plugin 동작과 설치 규칙은 [Plugins](/ko/tools/plugin)를 참조하세요.

## 설정

1. Matrix plugin을 사용할 수 있는지 확인합니다.
   - 현재 패키지된 OpenClaw 릴리스에는 이미 번들로 포함되어 있습니다.
   - 이전/사용자 지정 설치에서는 위 명령으로 수동 추가할 수 있습니다.
2. 홈서버에서 Matrix 계정을 만듭니다.
3. `channels.matrix`를 다음 중 하나로 구성합니다:
   - `homeserver` + `accessToken`, 또는
   - `homeserver` + `userId` + `password`.
4. gateway를 재시작합니다.
5. 봇과 DM을 시작하거나 room에 초대합니다.
   - 새 Matrix 초대는 `channels.matrix.autoJoin`이 허용하는 경우에만 동작합니다.

대화형 설정 경로:

```bash
openclaw channels add
openclaw configure --section channels
```

Matrix 마법사는 다음을 묻습니다:

- homeserver URL
- 인증 방식: access token 또는 password
- 사용자 ID (password 인증만)
- 선택적 device 이름
- E2EE 활성화 여부
- room 접근과 초대 자동 참여를 구성할지 여부

마법사의 주요 동작:

- Matrix 인증 env var가 이미 존재하고 해당 계정에 config에 저장된 인증이 아직 없으면, 마법사는 인증을 env var에 유지하는 env 바로가기를 제안합니다.
- 계정 이름은 계정 ID로 정규화됩니다. 예를 들어 `Ops Bot`은 `ops-bot`이 됩니다.
- DM 허용 목록 항목은 `@user:server`를 직접 받을 수 있습니다. 표시 이름은 라이브 디렉터리 조회에서 정확히 하나의 일치 항목을 찾을 때만 동작합니다.
- Room 허용 목록 항목은 room ID와 alias를 직접 받을 수 있습니다. `!room:server` 또는 `#alias:server`를 권장하며, 해석되지 않은 이름은 허용 목록 해석 시 런타임에서 무시됩니다.
- 초대 자동 참여 허용 목록 모드에서는 안정적인 초대 대상만 사용하세요: `!roomId:server`, `#alias:server`, 또는 `*`. 일반 room 이름은 거부됩니다.
- 저장 전에 room 이름을 해석하려면 `openclaw channels resolve --channel matrix "Project Room"`를 사용하세요.

<Warning>
`channels.matrix.autoJoin`의 기본값은 `off`입니다.

설정하지 않으면 봇은 초대된 room이나 새로운 DM 스타일 초대에 참여하지 않으므로, 먼저 수동으로 참여하지 않는 한 새 그룹이나 초대된 DM에 나타나지 않습니다.

수락할 초대를 제한하려면 `autoJoinAllowlist`와 함께 `autoJoin: "allowlist"`를 설정하거나, 모든 초대에 참여하게 하려면 `autoJoin: "always"`를 설정하세요.

`allowlist` 모드에서 `autoJoinAllowlist`는 `!roomId:server`, `#alias:server`, 또는 `*`만 허용합니다.
</Warning>

허용 목록 예제:

```json5
{
  channels: {
    matrix: {
      autoJoin: "allowlist",
      autoJoinAllowlist: ["!ops:example.org", "#support:example.org"],
      groups: {
        "!ops:example.org": {
          requireMention: true,
        },
      },
    },
  },
}
```

모든 초대에 참여:

```json5
{
  channels: {
    matrix: {
      autoJoin: "always",
    },
  },
}
```

최소 token 기반 설정:

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      dm: { policy: "pairing" },
    },
  },
}
```

password 기반 설정 (로그인 후 token이 캐시됨):

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      userId: "@bot:example.org",
      password: "replace-me", // pragma: allowlist secret
      deviceName: "OpenClaw Gateway",
    },
  },
}
```

Matrix는 캐시된 자격 증명을 `~/.openclaw/credentials/matrix/`에 저장합니다.
기본 계정은 `credentials.json`을 사용하고, 이름이 있는 계정은 `credentials-<account>.json`을 사용합니다.
그 위치에 캐시된 자격 증명이 있으면 현재 인증이 config에 직접 설정되어 있지 않더라도 OpenClaw는 설정, doctor, 채널 상태 검색에서 Matrix를 구성된 것으로 취급합니다.

환경 변수 대응 항목(설정 키가 지정되지 않은 경우 사용됨):

- `MATRIX_HOMESERVER`
- `MATRIX_ACCESS_TOKEN`
- `MATRIX_USER_ID`
- `MATRIX_PASSWORD`
- `MATRIX_DEVICE_ID`
- `MATRIX_DEVICE_NAME`

기본이 아닌 계정의 경우 계정 범위 env var를 사용하세요:

- `MATRIX_<ACCOUNT_ID>_HOMESERVER`
- `MATRIX_<ACCOUNT_ID>_ACCESS_TOKEN`
- `MATRIX_<ACCOUNT_ID>_USER_ID`
- `MATRIX_<ACCOUNT_ID>_PASSWORD`
- `MATRIX_<ACCOUNT_ID>_DEVICE_ID`
- `MATRIX_<ACCOUNT_ID>_DEVICE_NAME`

계정 `ops`의 예:

- `MATRIX_OPS_HOMESERVER`
- `MATRIX_OPS_ACCESS_TOKEN`

정규화된 계정 ID `ops-bot`의 경우 다음을 사용합니다:

- `MATRIX_OPS_X2D_BOT_HOMESERVER`
- `MATRIX_OPS_X2D_BOT_ACCESS_TOKEN`

Matrix는 계정 ID의 구두점을 이스케이프하여 범위 지정된 env var 간 충돌이 없도록 합니다.
예를 들어 `-`는 `_X2D_`가 되므로 `ops-prod`는 `MATRIX_OPS_X2D_PROD_*`에 매핑됩니다.

대화형 마법사는 해당 인증 env var가 이미 존재하고 선택한 계정에 Matrix 인증이 config에 아직 저장되어 있지 않은 경우에만 env-var 바로가기를 제안합니다.

## 구성 예제

다음은 DM 페어링, room 허용 목록, E2EE 활성화를 포함한 실용적인 기준 구성입니다:

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      encryption: true,

      dm: {
        policy: "pairing",
        sessionScope: "per-room",
        threadReplies: "off",
      },

      groupPolicy: "allowlist",
      groupAllowFrom: ["@admin:example.org"],
      groups: {
        "!roomid:example.org": {
          requireMention: true,
        },
      },

      autoJoin: "allowlist",
      autoJoinAllowlist: ["!roomid:example.org"],
      threadReplies: "inbound",
      replyToMode: "off",
      streaming: "partial",
    },
  },
}
```

`autoJoin`은 DM 스타일 초대를 포함한 모든 Matrix 초대에 적용됩니다. OpenClaw는 초대 시점에 초대된 room이 DM인지 그룹인지 신뢰성 있게 분류할 수 없으므로, 모든 초대는 먼저 `autoJoin`을 거칩니다. `dm.policy`는 봇이 참여한 뒤 room이 DM으로 분류된 후에 적용됩니다.

## 스트리밍 미리보기

Matrix 답변 스트리밍은 옵트인입니다.

OpenClaw가 하나의 라이브 미리보기 답변을 보내고, 모델이 텍스트를 생성하는 동안 그 미리보기를 제자리에서 편집한 다음, 답변이 완료되면 최종 확정하도록 하려면 `channels.matrix.streaming`을 `"partial"`로 설정하세요:

```json5
{
  channels: {
    matrix: {
      streaming: "partial",
    },
  },
}
```

- `streaming: "off"`가 기본값입니다. OpenClaw는 최종 답변을 기다렸다가 한 번만 전송합니다.
- `streaming: "partial"`은 현재 assistant 블록에 대해 편집 가능한 미리보기 메시지 하나를 일반 Matrix 텍스트 메시지로 생성합니다. 이는 Matrix의 기존 미리보기 우선 알림 동작을 유지하므로, 기본 클라이언트에서는 완성된 블록 대신 첫 번째 스트리밍 미리보기 텍스트에 대해 알림이 올 수 있습니다.
- `streaming: "quiet"`는 현재 assistant 블록에 대해 편집 가능한 조용한 미리보기 알림 하나를 생성합니다. 최종 확정된 미리보기 편집에 대한 수신자 푸시 규칙도 함께 구성하는 경우에만 이것을 사용하세요.
- `blockStreaming: true`는 별도의 Matrix 진행 상황 메시지를 활성화합니다. 미리보기 스트리밍이 활성화된 상태에서는 Matrix가 현재 블록의 라이브 초안을 유지하고 완료된 블록은 별도 메시지로 보존합니다.
- 미리보기 스트리밍이 켜져 있고 `blockStreaming`이 꺼져 있으면, Matrix는 라이브 초안을 제자리에서 편집하고 블록 또는 턴이 끝나면 같은 이벤트를 최종 확정합니다.
- 미리보기가 더 이상 하나의 Matrix 이벤트에 맞지 않으면 OpenClaw는 미리보기 스트리밍을 중단하고 일반 최종 전달로 대체합니다.
- 미디어 답변은 여전히 첨부 파일을 정상적으로 전송합니다. 오래된 미리보기를 더 이상 안전하게 재사용할 수 없으면 OpenClaw는 최종 미디어 답변을 보내기 전에 그것을 redact합니다.
- 미리보기 편집은 추가 Matrix API 호출 비용이 듭니다. 가장 보수적인 rate limit 동작을 원한다면 스트리밍을 꺼두세요.

`blockStreaming`만으로는 초안 미리보기가 활성화되지 않습니다.
미리보기 편집에는 `streaming: "partial"` 또는 `streaming: "quiet"`를 사용하고, 완료된 assistant 블록도 별도의 진행 상황 메시지로 남기고 싶을 때만 `blockStreaming: true`를 추가하세요.

사용자 지정 푸시 규칙 없이 기본 Matrix 알림이 필요하면 미리보기 우선 동작을 위해 `streaming: "partial"`을 사용하거나, 최종 결과만 전달하려면 `streaming`을 꺼두세요. `streaming: "off"`일 때:

- `blockStreaming: true`는 각 완료된 블록을 일반적인 알림 Matrix 메시지로 보냅니다.
- `blockStreaming: false`는 최종 완료된 답변만 일반적인 알림 Matrix 메시지로 보냅니다.

### 조용한 최종 확정 미리보기를 위한 자체 호스팅 푸시 규칙

자체 Matrix 인프라를 운영하며 블록 또는 최종 답변이 완료되었을 때만 조용한 미리보기에 알림이 가도록 하려면 `streaming: "quiet"`를 설정하고 최종 확정된 미리보기 편집에 대한 사용자별 푸시 규칙을 추가하세요.

이는 보통 homeserver 전역 config 변경이 아니라 수신자 사용자 설정입니다:

시작 전 빠른 개요:

- 수신자 사용자 = 알림을 받아야 하는 사람
- 봇 사용자 = 답변을 보내는 OpenClaw Matrix 계정
- 아래 API 호출에는 수신자 사용자의 access token을 사용
- 푸시 규칙의 `sender`는 봇 사용자의 전체 MXID와 일치시킴

1. OpenClaw가 조용한 미리보기를 사용하도록 구성합니다:

```json5
{
  channels: {
    matrix: {
      streaming: "quiet",
    },
  },
}
```

2. 수신자 계정이 이미 일반 Matrix 푸시 알림을 받고 있는지 확인합니다. 조용한 미리보기 규칙은 해당 사용자에게 이미 작동하는 pusher/device가 있을 때만 동작합니다.

3. 수신자 사용자의 access token을 가져옵니다.
   - 봇의 token이 아니라 수신 사용자 token을 사용하세요.
   - 기존 클라이언트 세션 token을 재사용하는 것이 보통 가장 쉽습니다.
   - 새 token을 발급해야 하면 표준 Matrix Client-Server API를 통해 로그인할 수 있습니다:

```bash
curl -sS -X POST \
  "https://matrix.example.org/_matrix/client/v3/login" \
  -H "Content-Type: application/json" \
  --data '{
    "type": "m.login.password",
    "identifier": {
      "type": "m.id.user",
      "user": "@alice:example.org"
    },
    "password": "REDACTED"
  }'
```

4. 수신자 계정에 이미 pusher가 있는지 확인합니다:

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushers"
```

활성 pusher/device가 하나도 반환되지 않으면, 아래 OpenClaw 규칙을 추가하기 전에 먼저 일반 Matrix 알림부터 수정하세요.

OpenClaw는 최종 확정된 텍스트 전용 미리보기 편집에 다음 표시를 추가합니다:

```json
{
  "com.openclaw.finalized_preview": true
}
```

5. 이러한 알림을 받아야 하는 각 수신자 계정에 대해 override 푸시 규칙을 생성합니다:

```bash
curl -sS -X PUT \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname" \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  --data '{
    "conditions": [
      { "kind": "event_match", "key": "type", "pattern": "m.room.message" },
      {
        "kind": "event_property_is",
        "key": "content.m\\.relates_to.rel_type",
        "value": "m.replace"
      },
      {
        "kind": "event_property_is",
        "key": "content.com\\.openclaw\\.finalized_preview",
        "value": true
      },
      { "kind": "event_match", "key": "sender", "pattern": "@bot:example.org" }
    ],
    "actions": [
      "notify",
      { "set_tweak": "sound", "value": "default" },
      { "set_tweak": "highlight", "value": false }
    ]
  }'
```

명령을 실행하기 전에 다음 값을 바꾸세요:

- `https://matrix.example.org`: homeserver 기본 URL
- `$USER_ACCESS_TOKEN`: 수신 사용자 access token
- `openclaw-finalized-preview-botname`: 이 수신 사용자에 대한 이 봇 전용 고유한 rule ID
- `@bot:example.org`: 수신 사용자 MXID가 아니라 OpenClaw Matrix 봇 MXID

여러 봇을 사용하는 구성에서 중요 사항:

- 푸시 규칙은 `ruleId`를 키로 사용합니다. 같은 rule ID에 대해 `PUT`을 다시 실행하면 해당 규칙 하나가 업데이트됩니다.
- 한 수신 사용자가 여러 OpenClaw Matrix 봇 계정에 대해 알림을 받아야 한다면, 각 sender 일치 항목마다 고유한 rule ID로 봇당 하나의 규칙을 생성하세요.
- 간단한 패턴은 `openclaw-finalized-preview-<botname>`이며, 예를 들어 `openclaw-finalized-preview-ops` 또는 `openclaw-finalized-preview-support`처럼 사용할 수 있습니다.

규칙은 이벤트 sender를 기준으로 평가됩니다:

- 수신 사용자 token으로 인증
- `sender`를 OpenClaw 봇 MXID와 일치시킴

6. 규칙이 존재하는지 확인합니다:

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname"
```

7. 스트리밍 답변을 테스트합니다. 조용한 모드에서는 room에 조용한 초안 미리보기가 표시되어야 하며, 블록이나 턴이 끝나면 제자리 최종 편집에 대해 한 번 알림이 와야 합니다.

나중에 규칙을 제거해야 하면 같은 수신 사용자 token으로 동일한 rule ID를 삭제하세요:

```bash
curl -sS -X DELETE \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname"
```

참고:

- 규칙은 봇 token이 아니라 수신 사용자 access token으로 생성하세요.
- 새 사용자 정의 `override` 규칙은 기본 억제 규칙보다 앞에 삽입되므로 추가 순서 지정 매개변수는 필요하지 않습니다.
- 이는 OpenClaw가 안전하게 제자리 확정할 수 있는 텍스트 전용 미리보기 편집에만 영향을 줍니다. 미디어 대체와 오래된 미리보기 대체는 여전히 일반 Matrix 전달을 사용합니다.
- `GET /_matrix/client/v3/pushers`에 pusher가 없으면, 해당 사용자는 아직 이 계정/device에 대해 작동하는 Matrix 푸시 전달을 갖고 있지 않습니다.

#### Synapse

Synapse에서는 위 설정만으로 보통 충분합니다:

- 최종 확정된 OpenClaw 미리보기 알림을 위해 특별한 `homeserver.yaml` 변경은 필요하지 않습니다.
- Synapse 배포가 이미 일반 Matrix 푸시 알림을 보내고 있다면, 사용자 token + 위 `pushrules` 호출이 주요 설정 단계입니다.
- Synapse를 reverse proxy 또는 worker 뒤에서 실행한다면 `/_matrix/client/.../pushrules/`가 Synapse에 올바르게 도달하는지 확인하세요.
- Synapse worker를 실행 중이라면 pusher가 정상인지 확인하세요. 푸시 전달은 메인 프로세스 또는 `synapse.app.pusher` / 구성된 pusher worker가 처리합니다.

#### Tuwunel

Tuwunel에서는 위에 표시된 동일한 설정 흐름과 push-rule API 호출을 사용하세요:

- 최종 확정 미리보기 마커 자체를 위한 Tuwunel 전용 config는 필요하지 않습니다.
- 해당 사용자에 대해 일반 Matrix 알림이 이미 작동한다면, 사용자 token + 위 `pushrules` 호출이 주요 설정 단계입니다.
- 사용자가 다른 device에서 활성 상태일 때 알림이 사라지는 것 같다면 `suppress_push_when_active`가 활성화되어 있는지 확인하세요. Tuwunel은 2025년 9월 12일 Tuwunel 1.4.2에서 이 옵션을 추가했으며, 하나의 device가 활성일 때 다른 device로의 푸시를 의도적으로 억제할 수 있습니다.

## 봇 간 room

기본적으로 다른 구성된 OpenClaw Matrix 계정에서 온 Matrix 메시지는 무시됩니다.

에이전트 간 Matrix 트래픽을 의도적으로 원할 때는 `allowBots`를 사용하세요:

```json5
{
  channels: {
    matrix: {
      allowBots: "mentions", // true | "mentions"
      groups: {
        "!roomid:example.org": {
          requireMention: true,
        },
      },
    },
  },
}
```

- `allowBots: true`는 허용된 room과 DM에서 다른 구성된 Matrix 봇 계정의 메시지를 수락합니다.
- `allowBots: "mentions"`는 room에서는 이 봇을 눈에 띄게 언급할 때만 해당 메시지를 수락합니다. DM은 여전히 허용됩니다.
- `groups.<room>.allowBots`는 하나의 room에 대해 계정 수준 설정을 재정의합니다.
- OpenClaw는 자기 자신에게 답하는 루프를 피하기 위해 동일한 Matrix 사용자 ID의 메시지는 여전히 무시합니다.
- Matrix는 여기서 기본 bot 플래그를 노출하지 않으므로, OpenClaw는 "봇 작성"을 "이 OpenClaw gateway에 구성된 다른 Matrix 계정이 보낸 것"으로 취급합니다.

공유 room에서 봇 간 트래픽을 활성화할 때는 엄격한 room 허용 목록과 멘션 요구 사항을 사용하세요.

## 암호화 및 검증

암호화된(E2EE) room에서 outbound 이미지 이벤트는 `thumbnail_file`을 사용하므로 이미지 미리보기도 전체 첨부 파일과 함께 암호화됩니다. 암호화되지 않은 room은 여전히 일반 `thumbnail_url`을 사용합니다. 별도 구성은 필요하지 않습니다 — plugin이 E2EE 상태를 자동으로 감지합니다.

암호화 활성화:

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      encryption: true,
      dm: { policy: "pairing" },
    },
  },
}
```

검증 상태 확인:

```bash
openclaw matrix verify status
```

자세한 상태(전체 진단):

```bash
openclaw matrix verify status --verbose
```

저장된 recovery key를 machine-readable 출력에 포함:

```bash
openclaw matrix verify status --include-recovery-key --json
```

교차 서명 및 검증 상태 부트스트랩:

```bash
openclaw matrix verify bootstrap
```

자세한 부트스트랩 진단:

```bash
openclaw matrix verify bootstrap --verbose
```

부트스트랩 전에 새 교차 서명 ID 재설정을 강제:

```bash
openclaw matrix verify bootstrap --force-reset-cross-signing
```

recovery key로 이 device 검증:

```bash
openclaw matrix verify device "<your-recovery-key>"
```

자세한 device 검증 세부 정보:

```bash
openclaw matrix verify device "<your-recovery-key>" --verbose
```

room-key 백업 상태 확인:

```bash
openclaw matrix verify backup status
```

자세한 백업 상태 진단:

```bash
openclaw matrix verify backup status --verbose
```

서버 백업에서 room key 복원:

```bash
openclaw matrix verify backup restore
```

자세한 복원 진단:

```bash
openclaw matrix verify backup restore --verbose
```

현재 서버 백업을 삭제하고 새 백업 기준 상태를 만듭니다. 저장된 백업 키를 정상적으로 불러올 수 없으면 이 재설정으로 secret storage도 다시 생성되어 향후 콜드 스타트에서 새 백업 키를 불러올 수 있게 됩니다:

```bash
openclaw matrix verify backup reset --yes
```

모든 `verify` 명령은 기본적으로 간결하게 동작하며(조용한 내부 SDK 로깅 포함), 자세한 진단은 `--verbose`에서만 표시합니다.
스크립팅할 때는 전체 machine-readable 출력을 위해 `--json`을 사용하세요.

다중 계정 구성에서 Matrix CLI 명령은 `--account <id>`를 전달하지 않으면 암묵적인 Matrix 기본 계정을 사용합니다.
이름이 지정된 여러 계정을 구성한 경우 `channels.matrix.defaultAccount`를 먼저 설정하지 않으면, 이러한 암묵적 CLI 작업은 중단되고 계정을 명시적으로 선택하라고 요청합니다.
검증 또는 device 작업이 특정 이름 있는 계정을 명시적으로 대상으로 하게 하려면 `--account`를 사용하세요:

```bash
openclaw matrix verify status --account assistant
openclaw matrix verify backup restore --account assistant
openclaw matrix devices list --account assistant
```

이름 있는 계정에서 암호화가 비활성화되었거나 사용할 수 없을 때 Matrix 경고와 검증 오류는 예를 들어 `channels.matrix.accounts.assistant.encryption`처럼 해당 계정의 config 키를 가리킵니다.

### "검증됨"의 의미

OpenClaw는 이 Matrix device가 자신의 교차 서명 ID에 의해 검증된 경우에만 검증된 것으로 취급합니다.
실제로 `openclaw matrix verify status --verbose`는 세 가지 신뢰 신호를 노출합니다:

- `Locally trusted`: 현재 클라이언트에서만 이 device를 신뢰함
- `Cross-signing verified`: SDK가 교차 서명을 통해 이 device를 검증된 것으로 보고함
- `Signed by owner`: 이 device가 자신의 self-signing key로 서명됨

`Verified by owner`는 교차 서명 검증 또는 소유자 서명이 있을 때만 `yes`가 됩니다.
로컬 신뢰만으로는 OpenClaw가 이 device를 완전히 검증된 것으로 취급하기에 충분하지 않습니다.

### bootstrap이 수행하는 작업

`openclaw matrix verify bootstrap`은 암호화된 Matrix 계정을 위한 복구 및 설정 명령입니다.
다음 작업을 순서대로 모두 수행합니다:

- 가능하면 기존 recovery key를 재사용하여 secret storage를 bootstrap
- 교차 서명을 bootstrap하고 누락된 공개 교차 서명 키 업로드
- 현재 device를 표시하고 교차 서명하려고 시도
- 서버 측 room-key 백업이 아직 없으면 새로 생성

홈서버가 교차 서명 키 업로드에 interactive auth를 요구하면, OpenClaw는 먼저 인증 없이 업로드를 시도한 다음 `m.login.dummy`, 그다음 `channels.matrix.password`가 구성된 경우 `m.login.password`로 시도합니다.

현재 교차 서명 ID를 폐기하고 새로 만들려는 의도가 있을 때만 `--force-reset-cross-signing`을 사용하세요.

현재 room-key 백업을 의도적으로 폐기하고 향후 메시지를 위한 새 백업 기준 상태를 시작하려면 `openclaw matrix verify backup reset --yes`를 사용하세요.
복구할 수 없는 이전 암호화 기록은 계속 사용할 수 없게 되며 OpenClaw가 현재 백업 secret을 안전하게 불러올 수 없을 경우 secret storage를 다시 만들 수 있다는 점을 받아들일 때만 이렇게 하세요.

### 새 백업 기준 상태

향후 암호화된 메시지를 계속 정상적으로 사용하고, 복구할 수 없는 이전 기록은 잃어도 괜찮다면 다음 명령을 순서대로 실행하세요:

```bash
openclaw matrix verify backup reset --yes
openclaw matrix verify backup status --verbose
openclaw matrix verify status
```

특정 이름 있는 Matrix 계정을 명시적으로 대상으로 하려면 각 명령에 `--account <id>`를 추가하세요.

### 시작 동작

`encryption: true`일 때 Matrix는 `startupVerification` 기본값을 `"if-unverified"`로 설정합니다.
시작 시 이 device가 아직 검증되지 않았으면 Matrix는 다른 Matrix 클라이언트에서 self-verification을 요청하고, 이미 하나가 대기 중일 때 중복 요청을 건너뛰며, 재시작 후 재시도 전에 로컬 cooldown을 적용합니다.
실패한 요청 시도는 기본적으로 성공적인 요청 생성보다 더 빨리 재시도됩니다.
자동 시작 요청을 비활성화하려면 `startupVerification: "off"`로 설정하거나, 더 짧거나 긴 재시도 창이 필요하면 `startupVerificationCooldownHours`를 조정하세요.

시작 시에는 보수적인 crypto bootstrap 단계도 자동으로 수행됩니다.
이 단계는 먼저 현재 secret storage와 교차 서명 ID를 재사용하려고 하며, 명시적인 bootstrap 복구 흐름을 실행하지 않는 한 교차 서명을 재설정하지 않습니다.

시작 중 손상된 bootstrap 상태를 발견하고 `channels.matrix.password`가 구성되어 있으면 OpenClaw는 더 엄격한 복구 경로를 시도할 수 있습니다.
현재 device가 이미 소유자 서명 상태라면 OpenClaw는 이를 자동으로 재설정하지 않고 해당 ID를 보존합니다.

전체 업그레이드 흐름, 제한 사항, 복구 명령, 일반적인 마이그레이션 메시지는 [Matrix migration](/ko/install/migrating-matrix)을 참조하세요.

### 검증 알림

Matrix는 검증 수명 주기 알림을 엄격한 DM 검증 room에 `m.notice` 메시지로 직접 게시합니다.
여기에는 다음이 포함됩니다:

- 검증 요청 알림
- 검증 준비 완료 알림(명시적인 "이모지로 검증" 안내 포함)
- 검증 시작 및 완료 알림
- 사용 가능한 경우 SAS 세부 정보(이모지 및 십진수)

다른 Matrix 클라이언트에서 들어온 검증 요청은 OpenClaw가 추적하고 자동 수락합니다.
self-verification 흐름의 경우 OpenClaw는 이모지 검증을 사용할 수 있게 되면 SAS 흐름도 자동으로 시작하고 자체 측 확인도 수행합니다.
다른 Matrix 사용자/device의 검증 요청에 대해서는 OpenClaw가 요청을 자동 수락한 뒤 SAS 흐름이 정상적으로 진행되도록 기다립니다.
검증을 완료하려면 여전히 Matrix 클라이언트에서 이모지 또는 십진수 SAS를 비교하고 "They match"를 확인해야 합니다.

OpenClaw는 self-initiated 중복 흐름을 무조건 자동 수락하지 않습니다. 시작 시에는 self-verification 요청이 이미 대기 중이면 새 요청을 만들지 않습니다.

검증 프로토콜/시스템 알림은 agent 채팅 파이프라인으로 전달되지 않으므로 `NO_REPLY`를 생성하지 않습니다.

### device 관리

이전 OpenClaw 관리 Matrix device가 계정에 쌓이면 암호화된 room의 신뢰 상태를 이해하기 어려워질 수 있습니다.
다음으로 목록을 확인하세요:

```bash
openclaw matrix devices list
```

오래된 OpenClaw 관리 device 제거:

```bash
openclaw matrix devices prune-stale
```

### crypto 저장소

Matrix E2EE는 Node에서 공식 `matrix-js-sdk` Rust crypto 경로를 사용하며 IndexedDB shim으로 `fake-indexeddb`를 사용합니다. Crypto 상태는 스냅샷 파일(`crypto-idb-snapshot.json`)에 영속화되고 시작 시 복원됩니다. 이 스냅샷 파일은 제한적인 파일 권한으로 저장되는 민감한 런타임 상태입니다.

암호화된 런타임 상태는 계정별, 사용자별 token hash 루트 아래의 `~/.openclaw/matrix/accounts/<account>/<homeserver>__<user>/<token-hash>/`에 저장됩니다.
이 디렉터리에는 sync 저장소(`bot-storage.json`), crypto 저장소(`crypto/`), recovery key 파일(`recovery-key.json`), IndexedDB 스냅샷(`crypto-idb-snapshot.json`), thread 바인딩(`thread-bindings.json`), 시작 검증 상태(`startup-verification.json`)가 포함됩니다.
token이 바뀌더라도 계정 ID가 같으면 OpenClaw는 해당 account/homeserver/user 조합에 대해 가장 적절한 기존 루트를 재사용하므로 이전 sync 상태, crypto 상태, thread 바인딩, 시작 검증 상태를 계속 볼 수 있습니다.

## 프로필 관리

선택한 계정의 Matrix self-profile을 다음으로 업데이트합니다:

```bash
openclaw matrix profile set --name "OpenClaw Assistant"
openclaw matrix profile set --avatar-url https://cdn.example.org/avatar.png
```

특정 이름 있는 계정을 명시적으로 대상으로 하려면 `--account <id>`를 추가하세요.

Matrix는 `mxc://` avatar URL을 직접 허용합니다. `http://` 또는 `https://` avatar URL을 전달하면 OpenClaw가 먼저 이를 Matrix에 업로드하고 해석된 `mxc://` URL을 `channels.matrix.avatarUrl`(또는 선택한 계정 재정의)에 다시 저장합니다.

## 스레드

Matrix는 자동 답변과 message-tool 전송 모두에 대해 기본 Matrix 스레드를 지원합니다.

- `dm.sessionScope: "per-user"`(기본값)는 Matrix DM 라우팅을 발신자 범위로 유지하므로, 같은 상대방으로 해석되면 여러 DM room이 하나의 세션을 공유할 수 있습니다.
- `dm.sessionScope: "per-room"`은 정상적인 DM 인증 및 허용 목록 검사를 계속 사용하면서 각 Matrix DM room을 자체 세션 키로 격리합니다.
- 명시적인 Matrix 대화 바인딩은 여전히 `dm.sessionScope`보다 우선하므로, 바인딩된 room과 스레드는 선택된 대상 세션을 유지합니다.
- `threadReplies: "off"`는 답변을 최상위로 유지하고 들어오는 스레드 메시지를 부모 세션에 유지합니다.
- `threadReplies: "inbound"`는 들어온 메시지가 이미 해당 스레드에 있을 때만 스레드 안에서 답변합니다.
- `threadReplies: "always"`는 room 답변을 트리거 메시지를 루트로 하는 스레드에 유지하고, 첫 트리거 메시지부터 일치하는 스레드 범위 세션을 통해 해당 대화를 라우팅합니다.
- `dm.threadReplies`는 DM에 대해서만 최상위 설정을 재정의합니다. 예를 들어 room 스레드는 격리한 채 DM은 평면으로 유지할 수 있습니다.
- 들어오는 스레드 메시지에는 추가 agent 컨텍스트로 스레드 루트 메시지가 포함됩니다.
- message-tool 전송은 명시적인 `threadId`가 제공되지 않는 한 대상이 같은 room 또는 같은 DM 사용자 대상일 때 현재 Matrix 스레드를 자동 상속합니다.
- 동일 세션 DM 사용자 대상 재사용은 현재 세션 메타데이터가 같은 Matrix 계정의 동일한 DM 상대방임을 입증할 때만 시작되며, 그렇지 않으면 OpenClaw는 일반적인 사용자 범위 라우팅으로 되돌아갑니다.
- OpenClaw가 같은 공유 Matrix DM 세션에서 Matrix DM room이 다른 DM room과 충돌하는 것을 감지하면, thread 바인딩이 활성화되어 있고 `dm.sessionScope` 힌트가 있는 경우 해당 room에 `/focus` 탈출구를 안내하는 일회성 `m.notice`를 게시합니다.
- Matrix는 런타임 thread 바인딩을 지원합니다. `/focus`, `/unfocus`, `/agents`, `/session idle`, `/session max-age`, 그리고 스레드에 바인딩된 `/acp spawn`은 Matrix room과 DM에서 동작합니다.
- 최상위 Matrix room/DM의 `/focus`는 `threadBindings.spawnSubagentSessions=true`일 때 새 Matrix 스레드를 만들고 이를 대상 세션에 바인딩합니다.
- 기존 Matrix 스레드 내부에서 `/focus` 또는 `/acp spawn --thread here`를 실행하면 대신 현재 스레드를 바인딩합니다.

## ACP 대화 바인딩

Matrix room, DM, 기존 Matrix 스레드는 채팅 표면을 바꾸지 않고도 지속 가능한 ACP 작업 공간으로 전환할 수 있습니다.

빠른 운영자 흐름:

- 계속 사용할 Matrix DM, room 또는 기존 스레드 안에서 `/acp spawn codex --bind here`를 실행합니다.
- 최상위 Matrix DM 또는 room에서는 현재 DM/room이 채팅 표면으로 유지되고 향후 메시지는 생성된 ACP 세션으로 라우팅됩니다.
- 기존 Matrix 스레드 안에서는 `--bind here`가 현재 스레드를 제자리에서 바인딩합니다.
- `/new`와 `/reset`은 같은 바인딩된 ACP 세션을 제자리에서 재설정합니다.
- `/acp close`는 ACP 세션을 닫고 바인딩을 제거합니다.

참고:

- `--bind here`는 하위 Matrix 스레드를 만들지 않습니다.
- `threadBindings.spawnAcpSessions`는 OpenClaw가 하위 Matrix 스레드를 만들거나 바인딩해야 하는 `/acp spawn --thread auto|here`에만 필요합니다.

### 스레드 바인딩 구성

Matrix는 `session.threadBindings`에서 전역 기본값을 상속하며 채널별 재정의도 지원합니다:

- `threadBindings.enabled`
- `threadBindings.idleHours`
- `threadBindings.maxAgeHours`
- `threadBindings.spawnSubagentSessions`
- `threadBindings.spawnAcpSessions`

Matrix 스레드 바인딩 spawn 플래그는 옵트인입니다:

- 최상위 `/focus`가 새 Matrix 스레드를 만들고 바인딩하도록 허용하려면 `threadBindings.spawnSubagentSessions: true`를 설정하세요.
- `/acp spawn --thread auto|here`가 ACP 세션을 Matrix 스레드에 바인딩하도록 허용하려면 `threadBindings.spawnAcpSessions: true`를 설정하세요.

## 반응

Matrix는 outbound 반응 작업, inbound 반응 알림, inbound ack 반응을 지원합니다.

- Outbound 반응 도구는 `channels["matrix"].actions.reactions`로 제어됩니다.
- `react`는 특정 Matrix 이벤트에 반응을 추가합니다.
- `reactions`는 특정 Matrix 이벤트의 현재 반응 요약을 나열합니다.
- `emoji=""`는 해당 이벤트에 대한 봇 계정 자신의 반응을 제거합니다.
- `remove: true`는 봇 계정의 지정한 이모지 반응만 제거합니다.

Ack 반응은 표준 OpenClaw 해석 순서를 사용합니다:

- `channels["matrix"].accounts.<accountId>.ackReaction`
- `channels["matrix"].ackReaction`
- `messages.ackReaction`
- agent identity emoji 대체값

Ack 반응 범위는 다음 순서로 해석됩니다:

- `channels["matrix"].accounts.<accountId>.ackReactionScope`
- `channels["matrix"].ackReactionScope`
- `messages.ackReactionScope`

반응 알림 모드는 다음 순서로 해석됩니다:

- `channels["matrix"].accounts.<accountId>.reactionNotifications`
- `channels["matrix"].reactionNotifications`
- 기본값: `own`

동작:

- `reactionNotifications: "own"`은 봇이 작성한 Matrix 메시지를 대상으로 하는 추가된 `m.reaction` 이벤트를 전달합니다.
- `reactionNotifications: "off"`는 반응 시스템 이벤트를 비활성화합니다.
- Matrix는 반응 제거를 독립적인 `m.reaction` 제거가 아니라 redaction으로 노출하므로, 반응 제거는 시스템 이벤트로 합성되지 않습니다.

## 기록 컨텍스트

- `channels.matrix.historyLimit`은 Matrix room 메시지가 agent를 트리거할 때 `InboundHistory`에 포함할 최근 room 메시지 수를 제어합니다. `messages.groupChat.historyLimit`으로 대체되며, 둘 다 설정되지 않으면 실제 기본값은 `0`입니다. 비활성화하려면 `0`으로 설정하세요.
- Matrix room 기록은 room 전용입니다. DM은 계속 일반 세션 기록을 사용합니다.
- Matrix room 기록은 pending 전용입니다. OpenClaw는 아직 답변을 트리거하지 않은 room 메시지를 버퍼링한 다음, 멘션이나 다른 트리거가 오면 해당 창을 스냅샷합니다.
- 현재 트리거 메시지는 `InboundHistory`에 포함되지 않습니다. 해당 턴의 기본 inbound 본문에 그대로 남습니다.
- 같은 Matrix 이벤트의 재시도는 새로운 room 메시지로 앞으로 이동하지 않고 원래 기록 스냅샷을 재사용합니다.

## 컨텍스트 가시성

Matrix는 가져온 답장 텍스트, 스레드 루트, 대기 중 기록과 같은 보조 room 컨텍스트를 위한 공용 `contextVisibility` 제어를 지원합니다.

- `contextVisibility: "all"`이 기본값입니다. 보조 컨텍스트는 받은 그대로 유지됩니다.
- `contextVisibility: "allowlist"`는 보조 컨텍스트를 활성 room/사용자 허용 목록 검사에서 허용된 발신자로 필터링합니다.
- `contextVisibility: "allowlist_quote"`는 `allowlist`처럼 동작하지만 하나의 명시적 인용 답장은 계속 유지합니다.

이 설정은 보조 컨텍스트 가시성에 영향을 주며, inbound 메시지 자체가 답변을 트리거할 수 있는지 여부에는 영향을 주지 않습니다.
트리거 권한은 여전히 `groupPolicy`, `groups`, `groupAllowFrom`, DM 정책 설정에서 결정됩니다.

## DM 및 room 정책

```json5
{
  channels: {
    matrix: {
      dm: {
        policy: "allowlist",
        allowFrom: ["@admin:example.org"],
        threadReplies: "off",
      },
      groupPolicy: "allowlist",
      groupAllowFrom: ["@admin:example.org"],
      groups: {
        "!roomid:example.org": {
          requireMention: true,
        },
      },
    },
  },
}
```

멘션 게이팅 및 허용 목록 동작은 [Groups](/ko/channels/groups)를 참조하세요.

Matrix DM 페어링 예제:

```bash
openclaw pairing list matrix
openclaw pairing approve matrix <CODE>
```

승인되지 않은 Matrix 사용자가 승인 전에 계속 메시지를 보내면 OpenClaw는 새 코드를 발급하지 않고 동일한 대기 중 pairing 코드를 재사용하며, 짧은 cooldown 후 다시 리마인더 답장을 보낼 수 있습니다.

공용 DM pairing 흐름 및 저장소 레이아웃은 [Pairing](/ko/channels/pairing)을 참조하세요.

## direct room 복구

direct-message 상태가 동기화되지 않으면 OpenClaw가 라이브 DM 대신 오래된 solo room을 가리키는 낡은 `m.direct` 매핑을 갖게 될 수 있습니다. 다음으로 상대방의 현재 매핑을 검사합니다:

```bash
openclaw matrix direct inspect --user-id @alice:example.org
```

다음으로 복구합니다:

```bash
openclaw matrix direct repair --user-id @alice:example.org
```

복구 흐름:

- 이미 `m.direct`에 매핑된 엄격한 1:1 DM을 우선 선택
- 해당 사용자와 현재 참여 중인 엄격한 1:1 DM으로 대체
- 정상적인 DM이 없으면 새 direct room을 만들고 `m.direct`를 다시 작성

복구 흐름은 오래된 room을 자동 삭제하지 않습니다. 정상적인 DM을 선택하고 매핑을 업데이트하여 새로운 Matrix 전송, 검증 알림, 기타 direct-message 흐름이 다시 올바른 room을 대상으로 하도록만 합니다.

## exec 승인

Matrix는 Matrix 계정의 네이티브 승인 클라이언트 역할을 할 수 있습니다. 네이티브 DM/채널 라우팅 제어는 여전히 exec 승인 config 아래에 있습니다:

- `channels.matrix.execApprovals.enabled`
- `channels.matrix.execApprovals.approvers` (선택 사항, `channels.matrix.dm.allowFrom`으로 대체)
- `channels.matrix.execApprovals.target` (`dm` | `channel` | `both`, 기본값: `dm`)
- `channels.matrix.execApprovals.agentFilter`
- `channels.matrix.execApprovals.sessionFilter`

승인자는 `@owner:example.org`와 같은 Matrix 사용자 ID여야 합니다. Matrix는 `enabled`가 설정되지 않았거나 `"auto"`이고 적어도 하나의 승인자를 해석할 수 있으면 네이티브 승인을 자동 활성화합니다. Exec 승인은 먼저 `execApprovals.approvers`를 사용하고 `channels.matrix.dm.allowFrom`으로 대체될 수 있습니다. Plugin 승인은 `channels.matrix.dm.allowFrom`을 통해 승인합니다. Matrix를 네이티브 승인 클라이언트로 명시적으로 비활성화하려면 `enabled: false`로 설정하세요. 그렇지 않으면 승인 요청은 다른 구성된 승인 경로 또는 승인 대체 정책으로 대체됩니다.

Matrix 네이티브 라우팅은 두 승인 종류를 모두 지원합니다:

- `channels.matrix.execApprovals.*`는 Matrix 승인 프롬프트의 네이티브 DM/채널 fanout 모드를 제어합니다.
- Exec 승인은 `execApprovals.approvers` 또는 `channels.matrix.dm.allowFrom`의 exec 승인자 집합을 사용합니다.
- Plugin 승인은 Matrix DM 허용 목록 `channels.matrix.dm.allowFrom`을 사용합니다.
- Matrix 반응 바로가기와 메시지 업데이트는 exec 승인과 plugin 승인 모두에 적용됩니다.

전달 규칙:

- `target: "dm"`은 승인 프롬프트를 승인자 DM으로 전송
- `target: "channel"`은 프롬프트를 원래 Matrix room 또는 DM으로 다시 전송
- `target: "both"`는 승인자 DM과 원래 Matrix room 또는 DM 둘 다로 전송

Matrix 승인 프롬프트는 기본 승인 메시지에 반응 바로가기를 추가합니다:

- `✅` = 한 번 허용
- `❌` = 거부
- `♾️` = 해당 결정이 유효한 exec 정책에서 허용될 때 항상 허용

승인자는 해당 메시지에 반응하거나 대체 슬래시 명령 `/approve <id> allow-once`, `/approve <id> allow-always`, 또는 `/approve <id> deny`를 사용할 수 있습니다.

해석된 승인자만 승인 또는 거부할 수 있습니다. Exec 승인의 경우 채널 전달에는 명령 텍스트가 포함되므로 신뢰할 수 있는 room에서만 `channel` 또는 `both`를 활성화하세요.

계정별 재정의:

- `channels.matrix.accounts.<account>.execApprovals`

관련 문서: [Exec approvals](/ko/tools/exec-approvals)

## 다중 계정

```json5
{
  channels: {
    matrix: {
      enabled: true,
      defaultAccount: "assistant",
      dm: { policy: "pairing" },
      accounts: {
        assistant: {
          homeserver: "https://matrix.example.org",
          accessToken: "syt_assistant_xxx",
          encryption: true,
        },
        alerts: {
          homeserver: "https://matrix.example.org",
          accessToken: "syt_alerts_xxx",
          dm: {
            policy: "allowlist",
            allowFrom: ["@ops:example.org"],
            threadReplies: "off",
          },
        },
      },
    },
  },
}
```

최상위 `channels.matrix` 값은 계정이 재정의하지 않는 한 이름 있는 계정의 기본값으로 동작합니다.
상속된 room 항목 하나를 특정 Matrix 계정에 범위 지정하려면 `groups.<room>.account`를 사용할 수 있습니다.
`account`가 없는 항목은 모든 Matrix 계정에서 공유되며, `account: "default"`가 있는 항목도 기본 계정이 최상위 `channels.matrix.*`에 직접 구성되어 있을 때 계속 동작합니다.
부분적인 공유 인증 기본값만으로는 별도의 암묵적 기본 계정을 만들지 않습니다. OpenClaw는 해당 기본값에 새로운 인증(`homeserver` + `accessToken` 또는 `homeserver` + `userId` + `password`)이 있을 때만 최상위 `default` 계정을 합성하며, 이름 있는 계정은 나중에 캐시된 자격 증명이 인증을 충족하면 `homeserver` + `userId`만으로도 계속 검색 가능할 수 있습니다.
Matrix에 이미 정확히 하나의 이름 있는 계정이 있거나 `defaultAccount`가 기존 이름 있는 계정 키를 가리키는 경우, 단일 계정에서 다중 계정으로의 복구/설정 승격은 새 `accounts.default` 항목을 만들지 않고 해당 계정을 보존합니다. Matrix 인증/bootstrap 키만 승격된 계정으로 이동하며, 공유 전달 정책 키는 최상위에 유지됩니다.
암묵적 라우팅, 프로빙, CLI 작업에 대해 하나의 이름 있는 Matrix 계정을 우선 사용하게 하려면 `defaultAccount`를 설정하세요.
이름 있는 계정을 여러 개 구성한 경우 암묵적 계정 선택에 의존하는 CLI 명령에는 `defaultAccount`를 설정하거나 `--account <id>`를 전달하세요.
하나의 명령에 대해 이 암묵적 선택을 재정의하려면 `openclaw matrix verify ...`와 `openclaw matrix devices ...`에 `--account <id>`를 전달하세요.

공용 다중 계정 패턴은 [Configuration reference](/ko/gateway/configuration-reference#multi-account-all-channels)를 참조하세요.

## 비공개/LAN homeserver

기본적으로 OpenClaw는 SSRF 보호를 위해 비공개/내부 Matrix homeserver를 차단하며, 계정별로 명시적으로 옵트인해야만 허용합니다.

homeserver가 localhost, LAN/Tailscale IP, 또는 내부 호스트 이름에서 실행되는 경우 해당 Matrix 계정에 대해 `network.dangerouslyAllowPrivateNetwork`를 활성화하세요:

```json5
{
  channels: {
    matrix: {
      homeserver: "http://matrix-synapse:8008",
      network: {
        dangerouslyAllowPrivateNetwork: true,
      },
      accessToken: "syt_internal_xxx",
    },
  },
}
```

CLI 설정 예제:

```bash
openclaw matrix account add \
  --account ops \
  --homeserver http://matrix-synapse:8008 \
  --allow-private-network \
  --access-token syt_ops_xxx
```

이 옵트인은 신뢰할 수 있는 비공개/내부 대상만 허용합니다. `http://matrix.example.org:8008` 같은 공개 cleartext homeserver는 여전히 차단됩니다. 가능하면 `https://`를 우선 사용하세요.

## Matrix 트래픽 프록시

Matrix 배포에 명시적인 outbound HTTP(S) 프록시가 필요하면 `channels.matrix.proxy`를 설정하세요:

```json5
{
  channels: {
    matrix: {
      homeserver: "https://matrix.example.org",
      accessToken: "syt_bot_xxx",
      proxy: "http://127.0.0.1:7890",
    },
  },
}
```

이름 있는 계정은 `channels.matrix.accounts.<id>.proxy`로 최상위 기본값을 재정의할 수 있습니다.
OpenClaw는 런타임 Matrix 트래픽과 계정 상태 프로브에 동일한 프록시 설정을 사용합니다.

## 대상 해석

Matrix는 OpenClaw가 room 또는 사용자 대상을 요청하는 모든 위치에서 다음 대상 형식을 허용합니다:

- 사용자: `@user:server`, `user:@user:server`, 또는 `matrix:user:@user:server`
- Room: `!room:server`, `room:!room:server`, 또는 `matrix:room:!room:server`
- Alias: `#alias:server`, `channel:#alias:server`, 또는 `matrix:channel:#alias:server`

라이브 디렉터리 조회는 로그인된 Matrix 계정을 사용합니다:

- 사용자 조회는 해당 homeserver의 Matrix 사용자 디렉터리를 조회합니다.
- Room 조회는 명시적인 room ID와 alias를 직접 허용한 뒤 해당 계정에 대해 참여 중인 room 이름 검색으로 대체됩니다.
- 참여 중인 room 이름 조회는 최선의 노력 방식입니다. room 이름을 ID 또는 alias로 해석할 수 없으면 허용 목록 해석 시 런타임에서 무시됩니다.

## 구성 참조

- `enabled`: 채널 활성화 또는 비활성화.
- `name`: 계정의 선택적 레이블.
- `defaultAccount`: 여러 Matrix 계정이 구성된 경우 선호되는 계정 ID.
- `homeserver`: homeserver URL, 예: `https://matrix.example.org`.
- `network.dangerouslyAllowPrivateNetwork`: 이 Matrix 계정이 비공개/내부 homeserver에 연결되도록 허용합니다. homeserver가 `localhost`, LAN/Tailscale IP, 또는 `matrix-synapse` 같은 내부 호스트로 해석될 때 이를 활성화하세요.
- `proxy`: Matrix 트래픽용 선택적 HTTP(S) 프록시 URL. 이름 있는 계정은 자체 `proxy`로 최상위 기본값을 재정의할 수 있습니다.
- `userId`: 전체 Matrix 사용자 ID, 예: `@bot:example.org`.
- `accessToken`: token 기반 인증용 access token. 일반 텍스트 값과 SecretRef 값은 env/file/exec provider 전반에서 `channels.matrix.accessToken` 및 `channels.matrix.accounts.<id>.accessToken`에 대해 지원됩니다. [Secrets Management](/ko/gateway/secrets)를 참조하세요.
- `password`: password 기반 로그인용 password. 일반 텍스트 값과 SecretRef 값이 지원됩니다.
- `deviceId`: 명시적 Matrix device ID.
- `deviceName`: password 로그인용 device 표시 이름.
- `avatarUrl`: 프로필 동기화와 `profile set` 업데이트를 위한 저장된 self-avatar URL.
- `initialSyncLimit`: 시작 sync 중 가져오는 최대 이벤트 수.
- `encryption`: E2EE 활성화.
- `allowlistOnly`: `true`일 때 `open` room 정책을 `allowlist`로 승격하고, 활성 DM 정책 중 `disabled`를 제외한 모든 정책(`pairing` 및 `open` 포함)을 `allowlist`로 강제합니다. `disabled` 정책에는 영향을 주지 않습니다.
- `allowBots`: 다른 구성된 OpenClaw Matrix 계정의 메시지 허용(`true` 또는 `"mentions"`).
- `groupPolicy`: `open`, `allowlist`, 또는 `disabled`.
- `contextVisibility`: 보조 room 컨텍스트 가시성 모드 (`all`, `allowlist`, `allowlist_quote`).
- `groupAllowFrom`: room 트래픽용 사용자 ID 허용 목록. 항목은 전체 Matrix 사용자 ID여야 하며, 해석되지 않은 이름은 런타임에서 무시됩니다.
- `historyLimit`: 그룹 기록 컨텍스트에 포함할 최대 room 메시지 수. `messages.groupChat.historyLimit`으로 대체되며, 둘 다 설정되지 않으면 실제 기본값은 `0`입니다. 비활성화하려면 `0`으로 설정하세요.
- `replyToMode`: `off`, `first`, `all`, 또는 `batched`.
- `markdown`: outbound Matrix 텍스트용 선택적 Markdown 렌더링 구성.
- `streaming`: `off`(기본값), `"partial"`, `"quiet"`, `true`, 또는 `false`. `"partial"`과 `true`는 일반 Matrix 텍스트 메시지로 미리보기 우선 초안 업데이트를 활성화합니다. `"quiet"`은 자체 호스팅 push-rule 설정을 위한 무알림 미리보기 알림을 사용합니다. `false`는 `"off"`와 같습니다.
- `blockStreaming`: `true`는 초안 미리보기 스트리밍이 활성화되어 있을 때 완료된 assistant 블록용 별도 진행 상황 메시지를 활성화합니다.
- `threadReplies`: `off`, `inbound`, 또는 `always`.
- `threadBindings`: 스레드 바인딩 세션 라우팅 및 수명 주기에 대한 채널별 재정의.
- `startupVerification`: 시작 시 자동 self-verification 요청 모드 (`if-unverified`, `off`).
- `startupVerificationCooldownHours`: 자동 시작 검증 요청을 다시 시도하기 전 cooldown.
- `textChunkLimit`: 문자 기준 outbound 메시지 청크 크기 (`chunkMode`가 `length`일 때 적용).
- `chunkMode`: `length`는 문자 수 기준으로 메시지를 분할하고, `newline`은 줄 경계에서 분할합니다.
- `responsePrefix`: 이 채널의 모든 outbound 답변 앞에 붙는 선택적 문자열.
- `ackReaction`: 이 채널/계정에 대한 선택적 ack 반응 재정의.
- `ackReactionScope`: 선택적 ack 반응 범위 재정의 (`group-mentions`, `group-all`, `direct`, `all`, `none`, `off`).
- `reactionNotifications`: inbound 반응 알림 모드 (`own`, `off`).
- `mediaMaxMb`: outbound 전송 및 inbound 미디어 처리용 미디어 크기 상한(MB).
- `autoJoin`: 초대 자동 참여 정책 (`always`, `allowlist`, `off`). 기본값: `off`. DM 스타일 초대를 포함한 모든 Matrix 초대에 적용됩니다.
- `autoJoinAllowlist`: `autoJoin`이 `allowlist`일 때 허용되는 room/alias. Alias 항목은 초대 처리 중 room ID로 해석되며, OpenClaw는 초대된 room이 주장하는 alias 상태를 신뢰하지 않습니다.
- `dm`: DM 정책 블록 (`enabled`, `policy`, `allowFrom`, `sessionScope`, `threadReplies`).
- `dm.policy`: OpenClaw가 room에 참여하고 이를 DM으로 분류한 후의 DM 접근을 제어합니다. 초대가 자동 참여되는지 여부는 변경하지 않습니다.
- `dm.allowFrom`: live directory lookup으로 이미 해석하지 않았다면 항목은 전체 Matrix 사용자 ID여야 합니다.
- `dm.sessionScope`: `per-user`(기본값) 또는 `per-room`. 같은 상대방이라도 각 Matrix DM room이 별도 컨텍스트를 유지하게 하려면 `per-room`을 사용하세요.
- `dm.threadReplies`: DM 전용 스레드 정책 재정의 (`off`, `inbound`, `always`). DMs에서 답변 배치와 세션 격리 모두에 대해 최상위 `threadReplies` 설정을 재정의합니다.
- `execApprovals`: Matrix 네이티브 exec 승인 전달 (`enabled`, `approvers`, `target`, `agentFilter`, `sessionFilter`).
- `execApprovals.approvers`: exec 요청을 승인할 수 있는 Matrix 사용자 ID. `dm.allowFrom`이 이미 승인자를 식별하는 경우 선택 사항입니다.
- `execApprovals.target`: `dm | channel | both` (기본값: `dm`).
- `accounts`: 이름 있는 계정별 재정의. 최상위 `channels.matrix` 값은 이 항목들의 기본값으로 동작합니다.
- `groups`: room별 정책 맵. room ID 또는 alias를 권장하며, 해석되지 않은 room 이름은 런타임에서 무시됩니다. 세션/그룹 ID는 해석 후 안정적인 room ID를 사용합니다.
- `groups.<room>.account`: 다중 계정 구성에서 하나의 상속된 room 항목을 특정 Matrix 계정으로 제한합니다.
- `groups.<room>.allowBots`: 구성된 봇 발신자에 대한 room 수준 재정의 (`true` 또는 `"mentions"`).
- `groups.<room>.users`: room별 발신자 허용 목록.
- `groups.<room>.tools`: room별 도구 허용/거부 재정의.
- `groups.<room>.autoReply`: room 수준 멘션 게이팅 재정의. `true`는 해당 room의 멘션 요구 사항을 비활성화하고, `false`는 다시 강제 적용합니다.
- `groups.<room>.skills`: 선택적 room 수준 Skills 필터.
- `groups.<room>.systemPrompt`: 선택적 room 수준 system prompt 스니펫.
- `rooms`: `groups`의 레거시 alias.
- `actions`: 작업별 도구 게이팅 (`messages`, `reactions`, `pins`, `profile`, `memberInfo`, `channelInfo`, `verification`).

## 관련 항목

- [Channels Overview](/ko/channels) — 지원되는 모든 채널
- [Pairing](/ko/channels/pairing) — DM 인증 및 pairing 흐름
- [Groups](/ko/channels/groups) — 그룹 채팅 동작 및 멘션 게이팅
- [Channel Routing](/ko/channels/channel-routing) — 메시지 세션 라우팅
- [Security](/ko/gateway/security) — 접근 모델 및 하드닝
