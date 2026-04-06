---
read_when:
    - أنت تريد تهيئة موفري بحث الذاكرة أو نماذج التضمين
    - أنت تريد إعداد الواجهة الخلفية QMD
    - أنت تريد ضبط البحث الهجين أو MMR أو الاضمحلال الزمني
    - أنت تريد تفعيل فهرسة الذاكرة متعددة الوسائط
summary: جميع خيارات الإعداد لبحث الذاكرة، وموفري التضمين، وQMD، والبحث الهجين، والفهرسة متعددة الوسائط
title: مرجع إعدادات الذاكرة
x-i18n:
    generated_at: "2026-04-06T03:13:12Z"
    model: gpt-5.4
    provider: openai
    source_hash: 0de0b85125443584f4e575cf673ca8d9bd12ecd849d73c537f4a17545afa93fd
    source_path: reference/memory-config.md
    workflow: 15
---

# مرجع إعدادات الذاكرة

تسرد هذه الصفحة كل خيارات الإعداد الخاصة ببحث الذاكرة في OpenClaw. للحصول على
نظرات عامة مفاهيمية، راجع:

- [نظرة عامة على الذاكرة](/ar/concepts/memory) -- كيف تعمل الذاكرة
- [المحرك المدمج](/ar/concepts/memory-builtin) -- الواجهة الخلفية الافتراضية لـ SQLite
- [محرك QMD](/ar/concepts/memory-qmd) -- خدمة جانبية محلية أولًا
- [بحث الذاكرة](/ar/concepts/memory-search) -- مسار البحث والضبط

توجد جميع إعدادات بحث الذاكرة ضمن `agents.defaults.memorySearch` في
`openclaw.json` ما لم يُذكر خلاف ذلك.

---

## اختيار الموفر

| المفتاح        | النوع      | الافتراضي          | الوصف                                                                                 |
| ---------- | --------- | ---------------- | ------------------------------------------------------------------------------------------- |
| `provider` | `string`  | يُكتشف تلقائيًا    | معرّف محول التضمين: `openai` أو `gemini` أو `voyage` أو `mistral` أو `bedrock` أو `ollama` أو `local` |
| `model`    | `string`  | افتراضي الموفر | اسم نموذج التضمين                                                                        |
| `fallback` | `string`  | `"none"`         | معرّف المحول الاحتياطي عند فشل الأساسي                                                  |
| `enabled`  | `boolean` | `true`           | تفعيل أو تعطيل بحث الذاكرة                                                             |

### ترتيب الاكتشاف التلقائي

عندما لا يتم تعيين `provider`، يختار OpenClaw أول خيار متاح:

1. `local` -- إذا كان `memorySearch.local.modelPath` مهيأ وكان الملف موجودًا.
2. `openai` -- إذا أمكن حل مفتاح OpenAI.
3. `gemini` -- إذا أمكن حل مفتاح Gemini.
4. `voyage` -- إذا أمكن حل مفتاح Voyage.
5. `mistral` -- إذا أمكن حل مفتاح Mistral.
6. `bedrock` -- إذا تم حل سلسلة بيانات اعتماد AWS SDK (دور المثيل، أو مفاتيح الوصول، أو الملف الشخصي، أو SSO، أو هوية الويب، أو الإعدادات المشتركة).

النوع `ollama` مدعوم لكنه لا يُكتشف تلقائيًا (قم بتعيينه صراحة).

### حل مفتاح API

يتطلب التضمين البعيد مفتاح API. أما Bedrock فيستخدم سلسلة
بيانات الاعتماد الافتراضية لـ AWS SDK بدلًا من ذلك (أدوار المثيل، وSSO، ومفاتيح الوصول).

| الموفر | متغير البيئة                        | مفتاح الإعداد                        |
| -------- | ------------------------------ | --------------------------------- |
| OpenAI   | `OPENAI_API_KEY`               | `models.providers.openai.apiKey`  |
| Gemini   | `GEMINI_API_KEY`               | `models.providers.google.apiKey`  |
| Voyage   | `VOYAGE_API_KEY`               | `models.providers.voyage.apiKey`  |
| Mistral  | `MISTRAL_API_KEY`              | `models.providers.mistral.apiKey` |
| Bedrock  | سلسلة بيانات اعتماد AWS           | لا حاجة إلى مفتاح API                 |
| Ollama   | `OLLAMA_API_KEY` (عنصر نائب) | --                                |

يغطي Codex OAuth الدردشة/الإكمالات فقط ولا يلبّي طلبات التضمين.

---

## إعداد نقطة النهاية البعيدة

لنقاط النهاية المخصصة المتوافقة مع OpenAI أو لتجاوز إعدادات الموفر الافتراضية:

| المفتاح              | النوع     | الوصف                                        |
| ---------------- | -------- | -------------------------------------------------- |
| `remote.baseUrl` | `string` | عنوان URL أساسي مخصص لـ API                                |
| `remote.apiKey`  | `string` | تجاوز مفتاح API                                   |
| `remote.headers` | `object` | ترويسات HTTP إضافية (تُدمج مع الإعدادات الافتراضية للموفر) |

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

| المفتاح                    | النوع     | الافتراضي                | الوصف                                |
| ---------------------- | -------- | ---------------------- | ------------------------------------------ |
| `model`                | `string` | `gemini-embedding-001` | يدعم أيضًا `gemini-embedding-2-preview` |
| `outputDimensionality` | `number` | `3072`                 | بالنسبة إلى Embedding 2: ‏768 أو 1536 أو 3072        |

<Warning>
يؤدي تغيير `model` أو `outputDimensionality` إلى إعادة فهرسة كاملة تلقائية.
</Warning>

---

## إعدادات تضمين Bedrock

يستخدم Bedrock سلسلة بيانات الاعتماد الافتراضية لـ AWS SDK -- ولا يحتاج إلى مفاتيح API.
إذا كان OpenClaw يعمل على EC2 بدور مثيل مفعّل لـ Bedrock، فقط عيّن
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

| المفتاح                    | النوع     | الافتراضي                        | الوصف                     |
| ---------------------- | -------- | ------------------------------ | ------------------------------- |
| `model`                | `string` | `amazon.titan-embed-text-v2:0` | أي معرّف نموذج تضمين لـ Bedrock  |
| `outputDimensionality` | `number` | الافتراضي الخاص بالنموذج                  | بالنسبة إلى Titan V2: ‏256 أو 512 أو 1024 |

### النماذج المدعومة

النماذج التالية مدعومة (مع اكتشاف العائلة وقيم الأبعاد
الافتراضية):

| معرّف النموذج                                   | الموفر   | الأبعاد الافتراضية | الأبعاد القابلة للتهيئة    |
| ------------------------------------------ | ---------- | ------------ | -------------------- |
| `amazon.titan-embed-text-v2:0`             | Amazon     | 1024         | 256, 512, 1024       |
| `amazon.titan-embed-text-v1`               | Amazon     | 1536         | --                   |
| `amazon.titan-embed-g1-text-02`            | Amazon     | 1536         | --                   |
| `amazon.titan-embed-image-v1`              | Amazon     | 1024         | --                   |
| `amazon.nova-2-multimodal-embeddings-v1:0` | Amazon     | 1024         | 256, 384, 1024, 3072 |
| `cohere.embed-english-v3`                  | Cohere     | 1024         | --                   |
| `cohere.embed-multilingual-v3`             | Cohere     | 1024         | --                   |
| `cohere.embed-v4:0`                        | Cohere     | 1536         | 256-1536             |
| `twelvelabs.marengo-embed-3-0-v1:0`        | TwelveLabs | 512          | --                   |
| `twelvelabs.marengo-embed-2-7-v1:0`        | TwelveLabs | 1024         | --                   |

ترث المتغيرات ذات اللاحقة الخاصة بمعدل النقل (مثل `amazon.titan-embed-text-v1:2:8k`)
إعدادات النموذج الأساسي.

### المصادقة

تستخدم مصادقة Bedrock ترتيب حل بيانات الاعتماد القياسي في AWS SDK:

1. متغيرات البيئة (`AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY`)
2. ذاكرة التخزين المؤقت لرمز SSO
3. بيانات اعتماد رمز هوية الويب
4. ملفات بيانات الاعتماد والإعدادات المشتركة
5. بيانات اعتماد بيانات تعريف ECS أو EC2

يتم حل المنطقة من `AWS_REGION` أو `AWS_DEFAULT_REGION` أو
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

ولتقليل الصلاحيات إلى الحد الأدنى، احصر `InvokeModel` في النموذج المحدد:

```
arn:aws:bedrock:*::foundation-model/amazon.titan-embed-text-v2:0
```

---

## إعدادات التضمين المحلي

| المفتاح                   | النوع     | الافتراضي                | الوصف                     |
| --------------------- | -------- | ---------------------- | ------------------------------- |
| `local.modelPath`     | `string` | يُنزَّل تلقائيًا        | المسار إلى ملف نموذج GGUF         |
| `local.modelCacheDir` | `string` | افتراضي node-llama-cpp | دليل التخزين المؤقت للنماذج المنزّلة |

النموذج الافتراضي: `embeddinggemma-300m-qat-Q8_0.gguf` (~0.6 GB، يُنزَّل تلقائيًا).
يتطلب بناءً أصليًا: `pnpm approve-builds` ثم `pnpm rebuild node-llama-cpp`.

---

## إعدادات البحث الهجين

كلها ضمن `memorySearch.query.hybrid`:

| المفتاح                   | النوع      | الافتراضي | الوصف                        |
| --------------------- | --------- | ------- | ---------------------------------- |
| `enabled`             | `boolean` | `true`  | تفعيل البحث الهجين BM25 + المتجهات |
| `vectorWeight`        | `number`  | `0.7`   | وزن درجات المتجهات (0-1)     |
| `textWeight`          | `number`  | `0.3`   | وزن درجات BM25 (0-1)       |
| `candidateMultiplier` | `number`  | `4`     | مضاعف حجم مجموعة المرشحين     |

### MMR (التنوع)

| المفتاح           | النوع      | الافتراضي | الوصف                          |
| ------------- | --------- | ------- | ------------------------------------ |
| `mmr.enabled` | `boolean` | `false` | تفعيل إعادة الترتيب MMR                |
| `mmr.lambda`  | `number`  | `0.7`   | 0 = أقصى تنوع، 1 = أقصى صلة |

### الاضمحلال الزمني (الحداثة)

| المفتاح                          | النوع      | الافتراضي | الوصف               |
| ---------------------------- | --------- | ------- | ------------------------- |
| `temporalDecay.enabled`      | `boolean` | `false` | تفعيل تعزيز الحداثة      |
| `temporalDecay.halfLifeDays` | `number`  | `30`    | تنخفض الدرجة إلى النصف كل N أيام |

لا يتم تطبيق الاضمحلال أبدًا على الملفات الدائمة (`MEMORY.md` والملفات غير المؤرخة في `memory/`).

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

| المفتاح          | النوع       | الوصف                              |
| ------------ | ---------- | ---------------------------------------- |
| `extraPaths` | `string[]` | أدلة أو ملفات إضافية لفهرستها |

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

يمكن أن تكون المسارات مطلقة أو مرتبطة بمساحة العمل. يتم فحص الأدلة
بشكل递归 بحثًا عن ملفات `.md`. ويعتمد التعامل مع الروابط الرمزية على الواجهة الخلفية النشطة:
يتجاهل المحرك المدمج الروابط الرمزية، بينما يتبع QMD سلوك ماسح QMD
الأساسي.

بالنسبة إلى بحث نصوص العبور بين الوكلاء ضمن نطاق الوكيل، استخدم
`agents.list[].memorySearch.qmd.extraCollections` بدلًا من `memory.qmd.paths`.
تتبع هذه المجموعات الإضافية الشكل نفسه `{ path, name, pattern? }`، لكنها
تُدمج لكل وكيل ويمكنها الحفاظ على الأسماء المشتركة الصريحة عندما يشير المسار
إلى خارج مساحة العمل الحالية.
إذا ظهر المسار المحلول نفسه في كلٍّ من `memory.qmd.paths` و
`memorySearch.qmd.extraCollections`، يحتفظ QMD بالإدخال الأول ويتجاوز
الإدخال المكرر.

---

## الذاكرة متعددة الوسائط (Gemini)

قم بفهرسة الصور والصوت إلى جانب Markdown باستخدام Gemini Embedding 2:

| المفتاح                       | النوع       | الافتراضي    | الوصف                            |
| ------------------------- | ---------- | ---------- | -------------------------------------- |
| `multimodal.enabled`      | `boolean`  | `false`    | تفعيل الفهرسة متعددة الوسائط             |
| `multimodal.modalities`   | `string[]` | --         | `["image"]` أو `["audio"]` أو `["all"]` |
| `multimodal.maxFileBytes` | `number`   | `10000000` | الحد الأقصى لحجم الملف من أجل الفهرسة             |

ينطبق ذلك فقط على الملفات الموجودة ضمن `extraPaths`. وتبقى جذور الذاكرة الافتراضية خاصة بـ Markdown فقط.
يتطلب `gemini-embedding-2-preview`. ويجب أن تكون قيمة `fallback` هي `"none"`.

التنسيقات المدعومة: `.jpg` و `.jpeg` و `.png` و `.webp` و `.gif` و `.heic` و `.heif`
(صور)؛ و`.mp3` و `.wav` و `.ogg` و `.opus` و `.m4a` و `.aac` و `.flac` (صوت).

---

## التخزين المؤقت للتضمين

| المفتاح                | النوع      | الافتراضي | الوصف                      |
| ------------------ | --------- | ------- | -------------------------------- |
| `cache.enabled`    | `boolean` | `false` | تخزين تضمينات المقاطع مؤقتًا في SQLite |
| `cache.maxEntries` | `number`  | `50000` | الحد الأقصى للتضمينات المخزنة مؤقتًا            |

يمنع إعادة تضمين النص غير المتغير أثناء إعادة الفهرسة أو تحديثات النصوص.

---

## الفهرسة على دفعات

| المفتاح                           | النوع      | الافتراضي | الوصف                |
| ----------------------------- | --------- | ------- | -------------------------- |
| `remote.batch.enabled`        | `boolean` | `false` | تفعيل API التضمين الدفعي |
| `remote.batch.concurrency`    | `number`  | `2`     | مهام دفعية متوازية        |
| `remote.batch.wait`           | `boolean` | `true`  | انتظار اكتمال الدفعة  |
| `remote.batch.pollIntervalMs` | `number`  | --      | فترة الاستطلاع              |
| `remote.batch.timeoutMinutes` | `number`  | --      | مهلة الدفعة              |

متاح لـ `openai` و `gemini` و `voyage`. وعادةً ما تكون الدفعات في OpenAI
الأسرع والأقل تكلفة لعمليات التعبئة الكبيرة.

---

## بحث ذاكرة الجلسة (تجريبي)

قم بفهرسة نصوص الجلسات وإظهارها عبر `memory_search`:

| المفتاح                           | النوع       | الافتراضي      | الوصف                             |
| ----------------------------- | ---------- | ------------ | --------------------------------------- |
| `experimental.sessionMemory`  | `boolean`  | `false`      | تفعيل فهرسة الجلسات                 |
| `sources`                     | `string[]` | `["memory"]` | أضف `"sessions"` لتضمين النصوص |
| `sync.sessions.deltaBytes`    | `number`   | `100000`     | حد البايتات لإعادة الفهرسة              |
| `sync.sessions.deltaMessages` | `number`   | `50`         | حد الرسائل لإعادة الفهرسة           |

تكون فهرسة الجلسات اختيارية وتعمل بشكل غير متزامن. وقد تكون النتائج
قديمة قليلًا. وتوجد سجلات الجلسات على القرص، لذا تعامل مع الوصول إلى نظام الملفات
على أنه حد الثقة.

---

## تسريع المتجهات في SQLite ‏(sqlite-vec)

| المفتاح                          | النوع      | الافتراضي | الوصف                       |
| ---------------------------- | --------- | ------- | --------------------------------- |
| `store.vector.enabled`       | `boolean` | `true`  | استخدام sqlite-vec لاستعلامات المتجهات |
| `store.vector.extensionPath` | `string`  | مضمّن | تجاوز مسار sqlite-vec          |

عندما لا يكون sqlite-vec متاحًا، يعود OpenClaw تلقائيًا إلى
تشابه جيب التمام داخل العملية.

---

## تخزين الفهرس

| المفتاح                   | النوع     | الافتراضي                               | الوصف                                 |
| --------------------- | -------- | ------------------------------------- | ------------------------------------------- |
| `store.path`          | `string` | `~/.openclaw/memory/{agentId}.sqlite` | موقع الفهرس (يدعم الرمز `{agentId}`) |
| `store.fts.tokenizer` | `string` | `unicode61`                           | محلل FTS5 (`unicode61` أو `trigram`)   |

---

## إعدادات الواجهة الخلفية QMD

قم بتعيين `memory.backend = "qmd"` للتفعيل. وتوجد جميع إعدادات QMD ضمن
`memory.qmd`:

| المفتاح                      | النوع      | الافتراضي  | الوصف                                  |
| ------------------------ | --------- | -------- | -------------------------------------------- |
| `command`                | `string`  | `qmd`    | مسار الملف التنفيذي QMD                          |
| `searchMode`             | `string`  | `search` | أمر البحث: `search` أو `vsearch` أو `query` |
| `includeDefaultMemory`   | `boolean` | `true`   | فهرسة `MEMORY.md` و `memory/**/*.md` تلقائيًا    |
| `paths[]`                | `array`   | --       | مسارات إضافية: `{ name, path, pattern? }`      |
| `sessions.enabled`       | `boolean` | `false`  | فهرسة نصوص الجلسات                    |
| `sessions.retentionDays` | `number`  | --       | الاحتفاظ بالنصوص                         |
| `sessions.exportDir`     | `string`  | --       | دليل التصدير                             |

يفضّل OpenClaw أشكال مجموعات QMD الحالية واستعلامات MCP الحالية، لكنه يبقي
إصدارات QMD الأقدم عاملة من خلال الرجوع إلى علامات المجموعات القديمة `--mask`
وأسماء أدوات MCP الأقدم عند الحاجة.

تبقى تجاوزات نماذج QMD على جانب QMD، وليس في إعدادات OpenClaw. وإذا كنت بحاجة إلى
تجاوز نماذج QMD عالميًا، فقم بتعيين متغيرات بيئة مثل
`QMD_EMBED_MODEL` و `QMD_RERANK_MODEL` و `QMD_GENERATE_MODEL` في
بيئة وقت تشغيل البوابة.

### جدول التحديث

| المفتاح                       | النوع      | الافتراضي | الوصف                           |
| ------------------------- | --------- | ------- | ------------------------------------- |
| `update.interval`         | `string`  | `5m`    | فترة التحديث                      |
| `update.debounceMs`       | `number`  | `15000` | إزالة الارتداد من تغييرات الملفات                 |
| `update.onBoot`           | `boolean` | `true`  | التحديث عند بدء التشغيل                    |
| `update.waitForBootSync`  | `boolean` | `false` | حظر بدء التشغيل حتى يكتمل التحديث |
| `update.embedInterval`    | `string`  | --      | وتيرة تضمين منفصلة                |
| `update.commandTimeoutMs` | `number`  | --      | مهلة أوامر QMD              |
| `update.updateTimeoutMs`  | `number`  | --      | مهلة عمليات تحديث QMD     |
| `update.embedTimeoutMs`   | `number`  | --      | مهلة عمليات تضمين QMD      |

### الحدود

| المفتاح                       | النوع     | الافتراضي | الوصف                |
| ------------------------- | -------- | ------- | -------------------------- |
| `limits.maxResults`       | `number` | `6`     | الحد الأقصى لنتائج البحث         |
| `limits.maxSnippetChars`  | `number` | --      | تقييد طول المقتطف       |
| `limits.maxInjectedChars` | `number` | --      | تقييد إجمالي الأحرف المحقونة |
| `limits.timeoutMs`        | `number` | `4000`  | مهلة البحث             |

### النطاق

يتحكم في الجلسات التي يمكنها تلقي نتائج بحث QMD. وهو بنفس مخطط
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

الافتراضي هو الرسائل المباشرة فقط. تطابق `match.keyPrefix` مفتاح الجلسة المطبّع؛
وتطابق `match.rawKeyPrefix` المفتاح الخام بما في ذلك `agent:<id>:`.

### الاستشهادات

تنطبق `memory.citations` على جميع الواجهات الخلفية:

| القيمة            | السلوك                                            |
| ---------------- | --------------------------------------------------- |
| `auto` (افتراضي) | تضمين تذييل `Source: <path#line>` في المقتطفات    |
| `on`             | تضمين التذييل دائمًا                               |
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

## Dreaming (تجريبي)

يتم إعداد Dreaming ضمن `plugins.entries.memory-core.config.dreaming`,
وليس ضمن `agents.defaults.memorySearch`.

يعمل Dreaming كمسح مجدول واحد ويستخدم المراحل الداخلية light/deep/REM بوصفها
تفصيلًا تنفيذيًا.

للاطلاع على السلوك المفاهيمي وأوامر الشرطة المائلة، راجع [Dreaming](/concepts/dreaming).

### إعدادات المستخدم

| المفتاح         | النوع      | الافتراضي     | الوصف                                       |
| ----------- | --------- | ----------- | ------------------------------------------------- |
| `enabled`   | `boolean` | `false`     | تفعيل Dreaming أو تعطيله بالكامل               |
| `frequency` | `string`  | `0 3 * * *` | وتيرة cron اختيارية لعملية Dreaming الكاملة |

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
- يكتب Dreaming مخرجات سردية مقروءة للبشر إلى `DREAMS.md` (أو `dreams.md` الموجود).
- إن سياسة وعتبات مراحل light/deep/REM هي سلوك داخلي وليست إعدادات موجهة للمستخدم.
