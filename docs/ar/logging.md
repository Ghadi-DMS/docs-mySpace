---
read_when:
    - تحتاج إلى نظرة عامة سهلة للمبتدئين حول التسجيل
    - تريد تهيئة مستويات أو تنسيقات السجل
    - أنت تقوم باستكشاف الأخطاء وتحتاج إلى العثور على السجلات بسرعة
summary: 'نظرة عامة على التسجيل: سجلات الملفات، ومخرجات console، وتتبع CLI، وControl UI'
title: نظرة عامة على التسجيل
x-i18n:
    generated_at: "2026-04-05T12:49:19Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3a5e3800b7c5128602d05d5a35df4f88c373cfbe9397cca7e7154fff56a7f7ef
    source_path: logging.md
    workflow: 15
---

# التسجيل

يمتلك OpenClaw سطحين رئيسيين للسجلات:

- **سجلات الملفات** ‏(أسطر JSON) التي يكتبها Gateway.
- **مخرجات Console** المعروضة في الطرفيات وواجهة Debug UI الخاصة بـ Gateway.

تقوم علامة التبويب **Logs** في Control UI بتتبع سجل ملف gateway. وتشرح هذه الصفحة
مكان وجود السجلات، وكيفية قراءتها، وكيفية تهيئة مستوياتها وتنسيقاتها.

## مكان وجود السجلات

افتراضيًا، يكتب Gateway ملف سجل متجدد ضمن:

`/tmp/openclaw/openclaw-YYYY-MM-DD.log`

ويستخدم التاريخ المنطقة الزمنية المحلية لمضيف gateway.

يمكنك تجاوز هذا في `~/.openclaw/openclaw.json`:

```json
{
  "logging": {
    "file": "/path/to/openclaw.log"
  }
}
```

## كيفية قراءة السجلات

### CLI: تتبع مباشر (موصى به)

استخدم CLI لتتبع ملف سجل gateway عبر RPC:

```bash
openclaw logs --follow
```

خيارات حالية مفيدة:

- `--local-time`: عرض الطوابع الزمنية بمنطقتك الزمنية المحلية
- `--url <url>` / `--token <token>` / `--timeout <ms>`: أعلام Gateway RPC القياسية
- `--expect-final`: علم انتظار الاستجابة النهائية لـ RPC المدعومة بالوكيل (مقبول هنا عبر طبقة العميل المشتركة)

أوضاع الخرج:

- **جلسات TTY**: أسطر سجل منظمة وجميلة وملونة.
- **جلسات غير TTY**: نص عادي.
- `--json`: JSON مفصول بأسطر (حدث سجل واحد في كل سطر).
- `--plain`: فرض النص العادي في جلسات TTY.
- `--no-color`: تعطيل ألوان ANSI.

عندما تمرر `--url` صريحًا، لا يقوم CLI تلقائيًا بتطبيق بيانات الاعتماد من الإعدادات أو
البيئة؛ أضف `--token` بنفسك إذا كان Gateway المستهدف
يتطلب المصادقة.

في وضع JSON، يصدر CLI كائنات موسومة بـ `type`:

- `meta`: بيانات وصفية للبث (الملف، والمؤشر، والحجم)
- `log`: إدخال سجل محلل
- `notice`: تلميحات حول الاقتطاع / التدوير
- `raw`: سطر سجل غير محلل

إذا طلب Gateway المحلي عبر loopback الاقتران، فإن `openclaw logs` يعود تلقائيًا إلى
ملف السجل المحلي المهيأ. أما الأهداف الصريحة عبر `--url` فلا
تستخدم هذا التراجع.

إذا كان Gateway غير قابل للوصول، يطبع CLI تلميحًا قصيرًا لتشغيل:

```bash
openclaw doctor
```

### Control UI ‏(الويب)

تقوم علامة التبويب **Logs** في Control UI بتتبع الملف نفسه باستخدام `logs.tail`.
راجع [/web/control-ui](/web/control-ui) لمعرفة كيفية فتحها.

### سجلات القنوات فقط

لتصفية نشاط القنوات (WhatsApp/Telegram/إلخ)، استخدم:

```bash
openclaw channels logs --channel whatsapp
```

## تنسيقات السجل

### سجلات الملفات (JSONL)

كل سطر في ملف السجل هو كائن JSON. ويقوم CLI وControl UI بتحليل هذه
الإدخالات لعرض خرج منظم (الوقت، والمستوى، والنظام الفرعي، والرسالة).

### مخرجات Console

تكون سجلات Console **مدركة لـ TTY** ومهيأة لسهولة القراءة:

- بادئات الأنظمة الفرعية (مثل `gateway/channels/whatsapp`)
- تلوين المستويات (info/warn/error)
- وضع compact أو JSON اختياري

يتم التحكم في تنسيق Console عبر `logging.consoleStyle`.

### سجلات Gateway WebSocket

يحتوي `openclaw gateway` أيضًا على تسجيل بروتوكول WebSocket لحركة RPC:

- الوضع العادي: النتائج المهمة فقط (الأخطاء، وأخطاء التحليل، والاستدعاءات البطيئة)
- `--verbose`: كل حركة الطلب/الاستجابة
- `--ws-log auto|compact|full`: اختيار نمط العرض المفصل
- `--compact`: اسم بديل لـ `--ws-log compact`

أمثلة:

```bash
openclaw gateway
openclaw gateway --verbose --ws-log compact
openclaw gateway --verbose --ws-log full
```

## تهيئة التسجيل

توجد جميع إعدادات التسجيل تحت `logging` في `~/.openclaw/openclaw.json`.

```json
{
  "logging": {
    "level": "info",
    "file": "/tmp/openclaw/openclaw-YYYY-MM-DD.log",
    "consoleLevel": "info",
    "consoleStyle": "pretty",
    "redactSensitive": "tools",
    "redactPatterns": ["sk-.*"]
  }
}
```

### مستويات السجل

- `logging.level`: مستوى **سجلات الملفات** ‏(JSONL).
- `logging.consoleLevel`: مستوى تفاصيل **Console**.

يمكنك تجاوز كليهما عبر متغير البيئة **`OPENCLAW_LOG_LEVEL`** ‏(مثل `OPENCLAW_LOG_LEVEL=debug`). ولهذا المتغير أولوية على ملف الإعدادات، بحيث يمكنك رفع مستوى التفاصيل لتشغيل واحد دون تعديل `openclaw.json`. ويمكنك أيضًا تمرير الخيار العام في CLI **`--log-level <level>`** ‏(على سبيل المثال `openclaw --log-level debug gateway run`) والذي يتجاوز متغير البيئة لذلك الأمر.

يؤثر `--verbose` فقط في مخرجات console وفي مستوى تفاصيل سجلات WS؛ ولا يغيّر
مستويات سجل الملفات.

### أنماط Console

`logging.consoleStyle`:

- `pretty`: مناسب للبشر، وملون، مع طوابع زمنية.
- `compact`: خرج أكثر إحكامًا (أفضل للجلسات الطويلة).
- `json`: JSON في كل سطر (لمعالجات السجل).

### التنقيح

يمكن لملخصات الأدوات تنقيح الرموز الحساسة قبل أن تصل إلى console:

- `logging.redactSensitive`: ‏`off` | `tools` ‏(الافتراضي: `tools`)
- `logging.redactPatterns`: قائمة سلاسل regex لتجاوز المجموعة الافتراضية

يؤثر التنقيح على **مخرجات console فقط** ولا يغيّر سجلات الملفات.

## التشخيصات + OpenTelemetry

التشخيصات هي أحداث منظمة قابلة للقراءة آليًا لتشغيلات النماذج **و**
telemetry تدفق الرسائل (webhooks، والطوابير، وحالة الجلسات). وهي **لا**
تحل محل السجلات؛ بل وُجدت لتغذية المقاييس، وtraces، والمصدّرات الأخرى.

تُصدر أحداث التشخيص داخل العملية، لكن لا يتم إرفاق المصدّرات إلا عند
تمكين التشخيصات + plugin الخاصة بالمصدّر.

### OpenTelemetry مقابل OTLP

- **OpenTelemetry (OTel)**: نموذج البيانات + SDKs الخاصة بـ traces والمقاييس والسجلات.
- **OTLP**: بروتوكول النقل المستخدم لتصدير بيانات OTel إلى جامع/خلفية.
- يقوم OpenClaw بالتصدير عبر **OTLP/HTTP (protobuf)** حاليًا.

### الإشارات المصدّرة

- **المقاييس**: عدادات + مدرجات تكرارية (استخدام الرموز، وتدفق الرسائل، والطوابير).
- **Traces**: spans لاستخدام النماذج + معالجة webhooks/الرسائل.
- **السجلات**: يتم تصديرها عبر OTLP عند تمكين `diagnostics.otel.logs`. وقد
  يكون حجم السجل كبيرًا؛ لذا ضع `logging.level` ومرشحات المصدّر في الحسبان.

### فهرس أحداث التشخيص

استخدام النموذج:

- `model.usage`: الرموز، والتكلفة، والمدة، والسياق، والمزوّد/النموذج/القناة، ومعرّفات الجلسات.

تدفق الرسائل:

- `webhook.received`: دخول webhook لكل قناة.
- `webhook.processed`: معالجة webhook + المدة.
- `webhook.error`: أخطاء معالج webhook.
- `message.queued`: إدراج رسالة في الطابور للمعالجة.
- `message.processed`: النتيجة + المدة + خطأ اختياري.

الطابور + الجلسة:

- `queue.lane.enqueue`: إدراج مسار طابور الأوامر + العمق.
- `queue.lane.dequeue`: سحب مسار طابور الأوامر + وقت الانتظار.
- `session.state`: انتقال حالة الجلسة + السبب.
- `session.stuck`: تحذير تعثر الجلسة + العمر.
- `run.attempt`: بيانات وصفية لمحاولة/إعادة محاولة التشغيل.
- `diagnostic.heartbeat`: عدادات مجمعة (webhooks/الطابور/الجلسة).

### تمكين التشخيصات (بدون مصدّر)

استخدم هذا إذا كنت تريد أن تكون أحداث التشخيص متاحة لـ plugins أو المصارف المخصصة:

```json
{
  "diagnostics": {
    "enabled": true
  }
}
```

### أعلام التشخيص (سجلات مستهدفة)

استخدم الأعلام لتشغيل سجلات تصحيح إضافية ومحددة من دون رفع `logging.level`.
وتكون الأعلام غير حساسة لحالة الأحرف وتدعم البدائل الشاملة (مثل `telegram.*` أو `*`).

```json
{
  "diagnostics": {
    "flags": ["telegram.http"]
  }
}
```

تجاوز بيئي (لمرة واحدة):

```
OPENCLAW_DIAGNOSTICS=telegram.http,telegram.payload
```

ملاحظات:

- تذهب سجلات الأعلام إلى ملف السجل القياسي (نفس `logging.file`).
- يبقى الخرج منقحًا وفقًا لـ `logging.redactSensitive`.
- الدليل الكامل: [/diagnostics/flags](/diagnostics/flags).

### التصدير إلى OpenTelemetry

يمكن تصدير التشخيصات عبر plugin ‏`diagnostics-otel` ‏(OTLP/HTTP). ويعمل هذا
مع أي جامع/خلفية OpenTelemetry تقبل OTLP/HTTP.

```json
{
  "plugins": {
    "allow": ["diagnostics-otel"],
    "entries": {
      "diagnostics-otel": {
        "enabled": true
      }
    }
  },
  "diagnostics": {
    "enabled": true,
    "otel": {
      "enabled": true,
      "endpoint": "http://otel-collector:4318",
      "protocol": "http/protobuf",
      "serviceName": "openclaw-gateway",
      "traces": true,
      "metrics": true,
      "logs": true,
      "sampleRate": 0.2,
      "flushIntervalMs": 60000
    }
  }
}
```

ملاحظات:

- يمكنك أيضًا تمكين plugin باستخدام `openclaw plugins enable diagnostics-otel`.
- يدعم `protocol` حاليًا فقط `http/protobuf`. ويتم تجاهل `grpc`.
- تتضمن المقاييس استخدام الرموز، والتكلفة، وحجم السياق، ومدة التشغيل، وعدادات/مدرجات تدفق الرسائل (webhooks، والطوابير، وحالة الجلسة، وعمق الطابور/الانتظار).
- يمكن تبديل traces/metrics عبر `traces` / `metrics` ‏(الافتراضي: مفعّلة). وتتضمن traces
  spans استخدام النموذج بالإضافة إلى spans معالجة webhooks/الرسائل عند التمكين.
- اضبط `headers` عندما يتطلب جامعك المصادقة.
- متغيرات البيئة المدعومة: `OTEL_EXPORTER_OTLP_ENDPOINT`،
  و`OTEL_SERVICE_NAME`، و`OTEL_EXPORTER_OTLP_PROTOCOL`.

### المقاييس المصدّرة (الأسماء + الأنواع)

استخدام النموذج:

- `openclaw.tokens` ‏(عداد، سمات: `openclaw.token` و`openclaw.channel`،
  و`openclaw.provider` و`openclaw.model`)
- `openclaw.cost.usd` ‏(عداد، سمات: `openclaw.channel` و`openclaw.provider`،
  و`openclaw.model`)
- `openclaw.run.duration_ms` ‏(مدرج تكراري، سمات: `openclaw.channel`،
  و`openclaw.provider` و`openclaw.model`)
- `openclaw.context.tokens` ‏(مدرج تكراري، سمات: `openclaw.context`،
  و`openclaw.channel` و`openclaw.provider` و`openclaw.model`)

تدفق الرسائل:

- `openclaw.webhook.received` ‏(عداد، سمات: `openclaw.channel`،
  و`openclaw.webhook`)
- `openclaw.webhook.error` ‏(عداد، سمات: `openclaw.channel`،
  و`openclaw.webhook`)
- `openclaw.webhook.duration_ms` ‏(مدرج تكراري، سمات: `openclaw.channel`،
  و`openclaw.webhook`)
- `openclaw.message.queued` ‏(عداد، سمات: `openclaw.channel`،
  و`openclaw.source`)
- `openclaw.message.processed` ‏(عداد، سمات: `openclaw.channel`،
  و`openclaw.outcome`)
- `openclaw.message.duration_ms` ‏(مدرج تكراري، سمات: `openclaw.channel`،
  و`openclaw.outcome`)

الطوابير + الجلسات:

- `openclaw.queue.lane.enqueue` ‏(عداد، سمات: `openclaw.lane`)
- `openclaw.queue.lane.dequeue` ‏(عداد، سمات: `openclaw.lane`)
- `openclaw.queue.depth` ‏(مدرج تكراري، سمات: `openclaw.lane` أو
  `openclaw.channel=heartbeat`)
- `openclaw.queue.wait_ms` ‏(مدرج تكراري، سمات: `openclaw.lane`)
- `openclaw.session.state` ‏(عداد، سمات: `openclaw.state` و`openclaw.reason`)
- `openclaw.session.stuck` ‏(عداد، سمات: `openclaw.state`)
- `openclaw.session.stuck_age_ms` ‏(مدرج تكراري، سمات: `openclaw.state`)
- `openclaw.run.attempt` ‏(عداد، سمات: `openclaw.attempt`)

### spans المصدّرة (الأسماء + السمات الأساسية)

- `openclaw.model.usage`
  - `openclaw.channel` و`openclaw.provider` و`openclaw.model`
  - `openclaw.sessionKey` و`openclaw.sessionId`
  - `openclaw.tokens.*` ‏(input/output/cache_read/cache_write/total)
- `openclaw.webhook.processed`
  - `openclaw.channel` و`openclaw.webhook` و`openclaw.chatId`
- `openclaw.webhook.error`
  - `openclaw.channel` و`openclaw.webhook` و`openclaw.chatId`،
    و`openclaw.error`
- `openclaw.message.processed`
  - `openclaw.channel` و`openclaw.outcome` و`openclaw.chatId`،
    و`openclaw.messageId` و`openclaw.sessionKey` و`openclaw.sessionId`،
    و`openclaw.reason`
- `openclaw.session.stuck`
  - `openclaw.state` و`openclaw.ageMs` و`openclaw.queueDepth`،
    و`openclaw.sessionKey` و`openclaw.sessionId`

### أخذ العينات والتفريغ

- أخذ عينات traces: ‏`diagnostics.otel.sampleRate` ‏(0.0–1.0، spans الجذر فقط).
- فاصل تصدير المقاييس: ‏`diagnostics.otel.flushIntervalMs` ‏(حد أدنى 1000ms).

### ملاحظات البروتوكول

- يمكن ضبط نقاط نهاية OTLP/HTTP عبر `diagnostics.otel.endpoint` أو
  `OTEL_EXPORTER_OTLP_ENDPOINT`.
- إذا كانت نقطة النهاية تحتوي بالفعل على `/v1/traces` أو `/v1/metrics`، فسيتم استخدامها كما هي.
- إذا كانت نقطة النهاية تحتوي بالفعل على `/v1/logs`، فسيتم استخدامها كما هي للسجلات.
- يؤدي `diagnostics.otel.logs` إلى تمكين تصدير سجلات OTLP لمخرجات المُسجّل الرئيسي.

### سلوك تصدير السجلات

- تستخدم سجلات OTLP السجلات المنظمة نفسها المكتوبة إلى `logging.file`.
- تحترم `logging.level` ‏(مستوى سجل الملف). ولا ينطبق تنقيح console
  على سجلات OTLP.
- يجب على عمليات التثبيت ذات الحجم الكبير تفضيل أخذ العينات/التصفية في جامع OTLP.

## نصائح لاستكشاف الأخطاء وإصلاحها

- **تعذر الوصول إلى Gateway؟** شغّل `openclaw doctor` أولًا.
- **السجلات فارغة؟** تحقّق من أن Gateway يعمل ويكتب إلى مسار الملف
  في `logging.file`.
- **تحتاج إلى مزيد من التفاصيل؟** اضبط `logging.level` على `debug` أو `trace` ثم أعد المحاولة.

## ذو صلة

- [الآليات الداخلية لتسجيل Gateway](/gateway/logging) — أنماط سجل WS، وبادئات الأنظمة الفرعية، والتقاط console
- [التشخيصات](/gateway/configuration-reference#diagnostics) — تصدير OpenTelemetry وإعدادات تتبع cache
