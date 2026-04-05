---
read_when:
    - أنت تستخدم المكون الإضافي للمكالمات الصوتية وتريد نقاط دخول CLI
    - تريد أمثلة سريعة للأوامر `voicecall call|continue|status|tail|expose`
summary: مرجع CLI للأمر `openclaw voicecall` (سطح أوامر المكون الإضافي للمكالمات الصوتية)
title: voicecall
x-i18n:
    generated_at: "2026-04-05T12:39:34Z"
    model: gpt-5.4
    provider: openai
    source_hash: 2c99e7a3d256e1c74a0f07faba9675cc5a88b1eb2fc6e22993caf3874d4f340a
    source_path: cli/voicecall.md
    workflow: 15
---

# `openclaw voicecall`

الأمر `voicecall` هو أمر يوفّره مكون إضافي. ولا يظهر إلا إذا كان المكون الإضافي للمكالمات الصوتية مثبتًا ومفعّلًا.

الوثيقة الأساسية:

- المكون الإضافي للمكالمات الصوتية: [Voice Call](/plugins/voice-call)

## الأوامر الشائعة

```bash
openclaw voicecall status --call-id <id>
openclaw voicecall call --to "+15555550123" --message "Hello" --mode notify
openclaw voicecall continue --call-id <id> --message "Any questions?"
openclaw voicecall end --call-id <id>
```

## كشف webhooks ‏(Tailscale)

```bash
openclaw voicecall expose --mode serve
openclaw voicecall expose --mode funnel
openclaw voicecall expose --mode off
```

ملاحظة أمنية: اكشف نقطة نهاية webhook فقط للشبكات التي تثق بها. ويفضَّل استخدام Tailscale Serve بدلًا من Funnel عندما يكون ذلك ممكنًا.
