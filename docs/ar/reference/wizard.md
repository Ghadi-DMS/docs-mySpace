---
read_when:
    - عند البحث عن خطوة إعداد أو عَلم معيّن
    - عند أتمتة الإعداد باستخدام الوضع غير التفاعلي
    - عند تصحيح سلوك الإعداد
sidebarTitle: Onboarding Reference
summary: 'المرجع الكامل لإعداد CLI: كل خطوة وعَلم وحقل تكوين'
title: مرجع الإعداد
x-i18n:
    generated_at: "2026-04-05T12:56:46Z"
    model: gpt-5.4
    provider: openai
    source_hash: ae6c76a31885c0678af2ac71254c5baf08f6de5481f85f6cfdf44d473946fdb8
    source_path: reference/wizard.md
    workflow: 15
---

# مرجع الإعداد

هذا هو المرجع الكامل للأمر `openclaw onboard`.
للحصول على نظرة عامة عالية المستوى، راجع [الإعداد (CLI)](/ar/start/wizard).

## تفاصيل التدفق (الوضع المحلي)

<Steps>
  <Step title="اكتشاف التكوين الحالي">
    - إذا كان `~/.openclaw/openclaw.json` موجودًا، فاختر **الاحتفاظ / التعديل / إعادة التعيين**.
    - إن إعادة تشغيل الإعداد لا تمحو أي شيء **إلا** إذا اخترت صراحةً **إعادة التعيين**
      (أو مررت `--reset`).
    - يستخدم `--reset` في CLI النطاق الافتراضي `config+creds+sessions`؛ استخدم `--reset-scope full`
      لإزالة مساحة العمل أيضًا.
    - إذا كان التكوين غير صالح أو يحتوي على مفاتيح قديمة، فسيتوقف المعالج ويطلب
      منك تشغيل `openclaw doctor` قبل المتابعة.
    - تستخدم إعادة التعيين `trash` (وليس `rm` مطلقًا) وتوفر النطاقات التالية:
      - التكوين فقط
      - التكوين + بيانات الاعتماد + الجلسات
      - إعادة تعيين كاملة (تزيل مساحة العمل أيضًا)
  </Step>
  <Step title="النموذج/المصادقة">
    - **مفتاح Anthropic API**: يستخدم `ANTHROPIC_API_KEY` إذا كان موجودًا أو يطلب مفتاحًا، ثم يحفظه لاستخدام daemon.
    - **Anthropic Claude CLI**: خيار مساعد Anthropic المفضل في الإعداد/configure. على macOS يتحقق الإعداد من عنصر Keychain المسمى "Claude Code-credentials" (اختر "Always Allow" حتى لا تمنع بدايات launchd)؛ وعلى Linux/Windows يعيد استخدام `~/.claude/.credentials.json` إذا كان موجودًا ويحوّل اختيار النموذج إلى مرجع قياسي `claude-cli/claude-*`.
    - **رمز إعداد Anthropic (قديم/يدوي)**: متاح مجددًا في الإعداد/configure، لكن Anthropic أبلغت مستخدمي OpenClaw أن مسار تسجيل دخول Claude في OpenClaw يُحتسب استخدامًا لحزمة طرف ثالث ويتطلب **Extra Usage** على حساب Claude.
    - **اشتراك OpenAI Code (Codex) ‏(Codex CLI)**: إذا كان `~/.codex/auth.json` موجودًا، يمكن للإعداد إعادة استخدامه. تظل بيانات اعتماد Codex CLI المعاد استخدامها مُدارة بواسطة Codex CLI؛ وعند انتهاء صلاحيتها يعيد OpenClaw قراءة هذا المصدر أولًا، وعندما يستطيع الموفر تحديثها يكتب بيانات الاعتماد المحدّثة مرة أخرى إلى تخزين Codex بدلًا من تولي إدارتها بنفسه.
    - **اشتراك OpenAI Code (Codex) ‏(OAuth)**: تدفق المتصفح؛ الصق `code#state`.
      - يضبط `agents.defaults.model` على `openai-codex/gpt-5.4` عندما يكون النموذج غير مضبوط أو `openai/*`.
    - **مفتاح OpenAI API**: يستخدم `OPENAI_API_KEY` إذا كان موجودًا أو يطلب مفتاحًا، ثم يخزنه في ملفات تعريف المصادقة.
      - يضبط `agents.defaults.model` على `openai/gpt-5.4` عندما يكون النموذج غير مضبوط، أو `openai/*`، أو `openai-codex/*`.
    - **مفتاح xAI (Grok) API**: يطلب `XAI_API_KEY` ويهيئ xAI كموفر نموذج.
    - **OpenCode**: يطلب `OPENCODE_API_KEY` (أو `OPENCODE_ZEN_API_KEY`، احصل عليه من https://opencode.ai/auth) ويتيح لك اختيار فهرس Zen أو Go.
    - **Ollama**: يطلب عنوان URL الأساسي لـ Ollama، ويعرض وضع **Cloud + Local** أو **Local**، ويكتشف النماذج المتاحة، ويسحب النموذج المحلي المحدد تلقائيًا عند الحاجة.
    - مزيد من التفاصيل: [Ollama](/ar/providers/ollama)
    - **مفتاح API**: يخزن المفتاح نيابةً عنك.
    - **Vercel AI Gateway (وكيل متعدد النماذج)**: يطلب `AI_GATEWAY_API_KEY`.
    - مزيد من التفاصيل: [Vercel AI Gateway](/ar/providers/vercel-ai-gateway)
    - **Cloudflare AI Gateway**: يطلب معرّف الحساب ومعرّف Gateway و`CLOUDFLARE_AI_GATEWAY_API_KEY`.
    - مزيد من التفاصيل: [Cloudflare AI Gateway](/ar/providers/cloudflare-ai-gateway)
    - **MiniMax**: يُكتب التكوين تلقائيًا؛ والافتراضي المستضاف هو `MiniMax-M2.7`.
      يستخدم إعداد مفتاح API ‏`minimax/...`، ويستخدم إعداد OAuth
      ‏`minimax-portal/...`.
    - مزيد من التفاصيل: [MiniMax](/ar/providers/minimax)
    - **StepFun**: يُكتب التكوين تلقائيًا لـ StepFun standard أو Step Plan على نقاط نهاية الصين أو النقاط العالمية.
    - يتضمن Standard حاليًا `step-3.5-flash`، كما يتضمن Step Plan أيضًا `step-3.5-flash-2603`.
    - مزيد من التفاصيل: [StepFun](/ar/providers/stepfun)
    - **Synthetic (متوافق مع Anthropic)**: يطلب `SYNTHETIC_API_KEY`.
    - مزيد من التفاصيل: [Synthetic](/ar/providers/synthetic)
    - **Moonshot (Kimi K2)**: يُكتب التكوين تلقائيًا.
    - **Kimi Coding**: يُكتب التكوين تلقائيًا.
    - مزيد من التفاصيل: [Moonshot AI (Kimi + Kimi Coding)](/ar/providers/moonshot)
    - **تخطي**: لم تُهيأ المصادقة بعد.
    - اختر نموذجًا افتراضيًا من الخيارات المكتشفة (أو أدخل provider/model يدويًا). للحصول على أفضل جودة وتقليل مخاطر حقن المطالبات، اختر أقوى نموذج متاح من أحدث جيل في مجموعة الموفرين لديك.
    - يُجري الإعداد فحصًا للنموذج ويعرض تحذيرًا إذا كان النموذج المُهيأ غير معروف أو تنقصه المصادقة.
    - يكون وضع تخزين مفاتيح API افتراضيًا هو قيم ملفات تعريف المصادقة بنص واضح. استخدم `--secret-input-mode ref` لتخزين مراجع مدعومة بمتغيرات البيئة بدلًا من ذلك (مثل `keyRef: { source: "env", provider: "default", id: "OPENAI_API_KEY" }`).
    - توجد ملفات تعريف المصادقة في `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` (مفاتيح API وOAuth). أما `~/.openclaw/credentials/oauth.json` فهو مصدر استيراد قديم فقط.
    - مزيد من التفاصيل: [/concepts/oauth](/ar/concepts/oauth)
    <Note>
    نصيحة للخوادم/البيئات بدون واجهة رسومية: أكمل OAuth على جهاز يحتوي على متصفح، ثم انسخ
    الملف `auth-profiles.json` لذلك الوكيل (على سبيل المثال
    `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`، أو المسار
    المطابق ضمن `$OPENCLAW_STATE_DIR/...`) إلى مضيف gateway. الملف `credentials/oauth.json`
    هو مجرد مصدر استيراد قديم.
    </Note>
  </Step>
  <Step title="مساحة العمل">
    - الافتراضي `~/.openclaw/workspace` (قابل للتكوين).
    - يملأ ملفات مساحة العمل اللازمة لطقس تمهيد الوكيل.
    - التخطيط الكامل لمساحة العمل + دليل النسخ الاحتياطي: [مساحة عمل الوكيل](/ar/concepts/agent-workspace)
  </Step>
  <Step title="Gateway">
    - المنفذ، والربط، ووضع المصادقة، وتعريض Tailscale.
    - توصية المصادقة: أبقِ **Token** حتى عند loopback لكي تضطر عملاء WS المحليون إلى المصادقة.
    - في وضع الرمز المميز، يوفر الإعداد التفاعلي:
      - **توليد/تخزين رمز مميز بنص واضح** (افتراضي)
      - **استخدام SecretRef** (اختياري)
      - يعيد Quickstart استخدام SecretRefs الموجودة في `gateway.auth.token` عبر موفري `env` و`file` و`exec` من أجل الفحص أثناء الإعداد وتهيئة dashboard.
      - إذا كان SecretRef هذا مهيأً لكن لا يمكن حله، يفشل الإعداد مبكرًا برسالة إصلاح واضحة بدلًا من التدهور الصامت في مصادقة runtime.
    - في وضع كلمة المرور، يدعم الإعداد التفاعلي أيضًا التخزين بنص واضح أو عبر SecretRef.
    - مسار SecretRef للرمز المميز في الوضع غير التفاعلي: `--gateway-token-ref-env <ENV_VAR>`.
      - يتطلب متغير بيئة غير فارغ في بيئة عملية الإعداد.
      - لا يمكن دمجه مع `--gateway-token`.
    - عطّل المصادقة فقط إذا كنت تثق تمامًا بكل عملية محلية.
    - تتطلب عمليات الربط غير loopback المصادقة أيضًا.
  </Step>
  <Step title="القنوات">
    - [WhatsApp](/ar/channels/whatsapp): تسجيل دخول QR اختياري.
    - [Telegram](/ar/channels/telegram): رمز مميز للبوت.
    - [Discord](/ar/channels/discord): رمز مميز للبوت.
    - [Google Chat](/ar/channels/googlechat): JSON لحساب خدمة + جمهور webhook.
    - [Mattermost](/ar/channels/mattermost) (plugin): رمز مميز للبوت + عنوان URL أساسي.
    - [Signal](/ar/channels/signal): تثبيت `signal-cli` اختياري + تكوين الحساب.
    - [BlueBubbles](/ar/channels/bluebubbles): **موصى به لـ iMessage**؛ عنوان URL للخادم + كلمة مرور + webhook.
    - [iMessage](/ar/channels/imessage): مسار CLI ‏`imsg` القديم + الوصول إلى قاعدة البيانات.
    - أمان الرسائل الخاصة: الافتراضي هو الاقتران. ترسل أول رسالة خاصة رمزًا؛ وافق عبر `openclaw pairing approve <channel> <code>` أو استخدم قوائم السماح.
  </Step>
  <Step title="البحث على الويب">
    - اختر موفرًا مدعومًا مثل Brave أو DuckDuckGo أو Exa أو Firecrawl أو Gemini أو Grok أو Kimi أو MiniMax Search أو Ollama Web Search أو Perplexity أو SearXNG أو Tavily (أو تخطَّ ذلك).
    - يمكن للموفرين المعتمدين على API استخدام متغيرات البيئة أو التكوين الحالي من أجل إعداد سريع؛ أما الموفرون الذين لا يحتاجون إلى مفتاح فيستخدمون المتطلبات المسبقة الخاصة بكل موفر.
    - تخطَّ ذلك باستخدام `--skip-search`.
    - هيّئه لاحقًا: `openclaw configure --section web`.
  </Step>
  <Step title="تثبيت daemon">
    - macOS: ‏LaunchAgent
      - يتطلب جلسة مستخدم مسجل دخوله؛ أما البيئات بدون واجهة فاستعمل LaunchDaemon مخصصًا (غير مشحون).
    - Linux (وWindows عبر WSL2): وحدة systemd للمستخدم
      - يحاول الإعداد تفعيل lingering عبر `loginctl enable-linger <user>` حتى يظل Gateway يعمل بعد تسجيل الخروج.
      - قد يطلب sudo (يكتب إلى `/var/lib/systemd/linger`)؛ ويحاول أولًا من دون sudo.
    - **اختيار runtime:** ‏Node (موصى به؛ مطلوب لـ WhatsApp/Telegram). أما Bun فهو **غير موصى به**.
    - إذا كانت مصادقة الرمز المميز تتطلب رمزًا وكان `gateway.auth.token` مُدارًا عبر SecretRef، فإن تثبيت daemon يتحقق منه لكنه لا يحفظ قيم الرمز المميز المحلولة بنص واضح ضمن بيانات بيئة خدمة المشرف.
    - إذا كانت مصادقة الرمز المميز تتطلب رمزًا وكان SecretRef المهيأ للرمز المميز غير محلول، فسيُمنع تثبيت daemon مع إرشادات عملية.
    - إذا كان كل من `gateway.auth.token` و`gateway.auth.password` مهيأين وكان `gateway.auth.mode` غير مضبوط، فسيُمنع تثبيت daemon حتى يُضبط الوضع صراحةً.
  </Step>
  <Step title="فحص الصحة">
    - يشغّل Gateway (عند الحاجة) ويشغّل `openclaw health`.
    - نصيحة: يضيف `openclaw status --deep` فحص صحة gateway الحي إلى مخرجات الحالة، بما في ذلك فحوصات القنوات عندما تكون مدعومة (يتطلب Gateway يمكن الوصول إليه).
  </Step>
  <Step title="Skills (موصى بها)">
    - يقرأ Skills المتاحة ويتحقق من المتطلبات.
    - يتيح لك اختيار مدير node: ‏**npm / pnpm** ‏(bun غير موصى به).
    - يثبت التبعيات الاختيارية (بعضها يستخدم Homebrew على macOS).
  </Step>
  <Step title="الإنهاء">
    - ملخص + الخطوات التالية، بما في ذلك تطبيقات iOS/Android/macOS لميزات إضافية.
  </Step>
</Steps>

<Note>
إذا لم يتم اكتشاف واجهة رسومية، فسيطبع الإعداد تعليمات إعادة توجيه منفذ SSH لواجهة Control UI بدلًا من فتح متصفح.
إذا كانت أصول Control UI مفقودة، فسيحاول الإعداد بناءها؛ والحل البديل هو `pnpm ui:build` (مع تثبيت تبعيات واجهة المستخدم تلقائيًا).
</Note>

## الوضع غير التفاعلي

استخدم `--non-interactive` لأتمتة الإعداد أو تشغيله برمجيًا:

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice apiKey \
  --anthropic-api-key "$ANTHROPIC_API_KEY" \
  --gateway-port 18789 \
  --gateway-bind loopback \
  --install-daemon \
  --daemon-runtime node \
  --skip-skills
```

أضف `--json` للحصول على ملخص قابل للقراءة آليًا.

SecretRef لرمز Gateway المميز في الوضع غير التفاعلي:

```bash
export OPENCLAW_GATEWAY_TOKEN="your-token"
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice skip \
  --gateway-auth token \
  --gateway-token-ref-env OPENCLAW_GATEWAY_TOKEN
```

`--gateway-token` و`--gateway-token-ref-env` متنافيان.

<Note>
إن `--json` **لا** يعني ضمنيًا الوضع غير التفاعلي. استخدم `--non-interactive` (و`--workspace`) في السكربتات.
</Note>

توجد أمثلة أوامر خاصة بالموفرين في [أتمتة CLI](/start/wizard-cli-automation#provider-specific-examples).
استخدم صفحة المرجع هذه لمعاني الأعلام وترتيب الخطوات.

### إضافة وكيل (غير تفاعلي)

```bash
openclaw agents add work \
  --workspace ~/.openclaw/workspace-work \
  --model openai/gpt-5.4 \
  --bind whatsapp:biz \
  --non-interactive \
  --json
```

## ‏RPC لمعالج Gateway

يكشف Gateway عن تدفق الإعداد عبر RPC ‏(`wizard.start` و`wizard.next` و`wizard.cancel` و`wizard.status`).
يمكن للعملاء (تطبيق macOS، وControl UI) عرض الخطوات دون إعادة تنفيذ منطق الإعداد.

## إعداد Signal ‏(`signal-cli`)

يمكن للإعداد تثبيت `signal-cli` من إصدارات GitHub:

- ينزّل أصل الإصدار المناسب.
- يخزنه تحت `~/.openclaw/tools/signal-cli/<version>/`.
- يكتب `channels.signal.cliPath` في التكوين الخاص بك.

ملاحظات:

- تتطلب إصدارات JVM ‏**Java 21**.
- تُستخدم الإصدارات الأصلية عندما تكون متاحة.
- يستخدم Windows ‏WSL2؛ ويتبع تثبيت signal-cli تدفق Linux داخل WSL.

## ما الذي يكتبه المعالج

الحقول المعتادة في `~/.openclaw/openclaw.json`:

- `agents.defaults.workspace`
- `agents.defaults.model` / `models.providers` (إذا تم اختيار Minimax)
- `tools.profile` (يضبط الإعداد المحلي افتراضيًا القيمة `"coding"` عندما لا تكون مضبوطة؛ وتُحفَظ القيم الصريحة الحالية)
- `gateway.*` (الوضع، والربط، والمصادقة، وTailscale)
- `session.dmScope` (تفاصيل السلوك: [مرجع إعداد CLI](/start/wizard-cli-reference#outputs-and-internals))
- `channels.telegram.botToken`, `channels.discord.token`, `channels.matrix.*`, `channels.signal.*`, `channels.imessage.*`
- قوائم السماح للقنوات (Slack/Discord/Matrix/Microsoft Teams) عندما تختار ذلك أثناء المطالبات (تُحل الأسماء إلى معرّفات عندما يكون ذلك ممكنًا).
- `skills.install.nodeManager`
  - يقبل `setup --node-manager` القيم `npm` أو `pnpm` أو `bun`.
  - لا يزال بالإمكان استخدام `yarn` في التكوين اليدوي عن طريق ضبط `skills.install.nodeManager` مباشرةً.
- `wizard.lastRunAt`
- `wizard.lastRunVersion`
- `wizard.lastRunCommit`
- `wizard.lastRunCommand`
- `wizard.lastRunMode`

يكتب `openclaw agents add` إلى `agents.list[]` و`bindings` الاختيارية.

توجد بيانات اعتماد WhatsApp تحت `~/.openclaw/credentials/whatsapp/<accountId>/`.
وتُخزن الجلسات تحت `~/.openclaw/agents/<agentId>/sessions/`.

تُسلَّم بعض القنوات على هيئة plugins. وعندما تختار واحدة أثناء الإعداد،
فسيطلب منك الإعداد تثبيتها (npm أو مسار محلي) قبل أن يمكن تهيئتها.

## الوثائق ذات الصلة

- نظرة عامة على الإعداد: [الإعداد (CLI)](/ar/start/wizard)
- إعداد تطبيق macOS: ‏[الإعداد](/start/onboarding)
- مرجع التكوين: ‏[إعداد Gateway](/ar/gateway/configuration)
- الموفرون: [WhatsApp](/ar/channels/whatsapp)، [Telegram](/ar/channels/telegram)، [Discord](/ar/channels/discord)، [Google Chat](/ar/channels/googlechat)، [Signal](/ar/channels/signal)، [BlueBubbles](/ar/channels/bluebubbles) ‏(iMessage)، [iMessage](/ar/channels/imessage) ‏(قديم)
- Skills: ‏[Skills](/tools/skills)، [إعداد Skills](/tools/skills-config)
