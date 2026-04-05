---
read_when:
    - أنت توافق على طلبات اقتران الأجهزة
    - تحتاج إلى تدوير رموز الأجهزة أو إلغائها
summary: مرجع CLI للأمر `openclaw devices` ‏(اقتران الأجهزة + تدوير/إلغاء رموز الأجهزة)
title: devices
x-i18n:
    generated_at: "2026-04-05T12:38:22Z"
    model: gpt-5.4
    provider: openai
    source_hash: e2f9fcb8e3508a703590f87caaafd953a5d3557e11c958cbb2be1d67bb8720f4
    source_path: cli/devices.md
    workflow: 15
---

# `openclaw devices`

إدارة طلبات اقتران الأجهزة والرموز المميزة الخاصة بنطاق الجهاز.

## الأوامر

### `openclaw devices list`

يعرض طلبات الاقتران المعلقة والأجهزة المقترنة.

```
openclaw devices list
openclaw devices list --json
```

يتضمن خرج الطلبات المعلقة الدور المطلوب والنطاقات حتى يمكن
مراجعة الموافقات قبل اعتمادها.

### `openclaw devices remove <deviceId>`

إزالة إدخال جهاز مقترن واحد.

عندما تكون مصادقًا باستخدام رمز جهاز مقترن، يمكن للمتصلين غير المشرفين
إزالة إدخال **أجهزتهم فقط**. وتتطلب إزالة جهاز آخر
الصلاحية `operator.admin`.

```
openclaw devices remove <deviceId>
openclaw devices remove <deviceId> --json
```

### `openclaw devices clear --yes [--pending]`

مسح الأجهزة المقترنة بشكل جماعي.

```
openclaw devices clear --yes
openclaw devices clear --yes --pending
openclaw devices clear --yes --pending --json
```

### `openclaw devices approve [requestId] [--latest]`

الموافقة على طلب اقتران جهاز معلق. إذا تم حذف `requestId`، فإن OpenClaw
يوافق تلقائيًا على أحدث طلب معلق.

ملاحظة: إذا أعاد جهاز محاولة الاقتران مع تغيّر في تفاصيل المصادقة (الدور/النطاقات/المفتاح
العام)، فإن OpenClaw يستبدل الإدخال المعلق السابق ويصدر
`requestId` جديدًا. شغّل `openclaw devices list` مباشرة قبل الموافقة لاستخدام
المعرّف الحالي.

```
openclaw devices approve
openclaw devices approve <requestId>
openclaw devices approve --latest
```

### `openclaw devices reject <requestId>`

رفض طلب اقتران جهاز معلق.

```
openclaw devices reject <requestId>
```

### `openclaw devices rotate --device <id> --role <role> [--scope <scope...>]`

تدوير رمز جهاز لدور محدد (مع تحديث النطاقات اختياريًا).
يجب أن يكون الدور المستهدف موجودًا بالفعل ضمن عقد الاقتران المعتمد لذلك الجهاز؛
ولا يمكن للتدوير إنشاء دور جديد غير معتمد.
إذا حذفت `--scope`، فإن عمليات إعادة الاتصال اللاحقة باستخدام الرمز المُدار المخزن تعيد استخدام
النطاقات المعتمدة المخزنة مؤقتًا لذلك الرمز. وإذا مررت قيم `--scope` صريحة، فإنها
تصبح مجموعة النطاقات المخزنة لعمليات إعادة الاتصال المستقبلية باستخدام الرمز المخزن مؤقتًا.
يمكن للمتصلين غير المشرفين من جلسات الأجهزة المقترنة تدوير **رمز أجهزتهم فقط**.
وأيضًا، يجب أن تبقى أي قيم `--scope` صريحة ضمن
نطاقات operator الخاصة بجلسة المتصل نفسها؛ فلا يمكن للتدوير إنشاء رمز operator
أوسع من الذي يملكه المتصل بالفعل.

```
openclaw devices rotate --device <deviceId> --role operator --scope operator.read --scope operator.write
```

يعيد حمولة الرمز الجديد بصيغة JSON.

### `openclaw devices revoke --device <id> --role <role>`

إلغاء رمز جهاز لدور محدد.

يمكن للمتصلين غير المشرفين من الأجهزة المقترنة إلغاء **رمز أجهزتهم فقط**.
أما إلغاء رمز جهاز آخر فيتطلب `operator.admin`.

```
openclaw devices revoke --device <deviceId> --role node
```

يعيد نتيجة الإلغاء بصيغة JSON.

## الخيارات الشائعة

- `--url <url>`: عنوان URL الخاص بـ Gateway WebSocket ‏(يستخدم `gateway.remote.url` افتراضيًا عند إعداده).
- `--token <token>`: رمز Gateway ‏(إذا كان مطلوبًا).
- `--password <password>`: كلمة مرور Gateway ‏(مصادقة كلمة المرور).
- `--timeout <ms>`: مهلة RPC.
- `--json`: خرج JSON ‏(موصى به للبرمجة النصية).

ملاحظة: عند ضبط `--url`، لا يعود CLI إلى بيانات الاعتماد الموجودة في الإعدادات أو البيئة.
مرّر `--token` أو `--password` صراحةً. ويعد غياب بيانات اعتماد صريحة خطأً.

## ملاحظات

- يعيد تدوير الرمز رمزًا جديدًا (حساسًا). تعامل معه على أنه سر.
- تتطلب هذه الأوامر النطاق `operator.pairing` ‏(أو `operator.admin`).
- يبقى تدوير الرمز ضمن مجموعة الأدوار المعتمدة في الاقتران وخط الأساس
  المعتمد للنطاقات لذلك الجهاز. ولا يمنح إدخال رمز مخزن مؤقتًا بالخطأ
  هدف تدوير جديدًا.
- بالنسبة إلى جلسات رموز الأجهزة المقترنة، تكون الإدارة عبر الأجهزة محصورة بالمشرف:
  تكون أوامر `remove` و`rotate` و`revoke` ذاتية فقط ما لم يكن لدى المتصل
  `operator.admin`.
- تم تقييد `devices clear` عمدًا بالعلم `--yes`.
- إذا لم يكن نطاق الاقتران متاحًا على local loopback (ولم يتم تمرير `--url` صراحةً)، يمكن للأمرين list/approve استخدام تراجع اقتران محلي.
- يختار `devices approve` أحدث طلب معلق تلقائيًا عندما تحذف `requestId` أو تمرر `--latest`.

## قائمة التحقق من استعادة انحراف الرمز

استخدم هذا عندما يستمر Control UI أو العملاء الآخرون في الفشل مع `AUTH_TOKEN_MISMATCH` أو `AUTH_DEVICE_TOKEN_MISMATCH`.

1. أكّد مصدر رمز gateway الحالي:

```bash
openclaw config get gateway.auth.token
```

2. اعرض الأجهزة المقترنة وحدد معرّف الجهاز المتأثر:

```bash
openclaw devices list
```

3. دوّر رمز operator للجهاز المتأثر:

```bash
openclaw devices rotate --device <deviceId> --role operator
```

4. إذا لم يكن التدوير كافيًا، فأزل الاقتران القديم ووافق من جديد:

```bash
openclaw devices remove <deviceId>
openclaw devices list
openclaw devices approve <requestId>
```

5. أعد محاولة اتصال العميل باستخدام الرمز/كلمة المرور المشتركة الحالية.

ملاحظات:

- تكون أولوية مصادقة إعادة الاتصال العادية هي: الرمز/كلمة المرور المشتركة الصريحة أولًا، ثم `deviceToken` الصريح، ثم رمز الجهاز المخزن، ثم رمز bootstrap.
- يمكن لاستعادة `AUTH_TOKEN_MISMATCH` الموثوقة أن ترسل مؤقتًا كلًا من الرمز المشترك ورمز الجهاز المخزن معًا لتلك إعادة المحاولة الواحدة المحدودة.

ذو صلة:

- [استكشاف مشكلات مصادقة لوحة المعلومات](/web/dashboard#if-you-see-unauthorized-1008)
- [استكشاف مشكلات Gateway وإصلاحها](/gateway/troubleshooting#dashboard-control-ui-connectivity)
