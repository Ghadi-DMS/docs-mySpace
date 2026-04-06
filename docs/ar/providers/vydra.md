---
read_when:
    - أنت تريد توليد الوسائط عبر Vydra في OpenClaw
    - أنت بحاجة إلى إرشادات إعداد مفتاح API لـ Vydra
summary: استخدم الصور والفيديو والكلام عبر Vydra في OpenClaw
title: Vydra
x-i18n:
    generated_at: "2026-04-06T03:11:56Z"
    model: gpt-5.4
    provider: openai
    source_hash: 0fe999e8a5414b8a31a6d7d127bc6bcfc3b4492b8f438ab17dfa9680c5b079b7
    source_path: providers/vydra.md
    workflow: 15
---

# Vydra

تضيف إضافة Vydra المضمّنة ما يلي:

- توليد الصور عبر `vydra/grok-imagine`
- توليد الفيديو عبر `vydra/veo3` و `vydra/kling`
- توليف الكلام عبر مسار TTS الخاص بـ Vydra والمدعوم من ElevenLabs

يستخدم OpenClaw المفتاح نفسه `VYDRA_API_KEY` للقدرات الثلاث جميعها.

## عنوان URL الأساسي المهم

استخدم `https://www.vydra.ai/api/v1`.

يقوم المضيف الجذري لـ Vydra (`https://vydra.ai/api/v1`) حاليًا بإعادة التوجيه إلى `www`. بعض عملاء HTTP يسقطون `Authorization` عند إعادة التوجيه هذه بين المضيفين، مما يحول مفتاح API صالحًا إلى فشل مصادقة مضلل. تستخدم الإضافة المضمّنة عنوان URL الأساسي `www` مباشرة لتجنب ذلك.

## الإعداد

الإعداد الأولي التفاعلي:

```bash
openclaw onboard --auth-choice vydra-api-key
```

أو عيّن متغير البيئة مباشرة:

```bash
export VYDRA_API_KEY="vydra_live_..."
```

## توليد الصور

نموذج الصور الافتراضي:

- `vydra/grok-imagine`

عيّنه كموفّر الصور الافتراضي:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "vydra/grok-imagine",
      },
    },
  },
}
```

يقتصر الدعم المضمّن الحالي على النص إلى صورة فقط. تتوقع مسارات التحرير المستضافة في Vydra عناوين URL لصور بعيدة، ولم يضف OpenClaw بعد جسر رفع خاصًا بـ Vydra في الإضافة المضمّنة.

راجع [توليد الصور](/ar/tools/image-generation) للاطلاع على سلوك الأداة المشتركة.

## توليد الفيديو

نماذج الفيديو المسجلة:

- `vydra/veo3` للنص إلى فيديو
- `vydra/kling` للصورة إلى فيديو

عيّن Vydra كموفّر الفيديو الافتراضي:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "vydra/veo3",
      },
    },
  },
}
```

ملاحظات:

- يتم تضمين `vydra/veo3` كنموذج نص إلى فيديو فقط.
- يتطلب `vydra/kling` حاليًا مرجع عنوان URL لصورة بعيدة. ويتم رفض عمليات رفع الملفات المحلية مسبقًا.
- تظل الإضافة المضمّنة محافظة ولا تمرر خيارات نمط غير موثقة مثل نسبة الأبعاد أو الدقة أو العلامة المائية أو الصوت المُولَّد.

راجع [توليد الفيديو](/tools/video-generation) للاطلاع على سلوك الأداة المشتركة.

## توليف الكلام

عيّن Vydra كموفّر الكلام:

```json5
{
  messages: {
    tts: {
      provider: "vydra",
      providers: {
        vydra: {
          apiKey: "${VYDRA_API_KEY}",
          voiceId: "21m00Tcm4TlvDq8ikWAM",
        },
      },
    },
  },
}
```

القيم الافتراضية:

- النموذج: `elevenlabs/tts`
- معرّف الصوت: `21m00Tcm4TlvDq8ikWAM`

تعرض الإضافة المضمّنة حاليًا صوتًا افتراضيًا واحدًا معروفًا بأنه جيد وتعيد ملفات صوت MP3.

## ذو صلة

- [دليل الموفّرين](/ar/providers/index)
- [توليد الصور](/ar/tools/image-generation)
- [توليد الفيديو](/tools/video-generation)
