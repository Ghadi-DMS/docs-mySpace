---
read_when:
    - تريد إزالة خدمة gateway و/أو الحالة المحلية
    - تريد تنفيذًا تجريبيًا أولًا
summary: مرجع CLI للأمر `openclaw uninstall` (إزالة خدمة gateway + البيانات المحلية)
title: uninstall
x-i18n:
    generated_at: "2026-04-05T12:39:31Z"
    model: gpt-5.4
    provider: openai
    source_hash: 2123a4f9c7a070ef7e13c60dafc189053ef61ce189fa4f29449dd50987c1894c
    source_path: cli/uninstall.md
    workflow: 15
---

# `openclaw uninstall`

إلغاء تثبيت خدمة gateway + البيانات المحلية (مع بقاء CLI).

الخيارات:

- `--service`: إزالة خدمة gateway
- `--state`: إزالة الحالة والتهيئة
- `--workspace`: إزالة أدلة workspace
- `--app`: إزالة تطبيق macOS
- `--all`: إزالة الخدمة، والحالة، وworkspace، والتطبيق
- `--yes`: تخطي مطالبات التأكيد
- `--non-interactive`: تعطيل المطالبات؛ ويتطلب `--yes`
- `--dry-run`: طباعة الإجراءات من دون إزالة الملفات

أمثلة:

```bash
openclaw backup create
openclaw uninstall
openclaw uninstall --service --yes --non-interactive
openclaw uninstall --state --workspace --yes --non-interactive
openclaw uninstall --all --yes
openclaw uninstall --dry-run
```

ملاحظات:

- شغّل `openclaw backup create` أولًا إذا كنت تريد لقطة قابلة للاستعادة قبل إزالة الحالة أو مساحات العمل.
- يُعد `--all` اختصارًا لإزالة الخدمة، والحالة، وworkspace، والتطبيق معًا.
- يتطلب `--non-interactive` الخيار `--yes`.
