---
read_when:
    - Шукаю визначення публічних каналів релізів
    - Шукаю назви версій і періодичність
summary: Публічні канали релізів, назви версій і періодичність
title: Політика релізів
x-i18n:
    generated_at: "2026-04-12T22:00:22Z"
    model: gpt-5.4
    provider: openai
    source_hash: dffc1ee5fdbb20bd1bf4b3f817d497fc0d87f70ed6c669d324fea66dc01d0b0b
    source_path: reference/RELEASING.md
    workflow: 15
---

# Політика релізів

OpenClaw має три публічні канали релізів:

- stable: релізи з тегами, які за замовчуванням публікуються в npm `beta`, або в npm `latest`, якщо це явно запитано
- beta: prerelease-теги, які публікуються в npm `beta`
- dev: поточна рухома вершина `main`

## Назви версій

- Версія stable-релізу: `YYYY.M.D`
  - Git-тег: `vYYYY.M.D`
- Версія stable-виправлення: `YYYY.M.D-N`
  - Git-тег: `vYYYY.M.D-N`
- Версія beta prerelease: `YYYY.M.D-beta.N`
  - Git-тег: `vYYYY.M.D-beta.N`
- Не додавайте провідні нулі до місяця або дня
- `latest` означає поточний підвищений stable-реліз у npm
- `beta` означає поточну ціль встановлення beta
- Stable-релізи та stable-виправлення за замовчуванням публікуються в npm `beta`; оператори релізу можуть явно націлити публікацію на `latest` або пізніше підвищити перевірену beta-збірку
- Кожен реліз OpenClaw постачається разом із npm-пакетом і застосунком для macOS

## Періодичність релізів

- Релізи спочатку проходять через beta
- Stable виходить лише після перевірки останньої beta
- Детальна процедура релізу, затвердження, облікові дані та примітки щодо відновлення
  доступні лише для maintainer-ів

## Передрелізна перевірка

- Запустіть `pnpm build && pnpm ui:build` перед `pnpm release:check`, щоб очікувані
  артефакти релізу `dist/*` і бандл Control UI існували для кроку
  перевірки pack
- Запускайте `pnpm release:check` перед кожним релізом із тегом
- Перевірки релізу тепер виконуються в окремому ручному workflow:
  `OpenClaw Release Checks`
- Цей поділ є навмисним: він зберігає реальний шлях npm-релізу коротким,
  детермінованим і зосередженим на артефактах, тоді як повільніші live-перевірки залишаються
  у власному каналі, щоб не затримувати і не блокувати публікацію
- Перевірки релізу потрібно запускати з workflow ref для `main`, щоб
  логіка workflow і секрети залишалися канонічними
- Цей workflow приймає або наявний тег релізу, або поточний повний
  40-символьний SHA коміту `main`
- У режимі SHA коміту він приймає лише поточний HEAD `origin/main`; для
  старіших комітів релізу використовуйте тег релізу
- Передрелізна перевірка лише для валідації `OpenClaw NPM Release` також приймає поточний
  повний 40-символьний SHA коміту `main` без вимоги наявності запушеного тега
- Цей шлях SHA є лише для валідації і не може бути підвищений до реальної публікації
- У режимі SHA workflow синтезує `v<package.json version>` лише для
  перевірки метаданих пакета; реальна публікація все одно потребує справжнього тега релізу
- Обидва workflows залишають реальний шлях публікації та підвищення на GitHub-hosted
  runners, тоді як шлях валідації без змін стану може використовувати більші
  Blacksmith Linux runners
- Цей workflow запускає
  `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_CACHE_TEST=1 pnpm test:live:cache`
  використовуючи обидва workflow secrets: `OPENAI_API_KEY` і `ANTHROPIC_API_KEY`
- Передрелізна перевірка npm-релізу більше не чекає окремий канал перевірок релізу
- Перед затвердженням запустіть `RELEASE_TAG=vYYYY.M.D node --import tsx scripts/openclaw-npm-release-check.ts`
  (або відповідний тег beta/виправлення)
- Після публікації в npm запустіть
  `node --import tsx scripts/openclaw-npm-postpublish-verify.ts YYYY.M.D`
  (або відповідну beta/версію виправлення), щоб перевірити опублікований шлях
  встановлення з реєстру в новому тимчасовому префіксі
- Автоматизація релізів maintainer-ів тепер використовує схему preflight-then-promote:
  - реальна публікація в npm має пройти успішний npm `preflight_run_id`
  - stable npm-релізи за замовчуванням націлюються на `beta`
  - stable npm-публікація може явно націлюватися на `latest` через вхід workflow
  - підвищення stable npm з `beta` до `latest` усе ще доступне як явний ручний режим у довіреному workflow `OpenClaw NPM Release`
  - цей режим підвищення все ще потребує дійсного `NPM_TOKEN` у середовищі `npm-release`, оскільки керування npm `dist-tag` виконується окремо від trusted publishing
  - публічний `macOS Release` призначений лише для валідації
  - реальна приватна mac-публікація має пройти успішні приватні mac
    `preflight_run_id` і `validate_run_id`
  - реальні шляхи публікації підвищують уже підготовлені артефакти замість їх повторного збирання
- Для stable-виправлень на кшталт `YYYY.M.D-N` post-publish verifier
  також перевіряє той самий шлях оновлення в тимчасовому префіксі з `YYYY.M.D` до `YYYY.M.D-N`,
  щоб виправлення релізу не могли непомітно залишити старі глобальні встановлення на
  базовому stable-вмісті
- Передрелізна перевірка npm-релізу завершується помилкою за замовчуванням, якщо tarball не містить
  і `dist/control-ui/index.html`, і непорожній вміст `dist/control-ui/assets/`,
  щоб ми знову не випустили порожню панель браузера
- Якщо робота над релізом зачепила планування CI, маніфести часу виконання extensions або
  матриці тестування extensions, перед затвердженням згенеруйте заново та перегляньте
  workflow matrix outputs, якими керує planner,
  `checks-node-extensions` з `.github/workflows/ci.yml`, щоб примітки до релізу не
  описували застарілу структуру CI
- Готовність stable-релізу для macOS також охоплює поверхні оновлення:
  - GitHub-реліз зрештою має містити паковані `.zip`, `.dmg` і `.dSYM.zip`
  - `appcast.xml` у `main` після публікації має вказувати на новий stable zip
  - пакований застосунок має зберігати non-debug bundle id, непорожній Sparkle feed
    URL і `CFBundleVersion` на рівні або вище канонічного мінімального Sparkle build
    для цієї версії релізу

## Вхідні параметри workflow NPM

`OpenClaw NPM Release` приймає такі керовані оператором вхідні параметри:

- `tag`: обов’язковий тег релізу, наприклад `v2026.4.2`, `v2026.4.2-1` або
  `v2026.4.2-beta.1`; коли `preflight_only=true`, це також може бути поточний
  повний 40-символьний SHA коміту `main` для передрелізної перевірки лише з валідацією
- `preflight_only`: `true` лише для валідації/збирання/пакування, `false` для
  реального шляху публікації
- `preflight_run_id`: обов’язковий на реальному шляху публікації, щоб workflow повторно використав
  підготовлений tarball з успішного запуску передрелізної перевірки
- `npm_dist_tag`: цільовий npm-тег для шляху публікації; за замовчуванням `beta`
- `promote_beta_to_latest`: `true`, щоб пропустити публікацію й перемістити вже опубліковану
  stable beta-збірку на `latest`

`OpenClaw Release Checks` приймає такі керовані оператором вхідні параметри:

- `ref`: наявний тег релізу або поточний повний 40-символьний SHA коміту `main`
  для перевірки

Правила:

- Stable-теги та теги виправлень можуть публікуватися або в `beta`, або в `latest`
- Beta prerelease-теги можуть публікуватися лише в `beta`
- Повний SHA коміту дозволений лише коли `preflight_only=true`
- Режим SHA коміту для перевірок релізу також вимагає поточний HEAD `origin/main`
- Реальний шлях публікації має використовувати той самий `npm_dist_tag`, який використовувався під час передрелізної перевірки;
  workflow перевіряє ці метадані перед продовженням публікації
- Режим підвищення має використовувати stable-тег або тег виправлення, `preflight_only=false`,
  порожній `preflight_run_id` і `npm_dist_tag=beta`
- Режим підвищення також вимагає дійсний `NPM_TOKEN` у середовищі `npm-release`,
  оскільки `npm dist-tag add` усе ще потребує звичайної npm-автентифікації

## Послідовність stable npm-релізу

Під час випуску stable npm-релізу:

1. Запустіть `OpenClaw NPM Release` з `preflight_only=true`
   - До появи тега ви можете використовувати поточний повний SHA коміту `main` для
     dry run передрелізного workflow лише з валідацією
2. Виберіть `npm_dist_tag=beta` для звичайного потоку beta-first або `latest` лише
   коли ви навмисно хочете пряму stable-публікацію
3. Окремо запустіть `OpenClaw Release Checks` з тим самим тегом або
   повним поточним SHA `main`, якщо вам потрібне live-покриття prompt cache
   - Це навмисно окремо, щоб live-покриття залишалося доступним без
     повторного поєднання довготривалих або нестабільних перевірок із workflow публікації
4. Збережіть успішний `preflight_run_id`
5. Знову запустіть `OpenClaw NPM Release` з `preflight_only=false`, тим самим
   `tag`, тим самим `npm_dist_tag` і збереженим `preflight_run_id`
6. Якщо реліз потрапив у `beta`, пізніше запустіть `OpenClaw NPM Release` з
   тим самим stable `tag`, `promote_beta_to_latest=true`, `preflight_only=false`,
   порожнім `preflight_run_id` і `npm_dist_tag=beta`, коли захочете перемістити цю
   опубліковану збірку до `latest`

Режим підвищення все ще потребує затвердження середовища `npm-release` і
дійсного `NPM_TOKEN` у цьому середовищі.

Це зберігає як шлях прямої публікації, так і шлях підвищення beta-first
задокументованими та видимими для оператора.

## Публічні посилання

- [`.github/workflows/openclaw-npm-release.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-npm-release.yml)
- [`.github/workflows/openclaw-release-checks.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-release-checks.yml)
- [`scripts/openclaw-npm-release-check.ts`](https://github.com/openclaw/openclaw/blob/main/scripts/openclaw-npm-release-check.ts)
- [`scripts/package-mac-dist.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/package-mac-dist.sh)
- [`scripts/make_appcast.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/make_appcast.sh)

Maintainer-и використовують приватну документацію релізів у
[`openclaw/maintainers/release/README.md`](https://github.com/openclaw/maintainers/blob/main/release/README.md)
для фактичного runbook.
