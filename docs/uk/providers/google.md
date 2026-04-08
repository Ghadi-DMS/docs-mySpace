---
read_when:
    - Ви хочете використовувати моделі Google Gemini з OpenClaw
    - Вам потрібен потік автентифікації API key або OAuth
summary: Налаштування Google Gemini (API key + OAuth, генерація зображень, розуміння медіа, вебпошук)
title: Google (Gemini)
x-i18n:
    generated_at: "2026-04-08T06:28:55Z"
    model: gpt-5.4
    provider: openai
    source_hash: fad2ff68987301bd86145fa6e10de8c7b38d5bd5dbcd13db9c883f7f5b9a4e01
    source_path: providers/google.md
    workflow: 15
---

# Google (Gemini)

Плагін Google надає доступ до моделей Gemini через Google AI Studio, а також до
генерації зображень, розуміння медіа (зображення/аудіо/відео) і вебпошуку через
Gemini Grounding.

- Провайдер: `google`
- Автентифікація: `GEMINI_API_KEY` або `GOOGLE_API_KEY`
- API: Google Gemini API
- Альтернативний провайдер: `google-gemini-cli` (OAuth)

## Швидкий старт

1. Установіть API key:

```bash
openclaw onboard --auth-choice gemini-api-key
```

2. Установіть модель за замовчуванням:

```json5
{
  agents: {
    defaults: {
      model: { primary: "google/gemini-3.1-pro-preview" },
    },
  },
}
```

## Неінтерактивний приклад

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice gemini-api-key \
  --gemini-api-key "$GEMINI_API_KEY"
```

## OAuth (Gemini CLI)

Альтернативний провайдер `google-gemini-cli` використовує PKCE OAuth замість API
key. Це неофіційна інтеграція; деякі користувачі повідомляють про обмеження
облікових записів. Використовуйте на власний ризик.

- Модель за замовчуванням: `google-gemini-cli/gemini-3-flash-preview`
- Псевдонім: `gemini-cli`
- Обов’язкова умова для встановлення: локальний Gemini CLI, доступний як `gemini`
  - Homebrew: `brew install gemini-cli`
  - npm: `npm install -g @google/gemini-cli`
- Вхід:

```bash
openclaw models auth login --provider google-gemini-cli --set-default
```

Змінні середовища:

- `OPENCLAW_GEMINI_OAUTH_CLIENT_ID`
- `OPENCLAW_GEMINI_OAUTH_CLIENT_SECRET`

(Або варіанти `GEMINI_CLI_*`.)

Якщо OAuth-запити Gemini CLI не вдаються після входу, установіть
`GOOGLE_CLOUD_PROJECT` або `GOOGLE_CLOUD_PROJECT_ID` на хості gateway і
повторіть спробу.

Якщо вхід не вдається ще до запуску потоку в браузері, переконайтеся, що
локальну команду `gemini` встановлено й вона доступна в `PATH`. OpenClaw
підтримує як встановлення через Homebrew, так і глобальні встановлення npm,
зокрема поширені схеми Windows/npm.

Нотатки щодо використання JSON у Gemini CLI:

- Текст відповіді береться з поля CLI JSON `response`.
- Показники використання беруться з резервного поля `stats`, якщо CLI залишає `usage` порожнім.
- `stats.cached` нормалізується в `cacheRead` OpenClaw.
- Якщо `stats.input` відсутній, OpenClaw обчислює вхідні токени з
  `stats.input_tokens - stats.cached`.

## Можливості

| Можливість             | Підтримується     |
| ---------------------- | ----------------- |
| Завершення чату        | Так               |
| Генерація зображень    | Так               |
| Генерація музики       | Так               |
| Розуміння зображень    | Так               |
| Транскрибування аудіо  | Так               |
| Розуміння відео        | Так               |
| Вебпошук (Grounding)   | Так               |
| Мислення/міркування    | Так (Gemini 3.1+) |
| Моделі Gemma 4         | Так               |

Моделі Gemma 4 (наприклад, `gemma-4-26b-a4b-it`) підтримують режим thinking. OpenClaw переписує `thinkingBudget` у підтримуваний Google `thinkingLevel` для Gemma 4. Установлення thinking в `off` зберігає його вимкненим замість зіставлення з `MINIMAL`.

## Пряме повторне використання кешу Gemini

Для прямих запусків Gemini API (`api: "google-generative-ai"`) OpenClaw тепер
передає налаштований дескриптор `cachedContent` у запити Gemini.

- Налаштовуйте параметри для моделі або глобально через
  `cachedContent` або застарілий `cached_content`
- Якщо присутні обидва, пріоритет має `cachedContent`
- Приклад значення: `cachedContents/prebuilt-context`
- Використання cache-hit у Gemini нормалізується в `cacheRead` OpenClaw з
  вихідного `cachedContentTokenCount`

Приклад:

```json5
{
  agents: {
    defaults: {
      models: {
        "google/gemini-2.5-pro": {
          params: {
            cachedContent: "cachedContents/prebuilt-context",
          },
        },
      },
    },
  },
}
```

## Генерація зображень

Вбудований провайдер генерації зображень `google` за замовчуванням використовує
`google/gemini-3.1-flash-image-preview`.

- Також підтримує `google/gemini-3-pro-image-preview`
- Генерація: до 4 зображень на запит
- Режим редагування: увімкнено, до 5 вхідних зображень
- Керування геометрією: `size`, `aspectRatio` і `resolution`

Провайдер `google-gemini-cli`, який працює лише через OAuth, є окремою
поверхнею для текстового inference. Генерація зображень, розуміння медіа та
Gemini Grounding залишаються на ідентифікаторі провайдера `google`.

Щоб використовувати Google як провайдера зображень за замовчуванням:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "google/gemini-3.1-flash-image-preview",
      },
    },
  },
}
```

Перегляньте [Генерація зображень](/uk/tools/image-generation), щоб дізнатися про
спільні параметри інструмента, вибір провайдера та поведінку failover.

## Генерація відео

Вбудований плагін `google` також реєструє генерацію відео через спільний
інструмент `video_generate`.

- Модель відео за замовчуванням: `google/veo-3.1-fast-generate-preview`
- Режими: text-to-video, image-to-video і потоки з посиланням на одне відео
- Підтримує `aspectRatio`, `resolution` і `audio`
- Поточне обмеження тривалості: **від 4 до 8 секунд**

Щоб використовувати Google як провайдера відео за замовчуванням:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "google/veo-3.1-fast-generate-preview",
      },
    },
  },
}
```

Перегляньте [Генерація відео](/uk/tools/video-generation), щоб дізнатися про
спільні параметри інструмента, вибір провайдера та поведінку failover.

## Генерація музики

Вбудований плагін `google` також реєструє генерацію музики через спільний
інструмент `music_generate`.

- Модель музики за замовчуванням: `google/lyria-3-clip-preview`
- Також підтримує `google/lyria-3-pro-preview`
- Керування підказкою: `lyrics` і `instrumental`
- Формат виводу: `mp3` за замовчуванням, а також `wav` у `google/lyria-3-pro-preview`
- Вхідні дані-посилання: до 10 зображень
- Запуски з підтримкою сесій від’єднуються через спільний потік завдань/статусу, зокрема `action: "status"`

Щоб використовувати Google як провайдера музики за замовчуванням:

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "google/lyria-3-clip-preview",
      },
    },
  },
}
```

Перегляньте [Генерація музики](/uk/tools/music-generation), щоб дізнатися про
спільні параметри інструмента, вибір провайдера та поведінку failover.

## Примітка щодо середовища

Якщо Gateway працює як демон (launchd/systemd), переконайтеся, що `GEMINI_API_KEY`
доступний для цього процесу (наприклад, у `~/.openclaw/.env` або через
`env.shellEnv`).
