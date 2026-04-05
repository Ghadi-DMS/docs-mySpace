---
read_when:
    - تشغيل الاختبارات أو إصلاحها
summary: كيفية تشغيل الاختبارات محليًا (vitest) ومتى تستخدم وضعي force وcoverage
title: الاختبارات
x-i18n:
    generated_at: "2026-04-05T12:56:18Z"
    model: gpt-5.4
    provider: openai
    source_hash: 78390107a9ac2bdc4294d4d0204467c5efdd98faebaf308f3a4597ab966a6d26
    source_path: reference/test.md
    workflow: 15
---

# الاختبارات

- مجموعة الاختبارات الكاملة (الأجنحة، الاختبارات الحية، Docker): [الاختبار](/ar/help/testing)

- `pnpm test:force`: يوقف أي عملية Gateway عالقة ما تزال تحتفظ بمنفذ التحكم الافتراضي، ثم يشغّل مجموعة Vitest الكاملة باستخدام منفذ Gateway معزول حتى لا تتعارض اختبارات الخادم مع مثيل قيد التشغيل. استخدم هذا عندما يترك تشغيل سابق لـ Gateway المنفذ 18789 مشغولًا.
- `pnpm test:coverage`: يشغّل مجموعة اختبارات الوحدة مع تغطية V8 (عبر `vitest.unit.config.ts`). الحدود العامة هي 70% للأسطر/الفروع/الدوال/العبارات. تستثني التغطية نقاط الإدخال الثقيلة تكامليًا (توصيلات CLI، وجسور gateway/telegram، وخادم webchat الثابت) للحفاظ على تركيز الهدف على المنطق القابل لاختبار الوحدة.
- `pnpm test:coverage:changed`: يشغّل تغطية اختبارات الوحدة فقط للملفات التي تغيرت منذ `origin/main`.
- `pnpm test:changed`: يشغّل تهيئة مشاريع Vitest الأصلية مع `--changed origin/main`. تتعامل التهيئة الأساسية مع ملفات المشاريع/التهيئة باعتبارها `forceRerunTriggers` حتى تُعاد عمليات التشغيل على نطاق واسع عند الحاجة بعد تغييرات التوصيل.
- `pnpm test`: يشغّل تهيئة مشاريع الجذر الأصلية لـ Vitest مباشرة. تعمل مرشحات الملفات بشكل أصلي عبر المشاريع المهيأة.
- تهيئة Vitest الأساسية تستخدم الآن افتراضيًا `pool: "threads"` و`isolate: false`، مع تمكين المشغّل المشترك غير المعزول عبر تهيئات المستودع.
- `pnpm test:channels` يشغّل `vitest.channels.config.ts`.
- `pnpm test:extensions` يشغّل `vitest.extensions.config.ts`.
- `pnpm test:extensions`: يشغّل مجموعات extension/plugin.
- `pnpm test:perf:imports`: يفعّل تقارير مدة الاستيراد + تفصيلات الاستيراد في Vitest لتشغيل مشاريع الجذر الأصلية.
- `pnpm test:perf:imports:changed`: نفس تحليل الأداء للاستيراد، لكن فقط للملفات التي تغيرت منذ `origin/main`.
- `pnpm test:perf:profile:main`: يكتب ملف تعريف CPU للخيط الرئيسي لـ Vitest (`.artifacts/vitest-main-profile`).
- `pnpm test:perf:profile:runner`: يكتب ملفات تعريف CPU + heap لمشغّل الوحدة (`.artifacts/vitest-runner-profile`).
- تكامل Gateway: اختياري عبر `OPENCLAW_TEST_INCLUDE_GATEWAY=1 pnpm test` أو `pnpm test:gateway`.
- `pnpm test:e2e`: يشغّل اختبارات الدخان الشاملة لـ gateway (إقران WS/HTTP/node متعدد المثيلات). يستخدم افتراضيًا `threads` + `isolate: false` مع عمّال تكيّفيين في `vitest.e2e.config.ts`؛ اضبطه عبر `OPENCLAW_E2E_WORKERS=<n>` وعيّن `OPENCLAW_E2E_VERBOSE=1` للحصول على سجلات تفصيلية.
- `pnpm test:live`: يشغّل اختبارات الموفر الحية (minimax/zai). يتطلب مفاتيح API و`LIVE=1` (أو `*_LIVE_TEST=1` الخاص بالموفر) لإلغاء التخطي.
- `pnpm test:docker:openwebui`: يبدأ OpenClaw وOpen WebUI داخل Docker، ويسجّل الدخول عبر Open WebUI، ويفحص `/api/models`، ثم يشغّل دردشة حقيقية ممررة عبر `/api/chat/completions`. يتطلب مفتاح نموذج حي قابلًا للاستخدام (مثل OpenAI في `~/.profile`)، ويسحب صورة Open WebUI خارجية، ولا يُتوقع أن يكون مستقرًا في CI مثل مجموعات الوحدة/e2e العادية.
- `pnpm test:docker:mcp-channels`: يبدأ حاوية Gateway مُهيأة مسبقًا وحاوية عميل ثانية تشغّل `openclaw mcp serve`، ثم يتحقق من اكتشاف المحادثات الموجهة، وقراءة النصوص، وبيانات المرفقات الوصفية، وسلوك قائمة انتظار الأحداث الحية، وتوجيه الإرسال الصادر، وإشعارات القنوات + الأذونات بأسلوب Claude عبر جسر stdio الحقيقي. تقرأ عملية التحقق من إشعار Claude إطارات MCP الخام عبر stdio مباشرة حتى يعكس اختبار الدخان ما يصدره الجسر فعليًا.

## بوابة PR المحلية

لفحوصات الإرساء/البوابة المحلية الخاصة بـ PR، شغّل:

- `pnpm check`
- `pnpm build`
- `pnpm test`
- `pnpm check:docs`

إذا كان `pnpm test` غير مستقر على جهاز مضغوط، فأعد تشغيله مرة واحدة قبل اعتباره تراجعًا، ثم اعزل المشكلة باستخدام `pnpm test <path/to/test>`. وبالنسبة للأجهزة محدودة الذاكرة، استخدم:

- `OPENCLAW_VITEST_MAX_WORKERS=1 pnpm test`
- `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/tmp/openclaw-vitest-cache pnpm test:changed`

## معيار زمن استجابة النموذج (مفاتيح محلية)

السكريبت: [`scripts/bench-model.ts`](https://github.com/openclaw/openclaw/blob/main/scripts/bench-model.ts)

الاستخدام:

- `source ~/.profile && pnpm tsx scripts/bench-model.ts --runs 10`
- متغيرات البيئة الاختيارية: `MINIMAX_API_KEY`، `MINIMAX_BASE_URL`، `MINIMAX_MODEL`، `ANTHROPIC_API_KEY`
- الموجّه الافتراضي: “Reply with a single word: ok. No punctuation or extra text.”

آخر تشغيل (2025-12-31، 20 تشغيلًا):

- median لـ minimax هو 1279ms (الحد الأدنى 1114، الحد الأقصى 2431)
- median لـ opus هو 2454ms (الحد الأدنى 1224، الحد الأقصى 3170)

## معيار بدء تشغيل CLI

السكريبت: [`scripts/bench-cli-startup.ts`](https://github.com/openclaw/openclaw/blob/main/scripts/bench-cli-startup.ts)

الاستخدام:

- `pnpm test:startup:bench`
- `pnpm test:startup:bench:smoke`
- `pnpm test:startup:bench:save`
- `pnpm test:startup:bench:update`
- `pnpm test:startup:bench:check`
- `pnpm tsx scripts/bench-cli-startup.ts`
- `pnpm tsx scripts/bench-cli-startup.ts --runs 12`
- `pnpm tsx scripts/bench-cli-startup.ts --preset real`
- `pnpm tsx scripts/bench-cli-startup.ts --preset real --case status --case gatewayStatus --runs 3`
- `pnpm tsx scripts/bench-cli-startup.ts --entry openclaw.mjs --entry-secondary dist/entry.js --preset all`
- `pnpm tsx scripts/bench-cli-startup.ts --preset all --output .artifacts/cli-startup-bench-all.json`
- `pnpm tsx scripts/bench-cli-startup.ts --preset real --case gatewayStatusJson --output .artifacts/cli-startup-bench-smoke.json`
- `pnpm tsx scripts/bench-cli-startup.ts --preset real --cpu-prof-dir .artifacts/cli-cpu`
- `pnpm tsx scripts/bench-cli-startup.ts --json`

الإعدادات المسبقة:

- `startup`: `--version`، `--help`، `health`، `health --json`، `status --json`، `status`
- `real`: `health`، `status`، `status --json`، `sessions`، `sessions --json`، `agents list --json`، `gateway status`، `gateway status --json`، `gateway health --json`، `config get gateway.port`
- `all`: كلا الإعدادين المسبقين

يتضمن الإخراج `sampleCount`، والمتوسط، وp50، وp95، والحدين الأدنى/الأقصى، وتوزيع exit-code/signal، وملخصات أقصى RSS لكل أمر. يكتب الخياران `--cpu-prof-dir` / `--heap-prof-dir` ملفات تعريف V8 لكل تشغيل بحيث يستخدم القياس الزمني والتقاط ملفات التعريف الأداة نفسها.

اصطلاحات الإخراج المحفوظ:

- `pnpm test:startup:bench:smoke` يكتب أثر اختبار الدخان المستهدف في `.artifacts/cli-startup-bench-smoke.json`
- `pnpm test:startup:bench:save` يكتب أثر المجموعة الكاملة في `.artifacts/cli-startup-bench-all.json` باستخدام `runs=5` و`warmup=1`
- `pnpm test:startup:bench:update` يحدّث fixture baseline المسجّل في `test/fixtures/cli-startup-bench.json` باستخدام `runs=5` و`warmup=1`

الـ fixture المسجّل:

- `test/fixtures/cli-startup-bench.json`
- حدّثه باستخدام `pnpm test:startup:bench:update`
- قارن النتائج الحالية مع الـ fixture باستخدام `pnpm test:startup:bench:check`

## Onboarding E2E (Docker)

Docker اختياري؛ وهذا مطلوب فقط لاختبارات الدخان الخاصة بالإعداد الأولي ضمن الحاويات.

تدفق بدء تشغيل كامل من الصفر داخل حاوية Linux نظيفة:

```bash
scripts/e2e/onboard-docker.sh
```

يقود هذا السكريبت المعالج التفاعلي عبر pseudo-tty، ويتحقق من ملفات التهيئة/مساحة العمل/الجلسة، ثم يبدأ gateway ويشغّل `openclaw health`.

## اختبار دخان استيراد QR (Docker)

يضمن أن `qrcode-terminal` يُحمَّل ضمن بيئات Node المدعومة في Docker (Node 24 افتراضيًا، وNode 22 متوافق):

```bash
pnpm test:docker:qr
```
