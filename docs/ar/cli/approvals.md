---
read_when:
    - تريد تعديل موافقات exec من CLI
    - تحتاج إلى إدارة قوائم السماح على مضيف gateway أو مضيفات العقد
summary: مرجع CLI للأمر `openclaw approvals` (موافقات exec لمضيف gateway أو مضيفات العقد)
title: approvals
x-i18n:
    generated_at: "2026-04-05T12:37:40Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7b2532bfd3e6e6ce43c96a2807df2dd00cb7b4320b77a7dfd09bee0531da610e
    source_path: cli/approvals.md
    workflow: 15
---

# `openclaw approvals`

أدِر موافقات exec لـ **المضيف المحلي** أو **مضيف gateway** أو **مضيف عقدة**.
بشكل افتراضي، تستهدف الأوامر ملف الموافقات المحلي على القرص. استخدم `--gateway` لاستهداف gateway، أو `--node` لاستهداف عقدة معيّنة.

الاسم البديل: `openclaw exec-approvals`

ذو صلة:

- موافقات Exec: [Exec approvals](/tools/exec-approvals)
- العقد: [Nodes](/nodes)

## الأوامر الشائعة

```bash
openclaw approvals get
openclaw approvals get --node <id|name|ip>
openclaw approvals get --gateway
```

يعرض `openclaw approvals get` الآن سياسة exec الفعالة للأهداف المحلية وأهداف gateway وأهداف العقد:

- سياسة `tools.exec` المطلوبة
- سياسة ملف الموافقات الخاصة بالمضيف
- النتيجة الفعالة بعد تطبيق قواعد الأسبقية

الأسبقية مقصودة:

- ملف موافقات المضيف هو مصدر الحقيقة القابل للفرض
- يمكن لسياسة `tools.exec` المطلوبة أن تضيّق المقصود أو توسّعه، لكن النتيجة الفعالة ما تزال مشتقة من قواعد المضيف
- يجمع `--node` بين ملف موافقات مضيف العقدة وسياسة `tools.exec` الخاصة بـ gateway، لأن كليهما ما يزال مطبقًا وقت التشغيل
- إذا لم يكن إعداد gateway متاحًا، يعود CLI إلى لقطة موافقات العقدة ويشير إلى أنه تعذّر حساب سياسة وقت التشغيل النهائية

## استبدال الموافقات من ملف

```bash
openclaw approvals set --file ./exec-approvals.json
openclaw approvals set --stdin <<'EOF'
{ version: 1, defaults: { security: "full", ask: "off" } }
EOF
openclaw approvals set --node <id|name|ip> --file ./exec-approvals.json
openclaw approvals set --gateway --file ./exec-approvals.json
```

يقبل `set` تنسيق JSON5، وليس JSON الصارم فقط. استخدم إما `--file` أو `--stdin`، وليس كليهما.

## مثال "عدم السؤال أبدًا" / YOLO

بالنسبة إلى مضيف يجب ألا يتوقف أبدًا عند موافقات exec، اضبط القيم الافتراضية لموافقات المضيف إلى `full` + `off`:

```bash
openclaw approvals set --stdin <<'EOF'
{
  version: 1,
  defaults: {
    security: "full",
    ask: "off",
    askFallback: "full"
  }
}
EOF
```

نسخة العقدة:

```bash
openclaw approvals set --node <id|name|ip> --stdin <<'EOF'
{
  version: 1,
  defaults: {
    security: "full",
    ask: "off",
    askFallback: "full"
  }
}
EOF
```

يغيّر هذا **ملف موافقات المضيف** فقط. ولإبقاء سياسة OpenClaw المطلوبة متوافقة، اضبط أيضًا:

```bash
openclaw config set tools.exec.host gateway
openclaw config set tools.exec.security full
openclaw config set tools.exec.ask off
```

سبب استخدام `tools.exec.host=gateway` في هذا المثال:

- ما يزال `host=auto` يعني "sandbox عند توفره، وإلا فـ gateway".
- يتعلق YOLO بالموافقات، وليس بالتوجيه.
- إذا كنت تريد exec على المضيف حتى عند تكوين sandbox، فاجعل اختيار المضيف صريحًا باستخدام `gateway` أو `/exec host=gateway`.

وهذا يطابق سلوك YOLO الافتراضي الحالي للمضيف. شدّده إذا كنت تريد الموافقات.

## مساعدات قائمة السماح

```bash
openclaw approvals allowlist add "~/Projects/**/bin/rg"
openclaw approvals allowlist add --agent main --node <id|name|ip> "/usr/bin/uptime"
openclaw approvals allowlist add --agent "*" "/usr/bin/uname"

openclaw approvals allowlist remove "~/Projects/**/bin/rg"
```

## الخيارات الشائعة

تدعم الأوامر `get` و`set` و`allowlist add|remove` جميعها ما يلي:

- `--node <id|name|ip>`
- `--gateway`
- خيارات RPC المشتركة للعقد: `--url`, `--token`, `--timeout`, `--json`

ملاحظات الاستهداف:

- عدم استخدام أي علامات استهداف يعني ملف الموافقات المحلي على القرص
- `--gateway` يستهدف ملف موافقات مضيف gateway
- `--node` يستهدف مضيف عقدة واحدًا بعد حل المعرّف أو الاسم أو IP أو بادئة المعرّف

كما يدعم `allowlist add|remove` ما يلي:

- `--agent <id>` (القيمة الافتراضية `*`)

## ملاحظات

- يستخدم `--node` محلل الأهداف نفسه الذي يستخدمه `openclaw nodes` (المعرّف أو الاسم أو ip أو بادئة المعرّف).
- القيمة الافتراضية لـ `--agent` هي `"*"`، وتُطبَّق على جميع الوكلاء.
- يجب أن يعلن مضيف العقدة عن `system.execApprovals.get/set` (تطبيق macOS أو مضيف عقدة headless).
- تُخزَّن ملفات الموافقات لكل مضيف في `~/.openclaw/exec-approvals.json`.
