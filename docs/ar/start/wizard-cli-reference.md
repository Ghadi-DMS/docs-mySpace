---
read_when:
    - تحتاج إلى سلوك مفصل للأمر openclaw onboard
    - أنت تستكشف نتائج الإعداد الأولي أو تدمج عملاء الإعداد الأولي
sidebarTitle: CLI reference
summary: مرجع كامل لتدفق إعداد CLI، وإعداد المصادقة/النموذج، والمخرجات، والبنية الداخلية
title: مرجع إعداد CLI
x-i18n:
    generated_at: "2026-04-06T03:13:17Z"
    model: gpt-5.4
    provider: openai
    source_hash: 92f379b34a2b48c68335dae4f759117c770f018ec51b275f4f40421c6b3abb23
    source_path: start/wizard-cli-reference.md
    workflow: 15
---

# مرجع إعداد CLI

هذه الصفحة هي المرجع الكامل للأمر `openclaw onboard`.
للدليل المختصر، راجع [الإعداد الأولي (CLI)](/ar/start/wizard).

## ما الذي يفعله المعالج

يقودك الوضع المحلي (الافتراضي) عبر ما يلي:

- إعداد النموذج والمصادقة (OpenAI Code subscription OAuth، وAnthropic Claude CLI أو مفتاح API، بالإضافة إلى خيارات MiniMax وGLM وOllama وMoonshot وStepFun وAI Gateway)
- موقع مساحة العمل وملفات التهيئة الأولية
- إعدادات البوابة (المنفذ، والربط، والمصادقة، وtailscale)
- القنوات والمزودون (Telegram وWhatsApp وDiscord وGoogle Chat وMattermost وSignal وBlueBubbles وإضافات القنوات المضمنة الأخرى)
- تثبيت الخدمة الخلفية (LaunchAgent، أو وحدة systemd للمستخدم، أو Scheduled Task أصلي في Windows مع fallback إلى مجلد Startup)
- فحص السلامة
- إعداد Skills

يقوم الوضع البعيد بإعداد هذا الجهاز للاتصال ببوابة موجودة في مكان آخر.
ولا يقوم بتثبيت أو تعديل أي شيء على المضيف البعيد.

## تفاصيل التدفق المحلي

<Steps>
  <Step title="اكتشاف الإعدادات الحالية">
    - إذا كان `~/.openclaw/openclaw.json` موجودًا، فاختر Keep أو Modify أو Reset.
    - لا تؤدي إعادة تشغيل المعالج إلى مسح أي شيء ما لم تختر Reset صراحةً (أو تمرر `--reset`).
    - يفترض `--reset` في CLI النطاق `config+creds+sessions`؛ استخدم `--reset-scope full` لإزالة مساحة العمل أيضًا.
    - إذا كانت الإعدادات غير صالحة أو تحتوي على مفاتيح قديمة، يتوقف المعالج ويطلب منك تشغيل `openclaw doctor` قبل المتابعة.
    - يستخدم Reset الأمر `trash` ويعرض النطاقات التالية:
      - الإعدادات فقط
      - الإعدادات + بيانات الاعتماد + الجلسات
      - إعادة تعيين كاملة (تزيل مساحة العمل أيضًا)
  </Step>
  <Step title="النموذج والمصادقة">
    - توجد مصفوفة الخيارات الكاملة في [خيارات المصادقة والنموذج](#auth-and-model-options).
  </Step>
  <Step title="مساحة العمل">
    - الافتراضي `~/.openclaw/workspace` (قابل للتخصيص).
    - يجهّز ملفات مساحة العمل اللازمة لطقوس التهيئة الأولية عند أول تشغيل.
    - تخطيط مساحة العمل: [مساحة عمل الوكيل](/ar/concepts/agent-workspace).
  </Step>
  <Step title="البوابة">
    - يطلب المنفذ، والربط، ووضع المصادقة، وتعريض tailscale.
    - الموصى به: الإبقاء على مصادقة الرمز المميز مفعّلة حتى مع loopback حتى تضطر عملاء WS المحليون إلى المصادقة.
    - في وضع الرمز المميز، يقدّم الإعداد التفاعلي:
      - **إنشاء/تخزين رمز مميز نصي صريح** (الافتراضي)
      - **استخدام SecretRef** (اختياري)
    - في وضع كلمة المرور، يدعم الإعداد التفاعلي أيضًا التخزين النصي الصريح أو SecretRef.
    - مسار SecretRef للرمز المميز في الوضع غير التفاعلي: `--gateway-token-ref-env <ENV_VAR>`.
      - يتطلب متغير env غير فارغ في بيئة عملية الإعداد الأولي.
      - لا يمكن دمجه مع `--gateway-token`.
    - عطّل المصادقة فقط إذا كنت تثق تمامًا بكل عملية محلية.
    - ما تزال عمليات الربط غير loopback تتطلب المصادقة.
  </Step>
  <Step title="القنوات">
    - [WhatsApp](/ar/channels/whatsapp): تسجيل دخول QR اختياري
    - [Telegram](/ar/channels/telegram): رمز بوت مميز
    - [Discord](/ar/channels/discord): رمز بوت مميز
    - [Google Chat](/ar/channels/googlechat): JSON لحساب خدمة + webhook audience
    - [Mattermost](/ar/channels/mattermost): رمز بوت مميز + base URL
    - [Signal](/ar/channels/signal): تثبيت `signal-cli` اختياري + إعداد الحساب
    - [BlueBubbles](/ar/channels/bluebubbles): موصى به لـ iMessage؛ عنوان URL للخادم + كلمة مرور + webhook
    - [iMessage](/ar/channels/imessage): مسار `imsg` CLI القديم + وصول إلى قاعدة البيانات
    - أمان الرسائل المباشرة: الافتراضي هو الإقران. ترسل أول رسالة مباشرة رمزًا؛ وافق عليه عبر
      `openclaw pairing approve <channel> <code>` أو استخدم قوائم السماح.
  </Step>
  <Step title="تثبيت الخدمة الخلفية">
    - macOS: ‏LaunchAgent
      - يتطلب جلسة مستخدم مسجّل الدخول؛ وللأوضاع عديمة الواجهة استخدم LaunchDaemon مخصصًا (غير مرفق).
    - Linux وWindows عبر WSL2: وحدة systemd للمستخدم
      - يحاول المعالج تنفيذ `loginctl enable-linger <user>` حتى تستمر البوابة في العمل بعد تسجيل الخروج.
      - قد يطلب sudo (يكتب إلى `/var/lib/systemd/linger`)؛ ويحاول أولًا من دون sudo.
    - Windows الأصلي: Scheduled Task أولًا
      - إذا رُفض إنشاء المهمة، يعود OpenClaw إلى عنصر تسجيل دخول لكل مستخدم في مجلد Startup ويبدأ البوابة فورًا.
      - تظل Scheduled Tasks مفضلة لأنها توفر حالة مشرف أفضل.
    - اختيار وقت التشغيل: Node ‏(موصى به؛ مطلوب لـ WhatsApp وTelegram). ولا يُنصح باستخدام Bun.
  </Step>
  <Step title="فحص السلامة">
    - يبدأ البوابة (إذا لزم الأمر) ويشغّل `openclaw health`.
    - يضيف `openclaw status --deep` فحص سلامة البوابة الحي إلى مخرجات الحالة، بما في ذلك فحوصات القنوات عند دعمها.
  </Step>
  <Step title="Skills">
    - يقرأ Skills المتاحة ويتحقق من المتطلبات.
    - يتيح لك اختيار node manager: ‏npm أو pnpm أو bun.
    - يثبت التبعيات الاختيارية (بعضها يستخدم Homebrew على macOS).
  </Step>
  <Step title="الإنهاء">
    - ملخص وخطوات تالية، بما في ذلك خيارات تطبيقات iOS وAndroid وmacOS.
  </Step>
</Steps>

<Note>
إذا لم يتم اكتشاف واجهة رسومية، يطبع المعالج تعليمات إعادة توجيه منفذ SSH لـ Control UI بدلًا من فتح متصفح.
إذا كانت أصول Control UI مفقودة، يحاول المعالج بناءها؛ ويكون fallback هو `pnpm ui:build` (مع تثبيت تلقائي لتبعيات UI).
</Note>

## تفاصيل الوضع البعيد

يقوم الوضع البعيد بإعداد هذا الجهاز للاتصال ببوابة موجودة في مكان آخر.

<Info>
الوضع البعيد لا يثبت أو يعدل أي شيء على المضيف البعيد.
</Info>

ما الذي تقوم بإعداده:

- عنوان URL للبوابة البعيدة (`ws://...`)
- الرمز المميز إذا كانت مصادقة البوابة البعيدة مطلوبة (موصى به)

<Note>
- إذا كانت البوابة مقيدة بـ loopback فقط، فاستخدم نفق SSH أو tailnet.
- تلميحات الاكتشاف:
  - macOS: ‏Bonjour (`dns-sd`)
  - Linux: ‏Avahi (`avahi-browse`)
</Note>

## خيارات المصادقة والنموذج

<AccordionGroup>
  <Accordion title="مفتاح API لـ Anthropic">
    يستخدم `ANTHROPIC_API_KEY` إذا كان موجودًا أو يطلب مفتاحًا، ثم يحفظه لاستخدام الخدمة الخلفية.
  </Accordion>
  <Accordion title="اشتراك OpenAI Code (إعادة استخدام Codex CLI)">
    إذا كان `~/.codex/auth.json` موجودًا، يمكن للمعالج إعادة استخدامه.
    تظل بيانات اعتماد Codex CLI المعاد استخدامها مُدارة بواسطة Codex CLI؛ وعند انتهاء الصلاحية يعيد OpenClaw
    قراءة ذلك المصدر أولًا، وعندما يستطيع المزود تحديثه، يكتب
    بيانات الاعتماد المحدّثة مرة أخرى إلى تخزين Codex بدلًا من الاستحواذ عليها
    بنفسه.
  </Accordion>
  <Accordion title="اشتراك OpenAI Code (OAuth)">
    تدفق متصفح؛ الصق `code#state`.

    يضبط `agents.defaults.model` على `openai-codex/gpt-5.4` عندما يكون النموذج غير مضبوط أو `openai/*`.

  </Accordion>
  <Accordion title="مفتاح API لـ OpenAI">
    يستخدم `OPENAI_API_KEY` إذا كان موجودًا أو يطلب مفتاحًا، ثم يخزن بيانات الاعتماد في ملفات تعريف المصادقة.

    يضبط `agents.defaults.model` على `openai/gpt-5.4` عندما يكون النموذج غير مضبوط أو `openai/*` أو `openai-codex/*`.

  </Accordion>
  <Accordion title="مفتاح API لـ xAI ‏(Grok)">
    يطلب `XAI_API_KEY` ويضبط xAI كمزود نموذج.
  </Accordion>
  <Accordion title="OpenCode">
    يطلب `OPENCODE_API_KEY` ‏(أو `OPENCODE_ZEN_API_KEY`) ويتيح لك اختيار فهرس Zen أو Go.
    عنوان URL للإعداد: [opencode.ai/auth](https://opencode.ai/auth).
  </Accordion>
  <Accordion title="مفتاح API (عام)">
    يخزن المفتاح لك.
  </Accordion>
  <Accordion title="Vercel AI Gateway">
    يطلب `AI_GATEWAY_API_KEY`.
    مزيد من التفاصيل: [Vercel AI Gateway](/ar/providers/vercel-ai-gateway).
  </Accordion>
  <Accordion title="Cloudflare AI Gateway">
    يطلب معرّف الحساب، ومعرّف البوابة، و`CLOUDFLARE_AI_GATEWAY_API_KEY`.
    مزيد من التفاصيل: [Cloudflare AI Gateway](/ar/providers/cloudflare-ai-gateway).
  </Accordion>
  <Accordion title="MiniMax">
    تُكتب الإعدادات تلقائيًا. الافتراضي المستضاف هو `MiniMax-M2.7`؛ ويستخدم إعداد مفتاح API
    `minimax/...`، بينما يستخدم إعداد OAuth ‏`minimax-portal/...`.
    مزيد من التفاصيل: [MiniMax](/ar/providers/minimax).
  </Accordion>
  <Accordion title="StepFun">
    تُكتب الإعدادات تلقائيًا لـ StepFun القياسي أو Step Plan على نقاط نهاية الصين أو النقاط العالمية.
    يتضمن Standard حاليًا `step-3.5-flash`، كما يتضمن Step Plan أيضًا `step-3.5-flash-2603`.
    مزيد من التفاصيل: [StepFun](/ar/providers/stepfun).
  </Accordion>
  <Accordion title="Synthetic (متوافق مع Anthropic)">
    يطلب `SYNTHETIC_API_KEY`.
    مزيد من التفاصيل: [Synthetic](/ar/providers/synthetic).
  </Accordion>
  <Accordion title="Ollama (نماذج سحابية ومحلية مفتوحة)">
    يطلب base URL ‏(الافتراضي `http://127.0.0.1:11434`)، ثم يقدّم وضع Cloud + Local أو Local.
    يكتشف النماذج المتاحة ويقترح القيم الافتراضية.
    مزيد من التفاصيل: [Ollama](/ar/providers/ollama).
  </Accordion>
  <Accordion title="Moonshot وKimi Coding">
    تُكتب إعدادات Moonshot ‏(Kimi K2) وKimi Coding تلقائيًا.
    مزيد من التفاصيل: [Moonshot AI (Kimi + Kimi Coding)](/ar/providers/moonshot).
  </Accordion>
  <Accordion title="مزود مخصص">
    يعمل مع نقاط النهاية المتوافقة مع OpenAI والمتوافقة مع Anthropic.

    يدعم الإعداد الأولي التفاعلي خيارات تخزين مفتاح API نفسها كما في تدفقات مفاتيح API الخاصة بالمزودين الآخرين:
    - **ألصق مفتاح API الآن** (نص صريح)
    - **استخدم مرجع سر** (مرجع env أو مرجع مزود مضبوط، مع تحقق تمهيدي مسبق)

    علامات الوضع غير التفاعلي:
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

- اختر النموذج الافتراضي من الخيارات المكتشفة، أو أدخل المزود والنموذج يدويًا.
- عندما يبدأ الإعداد الأولي من اختيار مصادقة مزود، فإن منتقي النموذج يفضّل
  ذلك المزود تلقائيًا. بالنسبة إلى Volcengine وBytePlus، يطابق هذا التفضيل نفسه
  أيضًا متغيرات coding-plan الخاصة بهما (`volcengine-plan/*`,
  `byteplus-plan/*`).
- إذا كان مرشح المزود المفضل هذا سيؤدي إلى قائمة فارغة، يعود المنتقي إلى
  الفهرس الكامل بدلًا من عرض عدم وجود نماذج.
- يشغّل المعالج فحصًا للنموذج ويحذّر إذا كان النموذج المضبوط غير معروف أو تفتقر مصادقته.

مسارات بيانات الاعتماد وملفات التعريف:

- ملفات تعريف المصادقة (مفاتيح API + OAuth): ‏`~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
- استيراد OAuth القديم: ‏`~/.openclaw/credentials/oauth.json`

وضع تخزين بيانات الاعتماد:

- السلوك الافتراضي للإعداد الأولي هو حفظ مفاتيح API كقيم نصية صريحة في ملفات تعريف المصادقة.
- يفعّل `--secret-input-mode ref` وضع المرجع بدلًا من التخزين النصي الصريح للمفتاح.
  في الإعداد التفاعلي، يمكنك اختيار أحد الخيارين:
  - مرجع متغير بيئة (على سبيل المثال `keyRef: { source: "env", provider: "default", id: "OPENAI_API_KEY" }`)
  - مرجع مزود مضبوط (`file` أو `exec`) مع اسم مزود مستعار + معرّف
- يشغّل وضع المرجع التفاعلي تحققًا تمهيديًا سريعًا قبل الحفظ.
  - مراجع Env: تتحقق من اسم المتغير + قيمة غير فارغة في بيئة الإعداد الأولي الحالية.
  - مراجع المزود: تتحقق من إعدادات المزود وتحل المعرّف المطلوب.
  - إذا فشل التحقق التمهيدي، يعرض الإعداد الأولي الخطأ ويتيح لك إعادة المحاولة.
- في الوضع غير التفاعلي، يكون `--secret-input-mode ref` مدعومًا بواسطة env فقط.
  - اضبط متغير env الخاص بالمزود في بيئة عملية الإعداد الأولي.
  - تتطلب علامات المفتاح المضمنة (مثل `--openai-api-key`) ضبط متغير env هذا؛ وإلا يفشل الإعداد الأولي سريعًا.
  - بالنسبة إلى المزودين المخصصين، يخزن وضع `ref` غير التفاعلي القيمة `models.providers.<id>.apiKey` بالشكل `{ source: "env", provider: "default", id: "CUSTOM_API_KEY" }`.
  - في حالة المزود المخصص هذه، يتطلب `--custom-api-key` ضبط `CUSTOM_API_KEY`؛ وإلا يفشل الإعداد الأولي سريعًا.
- تدعم بيانات اعتماد مصادقة البوابة خيارات النص الصريح وSecretRef في الإعداد التفاعلي:
  - وضع الرمز المميز: **إنشاء/تخزين رمز مميز نصي صريح** (الافتراضي) أو **استخدام SecretRef**.
  - وضع كلمة المرور: نص صريح أو SecretRef.
- مسار SecretRef للرمز المميز غير التفاعلي: `--gateway-token-ref-env <ENV_VAR>`.
- تستمر الإعدادات النصية الصريحة الحالية في العمل كما هي دون تغيير.

<Note>
نصيحة للخوادم والبيئات عديمة الواجهة: أكمل OAuth على جهاز يحتوي على متصفح، ثم انسخ
ملف `auth-profiles.json` الخاص بذلك الوكيل (على سبيل المثال
`~/.openclaw/agents/<agentId>/agent/auth-profiles.json`، أو المسار المطابق
`$OPENCLAW_STATE_DIR/...`) إلى gateway host. ويُعد `credentials/oauth.json`
مصدر استيراد قديمًا فقط.
</Note>

## المخرجات والبنية الداخلية

الحقول المعتادة في `~/.openclaw/openclaw.json`:

- `agents.defaults.workspace`
- `agents.defaults.model` / `models.providers` ‏(إذا تم اختيار Minimax)
- `tools.profile` ‏(يضبط الإعداد الأولي المحلي هذه القيمة افتراضيًا على `"coding"` عندما تكون غير مضبوطة؛ وتُحفظ القيم الصريحة الحالية)
- `gateway.*` ‏(الوضع، والربط، والمصادقة، وtailscale)
- `session.dmScope` ‏(يضبط الإعداد الأولي المحلي هذا افتراضيًا على `per-channel-peer` عندما يكون غير مضبوط؛ وتُحفظ القيم الصريحة الحالية)
- `channels.telegram.botToken`, `channels.discord.token`, `channels.matrix.*`, `channels.signal.*`, `channels.imessage.*`
- قوائم السماح للقنوات (Slack وDiscord وMatrix وMicrosoft Teams) عندما تشترك فيها أثناء المطالبات (تُحل الأسماء إلى معرّفات عندما يكون ذلك ممكنًا)
- `skills.install.nodeManager`
  - تقبل العلامة `setup --node-manager` القيم `npm` أو `pnpm` أو `bun`.
  - ما يزال الإعداد اليدوي قادرًا على ضبط `skills.install.nodeManager: "yarn"` لاحقًا.
- `wizard.lastRunAt`
- `wizard.lastRunVersion`
- `wizard.lastRunCommit`
- `wizard.lastRunCommand`
- `wizard.lastRunMode`

يكتب `openclaw agents add` إلى `agents.list[]` وإلى `bindings` الاختيارية.

توجد بيانات اعتماد WhatsApp تحت `~/.openclaw/credentials/whatsapp/<accountId>/`.
وتُخزَّن الجلسات تحت `~/.openclaw/agents/<agentId>/sessions/`.

<Note>
بعض القنوات تُسلَّم كإضافات. وعند اختيارها أثناء الإعداد، يطلب المعالج
تثبيت الإضافة (npm أو مسار محلي) قبل إعداد القناة.
</Note>

Gateway wizard RPC:

- `wizard.start`
- `wizard.next`
- `wizard.cancel`
- `wizard.status`

يمكن للعملاء (تطبيق macOS وControl UI) عرض الخطوات من دون إعادة تنفيذ منطق الإعداد الأولي.

سلوك إعداد Signal:

- ينزّل أصل الإصدار المناسب
- يخزنه تحت `~/.openclaw/tools/signal-cli/<version>/`
- يكتب `channels.signal.cliPath` في الإعدادات
- تتطلب إصدارات JVM استخدام Java 21
- تُستخدم الإصدارات الأصلية عندما تكون متاحة
- يستخدم Windows نظام WSL2 ويتبع تدفق Linux الخاص بـ signal-cli داخل WSL

## الوثائق ذات الصلة

- مركز الإعداد الأولي: [الإعداد الأولي (CLI)](/ar/start/wizard)
- الأتمتة والبرامج النصية: [أتمتة CLI](/ar/start/wizard-cli-automation)
- مرجع الأوامر: [`openclaw onboard`](/cli/onboard)
