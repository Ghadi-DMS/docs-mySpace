---
x-i18n:
    generated_at: "2026-04-08T02:02:07Z"
    model: gpt-5.4
    provider: openai
    source_hash: 0e156cc8e2fe946a0423862f937754a7caa1fe7e6863b50a80bff49a1c86e1e8
    source_path: refactor/qa.md
    workflow: 15
---

# Рефакторинг QA

Статус: базова міграція завершена.

## Мета

Перевести QA OpenClaw з моделі розділених визначень на єдине джерело істини:

- метадані сценаріїв
- промпти, що надсилаються моделі
- налаштування та очищення
- логіка harness
- перевірки та критерії успіху
- артефакти та підказки для звіту

Бажаний кінцевий стан — універсальний QA harness, який завантажує потужні файли визначень сценаріїв замість жорсткого кодування більшості поведінки в TypeScript.

## Поточний стан

Основне джерело істини тепер знаходиться в `qa/scenarios.md`.

Реалізовано:

- `qa/scenarios.md`
  - канонічний набір QA
  - ідентичність оператора
  - стартова місія
  - метадані сценаріїв
  - прив’язки обробників
- `extensions/qa-lab/src/scenario-catalog.ts`
  - парсер markdown-пакета + валідація zod
- `extensions/qa-lab/src/qa-agent-bootstrap.ts`
  - рендеринг плану з markdown-пакета
- `extensions/qa-lab/src/qa-agent-workspace.ts`
  - заповнює згенеровані файли сумісності плюс `QA_SCENARIOS.md`
- `extensions/qa-lab/src/suite.ts`
  - вибирає виконувані сценарії через визначені в markdown прив’язки обробників
- протокол шини QA + UI
  - універсальні вбудовані вкладення для рендерингу зображень/відео/аудіо/файлів

Поверхні, що залишаються розділеними:

- `extensions/qa-lab/src/suite.ts`
  - досі містить більшість виконуваної логіки спеціальних обробників
- `extensions/qa-lab/src/report.ts`
  - досі виводить структуру звіту з результатів часу виконання

Тож розділення джерела істини усунуто, але виконання все ще переважно спирається на обробники, а не є повністю декларативним.

## Як виглядає реальна поверхня сценаріїв

Перегляд поточного suite показує кілька різних класів сценаріїв.

### Проста взаємодія

- базовий сценарій каналу
- базовий сценарій DM
- подальша взаємодія в треді
- перемикання моделі
- проходження після схвалення
- реакція/редагування/видалення

### Мутація конфігурації та runtime

- вимкнення skill через patch конфігурації
- пробудження після перезапуску через застосування конфігурації
- перемикання можливостей після перезапуску конфігурації
- перевірка розходження інвентаря runtime

### Перевірки файлової системи та репозиторію

- звіт про виявлення source/docs
- збірка Lobster Invaders
- пошук артефакта згенерованого зображення

### Оркестрація пам’яті

- відновлення з пам’яті
- інструменти пам’яті в контексті каналу
- резервний сценарій при збої пам’яті
- ранжування пам’яті сесії
- ізоляція пам’яті треду
- sweep мрій пам’яті

### Інтеграція інструментів і плагінів

- виклик plugin-tools MCP
- видимість skill
- гаряче встановлення skill
- нативна генерація зображень
- повний цикл зображення
- розуміння зображення з вкладення

### Багатокрокові та багатосуб’єктні сценарії

- передача підагенту
- fanout synthesis підагента
- сценарії відновлення після перезапуску

Ці категорії важливі, оскільки вони визначають вимоги до DSL. Плоского списку промпт + очікуваний текст недостатньо.

## Напрямок

### Єдине джерело істини

Використовувати `qa/scenarios.md` як авторське джерело істини.

Пакет має залишатися:

- зрозумілим для читання під час рев’ю
- придатним для машинного парсингу
- достатньо насиченим, щоб керувати:
  - виконанням suite
  - bootstrap QA workspace
  - метаданими UI QA Lab
  - промптами для docs/discovery
  - генерацією звітів

### Бажаний формат авторингу

Використовувати markdown як формат верхнього рівня зі структурованим YAML усередині.

Рекомендована форма:

- YAML frontmatter
  - id
  - title
  - surface
  - tags
  - docs refs
  - code refs
  - model/provider overrides
  - prerequisites
- прозові секції
  - objective
  - notes
  - debugging hints
- fenced YAML-блоки
  - setup
  - steps
  - assertions
  - cleanup

Це дає:

- кращу читабельність PR, ніж гігантський JSON
- багатший контекст, ніж чистий YAML
- суворий парсинг і валідацію zod

Сирий JSON прийнятний лише як проміжна згенерована форма.

## Запропонована форма файлу сценарію

Приклад:

````md
---
id: image-generation-roundtrip
title: Image generation roundtrip
surface: image
tags: [media, image, roundtrip]
models:
  primary: openai/gpt-5.4
requires:
  tools: [image_generate]
  plugins: [openai, qa-channel]
docsRefs:
  - docs/help/testing.md
  - docs/concepts/model-providers.md
codeRefs:
  - extensions/qa-lab/src/suite.ts
  - src/gateway/chat-attachments.ts
---

# Objective

Verify generated media is reattached on the follow-up turn.

# Setup

```yaml scenario.setup
- action: config.patch
  patch:
    agents:
      defaults:
        imageGenerationModel:
          primary: openai/gpt-image-1
- action: session.create
  key: agent:qa:image-roundtrip
```

# Steps

```yaml scenario.steps
- action: agent.send
  session: agent:qa:image-roundtrip
  message: |
    Image generation check: generate a QA lighthouse image and summarize it in one short sentence.
- action: artifact.capture
  kind: generated-image
  promptSnippet: Image generation check
  saveAs: lighthouseImage
- action: agent.send
  session: agent:qa:image-roundtrip
  message: |
    Roundtrip image inspection check: describe the generated lighthouse attachment in one short sentence.
  attachments:
    - fromArtifact: lighthouseImage
```

# Expect

```yaml scenario.expect
- assert: outbound.textIncludes
  value: lighthouse
- assert: requestLog.matches
  where:
    promptIncludes: Roundtrip image inspection check
  imageInputCountGte: 1
- assert: artifact.exists
  ref: lighthouseImage
```
````

## Можливості runner, які має покривати DSL

На основі поточного suite універсальний runner потребує більшого, ніж просто виконання промптів.

### Дії середовища та налаштування

- `bus.reset`
- `gateway.waitHealthy`
- `channel.waitReady`
- `session.create`
- `thread.create`
- `workspace.writeSkill`

### Дії ходу агента

- `agent.send`
- `agent.wait`
- `bus.injectInbound`
- `bus.injectOutbound`

### Дії конфігурації та runtime

- `config.get`
- `config.patch`
- `config.apply`
- `gateway.restart`
- `tools.effective`
- `skills.status`

### Дії з файлами та артефактами

- `file.write`
- `file.read`
- `file.delete`
- `file.touchTime`
- `artifact.captureGeneratedImage`
- `artifact.capturePath`

### Дії з пам’яттю та cron

- `memory.indexForce`
- `memory.searchCli`
- `doctor.memory.status`
- `cron.list`
- `cron.run`
- `cron.waitCompletion`
- `sessionTranscript.write`

### Дії MCP

- `mcp.callTool`

### Перевірки

- `outbound.textIncludes`
- `outbound.inThread`
- `outbound.notInRoot`
- `tool.called`
- `tool.notPresent`
- `skill.visible`
- `skill.disabled`
- `file.contains`
- `memory.contains`
- `requestLog.matches`
- `sessionStore.matches`
- `cron.managedPresent`
- `artifact.exists`

## Змінні та посилання на артефакти

DSL має підтримувати збережені результати та подальші посилання на них.

Приклади з поточного suite:

- створити тред, а потім повторно використати `threadId`
- створити сесію, а потім повторно використати `sessionKey`
- згенерувати зображення, а потім прикріпити файл на наступному ході
- згенерувати рядок-маркер пробудження, а потім перевірити, що він з’являється пізніше

Необхідні можливості:

- `saveAs`
- `${vars.name}`
- `${artifacts.name}`
- типізовані посилання для шляхів, ключів сесій, id тредів, маркерів, результатів інструментів

Без підтримки змінних harness і надалі буде пропускати логіку сценаріїв назад у TypeScript.

## Що має залишитися як escape hatches

Повністю чистий декларативний runner нереалістичний на фазі 1.

Деякі сценарії за своєю природою потребують складної оркестрації:

- sweep мрій пам’яті
- пробудження після перезапуску через застосування конфігурації
- перемикання можливостей після перезапуску конфігурації
- визначення артефактів згенерованих зображень за timestamp/шляхом
- оцінка discovery-report

Поки що вони мають використовувати явні спеціальні обробники.

Рекомендоване правило:

- 85-90% декларативно
- явні кроки `customHandler` для складного залишку
- лише іменовані та задокументовані custom handlers
- жодного анонімного inline-коду у файлі сценарію

Це зберігає чистоту універсального рушія й водночас дає змогу рухатися вперед.

## Архітектурна зміна

### Поточний стан

Markdown сценаріїв уже є джерелом істини для:

- виконання suite
- bootstrap-файлів workspace
- каталогу сценаріїв UI QA Lab
- метаданих звітів
- промптів discovery

Згенерована сумісність:

- workspace із початковими даними все ще містить `QA_KICKOFF_TASK.md`
- workspace із початковими даними все ще містить `QA_SCENARIO_PLAN.md`
- workspace із початковими даними тепер також містить `QA_SCENARIOS.md`

## План рефакторингу

### Фаза 1: loader і schema

Завершено.

- додано `qa/scenarios.md`
- додано парсер для вмісту іменованого markdown YAML-пакета
- додано валідацію через zod
- споживачів переведено на розібраний пакет
- видалено `qa/seed-scenarios.json` і `qa/QA_KICKOFF_TASK.md` на рівні репозиторію

### Фаза 2: універсальний engine

- розділити `extensions/qa-lab/src/suite.ts` на:
  - loader
  - engine
  - registry дій
  - registry перевірок
  - custom handlers
- зберегти наявні допоміжні функції як операції engine

Результат:

- engine виконує прості декларативні сценарії

Почати зі сценаріїв, які переважно складаються з промпту + очікування + перевірки:

- подальша взаємодія в треді
- розуміння зображення з вкладення
- видимість і виклик skill
- базовий сценарій каналу

Результат:

- перші справжні сценарії, визначені в markdown, постачаються через універсальний engine

### Фаза 4: міграція сценаріїв середньої складності

- повний цикл генерації зображення
- інструменти пам’яті в контексті каналу
- ранжування пам’яті сесії
- передача підагенту
- fanout synthesis підагента

Результат:

- доведено роботу змінних, артефактів, перевірок інструментів і перевірок журналу запитів

### Фаза 5: залишити складні сценарії на custom handlers

- sweep мрій пам’яті
- пробудження після перезапуску через застосування конфігурації
- перемикання можливостей після перезапуску конфігурації
- розходження інвентаря runtime

Результат:

- той самий формат авторингу, але з явними блоками custom-step там, де це потрібно

### Фаза 6: видалити жорстко закодовану мапу сценаріїв

Щойно покриття пакета стане достатньо хорошим:

- видалити більшість специфічних для сценаріїв розгалужень TypeScript із `extensions/qa-lab/src/suite.ts`

## Підтримка fake Slack / rich media

Поточна шина QA орієнтована насамперед на текст.

Відповідні файли:

- `extensions/qa-channel/src/protocol.ts`
- `extensions/qa-lab/src/bus-state.ts`
- `extensions/qa-lab/src/bus-queries.ts`
- `extensions/qa-lab/src/bus-server.ts`
- `extensions/qa-lab/web/src/ui-render.ts`

Сьогодні шина QA підтримує:

- текст
- реакції
- треди

Вона ще не моделює вбудовані медіавкладення.

### Потрібний транспортний контракт

Додати універсальну модель вкладень шини QA:

```ts
type QaBusAttachment = {
  id: string;
  kind: "image" | "video" | "audio" | "file";
  mimeType: string;
  fileName?: string;
  inline?: boolean;
  url?: string;
  contentBase64?: string;
  width?: number;
  height?: number;
  durationMs?: number;
  altText?: string;
  transcript?: string;
};
```

Потім додати `attachments?: QaBusAttachment[]` до:

- `QaBusMessage`
- `QaBusInboundMessageInput`
- `QaBusOutboundMessageInput`

### Чому спочатку універсальна модель

Не створювати модель медіа лише для Slack.

Натомість:

- одна універсальна транспортна модель QA
- кілька рендерерів поверх неї
  - поточний чат QA Lab
  - майбутній fake Slack web
  - будь-які інші представлення fake transport

Це запобігає дублюванню логіки й дозволяє сценаріям із медіа залишатися незалежними від transport.

### Потрібна робота в UI

Оновити UI QA, щоб він рендерив:

- вбудований попередній перегляд зображення
- вбудований аудіоплеєр
- вбудований відеоплеєр
- chip вкладеного файла

Поточний UI вже може рендерити треди й реакції, тож рендеринг вкладень має нашаровуватися на ту саму модель картки повідомлення.

### Робота зі сценаріями, яку відкриває транспорт медіа

Щойно вкладення почнуть проходити через шину QA, можна буде додати багатші сценарії fake-chat:

- відповідь із вбудованим зображенням у fake Slack
- розуміння аудіовкладення
- розуміння відеовкладення
- змішаний порядок вкладень
- відповідь у треді зі збереженням медіа

## Рекомендація

Наступний етап реалізації має бути таким:

1. додати markdown loader сценаріїв + zod schema
2. згенерувати поточний каталог із markdown
3. спочатку мігрувати кілька простих сценаріїв
4. додати універсальну підтримку вкладень у шині QA
5. рендерити вбудоване зображення в UI QA
6. потім розширити на аудіо та відео

Це найкоротший шлях, який доводить обидві цілі:

- універсальний QA, визначений у markdown
- багатші поверхні fake messaging

## Відкриті питання

- чи мають файли сценаріїв дозволяти вбудовані markdown-шаблони промптів з інтерполяцією змінних
- чи мають setup/cleanup бути іменованими секціями або просто впорядкованими списками дій
- чи мають посилання на артефакти бути строго типізованими в schema або рядковими
- чи мають custom handlers жити в одному registry або в registry для кожної surface
- чи має згенерований JSON-файл сумісності залишатися закоміченим під час міграції
