---
read_when:
    - تريد إنشاء إضافة OpenClaw جديدة
    - تحتاج إلى بداية سريعة لتطوير الإضافات
    - أنت تضيف قناة أو موفرًا أو أداة أو قدرة أخرى جديدة إلى OpenClaw
sidebarTitle: Getting Started
summary: أنشئ إضافة OpenClaw الأولى الخاصة بك في دقائق
title: بناء الإضافات
x-i18n:
    generated_at: "2026-04-06T03:09:34Z"
    model: gpt-5.4
    provider: openai
    source_hash: 9be344cb300ecbcba08e593a95bcc93ab16c14b28a0ff0c29b26b79d8249146c
    source_path: plugins/building-plugins.md
    workflow: 15
---

# بناء الإضافات

توسّع الإضافات OpenClaw بإمكانات جديدة: القنوات، وموفرو النماذج،
والكلام، والنسخ الفوري، والصوت الفوري، وفهم الوسائط، وتوليد
الصور، وتوليد الفيديو، وجلب الويب، والبحث في الويب، وأدوات الوكيل، أو أي
مزيج منها.

لا تحتاج إلى إضافة إضافتك إلى مستودع OpenClaw. انشرها إلى
[ClawHub](/ar/tools/clawhub) أو npm وسيقوم المستخدمون بتثبيتها باستخدام
`openclaw plugins install <package-name>`. يحاول OpenClaw استخدام ClawHub أولًا ثم
يعود إلى npm تلقائيًا.

## المتطلبات المسبقة

- Node >= 22 ومدير حزم (npm أو pnpm)
- إلمام بـ TypeScript ‏(ESM)
- للإضافات داخل المستودع: يجب استنساخ المستودع وتشغيل `pnpm install`

## ما نوع الإضافة؟

<CardGroup cols={3}>
  <Card title="إضافة قناة" icon="messages-square" href="/ar/plugins/sdk-channel-plugins">
    اربط OpenClaw بمنصة مراسلة (Discord أو IRC أو غير ذلك)
  </Card>
  <Card title="إضافة موفر" icon="cpu" href="/ar/plugins/sdk-provider-plugins">
    أضف موفر نماذج (LLM أو proxy أو نقطة نهاية مخصصة)
  </Card>
  <Card title="إضافة أداة / hook" icon="wrench">
    سجّل أدوات الوكيل أو event hooks أو الخدمات — تابع أدناه
  </Card>
</CardGroup>

إذا كانت إضافة القناة اختيارية وقد لا تكون مثبّتة عند تشغيل الإعداد/التهيئة
فاستخدم `createOptionalChannelSetupSurface(...)` من
`openclaw/plugin-sdk/channel-setup`. فهي تنتج مُكيّف إعداد + زوج معالج
يعرض متطلب التثبيت ويفشل بشكل مغلق عند عمليات كتابة التكوين الفعلية
إلى أن يتم تثبيت الإضافة.

## بداية سريعة: إضافة أداة

ينشئ هذا الشرح إضافة دنيا تسجّل أداة وكيل. تحتوي إضافات القنوات
والموفرين على أدلة مخصصة مرتبطة أعلاه.

<Steps>
  <Step title="أنشئ الحزمة وmanifest">
    <CodeGroup>
    ```json package.json
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

    ```json openclaw.plugin.json
    {
      "id": "my-plugin",
      "name": "My Plugin",
      "description": "Adds a custom tool to OpenClaw",
      "configSchema": {
        "type": "object",
        "additionalProperties": false
      }
    }
    ```
    </CodeGroup>

    تحتاج كل إضافة إلى manifest، حتى من دون أي تكوين. راجع
    [Manifest](/ar/plugins/manifest) للاطلاع على المخطط الكامل. توجد مقتطفات نشر ClawHub
    الرسمية في `docs/snippets/plugin-publish/`.

  </Step>

  <Step title="اكتب نقطة الإدخال">

    ```typescript
    // index.ts
    import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
    import { Type } from "@sinclair/typebox";

    export default definePluginEntry({
      id: "my-plugin",
      name: "My Plugin",
      description: "Adds a custom tool to OpenClaw",
      register(api) {
        api.registerTool({
          name: "my_tool",
          description: "Do a thing",
          parameters: Type.Object({ input: Type.String() }),
          async execute(_id, params) {
            return { content: [{ type: "text", text: `Got: ${params.input}` }] };
          },
        });
      },
    });
    ```

    يُستخدم `definePluginEntry` للإضافات غير الخاصة بالقنوات. أما القنوات فاستخدم
    `defineChannelPluginEntry` — راجع [Channel Plugins](/ar/plugins/sdk-channel-plugins).
    وللاطلاع على خيارات نقطة الإدخال الكاملة، راجع [Entry Points](/ar/plugins/sdk-entrypoints).

  </Step>

  <Step title="اختبر وانشر">

    **الإضافات الخارجية:** تحقّق وانشر باستخدام ClawHub، ثم ثبّت:

    ```bash
    clawhub package publish your-org/your-plugin --dry-run
    clawhub package publish your-org/your-plugin
    openclaw plugins install clawhub:@myorg/openclaw-my-plugin
    ```

    يتحقق OpenClaw أيضًا من ClawHub قبل npm عند استخدام مواصفات حزم مجردة مثل
    `@myorg/openclaw-my-plugin`.

    **الإضافات داخل المستودع:** ضعها تحت شجرة مساحة عمل الإضافات المجمعة — وسيتم اكتشافها تلقائيًا.

    ```bash
    pnpm test -- <bundled-plugin-root>/my-plugin/
    ```

  </Step>
</Steps>

## قدرات الإضافة

يمكن لإضافة واحدة تسجيل أي عدد من القدرات عبر الكائن `api`:

| القدرة                 | طريقة التسجيل                                  | الدليل المفصل                                                                  |
| ---------------------- | ----------------------------------------------- | ------------------------------------------------------------------------------ |
| استدلال النص (LLM)     | `api.registerProvider(...)`                     | [Provider Plugins](/ar/plugins/sdk-provider-plugins)                              |
| القناة / المراسلة      | `api.registerChannel(...)`                      | [Channel Plugins](/ar/plugins/sdk-channel-plugins)                                |
| الكلام (TTS/STT)       | `api.registerSpeechProvider(...)`               | [Provider Plugins](/ar/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| النسخ الفوري           | `api.registerRealtimeTranscriptionProvider(...)` | [Provider Plugins](/ar/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| الصوت الفوري           | `api.registerRealtimeVoiceProvider(...)`        | [Provider Plugins](/ar/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| فهم الوسائط            | `api.registerMediaUnderstandingProvider(...)`   | [Provider Plugins](/ar/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| توليد الصور            | `api.registerImageGenerationProvider(...)`      | [Provider Plugins](/ar/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| توليد الموسيقى         | `api.registerMusicGenerationProvider(...)`      | [Provider Plugins](/ar/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| توليد الفيديو          | `api.registerVideoGenerationProvider(...)`      | [Provider Plugins](/ar/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| جلب الويب              | `api.registerWebFetchProvider(...)`             | [Provider Plugins](/ar/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| البحث في الويب         | `api.registerWebSearchProvider(...)`            | [Provider Plugins](/ar/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| أدوات الوكيل           | `api.registerTool(...)`                         | أدناه                                                                          |
| أوامر مخصصة            | `api.registerCommand(...)`                      | [Entry Points](/ar/plugins/sdk-entrypoints)                                       |
| event hooks            | `api.registerHook(...)`                         | [Entry Points](/ar/plugins/sdk-entrypoints)                                       |
| مسارات HTTP            | `api.registerHttpRoute(...)`                    | [Internals](/ar/plugins/architecture#gateway-http-routes)                         |
| أوامر CLI فرعية        | `api.registerCli(...)`                          | [Entry Points](/ar/plugins/sdk-entrypoints)                                       |

للاطلاع على API التسجيل الكامل، راجع [SDK Overview](/ar/plugins/sdk-overview#registration-api).

إذا كانت إضافتك تسجّل طرق RPC مخصصة للبوابة، فاحتفظ بها ضمن
بادئة خاصة بالإضافة. تظل مساحات أسماء الإدارة الأساسية (`config.*`،
`exec.approvals.*`، `wizard.*`، `update.*`) محجوزة وتُحل دائمًا إلى
`operator.admin`، حتى إذا طلبت إضافة نطاقًا أضيق.

دلالات حواجز hook التي يجب أخذها في الاعتبار:

- `before_tool_call`: تكون `{ block: true }` نهائية وتوقف المعالجات ذات الأولوية الأدنى.
- `before_tool_call`: تُعامل `{ block: false }` على أنها لا قرار.
- `before_tool_call`: تؤدي `{ requireApproval: true }` إلى إيقاف تنفيذ الوكيل مؤقتًا وتطلب من المستخدم الموافقة عبر تراكب موافقات exec أو أزرار Telegram أو تفاعلات Discord أو الأمر `/approve` على أي قناة.
- `before_install`: تكون `{ block: true }` نهائية وتوقف المعالجات ذات الأولوية الأدنى.
- `before_install`: تُعامل `{ block: false }` على أنها لا قرار.
- `message_sending`: تكون `{ cancel: true }` نهائية وتوقف المعالجات ذات الأولوية الأدنى.
- `message_sending`: تُعامل `{ cancel: false }` على أنها لا قرار.

يعالج الأمر `/approve` كلًا من موافقات exec وموافقات الإضافات مع رجوع محدود: عندما لا يتم العثور على معرّف موافقة exec، يعيد OpenClaw محاولة المعرّف نفسه عبر موافقات الإضافات. ويمكن تكوين إعادة توجيه موافقات الإضافات بشكل مستقل عبر `approvals.plugin` في التكوين.

إذا كانت بنية الموافقات المخصصة تحتاج إلى اكتشاف حالة الرجوع المحدود نفسها،
ففضّل استخدام `isApprovalNotFoundError` من `openclaw/plugin-sdk/error-runtime`
بدلًا من مطابقة سلاسل انتهاء صلاحية الموافقة يدويًا.

راجع [SDK Overview hook decision semantics](/ar/plugins/sdk-overview#hook-decision-semantics) لمزيد من التفاصيل.

## تسجيل أدوات الوكيل

الأدوات هي دوال typed يمكن لـ LLM استدعاؤها. ويمكن أن تكون مطلوبة (متاحة
دائمًا) أو اختيارية (يشترك فيها المستخدم):

```typescript
register(api) {
  // أداة مطلوبة — متاحة دائمًا
  api.registerTool({
    name: "my_tool",
    description: "Do a thing",
    parameters: Type.Object({ input: Type.String() }),
    async execute(_id, params) {
      return { content: [{ type: "text", text: params.input }] };
    },
  });

  // أداة اختيارية — يجب على المستخدم إضافتها إلى allowlist
  api.registerTool(
    {
      name: "workflow_tool",
      description: "Run a workflow",
      parameters: Type.Object({ pipeline: Type.String() }),
      async execute(_id, params) {
        return { content: [{ type: "text", text: params.pipeline }] };
      },
    },
    { optional: true },
  );
}
```

يُفعّل المستخدمون الأدوات الاختيارية في التكوين:

```json5
{
  tools: { allow: ["workflow_tool"] },
}
```

- يجب ألا تتعارض أسماء الأدوات مع الأدوات الأساسية (يتم تخطي التعارضات)
- استخدم `optional: true` للأدوات ذات التأثيرات الجانبية أو المتطلبات الثنائية الإضافية
- يمكن للمستخدمين تمكين جميع أدوات إضافة ما عبر إضافة معرّف الإضافة إلى `tools.allow`

## اصطلاحات الاستيراد

استورد دائمًا من مسارات `openclaw/plugin-sdk/<subpath>` المركزة:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";

// خطأ: جذر أحادي ضخم (مهمل وسيزال)
import { ... } from "openclaw/plugin-sdk";
```

للاطلاع على مرجع المسارات الفرعية الكامل، راجع [SDK Overview](/ar/plugins/sdk-overview).

داخل إضافتك، استخدم ملفات barrel المحلية (`api.ts` و`runtime-api.ts`) من أجل
عمليات الاستيراد الداخلية — ولا تستورد إضافتك أبدًا عبر مسار SDK الخاص بها.

بالنسبة إلى إضافات الموفرين، احتفظ بالمساعدات الخاصة بالموفر داخل تلك الملفات
الجامعة في جذر الحزمة ما لم يكن الموصل عامًا حقًا. أمثلة الحزم المجمعة الحالية:

- Anthropic: أغلفة تدفق Claude ومساعدات `service_tier` وbeta
- OpenAI: أدوات بناء الموفّر، ومساعدات النموذج الافتراضي، وموفرو الوقت الحقيقي
- OpenRouter: أداة بناء الموفّر بالإضافة إلى مساعدات الإعداد/التكوين

إذا كان المساعد مفيدًا فقط داخل حزمة موفّر مجمعة واحدة، فاحتفظ به على ذلك
الموصل عند جذر الحزمة بدلًا من ترقيته إلى `openclaw/plugin-sdk/*`.

ما تزال بعض موصلات المساعدة المولدة `openclaw/plugin-sdk/<bundled-id>` موجودة من أجل
صيانة الإضافات المجمعة والتوافق، مثل
`plugin-sdk/feishu-setup` أو `plugin-sdk/zalo-setup`. تعامل مع هذه باعتبارها
أسطحًا محجوزة، وليس كنمط افتراضي للإضافات الخارجية الجديدة.

## قائمة التحقق قبل الإرسال

<Check>يحتوي **package.json** على بيانات `openclaw` الوصفية الصحيحة</Check>
<Check>يوجد manifest **openclaw.plugin.json** وهو صالح</Check>
<Check>تستخدم نقطة الإدخال `defineChannelPluginEntry` أو `definePluginEntry`</Check>
<Check>تستخدم جميع عمليات الاستيراد مسارات `plugin-sdk/<subpath>` المركزة</Check>
<Check>تستخدم عمليات الاستيراد الداخلية وحدات محلية، وليس عمليات استيراد SDK ذاتية</Check>
<Check>تنجح الاختبارات (`pnpm test -- <bundled-plugin-root>/my-plugin/`)</Check>
<Check>ينجح `pnpm check` (للإضافات داخل المستودع)</Check>

## اختبار الإصدارات التجريبية

1. راقب وسوم إصدارات GitHub على [openclaw/openclaw](https://github.com/openclaw/openclaw/releases) واشترك عبر `Watch` > `Releases`. تبدو الوسوم التجريبية مثل `v2026.3.N-beta.1`. يمكنك أيضًا تفعيل الإشعارات لحساب OpenClaw الرسمي على X [@openclaw](https://x.com/openclaw) لإعلانات الإصدارات.
2. اختبر إضافتك مع الوسم التجريبي بمجرد ظهوره. تكون الفترة قبل الإصدار المستقر عادة بضع ساعات فقط.
3. انشر في سلسلة إضافتك ضمن قناة Discord المسماة `plugin-forum` بعد الاختبار مستخدمًا إما `all good` أو موضحًا ما الذي تعطل. إذا لم تكن لديك سلسلة بعد، فأنشئ واحدة.
4. إذا تعطل شيء ما، فافتح issue أو حدّث issue بعنوان `Beta blocker: <plugin-name> - <summary>` وطبّق التصنيف `beta-blocker`. ضع رابط issue في سلسلتك.
5. افتح PR إلى `main` بعنوان `fix(<plugin-id>): beta blocker - <summary>` واربط issue في كل من PR وسلسلة Discord الخاصة بك. لا يستطيع المساهمون وضع التصنيفات على PRs، لذا يكون العنوان هو الإشارة الخاصة بـ PR للمشرفين والأتمتة. يتم دمج العوائق التي لديها PR؛ أما العوائق التي لا تملك PR فقد تُشحن على أي حال. يراقب المشرفون هذه السلاسل أثناء الاختبار التجريبي.
6. الصمت يعني أن كل شيء على ما يرام. إذا فاتتك النافذة، فمن المرجح أن يصل إصلاحك في الدورة التالية.

## الخطوات التالية

<CardGroup cols={2}>
  <Card title="Channel Plugins" icon="messages-square" href="/ar/plugins/sdk-channel-plugins">
    ابنِ إضافة قناة مراسلة
  </Card>
  <Card title="Provider Plugins" icon="cpu" href="/ar/plugins/sdk-provider-plugins">
    ابنِ إضافة موفر نماذج
  </Card>
  <Card title="SDK Overview" icon="book-open" href="/ar/plugins/sdk-overview">
    مرجع خريطة الاستيراد وAPI التسجيل
  </Card>
  <Card title="Runtime Helpers" icon="settings" href="/ar/plugins/sdk-runtime">
    TTS والبحث والوكيل الفرعي عبر api.runtime
  </Card>
  <Card title="Testing" icon="test-tubes" href="/ar/plugins/sdk-testing">
    أدوات وأنماط الاختبار
  </Card>
  <Card title="Plugin Manifest" icon="file-json" href="/ar/plugins/manifest">
    المرجع الكامل لمخطط manifest
  </Card>
</CardGroup>

## ذو صلة

- [Plugin Architecture](/ar/plugins/architecture) — تعمق في البنية الداخلية
- [SDK Overview](/ar/plugins/sdk-overview) — مرجع Plugin SDK
- [Manifest](/ar/plugins/manifest) — تنسيق manifest الخاص بالإضافة
- [Channel Plugins](/ar/plugins/sdk-channel-plugins) — بناء إضافات القنوات
- [Provider Plugins](/ar/plugins/sdk-provider-plugins) — بناء إضافات الموفرين
