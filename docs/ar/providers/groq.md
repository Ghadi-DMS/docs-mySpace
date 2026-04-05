---
read_when:
    - تريد استخدام Groq مع OpenClaw
    - تحتاج إلى متغير env الخاص بمفتاح API أو خيار المصادقة في CLI
summary: إعداد Groq ‏(المصادقة + اختيار النموذج)
title: Groq
x-i18n:
    generated_at: "2026-04-05T12:53:06Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7e27532cafcdaf1ac336fa310e08e4e3245d2d0eb0e94e0bcf42c532c6a9a80b
    source_path: providers/groq.md
    workflow: 15
---

# Groq

توفّر [Groq](https://groq.com) استدلالًا فائق السرعة على النماذج مفتوحة المصدر
(Llama وGemma وMistral وغير ذلك) باستخدام عتاد LPU مخصص. ويتصل OpenClaw
بـ Groq عبر واجهة API المتوافقة مع OpenAI.

- الموفّر: `groq`
- المصادقة: `GROQ_API_KEY`
- API: متوافقة مع OpenAI

## بدء سريع

1. احصل على مفتاح API من [console.groq.com/keys](https://console.groq.com/keys).

2. اضبط مفتاح API:

```bash
export GROQ_API_KEY="gsk_..."
```

3. اضبط نموذجًا افتراضيًا:

```json5
{
  agents: {
    defaults: {
      model: { primary: "groq/llama-3.3-70b-versatile" },
    },
  },
}
```

## مثال على ملف الإعداد

```json5
{
  env: { GROQ_API_KEY: "gsk_..." },
  agents: {
    defaults: {
      model: { primary: "groq/llama-3.3-70b-versatile" },
    },
  },
}
```

## النسخ الصوتي

توفّر Groq أيضًا نسخًا صوتيًا سريعًا قائمًا على Whisper. وعند تهيئتها بوصفها
موفّرًا لفهم الوسائط، يستخدم OpenClaw نموذج `whisper-large-v3-turbo` من Groq
لنسخ الرسائل الصوتية عبر الواجهة المشتركة `tools.media.audio`.

```json5
{
  tools: {
    media: {
      audio: {
        models: [{ provider: "groq" }],
      },
    },
  },
}
```

## ملاحظة حول البيئة

إذا كانت Gateway تعمل بوصفها daemon ‏(`launchd/systemd`)، فتأكد من أن `GROQ_API_KEY` متاح
لتلك العملية (على سبيل المثال، في `~/.openclaw/.env` أو عبر
`env.shellEnv`).

## ملاحظات الصوت

- مسار الإعداد المشترك: `tools.media.audio`
- عنوان base URL الافتراضي لصوت Groq: ‏`https://api.groq.com/openai/v1`
- نموذج الصوت الافتراضي في Groq: ‏`whisper-large-v3-turbo`
- يستخدم النسخ الصوتي في Groq المسار المتوافق مع OpenAI ‏`/audio/transcriptions`

## النماذج المتاحة

يتغير فهرس نماذج Groq كثيرًا. شغّل `openclaw models list | grep groq`
لرؤية النماذج المتاحة حاليًا، أو راجع
[console.groq.com/docs/models](https://console.groq.com/docs/models).

تشمل الخيارات الشائعة:

- **Llama 3.3 70B Versatile** — للأغراض العامة، مع سياق كبير
- **Llama 3.1 8B Instant** — سريع وخفيف
- **Gemma 2 9B** — مدمج وفعّال
- **Mixtral 8x7B** — بنية MoE، مع قدرة استدلال قوية

## الروابط

- [Groq Console](https://console.groq.com)
- [وثائق API](https://console.groq.com/docs)
- [قائمة النماذج](https://console.groq.com/docs/models)
- [الأسعار](https://groq.com/pricing)
