---
read_when:
    - تريد اختيار قناة دردشة لـ OpenClaw
    - تحتاج إلى نظرة عامة سريعة على منصات المراسلة المدعومة
summary: منصات المراسلة التي يمكن لـ OpenClaw الاتصال بها
title: قنوات الدردشة
x-i18n:
    generated_at: "2026-04-05T12:35:16Z"
    model: gpt-5.4
    provider: openai
    source_hash: 246ee6f16aebe751241f00102bb435978ed21f6158385aff5d8e222e30567416
    source_path: channels/index.md
    workflow: 15
---

# قنوات الدردشة

يمكن لـ OpenClaw التحدث إليك عبر أي تطبيق دردشة تستخدمه بالفعل. تتصل كل قناة عبر Gateway.
النص مدعوم في كل مكان؛ أما الوسائط والتفاعلات فتختلف حسب القناة.

## القنوات المدعومة

- [BlueBubbles](/channels/bluebubbles) — **موصى به لـ iMessage**؛ يستخدم واجهة REST API لخادم BlueBubbles على macOS مع دعم كامل للميزات (إضافة مدمجة؛ التعديل، وإلغاء الإرسال، والتأثيرات، والتفاعلات، وإدارة المجموعات — التعديل معطّل حاليًا على macOS 26 Tahoe).
- [Discord](/channels/discord) — Discord Bot API + Gateway؛ يدعم الخوادم والقنوات والرسائل المباشرة.
- [Feishu](/channels/feishu) — بوت Feishu/Lark عبر WebSocket (إضافة مدمجة).
- [Google Chat](/channels/googlechat) — تطبيق Google Chat API عبر HTTP webhook.
- [iMessage (legacy)](/channels/imessage) — تكامل macOS قديم عبر CLI المسمى imsg (مهمل، استخدم BlueBubbles في الإعدادات الجديدة).
- [IRC](/channels/irc) — خوادم IRC التقليدية؛ قنوات + رسائل مباشرة مع عناصر تحكم في الاقتران/قائمة السماح.
- [LINE](/channels/line) — بوت LINE Messaging API (إضافة مدمجة).
- [Matrix](/channels/matrix) — بروتوكول Matrix (إضافة مدمجة).
- [Mattermost](/channels/mattermost) — Bot API + WebSocket؛ قنوات ومجموعات ورسائل مباشرة (إضافة مدمجة).
- [Microsoft Teams](/channels/msteams) — Bot Framework؛ دعم للمؤسسات (إضافة مدمجة).
- [Nextcloud Talk](/channels/nextcloud-talk) — دردشة مستضافة ذاتيًا عبر Nextcloud Talk (إضافة مدمجة).
- [Nostr](/channels/nostr) — رسائل مباشرة لامركزية عبر NIP-04 (إضافة مدمجة).
- [QQ Bot](/channels/qqbot) — QQ Bot API؛ دردشة خاصة ودردشة جماعية ووسائط غنية (إضافة مدمجة).
- [Signal](/channels/signal) — signal-cli؛ يركز على الخصوصية.
- [Slack](/channels/slack) — Bolt SDK؛ تطبيقات مساحات العمل.
- [Synology Chat](/channels/synology-chat) — Synology NAS Chat عبر webhooks صادرة + واردة (إضافة مدمجة).
- [Telegram](/channels/telegram) — Bot API عبر grammY؛ يدعم المجموعات.
- [Tlon](/channels/tlon) — تطبيق مراسلة قائم على Urbit (إضافة مدمجة).
- [Twitch](/channels/twitch) — دردشة Twitch عبر اتصال IRC (إضافة مدمجة).
- [Voice Call](/plugins/voice-call) — اتصالات هاتفية عبر Plivo أو Twilio (إضافة، تُثبت بشكل منفصل).
- [WebChat](/web/webchat) — واجهة Gateway WebChat عبر WebSocket.
- [WeChat](https://www.npmjs.com/package/@tencent-weixin/openclaw-weixin) — إضافة Tencent iLink Bot عبر تسجيل دخول QR؛ دردشات خاصة فقط.
- [WhatsApp](/channels/whatsapp) — الأكثر شيوعًا؛ يستخدم Baileys ويتطلب اقتران QR.
- [Zalo](/channels/zalo) — Zalo Bot API؛ تطبيق المراسلة الشائع في فيتنام (إضافة مدمجة).
- [Zalo Personal](/channels/zalouser) — حساب Zalo شخصي عبر تسجيل دخول QR (إضافة مدمجة).

## ملاحظات

- يمكن تشغيل القنوات في الوقت نفسه؛ اضبط عدة قنوات وسيقوم OpenClaw بالتوجيه حسب كل دردشة.
- غالبًا ما يكون **Telegram** هو الأسرع في الإعداد (رمز بوت بسيط). يتطلب WhatsApp اقتران QR و
  يخزن حالة أكثر على القرص.
- يختلف سلوك المجموعات حسب القناة؛ راجع [المجموعات](/channels/groups).
- يتم فرض الاقتران في الرسائل المباشرة وقوائم السماح لأسباب تتعلق بالسلامة؛ راجع [الأمان](/gateway/security).
- استكشاف الأخطاء وإصلاحها: [استكشاف أخطاء القنوات وإصلاحها](/channels/troubleshooting).
- يتم توثيق موفري النماذج بشكل منفصل؛ راجع [موفرو النماذج](/providers/models).
