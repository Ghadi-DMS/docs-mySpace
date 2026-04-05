---
read_when:
    - تريد استخدام Deepgram لتحويل الكلام إلى نص للمرفقات الصوتية
    - تحتاج إلى مثال سريع على تكوين Deepgram
summary: نسخ Deepgram للصوت الوارد
title: Deepgram
x-i18n:
    generated_at: "2026-04-05T12:52:52Z"
    model: gpt-5.4
    provider: openai
    source_hash: dabd1f6942c339fbd744fbf38040b6a663b06ddf4d9c9ee31e3ac034de9e79d9
    source_path: providers/deepgram.md
    workflow: 15
---

# Deepgram ‏(نسخ الصوت)

Deepgram هي واجهة API لتحويل الكلام إلى نص. وفي OpenClaw تُستخدم من أجل **نسخ الملاحظات الصوتية/الصوت الوارد**
عبر `tools.media.audio`.

عند التمكين، يرفع OpenClaw ملف الصوت إلى Deepgram ويحقن النص المنسوخ
في مسار الرد (`{{Transcript}}` + كتلة `[Audio]`). وهذا **ليس بثًا مباشرًا**؛
إذ يستخدم نقطة نهاية النسخ الخاصة بالتسجيلات المسبقة.

الموقع: [https://deepgram.com](https://deepgram.com)  
الوثائق: [https://developers.deepgram.com](https://developers.deepgram.com)

## البدء السريع

1. عيّن مفتاح API الخاص بك:

```
DEEPGRAM_API_KEY=dg_...
```

2. فعّل المزوّد:

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        models: [{ provider: "deepgram", model: "nova-3" }],
      },
    },
  },
}
```

## الخيارات

- `model`: معرّف نموذج Deepgram ‏(الافتراضي: `nova-3`)
- `language`: تلميح اللغة (اختياري)
- `tools.media.audio.providerOptions.deepgram.detect_language`: تمكين اكتشاف اللغة (اختياري)
- `tools.media.audio.providerOptions.deepgram.punctuate`: تمكين علامات الترقيم (اختياري)
- `tools.media.audio.providerOptions.deepgram.smart_format`: تمكين التنسيق الذكي (اختياري)

مثال مع اللغة:

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        models: [{ provider: "deepgram", model: "nova-3", language: "en" }],
      },
    },
  },
}
```

مثال مع خيارات Deepgram:

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        providerOptions: {
          deepgram: {
            detect_language: true,
            punctuate: true,
            smart_format: true,
          },
        },
        models: [{ provider: "deepgram", model: "nova-3" }],
      },
    },
  },
}
```

## ملاحظات

- تتبع المصادقة ترتيب مصادقة المزوّد القياسي؛ ويُعد `DEEPGRAM_API_KEY` أبسط مسار.
- تجاوز نقاط النهاية أو الرؤوس باستخدام `tools.media.audio.baseUrl` و`tools.media.audio.headers` عند استخدام proxy.
- يتبع الخرج قواعد الصوت نفسها الخاصة بالمزوّدين الآخرين (حدود الحجم، والمهلات، وحقن النص المنسوخ).
