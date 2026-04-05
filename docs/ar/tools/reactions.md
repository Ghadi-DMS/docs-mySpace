---
read_when:
    - العمل على التفاعلات في أي قناة
    - فهم كيفية اختلاف تفاعلات emoji عبر المنصات
summary: دلالات أداة التفاعلات عبر جميع القنوات المدعومة
title: التفاعلات
x-i18n:
    generated_at: "2026-04-05T12:59:27Z"
    model: gpt-5.4
    provider: openai
    source_hash: 9af2951eee32e73adb982dbdf39b32e4065993454e9cce2ad23b27565cab4f84
    source_path: tools/reactions.md
    workflow: 15
---

# التفاعلات

يمكن للوكيل إضافة تفاعلات emoji إلى الرسائل وإزالتها باستخدام أداة `message`
مع الإجراء `react`. يختلف سلوك التفاعلات حسب القناة.

## كيف يعمل

```json
{
  "action": "react",
  "messageId": "msg-123",
  "emoji": "thumbsup"
}
```

- الحقل `emoji` مطلوب عند إضافة تفاعل.
- اضبط `emoji` على سلسلة فارغة (`""`) لإزالة تفاعل (أو تفاعلات) البوت.
- اضبط `remove: true` لإزالة emoji محدد (ويتطلب `emoji` غير فارغ).

## سلوك القنوات

<AccordionGroup>
  <Accordion title="Discord and Slack">
    - تؤدي قيمة `emoji` الفارغة إلى إزالة جميع تفاعلات البوت على الرسالة.
    - يزيل `remove: true` الـ emoji المحدد فقط.
  </Accordion>

  <Accordion title="Google Chat">
    - تؤدي قيمة `emoji` الفارغة إلى إزالة تفاعلات التطبيق على الرسالة.
    - يزيل `remove: true` الـ emoji المحدد فقط.
  </Accordion>

  <Accordion title="Telegram">
    - تؤدي قيمة `emoji` الفارغة إلى إزالة تفاعلات البوت.
    - يؤدي `remove: true` أيضًا إلى إزالة التفاعلات، لكنه لا يزال يتطلب `emoji` غير فارغ للتحقق من الأداة.
  </Accordion>

  <Accordion title="WhatsApp">
    - تؤدي قيمة `emoji` الفارغة إلى إزالة تفاعل البوت.
    - يُحوَّل `remove: true` داخليًا إلى emoji فارغ (مع بقاء `emoji` مطلوبًا في استدعاء الأداة).
  </Accordion>

  <Accordion title="Zalo Personal (zalouser)">
    - يتطلب `emoji` غير فارغ.
    - يزيل `remove: true` تفاعل ذلك الـ emoji المحدد.
  </Accordion>

  <Accordion title="Feishu/Lark">
    - استخدم أداة `feishu_reaction` مع الإجراءات `add` و`remove` و`list`.
    - تتطلب الإضافة/الإزالة `emoji_type`؛ كما تتطلب الإزالة أيضًا `reaction_id`.
  </Accordion>

  <Accordion title="Signal">
    - تُضبط إشعارات التفاعلات الواردة عبر `channels.signal.reactionNotifications`: تؤدي `"off"` إلى تعطيلها، وتؤدي `"own"` (الافتراضي) إلى إصدار أحداث عندما يتفاعل المستخدمون مع رسائل البوت، وتؤدي `"all"` إلى إصدار أحداث لجميع التفاعلات.
  </Accordion>
</AccordionGroup>

## مستوى التفاعل

يتحكم إعداد `reactionLevel` لكل قناة في مدى استخدام الوكيل للتفاعلات. تكون القيم عادةً `off` أو `ack` أو `minimal` أو `extensive`.

- [Telegram reactionLevel](/ar/channels/telegram#reaction-notifications) — `channels.telegram.reactionLevel`
- [WhatsApp reactionLevel](/ar/channels/whatsapp#reactions) — `channels.whatsapp.reactionLevel`

اضبط `reactionLevel` على القنوات الفردية لضبط مدى نشاط الوكيل في التفاعل مع الرسائل على كل منصة.

## ذو صلة

- [Agent Send](/tools/agent-send) — أداة `message` التي تتضمن `react`
- [Channels](/ar/channels) — التكوين الخاص بكل قناة
