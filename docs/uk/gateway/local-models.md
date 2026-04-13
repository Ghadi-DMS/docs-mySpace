---
read_when:
    - Ви хочете надавати моделі з власного GPU-сервера
    - Ви налаштовуєте LM Studio або OpenAI-сумісний проксі-сервер
    - Вам потрібні найбезпечніші рекомендації щодо локальних моделей
summary: Запустіть OpenClaw на локальних LLM (LM Studio, vLLM, LiteLLM, власні кінцеві точки OpenAI)
title: Локальні моделі
x-i18n:
    generated_at: "2026-04-13T07:24:25Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3ecb61b3e6e34d3666f9b688cd694d92c5fb211cf8c420fa876f7ccf5789154a
    source_path: gateway/local-models.md
    workflow: 15
---

# Локальні моделі

Локальний запуск можливий, але OpenClaw очікує великий контекст + сильний захист від інʼєкцій у підказки. Малі відеокарти обрізають контекст і послаблюють безпеку. Орієнтуйтеся на високий рівень: **≥2 Mac Studio з максимальною конфігурацією або еквівалентну GPU-систему (~$30k+)**. Одна **24 GB** GPU підходить лише для простіших запитів із вищою затримкою. Використовуйте **найбільший / повнорозмірний варіант моделі, який можете запустити**; агресивно квантизовані або «малі» чекпойнти підвищують ризик інʼєкцій у підказки (див. [Безпека](/uk/gateway/security)).

Якщо вам потрібне локальне налаштування з мінімальними труднощами, почніть із [LM Studio](/uk/providers/lmstudio) або [Ollama](/uk/providers/ollama) і `openclaw onboard`. Ця сторінка — це практичний посібник для потужніших локальних стеків і власних локальних серверів, сумісних з OpenAI.

## Рекомендовано: LM Studio + велика локальна модель (Responses API)

Найкращий поточний локальний стек. Завантажте велику модель у LM Studio (наприклад, повнорозмірну збірку Qwen, DeepSeek або Llama), увімкніть локальний сервер (типово `http://127.0.0.1:1234`), і використовуйте Responses API, щоб відокремити міркування від фінального тексту.

```json5
{
  agents: {
    defaults: {
      model: { primary: “lmstudio/my-local-model” },
      models: {
        “anthropic/claude-opus-4-6”: { alias: “Opus” },
        “lmstudio/my-local-model”: { alias: “Local” },
      },
    },
  },
  models: {
    mode: “merge”,
    providers: {
      lmstudio: {
        baseUrl: “http://127.0.0.1:1234/v1”,
        apiKey: “lmstudio”,
        api: “openai-responses”,
        models: [
          {
            id: “my-local-model”,
            name: “Local Model”,
            reasoning: false,
            input: [“text”],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 196608,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

**Контрольний список налаштування**

- Встановіть LM Studio: [https://lmstudio.ai](https://lmstudio.ai)
- У LM Studio завантажте **найбільшу доступну збірку моделі** (уникайте «малих»/сильно квантизованих варіантів), запустіть сервер, переконайтеся, що `http://127.0.0.1:1234/v1/models` її показує.
- Замініть `my-local-model` на фактичний ідентифікатор моделі, показаний у LM Studio.
- Тримайте модель завантаженою; холодне завантаження додає затримку під час запуску.
- За потреби скоригуйте `contextWindow`/`maxTokens`, якщо ваша збірка LM Studio відрізняється.
- Для WhatsApp дотримуйтеся Responses API, щоб надсилався лише фінальний текст.

Залишайте хмарні моделі налаштованими навіть під час локального запуску; використовуйте `models.mode: "merge"`, щоб резервні варіанти лишалися доступними.

### Гібридна конфігурація: основна хмарна модель, локальний резерв

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-sonnet-4-6",
        fallbacks: ["lmstudio/my-local-model", "anthropic/claude-opus-4-6"],
      },
      models: {
        "anthropic/claude-sonnet-4-6": { alias: "Sonnet" },
        "lmstudio/my-local-model": { alias: "Local" },
        "anthropic/claude-opus-4-6": { alias: "Opus" },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      lmstudio: {
        baseUrl: "http://127.0.0.1:1234/v1",
        apiKey: "lmstudio",
        api: "openai-responses",
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 196608,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

### Спочатку локально, із хмарною страховкою

Поміняйте місцями основну модель і порядок резервних; збережіть той самий блок providers і `models.mode: "merge"`, щоб мати змогу переключитися на Sonnet або Opus, коли локальний сервер недоступний.

### Регіональний хостинг / маршрутизація даних

- Хмарні варіанти MiniMax/Kimi/GLM також доступні в OpenRouter з кінцевими точками, привʼязаними до регіону (наприклад, розміщеними у США). Виберіть там регіональний варіант, щоб трафік залишався в обраній вами юрисдикції, і водночас використовуйте `models.mode: "merge"` для резервних Anthropic/OpenAI.
- Лише локальний запуск залишається найкращим шляхом для приватності; регіональна хмарна маршрутизація — це компромісний варіант, якщо вам потрібні можливості провайдера, але ви хочете контролювати потік даних.

## Інші локальні проксі-сервери, сумісні з OpenAI

vLLM, LiteLLM, OAI-proxy або власні шлюзи працюють, якщо вони надають кінцеву точку `/v1` у стилі OpenAI. Замініть блок provider вище на вашу кінцеву точку та ідентифікатор моделі:

```json5
{
  models: {
    mode: "merge",
    providers: {
      local: {
        baseUrl: "http://127.0.0.1:8000/v1",
        apiKey: "sk-local",
        api: "openai-responses",
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 120000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

Залишайте `models.mode: "merge"`, щоб хмарні моделі були доступні як резервні.

Примітка щодо поведінки для локальних/проксійованих бекендів `/v1`:

- OpenClaw обробляє їх як проксі-маршрути, сумісні з OpenAI, а не як нативні кінцеві точки OpenAI
- нативне формування запитів лише для OpenAI тут не застосовується: немає
  `service_tier`, немає Responses `store`, немає формування payload сумісності міркувань OpenAI
  і немає підказок для кешу підказок
- приховані заголовки атрибуції OpenClaw (`originator`, `version`, `User-Agent`)
  не додаються до цих власних проксі-URL

Примітки щодо сумісності для суворіших бекендів, сумісних з OpenAI:

- Деякі сервери приймають лише рядковий `messages[].content` у Chat Completions, а не
  структуровані масиви частин контенту. Для таких кінцевих точок установіть
  `models.providers.<provider>.models[].compat.requiresStringContent: true`.
- Деякі менші або суворіші локальні бекенди нестабільно працюють із повною
  формою підказки середовища агента OpenClaw, особливо коли включено схеми інструментів. Якщо
  бекенд працює для маленьких прямих викликів `/v1/chat/completions`, але не працює під час звичайних
  ходів агента OpenClaw, спочатку спробуйте
  `models.providers.<provider>.models[].compat.supportsTools: false`.
- Якщо бекенд і далі не працює лише під час більших запусків OpenClaw, то
  проблема зазвичай полягає у можливостях моделі/сервера вище за рівнем або в помилці бекенда, а не в транспортному
  шарі OpenClaw.

## Усунення несправностей

- Gateway може дістатися до проксі? `curl http://127.0.0.1:1234/v1/models`.
- Модель LM Studio вивантажена? Завантажте її знову; холодний старт — поширена причина «зависання».
- Помилки контексту? Зменште `contextWindow` або підвищте ліміт на вашому сервері.
- OpenAI-сумісний сервер повертає `messages[].content ... expected a string`?
  Додайте `compat.requiresStringContent: true` до запису цієї моделі.
- Прямі маленькі виклики `/v1/chat/completions` працюють, але `openclaw infer model run`
  не працює на Gemma або іншій локальній моделі? Спочатку вимкніть схеми інструментів через
  `compat.supportsTools: false`, а потім повторіть перевірку. Якщо сервер і далі аварійно завершує роботу лише
  на більших підказках OpenClaw, вважайте це обмеженням моделі/сервера вище за рівнем.
- Безпека: локальні моделі обходять фільтри на боці провайдера; звужуйте область дії агентів і тримайте увімкненим Compaction, щоб обмежити радіус ураження від інʼєкцій у підказки.
