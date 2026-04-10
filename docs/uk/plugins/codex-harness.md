---
read_when:
    - Ви хочете використовувати вбудований каркас app-server Codex
    - Вам потрібні посилання на моделі Codex і приклади конфігурації
    - Ви хочете вимкнути резервний перехід на PI для розгортань лише з Codex
summary: Запускайте вбудовані ходи агента OpenClaw через вбудований каркас app-server Codex
title: Каркас Codex Harness
x-i18n:
    generated_at: "2026-04-10T21:20:36Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7ba7e0eeb264f846a66a927baf6ded83438a7b4a3df86ee9fe562105f5bb05aa
    source_path: plugins/codex-harness.md
    workflow: 15
---

# Каркас Codex

Вбудований плагін `codex` дозволяє OpenClaw запускати вбудовані ходи агента через
app-server Codex замість вбудованого каркаса PI.

Використовуйте це, коли хочете, щоб Codex керував низькорівневою сесією агента: виявленням
моделей, нативним відновленням потоку, нативним ущільненням і виконанням app-server.
OpenClaw, як і раніше, керує каналами чату, файлами сесій, вибором моделі, інструментами,
погодженнями, доставкою медіа та видимим дзеркалом транскрипту.

Каркас вимкнено за замовчуванням. Він вибирається лише тоді, коли плагін `codex`
увімкнено і визначена модель є моделлю `codex/*`, або коли ви явно
примусово задаєте `embeddedHarness.runtime: "codex"` чи `OPENCLAW_AGENT_RUNTIME=codex`.
Якщо ви ніколи не налаштовуєте `codex/*`, наявні запуск PI, OpenAI, Anthropic, Gemini, local
і custom-provider зберігають поточну поведінку.

## Виберіть правильний префікс моделі

OpenClaw має окремі маршрути для доступу у форматі OpenAI та Codex:

| Посилання на модель   | Шлях виконання                              | Використовуйте, коли                                                     |
| --------------------- | ------------------------------------------- | ------------------------------------------------------------------------ |
| `openai/gpt-5.4`      | Провайдер OpenAI через інфраструктуру OpenClaw/PI | Ви хочете прямий доступ до OpenAI Platform API з `OPENAI_API_KEY`.       |
| `openai-codex/gpt-5.4` | Провайдер OpenAI Codex OAuth через PI      | Ви хочете ChatGPT/Codex OAuth без каркаса app-server Codex.              |
| `codex/gpt-5.4`       | Вбудований провайдер Codex плюс каркас Codex | Ви хочете нативне виконання app-server Codex для вбудованого ходу агента. |

Каркас Codex обробляє лише посилання на моделі `codex/*`. Наявні посилання `openai/*`,
`openai-codex/*`, Anthropic, Gemini, xAI, local і custom provider зберігають
свої звичайні шляхи.

## Вимоги

- OpenClaw із доступним вбудованим плагіном `codex`.
- App-server Codex `0.118.0` або новіший.
- Автентифікація Codex, доступна для процесу app-server.

Плагін блокує старіші або неверсійовані узгодження app-server. Це зберігає
роботу OpenClaw на поверхні протоколу, з якою його було протестовано.

Для живих і Docker smoke-тестів автентифікація зазвичай надходить із `OPENAI_API_KEY`, а також
необов’язкових файлів Codex CLI, таких як `~/.codex/auth.json` і
`~/.codex/config.toml`. Використовуйте ті самі облікові дані автентифікації, що й ваш локальний app-server Codex.

## Мінімальна конфігурація

Використовуйте `codex/gpt-5.4`, увімкніть вбудований плагін і примусово задайте каркас `codex`:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
  agents: {
    defaults: {
      model: "codex/gpt-5.4",
      embeddedHarness: {
        runtime: "codex",
        fallback: "none",
      },
    },
  },
}
```

Якщо у вашій конфігурації використовується `plugins.allow`, також включіть туди `codex`:

```json5
{
  plugins: {
    allow: ["codex"],
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
}
```

Установлення `agents.defaults.model` або моделі агента на `codex/<model>` також
автоматично вмикає вбудований плагін `codex`. Явний запис плагіна все одно
корисний у спільних конфігураціях, оскільки робить намір розгортання очевидним.

## Додайте Codex без заміни інших моделей

Залишайте `runtime: "auto"`, коли хочете використовувати Codex для моделей `codex/*`, а PI — для
всього іншого:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
  agents: {
    defaults: {
      model: {
        primary: "codex/gpt-5.4",
        fallbacks: ["openai/gpt-5.4", "anthropic/claude-opus-4-6"],
      },
      models: {
        "codex/gpt-5.4": { alias: "codex" },
        "codex/gpt-5.4-mini": { alias: "codex-mini" },
        "openai/gpt-5.4": { alias: "gpt" },
        "anthropic/claude-opus-4-6": { alias: "opus" },
      },
      embeddedHarness: {
        runtime: "auto",
        fallback: "pi",
      },
    },
  },
}
```

За такої форми:

- `/model codex` або `/model codex/gpt-5.4` використовує каркас app-server Codex.
- `/model gpt` або `/model openai/gpt-5.4` використовує шлях провайдера OpenAI.
- `/model opus` використовує шлях провайдера Anthropic.
- Якщо вибрано не-Codex модель, PI залишається каркасом сумісності.

## Розгортання лише з Codex

Вимкніть резервний перехід на PI, коли потрібно довести, що кожен вбудований хід агента використовує
каркас Codex:

```json5
{
  agents: {
    defaults: {
      model: "codex/gpt-5.4",
      embeddedHarness: {
        runtime: "codex",
        fallback: "none",
      },
    },
  },
}
```

Перевизначення через середовище:

```bash
OPENCLAW_AGENT_RUNTIME=codex \
OPENCLAW_AGENT_HARNESS_FALLBACK=none \
openclaw gateway run
```

Коли резервний перехід вимкнено, OpenClaw завершується з помилкою на ранньому етапі, якщо плагін Codex вимкнено,
запитувана модель не є посиланням `codex/*`, app-server надто старий або
app-server не може запуститися.

## Codex для окремого агента

Ви можете зробити один агент лише з Codex, тоді як агент за замовчуванням зберігатиме звичайний
автовибір:

```json5
{
  agents: {
    defaults: {
      embeddedHarness: {
        runtime: "auto",
        fallback: "pi",
      },
    },
    list: [
      {
        id: "main",
        default: true,
        model: "anthropic/claude-opus-4-6",
      },
      {
        id: "codex",
        name: "Codex",
        model: "codex/gpt-5.4",
        embeddedHarness: {
          runtime: "codex",
          fallback: "none",
        },
      },
    ],
  },
}
```

Використовуйте звичайні команди сесії, щоб перемикати агентів і моделі. `/new` створює нову
сесію OpenClaw, а каркас Codex створює або відновлює свій допоміжний потік app-server
за потреби. `/reset` очищає прив’язку сесії OpenClaw для цього потоку.

## Виявлення моделей

За замовчуванням плагін Codex запитує доступні моделі в app-server. Якщо
виявлення завершується помилкою або перевищує час очікування, використовується вбудований резервний каталог:

- `codex/gpt-5.4`
- `codex/gpt-5.4-mini`
- `codex/gpt-5.2`

Ви можете налаштувати виявлення в `plugins.entries.codex.config.discovery`:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          discovery: {
            enabled: true,
            timeoutMs: 2500,
          },
        },
      },
    },
  },
}
```

Вимкніть виявлення, коли хочете, щоб під час запуску не виконувалася перевірка Codex і використовувався
резервний каталог:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          discovery: {
            enabled: false,
          },
        },
      },
    },
  },
}
```

## Підключення до app-server і політика

За замовчуванням плагін запускає Codex локально за допомогою:

```bash
codex app-server --listen stdio://
```

Ви можете зберегти це значення за замовчуванням і лише налаштувати нативну політику Codex:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            approvalPolicy: "on-request",
            sandbox: "workspace-write",
            serviceTier: "priority",
          },
        },
      },
    },
  },
}
```

Для вже запущеного app-server використовуйте транспорт WebSocket:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            transport: "websocket",
            url: "ws://127.0.0.1:39175",
            authToken: "${CODEX_APP_SERVER_TOKEN}",
            requestTimeoutMs: 60000,
          },
        },
      },
    },
  },
}
```

Підтримувані поля `appServer`:

| Поле                | Типове значення                           | Значення                                                                 |
| ------------------- | ----------------------------------------- | ------------------------------------------------------------------------ |
| `transport`         | `"stdio"`                                 | `"stdio"` запускає Codex; `"websocket"` підключається до `url`.          |
| `command`           | `"codex"`                                 | Виконуваний файл для транспорту stdio.                                   |
| `args`              | `["app-server", "--listen", "stdio://"]`  | Аргументи для транспорту stdio.                                          |
| `url`               | не задано                                 | URL WebSocket app-server.                                                |
| `authToken`         | не задано                                 | Bearer-токен для транспорту WebSocket.                                   |
| `headers`           | `{}`                                      | Додаткові заголовки WebSocket.                                           |
| `requestTimeoutMs`  | `60000`                                   | Час очікування для викликів площини керування app-server.                |
| `approvalPolicy`    | `"never"`                                 | Нативна політика погодження Codex, що надсилається на старт/відновлення/хід потоку. |
| `sandbox`           | `"workspace-write"`                       | Нативний режим sandbox Codex, що надсилається на старт/відновлення потоку. |
| `approvalsReviewer` | `"user"`                                  | Використовуйте `"guardian_subagent"`, щоб дозволити guardian Codex переглядати нативні погодження. |
| `serviceTier`       | не задано                                 | Необов’язковий рівень сервісу Codex, наприклад `"priority"`.             |

Старі змінні середовища все ще працюють як резервні значення для локального тестування, коли
відповідне поле конфігурації не задано:

- `OPENCLAW_CODEX_APP_SERVER_BIN`
- `OPENCLAW_CODEX_APP_SERVER_ARGS`
- `OPENCLAW_CODEX_APP_SERVER_APPROVAL_POLICY`
- `OPENCLAW_CODEX_APP_SERVER_SANDBOX`
- `OPENCLAW_CODEX_APP_SERVER_GUARDIAN=1`

Для відтворюваних розгортань перевага надається конфігурації.

## Команда Codex

Вбудований плагін реєструє `/codex` як авторизовану slash-команду. Вона
універсальна і працює в будь-якому каналі, який підтримує текстові команди OpenClaw.

Поширені форми:

- `/codex status` показує живе підключення до app-server, моделі, обліковий запис, ліміти швидкості, MCP-сервери та Skills.
- `/codex models` показує список живих моделей app-server Codex.
- `/codex threads [filter]` показує список нещодавніх потоків Codex.
- `/codex resume <thread-id>` прив’язує поточну сесію OpenClaw до наявного потоку Codex.
- `/codex compact` просить app-server Codex ущільнити прив’язаний потік.
- `/codex review` запускає нативну перевірку Codex для прив’язаного потоку.
- `/codex account` показує стан облікового запису та лімітів швидкості.
- `/codex mcp` показує стан MCP-серверів app-server Codex.
- `/codex skills` показує список Skills app-server Codex.

`/codex resume` записує той самий файл допоміжної прив’язки, який каркас використовує для
звичайних ходів. У наступному повідомленні OpenClaw відновлює цей потік Codex, передає
поточну вибрану модель OpenClaw `codex/*` до app-server і залишає розширену
історію ввімкненою.

## Інструменти, медіа та ущільнення

Каркас Codex змінює лише низькорівневий виконавець вбудованого агента.

OpenClaw, як і раніше, формує список інструментів і отримує динамічні результати інструментів від
каркаса. Текст, зображення, відео, музика, TTS, погодження та вивід інструментів обміну повідомленнями
продовжують проходити через звичайний шлях доставки OpenClaw.

Коли вибрана модель використовує каркас Codex, нативне ущільнення потоку
делегується app-server Codex. OpenClaw зберігає дзеркало транскрипту для історії каналу,
пошуку, `/new`, `/reset` і майбутнього перемикання моделі або каркаса.

Генерація медіа не потребує PI. Генерація зображень, відео, музики, PDF, TTS та
розуміння медіа й далі використовують відповідні налаштування провайдера/моделі, такі як
`agents.defaults.imageGenerationModel`, `videoGenerationModel`, `pdfModel` і
`messages.tts`.

## Усунення несправностей

**Codex не з’являється в `/model`:** увімкніть `plugins.entries.codex.enabled`,
задайте посилання на модель `codex/*` або перевірте, чи `plugins.allow` не виключає `codex`.

**OpenClaw переходить на PI:** задайте `embeddedHarness.fallback: "none"` або
`OPENCLAW_AGENT_HARNESS_FALLBACK=none` під час тестування.

**App-server відхиляється:** оновіть Codex, щоб узгодження app-server
повідомляло версію `0.118.0` або новішу.

**Виявлення моделей повільне:** зменште `plugins.entries.codex.config.discovery.timeoutMs`
або вимкніть виявлення.

**Транспорт WebSocket одразу завершується помилкою:** перевірте `appServer.url`, `authToken`
і те, що віддалений app-server підтримує ту саму версію протоколу app-server Codex.

**Модель не Codex використовує PI:** це очікувано. Каркас Codex обробляє лише
посилання на моделі `codex/*`.

## Пов’язане

- [Плагіни каркаса агента](/uk/plugins/sdk-agent-harness)
- [Провайдери моделей](/uk/concepts/model-providers)
- [Довідник із конфігурації](/uk/gateway/configuration-reference)
- [Тестування](/uk/help/testing#live-codex-app-server-harness-smoke)
