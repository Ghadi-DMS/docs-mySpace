---
read_when:
    - تريد فهم كيفية قيام OpenClaw بتجميع سياق النموذج
    - أنت تنتقل بين المحرك القديم ومحرك plugin
    - أنت تبني plugin لمحرك سياق
summary: 'محرك السياق: تجميع سياق قابل للتوصيل، والضغط، ودورة حياة الوكيل الفرعي'
title: محرك السياق
x-i18n:
    generated_at: "2026-04-08T02:14:24Z"
    model: gpt-5.4
    provider: openai
    source_hash: e8290ac73272eee275bce8e481ac7959b65386752caa68044d0c6f3e450acfb1
    source_path: concepts/context-engine.md
    workflow: 15
---

# محرك السياق

يتحكم **محرك السياق** في كيفية بناء OpenClaw لسياق النموذج لكل تشغيل.
ويقرر الرسائل التي يجب تضمينها، وكيفية تلخيص السجل الأقدم، وكيفية
إدارة السياق عبر حدود الوكلاء الفرعيين.

يأتي OpenClaw مزودًا بمحرك `legacy` مدمج. ويمكن لـ plugins تسجيل
محركات بديلة تحل محل دورة حياة محرك السياق النشطة.

## البدء السريع

تحقق من المحرك النشط حاليًا:

```bash
openclaw doctor
# or inspect config directly:
cat ~/.openclaw/openclaw.json | jq '.plugins.slots.contextEngine'
```

### تثبيت plugin لمحرك سياق

يتم تثبيت plugins الخاصة بمحرك السياق مثل أي plugin أخرى في OpenClaw. ثبّت
plugin أولًا، ثم حدّد المحرك في slot:

```bash
# Install from npm
openclaw plugins install @martian-engineering/lossless-claw

# Or install from a local path (for development)
openclaw plugins install -l ./my-context-engine
```

بعد ذلك، فعّل plugin وحددها كمحرك نشط في الإعدادات:

```json5
// openclaw.json
{
  plugins: {
    slots: {
      contextEngine: "lossless-claw", // must match the plugin's registered engine id
    },
    entries: {
      "lossless-claw": {
        enabled: true,
        // Plugin-specific config goes here (see the plugin's docs)
      },
    },
  },
}
```

أعد تشغيل البوابة بعد التثبيت والإعداد.

للرجوع إلى المحرك المدمج، اضبط `contextEngine` على `"legacy"` (أو
احذف المفتاح بالكامل — إذ إن `"legacy"` هو الافتراضي).

## كيف يعمل

في كل مرة يشغّل فيها OpenClaw مطالبة نموذج، يشارك محرك السياق في
أربع نقاط في دورة الحياة:

1. **الاستيعاب** — يُستدعى عند إضافة رسالة جديدة إلى الجلسة. ويمكن للمحرك
   تخزين الرسالة أو فهرستها في مخزن البيانات الخاص به.
2. **التجميع** — يُستدعى قبل كل تشغيل للنموذج. ويعيد المحرك مجموعة
   مرتبة من الرسائل (و`systemPromptAddition` اختياري) تتناسب مع
   ميزانية الرموز.
3. **الضغط** — يُستدعى عندما تمتلئ نافذة السياق، أو عندما يشغّل المستخدم
   `/compact`. ويلخّص المحرك السجل الأقدم لتحرير مساحة.
4. **بعد الدور** — يُستدعى بعد اكتمال التشغيل. ويمكن للمحرك حفظ الحالة،
   أو تشغيل ضغط في الخلفية، أو تحديث الفهارس.

### دورة حياة الوكيل الفرعي (اختياري)

يستدعي OpenClaw حاليًا خطافًا واحدًا لدورة حياة الوكيل الفرعي:

- **onSubagentEnded** — للتنظيف عند اكتمال جلسة وكيل فرعي أو اكتساحها.

خطاف `prepareSubagentSpawn` جزء من الواجهة لاستخدام مستقبلي، لكن
بيئة التشغيل لا تستدعيه بعد.

### إضافة مطالبة النظام

يمكن للطريقة `assemble` أن تعيد سلسلة `systemPromptAddition`. ويقوم OpenClaw
بإضافتها في بداية مطالبة النظام الخاصة بالتشغيل. ويتيح هذا للمحركات حقن
إرشادات استدعاء ديناميكية، أو تعليمات استرجاع، أو تلميحات
مدركة للسياق دون الحاجة إلى ملفات مساحة عمل ثابتة.

## المحرك القديم

يحافظ المحرك `legacy` المدمج على السلوك الأصلي لـ OpenClaw:

- **الاستيعاب**: لا إجراء (يتولى مدير الجلسة حفظ الرسائل مباشرة).
- **التجميع**: تمرير مباشر (يتولى خط الأنابيب الحالي sanitize → validate → limit
  في بيئة التشغيل تجميع السياق).
- **الضغط**: يفوّض إلى ضغط التلخيص المدمج، الذي ينشئ
  ملخصًا واحدًا للرسائل الأقدم ويحافظ على الرسائل الحديثة كما هي.
- **بعد الدور**: لا إجراء.

لا يسجل المحرك القديم أدوات ولا يوفّر `systemPromptAddition`.

عندما لا يكون `plugins.slots.contextEngine` مضبوطًا (أو يكون مضبوطًا على `"legacy"`)،
يُستخدم هذا المحرك تلقائيًا.

## محركات plugin

يمكن لـ plugin تسجيل محرك سياق باستخدام plugin API:

```ts
import { buildMemorySystemPromptAddition } from "openclaw/plugin-sdk/core";

export default function register(api) {
  api.registerContextEngine("my-engine", () => ({
    info: {
      id: "my-engine",
      name: "My Context Engine",
      ownsCompaction: true,
    },

    async ingest({ sessionId, message, isHeartbeat }) {
      // Store the message in your data store
      return { ingested: true };
    },

    async assemble({ sessionId, messages, tokenBudget, availableTools, citationsMode }) {
      // Return messages that fit the budget
      return {
        messages: buildContext(messages, tokenBudget),
        estimatedTokens: countTokens(messages),
        systemPromptAddition: buildMemorySystemPromptAddition({
          availableTools: availableTools ?? new Set(),
          citationsMode,
        }),
      };
    },

    async compact({ sessionId, force }) {
      // Summarize older context
      return { ok: true, compacted: true };
    },
  }));
}
```

ثم فعّله في الإعدادات:

```json5
{
  plugins: {
    slots: {
      contextEngine: "my-engine",
    },
    entries: {
      "my-engine": {
        enabled: true,
      },
    },
  },
}
```

### واجهة ContextEngine

الأعضاء المطلوبة:

| العضو             | النوع     | الغرض                                                   |
| ------------------ | -------- | -------------------------------------------------------- |
| `info`             | خاصية | معرّف المحرك واسمه وإصداره وما إذا كان يملك الضغط |
| `ingest(params)`   | طريقة   | تخزين رسالة واحدة                                   |
| `assemble(params)` | طريقة   | بناء السياق لتشغيل نموذج (يعيد `AssembleResult`) |
| `compact(params)`  | طريقة   | تلخيص/تقليل السياق                                 |

تعيد `assemble` قيمة `AssembleResult` تحتوي على:

- `messages` — الرسائل المرتبة التي ستُرسل إلى النموذج.
- `estimatedTokens` (مطلوب، `number`) — تقدير المحرك لإجمالي
  الرموز في السياق المجمّع. ويستخدم OpenClaw هذا لاتخاذ قرارات
  حدّ الضغط وللتقارير التشخيصية.
- `systemPromptAddition` (اختياري، `string`) — تُضاف في بداية مطالبة النظام.

الأعضاء الاختيارية:

| العضو                         | النوع   | الغرض                                                                                                         |
| ------------------------------ | ------ | --------------------------------------------------------------------------------------------------------------- |
| `bootstrap(params)`            | طريقة | تهيئة حالة المحرك لجلسة. تُستدعى مرة واحدة عندما يرى المحرك جلسة لأول مرة (مثل استيراد السجل). |
| `ingestBatch(params)`          | طريقة | استيعاب دور مكتمل كدفعة. تُستدعى بعد اكتمال التشغيل، مع جميع رسائل ذلك الدور دفعة واحدة.     |
| `afterTurn(params)`            | طريقة | أعمال دورة الحياة بعد التشغيل (حفظ الحالة، تشغيل ضغط في الخلفية).                                         |
| `prepareSubagentSpawn(params)` | طريقة | إعداد حالة مشتركة لجلسة فرعية.                                                                        |
| `onSubagentEnded(params)`      | طريقة | التنظيف بعد انتهاء وكيل فرعي.                                                                                 |
| `dispose()`                    | طريقة | تحرير الموارد. تُستدعى أثناء إيقاف تشغيل البوابة أو إعادة تحميل plugin — وليس لكل جلسة.                           |

### ownsCompaction

يتحكم `ownsCompaction` فيما إذا كان الضغط التلقائي المدمج أثناء المحاولة في Pi
يبقى مفعّلًا أثناء التشغيل:

- `true` — يملك المحرك سلوك الضغط. يعطّل OpenClaw الضغط التلقائي المدمج في Pi
  لذلك التشغيل، ويصبح تنفيذ `compact()` الخاص بالمحرك مسؤولًا عن `/compact`،
  وضغط الاسترداد من تجاوز السعة، وأي ضغط استباقي
  يريد تنفيذه في `afterTurn()`.
- `false` أو غير مضبوط — قد يستمر تشغيل الضغط التلقائي المدمج في Pi أثناء
  تنفيذ المطالبة، لكن تظل طريقة `compact()` الخاصة بالمحرك النشط
  مستدعاة من أجل `/compact` والاسترداد من تجاوز السعة.

لا يعني `ownsCompaction: false` أن OpenClaw يعود تلقائيًا إلى
مسار الضغط الخاص بالمحرك القديم.

وهذا يعني أن هناك نمطين صالحين لـ plugin:

- **وضع التملك** — نفّذ خوارزمية الضغط الخاصة بك واضبط
  `ownsCompaction: true`.
- **وضع التفويض** — اضبط `ownsCompaction: false` واجعل `compact()` تستدعي
  `delegateCompactionToRuntime(...)` من `openclaw/plugin-sdk/core` لاستخدام
  سلوك الضغط المدمج في OpenClaw.

إن `compact()` التي لا تنفّذ أي شيء غير آمنة لمحرك نشط غير مالك لأن ذلك
يعطّل مسار `/compact` العادي ومسار ضغط الاسترداد من تجاوز السعة
لـ slot ذلك المحرك.

## مرجع الإعدادات

```json5
{
  plugins: {
    slots: {
      // Select the active context engine. Default: "legacy".
      // Set to a plugin id to use a plugin engine.
      contextEngine: "legacy",
    },
  },
}
```

يكون الـ slot حصريًا في وقت التشغيل — إذ لا يُحل سوى محرك سياق مسجل واحد
لتشغيل أو عملية ضغط معينة. ويمكن لبقية plugins المفعّلة
من النوع `kind: "context-engine"` أن تُحمّل وتنفّذ
شيفرة التسجيل الخاصة بها؛ إذ يحدد `plugins.slots.contextEngine` فقط
معرّف المحرك المسجل الذي يحله OpenClaw عندما يحتاج إلى محرك سياق.

## العلاقة مع الضغط والذاكرة

- **الضغط** هو إحدى مسؤوليات محرك السياق. يفوّض المحرك القديم
  إلى التلخيص المدمج في OpenClaw. ويمكن لمحركات plugin تنفيذ
  أي استراتيجية ضغط (ملخصات DAG، أو استرجاع متجهي، وما إلى ذلك).
- **plugins الذاكرة** (`plugins.slots.memory`) منفصلة عن محركات السياق.
  توفّر plugins الذاكرة البحث/الاسترجاع؛ بينما تتحكم محركات السياق فيما
  يراه النموذج. ويمكنهما العمل معًا — فقد يستخدم محرك سياق
  بيانات plugin الذاكرة أثناء التجميع. ويُفضّل أن تستخدم محركات plugin التي تريد
  مسار مطالبة الذاكرة النشط `buildMemorySystemPromptAddition(...)` من
  `openclaw/plugin-sdk/core`، إذ يحوّل مقاطع مطالبة الذاكرة النشطة
  إلى `systemPromptAddition` جاهزة للإضافة في البداية. وإذا احتاج المحرك إلى
  تحكم على مستوى أدنى، فلا يزال بإمكانه سحب الأسطر الخام من
  `openclaw/plugin-sdk/memory-host-core` عبر
  `buildActiveMemoryPromptSection(...)`.
- **تهذيب الجلسة** (قص نتائج الأدوات القديمة داخل الذاكرة) يظل يعمل
  بغض النظر عن محرك السياق النشط.

## نصائح

- استخدم `openclaw doctor` للتحقق من أن محركك يُحمَّل بشكل صحيح.
- إذا كنت تبدّل بين المحركات، فستستمر الجلسات الحالية بسجلها الحالي.
  وسيتولى المحرك الجديد التشغيلات المستقبلية.
- تُسجّل أخطاء المحرك وتظهر في التشخيصات. وإذا فشل محرك plugin
  في التسجيل أو تعذر حل معرّف المحرك المحدد، فلن يعود OpenClaw
  تلقائيًا؛ بل ستفشل التشغيلات حتى تصلح plugin أو
  تعيد `plugins.slots.contextEngine` إلى `"legacy"`.
- لأغراض التطوير، استخدم `openclaw plugins install -l ./my-engine` لربط
  دليل plugin محليًا من دون نسخه.

راجع أيضًا: [الضغط](/ar/concepts/compaction)، [السياق](/ar/concepts/context),
[Plugins](/ar/tools/plugin)، [بيان plugin](/ar/plugins/manifest).

## ذو صلة

- [السياق](/ar/concepts/context) — كيف يُبنى السياق لأدوار الوكيل
- [معمارية Plugin](/ar/plugins/architecture) — تسجيل plugins لمحرك السياق
- [الضغط](/ar/concepts/compaction) — تلخيص المحادثات الطويلة
