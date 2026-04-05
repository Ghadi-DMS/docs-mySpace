---
read_when:
    - تريد قراءة الإعداد أو تعديله دون تفاعل
summary: مرجع CLI للأمر `openclaw config` ‏(get/set/unset/file/schema/validate)
title: config
x-i18n:
    generated_at: "2026-04-05T12:38:18Z"
    model: gpt-5.4
    provider: openai
    source_hash: e4de30f41e15297019151ad1a5b306cb331fd5c2beefd5ce5b98fcc51e95f0de
    source_path: cli/config.md
    workflow: 15
---

# `openclaw config`

مساعدات الإعداد لإجراء تعديلات غير تفاعلية في `openclaw.json`: أوامر get/set/unset/file/schema/validate
للقيم حسب المسار وطباعة ملف الإعداد النشط. شغّل الأمر من دون أمر فرعي لفتح
معالج الإعداد (وهو نفسه `openclaw configure`).

خيارات الجذر:

- `--section <section>`: عامل تصفية قابل للتكرار لأقسام الإعداد الموجّه عند تشغيل `openclaw config` من دون أمر فرعي

الأقسام الموجّهة المدعومة:

- `workspace`
- `model`
- `web`
- `gateway`
- `daemon`
- `channels`
- `plugins`
- `skills`
- `health`

## أمثلة

```bash
openclaw config file
openclaw config --section model
openclaw config --section gateway --section daemon
openclaw config schema
openclaw config get browser.executablePath
openclaw config set browser.executablePath "/usr/bin/google-chrome"
openclaw config set agents.defaults.heartbeat.every "2h"
openclaw config set agents.list[0].tools.exec.node "node-id-or-name"
openclaw config set channels.discord.token --ref-provider default --ref-source env --ref-id DISCORD_BOT_TOKEN
openclaw config set secrets.providers.vaultfile --provider-source file --provider-path /etc/openclaw/secrets.json --provider-mode json
openclaw config unset plugins.entries.brave.config.webSearch.apiKey
openclaw config set channels.discord.token --ref-provider default --ref-source env --ref-id DISCORD_BOT_TOKEN --dry-run
openclaw config validate
openclaw config validate --json
```

### `config schema`

يطبع JSON schema المُولَّد لملف `openclaw.json` إلى stdout بصيغة JSON.

ما الذي يتضمنه:

- schema إعداد الجذر الحالية، بالإضافة إلى حقل سلسلة `$schema` على مستوى الجذر لأدوات المحرر
- بيانات التوثيق الوصفية للحقلين `title` و`description` المستخدمة بواسطة Control UI
- ترث العقد الخاصة بالكائنات المتداخلة، والرمز الشامل (`*`)، وعناصر المصفوفات (`[]`) بيانات `title` / `description` الوصفية نفسها عندما تتوفر وثائق حقول مطابقة
- كما ترث فروع `anyOf` / `oneOf` / `allOf` بيانات التوثيق الوصفية نفسها عندما تتوفر وثائق حقول مطابقة
- بيانات schema وصفية مباشرة لأفضل جهد للإضافات + القنوات عندما يمكن تحميل manifests وقت التشغيل
- schema احتياطية نظيفة حتى عندما يكون الإعداد الحالي غير صالح

RPC وقت التشغيل ذو الصلة:

- يعيد `config.schema.lookup` مسار إعداد واحدًا مُطبَّعًا مع عقدة
  schema سطحية (`title`, `description`, `type`, `enum`, `const`, والحدود الشائعة)،
  وبيانات وصفية مطابقة لتلميحات واجهة المستخدم، وملخصات الأبناء المباشرين. استخدمه
  للاستكشاف الموجّه بالمسار في Control UI أو في العملاء المخصصين.

```bash
openclaw config schema
```

مرّره إلى ملف عندما تريد فحصه أو التحقق منه بأدوات أخرى:

```bash
openclaw config schema > openclaw.schema.json
```

### المسارات

تستخدم المسارات صيغة النقطة أو الأقواس:

```bash
openclaw config get agents.defaults.workspace
openclaw config get agents.list[0].id
```

استخدم فهرس قائمة الوكلاء لاستهداف وكيل محدد:

```bash
openclaw config get agents.list
openclaw config set agents.list[1].tools.exec.node "node-id-or-name"
```

## القيم

تُحلَّل القيم بصيغة JSON5 عند الإمكان؛ وإلا تُعامل كسلاسل نصية.
استخدم `--strict-json` لفرض تحليل JSON5. وما يزال `--json` مدعومًا كاسم بديل قديم.

```bash
openclaw config set agents.defaults.heartbeat.every "0m"
openclaw config set gateway.port 19001 --strict-json
openclaw config set channels.whatsapp.groups '["*"]' --strict-json
```

يطبع `config get <path> --json` القيمة الخام بصيغة JSON بدلًا من النص المنسق للطرفية.

## أوضاع `config set`

يدعم `openclaw config set` أربعة أنماط للإسناد:

1. وضع القيمة: `openclaw config set <path> <value>`
2. وضع منشئ SecretRef:

```bash
openclaw config set channels.discord.token \
  --ref-provider default \
  --ref-source env \
  --ref-id DISCORD_BOT_TOKEN
```

3. وضع منشئ المزوّد (للمسار `secrets.providers.<alias>` فقط):

```bash
openclaw config set secrets.providers.vault \
  --provider-source exec \
  --provider-command /usr/local/bin/openclaw-vault \
  --provider-arg read \
  --provider-arg openai/api-key \
  --provider-timeout-ms 5000
```

4. وضع الدُفعات (`--batch-json` أو `--batch-file`):

```bash
openclaw config set --batch-json '[
  {
    "path": "secrets.providers.default",
    "provider": { "source": "env" }
  },
  {
    "path": "channels.discord.token",
    "ref": { "source": "env", "provider": "default", "id": "DISCORD_BOT_TOKEN" }
  }
]'
```

```bash
openclaw config set --batch-file ./config-set.batch.json --dry-run
```

ملاحظة حول السياسة:

- تُرفض إسنادات SecretRef على الأسطح غير المدعومة القابلة للتغيير وقت التشغيل (مثل `hooks.token` و`commands.ownerDisplaySecret` وwebhook tokens الخاصة بربط سلاسل Discord وWhatsApp creds JSON). راجع [SecretRef Credential Surface](/reference/secretref-credential-surface).

يستخدم تحليل الدُفعات دائمًا حمولة الدُفعة (`--batch-json`/`--batch-file`) بوصفها مصدر الحقيقة.
لا يغيّر `--strict-json` / `--json` سلوك تحليل الدُفعات.

ما يزال وضع مسار/قيمة JSON مدعومًا لكل من SecretRefs والمزوّدات:

```bash
openclaw config set channels.discord.token \
  '{"source":"env","provider":"default","id":"DISCORD_BOT_TOKEN"}' \
  --strict-json

openclaw config set secrets.providers.vaultfile \
  '{"source":"file","path":"/etc/openclaw/secrets.json","mode":"json"}' \
  --strict-json
```

## علامات منشئ المزوّد

يجب أن تستخدم أهداف منشئ المزوّد `secrets.providers.<alias>` كمسار.

العلامات الشائعة:

- `--provider-source <env|file|exec>`
- `--provider-timeout-ms <ms>` ‏(`file`, `exec`)

مزوّد env ‏(`--provider-source env`):

- `--provider-allowlist <ENV_VAR>` (قابلة للتكرار)

مزوّد file ‏(`--provider-source file`):

- `--provider-path <path>` (مطلوب)
- `--provider-mode <singleValue|json>`
- `--provider-max-bytes <bytes>`

مزوّد exec ‏(`--provider-source exec`):

- `--provider-command <path>` (مطلوب)
- `--provider-arg <arg>` (قابلة للتكرار)
- `--provider-no-output-timeout-ms <ms>`
- `--provider-max-output-bytes <bytes>`
- `--provider-json-only`
- `--provider-env <KEY=VALUE>` (قابلة للتكرار)
- `--provider-pass-env <ENV_VAR>` (قابلة للتكرار)
- `--provider-trusted-dir <path>` (قابلة للتكرار)
- `--provider-allow-insecure-path`
- `--provider-allow-symlink-command`

مثال لمزوّد exec مُقوّى:

```bash
openclaw config set secrets.providers.vault \
  --provider-source exec \
  --provider-command /usr/local/bin/openclaw-vault \
  --provider-arg read \
  --provider-arg openai/api-key \
  --provider-json-only \
  --provider-pass-env VAULT_TOKEN \
  --provider-trusted-dir /usr/local/bin \
  --provider-timeout-ms 5000
```

## التشغيل التجريبي

استخدم `--dry-run` للتحقق من التغييرات من دون الكتابة إلى `openclaw.json`.

```bash
openclaw config set channels.discord.token \
  --ref-provider default \
  --ref-source env \
  --ref-id DISCORD_BOT_TOKEN \
  --dry-run

openclaw config set channels.discord.token \
  --ref-provider default \
  --ref-source env \
  --ref-id DISCORD_BOT_TOKEN \
  --dry-run \
  --json

openclaw config set channels.discord.token \
  --ref-provider vault \
  --ref-source exec \
  --ref-id discord/token \
  --dry-run \
  --allow-exec
```

سلوك التشغيل التجريبي:

- وضع المنشئ: يشغّل فحوصات قابلية حل SecretRef للمراجع/المزوّدات المتغيرة.
- وضع JSON ‏(`--strict-json` أو `--json` أو وضع الدُفعات): يشغّل التحقق من schema بالإضافة إلى فحوصات قابلية حل SecretRef.
- يجري أيضًا التحقق من السياسة للأسطح المعروفة غير المدعومة كأهداف SecretRef.
- تقيّم فحوصات السياسة كامل الإعداد بعد التغيير، لذلك لا يمكن لعمليات الكتابة على الكائن الأب (مثل ضبط `hooks` ككائن) تجاوز التحقق من الأسطح غير المدعومة.
- تُتخطى فحوصات SecretRef الخاصة بـ exec افتراضيًا أثناء التشغيل التجريبي لتجنب الآثار الجانبية للأوامر.
- استخدم `--allow-exec` مع `--dry-run` للاشتراك في فحوصات SecretRef الخاصة بـ exec (قد يؤدي هذا إلى تنفيذ أوامر المزوّد).
- `--allow-exec` مخصص للتشغيل التجريبي فقط ويُرجع خطأ إذا استُخدم من دون `--dry-run`.

يطبع `--dry-run --json` تقريرًا قابلًا للقراءة آليًا:

- `ok`: ما إذا نجح التشغيل التجريبي
- `operations`: عدد الإسنادات التي جرى تقييمها
- `checks`: ما إذا شُغِّلت فحوصات schema/قابلية الحل
- `checks.resolvabilityComplete`: ما إذا اكتملت فحوصات قابلية الحل (تكون false عند تخطي مراجع exec)
- `refsChecked`: عدد المراجع التي جرى حلها فعليًا أثناء التشغيل التجريبي
- `skippedExecRefs`: عدد مراجع exec التي جرى تخطيها لأن `--allow-exec` لم يُضبط
- `errors`: إخفاقات schema/قابلية الحل المنظمة عندما تكون `ok=false`

### شكل خرج JSON

```json5
{
  ok: boolean,
  operations: number,
  configPath: string,
  inputModes: ["value" | "json" | "builder", ...],
  checks: {
    schema: boolean,
    resolvability: boolean,
    resolvabilityComplete: boolean,
  },
  refsChecked: number,
  skippedExecRefs: number,
  errors?: [
    {
      kind: "schema" | "resolvability",
      message: string,
      ref?: string, // present for resolvability errors
    },
  ],
}
```

مثال نجاح:

```json
{
  "ok": true,
  "operations": 1,
  "configPath": "~/.openclaw/openclaw.json",
  "inputModes": ["builder"],
  "checks": {
    "schema": false,
    "resolvability": true,
    "resolvabilityComplete": true
  },
  "refsChecked": 1,
  "skippedExecRefs": 0
}
```

مثال فشل:

```json
{
  "ok": false,
  "operations": 1,
  "configPath": "~/.openclaw/openclaw.json",
  "inputModes": ["builder"],
  "checks": {
    "schema": false,
    "resolvability": true,
    "resolvabilityComplete": true
  },
  "refsChecked": 1,
  "skippedExecRefs": 0,
  "errors": [
    {
      "kind": "resolvability",
      "message": "Error: Environment variable \"MISSING_TEST_SECRET\" is not set.",
      "ref": "env:default:MISSING_TEST_SECRET"
    }
  ]
}
```

إذا فشل التشغيل التجريبي:

- `config schema validation failed`: شكل الإعداد بعد التغيير غير صالح؛ أصلح شكل المسار/القيمة أو كائن المزوّد/المرجع.
- `Config policy validation failed: unsupported SecretRef usage`: أعد بيانات الاعتماد هذه إلى إدخال نصي عادي/سلسلة نصية، وأبقِ SecretRefs على الأسطح المدعومة فقط.
- `SecretRef assignment(s) could not be resolved`: لا يمكن حاليًا حل المزوّد/المرجع المشار إليه (متغير env مفقود، أو مؤشر ملف غير صالح، أو فشل مزوّد exec، أو عدم تطابق المزوّد/المصدر).
- `Dry run note: skipped <n> exec SecretRef resolvability check(s)`: تخطى التشغيل التجريبي مراجع exec؛ أعد التشغيل مع `--allow-exec` إذا كنت تحتاج إلى التحقق من قابلية حل exec.
- في وضع الدُفعات، أصلح الإدخالات الفاشلة وأعد تشغيل `--dry-run` قبل الكتابة.

## الأوامر الفرعية

- `config file`: يطبع مسار ملف الإعداد النشط (المحلول من `OPENCLAW_CONFIG_PATH` أو من الموقع الافتراضي).

أعد تشغيل gateway بعد التعديلات.

## التحقق

تحقق من الإعداد الحالي مقابل schema النشطة من دون بدء
gateway.

```bash
openclaw config validate
openclaw config validate --json
```
