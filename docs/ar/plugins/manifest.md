---
read_when:
    - أنت تبني إضافة OpenClaw
    - تحتاج إلى شحن مخطط config لإضافة أو تصحيح أخطاء التحقق من الإضافة
summary: متطلبات plugin manifest ومخطط JSON (تحقق صارم من config)
title: Plugin Manifest
x-i18n:
    generated_at: "2026-04-06T03:10:08Z"
    model: gpt-5.4
    provider: openai
    source_hash: f6f915a761cdb5df77eba5d2ccd438c65445bd2ab41b0539d1200e63e8cf2c3a
    source_path: plugins/manifest.md
    workflow: 15
---

# Plugin manifest (`openclaw.plugin.json`)

هذه الصفحة مخصّصة **لـ plugin manifest الأصلي في OpenClaw** فقط.

للاطلاع على تخطيطات الحِزم المتوافقة، راجع [حِزم الإضافات](/ar/plugins/bundles).

تستخدم تنسيقات الحِزم المتوافقة ملفات manifest مختلفة:

- حزمة Codex: `.codex-plugin/plugin.json`
- حزمة Claude: `.claude-plugin/plugin.json` أو تخطيط مكوّن Claude الافتراضي
  من دون manifest
- حزمة Cursor: `.cursor-plugin/plugin.json`

يكتشف OpenClaw تلك التخطيطات المتوافقة للحِزم تلقائيًا أيضًا، لكنه لا يتحقق منها
مقابل مخطط `openclaw.plugin.json` الموضّح هنا.

بالنسبة إلى الحِزم المتوافقة، يقرأ OpenClaw حاليًا بيانات الحزمة الوصفية بالإضافة إلى
جذور Skills المعلنة، وجذور أوامر Claude، والقيم الافتراضية `settings.json` لحزمة Claude،
وقيم Claude bundle LSP الافتراضية، وحِزم hook المدعومة عندما يطابق التخطيط
توقعات وقت تشغيل OpenClaw.

يجب أن تُرفق كل إضافة OpenClaw أصلية ملف `openclaw.plugin.json` في
**جذر الإضافة**. يستخدم OpenClaw هذا الـ manifest للتحقق من config
**من دون تنفيذ شيفرة الإضافة**. وتُعامل ملفات manifest المفقودة أو غير الصالحة على أنها
أخطاء في الإضافة وتمنع التحقق من config.

راجع دليل نظام الإضافات الكامل: [Plugins](/ar/tools/plugin).
وللاطلاع على نموذج الإمكانات الأصلي وإرشادات التوافق الخارجي الحالية:
[نموذج الإمكانات](/ar/plugins/architecture#public-capability-model).

## ما الذي يفعله هذا الملف

`openclaw.plugin.json` هو metadata الذي يقرأه OpenClaw قبل أن يحمّل
شيفرة الإضافة الخاصة بك.

استخدمه من أجل:

- هوية الإضافة
- التحقق من config
- بيانات auth وonboarding الوصفية التي يجب أن تكون متاحة من دون تشغيل وقت
  تشغيل الإضافة
- بيانات alias وauto-enable الوصفية التي يجب أن تُحل قبل تحميل وقت تشغيل الإضافة
- بيانات وصفية مختصرة مملوكة لعائلات النماذج يجب أن تفعّل
  الإضافة تلقائيًا قبل تحميل وقت التشغيل
- لقطات ثابتة لملكية الإمكانات تُستخدم في أسلاك التوافق المجمعة
  وتغطية العقود
- بيانات وصفية خاصة بـ config القنوات يجب أن تُدمج في أسطح الفهرسة والتحقق
  من دون تحميل وقت التشغيل
- تلميحات UI الخاصة بـ config

لا تستخدمه من أجل:

- تسجيل سلوك وقت التشغيل
- إعلان code entrypoints
- بيانات npm الوصفية الخاصة بالتثبيت

فهذه تخص شيفرة الإضافة و`package.json`.

## مثال بسيط

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
  "description": "إضافة مزوّد OpenRouter",
  "version": "1.0.0",
  "providers": ["openrouter"],
  "modelSupport": {
    "modelPrefixes": ["router-"]
  },
  "providerAuthEnvVars": {
    "openrouter": ["OPENROUTER_API_KEY"]
  },
  "providerAuthChoices": [
    {
      "provider": "openrouter",
      "method": "api-key",
      "choiceId": "openrouter-api-key",
      "choiceLabel": "مفتاح API لـ OpenRouter",
      "groupId": "openrouter",
      "groupLabel": "OpenRouter",
      "optionKey": "openrouterApiKey",
      "cliFlag": "--openrouter-api-key",
      "cliOption": "--openrouter-api-key <key>",
      "cliDescription": "مفتاح API لـ OpenRouter",
      "onboardingScopes": ["text-inference"]
    }
  ],
  "uiHints": {
    "apiKey": {
      "label": "مفتاح API",
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

## مرجع الحقول ذات المستوى الأعلى

| الحقل                               | مطلوب | النوع                             | ما الذي يعنيه                                                                                                                                                                                                 |
| ----------------------------------- | ------ | --------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`                                | نعم    | `string`                          | معرّف الإضافة القانوني. هذا هو المعرّف المستخدم في `plugins.entries.<id>`.                                                                                                                                   |
| `configSchema`                      | نعم    | `object`                          | مخطط JSON Schema مضمّن لـ config هذه الإضافة.                                                                                                                                                                  |
| `enabledByDefault`                  | لا     | `true`                            | يحدّد أن الإضافة المجمعة مفعلة افتراضيًا. احذف هذا الحقل، أو عيّن أي قيمة لا تساوي `true`، لترك الإضافة معطلة افتراضيًا.                                                                                   |
| `legacyPluginIds`                   | لا     | `string[]`                        | معرّفات قديمة تُطبّع إلى معرّف الإضافة القانوني هذا.                                                                                                                                                          |
| `autoEnableWhenConfiguredProviders` | لا     | `string[]`                        | معرّفات المزوّدين التي ينبغي أن تفعّل هذه الإضافة تلقائيًا عندما تشير auth أو config أو مراجع النماذج إليها.                                                                                                 |
| `kind`                              | لا     | `"memory"` \| `"context-engine"`  | يصرّح عن نوع إضافة حصري يُستخدم بواسطة `plugins.slots.*`.                                                                                                                                                     |
| `channels`                          | لا     | `string[]`                        | معرّفات القنوات المملوكة لهذه الإضافة. تُستخدم للاكتشاف والتحقق من config.                                                                                                                                    |
| `providers`                         | لا     | `string[]`                        | معرّفات المزوّدين المملوكة لهذه الإضافة.                                                                                                                                                                       |
| `modelSupport`                      | لا     | `object`                          | بيانات وصفية مختصرة مملوكة للـ manifest لعائلات النماذج، تُستخدم لتحميل الإضافة تلقائيًا قبل وقت التشغيل.                                                                                                   |
| `providerAuthEnvVars`               | لا     | `Record<string, string[]>`        | بيانات وصفية رخيصة لـ auth المزوّد يمكن لـ OpenClaw فحصها من دون تحميل شيفرة الإضافة.                                                                                                                       |
| `providerAuthChoices`               | لا     | `object[]`                        | بيانات وصفية رخيصة لخيارات auth لمنتقيات onboarding، وحل المزوّد المفضّل، وربط رايات CLI البسيطة.                                                                                                           |
| `contracts`                         | لا     | `object`                          | لقطة ثابتة مجمّعة للإمكانات الخاصة بالكلام، والنسخ الحي الفوري، والصوت الحي الفوري، وفهم الوسائط، وتوليد الصور، وتوليد الموسيقى، وتوليد الفيديو، وweb-fetch، والبحث على الويب، وملكية الأدوات.             |
| `channelConfigs`                    | لا     | `Record<string, object>`          | بيانات وصفية مملوكة للـ manifest لـ config القنوات، تُدمج في أسطح الاكتشاف والتحقق قبل تحميل وقت التشغيل.                                                                                                   |
| `skills`                            | لا     | `string[]`                        | أدلة Skills لتحميلها، نسبةً إلى جذر الإضافة.                                                                                                                                                                   |
| `name`                              | لا     | `string`                          | اسم الإضافة المقروء للبشر.                                                                                                                                                                                     |
| `description`                       | لا     | `string`                          | ملخص قصير يظهر في أسطح الإضافات.                                                                                                                                                                               |
| `version`                           | لا     | `string`                          | إصدار معلوماتي للإضافة.                                                                                                                                                                                        |
| `uiHints`                           | لا     | `Record<string, object>`          | تسميات UI وعناصر نائبة وتلميحات الحساسية لحقول config.                                                                                                                                                         |

## مرجع `providerAuthChoices`

يصف كل إدخال في `providerAuthChoices` خيار onboarding أو auth واحدًا.
يقرأ OpenClaw هذا قبل تحميل وقت تشغيل المزوّد.

| الحقل                 | مطلوب | النوع                                            | ما الذي يعنيه                                                                                           |
| --------------------- | ------ | ------------------------------------------------ | ------------------------------------------------------------------------------------------------------- |
| `provider`            | نعم    | `string`                                         | معرّف المزوّد الذي ينتمي إليه هذا الخيار.                                                               |
| `method`              | نعم    | `string`                                         | معرّف طريقة auth التي سيجري التوجيه إليها.                                                              |
| `choiceId`            | نعم    | `string`                                         | معرّف ثابت لخيار auth يُستخدم بواسطة تدفقات onboarding وCLI.                                           |
| `choiceLabel`         | لا     | `string`                                         | تسمية موجهة للمستخدم. إذا حُذفت، يعود OpenClaw إلى `choiceId`.                                         |
| `choiceHint`          | لا     | `string`                                         | نص مساعد قصير للمنتقي.                                                                                 |
| `assistantPriority`   | لا     | `number`                                         | تُرتَّب القيم الأقل أولًا في المنتقيات التفاعلية التي يقودها المساعد.                                  |
| `assistantVisibility` | لا     | `"visible"` \| `"manual-only"`                   | يُخفي الخيار من منتقيات المساعد مع الاستمرار في السماح بالاختيار اليدوي عبر CLI.                      |
| `deprecatedChoiceIds` | لا     | `string[]`                                       | معرّفات خيارات قديمة يجب أن تعيد توجيه المستخدمين إلى هذا الخيار البديل.                               |
| `groupId`             | لا     | `string`                                         | معرّف مجموعة اختياري لتجميع الخيارات المرتبطة.                                                          |
| `groupLabel`          | لا     | `string`                                         | تسمية موجهة للمستخدم لتلك المجموعة.                                                                    |
| `groupHint`           | لا     | `string`                                         | نص مساعد قصير للمجموعة.                                                                                |
| `optionKey`           | لا     | `string`                                         | مفتاح خيار داخلي لتدفقات auth البسيطة ذات الراية الواحدة.                                              |
| `cliFlag`             | لا     | `string`                                         | اسم راية CLI، مثل `--openrouter-api-key`.                                                              |
| `cliOption`           | لا     | `string`                                         | صيغة خيار CLI الكاملة، مثل `--openrouter-api-key <key>`.                                               |
| `cliDescription`      | لا     | `string`                                         | الوصف المستخدم في مساعدة CLI.                                                                          |
| `onboardingScopes`    | لا     | `Array<"text-inference" \| "image-generation">`  | أي أسطح onboarding يجب أن يظهر فيها هذا الخيار. وإذا حُذف، فالقيمة الافتراضية هي `["text-inference"]`. |

## مرجع `uiHints`

`uiHints` عبارة عن خريطة من أسماء حقول config إلى تلميحات عرض صغيرة.

```json
{
  "uiHints": {
    "apiKey": {
      "label": "مفتاح API",
      "help": "يُستخدم لطلبات OpenRouter",
      "placeholder": "sk-or-v1-...",
      "sensitive": true
    }
  }
}
```

يمكن أن يتضمن تلميح كل حقل ما يلي:

| الحقل         | النوع      | ما الذي يعنيه                      |
| ------------- | ---------- | ---------------------------------- |
| `label`       | `string`   | تسمية الحقل الموجهة للمستخدم.      |
| `help`        | `string`   | نص مساعد قصير.                     |
| `tags`        | `string[]` | وسوم UI اختيارية.                  |
| `advanced`    | `boolean`  | يضع علامة على الحقل بأنه متقدم.    |
| `sensitive`   | `boolean`  | يضع علامة على الحقل بأنه سري أو حساس. |
| `placeholder` | `string`   | نص العنصر النائب لمدخلات النموذج.  |

## مرجع `contracts`

استخدم `contracts` فقط لبيانات وصفية ثابتة لملكية الإمكانات يمكن لـ OpenClaw
قراءتها من دون استيراد وقت تشغيل الإضافة.

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

| الحقل                            | النوع      | ما الذي يعنيه                                                |
| -------------------------------- | ---------- | ------------------------------------------------------------ |
| `speechProviders`                | `string[]` | معرّفات مزوّدي الكلام التي تملكها هذه الإضافة.               |
| `realtimeTranscriptionProviders` | `string[]` | معرّفات مزوّدي النسخ الحي الفوري التي تملكها هذه الإضافة.    |
| `realtimeVoiceProviders`         | `string[]` | معرّفات مزوّدي الصوت الحي الفوري التي تملكها هذه الإضافة.    |
| `mediaUnderstandingProviders`    | `string[]` | معرّفات مزوّدي فهم الوسائط التي تملكها هذه الإضافة.          |
| `imageGenerationProviders`       | `string[]` | معرّفات مزوّدي توليد الصور التي تملكها هذه الإضافة.          |
| `videoGenerationProviders`       | `string[]` | معرّفات مزوّدي توليد الفيديو التي تملكها هذه الإضافة.        |
| `webFetchProviders`              | `string[]` | معرّفات مزوّدي web-fetch التي تملكها هذه الإضافة.            |
| `webSearchProviders`             | `string[]` | معرّفات مزوّدي البحث على الويب التي تملكها هذه الإضافة.      |
| `tools`                          | `string[]` | أسماء أدوات الوكيل التي تملكها هذه الإضافة لفحوصات العقود المجمعة. |

## مرجع `channelConfigs`

استخدم `channelConfigs` عندما تحتاج إضافة قناة إلى بيانات وصفية رخيصة لـ config قبل
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
          "label": "URL الخادم الرئيسي",
          "placeholder": "https://matrix.example.com"
        }
      },
      "label": "Matrix",
      "description": "اتصال بخادم Matrix الرئيسي",
      "preferOver": ["matrix-legacy"]
    }
  }
}
```

يمكن أن يتضمن كل إدخال قناة ما يلي:

| الحقل         | النوع                     | ما الذي يعنيه                                                                 |
| ------------- | ------------------------ | ----------------------------------------------------------------------------- |
| `schema`      | `object`                 | JSON Schema لـ `channels.<id>`. وهو مطلوب لكل إدخال config قناة مُعلن عنه.   |
| `uiHints`     | `Record<string, object>` | تسميات UI/العناصر النائبة/تلميحات الحساسية الاختيارية لذلك القسم من config القناة. |
| `label`       | `string`                 | تسمية القناة التي تُدمج في أسطح المنتقي والفحص عندما لا تكون بيانات وقت التشغيل الوصفية جاهزة. |
| `description` | `string`                 | وصف قصير للقناة لأسطح الفحص والفهرس.                                          |
| `preferOver`  | `string[]`               | معرّفات إضافات قديمة أو أقل أولوية ينبغي أن تتفوق عليها هذه القناة في أسطح الاختيار. |

## مرجع `modelSupport`

استخدم `modelSupport` عندما ينبغي لـ OpenClaw استنتاج إضافة المزوّد الخاصة بك من
معرّفات نماذج مختصرة مثل `gpt-5.4` أو `claude-sonnet-4.6` قبل أن يُحمَّل
وقت تشغيل الإضافة.

```json
{
  "modelSupport": {
    "modelPrefixes": ["gpt-", "o1", "o3", "o4"],
    "modelPatterns": ["^computer-use-preview"]
  }
}
```

يطبّق OpenClaw ترتيب الأولوية هذا:

- تستخدم مراجع `provider/model` الصريحة بيانات `providers` الوصفية المملوكة للـ manifest
- تتفوق `modelPatterns` على `modelPrefixes`
- إذا طابقت إضافة غير مجمعة وأخرى مجمعة في الوقت نفسه، تفوز الإضافة غير المجمعة
- يُتجاهل الغموض المتبقي إلى أن يحدد المستخدم أو config مزوّدًا

الحقول:

| الحقل           | النوع      | ما الذي يعنيه                                                                   |
| --------------- | ---------- | ------------------------------------------------------------------------------- |
| `modelPrefixes` | `string[]` | بادئات تُطابق باستخدام `startsWith` مع معرّفات النماذج المختصرة.               |
| `modelPatterns` | `string[]` | مصادر Regex تُطابق مع معرّفات النماذج المختصرة بعد إزالة لاحقات الملف الشخصي. |

مفاتيح الإمكانات القديمة ذات المستوى الأعلى مهجورة. استخدم `openclaw doctor --fix` من أجل
نقل `speechProviders` و`realtimeTranscriptionProviders`،
و`realtimeVoiceProviders` و`mediaUnderstandingProviders`،
و`imageGenerationProviders` و`videoGenerationProviders`،
و`webFetchProviders` و`webSearchProviders` تحت `contracts`؛ إذ لم يعد
تحميل الـ manifest العادي يعامل تلك الحقول ذات المستوى الأعلى على أنها
ملكية للإمكانات.

## Manifest مقابل package.json

يخدم الملفان مهمتين مختلفتين:

| الملف                  | استخدمه من أجل                                                                                                                |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `openclaw.plugin.json` | الاكتشاف، والتحقق من config، وبيانات وصفية لخيارات auth، وتلميحات UI التي يجب أن تكون موجودة قبل تشغيل شيفرة الإضافة        |
| `package.json`         | بيانات npm الوصفية، وتثبيت الاعتماديات، وكتلة `openclaw` المستخدمة لنقاط الإدخال، أو بوابات التثبيت، أو الإعداد، أو بيانات الفهرس الوصفية |

إذا لم تكن متأكدًا أين تنتمي قطعة من البيانات الوصفية، فاستخدم هذه القاعدة:

- إذا كان يجب على OpenClaw معرفتها قبل تحميل شيفرة الإضافة، فضعها في `openclaw.plugin.json`
- وإذا كانت تتعلق بالتغليف، أو ملفات الإدخال، أو سلوك تثبيت npm، فضعها في `package.json`

### حقول `package.json` التي تؤثر في الاكتشاف

تعيش بعض بيانات الإضافات الوصفية السابقة لوقت التشغيل عمدًا في `package.json` تحت
كتلة `openclaw` بدلًا من `openclaw.plugin.json`.

أمثلة مهمة:

| الحقل                                                             | ما الذي يعنيه                                                                                                                                  |
| ----------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.extensions`                                             | يعلن عن entrypoints للإضافات الأصلية.                                                                                                         |
| `openclaw.setupEntry`                                             | entrypoint خفيف مخصص للإعداد فقط، يُستخدم أثناء onboarding وبدء القنوات المؤجل.                                                               |
| `openclaw.channel`                                                | بيانات وصفية رخيصة لفهرس القنوات مثل التسميات، ومسارات المستندات، والأسماء البديلة، ونسخة الاختيار.                                          |
| `openclaw.channel.configuredState`                                | بيانات وصفية خفيفة لفاحص حالة التهيئة يمكنها الإجابة عن سؤال "هل يوجد إعداد يعتمد على env فقط بالفعل؟" من دون تحميل وقت تشغيل القناة الكامل. |
| `openclaw.channel.persistedAuthState`                             | بيانات وصفية خفيفة لفاحص auth المخزن يمكنها الإجابة عن سؤال "هل هناك أي تسجيل دخول موجود بالفعل؟" من دون تحميل وقت تشغيل القناة الكامل.     |
| `openclaw.install.npmSpec` / `openclaw.install.localPath`         | تلميحات تثبيت/تحديث للإضافات المجمعة والمنشورة خارجيًا.                                                                                        |
| `openclaw.install.defaultChoice`                                  | مسار التثبيت المفضّل عندما تتوفر مصادر تثبيت متعددة.                                                                                           |
| `openclaw.install.minHostVersion`                                 | الحد الأدنى المدعوم لإصدار مضيف OpenClaw، باستخدام حد semver أدنى مثل `>=2026.3.22`.                                                          |
| `openclaw.install.allowInvalidConfigRecovery`                     | يسمح بمسار استرداد ضيق لإعادة تثبيت إضافة مجمعة عندما يكون config غير صالح.                                                                    |
| `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen` | يتيح تحميل أسطح القناة الخاصة بالإعداد فقط قبل الإضافة الكاملة للقناة أثناء بدء التشغيل.                                                       |

يُفرض `openclaw.install.minHostVersion` أثناء التثبيت وتحميل سجل الـ manifest.
تُرفض القيم غير الصالحة؛ أما القيم الصالحة الأحدث فتتجاوز
الإضافة على المضيفات الأقدم.

إن `openclaw.install.allowInvalidConfigRecovery` ضيق عمدًا. فهو
لا يجعل إعدادات config المعطوبة العشوائية قابلة للتثبيت. واليوم لا يسمح إلا
لتدفقات التثبيت بالتعافي من إخفاقات ترقية محددة قديمة في الإضافات المجمعة، مثل
مسار إضافة مجمعة مفقود أو إدخال `channels.<id>` قديم لتلك الإضافة
المجمعة نفسها. أما أخطاء config غير المرتبطة فلا تزال تمنع التثبيت وتوجّه المشغّلين
إلى `openclaw doctor --fix`.

يمثل `openclaw.channel.persistedAuthState` بيانات package الوصفية لوحدة فاحص
صغيرة:

```json
{
  "openclaw": {
    "channel": {
      "id": "whatsapp",
      "persistedAuthState": {
        "specifier": "./auth-presence",
        "exportName": "hasAnyWhatsAppAuth"
      }
    }
  }
}
```

استخدمه عندما تحتاج تدفقات الإعداد أو doctor أو حالة التهيئة إلى مسبار auth بسيط
بنعم/لا قبل تحميل إضافة القناة الكاملة. وينبغي أن يكون export المستهدف دالة صغيرة
تقرأ الحالة المخزنة فقط؛ ولا تمرره عبر شريط وقت تشغيل القناة الكامل.

يتبع `openclaw.channel.configuredState` البنية نفسها من أجل فحوصات تهيئة رخيصة
تعتمد على env فقط:

```json
{
  "openclaw": {
    "channel": {
      "id": "telegram",
      "configuredState": {
        "specifier": "./configured-state",
        "exportName": "hasTelegramConfiguredState"
      }
    }
  }
}
```

استخدمه عندما تكون القناة قادرة على الإجابة عن حالة التهيئة انطلاقًا من env أو من
مدخلات صغيرة غير وقت التشغيل. وإذا كان الفحص يتطلب تحليل config الكامل أو
وقت تشغيل القناة الحقيقي، فأبقِ هذا المنطق داخل hook الإضافة `config.hasConfiguredState` بدلًا من ذلك.

## متطلبات JSON Schema

- **يجب أن تُرفق كل إضافة JSON Schema**، حتى لو كانت لا تقبل أي config.
- يُقبل مخطط فارغ (مثلًا `{ "type": "object", "additionalProperties": false }`).
- يجري التحقق من المخططات في وقت قراءة/كتابة config، وليس في وقت التشغيل.

## سلوك التحقق

- مفاتيح `channels.*` غير المعروفة هي **أخطاء**، ما لم يكن معرّف القناة معلنًا بواسطة
  manifest لإضافة.
- يجب أن تشير `plugins.entries.<id>` و`plugins.allow` و`plugins.deny` و`plugins.slots.*`
  إلى معرّفات إضافات **قابلة للاكتشاف**. المعرفات غير المعروفة هي **أخطاء**.
- إذا كانت الإضافة مثبّتة لكن manifest الخاص بها أو schema فيه مكسور أو مفقود،
  فسيفشل التحقق وسيبلّغ Doctor عن خطأ الإضافة.
- إذا كان config الإضافة موجودًا لكن الإضافة **معطلة**، فسيُحتفَظ بـ config
  وتظهر **تحذيرات** في Doctor + السجلات.

راجع [مرجع Configuration](/ar/gateway/configuration) للاطلاع على مخطط `plugins.*` الكامل.

## ملاحظات

- إن الـ manifest **مطلوب للإضافات الأصلية في OpenClaw**، بما في ذلك التحميلات من نظام الملفات المحلي.
- لا يزال وقت التشغيل يحمّل وحدة الإضافة بشكل منفصل؛ فالـ manifest مخصّص فقط
  للاكتشاف + التحقق.
- تُحلل manifest الأصلية باستخدام JSON5، لذا تُقبل التعليقات والفواصل اللاحقة
  والمفاتيح غير المحاطة بعلامات اقتباس ما دامت القيمة النهائية لا تزال كائنًا.
- لا يقرأ محمّل الـ manifest إلا الحقول الموثقة في الـ manifest. تجنّب إضافة
  مفاتيح مخصصة ذات مستوى أعلى هنا.
- يمثل `providerAuthEnvVars` مسار البيانات الوصفية الرخيص لفحوصات auth، والتحقق من
  علامات env، والأسطح المشابهة لـ auth المزوّد التي لا ينبغي أن تشغّل وقت تشغيل الإضافة
  لمجرد فحص أسماء env.
- يمثل `providerAuthChoices` مسار البيانات الوصفية الرخيص لمنتقيات خيارات auth،
  وحل `--auth-choice`، وتعيين المزوّد المفضّل، وتسجيل رايات CLI البسيطة الخاصة بـ onboarding
  قبل تحميل وقت تشغيل المزوّد. أما بالنسبة إلى بيانات metadata الخاصة بمعالج وقت التشغيل
  التي تتطلب شيفرة المزوّد، فراجع
  [Provider runtime hooks](/ar/plugins/architecture#provider-runtime-hooks).
- تُختار أنواع الإضافات الحصرية عبر `plugins.slots.*`.
  - يُختار `kind: "memory"` بواسطة `plugins.slots.memory`.
  - يُختار `kind: "context-engine"` بواسطة `plugins.slots.contextEngine`
    (الافتراضي: `legacy` المضمّن).
- يمكن حذف `channels` و`providers` و`skills` عندما
  لا تحتاج الإضافة إليها.
- إذا كانت إضافتك تعتمد على وحدات أصلية، فوثّق خطوات البناء وأي
  متطلبات قائمة سماح لمدير الحزم (مثل pnpm `allow-build-scripts`
  - `pnpm rebuild <package>`).

## ذو صلة

- [بناء الإضافات](/ar/plugins/building-plugins) — البدء باستخدام الإضافات
- [بنية الإضافات](/ar/plugins/architecture) — البنية الداخلية
- [نظرة عامة على SDK](/ar/plugins/sdk-overview) — مرجع Plugin SDK
