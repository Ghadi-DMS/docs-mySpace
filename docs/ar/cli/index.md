---
read_when:
    - إضافة أو تعديل أوامر أو خيارات CLI
    - توثيق أسطح أوامر جديدة
summary: مرجع CLI الخاص بـ OpenClaw لأوامر `openclaw` والأوامر الفرعية والخيارات
title: مرجع CLI
x-i18n:
    generated_at: "2026-04-05T12:41:31Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7c25e5ebfe256412b44130dba39cf39b0a7d1d22e3abb417345e95c95ca139bf
    source_path: cli/index.md
    workflow: 15
---

# مرجع CLI

تصف هذه الصفحة سلوك CLI الحالي. إذا تغيّرت الأوامر، فحدّث هذا المستند.

## صفحات الأوامر

- [`setup`](/cli/setup)
- [`onboard`](/cli/onboard)
- [`configure`](/cli/configure)
- [`config`](/cli/config)
- [`completion`](/cli/completion)
- [`doctor`](/cli/doctor)
- [`dashboard`](/cli/dashboard)
- [`backup`](/cli/backup)
- [`reset`](/cli/reset)
- [`uninstall`](/cli/uninstall)
- [`update`](/cli/update)
- [`message`](/cli/message)
- [`agent`](/cli/agent)
- [`agents`](/cli/agents)
- [`acp`](/cli/acp)
- [`mcp`](/cli/mcp)
- [`status`](/cli/status)
- [`health`](/cli/health)
- [`sessions`](/cli/sessions)
- [`gateway`](/cli/gateway)
- [`logs`](/cli/logs)
- [`system`](/cli/system)
- [`models`](/cli/models)
- [`memory`](/cli/memory)
- [`directory`](/cli/directory)
- [`nodes`](/cli/nodes)
- [`devices`](/cli/devices)
- [`node`](/cli/node)
- [`approvals`](/cli/approvals)
- [`sandbox`](/cli/sandbox)
- [`tui`](/cli/tui)
- [`browser`](/cli/browser)
- [`cron`](/cli/cron)
- [`tasks`](/cli/index#tasks)
- [`flows`](/cli/flows)
- [`dns`](/cli/dns)
- [`docs`](/cli/docs)
- [`hooks`](/cli/hooks)
- [`webhooks`](/cli/webhooks)
- [`pairing`](/cli/pairing)
- [`qr`](/cli/qr)
- [`plugins`](/cli/plugins) (أوامر plugin)
- [`channels`](/cli/channels)
- [`security`](/cli/security)
- [`secrets`](/cli/secrets)
- [`skills`](/cli/skills)
- [`daemon`](/cli/daemon) (اسم مستعار قديم لأوامر خدمة gateway)
- [`clawbot`](/cli/clawbot) (مساحة أسماء اسم مستعار قديمة)
- [`voicecall`](/cli/voicecall) (plugin؛ إذا كان مثبتًا)

## العلامات العامة

- `--dev`: عزل الحالة تحت `~/.openclaw-dev` وتحويل المنافذ الافتراضية.
- `--profile <name>`: عزل الحالة تحت `~/.openclaw-<name>`.
- `--container <name>`: استهداف حاوية مسماة للتنفيذ.
- `--no-color`: تعطيل ألوان ANSI.
- `--update`: اختصار لـ `openclaw update` (لتثبيتات المصدر فقط).
- `-V`, `--version`, `-v`: طباعة الإصدار والخروج.

## تنسيق الإخراج

- لا تظهر ألوان ANSI ومؤشرات التقدم إلا في جلسات TTY.
- تظهر روابط OSC-8 كروابط قابلة للنقر في الطرفيات المدعومة؛ وإلا نعود إلى عناوين URL عادية.
- يعطل `--json` (و`--plain` حيثما كان مدعومًا) التنسيق للحصول على إخراج نظيف.
- يعطل `--no-color` تنسيق ANSI؛ كما يتم احترام `NO_COLOR=1`.
- تعرض الأوامر طويلة التشغيل مؤشر تقدم (OSC 9;4 عند الدعم).

## لوحة الألوان

يستخدم OpenClaw لوحة ألوان lobster لإخراج CLI.

- `accent` (#FF5A2D): العناوين، والتسميات، والإبرازات الأساسية.
- `accentBright` (#FF7A3D): أسماء الأوامر، والتأكيد.
- `accentDim` (#D14A22): نصوص الإبراز الثانوية.
- `info` (#FF8A5B): القيم المعلوماتية.
- `success` (#2FBF71): حالات النجاح.
- `warn` (#FFB020): التحذيرات، والرجوعات الاحتياطية، ولفت الانتباه.
- `error` (#E23D2D): الأخطاء، والإخفاقات.
- `muted` (#8B7F77): تقليل التركيز، والبيانات الوصفية.

المصدر المرجعي للوحة الألوان: `src/terminal/palette.ts` (وهي “لوحة lobster”).

## شجرة الأوامر

```
openclaw [--dev] [--profile <name>] <command>
  setup
  onboard
  configure
  config
    get
    set
    unset
    file
    schema
    validate
  completion
  doctor
  dashboard
  backup
    create
    verify
  security
    audit
  secrets
    reload
    audit
    configure
    apply
  reset
  uninstall
  update
    wizard
    status
  channels
    list
    status
    capabilities
    resolve
    logs
    add
    remove
    login
    logout
  directory
    self
    peers list
    groups list|members
  skills
    search
    install
    update
    list
    info
    check
  plugins
    list
    inspect
    install
    uninstall
    update
    enable
    disable
    doctor
    marketplace list
  memory
    status
    index
    search
  message
    send
    broadcast
    poll
    react
    reactions
    read
    edit
    delete
    pin
    unpin
    pins
    permissions
    search
    thread create|list|reply
    emoji list|upload
    sticker send|upload
    role info|add|remove
    channel info|list
    member info
    voice status
    event list|create
    timeout
    kick
    ban
  agent
  agents
    list
    add
    delete
    bindings
    bind
    unbind
    set-identity
  acp
  mcp
    serve
    list
    show
    set
    unset
  status
  health
  sessions
    cleanup
  tasks
    list
    audit
    maintenance
    show
    notify
    cancel
    flow list|show|cancel
  gateway
    call
    usage-cost
    health
    status
    probe
    discover
    install
    uninstall
    start
    stop
    restart
    run
  daemon
    status
    install
    uninstall
    start
    stop
    restart
  logs
  system
    event
    heartbeat last|enable|disable
    presence
  models
    list
    status
    set
    set-image
    aliases list|add|remove
    fallbacks list|add|remove|clear
    image-fallbacks list|add|remove|clear
    scan
    auth add|login|login-github-copilot|setup-token|paste-token
    auth order get|set|clear
  sandbox
    list
    recreate
    explain
  cron
    status
    list
    add
    edit
    rm
    enable
    disable
    runs
    run
  nodes
    status
    describe
    list
    pending
    approve
    reject
    rename
    invoke
    notify
    push
    canvas snapshot|present|hide|navigate|eval
    canvas a2ui push|reset
    camera list|snap|clip
    screen record
    location get
  devices
    list
    remove
    clear
    approve
    reject
    rotate
    revoke
  node
    run
    status
    install
    uninstall
    stop
    restart
  approvals
    get
    set
    allowlist add|remove
  browser
    status
    start
    stop
    reset-profile
    tabs
    open
    focus
    close
    profiles
    create-profile
    delete-profile
    screenshot
    snapshot
    navigate
    resize
    click
    type
    press
    hover
    drag
    select
    upload
    fill
    dialog
    wait
    evaluate
    console
    pdf
  hooks
    list
    info
    check
    enable
    disable
    install
    update
  webhooks
    gmail setup|run
  pairing
    list
    approve
  qr
  clawbot
    qr
  docs
  dns
    setup
  tui
```

ملاحظة: يمكن لـ plugins إضافة أوامر إضافية على المستوى الأعلى (مثل `openclaw voicecall`).

## الأمان

- `openclaw security audit` — تدقيق التكوين + الحالة المحلية بحثًا عن أخطاء أمان شائعة.
- `openclaw security audit --deep` — فحص Gateway مباشر بأفضل جهد ممكن.
- `openclaw security audit --fix` — تشديد الإعدادات الآمنة الافتراضية وأذونات الحالة/التكوين.

## الأسرار

### `secrets`

إدارة SecretRefs ونظافة وقت التشغيل/التكوين ذات الصلة.

الأوامر الفرعية:

- `secrets reload`
- `secrets audit`
- `secrets configure`
- `secrets apply --from <path>`

خيارات `secrets reload`:

- `--url`, `--token`, `--timeout`, `--expect-final`, `--json`

خيارات `secrets audit`:

- `--check`
- `--allow-exec`
- `--json`

خيارات `secrets configure`:

- `--apply`
- `--yes`
- `--providers-only`
- `--skip-provider-setup`
- `--agent <id>`
- `--allow-exec`
- `--plan-out <path>`
- `--json`

خيارات `secrets apply --from <path>`:

- `--dry-run`
- `--allow-exec`
- `--json`

ملاحظات:

- `reload` هو Gateway RPC ويحتفظ بآخر لقطة وقت تشغيل سليمة معروفة عندما يفشل الحل.
- يعيد `audit --check` قيمة غير صفرية عند وجود نتائج؛ وتستخدم المراجع غير المحلولة قيمة خروج غير صفرية ذات أولوية أعلى.
- يتم تخطي فحوصات exec التجريبية افتراضيًا؛ استخدم `--allow-exec` لتفعيلها.

## Plugins

إدارة الامتدادات وتكوينها:

- `openclaw plugins list` — اكتشاف plugins (استخدم `--json` لإخراج آلي).
- `openclaw plugins inspect <id>` — عرض تفاصيل plugin (`info` اسم مستعار).
- `openclaw plugins install <path|.tgz|npm-spec|plugin@marketplace>` — تثبيت plugin (أو إضافة مسار plugin إلى `plugins.load.paths`; استخدم `--force` لاستبدال هدف تثبيت موجود).
- `openclaw plugins marketplace list <marketplace>` — سرد إدخالات marketplace قبل التثبيت.
- `openclaw plugins enable <id>` / `disable <id>` — تبديل `plugins.entries.<id>.enabled`.
- `openclaw plugins doctor` — الإبلاغ عن أخطاء تحميل plugin.

تتطلب معظم تغييرات plugin إعادة تشغيل gateway. راجع [/plugin](/tools/plugin).

## الذاكرة

بحث متجهي عبر `MEMORY.md` + `memory/*.md`:

- `openclaw memory status` — عرض إحصاءات الفهرس؛ استخدم `--deep` لفحوصات جاهزية المتجهات + التضمين أو `--fix` لإصلاح آثار الاستدعاء/الترقية القديمة.
- `openclaw memory index` — إعادة فهرسة ملفات الذاكرة.
- `openclaw memory search "<query>"` (أو `--query "<query>"`) — بحث دلالي في الذاكرة.
- `openclaw memory promote` — ترتيب الاستدعاءات القصيرة الأجل وإلحاق أفضل الإدخالات اختياريًا في `MEMORY.md`.

## Sandbox

إدارة بيئات sandbox لتنفيذ الوكيل المعزول. راجع [/cli/sandbox](/cli/sandbox).

الأوامر الفرعية:

- `sandbox list [--browser] [--json]`
- `sandbox recreate [--all] [--session <key>] [--agent <id>] [--browser] [--force]`
- `sandbox explain [--session <key>] [--agent <id>] [--json]`

ملاحظات:

- يزيل `sandbox recreate` بيئات التشغيل الحالية بحيث تعاد تهيئتها في الاستخدام التالي وفق التكوين الحالي.
- بالنسبة إلى الخلفيات البعيدة `ssh` وOpenShell `remote`، فإن recreate يحذف مساحة العمل البعيدة القياسية للنطاق المحدد.

## أوامر الشرطة المائلة في الدردشة

تدعم رسائل الدردشة أوامر `/...` (النصية والأصلية). راجع [/tools/slash-commands](/tools/slash-commands).

أبرز النقاط:

- `/status` للتشخيص السريع.
- `/config` لتغييرات التكوين المحفوظة.
- `/debug` لتجاوزات تكوين وقت التشغيل فقط (في الذاكرة، وليس على القرص؛ يتطلب `commands.debug: true`).

## الإعداد + التهيئة الأولية

### `completion`

إنشاء نصوص الإكمال التلقائي للصدفة وتثبيتها اختياريًا داخل ملف تعريف الصدفة.

الخيارات:

- `-s, --shell <zsh|bash|powershell|fish>`
- `-i, --install`
- `--write-state`
- `-y, --yes`

ملاحظات:

- من دون `--install` أو `--write-state`، يطبع `completion` النص إلى stdout.
- يكتب `--install` كتلة `OpenClaw Completion` في ملف تعريف الصدفة ويوجهها إلى النص المخزن مؤقتًا ضمن دليل حالة OpenClaw.

### `setup`

تهيئة التكوين + مساحة العمل.

الخيارات:

- `--workspace <dir>`: مسار مساحة عمل الوكيل (الافتراضي `~/.openclaw/workspace`).
- `--wizard`: تشغيل التهيئة الأولية.
- `--non-interactive`: تشغيل التهيئة الأولية من دون مطالبات.
- `--mode <local|remote>`: وضع onboard.
- `--remote-url <url>`: عنوان URL لـ Gateway البعيد.
- `--remote-token <token>`: رمز Gateway البعيد.

يعمل onboarding تلقائيًا عند وجود أي من علامات التهيئة الأولية (`--non-interactive`, `--mode`, `--remote-url`, `--remote-token`).

### `onboard`

تهيئة أولية تفاعلية لـ gateway ومساحة العمل وSkills.

الخيارات:

- `--workspace <dir>`
- `--reset` (إعادة ضبط التكوين + بيانات الاعتماد + الجلسات قبل التهيئة الأولية)
- `--reset-scope <config|config+creds+sessions|full>` (الافتراضي `config+creds+sessions`؛ استخدم `full` لإزالة مساحة العمل أيضًا)
- `--non-interactive`
- `--mode <local|remote>`
- `--flow <quickstart|advanced|manual>` (`manual` اسم مستعار لـ `advanced`)
- `--auth-choice <choice>` حيث تكون `<choice>` واحدة من:
  `chutes`, `deepseek-api-key`, `openai-codex`, `openai-api-key`,
  `openrouter-api-key`, `kilocode-api-key`, `litellm-api-key`, `ai-gateway-api-key`,
  `cloudflare-ai-gateway-api-key`, `moonshot-api-key`, `moonshot-api-key-cn`,
  `kimi-code-api-key`, `synthetic-api-key`, `venice-api-key`, `together-api-key`,
  `huggingface-api-key`, `apiKey`, `gemini-api-key`, `google-gemini-cli`, `zai-api-key`,
  `zai-coding-global`, `zai-coding-cn`, `zai-global`, `zai-cn`, `xiaomi-api-key`,
  `minimax-global-oauth`, `minimax-global-api`, `minimax-cn-oauth`, `minimax-cn-api`,
  `opencode-zen`, `opencode-go`, `github-copilot`, `copilot-proxy`, `xai-api-key`,
  `mistral-api-key`, `volcengine-api-key`, `byteplus-api-key`, `qianfan-api-key`,
  `qwen-standard-api-key-cn`, `qwen-standard-api-key`, `qwen-api-key-cn`, `qwen-api-key`,
  `modelstudio-standard-api-key-cn`, `modelstudio-standard-api-key`,
  `modelstudio-api-key-cn`, `modelstudio-api-key`, `custom-api-key`, `skip`
- ملاحظة Qwen: عائلة `qwen-*` هي العائلة الأساسية لـ auth-choice. وتظل
  معرّفات `modelstudio-*` مقبولة فقط كأسماء مستعارة قديمة للتوافق.
- `--secret-input-mode <plaintext|ref>` (الافتراضي `plaintext`؛ استخدم `ref` لتخزين مراجع env الافتراضية للمزوّد بدلًا من المفاتيح النصية الصريحة)
- `--anthropic-api-key <key>`
- `--openai-api-key <key>`
- `--mistral-api-key <key>`
- `--openrouter-api-key <key>`
- `--ai-gateway-api-key <key>`
- `--moonshot-api-key <key>`
- `--kimi-code-api-key <key>`
- `--gemini-api-key <key>`
- `--zai-api-key <key>`
- `--minimax-api-key <key>`
- `--opencode-zen-api-key <key>`
- `--opencode-go-api-key <key>`
- `--custom-base-url <url>` (غير تفاعلي؛ يُستخدم مع `--auth-choice custom-api-key`)
- `--custom-model-id <id>` (غير تفاعلي؛ يُستخدم مع `--auth-choice custom-api-key`)
- `--custom-api-key <key>` (غير تفاعلي؛ اختياري؛ يُستخدم مع `--auth-choice custom-api-key`؛ يعود إلى `CUSTOM_API_KEY` عند عدم التحديد)
- `--custom-provider-id <id>` (غير تفاعلي؛ معرّف مزوّد مخصص اختياري)
- `--custom-compatibility <openai|anthropic>` (غير تفاعلي؛ اختياري؛ الافتراضي `openai`)
- `--gateway-port <port>`
- `--gateway-bind <loopback|lan|tailnet|auto|custom>`
- `--gateway-auth <token|password>`
- `--gateway-token <token>`
- `--gateway-token-ref-env <name>` (غير تفاعلي؛ تخزين `gateway.auth.token` كـ env SecretRef؛ يتطلب أن يكون متغير البيئة هذا مضبوطًا؛ لا يمكن دمجه مع `--gateway-token`)
- `--gateway-password <password>`
- `--remote-url <url>`
- `--remote-token <token>`
- `--tailscale <off|serve|funnel>`
- `--tailscale-reset-on-exit`
- `--install-daemon`
- `--no-install-daemon` (اسم مستعار: `--skip-daemon`)
- `--daemon-runtime <node|bun>`
- `--skip-channels`
- `--skip-skills`
- `--skip-search`
- `--skip-health`
- `--skip-ui`
- `--cloudflare-ai-gateway-account-id <id>`
- `--cloudflare-ai-gateway-gateway-id <id>`
- `--node-manager <npm|pnpm|bun>` (مدير عقد setup/onboarding لـ Skills؛ يوصى بـ pnpm، ويدعم bun أيضًا)
- `--json`

### `configure`

معالج تكوين تفاعلي (النماذج، القنوات، Skills، gateway).

الخيارات:

- `--section <section>` (قابل للتكرار؛ يقصر المعالج على أقسام محددة)

### `config`

مساعدات تكوين غير تفاعلية (`get/set/unset/file/schema/validate`). يؤدي تشغيل `openclaw config` دون
أمر فرعي إلى تشغيل المعالج.

الأوامر الفرعية:

- `config get <path>`: طباعة قيمة تكوين (مسار dot/bracket).
- `config set`: يدعم أربعة أوضاع للتعيين:
  - وضع القيمة: `config set <path> <value>` (تحليل JSON5 أو string)
  - وضع منشئ SecretRef: ‏`config set <path> --ref-provider <provider> --ref-source <source> --ref-id <id>`
  - وضع منشئ المزوّد: ‏`config set secrets.providers.<alias> --provider-source <env|file|exec> ...`
  - وضع الدُفعات: ‏`config set --batch-json '<json>'` أو `config set --batch-file <path>`
- `config set --dry-run`: التحقق من التعيينات من دون كتابة `openclaw.json` (يتم تخطي فحوصات exec SecretRef افتراضيًا).
- `config set --allow-exec --dry-run`: تفعيل فحوصات exec SecretRef التجريبية (قد تنفذ أوامر المزوّد).
- `config set --dry-run --json`: إصدار إخراج تجريبي قابل للقراءة آليًا (الفحوصات + إشارة الاكتمال، والعمليات، والمراجع التي تم فحصها/تخطيها، والأخطاء).
- `config set --strict-json`: فرض تحليل JSON5 لإدخال المسار/القيمة. يظل `--json` اسمًا مستعارًا قديمًا للتحليل الصارم خارج وضع إخراج dry-run.
- `config unset <path>`: إزالة قيمة.
- `config file`: طباعة مسار ملف التكوين النشط.
- `config schema`: طباعة JSON schema المولّد لـ `openclaw.json`، بما في ذلك نشر بيانات `title` / `description` التوثيقية عبر فروع الكائنات المتداخلة والبدائل العامة وعناصر المصفوفات والتركيبات، بالإضافة إلى بيانات تعريف schema للقنوات/plugins المباشرة بأفضل جهد.
- `config validate`: التحقق من صحة التكوين الحالي مقابل schema من دون بدء gateway.
- `config validate --json`: إصدار إخراج JSON قابل للقراءة آليًا.

### `doctor`

فحوصات صحية + إصلاحات سريعة (التكوين + gateway + الخدمات القديمة).

الخيارات:

- `--no-workspace-suggestions`: تعطيل تلميحات ذاكرة مساحة العمل.
- `--yes`: قبول القيم الافتراضية من دون مطالبة (للعمل بدون رأس).
- `--non-interactive`: تخطي المطالبات؛ وتطبيق عمليات الترحيل الآمنة فقط.
- `--deep`: فحص خدمات النظام بحثًا عن تثبيتات gateway إضافية.
- `--repair` (اسم مستعار: `--fix`): محاولة الإصلاح التلقائي للمشكلات المكتشفة.
- `--force`: فرض الإصلاحات حتى عندما لا تكون مطلوبة بشكل صارم.
- `--generate-gateway-token`: إنشاء رمز مصادقة جديد لـ gateway.

### `dashboard`

افتح Control UI باستخدام رمزك الحالي.

الخيارات:

- `--no-open`: طباعة عنوان URL دون تشغيل متصفح

ملاحظات:

- بالنسبة إلى رموز gateway التي تديرها SecretRef، يطبع `dashboard` أو يفتح عنوان URL غير مميز بالرمز بدلًا من كشف السر في مخرجات الطرفية أو وسيطات تشغيل المتصفح.

### `update`

تحديث CLI المثبت.

خيارات الجذر:

- `--json`
- `--no-restart`
- `--dry-run`
- `--channel <stable|beta|dev>`
- `--tag <dist-tag|version|spec>`
- `--timeout <seconds>`
- `--yes`

الأوامر الفرعية:

- `update status`
- `update wizard`

خيارات `update status`:

- `--json`
- `--timeout <seconds>`

خيارات `update wizard`:

- `--timeout <seconds>`

ملاحظات:

- يعيد `openclaw --update` الكتابة إلى `openclaw update`.

### `backup`

إنشاء أرشيفات نسخ احتياطي محلية والتحقق منها لحالة OpenClaw.

الأوامر الفرعية:

- `backup create`
- `backup verify <archive>`

خيارات `backup create`:

- `--output <path>`
- `--json`
- `--dry-run`
- `--verify`
- `--only-config`
- `--no-include-workspace`

خيارات `backup verify <archive>`:

- `--json`

## مساعدات القنوات

### `channels`

إدارة حسابات قنوات الدردشة (WhatsApp/Telegram/Discord/Google Chat/Slack/Mattermost (plugin)/Signal/iMessage/Microsoft Teams).

الأوامر الفرعية:

- `channels list`: عرض القنوات المكوّنة وملفات تعريف المصادقة.
- `channels status`: التحقق من إمكانية الوصول إلى gateway وصحة القناة (`--probe` يشغّل فحوصات probe/audit مباشرة لكل حساب عند إمكانية الوصول إلى gateway؛ وإذا لم يكن ممكنًا، يعود إلى ملخصات القنوات المعتمدة على التكوين فقط. استخدم `openclaw health` أو `openclaw status --deep` لفحوصات صحة gateway الأوسع).
- نصيحة: يطبع `channels status` تحذيرات مع إصلاحات مقترحة عندما يستطيع اكتشاف سوء التكوينات الشائعة (ثم يوجّهك إلى `openclaw doctor`).
- `channels logs`: عرض سجلات القنوات الأخيرة من ملف سجل gateway.
- `channels add`: إعداد بأسلوب المعالج عندما لا تُمرر أي علامات؛ وتحول العلامات إلى الوضع غير التفاعلي.
  - عند إضافة حساب غير افتراضي إلى قناة ما زالت تستخدم تكوينًا أحادي الحساب على المستوى الأعلى، يقوم OpenClaw بترقية القيم ذات النطاق الحسابي إلى خريطة حسابات القناة قبل كتابة الحساب الجديد. تستخدم معظم القنوات `accounts.default`؛ أما Matrix فيمكنه الاحتفاظ بهدف named/default موجود ومطابق بدلًا من ذلك.
  - لا يقوم `channels add` غير التفاعلي بإنشاء/ترقية الروابط تلقائيًا؛ وتستمر الروابط الخاصة بالقناة فقط في مطابقة الحساب الافتراضي.
- `channels remove`: يعطل افتراضيًا؛ مرر `--delete` لإزالة إدخالات التكوين من دون مطالبات.
- `channels login`: تسجيل دخول تفاعلي للقناة (WhatsApp Web فقط).
- `channels logout`: تسجيل الخروج من جلسة قناة (عند الدعم).

الخيارات الشائعة:

- `--channel <name>`: ‏`whatsapp|telegram|discord|googlechat|slack|mattermost|signal|imessage|msteams`
- `--account <id>`: معرّف حساب القناة (الافتراضي `default`)
- `--name <label>`: اسم العرض للحساب

خيارات `channels login`:

- `--channel <channel>` (الافتراضي `whatsapp`; يدعم `whatsapp`/`web`)
- `--account <id>`
- `--verbose`

خيارات `channels logout`:

- `--channel <channel>` (الافتراضي `whatsapp`)
- `--account <id>`

خيارات `channels list`:

- `--no-usage`: تخطي لقطات استخدام/حصة مزوّد النموذج (OAuth/API فقط).
- `--json`: إخراج JSON (يشمل الاستخدام ما لم يتم تعيين `--no-usage`).

خيارات `channels status`:

- `--probe`
- `--timeout <ms>`
- `--json`

خيارات `channels capabilities`:

- `--channel <name>`
- `--account <id>` (فقط مع `--channel`)
- `--target <dest>`
- `--timeout <ms>`
- `--json`

خيارات `channels resolve`:

- `<entries...>`
- `--channel <name>`
- `--account <id>`
- `--kind <auto|user|group>`
- `--json`

خيارات `channels logs`:

- `--channel <name|all>` (الافتراضي `all`)
- `--lines <n>` (الافتراضي `200`)
- `--json`

ملاحظات:

- يدعم `channels login` الخيار `--verbose`.
- لا ينطبق `channels capabilities --account` إلا عند تعيين `--channel`.
- يمكن لـ `channels status --probe` إظهار حالة النقل بالإضافة إلى نتائج probe/audit مثل `works` أو `probe failed` أو `audit ok` أو `audit failed` بحسب دعم القناة.

تفاصيل أكثر: [/concepts/oauth](/concepts/oauth)

أمثلة:

```bash
openclaw channels add --channel telegram --account alerts --name "Alerts Bot" --token $TELEGRAM_BOT_TOKEN
openclaw channels add --channel discord --account work --name "Work Bot" --token $DISCORD_BOT_TOKEN
openclaw channels remove --channel discord --account work --delete
openclaw channels status --probe
openclaw status --deep
```

### `directory`

البحث عن معرّفات الذات والأقران والمجموعات للقنوات التي تعرض سطح directory. راجع [`openclaw directory`](/cli/directory).

الخيارات الشائعة:

- `--channel <name>`
- `--account <id>`
- `--json`

الأوامر الفرعية:

- `directory self`
- `directory peers list [--query <text>] [--limit <n>]`
- `directory groups list [--query <text>] [--limit <n>]`
- `directory groups members --group-id <id> [--limit <n>]`

### `skills`

إدراج Skills المتاحة وفحصها بالإضافة إلى معلومات الجاهزية.

الأوامر الفرعية:

- `skills search [query...]`: البحث في Skills الخاصة بـ ClawHub.
- `skills search --limit <n> --json`: تحديد حد أقصى للنتائج أو إصدار إخراج قابل للقراءة آليًا.
- `skills install <slug>`: تثبيت Skill من ClawHub داخل مساحة العمل النشطة.
- `skills install <slug> --version <version>`: تثبيت إصدار محدد من ClawHub.
- `skills install <slug> --force`: استبدال مجلد Skill موجود في مساحة العمل.
- `skills update <slug|--all>`: تحديث Skills المتتبعة من ClawHub.
- `skills list`: إدراج Skills (الافتراضي عند عدم وجود أمر فرعي).
- `skills list --json`: إصدار قائمة Skills قابلة للقراءة آليًا إلى stdout.
- `skills list --verbose`: تضمين المتطلبات المفقودة في الجدول.
- `skills info <name>`: عرض تفاصيل Skill واحدة.
- `skills info <name> --json`: إصدار تفاصيل قابلة للقراءة آليًا إلى stdout.
- `skills check`: ملخص الجاهز مقابل ما تنقصه المتطلبات.
- `skills check --json`: إصدار مخرجات جاهزية قابلة للقراءة آليًا إلى stdout.

الخيارات:

- `--eligible`: إظهار Skills الجاهزة فقط.
- `--json`: إخراج JSON (من دون تنسيق).
- `-v`, `--verbose`: تضمين تفاصيل المتطلبات المفقودة.

نصيحة: استخدم `openclaw skills search` و`openclaw skills install` و`openclaw skills update` للـ Skills المدعومة من ClawHub.

### `pairing`

الموافقة على طلبات pairing للرسائل المباشرة عبر القنوات.

الأوامر الفرعية:

- `pairing list [channel] [--channel <channel>] [--account <id>] [--json]`
- `pairing approve <channel> <code> [--account <id>] [--notify]`
- `pairing approve --channel <channel> [--account <id>] <code> [--notify]`

ملاحظات:

- إذا كانت هناك قناة واحدة فقط قادرة على pairing ومكوّنة، فيُسمح أيضًا بـ `pairing approve <code>`.
- يدعم كل من `list` و`approve` الخيار `--account <id>` للقنوات متعددة الحسابات.

### `devices`

إدارة إدخالات pairing الخاصة بأجهزة gateway ورموز الأجهزة لكل دور.

الأوامر الفرعية:

- `devices list [--json]`
- `devices approve [requestId] [--latest]`
- `devices reject <requestId>`
- `devices remove <deviceId>`
- `devices clear --yes [--pending]`
- `devices rotate --device <id> --role <role> [--scope <scope...>]`
- `devices revoke --device <id> --role <role>`

ملاحظات:

- يمكن لـ `devices list` و`devices approve` الرجوع إلى ملفات pairing المحلية على local loopback عندما لا يتوفر نطاق pairing المباشر.
- يختار `devices approve` تلقائيًا أحدث طلب معلّق عندما لا يتم تمرير `requestId` أو عند تعيين `--latest`.
- تعيد عمليات إعادة الاتصال باستخدام الرموز المخزنة استخدام نطاقات الموافقة المؤقتة المخزنة للرمز؛ ويحدّث
  `devices rotate --scope ...` مجموعة النطاقات المخزنة تلك لعمليات
  إعادة الاتصال المستقبلية باستخدام الرمز المخزن مؤقتًا.
- يعيد كل من `devices rotate` و`devices revoke` حمولات JSON.

### `qr`

إنشاء QR pairing للجوال ورمز إعداد من تكوين Gateway الحالي. راجع [`openclaw qr`](/cli/qr).

الخيارات:

- `--remote`
- `--url <url>`
- `--public-url <url>`
- `--token <token>`
- `--password <password>`
- `--setup-code-only`
- `--no-ascii`
- `--json`

ملاحظات:

- `--token` و`--password` متنافيان.
- يحمل رمز الإعداد رمز bootstrap قصير الأجل، وليس رمز/كلمة مرور gateway المشتركة.
- يحافظ التسليم المضمّن لـ bootstrap على رمز العقدة الأساسية عند `scopes: []`.
- يظل أي رمز bootstrap للمشغل يتم تسليمه مقيّدًا بـ `operator.approvals` و`operator.read` و`operator.talk.secrets` و`operator.write`.
- تكون فحوصات نطاق bootstrap مسبوقة بالدور، بحيث لا تلبّي قائمة السماح الخاصة بالمشغل إلا طلبات المشغل؛ وما زالت الأدوار غير المشغلة تحتاج إلى نطاقات تحت بادئة دورها الخاص.
- يمكن لـ `--remote` استخدام `gateway.remote.url` أو عنوان URL النشط لـ Tailscale Serve/Funnel.
- بعد المسح، وافق على الطلب باستخدام `openclaw devices list` / `openclaw devices approve <requestId>`.

### `clawbot`

مساحة أسماء اسم مستعار قديمة. تدعم حاليًا `openclaw clawbot qr`، والتي تُطابق [`openclaw qr`](/cli/qr).

### `hooks`

إدارة الخطافات الداخلية للوكيل.

الأوامر الفرعية:

- `hooks list`
- `hooks info <name>`
- `hooks check`
- `hooks enable <name>`
- `hooks disable <name>`
- `hooks install <path-or-spec>` (اسم مستعار متروك لـ `openclaw plugins install`)
- `hooks update [id]` (اسم مستعار متروك لـ `openclaw plugins update`)

الخيارات الشائعة:

- `--json`
- `--eligible`
- `-v`, `--verbose`

ملاحظات:

- لا يمكن تمكين أو تعطيل الخطافات التي تديرها plugins عبر `openclaw hooks`؛ بل يجب تمكين أو تعطيل plugin المالك بدلًا من ذلك.
- لا يزال `hooks install` و`hooks update` يعملان كأسماء مستعارة للتوافق، لكنهما يطبعان تحذيرات تقادم ويعيدان التوجيه إلى أوامر plugin.

### `webhooks`

مساعدات webhook. السطح المضمّن الحالي هو إعداد + تشغيل Gmail Pub/Sub:

- `webhooks gmail setup`
- `webhooks gmail run`

### `webhooks gmail`

إعداد + تشغيل خطاف Gmail Pub/Sub. راجع [Gmail Pub/Sub](/automation/cron-jobs#gmail-pubsub-integration).

الأوامر الفرعية:

- `webhooks gmail setup` (يتطلب `--account <email>`؛ ويدعم `--project`, `--topic`, `--subscription`, `--label`, `--hook-url`, `--hook-token`, `--push-token`, `--bind`, `--port`, `--path`, `--include-body`, `--max-bytes`, `--renew-minutes`, `--tailscale`, `--tailscale-path`, `--tailscale-target`, `--push-endpoint`, `--json`)
- `webhooks gmail run` (تجاوزات وقت التشغيل للعلامات نفسها)

ملاحظات:

- يضبط `setup` Gmail watch بالإضافة إلى مسار push المواجه لـ OpenClaw.
- يبدأ `run` مراقب Gmail المحلي/حلقة التجديد مع تجاوزات وقت تشغيل اختيارية.

### `dns`

مساعدات DNS للاكتشاف واسع النطاق (CoreDNS + Tailscale). السطح المضمّن الحالي:

- `dns setup [--domain <domain>] [--apply]`

### `dns setup`

مساعد DNS للاكتشاف واسع النطاق (CoreDNS + Tailscale). راجع [/gateway/discovery](/gateway/discovery).

الخيارات:

- `--domain <domain>`
- `--apply`: تثبيت/تحديث تكوين CoreDNS (يتطلب sudo؛ macOS فقط).

ملاحظات:

- من دون `--apply`، يكون هذا مساعد تخطيط يطبع تكوين OpenClaw + Tailscale الموصى به لـ DNS.
- يدعم `--apply` حاليًا macOS مع CoreDNS من Homebrew فقط.

## المراسلة + الوكيل

### `message`

إرسال رسائل موحد + إجراءات القنوات.

راجع: [/cli/message](/cli/message)

الأوامر الفرعية:

- `message send|poll|react|reactions|read|edit|delete|pin|unpin|pins|permissions|search|timeout|kick|ban`
- `message thread <create|list|reply>`
- `message emoji <list|upload>`
- `message sticker <send|upload>`
- `message role <info|add|remove>`
- `message channel <info|list>`
- `message member info`
- `message voice status`
- `message event <list|create>`

أمثلة:

- `openclaw message send --target +15555550123 --message "Hi"`
- `openclaw message poll --channel discord --target channel:123 --poll-question "Snack?" --poll-option Pizza --poll-option Sushi`

### `agent`

تشغيل دورة وكيل واحدة عبر Gateway (أو مضمّنة باستخدام `--local`).

مرّر محدد جلسة واحدًا على الأقل: `--to` أو `--session-id` أو `--agent`.

مطلوب:

- `-m, --message <text>`

الخيارات:

- `-t, --to <dest>` (لمفتاح الجلسة والتسليم الاختياري)
- `--session-id <id>`
- `--agent <id>` (معرّف الوكيل؛ يتجاوز روابط التوجيه)
- `--thinking <off|minimal|low|medium|high|xhigh>` (يختلف دعم المزوّد؛ ولا يُقيّد على مستوى CLI حسب النموذج)
- `--verbose <on|off>`
- `--channel <channel>` (قناة التسليم؛ احذفها لاستخدام قناة الجلسة الرئيسية)
- `--reply-to <target>` (تجاوز هدف التسليم، منفصل عن توجيه الجلسة)
- `--reply-channel <channel>` (تجاوز قناة التسليم)
- `--reply-account <id>` (تجاوز معرّف حساب التسليم)
- `--local` (تشغيل مضمّن؛ وما زال سجل plugins يُحمّل مسبقًا أولًا)
- `--deliver`
- `--json`
- `--timeout <seconds>`

ملاحظات:

- يعود وضع Gateway إلى الوكيل المضمن عندما يفشل طلب Gateway.
- ما يزال `--local` يحمّل سجل plugins مسبقًا، لذلك تبقى providers والأدوات والقنوات التي توفرها plugins متاحة أثناء التشغيل المضمن.
- تؤثر `--channel` و`--reply-channel` و`--reply-account` في تسليم الرد، وليس في التوجيه.

### `agents`

إدارة الوكلاء المعزولين (مساحات العمل + المصادقة + التوجيه).

يعادل تشغيل `openclaw agents` دون أمر فرعي الأمر `openclaw agents list`.

#### `agents list`

إدراج الوكلاء المكوّنين.

الخيارات:

- `--json`
- `--bindings`

#### `agents add [name]`

إضافة وكيل معزول جديد. يشغّل المعالج الإرشادي ما لم يتم تمرير علامات (أو `--non-interactive`)؛ ويكون `--workspace` مطلوبًا في الوضع غير التفاعلي.

الخيارات:

- `--workspace <dir>`
- `--model <id>`
- `--agent-dir <dir>`
- `--bind <channel[:accountId]>` (قابل للتكرار)
- `--non-interactive`
- `--json`

تستخدم مواصفات الربط الصيغة `channel[:accountId]`. وعندما يُحذف `accountId`، قد يحل OpenClaw نطاق الحساب عبر افتراضيات القناة/خطافات plugin؛ وإلا يكون الربط ربط قناة دون نطاق حساب صريح.
يؤدي تمرير أي علامات add صريحة إلى تحويل الأمر إلى المسار غير التفاعلي. والاسم `main` محجوز ولا يمكن استخدامه كمعرّف للوكيل الجديد.

#### `agents bindings`

إدراج روابط التوجيه.

الخيارات:

- `--agent <id>`
- `--json`

#### `agents bind`

إضافة روابط توجيه لوكيل.

الخيارات:

- `--agent <id>` (الافتراضي هو الوكيل الافتراضي الحالي)
- `--bind <channel[:accountId]>` (قابل للتكرار)
- `--json`

#### `agents unbind`

إزالة روابط توجيه لوكيل.

الخيارات:

- `--agent <id>` (الافتراضي هو الوكيل الافتراضي الحالي)
- `--bind <channel[:accountId]>` (قابل للتكرار)
- `--all`
- `--json`

استخدم إما `--all` أو `--bind`، وليس كليهما.

#### `agents delete <id>`

حذف وكيل وتقليص مساحة عمله + حالته.

الخيارات:

- `--force`
- `--json`

ملاحظات:

- لا يمكن حذف `main`.
- من دون `--force`، يلزم تأكيد تفاعلي.

#### `agents set-identity`

تحديث هوية وكيل (الاسم/السمة/emoji/avatar).

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
- عند عدم توفير حقول هوية صريحة، يقرأ الأمر `IDENTITY.md`.

### `acp`

تشغيل جسر ACP الذي يربط IDEs بـ Gateway.

خيارات الجذر:

- `--url <url>`
- `--token <token>`
- `--token-file <path>`
- `--password <password>`
- `--password-file <path>`
- `--session <key>`
- `--session-label <label>`
- `--require-existing`
- `--reset-session`
- `--no-prefix-cwd`
- `--provenance <off|meta|meta+receipt>`
- `--verbose`

#### `acp client`

عميل ACP تفاعلي لتصحيح الجسر.

الخيارات:

- `--cwd <dir>`
- `--server <command>`
- `--server-args <args...>`
- `--server-verbose`
- `--verbose`

راجع [`acp`](/cli/acp) للسلوك الكامل والملاحظات الأمنية والأمثلة.

### `mcp`

إدارة تعريفات خوادم MCP المحفوظة وكشف قنوات OpenClaw عبر MCP stdio.

#### `mcp serve`

كشف محادثات قنوات OpenClaw الموجّهة عبر MCP stdio.

الخيارات:

- `--url <url>`
- `--token <token>`
- `--token-file <path>`
- `--password <password>`
- `--password-file <path>`
- `--claude-channel-mode <auto|on|off>`
- `--verbose`

#### `mcp list`

إدراج تعريفات خوادم MCP المحفوظة.

الخيارات:

- `--json`

#### `mcp show [name]`

عرض تعريف خادم MCP محفوظ واحد أو كائن خادم MCP المحفوظ الكامل.

الخيارات:

- `--json`

#### `mcp set <name> <value>`

حفظ تعريف خادم MCP واحد من كائن JSON.

#### `mcp unset <name>`

إزالة تعريف خادم MCP محفوظ واحد.

### `approvals`

إدارة موافقات exec. الاسم المستعار: `exec-approvals`.

#### `approvals get`

جلب لقطة موافقات exec والسياسة الفعالة.

الخيارات:

- `--node <node>`
- `--gateway`
- `--json`
- خيارات node RPC من `openclaw nodes`

#### `approvals set`

استبدال موافقات exec بـ JSON من ملف أو stdin.

الخيارات:

- `--node <node>`
- `--gateway`
- `--file <path>`
- `--stdin`
- `--json`
- خيارات node RPC من `openclaw nodes`

#### `approvals allowlist add|remove`

تحرير قائمة سماح exec لكل وكيل.

الخيارات:

- `--node <node>`
- `--gateway`
- `--agent <id>` (الافتراضي `*`)
- `--json`
- خيارات node RPC من `openclaw nodes`

### `status`

عرض صحة الجلسة المرتبطة والمستلمين الأخيرين.

الخيارات:

- `--json`
- `--all` (تشخيص كامل؛ للقراءة فقط وقابل للمشاركة)
- `--deep` (اطلب من gateway فحصًا صحيًا مباشرًا، بما في ذلك probes القنوات عند الدعم)
- `--usage` (عرض استخدام/حصة مزوّد النموذج)
- `--timeout <ms>`
- `--verbose`
- `--debug` (اسم مستعار لـ `--verbose`)

ملاحظات:

- تتضمن النظرة العامة حالة خدمة Gateway + مضيف العقدة عند توفرها.
- يطبع `--usage` نوافذ استخدام المزوّد المطبّعة بالشكل `X% left`.

### تتبع الاستخدام

يمكن لـ OpenClaw إظهار استخدام/حصة المزوّد عند توفر بيانات اعتماد OAuth/API.

الأسطح:

- `/status` (يضيف سطر استخدام قصيرًا للمزوّد عند توفره)
- `openclaw status --usage` (يطبع تفصيل استخدام المزوّد الكامل)
- شريط قوائم macOS (قسم Usage ضمن Context)

ملاحظات:

- تأتي البيانات مباشرة من نقاط نهاية استخدام المزوّد (من دون تقديرات).
- يُطبّع الإخراج المقروء بشريًا إلى `X% left` عبر جميع المزوّدين.
- المزوّدون ذوو نوافذ الاستخدام الحالية: Anthropic وGitHub Copilot وGemini CLI وOpenAI Codex وMiniMax وXiaomi وz.ai.
- ملاحظة MiniMax: تعني القيم الخام `usage_percent` / `usagePercent` الحصة المتبقية، لذا يعكسها OpenClaw قبل العرض؛ وتظل الحقول القائمة على العد صاحبة الأولوية عند وجودها. تفضّل استجابات `model_remains` إدخال chat-model، وتشتق تسمية النافذة من الطوابع الزمنية عند الحاجة، وتتضمن اسم النموذج في تسمية الخطة.
- تأتي مصادقة الاستخدام من خطافات خاصة بالمزوّد عند توفرها؛ وإلا يعود OpenClaw إلى مطابقة بيانات اعتماد OAuth/API key من ملفات تعريف المصادقة أو env أو التكوين. وإذا لم يُحل أي منها، يُخفى الاستخدام.
- التفاصيل: راجع [Usage tracking](/concepts/usage-tracking).

### `health`

جلب الحالة الصحية من Gateway الجاري تشغيله.

الخيارات:

- `--json`
- `--timeout <ms>`
- `--verbose` (فرض probe مباشر وطباعة تفاصيل اتصال gateway)
- `--debug` (اسم مستعار لـ `--verbose`)

ملاحظات:

- يمكن أن يعيد `health` الافتراضي لقطة gateway حديثة مخزنة مؤقتًا.
- يفرض `health --verbose` probe مباشرًا ويوسع الإخراج المقروء بشريًا عبر جميع الحسابات والوكلاء المكوّنين.

### `sessions`

إدراج جلسات المحادثة المخزنة.

الخيارات:

- `--json`
- `--verbose`
- `--store <path>`
- `--active <minutes>`
- `--agent <id>` (تصفية الجلسات حسب الوكيل)
- `--all-agents` (إظهار الجلسات عبر جميع الوكلاء)

الأوامر الفرعية:

- `sessions cleanup` — إزالة الجلسات المنتهية أو اليتيمة

ملاحظات:

- يدعم `sessions cleanup` أيضًا `--fix-missing` لتقليص الإدخالات التي اختفت ملفات transcript الخاصة بها.

## إعادة الضبط / إلغاء التثبيت

### `reset`

إعادة ضبط التكوين/الحالة المحلية (مع إبقاء CLI مثبتًا).

الخيارات:

- `--scope <config|config+creds+sessions|full>`
- `--yes`
- `--non-interactive`
- `--dry-run`

ملاحظات:

- يتطلب `--non-interactive` كلاً من `--scope` و`--yes`.

### `uninstall`

إلغاء تثبيت خدمة gateway + البيانات المحلية (ويبقى CLI).

الخيارات:

- `--service`
- `--state`
- `--workspace`
- `--app`
- `--all`
- `--yes`
- `--non-interactive`
- `--dry-run`

ملاحظات:

- يتطلب `--non-interactive` استخدام `--yes` ونطاقات صريحة (أو `--all`).
- يزيل `--all` الخدمة والحالة ومساحة العمل والتطبيق معًا.

### `tasks`

إدراج وإدارة عمليات تشغيل [المهام الخلفية](/automation/tasks) عبر الوكلاء.

- `tasks list` — عرض عمليات تشغيل المهام النشطة والأخيرة
- `tasks show <id>` — عرض تفاصيل عملية تشغيل مهمة محددة
- `tasks notify <id>` — تغيير سياسة الإشعارات لعملية تشغيل مهمة
- `tasks cancel <id>` — إلغاء مهمة قيد التشغيل
- `tasks audit` — إظهار المشكلات التشغيلية (القديمة، والمفقودة، وإخفاقات التسليم)
- `tasks maintenance [--apply] [--json]` — معاينة أو تطبيق تنظيف/تسوية المهام وTaskFlow (جلسات ACP/الوكلاء الفرعيين الأبناء، ومهام cron النشطة، وتشغيلات CLI المباشرة)
- `tasks flow list` — إدراج تدفقات Task Flow النشطة والأخيرة
- `tasks flow show <lookup>` — فحص تدفق حسب المعرف أو مفتاح lookup
- `tasks flow cancel <lookup>` — إلغاء تدفق قيد التشغيل ومهامه النشطة

### `flows`

اختصار توثيقي قديم. توجد أوامر التدفق تحت `openclaw tasks flow`:

- `tasks flow list [--json]`
- `tasks flow show <lookup>`
- `tasks flow cancel <lookup>`

## Gateway

### `gateway`

تشغيل WebSocket Gateway.

الخيارات:

- `--port <port>`
- `--bind <loopback|tailnet|lan|auto|custom>`
- `--token <token>`
- `--auth <token|password>`
- `--password <password>`
- `--password-file <path>`
- `--tailscale <off|serve|funnel>`
- `--tailscale-reset-on-exit`
- `--allow-unconfigured`
- `--dev`
- `--reset` (إعادة ضبط تكوين dev + بيانات الاعتماد + الجلسات + مساحة العمل)
- `--force` (قتل المستمع الموجود على المنفذ)
- `--verbose`
- `--cli-backend-logs`
- `--claude-cli-logs` (اسم مستعار متروك)
- `--ws-log <auto|full|compact>`
- `--compact` (اسم مستعار لـ `--ws-log compact`)
- `--raw-stream`
- `--raw-stream-path <path>`

### `gateway service`

إدارة خدمة Gateway ‏(launchd/systemd/schtasks).

الأوامر الفرعية:

- `gateway status` (يفحص Gateway RPC افتراضيًا)
- `gateway install` (تثبيت الخدمة)
- `gateway uninstall`
- `gateway start`
- `gateway stop`
- `gateway restart`

ملاحظات:

- يفحص `gateway status` Gateway RPC افتراضيًا باستخدام المنفذ/التكوين المحلول الخاص بالخدمة (يمكن التجاوز باستخدام `--url/--token/--password`).
- يدعم `gateway status` الخيارات `--no-probe` و`--deep` و`--require-rpc` و`--json` للبرمجة النصية.
- يعرض `gateway status` أيضًا خدمات gateway القديمة أو الإضافية عندما يستطيع اكتشافها (`--deep` يضيف فحوصات على مستوى النظام). وتُعامل خدمات OpenClaw المسماة حسب الملف الشخصي على أنها من الدرجة الأولى ولا تُعلَّم على أنها "إضافية".
- يظل `gateway status` متاحًا للتشخيص حتى عندما يكون تكوين CLI المحلي مفقودًا أو غير صالح.
- يطبع `gateway status` مسار سجل الملف المحلول، ولقطة صلاحية/مسارات التكوين لـ CLI مقابل الخدمة، وعنوان URL الخاص بالتحقق المحلول.
- إذا كانت Gateway auth SecretRefs غير محلولة في مسار الأمر الحالي، فإن `gateway status --json` يبلّغ عن `rpc.authWarning` فقط عندما يفشل probe في الاتصال/المصادقة (وتُخفى التحذيرات عند نجاح probe).
- في تثبيتات systemd على Linux، تتضمن فحوصات انحراف الرمز المميز في الحالة كلا من مصدري الوحدة `Environment=` و`EnvironmentFile=`.
- تدعم `gateway install|uninstall|start|stop|restart` الخيار `--json` للبرمجة النصية (ويبقى الإخراج الافتراضي سهل القراءة).
- يستخدم `gateway install` وقت تشغيل Node افتراضيًا؛ ولا يُنصح بـ bun **(أخطاء WhatsApp/Telegram)**.
- خيارات `gateway install`: ‏`--port`, `--runtime`, `--token`, `--force`, `--json`.

### `daemon`

اسم مستعار قديم لأوامر إدارة خدمة Gateway. راجع [/cli/daemon](/cli/daemon).

الأوامر الفرعية:

- `daemon status`
- `daemon install`
- `daemon uninstall`
- `daemon start`
- `daemon stop`
- `daemon restart`

الخيارات الشائعة:

- `status`: ‏`--url`, `--token`, `--password`, `--timeout`, `--no-probe`, `--require-rpc`, `--deep`, `--json`
- `install`: ‏`--port`, `--runtime <node|bun>`, `--token`, `--force`, `--json`
- `uninstall|start|stop|restart`: ‏`--json`

### `logs`

تتبّع سجلات ملفات Gateway عبر RPC.

الخيارات:

- `--limit <n>`: الحد الأقصى لعدد أسطر السجل التي ستُعاد
- `--max-bytes <n>`: الحد الأقصى للبايتات التي ستُقرأ من ملف السجل
- `--follow`: متابعة ملف السجل (على نمط `tail -f`)
- `--interval <ms>`: فترة الاستقصاء بالمللي ثانية أثناء المتابعة
- `--local-time`: عرض الطوابع الزمنية بالتوقيت المحلي
- `--json`: إصدار JSON بسطر لكل إدخال
- `--plain`: تعطيل التنسيق البنيوي
- `--no-color`: تعطيل ألوان ANSI
- `--url <url>`: عنوان WebSocket صريح لـ Gateway
- `--token <token>`: رمز Gateway
- `--timeout <ms>`: مهلة Gateway RPC
- `--expect-final`: انتظار استجابة نهائية عند الحاجة

أمثلة:

```bash
openclaw logs --follow
openclaw logs --limit 200
openclaw logs --plain
openclaw logs --json
openclaw logs --no-color
```

ملاحظات:

- إذا مررت `--url`، فلن يطبّق CLI بيانات اعتماد التكوين أو البيئة تلقائيًا.
- تعود إخفاقات pairing في local loopback إلى ملف السجل المحلي المكوَّن؛ ولا ينطبق ذلك على الأهداف الصريحة عبر `--url`.

### `gateway <subcommand>`

مساعدات CLI لـ Gateway (استخدم `--url`, `--token`, `--password`, `--timeout`, `--expect-final` مع أوامر RPC الفرعية).
عندما تمرر `--url`، لا يطبّق CLI بيانات اعتماد التكوين أو البيئة تلقائيًا.
ضمّن `--token` أو `--password` صراحةً. ويُعد غياب بيانات الاعتماد الصريحة خطأ.

الأوامر الفرعية:

- `gateway call <method> [--params <json>] [--url <url>] [--token <token>] [--password <password>] [--timeout <ms>] [--expect-final] [--json]`
- `gateway health`
- `gateway status`
- `gateway probe`
- `gateway discover`
- `gateway install|uninstall|start|stop|restart`
- `gateway run`

ملاحظات:

- يضيف `gateway status --deep` فحص خدمة على مستوى النظام. استخدم `gateway probe`،
  أو `health --verbose`، أو `status --deep` من المستوى الأعلى للحصول على تفاصيل أعمق عن probes وقت التشغيل.

عمليات RPC الشائعة:

- `config.schema.lookup` (فحص شجرة فرعية واحدة من التكوين مع عقدة schema سطحية، وبيانات hints المطابقة، وملخصات الأبناء المباشرين)
- `config.get` (قراءة لقطة التكوين الحالية + hash)
- `config.set` (التحقق + كتابة التكوين الكامل؛ استخدم `baseHash` للتزامن التفاؤلي)
- `config.apply` (التحقق + كتابة التكوين + إعادة التشغيل + الإيقاظ)
- `config.patch` (دمج تحديث جزئي + إعادة التشغيل + الإيقاظ)
- `update.run` (تشغيل التحديث + إعادة التشغيل + الإيقاظ)

نصيحة: عند استدعاء `config.set`/`config.apply`/`config.patch` مباشرة، مرّر `baseHash` من
`config.get` إذا كان هناك تكوين موجود بالفعل.
نصيحة: بالنسبة إلى التعديلات الجزئية، افحص أولًا باستخدام `config.schema.lookup` وفضّل `config.patch`.
نصيحة: تجري عمليات RPC الخاصة بكتابة التكوين هذه فحصًا مسبقًا لحل SecretRef النشط للمراجع الموجودة في حمولة التكوين المقدمة وترفض الكتابات عندما يكون مرجع مقدّم فعّال عمليًا غير محلول.
نصيحة: ما تزال أداة وقت التشغيل `gateway` الخاصة بالمالك فقط ترفض إعادة كتابة `tools.exec.ask` أو `tools.exec.security`؛ وتُطبَّع الأسماء المستعارة القديمة `tools.bash.*` إلى مسارات exec المحمية نفسها.

## النماذج

راجع [/concepts/models](/concepts/models) لسلوك fallback واستراتيجية الفحص.

ملاحظة الفوترة: نعتقد أن fallback الخاص بـ Claude Code CLI مسموح به على الأرجح
للأتمتة المحلية التي يديرها المستخدم استنادًا إلى وثائق CLI العامة من Anthropic.
ومع ذلك، فإن سياسة Anthropic الخاصة بأطر الطرف الثالث تُحدث قدرًا كافيًا من
الغموض حول الاستخدام المدعوم بالاشتراك في المنتجات الخارجية بحيث لا
نوصي به للإنتاج. كما أبلغت Anthropic مستخدمي OpenClaw في **4 أبريل 2026 الساعة
12:00 ظهرًا PT / 8:00 مساءً BST** أن مسار تسجيل الدخول إلى Claude في **OpenClaw**
يُعد استخدامًا لإطار طرف ثالث ويتطلب **Extra Usage** تُحتسب بشكل منفصل عن
الاشتراك. وللإنتاج، فضّل مفتاح Anthropic API أو مزودًا آخر مدعومًا بنمط
الاشتراك مثل OpenAI Codex أو Alibaba Cloud Model Studio
Coding Plan أو MiniMax Coding Plan أو Z.AI / GLM Coding Plan.

ترحيل Anthropic Claude CLI:

```bash
openclaw models auth login --provider anthropic --method cli --set-default
```

اختصار onboarding: ‏`openclaw onboard --auth-choice anthropic-cli`

أصبح Anthropic setup-token متاحًا مرة أخرى كمسار مصادقة قديم/يدوي.
استخدمه فقط مع توقع أن Anthropic أبلغت مستخدمي OpenClaw أن مسار
تسجيل الدخول إلى Claude في OpenClaw يتطلب **Extra Usage**.

ملاحظة الاسم المستعار القديم: ‏`claude-cli` هو الاسم المستعار المتقادم لـ onboarding auth-choice.
استخدم `anthropic-cli` للتهيئة الأولية، أو استخدم `models auth login` مباشرة.

### `models` (الجذر)

`openclaw models` هو اسم مستعار لـ `models status`.

خيارات الجذر:

- `--status-json` (اسم مستعار لـ `models status --json`)
- `--status-plain` (اسم مستعار لـ `models status --plain`)

### `models list`

الخيارات:

- `--all`
- `--local`
- `--provider <name>`
- `--json`
- `--plain`

### `models status`

الخيارات:

- `--json`
- `--plain`
- `--check` (رمز خروج 1=منتهي/مفقود، 2=قريب الانتهاء)
- `--probe` (probe مباشر لملفات تعريف المصادقة المكوّنة)
- `--probe-provider <name>`
- `--probe-profile <id>` (قابل للتكرار أو مفصول بفواصل)
- `--probe-timeout <ms>`
- `--probe-concurrency <n>`
- `--probe-max-tokens <n>`
- `--agent <id>`

يتضمن دائمًا نظرة عامة على المصادقة وحالة انتهاء OAuth لملفات التعريف الموجودة في مخزن المصادقة.
يشغّل `--probe` طلبات مباشرة (وقد يستهلك رموزًا ويُطلق حدود المعدل).
يمكن أن تأتي صفوف probe من ملفات تعريف المصادقة أو بيانات اعتماد env أو `models.json`.
توقّع حالات probe مثل `ok` و`auth` و`rate_limit` و`billing` و`timeout`،
و`format` و`unknown` و`no_model`.
عندما يحذف `auth.order.<provider>` الصريح ملف تعريف مخزنًا، يبلّغ probe
عن `excluded_by_auth_order` بدلًا من تجربته بصمت.

### `models set <model>`

تعيين `agents.defaults.model.primary`.

### `models set-image <model>`

تعيين `agents.defaults.imageModel.primary`.

### `models aliases list|add|remove`

الخيارات:

- `list`: ‏`--json`, `--plain`
- `add <alias> <model>`
- `remove <alias>`

### `models fallbacks list|add|remove|clear`

الخيارات:

- `list`: ‏`--json`, `--plain`
- `add <model>`
- `remove <model>`
- `clear`

### `models image-fallbacks list|add|remove|clear`

الخيارات:

- `list`: ‏`--json`, `--plain`
- `add <model>`
- `remove <model>`
- `clear`

### `models scan`

الخيارات:

- `--min-params <b>`
- `--max-age-days <days>`
- `--provider <name>`
- `--max-candidates <n>`
- `--timeout <ms>`
- `--concurrency <n>`
- `--no-probe`
- `--yes`
- `--no-input`
- `--set-default`
- `--set-image`
- `--json`

### `models auth add|login|login-github-copilot|setup-token|paste-token`

الخيارات:

- `add`: مساعد مصادقة تفاعلي (تدفق مصادقة المزوّد أو لصق الرمز)
- `login`: ‏`--provider <name>`, `--method <method>`, `--set-default`
- `login-github-copilot`: تدفق تسجيل دخول GitHub Copilot OAuth ‏(`--yes`)
- `setup-token`: ‏`--provider <name>`, `--yes`
- `paste-token`: ‏`--provider <name>`, `--profile-id <id>`, `--expires-in <duration>`

ملاحظات:

- `setup-token` و`paste-token` هما أوامر رموز عامة للمزوّدين الذين يوفّرون طرق مصادقة بالرمز.
- يتطلب `setup-token` طرفية تفاعلية TTY ويشغّل طريقة مصادقة الرمز الخاصة بالمزوّد.
- يطلب `paste-token` قيمة الرمز ويستخدم افتراضيًا معرّف ملف تعريف المصادقة `<provider>:manual` عندما لا يتم تمرير `--profile-id`.
- أصبح `setup-token` / `paste-token` الخاص بـ Anthropic متاحين مرة أخرى كمسار OpenClaw قديم/يدوي. وقد أبلغت Anthropic مستخدمي OpenClaw أن هذا المسار يتطلب **Extra Usage** على حساب Claude.

### `models auth order get|set|clear`

الخيارات:

- `get`: ‏`--provider <name>`, `--agent <id>`, `--json`
- `set`: ‏`--provider <name>`, `--agent <id>`, `<profileIds...>`
- `clear`: ‏`--provider <name>`, `--agent <id>`

## النظام

### `system event`

إدراج حدث نظام في قائمة الانتظار وتشغيل heartbeat اختياريًا (Gateway RPC).

مطلوب:

- `--text <text>`

الخيارات:

- `--mode <now|next-heartbeat>`
- `--json`
- `--url`, `--token`, `--timeout`, `--expect-final`

### `system heartbeat last|enable|disable`

عناصر التحكم في heartbeat ‏(Gateway RPC).

الخيارات:

- `--json`
- `--url`, `--token`, `--timeout`, `--expect-final`

### `system presence`

إدراج إدخالات system presence ‏(Gateway RPC).

الخيارات:

- `--json`
- `--url`, `--token`, `--timeout`, `--expect-final`

## Cron

إدارة المهام المجدولة (Gateway RPC). راجع [/automation/cron-jobs](/automation/cron-jobs).

الأوامر الفرعية:

- `cron status [--json]`
- `cron list [--all] [--json]` (إخراج جدولي افتراضيًا؛ استخدم `--json` للخام)
- `cron add` (اسم مستعار: `create`؛ يتطلب `--name` وواحدًا بالضبط من `--at` | `--every` | `--cron`، وحمولة واحدة بالضبط من `--system-event` | `--message`)
- `cron edit <id>` (ترقيع الحقول)
- `cron rm <id>` (أسماء مستعارة: `remove`, `delete`)
- `cron enable <id>`
- `cron disable <id>`
- `cron runs --id <id> [--limit <n>]`
- `cron run <id> [--due]`

تقبل جميع أوامر `cron` الخيارات `--url`, `--token`, `--timeout`, `--expect-final`.

يستخدم `cron add|edit --model ...` ذلك النموذج المسموح المحدد للمهمة. وإذا
لم يكن النموذج مسموحًا به، يحذر cron ويعود إلى اختيار نموذج
الوكيل/الافتراضي الخاص بالمهمة بدلًا من ذلك. وتظل سلاسل fallback
المكوّنة مطبقة، لكن تجاوز النموذج العادي بدون قائمة fallback صريحة لكل مهمة لم يعد
يضيف النموذج الأساسي للوكيل كهدف إعادة محاولة إضافي مخفي.

## مضيف العقدة

### `node`

يشغّل `node` **مضيف عقدة بدون واجهة** أو يديره كخدمة في الخلفية. راجع
[`openclaw node`](/cli/node).

الأوامر الفرعية:

- `node run --host <gateway-host> --port 18789`
- `node status`
- `node install [--host <gateway-host>] [--port <port>] [--tls] [--tls-fingerprint <sha256>] [--node-id <id>] [--display-name <name>] [--runtime <node|bun>] [--force]`
- `node uninstall`
- `node stop`
- `node restart`

ملاحظات المصادقة:

- يحل `node` مصادقة gateway من env/config (من دون علامات `--token`/`--password`): ‏`OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD`، ثم `gateway.auth.*`. وفي الوضع المحلي، يتجاهل مضيف العقدة عمدًا `gateway.remote.*`؛ وفي `gateway.mode=remote` يشارك `gateway.remote.*` وفق قواعد أولوية الوضع البعيد.
- لا يحترم حل مصادقة node-host إلا متغيرات البيئة `OPENCLAW_GATEWAY_*`.

## Nodes

يتحدث `nodes` إلى Gateway ويستهدف العقد المقترنة. راجع [/nodes](/nodes).

الخيارات الشائعة:

- `--url`, `--token`, `--timeout`, `--json`

الأوامر الفرعية:

- `nodes status [--connected] [--last-connected <duration>]`
- `nodes describe --node <id|name|ip>`
- `nodes list [--connected] [--last-connected <duration>]`
- `nodes pending`
- `nodes approve <requestId>`
- `nodes reject <requestId>`
- `nodes rename --node <id|name|ip> --name <displayName>`
- `nodes invoke --node <id|name|ip> --command <command> [--params <json>] [--invoke-timeout <ms>] [--idempotency-key <key>]`
- `nodes notify --node <id|name|ip> [--title <text>] [--body <text>] [--sound <name>] [--priority <passive|active|timeSensitive>] [--delivery <system|overlay|auto>] [--invoke-timeout <ms>]` (mac فقط)

الكاميرا:

- `nodes camera list --node <id|name|ip>`
- `nodes camera snap --node <id|name|ip> [--facing front|back|both] [--device-id <id>] [--max-width <px>] [--quality <0-1>] [--delay-ms <ms>] [--invoke-timeout <ms>]`
- `nodes camera clip --node <id|name|ip> [--facing front|back] [--device-id <id>] [--duration <ms|10s|1m>] [--no-audio] [--invoke-timeout <ms>]`

Canvas + الشاشة:

- `nodes canvas snapshot --node <id|name|ip> [--format png|jpg|jpeg] [--max-width <px>] [--quality <0-1>] [--invoke-timeout <ms>]`
- `nodes canvas present --node <id|name|ip> [--target <urlOrPath>] [--x <px>] [--y <px>] [--width <px>] [--height <px>] [--invoke-timeout <ms>]`
- `nodes canvas hide --node <id|name|ip> [--invoke-timeout <ms>]`
- `nodes canvas navigate <url> --node <id|name|ip> [--invoke-timeout <ms>]`
- `nodes canvas eval [<js>] --node <id|name|ip> [--js <code>] [--invoke-timeout <ms>]`
- `nodes canvas a2ui push --node <id|name|ip> (--jsonl <path> | --text <text>) [--invoke-timeout <ms>]`
- `nodes canvas a2ui reset --node <id|name|ip> [--invoke-timeout <ms>]`
- `nodes screen record --node <id|name|ip> [--screen <index>] [--duration <ms|10s>] [--fps <n>] [--no-audio] [--out <path>] [--invoke-timeout <ms>]`

الموقع:

- `nodes location get --node <id|name|ip> [--max-age <ms>] [--accuracy <coarse|balanced|precise>] [--location-timeout <ms>] [--invoke-timeout <ms>]`

## Browser

CLI للتحكم في Browser ‏(Chrome/Brave/Edge/Chromium مخصص). راجع [`openclaw browser`](/cli/browser) و[أداة Browser](/tools/browser).

الخيارات الشائعة:

- `--url`, `--token`, `--timeout`, `--expect-final`, `--json`
- `--browser-profile <name>`

الإدارة:

- `browser status`
- `browser start`
- `browser stop`
- `browser reset-profile`
- `browser tabs`
- `browser open <url>`
- `browser focus <targetId>`
- `browser close [targetId]`
- `browser profiles`
- `browser create-profile --name <name> [--color <hex>] [--cdp-url <url>] [--driver existing-session] [--user-data-dir <path>]`
- `browser delete-profile --name <name>`

الفحص:

- `browser screenshot [targetId] [--full-page] [--ref <ref>] [--element <selector>] [--type png|jpeg]`
- `browser snapshot [--format aria|ai] [--target-id <id>] [--limit <n>] [--interactive] [--compact] [--depth <n>] [--selector <sel>] [--out <path>]`

الإجراءات:

- `browser navigate <url> [--target-id <id>]`
- `browser resize <width> <height> [--target-id <id>]`
- `browser click <ref> [--double] [--button <left|right|middle>] [--modifiers <csv>] [--target-id <id>]`
- `browser type <ref> <text> [--submit] [--slowly] [--target-id <id>]`
- `browser press <key> [--target-id <id>]`
- `browser hover <ref> [--target-id <id>]`
- `browser drag <startRef> <endRef> [--target-id <id>]`
- `browser select <ref> <values...> [--target-id <id>]`
- `browser upload <paths...> [--ref <ref>] [--input-ref <ref>] [--element <selector>] [--target-id <id>] [--timeout-ms <ms>]`
- `browser fill [--fields <json>] [--fields-file <path>] [--target-id <id>]`
- `browser dialog --accept|--dismiss [--prompt <text>] [--target-id <id>] [--timeout-ms <ms>]`
- `browser wait [--time <ms>] [--text <value>] [--text-gone <value>] [--target-id <id>]`
- `browser evaluate --fn <code> [--ref <ref>] [--target-id <id>]`
- `browser console [--level <error|warn|info>] [--target-id <id>]`
- `browser pdf [--target-id <id>]`

## Voice call

### `voicecall`

أدوات voice-call مقدمة من plugin. لا تظهر إلا عندما يكون plugin الخاص بالمكالمات الصوتية مثبتًا ومفعّلًا. راجع [`openclaw voicecall`](/cli/voicecall).

الأوامر الشائعة:

- `voicecall call --to <phone> --message <text> [--mode notify|conversation]`
- `voicecall start --to <phone> [--message <text>] [--mode notify|conversation]`
- `voicecall continue --call-id <id> --message <text>`
- `voicecall speak --call-id <id> --message <text>`
- `voicecall end --call-id <id>`
- `voicecall status --call-id <id>`
- `voicecall tail [--file <path>] [--since <n>] [--poll <ms>]`
- `voicecall latency [--file <path>] [--last <n>]`
- `voicecall expose [--mode off|serve|funnel] [--path <path>] [--port <port>] [--serve-path <path>]`

## البحث في الوثائق

### `docs`

ابحث في فهرس وثائق OpenClaw المباشر.

### `docs [query...]`

ابحث في فهرس الوثائق المباشر.

## TUI

### `tui`

افتح واجهة الطرفية المتصلة بـ Gateway.

الخيارات:

- `--url <url>`
- `--token <token>`
- `--password <password>`
- `--session <key>`
- `--deliver`
- `--thinking <level>`
- `--message <text>`
- `--timeout-ms <ms>` (الافتراضي `agents.defaults.timeoutSeconds`)
- `--history-limit <n>`
