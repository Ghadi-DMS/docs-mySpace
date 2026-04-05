---
read_when:
    - تريد أتمتة قائمة على الأحداث لأوامر /new و/reset و/stop وأحداث دورة حياة الوكيل
    - تريد إنشاء الخطافات أو تثبيتها أو تصحيحها
summary: 'الخطافات: أتمتة قائمة على الأحداث للأوامر وأحداث دورة الحياة'
title: الخطافات
x-i18n:
    generated_at: "2026-04-05T12:34:28Z"
    model: gpt-5.4
    provider: openai
    source_hash: 66eb75bb2b3b2ad229bf3da24fdb0fe021ed08f812fd1d13c69b3bd9df0218e5
    source_path: automation/hooks.md
    workflow: 15
---

# الخطافات

الخطافات هي نصوص برمجية صغيرة تعمل عندما يحدث شيء ما داخل Gateway. يتم اكتشافها تلقائيًا من الأدلة ويمكن فحصها باستخدام `openclaw hooks`.

هناك نوعان من الخطافات في OpenClaw:

- **الخطافات الداخلية** (هذه الصفحة): تعمل داخل Gateway عند حدوث أحداث الوكيل، مثل `/new` أو `/reset` أو `/stop` أو أحداث دورة الحياة.
- **Webhooks**: نقاط نهاية HTTP خارجية تتيح للأنظمة الأخرى تشغيل العمل في OpenClaw. راجع [Webhooks](/automation/cron-jobs#webhooks).

يمكن أيضًا تضمين الخطافات داخل plugins. يعرض `openclaw hooks list` كلًا من الخطافات المستقلة والخطافات التي تديرها plugins.

## البدء السريع

```bash
# List available hooks
openclaw hooks list

# Enable a hook
openclaw hooks enable session-memory

# Check hook status
openclaw hooks check

# Get detailed information
openclaw hooks info session-memory
```

## أنواع الأحداث

| الحدث                    | وقت تشغيله                                     |
| ------------------------ | ---------------------------------------------- |
| `command:new`            | عند إصدار الأمر `/new`                        |
| `command:reset`          | عند إصدار الأمر `/reset`                      |
| `command:stop`           | عند إصدار الأمر `/stop`                       |
| `command`                | أي حدث أمر (مستمع عام)                        |
| `session:compact:before` | قبل أن يلخص الضغط السجل                        |
| `session:compact:after`  | بعد اكتمال الضغط                               |
| `session:patch`          | عند تعديل خصائص الجلسة                         |
| `agent:bootstrap`        | قبل إدخال ملفات التمهيد الخاصة بمساحة العمل    |
| `gateway:startup`        | بعد بدء القنوات وتحميل الخطافات                |
| `message:received`       | رسالة واردة من أي قناة                         |
| `message:transcribed`    | بعد اكتمال نسخ الصوت                           |
| `message:preprocessed`   | بعد اكتمال فهم جميع الوسائط والروابط           |
| `message:sent`           | تم تسليم الرسالة الصادرة                       |

## كتابة الخطافات

### بنية الخطاف

كل خطاف عبارة عن دليل يحتوي على ملفين:

```
my-hook/
├── HOOK.md          # Metadata + documentation
└── handler.ts       # Handler implementation
```

### تنسيق HOOK.md

```markdown
---
name: my-hook
description: "Short description of what this hook does"
metadata:
  { "openclaw": { "emoji": "🔗", "events": ["command:new"], "requires": { "bins": ["node"] } } }
---

# My Hook

Detailed documentation goes here.
```

**حقول Metadata** (`metadata.openclaw`):

| الحقل      | الوصف                                                 |
| ---------- | ----------------------------------------------------- |
| `emoji`    | رمز تعبيري للعرض في CLI                               |
| `events`   | مصفوفة بالأحداث المطلوب الاستماع إليها                |
| `export`   | التصدير المسمى المطلوب استخدامه (الافتراضي `"default"`) |
| `os`       | المنصات المطلوبة (مثل `["darwin", "linux"]`)          |
| `requires` | مسارات `bins` أو `anyBins` أو `env` أو `config` المطلوبة |
| `always`   | تجاوز فحوصات الأهلية (قيمة منطقية)                    |
| `install`  | طرق التثبيت                                           |

### تنفيذ المعالج

```typescript
const handler = async (event) => {
  if (event.type !== "command" || event.action !== "new") {
    return;
  }

  console.log(`[my-hook] New command triggered`);
  // Your logic here

  // Optionally send message to user
  event.messages.push("Hook executed!");
};

export default handler;
```

يتضمن كل حدث: `type` و`action` و`sessionKey` و`timestamp` و`messages` (استخدم push لإرسالها إلى المستخدم) و`context` (بيانات خاصة بالحدث).

### أبرز عناصر سياق الحدث

**أحداث الأوامر** (`command:new`, `command:reset`): ‏`context.sessionEntry` و`context.previousSessionEntry` و`context.commandSource` و`context.workspaceDir` و`context.cfg`.

**أحداث الرسائل** (`message:received`): ‏`context.from` و`context.content` و`context.channelId` و`context.metadata` (بيانات خاصة بالمزوّد تتضمن `senderId` و`senderName` و`guildId`).

**أحداث الرسائل** (`message:sent`): ‏`context.to` و`context.content` و`context.success` و`context.channelId`.

**أحداث الرسائل** (`message:transcribed`): ‏`context.transcript` و`context.from` و`context.channelId` و`context.mediaPath`.

**أحداث الرسائل** (`message:preprocessed`): ‏`context.bodyForAgent` (النص النهائي المُثْرى) و`context.from` و`context.channelId`.

**أحداث التمهيد** (`agent:bootstrap`): ‏`context.bootstrapFiles` (مصفوفة قابلة للتعديل) و`context.agentId`.

**أحداث تصحيح الجلسة** (`session:patch`): ‏`context.sessionEntry` و`context.patch` (الحقول المتغيرة فقط) و`context.cfg`. يمكن فقط للعملاء ذوي الامتيازات تشغيل أحداث التصحيح.

**أحداث الضغط**: يتضمن `session:compact:before` القيمتين `messageCount` و`tokenCount`. ويضيف `session:compact:after` القيم `compactedCount` و`summaryLength` و`tokensBefore` و`tokensAfter`.

## اكتشاف الخطافات

يتم اكتشاف الخطافات من هذه الأدلة، بترتيب تصاعدي لأولوية التجاوز:

1. **الخطافات المضمّنة**: تأتي مع OpenClaw
2. **خطافات plugins**: خطافات مضمّنة داخل plugins المثبتة
3. **الخطافات المُدارة**: ‏`~/.openclaw/hooks/` (مثبتة من المستخدم، ومشتركة بين مساحات العمل). تشترك الأدلة الإضافية من `hooks.internal.load.extraDirs` في هذه الأولوية.
4. **خطافات مساحة العمل**: ‏`<workspace>/hooks/` (لكل وكيل، ومعطّلة افتراضيًا حتى يتم تفعيلها صراحة)

يمكن لخطافات مساحة العمل إضافة أسماء خطافات جديدة، لكنها لا يمكنها تجاوز الخطافات المضمّنة أو المُدارة أو المقدمة من plugin بالاسم نفسه.

### حزم الخطافات

حزم الخطافات هي حزم npm تصدّر الخطافات عبر `openclaw.hooks` في `package.json`. ثبّتها باستخدام:

```bash
openclaw plugins install <path-or-spec>
```

تكون مواصفات npm من السجل فقط (اسم الحزمة مع إصدار دقيق اختياري أو dist-tag). يتم رفض مواصفات Git/URL/file ونطاقات semver.

## الخطافات المضمّنة

| الخطاف                | الأحداث                        | ما الذي يفعله                                         |
| --------------------- | ------------------------------ | ----------------------------------------------------- |
| session-memory        | `command:new`, `command:reset` | يحفظ سياق الجلسة في `<workspace>/memory/`             |
| bootstrap-extra-files | `agent:bootstrap`              | يدرج ملفات تمهيد إضافية من أنماط glob                 |
| command-logger        | `command`                      | يسجل جميع الأوامر في `~/.openclaw/logs/commands.log`  |
| boot-md               | `gateway:startup`              | يشغّل `BOOT.md` عند بدء gateway                       |

لتفعيل أي خطاف مضمّن:

```bash
openclaw hooks enable <hook-name>
```

### تفاصيل session-memory

يستخرج آخر 15 رسالة مستخدم/مساعد، وينشئ slug وصفيًا لاسم الملف عبر LLM، ويحفظه في `<workspace>/memory/YYYY-MM-DD-slug.md`. يتطلب تكوين `workspace.dir`.

### إعداد bootstrap-extra-files

```json
{
  "hooks": {
    "internal": {
      "entries": {
        "bootstrap-extra-files": {
          "enabled": true,
          "paths": ["packages/*/AGENTS.md", "packages/*/TOOLS.md"]
        }
      }
    }
  }
}
```

تُحلّ المسارات نسبةً إلى مساحة العمل. يتم تحميل أسماء ملفات التمهيد الأساسية المعترف بها فقط (`AGENTS.md` و`SOUL.md` و`TOOLS.md` و`IDENTITY.md` و`USER.md` و`HEARTBEAT.md` و`BOOTSTRAP.md` و`MEMORY.md`).

## خطافات plugins

يمكن لـ plugins تسجيل خطافات من خلال Plugin SDK من أجل تكامل أعمق: اعتراض استدعاءات الأدوات، وتعديل المطالبات، والتحكم في تدفق الرسائل، وغير ذلك. يوفّر Plugin SDK عدد 28 خطافًا تغطي تحليل النموذج، ودورة حياة الوكيل، وتدفق الرسائل، وتنفيذ الأدوات، وتنسيق الوكلاء الفرعيين، ودورة حياة gateway.

للاطلاع على المرجع الكامل لخطافات plugin بما في ذلك `before_tool_call` و`before_agent_reply` و`before_install` وجميع خطافات plugin الأخرى، راجع [Plugin Architecture](/plugins/architecture#provider-runtime-hooks).

## التكوين

```json
{
  "hooks": {
    "internal": {
      "enabled": true,
      "entries": {
        "session-memory": { "enabled": true },
        "command-logger": { "enabled": false }
      }
    }
  }
}
```

متغيرات البيئة لكل خطاف:

```json
{
  "hooks": {
    "internal": {
      "entries": {
        "my-hook": {
          "enabled": true,
          "env": { "MY_CUSTOM_VAR": "value" }
        }
      }
    }
  }
}
```

أدلة خطافات إضافية:

```json
{
  "hooks": {
    "internal": {
      "load": {
        "extraDirs": ["/path/to/more/hooks"]
      }
    }
  }
}
```

<Note>
لا تزال صيغة التكوين القديمة للمصفوفة `hooks.internal.handlers` مدعومة للتوافق مع الإصدارات السابقة، لكن يجب أن تستخدم الخطافات الجديدة النظام القائم على الاكتشاف.
</Note>

## مرجع CLI

```bash
# List all hooks (add --eligible, --verbose, or --json)
openclaw hooks list

# Show detailed info about a hook
openclaw hooks info <hook-name>

# Show eligibility summary
openclaw hooks check

# Enable/disable
openclaw hooks enable <hook-name>
openclaw hooks disable <hook-name>
```

## أفضل الممارسات

- **أبقِ المعالجات سريعة.** تعمل الخطافات أثناء معالجة الأوامر. نفّذ الأعمال الثقيلة بشكل fire-and-forget باستخدام `void processInBackground(event)`.
- **تعامل مع الأخطاء بسلاسة.** لف العمليات المحفوفة بالمخاطر داخل try/catch؛ ولا تُطلق الأخطاء حتى تتمكن المعالجات الأخرى من العمل.
- **قم بتصفية الأحداث مبكرًا.** أعد فورًا إذا لم يكن نوع الحدث/الإجراء ذا صلة.
- **استخدم مفاتيح أحداث محددة.** فضّل `"events": ["command:new"]` على `"events": ["command"]` لتقليل الحمل.

## استكشاف الأخطاء وإصلاحها

### لم يتم اكتشاف الخطاف

```bash
# Verify directory structure
ls -la ~/.openclaw/hooks/my-hook/
# Should show: HOOK.md, handler.ts

# List all discovered hooks
openclaw hooks list
```

### الخطاف غير مؤهل

```bash
openclaw hooks info my-hook
```

تحقق من الثنائيات المفقودة (PATH) أو متغيرات البيئة أو قيم التكوين أو توافق نظام التشغيل.

### الخطاف لا يتم تنفيذه

1. تحقق من أن الخطاف مفعّل: `openclaw hooks list`
2. أعد تشغيل عملية gateway حتى تُعاد تحميل الخطافات.
3. تحقق من سجلات gateway: ‏`./scripts/clawlog.sh | grep hook`

## ذو صلة

- [مرجع CLI: hooks](/cli/hooks)
- [Webhooks](/automation/cron-jobs#webhooks)
- [Plugin Architecture](/plugins/architecture#provider-runtime-hooks) — المرجع الكامل لخطافات plugin
- [Configuration](/gateway/configuration-reference#hooks)
