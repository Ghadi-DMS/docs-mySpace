---
read_when:
    - أنت تريد تكوين موفري بحث الذاكرة أو نماذج التضمين
    - أنت تريد إعداد الواجهة الخلفية لـ QMD
    - أنت تريد ضبط البحث الهجين أو MMR أو الاضمحلال الزمني
    - أنت تريد تمكين فهرسة الذاكرة متعددة الوسائط
summary: جميع مقابض الإعداد الخاصة ببحث الذاكرة، وموفري التضمين، وQMD، والبحث الهجين، والفهرسة متعددة الوسائط
title: مرجع إعدادات الذاكرة
x-i18n:
    generated_at: "2026-04-10T07:18:03Z"
    model: gpt-5.4
    provider: openai
    source_hash: 5f9076bdfad95b87bd70625821bf401326f8eaeb53842b70823881419dbe43cb
    source_path: reference/memory-config.md
    workflow: 15
---

# مرجع إعدادات الذاكرة

تسرد هذه الصفحة كل مقابض الإعداد الخاصة ببحث ذاكرة OpenClaw. للاطلاع على نظرات
عامة مفاهيمية، راجع:

- [نظرة عامة على الذاكرة](/ar/concepts/memory) -- كيف تعمل الذاكرة
- [المحرك المدمج](/ar/concepts/memory-builtin) -- الواجهة الخلفية الافتراضية لـ SQLite
- [محرك QMD](/ar/concepts/memory-qmd) -- sidecar محلي أولًا
- [بحث الذاكرة](/ar/concepts/memory-search) -- مسار البحث والضبط
- [الذاكرة النشطة](/ar/concepts/active-memory) -- تمكين الوكيل الفرعي للذاكرة في الجلسات التفاعلية

توجد جميع إعدادات بحث الذاكرة تحت `agents.defaults.memorySearch` في
`openclaw.json` ما لم يُذكر خلاف ذلك.

إذا كنت تبحث عن مفتاح تبديل ميزة **الذاكرة النشطة** وإعدادات الوكيل الفرعي،
فهي موجودة تحت `plugins.entries.active-memory` بدلًا من `memorySearch`.

تستخدم الذاكرة النشطة نموذجًا ببوابتين:

1. يجب أن يكون الـ plugin مُمكّنًا ويستهدف معرّف الوكيل الحالي
2. يجب أن يكون الطلب جلسة دردشة تفاعلية دائمة ومؤهلة

راجع [الذاكرة النشطة](/ar/concepts/active-memory) لمعرفة نموذج التفعيل،
والإعدادات المملوكة للـ plugin، واستمرارية النص الحواري، ونمط الطرح الآمن.

---

## اختيار الموفّر

| المفتاح   | النوع      | الافتراضي      | الوصف                                                                                       |
| --------- | ---------- | -------------- | ------------------------------------------------------------------------------------------- |
| `provider` | `string`  | مُكتشف تلقائيًا | معرّف محول التضمين: `openai`، `gemini`، `voyage`، `mistral`، `bedrock`، `ollama`، `local` |
| `model`    | `string`  | موفر افتراضي   | اسم نموذج التضمين                                                                            |
| `fallback` | `string`  | `"none"`       | معرّف المحول الاحتياطي عند فشل الأساسي                                                       |
| `enabled`  | `boolean` | `true`         | تمكين بحث الذاكرة أو تعطيله                                                                  |

### ترتيب الاكتشاف التلقائي

عندما لا يتم تعيين `provider`، يختار OpenClaw أول خيار متاح:

1. `local` -- إذا كان `memorySearch.local.modelPath` مُعدًا والملف موجودًا.
2. `openai` -- إذا أمكن حل مفتاح OpenAI.
3. `gemini` -- إذا أمكن حل مفتاح Gemini.
4. `voyage` -- إذا أمكن حل مفتاح Voyage.
5. `mistral` -- إذا أمكن حل مفتاح Mistral.
6. `bedrock` -- إذا تمكّنت سلسلة بيانات الاعتماد الافتراضية لـ AWS SDK من الحل (دور المثيل، أو مفاتيح الوصول، أو الملف الشخصي، أو SSO، أو هوية الويب، أو الإعدادات المشتركة).

`ollama` مدعوم لكنه لا يُكتشف تلقائيًا (قم بتعيينه صراحةً).

### حل مفتاح API

تتطلب التضمينات البعيدة مفتاح API. يستخدم Bedrock بدلًا من ذلك سلسلة
بيانات الاعتماد الافتراضية لـ AWS SDK (أدوار المثيل، وSSO، ومفاتيح الوصول).

| الموفّر | متغير env                      | مفتاح الإعداد                      |
| ------- | ------------------------------ | ---------------------------------- |
| OpenAI  | `OPENAI_API_KEY`               | `models.providers.openai.apiKey`   |
| Gemini  | `GEMINI_API_KEY`               | `models.providers.google.apiKey`   |
| Voyage  | `VOYAGE_API_KEY`               | `models.providers.voyage.apiKey`   |
| Mistral | `MISTRAL_API_KEY`              | `models.providers.mistral.apiKey`  |
| Bedrock | سلسلة بيانات اعتماد AWS        | لا حاجة إلى مفتاح API              |
| Ollama  | `OLLAMA_API_KEY` (عنصر نائبي) | --                                 |

يغطي Codex OAuth الدردشة/الإكمالات فقط ولا يلبّي طلبات التضمين.

---

## إعدادات نقطة النهاية البعيدة

لنقاط النهاية المخصصة المتوافقة مع OpenAI أو لتجاوز الإعدادات الافتراضية للموفر:

| المفتاح           | النوع    | الوصف                                              |
| ----------------- | -------- | -------------------------------------------------- |
| `remote.baseUrl` | `string` | عنوان URL مخصص لقاعدة API                          |
| `remote.apiKey`  | `string` | تجاوز مفتاح API                                    |
| `remote.headers` | `object` | رؤوس HTTP إضافية (تُدمج مع الإعدادات الافتراضية للموفر) |

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "openai",
        model: "text-embedding-3-small",
        remote: {
          baseUrl: "https://api.example.com/v1/",
          apiKey: "YOUR_KEY",
        },
      },
    },
  },
}
```

---

## إعدادات خاصة بـ Gemini

| المفتاح                | النوع    | الافتراضي             | الوصف                                        |
| ---------------------- | -------- | --------------------- | -------------------------------------------- |
| `model`                | `string` | `gemini-embedding-001` | يدعم أيضًا `gemini-embedding-2-preview`      |
| `outputDimensionality` | `number` | `3072`                | بالنسبة إلى Embedding 2: ‏768 أو 1536 أو 3072 |

<Warning>
يؤدي تغيير النموذج أو `outputDimensionality` إلى إعادة فهرسة كاملة تلقائية.
</Warning>

---

## إعدادات تضمين Bedrock

يستخدم Bedrock سلسلة بيانات الاعتماد الافتراضية لـ AWS SDK -- ولا حاجة إلى
مفاتيح API.
إذا كان OpenClaw يعمل على EC2 مع دور مثيل Bedrock مُمكّن، فما عليك سوى تعيين
الموفر والنموذج:

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "bedrock",
        model: "amazon.titan-embed-text-v2:0",
      },
    },
  },
}
```

| المفتاح                | النوع    | الافتراضي                     | الوصف                          |
| ---------------------- | -------- | ----------------------------- | ------------------------------ |
| `model`                | `string` | `amazon.titan-embed-text-v2:0` | أي معرّف نموذج تضمين Bedrock   |
| `outputDimensionality` | `number` | الإعداد الافتراضي للنموذج     | بالنسبة إلى Titan V2: ‏256 أو 512 أو 1024 |

### النماذج المدعومة

النماذج التالية مدعومة (مع اكتشاف العائلة والإعدادات الافتراضية للأبعاد):

| معرّف النموذج                                | الموفّر    | الأبعاد الافتراضية | الأبعاد القابلة للتهيئة |
| ------------------------------------------- | ---------- | ------------------ | ----------------------- |
| `amazon.titan-embed-text-v2:0`              | Amazon     | 1024               | 256، 512، 1024          |
| `amazon.titan-embed-text-v1`                | Amazon     | 1536               | --                      |
| `amazon.titan-embed-g1-text-02`             | Amazon     | 1536               | --                      |
| `amazon.titan-embed-image-v1`               | Amazon     | 1024               | --                      |
| `amazon.nova-2-multimodal-embeddings-v1:0`  | Amazon     | 1024               | 256، 384، 1024، 3072    |
| `cohere.embed-english-v3`                   | Cohere     | 1024               | --                      |
| `cohere.embed-multilingual-v3`              | Cohere     | 1024               | --                      |
| `cohere.embed-v4:0`                         | Cohere     | 1536               | 256-1536                |
| `twelvelabs.marengo-embed-3-0-v1:0`         | TwelveLabs | 512                | --                      |
| `twelvelabs.marengo-embed-2-7-v1:0`         | TwelveLabs | 1024               | --                      |

المتغيرات ذات اللاحقة الخاصة بالإنتاجية (مثل `amazon.titan-embed-text-v1:2:8k`) ترث
إعدادات النموذج الأساسي.

### المصادقة

تستخدم مصادقة Bedrock ترتيب حل بيانات الاعتماد القياسي في AWS SDK:

1. متغيرات البيئة (`AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY`)
2. ذاكرة التخزين المؤقت لرمز SSO
3. بيانات اعتماد رمز هوية الويب
4. ملفات بيانات الاعتماد والإعدادات المشتركة
5. بيانات اعتماد بيانات ECS أو EC2 الوصفية

تُحلّ المنطقة من `AWS_REGION` أو `AWS_DEFAULT_REGION` أو
`baseUrl` الخاص بموفر `amazon-bedrock`، أو تكون افتراضيًا `us-east-1`.

### أذونات IAM

يحتاج دور IAM أو المستخدم إلى:

```json
{
  "Effect": "Allow",
  "Action": "bedrock:InvokeModel",
  "Resource": "*"
}
```

ولتطبيق أقل قدر من الامتيازات، قيّد `InvokeModel` على النموذج المحدد:

```
arn:aws:bedrock:*::foundation-model/amazon.titan-embed-text-v2:0
```

---

## إعدادات التضمين المحلي

| المفتاح               | النوع    | الافتراضي              | الوصف                         |
| --------------------- | -------- | ---------------------- | ----------------------------- |
| `local.modelPath`     | `string` | يُنزَّل تلقائيًا        | المسار إلى ملف نموذج GGUF     |
| `local.modelCacheDir` | `string` | الإعداد الافتراضي لـ node-llama-cpp | دليل التخزين المؤقت للنماذج المُنزَّلة |

النموذج الافتراضي: `embeddinggemma-300m-qat-Q8_0.gguf` ‏(~0.6 جيجابايت، يُنزَّل تلقائيًا).
يتطلب بناءً أصليًا: `pnpm approve-builds` ثم `pnpm rebuild node-llama-cpp`.

---

## إعدادات البحث الهجين

كلها تحت `memorySearch.query.hybrid`:

| المفتاح               | النوع      | الافتراضي | الوصف                               |
| --------------------- | --------- | --------- | ----------------------------------- |
| `enabled`             | `boolean` | `true`    | تمكين بحث BM25 + المتجهات الهجين    |
| `vectorWeight`        | `number`  | `0.7`     | وزن درجات المتجهات (0-1)            |
| `textWeight`          | `number`  | `0.3`     | وزن درجات BM25 ‏(0-1)               |
| `candidateMultiplier` | `number`  | `4`       | مضاعف حجم مجموعة المرشحين           |

### MMR (التنوع)

| المفتاح       | النوع      | الافتراضي | الوصف                                  |
| ------------- | --------- | --------- | -------------------------------------- |
| `mmr.enabled` | `boolean` | `false`   | تمكين إعادة الترتيب باستخدام MMR       |
| `mmr.lambda`  | `number`  | `0.7`     | 0 = أقصى تنوع، 1 = أقصى صلة            |

### الاضمحلال الزمني (الحداثة)

| المفتاح                     | النوع      | الافتراضي | الوصف                          |
| --------------------------- | --------- | --------- | ------------------------------ |
| `temporalDecay.enabled`     | `boolean` | `false`   | تمكين تعزيز الحداثة            |
| `temporalDecay.halfLifeDays` | `number`  | `30`      | تنخفض الدرجة إلى النصف كل N يومًا |

لا يُطبَّق الاضمحلال أبدًا على الملفات الدائمة (`MEMORY.md` والملفات غير المؤرخة في `memory/`).

### مثال كامل

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        query: {
          hybrid: {
            vectorWeight: 0.7,
            textWeight: 0.3,
            mmr: { enabled: true, lambda: 0.7 },
            temporalDecay: { enabled: true, halfLifeDays: 30 },
          },
        },
      },
    },
  },
}
```

---

## مسارات ذاكرة إضافية

| المفتاح     | النوع       | الوصف                                      |
| ----------- | ---------- | ------------------------------------------ |
| `extraPaths` | `string[]` | أدلة أو ملفات إضافية للفهرسة               |

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        extraPaths: ["../team-docs", "/srv/shared-notes"],
      },
    },
  },
}
```

يمكن أن تكون المسارات مطلقة أو نسبية إلى مساحة العمل. تُفحَص الأدلة
بشكل تكراري بحثًا عن ملفات `.md`. وتعتمد معالجة الروابط الرمزية على
الواجهة الخلفية النشطة:
فالمحرك المدمج يتجاهل الروابط الرمزية، بينما يتبع QMD سلوك ماسح QMD الأساسي.

للبحث في النصوص الحوارية بين الوكلاء ضمن نطاق الوكيل، استخدم
`agents.list[].memorySearch.qmd.extraCollections` بدلًا من `memory.qmd.paths`.
تتبع هذه المجموعات الإضافية الشكل نفسه `{ path, name, pattern? }`، لكن
يتم دمجها لكل وكيل ويمكنها الحفاظ على الأسماء المشتركة الصريحة عندما يشير
المسار إلى خارج مساحة العمل الحالية.
إذا ظهر نفس المسار المحلول في كل من `memory.qmd.paths` و
`memorySearch.qmd.extraCollections`، فإن QMD يحتفظ بالإدخال الأول ويتخطى
الإدخال المكرر.

---

## الذاكرة متعددة الوسائط (Gemini)

افهرس الصور والصوت إلى جانب Markdown باستخدام Gemini Embedding 2:

| المفتاح                   | النوع       | الافتراضي  | الوصف                                   |
| ------------------------- | ---------- | ---------- | --------------------------------------- |
| `multimodal.enabled`      | `boolean`  | `false`    | تمكين الفهرسة متعددة الوسائط             |
| `multimodal.modalities`   | `string[]` | --         | `["image"]` أو `["audio"]` أو `["all"]` |
| `multimodal.maxFileBytes` | `number`   | `10000000` | الحد الأقصى لحجم الملف من أجل الفهرسة    |

ينطبق هذا فقط على الملفات الموجودة في `extraPaths`. تظل جذور الذاكرة الافتراضية
خاصة بـ Markdown فقط.
يتطلب `gemini-embedding-2-preview`. يجب أن تكون قيمة `fallback` هي `"none"`.

التنسيقات المدعومة: `.jpg` و`.jpeg` و`.png` و`.webp` و`.gif` و`.heic` و`.heif`
(صور)؛ و`.mp3` و`.wav` و`.ogg` و`.opus` و`.m4a` و`.aac` و`.flac` (صوت).

---

## ذاكرة التخزين المؤقت للتضمين

| المفتاح            | النوع      | الافتراضي | الوصف                             |
| ------------------ | --------- | --------- | --------------------------------- |
| `cache.enabled`    | `boolean` | `false`   | تخزين تضمينات المقاطع مؤقتًا في SQLite |
| `cache.maxEntries` | `number`  | `50000`   | الحد الأقصى للتضمينات المخزنة مؤقتًا    |

يمنع إعادة تضمين النص غير المتغير أثناء إعادة الفهرسة أو تحديثات النص الحواري.

---

## الفهرسة على دفعات

| المفتاح                       | النوع      | الافتراضي | الوصف                     |
| ----------------------------- | --------- | --------- | ------------------------- |
| `remote.batch.enabled`        | `boolean` | `false`   | تمكين API التضمين على دفعات |
| `remote.batch.concurrency`    | `number`  | `2`       | مهام الدفعات المتوازية     |
| `remote.batch.wait`           | `boolean` | `true`    | انتظار اكتمال الدفعة       |
| `remote.batch.pollIntervalMs` | `number`  | --        | فاصل الاستطلاع            |
| `remote.batch.timeoutMinutes` | `number`  | --        | مهلة الدفعة               |

متاح لـ `openai` و`gemini` و`voyage`. تكون الدفعات في OpenAI عادةً
الأسرع والأقل تكلفة لعمليات التعبئة الخلفية الكبيرة.

---

## بحث ذاكرة الجلسة (تجريبي)

افهرس نصوص الجلسات الحوارية وأظهرها عبر `memory_search`:

| المفتاح                      | النوع       | الافتراضي   | الوصف                                        |
| ---------------------------- | ---------- | ----------- | -------------------------------------------- |
| `experimental.sessionMemory` | `boolean`  | `false`     | تمكين فهرسة الجلسة                           |
| `sources`                    | `string[]` | `["memory"]` | أضف `"sessions"` لتضمين النصوص الحوارية      |
| `sync.sessions.deltaBytes`   | `number`   | `100000`    | حد البايت لإعادة الفهرسة                     |
| `sync.sessions.deltaMessages` | `number`  | `50`        | حد الرسائل لإعادة الفهرسة                    |

فهرسة الجلسات اختيارية وتعمل بشكل غير متزامن. قد تكون النتائج
قديمة قليلًا. توجد سجلات الجلسات على القرص، لذا تعامل مع الوصول إلى نظام
الملفات باعتباره حد الثقة.

---

## تسريع المتجهات في SQLite ‏(sqlite-vec)

| المفتاح                    | النوع      | الافتراضي | الوصف                              |
| -------------------------- | --------- | --------- | ---------------------------------- |
| `store.vector.enabled`     | `boolean` | `true`    | استخدام sqlite-vec لاستعلامات المتجهات |
| `store.vector.extensionPath` | `string` | bundled   | تجاوز مسار sqlite-vec              |

عندما لا يكون sqlite-vec متاحًا، يعود OpenClaw تلقائيًا إلى تشابه جيب
التمام داخل العملية.

---

## تخزين الفهرس

| المفتاح               | النوع    | الافتراضي                            | الوصف                                        |
| --------------------- | -------- | ------------------------------------ | -------------------------------------------- |
| `store.path`          | `string` | `~/.openclaw/memory/{agentId}.sqlite` | موقع الفهرس (يدعم الرمز `{agentId}`)         |
| `store.fts.tokenizer` | `string` | `unicode61`                          | محلل FTS5 ‏(`unicode61` أو `trigram`)        |

---

## إعدادات الواجهة الخلفية QMD

عيّن `memory.backend = "qmd"` للتمكين. توجد جميع إعدادات QMD تحت
`memory.qmd`:

| المفتاح                 | النوع      | الافتراضي | الوصف                                         |
| ----------------------- | --------- | --------- | --------------------------------------------- |
| `command`               | `string`  | `qmd`     | مسار الملف التنفيذي لـ QMD                    |
| `searchMode`            | `string`  | `search`  | أمر البحث: `search` أو `vsearch` أو `query`   |
| `includeDefaultMemory`  | `boolean` | `true`    | فهرسة `MEMORY.md` و`memory/**/*.md` تلقائيًا  |
| `paths[]`               | `array`   | --        | مسارات إضافية: `{ name, path, pattern? }`     |
| `sessions.enabled`      | `boolean` | `false`   | فهرسة نصوص الجلسات الحوارية                   |
| `sessions.retentionDays` | `number` | --        | مدة الاحتفاظ بالنصوص الحوارية                 |
| `sessions.exportDir`    | `string`  | --        | دليل التصدير                                  |

يفضّل OpenClaw الأشكال الحالية لمجموعات QMD واستعلامات MCP، لكنه يحافظ
على عمل إصدارات QMD الأقدم عبر الرجوع إلى أعلام المجموعات القديمة `--mask`
وأسماء أدوات MCP الأقدم عند الحاجة.

تظل تجاوزات نموذج QMD ضمن جانب QMD، وليس إعدادات OpenClaw. إذا كنت بحاجة إلى
تجاوز نماذج QMD على مستوى عام، فعيّن متغيرات البيئة مثل
`QMD_EMBED_MODEL` و`QMD_RERANK_MODEL` و`QMD_GENERATE_MODEL` في
بيئة تشغيل Gateway.

### جدول التحديث

| المفتاح                  | النوع      | الافتراضي | الوصف                                 |
| ------------------------ | --------- | --------- | ------------------------------------- |
| `update.interval`        | `string`  | `5m`      | فاصل التحديث                          |
| `update.debounceMs`      | `number`  | `15000`   | إزالة اهتزاز تغييرات الملفات          |
| `update.onBoot`          | `boolean` | `true`    | التحديث عند بدء التشغيل               |
| `update.waitForBootSync` | `boolean` | `false`   | حظر بدء التشغيل حتى اكتمال التحديث    |
| `update.embedInterval`   | `string`  | --        | وتيرة تضمين منفصلة                    |
| `update.commandTimeoutMs` | `number` | --        | مهلة أوامر QMD                        |
| `update.updateTimeoutMs` | `number`  | --        | مهلة عمليات تحديث QMD                 |
| `update.embedTimeoutMs`  | `number`  | --        | مهلة عمليات تضمين QMD                 |

### الحدود

| المفتاح                  | النوع     | الافتراضي | الوصف                         |
| ------------------------ | -------- | --------- | ----------------------------- |
| `limits.maxResults`      | `number` | `6`       | الحد الأقصى لنتائج البحث      |
| `limits.maxSnippetChars` | `number` | --        | تقييد طول المقتطف             |
| `limits.maxInjectedChars` | `number` | --       | تقييد إجمالي الأحرف المحقونة  |
| `limits.timeoutMs`       | `number` | `4000`    | مهلة البحث                    |

### النطاق

يتحكم في الجلسات التي يمكنها تلقي نتائج بحث QMD. البنية نفسها كما في
[`session.sendPolicy`](/ar/gateway/configuration-reference#session):

```json5
{
  memory: {
    qmd: {
      scope: {
        default: "deny",
        rules: [{ action: "allow", match: { chatType: "direct" } }],
      },
    },
  },
}
```

الافتراضي هو الرسائل المباشرة فقط. يطابق `match.keyPrefix` مفتاح الجلسة
المطبّع؛ ويطابق `match.rawKeyPrefix` المفتاح الخام بما في ذلك `agent:<id>:`.

### الاستشهادات

ينطبق `memory.citations` على جميع الواجهات الخلفية:

| القيمة           | السلوك                                                |
| ---------------- | ----------------------------------------------------- |
| `auto` (الافتراضي) | تضمين تذييل `Source: <path#line>` في المقتطفات       |
| `on`             | تضمين التذييل دائمًا                                  |
| `off`            | حذف التذييل (مع استمرار تمرير المسار إلى الوكيل داخليًا) |

### مثال كامل على QMD

```json5
{
  memory: {
    backend: "qmd",
    citations: "auto",
    qmd: {
      includeDefaultMemory: true,
      update: { interval: "5m", debounceMs: 15000 },
      limits: { maxResults: 6, timeoutMs: 4000 },
      scope: {
        default: "deny",
        rules: [{ action: "allow", match: { chatType: "direct" } }],
      },
      paths: [{ name: "docs", path: "~/notes", pattern: "**/*.md" }],
    },
  },
}
```

---

## Dreaming ‏(تجريبي)

يُضبط Dreaming تحت `plugins.entries.memory-core.config.dreaming`،
وليس تحت `agents.defaults.memorySearch`.

يعمل Dreaming كعملية اجتياز مجدولة واحدة ويستخدم مراحل light/deep/REM
الداخلية كتفصيل تنفيذي.

للاطلاع على السلوك المفاهيمي وأوامر الشرطة المائلة، راجع [Dreaming](/ar/concepts/dreaming).

### إعدادات المستخدم

| المفتاح     | النوع      | الافتراضي   | الوصف                                           |
| ----------- | --------- | ----------- | ----------------------------------------------- |
| `enabled`   | `boolean` | `false`     | تمكين Dreaming بالكامل أو تعطيله                |
| `frequency` | `string`  | `0 3 * * *` | وتيرة cron اختيارية لاجتياز Dreaming الكامل     |

### مثال

```json5
{
  plugins: {
    entries: {
      "memory-core": {
        config: {
          dreaming: {
            enabled: true,
            frequency: "0 3 * * *",
          },
        },
      },
    },
  },
}
```

ملاحظات:

- يكتب Dreaming حالة الآلة إلى `memory/.dreams/`.
- يكتب Dreaming المخرجات السردية المقروءة بشريًا إلى `DREAMS.md` (أو `dreams.md` الموجود).
- سياسة مراحل light/deep/REM والحدود الخاصة بها سلوك داخلي، وليست إعدادات موجهة للمستخدم.
