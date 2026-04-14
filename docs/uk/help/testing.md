---
read_when:
    - Запуск тестів локально або в CI
    - Додавання регресій для помилок моделі/провайдера
    - Налагодження поведінки Gateway + агента
summary: 'Набір для тестування: модульні/e2e/live набори, Docker-ранери та що охоплює кожен тест'
title: Тестування
x-i18n:
    generated_at: "2026-04-14T20:31:18Z"
    model: gpt-5.4
    provider: openai
    source_hash: 5efa95e37c3c4b8d4ff979b42981fb238dd8441b747d38a81c66d77050d78337
    source_path: help/testing.md
    workflow: 15
---

# Тестування

OpenClaw має три набори Vitest (unit/integration, e2e, live) і невеликий набір Docker-ранерів.

Цей документ — це посібник «як ми тестуємо»:

- Що охоплює кожен набір (і що він навмисно _не_ охоплює)
- Які команди запускати для поширених сценаріїв роботи (локально, перед push, налагодження)
- Як live-тести знаходять облікові дані та вибирають моделі/провайдерів
- Як додавати регресії для реальних проблем моделей/провайдерів

## Швидкий старт

У більшості випадків:

- Повний gate (очікується перед push): `pnpm build && pnpm check && pnpm test`
- Швидший локальний запуск повного набору на продуктивній машині: `pnpm test:max`
- Прямий цикл спостереження Vitest: `pnpm test:watch`
- Пряме націлення на файл тепер також маршрутизує шляхи extension/channel: `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts`
- Під час ітерацій над однією помилкою спочатку надавайте перевагу точковим запускам.
- QA-сайт на базі Docker: `pnpm qa:lab:up`
- QA-лінія на базі Linux VM: `pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline`

Коли ви змінюєте тести або хочете більше впевненості:

- Gate покриття: `pnpm test:coverage`
- Набір E2E: `pnpm test:e2e`

Під час налагодження реальних провайдерів/моделей (потрібні реальні облікові дані):

- Live-набір (моделі + gateway-проби інструментів/зображень): `pnpm test:live`
- Тихо запустити один live-файл: `pnpm test:live -- src/agents/models.profiles.live.test.ts`

Порада: коли вам потрібен лише один збійний випадок, краще звузити live-тести через env-змінні allowlist, описані нижче.

## Спеціалізовані QA-ранери

Ці команди розташовані поряд з основними тестовими наборами, коли вам потрібен реалізм QA-lab:

- `pnpm openclaw qa suite`
  - Запускає QA-сценарії з репозиторію безпосередньо на хості.
  - За замовчуванням запускає кілька вибраних сценаріїв паралельно з ізольованими worker-процесами gateway, до 64 воркерів або кількості вибраних сценаріїв. Використовуйте `--concurrency <count>`, щоб налаштувати кількість воркерів, або `--concurrency 1` для старішої послідовної лінії.
- `pnpm openclaw qa suite --runner multipass`
  - Запускає той самий QA-набір у тимчасовій Linux VM Multipass.
  - Зберігає ту саму поведінку вибору сценаріїв, що й `qa suite` на хості.
  - Повторно використовує ті самі прапори вибору провайдера/моделі, що й `qa suite`.
  - Live-запуски передають підтримувані вхідні дані QA-автентифікації, які практичні для гостьової системи:
    ключі провайдерів через env, шлях до конфігурації QA live-провайдера та `CODEX_HOME`, якщо він присутній.
  - Каталоги виводу мають залишатися в межах кореня репозиторію, щоб гостьова система могла записувати назад через змонтований workspace.
  - Записує звичайний QA-звіт і summary, а також логи Multipass у
    `.artifacts/qa-e2e/...`.
- `pnpm qa:lab:up`
  - Запускає QA-сайт на базі Docker для QA-роботи в операторському стилі.
- `pnpm openclaw qa matrix`
  - Запускає Matrix live QA-лінію на основі тимчасового homeserver Tuwunel у Docker.
  - Цей QA-хост наразі призначений лише для репозиторію/розробки. Упаковані інсталяції OpenClaw не постачають `qa-lab`, тому вони не надають `openclaw qa`.
  - Копії репозиторію можуть напряму підключати plugin з дерева:
    `openclaw plugins install -l ./extensions/qa-matrix`.
  - Налаштовує трьох тимчасових користувачів Matrix (`driver`, `sut`, `observer`) плюс одну приватну кімнату, потім запускає дочірній QA gateway з реальним Matrix plugin як транспортом SUT.
  - За замовчуванням використовує зафіксований стабільний образ Tuwunel `ghcr.io/matrix-construct/tuwunel:v1.5.1`. Замініть через `OPENCLAW_QA_MATRIX_TUWUNEL_IMAGE`, якщо потрібно протестувати інший образ.
  - Matrix не надає спільні прапори джерел облікових даних, оскільки ця лінія локально створює тимчасових користувачів.
  - Записує Matrix QA-звіт, summary та observed-events artifact у `.artifacts/qa-e2e/...`.
- `pnpm openclaw qa telegram`
  - Запускає Telegram live QA-лінію проти реальної приватної групи з токенами ботів driver і SUT з env.
  - Потребує `OPENCLAW_QA_TELEGRAM_GROUP_ID`, `OPENCLAW_QA_TELEGRAM_DRIVER_BOT_TOKEN` і `OPENCLAW_QA_TELEGRAM_SUT_BOT_TOKEN`. Ідентифікатор групи має бути числовим Telegram chat id.
  - Підтримує `--credential-source convex` для спільних пулових облікових даних. За замовчуванням використовуйте env-режим або встановіть `OPENCLAW_QA_CREDENTIAL_SOURCE=convex`, щоб увімкнути орендовані облікові дані з пулу.
  - Потребує двох різних ботів в одній приватній групі, причому бот SUT має надавати Telegram username.
  - Для стабільного спостереження бот-до-бота увімкніть Bot-to-Bot Communication Mode у `@BotFather` для обох ботів і переконайтеся, що бот driver може спостерігати груповий трафік ботів.
  - Записує Telegram QA-звіт, summary та observed-messages artifact у `.artifacts/qa-e2e/...`.

Live-транспортні лінії використовують один стандартний контракт, щоб нові транспорти не дрейфували:

`qa-channel` залишається широким синтетичним QA-набором і не входить до матриці покриття live-транспортів.

| Лінія    | Canary | Gating за згадками | Блокування allowlist | Відповідь верхнього рівня | Відновлення після перезапуску | Продовження у треді | Ізоляція тредів | Спостереження за реакціями | Команда help |
| -------- | ------ | ------------------ | -------------------- | ------------------------- | ----------------------------- | ------------------- | --------------- | --------------------------- | ------------ |
| Matrix   | x      | x                  | x                    | x                         | x                             | x                   | x               | x                           |              |
| Telegram | x      |                    |                      |                           |                               |                     |                 |                             | x            |

### Спільні Telegram-облікові дані через Convex (v1)

Коли для `openclaw qa telegram` увімкнено `--credential-source convex` (або `OPENCLAW_QA_CREDENTIAL_SOURCE=convex`), QA lab отримує ексклюзивну оренду з пулу на базі Convex, надсилає Heartbeat для цієї оренди під час роботи лінії та звільняє оренду під час завершення.

Еталонний scaffold проєкту Convex:

- `qa/convex-credential-broker/`

Потрібні env-змінні:

- `OPENCLAW_QA_CONVEX_SITE_URL` (наприклад, `https://your-deployment.convex.site`)
- Один секрет для вибраної ролі:
  - `OPENCLAW_QA_CONVEX_SECRET_MAINTAINER` для `maintainer`
  - `OPENCLAW_QA_CONVEX_SECRET_CI` для `ci`
- Вибір ролі облікових даних:
  - CLI: `--credential-role maintainer|ci`
  - Типове значення env: `OPENCLAW_QA_CREDENTIAL_ROLE` (типово `maintainer`)

Необов’язкові env-змінні:

- `OPENCLAW_QA_CREDENTIAL_LEASE_TTL_MS` (типово `1200000`)
- `OPENCLAW_QA_CREDENTIAL_HEARTBEAT_INTERVAL_MS` (типово `30000`)
- `OPENCLAW_QA_CREDENTIAL_ACQUIRE_TIMEOUT_MS` (типово `90000`)
- `OPENCLAW_QA_CREDENTIAL_HTTP_TIMEOUT_MS` (типово `15000`)
- `OPENCLAW_QA_CONVEX_ENDPOINT_PREFIX` (типово `/qa-credentials/v1`)
- `OPENCLAW_QA_CREDENTIAL_OWNER_ID` (необов’язковий trace id)
- `OPENCLAW_QA_ALLOW_INSECURE_HTTP=1` дозволяє loopback-URL Convex виду `http://` лише для локальної розробки.

У звичайному режимі `OPENCLAW_QA_CONVEX_SITE_URL` має використовувати `https://`.

Адміністративні команди для maintainer (додавання/видалення/перелік пулу) потребують саме `OPENCLAW_QA_CONVEX_SECRET_MAINTAINER`.

CLI-хелпери для maintainers:

```bash
pnpm openclaw qa credentials add --kind telegram --payload-file qa/telegram-credential.json
pnpm openclaw qa credentials list --kind telegram
pnpm openclaw qa credentials remove --credential-id <credential-id>
```

Використовуйте `--json` для машиночитаного виводу в скриптах і утилітах CI.

Типовий контракт endpoint (`OPENCLAW_QA_CONVEX_SITE_URL` + `/qa-credentials/v1`):

- `POST /acquire`
  - Запит: `{ kind, ownerId, actorRole, leaseTtlMs, heartbeatIntervalMs }`
  - Успіх: `{ status: "ok", credentialId, leaseToken, payload, leaseTtlMs?, heartbeatIntervalMs? }`
  - Вичерпано/можна повторити: `{ status: "error", code: "POOL_EXHAUSTED" | "NO_CREDENTIAL_AVAILABLE", ... }`
- `POST /heartbeat`
  - Запит: `{ kind, ownerId, actorRole, credentialId, leaseToken, leaseTtlMs }`
  - Успіх: `{ status: "ok" }` (або порожній `2xx`)
- `POST /release`
  - Запит: `{ kind, ownerId, actorRole, credentialId, leaseToken }`
  - Успіх: `{ status: "ok" }` (або порожній `2xx`)
- `POST /admin/add` (лише секрет maintainer)
  - Запит: `{ kind, actorId, payload, note?, status? }`
  - Успіх: `{ status: "ok", credential }`
- `POST /admin/remove` (лише секрет maintainer)
  - Запит: `{ credentialId, actorId }`
  - Успіх: `{ status: "ok", changed, credential }`
  - Захист від активної оренди: `{ status: "error", code: "LEASE_ACTIVE", ... }`
- `POST /admin/list` (лише секрет maintainer)
  - Запит: `{ kind?, status?, includePayload?, limit? }`
  - Успіх: `{ status: "ok", credentials, count }`

Форма payload для типу Telegram:

- `{ groupId: string, driverToken: string, sutToken: string }`
- `groupId` має бути рядком із числовим Telegram chat id.
- `admin/add` перевіряє цю форму для `kind: "telegram"` і відхиляє некоректний payload.

### Додавання каналу до QA

Додавання каналу до markdown QA-системи потребує рівно двох речей:

1. Транспортний адаптер для каналу.
2. Набір сценаріїв, який перевіряє контракт каналу.

Не додавайте новий кореневий QA-командний простір верхнього рівня, якщо спільний хост `qa-lab` може володіти цим потоком.

`qa-lab` володіє спільною хост-логікою:

- кореневою командою `openclaw qa`
- запуском і завершенням наборів
- паралелізмом воркерів
- записом артефактів
- генерацією звітів
- виконанням сценаріїв
- alias сумісності для старіших сценаріїв `qa-channel`

Runner plugins володіють транспортним контрактом:

- як `openclaw qa <runner>` монтується під спільним коренем `qa`
- як gateway налаштовується для цього транспорту
- як перевіряється готовність
- як інжектуються вхідні події
- як спостерігаються вихідні повідомлення
- як надаються транскрипти та нормалізований стан транспорту
- як виконуються дії на базі транспорту
- як обробляються транспортно-специфічні скидання або очищення

Мінімальна планка впровадження для нового каналу така:

1. Залишайте `qa-lab` власником спільного кореня `qa`.
2. Реалізуйте transport runner на спільному шві хоста `qa-lab`.
3. Залишайте транспортно-специфічну механіку в runner plugin або harness plugin.
4. Монтуйте ранер як `openclaw qa <runner>`, а не реєструйте конкуруючий кореневий простір команд.
   Runner plugins мають оголошувати `qaRunners` у `openclaw.plugin.json` і експортувати відповідний масив `qaRunnerCliRegistrations` з `runtime-api.ts`.
   Залишайте `runtime-api.ts` легким; ліниве виконання CLI і ранерів має залишатися за окремими entrypoint.
5. Створюйте або адаптуйте markdown-сценарії в `qa/scenarios/`.
6. Використовуйте generic helper-и для нових сценаріїв.
7. Зберігайте наявні alias сумісності працездатними, якщо тільки репозиторій не виконує навмисну міграцію.

Правило прийняття рішення суворе:

- Якщо поведінку можна один раз виразити в `qa-lab`, розміщуйте її в `qa-lab`.
- Якщо поведінка залежить від одного транспортного каналу, залишайте її в цьому runner plugin або plugin harness.
- Якщо сценарію потрібна нова можливість, яку можуть використовувати більше ніж один канал, додайте generic helper замість channel-specific гілки в `suite.ts`.
- Якщо поведінка має сенс лише для одного транспорту, залишайте сценарій transport-specific і явно відображайте це в контракті сценарію.

Бажані назви generic helper-ів для нових сценаріїв:

- `waitForTransportReady`
- `waitForChannelReady`
- `injectInboundMessage`
- `injectOutboundMessage`
- `waitForTransportOutboundMessage`
- `waitForChannelOutboundMessage`
- `waitForNoTransportOutbound`
- `getTransportSnapshot`
- `readTransportMessage`
- `readTransportTranscript`
- `formatTransportTranscript`
- `resetTransport`

Alias сумісності залишаються доступними для наявних сценаріїв, зокрема:

- `waitForQaChannelReady`
- `waitForOutboundMessage`
- `waitForNoOutbound`
- `formatConversationTranscript`
- `resetBus`

Нова робота над каналами має використовувати generic helper-и.
Alias сумісності існують, щоб уникнути міграції «за один день», а не як модель для
створення нових сценаріїв.

## Тестові набори (що де запускається)

Думайте про набори як про «зростання реалізму» (і зростання нестабільності/вартості):

### Unit / integration (типовий)

- Команда: `pnpm test`
- Конфігурація: десять послідовних shard-запусків (`vitest.full-*.config.ts`) поверх наявних scoped-проєктів Vitest
- Файли: інвентарі core/unit у `src/**/*.test.ts`, `packages/**/*.test.ts`, `test/**/*.test.ts` і внесені до allowlist node-тести `ui`, які охоплює `vitest.unit.config.ts`
- Обсяг:
  - Чисті unit-тести
  - In-process integration-тести (автентифікація gateway, маршрутизація, tooling, парсинг, конфігурація)
  - Детерміновані регресії для відомих помилок
- Очікування:
  - Запускається в CI
  - Реальні ключі не потрібні
  - Має бути швидким і стабільним
- Примітка щодо проєктів:
  - Нетаргетований `pnpm test` тепер запускає одинадцять менших shard-конфігурацій (`core-unit-src`, `core-unit-security`, `core-unit-ui`, `core-unit-support`, `core-support-boundary`, `core-contracts`, `core-bundled`, `core-runtime`, `agentic`, `auto-reply`, `extensions`) замість одного великого native root-project процесу. Це зменшує піковий RSS на завантажених машинах і не дає роботі auto-reply/extension виснажувати не пов’язані набори.
  - `pnpm test --watch` досі використовує native root-граф проєктів `vitest.config.ts`, оскільки цикл спостереження з кількома shard-ами непрактичний.
  - `pnpm test`, `pnpm test:watch` і `pnpm test:perf:imports` спочатку маршрутизують явні file/directory target-и через scoped-лінії, тому `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts` уникає повної вартості запуску root-проєкту.
  - `pnpm test:changed` розгортає змінені git-шляхи в ті самі scoped-лінії, коли diff торкається лише маршрутизованих source/test файлів; редагування config/setup усе ще повертаються до широкого повторного запуску root-проєкту.
  - Легкі для імпорту unit-тести з agents, commands, plugins, helper-ів auto-reply, `plugin-sdk` і подібних чистих утилітарних областей маршрутизуються через лінію `unit-fast`, яка пропускає `test/setup-openclaw-runtime.ts`; stateful/runtime-heavy файли залишаються на наявних лініях.
  - Вибрані helper source-файли `plugin-sdk` і `commands` також маплять changed-mode запуски на явні sibling-тести в цих легких лініях, тому редагування helper-ів не потребують повторного запуску всього важкого набору для цього каталогу.
  - `auto-reply` тепер має три окремі кошики: top-level core helper-и, top-level integration-тести `reply.*` і піддерево `src/auto-reply/reply/**`. Це не дає найважчій harness-роботі reply навантажувати дешеві тести status/chunk/token.
- Примітка щодо embedded runner:
  - Коли ви змінюєте вхідні дані виявлення message-tool або runtime-контекст Compaction,
    зберігайте обидва рівні покриття.
  - Додавайте сфокусовані helper-регресії для чистих меж маршрутизації/нормалізації.
  - Також підтримуйте здоровими integration-набори embedded runner:
    `src/agents/pi-embedded-runner/compact.hooks.test.ts`,
    `src/agents/pi-embedded-runner/run.overflow-compaction.test.ts` і
    `src/agents/pi-embedded-runner/run.overflow-compaction.loop.test.ts`.
  - Ці набори перевіряють, що scoped id та поведінка Compaction, як і раніше, проходять
    через реальні шляхи `run.ts` / `compact.ts`; лише helper-тести не є
    достатньою заміною для цих integration-шляхів.
- Примітка щодо pool:
  - Базова конфігурація Vitest тепер типово використовує `threads`.
  - Спільна конфігурація Vitest також фіксує `isolate: false` і використовує non-isolated runner у root-проєктах, e2e та live-конфігураціях.
  - Коренева лінія UI зберігає свій `jsdom` setup та optimizer, але тепер також працює на спільному non-isolated runner.
  - Кожен shard `pnpm test` успадковує ті самі типові значення `threads` + `isolate: false` зі спільної конфігурації Vitest.
  - Спільний launcher `scripts/run-vitest.mjs` тепер також типово додає `--no-maglev` для дочірніх Node-процесів Vitest, щоб зменшити churn компіляції V8 під час великих локальних запусків. Установіть `OPENCLAW_VITEST_ENABLE_MAGLEV=1`, якщо вам потрібно порівняти зі стандартною поведінкою V8.
- Примітка щодо швидкої локальної ітерації:
  - `pnpm test:changed` маршрутизується через scoped-лінії, коли змінені шляхи чисто мапляться на менший набір.
  - `pnpm test:max` і `pnpm test:changed:max` зберігають ту саму поведінку маршрутизації, лише з вищим лімітом воркерів.
  - Автомасштабування локальних воркерів тепер навмисно консервативне й також зменшує навантаження, коли середнє завантаження хоста вже високе, тому кілька одночасних запусків Vitest за замовчуванням завдають менше шкоди.
  - Базова конфігурація Vitest позначає файли проєктів/конфігурації як `forceRerunTriggers`, щоб повторні запуски в changed-mode залишалися коректними, коли змінюється wiring тестів.
  - Конфігурація зберігає `OPENCLAW_VITEST_FS_MODULE_CACHE` увімкненим на підтримуваних хостах; установіть `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/abs/path`, якщо хочете одну явну локацію кешу для прямого профілювання.
- Примітка щодо налагодження продуктивності:
  - `pnpm test:perf:imports` вмикає звітність про тривалість імпорту у Vitest та вивід import-breakdown.
  - `pnpm test:perf:imports:changed` звужує той самий профілювальний огляд до файлів, змінених відносно `origin/main`.
- `pnpm test:perf:changed:bench -- --ref <git-ref>` порівнює маршрутизований `test:changed` з native root-project шляхом для цього зафіксованого diff і виводить wall time та macOS max RSS.
- `pnpm test:perf:changed:bench -- --worktree` вимірює поточне брудне дерево, маршрутизуючи список змінених файлів через `scripts/test-projects.mjs` і root-конфігурацію Vitest.
  - `pnpm test:perf:profile:main` записує CPU-профіль main-thread для накладних витрат запуску та трансформації Vitest/Vite.
  - `pnpm test:perf:profile:runner` записує CPU+heap-профілі runner-а для unit-набору з вимкненим файловим паралелізмом.

### E2E (gateway smoke)

- Команда: `pnpm test:e2e`
- Конфігурація: `vitest.e2e.config.ts`
- Файли: `src/**/*.e2e.test.ts`, `test/**/*.e2e.test.ts`
- Типові параметри runtime:
  - Використовує `threads` у Vitest з `isolate: false`, як і решта репозиторію.
  - Використовує адаптивну кількість воркерів (CI: до 2, локально: типово 1).
  - Типово запускається в silent-режимі, щоб зменшити накладні витрати на console I/O.
- Корисні override-и:
  - `OPENCLAW_E2E_WORKERS=<n>` щоб примусово задати кількість воркерів (обмежено 16).
  - `OPENCLAW_E2E_VERBOSE=1` щоб знову ввімкнути докладний вивід у консоль.
- Обсяг:
  - Наскрізна поведінка gateway з кількома інстансами
  - Поверхні WebSocket/HTTP, pairing Node та важча мережева взаємодія
- Очікування:
  - Запускається в CI (коли увімкнено в pipeline)
  - Реальні ключі не потрібні
  - Більше рухомих частин, ніж у unit-тестах (може бути повільніше)

### E2E: smoke для backend OpenShell

- Команда: `pnpm test:e2e:openshell`
- Файл: `test/openshell-sandbox.e2e.test.ts`
- Обсяг:
  - Запускає ізольований gateway OpenShell на хості через Docker
  - Створює sandbox з тимчасового локального Dockerfile
  - Перевіряє backend OpenShell OpenClaw через реальні `sandbox ssh-config` + SSH exec
  - Перевіряє remote-canonical поведінку файлової системи через fs bridge sandbox
- Очікування:
  - Лише opt-in; не входить до типового запуску `pnpm test:e2e`
  - Потребує локальний CLI `openshell` і працездатний Docker daemon
  - Використовує ізольовані `HOME` / `XDG_CONFIG_HOME`, а потім знищує тестовий gateway і sandbox
- Корисні override-и:
  - `OPENCLAW_E2E_OPENSHELL=1` щоб увімкнути тест під час ручного запуску ширшого e2e-набору
  - `OPENCLAW_E2E_OPENSHELL_COMMAND=/path/to/openshell` щоб указати нестандартний CLI-бінарник або wrapper-скрипт

### Live (реальні провайдери + реальні моделі)

- Команда: `pnpm test:live`
- Конфігурація: `vitest.live.config.ts`
- Файли: `src/**/*.live.test.ts`
- Типово: **увімкнено** через `pnpm test:live` (встановлює `OPENCLAW_LIVE_TEST=1`)
- Обсяг:
  - «Чи ця модель/провайдер справді працює _сьогодні_ з реальними обліковими даними?»
  - Виявлення змін форматів провайдерів, особливостей виклику інструментів, проблем автентифікації та поведінки rate limit
- Очікування:
  - За задумом нестабільно для CI (реальні мережі, реальні політики провайдерів, квоти, збої)
  - Коштує грошей / використовує rate limit
  - Краще запускати звужені підмножини, а не «все»
- Live-запуски використовують `~/.profile`, щоб підхопити відсутні API-ключі.
- За замовчуванням live-запуски все одно ізолюють `HOME` і копіюють матеріали config/auth у тимчасовий test home, щоб unit-фікстури не могли змінити ваш реальний `~/.openclaw`.
- Установлюйте `OPENCLAW_LIVE_USE_REAL_HOME=1` лише тоді, коли навмисно потрібно, щоб live-тести використовували ваш реальний домашній каталог.
- `pnpm test:live` тепер типово працює в тихішому режимі: він зберігає вивід прогресу `[live] ...`, але приховує додаткове повідомлення про `~/.profile` і приглушує логи bootstrap gateway/шум Bonjour. Установіть `OPENCLAW_LIVE_TEST_QUIET=0`, якщо хочете повернути повні startup-логи.
- Ротація API-ключів (залежно від провайдера): установлюйте `*_API_KEYS` у форматі з комами/крапками з комою або `*_API_KEY_1`, `*_API_KEY_2` (наприклад, `OPENAI_API_KEYS`, `ANTHROPIC_API_KEYS`, `GEMINI_API_KEYS`) або використовуйте override на рівні live через `OPENCLAW_LIVE_*_KEY`; тести повторюють спробу у відповідь на rate limit.
- Вивід progress/Heartbeat:
  - Live-набори тепер виводять рядки прогресу в stderr, щоб довгі виклики провайдерів було видно як активні, навіть коли перехоплення консолі Vitest тихе.
  - `vitest.live.config.ts` вимикає перехоплення консолі Vitest, тому рядки прогресу провайдера/gateway відразу транслюються під час live-запусків.
  - Налаштовуйте Heartbeat для прямих моделей через `OPENCLAW_LIVE_HEARTBEAT_MS`.
  - Налаштовуйте Heartbeat для gateway/probe через `OPENCLAW_LIVE_GATEWAY_HEARTBEAT_MS`.

## Який набір мені запускати?

Скористайтеся цією таблицею рішень:

- Редагуєте логіку/тести: запускайте `pnpm test` (і `pnpm test:coverage`, якщо ви багато що змінили)
- Торкаєтеся мережевої взаємодії gateway / WS-протоколу / pairing: додайте `pnpm test:e2e`
- Налагоджуєте «мій бот не працює» / збої, специфічні для провайдера / виклик інструментів: запускайте звужений `pnpm test:live`

## Live: перевірка можливостей Android Node

- Тест: `src/gateway/android-node.capabilities.live.test.ts`
- Скрипт: `pnpm android:test:integration`
- Мета: викликати **кожну команду, яка зараз рекламується** підключеним Android Node, і перевірити поведінку контракту команди.
- Обсяг:
  - Попередньо підготовлене/ручне налаштування (набір не встановлює, не запускає і не pair-ить застосунок).
  - Перевірка gateway `node.invoke` для вибраного Android Node команда за командою.
- Потрібне попереднє налаштування:
  - Android-застосунок уже підключено та зpairено з gateway.
  - Застосунок утримується на передньому плані.
  - Надано дозволи/згоду на захоплення для можливостей, які ви очікуєте як успішні.
- Необов’язкові override-и цілі:
  - `OPENCLAW_ANDROID_NODE_ID` або `OPENCLAW_ANDROID_NODE_NAME`.
  - `OPENCLAW_ANDROID_GATEWAY_URL` / `OPENCLAW_ANDROID_GATEWAY_TOKEN` / `OPENCLAW_ANDROID_GATEWAY_PASSWORD`.
- Повні подробиці налаштування Android: [Android App](/uk/platforms/android)

## Live: smoke для моделей (ключі профілів)

Live-тести поділено на два шари, щоб можна було ізолювати збої:

- «Пряма модель» показує нам, чи може провайдер/модель взагалі відповісти з наданим ключем.
- «Smoke Gateway» показує нам, чи працює повний pipeline gateway+agent для цієї моделі (сесії, історія, інструменти, політика sandbox тощо).

### Шар 1: Пряме завершення моделі (без gateway)

- Тест: `src/agents/models.profiles.live.test.ts`
- Мета:
  - Перелічити виявлені моделі
  - Використати `getApiKeyForModel`, щоб вибрати моделі, для яких у вас є облікові дані
  - Запустити невелике completion для кожної моделі (і точкові регресії за потреби)
- Як увімкнути:
  - `pnpm test:live` (або `OPENCLAW_LIVE_TEST=1`, якщо запускати Vitest напряму)
- Установіть `OPENCLAW_LIVE_MODELS=modern` (або `all`, alias для modern), щоб фактично запустити цей набір; інакше його буде пропущено, щоб `pnpm test:live` залишався сфокусованим на gateway smoke
- Як вибирати моделі:
  - `OPENCLAW_LIVE_MODELS=modern` для запуску modern allowlist (Opus/Sonnet 4.6+, GPT-5.x + Codex, Gemini 3, GLM 4.7, MiniMax M2.7, Grok 4)
  - `OPENCLAW_LIVE_MODELS=all` — це alias для modern allowlist
  - або `OPENCLAW_LIVE_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,..."` (allowlist через коми)
  - Modern/all sweeps типово використовують curated high-signal cap; установіть `OPENCLAW_LIVE_MAX_MODELS=0` для вичерпного modern sweep або додатне число для меншого ліміту.
- Як вибирати провайдерів:
  - `OPENCLAW_LIVE_PROVIDERS="google,google-antigravity,google-gemini-cli"` (allowlist через коми)
- Звідки беруться ключі:
  - Типово: сховище профілів і env-fallback-и
  - Установіть `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`, щоб примусово використовувати **лише сховище профілів**
- Навіщо це існує:
  - Відокремлює «API провайдера зламаний / ключ недійсний» від «pipeline gateway agent зламаний»
  - Містить невеликі ізольовані регресії (наприклад: повторне відтворення reasoning OpenAI Responses/Codex Responses + потоки tool-call)

### Шар 2: smoke Gateway + dev agent (що насправді робить "@openclaw")

- Тест: `src/gateway/gateway-models.profiles.live.test.ts`
- Мета:
  - Підняти in-process gateway
  - Створити/оновити сесію `agent:dev:*` (override моделі для кожного запуску)
  - Ітерувати моделі з ключами та перевіряти:
    - «змістовну» відповідь (без інструментів)
    - що реальний виклик інструмента працює (проба `read`)
    - необов’язкові додаткові проби інструментів (проба `exec+read`)
    - що шляхи регресій OpenAI (лише tool-call → follow-up) продовжують працювати
- Деталі probes (щоб ви могли швидко пояснювати збої):
  - Проба `read`: тест записує файл з nonce у workspace і просить агента `read` його та повернути nonce у відповіді.
  - Проба `exec+read`: тест просить агента через `exec` записати nonce у тимчасовий файл, а потім через `read` прочитати його назад.
  - Проба зображення: тест додає згенерований PNG (кіт + рандомізований код) і очікує, що модель поверне `cat <CODE>`.
  - Еталонна реалізація: `src/gateway/gateway-models.profiles.live.test.ts` і `src/gateway/live-image-probe.ts`.
- Як увімкнути:
  - `pnpm test:live` (або `OPENCLAW_LIVE_TEST=1`, якщо запускати Vitest напряму)
- Як вибирати моделі:
  - Типово: modern allowlist (Opus/Sonnet 4.6+, GPT-5.x + Codex, Gemini 3, GLM 4.7, MiniMax M2.7, Grok 4)
  - `OPENCLAW_LIVE_GATEWAY_MODELS=all` — це alias для modern allowlist
  - Або встановіть `OPENCLAW_LIVE_GATEWAY_MODELS="provider/model"` (або список через кому), щоб звузити вибір
  - Modern/all gateway sweeps типово використовують curated high-signal cap; установіть `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=0` для вичерпного modern sweep або додатне число для меншого ліміту.
- Як вибирати провайдерів (уникати «усе OpenRouter»):
  - `OPENCLAW_LIVE_GATEWAY_PROVIDERS="google,google-antigravity,google-gemini-cli,openai,anthropic,zai,minimax"` (allowlist через кому)
- Проби інструментів і зображень у цьому live-тесті завжди ввімкнені:
  - проба `read` + проба `exec+read` (навантаження на інструменти)
  - проба зображення запускається, коли модель заявляє підтримку вхідних зображень
  - Потік (на високому рівні):
    - Тест генерує маленький PNG із «CAT» + випадковий код (`src/gateway/live-image-probe.ts`)
    - Надсилає його через `agent` `attachments: [{ mimeType: "image/png", content: "<base64>" }]`
    - Gateway розбирає attachments у `images[]` (`src/gateway/server-methods/agent.ts` + `src/gateway/chat-attachments.ts`)
    - Embedded agent передає моделі мультимодальне повідомлення користувача
    - Перевірка: відповідь містить `cat` + код (допускаються незначні OCR-помилки)

Порада: щоб побачити, що саме можна тестувати на вашій машині (і точні ідентифікатори `provider/model`), запустіть:

```bash
openclaw models list
openclaw models list --json
```

## Live: smoke для CLI backend (Claude, Codex, Gemini або інші локальні CLI)

- Тест: `src/gateway/gateway-cli-backend.live.test.ts`
- Мета: перевірити pipeline Gateway + agent з використанням локального CLI backend, не торкаючись вашої типової конфігурації.
- Типові smoke-параметри для конкретного backend зберігаються у визначенні `cli-backend.ts` відповідного extension.
- Увімкнення:
  - `pnpm test:live` (або `OPENCLAW_LIVE_TEST=1`, якщо запускати Vitest напряму)
  - `OPENCLAW_LIVE_CLI_BACKEND=1`
- Типові значення:
  - Типовий провайдер/модель: `claude-cli/claude-sonnet-4-6`
  - Поведінка команди/аргументів/зображень береться з метаданих plugin власника CLI backend.
- Override-и (необов’язково):
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL="codex-cli/gpt-5.4"`
  - `OPENCLAW_LIVE_CLI_BACKEND_COMMAND="/full/path/to/codex"`
  - `OPENCLAW_LIVE_CLI_BACKEND_ARGS='["exec","--json","--color","never","--sandbox","read-only","--skip-git-repo-check"]'`
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_PROBE=1` щоб надіслати реальне вкладення-зображення (шляхи інжектуються у prompt).
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_ARG="--image"` щоб передавати шляхи до файлів зображень як CLI-аргументи замість інжекції в prompt.
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_MODE="repeat"` (або `"list"`), щоб керувати способом передавання аргументів зображень, коли встановлено `IMAGE_ARG`.
  - `OPENCLAW_LIVE_CLI_BACKEND_RESUME_PROBE=1` щоб надіслати другий хід і перевірити потік resume.
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL_SWITCH_PROBE=0` щоб вимкнути типову пробу безперервності в тій самій сесії Claude Sonnet -> Opus (установіть `1`, щоб примусово її ввімкнути, коли вибрана модель підтримує ціль переключення).

Приклад:

```bash
OPENCLAW_LIVE_CLI_BACKEND=1 \
  OPENCLAW_LIVE_CLI_BACKEND_MODEL="codex-cli/gpt-5.4" \
  pnpm test:live src/gateway/gateway-cli-backend.live.test.ts
```

Рецепт Docker:

```bash
pnpm test:docker:live-cli-backend
```

Docker-рецепти для окремих провайдерів:

```bash
pnpm test:docker:live-cli-backend:claude
pnpm test:docker:live-cli-backend:claude-subscription
pnpm test:docker:live-cli-backend:codex
pnpm test:docker:live-cli-backend:gemini
```

Примітки:

- Docker-ранер розташований у `scripts/test-live-cli-backend-docker.sh`.
- Він запускає live smoke для CLI backend усередині Docker-образу репозиторію як непривілейований користувач `node`.
- Він визначає метадані CLI smoke з extension власника, а потім встановлює відповідний Linux-пакет CLI (`@anthropic-ai/claude-code`, `@openai/codex` або `@google/gemini-cli`) у кешований writable prefix за адресою `OPENCLAW_DOCKER_CLI_TOOLS_DIR` (типово: `~/.cache/openclaw/docker-cli-tools`).
- `pnpm test:docker:live-cli-backend:claude-subscription` потребує portable Claude Code subscription OAuth через `~/.claude/.credentials.json` з `claudeAiOauth.subscriptionType` або через `CLAUDE_CODE_OAUTH_TOKEN` з `claude setup-token`. Спочатку він перевіряє прямий `claude -p` у Docker, а потім запускає два ходи Gateway CLI backend без збереження env-змінних Anthropic API key. Ця subscription-лінія типово вимикає Claude MCP/tool і probes зображень, оскільки Claude наразі маршрутизує використання стороннього застосунку через білінг extra-usage, а не через звичайні ліміти subscription plan.
- Live smoke CLI backend тепер перевіряє той самий наскрізний потік для Claude, Codex і Gemini: текстовий хід, хід класифікації зображення, а потім виклик інструмента MCP `cron`, перевірений через Gateway CLI.
- Типовий smoke для Claude також оновлює сесію з Sonnet на Opus і перевіряє, що відновлена сесія все ще пам’ятає попередню нотатку.

## Live: smoke для прив’язки ACP (`/acp spawn ... --bind here`)

- Тест: `src/gateway/gateway-acp-bind.live.test.ts`
- Мета: перевірити реальний потік conversation-bind ACP із live ACP agent:
  - надіслати `/acp spawn <agent> --bind here`
  - прив’язати synthetic conversation каналу повідомлень на місці
  - надіслати звичайний follow-up у тій самій conversation
  - перевірити, що follow-up потрапляє в transcript прив’язаної ACP session
- Увімкнення:
  - `pnpm test:live src/gateway/gateway-acp-bind.live.test.ts`
  - `OPENCLAW_LIVE_ACP_BIND=1`
- Типові значення:
  - ACP-агенти в Docker: `claude,codex,gemini`
  - ACP-агент для прямого `pnpm test:live ...`: `claude`
  - Synthetic channel: контекст conversation у стилі Slack DM
  - ACP backend: `acpx`
- Override-и:
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=claude`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=codex`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude,codex,gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND='npx -y @agentclientprotocol/claude-agent-acp@<version>'`
- Примітки:
  - Ця лінія використовує поверхню gateway `chat.send` з synthetic полями originating-route, доступними лише адміністратору, щоб тести могли приєднати контекст message-channel без удавання зовнішньої доставки.
  - Коли `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND` не встановлено, тест використовує вбудований реєстр агентів plugin `acpx` для вибраного ACP harness agent.

Приклад:

```bash
OPENCLAW_LIVE_ACP_BIND=1 \
  OPENCLAW_LIVE_ACP_BIND_AGENT=claude \
  pnpm test:live src/gateway/gateway-acp-bind.live.test.ts
```

Рецепт Docker:

```bash
pnpm test:docker:live-acp-bind
```

Docker-рецепти для окремих агентів:

```bash
pnpm test:docker:live-acp-bind:claude
pnpm test:docker:live-acp-bind:codex
pnpm test:docker:live-acp-bind:gemini
```

Примітки щодо Docker:

- Docker-ранер розташований у `scripts/test-live-acp-bind-docker.sh`.
- За замовчуванням він послідовно запускає smoke ACP bind для всіх підтримуваних live CLI-агентів: `claude`, `codex`, потім `gemini`.
- Використовуйте `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude`, `OPENCLAW_LIVE_ACP_BIND_AGENTS=codex` або `OPENCLAW_LIVE_ACP_BIND_AGENTS=gemini`, щоб звузити матрицю.
- Він використовує `~/.profile`, готує відповідні матеріали автентифікації CLI в контейнері, встановлює `acpx` у writable npm prefix, а потім, за потреби, встановлює потрібний live CLI (`@anthropic-ai/claude-code`, `@openai/codex` або `@google/gemini-cli`).
- Усередині Docker ранер встановлює `OPENCLAW_LIVE_ACP_BIND_ACPX_COMMAND=$HOME/.npm-global/bin/acpx`, щоб acpx зберігав env-змінні провайдера зі зчитаного profile доступними для дочірнього harness CLI.

## Live: smoke для harness app-server Codex

- Мета: перевірити plugin-owned harness Codex через звичайний gateway-метод
  `agent`:
  - завантажити bundled plugin `codex`
  - вибрати `OPENCLAW_AGENT_RUNTIME=codex`
  - надіслати перший gateway agent turn до `codex/gpt-5.4`
  - надіслати другий turn до тієї самої сесії OpenClaw і перевірити, що thread
    app-server може відновитися
  - запустити `/codex status` і `/codex models` через той самий шлях
    gateway-команд
- Тест: `src/gateway/gateway-codex-harness.live.test.ts`
- Увімкнення: `OPENCLAW_LIVE_CODEX_HARNESS=1`
- Типова модель: `codex/gpt-5.4`
- Необов’язкова probe зображення: `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=1`
- Необов’язкова MCP/tool probe: `OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=1`
- Цей smoke встановлює `OPENCLAW_AGENT_HARNESS_FALLBACK=none`, щоб зламаний Codex
  harness не міг пройти тест, тихо перемкнувшись назад на PI.
- Автентифікація: `OPENAI_API_KEY` із shell/profile, а також за потреби скопійовані
  `~/.codex/auth.json` і `~/.codex/config.toml`

Локальний рецепт:

```bash
source ~/.profile
OPENCLAW_LIVE_CODEX_HARNESS=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_MODEL=codex/gpt-5.4 \
  pnpm test:live -- src/gateway/gateway-codex-harness.live.test.ts
```

Рецепт Docker:

```bash
source ~/.profile
pnpm test:docker:live-codex-harness
```

Примітки щодо Docker:

- Docker-ранер розташований у `scripts/test-live-codex-harness-docker.sh`.
- Він використовує змонтований `~/.profile`, передає `OPENAI_API_KEY`, копіює файли
  автентифікації CLI Codex, якщо вони присутні, встановлює `@openai/codex` у writable змонтований npm
  prefix, готує дерево вихідного коду, а потім запускає лише live-тест Codex-harness.
- У Docker probes зображення та MCP/tool увімкнені за замовчуванням. Установіть
  `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=0` або
  `OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=0`, коли потрібен вужчий запуск для налагодження.
- Docker також експортує `OPENCLAW_AGENT_HARNESS_FALLBACK=none`, як і конфігурація live-тесту,
  тож fallback на `openai-codex/*` або PI не зможе приховати регресію
  Codex harness.

### Рекомендовані live-рецепти

Найшвидшими та найменш нестабільними є вузькі, явні allowlist:

- Одна модель, напряму (без gateway):
  - `OPENCLAW_LIVE_MODELS="openai/gpt-5.4" pnpm test:live src/agents/models.profiles.live.test.ts`

- Одна модель, gateway smoke:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Виклик інструментів у кількох провайдерів:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3-flash-preview,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Фокус на Google (API key Gemini + Antigravity):
  - Gemini (API key): `OPENCLAW_LIVE_GATEWAY_MODELS="google/gemini-3-flash-preview" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`
  - Antigravity (OAuth): `OPENCLAW_LIVE_GATEWAY_MODELS="google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-pro-high" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

Примітки:

- `google/...` використовує Gemini API (API key).
- `google-antigravity/...` використовує OAuth-міст Antigravity (endpoint агента у стилі Cloud Code Assist).
- `google-gemini-cli/...` використовує локальний Gemini CLI на вашій машині (окрема автентифікація + особливості tooling).
- Gemini API проти Gemini CLI:
  - API: OpenClaw викликає розміщений Google Gemini API через HTTP (API key / автентифікація профілю); це те, що більшість користувачів мають на увазі під «Gemini».
  - CLI: OpenClaw виконує shell-виклик локального бінарника `gemini`; він має власну автентифікацію та може поводитися інакше (streaming/підтримка інструментів/version skew).

## Live: матриця моделей (що ми охоплюємо)

Фіксованого «списку моделей CI» немає (live — це opt-in), але це **рекомендовані** моделі, які варто регулярно покривати на dev-машині з ключами.

### Набір modern smoke (виклик інструментів + зображення)

Це запуск «поширених моделей», який ми очікуємо зберігати працездатним:

- OpenAI (не-Codex): `openai/gpt-5.4` (необов’язково: `openai/gpt-5.4-mini`)
- OpenAI Codex: `openai-codex/gpt-5.4`
- Anthropic: `anthropic/claude-opus-4-6` (або `anthropic/claude-sonnet-4-6`)
- Google (Gemini API): `google/gemini-3.1-pro-preview` і `google/gemini-3-flash-preview` (уникайте старіших моделей Gemini 2.x)
- Google (Antigravity): `google-antigravity/claude-opus-4-6-thinking` і `google-antigravity/gemini-3-flash`
- Z.AI (GLM): `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

Запускати gateway smoke з інструментами + зображенням:
`OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,openai-codex/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3.1-pro-preview,google/gemini-3-flash-preview,google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-flash,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

### Базовий рівень: виклик інструментів (Read + необов’язковий Exec)

Виберіть принаймні одну модель на сімейство провайдерів:

- OpenAI: `openai/gpt-5.4` (або `openai/gpt-5.4-mini`)
- Anthropic: `anthropic/claude-opus-4-6` (або `anthropic/claude-sonnet-4-6`)
- Google: `google/gemini-3-flash-preview` (або `google/gemini-3.1-pro-preview`)
- Z.AI (GLM): `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

Необов’язкове додаткове покриття (було б добре мати):

- xAI: `xai/grok-4` (або найновіша доступна)
- Mistral: `mistral/`… (виберіть одну модель з підтримкою “tools”, яку у вас ввімкнено)
- Cerebras: `cerebras/`… (якщо у вас є доступ)
- LM Studio: `lmstudio/`… (локально; виклик інструментів залежить від режиму API)

### Vision: надсилання зображення (вкладення → мультимодальне повідомлення)

Додайте принаймні одну модель із підтримкою зображень до `OPENCLAW_LIVE_GATEWAY_MODELS` (Claude/Gemini/OpenAI-варіанти з підтримкою vision тощо), щоб перевірити image probe.

### Агрегатори / альтернативні gateway

Якщо у вас увімкнені ключі, ми також підтримуємо тестування через:

- OpenRouter: `openrouter/...` (сотні моделей; використовуйте `openclaw models scan`, щоб знайти кандидатів із підтримкою інструментів+зображень)
- OpenCode: `opencode/...` для Zen і `opencode-go/...` для Go (автентифікація через `OPENCODE_API_KEY` / `OPENCODE_ZEN_API_KEY`)

Більше провайдерів, які можна включити до live-матриці (якщо у вас є облікові дані/конфігурація):

- Вбудовані: `openai`, `openai-codex`, `anthropic`, `google`, `google-vertex`, `google-antigravity`, `google-gemini-cli`, `zai`, `openrouter`, `opencode`, `opencode-go`, `xai`, `groq`, `cerebras`, `mistral`, `github-copilot`
- Через `models.providers` (custom endpoint-и): `minimax` (хмара/API), а також будь-який OpenAI/Anthropic-сумісний проксі (LM Studio, vLLM, LiteLLM тощо)

Порада: не намагайтеся жорстко зафіксувати в документації «всі моделі». Авторитетний список — це те, що повертає `discoverModels(...)` на вашій машині, плюс ті ключі, які доступні.

## Облікові дані (ніколи не комітьте)

Live-тести знаходять облікові дані так само, як і CLI. Практичні наслідки:

- Якщо CLI працює, live-тести мають знайти ті самі ключі.
- Якщо live-тест каже «немає облікових даних», налагоджуйте це так само, як налагоджували б `openclaw models list` / вибір моделі.

- Профілі автентифікації для кожного агента: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` (саме це означає «profile keys» у live-тестах)
- Конфігурація: `~/.openclaw/openclaw.json` (або `OPENCLAW_CONFIG_PATH`)
- Каталог застарілого стану: `~/.openclaw/credentials/` (копіюється у staged live home, якщо присутній, але не є основним сховищем profile keys)
- Локальні live-запуски типово копіюють активну конфігурацію, файли `auth-profiles.json` для кожного агента, застарілий каталог `credentials/` і підтримувані зовнішні каталоги автентифікації CLI у тимчасовий test home; staged live home пропускають `workspace/` і `sandboxes/`, а override-и шляхів `agents.*.workspace` / `agentDir` видаляються, щоб probes не торкалися вашого реального host-workspace.

Якщо ви хочете покладатися на env-ключі (наприклад, експортовані у вашому `~/.profile`), запускайте локальні тести після `source ~/.profile` або використовуйте Docker-ранери нижче (вони можуть монтувати `~/.profile` у контейнер).

## Live: Deepgram (транскрибування аудіо)

- Тест: `src/media-understanding/providers/deepgram/audio.live.test.ts`
- Увімкнення: `DEEPGRAM_API_KEY=... DEEPGRAM_LIVE_TEST=1 pnpm test:live src/media-understanding/providers/deepgram/audio.live.test.ts`

## Live: BytePlus coding plan

- Тест: `src/agents/byteplus.live.test.ts`
- Увімкнення: `BYTEPLUS_API_KEY=... BYTEPLUS_LIVE_TEST=1 pnpm test:live src/agents/byteplus.live.test.ts`
- Необов’язковий override моделі: `BYTEPLUS_CODING_MODEL=ark-code-latest`

## Live: media для workflow ComfyUI

- Тест: `extensions/comfy/comfy.live.test.ts`
- Увімкнення: `OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts`
- Обсяг:
  - Перевіряє вбудовані шляхи comfy для зображень, відео та `music_generate`
  - Пропускає кожну можливість, якщо `models.providers.comfy.<capability>` не налаштовано
  - Корисно після змін у надсиланні workflow comfy, polling, завантаженнях або реєстрації plugin

## Live: генерація зображень

- Тест: `src/image-generation/runtime.live.test.ts`
- Команда: `pnpm test:live src/image-generation/runtime.live.test.ts`
- Harness: `pnpm test:live:media image`
- Обсяг:
  - Перелічує кожен зареєстрований plugin провайдера генерації зображень
  - Завантажує відсутні env-змінні провайдера з вашої login shell (`~/.profile`) перед виконанням probes
  - Типово використовує live/env API keys раніше за збережені профілі автентифікації, щоб застарілі тестові ключі в `auth-profiles.json` не маскували реальні shell-облікові дані
  - Пропускає провайдерів без придатної автентифікації/профілю/моделі
  - Запускає стандартні варіанти генерації зображень через спільну runtime-можливість:
    - `google:flash-generate`
    - `google:pro-generate`
    - `google:pro-edit`
    - `openai:default-generate`
- Поточні вбудовані провайдери, які охоплено:
  - `openai`
  - `google`
- Необов’язкове звуження:
  - `OPENCLAW_LIVE_IMAGE_GENERATION_PROVIDERS="openai,google"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_MODELS="openai/gpt-image-1,google/gemini-3.1-flash-image-preview"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_CASES="google:flash-generate,google:pro-edit"`
- Необов’язкова поведінка автентифікації:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`, щоб примусово використовувати автентифікацію зі сховища профілів і ігнорувати override-и лише через env

## Live: генерація музики

- Тест: `extensions/music-generation-providers.live.test.ts`
- Увімкнення: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts`
- Harness: `pnpm test:live:media music`
- Обсяг:
  - Перевіряє спільний вбудований шлях провайдера генерації музики
  - Наразі охоплює Google і MiniMax
  - Завантажує env-змінні провайдера з вашої login shell (`~/.profile`) перед виконанням probes
  - Типово використовує live/env API keys раніше за збережені профілі автентифікації, щоб застарілі тестові ключі в `auth-profiles.json` не маскували реальні shell-облікові дані
  - Пропускає провайдерів без придатної автентифікації/профілю/моделі
  - Запускає обидва оголошені runtime-режими, коли вони доступні:
    - `generate` з вхідними даними лише у вигляді prompt
    - `edit`, коли провайдер оголошує `capabilities.edit.enabled`
  - Поточне покриття спільної лінії:
    - `google`: `generate`, `edit`
    - `minimax`: `generate`
    - `comfy`: окремий live-файл Comfy, не цей спільний sweep
- Необов’язкове звуження:
  - `OPENCLAW_LIVE_MUSIC_GENERATION_PROVIDERS="google,minimax"`
  - `OPENCLAW_LIVE_MUSIC_GENERATION_MODELS="google/lyria-3-clip-preview,minimax/music-2.5+"`
- Необов’язкова поведінка автентифікації:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`, щоб примусово використовувати автентифікацію зі сховища профілів і ігнорувати override-и лише через env

## Live: генерація відео

- Тест: `extensions/video-generation-providers.live.test.ts`
- Увімкнення: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/video-generation-providers.live.test.ts`
- Harness: `pnpm test:live:media video`
- Обсяг:
  - Перевіряє спільний вбудований шлях провайдера генерації відео
  - Типово використовує безпечний для релізу шлях smoke: провайдери без FAL, один текст-у-відео запит на провайдера, one-second lobster prompt і обмеження операції на провайдера з `OPENCLAW_LIVE_VIDEO_GENERATION_TIMEOUT_MS` (типово `180000`)
  - Типово пропускає FAL, оскільки затримка черги на стороні провайдера може домінувати над часом релізу; передайте `--video-providers fal` або `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="fal"`, щоб запустити його явно
  - Завантажує env-змінні провайдера з вашої login shell (`~/.profile`) перед виконанням probes
  - Типово використовує live/env API keys раніше за збережені профілі автентифікації, щоб застарілі тестові ключі в `auth-profiles.json` не маскували реальні shell-облікові дані
  - Пропускає провайдерів без придатної автентифікації/профілю/моделі
  - Типово запускає лише `generate`
  - Установіть `OPENCLAW_LIVE_VIDEO_GENERATION_FULL_MODES=1`, щоб також запускати оголошені режими transform, коли вони доступні:
    - `imageToVideo`, коли провайдер оголошує `capabilities.imageToVideo.enabled` і вибраний провайдер/модель приймає локальний ввід зображень у буфері в межах спільного sweep
    - `videoToVideo`, коли провайдер оголошує `capabilities.videoToVideo.enabled` і вибраний провайдер/модель приймає локальний ввід відео у буфері в межах спільного sweep
  - Поточні провайдери `imageToVideo`, оголошені, але пропущені у спільному sweep:
    - `vydra`, оскільки вбудований `veo3` є лише текстовим, а вбудований `kling` потребує віддалений URL зображення
  - Покриття Vydra, специфічне для провайдера:
    - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_VYDRA_VIDEO=1 pnpm test:live -- extensions/vydra/vydra.live.test.ts`
    - цей файл запускає `veo3` text-to-video плюс лінію `kling`, яка типово використовує fixture з віддаленим URL зображення
  - Поточне live-покриття `videoToVideo`:
    - лише `runway`, коли вибрана модель — `runway/gen4_aleph`
  - Поточні провайдери `videoToVideo`, оголошені, але пропущені у спільному sweep:
    - `alibaba`, `qwen`, `xai`, оскільки ці шляхи зараз потребують віддалених reference URL `http(s)` / MP4
    - `google`, оскільки поточна спільна лінія Gemini/Veo використовує локальний ввід у буфері, і цей шлях не приймається у спільному sweep
    - `openai`, оскільки поточна спільна лінія не гарантує доступ до org-specific video inpaint/remix
- Необов’язкове звуження:
  - `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="google,openai,runway"`
  - `OPENCLAW_LIVE_VIDEO_GENERATION_MODELS="google/veo-3.1-fast-generate-preview,openai/sora-2,runway/gen4_aleph"`
  - `OPENCLAW_LIVE_VIDEO_GENERATION_SKIP_PROVIDERS=""`, щоб включити кожного провайдера до типового sweep, включно з FAL
  - `OPENCLAW_LIVE_VIDEO_GENERATION_TIMEOUT_MS=60000`, щоб зменшити ліміт операції для кожного провайдера для агресивного smoke-запуску
- Необов’язкова поведінка автентифікації:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`, щоб примусово використовувати автентифікацію зі сховища профілів і ігнорувати override-и лише через env

## Media live harness

- Команда: `pnpm test:live:media`
- Призначення:
  - Запускає спільні live-набори для зображень, музики та відео через один вбудований entrypoint репозиторію
  - Автоматично завантажує відсутні env-змінні провайдера з `~/.profile`
  - Типово автоматично звужує кожен набір до провайдерів, які зараз мають придатну автентифікацію
  - Повторно використовує `scripts/test-live.mjs`, тож поведінка Heartbeat і quiet mode залишається узгодженою
- Приклади:
  - `pnpm test:live:media`
  - `pnpm test:live:media image video --providers openai,google,minimax`
  - `pnpm test:live:media video --video-providers openai,runway --all-providers`
  - `pnpm test:live:media music --quiet`

## Docker-ранери (необов’язкові перевірки «працює в Linux»)

Ці Docker-ранери поділяються на дві категорії:

- Live-model ранери: `test:docker:live-models` і `test:docker:live-gateway` запускають лише відповідний live-файл з profile keys усередині Docker-образу репозиторію (`src/agents/models.profiles.live.test.ts` і `src/gateway/gateway-models.profiles.live.test.ts`), монтують ваш локальний каталог конфігурації та workspace (і використовують `~/.profile`, якщо його змонтовано). Відповідні локальні entrypoint-и: `test:live:models-profiles` і `test:live:gateway-profiles`.
- Docker live-ранери типово використовують менший smoke-ліміт, щоб повний Docker sweep залишався практичним:
  `test:docker:live-models` типово встановлює `OPENCLAW_LIVE_MAX_MODELS=12`, а
  `test:docker:live-gateway` типово встановлює `OPENCLAW_LIVE_GATEWAY_SMOKE=1`,
  `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=8`,
  `OPENCLAW_LIVE_GATEWAY_STEP_TIMEOUT_MS=45000` і
  `OPENCLAW_LIVE_GATEWAY_MODEL_TIMEOUT_MS=90000`. Перевизначайте ці env-змінні, коли
  вам явно потрібен більший вичерпний scan.
- `test:docker:all` один раз збирає live Docker-образ через `test:docker:live-build`, а потім повторно використовує його для двох live Docker-ліній.
- Container smoke-ранери: `test:docker:openwebui`, `test:docker:onboard`, `test:docker:gateway-network`, `test:docker:mcp-channels` і `test:docker:plugins` підіймають один або більше реальних контейнерів і перевіряють integration-шляхи вищого рівня.

Live-model Docker-ранери також bind-mount лише потрібні home-каталоги автентифікації CLI (або всі підтримувані, якщо запуск не звужено), а потім копіюють їх у home контейнера перед запуском, щоб OAuth зовнішніх CLI міг оновлювати токени без змін у host-сховищі автентифікації:

- Прямі моделі: `pnpm test:docker:live-models` (скрипт: `scripts/test-live-models-docker.sh`)
- ACP bind smoke: `pnpm test:docker:live-acp-bind` (скрипт: `scripts/test-live-acp-bind-docker.sh`)
- CLI backend smoke: `pnpm test:docker:live-cli-backend` (скрипт: `scripts/test-live-cli-backend-docker.sh`)
- Codex app-server harness smoke: `pnpm test:docker:live-codex-harness` (скрипт: `scripts/test-live-codex-harness-docker.sh`)
- Gateway + dev agent: `pnpm test:docker:live-gateway` (скрипт: `scripts/test-live-gateway-models-docker.sh`)
- Live smoke Open WebUI: `pnpm test:docker:openwebui` (скрипт: `scripts/e2e/openwebui-docker.sh`)
- Майстер онбордингу (TTY, повне scaffolding): `pnpm test:docker:onboard` (скрипт: `scripts/e2e/onboard-docker.sh`)
- Мережа Gateway (два контейнери, WS auth + health): `pnpm test:docker:gateway-network` (скрипт: `scripts/e2e/gateway-network-docker.sh`)
- Міст каналу MCP (seeded Gateway + stdio bridge + raw Claude notification-frame smoke): `pnpm test:docker:mcp-channels` (скрипт: `scripts/e2e/mcp-channels-docker.sh`)
- Plugins (smoke інсталяції + alias `/plugin` + семантика перезапуску Claude-bundle): `pnpm test:docker:plugins` (скрипт: `scripts/e2e/plugins-docker.sh`)

Live-model Docker-ранери також bind-mount поточний checkout лише для читання і
готує його в тимчасовий workdir усередині контейнера. Це зберігає runtime-образ
компактним, але водночас дозволяє запускати Vitest точно на вашому локальному source/config.
Етап staging пропускає великі локальні кеші та вихідні дані збірки застосунків, як-от
`.pnpm-store`, `.worktrees`, `__openclaw_vitest__` і локальні для застосунків каталоги `.build` або
виводу Gradle, щоб Docker live-запуски не витрачали хвилини на копіювання
machine-specific artifact-ів.
Вони також встановлюють `OPENCLAW_SKIP_CHANNELS=1`, щоб gateway live-probes не запускали
реальні worker-и каналів Telegram/Discord тощо всередині контейнера.
`test:docker:live-models` усе ще запускає `pnpm test:live`, тож також передавайте
`OPENCLAW_LIVE_GATEWAY_*`, коли вам потрібно звузити або виключити gateway
live-покриття з цієї Docker-лінії.
`test:docker:openwebui` — це smoke сумісності вищого рівня: він запускає
контейнер Gateway OpenClaw з увімкненими OpenAI-сумісними HTTP endpoint-ами,
запускає pinned-контейнер Open WebUI проти цього gateway, виконує вхід через
Open WebUI, перевіряє, що `/api/models` показує `openclaw/default`, а потім надсилає
реальний chat-запит через проксі `/api/chat/completions` Open WebUI.
Перший запуск може бути помітно повільнішим, оскільки Docker може потребувати завантаження
образу Open WebUI, а сам Open WebUI може завершувати власний cold-start setup.
Ця лінія очікує наявність придатного ключа live-моделі, а `OPENCLAW_PROFILE_FILE`
(типово `~/.profile`) є основним способом надати його в Dockerized-запусках.
Успішні запуски виводять невеликий JSON payload на кшталт `{ "ok": true, "model":
"openclaw/default", ... }`.
`test:docker:mcp-channels` навмисно є детермінованим і не потребує
реального облікового запису Telegram, Discord або iMessage. Він запускає seeded Gateway
контейнер, стартує другий контейнер, який запускає `openclaw mcp serve`, а потім
перевіряє виявлення conversation із маршрутизацією, читання transcript-ів, metadata вкладень,
поведінку черги live-подій, маршрутизацію outbound send, а також channel- і
permission-сповіщення у стилі Claude через реальний stdio MCP bridge. Перевірка сповіщень
напряму досліджує raw stdio MCP frame-и, тож smoke перевіряє те, що міст
справді випромінює, а не лише те, що випадково показує конкретний client SDK.

Ручний smoke plain-language thread для ACP (не CI):

- `bun scripts/dev/discord-acp-plain-language-smoke.ts --channel <discord-channel-id> ...`
- Зберігайте цей скрипт для регресійних сценаріїв і налагодження. Він може знову знадобитися для перевірки маршрутизації ACP thread, тому не видаляйте його.

Корисні env-змінні:

- `OPENCLAW_CONFIG_DIR=...` (типово: `~/.openclaw`) монтується в `/home/node/.openclaw`
- `OPENCLAW_WORKSPACE_DIR=...` (типово: `~/.openclaw/workspace`) монтується в `/home/node/.openclaw/workspace`
- `OPENCLAW_PROFILE_FILE=...` (типово: `~/.profile`) монтується в `/home/node/.profile` і використовується перед запуском тестів
- `OPENCLAW_DOCKER_CLI_TOOLS_DIR=...` (типово: `~/.cache/openclaw/docker-cli-tools`) монтується в `/home/node/.npm-global` для кешованих установок CLI усередині Docker
- Зовнішні каталоги/файли автентифікації CLI в `$HOME` монтуються лише для читання під `/host-auth...`, а потім копіюються в `/home/node/...` перед початком тестів
  - Типові каталоги: `.minimax`
  - Типові файли: `~/.codex/auth.json`, `~/.codex/config.toml`, `.claude.json`, `~/.claude/.credentials.json`, `~/.claude/settings.json`, `~/.claude/settings.local.json`
  - Звужені запуски провайдерів монтують лише потрібні каталоги/файли, виведені з `OPENCLAW_LIVE_PROVIDERS` / `OPENCLAW_LIVE_GATEWAY_PROVIDERS`
  - Перевизначайте вручну через `OPENCLAW_DOCKER_AUTH_DIRS=all`, `OPENCLAW_DOCKER_AUTH_DIRS=none` або список через кому, наприклад `OPENCLAW_DOCKER_AUTH_DIRS=.claude,.codex`
- `OPENCLAW_LIVE_GATEWAY_MODELS=...` / `OPENCLAW_LIVE_MODELS=...`, щоб звузити запуск
- `OPENCLAW_LIVE_GATEWAY_PROVIDERS=...` / `OPENCLAW_LIVE_PROVIDERS=...`, щоб відфільтрувати провайдерів у контейнері
- `OPENCLAW_SKIP_DOCKER_BUILD=1`, щоб повторно використати наявний образ `openclaw:local-live` для повторних запусків без потреби в перебудові
- `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`, щоб гарантувати, що облікові дані беруться зі сховища профілів (а не з env)
- `OPENCLAW_OPENWEBUI_MODEL=...`, щоб вибрати модель, яку gateway показує для smoke Open WebUI
- `OPENCLAW_OPENWEBUI_PROMPT=...`, щоб перевизначити nonce-check prompt, використаний у smoke Open WebUI
- `OPENWEBUI_IMAGE=...`, щоб перевизначити pinned tag образу Open WebUI

## Перевірка документації

Запускайте перевірки docs після редагування документації: `pnpm check:docs`.
Запускайте повну перевірку anchor у Mintlify, коли вам також потрібні перевірки заголовків у межах сторінки: `pnpm docs:check-links:anchors`.

## Офлайн-регресія (безпечна для CI)

Це регресії «реального pipeline» без реальних провайдерів:

- Виклик інструментів Gateway (mock OpenAI, реальний цикл gateway + agent): `src/gateway/gateway.test.ts` (випадок: "runs a mock OpenAI tool call end-to-end via gateway agent loop")
- Майстер Gateway (WS `wizard.start`/`wizard.next`, запис конфігурації + примусова auth): `src/gateway/gateway.test.ts` (випадок: "runs wizard over ws and writes auth token config")

## Оцінювання надійності агентів (Skills)

У нас уже є кілька безпечних для CI тестів, які поводяться як «оцінювання надійності агентів»:

- Mock-виклик інструментів через реальний цикл gateway + agent (`src/gateway/gateway.test.ts`).
- Наскрізні потоки майстра, які перевіряють wiring сесії та ефекти конфігурації (`src/gateway/gateway.test.ts`).

Що ще бракує для Skills (див. [Skills](/uk/tools/skills)):

- **Прийняття рішень:** коли Skills перелічені в prompt, чи вибирає агент правильний skill (або уникає нерелевантних)?
- **Дотримання вимог:** чи читає агент `SKILL.md` перед використанням і чи виконує потрібні кроки/аргументи?
- **Контракти workflow:** багатокрокові сценарії, які перевіряють порядок інструментів, перенесення історії сесії та межі sandbox.

Майбутні eval-и мають насамперед залишатися детермінованими:

- Runner сценаріїв з mock-провайдерами для перевірки викликів інструментів + їх порядку, читання skill-файлів і wiring сесій.
- Невеликий набір сценаріїв, сфокусованих на skills (використовувати чи уникати, gating, prompt injection).
- Необов’язкові live eval-и (opt-in, керовані env) — лише після появи безпечного для CI набору.

## Контрактні тести (форма Plugin і каналу)

Контрактні тести перевіряють, що кожен зареєстрований Plugin і канал відповідає своєму
контракту інтерфейсу. Вони перебирають усі виявлені plugins і запускають набір
перевірок форми та поведінки. Типова unit-лінія `pnpm test` навмисно
пропускає ці спільні seam- і smoke-файли; запускайте контрактні команди явно,
коли торкаєтеся спільних поверхонь каналу або провайдера.

### Команди

- Усі контракти: `pnpm test:contracts`
- Лише контракти каналів: `pnpm test:contracts:channels`
- Лише контракти провайдерів: `pnpm test:contracts:plugins`

### Контракти каналів

Розташовані в `src/channels/plugins/contracts/*.contract.test.ts`:

- **plugin** - Базова форма plugin (id, name, capabilities)
- **setup** - Контракт майстра налаштування
- **session-binding** - Поведінка прив’язки сесії
- **outbound-payload** - Структура payload повідомлення
- **inbound** - Обробка вхідних повідомлень
- **actions** - Обробники дій каналу
- **threading** - Обробка ID thread
- **directory** - API каталогу/реєстру
- **group-policy** - Забезпечення політики групи

### Контракти стану провайдерів

Розташовані в `src/plugins/contracts/*.contract.test.ts`.

- **status** - Probes стану каналу
- **registry** - Форма реєстру Plugin

### Контракти провайдерів

Розташовані в `src/plugins/contracts/*.contract.test.ts`:

- **auth** - Контракт потоку автентифікації
- **auth-choice** - Вибір/селекція автентифікації
- **catalog** - API каталогу моделей
- **discovery** - Виявлення Plugin
- **loader** - Завантаження Plugin
- **runtime** - Runtime провайдера
- **shape** - Форма/інтерфейс Plugin
- **wizard** - Майстер налаштування

### Коли запускати

- Після змін export-ів або subpath-ів plugin-sdk
- Після додавання або зміни channel чи provider plugin
- Після рефакторингу реєстрації Plugin або виявлення

Контрактні тести запускаються в CI й не потребують реальних API key.

## Додавання регресій (рекомендації)

Коли ви виправляєте проблему провайдера/моделі, виявлену в live:

- Додайте безпечну для CI регресію, якщо це можливо (mock/stub провайдера або фіксація точної трансформації форми запиту)
- Якщо проблема за своєю природою лише live (rate limit, політики auth), залишайте live-тест вузьким і opt-in через env-змінні
- Віддавайте перевагу найменшому шару, який ловить помилку:
  - помилка конвертації/повторного відтворення запиту провайдера → тест прямих моделей
  - помилка pipeline gateway session/history/tool → gateway live smoke або безпечний для CI mock-тест gateway
- Guardrail для обходу SecretRef:
  - `src/secrets/exec-secret-ref-id-parity.test.ts` виводить одну вибіркову ціль на кожен клас SecretRef з metadata реєстру (`listSecretTargetRegistryEntries()`), а потім перевіряє, що traversal-segment exec id відхиляються.
  - Якщо ви додаєте нове сімейство цілей SecretRef `includeInPlan` у `src/secrets/target-registry-data.ts`, оновіть `classifyTargetClass` у цьому тесті. Тест навмисно завершується помилкою для некласифікованих target id, щоб нові класи не могли бути тихо пропущені.
