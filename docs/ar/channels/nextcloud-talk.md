---
read_when:
    - العمل على ميزات قناة Nextcloud Talk
summary: حالة دعم Nextcloud Talk وإمكاناته وتكوينه
title: Nextcloud Talk
x-i18n:
    generated_at: "2026-04-05T12:35:51Z"
    model: gpt-5.4
    provider: openai
    source_hash: 900402afe67cf3ce96103d55158eb28cffb29c9845b77248e70d7653b12ae810
    source_path: channels/nextcloud-talk.md
    workflow: 15
---

# Nextcloud Talk

الحالة: plugin مضمّن (روبوت webhook). الرسائل المباشرة، والغرف، والتفاعلات، ورسائل markdown مدعومة.

## Plugin مضمّن

يأتي Nextcloud Talk كـ plugin مضمّن في إصدارات OpenClaw الحالية، لذلك
لا تحتاج الإصدارات المجمعة العادية إلى تثبيت منفصل.

إذا كنت تستخدم إصدارًا أقدم أو تثبيتًا مخصصًا لا يتضمن Nextcloud Talk،
فقم بتثبيته يدويًا:

التثبيت عبر CLI (سجل npm):

```bash
openclaw plugins install @openclaw/nextcloud-talk
```

نسخة محلية (عند التشغيل من مستودع git):

```bash
openclaw plugins install ./path/to/local/nextcloud-talk-plugin
```

التفاصيل: [Plugins](/tools/plugin)

## إعداد سريع (للمبتدئين)

1. تأكد من أن plugin الخاص بـ Nextcloud Talk متاح.
   - تتضمنه إصدارات OpenClaw المجمعة الحالية بالفعل.
   - يمكن لعمليات التثبيت الأقدم/المخصصة إضافته يدويًا بالأوامر أعلاه.
2. على خادم Nextcloud لديك، أنشئ روبوتًا:

   ```bash
   ./occ talk:bot:install "OpenClaw" "<shared-secret>" "<webhook-url>" --feature reaction
   ```

3. فعّل الروبوت في إعدادات الغرفة المستهدفة.
4. اضبط OpenClaw:
   - التكوين: `channels.nextcloud-talk.baseUrl` + `channels.nextcloud-talk.botSecret`
   - أو متغير البيئة: `NEXTCLOUD_TALK_BOT_SECRET` (للحساب الافتراضي فقط)
5. أعد تشغيل gateway (أو أكمل الإعداد).

الحد الأدنى من التكوين:

```json5
{
  channels: {
    "nextcloud-talk": {
      enabled: true,
      baseUrl: "https://cloud.example.com",
      botSecret: "shared-secret",
      dmPolicy: "pairing",
    },
  },
}
```

## ملاحظات

- لا يمكن للروبوتات بدء الرسائل المباشرة. يجب أن يراسل المستخدم الروبوت أولًا.
- يجب أن يكون عنوان URL الخاص بـ webhook قابلاً للوصول من Gateway؛ اضبط `webhookPublicUrl` إذا كنت خلف proxy.
- تحميل الوسائط غير مدعوم بواسطة bot API؛ تُرسل الوسائط على شكل عناوين URL.
- لا تميّز حمولة webhook بين الرسائل المباشرة والغرف؛ اضبط `apiUser` + `apiPassword` لتمكين عمليات البحث عن نوع الغرفة (وإلا فستُعامل الرسائل المباشرة على أنها غرف).

## التحكم في الوصول (الرسائل المباشرة)

- الافتراضي: `channels.nextcloud-talk.dmPolicy = "pairing"`. يحصل المرسلون غير المعروفين على رمز pairing.
- وافق عبر:
  - `openclaw pairing list nextcloud-talk`
  - `openclaw pairing approve nextcloud-talk <CODE>`
- الرسائل المباشرة العامة: `channels.nextcloud-talk.dmPolicy="open"` مع `channels.nextcloud-talk.allowFrom=["*"]`.
- يطابق `allowFrom` معرّفات مستخدمي Nextcloud فقط؛ ويتم تجاهل أسماء العرض.

## الغرف (المجموعات)

- الافتراضي: `channels.nextcloud-talk.groupPolicy = "allowlist"` (مع تقييد الإشارات).
- أضف الغرف إلى قائمة السماح باستخدام `channels.nextcloud-talk.rooms`:

```json5
{
  channels: {
    "nextcloud-talk": {
      rooms: {
        "room-token": { requireMention: true },
      },
    },
  },
}
```

- لعدم السماح بأي غرف، اترك قائمة السماح فارغة أو اضبط `channels.nextcloud-talk.groupPolicy="disabled"`.

## الإمكانات

| الميزة           | الحالة        |
| ---------------- | ------------- |
| الرسائل المباشرة | مدعومة        |
| الغرف            | مدعومة        |
| الخيوط           | غير مدعومة    |
| الوسائط          | عناوين URL فقط |
| التفاعلات        | مدعومة        |
| الأوامر الأصلية  | غير مدعومة    |

## مرجع التكوين (Nextcloud Talk)

التكوين الكامل: [Configuration](/gateway/configuration)

خيارات المزوّد:

- `channels.nextcloud-talk.enabled`: تمكين/تعطيل بدء تشغيل القناة.
- `channels.nextcloud-talk.baseUrl`: عنوان URL لنسخة Nextcloud.
- `channels.nextcloud-talk.botSecret`: السر المشترك للروبوت.
- `channels.nextcloud-talk.botSecretFile`: مسار سر لملف عادي. يتم رفض الروابط الرمزية.
- `channels.nextcloud-talk.apiUser`: مستخدم API لعمليات البحث عن الغرف (اكتشاف الرسائل المباشرة).
- `channels.nextcloud-talk.apiPassword`: كلمة مرور API/app لعمليات البحث عن الغرف.
- `channels.nextcloud-talk.apiPasswordFile`: مسار ملف كلمة مرور API.
- `channels.nextcloud-talk.webhookPort`: منفذ مستمع webhook (الافتراضي: 8788).
- `channels.nextcloud-talk.webhookHost`: مضيف webhook (الافتراضي: 0.0.0.0).
- `channels.nextcloud-talk.webhookPath`: مسار webhook (الافتراضي: /nextcloud-talk-webhook).
- `channels.nextcloud-talk.webhookPublicUrl`: عنوان URL خارجي يمكن الوصول إليه لـ webhook.
- `channels.nextcloud-talk.dmPolicy`: ‏`pairing | allowlist | open | disabled`.
- `channels.nextcloud-talk.allowFrom`: قائمة سماح الرسائل المباشرة (معرّفات المستخدمين). يتطلب `open` القيمة `"*"`.
- `channels.nextcloud-talk.groupPolicy`: ‏`allowlist | open | disabled`.
- `channels.nextcloud-talk.groupAllowFrom`: قائمة سماح المجموعات (معرّفات المستخدمين).
- `channels.nextcloud-talk.rooms`: إعدادات لكل غرفة وقائمة السماح.
- `channels.nextcloud-talk.historyLimit`: حد سجل المجموعات (يعطل عند 0).
- `channels.nextcloud-talk.dmHistoryLimit`: حد سجل الرسائل المباشرة (يعطل عند 0).
- `channels.nextcloud-talk.dms`: عمليات تجاوز لكل رسالة مباشرة (`historyLimit`).
- `channels.nextcloud-talk.textChunkLimit`: حجم تجزئة النص الصادر (أحرف).
- `channels.nextcloud-talk.chunkMode`: ‏`length` (الافتراضي) أو `newline` للتقسيم عند الأسطر الفارغة (حدود الفقرات) قبل التجزئة حسب الطول.
- `channels.nextcloud-talk.blockStreaming`: تعطيل block streaming لهذه القناة.
- `channels.nextcloud-talk.blockStreamingCoalesce`: ضبط دمج block streaming.
- `channels.nextcloud-talk.mediaMaxMb`: الحد الأقصى للوسائط الواردة (MB).

## ذو صلة

- [نظرة عامة على القنوات](/channels) — جميع القنوات المدعومة
- [Pairing](/channels/pairing) — مصادقة الرسائل المباشرة وتدفق pairing
- [المجموعات](/channels/groups) — سلوك الدردشة الجماعية وتقييد الإشارات
- [توجيه القنوات](/channels/channel-routing) — توجيه الجلسات للرسائل
- [الأمان](/gateway/security) — نموذج الوصول والتقوية
