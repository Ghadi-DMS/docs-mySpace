---
read_when:
    - تشغيل coding harnesses عبر ACP
    - إعداد جلسات ACP المرتبطة بالمحادثات على قنوات المراسلة
    - ربط محادثة في قناة رسائل بجلسة ACP دائمة
    - استكشاف أخطاء ACP backend وتوصيلات plugin وإصلاحها
    - تشغيل أوامر `/acp` من الدردشة
summary: استخدام جلسات runtime الخاصة بـ ACP لـ Codex وClaude Code وCursor وGemini CLI وOpenClaw ACP ووكلاء harness الآخرين
title: وكلاء ACP
x-i18n:
    generated_at: "2026-04-06T03:14:35Z"
    model: gpt-5.4
    provider: openai
    source_hash: 302f3fe25b1ffe0576592b6e0ad9e8a5781fa5702b31d508d9ba8908f7df33bd
    source_path: tools/acp-agents.md
    workflow: 15
---

# وكلاء ACP

تتيح جلسات [Agent Client Protocol (ACP)](https://agentclientprotocol.com/) لـ OpenClaw تشغيل coding harnesses خارجية (على سبيل المثال Pi، وClaude Code، وCodex، وCursor، وCopilot، وOpenClaw ACP، وOpenCode، وGemini CLI، وharnesses أخرى مدعومة من ACPX) من خلال plugin خاص بـ ACP backend.

إذا طلبت من OpenClaw بلغة طبيعية "شغّل هذا في Codex" أو "ابدأ Claude Code في سلسلة"، فيجب على OpenClaw توجيه هذا الطلب إلى runtime الخاص بـ ACP (وليس إلى runtime الأصلي للوكلاء الفرعيين). ويجري تتبع كل عملية spawn لجلسة ACP بوصفها [مهمة خلفية](/ar/automation/tasks).

إذا كنت تريد أن يتصل Codex أو Claude Code مباشرة بصفته عميل MCP خارجيًا
بمحادثات قنوات OpenClaw الحالية، فاستخدم [`openclaw mcp serve`](/cli/mcp)
بدلًا من ACP.

## أي صفحة أريد؟

توجد ثلاث واجهات متجاورة يسهل الخلط بينها:

| تريد أن... | استخدم هذا | ملاحظات |
| ---------------------------------------------------------------------------------- | -------------------------- | ----------------------------------------------------------------------------------------------------------- |
| تشغيل Codex أو Claude Code أو Gemini CLI أو harness خارجي آخر _عبر_ OpenClaw | هذه الصفحة: وكلاء ACP | جلسات مرتبطة بالدردشة، و`/acp spawn`، و`sessions_spawn({ runtime: "acp" })`، ومهام الخلفية، وعناصر تحكم runtime |
| عرض جلسة OpenClaw Gateway _كخادم_ ACP لمحرر أو عميل | [`openclaw acp`](/cli/acp) | وضع الجسر. يتحدث IDE/العميل مع OpenClaw عبر ACP من خلال stdio/WebSocket |

## هل يعمل هذا مباشرة؟

عادة نعم.

- تأتي عمليات التثبيت الجديدة الآن مع plugin runtime المضمّن `acpx` مفعّلًا افتراضيًا.
- يفضل plugin المضمّن `acpx` الملف التنفيذي المثبت محليًا الخاص بالـ plugin.
- عند بدء التشغيل، يفحص OpenClaw ذلك الملف التنفيذي ويصلحه ذاتيًا عند الحاجة.
- ابدأ بـ `/acp doctor` إذا كنت تريد فحصًا سريعًا للجاهزية.

ما الذي قد يحدث مع أول استخدام:

- قد يتم جلب target harness adapter عند الطلب باستخدام `npx` في أول مرة تستخدم فيها ذلك harness.
- يجب أن تظل مصادقة المورّد موجودة على المضيف لذلك harness.
- إذا لم يكن لدى المضيف وصول إلى npm/الشبكة، فقد تفشل عمليات الجلب الأولى للمحوّل حتى يتم تدفئة caches مسبقًا أو تثبيت المحول بطريقة أخرى.

أمثلة:

- `/acp spawn codex`: يجب أن يكون OpenClaw جاهزًا لتهيئة `acpx`، ولكن قد يحتاج Codex ACP adapter إلى جلب أولي.
- `/acp spawn claude`: الأمر نفسه ينطبق على Claude ACP adapter، بالإضافة إلى مصادقة Claude على ذلك المضيف.

## تدفق التشغيل السريع

استخدم هذا عندما تريد دليل تشغيل عمليًا لأوامر `/acp`:

1. أنشئ جلسة:
   - `/acp spawn codex --bind here`
   - `/acp spawn codex --mode persistent --thread auto`
2. اعمل في المحادثة أو السلسلة المرتبطة (أو استهدف مفتاح الجلسة هذا صراحةً).
3. تحقق من حالة runtime:
   - `/acp status`
4. اضبط خيارات runtime حسب الحاجة:
   - `/acp model <provider/model>`
   - `/acp permissions <profile>`
   - `/acp timeout <seconds>`
5. وجّه جلسة نشطة من دون استبدال السياق:
   - `/acp steer tighten logging and continue`
6. أوقف العمل:
   - `/acp cancel` (إيقاف الدور الحالي)، أو
   - `/acp close` (إغلاق الجلسة + إزالة الروابط)

## البدء السريع للبشر

أمثلة على الطلبات الطبيعية:

- "اربط قناة Discord هذه بـ Codex."
- "ابدأ جلسة Codex دائمة في سلسلة هنا واحتفظ بتركيزها."
- "شغّل هذا كجلسة Claude Code ACP لمرة واحدة ثم لخّص النتيجة."
- "اربط محادثة iMessage هذه بـ Codex واحتفظ بالمتابعات في مساحة العمل نفسها."
- "استخدم Gemini CLI لهذه المهمة في سلسلة، ثم احتفظ بالمتابعات في السلسلة نفسها."

ما الذي ينبغي أن يفعله OpenClaw:

1. اختيار `runtime: "acp"`.
2. حل target harness المطلوب (`agentId`، مثل `codex`).
3. إذا طُلِب ربط المحادثة الحالية وكان channel النشط يدعمه، فربط جلسة ACP بتلك المحادثة.
4. بخلاف ذلك، إذا طُلِب ربط سلسلة وكان channel الحالي يدعمه، فربط جلسة ACP بتلك السلسلة.
5. توجيه الرسائل المرتبطة اللاحقة إلى جلسة ACP نفسها حتى يتم إلغاء التركيز/الإغلاق/الانتهاء.

## ACP مقابل الوكلاء الفرعيين

استخدم ACP عندما تريد runtime خارجيًا لـ harness. واستخدم الوكلاء الفرعيين عندما تريد تشغيلات مفوضة أصلية خاصة بـ OpenClaw.

| المجال | جلسة ACP | تشغيل وكيل فرعي |
| ------------- | ------------------------------------- | ---------------------------------- |
| Runtime | plugin خاص بـ ACP backend (مثل acpx) | runtime أصلي للوكلاء الفرعيين في OpenClaw |
| مفتاح الجلسة | `agent:<agentId>:acp:<uuid>` | `agent:<agentId>:subagent:<uuid>` |
| الأوامر الرئيسية | `/acp ...` | `/subagents ...` |
| أداة الإنشاء | `sessions_spawn` مع `runtime:"acp"` | `sessions_spawn` (runtime افتراضي) |

راجع أيضًا [الوكلاء الفرعيون](/ar/tools/subagents).

## كيف يشغّل ACP أداة Claude Code

بالنسبة إلى Claude Code عبر ACP، تكون البنية كالتالي:

1. مستوى التحكم في جلسة OpenClaw ACP
2. plugin runtime المضمّن `acpx`
3. Claude ACP adapter
4. آليات runtime/session في جانب Claude

تمييز مهم:

- Claude عبر ACP هو جلسة harness مع عناصر تحكم ACP، واستئناف الجلسات، وتتبع مهام الخلفية، وربط اختياري بالمحادثة/السلسلة.
  وبالنسبة إلى المشغلين، فالقاعدة العملية هي:

- إذا كنت تريد `/acp spawn`، أو جلسات قابلة للربط، أو عناصر تحكم runtime، أو عمل harness دائمًا: فاستخدم ACP

## الجلسات المرتبطة

### الربط بالمحادثة الحالية

استخدم `/acp spawn <harness> --bind here` عندما تريد أن تصبح المحادثة الحالية مساحة عمل ACP دائمة من دون إنشاء سلسلة فرعية.

السلوك:

- يحتفظ OpenClaw بملكية نقل القناة، والمصادقة، والسلامة، والتسليم.
- تُثبَّت المحادثة الحالية على مفتاح جلسة ACP الذي تم إنشاؤه.
- تُوجَّه الرسائل اللاحقة في تلك المحادثة إلى جلسة ACP نفسها.
- يقوم `/new` و`/reset` بإعادة ضبط جلسة ACP المرتبطة نفسها في مكانها.
- يغلق `/acp close` الجلسة ويزيل ربط المحادثة الحالية.

ما الذي يعنيه هذا عمليًا:

- يحتفظ `--bind here` بسطح الدردشة نفسه. على Discord، تبقى القناة الحالية هي نفسها.
- يمكن لـ `--bind here` مع ذلك إنشاء جلسة ACP جديدة إذا كنت تنشئ عملًا جديدًا.
- لا ينشئ `--bind here` سلسلة Discord فرعية أو موضوع Telegram بحد ذاته.
- لا يزال runtime الخاص بـ ACP قادرًا على امتلاك دليل عمل خاص به (`cwd`) أو مساحة عمل يديرها backend على القرص. وتبقى مساحة عمل runtime هذه منفصلة عن سطح الدردشة ولا تعني ضمنيًا سلسلة رسائل جديدة.
- إذا قمت بعملية spawn إلى ACP agent مختلف ولم تمرر `--cwd`، فإن OpenClaw يرث مساحة عمل **الوكيل الهدف** افتراضيًا، وليس مساحة عمل الطالب.
- إذا كان مسار مساحة العمل الموروث هذا مفقودًا (`ENOENT`/`ENOTDIR`)، فإن OpenClaw يعود إلى cwd الافتراضي الخاص بـ backend بدلًا من إعادة استخدام الشجرة الخاطئة بصمت.
- إذا كانت مساحة العمل الموروثة موجودة ولكن لا يمكن الوصول إليها (مثل `EACCES`)، فإن spawn يعيد خطأ الوصول الحقيقي بدلًا من إسقاط `cwd`.

النموذج الذهني:

- سطح الدردشة: المكان الذي يواصل الناس الحديث فيه (`Discord channel`، `Telegram topic`، `iMessage chat`)
- جلسة ACP: حالة runtime الدائمة لـ Codex/Claude/Gemini التي يوجه إليها OpenClaw
- سلسلة/موضوع فرعي: سطح رسائل إضافي اختياري يتم إنشاؤه فقط بواسطة `--thread ...`
- مساحة عمل runtime: موقع نظام الملفات الذي يعمل فيه harness (`cwd`، ونسخة المستودع، ومساحة عمل backend)

أمثلة:

- `/acp spawn codex --bind here`: احتفظ بهذه الدردشة، وأنشئ أو أرفق جلسة Codex ACP، ووجّه الرسائل المستقبلية هنا إليها
- `/acp spawn codex --thread auto`: قد ينشئ OpenClaw سلسلة/موضوعًا فرعيًا ويربط جلسة ACP هناك
- `/acp spawn codex --bind here --cwd /workspace/repo`: الربط بالمحادثة نفسه كما أعلاه، لكن Codex يعمل في `/workspace/repo`

دعم الربط بالمحادثة الحالية:

- يمكن لقنوات الدردشة/الرسائل التي تعلن دعم الربط بالمحادثة الحالية استخدام `--bind here` عبر مسار الربط المشترك للمحادثات.
- لا تزال القنوات ذات دلالات السلاسل/الموضوعات المخصصة قادرة على توفير canonicalization خاص بالقناة خلف الواجهة المشتركة نفسها.
- يعني `--bind here` دائمًا "اربط المحادثة الحالية في مكانها".
- تستخدم عمليات الربط العامة للمحادثة الحالية مخزن الربط المشترك في OpenClaw وتنجو من عمليات إعادة تشغيل gateway العادية.

ملاحظات:

- `--bind here` و`--thread ...` متنافيان في `/acp spawn`.
- على Discord، يربط `--bind here` القناة أو السلسلة الحالية في مكانها. ولا يصبح `spawnAcpSessions` مطلوبًا إلا عندما يحتاج OpenClaw إلى إنشاء سلسلة فرعية من أجل `--thread auto|here`.
- إذا لم يكشف channel النشط عن روابط ACP للمحادثة الحالية، فإن OpenClaw يعيد رسالة واضحة تفيد بعدم الدعم.
- أسئلة `resume` و"جلسة جديدة" هي أسئلة تخص جلسة ACP، وليست أسئلة تخص القناة. يمكنك إعادة استخدام حالة runtime أو استبدالها من دون تغيير سطح الدردشة الحالي.

### الجلسات المرتبطة بالسلاسل

عندما تكون روابط السلاسل مفعلة لمحول channel، يمكن ربط جلسات ACP بالسلاسل:

- يربط OpenClaw سلسلة بجلسة ACP مستهدفة.
- تُوجَّه الرسائل اللاحقة في تلك السلسلة إلى جلسة ACP المرتبطة.
- يُسلَّم خرج ACP إلى السلسلة نفسها.
- تؤدي حالات unfocus/close/archive/idle-timeout أو انتهاء max-age إلى إزالة الربط.

دعم الربط بالسلاسل خاص بكل محول. وإذا كان محول channel النشط لا يدعم روابط السلاسل، فسيعيد OpenClaw رسالة واضحة تفيد بعدم الدعم/التوفر.

علامات الميزات المطلوبة لـ ACP المرتبط بالسلاسل:

- `acp.enabled=true`
- تكون `acp.dispatch.enabled` مفعلة افتراضيًا (عيّنها إلى `false` لإيقاف ACP dispatch مؤقتًا)
- علامة إنشاء سلاسل ACP في محول القناة مفعلة (خاصة بكل محول)
  - Discord: `channels.discord.threadBindings.spawnAcpSessions=true`
  - Telegram: `channels.telegram.threadBindings.spawnAcpSessions=true`

### القنوات التي تدعم السلاسل

- أي محول قناة يكشف عن إمكانية ربط الجلسة/السلسلة.
- الدعم المضمن الحالي:
  - سلاسل/قنوات Discord
  - موضوعات Telegram (موضوعات المنتدى في المجموعات/المجموعات الفائقة وموضوعات الرسائل الخاصة)
- يمكن لقنوات plugin إضافة الدعم عبر واجهة الربط نفسها.

## إعدادات خاصة بالقنوات

بالنسبة إلى سير العمل غير المؤقت، قم بتهيئة روابط ACP الدائمة في إدخالات `bindings[]` ذات المستوى الأعلى.

### نموذج الربط

- تحدد `bindings[].type="acp"` رابط محادثة ACP دائمًا.
- تحدد `bindings[].match` المحادثة المستهدفة:
  - قناة أو سلسلة Discord: ‏`match.channel="discord"` + ‏`match.peer.id="<channelOrThreadId>"`
  - موضوع منتدى Telegram: ‏`match.channel="telegram"` + ‏`match.peer.id="<chatId>:topic:<topicId>"`
  - دردشة خاصة/جماعية في BlueBubbles: ‏`match.channel="bluebubbles"` + ‏`match.peer.id="<handle|chat_id:*|chat_guid:*|chat_identifier:*>"`
    يفضّل استخدام `chat_id:*` أو `chat_identifier:*` للروابط الجماعية المستقرة.
  - دردشة خاصة/جماعية في iMessage: ‏`match.channel="imessage"` + ‏`match.peer.id="<handle|chat_id:*|chat_guid:*|chat_identifier:*>"`
    يفضّل استخدام `chat_id:*` للروابط الجماعية المستقرة.
- تمثل `bindings[].agentId` معرّف وكيل OpenClaw المالك.
- توجد تجاوزات ACP الاختيارية تحت `bindings[].acp`:
  - `mode` ‏(`persistent` أو `oneshot`)
  - `label`
  - `cwd`
  - `backend`

### افتراضيات runtime لكل وكيل

استخدم `agents.list[].runtime` لتعريف افتراضيات ACP مرة واحدة لكل وكيل:

- `agents.list[].runtime.type="acp"`
- `agents.list[].runtime.acp.agent` (معرّف harness، مثل `codex` أو `claude`)
- `agents.list[].runtime.acp.backend`
- `agents.list[].runtime.acp.mode`
- `agents.list[].runtime.acp.cwd`

ترتيب أولوية التجاوز للجلسات المرتبطة بـ ACP:

1. ‏`bindings[].acp.*`
2. ‏`agents.list[].runtime.acp.*`
3. افتراضيات ACP العامة (مثل `acp.backend`)

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

- يضمن OpenClaw وجود جلسة ACP المهيأة قبل الاستخدام.
- تُوجَّه الرسائل في تلك القناة أو الموضوع إلى جلسة ACP المهيأة.
- في المحادثات المرتبطة، يقوم `/new` و`/reset` بإعادة ضبط مفتاح جلسة ACP نفسه في مكانه.
- لا تزال روابط runtime المؤقتة (مثل التي تُنشأ بواسطة تدفقات thread-focus) تُطبق عند وجودها.
- بالنسبة إلى عمليات spawn المتقاطعة بين الوكلاء من دون `cwd` صريح، فإن OpenClaw يرث مساحة عمل الوكيل الهدف من إعدادات الوكيل.
- تعود مسارات مساحة العمل الموروثة المفقودة إلى cwd الافتراضي الخاص بـ backend؛ أما حالات فشل الوصول الحقيقية غير المفقودة فتظهر كأخطاء spawn.

## بدء جلسات ACP (واجهات)

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

- القيمة الافتراضية لـ `runtime` هي `subagent`، لذا عيّن `runtime: "acp"` صراحةً لجلسات ACP.
- إذا لم يتم تعيين `agentId`، فإن OpenClaw يستخدم `acp.defaultAgent` عند تهيئته.
- يتطلب `mode: "session"` وجود `thread: true` للاحتفاظ بمحادثة دائمة مرتبطة.

تفاصيل الواجهة:

- `task` (مطلوب): المطالبة الأولية المرسلة إلى جلسة ACP.
- `runtime` (مطلوب لـ ACP): يجب أن تكون `"acp"`.
- `agentId` (اختياري): معرّف target harness في ACP. ويعود إلى `acp.defaultAgent` إذا كان مضبوطًا.
- `thread` (اختياري، الافتراضي `false`): طلب تدفق الربط بالسلسلة عند الدعم.
- `mode` (اختياري): ‏`run` (لمرة واحدة) أو `session` (دائم).
  - القيمة الافتراضية هي `run`
  - إذا كانت `thread: true` ولم يتم تحديد الوضع، فقد يختار OpenClaw السلوك الدائم افتراضيًا وفقًا لمسار runtime
  - يتطلب `mode: "session"` وجود `thread: true`
- `cwd` (اختياري): دليل العمل المطلوب في runtime (يتم التحقق منه عبر سياسة backend/runtime). وإذا لم يتم تعيينه، فإن spawn الخاص بـ ACP يرث مساحة عمل الوكيل الهدف عند تهيئتها؛ وتعود المسارات الموروثة المفقودة إلى افتراضيات backend، بينما يتم إرجاع أخطاء الوصول الحقيقية.
- `label` (اختياري): تسمية موجهة للمشغل تُستخدم في نص الجلسة/اللافتة.
- `resumeSessionId` (اختياري): استئناف جلسة ACP موجودة بدلًا من إنشاء جلسة جديدة. يعيد الوكيل تشغيل سجل محادثته عبر `session/load`. ويتطلب `runtime: "acp"`.
- `streamTo` (اختياري): ‏`"parent"` يقوم ببث ملخصات تقدم تشغيل ACP الأولي إلى جلسة الطالب كأحداث نظام.
  - عند توفره، تتضمن الردود المقبولة `streamLogPath` الذي يشير إلى سجل JSONL على مستوى الجلسة (`<sessionId>.acp-stream.jsonl`) يمكنك تتبعه للحصول على السجل الكامل للترحيل.

### استئناف جلسة موجودة

استخدم `resumeSessionId` لمتابعة جلسة ACP سابقة بدلًا من البدء من جديد. يعيد الوكيل تشغيل سجل محادثته عبر `session/load`، بحيث يكمل مع كامل السياق السابق.

```json
{
  "task": "Continue where we left off — fix the remaining test failures",
  "runtime": "acp",
  "agentId": "codex",
  "resumeSessionId": "<previous-session-id>"
}
```

حالات الاستخدام الشائعة:

- تسليم جلسة Codex من حاسوبك المحمول إلى هاتفك — أخبر وكيلك أن يتابع من حيث توقفت
- متابعة جلسة برمجة بدأتها تفاعليًا في CLI، والآن بدون واجهة مباشرة عبر وكيلك
- متابعة العمل الذي انقطع بسبب إعادة تشغيل gateway أو انتهاء مهلة الخمول

ملاحظات:

- يتطلب `resumeSessionId` وجود `runtime: "acp"` — ويعيد خطأ إذا استُخدم مع runtime الخاص بالوكيل الفرعي.
- يعيد `resumeSessionId` سجل المحادثة الأصلي في upstream ACP؛ ولا تزال `thread` و`mode` تُطبَّقان بشكل طبيعي على جلسة OpenClaw الجديدة التي تنشئها، لذا لا يزال `mode: "session"` يتطلب `thread: true`.
- يجب أن يدعم الوكيل الهدف `session/load` (ويدعمه كل من Codex وClaude Code).
- إذا لم يُعثر على معرّف الجلسة، تفشل عملية spawn بخطأ واضح — من دون fallback صامت إلى جلسة جديدة.

### اختبار فحص للمشغل

استخدم هذا بعد نشر gateway عندما تريد فحصًا حيًا سريعًا يؤكد أن ACP spawn
يعمل فعلًا من طرف إلى طرف، وليس فقط أنه ينجح في اختبارات unit.

البوابة الموصى بها:

1. تحقق من إصدار/commit الخاص بـ gateway المنشور على المضيف الهدف.
2. أكد أن المصدر المنشور يتضمن قبول lineage الخاصة بـ ACP في
   `src/gateway/sessions-patch.ts` (`subagent:* or acp:* sessions`).
3. افتح جلسة جسر ACPX مؤقتة لوكيل حي (مثل
   `razor(main)` على `jpclawhq`).
4. اطلب من ذلك الوكيل استدعاء `sessions_spawn` باستخدام:
   - `runtime: "acp"`
   - `agentId: "codex"`
   - `mode: "run"`
   - task: ‏`Reply with exactly LIVE-ACP-SPAWN-OK`
5. تحقق من أن الوكيل يبلّغ:
   - `accepted=yes`
   - ‏`childSessionKey` حقيقي
   - عدم وجود خطأ validator
6. نظف جلسة جسر ACPX المؤقتة.

مثال على prompt إلى الوكيل الحي:

```text
Use the sessions_spawn tool now with runtime: "acp", agentId: "codex", and mode: "run".
Set the task to: "Reply with exactly LIVE-ACP-SPAWN-OK".
Then report only: accepted=<yes/no>; childSessionKey=<value or none>; error=<exact text or none>.
```

ملاحظات:

- أبقِ اختبار الفحص هذا على `mode: "run"` ما لم تكن تختبر عمدًا
  جلسات ACP دائمة مرتبطة بسلاسل.
- لا تشترط `streamTo: "parent"` للبوابة الأساسية. فهذا المسار يعتمد على
  قدرات الطالب/الجلسة وهو فحص تكاملي منفصل.
- تعامل مع اختبار `mode: "session"` المرتبط بالسلسلة باعتباره مرور تكامل
  ثانيًا وأكثر غنى من سلسلة Discord حقيقية أو موضوع Telegram حقيقي.

## توافق Sandbox

تعمل جلسات ACP حاليًا على runtime الخاص بالمضيف، وليس داخل sandbox الخاص بـ OpenClaw.

القيود الحالية:

- إذا كانت جلسة الطالب داخل sandbox، فسيتم حظر عمليات spawn الخاصة بـ ACP لكل من `sessions_spawn({ runtime: "acp" })` و`/acp spawn`.
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

## حل target الجلسة

تقبل معظم إجراءات `/acp` هدف جلسة اختياريًا (`session-key` أو `session-id` أو `session-label`).

ترتيب الحل:

1. وسيطة target الصريحة (أو `--session` مع `/acp steer`)
   - يحاول المفتاح
   - ثم معرّف الجلسة بشكل UUID
   - ثم التسمية
2. رابط السلسلة الحالية (إذا كانت هذه المحادثة/السلسلة مرتبطة بجلسة ACP)
3. fallback إلى جلسة الطالب الحالية

تشارك روابط المحادثة الحالية وروابط السلاسل في الخطوة 2.

إذا لم يتم حل أي target، يعيد OpenClaw خطأ واضحًا (`Unable to resolve session target: ...`).

## أوضاع ربط spawn

يدعم `/acp spawn` الخيار `--bind here|off`.

| الوضع | السلوك |
| ------ | ---------------------------------------------------------------------- |
| `here` | يربط المحادثة النشطة الحالية في مكانها؛ ويفشل إذا لم توجد محادثة نشطة. |
| `off`  | لا ينشئ رابطًا للمحادثة الحالية. |

ملاحظات:

- يمثل `--bind here` أبسط مسار تشغيلي لعبارة "اجعل هذه القناة أو الدردشة مدعومة بـ Codex."
- لا ينشئ `--bind here` سلسلة فرعية.
- لا يتوفر `--bind here` إلا على القنوات التي تكشف عن دعم ربط المحادثة الحالية.
- لا يمكن الجمع بين `--bind` و`--thread` في الاستدعاء نفسه لـ `/acp spawn`.

## أوضاع سلاسل spawn

يدعم `/acp spawn` الخيار `--thread auto|here|off`.

| الوضع | السلوك |
| ------ | --------------------------------------------------------------------------------------------------- |
| `auto` | في سلسلة نشطة: يربط هذه السلسلة. خارج سلسلة: ينشئ/يربط سلسلة فرعية عند الدعم. |
| `here` | يطلب سلسلة نشطة حالية؛ ويفشل إذا لم تكن داخل سلسلة. |
| `off`  | لا يوجد ربط. تبدأ الجلسة غير مرتبطة. |

ملاحظات:

- على الأسطح التي لا تدعم ربط السلاسل، يكون السلوك الافتراضي فعليًا `off`.
- يتطلب spawn المرتبط بالسلسلة دعم سياسة القناة:
  - Discord: `channels.discord.threadBindings.spawnAcpSessions=true`
  - Telegram: `channels.telegram.threadBindings.spawnAcpSessions=true`
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

يعرض `/acp status` خيارات runtime الفعالة، وعند التوفر، كلًا من معرّفات الجلسة على مستوى runtime وعلى مستوى backend.

تعتمد بعض عناصر التحكم على قدرات backend. وإذا كان backend لا يدعم عنصر تحكم ما، فإن OpenClaw يعيد خطأ واضحًا يفيد بعدم دعم ذلك التحكم.

## دليل أوامر ACP المختصر

| الأمر | ما الذي يفعله | مثال |
| -------------------- | --------------------------------------------------------- | ------------------------------------------------------------- |
| `/acp spawn` | إنشاء جلسة ACP؛ مع ربط حالي اختياري أو ربط سلسلة. | `/acp spawn codex --bind here --cwd /repo` |
| `/acp cancel` | إلغاء الدور الجاري للجلسة المستهدفة. | `/acp cancel agent:codex:acp:<uuid>` |
| `/acp steer` | إرسال تعليمات توجيه إلى الجلسة الجارية. | `/acp steer --session support inbox prioritize failing tests` |
| `/acp close` | إغلاق الجلسة وفك ربط أهداف السلاسل. | `/acp close` |
| `/acp status` | عرض backend، والوضع، والحالة، وخيارات runtime، والقدرات. | `/acp status` |
| `/acp set-mode` | تعيين وضع runtime للجلسة المستهدفة. | `/acp set-mode plan` |
| `/acp set` | كتابة عامة لخيار إعداد runtime. | `/acp set model openai/gpt-5.4` |
| `/acp cwd` | تعيين تجاوز دليل العمل في runtime. | `/acp cwd /Users/user/Projects/repo` |
| `/acp permissions` | تعيين ملف تعريف سياسة الموافقة. | `/acp permissions strict` |
| `/acp timeout` | تعيين مهلة runtime (بالثواني). | `/acp timeout 120` |
| `/acp model` | تعيين تجاوز نموذج runtime. | `/acp model anthropic/claude-opus-4-6` |
| `/acp reset-options` | إزالة تجاوزات خيارات runtime للجلسة. | `/acp reset-options` |
| `/acp sessions` | سرد جلسات ACP الحديثة من المخزن. | `/acp sessions` |
| `/acp doctor` | صحة backend، والقدرات، والإصلاحات القابلة للتنفيذ. | `/acp doctor` |
| `/acp install` | طباعة خطوات تثبيت وتمكين حتمية. | `/acp install` |

يقرأ `/acp sessions` المخزن للجلسة الحالية المرتبطة أو جلسة الطالب. أما الأوامر التي تقبل رموز `session-key` أو `session-id` أو `session-label` فتقوم بحل الأهداف عبر اكتشاف جلسات gateway، بما في ذلك جذور `session.store` المخصصة لكل وكيل.

## تعيين خيارات runtime

يحتوي `/acp` على أوامر مريحة ومُعيّن عام.

العمليات المكافئة:

- يطابق `/acp model <id>` مفتاح إعداد runtime وهو `model`.
- يطابق `/acp permissions <profile>` مفتاح إعداد runtime وهو `approval_policy`.
- يطابق `/acp timeout <seconds>` مفتاح إعداد runtime وهو `timeout`.
- يقوم `/acp cwd <path>` بتحديث تجاوز cwd الخاص بـ runtime مباشرة.
- يمثل `/acp set <key> <value>` المسار العام.
  - حالة خاصة: إذا كان `key=cwd` فيستخدم مسار تجاوز cwd.
- يقوم `/acp reset-options` بمسح جميع تجاوزات runtime للجلسة المستهدفة.

## دعم acpx harness (الحالي)

الأسماء المستعارة المضمنة الحالية لـ acpx harness:

- `claude`
- `codex`
- `copilot`
- `cursor` (Cursor CLI: ‏`cursor-agent acp`)
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

عندما يستخدم OpenClaw backend من نوع acpx، فافضّل هذه القيم لـ `agentId` ما لم يكن إعداد acpx لديك يعرّف أسماء مستعارة مخصصة للوكلاء.
إذا كان تثبيت Cursor المحلي لديك لا يزال يعرض ACP تحت `agent acp`، فقم بتجاوز أمر الوكيل `cursor` في إعداد acpx بدلًا من تغيير القيمة الافتراضية المضمنة.

يمكن أيضًا لاستخدام acpx CLI المباشر استهداف محولات عشوائية عبر `--agent <command>`، لكن منفذ الهروب الخام هذا هو ميزة في acpx CLI (وليس المسار العادي لـ `agentId` في OpenClaw).

## الإعداد المطلوب

خط الأساس الأساسي لـ ACP:

```json5
{
  acp: {
    enabled: true,
    // اختياري. القيمة الافتراضية true؛ عيّن false لإيقاف ACP dispatch مؤقتًا مع إبقاء عناصر تحكم /acp.
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

إعداد ربط السلاسل خاص بكل محول قناة. مثال لـ Discord:

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

إذا لم يعمل spawn المرتبط بالسلاسل في ACP، فتحقق أولًا من علامة الميزة الخاصة بالمحول:

- Discord: ‏`channels.discord.threadBindings.spawnAcpSessions=true`

لا تتطلب عمليات الربط بالمحادثة الحالية إنشاء سلسلة فرعية. وإنما تتطلب سياق محادثة نشطًا ومحول قناة يكشف روابط ACP الخاصة بالمحادثة.

راجع [مرجع الإعدادات](/ar/gateway/configuration-reference).

## إعداد plugin لـ acpx backend

تأتي عمليات التثبيت الجديدة مع plugin runtime المضمّن `acpx` مفعّلًا افتراضيًا، لذا
فإن ACP يعمل عادةً من دون خطوة تثبيت plugin يدوية.

ابدأ بـ:

```text
/acp doctor
```

إذا قمت بتعطيل `acpx`، أو حظرته عبر `plugins.allow` / `plugins.deny`، أو كنت تريد
التبديل إلى نسخة سحب تطوير محلية، فاستخدم مسار plugin الصريح:

```bash
openclaw plugins install acpx
openclaw config set plugins.entries.acpx.enabled true
```

تثبيت مساحة العمل المحلية أثناء التطوير:

```bash
openclaw plugins install ./path/to/local/acpx-plugin
```

ثم تحقق من صحة backend:

```text
/acp doctor
```

### إعداد أمر وإصدار acpx

افتراضيًا، يستخدم plugin backend المضمّن acpx (`acpx`) الملف التنفيذي المثبت محليًا والمثبت بإصدار محدد داخل plugin:

1. يكون الأمر افتراضيًا هو `node_modules/.bin/acpx` المحلي داخل حزمة ACPX plugin.
2. يكون الإصدار المتوقع افتراضيًا هو الإصدار المثبّت في extension.
3. يسجل بدء التشغيل ACP backend مباشرة على أنه غير جاهز.
4. تتحقق مهمة ensure في الخلفية من `acpx --version`.
5. إذا كان الملف التنفيذي المحلي مفقودًا أو لا يطابق الإصدار، فإنه يشغل:
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
- تُحل المسارات النسبية من دليل مساحة عمل OpenClaw.
- يعطّل `expectedVersion: "any"` المطابقة الصارمة للإصدار.
- عندما يشير `command` إلى ملف تنفيذي/مسار مخصص، يتم تعطيل التثبيت التلقائي المحلي للـ plugin.
- يظل بدء تشغيل OpenClaw غير حاجز أثناء تنفيذ فحص صحة backend.

راجع [Plugins](/ar/tools/plugin).

### التثبيت التلقائي للتبعيات

عند تثبيت OpenClaw عالميًا باستخدام `npm install -g openclaw`، يتم تثبيت
تبعيات runtime الخاصة بـ acpx (ملفات تنفيذية خاصة بكل منصة) تلقائيًا
عبر postinstall hook. وإذا فشل التثبيت التلقائي، فإن gateway يبدأ
بشكل طبيعي ويبلغ عن التبعية المفقودة عبر `openclaw acp doctor`.

### جسر MCP لأدوات plugin

افتراضيًا، لا تكشف جلسات ACPX **أدوات OpenClaw المسجلة بواسطة plugins** إلى
ACP harness.

إذا كنت تريد أن تستدعي ACP agents مثل Codex أو Claude Code
أدوات OpenClaw plugin المثبتة مثل استرجاع/تخزين الذاكرة، ففعّل الجسر المخصص:

```bash
openclaw config set plugins.entries.acpx.config.pluginToolsMcpBridge true
```

ما الذي يفعله ذلك:

- يحقن خادم MCP مضمّنًا باسم `openclaw-plugin-tools` في bootstrap
  الخاص بجلسة ACPX.
- يكشف أدوات plugin المسجلة بالفعل بواسطة plugins المثبتة والمفعلة في OpenClaw.
- يبقي الميزة صريحة ومعطلة افتراضيًا.

ملاحظات الأمان والثقة:

- يوسّع هذا سطح الأدوات الخاص بـ ACP harness.
- تحصل ACP agents على وصول فقط إلى أدوات plugin النشطة بالفعل في gateway.
- تعامل مع هذا بوصفه حد الثقة نفسه الذي ينطبق على السماح لتلك plugins بالتنفيذ داخل
  OpenClaw نفسه.
- راجع plugins المثبتة قبل التمكين.

تستمر `mcpServers` المخصصة بالعمل كما كانت من قبل. وجسر plugin-tools المضمن هو
وسيلة راحة إضافية اختيارية، وليس بديلًا عن إعداد خادم MCP العام.

## إعدادات الأذونات

تعمل جلسات ACP بشكل غير تفاعلي — لا توجد واجهة TTY للموافقة أو الرفض على مطالبات إذن الكتابة إلى الملفات وتنفيذ shell. ويوفر plugin ‏acpx مفتاحي إعداد يتحكمان في كيفية التعامل مع الأذونات:

هذه الأذونات الخاصة بـ ACPX harness منفصلة عن موافقات exec في OpenClaw ومنفصلة عن علامات التجاوز الخاصة بالمورّد في CLI-backend مثل Claude CLI ‏`--permission-mode bypassPermissions`. ويُعد ACPX ‏`approve-all` مفتاح الطوارئ على مستوى harness لجلسات ACP.

### `permissionMode`

يتحكم في العمليات التي يستطيع harness agent تنفيذها من دون مطالبة.

| القيمة | السلوك |
| --------------- | --------------------------------------------------------- |
| `approve-all` | الموافقة التلقائية على جميع عمليات الكتابة إلى الملفات وأوامر shell. |
| `approve-reads` | الموافقة التلقائية على القراءات فقط؛ أما الكتابة وexec فتتطلبان مطالبات. |
| `deny-all` | رفض جميع مطالبات الأذونات. |

### `nonInteractivePermissions`

يتحكم فيما يحدث عندما كان من المفترض عرض مطالبة إذن ولكن لا توجد واجهة TTY تفاعلية متاحة (وهو الحال دائمًا في جلسات ACP).

| القيمة | السلوك |
| ------ | ----------------------------------------------------------------- |
| `fail` | إيقاف الجلسة بخطأ `AcpRuntimeError`. **(الافتراضي)** |
| `deny` | رفض الإذن بصمت ومتابعة العمل (تدهور سلس). |

### الإعداد

يُعيَّن عبر إعدادات plugin:

```bash
openclaw config set plugins.entries.acpx.config.permissionMode approve-all
openclaw config set plugins.entries.acpx.config.nonInteractivePermissions fail
```

أعد تشغيل gateway بعد تغيير هذه القيم.

> **مهم:** يستخدم OpenClaw حاليًا افتراضيًا `permissionMode=approve-reads` و`nonInteractivePermissions=fail`. وفي جلسات ACP غير التفاعلية، قد تفشل أي عملية كتابة أو exec تؤدي إلى مطالبة إذن بخطأ `AcpRuntimeError: Permission prompt unavailable in non-interactive mode`.
>
> إذا كنت تحتاج إلى تقييد الأذونات، فعيّن `nonInteractivePermissions` إلى `deny` حتى تتدهور الجلسات بسلاسة بدلًا من التعطل.

## استكشاف الأخطاء وإصلاحها

| العَرَض | السبب المحتمل | الإصلاح |
| --------------------------------------------------------------------------- | ------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ACP runtime backend is not configured` | plugin الخاص بـ backend مفقود أو معطّل. | ثبّت plugin الخاص بـ backend وفعّله، ثم شغّل `/acp doctor`. |
| `ACP is disabled by policy (acp.enabled=false)` | ‏ACP معطّل عالميًا. | عيّن `acp.enabled=true`. |
| `ACP dispatch is disabled by policy (acp.dispatch.enabled=false)` | ‏Dispatch من رسائل السلاسل العادية معطّل. | عيّن `acp.dispatch.enabled=true`. |
| `ACP agent "<id>" is not allowed by policy` | الوكيل غير موجود في allowlist. | استخدم `agentId` مسموحًا به أو حدّث `acp.allowedAgents`. |
| `Unable to resolve session target: ...` | رمز مفتاح/معرّف/تسمية غير صحيح. | شغّل `/acp sessions`، وانسخ المفتاح/التسمية الدقيقة، ثم أعد المحاولة. |
| `--bind here requires running /acp spawn inside an active ... conversation` | استُخدم `--bind here` من دون محادثة نشطة قابلة للربط. | انتقل إلى الدردشة/القناة الهدف وأعد المحاولة، أو استخدم spawn غير مرتبط. |
| `Conversation bindings are unavailable for <channel>.` | المحول يفتقر إلى إمكانية ربط ACP بالمحادثة الحالية. | استخدم `/acp spawn ... --thread ...` عند الدعم، أو هيئ `bindings[]` ذات المستوى الأعلى، أو انتقل إلى قناة مدعومة. |
| `--thread here requires running /acp spawn inside an active ... thread` | استُخدم `--thread here` خارج سياق سلسلة. | انتقل إلى السلسلة المستهدفة أو استخدم `--thread auto`/`off`. |
| `Only <user-id> can rebind this channel/conversation/thread.` | مستخدم آخر يملك هدف الربط النشط. | أعد الربط بصفتك المالك أو استخدم محادثة أو سلسلة أخرى. |
| `Thread bindings are unavailable for <channel>.` | المحول يفتقر إلى إمكانية ربط السلاسل. | استخدم `--thread off` أو انتقل إلى محول/قناة مدعومة. |
| `Sandboxed sessions cannot spawn ACP sessions ...` | runtime الخاص بـ ACP يعمل على المضيف؛ وجلسة الطالب داخل sandbox. | استخدم `runtime="subagent"` من جلسات sandbox، أو نفّذ ACP spawn من جلسة خارج sandbox. |
| `sessions_spawn sandbox="require" is unsupported for runtime="acp" ...` | طُلب `sandbox="require"` لـ runtime من نوع ACP. | استخدم `runtime="subagent"` عندما تكون sandbox مطلوبة، أو استخدم ACP مع `sandbox="inherit"` من جلسة غير sandbox. |
| فقدان بيانات ACP الوصفية للجلسة المرتبطة | بيانات ACP الوصفية قديمة/محذوفة. | أعد الإنشاء باستخدام `/acp spawn`، ثم أعد الربط/التركيز على السلسلة. |
| `AcpRuntimeError: Permission prompt unavailable in non-interactive mode` | يمنع `permissionMode` عمليات الكتابة/exec في جلسة ACP غير تفاعلية. | عيّن `plugins.entries.acpx.config.permissionMode` إلى `approve-all` وأعد تشغيل gateway. راجع [إعدادات الأذونات](#permission-configuration). |
| تفشل جلسة ACP مبكرًا مع قدر ضئيل من المخرجات | يتم حظر مطالبات الأذونات بواسطة `permissionMode`/`nonInteractivePermissions`. | تحقق من سجلات gateway بحثًا عن `AcpRuntimeError`. وللحصول على أذونات كاملة، عيّن `permissionMode=approve-all`؛ ولتدهور سلس، عيّن `nonInteractivePermissions=deny`. |
| تتوقف جلسة ACP إلى أجل غير مسمى بعد إكمال العمل | انتهت عملية harness لكن جلسة ACP لم تبلغ عن الإكمال. | راقب عبر `ps aux \| grep acpx`; ثم اقتل العمليات القديمة يدويًا. |
