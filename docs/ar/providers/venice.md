---
read_when:
    - أنت تريد استدلالًا يركز على الخصوصية في OpenClaw
    - أنت تريد إرشادات إعداد Venice AI
summary: استخدم نماذج Venice AI المركزة على الخصوصية في OpenClaw
title: Venice AI
x-i18n:
    generated_at: "2026-04-05T12:54:40Z"
    model: gpt-5.4
    provider: openai
    source_hash: 53313e45e197880feb7e90764ee8fd6bb7f5fd4fe03af46b594201c77fbc8eab
    source_path: providers/venice.md
    workflow: 15
---

# Venice AI ‏(إبراز Venice)

**Venice** هو خيارنا المميز لإعداد Venice من أجل استدلال يركز على الخصوصية مع إمكانية وصول مجهول اختياري إلى النماذج الاحتكارية.

توفر Venice AI استدلالًا للذكاء الاصطناعي يركز على الخصوصية مع دعم للنماذج غير المقيّدة وإمكانية الوصول إلى النماذج الاحتكارية الكبرى عبر proxy مجهول خاص بها. وكل الاستدلال يكون خاصًا افتراضيًا — لا تدريب على بياناتك، ولا تسجيل للسجلات.

## لماذا Venice في OpenClaw

- **استدلال خاص** للنماذج مفتوحة المصدر (من دون تسجيل).
- **نماذج غير مقيّدة** عند الحاجة إليها.
- **وصول مجهول** إلى النماذج الاحتكارية (Opus/GPT/Gemini) عندما تكون الجودة مهمة.
- نقاط نهاية `/v1` متوافقة مع OpenAI.

## أوضاع الخصوصية

تقدم Venice مستويين من الخصوصية — وفهم ذلك أساسي لاختيار النموذج المناسب:

| الوضع           | الوصف                                                                                                                         | النماذج                                                        |
| -------------- | ----------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------- |
| **خاص**        | خاص بالكامل. لا يتم **أبدًا** تخزين prompts/responses أو تسجيلها. مؤقتة.                                                     | Llama، وQwen، وDeepSeek، وKimi، وMiniMax، وVenice Uncensored، وغيرها |
| **مجهول**      | يتم تمريره عبر Venice مع إزالة البيانات الوصفية. يرى المزوّد الأساسي (OpenAI أو Anthropic أو Google أو xAI) الطلبات المجهولة. | Claude، وGPT، وGemini، وGrok                                     |

## الميزات

- **يركز على الخصوصية**: اختر بين وضعي "خاص" (خاص بالكامل) و"مجهول" (عبر proxy)
- **نماذج غير مقيّدة**: الوصول إلى نماذج من دون قيود على المحتوى
- **الوصول إلى النماذج الكبرى**: استخدم Claude وGPT وGemini وGrok عبر proxy المجهول من Venice
- **API متوافقة مع OpenAI**: نقاط نهاية `/v1` قياسية للتكامل السهل
- **البث المتدفق**: ✅ مدعوم على جميع النماذج
- **استدعاء الدوال**: ✅ مدعوم على نماذج محددة (تحقق من قدرات النموذج)
- **الرؤية**: ✅ مدعومة على النماذج التي تملك قدرة الرؤية
- **لا توجد حدود معدل صارمة**: قد يتم تطبيق throttling للاستخدام العادل عند الاستخدام المفرط جدًا

## الإعداد

### 1. احصل على مفتاح API

1. سجّل في [venice.ai](https://venice.ai)
2. انتقل إلى **Settings → API Keys → Create new key**
3. انسخ مفتاح API الخاص بك (بالتنسيق: `vapi_xxxxxxxxxxxx`)

### 2. هيّئ OpenClaw

**الخيار A: متغير البيئة**

```bash
export VENICE_API_KEY="vapi_xxxxxxxxxxxx"
```

**الخيار B: الإعداد التفاعلي (موصى به)**

```bash
openclaw onboard --auth-choice venice-api-key
```

سيقوم هذا بما يلي:

1. طلب مفتاح API الخاص بك (أو استخدام `VENICE_API_KEY` الموجود)
2. عرض جميع نماذج Venice المتاحة
3. إتاحة اختيار النموذج الافتراضي
4. تهيئة المزوّد تلقائيًا

**الخيار C: غير تفاعلي**

```bash
openclaw onboard --non-interactive \
  --auth-choice venice-api-key \
  --venice-api-key "vapi_xxxxxxxxxxxx"
```

### 3. تحقّق من الإعداد

```bash
openclaw agent --model venice/kimi-k2-5 --message "Hello, are you working?"
```

## اختيار النموذج

بعد الإعداد، يعرض OpenClaw جميع نماذج Venice المتاحة. اختر بناءً على احتياجاتك:

- **النموذج الافتراضي**: `venice/kimi-k2-5` للاستدلال الخاص القوي بالإضافة إلى الرؤية.
- **خيار عالي القدرات**: `venice/claude-opus-4-6` لأقوى مسار Venice مجهول.
- **الخصوصية**: اختر النماذج "الخاصة" للاستدلال الخاص بالكامل.
- **القدرات**: اختر النماذج "المجهولة" للوصول إلى Claude وGPT وGemini عبر proxy الخاصة بـ Venice.

غيّر نموذجك الافتراضي في أي وقت:

```bash
openclaw models set venice/kimi-k2-5
openclaw models set venice/claude-opus-4-6
```

اعرض جميع النماذج المتاحة:

```bash
openclaw models list | grep venice
```

## التهيئة عبر `openclaw configure`

1. شغّل `openclaw configure`
2. اختر **Model/auth**
3. اختر **Venice AI**

## أي نموذج يجب أن أستخدم؟

| حالة الاستخدام                | النموذج الموصى به                  | السبب                                         |
| ----------------------------- | ---------------------------------- | --------------------------------------------- |
| **الدردشة العامة (الافتراضي)** | `kimi-k2-5`                        | استدلال خاص قوي بالإضافة إلى الرؤية          |
| **أفضل جودة إجمالًا**         | `claude-opus-4-6`                  | أقوى خيار Venice مجهول                       |
| **الخصوصية + البرمجة**        | `qwen3-coder-480b-a35b-instruct`   | نموذج برمجة خاص بسياق كبير                   |
| **رؤية خاصة**                 | `kimi-k2-5`                        | دعم الرؤية من دون مغادرة الوضع الخاص         |
| **سريع + منخفض التكلفة**      | `qwen3-4b`                         | نموذج استدلال خفيف                           |
| **مهام خاصة معقدة**           | `deepseek-v3.2`                    | استدلال قوي، لكن من دون دعم أدوات Venice     |
| **غير مقيّد**                 | `venice-uncensored`                | بلا قيود على المحتوى                         |

## النماذج المتاحة (41 إجمالًا)

### النماذج الخاصة (26) - خاصة بالكامل، من دون تسجيل

| معرّف النموذج                           | الاسم                                 | السياق | الميزات                    |
| -------------------------------------- | ------------------------------------- | ------ | -------------------------- |
| `kimi-k2-5`                            | Kimi K2.5                             | 256k   | افتراضي، استدلال، رؤية     |
| `kimi-k2-thinking`                     | Kimi K2 Thinking                      | 256k   | استدلال                    |
| `llama-3.3-70b`                        | Llama 3.3 70B                         | 128k   | عام                        |
| `llama-3.2-3b`                         | Llama 3.2 3B                          | 128k   | عام                        |
| `hermes-3-llama-3.1-405b`              | Hermes 3 Llama 3.1 405B               | 128k   | عام، الأدوات معطلة         |
| `qwen3-235b-a22b-thinking-2507`        | Qwen3 235B Thinking                   | 128k   | استدلال                    |
| `qwen3-235b-a22b-instruct-2507`        | Qwen3 235B Instruct                   | 128k   | عام                        |
| `qwen3-coder-480b-a35b-instruct`       | Qwen3 Coder 480B                      | 256k   | برمجة                      |
| `qwen3-coder-480b-a35b-instruct-turbo` | Qwen3 Coder 480B Turbo                | 256k   | برمجة                      |
| `qwen3-5-35b-a3b`                      | Qwen3.5 35B A3B                       | 256k   | استدلال، رؤية              |
| `qwen3-next-80b`                       | Qwen3 Next 80B                        | 256k   | عام                        |
| `qwen3-vl-235b-a22b`                   | Qwen3 VL 235B ‏(رؤية)                 | 256k   | رؤية                       |
| `qwen3-4b`                             | Venice Small ‏(Qwen3 4B)              | 32k    | سريع، استدلال              |
| `deepseek-v3.2`                        | DeepSeek V3.2                         | 160k   | استدلال، الأدوات معطلة     |
| `venice-uncensored`                    | Venice Uncensored ‏(Dolphin-Mistral)  | 32k    | غير مقيّد، الأدوات معطلة   |
| `mistral-31-24b`                       | Venice Medium ‏(Mistral)              | 128k   | رؤية                       |
| `google-gemma-3-27b-it`                | Google Gemma 3 27B Instruct           | 198k   | رؤية                       |
| `openai-gpt-oss-120b`                  | OpenAI GPT OSS 120B                   | 128k   | عام                        |
| `nvidia-nemotron-3-nano-30b-a3b`       | NVIDIA Nemotron 3 Nano 30B            | 128k   | عام                        |
| `olafangensan-glm-4.7-flash-heretic`   | GLM 4.7 Flash Heretic                 | 128k   | استدلال                    |
| `zai-org-glm-4.6`                      | GLM 4.6                               | 198k   | عام                        |
| `zai-org-glm-4.7`                      | GLM 4.7                               | 198k   | استدلال                    |
| `zai-org-glm-4.7-flash`                | GLM 4.7 Flash                         | 128k   | استدلال                    |
| `zai-org-glm-5`                        | GLM 5                                 | 198k   | استدلال                    |
| `minimax-m21`                          | MiniMax M2.1                          | 198k   | استدلال                    |
| `minimax-m25`                          | MiniMax M2.5                          | 198k   | استدلال                    |

### النماذج المجهولة (15) - عبر proxy الخاصة بـ Venice

| معرّف النموذج                    | الاسم                               | السياق | الميزات                    |
| ------------------------------- | ----------------------------------- | ------ | -------------------------- |
| `claude-opus-4-6`               | Claude Opus 4.6 ‏(عبر Venice)       | 1M     | استدلال، رؤية              |
| `claude-opus-4-5`               | Claude Opus 4.5 ‏(عبر Venice)       | 198k   | استدلال، رؤية              |
| `claude-sonnet-4-6`             | Claude Sonnet 4.6 ‏(عبر Venice)     | 1M     | استدلال، رؤية              |
| `claude-sonnet-4-5`             | Claude Sonnet 4.5 ‏(عبر Venice)     | 198k   | استدلال، رؤية              |
| `openai-gpt-54`                 | GPT-5.4 ‏(عبر Venice)               | 1M     | استدلال، رؤية              |
| `openai-gpt-53-codex`           | GPT-5.3 Codex ‏(عبر Venice)         | 400k   | استدلال، رؤية، برمجة       |
| `openai-gpt-52`                 | GPT-5.2 ‏(عبر Venice)               | 256k   | استدلال                    |
| `openai-gpt-52-codex`           | GPT-5.2 Codex ‏(عبر Venice)         | 256k   | استدلال، رؤية، برمجة       |
| `openai-gpt-4o-2024-11-20`      | GPT-4o ‏(عبر Venice)                | 128k   | رؤية                       |
| `openai-gpt-4o-mini-2024-07-18` | GPT-4o Mini ‏(عبر Venice)           | 128k   | رؤية                       |
| `gemini-3-1-pro-preview`        | Gemini 3.1 Pro ‏(عبر Venice)        | 1M     | استدلال، رؤية              |
| `gemini-3-pro-preview`          | Gemini 3 Pro ‏(عبر Venice)          | 198k   | استدلال، رؤية              |
| `gemini-3-flash-preview`        | Gemini 3 Flash ‏(عبر Venice)        | 256k   | استدلال، رؤية              |
| `grok-41-fast`                  | Grok 4.1 Fast ‏(عبر Venice)         | 1M     | استدلال، رؤية              |
| `grok-code-fast-1`              | Grok Code Fast 1 ‏(عبر Venice)      | 256k   | استدلال، برمجة             |

## اكتشاف النماذج

يكتشف OpenClaw النماذج تلقائيًا من Venice API عند ضبط `VENICE_API_KEY`. وإذا تعذر الوصول إلى API، فإنه يعود إلى فهرس ثابت.

تكون نقطة النهاية `/models` عامة (لا تحتاج إلى مصادقة من أجل العرض)، لكن الاستدلال يتطلب مفتاح API صالحًا.

## دعم البث والأدوات

| الميزة               | الدعم                                                   |
| -------------------- | ------------------------------------------------------- |
| **البث المتدفق**      | ✅ جميع النماذج                                          |
| **استدعاء الدوال**    | ✅ معظم النماذج (تحقق من `supportsFunctionCalling` في API) |
| **الرؤية/الصور**      | ✅ النماذج الموسومة بميزة "Vision"                      |
| **وضع JSON**         | ✅ مدعوم عبر `response_format`                           |

## التسعير

تستخدم Venice نظامًا قائمًا على الأرصدة. راجع [venice.ai/pricing](https://venice.ai/pricing) لمعرفة الأسعار الحالية:

- **النماذج الخاصة**: أقل تكلفة بشكل عام
- **النماذج المجهولة**: مشابهة للتسعير المباشر لـ API + رسوم صغيرة لـ Venice

## مقارنة: Venice مقابل API المباشر

| الجانب        | Venice ‏(مجهول)                  | API مباشر            |
| ------------- | -------------------------------- | -------------------- |
| **الخصوصية**  | إزالة البيانات الوصفية، مجهول     | حسابك مرتبط          |
| **زمن الاستجابة** | +10-50ms ‏(proxy)                | مباشر                |
| **الميزات**   | معظم الميزات مدعومة              | الميزات الكاملة      |
| **الفوترة**   | أرصدة Venice                     | فوترة المزوّد        |

## أمثلة الاستخدام

```bash
# Use the default private model
openclaw agent --model venice/kimi-k2-5 --message "Quick health check"

# Use Claude Opus via Venice (anonymized)
openclaw agent --model venice/claude-opus-4-6 --message "Summarize this task"

# Use uncensored model
openclaw agent --model venice/venice-uncensored --message "Draft options"

# Use vision model with image
openclaw agent --model venice/qwen3-vl-235b-a22b --message "Review attached image"

# Use coding model
openclaw agent --model venice/qwen3-coder-480b-a35b-instruct --message "Refactor this function"
```

## استكشاف الأخطاء وإصلاحها

### لم يتم التعرف على مفتاح API

```bash
echo $VENICE_API_KEY
openclaw models list | grep venice
```

تأكد من أن المفتاح يبدأ بـ `vapi_`.

### النموذج غير متاح

يتم تحديث فهرس نماذج Venice ديناميكيًا. شغّل `openclaw models list` لرؤية النماذج المتاحة حاليًا. وقد تكون بعض النماذج غير متصلة مؤقتًا.

### مشكلات الاتصال

توجد Venice API عند `https://api.venice.ai/api/v1`. تأكد من أن شبكتك تسمح باتصالات HTTPS.

## مثال على ملف الإعدادات

```json5
{
  env: { VENICE_API_KEY: "vapi_..." },
  agents: { defaults: { model: { primary: "venice/kimi-k2-5" } } },
  models: {
    mode: "merge",
    providers: {
      venice: {
        baseUrl: "https://api.venice.ai/api/v1",
        apiKey: "${VENICE_API_KEY}",
        api: "openai-completions",
        models: [
          {
            id: "kimi-k2-5",
            name: "Kimi K2.5",
            reasoning: true,
            input: ["text", "image"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 256000,
            maxTokens: 65536,
          },
        ],
      },
    },
  },
}
```

## الروابط

- [Venice AI](https://venice.ai)
- [توثيق API](https://docs.venice.ai)
- [الأسعار](https://venice.ai/pricing)
- [الحالة](https://status.venice.ai)
