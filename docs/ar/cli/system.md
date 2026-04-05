---
read_when:
    - تريد إدراج حدث نظام في قائمة الانتظار من دون إنشاء مهمة cron
    - تحتاج إلى تمكين heartbeats أو تعطيلها
    - تريد فحص إدخالات حضور النظام
summary: مرجع CLI للأمر `openclaw system` (أحداث النظام، heartbeat، والحضور)
title: system
x-i18n:
    generated_at: "2026-04-05T12:39:33Z"
    model: gpt-5.4
    provider: openai
    source_hash: a7d19afde9d9cde8a79b0bb8cec6e5673466f4cb9b575fb40111fc32f4eee5d7
    source_path: cli/system.md
    workflow: 15
---

# `openclaw system`

مساعدات على مستوى النظام للبوابة: إدراج أحداث النظام في قائمة الانتظار، والتحكم في heartbeats،
وعرض الحضور.

تستخدم جميع الأوامر الفرعية ضمن `system` بروتوكول Gateway RPC وتقبل علامات العميل المشتركة:

- `--url <url>`
- `--token <token>`
- `--timeout <ms>`
- `--expect-final`

## الأوامر الشائعة

```bash
openclaw system event --text "Check for urgent follow-ups" --mode now
openclaw system event --text "Check for urgent follow-ups" --url ws://127.0.0.1:18789 --token "$OPENCLAW_GATEWAY_TOKEN"
openclaw system heartbeat enable
openclaw system heartbeat last
openclaw system presence
```

## `system event`

أدرج حدث نظام في قائمة الانتظار على الجلسة **الرئيسية**. سيقوم heartbeat التالي
بحقنه كسطر `System:` داخل الـ prompt. استخدم `--mode now` لتشغيل heartbeat
فورًا؛ أما `next-heartbeat` فينتظر النبضة المجدولة التالية.

العلامات:

- `--text <text>`: نص حدث النظام المطلوب.
- `--mode <mode>`: ‏`now` أو `next-heartbeat` (الافتراضي).
- `--json`: إخراج قابل للقراءة آليًا.
- `--url` و`--token` و`--timeout` و`--expect-final`: علامات Gateway RPC المشتركة.

## `system heartbeat last|enable|disable`

عناصر التحكم في heartbeat:

- `last`: عرض آخر حدث heartbeat.
- `enable`: إعادة تشغيل heartbeats (استخدم هذا إذا كانت معطلة).
- `disable`: إيقاف heartbeats مؤقتًا.

العلامات:

- `--json`: إخراج قابل للقراءة آليًا.
- `--url` و`--token` و`--timeout` و`--expect-final`: علامات Gateway RPC المشتركة.

## `system presence`

اعرض إدخالات حضور النظام الحالية التي تعرفها البوابة (العُقد،
والمثيلات، وأسطر الحالة المشابهة).

العلامات:

- `--json`: إخراج قابل للقراءة آليًا.
- `--url` و`--token` و`--timeout` و`--expect-final`: علامات Gateway RPC المشتركة.

## ملاحظات

- يتطلب Gateway قيد التشغيل ويمكن الوصول إليها عبر التكوين الحالي لديك (محليًا أو عن بُعد).
- أحداث النظام مؤقتة ولا تستمر عبر عمليات إعادة التشغيل.
