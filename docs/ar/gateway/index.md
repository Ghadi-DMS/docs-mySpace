---
read_when:
    - تشغيل عملية البوابة أو تصحيحها
summary: دليل تشغيل لخدمة البوابة ودورة حياتها وعملياتها
title: دليل تشغيل البوابة
x-i18n:
    generated_at: "2026-04-05T12:43:25Z"
    model: gpt-5.4
    provider: openai
    source_hash: 0ec17674370de4e171779389c83580317308a4f07ebf335ad236a47238af18e1
    source_path: gateway/index.md
    workflow: 15
---

# دليل تشغيل البوابة

استخدم هذه الصفحة للتشغيل في اليوم الأول والعمليات التشغيلية اللاحقة لخدمة البوابة.

<CardGroup cols={2}>
  <Card title="استكشاف الأخطاء العميق" icon="siren" href="/gateway/troubleshooting">
    تشخيصات تبدأ من الأعراض مع سلاسل أوامر دقيقة وتواقيع السجلات.
  </Card>
  <Card title="الإعداد" icon="sliders" href="/gateway/configuration">
    دليل إعداد موجّه بالمهام + مرجع إعداد كامل.
  </Card>
  <Card title="إدارة الأسرار" icon="key-round" href="/gateway/secrets">
    عقد SecretRef، وسلوك لقطات وقت التشغيل، وعمليات الترحيل/إعادة التحميل.
  </Card>
  <Card title="عقد خطة الأسرار" icon="shield-check" href="/gateway/secrets-plan-contract">
    قواعد `secrets apply` الدقيقة الخاصة بالهدف/المسار وسلوك ملف تعريف المصادقة المعتمد على المراجع فقط.
  </Card>
</CardGroup>

## تشغيل محلي خلال 5 دقائق

<Steps>
  <Step title="ابدأ البوابة">

```bash
openclaw gateway --port 18789
# عكس debug/trace إلى stdio
openclaw gateway --port 18789 --verbose
# اقتل المستمع على المنفذ المحدد بالقوة، ثم ابدأ
openclaw gateway --force
```

  </Step>

  <Step title="تحقق من سلامة الخدمة">

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
```

الخط الأساسي السليم: `Runtime: running` و`RPC probe: ok`.

  </Step>

  <Step title="تحقق من جاهزية القناة">

```bash
openclaw channels status --probe
```

مع بوابة يمكن الوصول إليها، يُشغّل هذا فحوصات قنوات مباشرة لكل حساب وتدقيقات اختيارية.
أما إذا تعذر الوصول إلى البوابة، فيعود CLI إلى ملخصات قنوات معتمدة على الإعداد فقط
بدلًا من إخراج الفحص المباشر.

  </Step>
</Steps>

<Note>
تراقب إعادة تحميل إعداد البوابة مسار ملف الإعداد النشط (المحلول من القيم الافتراضية للملف الشخصي/الحالة، أو `OPENCLAW_CONFIG_PATH` عند ضبطه).
الوضع الافتراضي هو `gateway.reload.mode="hybrid"`.
بعد أول تحميل ناجح، تخدم العملية الجارية لقطة الإعداد النشطة داخل الذاكرة؛ وتستبدل إعادة التحميل الناجحة تلك اللقطة بشكل ذري.
</Note>

## نموذج وقت التشغيل

- عملية واحدة دائمة التفعيل للتوجيه، ومستوى التحكم، واتصالات القنوات.
- منفذ واحد متعدد الإرسال من أجل:
  - التحكم/RPC عبر WebSocket
  - HTTP APIs، متوافقة مع OpenAI (`/v1/models`, `/v1/embeddings`, `/v1/chat/completions`, `/v1/responses`, `/tools/invoke`)
  - واجهة المستخدم الخاصة بالتحكم والخطافات
- وضع الربط الافتراضي: `loopback`.
- المصادقة مطلوبة افتراضيًا. وتستخدم إعدادات السر المشترك
  `gateway.auth.token` / `gateway.auth.password` (أو
  `OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD`)، ويمكن لإعدادات
  reverse-proxy غير المعتمدة على loopback استخدام `gateway.auth.mode: "trusted-proxy"`.

## نقاط النهاية المتوافقة مع OpenAI

أصبح سطح التوافق الأعلى فائدة في OpenClaw الآن:

- `GET /v1/models`
- `GET /v1/models/{id}`
- `POST /v1/embeddings`
- `POST /v1/chat/completions`
- `POST /v1/responses`

لماذا تهم هذه المجموعة:

- تبدأ معظم تكاملات Open WebUI وLobeChat وLibreChat بالفحص عبر `/v1/models`.
- تتوقع كثير من خطوط RAG والذاكرة وجود `/v1/embeddings`.
- يفضّل العملاء الأصليون للوكلاء بشكل متزايد `/v1/responses`.

ملاحظة تخطيطية:

- الأمر `/v1/models` موجه للوكلاء أولًا: فهو يعيد `openclaw` و`openclaw/default` و`openclaw/<agentId>`.
- `openclaw/default` هو الاسم المستعار الثابت الذي يرتبط دائمًا بالوكيل الافتراضي المُعد.
- استخدم `x-openclaw-model` عندما تريد تجاوز موفّر/نموذج الخلفية؛ وإلا يظل إعداد النموذج والتضمين العادي الخاص بالوكيل المحدد هو المتحكم.

تعمل جميع هذه النقاط على منفذ البوابة الرئيسي نفسه وتستخدم حد مصادقة المشغّل الموثوق نفسه المستخدم في بقية Gateway HTTP API.

### أسبقية المنفذ ووضع الربط

| الإعداد | ترتيب التحليل |
| ------------ | ------------------------------------------------------------- |
| منفذ البوابة | `--port` ← `OPENCLAW_GATEWAY_PORT` ← `gateway.port` ← `18789` |
| وضع الربط | CLI/التجاوز ← `gateway.bind` ← `loopback` |

### أوضاع إعادة التحميل السريع

| `gateway.reload.mode` | السلوك |
| --------------------- | ------------------------------------------ |
| `off`                 | لا توجد إعادة تحميل للإعداد |
| `hot`                 | تطبيق التغييرات الآمنة للسخونة فقط |
| `restart`             | إعادة تشغيل عند التغييرات التي تتطلب إعادة تحميل |
| `hybrid` (الافتراضي) | تطبيق سريع عندما يكون آمنًا، وإعادة تشغيل عند الحاجة |

## مجموعة أوامر المشغّل

```bash
openclaw gateway status
openclaw gateway status --deep   # يضيف فحص خدمة على مستوى النظام
openclaw gateway status --json
openclaw gateway install
openclaw gateway restart
openclaw gateway stop
openclaw secrets reload
openclaw logs --follow
openclaw doctor
```

إن `gateway status --deep` مخصص لاكتشاف الخدمات الإضافية (LaunchDaemons/وحدات systemd على مستوى النظام/schtasks)، وليس فحص RPC أعمق لسلامة النظام.

## بوابات متعددة (على المضيف نفسه)

ينبغي أن يشغّل معظم التثبيتات بوابة واحدة لكل جهاز. ويمكن لبوابة واحدة أن تستضيف عدة
وكلاء وقنوات.

أنت تحتاج إلى عدة بوابات فقط عندما تريد عمدًا العزل أو بوت إنقاذ.

فحوصات مفيدة:

```bash
openclaw gateway status --deep
openclaw gateway probe
```

ما الذي يجب توقعه:

- يمكن أن يعرض `gateway status --deep` الرسالة `Other gateway-like services detected (best effort)`
  ويطبع تلميحات تنظيف عندما تكون تثبيتات launchd/systemd/schtasks القديمة لا تزال موجودة.
- يمكن أن يحذّر `gateway probe` من `multiple reachable gateways` عندما يجيب أكثر من هدف واحد.
- إذا كان ذلك مقصودًا، فاعزل المنافذ، والإعداد/الحالة، وجذور مساحة العمل لكل بوابة.

الإعداد المفصل: [/gateway/multiple-gateways](/gateway/multiple-gateways).

## الوصول البعيد

المفضل: Tailscale/VPN.
البديل: نفق SSH.

```bash
ssh -N -L 18789:127.0.0.1:18789 user@host
```

ثم صِل العملاء بـ `ws://127.0.0.1:18789` محليًا.

<Warning>
لا تتجاوز أنفاق SSH مصادقة البوابة. ففي إعدادات السر المشترك، لا يزال يتعين على العملاء
إرسال `token`/`password` حتى عبر النفق. أما في الأوضاع المعتمدة على الهوية،
فيجب أن يستوفي الطلب أيضًا مسار المصادقة ذاك.
</Warning>

راجع: [البوابة البعيدة](/gateway/remote)، [المصادقة](/gateway/authentication)، [Tailscale](/gateway/tailscale).

## الإشراف ودورة حياة الخدمة

استخدم التشغيل تحت إشراف من أجل موثوقية تشبه الإنتاج.

<Tabs>
  <Tab title="macOS (launchd)">

```bash
openclaw gateway install
openclaw gateway status
openclaw gateway restart
openclaw gateway stop
```

تكون تسميات LaunchAgent هي `ai.openclaw.gateway` (الافتراضي) أو `ai.openclaw.<profile>` (للملفات الشخصية المسماة). ويقوم `openclaw doctor` بتدقيق انحراف إعداد الخدمة وإصلاحه.

  </Tab>

  <Tab title="Linux (systemd user)">

```bash
openclaw gateway install
systemctl --user enable --now openclaw-gateway[-<profile>].service
openclaw gateway status
```

للاستمرار بعد تسجيل الخروج، فعّل lingering:

```bash
sudo loginctl enable-linger <user>
```

مثال يدوي على وحدة مستخدم عندما تحتاج إلى مسار تثبيت مخصص:

```ini
[Unit]
Description=OpenClaw Gateway
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/local/bin/openclaw gateway --port 18789
Restart=always
RestartSec=5
TimeoutStopSec=30
TimeoutStartSec=30
SuccessExitStatus=0 143
KillMode=control-group

[Install]
WantedBy=default.target
```

  </Tab>

  <Tab title="Windows (أصلي)">

```powershell
openclaw gateway install
openclaw gateway status --json
openclaw gateway restart
openclaw gateway stop
```

يستخدم بدء التشغيل المُدار أصليًا في Windows مهمة مجدولة باسم `OpenClaw Gateway`
(أو `OpenClaw Gateway (<profile>)` للملفات الشخصية المسماة). وإذا تم رفض إنشاء المهمة المجدولة،
فيعود OpenClaw إلى مشغّل داخل مجلد Startup لكل مستخدم
يشير إلى `gateway.cmd` داخل دليل الحالة.

  </Tab>

  <Tab title="Linux (خدمة نظام)">

استخدم وحدة نظام لمضيفين متعددَي المستخدمين/دائمَي التشغيل.

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now openclaw-gateway[-<profile>].service
```

استخدم متن الخدمة نفسه كما في وحدة المستخدم، لكن ثبّته تحت
`/etc/systemd/system/openclaw-gateway[-<profile>].service` واضبط
`ExecStart=` إذا كان الملف التنفيذي `openclaw` لديك موجودًا في مكان آخر.

  </Tab>
</Tabs>

## بوابات متعددة على مضيف واحد

يجب أن تشغّل معظم الإعدادات **بوابة واحدة**.
استخدم عدة بوابات فقط من أجل العزل/الاعتمادية الصارمة (مثل ملف تعريف إنقاذ).

قائمة التحقق لكل مثيل:

- `gateway.port` فريد
- `OPENCLAW_CONFIG_PATH` فريد
- `OPENCLAW_STATE_DIR` فريد
- `agents.defaults.workspace` فريد

مثال:

```bash
OPENCLAW_CONFIG_PATH=~/.openclaw/a.json OPENCLAW_STATE_DIR=~/.openclaw-a openclaw gateway --port 19001
OPENCLAW_CONFIG_PATH=~/.openclaw/b.json OPENCLAW_STATE_DIR=~/.openclaw-b openclaw gateway --port 19002
```

راجع: [بوابات متعددة](/gateway/multiple-gateways).

### المسار السريع لملف تعريف التطوير

```bash
openclaw --dev setup
openclaw --dev gateway --allow-unconfigured
openclaw --dev status
```

تتضمن القيم الافتراضية حالة/إعدادًا معزولين ومنفذ بوابة أساسي `19001`.

## مرجع البروتوكول السريع (منظور المشغّل)

- يجب أن يكون أول إطار عميل هو `connect`.
- تعيد البوابة لقطة `hello-ok` (`presence`, `health`, `stateVersion`, `uptimeMs`, limits/policy).
- `hello-ok.features.methods` / `events` هي قائمة اكتشاف متحفظة، وليست
  تفريغًا مولدًا لكل مسار مساعد قابل للاستدعاء.
- الطلبات: `req(method, params)` ← `res(ok/payload|error)`.
- تشمل الأحداث الشائعة `connect.challenge` و`agent` و`chat` و
  `session.message` و`session.tool` و`sessions.changed` و`presence` و`tick` و
  `health` و`heartbeat` وأحداث دورة حياة الاقتران/الموافقة و`shutdown`.

تشغيلات الوكيل ذات مرحلتين:

1. إقرار فوري بالقبول (`status:"accepted"`)
2. استجابة نهائية عند الاكتمال (`status:"ok"|"error"`)، مع تدفق أحداث `agent` بينهما.

راجع وثائق البروتوكول الكاملة: [بروتوكول البوابة](/gateway/protocol).

## فحوصات تشغيلية

### التوفر

- افتح WS وأرسل `connect`.
- توقّع استجابة `hello-ok` مع لقطة.

### الجاهزية

```bash
openclaw gateway status
openclaw channels status --probe
openclaw health
```

### استرداد الفجوات

لا تتم إعادة تشغيل الأحداث. وعند وجود فجوات في التسلسل، حدّث الحالة (`health`، `system-presence`) قبل المتابعة.

## تواقيع الإخفاق الشائعة

| التوقيع | المشكلة المحتملة |
| -------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| `refusing to bind gateway ... without auth`                    | ربط غير loopback من دون مسار مصادقة صالح للبوابة |
| `another gateway instance is already listening` / `EADDRINUSE` | تعارض منفذ |
| `Gateway start blocked: set gateway.mode=local`                | الإعداد مضبوط على الوضع البعيد، أو أن ختم الوضع المحلي مفقود من إعداد تالف |
| `unauthorized` during connect                                  | عدم تطابق المصادقة بين العميل والبوابة |

للحصول على سلاسل التشخيص الكاملة، استخدم [استكشاف أخطاء البوابة وإصلاحها](/gateway/troubleshooting).

## ضمانات الأمان

- يفشل عملاء بروتوكول البوابة بسرعة عندما تكون البوابة غير متاحة (ولا يوجد رجوع ضمني مباشر إلى القنوات).
- يتم رفض وإغلاق الإطارات الأولى غير الصالحة/غير `connect`.
- يصدر الإغلاق السلس الحدث `shutdown` قبل إغلاق المقبس.

---

ذو صلة:

- [استكشاف الأخطاء وإصلاحها](/gateway/troubleshooting)
- [العملية الخلفية](/gateway/background-process)
- [الإعداد](/gateway/configuration)
- [السلامة](/gateway/health)
- [Doctor](/gateway/doctor)
- [المصادقة](/gateway/authentication)
