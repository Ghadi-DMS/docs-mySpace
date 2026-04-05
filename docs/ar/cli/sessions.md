---
read_when:
    - تريد عرض الجلسات المخزنة ورؤية النشاط الأخير
summary: مرجع CLI للأمر `openclaw sessions` (عرض الجلسات المخزنة + الاستخدام)
title: sessions
x-i18n:
    generated_at: "2026-04-05T12:39:25Z"
    model: gpt-5.4
    provider: openai
    source_hash: 47eb55d90bd0681676283310cfa50dcacc95dff7d9a39bf2bb188788c6e5e5ba
    source_path: cli/sessions.md
    workflow: 15
---

# `openclaw sessions`

اعرض جلسات المحادثة المخزنة.

```bash
openclaw sessions
openclaw sessions --agent work
openclaw sessions --all-agents
openclaw sessions --active 120
openclaw sessions --verbose
openclaw sessions --json
```

اختيار النطاق:

- الافتراضي: مخزن الوكيل الافتراضي المكوَّن
- `--verbose`: تسجيل verbose
- `--agent <id>`: مخزن وكيل مكوَّن واحد
- `--all-agents`: تجميع جميع مخازن الوكلاء المكوَّنة
- `--store <path>`: مسار مخزن صريح (لا يمكن دمجه مع `--agent` أو `--all-agents`)

يقرأ `openclaw sessions --all-agents` مخازن الوكلاء المكوَّنة. ويكون اكتشاف
جلسات Gateway وACP أوسع: إذ يشمل أيضًا المخازن الموجودة على القرص فقط التي
يتم العثور عليها ضمن الجذر الافتراضي `agents/` أو جذر `session.store`
المعتمد على قالب. ويجب أن تُحل هذه المخازن المكتشفة إلى ملفات `sessions.json`
عادية داخل جذر الوكيل؛ ويتم تخطي الروابط الرمزية والمسارات الخارجة عن الجذر.

أمثلة JSON:

`openclaw sessions --all-agents --json`:

```json
{
  "path": null,
  "stores": [
    { "agentId": "main", "path": "/home/user/.openclaw/agents/main/sessions/sessions.json" },
    { "agentId": "work", "path": "/home/user/.openclaw/agents/work/sessions/sessions.json" }
  ],
  "allAgents": true,
  "count": 2,
  "activeMinutes": null,
  "sessions": [
    { "agentId": "main", "key": "agent:main:main", "model": "gpt-5" },
    { "agentId": "work", "key": "agent:work:main", "model": "claude-opus-4-6" }
  ]
}
```

## صيانة التنظيف

شغّل الصيانة الآن (بدلًا من انتظار دورة الكتابة التالية):

```bash
openclaw sessions cleanup --dry-run
openclaw sessions cleanup --agent work --dry-run
openclaw sessions cleanup --all-agents --dry-run
openclaw sessions cleanup --enforce
openclaw sessions cleanup --enforce --active-key "agent:main:telegram:direct:123"
openclaw sessions cleanup --json
```

يستخدم `openclaw sessions cleanup` إعدادات `session.maintenance` من التكوين:

- ملاحظة النطاق: يحافظ `openclaw sessions cleanup` على مخازن الجلسات/النصوص فقط. ولا يقوم بتقليم سجلات تشغيل cron (`cron/runs/<jobId>.jsonl`) التي تُدار بواسطة `cron.runLog.maxBytes` و`cron.runLog.keepLines` في [تكوين Cron](/automation/cron-jobs#configuration) والمشروحة في [صيانة Cron](/automation/cron-jobs#maintenance).

- `--dry-run`: معاينة عدد الإدخالات التي سيتم تقليمها/تقييدها من دون كتابة.
  - في وضع النص، يطبع dry-run جدول إجراءات لكل جلسة (`Action` و`Key` و`Age` و`Model` و`Flags`) حتى تتمكن من معرفة ما سيتم الاحتفاظ به مقابل ما سيتم حذفه.
- `--enforce`: تطبيق الصيانة حتى عندما يكون `session.maintenance.mode` هو `warn`.
- `--fix-missing`: إزالة الإدخالات التي تفتقد ملفات النص الخاصة بها، حتى لو لم تكن ستخرج عادةً بسبب العمر/العدد بعد.
- `--active-key <key>`: حماية مفتاح نشط محدد من الإخلاء بسبب ميزانية القرص.
- `--agent <id>`: تشغيل التنظيف لمخزن وكيل مكوَّن واحد.
- `--all-agents`: تشغيل التنظيف لجميع مخازن الوكلاء المكوَّنة.
- `--store <path>`: التشغيل مقابل ملف `sessions.json` محدد.
- `--json`: طباعة ملخص JSON. ومع `--all-agents`، يتضمن الإخراج ملخصًا واحدًا لكل مخزن.

`openclaw sessions cleanup --all-agents --dry-run --json`:

```json
{
  "allAgents": true,
  "mode": "warn",
  "dryRun": true,
  "stores": [
    {
      "agentId": "main",
      "storePath": "/home/user/.openclaw/agents/main/sessions/sessions.json",
      "beforeCount": 120,
      "afterCount": 80,
      "pruned": 40,
      "capped": 0
    },
    {
      "agentId": "work",
      "storePath": "/home/user/.openclaw/agents/work/sessions/sessions.json",
      "beforeCount": 18,
      "afterCount": 18,
      "pruned": 0,
      "capped": 0
    }
  ]
}
```

ذو صلة:

- تكوين الجلسة: [مرجع التكوين](/gateway/configuration-reference#session)
