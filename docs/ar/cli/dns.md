---
read_when:
    - أنت تريد الاكتشاف عبر النطاق الواسع (DNS-SD) باستخدام Tailscale + CoreDNS
    - You’re setting up split DNS for a custom discovery domain (example: openclaw.internal)
summary: مرجع CLI لـ `openclaw dns` (مساعدات الاكتشاف عبر النطاق الواسع)
title: dns
x-i18n:
    generated_at: "2026-04-05T12:38:10Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4831fbb7791adfed5195bc4ba36bb248d2bc8830958334211d3c96f824617927
    source_path: cli/dns.md
    workflow: 15
---

# `openclaw dns`

مساعدات DNS للاكتشاف عبر النطاق الواسع (Tailscale + CoreDNS). وهي تركز حاليًا على macOS + Homebrew CoreDNS.

ذو صلة:

- اكتشاف البوابة: [الاكتشاف](/gateway/discovery)
- إعداد الاكتشاف عبر النطاق الواسع: [الإعداد](/gateway/configuration)

## الإعداد

```bash
openclaw dns setup
openclaw dns setup --domain openclaw.internal
openclaw dns setup --apply
```

## `dns setup`

تخطيط أو تطبيق إعداد CoreDNS لاكتشاف DNS-SD أحادي الإرسال.

الخيارات:

- `--domain <domain>`: نطاق الاكتشاف عبر النطاق الواسع (على سبيل المثال `openclaw.internal`)
- `--apply`: تثبيت أو تحديث إعداد CoreDNS وإعادة تشغيل الخدمة (يتطلب sudo؛ macOS فقط)

ما الذي يعرضه:

- نطاق الاكتشاف الذي تم تحليله
- مسار ملف المنطقة
- عناوين tailnet IP الحالية
- إعداد `openclaw.json` الموصى به للاكتشاف
- قيم nameserver/domain لـ Tailscale Split DNS التي يجب ضبطها

ملاحظات:

- من دون `--apply`، يكون الأمر مجرد مساعد للتخطيط ويطبع الإعداد الموصى به.
- إذا تم حذف `--domain`، يستخدم OpenClaw القيمة `discovery.wideArea.domain` من الإعداد.
- يدعم `--apply` حاليًا macOS فقط ويفترض استخدام Homebrew CoreDNS.
- يقوم `--apply` بتهيئة ملف المنطقة إذا لزم الأمر، ويضمن وجود مقطع import الخاص بـ CoreDNS، ويعيد تشغيل خدمة `coredns` عبر brew.
