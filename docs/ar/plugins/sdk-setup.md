---
read_when:
    - أنت تضيف معالج إعداد إلى إضافة
    - تحتاج إلى فهم الفرق بين `setup-entry.ts` و`index.ts`
    - أنت تعرّف مخططات config للإضافة أو بيانات `openclaw` الوصفية في `package.json`
sidebarTitle: Setup and Config
summary: معالجات الإعداد، و`setup-entry.ts`، ومخططات config، وبيانات `package.json` الوصفية
title: إعداد الإضافة وConfig الخاص بها
x-i18n:
    generated_at: "2026-04-06T03:11:22Z"
    model: gpt-5.4
    provider: openai
    source_hash: eac2586516d27bcd94cc4c259fe6274c792b3f9938c7ddd6dbf04a6dbb988dc9
    source_path: plugins/sdk-setup.md
    workflow: 15
---

# إعداد الإضافة وConfig الخاص بها

مرجع لتغليف الإضافات (بيانات `package.json` الوصفية)، وmanifest
(`openclaw.plugin.json`)، ومداخل الإعداد، ومخططات config.

<Tip>
  **هل تبحث عن شرح عملي؟** تغطي أدلة التنفيذ التغليف ضمن السياق:
  [إضافات القنوات](/ar/plugins/sdk-channel-plugins#step-1-package-and-manifest) و
  [إضافات المزوّدين](/ar/plugins/sdk-provider-plugins#step-1-package-and-manifest).
</Tip>

## بيانات الحزمة الوصفية

يحتاج `package.json` الخاص بك إلى حقل `openclaw` يخبر نظام الإضافات بما
توفّره إضافتك:

**إضافة قناة:**

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
      "blurb": "وصف قصير للقناة."
    }
  }
}
```

**إضافة مزوّد / خط أساس النشر على ClawHub:**

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

إذا كنت تنشر الإضافة خارجيًا على ClawHub، فإن حقلي `compat` و`build`
مطلوبان. وتوجد مقتطفات النشر القانونية في
`docs/snippets/plugin-publish/`.

### حقول `openclaw`

| الحقل        | النوع      | الوصف                                                                                             |
| ------------ | ---------- | ------------------------------------------------------------------------------------------------- |
| `extensions` | `string[]` | ملفات نقاط الإدخال (نسبةً إلى جذر الحزمة)                                                        |
| `setupEntry` | `string`   | إدخال خفيف مخصص للإعداد فقط (اختياري)                                                             |
| `channel`    | `object`   | بيانات وصفية لفهرس القنوات لأسطح الإعداد والمنتقي والبداية السريعة والحالة                         |
| `providers`  | `string[]` | معرّفات المزوّدين التي تسجلها هذه الإضافة                                                         |
| `install`    | `object`   | تلميحات التثبيت: `npmSpec` و`localPath` و`defaultChoice` و`minHostVersion` و`allowInvalidConfigRecovery` |
| `startup`    | `object`   | رايات سلوك بدء التشغيل                                                                             |

### `openclaw.channel`

يمثل `openclaw.channel` بيانات وصفية رخيصة على مستوى الحزمة لاكتشاف القنوات وأسطر الإعداد
قبل تحميل وقت التشغيل.

| الحقل                                  | النوع      | ما الذي يعنيه                                                                |
| -------------------------------------- | ---------- | ---------------------------------------------------------------------------- |
| `id`                                   | `string`   | معرّف القناة القانوني.                                                       |
| `label`                                | `string`   | التسمية الأساسية للقناة.                                                     |
| `selectionLabel`                       | `string`   | تسمية المنتقي/الإعداد عندما ينبغي أن تختلف عن `label`.                       |
| `detailLabel`                          | `string`   | تسمية تفاصيل ثانوية لفهارس القنوات الأغنى وأسطر الحالة.                     |
| `docsPath`                             | `string`   | مسار المستندات لروابط الإعداد والاختيار.                                     |
| `docsLabel`                            | `string`   | تجاوز للتسمية المستخدمة لروابط المستندات عندما ينبغي أن تختلف عن معرّف القناة. |
| `blurb`                                | `string`   | وصف قصير للفهرس/التهيئة الأولية.                                            |
| `order`                                | `number`   | ترتيب الفرز في فهارس القنوات.                                                |
| `aliases`                              | `string[]` | أسماء بديلة إضافية للبحث عن القناة عند الاختيار.                             |
| `preferOver`                           | `string[]` | معرّفات إضافات/قنوات أقل أولوية ينبغي أن تتفوق عليها هذه القناة.             |
| `systemImage`                          | `string`   | اسم icon/system-image اختياري لفهارس UI الخاصة بالقنوات.                    |
| `selectionDocsPrefix`                  | `string`   | نص بادئة قبل روابط المستندات في أسطح الاختيار.                               |
| `selectionDocsOmitLabel`               | `boolean`  | عرض مسار المستندات مباشرة بدل رابط مستندات ذي تسمية في نص الاختيار.          |
| `selectionExtras`                      | `string[]` | سلاسل قصيرة إضافية تُلحَق في نص الاختيار.                                    |
| `markdownCapable`                      | `boolean`  | يضع علامة على أن القناة تدعم Markdown لقرارات التنسيق الصادر.                |
| `exposure`                             | `object`   | عناصر تحكم في ظهور القناة في أسطح الإعداد والقوائم المهيأة والمستندات.        |
| `quickstartAllowFrom`                  | `boolean`  | يدرج هذه القناة في تدفق إعداد `allowFrom` القياسي للبداية السريعة.           |
| `forceAccountBinding`                  | `boolean`  | يطلب ربط حساب صريحًا حتى عند وجود حساب واحد فقط.                            |
| `preferSessionLookupForAnnounceTarget` | `boolean`  | يفضّل البحث في الجلسة عند حل أهداف الإعلان لهذه القناة.                      |

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
      "blurb": "تكامل دردشة مستضاف ذاتيًا قائم على webhook.",
      "order": 80,
      "aliases": ["mc"],
      "preferOver": ["my-channel-legacy"],
      "selectionDocsPrefix": "الدليل:",
      "selectionExtras": ["Markdown"],
      "markdownCapable": true,
      "exposure": {
        "configured": true,
        "setup": true,
        "docs": true
      },
      "quickstartAllowFrom": true
    }
  }
}
```

يدعم `exposure` ما يلي:

- `configured`: تضمين القناة في أسطح القوائم من نمط configured/status
- `setup`: تضمين القناة في منتقيات الإعداد/التهيئة التفاعلية
- `docs`: وضع علامة على أن القناة موجهة للعامة في أسطح المستندات/التنقل

لا يزال `showConfigured` و`showInSetup` مدعومين كأسماء بديلة قديمة. ويفضَّل
استخدام `exposure`.

### `openclaw.install`

يمثل `openclaw.install` بيانات وصفية على مستوى الحزمة، وليس بيانات manifest وصفية.

| الحقل                        | النوع                 | ما الذي يعنيه                                                                      |
| ---------------------------- | -------------------- | ---------------------------------------------------------------------------------- |
| `npmSpec`                    | `string`             | مواصفة npm القانونية لتدفقات التثبيت/التحديث.                                     |
| `localPath`                  | `string`             | مسار تثبيت محلي للتطوير أو للإضافات المجمعة.                                       |
| `defaultChoice`              | `"npm"` \| `"local"` | مصدر التثبيت المفضل عند توفر الاثنين.                                              |
| `minHostVersion`             | `string`             | الحد الأدنى لإصدار OpenClaw المدعوم بصيغة `>=x.y.z`.                              |
| `allowInvalidConfigRecovery` | `boolean`            | يتيح لتدفقات إعادة تثبيت الإضافات المجمعة التعافي من إخفاقات config قديمة محددة. |

إذا تم تعيين `minHostVersion`، فإن التثبيت وتحميل سجل manifest يفرضان
هذا الحقل كليهما. وتتجاوز المضيفات الأقدم الإضافة؛ أما سلاسل الإصدارات غير الصالحة فتُرفض.

إن `allowInvalidConfigRecovery` ليس تجاوزًا عامًا لإعدادات config المكسورة. بل هو
للتعافي الضيق الخاص بالإضافات المجمعة فقط، حتى تتمكن إعادة التثبيت/الإعداد من إصلاح
مخلّفات الترقيات المعروفة مثل مسار إضافة مجمعة مفقود أو إدخال `channels.<id>`
قديم لتلك الإضافة نفسها. وإذا كان config مكسورًا لأسباب غير مرتبطة، فلا يزال
التثبيت يفشل بشكل مغلق ويوجه المشغّل إلى تشغيل `openclaw doctor --fix`.

### تأجيل التحميل الكامل

يمكن لإضافات القنوات الاشتراك في التحميل المؤجل باستخدام:

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

عند التمكين، يحمّل OpenClaw `setupEntry` فقط أثناء مرحلة بدء التشغيل
التي تسبق الاستماع، حتى للقنوات المهيأة بالفعل. ويُحمّل الإدخال الكامل بعد أن
تبدأ البوابة في الاستماع.

<Warning>
  لا تفعّل التحميل المؤجل إلا عندما يسجل `setupEntry` الخاص بك كل ما
  تحتاجه البوابة قبل أن تبدأ الاستماع (تسجيل القناة، ومسارات HTTP، وطرائق البوابة). وإذا كان الإدخال الكامل يملك إمكانات بدء تشغيل مطلوبة، فأبقِ
  السلوك الافتراضي.
</Warning>

إذا كان إدخال الإعداد/الإدخال الكامل لديك يسجل طرائق Gateway RPC، فأبقِها على
بادئة خاصة بالإضافة. وتظل مساحات أسماء الإدارة الأساسية المحجوزة (`config.*`،
`exec.approvals.*`، `wizard.*`، `update.*`) مملوكة للنواة وتُحل دائمًا
إلى `operator.admin`.

## Plugin manifest

يجب أن تُرفق كل إضافة أصلية ملف `openclaw.plugin.json` في جذر الحزمة.
ويستخدم OpenClaw هذا للتحقق من config من دون تنفيذ شيفرة الإضافة.

```json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "description": "يضيف إمكانات My Plugin إلى OpenClaw",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "webhookSecret": {
        "type": "string",
        "description": "سر التحقق من webhook"
      }
    }
  }
}
```

بالنسبة إلى إضافات القنوات، أضف `kind` و`channels`:

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

حتى الإضافات التي لا تحتوي على config يجب أن تُرفق schema. ويُعد المخطط الفارغ صالحًا:

```json
{
  "id": "my-plugin",
  "configSchema": {
    "type": "object",
    "additionalProperties": false
  }
}
```

راجع [Plugin Manifest](/ar/plugins/manifest) للاطلاع على المرجع الكامل للمخطط.

## النشر على ClawHub

بالنسبة إلى حزم الإضافات، استخدم أمر ClawHub الخاص بالحزمة:

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

الاسم البديل القديم للنشر الخاص بالـ Skill فقط مخصص لـ Skills. ويجب أن تستخدم حزم الإضافات
دائمًا `clawhub package publish`.

## إدخال الإعداد

ملف `setup-entry.ts` هو بديل خفيف لـ `index.ts`
يحمّله OpenClaw عندما يحتاج فقط إلى أسطح الإعداد (التهيئة الأولية، وإصلاح config، وفحص القنوات المعطلة).

```typescript
// setup-entry.ts
import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
import { myChannelPlugin } from "./src/channel.js";

export default defineSetupPluginEntry(myChannelPlugin);
```

وهذا يتجنب تحميل شيفرة وقت تشغيل ثقيلة (مكتبات crypto، وتسجيلات CLI،
والخدمات الخلفية) أثناء تدفقات الإعداد.

**متى يستخدم OpenClaw `setupEntry` بدل الإدخال الكامل:**

- تكون القناة معطلة لكنها تحتاج إلى أسطح إعداد/تهيئة أولية
- تكون القناة مفعلة لكنها غير مهيأة
- يكون التحميل المؤجل مفعّلًا (`deferConfiguredChannelFullLoadUntilAfterListen`)

**ما الذي يجب أن يسجله `setupEntry`:**

- كائن إضافة القناة (عبر `defineSetupPluginEntry`)
- أي مسارات HTTP مطلوبة قبل استماع البوابة
- أي طرائق للبوابة مطلوبة أثناء بدء التشغيل

ويجب أن تتجنب طرائق البوابة الخاصة ببدء التشغيل تلك أيضًا مساحات أسماء الإدارة
الأساسية المحجوزة مثل `config.*` أو `update.*`.

**ما الذي يجب ألا يتضمنه `setupEntry`:**

- تسجيلات CLI
- الخدمات الخلفية
- واردات وقت تشغيل ثقيلة (crypto وSDKs)
- طرائق البوابة المطلوبة فقط بعد بدء التشغيل

### واردات مساعدات الإعداد الضيقة

بالنسبة إلى المسارات الساخنة الخاصة بالإعداد فقط، فَضّل واجهات المساعدات الضيقة هذه بدل الواجهة الأوسع
`plugin-sdk/setup` عندما لا تحتاج إلا إلى جزء من سطح الإعداد:

| مسار الاستيراد                      | استخدمه من أجل                                                                      | الصادرات الأساسية                                                                                                                                                                                                                                                                              |
| ----------------------------------- | ------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/setup-runtime`          | مساعدات وقت التشغيل الخاصة بالإعداد التي تبقى متاحة في `setupEntry` / بدء القناة المؤجل | `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
| `plugin-sdk/setup-adapter-runtime`  | محوّلات إعداد الحساب الواعية بالبيئة                                                | `createEnvPatchedAccountSetupAdapter`                                                                                                                                                                                                                                                          |
| `plugin-sdk/setup-tools`            | مساعدات CLI/التثبيت/الأرشيف/المستندات الخاصة بالإعداد                               | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR`                                                                                                                                                                                |

استخدم الواجهة الأوسع `plugin-sdk/setup` عندما تريد صندوق أدوات الإعداد
المشترك الكامل، بما في ذلك مساعدات تصحيح config مثل
`moveSingleAccountChannelSectionToDefaultAccount(...)`.

وتبقى محولات تصحيح الإعداد آمنة في المسار الساخن عند الاستيراد. ويكون
البحث عن سطح العقد الخاص بترقية الحساب الواحد المجمّع كسولًا، لذلك لا يؤدي استيراد
`plugin-sdk/setup-runtime` إلى تحميل اكتشاف سطح العقد المجمّع مبكرًا قبل استخدام
المحول فعليًا.

### ترقية الحساب الواحد المملوكة للقناة

عندما تنتقل القناة من config ذي مستوى أعلى لحساب واحد إلى
`channels.<id>.accounts.*`، يكون السلوك المشترك الافتراضي هو نقل القيم
التي تقع ضمن نطاق الحساب المرقّى إلى `accounts.default`.

يمكن للقنوات المجمعة تضييق هذه الترقية أو تجاوزها عبر سطح عقد الإعداد
الخاص بها:

- `singleAccountKeysToMove`: مفاتيح إضافية ذات مستوى أعلى ينبغي نقلها إلى
  الحساب المرقّى
- `namedAccountPromotionKeys`: عندما تكون الحسابات المسماة موجودة بالفعل، لا تُنقل إلا هذه
  المفاتيح إلى الحساب المرقّى؛ بينما تبقى مفاتيح السياسة/التسليم المشتركة في
  جذر القناة
- `resolveSingleAccountPromotionTarget(...)`: اختيار الحساب الموجود الذي
  يستقبل القيم المرقّاة

تمثل Matrix المثال المجمّع الحالي. وإذا كان هناك حساب Matrix مسمّى واحد موجود
بالفعل، أو إذا كان `defaultAccount` يشير إلى مفتاح غير قانوني موجود
مثل `Ops`، فإن الترقية تحافظ على ذلك الحساب بدل إنشاء إدخال جديد
`accounts.default`.

## مخطط config

يُتحقق من config الإضافة مقابل JSON Schema الموجود في manifest الخاص بك. ويهيّئ المستخدمون
الإضافات عبر:

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

تتلقى إضافتك هذا config على هيئة `api.pluginConfig` أثناء التسجيل.

أما بالنسبة إلى config الخاص بالقناة، فاستخدم قسم config الخاص بالقناة بدلًا من ذلك:

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

### بناء مخططات config القنوات

استخدم `buildChannelConfigSchema` من `openclaw/plugin-sdk/core` لتحويل
مخطط Zod إلى الغلاف `ChannelConfigSchema` الذي يتحقق منه OpenClaw:

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

يمكن لإضافات القنوات توفير معالجات إعداد تفاعلية لـ `openclaw onboard`.
ويكون المعالج عبارة عن كائن `ChannelSetupWizard` على `ChannelPlugin`:

```typescript
import type { ChannelSetupWizard } from "openclaw/plugin-sdk/channel-setup";

const setupWizard: ChannelSetupWizard = {
  channel: "my-channel",
  status: {
    configuredLabel: "متصل",
    unconfiguredLabel: "غير مهيأ",
    resolveConfigured: ({ cfg }) => Boolean((cfg.channels as any)?.["my-channel"]?.token),
  },
  credentials: [
    {
      inputKey: "token",
      providerHint: "my-channel",
      credentialLabel: "token البوت",
      preferredEnvVar: "MY_CHANNEL_BOT_TOKEN",
      envPrompt: "هل تريد استخدام MY_CHANNEL_BOT_TOKEN من البيئة؟",
      keepPrompt: "هل تريد الاحتفاظ بالـ token الحالي؟",
      inputPrompt: "أدخل token البوت:",
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

يدعم النوع `ChannelSetupWizard` الحقول `credentials` و`textInputs` و
`dmPolicy` و`allowFrom` و`groupAccess` و`prepare` و`finalize` وغيرها.
راجع حزم الإضافات المجمعة (مثلًا إضافة Discord في `src/channel.setup.ts`) للحصول على
أمثلة كاملة.

بالنسبة إلى مطالبات قائمة السماح في الرسائل المباشرة التي تحتاج فقط إلى التدفق القياسي
`note -> prompt -> parse -> merge -> patch`، فَضّل مساعدات الإعداد
المشتركة من `openclaw/plugin-sdk/setup`: وهي `createPromptParsedAllowFromForAccount(...)`،
و`createTopLevelChannelParsedAllowFromPrompt(...)`، و
`createNestedChannelParsedAllowFromPrompt(...)`.

أما لكتل حالة إعداد القناة التي تختلف فقط في التسميات، والدرجات، والأسطر
الإضافية الاختيارية، فَضّل `createStandardChannelSetupStatus(...)` من
`openclaw/plugin-sdk/setup` بدل بناء كائن `status` نفسه يدويًا
في كل إضافة.

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

يكشف `plugin-sdk/channel-setup` أيضًا عن البانيين الأدنى مستوى
`createOptionalChannelSetupAdapter(...)` و
`createOptionalChannelSetupWizard(...)` عندما تحتاج إلى نصف واحد فقط من
سطح التثبيت الاختياري هذا.

يفشل المحول/المعالج الاختياريان المولدان بشكل مغلق عند عمليات كتابة config الحقيقية. كما
يعيدان استخدام رسالة واحدة تفيد بأن التثبيت مطلوب عبر `validateInput` و
`applyAccountConfig` و`finalize`، ويُلحقان رابطًا للمستندات عندما يكون `docsPath`
مضبوطًا.

أما بالنسبة إلى واجهات إعداد UI المدعومة بملفات تنفيذية، فَضّل المساعدات المشتركة المفوّضة بدل
نسخ نفس منطق الملف التنفيذي/الحالة في كل قناة:

- `createDetectedBinaryStatus(...)` لكتل الحالة التي تختلف فقط بحسب التسميات،
  والتلميحات، والدرجات، واكتشاف الملف التنفيذي
- `createCliPathTextInput(...)` لمدخلات النص المدعومة بالمسار
- `createDelegatedSetupWizardStatusResolvers(...)`،
  و`createDelegatedPrepare(...)`، و`createDelegatedFinalize(...)`، و
  `createDelegatedResolveConfigured(...)` عندما يحتاج `setupEntry` إلى الإحالة
  كسولًا إلى معالج أثقل وكامل
- `createDelegatedTextInputShouldPrompt(...)` عندما يحتاج `setupEntry` فقط إلى
  تفويض قرار `textInputs[*].shouldPrompt`

## النشر والتثبيت

**الإضافات الخارجية:** انشرها على [ClawHub](/ar/tools/clawhub) أو npm، ثم ثبّتها:

```bash
openclaw plugins install @myorg/openclaw-my-plugin
```

يحاول OpenClaw استخدام ClawHub أولًا ثم يرجع إلى npm تلقائيًا. ويمكنك أيضًا
فرض ClawHub صراحةً:

```bash
openclaw plugins install clawhub:@myorg/openclaw-my-plugin   # ClawHub فقط
```

لا يوجد تجاوز مماثل لـ `npm:`. استخدم مواصفة حزمة npm العادية عندما
تريد مسار npm بعد الرجوع من ClawHub:

```bash
openclaw plugins install @myorg/openclaw-my-plugin
```

**الإضافات داخل المستودع:** ضعها تحت شجرة مساحة عمل الإضافات المجمعة وسيتم
اكتشافها تلقائيًا أثناء البناء.

**يمكن للمستخدمين التثبيت:**

```bash
openclaw plugins install <package-name>
```

<Info>
  بالنسبة إلى التثبيتات القادمة من npm، يشغّل `openclaw plugins install`
  الأمر `npm install --ignore-scripts` (من دون lifecycle scripts). أبقِ شجرة اعتماديات الإضافة
  JavaScript/TypeScript خالصة وتجنب الحزم التي تتطلب بناء `postinstall`.
</Info>

## ذو صلة

- [نقاط إدخال SDK](/ar/plugins/sdk-entrypoints) -- `definePluginEntry` و`defineChannelPluginEntry`
- [Plugin Manifest](/ar/plugins/manifest) -- المرجع الكامل لمخطط manifest
- [بناء الإضافات](/ar/plugins/building-plugins) -- دليل بدء خطوة بخطوة
