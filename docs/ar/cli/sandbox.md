---
read_when: You are managing sandbox runtimes or debugging sandbox/tool-policy behavior.
status: active
summary: إدارة بيئات تشغيل sandbox وفحص سياسة sandbox الفعالة
title: Sandbox CLI
x-i18n:
    generated_at: "2026-04-05T12:39:24Z"
    model: gpt-5.4
    provider: openai
    source_hash: fa2783037da2901316108d35e04bb319d5d57963c2764b9146786b3c6474b48a
    source_path: cli/sandbox.md
    workflow: 15
---

# Sandbox CLI

إدارة بيئات تشغيل sandbox لتنفيذ الوكلاء المعزول.

## نظرة عامة

يمكن لـ OpenClaw تشغيل الوكلاء داخل بيئات تشغيل sandbox معزولة لأسباب أمنية. تساعدك أوامر `sandbox` على فحص تلك البيئات وإعادة إنشائها بعد التحديثات أو تغييرات الإعدادات.

يعني ذلك اليوم عادةً:

- حاويات Docker sandbox
- بيئات تشغيل SSH sandbox عندما يكون `agents.defaults.sandbox.backend = "ssh"`
- بيئات تشغيل OpenShell sandbox عندما يكون `agents.defaults.sandbox.backend = "openshell"`

بالنسبة إلى `ssh` وOpenShell `remote`، تكون إعادة الإنشاء أكثر أهمية من Docker:

- تكون مساحة العمل البعيدة هي المرجع الأساسي بعد التهيئة الأولى
- يحذف `openclaw sandbox recreate` مساحة العمل البعيدة المرجعية تلك للنطاق المحدد
- وتقوم أول عملية استخدام لاحقة بتهيئتها مرة أخرى من مساحة العمل المحلية الحالية

## الأوامر

### `openclaw sandbox explain`

افحص وضع/nطاق/وصول مساحة العمل الفعّال في **sandbox**، وسياسة أداة sandbox، وبوابات الرفع، مع مسارات مفاتيح الإعدادات اللازمة للإصلاح.

```bash
openclaw sandbox explain
openclaw sandbox explain --session agent:main:main
openclaw sandbox explain --agent work
openclaw sandbox explain --json
```

### `openclaw sandbox list`

اعرض جميع بيئات تشغيل sandbox مع حالتها وإعداداتها.

```bash
openclaw sandbox list
openclaw sandbox list --browser  # List only browser containers
openclaw sandbox list --json     # JSON output
```

**يتضمن الخرج:**

- اسم بيئة التشغيل وحالتها
- الخلفية (`docker` أو `openshell` أو غيرها)
- تسمية الإعدادات وما إذا كانت تطابق الإعدادات الحالية
- العمر (الوقت منذ الإنشاء)
- وقت الخمول (الوقت منذ آخر استخدام)
- الجلسة/الوكيل المرتبط

### `openclaw sandbox recreate`

أزل بيئات تشغيل sandbox لفرض إعادة إنشائها باستخدام الإعدادات المحدّثة.

```bash
openclaw sandbox recreate --all                # Recreate all containers
openclaw sandbox recreate --session main       # Specific session
openclaw sandbox recreate --agent mybot        # Specific agent
openclaw sandbox recreate --browser            # Only browser containers
openclaw sandbox recreate --all --force        # Skip confirmation
```

**الخيارات:**

- `--all`: إعادة إنشاء جميع حاويات sandbox
- `--session <key>`: إعادة إنشاء الحاوية لجلسة محددة
- `--agent <id>`: إعادة إنشاء الحاويات لوكيل محدد
- `--browser`: إعادة إنشاء حاويات المتصفح فقط
- `--force`: تخطي مطالبة التأكيد

**مهم:** تتم إعادة إنشاء بيئات التشغيل تلقائيًا عند استخدام الوكيل في المرة التالية.

## حالات الاستخدام

### بعد تحديث صورة Docker

```bash
# Pull new image
docker pull openclaw-sandbox:latest
docker tag openclaw-sandbox:latest openclaw-sandbox:bookworm-slim

# Update config to use new image
# Edit config: agents.defaults.sandbox.docker.image (or agents.list[].sandbox.docker.image)

# Recreate containers
openclaw sandbox recreate --all
```

### بعد تغيير إعدادات sandbox

```bash
# Edit config: agents.defaults.sandbox.* (or agents.list[].sandbox.*)

# Recreate to apply new config
openclaw sandbox recreate --all
```

### بعد تغيير هدف SSH أو مواد مصادقة SSH

```bash
# Edit config:
# - agents.defaults.sandbox.backend
# - agents.defaults.sandbox.ssh.target
# - agents.defaults.sandbox.ssh.workspaceRoot
# - agents.defaults.sandbox.ssh.identityFile / certificateFile / knownHostsFile
# - agents.defaults.sandbox.ssh.identityData / certificateData / knownHostsData

openclaw sandbox recreate --all
```

بالنسبة إلى الخلفية الأساسية `ssh`، فإن إعادة الإنشاء تحذف جذر مساحة العمل البعيد لكل نطاق
على هدف SSH. وتقوم أول عملية تشغيل لاحقة بتهيئته مرة أخرى من مساحة العمل المحلية.

### بعد تغيير مصدر OpenShell أو السياسة أو الوضع

```bash
# Edit config:
# - agents.defaults.sandbox.backend
# - plugins.entries.openshell.config.from
# - plugins.entries.openshell.config.mode
# - plugins.entries.openshell.config.policy

openclaw sandbox recreate --all
```

بالنسبة إلى وضع OpenShell `remote`، فإن إعادة الإنشاء تحذف مساحة العمل البعيدة المرجعية
لذلك النطاق. وتقوم أول عملية تشغيل لاحقة بتهيئتها مرة أخرى من مساحة العمل المحلية.

### بعد تغيير setupCommand

```bash
openclaw sandbox recreate --all
# or just one agent:
openclaw sandbox recreate --agent family
```

### لوكيل محدد فقط

```bash
# Update only one agent's containers
openclaw sandbox recreate --agent alfred
```

## لماذا يلزم ذلك؟

**المشكلة:** عندما تحدّث إعدادات sandbox:

- تستمر بيئات التشغيل الموجودة في العمل بالإعدادات القديمة
- لا يتم تقليم بيئات التشغيل إلا بعد 24 ساعة من عدم النشاط
- تبقي الوكلاء الذين يُستخدمون بانتظام بيئات التشغيل القديمة حيّة إلى أجل غير مسمى

**الحل:** استخدم `openclaw sandbox recreate` لفرض إزالة بيئات التشغيل القديمة. وستُعاد إنشاؤها تلقائيًا بالإعدادات الحالية عند الحاجة إليها لاحقًا.

نصيحة: فضّل `openclaw sandbox recreate` على التنظيف اليدوي الخاص بكل خلفية.
فهو يستخدم سجل بيئات تشغيل Gateway ويتجنب حالات عدم التطابق عند تغيّر مفاتيح النطاق/الجلسة.

## الإعدادات

توجد إعدادات Sandbox في `~/.openclaw/openclaw.json` ضمن `agents.defaults.sandbox` (وتوضع التجاوزات لكل وكيل في `agents.list[].sandbox`):

```jsonc
{
  "agents": {
    "defaults": {
      "sandbox": {
        "mode": "all", // off, non-main, all
        "backend": "docker", // docker, ssh, openshell
        "scope": "agent", // session, agent, shared
        "docker": {
          "image": "openclaw-sandbox:bookworm-slim",
          "containerPrefix": "openclaw-sbx-",
          // ... more Docker options
        },
        "prune": {
          "idleHours": 24, // Auto-prune after 24h idle
          "maxAgeDays": 7, // Auto-prune after 7 days
        },
      },
    },
  },
}
```

## راجع أيضًا

- [توثيق Sandbox](/gateway/sandboxing)
- [إعداد الوكيل](/concepts/agent-workspace)
- [أمر Doctor](/gateway/doctor) - التحقق من إعداد sandbox
