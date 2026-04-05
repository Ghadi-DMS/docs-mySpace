---
read_when:
    - أنت تنشئ plugin لـ OpenClaw
    - تحتاج إلى شحن schema لتكوين plugin أو تصحيح أخطاء التحقق من plugin
summary: متطلبات manifest الخاصة بالـ plugin وJSON schema ‏(تحقق صارم من التكوين)
title: Plugin Manifest
x-i18n:
    generated_at: "2026-04-05T12:52:00Z"
    model: gpt-5.4
    provider: openai
    source_hash: 702447ad39f295cfffd4214c3e389bee667d2f9850754f2e02e325dde8e4ac00
    source_path: plugins/manifest.md
    workflow: 15
---

# Plugin manifest ‏(`openclaw.plugin.json`)

هذه الصفحة مخصصة فقط لـ **manifest الخاصة بالـ plugin الأصلية في OpenClaw**.

للتخطيطات المتوافقة مع bundles، راجع [Plugin bundles](/plugins/bundles).

تستخدم تنسيقات bundles المتوافقة ملفات manifest مختلفة:

- Codex bundle: ‏`.codex-plugin/plugin.json`
- Claude bundle: ‏`.claude-plugin/plugin.json` أو تخطيط مكوّن Claude
  الافتراضي من دون manifest
- Cursor bundle: ‏`.cursor-plugin/plugin.json`

يكتشف OpenClaw هذه التخطيطات المتوافقة مع bundles تلقائيًا أيضًا، لكنها لا تُتحقق
مقابل schema الخاصة بـ `openclaw.plugin.json` الموصوفة هنا.

بالنسبة إلى bundles المتوافقة، يقرأ OpenClaw حاليًا بيانات تعريف bundle بالإضافة إلى
جذور Skills المعلنة، وجذور أوامر Claude، والقيم الافتراضية لـ `settings.json` في Claude bundle،
والقيم الافتراضية لـ Claude bundle LSP، وحزم الخطافات المدعومة عندما يطابق التخطيط
توقعات وقت تشغيل OpenClaw.

يجب على كل plugin أصلية في OpenClaw **أن تشحن ملف `openclaw.plugin.json`** في
**جذر plugin**. يستخدم OpenClaw هذه manifest للتحقق من التكوين
**من دون تنفيذ شيفرة plugin**. وتُعامل manifestات المفقودة أو غير الصالحة على أنها
أخطاء plugin وتمنع التحقق من التكوين.

راجع الدليل الكامل لنظام plugins: [Plugins](/tools/plugin).
وللاطلاع على نموذج الإمكانات الأصلية وإرشادات التوافق الخارجي الحالية:
[Capability model](/plugins/architecture#public-capability-model).

## ما الذي يفعله هذا الملف

`openclaw.plugin.json` هي بيانات التعريف التي يقرأها OpenClaw قبل تحميل
شيفرة plugin.

استخدمها من أجل:

- هوية plugin
- التحقق من التكوين
- بيانات تعريف المصادقة والتهيئة الأولية التي يجب أن تكون متاحة من دون تشغيل وقت تشغيل plugin
- بيانات تعريف الاسم المستعار والتمكين التلقائي التي يجب أن تُحل قبل تحميل وقت تشغيل plugin
- بيانات تعريف ملكية عائلة النماذج المختصرة التي يجب أن تفعّل
  plugin تلقائيًا قبل تحميل وقت التشغيل
- لقطات ملكية الإمكانات الثابتة المستخدمة في توصيلات التوافق المضمّنة وتغطية العقود
- بيانات تعريف تكوين خاصة بالقناة يجب دمجها في أسطح الفهرس والتحقق من دون تحميل وقت التشغيل
- تلميحات واجهة التكوين

لا تستخدمها من أجل:

- تسجيل سلوك وقت التشغيل
- الإعلان عن entrypoints الشيفرة
- بيانات تعريف npm install

فهذه تنتمي إلى شيفرة plugin وإلى `package.json`.

## مثال أدنى

```json
{
  "id": "voice-call",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

## مثال غني

```json
{
  "id": "openrouter",
  "name": "OpenRouter",
  "description": "OpenRouter provider plugin",
  "version": "1.0.0",
  "providers": ["openrouter"],
  "modelSupport": {
    "modelPrefixes": ["router-"]
  },
  "cliBackends": ["openrouter-cli"],
  "providerAuthEnvVars": {
    "openrouter": ["OPENROUTER_API_KEY"]
  },
  "providerAuthChoices": [
    {
      "provider": "openrouter",
      "method": "api-key",
      "choiceId": "openrouter-api-key",
      "choiceLabel": "OpenRouter API key",
      "groupId": "openrouter",
      "groupLabel": "OpenRouter",
      "optionKey": "openrouterApiKey",
      "cliFlag": "--openrouter-api-key",
      "cliOption": "--openrouter-api-key <key>",
      "cliDescription": "OpenRouter API key",
      "onboardingScopes": ["text-inference"]
    }
  ],
  "uiHints": {
    "apiKey": {
      "label": "API key",
      "placeholder": "sk-or-v1-...",
      "sensitive": true
    }
  },
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "apiKey": {
        "type": "string"
      }
    }
  }
}
```

## مرجع الحقول العليا

| الحقل                               | مطلوب | النوع                             | المعنى                                                                                                                                                                                     |
| ----------------------------------- | ------ | --------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `id`                                | نعم    | `string`                          | معرّف plugin الأساسي. وهذا هو المعرّف المستخدم في `plugins.entries.<id>`.                                                                                                                  |
| `configSchema`                      | نعم    | `object`                          | JSON Schema مضمنة لتكوين هذه الـ plugin.                                                                                                                                                   |
| `enabledByDefault`                  | لا     | `true`                            | تضع علامة على plugin مضمّنة على أنها مفعلة افتراضيًا. احذف الحقل، أو عيّن أي قيمة غير `true`، لترك plugin معطلة افتراضيًا.                                                              |
| `legacyPluginIds`                   | لا     | `string[]`                        | معرّفات قديمة تُطبّع إلى معرّف plugin الأساسي هذا.                                                                                                                                         |
| `autoEnableWhenConfiguredProviders` | لا     | `string[]`                        | معرّفات مزوّدين يجب أن تفعّل هذه الـ plugin تلقائيًا عندما تشير إليهم المصادقة أو التكوين أو مراجع النماذج.                                                                              |
| `kind`                              | لا     | `"memory"` \| `"context-engine"`  | يعلن نوع plugin حصريًا يُستخدم بواسطة `plugins.slots.*`.                                                                                                                                   |
| `channels`                          | لا     | `string[]`                        | معرّفات القنوات المملوكة لهذه الـ plugin. تُستخدم للاكتشاف والتحقق من التكوين.                                                                                                             |
| `providers`                         | لا     | `string[]`                        | معرّفات المزوّدين المملوكة لهذه الـ plugin.                                                                                                                                                 |
| `modelSupport`                      | لا     | `object`                          | بيانات تعريف مختصرة لعائلة النماذج مملوكة للـ manifest وتُستخدم لتحميل plugin تلقائيًا قبل وقت التشغيل.                                                                                  |
| `cliBackends`                       | لا     | `string[]`                        | معرّفات backends استدلال CLI المملوكة لهذه الـ plugin. تُستخدم للتفعيل التلقائي عند بدء التشغيل من مراجع التكوين الصريحة.                                                                  |
| `providerAuthEnvVars`               | لا     | `Record<string, string[]>`        | بيانات تعريف رخيصة لمتغيرات بيئة مصادقة المزوّد يمكن لـ OpenClaw فحصها من دون تحميل شيفرة plugin.                                                                                        |
| `providerAuthChoices`               | لا     | `object[]`                        | بيانات تعريف رخيصة لاختيارات المصادقة لمحددات onboarding، وتحليل المزوّد المفضّل، وربط علامات CLI البسيطة.                                                                                 |
| `contracts`                         | لا     | `object`                          | لقطة ثابتة لإمكانات مضمّنة تشمل speech وrealtime transcription وrealtime voice وmedia-understanding وimage-generation وvideo-generation وweb-fetch وweb search وملكية الأدوات.          |
| `channelConfigs`                    | لا     | `Record<string, object>`          | بيانات تعريف تكوين القناة المملوكة للـ manifest والمُدمجة في أسطح الاكتشاف والتحقق قبل تحميل وقت التشغيل.                                                                                 |
| `skills`                            | لا     | `string[]`                        | أدلة Skills المطلوب تحميلها، نسبةً إلى جذر plugin.                                                                                                                                         |
| `name`                              | لا     | `string`                          | اسم plugin قابل للقراءة البشرية.                                                                                                                                                           |
| `description`                       | لا     | `string`                          | ملخص قصير يظهر في أسطح plugin.                                                                                                                                                             |
| `version`                           | لا     | `string`                          | إصدار plugin لأغراض معلوماتية.                                                                                                                                                             |
| `uiHints`                           | لا     | `Record<string, object>`          | تسميات واجهة المستخدم والعناصر النائبة وتلميحات الحساسية لحقول التكوين.                                                                                                                   |

## مرجع `providerAuthChoices`

يصف كل إدخال في `providerAuthChoices` اختيارًا واحدًا للتهيئة الأولية أو للمصادقة.
يقرأ OpenClaw هذا قبل تحميل وقت تشغيل المزوّد.

| الحقل                 | مطلوب | النوع                                            | المعنى                                                                                                  |
| --------------------- | ------ | ------------------------------------------------ | -------------------------------------------------------------------------------------------------------- |
| `provider`            | نعم    | `string`                                         | معرّف المزوّد الذي ينتمي إليه هذا الاختيار.                                                              |
| `method`              | نعم    | `string`                                         | معرّف طريقة المصادقة المطلوب توجيهها إليها.                                                              |
| `choiceId`            | نعم    | `string`                                         | معرّف auth-choice ثابت يُستخدم في onboarding وتدفقات CLI.                                                |
| `choiceLabel`         | لا     | `string`                                         | تسمية موجهة للمستخدم. وإذا حُذفت، يعود OpenClaw إلى `choiceId`.                                          |
| `choiceHint`          | لا     | `string`                                         | نص مساعد قصير للمحدد.                                                                                    |
| `assistantPriority`   | لا     | `number`                                         | القيم الأقل تُرتَّب أولًا في محددات التفاعل التي يقودها المساعد.                                         |
| `assistantVisibility` | لا     | `"visible"` \| `"manual-only"`                   | إخفاء الاختيار من محددات المساعد مع السماح باختياره يدويًا عبر CLI.                                     |
| `deprecatedChoiceIds` | لا     | `string[]`                                       | معرّفات اختيار قديمة يجب أن تعيد توجيه المستخدمين إلى اختيار الاستبدال هذا.                             |
| `groupId`             | لا     | `string`                                         | معرّف مجموعة اختياري لتجميع الاختيارات ذات الصلة.                                                        |
| `groupLabel`          | لا     | `string`                                         | تسمية موجهة للمستخدم لتلك المجموعة.                                                                      |
| `groupHint`           | لا     | `string`                                         | نص مساعد قصير للمجموعة.                                                                                  |
| `optionKey`           | لا     | `string`                                         | مفتاح خيار داخلي لتدفقات المصادقة البسيطة ذات العلم الواحد.                                              |
| `cliFlag`             | لا     | `string`                                         | اسم علم CLI، مثل `--openrouter-api-key`.                                                                 |
| `cliOption`           | لا     | `string`                                         | شكل خيار CLI الكامل، مثل `--openrouter-api-key <key>`.                                                   |
| `cliDescription`      | لا     | `string`                                         | الوصف المستخدم في مساعدة CLI.                                                                            |
| `onboardingScopes`    | لا     | `Array<"text-inference" \| "image-generation">`  | أسطح onboarding التي يجب أن يظهر فيها هذا الاختيار. وإذا حُذف، تكون القيمة الافتراضية `["text-inference"]`. |

## مرجع `uiHints`

تمثل `uiHints` خريطة من أسماء حقول التكوين إلى تلميحات عرض صغيرة.

```json
{
  "uiHints": {
    "apiKey": {
      "label": "API key",
      "help": "Used for OpenRouter requests",
      "placeholder": "sk-or-v1-...",
      "sensitive": true
    }
  }
}
```

يمكن أن يتضمن كل تلميح حقل:

| الحقل         | النوع      | المعنى                                  |
| ------------- | ---------- | --------------------------------------- |
| `label`       | `string`   | تسمية الحقل الموجهة للمستخدم.           |
| `help`        | `string`   | نص مساعد قصير.                          |
| `tags`        | `string[]` | وسوم اختيارية لواجهة المستخدم.          |
| `advanced`    | `boolean`  | يضع علامة على الحقل على أنه متقدم.      |
| `sensitive`   | `boolean`  | يضع علامة على الحقل كسري أو حساس.       |
| `placeholder` | `string`   | نص العنصر النائب لمدخلات النماذج.       |

## مرجع `contracts`

استخدم `contracts` فقط لبيانات تعريف ملكية الإمكانات الثابتة التي يستطيع OpenClaw
قراءتها من دون استيراد وقت تشغيل plugin.

```json
{
  "contracts": {
    "speechProviders": ["openai"],
    "realtimeTranscriptionProviders": ["openai"],
    "realtimeVoiceProviders": ["openai"],
    "mediaUnderstandingProviders": ["openai", "openai-codex"],
    "imageGenerationProviders": ["openai"],
    "videoGenerationProviders": ["qwen"],
    "webFetchProviders": ["firecrawl"],
    "webSearchProviders": ["gemini"],
    "tools": ["firecrawl_search", "firecrawl_scrape"]
  }
}
```

كل قائمة اختيارية:

| الحقل                            | النوع      | المعنى                                                     |
| -------------------------------- | ---------- | ---------------------------------------------------------- |
| `speechProviders`                | `string[]` | معرّفات مزوّدي speech التي تملكها هذه الـ plugin.          |
| `realtimeTranscriptionProviders` | `string[]` | معرّفات مزوّدي realtime-transcription التي تملكها هذه الـ plugin. |
| `realtimeVoiceProviders`         | `string[]` | معرّفات مزوّدي realtime-voice التي تملكها هذه الـ plugin.   |
| `mediaUnderstandingProviders`    | `string[]` | معرّفات مزوّدي media-understanding التي تملكها هذه الـ plugin. |
| `imageGenerationProviders`       | `string[]` | معرّفات مزوّدي image-generation التي تملكها هذه الـ plugin. |
| `videoGenerationProviders`       | `string[]` | معرّفات مزوّدي video-generation التي تملكها هذه الـ plugin. |
| `webFetchProviders`              | `string[]` | معرّفات مزوّدي web-fetch التي تملكها هذه الـ plugin.        |
| `webSearchProviders`             | `string[]` | معرّفات مزوّدي web-search التي تملكها هذه الـ plugin.       |
| `tools`                          | `string[]` | أسماء أدوات الوكيل التي تملكها هذه الـ plugin لفحوصات العقود المضمّنة. |

## مرجع `channelConfigs`

استخدم `channelConfigs` عندما تحتاج plugin قناة إلى بيانات تعريف تكوين رخيصة قبل
تحميل وقت التشغيل.

```json
{
  "channelConfigs": {
    "matrix": {
      "schema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "homeserverUrl": { "type": "string" }
        }
      },
      "uiHints": {
        "homeserverUrl": {
          "label": "Homeserver URL",
          "placeholder": "https://matrix.example.com"
        }
      },
      "label": "Matrix",
      "description": "Matrix homeserver connection",
      "preferOver": ["matrix-legacy"]
    }
  }
}
```

يمكن أن يتضمن كل إدخال قناة:

| الحقل         | النوع                     | المعنى                                                                                   |
| ------------- | ------------------------- | ----------------------------------------------------------------------------------------- |
| `schema`      | `object`                  | JSON Schema لـ `channels.<id>`. وهو مطلوب لكل إدخال تكوين قناة معلن.                    |
| `uiHints`     | `Record<string, object>`  | تسميات/عناصر نائبة/تلميحات حساسية اختيارية لواجهة المستخدم لذلك القسم من تكوين القناة. |
| `label`       | `string`                  | تسمية القناة المدمجة في أسطح المحدد والفحص عندما لا تكون بيانات تعريف وقت التشغيل جاهزة. |
| `description` | `string`                  | وصف قصير للقناة لأسطح الفحص والفهرس.                                                     |
| `preferOver`  | `string[]`                | معرّفات plugin قديمة أو أقل أولوية يجب أن تتفوق عليها هذه القناة في أسطح الاختيار.      |

## مرجع `modelSupport`

استخدم `modelSupport` عندما يجب أن يستنتج OpenClaw plugin المزوّد الخاصة بك من
معرّفات النماذج المختصرة مثل `gpt-5.4` أو `claude-sonnet-4.6` قبل تحميل
وقت تشغيل plugin.

```json
{
  "modelSupport": {
    "modelPrefixes": ["gpt-", "o1", "o3", "o4"],
    "modelPatterns": ["^computer-use-preview"]
  }
}
```

يطبّق OpenClaw أولوية كهذه:

- تستخدم مراجع `provider/model` الصريحة بيانات تعريف `providers` المملوكة للـ manifest
- تتفوق `modelPatterns` على `modelPrefixes`
- إذا طابقت plugin واحدة غير مضمّنة وواحدة مضمّنة معًا، فإن
  plugin غير المضمّنة تنتصر
- يتم تجاهل ما تبقى من غموض حتى يحدد المستخدم أو التكوين مزوّدًا

الحقول:

| الحقل           | النوع      | المعنى                                                                        |
| --------------- | ---------- | ----------------------------------------------------------------------------- |
| `modelPrefixes` | `string[]` | بادئات تُطابق باستخدام `startsWith` مع معرّفات النماذج المختصرة.             |
| `modelPatterns` | `string[]` | مصادر regex تُطابق مع معرّفات النماذج المختصرة بعد إزالة لاحقة ملف التعريف.  |

أصبحت مفاتيح الإمكانات القديمة ذات المستوى الأعلى متقادمة. استخدم `openclaw doctor --fix` لنقل
`speechProviders` و`realtimeTranscriptionProviders`,
و`realtimeVoiceProviders` و`mediaUnderstandingProviders`,
و`imageGenerationProviders` و`videoGenerationProviders`,
و`webFetchProviders` و`webSearchProviders` تحت `contracts`؛ إذ لم يعد
تحميل manifest العادي يعامل تلك الحقول ذات المستوى الأعلى كملكية
للإمكانات.

## manifest مقابل `package.json`

يخدم الملفان أغراضًا مختلفة:

| الملف                  | يُستخدم من أجل                                                                                                                       |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `openclaw.plugin.json` | الاكتشاف، والتحقق من التكوين، وبيانات تعريف auth-choice، وتلميحات واجهة المستخدم التي يجب أن توجد قبل تشغيل شيفرة plugin            |
| `package.json`         | بيانات تعريف npm، وتثبيت التبعيات، وكتلة `openclaw` المستخدمة للـ entrypoints، وضبط التثبيت، والإعداد، أو بيانات تعريف الفهرس      |

إذا لم تكن متأكدًا من مكان وضع جزء من بيانات التعريف، فاستخدم هذه القاعدة:

- إذا كان يجب على OpenClaw معرفته قبل تحميل شيفرة plugin، فضعه في `openclaw.plugin.json`
- إذا كان يتعلق بالتغليف، أو ملفات الإدخال، أو سلوك npm install، فضعه في `package.json`

### حقول `package.json` التي تؤثر في الاكتشاف

تعيش بعض بيانات تعريف plugin السابقة لوقت التشغيل عمدًا في `package.json` تحت
كتلة `openclaw` بدلًا من `openclaw.plugin.json`.

أمثلة مهمة:

| الحقل                                                             | المعنى                                                                                 |
| ----------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| `openclaw.extensions`                                             | يعلن entrypoints الخاصة بالـ plugin الأصلية.                                           |
| `openclaw.setupEntry`                                             | entrypoint خفيف للإعداد فقط، يُستخدم أثناء onboarding وبدء القنوات المؤجل.            |
| `openclaw.channel`                                                | بيانات تعريف خفيفة لفهرس القنوات مثل التسميات ومسارات الوثائق والأسماء المستعارة ونصوص الاختيار. |
| `openclaw.install.npmSpec` / `openclaw.install.localPath`         | تلميحات تثبيت/تحديث للـ plugins المضمّنة والمنشورة خارجيًا.                            |
| `openclaw.install.defaultChoice`                                  | مسار التثبيت المفضل عندما تتوفر مصادر تثبيت متعددة.                                    |
| `openclaw.install.minHostVersion`                                 | الحد الأدنى لإصدار مضيف OpenClaw المدعوم، باستخدام حد semver أدنى مثل `>=2026.3.22`.  |
| `openclaw.install.allowInvalidConfigRecovery`                     | يسمح بمسار ضيق لاستعادة إعادة تثبيت plugin المضمّنة عندما يكون التكوين غير صالح.      |
| `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen` | يسمح بتحميل أسطح القنوات الخاصة بالإعداد فقط قبل plugin القناة الكاملة أثناء بدء التشغيل. |

يتم فرض `openclaw.install.minHostVersion` أثناء التثبيت وتحميل سجل
manifest. وتُرفض القيم غير الصالحة؛ أما القيم الأحدث ولكن الصالحة فتتخطى
plugin على المضيفات الأقدم.

يكون `openclaw.install.allowInvalidConfigRecovery` ضيقًا عمدًا. فهو لا
يجعل التكوينات المعطلة عشوائيًا قابلة للتثبيت. اليوم يسمح فقط لتدفقات التثبيت
بالتعافي من إخفاقات ترقية محددة قديمة في plugins المضمّنة، مثل مسار plugin مضمّنة مفقود
أو إدخال `channels.<id>` قديم لتلك plugin المضمّنة نفسها.
ولا تزال أخطاء التكوين غير ذات الصلة تمنع التثبيت وتوجّه المشغلين إلى
`openclaw doctor --fix`.

## متطلبات JSON Schema

- **يجب أن تشحن كل plugin JSON Schema**، حتى إذا لم تكن تقبل أي تكوين.
- schema فارغة مقبولة (مثل `{ "type": "object", "additionalProperties": false }`).
- يتم التحقق من schemas في وقت قراءة/كتابة التكوين، وليس وقت التشغيل.

## سلوك التحقق

- تكون مفاتيح `channels.*` غير المعروفة **أخطاء**، ما لم يعلن
  manifest الخاصة بـ plugin عن معرّف القناة.
- يجب أن تشير `plugins.entries.<id>` و`plugins.allow` و`plugins.deny` و`plugins.slots.*`
  إلى معرّفات plugin **قابلة للاكتشاف**. وتكون المعرفات غير المعروفة **أخطاء**.
- إذا كانت plugin مثبتة لكن manifest أو schema الخاصة بها مكسورة أو مفقودة،
  يفشل التحقق ويبلّغ Doctor عن خطأ plugin.
- إذا كان تكوين plugin موجودًا لكن plugin **معطلة**، فيتم الاحتفاظ بالتكوين
  ويُعرَض **تحذير** في Doctor + السجلات.

راجع [Configuration reference](/gateway/configuration) للحصول على schema الكاملة لـ `plugins.*`.

## ملاحظات

- manifest **مطلوبة للـ plugins الأصلية في OpenClaw**، بما في ذلك التحميلات المحلية من نظام الملفات.
- ما يزال وقت التشغيل يحمّل وحدة plugin بشكل منفصل؛ وتُستخدم manifest فقط
  للاكتشاف + التحقق.
- تُحلل manifests الأصلية باستخدام JSON5، لذا تُقبل التعليقات والفواصل اللاحقة والمفاتيح غير المقتبسة
  ما دام أن القيمة النهائية ما تزال كائنًا.
- لا يقرأ محمّل manifest إلا الحقول الموثقة. تجنّب إضافة
  مفاتيح عليا مخصصة هنا.
- يمثّل `providerAuthEnvVars` مسار بيانات التعريف الرخيصة لفحوصات المصادقة،
  والتحقق من علامات env، والأسطح المشابهة لمصادقة المزوّد التي لا ينبغي أن
  تشغّل وقت تشغيل plugin فقط لفحص أسماء env.
- يمثّل `providerAuthChoices` مسار بيانات التعريف الرخيصة لمحددات auth-choice،
  وتحليل `--auth-choice`، وربط المزوّد المفضّل، وتسجيل أعلام CLI البسيطة قبل تحميل وقت تشغيل المزوّد. أما بالنسبة إلى بيانات تعريف معالج وقت التشغيل التي تتطلب شيفرة المزوّد، فراجع
  [Provider runtime hooks](/plugins/architecture#provider-runtime-hooks).
- يتم اختيار أنواع plugins الحصرية عبر `plugins.slots.*`.
  - يُختار `kind: "memory"` بواسطة `plugins.slots.memory`.
  - يُختار `kind: "context-engine"` بواسطة `plugins.slots.contextEngine`
    (والافتراضي: `legacy` المضمّنة).
- يمكن حذف `channels` و`providers` و`cliBackends` و`skills` عندما لا
  تحتاج plugin إليها.
- إذا كانت plugin تعتمد على وحدات أصلية، فوثّق خطوات البناء وأي
  متطلبات لقائمة السماح الخاصة بمدير الحزم (مثل pnpm `allow-build-scripts`
  - `pnpm rebuild <package>`).

## ذو صلة

- [Building Plugins](/plugins/building-plugins) — البدء مع plugins
- [Plugin Architecture](/plugins/architecture) — البنية الداخلية
- [SDK Overview](/plugins/sdk-overview) — مرجع Plugin SDK
