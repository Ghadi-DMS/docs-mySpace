---
read_when:
    - تريد Gateway داخل حاوية بدلًا من التثبيتات المحلية
    - أنت تتحقق من تدفق Docker
summary: إعداد وتهيئة OpenClaw الاختياريين باستخدام Docker
title: Docker
x-i18n:
    generated_at: "2026-04-05T12:47:05Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4628362d52597f85e72c214efe96b2923c7a59a8592b3044dc8c230318c515b8
    source_path: install/docker.md
    workflow: 15
---

# Docker (اختياري)

Docker **اختياري**. استخدمه فقط إذا كنت تريد Gateway داخل حاوية أو للتحقق من تدفق Docker.

## هل Docker مناسب لي؟

- **نعم**: تريد بيئة Gateway معزولة وقابلة للاستبدال أو تريد تشغيل OpenClaw على مضيف من دون تثبيتات محلية.
- **لا**: أنت تشغّل التطبيق على جهازك الخاص وتريد فقط أسرع حلقة تطوير. استخدم مسار التثبيت العادي بدلًا من ذلك.
- **ملاحظة حول sandboxing**: يستخدم عزل الوكيل Docker أيضًا، لكنه **لا** يتطلب تشغيل Gateway بالكامل داخل Docker. راجع [Sandboxing](/gateway/sandboxing).

## المتطلبات الأساسية

- Docker Desktop (أو Docker Engine) + Docker Compose v2
- ما لا يقل عن 2 GB RAM لبناء الصورة (`pnpm install` قد يُنهى بسبب OOM على المضيفات ذات 1 GB مع الخروج 137)
- مساحة قرص كافية للصور والسجلات
- إذا كنت تشغّل التطبيق على VPS/مضيف عام، فراجع
  [تقوية الأمان للتعرض الشبكي](/gateway/security)،
  وخاصة سياسة جدار الحماية `DOCKER-USER` في Docker.

## Gateway داخل حاوية

<Steps>
  <Step title="بناء الصورة">
    من جذر المستودع، شغّل نص الإعداد:

    ```bash
    ./scripts/docker/setup.sh
    ```

    هذا يبني صورة gateway محليًا. لاستخدام صورة مبنية مسبقًا بدلًا من ذلك:

    ```bash
    export OPENCLAW_IMAGE="ghcr.io/openclaw/openclaw:latest"
    ./scripts/docker/setup.sh
    ```

    تُنشر الصور المبنية مسبقًا في
    [GitHub Container Registry](https://github.com/openclaw/openclaw/pkgs/container/openclaw).
    الوسوم الشائعة: `main`, `latest`, `<version>` (مثل `2026.2.26`).

  </Step>

  <Step title="إكمال التهيئة الأولية">
    يشغّل نص الإعداد التهيئة الأولية تلقائيًا. وسيقوم بما يلي:

    - طلب مفاتيح API الخاصة بالمزوّد
    - إنشاء رمز gateway وكتابته إلى `.env`
    - بدء gateway عبر Docker Compose

    أثناء الإعداد، تعمل التهيئة الأولية قبل البدء وعمليات كتابة التكوين من خلال
    `openclaw-gateway` مباشرةً. أمّا `openclaw-cli` فهي للأوامر التي تشغّلها بعد
    أن تصبح حاوية gateway موجودة بالفعل.

  </Step>

  <Step title="فتح Control UI">
    افتح `http://127.0.0.1:18789/` في متصفحك والصق
    السر المشترك المكوَّن في Settings. يكتب نص الإعداد رمزًا إلى `.env`
    افتراضيًا؛ وإذا بدّلت تكوين الحاوية إلى مصادقة كلمة المرور، فاستخدم
    كلمة المرور تلك بدلًا من ذلك.

    هل تحتاج إلى عنوان URL مرة أخرى؟

    ```bash
    docker compose run --rm openclaw-cli dashboard --no-open
    ```

  </Step>

  <Step title="إعداد القنوات (اختياري)">
    استخدم حاوية CLI لإضافة قنوات المراسلة:

    ```bash
    # WhatsApp (QR)
    docker compose run --rm openclaw-cli channels login

    # Telegram
    docker compose run --rm openclaw-cli channels add --channel telegram --token "<token>"

    # Discord
    docker compose run --rm openclaw-cli channels add --channel discord --token "<token>"
    ```

    الوثائق: [WhatsApp](/channels/whatsapp)، [Telegram](/channels/telegram)، [Discord](/channels/discord)

  </Step>
</Steps>

### التدفق اليدوي

إذا كنت تفضّل تشغيل كل خطوة بنفسك بدلًا من استخدام نص الإعداد:

```bash
docker build -t openclaw:local -f Dockerfile .
docker compose run --rm --no-deps --entrypoint node openclaw-gateway \
  dist/index.js onboard --mode local --no-install-daemon
docker compose run --rm --no-deps --entrypoint node openclaw-gateway \
  dist/index.js config set gateway.mode local
docker compose run --rm --no-deps --entrypoint node openclaw-gateway \
  dist/index.js config set gateway.bind lan
docker compose run --rm --no-deps --entrypoint node openclaw-gateway \
  dist/index.js config set gateway.controlUi.allowedOrigins \
  '["http://localhost:18789","http://127.0.0.1:18789"]' --strict-json
docker compose up -d openclaw-gateway
```

<Note>
شغّل `docker compose` من جذر المستودع. إذا فعّلت `OPENCLAW_EXTRA_MOUNTS`
أو `OPENCLAW_HOME_VOLUME`، فسيكتب نص الإعداد الملف `docker-compose.extra.yml`؛
ضمّنه مع `-f docker-compose.yml -f docker-compose.extra.yml`.
</Note>

<Note>
نظرًا لأن `openclaw-cli` تشارك مساحة أسماء الشبكة الخاصة بـ `openclaw-gateway`، فهي
أداة لما بعد البدء. قبل `docker compose up -d openclaw-gateway`، شغّل التهيئة الأولية
وعمليات كتابة التكوين وقت الإعداد من خلال `openclaw-gateway` باستخدام
`--no-deps --entrypoint node`.
</Note>

### متغيرات البيئة

يقبل نص الإعداد متغيرات البيئة الاختيارية التالية:

| المتغير                       | الغرض                                                            |
| ----------------------------- | ---------------------------------------------------------------- |
| `OPENCLAW_IMAGE`              | استخدام صورة بعيدة بدلًا من البناء محليًا                       |
| `OPENCLAW_DOCKER_APT_PACKAGES`| تثبيت حزم apt إضافية أثناء البناء (أسماء مفصولة بمسافات)        |
| `OPENCLAW_EXTENSIONS`         | تثبيت تبعيات extension مسبقًا وقت البناء (أسماء مفصولة بمسافات) |
| `OPENCLAW_EXTRA_MOUNTS`       | bind mounts إضافية من المضيف (مفصولة بفواصل `source:target[:opts]`) |
| `OPENCLAW_HOME_VOLUME`        | حفظ `/home/node` في Docker volume مسماة                          |
| `OPENCLAW_SANDBOX`            | الاشتراك في bootstrap الخاص بـ sandbox (`1`, `true`, `yes`, `on`) |
| `OPENCLAW_DOCKER_SOCKET`      | تجاوز مسار Docker socket                                         |

### فحوصات الصحة

نقاط نهاية probe الخاصة بالحاوية (لا تتطلب مصادقة):

```bash
curl -fsS http://127.0.0.1:18789/healthz   # liveness
curl -fsS http://127.0.0.1:18789/readyz     # readiness
```

تتضمن صورة Docker ‏`HEALTHCHECK` مضمّنًا يجري ping إلى `/healthz`.
إذا استمرت الفحوصات في الفشل، يضع Docker علامة `unhealthy` على الحاوية،
ويمكن لأنظمة orchestration إعادة تشغيلها أو استبدالها.

لقطة صحة عميقة مع مصادقة:

```bash
docker compose exec openclaw-gateway node dist/index.js health --token "$OPENCLAW_GATEWAY_TOKEN"
```

### LAN مقابل loopback

يضبط `scripts/docker/setup.sh` القيمة الافتراضية `OPENCLAW_GATEWAY_BIND=lan` بحيث يعمل وصول المضيف إلى
`http://127.0.0.1:18789` مع نشر منافذ Docker.

- `lan` (الافتراضي): يمكن لمتصفح المضيف وCLI على المضيف الوصول إلى منفذ gateway المنشور.
- `loopback`: لا يمكن إلا للعمليات داخل مساحة أسماء شبكة الحاوية الوصول إلى
  gateway مباشرة.

<Note>
استخدم قيم وضع الربط في `gateway.bind` (`lan` / `loopback` / `custom` /
`tailnet` / `auto`)، وليس الأسماء المستعارة للمضيف مثل `0.0.0.0` أو `127.0.0.1`.
</Note>

### التخزين والاستمرارية

يقوم Docker Compose بعمل bind-mount لـ `OPENCLAW_CONFIG_DIR` إلى `/home/node/.openclaw` و
`OPENCLAW_WORKSPACE_DIR` إلى `/home/node/.openclaw/workspace`، لذا فإن هذه المسارات
تبقى بعد استبدال الحاوية.

ودليل التكوين المركّب هذا هو المكان الذي يحتفظ فيه OpenClaw بما يلي:

- `openclaw.json` لتكوين السلوك
- `agents/<agentId>/agent/auth-profiles.json` للمصادقة المخزنة للمزوّدين عبر OAuth/API-key
- `.env` للأسرار وقت التشغيل المدعومة بـ env مثل `OPENCLAW_GATEWAY_TOKEN`

للحصول على تفاصيل الاستمرارية الكاملة في عمليات النشر على VM، راجع
[Docker VM Runtime - What persists where](/install/docker-vm-runtime#what-persists-where).

**النقاط الساخنة لنمو القرص:** راقب `media/` وملفات JSONL الخاصة بالجلسات و`cron/runs/*.jsonl`،
وسجلات الملفات الدوّارة تحت `/tmp/openclaw/`.

### مساعدات shell (اختياري)

لتسهيل الإدارة اليومية لـ Docker، ثبّت `ClawDock`:

```bash
mkdir -p ~/.clawdock && curl -sL https://raw.githubusercontent.com/openclaw/openclaw/main/scripts/clawdock/clawdock-helpers.sh -o ~/.clawdock/clawdock-helpers.sh
echo 'source ~/.clawdock/clawdock-helpers.sh' >> ~/.zshrc && source ~/.zshrc
```

إذا كنت قد ثبّتت ClawDock من المسار الخام الأقدم `scripts/shell-helpers/clawdock-helpers.sh`، فأعد تشغيل أمر التثبيت أعلاه حتى يتتبع ملف المساعد المحلي الموقع الجديد.

بعد ذلك استخدم `clawdock-start` و`clawdock-stop` و`clawdock-dashboard` وغيرها. شغّل
`clawdock-help` لجميع الأوامر.
راجع [ClawDock](/install/clawdock) للحصول على دليل المساعد الكامل.

<AccordionGroup>
  <Accordion title="تمكين agent sandbox لبوابة Docker">
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

    يقوم النص بتركيب `docker.sock` فقط بعد نجاح متطلبات sandbox الأساسية. وإذا
    تعذّر إكمال إعداد sandbox، يعيد النص ضبط `agents.defaults.sandbox.mode`
    إلى `off`.

  </Accordion>

  <Accordion title="الأتمتة / CI (غير تفاعلي)">
    عطّل تخصيص Compose pseudo-TTY باستخدام `-T`:

    ```bash
    docker compose run -T --rm openclaw-cli gateway probe
    docker compose run -T --rm openclaw-cli devices list --json
    ```

  </Accordion>

  <Accordion title="ملاحظة أمان حول الشبكة المشتركة">
    تستخدم `openclaw-cli` القيمة `network_mode: "service:openclaw-gateway"` حتى تتمكن
    أوامر CLI من الوصول إلى gateway عبر `127.0.0.1`. تعامل مع هذا
    كحد ثقة مشترك. يسقط تكوين compose الصلاحيتين `NET_RAW`/`NET_ADMIN` ويفعل
    `no-new-privileges` على `openclaw-cli`.
  </Accordion>

  <Accordion title="الأذونات وEACCES">
    تعمل الصورة كمستخدم `node` ‏(uid 1000). إذا رأيت أخطاء أذونات على
    `/home/node/.openclaw`، فتأكد من أن bind mounts على المضيف مملوكة لـ uid 1000:

    ```bash
    sudo chown -R 1000:1000 /path/to/openclaw-config /path/to/openclaw-workspace
    ```

  </Accordion>

  <Accordion title="إعادة بناء أسرع">
    رتّب Dockerfile بحيث تكون طبقات التبعيات مخزنة مؤقتًا. هذا يتجنب إعادة تشغيل
    `pnpm install` ما لم تتغير lockfiles:

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

  <Accordion title="خيارات حاوية للمستخدمين المتقدمين">
    الصورة الافتراضية تركّز على الأمان وتعمل كمستخدم `node` غير root. وللحصول على
    حاوية أكثر اكتمالًا في الميزات:

    1. **الاحتفاظ بـ `/home/node`**: ‏`export OPENCLAW_HOME_VOLUME="openclaw_home"`
    2. **إدراج تبعيات النظام ضمن الصورة**: ‏`export OPENCLAW_DOCKER_APT_PACKAGES="git curl jq"`
    3. **تثبيت متصفحات Playwright**:
       ```bash
       docker compose run --rm openclaw-cli \
         node /app/node_modules/playwright-core/cli.js install chromium
       ```
    4. **الاحتفاظ بتنزيلات المتصفح**: اضبط
       `PLAYWRIGHT_BROWSERS_PATH=/home/node/.cache/ms-playwright` واستخدم
       `OPENCLAW_HOME_VOLUME` أو `OPENCLAW_EXTRA_MOUNTS`.

  </Accordion>

  <Accordion title="OpenAI Codex OAuth (Docker بدون واجهة)">
    إذا اخترت OpenAI Codex OAuth في المعالج، فسيفتح عنوان URL في المتصفح. في
    إعدادات Docker أو الإعدادات بدون واجهة، انسخ عنوان URL الكامل لإعادة التوجيه الذي تصل إليه والصقه
    مرة أخرى في المعالج لإكمال المصادقة.
  </Accordion>

  <Accordion title="بيانات تعريف الصورة الأساسية">
    تستخدم صورة Docker الرئيسية `node:24-bookworm` وتنشر تعليقات توضيحية
    خاصة بصورة OCI الأساسية بما في ذلك `org.opencontainers.image.base.name`,
    و`org.opencontainers.image.source`، وغيرها. راجع
    [OCI image annotations](https://github.com/opencontainers/image-spec/blob/main/annotations.md).
  </Accordion>
</AccordionGroup>

### التشغيل على VPS؟

راجع [Hetzner (Docker VPS)](/install/hetzner) و
[Docker VM Runtime](/install/docker-vm-runtime) لخطوات نشر VM المشتركة
بما في ذلك إدراج binaries في الصورة، والاستمرارية، والتحديثات.

## Agent Sandbox

عندما تكون `agents.defaults.sandbox` مفعلة، تشغّل gateway تنفيذ أدوات الوكيل
(shell وقراءة/كتابة الملفات، وما إلى ذلك) داخل حاويات Docker معزولة بينما تبقى
gateway نفسها على المضيف. يوفّر لك هذا حاجزًا صلبًا حول جلسات الوكيل غير الموثوقة أو متعددة المستأجرين من دون وضع gateway بالكامل داخل حاوية.

يمكن أن يكون نطاق sandbox لكل وكيل (الافتراضي)، أو لكل جلسة، أو مشتركًا. ويحصل كل نطاق
على مساحة العمل الخاصة به مركّبة عند `/workspace`. ويمكنك أيضًا تكوين
سياسات السماح/المنع للأدوات، وعزل الشبكة، وحدود الموارد، وحاويات المتصفح.

للاطلاع على التكوين الكامل، والصور، والملاحظات الأمنية، وملفات الوكلاء المتعددة، راجع:

- [Sandboxing](/gateway/sandboxing) -- المرجع الكامل لـ sandbox
- [OpenShell](/gateway/openshell) -- وصول shell تفاعلي إلى حاويات sandbox
- [Multi-Agent Sandbox and Tools](/tools/multi-agent-sandbox-tools) -- تجاوزات لكل وكيل

### تمكين سريع

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

  <Accordion title="أخطاء الأذونات في sandbox">
    اضبط `docker.user` على UID:GID يطابق ملكية مساحة العمل المركّبة لديك،
    أو غيّر ملكية مجلد مساحة العمل.
  </Accordion>

  <Accordion title="الأدوات المخصصة غير موجودة في sandbox">
    يشغّل OpenClaw الأوامر باستخدام `sh -lc` (login shell)، والتي تستورد
    `/etc/profile` وقد تعيد ضبط PATH. اضبط `docker.env.PATH` لإضافة
    مسارات أدواتك المخصصة في البداية، أو أضف script تحت `/etc/profile.d/` في Dockerfile لديك.
  </Accordion>

  <Accordion title="تم إنهاء البناء بسبب OOM (الخروج 137)">
    تحتاج VM إلى 2 GB RAM على الأقل. استخدم فئة جهاز أكبر ثم أعد المحاولة.
  </Accordion>

  <Accordion title="Unauthorized أو pairing required في Control UI">
    اجلب رابط dashboard جديدًا ووافق على جهاز المتصفح:

    ```bash
    docker compose run --rm openclaw-cli dashboard --no-open
    docker compose run --rm openclaw-cli devices list
    docker compose run --rm openclaw-cli devices approve <requestId>
    ```

    تفاصيل أكثر: [Dashboard](/web/dashboard)، [Devices](/cli/devices).

  </Accordion>

  <Accordion title="يعرض هدف Gateway القيمة ws://172.x.x.x أو تظهر أخطاء pairing من Docker CLI">
    أعد ضبط وضع gateway والربط:

    ```bash
    docker compose run --rm openclaw-cli config set gateway.mode local
    docker compose run --rm openclaw-cli config set gateway.bind lan
    docker compose run --rm openclaw-cli devices list --url ws://127.0.0.1:18789
    ```

  </Accordion>
</AccordionGroup>

## ذو صلة

- [Install Overview](/install) — جميع طرق التثبيت
- [Podman](/install/podman) — بديل Podman لـ Docker
- [ClawDock](/install/clawdock) — إعداد Docker Compose مجتمعي
- [Updating](/install/updating) — إبقاء OpenClaw محدثًا
- [Configuration](/gateway/configuration) — تكوين gateway بعد التثبيت
