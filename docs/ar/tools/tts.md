---
read_when:
    - تفعيل تحويل النص إلى كلام للردود
    - تهيئة موفري TTS أو الحدود
    - استخدام أوامر /tts
summary: تحويل النص إلى كلام (TTS) للردود الصادرة
title: تحويل النص إلى كلام
x-i18n:
    generated_at: "2026-04-05T13:00:54Z"
    model: gpt-5.4
    provider: openai
    source_hash: 8487c8acef7585bd4eb5e3b39e2a063ebc6b5f0103524abdcbadd3a7781ffc46
    source_path: tools/tts.md
    workflow: 15
---

# تحويل النص إلى كلام (TTS)

يمكن لـ OpenClaw تحويل الردود الصادرة إلى صوت باستخدام ElevenLabs أو Microsoft أو MiniMax أو OpenAI.
وهو يعمل في أي مكان يستطيع OpenClaw فيه إرسال صوت.

## الخدمات المدعومة

- **ElevenLabs** (موفّر أساسي أو احتياطي)
- **Microsoft** (موفّر أساسي أو احتياطي؛ يستخدم التنفيذ المضمّن الحالي `node-edge-tts`)
- **MiniMax** (موفّر أساسي أو احتياطي؛ يستخدم API ‏T2A v2)
- **OpenAI** (موفّر أساسي أو احتياطي؛ ويُستخدم أيضًا للملخصات)

### ملاحظات حول Microsoft speech

يستخدم موفّر Microsoft speech المضمّن حاليًا خدمة TTS العصبية عبر الإنترنت الخاصة بـ Microsoft Edge
من خلال مكتبة `node-edge-tts`. وهي خدمة مستضافة (وليست
محلية)، وتستخدم نقاط نهاية Microsoft، ولا تتطلب مفتاح API.
يعرض `node-edge-tts` خيارات تهيئة الكلام وتنسيقات الإخراج، لكن
ليست كل الخيارات مدعومة من الخدمة. وما زالت تهيئة الإدخال القديمة وتوجيهات
الاستخدام التي تعتمد على `edge` تعمل ويجري تطبيعها إلى `microsoft`.

وبما أن هذا المسار خدمة ويب عامة من دون SLA أو حصة منشورة،
فتعامل معه على أنه أفضل جهد. وإذا كنت تحتاج إلى حدود ودعم مضمونين، فاستخدم OpenAI
أو ElevenLabs.

## المفاتيح الاختيارية

إذا كنت تريد OpenAI أو ElevenLabs أو MiniMax:

- `ELEVENLABS_API_KEY` (أو `XI_API_KEY`)
- `MINIMAX_API_KEY`
- `OPENAI_API_KEY`

لا يتطلب Microsoft speech مفتاح API.

إذا جرى تهيئة عدة موفّرين، فسيُستخدم الموفّر المحدد أولًا وسيكون الآخرون خيارات احتياطية.
يستخدم التلخيص التلقائي `summaryModel` المُهيأ (أو `agents.defaults.model.primary`)،
لذلك يجب أيضًا أن يكون ذلك الموفّر موثّق المصادقة إذا فعّلت الملخصات.

## روابط الخدمات

- [دليل OpenAI لتحويل النص إلى كلام](https://platform.openai.com/docs/guides/text-to-speech)
- [مرجع OpenAI Audio API](https://platform.openai.com/docs/api-reference/audio)
- [تحويل النص إلى كلام في ElevenLabs](https://elevenlabs.io/docs/api-reference/text-to-speech)
- [المصادقة في ElevenLabs](https://elevenlabs.io/docs/api-reference/authentication)
- [MiniMax T2A v2 API](https://platform.minimaxi.com/document/T2A%20V2)
- [node-edge-tts](https://github.com/SchneeHertz/node-edge-tts)
- [تنسيقات إخراج Microsoft Speech](https://learn.microsoft.com/azure/ai-services/speech-service/rest-text-to-speech#audio-outputs)

## هل هو مفعّل افتراضيًا؟

لا. يكون Auto‑TTS **معطّلًا** افتراضيًا. فعّله في التكوين باستخدام
`messages.tts.auto` أو لكل جلسة باستخدام `/tts always` (الاسم المستعار: `/tts on`).

عندما لا يكون `messages.tts.provider` مضبوطًا، يختار OpenClaw أول
موفّر speech مُهيأ وفق ترتيب الاختيار التلقائي في السجل.

## التكوين

يوجد تكوين TTS ضمن `messages.tts` في `openclaw.json`.
المخطط الكامل موجود في [تهيئة Gateway](/ar/gateway/configuration).

### الحد الأدنى من التكوين (تفعيل + موفّر)

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "elevenlabs",
    },
  },
}
```

### OpenAI أساسي مع ElevenLabs احتياطي

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "openai",
      summaryModel: "openai/gpt-4.1-mini",
      modelOverrides: {
        enabled: true,
      },
      providers: {
        openai: {
          apiKey: "openai_api_key",
          baseUrl: "https://api.openai.com/v1",
          model: "gpt-4o-mini-tts",
          voice: "alloy",
        },
        elevenlabs: {
          apiKey: "elevenlabs_api_key",
          baseUrl: "https://api.elevenlabs.io",
          voiceId: "voice_id",
          modelId: "eleven_multilingual_v2",
          seed: 42,
          applyTextNormalization: "auto",
          languageCode: "en",
          voiceSettings: {
            stability: 0.5,
            similarityBoost: 0.75,
            style: 0.0,
            useSpeakerBoost: true,
            speed: 1.0,
          },
        },
      },
    },
  },
}
```

### Microsoft أساسي (من دون مفتاح API)

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "microsoft",
      providers: {
        microsoft: {
          enabled: true,
          voice: "en-US-MichelleNeural",
          lang: "en-US",
          outputFormat: "audio-24khz-48kbitrate-mono-mp3",
          rate: "+10%",
          pitch: "-5%",
        },
      },
    },
  },
}
```

### MiniMax أساسي

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "minimax",
      providers: {
        minimax: {
          apiKey: "minimax_api_key",
          baseUrl: "https://api.minimax.io",
          model: "speech-2.8-hd",
          voiceId: "English_expressive_narrator",
          speed: 1.0,
          vol: 1.0,
          pitch: 0,
        },
      },
    },
  },
}
```

### تعطيل Microsoft speech

```json5
{
  messages: {
    tts: {
      providers: {
        microsoft: {
          enabled: false,
        },
      },
    },
  },
}
```

### حدود مخصصة + مسار التفضيلات

```json5
{
  messages: {
    tts: {
      auto: "always",
      maxTextLength: 4000,
      timeoutMs: 30000,
      prefsPath: "~/.openclaw/settings/tts.json",
    },
  },
}
```

### الرد بالصوت فقط بعد رسالة صوتية واردة

```json5
{
  messages: {
    tts: {
      auto: "inbound",
    },
  },
}
```

### تعطيل التلخيص التلقائي للردود الطويلة

```json5
{
  messages: {
    tts: {
      auto: "always",
    },
  },
}
```

ثم شغّل:

```
/tts summary off
```

### ملاحظات حول الحقول

- `auto`: وضع Auto‑TTS (`off` أو `always` أو `inbound` أو `tagged`).
  - `inbound` يرسل الصوت فقط بعد رسالة صوتية واردة.
  - `tagged` يرسل الصوت فقط عندما يتضمن الرد وسوم `[[tts]]`.
- `enabled`: مفتاح قديم (ينقل `doctor` هذا إلى `auto`).
- `mode`: `"final"` (الافتراضي) أو `"all"` (يتضمن ردود الأدوات/الكتل).
- `provider`: معرّف موفّر speech مثل `"elevenlabs"` أو `"microsoft"` أو `"minimax"` أو `"openai"` (والاحتياطي تلقائي).
- إذا كان `provider` **غير مضبوط**، يستخدم OpenClaw أول موفّر speech مُهيأ وفق ترتيب الاختيار التلقائي في السجل.
- ما زال `provider: "edge"` القديم يعمل ويُطبَّع إلى `microsoft`.
- `summaryModel`: نموذج منخفض الكلفة اختياري للتلخيص التلقائي؛ والإعداد الافتراضي هو `agents.defaults.model.primary`.
  - يقبل `provider/model` أو اسمًا مستعارًا لنموذج مُهيأ.
- `modelOverrides`: السماح للنموذج بإصدار توجيهات TTS (مفعّل افتراضيًا).
  - تكون القيمة الافتراضية لـ `allowProvider` هي `false` (تبديل الموفّر يتطلب تفعيلًا صريحًا).
- `providers.<id>`: إعدادات مملوكة للموفّر ومفهرسة حسب معرّف موفّر speech.
- تُرحَّل تلقائيًا كتل الموفّرين المباشرة القديمة (`messages.tts.openai` و`messages.tts.elevenlabs` و`messages.tts.microsoft` و`messages.tts.edge`) إلى `messages.tts.providers.<id>` عند التحميل.
- `maxTextLength`: حد صارم لإدخال TTS (عدد الأحرف). يفشل `/tts audio` إذا جرى تجاوزه.
- `timeoutMs`: مهلة الطلب (مللي ثانية).
- `prefsPath`: تجاوز لمسار JSON المحلي للتفضيلات (الموفّر/الحد/التلخيص).
- ترجع قيم `apiKey` إلى متغيرات البيئة (`ELEVENLABS_API_KEY`/`XI_API_KEY` و`MINIMAX_API_KEY` و`OPENAI_API_KEY`).
- `providers.elevenlabs.baseUrl`: تجاوز عنوان URL الأساسي لـ API الخاص بـ ElevenLabs.
- `providers.openai.baseUrl`: تجاوز نقطة نهاية OpenAI TTS.
  - ترتيب التحليل: `messages.tts.providers.openai.baseUrl` -> `OPENAI_TTS_BASE_URL` -> `https://api.openai.com/v1`
  - تُعامَل القيم غير الافتراضية على أنها نقاط نهاية TTS متوافقة مع OpenAI، لذلك تُقبل أسماء النماذج والأصوات المخصصة.
- `providers.elevenlabs.voiceSettings`:
  - `stability` و`similarityBoost` و`style`: `0..1`
  - `useSpeakerBoost`: `true|false`
  - `speed`: `0.5..2.0` (1.0 = عادي)
- `providers.elevenlabs.applyTextNormalization`: ‏`auto|on|off`
- `providers.elevenlabs.languageCode`: رمز ISO 639-1 من حرفين (مثل `en` و`de`)
- `providers.elevenlabs.seed`: عدد صحيح `0..4294967295` (حتمية بأفضل جهد)
- `providers.minimax.baseUrl`: تجاوز عنوان URL الأساسي لـ MiniMax API (الافتراضي `https://api.minimax.io`، ومتغير البيئة: `MINIMAX_API_HOST`).
- `providers.minimax.model`: نموذج TTS (الافتراضي `speech-2.8-hd`، ومتغير البيئة: `MINIMAX_TTS_MODEL`).
- `providers.minimax.voiceId`: معرّف الصوت (الافتراضي `English_expressive_narrator`، ومتغير البيئة: `MINIMAX_TTS_VOICE_ID`).
- `providers.minimax.speed`: سرعة التشغيل `0.5..2.0` (الافتراضي 1.0).
- `providers.minimax.vol`: مستوى الصوت `(0, 10]` (الافتراضي 1.0؛ ويجب أن يكون أكبر من 0).
- `providers.minimax.pitch`: إزاحة الطبقة الصوتية `-12..12` (الافتراضي 0).
- `providers.microsoft.enabled`: السماح باستخدام Microsoft speech (الافتراضي `true`؛ من دون مفتاح API).
- `providers.microsoft.voice`: اسم الصوت العصبي من Microsoft (مثل `en-US-MichelleNeural`).
- `providers.microsoft.lang`: رمز اللغة (مثل `en-US`).
- `providers.microsoft.outputFormat`: تنسيق إخراج Microsoft (مثل `audio-24khz-48kbitrate-mono-mp3`).
  - راجع تنسيقات إخراج Microsoft Speech لمعرفة القيم الصالحة؛ ليست كل التنسيقات مدعومة بواسطة وسيلة النقل المضمّنة المعتمدة على Edge.
- `providers.microsoft.rate` / `providers.microsoft.pitch` / `providers.microsoft.volume`: سلاسل نسب مئوية (مثل `+10%` و`-5%`).
- `providers.microsoft.saveSubtitles`: كتابة ترجمات JSON إلى جانب الملف الصوتي.
- `providers.microsoft.proxy`: عنوان URL للبروكسي لطلبات Microsoft speech.
- `providers.microsoft.timeoutMs`: تجاوز مهلة الطلب (مللي ثانية).
- `edge.*`: اسم مستعار قديم لإعدادات Microsoft نفسها.

## التجاوزات المدفوعة بالنموذج (مفعّلة افتراضيًا)

افتراضيًا، **يمكن** للنموذج إصدار توجيهات TTS لرد واحد.
عندما تكون `messages.tts.auto` مضبوطة على `tagged`، تصبح هذه التوجيهات مطلوبة لتشغيل الصوت.

عند التفعيل، يمكن للنموذج إصدار توجيهات `[[tts:...]]` لتجاوز الصوت
لرد واحد، بالإضافة إلى كتلة اختيارية `[[tts:text]]...[[/tts:text]]`
لتوفير وسوم تعبيرية (ضحك، وإشارات غناء، وما إلى ذلك) يجب أن تظهر في
الصوت فقط.

تُتجاهل توجيهات `provider=...` ما لم تكن `modelOverrides.allowProvider: true`.

مثال على حمولة رد:

```
Here you go.

[[tts:voiceId=pMsXgVXv3BLzUgSXRplE model=eleven_v3 speed=1.1]]
[[tts:text]](laughs) Read the song once more.[[/tts:text]]
```

مفاتيح التوجيه المتاحة (عند التفعيل):

- `provider` (معرّف موفّر speech مُسجّل، مثل `openai` أو `elevenlabs` أو `minimax` أو `microsoft`؛ ويتطلب `allowProvider: true`)
- `voice` (صوت OpenAI) أو `voiceId` (ElevenLabs / MiniMax)
- `model` (نموذج OpenAI TTS، أو معرّف نموذج ElevenLabs، أو نموذج MiniMax)
- `stability` و`similarityBoost` و`style` و`speed` و`useSpeakerBoost`
- `vol` / `volume` (مستوى صوت MiniMax، من 0 إلى 10)
- `pitch` (طبقة MiniMax، من -12 إلى 12)
- `applyTextNormalization` (`auto|on|off`)
- `languageCode` (ISO 639-1)
- `seed`

تعطيل جميع تجاوزات النموذج:

```json5
{
  messages: {
    tts: {
      modelOverrides: {
        enabled: false,
      },
    },
  },
}
```

قائمة سماح اختيارية (تفعيل تبديل الموفّر مع إبقاء العناصر الأخرى قابلة للتهيئة):

```json5
{
  messages: {
    tts: {
      modelOverrides: {
        enabled: true,
        allowProvider: true,
        allowSeed: false,
      },
    },
  },
}
```

## التفضيلات لكل مستخدم

تكتب أوامر Slash تجاوزات محلية إلى `prefsPath` (الافتراضي:
`~/.openclaw/settings/tts.json`، مع إمكانية التجاوز بواسطة `OPENCLAW_TTS_PREFS` أو
`messages.tts.prefsPath`).

الحقول المخزنة:

- `enabled`
- `provider`
- `maxLength` (عتبة التلخيص؛ الافتراضي 1500 حرفًا)
- `summarize` (الافتراضي `true`)

وتتجاوز هذه الحقول `messages.tts.*` لذلك المضيف.

## تنسيقات الإخراج (ثابتة)

- **Feishu / Matrix / Telegram / WhatsApp**: رسالة صوتية Opus (`opus_48000_64` من ElevenLabs، و`opus` من OpenAI).
  - يُعد 48kHz / 64kbps توازنًا جيدًا للرسائل الصوتية.
- **القنوات الأخرى**: MP3 (`mp3_44100_128` من ElevenLabs، و`mp3` من OpenAI).
  - يُعد 44.1kHz / 128kbps التوازن الافتراضي لوضوح الكلام.
- **MiniMax**: ‏MP3 (نموذج `speech-2.8-hd`، ومعدل عينة 32kHz). لا يُدعَم تنسيق الملاحظات الصوتية أصلاً؛ استخدم OpenAI أو ElevenLabs إذا كنت تحتاج إلى رسائل صوتية Opus مضمونة.
- **Microsoft**: يستخدم `microsoft.outputFormat` (الافتراضي `audio-24khz-48kbitrate-mono-mp3`).
  - تقبل وسيلة النقل المضمّنة قيمة `outputFormat`، لكن ليست كل التنسيقات متاحة من الخدمة.
  - تتبع قيم تنسيق الإخراج تنسيقات إخراج Microsoft Speech (بما في ذلك Ogg/WebM Opus).
  - يقبل `sendVoice` في Telegram ملفات OGG/MP3/M4A؛ استخدم OpenAI/ElevenLabs إذا كنت تحتاج
    إلى رسائل صوتية Opus مضمونة.
  - إذا فشل تنسيق إخراج Microsoft المُهيأ، يعيد OpenClaw المحاولة باستخدام MP3.

تكون تنسيقات إخراج OpenAI/ElevenLabs ثابتة لكل قناة (انظر أعلاه).

## سلوك Auto-TTS

عند التفعيل، يقوم OpenClaw بما يلي:

- يتخطى TTS إذا كان الرد يحتوي بالفعل على وسائط أو توجيه `MEDIA:`.
- يتخطى الردود القصيرة جدًا (أقل من 10 أحرف).
- يلخّص الردود الطويلة عند التفعيل باستخدام `agents.defaults.model.primary` (أو `summaryModel`).
- يرفق الصوت المُولَّد بالرد.

إذا تجاوز الرد `maxLength` وكان التلخيص معطّلًا (أو لم يوجد مفتاح API
لنموذج التلخيص)،
فسيُتخطى الصوت ويُرسل الرد النصي العادي.

## مخطط التدفق

```
الرد -> هل TTS مفعّل؟
  لا  -> أرسل النص
  نعم -> هل توجد وسائط / MEDIA: / هل الرد قصير؟
          نعم -> أرسل النص
          لا  -> هل الطول > الحد؟
                   لا  -> TTS -> أرفق الصوت
                   نعم -> هل التلخيص مفعّل؟
                            لا  -> أرسل النص
                            نعم -> لخّص (summaryModel أو agents.defaults.model.primary)
                                      -> TTS -> أرفق الصوت
```

## استخدام أوامر Slash

يوجد أمر واحد فقط: `/tts`.
راجع [أوامر Slash](/tools/slash-commands) لمعرفة تفاصيل التفعيل.

ملاحظة Discord: الأمر `/tts` هو أمر مضمّن في Discord، لذلك يسجّل OpenClaw
الأمر `/voice` بوصفه الأمر الأصلي هناك. وما زال النص `/tts ...` يعمل.

```
/tts off
/tts always
/tts inbound
/tts tagged
/tts status
/tts provider openai
/tts limit 2000
/tts summary off
/tts audio Hello from OpenClaw
```

ملاحظات:

- تتطلب الأوامر مرسلًا مصرّحًا له (وما زالت قواعد allowlist/المالك سارية).
- يجب أن يكون `commands.text` أو تسجيل الأوامر الأصلية مفعّلًا.
- `off|always|inbound|tagged` هي مفاتيح تبديل لكل جلسة (`/tts on` اسم مستعار لـ `/tts always`).
- يُخزَّن `limit` و`summary` في التفضيلات المحلية، وليس في التكوين الرئيسي.
- يولّد `/tts audio` ردًا صوتيًا لمرة واحدة (ولا يفعّل TTS).
- يتضمن `/tts status` إظهارًا للاحتياطي لآخر محاولة:
  - احتياطي ناجح: `Fallback: <primary> -> <used>` بالإضافة إلى `Attempts: ...`
  - فشل: `Error: ...` بالإضافة إلى `Attempts: ...`
  - تشخيصات مفصلة: `Attempt details: provider:outcome(reasonCode) latency`
- تتضمن إخفاقات API الخاصة بـ OpenAI وElevenLabs الآن تفاصيل الخطأ التي جرى تحليلها ومعرّف الطلب (عند إرجاعه من الموفّر)، ويجري إظهار ذلك في أخطاء/سجلات TTS.

## أداة الوكيل

تحوّل أداة `tts` النص إلى كلام وتعيد مرفقًا صوتيًا من أجل
تسليم الرد. وعندما تكون القناة هي Feishu أو Matrix أو Telegram أو WhatsApp،
يُسلَّم الصوت بوصفه رسالة صوتية بدلًا من مرفق ملف.

## Gateway RPC

طرائق Gateway:

- `tts.status`
- `tts.enable`
- `tts.disable`
- `tts.convert`
- `tts.setProvider`
- `tts.providers`
