---
read_when:
    - تهيئة SecretRefs لبيانات اعتماد المزوّدين ومراجع `auth-profiles.json`
    - تشغيل إعادة تحميل الأسرار، والتدقيق، وconfigure، وapply بأمان في الإنتاج
    - فهم الفشل السريع عند بدء التشغيل، وتصفية الأسطح غير النشطة، وسلوك آخر حالة سليمة معروفة
summary: 'إدارة الأسرار: عقد SecretRef، وسلوك لقطات وقت التشغيل، والتنظيف الآمن أحادي الاتجاه'
title: إدارة الأسرار
x-i18n:
    generated_at: "2026-04-05T12:45:08Z"
    model: gpt-5.4
    provider: openai
    source_hash: b91778cb7801fe24f050c15c0a9dd708dda91cb1ce86096e6bae57ebb6e0d41d
    source_path: gateway/secrets.md
    workflow: 15
---

# إدارة الأسرار

يدعم OpenClaw مراجع SecretRef الإضافية بحيث لا يلزم تخزين بيانات الاعتماد المدعومة كنص صريح داخل الإعدادات.

لا يزال النص الصريح يعمل. ومراجع SecretRef اختيارية لكل اعتماد على حدة.

## الأهداف ونموذج وقت التشغيل

يتم حل الأسرار إلى لقطة وقت تشغيل داخل الذاكرة.

- يتم الحل بشكل متحمّس أثناء التفعيل، وليس كسلوك كسول في مسارات الطلبات.
- يفشل بدء التشغيل سريعًا عندما يتعذر حل SecretRef نشط فعليًا.
- تستخدم إعادة التحميل التبديل الذري: نجاح كامل، أو الاحتفاظ بآخر لقطة سليمة معروفة.
- تؤدي مخالفات سياسة SecretRef ‏(على سبيل المثال ملفات تعريف مصادقة في وضع OAuth مع إدخال SecretRef) إلى فشل التفعيل قبل تبديل وقت التشغيل.
- تقرأ طلبات وقت التشغيل من اللقطة النشطة داخل الذاكرة فقط.
- بعد أول تفعيل/تحميل ناجح للإعدادات، تستمر مسارات شيفرة وقت التشغيل في القراءة من تلك اللقطة النشطة داخل الذاكرة حتى تؤدي إعادة تحميل ناجحة إلى استبدالها.
- تقرأ مسارات التسليم الصادر أيضًا من تلك اللقطة النشطة (على سبيل المثال تسليم الرد/السلسلة في Discord وإرسالات الإجراءات في Telegram)؛ ولا تعيد حل SecretRefs عند كل إرسال.

وهذا يبقي انقطاعات مزوّدي الأسرار بعيدًا عن مسارات الطلبات الساخنة.

## تصفية الأسطح النشطة

يتم التحقق من SecretRefs فقط على الأسطح النشطة فعليًا.

- الأسطح المفعلة: تمنع المراجع غير المحلولة بدء التشغيل/إعادة التحميل.
- الأسطح غير النشطة: لا تمنع المراجع غير المحلولة بدء التشغيل/إعادة التحميل.
- تصدر المراجع غير النشطة تشخيصات غير قاتلة بالرمز `SECRETS_REF_IGNORED_INACTIVE_SURFACE`.

أمثلة على الأسطح غير النشطة:

- إدخالات القنوات/الحسابات المعطلة.
- بيانات اعتماد القناة العليا التي لا يرثها أي حساب مفعّل.
- أسطح الأدوات/الميزات المعطلة.
- مفاتيح مزوّدي البحث على الويب الخاصة بكل مزوّد غير المحددة بواسطة `tools.web.search.provider`.
  في الوضع التلقائي (عندما لا يكون المزوّد مضبوطًا)، تتم استشارة المفاتيح حسب الأولوية لاكتشاف المزوّد تلقائيًا حتى يتم حل أحدها.
  وبعد الاختيار، تُعامل مفاتيح المزوّدين غير المختارين على أنها غير نشطة حتى يتم اختيارها.
- مواد مصادقة SSH الخاصة بـ sandbox ‏(`agents.defaults.sandbox.ssh.identityData`،
  و`certificateData`، و`knownHostsData`، بالإضافة إلى التجاوزات لكل وكيل) تكون نشطة فقط
  عندما تكون خلفية sandbox الفعالة هي `ssh` للوكيل الافتراضي أو لوكيل مفعّل.
- تكون مراجع SecretRef الخاصة بـ `gateway.remote.token` / `gateway.remote.password` نشطة إذا تحقق أحد ما يلي:
  - `gateway.mode=remote`
  - تم إعداد `gateway.remote.url`
  - قيمة `gateway.tailscale.mode` هي `serve` أو `funnel`
  - في الوضع المحلي من دون تلك الأسطح البعيدة:
    - يكون `gateway.remote.token` نشطًا عندما يمكن أن تفوز مصادقة الرمز ولا يكون هناك رمز مضبوط في env/auth.
    - يكون `gateway.remote.password` نشطًا فقط عندما يمكن أن تفوز مصادقة كلمة المرور ولا تكون هناك كلمة مرور مضبوطة في env/auth.
- يكون SecretRef الخاص بـ `gateway.auth.token` غير نشط في حل مصادقة بدء التشغيل عندما يكون `OPENCLAW_GATEWAY_TOKEN` مضبوطًا، لأن إدخال رمز env يفوز لذلك وقت التشغيل.

## تشخيصات سطح مصادقة Gateway

عند إعداد SecretRef على `gateway.auth.token` أو `gateway.auth.password` أو
`gateway.remote.token` أو `gateway.remote.password`، تسجل عملية بدء تشغيل/إعادة تحميل gateway
حالة السطح صراحةً:

- `active`: يكون SecretRef جزءًا من سطح المصادقة الفعّال ويجب أن يُحل.
- `inactive`: يتم تجاهل SecretRef لهذا وقت التشغيل لأن سطح مصادقة آخر يفوز، أو
  لأن المصادقة البعيدة معطلة/غير نشطة.

يتم تسجيل هذه الإدخالات بالرمز `SECRETS_GATEWAY_AUTH_SURFACE` وتتضمن السبب المستخدم بواسطة
سياسة السطح النشط، حتى تتمكن من معرفة سبب اعتبار بيانات الاعتماد نشطة أو غير نشطة.

## الفحص المسبق لمرجع onboarding

عندما يعمل onboarding في الوضع التفاعلي وتختار تخزين SecretRef، يشغّل OpenClaw تحققًا مسبقًا قبل الحفظ:

- مراجع env: يتحقق من اسم متغير البيئة ويؤكد أن قيمة غير فارغة مرئية أثناء الإعداد.
- مراجع المزوّد (`file` أو `exec`): يتحقق من اختيار المزوّد، ويحل `id`، ويتحقق من نوع القيمة المحلولة.
- مسار إعادة استخدام البدء السريع: عندما يكون `gateway.auth.token` بالفعل SecretRef، يقوم onboarding بحلّه قبل التهيئة الأولية للفحص/لوحة المعلومات (بالنسبة إلى مراجع `env` و`file` و`exec`) باستخدام بوابة الفشل السريع نفسها.

إذا فشل التحقق، يعرض onboarding الخطأ ويتيح لك إعادة المحاولة.

## عقد SecretRef

استخدم شكل كائن واحد في كل مكان:

```json5
{ source: "env" | "file" | "exec", provider: "default", id: "..." }
```

### `source: "env"`

```json5
{ source: "env", provider: "default", id: "OPENAI_API_KEY" }
```

التحقق:

- يجب أن يطابق `provider` النمط `^[a-z][a-z0-9_-]{0,63}$`
- يجب أن يطابق `id` النمط `^[A-Z][A-Z0-9_]{0,127}$`

### `source: "file"`

```json5
{ source: "file", provider: "filemain", id: "/providers/openai/apiKey" }
```

التحقق:

- يجب أن يطابق `provider` النمط `^[a-z][a-z0-9_-]{0,63}$`
- يجب أن يكون `id` مؤشر JSON مطلقًا (`/...`)
- هروب RFC6901 داخل المقاطع: `~` => `~0`، و`/` => `~1`

### `source: "exec"`

```json5
{ source: "exec", provider: "vault", id: "providers/openai/apiKey" }
```

التحقق:

- يجب أن يطابق `provider` النمط `^[a-z][a-z0-9_-]{0,63}$`
- يجب أن يطابق `id` النمط `^[A-Za-z0-9][A-Za-z0-9._:/-]{0,255}$`
- يجب ألا يحتوي `id` على `.` أو `..` كمقاطع مسار مفصولة بشرطة مائلة (على سبيل المثال يتم رفض `a/../b`)

## إعدادات المزوّد

عرّف المزوّدين تحت `secrets.providers`:

```json5
{
  secrets: {
    providers: {
      default: { source: "env" },
      filemain: {
        source: "file",
        path: "~/.openclaw/secrets.json",
        mode: "json", // or "singleValue"
      },
      vault: {
        source: "exec",
        command: "/usr/local/bin/openclaw-vault-resolver",
        args: ["--profile", "prod"],
        passEnv: ["PATH", "VAULT_ADDR"],
        jsonOnly: true,
      },
    },
    defaults: {
      env: "default",
      file: "filemain",
      exec: "vault",
    },
    resolution: {
      maxProviderConcurrency: 4,
      maxRefsPerProvider: 512,
      maxBatchBytes: 262144,
    },
  },
}
```

### مزوّد Env

- قائمة السماح عبر `allowlist` اختيارية.
- تؤدي قيم env المفقودة/الفارغة إلى فشل الحل.

### مزوّد File

- يقرأ الملف المحلي من `path`.
- يتوقع `mode: "json"` حمولة كائن JSON ويحل `id` كمؤشر.
- يتوقع `mode: "singleValue"` أن يكون معرّف المرجع `"value"` ويعيد محتويات الملف.
- يجب أن يجتاز المسار فحوصات الملكية/الأذونات.
- ملاحظة Windows ذات الفشل المغلق: إذا لم يكن التحقق من ACL متاحًا لمسار ما، يفشل الحل. وبالنسبة إلى المسارات الموثوقة فقط، اضبط `allowInsecurePath: true` على ذلك المزوّد لتجاوز فحوصات أمان المسار.

### مزوّد Exec

- يشغّل المسار المطلق للملف التنفيذي المضبوط، من دون shell.
- افتراضيًا، يجب أن يشير `command` إلى ملف عادي (وليس رابطًا رمزيًا).
- اضبط `allowSymlinkCommand: true` للسماح بمسارات أوامر الروابط الرمزية (على سبيل المثال shims الخاصة بـ Homebrew). ويتحقق OpenClaw من المسار الهدف بعد الحل.
- اجمع `allowSymlinkCommand` مع `trustedDirs` لمسارات مديري الحزم (على سبيل المثال `["/opt/homebrew"]`).
- يدعم timeout، وtimeout لعدم وجود مخرجات، وحدود بايتات المخرجات، وقائمة سماح للبيئة، والأدلة الموثوقة.
- ملاحظة Windows ذات الفشل المغلق: إذا لم يكن التحقق من ACL متاحًا لمسار الأمر، يفشل الحل. وبالنسبة إلى المسارات الموثوقة فقط، اضبط `allowInsecurePath: true` على ذلك المزوّد لتجاوز فحوصات أمان المسار.

حمولة الطلب (`stdin`):

```json
{ "protocolVersion": 1, "provider": "vault", "ids": ["providers/openai/apiKey"] }
```

حمولة الاستجابة (`stdout`):

```jsonc
{ "protocolVersion": 1, "values": { "providers/openai/apiKey": "<openai-api-key>" } } // pragma: allowlist secret
```

أخطاء اختيارية لكل معرّف:

```json
{
  "protocolVersion": 1,
  "values": {},
  "errors": { "providers/openai/apiKey": { "message": "not found" } }
}
```

## أمثلة تكامل Exec

### CLI الخاص بـ 1Password

```json5
{
  secrets: {
    providers: {
      onepassword_openai: {
        source: "exec",
        command: "/opt/homebrew/bin/op",
        allowSymlinkCommand: true, // required for Homebrew symlinked binaries
        trustedDirs: ["/opt/homebrew"],
        args: ["read", "op://Personal/OpenClaw QA API Key/password"],
        passEnv: ["HOME"],
        jsonOnly: false,
      },
    },
  },
  models: {
    providers: {
      openai: {
        baseUrl: "https://api.openai.com/v1",
        models: [{ id: "gpt-5", name: "gpt-5" }],
        apiKey: { source: "exec", provider: "onepassword_openai", id: "value" },
      },
    },
  },
}
```

### CLI الخاص بـ HashiCorp Vault

```json5
{
  secrets: {
    providers: {
      vault_openai: {
        source: "exec",
        command: "/opt/homebrew/bin/vault",
        allowSymlinkCommand: true, // required for Homebrew symlinked binaries
        trustedDirs: ["/opt/homebrew"],
        args: ["kv", "get", "-field=OPENAI_API_KEY", "secret/openclaw"],
        passEnv: ["VAULT_ADDR", "VAULT_TOKEN"],
        jsonOnly: false,
      },
    },
  },
  models: {
    providers: {
      openai: {
        baseUrl: "https://api.openai.com/v1",
        models: [{ id: "gpt-5", name: "gpt-5" }],
        apiKey: { source: "exec", provider: "vault_openai", id: "value" },
      },
    },
  },
}
```

### `sops`

```json5
{
  secrets: {
    providers: {
      sops_openai: {
        source: "exec",
        command: "/opt/homebrew/bin/sops",
        allowSymlinkCommand: true, // required for Homebrew symlinked binaries
        trustedDirs: ["/opt/homebrew"],
        args: ["-d", "--extract", '["providers"]["openai"]["apiKey"]', "/path/to/secrets.enc.json"],
        passEnv: ["SOPS_AGE_KEY_FILE"],
        jsonOnly: false,
      },
    },
  },
  models: {
    providers: {
      openai: {
        baseUrl: "https://api.openai.com/v1",
        models: [{ id: "gpt-5", name: "gpt-5" }],
        apiKey: { source: "exec", provider: "sops_openai", id: "value" },
      },
    },
  },
}
```

## متغيرات بيئة خادم MCP

تدعم متغيرات بيئة خادم MCP المضبوطة عبر `plugins.entries.acpx.config.mcpServers` نوع SecretInput. وهذا يبقي مفاتيح API والرموز خارج الإعدادات النصية الصريحة:

```json5
{
  plugins: {
    entries: {
      acpx: {
        enabled: true,
        config: {
          mcpServers: {
            github: {
              command: "npx",
              args: ["-y", "@modelcontextprotocol/server-github"],
              env: {
                GITHUB_PERSONAL_ACCESS_TOKEN: {
                  source: "env",
                  provider: "default",
                  id: "MCP_GITHUB_PAT",
                },
              },
            },
          },
        },
      },
    },
  },
}
```

لا تزال القيم النصية الصريحة تعمل. ويتم حل مراجع قوالب env مثل `${MCP_SERVER_API_KEY}` وكائنات SecretRef أثناء تفعيل gateway قبل إنشاء عملية خادم MCP. وكما هو الحال مع أسطح SecretRef الأخرى، لا تمنع المراجع غير المحلولة التفعيل إلا عندما يكون plugin ‏`acpx` نشطًا فعليًا.

## مواد مصادقة SSH الخاصة بـ Sandbox

تدعم الخلفية الأساسية `ssh` لـ sandbox أيضًا مراجع SecretRef لمواد مصادقة SSH:

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "ssh",
        ssh: {
          target: "user@gateway-host:22",
          identityData: { source: "env", provider: "default", id: "SSH_IDENTITY" },
          certificateData: { source: "env", provider: "default", id: "SSH_CERTIFICATE" },
          knownHostsData: { source: "env", provider: "default", id: "SSH_KNOWN_HOSTS" },
        },
      },
    },
  },
}
```

سلوك وقت التشغيل:

- يحل OpenClaw هذه المراجع أثناء تفعيل sandbox، وليس كسلوك كسول أثناء كل استدعاء SSH.
- تُكتب القيم المحلولة إلى ملفات مؤقتة ذات أذونات مقيدة وتُستخدم في إعدادات SSH المُولّدة.
- إذا لم تكن الخلفية الفعالة لـ sandbox هي `ssh`، تبقى هذه المراجع غير نشطة ولا تمنع بدء التشغيل.

## سطح بيانات الاعتماد المدعوم

تُسرد بيانات الاعتماد المدعومة وغير المدعومة المعيارية في:

- [سطح بيانات اعتماد SecretRef](/reference/secretref-credential-surface)

يتم استبعاد بيانات الاعتماد التي تُنشأ في وقت التشغيل أو تدور، ومواد تحديث OAuth، عمدًا من حل SecretRef للقراءة فقط.

## السلوك المطلوب والأولوية

- الحقل من دون مرجع: لا تغيير.
- الحقل مع مرجع: مطلوب على الأسطح النشطة أثناء التفعيل.
- إذا وُجد كل من النص الصريح والمرجع، تكون الأولوية للمرجع في مسارات الأولوية المدعومة.
- إن قيمة التنقيح `__OPENCLAW_REDACTED__` محجوزة للتنقيح/الاستعادة الداخلية للإعدادات ويتم رفضها كبيانات إعدادات حرفية مُقدّمة.

إشارات التحذير والتدقيق:

- `SECRETS_REF_OVERRIDES_PLAINTEXT` ‏(تحذير وقت التشغيل)
- `REF_SHADOWED` ‏(نتيجة تدقيق عندما تكون بيانات اعتماد `auth-profiles.json` لها الأولوية على مراجع `openclaw.json`)

سلوك التوافق مع Google Chat:

- تكون الأولوية لـ `serviceAccountRef` على `serviceAccount` النصي الصريح.
- يتم تجاهل القيمة النصية الصريحة عندما يكون المرجع المجاور مضبوطًا.

## محفزات التفعيل

يعمل تفعيل الأسرار عند:

- بدء التشغيل (فحص مسبق ثم تفعيل نهائي)
- مسار التطبيق المباشر لإعادة تحميل الإعدادات
- مسار التحقق من إعادة التشغيل عند إعادة تحميل الإعدادات
- إعادة التحميل اليدوية عبر `secrets.reload`
- الفحص المسبق لاستدعاءات RPC لكتابة إعدادات Gateway ‏(`config.set` / `config.apply` / `config.patch`) لإمكانية حل SecretRef للأسطح النشطة داخل حمولة الإعدادات المُقدمة قبل حفظ التعديلات

عقد التفعيل:

- النجاح يبدّل اللقطة بشكل ذري.
- فشل بدء التشغيل يوقف بدء تشغيل gateway.
- فشل إعادة التحميل في وقت التشغيل يُبقي آخر لقطة سليمة معروفة.
- فشل الفحص المسبق لاستدعاء كتابة RPC يرفض الإعدادات المُقدمة ويُبقي كلًا من إعدادات القرص واللقطة النشطة في وقت التشغيل دون تغيير.
- توفير رمز قناة صريح لكل استدعاء لأداة/مساعد صادر لا يؤدي إلى تشغيل تفعيل SecretRef؛ وتظل نقاط التفعيل محصورة في بدء التشغيل، وإعادة التحميل، و`secrets.reload` الصريح.

## إشارات التدهور والتعافي

عندما يفشل التفعيل أثناء إعادة التحميل بعد حالة سليمة، يدخل OpenClaw في حالة أسرار متدهورة.

حدث نظام لمرة واحدة ورموز السجل:

- `SECRETS_RELOADER_DEGRADED`
- `SECRETS_RELOADER_RECOVERED`

السلوك:

- التدهور: يحتفظ وقت التشغيل بآخر لقطة سليمة معروفة.
- التعافي: يصدر مرة واحدة بعد التفعيل الناجح التالي.
- الإخفاقات المتكررة أثناء التدهور الحالي تسجل تحذيرات لكنها لا تزعج بالأحداث.
- الفشل السريع عند بدء التشغيل لا يصدر أحداث تدهور لأن وقت التشغيل لم يصبح نشطًا أصلًا.

## الحل في مسارات الأوامر

يمكن لمسارات الأوامر الاشتراك في حل SecretRef المدعوم عبر RPC للقطات gateway.

هناك سلوكان عامان:

- مسارات أوامر صارمة (على سبيل المثال مسارات الذاكرة البعيدة في `openclaw memory` و`openclaw qr --remote` عندما تحتاج إلى مراجع أسرار بعيدة مشتركة) تقرأ من اللقطة النشطة وتفشل سريعًا عندما لا يكون SecretRef المطلوب متاحًا.
- مسارات أوامر للقراءة فقط (على سبيل المثال `openclaw status` و`openclaw status --all` و`openclaw channels status` و`openclaw channels resolve` و`openclaw security audit` وتدفقات doctor/config repair للقراءة فقط) تفضّل أيضًا اللقطة النشطة، لكنها تتدهور بدلًا من الإجهاض عندما لا يكون SecretRef المستهدف متاحًا في مسار ذلك الأمر.

سلوك القراءة فقط:

- عندما يكون gateway قيد التشغيل، تقرأ هذه الأوامر من اللقطة النشطة أولًا.
- إذا كان حل gateway غير مكتمل أو كان gateway غير متاح، فإنها تحاول تراجعًا محليًا مستهدفًا لسطح الأمر المحدد.
- إذا ظل SecretRef المستهدف غير متاح، يستمر الأمر بمخرجات قراءة فقط متدهورة مع تشخيصات صريحة مثل “configured but unavailable in this command path”.
- هذا السلوك المتدهور محلي للأمر فقط. وهو لا يضعف مسارات بدء التشغيل، أو إعادة التحميل، أو الإرسال/المصادقة في وقت التشغيل.

ملاحظات أخرى:

- تتم معالجة تحديث اللقطة بعد تدوير الأسرار في الخلفية بواسطة `openclaw secrets reload`.
- طريقة RPC في gateway المستخدمة بواسطة مسارات الأوامر هذه: `secrets.resolve`.

## سير عمل التدقيق وconfigure

تدفق المشغّل الافتراضي:

```bash
openclaw secrets audit --check
openclaw secrets configure
openclaw secrets audit --check
```

### `secrets audit`

تتضمن النتائج:

- القيم النصية الصريحة المخزنة (`openclaw.json` و`auth-profiles.json` و`.env` و`agents/*/agent/models.json` المُولّد)
- بقايا ترويسات المزوّد الحساسة النصية الصريحة في إدخالات `models.json` المُولّدة
- المراجع غير المحلولة
- تظليل الأولوية (`auth-profiles.json` الذي يأخذ الأولوية على مراجع `openclaw.json`)
- البقايا القديمة (`auth.json` وتذكيرات OAuth)

ملاحظة Exec:

- افتراضيًا، يتخطى التدقيق فحوصات إمكانية حل SecretRef من نوع exec لتجنب الآثار الجانبية للأوامر.
- استخدم `openclaw secrets audit --allow-exec` لتنفيذ مزوّدي exec أثناء التدقيق.

ملاحظة بقايا الترويسات:

- يعتمد اكتشاف ترويسات المزوّد الحساسة على استدلال بالأسماء (أسماء ترويسات المصادقة/الاعتماد الشائعة وأجزاء مثل `authorization` و`x-api-key` و`token` و`secret` و`password` و`credential`).

### `secrets configure`

مساعد تفاعلي يقوم بما يلي:

- يضبط `secrets.providers` أولًا (`env`/`file`/`exec`، إضافة/تعديل/إزالة)
- يتيح لك تحديد الحقول المدعومة الحاملة للأسرار في `openclaw.json` بالإضافة إلى `auth-profiles.json` لنطاق وكيل واحد
- يمكنه إنشاء تعيين جديد في `auth-profiles.json` مباشرة داخل منتقي الهدف
- يلتقط تفاصيل SecretRef ‏(`source` و`provider` و`id`)
- يشغّل الحل المسبق preflight
- يمكنه التطبيق فورًا

ملاحظة Exec:

- يتخطى preflight فحوصات SecretRef من نوع exec ما لم يتم ضبط `--allow-exec`.
- إذا قمت بالتطبيق مباشرة من `configure --apply` وكانت الخطة تتضمن مراجع/مزوّدي exec، فأبقِ `--allow-exec` مضبوطًا لخطوة التطبيق أيضًا.

أوضاع مفيدة:

- `openclaw secrets configure --providers-only`
- `openclaw secrets configure --skip-provider-setup`
- `openclaw secrets configure --agent <id>`

الإعدادات الافتراضية لتطبيق `configure`:

- تنظيف بيانات الاعتماد الثابتة المطابقة من `auth-profiles.json` للمزوّدين المستهدفين
- تنظيف إدخالات `api_key` الثابتة القديمة من `auth.json`
- تنظيف أسطر الأسرار المعروفة المطابقة من `<config-dir>/.env`

### `secrets apply`

طبّق خطة محفوظة:

```bash
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --allow-exec
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run --allow-exec
```

ملاحظة Exec:

- يتخطى dry-run فحوصات exec ما لم يتم ضبط `--allow-exec`.
- يرفض وضع الكتابة الخطط التي تحتوي على مراجع/مزوّدي exec ما لم يتم ضبط `--allow-exec`.

للحصول على تفاصيل عقد الهدف/المسار الصارم وقواعد الرفض الدقيقة، راجع:

- [عقد خطة تطبيق الأسرار](/gateway/secrets-plan-contract)

## سياسة الأمان أحادية الاتجاه

يتعمد OpenClaw عدم كتابة نسخ احتياطية للاسترجاع تحتوي على قيم تاريخية نصية صريحة للأسرار.

نموذج الأمان:

- يجب أن ينجح preflight قبل وضع الكتابة
- يتم التحقق من تفعيل وقت التشغيل قبل الالتزام
- يقوم apply بتحديث الملفات باستخدام استبدال ذري للملفات واستعادة بأفضل جهد عند الفشل

## ملاحظات توافق المصادقة القديمة

بالنسبة إلى بيانات الاعتماد الثابتة، لم يعد وقت التشغيل يعتمد على تخزين المصادقة القديم النصي الصريح.

- مصدر بيانات الاعتماد في وقت التشغيل هو اللقطة المحلولة داخل الذاكرة.
- يتم تنظيف إدخالات `api_key` الثابتة القديمة عند اكتشافها.
- يظل سلوك التوافق المتعلق بـ OAuth منفصلًا.

## ملاحظة واجهة الويب

من الأسهل إعداد بعض union الخاصة بـ SecretInput في وضع المحرر الخام مقارنة بوضع النموذج.

## المستندات ذات الصلة

- أوامر CLI: [secrets](/cli/secrets)
- تفاصيل عقد الخطة: [عقد خطة تطبيق الأسرار](/gateway/secrets-plan-contract)
- سطح بيانات الاعتماد: [سطح بيانات اعتماد SecretRef](/reference/secretref-credential-surface)
- إعداد المصادقة: [المصادقة](/gateway/authentication)
- الوضع الأمني: [الأمان](/gateway/security)
- أولوية البيئة: [متغيرات البيئة](/help/environment)
