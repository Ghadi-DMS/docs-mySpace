---
read_when:
    - تريد التبديل بين stable/beta/dev
    - تريد تثبيت إصدار أو tag أو SHA محدد
    - أنت تقوم بوضع علامات أو نشر إصدارات prerelease
sidebarTitle: Release Channels
summary: 'القنوات المستقرة وbeta وdev: الدلالات، والتبديل، والتثبيت، ووضع العلامات'
title: قنوات الإصدار
x-i18n:
    generated_at: "2026-04-05T12:46:06Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3f33a77bf356f989cd4de5f8bb57f330c276e7571b955bea6994a4527e40258d
    source_path: install/development-channels.md
    workflow: 15
---

# قنوات التطوير

يشحن OpenClaw ثلاث قنوات تحديث:

- **stable**: ‏npm dist-tag `latest`. موصى بها لمعظم المستخدمين.
- **beta**: ‏npm dist-tag `beta` عندما تكون حالية؛ وإذا كانت beta مفقودة أو أقدم من
  أحدث إصدار stable، يعود تدفق التحديث إلى `latest`.
- **dev**: الرأس المتحرك لفرع `main` ‏(git). وnpm dist-tag هو: `dev` (عند النشر).
  فرع `main` مخصص للتجريب والتطوير النشط. وقد يحتوي على
  ميزات غير مكتملة أو تغييرات كاسرة. لا تستخدمه لبوابات الإنتاج.

نحن عادةً نشحن إصدارات stable إلى **beta** أولًا، ونختبرها هناك، ثم ننفذ
خطوة ترقية صريحة تنقل الإصدار المُعتمد إلى `latest` من دون
تغيير رقم الإصدار. ويمكن للمشرفين أيضًا نشر إصدار stable
مباشرة إلى `latest` عند الحاجة. وتُعد dist-tags هي المصدر المعتمد
لتثبيتات npm.

## التبديل بين القنوات

```bash
openclaw update --channel stable
openclaw update --channel beta
openclaw update --channel dev
```

يقوم `--channel` بحفظ اختيارك في التكوين (`update.channel`) ومواءمة
طريقة التثبيت:

- **`stable`** (تثبيتات الحزم): يحدّث عبر npm dist-tag ‏`latest`.
- **`beta`** (تثبيتات الحزم): يفضّل npm dist-tag ‏`beta`، لكنه يعود إلى
  `latest` عندما تكون `beta` مفقودة أو أقدم من وسم stable الحالي.
- **`stable`** (تثبيتات git): يجلب أحدث git tag مستقر.
- **`beta`** (تثبيتات git): يفضّل أحدث beta git tag، لكنه يعود إلى
  أحدث stable git tag عندما تكون beta مفقودة أو أقدم.
- **`dev`**: يضمن وجود checkout من git (الافتراضي `~/openclaw`، ويمكن تجاوزه بـ
  `OPENCLAW_GIT_DIR`)، وينتقل إلى `main`، ويجري rebase على upstream، ويبني،
  ويثبت CLI العالمية من ذلك checkout.

نصيحة: إذا كنت تريد stable + dev بالتوازي، فاحتفظ بنسختين مستنسختين ووجّه
البوابة إلى النسخة المستقرة.

## استهداف إصدار أو tag لمرة واحدة

استخدم `--tag` لاستهداف dist-tag أو إصدار أو package spec محدد لتحديث واحد
**من دون** تغيير القناة المحفوظة لديك:

```bash
# تثبيت إصدار محدد
openclaw update --tag 2026.4.1-beta.1

# التثبيت من beta dist-tag (لمرة واحدة، ولا يُحفظ)
openclaw update --tag beta

# التثبيت من فرع GitHub main (npm tarball)
openclaw update --tag main

# تثبيت npm package spec محدد
openclaw update --tag openclaw@2026.4.1-beta.1
```

ملاحظات:

- ينطبق `--tag` على **تثبيتات الحزم (npm) فقط**. وتتجاهله تثبيتات git.
- لا يتم حفظ tag. وسيستخدم `openclaw update` التالي القناة المكوّنة لديك
  كالمعتاد.
- حماية الرجوع إلى إصدار أقدم: إذا كان الإصدار المستهدف أقدم من إصدارك الحالي،
  فسيطلب OpenClaw التأكيد (تجاوزه باستخدام `--yes`).
- يختلف `--channel beta` عن `--tag beta`: إذ يمكن لتدفق القناة أن يعود
  إلى stable/latest عندما تكون beta مفقودة أو أقدم، بينما يستهدف `--tag beta`
  dist-tag الخام `beta` لهذا التشغيل فقط.

## تشغيل تجريبي

عاين ما الذي سيفعله `openclaw update` من دون إجراء أي تغييرات:

```bash
openclaw update --dry-run
openclaw update --channel beta --dry-run
openclaw update --tag 2026.4.1-beta.1 --dry-run
openclaw update --dry-run --json
```

يعرض التشغيل التجريبي القناة الفعلية، والإصدار المستهدف، والإجراءات المخطط لها،
وما إذا كان سيلزم تأكيد الرجوع إلى إصدار أقدم.

## plugins والقنوات

عند تبديل القنوات باستخدام `openclaw update`، يقوم OpenClaw أيضًا بمزامنة
مصادر plugins:

- يفضّل `dev` plugins المضمنة من git checkout.
- تستعيد `stable` و`beta` حزم plugins المثبتة عبر npm.
- يتم تحديث plugins المثبتة عبر npm بعد اكتمال تحديث core.

## التحقق من الحالة الحالية

```bash
openclaw update status
```

يعرض القناة النشطة، ونوع التثبيت (git أو package)، والإصدار الحالي،
والمصدر (التكوين، أو git tag، أو git branch، أو الافتراضي).

## أفضل ممارسات وضع العلامات

- ضع علامات على الإصدارات التي تريد أن تهبط عليها عمليات git checkout (`vYYYY.M.D` للإصدارات stable،
  و`vYYYY.M.D-beta.N` لإصدارات beta).
- يتم أيضًا التعرف على `vYYYY.M.D.beta.N` للتوافق، لكن يفضَّل استخدام `-beta.N`.
- ما زال يتم التعرف على الوسوم القديمة `vYYYY.M.D-<patch>` على أنها stable (غير beta).
- أبقِ الوسوم غير قابلة للتغيير: لا تنقل وسمًا ولا تعيد استخدامه أبدًا.
- تظل npm dist-tags هي المصدر المعتمد لتثبيتات npm:
  - `latest` -> stable
  - `beta` -> build مرشح أو build stable أُرسل أولًا إلى beta
  - `dev` -> لقطة من `main` (اختياري)

## توفر تطبيق macOS

قد **لا** تتضمن إصدارات beta وdev إصدارًا لتطبيق macOS. وهذا أمر مقبول:

- لا يزال من الممكن نشر git tag وnpm dist-tag.
- اذكر بوضوح "لا يوجد إصدار macOS لهذه beta" في ملاحظات الإصدار أو changelog.
