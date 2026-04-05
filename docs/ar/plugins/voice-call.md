---
read_when:
    - أنت تريد إجراء مكالمة صوتية صادرة من OpenClaw
    - أنت تقوم بتهيئة plugin ‏voice-call أو تطويره
summary: 'plugin ‏Voice Call: مكالمات صادرة + واردة عبر Twilio/Telnyx/Plivo ‏(تثبيت plugin + الإعدادات + CLI)'
title: plugin ‏Voice Call
x-i18n:
    generated_at: "2026-04-05T12:52:49Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4e6d10c9fde6ce1f51637af285edc0c710e9cb7702231c0a91b527b721eaddc1
    source_path: plugins/voice-call.md
    workflow: 15
---

# Voice Call (plugin)

المكالمات الصوتية في OpenClaw عبر plugin. يدعم الإشعارات الصادرة
والمحادثات متعددة الأدوار مع سياسات للمكالمات الواردة.

المزوّدون الحاليون:

- `twilio` ‏(Programmable Voice + Media Streams)
- `telnyx` ‏(Call Control v2)
- `plivo` ‏(Voice API + XML transfer + GetInput speech)
- `mock` ‏(للتطوير/من دون شبكة)

النموذج الذهني السريع:

- ثبّت plugin
- أعد تشغيل Gateway
- هيّئ تحت `plugins.entries.voice-call.config`
- استخدم `openclaw voicecall ...` أو أداة `voice_call`

## مكان التشغيل (محلي أم بعيد)

يعمل plugin ‏Voice Call **داخل عملية Gateway**.

إذا كنت تستخدم Gateway بعيدًا، فقم بتثبيت/تهيئة plugin على **الجهاز الذي يشغّل Gateway**، ثم أعد تشغيل Gateway لتحميله.

## التثبيت

### الخيار A: التثبيت من npm ‏(موصى به)

```bash
openclaw plugins install @openclaw/voice-call
```

أعد تشغيل Gateway بعد ذلك.

### الخيار B: التثبيت من مجلد محلي (للتطوير، من دون نسخ)

```bash
PLUGIN_SRC=./path/to/local/voice-call-plugin
openclaw plugins install "$PLUGIN_SRC"
cd "$PLUGIN_SRC" && pnpm install
```

أعد تشغيل Gateway بعد ذلك.

## الإعدادات

اضبط الإعدادات تحت `plugins.entries.voice-call.config`:

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        enabled: true,
        config: {
          provider: "twilio", // or "telnyx" | "plivo" | "mock"
          fromNumber: "+15550001234",
          toNumber: "+15550005678",

          twilio: {
            accountSid: "ACxxxxxxxx",
            authToken: "...",
          },

          telnyx: {
            apiKey: "...",
            connectionId: "...",
            // Telnyx webhook public key from the Telnyx Mission Control Portal
            // (Base64 string; can also be set via TELNYX_PUBLIC_KEY).
            publicKey: "...",
          },

          plivo: {
            authId: "MAxxxxxxxxxxxxxxxxxxxx",
            authToken: "...",
          },

          // Webhook server
          serve: {
            port: 3334,
            path: "/voice/webhook",
          },

          // Webhook security (recommended for tunnels/proxies)
          webhookSecurity: {
            allowedHosts: ["voice.example.com"],
            trustedProxyIPs: ["100.64.0.1"],
          },

          // Public exposure (pick one)
          // publicUrl: "https://example.ngrok.app/voice/webhook",
          // tunnel: { provider: "ngrok" },
          // tailscale: { mode: "funnel", path: "/voice/webhook" }

          outbound: {
            defaultMode: "notify", // notify | conversation
          },

          streaming: {
            enabled: true,
            provider: "openai", // optional; first registered realtime transcription provider when unset
            streamPath: "/voice/stream",
            providers: {
              openai: {
                apiKey: "sk-...", // optional if OPENAI_API_KEY is set
                model: "gpt-4o-transcribe",
                silenceDurationMs: 800,
                vadThreshold: 0.5,
              },
            },
            preStartTimeoutMs: 5000,
            maxPendingConnections: 32,
            maxPendingConnectionsPerIp: 4,
            maxConnections: 128,
          },
        },
      },
    },
  },
}
```

ملاحظات:

- يتطلب Twilio/Telnyx عنوان URL لـ webhook **قابلًا للوصول علنًا**.
- يتطلب Plivo عنوان URL لـ webhook **قابلًا للوصول علنًا**.
- `mock` هو مزوّد تطوير محلي (من دون استدعاءات شبكة).
- إذا كانت الإعدادات الأقدم لا تزال تستخدم `provider: "log"` أو `twilio.from` أو مفاتيح OpenAI القديمة في `streaming.*`، فشغّل `openclaw doctor --fix` لإعادة كتابتها.
- يتطلب Telnyx القيمة `telnyx.publicKey` ‏(أو `TELNYX_PUBLIC_KEY`) ما لم تكن `skipSignatureVerification` تساوي true.
- إن `skipSignatureVerification` مخصص للاختبار المحلي فقط.
- إذا كنت تستخدم الطبقة المجانية من ngrok، فاضبط `publicUrl` على عنوان URL الدقيق الخاص بـ ngrok؛ إذ يتم فرض التحقق من التوقيع دائمًا.
- تسمح `tunnel.allowNgrokFreeTierLoopbackBypass: true` بـ webhooks من Twilio ذات تواقيع غير صالحة **فقط** عندما يكون `tunnel.provider="ngrok"` ويكون `serve.bind` هو loopback ‏(وكيل ngrok المحلي). استخدم ذلك للتطوير المحلي فقط.
- قد تتغير عناوين URL الخاصة بالطبقة المجانية من ngrok أو تضيف سلوك interstitial؛ وإذا تغيّر `publicUrl`، فستفشل تواقيع Twilio. وللإنتاج، فضّل نطاقًا ثابتًا أو Tailscale funnel.
- الإعدادات الافتراضية لأمان البث:
  - تقوم `streaming.preStartTimeoutMs` بإغلاق المقابس التي لا ترسل مطلقًا إطار `start` صالحًا.
- تحد `streaming.maxPendingConnections` من إجمالي مقابس ما قبل البدء غير الموثقة.
- تحد `streaming.maxPendingConnectionsPerIp` من مقابس ما قبل البدء غير الموثقة لكل عنوان IP مصدر.
- تحد `streaming.maxConnections` من إجمالي مقابس بث الوسائط المفتوحة (المعلقة + النشطة).
- لا يزال التراجع في وقت التشغيل يقبل مفاتيح voice-call القديمة هذه في الوقت الحالي، لكن مسار إعادة الكتابة هو `openclaw doctor --fix` وطبقة التوافق مؤقتة.

## النسخ الفوري

يحدد `streaming` مزوّد النسخ الفوري للصوت المباشر للمكالمات.

سلوك وقت التشغيل الحالي:

- `streaming.provider` اختياري. وإذا لم يتم ضبطه، يستخدم Voice Call أول
  مزوّد نسخ فوري مسجل.
- اليوم، المزوّد المضمّن هو OpenAI، والمسجل بواسطة plugin المضمّن `openai`.
- توجد الإعدادات الخام المملوكة للمزوّد تحت `streaming.providers.<providerId>`.
- إذا كانت `streaming.provider` تشير إلى مزوّد غير مسجل، أو لم يكن هناك أي مزوّد
  نسخ فوري مسجل أصلًا، يسجل Voice Call تحذيرًا
  ويتخطى بث الوسائط بدلًا من فشل plugin بالكامل.

الإعدادات الافتراضية للنسخ الفوري في OpenAI:

- مفتاح API: ‏`streaming.providers.openai.apiKey` أو `OPENAI_API_KEY`
- النموذج: `gpt-4o-transcribe`
- `silenceDurationMs`: ‏`800`
- `vadThreshold`: ‏`0.5`

مثال:

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          streaming: {
            enabled: true,
            provider: "openai",
            streamPath: "/voice/stream",
            providers: {
              openai: {
                apiKey: "sk-...", // optional if OPENAI_API_KEY is set
                model: "gpt-4o-transcribe",
                silenceDurationMs: 800,
                vadThreshold: 0.5,
              },
            },
          },
        },
      },
    },
  },
}
```

لا تزال المفاتيح القديمة تُرحّل تلقائيًا بواسطة `openclaw doctor --fix`:

- `streaming.sttProvider` → `streaming.provider`
- `streaming.openaiApiKey` → `streaming.providers.openai.apiKey`
- `streaming.sttModel` → `streaming.providers.openai.model`
- `streaming.silenceDurationMs` → `streaming.providers.openai.silenceDurationMs`
- `streaming.vadThreshold` → `streaming.providers.openai.vadThreshold`

## منظف المكالمات القديمة

استخدم `staleCallReaperSeconds` لإنهاء المكالمات التي لا تتلقى مطلقًا webhook نهائيًا
(على سبيل المثال مكالمات وضع notify التي لا تكتمل أبدًا). القيمة الافتراضية هي `0`
‏(معطل).

النطاقات الموصى بها:

- **الإنتاج:** من `120` إلى `300` ثانية للتدفقات على نمط notify.
- أبقِ هذه القيمة **أعلى من `maxDurationSeconds`** حتى تتمكن المكالمات
  العادية من الاكتمال. ونقطة بداية جيدة هي `maxDurationSeconds + 30–60` ثانية.

مثال:

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          maxDurationSeconds: 300,
          staleCallReaperSeconds: 360,
        },
      },
    },
  },
}
```

## أمان Webhook

عندما يكون proxy أو tunnel أمام Gateway، يعيد plugin بناء
عنوان URL العام لأغراض التحقق من التوقيع. وتتحكم هذه الخيارات في أي الترويسات
المعادة توجيهها يتم الوثوق بها.

تضع `webhookSecurity.allowedHosts` قائمة سماح بالمضيفين القادمين من ترويسات إعادة التوجيه.

تجعل `webhookSecurity.trustForwardingHeaders` الثقة بالترويسات المعاد توجيهها ممكنة من دون قائمة سماح.

تجعل `webhookSecurity.trustedProxyIPs` الثقة بالترويسات المعاد توجيهها ممكنة فقط عندما
يطابق عنوان IP البعيد للطلب القائمة.

تكون حماية إعادة تشغيل webhook مفعلة لـ Twilio وPlivo. وتتم الموافقة على
طلبات webhook المعاد تشغيلها والصحيحة ولكن يتم تخطي آثارها الجانبية.

تتضمن أدوار المحادثة في Twilio رمزًا مميزًا لكل دور في استدعاءات `<Gather>`، بحيث
لا يمكن لاستدعاءات الكلام القديمة/المعاد تشغيلها أن تلبّي دور نسخ أحدث ما يزال معلقًا.

تُرفض طلبات webhook غير الموثقة قبل قراءة الجسم عندما تكون
ترويسات التوقيع المطلوبة من المزوّد مفقودة.

يستخدم webhook الخاص بـ voice-call ملف تعريف الجسم قبل المصادقة المشترك (64 KB / 5 ثوانٍ)
بالإضافة إلى حد لكل IP للطلبات قيد التنفيذ قبل التحقق من التوقيع.

مثال مع مضيف عام ثابت:

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          publicUrl: "https://voice.example.com/voice/webhook",
          webhookSecurity: {
            allowedHosts: ["voice.example.com"],
          },
        },
      },
    },
  },
}
```

## TTS للمكالمات

يستخدم Voice Call إعدادات `messages.tts` الأساسية من أجل
بث الكلام في المكالمات. ويمكنك تجاوزها تحت إعدادات plugin باستخدام
**البنية نفسها** — حيث تُدمج دمجًا عميقًا مع `messages.tts`.

```json5
{
  tts: {
    provider: "elevenlabs",
    providers: {
      elevenlabs: {
        voiceId: "pMsXgVXv3BLzUgSXRplE",
        modelId: "eleven_multilingual_v2",
      },
    },
  },
}
```

ملاحظات:

- تُرحّل مفاتيح `tts.<provider>` القديمة داخل إعدادات plugin ‏(`openai` و`elevenlabs` و`microsoft` و`edge`) تلقائيًا إلى `tts.providers.<provider>` عند التحميل. فضّل بنية `providers` في الإعدادات الملتزم بها.
- **يتم تجاهل Microsoft speech في المكالمات الصوتية** ‏(فالصوت الهاتفي يحتاج إلى PCM؛ أما النقل الحالي لـ Microsoft فلا يكشف خرج PCM الهاتفي).
- يتم استخدام TTS الأساسي عند تمكين Twilio media streaming؛ وإلا تعود المكالمات إلى الأصوات الأصلية للمزوّد.
- إذا كان Twilio media stream نشطًا بالفعل، فلن يعود Voice Call إلى TwiML `<Say>`. وإذا لم يكن TTS الهاتفي متاحًا في تلك الحالة، يفشل طلب التشغيل بدلًا من مزج مساري تشغيل.
- عندما يعود TTS الهاتفي إلى مزوّد ثانوي، يسجل Voice Call تحذيرًا مع سلسلة المزوّد (`from` و`to` و`attempts`) لأغراض التصحيح.

### مزيد من الأمثلة

استخدم TTS الأساسي فقط (من دون تجاوز):

```json5
{
  messages: {
    tts: {
      provider: "openai",
      providers: {
        openai: { voice: "alloy" },
      },
    },
  },
}
```

التجاوز إلى ElevenLabs للمكالمات فقط (مع إبقاء الإعداد الأساسي كما هو في أماكن أخرى):

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          tts: {
            provider: "elevenlabs",
            providers: {
              elevenlabs: {
                apiKey: "elevenlabs_key",
                voiceId: "pMsXgVXv3BLzUgSXRplE",
                modelId: "eleven_multilingual_v2",
              },
            },
          },
        },
      },
    },
  },
}
```

تجاوز نموذج OpenAI فقط للمكالمات (مثال على الدمج العميق):

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          tts: {
            providers: {
              openai: {
                model: "gpt-4o-mini-tts",
                voice: "marin",
              },
            },
          },
        },
      },
    },
  },
}
```

## المكالمات الواردة

تكون القيمة الافتراضية لسياسة الوارد `disabled`. ولتمكين المكالمات الواردة، اضبط:

```json5
{
  inboundPolicy: "allowlist",
  allowFrom: ["+15550001234"],
  inboundGreeting: "Hello! How can I help?",
}
```

إن `inboundPolicy: "allowlist"` هو فحص منخفض الضمان لمعرّف المتصل. يقوم plugin
بتطبيع قيمة `From` الموردة من المزوّد ويقارنها مع `allowFrom`.
وتصادق عملية التحقق من webhook على تسليم المزوّد وسلامة الحمولة، لكنها
لا تثبت ملكية رقم المتصل في PSTN/VoIP. لذا تعامل مع `allowFrom` على
أنه تصفية لمعرّف المتصل، وليس هوية قوية للمتصل.

تستخدم الردود التلقائية نظام الوكيل. ويمكن ضبطها عبر:

- `responseModel`
- `responseSystemPrompt`
- `responseTimeoutMs`

### عقد الخرج المنطوق

بالنسبة إلى الردود التلقائية، يضيف Voice Call عقد خرج منطوق صارمًا إلى system prompt:

- `{"spoken":"..."}`

ثم يستخرج Voice Call نص الكلام بشكل دفاعي:

- يتجاهل الحمولات الموسومة على أنها reasoning/error.
- يحلل JSON المباشر، أو JSON داخل fenced، أو مفاتيح `"spoken"` المضمنة.
- يعود إلى النص العادي ويزيل فقرات المقدمة التخطيطية/الوصفية المرجحة.

وهذا يبقي التشغيل المنطوق مركزًا على النص الموجّه للمتصل ويمنع تسرب نص التخطيط إلى الصوت.

### سلوك بدء المحادثة

بالنسبة إلى مكالمات `conversation` الصادرة، يرتبط التعامل مع أول رسالة بحالة التشغيل المباشر:

- يتم كبت مسح طابور barge-in والرد التلقائي فقط أثناء نطق التحية الأولية فعليًا.
- إذا فشل التشغيل الأولي، تعود المكالمة إلى `listening` وتبقى الرسالة الأولية في الطابور لإعادة المحاولة.
- يبدأ التشغيل الأولي للبث في Twilio عند اتصال stream من دون تأخير إضافي.

### مهلة سماح فصل بث Twilio

عندما ينفصل Twilio media stream، ينتظر Voice Call مدة `2000ms` قبل الإنهاء التلقائي للمكالمة:

- إذا أعاد stream الاتصال خلال تلك النافذة، يتم إلغاء الإنهاء التلقائي.
- إذا لم تتم إعادة تسجيل أي stream بعد فترة السماح، يتم إنهاء المكالمة لمنع بقاء مكالمات نشطة عالقة.

## CLI

```bash
openclaw voicecall call --to "+15555550123" --message "Hello from OpenClaw"
openclaw voicecall start --to "+15555550123"   # alias for call
openclaw voicecall continue --call-id <id> --message "Any questions?"
openclaw voicecall speak --call-id <id> --message "One moment"
openclaw voicecall end --call-id <id>
openclaw voicecall status --call-id <id>
openclaw voicecall tail
openclaw voicecall latency                     # summarize turn latency from logs
openclaw voicecall expose --mode funnel
```

يقرأ `latency` الملف `calls.jsonl` من مسار تخزين voice-call الافتراضي. استخدم
`--file <path>` للإشارة إلى سجل مختلف و`--last <n>` لقصر التحليل
على آخر N سجلات (الافتراضي 200). يتضمن الخرج p50/p90/p99 لكل من
زمن الاستجابة للدور وأوقات انتظار الاستماع.

## أداة الوكيل

اسم الأداة: `voice_call`

الإجراءات:

- `initiate_call` ‏(`message`، و`to?`، و`mode?`)
- `continue_call` ‏(`callId`، و`message`)
- `speak_to_user` ‏(`callId`، و`message`)
- `end_call` ‏(`callId`)
- `get_status` ‏(`callId`)

يشحن هذا المستودع مستند Skill مطابقًا في `skills/voice-call/SKILL.md`.

## Gateway RPC

- `voicecall.initiate` ‏(`to?`، و`message`، و`mode?`)
- `voicecall.continue` ‏(`callId`، و`message`)
- `voicecall.speak` ‏(`callId`، و`message`)
- `voicecall.end` ‏(`callId`)
- `voicecall.status` ‏(`callId`)
