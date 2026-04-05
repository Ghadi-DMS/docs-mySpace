---
read_when:
    - Реалізація або оновлення WS-клієнтів gateway
    - Налагодження невідповідностей протоколу або збоїв підключення
    - Повторна генерація схеми/моделей протоколу
summary: 'Протокол Gateway WebSocket: рукостискання, фрейми, версіонування'
title: Протокол Gateway
x-i18n:
    generated_at: "2026-04-05T22:13:31Z"
    model: gpt-5.4
    provider: openai
    source_hash: c37f5b686562dda3ba3516ac6982ad87b2f01d8148233284e9917099c6e96d87
    source_path: gateway/protocol.md
    workflow: 15
---

# Протокол Gateway (WebSocket)

Протокол Gateway WS — це **єдина control plane + node transport** для
OpenClaw. Усі клієнти (CLI, веб-UI, застосунок macOS, iOS/Android nodes, headless
nodes) підключаються через WebSocket і оголошують свою **role** + **scope** під
час рукостискання.

## Транспорт

- WebSocket, текстові фрейми з JSON-повідомленнями.
- Першим фреймом **має** бути запит `connect`.

## Рукостискання (connect)

Gateway → Client (виклик до підключення):

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

Під час передавання в trusted bootstrap `hello-ok.auth` також може містити додаткові
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

Для вбудованого потоку bootstrap node/operator основний токен node
залишається `scopes: []`, а будь-який переданий токен operator залишається обмеженим
списком дозволів bootstrap operator (`operator.approvals`, `operator.read`,
`operator.talk.secrets`, `operator.write`). Перевірки scope для bootstrap
залишаються прив’язаними до префікса ролі: записи operator задовольняють лише запити operator, а ролям, що не є operator,
усе одно потрібні scopes під префіксом їхньої власної ролі.

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

- **Запит**: `{type:"req", id, method, params}`
- **Відповідь**: `{type:"res", id, ok, payload|error}`
- **Подія**: `{type:"event", event, payload, seq?, stateVersion?}`

Методи з побічними ефектами потребують **ключів ідемпотентності** (див. схему).

## Roles + scopes

### Roles

- `operator` = клієнт control plane (CLI/UI/автоматизація).
- `node` = хост можливостей (camera/screen/canvas/system.run).

### Scopes (operator)

Поширені scopes:

- `operator.read`
- `operator.write`
- `operator.admin`
- `operator.approvals`
- `operator.pairing`
- `operator.talk.secrets`

Для `talk.config` з `includeSecrets: true` потрібен `operator.talk.secrets`
(або `operator.admin`).

RPC-методи gateway, зареєстровані плагінами, можуть запитувати власний scope operator, але
зарезервовані префікси core admin (`config.*`, `exec.approvals.*`, `wizard.*`,
`update.*`) завжди зіставляються з `operator.admin`.

Scope методу — це лише перший рівень перевірки. Деякі slash-команди, до яких звертаються через
`chat.send`, додатково застосовують суворіші перевірки на рівні команд. Наприклад, постійні
записи `/config set` і `/config unset` потребують `operator.admin`.

`node.pair.approve` також має додаткову перевірку scope під час схвалення поверх
базового scope методу:

- запити без команд: `operator.pairing`
- запити з командами node, що не є exec: `operator.pairing` + `operator.write`
- запити, що включають `system.run`, `system.run.prepare` або `system.which`:
  `operator.pairing` + `operator.admin`

### Caps/commands/permissions (node)

Nodes оголошують заявлені можливості під час підключення:

- `caps`: високорівневі категорії можливостей.
- `commands`: allowlist команд для invoke.
- `permissions`: детальні перемикачі (наприклад, `screen.record`, `camera.capture`).

Gateway трактує це як **заявлені можливості** та застосовує allowlist на боці сервера.

## Presence

- `system-presence` повертає записи, згруповані за ідентичністю пристрою.
- Записи присутності містять `deviceId`, `roles` і `scopes`, щоб UI могли показувати один рядок на пристрій
  навіть коли він підключається і як **operator**, і як **node**.

## Поширені сімейства RPC-методів

Ця сторінка не є згенерованим повним дампом, але публічна WS-поверхня ширша
за наведені вище приклади рукостискання/автентифікації. Ось основні сімейства методів, які
Gateway надає сьогодні.

`hello-ok.features.methods` — це консервативний список виявлення, зібраний із
`src/gateway/server-methods-list.ts` плюс експортів методів завантажених плагінів/каналів.
Розглядайте його як механізм виявлення можливостей, а не як згенерований дамп кожного викликуваного helper,
реалізованого в `src/gateway/server-methods/*.ts`.

### Система та ідентичність

- `health` повертає кешований або щойно перевірений знімок стану gateway.
- `status` повертає зведення gateway у стилі `/status`; чутливі поля
  включаються лише для operator-клієнтів з admin scope.
- `gateway.identity.get` повертає ідентичність пристрою gateway, що використовується в relay та
  потоках pairing.
- `system-presence` повертає поточний знімок присутності для підключених
  пристроїв operator/node.
- `system-event` додає системну подію та може оновлювати/транслювати контекст
  присутності.
- `last-heartbeat` повертає останню збережену heartbeat-подію.
- `set-heartbeats` перемикає обробку heartbeat на gateway.

### Моделі та використання

- `models.list` повертає каталог моделей, дозволених у runtime.
- `usage.status` повертає зведення вікон використання постачальників/залишку квоти.
- `usage.cost` повертає агреговані зведення витрат за діапазон дат.
- `doctor.memory.status` повертає готовність vector-memory / embedding для
  активного робочого простору агента за замовчуванням.
- `sessions.usage` повертає зведення використання по сесіях.
- `sessions.usage.timeseries` повертає часовий ряд використання для однієї сесії.
- `sessions.usage.logs` повертає записи журналу використання для однієї сесії.

### Канали та helper-и входу

- `channels.status` повертає зведення стану вбудованих і bundled каналів/плагінів.
- `channels.logout` виконує вихід із конкретного каналу/акаунта, якщо канал
  підтримує вихід.
- `web.login.start` запускає потік входу за QR/вебом для поточного веб-провайдера каналу
  з підтримкою QR.
- `web.login.wait` очікує завершення цього потоку входу за QR/вебом і запускає
  канал у разі успіху.
- `push.test` надсилає тестовий APNs push до зареєстрованого iOS node.
- `voicewake.get` повертає збережені тригери wake-word.
- `voicewake.set` оновлює тригери wake-word і транслює зміну.

### Повідомлення та журнали

- `send` — це прямий RPC outbound-delivery для надсилання в канал/акаунт/гілку
  поза chat runner.
- `logs.tail` повертає tail налаштованого файлового журналу gateway з cursor/limit і
  обмеженнями max-byte.

### Talk і TTS

- `talk.config` повертає ефективний payload конфігурації Talk; `includeSecrets`
  потребує `operator.talk.secrets` (або `operator.admin`).
- `talk.mode` встановлює/транслює поточний стан режиму Talk для клієнтів WebChat/Control UI.
- `talk.speak` синтезує мовлення через активного постачальника мовлення Talk.
- `tts.status` повертає стан увімкнення TTS, активного постачальника, резервних постачальників
  і стан конфігурації постачальника.
- `tts.providers` повертає видимий інвентар постачальників TTS.
- `tts.enable` і `tts.disable` перемикають стан налаштувань TTS.
- `tts.setProvider` оновлює бажаного постачальника TTS.
- `tts.convert` виконує одноразове перетворення text-to-speech.

### Секрети, конфігурація, оновлення та wizard

- `secrets.reload` повторно розв’язує активні SecretRef і замінює runtime-стан секретів
  лише за повного успіху.
- `secrets.resolve` розв’язує призначення секретів для конкретного
  набору command/target.
- `config.get` повертає поточний знімок конфігурації та хеш.
- `config.set` записує валідований payload конфігурації.
- `config.patch` об’єднує часткове оновлення конфігурації.
- `config.apply` перевіряє та замінює повний payload конфігурації.
- `config.schema` повертає payload актуальної схеми конфігурації, який використовують Control UI і
  інструменти CLI: schema, `uiHints`, версію та метадані генерації, включно з
  метаданими схем plugin + channel, коли runtime може їх завантажити. Схема
  містить метадані полів `title` / `description`, отримані з тих самих підписів
  і довідкового тексту, що використовує UI, включно з вкладеними object, wildcard, array-item
  та гілками композиції `anyOf` / `oneOf` / `allOf`, коли є відповідна
  документація поля.
- `config.schema.lookup` повертає payload пошуку в межах шляху для одного шляху конфігурації:
  нормалізований шлях, поверхневий вузол схеми, відповідний hint + `hintPath` та
  зведення безпосередніх дочірніх елементів для деталізації в UI/CLI.
  - Вузли схеми lookup зберігають орієнтовану на користувача документацію та поширені поля валідації:
    `title`, `description`, `type`, `enum`, `const`, `format`, `pattern`,
    числові/рядкові/масивні/об’єктні межі та булеві прапорці, як-от
    `additionalProperties`, `deprecated`, `readOnly`, `writeOnly`.
  - Зведення дочірніх елементів містять `key`, нормалізований `path`, `type`, `required`,
    `hasChildren`, а також відповідні `hint` / `hintPath`.
- `update.run` запускає потік оновлення gateway і планує перезапуск лише тоді,
  коли саме оновлення було успішним.
- `wizard.start`, `wizard.next`, `wizard.status` і `wizard.cancel` надають
  onboarding wizard через WS RPC.

### Наявні основні сімейства

#### Helper-и агента та робочого простору

- `agents.list` повертає налаштовані записи агентів.
- `agents.create`, `agents.update` і `agents.delete` керують записами агентів і
  прив’язкою робочого простору.
- `agents.files.list`, `agents.files.get` і `agents.files.set` керують файлами
  робочого простору bootstrap, доступними для агента.
- `agent.identity.get` повертає ефективну ідентичність помічника для агента або
  сесії.
- `agent.wait` очікує завершення запуску та повертає термінальний знімок, коли
  він доступний.

#### Керування сесіями

- `sessions.list` повертає поточний індекс сесій.
- `sessions.subscribe` і `sessions.unsubscribe` перемикають підписки на події змін сесій
  для поточного WS-клієнта.
- `sessions.messages.subscribe` і `sessions.messages.unsubscribe` перемикають
  підписки на події транскрипту/повідомлень для однієї сесії.
- `sessions.preview` повертає обмежені попередні перегляди транскриптів для вказаних
  ключів сесій.
- `sessions.resolve` розв’язує або канонізує ціль сесії.
- `sessions.create` створює новий запис сесії.
- `sessions.send` надсилає повідомлення в наявну сесію.
- `sessions.steer` — це варіант переривання і перенаправлення для активної сесії.
- `sessions.abort` перериває активну роботу для сесії.
- `sessions.patch` оновлює метадані/перевизначення сесії.
- `sessions.reset`, `sessions.delete` і `sessions.compact` виконують обслуговування сесій.
- `sessions.get` повертає повний збережений рядок сесії.
- виконання chat, як і раніше, використовує `chat.history`, `chat.send`, `chat.abort` і
  `chat.inject`.
- `chat.history` нормалізується для відображення для UI-клієнтів: вбудовані теги директив прибираються з видимого тексту, plain-text XML-повідомлення викликів інструментів (включно з
  `<tool_call>...</tool_call>`, `<function_call>...</function_call>`,
  `<tool_calls>...</tool_calls>`, `<function_calls>...</function_calls>` та
  обрізаними блоками викликів інструментів) і витіклі ASCII/full-width control tokens моделі
  видаляються, чисті рядки помічника з silent-token, такі як точні `NO_REPLY` /
  `no_reply`, опускаються, а надто великі рядки можуть бути замінені placeholder-ами.

#### Pairing пристроїв і токени пристроїв

- `device.pair.list` повертає очікувальні та схвалені paired-пристрої.
- `device.pair.approve`, `device.pair.reject` і `device.pair.remove` керують
  записами pairing пристроїв.
- `device.token.rotate` обертає токен paired-пристрою в межах його схваленої ролі
  та дозволених scopes.
- `device.token.revoke` відкликає токен paired-пристрою.

#### Pairing node, invoke і відкладена робота

- `node.pair.request`, `node.pair.list`, `node.pair.approve`,
  `node.pair.reject` і `node.pair.verify` охоплюють pairing node і bootstrap
  verification.
- `node.list` і `node.describe` повертають стан відомих/підключених nodes.
- `node.rename` оновлює мітку paired node.
- `node.invoke` пересилає команду до підключеного node.
- `node.invoke.result` повертає результат для запиту invoke.
- `node.event` переносить події, що походять від node, назад у gateway.
- `node.canvas.capability.refresh` оновлює токени canvas-capability з обмеженою областю дії.
- `node.pending.pull` і `node.pending.ack` — це API черги для підключених nodes.
- `node.pending.enqueue` і `node.pending.drain` керують довговічною відкладеною роботою
  для offline/disconnected nodes.

#### Сімейства approvals

- `exec.approval.request` і `exec.approval.resolve` охоплюють одноразові
  запити approval для exec.
- `exec.approval.waitDecision` очікує на одне pending approval для exec і повертає
  остаточне рішення (або `null` у разі тайм-ауту).
- `exec.approvals.get` і `exec.approvals.set` керують знімками політики
  approvals для exec у gateway.
- `exec.approvals.node.get` і `exec.approvals.node.set` керують локальною політикою approvals для exec у node
  через команди relay node.
- `plugin.approval.request`, `plugin.approval.waitDecision` і
  `plugin.approval.resolve` охоплюють потоки approval, визначені плагінами.

#### Інші основні сімейства

- automation:
  - `wake` планує негайне або на наступний heartbeat введення тексту пробудження
  - `cron.list`, `cron.status`, `cron.add`, `cron.update`, `cron.remove`,
    `cron.run`, `cron.runs`
- skills/tools: `skills.*`, `tools.catalog`, `tools.effective`

### Поширені сімейства подій

- `chat`: оновлення UI chat, такі як `chat.inject`, та інші події chat лише для транскрипту.
- `session.message` і `session.tool`: оновлення транскрипту/потоку подій для
  підписаної сесії.
- `sessions.changed`: змінився індекс сесій або їхні метадані.
- `presence`: оновлення знімка системної присутності.
- `tick`: періодична подія keepalive / liveness.
- `health`: оновлення знімка стану gateway.
- `heartbeat`: оновлення потоку heartbeat-подій.
- `cron`: подія зміни запуску/завдання cron.
- `shutdown`: сповіщення про вимкнення gateway.
- `node.pair.requested` / `node.pair.resolved`: життєвий цикл pairing node.
- `node.invoke.request`: трансляція запиту invoke node.
- `device.pair.requested` / `device.pair.resolved`: життєвий цикл paired-пристрою.
- `voicewake.changed`: змінено конфігурацію тригерів wake-word.
- `exec.approval.requested` / `exec.approval.resolved`: життєвий цикл
  approval для exec.
- `plugin.approval.requested` / `plugin.approval.resolved`: життєвий цикл approval плагіна.

### Helper-методи node

- Nodes можуть викликати `skills.bins`, щоб отримати поточний список виконуваних файлів skill
  для перевірок auto-allow.

### Helper-методи operator

- Operators можуть викликати `tools.catalog` (`operator.read`), щоб отримати runtime-каталог інструментів для
  агента. Відповідь містить згруповані інструменти та метадані походження:
  - `source`: `core` або `plugin`
  - `pluginId`: власник plugin, коли `source="plugin"`
  - `optional`: чи є інструмент plugin необов’язковим
- Operators можуть викликати `tools.effective` (`operator.read`), щоб отримати runtime-ефективний
  інвентар інструментів для сесії.
  - `sessionKey` є обов’язковим.
  - Gateway виводить trusted runtime-контекст із сесії на боці сервера замість прийняття
    контексту auth або доставки, наданого викликачем.
  - Відповідь обмежена сесією і відображає те, що активна розмова може використовувати просто зараз,
    включно з core, plugin і channel tools.
- Operators можуть викликати `skills.status` (`operator.read`), щоб отримати видимий
  інвентар skills для агента.
  - `agentId` необов’язковий; не вказуйте його, щоб читати робочий простір агента за замовчуванням.
  - Відповідь містить eligibility, відсутні вимоги, перевірки конфігурації та
    санітизовані параметри встановлення без розкриття сирих значень секретів.
- Operators можуть викликати `skills.search` і `skills.detail` (`operator.read`) для
  метаданих виявлення ClawHub.
- Operators можуть викликати `skills.install` (`operator.admin`) у двох режимах:
  - Режим ClawHub: `{ source: "clawhub", slug, version?, force? }` встановлює
    папку skill до каталогу `skills/` робочого простору агента за замовчуванням.
  - Режим інсталятора gateway: `{ name, installId, dangerouslyForceUnsafeInstall?, timeoutMs? }`
    запускає оголошену дію `metadata.openclaw.install` на хості gateway.
- Operators можуть викликати `skills.update` (`operator.admin`) у двох режимах:
  - Режим ClawHub оновлює один відстежуваний slug або всі відстежувані встановлення ClawHub у
    робочому просторі агента за замовчуванням.
  - Режим Config патчить значення `skills.entries.<skillKey>`, такі як `enabled`,
    `apiKey` і `env`.

## Approvals для exec

- Коли запит exec потребує approval, gateway транслює `exec.approval.requested`.
- Клієнти operator завершують це викликом `exec.approval.resolve` (потребує scope `operator.approvals`).
- Для `host=node` `exec.approval.request` має містити `systemRunPlan` (канонічні `argv`/`cwd`/`rawCommand`/метадані сесії). Запити без `systemRunPlan` відхиляються.
- Після approval переслані виклики `node.invoke system.run` повторно використовують цей канонічний
  `systemRunPlan` як авторитетний контекст команди/cwd/сесії.
- Якщо викликач змінює `command`, `rawCommand`, `cwd`, `agentId` або
  `sessionKey` між prepare і фінальним затвердженим пересиланням `system.run`,
  gateway відхиляє запуск замість довіри до зміненого payload.

## Резервна доставка агента

- Запити `agent` можуть включати `deliver=true`, щоб запросити outbound delivery.
- `bestEffortDeliver=false` зберігає сувору поведінку: нерозв’язані або лише внутрішні цілі доставки повертають `INVALID_REQUEST`.
- `bestEffortDeliver=true` дозволяє резервний перехід до виконання лише в межах сесії, коли неможливо визначити зовнішній маршрут доставки (наприклад, internal/webchat sessions або неоднозначні багатоканальні конфігурації).

## Версіонування

- `PROTOCOL_VERSION` розміщено в `src/gateway/protocol/schema.ts`.
- Клієнти надсилають `minProtocol` + `maxProtocol`; сервер відхиляє невідповідності.
- Схеми + моделі генеруються з визначень TypeBox:
  - `pnpm protocol:gen`
  - `pnpm protocol:gen:swift`
  - `pnpm protocol:check`

## Автентифікація

- Автентифікація gateway за спільним секретом використовує `connect.params.auth.token` або
  `connect.params.auth.password` залежно від налаштованого режиму auth.
- Режими з передаванням ідентичності, як-от Tailscale Serve
  (`gateway.auth.allowTailscale: true`) або не-loopback
  `gateway.auth.mode: "trusted-proxy"`, задовольняють перевірку auth для connect через
  заголовки запиту замість `connect.params.auth.*`.
- Режим приватного ingress `gateway.auth.mode: "none"` повністю пропускає shared-secret connect auth; не відкривайте цей режим у публічному/ненадійному ingress.
- Після pairing Gateway видає **токен пристрою**, обмежений роллю + scopes
  підключення. Він повертається в `hello-ok.auth.deviceToken` і має
  зберігатися клієнтом для майбутніх підключень.
- Клієнти мають зберігати основний `hello-ok.auth.deviceToken` після будь-якого
  успішного connect.
- Повторне підключення з цим **збереженим** токеном пристрою також має повторно використовувати
  збережений схвалений набір scope для цього токена. Це зберігає вже наданий
  доступ на читання/перевірку/статус і не дає повторним підключенням непомітно
  звузитися до вужчого неявного admin-only scope.
- Звичайний пріоритет auth для connect: спочатку явний shared token/password, потім
  явний `deviceToken`, потім збережений токен для пристрою, потім bootstrap token.
- Додаткові записи `hello-ok.auth.deviceTokens` — це токени передавання bootstrap.
  Зберігайте їх лише тоді, коли connect використовував bootstrap auth на trusted transport,
  як-от `wss://` або loopback/local pairing.
- Якщо клієнт передає **явний** `deviceToken` або явні `scopes`, цей
  набір scopes, запитаний викликачем, залишається авторитетним; кешовані scopes повторно
  використовуються лише тоді, коли клієнт повторно використовує збережений токен для пристрою.
- Токени пристрою можна обертати/відкликати через `device.token.rotate` і
  `device.token.revoke` (потребує scope `operator.pairing`).
- Видача/ротація токенів залишається обмеженою схваленим набором ролей, записаним у
  записі pairing цього пристрою; ротація токена не може розширити пристрій до
  ролі, яку ніколи не було надано під час схвалення pairing.
- Для paired-device token sessions керування пристроями обмежується самим пристроєм, якщо
  викликач також не має `operator.admin`: викликачі без admin можуть видаляти/відкликати/обертати
  лише **власний** запис пристрою.
- `device.token.rotate` також перевіряє запитаний набір scope operator щодо
  поточних scopes сесії викликача. Викликачі без admin не можуть обернути токен у
  ширший набір scope operator, ніж той, який вони вже мають.
- Збої auth містять `error.details.code` плюс підказки для відновлення:
  - `error.details.canRetryWithDeviceToken` (boolean)
  - `error.details.recommendedNextStep` (`retry_with_device_token`, `update_auth_configuration`, `update_auth_credentials`, `wait_then_retry`, `review_auth_configuration`)
- Поведінка клієнта для `AUTH_TOKEN_MISMATCH`:
  - Trusted-клієнти можуть виконати одну обмежену повторну спробу з кешованим токеном для пристрою.
  - Якщо ця повторна спроба не вдається, клієнти мають припинити автоматичні цикли перепідключення та показати оператору вказівки щодо подальших дій.

## Ідентичність пристрою + pairing

- Nodes мають включати стабільну ідентичність пристрою (`device.id`), похідну від
  відбитка keypair.
- Gateways видають токени на кожен пристрій + роль.
- Для нових `device.id` потрібні схвалення pairing, якщо не ввімкнено локальне авто-схвалення.
- Авто-схвалення pairing зосереджене на прямих локальних loopback-підключеннях.
- OpenClaw також має вузький шлях self-connect для trusted shared-secret helper flows у локальному backend/container.
- Підключення з tailnet або LAN на тому самому хості все одно вважаються віддаленими для pairing і
  потребують схвалення.
- Усі WS-клієнти мають включати ідентичність `device` під час `connect` (operator + node).
  Control UI може пропускати її лише в таких режимах:
  - `gateway.controlUi.allowInsecureAuth=true` для сумісності з небезпечним HTTP лише на localhost.
  - успішна auth operator Control UI з `gateway.auth.mode: "trusted-proxy"`.
  - `gateway.controlUi.dangerouslyDisableDeviceAuth=true` (аварійний режим, критичне зниження безпеки).
- Усі підключення мають підписувати наданий сервером nonce `connect.challenge`.

### Діагностика міграції auth пристрою

Для застарілих клієнтів, які все ще використовують поведінку підпису до challenge, `connect` тепер повертає
коди деталей `DEVICE_AUTH_*` у `error.details.code` зі стабільним `error.details.reason`.

Поширені збої міграції:

| Повідомлення                | details.code                     | details.reason           | Значення                                           |
| --------------------------- | -------------------------------- | ------------------------ | -------------------------------------------------- |
| `device nonce required`     | `DEVICE_AUTH_NONCE_REQUIRED`     | `device-nonce-missing`   | Клієнт не вказав `device.nonce` (або надіслав порожнє значення). |
| `device nonce mismatch`     | `DEVICE_AUTH_NONCE_MISMATCH`     | `device-nonce-mismatch`  | Клієнт підписав застарілим/неправильним nonce.     |
| `device signature invalid`  | `DEVICE_AUTH_SIGNATURE_INVALID`  | `device-signature`       | Payload підпису не відповідає payload версії v2.   |
| `device signature expired`  | `DEVICE_AUTH_SIGNATURE_EXPIRED`  | `device-signature-stale` | Підписана часова позначка виходить за допустиме зміщення. |
| `device identity mismatch`  | `DEVICE_AUTH_DEVICE_ID_MISMATCH` | `device-id-mismatch`     | `device.id` не відповідає відбитку публічного ключа. |
| `device public key invalid` | `DEVICE_AUTH_PUBLIC_KEY_INVALID` | `device-public-key`      | Не вдалося обробити формат/канонізацію публічного ключа. |

Ціль міграції:

- Завжди очікуйте `connect.challenge`.
- Підписуйте payload v2, який містить nonce сервера.
- Надсилайте той самий nonce у `connect.params.device.nonce`.
- Бажаний payload підпису — `v3`, який прив’язує `platform` і `deviceFamily`
  додатково до полів device/client/role/scopes/token/nonce.
- Застарілі підписи `v2` залишаються прийнятними для сумісності, але прив’язка метаданих
  paired-пристрою все одно керує політикою команд під час повторного підключення.

## TLS + pinning

- Для WS-підключень підтримується TLS.
- Клієнти можуть за бажанням прив’язувати відбиток сертифіката gateway (див. конфігурацію `gateway.tls`
  плюс `gateway.remote.tlsFingerprint` або прапорець CLI `--tls-fingerprint`).

## Обсяг

Цей протокол надає **повний API gateway** (status, channels, models, chat,
agent, sessions, nodes, approvals тощо). Точна поверхня визначається
схемами TypeBox у `src/gateway/protocol/schema.ts`.
