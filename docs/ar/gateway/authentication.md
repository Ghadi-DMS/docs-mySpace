---
read_when:
    - تصحيح مصادقة النموذج أو انتهاء صلاحية OAuth
    - توثيق المصادقة أو تخزين بيانات الاعتماد
summary: 'مصادقة النماذج: OAuth، ومفاتيح API، وإعادة استخدام Claude CLI'
title: المصادقة
x-i18n:
    generated_at: "2026-04-05T12:42:23Z"
    model: gpt-5.4
    provider: openai
    source_hash: 1c0ceee7d10fe8d10345f32889b63425d81773f3a08d8ecd3fd88d965b207ddc
    source_path: gateway/authentication.md
    workflow: 15
---

# المصادقة (موفرو النماذج)

<Note>
تغطي هذه الصفحة مصادقة **موفري النماذج** (مفاتيح API، وOAuth، وإعادة استخدام Claude CLI). أما مصادقة **اتصال البوابة** (الرمز، وكلمة المرور، وtrusted-proxy)، فراجع [الإعداد](/gateway/configuration) و[مصادقة Trusted Proxy](/gateway/trusted-proxy-auth).
</Note>

يدعم OpenClaw كلًا من OAuth ومفاتيح API لموفري النماذج. وبالنسبة إلى
مضيفي البوابة الدائمين، تكون مفاتيح API عادةً الخيار الأكثر قابلية للتنبؤ.
كما أن تدفقات الاشتراك/OAuth مدعومة أيضًا عندما تتوافق مع نموذج حساب الموفّر لديك.

راجع [/concepts/oauth](/concepts/oauth) للحصول على تدفق OAuth الكامل وتخطيط
التخزين.
وبالنسبة إلى المصادقة المعتمدة على SecretRef (موفرو `env`/`file`/`exec`)، راجع [إدارة الأسرار](/gateway/secrets).
وبالنسبة إلى قواعد أهلية بيانات الاعتماد/رموز الأسباب المستخدمة بواسطة `models status --probe`، راجع
[دلالات بيانات اعتماد المصادقة](/auth-credential-semantics).

## الإعداد الموصى به (مفتاح API، لأي موفّر)

إذا كنت تشغّل بوابة طويلة العمر، فابدأ بمفتاح API للموفّر الذي اخترته.
وبالنسبة إلى Anthropic تحديدًا، فإن المصادقة عبر مفتاح API هي المسار الآمن. أما إعادة استخدام Claude CLI
فهي المسار الآخر المدعوم للإعداد على نمط الاشتراك.

1. أنشئ مفتاح API في لوحة تحكم الموفّر لديك.
2. ضعه على **مضيف البوابة** (الجهاز الذي يشغّل `openclaw gateway`).

```bash
export <PROVIDER>_API_KEY="..."
openclaw models status
```

3. إذا كانت البوابة تعمل تحت systemd/launchd، فيُفضَّل وضع المفتاح في
   `~/.openclaw/.env` حتى يتمكن daemon من قراءته:

```bash
cat >> ~/.openclaw/.env <<'EOF'
<PROVIDER>_API_KEY=...
EOF
```

ثم أعد تشغيل daemon (أو أعد تشغيل عملية البوابة) وتحقق مرة أخرى:

```bash
openclaw models status
openclaw doctor
```

إذا كنت تفضل عدم إدارة متغيرات البيئة بنفسك، فيمكن للإعداد الأولي تخزين
مفاتيح API لاستخدام daemon: `openclaw onboard`.

راجع [المساعدة](/help) للحصول على تفاصيل حول وراثة البيئة (`env.shellEnv`,
و`~/.openclaw/.env`، وsystemd/launchd).

## Anthropic: توافق الرموز القديمة

لا تزال مصادقة Anthropic setup-token متاحة في OpenClaw
كمسار قديم/يدوي. ولا تزال وثائق Claude Code العامة من Anthropic تغطي الاستخدام المباشر
لـ Claude Code في الطرفية ضمن خطط Claude، لكن Anthropic أخبرت مستخدمي
OpenClaw بشكل منفصل أن مسار تسجيل الدخول إلى Claude في **OpenClaw** يُحتسب
كاستخدام harness تابع لجهة خارجية ويتطلب **Extra Usage** تتم فوترته بشكل منفصل عن
الاشتراك.

للحصول على أوضح مسار إعداد، استخدم مفتاح Anthropic API أو انتقل إلى Claude CLI
على مضيف البوابة.

الإدخال اليدوي للرمز (أي موفّر؛ يكتب إلى `auth-profiles.json` ويحدّث الإعداد):

```bash
openclaw models auth paste-token --provider openrouter
```

كما أن مراجع ملفات تعريف المصادقة مدعومة أيضًا لبيانات الاعتماد الثابتة:

- يمكن لبيانات اعتماد `api_key` استخدام `keyRef: { source, provider, id }`
- يمكن لبيانات اعتماد `token` استخدام `tokenRef: { source, provider, id }`
- لا تدعم ملفات التعريف ذات وضع OAuth بيانات اعتماد SecretRef؛ فإذا كان `auth.profiles.<id>.mode` مضبوطًا على `"oauth"`، فسيتم رفض إدخال `keyRef`/`tokenRef` المدعوم بـ SecretRef لذلك الملف التعريفي.

فحص مناسب للأتمتة (الخروج بـ `1` عند الانتهاء/الغياب، و`2` عند قرب الانتهاء):

```bash
openclaw models status --check
```

فحوصات المصادقة الحية:

```bash
openclaw models status --probe
```

ملاحظات:

- يمكن أن تأتي صفوف الفحص من ملفات تعريف المصادقة، أو بيانات اعتماد البيئة، أو `models.json`.
- إذا حذف `auth.order.<provider>` الصريح ملفًا تعريفيًا مخزنًا، فإن تقرير الفحص
  يعرض `excluded_by_auth_order` لذلك الملف بدلًا من محاولة استخدامه.
- إذا كانت المصادقة موجودة لكن OpenClaw لا يستطيع تحليل مرشح نموذج قابل للفحص لذلك
  الموفّر، فإن تقرير الفحص يعرض `status: no_model`.
- يمكن أن تكون فترات التهدئة الخاصة بحدود المعدل مرتبطة بالنموذج. وقد يظل الملف التعريفي
  الذي في فترة تهدئة لنموذج ما صالحًا للاستخدام مع نموذج شقيق على الموفّر نفسه.

يتم توثيق نصوص التشغيل الاختيارية (systemd/Termux) هنا:
[نصوص مراقبة المصادقة](/help/scripts#auth-monitoring-scripts)

## Anthropic: الترحيل إلى Claude CLI

إذا كان Claude CLI مثبتًا بالفعل ومسجل الدخول على مضيف البوابة، فيمكنك
تحويل إعداد Anthropic الحالي إلى خلفية CLI. وهذا
مسار ترحيل مدعوم في OpenClaw لإعادة استخدام تسجيل دخول Claude CLI المحلي على
ذلك المضيف.

المتطلبات الأساسية:

- تثبيت `claude` على مضيف البوابة
- Claude CLI مسجل الدخول بالفعل هناك عبر `claude auth login`

```bash
openclaw models auth login --provider anthropic --method cli --set-default
```

يحتفظ هذا بملفات تعريف المصادقة الحالية الخاصة بـ Anthropic من أجل التراجع، لكنه يغيّر
اختيار النموذج الافتراضي إلى `claude-cli/...` ويضيف إدخالات قائمة سماح Claude CLI
المطابقة ضمن `agents.defaults.models`.

تحقق من ذلك:

```bash
openclaw models status
```

اختصار الإعداد الأولي:

```bash
openclaw onboard --auth-choice anthropic-cli
```

لا يزال كل من `openclaw onboard` و`openclaw configure` التفاعليين يفضّلان Claude CLI
بالنسبة إلى Anthropic، لكن Anthropic setup-token أصبح متاحًا مجددًا كمسار
قديم/يدوي ويجب استخدامه مع توقع فوترته على أساس Extra Usage.

## التحقق من حالة مصادقة النموذج

```bash
openclaw models status
openclaw doctor
```

## سلوك تدوير مفاتيح API (البوابة)

تدعم بعض الموفّرين إعادة محاولة الطلب باستخدام مفاتيح بديلة عندما يصادف استدعاء API
حدًّا للمعدل لدى الموفّر.

- ترتيب الأولوية:
  - `OPENCLAW_LIVE_<PROVIDER>_KEY` (تجاوز مفرد)
  - `<PROVIDER>_API_KEYS`
  - `<PROVIDER>_API_KEY`
  - `<PROVIDER>_API_KEY_*`
- يتضمن موفرو Google أيضًا `GOOGLE_API_KEY` كرجوع إضافي.
- يتم إزالة التكرار من قائمة المفاتيح نفسها قبل الاستخدام.
- لا يعيد OpenClaw المحاولة باستخدام المفتاح التالي إلا عند أخطاء حدود المعدل (مثل
  `429`، أو `rate_limit`، أو `quota`، أو `resource exhausted`، أو `Too many concurrent
requests`، أو `ThrottlingException`، أو `concurrency limit reached`، أو
  `workers_ai ... quota limit exceeded`).
- لا تتم إعادة محاولة الأخطاء غير المرتبطة بحدود المعدل باستخدام مفاتيح بديلة.
- إذا فشلت جميع المفاتيح، يتم إرجاع الخطأ النهائي من آخر محاولة.

## التحكم في بيانات الاعتماد المستخدمة

### لكل جلسة (أمر دردشة)

استخدم `/model <alias-or-id>@<profileId>` لتثبيت بيانات اعتماد موفّر محددة للجلسة الحالية (أمثلة على معرّفات الملفات التعريفية: `anthropic:default` و`anthropic:work`).

استخدم `/model` (أو `/model list`) لمنتقٍ مختصر؛ واستخدم `/model status` للعرض الكامل (المرشحون + ملف تعريف المصادقة التالي، بالإضافة إلى تفاصيل endpoint الخاصة بالموفّر عند إعدادها).

### لكل وكيل (تجاوز CLI)

اضبط تجاوز ترتيب ملفات تعريف مصادقة صريحًا لوكيل (يُخزَّن في `auth-profiles.json` الخاص بذلك الوكيل):

```bash
openclaw models auth order get --provider anthropic
openclaw models auth order set --provider anthropic anthropic:default
openclaw models auth order clear --provider anthropic
```

استخدم `--agent <id>` لاستهداف وكيل محدد؛ واحذفه لاستخدام الوكيل الافتراضي المُعد.
عند تصحيح مشاكل الترتيب، يعرض `openclaw models status --probe` الملفات التعريفية
المخزنة والمحذوفة على أنها `excluded_by_auth_order` بدلًا من تجاوزها بصمت.
وعند تصحيح مشاكل التهدئة، تذكر أن فترات تهدئة حدود المعدل قد تكون مرتبطة
بمعرّف نموذج واحد بدلًا من ملف تعريف الموفّر بالكامل.

## استكشاف الأخطاء وإصلاحها

### "لم يتم العثور على بيانات اعتماد"

إذا كان ملف Anthropic التعريفي مفقودًا، فقم بترحيل ذلك الإعداد إلى Claude CLI أو مفتاح API
على **مضيف البوابة**، ثم تحقق مرة أخرى:

```bash
openclaw models status
```

### الرمز على وشك الانتهاء/منتهي الصلاحية

شغّل `openclaw models status` لتأكيد الملف التعريفي الذي تنتهي صلاحيته. وإذا كان ملف
رمز Anthropic القديم مفقودًا أو منتهي الصلاحية، فانقل ذلك الإعداد إلى Claude CLI
أو مفتاح API.

## متطلبات Claude CLI

مطلوب فقط لمسار إعادة استخدام Anthropic Claude CLI:

- تثبيت Claude Code CLI (وتوفر الأمر `claude`)
