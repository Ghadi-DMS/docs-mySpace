---
read_when:
    - تحديث OpenClaw
    - حدث خلل ما بعد التحديث
summary: تحديث OpenClaw بأمان (تثبيت عام أو من المصدر)، بالإضافة إلى استراتيجية التراجع
title: التحديث
x-i18n:
    generated_at: "2026-04-05T12:48:38Z"
    model: gpt-5.4
    provider: openai
    source_hash: b40429d38ca851be4fdf8063ed425faf4610a4b5772703e0481c5f1fb588ba58
    source_path: install/updating.md
    workflow: 15
---

# التحديث

حافظ على OpenClaw محدثًا.

## الموصى به: `openclaw update`

أسرع طريقة للتحديث. يكتشف نوع التثبيت لديك (npm أو git)، ويجلب أحدث إصدار، ويشغّل `openclaw doctor`، ثم يعيد تشغيل البوابة.

```bash
openclaw update
```

للتبديل بين القنوات أو استهداف إصدار محدد:

```bash
openclaw update --channel beta
openclaw update --tag main
openclaw update --dry-run   # preview without applying
```

يفضّل `--channel beta` قناة beta، لكن وقت التشغيل يرجع إلى stable/latest عندما
يكون وسم beta مفقودًا أو أقدم من أحدث إصدار stable. استخدم `--tag beta`
إذا كنت تريد وسم npm beta dist-tag الخام لتحديث حزمة لمرة واحدة.

راجع [قنوات التطوير](/install/development-channels) لمعرفة دلالات القنوات.

## بديل: أعد تشغيل المثبّت

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

أضف `--no-onboard` لتخطي onboarding. وبالنسبة إلى التثبيتات من المصدر، مرّر `--install-method git --no-onboard`.

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

## المُحدِّث التلقائي

يكون المُحدِّث التلقائي متوقفًا افتراضيًا. فعّله في `~/.openclaw/openclaw.json`:

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

| القناة | السلوك |
| ------ | ------ |
| `stable` | ينتظر `stableDelayHours`، ثم يطبّق مع jitter حتمي عبر `stableJitterHours` (طرح تدريجي موزع). |
| `beta` | يتحقق كل `betaCheckIntervalHours` (الافتراضي: كل ساعة) ويطبّق فورًا. |
| `dev` | لا يوجد تطبيق تلقائي. استخدم `openclaw update` يدويًا. |

كما تسجل البوابة تلميحًا للتحديث عند بدء التشغيل (عطّله عبر `update.checkOnStart: false`).

## بعد التحديث

<Steps>

### شغّل doctor

```bash
openclaw doctor
```

يرحّل الإعدادات، ويدقق سياسات الرسائل الخاصة، ويتحقق من صحة البوابة. التفاصيل: [Doctor](/gateway/doctor)

### أعد تشغيل البوابة

```bash
openclaw gateway restart
```

### تحقّق

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

### تثبيت commit محدد (من المصدر)

```bash
git fetch origin
git checkout "$(git rev-list -n 1 --before=\"2026-01-01\" origin/main)"
pnpm install && pnpm build
openclaw gateway restart
```

للعودة إلى الأحدث: `git checkout main && git pull`.

## إذا علقت

- شغّل `openclaw doctor` مرة أخرى واقرأ المخرجات بعناية.
- راجع: [استكشاف الأخطاء وإصلاحها](/gateway/troubleshooting)
- اسأل في Discord: [https://discord.gg/clawd](https://discord.gg/clawd)

## ذو صلة

- [نظرة عامة على التثبيت](/install) — جميع طرق التثبيت
- [Doctor](/gateway/doctor) — فحوصات الصحة بعد التحديثات
- [الترحيل](/install/migrating) — أدلة الترحيل بين الإصدارات الرئيسية
