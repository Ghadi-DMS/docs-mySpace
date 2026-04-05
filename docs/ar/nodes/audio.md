---
read_when:
    - تغيير نسخ الصوت أو معالجة الوسائط
summary: كيفية تنزيل الصوت/الرسائل الصوتية الواردة ونسخها وحقنها في الردود
title: الصوت والرسائل الصوتية
x-i18n:
    generated_at: "2026-04-05T12:49:05Z"
    model: gpt-5.4
    provider: openai
    source_hash: dd464df24268b1104c9bbdb6f424ba90747342b4c0f4d2e39d95055708cbd0ae
    source_path: nodes/audio.md
    workflow: 15
---

# الصوت / الرسائل الصوتية (2026-01-17)

## ما الذي يعمل

- **فهم الوسائط (الصوت)**: إذا كان فهم الصوت مفعّلًا (أو تم اكتشافه تلقائيًا)، فإن OpenClaw:
  1. يحدّد أول مرفق صوتي (مسار محلي أو URL) ويقوم بتنزيله إذا لزم الأمر.
  2. يفرض `maxBytes` قبل الإرسال إلى كل إدخال نموذج.
  3. يشغّل أول إدخال نموذج مؤهل بالترتيب (مزوّد أو CLI).
  4. إذا فشل أو تم تخطيه (بسبب الحجم/المهلة)، فإنه يجرّب الإدخال التالي.
  5. عند النجاح، يستبدل `Body` بكتلة `[Audio]` ويضبط `{{Transcript}}`.
- **تحليل الأوامر**: عندما ينجح النسخ، يتم تعيين `CommandBody`/`RawBody` إلى النص المنسوخ حتى تستمر أوامر الشرطة المائلة في العمل.
- **السجلات التفصيلية**: في `--verbose`، نسجل وقت تشغيل النسخ ووقت استبداله للنص.

## الاكتشاف التلقائي (الافتراضي)

إذا **لم تضبط نماذج** ولم يتم ضبط `tools.media.audio.enabled` على **`false`**،
فإن OpenClaw يكتشف تلقائيًا بهذا الترتيب ويتوقف عند أول خيار يعمل:

1. **نموذج الرد النشط** عندما يدعم مزوّده فهم الصوت.
2. **CLI المحلية** (إذا كانت مثبتة)
   - `sherpa-onnx-offline` (يتطلب `SHERPA_ONNX_MODEL_DIR` مع encoder/decoder/joiner/tokens)
   - `whisper-cli` (من `whisper-cpp`; يستخدم `WHISPER_CPP_MODEL` أو النموذج tiny المضمّن)
   - `whisper` (Python CLI؛ ينزّل النماذج تلقائيًا)
3. **Gemini CLI** (`gemini`) باستخدام `read_many_files`
4. **مصادقة المزوّد**
   - تُجرَّب أولًا الإدخالات المكوّنة في `models.providers.*` التي تدعم الصوت
   - ترتيب fallback المضمّن: OpenAI → Groq → Deepgram → Google → Mistral

لتعطيل الاكتشاف التلقائي، اضبط `tools.media.audio.enabled: false`.
وللتخصيص، اضبط `tools.media.audio.models`.
ملاحظة: اكتشاف الثنائيات يتم بأفضل جهد عبر macOS/Linux/Windows؛ تأكد من أن CLI موجودة على `PATH` (نحن نوسّع `~`)، أو اضبط نموذج CLI صريحًا مع مسار أمر كامل.

## أمثلة على التكوين

### مزوّد + رجوع احتياطي إلى CLI ‏(OpenAI + Whisper CLI)

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        maxBytes: 20971520,
        models: [
          { provider: "openai", model: "gpt-4o-mini-transcribe" },
          {
            type: "cli",
            command: "whisper",
            args: ["--model", "base", "{{MediaPath}}"],
            timeoutSeconds: 45,
          },
        ],
      },
    },
  },
}
```

### مزوّد فقط مع تقييد بالنطاق

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        scope: {
          default: "allow",
          rules: [{ action: "deny", match: { chatType: "group" } }],
        },
        models: [{ provider: "openai", model: "gpt-4o-mini-transcribe" }],
      },
    },
  },
}
```

### مزوّد فقط (Deepgram)

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        models: [{ provider: "deepgram", model: "nova-3" }],
      },
    },
  },
}
```

### مزوّد فقط (Mistral Voxtral)

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        models: [{ provider: "mistral", model: "voxtral-mini-latest" }],
      },
    },
  },
}
```

### إعادة إرسال النص المنسوخ إلى الدردشة (اشتراك اختياري)

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        echoTranscript: true, // default is false
        echoFormat: '📝 "{transcript}"', // optional, supports {transcript}
        models: [{ provider: "openai", model: "gpt-4o-mini-transcribe" }],
      },
    },
  },
}
```

## ملاحظات وحدود

- تتبع مصادقة المزوّد ترتيب مصادقة النماذج القياسي (ملفات تعريف المصادقة، ومتغيرات env، و`models.providers.*.apiKey`).
- تفاصيل إعداد Groq: [Groq](/providers/groq).
- يلتقط Deepgram القيمة `DEEPGRAM_API_KEY` عند استخدام `provider: "deepgram"`.
- تفاصيل إعداد Deepgram: [Deepgram (audio transcription)](/providers/deepgram).
- تفاصيل إعداد Mistral: [Mistral](/providers/mistral).
- يمكن لمزوّدي الصوت تجاوز `baseUrl` و`headers` و`providerOptions` عبر `tools.media.audio`.
- حد الحجم الافتراضي هو 20MB ‏(`tools.media.audio.maxBytes`). يتم تخطي الصوت الأكبر من اللازم لذلك النموذج وتجربة الإدخال التالي.
- يتم تخطي ملفات الصوت الصغيرة جدًا/الفارغة التي تقل عن 1024 بايت قبل نسخها بواسطة المزوّد/CLI.
- تكون القيمة الافتراضية `maxChars` للصوت **غير مضبوطة** (النص الكامل). اضبط `tools.media.audio.maxChars` أو `maxChars` لكل إدخال لتقليم الإخراج.
- القيمة الافتراضية التلقائية لـ OpenAI هي `gpt-4o-mini-transcribe`؛ اضبط `model: "gpt-4o-transcribe"` للحصول على دقة أعلى.
- استخدم `tools.media.audio.attachments` لمعالجة عدة رسائل صوتية (`mode: "all"` + `maxAttachments`).
- يكون النص المنسوخ متاحًا للقوالب باسم `{{Transcript}}`.
- تكون `tools.media.audio.echoTranscript` معطلة افتراضيًا؛ فعّلها لإرسال تأكيد النص المنسوخ مرة أخرى إلى الدردشة الأصلية قبل معالجة الوكيل.
- تخصص `tools.media.audio.echoFormat` نص echo (العنصر النائب: `{transcript}`).
- يتم وضع حد أقصى على stdout الخاصة بـ CLI ‏(5MB)؛ لذا اجعل إخراج CLI موجزًا.

### دعم بيئة proxy

تحترم عملية نسخ الصوت المعتمدة على المزوّد متغيرات بيئة proxy القياسية الخاصة بالخروج:

- `HTTPS_PROXY`
- `HTTP_PROXY`
- `https_proxy`
- `http_proxy`

إذا لم يتم تعيين أي متغيرات بيئة للـ proxy، فسيُستخدم الخروج المباشر. وإذا كان تكوين proxy غير صحيح، فسيسجل OpenClaw تحذيرًا ويعود إلى الجلب المباشر.

## اكتشاف الإشارات في المجموعات

عند تعيين `requireMention: true` لدردشة جماعية، يقوم OpenClaw الآن بنسخ الصوت **قبل** التحقق من الإشارات. وهذا يسمح بمعالجة الرسائل الصوتية حتى عندما تتضمن إشارات.

**كيف يعمل:**

1. إذا لم تكن للرسالة الصوتية body نصية وكانت المجموعة تتطلب إشارات، يجري OpenClaw نسخ "preflight".
2. يتم التحقق من النص المنسوخ مقابل أنماط الإشارة (مثل `@BotName` أو محفزات emoji).
3. إذا تم العثور على إشارة، تنتقل الرسالة عبر مسار الرد الكامل.
4. يُستخدم النص المنسوخ لاكتشاف الإشارات بحيث يمكن للرسائل الصوتية تجاوز بوابة الإشارة.

**سلوك الرجوع الاحتياطي:**

- إذا فشل النسخ أثناء preflight ‏(مهلة، خطأ API، إلخ)، تُعالج الرسالة بناءً على اكتشاف الإشارة النصية فقط.
- وهذا يضمن ألا يتم إسقاط الرسائل المختلطة (نص + صوت) بشكل غير صحيح أبدًا.

**إلغاء الاشتراك لكل مجموعة/موضوع في Telegram:**

- اضبط `channels.telegram.groups.<chatId>.disableAudioPreflight: true` لتخطي فحوصات الإشارة في النص المنسوخ في preflight لتلك المجموعة.
- اضبط `channels.telegram.groups.<chatId>.topics.<threadId>.disableAudioPreflight` للتجاوز لكل موضوع (`true` للتخطي، `false` للفرض).
- تكون القيمة الافتراضية `false` ‏(preflight مفعلة عندما تتطابق شروط تقييد الإشارة).

**مثال:** يرسل مستخدم رسالة صوتية تقول "Hey @Claude, what's the weather?" في مجموعة Telegram تحتوي على `requireMention: true`. يتم نسخ الرسالة الصوتية، واكتشاف الإشارة، ثم يرد الوكيل.

## مشكلات يجب الانتباه لها

- تستخدم قواعد النطاق مبدأ أول تطابق يفوز. ويتم تطبيع `chatType` إلى `direct` أو `group` أو `room`.
- تأكد من أن CLI تنهي التنفيذ بالرمز 0 وتطبع نصًا عاديًا؛ أما JSON فتحتاج إلى معالجتها عبر `jq -r .text`.
- بالنسبة إلى `parakeet-mlx`، إذا مررت `--output-dir`، يقرأ OpenClaw الملف `<output-dir>/<media-basename>.txt` عندما تكون `--output-format` هي `txt` (أو عند حذفها)؛ أما تنسيقات الإخراج غير `txt` فتعود إلى تحليل stdout.
- اجعل المهلات معقولة (`timeoutSeconds`، والافتراضي 60 ثانية) لتجنب حظر طابور الردود.
- يعالج نسخ preflight **أول** مرفق صوتي فقط لاكتشاف الإشارة. أما الصوتيات الإضافية فتُعالج أثناء مرحلة فهم الوسائط الرئيسية.
