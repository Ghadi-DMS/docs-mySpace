---
read_when:
    - أنت تريد استخدام توليد الفيديو Alibaba Wan في OpenClaw
    - أنت بحاجة إلى إعداد مفتاح API لـ Model Studio أو DashScope من أجل توليد الفيديو
summary: توليد الفيديو Wan عبر Alibaba Model Studio في OpenClaw
title: Alibaba Model Studio
x-i18n:
    generated_at: "2026-04-06T03:10:23Z"
    model: gpt-5.4
    provider: openai
    source_hash: 97a1eddc7cbd816776b9368f2a926b5ef9ee543f08d151a490023736f67dc635
    source_path: providers/alibaba.md
    workflow: 15
---

# Alibaba Model Studio

يشحن OpenClaw موفرًا مضمّنًا لتوليد الفيديو باسم `alibaba` لنماذج Wan على
Alibaba Model Studio / DashScope.

- الموفر: `alibaba`
- المصادقة المفضلة: `MODELSTUDIO_API_KEY`
- المقبول أيضًا: `DASHSCOPE_API_KEY`، `QWEN_API_KEY`
- API: توليد فيديو غير متزامن عبر DashScope / Model Studio

## البدء السريع

1. عيّن مفتاح API:

```bash
openclaw onboard --auth-choice qwen-standard-api-key
```

2. عيّن نموذج الفيديو الافتراضي:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "alibaba/wan2.6-t2v",
      },
    },
  },
}
```

## نماذج Wan المدمجة

يسجل الموفر المضمّن `alibaba` حاليًا:

- `alibaba/wan2.6-t2v`
- `alibaba/wan2.6-i2v`
- `alibaba/wan2.6-r2v`
- `alibaba/wan2.6-r2v-flash`
- `alibaba/wan2.7-r2v`

## الحدود الحالية

- حتى **1** فيديو ناتج لكل طلب
- حتى **1** صورة إدخال
- حتى **4** مقاطع فيديو إدخال
- مدة تصل إلى **10 ثوانٍ**
- يدعم `size` و `aspectRatio` و `resolution` و `audio` و `watermark`
- يتطلب وضع الصورة/الفيديو المرجعي حاليًا **عناوين URL بعيدة من نوع http(s)**

## العلاقة مع Qwen

يستخدم الموفر المضمّن `qwen` أيضًا نقاط نهاية DashScope المستضافة لدى Alibaba من أجل
توليد الفيديو Wan. استخدم:

- `qwen/...` عندما تريد سطح موفر Qwen القياسي
- `alibaba/...` عندما تريد سطح Wan المباشر المملوك للمورّد

## ذو صلة

- [توليد الفيديو](/tools/video-generation)
- [Qwen](/ar/providers/qwen)
- [مرجع الإعدادات](/ar/gateway/configuration-reference#agent-defaults)
