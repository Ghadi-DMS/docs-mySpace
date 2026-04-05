---
read_when:
    - إضافة تكاملات CLI خارجية أو تغييرها
    - تصحيح أخطاء مهايئات RPC ‏(signal-cli وimsg)
summary: مهايئات RPC لواجهات CLI الخارجية (signal-cli وimsg القديم) وأنماط البوابة
title: مهايئات RPC
x-i18n:
    generated_at: "2026-04-05T12:54:38Z"
    model: gpt-5.4
    provider: openai
    source_hash: 06dc6b97184cc704ba4ec4a9af90502f4316bcf717c3f4925676806d8b184c57
    source_path: reference/rpc.md
    workflow: 15
---

# مهايئات RPC

يدمج OpenClaw واجهات CLI الخارجية عبر JSON-RPC. ويتم استخدام نمطين اليوم.

## النمط A: daemon عبر HTTP ‏(signal-cli)

- يعمل `signal-cli` كخدمة daemon مع JSON-RPC عبر HTTP.
- يكون تدفق الأحداث عبر SSE ‏(`/api/v1/events`).
- فحص الصحة: `/api/v1/check`.
- يتولى OpenClaw دورة الحياة عندما تكون `channels.signal.autoStart=true`.

راجع [Signal](/channels/signal) لمعرفة الإعداد ونقاط النهاية.

## النمط B: عملية فرعية عبر stdio ‏(قديم: imsg)

> **ملاحظة:** بالنسبة إلى إعدادات iMessage الجديدة، استخدم [BlueBubbles](/channels/bluebubbles) بدلًا من ذلك.

- يقوم OpenClaw بتشغيل `imsg rpc` كعملية فرعية (تكامل iMessage القديم).
- يكون JSON-RPC محددًا سطرًا بسطر عبر stdin/stdout (كائن JSON واحد لكل سطر).
- لا يوجد منفذ TCP ولا حاجة إلى daemon.

الأساليب الأساسية المستخدمة:

- `watch.subscribe` → إشعارات (`method: "message"`)
- `watch.unsubscribe`
- `send`
- `chats.list` ‏(للفحص/التشخيص)

راجع [iMessage](/channels/imessage) لمعرفة الإعداد القديم والعنونة (ويُفضَّل `chat_id`).

## إرشادات المهايئ

- تتولى البوابة العملية (يرتبط البدء/الإيقاف بدورة حياة المزوّد).
- اجعل عملاء RPC قادرين على الصمود: مهلات زمنية، وإعادة تشغيل عند الإنهاء.
- فضّل المعرّفات المستقرة (مثل `chat_id`) على سلاسل العرض.
