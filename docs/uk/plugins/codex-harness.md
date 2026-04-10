---
read_when:
    - Ви хочете використовувати вбудований app-server harness Codex
    - Вам потрібні посилання на моделі Codex і приклади конфігурації
    - Ви хочете вимкнути резервний перехід на PI для розгортань лише з Codex
summary: Запустіть ходи вбудованого агента OpenClaw через вбудований app-server harness Codex
title: Codex Harness
x-i18n:
    generated_at: "2026-04-10T22:09:02Z"
    model: gpt-5.4
    provider: openai
    source_hash: 71f4f8e67cfc8f573ad8f082dd4fc36ce6524f7bf235e2757117d4653cbb3d71
    source_path: plugins/codex-harness.md
    workflow: 15
---

# Codex Harness

Вбудований плагін `codex` дає OpenClaw змогу запускати ходи вбудованого агента через Codex app-server замість вбудованого harness PI.

Використовуйте це, коли хочете, щоб Codex керував низькорівневою сесією агента: виявленням моделей, нативним відновленням thread, нативною компакцією та виконанням app-server.
OpenClaw, як і раніше, керує каналами чату, файлами сесій, вибором моделей, інструментами, погодженнями, доставкою медіа та видимим дзеркалом транскрипту.

Harness вимкнений за замовчуванням. Він вибирається лише тоді, коли плагін `codex` увімкнено і визначена модель є моделлю `codex/*`, або коли ви явно примусово задаєте `embeddedHarness.runtime: "codex"` чи `OPENCLAW_AGENT_RUNTIME=codex`.
Якщо ви ніколи не налаштовуєте `codex/*`, наявні запуски PI, OpenAI, Anthropic, Gemini, local і custom-provider зберігають свою поточну поведінку.

## Виберіть правильний префікс моделі

OpenClaw має окремі маршрути для доступу у форматі OpenAI та Codex:

| Model ref              | Шлях runtime                                | Використовуйте, коли                                                     |
| ---------------------- | ------------------------------------------- | ------------------------------------------------------------------------ |
| `openai/gpt-5.4`       | Провайдер OpenAI через інфраструктуру OpenClaw/PI | Ви хочете прямий доступ до OpenAI Platform API з `OPENAI_API_KEY`.       |
| `openai-codex/gpt-5.4` | Провайдер OpenAI Codex OAuth через PI       | Ви хочете ChatGPT/Codex OAuth без Codex app-server harness.              |
| `codex/gpt-5.4`        | Вбудований провайдер Codex плюс Codex harness | Ви хочете нативне виконання Codex app-server для ходу вбудованого агента. |

Codex harness обробляє лише посилання на моделі `codex/*`. Наявні посилання `openai/*`,
`openai-codex/*`, Anthropic, Gemini, xAI, local і custom provider зберігають
свої звичайні шляхи.

## Вимоги

- OpenClaw із доступним вбудованим плагіном `codex`.
- Codex app-server `0.118.0` або новіший.
- Автентифікація Codex, доступна для процесу app-server.

Плагін блокує старіші або неверсійовані handshakes app-server. Це гарантує, що
OpenClaw працює з поверхнею протоколу, з якою його тестували.

Для live- і Docker smoke-тестів автентифікація зазвичай надходить із `OPENAI_API_KEY`, а також
необов’язкових файлів Codex CLI, таких як `~/.codex/auth.json` і
`~/.codex/config.toml`. Використовуйте ті самі автентифікаційні матеріали, що й ваш локальний Codex app-server.

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

Встановлення `agents.defaults.model` або моделі агента в `codex/<model>` також
автоматично вмикає вбудований плагін `codex`. Явний запис плагіна все одно
корисний у спільних конфігураціях, оскільки робить намір розгортання очевидним.

## Додати Codex без заміни інших моделей

Залишайте `runtime: "auto"`, якщо хочете використовувати Codex для моделей `codex/*`, а PI — для
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

За такої конфігурації:

- `/model codex` або `/model codex/gpt-5.4` використовує Codex app-server harness.
- `/model gpt` або `/model openai/gpt-5.4` використовує шлях провайдера OpenAI.
- `/model opus` використовує шлях провайдера Anthropic.
- Якщо вибрано не-Codex модель, PI залишається harness сумісності.

## Розгортання лише з Codex

Вимкніть резервний перехід на PI, якщо вам потрібно підтвердити, що кожен хід вбудованого агента використовує
Codex harness:

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

Якщо резервний перехід вимкнено, OpenClaw завершується з помилкою на ранньому етапі, якщо плагін Codex вимкнено,
потрібна модель не є посиланням `codex/*`, app-server надто старий або
app-server не вдається запустити.

## Codex для окремого агента

Ви можете зробити одного агента Codex-only, а для агента за замовчуванням залишити звичайний
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
сесію OpenClaw, а Codex harness за потреби створює або відновлює свій sidecar app-server
thread. `/reset` очищає прив’язку сесії OpenClaw для цього thread.

## Виявлення моделей

За замовчуванням плагін Codex запитує app-server про доступні моделі. Якщо
виявлення завершується помилкою або перевищує час очікування, він використовує вбудований резервний каталог:

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

Вимкніть виявлення, якщо хочете, щоб під час запуску не виконувалась перевірка Codex і використовувався
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

За замовчуванням плагін локально запускає Codex командою:

```bash
codex app-server --listen stdio://
```

Ви можете залишити це значення за замовчуванням і налаштувати лише нативну політику Codex:

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

| Field               | Default                                  | Значення                                                                 |
| ------------------- | ---------------------------------------- | ------------------------------------------------------------------------ |
| `transport`         | `"stdio"`                                | `"stdio"` запускає Codex; `"websocket"` підключається до `url`.          |
| `command`           | `"codex"`                                | Виконуваний файл для транспорту stdio.                                   |
| `args`              | `["app-server", "--listen", "stdio://"]` | Аргументи для транспорту stdio.                                          |
| `url`               | unset                                    | URL WebSocket app-server.                                                |
| `authToken`         | unset                                    | Bearer-токен для транспорту WebSocket.                                   |
| `headers`           | `{}`                                     | Додаткові заголовки WebSocket.                                           |
| `requestTimeoutMs`  | `60000`                                  | Тайм-аут для викликів control-plane app-server.                          |
| `approvalPolicy`    | `"never"`                                | Нативна політика погодження Codex, що надсилається до start/resume/turn thread. |
| `sandbox`           | `"workspace-write"`                      | Нативний режим sandbox Codex, що надсилається до start/resume thread.    |
| `approvalsReviewer` | `"user"`                                 | Використовуйте `"guardian_subagent"`, щоб Codex guardian перевіряв нативні погодження. |
| `serviceTier`       | unset                                    | Необов’язковий рівень сервісу Codex, наприклад `"priority"`.             |

Старі змінні середовища все ще працюють як резервні значення для локального тестування, коли
відповідне поле конфігурації не задано:

- `OPENCLAW_CODEX_APP_SERVER_BIN`
- `OPENCLAW_CODEX_APP_SERVER_ARGS`
- `OPENCLAW_CODEX_APP_SERVER_APPROVAL_POLICY`
- `OPENCLAW_CODEX_APP_SERVER_SANDBOX`
- `OPENCLAW_CODEX_APP_SERVER_GUARDIAN=1`

Для відтворюваних розгортань перевага надається конфігурації.

## Поширені рецепти

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

Перевірка harness лише для Codex, з вимкненим резервним переходом на PI:

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

Погодження Codex, що перевіряються guardian:

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

Віддалений app-server з явними заголовками:

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

Перемикання моделей залишається під керуванням OpenClaw. Коли сесію OpenClaw прив’язано
до наявного Codex thread, наступний хід знову надсилає до
app-server поточну вибрану модель `codex/*`, провайдера, політику погодження, sandbox і service tier.
Перехід з `codex/gpt-5.4` на `codex/gpt-5.2` зберігає прив’язку до
thread, але просить Codex продовжити з новою вибраною моделлю.

## Команда Codex

Вбудований плагін реєструє `/codex` як авторизовану slash-команду. Вона є
універсальною і працює на будь-якому каналі, що підтримує текстові команди OpenClaw.

Поширені форми:

- `/codex status` показує в реальному часі підключення до app-server, моделі, обліковий запис, ліміти швидкості, MCP servers і skills.
- `/codex models` виводить список моделей Codex app-server у реальному часі.
- `/codex threads [filter]` виводить список нещодавніх thread Codex.
- `/codex resume <thread-id>` прив’язує поточну сесію OpenClaw до наявного thread Codex.
- `/codex compact` просить Codex app-server виконати компакцію прив’язаного thread.
- `/codex review` запускає нативну перевірку Codex для прив’язаного thread.
- `/codex account` показує стан облікового запису та лімітів швидкості.
- `/codex mcp` показує стан MCP server Codex app-server.
- `/codex skills` виводить список skills Codex app-server.

`/codex resume` записує той самий sidecar-файл прив’язки, який harness використовує для
звичайних ходів. У наступному повідомленні OpenClaw відновлює цей thread Codex, передає
поточну вибрану в OpenClaw модель `codex/*` до app-server і зберігає
увімкнену розширену історію.

Поверхня команд вимагає Codex app-server `0.118.0` або новішої версії. Для окремих
методів керування повідомляється `unsupported by this Codex app-server`, якщо
майбутній або кастомний app-server не надає цей метод JSON-RPC.

## Інструменти, медіа та компакція

Codex harness змінює лише низькорівневий виконавець вбудованого агента.

OpenClaw, як і раніше, формує список інструментів і отримує динамічні результати інструментів від
harness. Текст, зображення, відео, музика, TTS, погодження та вивід інструментів обміну повідомленнями
і далі проходять через звичайний шлях доставки OpenClaw.

Коли вибрана модель використовує Codex harness, нативна компакція thread делегується
Codex app-server. OpenClaw зберігає дзеркало транскрипту для історії каналу,
пошуку, `/new`, `/reset` і майбутнього перемикання моделі або harness.

Генерація медіа не потребує PI. Генерація зображень, відео, музики, PDF, TTS і
розуміння медіа продовжують використовувати відповідні налаштування провайдера/моделі, такі як
`agents.defaults.imageGenerationModel`, `videoGenerationModel`, `pdfModel` і
`messages.tts`.

## Усунення неполадок

**Codex не з’являється в `/model`:** увімкніть `plugins.entries.codex.enabled`,
задайте посилання на модель `codex/*` або перевірте, чи `plugins.allow` не виключає `codex`.

**OpenClaw переходить на PI:** задайте `embeddedHarness.fallback: "none"` або
`OPENCLAW_AGENT_HARNESS_FALLBACK=none` під час тестування.

**app-server відхиляється:** оновіть Codex, щоб handshake app-server
повідомляв версію `0.118.0` або новішу.

**Виявлення моделей повільне:** зменште `plugins.entries.codex.config.discovery.timeoutMs`
або вимкніть виявлення.

**Транспорт WebSocket відразу завершується помилкою:** перевірте `appServer.url`, `authToken`
і те, що віддалений app-server використовує ту саму версію протоколу Codex app-server.

**Не-Codex модель використовує PI:** це очікувана поведінка. Codex harness обробляє лише
посилання на моделі `codex/*`.

## Пов’язане

- [Плагіни Agent Harness](/uk/plugins/sdk-agent-harness)
- [Провайдери моделей](/uk/concepts/model-providers)
- [Довідник із конфігурації](/uk/gateway/configuration-reference)
- [Тестування](/uk/help/testing#live-codex-app-server-harness-smoke)
