---
read_when:
    - Перевірка покриття облікових даних SecretRef
    - Аудит того, чи облікові дані підпадають під `secrets configure` або `secrets apply`
    - Перевірка, чому облікові дані перебувають поза підтримуваною поверхнею
summary: Канонічна підтримувана та непідтримувана поверхня облікових даних SecretRef
title: Поверхня облікових даних SecretRef
x-i18n:
    generated_at: "2026-04-06T16:23:36Z"
    model: gpt-5.4
    provider: openai
    source_hash: 211f4b504c5808f7790683066fc2c8b700c705c598f220a264daf971b81cc593
    source_path: reference/secretref-credential-surface.md
    workflow: 15
---

# Поверхня облікових даних SecretRef

Ця сторінка визначає канонічну поверхню облікових даних SecretRef.

Наміри щодо сфери охоплення:

- У сфері охоплення: виключно облікові дані, надані користувачем, які OpenClaw не створює і не ротує.
- Поза сферою охоплення: облікові дані, що створюються або ротуються під час виконання, матеріали OAuth refresh і артефакти, подібні до сесій.

## Підтримувані облікові дані

### Цілі `openclaw.json` (`secrets configure` + `secrets apply` + `secrets audit`)

[//]: # "secretref-supported-list-start"

- `models.providers.*.apiKey`
- `models.providers.*.headers.*`
- `models.providers.*.request.auth.token`
- `models.providers.*.request.auth.value`
- `models.providers.*.request.headers.*`
- `models.providers.*.request.proxy.tls.ca`
- `models.providers.*.request.proxy.tls.cert`
- `models.providers.*.request.proxy.tls.key`
- `models.providers.*.request.proxy.tls.passphrase`
- `models.providers.*.request.tls.ca`
- `models.providers.*.request.tls.cert`
- `models.providers.*.request.tls.key`
- `models.providers.*.request.tls.passphrase`
- `skills.entries.*.apiKey`
- `agents.defaults.memorySearch.remote.apiKey`
- `agents.list[].memorySearch.remote.apiKey`
- `talk.providers.*.apiKey`
- `messages.tts.providers.*.apiKey`
- `tools.web.fetch.firecrawl.apiKey`
- `plugins.entries.brave.config.webSearch.apiKey`
- `plugins.entries.google.config.webSearch.apiKey`
- `plugins.entries.xai.config.webSearch.apiKey`
- `plugins.entries.moonshot.config.webSearch.apiKey`
- `plugins.entries.perplexity.config.webSearch.apiKey`
- `plugins.entries.firecrawl.config.webSearch.apiKey`
- `plugins.entries.minimax.config.webSearch.apiKey`
- `plugins.entries.tavily.config.webSearch.apiKey`
- `tools.web.search.apiKey`
- `gateway.auth.password`
- `gateway.auth.token`
- `gateway.remote.token`
- `gateway.remote.password`
- `cron.webhookToken`
- `channels.telegram.botToken`
- `channels.telegram.webhookSecret`
- `channels.telegram.accounts.*.botToken`
- `channels.telegram.accounts.*.webhookSecret`
- `channels.slack.botToken`
- `channels.slack.appToken`
- `channels.slack.userToken`
- `channels.slack.signingSecret`
- `channels.slack.accounts.*.botToken`
- `channels.slack.accounts.*.appToken`
- `channels.slack.accounts.*.userToken`
- `channels.slack.accounts.*.signingSecret`
- `channels.discord.token`
- `channels.discord.pluralkit.token`
- `channels.discord.voice.tts.providers.*.apiKey`
- `channels.discord.accounts.*.token`
- `channels.discord.accounts.*.pluralkit.token`
- `channels.discord.accounts.*.voice.tts.providers.*.apiKey`
- `channels.irc.password`
- `channels.irc.nickserv.password`
- `channels.irc.accounts.*.password`
- `channels.irc.accounts.*.nickserv.password`
- `channels.bluebubbles.password`
- `channels.bluebubbles.accounts.*.password`
- `channels.feishu.appSecret`
- `channels.feishu.encryptKey`
- `channels.feishu.verificationToken`
- `channels.feishu.accounts.*.appSecret`
- `channels.feishu.accounts.*.encryptKey`
- `channels.feishu.accounts.*.verificationToken`
- `channels.msteams.appPassword`
- `channels.mattermost.botToken`
- `channels.mattermost.accounts.*.botToken`
- `channels.matrix.accessToken`
- `channels.matrix.password`
- `channels.matrix.accounts.*.accessToken`
- `channels.matrix.accounts.*.password`
- `channels.nextcloud-talk.botSecret`
- `channels.nextcloud-talk.apiPassword`
- `channels.nextcloud-talk.accounts.*.botSecret`
- `channels.nextcloud-talk.accounts.*.apiPassword`
- `channels.zalo.botToken`
- `channels.zalo.webhookSecret`
- `channels.zalo.accounts.*.botToken`
- `channels.zalo.accounts.*.webhookSecret`
- `channels.googlechat.serviceAccount` через сусідній `serviceAccountRef` (виняток сумісності)
- `channels.googlechat.accounts.*.serviceAccount` через сусідній `serviceAccountRef` (виняток сумісності)

### Цілі `auth-profiles.json` (`secrets configure` + `secrets apply` + `secrets audit`)

- `profiles.*.keyRef` (`type: "api_key"`; не підтримується, коли `auth.profiles.<id>.mode = "oauth"`)
- `profiles.*.tokenRef` (`type: "token"`; не підтримується, коли `auth.profiles.<id>.mode = "oauth"`)

[//]: # "secretref-supported-list-end"

Примітки:

- Цілі плану auth-profile вимагають `agentId`.
- Записи плану націлюються на `profiles.*.key` / `profiles.*.token` і записують сусідні посилання (`keyRef` / `tokenRef`).
- Посилання auth-profile включені в покриття розв’язання під час виконання та аудиту.
- Захист політики OAuth: `auth.profiles.<id>.mode = "oauth"` не можна поєднувати з входами SecretRef для цього профілю. Запуск/перезавантаження та розв’язання auth-profile аварійно завершуються одразу, якщо цю політику порушено.
- Для постачальників моделей, керованих SecretRef, згенеровані записи `agents/*/agent/models.json` зберігають не секретні маркери (а не розв’язані секретні значення) для поверхонь `apiKey`/заголовків.
- Збереження маркерів є авторитетним щодо джерела: OpenClaw записує маркери з активного знімка конфігурації джерела (до розв’язання), а не з розв’язаних значень секретів під час виконання.
- Для вебпошуку:
  - У режимі явного постачальника (`tools.web.search.provider` задано) активним є лише ключ вибраного постачальника.
  - В автоматичному режимі (`tools.web.search.provider` не задано) активним є лише перший ключ постачальника, який розв’язується за пріоритетом.
  - В автоматичному режимі посилання не вибраних постачальників вважаються неактивними, доки їх не буде вибрано.
  - Застарілі шляхи постачальників `tools.web.search.*` усе ще розв’язуються протягом вікна сумісності, але канонічна поверхня SecretRef — це `plugins.entries.<plugin>.config.webSearch.*`.

## Непідтримувані облікові дані

Облікові дані поза сферою охоплення включають:

[//]: # "secretref-unsupported-list-start"

- `commands.ownerDisplaySecret`
- `hooks.token`
- `hooks.gmail.pushToken`
- `hooks.mappings[].sessionKey`
- `auth-profiles.oauth.*`
- `channels.discord.threadBindings.webhookToken`
- `channels.discord.accounts.*.threadBindings.webhookToken`
- `channels.whatsapp.creds.json`
- `channels.whatsapp.accounts.*.creds.json`

[//]: # "secretref-unsupported-list-end"

Обґрунтування:

- Це створювані, ротаційні, сесійні або OAuth-стійкі класи облікових даних, які не відповідають моделі розв’язання зовнішнього SecretRef лише для читання.
