---
read_when:
    - أنت تريد أن يستقبل OpenClaw الرسائل المباشرة عبر Nostr
    - أنت تقوم بإعداد مراسلة لامركزية
summary: قناة الرسائل المباشرة Nostr عبر رسائل NIP-04 المشفرة
title: Nostr
x-i18n:
    generated_at: "2026-04-05T12:36:01Z"
    model: gpt-5.4
    provider: openai
    source_hash: f82829ee66fbeb3367007af343797140049ea49f2e842a695fa56acea0c80728
    source_path: channels/nostr.md
    workflow: 15
---

# Nostr

**الحالة:** plugin مضمّن اختياري (معطل افتراضيًا حتى تتم تهيئته).

Nostr هو بروتوكول لامركزي للشبكات الاجتماعية. تتيح هذه القناة لـ OpenClaw استقبال الرسائل المباشرة (DMs) المشفرة والرد عليها عبر NIP-04.

## plugin المضمّن

تشحن إصدارات OpenClaw الحالية Nostr باعتباره plugin مضمّنًا، لذلك لا تحتاج
البنيات المعبأة العادية إلى تثبيت منفصل.

### عمليات التثبيت الأقدم/المخصصة

- لا يزال `openclaw onboard` و`openclaw channels add` يعرضان
  Nostr من فهرس القنوات المشترك.
- إذا كان البناء الخاص بك يستبعد Nostr المضمّن، فثبّته يدويًا.

```bash
openclaw plugins install @openclaw/nostr
```

استخدم نسخة checkout محلية (سير عمل التطوير):

```bash
openclaw plugins install --link <path-to-local-nostr-plugin>
```

أعد تشغيل Gateway بعد تثبيت plugins أو تمكينها.

### إعداد غير تفاعلي

```bash
openclaw channels add --channel nostr --private-key "$NOSTR_PRIVATE_KEY"
openclaw channels add --channel nostr --private-key "$NOSTR_PRIVATE_KEY" --relay-urls "wss://relay.damus.io,wss://relay.primal.net"
```

استخدم `--use-env` للاحتفاظ بـ `NOSTR_PRIVATE_KEY` في البيئة بدلًا من تخزين المفتاح في الإعدادات.

## إعداد سريع

1. أنشئ زوج مفاتيح Nostr ‏(إذا لزم الأمر):

```bash
# Using nak
nak key generate
```

2. أضفه إلى الإعدادات:

```json5
{
  channels: {
    nostr: {
      privateKey: "${NOSTR_PRIVATE_KEY}",
    },
  },
}
```

3. صدّر المفتاح:

```bash
export NOSTR_PRIVATE_KEY="nsec1..."
```

4. أعد تشغيل Gateway.

## مرجع الإعدادات

| المفتاح          | النوع     | الافتراضي                                 | الوصف                              |
| ------------ | -------- | ------------------------------------------- | ----------------------------------- |
| `privateKey` | string   | مطلوب                                      | المفتاح الخاص بتنسيق `nsec` أو hex |
| `relays`     | string[] | `['wss://relay.damus.io', 'wss://nos.lol']` | عناوين URL للـ relay ‏(WebSocket)   |
| `dmPolicy`   | string   | `pairing`                                   | سياسة الوصول للرسائل المباشرة      |
| `allowFrom`  | string[] | `[]`                                        | مفاتيح pubkeys المسموح بها للمرسلين |
| `enabled`    | boolean  | `true`                                      | تمكين/تعطيل القناة                 |
| `name`       | string   | -                                           | اسم العرض                          |
| `profile`    | object   | -                                           | بيانات تعريف ملف NIP-01            |

## بيانات تعريف الملف الشخصي

يتم نشر بيانات الملف الشخصي كحدث NIP-01 من النوع `kind:0`. يمكنك إدارتها من واجهة Control UI ‏(Channels -> Nostr -> Profile) أو ضبطها مباشرة في الإعدادات.

مثال:

```json5
{
  channels: {
    nostr: {
      privateKey: "${NOSTR_PRIVATE_KEY}",
      profile: {
        name: "openclaw",
        displayName: "OpenClaw",
        about: "Personal assistant DM bot",
        picture: "https://example.com/avatar.png",
        banner: "https://example.com/banner.png",
        website: "https://example.com",
        nip05: "openclaw@example.com",
        lud16: "openclaw@example.com",
      },
    },
  },
}
```

ملاحظات:

- يجب أن تستخدم عناوين URL للملف الشخصي `https://`.
- يؤدي الاستيراد من relays إلى دمج الحقول مع الحفاظ على التجاوزات المحلية.

## التحكم في الوصول

### سياسات الرسائل المباشرة

- **pairing** (الافتراضي): يحصل المرسلون غير المعروفين على رمز اقتران.
- **allowlist**: لا يمكن إلا للمفاتيح العامة الموجودة في `allowFrom` إرسال رسائل مباشرة.
- **open**: رسائل مباشرة عامة واردة (يتطلب `allowFrom: ["*"]`).
- **disabled**: تجاهل الرسائل المباشرة الواردة.

ملاحظات التنفيذ:

- يتم التحقق من تواقيع الأحداث الواردة قبل سياسة المرسل وفك تشفير NIP-04، لذلك تُرفض الأحداث المزورة مبكرًا.
- يتم إرسال ردود الاقتران من دون معالجة نص الرسالة المباشرة الأصلي.
- يتم تطبيق حد للمعدل على الرسائل المباشرة الواردة، ويتم إسقاط الحمولات كبيرة الحجم قبل فك التشفير.

### مثال على قائمة السماح

```json5
{
  channels: {
    nostr: {
      privateKey: "${NOSTR_PRIVATE_KEY}",
      dmPolicy: "allowlist",
      allowFrom: ["npub1abc...", "npub1xyz..."],
    },
  },
}
```

## تنسيقات المفاتيح

التنسيقات المقبولة:

- **المفتاح الخاص:** `nsec...` أو hex من 64 حرفًا
- **المفاتيح العامة (`allowFrom`):** `npub...` أو hex

## Relays

القيم الافتراضية: `relay.damus.io` و`nos.lol`.

```json5
{
  channels: {
    nostr: {
      privateKey: "${NOSTR_PRIVATE_KEY}",
      relays: ["wss://relay.damus.io", "wss://relay.primal.net", "wss://nostr.wine"],
    },
  },
}
```

نصائح:

- استخدم 2 إلى 3 relays للتكرار.
- تجنب عددًا كبيرًا جدًا من relays ‏(زمن الاستجابة، التكرار).
- يمكن أن تحسن relays المدفوعة الموثوقية.
- relays المحلية مناسبة للاختبار (`ws://localhost:7777`).

## دعم البروتوكول

| NIP    | الحالة    | الوصف                                |
| ------ | --------- | ------------------------------------- |
| NIP-01 | مدعوم     | تنسيق الحدث الأساسي + بيانات الملف الشخصي |
| NIP-04 | مدعوم     | الرسائل المباشرة المشفرة (`kind:4`)  |
| NIP-17 | مخطط له   | الرسائل المباشرة المغلفة بهدايا       |
| NIP-44 | مخطط له   | تشفير بإصدارات                       |

## الاختبار

### Relay محلي

```bash
# Start strfry
docker run -p 7777:7777 ghcr.io/hoytech/strfry
```

```json5
{
  channels: {
    nostr: {
      privateKey: "${NOSTR_PRIVATE_KEY}",
      relays: ["ws://localhost:7777"],
    },
  },
}
```

### اختبار يدوي

1. دوّن المفتاح العام الخاص بالبوت (npub) من السجلات.
2. افتح عميل Nostr ‏(Damus أو Amethyst أو غيرهما).
3. أرسل رسالة مباشرة إلى المفتاح العام الخاص بالبوت.
4. تحقق من الاستجابة.

## استكشاف الأخطاء وإصلاحها

### لا يتم استقبال الرسائل

- تحقّق من أن المفتاح الخاص صالح.
- تأكد من إمكانية الوصول إلى عناوين URL الخاصة بـ relay وأنها تستخدم `wss://` ‏(أو `ws://` للمحلي).
- أكّد أن `enabled` ليس `false`.
- تحقّق من سجلات Gateway بحثًا عن أخطاء اتصال relay.

### لا يتم إرسال الردود

- تحقّق من أن relay يقبل عمليات الكتابة.
- تحقّق من الاتصال الصادر.
- راقب حدود المعدل الخاصة بـ relay.

### ردود مكررة

- هذا متوقع عند استخدام relays متعددة.
- تتم إزالة التكرار من الرسائل حسب معرّف الحدث؛ ولا يؤدي إلا أول تسليم إلى تشغيل رد.

## الأمان

- لا تلتزم أبدًا بالمفاتيح الخاصة في المستودع.
- استخدم متغيرات البيئة للمفاتيح.
- فكّر في استخدام `allowlist` لبوتات الإنتاج.
- يتم التحقق من التواقيع قبل سياسة المرسل، ويتم فرض سياسة المرسل قبل فك التشفير، لذلك تُرفض الأحداث المزورة مبكرًا ولا يمكن للمرسلين غير المعروفين فرض عمل تشفيري كامل.

## القيود (MVP)

- الرسائل المباشرة فقط (من دون دردشات جماعية).
- لا توجد مرفقات وسائط.
- NIP-04 فقط (تغليف الهدايا في NIP-17 مخطط له).

## ذو صلة

- [نظرة عامة على القنوات](/channels) — جميع القنوات المدعومة
- [الاقتران](/channels/pairing) — مصادقة الرسائل المباشرة وتدفق الاقتران
- [المجموعات](/channels/groups) — سلوك دردشات المجموعات وبوابة الإشارات
- [توجيه القنوات](/channels/channel-routing) — توجيه الجلسات للرسائل
- [الأمان](/gateway/security) — نموذج الوصول والتقوية
