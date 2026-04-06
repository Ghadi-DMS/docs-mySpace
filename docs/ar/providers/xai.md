---
read_when:
    - تريد استخدام نماذج Grok في OpenClaw
    - أنت تهيئ مصادقة xAI أو معرّفات النماذج
summary: استخدام نماذج xAI Grok في OpenClaw
title: xAI
x-i18n:
    generated_at: "2026-04-06T03:12:10Z"
    model: gpt-5.4
    provider: openai
    source_hash: 64bc899655427cc10bdc759171c7d1ec25ad9f1e4f9d803f1553d3d586c6d71d
    source_path: providers/xai.md
    workflow: 15
---

# xAI

يشحن OpenClaw plugin موفّر `xai` مضمّنًا لنماذج Grok.

## الإعداد

1. أنشئ مفتاح API في وحدة تحكم xAI.
2. عيّن `XAI_API_KEY`، أو شغّل:

```bash
openclaw onboard --auth-choice xai-api-key
```

3. اختر نموذجًا مثل:

```json5
{
  agents: { defaults: { model: { primary: "xai/grok-4" } } },
}
```

يستخدم OpenClaw الآن xAI Responses API كناقل xAI المضمّن. ويمكن للمفتاح نفسه
`XAI_API_KEY` أيضًا تشغيل `web_search` المدعوم بـ Grok، و`x_search` الأصلي،
و`code_execution` البعيد.
إذا خزّنت مفتاح xAI تحت `plugins.entries.xai.config.webSearch.apiKey`،
فإن موفّر نموذج xAI المضمّن يعيد استخدام ذلك المفتاح الآن كبديل احتياطي أيضًا.
توجد إعدادات ضبط `code_execution` تحت `plugins.entries.xai.config.codeExecution`.

## فهرس النماذج المضمن الحالي

يتضمن OpenClaw الآن عائلات نماذج xAI التالية بشكل مضمّن:

- `grok-3`, `grok-3-fast`, `grok-3-mini`, `grok-3-mini-fast`
- `grok-4`, `grok-4-0709`
- `grok-4-fast`, `grok-4-fast-non-reasoning`
- `grok-4-1-fast`, `grok-4-1-fast-non-reasoning`
- `grok-4.20-beta-latest-reasoning`, `grok-4.20-beta-latest-non-reasoning`
- `grok-code-fast-1`

كما يحل plugin استباقيًا أيضًا معرّفات `grok-4*` و`grok-code-fast*` الأحدث عندما
تتبع شكل API نفسه.

ملاحظات النماذج السريعة:

- تمثل `grok-4-fast` و`grok-4-1-fast` ومتغيرات `grok-4.20-beta-*`
  مراجع Grok الحالية القادرة على الصور في الفهرس المضمّن.
- يعيد `/fast on` أو `agents.defaults.models["xai/<model>"].params.fastMode: true`
  كتابة طلبات xAI الأصلية على النحو التالي:
  - `grok-3` -> `grok-3-fast`
  - `grok-3-mini` -> `grok-3-mini-fast`
  - `grok-4` -> `grok-4-fast`
  - `grok-4-0709` -> `grok-4-fast`

لا تزال الأسماء المستعارة القديمة للتوافق تُطبّع إلى المعرّفات المضمنة القياسية. على
سبيل المثال:

- `grok-4-fast-reasoning` -> `grok-4-fast`
- `grok-4-1-fast-reasoning` -> `grok-4-1-fast`
- `grok-4.20-reasoning` -> `grok-4.20-beta-latest-reasoning`
- `grok-4.20-non-reasoning` -> `grok-4.20-beta-latest-non-reasoning`

## البحث على الويب

يستخدم موفّر البحث على الويب `grok` المضمّن أيضًا `XAI_API_KEY`:

```bash
openclaw config set tools.web.search.provider grok
```

## إنشاء الفيديو

يسجل plugin المضمن `xai` أيضًا إنشاء الفيديو عبر الأداة المشتركة
`video_generate`.

- نموذج الفيديو الافتراضي: `xai/grok-imagine-video`
- الأوضاع: نص إلى فيديو، وصورة إلى فيديو، وتدفقات تحرير/تمديد الفيديو البعيدة
- يدعم `aspectRatio` و`resolution`
- القيد الحالي: لا تُقبل مخازن الفيديو المحلية؛ استخدم عناوين URL بعيدة من نوع `http(s)`
  لمدخلات مرجع/تحرير الفيديو

لاستخدام xAI بوصفه موفّر الفيديو الافتراضي:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "xai/grok-imagine-video",
      },
    },
  },
}
```

راجع [إنشاء الفيديو](/tools/video-generation) للاطلاع على معلمات
الأداة المشتركة، واختيار الموفّر، وسلوك التحويل الاحتياطي.

## القيود المعروفة

- المصادقة بمفتاح API فقط حاليًا. لا يوجد حتى الآن تدفق xAI OAuth/device-code في OpenClaw.
- لا يتم دعم `grok-4.20-multi-agent-experimental-beta-0304` على مسار موفّر xAI العادي لأنه يتطلب سطح API مختلفًا في upstream عن ناقل OpenClaw xAI القياسي.

## ملاحظات

- يطبق OpenClaw تلقائيًا إصلاحات توافق خاصة بـ xAI لمخطط الأدوات واستدعاء الأدوات على مسار المشغّل المشترك.
- تُفعَّل طلبات xAI الأصلية افتراضيًا مع `tool_stream: true`. عيّن
  `agents.defaults.models["xai/<model>"].params.tool_stream` إلى `false` من أجل
  تعطيله.
- يزيل الغلاف المضمن لـ xAI علامات strict tool-schema غير المدعومة ومفاتيح
  حمولة reasoning قبل إرسال طلبات xAI الأصلية.
- تُعرض `web_search` و`x_search` و`code_execution` كأدوات OpenClaw. ويمكّن OpenClaw المكوّن الأصلي المحدد في xAI الذي يحتاجه داخل كل طلب أداة بدلًا من إرفاق جميع الأدوات الأصلية بكل دور دردشة.
- `x_search` و`code_execution` مملوكان للـ plugin المضمن xAI بدلًا من ترميزهما بشكل ثابت داخل runtime النماذج الأساسي.
- `code_execution` هو تنفيذ صندوق حماية xAI بعيد، وليس [`exec`](/ar/tools/exec) محليًا.
- للحصول على نظرة عامة أشمل على الموفّرين، راجع [موفرو النماذج](/ar/providers/index).
