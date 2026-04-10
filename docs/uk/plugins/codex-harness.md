---
read_when:
    - Ви хочете використовувати вбудований harness app-server Codex
    - Вам потрібні посилання на модель Codex і приклади конфігурації
    - Ви хочете вимкнути резервний перехід на Pi для розгортань лише з Codex
summary: Запускайте вбудовані ходи агента OpenClaw через вбудований harness app-server Codex
title: Harness Codex
x-i18n:
    generated_at: "2026-04-10T21:45:23Z"
    model: gpt-5.4
    provider: openai
    source_hash: 93aa65d08b8ffd39625d21c054856edb75a1e12d66fc7c6fae343fe398abab8b
    source_path: plugins/codex-harness.md
    workflow: 15
---

# Harness Codex

Вбудований плагін `codex` дає OpenClaw змогу запускати вбудовані ходи агента через
app-server Codex замість вбудованого harness Pi.

Використовуйте це, коли хочете, щоб Codex керував низькорівневою сесією агента: виявленням
моделей, нативним відновленням нитки, нативною компакцією та виконанням через app-server.
OpenClaw і далі керує каналами чату, файлами сесій, вибором моделі, інструментами,
погодженнями, доставкою медіа та видимим дзеркалом транскрипту.

Harness вимкнено за замовчуванням. Його вибирають лише тоді, коли плагін `codex`
увімкнено і визначена модель є моделлю `codex/*`, або коли ви явно
примусово задаєте `embeddedHarness.runtime: "codex"` чи `OPENCLAW_AGENT_RUNTIME=codex`.
Якщо ви взагалі не налаштовуєте `codex/*`, наявні запуски PI, OpenAI, Anthropic, Gemini, local
і custom-provider зберігають поточну поведінку.

## Виберіть правильний префікс моделі

OpenClaw має окремі маршрути для доступу у форматі OpenAI та Codex:

| Посилання на модель   | Шлях середовища виконання                   | Використовуйте, коли                                                     |
| --------------------- | ------------------------------------------- | ------------------------------------------------------------------------ |
| `openai/gpt-5.4`      | Провайдер OpenAI через механізми OpenClaw/PI | Ви хочете прямий доступ до API OpenAI Platform за допомогою `OPENAI_API_KEY`. |
| `openai-codex/gpt-5.4` | Провайдер OpenAI Codex OAuth через PI      | Ви хочете ChatGPT/Codex OAuth без harness app-server Codex.              |
| `codex/gpt-5.4`       | Вбудований провайдер Codex плюс harness Codex | Ви хочете нативне виконання через app-server Codex для вбудованого ходу агента. |

Harness Codex працює лише з посиланнями на моделі `codex/*`. Наявні посилання `openai/*`,
`openai-codex/*`, Anthropic, Gemini, xAI, local і custom provider зберігають
свої звичайні шляхи.

## Вимоги

- OpenClaw із доступним вбудованим плагіном `codex`.
- App-server Codex версії `0.118.0` або новішої.
- Автентифікація Codex, доступна для процесу app-server.

Плагін блокує старіші або безверсійні handshake app-server. Це дає змогу
OpenClaw працювати лише з тією поверхнею протоколу, з якою його було протестовано.

Для live- і Docker smoke-тестів автентифікація зазвичай надходить з `OPENAI_API_KEY`, а також
із необов’язкових файлів CLI Codex, таких як `~/.codex/auth.json` і
`~/.codex/config.toml`. Використовуйте ті самі матеріали автентифікації, що й ваш локальний
app-server Codex.

## Мінімальна конфігурація

Використовуйте `codex/gpt-5.4`, увімкніть вбудований плагін і примусово задайте harness `codex`:

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

Якщо у вашій конфігурації використовується `plugins.allow`, додайте туди також `codex`:

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

Налаштування `agents.defaults.model` або моделі агента на `codex/<model>` також
автоматично вмикає вбудований плагін `codex`. Явний запис плагіна все одно
корисний у спільних конфігураціях, оскільки робить намір розгортання очевидним.

## Додайте Codex без заміни інших моделей

Залишайте `runtime: "auto"`, якщо хочете використовувати Codex для моделей `codex/*` і PI для
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

У такій конфігурації:

- `/model codex` або `/model codex/gpt-5.4` використовує harness app-server Codex.
- `/model gpt` або `/model openai/gpt-5.4` використовує шлях провайдера OpenAI.
- `/model opus` використовує шлях провайдера Anthropic.
- Якщо вибрано не-Codex модель, PI залишається harness сумісності.

## Розгортання лише з Codex

Вимкніть резервний перехід на Pi, якщо вам потрібно довести, що кожен вбудований хід агента використовує
harness Codex:

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

Якщо резервний перехід вимкнено, OpenClaw завершує роботу з помилкою на ранньому етапі, якщо плагін Codex вимкнено,
запитана модель не є посиланням `codex/*`, app-server надто старий або
app-server не може запуститися.

## Codex для окремого агента

Ви можете зробити один агент лише для Codex, тоді як типовий агент зберігатиме звичайний
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

Використовуйте звичайні команди сесії для перемикання агентів і моделей. `/new` створює нову
сесію OpenClaw, а harness Codex створює або відновлює власну sidecar-нитку
app-server за потреби. `/reset` очищає прив’язку сесії OpenClaw для цієї нитки.

## Виявлення моделей

За замовчуванням плагін Codex запитує app-server про доступні моделі. Якщо
виявлення не вдається або перевищує час очікування, він використовує вбудований резервний каталог:

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

Вимкніть виявлення, якщо хочете, щоб під час запуску не виконувалась перевірка Codex і використовувався лише
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

## Підключення app-server і політика

За замовчуванням плагін локально запускає Codex так:

```bash
codex app-server --listen stdio://
```

Ви можете зберегти це значення за замовчуванням і налаштувати лише нативну політику Codex:

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

| Поле                | За замовчуванням                           | Значення                                                                  |
| ------------------- | ----------------------------------------- | ------------------------------------------------------------------------- |
| `transport`         | `"stdio"`                                 | `"stdio"` запускає Codex; `"websocket"` підключається до `url`.           |
| `command`           | `"codex"`                                 | Виконуваний файл для транспорту stdio.                                    |
| `args`              | `["app-server", "--listen", "stdio://"]`  | Аргументи для транспорту stdio.                                           |
| `url`               | не задано                                 | URL app-server WebSocket.                                                 |
| `authToken`         | не задано                                 | Bearer-токен для транспорту WebSocket.                                    |
| `headers`           | `{}`                                      | Додаткові заголовки WebSocket.                                            |
| `requestTimeoutMs`  | `60000`                                   | Час очікування для викликів control-plane app-server.                     |
| `approvalPolicy`    | `"never"`                                 | Нативна політика погодження Codex, що надсилається під час старту/відновлення/ходу нитки. |
| `sandbox`           | `"workspace-write"`                       | Нативний режим sandbox Codex, що надсилається під час старту/відновлення нитки. |
| `approvalsReviewer` | `"user"`                                  | Використовуйте `"guardian_subagent"`, щоб guardian Codex перевіряв нативні погодження. |
| `serviceTier`       | не задано                                 | Необов’язковий рівень сервісу Codex, наприклад `"priority"`.              |

Старі змінні середовища все ще працюють як резервні значення для локального тестування, коли
відповідне поле конфігурації не задане:

- `OPENCLAW_CODEX_APP_SERVER_BIN`
- `OPENCLAW_CODEX_APP_SERVER_ARGS`
- `OPENCLAW_CODEX_APP_SERVER_APPROVAL_POLICY`
- `OPENCLAW_CODEX_APP_SERVER_SANDBOX`
- `OPENCLAW_CODEX_APP_SERVER_GUARDIAN=1`

Для відтворюваних розгортань перевагу слід надавати конфігурації.

## Типові рецепти

Локальний Codex зі стандартним транспортом stdio:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
}
```

Перевірка harness лише з Codex із вимкненим резервним переходом на Pi:

```json5
{
  embeddedHarness: {
    fallback: "none",
  },
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
}
```

Погодження Codex із перевіркою guardian:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            approvalPolicy: "on-request",
            approvalsReviewer: "guardian_subagent",
            sandbox: "workspace-write",
          },
        },
      },
    },
  },
}
```

Віддалений app-server із явними заголовками:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            transport: "websocket",
            url: "ws://gateway-host:39175",
            headers: {
              "X-OpenClaw-Agent": "main",
            },
          },
        },
      },
    },
  },
}
```

Перемикання моделей і далі контролюється OpenClaw. Коли сесію OpenClaw приєднано
до наявної нитки Codex, наступний хід знову надсилає до
app-server поточну вибрану модель `codex/*`, провайдера, політику погодження, sandbox і service tier.
Перемикання з `codex/gpt-5.4` на `codex/gpt-5.2` зберігає прив’язку
нитки, але просить Codex продовжити роботу з новою вибраною моделлю.

## Команда Codex

Вбудований плагін реєструє `/codex` як авторизовану slash-команду. Вона є
універсальною і працює в будь-якому каналі, що підтримує текстові команди OpenClaw.

Поширені форми:

- `/codex status` показує живе підключення до app-server, моделі, обліковий запис, ліміти швидкості, MCP-сервери та Skills.
- `/codex models` показує список живих моделей app-server Codex.
- `/codex threads [filter]` показує список нещодавніх ниток Codex.
- `/codex resume <thread-id>` приєднує поточну сесію OpenClaw до наявної нитки Codex.
- `/codex compact` просить app-server Codex стиснути приєднану нитку.
- `/codex review` запускає нативну перевірку Codex для приєднаної нитки.
- `/codex account` показує стан облікового запису та лімітів швидкості.
- `/codex mcp` показує список станів MCP-серверів app-server Codex.
- `/codex skills` показує список Skills app-server Codex.

`/codex resume` записує той самий файл sidecar-прив’язки, який harness використовує для
звичайних ходів. У наступному повідомленні OpenClaw відновлює цю нитку Codex, передає
поточну вибрану модель OpenClaw `codex/*` до app-server і зберігає
увімкнену розширену історію.

## Інструменти, медіа та компакція

Harness Codex змінює лише низькорівневий виконавець вбудованого агента.

OpenClaw і далі формує список інструментів і отримує динамічні результати інструментів від
harness. Текст, зображення, відео, музика, TTS, погодження та вивід інструментів повідомлень
і далі проходять через звичайний шлях доставки OpenClaw.

Коли вибрана модель використовує harness Codex, нативна компакція нитки
делегується app-server Codex. OpenClaw зберігає дзеркало транскрипту для історії каналу,
пошуку, `/new`, `/reset` і майбутнього перемикання моделі або harness.

Генерація медіа не потребує PI. Генерація зображень, відео, музики, PDF, TTS і
аналіз медіа й далі використовують відповідні налаштування провайдера/моделі, як-от
`agents.defaults.imageGenerationModel`, `videoGenerationModel`, `pdfModel` і
`messages.tts`.

## Усунення проблем

**Codex не з’являється в `/model`:** увімкніть `plugins.entries.codex.enabled`,
задайте посилання на модель `codex/*` або перевірте, чи `plugins.allow` не виключає `codex`.

**OpenClaw переходить на резервний Pi:** задайте `embeddedHarness.fallback: "none"` або
`OPENCLAW_AGENT_HARNESS_FALLBACK=none` під час тестування.

**App-server відхиляється:** оновіть Codex, щоб handshake app-server
повідомляв версію `0.118.0` або новішу.

**Виявлення моделей повільне:** зменште `plugins.entries.codex.config.discovery.timeoutMs`
або вимкніть виявлення.

**Транспорт WebSocket одразу завершується з помилкою:** перевірте `appServer.url`, `authToken`
і те, що віддалений app-server підтримує ту саму версію протоколу app-server Codex.

**Не-Codex модель використовує PI:** це очікувано. Harness Codex працює лише з
посиланнями на моделі `codex/*`.

## Пов’язано

- [Плагіни harness агента](/uk/plugins/sdk-agent-harness)
- [Провайдери моделей](/uk/concepts/model-providers)
- [Довідник із конфігурації](/uk/gateway/configuration-reference)
- [Тестування](/uk/help/testing#live-codex-app-server-harness-smoke)
