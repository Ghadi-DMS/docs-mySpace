---
read_when:
    - تريد استخدام نماذج Grok في OpenClaw
    - أنت تضبط مصادقة xAI أو معرّفات النماذج
summary: استخدام نماذج Grok من xAI في OpenClaw
title: xAI
x-i18n:
    generated_at: "2026-04-05T12:54:21Z"
    model: gpt-5.4
    provider: openai
    source_hash: d11f27b48c69eed6324595977bca3506c7709424eef64cc73899f8d049148b82
    source_path: providers/xai.md
    workflow: 15
---

# xAI

يأتي OpenClaw مع plugin مزوّد مضمّنة باسم `xai` لنماذج Grok.

## الإعداد

1. أنشئ مفتاح API في لوحة xAI.
2. اضبط `XAI_API_KEY`، أو شغّل:

```bash
openclaw onboard --auth-choice xai-api-key
```

3. اختر نموذجًا مثل:

```json5
{
  agents: { defaults: { model: { primary: "xai/grok-4" } } },
}
```

يستخدم OpenClaw الآن xAI Responses API كوسيلة النقل المضمّنة الخاصة بـ xAI. ويمكن لمفتاح
`XAI_API_KEY` نفسه أيضًا تشغيل `web_search` المدعومة بـ Grok، و`x_search` من الدرجة الأولى،
و`code_execution` البعيدة.
إذا خزّنت مفتاح xAI تحت `plugins.entries.xai.config.webSearch.apiKey`,
فإن مزوّد النماذج المضمّن الخاص بـ xAI يعيد استخدام ذلك المفتاح الآن كرجوع احتياطي أيضًا.
يعيش ضبط `code_execution` تحت `plugins.entries.xai.config.codeExecution`.

## فهرس النماذج المضمّن الحالي

يتضمن OpenClaw الآن عائلات نماذج xAI التالية جاهزة للاستخدام:

- `grok-3`, `grok-3-fast`, `grok-3-mini`, `grok-3-mini-fast`
- `grok-4`, `grok-4-0709`
- `grok-4-fast`, `grok-4-fast-non-reasoning`
- `grok-4-1-fast`, `grok-4-1-fast-non-reasoning`
- `grok-4.20-beta-latest-reasoning`, `grok-4.20-beta-latest-non-reasoning`
- `grok-code-fast-1`

كما تقوم plugin أيضًا بتحليل معرّفات `grok-4*` و`grok-code-fast*` الأحدث
مسبقًا عندما تتبع شكل API نفسه.

ملاحظات حول النماذج السريعة:

- تمثل `grok-4-fast` و`grok-4-1-fast` ومتغيرات `grok-4.20-beta-*`
  مراجع Grok الحالية القادرة على التعامل مع الصور في الفهرس المضمّن.
- تعيد `/fast on` أو `agents.defaults.models["xai/<model>"].params.fastMode: true`
  كتابة طلبات xAI الأصلية كما يلي:
  - `grok-3` -> `grok-3-fast`
  - `grok-3-mini` -> `grok-3-mini-fast`
  - `grok-4` -> `grok-4-fast`
  - `grok-4-0709` -> `grok-4-fast`

ما تزال الأسماء المستعارة القديمة للتوافق تُطبّع إلى المعرّفات المضمّنة الأساسية. على
سبيل المثال:

- `grok-4-fast-reasoning` -> `grok-4-fast`
- `grok-4-1-fast-reasoning` -> `grok-4-1-fast`
- `grok-4.20-reasoning` -> `grok-4.20-beta-latest-reasoning`
- `grok-4.20-non-reasoning` -> `grok-4.20-beta-latest-non-reasoning`

## Web search

يستخدم مزوّد البحث على الويب `grok` المضمّن القيمة `XAI_API_KEY` أيضًا:

```bash
openclaw config set tools.web.search.provider grok
```

## القيود المعروفة

- المصادقة تعتمد على مفتاح API فقط حاليًا. ولا يوجد بعد تدفق xAI OAuth/device-code في OpenClaw.
- لا يكون `grok-4.20-multi-agent-experimental-beta-0304` مدعومًا على مسار المزوّد العادي لـ xAI لأنه يتطلب سطح API مختلفًا في المنبع عن وسيلة النقل القياسية لـ xAI في OpenClaw.

## ملاحظات

- يطبّق OpenClaw إصلاحات التوافق الخاصة بـ xAI تلقائيًا على schemas الأدوات واستدعاءات الأدوات على مسار المشغّل المشترك.
- تضبط طلبات xAI الأصلية القيمة الافتراضية `tool_stream: true`. اضبط
  `agents.defaults.models["xai/<model>"].params.tool_stream` على `false` من أجل
  تعطيلها.
- يزيل الغلاف المضمّن لـ xAI أعلام schema الصارمة غير المدعومة للأدوات ومفاتيح
  حمولة reasoning قبل إرسال طلبات xAI الأصلية.
- تُكشف `web_search` و`x_search` و`code_execution` كأدوات OpenClaw. ويمكّن OpenClaw الوظيفة المضمّنة المحددة من xAI التي يحتاجها داخل كل طلب أداة بدلًا من إرفاق جميع الأدوات الأصلية بكل دور دردشة.
- تكون `x_search` و`code_execution` مملوكتين للـ plugin المضمّنة الخاصة بـ xAI بدلًا من تضمينهما بشكل صلب في وقت تشغيل النموذج الأساسي.
- تكون `code_execution` تنفيذ sandbox بعيدًا في xAI، وليست [`exec`](/tools/exec) محلية.
- للاطلاع على نظرة أوسع على المزوّدات، راجع [Model providers](/providers/index).
