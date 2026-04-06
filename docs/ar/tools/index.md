---
read_when:
    - تريد فهم الأدوات التي يوفرها OpenClaw
    - تحتاج إلى إعداد الأدوات أو السماح بها أو منعها
    - أنت بصدد الاختيار بين الأدوات المدمجة وSkills وplugins
summary: 'نظرة عامة على أدوات OpenClaw وplugins: ما الذي يمكن للوكيل فعله وكيفية توسيعه'
title: الأدوات وPlugins
x-i18n:
    generated_at: "2026-04-06T03:13:30Z"
    model: gpt-5.4
    provider: openai
    source_hash: b2371239316997b0fe389bfa2ec38404e1d3e177755ad81ff8035ac583d9adeb
    source_path: tools/index.md
    workflow: 15
---

# الأدوات وPlugins

كل ما يفعله الوكيل خارج توليد النص يتم من خلال **الأدوات**.
الأدوات هي الطريقة التي يقرأ بها الوكيل الملفات، ويشغّل الأوامر، ويتصفح الويب، ويرسل
الرسائل، ويتفاعل مع الأجهزة.

## الأدوات والمهارات وplugins

يحتوي OpenClaw على ثلاث طبقات تعمل معًا:

<Steps>
  <Step title="الأدوات هي ما يستدعيه الوكيل">
    الأداة هي دالة مكتوبة النوع يمكن للوكيل استدعاؤها (مثل `exec` و`browser`،
    و`web_search` و`message`). يوفّر OpenClaw مجموعة من **الأدوات المدمجة** ويمكن
    للplugins تسجيل أدوات إضافية.

    يرى الوكيل الأدوات على أنها تعريفات دوال مهيكلة تُرسل إلى API الخاصة بالنموذج.

  </Step>

  <Step title="Skills تعلّم الوكيل متى وكيف">
    الـ skill هي ملف markdown ‏(`SKILL.md`) يُحقن في system prompt.
    تمنح Skills الوكيل السياق، والقيود، والإرشادات خطوة بخطوة من أجل
    استخدام الأدوات بفعالية. وتوجد Skills في مساحة العمل الخاصة بك، أو في مجلدات
    مشتركة، أو تأتي ضمن plugins.

    [مرجع Skills](/ar/tools/skills) | [إنشاء Skills](/ar/tools/creating-skills)

  </Step>

  <Step title="Plugins تجمع كل شيء معًا">
    الـ plugin هي حزمة يمكنها تسجيل أي مجموعة من القدرات:
    القنوات، وموفّري النماذج، والأدوات، وSkills، والكلام، والنسخ الفوري،
    والصوت الفوري، وفهم الوسائط، وتوليد الصور، وتوليد الفيديو،
    وجلب الويب، والبحث في الويب، وغير ذلك. بعض plugins تكون **core** (مرفقة مع
    OpenClaw)، وأخرى تكون **خارجية** (منشورة على npm من قبل المجتمع).

    [تثبيت plugins وإعدادها](/ar/tools/plugin) | [ابنِ plugin خاصة بك](/ar/plugins/building-plugins)

  </Step>
</Steps>

## الأدوات المدمجة

تأتي هذه الأدوات مع OpenClaw وتكون متاحة من دون تثبيت أي plugins:

| الأداة                                     | ما الذي تفعله                                                      | الصفحة                                      |
| ------------------------------------------ | ------------------------------------------------------------------ | ------------------------------------------- |
| `exec` / `process`                         | تشغيل أوامر shell، وإدارة العمليات في الخلفية                     | [Exec](/ar/tools/exec)                         |
| `code_execution`                           | تشغيل تحليل Python بعيد ومعزول                                     | [Code Execution](/ar/tools/code-execution)     |
| `browser`                                  | التحكم في متصفح Chromium ‏(التنقل، والنقر، ولقطات الشاشة)          | [Browser](/ar/tools/browser)                   |
| `web_search` / `x_search` / `web_fetch`    | البحث في الويب، والبحث في منشورات X، وجلب محتوى الصفحات          | [الويب](/ar/tools/web)                         |
| `read` / `write` / `edit`                  | إدخال/إخراج الملفات في مساحة العمل                                |                                             |
| `apply_patch`                              | ترقيعات ملفات متعددة المقاطع                                       | [Apply Patch](/ar/tools/apply-patch)           |
| `message`                                  | إرسال الرسائل عبر جميع القنوات                                     | [Agent Send](/ar/tools/agent-send)             |
| `canvas`                                   | تشغيل Canvas الخاص بالعقدة ‏(present وeval وsnapshot)              |                                             |
| `nodes`                                    | اكتشاف الأجهزة المقترنة واستهدافها                                 |                                             |
| `cron` / `gateway`                         | إدارة الوظائف المجدولة؛ وفحص gateway أو ترقيعها أو إعادة تشغيلها أو تحديثها |                                             |
| `image` / `image_generate`                 | تحليل الصور أو توليدها                                             | [توليد الصور](/ar/tools/image-generation)      |
| `music_generate`                           | توليد مقاطع موسيقية                                                | [توليد الموسيقى](/tools/music-generation)   |
| `video_generate`                           | توليد الفيديوهات                                                   | [توليد الفيديو](/tools/video-generation)    |
| `tts`                                      | تحويل نص إلى كلام لمرة واحدة                                       | [TTS](/ar/tools/tts)                           |
| `sessions_*` / `subagents` / `agents_list` | إدارة الجلسات، والحالة، وتنسيق الوكلاء الفرعيين                    | [الوكلاء الفرعيون](/ar/tools/subagents)        |
| `session_status`                           | أداة قراءة خفيفة بأسلوب `/status` وتجاوز نموذج الجلسة              | [أدوات الجلسة](/ar/concepts/session-tool)      |

بالنسبة إلى العمل على الصور، استخدم `image` للتحليل و`image_generate` للتوليد أو التحرير. إذا استهدفت `openai/*` أو `google/*` أو `fal/*` أو موفّر صور غير افتراضي آخر، فقم أولًا بإعداد المصادقة/مفتاح API الخاص بذلك الموفّر.

بالنسبة إلى العمل على الموسيقى، استخدم `music_generate`. إذا استهدفت `google/*` أو `minimax/*` أو موفّر موسيقى غير افتراضي آخر، فقم أولًا بإعداد المصادقة/مفتاح API الخاص بذلك الموفّر.

بالنسبة إلى العمل على الفيديو، استخدم `video_generate`. إذا استهدفت `qwen/*` أو موفّر فيديو غير افتراضي آخر، فقم أولًا بإعداد المصادقة/مفتاح API الخاص بذلك الموفّر.

بالنسبة إلى توليد الصوت المدفوع بسير العمل، استخدم `music_generate` عندما
تسجّله plugin مثل ComfyUI. وهذا منفصل عن `tts`، التي هي تحويل النص إلى كلام.

`session_status` هي أداة الحالة/القراءة الخفيفة في مجموعة sessions.
فهي تجيب عن الأسئلة بأسلوب `/status` حول الجلسة الحالية ويمكنها
اختياريًا تعيين تجاوز نموذج على مستوى الجلسة؛ أما `model=default` فيزيل هذا
التجاوز. ومثل `/status`، يمكنها أن تملأ عدادات الرموز/cache الناقصة و
تسمية نموذج وقت التشغيل النشط انطلاقًا من أحدث إدخال استخدام في transcript.

`gateway` هي أداة وقت التشغيل الخاصة بالمالك فقط لعمليات gateway:

- `config.schema.lookup` من أجل شجرة إعدادات فرعية محددة بمسار قبل التعديلات
- `config.get` من أجل لقطة الإعدادات الحالية + hash
- `config.patch` من أجل تحديثات إعدادات جزئية مع إعادة التشغيل
- `config.apply` فقط من أجل استبدال الإعدادات كاملة
- `update.run` من أجل التحديث الذاتي الصريح + إعادة التشغيل

بالنسبة إلى التغييرات الجزئية، فضّل `config.schema.lookup` ثم `config.patch`.
واستخدم `config.apply` فقط عندما تقصد استبدال الإعدادات بالكامل.
كما ترفض الأداة أيضًا تغيير `tools.exec.ask` أو `tools.exec.security`؛ ويتم
تطبيع الأسماء المستعارة القديمة `tools.bash.*` إلى مسارات exec المحمية نفسها.

### الأدوات التي توفرها plugins

يمكن للplugins تسجيل أدوات إضافية. بعض الأمثلة:

- [Lobster](/ar/tools/lobster) — وقت تشغيل سير عمل مكتوب النوع مع موافقات قابلة للاستئناف
- [LLM Task](/ar/tools/llm-task) — خطوة LLM بصيغة JSON فقط من أجل المخرجات المهيكلة
- [Music Generation](/tools/music-generation) — أداة `music_generate` مشتركة مع موفّرين مدفوعين بسير العمل
- [Diffs](/ar/tools/diffs) — عارض فروق ومُصيّر لها
- [OpenProse](/ar/prose) — تنسيق سير عمل قائم على Markdown أولًا

## إعداد الأدوات

### قوائم السماح والمنع

تحكم في الأدوات التي يمكن للوكيل استدعاؤها عبر `tools.allow` و`tools.deny` في
الإعدادات. ويفوز المنع دائمًا على السماح.

```json5
{
  tools: {
    allow: ["group:fs", "browser", "web_search"],
    deny: ["exec"],
  },
}
```

### ملفات تعريف الأدوات

يضبط `tools.profile` قائمة السماح الأساسية قبل تطبيق `allow`/`deny`.
التجاوز لكل وكيل: `agents.list[].tools.profile`.

| ملف التعريف | ما الذي يتضمنه                                                                                                                                  |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `full`      | بلا قيود (يماثل عدم الضبط)                                                                                                                       |
| `coding`    | `group:fs`, `group:runtime`, `group:web`, `group:sessions`, `group:memory`, `cron`, `image`, `image_generate`, `music_generate`, `video_generate` |
| `messaging` | `group:messaging`, `sessions_list`, `sessions_history`, `sessions_send`, `session_status`                                                        |
| `minimal`   | `session_status` فقط                                                                                                                              |

### مجموعات الأدوات

استخدم اختصارات `group:*` في قوائم السماح/المنع:

| المجموعة           | الأدوات                                                                                                     |
| ------------------ | ----------------------------------------------------------------------------------------------------------- |
| `group:runtime`    | exec, process, code_execution (يُقبل `bash` كاسم مستعار لـ `exec`)                                         |
| `group:fs`         | read, write, edit, apply_patch                                                                              |
| `group:sessions`   | sessions_list, sessions_history, sessions_send, sessions_spawn, sessions_yield, subagents, session_status   |
| `group:memory`     | memory_search, memory_get                                                                                   |
| `group:web`        | web_search, x_search, web_fetch                                                                             |
| `group:ui`         | browser, canvas                                                                                             |
| `group:automation` | cron, gateway                                                                                               |
| `group:messaging`  | message                                                                                                     |
| `group:nodes`      | nodes                                                                                                       |
| `group:agents`     | agents_list                                                                                                 |
| `group:media`      | image, image_generate, music_generate, video_generate, tts                                                  |
| `group:openclaw`   | جميع أدوات OpenClaw المدمجة (باستثناء أدوات plugin)                                                        |

تعيد `sessions_history` عرض استدعاء مقيّدًا ومفلترًا لأسباب الأمان. فهي تزيل
وسوم التفكير، وهياكل `<relevant-memories>`، وحمولات XML النصية البسيطة الخاصة باستدعاءات الأدوات
(بما في ذلك `<tool_call>...</tool_call>`,
و`<function_call>...</function_call>`, و`<tool_calls>...</tool_calls>`,
و`<function_calls>...</function_calls>`,
وكتل استدعاءات الأدوات المقتطعة)، وهياكل استدعاءات الأدوات المخفّضة،
ورموز التحكم بالنموذج المتسربة بصيغة ASCII/العرض الكامل،
وXML استدعاءات الأدوات المشوهة الخاصة بـ MiniMax من نص المساعد، ثم تطبق
الإخفاء/الاقتطاع وإمكانية استخدام عناصر نائبة للصفوف كبيرة الحجم بدلًا من العمل
كتفريغ transcript خام.

### القيود الخاصة بكل موفّر

استخدم `tools.byProvider` لتقييد الأدوات لموفّرين محددين من دون
تغيير الإعدادات العامة الافتراضية:

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
