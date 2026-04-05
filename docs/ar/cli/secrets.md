---
read_when:
    - إعادة حل مراجع الأسرار في وقت التشغيل
    - تدقيق بقايا النص الصريح والمراجع غير المحلولة
    - تكوين SecretRefs وتطبيق تغييرات تنظيف أحادية الاتجاه
summary: مرجع CLI للأمر `openclaw secrets` ‏(reload وaudit وconfigure وapply)
title: secrets
x-i18n:
    generated_at: "2026-04-05T12:39:30Z"
    model: gpt-5.4
    provider: openai
    source_hash: f436ba089d752edb766c0a3ce746ee6bca1097b22c9b30e3d9715cb0bb50bf47
    source_path: cli/secrets.md
    workflow: 15
---

# `openclaw secrets`

استخدم `openclaw secrets` لإدارة SecretRefs والحفاظ على سلامة لقطة وقت التشغيل النشطة.

أدوار الأوامر:

- `reload`: ‏Gateway RPC ‏(`secrets.reload`) الذي يعيد حل المراجع ويبدّل لقطة وقت التشغيل فقط عند النجاح الكامل (من دون كتابة أي تكوين).
- `audit`: فحص للقراءة فقط للتكوين/المصادقة/مخازن النماذج المُولدة وبقايا الأنظمة القديمة بحثًا عن النص الصريح، والمراجع غير المحلولة، وانحراف الأولوية (يتم تخطي مراجع exec ما لم يتم تعيين `--allow-exec`).
- `configure`: مخطط تفاعلي لإعداد المزوّد، وربط الأهداف، والفحص المسبق (يتطلب TTY).
- `apply`: تنفيذ خطة محفوظة (`--dry-run` للتحقق فقط؛ يتخطى التشغيل الجاف فحوصات exec افتراضيًا، بينما يرفض وضع الكتابة الخطط التي تحتوي على exec ما لم يتم تعيين `--allow-exec`)، ثم تنظيف بقايا النص الصريح المستهدفة.

حلقة التشغيل الموصى بها:

```bash
openclaw secrets audit --check
openclaw secrets configure
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json
openclaw secrets audit --check
openclaw secrets reload
```

إذا كانت خطتك تتضمن `exec` SecretRefs/providers، فمرّر `--allow-exec` في كل من أوامر التشغيل الجاف وأوامر التطبيق في وضع الكتابة.

ملاحظة حول رمز الخروج في CI/البوابات:

- يعيد `audit --check` القيمة `1` عند وجود نتائج.
- تعيد المراجع غير المحلولة القيمة `2`.

ذو صلة:

- دليل الأسرار: [إدارة الأسرار](/gateway/secrets)
- سطح بيانات الاعتماد: [سطح بيانات اعتماد SecretRef](/reference/secretref-credential-surface)
- دليل الأمان: [الأمان](/gateway/security)

## إعادة تحميل لقطة وقت التشغيل

إعادة حل مراجع الأسرار وتبديل لقطة وقت التشغيل بشكل ذري.

```bash
openclaw secrets reload
openclaw secrets reload --json
openclaw secrets reload --url ws://127.0.0.1:18789 --token <token>
```

ملاحظات:

- يستخدم طريقة Gateway RPC المسماة `secrets.reload`.
- إذا فشل الحل، يحتفظ gateway بآخر لقطة سليمة معروفة ويعيد خطأً (من دون تفعيل جزئي).
- يتضمن رد JSON الحقل `warningCount`.

الخيارات:

- `--url <url>`
- `--token <token>`
- `--timeout <ms>`
- `--json`

## التدقيق

فحص حالة OpenClaw بحثًا عن:

- تخزين الأسرار بنص صريح
- المراجع غير المحلولة
- انحراف الأولوية (حيث تقوم بيانات اعتماد `auth-profiles.json` بحجب مراجع `openclaw.json`)
- بقايا `agents/*/agent/models.json` المُولدة (قيم `apiKey` الخاصة بالمزوّد والرؤوس الحساسة الخاصة بالمزوّد)
- بقايا الأنظمة القديمة (إدخالات مخزن المصادقة القديم، وتذكيرات OAuth)

ملاحظة بقايا الرؤوس:

- يعتمد اكتشاف الرؤوس الحساسة الخاصة بالمزوّد على الاستدلال بالاسم (أسماء رؤوس المصادقة/بيانات الاعتماد الشائعة والأجزاء مثل `authorization` و`x-api-key` و`token` و`secret` و`password` و`credential`).

```bash
openclaw secrets audit
openclaw secrets audit --check
openclaw secrets audit --json
openclaw secrets audit --allow-exec
```

سلوك الخروج:

- يُنهي `--check` التنفيذ بقيمة غير صفرية عند وجود نتائج.
- تُنهي المراجع غير المحلولة التنفيذ برمز غير صفري ذي أولوية أعلى.

أبرز معالم شكل التقرير:

- `status`: ‏`clean | findings | unresolved`
- `resolution`: ‏`refsChecked` و`skippedExecRefs` و`resolvabilityComplete`
- `summary`: ‏`plaintextCount` و`unresolvedRefCount` و`shadowedRefCount` و`legacyResidueCount`
- رموز النتائج:
  - `PLAINTEXT_FOUND`
  - `REF_UNRESOLVED`
  - `REF_SHADOWED`
  - `LEGACY_RESIDUE`

## التكوين (مساعد تفاعلي)

أنشئ تغييرات المزوّد وSecretRef بشكل تفاعلي، وشغّل الفحص المسبق، وطبّقها اختياريًا:

```bash
openclaw secrets configure
openclaw secrets configure --plan-out /tmp/openclaw-secrets-plan.json
openclaw secrets configure --apply --yes
openclaw secrets configure --providers-only
openclaw secrets configure --skip-provider-setup
openclaw secrets configure --agent ops
openclaw secrets configure --json
```

التدفق:

- إعداد المزوّد أولًا (`add/edit/remove` لأسماء `secrets.providers` المستعارة).
- ربط بيانات الاعتماد ثانيًا (تحديد الحقول وتعيين مراجع `{source, provider, id}`).
- الفحص المسبق والتطبيق الاختياري أخيرًا.

العلامات:

- `--providers-only`: تكوين `secrets.providers` فقط، وتخطي ربط بيانات الاعتماد.
- `--skip-provider-setup`: تخطي إعداد المزوّد وربط بيانات الاعتماد بالمزوّدين الحاليين.
- `--agent <id>`: قصر اكتشاف الأهداف والكتابات في `auth-profiles.json` على مخزن وكيل واحد.
- `--allow-exec`: السماح بفحوصات exec SecretRef أثناء الفحص المسبق/التطبيق (قد يؤدي إلى تنفيذ أوامر المزوّد).

ملاحظات:

- يتطلب TTY تفاعليًا.
- لا يمكنك الجمع بين `--providers-only` و`--skip-provider-setup`.
- يستهدف `configure` الحقول الحاملة للأسرار في `openclaw.json` بالإضافة إلى `auth-profiles.json` ضمن نطاق الوكيل المحدد.
- يدعم `configure` إنشاء تعيينات جديدة في `auth-profiles.json` مباشرة داخل تدفق الاختيار.
- السطح المدعوم المعتمد: [سطح بيانات اعتماد SecretRef](/reference/secretref-credential-surface).
- ينفذ حلًا مسبقًا قبل التطبيق.
- إذا كان الفحص المسبق/التطبيق يتضمن مراجع exec، فأبقِ `--allow-exec` مضبوطًا في كلتا الخطوتين.
- تفعل الخطط المُولدة افتراضيًا خيارات التنظيف (`scrubEnv` و`scrubAuthProfilesForProviderTargets` و`scrubLegacyAuthJson` كلها مفعلة).
- مسار التطبيق أحادي الاتجاه للقيم النصية الصريحة التي تم تنظيفها.
- من دون `--apply`، سيظل CLI يطالبك بالسؤال `Apply this plan now?` بعد الفحص المسبق.
- مع `--apply` (ومن دون `--yes`)، يطلب CLI تأكيدًا إضافيًا غير قابل للتراجع.
- يطبع `--json` الخطة + تقرير الفحص المسبق، لكن الأمر يظل يتطلب TTY تفاعليًا.

ملاحظة أمان مزوّد exec:

- غالبًا ما تعرض تثبيتات Homebrew ملفات تنفيذية مرتبطة رمزيًا ضمن `/opt/homebrew/bin/*`.
- عيّن `allowSymlinkCommand: true` فقط عند الحاجة إلى مسارات موثوقة يديرها مدير الحزم، وأقرنه مع `trustedDirs` (مثل `["/opt/homebrew"]`).
- في Windows، إذا لم يكن تحقق ACL متاحًا لمسار مزوّد ما، فإن OpenClaw يفشل بشكل مغلق. للمسارات الموثوقة فقط، عيّن `allowInsecurePath: true` على ذلك المزوّد لتجاوز فحوصات أمان المسار.

## تطبيق خطة محفوظة

طبّق أو نفّذ فحصًا مسبقًا لخطة مُولدة سابقًا:

```bash
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --allow-exec
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run --allow-exec
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --json
```

سلوك exec:

- يتحقق `--dry-run` من الفحص المسبق من دون كتابة ملفات.
- يتم تخطي فحوصات exec SecretRef افتراضيًا في التشغيل الجاف.
- يرفض وضع الكتابة الخطط التي تحتوي على exec SecretRefs/providers ما لم يتم تعيين `--allow-exec`.
- استخدم `--allow-exec` للاشتراك في فحوصات/تنفيذ مزوّد exec في أي من الوضعين.

تفاصيل عقد الخطة (مسارات الأهداف المسموح بها، وقواعد التحقق، ودلالات الفشل):

- [عقد خطة تطبيق الأسرار](/gateway/secrets-plan-contract)

ما الذي قد يحدّثه `apply`:

- `openclaw.json` ‏(أهداف SecretRef + عمليات upsert/delete للمزوّد)
- `auth-profiles.json` ‏(تنظيف أهداف المزوّد)
- بقايا `auth.json` القديمة
- مفاتيح الأسرار المعروفة في `~/.openclaw/.env` التي تم ترحيل قيمها

## لماذا لا توجد نسخ احتياطية للتراجع

لا يقوم `secrets apply` عمدًا بكتابة نسخ احتياطية للتراجع تحتوي على قيم النص الصريح القديمة.

تأتي السلامة من فحص مسبق صارم + تطبيق شبه ذري مع استعادة داخل الذاكرة بأفضل جهد عند الفشل.

## مثال

```bash
openclaw secrets audit --check
openclaw secrets configure
openclaw secrets audit --check
```

إذا استمر `audit --check` في الإبلاغ عن نتائج نص صريح، فحدّث مسارات الأهداف المتبقية التي تم الإبلاغ عنها وأعد تشغيل التدقيق.
