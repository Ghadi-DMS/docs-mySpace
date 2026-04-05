---
read_when:
    - تريد تثبيت حزمة متوافقة مع Codex أو Claude أو Cursor
    - تحتاج إلى فهم كيفية تعيين OpenClaw لمحتوى الحزمة إلى الميزات الأصلية
    - أنت تصحح أخطاء اكتشاف الحزمة أو الإمكانات المفقودة
summary: تثبيت واستخدام حزم Codex وClaude وCursor كـ plugins في OpenClaw
title: حزم plugins
x-i18n:
    generated_at: "2026-04-05T12:51:17Z"
    model: gpt-5.4
    provider: openai
    source_hash: a8b1eb4633bdff75425d8c2e29be352e11a4cdad7f420c0c66ae5ef07bf9bdcc
    source_path: plugins/bundles.md
    workflow: 15
---

# حزم plugins

يمكن لـ OpenClaw تثبيت plugins من ثلاثة أنظمة خارجية: **Codex** و**Claude**،
و**Cursor**. وتسمى هذه **حزم** — وهي حزم محتوى وبيانات وصفية
يقوم OpenClaw بتعيينها إلى ميزات أصلية مثل Skills وhooks وأدوات MCP.

<Info>
  الحزم **ليست** هي نفسها plugins الأصلية في OpenClaw. فالـ plugins الأصلية تعمل
  داخل العملية ويمكنها تسجيل أي إمكانية. أما الحزم فهي حزم محتوى مع
  تعيين انتقائي للميزات وحد ثقة أضيق.
</Info>

## لماذا توجد الحزم

يتم نشر العديد من plugins المفيدة بصيغة Codex أو Claude أو Cursor. وبدلًا
من مطالبة المؤلفين بإعادة كتابتها كـ plugins أصلية في OpenClaw، يقوم OpenClaw
باكتشاف هذه الصيغ وتعيين محتواها المدعوم إلى مجموعة الميزات الأصلية.
وهذا يعني أنه يمكنك تثبيت حزمة أوامر Claude أو حزمة Skills من Codex
واستخدامها فورًا.

## تثبيت حزمة

<Steps>
  <Step title="التثبيت من دليل، أو أرشيف، أو Marketplace">
    ```bash
    # Local directory
    openclaw plugins install ./my-bundle

    # Archive
    openclaw plugins install ./my-bundle.tgz

    # Claude marketplace
    openclaw plugins marketplace list <marketplace-name>
    openclaw plugins install <plugin-name>@<marketplace-name>
    ```

  </Step>

  <Step title="التحقق من الاكتشاف">
    ```bash
    openclaw plugins list
    openclaw plugins inspect <id>
    ```

    تظهر الحزم بالشكل `Format: bundle` مع نوع فرعي هو `codex` أو `claude` أو `cursor`.

  </Step>

  <Step title="أعد التشغيل واستخدمها">
    ```bash
    openclaw gateway restart
    ```

    تصبح الميزات المعيّنة (Skills، وhooks، وأدوات MCP، وإعدادات LSP الافتراضية) متاحة في الجلسة التالية.

  </Step>
</Steps>

## ما الذي يعيّنه OpenClaw من الحزم

ليست كل ميزات الحزمة تعمل في OpenClaw اليوم. فيما يلي ما يعمل وما
يُكتشف لكنه لم يُوصل بعد.

### المدعوم حاليًا

| الميزة       | كيفية تعيينها                                                                                 | ينطبق على     |
| ------------- | ------------------------------------------------------------------------------------------- | -------------- |
| محتوى Skills | يتم تحميل جذور Skills الخاصة بالحزمة كجذور Skills عادية في OpenClaw                                           | جميع الصيغ    |
| الأوامر      | يتم التعامل مع `commands/` و`.cursor/commands/` كجذور Skills                                  | Claude, Cursor |
| حزم Hook    | تخطيطات `HOOK.md` + `handler.ts` بأسلوب OpenClaw                                             | Codex          |
| أدوات MCP     | يتم دمج تكوين MCP الخاص بالحزمة في إعدادات Pi المضمنة؛ ويتم تحميل خوادم stdio وHTTP المدعومة | جميع الصيغ    |
| خوادم LSP   | يتم دمج `Claude .lsp.json` و`lspServers` المعلنة في manifest في إعدادات LSP الافتراضية لـ Pi المضمنة  | Claude         |
| الإعدادات      | يتم استيراد `Claude settings.json` كإعدادات Pi مضمنة افتراضية                                     | Claude         |

#### محتوى Skills

- يتم تحميل جذور Skills الخاصة بالحزمة كجذور Skills عادية في OpenClaw
- يتم التعامل مع جذور `commands` الخاصة بـ Claude على أنها جذور Skills إضافية
- يتم التعامل مع جذور `.cursor/commands` الخاصة بـ Cursor على أنها جذور Skills إضافية

وهذا يعني أن ملفات أوامر Claude المكتوبة بـ Markdown تعمل عبر محمّل Skills العادي في OpenClaw.
كما تعمل أوامر Markdown الخاصة بـ Cursor عبر المسار نفسه.

#### حزم Hook

- تعمل جذور hooks الخاصة بالحزمة **فقط** عندما تستخدم تخطيط
  حزم hooks العادي في OpenClaw. واليوم تكون هذه الحالة أساسًا متوافقة مع Codex:
  - `HOOK.md`
  - `handler.ts` أو `handler.js`

#### MCP لـ Pi

- يمكن للحزم المفعّلة أن تساهم في تكوين خادم MCP
- يدمج OpenClaw تكوين MCP الخاص بالحزمة في إعدادات Pi المضمنة الفعلية على شكل
  `mcpServers`
- يكشف OpenClaw أدوات MCP المدعومة الخاصة بالحزمة أثناء أدوار الوكيل Pi المضمنة عن طريق
  تشغيل خوادم stdio أو الاتصال بخوادم HTTP
- تظل إعدادات Pi المحلية الخاصة بالمشروع مطبقة بعد الإعدادات الافتراضية للحزمة، لذا
  يمكن لإعدادات مساحة العمل تجاوز إدخالات MCP الخاصة بالحزمة عند الحاجة
- تُرتب فهارس أدوات MCP الخاصة بالحزمة ترتيبًا حتميًا قبل التسجيل، بحيث
  لا تؤدي تغييرات ترتيب `listTools()` في المصدر إلى إرباك كتل أدوات prompt-cache

##### وسائل النقل

يمكن لخوادم MCP استخدام stdio أو HTTP:

**Stdio** يشغّل عملية فرعية:

```json
{
  "mcp": {
    "servers": {
      "my-server": {
        "command": "node",
        "args": ["server.js"],
        "env": { "PORT": "3000" }
      }
    }
  }
}
```

**HTTP** يتصل بخادم MCP عامل عبر `sse` افتراضيًا، أو `streamable-http` عند الطلب:

```json
{
  "mcp": {
    "servers": {
      "my-server": {
        "url": "http://localhost:3100/mcp",
        "transport": "streamable-http",
        "headers": {
          "Authorization": "Bearer ${MY_SECRET_TOKEN}"
        },
        "connectionTimeoutMs": 30000
      }
    }
  }
}
```

- يمكن ضبط `transport` على `"streamable-http"` أو `"sse"`؛ وعند حذفه يستخدم OpenClaw القيمة `sse`
- لا يُسمح إلا بمخططي URL ‏`http:` و`https:`
- تدعم قيم `headers` الإقحام من نوع `${ENV_VAR}`
- يُرفض إدخال الخادم الذي يحتوي على كل من `command` و`url`
- يتم حجب بيانات اعتماد URL ‏(userinfo ومعلمات الاستعلام) من أوصاف الأدوات
  والسجلات
- يتجاوز `connectionTimeoutMs` المهلة الافتراضية للاتصال البالغة 30 ثانية لكل من
  وسائل نقل stdio وHTTP

##### تسمية الأدوات

يسجل OpenClaw أدوات MCP الخاصة بالحزمة بأسماء آمنة للمزوّدين بالصيغة
`serverName__toolName`. فعلى سبيل المثال، إذا كان الخادم بالمفتاح `"vigil-harbor"` يكشف
أداة `memory_search`، فسيتم تسجيلها باسم `vigil-harbor__memory_search`.

- يتم استبدال الأحرف الخارجة عن `A-Za-z0-9_-` بالرمز `-`
- تُحد بادئات الخوادم عند 30 حرفًا
- تُحد أسماء الأدوات الكاملة عند 64 حرفًا
- تعود أسماء الخوادم الفارغة إلى `mcp`
- يتم فض الاشتباك في الأسماء الموحّدة المتصادمة باستخدام لواحق رقمية
- يكون الترتيب النهائي المكشوف للأدوات حتميًا بحسب الاسم الآمن للحفاظ على ثبات cache في
  أدوار Pi المتكررة

#### إعدادات Pi المضمنة

- يتم استيراد `Claude settings.json` كإعدادات Pi مضمنة افتراضية عندما تكون
  الحزمة مفعّلة
- يقوم OpenClaw بتنقية مفاتيح تجاوز shell قبل تطبيقها

المفاتيح المنقاة:

- `shellPath`
- `shellCommandPrefix`

#### LSP المضمن لـ Pi

- يمكن لحزم Claude المفعّلة أن تساهم في تكوين خادم LSP
- يحمّل OpenClaw الملف `.lsp.json` بالإضافة إلى أي مسارات `lspServers` معلنة في manifest
- يتم دمج تكوين LSP الخاص بالحزمة في إعدادات LSP الافتراضية الفعلية لـ Pi المضمنة
- لا يمكن تشغيل سوى خوادم LSP المدعومة المعتمدة على stdio اليوم؛ أما وسائل النقل
  غير المدعومة فما زالت تظهر في `openclaw plugins inspect <id>`

### ما يتم اكتشافه لكن لا يتم تنفيذه

يتم التعرف على هذه العناصر وإظهارها في التشخيصات، لكن OpenClaw لا يشغّلها:

- `agents` و`hooks.json` automation و`outputStyles` الخاصة بـ Claude
- `.cursor/agents` و`.cursor/hooks.json` و`.cursor/rules` الخاصة بـ Cursor
- بيانات metadata الخاصة بـ Codex inline/app خارج تقارير الإمكانات

## صيغ الحزم

<AccordionGroup>
  <Accordion title="حزم Codex">
    العلامات: `.codex-plugin/plugin.json`

    المحتوى الاختياري: `skills/`، و`hooks/`، و`.mcp.json`، و`.app.json`

    تتوافق حزم Codex مع OpenClaw بشكل أفضل عندما تستخدم جذور Skills و
    أدلة حزم hooks بأسلوب OpenClaw (`HOOK.md` + `handler.ts`).

  </Accordion>

  <Accordion title="حزم Claude">
    وضعا اكتشاف:

    - **قائم على Manifest:** ‏`.claude-plugin/plugin.json`
    - **بلا Manifest:** تخطيط Claude الافتراضي (`skills/`، و`commands/`، و`agents/`، و`hooks/`، و`.mcp.json`، و`.lsp.json`، و`settings.json`)

    السلوك الخاص بـ Claude:

    - يتم التعامل مع `commands/` كمحتوى Skills
    - يتم استيراد `settings.json` إلى إعدادات Pi المضمنة (مع تنقية مفاتيح تجاوز shell)
    - يكشف `.mcp.json` أدوات stdio المدعومة إلى Pi المضمنة
    - يتم تحميل `.lsp.json` بالإضافة إلى مسارات `lspServers` المعلنة في manifest إلى إعدادات LSP الافتراضية لـ Pi المضمنة
    - يتم اكتشاف `hooks/hooks.json` لكن لا يتم تنفيذه
    - تكون مسارات المكونات المخصصة في manifest إضافية (فهي توسّع الإعدادات الافتراضية ولا تستبدلها)

  </Accordion>

  <Accordion title="حزم Cursor">
    العلامات: `.cursor-plugin/plugin.json`

    المحتوى الاختياري: `skills/`، و`.cursor/commands/`، و`.cursor/agents/`، و`.cursor/rules/`، و`.cursor/hooks.json`، و`.mcp.json`

    - يتم التعامل مع `.cursor/commands/` كمحتوى Skills
    - تكون `.cursor/rules/` و`.cursor/agents/` و`.cursor/hooks.json` للاكتشاف فقط

  </Accordion>
</AccordionGroup>

## أولوية الاكتشاف

يتحقق OpenClaw أولًا من صيغة plugin الأصلية:

1. `openclaw.plugin.json` أو `package.json` صالح يحتوي على `openclaw.extensions` — ويُعامل كـ **plugin أصلية**
2. علامات الحزم (`.codex-plugin/`، أو `.claude-plugin/`، أو التخطيط الافتراضي لـ Claude/Cursor) — وتُعامل كـ **حزمة**

إذا احتوى الدليل على الاثنين معًا، يستخدم OpenClaw المسار الأصلي. وهذا يمنع
تثبيت الحزم ثنائية الصيغة جزئيًا كحزم.

## الأمان

للحزم حد ثقة أضيق من plugins الأصلية:

- لا يقوم OpenClaw **بتحميل** وحدات runtime العشوائية الخاصة بالحزم داخل العملية
- يجب أن تبقى مسارات Skills وحزم hooks داخل جذر plugin (مع التحقق من الحدود)
- تتم قراءة ملفات الإعدادات باستخدام فحوصات الحدود نفسها
- يمكن تشغيل خوادم MCP المدعومة من نوع stdio كعمليات فرعية

وهذا يجعل الحزم أكثر أمانًا افتراضيًا، لكن يجب عليك مع ذلك التعامل مع الحزم الخارجية
على أنها محتوى موثوق بالنسبة إلى الميزات التي تكشفها.

## استكشاف الأخطاء وإصلاحها

<AccordionGroup>
  <Accordion title="يتم اكتشاف الحزمة لكن الإمكانات لا تعمل">
    شغّل `openclaw plugins inspect <id>`. فإذا كانت الإمكانية مدرجة لكن معلمة على أنها
    غير موصولة، فهذا حد من حدود المنتج — وليس تثبيتًا معطلاً.
  </Accordion>

  <Accordion title="ملفات أوامر Claude لا تظهر">
    تأكد من أن الحزمة مفعّلة وأن ملفات Markdown موجودة داخل جذر
    `commands/` أو `skills/` المكتشف.
  </Accordion>

  <Accordion title="إعدادات Claude لا تُطبَّق">
    لا يتم دعم سوى إعدادات Pi المضمنة من `settings.json`. ولا يتعامل OpenClaw
    مع إعدادات الحزمة على أنها تصحيحات raw config.
  </Accordion>

  <Accordion title="Hooks الخاصة بـ Claude لا تُنفَّذ">
    يكون `hooks/hooks.json` للاكتشاف فقط. وإذا كنت تحتاج إلى hooks قابلة للتشغيل، فاستخدم
    تخطيط حزم hooks في OpenClaw أو اشحن plugin أصلية.
  </Accordion>
</AccordionGroup>

## ذو صلة

- [تثبيت plugins وتكوينها](/tools/plugin)
- [بناء plugins](/plugins/building-plugins) — أنشئ plugin أصلية
- [Plugin Manifest](/plugins/manifest) — مخطط manifest الأصلي
