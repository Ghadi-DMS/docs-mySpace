---
read_when:
    - تشغيل حِزم البرمجة عبر ACP
    - إعداد جلسات ACP مرتبطة بالمحادثة على قنوات المراسلة
    - ربط محادثة قناة رسائل بجلسة ACP دائمة
    - استكشاف أخطاء الواجهة الخلفية لـ ACP وتوصيلات plugin وإصلاحها
    - تشغيل أوامر /acp من الدردشة
summary: استخدم جلسات وقت التشغيل ACP مع Codex وClaude Code وCursor وGemini CLI وOpenClaw ACP ووكلاء الحِزم الآخرين
title: وكلاء ACP
x-i18n:
    generated_at: "2026-04-05T12:58:43Z"
    model: gpt-5.4
    provider: openai
    source_hash: 47063abc8170129cd22808d9a4b23160d0f340f6dc789907589d349f68c12e3e
    source_path: tools/acp-agents.md
    workflow: 15
---

# وكلاء ACP

تتيح جلسات [Agent Client Protocol (ACP)](https://agentclientprotocol.com/) لـ OpenClaw تشغيل حِزم برمجة خارجية (مثل Pi وClaude Code وCodex وCursor وCopilot وOpenClaw ACP وOpenCode وGemini CLI وغيرها من حِزم ACPX المدعومة) عبر plugin واجهة خلفية لـ ACP.

إذا طلبت من OpenClaw بلغة طبيعية أن "يشغّل هذا في Codex" أو "يبدأ Claude Code في سلسلة محادثة"، فيجب على OpenClaw توجيه هذا الطلب إلى وقت تشغيل ACP (وليس إلى وقت تشغيل الوكيل الفرعي الأصلي). ويتم تتبّع كل عملية إنشاء جلسة ACP بوصفها [مهمة في الخلفية](/ar/automation/tasks).

إذا كنت تريد أن يتصل Codex أو Claude Code مباشرةً كعميل MCP خارجي
بمحادثات قنوات OpenClaw الموجودة، فاستخدم [`openclaw mcp serve`](/cli/mcp)
بدلًا من ACP.

## أي صفحة أريد؟

هناك ثلاثة أسطح قريبة يسهل الخلط بينها:

| تريد أن... | استخدم هذا | ملاحظات |
| --- | --- | --- |
| تشغيل Codex أو Claude Code أو Gemini CLI أو حزمة خارجية أخرى _عبر_ OpenClaw | هذه الصفحة: وكلاء ACP | جلسات مرتبطة بالدردشة، و`/acp spawn`، و`sessions_spawn({ runtime: "acp" })`، ومهام الخلفية، وعناصر تحكم وقت التشغيل |
| كشف جلسة OpenClaw Gateway _بوصفها_ خادم ACP لمحرر أو عميل | [`openclaw acp`](/cli/acp) | وضع الجسر. يتحدث IDE/العميل بروتوكول ACP مع OpenClaw عبر stdio/WebSocket |
| إعادة استخدام AI CLI محلي كنموذج احتياطي نصّي فقط | [واجهات CLI الخلفية](/ar/gateway/cli-backends) | ليس ACP. لا توجد أدوات OpenClaw، ولا عناصر تحكم ACP، ولا وقت تشغيل للحزمة |

## هل يعمل هذا مباشرةً بعد التثبيت؟

عادةً، نعم.

- تأتي التثبيتات الجديدة الآن مع plugin وقت التشغيل المضمّن `acpx` مفعّلًا افتراضيًا.
- يفضّل plugin ‏`acpx` المضمّن ملفه التنفيذي `acpx` المثبّت محليًا داخل plugin.
- عند بدء التشغيل، يفحص OpenClaw هذا الملف التنفيذي ويصلحه ذاتيًا عند الحاجة.
- ابدأ بـ `/acp doctor` إذا أردت فحص جاهزية سريعًا.

ما الذي قد يظل يحدث عند أول استخدام:

- قد يتم جلب مهايئ الحزمة الهدف عند الطلب باستخدام `npx` في أول مرة تستخدم فيها تلك الحزمة.
- لا يزال يجب أن تكون مصادقة المورّد موجودة على المضيف لتلك الحزمة.
- إذا لم يكن لدى المضيف وصول إلى npm/الشبكة، فقد تفشل عمليات جلب المهايئات في أول تشغيل حتى يتم تدفئة الذاكرات المؤقتة مسبقًا أو تثبيت المهايئ بطريقة أخرى.

أمثلة:

- `/acp spawn codex`: يجب أن يكون OpenClaw جاهزًا لتمهيد `acpx`، لكن مهايئ Codex ACP قد يظل بحاجة إلى جلب في أول تشغيل.
- `/acp spawn claude`: الأمر نفسه بالنسبة إلى مهايئ Claude ACP، بالإضافة إلى مصادقة Claude على ذلك المضيف.

## تدفق تشغيل سريع للمشغّل

استخدم هذا عندما تريد دليلاً عمليًا لأوامر `/acp`:

1. أنشئ جلسة:
   - `/acp spawn codex --bind here`
   - `/acp spawn codex --mode persistent --thread auto`
2. اعمل في المحادثة أو سلسلة المحادثة المرتبطة (أو استهدف مفتاح تلك الجلسة صراحةً).
3. تحقّق من حالة وقت التشغيل:
   - `/acp status`
4. اضبط خيارات وقت التشغيل حسب الحاجة:
   - `/acp model <provider/model>`
   - `/acp permissions <profile>`
   - `/acp timeout <seconds>`
5. وجّه جلسة نشطة من دون استبدال السياق:
   - `/acp steer tighten logging and continue`
6. أوقف العمل:
   - `/acp cancel` (إيقاف الدور الحالي)، أو
   - `/acp close` (إغلاق الجلسة + إزالة الارتباطات)

## بداية سريعة للبشر

أمثلة على الطلبات الطبيعية:

- "اربط قناة Discord هذه بـ Codex."
- "ابدأ جلسة Codex دائمة في سلسلة محادثة هنا وأبقها مركزة."
- "شغّل هذا كجلسة Claude Code ACP لمرة واحدة ولخّص النتيجة."
- "اربط دردشة iMessage هذه بـ Codex وأبق المتابعات في مساحة العمل نفسها."
- "استخدم Gemini CLI لهذه المهمة في سلسلة محادثة، ثم أبق المتابعات في تلك السلسلة نفسها."

ما الذي ينبغي أن يفعله OpenClaw:

1. يختار `runtime: "acp"`.
2. يحل هدف الحزمة المطلوب (`agentId`، مثل `codex`).
3. إذا طُلب ربط المحادثة الحالية وكان القناة النشطة تدعمه، يربط جلسة ACP بتلك المحادثة.
4. وإلا، إذا طُلب ربط سلسلة محادثة وكانت القناة الحالية تدعمه، يربط جلسة ACP بسلسلة المحادثة.
5. يوجّه رسائل المتابعة المرتبطة إلى جلسة ACP نفسها حتى يتم إلغاء التركيز/الإغلاق/الانتهاء.

## ACP مقابل الوكلاء الفرعيين

استخدم ACP عندما تريد وقت تشغيل لحزمة خارجية. واستخدم الوكلاء الفرعيين عندما تريد تشغيلات مفوّضة أصلية داخل OpenClaw.

| المجال | جلسة ACP | تشغيل وكيل فرعي |
| --- | --- | --- |
| وقت التشغيل | plugin واجهة خلفية لـ ACP ‏(مثل acpx) | وقت تشغيل الوكيل الفرعي الأصلي لـ OpenClaw |
| مفتاح الجلسة | `agent:<agentId>:acp:<uuid>` | `agent:<agentId>:subagent:<uuid>` |
| الأوامر الرئيسية | `/acp ...` | `/subagents ...` |
| أداة الإنشاء | `sessions_spawn` مع `runtime:"acp"` | `sessions_spawn` (وقت التشغيل الافتراضي) |

راجع أيضًا [الوكلاء الفرعيين](/tools/subagents).

## كيف يشغّل ACP ‏Claude Code

بالنسبة إلى Claude Code عبر ACP، تكون الطبقات كما يلي:

1. طبقة التحكم في جلسة OpenClaw ACP
2. plugin وقت التشغيل المضمّن `acpx`
3. مهايئ Claude ACP
4. وقت تشغيل/آلية جلسة Claude على جانبه

تمييز مهم:

- Claude عبر ACP ليس هو نفسه وقت التشغيل الاحتياطي المباشر `claude-cli/...`.
- Claude عبر ACP هو جلسة حزمة تتضمن عناصر تحكم ACP، واستئناف الجلسة، وتتبع مهام الخلفية، وربطًا اختياريًا بالمحادثة/سلسلة المحادثة.
- `claude-cli/...` هو واجهة CLI خلفية محلية نصية فقط. راجع [واجهات CLI الخلفية](/ar/gateway/cli-backends).

بالنسبة إلى المشغلين، القاعدة العملية هي:

- إذا كنت تريد `/acp spawn`، أو جلسات قابلة للربط، أو عناصر تحكم وقت التشغيل، أو عمل حزمة دائم: استخدم ACP
- إذا كنت تريد احتياطيًا نصيًا محليًا بسيطًا عبر CLI الخام: استخدم واجهات CLI الخلفية

## الجلسات المرتبطة

### الارتباطات بالمحادثة الحالية

استخدم `/acp spawn <harness> --bind here` عندما تريد أن تصبح المحادثة الحالية مساحة عمل ACP دائمة من دون إنشاء سلسلة فرعية.

السلوك:

- يحتفظ OpenClaw بملكية نقل القناة، والمصادقة، والأمان، والتسليم.
- تُثبَّت المحادثة الحالية على مفتاح جلسة ACP التي أُنشئت.
- تُوجَّه رسائل المتابعة في تلك المحادثة إلى جلسة ACP نفسها.
- يعيد `/new` و`/reset` ضبط جلسة ACP المرتبطة نفسها في مكانها.
- يغلق `/acp close` الجلسة ويزيل ارتباط المحادثة الحالية.

ما الذي يعنيه ذلك عمليًا:

- يبقي `--bind here` سطح الدردشة نفسه. ففي Discord، تبقى القناة الحالية هي القناة الحالية.
- يمكن لـ `--bind here` مع ذلك إنشاء جلسة ACP جديدة إذا كنت تنشئ عملًا جديدًا. يقوم الربط بإلحاق تلك الجلسة بالمحادثة الحالية.
- لا ينشئ `--bind here` سلسلة Discord فرعية أو موضوع Telegram بحد ذاته.
- يمكن لوقت تشغيل ACP أن يمتلك مع ذلك دليل عمل خاصًا به (`cwd`) أو مساحة عمل يديرها الـ backend على القرص. وهذه مساحة عمل وقت تشغيل منفصلة عن سطح الدردشة ولا تعني إنشاء سلسلة رسائل جديدة.
- إذا أنشأت جلسة لوكيل ACP مختلف ولم تمرّر `--cwd`، فإن OpenClaw يرث مساحة عمل **الوكيل الهدف** افتراضيًا، وليس مساحة عمل الطالب.
- إذا كان مسار مساحة العمل الموروثة هذا مفقودًا (`ENOENT`/`ENOTDIR`)، فإن OpenClaw يعود إلى `cwd` الافتراضي للواجهة الخلفية بدلًا من إعادة استخدام الشجرة الخاطئة بصمت.
- إذا كانت مساحة العمل الموروثة موجودة ولكن لا يمكن الوصول إليها (مثل `EACCES`)، فإن الإنشاء يعيد خطأ الوصول الحقيقي بدلًا من إسقاط `cwd`.

النموذج الذهني:

- سطح الدردشة: المكان الذي يواصل فيه الناس الكلام (`قناة Discord`، `موضوع Telegram`، `دردشة iMessage`)
- جلسة ACP: حالة وقت تشغيل Codex/Claude/Gemini الدائمة التي يوجّه إليها OpenClaw
- سلسلة/موضوع فرعي: سطح مراسلة إضافي اختياري يُنشأ فقط بواسطة `--thread ...`
- مساحة عمل وقت التشغيل: الموقع في نظام الملفات الذي تعمل فيه الحزمة (`cwd`، أو نسخة المستودع، أو مساحة عمل الواجهة الخلفية)

أمثلة:

- `/acp spawn codex --bind here`: أبقِ هذه الدردشة، وأنشئ أو أرفق جلسة Codex ACP، ووجّه الرسائل المستقبلية هنا إليها
- `/acp spawn codex --thread auto`: قد ينشئ OpenClaw سلسلة/موضوعًا فرعيًا ويربط جلسة ACP هناك
- `/acp spawn codex --bind here --cwd /workspace/repo`: الربط بالدردشة نفسها كما أعلاه، لكن Codex يعمل في `/workspace/repo`

دعم الربط بالمحادثة الحالية:

- يمكن لقنوات الدردشة/الرسائل التي تعلن دعم ربط المحادثة الحالية استخدام `--bind here` عبر مسار الربط المشترك للمحادثات.
- يمكن للقنوات ذات دلالات السلاسل/الموضوعات المخصصة أن توفّر مع ذلك تسوية خاصة بالقناة خلف الواجهة المشتركة نفسها.
- يعني `--bind here` دائمًا "اربط المحادثة الحالية في مكانها".
- تستخدم الارتباطات العامة بالمحادثة الحالية مخزن الارتباطات المشترك في OpenClaw وتبقى بعد عمليات إعادة تشغيل gateway العادية.

ملاحظات:

- لا يمكن الجمع بين `--bind here` و`--thread ...` في `/acp spawn`.
- في Discord، يربط `--bind here` القناة أو السلسلة الحالية في مكانها. ولا يكون `spawnAcpSessions` مطلوبًا إلا عندما يحتاج OpenClaw إلى إنشاء سلسلة فرعية باستخدام `--thread auto|here`.
- إذا لم تكشف القناة النشطة عن ارتباطات ACP للمحادثة الحالية، يعيد OpenClaw رسالة عدم دعم واضحة.
- أسئلة `resume` و"الجلسة الجديدة" هي أسئلة جلسة ACP وليست أسئلة قناة. يمكنك إعادة استخدام حالة وقت التشغيل أو استبدالها من دون تغيير سطح الدردشة الحالي.

### الجلسات المرتبطة بسلسلة محادثة

عند تفعيل ارتباطات السلاسل لمهايئ قناة ما، يمكن ربط جلسات ACP بالسلاسل:

- يربط OpenClaw سلسلة محادثة بجلسة ACP مستهدفة.
- تُوجَّه رسائل المتابعة في تلك السلسلة إلى جلسة ACP المرتبطة.
- يُعاد تسليم مخرجات ACP إلى السلسلة نفسها.
- تؤدي إزالة التركيز/الإغلاق/الأرشفة/انتهاء مهلة الخمول أو انتهاء العمر الأقصى إلى إزالة الارتباط.

يعتمد دعم الربط بسلسلة المحادثة على المهايئ. وإذا لم يكن مهايئ القناة النشطة يدعم ارتباطات السلاسل، يعيد OpenClaw رسالة واضحة تفيد بعدم الدعم/عدم التوفر.

أعلام الميزات المطلوبة لـ ACP المرتبط بسلسلة محادثة:

- `acp.enabled=true`
- `acp.dispatch.enabled` مفعّل افتراضيًا (اضبطه على `false` لإيقاف توجيه ACP مؤقتًا)
- تفعيل علم إنشاء سلسلة ACP في مهايئ القناة (خاص بالمهايئ)
  - Discord: ‏`channels.discord.threadBindings.spawnAcpSessions=true`
  - Telegram: ‏`channels.telegram.threadBindings.spawnAcpSessions=true`

### القنوات التي تدعم السلاسل

- أي مهايئ قناة يكشف قدرة ربط الجلسة/السلسلة.
- الدعم المضمّن الحالي:
  - سلاسل/قنوات Discord
  - مواضيع Telegram ‏(مواضيع المنتدى في المجموعات/المجموعات الخارقة ومواضيع الرسائل الخاصة)
- يمكن لقنوات plugin إضافة الدعم عبر واجهة الربط نفسها.

## إعدادات خاصة بالقناة

للتدفقات غير الزائلة، اضبط ارتباطات ACP الدائمة في إدخالات `bindings[]` ذات المستوى الأعلى.

### نموذج الربط

- يضع `bindings[].type="acp"` علامة على ربط محادثة ACP دائم.
- يحدّد `bindings[].match` المحادثة الهدف:
  - قناة أو سلسلة Discord: ‏`match.channel="discord"` + ‏`match.peer.id="<channelOrThreadId>"`
  - موضوع منتدى Telegram: ‏`match.channel="telegram"` + ‏`match.peer.id="<chatId>:topic:<topicId>"`
  - دردشة BlueBubbles الخاصة/الجماعية: ‏`match.channel="bluebubbles"` + ‏`match.peer.id="<handle|chat_id:*|chat_guid:*|chat_identifier:*>"`
    فضّل `chat_id:*` أو `chat_identifier:*` للارتباطات الجماعية المستقرة.
  - دردشة iMessage الخاصة/الجماعية: ‏`match.channel="imessage"` + ‏`match.peer.id="<handle|chat_id:*|chat_guid:*|chat_identifier:*>"`
    فضّل `chat_id:*` للارتباطات الجماعية المستقرة.
- `bindings[].agentId` هو معرّف وكيل OpenClaw المالك.
- تعيش تجاوزات ACP الاختيارية تحت `bindings[].acp`:
  - `mode` ‏(`persistent` أو `oneshot`)
  - `label`
  - `cwd`
  - `backend`

### الإعدادات الافتراضية لوقت التشغيل لكل وكيل

استخدم `agents.list[].runtime` لتعريف إعدادات ACP الافتراضية مرة واحدة لكل وكيل:

- `agents.list[].runtime.type="acp"`
- `agents.list[].runtime.acp.agent` ‏(معرّف الحزمة، مثل `codex` أو `claude`)
- `agents.list[].runtime.acp.backend`
- `agents.list[].runtime.acp.mode`
- `agents.list[].runtime.acp.cwd`

أولوية التجاوز للجلسات المرتبطة بـ ACP:

1. `bindings[].acp.*`
2. `agents.list[].runtime.acp.*`
3. إعدادات ACP العامة الافتراضية (مثل `acp.backend`)

مثال:

```json5
{
  agents: {
    list: [
      {
        id: "codex",
        runtime: {
          type: "acp",
          acp: {
            agent: "codex",
            backend: "acpx",
            mode: "persistent",
            cwd: "/workspace/openclaw",
          },
        },
      },
      {
        id: "claude",
        runtime: {
          type: "acp",
          acp: { agent: "claude", backend: "acpx", mode: "persistent" },
        },
      },
    ],
  },
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "discord",
        accountId: "default",
        peer: { kind: "channel", id: "222222222222222222" },
      },
      acp: { label: "codex-main" },
    },
    {
      type: "acp",
      agentId: "claude",
      match: {
        channel: "telegram",
        accountId: "default",
        peer: { kind: "group", id: "-1001234567890:topic:42" },
      },
      acp: { cwd: "/workspace/repo-b" },
    },
    {
      type: "route",
      agentId: "main",
      match: { channel: "discord", accountId: "default" },
    },
    {
      type: "route",
      agentId: "main",
      match: { channel: "telegram", accountId: "default" },
    },
  ],
  channels: {
    discord: {
      guilds: {
        "111111111111111111": {
          channels: {
            "222222222222222222": { requireMention: false },
          },
        },
      },
    },
    telegram: {
      groups: {
        "-1001234567890": {
          topics: { "42": { requireMention: false } },
        },
      },
    },
  },
}
```

السلوك:

- يضمن OpenClaw وجود جلسة ACP المضبوطة قبل الاستخدام.
- تُوجَّه الرسائل في تلك القناة أو الموضوع إلى جلسة ACP المضبوطة.
- في المحادثات المرتبطة، يعيد `/new` و`/reset` ضبط مفتاح جلسة ACP نفسه في مكانه.
- تظل ارتباطات وقت التشغيل المؤقتة (مثل التي تنشئها تدفقات التركيز على سلسلة محادثة) مطبّقة عند وجودها.
- في عمليات إنشاء ACP عبر وكلاء متعددة من دون `cwd` صريح، يرث OpenClaw مساحة عمل الوكيل الهدف من إعدادات الوكيل.
- تعود مسارات مساحة العمل الموروثة المفقودة إلى `cwd` الافتراضي للواجهة الخلفية؛ أما حالات فشل الوصول للمسارات الموجودة فتظهر كأخطاء إنشاء.

## بدء جلسات ACP ‏(الواجهات)

### من `sessions_spawn`

استخدم `runtime: "acp"` لبدء جلسة ACP من دور وكيل أو استدعاء أداة.

```json
{
  "task": "Open the repo and summarize failing tests",
  "runtime": "acp",
  "agentId": "codex",
  "thread": true,
  "mode": "session"
}
```

ملاحظات:

- تكون القيمة الافتراضية لـ `runtime` هي `subagent`، لذا اضبط `runtime: "acp"` صراحةً لجلسات ACP.
- إذا تم حذف `agentId`، يستخدم OpenClaw القيمة `acp.defaultAgent` عند ضبطها.
- يتطلب `mode: "session"` أن يكون `thread: true` للاحتفاظ بمحادثة مرتبطة دائمة.

تفاصيل الواجهة:

- `task` ‏(مطلوب): الموجّه الأولي المرسَل إلى جلسة ACP.
- `runtime` ‏(مطلوب لـ ACP): يجب أن يكون `"acp"`.
- `agentId` ‏(اختياري): معرّف حزمة ACP الهدف. يعود إلى `acp.defaultAgent` إذا كانت مضبوطة.
- `thread` ‏(اختياري، الافتراضي `false`): طلب تدفق ربط سلسلة محادثة حيثما كان مدعومًا.
- `mode` ‏(اختياري): `run` ‏(لمرة واحدة) أو `session` ‏(دائم).
  - القيمة الافتراضية هي `run`
  - إذا كان `thread: true` وتم حذف النمط، فقد يستخدم OpenClaw سلوكًا دائمًا افتراضيًا بحسب مسار وقت التشغيل
  - يتطلب `mode: "session"` أن يكون `thread: true`
- `cwd` ‏(اختياري): دليل العمل المطلوب لوقت التشغيل (تتحقق منه سياسة الـ backend/وقت التشغيل). وإذا تم حذفه، يرث إنشاء ACP مساحة عمل الوكيل الهدف عند ضبطها؛ وتعاد المسارات الموروثة المفقودة إلى إعدادات backend الافتراضية، بينما تُعاد أخطاء الوصول الحقيقية.
- `label` ‏(اختياري): تسمية موجهة للمشغّل تُستخدم في نص الجلسة/اللافتة.
- `resumeSessionId` ‏(اختياري): استئناف جلسة ACP موجودة بدلًا من إنشاء جلسة جديدة. يعيد الوكيل تشغيل محفوظات المحادثة عبر `session/load`. ويتطلب `runtime: "acp"`.
- `streamTo` ‏(اختياري): `"parent"` يبث ملخصات تقدم تشغيل ACP الأولي إلى جلسة الطالب على هيئة أحداث نظام.
  - عند توفره، قد تتضمن الردود المقبولة `streamLogPath` يشير إلى سجل JSONL خاص بالنطاق (`<sessionId>.acp-stream.jsonl`) يمكنك متابعته للاطلاع على كامل سجل الترحيل.

### استئناف جلسة موجودة

استخدم `resumeSessionId` لمتابعة جلسة ACP سابقة بدلًا من البدء من جديد. يعيد الوكيل تشغيل محفوظات المحادثة عبر `session/load`، لذا يواصل مع كامل سياق ما سبق.

```json
{
  "task": "Continue where we left off — fix the remaining test failures",
  "runtime": "acp",
  "agentId": "codex",
  "resumeSessionId": "<previous-session-id>"
}
```

حالات الاستخدام الشائعة:

- تسليم جلسة Codex من حاسوبك المحمول إلى هاتفك — اطلب من وكيلك أن يكمل من حيث توقفت
- متابعة جلسة برمجة بدأتَها تفاعليًا في CLI، لكن الآن من دون واجهة مباشرة عبر وكيلك
- متابعة عمل انقطع بسبب إعادة تشغيل gateway أو انتهاء مهلة الخمول

ملاحظات:

- يتطلب `resumeSessionId` ‏`runtime: "acp"` — ويعيد خطأ إذا استُخدم مع وقت تشغيل الوكيل الفرعي.
- يعيد `resumeSessionId` محفوظات المحادثة في ACP من الجهة العليا؛ بينما يظل `thread` و`mode` مطبّقين بشكل طبيعي على جلسة OpenClaw الجديدة التي تنشئها، لذا يظل `mode: "session"` يتطلب `thread: true`.
- يجب أن يدعم الوكيل الهدف `session/load` ‏(Codex وClaude Code يفعلان ذلك).
- إذا لم يُعثر على معرّف الجلسة، يفشل الإنشاء بخطأ واضح — من دون رجوع صامت إلى جلسة جديدة.

### اختبار دخاني للمشغّل

استخدم هذا بعد نشر gateway عندما تريد تحققًا مباشرًا سريعًا من أن إنشاء ACP
يعمل فعلًا من طرف إلى طرف، وليس مجرد اجتياز اختبارات الوحدة.

البوابة الموصى بها:

1. تحقّق من إصدار/التزام gateway المنشور على المضيف الهدف.
2. أكّد أن المصدر المنشور يتضمن قبول سلالة ACP في
   `src/gateway/sessions-patch.ts` ‏(`subagent:* or acp:* sessions`).
3. افتح جلسة جسر ACPX مؤقتة إلى وكيل مباشر (مثل
   `razor(main)` على `jpclawhq`).
4. اطلب من ذلك الوكيل استدعاء `sessions_spawn` بالقيم التالية:
   - `runtime: "acp"`
   - `agentId: "codex"`
   - `mode: "run"`
   - المهمة: `Reply with exactly LIVE-ACP-SPAWN-OK`
5. تحقّق من أن الوكيل يبلّغ عن:
   - `accepted=yes`
   - `childSessionKey` حقيقي
   - لا يوجد خطأ تحقق
6. نظّف جلسة جسر ACPX المؤقتة.

مثال على موجّه إلى وكيل مباشر:

```text
Use the sessions_spawn tool now with runtime: "acp", agentId: "codex", and mode: "run".
Set the task to: "Reply with exactly LIVE-ACP-SPAWN-OK".
Then report only: accepted=<yes/no>; childSessionKey=<value or none>; error=<exact text or none>.
```

ملاحظات:

- أبقِ هذا الاختبار الدخاني على `mode: "run"` ما لم تكن تختبر عمدًا
  جلسات ACP دائمة مرتبطة بسلسلة محادثة.
- لا تشترط `streamTo: "parent"` للبوابة الأساسية. فهذا المسار يعتمد على
  قدرات جلسة/الطالب وهو فحص تكاملي منفصل.
- تعامل مع اختبار `mode: "session"` المرتبط بسلسلة محادثة على أنه
  تمرير تكاملي ثانٍ وأكثر غنى من سلسلة Discord حقيقية أو موضوع Telegram.

## التوافق مع sandbox

تعمل جلسات ACP حاليًا على وقت تشغيل المضيف، وليس داخل sandbox الخاص بـ OpenClaw.

القيود الحالية:

- إذا كانت جلسة الطالب داخل sandbox، يتم حظر عمليات إنشاء ACP لكل من `sessions_spawn({ runtime: "acp" })` و`/acp spawn`.
  - الخطأ: `Sandboxed sessions cannot spawn ACP sessions because runtime="acp" runs on the host. Use runtime="subagent" from sandboxed sessions.`
- لا يدعم `sessions_spawn` مع `runtime: "acp"` القيمة `sandbox: "require"`.
  - الخطأ: `sessions_spawn sandbox="require" is unsupported for runtime="acp" because ACP sessions run outside the sandbox. Use runtime="subagent" or sandbox="inherit".`

استخدم `runtime: "subagent"` عندما تحتاج إلى تنفيذ مفروض بواسطة sandbox.

### من أمر `/acp`

استخدم `/acp spawn` لتحكم صريح من المشغّل عبر الدردشة عند الحاجة.

```text
/acp spawn codex --mode persistent --thread auto
/acp spawn codex --mode oneshot --thread off
/acp spawn codex --bind here
/acp spawn codex --thread here
```

الأعلام الرئيسية:

- `--mode persistent|oneshot`
- `--bind here|off`
- `--thread auto|here|off`
- `--cwd <absolute-path>`
- `--label <name>`

راجع [أوامر الشرطة المائلة](/tools/slash-commands).

## حل هدف الجلسة

تقبل معظم إجراءات `/acp` هدف جلسة اختياريًا (`session-key` أو `session-id` أو `session-label`).

ترتيب الحل:

1. وسيطة الهدف الصريحة (أو `--session` في `/acp steer`)
   - يحاول أولًا المفتاح
   - ثم `session id` على شكل UUID
   - ثم التسمية
2. ارتباط سلسلة المحادثة الحالية (إذا كانت هذه المحادثة/السلسلة مرتبطة بجلسة ACP)
3. الرجوع إلى جلسة الطالب الحالية

تشارك ارتباطات المحادثة الحالية وارتباطات سلاسل المحادثة كلاهما في الخطوة 2.

إذا لم يُحل أي هدف، يعيد OpenClaw خطأ واضحًا (`Unable to resolve session target: ...`).

## أوضاع ربط الإنشاء

يدعم `/acp spawn` الخيار `--bind here|off`.

| الوضع | السلوك |
| --- | --- |
| `here` | اربط المحادثة النشطة الحالية في مكانها؛ وافشل إذا لم تكن هناك محادثة نشطة. |
| `off` | لا تنشئ ارتباطًا بالمحادثة الحالية. |

ملاحظات:

- `--bind here` هو أبسط مسار للمشغّل لعبارة "اجعل هذه القناة أو الدردشة مدعومة بواسطة Codex."
- لا ينشئ `--bind here` سلسلة فرعية.
- يتوفر `--bind here` فقط في القنوات التي تكشف دعم الربط بالمحادثة الحالية.
- لا يمكن الجمع بين `--bind` و`--thread` في استدعاء `/acp spawn` نفسه.

## أوضاع سلاسل الإنشاء

يدعم `/acp spawn` الخيار `--thread auto|here|off`.

| الوضع | السلوك |
| --- | --- |
| `auto` | داخل سلسلة نشطة: اربط هذه السلسلة. خارج سلسلة: أنشئ/اربط سلسلة فرعية عند الدعم. |
| `here` | اشترط وجود سلسلة نشطة حاليًا؛ وافشل إذا لم تكن داخل سلسلة. |
| `off` | بلا ربط. تبدأ الجلسة غير مرتبطة. |

ملاحظات:

- على الأسطح التي لا تدعم ربط السلاسل، يكون السلوك الافتراضي فعليًا هو `off`.
- يتطلب الإنشاء المرتبط بسلسلة دعم سياسة القناة:
  - Discord: ‏`channels.discord.threadBindings.spawnAcpSessions=true`
  - Telegram: ‏`channels.telegram.threadBindings.spawnAcpSessions=true`
- استخدم `--bind here` عندما تريد تثبيت المحادثة الحالية من دون إنشاء سلسلة فرعية.

## عناصر تحكم ACP

عائلة الأوامر المتاحة:

- `/acp spawn`
- `/acp cancel`
- `/acp steer`
- `/acp close`
- `/acp status`
- `/acp set-mode`
- `/acp set`
- `/acp cwd`
- `/acp permissions`
- `/acp timeout`
- `/acp model`
- `/acp reset-options`
- `/acp sessions`
- `/acp doctor`
- `/acp install`

يعرض `/acp status` خيارات وقت التشغيل الفعلية، وعند توفرها، كلًا من معرّفات الجلسة على مستوى وقت التشغيل وعلى مستوى الواجهة الخلفية.

تعتمد بعض عناصر التحكم على قدرات الواجهة الخلفية. وإذا لم تدعم واجهة خلفية عنصر تحكم ما، يعيد OpenClaw خطأ واضحًا يفيد بعدم دعم عنصر التحكم.

## كتاب وصفات أوامر ACP

| الأمر | ما الذي يفعله | مثال |
| --- | --- | --- |
| `/acp spawn` | إنشاء جلسة ACP؛ مع ربط حالي اختياري أو ربط بسلسلة محادثة. | `/acp spawn codex --bind here --cwd /repo` |
| `/acp cancel` | إلغاء الدور الجاري للجلسة الهدف. | `/acp cancel agent:codex:acp:<uuid>` |
| `/acp steer` | إرسال تعليمة توجيه إلى جلسة قيد التشغيل. | `/acp steer --session support inbox prioritize failing tests` |
| `/acp close` | إغلاق الجلسة وفك ارتباط أهداف السلاسل. | `/acp close` |
| `/acp status` | عرض الواجهة الخلفية، والنمط، والحالة، وخيارات وقت التشغيل، والقدرات. | `/acp status` |
| `/acp set-mode` | ضبط نمط وقت التشغيل للجلسة الهدف. | `/acp set-mode plan` |
| `/acp set` | كتابة عامة لخيار إعداد وقت التشغيل. | `/acp set model openai/gpt-5.4` |
| `/acp cwd` | ضبط تجاوز دليل العمل لوقت التشغيل. | `/acp cwd /Users/user/Projects/repo` |
| `/acp permissions` | ضبط ملف تعريف سياسة الموافقة. | `/acp permissions strict` |
| `/acp timeout` | ضبط مهلة وقت التشغيل (بالثواني). | `/acp timeout 120` |
| `/acp model` | ضبط تجاوز نموذج وقت التشغيل. | `/acp model anthropic/claude-opus-4-6` |
| `/acp reset-options` | إزالة تجاوزات خيارات وقت التشغيل للجلسة. | `/acp reset-options` |
| `/acp sessions` | سرد جلسات ACP الحديثة من المخزن. | `/acp sessions` |
| `/acp doctor` | حالة الواجهة الخلفية، والقدرات، والإصلاحات العملية. | `/acp doctor` |
| `/acp install` | طباعة خطوات تثبيت وتمكين حتمية. | `/acp install` |

يقرأ `/acp sessions` المخزن للجلسة المرتبطة الحالية أو جلسة الطالب. وتقوم الأوامر التي تقبل رموز `session-key` أو `session-id` أو `session-label` بحل الأهداف عبر اكتشاف جلسات gateway، بما في ذلك جذور `session.store` المخصصة لكل وكيل.

## تعيين خيارات وقت التشغيل

يحتوي `/acp` على أوامر مريحة ومُعيّن عام.

عمليات مكافئة:

- ‏`/acp model <id>` تُعيَّن إلى مفتاح إعداد وقت التشغيل `model`.
- ‏`/acp permissions <profile>` تُعيَّن إلى مفتاح إعداد وقت التشغيل `approval_policy`.
- ‏`/acp timeout <seconds>` تُعيَّن إلى مفتاح إعداد وقت التشغيل `timeout`.
- ‏`/acp cwd <path>` يحدّث تجاوز `cwd` لوقت التشغيل مباشرةً.
- ‏`/acp set <key> <value>` هو المسار العام.
  - حالة خاصة: تستخدم `key=cwd` مسار تجاوز `cwd`.
- ‏`/acp reset-options` يمسح كل تجاوزات وقت التشغيل للجلسة الهدف.

## دعم حِزم acpx ‏(الحالي)

أسماء الحزم المستعارة المضمّنة حاليًا في acpx:

- `claude`
- `codex`
- `copilot`
- `cursor` ‏(Cursor CLI: ‏`cursor-agent acp`)
- `droid`
- `gemini`
- `iflow`
- `kilocode`
- `kimi`
- `kiro`
- `openclaw`
- `opencode`
- `pi`
- `qwen`

عندما يستخدم OpenClaw الواجهة الخلفية acpx، ففضّل هذه القيم لـ `agentId` ما لم يكن إعداد acpx لديك يعرّف أسماء مستعارة مخصصة للوكلاء.
إذا كان تثبيت Cursor المحلي لديك لا يزال يكشف ACP باسم `agent acp`، فتجاوز أمر الوكيل `cursor` في إعداد acpx بدلًا من تغيير القيمة الافتراضية المضمّنة.

يمكن أيضًا لاستخدام acpx CLI المباشر استهداف مهايئات اعتباطية عبر `--agent <command>`، لكن هذا المخرج الخام هو ميزة في acpx CLI (وليس المسار العادي لـ OpenClaw ‏`agentId`).

## الإعدادات المطلوبة

الخط الأساس الأساسي لـ ACP:

```json5
{
  acp: {
    enabled: true,
    // Optional. Default is true; set false to pause ACP dispatch while keeping /acp controls.
    dispatch: { enabled: true },
    backend: "acpx",
    defaultAgent: "codex",
    allowedAgents: [
      "claude",
      "codex",
      "copilot",
      "cursor",
      "droid",
      "gemini",
      "iflow",
      "kilocode",
      "kimi",
      "kiro",
      "openclaw",
      "opencode",
      "pi",
      "qwen",
    ],
    maxConcurrentSessions: 8,
    stream: {
      coalesceIdleMs: 300,
      maxChunkChars: 1200,
    },
    runtime: {
      ttlMinutes: 120,
    },
  },
}
```

إعداد ربط السلاسل خاص بمهايئ القناة. مثال لـ Discord:

```json5
{
  session: {
    threadBindings: {
      enabled: true,
      idleHours: 24,
      maxAgeHours: 0,
    },
  },
  channels: {
    discord: {
      threadBindings: {
        enabled: true,
        spawnAcpSessions: true,
      },
    },
  },
}
```

إذا لم ينجح إنشاء ACP المرتبط بسلسلة محادثة، فتحقق أولًا من علم ميزة المهايئ:

- Discord: ‏`channels.discord.threadBindings.spawnAcpSessions=true`

لا تتطلب ارتباطات المحادثة الحالية إنشاء سلسلة فرعية. لكنها تتطلب سياق محادثة نشطًا ومهايئ قناة يكشف ارتباطات محادثة ACP.

راجع [مرجع الإعدادات](/ar/gateway/configuration-reference).

## إعداد plugin للواجهة الخلفية acpx

تأتي التثبيتات الجديدة مع plugin وقت التشغيل المضمّن `acpx` مفعّلًا افتراضيًا، لذا فإن ACP
يعمل عادةً من دون خطوة تثبيت plugin يدوية.

ابدأ بـ:

```text
/acp doctor
```

إذا كنت قد عطلت `acpx`، أو حظرته عبر `plugins.allow` / `plugins.deny`، أو تريد
التبديل إلى نسخة تطوير محلية، فاستخدم مسار plugin الصريح:

```bash
openclaw plugins install acpx
openclaw config set plugins.entries.acpx.enabled true
```

تثبيت مساحة عمل محلية أثناء التطوير:

```bash
openclaw plugins install ./path/to/local/acpx-plugin
```

ثم تحقّق من سلامة الواجهة الخلفية:

```text
/acp doctor
```

### إعداد الأمر والإصدار لـ acpx

افتراضيًا، يستخدم plugin الواجهة الخلفية acpx المضمّن (`acpx`) الملف التنفيذي المثبّت محليًا داخل plugin:

1. يكون الأمر افتراضيًا هو `node_modules/.bin/acpx` المحلي داخل حزمة ACPX plugin.
2. يكون الإصدار المتوقع افتراضيًا هو الإصدار المثبّت في extension.
3. عند بدء التشغيل، يتم تسجيل واجهة ACP الخلفية فورًا على أنها غير جاهزة.
4. تتحقق مهمة ضمان في الخلفية من `acpx --version`.
5. إذا كان الملف التنفيذي المحلي داخل plugin مفقودًا أو لا يطابق الإصدار، فإنه يشغّل:
   `npm install --omit=dev --no-save acpx@<pinned>` ثم يعيد التحقق.

يمكنك تجاوز الأمر/الإصدار في إعدادات plugin:

```json
{
  "plugins": {
    "entries": {
      "acpx": {
        "enabled": true,
        "config": {
          "command": "../acpx/dist/cli.js",
          "expectedVersion": "any"
        }
      }
    }
  }
}
```

ملاحظات:

- يقبل `command` مسارًا مطلقًا، أو مسارًا نسبيًا، أو اسم أمر (`acpx`).
- تُحل المسارات النسبية انطلاقًا من دليل مساحة عمل OpenClaw.
- يؤدي `expectedVersion: "any"` إلى تعطيل المطابقة الصارمة للإصدار.
- عندما يشير `command` إلى ملف تنفيذي/مسار مخصص، يتم تعطيل التثبيت التلقائي المحلي داخل plugin.
- يظل بدء تشغيل OpenClaw غير حاجب أثناء تنفيذ فحص سلامة الواجهة الخلفية.

راجع [Plugins](/tools/plugin).

### التثبيت التلقائي للتبعيات

عندما تثبّت OpenClaw بشكل عام باستخدام `npm install -g openclaw`، يتم تثبيت
تبعيات وقت تشغيل acpx ‏(الملفات التنفيذية الخاصة بالمنصة) تلقائيًا
عبر خطاف postinstall. وإذا فشل التثبيت التلقائي، فإن gateway يبدأ
بشكل طبيعي مع الإبلاغ عن التبعية المفقودة عبر `openclaw acp doctor`.

### جسر MCP لأدوات plugin

افتراضيًا، لا تكشف جلسات ACPX **أدوات OpenClaw المسجّلة بواسطة plugins** إلى
حزمة ACP.

إذا كنت تريد أن تستدعي وكلاء ACP مثل Codex أو Claude Code
أدوات OpenClaw plugin المثبّتة مثل استدعاء/تخزين الذاكرة، فعِّل الجسر المخصص:

```bash
openclaw config set plugins.entries.acpx.config.pluginToolsMcpBridge true
```

ما الذي يفعله هذا:

- يحقن خادم MCP مضمّنًا باسم `openclaw-plugin-tools` في تمهيد جلسة ACPX.
- يكشف أدوات plugin المسجّلة مسبقًا بواسطة plugins المثبّتة والمفعّلة في OpenClaw.
- يبقي الميزة صريحة وموقوفة افتراضيًا.

ملاحظات الأمان والثقة:

- يوسّع هذا سطح الأدوات في حزمة ACP.
- يحصل وكلاء ACP على وصول فقط إلى أدوات plugin النشطة أصلًا في gateway.
- تعامل مع هذا على أنه حد الثقة نفسه الذي ينطبق عند السماح لتلك plugins بالتنفيذ داخل OpenClaw نفسه.
- راجع plugins المثبّتة قبل تفعيل ذلك.

تظل إعدادات `mcpServers` المخصصة تعمل كما كانت من قبل. وجسر أدوات plugin المضمّن هو
ميزة راحة إضافية اختيارية، وليس بديلًا عن إعداد خادم MCP العام.

## إعدادات الأذونات

تعمل جلسات ACP دون تفاعل مباشر — فلا توجد واجهة TTY للموافقة على مطالبات أذونات كتابة الملفات وتشغيل الصدفة أو رفضها. يوفّر plugin ‏acpx مفتاحي إعداد يتحكمان في كيفية التعامل مع الأذونات:

أذونات حِزم ACPX هذه منفصلة عن موافقات exec في OpenClaw ومنفصلة عن أعلام تجاوز المورّد في واجهات CLI الخلفية مثل Claude CLI ‏`--permission-mode bypassPermissions`. وتُعد ACPX ‏`approve-all` مفتاح الطوارئ على مستوى الحزمة لجلسات ACP.

### `permissionMode`

يتحكم في العمليات التي يمكن لوكيل الحزمة تنفيذها من دون مطالبة.

| القيمة | السلوك |
| --- | --- |
| `approve-all` | الموافقة تلقائيًا على جميع عمليات كتابة الملفات وأوامر الصدفة. |
| `approve-reads` | الموافقة تلقائيًا على القراءات فقط؛ أما الكتابات والتنفيذ فتتطلب مطالبات. |
| `deny-all` | رفض جميع مطالبات الأذونات. |

### `nonInteractivePermissions`

يتحكم فيما يحدث عندما يفترض عرض مطالبة إذن لكن لا تتوفر واجهة TTY تفاعلية (وهذا هو الحال دائمًا لجلسات ACP).

| القيمة | السلوك |
| --- | --- |
| `fail` | إجهاض الجلسة مع `AcpRuntimeError`. **(الافتراضي)** |
| `deny` | رفض الإذن بصمت والمتابعة (تدهور سلس). |

### الإعداد

اضبط عبر إعدادات plugin:

```bash
openclaw config set plugins.entries.acpx.config.permissionMode approve-all
openclaw config set plugins.entries.acpx.config.nonInteractivePermissions fail
```

أعد تشغيل gateway بعد تغيير هذه القيم.

> **مهم:** يستخدم OpenClaw حاليًا افتراضيًا `permissionMode=approve-reads` و`nonInteractivePermissions=fail`. في جلسات ACP غير التفاعلية، قد تفشل أي عملية كتابة أو تنفيذ تُطلق مطالبة إذن بخطأ `AcpRuntimeError: Permission prompt unavailable in non-interactive mode`.
>
> إذا كنت بحاجة إلى تقييد الأذونات، فاضبط `nonInteractivePermissions` على `deny` بحيث تتدهور الجلسات بسلاسة بدلًا من التعطل.

## استكشاف الأخطاء وإصلاحها

| العَرَض | السبب المحتمل | الإصلاح |
| --- | --- | --- |
| `ACP runtime backend is not configured` | plugin الواجهة الخلفية مفقود أو معطّل. | ثبّت plugin الواجهة الخلفية وفعّله، ثم شغّل `/acp doctor`. |
| `ACP is disabled by policy (acp.enabled=false)` | تم تعطيل ACP عالميًا. | اضبط `acp.enabled=true`. |
| `ACP dispatch is disabled by policy (acp.dispatch.enabled=false)` | تم تعطيل التوجيه من رسائل السلسلة العادية. | اضبط `acp.dispatch.enabled=true`. |
| `ACP agent "<id>" is not allowed by policy` | الوكيل غير موجود في قائمة السماح. | استخدم `agentId` مسموحًا به أو حدّث `acp.allowedAgents`. |
| `Unable to resolve session target: ...` | رمز مفتاح/معرّف/تسمية غير صحيح. | شغّل `/acp sessions`، وانسخ المفتاح/التسمية بدقة، ثم أعد المحاولة. |
| `--bind here requires running /acp spawn inside an active ... conversation` | استُخدم `--bind here` من دون محادثة نشطة قابلة للربط. | انتقل إلى الدردشة/القناة المستهدفة وأعد المحاولة، أو استخدم إنشاء غير مرتبط. |
| `Conversation bindings are unavailable for <channel>.` | لا يملك المهايئ قدرة ربط ACP للمحادثة الحالية. | استخدم `/acp spawn ... --thread ...` حيثما كان مدعومًا، أو اضبط `bindings[]` ذات المستوى الأعلى، أو انتقل إلى قناة مدعومة. |
| `--thread here requires running /acp spawn inside an active ... thread` | استُخدم `--thread here` خارج سياق سلسلة محادثة. | انتقل إلى السلسلة المستهدفة أو استخدم `--thread auto`/`off`. |
| `Only <user-id> can rebind this channel/conversation/thread.` | مستخدم آخر يملك هدف الربط النشط. | أعد الربط بصفتك المالك أو استخدم محادثة أو سلسلة مختلفة. |
| `Thread bindings are unavailable for <channel>.` | لا يملك المهايئ قدرة ربط السلاسل. | استخدم `--thread off` أو انتقل إلى مهايئ/قناة مدعومة. |
| `Sandboxed sessions cannot spawn ACP sessions ...` | وقت تشغيل ACP يعمل على المضيف؛ وجلسة الطالب داخل sandbox. | استخدم `runtime="subagent"` من جلسات sandbox، أو شغّل إنشاء ACP من جلسة خارج sandbox. |
| `sessions_spawn sandbox="require" is unsupported for runtime="acp" ...` | تم طلب `sandbox="require"` لوقت تشغيل ACP. | استخدم `runtime="subagent"` عند الحاجة إلى sandbox إلزامي، أو استخدم ACP مع `sandbox="inherit"` من جلسة خارج sandbox. |
| Missing ACP metadata for bound session | بيانات تعريف ACP قديمة/محذوفة للجلسة المرتبطة. | أعد إنشاءها باستخدام `/acp spawn`، ثم أعد الربط/التركيز على السلسلة. |
| `AcpRuntimeError: Permission prompt unavailable in non-interactive mode` | يمنع `permissionMode` عمليات الكتابة/التنفيذ في جلسة ACP غير تفاعلية. | اضبط `plugins.entries.acpx.config.permissionMode` على `approve-all` وأعد تشغيل gateway. راجع [إعدادات الأذونات](#إعدادات-الأذونات). |
| ACP session fails early with little output | تُحظر مطالبات الأذونات بواسطة `permissionMode`/`nonInteractivePermissions`. | تحقّق من سجلات gateway بحثًا عن `AcpRuntimeError`. للحصول على أذونات كاملة، اضبط `permissionMode=approve-all`؛ وللتدهور السلس، اضبط `nonInteractivePermissions=deny`. |
| ACP session stalls indefinitely after completing work | انتهت عملية الحزمة لكن جلسة ACP لم تبلغ عن الاكتمال. | راقب باستخدام `ps aux \| grep acpx`؛ واقتل العمليات القديمة يدويًا. |
