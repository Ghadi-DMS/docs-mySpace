---
read_when:
    - تثبيت الإضافات أو تهيئتها
    - فهم قواعد اكتشاف الإضافات وتحميلها
    - العمل مع حِزم الإضافات المتوافقة مع Codex/Claude
sidebarTitle: Install and Configure
summary: تثبيت إضافات OpenClaw وتهيئتها وإدارتها
title: Plugins
x-i18n:
    generated_at: "2026-04-06T03:14:30Z"
    model: gpt-5.4
    provider: openai
    source_hash: 9e2472a3023f3c1c6ee05b0cdc228f6b713cc226a08695b327de8a3ad6973c83
    source_path: tools/plugin.md
    workflow: 15
---

# Plugins

توسّع Plugins إمكانات OpenClaw بإضافة قدرات جديدة: القنوات، ومزوّدي النماذج،
والأدوات، وSkills، والكلام، والنسخ الحي الفوري، والصوت الحي الفوري،
وفهم الوسائط، وتوليد الصور، وتوليد الفيديو، وweb fetch، والبحث على الويب،
وغير ذلك. بعض Plugins تكون **أساسية** (تأتي مع OpenClaw)، وأخرى
**خارجية** (ينشرها المجتمع على npm).

## بداية سريعة

<Steps>
  <Step title="اطّلع على ما هو محمّل">
    ```bash
    openclaw plugins list
    ```
  </Step>

  <Step title="ثبّت إضافة">
    ```bash
    # من npm
    openclaw plugins install @openclaw/voice-call

    # من دليل محلي أو أرشيف
    openclaw plugins install ./my-plugin
    openclaw plugins install ./my-plugin.tgz
    ```

  </Step>

  <Step title="أعد تشغيل Gateway">
    ```bash
    openclaw gateway restart
    ```

    ثم هيّئ تحت `plugins.entries.\<id\>.config` في ملف config الخاص بك.

  </Step>
</Steps>

إذا كنت تفضّل التحكم الأصلي داخل الدردشة، فعّل `commands.plugins: true` واستخدم:

```text
/plugin install clawhub:@openclaw/voice-call
/plugin show voice-call
/plugin enable voice-call
```

يستخدم مسار التثبيت نفس محلّل CLI: مسار/أرشيف محلي، أو
`clawhub:<pkg>` صريح، أو مواصفة حزمة عارية (ClawHub أولًا، ثم fallback إلى npm).

إذا كان config غير صالح، يفشل التثبيت عادةً بشكل مغلق ويوجهك إلى
`openclaw doctor --fix`. والاستثناء الوحيد للتعافي هو مسار ضيق لإعادة تثبيت
الإضافات المجمعة بالنسبة إلى الإضافات التي تشترك في
`openclaw.install.allowInvalidConfigRecovery`.

## أنواع Plugins

يتعرف OpenClaw على تنسيقين للإضافات:

| التنسيق   | كيف يعمل                                                        | أمثلة                                                  |
| --------- | ---------------------------------------------------------------- | ------------------------------------------------------ |
| **أصلي**  | `openclaw.plugin.json` + وحدة وقت تشغيل؛ يُنفَّذ داخل العملية    | الإضافات الرسمية، وحزم npm المجتمعية                  |
| **حزمة**  | تخطيط متوافق مع Codex/Claude/Cursor؛ يُربط بميزات OpenClaw      | `.codex-plugin/` و`.claude-plugin/` و`.cursor-plugin/` |

يظهر كلاهما تحت `openclaw plugins list`. راجع [حِزم Plugins](/ar/plugins/bundles) لمعرفة تفاصيل الحِزم.

إذا كنت تكتب إضافة أصلية، فابدأ من [بناء Plugins](/ar/plugins/building-plugins)
و[نظرة عامة على Plugin SDK](/ar/plugins/sdk-overview).

## Plugins الرسمية

### قابلة للتثبيت (npm)

| Plugin          | الحزمة                 | المستندات                            |
| --------------- | ---------------------- | ------------------------------------ |
| Matrix          | `@openclaw/matrix`     | [Matrix](/ar/channels/matrix)           |
| Microsoft Teams | `@openclaw/msteams`    | [Microsoft Teams](/ar/channels/msteams) |
| Nostr           | `@openclaw/nostr`      | [Nostr](/ar/channels/nostr)             |
| Voice Call      | `@openclaw/voice-call` | [Voice Call](/ar/plugins/voice-call)    |
| Zalo            | `@openclaw/zalo`       | [Zalo](/ar/channels/zalo)               |
| Zalo Personal   | `@openclaw/zalouser`   | [Zalo Personal](/ar/plugins/zalouser)   |

### أساسية (مرفقة مع OpenClaw)

<AccordionGroup>
  <Accordion title="مزوّدو النماذج (مفعّلون افتراضيًا)">
    `anthropic` و`byteplus` و`cloudflare-ai-gateway` و`github-copilot` و`google`،
    و`huggingface` و`kilocode` و`kimi-coding` و`minimax` و`mistral` و`qwen`،
    و`moonshot` و`nvidia` و`openai` و`opencode` و`opencode-go` و`openrouter`،
    و`qianfan` و`synthetic` و`together` و`venice`،
    و`vercel-ai-gateway` و`volcengine` و`xiaomi` و`zai`
  </Accordion>

  <Accordion title="إضافات الذاكرة">
    - `memory-core` — بحث الذاكرة المجمّع (الافتراضي عبر `plugins.slots.memory`)
    - `memory-lancedb` — ذاكرة طويلة الأمد تُثبَّت عند الطلب مع استدعاء/التقاط تلقائي (اضبط `plugins.slots.memory = "memory-lancedb"`)
  </Accordion>

  <Accordion title="مزوّدو الكلام (مفعّلون افتراضيًا)">
    `elevenlabs`، `microsoft`
  </Accordion>

  <Accordion title="أخرى">
    - `browser` — إضافة المتصفح المجمّعة لأداة المتصفح، وCLI ‏`openclaw browser`، وطريقة البوابة `browser.request`، ووقت تشغيل المتصفح، وخدمة التحكم الافتراضية للمتصفح (مفعّلة افتراضيًا؛ عطّلها قبل استبدالها)
    - `copilot-proxy` — جسر VS Code Copilot Proxy (معطّل افتراضيًا)
  </Accordion>
</AccordionGroup>

هل تبحث عن Plugins من جهات خارجية؟ راجع [Plugins المجتمع](/ar/plugins/community).

## الإعداد

```json5
{
  plugins: {
    enabled: true,
    allow: ["voice-call"],
    deny: ["untrusted-plugin"],
    load: { paths: ["~/Projects/oss/voice-call-extension"] },
    entries: {
      "voice-call": { enabled: true, config: { provider: "twilio" } },
    },
  },
}
```

| الحقل            | الوصف                                                  |
| ---------------- | ------------------------------------------------------ |
| `enabled`        | مفتاح تشغيل عام (الافتراضي: `true`)                   |
| `allow`          | قائمة سماح للإضافات (اختيارية)                         |
| `deny`           | قائمة منع للإضافات (اختيارية؛ والمنع يتفوق)           |
| `load.paths`     | ملفات/أدلة إضافات إضافية                               |
| `slots`          | محددات خانات حصرية (مثل `memory` و`contextEngine`)     |
| `entries.\<id\>` | مفاتيح تشغيل + config لكل إضافة                        |

تتطلب تغييرات config **إعادة تشغيل للبوابة**. وإذا كانت Gateway تعمل مع
مراقبة config + إعادة تشغيل داخل العملية مفعّلة (وهو المسار الافتراضي لـ `openclaw gateway`)،
فعادةً ما تُنفّذ إعادة التشغيل تلك تلقائيًا بعد لحظة من وصول كتابة config.

<Accordion title="حالات Plugins: معطلة مقابل مفقودة مقابل غير صالحة">
  - **معطلة**: الإضافة موجودة لكن قواعد التمكين عطّلتها. ويُحافَظ على config.
  - **مفقودة**: يشير config إلى معرّف إضافة لم يجده الاكتشاف.
  - **غير صالحة**: الإضافة موجودة لكن config الخاص بها لا يطابق schema المعلن.
</Accordion>

## الاكتشاف والأولوية

يفحص OpenClaw الإضافات بهذا الترتيب (وأول تطابق يفوز):

<Steps>
  <Step title="مسارات config">
    `plugins.load.paths` — مسارات صريحة إلى ملفات أو أدلة.
  </Step>

  <Step title="إضافات مساحة العمل">
    `\<workspace\>/.openclaw/<plugin-root>/*.ts` و `\<workspace\>/.openclaw/<plugin-root>/*/index.ts`.
  </Step>

  <Step title="إضافات عامة">
    `~/.openclaw/<plugin-root>/*.ts` و `~/.openclaw/<plugin-root>/*/index.ts`.
  </Step>

  <Step title="الإضافات المجمعة">
    مرفقة مع OpenClaw. كثير منها مفعّل افتراضيًا (مزوّدو النماذج، والكلام).
    وأخرى تتطلب تمكينًا صريحًا.
  </Step>
</Steps>

### قواعد التمكين

- `plugins.enabled: false` يعطّل جميع Plugins
- يفوز `plugins.deny` دائمًا على allow
- `plugins.entries.\<id\>.enabled: false` يعطّل تلك الإضافة
- إضافات مصدر مساحة العمل تكون **معطلة افتراضيًا** (ويجب تمكينها صراحةً)
- تتبع الإضافات المجمعة مجموعة التفعيل الافتراضية المضمّنة ما لم يُتجاوز ذلك
- يمكن للخانات الحصرية فرض تفعيل الإضافة المختارة لذلك slot

## خانات Plugins (فئات حصرية)

بعض الفئات حصرية (يمكن تنشيط واحدة فقط في كل مرة):

```json5
{
  plugins: {
    slots: {
      memory: "memory-core", // أو "none" للتعطيل
      contextEngine: "legacy", // أو معرّف إضافة
    },
  },
}
```

| الخانة          | ما الذي تتحكم فيه      | الافتراضي           |
| --------------- | ---------------------- | ------------------- |
| `memory`        | إضافة الذاكرة النشطة   | `memory-core`       |
| `contextEngine` | محرك السياق النشط      | `legacy` (مضمّن)    |

## مرجع CLI

```bash
openclaw plugins list                       # جرد موجز
openclaw plugins list --enabled            # الإضافات المحمّلة فقط
openclaw plugins list --verbose            # أسطر تفاصيل لكل إضافة
openclaw plugins list --json               # جرد قابل للقراءة آليًا
openclaw plugins inspect <id>              # تفاصيل متعمقة
openclaw plugins inspect <id> --json       # قابل للقراءة آليًا
openclaw plugins inspect --all             # جدول على مستوى الأسطول
openclaw plugins info <id>                 # اسم بديل لـ inspect
openclaw plugins doctor                    # تشخيصات

openclaw plugins install <package>         # تثبيت (ClawHub أولًا، ثم npm)
openclaw plugins install clawhub:<pkg>     # تثبيت من ClawHub فقط
openclaw plugins install <spec> --force    # الكتابة فوق تثبيت موجود
openclaw plugins install <path>            # تثبيت من مسار محلي
openclaw plugins install -l <path>         # ربط (من دون نسخ) للتطوير
openclaw plugins install <plugin> --marketplace <source>
openclaw plugins install <plugin> --marketplace https://github.com/<owner>/<repo>
openclaw plugins install <spec> --pin      # تسجيل مواصفة npm الدقيقة التي جرى حلها
openclaw plugins install <spec> --dangerously-force-unsafe-install
openclaw plugins update <id>             # تحديث إضافة واحدة
openclaw plugins update <id> --dangerously-force-unsafe-install
openclaw plugins update --all            # تحديث الجميع
openclaw plugins uninstall <id>          # إزالة سجلات config/التثبيت
openclaw plugins uninstall <id> --keep-files
openclaw plugins marketplace list <source>
openclaw plugins marketplace list <source> --json

openclaw plugins enable <id>
openclaw plugins disable <id>
```

تأتي الإضافات المجمعة مع OpenClaw. وكثير منها مفعّل افتراضيًا (مثل
مزوّدي النماذج المجمّعين، ومزوّدي الكلام المجمّعين، وإضافة المتصفح
المجمعة). أما الإضافات المجمعة الأخرى فلا تزال تحتاج إلى `openclaw plugins enable <id>`.

يقوم `--force` بالكتابة فوق إضافة أو حزمة hook مثبتة موجودة في مكانها.
وهو غير مدعوم مع `--link`، الذي يعيد استخدام المسار المصدر بدل
نسخه فوق هدف تثبيت مُدار.

يخص `--pin` npm فقط. وهو غير مدعوم مع `--marketplace`، لأن
تثبيتات marketplace تحفظ بيانات مصدر marketplace الوصفية بدل مواصفة npm.

يمثل `--dangerously-force-unsafe-install` تجاوزًا للطوارئ لحالات الإيجابيات
الكاذبة من الماسح المضمّن للشيفرة الخطرة. فهو يسمح لتثبيتات Plugins وتحديثاتها
بالمتابعة بعد نتائج `critical` المضمّنة، لكنه مع ذلك
لا يتجاوز حظر سياسة `before_install` الخاصة بالإضافة أو حظر فشل الفحص.

تنطبق راية CLI هذه على تدفقات تثبيت/تحديث Plugins فقط. أما تثبيتات اعتماديات Skills
المدعومة من Gateway فتستخدم تجاوز الطلب المطابق `dangerouslyForceUnsafeInstall` بدلًا من ذلك، بينما يبقى `openclaw skills install` تدفق تنزيل/تثبيت Skills المنفصل من ClawHub.

تشارك الحِزم المتوافقة في التدفق نفسه لـ list/inspect/enable/disable الخاص بالإضافات. ويتضمن دعم وقت التشغيل الحالي Skills الخاصة بالحزم، وClaude command-skills،
وقيم `settings.json` الافتراضية الخاصة بـ Claude، وقيم `.lsp.json` و
`lspServers` المعلنة في manifest الخاصة بـ Claude، وCursor command-skills،
وأدلة hook المتوافقة مع Codex.

كما يبلّغ `openclaw plugins inspect <id>` أيضًا عن إمكانات الحزمة المكتشفة، بالإضافة إلى إدخالات MCP وLSP server المدعومة أو غير المدعومة للإضافات المدعومة بالحِزم.

يمكن أن تكون مصادر marketplace اسم marketplace معروف لـ Claude من
`~/.claude/plugins/known_marketplaces.json`، أو جذر marketplace محليًا، أو
مسار `marketplace.json`، أو صيغة GitHub مختصرة مثل `owner/repo`، أو URL لمستودع GitHub، أو URL لـ git. وبالنسبة إلى marketplaces البعيدة، يجب أن تبقى إدخالات الإضافة داخل
مستودع marketplace المستنسخ وأن تستخدم مصادر مسارات نسبية فقط.

راجع [مرجع CLI ‏`openclaw plugins`](/cli/plugins) للاطلاع على التفاصيل الكاملة.

## نظرة عامة على Plugin API

تُصدِّر Plugins الأصلية كائن إدخال يكشف `register(api)`. وقد لا تزال
الإضافات الأقدم تستخدم `activate(api)` كاسم بديل قديم، لكن ينبغي للإضافات الجديدة
أن تستخدم `register`.

```typescript
export default definePluginEntry({
  id: "my-plugin",
  name: "My Plugin",
  register(api) {
    api.registerProvider({
      /* ... */
    });
    api.registerTool({
      /* ... */
    });
    api.registerChannel({
      /* ... */
    });
  },
});
```

يحمّل OpenClaw كائن الإدخال ويستدعي `register(api)` أثناء تفعيل الإضافة.
ولا يزال المحمّل يعود إلى `activate(api)` بالنسبة إلى الإضافات الأقدم،
لكن ينبغي للإضافات المجمعة والإضافات الخارجية الجديدة أن تتعامل مع `register`
باعتباره العقد العام.

طرق التسجيل الشائعة:

| الطريقة                                 | ما الذي تسجله             |
| -------------------------------------- | ------------------------- |
| `registerProvider`                     | مزوّد نموذج (LLM)         |
| `registerChannel`                      | قناة دردشة                |
| `registerTool`                         | أداة وكيل                 |
| `registerHook` / `on(...)`             | hooks دورة الحياة         |
| `registerSpeechProvider`               | تحويل النص إلى كلام / STT |
| `registerRealtimeTranscriptionProvider`| STT بالبث                 |
| `registerRealtimeVoiceProvider`        | صوت حي فوري مزدوج         |
| `registerMediaUnderstandingProvider`   | تحليل الصور/الصوت         |
| `registerImageGenerationProvider`      | توليد الصور               |
| `registerMusicGenerationProvider`      | توليد الموسيقى            |
| `registerVideoGenerationProvider`      | توليد الفيديو             |
| `registerWebFetchProvider`             | مزوّد web fetch / scrape  |
| `registerWebSearchProvider`            | بحث على الويب             |
| `registerHttpRoute`                    | نقطة نهاية HTTP           |
| `registerCommand` / `registerCli`      | أوامر CLI                 |
| `registerContextEngine`                | محرك السياق               |
| `registerService`                      | خدمة خلفية                |

سلوك حارس الـ hooks الخاصة بدورة الحياة المTyped:

- `before_tool_call`: القيمة `{ block: true }` نهائية؛ وتُتخطى المعالجات ذات الأولوية الأدنى.
- `before_tool_call`: القيمة `{ block: false }` لا تؤدي إلى شيء ولا تمحو حظرًا سابقًا.
- `before_install`: القيمة `{ block: true }` نهائية؛ وتُتخطى المعالجات ذات الأولوية الأدنى.
- `before_install`: القيمة `{ block: false }` لا تؤدي إلى شيء ولا تمحو حظرًا سابقًا.
- `message_sending`: القيمة `{ cancel: true }` نهائية؛ وتُتخطى المعالجات ذات الأولوية الأدنى.
- `message_sending`: القيمة `{ cancel: false }` لا تؤدي إلى شيء ولا تمحو إلغاءً سابقًا.

للاطلاع على السلوك الكامل المTyped للـ hooks، راجع [نظرة عامة على SDK](/ar/plugins/sdk-overview#hook-decision-semantics).

## ذو صلة

- [بناء Plugins](/ar/plugins/building-plugins) — أنشئ الإضافة الخاصة بك
- [حِزم Plugins](/ar/plugins/bundles) — التوافق مع حِزم Codex/Claude/Cursor
- [Plugin Manifest](/ar/plugins/manifest) — مخطط manifest
- [تسجيل الأدوات](/ar/plugins/building-plugins#registering-agent-tools) — أضف أدوات وكيل داخل إضافة
- [الداخليات الخاصة بـ Plugins](/ar/plugins/architecture) — نموذج الإمكانات ومسار التحميل
- [Plugins المجتمع](/ar/plugins/community) — قوائم الجهات الخارجية
