---
read_when:
    - تريد مسح الحالة المحلية مع الإبقاء على CLI مثبتًا
    - تريد تنفيذًا تجريبيًا لما سيتم حذفه
summary: مرجع CLI للأمر `openclaw reset` (إعادة تعيين الحالة/التهيئة المحلية)
title: reset
x-i18n:
    generated_at: "2026-04-05T12:39:04Z"
    model: gpt-5.4
    provider: openai
    source_hash: ad464700f948bebe741ec309f25150714f0b280834084d4f531327418a42c79b
    source_path: cli/reset.md
    workflow: 15
---

# `openclaw reset`

إعادة تعيين التهيئة/الحالة المحلية (مع الإبقاء على CLI مثبتًا).

الخيارات:

- `--scope <scope>`: ‏`config` أو `config+creds+sessions` أو `full`
- `--yes`: تخطي مطالبات التأكيد
- `--non-interactive`: تعطيل المطالبات؛ ويتطلب `--scope` و`--yes`
- `--dry-run`: طباعة الإجراءات من دون إزالة الملفات

أمثلة:

```bash
openclaw backup create
openclaw reset
openclaw reset --dry-run
openclaw reset --scope config --yes --non-interactive
openclaw reset --scope config+creds+sessions --yes --non-interactive
openclaw reset --scope full --yes --non-interactive
```

ملاحظات:

- شغّل `openclaw backup create` أولًا إذا كنت تريد لقطة قابلة للاستعادة قبل إزالة الحالة المحلية.
- إذا حذفت `--scope`، فسيستخدم `openclaw reset` مطالبة تفاعلية لاختيار ما يجب إزالته.
- لا يكون `--non-interactive` صالحًا إلا عند ضبط كل من `--scope` و`--yes`.
