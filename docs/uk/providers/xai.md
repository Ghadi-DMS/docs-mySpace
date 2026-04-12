---
read_when:
    - Ви хочете використовувати моделі Grok в OpenClaw
    - Ви налаштовуєте автентифікацію xAI або ідентифікатори моделей
summary: Використовуйте моделі xAI Grok в OpenClaw
title: xAI
x-i18n:
    generated_at: "2026-04-12T10:19:22Z"
    model: gpt-5.4
    provider: openai
    source_hash: 064aa758f9791f704dc6b0cc0059bb73b7de68ae2edd731bb87dbc26730eea9f
    source_path: providers/xai.md
    workflow: 15
---

# xAI

OpenClaw постачається з вбудованим Plugin провайдера `xai` для моделей Grok.

## Початок роботи

<Steps>
  <Step title="Створіть API-ключ">
    Створіть API-ключ у [консолі xAI](https://console.x.ai/).
  </Step>
  <Step title="Установіть свій API-ключ">
    Установіть `XAI_API_KEY` або виконайте:

    ```bash
    openclaw onboard --auth-choice xai-api-key
    ```

  </Step>
  <Step title="Виберіть модель">
    ```json5
    {
      agents: { defaults: { model: { primary: "xai/grok-4" } } },
    }
    ```
  </Step>
</Steps>

<Note>
OpenClaw використовує xAI Responses API як вбудований транспорт xAI. Той самий
`XAI_API_KEY` також може забезпечувати роботу `web_search` на базі Grok, вбудованого `x_search`
та віддаленого `code_execution`.
Якщо ви зберігаєте ключ xAI у `plugins.entries.xai.config.webSearch.apiKey`,
вбудований провайдер моделей xAI також повторно використовує цей ключ як резервний варіант.
Налаштування `code_execution` розміщуються в `plugins.entries.xai.config.codeExecution`.
</Note>

## Каталог вбудованих моделей

OpenClaw містить такі сімейства моделей xAI з коробки:

| Сімейство     | Ідентифікатори моделей                                                     |
| ------------- | -------------------------------------------------------------------------- |
| Grok 3        | `grok-3`, `grok-3-fast`, `grok-3-mini`, `grok-3-mini-fast`                 |
| Grok 4        | `grok-4`, `grok-4-0709`                                                    |
| Grok 4 Fast   | `grok-4-fast`, `grok-4-fast-non-reasoning`                                 |
| Grok 4.1 Fast | `grok-4-1-fast`, `grok-4-1-fast-non-reasoning`                             |
| Grok 4.20 Beta | `grok-4.20-beta-latest-reasoning`, `grok-4.20-beta-latest-non-reasoning` |
| Grok Code     | `grok-code-fast-1`                                                         |

Plugin також перенаправлено розпізнає новіші ідентифікатори `grok-4*` і `grok-code-fast*`, коли
вони відповідають тій самій формі API.

<Tip>
`grok-4-fast`, `grok-4-1-fast` і варіанти `grok-4.20-beta-*` —
це поточні посилання Grok з підтримкою зображень у вбудованому каталозі.
</Tip>

### Відображення Fast-mode

`/fast on` або `agents.defaults.models["xai/<model>"].params.fastMode: true`
перезаписує нативні запити xAI таким чином:

| Вихідна модель | Ціль Fast-mode    |
| -------------- | ----------------- |
| `grok-3`       | `grok-3-fast`     |
| `grok-3-mini`  | `grok-3-mini-fast` |
| `grok-4`       | `grok-4-fast`     |
| `grok-4-0709`  | `grok-4-fast`     |

### Застарілі аліаси сумісності

Застарілі аліаси й далі нормалізуються до канонічних вбудованих ідентифікаторів:

| Застарілий аліас         | Канонічний ідентифікатор            |
| ------------------------ | ----------------------------------- |
| `grok-4-fast-reasoning`   | `grok-4-fast`                       |
| `grok-4-1-fast-reasoning` | `grok-4-1-fast`                     |
| `grok-4.20-reasoning`     | `grok-4.20-beta-latest-reasoning`   |
| `grok-4.20-non-reasoning` | `grok-4.20-beta-latest-non-reasoning` |

## Можливості

<AccordionGroup>
  <Accordion title="Веб-пошук">
    Вбудований провайдер веб-пошуку `grok` також використовує `XAI_API_KEY`:

    ```bash
    openclaw config set tools.web.search.provider grok
    ```

  </Accordion>

  <Accordion title="Генерація відео">
    Вбудований Plugin `xai` реєструє генерацію відео через спільний
    інструмент `video_generate`.

    - Модель відео за замовчуванням: `xai/grok-imagine-video`
    - Режими: text-to-video, image-to-video та віддалені потоки редагування/розширення відео
    - Підтримує `aspectRatio` і `resolution`

    <Warning>
    Локальні буфери відео не приймаються. Використовуйте віддалені URL `http(s)` для
    посилань на відео та вхідних даних редагування.
    </Warning>

    Щоб використовувати xAI як провайдера відео за замовчуванням:

    ```json5
    {
      agents: {
        defaults: {
          videoGenerationModel: {
            primary: "xai/grok-imagine-video",
          },
        },
      },
    }
    ```

    <Note>
    Див. [Генерація відео](/uk/tools/video-generation), щоб дізнатися про спільні параметри інструмента,
    вибір провайдера та поведінку перемикання на резервний варіант.
    </Note>

  </Accordion>

  <Accordion title="Відомі обмеження">
    - Наразі автентифікація підтримується лише через API-ключ. У
      OpenClaw поки немає потоку xAI OAuth або device-code.
    - `grok-4.20-multi-agent-experimental-beta-0304` не підтримується у
      звичайному шляху провайдера xAI, оскільки він вимагає іншої поверхні
      upstream API, ніж стандартний транспорт xAI в OpenClaw.
  </Accordion>

  <Accordion title="Додаткові примітки">
    - OpenClaw автоматично застосовує виправлення сумісності, специфічні для xAI, для схем інструментів і викликів інструментів
      у спільному шляху виконання.
    - Для нативних запитів xAI за замовчуванням використовується `tool_stream: true`. Установіть
      `agents.defaults.models["xai/<model>"].params.tool_stream` у `false`, щоб
      вимкнути це.
    - Вбудована обгортка xAI вилучає непідтримувані строгі прапорці схем інструментів і
      ключі payload reasoning перед надсиланням нативних запитів xAI.
    - `web_search`, `x_search` і `code_execution` доступні як інструменти OpenClaw.
      OpenClaw вмикає конкретний вбудований інструмент xAI, який потрібен, усередині кожного запиту інструмента,
      замість того щоб прикріплювати всі нативні інструменти до кожного ходу чату.
    - `x_search` і `code_execution` належать вбудованому Plugin xAI, а не
      жорстко закодовані в основному середовищі виконання моделі.
    - `code_execution` — це віддалене виконання в пісочниці xAI, а не локальний
      [`exec`](/uk/tools/exec).
  </Accordion>
</AccordionGroup>

## Пов’язані матеріали

<CardGroup cols={2}>
  <Card title="Вибір моделі" href="/uk/concepts/model-providers" icon="layers">
    Вибір провайдерів, посилань на моделі та поведінки резервного перемикання.
  </Card>
  <Card title="Генерація відео" href="/uk/tools/video-generation" icon="video">
    Спільні параметри інструмента відео та вибір провайдера.
  </Card>
  <Card title="Усі провайдери" href="/uk/providers/index" icon="grid-2">
    Ширший огляд провайдерів.
  </Card>
  <Card title="Усунення несправностей" href="/uk/help/troubleshooting" icon="wrench">
    Поширені проблеми та способи їх вирішення.
  </Card>
</CardGroup>
