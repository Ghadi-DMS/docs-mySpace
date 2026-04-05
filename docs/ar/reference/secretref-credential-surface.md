---
read_when:
    - التحقق من تغطية بيانات اعتماد SecretRef
    - تدقيق ما إذا كانت بيانات الاعتماد مؤهلة لـ `secrets configure` أو `secrets apply`
    - التحقق من سبب كون بيانات الاعتماد خارج السطح المعتمد
summary: السطح الرسمي المعتمد وغير المعتمد لبيانات اعتماد SecretRef
title: سطح بيانات اعتماد SecretRef
x-i18n:
    generated_at: "2026-04-05T12:55:13Z"
    model: gpt-5.4
    provider: openai
    source_hash: bf997389de1dae8c059d8dfbf186eda979f864de632a033177d6cd5e5544675d
    source_path: reference/secretref-credential-surface.md
    workflow: 15
---

# سطح بيانات اعتماد SecretRef

تحدد هذه الصفحة السطح الرسمي لبيانات اعتماد SecretRef.

نية النطاق:

- ضمن النطاق: بيانات الاعتماد التي يوفّرها المستخدم مباشرةً والتي لا يقوم OpenClaw بإصدارها أو تدويرها.
- خارج النطاق: بيانات الاعتماد التي يتم إصدارها أو تدويرها وقت التشغيل، ومواد تحديث OAuth، والعناصر الشبيهة بالجلسات.

## بيانات الاعتماد المعتمدة

### أهداف `openclaw.json` (`secrets configure` + `secrets apply` + `secrets audit`)

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
- `plugins.entries.firecrawl.config.webFetch.apiKey`
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
- `channels.googlechat.serviceAccount` عبر `serviceAccountRef` المجاور (استثناء توافق)
- `channels.googlechat.accounts.*.serviceAccount` عبر `serviceAccountRef` المجاور (استثناء توافق)

### أهداف `auth-profiles.json` (`secrets configure` + `secrets apply` + `secrets audit`)

- `profiles.*.keyRef` (`type: "api_key"`؛ غير معتمد عندما تكون `auth.profiles.<id>.mode = "oauth"`)
- `profiles.*.tokenRef` (`type: "token"`؛ غير معتمد عندما تكون `auth.profiles.<id>.mode = "oauth"`)

[//]: # "secretref-supported-list-end"

ملاحظات:

- تتطلب أهداف خطة ملف التعريف للمصادقة `agentId`.
- تستهدف إدخالات الخطة `profiles.*.key` / `profiles.*.token` وتكتب المراجع المجاورة (`keyRef` / `tokenRef`).
- يتم تضمين مراجع ملفات تعريف المصادقة في تحليل وقت التشغيل وتغطية التدقيق.
- حاجز سياسة OAuth: لا يمكن الجمع بين `auth.profiles.<id>.mode = "oauth"` ومدخلات SecretRef لذلك الملف التعريفي. تفشل عملية البدء/إعادة التحميل وتحليل ملف تعريف المصادقة بسرعة عند انتهاك هذه السياسة.
- بالنسبة إلى موفري النماذج المُدارين بواسطة SecretRef، فإن إدخالات `agents/*/agent/models.json` المُولَّدة تحتفظ بعلامات غير سرية (وليس قيم الأسرار المحلولة) لأسطح `apiKey`/الترويسات.
- استمرار العلامات يعتمد على المصدر: يكتب OpenClaw العلامات من لقطة تهيئة المصدر النشطة (قبل التحليل)، وليس من قيم الأسرار المحلولة في وقت التشغيل.
- بالنسبة إلى البحث على الويب:
  - في وضع الموفّر الصريح (عند تعيين `tools.web.search.provider`)، يكون مفتاح الموفّر المحدد فقط نشطًا.
  - في الوضع التلقائي (عند عدم تعيين `tools.web.search.provider`)، يكون مفتاح الموفّر الأول فقط الذي يتم تحليله حسب ترتيب الأولوية نشطًا.
  - في الوضع التلقائي، تُعامل مراجع الموفّرين غير المحددين على أنها غير نشطة حتى يتم تحديدها.
  - لا تزال مسارات الموفّر القديمة `tools.web.search.*` تُحل خلال نافذة التوافق، لكن السطح الرسمي لـ SecretRef هو `plugins.entries.<plugin>.config.webSearch.*`.

## بيانات الاعتماد غير المعتمدة

تتضمن بيانات الاعتماد خارج النطاق ما يلي:

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

المبرر:

- هذه بيانات اعتماد مُصدَرة أو مُدوَّرة أو مرتبطة بالجلسات أو من الفئات الدائمة لـ OAuth التي لا تتوافق مع تحليل SecretRef الخارجي للقراءة فقط.
