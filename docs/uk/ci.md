---
read_when:
    - Вам потрібно зрозуміти, чому завдання CI запустилося або не запустилося
    - Ви налагоджуєте збої перевірок GitHub Actions
summary: Граф CI-завдань, межі охоплення та локальні еквіваленти команд
title: CI-конвеєр
x-i18n:
    generated_at: "2026-04-09T02:58:21Z"
    model: gpt-5.4
    provider: openai
    source_hash: d104f2510fadd674d7952aa08ad73e10f685afebea8d7f19adc1d428e2bdc908
    source_path: ci.md
    workflow: 15
---

# CI-конвеєр

CI запускається для кожного push до `main` і для кожного pull request. Він використовує розумне визначення охоплення, щоб пропускати дорогі завдання, коли змінилися лише непов’язані ділянки.

## Огляд завдань

| Завдання                 | Призначення                                                                              | Коли запускається                   |
| ------------------------ | ---------------------------------------------------------------------------------------- | ----------------------------------- |
| `preflight`              | Виявлення змін лише в документації, змінених областей, змінених розширень і побудова маніфесту CI | Завжди для нечернеткових push і PR  |
| `security-fast`          | Виявлення приватних ключів, аудит workflow через `zizmor`, аудит production-залежностей  | Завжди для нечернеткових push і PR  |
| `build-artifacts`        | Один раз зібрати `dist/` і Control UI, завантажити повторно використовувані артефакти для наступних завдань | Зміни, релевантні для Node          |
| `checks-fast-core`       | Швидкі Linux-лінії перевірки коректності, як-от bundled/plugin-contract/protocol checks  | Зміни, релевантні для Node          |
| `checks-fast-extensions` | Агрегувати шардовані лінії розширень після завершення `checks-fast-extensions-shard`     | Зміни, релевантні для Node          |
| `extension-fast`         | Цільові тести лише для змінених bundled plugins                                          | Коли виявлено зміни розширень       |
| `check`                  | Основний локальний gate у CI: `pnpm check` плюс `pnpm build:strict-smoke`                | Зміни, релевантні для Node          |
| `check-additional`       | Архітектурні перевірки, перевірки меж, guardrails для циклів імпорту та regression harness для gateway watch | Зміни, релевантні для Node          |
| `build-smoke`            | Smoke-тести зібраного CLI та smoke-перевірка пам’яті під час запуску                     | Зміни, релевантні для Node          |
| `checks`                 | Важчі Linux Node-лінії: повні тести, тести каналів і сумісність із Node 22 лише для push | Зміни, релевантні для Node          |
| `check-docs`             | Форматування документації, lint і перевірка битих посилань                               | Змінено документацію                |
| `skills-python`          | Ruff + pytest для Skills на базі Python                                                  | Зміни, релевантні для Python Skills |
| `checks-windows`         | Специфічні для Windows лінії тестування                                                  | Зміни, релевантні для Windows       |
| `macos-node`             | Лінія TypeScript-тестів на macOS зі спільними зібраними артефактами                      | Зміни, релевантні для macOS         |
| `macos-swift`            | Swift lint, збірка та тести для застосунку macOS                                         | Зміни, релевантні для macOS         |
| `android`                | Матриця збірки та тестів Android                                                         | Зміни, релевантні для Android       |

## Порядок Fail-Fast

Завдання впорядковані так, щоб дешеві перевірки завершувалися з помилкою раніше, ніж запускатимуться дорогі:

1. `preflight` визначає, які лінії взагалі існують. Логіка `docs-scope` і `changed-scope` — це кроки всередині цього завдання, а не окремі завдання.
2. `security-fast`, `check`, `check-additional`, `check-docs` і `skills-python` завершуються з помилкою швидко, не чекаючи важчих завдань із артефактами та платформними матрицями.
3. `build-artifacts` виконується паралельно зі швидкими Linux-лініями, щоб наступні споживачі могли стартувати, щойно спільна збірка буде готова.
4. Далі розгалужуються важчі платформні та runtime-лінії: `checks-fast-core`, `checks-fast-extensions`, `extension-fast`, `checks`, `checks-windows`, `macos-node`, `macos-swift` і `android`.

Логіка охоплення знаходиться в `scripts/ci-changed-scope.mjs` і покрита unit-тестами в `src/scripts/ci-changed-scope.test.ts`.
Окремий workflow `install-smoke` повторно використовує той самий скрипт охоплення через власне завдання `preflight`. Він обчислює `run_install_smoke` на основі вужчого сигналу changed-smoke, тому Docker/install smoke запускається лише для змін, релевантних для встановлення, пакування та контейнерів.

Для push матриця `checks` додає лінію `compat-node22`, яка виконується лише для push. Для pull request ця лінія пропускається, і матриця залишається зосередженою на звичайних тестових лініях і лініях каналів.

## Виконавці

| Виконавець                       | Завдання                                                                                             |
| -------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `blacksmith-16vcpu-ubuntu-2404`  | `preflight`, `security-fast`, `build-artifacts`, Linux-перевірки, перевірки документації, Python Skills, `android` |
| `blacksmith-32vcpu-windows-2025` | `checks-windows`                                                                                     |
| `macos-latest`                   | `macos-node`, `macos-swift`                                                                          |

## Локальні еквіваленти

```bash
pnpm check          # типи + lint + форматування
pnpm build:strict-smoke
pnpm check:import-cycles
pnpm test:gateway:watch-regression
pnpm test           # тести vitest
pnpm test:channels
pnpm check:docs     # форматування документації + lint + биті посилання
pnpm build          # зібрати dist, коли важливі лінії CI artifact/build-smoke
```
