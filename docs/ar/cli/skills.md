---
read_when:
    - أنت تريد معرفة المهارات المتاحة والجاهزة للتشغيل
    - أنت تريد البحث عن المهارات أو تثبيتها أو تحديثها من ClawHub
    - أنت تريد استكشاف أخطاء الثنائيات/متغيرات البيئة/الإعدادات المفقودة للمهارات وإصلاحها
summary: مرجع CLI للأمر `openclaw skills` (بحث/تثبيت/تحديث/قائمة/معلومات/فحص)
title: skills
x-i18n:
    generated_at: "2026-04-05T12:39:31Z"
    model: gpt-5.4
    provider: openai
    source_hash: 11af59b1b6bff19cc043acd8d67bdd4303201d3f75f23c948b83bf14882c7bb1
    source_path: cli/skills.md
    workflow: 15
---

# `openclaw skills`

افحص المهارات المحلية وثبّت/حدّث المهارات من ClawHub.

ذو صلة:

- نظام Skills: [Skills](/tools/skills)
- إعدادات Skills: [Skills config](/tools/skills-config)
- عمليات التثبيت من ClawHub: [ClawHub](/tools/clawhub)

## الأوامر

```bash
openclaw skills search "calendar"
openclaw skills search --limit 20 --json
openclaw skills install <slug>
openclaw skills install <slug> --version <version>
openclaw skills install <slug> --force
openclaw skills update <slug>
openclaw skills update --all
openclaw skills list
openclaw skills list --eligible
openclaw skills list --json
openclaw skills list --verbose
openclaw skills info <name>
openclaw skills info <name> --json
openclaw skills check
openclaw skills check --json
```

يستخدم `search`/`install`/`update` ClawHub مباشرةً ويثبّت في دليل
`skills/` الخاص بمساحة العمل النشطة. بينما تظل `list`/`info`/`check` تفحص
المهارات المحلية المرئية لمساحة العمل والإعدادات الحالية.

يقوم أمر CLI `install` هذا بتنزيل مجلدات Skills من ClawHub. أما
عمليات تثبيت تبعيات Skills المدعومة من Gateway والتي يتم تشغيلها من onboarding أو إعدادات Skills
فتستخدم مسار الطلب المنفصل `skills.install` بدلًا من ذلك.

ملاحظات:

- يقبل `search [query...]` استعلامًا اختياريًا؛ احذفه لتصفح موجز البحث
  الافتراضي في ClawHub.
- يحدد `search --limit <n>` الحد الأقصى للنتائج المعادة.
- يؤدي `install --force` إلى الكتابة فوق مجلد Skill موجود في مساحة العمل
  لنفس `slug`.
- لا يقوم `update --all` إلا بتحديث عمليات التثبيت المتتبعة من ClawHub في مساحة العمل النشطة.
- تكون `list` هي الإجراء الافتراضي عندما لا يتم توفير أمر فرعي.
- تكتب `list` و`info` و`check` مخرجاتها المعروضة إلى stdout. ومع
  `--json`، فهذا يعني أن الحمولة القابلة للقراءة آليًا تبقى على stdout
  من أجل الأنابيب والبرامج النصية.
