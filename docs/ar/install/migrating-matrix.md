---
read_when:
    - ترقية تثبيت Matrix موجود
    - ترحيل سجل Matrix المشفر وحالة الجهاز
summary: كيف يقوم OpenClaw بترقية plugin Matrix السابقة في مكانها، بما في ذلك حدود استرداد الحالة المشفرة وخطوات الاسترداد اليدوي.
title: ترحيل Matrix
x-i18n:
    generated_at: "2026-04-05T12:48:13Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7b1ade057d90a524e09756bd981921988c980ea6259f5c4316a796a831e9f83b
    source_path: install/migrating-matrix.md
    workflow: 15
---

# ترحيل Matrix

تغطي هذه الصفحة الترقيات من plugin ‏`matrix` العامة السابقة إلى التنفيذ الحالي.

بالنسبة إلى معظم المستخدمين، تتم الترقية في مكانها:

- يبقى plugin هو `@openclaw/matrix`
- تبقى القناة هي `matrix`
- يبقى التكوين تحت `channels.matrix`
- تبقى بيانات الاعتماد المؤقتة تحت `~/.openclaw/credentials/matrix/`
- تبقى حالة وقت التشغيل تحت `~/.openclaw/matrix/`

لا تحتاج إلى إعادة تسمية مفاتيح التكوين أو إعادة تثبيت plugin باسم جديد.

## ما الذي يفعله الترحيل تلقائيًا

عند بدء تشغيل gateway، وعند تشغيل [`openclaw doctor --fix`](/gateway/doctor)، يحاول OpenClaw إصلاح حالة Matrix القديمة تلقائيًا.
وقبل أن تقوم أي خطوة ترحيل Matrix قابلة للتنفيذ بتعديل الحالة على القرص، ينشئ OpenClaw لقطة استرداد مركزة أو يعيد استخدامها.

عند استخدام `openclaw update`، يعتمد المشغّل الدقيق على طريقة تثبيت OpenClaw:

- تقوم التثبيتات من المصدر بتشغيل `openclaw doctor --fix` أثناء تدفق التحديث، ثم تعيد تشغيل gateway افتراضيًا
- تقوم التثبيتات عبر مديري الحزم بتحديث الحزمة، وتشغيل doctor غير تفاعلي، ثم تعتمد على إعادة تشغيل gateway الافتراضية حتى يتمكن بدء التشغيل من إنهاء ترحيل Matrix
- إذا استخدمت `openclaw update --no-restart`، فسيتم تأجيل ترحيل Matrix المدعوم ببدء التشغيل إلى أن تشغّل لاحقًا `openclaw doctor --fix` وتعيد تشغيل gateway

يغطي الترحيل التلقائي ما يلي:

- إنشاء أو إعادة استخدام لقطة ما قبل الترحيل تحت `~/Backups/openclaw-migrations/`
- إعادة استخدام بيانات اعتماد Matrix المخزنة مؤقتًا
- الاحتفاظ بتحديد الحساب نفسه وتكوين `channels.matrix`
- نقل أقدم مخزن مزامنة Matrix المسطح إلى الموقع الحالي ذي النطاق الخاص بالحساب
- نقل أقدم مخزن crypto المسطح لـ Matrix إلى الموقع الحالي ذي النطاق الخاص بالحساب عندما يمكن حل الحساب الهدف بأمان
- استخراج مفتاح فك تشفير نسخة Matrix الاحتياطية من مفاتيح الغرف المحفوظ مسبقًا من مخزن rust crypto القديم، عندما يكون هذا المفتاح موجودًا محليًا
- إعادة استخدام جذر تخزين token-hash الحالي الأكثر اكتمالًا للحساب نفسه في Matrix، وhomeserver، والمستخدم نفسه عندما يتغير access token لاحقًا
- فحص جذور تخزين token-hash المجاورة بحثًا عن بيانات تعريف لاستعادة حالة مشفرة معلقة عندما يتغير access token في Matrix لكن تبقى هوية الحساب/الجهاز كما هي
- استعادة مفاتيح الغرف الاحتياطية إلى مخزن crypto الجديد عند بدء تشغيل Matrix التالي

تفاصيل اللقطة:

- يكتب OpenClaw ملف marker في `~/.openclaw/matrix/migration-snapshot.json` بعد نجاح اللقطة حتى تتمكن عمليات بدء التشغيل والإصلاح اللاحقة من إعادة استخدام الأرشيف نفسه.
- تقوم لقطات ترحيل Matrix التلقائية هذه بعمل نسخة احتياطية من التكوين + الحالة فقط (`includeWorkspace: false`).
- إذا كانت Matrix تحتوي فقط على حالة ترحيل تحذيرية، مثل أن `userId` أو `accessToken` ما زال مفقودًا، فلن ينشئ OpenClaw اللقطة بعد لأنه لا توجد أي عملية تعديل Matrix قابلة للتنفيذ.
- إذا فشلت خطوة اللقطة، يتخطى OpenClaw ترحيل Matrix في تلك الجولة بدلًا من تعديل الحالة من دون نقطة استرداد.

حول الترقيات متعددة الحسابات:

- جاء أقدم مخزن Matrix مسطح (`~/.openclaw/matrix/bot-storage.json` و`~/.openclaw/matrix/crypto/`) من تخطيط مخزن واحد، لذلك لا يمكن لـ OpenClaw ترحيله إلا إلى هدف حساب Matrix واحد محلول
- يتم اكتشاف مخازن Matrix القديمة ذات النطاق لكل حساب بالفعل وتجهيزها لكل حساب Matrix مكوَّن

## ما الذي لا يمكن للترحيل فعله تلقائيًا

لم تقم plugin ‏Matrix العامة السابقة **بإنشاء نسخ احتياطية من مفاتيح غرف Matrix تلقائيًا**. فقد كانت تحتفظ بحالة crypto المحلية وتطلب التحقق من الجهاز، لكنها لم تضمن أن تكون مفاتيح الغرف لديك قد حُفظت احتياطيًا على homeserver.

وهذا يعني أن بعض التثبيتات المشفرة لا يمكن ترحيلها إلا جزئيًا.

لا يستطيع OpenClaw استرداد ما يلي تلقائيًا:

- مفاتيح الغرف المحلية فقط التي لم تُحفظ احتياطيًا مطلقًا
- الحالة المشفرة عندما لا يمكن حل حساب Matrix الهدف بعد لأن `homeserver` أو `userId` أو `accessToken` غير متاحة بعد
- الترحيل التلقائي لمخزن Matrix مسطح مشترك واحد عندما تكون عدة حسابات Matrix مكوّنة لكن `channels.matrix.defaultAccount` غير معيّن
- تثبيتات plugin المخصصة عبر مسار مخصص والمثبّتة إلى مسار مستودع بدلًا من حزمة Matrix القياسية
- مفتاح استرداد مفقود عندما كان المخزن القديم يحتوي على مفاتيح محفوظة احتياطيًا لكنه لم يحتفظ بمفتاح فك التشفير محليًا

نطاق التحذير الحالي:

- يتم إظهار تثبيتات plugin ‏Matrix المخصصة عبر المسار من كل من بدء تشغيل gateway و`openclaw doctor`

إذا كان تثبيتك القديم يحتوي على سجل مشفر محلي فقط ولم يُحفظ احتياطيًا مطلقًا، فقد تظل بعض الرسائل المشفرة الأقدم غير قابلة للقراءة بعد الترقية.

## تدفق الترقية الموصى به

1. حدّث OpenClaw وplugin ‏Matrix بشكل عادي.
   يفضّل استخدام `openclaw update` العادي من دون `--no-restart` حتى يتمكن بدء التشغيل من إنهاء ترحيل Matrix فورًا.
2. شغّل:

   ```bash
   openclaw doctor --fix
   ```

   إذا كان لدى Matrix عمل ترحيل قابل للتنفيذ، فسيقوم doctor بإنشاء أو إعادة استخدام لقطة ما قبل الترحيل أولًا ويطبع مسار الأرشيف.

3. ابدأ أو أعد تشغيل gateway.
4. تحقّق من حالة التحقق والنسخ الاحتياطي الحالية:

   ```bash
   openclaw matrix verify status
   openclaw matrix verify backup status
   ```

5. إذا أخبرك OpenClaw بأن مفتاح الاسترداد مطلوب، فشغّل:

   ```bash
   openclaw matrix verify backup restore --recovery-key "<your-recovery-key>"
   ```

6. إذا كان هذا الجهاز ما يزال غير متحقق، فشغّل:

   ```bash
   openclaw matrix verify device "<your-recovery-key>"
   ```

7. إذا كنت تتخلى عمدًا عن السجل القديم غير القابل للاسترداد وتريد خط أساس جديدًا للنسخ الاحتياطي للرسائل المستقبلية، فشغّل:

   ```bash
   openclaw matrix verify backup reset --yes
   ```

8. إذا لم توجد نسخة احتياطية من المفاتيح على الخادم بعد، فأنشئ واحدة لعمليات الاسترداد المستقبلية:

   ```bash
   openclaw matrix verify bootstrap
   ```

## كيف يعمل ترحيل الحالة المشفرة

ترحيل الحالة المشفرة هو عملية ذات مرحلتين:

1. يقوم بدء التشغيل أو `openclaw doctor --fix` بإنشاء أو إعادة استخدام لقطة ما قبل الترحيل إذا كان ترحيل الحالة المشفرة قابلاً للتنفيذ.
2. يقوم بدء التشغيل أو `openclaw doctor --fix` بفحص مخزن crypto القديم لـ Matrix عبر تثبيت plugin ‏Matrix النشط.
3. إذا تم العثور على مفتاح لفك تشفير النسخة الاحتياطية، يكتبه OpenClaw في تدفق recovery-key الجديد ويضع علامة على أن استعادة مفاتيح الغرف معلقة.
4. عند بدء تشغيل Matrix التالي، يعيد OpenClaw تلقائيًا مفاتيح الغرف الاحتياطية إلى مخزن crypto الجديد.

إذا أبلغ المخزن القديم عن مفاتيح غرف لم تُحفظ احتياطيًا من قبل، فإن OpenClaw يصدر تحذيرًا بدلًا من التظاهر بنجاح الاسترداد.

## الرسائل الشائعة ومعناها

### رسائل الترقية والاكتشاف

`Matrix plugin upgraded in place.`

- المعنى: تم اكتشاف حالة Matrix القديمة على القرص وترحيلها إلى التخطيط الحالي.
- ما يجب فعله: لا شيء إلا إذا كان الخرج نفسه يتضمن تحذيرات أيضًا.

`Matrix migration snapshot created before applying Matrix upgrades.`

- المعنى: أنشأ OpenClaw أرشيف استرداد قبل تعديل حالة Matrix.
- ما يجب فعله: احتفظ بمسار الأرشيف المطبوع حتى تؤكد نجاح الترحيل.

`Matrix migration snapshot reused before applying Matrix upgrades.`

- المعنى: عثر OpenClaw على marker موجود مسبقًا للقطة ترحيل Matrix وأعاد استخدام ذلك الأرشيف بدلًا من إنشاء نسخة احتياطية مكررة.
- ما يجب فعله: احتفظ بمسار الأرشيف المطبوع حتى تؤكد نجاح الترحيل.

`Legacy Matrix state detected at ... but channels.matrix is not configured yet.`

- المعنى: توجد حالة Matrix قديمة، لكن OpenClaw لا يستطيع ربطها بحساب Matrix الحالي لأن Matrix غير مكوّنة.
- ما يجب فعله: قم بتكوين `channels.matrix`، ثم أعد تشغيل `openclaw doctor --fix` أو أعد تشغيل gateway.

`Legacy Matrix state detected at ... but the new account-scoped target could not be resolved yet (need homeserver, userId, and access token for channels.matrix...).`

- المعنى: عثر OpenClaw على حالة قديمة، لكنه لا يزال لا يستطيع تحديد الجذر الحالي الدقيق للحساب/الجهاز.
- ما يجب فعله: ابدأ تشغيل gateway مرة واحدة باستخدام تسجيل دخول Matrix صالح، أو أعد تشغيل `openclaw doctor --fix` بعد توفر بيانات الاعتماد المؤقتة.

`Legacy Matrix state detected at ... but multiple Matrix accounts are configured and channels.matrix.defaultAccount is not set.`

- المعنى: عثر OpenClaw على مخزن Matrix مسطح مشترك واحد، لكنه يرفض التخمين أي حساب Matrix مسمى يجب أن يتلقاه.
- ما يجب فعله: عيّن `channels.matrix.defaultAccount` إلى الحساب المقصود، ثم أعد تشغيل `openclaw doctor --fix` أو أعد تشغيل gateway.

`Matrix legacy sync store not migrated because the target already exists (...)`

- المعنى: يحتوي الموقع الجديد ذو النطاق الخاص بالحساب بالفعل على مخزن مزامنة أو crypto، لذلك لم يقم OpenClaw بالكتابة فوقه تلقائيًا.
- ما يجب فعله: تحقق من أن الحساب الحالي هو الحساب الصحيح قبل إزالة الهدف المتعارض أو نقله يدويًا.

`Failed migrating Matrix legacy sync store (...)` أو `Failed migrating Matrix legacy crypto store (...)`

- المعنى: حاول OpenClaw نقل حالة Matrix القديمة لكن عملية نظام الملفات فشلت.
- ما يجب فعله: افحص أذونات نظام الملفات وحالة القرص، ثم أعد تشغيل `openclaw doctor --fix`.

`Legacy Matrix encrypted state detected at ... but channels.matrix is not configured yet.`

- المعنى: عثر OpenClaw على مخزن Matrix مشفر قديم، لكن لا يوجد تكوين Matrix حالي لإرفاقه به.
- ما يجب فعله: قم بتكوين `channels.matrix`، ثم أعد تشغيل `openclaw doctor --fix` أو أعد تشغيل gateway.

`Legacy Matrix encrypted state detected at ... but the account-scoped target could not be resolved yet (need homeserver, userId, and access token for channels.matrix...).`

- المعنى: يوجد المخزن المشفر، لكن OpenClaw لا يستطيع أن يقرر بأمان إلى أي حساب/جهاز حالي ينتمي.
- ما يجب فعله: ابدأ تشغيل gateway مرة واحدة باستخدام تسجيل دخول Matrix صالح، أو أعد تشغيل `openclaw doctor --fix` بعد توفر بيانات الاعتماد المؤقتة.

`Legacy Matrix encrypted state detected at ... but multiple Matrix accounts are configured and channels.matrix.defaultAccount is not set.`

- المعنى: عثر OpenClaw على مخزن crypto قديم مسطح مشترك واحد، لكنه يرفض التخمين أي حساب Matrix مسمى يجب أن يتلقاه.
- ما يجب فعله: عيّن `channels.matrix.defaultAccount` إلى الحساب المقصود، ثم أعد تشغيل `openclaw doctor --fix` أو أعد تشغيل gateway.

`Matrix migration warnings are present, but no on-disk Matrix mutation is actionable yet. No pre-migration snapshot was needed.`

- المعنى: اكتشف OpenClaw حالة Matrix قديمة، لكن الترحيل ما يزال محظورًا بسبب بيانات هوية أو اعتماد مفقودة.
- ما يجب فعله: أكمل تسجيل دخول Matrix أو إعداد التكوين، ثم أعد تشغيل `openclaw doctor --fix` أو أعد تشغيل gateway.

`Legacy Matrix encrypted state was detected, but the Matrix plugin helper is unavailable. Install or repair @openclaw/matrix so OpenClaw can inspect the old rust crypto store before upgrading.`

- المعنى: عثر OpenClaw على حالة Matrix مشفرة قديمة، لكنه لم يتمكن من تحميل نقطة الدخول المساعدة من plugin ‏Matrix التي تفحص ذلك المخزن عادةً.
- ما يجب فعله: أعد تثبيت أو إصلاح plugin ‏Matrix ‏(`openclaw plugins install @openclaw/matrix`، أو `openclaw plugins install ./path/to/local/matrix-plugin` لنسخة مستودع)، ثم أعد تشغيل `openclaw doctor --fix` أو أعد تشغيل gateway.

`Matrix plugin helper path is unsafe: ... Reinstall @openclaw/matrix and try again.`

- المعنى: عثر OpenClaw على مسار ملف مساعد يهرب من جذر plugin أو يفشل في فحوصات حدود plugin، ولذلك رفض استيراده.
- ما يجب فعله: أعد تثبيت plugin ‏Matrix من مسار موثوق، ثم أعد تشغيل `openclaw doctor --fix` أو أعد تشغيل gateway.

`- Failed creating a Matrix migration snapshot before repair: ...`

`- Skipping Matrix migration changes for now. Resolve the snapshot failure, then rerun "openclaw doctor --fix".`

- المعنى: رفض OpenClaw تعديل حالة Matrix لأنه لم يتمكن من إنشاء لقطة الاسترداد أولًا.
- ما يجب فعله: أصلح خطأ النسخ الاحتياطي، ثم أعد تشغيل `openclaw doctor --fix` أو أعد تشغيل gateway.

`Failed migrating legacy Matrix client storage: ...`

- المعنى: عثر بديل عميل Matrix على تخزين قديم مسطح، لكن النقل فشل. ويقوم OpenClaw الآن بإيقاف ذلك البديل بدلًا من البدء بصمت بمخزن جديد.
- ما يجب فعله: افحص أذونات نظام الملفات أو التعارضات، وأبقِ الحالة القديمة سليمة، ثم أعد المحاولة بعد إصلاح الخطأ.

`Matrix is installed from a custom path: ...`

- المعنى: Matrix مثبّتة عبر مسار مخصص، لذلك لا تستبدل تحديثات mainline هذا التثبيت تلقائيًا بحزمة Matrix القياسية في المستودع.
- ما يجب فعله: أعد التثبيت باستخدام `openclaw plugins install @openclaw/matrix` عندما تريد العودة إلى plugin ‏Matrix الافتراضية.

### رسائل استرداد الحالة المشفرة

`matrix: restored X/Y room key(s) from legacy encrypted-state backup`

- المعنى: تمت استعادة مفاتيح الغرف الاحتياطية بنجاح إلى مخزن crypto الجديد.
- ما يجب فعله: عادة لا شيء.

`matrix: N legacy local-only room key(s) were never backed up and could not be restored automatically`

- المعنى: كانت بعض مفاتيح الغرف القديمة موجودة فقط في المخزن المحلي القديم ولم تُرفع أبدًا إلى النسخة الاحتياطية في Matrix.
- ما يجب فعله: توقّع أن يبقى بعض السجل المشفر القديم غير متاح ما لم تتمكن من استرداد تلك المفاتيح يدويًا من عميل Matrix آخر متحقق.

`Legacy Matrix encrypted state for account "..." has backed-up room keys, but no local backup decryption key was found. Ask the operator to run "openclaw matrix verify backup restore --recovery-key <key>" after upgrade if they have the recovery key.`

- المعنى: النسخة الاحتياطية موجودة، لكن OpenClaw لم يتمكن من استرداد مفتاح الاسترداد تلقائيًا.
- ما يجب فعله: شغّل `openclaw matrix verify backup restore --recovery-key "<your-recovery-key>"`.

`Failed inspecting legacy Matrix encrypted state for account "..." (...): ...`

- المعنى: عثر OpenClaw على المخزن المشفر القديم، لكنه لم يتمكن من فحصه بشكل آمن بما يكفي لتحضير الاسترداد.
- ما يجب فعله: أعد تشغيل `openclaw doctor --fix`. وإذا تكرر الأمر، فأبقِ دليل الحالة القديم كما هو واسترد باستخدام عميل Matrix آخر متحقق بالإضافة إلى `openclaw matrix verify backup restore --recovery-key "<your-recovery-key>"`.

`Legacy Matrix backup key was found for account "...", but .../recovery-key.json already contains a different recovery key. Leaving the existing file unchanged.`

- المعنى: اكتشف OpenClaw تعارضًا في مفتاح النسخة الاحتياطية ورفض الكتابة فوق ملف recovery-key الحالي تلقائيًا.
- ما يجب فعله: تحقق من مفتاح الاسترداد الصحيح قبل إعادة محاولة أي أمر استرداد.

`Legacy Matrix encrypted state for account "..." cannot be fully converted automatically because the old rust crypto store does not expose all local room keys for export.`

- المعنى: هذا هو الحد الصلب لتنسيق التخزين القديم.
- ما يجب فعله: لا يزال يمكن استعادة المفاتيح المحفوظة احتياطيًا، لكن السجل المشفر المحلي فقط قد يظل غير متاح.

`matrix: failed restoring room keys from legacy encrypted-state backup: ...`

- المعنى: حاولت plugin الجديدة تنفيذ الاستعادة لكن Matrix أعادت خطأ.
- ما يجب فعله: شغّل `openclaw matrix verify backup status`، ثم أعد المحاولة باستخدام `openclaw matrix verify backup restore --recovery-key "<your-recovery-key>"` إذا لزم الأمر.

### رسائل الاسترداد اليدوي

`Backup key is not loaded on this device. Run 'openclaw matrix verify backup restore' to load it and restore old room keys.`

- المعنى: يعرف OpenClaw أنه من المفترض أن تملك مفتاح نسخة احتياطية، لكنه غير نشط على هذا الجهاز.
- ما يجب فعله: شغّل `openclaw matrix verify backup restore`، أو مرّر `--recovery-key` إذا لزم الأمر.

`Store a recovery key with 'openclaw matrix verify device <key>', then run 'openclaw matrix verify backup restore'.`

- المعنى: لا يحتوي هذا الجهاز حاليًا على مفتاح الاسترداد المخزن.
- ما يجب فعله: تحقّق من الجهاز باستخدام مفتاح الاسترداد أولًا، ثم استعد النسخة الاحتياطية.

`Backup key mismatch on this device. Re-run 'openclaw matrix verify device <key>' with the matching recovery key.`

- المعنى: لا يطابق المفتاح المخزن النسخة الاحتياطية النشطة لـ Matrix.
- ما يجب فعله: أعد تشغيل `openclaw matrix verify device "<your-recovery-key>"` باستخدام المفتاح الصحيح.

إذا قبلت فقدان السجل المشفر القديم غير القابل للاسترداد، فيمكنك بدلًا من ذلك إعادة تعيين
خط أساس النسخة الاحتياطية الحالي باستخدام `openclaw matrix verify backup reset --yes`. وعندما يكون
سر النسخة الاحتياطية المخزن معطوبًا، فقد تعيد عملية الإعادة تلك أيضًا إنشاء تخزين الأسرار بحيث
يمكن تحميل مفتاح النسخة الاحتياطية الجديد بشكل صحيح بعد إعادة التشغيل.

`Backup trust chain is not verified on this device. Re-run 'openclaw matrix verify device <key>'.`

- المعنى: النسخة الاحتياطية موجودة، لكن هذا الجهاز لا يثق بعد بسلسلة التوقيع المتبادل بدرجة كافية.
- ما يجب فعله: أعد تشغيل `openclaw matrix verify device "<your-recovery-key>"`.

`Matrix recovery key is required`

- المعنى: حاولت تنفيذ خطوة استرداد من دون توفير مفتاح استرداد عندما كان مطلوبًا.
- ما يجب فعله: أعد تشغيل الأمر مع مفتاح الاسترداد الخاص بك.

`Invalid Matrix recovery key: ...`

- المعنى: تعذر تحليل المفتاح المقدم أو لم يطابق التنسيق المتوقع.
- ما يجب فعله: أعد المحاولة باستخدام مفتاح الاسترداد الدقيق من عميل Matrix أو من ملف recovery-key.

`Matrix device is still unverified after applying recovery key. Verify your recovery key and ensure cross-signing is available.`

- المعنى: تم تطبيق المفتاح، لكن الجهاز ما زال غير قادر على إكمال التحقق.
- ما يجب فعله: تأكد من أنك استخدمت المفتاح الصحيح وأن cross-signing متاح على الحساب، ثم أعد المحاولة.

`Matrix key backup is not active on this device after loading from secret storage.`

- المعنى: لم ينتج عن تخزين الأسرار جلسة نسخ احتياطي نشطة على هذا الجهاز.
- ما يجب فعله: تحقّق من الجهاز أولًا، ثم أعد الفحص باستخدام `openclaw matrix verify backup status`.

`Matrix crypto backend cannot load backup keys from secret storage. Verify this device with 'openclaw matrix verify device <key>' first.`

- المعنى: لا يستطيع هذا الجهاز الاستعادة من تخزين الأسرار حتى يكتمل التحقق من الجهاز.
- ما يجب فعله: شغّل `openclaw matrix verify device "<your-recovery-key>"` أولًا.

### رسائل تثبيت plugin المخصصة

`Matrix is installed from a custom path that no longer exists: ...`

- المعنى: يشير سجل تثبيت plugin لديك إلى مسار محلي لم يعد موجودًا.
- ما يجب فعله: أعد التثبيت باستخدام `openclaw plugins install @openclaw/matrix`، أو إذا كنت تعمل من نسخة مستودع، فاستخدم `openclaw plugins install ./path/to/local/matrix-plugin`.

## إذا لم يعد السجل المشفر بعد

شغّل هذه الفحوصات بالترتيب:

```bash
openclaw matrix verify status --verbose
openclaw matrix verify backup status --verbose
openclaw matrix verify backup restore --recovery-key "<your-recovery-key>" --verbose
```

إذا تمت استعادة النسخة الاحتياطية بنجاح لكن بعض الغرف القديمة ما زالت تفتقد إلى السجل، فمن المرجح أن تلك المفاتيح المفقودة لم تُحفظ احتياطيًا مطلقًا بواسطة plugin السابقة.

## إذا كنت تريد البدء من جديد للرسائل المستقبلية

إذا كنت تقبل فقدان السجل المشفر القديم غير القابل للاسترداد وتريد فقط خط أساس نظيفًا للنسخ الاحتياطي مستقبلًا، فشغّل هذه الأوامر بالترتيب:

```bash
openclaw matrix verify backup reset --yes
openclaw matrix verify backup status --verbose
openclaw matrix verify status
```

إذا كان الجهاز ما يزال غير متحقق بعد ذلك، فأكمل التحقق من عميل Matrix الخاص بك بمقارنة رموز SAS التعبيرية أو الرموز العشرية والتأكد من تطابقها.

## صفحات ذات صلة

- [Matrix](/channels/matrix)
- [Doctor](/gateway/doctor)
- [الترحيل](/install/migrating)
- [Plugins](/tools/plugin)
