---
read_when: “You want per-agent sandboxing or per-agent tool allow/deny policies in a multi-agent gateway.”
status: active
summary: تقييد الصندوق المعزول والأدوات لكل وكيل، والأولوية، والأمثلة
title: الصندوق المعزول والأدوات في تعدد الوكلاء
x-i18n:
    generated_at: "2026-04-05T12:59:35Z"
    model: gpt-5.4
    provider: openai
    source_hash: 07985f7c8fae860a7b9bf685904903a4a8f90249e95e4179cf0775a1208c0597
    source_path: tools/multi-agent-sandbox-tools.md
    workflow: 15
---

# إعداد الصندوق المعزول والأدوات في تعدد الوكلاء

يمكن لكل وكيل في إعداد متعدد الوكلاء تجاوز سياسة الصندوق المعزول
وسياسة الأدوات العامة. تغطي هذه الصفحة الإعدادات لكل وكيل، وقواعد
الأولوية، والأمثلة.

- **الواجهات الخلفية وأوضاع الصندوق المعزول**: راجع [العزل](/ar/gateway/sandboxing).
- **تصحيح الأدوات المحظورة**: راجع [الصندوق المعزول مقابل سياسة الأدوات مقابل الوضع المرتفع](/ar/gateway/sandbox-vs-tool-policy-vs-elevated) و`openclaw sandbox explain`.
- **`exec` المرتفع**: راجع [الوضع المرتفع](/tools/elevated).

المصادقة لكل وكيل: يقرأ كل وكيل من مخزن المصادقة الخاص به في `agentDir` عند
`~/.openclaw/agents/<agentId>/agent/auth-profiles.json`.
**لا** تتم مشاركة بيانات الاعتماد بين الوكلاء. لا تعِد استخدام `agentDir` بين الوكلاء مطلقًا.
إذا كنت تريد مشاركة بيانات الاعتماد، فانسخ `auth-profiles.json` إلى `agentDir` الخاص بالوكيل الآخر.

---

## أمثلة على الإعدادات

### المثال 1: وكيل شخصي + وكيل عائلي مقيّد

```json
{
  "agents": {
    "list": [
      {
        "id": "main",
        "default": true,
        "name": "Personal Assistant",
        "workspace": "~/.openclaw/workspace",
        "sandbox": { "mode": "off" }
      },
      {
        "id": "family",
        "name": "Family Bot",
        "workspace": "~/.openclaw/workspace-family",
        "sandbox": {
          "mode": "all",
          "scope": "agent"
        },
        "tools": {
          "allow": ["read"],
          "deny": ["exec", "write", "edit", "apply_patch", "process", "browser"]
        }
      }
    ]
  },
  "bindings": [
    {
      "agentId": "family",
      "match": {
        "provider": "whatsapp",
        "accountId": "*",
        "peer": {
          "kind": "group",
          "id": "120363424282127706@g.us"
        }
      }
    }
  ]
}
```

**النتيجة:**

- الوكيل `main`: يعمل على المضيف، مع وصول كامل إلى الأدوات
- الوكيل `family`: يعمل في Docker (حاوية واحدة لكل وكيل)، ومعه أداة `read` فقط

---

### المثال 2: وكيل عمل مع صندوق معزول مشترك

```json
{
  "agents": {
    "list": [
      {
        "id": "personal",
        "workspace": "~/.openclaw/workspace-personal",
        "sandbox": { "mode": "off" }
      },
      {
        "id": "work",
        "workspace": "~/.openclaw/workspace-work",
        "sandbox": {
          "mode": "all",
          "scope": "shared",
          "workspaceRoot": "/tmp/work-sandboxes"
        },
        "tools": {
          "allow": ["read", "write", "apply_patch", "exec"],
          "deny": ["browser", "gateway", "discord"]
        }
      }
    ]
  }
}
```

---

### المثال 2b: ملف تعريف عام للبرمجة + وكيل مخصص للمراسلة فقط

```json
{
  "tools": { "profile": "coding" },
  "agents": {
    "list": [
      {
        "id": "support",
        "tools": { "profile": "messaging", "allow": ["slack"] }
      }
    ]
  }
}
```

**النتيجة:**

- تحصل الوكلاء الافتراضية على أدوات البرمجة
- الوكيل `support` مخصص للمراسلة فقط (+ أداة Slack)

---

### المثال 3: أوضاع صندوق معزول مختلفة لكل وكيل

```json
{
  "agents": {
    "defaults": {
      "sandbox": {
        "mode": "non-main", // الافتراضي العام
        "scope": "session"
      }
    },
    "list": [
      {
        "id": "main",
        "workspace": "~/.openclaw/workspace",
        "sandbox": {
          "mode": "off" // تجاوز: main لا يُعزل أبدًا
        }
      },
      {
        "id": "public",
        "workspace": "~/.openclaw/workspace-public",
        "sandbox": {
          "mode": "all", // تجاوز: public دائمًا داخل الصندوق المعزول
          "scope": "agent"
        },
        "tools": {
          "allow": ["read"],
          "deny": ["exec", "write", "edit", "apply_patch"]
        }
      }
    ]
  }
}
```

---

## أولوية الإعدادات

عندما توجد إعدادات عامة (`agents.defaults.*`) وإعدادات خاصة بالوكيل (`agents.list[].*`) معًا:

### إعدادات الصندوق المعزول

تتجاوز الإعدادات الخاصة بالوكيل الإعدادات العامة:

```
agents.list[].sandbox.mode > agents.defaults.sandbox.mode
agents.list[].sandbox.scope > agents.defaults.sandbox.scope
agents.list[].sandbox.workspaceRoot > agents.defaults.sandbox.workspaceRoot
agents.list[].sandbox.workspaceAccess > agents.defaults.sandbox.workspaceAccess
agents.list[].sandbox.docker.* > agents.defaults.sandbox.docker.*
agents.list[].sandbox.browser.* > agents.defaults.sandbox.browser.*
agents.list[].sandbox.prune.* > agents.defaults.sandbox.prune.*
```

**ملاحظات:**

- تتجاوز `agents.list[].sandbox.{docker,browser,prune}.*` القيم `agents.defaults.sandbox.{docker,browser,prune}.*` لذلك الوكيل (ويتم تجاهلها عندما تُحل قيمة نطاق الصندوق المعزول إلى `"shared"`).

### قيود الأدوات

ترتيب التصفية هو:

1. **ملف تعريف الأداة** (`tools.profile` أو `agents.list[].tools.profile`)
2. **ملف تعريف أداة المزوّد** (`tools.byProvider[provider].profile` أو `agents.list[].tools.byProvider[provider].profile`)
3. **سياسة الأدوات العامة** (`tools.allow` / `tools.deny`)
4. **سياسة أداة المزوّد** (`tools.byProvider[provider].allow/deny`)
5. **سياسة الأدوات الخاصة بالوكيل** (`agents.list[].tools.allow/deny`)
6. **سياسة مزوّد الوكيل** (`agents.list[].tools.byProvider[provider].allow/deny`)
7. **سياسة أدوات الصندوق المعزول** (`tools.sandbox.tools` أو `agents.list[].tools.sandbox.tools`)
8. **سياسة أدوات الوكيل الفرعي** (`tools.subagents.tools`، عند الاقتضاء)

يمكن لكل مستوى زيادة تقييد الأدوات، لكنه لا يستطيع إعادة منح الأدوات المرفوضة في المستويات السابقة.
إذا تم تعيين `agents.list[].tools.sandbox.tools`، فإنه يستبدل `tools.sandbox.tools` لذلك الوكيل.
إذا تم تعيين `agents.list[].tools.profile`، فإنه يتجاوز `tools.profile` لذلك الوكيل.
تقبل مفاتيح أدوات المزوّد إما `provider` (مثل `google-antigravity`) أو `provider/model` (مثل `openai/gpt-5.4`).

تدعم سياسات الأدوات صيغ الاختصار `group:*` التي تتوسع إلى عدة أدوات. راجع [مجموعات الأدوات](/ar/gateway/sandbox-vs-tool-policy-vs-elevated#tool-groups-shorthands) للحصول على القائمة الكاملة.

يمكن لتجاوزات الوضع المرتفع لكل وكيل (`agents.list[].tools.elevated`) زيادة تقييد `exec` المرتفع لوكلاء محددين. راجع [الوضع المرتفع](/tools/elevated) للتفاصيل.

---

## الترحيل من وكيل واحد

**قبل (وكيل واحد):**

```json
{
  "agents": {
    "defaults": {
      "workspace": "~/.openclaw/workspace",
      "sandbox": {
        "mode": "non-main"
      }
    }
  },
  "tools": {
    "sandbox": {
      "tools": {
        "allow": ["read", "write", "apply_patch", "exec"],
        "deny": []
      }
    }
  }
}
```

**بعد (تعدد الوكلاء مع ملفات تعريف مختلفة):**

```json
{
  "agents": {
    "list": [
      {
        "id": "main",
        "default": true,
        "workspace": "~/.openclaw/workspace",
        "sandbox": { "mode": "off" }
      }
    ]
  }
}
```

يتم ترحيل إعدادات `agent.*` القديمة بواسطة `openclaw doctor`؛ ويفضَّل استخدام `agents.defaults` + `agents.list` مستقبلًا.

---

## أمثلة على تقييد الأدوات

### وكيل للقراءة فقط

```json
{
  "tools": {
    "allow": ["read"],
    "deny": ["exec", "write", "edit", "apply_patch", "process"]
  }
}
```

### وكيل تنفيذ آمن (من دون تعديل الملفات)

```json
{
  "tools": {
    "allow": ["read", "exec", "process"],
    "deny": ["write", "edit", "apply_patch", "browser", "gateway"]
  }
}
```

### وكيل مخصص للتواصل فقط

```json
{
  "tools": {
    "sessions": { "visibility": "tree" },
    "allow": ["sessions_list", "sessions_send", "sessions_history", "session_status"],
    "deny": ["exec", "write", "edit", "apply_patch", "read", "browser"]
  }
}
```

تعيد `sessions_history` في ملف التعريف هذا عرض استرجاع محدودًا ومنقحًا
بدلًا من تفريغ خام للنص الحواري. ويزيل الاسترجاع الخاص بالمساعد وسوم التفكير،
وبنية `<relevant-memories>`،
وحمولات XML النصية العادية لاستدعاءات الأدوات
(بما في ذلك `<tool_call>...</tool_call>`،
و`<function_call>...</function_call>`، و`<tool_calls>...</tool_calls>`،
و`<function_calls>...</function_calls>`، وكتل استدعاءات الأدوات المقتطعة)،
وبنية استدعاءات الأدوات المخفّضة، ورموز التحكم الخاصة بالنماذج المسرّبة بتنسيق ASCII/العرض الكامل،
وXML المشوّه لاستدعاءات أدوات MiniMax قبل التنقيح/الاقتطاع.

---

## مشكلة شائعة: `"non-main"`

تعتمد `agents.defaults.sandbox.mode: "non-main"` على `session.mainKey` (الافتراضي `"main"`)،
وليس على معرّف الوكيل. تحصل جلسات المجموعات/القنوات دائمًا على مفاتيحها الخاصة، لذلك
تُعامل على أنها غير main وسيتم عزلها. إذا كنت تريد أن لا يُعزل وكيل أبدًا،
فعيّن `agents.list[].sandbox.mode: "off"`.

---

## الاختبار

بعد إعداد الصندوق المعزول والأدوات في تعدد الوكلاء:

1. **تحقق من تحديد الوكيل:**

   ```exec
   openclaw agents list --bindings
   ```

2. **تحقق من حاويات الصندوق المعزول:**

   ```exec
   docker ps --filter "name=openclaw-sbx-"
   ```

3. **اختبر قيود الأدوات:**
   - أرسل رسالة تتطلب أدوات مقيّدة
   - تحقق من أن الوكيل لا يستطيع استخدام الأدوات المرفوضة

4. **راقب السجلات:**

   ```exec
   tail -f "${OPENCLAW_STATE_DIR:-$HOME/.openclaw}/logs/gateway.log" | grep -E "routing|sandbox|tools"
   ```

---

## استكشاف الأخطاء وإصلاحها

### الوكيل غير معزول رغم `mode: "all"`

- تحقق مما إذا كانت هناك قيمة عامة `agents.defaults.sandbox.mode` تتجاوز ذلك
- تكون إعدادات الوكيل الخاصة ذات أولوية، لذا عيّن `agents.list[].sandbox.mode: "all"`

### الأدوات ما تزال متاحة رغم قائمة الرفض

- تحقق من ترتيب تصفية الأدوات: عام → وكيل → صندوق معزول → وكيل فرعي
- يمكن لكل مستوى زيادة التقييد فقط، لا إعادة المنح
- تحقق عبر السجلات: `[tools] filtering tools for agent:${agentId}`

### الحاوية غير معزولة لكل وكيل

- عيّن `scope: "agent"` في إعدادات الصندوق المعزول الخاصة بالوكيل
- القيمة الافتراضية هي `"session"`، ما ينشئ حاوية واحدة لكل جلسة

---

## راجع أيضًا

- [العزل](/ar/gateway/sandboxing) -- المرجع الكامل للصندوق المعزول (الأوضاع، والنطاقات، والواجهات الخلفية، والصور)
- [الصندوق المعزول مقابل سياسة الأدوات مقابل الوضع المرتفع](/ar/gateway/sandbox-vs-tool-policy-vs-elevated) -- لتصحيح "لماذا تم حظر هذا؟"
- [الوضع المرتفع](/tools/elevated)
- [التوجيه متعدد الوكلاء](/ar/concepts/multi-agent)
- [إعدادات الصندوق المعزول](/ar/gateway/configuration-reference#agentsdefaultssandbox)
- [إدارة الجلسات](/ar/concepts/session)
