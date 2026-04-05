---
read_when:
    - إنشاء أو مراجعة خطط `openclaw secrets apply`
    - تصحيح أخطاء `Invalid plan target path`
    - فهم سلوك التحقق من نوع الهدف والمسار
summary: 'العقد الخاص بخطط `secrets apply`: التحقق من الهدف، ومطابقة المسار، ونطاق هدف `auth-profiles.json`'
title: عقد خطة Secrets Apply
x-i18n:
    generated_at: "2026-04-05T12:44:03Z"
    model: gpt-5.4
    provider: openai
    source_hash: cb89a426ca937cf4d745f641b43b330c7fbb1aa9e4359b106ecd28d7a65ca327
    source_path: gateway/secrets-plan-contract.md
    workflow: 15
---

# عقد خطة secrets apply

تحدد هذه الصفحة العقد الصارم الذي يفرضه `openclaw secrets apply`.

إذا لم يطابق هدف ما هذه القواعد، يفشل التطبيق قبل تعديل الإعداد.

## شكل ملف الخطة

يتوقع `openclaw secrets apply --from <plan.json>` مصفوفة `targets` من أهداف الخطة:

```json5
{
  version: 1,
  protocolVersion: 1,
  targets: [
    {
      type: "models.providers.apiKey",
      path: "models.providers.openai.apiKey",
      pathSegments: ["models", "providers", "openai", "apiKey"],
      providerId: "openai",
      ref: { source: "env", provider: "default", id: "OPENAI_API_KEY" },
    },
    {
      type: "auth-profiles.api_key.key",
      path: "profiles.openai:default.key",
      pathSegments: ["profiles", "openai:default", "key"],
      agentId: "main",
      ref: { source: "env", provider: "default", id: "OPENAI_API_KEY" },
    },
  ],
}
```

## نطاق الأهداف المدعوم

تُقبل أهداف الخطة لمسارات بيانات الاعتماد المدعومة في:

- [سطح بيانات اعتماد SecretRef](/reference/secretref-credential-surface)

## سلوك نوع الهدف

القاعدة العامة:

- يجب أن يكون `target.type` معروفًا ويجب أن يطابق شكل `target.path` المُطبَّع.

لا تزال الأسماء المستعارة للتوافق مقبولة للخطط الموجودة:

- `models.providers.apiKey`
- `skills.entries.apiKey`
- `channels.googlechat.serviceAccount`

## قواعد التحقق من المسار

يتم التحقق من كل هدف باستخدام جميع ما يلي:

- يجب أن يكون `type` نوع هدف معروفًا.
- يجب أن يكون `path` مسار dot غير فارغ.
- يمكن حذف `pathSegments`. وإذا تم توفيره، فيجب أن يُطبَّع إلى المسار نفسه تمامًا مثل `path`.
- يتم رفض المقاطع المحظورة: `__proto__`, `prototype`, `constructor`.
- يجب أن يطابق المسار المُطبَّع شكل المسار المسجّل لنوع الهدف.
- إذا تم ضبط `providerId` أو `accountId`، فيجب أن يطابق المعرّف المشفّر في المسار.
- تتطلب أهداف `auth-profiles.json` وجود `agentId`.
- عند إنشاء ربط جديد في `auth-profiles.json`، قم بتضمين `authProfileProvider`.

## سلوك الإخفاق

إذا فشل التحقق من هدف، ينتهي التطبيق بخطأ مثل:

```text
Invalid plan target path for models.providers.apiKey: models.providers.openai.baseUrl
```

لا يتم تنفيذ أي كتابة لخطة غير صالحة.

## سلوك الموافقة على موفّر exec

- يتجاوز `--dry-run` فحوصات exec SecretRef افتراضيًا.
- يتم رفض الخطط التي تحتوي على exec SecretRefs/providers في وضع الكتابة ما لم يتم ضبط `--allow-exec`.
- عند التحقق من الخطط التي تحتوي على exec أو تطبيقها، مرّر `--allow-exec` في أوامر dry-run والكتابة معًا.

## ملاحظات وقت التشغيل ونطاق التدقيق

- يتم تضمين إدخالات `auth-profiles.json` المعتمدة على المراجع فقط (`keyRef`/`tokenRef`) في تحليل وقت التشغيل وتغطية التدقيق.
- يكتب `secrets apply` أهداف `openclaw.json` المدعومة، وأهداف `auth-profiles.json` المدعومة، وأهداف التنظيف الاختيارية.

## فحوصات المشغّل

```bash
# التحقق من الخطة من دون كتابة
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run

# ثم التطبيق الحقيقي
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json

# بالنسبة إلى الخطط التي تحتوي على exec، اشترك صراحةً في كلا الوضعين
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run --allow-exec
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --allow-exec
```

إذا فشل التطبيق برسالة مسار هدف غير صالح، فأعد إنشاء الخطة باستخدام `openclaw secrets configure` أو أصلح مسار الهدف إلى شكل مدعوم مما سبق.

## مستندات ذات صلة

- [إدارة الأسرار](/gateway/secrets)
- [CLI `secrets`](/cli/secrets)
- [سطح بيانات اعتماد SecretRef](/reference/secretref-credential-surface)
- [مرجع الإعداد](/gateway/configuration-reference)
