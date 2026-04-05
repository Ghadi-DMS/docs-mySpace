---
read_when:
    - تشغيل الإعداد التمهيدي عبر CLI أو تهيئته
    - إعداد جهاز جديد
sidebarTitle: 'Onboarding: CLI'
summary: 'الإعداد التمهيدي عبر CLI: إعداد موجّه للبوابة ومساحة العمل والقنوات وSkills'
title: الإعداد التمهيدي (CLI)
x-i18n:
    generated_at: "2026-04-05T11:37:03Z"
    model: gpt-5.4
    provider: openai
    source_hash: 81e33fb4f8be30e7c2c6e0024bf9bdcf48583ca58eaf5fff5afd37a1cd628523
    source_path: start/wizard.md
    workflow: 15
---

# الإعداد التمهيدي (CLI)

يُعد الإعداد التمهيدي عبر CLI الطريقة **الموصى بها** لإعداد OpenClaw على macOS،
وLinux، أو Windows (عبر WSL2؛ موصى به بشدة).
وهو يهيّئ Gateway محليًا أو اتصالًا بـ Gateway بعيد، بالإضافة إلى القنوات وSkills،
وإعدادات مساحة العمل الافتراضية ضمن تدفق موجّه واحد.

```bash
openclaw onboard
```

<Info>
أسرع طريقة لأول محادثة: افتح واجهة التحكم (لا حاجة إلى إعداد قناة). شغّل
`openclaw dashboard` وابدأ الدردشة في المتصفح. الوثائق: [لوحة التحكم](/web/dashboard).
</Info>

لإعادة التهيئة لاحقًا:

```bash
openclaw configure
openclaw agents add <name>
```

<Note>
لا يعني `--json` وضع عدم التفاعل تلقائيًا. بالنسبة إلى السكربتات، استخدم `--non-interactive`.
</Note>

<Tip>
يتضمن الإعداد التمهيدي عبر CLI خطوة للبحث على الويب يمكنك فيها اختيار موفّر
مثل Brave أو DuckDuckGo أو Exa أو Firecrawl أو Gemini أو Grok أو Kimi أو MiniMax Search،
أو Ollama Web Search أو Perplexity أو SearXNG أو Tavily. تتطلب بعض الموفّرات
مفتاح API، بينما لا يتطلب بعضها الآخر أي مفتاح. يمكنك أيضًا تهيئة ذلك لاحقًا باستخدام
`openclaw configure --section web`. الوثائق: [أدوات الويب](/tools/web).
</Tip>

## QuickStart مقابل Advanced

يبدأ الإعداد التمهيدي بخيار **QuickStart** (الإعدادات الافتراضية) مقابل **Advanced** (تحكم كامل).

<Tabs>
  <Tab title="QuickStart (الإعدادات الافتراضية)">
    - Gateway محلي (loopback)
    - مساحة العمل الافتراضية (أو مساحة عمل موجودة)
    - منفذ Gateway **18789**
    - مصادقة Gateway باستخدام **Token** (يُنشأ تلقائيًا، حتى على loopback)
    - سياسة الأدوات الافتراضية للإعدادات المحلية الجديدة: `tools.profile: "coding"` (يتم الاحتفاظ بأي profile صريح موجود)
    - إعداد عزل الرسائل المباشرة الافتراضي: يكتب الإعداد التمهيدي المحلي `session.dmScope: "per-channel-peer"` عندما لا يكون معيّنًا. التفاصيل: [مرجع إعداد CLI](/start/wizard-cli-reference#outputs-and-internals)
    - إتاحة Tailscale **معطلة**
    - تكون الرسائل المباشرة في Telegram وWhatsApp افتراضيًا ضمن **قائمة السماح** (سيُطلب منك رقم هاتفك)
  </Tab>
  <Tab title="Advanced (تحكم كامل)">
    - يكشف كل خطوة (الوضع، مساحة العمل، Gateway، القنوات، daemon، Skills).
  </Tab>
</Tabs>

## ما الذي يهيئه الإعداد التمهيدي

**الوضع المحلي (الافتراضي)** يرشدك خلال هذه الخطوات:

1. **النموذج/المصادقة** — اختر أي موفّر/تدفق مصادقة مدعوم (مفتاح API، أو OAuth، أو مصادقة يدوية خاصة بالموفّر)، بما في ذلك Custom Provider
   (متوافق مع OpenAI، أو متوافق مع Anthropic، أو Unknown مع اكتشاف تلقائي). اختر نموذجًا افتراضيًا.
   ملاحظة أمنية: إذا كان هذا الوكيل سيشغّل أدوات أو يعالج محتوى webhook/hooks، ففضّل أقوى نموذج متاح من الجيل الأحدث، وأبقِ سياسة الأدوات صارمة. تكون الفئات الأضعف/الأقدم أسهل في حقن المطالبات فيها.
   بالنسبة إلى عمليات التشغيل غير التفاعلية، يخزّن `--secret-input-mode ref` مراجع مدعومة بالبيئة داخل ملفات تعريف المصادقة بدلًا من قيم مفاتيح API النصية الصريحة.
   في وضع `ref` غير التفاعلي، يجب تعيين متغير البيئة الخاص بالموفّر؛ ويفشل التمرير المضمن لعلامات المفتاح من دون متغير البيئة هذا مباشرة.
   في عمليات التشغيل التفاعلية، يتيح لك اختيار وضع المرجع السري الإشارة إما إلى متغير بيئة أو إلى مرجع موفّر مُهيأ (`file` أو `exec`) مع تحقق تمهيدي سريع قبل الحفظ.
   بالنسبة إلى Anthropic، يوفّر الإعداد التمهيدي/التهيئة التفاعلية **Anthropic Claude CLI** كخيار احتياطي محلي و**مفتاح Anthropic API** كمسار الإنتاج الموصى به. أصبح Anthropic setup-token متاحًا أيضًا مرة أخرى كمسار OpenClaw قديم/يدوي، مع توقّع **Extra Usage** الخاص بـ OpenClaw في الفوترة لدى Anthropic.
2. **مساحة العمل** — موقع ملفات الوكيل (الافتراضي `~/.openclaw/workspace`). يملأ ملفات bootstrap الأولية.
3. **Gateway** — المنفذ، وعنوان الربط، ووضع المصادقة، وإتاحة Tailscale.
   في وضع token التفاعلي، اختر تخزين token النصي الصريح الافتراضي أو انتقل إلى SecretRef.
   مسار SecretRef الخاص بـ token في الوضع غير التفاعلي: `--gateway-token-ref-env <ENV_VAR>`.
4. **القنوات** — قنوات الدردشة المضمنة والمجمعة مثل BlueBubbles وDiscord وFeishu وGoogle Chat وMattermost وMicrosoft Teams وQQ Bot وSignal وSlack وTelegram وWhatsApp والمزيد.
5. **Daemon** — يثبّت LaunchAgent (على macOS)، أو وحدة systemd للمستخدم (على Linux/WSL2)، أو Scheduled Task أصليًا في Windows مع بديل لكل مستخدم في مجلد Startup.
   إذا كانت مصادقة token تتطلب token وكان `gateway.auth.token` مُدارًا عبر SecretRef، فإن تثبيت daemon يتحقق منه لكنه لا يستمر في حفظ token الذي تم حلّه داخل بيانات بيئة الخدمة التابعة للمشرف.
   إذا كانت مصادقة token تتطلب token وكان SecretRef المهيأ لـ token غير قابل للحل، فسيُحظر تثبيت daemon مع إرشادات قابلة للتنفيذ.
   إذا كان كل من `gateway.auth.token` و`gateway.auth.password` مهيأين وكانت `gateway.auth.mode` غير معيّنة، فسيُحظر تثبيت daemon حتى يتم تعيين الوضع صراحة.
6. **فحص السلامة** — يشغّل Gateway ويتحقق من أنه يعمل.
7. **Skills** — يثبّت Skills الموصى بها والاعتماديات الاختيارية.

<Note>
إن إعادة تشغيل الإعداد التمهيدي **لا** تمسح أي شيء ما لم تختر **Reset** صراحةً (أو تمرر `--reset`).
يستخدم `--reset` في CLI افتراضيًا النطاقات config وcredentials وsessions؛ استخدم `--reset-scope full` لتضمين مساحة العمل.
إذا كانت الإعدادات غير صالحة أو تحتوي على مفاتيح قديمة، فسيطلب منك الإعداد التمهيدي تشغيل `openclaw doctor` أولًا.
</Note>

**الوضع البعيد** يهيّئ العميل المحلي فقط للاتصال بـ Gateway موجود في مكان آخر.
وهو **لا** يثبت أو يغيّر أي شيء على المضيف البعيد.

## إضافة وكيل آخر

استخدم `openclaw agents add <name>` لإنشاء وكيل منفصل له مساحة العمل الخاصة به،
وجلساته، وملفات تعريف المصادقة الخاصة به. يؤدي التشغيل من دون `--workspace` إلى بدء الإعداد التمهيدي.

ما الذي يعيّنه:

- `agents.list[].name`
- `agents.list[].workspace`
- `agents.list[].agentDir`

ملاحظات:

- تتبع مساحات العمل الافتراضية النمط `~/.openclaw/workspace-<agentId>`.
- أضف `bindings` لتوجيه الرسائل الواردة (يمكن للإعداد التمهيدي تنفيذ ذلك).
- علامات الوضع غير التفاعلي: `--model` و`--agent-dir` و`--bind` و`--non-interactive`.

## المرجع الكامل

للاطلاع على تفصيلات خطوة بخطوة ومخرجات الإعدادات، راجع
[مرجع إعداد CLI](/start/wizard-cli-reference).
وللحصول على أمثلة غير تفاعلية، راجع [أتمتة CLI](/start/wizard-cli-automation).
وللمرجع التقني الأعمق، بما في ذلك تفاصيل RPC، راجع
[مرجع الإعداد التمهيدي](/reference/wizard).

## الوثائق ذات الصلة

- مرجع أوامر CLI: [`openclaw onboard`](/cli/onboard)
- نظرة عامة على الإعداد التمهيدي: [نظرة عامة على الإعداد التمهيدي](/start/onboarding-overview)
- الإعداد التمهيدي لتطبيق macOS: [الإعداد التمهيدي](/start/onboarding)
- طقس التشغيل الأول للوكيل: [التهيئة الأولية للوكيل](/start/bootstrapping)
