---
x-i18n:
    generated_at: "2026-04-08T02:19:16Z"
    model: gpt-5.4
    provider: openai
    source_hash: 0e156cc8e2fe946a0423862f937754a7caa1fe7e6863b50a80bff49a1c86e1e8
    source_path: refactor/qa.md
    workflow: 15
---

# إعادة هيكلة QA

الحالة: تم إنجاز الترحيل التأسيسي.

## الهدف

نقل QA في OpenClaw من نموذج تعريف منقسم إلى مصدر حقيقة واحد:

- بيانات السيناريو الوصفية
- الموجّهات المرسلة إلى النموذج
- الإعداد والتنظيف
- منطق أداة التشغيل
- التأكيدات ومعايير النجاح
- العناصر الناتجة وتلميحات التقرير

الحالة النهائية المطلوبة هي أداة QA عامة تقوم بتحميل ملفات تعريف سيناريو قوية بدلًا من ترميز معظم السلوك بشكل ثابت في TypeScript.

## الحالة الحالية

المصدر الأساسي للحقيقة موجود الآن في `qa/scenarios.md`.

تم تنفيذ ما يلي:

- `qa/scenarios.md`
  - حزمة QA القياسية
  - هوية المشغّل
  - مهمة الانطلاق
  - بيانات السيناريو الوصفية
  - روابط المعالِجات
- `extensions/qa-lab/src/scenario-catalog.ts`
  - محلّل حزمة markdown + تحقق zod
- `extensions/qa-lab/src/qa-agent-bootstrap.ts`
  - عرض الخطة من حزمة markdown
- `extensions/qa-lab/src/qa-agent-workspace.ts`
  - يزرع ملفات توافق مولّدة بالإضافة إلى `QA_SCENARIOS.md`
- `extensions/qa-lab/src/suite.ts`
  - يختار السيناريوهات القابلة للتنفيذ عبر روابط المعالِجات المعرّفة في markdown
- بروتوكول ناقل QA + UI
  - مرفقات مضمنة عامة لعرض الصور/الفيديو/الصوت/الملفات

الأسطح المنقسمة المتبقية:

- `extensions/qa-lab/src/suite.ts`
  - لا يزال يملك معظم منطق المعالِجات المخصصة القابلة للتنفيذ
- `extensions/qa-lab/src/report.ts`
  - لا يزال يشتق بنية التقرير من مخرجات runtime

لذا تم إصلاح انقسام مصدر الحقيقة، لكن التنفيذ لا يزال يعتمد في الغالب على المعالِجات بدلًا من أن يكون تصريحيًا بالكامل.

## كيف يبدو سطح السيناريو الحقيقي

تُظهر قراءة الحزمة الحالية بضع فئات مميزة من السيناريوهات.

### تفاعل بسيط

- خط أساس القناة
- خط أساس الرسائل المباشرة
- متابعة مترابطة
- تبديل النموذج
- استكمال الموافقة
- التفاعل/التحرير/الحذف

### تعديل الإعدادات وruntime

- تصحيح config لتعطيل skill
- إيقاظ إعادة تشغيل تطبيق config
- قلب قدرة إعادة تشغيل config
- فحص انجراف مخزون runtime

### تأكيدات نظام الملفات والمستودع

- تقرير اكتشاف المصدر/الوثائق
- بناء Lobster Invaders
- البحث عن عنصر ناتج صورة مولّدة

### تنظيم الذاكرة

- استدعاء الذاكرة
- أدوات الذاكرة في سياق القناة
- احتياط فشل الذاكرة
- ترتيب ذاكرة الجلسة
- عزل ذاكرة السلسلة
- اكتساح أحلام الذاكرة

### تكامل الأدوات وplugin

- استدعاء أدوات plugin الخاصة بـ MCP
- ظهور skill
- تثبيت skill ساخن
- إنشاء صور أصلي
- دورة كاملة للصورة
- فهم الصورة من المرفق

### متعدد الأدوار ومتعدد الأطراف

- تسليم إلى وكيل فرعي
- تجميع توليف الوكيل الفرعي
- تدفقات بأسلوب الاسترداد بعد إعادة التشغيل

هذه الفئات مهمة لأنها تحدد متطلبات DSL. لا تكفي قائمة مسطحة من موجّه + نص متوقع.

## الاتجاه

### مصدر حقيقة واحد

استخدم `qa/scenarios.md` بوصفه مصدر الحقيقة المؤلف.

يجب أن تظل الحزمة:

- قابلة للقراءة البشرية أثناء المراجعة
- قابلة للتحليل آليًا
- غنية بما يكفي لقيادة:
  - تنفيذ الحزمة
  - تمهيد مساحة عمل QA
  - بيانات تعريف QA Lab في UI
  - موجّهات الوثائق/الاكتشاف
  - توليد التقارير

### تنسيق التأليف المفضّل

استخدم markdown باعتباره التنسيق الأعلى، مع YAML منظّم بداخله.

البنية الموصى بها:

- YAML frontmatter
  - id
  - title
  - surface
  - tags
  - مراجع الوثائق
  - مراجع الشيفرة
  - تجاوزات النموذج/الموفر
  - المتطلبات المسبقة
- أقسام نثرية
  - الهدف
  - الملاحظات
  - تلميحات التصحيح
- كتل YAML مسوّرة
  - setup
  - steps
  - assertions
  - cleanup

وهذا يوفّر:

- قابلية قراءة أفضل في PR من JSON ضخم
- سياقًا أغنى من YAML صرف
- تحليلًا صارمًا وتحقق zod

يكون JSON الخام مقبولًا فقط بوصفه صيغة وسيطة مولّدة.

## البنية المقترحة لملف السيناريو

مثال:

````md
---
id: image-generation-roundtrip
title: Image generation roundtrip
surface: image
tags: [media, image, roundtrip]
models:
  primary: openai/gpt-5.4
requires:
  tools: [image_generate]
  plugins: [openai, qa-channel]
docsRefs:
  - docs/help/testing.md
  - docs/concepts/model-providers.md
codeRefs:
  - extensions/qa-lab/src/suite.ts
  - src/gateway/chat-attachments.ts
---

# Objective

Verify generated media is reattached on the follow-up turn.

# Setup

```yaml scenario.setup
- action: config.patch
  patch:
    agents:
      defaults:
        imageGenerationModel:
          primary: openai/gpt-image-1
- action: session.create
  key: agent:qa:image-roundtrip
```

# Steps

```yaml scenario.steps
- action: agent.send
  session: agent:qa:image-roundtrip
  message: |
    Image generation check: generate a QA lighthouse image and summarize it in one short sentence.
- action: artifact.capture
  kind: generated-image
  promptSnippet: Image generation check
  saveAs: lighthouseImage
- action: agent.send
  session: agent:qa:image-roundtrip
  message: |
    Roundtrip image inspection check: describe the generated lighthouse attachment in one short sentence.
  attachments:
    - fromArtifact: lighthouseImage
```

# Expect

```yaml scenario.expect
- assert: outbound.textIncludes
  value: lighthouse
- assert: requestLog.matches
  where:
    promptIncludes: Roundtrip image inspection check
  imageInputCountGte: 1
- assert: artifact.exists
  ref: lighthouseImage
```
````

## قدرات أداة التشغيل التي يجب أن يغطيها DSL

استنادًا إلى الحزمة الحالية، تحتاج أداة التشغيل العامة إلى أكثر من مجرد تنفيذ الموجّهات.

### إجراءات البيئة والإعداد

- `bus.reset`
- `gateway.waitHealthy`
- `channel.waitReady`
- `session.create`
- `thread.create`
- `workspace.writeSkill`

### إجراءات دور الوكيل

- `agent.send`
- `agent.wait`
- `bus.injectInbound`
- `bus.injectOutbound`

### إجراءات الإعدادات وruntime

- `config.get`
- `config.patch`
- `config.apply`
- `gateway.restart`
- `tools.effective`
- `skills.status`

### إجراءات الملفات والعناصر الناتجة

- `file.write`
- `file.read`
- `file.delete`
- `file.touchTime`
- `artifact.captureGeneratedImage`
- `artifact.capturePath`

### إجراءات الذاكرة وcron

- `memory.indexForce`
- `memory.searchCli`
- `doctor.memory.status`
- `cron.list`
- `cron.run`
- `cron.waitCompletion`
- `sessionTranscript.write`

### إجراءات MCP

- `mcp.callTool`

### التأكيدات

- `outbound.textIncludes`
- `outbound.inThread`
- `outbound.notInRoot`
- `tool.called`
- `tool.notPresent`
- `skill.visible`
- `skill.disabled`
- `file.contains`
- `memory.contains`
- `requestLog.matches`
- `sessionStore.matches`
- `cron.managedPresent`
- `artifact.exists`

## المتغيرات ومراجع العناصر الناتجة

يجب أن يدعم DSL المخرجات المحفوظة والمراجع اللاحقة.

أمثلة من الحزمة الحالية:

- إنشاء سلسلة، ثم إعادة استخدام `threadId`
- إنشاء جلسة، ثم إعادة استخدام `sessionKey`
- إنشاء صورة، ثم إرفاق الملف في الدور التالي
- إنشاء سلسلة علامة إيقاظ، ثم التأكيد على ظهورها لاحقًا

القدرات المطلوبة:

- `saveAs`
- `${vars.name}`
- `${artifacts.name}`
- مراجع مكتوبة النوع للمسارات، ومفاتيح الجلسة، ومعرّفات السلاسل، والعلامات، ومخرجات الأدوات

من دون دعم المتغيرات، ستظل أداة التشغيل تسرّب منطق السيناريو إلى TypeScript.

## ما الذي يجب أن يبقى كمنافذ هروب

ليست أداة تشغيل تصريحية نقية بالكامل واقعية في المرحلة 1.

بعض السيناريوهات بطبيعتها كثيفة التنسيق:

- اكتساح أحلام الذاكرة
- إيقاظ إعادة تشغيل تطبيق config
- قلب قدرة إعادة تشغيل config
- حل عنصر الصورة المولّد حسب الطابع الزمني/المسار
- تقييم تقرير الاكتشاف

يجب أن تستخدم هذه السيناريوهات معالِجات مخصصة صريحة في الوقت الحالي.

القاعدة الموصى بها:

- 85-90% تصريحي
- خطوات `customHandler` صريحة للباقي الصعب
- معالِجات مخصصة مسماة وموثقة فقط
- لا شيفرة مضمنة مجهولة داخل ملف السيناريو

يبقي ذلك المحرك العام نظيفًا مع السماح بالتقدم.

## التغيير المعماري

### الحالي

يعد markdown الخاص بالسيناريو بالفعل مصدر الحقيقة لـ:

- تنفيذ الحزمة
- ملفات تمهيد مساحة العمل
- فهرس سيناريو QA Lab في UI
- بيانات التقرير الوصفية
- موجّهات الاكتشاف

التوافق المولّد:

- لا تزال مساحة العمل المزروعة تتضمن `QA_KICKOFF_TASK.md`
- لا تزال مساحة العمل المزروعة تتضمن `QA_SCENARIO_PLAN.md`
- تتضمن مساحة العمل المزروعة الآن أيضًا `QA_SCENARIOS.md`

## خطة إعادة الهيكلة

### المرحلة 1: أداة التحميل والمخطط

تمت.

- تمت إضافة `qa/scenarios.md`
- تمت إضافة محلّل لمحتوى حزمة markdown YAML المسماة
- تم التحقق باستخدام zod
- تم تحويل المستهلكين إلى الحزمة المحللة
- تمت إزالة `qa/seed-scenarios.json` و`qa/QA_KICKOFF_TASK.md` على مستوى المستودع

### المرحلة 2: المحرك العام

- تقسيم `extensions/qa-lab/src/suite.ts` إلى:
  - أداة تحميل
  - محرك
  - سجل إجراءات
  - سجل تأكيدات
  - معالِجات مخصصة
- الإبقاء على الدوال المساعدة الحالية كعمليات للمحرك

المُنجَز:

- ينفذ المحرك السيناريوهات التصريحية البسيطة

ابدأ بالسيناريوهات التي تتكون غالبًا من موجّه + انتظار + تأكيد:

- متابعة مترابطة
- فهم الصورة من المرفق
- ظهور skill واستدعاؤها
- خط أساس القناة

المُنجَز:

- أول سيناريوهات حقيقية معرّفة في markdown تُشحن عبر المحرك العام

### المرحلة 4: ترحيل السيناريوهات المتوسطة

- دورة كاملة لإنشاء الصور
- أدوات الذاكرة في سياق القناة
- ترتيب ذاكرة الجلسة
- تسليم إلى وكيل فرعي
- تجميع توليف الوكيل الفرعي

المُنجَز:

- إثبات المتغيرات والعناصر الناتجة وتأكيدات الأدوات وتأكيدات سجل الطلبات

### المرحلة 5: إبقاء السيناريوهات الصعبة على المعالِجات المخصصة

- اكتساح أحلام الذاكرة
- إيقاظ إعادة تشغيل تطبيق config
- قلب قدرة إعادة تشغيل config
- انجراف مخزون runtime

المُنجَز:

- تنسيق التأليف نفسه، لكن مع كتل خطوات مخصصة صريحة عند الحاجة

### المرحلة 6: حذف خريطة السيناريوهات المرمّزة ثابتًا

بمجرد أن تصبح تغطية الحزمة جيدة بما يكفي:

- إزالة معظم التفرعات الخاصة بالسيناريوهات في TypeScript من `extensions/qa-lab/src/suite.ts`

## دعم Fake Slack / الوسائط الغنية

ناقل QA الحالي يركز على النص أولًا.

الملفات ذات الصلة:

- `extensions/qa-channel/src/protocol.ts`
- `extensions/qa-lab/src/bus-state.ts`
- `extensions/qa-lab/src/bus-queries.ts`
- `extensions/qa-lab/src/bus-server.ts`
- `extensions/qa-lab/web/src/ui-render.ts`

اليوم يدعم ناقل QA:

- النص
- التفاعلات
- السلاسل

ولا يقوم بعد بنمذجة مرفقات الوسائط المضمنة.

### عقد النقل المطلوب

أضف نموذج مرفقات ناقل QA عامًا:

```ts
type QaBusAttachment = {
  id: string;
  kind: "image" | "video" | "audio" | "file";
  mimeType: string;
  fileName?: string;
  inline?: boolean;
  url?: string;
  contentBase64?: string;
  width?: number;
  height?: number;
  durationMs?: number;
  altText?: string;
  transcript?: string;
};
```

ثم أضف `attachments?: QaBusAttachment[]` إلى:

- `QaBusMessage`
- `QaBusInboundMessageInput`
- `QaBusOutboundMessageInput`

### لماذا العام أولًا

لا تبنِ نموذج وسائط خاصًا بـ Slack فقط.

بدلًا من ذلك:

- نموذج نقل QA عام واحد
- عدة عارِضات فوقه
  - دردشة QA Lab الحالية
  - ويب Fake Slack مستقبلي
  - أي عروض نقل وهمية أخرى

يمنع هذا ازدواجية المنطق ويسمح لسيناريوهات الوسائط بأن تظل غير مرتبطة بوسيلة نقل معينة.

### أعمال UI المطلوبة

حدّث UI الخاص بـ QA لعرض:

- معاينة صورة مضمنة
- مشغل صوت مضمن
- مشغل فيديو مضمن
- شريحة مرفق ملف

يمكن لـ UI الحالي بالفعل عرض السلاسل والتفاعلات، لذا يجب أن يضاف عرض المرفقات فوق نموذج بطاقة الرسالة نفسه.

### أعمال السيناريو التي يتيحها نقل الوسائط

بمجرد تدفق المرفقات عبر ناقل QA، يمكننا إضافة سيناريوهات دردشة وهمية أغنى:

- رد بصورة مضمنة في Fake Slack
- فهم مرفق صوتي
- فهم مرفق فيديو
- ترتيب مختلط للمرفقات
- رد في سلسلة مع الاحتفاظ بالوسائط

## التوصية

يجب أن تكون كتلة التنفيذ التالية:

1. إضافة أداة تحميل سيناريو markdown + مخطط zod
2. توليد الفهرس الحالي من markdown
3. ترحيل بعض السيناريوهات البسيطة أولًا
4. إضافة دعم مرفقات ناقل QA عام
5. عرض صورة مضمنة في UI الخاص بـ QA
6. ثم التوسع إلى الصوت والفيديو

هذا هو أصغر مسار يثبت الهدفين معًا:

- QA عام معرّف في markdown
- أسطح مراسلة وهمية أغنى

## أسئلة مفتوحة

- ما إذا كان يجب أن تسمح ملفات السيناريو بقوالب موجّهات markdown مضمنة مع استبدال المتغيرات
- ما إذا كان يجب أن يكون setup/cleanup أقسامًا مسماة أو مجرد قوائم إجراءات مرتبة
- ما إذا كان يجب أن تكون مراجع العناصر الناتجة مكتوبة النوع بقوة في المخطط أو معتمدة على السلاسل
- ما إذا كان يجب أن تعيش المعالِجات المخصصة في سجل واحد أو في سجلات بحسب السطح
- ما إذا كان يجب أن يظل ملف التوافق JSON المولّد محفوظًا في المستودع أثناء الترحيل
