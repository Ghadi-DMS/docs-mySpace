---
read_when:
    - عند تصحيح مصادقة النموذج أو انتهاء صلاحية OAuth
    - عند توثيق المصادقة أو تخزين بيانات الاعتماد
summary: 'مصادقة النماذج: OAuth ومفاتيح API وsetup-token القديم الخاص بـ Anthropic'
title: المصادقة
x-i18n:
    generated_at: "2026-04-06T03:07:32Z"
    model: gpt-5.4
    provider: openai
    source_hash: f59ede3fcd7e692ad4132287782a850526acf35474b5bfcea29e0e23610636c2
    source_path: gateway/authentication.md
    workflow: 15
---

# المصادقة (موفرو النماذج)

<Note>
تغطي هذه الصفحة مصادقة **موفر النموذج** (مفاتيح API وOAuth وsetup-token القديم الخاص بـ Anthropic). أما مصادقة **اتصال gateway** (token وكلمة المرور وtrusted-proxy)، فراجع [الإعدادات](/ar/gateway/configuration) و[مصادقة Trusted Proxy](/ar/gateway/trusted-proxy-auth).
</Note>

يدعم OpenClaw كلاً من OAuth ومفاتيح API لموفري النماذج. بالنسبة إلى
مضيفي gateway الذين يعملون دائمًا، تكون مفاتيح API عادةً الخيار الأكثر قابلية للتنبؤ.
كما أن تدفقات الاشتراك/OAuth مدعومة أيضًا عندما تتوافق مع نموذج حساب الموفر لديك.

راجع [/concepts/oauth](/ar/concepts/oauth) للاطلاع على تدفق OAuth الكامل
ومخطط التخزين.
وبالنسبة إلى المصادقة المستندة إلى SecretRef (موفرو `env`/`file`/`exec`)، راجع [إدارة الأسرار](/ar/gateway/secrets).
وللاطلاع على قواعد أهلية بيانات الاعتماد/رموز الأسباب التي يستخدمها `models status --probe`، راجع
[دلالات بيانات اعتماد المصادقة](/ar/auth-credential-semantics).

## الإعداد الموصى به (مفتاح API، أي موفر)

إذا كنت تشغّل gateway طويل الأمد، فابدأ بمفتاح API للموفر
الذي اخترته.
وبالنسبة إلى Anthropic تحديدًا، فإن مصادقة مفتاح API هي المسار الآمن. أما
المصادقة بنمط الاشتراك داخل OpenClaw فهي مسار setup-token قديم
ويجب التعامل معها باعتبارها مسار **Extra Usage**، وليس مسار حدود الخطة.

1. أنشئ مفتاح API في لوحة تحكم الموفر لديك.
2. ضعه على **مضيف gateway** (الجهاز الذي يشغّل `openclaw gateway`).

```bash
export <PROVIDER>_API_KEY="..."
openclaw models status
```

3. إذا كانت Gateway تعمل تحت systemd/launchd، فمن الأفضل وضع المفتاح في
   `~/.openclaw/.env` حتى يتمكن daemon من قراءته:

```bash
cat >> ~/.openclaw/.env <<'EOF'
<PROVIDER>_API_KEY=...
EOF
```

ثم أعد تشغيل daemon (أو أعد تشغيل عملية Gateway) وتحقق مرة أخرى:

```bash
openclaw models status
openclaw doctor
```

إذا كنت تفضل عدم إدارة متغيرات env بنفسك، يمكن لعملية الإعداد الأولي تخزين
مفاتيح API لاستخدام daemon: `openclaw onboard`.

راجع [المساعدة](/ar/help) لمعرفة تفاصيل توريث env (`env.shellEnv`،
و`~/.openclaw/.env`، وsystemd/launchd).

## Anthropic: توافق token القديم

لا تزال مصادقة setup-token الخاصة بـ Anthropic متاحة في OpenClaw باعتبارها
مسارًا قديمًا/يدويًا. ولا تزال مستندات Claude Code العامة من Anthropic
تغطي استخدام Claude Code الطرفي المباشر ضمن خطط Claude، لكن Anthropic
أبلغت مستخدمي OpenClaw بشكل منفصل أن مسار تسجيل دخول Claude في **OpenClaw**
يُحتسب على أنه استخدام harness لطرف ثالث ويتطلب **Extra Usage**
تُفوَّتر بشكل منفصل عن الاشتراك.

للحصول على أوضح مسار إعداد، استخدم مفتاح API من Anthropic. وإذا اضطررت إلى الإبقاء على
مسار Anthropic بنمط الاشتراك داخل OpenClaw، فاستخدم مسار setup-token القديم
مع توقّع أن Anthropic تتعامل معه باعتباره **Extra Usage**.

إدخال token يدويًا (أي موفر؛ يكتب `auth-profiles.json` ويحدّث الإعدادات):

```bash
openclaw models auth paste-token --provider openrouter
```

كما أن مراجع ملفات تعريف المصادقة مدعومة أيضًا لبيانات الاعتماد الثابتة:

- يمكن لبيانات اعتماد `api_key` استخدام `keyRef: { source, provider, id }`
- يمكن لبيانات اعتماد `token` استخدام `tokenRef: { source, provider, id }`
- ملفات التعريف في وضع OAuth لا تدعم بيانات الاعتماد من نوع SecretRef؛ إذا تم تعيين `auth.profiles.<id>.mode` إلى `"oauth"`، فسيتم رفض إدخال `keyRef`/`tokenRef` المستند إلى SecretRef لهذا الملف التعريفي.

فحص مناسب للأتمتة (يعيد `1` عند الانتهاء/الغياب، و`2` عند قرب الانتهاء):

```bash
openclaw models status --check
```

تحققات المصادقة الحية:

```bash
openclaw models status --probe
```

ملاحظات:

- يمكن أن تأتي صفوف probe من ملفات تعريف المصادقة أو بيانات اعتماد env أو `models.json`.
- إذا كانت `auth.order.<provider>` الصريحة تستبعد ملفًا تعريفيًا مخزنًا، فسيعرض probe
  `excluded_by_auth_order` لذلك الملف بدلًا من محاولة استخدامه.
- إذا كانت المصادقة موجودة لكن OpenClaw لا يستطيع تحديد نموذج مرشح قابل للفحص
  لذلك الموفر، فسيعرض probe `status: no_model`.
- يمكن أن تكون فترات تهدئة rate-limit مرتبطة بنطاق النموذج. وقد يظل الملف التعريفي
  الذي يمر بفترة تهدئة لنموذج واحد قابلًا للاستخدام مع نموذج شقيق على الموفر نفسه.

يتم توثيق نصوص التشغيل الاختيارية (systemd/Termux) هنا:
[نصوص مراقبة المصادقة](/ar/help/scripts#auth-monitoring-scripts)

## ملاحظة حول Anthropic

تمت إزالة الواجهة الخلفية `claude-cli` الخاصة بـ Anthropic.

- استخدم مفاتيح API من Anthropic لحركة Anthropic في OpenClaw.
- لا يزال setup-token الخاص بـ Anthropic مسارًا قديمًا/يدويًا ويجب استخدامه مع
  توقّع فوترة Extra Usage الذي أبلغت به Anthropic مستخدمي OpenClaw.
- أصبح `openclaw doctor` الآن يكتشف حالة Anthropic Claude CLI القديمة التي أُزيلت. إذا
  كانت بايتات بيانات الاعتماد المخزنة لا تزال موجودة، فسيحوّلها doctor مرة أخرى إلى
  ملفات تعريف Anthropic token/OAuth. وإذا لم تكن موجودة، فسيزيل doctor
  إعداد Claude CLI القديم ويوجهك إلى استرداد مفتاح API أو setup-token.

## التحقق من حالة مصادقة النموذج

```bash
openclaw models status
openclaw doctor
```

## سلوك تدوير مفاتيح API (gateway)

يدعم بعض الموفرين إعادة محاولة الطلب باستخدام مفاتيح بديلة عندما تصطدم مكالمة API
بحدود rate limit الخاصة بالموفر.

- ترتيب الأولوية:
  - `OPENCLAW_LIVE_<PROVIDER>_KEY` (تجاوز واحد)
  - `<PROVIDER>_API_KEYS`
  - `<PROVIDER>_API_KEY`
  - `<PROVIDER>_API_KEY_*`
- يتضمن موفرو Google أيضًا `GOOGLE_API_KEY` كبديل احتياطي إضافي.
- تتم إزالة التكرارات من قائمة المفاتيح نفسها قبل الاستخدام.
- لا يعيد OpenClaw المحاولة بالمفتاح التالي إلا في أخطاء rate limit (على سبيل المثال
  `429` أو `rate_limit` أو `quota` أو `resource exhausted` أو `Too many concurrent
requests` أو `ThrottlingException` أو `concurrency limit reached` أو
  `workers_ai ... quota limit exceeded`).
- لا تتم إعادة محاولة الأخطاء غير المتعلقة بـ rate limit باستخدام مفاتيح بديلة.
- إذا فشلت كل المفاتيح، فسيتم إرجاع الخطأ النهائي من آخر محاولة.

## التحكم في بيانات الاعتماد المستخدمة

### لكل جلسة (أمر الدردشة)

استخدم `/model <alias-or-id>@<profileId>` لتثبيت بيانات اعتماد موفر محددة للجلسة الحالية (أمثلة على معرّفات ملفات التعريف: `anthropic:default` و`anthropic:work`).

استخدم `/model` (أو `/model list`) لمنتقي مضغوط؛ واستخدم `/model status` للعرض الكامل (المرشحون + ملف تعريف المصادقة التالي، بالإضافة إلى تفاصيل endpoint الخاصة بالموفر عند تهيئتها).

### لكل وكيل (تجاوز CLI)

اضبط تجاوزًا صريحًا لترتيب ملفات تعريف المصادقة لوكيل ما (يُخزَّن في `auth-profiles.json` الخاص بذلك الوكيل):

```bash
openclaw models auth order get --provider anthropic
openclaw models auth order set --provider anthropic anthropic:default
openclaw models auth order clear --provider anthropic
```

استخدم `--agent <id>` لاستهداف وكيل محدد؛ واحذفها لاستخدام الوكيل الافتراضي المكوَّن.
وعند تصحيح مشكلات الترتيب، يعرض `openclaw models status --probe` ملفات التعريف
المخزنة المستبعدة على أنها `excluded_by_auth_order` بدلًا من تخطيها بصمت.
وعند تصحيح مشكلات التهدئة، تذكّر أن فترات تهدئة rate-limit قد تكون مرتبطة
بمعرّف نموذج واحد بدلًا من ملف تعريف الموفر بالكامل.

## استكشاف الأخطاء وإصلاحها

### "لم يتم العثور على بيانات اعتماد"

إذا كان ملف تعريف Anthropic مفقودًا، فقم بتكوين مفتاح API من Anthropic على
**مضيف gateway** أو أعد إعداد مسار setup-token القديم الخاص بـ Anthropic، ثم تحقق مرة أخرى:

```bash
openclaw models status
```

### اقتراب انتهاء صلاحية token/انتهاؤها

شغّل `openclaw models status` للتأكد من الملف التعريفي الذي توشك صلاحيته على الانتهاء. إذا كان
ملف تعريف token القديم الخاص بـ Anthropic مفقودًا أو منتهي الصلاحية، فقم بتحديث ذلك الإعداد عبر
setup-token أو انتقل إلى مفتاح API من Anthropic.

إذا كان الجهاز لا يزال يحتوي على حالة Anthropic Claude CLI قديمة ومزالة من
الإصدارات الأقدم، فشغّل:

```bash
openclaw doctor --yes
```

يقوم Doctor بتحويل `anthropic:claude-cli` مرة أخرى إلى Anthropic token/OAuth عندما
تكون بايتات بيانات الاعتماد المخزنة لا تزال موجودة. وإلا فإنه يزيل ملف تعريف/إعدادات/مراجع نماذج
Claude CLI القديمة ويترك لك إرشادات الخطوة التالية.
