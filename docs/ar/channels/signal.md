---
read_when:
    - إعداد دعم Signal
    - تصحيح إرسال/استقبال Signal
summary: دعم Signal عبر signal-cli ‏(JSON-RPC + SSE)، ومسارات الإعداد، ونموذج الأرقام
title: Signal
x-i18n:
    generated_at: "2026-04-05T12:36:40Z"
    model: gpt-5.4
    provider: openai
    source_hash: cdd855eb353aca6a1c2b04d14af0e3da079349297b54fa8243562c52b29118d9
    source_path: channels/signal.md
    workflow: 15
---

# Signal (signal-cli)

الحالة: تكامل CLI خارجي. يتواصل Gateway مع `signal-cli` عبر HTTP JSON-RPC + SSE.

## المتطلبات الأساسية

- تثبيت OpenClaw على خادمك (تم اختبار التدفق أدناه على Ubuntu 24).
- توفر `signal-cli` على المضيف الذي يعمل عليه gateway.
- رقم هاتف يمكنه استلام رسالة SMS واحدة للتحقق (لمسار التسجيل عبر SMS).
- إمكانية الوصول عبر المتصفح إلى captcha الخاصة بـ Signal (`signalcaptchas.org`) أثناء التسجيل.

## إعداد سريع (للمبتدئين)

1. استخدم **رقم Signal منفصلًا** للروبوت (موصى به).
2. ثبّت `signal-cli` (يتطلب Java إذا كنت تستخدم إصدار JVM).
3. اختر أحد مساري الإعداد:
   - **المسار A (ربط QR):** ‏`signal-cli link -n "OpenClaw"` ثم امسح الرمز باستخدام Signal.
   - **المسار B (تسجيل SMS):** سجّل رقمًا مخصصًا باستخدام captcha + التحقق عبر SMS.
4. اضبط OpenClaw وأعد تشغيل gateway.
5. أرسل أول رسالة مباشرة ووافق على pairing (`openclaw pairing approve signal <CODE>`).

الحد الأدنى من التكوين:

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15551234567",
      cliPath: "signal-cli",
      dmPolicy: "pairing",
      allowFrom: ["+15557654321"],
    },
  },
}
```

مرجع الحقول:

| الحقل       | الوصف                                                |
| ----------- | ---------------------------------------------------- |
| `account`   | رقم هاتف الروبوت بتنسيق E.164 ‏(`+15551234567`)      |
| `cliPath`   | مسار `signal-cli` ‏(`signal-cli` إذا كان على `PATH`) |
| `dmPolicy`  | سياسة الوصول إلى الرسائل المباشرة (`pairing` موصى به) |
| `allowFrom` | أرقام الهواتف أو قيم `uuid:<id>` المسموح لها بإرسال رسائل مباشرة |

## ما هو

- قناة Signal عبر `signal-cli` (وليست libsignal مضمّنة).
- توجيه حتمي: تذهب الردود دائمًا إلى Signal.
- تشارك الرسائل المباشرة الجلسة الرئيسية للوكيل؛ بينما تكون المجموعات معزولة (`agent:<agentId>:signal:group:<groupId>`).

## عمليات كتابة التكوين

بشكل افتراضي، يُسمح لـ Signal بكتابة تحديثات التكوين التي يتم تشغيلها بواسطة `/config set|unset` (يتطلب `commands.config: true`).

للتعطيل:

```json5
{
  channels: { signal: { configWrites: false } },
}
```

## نموذج الرقم (مهم)

- يتصل gateway بـ **جهاز Signal** (حساب `signal-cli`).
- إذا شغّلت الروبوت على **حساب Signal الشخصي** الخاص بك، فسيتجاهل رسائلك أنت (حماية من الحلقة).
- للحصول على سلوك "أرسل رسالة إلى الروبوت فيرد"، استخدم **رقم روبوت منفصلًا**.

## مسار الإعداد A: ربط حساب Signal موجود (QR)

1. ثبّت `signal-cli` (إصدار JVM أو الإصدار الأصلي).
2. اربط حساب روبوت:
   - `signal-cli link -n "OpenClaw"` ثم امسح QR في Signal.
3. اضبط Signal وابدأ gateway.

مثال:

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15551234567",
      cliPath: "signal-cli",
      dmPolicy: "pairing",
      allowFrom: ["+15557654321"],
    },
  },
}
```

دعم الحسابات المتعددة: استخدم `channels.signal.accounts` مع تكوين لكل حساب و`name` اختياري. راجع [`gateway/configuration`](/gateway/configuration-reference#multi-account-all-channels) للنمط المشترك.

## مسار الإعداد B: تسجيل رقم روبوت مخصص (SMS، Linux)

استخدم هذا إذا كنت تريد رقم روبوت مخصصًا بدلًا من ربط حساب تطبيق Signal موجود.

1. احصل على رقم يمكنه استلام SMS (أو التحقق الصوتي للهواتف الأرضية).
   - استخدم رقم روبوت مخصصًا لتجنب تعارضات الحساب/الجلسة.
2. ثبّت `signal-cli` على مضيف gateway:

```bash
VERSION=$(curl -Ls -o /dev/null -w %{url_effective} https://github.com/AsamK/signal-cli/releases/latest | sed -e 's/^.*\/v//')
curl -L -O "https://github.com/AsamK/signal-cli/releases/download/v${VERSION}/signal-cli-${VERSION}-Linux-native.tar.gz"
sudo tar xf "signal-cli-${VERSION}-Linux-native.tar.gz" -C /opt
sudo ln -sf /opt/signal-cli /usr/local/bin/
signal-cli --version
```

إذا كنت تستخدم إصدار JVM ‏(`signal-cli-${VERSION}.tar.gz`)، فثبّت JRE 25+ أولًا.
احرص على تحديث `signal-cli`؛ إذ تشير ملاحظات المنبع إلى أن الإصدارات القديمة قد تتعطل مع تغيّر واجهات Signal server API.

3. سجّل الرقم وتحقق منه:

```bash
signal-cli -a +<BOT_PHONE_NUMBER> register
```

إذا كانت captcha مطلوبة:

1. افتح `https://signalcaptchas.org/registration/generate.html`.
2. أكمل captcha، ثم انسخ هدف الرابط `signalcaptcha://...` من "Open Signal".
3. شغّل من نفس عنوان IP الخارجي الخاص بجلسة المتصفح إن أمكن.
4. أعد تشغيل التسجيل فورًا (تنتهي صلاحية رموز captcha بسرعة):

```bash
signal-cli -a +<BOT_PHONE_NUMBER> register --captcha '<SIGNALCAPTCHA_URL>'
signal-cli -a +<BOT_PHONE_NUMBER> verify <VERIFICATION_CODE>
```

4. اضبط OpenClaw، وأعد تشغيل gateway، وتحقق من القناة:

```bash
# If you run the gateway as a user systemd service:
systemctl --user restart openclaw-gateway.service

# Then verify:
openclaw doctor
openclaw channels status --probe
```

5. نفّذ pairing لمرسل الرسائل المباشرة:
   - أرسل أي رسالة إلى رقم الروبوت.
   - وافق على الرمز على الخادم: `openclaw pairing approve signal <PAIRING_CODE>`.
   - احفظ رقم الروبوت كجهة اتصال على هاتفك لتجنب "Unknown contact".

مهم: قد يؤدي تسجيل حساب رقم هاتف باستخدام `signal-cli` إلى إلغاء مصادقة جلسة تطبيق Signal الرئيسية لذلك الرقم. يُفضّل استخدام رقم روبوت مخصص، أو استخدام وضع الربط عبر QR إذا كنت بحاجة إلى الاحتفاظ بإعداد تطبيق الهاتف الحالي.

مراجع المنبع:

- ملف `signal-cli` README: ‏`https://github.com/AsamK/signal-cli`
- تدفق captcha: ‏`https://github.com/AsamK/signal-cli/wiki/Registration-with-captcha`
- تدفق الربط: ‏`https://github.com/AsamK/signal-cli/wiki/Linking-other-devices-(Provisioning)`

## وضع daemon الخارجي (httpUrl)

إذا كنت تريد إدارة `signal-cli` بنفسك (بطء التشغيل الأولي لـ JVM، أو تهيئة الحاوية، أو وحدات CPU مشتركة)، فشغّل daemon بشكل منفصل ووجّه OpenClaw إليه:

```json5
{
  channels: {
    signal: {
      httpUrl: "http://127.0.0.1:8080",
      autoStart: false,
    },
  },
}
```

يتجاوز هذا التشغيل التلقائي والانتظار عند بدء التشغيل داخل OpenClaw. عند بطء البدء أثناء التشغيل التلقائي، اضبط `channels.signal.startupTimeoutMs`.

## التحكم في الوصول (الرسائل المباشرة + المجموعات)

الرسائل المباشرة:

- الافتراضي: `channels.signal.dmPolicy = "pairing"`.
- يتلقى المرسلون غير المعروفين رمز pairing؛ ويتم تجاهل الرسائل حتى الموافقة عليها (تنتهي صلاحية الرموز بعد ساعة واحدة).
- تتم الموافقة عبر:
  - `openclaw pairing list signal`
  - `openclaw pairing approve signal <CODE>`
- pairing هو تبادل الرموز الافتراضي لرسائل Signal المباشرة. التفاصيل: [Pairing](/channels/pairing)
- يتم تخزين المرسلين الذين لديهم UUID فقط (من `sourceUuid`) كـ `uuid:<id>` في `channels.signal.allowFrom`.

المجموعات:

- `channels.signal.groupPolicy = open | allowlist | disabled`.
- يتحكم `channels.signal.groupAllowFrom` في من يمكنه التفعيل في المجموعات عند تعيين `allowlist`.
- يمكن لـ `channels.signal.groups["<group-id>" | "*"]` تجاوز سلوك المجموعات باستخدام `requireMention` و`tools` و`toolsBySender`.
- استخدم `channels.signal.accounts.<id>.groups` لعمليات التجاوز لكل حساب في إعدادات الحسابات المتعددة.
- ملاحظة وقت التشغيل: إذا كان `channels.signal` مفقودًا بالكامل، فسيعود وقت التشغيل إلى `groupPolicy="allowlist"` لفحوصات المجموعات (حتى إذا تم تعيين `channels.defaults.groupPolicy`).

## كيف يعمل (السلوك)

- يعمل `signal-cli` كـ daemon؛ ويقرأ gateway الأحداث عبر SSE.
- تُطبّع الرسائل الواردة إلى غلاف القناة المشترك.
- تُوجَّه الردود دائمًا إلى الرقم أو المجموعة نفسها.

## الوسائط والحدود

- يُجزّأ النص الصادر إلى `channels.signal.textChunkLimit` (الافتراضي 4000).
- تجزئة اختيارية حسب الأسطر الجديدة: اضبط `channels.signal.chunkMode="newline"` للتقسيم عند الأسطر الفارغة (حدود الفقرات) قبل التجزئة حسب الطول.
- المرفقات مدعومة (يتم جلب base64 من `signal-cli`).
- الحد الافتراضي للوسائط: `channels.signal.mediaMaxMb` (الافتراضي 8).
- استخدم `channels.signal.ignoreAttachments` لتخطي تنزيل الوسائط.
- يستخدم سياق سجل المجموعات `channels.signal.historyLimit` (أو `channels.signal.accounts.*.historyLimit`) مع الرجوع إلى `messages.groupChat.historyLimit`. اضبط `0` للتعطيل (الافتراضي 50).

## مؤشرات الكتابة وإيصالات القراءة

- **مؤشرات الكتابة**: يرسل OpenClaw إشارات الكتابة عبر `signal-cli sendTyping` ويحدّثها أثناء تشغيل الرد.
- **إيصالات القراءة**: عندما تكون `channels.signal.sendReadReceipts` صحيحة، يمرر OpenClaw إيصالات القراءة للرسائل المباشرة المسموح بها.
- لا يوفّر Signal-cli إيصالات قراءة للمجموعات.

## التفاعلات (أداة الرسائل)

- استخدم `message action=react` مع `channel=signal`.
- الأهداف: E.164 للمرسل أو UUID (استخدم `uuid:<id>` من مخرجات pairing؛ يعمل UUID الخام أيضًا).
- `messageId` هو الطابع الزمني في Signal للرسالة التي تتفاعل معها.
- تتطلب تفاعلات المجموعات `targetAuthor` أو `targetAuthorUuid`.

أمثلة:

```
message action=react channel=signal target=uuid:123e4567-e89b-12d3-a456-426614174000 messageId=1737630212345 emoji=🔥
message action=react channel=signal target=+15551234567 messageId=1737630212345 emoji=🔥 remove=true
message action=react channel=signal target=signal:group:<groupId> targetAuthor=uuid:<sender-uuid> messageId=1737630212345 emoji=✅
```

التكوين:

- `channels.signal.actions.reactions`: تمكين/تعطيل إجراءات التفاعل (الافتراضي true).
- `channels.signal.reactionLevel`: ‏`off | ack | minimal | extensive`.
  - يؤدي `off`/`ack` إلى تعطيل تفاعلات الوكيل (وستُرجع أداة الرسائل `react` خطأ).
  - يفعّل `minimal`/`extensive` تفاعلات الوكيل ويحددان مستوى الإرشاد.
- عمليات تجاوز لكل حساب: `channels.signal.accounts.<id>.actions.reactions` و`channels.signal.accounts.<id>.reactionLevel`.

## أهداف التسليم (CLI/cron)

- الرسائل المباشرة: `signal:+15551234567` (أو E.164 عادي).
- الرسائل المباشرة عبر UUID: ‏`uuid:<id>` (أو UUID خام).
- المجموعات: `signal:group:<groupId>`.
- أسماء المستخدمين: `username:<name>` (إذا كان حساب Signal لديك يدعمها).

## استكشاف الأخطاء وإصلاحها

شغّل هذا التسلسل أولًا:

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

ثم تأكد من حالة pairing للرسائل المباشرة إذا لزم الأمر:

```bash
openclaw pairing list signal
```

الأعطال الشائعة:

- يمكن الوصول إلى daemon ولكن لا توجد ردود: تحقق من إعدادات الحساب/daemon (`httpUrl`, `account`) ووضع الاستقبال.
- يتم تجاهل الرسائل المباشرة: المرسل بانتظار موافقة pairing.
- يتم تجاهل رسائل المجموعات: تقييد المرسل/الإشارة في المجموعة يمنع التسليم.
- أخطاء التحقق من التكوين بعد التعديلات: شغّل `openclaw doctor --fix`.
- غياب Signal من أدوات التشخيص: تأكد من `channels.signal.enabled: true`.

فحوصات إضافية:

```bash
openclaw pairing list signal
pgrep -af signal-cli
grep -i "signal" "/tmp/openclaw/openclaw-$(date +%Y-%m-%d).log" | tail -20
```

لتدفق الفرز: [/channels/troubleshooting](/channels/troubleshooting).

## ملاحظات أمنية

- يخزّن `signal-cli` مفاتيح الحساب محليًا (عادةً `~/.local/share/signal-cli/data/`).
- انسخ حالة حساب Signal احتياطيًا قبل ترحيل الخادم أو إعادة بنائه.
- أبقِ `channels.signal.dmPolicy: "pairing"` ما لم تكن تريد صراحةً وصولًا أوسع للرسائل المباشرة.
- لا يلزم التحقق عبر SMS إلا لمسارات التسجيل أو الاستعادة، لكن فقدان التحكم في الرقم/الحساب قد يعقّد إعادة التسجيل.

## مرجع التكوين (Signal)

التكوين الكامل: [Configuration](/gateway/configuration)

خيارات المزوّد:

- `channels.signal.enabled`: تمكين/تعطيل بدء تشغيل القناة.
- `channels.signal.account`: E.164 لحساب الروبوت.
- `channels.signal.cliPath`: مسار `signal-cli`.
- `channels.signal.httpUrl`: عنوان URL الكامل للـ daemon (يتجاوز المضيف/المنفذ).
- `channels.signal.httpHost`, `channels.signal.httpPort`: ربط daemon ‏(الافتراضي 127.0.0.1:8080).
- `channels.signal.autoStart`: تشغيل daemon تلقائيًا (الافتراضي true إذا لم يتم تعيين `httpUrl`).
- `channels.signal.startupTimeoutMs`: مهلة انتظار بدء التشغيل بالمللي ثانية (الحد الأقصى 120000).
- `channels.signal.receiveMode`: ‏`on-start | manual`.
- `channels.signal.ignoreAttachments`: تخطي تنزيل المرفقات.
- `channels.signal.ignoreStories`: تجاهل القصص من daemon.
- `channels.signal.sendReadReceipts`: تمرير إيصالات القراءة.
- `channels.signal.dmPolicy`: ‏`pairing | allowlist | open | disabled` (الافتراضي: pairing).
- `channels.signal.allowFrom`: قائمة سماح الرسائل المباشرة (E.164 أو `uuid:<id>`). يتطلب `open` القيمة `"*"`. لا يملك Signal أسماء مستخدمين؛ استخدم معرّفات الهاتف/UUID.
- `channels.signal.groupPolicy`: ‏`open | allowlist | disabled` (الافتراضي: allowlist).
- `channels.signal.groupAllowFrom`: قائمة سماح مرسلي المجموعات.
- `channels.signal.groups`: عمليات تجاوز لكل مجموعة بمفتاح معرّف مجموعة Signal (أو `"*"`). الحقول المدعومة: `requireMention` و`tools` و`toolsBySender`.
- `channels.signal.accounts.<id>.groups`: النسخة الخاصة بكل حساب من `channels.signal.groups` لإعدادات الحسابات المتعددة.
- `channels.signal.historyLimit`: الحد الأقصى لرسائل المجموعات المضمنة كسياق (يعطل عند 0).
- `channels.signal.dmHistoryLimit`: حد سجل الرسائل المباشرة بوحدات أدوار المستخدم. عمليات تجاوز لكل مستخدم: `channels.signal.dms["<phone_or_uuid>"].historyLimit`.
- `channels.signal.textChunkLimit`: حجم التجزئة الصادرة (أحرف).
- `channels.signal.chunkMode`: ‏`length` (الافتراضي) أو `newline` للتقسيم عند الأسطر الفارغة (حدود الفقرات) قبل التجزئة حسب الطول.
- `channels.signal.mediaMaxMb`: الحد الأقصى للوسائط الواردة/الصادرة (MB).

خيارات عامة ذات صلة:

- `agents.list[].groupChat.mentionPatterns` (لا يدعم Signal الإشارات الأصلية).
- `messages.groupChat.mentionPatterns` (الرجوع العام).
- `messages.responsePrefix`.

## ذو صلة

- [نظرة عامة على القنوات](/channels) — جميع القنوات المدعومة
- [Pairing](/channels/pairing) — مصادقة الرسائل المباشرة وتدفق pairing
- [المجموعات](/channels/groups) — سلوك الدردشة الجماعية وتقييد الإشارات
- [توجيه القنوات](/channels/channel-routing) — توجيه الجلسات للرسائل
- [الأمان](/gateway/security) — نموذج الوصول والتقوية
