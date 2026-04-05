---
read_when:
    - تريد sandboxes مُدارة سحابيًا بدلًا من Docker المحلي
    - أنت تقوم بإعداد plugin الخاص بـ OpenShell
    - تحتاج إلى الاختيار بين وضعي mirror وremote لمساحة العمل
summary: استخدم OpenShell كواجهة خلفية مُدارة لـ sandbox لوكلاء OpenClaw
title: OpenShell
x-i18n:
    generated_at: "2026-04-05T12:43:44Z"
    model: gpt-5.4
    provider: openai
    source_hash: aaf9027d0632a70fb86455f8bc46dc908ff766db0eb0cdf2f7df39c715241ead
    source_path: gateway/openshell.md
    workflow: 15
---

# OpenShell

OpenShell هو واجهة خلفية مُدارة لـ sandbox في OpenClaw. فبدلًا من تشغيل حاويات Docker
محليًا، يفوض OpenClaw دورة حياة sandbox إلى CLI ‏`openshell`،
الذي يوفّر بيئات بعيدة مع تنفيذ أوامر قائم على SSH.

يعيد plugin الخاص بـ OpenShell استخدام نقل SSH الأساسي نفسه وجسر نظام الملفات البعيد
كما في [واجهة SSH الخلفية](/gateway/sandboxing#ssh-backend) العامة. ويضيف
دورة حياة خاصة بـ OpenShell (`sandbox create/get/delete` و`sandbox ssh-config`)
ووضع مساحة عمل اختياريًا هو `mirror`.

## المتطلبات المسبقة

- أن يكون CLI ‏`openshell` مثبتًا وموجودًا في `PATH` (أو تعيين مسار مخصص عبر
  `plugins.entries.openshell.config.command`)
- حساب OpenShell مع وصول إلى sandbox
- تشغيل OpenClaw Gateway على المضيف

## البدء السريع

1. فعّل plugin واضبط واجهة sandbox الخلفية:

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "openshell",
        scope: "session",
        workspaceAccess: "rw",
      },
    },
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "remote",
        },
      },
    },
  },
}
```

2. أعد تشغيل Gateway. في دورة الوكيل التالية، سيُنشئ OpenClaw
   sandbox من OpenShell ويوجه تنفيذ الأدوات من خلاله.

3. تحقّق:

```bash
openclaw sandbox list
openclaw sandbox explain
```

## أوضاع مساحة العمل

هذا هو القرار الأهم عند استخدام OpenShell.

### `mirror`

استخدم `plugins.entries.openshell.config.mode: "mirror"` عندما تريد أن تظل **مساحة العمل المحلية
هي المرجع الأساسي**.

السلوك:

- قبل `exec`، يزامن OpenClaw مساحة العمل المحلية إلى sandbox الخاصة بـ OpenShell.
- بعد `exec`، يزامن OpenClaw مساحة العمل البعيدة مرة أخرى إلى مساحة العمل المحلية.
- ما زالت أدوات الملفات تعمل عبر جسر sandbox، لكن مساحة العمل المحلية
  تظل مصدر الحقيقة بين الدورات.

الأفضل لـ:

- إذا كنت تعدّل الملفات محليًا خارج OpenClaw وتريد أن تظهر هذه التغييرات في
  sandbox تلقائيًا.
- إذا كنت تريد أن تتصرف sandbox الخاصة بـ OpenShell بأكبر قدر ممكن مثل واجهة Docker الخلفية.
- إذا كنت تريد أن تعكس مساحة عمل المضيف كتابات sandbox بعد كل دورة `exec`.

المقايضة: تكلفة مزامنة إضافية قبل كل `exec` وبعده.

### `remote`

استخدم `plugins.entries.openshell.config.mode: "remote"` عندما تريد أن تصبح
**مساحة عمل OpenShell هي المرجع الأساسي**.

السلوك:

- عند إنشاء sandbox لأول مرة، يهيّئ OpenClaw مساحة العمل البعيدة من
  مساحة العمل المحلية مرة واحدة.
- بعد ذلك، تعمل `exec` و`read` و`write` و`edit` و`apply_patch`
  مباشرةً على مساحة عمل OpenShell البعيدة.
- لا يقوم OpenClaw **بمزامنة** التغييرات البعيدة مرة أخرى إلى مساحة العمل المحلية.
- وما زالت قراءات الوسائط وقت الـ prompt تعمل لأن أدوات الملفات والوسائط تقرأ عبر
  جسر sandbox.

الأفضل لـ:

- عندما يجب أن تعيش sandbox أساسًا في الجهة البعيدة.
- عندما تريد حمل مزامنة أقل لكل دورة.
- عندما لا تريد أن تؤدي تعديلات المضيف المحلية إلى الكتابة فوق حالة sandbox البعيدة بصمت.

مهم: إذا عدّلت الملفات على المضيف خارج OpenClaw بعد التهيئة الأولية،
فلن ترى sandbox البعيدة تلك التغييرات. استخدم
`openclaw sandbox recreate` لإعادة التهيئة.

### اختيار وضع

|                          | `mirror`                      | `remote`                     |
| ------------------------ | ----------------------------- | ---------------------------- |
| **مساحة العمل المرجعية** | المضيف المحلي                 | OpenShell البعيدة            |
| **اتجاه المزامنة**       | ثنائي الاتجاه (كل `exec`)     | تهيئة لمرة واحدة             |
| **الحمل لكل دورة**       | أعلى (رفع + تنزيل)            | أقل (عمليات بعيدة مباشرة)   |
| **هل تظهر التعديلات المحلية؟** | نعم، في `exec` التالية         | لا، حتى إعادة الإنشاء        |
| **الأفضل لـ**            | سير عمل التطوير               | الوكلاء طويلو التشغيل، CI    |

## مرجع التكوين

يوجد كل تكوين OpenShell تحت `plugins.entries.openshell.config`:

| المفتاح                   | النوع                    | الافتراضي     | الوصف                                               |
| ------------------------- | ------------------------ | ------------- | --------------------------------------------------- |
| `mode`                    | `"mirror"` أو `"remote"` | `"mirror"`    | وضع مزامنة مساحة العمل                              |
| `command`                 | `string`                 | `"openshell"` | المسار أو الاسم الخاص بـ CLI ‏`openshell`           |
| `from`                    | `string`                 | `"openclaw"`  | مصدر sandbox عند الإنشاء لأول مرة                  |
| `gateway`                 | `string`                 | —             | اسم OpenShell Gateway (`--gateway`)                |
| `gatewayEndpoint`         | `string`                 | —             | URL لنقطة نهاية OpenShell Gateway (`--gateway-endpoint`) |
| `policy`                  | `string`                 | —             | معرّف سياسة OpenShell لإنشاء sandbox                |
| `providers`               | `string[]`               | `[]`          | أسماء المزوّدين المطلوب إرفاقها عند إنشاء sandbox   |
| `gpu`                     | `boolean`                | `false`       | طلب موارد GPU                                       |
| `autoProviders`           | `boolean`                | `true`        | تمرير `--auto-providers` أثناء إنشاء sandbox       |
| `remoteWorkspaceDir`      | `string`                 | `"/sandbox"`  | مساحة العمل الأساسية القابلة للكتابة داخل sandbox   |
| `remoteAgentWorkspaceDir` | `string`                 | `"/agent"`    | مسار تحميل مساحة عمل الوكيل (للوصول للقراءة فقط)    |
| `timeoutSeconds`          | `number`                 | `120`         | المهلة الزمنية لعمليات CLI ‏`openshell`             |

يتم تكوين الإعدادات على مستوى sandbox (`mode` و`scope` و`workspaceAccess`) ضمن
`agents.defaults.sandbox` كما هو الحال مع أي واجهة خلفية. راجع
[Sandboxing](/gateway/sandboxing) للاطلاع على المصفوفة الكاملة.

## أمثلة

### إعداد remote بسيط

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "openshell",
      },
    },
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "remote",
        },
      },
    },
  },
}
```

### وضع mirror مع GPU

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "openshell",
        scope: "agent",
        workspaceAccess: "rw",
      },
    },
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "mirror",
          gpu: true,
          providers: ["openai"],
          timeoutSeconds: 180,
        },
      },
    },
  },
}
```

### OpenShell لكل وكيل مع Gateway مخصصة

```json5
{
  agents: {
    defaults: {
      sandbox: { mode: "off" },
    },
    list: [
      {
        id: "researcher",
        sandbox: {
          mode: "all",
          backend: "openshell",
          scope: "agent",
          workspaceAccess: "rw",
        },
      },
    ],
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "remote",
          gateway: "lab",
          gatewayEndpoint: "https://lab.example",
          policy: "strict",
        },
      },
    },
  },
}
```

## إدارة دورة الحياة

تُدار sandboxes الخاصة بـ OpenShell عبر CLI العادي الخاص بـ sandbox:

```bash
# عرض جميع بيئات sandbox runtime (Docker + OpenShell)
openclaw sandbox list

# فحص السياسة الفعلية
openclaw sandbox explain

# إعادة الإنشاء (يحذف مساحة العمل البعيدة، ويعيد التهيئة عند الاستخدام التالي)
openclaw sandbox recreate --all
```

بالنسبة إلى وضع `remote`، تكون **إعادة الإنشاء مهمة بشكل خاص**: فهي تحذف
مساحة العمل البعيدة المرجعية لذلك النطاق. وعند الاستخدام التالي، تتم تهيئة مساحة عمل بعيدة جديدة
من مساحة العمل المحلية.

أما في وضع `mirror`، فتعيد إعادة الإنشاء أساسًا ضبط بيئة التنفيذ البعيدة لأن
مساحة العمل المحلية تظل المرجع الأساسي.

### متى تعيد الإنشاء

أعد الإنشاء بعد تغيير أي من الآتي:

- `agents.defaults.sandbox.backend`
- `plugins.entries.openshell.config.from`
- `plugins.entries.openshell.config.mode`
- `plugins.entries.openshell.config.policy`

```bash
openclaw sandbox recreate --all
```

## القيود الحالية

- لا يتم دعم browser الخاص بـ sandbox على واجهة OpenShell الخلفية.
- لا ينطبق `sandbox.docker.binds` على OpenShell.
- تنطبق مفاتيح runtime الخاصة بـ Docker ضمن `sandbox.docker.*` على
  واجهة Docker الخلفية فقط.

## كيف يعمل

1. يستدعي OpenClaw الأمر `openshell sandbox create` (مع العلامات `--from` و`--gateway`،
   و`--policy` و`--providers` و`--gpu` وفق التكوين).
2. يستدعي OpenClaw الأمر `openshell sandbox ssh-config <name>` للحصول على تفاصيل
   اتصال SSH الخاصة بـ sandbox.
3. يكتب core تكوين SSH إلى ملف مؤقت ويفتح جلسة SSH باستخدام
   جسر نظام الملفات البعيد نفسه الموجود في واجهة SSH الخلفية العامة.
4. في وضع `mirror`: يزامن من المحلي إلى البعيد قبل `exec`، ثم يشغّل، ثم يزامن مرة أخرى بعد `exec`.
5. في وضع `remote`: يهيّئ مرة واحدة عند الإنشاء، ثم يعمل مباشرة على مساحة العمل
   البعيدة.

## راجع أيضًا

- [Sandboxing](/gateway/sandboxing) -- الأوضاع، والنطاقات، ومقارنة الواجهات الخلفية
- [Sandbox vs Tool Policy vs Elevated](/gateway/sandbox-vs-tool-policy-vs-elevated) -- تصحيح أخطاء الأدوات المحظورة
- [Multi-Agent Sandbox and Tools](/tools/multi-agent-sandbox-tools) -- تجاوزات لكل وكيل
- [Sandbox CLI](/cli/sandbox) -- أوامر `openclaw sandbox`
