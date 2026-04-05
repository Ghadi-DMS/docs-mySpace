---
read_when:
    - أنت تريد استخدام GitHub Copilot كمزوّد نماذج
    - أنت تحتاج إلى تدفق `openclaw models auth login-github-copilot`
summary: سجّل الدخول إلى GitHub Copilot من OpenClaw باستخدام تدفق الجهاز
title: GitHub Copilot
x-i18n:
    generated_at: "2026-04-05T12:53:00Z"
    model: gpt-5.4
    provider: openai
    source_hash: 92857c119c314e698f922dbdbbc15d21b64d33a25979a2ec0ac1e82e586db6d6
    source_path: providers/github-copilot.md
    workflow: 15
---

# GitHub Copilot

## ما هو GitHub Copilot؟

GitHub Copilot هو مساعد GitHub البرمجي المعتمد على الذكاء الاصطناعي. ويوفر الوصول إلى
نماذج Copilot لحسابك وخطتك في GitHub. ويمكن لـ OpenClaw استخدام Copilot كمزوّد
نماذج بطريقتين مختلفتين.

## طريقتان لاستخدام Copilot في OpenClaw

### 1) مزوّد GitHub Copilot المدمج (`github-copilot`)

استخدم تدفق تسجيل الدخول الأصلي عبر الجهاز للحصول على رمز GitHub، ثم بدّله إلى
رموز Copilot API عندما يعمل OpenClaw. وهذا هو المسار **الافتراضي** والأبسط
لأنه لا يتطلب VS Code.

### 2) plugin ‏Copilot Proxy ‏(`copilot-proxy`)

استخدم امتداد **Copilot Proxy** في VS Code كجسر محلي. يتحدث OpenClaw إلى
نقطة النهاية `/v1` الخاصة بالـ proxy ويستخدم قائمة النماذج التي تهيئها هناك. اختر
هذا إذا كنت تشغّل Copilot Proxy بالفعل داخل VS Code أو تحتاج إلى التوجيه عبره.
يجب عليك تمكين plugin والإبقاء على امتداد VS Code قيد التشغيل.

استخدم GitHub Copilot كمزوّد نماذج (`github-copilot`). يشغّل أمر تسجيل الدخول
تدفق جهاز GitHub، ويحفظ profile مصادقة، ويحدّث إعداداتك لاستخدام ذلك
الملف التعريفي.

## إعداد CLI

```bash
openclaw models auth login-github-copilot
```

سيُطلب منك زيارة عنوان URL وإدخال رمز لمرة واحدة. أبقِ الطرفية
مفتوحة حتى يكتمل الإجراء.

### أعلام اختيارية

```bash
openclaw models auth login-github-copilot --yes
```

ولكي تطبق أيضًا النموذج الافتراضي الموصى به من المزوّد في خطوة واحدة، استخدم
أمر المصادقة العام بدلًا من ذلك:

```bash
openclaw models auth login --provider github-copilot --method device --set-default
```

## تعيين نموذج افتراضي

```bash
openclaw models set github-copilot/gpt-4o
```

### مقتطف إعدادات

```json5
{
  agents: { defaults: { model: { primary: "github-copilot/gpt-4o" } } },
}
```

## ملاحظات

- يتطلب TTY تفاعليًا؛ شغّله مباشرة في طرفية.
- يعتمد توفر نماذج Copilot على خطتك؛ وإذا تم رفض نموذج ما، فجرّب
  معرّفًا آخر (على سبيل المثال `github-copilot/gpt-4.1`).
- تستخدم معرّفات نماذج Claude نقل Anthropic Messages تلقائيًا؛ بينما تحتفظ نماذج GPT وo-series
  وGemini بنقل OpenAI Responses.
- يخزن تسجيل الدخول رمز GitHub في مخزن profile المصادقة ويبدّله إلى
  رمز Copilot API عندما يعمل OpenClaw.
