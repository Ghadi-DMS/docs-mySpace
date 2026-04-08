---
read_when:
    - عند إضافة أو تعديل ترحيلات doctor
    - عند إدخال تغييرات جذرية على الإعدادات
summary: 'أمر Doctor: فحوصات السلامة، وترحيلات الإعدادات، وخطوات الإصلاح'
title: Doctor
x-i18n:
    generated_at: "2026-04-08T02:15:41Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3761a222d9db7088f78215575fa84e5896794ad701aa716e8bf9039a4424dca6
    source_path: gateway/doctor.md
    workflow: 15
---

# Doctor

إن `openclaw doctor` هو أداة الإصلاح + الترحيل في OpenClaw. فهو يصلح
الإعدادات/الحالة القديمة، ويتحقق من السلامة، ويوفر خطوات إصلاح قابلة للتنفيذ.

## البدء السريع

```bash
openclaw doctor
```

### بدون واجهة / للأتمتة

```bash
openclaw doctor --yes
```

اقبل القيم الافتراضية من دون مطالبة (بما في ذلك خطوات إصلاح إعادة التشغيل/الخدمة/العزل عند الاقتضاء).

```bash
openclaw doctor --repair
```

طبّق الإصلاحات الموصى بها من دون مطالبة (الإصلاحات + إعادة التشغيل عندما يكون ذلك آمنًا).

```bash
openclaw doctor --repair --force
```

طبّق الإصلاحات العدوانية أيضًا (يستبدل إعدادات supervisor المخصصة).

```bash
openclaw doctor --non-interactive
```

شغّل من دون مطالبات وطبّق فقط الترحيلات الآمنة (تطبيع الإعدادات + نقل الحالة على القرص). يتخطى إجراءات إعادة التشغيل/الخدمة/العزل التي تتطلب تأكيدًا بشريًا.
تعمل ترحيلات الحالة القديمة تلقائيًا عند اكتشافها.

```bash
openclaw doctor --deep
```

افحص خدمات النظام لاكتشاف عمليات تثبيت إضافية لـ gateway ‏(launchd/systemd/schtasks).

إذا كنت تريد مراجعة التغييرات قبل الكتابة، فافتح ملف الإعدادات أولًا:

```bash
cat ~/.openclaw/openclaw.json
```

## ما الذي يفعله (ملخص)

- تحديث اختياري قبل التنفيذ لتثبيتات git ‏(تفاعلي فقط).
- فحص حداثة بروتوكول واجهة المستخدم (يعيد بناء Control UI عندما يكون مخطط البروتوكول أحدث).
- فحص السلامة + مطالبة بإعادة التشغيل.
- ملخص حالة Skills ‏(المؤهلة/المفقودة/المحجوبة) وحالة plugin.
- تطبيع الإعدادات للقيم القديمة.
- ترحيل إعدادات Talk من حقول `talk.*` القديمة المسطحة إلى `talk.provider` + `talk.providers.<provider>`.
- فحوصات ترحيل المتصفح لإعدادات Chrome extension القديمة وجهوزية Chrome MCP.
- تحذيرات تجاوزات موفّر OpenCode ‏(`models.providers.opencode` / `models.providers.opencode-go`).
- تحذيرات حجب Codex OAuth ‏(`models.providers.openai-codex`).
- فحص متطلبات OAuth TLS لملفات OpenAI Codex OAuth الشخصية.
- ترحيل الحالة القديمة على القرص (الجلسات/دليل الوكيل/مصادقة WhatsApp).
- ترحيل مفتاح عقد plugin manifest القديم (`speechProviders`, `realtimeTranscriptionProviders`, `realtimeVoiceProviders`, `mediaUnderstandingProviders`, `imageGenerationProviders`, `videoGenerationProviders`, `webFetchProviders`, `webSearchProviders` ← `contracts`).
- ترحيل مخزن cron القديم (`jobId`, ‏`schedule.cron`, وحقول delivery/payload ذات المستوى الأعلى، و`provider` في payload، ووظائف fallback القديمة البسيطة من نوع `notify: true` webhook).
- فحص ملف قفل الجلسة وتنظيف الأقفال القديمة.
- فحوصات تكامل الحالة والأذونات (الجلسات، والنصوص، ودليل الحالة).
- فحوصات أذونات ملف الإعدادات (`chmod 600`) عند التشغيل محليًا.
- سلامة مصادقة النموذج: يتحقق من انتهاء OAuth، ويمكنه تحديث الرموز الموشكة على الانتهاء، ويبلغ عن حالات auth-profile المهدأة/المعطلة.
- اكتشاف أدلة workspace الإضافية (`~/openclaw`).
- إصلاح صورة sandbox عند تفعيل العزل.
- ترحيل الخدمة القديمة واكتشاف gateway الإضافية.
- ترحيل الحالة القديمة لقناة Matrix ‏(في وضع `--fix` / `--repair`).
- فحوصات وقت تشغيل Gateway ‏(الخدمة مثبتة ولكنها لا تعمل؛ ووسم launchd المخزن مؤقتًا).
- تحذيرات حالة القناة (يتم فحصها من gateway الجاري تشغيله).
- تدقيق إعدادات supervisor ‏(launchd/systemd/schtasks) مع إصلاح اختياري.
- فحوصات أفضل الممارسات لوقت تشغيل Gateway ‏(Node مقابل Bun، ومسارات مديري الإصدارات).
- تشخيص تعارض منفذ Gateway ‏(الافتراضي `18789`).
- تحذيرات أمنية لسياسات DM المفتوحة.
- فحوصات مصادقة Gateway لوضع الرمز المحلي (يعرض توليد رمز عند عدم وجود مصدر للرمز؛ ولا يستبدل إعدادات token SecretRef).
- فحص systemd linger على Linux.
- فحص حجم ملف bootstrap في workspace ‏(تحذيرات الاقتطاع/الاقتراب من الحد لملفات السياق).
- فحص حالة shell completion والتثبيت/الترقية التلقائيين.
- فحص جهوزية موفّر embedding لبحث الذاكرة (نموذج محلي، أو مفتاح API بعيد، أو ملف QMD تنفيذي).
- فحوصات تثبيت المصدر (عدم تطابق pnpm workspace، وغياب أصول UI، وغياب ملف tsx التنفيذي).
- يكتب الإعدادات المحدّثة + بيانات wizard الوصفية.

## السلوك التفصيلي والمبررات

### 0) تحديث اختياري (تثبيتات git)

إذا كان هذا مستخرج git وكان doctor يعمل في وضع تفاعلي، فإنه يعرض
التحديث (fetch/rebase/build) قبل تشغيل doctor.

### 1) تطبيع الإعدادات

إذا كانت الإعدادات تحتوي على أشكال قيم قديمة (مثل `messages.ackReaction`
من دون تجاوز خاص بالقناة)، فإن doctor يطبعها إلى المخطط الحالي.

ويتضمن ذلك حقول Talk القديمة المسطحة. إعداد Talk العام الحالي هو
`talk.provider` + `talk.providers.<provider>`. ويعيد doctor كتابة
الأشكال القديمة `talk.voiceId` / `talk.voiceAliases` / `talk.modelId` / `talk.outputFormat` /
`talk.apiKey` إلى خريطة الموفّر.

### 2) ترحيلات مفاتيح الإعدادات القديمة

عندما تحتوي الإعدادات على مفاتيح متقادمة، ترفض الأوامر الأخرى التشغيل وتطلب
منك تشغيل `openclaw doctor`.

سيقوم Doctor بما يلي:

- يشرح أي مفاتيح قديمة تم العثور عليها.
- يعرض الترحيل الذي طبقه.
- يعيد كتابة `~/.openclaw/openclaw.json` بالمخطط المحدّث.

كما أن Gateway يشغّل ترحيلات doctor تلقائيًا عند بدء التشغيل عندما يكتشف
تنسيق إعدادات قديمًا، بحيث تُصلح الإعدادات القديمة من دون تدخل يدوي.
تتم معالجة ترحيلات مخزن وظائف Cron بواسطة `openclaw doctor --fix`.

الترحيلات الحالية:

- `routing.allowFrom` → `channels.whatsapp.allowFrom`
- `routing.groupChat.requireMention` → `channels.whatsapp/telegram/imessage.groups."*".requireMention`
- `routing.groupChat.historyLimit` → `messages.groupChat.historyLimit`
- `routing.groupChat.mentionPatterns` → `messages.groupChat.mentionPatterns`
- `routing.queue` → `messages.queue`
- `routing.bindings` → `bindings` في المستوى الأعلى
- `routing.agents`/`routing.defaultAgentId` → `agents.list` + `agents.list[].default`
- القديم `talk.voiceId`/`talk.voiceAliases`/`talk.modelId`/`talk.outputFormat`/`talk.apiKey` → `talk.provider` + `talk.providers.<provider>`
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
- بالنسبة للقنوات التي تحتوي على `accounts` مسماة ولكن لا تزال فيها قيم قنوات ذات حساب واحد على المستوى الأعلى، انقل تلك القيم ذات النطاق الخاص بالحساب إلى الحساب المُرقّى المختار لتلك القناة (`accounts.default` لمعظم القنوات؛ ويمكن لـ Matrix الاحتفاظ بهدف مسمى/افتراضي مطابق موجود)
- `identity` → `agents.list[].identity`
- `agent.*` → `agents.defaults` + `tools.*` ‏(tools/elevated/exec/sandbox/subagents)
- `agent.model`/`allowedModels`/`modelAliases`/`modelFallbacks`/`imageModelFallbacks`
  → `agents.defaults.models` + `agents.defaults.model.primary/fallbacks` + `agents.defaults.imageModel.primary/fallbacks`
- `browser.ssrfPolicy.allowPrivateNetwork` → `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork`
- `browser.profiles.*.driver: "extension"` → `"existing-session"`
- إزالة `browser.relayBindHost` ‏(إعداد relay قديم للامتداد)

تتضمن تحذيرات Doctor أيضًا إرشادات حول الحساب الافتراضي للقنوات متعددة الحسابات:

- إذا تم إعداد إدخالين أو أكثر في `channels.<channel>.accounts` من دون `channels.<channel>.defaultAccount` أو `accounts.default`، فإن doctor يحذر من أن التوجيه الاحتياطي قد يختار حسابًا غير متوقع.
- إذا تم تعيين `channels.<channel>.defaultAccount` إلى معرّف حساب غير معروف، فإن doctor يحذر ويسرد معرّفات الحسابات المهيأة.

### 2b) تجاوزات موفّر OpenCode

إذا كنت قد أضفت `models.providers.opencode` أو `opencode-zen` أو `opencode-go`
يدويًا، فإنه يتجاوز كتالوج OpenCode المدمج القادم من `@mariozechner/pi-ai`.
وقد يفرض ذلك النماذج على API خاطئ أو يصفر التكاليف. ويحذر doctor لكي
تزيل التجاوز وتستعيد توجيه API + التكاليف لكل نموذج.

### 2c) ترحيل المتصفح وجهوزية Chrome MCP

إذا كانت إعدادات متصفحك لا تزال تشير إلى مسار Chrome extension الذي تمت إزالته، فإن doctor
يطبعها إلى نموذج الإرفاق الحالي المحلي للمضيف Chrome MCP:

- `browser.profiles.*.driver: "extension"` تصبح `"existing-session"`
- تتم إزالة `browser.relayBindHost`

كما يقوم doctor بتدقيق المسار المحلي للمضيف Chrome MCP عندما تستخدم `defaultProfile:
"user"` أو ملف `existing-session` شخصي مهيأ:

- يتحقق مما إذا كان Google Chrome مثبتًا على المضيف نفسه لملفات
  الاتصال التلقائي الافتراضية
- يتحقق من إصدار Chrome المكتشف ويحذر عندما يكون أقل من Chrome 144
- يذكّرك بتمكين التصحيح عن بُعد في صفحة فحص المتصفح (مثل
  `chrome://inspect/#remote-debugging` أو `brave://inspect/#remote-debugging`
  أو `edge://inspect/#remote-debugging`)

لا يمكن لـ Doctor تمكين الإعداد من جهة Chrome نيابةً عنك. ولا يزال Chrome MCP المحلي للمضيف
يتطلب:

- متصفحًا قائمًا على Chromium بإصدار 144+ على مضيف gateway/node
- أن يكون المتصفح قيد التشغيل محليًا
- تمكين التصحيح عن بُعد في ذلك المتصفح
- الموافقة على أول مطالبة موافقة على الإرفاق في المتصفح

الجهوزية هنا تتعلق فقط بمتطلبات الإرفاق المحلي. ولا يزال existing-session
يحتفظ بحدود مسارات Chrome MCP الحالية؛ أما المسارات المتقدمة مثل `responsebody` وتصدير PDF
واعتراض التنزيلات وإجراءات الدُفعات فلا تزال تتطلب
متصفحًا مُدارًا أو ملف CDP شخصيًا خامًا.

لا ينطبق هذا الفحص **على** تدفقات Docker أو sandbox أو remote-browser أو التدفقات
الأخرى عديمة الواجهة. فهي تواصل استخدام CDP الخام.

### 2d) متطلبات OAuth TLS المسبقة

عندما يتم إعداد ملف OpenAI Codex OAuth شخصي، يقوم doctor بفحص نقطة نهاية
التفويض الخاصة بـ OpenAI للتحقق من أن مكدس TLS المحلي في Node/OpenSSL يمكنه
التحقق من سلسلة الشهادات. إذا فشل الفحص بسبب خطأ في الشهادة (مثل
`UNABLE_TO_GET_ISSUER_CERT_LOCALLY` أو شهادة منتهية أو شهادة موقعة ذاتيًا)،
فإن doctor يطبع إرشادات إصلاح خاصة بالنظام. على macOS مع Node من Homebrew،
يكون الإصلاح عادةً `brew postinstall ca-certificates`. ومع `--deep`، يعمل
الفحص حتى لو كانت gateway سليمة.

### 2c) تجاوزات موفّر Codex OAuth

إذا كنت قد أضفت سابقًا إعدادات نقل OpenAI القديمة ضمن
`models.providers.openai-codex`، فقد تحجب مسار
موفّر Codex OAuth المدمج الذي تستخدمه الإصدارات الأحدث تلقائيًا. ويحذر doctor عند رؤيته
هذه الإعدادات القديمة للنقل إلى جانب Codex OAuth لكي تتمكن من إزالة أو إعادة كتابة
تجاوز النقل القديم واستعادة سلوك التوجيه/الاحتياط المدمج.
لا تزال البروكسيات المخصصة وتجاوزات الترويسات فقط مدعومة ولا تؤدي إلى هذا التحذير.

### 3) ترحيلات الحالة القديمة (بنية القرص)

يمكن لـ Doctor ترحيل البُنى القديمة على القرص إلى البنية الحالية:

- مخزن الجلسات + النصوص:
  - من `~/.openclaw/sessions/` إلى `~/.openclaw/agents/<agentId>/sessions/`
- دليل الوكيل:
  - من `~/.openclaw/agent/` إلى `~/.openclaw/agents/<agentId>/agent/`
- حالة مصادقة WhatsApp ‏(Baileys):
  - من `~/.openclaw/credentials/*.json` القديمة (باستثناء `oauth.json`)
  - إلى `~/.openclaw/credentials/whatsapp/<accountId>/...` ‏(معرّف الحساب الافتراضي: `default`)

هذه الترحيلات تبذل أفضل جهد وهي قابلة للتكرار بأمان؛ وسيُصدر doctor تحذيرات عندما
يترك أي مجلدات قديمة خلفه كنسخ احتياطية. كما أن Gateway/CLI يرحّلان تلقائيًا
الجلسات القديمة + دليل الوكيل عند بدء التشغيل بحيث تهبط السجلات/المصادقة/النماذج في
المسار الخاص بكل وكيل من دون تشغيل doctor يدويًا. أما مصادقة WhatsApp فمقصود
أن تُرحّل فقط عبر `openclaw doctor`. كما أن تطبيع موفّر Talk/خريطة الموفّرين
يقارن الآن بالمساواة البنيوية، لذلك لم تعد الفروق في ترتيب المفاتيح فقط
تؤدي إلى تغييرات متكررة عديمة الأثر في `doctor --fix`.

### 3a) ترحيلات plugin manifest القديمة

يفحص Doctor جميع plugin manifests المثبتة بحثًا عن مفاتيح
القدرات العلوية المتقادمة (`speechProviders`, `realtimeTranscriptionProviders`,
`realtimeVoiceProviders`, `mediaUnderstandingProviders`,
`imageGenerationProviders`, `videoGenerationProviders`, `webFetchProviders`,
`webSearchProviders`). وعند العثور عليها، يعرض نقلها إلى كائن `contracts`
وإعادة كتابة ملف manifest في مكانه. هذا الترحيل قابل للتكرار بأمان؛
إذا كان مفتاح `contracts` يحتوي بالفعل على القيم نفسها، فإن المفتاح القديم يُزال
من دون تكرار البيانات.

### 3b) ترحيلات مخزن cron القديمة

يتحقق Doctor أيضًا من مخزن وظائف cron ‏(`~/.openclaw/cron/jobs.json` افتراضيًا،
أو `cron.store` عند تجاوزه) بحثًا عن أشكال وظائف قديمة لا يزال المجدول
يقبلها للتوافق.

تنظيفات cron الحالية تشمل:

- `jobId` → `id`
- `schedule.cron` → `schedule.expr`
- حقول payload ذات المستوى الأعلى (`message`, `model`, `thinking`, ...) → `payload`
- حقول delivery ذات المستوى الأعلى (`deliver`, `channel`, `to`, `provider`, ...) → `delivery`
- أسماء delivery البديلة في `provider` داخل payload → `delivery.channel` صريح
- وظائف fallback القديمة البسيطة من نوع `notify: true` webhook → `delivery.mode="webhook"` صريح مع `delivery.to=cron.webhook`

لا يرحّل Doctor وظائف `notify: true` تلقائيًا إلا عندما يستطيع فعل ذلك من دون
تغيير السلوك. وإذا كانت الوظيفة تجمع بين fallback الإشعار القديم ووضع
delivery موجود غير webhook، فإن doctor يحذر ويترك تلك الوظيفة للمراجعة اليدوية.

### 3c) تنظيف قفل الجلسة

يفحص Doctor كل دليل جلسات وكلاء بحثًا عن ملفات write-lock قديمة —
أي الملفات التي تُترك خلفها عندما تخرج الجلسة بشكل غير طبيعي. ولكل ملف قفل يعثر عليه يبلغ عن:
المسار، وPID، وما إذا كان PID ما يزال حيًا، وعمر القفل، وما إذا كان
يُعد قديمًا (PID ميت أو أقدم من 30 دقيقة). وفي وضع `--fix` / `--repair`
فإنه يزيل ملفات القفل القديمة تلقائيًا؛ وإلا فإنه يطبع ملاحظة
ويطلب منك إعادة التشغيل باستخدام `--fix`.

### 4) فحوصات تكامل الحالة (استمرارية الجلسة، والتوجيه، والسلامة)

دليل الحالة هو الجذع التشغيلي الأساسي. وإذا اختفى، فستفقد
الجلسات، وبيانات الاعتماد، والسجلات، والإعدادات (ما لم تكن لديك نسخ احتياطية في مكان آخر).

يتحقق Doctor من:

- **غياب دليل الحالة**: يحذر من فقدان كارثي للحالة، ويطلب إعادة إنشاء
  الدليل، ويذكرك بأنه لا يستطيع استعادة البيانات المفقودة.
- **أذونات دليل الحالة**: يتحقق من إمكانية الكتابة؛ ويعرض إصلاح الأذونات
  (ويصدر تلميح `chown` عند اكتشاف عدم تطابق المالك/المجموعة).
- **دليل حالة متزامن عبر السحابة على macOS**: يحذر عندما تُحل الحالة تحت iCloud Drive
  (`~/Library/Mobile Documents/com~apple~CloudDocs/...`) أو
  `~/Library/CloudStorage/...` لأن المسارات المدعومة بالمزامنة قد تسبب بطء I/O
  وسباقات في القفل/المزامنة.
- **دليل حالة على SD أو eMMC في Linux**: يحذر عندما تُحل الحالة إلى مصدر ربط `mmcblk*`،
  لأن I/O العشوائي المدعوم ببطاقات SD أو eMMC قد يكون أبطأ ويتعرض للتآكل
  بشكل أسرع تحت عمليات كتابة الجلسات وبيانات الاعتماد.
- **غياب أدلة الجلسات**: إن `sessions/` ودليل مخزن الجلسات
  مطلوبان لحفظ السجل وتجنب أعطال `ENOENT`.
- **عدم تطابق النصوص**: يحذر عندما تكون إدخالات الجلسات الحديثة تفتقد
  إلى ملفات النصوص.
- **الجلسة الرئيسية "JSONL من سطر واحد"**: يعلّم عندما يحتوي النص الرئيسي على سطر واحد فقط
  (السجل لا يتراكم).
- **أدلة حالة متعددة**: يحذر عند وجود عدة مجلدات `~/.openclaw` عبر
  الأدلة المنزلية أو عندما يشير `OPENCLAW_STATE_DIR` إلى مكان آخر (قد ينقسم السجل
  بين التثبيتات).
- **تذكير الوضع البعيد**: إذا كان `gateway.mode=remote`، يذكرك doctor بتشغيله
  على المضيف البعيد (الحالة موجودة هناك).
- **أذونات ملف الإعدادات**: يحذر إذا كان `~/.openclaw/openclaw.json`
  قابلاً للقراءة من المجموعة/العالم ويعرض تشديده إلى `600`.

### 5) سلامة مصادقة النموذج (انتهاء OAuth)

يفحص Doctor ملفات OAuth الشخصية في مخزن المصادقة، ويحذر عندما تكون الرموز
على وشك الانتهاء/منتهية، ويمكنه تحديثها عندما يكون ذلك آمنًا. وإذا كان ملف
Anthropic OAuth/token الشخصي قديمًا، فإنه يقترح مفتاح Anthropic API أو
مسار إعداد رمز Anthropic.
لا تظهر مطالبات التحديث إلا عند التشغيل تفاعليًا (TTY)؛ أما `--non-interactive`
فيتخطى محاولات التحديث.

كما يبلغ Doctor عن ملفات المصادقة الشخصية التي تكون غير قابلة للاستخدام مؤقتًا بسبب:

- فترات تهدئة قصيرة (حدود المعدل/انتهاء المهلة/فشل المصادقة)
- تعطيلات أطول (فشل الفوترة/الرصيد)

### 6) التحقق من نموذج Hooks

إذا كان `hooks.gmail.model` مضبوطًا، فإن doctor يتحقق من مرجع النموذج مقابل
الكتالوج وقائمة السماح ويحذر عندما لا يمكن حله أو يكون غير مسموح به.

### 7) إصلاح صورة sandbox

عند تفعيل العزل، يتحقق doctor من صور Docker ويعرض إنشاءها أو
التبديل إلى الأسماء القديمة إذا كانت الصورة الحالية مفقودة.

### 7b) تبعيات وقت تشغيل plugin المضمنة

يتحقق Doctor من وجود تبعيات وقت تشغيل plugin المضمنة (مثل
حزم وقت تشغيل Discord plugin) في جذر تثبيت OpenClaw.
إذا كان أي منها مفقودًا، فإن doctor يبلغ عن الحزم ويثبتها في وضع
`openclaw doctor --fix` / `openclaw doctor --repair`.

### 8) ترحيلات خدمة Gateway وتلميحات التنظيف

يكتشف Doctor خدمات gateway القديمة (launchd/systemd/schtasks) و
يعرض إزالتها وتثبيت خدمة OpenClaw باستخدام منفذ gateway الحالي.
كما يمكنه فحص الخدمات الإضافية الشبيهة بـ gateway وطباعة تلميحات تنظيف.
تُعد خدمات OpenClaw gateway المسماة بحسب الملف الشخصي خدمات من الدرجة الأولى ولا
تُعلَّم على أنها "إضافية".

### 8b) ترحيل Matrix عند بدء التشغيل

عندما يكون لدى حساب قناة Matrix ترحيل حالة قديم معلق أو قابل للتنفيذ،
فإن doctor (في وضع `--fix` / `--repair`) ينشئ لقطة قبل الترحيل ثم
يشغّل خطوات الترحيل بأفضل جهد: ترحيل حالة Matrix القديمة وإعداد الحالة المشفرة القديمة.
كلتا الخطوتين غير قاتلتين؛ يتم تسجيل الأخطاء ويستمر بدء التشغيل. أما في وضع القراءة فقط (`openclaw doctor` من دون `--fix`) فيتم
تخطي هذا الفحص بالكامل.

### 9) التحذيرات الأمنية

يصدر Doctor تحذيرات عندما يكون موفّر ما مفتوحًا لرسائل DM من دون allowlist، أو
عندما تكون سياسة ما مهيأة بطريقة خطرة.

### 10) systemd linger ‏(Linux)

إذا كان التشغيل يتم كخدمة مستخدم systemd، فإن doctor يضمن تمكين lingering بحيث
تبقى gateway حية بعد تسجيل الخروج.

### 11) حالة workspace ‏(Skills وplugins والأدلة القديمة)

يطبع Doctor ملخصًا عن حالة workspace للوكيل الافتراضي:

- **حالة Skills**: يحسب Skills المؤهلة وتلك التي تفتقد المتطلبات وتلك المحجوبة بقائمة السماح.
- **أدلة workspace القديمة**: يحذر عند وجود `~/openclaw` أو أدلة workspace قديمة أخرى
  إلى جانب workspace الحالي.
- **حالة plugin**: يحسب plugins المحملة/المعطلة/ذات الأخطاء؛ ويسرد معرفات plugin لأي
  أخطاء؛ ويبلغ عن قدرات bundle plugin.
- **تحذيرات توافق plugin**: يعلّم plugins التي لديها مشكلات توافق مع
  وقت التشغيل الحالي.
- **تشخيصات plugin**: يعرض أي تحذيرات أو أخطاء وقت التحميل صادرة من
  سجل plugin.

### 11b) حجم ملف Bootstrap

يتحقق Doctor مما إذا كانت ملفات bootstrap في workspace ‏(مثل `AGENTS.md`,
و`CLAUDE.md`، أو ملفات السياق الأخرى المحقونة) قريبة من حد ميزانية الأحرف
أو متجاوزه. ويبلغ لكل ملف عن عدد الأحرف الخام مقابل المحقون، ونسبة الاقتطاع،
وسبب الاقتطاع (`max/file` أو `max/total`)، وإجمالي الأحرف المحقونة
بوصفها جزءًا من الميزانية الكلية. وعندما تكون الملفات مقتطعة أو قريبة من
الحد، يطبع doctor نصائح لضبط `agents.defaults.bootstrapMaxChars`
و`agents.defaults.bootstrapTotalMaxChars`.

### 11c) Shell completion

يتحقق Doctor مما إذا كان إكمال التبويب مثبتًا للصدفة الحالية
(zsh أو bash أو fish أو PowerShell):

- إذا كان ملف shell الشخصي يستخدم نمط إكمال ديناميكي بطيء
  (`source <(openclaw completion ...)`)، فإن doctor يرقّيه إلى الصيغة
  الأسرع المعتمدة على الملف المخزن مؤقتًا.
- إذا كان الإكمال مهيأ في الملف الشخصي ولكن ملف التخزين المؤقت مفقودًا،
  فإن doctor يعيد توليد التخزين المؤقت تلقائيًا.
- إذا لم يكن هناك أي إكمال مهيأ أصلًا، فإن doctor يطلب تثبيته
  (في الوضع التفاعلي فقط؛ ويتم تخطيه مع `--non-interactive`).

شغّل `openclaw completion --write-state` لإعادة توليد التخزين المؤقت يدويًا.

### 12) فحوصات مصادقة Gateway ‏(الرمز المحلي)

يتحقق Doctor من جهوزية مصادقة رمز gateway المحلي.

- إذا كان وضع الرمز يحتاج إلى رمز ولا يوجد مصدر له، فإن doctor يعرض توليد واحد.
- إذا كان `gateway.auth.token` مدارًا بواسطة SecretRef ولكنه غير متاح، فإن doctor يحذر ولا يستبدله بنص صريح.
- يفرض `openclaw doctor --generate-gateway-token` التوليد فقط عندما لا يكون هناك token SecretRef مهيأ.

### 12b) إصلاحات واعية بـ SecretRef للقراءة فقط

تحتاج بعض تدفقات الإصلاح إلى فحص بيانات الاعتماد المهيأة دون إضعاف سلوك الفشل السريع في وقت التشغيل.

- يستخدم `openclaw doctor --fix` الآن نموذج الملخص نفسه المعتمد على SecretRef للقراءة فقط كما في أوامر عائلة الحالة، وذلك لإصلاحات إعدادات مستهدفة.
- مثال: يحاول إصلاح Telegram `allowFrom` / `groupAllowFrom` لأسماء `@username` استخدام بيانات اعتماد البوت المهيأة عندما تكون متاحة.
- إذا كان رمز Telegram bot مهيأ عبر SecretRef ولكنه غير متاح في مسار الأمر الحالي، فإن doctor يبلغ أن بيانات الاعتماد مهيأة-لكن-غير-متاحة ويتخطى الحل التلقائي بدلًا من التعطل أو الإبلاغ خطأً عن أن الرمز مفقود.

### 13) فحص سلامة Gateway + إعادة التشغيل

يشغّل Doctor فحص سلامة ويعرض إعادة تشغيل gateway عندما تبدو
غير سليمة.

### 13b) جهوزية بحث الذاكرة

يتحقق Doctor مما إذا كان موفّر embedding المهيأ لبحث الذاكرة جاهزًا
للوكيل الافتراضي. ويعتمد السلوك على الواجهة الخلفية والموفّر المهيأين:

- **واجهة QMD الخلفية**: يفحص ما إذا كان الملف التنفيذي `qmd` متاحًا وقابلًا للتشغيل.
  وإذا لم يكن كذلك، يطبع إرشادات إصلاح تتضمن حزمة npm وخيار مسار يدوي للملف التنفيذي.
- **موفّر محلي صريح**: يتحقق من وجود ملف نموذج محلي أو عنوان URL معروف
  لنموذج بعيد/قابل للتنزيل. وإذا كان مفقودًا، يقترح التبديل إلى موفّر بعيد.
- **موفّر بعيد صريح** (`openai` أو `voyage` وغيرهما): يتحقق من وجود مفتاح API
  في البيئة أو مخزن المصادقة. ويطبع تلميحات إصلاح قابلة للتنفيذ إذا كان مفقودًا.
- **موفّر تلقائي**: يتحقق أولًا من توفر النموذج المحلي، ثم يجرب كل موفّر بعيد
  وفق ترتيب الاختيار التلقائي.

عندما تكون نتيجة فحص gateway متاحة (كانت gateway سليمة وقت
الفحص)، فإن doctor يطابق نتيجتها مع الإعدادات المرئية في CLI ويشير
إلى أي اختلاف.

استخدم `openclaw memory status --deep` للتحقق من جهوزية embedding في وقت التشغيل.

### 14) تحذيرات حالة القناة

إذا كانت gateway سليمة، فإن doctor يشغّل فحص حالة القناة ويبلغ
عن التحذيرات مع الإصلاحات المقترحة.

### 15) تدقيق إعدادات Supervisor + الإصلاح

يتحقق Doctor من إعدادات supervisor المثبتة (launchd/systemd/schtasks) بحثًا عن
الافتراضيات المفقودة أو القديمة (مثل تبعيات network-online في systemd و
تأخير إعادة التشغيل). وعندما يجد عدم تطابق، فإنه يوصي بالتحديث ويمكنه
إعادة كتابة ملف الخدمة/المهمة إلى الافتراضيات الحالية.

ملاحظات:

- يطلب `openclaw doctor` تأكيدًا قبل إعادة كتابة إعدادات supervisor.
- يقبل `openclaw doctor --yes` مطالبات الإصلاح الافتراضية.
- يطبق `openclaw doctor --repair` الإصلاحات الموصى بها من دون مطالبات.
- يستبدل `openclaw doctor --repair --force` إعدادات supervisor المخصصة.
- إذا كانت مصادقة الرمز تتطلب رمزًا وكان `gateway.auth.token` مدارًا بواسطة SecretRef، فإن doctor عند تثبيت/إصلاح الخدمة يتحقق من SecretRef لكنه لا يحفظ قيم الرموز الصريحة المحلولة في بيانات البيئة الوصفية لخدمة supervisor.
- إذا كانت مصادقة الرمز تتطلب رمزًا وكان token SecretRef المهيأ غير محلول، فإن doctor يمنع مسار التثبيت/الإصلاح مع إرشادات قابلة للتنفيذ.
- إذا كان كل من `gateway.auth.token` و`gateway.auth.password` مهيأين وكان `gateway.auth.mode` غير مضبوط، فإن doctor يمنع التثبيت/الإصلاح حتى يُضبط الوضع صراحة.
- بالنسبة لوحدات Linux user-systemd، فإن فحوصات انجراف الرمز في doctor تشمل الآن كلا مصدري `Environment=` و`EnvironmentFile=` عند مقارنة بيانات المصادقة الوصفية للخدمة.
- يمكنك دائمًا فرض إعادة كتابة كاملة عبر `openclaw gateway install --force`.

### 16) تشخيصات وقت تشغيل Gateway + المنفذ

يفحص Doctor وقت تشغيل الخدمة (PID، وآخر حالة خروج) ويحذر عندما تكون
الخدمة مثبتة ولكنها لا تعمل فعليًا. كما يتحقق من تعارضات المنفذ
على منفذ gateway ‏(الافتراضي `18789`) ويبلغ عن الأسباب المحتملة (gateway تعمل بالفعل،
أو نفق SSH).

### 17) أفضل ممارسات وقت تشغيل Gateway

يحذر Doctor عندما تعمل خدمة gateway على Bun أو على مسار Node مُدار بمدير إصدارات
(`nvm` أو `fnm` أو `volta` أو `asdf` وغيرها). تتطلب قنوات WhatsApp + Telegram استخدام Node،
وقد تتعطل مسارات مديري الإصدارات بعد الترقية لأن الخدمة لا تحمّل
تهيئة الصدفة الخاصة بك. ويعرض doctor الترحيل إلى تثبيت Node على مستوى النظام عند
توفره (Homebrew/apt/choco).

### 18) كتابة الإعدادات + بيانات wizard الوصفية

يحفظ Doctor أي تغييرات في الإعدادات ويضع بيانات وصفية لـ wizard لتسجيل
تشغيل doctor.

### 19) تلميحات workspace ‏(النسخ الاحتياطي + نظام الذاكرة)

يقترح Doctor نظام ذاكرة لـ workspace عند غيابه ويطبع تلميح نسخ احتياطي
إذا لم يكن workspace موجودًا أصلًا تحت git.

راجع [/concepts/agent-workspace](/ar/concepts/agent-workspace) للحصول على دليل كامل حول
بنية workspace والنسخ الاحتياطي عبر git ‏(يوصى باستخدام GitHub أو GitLab خاص).
