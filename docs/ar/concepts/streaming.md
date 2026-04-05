---
read_when:
    - شرح كيفية عمل البث أو التجزئة على القنوات
    - تغيير سلوك بث الكتل أو تجزئة القنوات
    - تصحيح ردود الكتل المكررة/المبكرة أو بث المعاينة في القنوات
summary: سلوك البث + التجزئة (ردود الكتل، وبث المعاينة في القنوات، وربط الأوضاع)
title: البث والتجزئة
x-i18n:
    generated_at: "2026-04-05T12:41:48Z"
    model: gpt-5.4
    provider: openai
    source_hash: 44b0d08c7eafcb32030ef7c8d5719c2ea2d34e4bac5fdad8cc8b3f4e9e9fad97
    source_path: concepts/streaming.md
    workflow: 15
---

# البث + التجزئة

يحتوي OpenClaw على طبقتين منفصلتين للبث:

- **بث الكتل (القنوات):** إصدار **كتل** مكتملة أثناء كتابة المساعد. وهذه رسائل قنوات عادية (وليست فروق رموز token deltas).
- **بث المعاينة (Telegram/Discord/Slack):** تحديث **رسالة معاينة** مؤقتة أثناء التوليد.

لا يوجد **بث حقيقي لفروق الرموز** إلى رسائل القنوات اليوم. فبث المعاينة يعتمد على الرسائل (إرسال + تعديلات/إلحاقات).

## بث الكتل (رسائل القنوات)

يرسل بث الكتل مخرجات المساعد على شكل أجزاء كبيرة نسبيًا عندما تصبح متاحة.

```
Model output
  └─ text_delta/events
       ├─ (blockStreamingBreak=text_end)
       │    └─ chunker emits blocks as buffer grows
       └─ (blockStreamingBreak=message_end)
            └─ chunker flushes at message_end
                   └─ channel send (block replies)
```

المفتاح:

- `text_delta/events`: أحداث تدفق النموذج (وقد تكون متفرقة في النماذج غير المتدفقة).
- `chunker`: ‏`EmbeddedBlockChunker` الذي يطبق حدودًا دنيا/عليا + تفضيل نقاط الانقطاع.
- `channel send`: الرسائل الصادرة الفعلية (ردود الكتل).

**عناصر التحكم:**

- `agents.defaults.blockStreamingDefault`: ‏`"on"`/`"off"` (الافتراضي off).
- تجاوزات القنوات: ‏`*.blockStreaming` (ومتغيرات لكل حساب) لفرض `"on"`/`"off"` لكل قناة.
- `agents.defaults.blockStreamingBreak`: ‏`"text_end"` أو `"message_end"`.
- `agents.defaults.blockStreamingChunk`: ‏`{ minChars, maxChars, breakPreference? }`.
- `agents.defaults.blockStreamingCoalesce`: ‏`{ minChars?, maxChars?, idleMs? }` (دمج الكتل المتدفقة قبل الإرسال).
- الحد الصارم للقناة: ‏`*.textChunkLimit` (مثل `channels.whatsapp.textChunkLimit`).
- وضع تجزئة القناة: ‏`*.chunkMode` (`length` افتراضيًا، و`newline` يقسم عند الأسطر الفارغة (حدود الفقرات) قبل التجزئة حسب الطول).
- الحد المرن لـ Discord: ‏`channels.discord.maxLinesPerMessage` (الافتراضي 17) يقسم الردود الطويلة رأسيًا لتجنب قص واجهة المستخدم.

**دلالات الحدود:**

- `text_end`: بث الكتل بمجرد أن يصدرها chunker؛ والتفريغ عند كل `text_end`.
- `message_end`: الانتظار حتى تنتهي رسالة المساعد، ثم تفريغ المخرجات المخزنة.

يستخدم `message_end` أيضًا chunker إذا تجاوز النص المخزن `maxChars`، لذلك يمكنه إصدار عدة أجزاء في النهاية.

## خوارزمية التجزئة (الحدود الدنيا/العليا)

تُنفَّذ تجزئة الكتل بواسطة `EmbeddedBlockChunker`:

- **الحد الأدنى:** لا تُصدر شيئًا حتى يصبح المخزن >= `minChars` (ما لم يكن ذلك إجباريًا).
- **الحد الأعلى:** يُفضَّل التقسيم قبل `maxChars`؛ وإذا كان ذلك إجباريًا، فيتم التقسيم عند `maxChars`.
- **تفضيل نقطة الانقطاع:** `paragraph` → `newline` → `sentence` → `whitespace` → انقطاع صارم.
- **أسوار الشيفرة:** لا يتم أبدًا التقسيم داخل الأسوار؛ وعند الإجبار عند `maxChars`، يتم إغلاق السياج ثم إعادة فتحه للحفاظ على صحة Markdown.

يتم تقييد `maxChars` بحد `textChunkLimit` الخاص بالقناة، لذلك لا يمكنك تجاوز الحدود الخاصة بكل قناة.

## الدمج (دمج الكتل المتدفقة)

عند تفعيل بث الكتل، يمكن لـ OpenClaw **دمج أجزاء الكتل المتتالية**
قبل إرسالها. ويقلل هذا من "إزعاج الأسطر المفردة" مع الاستمرار في
توفير مخرجات تدريجية.

- ينتظر الدمج **فجوات الخمول** (`idleMs`) قبل التفريغ.
- تُحدَّد المخازن بـ `maxChars` وسيتم تفريغها إذا تجاوزته.
- يمنع `minChars` إرسال الأجزاء الصغيرة جدًا حتى يتراكم مقدار كافٍ من النص
  (ويُرسل التفريغ النهائي دائمًا النص المتبقي).
- يُشتق الموصل من `blockStreamingChunk.breakPreference`
  (`paragraph` → `\n\n`، `newline` → `\n`، `sentence` → مسافة).
- تتوفر تجاوزات القنوات عبر `*.blockStreamingCoalesce` (بما في ذلك إعدادات لكل حساب).
- يتم رفع القيمة الافتراضية لـ coalesce `minChars` إلى 1500 في Signal/Slack/Discord ما لم يتم تجاوزها.

## الإيقاع الشبيه بالبشر بين الكتل

عند تفعيل بث الكتل، يمكنك إضافة **توقف عشوائي** بين
ردود الكتل (بعد الكتلة الأولى). وهذا يجعل الردود متعددة الفقاعات تبدو
أكثر طبيعية.

- الإعداد: `agents.defaults.humanDelay` (يمكن تجاوزه لكل وكيل عبر `agents.list[].humanDelay`).
- الأوضاع: `off` (الافتراضي)، `natural` ‏(800–2500ms)، `custom` ‏(`minMs`/`maxMs`).
- ينطبق فقط على **ردود الكتل**، وليس الردود النهائية أو ملخصات الأدوات.

## "بث الأجزاء أم كل شيء"

يُربط هذا إلى:

- **بث الأجزاء:** ‏`blockStreamingDefault: "on"` + ‏`blockStreamingBreak: "text_end"` (إصدار أثناء التقدم). كما تحتاج القنوات غير Telegram أيضًا إلى `*.blockStreaming: true`.
- **بث كل شيء في النهاية:** ‏`blockStreamingBreak: "message_end"` (تفريغ مرة واحدة، وربما عدة أجزاء إذا كان طويلًا جدًا).
- **لا يوجد بث كتل:** ‏`blockStreamingDefault: "off"` (الرد النهائي فقط).

**ملاحظة خاصة بالقنوات:** يكون بث الكتل **معطّلًا ما لم**
يتم ضبط `*.blockStreaming` صراحةً على `true`. ويمكن للقنوات بث معاينة مباشرة
(`channels.<channel>.streaming`) من دون ردود كتل.

تذكير بموقع الإعداد: توجد القيم الافتراضية `blockStreaming*` تحت
`agents.defaults`، وليس في جذر الإعداد.

## أوضاع بث المعاينة

المفتاح الأساسي: `channels.<channel>.streaming`

الأوضاع:

- `off`: تعطيل بث المعاينة.
- `partial`: معاينة واحدة يتم استبدالها بأحدث نص.
- `block`: تحديثات معاينة مجزأة/ملحقة.
- `progress`: معاينة تقدم/حالة أثناء التوليد، ثم الإجابة النهائية عند الاكتمال.

### ربط القنوات

| القناة | `off` | `partial` | `block` | `progress`        |
| -------- | ----- | --------- | ------- | ----------------- |
| Telegram | ✅    | ✅        | ✅      | يُربط إلى `partial` |
| Discord  | ✅    | ✅        | ✅      | يُربط إلى `partial` |
| Slack    | ✅    | ✅        | ✅      | ✅                |

خاص بـ Slack فقط:

- يبدّل `channels.slack.nativeStreaming` استدعاءات API الخاصة بالبث الأصلي في Slack عندما تكون `streaming=partial` (الافتراضي: `true`).

ترحيل المفاتيح القديمة:

- Telegram: تتم الترحيل التلقائي لـ `streamMode` وboolean `streaming` إلى enum `streaming`.
- Discord: تتم الترحيل التلقائي لـ `streamMode` وboolean `streaming` إلى enum `streaming`.
- Slack: تتم الترحيل التلقائي لـ `streamMode` إلى enum `streaming`؛ وتتم الترحيل التلقائي لـ boolean `streaming` إلى `nativeStreaming`.

### سلوك وقت التشغيل

Telegram:

- يستخدم `sendMessage` + `editMessageText` لتحديثات المعاينة عبر الرسائل المباشرة والمجموعات/الموضوعات.
- يتم تخطي بث المعاينة عندما يكون بث كتل Telegram مفعّلًا صراحةً (لتجنب البث المزدوج).
- يمكن لـ `/reasoning stream` كتابة الاستدلال في المعاينة.

Discord:

- يستخدم رسائل معاينة من نوع send + edit.
- يستخدم وضع `block` تجزئة المسودة (`draftChunk`).
- يتم تخطي بث المعاينة عندما يكون بث كتل Discord مفعّلًا صراحةً.

Slack:

- يمكن أن يستخدم `partial` البث الأصلي في Slack (`chat.startStream`/`append`/`stop`) عندما يكون متاحًا.
- يستخدم `block` معاينات مسودة بنمط الإلحاق.
- يستخدم `progress` نص معاينة للحالة، ثم الإجابة النهائية.

## ذو صلة

- [الرسائل](/concepts/messages) — دورة حياة الرسالة وتسليمها
- [إعادة المحاولة](/concepts/retry) — سلوك إعادة المحاولة عند فشل التسليم
- [القنوات](/channels) — دعم البث لكل قناة
