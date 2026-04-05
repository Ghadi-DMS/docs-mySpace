---
read_when:
    - تحتاج إلى تتبّع سجلات Gateway عن بُعد (من دون SSH)
    - تريد أسطر سجلات JSON لأغراض الأدوات
summary: مرجع CLI للأمر `openclaw logs` (تتبّع سجلات gateway عبر RPC)
title: logs
x-i18n:
    generated_at: "2026-04-05T12:38:28Z"
    model: gpt-5.4
    provider: openai
    source_hash: 238a52e31a9a332cab513ced049e92d032b03c50376895ce57dffa2ee7d1e4b4
    source_path: cli/logs.md
    workflow: 15
---

# `openclaw logs`

تتبّع سجلات ملفات Gateway عبر RPC ‏(يعمل في الوضع البعيد).

ذو صلة:

- نظرة عامة على التسجيل: [التسجيل](/logging)
- CLI الخاص بـ Gateway: [gateway](/cli/gateway)

## الخيارات

- `--limit <n>`: الحد الأقصى لعدد أسطر السجل المطلوب إرجاعها (الافتراضي `200`)
- `--max-bytes <n>`: الحد الأقصى للبايتات المقروءة من ملف السجل (الافتراضي `250000`)
- `--follow`: متابعة تدفق السجل
- `--interval <ms>`: فترة الاستطلاع أثناء المتابعة (الافتراضي `1000`)
- `--json`: إخراج أحداث JSON مفصولة بأسطر
- `--plain`: إخراج نص عادي من دون تنسيق مزيّن
- `--no-color`: تعطيل ألوان ANSI
- `--local-time`: عرض الطوابع الزمنية في منطقتك الزمنية المحلية

## خيارات Gateway RPC المشتركة

يقبل `openclaw logs` أيضًا علامات عميل Gateway القياسية:

- `--url <url>`: عنوان Gateway WebSocket URL
- `--token <token>`: رمز Gateway
- `--timeout <ms>`: المهلة بالمللي ثانية (الافتراضي `30000`)
- `--expect-final`: الانتظار حتى الاستجابة النهائية عندما يكون استدعاء Gateway مدعومًا من وكيل

عند تمرير `--url`، لا يطبّق CLI تلقائيًا بيانات الاعتماد من الإعداد أو البيئة. أدرج `--token` صراحةً إذا كان Gateway الهدف يتطلب المصادقة.

## أمثلة

```bash
openclaw logs
openclaw logs --follow
openclaw logs --follow --interval 2000
openclaw logs --limit 500 --max-bytes 500000
openclaw logs --json
openclaw logs --plain
openclaw logs --no-color
openclaw logs --limit 500
openclaw logs --local-time
openclaw logs --follow --local-time
openclaw logs --url ws://127.0.0.1:18789 --token "$OPENCLAW_GATEWAY_TOKEN"
```

## ملاحظات

- استخدم `--local-time` لعرض الطوابع الزمنية في منطقتك الزمنية المحلية.
- إذا طلب Gateway على local loopback المحلي إقرانًا، يعود `openclaw logs` تلقائيًا إلى ملف السجل المحلي المُكوَّن. ولا تستخدم أهداف `--url` الصريحة هذا السلوك الاحتياطي.
