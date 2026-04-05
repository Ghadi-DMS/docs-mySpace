---
read_when:
    - البحث عن تعريفات قنوات الإصدار العامة
    - البحث عن تسمية الإصدارات والوتيرة
summary: قنوات الإصدار العامة، وتسمية الإصدارات، والوتيرة
title: سياسة الإصدار
x-i18n:
    generated_at: "2026-04-05T12:55:22Z"
    model: gpt-5.4
    provider: openai
    source_hash: bb52a13264c802395aa55404c6baeec5c7b2a6820562e7a684057e70cc85668f
    source_path: reference/RELEASING.md
    workflow: 15
---

# سياسة الإصدار

لدى OpenClaw ثلاثة مسارات إصدار عامة:

- stable: إصدارات موسومة تُنشر إلى npm `beta` افتراضيًا، أو إلى npm `latest` عند الطلب الصريح
- beta: وسوم الإصدارات التمهيدية التي تُنشر إلى npm `beta`
- dev: الرأس المتحرك لفرع `main`

## تسمية الإصدارات

- إصدار الإصدار المستقر: `YYYY.M.D`
  - وسم Git: `vYYYY.M.D`
- إصدار التصحيح للإصدار المستقر: `YYYY.M.D-N`
  - وسم Git: `vYYYY.M.D-N`
- إصدار beta التمهيدي: `YYYY.M.D-beta.N`
  - وسم Git: `vYYYY.M.D-beta.N`
- لا تُضِف أصفارًا بادئة للشهر أو اليوم
- `latest` يعني إصدار npm المستقر المُرقّى الحالي
- `beta` يعني هدف تثبيت beta الحالي
- تُنشر الإصدارات المستقرة وتصحيحات الإصدارات المستقرة إلى npm `beta` افتراضيًا؛ ويمكن لمشغّلي الإصدار استهداف `latest` صراحةً، أو ترقية بنية beta مُعتمدة لاحقًا
- كل إصدار من OpenClaw يشحن حزمة npm وتطبيق macOS معًا

## وتيرة الإصدار

- تبدأ الإصدارات عبر beta أولًا
- يتبع الإصدار المستقر فقط بعد التحقق من أحدث إصدار beta
- إجراء الإصدار التفصيلي، والموافقات، وبيانات الاعتماد، وملاحظات الاستعادة
  مخصّصة للمشرفين فقط

## الفحوصات التمهيدية للإصدار

- شغّل `pnpm build && pnpm ui:build` قبل `pnpm release:check` حتى تتوفر
  عناصر الإصدار المتوقعة `dist/*` وحزمة واجهة المستخدم Control UI من أجل
  خطوة التحقق من الحزمة
- شغّل `pnpm release:check` قبل كل إصدار موسوم
- الفحص التمهيدي لـ npm على الفرع الرئيسي يشغّل أيضًا
  `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_CACHE_TEST=1 pnpm test:live:cache`
  قبل تغليف tarball، باستخدام كلٍّ من أسرار سير العمل `OPENAI_API_KEY` و
  `ANTHROPIC_API_KEY`
- شغّل `RELEASE_TAG=vYYYY.M.D node --import tsx scripts/openclaw-npm-release-check.ts`
  (أو وسم beta/التصحيح المطابق) قبل الموافقة
- بعد النشر إلى npm، شغّل
  `node --import tsx scripts/openclaw-npm-postpublish-verify.ts YYYY.M.D`
  (أو إصدار beta/التصحيح المطابق) للتحقق من مسار التثبيت المنشور في السجل
  داخل بادئة مؤقتة جديدة
- تستخدم أتمتة الإصدار لدى المشرفين الآن أسلوب الفحص التمهيدي ثم الترقية:
  - يجب أن يجتاز النشر الفعلي إلى npm `preflight_run_id` ناجحًا
  - إصدارات npm المستقرة تضبط افتراضيًا على `beta`
  - يمكن أن يستهدف نشر npm المستقر `latest` صراحةً عبر مُدخل سير العمل
  - ما زالت ترقية npm المستقرة من `beta` إلى `latest` متاحة بوصفها وضعًا يدويًا صريحًا ضمن سير العمل الموثوق `OpenClaw NPM Release`
  - ما زال وضع الترقية هذا يحتاج إلى `NPM_TOKEN` صالح في بيئة `npm-release` لأن إدارة `dist-tag` في npm منفصلة عن النشر الموثوق
  - `macOS Release` العام مخصّص للتحقق فقط
  - يجب أن يجتاز النشر الخاص الفعلي لتطبيق mac بنجاح مرحلتي
    `preflight_run_id` و`validate_run_id` الخاصتين بـ mac الخاص
  - تعمل مسارات النشر الفعلية على ترقية العناصر المُحضّرة بدلًا من إعادة
    بنائها مرة أخرى
- بالنسبة إلى إصدارات التصحيح المستقرة مثل `YYYY.M.D-N`، يتحقق مُراجع ما بعد النشر
  أيضًا من مسار الترقية نفسه داخل بادئة مؤقتة من `YYYY.M.D` إلى `YYYY.M.D-N`
  حتى لا تترك تصحيحات الإصدار تثبيتات عامة أقدم بصمت على الحمولة الأساسية
  للإصدار المستقر
- يفشل الفحص التمهيدي لإصدار npm بشكل مغلق ما لم تتضمن tarball كلا الملفين
  `dist/control-ui/index.html` وحمولة غير فارغة ضمن `dist/control-ui/assets/`
  حتى لا نشحن لوحة تحكم متصفح فارغة مرة أخرى
- إذا لامس عمل الإصدار تخطيط CI، أو بيانات توقيت الإضافات، أو مصفوفات الاختبار
  السريعة، فأعد توليد وراجِع مخرجات مصفوفة سير العمل
  `checks-fast-extensions` المملوكة للمخطِّط من `.github/workflows/ci.yml`
  قبل الموافقة حتى لا تصف ملاحظات الإصدار تخطيط CI قديمًا
- يتضمن استعداد إصدار macOS المستقر أيضًا أسطح المحدِّث:
  - يجب أن ينتهي إصدار GitHub بملفات `.zip` و`.dmg` و`.dSYM.zip` المعبّأة
  - يجب أن يشير `appcast.xml` على `main` إلى ملف zip المستقر الجديد بعد النشر
  - يجب أن يحتفظ التطبيق المعبّأ بمعرّف حزمة غير مخصّص للتصحيح، وعنوان URL
    غير فارغ لخلاصة Sparkle، و`CFBundleVersion` يساوي أو يتجاوز حد بناء
    Sparkle الأدنى القياسي لذلك الإصدار

## مُدخلات سير عمل NPM

يقبل `OpenClaw NPM Release` مُدخلات يتحكم فيها المشغّل كما يلي:

- `tag`: وسم الإصدار المطلوب، مثل `v2026.4.2` أو `v2026.4.2-1` أو
  `v2026.4.2-beta.1`
- `preflight_only`: القيمة `true` للتحقق/البناء/التغليف فقط، و`false` لمسار
  النشر الفعلي
- `preflight_run_id`: مطلوب في مسار النشر الفعلي حتى يعيد سير العمل استخدام
  tarball المُحضّر من تشغيل الفحص التمهيدي الناجح
- `npm_dist_tag`: وسم npm المستهدف لمسار النشر؛ والقيمة الافتراضية هي `beta`
- `promote_beta_to_latest`: القيمة `true` لتخطي النشر ونقل بنية `beta`
  مستقرة منشورة مسبقًا إلى `latest`

القواعد:

- يمكن لوسوم الإصدارات المستقرة والتصحيحية النشر إلى `beta` أو `latest`
- يمكن لوسوم إصدارات beta التمهيدية النشر إلى `beta` فقط
- يجب أن يستخدم مسار النشر الفعلي قيمة `npm_dist_tag` نفسها المستخدمة أثناء الفحص التمهيدي؛
  ويتحقق سير العمل من تلك البيانات الوصفية قبل متابعة النشر
- يجب أن يستخدم وضع الترقية وسمًا مستقرًا أو تصحيحيًا، و`preflight_only=false`،
  و`preflight_run_id` فارغًا، و`npm_dist_tag=beta`
- يتطلب وضع الترقية أيضًا `NPM_TOKEN` صالحًا في بيئة `npm-release`
  لأن `npm dist-tag add` ما زال يحتاج إلى مصادقة npm العادية

## تسلسل إصدار npm المستقر

عند إصدار إصدار npm مستقر:

1. شغّل `OpenClaw NPM Release` مع `preflight_only=true`
2. اختر `npm_dist_tag=beta` لتدفق beta-first العادي، أو `latest` فقط
   عندما تريد عمدًا نشر إصدار مستقر مباشر
3. احفظ `preflight_run_id` الناجح
4. شغّل `OpenClaw NPM Release` مرة أخرى مع `preflight_only=false`، و`tag`
   نفسه، و`npm_dist_tag` نفسه، و`preflight_run_id` المحفوظ
5. إذا وصل الإصدار إلى `beta`، فشغّل `OpenClaw NPM Release` لاحقًا باستخدام
   `tag` المستقر نفسه، و`promote_beta_to_latest=true`، و`preflight_only=false`،
   و`preflight_run_id` فارغ، و`npm_dist_tag=beta` عندما تريد نقل تلك
   البنية المنشورة إلى `latest`

ما زال وضع الترقية يتطلب موافقة بيئة `npm-release` ووجود `NPM_TOKEN`
صالح في تلك البيئة.

ويُبقي ذلك مسار النشر المباشر ومسار الترقية من beta أولًا
موثَّقين ومرئيين للمشغّل.

## المراجع العامة

- [`.github/workflows/openclaw-npm-release.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-npm-release.yml)
- [`scripts/openclaw-npm-release-check.ts`](https://github.com/openclaw/openclaw/blob/main/scripts/openclaw-npm-release-check.ts)
- [`scripts/package-mac-dist.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/package-mac-dist.sh)
- [`scripts/make_appcast.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/make_appcast.sh)

يستخدم المشرفون وثائق الإصدار الخاصة في
[`openclaw/maintainers/release/README.md`](https://github.com/openclaw/maintainers/blob/main/release/README.md)
لدليل التشغيل الفعلي.
