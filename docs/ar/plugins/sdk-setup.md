---
read_when:
    - أنت تضيف معالج إعداد إلى مكوّن إضافي
    - تحتاج إلى فهم الفرق بين `setup-entry.ts` و`index.ts`
    - أنت تعرّف schemas إعداد المكوّن الإضافي أو metadata الخاصة بـ `package.json` لـ OpenClaw
sidebarTitle: Setup and Config
summary: معالجات الإعداد، و`setup-entry.ts`، وschemas الخاصة بالإعداد، وmetadata في `package.json`
title: إعداد المكوّنات الإضافية وتكوينها
x-i18n:
    generated_at: "2026-04-05T12:52:21Z"
    model: gpt-5.4
    provider: openai
    source_hash: 68fda27be1c89ea6ba906833113e9190ddd0ab358eb024262fb806746d54f7bf
    source_path: plugins/sdk-setup.md
    workflow: 15
---

# إعداد المكوّنات الإضافية وتكوينها

مرجع لتغليف المكوّنات الإضافية (`package.json` metadata)، وملفات manifest
(`openclaw.plugin.json`)، ونقاط دخول الإعداد، وschemas الخاصة بالإعداد.

<Tip>
  **هل تبحث عن شرح عملي؟** تغطي أدلة كيفية التنفيذ موضوع التغليف في سياقه:
  [Channel Plugins](/plugins/sdk-channel-plugins#step-1-package-and-manifest) و
  [Provider Plugins](/plugins/sdk-provider-plugins#step-1-package-and-manifest).
</Tip>

## metadata الخاصة بالحزمة

يحتاج `package.json` الخاص بك إلى حقل `openclaw` يوضح لنظام المكوّنات الإضافية
ما الذي يوفّره المكوّن الإضافي:

**مكوّن إضافي لقناة:**

```json
{
  "name": "@myorg/openclaw-my-channel",
  "version": "1.0.0",
  "type": "module",
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "channel": {
      "id": "my-channel",
      "label": "My Channel",
      "blurb": "Short description of the channel."
    }
  }
}
```

**مكوّن إضافي لمزوّد / خط أساس للنشر على ClawHub:**

```json openclaw-clawhub-package.json
{
  "name": "@myorg/openclaw-my-plugin",
  "version": "1.0.0",
  "type": "module",
  "openclaw": {
    "extensions": ["./index.ts"],
    "compat": {
      "pluginApi": ">=2026.3.24-beta.2",
      "minGatewayVersion": "2026.3.24-beta.2"
    },
    "build": {
      "openclawVersion": "2026.3.24-beta.2",
      "pluginSdkVersion": "2026.3.24-beta.2"
    }
  }
}
```

إذا كنت تنشر المكوّن الإضافي خارجيًا على ClawHub، فإن الحقول `compat` و`build`
هذه تكون مطلوبة. وتوجد مقتطفات النشر الرسمية في
`docs/snippets/plugin-publish/`.

### حقول `openclaw`

| الحقل        | النوع      | الوصف                                                                                             |
| ------------ | ---------- | ------------------------------------------------------------------------------------------------- |
| `extensions` | `string[]` | ملفات نقطة الدخول (نسبية إلى جذر الحزمة)                                                         |
| `setupEntry` | `string`   | نقطة دخول خفيفة للإعداد فقط (اختياري)                                                             |
| `channel`    | `object`   | metadata لفهرس القنوات من أجل الإعداد، والمنتقي، والبدء السريع، وأسطر الحالة                     |
| `providers`  | `string[]` | معرّفات المزوّدين التي يسجلها هذا المكوّن الإضافي                                                 |
| `install`    | `object`   | تلميحات التثبيت: `npmSpec`, `localPath`, `defaultChoice`, `minHostVersion`, `allowInvalidConfigRecovery` |
| `startup`    | `object`   | أعلام سلوك بدء التشغيل                                                                            |

### `openclaw.channel`

يمثل `openclaw.channel` metadata رخيصة على مستوى الحزمة لاكتشاف القنوات وأسطر الإعداد
قبل تحميل وقت التشغيل.

| الحقل                                  | النوع      | المعنى                                                                  |
| -------------------------------------- | ---------- | ----------------------------------------------------------------------- |
| `id`                                   | `string`   | معرّف القناة الرسمي.                                                   |
| `label`                                | `string`   | التسمية الأساسية للقناة.                                               |
| `selectionLabel`                       | `string`   | تسمية المنتقي/الإعداد عندما ينبغي أن تختلف عن `label`.                 |
| `detailLabel`                          | `string`   | تسمية تفصيلية ثانوية لفهارس القنوات وأسطح الحالة الأغنى.               |
| `docsPath`                             | `string`   | مسار الوثائق لروابط الإعداد والاختيار.                                  |
| `docsLabel`                            | `string`   | تسمية بديلة مستخدمة في روابط الوثائق عندما ينبغي أن تختلف عن معرّف القناة. |
| `blurb`                                | `string`   | وصف قصير للإعداد الأولي/الفهرس.                                        |
| `order`                                | `number`   | ترتيب الفرز في فهارس القنوات.                                          |
| `aliases`                              | `string[]` | أسماء بديلة إضافية للبحث عن القناة عند الاختيار.                       |
| `preferOver`                           | `string[]` | معرّفات plugin/channel ذات أولوية أدنى ينبغي أن تتفوق عليها هذه القناة. |
| `systemImage`                          | `string`   | اسم أيقونة/صورة نظام اختيارية لفهارس UI الخاصة بالقنوات.               |
| `selectionDocsPrefix`                  | `string`   | نص بادئة قبل روابط الوثائق في أسطح الاختيار.                            |
| `selectionDocsOmitLabel`               | `boolean`  | يعرض مسار الوثائق مباشرة بدل رابط وثائق معنّون في نص الاختيار.          |
| `selectionExtras`                      | `string[]` | سلاسل قصيرة إضافية تُلحق في نص الاختيار.                               |
| `markdownCapable`                      | `boolean`  | يحدد القناة على أنها قادرة على Markdown لقرارات التنسيق الصادر.         |
| `showConfigured`                       | `boolean`  | يتحكم في ما إذا كانت أسطح عرض القنوات المُكوّنة تُظهر هذه القناة.        |
| `quickstartAllowFrom`                  | `boolean`  | يدرج هذه القناة في تدفق الإعداد السريع القياسي لـ `allowFrom`.          |
| `forceAccountBinding`                  | `boolean`  | يفرض ربط الحساب صراحةً حتى عند وجود حساب واحد فقط.                     |
| `preferSessionLookupForAnnounceTarget` | `boolean`  | يفضّل lookup الخاصة بالجلسة عند حل أهداف announce لهذه القناة.          |

مثال:

```json
{
  "openclaw": {
    "channel": {
      "id": "my-channel",
      "label": "My Channel",
      "selectionLabel": "My Channel (self-hosted)",
      "detailLabel": "My Channel Bot",
      "docsPath": "/channels/my-channel",
      "docsLabel": "my-channel",
      "blurb": "Webhook-based self-hosted chat integration.",
      "order": 80,
      "aliases": ["mc"],
      "preferOver": ["my-channel-legacy"],
      "selectionDocsPrefix": "Guide:",
      "selectionExtras": ["Markdown"],
      "markdownCapable": true,
      "quickstartAllowFrom": true
    }
  }
}
```

### `openclaw.install`

إن `openclaw.install` هي metadata خاصة بالحزمة، وليست metadata خاصة بالـ manifest.

| الحقل                        | النوع                 | المعنى                                                                        |
| ---------------------------- | -------------------- | ----------------------------------------------------------------------------- |
| `npmSpec`                    | `string`             | مواصفة npm الرسمية لتدفقات التثبيت/التحديث.                                   |
| `localPath`                  | `string`             | مسار تثبيت محلي للتطوير أو التثبيت المضمّن.                                    |
| `defaultChoice`              | `"npm"` \| `"local"` | مصدر التثبيت المفضل عند توفر الاثنين.                                         |
| `minHostVersion`             | `string`             | الحد الأدنى لإصدار OpenClaw المدعوم بصيغة `>=x.y.z`.                          |
| `allowInvalidConfigRecovery` | `boolean`            | يسمح لتدفقات إعادة تثبيت المكوّن الإضافي المضمّن بالتعافي من بعض أعطال الإعداد القديمة. |

إذا كان `minHostVersion` مضبوطًا، فإن التثبيت وتحميل سجل manifest كلاهما يفرضان
هذا القيد. وتتخطى المضيفات الأقدم المكوّن الإضافي؛ كما تُرفض سلاسل الإصدارات غير الصالحة.

إن `allowInvalidConfigRecovery` ليس تجاوزًا عامًا للإعدادات المعطوبة. فهو
مخصص فقط للتعافي الضيق للمكوّنات الإضافية المضمّنة، بحيث يمكن لإعادة التثبيت/الإعداد إصلاح
بقايا ترقيات معروفة مثل غياب مسار مكوّن إضافي مضمّن أو إدخال قديم
`channels.<id>` لذلك المكوّن الإضافي نفسه. وإذا كان الإعداد معطوبًا لأسباب غير ذات صلة، فإن
التثبيت يظل يفشل بإغلاق آمن ويطلب من المشغّل تشغيل `openclaw doctor --fix`.

### تأجيل التحميل الكامل

يمكن لمكونات القنوات الإضافية الاشتراك في التحميل المؤجل باستخدام:

```json
{
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "startup": {
      "deferConfiguredChannelFullLoadUntilAfterListen": true
    }
  }
}
```

عند التفعيل، يحمّل OpenClaw فقط `setupEntry` خلال مرحلة بدء التشغيل
السابقة للاستماع، حتى بالنسبة إلى القنوات المُكوّنة مسبقًا. ثم تُحمّل نقطة الدخول الكاملة بعد
أن يبدأ gateway في الاستماع.

<Warning>
  لا تفعّل التحميل المؤجل إلا عندما يسجل `setupEntry` لديك كل ما
  يحتاجه gateway قبل أن يبدأ الاستماع (تسجيل القناة، ومسارات HTTP،
  وطرق gateway). وإذا كانت نقطة الدخول الكاملة تملك إمكانات بدء تشغيل مطلوبة، فأبقِ
  السلوك الافتراضي.
</Warning>

إذا كانت نقطة دخول setup/full لديك تسجل طرق Gateway RPC، فأبقها ضمن
بادئة خاصة بالمكوّن الإضافي. أما مساحات أسماء الإدارة الأساسية المحجوزة (`config.*`,
`exec.approvals.*`, `wizard.*`, `update.*`) فتبقى مملوكة للأساس وتُحل دائمًا
إلى `operator.admin`.

## Plugin manifest

يجب أن يشحن كل مكوّن إضافي أصلي ملف `openclaw.plugin.json` في جذر الحزمة.
ويستخدم OpenClaw هذا للتحقق من الإعداد من دون تنفيذ كود المكوّن الإضافي.

```json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "description": "Adds My Plugin capabilities to OpenClaw",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "webhookSecret": {
        "type": "string",
        "description": "Webhook verification secret"
      }
    }
  }
}
```

بالنسبة إلى مكونات القنوات الإضافية، أضف `kind` و`channels`:

```json
{
  "id": "my-channel",
  "kind": "channel",
  "channels": ["my-channel"],
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

حتى المكونات الإضافية التي لا تحتوي على إعداد يجب أن تشحن schema. وتكون schema الفارغة صالحة:

```json
{
  "id": "my-plugin",
  "configSchema": {
    "type": "object",
    "additionalProperties": false
  }
}
```

راجع [Plugin Manifest](/plugins/manifest) للمرجع الكامل الخاص بالـ schema.

## النشر على ClawHub

بالنسبة إلى حزم المكونات الإضافية، استخدم أمر ClawHub الخاص بالحزمة:

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

إن الاسم البديل القديم للنشر المخصص للمهارات فقط هو للمهارات. أما حزم المكونات الإضافية فينبغي
أن تستخدم دائمًا `clawhub package publish`.

## نقطة دخول الإعداد

يمثل الملف `setup-entry.ts` بديلًا خفيفًا لـ `index.ts` يقوم
OpenClaw بتحميله عندما يحتاج فقط إلى أسطح الإعداد (الإعداد الأولي، وإصلاح الإعداد،
وفحص القنوات المعطلة).

```typescript
// setup-entry.ts
import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
import { myChannelPlugin } from "./src/channel.js";

export default defineSetupPluginEntry(myChannelPlugin);
```

يتجنب هذا تحميل كود وقت تشغيل ثقيل (مكتبات التشفير، وتسجيلات CLI،
والخدمات الخلفية) أثناء تدفقات الإعداد.

**متى يستخدم OpenClaw `setupEntry` بدل نقطة الدخول الكاملة:**

- تكون القناة معطلة لكنها تحتاج إلى أسطح إعداد/إعداد أولي
- تكون القناة مفعلة لكنها غير مهيأة
- يكون التحميل المؤجل مفعّلًا (`deferConfiguredChannelFullLoadUntilAfterListen`)

**ما الذي يجب أن تسجله `setupEntry`:**

- كائن plugin الخاص بالقناة (عبر `defineSetupPluginEntry`)
- أي مسارات HTTP مطلوبة قبل استماع gateway
- أي طرق gateway لازمة أثناء بدء التشغيل

ومع ذلك ينبغي أن تتجنب طرق gateway الخاصة ببدء التشغيل هذه مساحات أسماء الإدارة الأساسية
المحجوزة مثل `config.*` أو `update.*`.

**ما الذي يجب ألا تتضمنه `setupEntry`:**

- تسجيلات CLI
- الخدمات الخلفية
- استيرادات وقت تشغيل ثقيلة (crypto، SDKs)
- طرق gateway المطلوبة فقط بعد بدء التشغيل

### استيرادات مساعدات الإعداد الضيقة

بالنسبة إلى المسارات الساخنة الخاصة بالإعداد فقط، فضّل نقاط الوصل الضيقة لمساعدات الإعداد بدل المظلة الأوسع
`plugin-sdk/setup` عندما لا تحتاج إلا إلى جزء من سطح الإعداد:

| مسار الاستيراد                     | استخدمه من أجل                                                                          | أهم الصادرات                                                                                                                                                                                                                                                                                      |
| ---------------------------------- | --------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/setup-runtime`         | مساعدات وقت التشغيل الخاصة بالإعداد التي تبقى متاحة في `setupEntry` / بدء التشغيل المؤجل للقناة | `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
| `plugin-sdk/setup-adapter-runtime` | محولات إعداد الحساب المدركة للبيئة                                                     | `createEnvPatchedAccountSetupAdapter`                                                                                                                                                                                                                                                              |
| `plugin-sdk/setup-tools`           | مساعدات CLI/الأرشفة/الوثائق الخاصة بالإعداد/التثبيت                                     | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR`                                                                                                                                                                                    |

استخدم نقطة الوصل الأوسع `plugin-sdk/setup` عندما تريد صندوق أدوات الإعداد
المشترك الكامل، بما في ذلك مساعدات ترقيع الإعداد مثل
`moveSingleAccountChannelSectionToDefaultAccount(...)`.

تبقى محولات ترقيع الإعداد آمنة للمسار الساخن عند الاستيراد. فبحث سطح العقود الخاص
بترقية الحساب الفردي المضمّن يتم بكسل، لذلك فإن استيراد
`plugin-sdk/setup-runtime` لا يحمّل استكشاف سطح العقود المجمعة مسبقًا
قبل استخدام المحول فعليًا.

### ترقية الحساب الفردي المملوكة للقناة

عندما تنتقل قناة من إعداد حساب فردي على المستوى الأعلى إلى
`channels.<id>.accounts.*`، يكون السلوك المشترك الافتراضي هو نقل القيم الخاصة
بالحساب المرقّى إلى `accounts.default`.

يمكن للقنوات المضمّنة تضييق هذا السلوك أو تجاوزه عبر سطح عقد الإعداد
الخاص بها:

- `singleAccountKeysToMove`: مفاتيح إضافية على المستوى الأعلى ينبغي نقلها إلى
  الحساب المرقّى
- `namedAccountPromotionKeys`: عندما توجد حسابات مسماة مسبقًا، تُنقل فقط هذه
  المفاتيح إلى الحساب المرقّى؛ بينما تبقى مفاتيح السياسة/التسليم المشتركة على
  جذر القناة
- `resolveSingleAccountPromotionTarget(...)`: يختار أي حساب موجود
  يستقبل القيم المرقّاة

تُعد Matrix المثال المضمّن الحالي. فإذا كان هناك حساب Matrix مسمى واحد فقط
موجود بالفعل، أو إذا كان `defaultAccount` يشير إلى مفتاح غير قانوني موجود
مثل `Ops`، فإن الترقية تحافظ على ذلك الحساب بدل إنشاء
إدخال جديد `accounts.default`.

## Config schema

يُتحقق من إعداد المكوّن الإضافي مقابل JSON Schema الموجودة في manifest لديك. ويقوم المستخدمون
بتهيئة المكونات الإضافية عبر:

```json5
{
  plugins: {
    entries: {
      "my-plugin": {
        config: {
          webhookSecret: "abc123",
        },
      },
    },
  },
}
```

يتلقى المكوّن الإضافي هذا الإعداد كـ `api.pluginConfig` أثناء التسجيل.

وبالنسبة إلى الإعداد الخاص بالقنوات، استخدم قسم إعداد القناة بدلًا من ذلك:

```json5
{
  channels: {
    "my-channel": {
      token: "bot-token",
      allowFrom: ["user1", "user2"],
    },
  },
}
```

### بناء schemas إعداد القنوات

استخدم `buildChannelConfigSchema` من `openclaw/plugin-sdk/core` لتحويل
Zod schema إلى غلاف `ChannelConfigSchema` الذي يتحقق منه OpenClaw:

```typescript
import { z } from "zod";
import { buildChannelConfigSchema } from "openclaw/plugin-sdk/core";

const accountSchema = z.object({
  token: z.string().optional(),
  allowFrom: z.array(z.string()).optional(),
  accounts: z.object({}).catchall(z.any()).optional(),
  defaultAccount: z.string().optional(),
});

const configSchema = buildChannelConfigSchema(accountSchema);
```

## معالجات الإعداد

يمكن لمكونات القنوات الإضافية توفير معالجات إعداد تفاعلية لأمر `openclaw onboard`.
ويمثل المعالج كائن `ChannelSetupWizard` على `ChannelPlugin`:

```typescript
import type { ChannelSetupWizard } from "openclaw/plugin-sdk/channel-setup";

const setupWizard: ChannelSetupWizard = {
  channel: "my-channel",
  status: {
    configuredLabel: "Connected",
    unconfiguredLabel: "Not configured",
    resolveConfigured: ({ cfg }) => Boolean((cfg.channels as any)?.["my-channel"]?.token),
  },
  credentials: [
    {
      inputKey: "token",
      providerHint: "my-channel",
      credentialLabel: "Bot token",
      preferredEnvVar: "MY_CHANNEL_BOT_TOKEN",
      envPrompt: "Use MY_CHANNEL_BOT_TOKEN from environment?",
      keepPrompt: "Keep current token?",
      inputPrompt: "Enter your bot token:",
      inspect: ({ cfg, accountId }) => {
        const token = (cfg.channels as any)?.["my-channel"]?.token;
        return {
          accountConfigured: Boolean(token),
          hasConfiguredValue: Boolean(token),
        };
      },
    },
  ],
};
```

يدعم النوع `ChannelSetupWizard` الحقول `credentials` و`textInputs`,
و`dmPolicy`، و`allowFrom`، و`groupAccess`، و`prepare`، و`finalize`، وغير ذلك.
راجع حزم المكونات الإضافية المضمنة (مثل مكوّن Discord الإضافي في `src/channel.setup.ts`) للحصول على
أمثلة كاملة.

وبالنسبة إلى مطالبات قائمة سماح DM التي تحتاج فقط إلى
تدفق `note -> prompt -> parse -> merge -> patch` القياسي، ففضّل مساعدات الإعداد
المشتركة من `openclaw/plugin-sdk/setup`: ‏`createPromptParsedAllowFromForAccount(...)`,
و`createTopLevelChannelParsedAllowFromPrompt(...)`، و
`createNestedChannelParsedAllowFromPrompt(...)`.

وبالنسبة إلى كتل حالة إعداد القناة التي تختلف فقط في التسميات، والدرجات،
والأسطر الإضافية الاختيارية، ففضّل `createStandardChannelSetupStatus(...)` من
`openclaw/plugin-sdk/setup` بدل إنشاء الكائن `status` نفسه يدويًا في
كل plugin.

وبالنسبة إلى أسطح الإعداد الاختيارية التي ينبغي أن تظهر فقط في سياقات معينة، استخدم
`createOptionalChannelSetupSurface` من `openclaw/plugin-sdk/channel-setup`:

```typescript
import { createOptionalChannelSetupSurface } from "openclaw/plugin-sdk/channel-setup";

const setupSurface = createOptionalChannelSetupSurface({
  channel: "my-channel",
  label: "My Channel",
  npmSpec: "@myorg/openclaw-my-channel",
  docsPath: "/channels/my-channel",
});
// Returns { setupAdapter, setupWizard }
```

كما يعرض `plugin-sdk/channel-setup` أيضًا البانيات الأدنى مستوى
`createOptionalChannelSetupAdapter(...)` و
`createOptionalChannelSetupWizard(...)` عندما تحتاج فقط إلى أحد نصفي
سطح التثبيت الاختياري ذاك.

تفشل المحولات/المعالجات الاختيارية المولَّدة بإغلاق آمن عند عمليات كتابة الإعداد الحقيقية.
وهي تعيد استخدام رسالة واحدة تفيد بضرورة التثبيت عبر `validateInput`,
و`applyAccountConfig`, و`finalize`، وتضيف رابط وثائق عندما يكون `docsPath`
مضبوطًا.

وبالنسبة إلى واجهات الإعداد المعتمدة على الثنائيات، ففضّل المساعدات المفوضة المشتركة بدل
نسخ منطق الثنائي/الحالة نفسه داخل كل قناة:

- `createDetectedBinaryStatus(...)` لكتل الحالة التي تختلف فقط في التسميات،
  والتلميحات، والدرجات، واكتشاف الثنائي
- `createCliPathTextInput(...)` لمدخلات النص المعتمدة على المسار
- `createDelegatedSetupWizardStatusResolvers(...)`,
  و`createDelegatedPrepare(...)`, `createDelegatedFinalize(...)`, و
  `createDelegatedResolveConfigured(...)` عندما تحتاج `setupEntry` إلى تمرير
  كسول إلى معالج كامل أثقل
- `createDelegatedTextInputShouldPrompt(...)` عندما تحتاج `setupEntry` فقط إلى
  تفويض قرار `textInputs[*].shouldPrompt`

## النشر والتثبيت

**المكونات الإضافية الخارجية:** انشر على [ClawHub](/tools/clawhub) أو npm، ثم ثبّت عبر:

```bash
openclaw plugins install @myorg/openclaw-my-plugin
```

تحاول OpenClaw استخدام ClawHub أولًا ثم تعود إلى npm تلقائيًا. ويمكنك أيضًا
فرض ClawHub صراحةً:

```bash
openclaw plugins install clawhub:@myorg/openclaw-my-plugin   # ClawHub only
```

لا يوجد تجاوز مماثل باسم `npm:`. استخدم مواصفة حزمة npm العادية عندما
تريد مسار npm بعد fallback من ClawHub:

```bash
openclaw plugins install @myorg/openclaw-my-plugin
```

**المكونات الإضافية داخل المستودع:** ضعها تحت شجرة مساحة العمل الخاصة بالمكونات الإضافية المضمّنة وسيتم
اكتشافها تلقائيًا أثناء البناء.

**يمكن للمستخدمين التثبيت عبر:**

```bash
openclaw plugins install <package-name>
```

<Info>
  بالنسبة إلى التثبيتات القادمة من npm، يشغّل `openclaw plugins install`
  الأمر `npm install --ignore-scripts` ‏(من دون lifecycle scripts). أبقِ أشجار تبعيات المكوّن الإضافي
  JS/TS خالصة وتجنب الحزم التي تتطلب بناء `postinstall`.
</Info>

## ذو صلة

- [SDK Entry Points](/plugins/sdk-entrypoints) -- `definePluginEntry` و`defineChannelPluginEntry`
- [Plugin Manifest](/plugins/manifest) -- المرجع الكامل لـ manifest schema
- [Building Plugins](/plugins/building-plugins) -- دليل البدء خطوة بخطوة
