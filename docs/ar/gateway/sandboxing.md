---
read_when: You want a dedicated explanation of sandboxing or need to tune agents.defaults.sandbox.
status: active
summary: 'كيف يعمل sandboxing في OpenClaw: الأوضاع، والنطاقات، والوصول إلى مساحة العمل، والصور'
title: Sandboxing
x-i18n:
    generated_at: "2026-04-05T12:44:57Z"
    model: gpt-5.4
    provider: openai
    source_hash: 756ebd5b9806c23ba720a311df7e3b4ffef6ce41ba4315ee4b36b5ea87b26e60
    source_path: gateway/sandboxing.md
    workflow: 15
---

# Sandboxing

يمكن لـ OpenClaw تشغيل **الأدوات داخل واجهات sandbox الخلفية** لتقليل مساحة الضرر.
هذا **اختياري** ويُتحكم فيه عبر التكوين (`agents.defaults.sandbox` أو
`agents.list[].sandbox`). وإذا كان sandboxing معطلًا، فستعمل الأدوات على المضيف.
يبقى Gateway على المضيف؛ بينما يعمل تنفيذ الأدوات داخل sandbox معزول
عند التمكين.

هذا ليس حدًا أمنيًا مثاليًا، لكنه يحد بشكل ملموس من الوصول إلى نظام الملفات
والعمليات عندما يقوم النموذج بشيء غير حكيم.

## ما الذي يتم وضعه داخل sandbox

- تنفيذ الأدوات (`exec` و`read` و`write` و`edit` و`apply_patch` و`process` وما إلى ذلك).
- متصفح sandbox اختياري (`agents.defaults.sandbox.browser`).
  - افتراضيًا، يبدأ متصفح sandbox تلقائيًا (للتأكد من إمكانية الوصول إلى CDP) عندما تحتاجه أداة المتصفح.
    قم بالتكوين عبر `agents.defaults.sandbox.browser.autoStart` و`agents.defaults.sandbox.browser.autoStartTimeoutMs`.
  - افتراضيًا، تستخدم حاويات متصفح sandbox شبكة Docker مخصصة (`openclaw-sandbox-browser`) بدلًا من شبكة `bridge` العامة.
    قم بالتكوين عبر `agents.defaults.sandbox.browser.network`.
  - يقيّد الخيار الاختياري `agents.defaults.sandbox.browser.cdpSourceRange` دخول CDP على حافة الحاوية باستخدام قائمة سماح CIDR ‏(مثل `172.21.0.1/32`).
  - يكون الوصول إلى noVNC للمراقبة محميًا بكلمة مرور افتراضيًا؛ ويصدر OpenClaw عنوان URL لرمز مميز قصير العمر يخدم صفحة bootstrap محلية ويفتح noVNC مع كلمة المرور داخل جزء URL ‏(وليس في query/header logs).
  - يسمح `agents.defaults.sandbox.browser.allowHostControl` للجلسات الموجودة داخل sandbox باستهداف متصفح المضيف صراحةً.
  - تتحكم قوائم السماح الاختيارية في `target: "custom"`: ‏`allowedControlUrls` و`allowedControlHosts` و`allowedControlPorts`.

ما لا يتم وضعه داخل sandbox:

- عملية Gateway نفسها.
- أي أداة يُسمح لها صراحةً بالتشغيل خارج sandbox ‏(مثل `tools.elevated`).
  - **يتجاوز exec المرتفع sandboxing ويستخدم مسار الهروب المكوَّن (`gateway` افتراضيًا، أو `node` عندما يكون هدف exec هو `node`).**
  - إذا كان sandboxing معطلًا، فإن `tools.elevated` لا يغيّر التنفيذ (إذ إنه يعمل بالفعل على المضيف). راجع [الوضع المرتفع](/tools/elevated).

## الأوضاع

يتحكم `agents.defaults.sandbox.mode` في **وقت** استخدام sandboxing:

- `"off"`: لا يوجد sandboxing.
- `"non-main"`: وضع sandbox للجلسات **غير الرئيسية** فقط (الافتراضي إذا كنت تريد المحادثات العادية على المضيف).
- `"all"`: كل جلسة تعمل داخل sandbox.
  ملاحظة: يعتمد `"non-main"` على `session.mainKey` ‏(الافتراضي `"main"`)، وليس على معرّف الوكيل.
  تستخدم جلسات المجموعة/القناة مفاتيحها الخاصة، لذلك تُحسب على أنها غير رئيسية وستوضع داخل sandbox.

## النطاق

يتحكم `agents.defaults.sandbox.scope` في **عدد الحاويات** التي يتم إنشاؤها:

- `"agent"` ‏(الافتراضي): حاوية واحدة لكل وكيل.
- `"session"`: حاوية واحدة لكل جلسة.
- `"shared"`: حاوية واحدة مشتركة بين جميع الجلسات الموجودة داخل sandbox.

## الواجهة الخلفية

يتحكم `agents.defaults.sandbox.backend` في **بيئة التشغيل** التي توفر sandbox:

- `"docker"` ‏(الافتراضي): بيئة sandbox محلية معتمدة على Docker.
- `"ssh"`: بيئة sandbox بعيدة عامة معتمدة على SSH.
- `"openshell"`: بيئة sandbox معتمدة على OpenShell.

يوجد التكوين الخاص بـ SSH تحت `agents.defaults.sandbox.ssh`.
ويوجد التكوين الخاص بـ OpenShell تحت `plugins.entries.openshell.config`.

### اختيار واجهة خلفية

|                     | Docker                           | SSH                             | OpenShell                                              |
| ------------------- | -------------------------------- | ------------------------------- | ------------------------------------------------------ |
| **مكان التشغيل**    | حاوية محلية                      | أي مضيف يمكن الوصول إليه عبر SSH | sandbox مُدار بواسطة OpenShell                         |
| **الإعداد**         | `scripts/sandbox-setup.sh`       | مفتاح SSH + مضيف الهدف         | تمكين plugin OpenShell                                 |
| **نموذج مساحة العمل** | bind-mount أو نسخ               | بعيد-معتمد (بذر مرة واحدة)      | `mirror` أو `remote`                                   |
| **التحكم في الشبكة** | `docker.network` ‏(الافتراضي: none) | يعتمد على المضيف البعيد        | يعتمد على OpenShell                                    |
| **متصفح sandbox**   | مدعوم                            | غير مدعوم                       | غير مدعوم بعد                                           |
| **Bind mounts**     | `docker.binds`                   | غير متاح                        | غير متاح                                               |
| **الأفضل لـ**       | التطوير المحلي، والعزل الكامل    | نقل الحمل إلى جهاز بعيد         | sandboxes بعيدة مُدارة مع مزامنة ثنائية الاتجاه اختيارية |

### الواجهة الخلفية SSH

استخدم `backend: "ssh"` عندما تريد أن يضع OpenClaw أدوات `exec` وأدوات الملفات وقراءات الوسائط داخل sandbox على
أي جهاز يمكن الوصول إليه عبر SSH.

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "ssh",
        scope: "session",
        workspaceAccess: "rw",
        ssh: {
          target: "user@gateway-host:22",
          workspaceRoot: "/tmp/openclaw-sandboxes",
          strictHostKeyChecking: true,
          updateHostKeys: true,
          identityFile: "~/.ssh/id_ed25519",
          certificateFile: "~/.ssh/id_ed25519-cert.pub",
          knownHostsFile: "~/.ssh/known_hosts",
          // أو استخدم SecretRefs / محتويات مضمّنة بدلًا من الملفات المحلية:
          // identityData: { source: "env", provider: "default", id: "SSH_IDENTITY" },
          // certificateData: { source: "env", provider: "default", id: "SSH_CERTIFICATE" },
          // knownHostsData: { source: "env", provider: "default", id: "SSH_KNOWN_HOSTS" },
        },
      },
    },
  },
}
```

كيف يعمل ذلك:

- ينشئ OpenClaw جذرًا بعيدًا لكل نطاق تحت `sandbox.ssh.workspaceRoot`.
- عند أول استخدام بعد الإنشاء أو إعادة الإنشاء، يقوم OpenClaw ببذر مساحة العمل البعيدة تلك من مساحة العمل المحلية مرة واحدة.
- بعد ذلك، تعمل `exec` و`read` و`write` و`edit` و`apply_patch` وقراءات وسائط المطالبة وتحضير الوسائط الواردة مباشرة على مساحة العمل البعيدة عبر SSH.
- لا يقوم OpenClaw بمزامنة التغييرات البعيدة مرة أخرى إلى مساحة العمل المحلية تلقائيًا.

مواد المصادقة:

- `identityFile` و`certificateFile` و`knownHostsFile`: تستخدم ملفاتك المحلية الحالية وتمريرها عبر تكوين OpenSSH.
- `identityData` و`certificateData` و`knownHostsData`: تستخدم سلاسل مضمّنة أو SecretRefs. ويقوم OpenClaw بحلها عبر لقطة وقت تشغيل الأسرار العادية، ويكتبها إلى ملفات مؤقتة بأذونات `0600`، ثم يحذفها عند انتهاء جلسة SSH.
- إذا تم تعيين كل من `*File` و`*Data` للعنصر نفسه، فإن `*Data` تفوز لتلك الجلسة من SSH.

هذا نموذج **بعيد-معتمد**. تصبح مساحة العمل البعيدة عبر SSH هي حالة sandbox الحقيقية بعد البذر الأولي.

النتائج المهمة:

- لا تكون التعديلات المحلية على المضيف التي تتم خارج OpenClaw بعد خطوة البذر مرئية عن بُعد حتى تعيد إنشاء sandbox.
- يؤدي `openclaw sandbox recreate` إلى حذف الجذر البعيد لكل نطاق ثم البذر مرة أخرى من المحلي عند الاستخدام التالي.
- لا يتم دعم متصفح sandbox على الواجهة الخلفية SSH.
- لا تنطبق إعدادات `sandbox.docker.*` على الواجهة الخلفية SSH.

### الواجهة الخلفية OpenShell

استخدم `backend: "openshell"` عندما تريد من OpenClaw أن يضع الأدوات داخل sandbox في
بيئة بعيدة مُدارة بواسطة OpenShell. للحصول على دليل الإعداد الكامل، ومرجع
التكوين، ومقارنة أوضاع مساحة العمل، راجع صفحة
[OpenShell](/gateway/openshell) المخصصة.

يعيد OpenShell استخدام النقل الأساسي نفسه عبر SSH وجسر نظام الملفات البعيد الذي يستخدمه
الـ backend العام SSH، ويضيف دورة حياة خاصة بـ OpenShell
‏(`sandbox create/get/delete` و`sandbox ssh-config`) بالإضافة إلى وضع مساحة العمل الاختياري `mirror`.

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
          mode: "remote", // mirror | remote
          remoteWorkspaceDir: "/sandbox",
          remoteAgentWorkspaceDir: "/agent",
        },
      },
    },
  },
}
```

أوضاع OpenShell:

- `mirror` ‏(الافتراضي): تظل مساحة العمل المحلية هي المعتمدة. ويزامن OpenClaw الملفات المحلية إلى OpenShell قبل exec ويزامن مساحة العمل البعيدة مرة أخرى بعد exec.
- `remote`: تصبح مساحة عمل OpenShell هي المعتمدة بعد إنشاء sandbox. ويقوم OpenClaw ببذر مساحة العمل البعيدة مرة واحدة من مساحة العمل المحلية، ثم تعمل أدوات الملفات وexec مباشرة على sandbox البعيد من دون مزامنة التغييرات مرة أخرى.

تفاصيل النقل البعيد:

- يطلب OpenClaw من OpenShell إعداد SSH خاصًا بـ sandbox عبر `openshell sandbox ssh-config <name>`.
- تكتب النواة تكوين SSH هذا إلى ملف مؤقت، وتفتح جلسة SSH، وتعيد استخدام جسر نظام الملفات البعيد نفسه المستخدم مع `backend: "ssh"`.
- في وضع `mirror` فقط تختلف دورة الحياة: مزامنة من المحلي إلى البعيد قبل exec، ثم مزامنة مرة أخرى بعد exec.

القيود الحالية لـ OpenShell:

- لا يتم دعم متصفح sandbox بعد
- لا يتم دعم `sandbox.docker.binds` على الواجهة الخلفية OpenShell
- تظل مفاتيح وقت التشغيل الخاصة بـ Docker تحت `sandbox.docker.*` منطبقة فقط على الواجهة الخلفية Docker

#### أوضاع مساحة العمل

يملك OpenShell نموذجين لمساحة العمل. وهذا هو الجزء الأهم عمليًا.

##### `mirror`

استخدم `plugins.entries.openshell.config.mode: "mirror"` عندما تريد أن **تظل مساحة العمل المحلية هي المعتمدة**.

السلوك:

- قبل `exec`، يزامن OpenClaw مساحة العمل المحلية إلى sandbox الخاص بـ OpenShell.
- بعد `exec`، يزامن OpenClaw مساحة العمل البعيدة مرة أخرى إلى مساحة العمل المحلية.
- ما زالت أدوات الملفات تعمل عبر جسر sandbox، لكن مساحة العمل المحلية تظل مصدر الحقيقة بين الأدوار.

استخدم هذا عندما:

- تعدّل الملفات محليًا خارج OpenClaw وتريد أن تظهر تلك التغييرات في sandbox تلقائيًا
- تريد أن يتصرف sandbox الخاص بـ OpenShell بطريقة تشبه الواجهة الخلفية Docker قدر الإمكان
- تريد أن تعكس مساحة عمل المضيف عمليات الكتابة داخل sandbox بعد كل دور exec

المقايضة:

- تكلفة مزامنة إضافية قبل exec وبعده

##### `remote`

استخدم `plugins.entries.openshell.config.mode: "remote"` عندما تريد أن **تصبح مساحة عمل OpenShell هي المعتمدة**.

السلوك:

- عندما يُنشأ sandbox لأول مرة، يقوم OpenClaw ببذر مساحة العمل البعيدة من مساحة العمل المحلية مرة واحدة.
- بعد ذلك، تعمل `exec` و`read` و`write` و`edit` و`apply_patch` مباشرة على مساحة العمل البعيدة الخاصة بـ OpenShell.
- لا يقوم OpenClaw **بمزامنة** التغييرات البعيدة مرة أخرى إلى مساحة العمل المحلية بعد exec.
- ما زالت قراءات الوسائط وقت المطالبة تعمل لأن أدوات الملفات والوسائط تقرأ عبر جسر sandbox بدلًا من افتراض مسار مضيف محلي.
- يكون النقل عبر SSH إلى sandbox الخاص بـ OpenShell الذي تعيده `openshell sandbox ssh-config`.

نتائج مهمة:

- إذا عدّلت الملفات على المضيف خارج OpenClaw بعد خطوة البذر، فلن يرى sandbox البعيد **تلك التغييرات تلقائيًا**.
- إذا أُعيد إنشاء sandbox، فستُبذَر مساحة العمل البعيدة مرة أخرى من مساحة العمل المحلية.
- مع `scope: "agent"` أو `scope: "shared"`، تتم مشاركة مساحة العمل البعيدة هذه على النطاق نفسه.

استخدم هذا عندما:

- يجب أن يعيش sandbox في الأساس على الجانب البعيد الخاص بـ OpenShell
- تريد تقليل حمل المزامنة لكل دور
- لا تريد أن تؤدي تعديلات المضيف المحلية إلى الكتابة فوق حالة sandbox البعيدة بصمت

اختر `mirror` إذا كنت تنظر إلى sandbox على أنه بيئة تنفيذ مؤقتة.
واختر `remote` إذا كنت تنظر إلى sandbox على أنه مساحة العمل الحقيقية.

#### دورة حياة OpenShell

تظل sandboxes الخاصة بـ OpenShell مُدارة عبر دورة حياة sandbox العادية:

- يعرض `openclaw sandbox list` بيئات تشغيل OpenShell بالإضافة إلى بيئات Docker
- يحذف `openclaw sandbox recreate` بيئة التشغيل الحالية ويتيح لـ OpenClaw إعادة إنشائها عند الاستخدام التالي
- يكون منطق التنظيف prune مدركًا للواجهة الخلفية أيضًا

في وضع `remote`، تكون إعادة الإنشاء مهمة بشكل خاص:

- تؤدي إعادة الإنشاء إلى حذف مساحة العمل البعيدة المعتمدة لذلك النطاق
- يؤدي الاستخدام التالي إلى بذر مساحة عمل بعيدة جديدة من مساحة العمل المحلية

أما في وضع `mirror`، فتعيد إعادة الإنشاء تعيين بيئة التنفيذ البعيدة أساسًا
لأن مساحة العمل المحلية تظل معتمدة على أي حال.

## الوصول إلى مساحة العمل

يتحكم `agents.defaults.sandbox.workspaceAccess` في **ما الذي يمكن لـ sandbox رؤيته**:

- `"none"` ‏(الافتراضي): ترى الأدوات مساحة عمل sandbox تحت `~/.openclaw/sandboxes`.
- `"ro"`: يركّب مساحة عمل الوكيل للقراءة فقط عند `/agent` ‏(ويعطل `write`/`edit`/`apply_patch`).
- `"rw"`: يركّب مساحة عمل الوكيل للقراءة/الكتابة عند `/workspace`.

مع الواجهة الخلفية OpenShell:

- يظل وضع `mirror` يستخدم مساحة العمل المحلية كمصدر معتمد بين أدوار exec
- يستخدم وضع `remote` مساحة عمل OpenShell البعيدة كمصدر معتمد بعد البذر الأولي
- ما زالت `workspaceAccess: "ro"` و`"none"` تقيّدان سلوك الكتابة بالطريقة نفسها

يتم نسخ الوسائط الواردة إلى مساحة عمل sandbox النشطة (`media/inbound/*`).
ملاحظة Skills: أداة `read` مرتبطة بجذر sandbox. ومع `workspaceAccess: "none"`،
يعكس OpenClaw الـ Skills المؤهلة إلى مساحة عمل sandbox (`.../skills`) حتى
يمكن قراءتها. ومع `"rw"`، تصبح Skills الموجودة في مساحة العمل قابلة للقراءة من
`/workspace/skills`.

## Bind mounts مخصصة

يقوم `agents.defaults.sandbox.docker.binds` بتركيب أدلة مضيف إضافية داخل الحاوية.
التنسيق: `host:container:mode` ‏(مثل `"/home/user/source:/source:rw"`).

يتم **دمج** bind mounts العامة وتلك الخاصة بكل وكيل (ولا يتم استبدالها). وتحت `scope: "shared"`، يتم تجاهل bind mounts الخاصة بكل وكيل.

يقوم `agents.defaults.sandbox.browser.binds` بتركيب أدلة مضيف إضافية داخل **حاوية متصفح sandbox** فقط.

- عند تعيينها (بما في ذلك `[]`)، فإنها تستبدل `agents.defaults.sandbox.docker.binds` بالنسبة إلى حاوية المتصفح.
- عند حذفها، تعود حاوية المتصفح إلى `agents.defaults.sandbox.docker.binds` ‏(للتوافق مع الإصدارات السابقة).

مثال (مصدر للقراءة فقط + دليل بيانات إضافي):

```json5
{
  agents: {
    defaults: {
      sandbox: {
        docker: {
          binds: ["/home/user/source:/source:ro", "/var/data/myapp:/data:ro"],
        },
      },
    },
    list: [
      {
        id: "build",
        sandbox: {
          docker: {
            binds: ["/mnt/cache:/cache:rw"],
          },
        },
      },
    ],
  },
}
```

ملاحظات أمنية:

- تتجاوز bind mounts نظام ملفات sandbox: فهي تكشف مسارات المضيف بالنمط الذي تحدده (`:ro` أو `:rw`).
- يمنع OpenClaw مصادر الربط الخطرة (مثل: `docker.sock` و`/etc` و`/proc` و`/sys` و`/dev` وعمليات التركيب الأصلية التي قد تكشفها).
- يمنع OpenClaw أيضًا جذور بيانات الاعتماد الشائعة في الدليل المنزلي مثل `~/.aws` و`~/.cargo` و`~/.config` و`~/.docker` و`~/.gnupg` و`~/.netrc` و`~/.npm` و`~/.ssh`.
- لا يعتمد التحقق من bind على مطابقة السلاسل فقط. بل يقوم OpenClaw بتطبيع مسار المصدر، ثم يحله مرة أخرى عبر أعمق أصل موجود قبل إعادة التحقق من المسارات المحظورة والجذور المسموح بها.
- وهذا يعني أن محاولات الهروب عبر الروابط الرمزية في الأصل ما زالت تفشل بشكل مغلق حتى عندما لا تكون الورقة النهائية موجودة بعد. مثال: ما زال `/workspace/run-link/new-file` يُحل إلى `/var/run/...` إذا كان `run-link` يشير إلى هناك.
- يتم أيضًا تحويل جذور المصدر المسموح بها إلى الشكل المعتمد نفسه، لذا فإن المسار الذي يبدو داخل قائمة السماح فقط قبل حل الروابط الرمزية سيُرفض أيضًا على أنه `outside allowed roots`.
- يجب أن تكون عمليات التركيب الحساسة (الأسرار، ومفاتيح SSH، وبيانات اعتماد الخدمة) بنمط `:ro` ما لم تكن هناك حاجة مطلقة.
- اجمع ذلك مع `workspaceAccess: "ro"` إذا كنت تحتاج فقط إلى وصول قراءة إلى مساحة العمل؛ إذ تبقى أوضاع bind مستقلة.
- راجع [Sandbox vs Tool Policy vs Elevated](/gateway/sandbox-vs-tool-policy-vs-elevated) لمعرفة كيفية تفاعل bind mounts مع سياسة الأدوات وexec المرتفع.

## الصور + الإعداد

صورة Docker الافتراضية: `openclaw-sandbox:bookworm-slim`

ابنها مرة واحدة:

```bash
scripts/sandbox-setup.sh
```

ملاحظة: الصورة الافتراضية **لا** تتضمن Node. وإذا احتاجت Skill إلى Node ‏(أو
بيئات تشغيل أخرى)، فقم إما بإنشاء صورة مخصصة أو بتثبيتها عبر
`sandbox.docker.setupCommand` ‏(يتطلب خروجًا شبكيًا + جذرًا قابلاً للكتابة +
مستخدم root).

إذا كنت تريد صورة sandbox أكثر عملية مع أدوات شائعة (مثل
`curl` و`jq` و`nodejs` و`python3` و`git`)، فابنِ:

```bash
scripts/sandbox-common-setup.sh
```

ثم اضبط `agents.defaults.sandbox.docker.image` على
`openclaw-sandbox-common:bookworm-slim`.

صورة متصفح sandbox:

```bash
scripts/sandbox-browser-setup.sh
```

افتراضيًا، تعمل حاويات Docker sandbox **من دون شبكة**.
قم بالتجاوز عبر `agents.defaults.sandbox.docker.network`.

تطبق صورة متصفح sandbox المضمّنة أيضًا إعدادات تشغيل افتراضية محافظة لـ Chromium
لأحمال العمل المعبأة داخل حاويات. تتضمن افتراضيات الحاوية الحالية ما يلي:

- `--remote-debugging-address=127.0.0.1`
- `--remote-debugging-port=<derived from OPENCLAW_BROWSER_CDP_PORT>`
- `--user-data-dir=${HOME}/.chrome`
- `--no-first-run`
- `--no-default-browser-check`
- `--disable-3d-apis`
- `--disable-gpu`
- `--disable-dev-shm-usage`
- `--disable-background-networking`
- `--disable-extensions`
- `--disable-features=TranslateUI`
- `--disable-breakpad`
- `--disable-crash-reporter`
- `--disable-software-rasterizer`
- `--no-zygote`
- `--metrics-recording-only`
- `--renderer-process-limit=2`
- `--no-sandbox` و`--disable-setuid-sandbox` عند تمكين `noSandbox`.
- تكون أعلام تقوية الرسوميات الثلاثة (`--disable-3d-apis`،
  و`--disable-software-rasterizer`، و`--disable-gpu`) اختيارية، وتكون مفيدة
  عندما تفتقر الحاويات إلى دعم GPU. اضبط `OPENCLAW_BROWSER_DISABLE_GRAPHICS_FLAGS=0`
  إذا كانت أحمال العمل لديك تتطلب WebGL أو ميزات ثلاثية الأبعاد/متصفح أخرى.
- يكون `--disable-extensions` مفعّلًا افتراضيًا ويمكن تعطيله عبر
  `OPENCLAW_BROWSER_DISABLE_EXTENSIONS=0` للتدفقات التي تعتمد على الإضافات.
- يتحكم في `--renderer-process-limit=2` المتغير
  `OPENCLAW_BROWSER_RENDERER_PROCESS_LIMIT=<N>`، حيث تُبقي القيمة `0` افتراضي Chromium.

إذا كنت تحتاج إلى ملف تعريف تشغيل مختلف، فاستخدم صورة متصفح مخصصة ووفّر
entrypoint الخاص بك. وبالنسبة إلى ملفات تعريف Chromium المحلية (غير المعبأة)، استخدم
`browser.extraArgs` لإلحاق أعلام تشغيل إضافية.

الافتراضيات الأمنية:

- يتم حظر `network: "host"`.
- يتم حظر `network: "container:<id>"` افتراضيًا (بسبب خطر تجاوز الانضمام إلى namespace).
- تجاوز break-glass: ‏`agents.defaults.sandbox.docker.dangerouslyAllowContainerNamespaceJoin: true`.

توجد تثبيتات Docker وgateway المعبأ داخل الحاوية هنا:
[Docker](/install/docker)

وبالنسبة إلى عمليات نشر Docker gateway، يمكن لـ `scripts/docker/setup.sh` تهيئة تكوين sandbox.
اضبط `OPENCLAW_SANDBOX=1` ‏(أو `true`/`yes`/`on`) لتمكين هذا المسار. ويمكنك
تجاوز موقع المقبس عبر `OPENCLAW_DOCKER_SOCKET`. الإعداد الكامل ومرجع
متغيرات البيئة: [Docker](/install/docker#agent-sandbox)

## setupCommand ‏(إعداد الحاوية لمرة واحدة)

يعمل `setupCommand` **مرة واحدة** بعد إنشاء حاوية sandbox ‏(وليس في كل تشغيل).
ويُنفذ داخل الحاوية عبر `sh -lc`.

المسارات:

- عام: `agents.defaults.sandbox.docker.setupCommand`
- لكل وكيل: `agents.list[].sandbox.docker.setupCommand`

المشكلات الشائعة:

- قيمة `docker.network` الافتراضية هي `"none"` ‏(من دون خروج)، لذا ستفشل عمليات تثبيت الحزم.
- يتطلب `docker.network: "container:<id>"` القيمة `dangerouslyAllowContainerNamespaceJoin: true` وهو مخصص لحالات break-glass فقط.
- تمنع `readOnlyRoot: true` عمليات الكتابة؛ اضبط `readOnlyRoot: false` أو أنشئ صورة مخصصة.
- يجب أن يكون `user` هو root لعمليات تثبيت الحزم (احذف `user` أو اضبط `user: "0:0"`).
- لا يرث Sandbox exec قيمة `process.env` من المضيف. استخدم
  `agents.defaults.sandbox.docker.env` ‏(أو صورة مخصصة) لمفاتيح API الخاصة بـ Skills.

## سياسة الأدوات + مخارج الهروب

تظل سياسات السماح/المنع للأدوات منطبقة قبل قواعد sandbox. فإذا كانت الأداة ممنوعة
عالميًا أو لكل وكيل، فلن يعيدها sandboxing.

يمثل `tools.elevated` مخرج هروب صريح يشغّل `exec` خارج sandbox ‏(`gateway` افتراضيًا، أو `node` عندما يكون هدف exec هو `node`).
وتنطبق توجيهات `/exec` فقط على المرسلين المصرح لهم وتستمر لكل جلسة؛ ولتعطيل
`exec` بشكل صارم، استخدم منع سياسة الأدوات (راجع [Sandbox vs Tool Policy vs Elevated](/gateway/sandbox-vs-tool-policy-vs-elevated)).

تصحيح الأخطاء:

- استخدم `openclaw sandbox explain` لفحص وضع sandbox الفعّال، وسياسة الأدوات، ومفاتيح التكوين الخاصة بالإصلاح.
- راجع [Sandbox vs Tool Policy vs Elevated](/gateway/sandbox-vs-tool-policy-vs-elevated) للحصول على نموذج ذهني لسؤال “لماذا تم حظر هذا؟”.
  أبقِ الأمور مقيدة بإحكام.

## تجاوزات متعددة الوكلاء

يمكن لكل وكيل تجاوز sandbox + الأدوات:
`agents.list[].sandbox` و`agents.list[].tools` ‏(بالإضافة إلى `agents.list[].tools.sandbox.tools` لسياسة أدوات sandbox).
راجع [Sandbox والأدوات متعددة الوكلاء](/tools/multi-agent-sandbox-tools) لمعرفة الأولوية.

## مثال تمكين أدنى

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",
        scope: "session",
        workspaceAccess: "none",
      },
    },
  },
}
```

## وثائق ذات صلة

- [OpenShell](/gateway/openshell) -- إعداد الواجهة الخلفية المُدارة لـ sandbox، وأوضاع مساحة العمل، ومرجع التكوين
- [تكوين Sandbox](/gateway/configuration-reference#agentsdefaultssandbox)
- [Sandbox vs Tool Policy vs Elevated](/gateway/sandbox-vs-tool-policy-vs-elevated) -- تصحيح “لماذا تم حظر هذا؟”
- [Sandbox والأدوات متعددة الوكلاء](/tools/multi-agent-sandbox-tools) -- التجاوزات لكل وكيل والأولوية
- [الأمان](/gateway/security)
