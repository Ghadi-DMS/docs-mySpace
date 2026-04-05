---
read_when:
    - إعداد جهاز جديد
    - تريد "أحدث وأفضل" دون كسر إعدادك الشخصي
summary: إعدادات متقدمة ومسارات عمل التطوير لـ OpenClaw
title: الإعداد
x-i18n:
    generated_at: "2026-04-05T12:56:56Z"
    model: gpt-5.4
    provider: openai
    source_hash: be4e280dde7f3a224345ca557ef2fb35a9c9db8520454ff63794ac6f8d4e71e7
    source_path: start/setup.md
    workflow: 15
---

# الإعداد

<Note>
إذا كنت تُعد النظام لأول مرة، فابدأ بـ [البدء](/ar/start/getting-started).
للحصول على تفاصيل الإعداد الأولي، راجع [الإعداد الأولي (CLI)](/ar/start/wizard).
</Note>

## الخلاصة

- **التخصيص يوجد خارج المستودع:** `~/.openclaw/workspace` (مساحة العمل) + `~/.openclaw/openclaw.json` (التهيئة).
- **مسار العمل المستقر:** ثبّت تطبيق macOS ودعه يشغّل Gateway المضمّن.
- **مسار العمل الأحدث:** شغّل Gateway بنفسك عبر `pnpm gateway:watch`، ثم دع تطبيق macOS يتصل في الوضع المحلي.

## المتطلبات المسبقة (من المصدر)

- يوصى باستخدام Node 24 (ولا يزال Node 22 LTS، حاليًا `22.14+`، مدعومًا)
- يُفضّل `pnpm` (أو Bun إذا كنت تستخدم عمدًا [مسار عمل Bun](/ar/install/bun))
- Docker (اختياري؛ فقط للإعداد/اختبارات e2e داخل الحاويات — راجع [Docker](/ar/install/docker))

## استراتيجية التخصيص (حتى لا تضر التحديثات)

إذا كنت تريد "مخصصًا لي 100%" _و_ تحديثات سهلة، فاحتفظ بتخصيصك في:

- **التهيئة:** `~/.openclaw/openclaw.json` ‏(JSON/بصيغة شبيهة بـ JSON5)
- **مساحة العمل:** `~/.openclaw/workspace` ‏(Skills، والموجّهات، والذكريات؛ اجعلها مستودع git خاصًا)

نفّذ التمهيد مرة واحدة:

```bash
openclaw setup
```

من داخل هذا المستودع، استخدم نقطة إدخال CLI المحلية:

```bash
openclaw setup
```

إذا لم يكن لديك تثبيت عام بعد، فشغّله عبر `pnpm openclaw setup` (أو `bun run openclaw setup` إذا كنت تستخدم مسار عمل Bun).

## تشغيل Gateway من هذا المستودع

بعد `pnpm build`، يمكنك تشغيل CLI المجمّع مباشرة:

```bash
node openclaw.mjs gateway --port 18789 --verbose
```

## مسار العمل المستقر (تطبيق macOS أولًا)

1. ثبّت وشغّل **OpenClaw.app** (شريط القوائم).
2. أكمل قائمة التحقق الخاصة بالإعداد الأولي/الأذونات (مطالبات TCC).
3. تأكد من أن Gateway مضبوط على **Local** ويعمل (يتولى التطبيق إدارته).
4. اربط الواجهات (مثال: WhatsApp):

```bash
openclaw channels login
```

5. تحقق سريعًا:

```bash
openclaw health
```

إذا لم يكن الإعداد الأولي متاحًا في نسختك:

- شغّل `openclaw setup`، ثم `openclaw channels login`، ثم ابدأ Gateway يدويًا (`openclaw gateway`).

## مسار العمل الأحدث (Gateway في طرفية)

الهدف: العمل على TypeScript Gateway، والحصول على إعادة تحميل فورية، مع إبقاء واجهة تطبيق macOS متصلة.

### 0) (اختياري) شغّل تطبيق macOS من المصدر أيضًا

إذا كنت تريد أيضًا أن يكون تطبيق macOS على أحدث نسخة:

```bash
./scripts/restart-mac.sh
```

### 1) ابدأ Gateway الخاص بالتطوير

```bash
pnpm install
pnpm gateway:watch
```

يقوم `gateway:watch` بتشغيل gateway في وضع المراقبة ويعيد التحميل عند تغييرات المصدر،
والتهيئة، وبيانات تعريف plugin المجمّعة ذات الصلة.

إذا كنت تستخدم عمدًا مسار عمل Bun، فالأوامر المكافئة هي:

```bash
bun install
bun run gateway:watch
```

### 2) وجّه تطبيق macOS إلى Gateway الذي تشغّله

في **OpenClaw.app**:

- وضع الاتصال: **Local**
  سيتصل التطبيق بالـ gateway الجاري تشغيله على المنفذ المهيأ.

### 3) تحقّق

- يجب أن تعرض حالة Gateway داخل التطبيق: **“Using existing gateway …”**
- أو عبر CLI:

```bash
openclaw health
```

### الأخطاء الشائعة

- **منفذ خاطئ:** يستخدم Gateway WS افتراضيًا `ws://127.0.0.1:18789`؛ حافظ على التطبيق وCLI على المنفذ نفسه.
- **مكان تخزين الحالة:**
  - حالة القناة/الموفر: `~/.openclaw/credentials/`
  - ملفات تعريف مصادقة النموذج: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
  - الجلسات: `~/.openclaw/agents/<agentId>/sessions/`
  - السجلات: `/tmp/openclaw/`

## خريطة تخزين بيانات الاعتماد

استخدم هذا عند تصحيح أخطاء المصادقة أو عند تحديد ما يجب نسخه احتياطيًا:

- **WhatsApp**: `~/.openclaw/credentials/whatsapp/<accountId>/creds.json`
- **رمز bot الخاص بـ Telegram**: التهيئة/البيئة أو `channels.telegram.tokenFile` (ملف عادي فقط؛ الروابط الرمزية مرفوضة)
- **رمز bot الخاص بـ Discord**: التهيئة/البيئة أو SecretRef (موفرو env/file/exec)
- **رموز Slack**: التهيئة/البيئة (`channels.slack.*`)
- **قوائم السماح للإقران**:
  - `~/.openclaw/credentials/<channel>-allowFrom.json` (الحساب الافتراضي)
  - `~/.openclaw/credentials/<channel>-<accountId>-allowFrom.json` (للحسابات غير الافتراضية)
- **ملفات تعريف مصادقة النموذج**: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
- **حمولة الأسرار المعتمدة على الملفات (اختياري)**: `~/.openclaw/secrets.json`
- **استيراد OAuth القديم**: `~/.openclaw/credentials/oauth.json`
  مزيد من التفاصيل: [الأمان](/ar/gateway/security#credential-storage-map).

## التحديث (من دون إفساد إعدادك)

- احتفظ بـ `~/.openclaw/workspace` و`~/.openclaw/` باعتبارهما "أغراضك الخاصة"؛ لا تضع الموجّهات/التهيئة الشخصية داخل مستودع `openclaw`.
- تحديث المصدر: `git pull` + خطوة تثبيت مدير الحزم الذي اخترته (`pnpm install` افتراضيًا؛ `bun install` لمسار عمل Bun) + الاستمرار في استخدام أمر `gateway:watch` المطابق.

## Linux (خدمة مستخدم systemd)

تستخدم تثبيتات Linux خدمة مستخدم **systemd**. بشكل افتراضي، يوقف systemd خدمات المستخدم
عند تسجيل الخروج/الخمول، مما يؤدي إلى إيقاف Gateway. يحاول الإعداد الأولي تمكين
الاستمرار نيابةً عنك (وقد يطلب sudo). إذا ظل معطّلًا، فشغّل:

```bash
sudo loginctl enable-linger $USER
```

بالنسبة للخوادم الدائمة التشغيل أو متعددة المستخدمين، فكر في استخدام خدمة **system** بدلًا من
خدمة مستخدم (ولا حاجة إلى الاستمرار). راجع [دليل تشغيل Gateway](/ar/gateway) لملاحظات systemd.

## مستندات ذات صلة

- [دليل تشغيل Gateway](/ar/gateway) (العلامات، والإشراف، والمنافذ)
- [تهيئة Gateway](/ar/gateway/configuration) (مخطط التهيئة + أمثلة)
- [Discord](/ar/channels/discord) و[Telegram](/ar/channels/telegram) (علامات الرد + إعدادات replyToMode)
- [إعداد مساعد OpenClaw](/start/openclaw)
- [تطبيق macOS](/ar/platforms/macos) (دورة حياة gateway)
