---
read_when:
    - تريد استخدام Chutes مع OpenClaw
    - تحتاج إلى مسار إعداد OAuth أو مفتاح API
    - تريد معرفة النموذج الافتراضي، أو الأسماء المستعارة، أو سلوك الاكتشاف
summary: إعداد Chutes ‏(OAuth أو مفتاح API، واكتشاف النماذج، والأسماء المستعارة)
title: Chutes
x-i18n:
    generated_at: "2026-04-05T12:52:41Z"
    model: gpt-5.4
    provider: openai
    source_hash: e275f32e7a19fa5b4c64ffabfb4bf116dd5c9ab95bfa25bd3b1a15d15e237674
    source_path: providers/chutes.md
    workflow: 15
---

# Chutes

توفّر [Chutes](https://chutes.ai) فهارس النماذج مفتوحة المصدر عبر
واجهة API متوافقة مع OpenAI. ويدعم OpenClaw كلاً من OAuth عبر المتصفح
ومصادقة مفتاح API المباشرة للموفّر المدمج `chutes`.

- الموفّر: `chutes`
- API: متوافقة مع OpenAI
- Base URL: ‏`https://llm.chutes.ai/v1`
- المصادقة:
  - OAuth عبر `openclaw onboard --auth-choice chutes`
  - مفتاح API عبر `openclaw onboard --auth-choice chutes-api-key`
  - متغيرات env وقت التشغيل: `CHUTES_API_KEY`، `CHUTES_OAUTH_TOKEN`

## بدء سريع

### OAuth

```bash
openclaw onboard --auth-choice chutes
```

يشغّل OpenClaw تدفق المتصفح محليًا، أو يعرض عنوان URL + تدفق
لصق إعادة التوجيه على المضيفين البعيدين/من دون واجهة. ويتم تحديث رموز OAuth تلقائيًا
عبر ملفات تعريف المصادقة في OpenClaw.

تجاوزات OAuth الاختيارية:

- `CHUTES_CLIENT_ID`
- `CHUTES_CLIENT_SECRET`
- `CHUTES_OAUTH_REDIRECT_URI`
- `CHUTES_OAUTH_SCOPES`

### مفتاح API

```bash
openclaw onboard --auth-choice chutes-api-key
```

احصل على مفتاحك من
[chutes.ai/settings/api-keys](https://chutes.ai/settings/api-keys).

يسجل كلا مساري المصادقة فهرس Chutes المدمج ويضبطان النموذج الافتراضي
على `chutes/zai-org/GLM-4.7-TEE`.

## سلوك الاكتشاف

عندما تكون مصادقة Chutes متاحة، يستعلم OpenClaw فهرس Chutes باستخدام
بيانات الاعتماد تلك ويستخدم النماذج المكتشفة. وإذا فشل الاكتشاف، يعود OpenClaw
إلى فهرس ثابت مدمج حتى تستمر onboarding وبدء التشغيل في العمل.

## الأسماء المستعارة الافتراضية

يسجل OpenClaw أيضًا ثلاثة أسماء مستعارة مريحة لفهرس Chutes المدمج:

- `chutes-fast` ← `chutes/zai-org/GLM-4.7-FP8`
- `chutes-pro` ← `chutes/deepseek-ai/DeepSeek-V3.2-TEE`
- `chutes-vision` ← `chutes/chutesai/Mistral-Small-3.2-24B-Instruct-2506`

## فهرس البدء المدمج

يتضمن فهرس التراجع الاحتياطي المدمج مراجع Chutes الحالية مثل:

- `chutes/zai-org/GLM-4.7-TEE`
- `chutes/zai-org/GLM-5-TEE`
- `chutes/deepseek-ai/DeepSeek-V3.2-TEE`
- `chutes/deepseek-ai/DeepSeek-R1-0528-TEE`
- `chutes/moonshotai/Kimi-K2.5-TEE`
- `chutes/chutesai/Mistral-Small-3.2-24B-Instruct-2506`
- `chutes/Qwen/Qwen3-Coder-Next-TEE`
- `chutes/openai/gpt-oss-120b-TEE`

## مثال على الإعداد

```json5
{
  agents: {
    defaults: {
      model: { primary: "chutes/zai-org/GLM-4.7-TEE" },
      models: {
        "chutes/zai-org/GLM-4.7-TEE": { alias: "Chutes GLM 4.7" },
        "chutes/deepseek-ai/DeepSeek-V3.2-TEE": { alias: "Chutes DeepSeek V3.2" },
      },
    },
  },
}
```

## ملاحظات

- مساعدة OAuth ومتطلبات تطبيق إعادة التوجيه: [وثائق Chutes OAuth](https://chutes.ai/docs/sign-in-with-chutes/overview)
- يستخدم اكتشاف مفتاح API وOAuth كليهما معرّف الموفّر نفسه `chutes`.
- تُسجل نماذج Chutes بالشكل `chutes/<model-id>`.
