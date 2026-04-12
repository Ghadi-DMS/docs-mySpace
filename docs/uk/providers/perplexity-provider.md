---
read_when:
    - Ви хочете налаштувати Perplexity як вебпровайдера пошуку
    - Вам потрібен ключ API Perplexity або налаштування проксі OpenRouter
summary: Налаштування вебпровайдера пошуку Perplexity (ключ API, режими пошуку, фільтрація)
title: Perplexity (Provider)
x-i18n:
    generated_at: "2026-04-12T10:36:42Z"
    model: gpt-5.4
    provider: openai
    source_hash: 9a6cfa63541e875869db26e407c9433442b954933a5658f443be206a496aac83
    source_path: providers/perplexity-provider.md
    workflow: 15
---

# Perplexity (вебпровайдер пошуку)

Плагін Perplexity надає можливості вебпошуку через API пошуку Perplexity
або Perplexity Sonar через OpenRouter.

<Note>
Ця сторінка описує налаштування **провайдера** Perplexity. Для **інструмента**
Perplexity (як агент його використовує) див. [інструмент Perplexity](/uk/tools/perplexity-search).
</Note>

| Властивість | Значення                                                               |
| ----------- | ---------------------------------------------------------------------- |
| Тип         | Вебпровайдер пошуку (не провайдер моделей)                             |
| Автентифікація | `PERPLEXITY_API_KEY` (напряму) або `OPENROUTER_API_KEY` (через OpenRouter) |
| Шлях конфігурації | `plugins.entries.perplexity.config.webSearch.apiKey`                   |

## Початок роботи

<Steps>
  <Step title="Установіть ключ API">
    Запустіть інтерактивний процес налаштування вебпошуку:

    ```bash
    openclaw configure --section web
    ```

    Або встановіть ключ безпосередньо:

    ```bash
    openclaw config set plugins.entries.perplexity.config.webSearch.apiKey "pplx-xxxxxxxxxxxx"
    ```

  </Step>
  <Step title="Почніть шукати">
    Агент автоматично використовуватиме Perplexity для вебпошуку, щойно ключ
    буде налаштовано. Додаткових кроків не потрібно.
  </Step>
</Steps>

## Режими пошуку

Плагін автоматично вибирає транспорт на основі префікса ключа API:

<Tabs>
  <Tab title="Власний API Perplexity (pplx-)">
    Коли ваш ключ починається з `pplx-`, OpenClaw використовує власний API пошуку
    Perplexity. Цей транспорт повертає структуровані результати та підтримує
    фільтри домену, мови й дати (див. параметри фільтрації нижче).
  </Tab>
  <Tab title="OpenRouter / Sonar (sk-or-)">
    Коли ваш ключ починається з `sk-or-`, OpenClaw спрямовує запити через
    OpenRouter, використовуючи модель Perplexity Sonar. Цей транспорт повертає
    синтезовані ШІ відповіді з посиланнями на джерела.
  </Tab>
</Tabs>

| Префікс ключа | Транспорт                   | Можливості                                      |
| ------------- | --------------------------- | ----------------------------------------------- |
| `pplx-`       | Власний API пошуку Perplexity | Структуровані результати, фільтри домену/мови/дати |
| `sk-or-`      | OpenRouter (Sonar)          | Синтезовані ШІ відповіді з посиланнями на джерела |

## Фільтрація у власному API

<Note>
Параметри фільтрації доступні лише під час використання власного API Perplexity
(ключ `pplx-`). Пошук через OpenRouter/Sonar не підтримує ці параметри.
</Note>

Під час використання власного API Perplexity пошук підтримує такі фільтри:

| Фільтр         | Опис                                      | Приклад                             |
| -------------- | ----------------------------------------- | ----------------------------------- |
| Країна         | 2-літерний код країни                     | `us`, `de`, `jp`                    |
| Мова           | Код мови ISO 639-1                        | `en`, `fr`, `zh`                    |
| Діапазон дат   | Вікно актуальності                        | `day`, `week`, `month`, `year`      |
| Фільтри доменів | Список дозволених або заборонених доменів (макс. 20 доменів) | `example.com`                       |
| Бюджет вмісту  | Обмеження токенів на відповідь / на сторінку | `max_tokens`, `max_tokens_per_page` |

## Додаткові примітки

<AccordionGroup>
  <Accordion title="Змінна середовища для фонових процесів">
    Якщо Gateway OpenClaw працює як фоновий процес (launchd/systemd), переконайтеся,
    що `PERPLEXITY_API_KEY` доступний для цього процесу.

    <Warning>
    Ключ, заданий лише в `~/.profile`, не буде видимий для демона launchd/systemd,
    якщо це середовище не було явно імпортовано. Установіть ключ у
    `~/.openclaw/.env` або через `env.shellEnv`, щоб процес gateway міг його
    прочитати.
    </Warning>

  </Accordion>

  <Accordion title="Налаштування проксі OpenRouter">
    Якщо ви хочете спрямовувати пошук Perplexity через OpenRouter, установіть
    `OPENROUTER_API_KEY` (префікс `sk-or-`) замість власного ключа Perplexity.
    OpenClaw виявить префікс і автоматично переключиться на транспорт Sonar.

    <Tip>
    Транспорт OpenRouter корисний, якщо у вас уже є обліковий запис OpenRouter
    і ви хочете консолідоване виставлення рахунків для кількох провайдерів.
    </Tip>

  </Accordion>
</AccordionGroup>

## Пов’язані матеріали

<CardGroup cols={2}>
  <Card title="Інструмент пошуку Perplexity" href="/uk/tools/perplexity-search" icon="magnifying-glass">
    Як агент викликає пошук Perplexity та інтерпретує результати.
  </Card>
  <Card title="Довідник із конфігурації" href="/uk/gateway/configuration-reference" icon="gear">
    Повний довідник із конфігурації, включно із записами плагінів.
  </Card>
</CardGroup>
