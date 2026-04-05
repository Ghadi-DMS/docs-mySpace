---
read_when:
    - أنت تجري إعداد التشغيل الأول من دون الإعداد الأولي الكامل لـ CLI
    - تريد تعيين مسار مساحة العمل الافتراضي
summary: مرجع CLI للأمر `openclaw setup` (تهيئة الإعداد + مساحة العمل)
title: setup
x-i18n:
    generated_at: "2026-04-05T12:39:25Z"
    model: gpt-5.4
    provider: openai
    source_hash: f538aac341c749043ad959e35f2ed99c844ab8c3500ff59aa159d940bd301792
    source_path: cli/setup.md
    workflow: 15
---

# `openclaw setup`

هيّئ `~/.openclaw/openclaw.json` ومساحة عمل الوكيل.

ذو صلة:

- البدء: [البدء](/ar/start/getting-started)
- الإعداد الأولي لـ CLI: [Onboarding (CLI)](/ar/start/wizard)

## أمثلة

```bash
openclaw setup
openclaw setup --workspace ~/.openclaw/workspace
openclaw setup --wizard
openclaw setup --non-interactive --mode remote --remote-url wss://gateway-host:18789 --remote-token <token>
```

## الخيارات

- `--workspace <dir>`: دليل مساحة عمل الوكيل (يُخزَّن على أنه `agents.defaults.workspace`)
- `--wizard`: تشغيل الإعداد الأولي
- `--non-interactive`: تشغيل الإعداد الأولي من دون مطالبات
- `--mode <local|remote>`: وضع الإعداد الأولي
- `--remote-url <url>`: عنوان Gateway WebSocket URL البعيد
- `--remote-token <token>`: رمز Gateway البعيد

لتشغيل الإعداد الأولي عبر setup:

```bash
openclaw setup --wizard
```

ملاحظات:

- يهيّئ `openclaw setup` العادي الإعداد + مساحة العمل من دون تدفق الإعداد الأولي الكامل.
- يعمل الإعداد الأولي تلقائيًا عند وجود أي من علامات الإعداد الأولي (`--wizard`, `--non-interactive`, `--mode`, `--remote-url`, `--remote-token`).
