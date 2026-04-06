---
read_when:
    - إضافة أو تعديل ترحيلات doctor
    - إدخال تغييرات جذرية على الإعدادات
summary: 'أمر Doctor: فحوصات السلامة، وترحيلات الإعدادات، وخطوات الإصلاح'
title: Doctor
x-i18n:
    generated_at: "2026-04-06T03:08:31Z"
    model: gpt-5.4
    provider: openai
    source_hash: 6c0a15c522994552a1eef39206bed71fc5bf45746776372f24f31c101bfbd411
    source_path: gateway/doctor.md
    workflow: 15
---

# Doctor

`openclaw doctor` هو أداة الإصلاح + الترحيل في OpenClaw. فهو يصلح
الإعدادات/الحالة القديمة، ويتحقق من السلامة، ويوفر خطوات إصلاح قابلة للتنفيذ.

## بداية سريعة

```bash
openclaw doctor
```

### بدون واجهة / للأتمتة

```bash
openclaw doctor --yes
```

اقبل الإعدادات الافتراضية من دون مطالبة (بما في ذلك خطوات إصلاح إعادة التشغيل/الخدمة/صندوق الحماية عند الاقتضاء).

```bash
openclaw doctor --repair
```

طبّق الإصلاحات الموصى بها من دون مطالبة (الإصلاحات + إعادة التشغيل حيث يكون ذلك آمنًا).

```bash
openclaw doctor --repair --force
```

طبّق الإصلاحات الجذرية أيضًا (يستبدل إعدادات المشرف المخصصة).

```bash
openclaw doctor --non-interactive
```

شغّل من دون مطالبات وطبّق فقط الترحيلات الآمنة (تطبيع الإعدادات + نقل الحالة على القرص). يتخطى إجراءات إعادة التشغيل/الخدمة/صندوق الحماية التي تتطلب تأكيدًا بشريًا.
تُشغَّل ترحيلات الحالة القديمة تلقائيًا عند اكتشافها.

```bash
openclaw doctor --deep
```

افحص خدمات النظام بحثًا عن تثبيتات إضافية لـ gateway ‏(`launchd`/`systemd`/`schtasks`).

إذا كنت تريد مراجعة التغييرات قبل الكتابة، فافتح ملف الإعدادات أولًا:

```bash
cat ~/.openclaw/openclaw.json
```

## ما الذي يفعله (ملخص)

- تحديث اختياري قبل التنفيذ لتثبيتات git (في الوضع التفاعلي فقط).
- فحص حداثة بروتوكول UI (يعيد بناء Control UI عندما يكون مخطط البروتوكول أحدث).
- فحص السلامة + مطالبة بإعادة التشغيل.
- ملخص حالة Skills ‏(المؤهلة/المفقودة/المحجوبة) وحالة plugins.
- تطبيع الإعدادات للقيم القديمة.
- ترحيل إعدادات Talk من حقول `talk.*` المسطحة القديمة إلى `talk.provider` + `talk.providers.<provider>`.
- فحوصات ترحيل المتصفح لإعدادات إضافات Chrome القديمة وجهوزية Chrome MCP.
- تحذيرات تجاوز موفر OpenCode ‏(`models.providers.opencode` / `models.providers.opencode-go`).
- فحص متطلبات TLS المسبقة لـ OpenAI Codex OAuth profiles.
- ترحيل الحالة القديمة على القرص (sessions/دليل agent/مصادقة WhatsApp).
- ترحيل مفتاح عقد بيان plugin القديم (`speechProviders`, `realtimeTranscriptionProviders`, `realtimeVoiceProviders`, `mediaUnderstandingProviders`, `imageGenerationProviders`, `videoGenerationProviders`, `webFetchProviders`, `webSearchProviders` → `contracts`).
- ترحيل مخزن cron القديم (`jobId`, `schedule.cron`, حقول delivery/payload ذات المستوى الأعلى، و`provider` في payload، ووظائف fallback القديمة البسيطة `notify: true` الخاصة بـ webhook).
- فحص ملفات قفل الجلسات وتنظيف الأقفال القديمة.
- فحوصات سلامة الحالة والأذونات (sessions، وtranscripts، ودليل الحالة).
- فحوصات أذونات ملف الإعدادات (`chmod 600`) عند التشغيل محليًا.
- سلامة مصادقة النماذج: يتحقق من انتهاء OAuth، ويمكنه تحديث الرموز التي أوشكت على الانتهاء، ويعرض حالات cooldown/التعطيل في auth-profile.
- اكتشاف دليل مساحة عمل إضافي (`~/openclaw`).
- إصلاح صورة sandbox عند تمكين sandboxing.
- ترحيل الخدمات القديمة واكتشاف gateways الإضافية.
- ترحيل حالة Matrix channel القديمة (في وضع `--fix` / `--repair`).
- فحوصات وقت تشغيل Gateway ‏(الخدمة مثبتة لكن لا تعمل؛ تسمية `launchd` المخبأة).
- تحذيرات حالة القنوات (يتم فحصها من gateway العامل).
- تدقيق إعدادات المشرف (`launchd`/`systemd`/`schtasks`) مع إصلاح اختياري.
- فحوصات أفضل الممارسات لوقت تشغيل Gateway ‏(Node مقابل Bun، ومسارات مديري الإصدارات).
- تشخيص تضارب منفذ Gateway ‏(الافتراضي `18789`).
- تحذيرات أمنية لسياسات الرسائل المباشرة المفتوحة.
- فحوصات مصادقة Gateway لوضع الرمز المحلي (يقترح إنشاء رمز عند عدم وجود مصدر له؛ ولا يستبدل إعدادات الرمز المُدارة عبر SecretRef).
- فحص systemd linger على Linux.
- فحص حجم ملفات تمهيد مساحة العمل (تحذيرات الاقتطاع/الاقتراب من الحد لملفات السياق).
- فحص حالة الإكمال في shell والتثبيت/الترقية التلقائية.
- فحص جاهزية موفر تضمين بحث الذاكرة (نموذج محلي، أو مفتاح API بعيد، أو ملف QMD ثنائي).
- فحوصات تثبيت المصدر (عدم تطابق مساحة عمل pnpm، وأصول UI المفقودة، وملف tsx الثنائي المفقود).
- يكتب الإعدادات المحدّثة + بيانات المعالج الوصفية.

## السلوك المفصل والمبررات

### 0) تحديث اختياري (تثبيتات git)

إذا كان هذا checkout من git وكان doctor يعمل بشكل تفاعلي، فإنه يعرض
التحديث (fetch/rebase/build) قبل تشغيل doctor.

### 1) تطبيع الإعدادات

إذا كانت الإعدادات تحتوي على أشكال قيم قديمة (على سبيل المثال `messages.ackReaction`
من دون تجاوز خاص بالقناة)، فإن doctor يطبعها إلى
المخطط الحالي.

يشمل ذلك حقول Talk المسطحة القديمة. الإعدادات العامة الحالية لـ Talk هي
`talk.provider` + `talk.providers.<provider>`. ويعيد doctor كتابة
أشكال `talk.voiceId` / `talk.voiceAliases` / `talk.modelId` / `talk.outputFormat` /
`talk.apiKey` القديمة إلى خريطة الموفّر.

### 2) ترحيلات مفاتيح الإعدادات القديمة

عندما تحتوي الإعدادات على مفاتيح مهجورة، ترفض الأوامر الأخرى التشغيل وتطلب
منك تشغيل `openclaw doctor`.

سيقوم Doctor بما يلي:

- شرح مفاتيح الإعدادات القديمة التي تم العثور عليها.
- عرض الترحيل الذي طبّقه.
- إعادة كتابة `~/.openclaw/openclaw.json` بالمخطط المحدّث.

كما يشغّل Gateway ترحيلات doctor تلقائيًا عند البدء عندما يكتشف
تنسيق إعدادات قديمًا، لذلك تُصلَح الإعدادات القديمة من دون تدخل يدوي.
وتُعالَج ترحيلات مخزن cron job عبر `openclaw doctor --fix`.

الترحيلات الحالية:

- `routing.allowFrom` → `channels.whatsapp.allowFrom`
- `routing.groupChat.requireMention` → `channels.whatsapp/telegram/imessage.groups."*".requireMention`
- `routing.groupChat.historyLimit` → `messages.groupChat.historyLimit`
- `routing.groupChat.mentionPatterns` → `messages.groupChat.mentionPatterns`
- `routing.queue` → `messages.queue`
- `routing.bindings` → `bindings` على المستوى الأعلى
- `routing.agents`/`routing.defaultAgentId` → `agents.list` + `agents.list[].default`
- `talk.voiceId`/`talk.voiceAliases`/`talk.modelId`/`talk.outputFormat`/`talk.apiKey` القديمة → `talk.provider` + `talk.providers.<provider>`
- `routing.agentToAgent` → `tools.agentToAgent`
- `routing.transcribeAudio` → `tools.media.audio.models`
- `messages.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `messages.tts.providers.<provider>`
- `channels.discord.voice.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `channels.discord.voice.tts.providers.<provider>`
- `channels.discord.accounts.<id>.voice.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `channels.discord.accounts.<id>.voice.tts.providers.<provider>`
- `plugins.entries.voice-call.config.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `plugins.entries.voice-call.config.tts.providers.<provider>`
- `plugins.entries.voice-call.config.provider: "log"` → `"mock"`
- `plugins.entries.voice-call.config.twilio.from` → `plugins.entries.voice-call.config.fromNumber`
- `plugins.entries.voice-call.config.streaming.sttProvider` → `plugins.entries.voice-call.config.streaming.provider`
- `plugins.entries.voice-call.config.streaming.openaiApiKey|sttModel|silenceDurationMs|vadThreshold`
  → `plugins.entries.voice-call.config.streaming.providers.openai.*`
- `bindings[].match.accountID` → `bindings[].match.accountId`
- بالنسبة إلى القنوات التي تحتوي على `accounts` مسماة لكن لا تزال فيها قيم قنوات أحادية الحساب على المستوى الأعلى، انقل تلك القيم ذات النطاق الخاص بالحساب إلى الحساب المُرقّى المختار لتلك القناة (`accounts.default` لمعظم القنوات؛ ويمكن لـ Matrix الاحتفاظ بهدف مسمى/افتراضي مطابق موجود)
- `identity` → `agents.list[].identity`
- `agent.*` → `agents.defaults` + `tools.*` (الأدوات/elevated/exec/sandbox/subagents)
- `agent.model`/`allowedModels`/`modelAliases`/`modelFallbacks`/`imageModelFallbacks`
  → `agents.defaults.models` + `agents.defaults.model.primary/fallbacks` + `agents.defaults.imageModel.primary/fallbacks`
- `browser.ssrfPolicy.allowPrivateNetwork` → `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork`
- `browser.profiles.*.driver: "extension"` → `"existing-session"`
- إزالة `browser.relayBindHost` (إعداد relay قديم للإضافة)

تشمل تحذيرات Doctor أيضًا إرشادات الحساب الافتراضي للقنوات متعددة الحسابات:

- إذا جرى إعداد إدخالين أو أكثر في `channels.<channel>.accounts` من دون `channels.<channel>.defaultAccount` أو `accounts.default`، فإن doctor يحذر من أن توجيه fallback قد يختار حسابًا غير متوقع.
- إذا كانت قيمة `channels.<channel>.defaultAccount` تشير إلى معرّف حساب غير معروف، فإن doctor يحذر ويعرض معرّفات الحسابات المُعدّة.

### 2b) تجاوزات موفر OpenCode

إذا أضفت `models.providers.opencode` أو `opencode-zen` أو `opencode-go`
يدويًا، فهذا يتجاوز كتالوج OpenCode المدمج القادم من `@mariozechner/pi-ai`.
وقد يؤدي ذلك إلى فرض استخدام واجهة API خاطئة للنماذج أو تصفير التكاليف. يحذر doctor حتى تتمكن
من إزالة التجاوز واستعادة توجيه API لكل نموذج + التكاليف.

### 2c) ترحيل المتصفح وجهوزية Chrome MCP

إذا كان إعداد المتصفح لا يزال يشير إلى مسار إضافة Chrome الذي أُزيل، فإن doctor
يطبّعه إلى نموذج الربط الحالي المحلي على المضيف لـ Chrome MCP:

- `browser.profiles.*.driver: "extension"` يصبح `"existing-session"`
- تتم إزالة `browser.relayBindHost`

كما يدقق doctor في مسار Chrome MCP المحلي على المضيف عندما تستخدم
`defaultProfile: "user"` أو ملف تعريف `existing-session` مُعدًّا:

- يتحقق مما إذا كان Google Chrome مثبتًا على المضيف نفسه من أجل ملفات
  التعريف الافتراضية ذات الاتصال التلقائي
- يتحقق من إصدار Chrome المكتشف ويحذر عندما يكون أقل من Chrome 144
- يذكّرك بتمكين التصحيح عن بُعد في صفحة فحص المتصفح (على
  سبيل المثال `chrome://inspect/#remote-debugging` أو `brave://inspect/#remote-debugging`
  أو `edge://inspect/#remote-debugging`)

لا يستطيع Doctor تمكين إعداد جهة Chrome بالنيابة عنك. ولا يزال Chrome MCP المحلي على المضيف
يتطلب ما يلي:

- متصفحًا قائمًا على Chromium بإصدار 144+ على مضيف gateway/node
- تشغيل المتصفح محليًا
- تمكين التصحيح عن بُعد في ذلك المتصفح
- الموافقة على أول مطالبة موافقة على الربط في المتصفح

الجهوزية هنا تتعلق فقط بمتطلبات الربط المحلية. يحافظ existing-session على حدود المسار الحالية لـ Chrome MCP؛ أما المسارات المتقدمة مثل `responsebody` وتصدير PDF واعتراض التنزيلات والإجراءات الدفعية فلا تزال تتطلب متصفحًا مُدارًا أو ملف تعريف CDP خام.

لا ينطبق هذا الفحص على تدفقات Docker أو sandbox أو remote-browser أو غيرها من
التدفقات عديمة الواجهة. فهذه تواصل استخدام CDP الخام.

### 2d) متطلبات OAuth TLS المسبقة

عند إعداد OpenAI Codex OAuth profile، يفحص doctor نقطة نهاية
تفويض OpenAI للتحقق من أن مكدس TLS المحلي لـ Node/OpenSSL يمكنه
التحقق من سلسلة الشهادات. إذا فشل الفحص بسبب خطأ في الشهادة (على
سبيل المثال `UNABLE_TO_GET_ISSUER_CERT_LOCALLY`، أو شهادة منتهية، أو شهادة موقعة ذاتيًا)،
فإن doctor يطبع إرشادات إصلاح خاصة بالمنصة. على macOS مع Node من Homebrew، يكون
الإصلاح عادةً `brew postinstall ca-certificates`. ومع `--deep`، يُشغَّل الفحص
حتى لو كانت gateway سليمة.

### 3) ترحيلات الحالة القديمة (تخطيط القرص)

يمكن لـ Doctor ترحيل التخطيطات الأقدم على القرص إلى البنية الحالية:

- مخزن الجلسات + transcripts:
  - من `~/.openclaw/sessions/` إلى `~/.openclaw/agents/<agentId>/sessions/`
- دليل agent:
  - من `~/.openclaw/agent/` إلى `~/.openclaw/agents/<agentId>/agent/`
- حالة مصادقة WhatsApp ‏(Baileys):
  - من `~/.openclaw/credentials/*.json` القديمة (باستثناء `oauth.json`)
  - إلى `~/.openclaw/credentials/whatsapp/<accountId>/...` (معرّف الحساب الافتراضي: `default`)

هذه الترحيلات تُنفَّذ بأفضل جهد وبشكل idempotent؛ وسيصدر doctor تحذيرات عندما
يترك أي مجلدات قديمة خلفه كنسخ احتياطية. كما يقوم Gateway/CLI بترحيل
الجلسات القديمة + دليل agent تلقائيًا عند البدء بحيث ينتقل السجل/المصادقة/النماذج إلى
المسار الخاص بكل agent من دون تشغيل doctor يدويًا. وتُرحَّل مصادقة WhatsApp عمدًا فقط
عبر `openclaw doctor`. ويقارن تطبيع Talk provider/provider-map الآن
بالمساواة الهيكلية، لذلك لم تعد الفروق في ترتيب المفاتيح فقط تؤدي إلى
تغييرات `doctor --fix` متكررة بلا أثر.

### 3a) ترحيلات بيانات plugin القديمة

يفحص Doctor جميع بيانات plugin المثبتة بحثًا عن مفاتيح
القدرات المهجورة ذات المستوى الأعلى (`speechProviders`, `realtimeTranscriptionProviders`,
`realtimeVoiceProviders`, `mediaUnderstandingProviders`,
`imageGenerationProviders`, `videoGenerationProviders`, `webFetchProviders`,
`webSearchProviders`). وعند العثور عليها، يعرض نقلها إلى الكائن `contracts`
وإعادة كتابة ملف البيان في مكانه. وهذا الترحيل idempotent؛
فإذا كان مفتاح `contracts` يحتوي بالفعل على القيم نفسها، تتم إزالة
المفتاح القديم من دون تكرار البيانات.

### 3b) ترحيلات مخزن cron القديمة

يتحقق Doctor أيضًا من مخزن cron job ‏(`~/.openclaw/cron/jobs.json` افتراضيًا،
أو `cron.store` عند التجاوز) بحثًا عن أشكال الوظائف القديمة التي لا يزال المجدول
يقبلها لأغراض التوافق.

تشمل عمليات تنظيف cron الحالية ما يلي:

- `jobId` → `id`
- `schedule.cron` → `schedule.expr`
- حقول payload ذات المستوى الأعلى (`message`, `model`, `thinking`, ...) → `payload`
- حقول delivery ذات المستوى الأعلى (`deliver`, `channel`, `to`, `provider`, ...) → `delivery`
- أسماء delivery البديلة في payload `provider` → `delivery.channel` الصريح
- وظائف fallback القديمة البسيطة `notify: true` الخاصة بـ webhook → `delivery.mode="webhook"` الصريح مع `delivery.to=cron.webhook`

لا يرحّل Doctor وظائف `notify: true` تلقائيًا إلا عندما يمكنه القيام بذلك
من دون تغيير السلوك. وإذا جمعت وظيفة بين fallback notify قديم ووضع delivery
غير webhook موجود، فإن doctor يحذر ويترك تلك الوظيفة للمراجعة اليدوية.

### 3c) تنظيف أقفال الجلسات

يفحص Doctor دليل جلسات كل agent بحثًا عن ملفات write-lock القديمة —
وهي الملفات التي تُترك خلفها عندما تنتهي الجلسة بشكل غير طبيعي. ولكل ملف قفل يعثر عليه، يعرض:
المسار، وPID، وما إذا كان PID لا يزال حيًا، وعمر القفل، وما إذا كان
يُعد قديمًا (PID ميت أو أقدم من 30 دقيقة). وفي وضع `--fix` / `--repair`
يزيل ملفات القفل القديمة تلقائيًا؛ وإلا فإنه يطبع ملاحظة
ويطلب منك إعادة التشغيل مع `--fix`.

### 4) فحوصات سلامة الحالة (استمرارية الجلسة، والتوجيه، والأمان)

دليل الحالة هو الجذع التشغيلي للنظام. إذا اختفى، فستفقد
الجلسات وبيانات الاعتماد والسجلات والإعدادات (ما لم تكن لديك نسخ احتياطية في مكان آخر).

يتحقق Doctor مما يلي:

- **دليل الحالة مفقود**: يحذر من فقدان الحالة الكارثي، ويطلب إعادة إنشاء
  الدليل، ويذكّرك بأنه لا يمكنه استعادة البيانات المفقودة.
- **أذونات دليل الحالة**: يتحقق من قابلية الكتابة؛ ويعرض إصلاح الأذونات
  (ويصدر تلميح `chown` عند اكتشاف عدم تطابق المالك/المجموعة).
- **دليل حالة macOS المتزامن سحابيًا**: يحذر عندما تُحل الحالة ضمن iCloud Drive
  (`~/Library/Mobile Documents/com~apple~CloudDocs/...`) أو
  `~/Library/CloudStorage/...` لأن المسارات المدعومة بالمزامنة قد تسبب بطء I/O
  وسباقات القفل/المزامنة.
- **دليل حالة Linux على SD أو eMMC**: يحذر عندما تُحل الحالة إلى مصدر تحميل `mmcblk*`،
  لأن I/O العشوائي المدعوم بـ SD أو eMMC قد يكون أبطأ ويتآكل
  بسرعة أكبر تحت كتابات الجلسات وبيانات الاعتماد.
- **أدلة الجلسات مفقودة**: إن `sessions/` ودليل مخزن الجلسات
  ضروريان لاستمرار السجل وتجنب أعطال `ENOENT`.
- **عدم تطابق transcript**: يحذر عندما تحتوي إدخالات الجلسات الحديثة على ملفات
  transcript مفقودة.
- **الجلسة الرئيسية "JSONL سطر واحد"**: يحدد عندما يحتوي transcript الرئيسي على سطر
  واحد فقط (السجل لا يتراكم).
- **أدلة حالة متعددة**: يحذر عندما توجد عدة مجلدات `~/.openclaw` عبر
  أدلة home أو عندما يشير `OPENCLAW_STATE_DIR` إلى مكان آخر (قد ينقسم السجل
  بين التثبيتات).
- **تذكير الوضع البعيد**: إذا كانت `gateway.mode=remote`، فإن doctor يذكرك بتشغيله
  على المضيف البعيد (فالحالة موجودة هناك).
- **أذونات ملف الإعدادات**: يحذر إذا كان `~/.openclaw/openclaw.json`
  قابلًا للقراءة من المجموعة/العامة ويعرض تشديده إلى `600`.

### 5) سلامة مصادقة النموذج (انتهاء OAuth)

يفحص Doctor OAuth profiles في مخزن المصادقة، ويحذر عندما تكون الرموز
على وشك الانتهاء/منتهية، ويمكنه تحديثها عندما يكون ذلك آمنًا. وإذا كان Anthropic
OAuth/token profile قديمًا، فإنه يقترح مفتاح Anthropic API أو
مسار Anthropic setup-token القديم.
لا تظهر مطالبات التحديث إلا عند التشغيل بشكل تفاعلي (TTY)؛ أما `--non-interactive`
فيتخطى محاولات التحديث.

كما يكتشف Doctor حالة Anthropic Claude CLI القديمة التي أُزيلت. فإذا كانت بايتات بيانات الاعتماد القديمة
`anthropic:claude-cli` لا تزال موجودة في `auth-profiles.json`،
فإن doctor يعيد تحويلها إلى Anthropic token/OAuth profiles ويعيد كتابة
مراجع النماذج القديمة `claude-cli/...`.
أما إذا كانت البايتات قد اختفت، فإن doctor يزيل الإعدادات القديمة ويطبع
أوامر الاستعادة بدلًا من ذلك.

كما يعرض Doctor auth profiles غير القابلة للاستخدام مؤقتًا بسبب:

- cooldowns قصيرة (حدود المعدل/المهل/إخفاقات المصادقة)
- تعطيلات أطول (إخفاقات الفوترة/الرصيد)

### 6) التحقق من نموذج Hooks

إذا كانت `hooks.gmail.model` مضبوطة، فإن doctor يتحقق من مرجع النموذج مقابل
الكتالوج وقائمة السماح ويحذر عندما لا يمكن حله أو عندما يكون غير مسموح به.

### 7) إصلاح صورة Sandbox

عند تمكين sandboxing، يتحقق doctor من صور Docker ويعرض بناءها أو
التبديل إلى الأسماء القديمة إذا كانت الصورة الحالية مفقودة.

### 7b) تبعيات وقت تشغيل plugin المجمعة

يتحقق Doctor من أن تبعيات وقت تشغيل plugin المجمعة (على سبيل المثال
حزم وقت تشغيل Discord plugin) موجودة في جذر تثبيت OpenClaw.
إذا كان أي منها مفقودًا، يعرض doctor الحزم ويثبتها في
وضع `openclaw doctor --fix` / `openclaw doctor --repair`.

### 8) ترحيلات خدمة Gateway وتلميحات التنظيف

يكتشف Doctor خدمات gateway القديمة (`launchd`/`systemd`/`schtasks`) و
يعرض إزالتها وتثبيت خدمة OpenClaw باستخدام منفذ gateway الحالي.
كما يمكنه أيضًا فحص خدمات إضافية شبيهة بـ gateway وطباعة تلميحات تنظيف.
وتُعد خدمات OpenClaw gateway المسماة باسم profile خدمات من الدرجة الأولى ولا
يتم وسمها بأنها "إضافية".

### 8b) ترحيل Matrix عند البدء

عندما يكون لدى حساب Matrix channel ترحيل حالة قديم معلّق أو قابل للتنفيذ،
يقوم doctor (في وضع `--fix` / `--repair`) بإنشاء لقطة قبل الترحيل ثم
يشغّل خطوات الترحيل بأفضل جهد: ترحيل حالة Matrix القديمة وتحضير الحالة
المشفرة القديمة. كلتا الخطوتين غير قاتلتين؛ تُسجّل الأخطاء ويستمر
البدء. أما في وضع القراءة فقط (`openclaw doctor` من دون `--fix`) فيتم
تخطي هذا الفحص بالكامل.

### 9) تحذيرات الأمان

يصدر Doctor تحذيرات عندما يكون الموفّر مفتوحًا للرسائل المباشرة من دون
قائمة سماح، أو عندما تكون إحدى السياسات مُعدّة بطريقة خطيرة.

### 10) systemd linger ‏(Linux)

إذا كان التشغيل يتم كخدمة مستخدم systemd، فإن doctor يضمن تمكين lingering حتى
تبقى gateway عاملة بعد تسجيل الخروج.

### 11) حالة مساحة العمل (Skills وplugins والأدلة القديمة)

يطبع Doctor ملخصًا لحالة مساحة العمل الخاصة بالـ agent الافتراضي:

- **حالة Skills**: يحصي المهارات المؤهلة، وناقصة المتطلبات، والمحجوبة بقائمة السماح.
- **أدلة مساحة العمل القديمة**: يحذر عندما توجد `~/openclaw` أو أدلة مساحة عمل قديمة أخرى
  إلى جانب مساحة العمل الحالية.
- **حالة Plugin**: يحصي plugins المحمّلة/المعطلة/التي بها أخطاء؛ ويعرض معرّفات plugin لأي
  أخطاء؛ ويعرض قدرات bundle plugin.
- **تحذيرات توافق plugin**: يحدد plugins التي لديها مشكلات توافق مع
  وقت التشغيل الحالي.
- **تشخيصات plugin**: يعرض أي تحذيرات أو أخطاء أثناء التحميل صادرة عن
  سجل plugin.

### 11b) حجم ملف التمهيد

يتحقق Doctor مما إذا كانت ملفات تمهيد مساحة العمل (على سبيل المثال `AGENTS.md`،
أو `CLAUDE.md`، أو غيرها من ملفات السياق المحقونة) تقترب من ميزانية
الأحرف المكوّنة أو تتجاوزها. ويعرض عدد الأحرف الخام مقابل المحقونة لكل ملف،
ونسبة الاقتطاع، وسبب الاقتطاع (`max/file` أو `max/total`)، وإجمالي الأحرف
المحقونة كنسبة من الميزانية الكلية. وعندما تُقتطع الملفات أو تقترب من الحد،
يطبع doctor نصائح لضبط `agents.defaults.bootstrapMaxChars`
و`agents.defaults.bootstrapTotalMaxChars`.

### 11c) إكمال Shell

يتحقق Doctor مما إذا كان الإكمال بالضغط على Tab مثبتًا للـ shell الحالية
(`zsh` أو `bash` أو `fish` أو `PowerShell`):

- إذا كان ملف تعريف shell يستخدم نمط إكمال ديناميكي بطيء
  (`source <(openclaw completion ...)`)، فإن doctor يرقّيه إلى
  صيغة الملف المخبأ الأسرع.
- إذا كان الإكمال مُعدًّا في ملف التعريف لكن ملف cache مفقودًا،
  فإن doctor يعيد إنشاء cache تلقائيًا.
- إذا لم يكن هناك أي إعداد للإكمال على الإطلاق، فإن doctor يطلب تثبيته
  (في الوضع التفاعلي فقط؛ ويتم تخطيه مع `--non-interactive`).

شغّل `openclaw completion --write-state` لإعادة إنشاء cache يدويًا.

### 12) فحوصات مصادقة Gateway ‏(الرمز المحلي)

يتحقق Doctor من جاهزية مصادقة رمز gateway المحلي.

- إذا كان وضع الرمز يحتاج إلى رمز ولا يوجد مصدر له، فإن doctor يعرض إنشاء واحد.
- إذا كانت `gateway.auth.token` مُدارة عبر SecretRef لكنها غير متاحة، فإن doctor يحذر ولا يستبدلها بنص صريح.
- يفرض `openclaw doctor --generate-gateway-token` الإنشاء فقط عندما لا يكون هناك token SecretRef مُعدّ.

### 12b) إصلاحات للقراءة فقط ومراعية لـ SecretRef

تحتاج بعض تدفقات الإصلاح إلى فحص بيانات الاعتماد المُعدة من دون إضعاف سلوك الفشل السريع أثناء التشغيل.

- يستخدم `openclaw doctor --fix` الآن نموذج ملخص SecretRef للقراءة فقط نفسه المستخدم في أوامر عائلة status من أجل إصلاحات إعدادات موجهة.
- مثال: يحاول إصلاح Telegram `allowFrom` / `groupAllowFrom` من نوع `@username` استخدام بيانات اعتماد bot المُعدة عندما تكون متاحة.
- إذا كان Telegram bot token مُعدًّا عبر SecretRef لكنه غير متاح في مسار الأمر الحالي، فإن doctor يعرض أن بيانات الاعتماد مُعدة لكنها غير متاحة ويتخطى الحل التلقائي بدلًا من التعطل أو الإبلاغ خطأً عن الرمز على أنه مفقود.

### 13) فحص سلامة Gateway + إعادة التشغيل

يشغّل Doctor فحص سلامة ويعرض إعادة تشغيل gateway عندما تبدو
غير سليمة.

### 13b) جاهزية بحث الذاكرة

يتحقق Doctor مما إذا كان موفر تضمين بحث الذاكرة المُعدّ جاهزًا
للـ agent الافتراضي. ويعتمد السلوك على الواجهة الخلفية والموفر المُعدين:

- **واجهة QMD الخلفية**: يفحص ما إذا كان الملف الثنائي `qmd` متاحًا وقابلًا للتشغيل.
  وإذا لم يكن كذلك، يطبع إرشادات إصلاح تتضمن حزمة npm وخيار مسار يدوي للملف الثنائي.
- **موفر محلي صريح**: يتحقق من وجود ملف نموذج محلي أو
  URL معروف لنموذج بعيد/قابل للتنزيل. وإذا كان مفقودًا، يقترح التبديل إلى موفر بعيد.
- **موفر بعيد صريح** (`openai`, `voyage`, إلخ): يتحقق من وجود مفتاح API
  في البيئة أو مخزن المصادقة. ويطبع تلميحات إصلاح قابلة للتنفيذ إذا كان مفقودًا.
- **موفر تلقائي**: يتحقق أولًا من توفر النموذج المحلي، ثم يجرب كل موفر بعيد
  وفق ترتيب الاختيار التلقائي.

عندما تكون نتيجة فحص gateway متاحة (كانت gateway سليمة وقت
الفحص)، فإن doctor يقاطع نتيجتها مع الإعدادات المرئية من CLI ويشير
إلى أي اختلاف.

استخدم `openclaw memory status --deep` للتحقق من جاهزية التضمين أثناء التشغيل.

### 14) تحذيرات حالة القنوات

إذا كانت gateway سليمة، فإن doctor يشغّل فحص حالة القنوات ويعرض
التحذيرات مع الإصلاحات المقترحة.

### 15) تدقيق إعدادات المشرف + الإصلاح

يتحقق Doctor من إعدادات المشرف المثبتة (`launchd`/`systemd`/`schtasks`) بحثًا عن
افتراضيات مفقودة أو قديمة (مثل تبعيات systemd الخاصة بـ network-online و
تأخير إعادة التشغيل). وعندما يجد عدم تطابق، فإنه يوصي بالتحديث ويمكنه
إعادة كتابة ملف الخدمة/المهمة إلى الافتراضيات الحالية.

ملاحظات:

- يطلب `openclaw doctor` التأكيد قبل إعادة كتابة إعدادات المشرف.
- يقبل `openclaw doctor --yes` مطالبات الإصلاح الافتراضية.
- يطبّق `openclaw doctor --repair` الإصلاحات الموصى بها من دون مطالبات.
- يستبدل `openclaw doctor --repair --force` إعدادات المشرف المخصصة.
- إذا كانت مصادقة الرمز تتطلب رمزًا وكانت `gateway.auth.token` مُدارة عبر SecretRef، فإن doctor عند تثبيت/إصلاح الخدمة يتحقق من SecretRef لكنه لا يحفظ قيم الرمز الصريح التي تم حلها داخل بيانات بيئة خدمة المشرف الوصفية.
- إذا كانت مصادقة الرمز تتطلب رمزًا وكان token SecretRef المُعدّ غير محلول، فإن doctor يمنع مسار التثبيت/الإصلاح مع إرشادات قابلة للتنفيذ.
- إذا كانت كل من `gateway.auth.token` و`gateway.auth.password` مُعدّتين وكانت `gateway.auth.mode` غير مضبوطة، فإن doctor يمنع التثبيت/الإصلاح حتى يتم ضبط الوضع صراحةً.
- بالنسبة إلى وحدات Linux user-systemd، تتضمن فحوصات doctor لانحراف الرمز الآن كلا المصدرين `Environment=` و`EnvironmentFile=` عند مقارنة بيانات مصادقة الخدمة الوصفية.
- يمكنك دائمًا فرض إعادة كتابة كاملة عبر `openclaw gateway install --force`.

### 16) تشخيصات وقت تشغيل Gateway + المنفذ

يفحص Doctor وقت تشغيل الخدمة (PID، وآخر حالة خروج) ويحذر عندما تكون
الخدمة مثبتة ولكنها لا تعمل فعليًا. كما يتحقق من تضارب المنافذ
على منفذ gateway ‏(الافتراضي `18789`) ويعرض الأسباب المحتملة (gateway تعمل بالفعل،
أو نفق SSH).

### 17) أفضل ممارسات وقت تشغيل Gateway

يحذر Doctor عندما تعمل خدمة gateway على Bun أو على مسار Node مُدار
بمدير إصدارات (`nvm` أو `fnm` أو `volta` أو `asdf`، إلخ). تتطلب قناتا WhatsApp + Telegram
Node، كما يمكن أن تنكسر مسارات مديري الإصدارات بعد الترقيات لأن الخدمة لا
تحمّل تهيئة shell الخاصة بك. ويعرض doctor الترحيل إلى تثبيت Node على مستوى النظام عندما
يكون متاحًا (`Homebrew`/`apt`/`choco`).

### 18) كتابة الإعدادات + بيانات المعالج الوصفية

يحفظ Doctor أي تغييرات في الإعدادات ويختم بيانات المعالج الوصفية لتسجيل
تشغيل doctor.

### 19) نصائح مساحة العمل (النسخ الاحتياطي + نظام الذاكرة)

يقترح Doctor نظام ذاكرة لمساحة العمل عندما يكون مفقودًا ويطبع نصيحة نسخ احتياطي
إذا لم تكن مساحة العمل تحت git بالفعل.

راجع [/concepts/agent-workspace](/ar/concepts/agent-workspace) للحصول على دليل كامل حول
بنية مساحة العمل والنسخ الاحتياطي باستخدام git (ويُوصى باستخدام GitHub أو GitLab خاص).
