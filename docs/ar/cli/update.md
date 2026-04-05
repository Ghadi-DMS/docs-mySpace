---
read_when:
    - تريد تحديث نسخة مصدرية بأمان
    - تحتاج إلى فهم سلوك الاختصار `--update`
summary: مرجع CLI للأمر `openclaw update` ‏(تحديث آمن نسبيًا للمصدر + إعادة تشغيل تلقائية لـ gateway)
title: update
x-i18n:
    generated_at: "2026-04-05T12:39:46Z"
    model: gpt-5.4
    provider: openai
    source_hash: 12c8098654b644c3666981d379f6c018e84fde56a5420f295d78052f9001bdad
    source_path: cli/update.md
    workflow: 15
---

# `openclaw update`

حدّث OpenClaw بأمان وبدّل بين قنوات stable/beta/dev.

إذا قمت بالتثبيت عبر **npm/pnpm/bun** (تثبيت عام، من دون بيانات git الوصفية)،
فتتم التحديثات عبر تدفق مدير الحزم الموضح في [التحديث](/install/updating).

## الاستخدام

```bash
openclaw update
openclaw update status
openclaw update wizard
openclaw update --channel beta
openclaw update --channel dev
openclaw update --tag beta
openclaw update --tag main
openclaw update --dry-run
openclaw update --no-restart
openclaw update --yes
openclaw update --json
openclaw --update
```

## الخيارات

- `--no-restart`: تخطي إعادة تشغيل خدمة Gateway بعد تحديث ناجح.
- `--channel <stable|beta|dev>`: تعيين قناة التحديث (git + npm؛ وتُحفَظ في التكوين).
- `--tag <dist-tag|version|spec>`: تجاوز هدف الحزمة لهذا التحديث فقط. بالنسبة إلى تثبيتات الحزم، يتم تعيين `main` إلى `github:openclaw/openclaw#main`.
- `--dry-run`: معاينة إجراءات التحديث المخطط لها (القناة/الوسم/الهدف/تدفق إعادة التشغيل) من دون كتابة التكوين أو التثبيت أو مزامنة plugins أو إعادة التشغيل.
- `--json`: طباعة JSON قابل للقراءة آليًا من نوع `UpdateRunResult`.
- `--timeout <seconds>`: مهلة لكل خطوة (الافتراضي 1200 ثانية).
- `--yes`: تخطي مطالبات التأكيد (مثل تأكيد الرجوع إلى إصدار أقدم)

ملاحظة: تتطلب عمليات الرجوع إلى إصدار أقدم تأكيدًا لأن الإصدارات الأقدم قد تكسر التكوين.

## `update status`

إظهار قناة التحديث النشطة + وسم/فرع/SHA الخاص بـ git (لنسخ المصدر)، بالإضافة إلى توفر التحديث.

```bash
openclaw update status
openclaw update status --json
openclaw update status --timeout 10
```

الخيارات:

- `--json`: طباعة JSON للحالة قابل للقراءة آليًا.
- `--timeout <seconds>`: مهلة للفحوصات (الافتراضي 3 ثوانٍ).

## `update wizard`

تدفق تفاعلي لاختيار قناة تحديث وتأكيد ما إذا كان يجب إعادة تشغيل Gateway
بعد التحديث (الافتراضي هو إعادة التشغيل). إذا اخترت `dev` من دون نسخة git،
فسيعرض إنشاء واحدة.

الخيارات:

- `--timeout <seconds>`: مهلة لكل خطوة تحديث (الافتراضي `1200`)

## ما الذي يفعله

عندما تبدّل القنوات صراحةً (`--channel ...`)، يحافظ OpenClaw أيضًا على
توافق طريقة التثبيت:

- `dev` ← يضمن وجود نسخة git محلية (الافتراضي: `~/openclaw`، ويمكن التجاوز عبر `OPENCLAW_GIT_DIR`)،
  ويحدّثها، ويثبت CLI العام من تلك النسخة.
- `stable` ← يثبّت من npm باستخدام `latest`.
- `beta` ← يفضل npm dist-tag ‏`beta`، لكنه يعود إلى `latest` عندما تكون beta
  مفقودة أو أقدم من الإصدار المستقر الحالي.

يعيد المحدّث التلقائي لنواة Gateway (عند تمكينه عبر التكوين) استخدام مسار التحديث نفسه هذا.

## تدفق نسخة git

القنوات:

- `stable`: سحب أحدث وسم غير beta، ثم البناء + doctor.
- `beta`: يفضل أحدث وسم `-beta`، لكنه يعود إلى أحدث وسم stable
  عندما تكون beta مفقودة أو أقدم.
- `dev`: سحب `main`، ثم fetch + rebase.

على مستوى عالٍ:

1. يتطلب شجرة عمل نظيفة (من دون تغييرات غير ملتزم بها).
2. يبدّل إلى القناة المحددة (وسم أو فرع).
3. يجلب من upstream ‏(لـ dev فقط).
4. لـ dev فقط: تنفيذ فحص lint مسبق + بناء TypeScript في worktree مؤقت؛ وإذا فشل الطرف الحالي، يتراجع حتى 10 التزامات للعثور على أحدث بناء نظيف.
5. يعيد rebase على الالتزام المحدد (لـ dev فقط).
6. يثبت التبعيات (يفضل pnpm؛ مع بديل npm؛ ويظل bun متاحًا كبديل توافق ثانوي).
7. يبني + يبني Control UI.
8. يشغّل `openclaw doctor` كفحص "تحديث آمن" نهائي.
9. يزامن plugins مع القناة النشطة (يستخدم dev الامتدادات المضمّنة؛ بينما يستخدم stable/beta npm) ويحدّث plugins المثبتة عبر npm.

## الاختصار `--update`

يعيد `openclaw --update` الكتابة إلى `openclaw update` (وهو مفيد للأصداف وscripts الخاصة بالمشغّل).

## راجع أيضًا

- `openclaw doctor` ‏(يعرض تشغيل التحديث أولًا على نسخ git)
- [قنوات التطوير](/install/development-channels)
- [التحديث](/install/updating)
- [مرجع CLI](/cli)
