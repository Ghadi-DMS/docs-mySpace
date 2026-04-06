---
read_when: You hit 'sandbox jail' or see a tool/elevated refusal and want the exact config key to change.
status: active
summary: 'سبب حظر أداة ما: وقت تشغيل sandbox، وسياسة السماح/الحظر للأدوات، وبوابات exec المرتفعة'
title: Sandbox مقابل سياسة الأدوات مقابل Elevated
x-i18n:
    generated_at: "2026-04-06T03:07:37Z"
    model: gpt-5.4
    provider: openai
    source_hash: 331f5b2f0d5effa1320125d9f29948e16d0deaffa59eb1e4f25a63481cbe22d6
    source_path: gateway/sandbox-vs-tool-policy-vs-elevated.md
    workflow: 15
---

# Sandbox مقابل سياسة الأدوات مقابل Elevated

يحتوي OpenClaw على ثلاثة عناصر تحكم مترابطة (لكنها مختلفة):

1. **Sandbox** (`agents.defaults.sandbox.*` / `agents.list[].sandbox.*`) يحدد **أين تعمل الأدوات** (Docker أم المضيف).
2. **سياسة الأدوات** (`tools.*`, `tools.sandbox.tools.*`, `agents.list[].tools.*`) تحدد **أي الأدوات متاحة/مسموح بها**.
3. **Elevated** (`tools.elevated.*`, `agents.list[].tools.elevated.*`) هو **منفذ هروب خاص بـ exec فقط** للتشغيل خارج sandbox عندما تكون في وضع sandbox (`gateway` افتراضيًا، أو `node` عندما يكون هدف exec مضبوطًا على `node`).

## تصحيح سريع

استخدم أداة الفحص لمعرفة ما الذي يفعله OpenClaw _فعليًا_:

```bash
openclaw sandbox explain
openclaw sandbox explain --session agent:main:main
openclaw sandbox explain --agent work
openclaw sandbox explain --json
```

يطبع ما يلي:

- وضع sandbox/النطاق/وصول مساحة العمل الفعّال
- ما إذا كانت الجلسة حاليًا داخل sandbox (الرئيسية مقابل غير الرئيسية)
- سياسة السماح/الحظر الفعالة لأدوات sandbox (وما إذا كانت جاءت من الوكيل/الإعدادات العامة/الافتراضية)
- بوابات elevated ومسارات مفاتيح الإصلاح

## Sandbox: أين تعمل الأدوات

يُتحكم في sandboxing عبر `agents.defaults.sandbox.mode`:

- `"off"`: كل شيء يعمل على المضيف.
- `"non-main"`: تُعزل الجلسات غير الرئيسية فقط داخل sandbox (وهذا سبب "المفاجأة" الشائع للمجموعات/القنوات).
- `"all"`: كل شيء داخل sandbox.

راجع [Sandboxing](/ar/gateway/sandboxing) للاطلاع على المصفوفة الكاملة (النطاق، وربط مساحات العمل، والصور).

### ربط Bind mounts (فحص أمني سريع)

- يخترق `docker.binds` نظام ملفات sandbox: كل ما تقوم بربطه يصبح مرئيًا داخل الحاوية بالنمط الذي تعيّنه (`:ro` أو `:rw`).
- الوضع الافتراضي هو القراءة والكتابة إذا حذفت النمط؛ ويفضل استخدام `:ro` للمصدر/الأسرار.
- يتجاهل `scope: "shared"` عمليات الربط الخاصة بكل وكيل (تنطبق عمليات الربط العامة فقط).
- يتحقق OpenClaw من مصادر الربط مرتين: أولًا على مسار المصدر المُطبّع، ثم مرة أخرى بعد الحل عبر أعمق أصل موجود. لا تتجاوز عمليات الهروب عبر أصل الرابط الرمزي عمليات التحقق من المسارات المحظورة أو الجذور المسموح بها.
- تظل مسارات الأوراق غير الموجودة خاضعة للتحقق بشكل آمن. إذا تم حل `/workspace/alias-out/new-file` عبر أصل مرتبط رمزيًا إلى مسار محظور أو إلى خارج الجذور المسموح بها المُهيأة، فسيُرفض الربط.
- إن ربط `/var/run/docker.sock` يمنح فعليًا التحكم بالمضيف إلى sandbox؛ ولا تفعل ذلك إلا عن قصد.
- وصول مساحة العمل (`workspaceAccess: "ro"`/`"rw"`) مستقل عن أوضاع الربط.

## سياسة الأدوات: ما الأدوات الموجودة/القابلة للاستدعاء

هناك مستويان مهمان:

- **ملف تعريف الأداة**: `tools.profile` و`agents.list[].tools.profile` (قائمة السماح الأساسية)
- **ملف تعريف أداة الموفر**: `tools.byProvider[provider].profile` و`agents.list[].tools.byProvider[provider].profile`
- **سياسة الأدوات العامة/لكل وكيل**: `tools.allow`/`tools.deny` و`agents.list[].tools.allow`/`agents.list[].tools.deny`
- **سياسة أداة الموفر**: `tools.byProvider[provider].allow/deny` و`agents.list[].tools.byProvider[provider].allow/deny`
- **سياسة أدوات sandbox** (تنطبق فقط عند العمل داخل sandbox): `tools.sandbox.tools.allow`/`tools.sandbox.tools.deny` و`agents.list[].tools.sandbox.tools.*`

قواعد عامة:

- يفوز `deny` دائمًا.
- إذا كان `allow` غير فارغ، فسيُعامل كل ما عداه على أنه محظور.
- سياسة الأدوات هي نقطة التوقف الصارمة: لا يمكن لـ `/exec` تجاوز أداة `exec` محظورة.
- يغيّر `/exec` الإعدادات الافتراضية للجلسة فقط للمرسلين المصرح لهم؛ ولا يمنح وصولًا إلى الأدوات.
  تقبل مفاتيح أدوات الموفر إما `provider` (مثل `google-antigravity`) أو `provider/model` (مثل `openai/gpt-5.4`).

### مجموعات الأدوات (اختصارات)

تدعم سياسات الأدوات (العامة، والوكيل، وsandbox) إدخالات `group:*` التي تتوسع إلى أدوات متعددة:

```json5
{
  tools: {
    sandbox: {
      tools: {
        allow: ["group:runtime", "group:fs", "group:sessions", "group:memory"],
      },
    },
  },
}
```

المجموعات المتاحة:

- `group:runtime`: `exec`, `process`, `code_execution` (يُقبل `bash` كاسم
  مستعار لـ `exec`)
- `group:fs`: `read`, `write`, `edit`, `apply_patch`
- `group:sessions`: `sessions_list`, `sessions_history`, `sessions_send`, `sessions_spawn`, `sessions_yield`, `subagents`, `session_status`
- `group:memory`: `memory_search`, `memory_get`
- `group:web`: `web_search`, `x_search`, `web_fetch`
- `group:ui`: `browser`, `canvas`
- `group:automation`: `cron`, `gateway`
- `group:messaging`: `message`
- `group:nodes`: `nodes`
- `group:agents`: `agents_list`
- `group:media`: `image`, `image_generate`, `video_generate`, `tts`
- `group:openclaw`: جميع أدوات OpenClaw المضمنة (باستثناء إضافات الموفر)

## Elevated: "التشغيل على المضيف" الخاص بـ exec فقط

لا يمنح Elevated أدوات إضافية؛ بل يؤثر فقط في `exec`.

- إذا كنت داخل sandbox، فإن `/elevated on` (أو `exec` مع `elevated: true`) يُشغَّل خارج sandbox (وقد تظل الموافقات مطلوبة).
- استخدم `/elevated full` لتخطي موافقات exec للجلسة.
- إذا كنت تعمل مباشرة بالفعل، فإن elevated يكون فعليًا بلا تأثير (مع بقائه خاضعًا للبوابات).
- Elevated **ليس** محصورًا بنطاق skill و**لا** يتجاوز سياسة السماح/الحظر للأدوات.
- لا يمنح Elevated تجاوزات عشوائية عبر المضيف من `host=auto`؛ بل يتبع قواعد هدف exec العادية ويحافظ على `node` فقط عندما يكون الهدف المُهيأ/الخاص بالجلسة مضبوطًا بالفعل على `node`.
- `/exec` منفصل عن elevated. فهو يضبط فقط إعدادات exec الافتراضية لكل جلسة للمرسلين المصرح لهم.

البوابات:

- التمكين: `tools.elevated.enabled` (واختياريًا `agents.list[].tools.elevated.enabled`)
- قوائم سماح المرسلين: `tools.elevated.allowFrom.<provider>` (واختياريًا `agents.list[].tools.elevated.allowFrom.<provider>`)

راجع [Elevated Mode](/ar/tools/elevated).

## إصلاحات شائعة لـ "سجن sandbox"

### "Tool X blocked by sandbox tool policy"

مفاتيح الإصلاح (اختر واحدًا):

- تعطيل sandbox: `agents.defaults.sandbox.mode=off` (أو لكل وكيل `agents.list[].sandbox.mode=off`)
- السماح بالأداة داخل sandbox:
  - أزلها من `tools.sandbox.tools.deny` (أو لكل وكيل `agents.list[].tools.sandbox.tools.deny`)
  - أو أضفها إلى `tools.sandbox.tools.allow` (أو إلى قائمة السماح لكل وكيل)

### "I thought this was main, why is it sandboxed?"

في وضع `"non-main"`، لا تُعد مفاتيح المجموعة/القناة جلسات رئيسية. استخدم مفتاح الجلسة الرئيسية (المعروض بواسطة `sandbox explain`) أو بدّل الوضع إلى `"off"`.

## راجع أيضًا

- [Sandboxing](/ar/gateway/sandboxing) -- المرجع الكامل لـ sandbox (الأوضاع، والنطاقات، والواجهات الخلفية، والصور)
- [Sandbox والأدوات في الوكلاء المتعددين](/ar/tools/multi-agent-sandbox-tools) -- التجاوزات لكل وكيل وترتيب الأولوية
- [Elevated Mode](/ar/tools/elevated)
