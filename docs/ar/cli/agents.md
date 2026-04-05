---
read_when:
    - أنت تريد عدة وكلاء معزولين (مساحات عمل + توجيه + مصادقة)
summary: مرجع CLI للأمر `openclaw agents` ‏(list/add/delete/bindings/bind/unbind/set identity)
title: agents
x-i18n:
    generated_at: "2026-04-05T12:37:46Z"
    model: gpt-5.4
    provider: openai
    source_hash: 90b90c4915993bd8af322c0590d4cb59baabb8940598ce741315f8f95ef43179
    source_path: cli/agents.md
    workflow: 15
---

# `openclaw agents`

إدارة الوكلاء المعزولين (مساحات العمل + المصادقة + التوجيه).

ذو صلة:

- التوجيه متعدد الوكلاء: [التوجيه متعدد الوكلاء](/concepts/multi-agent)
- مساحة عمل الوكيل: [مساحة عمل الوكيل](/concepts/agent-workspace)
- إعدادات ظهور Skills: [إعدادات Skills](/tools/skills-config)

## أمثلة

```bash
openclaw agents list
openclaw agents list --bindings
openclaw agents add work --workspace ~/.openclaw/workspace-work
openclaw agents add ops --workspace ~/.openclaw/workspace-ops --bind telegram:ops --non-interactive
openclaw agents bindings
openclaw agents bind --agent work --bind telegram:ops
openclaw agents unbind --agent work --bind telegram:ops
openclaw agents set-identity --workspace ~/.openclaw/workspace --from-identity
openclaw agents set-identity --agent main --avatar avatars/openclaw.png
openclaw agents delete work
```

## ارتباطات التوجيه

استخدم ارتباطات التوجيه لتثبيت حركة القناة الواردة على وكيل محدد.

إذا كنت تريد أيضًا Skills مرئية مختلفة لكل وكيل، فقم بإعداد
`agents.defaults.skills` و`agents.list[].skills` في `openclaw.json`. راجع
[إعدادات Skills](/tools/skills-config) و
[مرجع الإعدادات](/gateway/configuration-reference#agentsdefaultsskills).

عرض الارتباطات:

```bash
openclaw agents bindings
openclaw agents bindings --agent work
openclaw agents bindings --json
```

إضافة ارتباطات:

```bash
openclaw agents bind --agent work --bind telegram:ops --bind discord:guild-a
```

إذا حذفت `accountId` ‏(`--bind <channel>`)، فإن OpenClaw يحلّه من القيم الافتراضية للقناة وخطافات إعداد plugin عند توفرها.

إذا حذفت `--agent` في `bind` أو `unbind`، فإن OpenClaw يستهدف الوكيل الافتراضي الحالي.

### سلوك نطاق الارتباط

- الارتباط من دون `accountId` يطابق الحساب الافتراضي للقناة فقط.
- `accountId: "*"` هو التراجع على مستوى القناة (كل الحسابات)، وهو أقل تحديدًا من ارتباط حساب صريح.
- إذا كان الوكيل نفسه يملك بالفعل ارتباط قناة مطابقًا من دون `accountId`، ثم قمت لاحقًا بالارتباط باستخدام `accountId` صريح أو محلول، فإن OpenClaw يرقّي ذلك الارتباط الموجود في مكانه بدلًا من إضافة ارتباط مكرر.

مثال:

```bash
# initial channel-only binding
openclaw agents bind --agent work --bind telegram

# later upgrade to account-scoped binding
openclaw agents bind --agent work --bind telegram:ops
```

بعد الترقية، يصبح التوجيه لذلك الارتباط ضمن النطاق `telegram:ops`. إذا كنت تريد أيضًا توجيه الحساب الافتراضي، فأضفه صراحةً (على سبيل المثال `--bind telegram:default`).

إزالة الارتباطات:

```bash
openclaw agents unbind --agent work --bind telegram:ops
openclaw agents unbind --agent work --all
```

يقبل `unbind` إما `--all` أو قيمة واحدة أو أكثر من `--bind`، وليس كليهما معًا.

## سطح الأوامر

### `agents`

تشغيل `openclaw agents` من دون أمر فرعي يعادل `openclaw agents list`.

### `agents list`

الخيارات:

- `--json`
- `--bindings`: تضمين قواعد التوجيه الكاملة، وليس فقط الأعداد/الملخصات لكل وكيل

### `agents add [name]`

الخيارات:

- `--workspace <dir>`
- `--model <id>`
- `--agent-dir <dir>`
- `--bind <channel[:accountId]>` (قابل للتكرار)
- `--non-interactive`
- `--json`

ملاحظات:

- تمرير أي أعلام إضافة صريحة يحوّل الأمر إلى المسار غير التفاعلي.
- يتطلب الوضع غير التفاعلي اسم وكيل و`--workspace` معًا.
- `main` محجوز ولا يمكن استخدامه كمعرّف للوكيل الجديد.

### `agents bindings`

الخيارات:

- `--agent <id>`
- `--json`

### `agents bind`

الخيارات:

- `--agent <id>` (الافتراضي هو الوكيل الافتراضي الحالي)
- `--bind <channel[:accountId]>` (قابل للتكرار)
- `--json`

### `agents unbind`

الخيارات:

- `--agent <id>` (الافتراضي هو الوكيل الافتراضي الحالي)
- `--bind <channel[:accountId]>` (قابل للتكرار)
- `--all`
- `--json`

### `agents delete <id>`

الخيارات:

- `--force`
- `--json`

ملاحظات:

- لا يمكن حذف `main`.
- من دون `--force`، يلزم تأكيد تفاعلي.
- يتم نقل مجلدات مساحة العمل وحالة الوكيل وسجل الجلسة إلى سلة المهملات، ولا تُحذف نهائيًا.

## ملفات الهوية

يمكن أن تتضمن مساحة عمل كل وكيل ملف `IDENTITY.md` في جذر مساحة العمل:

- مسار مثال: `~/.openclaw/workspace/IDENTITY.md`
- يقرأ `set-identity --from-identity` من جذر مساحة العمل (أو من `--identity-file` صريح)

يتم حل مسارات الصورة الرمزية نسبةً إلى جذر مساحة العمل.

## تعيين الهوية

يكتب `set-identity` الحقول داخل `agents.list[].identity`:

- `name`
- `theme`
- `emoji`
- `avatar` (مسار نسبةً إلى مساحة العمل، أو URL من نوع http(s)، أو data URI)

الخيارات:

- `--agent <id>`
- `--workspace <dir>`
- `--identity-file <path>`
- `--from-identity`
- `--name <name>`
- `--theme <theme>`
- `--emoji <emoji>`
- `--avatar <value>`
- `--json`

ملاحظات:

- يمكن استخدام `--agent` أو `--workspace` لتحديد الوكيل المستهدف.
- إذا كنت تعتمد على `--workspace` وكان عدة وكلاء يشاركون مساحة العمل نفسها، يفشل الأمر ويطلب منك تمرير `--agent`.
- عند عدم توفير أي حقول هوية صريحة، يقرأ الأمر بيانات الهوية من `IDENTITY.md`.

التحميل من `IDENTITY.md`:

```bash
openclaw agents set-identity --workspace ~/.openclaw/workspace --from-identity
```

تجاوز الحقول صراحةً:

```bash
openclaw agents set-identity --agent main --name "OpenClaw" --emoji "🦞" --avatar avatars/openclaw.png
```

نموذج إعدادات:

```json5
{
  agents: {
    list: [
      {
        id: "main",
        identity: {
          name: "OpenClaw",
          theme: "space lobster",
          emoji: "🦞",
          avatar: "avatars/openclaw.png",
        },
      },
    ],
  },
}
```
