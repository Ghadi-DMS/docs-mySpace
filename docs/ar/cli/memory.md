---
read_when:
    - تريد فهرسة الذاكرة الدلالية أو البحث فيها
    - أنت تصحح أخطاء توفر الذاكرة أو الفهرسة
    - تريد ترقية الذاكرة قصيرة المدى المسترجعة إلى `MEMORY.md`
summary: مرجع CLI للأمر `openclaw memory` ‏(status/index/search/promote)
title: memory
x-i18n:
    generated_at: "2026-04-05T12:38:43Z"
    model: gpt-5.4
    provider: openai
    source_hash: a89e3a819737bb63521128ae63d9e25b5cd9db35c3ea4606d087a8ad48b41eab
    source_path: cli/memory.md
    workflow: 15
---

# `openclaw memory`

إدارة فهرسة الذاكرة الدلالية والبحث فيها.
يوفرها plugin الذاكرة النشط (الافتراضي: `memory-core`؛ اضبط `plugins.slots.memory = "none"` للتعطيل).

ذو صلة:

- مفهوم الذاكرة: [الذاكرة](/concepts/memory)
- Plugins: [Plugins](/tools/plugin)

## أمثلة

```bash
openclaw memory status
openclaw memory status --deep
openclaw memory status --fix
openclaw memory index --force
openclaw memory search "meeting notes"
openclaw memory search --query "deployment" --max-results 20
openclaw memory promote --limit 10 --min-score 0.75
openclaw memory promote --apply
openclaw memory promote --json --min-recall-count 0 --min-unique-queries 0
openclaw memory status --json
openclaw memory status --deep --index
openclaw memory status --deep --index --verbose
openclaw memory status --agent main
openclaw memory index --agent main --verbose
```

## الخيارات

`memory status` و`memory index`:

- `--agent <id>`: قصر النطاق على وكيل واحد. من دونه، تعمل هذه الأوامر لكل وكيل مكوَّن؛ وإذا لم تكن هناك قائمة وكلاء مكوّنة، فإنها تعود إلى الوكيل الافتراضي.
- `--verbose`: إصدار سجلات تفصيلية أثناء الفحوصات والفهرسة.

`memory status`:

- `--deep`: فحص توفر vector + embedding.
- `--index`: تشغيل إعادة فهرسة إذا كان المخزن متسخًا (ويتضمن `--deep`).
- `--fix`: إصلاح أقفال الاسترجاع القديمة وتطبيع بيانات الترقية الوصفية.
- `--json`: طباعة خرج JSON.

`memory index`:

- `--force`: فرض إعادة فهرسة كاملة.

`memory search`:

- إدخال الاستعلام: مرّر إما `[query]` الموضعي أو `--query <text>`.
- إذا تم توفير الاثنين، فإن `--query` هو الفائز.
- إذا لم يتم توفير أي منهما، يخرج الأمر بخطأ.
- `--agent <id>`: قصر النطاق على وكيل واحد (الافتراضي: الوكيل الافتراضي).
- `--max-results <n>`: تحديد عدد النتائج المعادة.
- `--min-score <n>`: تصفية المطابقات منخفضة الدرجات.
- `--json`: طباعة نتائج JSON.

`memory promote`:

معاينة وتطبيق ترقيات الذاكرة قصيرة المدى.

```bash
openclaw memory promote [--apply] [--limit <n>] [--include-promoted]
```

- `--apply` -- كتابة الترقيات إلى `MEMORY.md` (الافتراضي: المعاينة فقط).
- `--limit <n>` -- وضع حد أقصى لعدد المرشحين المعروضين.
- `--include-promoted` -- تضمين الإدخالات التي تمت ترقيتها بالفعل في الدورات السابقة.

الخيارات الكاملة:

- يرتّب المرشحين قصيري المدى من `memory/YYYY-MM-DD.md` باستخدام إشارات استرجاع موزونة (`frequency` و`relevance` و`query diversity` و`recency`).
- يستخدم أحداث الاسترجاع الملتقطة عندما يعيد `memory_search` نتائج من ذاكرة يومية.
- وضع dreaming التلقائي الاختياري: عندما تكون `plugins.entries.memory-core.config.dreaming.mode` هي `core` أو `deep` أو `rem`، يدير `memory-core` تلقائيًا مهمة cron تؤدي إلى الترقية في الخلفية (ولا حاجة إلى `openclaw cron add` يدوي).
- `--agent <id>`: قصر النطاق على وكيل واحد (الافتراضي: الوكيل الافتراضي).
- `--limit <n>`: الحد الأقصى للمرشحين الذين سيتم إرجاعهم/تطبيقهم.
- `--min-score <n>`: الحد الأدنى للدرجة الموزونة للترقية.
- `--min-recall-count <n>`: الحد الأدنى المطلوب لعدد مرات الاسترجاع لمرشح ما.
- `--min-unique-queries <n>`: الحد الأدنى المطلوب لعدد الاستعلامات المميزة لمرشح ما.
- `--apply`: إلحاق المرشحين المحددين إلى `MEMORY.md` ووضع علامة عليهم كمُرقّين.
- `--include-promoted`: تضمين المرشحين الذين تمت ترقيتهم بالفعل في الخرج.
- `--json`: طباعة خرج JSON.

## Dreaming (تجريبي)

Dreaming هو تمرير التأمل الليلي للذاكرة. يسمى "dreaming" لأن النظام يعيد زيارة ما تم استرجاعه خلال اليوم ويقرر ما الذي يستحق الاحتفاظ به على المدى الطويل.

- هو اختياري ومعطّل افتراضيًا.
- فعّله باستخدام `plugins.entries.memory-core.config.dreaming.mode`.
- يمكنك تبديل الأوضاع من الدردشة باستخدام `/dreaming off|core|rem|deep`. شغّل `/dreaming` (أو `/dreaming options`) لمعرفة ما يفعله كل وضع.
- عند تمكينه، ينشئ `memory-core` تلقائيًا مهمة cron مُدارة ويحافظ عليها.
- اضبط `dreaming.limit` على `0` إذا كنت تريد تمكين dreaming ولكن مع إيقاف الترقية التلقائية فعليًا.
- يستخدم الترتيب إشارات موزونة: تكرار الاسترجاع، وملاءمة الاسترجاع، وتنوع الاستعلامات، والحداثة الزمنية (تتلاشى عمليات الاسترجاع الحديثة بمرور الوقت).
- لا تحدث الترقية إلى `MEMORY.md` إلا عند استيفاء حدود الجودة، حتى تظل الذاكرة طويلة المدى عالية الإشارة بدلًا من جمع تفاصيل لمرة واحدة.

الإعدادات المسبقة الافتراضية للأوضاع:

- `core`: يوميًا عند `0 3 * * *`، و`minScore=0.75`، و`minRecallCount=3`، و`minUniqueQueries=2`
- `deep`: كل 12 ساعة (`0 */12 * * *`)، و`minScore=0.8`، و`minRecallCount=3`، و`minUniqueQueries=3`
- `rem`: كل 6 ساعات (`0 */6 * * *`)، و`minScore=0.85`، و`minRecallCount=4`، و`minUniqueQueries=3`

مثال:

```json
{
  "plugins": {
    "entries": {
      "memory-core": {
        "config": {
          "dreaming": {
            "mode": "core"
          }
        }
      }
    }
  }
}
```

ملاحظات:

- يطبع `memory index --verbose` تفاصيل لكل مرحلة (المزوّد، والنموذج، والمصادر، ونشاط الدُفعات).
- يتضمن `memory status` أي مسارات إضافية مكوّنة عبر `memorySearch.extraPaths`.
- إذا كانت حقول مفاتيح API البعيدة الخاصة بالذاكرة النشطة فعليًا مكوّنة كـ SecretRefs، فإن الأمر يحل هذه القيم من لقطة gateway النشطة. وإذا لم يكن gateway متاحًا، يفشل الأمر سريعًا.
- ملاحظة حول اختلاف إصدار Gateway: يتطلب مسار هذا الأمر gateway يدعم `secrets.resolve`؛ تعيد البوابات الأقدم خطأ method غير معروف.
- يكون إيقاع dreaming افتراضيًا حسب الجدول المسبق لكل وضع. تجاوز الإيقاع باستخدام `plugins.entries.memory-core.config.dreaming.frequency` كتعبير cron (مثل `0 3 * * *`) واضبطه بدقة باستخدام `timezone` و`limit` و`minScore` و`minRecallCount` و`minUniqueQueries`.
