---
read_when:
    - تمكين تحويل النص إلى كلام للردود
    - تهيئة موفري TTS أو الحدود
    - استخدام أوامر `/tts`
summary: تحويل النص إلى كلام (TTS) للردود الصادرة
title: تحويل النص إلى كلام (المسار القديم)
x-i18n:
    generated_at: "2026-04-05T13:00:56Z"
    model: gpt-5.4
    provider: openai
    source_hash: acca61773996299a582ab88e5a5db12d8f22ce8a28292ce97cc5dd5fdc2d3b83
    source_path: tts.md
    workflow: 15
---

# تحويل النص إلى كلام (TTS)

يمكن لـ OpenClaw تحويل الردود الصادرة إلى صوت باستخدام ElevenLabs أو Microsoft أو MiniMax أو OpenAI.
ويعمل هذا في أي مكان يستطيع OpenClaw إرسال الصوت إليه.

## الخدمات المدعومة

- **ElevenLabs** (الموفّر الأساسي أو الاحتياطي)
- **Microsoft** (الموفّر الأساسي أو الاحتياطي؛ يستخدم التنفيذ المضمّن الحالي `node-edge-tts`)
- **MiniMax** (الموفّر الأساسي أو الاحتياطي؛ يستخدم واجهة T2A v2 API)
- **OpenAI** (الموفّر الأساسي أو الاحتياطي؛ ويُستخدم أيضًا للملخصات)

### ملاحظات حول Microsoft speech

يستخدم موفّر Microsoft speech المضمّن حاليًا خدمة TTS العصبية عبر الإنترنت من Microsoft Edge
من خلال مكتبة `node-edge-tts`. وهي خدمة مستضافة (وليست
محلية)، وتستخدم نقاط نهاية Microsoft، ولا تتطلب مفتاح API.
تكشف `node-edge-tts` خيارات إعداد الكلام وتنسيقات الإخراج، لكن
ليست كل الخيارات مدعومة من الخدمة. لا تزال المدخلات القديمة للتهيئة والتوجيه
باستخدام `edge` تعمل ويجري توحيدها إلى `microsoft`.

ولأن هذا المسار عبارة عن خدمة ويب عامة من دون SLA أو حصة منشورة،
فتعامل معه على أنه أفضل جهد. وإذا كنت تحتاج إلى حدود ودعم مضمونين، فاستخدم OpenAI
أو ElevenLabs.

## المفاتيح الاختيارية

إذا كنت تريد OpenAI أو ElevenLabs أو MiniMax:

- `ELEVENLABS_API_KEY` (أو `XI_API_KEY`)
- `MINIMAX_API_KEY`
- `OPENAI_API_KEY`

لا تتطلب Microsoft speech مفتاح API.

إذا جرى إعداد عدة موفّرين، فسيُستخدم الموفّر المحدد أولًا ويُستخدم الآخرون كخيارات احتياطية.
يستخدم التلخيص التلقائي `summaryModel` المُعدّ (أو `agents.defaults.model.primary`)،
لذلك يجب أيضًا مصادقة ذلك الموفّر إذا فعّلت الملخصات.

## روابط الخدمات

- [دليل OpenAI لتحويل النص إلى كلام](https://platform.openai.com/docs/guides/text-to-speech)
- [مرجع OpenAI Audio API](https://platform.openai.com/docs/api-reference/audio)
- [تحويل النص إلى كلام في ElevenLabs](https://elevenlabs.io/docs/api-reference/text-to-speech)
- [المصادقة في ElevenLabs](https://elevenlabs.io/docs/api-reference/authentication)
- [واجهة MiniMax T2A v2 API](https://platform.minimaxi.com/document/T2A%20V2)
- [node-edge-tts](https://github.com/SchneeHertz/node-edge-tts)
- [تنسيقات إخراج Microsoft Speech](https://learn.microsoft.com/azure/ai-services/speech-service/rest-text-to-speech#audio-outputs)

## هل هو مفعّل افتراضيًا؟

لا. يكون Auto‑TTS **معطّلًا** افتراضيًا. فعّله في التهيئة باستخدام
`messages.tts.auto` أو لكل جلسة باستخدام `/tts always` (الاسم البديل: `/tts on`).

عندما لا تكون `messages.tts.provider` معيّنة، يختار OpenClaw أول
موفّر speech مُعدّ حسب ترتيب الاختيار التلقائي في السجل.

## التهيئة

توجد تهيئة TTS ضمن `messages.tts` في `openclaw.json`.
المخطط الكامل موجود في [تهيئة Gateway](/ar/gateway/configuration).

### تهيئة دنيا (تمكين + موفّر)

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

### حدود مخصصة + مسار prefs

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

### ملاحظات على الحقول

- `auto`: وضع Auto‑TTS (`off` أو `always` أو `inbound` أو `tagged`).
  - يرسل `inbound` الصوت فقط بعد رسالة صوتية واردة.
  - يرسل `tagged` الصوت فقط عندما يتضمن الرد وسوم `[[tts]]`.
- `enabled`: مفتاح تشغيل قديم (يقوم doctor بترحيله إلى `auto`).
- `mode`: `"final"` (الافتراضي) أو `"all"` (يتضمن ردود الأدوات/الكتل).
- `provider`: معرّف موفّر speech مثل `"elevenlabs"` أو `"microsoft"` أو `"minimax"` أو `"openai"` (يكون fallback تلقائيًا).
- إذا لم يكن `provider` **معيّنًا**، يستخدم OpenClaw أول موفّر speech مُعدّ حسب ترتيب الاختيار التلقائي في السجل.
- لا يزال `provider: "edge"` القديم يعمل ويُوحَّد إلى `microsoft`.
- `summaryModel`: نموذج منخفض الكلفة اختياري للتلخيص التلقائي؛ افتراضيًا `agents.defaults.model.primary`.
  - يقبل `provider/model` أو اسمًا بديلًا لنموذج مُعدّ.
- `modelOverrides`: يسمح للنموذج بإصدار توجيهات TTS (مفعّل افتراضيًا).
  - تكون القيمة الافتراضية لـ `allowProvider` هي `false` (ويكون تبديل الموفّر عبر الاشتراك الاختياري).
- `providers.<id>`: إعدادات يملكها الموفّر ومفاتيحها هي معرّف موفّر speech.
- تُرحَّل تلقائيًا عند التحميل كتل الموفّر المباشرة القديمة (`messages.tts.openai` و`messages.tts.elevenlabs` و`messages.tts.microsoft` و`messages.tts.edge`) إلى `messages.tts.providers.<id>`.
- `maxTextLength`: حد أقصى صارم لمدخل TTS (أحرف). يفشل `/tts audio` إذا جرى تجاوزه.
- `timeoutMs`: مهلة الطلب (مللي ثانية).
- `prefsPath`: تجاوز مسار JSON المحلي لـ prefs (الموفّر/الحد/الملخص).
- تعود قيم `apiKey` إلى متغيرات env (`ELEVENLABS_API_KEY`/`XI_API_KEY` و`MINIMAX_API_KEY` و`OPENAI_API_KEY`).
- `providers.elevenlabs.baseUrl`: تجاوز عنوان URL الأساسي لـ ElevenLabs API.
- `providers.openai.baseUrl`: تجاوز نقطة نهاية OpenAI TTS.
  - ترتيب التحليل: `messages.tts.providers.openai.baseUrl` -> `OPENAI_TTS_BASE_URL` -> `https://api.openai.com/v1`
  - تُعامَل القيم غير الافتراضية على أنها نقاط نهاية TTS متوافقة مع OpenAI، لذلك تُقبل أسماء النماذج والأصوات المخصصة.
- `providers.elevenlabs.voiceSettings`:
  - `stability` و`similarityBoost` و`style`: ‏`0..1`
  - `useSpeakerBoost`: ‏`true|false`
  - `speed`: ‏`0.5..2.0` (1.0 = عادي)
- `providers.elevenlabs.applyTextNormalization`: ‏`auto|on|off`
- `providers.elevenlabs.languageCode`: رمز ISO 639-1 من حرفين (مثل `en` و`de`)
- `providers.elevenlabs.seed`: عدد صحيح `0..4294967295` (حتمية بأفضل جهد)
- `providers.minimax.baseUrl`: تجاوز عنوان URL الأساسي لـ MiniMax API (الافتراضي `https://api.minimax.io`، env: ‏`MINIMAX_API_HOST`).
- `providers.minimax.model`: نموذج TTS (الافتراضي `speech-2.8-hd`، env: ‏`MINIMAX_TTS_MODEL`).
- `providers.minimax.voiceId`: معرّف الصوت (الافتراضي `English_expressive_narrator`، env: ‏`MINIMAX_TTS_VOICE_ID`).
- `providers.minimax.speed`: سرعة التشغيل `0.5..2.0` (الافتراضي 1.0).
- `providers.minimax.vol`: مستوى الصوت `(0, 10]` (الافتراضي 1.0؛ ويجب أن يكون أكبر من 0).
- `providers.minimax.pitch`: تغيير النغمة `-12..12` (الافتراضي 0).
- `providers.microsoft.enabled`: السماح باستخدام Microsoft speech (الافتراضي `true`؛ من دون مفتاح API).
- `providers.microsoft.voice`: اسم الصوت العصبي من Microsoft (مثل `en-US-MichelleNeural`).
- `providers.microsoft.lang`: رمز اللغة (مثل `en-US`).
- `providers.microsoft.outputFormat`: تنسيق إخراج Microsoft (مثل `audio-24khz-48kbitrate-mono-mp3`).
  - راجع تنسيقات إخراج Microsoft Speech لمعرفة القيم الصالحة؛ فليست كل التنسيقات مدعومة من النقل المضمّن المعتمد على Edge.
- `providers.microsoft.rate` / `providers.microsoft.pitch` / `providers.microsoft.volume`: سلاسل نسب مئوية (مثل `+10%` و`-5%`).
- `providers.microsoft.saveSubtitles`: كتابة ترجمات JSON بجانب ملف الصوت.
- `providers.microsoft.proxy`: عنوان URL للوكيل لطلبات Microsoft speech.
- `providers.microsoft.timeoutMs`: تجاوز مهلة الطلب (مللي ثانية).
- `edge.*`: اسم بديل قديم لإعدادات Microsoft نفسها.

## التجاوزات المعتمدة على النموذج (مفعّلة افتراضيًا)

افتراضيًا، **يمكن** للنموذج إصدار توجيهات TTS لرد واحد.
عندما تكون `messages.tts.auto` هي `tagged`، تكون هذه التوجيهات مطلوبة لتشغيل الصوت.

عند التمكين، يمكن للنموذج إصدار توجيهات `[[tts:...]]` لتجاوز الصوت
لرد واحد، بالإضافة إلى كتلة اختيارية `[[tts:text]]...[[/tts:text]]` من أجل
توفير وسوم تعبيرية (الضحك، وإشارات الغناء، وما إلى ذلك) يجب أن تظهر فقط في
الصوت.

يتم تجاهل توجيهات `provider=...` ما لم تكن `modelOverrides.allowProvider: true`.

مثال على حمولة الرد:

```
ها أنت ذا.

[[tts:voiceId=pMsXgVXv3BLzUgSXRplE model=eleven_v3 speed=1.1]]
[[tts:text]](يضحك) اقرأ الأغنية مرة أخرى.[[/tts:text]]
```

مفاتيح التوجيه المتاحة (عند التمكين):

- `provider` (معرّف موفّر speech مسجل، مثل `openai` أو `elevenlabs` أو `minimax` أو `microsoft`؛ ويتطلب `allowProvider: true`)
- `voice` (صوت OpenAI) أو `voiceId` (في ElevenLabs / MiniMax)
- `model` (نموذج OpenAI TTS، أو model id في ElevenLabs، أو نموذج MiniMax)
- `stability` و`similarityBoost` و`style` و`speed` و`useSpeakerBoost`
- `vol` / `volume` (مستوى صوت MiniMax، ‏0-10)
- `pitch` (نغمة MiniMax، ‏-12 إلى 12)
- `applyTextNormalization` (`auto|on|off`)
- `languageCode` (ISO 639-1)
- `seed`

لتعطيل جميع تجاوزات النموذج:

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

allowlist اختيارية (لتمكين تبديل الموفّر مع الإبقاء على بقية الإعدادات قابلة للتهيئة):

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

## تفضيلات لكل مستخدم

تكتب أوامر الشرطة المائلة تجاوزات محلية إلى `prefsPath` (الافتراضي:
`~/.openclaw/settings/tts.json`، ويمكن تجاوزه بـ `OPENCLAW_TTS_PREFS` أو
`messages.tts.prefsPath`).

الحقول المخزنة:

- `enabled`
- `provider`
- `maxLength` (حد التلخيص؛ الافتراضي 1500 حرفًا)
- `summarize` (الافتراضي `true`)

وتتجاوز هذه الحقول `messages.tts.*` على ذلك المضيف.

## تنسيقات الإخراج (ثابتة)

- **Feishu / Matrix / Telegram / WhatsApp**: رسالة صوتية Opus (`opus_48000_64` من ElevenLabs، و`opus` من OpenAI).
  - يُعد 48kHz / 64kbps توازنًا جيدًا للرسائل الصوتية.
- **القنوات الأخرى**: MP3 (`mp3_44100_128` من ElevenLabs، و`mp3` من OpenAI).
  - يُعد 44.1kHz / 128kbps التوازن الافتراضي لوضوح الكلام.
- **MiniMax**: ‏MP3 (نموذج `speech-2.8-hd`، بمعدل عينة 32kHz). لا يكون تنسيق الملاحظات الصوتية مدعومًا أصلًا؛ استخدم OpenAI أو ElevenLabs إذا كنت تحتاج إلى رسائل صوتية Opus مضمونة.
- **Microsoft**: يستخدم `microsoft.outputFormat` (الافتراضي `audio-24khz-48kbitrate-mono-mp3`).
  - يقبل النقل المضمّن قيمة `outputFormat`، لكن ليست كل التنسيقات متاحة من الخدمة.
  - تتبع قيم تنسيق الإخراج تنسيقات إخراج Microsoft Speech (بما في ذلك Ogg/WebM Opus).
  - يقبل `sendVoice` في Telegram تنسيقات OGG/MP3/M4A؛ استخدم OpenAI/ElevenLabs إذا كنت تحتاج
    إلى رسائل صوتية Opus مضمونة.
  - إذا فشل تنسيق إخراج Microsoft المُعدّ، يعيد OpenClaw المحاولة باستخدام MP3.

تكون تنسيقات إخراج OpenAI/ElevenLabs ثابتة لكل قناة (انظر أعلاه).

## سلوك Auto-TTS

عند التمكين، يقوم OpenClaw بما يلي:

- يتجاوز TTS إذا كان الرد يحتوي أصلًا على وسائط أو توجيه `MEDIA:`.
- يتجاوز الردود القصيرة جدًا (< 10 أحرف).
- يلخص الردود الطويلة عند التمكين باستخدام `agents.defaults.model.primary` (أو `summaryModel`).
- يرفق الصوت المُنشأ بالرد.

إذا تجاوز الرد `maxLength` وكان التلخيص معطّلًا (أو لا يوجد مفتاح API لـ
نموذج التلخيص)،
فسيُتجاوز الصوت ويُرسل الرد النصي العادي.

## مخطط التدفق

```
Reply -> TTS enabled?
  no  -> send text
  yes -> has media / MEDIA: / short?
          yes -> send text
          no  -> length > limit?
                   no  -> TTS -> attach audio
                   yes -> summary enabled?
                            no  -> send text
                            yes -> summarize (summaryModel or agents.defaults.model.primary)
                                      -> TTS -> attach audio
```

## استخدام أوامر الشرطة المائلة

يوجد أمر واحد: `/tts`.
راجع [أوامر الشرطة المائلة](/tools/slash-commands) للحصول على تفاصيل التمكين.

ملاحظة Discord: ‏`/tts` هو أمر مضمّن في Discord، لذلك يسجّل OpenClaw
الأمر الأصلي `/voice` هناك. ولا يزال النص `/tts ...` يعمل.

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

- تتطلب الأوامر مُرسِلًا مخوّلًا (ولا تزال قواعد allowlist/المالك سارية).
- يجب أن يكون `commands.text` أو تسجيل الأوامر الأصلية مفعّلًا.
- `off|always|inbound|tagged` هي مفاتيح تبديل لكل جلسة (`/tts on` هو اسم بديل لـ `/tts always`).
- يُخزَّن `limit` و`summary` في prefs المحلية، وليس في التهيئة الرئيسية.
- يُنشئ `/tts audio` ردًا صوتيًا لمرة واحدة (ولا يفعّل TTS).
- يتضمن `/tts status` رؤية fallback لآخر محاولة:
  - fallback ناجح: `Fallback: <primary> -> <used>` بالإضافة إلى `Attempts: ...`
  - فشل: `Error: ...` بالإضافة إلى `Attempts: ...`
  - تشخيصات مفصلة: `Attempt details: provider:outcome(reasonCode) latency`
- تتضمن حالات فشل OpenAI وElevenLabs API الآن تفاصيل خطأ الموفّر المُحلَّلة ومعرّف الطلب (عند إرجاعه من الموفّر)، ويظهر ذلك في أخطاء/سجلات TTS.

## أداة العامل

تقوم أداة `tts` بتحويل النص إلى كلام وتعيد مرفقًا صوتيًا من أجل
تسليم الرد. وعندما تكون القناة هي Feishu أو Matrix أو Telegram أو WhatsApp،
يُسلَّم الصوت كرسالة صوتية بدلًا من مرفق ملف.

## Gateway RPC

طرق Gateway:

- `tts.status`
- `tts.enable`
- `tts.disable`
- `tts.convert`
- `tts.setProvider`
- `tts.providers`
