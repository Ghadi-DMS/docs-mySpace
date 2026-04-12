---
read_when:
    - Ви хочете налаштувати постачальників пошуку в пам’яті або моделі ембедингів
    - Ви хочете налаштувати бекенд QMD
    - Ви хочете налаштувати гібридний пошук, MMR або часове згасання
    - Ви хочете увімкнути мультимодальну індексацію пам’яті
summary: Усі параметри конфігурації для пошуку в пам’яті, постачальників ембедингів, QMD, гібридного пошуку та мультимодальної індексації
title: Довідник із конфігурації пам’яті
x-i18n:
    generated_at: "2026-04-12T17:53:49Z"
    model: gpt-5.4
    provider: openai
    source_hash: 299ca9b69eea292ea557a2841232c637f5c1daf2bc0f73c0a42f7c0d8d566ce2
    source_path: reference/memory-config.md
    workflow: 15
---

# Довідник із конфігурації пам’яті

На цій сторінці наведено всі параметри конфігурації пошуку в пам’яті OpenClaw. Для
концептуальних оглядів дивіться:

- [Огляд пам’яті](/uk/concepts/memory) -- як працює пам’ять
- [Вбудований рушій](/uk/concepts/memory-builtin) -- стандартний бекенд SQLite
- [Рушій QMD](/uk/concepts/memory-qmd) -- local-first сайдкар
- [Пошук у пам’яті](/uk/concepts/memory-search) -- конвеєр пошуку та налаштування
- [Active Memory](/uk/concepts/active-memory) -- увімкнення субагента пам’яті для інтерактивних сесій

Усі налаштування пошуку в пам’яті знаходяться в `agents.defaults.memorySearch` у
`openclaw.json`, якщо не зазначено інше.

Якщо ви шукаєте перемикач функції **active memory** і конфігурацію субагента,
вони знаходяться в `plugins.entries.active-memory`, а не в `memorySearch`.

Active memory використовує двоетапну модель умов:

1. plugin має бути увімкнений і націлений на поточний ідентифікатор агента
2. запит має бути відповідною інтерактивною постійною сесією чату

Дивіться [Active Memory](/uk/concepts/active-memory), щоб дізнатися про модель активації,
конфігурацію, що належить plugin, збереження транскриптів і безпечний шаблон розгортання.

---

## Вибір постачальника

| Key        | Type      | Default          | Description                                                                                 |
| ---------- | --------- | ---------------- | ------------------------------------------------------------------------------------------- |
| `provider` | `string`  | автовизначення   | Ідентифікатор адаптера ембедингів: `openai`, `gemini`, `voyage`, `mistral`, `bedrock`, `ollama`, `local` |
| `model`    | `string`  | типовий для постачальника | Назва моделі ембедингів                                                              |
| `fallback` | `string`  | `"none"`         | Ідентифікатор резервного адаптера, якщо основний завершується помилкою                     |
| `enabled`  | `boolean` | `true`           | Увімкнути або вимкнути пошук у пам’яті                                                     |

### Порядок автовизначення

Коли `provider` не задано, OpenClaw вибирає перший доступний:

1. `local` -- якщо налаштовано `memorySearch.local.modelPath` і файл існує.
2. `openai` -- якщо вдається визначити ключ OpenAI.
3. `gemini` -- якщо вдається визначити ключ Gemini.
4. `voyage` -- якщо вдається визначити ключ Voyage.
5. `mistral` -- якщо вдається визначити ключ Mistral.
6. `bedrock` -- якщо ланцюжок облікових даних AWS SDK успішно визначається (роль екземпляра, ключі доступу, профіль, SSO, вебідентичність або спільна конфігурація).

`ollama` підтримується, але не визначається автоматично (вкажіть його явно).

### Визначення API-ключа

Для віддалених ембедингів потрібен API-ключ. Bedrock натомість використовує
стандартний ланцюжок облікових даних AWS SDK (ролі екземпляра, SSO, ключі доступу).

| Provider | Env var                        | Config key                        |
| -------- | ------------------------------ | --------------------------------- |
| OpenAI   | `OPENAI_API_KEY`               | `models.providers.openai.apiKey`  |
| Gemini   | `GEMINI_API_KEY`               | `models.providers.google.apiKey`  |
| Voyage   | `VOYAGE_API_KEY`               | `models.providers.voyage.apiKey`  |
| Mistral  | `MISTRAL_API_KEY`              | `models.providers.mistral.apiKey` |
| Bedrock  | ланцюжок облікових даних AWS   | API-ключ не потрібен              |
| Ollama   | `OLLAMA_API_KEY` (заповнювач)  | --                                |

Codex OAuth покриває лише chat/completions і не задовольняє запити
ембедингів.

---

## Конфігурація віддаленої кінцевої точки

Для власних OpenAI-сумісних кінцевих точок або перевизначення стандартних значень постачальника:

| Key              | Type     | Description                                        |
| ---------------- | -------- | -------------------------------------------------- |
| `remote.baseUrl` | `string` | Власна базова URL-адреса API                       |
| `remote.apiKey`  | `string` | Перевизначення API-ключа                           |
| `remote.headers` | `object` | Додаткові HTTP-заголовки (об’єднуються з типовими значеннями постачальника) |

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "openai",
        model: "text-embedding-3-small",
        remote: {
          baseUrl: "https://api.example.com/v1/",
          apiKey: "YOUR_KEY",
        },
      },
    },
  },
}
```

---

## Конфігурація, специфічна для Gemini

| Key                    | Type     | Default                | Description                                |
| ---------------------- | -------- | ---------------------- | ------------------------------------------ |
| `model`                | `string` | `gemini-embedding-001` | Також підтримується `gemini-embedding-2-preview` |
| `outputDimensionality` | `number` | `3072`                 | Для Embedding 2: 768, 1536 або 3072        |

<Warning>
Зміна `model` або `outputDimensionality` запускає автоматичну повну переіндексацію.
</Warning>

---

## Конфігурація ембедингів Bedrock

Bedrock використовує стандартний ланцюжок облікових даних AWS SDK -- API-ключі не потрібні.
Якщо OpenClaw працює на EC2 з роллю екземпляра, увімкненою для Bedrock, просто задайте
постачальника і модель:

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "bedrock",
        model: "amazon.titan-embed-text-v2:0",
      },
    },
  },
}
```

| Key                    | Type     | Default                        | Description                     |
| ---------------------- | -------- | ------------------------------ | ------------------------------- |
| `model`                | `string` | `amazon.titan-embed-text-v2:0` | Будь-який ідентифікатор моделі ембедингів Bedrock |
| `outputDimensionality` | `number` | типове значення моделі         | Для Titan V2: 256, 512 або 1024 |

### Підтримувані моделі

Підтримуються такі моделі (з визначенням сімейства та типовими
вимірами):

| Model ID                                   | Provider   | Default Dims | Configurable Dims    |
| ------------------------------------------ | ---------- | ------------ | -------------------- |
| `amazon.titan-embed-text-v2:0`             | Amazon     | 1024         | 256, 512, 1024       |
| `amazon.titan-embed-text-v1`               | Amazon     | 1536         | --                   |
| `amazon.titan-embed-g1-text-02`            | Amazon     | 1536         | --                   |
| `amazon.titan-embed-image-v1`              | Amazon     | 1024         | --                   |
| `amazon.nova-2-multimodal-embeddings-v1:0` | Amazon     | 1024         | 256, 384, 1024, 3072 |
| `cohere.embed-english-v3`                  | Cohere     | 1024         | --                   |
| `cohere.embed-multilingual-v3`             | Cohere     | 1024         | --                   |
| `cohere.embed-v4:0`                        | Cohere     | 1536         | 256-1536             |
| `twelvelabs.marengo-embed-3-0-v1:0`        | TwelveLabs | 512          | --                   |
| `twelvelabs.marengo-embed-2-7-v1:0`        | TwelveLabs | 1024         | --                   |

Варіанти з суфіксом пропускної здатності (наприклад, `amazon.titan-embed-text-v1:2:8k`) успадковують
конфігурацію базової моделі.

### Автентифікація

Автентифікація Bedrock використовує стандартний порядок визначення облікових даних AWS SDK:

1. Змінні середовища (`AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY`)
2. Кеш токенів SSO
3. Облікові дані токена вебідентичності
4. Спільні файли облікових даних і конфігурації
5. Облікові дані метаданих ECS або EC2

Регіон визначається з `AWS_REGION`, `AWS_DEFAULT_REGION`, `baseUrl`
постачальника `amazon-bedrock` або за замовчуванням використовується `us-east-1`.

### Дозволи IAM

Ролі або користувачу IAM потрібні такі дозволи:

```json
{
  "Effect": "Allow",
  "Action": "bedrock:InvokeModel",
  "Resource": "*"
}
```

Для принципу найменших привілеїв обмежте `InvokeModel` конкретною моделлю:

```
arn:aws:bedrock:*::foundation-model/amazon.titan-embed-text-v2:0
```

---

## Конфігурація локальних ембедингів

| Key                   | Type     | Default                | Description                     |
| --------------------- | -------- | ---------------------- | ------------------------------- |
| `local.modelPath`     | `string` | завантажується автоматично | Шлях до файлу моделі GGUF    |
| `local.modelCacheDir` | `string` | типове значення node-llama-cpp | Каталог кешу для завантажених моделей |

Типова модель: `embeddinggemma-300m-qat-Q8_0.gguf` (~0.6 ГБ, завантажується автоматично).
Потрібне нативне збирання: `pnpm approve-builds`, потім `pnpm rebuild node-llama-cpp`.

---

## Конфігурація гібридного пошуку

Усе знаходиться в `memorySearch.query.hybrid`:

| Key                   | Type      | Default | Description                        |
| --------------------- | --------- | ------- | ---------------------------------- |
| `enabled`             | `boolean` | `true`  | Увімкнути гібридний пошук BM25 + векторний пошук |
| `vectorWeight`        | `number`  | `0.7`   | Вага для векторних оцінок (0-1)    |
| `textWeight`          | `number`  | `0.3`   | Вага для оцінок BM25 (0-1)         |
| `candidateMultiplier` | `number`  | `4`     | Множник розміру пулу кандидатів    |

### MMR (різноманітність)

| Key           | Type      | Default | Description                          |
| ------------- | --------- | ------- | ------------------------------------ |
| `mmr.enabled` | `boolean` | `false` | Увімкнути повторне ранжування MMR    |
| `mmr.lambda`  | `number`  | `0.7`   | 0 = максимальна різноманітність, 1 = максимальна релевантність |

### Часове згасання (актуальність)

| Key                          | Type      | Default | Description               |
| ---------------------------- | --------- | ------- | ------------------------- |
| `temporalDecay.enabled`      | `boolean` | `false` | Увімкнути підсилення за актуальністю |
| `temporalDecay.halfLifeDays` | `number`  | `30`    | Оцінка зменшується вдвічі кожні N днів |

Для evergreen-файлів (`MEMORY.md`, файли без дати в `memory/`) згасання ніколи не застосовується.

### Повний приклад

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        query: {
          hybrid: {
            vectorWeight: 0.7,
            textWeight: 0.3,
            mmr: { enabled: true, lambda: 0.7 },
            temporalDecay: { enabled: true, halfLifeDays: 30 },
          },
        },
      },
    },
  },
}
```

---

## Додаткові шляхи до пам’яті

| Key          | Type       | Description                              |
| ------------ | ---------- | ---------------------------------------- |
| `extraPaths` | `string[]` | Додаткові каталоги або файли для індексації |

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        extraPaths: ["../team-docs", "/srv/shared-notes"],
      },
    },
  },
}
```

Шляхи можуть бути абсолютними або відносними до робочого простору. Каталоги
скануються рекурсивно на наявність файлів `.md`. Обробка символьних посилань залежить від активного бекенда:
вбудований рушій ігнорує символьні посилання, тоді як QMD наслідує поведінку
базового сканера QMD.

Для пошуку транскриптів між агентами в межах агента використовуйте
`agents.list[].memorySearch.qmd.extraCollections` замість `memory.qmd.paths`.
Ці додаткові колекції мають ту саму структуру `{ path, name, pattern? }`, але
об’єднуються для кожного агента й можуть зберігати явні спільні назви, коли шлях
вказує за межі поточного робочого простору.
Якщо той самий визначений шлях з’являється і в `memory.qmd.paths`, і в
`memorySearch.qmd.extraCollections`, QMD зберігає перший запис і пропускає
дублікат.

---

## Мультимодальна пам’ять (Gemini)

Індексуйте зображення та аудіо разом із Markdown за допомогою Gemini Embedding 2:

| Key                       | Type       | Default    | Description                            |
| ------------------------- | ---------- | ---------- | -------------------------------------- |
| `multimodal.enabled`      | `boolean`  | `false`    | Увімкнути мультимодальну індексацію    |
| `multimodal.modalities`   | `string[]` | --         | `["image"]`, `["audio"]` або `["all"]` |
| `multimodal.maxFileBytes` | `number`   | `10000000` | Максимальний розмір файлу для індексації |

Застосовується лише до файлів у `extraPaths`. Типові корені пам’яті залишаються лише для Markdown.
Потрібен `gemini-embedding-2-preview`. Значення `fallback` має бути `"none"`.

Підтримувані формати: `.jpg`, `.jpeg`, `.png`, `.webp`, `.gif`, `.heic`, `.heif`
(зображення); `.mp3`, `.wav`, `.ogg`, `.opus`, `.m4a`, `.aac`, `.flac` (аудіо).

---

## Кеш ембедингів

| Key                | Type      | Default | Description                      |
| ------------------ | --------- | ------- | -------------------------------- |
| `cache.enabled`    | `boolean` | `false` | Кешувати ембединги чанків у SQLite |
| `cache.maxEntries` | `number`  | `50000` | Максимальна кількість кешованих ембедингів |

Запобігає повторному створенню ембедингів для незміненого тексту під час переіндексації або оновлення транскриптів.

---

## Пакетна індексація

| Key                           | Type      | Default | Description                |
| ----------------------------- | --------- | ------- | -------------------------- |
| `remote.batch.enabled`        | `boolean` | `false` | Увімкнути API пакетних ембедингів |
| `remote.batch.concurrency`    | `number`  | `2`     | Паралельні пакетні завдання |
| `remote.batch.wait`           | `boolean` | `true`  | Очікувати завершення пакета |
| `remote.batch.pollIntervalMs` | `number`  | --      | Інтервал опитування        |
| `remote.batch.timeoutMinutes` | `number`  | --      | Тайм-аут пакета            |

Доступно для `openai`, `gemini` і `voyage`. Пакетний режим OpenAI зазвичай
є найшвидшим і найдешевшим для великих зворотних заповнень.

---

## Пошук пам’яті сесії (експериментально)

Індексує транскрипти сесій і показує їх через `memory_search`:

| Key                           | Type       | Default      | Description                             |
| ----------------------------- | ---------- | ------------ | --------------------------------------- |
| `experimental.sessionMemory`  | `boolean`  | `false`      | Увімкнути індексацію сесій              |
| `sources`                     | `string[]` | `["memory"]` | Додайте `"sessions"`, щоб включити транскрипти |
| `sync.sessions.deltaBytes`    | `number`   | `100000`     | Поріг байтів для переіндексації         |
| `sync.sessions.deltaMessages` | `number`   | `50`         | Поріг повідомлень для переіндексації    |

Індексація сесій вмикається за бажанням і працює асинхронно. Результати можуть бути трохи
застарілими. Журнали сесій зберігаються на диску, тому розглядайте доступ до файлової системи як межу довіри.

---

## Прискорення векторів SQLite (sqlite-vec)

| Key                          | Type      | Default | Description                       |
| ---------------------------- | --------- | ------- | --------------------------------- |
| `store.vector.enabled`       | `boolean` | `true`  | Використовувати sqlite-vec для векторних запитів |
| `store.vector.extensionPath` | `string`  | вбудовано | Перевизначити шлях до sqlite-vec |

Коли sqlite-vec недоступний, OpenClaw автоматично повертається до
подібності косинусів у процесі.

---

## Сховище індексу

| Key                   | Type     | Default                               | Description                                 |
| --------------------- | -------- | ------------------------------------- | ------------------------------------------- |
| `store.path`          | `string` | `~/.openclaw/memory/{agentId}.sqlite` | Розташування індексу (підтримує токен `{agentId}`) |
| `store.fts.tokenizer` | `string` | `unicode61`                           | Токенізатор FTS5 (`unicode61` або `trigram`) |

---

## Конфігурація бекенда QMD

Щоб увімкнути, задайте `memory.backend = "qmd"`. Усі налаштування QMD знаходяться в
`memory.qmd`:

| Key                      | Type      | Default  | Description                                  |
| ------------------------ | --------- | -------- | -------------------------------------------- |
| `command`                | `string`  | `qmd`    | Шлях до виконуваного файла QMD               |
| `searchMode`             | `string`  | `search` | Команда пошуку: `search`, `vsearch`, `query` |
| `includeDefaultMemory`   | `boolean` | `true`   | Автоматично індексувати `MEMORY.md` + `memory/**/*.md` |
| `paths[]`                | `array`   | --       | Додаткові шляхи: `{ name, path, pattern? }`  |
| `sessions.enabled`       | `boolean` | `false`  | Індексувати транскрипти сесій                |
| `sessions.retentionDays` | `number`  | --       | Зберігання транскриптів                      |
| `sessions.exportDir`     | `string`  | --       | Каталог експорту                             |

OpenClaw надає перевагу поточним формам колекцій QMD і запитів MCP, але
підтримує роботу старіших випусків QMD, повертаючись до застарілих прапорців колекцій `--mask`
і старіших назв інструментів MCP, коли це потрібно.

Перевизначення моделей QMD залишаються на боці QMD, а не в конфігурації OpenClaw. Якщо вам потрібно
глобально перевизначити моделі QMD, задайте такі змінні середовища, як
`QMD_EMBED_MODEL`, `QMD_RERANK_MODEL` і `QMD_GENERATE_MODEL` у середовищі
виконання Gateway.

### Розклад оновлення

| Key                       | Type      | Default | Description                           |
| ------------------------- | --------- | ------- | ------------------------------------- |
| `update.interval`         | `string`  | `5m`    | Інтервал оновлення                    |
| `update.debounceMs`       | `number`  | `15000` | Усунення дребезгу змін файлів         |
| `update.onBoot`           | `boolean` | `true`  | Оновлювати під час запуску            |
| `update.waitForBootSync`  | `boolean` | `false` | Блокувати запуск до завершення оновлення |
| `update.embedInterval`    | `string`  | --      | Окремий інтервал для ембедингів       |
| `update.commandTimeoutMs` | `number`  | --      | Тайм-аут для команд QMD               |
| `update.updateTimeoutMs`  | `number`  | --      | Тайм-аут для операцій оновлення QMD   |
| `update.embedTimeoutMs`   | `number`  | --      | Тайм-аут для операцій ембедингів QMD  |

### Обмеження

| Key                       | Type     | Default | Description                |
| ------------------------- | -------- | ------- | -------------------------- |
| `limits.maxResults`       | `number` | `6`     | Максимальна кількість результатів пошуку |
| `limits.maxSnippetChars`  | `number` | --      | Обмежити довжину фрагмента |
| `limits.maxInjectedChars` | `number` | --      | Обмежити загальну кількість вставлених символів |
| `limits.timeoutMs`        | `number` | `4000`  | Тайм-аут пошуку            |

### Область дії

Керує тим, які сесії можуть отримувати результати пошуку QMD. Та сама схема, що й у
[`session.sendPolicy`](/uk/gateway/configuration-reference#session):

```json5
{
  memory: {
    qmd: {
      scope: {
        default: "deny",
        rules: [{ action: "allow", match: { chatType: "direct" } }],
      },
    },
  },
}
```

Типове постачання дозволяє direct і channel сесії, але все одно забороняє
групи.

За замовчуванням — лише DM. `match.keyPrefix` збігається з нормалізованим ключем сесії;
`match.rawKeyPrefix` збігається з сирим ключем, включно з `agent:<id>:`.

### Цитування

`memory.citations` застосовується до всіх бекендів:

| Value            | Behavior                                            |
| ---------------- | --------------------------------------------------- |
| `auto` (default) | Додавати нижній колонтитул `Source: <path#line>` у фрагменти |
| `on`             | Завжди додавати нижній колонтитул                   |
| `off`            | Не додавати нижній колонтитул (шлях усе одно передається агенту внутрішньо) |

### Повний приклад QMD

```json5
{
  memory: {
    backend: "qmd",
    citations: "auto",
    qmd: {
      includeDefaultMemory: true,
      update: { interval: "5m", debounceMs: 15000 },
      limits: { maxResults: 6, timeoutMs: 4000 },
      scope: {
        default: "deny",
        rules: [{ action: "allow", match: { chatType: "direct" } }],
      },
      paths: [{ name: "docs", path: "~/notes", pattern: "**/*.md" }],
    },
  },
}
```

---

## Dreaming (експериментально)

Dreaming налаштовується в `plugins.entries.memory-core.config.dreaming`,
а не в `agents.defaults.memorySearch`.

Dreaming запускається як один запланований прохід і використовує внутрішні фази light/deep/REM як
деталь реалізації.

Про концептуальну поведінку та slash-команди дивіться [Dreaming](/uk/concepts/dreaming).

### Налаштування користувача

| Key         | Type      | Default     | Description                                       |
| ----------- | --------- | ----------- | ------------------------------------------------- |
| `enabled`   | `boolean` | `false`     | Повністю увімкнути або вимкнути Dreaming          |
| `frequency` | `string`  | `0 3 * * *` | Необов’язкова Cron-частота для повного циклу Dreaming |

### Приклад

```json5
{
  plugins: {
    entries: {
      "memory-core": {
        config: {
          dreaming: {
            enabled: true,
            frequency: "0 3 * * *",
          },
        },
      },
    },
  },
}
```

Примітки:

- Dreaming записує машинний стан у `memory/.dreams/`.
- Dreaming записує читабельний для людини описовий вивід у `DREAMS.md` (або наявний `dreams.md`).
- Політика фаз light/deep/REM і пороги є внутрішньою поведінкою, а не користувацькою конфігурацією.
