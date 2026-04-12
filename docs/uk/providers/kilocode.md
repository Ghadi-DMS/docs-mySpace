---
read_when:
    - Вам потрібен один API-ключ для багатьох LLM
    - Ви хочете запускати моделі через Kilo Gateway в OpenClaw
summary: Використовуйте уніфікований API Kilo Gateway для доступу до багатьох моделей в OpenClaw
title: Kilo Gateway
x-i18n:
    generated_at: "2026-04-12T10:30:01Z"
    model: gpt-5.4
    provider: openai
    source_hash: bf08ee5157b44233692795294b263668a9e0003795c8c170af74ad75c88b1040
    source_path: providers/kilocode.md
    workflow: 15
---

# Kilo Gateway

Kilo Gateway надає **уніфікований API**, який маршрутизує запити до багатьох моделей через єдину
кінцеву точку та API-ключ. Він сумісний з OpenAI, тож більшість SDK OpenAI працюють, якщо змінити base URL.

| Property | Value                              |
| -------- | ---------------------------------- |
| Провайдер | `kilocode`                         |
| Автентифікація     | `KILOCODE_API_KEY`                 |
| API      | Сумісний з OpenAI                  |
| Base URL | `https://api.kilo.ai/api/gateway/` |

## Початок роботи

<Steps>
  <Step title="Створіть обліковий запис">
    Перейдіть на [app.kilo.ai](https://app.kilo.ai), увійдіть або створіть обліковий запис, потім перейдіть до API Keys і згенеруйте новий ключ.
  </Step>
  <Step title="Запустіть онбординг">
    ```bash
    openclaw onboard --auth-choice kilocode-api-key
    ```

    Або встановіть змінну середовища напряму:

    ```bash
    export KILOCODE_API_KEY="<your-kilocode-api-key>" # pragma: allowlist secret
    ```

  </Step>
  <Step title="Переконайтеся, що модель доступна">
    ```bash
    openclaw models list --provider kilocode
    ```
  </Step>
</Steps>

## Модель за замовчуванням

Моделлю за замовчуванням є `kilocode/kilo/auto`, модель
розумної маршрутизації, що належить провайдеру та керується Kilo Gateway.

<Note>
OpenClaw розглядає `kilocode/kilo/auto` як стабільне посилання за замовчуванням, але не
публікує підтверджене вихідним кодом зіставлення між завданнями та висхідними моделями для цього маршруту. Точна
висхідна маршрутизація за `kilocode/kilo/auto` визначається Kilo Gateway, а не
жорстко закодована в OpenClaw.
</Note>

## Доступні моделі

OpenClaw динамічно виявляє доступні моделі з Kilo Gateway під час запуску. Використовуйте
`/models kilocode`, щоб побачити повний список моделей, доступних для вашого облікового запису.

Будь-яку модель, доступну в gateway, можна використовувати з префіксом `kilocode/`:

| Model ref                              | Notes                              |
| -------------------------------------- | ---------------------------------- |
| `kilocode/kilo/auto`                   | За замовчуванням — розумна маршрутизація            |
| `kilocode/anthropic/claude-sonnet-4`   | Anthropic через Kilo                 |
| `kilocode/openai/gpt-5.4`              | OpenAI через Kilo                    |
| `kilocode/google/gemini-3-pro-preview` | Google через Kilo                    |
| ...and many more                       | Використовуйте `/models kilocode`, щоб переглянути всі |

<Tip>
Під час запуску OpenClaw виконує запит `GET https://api.kilo.ai/api/gateway/models` і об’єднує
виявлені моделі перед статичним резервним каталогом. Вбудований резервний каталог завжди
містить `kilocode/kilo/auto` (`Kilo Auto`) з `input: ["text", "image"]`,
`reasoning: true`, `contextWindow: 1000000` і `maxTokens: 128000`.
</Tip>

## Приклад конфігурації

```json5
{
  env: { KILOCODE_API_KEY: "<your-kilocode-api-key>" }, // pragma: allowlist secret
  agents: {
    defaults: {
      model: { primary: "kilocode/kilo/auto" },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="Транспорт і сумісність">
    Kilo Gateway задокументовано у вихідному коді як сумісний з OpenRouter, тому він залишається на
    шляху проксі-стилю, сумісного з OpenAI, а не використовує власне формування запитів OpenAI.

    - Kilo-посилання на основі Gemini залишаються на шляху проксі-Gemini, тому OpenClaw зберігає
      санітизацію thought-signature Gemini там, не вмикаючи нативну
      перевірку відтворення Gemini або переписування bootstrap.
    - Kilo Gateway під капотом використовує Bearer token з вашим API-ключем.

  </Accordion>

  <Accordion title="Обгортка потоку та reasoning">
    Спільна обгортка потоку Kilo додає заголовок застосунку провайдера та нормалізує
    проксі-повідомлення reasoning для підтримуваних конкретних посилань на моделі.

    <Warning>
    `kilocode/kilo/auto` та інші підказки, що не підтримують proxy-reasoning, пропускають ін’єкцію reasoning. Якщо вам потрібна підтримка reasoning, використовуйте конкретне посилання на модель, наприклад
    `kilocode/anthropic/claude-sonnet-4`.
    </Warning>

  </Accordion>

  <Accordion title="Усунення несправностей">
    - Якщо виявлення моделей не вдається під час запуску, OpenClaw повертається до вбудованого статичного каталогу, що містить `kilocode/kilo/auto`.
    - Переконайтеся, що ваш API-ключ дійсний і що у вашому обліковому записі Kilo увімкнено потрібні моделі.
    - Коли Gateway працює як демон, переконайтеся, що `KILOCODE_API_KEY` доступний цьому процесу (наприклад, у `~/.openclaw/.env` або через `env.shellEnv`).
  </Accordion>
</AccordionGroup>

## Пов’язане

<CardGroup cols={2}>
  <Card title="Вибір моделі" href="/uk/concepts/model-providers" icon="layers">
    Вибір провайдерів, посилань на моделі та поведінки перемикання при відмові.
  </Card>
  <Card title="Довідник з конфігурації" href="/uk/gateway/configuration" icon="gear">
    Повний довідник з конфігурації OpenClaw.
  </Card>
  <Card title="Kilo Gateway" href="https://app.kilo.ai" icon="arrow-up-right-from-square">
    Панель керування Kilo Gateway, API-ключі та керування обліковим записом.
  </Card>
</CardGroup>
