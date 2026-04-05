---
read_when:
    - تريد البحث عن معرّفات جهات الاتصال/المجموعات/المعرّف الذاتي لقناة ما
    - أنت تطور مهايئ دليل لقناة
summary: مرجع CLI للأمر `openclaw directory` (الذات، والأقران، والمجموعات)
title: directory
x-i18n:
    generated_at: "2026-04-05T12:38:08Z"
    model: gpt-5.4
    provider: openai
    source_hash: 6a81a037e0a33f77c24b1adabbc4be16ed4d03c419873f3cbdd63f2ce84a1064
    source_path: cli/directory.md
    workflow: 15
---

# `openclaw directory`

عمليات بحث الدليل للقنوات التي تدعم ذلك (جهات الاتصال/الأقران، والمجموعات، و"أنا").

## الوسائط الشائعة

- `--channel <name>`: معرّف/اسم مستعار للقناة (مطلوب عند تهيئة عدة قنوات؛ ويُستنتج تلقائيًا عند تهيئة قناة واحدة فقط)
- `--account <id>`: معرّف الحساب (الافتراضي: الحساب الافتراضي للقناة)
- `--json`: إخراج JSON

## ملاحظات

- الغرض من `directory` هو مساعدتك في العثور على المعرّفات التي يمكنك لصقها في أوامر أخرى (وخاصة `openclaw message send --target ...`).
- بالنسبة إلى كثير من القنوات، تكون النتائج مدعومة بالإعداد (قوائم السماح / المجموعات المهيأة) بدلًا من دليل موفّر حي.
- يكون الإخراج الافتراضي هو `id` (وأحيانًا `name`) مفصولًا بعلامة جدولة؛ استخدم `--json` في البرامج النصية.

## استخدام النتائج مع `message send`

```bash
openclaw directory peers list --channel slack --query "U0"
openclaw message send --channel slack --target user:U012ABCDEF --message "hello"
```

## صيغ المعرّفات (حسب القناة)

- WhatsApp: ‏`+15551234567` (رسالة مباشرة)، ‏`1234567890-1234567890@g.us` (مجموعة)
- Telegram: ‏`@username` أو معرّف دردشة رقمي؛ المجموعات هي معرّفات رقمية
- Slack: ‏`user:U…` و`channel:C…`
- Discord: ‏`user:<id>` و`channel:<id>`
- Matrix (إضافة): ‏`user:@user:server` أو `room:!roomId:server` أو `#alias:server`
- Microsoft Teams (إضافة): ‏`user:<id>` و`conversation:<id>`
- Zalo (إضافة): معرّف المستخدم (Bot API)
- Zalo Personal / `zalouser` (إضافة): معرّف سلسلة الرسائل (رسالة مباشرة/مجموعة) من `zca` (`me` و`friend list` و`group list`)

## الذات ("أنا")

```bash
openclaw directory self --channel zalouser
```

## الأقران (جهات الاتصال/المستخدمون)

```bash
openclaw directory peers list --channel zalouser
openclaw directory peers list --channel zalouser --query "name"
openclaw directory peers list --channel zalouser --limit 50
```

## المجموعات

```bash
openclaw directory groups list --channel zalouser
openclaw directory groups list --channel zalouser --query "work"
openclaw directory groups members --channel zalouser --group-id <id>
```
