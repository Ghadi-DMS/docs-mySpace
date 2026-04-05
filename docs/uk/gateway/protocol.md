---
read_when:
    - Реалізація або оновлення WS-клієнтів gateway
    - Налагодження невідповідностей протоколу або збоїв підключення
    - Повторна генерація схеми/моделей протоколу
summary: 'Протокол Gateway WebSocket: рукостискання, фрейми, версіонування'
title: Протокол Gateway
x-i18n:
    generated_at: "2026-04-05T21:59:53Z"
    model: gpt-5.4
    provider: openai
    source_hash: dcf6b4a88bb213e0704a767bf466329c8b33656919e933668f5f6ef06864ea55
    source_path: gateway/protocol.md
    workflow: 15
---

# Протокол Gateway (WebSocket)

Протокол Gateway WS — це **єдина контрольна площина + транспорт вузлів** для
OpenClaw. Усі клієнти (CLI, web UI, macOS app, iOS/Android вузли, headless
вузли) підключаються через WebSocket і оголошують свою **роль** + **область**
під час рукостискання.

## Транспорт

- WebSocket, текстові фрейми з JSON-навантаженнями.
- Перший фрейм **має** бути запитом `connect`.

## Рукостискання (connect)

Gateway → Client (виклик перед підключенням):

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

Коли видається токен пристрою, `hello-ok` також включає:

```json
{
  "auth": {
    "deviceToken": "…",
    "role": "operator",
    "scopes": ["operator.read", "operator.write"]
  }
}
```

Під час довіреної передачі bootstrap `hello-ok.auth` також може включати
додаткові записи ролей з обмеженнями в `deviceTokens`:

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

Для вбудованого bootstrap-потоку node/operator первинний токен вузла
залишається `scopes: []`, а будь-який переданий токен оператора залишається
обмеженим allowlist bootstrap-оператора (`operator.approvals`, `operator.read`,
`operator.talk.secrets`, `operator.write`). Перевірки bootstrap-областей
залишаються прив'язаними до префікса ролі: записи operator задовольняють лише
запити operator, а ролям, що не є operator, усе одно потрібні області під їхнім
власним префіксом ролі.

### Приклад вузла

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

## Фреймування

- **Запит**: `{type:"req", id, method, params}`
- **Відповідь**: `{type:"res", id, ok, payload|error}`
- **Подія**: `{type:"event", event, payload, seq?, stateVersion?}`

Методи з побічними ефектами потребують **ключів ідемпотентності** (див. схему).

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

Для `talk.config` з `includeSecrets: true` потрібен `operator.talk.secrets`
(або `operator.admin`).

Зареєстровані плагінами gateway RPC-методи можуть вимагати власну область
operator, але зарезервовані префікси core admin (`config.*`,
`exec.approvals.*`, `wizard.*`, `update.*`) завжди зводяться до
`operator.admin`.

Область методу — це лише перший бар'єр. Деякі slash-команди, що досягаються
через `chat.send`, додатково застосовують суворіші перевірки на рівні команд.
Наприклад, постійні записи `/config set` і `/config unset` потребують
`operator.admin`.

`node.pair.approve` також має додаткову перевірку області під час схвалення
поверх базової області методу:

- запити без команд: `operator.pairing`
- запити з командами вузла, що не є exec: `operator.pairing` + `operator.write`
- запити, що включають `system.run`, `system.run.prepare` або `system.which`:
  `operator.pairing` + `operator.admin`

### Caps/commands/permissions (node)

Вузли оголошують твердження про можливості під час підключення:

- `caps`: високорівневі категорії можливостей.
- `commands`: allowlist команд для invoke.
- `permissions`: деталізовані перемикачі (наприклад, `screen.record`, `camera.capture`).

Gateway розглядає їх як **твердження** і застосовує серверні allowlist.

## Присутність

- `system-presence` повертає записи, згруповані за ідентичністю пристрою.
- Записи присутності включають `deviceId`, `roles` і `scopes`, щоб UI могли показувати один рядок на пристрій
  навіть коли він підключається і як **operator**, і як **node**.

## Поширені сімейства RPC-методів

Ця сторінка не є згенерованим повним вивантаженням, але публічна WS-поверхня
ширша, ніж наведені вище приклади рукостискання/автентифікації. Це основні
сімейства методів, які Gateway надає сьогодні.

`hello-ok.features.methods` — це консервативний список виявлення, побудований з
`src/gateway/server-methods-list.ts` плюс експортів методів завантажених
плагінів/каналів. Сприймайте його як механізм виявлення можливостей, а не як
згенерований вивантаження кожного викликаного хелпера, реалізованого в
`src/gateway/server-methods/*.ts`.

### Система та ідентичність

- `health` повертає кешований або щойно перевірений знімок стану gateway.
- `status` повертає підсумок gateway у стилі `/status`; чутливі поля
  включаються лише для operator-клієнтів з admin-областю.
- `gateway.identity.get` повертає ідентичність пристрою gateway, що
  використовується в relay і pairing-потоках.
- `system-presence` повертає поточний знімок присутності для підключених
  пристроїв operator/node.
- `system-event` додає системну подію і може оновлювати/транслювати контекст
  присутності.
- `last-heartbeat` повертає останню збережену подію heartbeat.
- `set-heartbeats` вмикає або вимикає обробку heartbeat у gateway.

### Моделі та використання

- `models.list` повертає каталог моделей, дозволених у runtime.
- `usage.status` повертає зведення вікон використання провайдерів/залишкових квот.
- `usage.cost` повертає агреговані зведення витрат за діапазон дат.
- `doctor.memory.status` повертає стан готовності vector-memory / embedding для
  активного робочого простору агента за замовчуванням.
- `sessions.usage` повертає зведення використання для кожної сесії.
- `sessions.usage.timeseries` повертає часовий ряд використання для однієї сесії.
- `sessions.usage.logs` повертає записи журналу використання для однієї сесії.

### Канали та хелпери входу

- `channels.status` повертає зведення статусу вбудованих і bundled каналів/плагінів.
- `channels.logout` виходить із конкретного каналу/акаунта, якщо канал
  підтримує вихід.
- `web.login.start` запускає потік QR/web входу для поточного web-провайдера
  каналу, що підтримує QR.
- `web.login.wait` очікує завершення цього потоку QR/web входу і запускає
  канал у разі успіху.
- `push.test` надсилає тестовий APNs push до зареєстрованого iOS-вузла.
- `voicewake.get` повертає збережені тригери wake-word.
- `voicewake.set` оновлює тригери wake-word і транслює зміни.

### Повідомлення та журнали

- `send` — це прямий RPC для вихідної доставки для
  каналу/акаунта/потоку поза chat runner.
- `logs.tail` повертає tail налаштованого файлового журналу gateway з
  керуванням cursor/limit і max-byte.

### Talk і TTS

- `talk.config` повертає ефективне навантаження конфігурації Talk; `includeSecrets`
  потребує `operator.talk.secrets` (або `operator.admin`).
- `talk.mode` встановлює/транслює поточний стан режиму Talk для клієнтів
  WebChat/Control UI.
- `talk.speak` синтезує мовлення через активний speech-провайдер Talk.
- `tts.status` повертає стан увімкнення TTS, активного провайдера, резервних провайдерів
  і стан конфігурації провайдера.
- `tts.providers` повертає видимий інвентар TTS-провайдерів.
- `tts.enable` і `tts.disable` перемикають стан налаштувань TTS.
- `tts.setProvider` оновлює бажаного TTS-провайдера.
- `tts.convert` запускає одноразове перетворення text-to-speech.

### Secrets, config, update і wizard

- `secrets.reload` повторно розв'язує активні SecretRef і замінює runtime-стан
  секретів лише за повного успіху.
- `secrets.resolve` розв'язує прив'язки секретів до команд-цілей для конкретного
  набору command/target.
- `config.get` повертає поточний знімок config і hash.
- `config.set` записує перевірене навантаження config.
- `config.patch` об'єднує часткове оновлення config.
- `config.apply` перевіряє і замінює повне навантаження config.
- `config.schema` повертає live-навантаження схеми config, яке використовують Control UI і
  CLI tooling: schema, `uiHints`, version і метадані генерації, включно з
  метаданими схем плагінів + каналів, коли runtime може їх завантажити. Схема
  включає метадані полів `title` / `description`, отримані з тих самих labels
  і тексту довідки, які використовує UI, включно з вкладеними object, wildcard,
  array-item і гілками композиції `anyOf` / `oneOf` / `allOf`, коли існує
  відповідна документація поля.
- `config.schema.lookup` повертає навантаження lookup для одного шляху config:
  нормалізований шлях, вузол поверхневої схеми, знайдений hint + `hintPath` та
  підсумки безпосередніх дочірніх елементів для деталізації в UI/CLI.
  - Вузли схеми lookup зберігають документацію для користувача та поширені поля валідації:
    `title`, `description`, `type`, `enum`, `const`, `format`, `pattern`,
    межі numeric/string/array/object і булеві прапорці на кшталт
    `additionalProperties`, `deprecated`, `readOnly`, `writeOnly`.
  - Підсумки дочірніх елементів показують `key`, нормалізований `path`, `type`, `required`,
    `hasChildren`, а також знайдені `hint` / `hintPath`.
- `update.run` запускає потік оновлення gateway і планує перезапуск лише тоді,
  коли саме оновлення було успішним.
- `wizard.start`, `wizard.next`, `wizard.status` і `wizard.cancel` надають
  wizard онбордингу через WS RPC.

### Наявні основні сімейства

#### Хелпери агента та робочого простору

- `agents.list` повертає налаштовані записи агентів.
- `agents.create`, `agents.update` і `agents.delete` керують записами агентів і
  прив'язкою робочого простору.
- `agents.files.list`, `agents.files.get` і `agents.files.set` керують файлами
  bootstrap-робочого простору, доступними для агента.
- `agent.identity.get` повертає ефективну ідентичність помічника для агента або
  сесії.
- `agent.wait` очікує завершення запуску і повертає термінальний знімок, коли
  він доступний.

#### Керування сесіями

- `sessions.list` повертає поточний індекс сесій.
- `sessions.subscribe` і `sessions.unsubscribe` вмикають або вимикають
  підписки на події змін сесій для поточного WS-клієнта.
- `sessions.messages.subscribe` і `sessions.messages.unsubscribe` вмикають або вимикають
  підписки на події transcript/message для однієї сесії.
- `sessions.preview` повертає обмежені попередні перегляди transcript для вказаних
  ключів сесій.
- `sessions.resolve` розв'язує або канонізує ціль сесії.
- `sessions.create` створює новий запис сесії.
- `sessions.send` надсилає повідомлення в наявну сесію.
- `sessions.steer` — це варіант interrupt-and-steer для активної сесії.
- `sessions.abort` перериває активну роботу для сесії.
- `sessions.patch` оновлює метадані/override сесії.
- `sessions.reset`, `sessions.delete` і `sessions.compact` виконують
  обслуговування сесій.
- `sessions.get` повертає повний збережений рядок сесії.
- виконання chat, як і раніше, використовує `chat.history`, `chat.send`, `chat.abort` і
  `chat.inject`.
- `chat.history` нормалізований для відображення в UI-клієнтах: вбудовані теги директив
  видаляються з видимого тексту, XML-навантаження plain-text викликів інструментів (включно з
  `<tool_call>...</tool_call>`, `<function_call>...</function_call>`,
  `<tool_calls>...</tool_calls>`, `<function_calls>...</function_calls>` і
  усіченими блоками викликів інструментів) та витоки ASCII/full-width токенів керування моделі
  видаляються, чисті рядки помічника з silent-token, як-от точні `NO_REPLY` /
  `no_reply`, пропускаються, а надто великі рядки можуть замінюватися заповнювачами.

#### Pairing пристроїв і токени пристроїв

- `device.pair.list` повертає очікувані та схвалені пов'язані пристрої.
- `device.pair.approve`, `device.pair.reject` і `device.pair.remove` керують
  записами pairing пристроїв.
- `device.token.rotate` ротує токен пов'язаного пристрою в межах схвалених для нього ролей
  і областей.
- `device.token.revoke` відкликає токен пов'язаного пристрою.

#### Pairing вузлів, invoke і відкладена робота

- `node.pair.request`, `node.pair.list`, `node.pair.approve`,
  `node.pair.reject` і `node.pair.verify` охоплюють pairing вузлів і bootstrap-
  перевірку.
- `node.list` і `node.describe` повертають стан відомих/підключених вузлів.
- `node.rename` оновлює мітку пов'язаного вузла.
- `node.invoke` пересилає команду до підключеного вузла.
- `node.invoke.result` повертає результат запиту invoke.
- `node.event` переносить події, що походять від вузла, назад у gateway.
- `node.canvas.capability.refresh` оновлює токени можливостей canvas з обмеженою областю.
- `node.pending.pull` і `node.pending.ack` — це API черги підключених вузлів.
- `node.pending.enqueue` і `node.pending.drain` керують надійною відкладеною роботою
  для офлайн/відключених вузлів.

#### Сімейства схвалень

- `exec.approval.request` і `exec.approval.resolve` охоплюють одноразові
  запити на схвалення exec.
- `exec.approval.waitDecision` очікує на одне відкладене схвалення exec і повертає
  фінальне рішення (або `null` за timeout).
- `exec.approvals.get` і `exec.approvals.set` керують знімками політик
  схвалення exec у gateway.
- `exec.approvals.node.get` і `exec.approvals.node.set` керують локальною для вузла
  політикою схвалення exec через relay-команди вузла.
- `plugin.approval.request`, `plugin.approval.waitDecision` і
  `plugin.approval.resolve` охоплюють потоки схвалення, визначені плагінами.

#### Інші основні сімейства

- automation:
  - `wake` планує негайне або наступне через heartbeat вставлення тексту wake
  - `cron.list`, `cron.status`, `cron.add`, `cron.update`, `cron.remove`,
    `cron.run`, `cron.runs`
- skills/tools: `skills.*`, `tools.catalog`, `tools.effective`

### Поширені сімейства подій

- `chat`: оновлення UI chat, такі як `chat.inject`, та інші події chat лише для transcript.
- `session.message` і `session.tool`: оновлення transcript/event-stream для
  сесії, на яку оформлено підписку.
- `sessions.changed`: індекс сесій або метадані змінено.
- `presence`: оновлення знімка системної присутності.
- `tick`: періодична подія keepalive / liveness.
- `health`: оновлення знімка стану gateway.
- `heartbeat`: оновлення потоку подій heartbeat.
- `cron`: подія зміни запуску/завдання cron.
- `shutdown`: сповіщення про вимкнення gateway.
- `node.pair.requested` / `node.pair.resolved`: життєвий цикл pairing вузлів.
- `node.invoke.request`: трансляція запиту invoke вузла.
- `device.pair.requested` / `device.pair.resolved`: життєвий цикл пов'язаних пристроїв.
- `voicewake.changed`: змінено конфігурацію тригерів wake-word.
- `exec.approval.requested` / `exec.approval.resolved`: життєвий цикл
  схвалення exec.
- `plugin.approval.requested` / `plugin.approval.resolved`: життєвий цикл схвалення плагіна.

### Хелпер-методи вузлів

- Вузли можуть викликати `skills.bins`, щоб отримати поточний список виконуваних файлів skill
  для перевірок auto-allow.

### Хелпер-методи operator

- Operator можуть викликати `tools.catalog` (`operator.read`), щоб отримати runtime-каталог інструментів для
  агента. Відповідь включає згруповані інструменти та метадані походження:
  - `source`: `core` або `plugin`
  - `pluginId`: власник плагіна, коли `source="plugin"`
  - `optional`: чи є інструмент плагіна optional
- Operator можуть викликати `tools.effective` (`operator.read`), щоб отримати runtime-ефективний
  інвентар інструментів для сесії.
  - `sessionKey` є обов'язковим.
  - Gateway виводить довірений runtime-контекст із сесії на боці сервера замість приймати
    auth або контекст доставки, наданий викликачем.
  - Відповідь прив'язана до сесії й відображає те, що активна розмова може використовувати просто зараз,
    включно з core, plugin і channel tools.
- Operator можуть викликати `skills.status` (`operator.read`), щоб отримати видимий
  інвентар skills для агента.
  - `agentId` необов'язковий; пропустіть його, щоб читати робочий простір агента за замовчуванням.
  - Відповідь включає eligibility, відсутні вимоги, перевірки config і
    санітизовані параметри встановлення без розкриття сирих секретних значень.
- Operator можуть викликати `skills.search` і `skills.detail` (`operator.read`) для
  метаданих виявлення ClawHub.
- Operator можуть викликати `skills.install` (`operator.admin`) у двох режимах:
  - Режим ClawHub: `{ source: "clawhub", slug, version?, force? }` встановлює
    папку skill до каталогу `skills/` робочого простору агента за замовчуванням.
  - Режим установника gateway: `{ name, installId, dangerouslyForceUnsafeInstall?, timeoutMs? }`
    запускає оголошену дію `metadata.openclaw.install` на хості gateway.
- Operator можуть викликати `skills.update` (`operator.admin`) у двох режимах:
  - Режим ClawHub оновлює один відстежуваний slug або всі відстежувані встановлення ClawHub у
    робочому просторі агента за замовчуванням.
  - Режим Config патчить значення `skills.entries.<skillKey>`, такі як `enabled`,
    `apiKey` і `env`.

## Схвалення exec

- Коли запит exec потребує схвалення, gateway транслює `exec.approval.requested`.
- Клієнти operator розв'язують це, викликаючи `exec.approval.resolve` (потребує області `operator.approvals`).
- Для `host=node` `exec.approval.request` має включати `systemRunPlan` (канонічні `argv`/`cwd`/`rawCommand`/метадані сесії). Запити без `systemRunPlan` відхиляються.
- Після схвалення переслані виклики `node.invoke system.run` повторно використовують цей канонічний
  `systemRunPlan` як авторитетний контекст command/cwd/session.
- Якщо викликач змінює `command`, `rawCommand`, `cwd`, `agentId` або
  `sessionKey` між prepare і фінальним пересиланням схваленого `system.run`,
  gateway відхиляє запуск замість довіряти зміненому навантаженню.

## Резервна доставка агента

- Запити `agent` можуть включати `deliver=true`, щоб запитати вихідну доставку.
- `bestEffortDeliver=false` зберігає сувору поведінку: нерозв'язані або лише внутрішні цілі доставки повертають `INVALID_REQUEST`.
- `bestEffortDeliver=true` дозволяє резервний перехід до виконання лише в межах сесії, коли жоден зовнішній маршрут доставки не може бути розв'язаний (наприклад, для внутрішніх/webchat-сесій або неоднозначних багатоканальних конфігурацій).

## Версіонування

- `PROTOCOL_VERSION` розміщений у `src/gateway/protocol/schema.ts`.
- Клієнти надсилають `minProtocol` + `maxProtocol`; сервер відхиляє невідповідності.
- Схеми + моделі генеруються з визначень TypeBox:
  - `pnpm protocol:gen`
  - `pnpm protocol:gen:swift`
  - `pnpm protocol:check`

## Автентифікація

- Автентифікація gateway зі спільним секретом використовує `connect.params.auth.token` або
  `connect.params.auth.password` залежно від налаштованого режиму автентифікації.
- Режими, що несуть ідентичність, як-от Tailscale Serve
  (`gateway.auth.allowTailscale: true`) або режим не-loopback
  `gateway.auth.mode: "trusted-proxy"`, задовольняють перевірку автентифікації connect через
  заголовки запиту замість `connect.params.auth.*`.
- Приватний вхідний трафік `gateway.auth.mode: "none"` повністю пропускає автентифікацію connect зі спільним секретом; не виставляйте цей режим у публічний/недовірений ingress.
- Після pairing Gateway видає **токен пристрою**, обмежений роллю + областями
  підключення. Він повертається в `hello-ok.auth.deviceToken` і має
  бути збережений клієнтом для майбутніх підключень.
- Клієнти мають зберігати основний `hello-ok.auth.deviceToken` після будь-якого
  успішного підключення.
- Повторне підключення з цим **збереженим** токеном пристрою також має повторно використовувати збережений
  набір схвалених областей для цього токена. Це зберігає вже наданий доступ
  до читання/перевірки/status і запобігає тихому звуженню повторних підключень до
  вужчої неявної admin-only області.
- Звичайний пріоритет автентифікації підключення: спочатку явний спільний token/password, потім
  явний `deviceToken`, потім збережений токен для пристрою, потім bootstrap-токен.
- Додаткові записи `hello-ok.auth.deviceTokens` — це токени передачі bootstrap.
  Зберігайте їх лише тоді, коли підключення використовувало bootstrap-автентифікацію на довіреному транспорті,
  такому як `wss://` або loopback/local pairing.
- Якщо клієнт надає **явний** `deviceToken` або явні `scopes`, цей
  запитаний викликачем набір областей залишається авторитетним; кешовані області
  повторно використовуються лише тоді, коли клієнт повторно використовує збережений токен для пристрою.
- Токени пристроїв можна ротувати/відкликати через `device.token.rotate` і
  `device.token.revoke` (потребує області `operator.pairing`).
- Видача/ротація токенів залишається обмеженою схваленим набором ролей, записаним у
  записі pairing цього пристрою; ротація токена не може розширити пристрій до
  ролі, якої схвалення pairing ніколи не надавало.
- Для сесій з токеном пов'язаного пристрою керування пристроєм обмежене власним контекстом, якщо тільки
  викликач також не має `operator.admin`: викликачі без admin можуть видаляти/відкликати/ротувати
  лише **власний** запис пристрою.
- `device.token.rotate` також перевіряє запитаний набір operator-областей щодо
  поточних областей сесії викликача. Викликачі без admin не можуть ротувати токен у
  ширший набір operator-областей, ніж уже мають.
- Збої автентифікації включають `error.details.code` плюс підказки для відновлення:
  - `error.details.canRetryWithDeviceToken` (boolean)
  - `error.details.recommendedNextStep` (`retry_with_device_token`, `update_auth_configuration`, `update_auth_credentials`, `wait_then_retry`, `review_auth_configuration`)
- Поведінка клієнта для `AUTH_TOKEN_MISMATCH`:
  - Довірені клієнти можуть виконати одну обмежену повторну спробу з кешованим токеном для пристрою.
  - Якщо ця повторна спроба не вдається, клієнти мають зупинити автоматичні цикли повторного підключення й показати вказівки для дій оператора.

## Ідентичність пристрою + pairing

- Вузли мають включати стабільну ідентичність пристрою (`device.id`), похідну від
  відбитка keypair.
- Gateway видає токени для кожного пристрою + ролі.
- Для нових `device.id` потрібні схвалення pairing, якщо не ввімкнено локальне auto-approval.
- Auto-approval pairing зосереджене на прямих локальних loopback-підключеннях.
- OpenClaw також має вузький шлях self-connect для backend/container-local
  для довірених helper-потоків зі спільним секретом.
- Локальне auto-approval для operator-пристроїв ініціалізує обмежений базовий токен для пристрою замість збереження довільно запитаних operator-областей. Сесія зі shared-
  secret усе одно може бути ширшою, ніж тихо виданий токен пристрою.
- Підключення через tailnet або LAN з того самого хоста все одно вважаються віддаленими для pairing і
  потребують схвалення.
- Усі WS-клієнти мають включати ідентичність `device` під час `connect` (operator + node).
  Control UI може пропускати її лише в таких режимах:
  - `gateway.controlUi.allowInsecureAuth=true` для сумісності з localhost-only insecure HTTP.
  - успішна operator-автентифікація Control UI через `gateway.auth.mode: "trusted-proxy"`.
  - `gateway.controlUi.dangerouslyDisableDeviceAuth=true` (аварійний варіант, серйозне зниження безпеки).
- Усі підключення мають підписувати наданий сервером nonce `connect.challenge`.

### Діагностика міграції автентифікації пристрою

Для застарілих клієнтів, які все ще використовують поведінку підписування до challenge, `connect` тепер повертає
коди деталей `DEVICE_AUTH_*` у `error.details.code` зі стабільним `error.details.reason`.

Поширені збої міграції:

| Повідомлення                 | details.code                     | details.reason           | Значення                                           |
| ---------------------------- | -------------------------------- | ------------------------ | -------------------------------------------------- |
| `device nonce required`      | `DEVICE_AUTH_NONCE_REQUIRED`     | `device-nonce-missing`   | Клієнт не вказав `device.nonce` (або надіслав порожнє значення). |
| `device nonce mismatch`      | `DEVICE_AUTH_NONCE_MISMATCH`     | `device-nonce-mismatch`  | Клієнт підписав зі застарілим/неправильним nonce.  |
| `device signature invalid`   | `DEVICE_AUTH_SIGNATURE_INVALID`  | `device-signature`       | Навантаження підпису не відповідає навантаженню v2. |
| `device signature expired`   | `DEVICE_AUTH_SIGNATURE_EXPIRED`  | `device-signature-stale` | Підписана позначка часу поза межами дозволеного skew. |
| `device identity mismatch`   | `DEVICE_AUTH_DEVICE_ID_MISMATCH` | `device-id-mismatch`     | `device.id` не відповідає відбитку public key. |
| `device public key invalid`  | `DEVICE_AUTH_PUBLIC_KEY_INVALID` | `device-public-key`      | Формат public key/канонізація не пройшли.         |

Ціль міграції:

- Завжди дочікуйтеся `connect.challenge`.
- Підписуйте навантаження v2, яке включає серверний nonce.
- Надсилайте той самий nonce у `connect.params.device.nonce`.
- Бажане навантаження підпису — `v3`, яке прив'язує `platform` і `deviceFamily`
  на додачу до полів device/client/role/scopes/token/nonce.
- Застарілі підписи `v2` усе ще приймаються для сумісності, але прив'язка метаданих
  пов'язаного пристрою все одно контролює політику команд під час повторного підключення.

## TLS + pinning

- TLS підтримується для WS-підключень.
- Клієнти можуть за бажанням pin-ити відбиток сертифіката gateway (див. config `gateway.tls`
  плюс `gateway.remote.tlsFingerprint` або CLI `--tls-fingerprint`).

## Область дії

Цей протокол надає **повний API gateway** (status, channels, models, chat,
agent, sessions, nodes, approvals тощо). Точна поверхня визначається
схемами TypeBox у `src/gateway/protocol/schema.ts`.
