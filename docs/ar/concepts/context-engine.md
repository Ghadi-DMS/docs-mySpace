---
read_when:
    - تريد فهم كيفية قيام OpenClaw بتجميع سياق النموذج
    - أنت تنتقل بين المحرك القديم ومحرك plugin
    - أنت تبني plugin لمحرك سياق
summary: 'محرك السياق: تجميع سياق قابل للتوصيل، والضغط، ودورة حياة الوكلاء الفرعيين'
title: Context Engine
x-i18n:
    generated_at: "2026-04-05T12:40:19Z"
    model: gpt-5.4
    provider: openai
    source_hash: 19fd8cbb0e953f58fd84637fc4ceefc65984312cf2896d338318bc8cf860e6d9
    source_path: concepts/context-engine.md
    workflow: 15
---

# Context Engine

يتحكم **محرك السياق** في كيفية بناء OpenClaw لسياق النموذج لكل تشغيل.
فهو يحدد الرسائل التي يجب تضمينها، وكيفية تلخيص السجل الأقدم، وكيفية
إدارة السياق عبر حدود الوكلاء الفرعيين.

يوفر OpenClaw محركًا مدمجًا باسم `legacy`. ويمكن لـ Plugins تسجيل
محركات بديلة تستبدل دورة حياة محرك السياق النشط.

## البدء السريع

تحقق من المحرك النشط:

```bash
openclaw doctor
# أو افحص التكوين مباشرة:
cat ~/.openclaw/openclaw.json | jq '.plugins.slots.contextEngine'
```

### تثبيت plugin لمحرك سياق

يتم تثبيت Plugins الخاصة بمحرك السياق مثل أي plugin آخر في OpenClaw. قم بالتثبيت
أولًا، ثم حدد المحرك في الـ slot:

```bash
# التثبيت من npm
openclaw plugins install @martian-engineering/lossless-claw

# أو التثبيت من مسار محلي (للتطوير)
openclaw plugins install -l ./my-context-engine
```

ثم قم بتمكين plugin وحدده كمحرك نشط في التكوين:

```json5
// openclaw.json
{
  plugins: {
    slots: {
      contextEngine: "lossless-claw", // يجب أن يطابق معرّف المحرك المسجل في plugin
    },
    entries: {
      "lossless-claw": {
        enabled: true,
        // يذهب هنا التكوين الخاص بـ plugin (راجع وثائق plugin)
      },
    },
  },
}
```

أعد تشغيل gateway بعد التثبيت والتكوين.

للعودة إلى المحرك المدمج، اضبط `contextEngine` على `"legacy"` (أو
احذف المفتاح بالكامل — إذ إن `"legacy"` هو الافتراضي).

## كيف يعمل

في كل مرة يشغّل فيها OpenClaw مطالبة نموذج، يشارك محرك السياق في
أربع نقاط في دورة الحياة:

1. **Ingest** — يُستدعى عند إضافة رسالة جديدة إلى الجلسة. يمكن للمحرك
   تخزين الرسالة أو فهرستها في مخزن البيانات الخاص به.
2. **Assemble** — يُستدعى قبل كل تشغيل للنموذج. يعيد المحرك مجموعة
   مرتبة من الرسائل (و`systemPromptAddition` اختياريًا) تتلاءم مع
   ميزانية الرموز.
3. **Compact** — يُستدعى عندما تمتلئ نافذة السياق، أو عندما يشغّل المستخدم
   `/compact`. يقوم المحرك بتلخيص السجل الأقدم لتحرير مساحة.
4. **After turn** — يُستدعى بعد اكتمال التشغيل. يمكن للمحرك حفظ الحالة،
   أو تشغيل ضغط في الخلفية، أو تحديث الفهارس.

### دورة حياة الوكيل الفرعي (اختياري)

يستدعي OpenClaw حاليًا hook واحدة فقط لدورة حياة الوكيل الفرعي:

- **onSubagentEnded** — تنظيف عند اكتمال جلسة وكيل فرعي أو عند إزالتها.

تُعد hook ‏`prepareSubagentSpawn` جزءًا من الواجهة لاستخدام مستقبلي، لكن
وقت التشغيل لا يستدعيها بعد.

### إضافة مطالبة النظام

يمكن لطريقة `assemble` أن تعيد سلسلة `systemPromptAddition`. يقوم OpenClaw
بإضافتها في مقدمة مطالبة النظام لذلك التشغيل. يتيح ذلك للمحركات حقن
إرشادات استرجاع ديناميكية، أو تعليمات استدعاء، أو تلميحات
مدركة للسياق من دون الحاجة إلى ملفات مساحة عمل ثابتة.

## المحرك القديم

يحافظ المحرك المدمج `legacy` على السلوك الأصلي لـ OpenClaw:

- **Ingest**: لا عملية (إذ يتولى مدير الجلسة حفظ الرسائل مباشرة).
- **Assemble**: تمرير مباشر (يتولى المسار الحالي sanitize → validate → limit
  في وقت التشغيل تجميع السياق).
- **Compact**: يفوض إلى ضغط التلخيص المدمج، والذي ينشئ
  ملخصًا واحدًا للرسائل الأقدم ويُبقي الرسائل الحديثة كما هي.
- **After turn**: لا عملية.

لا يسجل المحرك القديم أدوات ولا يوفّر `systemPromptAddition`.

عندما لا تكون `plugins.slots.contextEngine` مضبوطة (أو تكون مضبوطة على `"legacy"`)،
يُستخدم هذا المحرك تلقائيًا.

## محركات plugin

يمكن لأي plugin تسجيل محرك سياق باستخدام API الخاصة بـ plugin:

```ts
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

    async assemble({ sessionId, messages, tokenBudget }) {
      // Return messages that fit the budget
      return {
        messages: buildContext(messages, tokenBudget),
        estimatedTokens: countTokens(messages),
        systemPromptAddition: "Use lcm_grep to search history...",
      };
    },

    async compact({ sessionId, force }) {
      // Summarize older context
      return { ok: true, compacted: true };
    },
  }));
}
```

ثم قم بتمكينه في التكوين:

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

| العضو              | النوع    | الغرض                                                  |
| ------------------ | -------- | ------------------------------------------------------ |
| `info`             | خاصية    | معرّف المحرك واسمه وإصداره وما إذا كان يملك الضغط      |
| `ingest(params)`   | طريقة    | تخزين رسالة واحدة                                      |
| `assemble(params)` | طريقة    | بناء سياق لتشغيل نموذج (تعيد `AssembleResult`)         |
| `compact(params)`  | طريقة    | تلخيص/تقليل السياق                                     |

تعيد `assemble` قيمة `AssembleResult` تحتوي على:

- `messages` — الرسائل المرتبة التي سيتم إرسالها إلى النموذج.
- `estimatedTokens` ‏(مطلوب، `number`) — تقدير المحرك لإجمالي
  الرموز في السياق المجمع. يستخدم OpenClaw هذا لاتخاذ قرارات حدود الضغط
  ولإعداد تقارير التشخيص.
- `systemPromptAddition` ‏(اختياري، `string`) — يُضاف في مقدمة مطالبة النظام.

الأعضاء الاختيارية:

| العضو                         | النوع  | الغرض                                                                                                           |
| ----------------------------- | ------ | --------------------------------------------------------------------------------------------------------------- |
| `bootstrap(params)`           | طريقة  | تهيئة حالة المحرك لجلسة. تُستدعى مرة واحدة عندما يرى المحرك جلسة لأول مرة (مثل استيراد السجل).                |
| `ingestBatch(params)`         | طريقة  | استيعاب دور مكتمل على شكل دفعة. تُستدعى بعد اكتمال تشغيل، مع جميع رسائل ذلك الدور دفعة واحدة.                 |
| `afterTurn(params)`           | طريقة  | أعمال ما بعد التشغيل في دورة الحياة (حفظ الحالة، تشغيل ضغط في الخلفية).                                       |
| `prepareSubagentSpawn(params)`| طريقة  | إعداد الحالة المشتركة لجلسة فرعية.                                                                             |
| `onSubagentEnded(params)`     | طريقة  | التنظيف بعد انتهاء وكيل فرعي.                                                                                  |
| `dispose()`                   | طريقة  | تحرير الموارد. تُستدعى أثناء إيقاف gateway أو إعادة تحميل plugin — وليس لكل جلسة.                              |

### ownsCompaction

يتحكم `ownsCompaction` فيما إذا كان الضغط التلقائي المدمج داخل المحاولة في Pi
يبقى مفعّلًا أثناء التشغيل:

- `true` — يمتلك المحرك سلوك الضغط. يعطّل OpenClaw الضغط التلقائي المدمج في Pi
  لذلك التشغيل، ويصبح تنفيذ `compact()` الخاص بالمحرك مسؤولًا عن `/compact`،
  وضغط استعادة تجاوز السعة، وأي ضغط استباقي
  يريد تنفيذه في `afterTurn()`.
- `false` أو غير معيّن — قد يستمر الضغط التلقائي المدمج في Pi أثناء
  تنفيذ المطالبة، لكن تظل طريقة `compact()` الخاصة بالمحرك النشط تُستدعى
  من أجل `/compact` واستعادة تجاوز السعة.

لا تعني `ownsCompaction: false` أن OpenClaw يعود تلقائيًا
إلى مسار الضغط الخاص بالمحرك القديم.

وهذا يعني وجود نمطين صالحين لـ plugin:

- **وضع التملك** — نفّذ خوارزمية الضغط الخاصة بك واضبط
  `ownsCompaction: true`.
- **وضع التفويض** — اضبط `ownsCompaction: false` واجعل `compact()` تستدعي
  `delegateCompactionToRuntime(...)` من `openclaw/plugin-sdk/core` لاستخدام
  سلوك الضغط المدمج في OpenClaw.

إن `compact()` التي لا تقوم بأي شيء غير آمنة لمحرك نشط غير مالك لأنها
تعطّل المسار العادي لـ `/compact` وضغط استعادة تجاوز السعة لذلك الـ engine slot.

## مرجع التكوين

```json5
{
  plugins: {
    slots: {
      // حدد محرك السياق النشط. الافتراضي: "legacy".
      // اضبطه على معرّف plugin لاستخدام محرك plugin.
      contextEngine: "legacy",
    },
  },
}
```

يكون الـ slot حصريًا في وقت التشغيل — لا يتم
حل سوى محرك سياق مسجل واحد لتشغيل معيّن أو عملية ضغط. أما بقية
Plugins من النوع `kind: "context-engine"` والمفعلة فيمكنها أن تُحمّل وتنفّذ
شفرة التسجيل الخاصة بها؛ إذ يحدد `plugins.slots.contextEngine` فقط أي معرّف محرك مسجل
يحلّه OpenClaw عندما يحتاج إلى محرك سياق.

## العلاقة مع الضغط والذاكرة

- **الضغط** هو إحدى مسؤوليات محرك السياق. يفوض المحرك القديم
  إلى التلخيص المدمج في OpenClaw. ويمكن لمحركات plugin تنفيذ
  أي استراتيجية ضغط (ملخصات DAG، أو استدعاء vector، وما إلى ذلك).
- **Plugins الذاكرة** (`plugins.slots.memory`) منفصلة عن محركات السياق.
  توفر Plugins الذاكرة البحث/الاستدعاء؛ بينما تتحكم محركات السياق فيما
  يراه النموذج. ويمكنهما العمل معًا — فقد يستخدم محرك السياق بيانات
  plugin الذاكرة أثناء التجميع.
- يظل **تشذيب الجلسة** (قص نتائج الأدوات القديمة في الذاكرة)
  يعمل بغض النظر عن محرك السياق النشط.

## نصائح

- استخدم `openclaw doctor` للتحقق من أن محركك يتم تحميله بشكل صحيح.
- إذا قمت بالتبديل بين المحركات، فستستمر الجلسات الحالية بسجلها الحالي.
  وسيتولى المحرك الجديد التشغيلات المستقبلية.
- يتم تسجيل أخطاء المحرك وعرضها في التشخيصات. إذا فشل محرك plugin
  في التسجيل أو تعذر حل معرّف المحرك المحدد، فلن يعود OpenClaw تلقائيًا؛
  بل ستفشل التشغيلات حتى تصلح plugin أو
  تعيد `plugins.slots.contextEngine` إلى `"legacy"`.
- لأغراض التطوير، استخدم `openclaw plugins install -l ./my-engine` لربط
  دليل plugin محلي من دون نسخه.

راجع أيضًا: [الضغط](/concepts/compaction)، [السياق](/concepts/context)،
[Plugins](/tools/plugin)، [بيان plugin](/plugins/manifest).

## ذو صلة

- [السياق](/concepts/context) — كيفية بناء السياق لأدوار الوكيل
- [بنية plugin](/plugins/architecture) — تسجيل Plugins لمحركات السياق
- [الضغط](/concepts/compaction) — تلخيص المحادثات الطويلة
