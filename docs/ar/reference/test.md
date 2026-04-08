---
read_when:
    - عند تشغيل الاختبارات أو إصلاحها
summary: كيفية تشغيل الاختبارات محليًا (vitest) ومتى تستخدم أوضاع force/coverage
title: الاختبارات
x-i18n:
    generated_at: "2026-04-08T02:19:21Z"
    model: gpt-5.4
    provider: openai
    source_hash: f7c19390f7577b3a29796c67514c96fe4c86c9fa0c7686cd4e377c6e31dcd085
    source_path: reference/test.md
    workflow: 15
---

# الاختبارات

- عدة الاختبار الكاملة (الأجنحة، وlive، وDocker): [الاختبار](/ar/help/testing)

- `pnpm test:force`: يقتل أي عملية gateway متبقية تحتجز منفذ التحكم الافتراضي، ثم يشغّل جناح Vitest الكامل بمنفذ gateway معزول حتى لا تتعارض اختبارات الخادم مع مثيل قيد التشغيل. استخدم هذا عندما يترك تشغيل gateway سابق المنفذ 18789 مشغولًا.
- `pnpm test:coverage`: يشغّل جناح unit مع تغطية V8 ‏(عبر `vitest.unit.config.ts`). الحدود العامة هي 70% للأسطر/الفروع/الدوال/التعليمات. تستثني التغطية نقاط الدخول الثقيلة على مستوى integration ‏(توصيلات CLI، وجسور gateway/telegram، وخادم webchat الثابت) للإبقاء على الهدف مركزًا على المنطق القابل للاختبار بوحدة.
- `pnpm test:coverage:changed`: يشغّل تغطية unit فقط للملفات المتغيرة منذ `origin/main`.
- `pnpm test:changed`: يوسّع مسارات git المتغيرة إلى مسارات Vitest محددة عندما يمس الفرق فقط ملفات المصدر/الاختبار القابلة للتوجيه. أما تغييرات config/setup فتعود إلى تشغيل root projects الأصلي بحيث تعيد تعديلات التوصيل التشغيل على نطاق واسع عند الحاجة.
- `pnpm test`: يوجّه أهداف الملفات/الأدلة الصريحة عبر مسارات Vitest محددة. أما التشغيلات غير المستهدفة فتنفذ الآن إحدى عشرة إعدادات shard متتابعة (`vitest.full-core-unit-src.config.ts` و`vitest.full-core-unit-security.config.ts` و`vitest.full-core-unit-ui.config.ts` و`vitest.full-core-unit-support.config.ts` و`vitest.full-core-support-boundary.config.ts` و`vitest.full-core-contracts.config.ts` و`vitest.full-core-bundled.config.ts` و`vitest.full-core-runtime.config.ts` و`vitest.full-agentic.config.ts` و`vitest.full-auto-reply.config.ts` و`vitest.full-extensions.config.ts`) بدلًا من عملية root-project ضخمة واحدة.
- تُوجَّه الآن ملفات الاختبار المحددة في `plugin-sdk` و`commands` عبر مسارات خفيفة مخصصة تُبقي فقط `test/setup.ts`، بينما تبقى الحالات الثقيلة على مستوى runtime في مساراتها الحالية.
- كما تربط ملفات المصدر المساعدة المحددة في `plugin-sdk` و`commands` بين `pnpm test:changed` واختبارات شقيقة صريحة في تلك المسارات الخفيفة، بحيث تتجنب تعديلات المساعدات الصغيرة إعادة تشغيل الأجنحة الثقيلة المدعومة بـ runtime.
- ينقسم `auto-reply` الآن أيضًا إلى ثلاثة إعدادات مخصصة (`core` و`top-level` و`reply`) حتى لا تهيمن حاضنة reply على اختبارات الحالة/الرمز/المساعدات الأخف في المستوى الأعلى.
- أصبح إعداد Vitest الأساسي يستخدم افتراضيًا `pool: "threads"` و`isolate: false`، مع تفعيل المشغّل المشترك غير المعزول عبر إعدادات المستودع.
- `pnpm test:channels` يشغّل `vitest.channels.config.ts`.
- `pnpm test:extensions` يشغّل `vitest.extensions.config.ts`.
- `pnpm test:extensions`: يشغّل أجنحة extension/plugin.
- `pnpm test:perf:imports`: يفعّل تقارير مدة الاستيراد + تفصيل الاستيراد في Vitest، مع الاستمرار في استخدام توجيه المسارات المحددة لأهداف الملفات/الأدلة الصريحة.
- `pnpm test:perf:imports:changed`: profiling الاستيراد نفسه، ولكن فقط للملفات المتغيرة منذ `origin/main`.
- `pnpm test:perf:changed:bench -- --ref <git-ref>` يقيس مسار الوضع المتغير الموجّه مقابل تشغيل root-project الأصلي للفرق نفسه في git بعد الالتزام.
- `pnpm test:perf:changed:bench -- --worktree` يقيس مجموعة تغييرات worktree الحالية من دون الالتزام أولًا.
- `pnpm test:perf:profile:main`: يكتب ملف CPU profile لخيط Vitest الرئيسي (`.artifacts/vitest-main-profile`).
- `pnpm test:perf:profile:runner`: يكتب ملفات CPU + heap profiles لمشغّل unit ‏(`.artifacts/vitest-runner-profile`).
- Gateway integration: تفعيل اختياري عبر `OPENCLAW_TEST_INCLUDE_GATEWAY=1 pnpm test` أو `pnpm test:gateway`.
- `pnpm test:e2e`: يشغّل اختبارات الفحص الأساسية end-to-end لـ gateway ‏(تعدد المثيلات، وWS/HTTP، واقتران العقد). يستخدم افتراضيًا `threads` + `isolate: false` مع عمال تكيفيين في `vitest.e2e.config.ts`؛ اضبطه عبر `OPENCLAW_E2E_WORKERS=<n>` واضبط `OPENCLAW_E2E_VERBOSE=1` للحصول على سجلات تفصيلية.
- `pnpm test:live`: يشغّل اختبارات الموفّرين الحية (`minimax`/`zai`). يتطلب مفاتيح API و`LIVE=1` ‏(أو `*_LIVE_TEST=1` الخاص بكل موفّر) لإلغاء التخطي.
- `pnpm test:docker:openwebui`: يبدأ OpenClaw + Open WebUI داخل Docker، ويسجل الدخول عبر Open WebUI، ويفحص `/api/models`، ثم يشغّل دردشة حقيقية عبر proxy من خلال `/api/chat/completions`. ويتطلب مفتاح نموذج حي صالحًا (مثل OpenAI في `~/.profile`)، ويجلب صورة Open WebUI خارجية، ولا يُتوقع أن يكون مستقرًا في CI مثل أجنحة unit/e2e العادية.
- `pnpm test:docker:mcp-channels`: يبدأ حاوية Gateway مهيأة مسبقًا وحاوية عميل ثانية تشغّل `openclaw mcp serve`، ثم يتحقق من اكتشاف المحادثات الموجّهة، وقراءة transcript، وmetadata الخاصة بالمرفقات، وسلوك قائمة الأحداث الحية، وتوجيه الإرسال الصادر، وإشعارات القنوات + الأذونات بأسلوب Claude عبر جسر stdio الحقيقي. ويقرأ تحقق إشعارات Claude إطارات stdio MCP الخام مباشرة بحيث يعكس الفحص ما يصدره الجسر فعليًا.

## بوابة PR المحلية

لإجراء فحوصات بوابة/الهبوط الخاصة بـ PR محليًا، شغّل:

- `pnpm check`
- `pnpm build`
- `pnpm test`
- `pnpm check:docs`

إذا تعطل `pnpm test` بشكل متقطع على مضيف محمّل، فأعد تشغيله مرة واحدة قبل اعتباره تراجعًا، ثم اعزله باستخدام `pnpm test <path/to/test>`. وبالنسبة للمضيفين ذوي الذاكرة المحدودة، استخدم:

- `OPENCLAW_VITEST_MAX_WORKERS=1 pnpm test`
- `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/tmp/openclaw-vitest-cache pnpm test:changed`

## قياس زمن استجابة النموذج (مفاتيح محلية)

السكربت: [`scripts/bench-model.ts`](https://github.com/openclaw/openclaw/blob/main/scripts/bench-model.ts)

الاستخدام:

- `source ~/.profile && pnpm tsx scripts/bench-model.ts --runs 10`
- متغيرات بيئة اختيارية: `MINIMAX_API_KEY` و`MINIMAX_BASE_URL` و`MINIMAX_MODEL` و`ANTHROPIC_API_KEY`
- المطالبة الافتراضية: “Reply with a single word: ok. No punctuation or extra text.”

آخر تشغيل (2025-12-31، 20 تشغيلًا):

- median لـ minimax هو 1279ms ‏(الحد الأدنى 1114، والحد الأقصى 2431)
- median لـ opus هو 2454ms ‏(الحد الأدنى 1224، والحد الأقصى 3170)

## قياس بدء تشغيل CLI

السكربت: [`scripts/bench-cli-startup.ts`](https://github.com/openclaw/openclaw/blob/main/scripts/bench-cli-startup.ts)

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

المجموعات المسبقة:

- `startup`: ‏`--version` و`--help` و`health` و`health --json` و`status --json` و`status`
- `real`: ‏`health` و`status` و`status --json` و`sessions` و`sessions --json` و`agents list --json` و`gateway status` و`gateway status --json` و`gateway health --json` و`config get gateway.port`
- `all`: كلتا المجموعتين المسبقتين

يتضمن الإخراج `sampleCount`، وavg، وp50، وp95، وmin/max، وتوزيع exit-code/signal، وملخصات max RSS لكل أمر. وتكتب الخيارات `--cpu-prof-dir` / `--heap-prof-dir` ملفات V8 profile لكل تشغيل بحيث يستخدم الالتقاط الزمني وprofile الحاضنة نفسها.

اصطلاحات الإخراج المحفوظ:

- يكتب `pnpm test:startup:bench:smoke` ملف smoke المستهدف في `.artifacts/cli-startup-bench-smoke.json`
- يكتب `pnpm test:startup:bench:save` ملف الجناح الكامل في `.artifacts/cli-startup-bench-all.json` باستخدام `runs=5` و`warmup=1`
- يحدّث `pnpm test:startup:bench:update` baseline fixture الملتزم في `test/fixtures/cli-startup-bench.json` باستخدام `runs=5` و`warmup=1`

الـ fixture الملتزم:

- `test/fixtures/cli-startup-bench.json`
- حدّثه عبر `pnpm test:startup:bench:update`
- قارن النتائج الحالية مع الـ fixture عبر `pnpm test:startup:bench:check`

## Onboarding E2E ‏(Docker)

Docker اختياري؛ وهذا مطلوب فقط لاختبارات onboarding الأساسية المعتمدة على الحاويات.

تدفق البدء البارد الكامل داخل حاوية Linux نظيفة:

```bash
scripts/e2e/onboard-docker.sh
```

يقود هذا السكربت المعالج التفاعلي عبر pseudo-tty، ويتحقق من ملفات config/workspace/session، ثم يبدأ gateway ويشغّل `openclaw health`.

## فحص QR import الأساسي (Docker)

يضمن تحميل `qrcode-terminal` تحت بيئات Node المدعومة في Docker ‏(Node 24 افتراضي، وNode 22 متوافق):

```bash
pnpm test:docker:qr
```
