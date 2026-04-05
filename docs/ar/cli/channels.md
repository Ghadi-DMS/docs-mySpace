---
read_when:
    - تريد إضافة/إزالة حسابات القنوات (WhatsApp/Telegram/Discord/Google Chat/Slack/Mattermost (إضافة)/Signal/iMessage/Matrix)
    - تريد التحقق من حالة القناة أو متابعة سجلات القناة
summary: مرجع CLI للأمر `openclaw channels` (الحسابات، والحالة، وتسجيل الدخول/الخروج، والسجلات)
title: channels
x-i18n:
    generated_at: "2026-04-05T12:37:59Z"
    model: gpt-5.4
    provider: openai
    source_hash: d0f558fdb5f6ec54e7fdb7a88e5c24c9d2567174341bd3ea87848bce4cba5d29
    source_path: cli/channels.md
    workflow: 15
---

# `openclaw channels`

إدارة حسابات قنوات الدردشة وحالة وقت التشغيل الخاصة بها على Gateway.

المستندات ذات الصلة:

- أدلة القنوات: [القنوات](/channels/index)
- تهيئة Gateway: [التهيئة](/gateway/configuration)

## الأوامر الشائعة

```bash
openclaw channels list
openclaw channels status
openclaw channels capabilities
openclaw channels capabilities --channel discord --target channel:123
openclaw channels resolve --channel slack "#general" "@jane"
openclaw channels logs --channel all
```

## الحالة / الإمكانات / الحل / السجلات

- `channels status`: ‏`--probe`، ‏`--timeout <ms>`، ‏`--json`
- `channels capabilities`: ‏`--channel <name>`، ‏`--account <id>` (فقط مع `--channel`)، ‏`--target <dest>`، ‏`--timeout <ms>`، ‏`--json`
- `channels resolve`: ‏`<entries...>`، ‏`--channel <name>`، ‏`--account <id>`، ‏`--kind <auto|user|group>`، ‏`--json`
- `channels logs`: ‏`--channel <name|all>`، ‏`--lines <n>`، ‏`--json`

يُعد `channels status --probe` هو المسار الحي: عند الوصول إلى gateway، فإنه يشغّل
عمليات تحقق `probeAccount` لكل حساب وعمليات `auditAccount` الاختيارية، لذلك قد يتضمن الإخراج
حالة النقل بالإضافة إلى نتائج الفحص مثل `works` أو `probe failed` أو `audit ok` أو `audit failed`.
إذا تعذر الوصول إلى gateway، يعود `channels status` إلى الملخصات المعتمدة على التهيئة فقط
بدلًا من إخراج الفحص الحي.

## إضافة / إزالة الحسابات

```bash
openclaw channels add --channel telegram --token <bot-token>
openclaw channels add --channel nostr --private-key "$NOSTR_PRIVATE_KEY"
openclaw channels remove --channel telegram --delete
```

نصيحة: يعرض `openclaw channels add --help` الوسائط الخاصة بكل قناة (الرمز المميز، والمفتاح الخاص، ورمز التطبيق، ومسارات signal-cli، وما إلى ذلك).

تتضمن أسطح الإضافة الشائعة غير التفاعلية ما يلي:

- قنوات bot-token: ‏`--token`، ‏`--bot-token`، ‏`--app-token`، ‏`--token-file`
- حقول النقل لـ Signal/iMessage: ‏`--signal-number`، ‏`--cli-path`، ‏`--http-url`، ‏`--http-host`، ‏`--http-port`، ‏`--db-path`، ‏`--service`، ‏`--region`
- حقول Google Chat: ‏`--webhook-path`، ‏`--webhook-url`، ‏`--audience-type`، ‏`--audience`
- حقول Matrix: ‏`--homeserver`، ‏`--user-id`، ‏`--access-token`، ‏`--password`، ‏`--device-name`، ‏`--initial-sync-limit`
- حقول Nostr: ‏`--private-key`، ‏`--relay-urls`
- حقول Tlon: ‏`--ship`، ‏`--url`، ‏`--code`، ‏`--group-channels`، ‏`--dm-allowlist`، ‏`--auto-discover-channels`
- ‏`--use-env` للمصادقة المعتمدة على env للحساب الافتراضي عندما تكون مدعومة

عند تشغيل `openclaw channels add` من دون وسائط، يمكن للمعالج التفاعلي أن يطلب:

- معرّفات الحسابات لكل قناة محددة
- أسماء عرض اختيارية لتلك الحسابات
- ‏`Bind configured channel accounts to agents now?`

إذا أكدت الربط الآن، يسأل المعالج عن الوكيل الذي يجب أن يملك كل حساب قناة مهيأ ويكتب روابط توجيه ضمن نطاق الحساب.

يمكنك أيضًا إدارة قواعد التوجيه نفسها لاحقًا باستخدام `openclaw agents bindings` و`openclaw agents bind` و`openclaw agents unbind` (راجع [agents](/cli/agents)).

عند إضافة حساب غير افتراضي إلى قناة ما تزال تستخدم إعدادات المستوى الأعلى لحساب واحد، يقوم OpenClaw بترقية القيم ذات النطاق الحسابي في المستوى الأعلى إلى خريطة حسابات القناة قبل كتابة الحساب الجديد. تضع معظم القنوات تلك القيم في `channels.<channel>.accounts.default`، لكن القنوات المدمجة يمكنها الحفاظ على حساب مُرقّى مطابق موجود بدلًا من ذلك. Matrix هي المثال الحالي: إذا كان هناك حساب مسمى واحد موجود بالفعل، أو كانت `defaultAccount` تشير إلى حساب مسمى موجود، فإن الترقية تحافظ على ذلك الحساب بدلًا من إنشاء `accounts.default` جديد.

يبقى سلوك التوجيه متسقًا:

- تستمر الروابط الموجودة الخاصة بالقناة فقط (من دون `accountId`) في مطابقة الحساب الافتراضي.
- لا ينشئ `channels add` الروابط أو يعيد كتابتها تلقائيًا في الوضع غير التفاعلي.
- يمكن للإعداد التفاعلي اختياريًا إضافة روابط ضمن نطاق الحساب.

إذا كانت تهيئتك بالفعل في حالة مختلطة (وجود حسابات مسماة مع بقاء قيم المستوى الأعلى الخاصة بالحساب الواحد مضبوطة)، فشغّل `openclaw doctor --fix` لنقل القيم ذات النطاق الحسابي إلى الحساب المُرقّى المختار لتلك القناة. تقوم معظم القنوات بالترقية إلى `accounts.default`؛ ويمكن لـ Matrix الحفاظ على هدف مسمى/افتراضي موجود بدلًا من ذلك.

## تسجيل الدخول / تسجيل الخروج (تفاعلي)

```bash
openclaw channels login --channel whatsapp
openclaw channels logout --channel whatsapp
```

ملاحظات:

- يدعم `channels login` الخيار `--verbose`.
- يمكن للأمرين `channels login` / `logout` استنتاج القناة عندما يكون هناك هدف تسجيل دخول مدعوم واحد فقط مهيأ.

## استكشاف الأخطاء وإصلاحها

- شغّل `openclaw status --deep` لإجراء فحص واسع.
- استخدم `openclaw doctor` للحصول على إصلاحات إرشادية.
- إذا طبع `openclaw channels list` القيمة `Claude: HTTP 403 ... user:profile` فهذا يعني أن لقطة الاستخدام تحتاج إلى النطاق `user:profile`. استخدم `--no-usage`، أو وفّر مفتاح جلسة claude.ai (`CLAUDE_WEB_SESSION_KEY` / `CLAUDE_WEB_COOKIE`)، أو أعد المصادقة عبر Claude CLI.
- يعود `openclaw channels status` إلى الملخصات المعتمدة على التهيئة فقط عندما يتعذر الوصول إلى gateway. إذا كانت بيانات اعتماد قناة مدعومة مهيأة عبر SecretRef لكنها غير متاحة في مسار الأمر الحالي، فإنه يبلّغ عن ذلك الحساب على أنه مهيأ مع ملاحظات متدهورة بدلًا من إظهاره على أنه غير مهيأ.

## فحص الإمكانات

اجلب تلميحات إمكانات الموفّر (intents/scopes عند توفرها) بالإضافة إلى دعم الميزات الثابتة:

```bash
openclaw channels capabilities
openclaw channels capabilities --channel discord --target channel:123
```

ملاحظات:

- `--channel` اختياري؛ احذفه لإدراج كل قناة (بما في ذلك الإضافات).
- يكون `--account` صالحًا فقط مع `--channel`.
- يقبل `--target` القيمة `channel:<id>` أو معرّف قناة رقمي خام، وينطبق فقط على Discord.
- تكون عمليات الفحص خاصة بالموفّر: Discord intents + أذونات القناة الاختيارية؛ Slack bot + user scopes؛ Telegram bot flags + webhook؛ Signal daemon version؛ Microsoft Teams app token + Graph roles/scopes (مع تعليقات توضيحية عند المعرفة). أما القنوات التي لا تحتوي على فحوصات فتبلّغ `Probe: unavailable`.

## حل الأسماء إلى معرّفات

حل أسماء القنوات/المستخدمين إلى معرّفات باستخدام دليل الموفّر:

```bash
openclaw channels resolve --channel slack "#general" "@jane"
openclaw channels resolve --channel discord "My Server/#support" "@someone"
openclaw channels resolve --channel matrix "Project Room"
```

ملاحظات:

- استخدم `--kind user|group|auto` لفرض نوع الهدف.
- يفضّل الحل المطابقات النشطة عندما تتشارك عدة إدخالات الاسم نفسه.
- يكون `channels resolve` للقراءة فقط. إذا كان الحساب المحدد مهيأ عبر SecretRef لكن بيانات الاعتماد تلك غير متاحة في مسار الأمر الحالي، فإن الأمر يعيد نتائج غير محلولة متدهورة مع ملاحظات بدلًا من إلغاء التشغيل بالكامل.
