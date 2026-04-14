---
read_when:
    - Шукаю визначення публічних каналів випуску
    - Шукаю найменування версій і частоту випусків
summary: Публічні канали випуску, найменування версій і частота випусків
title: Політика випусків
x-i18n:
    generated_at: "2026-04-14T16:20:31Z"
    model: gpt-5.4
    provider: openai
    source_hash: 021ae4b3e6a258c8396eccb99f00ca6cc268c2496258786521a053ebd776ba60
    source_path: reference/RELEASING.md
    workflow: 15
---

# Політика випусків

OpenClaw має три публічні канали випусків:

- stable: теговані випуски, які за замовчуванням публікуються в npm `beta`, або в npm `latest`, якщо це явно запитано
- beta: prerelease-теги, які публікуються в npm `beta`
- dev: рухома вершина `main`

## Найменування версій

- Версія stable-випуску: `YYYY.M.D`
  - Git-тег: `vYYYY.M.D`
- Версія stable-коригувального випуску: `YYYY.M.D-N`
  - Git-тег: `vYYYY.M.D-N`
- Версія beta-prerelease: `YYYY.M.D-beta.N`
  - Git-тег: `vYYYY.M.D-beta.N`
- Не додавайте провідні нулі до місяця або дня
- `latest` означає поточний просунутий stable-випуск npm
- `beta` означає поточну ціль встановлення beta
- Stable і stable-коригувальні випуски за замовчуванням публікуються в npm `beta`; оператори випусків можуть явно націлити `latest` або просунути перевірену beta-збірку пізніше
- Кожен випуск OpenClaw постачається разом як npm-пакет і macOS app

## Частота випусків

- Випуски спочатку проходять через beta
- Stable з’являється лише після перевірки останньої beta
- Детальна процедура випуску, погодження, облікові дані та примітки щодо відновлення
  доступні лише мейнтейнерам

## Передрелізна перевірка

- Запустіть `pnpm build && pnpm ui:build` перед `pnpm release:check`, щоб для кроку
  перевірки pack були наявні очікувані артефакти випуску `dist/*` і збірка Control UI
- Запускайте `pnpm release:check` перед кожним тегованим випуском
- Перевірки випуску тепер запускаються в окремому ручному workflow:
  `OpenClaw Release Checks`
- Цей поділ є навмисним: він зберігає реальний шлях npm-випуску коротким,
  детермінованим і зосередженим на артефактах, тоді як повільніші live-перевірки
  залишаються у власному каналі, щоб не затримувати й не блокувати публікацію
- Перевірки випуску мають запускатися з workflow ref `main`, щоб логіка
  workflow і секрети залишалися канонічними
- Цей workflow приймає або наявний тег випуску, або поточний повний
  40-символьний SHA коміту `main`
- У режимі commit-SHA він приймає лише поточний HEAD `origin/main`; для
  старіших комітів випуску використовуйте тег випуску
- Передрелізна перевірка лише для валідації `OpenClaw NPM Release` також приймає
  поточний повний 40-символьний SHA коміту `main` без вимоги наявності
  запушеного тега
- Цей шлях SHA є лише для валідації й не може бути просунутий у реальну публікацію
- У режимі SHA workflow синтезує `v<package.json version>` лише для перевірки
  метаданих пакета; реальна публікація все одно вимагає реального тега випуску
- Обидва workflow залишають реальний шлях публікації й просування на
  GitHub-hosted runners, тоді як немутувальний шлях валідації може
  використовувати більші Linux runners Blacksmith
- Цей workflow запускає
  `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_CACHE_TEST=1 pnpm test:live:cache`
  з використанням обох workflow secrets: `OPENAI_API_KEY` і `ANTHROPIC_API_KEY`
- Передрелізна перевірка npm більше не очікує завершення окремого каналу
  перевірок випуску
- Перед погодженням запустіть `RELEASE_TAG=vYYYY.M.D node --import tsx scripts/openclaw-npm-release-check.ts`
  (або відповідний beta/коригувальний тег)
- Після публікації в npm запустіть
  `node --import tsx scripts/openclaw-npm-postpublish-verify.ts YYYY.M.D`
  (або відповідну beta/коригувальну версію), щоб перевірити шлях встановлення
  опублікованого реєстру в новому тимчасовому prefix
- Автоматизація випусків мейнтейнерів тепер використовує підхід preflight-then-promote:
  - реальна публікація в npm має пройти успішний npm `preflight_run_id`
  - stable npm-випуски за замовчуванням націлені на `beta`
  - stable npm-публікація може явно націлювати `latest` через вхід workflow
  - мутація npm dist-tag на основі токена тепер знаходиться в
    `openclaw/releases-private/.github/workflows/openclaw-npm-dist-tags.yml`
    з міркувань безпеки, оскільки `npm dist-tag add` усе ще потребує `NPM_TOKEN`, тоді як
    публічний репозиторій зберігає публікацію лише через OIDC
  - публічний `macOS Release` призначений лише для валідації
  - реальна приватна mac-публікація має пройти успішні приватні mac
    `preflight_run_id` і `validate_run_id`
  - реальні шляхи публікації просувають підготовлені артефакти замість їх повторного збирання
- Для stable-коригувальних випусків на кшталт `YYYY.M.D-N` верифікатор після публікації
  також перевіряє той самий шлях оновлення тимчасового prefix з `YYYY.M.D` до `YYYY.M.D-N`,
  щоб коригування випусків не могли непомітно залишити старіші глобальні встановлення
  на базовому stable-навантаженні
- Передрелізна перевірка npm завершується за принципом fail closed, якщо tarball не містить
  і `dist/control-ui/index.html`, і непорожнє навантаження `dist/control-ui/assets/`,
  щоб ми знову не випустили порожню панель браузера
- `pnpm test:install:smoke` також застосовує бюджет npm pack `unpackedSize` до
  tarball кандидата на оновлення, щоб installer e2e перехоплював випадкове
  роздуття pack до шляху публікації випуску
- Якщо робота над випуском торкалася планування CI, маніфестів часу extension або
  матриць тестів extension, перед погодженням повторно згенеруйте й перегляньте
  workflow matrix outputs, що належать planner, для `checks-node-extensions` з `.github/workflows/ci.yml`,
  щоб примітки до випуску не описували застарілу структуру CI
- Готовність stable macOS-випуску також включає поверхні оновлювача:
  - GitHub release має зрештою містити запаковані `.zip`, `.dmg` і `.dSYM.zip`
  - `appcast.xml` у `main` після публікації має вказувати на новий stable zip
  - запакований app має зберігати не-debug bundle id, непорожній Sparkle feed
    URL і `CFBundleVersion` на рівні або вище канонічного Sparkle build floor
    для цієї версії випуску

## Входи workflow NPM

`OpenClaw NPM Release` приймає такі вхідні дані, керовані оператором:

- `tag`: обов’язковий тег випуску, наприклад `v2026.4.2`, `v2026.4.2-1` або
  `v2026.4.2-beta.1`; коли `preflight_only=true`, це також може бути поточний
  повний 40-символьний SHA коміту `main` для передрелізної перевірки лише для валідації
- `preflight_only`: `true` для лише валідації/збирання/пакування, `false` для
  реального шляху публікації
- `preflight_run_id`: обов’язковий у реальному шляху публікації, щоб workflow повторно використав
  підготовлений tarball з успішного виконання передрелізної перевірки
- `npm_dist_tag`: цільовий npm-тег для шляху публікації; за замовчуванням `beta`

`OpenClaw Release Checks` приймає такі вхідні дані, керовані оператором:

- `ref`: наявний тег випуску або поточний повний 40-символьний commit
  SHA `main` для перевірки

Правила:

- Stable і correction-теги можуть публікуватися або в `beta`, або в `latest`
- Beta prerelease-теги можуть публікуватися лише в `beta`
- Повний вхід commit SHA дозволений лише коли `preflight_only=true`
- Режим commit-SHA для перевірок випуску також вимагає поточний HEAD `origin/main`
- Реальний шлях публікації має використовувати той самий `npm_dist_tag`, який використовувався під час передрелізної перевірки;
  workflow перевіряє ці метадані перед продовженням публікації

## Послідовність stable npm-випуску

Під час створення stable npm-випуску:

1. Запустіть `OpenClaw NPM Release` з `preflight_only=true`
   - До появи тега ви можете використати поточний повний SHA коміту `main` для
     dry run передрелізного workflow лише для валідації
2. Виберіть `npm_dist_tag=beta` для звичайного beta-first потоку або `latest` лише
   коли ви навмисно хочете пряму stable-публікацію
3. Окремо запустіть `OpenClaw Release Checks` з тим самим тегом або
   повним поточним SHA `main`, якщо вам потрібне live-покриття prompt cache
   - Це окремо навмисно, щоб live-покриття залишалося доступним без
     повторного зв’язування довготривалих або нестабільних перевірок із workflow публікації
4. Збережіть успішний `preflight_run_id`
5. Знову запустіть `OpenClaw NPM Release` з `preflight_only=false`, тим самим
   `tag`, тим самим `npm_dist_tag` і збереженим `preflight_run_id`
6. Якщо випуск потрапив у `beta`, використайте приватний workflow
   `openclaw/releases-private/.github/workflows/openclaw-npm-dist-tags.yml`,
   щоб просунути цю stable-версію з `beta` у `latest`
7. Якщо випуск навмисно було опубліковано безпосередньо в `latest`, а `beta`
   має одразу перейти на ту саму stable-збірку, використайте той самий приватний
   workflow, щоб спрямувати обидва dist-tags на stable-версію, або дозвольте його запланованій
   самовідновлювальній синхронізації перемістити `beta` пізніше

Мутація dist-tag знаходиться в приватному репозиторії з міркувань безпеки, оскільки вона все ще
потребує `NPM_TOKEN`, тоді як публічний репозиторій зберігає публікацію лише через OIDC.

Це залишає і шлях прямої публікації, і шлях beta-first просування
задокументованими та видимими для операторів.

## Публічні посилання

- [`.github/workflows/openclaw-npm-release.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-npm-release.yml)
- [`.github/workflows/openclaw-release-checks.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-release-checks.yml)
- [`scripts/openclaw-npm-release-check.ts`](https://github.com/openclaw/openclaw/blob/main/scripts/openclaw-npm-release-check.ts)
- [`scripts/package-mac-dist.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/package-mac-dist.sh)
- [`scripts/make_appcast.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/make_appcast.sh)

Мейнтейнери використовують приватну документацію з випусків у
[`openclaw/maintainers/release/README.md`](https://github.com/openclaw/maintainers/blob/main/release/README.md)
як фактичний runbook.
