---
read_when:
    - إضافة أو تعديل تحليل مواقع القنوات
    - استخدام حقول سياق الموقع في مطالبات الوكيل أو الأدوات
summary: تحليل مواقع القنوات الواردة (Telegram/WhatsApp/Matrix) وحقول السياق
title: تحليل مواقع القنوات
x-i18n:
    generated_at: "2026-04-05T12:35:18Z"
    model: gpt-5.4
    provider: openai
    source_hash: 10061f0c109240a9e0bcab649b17f03b674e8bdf410debf3669b7b6da8189d96
    source_path: channels/location.md
    workflow: 15
---

# تحليل مواقع القنوات

يقوم OpenClaw بتوحيد المواقع المشتركة من قنوات الدردشة إلى:

- نص سهل القراءة يُضاف إلى نص الرسالة الواردة، و
- حقول مُهيكلة في حمولة سياق الرد التلقائي.

المدعوم حاليًا:

- **Telegram** (دبابيس المواقع + الأماكن + المواقع الحية)
- **WhatsApp** (`locationMessage` + `liveLocationMessage`)
- **Matrix** (`m.location` مع `geo_uri`)

## تنسيق النص

تُعرض المواقع كسطور واضحة من دون أقواس:

- دبوس:
  - `📍 48.858844, 2.294351 ±12m`
- مكان مسمى:
  - `📍 Eiffel Tower — Champ de Mars, Paris (48.858844, 2.294351 ±12m)`
- مشاركة حية:
  - `🛰 الموقع الحي: 48.858844, 2.294351 ±12m`

إذا كانت القناة تتضمن تعليقًا/وصفًا، فسيُضاف في السطر التالي:

```
📍 48.858844, 2.294351 ±12m
التقِ هنا
```

## حقول السياق

عند وجود موقع، تُضاف هذه الحقول إلى `ctx`:

- `LocationLat` (رقم)
- `LocationLon` (رقم)
- `LocationAccuracy` (رقم، بالأمتار؛ اختياري)
- `LocationName` (سلسلة نصية؛ اختياري)
- `LocationAddress` (سلسلة نصية؛ اختياري)
- `LocationSource` (`pin | place | live`)
- `LocationIsLive` (منطقي)

## ملاحظات القنوات

- **Telegram**: تُعيَّن الأماكن إلى `LocationName/LocationAddress`؛ وتستخدم المواقع الحية `live_period`.
- **WhatsApp**: يُضاف `locationMessage.comment` و`liveLocationMessage.caption` كسطر التسمية التوضيحية.
- **Matrix**: يتم تحليل `geo_uri` كموقع دبوس؛ ويتم تجاهل الارتفاع وتكون قيمة `LocationIsLive` دائمًا false.
