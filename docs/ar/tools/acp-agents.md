---
read_when:
    - تشغيل حاضنات البرمجة عبر ACP
    - إعداد جلسات ACP مرتبطة بالمحادثة على قنوات المراسلة
    - ربط محادثة قناة رسائل بجلسة ACP مستمرة
    - استكشاف مشكلات الواجهة الخلفية لـ ACP وتوصيلات plugin وإصلاحها
    - تشغيل أوامر `/acp` من الدردشة
summary: استخدم جلسات تشغيل ACP لـ Codex وClaude Code وCursor وGemini CLI وOpenClaw ACP ووكلاء الحاضنة الآخرين
title: وكلاء ACP
x-i18n:
    generated_at: "2026-04-08T02:22:49Z"
    model: gpt-5.4
    provider: openai
    source_hash: 71c7c0cdae5247aefef17a0029360950a1c2987ddcee21a1bb7d78c67da52950
    source_path: tools/acp-agents.md
    workflow: 15
---

# وكلاء ACP

تتيح جلسات [Agent Client Protocol (ACP)](https://agentclientprotocol.com/) لـ OpenClaw تشغيل حاضنات برمجة خارجية (مثل Pi وClaude Code وCodex وCursor وCopilot وOpenClaw ACP وOpenCode وGemini CLI وغيرها من حاضنات ACPX المدعومة) عبر plugin واجهة خلفية لـ ACP.

إذا طلبت من OpenClaw بلغة عادية "شغّل هذا في Codex" أو "ابدأ Claude Code في سلسلة محادثة"، فيفترض أن يوجّه OpenClaw هذا الطلب إلى وقت تشغيل ACP (وليس إلى وقت تشغيل الوكيل الفرعي الأصلي). يتم تتبع كل عملية إنشاء جلسة ACP باعتبارها [مهمة في الخلفية](/ar/automation/tasks).

إذا كنت تريد أن يتصل Codex أو Claude Code مباشرةً كعميل MCP خارجي
بمحادثات قنوات OpenClaw الموجودة، فاستخدم [`openclaw mcp serve`](/cli/mcp)
بدلًا من ACP.

## ما الصفحة التي أريدها؟

هناك ثلاثة أسطح متقاربة يسهل الخلط بينها:

| أنت تريد أن...                                                                     | استخدم هذا                              | ملاحظات                                                                                                       |
| ---------------------------------------------------------------------------------- | --------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| تشغيل Codex أو Claude Code أو Gemini CLI أو حاضنة خارجية أخرى _عبر_ OpenClaw | هذه الصفحة: وكلاء ACP                 | جلسات مرتبطة بالدردشة، و`/acp spawn`، و`sessions_spawn({ runtime: "acp" })`، والمهام في الخلفية، وعناصر تحكم وقت التشغيل |
| عرض جلسة OpenClaw Gateway _باعتبارها_ خادم ACP لمحرر أو عميل      | [`openclaw acp`](/cli/acp)              | وضع الجسر. يتحدث IDE/العميل مع OpenClaw عبر ACP باستخدام stdio/WebSocket                                          |
| إعادة استخدام AI CLI محلي كنموذج احتياطي نصي فقط                                 | [الواجهات الخلفية لـ CLI](/ar/gateway/cli-backends) | ليس ACP. لا توجد أدوات OpenClaw ولا عناصر تحكم ACP ولا وقت تشغيل للحاضنة                                             |

## هل يعمل هذا مباشرةً دون إعداد إضافي؟

عادةً، نعم.

- تأتي التثبيتات الجديدة الآن مع plugin وقت التشغيل المضمّن `acpx` مفعّلًا افتراضيًا.
- يفضّل plugin `acpx` المضمّن ثنائي `acpx` المثبّت محليًا داخل plugin.
- عند بدء التشغيل، يفحص OpenClaw هذا الثنائي ويصلحه ذاتيًا إذا لزم الأمر.
- ابدأ بـ `/acp doctor` إذا كنت تريد فحصًا سريعًا للجاهزية.

ما الذي قد يحدث مع أول استخدام رغم ذلك:

- قد يتم جلب مهيئ الحاضنة المستهدفة عند الطلب باستخدام `npx` في أول مرة تستخدم فيها تلك الحاضنة.
- يجب أن يكون توثيق المورّد موجودًا على المضيف لتلك الحاضنة.
- إذا لم يكن لدى المضيف وصول إلى npm/الشبكة، فقد يفشل جلب المهيئات عند أول تشغيل إلى أن تُسخّن الذاكرات المؤقتة مسبقًا أو يُثبَّت المهيئ بطريقة أخرى.

أمثلة:

- `/acp spawn codex`: ينبغي أن يكون OpenClaw جاهزًا لتهيئة `acpx`، لكن قد يظل مهيئ Codex ACP بحاجة إلى جلب أولي.
- `/acp spawn claude`: نفس الفكرة بالنسبة إلى مهيئ Claude ACP، إضافةً إلى توثيق Claude على ذلك المضيف.

## تدفق التشغيل السريع

استخدم هذا عندما تريد دليلاً عمليًا لتشغيل `/acp`:

1. أنشئ جلسة:
   - `/acp spawn codex --bind here`
   - `/acp spawn codex --mode persistent --thread auto`
2. اعمل في المحادثة أو السلسلة المرتبطة (أو استهدف مفتاح تلك الجلسة صراحةً).
3. تحقّق من حالة وقت التشغيل:
   - `/acp status`
4. اضبط خيارات وقت التشغيل حسب الحاجة:
   - `/acp model <provider/model>`
   - `/acp permissions <profile>`
   - `/acp timeout <seconds>`
5. وجّه جلسة نشطة دون استبدال السياق:
   - `/acp steer tighten logging and continue`
6. أوقف العمل:
   - `/acp cancel` (إيقاف الدور الحالي)، أو
   - `/acp close` (إغلاق الجلسة + إزالة الارتباطات)

## البدء السريع للبشر

أمثلة على الطلبات الطبيعية:

- "اربط قناة Discord هذه بـ Codex."
- "ابدأ جلسة Codex مستمرة في سلسلة محادثة هنا وأبقها مركّزة."
- "شغّل هذا كجلسة Claude Code ACP لمرة واحدة ولخّص النتيجة."
- "اربط دردشة iMessage هذه بـ Codex وأبقِ المتابعات في مساحة العمل نفسها."
- "استخدم Gemini CLI لهذه المهمة في سلسلة محادثة، ثم أبقِ المتابعات في السلسلة نفسها."

ما الذي ينبغي أن يفعله OpenClaw:

1. يختار `runtime: "acp"`.
2. يحلّ هدف الحاضنة المطلوب (`agentId`، مثل `codex`).
3. إذا طُلِب الربط بالمحادثة الحالية وكانت القناة النشطة تدعمه، يربط جلسة ACP بهذه المحادثة.
4. بخلاف ذلك، إذا طُلِب الربط بسلسلة محادثة وكانت القناة الحالية تدعمه، يربط جلسة ACP بهذه السلسلة.
5. يوجّه رسائل المتابعة المرتبطة إلى جلسة ACP نفسها إلى أن يتم إلغاء التركيز عنها أو إغلاقها أو انتهاء صلاحيتها.

## ACP مقابل الوكلاء الفرعيين

استخدم ACP عندما تريد وقت تشغيل لحاضنة خارجية. واستخدم الوكلاء الفرعيين عندما تريد تشغيلات مفوّضة أصلية في OpenClaw.

| المجال          | جلسة ACP                           | تشغيل وكيل فرعي                      |
| --------------- | ---------------------------------- | ------------------------------------ |
| وقت التشغيل       | plugin واجهة خلفية لـ ACP (مثل acpx) | وقت تشغيل الوكيل الفرعي الأصلي في OpenClaw  |
| مفتاح الجلسة   | `agent:<agentId>:acp:<uuid>`          | `agent:<agentId>:subagent:<uuid>`    |
| الأوامر الرئيسية | `/acp ...`                            | `/subagents ...`                     |
| أداة الإنشاء    | `sessions_spawn` مع `runtime:"acp"` | `sessions_spawn` (وقت التشغيل الافتراضي) |

راجع أيضًا [الوكلاء الفرعيون](/ar/tools/subagents).

## كيف يشغّل ACP ‏Claude Code

عند تشغيل Claude Code عبر ACP، تكون البنية كالتالي:

1. مستوى التحكم في جلسات OpenClaw ACP
2. plugin وقت التشغيل المضمّن `acpx`
3. مهيئ Claude ACP
4. آليات وقت التشغيل/الجلسة الخاصة بـ Claude

تمييز مهم:

- Claude عبر ACP هو جلسة حاضنة مع عناصر تحكم ACP، واستئناف الجلسة، وتتبع المهام في الخلفية، وإمكانية ربط المحادثة/سلسلة المحادثة اختياريًا.
- الواجهات الخلفية لـ CLI هي أوقات تشغيل احتياطية محلية منفصلة تعمل بالنص فقط. راجع [الواجهات الخلفية لـ CLI](/ar/gateway/cli-backends).

بالنسبة إلى المشغّلين، القاعدة العملية هي:

- إذا كنت تريد `/acp spawn` أو جلسات قابلة للربط أو عناصر تحكم لوقت التشغيل أو عمل حاضنة مستمر: استخدم ACP
- إذا كنت تريد احتياطيًا نصيًا محليًا بسيطًا عبر CLI الخام: استخدم الواجهات الخلفية لـ CLI

## الجلسات المرتبطة

### الارتباطات بالمحادثة الحالية

استخدم `/acp spawn <harness> --bind here` عندما تريد أن تصبح المحادثة الحالية مساحة عمل ACP دائمة دون إنشاء سلسلة محادثة فرعية.

السلوك:

- يظل OpenClaw مسؤولًا عن نقل القناة، والتوثيق، والسلامة، والتسليم.
- تُثبَّت المحادثة الحالية على مفتاح جلسة ACP المنشأة.
- تُوجَّه رسائل المتابعة في تلك المحادثة إلى جلسة ACP نفسها.
- يعيد `/new` و`/reset` تعيين جلسة ACP المرتبطة نفسها في مكانها.
- يغلق `/acp close` الجلسة ويزيل ارتباط المحادثة الحالية.

ما الذي يعنيه هذا عمليًا:

- `--bind here` يبقي نفس سطح الدردشة. على Discord، تظل القناة الحالية هي القناة الحالية.
- يمكن أن ينشئ `--bind here` جلسة ACP جديدة إذا كنت تنشئ عملًا جديدًا. يربط هذا الخيار تلك الجلسة بالمحادثة الحالية.
- لا ينشئ `--bind here` سلسلة Discord فرعية أو موضوع Telegram من تلقاء نفسه.
- يمكن لوقت تشغيل ACP أن يملك مع ذلك دليل عمل (`cwd`) خاصًا به أو مساحة عمل يديرها في الواجهة الخلفية على القرص. مساحة عمل وقت التشغيل هذه منفصلة عن سطح الدردشة ولا تعني وجود سلسلة رسائل جديدة.
- إذا أنشأت جلسة لوكيل ACP مختلف ولم تمرر `--cwd`، فسيرث OpenClaw مساحة عمل **الوكيل المستهدف** افتراضيًا، وليس مساحة عمل الطالب.
- إذا كان مسار مساحة العمل الموروثة هذا مفقودًا (`ENOENT`/`ENOTDIR`)، فسيعود OpenClaw إلى `cwd` الافتراضي للواجهة الخلفية بدلًا من إعادة استخدام الشجرة الخاطئة بصمت.
- إذا كانت مساحة العمل الموروثة موجودة لكن تعذر الوصول إليها (مثل `EACCES`)، فستعيد عملية الإنشاء خطأ الوصول الحقيقي بدلًا من إسقاط `cwd`.

النموذج الذهني:

- سطح الدردشة: المكان الذي يواصل فيه الناس الحديث (`Discord channel`، `Telegram topic`، `iMessage chat`)
- جلسة ACP: حالة وقت تشغيل Codex/Claude/Gemini الدائمة التي يوجّه إليها OpenClaw
- سلسلة/موضوع فرعي: سطح مراسلة إضافي اختياري يُنشأ فقط بواسطة `--thread ...`
- مساحة عمل وقت التشغيل: موقع نظام الملفات الذي تعمل فيه الحاضنة (`cwd`، أو مستودع تم سحبه، أو مساحة عمل للواجهة الخلفية)

أمثلة:

- `/acp spawn codex --bind here`: احتفظ بهذه الدردشة، وأنشئ أو أرفق جلسة Codex ACP، ووجّه الرسائل المستقبلية هنا إليها
- `/acp spawn codex --thread auto`: قد ينشئ OpenClaw سلسلة/موضوعًا فرعيًا ويربط جلسة ACP هناك
- `/acp spawn codex --bind here --cwd /workspace/repo`: نفس ربط الدردشة أعلاه، لكن Codex يعمل في `/workspace/repo`

دعم الربط بالمحادثة الحالية:

- يمكن لقنوات الدردشة/الرسائل التي تعلن دعم ربط المحادثة الحالية استخدام `--bind here` عبر مسار ربط المحادثة المشترك.
- يمكن للقنوات ذات دلالات السلاسل/الموضوعات المخصصة أن توفّر مع ذلك تسوية خاصة بالقناة خلف الواجهة المشتركة نفسها.
- يعني `--bind here` دائمًا "اربط المحادثة الحالية في مكانها".
- تستخدم الارتباطات العامة للمحادثة الحالية مخزن الربط المشترك في OpenClaw وتستمر عبر عمليات إعادة تشغيل البوابة العادية.

ملاحظات:

- `--bind here` و`--thread ...` متنافيان في `/acp spawn`.
- على Discord، يربط `--bind here` القناة الحالية أو السلسلة الحالية في مكانها. لا تكون `spawnAcpSessions` مطلوبة إلا عندما يحتاج OpenClaw إلى إنشاء سلسلة فرعية لـ `--thread auto|here`.
- إذا كانت القناة النشطة لا تعرض ارتباطات ACP للمحادثة الحالية، فسيعيد OpenClaw رسالة واضحة بعدم الدعم.
- أسئلة `resume` و"الجلسة الجديدة" هي أسئلة تخص جلسات ACP، وليست أسئلة تخص القنوات. يمكنك إعادة استخدام حالة وقت التشغيل أو استبدالها دون تغيير سطح الدردشة الحالي.

### الجلسات المرتبطة بسلسلة محادثة

عندما تكون ارتباطات السلاسل مفعّلة لمهايئ القناة، يمكن ربط جلسات ACP بسلاسل المحادثة:

- يربط OpenClaw سلسلة محادثة بجلسة ACP مستهدفة.
- تُوجَّه رسائل المتابعة في تلك السلسلة إلى جلسة ACP المرتبطة.
- يتم تسليم مخرجات ACP إلى سلسلة المحادثة نفسها.
- تؤدي إزالة التركيز أو الإغلاق أو الأرشفة أو انتهاء مهلة الخمول أو انتهاء الحد الأقصى للعمر إلى إزالة الارتباط.

يعتمد دعم ربط السلاسل على المهايئ. إذا لم يكن مهايئ القناة النشطة يدعم ارتباطات السلاسل، فسيعيد OpenClaw رسالة واضحة تفيد بعدم الدعم/التوفر.

علامات الميزات المطلوبة لـ ACP المرتبط بالسلسلة:

- `acp.enabled=true`
- يكون `acp.dispatch.enabled` مفعّلًا افتراضيًا (اضبطه على `false` لإيقاف توجيه ACP مؤقتًا)
- تفعيل علامة إنشاء سلاسل ACP في مهايئ القناة (خاصة بالمهايئ)
  - Discord: `channels.discord.threadBindings.spawnAcpSessions=true`
  - Telegram: `channels.telegram.threadBindings.spawnAcpSessions=true`

### القنوات التي تدعم السلاسل

- أي مهايئ قناة يكشف قدرة ربط الجلسة/السلسلة.
- الدعم المضمّن الحالي:
  - سلاسل/قنوات Discord
  - مواضيع Telegram (مواضيع المنتدى في المجموعات/المجموعات الفائقة ومواضيع الرسائل الخاصة)
- يمكن لقنوات plugin إضافة الدعم عبر واجهة الربط نفسها.

## الإعدادات الخاصة بالقنوات

للتدفقات غير المؤقتة، اضبط ارتباطات ACP المستمرة في إدخالات `bindings[]` ذات المستوى الأعلى.

### نموذج الربط

- تشير `bindings[].type="acp"` إلى ربط محادثة ACP مستمر.
- تحدد `bindings[].match` المحادثة المستهدفة:
  - قناة أو سلسلة Discord: `match.channel="discord"` + `match.peer.id="<channelOrThreadId>"`
  - موضوع منتدى Telegram: `match.channel="telegram"` + `match.peer.id="<chatId>:topic:<topicId>"`
  - دردشة BlueBubbles فردية/جماعية: `match.channel="bluebubbles"` + `match.peer.id="<handle|chat_id:*|chat_guid:*|chat_identifier:*>"`
    فضّل `chat_id:*` أو `chat_identifier:*` لارتباطات المجموعات المستقرة.
  - دردشة iMessage فردية/جماعية: `match.channel="imessage"` + `match.peer.id="<handle|chat_id:*|chat_guid:*|chat_identifier:*>"`
    فضّل `chat_id:*` لارتباطات المجموعات المستقرة.
- `bindings[].agentId` هو معرّف وكيل OpenClaw المالك.
- توجد تجاوزات ACP الاختيارية تحت `bindings[].acp`:
  - `mode` (`persistent` أو `oneshot`)
  - `label`
  - `cwd`
  - `backend`

### القيم الافتراضية لوقت التشغيل لكل وكيل

استخدم `agents.list[].runtime` لتعريف قيم ACP الافتراضية مرة واحدة لكل وكيل:

- `agents.list[].runtime.type="acp"`
- `agents.list[].runtime.acp.agent` (معرّف الحاضنة، مثل `codex` أو `claude`)
- `agents.list[].runtime.acp.backend`
- `agents.list[].runtime.acp.mode`
- `agents.list[].runtime.acp.cwd`

ترتيب أولوية التجاوز لجلسات ACP المرتبطة:

1. `bindings[].acp.*`
2. `agents.list[].runtime.acp.*`
3. القيم الافتراضية العامة لـ ACP (مثل `acp.backend`)

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

- يضمن OpenClaw وجود جلسة ACP المضبوطة قبل استخدامها.
- تُوجَّه الرسائل في تلك القناة أو الموضوع إلى جلسة ACP المضبوطة.
- في المحادثات المرتبطة، يعيد `/new` و`/reset` تعيين مفتاح جلسة ACP نفسه في مكانه.
- تستمر ارتباطات وقت التشغيل المؤقتة (مثل تلك التي تنشئها تدفقات التركيز على السلسلة) في التطبيق عند وجودها.
- في عمليات إنشاء ACP بين وكلاء مختلفين من دون `cwd` صريح، يرث OpenClaw مساحة عمل الوكيل المستهدف من إعدادات الوكيل.
- تعود مسارات مساحة العمل الموروثة المفقودة إلى `cwd` الافتراضي للواجهة الخلفية؛ أما حالات فشل الوصول غير المفقودة فتظهر كأخطاء إنشاء.

## بدء جلسات ACP (الواجهات)

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

- القيمة الافتراضية لـ `runtime` هي `subagent`، لذا اضبط `runtime: "acp"` صراحةً لجلسات ACP.
- إذا تم حذف `agentId`، يستخدم OpenClaw القيمة `acp.defaultAgent` عند ضبطها.
- يتطلب `mode: "session"` وجود `thread: true` للإبقاء على محادثة مرتبطة مستمرة.

تفاصيل الواجهة:

- `task` (مطلوب): الموجّه الأولي المرسل إلى جلسة ACP.
- `runtime` (مطلوب لـ ACP): يجب أن يكون `"acp"`.
- `agentId` (اختياري): معرّف حاضنة ACP المستهدفة. يعود إلى `acp.defaultAgent` إذا تم ضبطه.
- `thread` (اختياري، الافتراضي `false`): طلب تدفق ربط السلسلة عند دعمه.
- `mode` (اختياري): `run` (مرة واحدة) أو `session` (مستمر).
  - القيمة الافتراضية هي `run`
  - إذا كانت `thread: true` وتم حذف الوضع، فقد يستخدم OpenClaw سلوكًا مستمرًا افتراضيًا بحسب مسار وقت التشغيل
  - يتطلب `mode: "session"` وجود `thread: true`
- `cwd` (اختياري): دليل عمل وقت التشغيل المطلوب (ويتم التحقق منه وفق سياسة الواجهة الخلفية/وقت التشغيل). إذا تم حذفه، يرث إنشاء ACP مساحة عمل الوكيل المستهدف عند ضبطها؛ وتعود المسارات الموروثة المفقودة إلى القيم الافتراضية للواجهة الخلفية، بينما تُعاد أخطاء الوصول الحقيقية.
- `label` (اختياري): تسمية موجّهة للمشغّل تُستخدم في نص الجلسة/الشعار.
- `resumeSessionId` (اختياري): استئناف جلسة ACP موجودة بدلًا من إنشاء جلسة جديدة. يعيد الوكيل تشغيل سجل محادثته عبر `session/load`. يتطلب `runtime: "acp"`.
- `streamTo` (اختياري): تؤدي القيمة `"parent"` إلى بث ملخصات تقدم تشغيل ACP الأولي إلى جلسة الطالب كأحداث نظام.
  - عند التوفر، تتضمن الاستجابات المقبولة `streamLogPath` الذي يشير إلى سجل JSONL خاص بالجسلة (`<sessionId>.acp-stream.jsonl`) يمكنك مراقبته للاطلاع على سجل الترحيل الكامل.

### استئناف جلسة موجودة

استخدم `resumeSessionId` لمتابعة جلسة ACP سابقة بدلًا من البدء من جديد. يعيد الوكيل تشغيل سجل محادثته عبر `session/load`، بحيث يواصل العمل مع كامل السياق السابق.

```json
{
  "task": "Continue where we left off — fix the remaining test failures",
  "runtime": "acp",
  "agentId": "codex",
  "resumeSessionId": "<previous-session-id>"
}
```

حالات الاستخدام الشائعة:

- تسليم جلسة Codex من حاسوبك المحمول إلى هاتفك — اطلب من وكيلك متابعة العمل من حيث توقفت
- متابعة جلسة برمجة بدأت بها بشكل تفاعلي في CLI، ولكن الآن بدون واجهة مباشرة عبر وكيلك
- استكمال العمل الذي انقطع بسبب إعادة تشغيل البوابة أو انتهاء مهلة الخمول

ملاحظات:

- يتطلب `resumeSessionId` وجود `runtime: "acp"` — ويعيد خطأ إذا استُخدم مع وقت تشغيل الوكيل الفرعي.
- يعيد `resumeSessionId` سجل محادثة ACP الصاعد؛ ولا يزال `thread` و`mode` يُطبّقان بشكل طبيعي على جلسة OpenClaw الجديدة التي تنشئها، لذا يظل `mode: "session"` يتطلب `thread: true`.
- يجب أن يدعم الوكيل المستهدف `session/load` (ويدعمه كل من Codex وClaude Code).
- إذا لم يتم العثور على معرّف الجلسة، تفشل عملية الإنشاء مع خطأ واضح — دون عودة صامتة إلى جلسة جديدة.

### اختبار دخاني للمشغّل

استخدم هذا بعد نشر البوابة عندما تريد فحصًا حيًا سريعًا يثبت أن إنشاء ACP
يعمل فعلًا من طرف إلى طرف، وليس فقط أنه ينجح في اختبارات الوحدة.

البوابة الموصى بها:

1. تحقّق من إصدار/التزام البوابة المنشورة على المضيف المستهدف.
2. أكّد أن المصدر المنشور يتضمن قبول سلالة ACP في
   `src/gateway/sessions-patch.ts` (`subagent:* or acp:* sessions`).
3. افتح جلسة جسر ACPX مؤقتة إلى وكيل حي (مثل
   `razor(main)` على `jpclawhq`).
4. اطلب من ذلك الوكيل استدعاء `sessions_spawn` باستخدام:
   - `runtime: "acp"`
   - `agentId: "codex"`
   - `mode: "run"`
   - المهمة: `Reply with exactly LIVE-ACP-SPAWN-OK`
5. تحقّق من أن الوكيل يبلغ عن:
   - `accepted=yes`
   - `childSessionKey` حقيقي
   - عدم وجود خطأ تحقق
6. نظّف جلسة جسر ACPX المؤقتة.

مثال على موجّه إلى الوكيل الحي:

```text
Use the sessions_spawn tool now with runtime: "acp", agentId: "codex", and mode: "run".
Set the task to: "Reply with exactly LIVE-ACP-SPAWN-OK".
Then report only: accepted=<yes/no>; childSessionKey=<value or none>; error=<exact text or none>.
```

ملاحظات:

- أبقِ هذا الاختبار الدخاني على `mode: "run"` ما لم تكن تختبر عمدًا
  جلسات ACP المستمرة المرتبطة بالسلسلة.
- لا تشترط `streamTo: "parent"` للبوابة الأساسية. يعتمد هذا المسار على
  قدرات الطالب/الجلسة ويُعد فحص تكامل منفصل.
- تعامل مع اختبار `mode: "session"` المرتبط بالسلسلة باعتباره تمريرة تكامل
  ثانية وأكثر ثراءً من سلسلة Discord حقيقية أو موضوع Telegram حقيقي.

## توافق Sandbox

تعمل جلسات ACP حاليًا على وقت تشغيل المضيف، وليس داخل Sandbox الخاص بـ OpenClaw.

القيود الحالية:

- إذا كانت جلسة الطالب داخل sandbox، فسيتم حظر عمليات إنشاء ACP لكل من `sessions_spawn({ runtime: "acp" })` و`/acp spawn`.
  - الخطأ: `Sandboxed sessions cannot spawn ACP sessions because runtime="acp" runs on the host. Use runtime="subagent" from sandboxed sessions.`
- لا يدعم `sessions_spawn` مع `runtime: "acp"` القيمة `sandbox: "require"`.
  - الخطأ: `sessions_spawn sandbox="require" is unsupported for runtime="acp" because ACP sessions run outside the sandbox. Use runtime="subagent" or sandbox="inherit".`

استخدم `runtime: "subagent"` عندما تحتاج إلى تنفيذ مفروض بواسطة sandbox.

### من أمر `/acp`

استخدم `/acp spawn` للتحكم التشغيلي الصريح من الدردشة عند الحاجة.

```text
/acp spawn codex --mode persistent --thread auto
/acp spawn codex --mode oneshot --thread off
/acp spawn codex --bind here
/acp spawn codex --thread here
```

العلامات الرئيسية:

- `--mode persistent|oneshot`
- `--bind here|off`
- `--thread auto|here|off`
- `--cwd <absolute-path>`
- `--label <name>`

راجع [أوامر الشرطة المائلة](/ar/tools/slash-commands).

## حل هدف الجلسة

تقبل معظم إجراءات `/acp` هدف جلسة اختياريًا (`session-key` أو `session-id` أو `session-label`).

ترتيب الحل:

1. وسيطة الهدف الصريحة (أو `--session` في `/acp steer`)
   - يحاول أولًا المفتاح
   - ثم معرّف الجلسة على هيئة UUID
   - ثم التسمية
2. ارتباط السلسلة الحالية (إذا كانت هذه المحادثة/السلسلة مرتبطة بجلسة ACP)
3. الرجوع إلى جلسة الطالب الحالية

تشارك ارتباطات المحادثة الحالية وارتباطات السلاسل في كلتيهما في الخطوة 2.

إذا لم يتم حل أي هدف، يعيد OpenClaw خطأ واضحًا (`Unable to resolve session target: ...`).

## أوضاع ربط الإنشاء

يدعم `/acp spawn` الخيار `--bind here|off`.

| الوضع   | السلوك                                                               |
| ------ | -------------------------------------------------------------------- |
| `here` | اربط المحادثة النشطة الحالية في مكانها؛ وافشل إذا لم تكن هناك محادثة نشطة. |
| `off`  | لا تنشئ ارتباطًا للمحادثة الحالية.                          |

ملاحظات:

- `--bind here` هو أبسط مسار تشغيلي لعبارة "اجعل هذه القناة أو الدردشة مدعومة بواسطة Codex."
- لا ينشئ `--bind here` سلسلة فرعية.
- يتوفر `--bind here` فقط على القنوات التي تكشف دعم ربط المحادثة الحالية.
- لا يمكن الجمع بين `--bind` و`--thread` في استدعاء `/acp spawn` نفسه.

## أوضاع سلسلة الإنشاء

يدعم `/acp spawn` الخيار `--thread auto|here|off`.

| الوضع   | السلوك                                                                                            |
| ------ | ------------------------------------------------------------------------------------------------- |
| `auto` | في سلسلة نشطة: اربط تلك السلسلة. خارج سلسلة: أنشئ/اربط سلسلة فرعية عند الدعم. |
| `here` | اشترط وجود سلسلة نشطة حالية؛ وافشل إذا لم تكن داخل واحدة.                                                  |
| `off`  | بلا ربط. تبدأ الجلسة غير مرتبطة.                                                                 |

ملاحظات:

- على الأسطح التي لا تدعم ربط السلاسل، يكون السلوك الافتراضي فعليًا هو `off`.
- يتطلب إنشاء جلسة مرتبطة بسلسلة دعم سياسة القناة:
  - Discord: `channels.discord.threadBindings.spawnAcpSessions=true`
  - Telegram: `channels.telegram.threadBindings.spawnAcpSessions=true`
- استخدم `--bind here` عندما تريد تثبيت المحادثة الحالية دون إنشاء سلسلة فرعية.

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

يعرض `/acp status` خيارات وقت التشغيل الفعالة، وعند التوفر، كلًا من معرّفات الجلسة على مستوى وقت التشغيل ومستوى الواجهة الخلفية.

تعتمد بعض عناصر التحكم على قدرات الواجهة الخلفية. إذا كانت الواجهة الخلفية لا تدعم عنصر تحكم ما، فسيعيد OpenClaw خطأ واضحًا يفيد بعدم دعم عنصر التحكم.

## كتاب وصفات أوامر ACP

| الأمر              | ما الذي يفعله                                              | مثال                                                       |
| ------------------ | ---------------------------------------------------------- | ----------------------------------------------------------- |
| `/acp spawn`         | إنشاء جلسة ACP؛ مع ربط اختياري للمحادثة الحالية أو ربط بسلسلة. | `/acp spawn codex --bind here --cwd /repo`                    |
| `/acp cancel`        | إلغاء الدور الجاري للجلسة المستهدفة.                 | `/acp cancel agent:codex:acp:<uuid>`                          |
| `/acp steer`         | إرسال توجيه إلى جلسة قيد التشغيل.                | `/acp steer --session support inbox prioritize failing tests` |
| `/acp close`         | إغلاق الجلسة وفك ربط أهداف السلاسل.                  | `/acp close`                                                  |
| `/acp status`        | عرض الواجهة الخلفية والوضع والحالة وخيارات وقت التشغيل والقدرات. | `/acp status`                                                 |
| `/acp set-mode`      | تعيين وضع وقت التشغيل للجلسة المستهدفة.                      | `/acp set-mode plan`                                          |
| `/acp set`           | كتابة خيار إعداد عام لوقت التشغيل.                      | `/acp set model openai/gpt-5.4`                               |
| `/acp cwd`           | تعيين تجاوز دليل عمل وقت التشغيل.                   | `/acp cwd /Users/user/Projects/repo`                          |
| `/acp permissions`   | تعيين ملف سياسة الموافقة.                              | `/acp permissions strict`                                     |
| `/acp timeout`       | تعيين مهلة وقت التشغيل (بالثواني).                            | `/acp timeout 120`                                            |
| `/acp model`         | تعيين تجاوز نموذج وقت التشغيل.                               | `/acp model anthropic/claude-opus-4-6`                        |
| `/acp reset-options` | إزالة تجاوزات خيارات وقت التشغيل للجلسة.                  | `/acp reset-options`                                          |
| `/acp sessions`      | سرد جلسات ACP الحديثة من المخزن.                      | `/acp sessions`                                               |
| `/acp doctor`        | سلامة الواجهة الخلفية، والقدرات، والإصلاحات القابلة للتنفيذ.           | `/acp doctor`                                                 |
| `/acp install`       | طباعة خطوات تثبيت وتمكين حتمية.             | `/acp install`                                                |

يقرأ `/acp sessions` المخزن للجلسة المرتبطة الحالية أو جلسة الطالب الحالية. تحل الأوامر التي تقبل رموز `session-key` أو `session-id` أو `session-label` الأهداف عبر اكتشاف جلسات البوابة، بما في ذلك الجذور المخصصة لـ `session.store` لكل وكيل.

## ربط خيارات وقت التشغيل

تحتوي `/acp` على أوامر مريحة وأداة تعيين عامة.

العمليات المكافئة:

- يطابق `/acp model <id>` مفتاح إعداد وقت التشغيل `model`.
- يطابق `/acp permissions <profile>` مفتاح إعداد وقت التشغيل `approval_policy`.
- يطابق `/acp timeout <seconds>` مفتاح إعداد وقت التشغيل `timeout`.
- يحدّث `/acp cwd <path>` تجاوز `cwd` لوقت التشغيل مباشرةً.
- يشكّل `/acp set <key> <value>` المسار العام.
  - حالة خاصة: يستخدم `key=cwd` مسار تجاوز cwd.
- يزيل `/acp reset-options` جميع تجاوزات وقت التشغيل للجلسة المستهدفة.

## دعم حاضنات acpx (الحالي)

الأسماء المستعارة المضمّنة الحالية للحاضنات في acpx:

- `claude`
- `codex`
- `copilot`
- `cursor` (Cursor CLI: `cursor-agent acp`)
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

عندما يستخدم OpenClaw الواجهة الخلفية acpx، ففضّل هذه القيم لـ `agentId` ما لم يعرّف إعداد acpx لديك أسماء مستعارة مخصصة للوكلاء.
إذا كان تثبيت Cursor المحلي لديك لا يزال يعرض ACP باعتباره `agent acp`، فتجاوز أمر الوكيل `cursor` في إعداد acpx بدلًا من تغيير القيمة الافتراضية المضمّنة.

يمكن لاستخدام CLI المباشر لـ acpx أيضًا استهداف مهيئات اعتباطية عبر `--agent <command>`، لكن منفذ الهروب الخام هذا هو ميزة في CLI الخاص بـ acpx (وليس مسار `agentId` المعتاد في OpenClaw).

## الإعداد المطلوب

خط الأساس الأساسي لـ ACP:

```json5
{
  acp: {
    enabled: true,
    // اختياري. القيمة الافتراضية true؛ اضبطها على false لإيقاف توجيه ACP مؤقتًا مع الإبقاء على عناصر تحكم /acp.
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

إعداد ربط السلاسل خاص بمهايئ القناة. مثال على Discord:

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

إذا لم يعمل إنشاء ACP المرتبط بالسلسلة، فتحقق أولًا من علامة ميزة المهايئ:

- Discord: `channels.discord.threadBindings.spawnAcpSessions=true`

لا تتطلب ارتباطات المحادثة الحالية إنشاء سلسلة فرعية. وهي تتطلب سياق محادثة نشطًا ومهايئ قناة يكشف ارتباطات محادثة ACP.

راجع [مرجع الإعداد](/ar/gateway/configuration-reference).

## إعداد plugin للواجهة الخلفية acpx

تأتي التثبيتات الجديدة مع plugin وقت التشغيل المضمّن `acpx` مفعّلًا افتراضيًا، لذا فإن ACP
يعمل عادةً بدون خطوة تثبيت plugin يدوية.

ابدأ بـ:

```text
/acp doctor
```

إذا كنت قد عطلت `acpx` أو منعتَه عبر `plugins.allow` / `plugins.deny` أو كنت
تريد التبديل إلى نسخة تطوير محلية، فاستخدم مسار plugin الصريح:

```bash
openclaw plugins install acpx
openclaw config set plugins.entries.acpx.enabled true
```

التثبيت المحلي في مساحة العمل أثناء التطوير:

```bash
openclaw plugins install ./path/to/local/acpx-plugin
```

ثم تحقّق من سلامة الواجهة الخلفية:

```text
/acp doctor
```

### إعدادات الأمر والإصدار لـ acpx

افتراضيًا، يستخدم plugin الواجهة الخلفية المضمّن acpx (`acpx`) الثنائي المثبّت محليًا داخل plugin:

1. تكون القيمة الافتراضية للأمر هي `node_modules/.bin/acpx` المحلي داخل حزمة plugin ACPX.
2. تكون القيمة الافتراضية للإصدار المتوقع هي الإصدار المثبّت في extension.
3. يسجّل بدء التشغيل الواجهة الخلفية لـ ACP فورًا على أنها غير جاهزة.
4. تتحقق مهمة ضمان في الخلفية من `acpx --version`.
5. إذا كان الثنائي المحلي داخل plugin مفقودًا أو غير مطابق، يشغّل:
   `npm install --omit=dev --no-save acpx@<pinned>` ثم يعيد التحقق.

يمكنك تجاوز الأمر/الإصدار في إعداد plugin:

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

- يقبل `command` مسارًا مطلقًا أو مسارًا نسبيًا أو اسم أمر (`acpx`).
- تُحل المسارات النسبية انطلاقًا من دليل مساحة عمل OpenClaw.
- يؤدي `expectedVersion: "any"` إلى تعطيل المطابقة الصارمة للإصدار.
- عندما يشير `command` إلى ثنائي/مسار مخصص، يتم تعطيل التثبيت التلقائي المحلي داخل plugin.
- يظل بدء تشغيل OpenClaw غير معيق أثناء تنفيذ فحص سلامة الواجهة الخلفية.

راجع [Plugins](/ar/tools/plugin).

### التثبيت التلقائي للتبعيات

عند تثبيت OpenClaw عالميًا باستخدام `npm install -g openclaw`، يتم تثبيت
تبعيات وقت التشغيل الخاصة بـ acpx (الثنائيات الخاصة بالنظام الأساسي) تلقائيًا
عبر خطاف postinstall. إذا فشل التثبيت التلقائي، فستبدأ البوابة بشكل طبيعي
وستبلّغ عن التبعية المفقودة عبر `openclaw acp doctor`.

### جسر MCP لأدوات plugin

افتراضيًا، لا تكشف جلسات ACPX **عن** أدوات OpenClaw المسجلة بواسطة plugin
إلى حاضنة ACP.

إذا كنت تريد أن تتمكن وكلاء ACP مثل Codex أو Claude Code من استدعاء
أدوات OpenClaw المثبّتة بواسطة plugin مثل memory recall/store، فعِّل الجسر المخصص:

```bash
openclaw config set plugins.entries.acpx.config.pluginToolsMcpBridge true
```

ما الذي يفعله هذا:

- يحقن خادم MCP مضمّنًا باسم `openclaw-plugin-tools` في تهيئة جلسة ACPX.
- يكشف أدوات plugin المسجلة بالفعل بواسطة plugins OpenClaw المثبّتة والمفعّلة.
- يبقي الميزة صريحة ومُعطّلة افتراضيًا.

ملاحظات الأمان والثقة:

- يؤدي هذا إلى توسيع سطح الأدوات في حاضنة ACP.
- لا يحصل وكلاء ACP إلا على الوصول إلى أدوات plugin النشطة بالفعل في البوابة.
- تعامل مع هذا باعتباره الحد نفسه من الثقة الذي ينطبق على السماح لتلك plugins بالتنفيذ داخل OpenClaw نفسه.
- راجع plugins المثبّتة قبل تفعيله.

تظل `mcpServers` المخصصة تعمل كما كانت من قبل. جسر أدوات plugin المضمّن هو
وسيلة راحة إضافية اختيارية، وليس بديلًا عن إعداد خادم MCP العام.

### إعداد مهلة وقت التشغيل

يضبط plugin `acpx` المضمّن مهلة أدوار وقت التشغيل المضمّن افتراضيًا على 120
ثانية. يمنح هذا الحاضنات الأبطأ مثل Gemini CLI وقتًا كافيًا لإكمال
بدء تشغيل ACP والتهيئة. تجاوز هذا إذا كان مضيفك يحتاج إلى حد مختلف
لوقت التشغيل:

```bash
openclaw config set plugins.entries.acpx.config.timeoutSeconds 180
```

أعد تشغيل البوابة بعد تغيير هذه القيمة.

## إعدادات الأذونات

تعمل جلسات ACP بشكل غير تفاعلي — لا توجد TTY للموافقة على مطالبات أذونات كتابة الملفات وتنفيذ shell أو رفضها. يوفّر plugin ‏acpx مفتاحَي إعداد يتحكمان في كيفية التعامل مع الأذونات:

هذه الأذونات الخاصة بحاضنات ACPX منفصلة عن موافقات تنفيذ OpenClaw ومنفصلة عن علامات تجاوز المورّد في الواجهات الخلفية لـ CLI مثل Claude CLI `--permission-mode bypassPermissions`. وتمثل ACPX ‏`approve-all` مفتاح الكسر الطارئ على مستوى الحاضنة لجلسات ACP.

### `permissionMode`

يتحكم في العمليات التي يمكن لوكيل الحاضنة تنفيذها دون مطالبة.

| القيمة           | السلوك                                                  |
| ---------------- | ------------------------------------------------------- |
| `approve-all`   | الموافقة التلقائية على جميع عمليات كتابة الملفات وأوامر shell.          |
| `approve-reads` | الموافقة التلقائية على القراءات فقط؛ تتطلب الكتابة والتنفيذ مطالبات. |
| `deny-all`      | رفض جميع مطالبات الأذونات.                              |

### `nonInteractivePermissions`

يتحكم في ما يحدث عندما كان من المفترض عرض مطالبة إذن ولكن لا توجد TTY تفاعلية متاحة (وهو الحال دائمًا في جلسات ACP).

| القيمة  | السلوك                                                          |
| ------ | ---------------------------------------------------------------- |
| `fail` | إيقاف الجلسة مع `AcpRuntimeError`. **(الافتراضي)**           |
| `deny` | رفض الإذن بصمت والمتابعة (تدهور سلس). |

### الإعداد

اضبطه عبر إعداد plugin:

```bash
openclaw config set plugins.entries.acpx.config.permissionMode approve-all
openclaw config set plugins.entries.acpx.config.nonInteractivePermissions fail
```

أعد تشغيل البوابة بعد تغيير هذه القيم.

> **مهم:** يستخدم OpenClaw حاليًا افتراضيًا `permissionMode=approve-reads` و`nonInteractivePermissions=fail`. في جلسات ACP غير التفاعلية، قد تفشل أي عملية كتابة أو تنفيذ تؤدي إلى مطالبة إذن مع `AcpRuntimeError: Permission prompt unavailable in non-interactive mode`.
>
> إذا كنت بحاجة إلى تقييد الأذونات، فاضبط `nonInteractivePermissions` على `deny` حتى تتدهور الجلسات بسلاسة بدلًا من أن تتعطل.

## استكشاف الأخطاء وإصلاحها

| العَرَض                                                                     | السبب المرجّح                                                                    | الإصلاح                                                                                                                                                               |
| --------------------------------------------------------------------------- | ------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ACP runtime backend is not configured`                                     | plugin الواجهة الخلفية مفقود أو معطّل.                                             | ثبّت plugin الواجهة الخلفية وفعّله، ثم شغّل `/acp doctor`.                                                                                                        |
| `ACP is disabled by policy (acp.enabled=false)`                             | تم تعطيل ACP عالميًا.                                                          | اضبط `acp.enabled=true`.                                                                                                                                           |
| `ACP dispatch is disabled by policy (acp.dispatch.enabled=false)`           | تم تعطيل التوجيه من رسائل السلاسل العادية.                                  | اضبط `acp.dispatch.enabled=true`.                                                                                                                                  |
| `ACP agent "<id>" is not allowed by policy`                                 | الوكيل ليس ضمن قائمة السماح.                                                         | استخدم `agentId` مسموحًا به أو حدّث `acp.allowedAgents`.                                                                                                              |
| `Unable to resolve session target: ...`                                     | رمز مفتاح/معرّف/تسمية غير صحيح.                                                         | شغّل `/acp sessions`، وانسخ المفتاح/التسمية بدقة، ثم أعد المحاولة.                                                                                                                 |
| `--bind here requires running /acp spawn inside an active ... conversation` | استُخدم `--bind here` بدون وجود محادثة نشطة قابلة للربط.                     | انتقل إلى الدردشة/القناة المستهدفة وأعد المحاولة، أو استخدم إنشاءً غير مرتبط.                                                                                                  |
| `Conversation bindings are unavailable for <channel>.`                      | يفتقر المهايئ إلى قدرة ربط ACP بالمحادثة الحالية.                      | استخدم `/acp spawn ... --thread ...` عند الدعم، أو اضبط `bindings[]` ذات المستوى الأعلى، أو انتقل إلى قناة مدعومة.                                              |
| `--thread here requires running /acp spawn inside an active ... thread`     | استُخدم `--thread here` خارج سياق سلسلة.                                  | انتقل إلى السلسلة المستهدفة أو استخدم `--thread auto`/`off`.                                                                                                               |
| `Only <user-id> can rebind this channel/conversation/thread.`               | يملك مستخدم آخر هدف الربط النشط.                                    | أعد الربط بصفتك المالك أو استخدم محادثة أو سلسلة أخرى.                                                                                                        |
| `Thread bindings are unavailable for <channel>.`                            | يفتقر المهايئ إلى قدرة ربط السلاسل.                                        | استخدم `--thread off` أو انتقل إلى مهايئ/قناة مدعومين.                                                                                                          |
| `Sandboxed sessions cannot spawn ACP sessions ...`                          | وقت تشغيل ACP يعمل على المضيف؛ وجلسة الطالب داخل sandbox.                       | استخدم `runtime="subagent"` من الجلسات داخل sandbox، أو شغّل إنشاء ACP من جلسة خارج sandbox.                                                                  |
| `sessions_spawn sandbox="require" is unsupported for runtime="acp" ...`     | طُلِب `sandbox="require"` لوقت تشغيل ACP.                                  | استخدم `runtime="subagent"` إذا كان العزل الإلزامي مطلوبًا، أو استخدم ACP مع `sandbox="inherit"` من جلسة خارج sandbox.                                               |
| Missing ACP metadata for bound session                                      | بيانات تعريف ACP قديمة/محذوفة للجلسة المرتبطة.                                             | أعد الإنشاء باستخدام `/acp spawn`، ثم أعد الربط/التركيز على السلسلة.                                                                                                             |
| `AcpRuntimeError: Permission prompt unavailable in non-interactive mode`    | يمنع `permissionMode` عمليات الكتابة/التنفيذ في جلسة ACP غير تفاعلية.             | اضبط `plugins.entries.acpx.config.permissionMode` على `approve-all` وأعد تشغيل البوابة. راجع [إعدادات الأذونات](#permission-configuration).                 |
| ACP session fails early with little output                                  | يتم حظر مطالبات الأذونات بواسطة `permissionMode`/`nonInteractivePermissions`. | تحقّق من سجلات البوابة بحثًا عن `AcpRuntimeError`. للحصول على أذونات كاملة، اضبط `permissionMode=approve-all`؛ وللتدهور السلس، اضبط `nonInteractivePermissions=deny`. |
| ACP session stalls indefinitely after completing work                       | انتهت عملية الحاضنة لكن جلسة ACP لم تُبلغ عن الاكتمال.             | راقب باستخدام `ps aux \| grep acpx`; واقتل العمليات العالقة يدويًا.                                                                                                |
