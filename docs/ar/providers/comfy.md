---
read_when:
    - أنت تريد استخدام مسارات عمل ComfyUI المحلية مع OpenClaw
    - أنت تريد استخدام Comfy Cloud مع مسارات عمل الصور أو الفيديو أو الموسيقى
    - أنت بحاجة إلى مفاتيح إعدادات إضافة `comfy` المضمّنة
summary: إعداد توليد الصور والفيديو والموسيقى عبر سير عمل ComfyUI في OpenClaw
title: ComfyUI
x-i18n:
    generated_at: "2026-04-06T03:11:34Z"
    model: gpt-5.4
    provider: openai
    source_hash: e645f32efdffdf4cd498684f1924bb953a014d3656b48f4b503d64e38c61ba9c
    source_path: providers/comfy.md
    workflow: 15
---

# ComfyUI

يشحن OpenClaw إضافة مضمّنة باسم `comfy` لتشغيلات ComfyUI المعتمدة على مسارات العمل.

- الموفّر: `comfy`
- النماذج: `comfy/workflow`
- الأسطح المشتركة: `image_generate` و `video_generate` و `music_generate`
- المصادقة: لا شيء لـ ComfyUI المحلي؛ و`COMFY_API_KEY` أو `COMFY_CLOUD_API_KEY` لـ Comfy Cloud
- API: ‏ComfyUI ‏`/prompt` / `/history` / `/view` و Comfy Cloud ‏`/api/*`

## ما الذي يدعمه

- توليد الصور من ملف JSON لمسار عمل
- تحرير الصور باستخدام صورة مرجعية واحدة مرفوعة
- توليد الفيديو من ملف JSON لمسار عمل
- توليد الفيديو باستخدام صورة مرجعية واحدة مرفوعة
- توليد الموسيقى أو الصوت عبر أداة `music_generate` المشتركة
- تنزيل المخرجات من عقدة مهيأة أو من جميع عقد المخرجات المطابقة

تعتمد الإضافة المضمّنة على مسارات العمل، لذلك لا يحاول OpenClaw ربط
عناصر التحكم العامة مثل `size` أو `aspectRatio` أو `resolution` أو `durationSeconds` أو
عناصر التحكم على نمط TTS بالرسم البياني الخاص بك.

## بنية الإعدادات

يدعم Comfy إعدادات اتصال عليا مشتركة بالإضافة إلى أقسام مسارات عمل
لكل قدرة:

```json5
{
  models: {
    providers: {
      comfy: {
        mode: "local",
        baseUrl: "http://127.0.0.1:8188",
        image: {
          workflowPath: "./workflows/flux-api.json",
          promptNodeId: "6",
          outputNodeId: "9",
        },
        video: {
          workflowPath: "./workflows/video-api.json",
          promptNodeId: "12",
          outputNodeId: "21",
        },
        music: {
          workflowPath: "./workflows/music-api.json",
          promptNodeId: "3",
          outputNodeId: "18",
        },
      },
    },
  },
}
```

المفاتيح المشتركة:

- `mode`: ‏`local` أو `cloud`
- `baseUrl`: تكون افتراضيًا `http://127.0.0.1:8188` للوضع المحلي أو `https://cloud.comfy.org` للوضع السحابي
- `apiKey`: بديل اختياري للمفتاح المضمّن بدلًا من متغيرات البيئة
- `allowPrivateNetwork`: السماح بعنوان `baseUrl` خاص/ضمن LAN في وضع السحابة

المفاتيح الخاصة بكل قدرة ضمن `image` أو `video` أو `music`:

- `workflow` أو `workflowPath`: مطلوب
- `promptNodeId`: مطلوب
- `promptInputName`: القيمة الافتراضية هي `text`
- `outputNodeId`: اختياري
- `pollIntervalMs`: اختياري
- `timeoutMs`: اختياري

تدعم أقسام الصور والفيديو أيضًا:

- `inputImageNodeId`: مطلوب عند تمرير صورة مرجعية
- `inputImageInputName`: القيمة الافتراضية هي `image`

## التوافق مع الإصدارات السابقة

لا يزال إعداد الصور الحالي على المستوى الأعلى يعمل:

```json5
{
  models: {
    providers: {
      comfy: {
        workflowPath: "./workflows/flux-api.json",
        promptNodeId: "6",
        outputNodeId: "9",
      },
    },
  },
}
```

يتعامل OpenClaw مع هذا الشكل القديم على أنه إعداد مسار عمل الصور.

## مسارات عمل الصور

عيّن نموذج الصور الافتراضي:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "comfy/workflow",
      },
    },
  },
}
```

مثال على التحرير باستخدام صورة مرجعية:

```json5
{
  models: {
    providers: {
      comfy: {
        image: {
          workflowPath: "./workflows/edit-api.json",
          promptNodeId: "6",
          inputImageNodeId: "7",
          inputImageInputName: "image",
          outputNodeId: "9",
        },
      },
    },
  },
}
```

## مسارات عمل الفيديو

عيّن نموذج الفيديو الافتراضي:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "comfy/workflow",
      },
    },
  },
}
```

تدعم مسارات عمل الفيديو في Comfy حاليًا تحويل النص إلى فيديو وتحويل الصورة إلى فيديو عبر
الرسم البياني المهيأ. ولا يمرر OpenClaw مقاطع الفيديو المدخلة إلى مسارات عمل Comfy.

## مسارات عمل الموسيقى

تسجل الإضافة المضمّنة موفرًا لتوليد الموسيقى للمخرجات
الصوتية أو الموسيقية المحددة في مسار العمل، والتي تظهر عبر أداة `music_generate` المشتركة:

```text
/tool music_generate prompt="Warm ambient synth loop with soft tape texture"
```

استخدم قسم إعداد `music` للإشارة إلى ملف JSON لمسار عمل الصوت لديك وعقدة
المخرجات.

## Comfy Cloud

استخدم `mode: "cloud"` بالإضافة إلى أحد الخيارات التالية:

- `COMFY_API_KEY`
- `COMFY_CLOUD_API_KEY`
- `models.providers.comfy.apiKey`

لا يزال وضع السحابة يستخدم أقسام مسارات العمل نفسها `image` و`video` و`music`.

## الاختبارات الحية

توجد تغطية حية اختيارية للإضافة المضمّنة:

```bash
OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts
```

يتجاوز الاختبار الحي حالات الصور أو الفيديو أو الموسيقى الفردية ما لم يكن
قسم مسار عمل Comfy المطابق مهيأ.

## ذو صلة

- [توليد الصور](/ar/tools/image-generation)
- [توليد الفيديو](/tools/video-generation)
- [توليد الموسيقى](/tools/music-generation)
- [دليل الموفّرين](/ar/providers/index)
- [مرجع الإعدادات](/ar/gateway/configuration-reference#agent-defaults)
