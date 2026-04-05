---
read_when:
    - تريد تهيئة موفري بحث الذاكرة أو نماذج التضمين
    - تريد إعداد الواجهة الخلفية لـ QMD
    - تريد ضبط البحث الهجين أو MMR أو الاضمحلال الزمني
    - تريد تمكين فهرسة الذاكرة متعددة الوسائط
summary: جميع خيارات التهيئة لبحث الذاكرة، وموفري التضمين، وQMD، والبحث الهجين، والفهرسة متعددة الوسائط
title: مرجع تهيئة الذاكرة
x-i18n:
    generated_at: "2026-04-05T12:55:45Z"
    model: gpt-5.4
    provider: openai
    source_hash: 89e4c9740f71f5a47fc5e163742339362d6b95cb4757650c0c8a095cf3078caa
    source_path: reference/memory-config.md
    workflow: 15
---

# مرجع تهيئة الذاكرة

تسرد هذه الصفحة كل خيارات التهيئة لبحث ذاكرة OpenClaw. للحصول على
نظرات عامة مفاهيمية، راجع:

- [نظرة عامة على الذاكرة](/ar/concepts/memory) -- كيفية عمل الذاكرة
- [المحرّك المدمج](/ar/concepts/memory-builtin) -- الواجهة الخلفية الافتراضية لـ SQLite
- [محرّك QMD](/ar/concepts/memory-qmd) -- خدمة جانبية محلية أولًا
- [بحث الذاكرة](/ar/concepts/memory-search) -- مسار البحث والضبط

توجد جميع إعدادات بحث الذاكرة تحت `agents.defaults.memorySearch` في
`openclaw.json` ما لم يُذكر خلاف ذلك.

---

## اختيار الموفر

| المفتاح   | النوع      | الافتراضي       | الوصف                                                                            |
| ---------- | --------- | ---------------- | -------------------------------------------------------------------------------- |
| `provider` | `string`  | يُكتشف تلقائيًا | معرّف مهايئ التضمين: `openai`، `gemini`، `voyage`، `mistral`، `ollama`، `local` |
| `model`    | `string`  | الافتراضي للموفر | اسم نموذج التضمين                                                                |
| `fallback` | `string`  | `"none"`         | معرّف المهايئ الاحتياطي عند فشل الأساسي                                          |
| `enabled`  | `boolean` | `true`           | تمكين أو تعطيل بحث الذاكرة                                                       |

### ترتيب الاكتشاف التلقائي

عندما لا يتم تعيين `provider`، يختار OpenClaw أول خيار متاح:

1. `local` -- إذا كانت `memorySearch.local.modelPath` مهيأة والملف موجودًا.
2. `openai` -- إذا أمكن حل مفتاح OpenAI.
3. `gemini` -- إذا أمكن حل مفتاح Gemini.
4. `voyage` -- إذا أمكن حل مفتاح Voyage.
5. `mistral` -- إذا أمكن حل مفتاح Mistral.

`ollama` مدعوم لكنه لا يُكتشف تلقائيًا (عيّنه صراحةً).

### حل مفتاح API

تتطلب عمليات التضمين البعيدة مفتاح API. يحل OpenClaw المفتاح من:
ملفات تعريف المصادقة، أو `models.providers.*.apiKey`، أو متغيرات البيئة.

| الموفر | متغير البيئة                  | مفتاح التهيئة                      |
| -------- | ------------------------------ | --------------------------------- |
| OpenAI   | `OPENAI_API_KEY`               | `models.providers.openai.apiKey`  |
| Gemini   | `GEMINI_API_KEY`               | `models.providers.google.apiKey`  |
| Voyage   | `VOYAGE_API_KEY`               | `models.providers.voyage.apiKey`  |
| Mistral  | `MISTRAL_API_KEY`              | `models.providers.mistral.apiKey` |
| Ollama   | `OLLAMA_API_KEY` (عنصر نائب)  | --                                |

يغطي Codex OAuth الدردشة/الإكمالات فقط ولا يفي بطلبات
التضمين.

---

## تهيئة نقطة النهاية البعيدة

لنقاط النهاية المخصصة المتوافقة مع OpenAI أو لتجاوز الإعدادات الافتراضية للموفر:

| المفتاح          | النوع    | الوصف                                              |
| ---------------- | -------- | -------------------------------------------------- |
| `remote.baseUrl` | `string` | عنوان URL أساسي مخصص للـ API                       |
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

## تهيئة خاصة بـ Gemini

| المفتاح               | النوع    | الافتراضي             | الوصف                                      |
| ---------------------- | -------- | ---------------------- | ------------------------------------------ |
| `model`                | `string` | `gemini-embedding-001` | يدعم أيضًا `gemini-embedding-2-preview`    |
| `outputDimensionality` | `number` | `3072`                 | بالنسبة إلى Embedding 2: 768 أو 1536 أو 3072 |

<Warning>
يؤدي تغيير النموذج أو `outputDimensionality` إلى إعادة فهرسة كاملة تلقائية.
</Warning>

---

## تهيئة التضمين المحلي

| المفتاح              | النوع    | الافتراضي              | الوصف                         |
| --------------------- | -------- | ---------------------- | ------------------------------- |
| `local.modelPath`     | `string` | يُنزَّل تلقائيًا       | المسار إلى ملف نموذج GGUF      |
| `local.modelCacheDir` | `string` | افتراضي node-llama-cpp | دليل التخزين المؤقت للنماذج المُنزَّلة |

النموذج الافتراضي: `embeddinggemma-300m-qat-Q8_0.gguf` (~0.6 غيغابايت، يُنزَّل تلقائيًا).
يتطلب بناءً أصليًا: `pnpm approve-builds` ثم `pnpm rebuild node-llama-cpp`.

---

## تهيئة البحث الهجين

كلها تحت `memorySearch.query.hybrid`:

| المفتاح              | النوع      | الافتراضي | الوصف                            |
| --------------------- | --------- | ------- | ---------------------------------- |
| `enabled`             | `boolean` | `true`  | تمكين بحث BM25 + المتجهات الهجين |
| `vectorWeight`        | `number`  | `0.7`   | الوزن لدرجات المتجهات (0-1)      |
| `textWeight`          | `number`  | `0.3`   | الوزن لدرجات BM25 (0-1)          |
| `candidateMultiplier` | `number`  | `4`     | مضاعف حجم مجموعة المرشحين        |

### MMR (التنوع)

| المفتاح      | النوع      | الافتراضي | الوصف                              |
| ------------- | --------- | ------- | ------------------------------------ |
| `mmr.enabled` | `boolean` | `false` | تمكين إعادة الترتيب باستخدام MMR    |
| `mmr.lambda`  | `number`  | `0.7`   | 0 = أقصى تنوع، 1 = أقصى صلة         |

### الاضمحلال الزمني (الحداثة)

| المفتاح                     | النوع      | الافتراضي | الوصف                      |
| ---------------------------- | --------- | ------- | ------------------------- |
| `temporalDecay.enabled`      | `boolean` | `false` | تمكين تعزيز الحداثة       |
| `temporalDecay.halfLifeDays` | `number`  | `30`    | تنخفض الدرجة إلى النصف كل N يومًا |

الملفات الدائمة (`MEMORY.md` والملفات غير المؤرخة في `memory/`) لا يطبق عليها الاضمحلال أبدًا.

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

| المفتاح     | النوع      | الوصف                                  |
| ------------ | ---------- | ---------------------------------------- |
| `extraPaths` | `string[]` | أدلة أو ملفات إضافية للفهرسة            |

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

يمكن أن تكون المسارات مطلقة أو نسبة إلى مساحة العمل. تُفحص الأدلة
بشكل递归 بحثًا عن ملفات `.md`. يعتمد التعامل مع الروابط الرمزية على الواجهة الخلفية النشطة:
إذ يتجاهل المحرّك المدمج الروابط الرمزية، بينما يتبع QMD سلوك ماسح
QMD الأساسي.

للبحث في نصوص الجلسات عبر الوكلاء ضمن نطاق وكيل محدد، استخدم
`agents.list[].memorySearch.qmd.extraCollections` بدلًا من `memory.qmd.paths`.
تتبع هذه المجموعات الإضافية البنية نفسها `{ path, name, pattern? }`، لكنها
تُدمج لكل وكيل ويمكنها الاحتفاظ بأسماء مشتركة صريحة عندما يشير المسار
إلى خارج مساحة العمل الحالية.
إذا ظهر المسار المحلول نفسه في كل من `memory.qmd.paths` و
`memorySearch.qmd.extraCollections`، يحتفظ QMD بالإدخال الأول ويتجاوز
الإدخال المكرر.

---

## الذاكرة متعددة الوسائط (Gemini)

افهرس الصور والصوت إلى جانب Markdown باستخدام Gemini Embedding 2:

| المفتاح                  | النوع      | الافتراضي | الوصف                                   |
| ------------------------- | ---------- | ---------- | -------------------------------------- |
| `multimodal.enabled`      | `boolean`  | `false`    | تمكين الفهرسة متعددة الوسائط            |
| `multimodal.modalities`   | `string[]` | --         | `["image"]` أو `["audio"]` أو `["all"]` |
| `multimodal.maxFileBytes` | `number`   | `10000000` | الحد الأقصى لحجم الملف للفهرسة          |

ينطبق هذا فقط على الملفات ضمن `extraPaths`. تظل جذور الذاكرة الافتراضية خاصة بـ Markdown فقط.
يتطلب `gemini-embedding-2-preview`. يجب أن تكون قيمة `fallback` هي `"none"`.

التنسيقات المدعومة: `.jpg` و`.jpeg` و`.png` و`.webp` و`.gif` و`.heic` و`.heif`
(صور)؛ `.mp3` و`.wav` و`.ogg` و`.opus` و`.m4a` و`.aac` و`.flac` (صوت).

---

## التخزين المؤقت للتضمين

| المفتاح           | النوع      | الافتراضي | الوصف                               |
| ------------------ | --------- | ------- | -------------------------------- |
| `cache.enabled`    | `boolean` | `false` | تخزين تضمينات المقاطع في SQLite   |
| `cache.maxEntries` | `number`  | `50000` | الحد الأقصى للتضمينات المخزنة      |

يمنع إعادة تضمين النص غير المتغير أثناء إعادة الفهرسة أو تحديثات النصوص.

---

## الفهرسة على دفعات

| المفتاح                      | النوع      | الافتراضي | الوصف                     |
| ----------------------------- | --------- | ------- | -------------------------- |
| `remote.batch.enabled`        | `boolean` | `false` | تمكين API التضمين الدفعي   |
| `remote.batch.concurrency`    | `number`  | `2`     | مهام الدفعات المتوازية     |
| `remote.batch.wait`           | `boolean` | `true`  | انتظار اكتمال الدفعة       |
| `remote.batch.pollIntervalMs` | `number`  | --      | فترة الاستطلاع             |
| `remote.batch.timeoutMinutes` | `number`  | --      | مهلة الدفعة                |

متاح لـ `openai` و`gemini` و`voyage`. عادةً ما تكون دفعات OpenAI
الأسرع والأرخص لعمليات التعبئة الخلفية الكبيرة.

---

## بحث ذاكرة الجلسة (تجريبي)

افهرس نصوص الجلسات وأظهرها عبر `memory_search`:

| المفتاح                      | النوع      | الافتراضي   | الوصف                                   |
| ----------------------------- | ---------- | ------------ | --------------------------------------- |
| `experimental.sessionMemory`  | `boolean`  | `false`      | تمكين فهرسة الجلسات                     |
| `sources`                     | `string[]` | `["memory"]` | أضف `"sessions"` لتضمين النصوص          |
| `sync.sessions.deltaBytes`    | `number`   | `100000`     | عتبة البايت لإعادة الفهرسة              |
| `sync.sessions.deltaMessages` | `number`   | `50`         | عتبة الرسائل لإعادة الفهرسة             |

فهرسة الجلسات اختيارية وتعمل بشكل غير متزامن. قد تكون النتائج
قديمة قليلًا. تُخزن سجلات الجلسات على القرص، لذا اعتبر الوصول إلى نظام الملفات
هو حد الثقة.

---

## تسريع المتجهات في SQLite (sqlite-vec)

| المفتاح                     | النوع      | الافتراضي | الوصف                              |
| ---------------------------- | --------- | ------- | --------------------------------- |
| `store.vector.enabled`       | `boolean` | `true`  | استخدام sqlite-vec لاستعلامات المتجهات |
| `store.vector.extensionPath` | `string`  | مضمّن   | تجاوز مسار sqlite-vec             |

عندما لا يكون sqlite-vec متاحًا، يعود OpenClaw تلقائيًا إلى
تشابه جيب التمام داخل العملية.

---

## تخزين الفهرس

| المفتاح              | النوع    | الافتراضي                            | الوصف                                      |
| --------------------- | -------- | ------------------------------------- | ------------------------------------------- |
| `store.path`          | `string` | `~/.openclaw/memory/{agentId}.sqlite` | موقع الفهرس (يدعم الرمز `{agentId}`)        |
| `store.fts.tokenizer` | `string` | `unicode61`                           | محلل FTS5 (`unicode61` أو `trigram`)        |

---

## تهيئة الواجهة الخلفية لـ QMD

عيّن `memory.backend = "qmd"` للتمكين. توجد جميع إعدادات QMD تحت
`memory.qmd`:

| المفتاح                 | النوع      | الافتراضي | الوصف                                         |
| ------------------------ | --------- | -------- | -------------------------------------------- |
| `command`                | `string`  | `qmd`    | مسار الملف التنفيذي لـ QMD                    |
| `searchMode`             | `string`  | `search` | أمر البحث: `search` أو `vsearch` أو `query`  |
| `includeDefaultMemory`   | `boolean` | `true`   | فهرسة `MEMORY.md` و`memory/**/*.md` تلقائيًا |
| `paths[]`                | `array`   | --       | مسارات إضافية: `{ name, path, pattern? }`    |
| `sessions.enabled`       | `boolean` | `false`  | فهرسة نصوص الجلسات                            |
| `sessions.retentionDays` | `number`  | --       | مدة الاحتفاظ بالنصوص                          |
| `sessions.exportDir`     | `string`  | --       | دليل التصدير                                  |

### جدول التحديث

| المفتاح                  | النوع      | الافتراضي | الوصف                                 |
| ------------------------- | --------- | ------- | ------------------------------------- |
| `update.interval`         | `string`  | `5m`    | فترة التحديث                          |
| `update.debounceMs`       | `number`  | `15000` | إزالة ارتداد تغييرات الملفات         |
| `update.onBoot`           | `boolean` | `true`  | التحديث عند بدء التشغيل               |
| `update.waitForBootSync`  | `boolean` | `false` | حظر بدء التشغيل حتى اكتمال التحديث    |
| `update.embedInterval`    | `string`  | --      | وتيرة تضمين منفصلة                    |
| `update.commandTimeoutMs` | `number`  | --      | مهلة أوامر QMD                        |
| `update.updateTimeoutMs`  | `number`  | --      | مهلة عمليات تحديث QMD                 |
| `update.embedTimeoutMs`   | `number`  | --      | مهلة عمليات تضمين QMD                 |

### الحدود

| المفتاح                  | النوع    | الافتراضي | الوصف                          |
| ------------------------- | -------- | ------- | -------------------------- |
| `limits.maxResults`       | `number` | `6`     | الحد الأقصى لنتائج البحث    |
| `limits.maxSnippetChars`  | `number` | --      | تقييد طول المقتطف           |
| `limits.maxInjectedChars` | `number` | --      | تقييد إجمالي الأحرف المُدرجة |
| `limits.timeoutMs`        | `number` | `4000`  | مهلة البحث                  |

### النطاق

يتحكم في الجلسات التي يمكنها تلقي نتائج بحث QMD. نفس المخطط كما في
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

الإعداد الافتراضي هو الرسائل المباشرة فقط. يطابق `match.keyPrefix` مفتاح الجلسة الموحّد؛
ويطابق `match.rawKeyPrefix` المفتاح الخام بما في ذلك `agent:<id>:`.

### الاستشهادات

ينطبق `memory.citations` على جميع الواجهات الخلفية:

| القيمة            | السلوك                                               |
| ---------------- | --------------------------------------------------- |
| `auto` (افتراضي) | تضمين تذييل `Source: <path#line>` في المقتطفات      |
| `on`             | تضمين التذييل دائمًا                                |
| `off`            | حذف التذييل (يظل المسار يُمرر إلى الوكيل داخليًا)   |

### مثال كامل لـ QMD

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

تتم تهيئة Dreaming تحت `plugins.entries.memory-core.config.dreaming`,
وليس تحت `agents.defaults.memorySearch`. للحصول على التفاصيل المفاهيمية
وأوامر الدردشة، راجع [Dreaming](/ar/concepts/memory-dreaming).

| المفتاح           | النوع    | الافتراضي       | الوصف                                      |
| ------------------ | -------- | -------------- | ----------------------------------------- |
| `mode`             | `string` | `"off"`        | إعداد مسبق: `off` أو `core` أو `rem` أو `deep` |
| `cron`             | `string` | الإعداد المسبق الافتراضي | تجاوز تعبير Cron للجدول الزمني      |
| `timezone`         | `string` | المنطقة الزمنية للمستخدم | المنطقة الزمنية لتقييم الجدول الزمني |
| `limit`            | `number` | الإعداد المسبق الافتراضي | الحد الأقصى للمرشحين للترقية في كل دورة |
| `minScore`         | `number` | الإعداد المسبق الافتراضي | الحد الأدنى للدرجة الموزونة للترقية   |
| `minRecallCount`   | `number` | الإعداد المسبق الافتراضي | الحد الأدنى لعتبة عدد الاستدعاءات     |
| `minUniqueQueries` | `number` | الإعداد المسبق الافتراضي | الحد الأدنى لعتبة عدد الاستعلامات المميزة |

### الإعدادات الافتراضية المسبقة

| الوضع   | الوتيرة          | minScore | minRecallCount | minUniqueQueries |
| ------ | -------------- | -------- | -------------- | ---------------- |
| `off`  | معطل           | --       | --             | --               |
| `core` | يوميًا 3 صباحًا | 0.75     | 3              | 2                |
| `rem`  | كل 6 ساعات      | 0.85     | 4              | 3                |
| `deep` | كل 12 ساعة      | 0.80     | 3              | 3                |

### مثال

```json5
{
  plugins: {
    entries: {
      "memory-core": {
        config: {
          dreaming: {
            mode: "core",
            timezone: "America/New_York",
          },
        },
      },
    },
  },
}
```
