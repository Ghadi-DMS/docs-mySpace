---
read_when:
    - تثبيت plugins أو تهيئتها
    - فهم اكتشاف plugins وقواعد تحميلها
    - العمل مع حزم plugins المتوافقة مع Codex/Claude
sidebarTitle: Install and Configure
summary: تثبيت Plugins في OpenClaw وتهيئتها وإدارتها
title: Plugins
x-i18n:
    generated_at: "2026-04-05T12:59:59Z"
    model: gpt-5.4
    provider: openai
    source_hash: 707bd3625596f290322aeac9fecb7f4c6f45d595fdfb82ded7cbc8e04457ac7f
    source_path: tools/plugin.md
    workflow: 15
---

# Plugins

توسّع Plugins إمكانات OpenClaw بقدرات جديدة: القنوات، وموفري النماذج،
والأدوات، وSkills، والكلام، والنسخ الفوري، والصوت الفوري،
وفهم الوسائط، وتوليد الصور، وتوليد الفيديو، وجلب الويب، والبحث على الويب،
وغير ذلك. بعض plugins تكون **أساسية** (تأتي مع OpenClaw)، وأخرى
**خارجية** (ينشرها المجتمع على npm).

## بدء سريع

<Steps>
  <Step title="اطلع على ما تم تحميله">
    ```bash
    openclaw plugins list
    ```
  </Step>

  <Step title="ثبّت plugin">
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

    ثم قم بالتهيئة ضمن `plugins.entries.\<id\>.config` في ملف التهيئة الخاص بك.

  </Step>
</Steps>

إذا كنت تفضّل التحكم مباشرة من الدردشة، فعِّل `commands.plugins: true` واستخدم:

```text
/plugin install clawhub:@openclaw/voice-call
/plugin show voice-call
/plugin enable voice-call
```

يستخدم مسار التثبيت نفس محلّل CLI: مسار/أرشيف محلي، أو
`clawhub:<pkg>` صريح، أو مواصفة حزمة مجردة (ClawHub أولًا، ثم fallback إلى npm).

إذا كانت التهيئة غير صالحة، يفشل التثبيت عادةً بشكل مغلق ويوجهك إلى
`openclaw doctor --fix`. والاستثناء الوحيد للاسترداد هو مسار ضيق لإعادة تثبيت plugin مضمّن
لـ plugins التي تختار الاشتراك في
`openclaw.install.allowInvalidConfigRecovery`.

## أنواع plugins

يتعرف OpenClaw على تنسيقين من plugins:

| التنسيق | كيف يعمل | أمثلة |
| ---------- | ------------------------------------------------------------------ | ------------------------------------------------------ |
| **Native** | `openclaw.plugin.json` + وحدة runtime؛ يُنفّذ داخل العملية | plugins الرسمية، وحزم npm المجتمعية |
| **Bundle** | تخطيط متوافق مع Codex/Claude/Cursor؛ يُربط بميزات OpenClaw | `.codex-plugin/`، `.claude-plugin/`، `.cursor-plugin/` |

يظهر كلاهما ضمن `openclaw plugins list`. راجع [Plugin Bundles](/ar/plugins/bundles) لمعرفة تفاصيل الحزم.

إذا كنت تكتب plugin native، فابدأ من [بناء Plugins](/ar/plugins/building-plugins)
و[نظرة عامة على Plugin SDK](/ar/plugins/sdk-overview).

## plugins الرسمية

### قابلة للتثبيت (npm)

| Plugin | الحزمة | الوثائق |
| --------------- | ---------------------- | ------------------------------------ |
| Matrix | `@openclaw/matrix` | [Matrix](/ar/channels/matrix) |
| Microsoft Teams | `@openclaw/msteams` | [Microsoft Teams](/ar/channels/msteams) |
| Nostr | `@openclaw/nostr` | [Nostr](/ar/channels/nostr) |
| Voice Call | `@openclaw/voice-call` | [Voice Call](/ar/plugins/voice-call) |
| Zalo | `@openclaw/zalo` | [Zalo](/ar/channels/zalo) |
| Zalo Personal | `@openclaw/zalouser` | [Zalo Personal](/ar/plugins/zalouser) |

### أساسية (تأتي مع OpenClaw)

<AccordionGroup>
  <Accordion title="موفرو النماذج (مفعّلة افتراضيًا)">
    `anthropic` و`byteplus` و`cloudflare-ai-gateway` و`github-copilot` و`google`،
    و`huggingface` و`kilocode` و`kimi-coding` و`minimax` و`mistral` و`qwen`،
    و`moonshot` و`nvidia` و`openai` و`opencode` و`opencode-go` و`openrouter`،
    و`qianfan` و`synthetic` و`together` و`venice`،
    و`vercel-ai-gateway` و`volcengine` و`xiaomi` و`zai`
  </Accordion>

  <Accordion title="Plugins الذاكرة">
    - `memory-core` — بحث الذاكرة المضمّن (الافتراضي عبر `plugins.slots.memory`)
    - `memory-lancedb` — ذاكرة طويلة المدى تُثبَّت عند الطلب مع auto-recall/capture (عيّن `plugins.slots.memory = "memory-lancedb"`)
  </Accordion>

  <Accordion title="موفرو الكلام (مفعّلة افتراضيًا)">
    `elevenlabs` و`microsoft`
  </Accordion>

  <Accordion title="أخرى">
    - `browser` — plugin متصفح مضمّنة لأداة المتصفح، وCLI ‏`openclaw browser`، وطريقة gateway ‏`browser.request`، وruntime المتصفح، وخدمة التحكم الافتراضية في المتصفح (مفعّلة افتراضيًا؛ عطّلها قبل استبدالها)
    - `copilot-proxy` — جسر VS Code Copilot Proxy ‏(معطّل افتراضيًا)
  </Accordion>
</AccordionGroup>

هل تبحث عن plugins من جهات خارجية؟ راجع [Community Plugins](/ar/plugins/community).

## التهيئة

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

| الحقل | الوصف |
| ---------------- | --------------------------------------------------------- |
| `enabled` | مفتاح تشغيل رئيسي (الافتراضي: `true`) |
| `allow` | allowlist لـ plugins (اختياري) |
| `deny` | denylist لـ plugins (اختياري؛ deny له الأولوية) |
| `load.paths` | ملفات/أدلة plugins إضافية |
| `slots` | محددات خانات حصرية (مثل `memory` و`contextEngine`) |
| `entries.\<id\>` | مفاتيح التشغيل + التهيئة لكل plugin |

تتطلب تغييرات التهيئة **إعادة تشغيل gateway**. إذا كان Gateway يعمل مع
مراقبة التهيئة + إعادة تشغيل داخل العملية مفعّلة (وهو المسار الافتراضي لـ `openclaw gateway`)،
فعادةً ما يتم تنفيذ إعادة التشغيل هذه تلقائيًا بعد لحظات من وصول كتابة التهيئة.

<Accordion title="حالات plugin: معطّلة مقابل مفقودة مقابل غير صالحة">
  - **معطّلة**: plugin موجودة لكن قواعد التمكين عطّلتها. يتم الاحتفاظ بالتهيئة.
  - **مفقودة**: تشير التهيئة إلى معرّف plugin لم يعثر عليه الاكتشاف.
  - **غير صالحة**: plugin موجودة لكن تهيئتها لا تطابق المخطط المعلن.
</Accordion>

## الاكتشاف والأولوية

يفحص OpenClaw plugins بهذا الترتيب (أول تطابق يفوز):

<Steps>
  <Step title="مسارات التهيئة">
    `plugins.load.paths` — مسارات ملفات أو أدلة صريحة.
  </Step>

  <Step title="امتدادات مساحة العمل">
    `\<workspace\>/.openclaw/<plugin-root>/*.ts` و `\<workspace\>/.openclaw/<plugin-root>/*/index.ts`.
  </Step>

  <Step title="الامتدادات العامة">
    `~/.openclaw/<plugin-root>/*.ts` و `~/.openclaw/<plugin-root>/*/index.ts`.
  </Step>

  <Step title="Plugins المضمّنة">
    تأتي مع OpenClaw. كثير منها مفعّل افتراضيًا (موفرو النماذج، والكلام).
    ويتطلب بعضها الآخر تمكينًا صريحًا.
  </Step>
</Steps>

### قواعد التمكين

- يؤدي `plugins.enabled: false` إلى تعطيل جميع plugins
- يتغلب `plugins.deny` دائمًا على allow
- يؤدي `plugins.entries.\<id\>.enabled: false` إلى تعطيل تلك plugin
- plugins ذات المصدر من مساحة العمل تكون **معطّلة افتراضيًا** (ويجب تمكينها صراحةً)
- تتبع plugins المضمّنة مجموعة التفعيل الافتراضية المدمجة ما لم يتم تجاوزها
- يمكن للخانات الحصرية فرض تمكين plugin المحددة لتلك الخانة

## خانات plugin (فئات حصرية)

بعض الفئات حصرية (واحدة فقط تكون نشطة في كل مرة):

```json5
{
  plugins: {
    slots: {
      memory: "memory-core", // أو "none" للتعطيل
      contextEngine: "legacy", // أو معرّف plugin
    },
  },
}
```

| الخانة | ما الذي تتحكم فيه | الافتراضي |
| --------------- | --------------------- | ------------------- |
| `memory` | plugin الذاكرة النشطة | `memory-core` |
| `contextEngine` | محرك السياق النشط | `legacy` (مضمّن) |

## مرجع CLI

```bash
openclaw plugins list                       # جرد مضغوط
openclaw plugins list --enabled            # plugins المحمّلة فقط
openclaw plugins list --verbose            # أسطر تفاصيل لكل plugin
openclaw plugins list --json               # جرد قابل للقراءة الآلية
openclaw plugins inspect <id>              # تفاصيل عميقة
openclaw plugins inspect <id> --json       # قابل للقراءة الآلية
openclaw plugins inspect --all             # جدول على مستوى المجموعة
openclaw plugins info <id>                 # اسم بديل لـ inspect
openclaw plugins doctor                    # تشخيصات

openclaw plugins install <package>         # تثبيت (ClawHub أولًا، ثم npm)
openclaw plugins install clawhub:<pkg>     # تثبيت من ClawHub فقط
openclaw plugins install <spec> --force    # الكتابة فوق تثبيت موجود
openclaw plugins install <path>            # تثبيت من مسار محلي
openclaw plugins install -l <path>         # ربط (من دون نسخ) للتطوير
openclaw plugins install <plugin> --marketplace <source>
openclaw plugins install <plugin> --marketplace https://github.com/<owner>/<repo>
openclaw plugins install <spec> --pin      # تسجيل مواصفة npm الدقيقة المحلولة
openclaw plugins install <spec> --dangerously-force-unsafe-install
openclaw plugins update <id>             # تحديث plugin واحدة
openclaw plugins update <id> --dangerously-force-unsafe-install
openclaw plugins update --all            # تحديث الجميع
openclaw plugins uninstall <id>          # إزالة سجلات التهيئة/التثبيت
openclaw plugins uninstall <id> --keep-files
openclaw plugins marketplace list <source>
openclaw plugins marketplace list <source> --json

openclaw plugins enable <id>
openclaw plugins disable <id>
```

تأتي plugins المضمّنة مع OpenClaw. وكثير منها مفعّل افتراضيًا (مثل
موفري النماذج المضمّنين، وموفري الكلام المضمّنين، وplugin المتصفح
المضمّنة). بينما لا تزال plugins المضمّنة الأخرى تتطلب `openclaw plugins enable <id>`.

يقوم `--force` بالكتابة فوق plugin مثبتة موجودة أو حزمة hooks موجودة في مكانها.
وهو غير مدعوم مع `--link`، الذي يعيد استخدام مسار المصدر بدلًا من
النسخ فوق هدف تثبيت مُدار.

`--pin` خاص بـ npm فقط. وهو غير مدعوم مع `--marketplace`، لأن
تثبيتات marketplace تحفظ بيانات مصدر marketplace الوصفية بدلًا من مواصفة npm.

يُعد `--dangerously-force-unsafe-install` تجاوزًا طارئًا لكسر الزجاج من أجل النتائج الإيجابية الكاذبة
من الماسح المدمج للكود الخطير. فهو يسمح باستمرار تثبيتات plugins
وتحديثاتها بعد نتائج `critical` المدمجة، لكنه
لا يتجاوز مع ذلك كتل سياسة plugin ‏`before_install` أو حظر فشل الفحص.

تنطبق راية CLI هذه على تدفقات تثبيت/تحديث plugins فقط. أما تثبيتات تبعيات skill
المعتمدة على Gateway فتستخدم بدلًا من ذلك تجاوز الطلب المطابق `dangerouslyForceUnsafeInstall`، بينما يظل `openclaw skills install` تدفق تنزيل/تثبيت Skills منفصلًا عبر ClawHub.

تشارك الحزم المتوافقة في نفس تدفق list/inspect/enable/disable الخاص بـ plugins.
يشمل دعم runtime الحالي Skills الخاصة بالحزم، وClaude command-skills،
وإعدادات Claude الافتراضية في `settings.json`، والإعدادات الافتراضية لـ Claude في `.lsp.json` وخوادم `lspServers` المعلنة في manifest،
وCursor command-skills، وأدلة hooks المتوافقة مع Codex.

يعرض `openclaw plugins inspect <id>` أيضًا القدرات المكتشفة الخاصة بالحزمة، إضافةً إلى
إدخالات MCP وLSP server المدعومة أو غير المدعومة للـ plugins المعتمدة على الحزم.

يمكن أن تكون مصادر Marketplace اسم marketplace معروفًا لدى Claude من
`~/.claude/plugins/known_marketplaces.json`، أو جذر marketplace محليًا أو
مسار `marketplace.json`، أو اختصار GitHub مثل `owner/repo`، أو عنوان URL لمستودع GitHub، أو عنوان URL لـ git. بالنسبة إلى marketplaces البعيدة، يجب أن تبقى إدخالات plugins داخل
مستودع marketplace المستنسخ وأن تستخدم مصادر مسارات نسبية فقط.

راجع [مرجع CLI ‏`openclaw plugins`](/cli/plugins) للحصول على التفاصيل الكاملة.

## نظرة عامة على Plugin API

تصدّر plugins native كائن entry يكشف `register(api)`. وقد لا تزال plugins
الأقدم تستخدم `activate(api)` كاسم بديل قديم، لكن plugins الجديدة ينبغي أن
تستخدم `register`.

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

يقوم OpenClaw بتحميل كائن entry واستدعاء `register(api)` أثناء تفعيل plugin.
ولا يزال المحمّل يرجع إلى `activate(api)` للـ plugins الأقدم،
لكن plugins المضمّنة وplugins الخارجية الجديدة ينبغي أن تتعامل مع
`register` بوصفه العقد العام.

طرق التسجيل الشائعة:

| الطريقة | ما الذي تسجّله |
| --------------------------------------- | --------------------------- |
| `registerProvider` | موفّر نماذج (LLM) |
| `registerChannel` | قناة دردشة |
| `registerTool` | أداة عامل |
| `registerHook` / `on(...)` | hooks دورة الحياة |
| `registerSpeechProvider` | تحويل النص إلى كلام / STT |
| `registerRealtimeTranscriptionProvider` | STT متدفق |
| `registerRealtimeVoiceProvider` | صوت فوري ثنائي الاتجاه |
| `registerMediaUnderstandingProvider` | تحليل الصور/الصوت |
| `registerImageGenerationProvider` | توليد الصور |
| `registerVideoGenerationProvider` | توليد الفيديو |
| `registerWebFetchProvider` | موفّر جلب / scrape للويب |
| `registerWebSearchProvider` | بحث على الويب |
| `registerHttpRoute` | نقطة نهاية HTTP |
| `registerCommand` / `registerCli` | أوامر CLI |
| `registerContextEngine` | محرك سياق |
| `registerService` | خدمة خلفية |

سلوك حواجز hooks الخاصة بـ hooks دورة الحياة المTypedة:

- `before_tool_call`: تكون `{ block: true }` نهائية؛ ويتم تخطي المعالِجات ذات الأولوية الأقل.
- `before_tool_call`: تكون `{ block: false }` بلا تأثير ولا تزيل حظرًا سابقًا.
- `before_install`: تكون `{ block: true }` نهائية؛ ويتم تخطي المعالِجات ذات الأولوية الأقل.
- `before_install`: تكون `{ block: false }` بلا تأثير ولا تزيل حظرًا سابقًا.
- `message_sending`: تكون `{ cancel: true }` نهائية؛ ويتم تخطي المعالِجات ذات الأولوية الأقل.
- `message_sending`: تكون `{ cancel: false }` بلا تأثير ولا تزيل إلغاءً سابقًا.

للاطلاع على السلوك الكامل المTyped للـ hooks، راجع [نظرة عامة على SDK](/ar/plugins/sdk-overview#hook-decision-semantics).

## ذو صلة

- [بناء Plugins](/ar/plugins/building-plugins) — أنشئ plugin خاصة بك
- [Plugin Bundles](/ar/plugins/bundles) — التوافق مع حزم Codex/Claude/Cursor
- [Plugin Manifest](/ar/plugins/manifest) — مخطط manifest
- [تسجيل الأدوات](/ar/plugins/building-plugins#registering-agent-tools) — أضف أدوات العامل في plugin
- [Plugin Internals](/plugins/architecture) — نموذج القدرات وخط أنابيب التحميل
- [Community Plugins](/ar/plugins/community) — قوائم الجهات الخارجية
