---
read_when:
    - تريد تشغيل دورة وكيل واحدة من النصوص البرمجية (مع تسليم الرد اختياريًا)
summary: مرجع CLI للأمر `openclaw agent` (تشغيل دورة وكيل واحدة عبر Gateway)
title: agent
x-i18n:
    generated_at: "2026-04-05T12:37:32Z"
    model: gpt-5.4
    provider: openai
    source_hash: 0627f943bc7f3556318008f76dc6150788cf06927dccdc7d2681acb98f257d56
    source_path: cli/agent.md
    workflow: 15
---

# `openclaw agent`

شغّل دورة وكيل عبر Gateway (استخدم `--local` للتشغيل المضمن).
استخدم `--agent <id>` لاستهداف وكيل مكوَّن مباشرةً.

مرّر محدد جلسة واحدًا على الأقل:

- `--to <dest>`
- `--session-id <id>`
- `--agent <id>`

ذو صلة:

- أداة إرسال الوكيل: [Agent send](/tools/agent-send)

## الخيارات

- `-m, --message <text>`: نص الرسالة المطلوب
- `-t, --to <dest>`: المستلم المستخدم لاشتقاق مفتاح الجلسة
- `--session-id <id>`: معرّف جلسة صريح
- `--agent <id>`: معرّف الوكيل؛ يتجاوز روابط التوجيه
- `--thinking <off|minimal|low|medium|high|xhigh>`: مستوى تفكير الوكيل
- `--verbose <on|off>`: حفظ مستوى التفصيل للجلسة
- `--channel <channel>`: قناة التسليم؛ احذفه لاستخدام قناة الجلسة الرئيسية
- `--reply-to <target>`: تجاوز هدف التسليم
- `--reply-channel <channel>`: تجاوز قناة التسليم
- `--reply-account <id>`: تجاوز حساب التسليم
- `--local`: شغّل الوكيل المضمن مباشرةً (بعد التحميل المسبق لسجل plugins)
- `--deliver`: أرسل الرد مرة أخرى إلى القناة/الهدف المحدد
- `--timeout <seconds>`: تجاوز مهلة الوكيل (الافتراضي 600 أو قيمة التكوين)
- `--json`: إخراج JSON

## أمثلة

```bash
openclaw agent --to +15555550123 --message "status update" --deliver
openclaw agent --agent ops --message "Summarize logs"
openclaw agent --session-id 1234 --message "Summarize inbox" --thinking medium
openclaw agent --to +15555550123 --message "Trace logs" --verbose on --json
openclaw agent --agent ops --message "Generate report" --deliver --reply-channel slack --reply-to "#reports"
openclaw agent --agent ops --message "Run locally" --local
```

## ملاحظات

- يعود وضع Gateway إلى الوكيل المضمن عندما يفشل طلب Gateway. استخدم `--local` لفرض التنفيذ المضمن من البداية.
- ما يزال `--local` يحمّل سجل plugins مسبقًا أولًا، لذلك تبقى providers والأدوات والقنوات التي توفرها plugins متاحة أثناء التشغيل المضمن.
- تؤثر `--channel` و`--reply-channel` و`--reply-account` في تسليم الرد، وليس في توجيه الجلسة.
- عندما يؤدي هذا الأمر إلى إعادة إنشاء `models.json`، يتم حفظ بيانات اعتماد المزوّد التي تديرها SecretRef كعلامات غير سرية (مثل أسماء متغيرات البيئة أو `secretref-env:ENV_VAR_NAME` أو `secretref-managed`) وليس كنصوص أسرار صريحة محلولة.
- تكون عمليات كتابة العلامات معتمدة على المصدر: يحفظ OpenClaw العلامات من لقطة التكوين المصدر النشطة، وليس من قيم الأسرار المحلولة وقت التشغيل.
