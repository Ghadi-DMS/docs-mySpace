---
read_when:
    - تحتاج إلى التوقيع النوعي الدقيق لـ `definePluginEntry` أو `defineChannelPluginEntry`
    - تريد فهم وضع التسجيل (كامل مقابل إعداد فقط مقابل بيانات CLI الوصفية)
    - تبحث عن خيارات نقطة الدخول
sidebarTitle: Entry Points
summary: مرجع `definePluginEntry` و`defineChannelPluginEntry` و`defineSetupPluginEntry`
title: نقاط دخول المكونات الإضافية
x-i18n:
    generated_at: "2026-04-05T12:51:23Z"
    model: gpt-5.4
    provider: openai
    source_hash: 799dbfe71e681dd8ba929a7a631dfe745c3c5c69530126fea2f9c137b120f51f
    source_path: plugins/sdk-entrypoints.md
    workflow: 15
---

# نقاط دخول المكونات الإضافية

يصدّر كل مكوّن إضافي كائن دخول افتراضيًا. ويوفر SDK ثلاث أدوات مساعدة
لإنشائها.

<Tip>
  **هل تبحث عن شرح عملي؟** راجع [Channel Plugins](/plugins/sdk-channel-plugins)
  أو [Provider Plugins](/plugins/sdk-provider-plugins) للحصول على أدلة خطوة بخطوة.
</Tip>

## `definePluginEntry`

**الاستيراد:** `openclaw/plugin-sdk/plugin-entry`

لمكونات provider الإضافية، ومكونات الأدوات الإضافية، ومكونات hooks الإضافية، وأي شيء **ليس**
قناة مراسلة.

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

export default definePluginEntry({
  id: "my-plugin",
  name: "My Plugin",
  description: "Short summary",
  register(api) {
    api.registerProvider({
      /* ... */
    });
    api.registerTool({
      /* ... */
    });
  },
});
```

| الحقل          | النوع                                                            | مطلوب | الافتراضي            |
| -------------- | ---------------------------------------------------------------- | ------ | -------------------- |
| `id`           | `string`                                                         | نعم    | —                    |
| `name`         | `string`                                                         | نعم    | —                    |
| `description`  | `string`                                                         | نعم    | —                    |
| `kind`         | `string`                                                         | لا     | —                    |
| `configSchema` | `OpenClawPluginConfigSchema \| () => OpenClawPluginConfigSchema` | لا     | schema كائن فارغة    |
| `register`     | `(api: OpenClawPluginApi) => void`                               | نعم    | —                    |

- يجب أن يطابق `id` ملف manifest الخاص بك في `openclaw.plugin.json`.
- يُستخدم `kind` للفتحات الحصرية: `"memory"` أو `"context-engine"`.
- يمكن أن تكون `configSchema` دالة للتقييم الكسول.
- يحل OpenClaw هذه الـ schema ويخزنها مؤقتًا عند أول وصول، لذلك لا تعمل
  أدوات بناء schema المكلفة إلا مرة واحدة.

## `defineChannelPluginEntry`

**الاستيراد:** `openclaw/plugin-sdk/channel-core`

يغلف `definePluginEntry` بربط خاص بالقنوات. ويستدعي تلقائيًا
`api.registerChannel({ plugin })`، ويعرض نقطة وصل اختيارية لبيانات CLI الوصفية الخاصة بمساعدة الجذر،
ويقيّد `registerFull` بحسب وضع التسجيل.

```typescript
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";

export default defineChannelPluginEntry({
  id: "my-channel",
  name: "My Channel",
  description: "Short summary",
  plugin: myChannelPlugin,
  setRuntime: setMyRuntime,
  registerCliMetadata(api) {
    api.registerCli(/* ... */);
  },
  registerFull(api) {
    api.registerGatewayMethod(/* ... */);
  },
});
```

| الحقل                 | النوع                                                            | مطلوب | الافتراضي            |
| --------------------- | ---------------------------------------------------------------- | ------ | -------------------- |
| `id`                  | `string`                                                         | نعم    | —                    |
| `name`                | `string`                                                         | نعم    | —                    |
| `description`         | `string`                                                         | نعم    | —                    |
| `plugin`              | `ChannelPlugin`                                                  | نعم    | —                    |
| `configSchema`        | `OpenClawPluginConfigSchema \| () => OpenClawPluginConfigSchema` | لا     | schema كائن فارغة    |
| `setRuntime`          | `(runtime: PluginRuntime) => void`                               | لا     | —                    |
| `registerCliMetadata` | `(api: OpenClawPluginApi) => void`                               | لا     | —                    |
| `registerFull`        | `(api: OpenClawPluginApi) => void`                               | لا     | —                    |

- يُستدعى `setRuntime` أثناء التسجيل حتى تتمكن من تخزين مرجع وقت التشغيل
  (عادةً عبر `createPluginRuntimeStore`). ويُتخطى أثناء
  التقاط بيانات CLI الوصفية.
- تعمل `registerCliMetadata` خلال كل من `api.registrationMode === "cli-metadata"`
  و`api.registrationMode === "full"`.
  استخدمها بوصفها المكان الرسمي لوصافات CLI المملوكة للقناة حتى تبقى
  المساعدة الجذرية غير مفعِّلة بينما يظل تسجيل أوامر CLI العادي متوافقًا
  مع تحميلات المكونات الإضافية الكاملة.
- لا تعمل `registerFull` إلا عندما يكون `api.registrationMode === "full"`. وتُتخطى
  أثناء التحميل الخاص بالإعداد فقط.
- مثل `definePluginEntry`، يمكن أن تكون `configSchema` مصنعًا كسولًا ويقوم OpenClaw
  بتخزين schema المحلولة مؤقتًا عند أول وصول.
- بالنسبة إلى أوامر CLI الجذرية المملوكة للمكون الإضافي، فضّل `api.registerCli(..., { descriptors: [...] })`
  عندما تريد أن يبقى الأمر محمَّلًا كسولًا من دون أن يختفي من
  شجرة تحليل CLI الجذرية. وبالنسبة إلى مكونات القنوات الإضافية، فضّل تسجيل هذه الواصفات
  من `registerCliMetadata(...)` وأبقِ `registerFull(...)` مركزة على العمل الخاص بوقت التشغيل فقط.
- إذا كانت `registerFull(...)` تسجل أيضًا طرق Gateway RPC، فأبقها ضمن
  بادئة خاصة بالمكون الإضافي. أما مساحات أسماء الإدارة الأساسية المحجوزة (`config.*`,
  `exec.approvals.*`, `wizard.*`, `update.*`) فستُحوَّل دائمًا إلى
  `operator.admin`.

## `defineSetupPluginEntry`

**الاستيراد:** `openclaw/plugin-sdk/channel-core`

مخصص لملف `setup-entry.ts` الخفيف. ويعيد فقط `{ plugin }` من دون
ربط وقت تشغيل أو CLI.

```typescript
import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";

export default defineSetupPluginEntry(myChannelPlugin);
```

يحمّل OpenClaw هذا بدل نقطة الدخول الكاملة عندما تكون قناة معطلة،
أو غير مهيأة، أو عندما يكون التحميل المؤجل مفعّلًا. راجع
[Setup and Config](/plugins/sdk-setup#setup-entry) لمعرفة متى يكون هذا مهمًا.

عمليًا، اربط `defineSetupPluginEntry(...)` بعائلات مساعدات الإعداد
الضيقة التالية:

- `openclaw/plugin-sdk/setup-runtime` لمساعدات الإعداد الآمنة وقت التشغيل مثل
  محولات setup patch الآمنة للاستيراد، ومخرجات lookup-note،
  و`promptResolvedAllowFrom`، و`splitSetupEntries`، ووكلاء الإعداد المفوضين
- `openclaw/plugin-sdk/channel-setup` لأسطح الإعداد الخاصة بالتثبيت الاختياري
- `openclaw/plugin-sdk/setup-tools` لمساعدات CLI/الأرشفة/الوثائق الخاصة بالإعداد/التثبيت

أبقِ SDKs الثقيلة، وتسجيل CLI، وخدمات وقت التشغيل طويلة العمر في
نقطة الدخول الكاملة.

## وضع التسجيل

يخبرك `api.registrationMode` كيف جرى تحميل المكوّن الإضافي:

| الوضع             | متى                               | ما الذي يجب تسجيله                                                                      |
| ----------------- | --------------------------------- | ---------------------------------------------------------------------------------------- |
| `"full"`          | بدء تشغيل gateway العادي          | كل شيء                                                                                   |
| `"setup-only"`    | قناة معطلة/غير مهيأة             | تسجيل القناة فقط                                                                         |
| `"setup-runtime"` | تدفق إعداد مع توفر وقت التشغيل    | تسجيل القناة بالإضافة إلى وقت التشغيل الخفيف فقط المطلوب قبل تحميل نقطة الدخول الكاملة |
| `"cli-metadata"`  | التقاط مساعدة الجذر / بيانات CLI الوصفية | واصفات CLI فقط                                                                     |

يتولى `defineChannelPluginEntry` هذا التقسيم تلقائيًا. وإذا استخدمت
`definePluginEntry` مباشرة لقناة، فتحقق من الوضع بنفسك:

```typescript
register(api) {
  if (api.registrationMode === "cli-metadata" || api.registrationMode === "full") {
    api.registerCli(/* ... */);
    if (api.registrationMode === "cli-metadata") return;
  }

  api.registerChannel({ plugin: myPlugin });
  if (api.registrationMode !== "full") return;

  // Heavy runtime-only registrations
  api.registerService(/* ... */);
}
```

تعامل مع `"setup-runtime"` باعتباره النافذة التي يجب أن توجد فيها
أسطح بدء التشغيل الخاصة بالإعداد فقط من دون إعادة الدخول إلى وقت تشغيل القناة
المجمعة الكامل. ومن الملائم هنا تسجيل القنوات، ومسارات HTTP الآمنة للإعداد،
وطرق gateway الآمنة للإعداد، ومساعدات الإعداد المفوضة. أما الخدمات الخلفية الثقيلة،
ومسجلات CLI، وتهيئة SDKs الخاصة بالمزوّد/العميل فما تزال تنتمي إلى `"full"`.

وبالنسبة إلى مسجلات CLI تحديدًا:

- استخدم `descriptors` عندما يملك المسجل أمرًا جذريًا واحدًا أو أكثر وتريد
  من OpenClaw تحميل وحدة CLI الفعلية بكسل عند أول استدعاء
- تأكد من أن هذه الواصفات تغطي كل جذر أمر من المستوى الأعلى يعرضه
  المسجل
- استخدم `commands` وحدها فقط لمسارات التوافق المتحمسة

## أشكال المكونات الإضافية

يصنف OpenClaw المكونات الإضافية المحمّلة بحسب سلوك التسجيل الخاص بها:

| الشكل                 | الوصف                                               |
| --------------------- | --------------------------------------------------- |
| **plain-capability**  | نوع إمكانة واحد (مثل مزوّد فقط)                     |
| **hybrid-capability** | عدة أنواع من الإمكانات (مثل مزوّد + speech)         |
| **hook-only**         | hooks فقط، من دون إمكانات                           |
| **non-capability**    | أدوات/أوامر/خدمات لكن من دون إمكانات                |

استخدم `openclaw plugins inspect <id>` لرؤية شكل المكوّن الإضافي.

## ذو صلة

- [SDK Overview](/plugins/sdk-overview) — واجهة API الخاصة بالتسجيل ومرجع subpath
- [Runtime Helpers](/plugins/sdk-runtime) — `api.runtime` و`createPluginRuntimeStore`
- [Setup and Config](/plugins/sdk-setup) — manifest، ونقطة دخول الإعداد، والتحميل المؤجل
- [Channel Plugins](/plugins/sdk-channel-plugins) — بناء كائن `ChannelPlugin`
- [Provider Plugins](/plugins/sdk-provider-plugins) — تسجيل المزوّد وhooks
