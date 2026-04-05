---
read_when:
    - أنت تدير nodes مقترنة (كاميرات، شاشة، canvas)
    - تحتاج إلى الموافقة على الطلبات أو استدعاء أوامر node
summary: مرجع CLI للأمر `openclaw nodes` ‏(status وpairing وinvoke وcamera/canvas/screen)
title: nodes
x-i18n:
    generated_at: "2026-04-05T12:38:54Z"
    model: gpt-5.4
    provider: openai
    source_hash: 1ce3095591c4623ad18e3eca8d8083e5c10266fbf94afea2d025f0ba8093a175
    source_path: cli/nodes.md
    workflow: 15
---

# `openclaw nodes`

إدارة nodes المقترنة (الأجهزة) واستدعاء قدرات node.

ذو صلة:

- نظرة عامة على Nodes: [Nodes](/nodes)
- الكاميرا: [Camera nodes](/nodes/camera)
- الصور: [Image nodes](/nodes/images)

الخيارات الشائعة:

- `--url` و`--token` و`--timeout` و`--json`

## الأوامر الشائعة

```bash
openclaw nodes list
openclaw nodes list --connected
openclaw nodes list --last-connected 24h
openclaw nodes pending
openclaw nodes approve <requestId>
openclaw nodes reject <requestId>
openclaw nodes rename --node <id|name|ip> --name <displayName>
openclaw nodes status
openclaw nodes status --connected
openclaw nodes status --last-connected 24h
```

يطبع `nodes list` جداول الطلبات المعلقة/المقترنة. تتضمن الصفوف المقترنة أحدث مدة منذ الاتصال (Last Connect).
استخدم `--connected` لإظهار nodes المتصلة حاليًا فقط. استخدم `--last-connected <duration>` من أجل
التصفية إلى nodes التي اتصلت خلال مدة معينة (مثل `24h` أو `7d`).

ملاحظة الموافقة:

- يحتاج `openclaw nodes pending` فقط إلى نطاق pairing.
- يرث `openclaw nodes approve <requestId>` متطلبات النطاق الإضافية من
  الطلب المعلق:
  - طلب بلا أوامر: pairing فقط
  - أوامر node غير التنفيذية: pairing + write
  - `system.run` / `system.run.prepare` / `system.which`: pairing + admin

## الاستدعاء

```bash
openclaw nodes invoke --node <id|name|ip> --command <command> --params <json>
```

علامات الاستدعاء:

- `--params <json>`: سلسلة كائن JSON ‏(الافتراضي `{}`).
- `--invoke-timeout <ms>`: مهلة استدعاء node ‏(الافتراضي `15000`).
- `--idempotency-key <key>`: مفتاح idempotency اختياري.
- يتم حظر `system.run` و`system.run.prepare` هنا؛ استخدم أداة `exec` مع `host=node` لتنفيذ shell.

لتنفيذ shell على node، استخدم أداة `exec` مع `host=node` بدلًا من `openclaw nodes run`.
أصبحت واجهة CLI الخاصة بـ `nodes` الآن مركزة على القدرات: RPC مباشر عبر `nodes invoke`، بالإضافة إلى pairing والكاميرا
والشاشة والموقع وcanvas والإشعارات.
