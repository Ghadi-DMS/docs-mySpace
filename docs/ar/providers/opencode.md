---
read_when:
    - تريد وصولًا إلى نماذج مستضافة على OpenCode
    - تريد الاختيار بين فهرسي Zen وGo
summary: استخدم فهارس OpenCode Zen وGo مع OpenClaw
title: OpenCode
x-i18n:
    generated_at: "2026-04-05T12:53:34Z"
    model: gpt-5.4
    provider: openai
    source_hash: c23bc99208d9275afcb1731c28eee250c9f4b7d0578681ace31416135c330865
    source_path: providers/opencode.md
    workflow: 15
---

# OpenCode

تعرض OpenCode فهرسين مستضافين في OpenClaw:

- `opencode/...` لفهرس **Zen**
- `opencode-go/...` لفهرس **Go**

يستخدم كلا الفهرسين مفتاح OpenCode API نفسه. ويُبقي OpenClaw معرّفات مزود وقت التشغيل
منفصلة بحيث يبقى التوجيه الأعلى لكل نموذج صحيحًا، لكن الإعداد الأولي والوثائق يعاملانها
بوصفها إعداد OpenCode واحدًا.

## إعداد CLI

### فهرس Zen

```bash
openclaw onboard --auth-choice opencode-zen
openclaw onboard --opencode-zen-api-key "$OPENCODE_API_KEY"
```

### فهرس Go

```bash
openclaw onboard --auth-choice opencode-go
openclaw onboard --opencode-go-api-key "$OPENCODE_API_KEY"
```

## مقتطف إعداد

```json5
{
  env: { OPENCODE_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "opencode/claude-opus-4-6" } } },
}
```

## الفهارس

### Zen

- مزود وقت التشغيل: `opencode`
- أمثلة النماذج: `opencode/claude-opus-4-6`, `opencode/gpt-5.4`, `opencode/gemini-3-pro`
- الأنسب عندما تريد OpenCode multi-model proxy منسقًا

### Go

- مزود وقت التشغيل: `opencode-go`
- أمثلة النماذج: `opencode-go/kimi-k2.5`, `opencode-go/glm-5`, `opencode-go/minimax-m2.5`
- الأنسب عندما تريد التشكيلة المستضافة على OpenCode من Kimi/GLM/MiniMax

## ملاحظات

- `OPENCODE_ZEN_API_KEY` مدعوم أيضًا.
- يؤدي إدخال مفتاح OpenCode واحد أثناء الإعداد إلى تخزين بيانات الاعتماد لكلا مزودي وقت التشغيل.
- تقوم بتسجيل الدخول إلى OpenCode، وتضيف تفاصيل الفوترة، ثم تنسخ API key الخاصة بك.
- تتم إدارة الفوترة وتوفر الفهرس من لوحة تحكم OpenCode.
- تبقى مراجع OpenCode المدعومة من Gemini على مسار proxy-Gemini، لذلك يحتفظ OpenClaw
  هناك بتنقية Gemini thought-signature من دون تفعيل التحقق الأصلي من
  إعادة تشغيل Gemini أو إعادات كتابة bootstrap.
- تحتفظ مراجع OpenCode غير Gemini بسياسة إعادة التشغيل الدنيا المتوافقة مع OpenAI.
