---
read_when:
    - تكوين مجموعات البث
    - تصحيح أخطاء الردود متعددة الوكلاء في WhatsApp
status: experimental
summary: بث رسالة WhatsApp إلى عدة وكلاء
title: مجموعات البث
x-i18n:
    generated_at: "2026-04-05T12:35:05Z"
    model: gpt-5.4
    provider: openai
    source_hash: 1d117ae65ec3b63c2bd4b3c215d96f32d7eafa0f99a9cd7378e502c15e56ca56
    source_path: channels/broadcast-groups.md
    workflow: 15
---

# مجموعات البث

**الحالة:** تجريبية  
**الإصدار:** أضيفت في 2026.1.9

## نظرة عامة

تتيح مجموعات البث لعدة وكلاء معالجة الرسالة نفسها والرد عليها في الوقت نفسه. يتيح لك ذلك إنشاء فرق وكلاء متخصصة تعمل معًا داخل مجموعة WhatsApp واحدة أو رسالة خاصة — وكل ذلك باستخدام رقم هاتف واحد.

النطاق الحالي: **WhatsApp فقط** (قناة الويب).

تُقيَّم مجموعات البث بعد قوائم السماح الخاصة بالقنوات وقواعد تفعيل المجموعات. في مجموعات WhatsApp، يعني هذا أن عمليات البث تحدث عندما يكون OpenClaw سيرد عادةً (على سبيل المثال: عند الإشارة، حسب إعدادات مجموعتك).

## حالات الاستخدام

### 1. فرق الوكلاء المتخصصة

انشر عدة وكلاء بمسؤوليات محددة ومركزة:

```
Group: "Development Team"
Agents:
  - CodeReviewer (reviews code snippets)
  - DocumentationBot (generates docs)
  - SecurityAuditor (checks for vulnerabilities)
  - TestGenerator (suggests test cases)
```

يعالج كل وكيل الرسالة نفسها ويقدم منظوره المتخصص.

### 2. دعم متعدد اللغات

```
Group: "International Support"
Agents:
  - Agent_EN (responds in English)
  - Agent_DE (responds in German)
  - Agent_ES (responds in Spanish)
```

### 3. سير عمل ضمان الجودة

```
Group: "Customer Support"
Agents:
  - SupportAgent (provides answer)
  - QAAgent (reviews quality, only responds if issues found)
```

### 4. أتمتة المهام

```
Group: "Project Management"
Agents:
  - TaskTracker (updates task database)
  - TimeLogger (logs time spent)
  - ReportGenerator (creates summaries)
```

## التكوين

### الإعداد الأساسي

أضف قسم `broadcast` على المستوى الأعلى (بجانب `bindings`). المفاتيح هي معرّفات نظير WhatsApp:

- محادثات المجموعات: group JID (مثل `120363403215116621@g.us`)
- الرسائل الخاصة: رقم هاتف بتنسيق E.164 (مثل `+15551234567`)

```json
{
  "broadcast": {
    "120363403215116621@g.us": ["alfred", "baerbel", "assistant3"]
  }
}
```

**النتيجة:** عندما يكون OpenClaw سيرد في هذه الدردشة، سيشغّل الوكلاء الثلاثة جميعهم.

### استراتيجية المعالجة

تحكم في كيفية معالجة الوكلاء للرسائل:

#### بالتوازي (الافتراضي)

يعالج جميع الوكلاء الرسائل في الوقت نفسه:

```json
{
  "broadcast": {
    "strategy": "parallel",
    "120363403215116621@g.us": ["alfred", "baerbel"]
  }
}
```

#### بالتسلسل

يعالج الوكلاء الرسائل بالترتيب (ينتظر أحدهم انتهاء السابق):

```json
{
  "broadcast": {
    "strategy": "sequential",
    "120363403215116621@g.us": ["alfred", "baerbel"]
  }
}
```

### مثال كامل

```json
{
  "agents": {
    "list": [
      {
        "id": "code-reviewer",
        "name": "Code Reviewer",
        "workspace": "/path/to/code-reviewer",
        "sandbox": { "mode": "all" }
      },
      {
        "id": "security-auditor",
        "name": "Security Auditor",
        "workspace": "/path/to/security-auditor",
        "sandbox": { "mode": "all" }
      },
      {
        "id": "docs-generator",
        "name": "Documentation Generator",
        "workspace": "/path/to/docs-generator",
        "sandbox": { "mode": "all" }
      }
    ]
  },
  "broadcast": {
    "strategy": "parallel",
    "120363403215116621@g.us": ["code-reviewer", "security-auditor", "docs-generator"],
    "120363424282127706@g.us": ["support-en", "support-de"],
    "+15555550123": ["assistant", "logger"]
  }
}
```

## كيف يعمل

### تدفق الرسائل

1. **تصل رسالة واردة** في مجموعة WhatsApp
2. **فحص البث**: يتحقق النظام مما إذا كان معرّف النظير موجودًا في `broadcast`
3. **إذا كان في قائمة البث**:
   - يعالج جميع الوكلاء المدرجين الرسالة
   - لكل وكيل مفتاح جلسة خاص به وسياق معزول
   - يعالج الوكلاء الرسائل بالتوازي (افتراضيًا) أو بالتسلسل
4. **إذا لم يكن في قائمة البث**:
   - يُطبَّق التوجيه العادي (أول binding مطابق)

ملاحظة: لا تتجاوز مجموعات البث قوائم السماح الخاصة بالقنوات أو قواعد تفعيل المجموعات (الإشارات/الأوامر/إلخ). فهي تغيّر فقط _أي الوكلاء يعملون_ عندما تكون الرسالة مؤهلة للمعالجة.

### عزل الجلسات

يحافظ كل وكيل في مجموعة بث على فصل كامل لما يلي:

- **مفاتيح الجلسات** (`agent:alfred:whatsapp:group:120363...` مقابل `agent:baerbel:whatsapp:group:120363...`)
- **سجل المحادثة** (الوكيل لا يرى رسائل الوكلاء الآخرين)
- **مساحة العمل** (بيئات معزولة منفصلة إذا تم تكوينها)
- **الوصول إلى الأدوات** (قوائم سماح/منع مختلفة)
- **الذاكرة/السياق** (`IDENTITY.md` و`SOUL.md` وما إلى ذلك بشكل منفصل)
- **مخزن سياق المجموعة** (رسائل المجموعة الحديثة المستخدمة للسياق) يكون مشتركًا لكل نظير، لذا يرى جميع وكلاء البث السياق نفسه عند التشغيل

وهذا يتيح لكل وكيل أن يمتلك:

- شخصيات مختلفة
- وصولًا مختلفًا إلى الأدوات (مثل للقراءة فقط مقابل القراءة والكتابة)
- نماذج مختلفة (مثل opus مقابل sonnet)
- Skills مختلفة مثبّتة

### مثال: جلسات معزولة

في المجموعة `120363403215116621@g.us` مع الوكلاء `["alfred", "baerbel"]`:

**سياق Alfred:**

```
Session: agent:alfred:whatsapp:group:120363403215116621@g.us
History: [user message, alfred's previous responses]
Workspace: /Users/user/openclaw-alfred/
Tools: read, write, exec
```

**سياق Bärbel:**

```
Session: agent:baerbel:whatsapp:group:120363403215116621@g.us
History: [user message, baerbel's previous responses]
Workspace: /Users/user/openclaw-baerbel/
Tools: read only
```

## أفضل الممارسات

### 1. أبقِ الوكلاء مركّزين

صمّم كل وكيل بحيث تكون له مسؤولية واحدة وواضحة:

```json
{
  "broadcast": {
    "DEV_GROUP": ["formatter", "linter", "tester"]
  }
}
```

✅ **جيد:** لكل وكيل مهمة واحدة  
❌ **سيئ:** وكيل عام واحد باسم "dev-helper"

### 2. استخدم أسماء وصفية

اجعل وظيفة كل وكيل واضحة:

```json
{
  "agents": {
    "security-scanner": { "name": "Security Scanner" },
    "code-formatter": { "name": "Code Formatter" },
    "test-generator": { "name": "Test Generator" }
  }
}
```

### 3. كوّن وصولًا مختلفًا إلى الأدوات

امنح الوكلاء الأدوات التي يحتاجونها فقط:

```json
{
  "agents": {
    "reviewer": {
      "tools": { "allow": ["read", "exec"] } // Read-only
    },
    "fixer": {
      "tools": { "allow": ["read", "write", "edit", "exec"] } // Read-write
    }
  }
}
```

### 4. راقب الأداء

مع وجود عدد كبير من الوكلاء، ضع في اعتبارك ما يلي:

- استخدام `"strategy": "parallel"` (الافتراضي) للسرعة
- حصر مجموعات البث في 5-10 وكلاء
- استخدام نماذج أسرع للوكلاء الأبسط

### 5. تعامل مع الإخفاقات بسلاسة

يفشل الوكلاء بشكل مستقل. لا يمنع خطأ أحد الوكلاء الآخرين:

```
Message → [Agent A ✓, Agent B ✗ error, Agent C ✓]
Result: Agent A and C respond, Agent B logs error
```

## التوافق

### المزوّدون

تعمل مجموعات البث حاليًا مع:

- ✅ WhatsApp (منفذ)
- 🚧 Telegram (مخطط له)
- 🚧 Discord (مخطط له)
- 🚧 Slack (مخطط له)

### التوجيه

تعمل مجموعات البث إلى جانب التوجيه الحالي:

```json
{
  "bindings": [
    {
      "match": { "channel": "whatsapp", "peer": { "kind": "group", "id": "GROUP_A" } },
      "agentId": "alfred"
    }
  ],
  "broadcast": {
    "GROUP_B": ["agent1", "agent2"]
  }
}
```

- `GROUP_A`: يرد alfred فقط (توجيه عادي)
- `GROUP_B`: يرد agent1 وagent2 معًا (بث)

**الأسبقية:** يأخذ `broadcast` الأولوية على `bindings`.

## استكشاف الأخطاء وإصلاحها

### الوكلاء لا يردون

**تحقق من:**

1. وجود معرّفات الوكلاء في `agents.list`
2. صحة تنسيق معرّف النظير (مثل `120363403215116621@g.us`)
3. عدم وجود الوكلاء في قوائم المنع

**التصحيح:**

```bash
tail -f ~/.openclaw/logs/gateway.log | grep broadcast
```

### وكيل واحد فقط يرد

**السبب:** قد يكون معرّف النظير موجودًا في `bindings` ولكن ليس في `broadcast`.

**الحل:** أضفه إلى تكوين البث أو أزله من bindings.

### مشكلات الأداء

**إذا كان الأداء بطيئًا مع عدد كبير من الوكلاء:**

- قلّل عدد الوكلاء لكل مجموعة
- استخدم نماذج أخف (sonnet بدلًا من opus)
- تحقق من وقت بدء تشغيل sandbox

## أمثلة

### المثال 1: فريق مراجعة الشيفرة

```json
{
  "broadcast": {
    "strategy": "parallel",
    "120363403215116621@g.us": [
      "code-formatter",
      "security-scanner",
      "test-coverage",
      "docs-checker"
    ]
  },
  "agents": {
    "list": [
      {
        "id": "code-formatter",
        "workspace": "~/agents/formatter",
        "tools": { "allow": ["read", "write"] }
      },
      {
        "id": "security-scanner",
        "workspace": "~/agents/security",
        "tools": { "allow": ["read", "exec"] }
      },
      {
        "id": "test-coverage",
        "workspace": "~/agents/testing",
        "tools": { "allow": ["read", "exec"] }
      },
      { "id": "docs-checker", "workspace": "~/agents/docs", "tools": { "allow": ["read"] } }
    ]
  }
}
```

**يرسل المستخدم:** مقتطف شيفرة  
**الردود:**

- code-formatter: "Fixed indentation and added type hints"
- security-scanner: "⚠️ SQL injection vulnerability in line 12"
- test-coverage: "Coverage is 45%, missing tests for error cases"
- docs-checker: "Missing docstring for function `process_data`"

### المثال 2: دعم متعدد اللغات

```json
{
  "broadcast": {
    "strategy": "sequential",
    "+15555550123": ["detect-language", "translator-en", "translator-de"]
  },
  "agents": {
    "list": [
      { "id": "detect-language", "workspace": "~/agents/lang-detect" },
      { "id": "translator-en", "workspace": "~/agents/translate-en" },
      { "id": "translator-de", "workspace": "~/agents/translate-de" }
    ]
  }
}
```

## مرجع API

### مخطط التكوين

```typescript
interface OpenClawConfig {
  broadcast?: {
    strategy?: "parallel" | "sequential";
    [peerId: string]: string[];
  };
}
```

### الحقول

- `strategy` (اختياري): كيفية معالجة الوكلاء
  - `"parallel"` (افتراضي): يعالج جميع الوكلاء في الوقت نفسه
  - `"sequential"`: يعالج الوكلاء وفق ترتيبهم في المصفوفة
- `[peerId]`: group JID لـ WhatsApp، أو رقم E.164، أو معرّف نظير آخر
  - القيمة: مصفوفة من معرّفات الوكلاء الذين يجب أن يعالجوا الرسائل

## القيود

1. **الحد الأقصى للوكلاء:** لا يوجد حد صارم، لكن 10+ وكلاء قد يكونون بطيئين
2. **السياق المشترك:** لا يرى الوكلاء ردود بعضهم البعض (عن قصد)
3. **ترتيب الرسائل:** قد تصل الردود المتوازية بأي ترتيب
4. **حدود المعدل:** تُحتسب جميع الوكلاء ضمن حدود معدل WhatsApp

## تحسينات مستقبلية

الميزات المخطط لها:

- [ ] وضع السياق المشترك (يرى الوكلاء ردود بعضهم البعض)
- [ ] تنسيق الوكلاء (يمكن للوكلاء إرسال إشارات إلى بعضهم البعض)
- [ ] اختيار ديناميكي للوكلاء (اختيار الوكلاء بناءً على محتوى الرسالة)
- [ ] أولويات الوكلاء (يرد بعض الوكلاء قبل الآخرين)

## راجع أيضًا

- [تكوين متعدد الوكلاء](/tools/multi-agent-sandbox-tools)
- [تكوين التوجيه](/channels/channel-routing)
- [إدارة الجلسات](/concepts/session)
