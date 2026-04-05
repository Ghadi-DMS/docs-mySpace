---
read_when:
    - تحتاج إلى فهم كيفية توحيد الطوابع الزمنية للنموذج
    - إعداد المنطقة الزمنية للمستخدم لمطالبات النظام
summary: معالجة المناطق الزمنية للوكلاء والمغلفات والمطالبات
title: المناطق الزمنية
x-i18n:
    generated_at: "2026-04-05T12:41:42Z"
    model: gpt-5.4
    provider: openai
    source_hash: 31a195fa43e3fc17b788d8e70d74ef55da998fc7997c4f0538d4331b1260baac
    source_path: concepts/timezone.md
    workflow: 15
---

# المناطق الزمنية

يوحّد OpenClaw الطوابع الزمنية بحيث يرى النموذج **وقتًا مرجعيًا واحدًا**.

## مغلفات الرسائل (محلية افتراضيًا)

تُغلَّف الرسائل الواردة في مغلف مثل:

```
[Provider ... 2026-01-05 16:26 PST] message text
```

يكون الطابع الزمني في المغلف **محليًا على المضيف افتراضيًا**، بدقة الدقائق.

يمكنك تجاوز ذلك باستخدام:

```json5
{
  agents: {
    defaults: {
      envelopeTimezone: "local", // "utc" | "local" | "user" | IANA timezone
      envelopeTimestamp: "on", // "on" | "off"
      envelopeElapsed: "on", // "on" | "off"
    },
  },
}
```

- يستخدم `envelopeTimezone: "utc"` توقيت UTC.
- يستخدم `envelopeTimezone: "user"` القيمة `agents.defaults.userTimezone` (مع الرجوع إلى المنطقة الزمنية للمضيف).
- استخدم منطقة زمنية IANA صريحة (مثل `"Europe/Vienna"`) للحصول على إزاحة ثابتة.
- يزيل `envelopeTimestamp: "off"` الطوابع الزمنية المطلقة من رؤوس المغلفات.
- يزيل `envelopeElapsed: "off"` لواحق الزمن المنقضي (بنمط `+2m`).

### أمثلة

**محلي (الافتراضي):**

```
[Signal Alice +1555 2026-01-18 00:19 PST] hello
```

**منطقة زمنية ثابتة:**

```
[Signal Alice +1555 2026-01-18 06:19 GMT+1] hello
```

**الزمن المنقضي:**

```
[Signal Alice +1555 +2m 2026-01-18T05:19Z] follow-up
```

## حمولات الأدوات (بيانات المزوّد الخام + حقول موحّدة)

تعيد استدعاءات الأدوات (`channels.discord.readMessages`, `channels.slack.readMessages`، وغيرها) **طوابع زمنية خام خاصة بالمزوّد**.
ونرفق أيضًا حقولًا موحّدة للاتساق:

- `timestampMs` (مللي ثانية epoch بتوقيت UTC)
- `timestampUtc` (سلسلة UTC بصيغة ISO 8601)

يتم الاحتفاظ بحقول المزوّد الخام.

## المنطقة الزمنية للمستخدم لمطالبة النظام

اضبط `agents.defaults.userTimezone` لإخبار النموذج بالمنطقة الزمنية المحلية للمستخدم. إذا كانت
غير مضبوطة، يحل OpenClaw **المنطقة الزمنية للمضيف وقت التشغيل** (من دون كتابة إلى التكوين).

```json5
{
  agents: { defaults: { userTimezone: "America/Chicago" } },
}
```

تتضمن مطالبة النظام:

- قسم `Current Date & Time` مع الوقت المحلي والمنطقة الزمنية
- `Time format: 12-hour` أو `24-hour`

يمكنك التحكم في تنسيق المطالبة باستخدام `agents.defaults.timeFormat` (`auto` | `12` | `24`).

راجع [Date & Time](/date-time) للاطلاع على السلوك الكامل والأمثلة.

## ذو صلة

- [Heartbeat](/gateway/heartbeat) — تستخدم الساعات النشطة المنطقة الزمنية للجدولة
- [Cron Jobs](/automation/cron-jobs) — تستخدم تعبيرات cron المنطقة الزمنية للجدولة
- [Date & Time](/date-time) — السلوك الكامل للتاريخ/الوقت والأمثلة
