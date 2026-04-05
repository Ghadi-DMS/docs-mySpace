---
read_when:
    - ربط Codex أو Claude Code أو عميل MCP آخر بقنوات مدعومة من OpenClaw
    - تشغيل `openclaw mcp serve`
    - إدارة تعريفات خوادم MCP المحفوظة في OpenClaw
summary: كشف محادثات قنوات OpenClaw عبر MCP وإدارة تعريفات خوادم MCP المحفوظة في OpenClaw
title: mcp
x-i18n:
    generated_at: "2026-04-05T12:39:20Z"
    model: gpt-5.4
    provider: openai
    source_hash: b35de9e14f96666eeca2f93c06cb214e691152f911d45ee778efe9cf5bf96cc2
    source_path: cli/mcp.md
    workflow: 15
---

# mcp

لدى `openclaw mcp` مهمتان:

- تشغيل OpenClaw كخادم MCP باستخدام `openclaw mcp serve`
- إدارة تعريفات خوادم MCP الصادرة والمملوكة لـ OpenClaw باستخدام `list` و`show`،
  و`set`، و`unset`

بمعنى آخر:

- `serve` يعني أن OpenClaw يعمل كخادم MCP
- `list` / `show` / `set` / `unset` تعني أن OpenClaw يعمل كسجل
  MCP من جهة العميل لخوادم MCP أخرى قد تستهلكها بيئات تشغيله لاحقًا

استخدم [`openclaw acp`](/cli/acp) عندما يجب أن يستضيف OpenClaw
جلسة حزمة ترميز بنفسه ويوجه وقت التشغيل هذا عبر ACP.

## OpenClaw كخادم MCP

هذا هو مسار `openclaw mcp serve`.

## متى تستخدم `serve`

استخدم `openclaw mcp serve` عندما:

- يجب أن يتحدث Codex أو Claude Code أو عميل MCP آخر مباشرةً إلى
  محادثات القنوات المدعومة من OpenClaw
- يكون لديك بالفعل Gateway محلي أو بعيد لـ OpenClaw مع جلسات موجهة
- تريد خادم MCP واحدًا يعمل عبر واجهات القنوات في OpenClaw بدلًا
  من تشغيل جسور منفصلة لكل قناة

استخدم [`openclaw acp`](/cli/acp) بدلًا من ذلك عندما يجب على OpenClaw استضافة
وقت تشغيل الترميز نفسه والإبقاء على جلسة الوكيل داخل OpenClaw.

## كيف يعمل

يبدأ `openclaw mcp serve` خادم MCP عبر stdio. يمتلك عميل MCP
هذه العملية. وبينما يُبقي العميل جلسة stdio مفتوحة، يتصل الجسر بـ
OpenClaw Gateway محلي أو بعيد عبر WebSocket ويكشف محادثات القنوات
الموجّهة عبر MCP.

دورة الحياة:

1. يشغّل عميل MCP الأمر `openclaw mcp serve`
2. يتصل الجسر بـ Gateway
3. تصبح الجلسات الموجّهة محادثات MCP وأدوات النصوص/السجل
4. تُصفّ أحداث البث المباشر في الذاكرة بينما يكون الجسر متصلًا
5. إذا كان وضع قناة Claude مفعّلًا، يمكن للجلسة نفسها أيضًا تلقي
   إشعارات دفع خاصة بـ Claude

سلوك مهم:

- تبدأ حالة قائمة الانتظار الحية عندما يتصل الجسر
- يُقرأ سجل النصوص الأقدم باستخدام `messages_read`
- لا توجد إشعارات دفع Claude إلا أثناء بقاء جلسة MCP حيّة
- عندما ينفصل العميل، يخرج الجسر وتختفي قائمة الانتظار الحية

## اختر وضع العميل

استخدم الجسر نفسه بطريقتين مختلفتين:

- عملاء MCP العامون: أدوات MCP قياسية فقط. استخدم `conversations_list`،
  و`messages_read`، و`events_poll`، و`events_wait`، و`messages_send`، و
  أدوات الموافقة.
- Claude Code: أدوات MCP القياسية بالإضافة إلى مكيّف القناة الخاص بـ Claude.
  فعّل `--claude-channel-mode on` أو اترك القيمة الافتراضية `auto`.

حاليًا، يتصرف `auto` بالطريقة نفسها التي يتصرف بها `on`. لا يوجد اكتشاف
لقدرات العميل بعد.

## ما الذي يكشفه `serve`

يستخدم الجسر بيانات تعريف مسار الجلسة الموجودة في Gateway لكشف
المحادثات المدعومة بالقنوات. تظهر المحادثة عندما يكون لدى OpenClaw بالفعل
حالة جلسة مع مسار معروف مثل:

- `channel`
- بيانات تعريف المستلم أو الوجهة
- `accountId` اختياري
- `threadId` اختياري

وهذا يمنح عملاء MCP مكانًا واحدًا من أجل:

- إدراج المحادثات الموجهة الحديثة
- قراءة سجل النصوص الحديث
- انتظار أحداث واردة جديدة
- إرسال رد عبر المسار نفسه
- رؤية طلبات الموافقة التي تصل بينما يكون الجسر متصلًا

## الاستخدام

```bash
# Gateway محلي
openclaw mcp serve

# Gateway بعيد
openclaw mcp serve --url wss://gateway-host:18789 --token-file ~/.openclaw/gateway.token

# Gateway بعيد مع مصادقة كلمة مرور
openclaw mcp serve --url wss://gateway-host:18789 --password-file ~/.openclaw/gateway.password

# تمكين سجلات الجسر المفصلة
openclaw mcp serve --verbose

# تعطيل إشعارات الدفع الخاصة بـ Claude
openclaw mcp serve --claude-channel-mode off
```

## أدوات الجسر

يكشف الجسر الحالي أدوات MCP التالية:

- `conversations_list`
- `conversation_get`
- `messages_read`
- `attachments_fetch`
- `events_poll`
- `events_wait`
- `messages_send`
- `permissions_list_open`
- `permissions_respond`

### `conversations_list`

يسرد المحادثات الحديثة المدعومة بالجلسات والتي لديها بالفعل بيانات تعريف
المسار في حالة جلسة Gateway.

مرشحات مفيدة:

- `limit`
- `search`
- `channel`
- `includeDerivedTitles`
- `includeLastMessage`

### `conversation_get`

يعيد محادثة واحدة حسب `session_key`.

### `messages_read`

يقرأ رسائل النصوص الحديثة لمحادثة واحدة مدعومة بالجلسة.

### `attachments_fetch`

يستخرج كتل محتوى الرسائل غير النصية من رسالة نصية واحدة. هذه
رؤية للبيانات الوصفية فوق محتوى النصوص، وليست مخزن كتل مرفقات
دائمًا مستقلًا.

### `events_poll`

يقرأ الأحداث الحية الموجودة في قائمة الانتظار بدءًا من مؤشر رقمي.

### `events_wait`

يجري long-poll حتى يصل الحدث المطابق التالي الموجود في قائمة الانتظار أو
تنتهي المهلة.

استخدم هذا عندما يحتاج عميل MCP عام إلى تسليم شبه فوري دون
بروتوكول دفع خاص بـ Claude.

### `messages_send`

يرسل نصًا مرة أخرى عبر المسار نفسه المسجل بالفعل على الجلسة.

السلوك الحالي:

- يتطلب مسار محادثة موجودًا
- يستخدم القناة الخاصة بالجلسة، والمستلم، ومعرّف الحساب، ومعرّف السلسلة
- يرسل نصًا فقط

### `permissions_list_open`

يسرد طلبات موافقة exec/plugin المعلقة التي لاحظها الجسر منذ
اتصاله بـ Gateway.

### `permissions_respond`

يعالج طلب موافقة exec/plugin معلقًا واحدًا باستخدام:

- `allow-once`
- `allow-always`
- `deny`

## نموذج الأحداث

يحتفظ الجسر بقائمة انتظار أحداث داخل الذاكرة أثناء اتصاله.

أنواع الأحداث الحالية:

- `message`
- `exec_approval_requested`
- `exec_approval_resolved`
- `plugin_approval_requested`
- `plugin_approval_resolved`
- `claude_permission_request`

قيود مهمة:

- قائمة الانتظار حية فقط؛ تبدأ عندما يبدأ جسر MCP
- لا يعيد `events_poll` و`events_wait` تشغيل سجل Gateway الأقدم
  من تلقاء نفسيهما
- يجب قراءة التراكم الدائم باستخدام `messages_read`

## إشعارات قناة Claude

يمكن للجسر أيضًا كشف إشعارات قنوات خاصة بـ Claude. وهذا هو
المكافئ في OpenClaw لمكيّف قناة Claude Code: تظل أدوات MCP القياسية متاحة،
لكن الرسائل الواردة الحية يمكن أن تصل أيضًا كإشعارات MCP خاصة بـ Claude.

العلامات:

- `--claude-channel-mode off`: أدوات MCP قياسية فقط
- `--claude-channel-mode on`: تمكين إشعارات قناة Claude
- `--claude-channel-mode auto`: القيمة الافتراضية الحالية؛ سلوك الجسر نفسه كما في `on`

عند تمكين وضع قناة Claude، يعلن الخادم عن قدرات Claude التجريبية
ويمكنه إصدار:

- `notifications/claude/channel`
- `notifications/claude/channel/permission`

السلوك الحالي للجسر:

- تُعاد توجيه رسائل النصوص الواردة من نوع `user` كـ
  `notifications/claude/channel`
- تُتتبع طلبات أذونات Claude المستلمة عبر MCP داخل الذاكرة
- إذا أرسلت المحادثة المرتبطة لاحقًا `yes abcde` أو `no abcde`، فإن الجسر
  يحوّل ذلك إلى `notifications/claude/channel/permission`
- هذه الإشعارات مخصصة للجلسة الحية فقط؛ إذا انفصل عميل MCP،
  فلا يوجد هدف دفع

هذا مقصود ليكون خاصًا بالعميل. يجب على عملاء MCP العامين الاعتماد على
أدوات الاستطلاع القياسية.

## إعداد عميل MCP

مثال على إعداد عميل stdio:

```json
{
  "mcpServers": {
    "openclaw": {
      "command": "openclaw",
      "args": [
        "mcp",
        "serve",
        "--url",
        "wss://gateway-host:18789",
        "--token-file",
        "/path/to/gateway.token"
      ]
    }
  }
}
```

بالنسبة إلى معظم عملاء MCP العامين، ابدأ بسطح الأدوات القياسي وتجاهل
وضع Claude. فعّل وضع Claude فقط للعملاء الذين يفهمون فعليًا
طرق الإشعارات الخاصة بـ Claude.

## الخيارات

يدعم `openclaw mcp serve` ما يلي:

- `--url <url>`: عنوان URL لـ Gateway WebSocket
- `--token <token>`: رمز Gateway المميز
- `--token-file <path>`: قراءة الرمز المميز من ملف
- `--password <password>`: كلمة مرور Gateway
- `--password-file <path>`: قراءة كلمة المرور من ملف
- `--claude-channel-mode <auto|on|off>`: وضع إشعارات Claude
- `-v`, `--verbose`: سجلات مفصلة على stderr

فضّل `--token-file` أو `--password-file` بدلًا من الأسرار المضمنة متى أمكن.

## الأمان وحدود الثقة

لا يخترع الجسر التوجيه. بل يكشف فقط المحادثات التي يعرف Gateway
بالفعل كيفية توجيهها.

وهذا يعني:

- تظل قوائم سماح المرسلين، والاقتران، والثقة على مستوى القناة تابعة لـ
  إعدادات قناة OpenClaw الأساسية
- يمكن لـ `messages_send` الرد فقط عبر مسار مخزن موجود
- تكون حالة الموافقة حية/في الذاكرة فقط لجلسة الجسر الحالية
- يجب أن تستخدم مصادقة الجسر عناصر تحكم الرمز المميز أو كلمة المرور في Gateway نفسها
  التي ستثق بها لأي عميل Gateway بعيد آخر

إذا كانت محادثة مفقودة من `conversations_list`، فالسبب المعتاد ليس
إعداد MCP. بل هو فقدان بيانات تعريف المسار أو عدم اكتمالها في جلسة
Gateway الأساسية.

## الاختبار

يوفر OpenClaw اختبار Docker دخانيًا حتميًا لهذا الجسر:

```bash
pnpm test:docker:mcp-channels
```

هذا الاختبار الدخاني:

- يبدأ حاوية Gateway مهيأة مسبقًا
- يبدأ حاوية ثانية تشغّل `openclaw mcp serve`
- يتحقق من اكتشاف المحادثات، وقراءات النصوص، وقراءات بيانات تعريف المرفقات،
  وسلوك قائمة انتظار الأحداث الحية، وتوجيه الإرسال الصادر
- يتحقق من إشعارات القنوات والأذونات بنمط Claude عبر
  جسر MCP الحقيقي عبر stdio

هذه أسرع طريقة لإثبات أن الجسر يعمل من دون توصيل حساب
Telegram أو Discord أو iMessage حقيقي أثناء تشغيل الاختبار.

للحصول على سياق اختبار أوسع، راجع [الاختبار](/help/testing).

## استكشاف الأخطاء وإصلاحها

### لم يتم إرجاع أي محادثات

هذا يعني عادةً أن جلسة Gateway غير قابلة للتوجيه بالفعل. أكّد أن
الجلسة الأساسية تحتوي على بيانات تعريف مسار مخزنة للقناة/الموفر، والمستلم،
وبيانات تعريف الحساب/السلسلة الاختيارية.

### يفوّت `events_poll` أو `events_wait` الرسائل الأقدم

هذا متوقع. تبدأ قائمة الانتظار الحية عندما يتصل الجسر. اقرأ سجل
النصوص الأقدم باستخدام `messages_read`.

### لا تظهر إشعارات Claude

تحقق من كل ما يلي:

- أبقى العميل جلسة stdio MCP مفتوحة
- كانت قيمة `--claude-channel-mode` هي `on` أو `auto`
- يفهم العميل فعليًا طرق الإشعارات الخاصة بـ Claude
- حدثت الرسالة الواردة بعد اتصال الجسر

### الموافقات مفقودة

لا يعرض `permissions_list_open` سوى طلبات الموافقة التي تمت ملاحظتها بينما كان الجسر
متصلًا. وهو ليس API لسجل موافقات دائم.

## OpenClaw كسجل عميل MCP

هذا هو مسار `openclaw mcp list` و`show` و`set` و`unset`.

لا تكشف هذه الأوامر OpenClaw عبر MCP. بل تدير تعريفات خوادم MCP
المملوكة لـ OpenClaw ضمن `mcp.servers` في إعدادات OpenClaw.

تلك التعريفات المحفوظة مخصصة لبيئات التشغيل التي يقوم OpenClaw بتشغيلها أو إعدادها
لاحقًا، مثل Pi المضمّن ومكيفات وقت التشغيل الأخرى. يخزّن OpenClaw
التعريفات مركزيًا بحيث لا تحتاج تلك البيئات إلى الاحتفاظ بقوائم MCP
مكررة خاصة بها.

سلوك مهم:

- هذه الأوامر تقرأ أو تكتب إعدادات OpenClaw فقط
- لا تتصل بخادم MCP الهدف
- لا تتحقق مما إذا كان الأمر أو عنوان URL أو النقل البعيد
  قابلًا للوصول الآن
- تقرر مكيّفات وقت التشغيل أشكال النقل التي تدعمها فعليًا
  وقت التنفيذ

## تعريفات خوادم MCP المحفوظة

يخزن OpenClaw أيضًا سجلًا خفيفًا لخوادم MCP في الإعدادات للأسطح
التي تريد تعريفات MCP مُدارة من OpenClaw.

الأوامر:

- `openclaw mcp list`
- `openclaw mcp show [name]`
- `openclaw mcp set <name> <json>`
- `openclaw mcp unset <name>`

ملاحظات:

- يقوم `list` بترتيب أسماء الخوادم.
- يطبع `show` من دون اسم كائن خادم MCP المهيأ بالكامل.
- يتوقع `set` قيمة كائن JSON واحدة في سطر الأوامر.
- يفشل `unset` إذا لم يكن الخادم المسمى موجودًا.

أمثلة:

```bash
openclaw mcp list
openclaw mcp show context7 --json
openclaw mcp set context7 '{"command":"uvx","args":["context7-mcp"]}'
openclaw mcp set docs '{"url":"https://mcp.example.com"}'
openclaw mcp unset context7
```

مثال على شكل الإعدادات:

```json
{
  "mcp": {
    "servers": {
      "context7": {
        "command": "uvx",
        "args": ["context7-mcp"]
      },
      "docs": {
        "url": "https://mcp.example.com"
      }
    }
  }
}
```

### نقل stdio

يشغّل عملية فرعية محلية ويتواصل عبر stdin/stdout.

| الحقل | الوصف |
| ----- | ----- |
| `command` | الملف التنفيذي المراد تشغيله (مطلوب) |
| `args` | مصفوفة وسائط سطر الأوامر |
| `env` | متغيرات بيئة إضافية |
| `cwd` / `workingDirectory` | دليل العمل الخاص بالعملية |

### نقل SSE / HTTP

يتصل بخادم MCP بعيد عبر HTTP Server-Sent Events.

| الحقل | الوصف |
| ----- | ----- |
| `url` | عنوان URL بخاصية HTTP أو HTTPS للخادم البعيد (مطلوب) |
| `headers` | خريطة مفتاح-قيمة اختيارية لترويسات HTTP (مثل رموز المصادقة) |
| `connectionTimeoutMs` | مهلة الاتصال لكل خادم بالمللي ثانية (اختياري) |

مثال:

```json
{
  "mcp": {
    "servers": {
      "remote-tools": {
        "url": "https://mcp.example.com",
        "headers": {
          "Authorization": "Bearer <token>"
        }
      }
    }
  }
}
```

تُحجب القيم الحساسة في `url` (userinfo) و`headers` في السجلات
ومخرجات الحالة.

### نقل streamable HTTP

يمثل `streamable-http` خيار نقل إضافيًا إلى جانب `sse` و`stdio`. ويستخدم بث HTTP للاتصال ثنائي الاتجاه مع خوادم MCP البعيدة.

| الحقل | الوصف |
| ----- | ----- |
| `url` | عنوان URL بخاصية HTTP أو HTTPS للخادم البعيد (مطلوب) |
| `transport` | اضبطه على `"streamable-http"` لاختيار هذا النقل؛ وعند حذفه يستخدم OpenClaw `sse` |
| `headers` | خريطة مفتاح-قيمة اختيارية لترويسات HTTP (مثل رموز المصادقة) |
| `connectionTimeoutMs` | مهلة الاتصال لكل خادم بالمللي ثانية (اختياري) |

مثال:

```json
{
  "mcp": {
    "servers": {
      "streaming-tools": {
        "url": "https://mcp.example.com/stream",
        "transport": "streamable-http",
        "connectionTimeoutMs": 10000,
        "headers": {
          "Authorization": "Bearer <token>"
        }
      }
    }
  }
}
```

تدير هذه الأوامر الإعدادات المحفوظة فقط. فهي لا تبدأ جسر القناة،
ولا تفتح جلسة عميل MCP حية، ولا تثبت أن الخادم الهدف قابل للوصول.

## القيود الحالية

توثق هذه الصفحة الجسر كما هو مشحون اليوم.

القيود الحالية:

- يعتمد اكتشاف المحادثات على بيانات تعريف مسار جلسة Gateway الموجودة
- لا يوجد بروتوكول دفع عام يتجاوز المكيّف الخاص بـ Claude
- لا توجد حتى الآن أدوات لتعديل الرسائل أو التفاعل معها
- يتصل نقل HTTP/SSE/streamable-http بخادم بعيد واحد؛ ولا يوجد upstream متعدد الإرسال بعد
- لا يتضمن `permissions_list_open` سوى الموافقات التي تمت ملاحظتها بينما كان الجسر
  متصلًا
