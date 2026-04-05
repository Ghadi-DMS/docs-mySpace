---
read_when:
    - تريد واجهة طرفية لـ Gateway (ملائمة للاستخدام عن بُعد)
    - تريد تمرير url/token/session من النصوص البرمجية
summary: مرجع CLI للأمر `openclaw tui` (واجهة طرفية متصلة بـ Gateway)
title: tui
x-i18n:
    generated_at: "2026-04-05T12:39:30Z"
    model: gpt-5.4
    provider: openai
    source_hash: 60e35062c0551f85ce0da604a915b3e1ca2514d00d840afe3b94c529304c2c1a
    source_path: cli/tui.md
    workflow: 15
---

# `openclaw tui`

افتح الواجهة الطرفية المتصلة بـ Gateway.

ذو صلة:

- دليل TUI: [TUI](/web/tui)

ملاحظات:

- يقوم `tui` بحل SecretRefs الخاصة بمصادقة gateway المكوّنة لمصادقة token/password عند الإمكان (موفرو `env`/`file`/`exec`).
- عند تشغيله من داخل دليل مساحة عمل وكيل مُعد، يختار TUI ذلك الوكيل تلقائيًا كمفتاح جلسة افتراضي (ما لم يكن `--session` مضبوطًا صراحةً على `agent:<id>:...`).

## أمثلة

```bash
openclaw tui
openclaw tui --url ws://127.0.0.1:18789 --token <token>
openclaw tui --session main --deliver
# عند تشغيله داخل مساحة عمل وكيل، يستنتج ذلك الوكيل تلقائيًا
openclaw tui --session bugfix
```
