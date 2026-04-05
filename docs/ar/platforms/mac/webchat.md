---
read_when:
    - تصحيح عرض WebChat على mac أو منفذ loopback
summary: كيف يضمّن تطبيق mac WebChat الخاص بالبوابة وكيفية تصحيحه
title: WebChat (macOS)
x-i18n:
    generated_at: "2026-04-05T12:50:21Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4f2c45fa5512cc9c5d3b3aa188d94e2e5a90e4bcce607d959d40bea8b17c90c5
    source_path: platforms/mac/webchat.md
    workflow: 15
---

# WebChat ‏(تطبيق macOS)

يقوم تطبيق شريط القوائم في macOS بتضمين واجهة WebChat كعرض SwiftUI أصلي. وهو
يتصل بالبوابة ويستخدم افتراضيًا **الجلسة الرئيسية** للوكيل المحدد
(مع مبدّل جلسات للجلسات الأخرى).

- **الوضع المحلي**: يتصل مباشرة بـ Gateway WebSocket المحلي.
- **الوضع البعيد**: يعيد توجيه منفذ التحكم في البوابة عبر SSH ويستخدم
  ذلك النفق كمستوى بيانات.

## التشغيل والتصحيح

- يدويًا: قائمة Lobster ← “Open Chat”.
- الفتح التلقائي للاختبار:

  ```bash
  dist/OpenClaw.app/Contents/MacOS/OpenClaw --webchat
  ```

- السجلات: `./scripts/clawlog.sh` ‏(subsystem ‏`ai.openclaw`، والفئة `WebChatSwiftUI`).

## كيفية التوصيل

- مستوى البيانات: أساليب Gateway WS وهي `chat.history` و`chat.send` و`chat.abort` و
  `chat.inject` والأحداث `chat` و`agent` و`presence` و`tick` و`health`.
- يعيد `chat.history` صفوف نصوص مطبّعة للعرض: تتم إزالة
  علامات التوجيه المضمّنة من النص المرئي، كما تتم إزالة حمولات XML النصية العادية
  الخاصة باستدعاءات الأدوات (بما في ذلك
  `<tool_call>...</tool_call>`،
  و`<function_call>...</function_call>`،
  و`<tool_calls>...</tool_calls>`،
  و`<function_calls>...</function_calls>`، وكتل استدعاء الأدوات المقتطعة)،
  كما تتم إزالة رموز التحكم الخاصة بالنموذج المسربة بصيغة ASCII/العرض الكامل،
  ويتم حذف صفوف المساعد التي تحتوي فقط على رموز صامتة مثل المطابقة الدقيقة لـ
  `NO_REPLY` / `no_reply`، ويمكن استبدال الصفوف كبيرة الحجم بعناصر نائبة.
- الجلسة: تستخدم افتراضيًا الجلسة الأساسية (`main`، أو `global` عندما يكون النطاق
  عامًا). ويمكن لواجهة المستخدم التبديل بين الجلسات.
- يستخدم الإعداد الأولي جلسة مخصصة لإبقاء إعداد التشغيل الأول منفصلًا.

## سطح الأمان

- يعيد الوضع البعيد توجيه منفذ التحكم WebSocket الخاص بالبوابة فقط عبر SSH.

## القيود المعروفة

- تم تحسين واجهة المستخدم لجلسات الدردشة (وليست صندوق حماية متصفح كاملًا).
