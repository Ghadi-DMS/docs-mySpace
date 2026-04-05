---
read_when:
    - تريد كتالوج OpenCode Go
    - تحتاج إلى model refs الخاصة بوقت التشغيل للنماذج المستضافة عبر Go
summary: استخدم كتالوج OpenCode Go مع إعداد OpenCode المشترك
title: OpenCode Go
x-i18n:
    generated_at: "2026-04-05T12:53:42Z"
    model: gpt-5.4
    provider: openai
    source_hash: 8650af7c64220c14bab8c22472fff8bebd7abde253e972b6a11784ad833d321c
    source_path: providers/opencode-go.md
    workflow: 15
---

# OpenCode Go

OpenCode Go هو كتالوج Go ضمن [OpenCode](/providers/opencode).
ويستخدم مفتاح `OPENCODE_API_KEY` نفسه مثل كتالوج Zen، لكنه يحتفظ
بمعرّف مزوّد وقت التشغيل `opencode-go` حتى يبقى التوجيه لكل نموذج من المصدر صحيحًا.

## النماذج المدعومة

- `opencode-go/kimi-k2.5`
- `opencode-go/glm-5`
- `opencode-go/minimax-m2.5`

## إعداد CLI

```bash
openclaw onboard --auth-choice opencode-go
# أو بشكل غير تفاعلي
openclaw onboard --opencode-go-api-key "$OPENCODE_API_KEY"
```

## مقتطف تكوين

```json5
{
  env: { OPENCODE_API_KEY: "YOUR_API_KEY_HERE" }, // pragma: allowlist secret
  agents: { defaults: { model: { primary: "opencode-go/kimi-k2.5" } } },
}
```

## سلوك التوجيه

يتولى OpenClaw التوجيه لكل نموذج تلقائيًا عندما يستخدم model ref الصيغة `opencode-go/...`.

## ملاحظات

- استخدم [OpenCode](/providers/opencode) للاطلاع على onboarding المشتركة ونظرة عامة على الكتالوج.
- تظل مراجع وقت التشغيل صريحة: `opencode/...` لـ Zen، و`opencode-go/...` لـ Go.
