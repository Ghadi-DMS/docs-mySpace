---
read_when:
    - البحث عن خطوة أو عَلَم محدد في الإعداد الأولي
    - أتمتة الإعداد الأولي باستخدام الوضع غير التفاعلي
    - تصحيح سلوك الإعداد الأولي
sidebarTitle: Onboarding Reference
summary: 'المرجع الكامل للإعداد الأولي عبر CLI: كل خطوة وعَلَم وحقل تكوين'
title: مرجع الإعداد الأولي
x-i18n:
    generated_at: "2026-04-06T03:12:56Z"
    model: gpt-5.4
    provider: openai
    source_hash: e02a4da4a39ba335199095723f5d3b423671eb12efc2d9e4f9e48c1e8ee18419
    source_path: reference/wizard.md
    workflow: 15
---

# مرجع الإعداد الأولي

هذا هو المرجع الكامل للأمر `openclaw onboard`.
للحصول على نظرة عامة عالية المستوى، راجع [Onboarding (CLI)](/ar/start/wizard).

## تفاصيل التدفق (الوضع المحلي)

<Steps>
  <Step title="اكتشاف التكوين الموجود">
    - إذا كان `~/.openclaw/openclaw.json` موجودًا، فاختر **Keep / Modify / Reset**.
    - لا تؤدي إعادة تشغيل الإعداد الأولي إلى مسح أي شيء ما لم تختر صراحة **Reset**
      (أو تمرر `--reset`).
    - الإعداد الافتراضي لـ CLI مع `--reset` هو `config+creds+sessions`؛ استخدم `--reset-scope full`
      لإزالة مساحة العمل أيضًا.
    - إذا كان التكوين غير صالح أو يحتوي على مفاتيح قديمة، يتوقف المعالج ويطلب
      منك تشغيل `openclaw doctor` قبل المتابعة.
    - تستخدم إعادة التعيين `trash` (وليس `rm` مطلقًا) وتوفر النطاقات التالية:
      - التكوين فقط
      - التكوين + بيانات الاعتماد + الجلسات
      - إعادة تعيين كاملة (تزيل مساحة العمل أيضًا)
  </Step>
  <Step title="النموذج/المصادقة">
    - **Anthropic API key**: يستخدم `ANTHROPIC_API_KEY` إذا كان موجودًا أو يطلب مفتاحًا، ثم يحفظه لاستخدام daemon.
    - **Anthropic API key**: خيار مساعد Anthropic المفضل في onboarding/configure.
    - **Anthropic setup-token (legacy/manual)**: متاح مرة أخرى في onboarding/configure، لكن Anthropic أخبرت مستخدمي OpenClaw أن مسار Claude-login في OpenClaw يُحتسب كاستخدام لطرف ثالث ويتطلب **Extra Usage** على حساب Claude.
    - **OpenAI Code (Codex) subscription (Codex CLI)**: إذا كان `~/.codex/auth.json` موجودًا، يمكن للإعداد الأولي إعادة استخدامه. تظل بيانات اعتماد Codex CLI المعاد استخدامها مُدارة بواسطة Codex CLI؛ وعند انتهاء صلاحيتها، يعيد OpenClaw قراءة ذلك المصدر أولًا، وعندما يتمكن الموفّر من تحديثها، يكتب بيانات الاعتماد المحدّثة مرة أخرى إلى تخزين Codex بدلًا من تولي إدارتها بنفسه.
    - **OpenAI Code (Codex) subscription (OAuth)**: تدفق عبر المتصفح؛ الصق `code#state`.
      - يضبط `agents.defaults.model` على `openai-codex/gpt-5.4` عندما لا يكون النموذج مضبوطًا أو يكون `openai/*`.
    - **OpenAI API key**: يستخدم `OPENAI_API_KEY` إذا كان موجودًا أو يطلب مفتاحًا، ثم يخزنه في ملفات المصادقة.
      - يضبط `agents.defaults.model` على `openai/gpt-5.4` عندما لا يكون النموذج مضبوطًا، أو يكون `openai/*`، أو `openai-codex/*`.
    - **xAI (Grok) API key**: يطلب `XAI_API_KEY` ويكوّن xAI كموفّر نماذج.
    - **OpenCode**: يطلب `OPENCODE_API_KEY` (أو `OPENCODE_ZEN_API_KEY`، احصل عليه من https://opencode.ai/auth) ويتيح لك اختيار كتالوج Zen أو Go.
    - **Ollama**: يطلب عنوان URL الأساسي لـ Ollama، ويعرض وضعي **Cloud + Local** أو **Local**، ويكتشف النماذج المتاحة، ويجلب النموذج المحلي المحدد تلقائيًا عند الحاجة.
    - مزيد من التفاصيل: [Ollama](/ar/providers/ollama)
    - **API key**: يخزن المفتاح نيابة عنك.
    - **Vercel AI Gateway (multi-model proxy)**: يطلب `AI_GATEWAY_API_KEY`.
    - مزيد من التفاصيل: [Vercel AI Gateway](/ar/providers/vercel-ai-gateway)
    - **Cloudflare AI Gateway**: يطلب Account ID وGateway ID و`CLOUDFLARE_AI_GATEWAY_API_KEY`.
    - مزيد من التفاصيل: [Cloudflare AI Gateway](/ar/providers/cloudflare-ai-gateway)
    - **MiniMax**: يُكتب التكوين تلقائيًا؛ والافتراضي المستضاف هو `MiniMax-M2.7`.
      يستخدم إعداد API key الصيغة `minimax/...`، ويستخدم إعداد OAuth
      الصيغة `minimax-portal/...`.
    - مزيد من التفاصيل: [MiniMax](/ar/providers/minimax)
    - **StepFun**: يُكتب التكوين تلقائيًا لـ StepFun standard أو Step Plan على نقاط نهاية الصين أو العالمية.
    - يتضمن Standard حاليًا `step-3.5-flash`، ويتضمن Step Plan أيضًا `step-3.5-flash-2603`.
    - مزيد من التفاصيل: [StepFun](/ar/providers/stepfun)
    - **Synthetic (Anthropic-compatible)**: يطلب `SYNTHETIC_API_KEY`.
    - مزيد من التفاصيل: [Synthetic](/ar/providers/synthetic)
    - **Moonshot (Kimi K2)**: يُكتب التكوين تلقائيًا.
    - **Kimi Coding**: يُكتب التكوين تلقائيًا.
    - مزيد من التفاصيل: [Moonshot AI (Kimi + Kimi Coding)](/ar/providers/moonshot)
    - **Skip**: لم تُكوَّن أي مصادقة بعد.
    - اختر نموذجًا افتراضيًا من الخيارات المكتشفة (أو أدخل الموفّر/النموذج يدويًا). للحصول على أفضل جودة وتقليل خطر حقن المطالبات، اختر أقوى نموذج من أحدث جيل متاح في مجموعة الموفّرين لديك.
    - يجري الإعداد الأولي فحصًا للنموذج ويحذّر إذا كان النموذج المكوَّن غير معروف أو تنقصه المصادقة.
    - الوضع الافتراضي لتخزين API key هو قيم ملفات المصادقة بنص صريح. استخدم `--secret-input-mode ref` لتخزين مراجع مدعومة بـ env بدلًا من ذلك (مثل `keyRef: { source: "env", provider: "default", id: "OPENAI_API_KEY" }`).
    - توجد ملفات المصادقة في `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` ‏(API keys + OAuth). ويُعد `~/.openclaw/credentials/oauth.json` مصدر استيراد قديم فقط.
    - مزيد من التفاصيل: [/concepts/oauth](/ar/concepts/oauth)
    <Note>
    نصيحة للبيئات عديمة الواجهة/الخوادم: أكمل OAuth على جهاز يحتوي على متصفح، ثم انسخ
    ملف `auth-profiles.json` الخاص بذلك الوكيل (مثل
    `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`، أو المسار المطابق
    `$OPENCLAW_STATE_DIR/...`) إلى مضيف البوابة. ويظل `credentials/oauth.json`
    مجرد مصدر استيراد قديم.
    </Note>
  </Step>
  <Step title="مساحة العمل">
    - الافتراضي هو `~/.openclaw/workspace` (قابل للتكوين).
    - يهيئ ملفات مساحة العمل المطلوبة لطقس bootstrap الخاص بالوكيل.
    - دليل كامل لتخطيط مساحة العمل والنسخ الاحتياطي: [Agent workspace](/ar/concepts/agent-workspace)
  </Step>
  <Step title="البوابة">
    - المنفذ، والربط، ووضع المصادقة، وتعريض Tailscale.
    - توصية المصادقة: احتفظ بـ **Token** حتى مع loopback حتى تظل عملاء WS المحليون بحاجة إلى المصادقة.
    - في وضع token، يوفّر الإعداد التفاعلي:
      - **Generate/store plaintext token** (الافتراضي)
      - **Use SecretRef** (اختياري)
      - تعيد Quickstart استخدام قيم SecretRef الموجودة في `gateway.auth.token` عبر موفري `env` و`file` و`exec` من أجل probe/bootstrap الإعداد الأولي ولوحة المعلومات.
      - إذا كان SecretRef هذا مكوَّنًا ولكن لا يمكن حله، يفشل الإعداد الأولي مبكرًا مع رسالة إصلاح واضحة بدلًا من إضعاف مصادقة وقت التشغيل بصمت.
    - في وضع كلمة المرور، يدعم الإعداد التفاعلي أيضًا التخزين بنص صريح أو عبر SecretRef.
    - مسار SecretRef الخاص بـ token في الوضع غير التفاعلي: `--gateway-token-ref-env <ENV_VAR>`.
      - يتطلب متغير env غير فارغ في بيئة عملية الإعداد الأولي.
      - لا يمكن دمجه مع `--gateway-token`.
    - عطّل المصادقة فقط إذا كنت تثق تمامًا بكل عملية محلية.
    - تتطلب عمليات الربط غير loopback المصادقة أيضًا.
  </Step>
  <Step title="القنوات">
    - [WhatsApp](/ar/channels/whatsapp): تسجيل دخول اختياري عبر QR.
    - [Telegram](/ar/channels/telegram): token البوت.
    - [Discord](/ar/channels/discord): token البوت.
    - [Google Chat](/ar/channels/googlechat): JSON حساب خدمة + جمهور webhook.
    - [Mattermost](/ar/channels/mattermost) ‏(plugin): token البوت + عنوان URL الأساسي.
    - [Signal](/ar/channels/signal): تثبيت اختياري لـ `signal-cli` + تكوين الحساب.
    - [BlueBubbles](/ar/channels/bluebubbles): **موصى به لـ iMessage**؛ عنوان URL للخادم + كلمة مرور + webhook.
    - [iMessage](/ar/channels/imessage): مسار `imsg` CLI القديم + الوصول إلى قاعدة البيانات.
    - أمان الرسائل الخاصة: الإعداد الافتراضي هو الاقتران. ترسل أول رسالة خاصة رمزًا؛ وافق عبر `openclaw pairing approve <channel> <code>` أو استخدم قوائم السماح.
  </Step>
  <Step title="البحث في الويب">
    - اختر موفرًا مدعومًا مثل Brave أو DuckDuckGo أو Exa أو Firecrawl أو Gemini أو Grok أو Kimi أو MiniMax Search أو Ollama Web Search أو Perplexity أو SearXNG أو Tavily (أو تخطَّ هذه الخطوة).
    - يمكن للموفرين المعتمدين على API استخدام متغيرات env أو التكوين الحالي للإعداد السريع؛ أما الموفّرون الذين لا يحتاجون إلى مفتاح فيستخدمون المتطلبات المسبقة الخاصة بكل موفّر.
    - تخطَّ هذه الخطوة باستخدام `--skip-search`.
    - قم بالتكوين لاحقًا: `openclaw configure --section web`.
  </Step>
  <Step title="تثبيت daemon">
    - macOS: ‏LaunchAgent
      - يتطلب جلسة مستخدم مسجل دخوله؛ وللبيئات عديمة الواجهة، استخدم LaunchDaemon مخصصًا (غير مرفق).
    - Linux (وWindows عبر WSL2): وحدة مستخدم systemd
      - يحاول الإعداد الأولي تمكين lingering عبر `loginctl enable-linger <user>` حتى تظل البوابة قيد التشغيل بعد تسجيل الخروج.
      - قد يطلب sudo (يكتب إلى `/var/lib/systemd/linger`)؛ ويحاول أولًا بدون sudo.
    - **اختيار وقت التشغيل:** Node ‏(موصى به؛ مطلوب لـ WhatsApp/Telegram). أما Bun فهو **غير موصى به**.
    - إذا كانت مصادقة token تتطلب token وكان `gateway.auth.token` مُدارًا بواسطة SecretRef، فإن تثبيت daemon يتحقق منه لكنه لا يحفظ قيم token النصية المحلولة في بيانات بيئة الخدمة الخاصة بالمشرف.
    - إذا كانت مصادقة token تتطلب token وكان SecretRef المكوَّن للـ token غير محلول، يُمنع تثبيت daemon مع إرشادات عملية.
    - إذا كان كل من `gateway.auth.token` و`gateway.auth.password` مكوَّنين بينما لم يُضبط `gateway.auth.mode`، يُمنع تثبيت daemon إلى أن يُضبط الوضع صراحة.
  </Step>
  <Step title="فحص السلامة">
    - يبدأ البوابة (عند الحاجة) ويشغّل `openclaw health`.
    - نصيحة: يضيف `openclaw status --deep` probe سلامة البوابة المباشر إلى مخرجات الحالة، بما في ذلك probes القنوات عند الدعم (ويتطلب بوابة يمكن الوصول إليها).
  </Step>
  <Step title="Skills (موصى بها)">
    - يقرأ Skills المتاحة ويتحقق من المتطلبات.
    - يتيح لك اختيار مدير node: ‏**npm / pnpm** ‏(bun غير موصى به).
    - يثبّت التبعيات الاختيارية (يستخدم بعضها Homebrew على macOS).
  </Step>
  <Step title="الإنهاء">
    - ملخص + الخطوات التالية، بما في ذلك تطبيقات iOS/Android/macOS للحصول على ميزات إضافية.
  </Step>
</Steps>

<Note>
إذا لم يتم اكتشاف واجهة رسومية، يطبع الإعداد الأولي تعليمات توجيه منفذ SSH الخاصة بـ Control UI بدلًا من فتح متصفح.
إذا كانت أصول Control UI مفقودة، يحاول الإعداد الأولي بناءها؛ وخيار الرجوع هو `pnpm ui:build` (مع تثبيت تبعيات UI تلقائيًا).
</Note>

## الوضع غير التفاعلي

استخدم `--non-interactive` لأتمتة الإعداد الأولي أو تشغيله عبر سكربت:

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

SecretRef الخاص بـ token البوابة في الوضع غير التفاعلي:

```bash
export OPENCLAW_GATEWAY_TOKEN="your-token"
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice skip \
  --gateway-auth token \
  --gateway-token-ref-env OPENCLAW_GATEWAY_TOKEN
```

لا يمكن استخدام `--gateway-token` و`--gateway-token-ref-env` معًا.

<Note>
لا يعني `--json` **الوضع غير التفاعلي** تلقائيًا. استخدم `--non-interactive` (و`--workspace`) مع السكربتات.
</Note>

توجد أمثلة الأوامر الخاصة بكل موفّر في [CLI Automation](/ar/start/wizard-cli-automation#provider-specific-examples).
استخدم هذه الصفحة المرجعية لمعاني الأعلام وترتيب الخطوات.

### إضافة وكيل (غير تفاعلي)

```bash
openclaw agents add work \
  --workspace ~/.openclaw/workspace-work \
  --model openai/gpt-5.4 \
  --bind whatsapp:biz \
  --non-interactive \
  --json
```

## Gateway wizard RPC

تعرض البوابة تدفق الإعداد الأولي عبر RPC ‏(`wizard.start`, `wizard.next`, `wizard.cancel`, `wizard.status`).
يمكن للعملاء (تطبيق macOS وControl UI) عرض الخطوات من دون إعادة تنفيذ منطق الإعداد الأولي.

## إعداد Signal ‏(`signal-cli`)

يمكن للإعداد الأولي تثبيت `signal-cli` من إصدارات GitHub:

- ينزّل أصل الإصدار المناسب.
- يخزنه ضمن `~/.openclaw/tools/signal-cli/<version>/`.
- يكتب `channels.signal.cliPath` إلى التكوين الخاص بك.

ملاحظات:

- تتطلب بنيات JVM وجود **Java 21**.
- تُستخدم البنيات الأصلية عندما تكون متاحة.
- يستخدم Windows ‏WSL2؛ ويتبع تثبيت signal-cli تدفق Linux داخل WSL.

## ما الذي يكتبه المعالج

الحقول المعتادة في `~/.openclaw/openclaw.json`:

- `agents.defaults.workspace`
- `agents.defaults.model` / `models.providers` (إذا تم اختيار Minimax)
- `tools.profile` (يضبط onboarding المحلي افتراضيًا القيمة `"coding"` عندما لا تكون مضبوطة؛ وتظل القيم الصريحة الحالية محفوظة)
- `gateway.*` (الوضع، والربط، والمصادقة، وTailscale)
- `session.dmScope` (تفاصيل السلوك: [CLI Setup Reference](/ar/start/wizard-cli-reference#outputs-and-internals))
- `channels.telegram.botToken`, `channels.discord.token`, `channels.matrix.*`, `channels.signal.*`, `channels.imessage.*`
- قوائم السماح للقنوات (Slack/Discord/Matrix/Microsoft Teams) عندما تختار تفعيلها أثناء المطالبات (تُحل الأسماء إلى معرّفات عند الإمكان).
- `skills.install.nodeManager`
  - يقبل `setup --node-manager` القيم `npm` أو `pnpm` أو `bun`.
  - ما يزال التكوين اليدوي قادرًا على استخدام `yarn` عبر ضبط `skills.install.nodeManager` مباشرة.
- `wizard.lastRunAt`
- `wizard.lastRunVersion`
- `wizard.lastRunCommit`
- `wizard.lastRunCommand`
- `wizard.lastRunMode`

يكتب `openclaw agents add` إلى `agents.list[]` و`bindings` الاختيارية.

توجد بيانات اعتماد WhatsApp ضمن `~/.openclaw/credentials/whatsapp/<accountId>/`.
وتُخزن الجلسات ضمن `~/.openclaw/agents/<agentId>/sessions/`.

تُسلَّم بعض القنوات على هيئة plugins. وعندما تختار إحداها أثناء الإعداد،
سيطلب منك الإعداد الأولي تثبيتها (من npm أو من مسار محلي) قبل أن يمكن تكوينها.

## المستندات ذات الصلة

- نظرة عامة على الإعداد الأولي: [Onboarding (CLI)](/ar/start/wizard)
- الإعداد الأولي لتطبيق macOS: ‏[Onboarding](/ar/start/onboarding)
- مرجع التكوين: ‏[Gateway configuration](/ar/gateway/configuration)
- الموفّرون: [WhatsApp](/ar/channels/whatsapp)، [Telegram](/ar/channels/telegram)، [Discord](/ar/channels/discord)، [Google Chat](/ar/channels/googlechat)، [Signal](/ar/channels/signal)، [BlueBubbles](/ar/channels/bluebubbles) ‏(iMessage)، [iMessage](/ar/channels/imessage) ‏(قديم)
- Skills: ‏[Skills](/ar/tools/skills)، [Skills config](/ar/tools/skills-config)
