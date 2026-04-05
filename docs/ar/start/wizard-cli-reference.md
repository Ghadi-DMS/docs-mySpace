---
read_when:
    - عندما تحتاج إلى السلوك التفصيلي للأمر openclaw onboard
    - عندما تكون بصدد تصحيح نتائج الإعداد أو دمج عملاء الإعداد
sidebarTitle: CLI reference
summary: المرجع الكامل لتدفق إعداد CLI، وإعداد المصادقة/النموذج، والمخرجات، والتفاصيل الداخلية
title: مرجع إعداد CLI
x-i18n:
    generated_at: "2026-04-05T12:57:37Z"
    model: gpt-5.4
    provider: openai
    source_hash: 9ec4e685e3237e450d11c45826c2bb34b82c0bba1162335f8fbb07f51ba00a70
    source_path: start/wizard-cli-reference.md
    workflow: 15
---

# مرجع إعداد CLI

هذه الصفحة هي المرجع الكامل للأمر `openclaw onboard`.
للاطلاع على الدليل المختصر، راجع [الإعداد (CLI)](/ar/start/wizard).

## ما الذي يفعله المعالج

يرشدك الوضع المحلي (الافتراضي) خلال ما يلي:

- إعداد النموذج والمصادقة (OAuth لاشتراك OpenAI Code، وAnthropic Claude CLI أو مفتاح API، بالإضافة إلى خيارات MiniMax وGLM وOllama وMoonshot وStepFun وAI Gateway)
- موقع مساحة العمل وملفات التمهيد
- إعدادات Gateway ‏(المنفذ، والربط، والمصادقة، وTailscale)
- القنوات والموفرون (Telegram، وWhatsApp، وDiscord، وGoogle Chat، وMattermost، وSignal، وBlueBubbles، وغيرها من plugins القنوات المضمّنة)
- تثبيت daemon ‏(LaunchAgent، أو وحدة systemd للمستخدم، أو Scheduled Task أصلي في Windows مع بديل مجلد Startup)
- فحص الصحة
- إعداد Skills

يهيئ الوضع البعيد هذا الجهاز للاتصال بـ gateway موجود في مكان آخر.
ولا يثبت أو يعدل أي شيء على المضيف البعيد.

## تفاصيل التدفق المحلي

<Steps>
  <Step title="اكتشاف التكوين الحالي">
    - إذا كان `~/.openclaw/openclaw.json` موجودًا، فاختر Keep أو Modify أو Reset.
    - لا تؤدي إعادة تشغيل المعالج إلى محو أي شيء ما لم تختر Reset صراحةً (أو تمرر `--reset`).
    - يستخدم `--reset` في CLI افتراضيًا النطاق `config+creds+sessions`؛ استخدم `--reset-scope full` لإزالة مساحة العمل أيضًا.
    - إذا كان التكوين غير صالح أو يحتوي على مفاتيح قديمة، فسيتوقف المعالج ويطلب منك تشغيل `openclaw doctor` قبل المتابعة.
    - تستخدم إعادة التعيين `trash` وتوفر النطاقات التالية:
      - التكوين فقط
      - التكوين + بيانات الاعتماد + الجلسات
      - إعادة تعيين كاملة (تزيل مساحة العمل أيضًا)
  </Step>
  <Step title="النموذج والمصادقة">
    - توجد مصفوفة الخيارات الكاملة في [خيارات المصادقة والنموذج](#auth-and-model-options).
  </Step>
  <Step title="مساحة العمل">
    - الافتراضي هو `~/.openclaw/workspace` (قابل للتكوين).
    - يملأ ملفات مساحة العمل اللازمة لطقس التمهيد عند التشغيل الأول.
    - تخطيط مساحة العمل: [مساحة عمل الوكيل](/ar/concepts/agent-workspace).
  </Step>
  <Step title="Gateway">
    - يطلب المنفذ، والربط، ووضع المصادقة، وتعريض Tailscale.
    - الموصى به: إبقاء مصادقة الرمز المميز مفعلة حتى مع loopback حتى تضطر عملاء WS المحليون إلى المصادقة.
    - في وضع الرمز المميز، يوفر الإعداد التفاعلي:
      - **توليد/تخزين رمز مميز بنص واضح** (الافتراضي)
      - **استخدام SecretRef** (اختياري)
    - في وضع كلمة المرور، يدعم الإعداد التفاعلي أيضًا التخزين بنص واضح أو عبر SecretRef.
    - مسار SecretRef للرمز المميز في الوضع غير التفاعلي: `--gateway-token-ref-env <ENV_VAR>`.
      - يتطلب متغير بيئة غير فارغ في بيئة عملية الإعداد.
      - لا يمكن دمجه مع `--gateway-token`.
    - عطّل المصادقة فقط إذا كنت تثق تمامًا بكل عملية محلية.
    - تتطلب عمليات الربط غير loopback المصادقة أيضًا.
  </Step>
  <Step title="القنوات">
    - [WhatsApp](/ar/channels/whatsapp): تسجيل دخول QR اختياري
    - [Telegram](/ar/channels/telegram): رمز مميز للبوت
    - [Discord](/ar/channels/discord): رمز مميز للبوت
    - [Google Chat](/ar/channels/googlechat): JSON لحساب خدمة + جمهور webhook
    - [Mattermost](/ar/channels/mattermost): رمز مميز للبوت + عنوان URL الأساسي
    - [Signal](/ar/channels/signal): تثبيت `signal-cli` اختياري + إعداد الحساب
    - [BlueBubbles](/ar/channels/bluebubbles): موصى به لـ iMessage؛ عنوان URL للخادم + كلمة مرور + webhook
    - [iMessage](/ar/channels/imessage): مسار CLI القديم `imsg` + الوصول إلى قاعدة البيانات
    - أمان الرسائل الخاصة: الافتراضي هو الاقتران. ترسل أول رسالة خاصة رمزًا؛ وافق عبر
      `openclaw pairing approve <channel> <code>` أو استخدم قوائم السماح.
  </Step>
  <Step title="تثبيت daemon">
    - macOS: ‏LaunchAgent
      - يتطلب جلسة مستخدم مسجل دخوله؛ أما البيئات دون واجهة رسومية فاستخدم LaunchDaemon مخصصًا (غير مشحون).
    - Linux وWindows عبر WSL2: وحدة systemd للمستخدم
      - يحاول المعالج تنفيذ `loginctl enable-linger <user>` حتى يظل gateway يعمل بعد تسجيل الخروج.
      - قد يطلب sudo (يكتب إلى `/var/lib/systemd/linger`)؛ ويحاول أولًا من دون sudo.
    - Windows الأصلي: Scheduled Task أولًا
      - إذا رُفض إنشاء المهمة، يعود OpenClaw إلى عنصر تسجيل دخول لكل مستخدم داخل مجلد Startup ويبدأ gateway فورًا.
      - تظل Scheduled Tasks مفضلة لأنها توفر حالة إشراف أفضل.
    - اختيار runtime: ‏Node (موصى به؛ مطلوب لـ WhatsApp وTelegram). لا يُنصح باستخدام Bun.
  </Step>
  <Step title="فحص الصحة">
    - يبدأ gateway (عند الحاجة) ويشغّل `openclaw health`.
    - يضيف `openclaw status --deep` فحص صحة gateway الحي إلى مخرجات الحالة، بما في ذلك فحوصات القنوات عندما تكون مدعومة.
  </Step>
  <Step title="Skills">
    - يقرأ Skills المتاحة ويتحقق من المتطلبات.
    - يتيح لك اختيار مدير node: ‏npm أو pnpm أو bun.
    - يثبت التبعيات الاختيارية (يستخدم بعضها Homebrew على macOS).
  </Step>
  <Step title="الإنهاء">
    - ملخص والخطوات التالية، بما في ذلك خيارات تطبيقات iOS وAndroid وmacOS.
  </Step>
</Steps>

<Note>
إذا لم يتم اكتشاف واجهة رسومية، فسيطبع المعالج تعليمات إعادة توجيه منفذ SSH لواجهة Control UI بدلًا من فتح متصفح.
إذا كانت أصول Control UI مفقودة، فسيحاول المعالج بناءها؛ والحل البديل هو `pnpm ui:build` (مع التثبيت التلقائي لتبعيات واجهة المستخدم).
</Note>

## تفاصيل الوضع البعيد

يهيئ الوضع البعيد هذا الجهاز للاتصال بـ gateway موجود في مكان آخر.

<Info>
لا يثبت الوضع البعيد أي شيء على المضيف البعيد ولا يعدله.
</Info>

ما الذي تضبطه:

- عنوان URL لـ gateway البعيد (`ws://...`)
- الرمز المميز إذا كانت مصادقة gateway البعيد مطلوبة (موصى به)

<Note>
- إذا كان gateway مقصورًا على loopback فقط، فاستخدم نفق SSH أو tailnet.
- تلميحات الاكتشاف:
  - macOS: ‏Bonjour ‏(`dns-sd`)
  - Linux: ‏Avahi ‏(`avahi-browse`)
</Note>

## خيارات المصادقة والنموذج

<AccordionGroup>
  <Accordion title="مفتاح Anthropic API">
    يستخدم `ANTHROPIC_API_KEY` إذا كان موجودًا أو يطلب مفتاحًا، ثم يحفظه لاستخدام daemon.
  </Accordion>
  <Accordion title="Anthropic Claude CLI">
    يعيد استخدام تسجيل دخول Claude CLI محلي على مضيف gateway ويحوّل اختيار
    النموذج إلى مرجع قياسي `claude-cli/claude-*`.

    هذا مسار محلي احتياطي متاح في `openclaw onboard` و
    `openclaw configure`. وللاستخدام الإنتاجي، يفضَّل استخدام مفتاح Anthropic API.

    - macOS: يتحقق من عنصر Keychain المسمى "Claude Code-credentials"
    - Linux وWindows: يعيد استخدام `~/.claude/.credentials.json` إذا كان موجودًا

    على macOS، اختر "Always Allow" حتى لا تمنع بدايات launchd.

  </Accordion>
  <Accordion title="اشتراك OpenAI Code (إعادة استخدام Codex CLI)">
    إذا كان `~/.codex/auth.json` موجودًا، يمكن للمعالج إعادة استخدامه.
    تظل بيانات اعتماد Codex CLI المعاد استخدامها مُدارة بواسطة Codex CLI؛ وعند انتهاء صلاحيتها يعيد OpenClaw
    قراءة هذا المصدر أولًا، وعندما يستطيع الموفر تحديثها يكتب
    بيانات الاعتماد المحدّثة مرة أخرى إلى تخزين Codex بدلًا من تولي إدارتها
    بنفسه.
  </Accordion>
  <Accordion title="اشتراك OpenAI Code (OAuth)">
    تدفق المتصفح؛ الصق `code#state`.

    يضبط `agents.defaults.model` على `openai-codex/gpt-5.4` عندما يكون النموذج غير مضبوط أو `openai/*`.

  </Accordion>
  <Accordion title="مفتاح OpenAI API">
    يستخدم `OPENAI_API_KEY` إذا كان موجودًا أو يطلب مفتاحًا، ثم يخزن بيانات الاعتماد في ملفات تعريف المصادقة.

    يضبط `agents.defaults.model` على `openai/gpt-5.4` عندما يكون النموذج غير مضبوط، أو `openai/*`، أو `openai-codex/*`.

  </Accordion>
  <Accordion title="مفتاح xAI (Grok) API">
    يطلب `XAI_API_KEY` ويهيئ xAI كموفر نموذج.
  </Accordion>
  <Accordion title="OpenCode">
    يطلب `OPENCODE_API_KEY` (أو `OPENCODE_ZEN_API_KEY`) ويتيح لك اختيار فهرس Zen أو Go.
    عنوان URL للإعداد: [opencode.ai/auth](https://opencode.ai/auth).
  </Accordion>
  <Accordion title="مفتاح API (عام)">
    يخزن المفتاح نيابةً عنك.
  </Accordion>
  <Accordion title="Vercel AI Gateway">
    يطلب `AI_GATEWAY_API_KEY`.
    مزيد من التفاصيل: [Vercel AI Gateway](/ar/providers/vercel-ai-gateway).
  </Accordion>
  <Accordion title="Cloudflare AI Gateway">
    يطلب معرّف الحساب، ومعرّف gateway، و`CLOUDFLARE_AI_GATEWAY_API_KEY`.
    مزيد من التفاصيل: [Cloudflare AI Gateway](/ar/providers/cloudflare-ai-gateway).
  </Accordion>
  <Accordion title="MiniMax">
    يُكتب التكوين تلقائيًا. والافتراضي المستضاف هو `MiniMax-M2.7`؛ ويستخدم إعداد مفتاح API
    ‏`minimax/...`، ويستخدم إعداد OAuth ‏`minimax-portal/...`.
    مزيد من التفاصيل: [MiniMax](/ar/providers/minimax).
  </Accordion>
  <Accordion title="StepFun">
    يُكتب التكوين تلقائيًا لـ StepFun standard أو Step Plan على نقاط نهاية الصين أو العالمية.
    يتضمن Standard حاليًا `step-3.5-flash`، كما يتضمن Step Plan أيضًا `step-3.5-flash-2603`.
    مزيد من التفاصيل: [StepFun](/ar/providers/stepfun).
  </Accordion>
  <Accordion title="Synthetic (متوافق مع Anthropic)">
    يطلب `SYNTHETIC_API_KEY`.
    مزيد من التفاصيل: [Synthetic](/ar/providers/synthetic).
  </Accordion>
  <Accordion title="Ollama (نماذج مفتوحة سحابية ومحلية)">
    يطلب عنوان URL الأساسي (الافتراضي `http://127.0.0.1:11434`)، ثم يوفر وضع Cloud + Local أو Local.
    ويكتشف النماذج المتاحة ويقترح افتراضيات.
    مزيد من التفاصيل: [Ollama](/ar/providers/ollama).
  </Accordion>
  <Accordion title="Moonshot وKimi Coding">
    تُكتب إعدادات Moonshot ‏(Kimi K2) وKimi Coding تلقائيًا.
    مزيد من التفاصيل: [Moonshot AI (Kimi + Kimi Coding)](/ar/providers/moonshot).
  </Accordion>
  <Accordion title="موفر مخصص">
    يعمل مع نقاط النهاية المتوافقة مع OpenAI والمتوافقة مع Anthropic.

    يدعم الإعداد التفاعلي خيارات تخزين مفتاح API نفسها مثل تدفقات مفاتيح API لموفرين آخرين:
    - **الصق مفتاح API الآن** (نص واضح)
    - **استخدم مرجع سر** (مرجع env أو مرجع موفر مهيأ، مع تحقق تمهيدي مسبق)

    أعلام الوضع غير التفاعلي:
    - `--auth-choice custom-api-key`
    - `--custom-base-url`
    - `--custom-model-id`
    - `--custom-api-key` (اختياري؛ يعود إلى `CUSTOM_API_KEY`)
    - `--custom-provider-id` (اختياري)
    - `--custom-compatibility <openai|anthropic>` (اختياري؛ الافتراضي `openai`)

  </Accordion>
  <Accordion title="تخطي">
    يترك المصادقة غير مهيأة.
  </Accordion>
</AccordionGroup>

سلوك النموذج:

- اختر النموذج الافتراضي من الخيارات المكتشفة، أو أدخل الموفر والنموذج يدويًا.
- عندما يبدأ الإعداد من خيار مصادقة موفر، فإن منتقي النموذج يفضّل
  ذلك الموفر تلقائيًا. بالنسبة إلى Volcengine وBytePlus، يطابق هذا التفضيل
  أيضًا متغيرات coding-plan الخاصة بهما (`volcengine-plan/*`,
  `byteplus-plan/*`).
- إذا كان مرشح الموفر المفضّل سيصبح فارغًا، يعود المنتقي إلى
  الفهرس الكامل بدلًا من عرض عدم وجود نماذج.
- يشغّل المعالج فحصًا للنموذج ويعرض تحذيرًا إذا كان النموذج المهيأ غير معروف أو تنقصه المصادقة.

مسارات بيانات الاعتماد وملفات التعريف:

- ملفات تعريف المصادقة (مفاتيح API وOAuth): ‏`~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
- استيراد OAuth القديم: ‏`~/.openclaw/credentials/oauth.json`

وضع تخزين بيانات الاعتماد:

- السلوك الافتراضي أثناء الإعداد هو حفظ مفاتيح API كقيم بنص واضح في ملفات تعريف المصادقة.
- يفعّل `--secret-input-mode ref` وضع المرجع بدلًا من تخزين المفاتيح بنص واضح.
  في الإعداد التفاعلي، يمكنك اختيار أحد الخيارين:
  - مرجع متغير بيئة (مثل `keyRef: { source: "env", provider: "default", id: "OPENAI_API_KEY" }`)
  - مرجع موفر مهيأ (`file` أو `exec`) مع الاسم المستعار للموفر + المعرّف
- يشغّل وضع المرجع التفاعلي تحققًا تمهيديًا سريعًا قبل الحفظ.
  - مراجع env: تتحقق من اسم المتغير ومن وجود قيمة غير فارغة في بيئة الإعداد الحالية.
  - مراجع الموفر: تتحقق من تكوين الموفر وتحل المعرّف المطلوب.
  - إذا فشل التحقق التمهيدي، يعرض الإعداد الخطأ ويتيح لك إعادة المحاولة.
- في الوضع غير التفاعلي، يكون `--secret-input-mode ref` معتمدًا على env فقط.
  - اضبط متغير بيئة الموفّر في بيئة عملية الإعداد.
  - تتطلب أعلام المفاتيح المضمنة (مثل `--openai-api-key`) ضبط متغير env؛ وإلا يفشل الإعداد سريعًا.
  - بالنسبة إلى الموفرين المخصصين، يخزن وضع `ref` غير التفاعلي `models.providers.<id>.apiKey` بوصفه `{ source: "env", provider: "default", id: "CUSTOM_API_KEY" }`.
  - في حالة الموفر المخصص تلك، يتطلب `--custom-api-key` ضبط `CUSTOM_API_KEY`؛ وإلا يفشل الإعداد سريعًا.
- تدعم بيانات اعتماد مصادقة Gateway خيارات النص الواضح وSecretRef في الإعداد التفاعلي:
  - وضع الرمز المميز: **توليد/تخزين رمز مميز بنص واضح** (الافتراضي) أو **استخدام SecretRef**.
  - وضع كلمة المرور: نص واضح أو SecretRef.
- مسار SecretRef للرمز المميز في الوضع غير التفاعلي: `--gateway-token-ref-env <ENV_VAR>`.
- تظل إعدادات النص الواضح الحالية تعمل كما هي دون تغيير.

<Note>
نصيحة للخوادم والبيئات دون واجهة: أكمل OAuth على جهاز يحتوي على متصفح، ثم انسخ
الملف `auth-profiles.json` لذلك الوكيل (على سبيل المثال
`~/.openclaw/agents/<agentId>/agent/auth-profiles.json`، أو المسار
المطابق ضمن `$OPENCLAW_STATE_DIR/...`) إلى مضيف gateway. الملف `credentials/oauth.json`
هو مجرد مصدر استيراد قديم.
</Note>

## المخرجات والتفاصيل الداخلية

الحقول المعتادة في `~/.openclaw/openclaw.json`:

- `agents.defaults.workspace`
- `agents.defaults.model` / `models.providers` (إذا تم اختيار Minimax)
- `tools.profile` (يضبط الإعداد المحلي هذا على `"coding"` افتراضيًا عندما لا تكون القيمة مضبوطة؛ وتُحفَظ القيم الصريحة الحالية)
- `gateway.*` (الوضع، والربط، والمصادقة، وTailscale)
- `session.dmScope` (يضبط الإعداد المحلي هذا افتراضيًا على `per-channel-peer` عندما لا تكون القيمة مضبوطة؛ وتُحفَظ القيم الصريحة الحالية)
- `channels.telegram.botToken`, `channels.discord.token`, `channels.matrix.*`, `channels.signal.*`, `channels.imessage.*`
- قوائم السماح للقنوات (Slack، وDiscord، وMatrix، وMicrosoft Teams) عندما تختار ذلك أثناء المطالبات (تُحل الأسماء إلى معرّفات عندما يكون ذلك ممكنًا)
- `skills.install.nodeManager`
  - يقبل العلَم `setup --node-manager` القيم `npm` أو `pnpm` أو `bun`.
  - يمكن للتكوين اليدوي لاحقًا أيضًا ضبط `skills.install.nodeManager: "yarn"`.
- `wizard.lastRunAt`
- `wizard.lastRunVersion`
- `wizard.lastRunCommit`
- `wizard.lastRunCommand`
- `wizard.lastRunMode`

يكتب `openclaw agents add` إلى `agents.list[]` و`bindings` الاختيارية.

توجد بيانات اعتماد WhatsApp تحت `~/.openclaw/credentials/whatsapp/<accountId>/`.
وتُخزن الجلسات تحت `~/.openclaw/agents/<agentId>/sessions/`.

<Note>
تُسلَّم بعض القنوات على هيئة plugins. وعند اختيارها أثناء الإعداد، يطالبك المعالج
بتثبيت plugin ‏(npm أو مسار محلي) قبل تهيئة القناة.
</Note>

‏RPC لمعالج Gateway:

- `wizard.start`
- `wizard.next`
- `wizard.cancel`
- `wizard.status`

يمكن للعملاء (تطبيق macOS وControl UI) عرض الخطوات دون إعادة تنفيذ منطق الإعداد.

سلوك إعداد Signal:

- ينزّل أصل الإصدار المناسب
- يخزنه تحت `~/.openclaw/tools/signal-cli/<version>/`
- يكتب `channels.signal.cliPath` في التكوين
- تتطلب إصدارات JVM ‏Java 21
- تُستخدم الإصدارات الأصلية عندما تكون متاحة
- يستخدم Windows ‏WSL2 ويتبع تدفق signal-cli في Linux داخل WSL

## الوثائق ذات الصلة

- مركز الإعداد: [الإعداد (CLI)](/ar/start/wizard)
- الأتمتة والسكربتات: [أتمتة CLI](/start/wizard-cli-automation)
- مرجع الأوامر: [`openclaw onboard`](/cli/onboard)
