---
read_when:
    - تريد استخدام DeepSeek مع OpenClaw
    - تحتاج إلى متغير env الخاص بمفتاح API أو خيار المصادقة في CLI
summary: إعداد DeepSeek ‏(المصادقة + اختيار النموذج)
x-i18n:
    generated_at: "2026-04-05T12:52:56Z"
    model: gpt-5.4
    provider: openai
    source_hash: 35f339ca206399496ce094eb8350e0870029ce9605121bcf86c4e9b94f3366c6
    source_path: providers/deepseek.md
    workflow: 15
---

# DeepSeek

توفّر [DeepSeek](https://www.deepseek.com) نماذج ذكاء اصطناعي قوية عبر واجهة API متوافقة مع OpenAI.

- الموفّر: `deepseek`
- المصادقة: `DEEPSEEK_API_KEY`
- API: متوافقة مع OpenAI
- Base URL: `https://api.deepseek.com`

## بدء سريع

اضبط مفتاح API ‏(يوصى بتخزينه من أجل Gateway):

```bash
openclaw onboard --auth-choice deepseek-api-key
```

سيطلب منك ذلك مفتاح API الخاص بك ويضبط `deepseek/deepseek-chat` بوصفه النموذج الافتراضي.

## مثال غير تفاعلي

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice deepseek-api-key \
  --deepseek-api-key "$DEEPSEEK_API_KEY" \
  --skip-health \
  --accept-risk
```

## ملاحظة حول البيئة

إذا كانت Gateway تعمل بوصفها daemon ‏(`launchd/systemd`)، فتأكد من أن `DEEPSEEK_API_KEY`
متاح لتلك العملية (على سبيل المثال، في `~/.openclaw/.env` أو عبر
`env.shellEnv`).

## الفهرس المدمج

| مرجع النموذج | الاسم | الإدخال | السياق | الحد الأقصى للإخراج | ملاحظات |
| ---------------------------- | ----------------- | ----- | ------- | ---------- | ------------------------------------------------- |
| `deepseek/deepseek-chat` | DeepSeek Chat | نص | 131,072 | 8,192 | النموذج الافتراضي؛ واجهة DeepSeek V3.2 غير الخاصة بالتفكير |
| `deepseek/deepseek-reasoner` | DeepSeek Reasoner | نص | 131,072 | 65,536 | واجهة V3.2 المفعلة للتفكير |

يعلن كلا النموذجين المدمجين حاليًا عن توافق استخدام البث في المصدر.

احصل على مفتاح API الخاص بك من [platform.deepseek.com](https://platform.deepseek.com/api_keys).
