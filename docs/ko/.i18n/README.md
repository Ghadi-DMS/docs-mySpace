---
x-i18n:
    generated_at: "2026-04-06T03:05:51Z"
    model: gpt-5.4
    provider: openai
    source_hash: 6e1cf417b0c04d001bc494fbe03ac2fcb66866f759e21646dbfd1a9c3a968bff
    source_path: .i18n/README.md
    workflow: 15
---

# OpenClaw 문서 i18n 자산

이 폴더에는 소스 문서 저장소용 번역 구성이 저장됩니다.

생성된 로케일 페이지와 라이브 로케일 번역 메모리는 이제 게시 저장소(`openclaw/docs`, 로컬 형제 체크아웃 `~/Projects/openclaw-docs`)에 있습니다.

## 파일

- `glossary.<lang>.json` — 선호 용어 매핑(프롬프트 안내에 사용됨)
- `<lang>.tm.jsonl` — 워크플로 + 모델 + 텍스트 해시를 기준으로 한 번역 메모리(캐시). 이 저장소에서는 로케일 TM 파일이 필요할 때 생성됩니다.

## 용어집 형식

`glossary.<lang>.json`은 항목 배열입니다.

```json
{
  "source": "troubleshooting",
  "target": "故障排除",
  "ignore_case": true,
  "whole_word": false
}
```

필드:

- `source`: 선호할 영어(또는 원본) 구문
- `target`: 선호 번역 출력

## 참고

- 용어집 항목은 모델에 **프롬프트 안내**로 전달됩니다(결정적 재작성 없음).
- `scripts/docs-i18n`는 여전히 번역 생성의 소유자입니다.
- 소스 저장소는 영어 문서를 게시 저장소에 동기화하며, 로케일 생성은 푸시, 예약 실행, 릴리스 디스패치 시 로케일별로 그곳에서 실행됩니다.
