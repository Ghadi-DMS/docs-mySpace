---
read_when:
    - استضافة PeekabooBridge داخل OpenClaw.app
    - دمج Peekaboo عبر Swift Package Manager
    - تغيير بروتوكول/مسارات PeekabooBridge
summary: تكامل PeekabooBridge لأتمتة واجهة مستخدم macOS
title: Peekaboo Bridge
x-i18n:
    generated_at: "2026-04-05T12:50:08Z"
    model: gpt-5.4
    provider: openai
    source_hash: 30961eb502eecd23c017b58b834bd8cb00cab8b17302617d541afdace3ad8dba
    source_path: platforms/mac/peekaboo.md
    workflow: 15
---

# Peekaboo Bridge (أتمتة واجهة مستخدم macOS)

يمكن لـ OpenClaw استضافة **PeekabooBridge** بوصفه وسيطًا محليًا واعيًا بالأذونات
لأتمتة واجهة المستخدم. ويتيح ذلك لـ CLI المسمى `peekaboo` تشغيل أتمتة واجهة المستخدم مع
إعادة استخدام أذونات TCC الخاصة بتطبيق macOS.

## ما هذا (وما ليس كذلك)

- **المضيف**: يمكن لـ OpenClaw.app أن يعمل كمضيف PeekabooBridge.
- **العميل**: استخدم CLI المسمى `peekaboo` ‏(ولا توجد واجهة مستقلة من نوع `openclaw ui ...`).
- **واجهة المستخدم**: تبقى الطبقات المرئية في Peekaboo.app؛ ويكون OpenClaw مضيف وسيطًا خفيفًا.

## تمكين الجسر

في تطبيق macOS:

- Settings ← **Enable Peekaboo Bridge**

عند التمكين، يبدأ OpenClaw خادم UNIX socket محليًا. وإذا تم تعطيله، يتم إيقاف المضيف
ويعود `peekaboo` إلى المضيفين الآخرين المتاحين.

## ترتيب اكتشاف العميل

تحاول عملاء Peekaboo عادةً المضيفين بهذا الترتيب:

1. Peekaboo.app ‏(تجربة كاملة)
2. Claude.app ‏(إذا كان مثبتًا)
3. OpenClaw.app ‏(وسيط خفيف)

استخدم `peekaboo bridge status --verbose` لمعرفة المضيف النشط ومسار
المقبس المستخدم. ويمكنك التجاوز عبر:

```bash
export PEEKABOO_BRIDGE_SOCKET=/path/to/bridge.sock
```

## الأمان والأذونات

- يتحقق الجسر من **توقيعات الشيفرة الخاصة بالمتصل**؛ وتُفرَض قائمة سماح لـ TeamIDs
  ‏(TeamID الخاص بمضيف Peekaboo + TeamID الخاص بتطبيق OpenClaw).
- تنتهي مهلة الطلبات بعد نحو 10 ثوانٍ.
- إذا كانت الأذونات المطلوبة مفقودة، يعيد الجسر رسالة خطأ واضحة
  بدلًا من تشغيل System Settings.

## سلوك Snapshot ‏(الأتمتة)

تُخزَّن Snapshots في الذاكرة وتنتهي صلاحيتها تلقائيًا بعد نافذة قصيرة.
وإذا كنت تحتاج إلى احتفاظ أطول، فأعد الالتقاط من العميل.

## استكشاف الأخطاء وإصلاحها

- إذا أبلغ `peekaboo` عن “bridge client is not authorized”، فتأكد من أن العميل
  موقّع بشكل صحيح أو شغّل المضيف مع `PEEKABOO_ALLOW_UNSIGNED_SOCKET_CLIENTS=1`
  في وضع **debug** فقط.
- إذا لم يتم العثور على أي مضيفين، فافتح أحد تطبيقات المضيف (Peekaboo.app أو OpenClaw.app)
  وتأكد من منح الأذونات.
