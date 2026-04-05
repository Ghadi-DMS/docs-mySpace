---
read_when:
    - تريد استخدام Fireworks مع OpenClaw
    - تحتاج إلى متغير env الخاص بـ Fireworks API key أو معرّف النموذج الافتراضي
summary: إعداد Fireworks ‏(المصادقة + اختيار النموذج)
x-i18n:
    generated_at: "2026-04-05T12:52:58Z"
    model: gpt-5.4
    provider: openai
    source_hash: 20083d5c248abd9a7223e6d188f0265ae27381940ee0067dff6d1d46d908c552
    source_path: providers/fireworks.md
    workflow: 15
---

# Fireworks

توفّر [Fireworks](https://fireworks.ai) نماذج مفتوحة الأوزان ونماذج موجّهة عبر API متوافقة مع OpenAI. يتضمن OpenClaw الآن مكوّن Fireworks إضافيًا مضمّنًا للمزوّد.

- المزوّد: `fireworks`
- المصادقة: `FIREWORKS_API_KEY`
- API: chat/completions متوافقة مع OpenAI
- Base URL: `https://api.fireworks.ai/inference/v1`
- النموذج الافتراضي: `fireworks/accounts/fireworks/routers/kimi-k2p5-turbo`

## بدء سريع

اضبط مصادقة Fireworks عبر الإعداد الأولي:

```bash
openclaw onboard --auth-choice fireworks-api-key
```

يخزن هذا مفتاح Fireworks الخاص بك في إعداد OpenClaw ويضبط نموذج Fire Pass المبدئي كنموذج افتراضي.

## مثال غير تفاعلي

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice fireworks-api-key \
  --fireworks-api-key "$FIREWORKS_API_KEY" \
  --skip-health \
  --accept-risk
```

## ملاحظة حول البيئة

إذا كانت Gateway تعمل خارج shell التفاعلية الخاصة بك، فتأكد من أن `FIREWORKS_API_KEY`
متاح لتلك العملية أيضًا. فالمفتاح الموجود فقط في `~/.profile` لن
يفيد daemon تعمل عبر launchd/systemd ما لم تُستورد تلك البيئة هناك أيضًا.

## الفهرس المضمّن

| مرجع النموذج                                            | الاسم                        | الإدخال     | السياق  | أقصى إخراج | ملاحظات                                       |
| ------------------------------------------------------- | ---------------------------- | ----------- | ------- | ---------- | --------------------------------------------- |
| `fireworks/accounts/fireworks/routers/kimi-k2p5-turbo` | Kimi K2.5 Turbo ‏(Fire Pass) | text,image  | 256,000 | 256,000    | النموذج الابتدائي المضمّن الافتراضي على Fireworks |

## معرّفات نماذج Fireworks المخصصة

يقبل OpenClaw أيضًا معرّفات نماذج Fireworks الديناميكية. استخدم معرّف النموذج أو router المطابق تمامًا كما تعرضه Fireworks، وأضف قبله البادئة `fireworks/`.

مثال:

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "fireworks/accounts/fireworks/routers/kimi-k2p5-turbo",
      },
    },
  },
}
```

إذا نشرت Fireworks نموذجًا أحدث مثل إصدار جديد من Qwen أو Gemma، فيمكنك التبديل إليه مباشرةً باستخدام معرّف نموذج Fireworks الخاص به من دون انتظار تحديث الفهرس المضمّن.
