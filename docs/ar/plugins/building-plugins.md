---
read_when:
    - أنت تريد إنشاء plugin جديد لـ OpenClaw
    - تحتاج إلى بدء سريع لتطوير plugin
    - أنت تضيف قناة جديدة، أو مزوّدًا، أو أداة، أو قدرة أخرى إلى OpenClaw
sidebarTitle: Getting Started
summary: أنشئ أول plugin لـ OpenClaw خلال دقائق
title: Building Plugins
x-i18n:
    generated_at: "2026-04-05T12:51:23Z"
    model: gpt-5.4
    provider: openai
    source_hash: 26e780d3f04270b79d1d8f8076d6c3c5031915043e78fb8174be921c6bdd60c9
    source_path: plugins/building-plugins.md
    workflow: 15
---

# Building Plugins

توسّع Plugins إمكانات OpenClaw بقدرات جديدة: القنوات، ومزوّدو النماذج،
والنطق، والنسخ الفوري، والصوت الفوري، وفهم الوسائط، وتوليد
الصور، وتوليد الفيديو، وجلب الويب، والبحث على الويب، وأدوات الوكيل، أو أي
مزيج منها.

لا تحتاج إلى إضافة plugin الخاص بك إلى مستودع OpenClaw. انشره إلى
[ClawHub](/tools/clawhub) أو npm وسيقوم المستخدمون بالتثبيت باستخدام
`openclaw plugins install <package-name>`. يحاول OpenClaw استخدام ClawHub أولًا ثم
يعود إلى npm تلقائيًا.

## المتطلبات المسبقة

- Node >= 22 ومدير حزم (npm أو pnpm)
- إلمام بـ TypeScript ‏(ESM)
- بالنسبة إلى plugins داخل المستودع: يجب استنساخ المستودع وتشغيل `pnpm install`

## أي نوع من plugins؟

<CardGroup cols={3}>
  <Card title="plugin قناة" icon="messages-square" href="/plugins/sdk-channel-plugins">
    صِل OpenClaw بمنصة مراسلة (Discord، IRC، إلخ)
  </Card>
  <Card title="plugin مزوّد" icon="cpu" href="/plugins/sdk-provider-plugins">
    أضف مزوّد نماذج (LLM، أو proxy، أو نقطة نهاية مخصصة)
  </Card>
  <Card title="plugin أداة / hook" icon="wrench">
    سجّل أدوات الوكيل، أو event hooks، أو الخدمات — تابع أدناه
  </Card>
</CardGroup>

إذا كان plugin القناة اختياريًا وقد لا يكون مثبتًا عند تشغيل onboarding/setup،
فاستخدم `createOptionalChannelSetupSurface(...)` من
`openclaw/plugin-sdk/channel-setup`. حيث ينتج محول إعداد + زوج معالج
يعلن متطلب التثبيت ويفشل بشكل مغلق عند عمليات كتابة الإعدادات الحقيقية
إلى أن يتم تثبيت plugin.

## بدء سريع: plugin أداة

ينشئ هذا الشرح plugin أدنى يسجل أداة وكيل. أما
Plugins القنوات والمزوّدين فلها أدلة مخصصة مرتبطة أعلاه.

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

    يحتاج كل plugin إلى manifest، حتى لو لم يكن له إعدادات. راجع
    [Manifest](/plugins/manifest) للمخطط الكامل. وتوجد مقتطفات النشر المعيارية إلى ClawHub
    في `docs/snippets/plugin-publish/`.

  </Step>

  <Step title="اكتب نقطة الدخول">

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

    يُستخدم `definePluginEntry` مع Plugins غير الخاصة بالقنوات. أما القنوات فاستخدم
    `defineChannelPluginEntry` — راجع [Channel Plugins](/plugins/sdk-channel-plugins).
    وللحصول على خيارات نقاط الدخول الكاملة، راجع [Entry Points](/plugins/sdk-entrypoints).

  </Step>

  <Step title="اختبر وانشر">

    **Plugins الخارجية:** تحقّق وانشر باستخدام ClawHub، ثم ثبّت:

    ```bash
    clawhub package publish your-org/your-plugin --dry-run
    clawhub package publish your-org/your-plugin
    openclaw plugins install clawhub:@myorg/openclaw-my-plugin
    ```

    يتحقق OpenClaw أيضًا من ClawHub قبل npm لمواصفات الحزم المجردة مثل
    `@myorg/openclaw-my-plugin`.

    **Plugins داخل المستودع:** ضعها تحت شجرة مساحة العمل الخاصة بالplugins المضمنة — سيتم اكتشافها تلقائيًا.

    ```bash
    pnpm test -- <bundled-plugin-root>/my-plugin/
    ```

  </Step>
</Steps>

## قدرات plugin

يمكن لـ plugin واحد تسجيل أي عدد من القدرات عبر الكائن `api`:

| القدرة                 | طريقة التسجيل                                  | الدليل التفصيلي                                                                 |
| ---------------------- | ----------------------------------------------- | ------------------------------------------------------------------------------- |
| الاستدلال النصي (LLM)  | `api.registerProvider(...)`                     | [Provider Plugins](/plugins/sdk-provider-plugins)                               |
| خلفية استدلال CLI      | `api.registerCliBackend(...)`                   | [CLI Backends](/gateway/cli-backends)                                           |
| القناة / المراسلة      | `api.registerChannel(...)`                      | [Channel Plugins](/plugins/sdk-channel-plugins)                                 |
| النطق (TTS/STT)        | `api.registerSpeechProvider(...)`               | [Provider Plugins](/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| النسخ الفوري           | `api.registerRealtimeTranscriptionProvider(...)` | [Provider Plugins](/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| الصوت الفوري           | `api.registerRealtimeVoiceProvider(...)`        | [Provider Plugins](/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| فهم الوسائط            | `api.registerMediaUnderstandingProvider(...)`   | [Provider Plugins](/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| توليد الصور            | `api.registerImageGenerationProvider(...)`      | [Provider Plugins](/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| توليد الفيديو          | `api.registerVideoGenerationProvider(...)`      | [Provider Plugins](/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| جلب الويب              | `api.registerWebFetchProvider(...)`             | [Provider Plugins](/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| البحث على الويب        | `api.registerWebSearchProvider(...)`            | [Provider Plugins](/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| أدوات الوكيل           | `api.registerTool(...)`                         | أدناه                                                                           |
| أوامر مخصصة            | `api.registerCommand(...)`                      | [Entry Points](/plugins/sdk-entrypoints)                                        |
| event hooks            | `api.registerHook(...)`                         | [Entry Points](/plugins/sdk-entrypoints)                                        |
| مسارات HTTP            | `api.registerHttpRoute(...)`                    | [الآليات الداخلية](/plugins/architecture#gateway-http-routes)                   |
| أوامر CLI فرعية        | `api.registerCli(...)`                          | [Entry Points](/plugins/sdk-entrypoints)                                        |

للحصول على API التسجيل الكاملة، راجع [نظرة عامة على SDK](/plugins/sdk-overview#registration-api).

إذا كان plugin الخاص بك يسجل طرق Gateway RPC مخصصة، فأبقها تحت
بادئة خاصة بذلك plugin. وتبقى مساحات أسماء الإدارة الأساسية (`config.*`،
و`exec.approvals.*`، و`wizard.*`، و`update.*`) محجوزة وتحل دائمًا إلى
`operator.admin`، حتى لو طلب plugin نطاقًا أضيق.

دلالات بوابات hooks التي يجب أخذها في الاعتبار:

- `before_tool_call`: تكون `{ block: true }` نهائية وتوقف المعالجات ذات الأولوية الأدنى.
- `before_tool_call`: تُعامل `{ block: false }` على أنها بلا قرار.
- `before_tool_call`: تؤدي `{ requireApproval: true }` إلى إيقاف تنفيذ الوكيل مؤقتًا وتطلب موافقة المستخدم عبر واجهة موافقة exec أو أزرار Telegram أو تفاعلات Discord أو الأمر `/approve` على أي قناة.
- `before_install`: تكون `{ block: true }` نهائية وتوقف المعالجات ذات الأولوية الأدنى.
- `before_install`: تُعامل `{ block: false }` على أنها بلا قرار.
- `message_sending`: تكون `{ cancel: true }` نهائية وتوقف المعالجات ذات الأولوية الأدنى.
- `message_sending`: تُعامل `{ cancel: false }` على أنها بلا قرار.

يعالج الأمر `/approve` موافقات exec وplugin معًا عبر تراجع محدود: فعندما لا يتم العثور على معرّف موافقة exec، يعيد OpenClaw محاولة المعرّف نفسه عبر موافقات plugin. ويمكن إعداد إعادة توجيه موافقات plugin بشكل مستقل عبر `approvals.plugin` في الإعدادات.

إذا كانت البنية المخصصة للموافقات تحتاج إلى اكتشاف حالة التراجع المحدود نفسها،
فافضّل استخدام `isApprovalNotFoundError` من `openclaw/plugin-sdk/error-runtime`
بدلًا من مطابقة سلاسل انتهاء صلاحية الموافقة يدويًا.

راجع [دلالات قرارات hooks في نظرة عامة على SDK](/plugins/sdk-overview#hook-decision-semantics) للتفاصيل.

## تسجيل أدوات الوكيل

الأدوات هي دوال Typed يمكن لـ LLM استدعاؤها. ويمكن أن تكون مطلوبة (متاحة
دائمًا) أو اختيارية (اشتراك مستخدم):

```typescript
register(api) {
  // Required tool — always available
  api.registerTool({
    name: "my_tool",
    description: "Do a thing",
    parameters: Type.Object({ input: Type.String() }),
    async execute(_id, params) {
      return { content: [{ type: "text", text: params.input }] };
    },
  });

  // Optional tool — user must add to allowlist
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

يقوم المستخدمون بتمكين الأدوات الاختيارية في الإعدادات:

```json5
{
  tools: { allow: ["workflow_tool"] },
}
```

- يجب ألا تتعارض أسماء الأدوات مع أدوات core ‏(يتم تخطي التعارضات)
- استخدم `optional: true` للأدوات ذات الآثار الجانبية أو التي تتطلب ملفات تنفيذية إضافية
- يمكن للمستخدمين تمكين جميع أدوات plugin بإضافة معرّف plugin إلى `tools.allow`

## اصطلاحات الاستيراد

استورد دائمًا من المسارات المركزة `openclaw/plugin-sdk/<subpath>`:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";

// Wrong: monolithic root (deprecated, will be removed)
import { ... } from "openclaw/plugin-sdk";
```

للحصول على المرجع الكامل للمسارات الفرعية، راجع [نظرة عامة على SDK](/plugins/sdk-overview).

داخل plugin الخاص بك، استخدم ملفات barrel محلية (`api.ts` و`runtime-api.ts`) من أجل
الاستيرادات الداخلية — ولا تستورد plugin الخاص بك أبدًا عبر مسار SDK الخاص به.

بالنسبة إلى Provider Plugins، أبقِ المساعدات الخاصة بالمزوّد في ملفات barrel
عند جذر تلك الحزمة ما لم يكن الوصل عامًا حقًا. ومن الأمثلة المضمنة الحالية:

- Anthropic: مغلّفات بث Claude ومساعدات `service_tier` / beta
- OpenAI: بُنّاة المزوّد، ومساعدات النموذج الافتراضي، ومزوّدو الوقت الفعلي
- OpenRouter: بنّاء المزوّد بالإضافة إلى مساعدات onboarding/config

إذا كان المساعد مفيدًا فقط داخل حزمة مزوّد مضمنة واحدة، فأبقِه على ذلك
الوصل الموجود عند جذر الحزمة بدلًا من ترقيته إلى `openclaw/plugin-sdk/*`.

لا تزال بعض وصلات المساعدة المولدة في `openclaw/plugin-sdk/<bundled-id>` موجودة من أجل
صيانة bundled-plugin والتوافق، مثل
`plugin-sdk/feishu-setup` أو `plugin-sdk/zalo-setup`. تعامل مع هذه
على أنها أسطح محجوزة، وليست النمط الافتراضي لـ Plugins الجهات الخارجية الجديدة.

## قائمة التحقق قبل الإرسال

<Check>يحتوي **package.json** على بيانات `openclaw` الوصفية الصحيحة</Check>
<Check>يوجد manifest صالح في **openclaw.plugin.json**</Check>
<Check>تستخدم نقطة الدخول `defineChannelPluginEntry` أو `definePluginEntry`</Check>
<Check>تستخدم جميع الاستيرادات المسارات المركزة `plugin-sdk/<subpath>`</Check>
<Check>تستخدم الاستيرادات الداخلية وحدات محلية، لا عمليات استيراد ذاتية عبر SDK</Check>
<Check>نجحت الاختبارات (`pnpm test -- <bundled-plugin-root>/my-plugin/`)</Check>
<Check>نجح `pnpm check` ‏(لـ plugins داخل المستودع)</Check>

## اختبار الإصدارات التجريبية

1. راقب وسوم إصدارات GitHub على [openclaw/openclaw](https://github.com/openclaw/openclaw/releases) واشترك عبر `Watch` > `Releases`. تبدو وسوم beta مثل `v2026.3.N-beta.1`. ويمكنك أيضًا تشغيل الإشعارات على الحساب الرسمي لـ OpenClaw على X ‏[@openclaw](https://x.com/openclaw) لإعلانات الإصدارات.
2. اختبر plugin الخاص بك مقابل وسم beta فور ظهوره. فالنافذة قبل الإصدار المستقر تكون عادةً بضع ساعات فقط.
3. انشر في سلسلة plugin الخاصة بك في قناة Discord ‏`plugin-forum` بعد الاختبار مع عبارة `all good` أو وصف ما الذي تعطل. وإذا لم تكن لديك سلسلة بعد، فأنشئ واحدة.
4. إذا تعطل شيء ما، فافتح issue جديدًا أو حدّث واحدًا بعنوان `Beta blocker: <plugin-name> - <summary>` وطبّق الوسم `beta-blocker`. وضع رابط issue في سلسلتك.
5. افتح PR إلى `main` بعنوان `fix(<plugin-id>): beta blocker - <summary>` واربط issue في كل من PR وسلسلة Discord. لا يستطيع المساهمون وضع وسوم على PRs، لذا يكون العنوان هو الإشارة الخاصة بـ PR للمشرفين والأتمتة. يتم دمج الأعطال الحاجزة التي لها PR؛ أما الأعطال الحاجزة التي بلا PR فقد تُشحن على أي حال. ويراقب المشرفون هذه السلاسل أثناء اختبار beta.
6. الصمت يعني أن كل شيء أخضر. وإذا فاتتك النافذة، فمن المحتمل أن يصل إصلاحك في الدورة التالية.

## الخطوات التالية

<CardGroup cols={2}>
  <Card title="Channel Plugins" icon="messages-square" href="/plugins/sdk-channel-plugins">
    ابنِ plugin قناة للمراسلة
  </Card>
  <Card title="Provider Plugins" icon="cpu" href="/plugins/sdk-provider-plugins">
    ابنِ plugin مزوّد نماذج
  </Card>
  <Card title="SDK Overview" icon="book-open" href="/plugins/sdk-overview">
    مرجع خريطة الاستيراد وAPI التسجيل
  </Card>
  <Card title="Runtime Helpers" icon="settings" href="/plugins/sdk-runtime">
    TTS والبحث وsubagent عبر api.runtime
  </Card>
  <Card title="Testing" icon="test-tubes" href="/plugins/sdk-testing">
    أدوات وأنماط الاختبار
  </Card>
  <Card title="Plugin Manifest" icon="file-json" href="/plugins/manifest">
    المرجع الكامل لمخطط manifest
  </Card>
</CardGroup>

## ذو صلة

- [بنية plugin](/plugins/architecture) — تعمق في البنية الداخلية
- [نظرة عامة على SDK](/plugins/sdk-overview) — مرجع Plugin SDK
- [Manifest](/plugins/manifest) — تنسيق manifest الخاص بالplugin
- [Channel Plugins](/plugins/sdk-channel-plugins) — بناء plugins القنوات
- [Provider Plugins](/plugins/sdk-provider-plugins) — بناء plugins المزوّدين
