---
read_when: You hit 'sandbox jail' or see a tool/elevated refusal and want the exact config key to change.
status: active
summary: 'سبب حظر أداة ما: بيئة تشغيل sandbox، وسياسة السماح/المنع للأدوات، وبوابات exec المرتفعة الصلاحيات'
title: Sandbox مقابل Tool Policy مقابل Elevated
x-i18n:
    generated_at: "2026-04-05T12:44:10Z"
    model: gpt-5.4
    provider: openai
    source_hash: 8d5ddc1dbf02b89f18d46e5473ff0a29b8a984426fe2db7270c170f2de0cdeac
    source_path: gateway/sandbox-vs-tool-policy-vs-elevated.md
    workflow: 15
---

# Sandbox مقابل Tool Policy مقابل Elevated

يحتوي OpenClaw على ثلاثة عناصر تحكم مترابطة (لكنها مختلفة):

1. **Sandbox** (`agents.defaults.sandbox.*` / `agents.list[].sandbox.*`) يحدد **مكان تشغيل الأدوات** (Docker أم المضيف).
2. **Tool policy** (`tools.*` و`tools.sandbox.tools.*` و`agents.list[].tools.*`) تحدد **أي الأدوات متاحة/مسموح بها**.
3. **Elevated** (`tools.elevated.*` و`agents.list[].tools.elevated.*`) هو **منفذ هروب خاص بـ exec فقط** للتشغيل خارج sandbox عندما تكون داخل sandbox (`gateway` افتراضيًا، أو `node` عندما يكون هدف exec مضبوطًا على `node`).

## تصحيح سريع

استخدم أداة الفحص لمعرفة ما الذي يفعله OpenClaw _فعليًا_:

```bash
openclaw sandbox explain
openclaw sandbox explain --session agent:main:main
openclaw sandbox explain --agent work
openclaw sandbox explain --json
```

ويطبع ما يلي:

- وضع/nطاق/وصول مساحة العمل الفعلي لـ sandbox
- ما إذا كانت الجلسة تعمل حاليًا داخل sandbox (main مقابل non-main)
- سياسة السماح/المنع الفعلية لأدوات sandbox (وما إذا كانت جاءت من agent/global/default)
- بوابات elevated ومسارات المفاتيح المقترحة للإصلاح

## Sandbox: أين تعمل الأدوات

يتم التحكم في sandboxing بواسطة `agents.defaults.sandbox.mode`:

- `"off"`: كل شيء يعمل على المضيف.
- `"non-main"`: تُعزل فقط الجلسات غير الرئيسية (وهذا "مفاجأة" شائعة للمجموعات/القنوات).
- `"all"`: كل شيء يعمل داخل sandbox.

راجع [Sandboxing](/gateway/sandboxing) للاطلاع على المصفوفة الكاملة (النطاق، وعمليات تحميل مساحة العمل، والصور).

### bind mounts (فحص أمني سريع)

- يقوم `docker.binds` _باختراق_ نظام ملفات sandbox: فكل ما تقوم بتحميله يصبح مرئيًا داخل الحاوية بالنمط الذي تضبطه (`:ro` أو `:rw`).
- يكون الوضع الافتراضي هو القراءة والكتابة إذا حذفت النمط؛ ويفضّل استخدام `:ro` للمصدر/الأسرار.
- يتجاهل `scope: "shared"` عمليات bind الخاصة بكل وكيل (ولا تُطبّق إلا عمليات bind العامة).
- يتحقق OpenClaw من مصادر bind مرتين: أولًا على مسار المصدر الموحّد، ثم مرة أخرى بعد الحل عبر أعمق سلف موجود. ولا تتجاوز عمليات الهروب عبر symlink-parent فحوصات المسارات المحظورة أو الجذور المسموح بها.
- ما زالت مسارات الأوراق غير الموجودة تُفحص بأمان. فإذا تم حل `/workspace/alias-out/new-file` عبر parent مرتبط بـ symlink إلى مسار محظور أو خارج الجذور المسموح بها المكوّنة، فسيتم رفض bind.
- يؤدي bind الخاص بـ `/var/run/docker.sock` فعليًا إلى منح التحكم بالمضيف إلى sandbox؛ فلا تفعل ذلك إلا عن قصد.
- يكون وصول مساحة العمل (`workspaceAccess: "ro"`/`"rw"`) مستقلاً عن أنماط bind.

## Tool policy: أي الأدوات موجودة/يمكن استدعاؤها

توجد طبقتان مهمتان:

- **Tool profile**: ‏`tools.profile` و`agents.list[].tools.profile` (قائمة السماح الأساسية)
- **Provider tool profile**: ‏`tools.byProvider[provider].profile` و`agents.list[].tools.byProvider[provider].profile`
- **Global/per-agent tool policy**: ‏`tools.allow`/`tools.deny` و`agents.list[].tools.allow`/`agents.list[].tools.deny`
- **Provider tool policy**: ‏`tools.byProvider[provider].allow/deny` و`agents.list[].tools.byProvider[provider].allow/deny`
- **Sandbox tool policy** (تنطبق فقط عند العمل داخل sandbox): ‏`tools.sandbox.tools.allow`/`tools.sandbox.tools.deny` و`agents.list[].tools.sandbox.tools.*`

قواعد أساسية:

- `deny` يفوز دائمًا.
- إذا كانت `allow` غير فارغة، فسيُعامل كل ما عداها على أنه محظور.
- تكون Tool policy هي نقطة التوقف الصارمة: لا يمكن لـ `/exec` تجاوز أداة `exec` المحظورة.
- يغيّر `/exec` الإعدادات الافتراضية للجلسة فقط للمرسلين المصرح لهم؛ ولا يمنح وصولًا إلى الأدوات.
  تقبل مفاتيح أدوات المزوّد إما `provider` (مثل `google-antigravity`) أو `provider/model` (مثل `openai/gpt-5.4`).

### مجموعات الأدوات (اختصارات)

تدعم سياسات الأدوات (العامة، ولكل وكيل، وداخل sandbox) إدخالات `group:*` التي تتوسع إلى عدة أدوات:

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

- `group:runtime`: ‏`exec` و`process` و`code_execution` (ويُقبل `bash` كاسم مستعار
  لـ `exec`)
- `group:fs`: ‏`read` و`write` و`edit` و`apply_patch`
- `group:sessions`: ‏`sessions_list` و`sessions_history` و`sessions_send` و`sessions_spawn` و`sessions_yield` و`subagents` و`session_status`
- `group:memory`: ‏`memory_search` و`memory_get`
- `group:web`: ‏`web_search` و`x_search` و`web_fetch`
- `group:ui`: ‏`browser` و`canvas`
- `group:automation`: ‏`cron` و`gateway`
- `group:messaging`: ‏`message`
- `group:nodes`: ‏`nodes`
- `group:agents`: ‏`agents_list`
- `group:media`: ‏`image` و`image_generate` و`tts`
- `group:openclaw`: جميع أدوات OpenClaw المضمنة (باستثناء plugins الخاصة بالمزوّدين)

## Elevated: "تشغيل على المضيف" خاص بـ exec فقط

لا يمنح Elevated أدوات إضافية؛ فهو يؤثر فقط في `exec`.

- إذا كنت داخل sandbox، فإن `/elevated on` (أو `exec` مع `elevated: true`) يعمل خارج sandbox (وقد تظل الموافقات مطلوبة).
- استخدم `/elevated full` لتخطي موافقات exec للجلسة.
- إذا كنت تعمل بالفعل بشكل مباشر، فإن elevated يصبح فعليًا بلا تأثير (مع استمرار التقييد).
- لا يكون Elevated ضمن نطاق Skills و**لا** يتجاوز allow/deny الخاصة بالأدوات.
- لا يمنح Elevated تجاوزات عشوائية عبر المضيف من `host=auto`؛ بل يتبع قواعد هدف exec العادية ولا يحتفظ إلا بـ `node` عندما يكون الهدف المكوّن/الخاص بالجلسة مضبوطًا أصلًا على `node`.
- يختلف `/exec` عن elevated. فهو يضبط فقط الإعدادات الافتراضية الخاصة بـ exec لكل جلسة للمرسلين المصرح لهم.

البوابات:

- التمكين: `tools.elevated.enabled` (واختياريًا `agents.list[].tools.elevated.enabled`)
- allowlists الخاصة بالمرسلين: ‏`tools.elevated.allowFrom.<provider>` (واختياريًا `agents.list[].tools.elevated.allowFrom.<provider>`)

راجع [Elevated Mode](/tools/elevated).

## إصلاحات شائعة لـ "سجن sandbox"

### "الأداة X محظورة بواسطة سياسة أدوات sandbox"

مفاتيح الإصلاح (اختر واحدًا):

- تعطيل sandbox: ‏`agents.defaults.sandbox.mode=off` (أو لكل وكيل `agents.list[].sandbox.mode=off`)
- السماح بالأداة داخل sandbox:
  - أزلها من `tools.sandbox.tools.deny` (أو `agents.list[].tools.sandbox.tools.deny` لكل وكيل)
  - أو أضفها إلى `tools.sandbox.tools.allow` (أو allow الخاصة بكل وكيل)

### "كنت أظن أن هذه هي main، فلماذا هي داخل sandbox؟"

في وضع `"non-main"`، لا تُعد مفاتيح المجموعة/القناة هي main. استخدم مفتاح الجلسة الرئيسي (المعروض بواسطة `sandbox explain`) أو غيّر الوضع إلى `"off"`.

## راجع أيضًا

- [Sandboxing](/gateway/sandboxing) -- المرجع الكامل لـ sandbox (الأوضاع، والنطاقات، والواجهات الخلفية، والصور)
- [Multi-Agent Sandbox & Tools](/tools/multi-agent-sandbox-tools) -- التجاوزات لكل وكيل وترتيب الأولوية
- [Elevated Mode](/tools/elevated)
