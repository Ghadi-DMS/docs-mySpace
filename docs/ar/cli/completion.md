---
read_when:
    - أنت تريد إكمالات للصدفة zsh/bash/fish/PowerShell
    - تحتاج إلى تخزين برامج الإكمال النصية مؤقتًا ضمن حالة OpenClaw
summary: مرجع CLI للأمر `openclaw completion` (إنشاء/تثبيت برامج الإكمال النصية للصدفة)
title: completion
x-i18n:
    generated_at: "2026-04-05T12:37:57Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7bbf140a880bafdb7140149f85465d66d0d46e5a3da6a1e41fb78be2fd2bd4d0
    source_path: cli/completion.md
    workflow: 15
---

# `openclaw completion`

أنشئ برامج الإكمال النصية للصدفة، ويمكنك اختياريًا تثبيتها في ملف تعريف الصدفة الخاص بك.

## الاستخدام

```bash
openclaw completion
openclaw completion --shell zsh
openclaw completion --install
openclaw completion --shell fish --install
openclaw completion --write-state
openclaw completion --shell bash --write-state
```

## الخيارات

- `-s, --shell <shell>`: هدف الصدفة (`zsh` أو `bash` أو `powershell` أو `fish`؛ الافتراضي: `zsh`)
- `-i, --install`: تثبيت الإكمال عبر إضافة سطر source إلى ملف تعريف الصدفة الخاص بك
- `--write-state`: كتابة برنامج/برامج الإكمال النصية إلى `$OPENCLAW_STATE_DIR/completions` دون الطباعة إلى stdout
- `-y, --yes`: تخطي مطالبات تأكيد التثبيت

## ملاحظات

- يكتب `--install` كتلة صغيرة باسم "OpenClaw Completion" في ملف تعريف الصدفة الخاص بك ويوجهها إلى البرنامج النصي المخزن مؤقتًا.
- من دون `--install` أو `--write-state`، يطبع الأمر البرنامج النصي إلى stdout.
- يقوم إنشاء الإكمال بتحميل أشجار الأوامر مسبقًا بحيث يتم تضمين الأوامر الفرعية المتداخلة.
