---
read_when:
    - تريد خطوة LLM بتنسيق JSON فقط داخل سير العمل
    - تحتاج إلى مخرجات LLM متحقق منها بالمخطط لأغراض الأتمتة
summary: مهام LLM بتنسيق JSON فقط لسير العمل (أداة plugin اختيارية)
title: LLM Task
x-i18n:
    generated_at: "2026-04-05T12:58:53Z"
    model: gpt-5.4
    provider: openai
    source_hash: cbe9b286a8e958494de06a59b6e7b750a82d492158df344c7afe30fce24f0584
    source_path: tools/llm-task.md
    workflow: 15
---

# LLM Task

تُعد `llm-task` **أداة plugin اختيارية** تشغّل مهمة LLM بتنسيق JSON فقط
وتعيد مخرجات منظَّمة (مع تحقق اختياري وفق JSON Schema).

وهذا مثالي لمحركات سير العمل مثل Lobster: إذ يمكنك إضافة خطوة LLM واحدة
من دون كتابة شيفرة OpenClaw مخصصة لكل سير عمل.

## تفعيل plugin

1. فعّل plugin:

```json
{
  "plugins": {
    "entries": {
      "llm-task": { "enabled": true }
    }
  }
}
```

2. أضف الأداة إلى قائمة السماح (فهي تُسجَّل مع `optional: true`):

```json
{
  "agents": {
    "list": [
      {
        "id": "main",
        "tools": { "allow": ["llm-task"] }
      }
    ]
  }
}
```

## الإعدادات (اختياري)

```json
{
  "plugins": {
    "entries": {
      "llm-task": {
        "enabled": true,
        "config": {
          "defaultProvider": "openai-codex",
          "defaultModel": "gpt-5.4",
          "defaultAuthProfileId": "main",
          "allowedModels": ["openai-codex/gpt-5.4"],
          "maxTokens": 800,
          "timeoutMs": 30000
        }
      }
    }
  }
}
```

تُعد `allowedModels` قائمة سماح من سلاسل `provider/model`. وإذا كانت مضبوطة،
فسيتم رفض أي طلب خارج هذه القائمة.

## معلمات الأداة

- `prompt` ‏(سلسلة، مطلوب)
- `input` ‏(أي نوع، اختياري)
- `schema` ‏(كائن، JSON Schema اختياري)
- `provider` ‏(سلسلة، اختياري)
- `model` ‏(سلسلة، اختياري)
- `thinking` ‏(سلسلة، اختياري)
- `authProfileId` ‏(سلسلة، اختياري)
- `temperature` ‏(رقم، اختياري)
- `maxTokens` ‏(رقم، اختياري)
- `timeoutMs` ‏(رقم، اختياري)

تقبل `thinking` إعدادات التفكير المسبقة القياسية في OpenClaw، مثل `low` أو `medium`.

## المخرجات

تعيد `details.json` التي تحتوي على JSON الذي تم تحليله (ويتم التحقق منه وفق
`schema` عند توفيره).

## مثال: خطوة في سير عمل Lobster

```lobster
openclaw.invoke --tool llm-task --action json --args-json '{
  "prompt": "Given the input email, return intent and draft.",
  "thinking": "low",
  "input": {
    "subject": "Hello",
    "body": "Can you help?"
  },
  "schema": {
    "type": "object",
    "properties": {
      "intent": { "type": "string" },
      "draft": { "type": "string" }
    },
    "required": ["intent", "draft"],
    "additionalProperties": false
  }
}'
```

## ملاحظات الأمان

- الأداة **JSON-only** وتوجّه النموذج لإخراج JSON فقط (من دون
  أسوار شيفرة، ومن دون تعليق).
- لا يتم كشف أي أدوات للنموذج في هذا التشغيل.
- تعامل مع المخرجات على أنها غير موثوقة ما لم تتحقق منها باستخدام `schema`.
- ضع الموافقات قبل أي خطوة ذات تأثير جانبي (إرسال، نشر، تنفيذ).
