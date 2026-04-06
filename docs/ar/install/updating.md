---
read_when:
    - تحديث OpenClaw
    - تعطل شيء ما بعد التحديث
summary: تحديث OpenClaw بأمان (تثبيت عام أو من المصدر)، بالإضافة إلى استراتيجية التراجع
title: التحديث
x-i18n:
    generated_at: "2026-04-06T03:08:12Z"
    model: gpt-5.4
    provider: openai
    source_hash: ca9fff0776b9f5977988b649e58a5d169e5fa3539261cb02779d724d4ca92877
    source_path: install/updating.md
    workflow: 15
---

# التحديث

حافظ على تحديث OpenClaw.

## موصى به: `openclaw update`

أسرع طريقة للتحديث. يكتشف نوع التثبيت لديك (npm أو git)، ويجلب أحدث إصدار، ويشغّل `openclaw doctor`، ثم يعيد تشغيل البوابة.

```bash
openclaw update
```

للتبديل بين القنوات أو استهداف إصدار محدد:

```bash
openclaw update --channel beta
openclaw update --tag main
openclaw update --dry-run   # معاينة بدون تطبيق
```

يفضّل `--channel beta` قناة beta، لكن وقت التشغيل يعود إلى stable/latest عندما
تكون علامة beta مفقودة أو أقدم من أحدث إصدار مستقر. استخدم `--tag beta`
إذا كنت تريد npm beta dist-tag الخام لتحديث حزمة لمرة واحدة.

راجع [قنوات التطوير](/ar/install/development-channels) لمعرفة دلالات القنوات.

## بديل: أعد تشغيل المثبّت

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

أضف `--no-onboard` لتخطي الإعداد الأولي. بالنسبة إلى التثبيتات من المصدر، مرّر `--install-method git --no-onboard`.

## بديل: npm أو pnpm أو bun يدويًا

```bash
npm i -g openclaw@latest
```

```bash
pnpm add -g openclaw@latest
```

```bash
bun add -g openclaw@latest
```

## أداة التحديث التلقائي

أداة التحديث التلقائي متوقفة افتراضيًا. فعّلها في `~/.openclaw/openclaw.json`:

```json5
{
  update: {
    channel: "stable",
    auto: {
      enabled: true,
      stableDelayHours: 6,
      stableJitterHours: 12,
      betaCheckIntervalHours: 1,
    },
  },
}
```

| القناة  | السلوك                                                                                                      |
| -------- | ------------------------------------------------------------------------------------------------------------- |
| `stable` | ينتظر `stableDelayHours`، ثم يطبّق مع تذبذب حتمي عبر `stableJitterHours` (نشر موزع). |
| `beta`   | يتحقق كل `betaCheckIntervalHours` (الافتراضي: كل ساعة) ويطبّق فورًا.                              |
| `dev`    | لا يوجد تطبيق تلقائي. استخدم `openclaw update` يدويًا.                                                           |

تسجل البوابة أيضًا تلميح تحديث عند بدء التشغيل (يمكن تعطيله عبر `update.checkOnStart: false`).

## بعد التحديث

<Steps>

### تشغيل doctor

```bash
openclaw doctor
```

يُرحّل الإعدادات، ويدقق سياسات الرسائل المباشرة، ويفحص سلامة البوابة. التفاصيل: [Doctor](/ar/gateway/doctor)

### إعادة تشغيل البوابة

```bash
openclaw gateway restart
```

### التحقق

```bash
openclaw health
```

</Steps>

## التراجع

### تثبيت إصدار محدد (npm)

```bash
npm i -g openclaw@<version>
openclaw doctor
openclaw gateway restart
```

نصيحة: يعرض `npm view openclaw version` الإصدار المنشور الحالي.

### تثبيت commit محدد (المصدر)

```bash
git fetch origin
git checkout "$(git rev-list -n 1 --before=\"2026-01-01\" origin/main)"
pnpm install && pnpm build
openclaw gateway restart
```

للعودة إلى الأحدث: `git checkout main && git pull`.

## إذا كنت عالقًا

- شغّل `openclaw doctor` مرة أخرى واقرأ المخرجات بعناية.
- بالنسبة إلى `openclaw update --channel dev` في checkouts المصدر، يقوم برنامج التحديث بتهيئة `pnpm` تلقائيًا عند الحاجة. إذا رأيت خطأ في تهيئة pnpm/corepack، فثبّت `pnpm` يدويًا (أو أعد تفعيل `corepack`) ثم أعد تشغيل التحديث.
- تحقق من: [استكشاف الأخطاء وإصلاحها](/ar/gateway/troubleshooting)
- اسأل في Discord: [https://discord.gg/clawd](https://discord.gg/clawd)

## ذو صلة

- [نظرة عامة على التثبيت](/ar/install) — جميع طرق التثبيت
- [Doctor](/ar/gateway/doctor) — فحوصات السلامة بعد التحديثات
- [الترحيل](/ar/install/migrating) — أدلة ترحيل الإصدارات الرئيسية
