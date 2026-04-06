---
read_when:
    - تريد بوابة ضمن حاوية بدلًا من التثبيتات المحلية
    - أنت تتحقق من مسار عمل Docker
summary: إعداد وتهيئة اختيارية لـ OpenClaw بالاعتماد على Docker
title: Docker
x-i18n:
    generated_at: "2026-04-06T03:08:44Z"
    model: gpt-5.4
    provider: openai
    source_hash: d6aa0453340d7683b4954316274ba6dd1aa7c0ce2483e9bd8ae137ff4efd4c3c
    source_path: install/docker.md
    workflow: 15
---

# Docker (اختياري)

Docker **اختياري**. استخدمه فقط إذا كنت تريد بوابة ضمن حاوية أو للتحقق من مسار عمل Docker.

## هل Docker مناسب لي؟

- **نعم**: تريد بيئة بوابة معزولة وقابلة للتخلص منها أو تريد تشغيل OpenClaw على مضيف من دون تثبيتات محلية.
- **لا**: أنت تشغّل OpenClaw على جهازك وتريد فقط أسرع حلقة تطوير. استخدم مسار التثبيت العادي بدلًا من ذلك.
- **ملاحظة حول Sandbox**: يستخدم عزل الوكيل Docker أيضًا، لكنه **لا** يتطلب تشغيل البوابة كاملة داخل Docker. راجع [Sandboxing](/ar/gateway/sandboxing).

## المتطلبات المسبقة

- Docker Desktop (أو Docker Engine) + Docker Compose v2
- ذاكرة RAM لا تقل عن 2 GB لبناء الصورة (قد تتعرض `pnpm install` للقتل بسبب نفاد الذاكرة على المضيفات ذات 1 GB مع الخروج 137)
- مساحة قرص كافية للصور والسجلات
- إذا كنت تشغّله على VPS/مضيف عام، فراجع
  [تقوية الأمان للتعرّض للشبكة](/ar/gateway/security)،
  وخاصة سياسة جدار الحماية `DOCKER-USER` في Docker.

## البوابة ضمن حاوية

<Steps>
  <Step title="ابنِ الصورة">
    من جذر المستودع، شغّل برنامج الإعداد النصي:

    ```bash
    ./scripts/docker/setup.sh
    ```

    سيبني هذا صورة البوابة محليًا. لاستخدام صورة مبنية مسبقًا بدلًا من ذلك:

    ```bash
    export OPENCLAW_IMAGE="ghcr.io/openclaw/openclaw:latest"
    ./scripts/docker/setup.sh
    ```

    تُنشر الصور المبنية مسبقًا في
    [GitHub Container Registry](https://github.com/openclaw/openclaw/pkgs/container/openclaw).
    الوسوم الشائعة: `main` و`latest` و`<version>` (مثل `2026.2.26`).

  </Step>

  <Step title="أكمل التهيئة الأولية">
    يشغّل برنامج الإعداد النصي التهيئة الأولية تلقائيًا. وسيقوم بما يلي:

    - طلب مفاتيح API الخاصة بالمزوّد
    - توليد token للبوابة وكتابته إلى `.env`
    - تشغيل البوابة عبر Docker Compose

    أثناء الإعداد، تمر التهيئة الأولية قبل البدء وعمليات كتابة config عبر
    `openclaw-gateway` مباشرة. أما `openclaw-cli` فهو للأوامر التي تشغّلها بعد
    أن تكون حاوية البوابة موجودة بالفعل.

  </Step>

  <Step title="افتح Control UI">
    افتح `http://127.0.0.1:18789/` في متصفحك والصق السر المشترك المهيأ
    في Settings. يكتب برنامج الإعداد النصي token إلى `.env` افتراضيًا؛
    وإذا بدّلت config الحاوية إلى مصادقة بكلمة مرور، فاستخدم كلمة
    المرور تلك بدلًا من ذلك.

    هل تحتاج إلى URL مرة أخرى؟

    ```bash
    docker compose run --rm openclaw-cli dashboard --no-open
    ```

  </Step>

  <Step title="هيّئ القنوات (اختياري)">
    استخدم حاوية CLI لإضافة قنوات المراسلة:

    ```bash
    # WhatsApp (QR)
    docker compose run --rm openclaw-cli channels login

    # Telegram
    docker compose run --rm openclaw-cli channels add --channel telegram --token "<token>"

    # Discord
    docker compose run --rm openclaw-cli channels add --channel discord --token "<token>"
    ```

    المستندات: [WhatsApp](/ar/channels/whatsapp)، [Telegram](/ar/channels/telegram)، [Discord](/ar/channels/discord)

  </Step>
</Steps>

### المسار اليدوي

إذا كنت تفضّل تشغيل كل خطوة بنفسك بدلًا من استخدام برنامج الإعداد النصي:

```bash
docker build -t openclaw:local -f Dockerfile .
docker compose run --rm --no-deps --entrypoint node openclaw-gateway \
  dist/index.js onboard --mode local --no-install-daemon
docker compose run --rm --no-deps --entrypoint node openclaw-gateway \
  dist/index.js config set --batch-json '[{"path":"gateway.mode","value":"local"},{"path":"gateway.bind","value":"lan"},{"path":"gateway.controlUi.allowedOrigins","value":["http://localhost:18789","http://127.0.0.1:18789"]}]'
docker compose up -d openclaw-gateway
```

<Note>
شغّل `docker compose` من جذر المستودع. إذا فعّلت `OPENCLAW_EXTRA_MOUNTS`
أو `OPENCLAW_HOME_VOLUME`، فسيكتب برنامج الإعداد النصي `docker-compose.extra.yml`؛
ضمّنه باستخدام `-f docker-compose.yml -f docker-compose.extra.yml`.
</Note>

<Note>
لأن `openclaw-cli` يشارك مساحة اسم الشبكة الخاصة بـ `openclaw-gateway`، فهو
أداة لما بعد البدء. وقبل `docker compose up -d openclaw-gateway`، شغّل التهيئة الأولية
وكتابات config وقت الإعداد عبر `openclaw-gateway` باستخدام
`--no-deps --entrypoint node`.
</Note>

### متغيرات البيئة

يقبل برنامج الإعداد النصي متغيرات البيئة الاختيارية التالية:

| المتغير                       | الغرض                                                            |
| ---------------------------- | ---------------------------------------------------------------- |
| `OPENCLAW_IMAGE`             | استخدام صورة بعيدة بدلًا من البناء محليًا                       |
| `OPENCLAW_DOCKER_APT_PACKAGES` | تثبيت حزم apt إضافية أثناء البناء (أسماء مفصولة بمسافات)      |
| `OPENCLAW_EXTENSIONS`        | التثبيت المسبق لاعتماديات الإضافات وقت البناء (أسماء مفصولة بمسافات) |
| `OPENCLAW_EXTRA_MOUNTS`      | عمليات bind mount إضافية من المضيف (بصيغة `source:target[:opts]` مفصولة بفواصل) |
| `OPENCLAW_HOME_VOLUME`       | جعل `/home/node` دائمًا في Docker volume مُسمّى                 |
| `OPENCLAW_SANDBOX`           | الاشتراك في تهيئة sandbox (`1` أو `true` أو `yes` أو `on`)     |
| `OPENCLAW_DOCKER_SOCKET`     | تجاوز مسار Docker socket                                        |

### فحوصات السلامة

نقاط نهاية فحص الحاوية (لا تتطلب مصادقة):

```bash
curl -fsS http://127.0.0.1:18789/healthz   # التحقق من الحيوية
curl -fsS http://127.0.0.1:18789/readyz     # التحقق من الجاهزية
```

تتضمن صورة Docker فحص `HEALTHCHECK` مضمّنًا يطلب `/healthz`.
إذا استمرت الفحوصات في الفشل، يضع Docker علامة `unhealthy` على الحاوية،
ويمكن لأنظمة التنسيق إعادة تشغيلها أو استبدالها.

لقطة سلامة عميقة موثّقة:

```bash
docker compose exec openclaw-gateway node dist/index.js health --token "$OPENCLAW_GATEWAY_TOKEN"
```

### LAN مقابل loopback

يضبط `scripts/docker/setup.sh` القيمة الافتراضية `OPENCLAW_GATEWAY_BIND=lan` بحيث يعمل وصول المضيف إلى
`http://127.0.0.1:18789` مع نشر منفذ Docker.

- `lan` (الافتراضي): يمكن لمتصفح المضيف وCLI على المضيف الوصول إلى منفذ البوابة المنشور.
- `loopback`: لا يمكن إلا للعمليات داخل مساحة اسم شبكة الحاوية الوصول
  مباشرة إلى البوابة.

<Note>
استخدم قيم وضع الربط في `gateway.bind` ‏(`lan` / `loopback` / `custom` /
`tailnet` / `auto`) وليس الأسماء البديلة للمضيف مثل `0.0.0.0` أو `127.0.0.1`.
</Note>

### التخزين والاستمرارية

يقوم Docker Compose بعمل bind mount لكل من `OPENCLAW_CONFIG_DIR` إلى `/home/node/.openclaw` و
`OPENCLAW_WORKSPACE_DIR` إلى `/home/node/.openclaw/workspace`، بحيث تظل هذه المسارات
موجودة بعد استبدال الحاوية.

ودليل config المركّب هذا هو المكان الذي يحتفظ فيه OpenClaw بما يلي:

- `openclaw.json` لإعدادات السلوك
- `agents/<agentId>/agent/auth-profiles.json` لمصادقة OAuth/API-key المخزنة الخاصة بالمزوّد
- `.env` للأسرار الخاصة بوقت التشغيل المعتمدة على متغيرات البيئة مثل `OPENCLAW_GATEWAY_TOKEN`

للحصول على تفاصيل الاستمرارية الكاملة في عمليات نشر VM، راجع
[Docker VM Runtime - أين تستمر البيانات](/ar/install/docker-vm-runtime#what-persists-where).

**نقاط زيادة استخدام القرص:** راقب `media/` وملفات JSONL الخاصة بالجلسات و`cron/runs/*.jsonl`،
وسجلات الملفات الدوّارة تحت `/tmp/openclaw/`.

### مساعدات shell (اختياري)

لتسهيل إدارة Docker اليومية، ثبّت `ClawDock`:

```bash
mkdir -p ~/.clawdock && curl -sL https://raw.githubusercontent.com/openclaw/openclaw/main/scripts/clawdock/clawdock-helpers.sh -o ~/.clawdock/clawdock-helpers.sh
echo 'source ~/.clawdock/clawdock-helpers.sh' >> ~/.zshrc && source ~/.zshrc
```

إذا كنت قد ثبّت ClawDock من المسار الخام الأقدم `scripts/shell-helpers/clawdock-helpers.sh`، فأعد تشغيل أمر التثبيت أعلاه حتى يتتبّع ملف المساعد المحلي الموقع الجديد.

ثم استخدم `clawdock-start` و`clawdock-stop` و`clawdock-dashboard` وما إلى ذلك. شغّل
`clawdock-help` لجميع الأوامر.
راجع [ClawDock](/ar/install/clawdock) للحصول على دليل المساعد الكامل.

<AccordionGroup>
  <Accordion title="فعّل sandbox للوكيل لبوابة Docker">
    ```bash
    export OPENCLAW_SANDBOX=1
    ./scripts/docker/setup.sh
    ```

    مسار socket مخصص (مثل Docker بدون root):

    ```bash
    export OPENCLAW_SANDBOX=1
    export OPENCLAW_DOCKER_SOCKET=/run/user/1000/docker.sock
    ./scripts/docker/setup.sh
    ```

    لا يركّب البرنامج النصي `docker.sock` إلا بعد نجاح المتطلبات المسبقة لـ sandbox. وإذا
    تعذر إكمال إعداد sandbox، يعيد البرنامج النصي تعيين `agents.defaults.sandbox.mode`
    إلى `off`.

  </Accordion>

  <Accordion title="الأتمتة / CI (غير تفاعلي)">
    عطّل تخصيص pseudo-TTY في Compose باستخدام `-T`:

    ```bash
    docker compose run -T --rm openclaw-cli gateway probe
    docker compose run -T --rm openclaw-cli devices list --json
    ```

  </Accordion>

  <Accordion title="ملاحظة أمنية حول الشبكة المشتركة">
    يستخدم `openclaw-cli` القيمة `network_mode: "service:openclaw-gateway"` بحيث يمكن
    لأوامر CLI الوصول إلى البوابة عبر `127.0.0.1`. تعامل مع هذا على أنه
    حد ثقة مشترك. يقوم config الخاص بـ compose بإزالة `NET_RAW` و`NET_ADMIN` ويفعّل
    `no-new-privileges` على `openclaw-cli`.
  </Accordion>

  <Accordion title="الأذونات وEACCES">
    تعمل الصورة بصفة `node` ‏(uid 1000). إذا رأيت أخطاء أذونات على
    `/home/node/.openclaw`، فتأكد من أن عمليات bind mount على مضيفك مملوكة للمعرّف uid 1000:

    ```bash
    sudo chown -R 1000:1000 /path/to/openclaw-config /path/to/openclaw-workspace
    ```

  </Accordion>

  <Accordion title="إعادات بناء أسرع">
    رتّب Dockerfile بحيث تُخزَّن طبقات الاعتماديات مؤقتًا. وهذا يتجنب إعادة تشغيل
    `pnpm install` ما لم تتغير ملفات القفل:

    ```dockerfile
    FROM node:24-bookworm
    RUN curl -fsSL https://bun.sh/install | bash
    ENV PATH="/root/.bun/bin:${PATH}"
    RUN corepack enable
    WORKDIR /app
    COPY package.json pnpm-lock.yaml pnpm-workspace.yaml .npmrc ./
    COPY ui/package.json ./ui/package.json
    COPY scripts ./scripts
    RUN pnpm install --frozen-lockfile
    COPY . .
    RUN pnpm build
    RUN pnpm ui:install
    RUN pnpm ui:build
    ENV NODE_ENV=production
    CMD ["node","dist/index.js"]
    ```

  </Accordion>

  <Accordion title="خيارات الحاويات للمستخدمين المتقدمين">
    الصورة الافتراضية تركز على الأمان أولًا وتعمل بصفة `node` غير الجذرية. للحصول على
    حاوية أكثر اكتمالًا في الميزات:

    1. **اجعل `/home/node` دائمًا**: `export OPENCLAW_HOME_VOLUME="openclaw_home"`
    2. **ضمّن اعتماديات النظام**: `export OPENCLAW_DOCKER_APT_PACKAGES="git curl jq"`
    3. **ثبّت متصفحات Playwright**:
       ```bash
       docker compose run --rm openclaw-cli \
         node /app/node_modules/playwright-core/cli.js install chromium
       ```
    4. **اجعل تنزيلات المتصفح دائمة**: اضبط
       `PLAYWRIGHT_BROWSERS_PATH=/home/node/.cache/ms-playwright` واستخدم
       `OPENCLAW_HOME_VOLUME` أو `OPENCLAW_EXTRA_MOUNTS`.

  </Accordion>

  <Accordion title="OpenAI Codex OAuth (Docker دون واجهة)">
    إذا اخترت OpenAI Codex OAuth في المعالج، فسيفتح URL في المتصفح. في
    إعدادات Docker أو الإعدادات دون واجهة، انسخ URL الكامل لإعادة التوجيه الذي تصل إليه والصقه
    مرة أخرى في المعالج لإكمال المصادقة.
  </Accordion>

  <Accordion title="بيانات الصورة الأساسية الوصفية">
    تستخدم صورة Docker الرئيسية `node:24-bookworm` وتنشر تعليقات توضيحية للصورة الأساسية وفق OCI
    بما في ذلك `org.opencontainers.image.base.name`،
    و`org.opencontainers.image.source`، وغيرها. راجع
    [تعليقات صور OCI التوضيحية](https://github.com/opencontainers/image-spec/blob/main/annotations.md).
  </Accordion>
</AccordionGroup>

### هل تعمل على VPS؟

راجع [Hetzner (Docker VPS)](/ar/install/hetzner) و
[Docker VM Runtime](/ar/install/docker-vm-runtime) لخطوات النشر المشتركة على VM
بما في ذلك تضمين الثنائيات، والاستمرارية، والتحديثات.

## Agent Sandbox

عند تمكين `agents.defaults.sandbox`، تشغّل البوابة تنفيذ أدوات الوكيل
(shell وقراءة/كتابة الملفات وما إلى ذلك) داخل حاويات Docker معزولة بينما
تبقى البوابة نفسها على المضيف. وهذا يمنحك حاجزًا صلبًا حول جلسات الوكيل غير الموثوقة أو
متعددة المستأجرين من دون وضع البوابة كاملة داخل حاوية.

يمكن أن يكون نطاق sandbox لكل وكيل على حدة (الافتراضي)، أو لكل جلسة، أو مشتركًا. ويحصل كل نطاق
على مساحة عمله الخاصة المركّبة عند `/workspace`. ويمكنك أيضًا تهيئة
سياسات السماح/المنع للأدوات، وعزل الشبكة، وحدود الموارد، وحاويات
المتصفح.

للحصول على الإعداد الكامل، والصور، والملاحظات الأمنية، وملفات تعريف الوكلاء المتعددين، راجع:

- [Sandboxing](/ar/gateway/sandboxing) -- المرجع الكامل لـ sandbox
- [OpenShell](/ar/gateway/openshell) -- وصول shell تفاعلي إلى حاويات sandbox
- [Multi-Agent Sandbox and Tools](/ar/tools/multi-agent-sandbox-tools) -- تجاوزات لكل وكيل

### تفعيل سريع

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main", // off | non-main | all
        scope: "agent", // session | agent | shared
      },
    },
  },
}
```

ابنِ صورة sandbox الافتراضية:

```bash
scripts/sandbox-setup.sh
```

## استكشاف الأخطاء وإصلاحها

<AccordionGroup>
  <Accordion title="الصورة مفقودة أو حاوية sandbox لا تبدأ">
    ابنِ صورة sandbox باستخدام
    [`scripts/sandbox-setup.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/sandbox-setup.sh)
    أو اضبط `agents.defaults.sandbox.docker.image` على صورتك المخصصة.
    تُنشأ الحاويات تلقائيًا لكل جلسة عند الطلب.
  </Accordion>

  <Accordion title="أخطاء أذونات داخل sandbox">
    اضبط `docker.user` على UID:GID يطابق ملكية مساحة العمل المركّبة لديك،
    أو غيّر ملكية مجلد مساحة العمل.
  </Accordion>

  <Accordion title="أدوات مخصصة غير موجودة داخل sandbox">
    يشغّل OpenClaw الأوامر باستخدام `sh -lc` (shell تسجيل دخول)، وهو ما يحمّل
    `/etc/profile` وقد يعيد تعيين PATH. اضبط `docker.env.PATH` لإضافة
    مسارات أدواتك المخصصة في المقدمة، أو أضف برنامجًا نصيًا تحت `/etc/profile.d/` في Dockerfile.
  </Accordion>

  <Accordion title="تم القتل بسبب نفاد الذاكرة أثناء بناء الصورة (الخروج 137)">
    تحتاج VM إلى ذاكرة RAM لا تقل عن 2 GB. استخدم فئة جهاز أكبر وأعد المحاولة.
  </Accordion>

  <Accordion title="غير مخوّل أو يتطلب pairing في Control UI">
    اجلب رابط dashboard جديدًا ووافق على جهاز المتصفح:

    ```bash
    docker compose run --rm openclaw-cli dashboard --no-open
    docker compose run --rm openclaw-cli devices list
    docker compose run --rm openclaw-cli devices approve <requestId>
    ```

    مزيد من التفاصيل: [Dashboard](/web/dashboard)، [Devices](/cli/devices).

  </Accordion>

  <Accordion title="هدف البوابة يعرض ws://172.x.x.x أو أخطاء pairing من Docker CLI">
    أعد ضبط وضع البوابة والربط:

    ```bash
    docker compose run --rm openclaw-cli config set --batch-json '[{"path":"gateway.mode","value":"local"},{"path":"gateway.bind","value":"lan"}]'
    docker compose run --rm openclaw-cli devices list --url ws://127.0.0.1:18789
    ```

  </Accordion>
</AccordionGroup>

## ذو صلة

- [نظرة عامة على التثبيت](/ar/install) — جميع طرق التثبيت
- [Podman](/ar/install/podman) — بديل Podman لـ Docker
- [ClawDock](/ar/install/clawdock) — إعداد Docker Compose من المجتمع
- [التحديث](/ar/install/updating) — إبقاء OpenClaw محدثًا
- [Configuration](/ar/gateway/configuration) — إعدادات البوابة بعد التثبيت
