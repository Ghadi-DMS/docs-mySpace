---
read_when:
    - Ви хочете контейнеризований gateway замість локальних встановлень
    - Ви перевіряєте Docker-потік
summary: Необов’язкове налаштування та онбординг OpenClaw на основі Docker
title: Docker
x-i18n:
    generated_at: "2026-04-05T21:33:22Z"
    model: gpt-5.4
    provider: openai
    source_hash: d6aa0453340d7683b4954316274ba6dd1aa7c0ce2483e9bd8ae137ff4efd4c3c
    source_path: install/docker.md
    workflow: 15
---

# Docker (необов’язково)

Docker **необов’язковий**. Використовуйте його лише якщо хочете контейнеризований gateway або перевіряєте Docker-потік.

## Чи підходить мені Docker?

- **Так**: вам потрібне ізольоване, тимчасове середовище gateway або ви хочете запускати OpenClaw на хості без локальних встановлень.
- **Ні**: ви запускаєте на власній машині й просто хочете найшвидший цикл розробки. Натомість використовуйте звичайний процес встановлення.
- **Примітка щодо sandboxing**: ізоляція агентів також використовує Docker, але для цього **не** потрібно запускати весь gateway у Docker. Див. [Sandboxing](/uk/gateway/sandboxing).

## Передумови

- Docker Desktop (або Docker Engine) + Docker Compose v2
- Щонайменше 2 ГБ RAM для збирання образу (`pnpm install` може бути завершено через OOM на хостах із 1 ГБ з кодом виходу 137)
- Достатньо місця на диску для образів і журналів
- Якщо запускаєте на VPS/публічному хості, перегляньте
  [Посилення безпеки для мережевої доступності](/uk/gateway/security),
  особливо політику фаєрвола Docker `DOCKER-USER`.

## Контейнеризований Gateway

<Steps>
  <Step title="Зберіть образ">
    У корені репозиторію виконайте скрипт налаштування:

    ```bash
    ./scripts/docker/setup.sh
    ```

    Це локально збирає образ gateway. Щоб натомість використати попередньо зібраний образ:

    ```bash
    export OPENCLAW_IMAGE="ghcr.io/openclaw/openclaw:latest"
    ./scripts/docker/setup.sh
    ```

    Попередньо зібрані образи публікуються в
    [GitHub Container Registry](https://github.com/openclaw/openclaw/pkgs/container/openclaw).
    Поширені теги: `main`, `latest`, `<version>` (наприклад, `2026.2.26`).

  </Step>

  <Step title="Завершіть онбординг">
    Скрипт налаштування автоматично запускає онбординг. Він:

    - запросить API-ключі провайдера
    - згенерує токен gateway і запише його в `.env`
    - запустить gateway через Docker Compose

    Під час налаштування онбординг перед запуском і запис конфігурації виконуються
    безпосередньо через `openclaw-gateway`. `openclaw-cli` призначений для команд,
    які ви запускаєте після того, як контейнер gateway уже існує.

  </Step>

  <Step title="Відкрийте Control UI">
    Відкрийте `http://127.0.0.1:18789/` у браузері та вставте налаштований
    спільний секрет у Settings. Скрипт налаштування типово записує токен у `.env`;
    якщо ви перемкнете конфігурацію контейнера на автентифікацію за паролем,
    використовуйте натомість цей пароль.

    Потрібна URL-адреса знову?

    ```bash
    docker compose run --rm openclaw-cli dashboard --no-open
    ```

  </Step>

  <Step title="Налаштуйте канали (необов’язково)">
    Використайте контейнер CLI, щоб додати канали обміну повідомленнями:

    ```bash
    # WhatsApp (QR)
    docker compose run --rm openclaw-cli channels login

    # Telegram
    docker compose run --rm openclaw-cli channels add --channel telegram --token "<token>"

    # Discord
    docker compose run --rm openclaw-cli channels add --channel discord --token "<token>"
    ```

    Документація: [WhatsApp](/uk/channels/whatsapp), [Telegram](/uk/channels/telegram), [Discord](/uk/channels/discord)

  </Step>
</Steps>

### Ручний процес

Якщо ви віддаєте перевагу виконанню кожного кроку вручну замість використання скрипта налаштування:

```bash
docker build -t openclaw:local -f Dockerfile .
docker compose run --rm --no-deps --entrypoint node openclaw-gateway \
  dist/index.js onboard --mode local --no-install-daemon
docker compose run --rm --no-deps --entrypoint node openclaw-gateway \
  dist/index.js config set --batch-json '[{"path":"gateway.mode","value":"local"},{"path":"gateway.bind","value":"lan"},{"path":"gateway.controlUi.allowedOrigins","value":["http://localhost:18789","http://127.0.0.1:18789"]}]'
docker compose up -d openclaw-gateway
```

<Note>
Запускайте `docker compose` з кореня репозиторію. Якщо ви ввімкнули `OPENCLAW_EXTRA_MOUNTS`
або `OPENCLAW_HOME_VOLUME`, скрипт налаштування записує `docker-compose.extra.yml`;
додавайте його через `-f docker-compose.yml -f docker-compose.extra.yml`.
</Note>

<Note>
Оскільки `openclaw-cli` використовує спільний мережевий простір імен з `openclaw-gateway`,
це інструмент для використання після запуску. До `docker compose up -d openclaw-gateway`
виконуйте онбординг і запис конфігурації під час налаштування через `openclaw-gateway` з
`--no-deps --entrypoint node`.
</Note>

### Змінні середовища

Скрипт налаштування приймає такі необов’язкові змінні середовища:

| Змінна                        | Призначення                                                      |
| ----------------------------- | ---------------------------------------------------------------- |
| `OPENCLAW_IMAGE`              | Використовувати віддалений образ замість локального збирання     |
| `OPENCLAW_DOCKER_APT_PACKAGES` | Встановити додаткові apt-пакети під час збирання (через пробіл) |
| `OPENCLAW_EXTENSIONS`         | Попередньо встановити залежності розширень під час збирання (назви через пробіл) |
| `OPENCLAW_EXTRA_MOUNTS`       | Додаткові bind mounts хоста (через кому `source:target[:opts]`)  |
| `OPENCLAW_HOME_VOLUME`        | Зберігати `/home/node` в іменованому Docker volume               |
| `OPENCLAW_SANDBOX`            | Увімкнути bootstrap sandbox (`1`, `true`, `yes`, `on`)           |
| `OPENCLAW_DOCKER_SOCKET`      | Перевизначити шлях до Docker socket                              |

### Перевірки стану

Кінцеві точки перевірки контейнера (автентифікація не потрібна):

```bash
curl -fsS http://127.0.0.1:18789/healthz   # liveness
curl -fsS http://127.0.0.1:18789/readyz     # readiness
```

Образ Docker містить вбудований `HEALTHCHECK`, який опитує `/healthz`.
Якщо перевірки продовжують завершуватися невдало, Docker позначає контейнер як `unhealthy`,
а системи оркестрації можуть перезапустити або замінити його.

Автентифікований детальний знімок стану:

```bash
docker compose exec openclaw-gateway node dist/index.js health --token "$OPENCLAW_GATEWAY_TOKEN"
```

### LAN проти loopback

`scripts/docker/setup.sh` типово встановлює `OPENCLAW_GATEWAY_BIND=lan`, щоб доступ із хоста до
`http://127.0.0.1:18789` працював із публікацією порту Docker.

- `lan` (типово): браузер і CLI на хості можуть звертатися до опублікованого порту gateway.
- `loopback`: напряму до gateway можуть звертатися лише процеси всередині мережевого простору імен контейнера.

<Note>
Використовуйте значення режиму прив’язки в `gateway.bind` (`lan` / `loopback` / `custom` /
`tailnet` / `auto`), а не псевдоніми хоста на кшталт `0.0.0.0` або `127.0.0.1`.
</Note>

### Сховище та збереження даних

Docker Compose монтує `OPENCLAW_CONFIG_DIR` до `/home/node/.openclaw` і
`OPENCLAW_WORKSPACE_DIR` до `/home/node/.openclaw/workspace`, тож ці шляхи
зберігаються після заміни контейнера.

У цьому змонтованому каталозі конфігурації OpenClaw зберігає:

- `openclaw.json` для конфігурації поведінки
- `agents/<agentId>/agent/auth-profiles.json` для збереженої OAuth/API-key автентифікації провайдерів
- `.env` для секретів середовища виконання на основі env, таких як `OPENCLAW_GATEWAY_TOKEN`

Повні відомості про збереження даних у VM-розгортаннях див. у
[Docker VM Runtime - Що і де зберігається](/uk/install/docker-vm-runtime#what-persists-where).

**Точки зростання дискового простору:** стежте за `media/`, файлами session JSONL, `cron/runs/*.jsonl`
і ротаційними файловими журналами в `/tmp/openclaw/`.

### Допоміжні команди оболонки (необов’язково)

Для зручнішого щоденного керування Docker установіть `ClawDock`:

```bash
mkdir -p ~/.clawdock && curl -sL https://raw.githubusercontent.com/openclaw/openclaw/main/scripts/clawdock/clawdock-helpers.sh -o ~/.clawdock/clawdock-helpers.sh
echo 'source ~/.clawdock/clawdock-helpers.sh' >> ~/.zshrc && source ~/.zshrc
```

Якщо ви встановили ClawDock зі старого raw-шляху `scripts/shell-helpers/clawdock-helpers.sh`, повторно виконайте наведену вище команду встановлення, щоб ваш локальний файл helpers відстежував нове розташування.

Потім використовуйте `clawdock-start`, `clawdock-stop`, `clawdock-dashboard` тощо. Виконайте
`clawdock-help`, щоб побачити всі команди.
Див. [ClawDock](/uk/install/clawdock), щоб переглянути повний посібник із helpers.

<AccordionGroup>
  <Accordion title="Увімкнення sandbox агента для Docker gateway">
    ```bash
    export OPENCLAW_SANDBOX=1
    ./scripts/docker/setup.sh
    ```

    Нестандартний шлях до socket (наприклад, rootless Docker):

    ```bash
    export OPENCLAW_SANDBOX=1
    export OPENCLAW_DOCKER_SOCKET=/run/user/1000/docker.sock
    ./scripts/docker/setup.sh
    ```

    Скрипт монтує `docker.sock` лише після успішного проходження передумов sandbox. Якщо
    налаштування sandbox не вдається завершити, скрипт скидає `agents.defaults.sandbox.mode`
    до `off`.

  </Accordion>

  <Accordion title="Автоматизація / CI (неінтерактивно)">
    Вимкніть виділення pseudo-TTY для Compose за допомогою `-T`:

    ```bash
    docker compose run -T --rm openclaw-cli gateway probe
    docker compose run -T --rm openclaw-cli devices list --json
    ```

  </Accordion>

  <Accordion title="Примітка щодо безпеки спільної мережі">
    `openclaw-cli` використовує `network_mode: "service:openclaw-gateway"`, тому команди CLI
    можуть звертатися до gateway через `127.0.0.1`. Розглядайте це як спільну
    межу довіри. Конфігурація compose прибирає `NET_RAW`/`NET_ADMIN` та вмикає
    `no-new-privileges` для `openclaw-cli`.
  </Accordion>

  <Accordion title="Права доступу та EACCES">
    Образ працює від імені `node` (uid 1000). Якщо ви бачите помилки прав доступу до
    `/home/node/.openclaw`, переконайтеся, що ваші bind mounts хоста належать uid 1000:

    ```bash
    sudo chown -R 1000:1000 /path/to/openclaw-config /path/to/openclaw-workspace
    ```

  </Accordion>

  <Accordion title="Швидші перебудови">
    Упорядкуйте Dockerfile так, щоб шари залежностей кешувалися. Це дозволяє не запускати
    `pnpm install` повторно, доки не зміняться lockfile-файли:

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

  <Accordion title="Параметри контейнера для досвідчених користувачів">
    Типовий образ орієнтований на безпеку та працює від непривілейованого користувача `node`. Для більш
    функціонального контейнера:

    1. **Зберігайте `/home/node`**: `export OPENCLAW_HOME_VOLUME="openclaw_home"`
    2. **Вбудуйте системні залежності**: `export OPENCLAW_DOCKER_APT_PACKAGES="git curl jq"`
    3. **Установіть браузери Playwright**:
       ```bash
       docker compose run --rm openclaw-cli \
         node /app/node_modules/playwright-core/cli.js install chromium
       ```
    4. **Зберігайте завантаження браузера**: установіть
       `PLAYWRIGHT_BROWSERS_PATH=/home/node/.cache/ms-playwright` і використовуйте
       `OPENCLAW_HOME_VOLUME` або `OPENCLAW_EXTRA_MOUNTS`.

  </Accordion>

  <Accordion title="OpenAI Codex OAuth (headless Docker)">
    Якщо ви виберете у майстрі OpenAI Codex OAuth, відкриється URL-адреса в браузері. У
    Docker або headless-середовищах скопіюйте повну URL-адресу перенаправлення, на яку ви потрапите, і вставте
    її назад у майстер, щоб завершити автентифікацію.
  </Accordion>

  <Accordion title="Метадані базового образу">
    Основний образ Docker використовує `node:24-bookworm` і публікує анотації базового OCI-образу,
    зокрема `org.opencontainers.image.base.name`,
    `org.opencontainers.image.source` та інші. Див.
    [Анотації OCI-образу](https://github.com/opencontainers/image-spec/blob/main/annotations.md).
  </Accordion>
</AccordionGroup>

### Запуск на VPS?

Див. [Hetzner (Docker VPS)](/uk/install/hetzner) і
[Docker VM Runtime](/uk/install/docker-vm-runtime), щоб переглянути кроки спільного VM-розгортання,
зокрема вбудовування бінарних файлів, збереження даних і оновлення.

## Sandbox агента

Коли ввімкнено `agents.defaults.sandbox`, gateway виконує інструменти агента
(shell, читання/запис файлів тощо) в ізольованих Docker-контейнерах, тоді як
сам gateway залишається на хості. Це дає вам жорстку межу навколо недовірених або
багатокористувацьких сесій агентів без контейнеризації всього gateway.

Область дії sandbox може бути на рівні агента (типово), сесії або спільною. Для кожної області
використовується власний workspace, змонтований у `/workspace`. Ви також можете налаштувати
політики дозволу/заборони інструментів, ізоляцію мережі, обмеження ресурсів і
контейнери браузера.

Повну інформацію про конфігурацію, образи, примітки щодо безпеки та профілі з кількома агентами див. тут:

- [Sandboxing](/uk/gateway/sandboxing) -- повний довідник із sandbox
- [OpenShell](/uk/gateway/openshell) -- інтерактивний доступ до shell у sandbox-контейнерах
- [Multi-Agent Sandbox and Tools](/uk/tools/multi-agent-sandbox-tools) -- перевизначення для окремих агентів

### Швидке ввімкнення

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

Зберіть типовий образ sandbox:

```bash
scripts/sandbox-setup.sh
```

## Усунення несправностей

<AccordionGroup>
  <Accordion title="Образ відсутній або sandbox-контейнер не запускається">
    Зберіть образ sandbox за допомогою
    [`scripts/sandbox-setup.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/sandbox-setup.sh)
    або встановіть для `agents.defaults.sandbox.docker.image` свій користувацький образ.
    Контейнери автоматично створюються за потреби для кожної сесії.
  </Accordion>

  <Accordion title="Помилки прав доступу в sandbox">
    Установіть `docker.user` у UID:GID, що відповідає власнику змонтованого workspace,
    або змініть власника папки workspace через chown.
  </Accordion>

  <Accordion title="Користувацькі інструменти не знайдено в sandbox">
    OpenClaw запускає команди через `sh -lc` (login shell), який зчитує
    `/etc/profile` і може скинути PATH. Установіть `docker.env.PATH`, щоб додати
    ваші шляхи до користувацьких інструментів на початок, або додайте скрипт у `/etc/profile.d/`
    у вашому Dockerfile.
  </Accordion>

  <Accordion title="Завершено через OOM під час збирання образу (exit 137)">
    VM потрібні щонайменше 2 ГБ RAM. Використайте більший клас машини та повторіть спробу.
  </Accordion>

  <Accordion title="Unauthorized або потрібне pairing у Control UI">
    Отримайте нове посилання dashboard і схваліть пристрій браузера:

    ```bash
    docker compose run --rm openclaw-cli dashboard --no-open
    docker compose run --rm openclaw-cli devices list
    docker compose run --rm openclaw-cli devices approve <requestId>
    ```

    Докладніше: [Dashboard](/web/dashboard), [Devices](/cli/devices).

  </Accordion>

  <Accordion title="Ціль gateway показує ws://172.x.x.x або помилки pairing із Docker CLI">
    Скиньте режим і прив’язку gateway:

    ```bash
    docker compose run --rm openclaw-cli config set --batch-json '[{"path":"gateway.mode","value":"local"},{"path":"gateway.bind","value":"lan"}]'
    docker compose run --rm openclaw-cli devices list --url ws://127.0.0.1:18789
    ```

  </Accordion>
</AccordionGroup>

## Пов’язане

- [Install Overview](/uk/install) — усі методи встановлення
- [Podman](/uk/install/podman) — альтернатива Docker на базі Podman
- [ClawDock](/uk/install/clawdock) — спільнотне налаштування Docker Compose
- [Updating](/uk/install/updating) — як підтримувати OpenClaw в актуальному стані
- [Configuration](/uk/gateway/configuration) — конфігурація gateway після встановлення
