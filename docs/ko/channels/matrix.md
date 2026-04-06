---
read_when:
    - OpenClaw에서 Matrix 설정하기
    - Matrix E2EE 및 검증 구성하기
summary: Matrix 지원 상태, 설정, 구성 예시
title: Matrix
x-i18n:
    generated_at: "2026-04-06T03:08:20Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3e2d84c08d7d5b96db14b914e54f08d25334401cdd92eb890bc8dfb37b0ca2dc
    source_path: channels/matrix.md
    workflow: 15
---

# Matrix

Matrix는 OpenClaw용 Matrix 번들 채널 plugin입니다.
공식 `matrix-js-sdk`를 사용하며 DM, 방, 스레드, 미디어, 리액션, 투표, 위치, E2EE를 지원합니다.

## 번들 plugin

Matrix는 현재 OpenClaw 릴리스에 번들 plugin으로 포함되어 제공되므로, 일반적인
패키지 빌드에서는 별도 설치가 필요하지 않습니다.

오래된 빌드 또는 Matrix가 제외된 커스텀 설치를 사용 중이라면 수동으로
설치하세요:

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
   - 현재 패키지형 OpenClaw 릴리스에는 이미 번들로 포함되어 있습니다.
   - 오래되었거나 커스텀 설치의 경우 위 명령으로 수동 추가할 수 있습니다.
2. 홈서버에서 Matrix 계정을 만듭니다.
3. `channels.matrix`를 다음 중 하나로 구성합니다:
   - `homeserver` + `accessToken`, 또는
   - `homeserver` + `userId` + `password`.
4. 게이트웨이를 재시작합니다.
5. 봇과 DM을 시작하거나 방에 초대합니다.

대화형 설정 경로:

```bash
openclaw channels add
openclaw configure --section channels
```

Matrix 마법사가 실제로 묻는 항목:

- 홈서버 URL
- 인증 방식: 액세스 토큰 또는 비밀번호
- 비밀번호 인증을 선택한 경우에만 사용자 ID
- 선택적 디바이스 이름
- E2EE 활성화 여부
- 지금 Matrix 방 접근을 구성할지 여부

중요한 마법사 동작:

- 선택한 계정에 대한 Matrix 인증 환경 변수가 이미 존재하고, 해당 계정에 인증 정보가 아직 config에 저장되어 있지 않다면, 마법사는 환경 변수 바로가기를 제안하고 해당 계정에 `enabled: true`만 기록합니다.
- 대화형으로 다른 Matrix 계정을 추가할 때 입력한 계정 이름은 config와 환경 변수에서 사용되는 계정 ID로 정규화됩니다. 예를 들어 `Ops Bot`은 `ops-bot`이 됩니다.
- DM 허용 목록 프롬프트는 전체 `@user:server` 값을 즉시 받아들입니다. 표시 이름은 실시간 디렉터리 조회로 정확히 하나의 일치 항목을 찾을 때만 동작하며, 그렇지 않으면 마법사가 전체 Matrix ID로 다시 시도하라고 요청합니다.
- 방 허용 목록 프롬프트는 방 ID와 별칭을 직접 받아들입니다. 가입된 방 이름을 실시간으로 해석할 수도 있지만, 해석되지 않은 이름은 설정 중 입력된 그대로만 유지되며 나중에 런타임 허용 목록 해석에서는 무시됩니다. `!room:server` 또는 `#alias:server` 사용을 권장합니다.
- 런타임 방/세션 식별은 안정적인 Matrix 방 ID를 사용합니다. 방에 선언된 별칭은 조회 입력에만 사용되며, 장기 세션 키나 안정적인 그룹 식별자로는 사용되지 않습니다.
- 저장 전에 방 이름을 해석하려면 `openclaw channels resolve --channel matrix "Project Room"`을 사용하세요.

최소 토큰 기반 설정:

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

비밀번호 기반 설정(로그인 후 토큰이 캐시됨):

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
기본 계정은 `credentials.json`을 사용하고, 이름 있는 계정은 `credentials-<account>.json`을 사용합니다.
여기에 캐시된 자격 증명이 있으면 현재 인증이 config에 직접 설정되어 있지 않더라도 OpenClaw는 설정, doctor, 채널 상태 탐지에서 Matrix가 구성된 것으로 간주합니다.

환경 변수 동등 항목(config 키가 설정되지 않은 경우 사용됨):

- `MATRIX_HOMESERVER`
- `MATRIX_ACCESS_TOKEN`
- `MATRIX_USER_ID`
- `MATRIX_PASSWORD`
- `MATRIX_DEVICE_ID`
- `MATRIX_DEVICE_NAME`

기본이 아닌 계정의 경우 계정 범위 환경 변수를 사용하세요:

- `MATRIX_<ACCOUNT_ID>_HOMESERVER`
- `MATRIX_<ACCOUNT_ID>_ACCESS_TOKEN`
- `MATRIX_<ACCOUNT_ID>_USER_ID`
- `MATRIX_<ACCOUNT_ID>_PASSWORD`
- `MATRIX_<ACCOUNT_ID>_DEVICE_ID`
- `MATRIX_<ACCOUNT_ID>_DEVICE_NAME`

계정 `ops` 예시:

- `MATRIX_OPS_HOMESERVER`
- `MATRIX_OPS_ACCESS_TOKEN`

정규화된 계정 ID `ops-bot`의 경우 다음을 사용합니다:

- `MATRIX_OPS_X2D_BOT_HOMESERVER`
- `MATRIX_OPS_X2D_BOT_ACCESS_TOKEN`

Matrix는 계정 ID 내 문장부호를 이스케이프하여 계정 범위 환경 변수가 충돌하지 않도록 합니다.
예를 들어 `-`는 `_X2D_`가 되므로, `ops-prod`는 `MATRIX_OPS_X2D_PROD_*`로 매핑됩니다.

대화형 마법사는 해당 인증 환경 변수가 이미 존재하고 선택한 계정에 Matrix 인증이 아직 config에 저장되어 있지 않은 경우에만 환경 변수 바로가기를 제안합니다.

## 구성 예시

다음은 DM 페어링, 방 허용 목록, E2EE 활성화를 포함한 실용적인 기본 구성입니다:

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

## 스트리밍 미리보기

Matrix 답변 스트리밍은 옵트인 방식입니다.

OpenClaw가 하나의 라이브 미리보기
답장을 보내고, 모델이 텍스트를 생성하는 동안 그 미리보기를 제자리에서 수정한 뒤,
답장이 완료되면 최종 확정하도록 하려면 `channels.matrix.streaming`을 `"partial"`로 설정하세요:

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
- `streaming: "partial"`은 현재 assistant 블록에 대해 수정 가능한 미리보기 메시지 하나를 일반 Matrix 텍스트 메시지로 생성합니다. 이 방식은 Matrix의 기존 미리보기 우선 알림 동작을 유지하므로, 기본 클라이언트는 완성된 블록 대신 첫 번째 스트리밍 미리보기 텍스트에 대해 알림할 수 있습니다.
- `streaming: "quiet"`은 현재 assistant 블록에 대해 수정 가능한 조용한 미리보기 알림 하나를 생성합니다. 이는 완료된 미리보기 수정에 대해 수신자 푸시 규칙도 함께 구성하는 경우에만 사용하세요.
- `blockStreaming: true`는 별도의 Matrix 진행 상황 메시지를 활성화합니다. 미리보기 스트리밍이 활성화된 경우 Matrix는 현재 블록의 라이브 초안을 유지하고 완료된 블록은 별도 메시지로 보존합니다.
- 미리보기 스트리밍이 켜져 있고 `blockStreaming`이 꺼져 있으면, Matrix는 라이브 초안을 제자리에서 수정하고 블록 또는 턴이 끝날 때 동일한 이벤트를 최종 확정합니다.
- 미리보기가 더 이상 하나의 Matrix 이벤트에 들어맞지 않으면, OpenClaw는 미리보기 스트리밍을 중단하고 일반 최종 전송으로 대체합니다.
- 미디어 답장은 여전히 첨부 파일을 일반적으로 전송합니다. 오래된 미리보기를 더 이상 안전하게 재사용할 수 없으면, OpenClaw는 최종 미디어 답장을 보내기 전에 이를 삭제 처리합니다.
- 미리보기 수정은 추가 Matrix API 호출 비용이 듭니다. 가장 보수적인 rate-limit 동작을 원한다면 스트리밍을 꺼 두세요.

`blockStreaming`만으로는 초안 미리보기가 활성화되지 않습니다.
미리보기 수정을 위해 `streaming: "partial"` 또는 `streaming: "quiet"`을 사용한 다음, 완료된 assistant 블록도 별도의 진행 상황 메시지로 남기고 싶을 때만 `blockStreaming: true`를 추가하세요.

커스텀 푸시 규칙 없이 기본 Matrix 알림이 필요하다면, 미리보기 우선 동작을 위해 `streaming: "partial"`을 사용하거나 최종 결과만 전송하려면 `streaming`을 꺼 두세요. `streaming: "off"`일 때:

- `blockStreaming: true`는 완료된 각 블록을 일반 알림 Matrix 메시지로 전송합니다.
- `blockStreaming: false`는 최종 완료된 답변만 일반 알림 Matrix 메시지로 전송합니다.

### 조용한 완료 미리보기를 위한 자체 호스팅 푸시 규칙

자체 Matrix 인프라를 운영하고 조용한 미리보기가 블록 또는
최종 답변이 완료될 때만 알림하도록 하려면, `streaming: "quiet"`을 설정하고 완료된 미리보기 수정에 대한 사용자별 푸시 규칙을 추가하세요.

이는 일반적으로 홈서버 전체 설정 변경이 아니라 수신 사용자 측 설정입니다:

시작 전 빠른 개요:

- 수신 사용자 = 알림을 받아야 하는 사람
- 봇 사용자 = 답변을 보내는 OpenClaw Matrix 계정
- 아래 API 호출에는 수신 사용자의 액세스 토큰을 사용하세요
- 푸시 규칙의 `sender`는 봇 사용자의 전체 MXID와 일치해야 합니다

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

2. 수신 계정이 이미 일반 Matrix 푸시 알림을 받고 있는지 확인합니다. 조용한 미리보기
   규칙은 해당 사용자에게 이미 동작하는 pusher/디바이스가 있을 때만 작동합니다.

3. 수신 사용자의 액세스 토큰을 가져옵니다.
   - 봇의 토큰이 아니라 수신 사용자의 토큰을 사용하세요.
   - 기존 클라이언트 세션 토큰을 재사용하는 것이 보통 가장 쉽습니다.
   - 새 토큰을 발급해야 한다면 표준 Matrix Client-Server API로 로그인할 수 있습니다:

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

4. 수신 계정에 이미 pusher가 있는지 확인합니다:

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushers"
```

이 호출 결과에 활성 pusher/디바이스가 없다면, 아래 OpenClaw
규칙을 추가하기 전에 먼저 일반 Matrix 알림부터 수정하세요.

OpenClaw는 완료된 텍스트 전용 미리보기 수정을 다음과 같이 표시합니다:

```json
{
  "com.openclaw.finalized_preview": true
}
```

5. 이러한 알림을 받아야 하는 각 수신 계정에 대해 override 푸시 규칙을 생성합니다:

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

명령 실행 전에 다음 값을 바꾸세요:

- `https://matrix.example.org`: 홈서버 기본 URL
- `$USER_ACCESS_TOKEN`: 수신 사용자의 액세스 토큰
- `openclaw-finalized-preview-botname`: 이 수신 사용자에 대해 이 봇에 고유한 규칙 ID
- `@bot:example.org`: 수신 사용자의 MXID가 아니라 OpenClaw Matrix 봇 MXID

다중 봇 설정에서 중요:

- 푸시 규칙은 `ruleId`를 키로 사용합니다. 동일한 규칙 ID에 대해 `PUT`을 다시 실행하면 그 하나의 규칙이 업데이트됩니다.
- 하나의 수신 사용자가 여러 OpenClaw Matrix 봇 계정에 대해 알림을 받아야 한다면, 각 sender 일치 조건마다 고유한 규칙 ID로 봇별 규칙 하나를 생성하세요.
- 단순한 패턴으로는 `openclaw-finalized-preview-<botname>`을 사용할 수 있으며, 예: `openclaw-finalized-preview-ops` 또는 `openclaw-finalized-preview-support`.

이 규칙은 이벤트 발신자를 기준으로 평가됩니다:

- 수신 사용자의 토큰으로 인증
- `sender`를 OpenClaw 봇 MXID와 일치시킴

6. 규칙이 존재하는지 확인합니다:

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname"
```

7. 스트리밍된 답변을 테스트합니다. 조용한 모드에서는 방에 조용한 초안 미리보기가 표시되고,
   최종 제자리 수정은 블록 또는 턴이 완료될 때 한 번 알림해야 합니다.

나중에 규칙을 제거해야 한다면, 동일한 규칙 ID를 수신 사용자의 토큰으로 삭제하세요:

```bash
curl -sS -X DELETE \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname"
```

참고:

- 규칙은 봇의 토큰이 아니라 수신 사용자의 액세스 토큰으로 생성하세요.
- 새 사용자 정의 `override` 규칙은 기본 suppress 규칙보다 앞에 삽입되므로, 추가 정렬 매개변수는 필요하지 않습니다.
- 이는 OpenClaw가 제자리에서 안전하게 완료 처리할 수 있는 텍스트 전용 미리보기 수정에만 영향을 줍니다. 미디어 폴백과 오래된 미리보기 폴백은 여전히 일반 Matrix 전송을 사용합니다.
- `GET /_matrix/client/v3/pushers`에 pusher가 보이지 않으면, 해당 사용자는 아직 이 계정/디바이스에서 동작하는 Matrix 푸시 전송을 갖고 있지 않습니다.

#### Synapse

Synapse에서는 위 설정만으로도 보통 충분합니다:

- 완료된 OpenClaw 미리보기 알림을 위해 특별한 `homeserver.yaml` 변경은 필요하지 않습니다.
- Synapse 배포가 이미 일반 Matrix 푸시 알림을 보내고 있다면, 사용자 토큰과 위의 `pushrules` 호출이 주요 설정 단계입니다.
- Synapse를 리버스 프록시나 워커 뒤에서 운영한다면 `/_matrix/client/.../pushrules/`가 Synapse로 올바르게 전달되는지 확인하세요.
- Synapse 워커를 사용하는 경우 pusher가 정상인지 확인하세요. 푸시 전송은 메인 프로세스 또는 `synapse.app.pusher` / 구성된 pusher 워커에서 처리됩니다.

#### Tuwunel

Tuwunel에서는 위에 표시된 동일한 설정 흐름과 push-rule API 호출을 사용하세요:

- 완료된 미리보기 표시 자체를 위해 Tuwunel 전용 config는 필요하지 않습니다.
- 해당 사용자에게 일반 Matrix 알림이 이미 동작한다면, 사용자 토큰과 위의 `pushrules` 호출이 주요 설정 단계입니다.
- 사용자가 다른 디바이스에서 활성 상태일 때 알림이 사라지는 것처럼 보인다면 `suppress_push_when_active`가 활성화되어 있는지 확인하세요. Tuwunel은 2025년 9월 12일 Tuwunel 1.4.2에서 이 옵션을 추가했으며, 한 디바이스가 활성 상태일 때 다른 디바이스로의 푸시를 의도적으로 억제할 수 있습니다.

## 암호화 및 검증

암호화된(E2EE) 방에서는 아웃바운드 이미지 이벤트가 `thumbnail_file`을 사용하므로 이미지 미리보기도 전체 첨부 파일과 함께 암호화됩니다. 암호화되지 않은 방은 여전히 일반 `thumbnail_url`을 사용합니다. 별도 구성은 필요하지 않습니다 — plugin이 E2EE 상태를 자동으로 감지합니다.

### 봇 대 봇 방

기본적으로, 다른 구성된 OpenClaw Matrix 계정에서 온 Matrix 메시지는 무시됩니다.

의도적으로 에이전트 간 Matrix 트래픽을 허용하려면 `allowBots`를 사용하세요:

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

- `allowBots: true`는 허용된 방과 DM에서 다른 구성된 Matrix 봇 계정의 메시지를 허용합니다.
- `allowBots: "mentions"`는 방에서 해당 메시지가 이 봇을 눈에 띄게 멘션하는 경우에만 허용합니다. DM은 여전히 허용됩니다.
- `groups.<room>.allowBots`는 한 방에 대해 계정 수준 설정을 재정의합니다.
- OpenClaw는 자기 자신에게 답장하는 루프를 피하기 위해 동일한 Matrix 사용자 ID의 메시지는 여전히 무시합니다.
- Matrix는 여기서 네이티브 봇 플래그를 제공하지 않으므로, OpenClaw는 "봇이 작성한" 것을 "이 OpenClaw 게이트웨이에 구성된 다른 Matrix 계정이 보낸 것"으로 간주합니다.

공유 방에서 봇 대 봇 트래픽을 활성화할 때는 엄격한 방 허용 목록과 멘션 요구 사항을 사용하세요.

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

기계 판독 가능한 출력에 저장된 복구 키 포함:

```bash
openclaw matrix verify status --include-recovery-key --json
```

교차 서명 및 검증 상태 부트스트랩:

```bash
openclaw matrix verify bootstrap
```

다중 계정 지원: 계정별 자격 증명과 선택적 `name`을 포함한 `channels.matrix.accounts`를 사용하세요. 공통 패턴은 [Configuration reference](/ko/gateway/configuration-reference#multi-account-all-channels)를 참조하세요.

상세 부트스트랩 진단:

```bash
openclaw matrix verify bootstrap --verbose
```

부트스트랩 전에 새로운 교차 서명 ID를 강제로 재설정:

```bash
openclaw matrix verify bootstrap --force-reset-cross-signing
```

복구 키로 이 디바이스를 검증:

```bash
openclaw matrix verify device "<your-recovery-key>"
```

자세한 디바이스 검증 정보:

```bash
openclaw matrix verify device "<your-recovery-key>" --verbose
```

방 키 백업 상태 확인:

```bash
openclaw matrix verify backup status
```

자세한 백업 상태 진단:

```bash
openclaw matrix verify backup status --verbose
```

서버 백업에서 방 키 복원:

```bash
openclaw matrix verify backup restore
```

자세한 복원 진단:

```bash
openclaw matrix verify backup restore --verbose
```

현재 서버 백업을 삭제하고 새로운 백업 기준선을 생성합니다. 저장된
백업 키를 정상적으로 불러올 수 없는 경우, 이 재설정은 시크릿 저장소도 다시 생성하여
향후 콜드 스타트에서 새 백업 키를 불러올 수 있게 할 수 있습니다:

```bash
openclaw matrix verify backup reset --yes
```

모든 `verify` 명령은 기본적으로 간결하게 동작하며(조용한 내부 SDK 로깅 포함), 자세한 진단은 `--verbose`에서만 표시됩니다.
스크립팅에는 전체 기계 판독 가능한 출력인 `--json`을 사용하세요.

다중 계정 설정에서는 `--account <id>`를 전달하지 않으면 Matrix CLI 명령은 암묵적인 Matrix 기본 계정을 사용합니다.
이름 있는 계정을 여러 개 구성한 경우 `channels.matrix.defaultAccount`를 먼저 설정해야 하며, 그렇지 않으면 이러한 암묵적 CLI 작업은 중단되고 계정을 명시적으로 선택하라고 요청합니다.
검증 또는 디바이스 작업이 명시적으로 이름 있는 계정을 대상으로 하게 하려면 항상 `--account`를 사용하세요:

```bash
openclaw matrix verify status --account assistant
openclaw matrix verify backup restore --account assistant
openclaw matrix devices list --account assistant
```

암호화가 비활성화되어 있거나 이름 있는 계정에서 사용할 수 없는 경우, Matrix 경고 및 검증 오류는 해당 계정의 config 키를 가리킵니다. 예: `channels.matrix.accounts.assistant.encryption`.

### "검증됨"의 의미

OpenClaw는 이 Matrix 디바이스가 사용자의 자체 교차 서명 ID에 의해 검증된 경우에만 검증된 것으로 취급합니다.
실제로 `openclaw matrix verify status --verbose`는 세 가지 신뢰 신호를 노출합니다:

- `Locally trusted`: 이 디바이스는 현재 클라이언트에서만 신뢰됨
- `Cross-signing verified`: SDK가 이 디바이스를 교차 서명을 통해 검증된 것으로 보고함
- `Signed by owner`: 이 디바이스가 사용자의 자체 self-signing 키로 서명됨

`Verified by owner`는 교차 서명 검증 또는 소유자 서명이 있을 때만 `yes`가 됩니다.
로컬 신뢰만으로는 OpenClaw가 이 디바이스를 완전히 검증된 것으로 취급하기에 충분하지 않습니다.

### 부트스트랩이 하는 일

`openclaw matrix verify bootstrap`은 암호화된 Matrix 계정용 복구 및 설정 명령입니다.
이 명령은 다음을 순서대로 모두 수행합니다:

- 가능하면 기존 복구 키를 재사용하면서 시크릿 저장소를 부트스트랩
- 교차 서명을 부트스트랩하고 누락된 공개 교차 서명 키 업로드
- 현재 디바이스를 표시하고 교차 서명하려 시도
- 서버 측 방 키 백업이 아직 없으면 새로 생성

홈서버가 교차 서명 키 업로드에 대화형 인증을 요구하는 경우, OpenClaw는 먼저 인증 없이 업로드를 시도한 뒤, `m.login.dummy`, 그 다음 `channels.matrix.password`가 구성되어 있으면 `m.login.password` 순으로 시도합니다.

현재 교차 서명 ID를 폐기하고 새로 만들려는 경우에만 `--force-reset-cross-signing`을 사용하세요.

현재 방 키 백업을 의도적으로 폐기하고 향후 메시지용 새
백업 기준선을 시작하려면 `openclaw matrix verify backup reset --yes`를 사용하세요.
복구할 수 없는 오래된 암호화 기록이 계속
사용 불가능한 상태로 남는 것과, OpenClaw가 현재 백업
시크릿을 안전하게 불러올 수 없을 때 시크릿 저장소를 다시 생성할 수 있다는 점을 받아들일 때만 이렇게 하세요.

### 새 백업 기준선

향후 암호화된 메시지가 계속 동작하도록 유지하면서 복구할 수 없는 오래된 기록의 손실을 감수할 수 있다면, 다음 명령을 순서대로 실행하세요:

```bash
openclaw matrix verify backup reset --yes
openclaw matrix verify backup status --verbose
openclaw matrix verify status
```

명시적으로 이름 있는 Matrix 계정을 대상으로 하려면 각 명령에 `--account <id>`를 추가하세요.

### 시작 시 동작

`encryption: true`일 때 Matrix는 `startupVerification`을 기본적으로 `"if-unverified"`로 설정합니다.
시작 시 이 디바이스가 아직 검증되지 않았다면 Matrix는 다른 Matrix 클라이언트에서 자기 검증을 요청하고,
이미 하나가 대기 중이면 중복 요청을 건너뛰며, 재시작 후 재시도하기 전에 로컬 쿨다운을 적용합니다.
기본적으로 요청 생성이 성공한 경우보다 실패한 요청 시도가 더 빨리 재시도됩니다.
자동 시작 요청을 비활성화하려면 `startupVerification: "off"`로 설정하거나, 더 짧거나 긴 재시도 창을 원하면 `startupVerificationCooldownHours`
를 조정하세요.

시작 시에는 보수적인 crypto 부트스트랩 패스도 자동으로 수행합니다.
이 패스는 먼저 현재 시크릿 저장소와 교차 서명 ID를 재사용하려고 하며, 명시적인 부트스트랩 복구 흐름을 실행하지 않는 한 교차 서명 재설정을 피합니다.

시작 시 손상된 부트스트랩 상태가 발견되고 `channels.matrix.password`가 구성되어 있으면, OpenClaw는 더 엄격한 복구 경로를 시도할 수 있습니다.
현재 디바이스가 이미 소유자 서명 상태라면 OpenClaw는 이를 자동으로 재설정하지 않고 해당 ID를 보존합니다.

이전 공개 Matrix plugin에서 업그레이드하는 경우:

- OpenClaw는 가능하면 동일한 Matrix 계정, 액세스 토큰, 디바이스 ID를 자동으로 재사용합니다.
- 실행 가능한 Matrix 마이그레이션 변경이 수행되기 전에 OpenClaw는 `~/Backups/openclaw-migrations/` 아래에 복구 스냅샷을 생성하거나 재사용합니다.
- 여러 Matrix 계정을 사용하는 경우, 오래된 flat-store 레이아웃에서 업그레이드하기 전에 `channels.matrix.defaultAccount`를 설정하여 어떤 계정이 해당 공유 레거시 상태를 받아야 하는지 OpenClaw가 알 수 있도록 하세요.
- 이전 plugin이 Matrix 방 키 백업 복호화 키를 로컬에 저장했다면, 시작 시 또는 `openclaw doctor --fix`가 이를 새 복구 키 흐름으로 자동 가져옵니다.
- Matrix 액세스 토큰이 마이그레이션 준비 후 변경된 경우, 시작 시 이제 자동 백업 복원을 포기하기 전에 대기 중인 레거시 복원 상태를 찾기 위해 형제 token-hash 저장소 루트를 검사합니다.
- 이후 동일한 계정, 홈서버, 사용자에 대해 Matrix 액세스 토큰이 바뀌더라도, OpenClaw는 이제 빈 Matrix 상태 디렉터리에서 시작하는 대신 가장 완전한 기존 token-hash 저장소 루트를 재사용하는 것을 우선합니다.
- 다음 게이트웨이 시작 시 백업된 방 키가 새로운 crypto 저장소로 자동 복원됩니다.
- 오래된 plugin에 백업되지 않은 로컬 전용 방 키가 있었다면 OpenClaw는 이를 명확히 경고합니다. 이러한 키는 이전 rust crypto 저장소에서 자동으로 내보낼 수 없으므로, 일부 오래된 암호화 기록은 수동으로 복구할 때까지 계속 사용할 수 없을 수 있습니다.
- 전체 업그레이드 흐름, 제한 사항, 복구 명령, 일반적인 마이그레이션 메시지는 [Matrix migration](/ko/install/migrating-matrix)을 참조하세요.

암호화된 런타임 상태는 계정별, 사용자별 token-hash 루트 아래
`~/.openclaw/matrix/accounts/<account>/<homeserver>__<user>/<token-hash>/`에 구성됩니다.
이 디렉터리에는 동기화 저장소(`bot-storage.json`), crypto 저장소(`crypto/`),
복구 키 파일(`recovery-key.json`), IndexedDB 스냅샷(`crypto-idb-snapshot.json`),
스레드 바인딩(`thread-bindings.json`), 시작 시 검증 상태(`startup-verification.json`)
가 해당 기능 사용 시 포함됩니다.
토큰이 변경되더라도 계정 ID는 동일하다면 OpenClaw는 해당 account/homeserver/user 튜플에 대해 가장 적절한 기존
루트를 재사용하므로 이전 동기화 상태, crypto 상태, 스레드 바인딩,
시작 시 검증 상태를 계속 볼 수 있습니다.

### Node crypto 저장소 모델

이 plugin의 Matrix E2EE는 Node에서 공식 `matrix-js-sdk` Rust crypto 경로를 사용합니다.
이 경로는 crypto 상태를 재시작 후에도 유지하려면 IndexedDB 기반 영속성을 기대합니다.

현재 OpenClaw는 Node에서 이를 다음과 같이 제공합니다:

- SDK가 기대하는 IndexedDB API shim으로 `fake-indexeddb` 사용
- `initRustCrypto` 전에 `crypto-idb-snapshot.json`에서 Rust crypto IndexedDB 내용을 복원
- init 후와 런타임 중 갱신된 IndexedDB 내용을 다시 `crypto-idb-snapshot.json`에 저장
- 게이트웨이 런타임 영속화와 CLI 유지 관리가 같은 스냅샷 파일에서 경합하지 않도록 advisory 파일 잠금으로 `crypto-idb-snapshot.json`에 대한 스냅샷 복원과 저장을 직렬화

이는 커스텀 crypto 구현이 아니라 호환성/저장소용 배선입니다.
스냅샷 파일은 민감한 런타임 상태이며 제한적인 파일 권한으로 저장됩니다.
OpenClaw의 보안 모델에서는 게이트웨이 호스트와 로컬 OpenClaw 상태 디렉터리가 이미 신뢰된 운영자 경계 안에 있으므로, 이는 별도의 원격 신뢰 경계라기보다 주로 운영상 내구성 문제입니다.

계획된 개선 사항:

- 영속적인 Matrix 키 자료에 대한 SecretRef 지원 추가로, 복구 키와 관련 저장소 암호화 시크릿을 로컬 파일뿐 아니라 OpenClaw 시크릿 provider에서도 가져올 수 있도록 개선

## 프로필 관리

선택한 계정의 Matrix 자기 프로필을 업데이트하려면 다음을 사용하세요:

```bash
openclaw matrix profile set --name "OpenClaw Assistant"
openclaw matrix profile set --avatar-url https://cdn.example.org/avatar.png
```

명시적으로 이름 있는 Matrix 계정을 대상으로 하려면 `--account <id>`를 추가하세요.

Matrix는 `mxc://` 아바타 URL을 직접 받아들입니다. `http://` 또는 `https://` 아바타 URL을 전달하면 OpenClaw가 먼저 이를 Matrix에 업로드하고, 해석된 `mxc://` URL을 다시 `channels.matrix.avatarUrl`(또는 선택한 계정 override)에 저장합니다.

## 자동 검증 알림

이제 Matrix는 엄격한 DM 검증 방에 `m.notice` 메시지로 검증 수명 주기 알림을 직접 게시합니다.
여기에는 다음이 포함됩니다:

- 검증 요청 알림
- 검증 준비 완료 알림(명시적인 "이모지로 검증" 안내 포함)
- 검증 시작 및 완료 알림
- 사용 가능한 경우 SAS 세부 정보(이모지 및 숫자)

다른 Matrix 클라이언트에서 들어오는 검증 요청은 OpenClaw가 추적하고 자동 수락합니다.
자기 검증 흐름의 경우, OpenClaw는 이모지 검증이 가능해지면 SAS 흐름도 자동으로 시작하고 자체 측을 확인합니다.
다른 Matrix 사용자/디바이스의 검증 요청의 경우, OpenClaw는 요청을 자동 수락한 뒤 SAS 흐름이 정상적으로 진행되기를 기다립니다.
검증을 완료하려면 여전히 Matrix 클라이언트에서 이모지 또는 숫자 SAS를 비교하고 "일치함"을 확인해야 합니다.

OpenClaw는 자기가 시작한 중복 흐름을 무작정 자동 수락하지 않습니다. 시작 시 이미 자기 검증 요청이 대기 중이면 새 요청 생성을 건너뜁니다.

검증 프로토콜/시스템 알림은 에이전트 채팅 파이프라인으로 전달되지 않으므로 `NO_REPLY`를 생성하지 않습니다.

### 디바이스 위생

OpenClaw가 관리하는 오래된 Matrix 디바이스가 계정에 누적되면 암호화된 방의 신뢰를 이해하기 더 어려워질 수 있습니다.
다음 명령으로 목록을 확인하세요:

```bash
openclaw matrix devices list
```

오래된 OpenClaw 관리 디바이스를 제거하려면 다음을 사용하세요:

```bash
openclaw matrix devices prune-stale
```

### Direct Room 복구

다이렉트 메시지 상태가 동기화되지 않으면 OpenClaw가 오래된 단독 방을 가리키는 낡은 `m.direct` 매핑을 유지하고, 실제 라이브 DM이 아닌 곳을 참조하게 될 수 있습니다. 상대방에 대한 현재 매핑을 확인하려면 다음을 사용하세요:

```bash
openclaw matrix direct inspect --user-id @alice:example.org
```

다음으로 복구할 수 있습니다:

```bash
openclaw matrix direct repair --user-id @alice:example.org
```

복구는 Matrix 전용 로직을 plugin 내부에 유지합니다:

- 먼저 `m.direct`에 이미 매핑된 엄격한 1:1 DM을 선호합니다
- 그렇지 않으면 해당 사용자와 현재 가입 중인 엄격한 1:1 DM으로 대체합니다
- 정상적인 DM이 없으면 새 direct 방을 만들고 `m.direct`를 다시 써서 이를 가리키게 합니다

복구 흐름은 오래된 방을 자동으로 삭제하지 않습니다. 정상적인 DM을 선택하고 매핑을 업데이트하여 새 Matrix 전송, 검증 알림, 기타 direct-message 흐름이 다시 올바른 방을 대상으로 하도록만 합니다.

## 스레드

Matrix는 자동 답변과 메시지 도구 전송 모두에 대해 네이티브 Matrix 스레드를 지원합니다.

- `dm.sessionScope: "per-user"`(기본값)는 Matrix DM 라우팅을 발신자 범위로 유지하므로, 동일한 상대방으로 해석되면 여러 DM 방이 하나의 세션을 공유할 수 있습니다.
- `dm.sessionScope: "per-room"`은 정상적인 DM 인증 및 허용 목록 검사를 계속 사용하면서 각 Matrix DM 방을 고유한 세션 키로 분리합니다.
- 명시적인 Matrix 대화 바인딩은 여전히 `dm.sessionScope`보다 우선하므로, 바인딩된 방과 스레드는 선택한 대상 세션을 유지합니다.
- `threadReplies: "off"`는 답변을 최상위에 유지하고, 들어오는 스레드 메시지도 부모 세션에 유지합니다.
- `threadReplies: "inbound"`는 들어오는 메시지가 이미 해당 스레드에 있었을 때만 스레드 안에서 답변합니다.
- `threadReplies: "always"`는 방 답변을 트리거 메시지를 루트로 하는 스레드에 유지하고, 첫 번째 트리거 메시지부터 해당 대화를 일치하는 스레드 범위 세션으로 라우팅합니다.
- `dm.threadReplies`는 DM에 대해서만 최상위 설정을 재정의합니다. 예를 들어 방 스레드는 분리해 두면서 DM은 평면적으로 유지할 수 있습니다.
- 들어오는 스레드 메시지에는 추가 에이전트 컨텍스트로 스레드 루트 메시지가 포함됩니다.
- 이제 메시지 도구 전송은 명시적인 `threadId`가 제공되지 않는 한, 대상이 동일한 방이거나 동일한 DM 사용자 대상이면 현재 Matrix 스레드를 자동 상속합니다.
- 동일 세션 DM 사용자 대상 재사용은 현재 세션 메타데이터가 동일 Matrix 계정의 동일 DM 상대를 증명할 때만 작동하며, 그렇지 않으면 OpenClaw는 일반 사용자 범위 라우팅으로 되돌아갑니다.
- OpenClaw가 동일한 공유 Matrix DM 세션에서 한 Matrix DM 방이 다른 DM 방과 충돌하는 것을 감지하면, 스레드 바인딩이 활성화되어 있고 `dm.sessionScope` 힌트가 있는 경우 해당 방에 `/focus` 탈출구가 포함된 일회성 `m.notice`를 게시합니다.
- 런타임 스레드 바인딩은 Matrix에서 지원됩니다. 이제 `/focus`, `/unfocus`, `/agents`, `/session idle`, `/session max-age`, 그리고 스레드 바인딩된 `/acp spawn`이 Matrix 방과 DM에서 동작합니다.
- 최상위 Matrix 방/DM의 `/focus`는 `threadBindings.spawnSubagentSessions=true`일 때 새 Matrix 스레드를 만들고 이를 대상 세션에 바인딩합니다.
- 기존 Matrix 스레드 안에서 `/focus` 또는 `/acp spawn --thread here`를 실행하면 현재 스레드를 대신 바인딩합니다.

## ACP 대화 바인딩

Matrix 방, DM, 기존 Matrix 스레드는 채팅 표면을 바꾸지 않고도 지속적인 ACP 작업 공간으로 전환할 수 있습니다.

빠른 운영자 흐름:

- 계속 사용할 Matrix DM, 방, 기존 스레드 안에서 `/acp spawn codex --bind here`를 실행합니다.
- 최상위 Matrix DM 또는 방에서는 현재 DM/방이 채팅 표면으로 유지되고 이후 메시지는 생성된 ACP 세션으로 라우팅됩니다.
- 기존 Matrix 스레드 안에서는 `--bind here`가 현재 스레드를 그 자리에서 바인딩합니다.
- `/new`와 `/reset`은 동일한 바인딩된 ACP 세션을 그 자리에서 재설정합니다.
- `/acp close`는 ACP 세션을 닫고 바인딩을 제거합니다.

참고:

- `--bind here`는 하위 Matrix 스레드를 생성하지 않습니다.
- `threadBindings.spawnAcpSessions`는 OpenClaw가 하위 Matrix 스레드를 생성하거나 바인딩해야 하는 `/acp spawn --thread auto|here`에만 필요합니다.

### 스레드 바인딩 Config

Matrix는 `session.threadBindings`에서 전역 기본값을 상속하며, 채널별 override도 지원합니다:

- `threadBindings.enabled`
- `threadBindings.idleHours`
- `threadBindings.maxAgeHours`
- `threadBindings.spawnSubagentSessions`
- `threadBindings.spawnAcpSessions`

Matrix 스레드 바인딩 생성 플래그는 옵트인입니다:

- 최상위 `/focus`가 새 Matrix 스레드를 생성하고 바인딩하도록 허용하려면 `threadBindings.spawnSubagentSessions: true`를 설정하세요.
- `/acp spawn --thread auto|here`가 ACP 세션을 Matrix 스레드에 바인딩하도록 허용하려면 `threadBindings.spawnAcpSessions: true`를 설정하세요.

## 리액션

Matrix는 아웃바운드 리액션 액션, 인바운드 리액션 알림, 인바운드 ack 리액션을 지원합니다.

- 아웃바운드 리액션 도구는 `channels["matrix"].actions.reactions`로 제어됩니다.
- `react`는 특정 Matrix 이벤트에 리액션을 추가합니다.
- `reactions`는 특정 Matrix 이벤트의 현재 리액션 요약을 나열합니다.
- `emoji=""`는 해당 이벤트에서 봇 계정 자신의 리액션을 제거합니다.
- `remove: true`는 봇 계정의 지정된 이모지 리액션만 제거합니다.

Ack 리액션은 표준 OpenClaw 해석 순서를 사용합니다:

- `channels["matrix"].accounts.<accountId>.ackReaction`
- `channels["matrix"].ackReaction`
- `messages.ackReaction`
- 에이전트 ID 이모지 폴백

Ack 리액션 범위는 다음 순서로 해석됩니다:

- `channels["matrix"].accounts.<accountId>.ackReactionScope`
- `channels["matrix"].ackReactionScope`
- `messages.ackReactionScope`

리액션 알림 모드는 다음 순서로 해석됩니다:

- `channels["matrix"].accounts.<accountId>.reactionNotifications`
- `channels["matrix"].reactionNotifications`
- 기본값: `own`

현재 동작:

- `reactionNotifications: "own"`은 봇이 작성한 Matrix 메시지를 대상으로 하는 추가된 `m.reaction` 이벤트를 전달합니다.
- `reactionNotifications: "off"`는 리액션 시스템 이벤트를 비활성화합니다.
- 리액션 제거는 Matrix가 이를 독립된 `m.reaction` 제거가 아니라 redaction으로 노출하므로 아직 시스템 이벤트로 합성되지 않습니다.

## 기록 컨텍스트

- `channels.matrix.historyLimit`은 Matrix 방 메시지가 에이전트를 트리거할 때 `InboundHistory`로 포함할 최근 방 메시지 수를 제어합니다.
- 이는 `messages.groupChat.historyLimit`로 폴백합니다. 비활성화하려면 `0`으로 설정하세요.
- Matrix 방 기록은 방 전용입니다. DM은 계속 일반 세션 기록을 사용합니다.
- Matrix 방 기록은 pending-only입니다. OpenClaw는 아직 답변을 트리거하지 않은 방 메시지를 버퍼링한 뒤, 멘션 또는 다른 트리거가 들어오면 그 창을 스냅샷합니다.
- 현재 트리거 메시지는 `InboundHistory`에 포함되지 않습니다. 해당 턴의 메인 인바운드 본문에 그대로 유지됩니다.
- 동일한 Matrix 이벤트를 재시도할 때는 더 새로운 방 메시지로 앞으로 밀리지 않고 원래 기록 스냅샷을 재사용합니다.

## 컨텍스트 가시성

Matrix는 가져온 답장 텍스트, 스레드 루트, 보류 중인 기록과 같은 보조 방 컨텍스트에 대해 공통 `contextVisibility` 제어를 지원합니다.

- `contextVisibility: "all"`이 기본값입니다. 보조 컨텍스트는 받은 그대로 유지됩니다.
- `contextVisibility: "allowlist"`는 보조 컨텍스트를 현재 방/사용자 허용 목록 검사에서 허용된 발신자로 필터링합니다.
- `contextVisibility: "allowlist_quote"`는 `allowlist`처럼 동작하지만, 명시적으로 인용된 답장 하나는 계속 유지합니다.

이 설정은 보조 컨텍스트의 가시성에 영향을 주며, 인바운드 메시지 자체가 답변을 트리거할 수 있는지에는 영향을 주지 않습니다.
트리거 권한은 여전히 `groupPolicy`, `groups`, `groupAllowFrom`, DM 정책 설정에서 결정됩니다.

## DM 및 방 정책 예시

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

Matrix DM용 페어링 예시:

```bash
openclaw pairing list matrix
openclaw pairing approve matrix <CODE>
```

승인되지 않은 Matrix 사용자가 승인 전에 계속 메시지를 보낸다면, OpenClaw는 새 코드를 발급하는 대신 동일한 대기 중인 페어링 코드를 재사용하고 짧은 쿨다운 후 다시 리마인더 답장을 보낼 수 있습니다.

공통 DM 페어링 흐름과 저장소 레이아웃은 [Pairing](/ko/channels/pairing)을 참조하세요.

## Exec 승인

Matrix는 Matrix 계정의 exec 승인 클라이언트로 동작할 수 있습니다.

- `channels.matrix.execApprovals.enabled`
- `channels.matrix.execApprovals.approvers` (선택 사항; `channels.matrix.dm.allowFrom`으로 폴백)
- `channels.matrix.execApprovals.target` (`dm` | `channel` | `both`, 기본값: `dm`)
- `channels.matrix.execApprovals.agentFilter`
- `channels.matrix.execApprovals.sessionFilter`

승인자는 `@owner:example.org` 같은 Matrix 사용자 ID여야 합니다. `enabled`가 설정되지 않았거나 `"auto"`이고, `execApprovals.approvers` 또는 `channels.matrix.dm.allowFrom`에서 하나 이상의 승인자를 해석할 수 있으면 Matrix는 네이티브 exec 승인을 자동 활성화합니다. Matrix를 네이티브 승인 클라이언트로 명시적으로 비활성화하려면 `enabled: false`를 설정하세요. 그렇지 않으면 승인 요청은 다른 구성된 승인 경로 또는 exec 승인 폴백 정책으로 대체됩니다.

네이티브 Matrix 라우팅은 현재 exec 전용입니다:

- `channels.matrix.execApprovals.*`는 exec 승인에 대해서만 네이티브 DM/채널 라우팅을 제어합니다.
- plugin 승인은 여전히 공통 same-chat `/approve`와 구성된 `approvals.plugin` 포워딩을 사용합니다.
- Matrix는 승인자를 안전하게 추론할 수 있을 때 plugin 승인 권한 부여에 `channels.matrix.dm.allowFrom`을 재사용할 수 있지만, 별도의 네이티브 plugin 승인 DM/채널 팬아웃 경로를 노출하지는 않습니다.

전송 규칙:

- `target: "dm"`은 승인 프롬프트를 승인자 DM으로 보냅니다
- `target: "channel"`은 프롬프트를 원래 Matrix 방 또는 DM으로 다시 보냅니다
- `target: "both"`는 승인자 DM과 원래 Matrix 방 또는 DM 둘 다로 보냅니다

Matrix 승인 프롬프트는 기본 승인 메시지에 리액션 바로가기를 심습니다:

- `✅` = 한 번 허용
- `❌` = 거부
- `♾️` = 유효한 exec 정책상 가능한 경우 항상 허용

승인자는 해당 메시지에 리액션하거나, 폴백 슬래시 명령 `/approve <id> allow-once`, `/approve <id> allow-always`, 또는 `/approve <id> deny`를 사용할 수 있습니다.

해석된 승인자만 승인 또는 거부할 수 있습니다. 채널 전송은 명령 텍스트를 포함하므로, `channel` 또는 `both`는 신뢰된 방에서만 활성화하세요.

Matrix 승인 프롬프트는 공통 코어 승인 planner를 재사용합니다. Matrix 전용 네이티브 표면은 exec 승인에 대해 전송만 담당합니다: 방/DM 라우팅과 메시지 전송/업데이트/삭제 동작입니다.

계정별 override:

- `channels.matrix.accounts.<account>.execApprovals`

관련 문서: [Exec approvals](/ko/tools/exec-approvals)

## 다중 계정 예시

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

최상위 `channels.matrix` 값은 계정이 이를 override하지 않는 한 이름 있는 계정의 기본값으로 작동합니다.
`groups.<room>.account`(또는 레거시 `rooms.<room>.account`)로 상속된 방 항목을 하나의 Matrix 계정에 범위 지정할 수 있습니다.
`account`가 없는 항목은 모든 Matrix 계정에서 공유되고, `account: "default"`가 있는 항목은 기본 계정이 최상위 `channels.matrix.*`에 직접 구성된 경우에도 계속 동작합니다.
부분적인 공유 인증 기본값만으로는 별도의 암묵적 기본 계정이 생성되지 않습니다. OpenClaw는 해당 기본값에 새 인증(`homeserver` + `accessToken`, 또는 `homeserver` + `userId` + `password`)이 있을 때만 최상위 `default` 계정을 합성합니다. 이름 있는 계정은 나중에 캐시된 자격 증명이 인증 요건을 만족하면 `homeserver` + `userId`만으로도 계속 탐지 가능할 수 있습니다.
Matrix에 이미 정확히 하나의 이름 있는 계정이 있거나 `defaultAccount`가 기존 이름 있는 계정 키를 가리키는 경우, 단일 계정에서 다중 계정으로의 복구/설정 승격은 새 `accounts.default` 항목을 만드는 대신 해당 계정을 보존합니다. Matrix 인증/부트스트랩 키만 이렇게 승격된 계정으로 이동하며, 공유 전송 정책 키는 최상위에 유지됩니다.
암묵적 라우팅, 프로빙, CLI 작업에서 이름 있는 Matrix 계정 하나를 우선 사용하게 하려면 `defaultAccount`를 설정하세요.
이름 있는 계정을 여러 개 구성한 경우, 암묵적 계정 선택에 의존하는 CLI 명령에는 `defaultAccount`를 설정하거나 `--account <id>`를 전달하세요.
한 명령에서 그 암묵적 선택을 override하려면 `openclaw matrix verify ...`와 `openclaw matrix devices ...`에 `--account <id>`를 전달하세요.

## 비공개/LAN 홈서버

기본적으로 OpenClaw는 SSRF 보호를 위해 비공개/내부 Matrix 홈서버를 차단하며,
계정별로 명시적으로 옵트인해야만 허용합니다.

홈서버가 localhost, LAN/Tailscale IP, 또는 내부 호스트명에서 실행된다면,
해당 Matrix 계정에 `allowPrivateNetwork`를 활성화하세요:

```json5
{
  channels: {
    matrix: {
      homeserver: "http://matrix-synapse:8008",
      allowPrivateNetwork: true,
      accessToken: "syt_internal_xxx",
    },
  },
}
```

CLI 설정 예시:

```bash
openclaw matrix account add \
  --account ops \
  --homeserver http://matrix-synapse:8008 \
  --allow-private-network \
  --access-token syt_ops_xxx
```

이 옵트인은 신뢰된 비공개/내부 대상만 허용합니다. 다음과 같은 공개 평문 홈서버는
`http://matrix.example.org:8008`처럼 여전히 차단됩니다. 가능하면 `https://`를 사용하세요.

## Matrix 트래픽 프록시

Matrix 배포에 명시적 아웃바운드 HTTP(S) 프록시가 필요하다면 `channels.matrix.proxy`를 설정하세요:

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

이름 있는 계정은 `channels.matrix.accounts.<id>.proxy`로 최상위 기본값을 override할 수 있습니다.
OpenClaw는 런타임 Matrix 트래픽과 계정 상태 프로브 모두에 동일한 프록시 설정을 사용합니다.

## 대상 해석

OpenClaw가 방 또는 사용자 대상을 요청하는 모든 곳에서 Matrix는 다음 대상 형식을 받아들입니다:

- 사용자: `@user:server`, `user:@user:server`, 또는 `matrix:user:@user:server`
- 방: `!room:server`, `room:!room:server`, 또는 `matrix:room:!room:server`
- 별칭: `#alias:server`, `channel:#alias:server`, 또는 `matrix:channel:#alias:server`

실시간 디렉터리 조회는 로그인된 Matrix 계정을 사용합니다:

- 사용자 조회는 해당 홈서버의 Matrix 사용자 디렉터리를 질의합니다.
- 방 조회는 명시적 방 ID와 별칭을 직접 받아들인 뒤, 해당 계정의 가입된 방 이름 검색으로 폴백합니다.
- 가입된 방 이름 조회는 최선형(best-effort)입니다. 방 이름을 ID 또는 별칭으로 해석할 수 없으면 런타임 허용 목록 해석에서 무시됩니다.

## Configuration reference

- `enabled`: 채널 활성화 또는 비활성화.
- `name`: 계정의 선택적 레이블.
- `defaultAccount`: 여러 Matrix 계정이 구성된 경우 선호되는 계정 ID.
- `homeserver`: 홈서버 URL, 예: `https://matrix.example.org`.
- `allowPrivateNetwork`: 이 Matrix 계정이 비공개/내부 홈서버에 연결되도록 허용합니다. 홈서버가 `localhost`, LAN/Tailscale IP, 또는 `matrix-synapse` 같은 내부 호스트로 해석되는 경우 이를 활성화하세요.
- `proxy`: Matrix 트래픽용 선택적 HTTP(S) 프록시 URL. 이름 있는 계정은 자체 `proxy`로 최상위 기본값을 override할 수 있습니다.
- `userId`: 전체 Matrix 사용자 ID, 예: `@bot:example.org`.
- `accessToken`: 토큰 기반 인증용 액세스 토큰. 일반 텍스트 값과 SecretRef 값은 `channels.matrix.accessToken` 및 `channels.matrix.accounts.<id>.accessToken`에서 env/file/exec provider 전반에 걸쳐 지원됩니다. [Secrets Management](/ko/gateway/secrets)를 참조하세요.
- `password`: 비밀번호 기반 로그인용 비밀번호. 일반 텍스트 값과 SecretRef 값이 지원됩니다.
- `deviceId`: 명시적 Matrix 디바이스 ID.
- `deviceName`: 비밀번호 로그인용 디바이스 표시 이름.
- `avatarUrl`: 프로필 동기화 및 `set-profile` 업데이트용 저장된 자기 아바타 URL.
- `initialSyncLimit`: 시작 시 동기화 이벤트 제한.
- `encryption`: E2EE 활성화.
- `allowlistOnly`: DM과 방에 대해 허용 목록 전용 동작을 강제.
- `allowBots`: 다른 구성된 OpenClaw Matrix 계정의 메시지를 허용(`true` 또는 `"mentions"`).
- `groupPolicy`: `open`, `allowlist`, 또는 `disabled`.
- `contextVisibility`: 보조 방 컨텍스트 가시성 모드(`all`, `allowlist`, `allowlist_quote`).
- `groupAllowFrom`: 방 트래픽에 대한 사용자 ID 허용 목록.
- `groupAllowFrom` 항목은 전체 Matrix 사용자 ID여야 합니다. 해석되지 않은 이름은 런타임에 무시됩니다.
- `historyLimit`: 그룹 기록 컨텍스트에 포함할 최대 방 메시지 수. `messages.groupChat.historyLimit`로 폴백합니다. 비활성화하려면 `0`으로 설정하세요.
- `replyToMode`: `off`, `first`, 또는 `all`.
- `markdown`: 아웃바운드 Matrix 텍스트용 선택적 Markdown 렌더링 구성.
- `streaming`: `off`(기본값), `partial`, `quiet`, `true`, 또는 `false`. `partial`과 `true`는 일반 Matrix 텍스트 메시지로 미리보기 우선 초안 업데이트를 활성화합니다. `quiet`은 자체 호스팅 push-rule 설정용 비알림 미리보기 notice를 사용합니다.
- `blockStreaming`: `true`는 초안 미리보기 스트리밍이 활성 상태일 때 완료된 assistant 블록에 대해 별도의 진행 상황 메시지를 활성화합니다.
- `threadReplies`: `off`, `inbound`, 또는 `always`.
- `threadBindings`: 스레드 바인딩 세션 라우팅 및 수명 주기에 대한 채널별 override.
- `startupVerification`: 시작 시 자동 자기 검증 요청 모드(`if-unverified`, `off`).
- `startupVerificationCooldownHours`: 자동 시작 검증 요청을 다시 시도하기 전의 쿨다운 시간.
- `textChunkLimit`: 아웃바운드 메시지 청크 크기.
- `chunkMode`: `length` 또는 `newline`.
- `responsePrefix`: 아웃바운드 답변용 선택적 메시지 접두사.
- `ackReaction`: 이 채널/계정용 선택적 ack 리액션 override.
- `ackReactionScope`: 선택적 ack 리액션 범위 override(`group-mentions`, `group-all`, `direct`, `all`, `none`, `off`).
- `reactionNotifications`: 인바운드 리액션 알림 모드(`own`, `off`).
- `mediaMaxMb`: Matrix 미디어 처리용 MB 단위 미디어 크기 제한. 아웃바운드 전송과 인바운드 미디어 처리에 적용됩니다.
- `autoJoin`: 초대 자동 참가 정책(`always`, `allowlist`, `off`). 기본값: `off`.
- `autoJoinAllowlist`: `autoJoin`이 `allowlist`일 때 허용되는 방/별칭. 별칭 항목은 초대 처리 중 방 ID로 해석되며, OpenClaw는 초대된 방이 주장하는 별칭 상태를 신뢰하지 않습니다.
- `dm`: DM 정책 블록(`enabled`, `policy`, `allowFrom`, `sessionScope`, `threadReplies`).
- `dm.allowFrom` 항목은 이미 실시간 디렉터리 조회를 통해 해석한 경우가 아니라면 전체 Matrix 사용자 ID여야 합니다.
- `dm.sessionScope`: `per-user`(기본값) 또는 `per-room`. 동일한 상대라도 각 Matrix DM 방이 별도 컨텍스트를 유지하게 하려면 `per-room`을 사용하세요.
- `dm.threadReplies`: DM 전용 스레드 정책 override(`off`, `inbound`, `always`). DM에서의 답변 배치와 세션 분리에 대해 최상위 `threadReplies` 설정을 override합니다.
- `execApprovals`: Matrix 네이티브 exec 승인 전송(`enabled`, `approvers`, `target`, `agentFilter`, `sessionFilter`).
- `execApprovals.approvers`: exec 요청을 승인할 수 있는 Matrix 사용자 ID. `dm.allowFrom`이 이미 승인자를 식별하는 경우 선택 사항입니다.
- `execApprovals.target`: `dm | channel | both` (기본값: `dm`).
- `accounts`: 이름 있는 계정별 override. 최상위 `channels.matrix` 값은 이 항목들의 기본값으로 작동합니다.
- `groups`: 방별 정책 맵. 방 ID 또는 별칭 사용을 권장하며, 해석되지 않은 방 이름은 런타임에 무시됩니다. 세션/그룹 ID는 해석 후 안정적인 방 ID를 사용하며, 사람이 읽는 레이블은 여전히 방 이름에서 옵니다.
- `groups.<room>.account`: 다중 계정 설정에서 상속된 하나의 방 항목을 특정 Matrix 계정으로 제한합니다.
- `groups.<room>.allowBots`: 구성된 봇 발신자에 대한 방 수준 override(`true` 또는 `"mentions"`).
- `groups.<room>.users`: 방별 발신자 허용 목록.
- `groups.<room>.tools`: 방별 도구 허용/거부 override.
- `groups.<room>.autoReply`: 방 수준 멘션 게이팅 override. `true`는 해당 방의 멘션 요구 사항을 비활성화하고, `false`는 이를 다시 강제로 활성화합니다.
- `groups.<room>.skills`: 선택적 방 수준 Skills 필터.
- `groups.<room>.systemPrompt`: 선택적 방 수준 system prompt 스니펫.
- `rooms`: `groups`의 레거시 별칭.
- `actions`: 액션별 도구 게이팅(`messages`, `reactions`, `pins`, `profile`, `memberInfo`, `channelInfo`, `verification`).

## 관련 항목

- [Channels Overview](/ko/channels) — 지원되는 모든 채널
- [Pairing](/ko/channels/pairing) — DM 인증 및 페어링 흐름
- [Groups](/ko/channels/groups) — 그룹 채팅 동작 및 멘션 게이팅
- [Channel Routing](/ko/channels/channel-routing) — 메시지용 세션 라우팅
- [Security](/ko/gateway/security) — 접근 모델 및 강화
