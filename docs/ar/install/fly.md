---
read_when:
    - نشر OpenClaw على Fly.io
    - إعداد Fly volumes والأسرار وتكوين التشغيل الأول
summary: نشر Fly.io خطوة بخطوة لـ OpenClaw مع تخزين دائم وHTTPS تلقائي
title: Fly.io
x-i18n:
    generated_at: "2026-04-05T12:46:55Z"
    model: gpt-5.4
    provider: openai
    source_hash: a5f8c2c03295d786c0d8df98f8a5ae9335fa0346a188b81aae3e07d566a2c0ef
    source_path: install/fly.md
    workflow: 15
---

# النشر على Fly.io

**الهدف:** تشغيل OpenClaw Gateway على جهاز [Fly.io](https://fly.io) مع تخزين دائم وHTTPS تلقائي ووصول Discord/القنوات.

## ما الذي تحتاج إليه

- تثبيت [flyctl CLI](https://fly.io/docs/hands-on/install-flyctl/)
- حساب Fly.io ‏(تعمل الفئة المجانية)
- مصادقة النموذج: مفتاح API لمزوّد النموذج الذي اخترته
- بيانات اعتماد القنوات: رمز Discord bot، ورمز Telegram، وما إلى ذلك

## المسار السريع للمبتدئين

1. انسخ المستودع → خصص `fly.toml`
2. أنشئ التطبيق + volume → اضبط الأسرار
3. انشر باستخدام `fly deploy`
4. اتصل عبر SSH لإنشاء التكوين أو استخدم Control UI

<Steps>
  <Step title="أنشئ تطبيق Fly">
    ```bash
    # انسخ المستودع
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw

    # أنشئ تطبيق Fly جديدًا (اختر اسمك الخاص)
    fly apps create my-openclaw

    # أنشئ volume دائمة (1GB تكفي عادة)
    fly volumes create openclaw_data --size 1 --region iad
    ```

    **نصيحة:** اختر منطقة قريبة منك. من الخيارات الشائعة: `lhr` ‏(لندن)، و`iad` ‏(فرجينيا)، و`sjc` ‏(سان خوسيه).

  </Step>

  <Step title="كوّن fly.toml">
    حرر `fly.toml` ليتطابق مع اسم تطبيقك ومتطلباتك.

    **ملاحظة أمنية:** يكشف التكوين الافتراضي عنوان URL عامًا. وللحصول على نشر مقوى من دون عنوان IP عام، راجع [النشر الخاص](#private-deployment-hardened) أو استخدم `fly.private.toml`.

    ```toml
    app = "my-openclaw"  # اسم تطبيقك
    primary_region = "iad"

    [build]
      dockerfile = "Dockerfile"

    [env]
      NODE_ENV = "production"
      OPENCLAW_PREFER_PNPM = "1"
      OPENCLAW_STATE_DIR = "/data"
      NODE_OPTIONS = "--max-old-space-size=1536"

    [processes]
      app = "node dist/index.js gateway --allow-unconfigured --port 3000 --bind lan"

    [http_service]
      internal_port = 3000
      force_https = true
      auto_stop_machines = false
      auto_start_machines = true
      min_machines_running = 1
      processes = ["app"]

    [[vm]]
      size = "shared-cpu-2x"
      memory = "2048mb"

    [mounts]
      source = "openclaw_data"
      destination = "/data"
    ```

    **الإعدادات الأساسية:**

    | الإعداد                         | السبب                                                                        |
    | ------------------------------- | ---------------------------------------------------------------------------- |
    | `--bind lan`                    | يربط على `0.0.0.0` حتى يتمكن proxy الخاص بـ Fly من الوصول إلى gateway       |
    | `--allow-unconfigured`          | يبدأ من دون ملف تكوين (ستنشئه لاحقًا)                                       |
    | `internal_port = 3000`          | يجب أن يطابق `--port 3000` ‏(أو `OPENCLAW_GATEWAY_PORT`) لفحوصات سلامة Fly   |
    | `memory = "2048mb"`             | 512MB صغير جدًا؛ ويوصى بـ 2GB                                                |
    | `OPENCLAW_STATE_DIR = "/data"`  | يجعل الحالة دائمة على volume                                                |

  </Step>

  <Step title="اضبط الأسرار">
    ```bash
    # مطلوب: رمز Gateway ‏(للربط غير الخاص بـ loopback)
    fly secrets set OPENCLAW_GATEWAY_TOKEN=$(openssl rand -hex 32)

    # مفاتيح API لمزوّد النموذج
    fly secrets set ANTHROPIC_API_KEY=sk-ant-...

    # اختياري: مزوّدون آخرون
    fly secrets set OPENAI_API_KEY=sk-...
    fly secrets set GOOGLE_API_KEY=...

    # رموز القنوات
    fly secrets set DISCORD_BOT_TOKEN=MTQ...
    ```

    **ملاحظات:**

    - تتطلب عمليات الربط غير الخاصة بـ loopback ‏(`--bind lan`) مسار مصادقة صالحًا لـ gateway. يستخدم مثال Fly.io هذا `OPENCLAW_GATEWAY_TOKEN`، لكن `gateway.auth.password` أو نشر `trusted-proxy` غير loopback المكوّن بشكل صحيح يفي أيضًا بالمتطلب.
    - تعامل مع هذه الرموز كما لو كانت كلمات مرور.
    - **يفضل استخدام متغيرات البيئة بدلًا من ملف التكوين** لكل مفاتيح API والرموز. فهذا يبقي الأسرار خارج `openclaw.json` حيث قد تُكشَف أو تُسجَّل عن طريق الخطأ.

  </Step>

  <Step title="انشر">
    ```bash
    fly deploy
    ```

    يبني أول نشر صورة Docker ‏(~2-3 دقائق). وتكون عمليات النشر اللاحقة أسرع.

    بعد النشر، تحقّق باستخدام:

    ```bash
    fly status
    fly logs
    ```

    يجب أن ترى:

    ```
    [gateway] listening on ws://0.0.0.0:3000 (PID xxx)
    [discord] logged in to discord as xxx
    ```

  </Step>

  <Step title="أنشئ ملف التكوين">
    اتصل بالجهاز عبر SSH لإنشاء تكوين صحيح:

    ```bash
    fly ssh console
    ```

    أنشئ دليل التكوين والملف:

    ```bash
    mkdir -p /data
    cat > /data/openclaw.json << 'EOF'
    {
      "agents": {
        "defaults": {
          "model": {
            "primary": "anthropic/claude-opus-4-6",
            "fallbacks": ["anthropic/claude-sonnet-4-6", "openai/gpt-5.4"]
          },
          "maxConcurrent": 4
        },
        "list": [
          {
            "id": "main",
            "default": true
          }
        ]
      },
      "auth": {
        "profiles": {
          "anthropic:default": { "mode": "token", "provider": "anthropic" },
          "openai:default": { "mode": "token", "provider": "openai" }
        }
      },
      "bindings": [
        {
          "agentId": "main",
          "match": { "channel": "discord" }
        }
      ],
      "channels": {
        "discord": {
          "enabled": true,
          "groupPolicy": "allowlist",
          "guilds": {
            "YOUR_GUILD_ID": {
              "channels": { "general": { "allow": true } },
              "requireMention": false
            }
          }
        }
      },
      "gateway": {
        "mode": "local",
        "bind": "auto"
      },
      "meta": {}
    }
    EOF
    ```

    **ملاحظة:** مع `OPENCLAW_STATE_DIR=/data`، يكون مسار التكوين هو `/data/openclaw.json`.

    **ملاحظة:** يمكن أن يأتي رمز Discord من أحد المصدرين التاليين:

    - متغير البيئة: `DISCORD_BOT_TOKEN` ‏(موصى به للأسرار)
    - ملف التكوين: `channels.discord.token`

    إذا كنت تستخدم متغير البيئة، فلا حاجة إلى إضافة الرمز إلى التكوين. يقرأ gateway قيمة `DISCORD_BOT_TOKEN` تلقائيًا.

    أعد التشغيل لتطبيقه:

    ```bash
    exit
    fly machine restart <machine-id>
    ```

  </Step>

  <Step title="الوصول إلى Gateway">
    ### Control UI

    افتحها في المتصفح:

    ```bash
    fly open
    ```

    أو زر `https://my-openclaw.fly.dev/`

    قم بالمصادقة باستخدام السر المشترك المكوَّن. يستخدم هذا الدليل رمز gateway
    من `OPENCLAW_GATEWAY_TOKEN`؛ وإذا انتقلت إلى مصادقة كلمة المرور، فاستخدم
    تلك الكلمة بدلًا من ذلك.

    ### السجلات

    ```bash
    fly logs              # سجلات مباشرة
    fly logs --no-tail    # السجلات الأخيرة
    ```

    ### وحدة SSH

    ```bash
    fly ssh console
    ```

  </Step>
</Steps>

## استكشاف الأخطاء وإصلاحها

### "App is not listening on expected address"

يرتبط gateway بـ `127.0.0.1` بدلًا من `0.0.0.0`.

**الإصلاح:** أضف `--bind lan` إلى أمر العملية في `fly.toml`.

### فشل فحوصات السلامة / رفض الاتصال

لا يستطيع Fly الوصول إلى gateway على المنفذ المكوَّن.

**الإصلاح:** تأكد من أن `internal_port` يطابق منفذ gateway ‏(اضبط `--port 3000` أو `OPENCLAW_GATEWAY_PORT=3000`).

### OOM / مشكلات الذاكرة

تستمر الحاوية في إعادة التشغيل أو القتل. ومن العلامات: `SIGABRT`، أو `v8::internal::Runtime_AllocateInYoungGeneration`، أو عمليات إعادة تشغيل صامتة.

**الإصلاح:** زد الذاكرة في `fly.toml`:

```toml
[[vm]]
  memory = "2048mb"
```

أو حدّث جهازًا موجودًا:

```bash
fly machine update <machine-id> --vm-memory 2048 -y
```

**ملاحظة:** 512MB صغير جدًا. وقد تعمل 1GB لكنها قد تسبب OOM تحت الحمل أو مع التسجيل المطول. **يوصى بـ 2GB.**

### مشكلات قفل Gateway

يرفض Gateway البدء مع أخطاء تفيد بأنه "قيد التشغيل بالفعل".

يحدث هذا عندما تُعاد تشغيل الحاوية لكن يبقى ملف قفل PID على volume.

**الإصلاح:** احذف ملف القفل:

```bash
fly ssh console --command "rm -f /data/gateway.*.lock"
fly machine restart <machine-id>
```

يوجد ملف القفل في `/data/gateway.*.lock` ‏(وليس في دليل فرعي).

### عدم قراءة التكوين

يتجاوز `--allow-unconfigured` حارس بدء التشغيل فقط. وهو لا ينشئ أو يصلح `/data/openclaw.json`، لذا تأكد من وجود التكوين الحقيقي لديك وأنه يتضمن `gateway.mode="local"` عندما تريد بدء gateway محليًا بشكل عادي.

تحقّق من وجود التكوين:

```bash
fly ssh console --command "cat /data/openclaw.json"
```

### كتابة التكوين عبر SSH

لا يدعم الأمر `fly ssh console -C` إعادة توجيه shell. ولكتابة ملف تكوين:

```bash
# استخدم echo + tee (مرّر من المحلي إلى البعيد)
echo '{"your":"config"}' | fly ssh console -C "tee /data/openclaw.json"

# أو استخدم sftp
fly sftp shell
> put /local/path/config.json /data/openclaw.json
```

**ملاحظة:** قد يفشل `fly sftp` إذا كان الملف موجودًا بالفعل. احذفه أولًا:

```bash
fly ssh console --command "rm /data/openclaw.json"
```

### عدم استمرار الحالة

إذا فقدت ملفات تعريف المصادقة أو حالة القنوات/المزوّد أو الجلسات بعد إعادة التشغيل،
فإن دليل الحالة يكتب إلى نظام ملفات الحاوية.

**الإصلاح:** تأكد من تعيين `OPENCLAW_STATE_DIR=/data` في `fly.toml` ثم أعد النشر.

## التحديثات

```bash
# اسحب آخر التغييرات
git pull

# أعد النشر
fly deploy

# تحقق من السلامة
fly status
fly logs
```

### تحديث أمر الجهاز

إذا احتجت إلى تغيير أمر بدء التشغيل من دون إعادة نشر كاملة:

```bash
# احصل على معرّف الجهاز
fly machines list

# حدّث الأمر
fly machine update <machine-id> --command "node dist/index.js gateway --port 3000 --bind lan" -y

# أو مع زيادة الذاكرة
fly machine update <machine-id> --vm-memory 2048 --command "node dist/index.js gateway --port 3000 --bind lan" -y
```

**ملاحظة:** بعد `fly deploy`، قد يُعاد تعيين أمر الجهاز إلى ما هو موجود في `fly.toml`. وإذا أجريت تغييرات يدوية، فأعد تطبيقها بعد النشر.

## النشر الخاص (مقوّى)

افتراضيًا، يخصص Fly عناوين IP عامة، مما يجعل gateway لديك قابلة للوصول عند `https://your-app.fly.dev`. وهذا مناسب لكنه يعني أن نشرك يمكن اكتشافه بواسطة ماسحات الإنترنت (Shodan وCensys وغيرهما).

للحصول على نشر مقوّى **من دون أي تعرّض عام**، استخدم القالب الخاص.

### متى تستخدم النشر الخاص

- كنت تجري فقط **مكالمات/رسائل صادرة** (من دون webhooks واردة)
- تستخدم **ngrok أو Tailscale** لأي callbacks خاصة بـ webhook
- تصل إلى gateway عبر **SSH أو proxy أو WireGuard** بدلًا من المتصفح
- تريد أن يكون النشر **مخفيًا عن ماسحات الإنترنت**

### الإعداد

استخدم `fly.private.toml` بدلًا من التكوين القياسي:

```bash
# انشر باستخدام التكوين الخاص
fly deploy -c fly.private.toml
```

أو حوّل نشرًا موجودًا:

```bash
# اعرض عناوين IP الحالية
fly ips list -a my-openclaw

# حرّر عناوين IP العامة
fly ips release <public-ipv4> -a my-openclaw
fly ips release <public-ipv6> -a my-openclaw

# انتقل إلى التكوين الخاص حتى لا تعيد عمليات النشر المستقبلية تخصيص عناوين IP عامة
# (احذف [http_service] أو انشر باستخدام القالب الخاص)
fly deploy -c fly.private.toml

# خصص IPv6 خاصًا فقط
fly ips allocate-v6 --private -a my-openclaw
```

بعد ذلك، يجب أن يعرض `fly ips list` عنوان IP من النوع `private` فقط:

```
VERSION  IP                   TYPE             REGION
v6       fdaa:x:x:x:x::x      private          global
```

### الوصول إلى نشر خاص

نظرًا لعدم وجود عنوان URL عام، استخدم إحدى هذه الطرق:

**الخيار 1: proxy محلي (الأبسط)**

```bash
# مرر المنفذ المحلي 3000 إلى التطبيق
fly proxy 3000:3000 -a my-openclaw

# ثم افتح http://localhost:3000 في المتصفح
```

**الخيار 2: WireGuard VPN**

```bash
# أنشئ تكوين WireGuard (مرة واحدة)
fly wireguard create

# استورده إلى عميل WireGuard، ثم استخدم IPv6 الداخلي للوصول
# مثال: http://[fdaa:x:x:x:x::x]:3000
```

**الخيار 3: SSH فقط**

```bash
fly ssh console -a my-openclaw
```

### Webhooks مع النشر الخاص

إذا كنت تحتاج إلى callbacks خاصة بـ webhook ‏(Twilio، وTelnyx، وما إلى ذلك) من دون تعرّض عام:

1. **نفق ngrok** - شغّل ngrok داخل الحاوية أو كـ sidecar
2. **Tailscale Funnel** - اكشف مسارات محددة عبر Tailscale
3. **صادر فقط** - بعض المزوّدين (Twilio) يعملون جيدًا للمكالمات الصادرة من دون webhooks

مثال على تكوين المكالمات الصوتية باستخدام ngrok:

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        enabled: true,
        config: {
          provider: "twilio",
          tunnel: { provider: "ngrok" },
          webhookSecurity: {
            allowedHosts: ["example.ngrok.app"],
          },
        },
      },
    },
  },
}
```

يعمل نفق ngrok داخل الحاوية ويوفر عنوان URL عامًا لـ webhook من دون كشف تطبيق Fly نفسه. اضبط `webhookSecurity.allowedHosts` على اسم مضيف النفق العام حتى يتم قبول رؤوس المضيف المُمرّرة.

### الفوائد الأمنية

| الجانب             | عام         | خاص       |
| ------------------ | ----------- | --------- |
| ماسحات الإنترنت    | قابل للاكتشاف | مخفي     |
| الهجمات المباشرة   | ممكنة        | محظورة    |
| وصول Control UI    | متصفح        | Proxy/VPN |
| تسليم webhook      | مباشر        | عبر نفق   |

## ملاحظات

- يستخدم Fly.io بنية **x86** ‏(وليس ARM)
- Dockerfile متوافق مع كلتا البنيتين
- بالنسبة إلى onboarding الخاص بـ WhatsApp/Telegram، استخدم `fly ssh console`
- تعيش البيانات الدائمة على volume عند `/data`
- يتطلب Signal وجود Java + ‏signal-cli؛ استخدم صورة مخصصة وأبقِ الذاكرة عند 2GB+.

## التكلفة

مع التكوين الموصى به (`shared-cpu-2x`، و2GB RAM):

- نحو 10-15 دولارًا شهريًا حسب الاستخدام
- تتضمن الفئة المجانية بعض الحصة

راجع [تسعير Fly.io](https://fly.io/docs/about/pricing/) للتفاصيل.

## الخطوات التالية

- إعداد قنوات المراسلة: [القنوات](/channels)
- تكوين Gateway: [تكوين Gateway](/gateway/configuration)
- إبقاء OpenClaw محدّثًا: [التحديث](/install/updating)
