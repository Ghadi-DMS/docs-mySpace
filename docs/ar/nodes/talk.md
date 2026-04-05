---
read_when:
    - تنفيذ وضع التحدث على macOS/iOS/Android
    - تغيير سلوك الصوت/TTS/المقاطعة
summary: 'وضع التحدث: محادثات صوتية مستمرة باستخدام ElevenLabs TTS'
title: وضع التحدث
x-i18n:
    generated_at: "2026-04-05T12:49:15Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3f10a3e9ee8fc2b4f7a89771d6e7b7373166a51ef9e9aa2d8c5ea67fc0729f9d
    source_path: nodes/talk.md
    workflow: 15
---

# وضع التحدث

وضع التحدث هو حلقة محادثة صوتية مستمرة:

1. الاستماع إلى الكلام
2. إرسال النص المفرغ إلى النموذج (الجلسة الرئيسية، `chat.send`)
3. انتظار الرد
4. نطقه عبر موفّر Talk المُعدّ (`talk.speak`)

## السلوك (macOS)

- **تراكب دائم الظهور** ما دام وضع التحدث مفعّلًا.
- انتقالات بين مراحل **الاستماع → التفكير → التحدث**.
- عند **توقف قصير** (نافذة صمت)، يتم إرسال النص المفرغ الحالي.
- تُكتب الردود في **WebChat** (مثلها مثل الكتابة).
- **المقاطعة عند الكلام** (مفعّلة افتراضيًا): إذا بدأ المستخدم بالكلام بينما كان المساعد يتحدث، فإننا نوقف التشغيل ونسجل الطابع الزمني للمقاطعة من أجل prompt التالي.

## توجيهات الصوت داخل الردود

يمكن للمساعد أن يسبق رده **بسطر JSON واحد** للتحكم في الصوت:

```json
{ "voice": "<voice-id>", "once": true }
```

القواعد:

- أول سطر غير فارغ فقط.
- يتم تجاهل المفاتيح غير المعروفة.
- تطبق `once: true` على الرد الحالي فقط.
- من دون `once`، يصبح الصوت هو الافتراضي الجديد لوضع التحدث.
- تتم إزالة سطر JSON قبل تشغيل TTS.

المفاتيح المدعومة:

- `voice` / `voice_id` / `voiceId`
- `model` / `model_id` / `modelId`
- `speed` و`rate` ‏(كلمات في الدقيقة)، و`stability`، و`similarity`، و`style`، و`speakerBoost`
- `seed` و`normalize` و`lang` و`output_format` و`latency_tier`
- `once`

## الإعدادات (`~/.openclaw/openclaw.json`)

```json5
{
  talk: {
    voiceId: "elevenlabs_voice_id",
    modelId: "eleven_v3",
    outputFormat: "mp3_44100_128",
    apiKey: "elevenlabs_api_key",
    silenceTimeoutMs: 1500,
    interruptOnSpeech: true,
  },
}
```

القيم الافتراضية:

- `interruptOnSpeech`: ‏true
- `silenceTimeoutMs`: عند عدم تعيينه، يحتفظ Talk بنافذة التوقف الافتراضية الخاصة بالمنصة قبل إرسال النص المفرغ (`700 ms` على macOS وAndroid، و`900 ms` على iOS)
- `voiceId`: يرجع إلى `ELEVENLABS_VOICE_ID` / `SAG_VOICE_ID` (أو أول صوت من ElevenLabs عندما يكون API key متاحًا)
- `modelId`: القيمة الافتراضية هي `eleven_v3` عند عدم التعيين
- `apiKey`: يرجع إلى `ELEVENLABS_API_KEY` (أو ملف shell profile الخاص بالبوابة إذا كان متاحًا)
- `outputFormat`: القيمة الافتراضية هي `pcm_44100` على macOS/iOS و`pcm_24000` على Android (اضبط `mp3_*` لفرض بث MP3)

## واجهة macOS

- مفتاح التبديل في شريط القوائم: **Talk**
- تبويب الإعدادات: مجموعة **Talk Mode** ‏(معرّف الصوت + مفتاح المقاطعة)
- التراكب:
  - **الاستماع**: تنبض السحابة مع مستوى الميكروفون
  - **التفكير**: حركة غوص
  - **التحدث**: حلقات متشععة
  - انقر السحابة: أوقف التحدث
  - انقر X: اخرج من وضع التحدث

## ملاحظات

- يتطلب أذونات Speech + Microphone.
- يستخدم `chat.send` على مفتاح الجلسة `main`.
- تحل البوابة تشغيل Talk عبر `talk.speak` باستخدام موفّر Talk النشط. ويرجع Android إلى TTS المحلي للنظام فقط عندما لا يكون RPC هذا متاحًا.
- يتم التحقق من `stability` في `eleven_v3` بحيث تكون `0.0` أو `0.5` أو `1.0`؛ أما النماذج الأخرى فتقبل `0..1`.
- يتم التحقق من `latency_tier` بحيث يكون `0..4` عند تعيينه.
- يدعم Android صيغ الإخراج `pcm_16000` و`pcm_22050` و`pcm_24000` و`pcm_44100` من أجل البث منخفض الكمون عبر AudioTrack.
