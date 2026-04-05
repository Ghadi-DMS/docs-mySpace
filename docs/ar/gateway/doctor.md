---
read_when:
    - إضافة ترحيلات Doctor أو تعديلها
    - إدخال تغييرات كاسرة في الإعداد
summary: 'أمر Doctor: فحوصات السلامة، وترحيلات الإعداد، وخطوات الإصلاح'
title: Doctor
x-i18n:
    generated_at: "2026-04-05T12:43:35Z"
    model: gpt-5.4
    provider: openai
    source_hash: 119080ef6afe1b14382a234f844ea71336923355d991fe6d816fddc6c83cf88f
    source_path: gateway/doctor.md
    workflow: 15
---

# Doctor

يُعد `openclaw doctor` أداة الإصلاح + الترحيل في OpenClaw. فهو يصلح
الحالة/الإعداد القديمين، ويفحص السلامة، ويوفر خطوات إصلاح عملية.

## بدء سريع

```bash
openclaw doctor
```

### وضع headless / الأتمتة

```bash
openclaw doctor --yes
```

اقبل القيم الافتراضية من دون مطالبة (بما في ذلك خطوات إصلاح إعادة التشغيل/الخدمة/sandbox عند الاقتضاء).

```bash
openclaw doctor --repair
```

طبّق الإصلاحات الموصى بها من دون مطالبة (الإصلاحات + إعادة التشغيل حيثما كان ذلك آمنًا).

```bash
openclaw doctor --repair --force
```

طبّق الإصلاحات العدوانية أيضًا (يستبدل إعدادات supervisor المخصصة).

```bash
openclaw doctor --non-interactive
```

شغّل من دون مطالبات وطبّق فقط الترحيلات الآمنة (تطبيع الإعداد + نقل الحالة على القرص). ويتخطى إجراءات إعادة التشغيل/الخدمة/sandbox التي تتطلب تأكيدًا بشريًا.
تعمل ترحيلات الحالة القديمة تلقائيًا عند اكتشافها.

```bash
openclaw doctor --deep
```

افحص خدمات النظام بحثًا عن تثبيتات gateway إضافية (launchd/systemd/schtasks).

إذا كنت تريد مراجعة التغييرات قبل الكتابة، فافتح ملف الإعداد أولًا:

```bash
cat ~/.openclaw/openclaw.json
```

## ما الذي يفعله (ملخص)

- تحديث تمهيدي اختياري لتثبيتات git ‏(تفاعلي فقط).
- فحص حداثة بروتوكول UI ‏(يعيد بناء Control UI عندما يكون schema البروتوكول أحدث).
- فحص السلامة + مطالبة بإعادة التشغيل.
- ملخص حالة Skills ‏(القابلة للاستخدام/المفقودة/المحجوبة) وحالة المكونات الإضافية.
- تطبيع الإعداد للقيم القديمة.
- ترحيل إعداد Talk من الحقول القديمة المسطحة `talk.*` إلى `talk.provider` + `talk.providers.<provider>`.
- فحوصات ترحيل المتصفح لإعدادات Chrome extension القديمة وجاهزية Chrome MCP.
- تحذيرات تجاوز مزود OpenCode ‏(`models.providers.opencode` / `models.providers.opencode-go`).
- فحص متطلبات TLS لـ OAuth الخاصة بـ OpenAI Codex profiles.
- ترحيل الحالة القديمة على القرص (الجلسات/دليل الوكيل/مصادقة WhatsApp).
- ترحيل مفاتيح عقد manifest القديمة للمكونات الإضافية (`speechProviders`, `realtimeTranscriptionProviders`, `realtimeVoiceProviders`, `mediaUnderstandingProviders`, `imageGenerationProviders`, `videoGenerationProviders`, `webFetchProviders`, `webSearchProviders` → `contracts`).
- ترحيل مخزن cron القديم (`jobId`, `schedule.cron`, حقول delivery/payload ذات المستوى الأعلى، و`provider` داخل payload، ووظائف webhook fallback القديمة البسيطة `notify: true`).
- فحص ملفات قفل الجلسات وتنظيف الأقفال القديمة.
- فحوصات سلامة الحالة والأذونات (الجلسات، والنصوص، ودليل الحالة).
- فحوصات أذونات ملف الإعداد (chmod 600) عند التشغيل محليًا.
- سلامة مصادقة النموذج: يفحص انتهاء OAuth، ويمكنه تحديث الرموز القريبة من الانتهاء، ويبلغ عن حالات التهدئة/التعطيل في auth-profile.
- اكتشاف دليل مساحة عمل إضافي (`~/openclaw`).
- إصلاح صورة sandbox عند تفعيل sandboxing.
- ترحيل الخدمة القديمة واكتشاف gateway الإضافي.
- ترحيل الحالة القديمة لقناة Matrix ‏(في وضع `--fix` / `--repair`).
- فحوصات وقت تشغيل Gateway ‏(الخدمة مثبتة لكنها لا تعمل؛ وlaunchd label مخزن مؤقتًا).
- تحذيرات حالة القنوات (يتم فحصها من gateway قيد التشغيل).
- تدقيق إعداد supervisor ‏(launchd/systemd/schtasks) مع إصلاح اختياري.
- فحوصات أفضل الممارسات لوقت تشغيل Gateway ‏(Node مقابل Bun، ومسارات مديري الإصدارات).
- تشخيص تعارض منفذ Gateway ‏(الافتراضي `18789`).
- تحذيرات أمنية لسياسات DM المفتوحة.
- فحوصات مصادقة Gateway لوضع token المحلي (يعرض توليد token عندما لا يوجد مصدر token؛ ولا يستبدل إعدادات token SecretRef).
- فحص systemd linger على Linux.
- فحص حجم ملفات bootstrap الخاصة بمساحة العمل (تحذيرات القص/الاقتراب من الحد لملفات السياق).
- فحص حالة shell completion والتثبيت/الترقية التلقائيين.
- فحص جاهزية موفر embedding الخاص ببحث الذاكرة (نموذج محلي، أو مفتاح API بعيد، أو ثنائي QMD).
- فحوصات تثبيتات المصدر (عدم تطابق pnpm workspace، وأصول UI المفقودة، وثنائي tsx المفقود).
- يكتب الإعداد المحدَّث + بيانات وصفية للمعالج.

## السلوك التفصيلي والمبررات

### 0) تحديث اختياري (تثبيتات git)

إذا كان هذا checkout من git وكان doctor يعمل تفاعليًا، فإنه يعرض
التحديث (fetch/rebase/build) قبل تشغيل doctor.

### 1) تطبيع الإعداد

إذا كان الإعداد يحتوي على أشكال قيم قديمة (مثل `messages.ackReaction`
من دون تجاوز خاص بالقناة)، فإن doctor يطبعها إلى
schema الحالية.

ويشمل ذلك الحقول المسطحة القديمة لـ Talk. فإعداد Talk العام الحالي هو
`talk.provider` + `talk.providers.<provider>`. ويعيد doctor كتابة
الأشكال القديمة `talk.voiceId` / `talk.voiceAliases` / `talk.modelId` / `talk.outputFormat` /
`talk.apiKey` إلى خريطة المزوّد.

### 2) ترحيلات مفاتيح الإعداد القديمة

عندما يحتوي الإعداد على مفاتيح مهجورة، ترفض الأوامر الأخرى العمل
وتطلب منك تشغيل `openclaw doctor`.

وسيقوم doctor بما يلي:

- شرح مفاتيح legacy التي عُثر عليها.
- عرض الترحيل الذي طبّقه.
- إعادة كتابة `~/.openclaw/openclaw.json` باستخدام schema المحدّثة.

كما يشغّل Gateway تلقائيًا ترحيلات doctor عند البدء عندما يكتشف
تنسيق إعداد قديمًا، لذلك تُصلح الإعدادات القديمة من دون تدخل يدوي.
وتُعالَج ترحيلات مخزن وظائف cron بواسطة `openclaw doctor --fix`.

الترحيلات الحالية:

- `routing.allowFrom` → `channels.whatsapp.allowFrom`
- `routing.groupChat.requireMention` → `channels.whatsapp/telegram/imessage.groups."*".requireMention`
- `routing.groupChat.historyLimit` → `messages.groupChat.historyLimit`
- `routing.groupChat.mentionPatterns` → `messages.groupChat.mentionPatterns`
- `routing.queue` → `messages.queue`
- `routing.bindings` → `bindings` ذات المستوى الأعلى
- `routing.agents`/`routing.defaultAgentId` → `agents.list` + `agents.list[].default`
- الحقول القديمة `talk.voiceId`/`talk.voiceAliases`/`talk.modelId`/`talk.outputFormat`/`talk.apiKey` → `talk.provider` + `talk.providers.<provider>`
- `routing.agentToAgent` → `tools.agentToAgent`
- `routing.transcribeAudio` → `tools.media.audio.models`
- `messages.tts.<provider>` ‏(`openai`/`elevenlabs`/`microsoft`/`edge`) → `messages.tts.providers.<provider>`
- `channels.discord.voice.tts.<provider>` ‏(`openai`/`elevenlabs`/`microsoft`/`edge`) → `channels.discord.voice.tts.providers.<provider>`
- `channels.discord.accounts.<id>.voice.tts.<provider>` ‏(`openai`/`elevenlabs`/`microsoft`/`edge`) → `channels.discord.accounts.<id>.voice.tts.providers.<provider>`
- `plugins.entries.voice-call.config.tts.<provider>` ‏(`openai`/`elevenlabs`/`microsoft`/`edge`) → `plugins.entries.voice-call.config.tts.providers.<provider>`
- `plugins.entries.voice-call.config.provider: "log"` → `"mock"`
- `plugins.entries.voice-call.config.twilio.from` → `plugins.entries.voice-call.config.fromNumber`
- `plugins.entries.voice-call.config.streaming.sttProvider` → `plugins.entries.voice-call.config.streaming.provider`
- `plugins.entries.voice-call.config.streaming.openaiApiKey|sttModel|silenceDurationMs|vadThreshold`
  → `plugins.entries.voice-call.config.streaming.providers.openai.*`
- `bindings[].match.accountID` → `bindings[].match.accountId`
- بالنسبة إلى القنوات التي تحتوي على `accounts` مسماة مع بقاء قيم قناة ذات حساب واحد على المستوى الأعلى، انقل تلك القيم ذات النطاق الحسابي إلى الحساب المُرقّى المختار لتلك القناة (`accounts.default` لمعظم القنوات؛ ويمكن لـ Matrix الحفاظ على هدف مسمى/افتراضي مطابق موجود)
- `identity` → `agents.list[].identity`
- `agent.*` → `agents.defaults` + `tools.*` ‏(tools/elevated/exec/sandbox/subagents)
- `agent.model`/`allowedModels`/`modelAliases`/`modelFallbacks`/`imageModelFallbacks`
  → `agents.defaults.models` + `agents.defaults.model.primary/fallbacks` + `agents.defaults.imageModel.primary/fallbacks`
- `browser.ssrfPolicy.allowPrivateNetwork` → `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork`
- `browser.profiles.*.driver: "extension"` → `"existing-session"`
- إزالة `browser.relayBindHost` ‏(إعداد relay قديم للامتداد)

تتضمن تحذيرات doctor أيضًا إرشادات حول الحساب الافتراضي لقنوات الحسابات المتعددة:

- إذا جرى إعداد إدخالين أو أكثر ضمن `channels.<channel>.accounts` من دون `channels.<channel>.defaultAccount` أو `accounts.default`، يحذّر doctor من أن التوجيه الاحتياطي قد يختار حسابًا غير متوقع.
- إذا ضُبط `channels.<channel>.defaultAccount` على معرّف حساب غير معروف، يحذّر doctor ويعرض معرّفات الحسابات المكوّنة.

### 2b) تجاوزات مزود OpenCode

إذا أضفت يدويًا `models.providers.opencode` أو `opencode-zen` أو `opencode-go`،
فإن ذلك يتجاوز فهرس OpenCode المضمّن من `@mariozechner/pi-ai`.
وقد يفرض ذلك على النماذج استخدام API خاطئة أو يصفر التكاليف. ويحذّر doctor لكي
تتمكن من إزالة التجاوز واستعادة توجيه API + التكاليف لكل نموذج.

### 2c) ترحيل المتصفح وجاهزية Chrome MCP

إذا كان إعداد المتصفح ما يزال يشير إلى مسار Chrome extension المُزال، فإن doctor
يطبع ذلك إلى نموذج الإرفاق الحالي المحلي على المضيف لـ Chrome MCP:

- `browser.profiles.*.driver: "extension"` يصبح `"existing-session"`
- تتم إزالة `browser.relayBindHost`

كما يدقق doctor في المسار المحلي على المضيف لـ Chrome MCP عندما تستخدم `defaultProfile:
"user"` أو profile `existing-session` مكوَّنة:

- يتحقق مما إذا كان Google Chrome مثبتًا على المضيف نفسه لملفات
  الاتصال التلقائي الافتراضية
- يتحقق من إصدار Chrome المكتشف ويحذر عندما يكون أقل من Chrome 144
- يذكّرك بتفعيل remote debugging في صفحة inspect للمتصفح (مثل
  `chrome://inspect/#remote-debugging`, `brave://inspect/#remote-debugging`,
  أو `edge://inspect/#remote-debugging`)

لا يستطيع doctor تفعيل إعداد Chrome من جانب المتصفح نيابةً عنك. فما يزال Chrome MCP المحلي على المضيف
يتطلب:

- متصفحًا مبنيًا على Chromium بالإصدار 144+ على مضيف gateway/node
- أن يعمل المتصفح محليًا
- تفعيل remote debugging في ذلك المتصفح
- الموافقة على مطالبة consent الأولى الخاصة بالإرفاق في المتصفح

الجاهزية هنا تتعلق فقط بمتطلبات الإرفاق المحلي. ويحتفظ existing-session
بحدود مسار Chrome MCP الحالي؛ أما المسارات المتقدمة مثل `responsebody`، وتصدير PDF،
واعتراض التنزيل، والإجراءات الدفعية، فما تزال تتطلب متصفحًا مُدارًا
أو profile CDP خامًا.

لا ينطبق هذا الفحص على تدفقات Docker أو sandbox أو remote-browser أو
أي تدفقات headless أخرى. فهذه تواصل استخدام CDP الخام.

### 2d) متطلبات OAuth TLS

عند إعداد OpenAI Codex OAuth profile، يقوم doctor بفحص نقطة نهاية التفويض في OpenAI
للتحقق من أن مكدس TLS المحلي في Node/OpenSSL يمكنه التحقق من سلسلة الشهادات.
وإذا فشل الفحص بخطأ شهادة (مثل
`UNABLE_TO_GET_ISSUER_CERT_LOCALLY`، أو شهادة منتهية، أو شهادة self-signed)،
يطبع doctor إرشادات إصلاح خاصة بالنظام. وعلى macOS مع Homebrew Node، يكون
الإصلاح عادةً `brew postinstall ca-certificates`. ومع `--deep`، يعمل الفحص
حتى إذا كان gateway سليمًا.

### 3) ترحيلات الحالة القديمة (تخطيط القرص)

يمكن لـ doctor ترحيل تخطيطات القرص القديمة إلى البنية الحالية:

- مخزن الجلسات + النصوص:
  - من `~/.openclaw/sessions/` إلى `~/.openclaw/agents/<agentId>/sessions/`
- دليل الوكيل:
  - من `~/.openclaw/agent/` إلى `~/.openclaw/agents/<agentId>/agent/`
- حالة مصادقة WhatsApp ‏(Baileys):
  - من `~/.openclaw/credentials/*.json` القديمة (باستثناء `oauth.json`)
  - إلى `~/.openclaw/credentials/whatsapp/<accountId>/...` ‏(معرّف الحساب الافتراضي: `default`)

هذه الترحيلات تعمل على أساس أفضل جهد وهي idempotent؛ وسيصدر doctor تحذيرات عندما
يترك أي مجلدات قديمة خلفه كنسخ احتياطية. كما يقوم Gateway/CLI بترحيل
الجلسات القديمة + دليل الوكيل تلقائيًا عند البدء بحيث تنتقل السجلات/المصادقة/النماذج إلى
المسار الخاص بكل وكيل من دون الحاجة إلى تشغيل doctor يدويًا. أما مصادقة WhatsApp فمقصود بها أن
تُرحَّل فقط عبر `openclaw doctor`. كما أن تطبيع موفر Talk/خريطة المزوّد
يقارن الآن وفق المساواة البنيوية، لذلك لم تعد الفروق الخاصة بترتيب المفاتيح فقط
تؤدي إلى تغييرات no-op متكررة في `doctor --fix`.

### 3a) ترحيلات manifest القديمة للمكونات الإضافية

يفحص doctor جميع ملفات manifest للمكونات الإضافية المثبتة بحثًا عن
مفاتيح الإمكانات القديمة ذات المستوى الأعلى
(`speechProviders`, `realtimeTranscriptionProviders`,
`realtimeVoiceProviders`, `mediaUnderstandingProviders`,
`imageGenerationProviders`, `videoGenerationProviders`, `webFetchProviders`,
`webSearchProviders`). وعند العثور عليها، يعرض نقلها إلى الكائن `contracts`
وإعادة كتابة ملف manifest في مكانه. وهذه الترحيل idempotent؛
فإذا كان مفتاح `contracts` يحتوي بالفعل على القيم نفسها، يُزال المفتاح القديم
من دون تكرار البيانات.

### 3b) ترحيلات مخزن cron القديمة

يفحص doctor أيضًا مخزن وظائف cron ‏(`~/.openclaw/cron/jobs.json` افتراضيًا،
أو `cron.store` عند تجاوزه) بحثًا عن أشكال وظائف قديمة ما يزال المجدول يقبلها
لأغراض التوافق.

تشمل عمليات تنظيف cron الحالية:

- `jobId` → `id`
- `schedule.cron` → `schedule.expr`
- حقول payload ذات المستوى الأعلى (`message`, `model`, `thinking`, ...) → `payload`
- حقول delivery ذات المستوى الأعلى (`deliver`, `channel`, `to`, `provider`, ...) → `delivery`
- الأسماء البديلة الخاصة بـ delivery `provider` داخل payload → `delivery.channel` صريح
- وظائف webhook fallback القديمة البسيطة `notify: true` → `delivery.mode="webhook"` صريح مع `delivery.to=cron.webhook`

لا يرحّل doctor تلقائيًا وظائف `notify: true` إلا عندما يستطيع
القيام بذلك من دون تغيير السلوك. فإذا كانت الوظيفة تجمع بين notify fallback القديم ووضع
delivery غير webhook موجود، يحذّر doctor ويترك تلك الوظيفة للمراجعة اليدوية.

### 3c) تنظيف أقفال الجلسات

يفحص doctor كل دليل جلسات للوكلاء بحثًا عن ملفات أقفال كتابة قديمة — وهي الملفات التي تُترك
عندما تنتهي الجلسة بشكل غير طبيعي. ولكل ملف قفل يعثر عليه، يبلّغ بما يلي:
المسار، وPID، وما إذا كان PID ما يزال حيًا، وعمر القفل، وما إذا كان
يُعد قديمًا (PID ميت أو أقدم من 30 دقيقة). وفي وضع `--fix` / `--repair`
يزيل ملفات الأقفال القديمة تلقائيًا؛ أما في الوضع للقراءة فقط فيطبع ملاحظة
ويطلب منك إعادة التشغيل مع `--fix`.

### 4) فحوصات سلامة الحالة (استمرارية الجلسة، والتوجيه، والسلامة)

يُعد دليل الحالة بمثابة جذع الدماغ التشغيلي. وإذا اختفى، فستفقد
الجلسات، وبيانات الاعتماد، والسجلات، والإعداد (ما لم تكن لديك نسخ احتياطية في مكان آخر).

يفحص doctor ما يلي:

- **دليل الحالة مفقود**: يحذّر من فقدان كارثي للحالة، ويطلب إعادة إنشاء
  الدليل، ويذكّرك بأنه لا يستطيع استعادة البيانات المفقودة.
- **أذونات دليل الحالة**: يتحقق من إمكانية الكتابة؛ ويعرض إصلاح الأذونات
  (ويصدر تلميح `chown` عند اكتشاف عدم تطابق المالك/المجموعة).
- **دليل حالة macOS المتزامن سحابيًا**: يحذّر عندما تُحل الحالة تحت iCloud Drive
  (`~/Library/Mobile Documents/com~apple~CloudDocs/...`) أو
  `~/Library/CloudStorage/...` لأن المسارات المعتمدة على المزامنة قد تسبب
  I/O أبطأ وتعارضات قفل/مزامنة.
- **دليل حالة Linux على SD أو eMMC**: يحذّر عندما تُحل الحالة إلى مصدر ربط `mmcblk*`،
  لأن I/O العشوائي المدعوم بـ SD أو eMMC يمكن أن يكون أبطأ ويهترئ
  أسرع مع كتابات الجلسات وبيانات الاعتماد.
- **أدلة الجلسات مفقودة**: يتطلب كل من `sessions/` ودليل مخزن الجلسات
  للاحتفاظ بالسجل وتجنب أعطال `ENOENT`.
- **عدم تطابق النصوص**: يحذّر عندما تحتوي إدخالات الجلسات الحديثة على ملفات
  نصوص مفقودة.
- **الجلسة الرئيسية “JSONL من سطر واحد”**: يشير إلى أن النص الرئيسي يحوي سطرًا واحدًا فقط
  (أي أن السجل لا يتراكم).
- **أدلة حالة متعددة**: يحذّر عندما توجد عدة مجلدات `~/.openclaw` عبر
  أدلة home أو عندما يشير `OPENCLAW_STATE_DIR` إلى مكان آخر (قد
  ينقسم السجل بين التثبيتات).
- **تذكير الوضع البعيد**: إذا كان `gateway.mode=remote`، يذكّرك doctor
  بتشغيله على المضيف البعيد (فالحالة تعيش هناك).
- **أذونات ملف الإعداد**: يحذّر إذا كان `~/.openclaw/openclaw.json`
  قابلًا للقراءة من المجموعة/العالم ويعرض تضييقها إلى `600`.

### 5) سلامة مصادقة النموذج (انتهاء OAuth)

يفحص doctor ملفات OAuth profiles في مخزن المصادقة، ويحذر عندما تكون الرموز
قريبة من الانتهاء/منتهية، ويمكنه تحديثها عندما يكون ذلك آمنًا. وإذا كانت Anthropic
OAuth/token profile قديمة، فإنه يقترح الترحيل إلى Claude CLI أو
Anthropic API key.
ولا تظهر مطالبات التحديث إلا عند التشغيل التفاعلي (TTY)؛ أما `--non-interactive`
فيتجاوز محاولات التحديث.

كما يبلغ doctor عن auth profiles غير القابلة للاستخدام مؤقتًا بسبب:

- تهدئات قصيرة (rate limits/timeouts/auth failures)
- تعطيلات أطول (billing/credit failures)

### 6) التحقق من نموذج hooks

إذا كان `hooks.gmail.model` مضبوطًا، فإن doctor يتحقق من مرجع النموذج مقابل
الفهرس وقائمة السماح ويحذر عندما لا يمكن حله أو يكون غير مسموح به.

### 7) إصلاح صورة sandbox

عند تفعيل sandboxing، يفحص doctor صور Docker ويعرض بناءها أو
التحول إلى الأسماء القديمة إذا كانت الصورة الحالية مفقودة.

### 7b) تبعيات وقت التشغيل للمكونات الإضافية المضمّنة

يتحقق doctor من أن تبعيات وقت التشغيل الخاصة بالمكونات الإضافية المضمّنة (مثل
حزم وقت تشغيل مكون Discord الإضافي) موجودة في جذر تثبيت OpenClaw.
وإذا كانت أي منها مفقودة، يبلغ doctor عن الحزم ويثبتها في وضع
`openclaw doctor --fix` / `openclaw doctor --repair`.

### 8) ترحيلات خدمات Gateway وتلميحات التنظيف

يكتشف doctor خدمات gateway القديمة (launchd/systemd/schtasks)
ويعرض إزالتها وتثبيت خدمة OpenClaw باستخدام منفذ gateway الحالي.
كما يمكنه فحص الخدمات الإضافية الشبيهة بـ gateway وطباعة تلميحات تنظيف.
وتُعد خدمات OpenClaw gateway المسماة حسب profile خدمات أساسية وليست
موسومة على أنها "إضافية".

### 8b) ترحيل Matrix عند البدء

عندما يكون لدى حساب قناة Matrix ترحيل حالة قديم معلّق أو قابل للتنفيذ،
يقوم doctor (في وضع `--fix` / `--repair`) بإنشاء لقطة ما قبل الترحيل ثم
يشغّل خطوات الترحيل على أساس أفضل جهد: ترحيل حالة Matrix القديمة وتحضير
الحالة المشفرة القديمة. وكلتا الخطوتين غير قاتلتين؛ تُسجل الأخطاء
ويستمر البدء. أما في وضع القراءة فقط (`openclaw doctor` من دون `--fix`) فيُتخطى هذا الفحص
بالكامل.

### 9) التحذيرات الأمنية

يصدر doctor تحذيرات عندما يكون مزود ما مفتوحًا على الرسائل المباشرة من دون قائمة سماح،
أو عندما تكون سياسة ما مضبوطة بطريقة خطرة.

### 10) systemd linger ‏(Linux)

إذا كان يعمل كخدمة مستخدم systemd، فإن doctor يضمن تفعيل lingering حتى يبقى
gateway حيًا بعد تسجيل الخروج.

### 11) حالة مساحة العمل (Skills، والمكونات الإضافية، والأدلة القديمة)

يطبع doctor ملخصًا لحالة مساحة العمل للوكيل الافتراضي:

- **حالة Skills**: عدد Skills القابلة للاستخدام، والتي ينقصها متطلبات، والمحجوبة بقائمة السماح.
- **أدلة مساحة العمل القديمة**: يحذر عند وجود `~/openclaw` أو أدلة مساحة عمل قديمة أخرى
  إلى جانب مساحة العمل الحالية.
- **حالة المكونات الإضافية**: عدد المكونات الإضافية المحمّلة/المعطلة/المخطئة؛ ويسرد معرّفات المكونات الإضافية لأي
  أخطاء؛ ويبلغ عن إمكانات bundle plugin.
- **تحذيرات توافق المكونات الإضافية**: يشير إلى المكونات الإضافية التي تعاني من مشكلات توافق مع
  وقت التشغيل الحالي.
- **تشخيصات المكونات الإضافية**: يعرض أي تحذيرات أو أخطاء وقت التحميل الصادرة عن
  سجل المكونات الإضافية.

### 11b) حجم ملفات bootstrap

يفحص doctor ما إذا كانت ملفات bootstrap الخاصة بمساحة العمل (مثل `AGENTS.md`,
أو `CLAUDE.md`، أو ملفات السياق المحقونة الأخرى) قريبة من حد
عدد الأحرف المكوَّن أو تتجاوزه. ويبلغ لكل ملف عن عدد الأحرف الخام مقابل المحقون، ونسبة
القص، وسبب القص (`max/file` أو `max/total`)، وإجمالي الأحرف
المحقونة كنسبة من الميزانية الإجمالية. وعندما تُقص الملفات أو تقترب من
الحد، يطبع doctor نصائح لضبط `agents.defaults.bootstrapMaxChars`
و`agents.defaults.bootstrapTotalMaxChars`.

### 11c) shell completion

يفحص doctor ما إذا كان إكمال tab مثبتًا للـ shell الحالية
(zsh, bash, fish, أو PowerShell):

- إذا كان ملف shell profile يستخدم نمط completion ديناميكيًا بطيئًا
  (`source <(openclaw completion ...)`)، فإن doctor يرقّيه إلى
  متغير الملف المخبأ الأسرع.
- إذا كان completion مكوّنًا في profile لكن ملف cache مفقود،
  يعيد doctor إنشاء cache تلقائيًا.
- إذا لم يكن completion مكوّنًا إطلاقًا، يطلب doctor تثبيته
  (في الوضع التفاعلي فقط؛ ويُتخطى مع `--non-interactive`).

شغّل `openclaw completion --write-state` لإعادة إنشاء cache يدويًا.

### 12) فحوصات مصادقة Gateway ‏(token المحلي)

يفحص doctor جاهزية مصادقة token المحلية لـ gateway.

- إذا كان وضع token يحتاج إلى token ولم يوجد مصدر token، يعرض doctor توليد واحد.
- إذا كانت `gateway.auth.token` مُدارة عبر SecretRef ولكنها غير متاحة، يحذّر doctor ولا يستبدلها بنص صريح.
- يفرض `openclaw doctor --generate-gateway-token` التوليد فقط عندما لا يكون هناك token SecretRef مكوّن.

### 12b) إصلاحات للقراءة فقط ومدركة لـ SecretRef

تحتاج بعض تدفقات الإصلاح إلى فحص بيانات الاعتماد المكوّنة من دون إضعاف
سلوك الفشل السريع في وقت التشغيل.

- يستخدم `openclaw doctor --fix` الآن نموذج ملخص SecretRef للقراءة فقط نفسه المستخدم في أوامر عائلة الحالة لإصلاحات الإعداد المستهدفة.
- مثال: يحاول إصلاح `allowFrom` / `groupAllowFrom` لـ Telegram من نوع `@username` استخدام بيانات اعتماد البوت المكوّنة عندما تكون متاحة.
- إذا كان Telegram bot token مكوّنًا عبر SecretRef لكنه غير متاح في مسار الأمر الحالي، يبلغ doctor أن بيانات الاعتماد مكوّنة-لكن-غير-متاحة ويتخطى الحل التلقائي بدلًا من الانهيار أو الإبلاغ خطأً عن أن token مفقود.

### 13) فحص سلامة Gateway + إعادة التشغيل

يشغّل doctor فحص سلامة ويعرض إعادة تشغيل gateway عندما يبدو
غير سليم.

### 13b) جاهزية بحث الذاكرة

يفحص doctor ما إذا كان موفر embedding المكوَّن لبحث الذاكرة جاهزًا
للوكيل الافتراضي. ويعتمد السلوك على backend والمزوّد المكوّنين:

- **QMD backend**: يفحص ما إذا كان الثنائي `qmd` متاحًا وقابلًا للبدء.
  وإذا لم يكن كذلك، يطبع إرشادات إصلاح تتضمن حزمة npm وخيار مسار ثنائي يدوي.
- **مزوّد محلي صريح**: يتحقق من وجود ملف نموذج محلي أو URL نموذج معروف
  بعيد/قابل للتنزيل. وإذا كان مفقودًا، يقترح التحول إلى مزود بعيد.
- **مزوّد بعيد صريح** ‏(`openai`, `voyage`, إلخ): يتحقق من وجود API key
  في البيئة أو مخزن المصادقة. ويطبع تلميحات إصلاح عملية إذا كانت مفقودة.
- **مزوّد auto**: يفحص توافر النموذج المحلي أولًا، ثم يحاول كل مزود بعيد
  وفق ترتيب الاختيار التلقائي.

عندما تكون نتيجة فحص gateway متاحة (كان gateway سليمًا وقت
الفحص)، يقارن doctor نتيجتها مع الإعداد المرئي عبر CLI ويشير
إلى أي عدم تطابق.

استخدم `openclaw memory status --deep` للتحقق من جاهزية embedding في وقت التشغيل.

### 14) تحذيرات حالة القنوات

إذا كان gateway سليمًا، يشغّل doctor فحص حالة القنوات ويبلغ
عن التحذيرات مع الإصلاحات المقترحة.

### 15) تدقيق إعداد supervisor + الإصلاح

يفحص doctor إعداد supervisor المثبّت (launchd/systemd/schtasks) بحثًا عن
القيم الافتراضية المفقودة أو القديمة (مثل تبعيات systemd الخاصة بـ network-online و
تأخير إعادة التشغيل). وعندما يجد عدم تطابق، يوصي بتحديث ويمكنه
إعادة كتابة ملف الخدمة/المهمة إلى القيم الافتراضية الحالية.

ملاحظات:

- يطلب `openclaw doctor` التأكيد قبل إعادة كتابة إعداد supervisor.
- يقبل `openclaw doctor --yes` مطالبات الإصلاح الافتراضية.
- يطبّق `openclaw doctor --repair` الإصلاحات الموصى بها من دون مطالبات.
- يستبدل `openclaw doctor --repair --force` إعدادات supervisor المخصصة.
- إذا كانت مصادقة token تتطلب token وكانت `gateway.auth.token` مُدارة عبر SecretRef، فإن تثبيت/إصلاح خدمة doctor يتحقق من SecretRef لكنه لا يحفظ قيم token النصية المحلولة داخل بيانات البيئة الوصفية لخدمة supervisor.
- إذا كانت مصادقة token تتطلب token وكان token SecretRef المكوَّن غير قابل للحل، يمنع doctor مسار التثبيت/الإصلاح مع إرشادات عملية.
- إذا كان كل من `gateway.auth.token` و`gateway.auth.password` مكوّنين وكان `gateway.auth.mode` غير مضبوط، يمنع doctor التثبيت/الإصلاح حتى يُضبط الوضع صراحةً.
- بالنسبة إلى وحدات Linux user-systemd، تتضمن فحوصات انجراف token في doctor الآن كلاً من مصدري `Environment=` و`EnvironmentFile=` عند مقارنة بيانات مصادقة الخدمة الوصفية.
- يمكنك دائمًا فرض إعادة كتابة كاملة عبر `openclaw gateway install --force`.

### 16) تشخيصات وقت تشغيل Gateway + المنفذ

يفحص doctor وقت تشغيل الخدمة (PID، وآخر حالة خروج) ويحذر عندما تكون
الخدمة مثبتة لكنها لا تعمل فعليًا. كما يفحص تعارضات المنافذ
على منفذ gateway ‏(الافتراضي `18789`) ويبلغ عن الأسباب المرجحة (gateway يعمل بالفعل،
أو SSH tunnel).

### 17) أفضل الممارسات لوقت تشغيل Gateway

يحذر doctor عندما تعمل خدمة gateway على Bun أو على مسار Node مُدار عبر مدير إصدارات
(`nvm`, `fnm`, `volta`, `asdf`, إلخ). فقنوات WhatsApp + Telegram تتطلب Node،
ويمكن لمسارات مديري الإصدارات أن تنكسر بعد الترقيات لأن الخدمة لا
تحمّل تهيئة shell الخاصة بك. ويعرض doctor الترحيل إلى تثبيت Node نظامي عندما
يكون متاحًا (Homebrew/apt/choco).

### 18) كتابة الإعداد + بيانات المعالج الوصفية

يحفظ doctor أي تغييرات في الإعداد ويضع ختمًا على بيانات المعالج الوصفية لتسجيل
تشغيل doctor.

### 19) نصائح مساحة العمل (النسخ الاحتياطي + نظام الذاكرة)

يقترح doctor نظام ذاكرة لمساحة العمل عند غيابه ويطبع تلميحًا للنسخ الاحتياطي
إذا لم تكن مساحة العمل موجودة أصلًا تحت git.

راجع [/concepts/agent-workspace](/concepts/agent-workspace) للحصول على دليل كامل
لبنية مساحة العمل والنسخ الاحتياطي عبر git ‏(يوصى باستخدام GitHub أو GitLab خاص).
