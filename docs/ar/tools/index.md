---
read_when:
    - تريد فهم الأدوات التي يوفرها OpenClaw
    - تحتاج إلى تهيئة الأدوات أو السماح بها أو منعها
    - أنت تقرر بين الأدوات المضمّنة وSkills وPlugins
summary: 'نظرة عامة على أدوات وإضافات OpenClaw: ما الذي يستطيع الوكيل فعله وكيفية توسيعه'
title: الأدوات والإضافات
x-i18n:
    generated_at: "2026-04-05T12:59:05Z"
    model: gpt-5.4
    provider: openai
    source_hash: 17768048b23f980de5e502cc30fbddbadc2e26ae62f0f03c5ab5bbcdeea67e50
    source_path: tools/index.md
    workflow: 15
---

# الأدوات والإضافات

كل ما يفعله الوكيل خارج توليد النص يحدث عبر **الأدوات**.
الأدوات هي الطريقة التي يقرأ بها الوكيل الملفات، ويشغّل الأوامر، ويتصفح الويب، ويرسل
الرسائل، ويتفاعل مع الأجهزة.

## الأدوات وSkills وPlugins

لدى OpenClaw ثلاث طبقات تعمل معًا:

<Steps>
  <Step title="الأدوات هي ما يستدعيه الوكيل">
    الأداة هي دالة مكتوبة الأنواع يمكن للوكيل استدعاؤها (مثل `exec` و`browser` و
    `web_search` و`message`). يشحن OpenClaw مجموعة من **الأدوات المضمّنة** ويمكن
    لـ plugins تسجيل أدوات إضافية.

    يرى الوكيل الأدوات بوصفها تعريفات دوال منظَّمة تُرسل إلى API النموذج.

  </Step>

  <Step title="Skills تعلّم الوكيل متى وكيف">
    Skill هو ملف markdown (`SKILL.md`) يُحقن في system prompt.
    تمنح Skills الوكيل سياقًا وقيودًا وإرشادًا خطوة بخطوة من أجل
    استخدام الأدوات بفاعلية. تعيش Skills في مساحة العمل لديك، أو في المجلدات المشتركة،
    أو تأتي ضمن plugins.

    [مرجع Skills](/tools/skills) | [إنشاء Skills](/tools/creating-skills)

  </Step>

  <Step title="Plugins تحزم كل شيء معًا">
    plugin هي حزمة يمكنها تسجيل أي مجموعة من القدرات:
    القنوات، وموفرو النماذج، والأدوات، وSkills، والكلام، والنسخ الفوري،
    والصوت الفوري، وفهم الوسائط، وإنشاء الصور، وإنشاء الفيديو،
    وجلب الويب، والبحث في الويب، وغير ذلك. بعض plugins **أساسية** (تأتي مع
    OpenClaw)، وبعضها **خارجية** (ينشرها المجتمع على npm).

    [تثبيت plugins وتهيئتها](/tools/plugin) | [ابنِ plugin خاصًا بك](/ar/plugins/building-plugins)

  </Step>
</Steps>

## الأدوات المضمّنة

تشحن هذه الأدوات مع OpenClaw وتكون متاحة من دون تثبيت أي plugins:

| الأداة | ما الذي تفعله | الصفحة |
| ------ | ------------- | ------ |
| `exec` / `process` | تشغيل أوامر shell، وإدارة العمليات في الخلفية | [Exec](/tools/exec) |
| `code_execution` | تشغيل تحليل Python عن بُعد داخل بيئة معزولة | [Code Execution](/tools/code-execution) |
| `browser` | التحكم في متصفح Chromium (التنقل، والنقر، والتقاط لقطات الشاشة) | [Browser](/tools/browser) |
| `web_search` / `x_search` / `web_fetch` | البحث في الويب، والبحث في منشورات X، وجلب محتوى الصفحات | [Web](/tools/web) |
| `read` / `write` / `edit` | إدخال/إخراج الملفات في مساحة العمل |  |
| `apply_patch` | تصحيحات ملفات متعددة المقاطع | [Apply Patch](/tools/apply-patch) |
| `message` | إرسال الرسائل عبر جميع القنوات | [Agent Send](/tools/agent-send) |
| `canvas` | تشغيل Canvas الخاصة بـ node (عرض، eval، snapshot) |  |
| `nodes` | اكتشاف الأجهزة المقترنة واستهدافها |  |
| `cron` / `gateway` | إدارة المهام المجدولة؛ وفحص البوابة أو ترقيعها أو إعادة تشغيلها أو تحديثها |  |
| `image` / `image_generate` | تحليل الصور أو إنشاؤها |  |
| `tts` | تحويل نص إلى كلام لمرة واحدة | [TTS](/tools/tts) |
| `sessions_*` / `subagents` / `agents_list` | إدارة الجلسات، والحالة، وتنسيق الوكلاء الفرعيين | [Sub-agents](/tools/subagents) |
| `session_status` | استرجاع خفيف على نمط `/status` وتجاوز نموذج الجلسة | [Session Tools](/ar/concepts/session-tool) |

بالنسبة إلى العمل على الصور، استخدم `image` للتحليل و`image_generate` للإنشاء أو التحرير. إذا كنت تستهدف `openai/*` أو `google/*` أو `fal/*` أو موفر صور غير افتراضي آخر، فقم أولًا بتهيئة المصادقة/مفتاح API الخاص بذلك الموفّر.

`session_status` هي أداة الحالة/الاسترجاع الخفيفة ضمن مجموعة sessions.
وهي تجيب عن الأسئلة على نمط `/status` حول الجلسة الحالية، ويمكنها
اختياريًا تعيين تجاوز نموذج لكل جلسة؛ تؤدي `model=default` إلى مسح ذلك
التجاوز. ومثل `/status`، يمكنها استكمال عدادات الرموز/التخزين المؤقت المتفرقة وعلامة
نموذج runtime النشط من أحدث إدخال استخدام في النص.

`gateway` هي أداة runtime خاصة بالمالك لعمليات البوابة:

- `config.schema.lookup` لشجرة فرعية واحدة من التكوين ضمن نطاق مسار قبل التعديلات
- `config.get` للحصول على لقطة التكوين الحالية + hash
- `config.patch` لتحديثات تكوين جزئية مع إعادة التشغيل
- `config.apply` فقط لاستبدال التكوين كاملًا
- `update.run` لإجراء تحديث ذاتي صريح + إعادة تشغيل

للتغييرات الجزئية، يُفضّل استخدام `config.schema.lookup` ثم `config.patch`. استخدم
`config.apply` فقط عندما تكون تنوي عمدًا استبدال التكوين بالكامل.
كما ترفض الأداة أيضًا تغيير `tools.exec.ask` أو `tools.exec.security`؛
وتُطبَّع الأسماء المستعارة القديمة `tools.bash.*` إلى مسارات exec المحمية نفسها.

### أدوات توفرها plugins

يمكن لـ plugins تسجيل أدوات إضافية. بعض الأمثلة:

- [Lobster](/tools/lobster) — بيئة تشغيل سير عمل مكتوبة الأنواع مع موافقات قابلة للاستئناف
- [LLM Task](/tools/llm-task) — خطوة LLM تعمل بـ JSON فقط للحصول على مخرجات منظَّمة
- [Diffs](/tools/diffs) — عارض ومُصيِّر للفروق
- [OpenProse](/ar/prose) — تنسيق سير عمل يقوم على Markdown أولًا

## تهيئة الأدوات

### قوائم السماح والمنع

تحكّم في الأدوات التي يمكن للوكيل استدعاؤها عبر `tools.allow` / `tools.deny` في
التكوين. المنع يتغلب دائمًا على السماح.

```json5
{
  tools: {
    allow: ["group:fs", "browser", "web_search"],
    deny: ["exec"],
  },
}
```

### ملفات تعريف الأدوات

يضبط `tools.profile` قائمة سماح أساسية قبل تطبيق `allow`/`deny`.
التجاوز لكل وكيل: `agents.list[].tools.profile`.

| ملف التعريف | ما الذي يتضمنه |
| ----------- | -------------- |
| `full` | بلا قيود (مثل عدم تعيينه) |
| `coding` | `group:fs`, `group:runtime`, `group:web`, `group:sessions`, `group:memory`, `cron`, `image`, `image_generate` |
| `messaging` | `group:messaging`, `sessions_list`, `sessions_history`, `sessions_send`, `session_status` |
| `minimal` | `session_status` فقط |

### مجموعات الأدوات

استخدم صيغ الاختصار `group:*` في قوائم السماح/المنع:

| المجموعة | الأدوات |
| -------- | ------- |
| `group:runtime` | exec, process, code_execution (`bash` مقبول بوصفه اسمًا مستعارًا لـ `exec`) |
| `group:fs` | read, write, edit, apply_patch |
| `group:sessions` | sessions_list, sessions_history, sessions_send, sessions_spawn, sessions_yield, subagents, session_status |
| `group:memory` | memory_search, memory_get |
| `group:web` | web_search, x_search, web_fetch |
| `group:ui` | browser, canvas |
| `group:automation` | cron, gateway |
| `group:messaging` | message |
| `group:nodes` | nodes |
| `group:agents` | agents_list |
| `group:media` | image, image_generate, tts |
| `group:openclaw` | جميع أدوات OpenClaw المضمّنة (باستثناء أدوات plugins) |

يعيد `sessions_history` عرض استرجاع محدودًا ومفلترًا لأغراض السلامة. فهو يزيل
وسوم التفكير، وهيكل `<relevant-memories>`، وحمولات XML النصية العادية
لاستدعاءات الأدوات (بما في ذلك `<tool_call>...</tool_call>`،
و`<function_call>...</function_call>`، و`<tool_calls>...</tool_calls>`،
و`<function_calls>...</function_calls>`، وكتل استدعاء الأدوات المقتطعة)،
وهيكل استدعاء الأدوات المُخفَّض، ورموز التحكم بالنموذج المسرّبة بصيغة ASCII/العرض الكامل،
وXML المعطوب الخاص باستدعاءات أدوات MiniMax من نص المساعد، ثم يطبّق
الحجب/الاقتطاع وإمكانية وضع عناصر نائبة للأسطر كبيرة الحجم بدلًا من العمل
بوصفه تفريغًا خامًا للنص.

### القيود الخاصة بالموفر

استخدم `tools.byProvider` لتقييد الأدوات لموفرين محددين من دون
تغيير القيم الافتراضية العامة:

```json5
{
  tools: {
    profile: "coding",
    byProvider: {
      "google-antigravity": { profile: "minimal" },
    },
  },
}
```
