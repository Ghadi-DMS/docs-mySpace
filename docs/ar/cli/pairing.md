---
read_when:
    - أنت تستخدم الرسائل الخاصة بوضع الاقتران وتحتاج إلى الموافقة على المرسلين
summary: مرجع CLI للأمر `openclaw pairing` ‏(approve/list لطلبات الاقتران)
title: pairing
x-i18n:
    generated_at: "2026-04-05T12:39:02Z"
    model: gpt-5.4
    provider: openai
    source_hash: 122a608ef83ec2b1011fdfd1b59b94950a4dcc8b598335b0956e2eedece4958f
    source_path: cli/pairing.md
    workflow: 15
---

# `openclaw pairing`

الموافقة على طلبات اقتران الرسائل الخاصة أو فحصها (للقنوات التي تدعم الاقتران).

ذو صلة:

- تدفق الاقتران: [الاقتران](/channels/pairing)

## الأوامر

```bash
openclaw pairing list telegram
openclaw pairing list --channel telegram --account work
openclaw pairing list telegram --json

openclaw pairing approve <code>
openclaw pairing approve telegram <code>
openclaw pairing approve --channel telegram --account work <code> --notify
```

## `pairing list`

عرض طلبات الاقتران المعلقة لقناة واحدة.

الخيارات:

- `[channel]`: معرّف القناة الموضعي
- `--channel <channel>`: معرّف القناة الصريح
- `--account <accountId>`: معرّف الحساب للقنوات متعددة الحسابات
- `--json`: خرج قابل للقراءة آليًا

ملاحظات:

- إذا كانت هناك عدة قنوات مكوّنة وقابلة للاقتـران، فيجب عليك توفير قناة إما موضعيًا أو باستخدام `--channel`.
- القنوات الامتدادية مسموح بها طالما أن معرّف القناة صالح.

## `pairing approve`

الموافقة على رمز اقتران معلّق والسماح لذلك المرسل.

الاستخدام:

- `openclaw pairing approve <channel> <code>`
- `openclaw pairing approve --channel <channel> <code>`
- `openclaw pairing approve <code>` عندما تكون هناك قناة واحدة فقط مكوّنة وقابلة للاقتـران

الخيارات:

- `--channel <channel>`: معرّف القناة الصريح
- `--account <accountId>`: معرّف الحساب للقنوات متعددة الحسابات
- `--notify`: إرسال تأكيد مرة أخرى إلى صاحب الطلب على القناة نفسها

## ملاحظات

- إدخال القناة: مرّره موضعيًا (`pairing list telegram`) أو باستخدام `--channel <channel>`.
- يدعم `pairing list` الخيار `--account <accountId>` للقنوات متعددة الحسابات.
- يدعم `pairing approve` الخيارين `--account <accountId>` و`--notify`.
- إذا كانت هناك قناة واحدة فقط مكوّنة وقابلة للاقتـران، فيُسمح باستخدام `pairing approve <code>`.
