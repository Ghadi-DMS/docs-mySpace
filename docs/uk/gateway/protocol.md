---
read_when:
    - Реалізація або оновлення gateway WS-клієнтів
    - Налагодження невідповідностей протоколу або збоїв підключення
    - Повторна генерація схеми/моделей протоколу
summary: 'Протокол Gateway WebSocket: рукостискання, фрейми, версіонування'
title: Протокол Gateway
x-i18n:
    generated_at: "2026-04-07T19:40:42Z"
    model: gpt-5.4
    provider: openai
    source_hash: 8635e3ac1dd311dbd3a770b088868aa1495a8d53b3ebc1eae0dfda3b2bf4694a
    source_path: gateway/protocol.md
    workflow: 15
---

# Протокол Gateway (WebSocket)

Протокол Gateway WS — це **єдина контрольна площина + транспорт вузлів** для
OpenClaw. Усі клієнти (CLI, веб-інтерфейс, застосунок macOS, вузли iOS/Android, headless
вузли) підключаються через WebSocket і оголошують свою **роль** + **область**
під час рукостискання.

## Транспорт

- WebSocket, текстові фрейми з JSON-навантаженнями.
- Першим фреймом **має** бути запит `connect`.

## Рукостискання (connect)

Gateway → Client (передпідключний challenge):

```json
{
  "type": "event",
  "event": "connect.challenge",
  "payload": { "nonce": "…", "ts": 1737264000000 }
}
```

Client → Gateway:

```json
{
  "type": "req",
  "id": "…",
  "method": "connect",
  "params": {
    "minProtocol": 3,
    "maxProtocol": 3,
    "client": {
      "id": "cli",
      "version": "1.2.3",
      "platform": "macos",
      "mode": "operator"
    },
    "role": "operator",
    "scopes": ["operator.read", "operator.write"],
    "caps": [],
    "commands": [],
    "permissions": {},
    "auth": { "token": "…" },
    "locale": "en-US",
    "userAgent": "openclaw-cli/1.2.3",
    "device": {
      "id": "device_fingerprint",
      "publicKey": "…",
      "signature": "…",
      "signedAt": 1737264000000,
      "nonce": "…"
    }
  }
}
```

Gateway → Client:

```json
{
  "type": "res",
  "id": "…",
  "ok": true,
  "payload": { "type": "hello-ok", "protocol": 3, "policy": { "tickIntervalMs": 15000 } }
}
```

Коли видається токен пристрою, `hello-ok` також містить:

```json
{
  "auth": {
    "deviceToken": "…",
    "role": "operator",
    "scopes": ["operator.read", "operator.write"]
  }
}
```

Під час передавання довіреного bootstrap `hello-ok.auth` також може містити додаткові
обмежені записи ролей у `deviceTokens`:

```json
{
  "auth": {
    "deviceToken": "…",
    "role": "node",
    "scopes": [],
    "deviceTokens": [
      {
        "deviceToken": "…",
        "role": "operator",
        "scopes": ["operator.approvals", "operator.read", "operator.talk.secrets", "operator.write"]
      }
    ]
  }
}
```

Для вбудованого bootstrap-процесу node/operator основний токен вузла зберігає
`scopes: []`, а будь-який переданий токен оператора залишається обмеженим
списком дозволів bootstrap-оператора (`operator.approvals`, `operator.read`,
`operator.talk.secrets`, `operator.write`). Перевірки області bootstrap
залишаються префіксованими роллю: записи оператора задовольняють лише запити
оператора, а ролям, що не є оператором, як і раніше потрібні області під їхнім
власним префіксом ролі.

### Приклад node

```json
{
  "type": "req",
  "id": "…",
  "method": "connect",
  "params": {
    "minProtocol": 3,
    "maxProtocol": 3,
    "client": {
      "id": "ios-node",
      "version": "1.2.3",
      "platform": "ios",
      "mode": "node"
    },
    "role": "node",
    "scopes": [],
    "caps": ["camera", "canvas", "screen", "location", "voice"],
    "commands": ["camera.snap", "canvas.navigate", "screen.record", "location.get"],
    "permissions": { "camera.capture": true, "screen.record": false },
    "auth": { "token": "…" },
    "locale": "en-US",
    "userAgent": "openclaw-ios/1.2.3",
    "device": {
      "id": "device_fingerprint",
      "publicKey": "…",
      "signature": "…",
      "signedAt": 1737264000000,
      "nonce": "…"
    }
  }
}
```

## Форматування фреймів

- **Request**: `{type:"req", id, method, params}`
- **Response**: `{type:"res", id, ok, payload|error}`
- **Event**: `{type:"event", event, payload, seq?, stateVersion?}`

Методи з побічними ефектами потребують **ключів ідемпотентності** (див. schema).

## Ролі + області

### Ролі

- `operator` = клієнт контрольної площини (CLI/UI/автоматизація).
- `node` = хост можливостей (camera/screen/canvas/system.run).

### Області (operator)

Поширені області:

- `operator.read`
- `operator.write`
- `operator.admin`
- `operator.approvals`
- `operator.pairing`
- `operator.talk.secrets`

`talk.config` з `includeSecrets: true` потребує `operator.talk.secrets`
(або `operator.admin`).

RPC-методи Gateway, зареєстровані plugin, можуть запитувати власну область
оператора, але зарезервовані префікси core admin (`config.*`, `exec.approvals.*`, `wizard.*`,
`update.*`) завжди зіставляються з `operator.admin`.

Область методу — це лише перший рівень перевірки. Деякі slash-команди,
викликані через `chat.send`, додатково застосовують суворіші перевірки на рівні команди.
Наприклад, постійні записи `/config set` і `/config unset` потребують `operator.admin`.

`node.pair.approve` також має додаткову перевірку області під час схвалення понад
базову область методу:

- запити без команди: `operator.pairing`
- запити з командами вузла без exec: `operator.pairing` + `operator.write`
- запити, що містять `system.run`, `system.run.prepare` або `system.which`:
  `operator.pairing` + `operator.admin`

### Caps/commands/permissions (node)

Вузли оголошують заявлені можливості під час підключення:

- `caps`: високорівневі категорії можливостей.
- `commands`: список дозволених команд для invoke.
- `permissions`: детальні перемикачі (наприклад, `screen.record`, `camera.capture`).

Gateway розглядає це як **заяви** і застосовує серверні allowlist.

## Присутність

- `system-presence` повертає записи з ключами за ідентичністю пристрою.
- Записи присутності містять `deviceId`, `roles` і `scopes`, щоб UI міг показувати один рядок на пристрій
  навіть коли він підключається і як **operator**, і як **node**.

## Поширені сімейства RPC-методів

Ця сторінка не є згенерованим повним дампом, але публічна поверхня WS ширша,
ніж наведені вище приклади рукостискання/автентифікації. Це основні сімейства методів,
які сьогодні надає Gateway.

`hello-ok.features.methods` — це консервативний список виявлення, побудований з
`src/gateway/server-methods-list.ts` плюс експортів методів завантажених plugin/channel.
Сприймайте його як виявлення можливостей, а не як згенерований дамп кожного допоміжного виклику,
реалізованого в `src/gateway/server-methods/*.ts`.

### System and identity

- `health` повертає кешований або щойно перевірений знімок стану gateway.
- `status` повертає зведення gateway у стилі `/status`; чутливі поля
  включаються лише для operator-клієнтів з admin-областю.
- `gateway.identity.get` повертає ідентичність пристрою gateway, що використовується relay і
  потоками pairing.
- `system-presence` повертає поточний знімок присутності для підключених
  пристроїв operator/node.
- `system-event` додає системну подію та може оновлювати/транслювати контекст
  присутності.
- `last-heartbeat` повертає останню збережену подію heartbeat.
- `set-heartbeats` перемикає обробку heartbeat на gateway.

### Models and usage

- `models.list` повертає каталог моделей, дозволених під час виконання.
- `usage.status` повертає зведення вікон використання постачальника/залишку квоти.
- `usage.cost` повертає агреговані зведення вартості використання за діапазон дат.
- `doctor.memory.status` повертає готовність vector-memory / embedding для
  активного робочого простору агента за замовчуванням.
- `sessions.usage` повертає зведення використання по сесіях.
- `sessions.usage.timeseries` повертає часовий ряд використання для однієї сесії.
- `sessions.usage.logs` повертає записи журналу використання для однієї сесії.

### Channels and login helpers

- `channels.status` повертає зведення стану вбудованих і bundled channel/plugin.
- `channels.logout` виконує вихід для конкретного channel/account, де channel
  підтримує вихід.
- `web.login.start` запускає потік входу QR/web для поточного QR-сумісного веб-
  провайдера channel.
- `web.login.wait` очікує завершення цього потоку входу QR/web і запускає
  channel у разі успіху.
- `push.test` надсилає тестовий APNs push до зареєстрованого вузла iOS.
- `voicewake.get` повертає збережені тригери wake-word.
- `voicewake.set` оновлює тригери wake-word і транслює зміну.

### Messaging and logs

- `send` — це RPC прямої вихідної доставки для channel/account/thread-орієнтованих
  надсилань поза chat runner.
- `logs.tail` повертає налаштований tail файлового журналу gateway з cursor/limit і
  елементами керування max-byte.

### Talk and TTS

- `talk.config` повертає ефективне конфігураційне навантаження Talk; `includeSecrets`
  потребує `operator.talk.secrets` (або `operator.admin`).
- `talk.mode` встановлює/транслює поточний стан режиму Talk для клієнтів WebChat/Control UI.
- `talk.speak` синтезує мовлення через активного speech-провайдера Talk.
- `tts.status` повертає стан увімкнення TTS, активного провайдера, резервних провайдерів
  і стан конфігурації провайдера.
- `tts.providers` повертає видимий інвентар TTS-провайдерів.
- `tts.enable` і `tts.disable` перемикають стан налаштувань TTS.
- `tts.setProvider` оновлює бажаного TTS-провайдера.
- `tts.convert` виконує одноразове перетворення text-to-speech.

### Secrets, config, update, and wizard

- `secrets.reload` повторно розв’язує активні SecretRef і замінює стан секретів під час виконання
  лише за повного успіху.
- `secrets.resolve` розв’язує призначення секретів цільовій команді для конкретного
  набору command/target.
- `config.get` повертає поточний знімок config і hash.
- `config.set` записує перевірене навантаження config.
- `config.patch` об’єднує часткове оновлення config.
- `config.apply` перевіряє й замінює повне навантаження config.
- `config.schema` повертає живе навантаження схеми config, що використовується Control UI і
  інструментами CLI: schema, `uiHints`, версію та метадані генерації, включно з
  метаданими схем plugin + channel, коли середовище виконання може їх завантажити. Схема
  містить метадані полів `title` / `description`, похідні від тих самих міток
  і тексту довідки, які використовує UI, включно з вкладеними object, wildcard,
  array-item і гілками композиції `anyOf` / `oneOf` / `allOf`, коли існує
  відповідна документація поля.
- `config.schema.lookup` повертає навантаження пошуку з обмеженням шляхом для одного шляху config:
  нормалізований шлях, неглибокий вузол schema, зіставлені hint + `hintPath` і
  зведення безпосередніх дочірніх елементів для деталізації в UI/CLI.
  - Вузли схеми пошуку зберігають орієнтовану на користувача документацію та поширені поля валідації:
    `title`, `description`, `type`, `enum`, `const`, `format`, `pattern`,
    числові/рядкові/масивні/об’єктні межі та булеві прапорці, як-от
    `additionalProperties`, `deprecated`, `readOnly`, `writeOnly`.
  - Зведення дочірніх елементів показують `key`, нормалізований `path`, `type`, `required`,
    `hasChildren`, а також зіставлені `hint` / `hintPath`.
- `update.run` запускає потік оновлення gateway і планує перезапуск лише тоді,
  коли саме оновлення було успішним.
- `wizard.start`, `wizard.next`, `wizard.status` і `wizard.cancel` надають
  wizard онбордингу через WS RPC.

### Existing major families

#### Agent and workspace helpers

- `agents.list` повертає налаштовані записи агентів.
- `agents.create`, `agents.update` і `agents.delete` керують записами агентів і
  підключенням робочих просторів.
- `agents.files.list`, `agents.files.get` і `agents.files.set` керують файлами
  bootstrap-робочого простору, доступними для агента.
- `agent.identity.get` повертає ефективну ідентичність асистента для агента або
  сесії.
- `agent.wait` очікує завершення запуску та повертає фінальний знімок, коли він
  доступний.

#### Session control

- `sessions.list` повертає поточний індекс сесій.
- `sessions.subscribe` і `sessions.unsubscribe` перемикають підписки на події змін сесій
  для поточного WS-клієнта.
- `sessions.messages.subscribe` і `sessions.messages.unsubscribe` перемикають
  підписки на події transcript/message для однієї сесії.
- `sessions.preview` повертає обмежені попередні перегляди transcript для конкретних
  ключів сесій.
- `sessions.resolve` розв’язує або канонізує ціль сесії.
- `sessions.create` створює новий запис сесії.
- `sessions.send` надсилає повідомлення в наявну сесію.
- `sessions.steer` — це варіант переривання й спрямування для активної сесії.
- `sessions.abort` перериває активну роботу для сесії.
- `sessions.patch` оновлює метадані/перевизначення сесії.
- `sessions.reset`, `sessions.delete` і `sessions.compact` виконують обслуговування сесії.
- `sessions.get` повертає повний збережений рядок сесії.
- виконання chat, як і раніше, використовує `chat.history`, `chat.send`, `chat.abort` і
  `chat.inject`.
- `chat.history` нормалізується для відображення для UI-клієнтів: вбудовані теги директив
  видаляються з видимого тексту, XML-навантаження виклику інструментів у простому тексті (включно з
  `<tool_call>...</tool_call>`, `<function_call>...</function_call>`,
  `<tool_calls>...</tool_calls>`, `<function_calls>...</function_calls>` і
  обрізаними блоками виклику інструментів) та витеклі ASCII/повноширинні токени керування моделі
  видаляються, суто беззвучні рядки асистента, як-от точні `NO_REPLY` /
  `no_reply`, пропускаються, а надто великі рядки можуть замінюватися заповнювачами.

#### Device pairing and device tokens

- `device.pair.list` повертає очікуючі й схвалені спарені пристрої.
- `device.pair.approve`, `device.pair.reject` і `device.pair.remove` керують
  записами спарювання пристроїв.
- `device.token.rotate` ротатує токен спареного пристрою в межах схвалених ролей
  і областей.
- `device.token.revoke` відкликає токен пристрою.

#### Node pairing, invoke, and pending work

- `node.pair.request`, `node.pair.list`, `node.pair.approve`,
  `node.pair.reject` і `node.pair.verify` охоплюють спарювання вузлів і bootstrap-
  перевірку.
- `node.list` і `node.describe` повертають стан відомих/підключених вузлів.
- `node.rename` оновлює мітку спареного вузла.
- `node.invoke` пересилає команду підключеному вузлу.
- `node.invoke.result` повертає результат для запиту invoke.
- `node.event` переносить події, ініційовані вузлом, назад у gateway.
- `node.canvas.capability.refresh` оновлює scoped-токени можливостей canvas.
- `node.pending.pull` і `node.pending.ack` — це API черги підключених вузлів.
- `node.pending.enqueue` і `node.pending.drain` керують довговічною відкладеною роботою
  для офлайн/відключених вузлів.

#### Approval families

- `exec.approval.request`, `exec.approval.get`, `exec.approval.list` і
  `exec.approval.resolve` охоплюють одноразові запити на схвалення exec плюс
  пошук/повторення очікуючих схвалень.
- `exec.approval.waitDecision` очікує рішення щодо одного очікуючого схвалення exec і повертає
  фінальне рішення (або `null` у разі тайм-ауту).
- `exec.approvals.get` і `exec.approvals.set` керують знімками політики схвалення exec
  gateway.
- `exec.approvals.node.get` і `exec.approvals.node.set` керують локальною для вузла exec-
  політикою схвалення через relay-команди вузла.
- `plugin.approval.request`, `plugin.approval.list`,
  `plugin.approval.waitDecision` і `plugin.approval.resolve` охоплюють
  потоки схвалення, визначені plugin.

#### Other major families

- automation:
  - `wake` планує негайне або на наступний heartbeat введення тексту пробудження
  - `cron.list`, `cron.status`, `cron.add`, `cron.update`, `cron.remove`,
    `cron.run`, `cron.runs`
- skills/tools: `skills.*`, `tools.catalog`, `tools.effective`

### Поширені сімейства подій

- `chat`: оновлення chat у UI, такі як `chat.inject` та інші події chat,
  пов’язані лише з transcript.
- `session.message` і `session.tool`: оновлення transcript/потоку подій для
  підписаної сесії.
- `sessions.changed`: змінився індекс сесій або метадані.
- `presence`: оновлення знімка системної присутності.
- `tick`: періодична подія keepalive / liveness.
- `health`: оновлення знімка стану gateway.
- `heartbeat`: оновлення потоку подій heartbeat.
- `cron`: подія зміни запуску/завдання cron.
- `shutdown`: сповіщення про вимкнення gateway.
- `node.pair.requested` / `node.pair.resolved`: життєвий цикл спарювання вузлів.
- `node.invoke.request`: трансляція запиту invoke вузла.
- `device.pair.requested` / `device.pair.resolved`: життєвий цикл спареного пристрою.
- `voicewake.changed`: змінилася конфігурація тригерів wake-word.
- `exec.approval.requested` / `exec.approval.resolved`: життєвий цикл
  схвалення exec.
- `plugin.approval.requested` / `plugin.approval.resolved`: життєвий цикл схвалення
  plugin.

### Допоміжні методи node

- Вузли можуть викликати `skills.bins`, щоб отримати поточний список виконуваних файлів skill
  для автоматичних перевірок allow.

### Допоміжні методи operator

- Оператори можуть викликати `tools.catalog` (`operator.read`), щоб отримати каталог runtime tools для
  агента. Відповідь містить згруповані tools і метадані походження:
  - `source`: `core` або `plugin`
  - `pluginId`: власник plugin, коли `source="plugin"`
  - `optional`: чи є інструмент plugin необов’язковим
- Оператори можуть викликати `tools.effective` (`operator.read`), щоб отримати runtime-effective
  інвентар tools для сесії.
  - `sessionKey` є обов’язковим.
  - Gateway виводить довірений контекст runtime із сесії на боці сервера, а не приймає
    auth або контекст доставки, надані викликачем.
  - Відповідь має область сесії та відображає те, що активна розмова може використовувати просто зараз,
    включно з core, plugin і channel tools.
- Оператори можуть викликати `skills.status` (`operator.read`), щоб отримати видимий
  інвентар Skills для агента.
  - `agentId` необов’язковий; пропустіть його, щоб читати робочий простір агента за замовчуванням.
  - Відповідь містить право на використання, відсутні вимоги, перевірки config і
    очищені параметри встановлення без розкриття сирих значень секретів.
- Оператори можуть викликати `skills.search` і `skills.detail` (`operator.read`) для
  метаданих виявлення ClawHub.
- Оператори можуть викликати `skills.install` (`operator.admin`) у двох режимах:
  - Режим ClawHub: `{ source: "clawhub", slug, version?, force? }` установлює
    теку skill у каталог `skills/` робочого простору агента за замовчуванням.
  - Режим інсталятора gateway: `{ name, installId, dangerouslyForceUnsafeInstall?, timeoutMs? }`
    запускає оголошену дію `metadata.openclaw.install` на хості gateway.
- Оператори можуть викликати `skills.update` (`operator.admin`) у двох режимах:
  - Режим ClawHub оновлює один відстежуваний slug або всі відстежувані інсталяції ClawHub у
    робочому просторі агента за замовчуванням.
  - Режим Config патчить значення `skills.entries.<skillKey>`, такі як `enabled`,
    `apiKey` і `env`.

## Схвалення exec

- Коли запит exec потребує схвалення, gateway транслює `exec.approval.requested`.
- Клієнти operator виконують розв’язання, викликаючи `exec.approval.resolve` (потребує області `operator.approvals`).
- Для `host=node` `exec.approval.request` має містити `systemRunPlan` (канонічні `argv`/`cwd`/`rawCommand`/метадані сесії). Запити без `systemRunPlan` відхиляються.
- Після схвалення переслані виклики `node.invoke system.run` повторно використовують цей канонічний
  `systemRunPlan` як авторитетний контекст команди/cwd/сесії.
- Якщо викликач змінює `command`, `rawCommand`, `cwd`, `agentId` або
  `sessionKey` між підготовкою та фінальним пересланням схваленого `system.run`, gateway
  відхиляє запуск замість довіри до зміненого навантаження.

## Резервна доставка агента

- Запити `agent` можуть включати `deliver=true`, щоб запросити вихідну доставку.
- `bestEffortDeliver=false` зберігає строгий режим: нерозв’язані або лише внутрішні цілі доставки повертають `INVALID_REQUEST`.
- `bestEffortDeliver=true` дозволяє резервне виконання лише в межах сесії, коли не вдається розв’язати жоден зовнішній маршрут доставки (наприклад, для internal/webchat-сесій або неоднозначних конфігурацій багатьох channel).

## Версіонування

- `PROTOCOL_VERSION` розташований у `src/gateway/protocol/schema.ts`.
- Клієнти надсилають `minProtocol` + `maxProtocol`; сервер відхиляє невідповідності.
- Schema + models генеруються з визначень TypeBox:
  - `pnpm protocol:gen`
  - `pnpm protocol:gen:swift`
  - `pnpm protocol:check`

## Автентифікація

- Автентифікація gateway за спільним секретом використовує `connect.params.auth.token` або
  `connect.params.auth.password` залежно від налаштованого режиму auth.
- Режими з носієм ідентичності, такі як Tailscale Serve
  (`gateway.auth.allowTailscale: true`) або non-loopback
  `gateway.auth.mode: "trusted-proxy"`, задовольняють перевірку auth під час connect з
  заголовків запиту замість `connect.params.auth.*`.
- `gateway.auth.mode: "none"` для private-ingress повністю пропускає shared-secret connect auth; не відкривайте цей режим для публічного/недовіреного ingress.
- Після pairing Gateway видає **токен пристрою**, прив’язаний до ролі + областей
  підключення. Він повертається в `hello-ok.auth.deviceToken` і має
  зберігатися клієнтом для майбутніх підключень.
- Клієнти повинні зберігати основний `hello-ok.auth.deviceToken` після будь-якого
  успішного підключення.
- Повторне підключення з цим **збереженим** токеном пристрою також повинно повторно використовувати
  збережений схвалений набір областей для цього токена. Це зберігає вже наданий
  доступ на читання/перевірку/статус і запобігає непомітному звуженню повторних підключень до
  вужчої неявної admin-only області.
- Звичайний пріоритет auth під час connect: спочатку явний спільний token/password, потім
  явний `deviceToken`, потім збережений токен для пристрою, потім bootstrap-токен.
- Додаткові записи `hello-ok.auth.deviceTokens` — це токени передавання bootstrap.
  Зберігайте їх лише тоді, коли підключення використовувало bootstrap auth через довірений транспорт,
  такий як `wss://` або loopback/local pairing.
- Якщо клієнт передає **явний** `deviceToken` або явні `scopes`, цей
  запитаний викликачем набір областей залишається авторитетним; кешовані області повторно використовуються
  лише тоді, коли клієнт повторно використовує збережений токен для пристрою.
- Токени пристроїв можна ротувати/відкликати через `device.token.rotate` і
  `device.token.revoke` (потребує області `operator.pairing`).
- Видача/ротація токенів залишається обмеженою схваленим набором ролей, записаним у
  записі pairing цього пристрою; ротація токена не може розширити пристрій до
  ролі, яку схвалення pairing ніколи не надавало.
- Для paired-device сеансів токенів керування пристроями має власну область, якщо
  викликач також не має `operator.admin`: викликачі без admin можуть видаляти/відкликати/ротувати
  лише **власний** запис пристрою.
- `device.token.rotate` також перевіряє запитаний набір областей оператора відносно
  поточних областей сеансу викликача. Викликачі без admin не можуть ротувати токен до
  ширшого набору областей оператора, ніж той, який вони вже мають.
- Помилки auth містять `error.details.code` плюс підказки щодо відновлення:
  - `error.details.canRetryWithDeviceToken` (boolean)
  - `error.details.recommendedNextStep` (`retry_with_device_token`, `update_auth_configuration`, `update_auth_credentials`, `wait_then_retry`, `review_auth_configuration`)
- Поведінка клієнта для `AUTH_TOKEN_MISMATCH`:
  - Довірені клієнти можуть виконати одну обмежену повторну спробу з кешованим токеном для пристрою.
  - Якщо ця повторна спроба не вдається, клієнти повинні припинити автоматичні цикли повторного підключення й показати оператору вказівки щодо потрібних дій.

## Ідентичність пристрою + pairing

- Вузли повинні включати стабільну ідентичність пристрою (`device.id`), похідну від
  відбитка fingerprint keypair.
- Gateway видає токени для кожного пристрою + ролі.
- Для нових `device.id` потрібні схвалення pairing, якщо не ввімкнено локальне автосхвалення.
- Автосхвалення pairing зосереджене на прямих локальних loopback-підключеннях.
- OpenClaw також має вузький шлях самопідключення backend/container-local для
  довірених helper-потоків зі спільним секретом.
- Підключення tailnet або LAN на тому ж хості все одно вважаються віддаленими для pairing і
  потребують схвалення.
- Усі WS-клієнти повинні включати ідентичність `device` під час `connect` (operator + node).
  Control UI може пропускати її лише в таких режимах:
  - `gateway.controlUi.allowInsecureAuth=true` для сумісності з небезпечним HTTP лише для localhost.
  - успішна auth Control UI оператора з `gateway.auth.mode: "trusted-proxy"`.
  - `gateway.controlUi.dangerouslyDisableDeviceAuth=true` (аварійний режим, серйозне зниження безпеки).
- Усі підключення повинні підписувати наданий сервером nonce `connect.challenge`.

### Діагностика міграції автентифікації пристрою

Для застарілих клієнтів, які все ще використовують поведінку підпису до challenge, `connect` тепер повертає
коди деталей `DEVICE_AUTH_*` у `error.details.code` зі стабільним `error.details.reason`.

Поширені збої міграції:

| Message                     | details.code                     | details.reason           | Meaning                                            |
| --------------------------- | -------------------------------- | ------------------------ | -------------------------------------------------- |
| `device nonce required`     | `DEVICE_AUTH_NONCE_REQUIRED`     | `device-nonce-missing`   | Клієнт пропустив `device.nonce` (або надіслав порожнє значення). |
| `device nonce mismatch`     | `DEVICE_AUTH_NONCE_MISMATCH`     | `device-nonce-mismatch`  | Клієнт підписав застарілим/неправильним nonce.            |
| `device signature invalid`  | `DEVICE_AUTH_SIGNATURE_INVALID`  | `device-signature`       | Навантаження підпису не відповідає навантаженню v2.       |
| `device signature expired`  | `DEVICE_AUTH_SIGNATURE_EXPIRED`  | `device-signature-stale` | Підписана позначка часу поза дозволеним зсувом.          |
| `device identity mismatch`  | `DEVICE_AUTH_DEVICE_ID_MISMATCH` | `device-id-mismatch`     | `device.id` не відповідає відбитку public key. |
| `device public key invalid` | `DEVICE_AUTH_PUBLIC_KEY_INVALID` | `device-public-key`      | Не вдалося виконати формат/канонізацію public key.         |

Ціль міграції:

- Завжди очікуйте `connect.challenge`.
- Підписуйте навантаження v2, яке містить server nonce.
- Надсилайте той самий nonce у `connect.params.device.nonce`.
- Бажане навантаження підпису — `v3`, яке прив’язує `platform` і `deviceFamily`
  на додачу до полів device/client/role/scopes/token/nonce.
- Застарілі підписи `v2` залишаються прийнятними для сумісності, але прив’язка метаданих
  paired-device усе одно керує політикою команд під час повторного підключення.

## TLS + pinning

- Для WS-підключень підтримується TLS.
- Клієнти можуть за бажанням закріпити відбиток сертифіката gateway (див. config `gateway.tls`
  плюс `gateway.remote.tlsFingerprint` або CLI `--tls-fingerprint`).

## Область застосування

Цей протокол надає **повний API gateway** (status, channels, models, chat,
agent, sessions, nodes, approvals тощо). Точна поверхня визначається
схемами TypeBox у `src/gateway/protocol/schema.ts`.
