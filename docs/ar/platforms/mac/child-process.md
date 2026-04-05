---
read_when:
    - دمج تطبيق mac مع دورة حياة gateway
summary: دورة حياة Gateway على macOS ‏(launchd)
title: دورة حياة Gateway
x-i18n:
    generated_at: "2026-04-05T12:49:55Z"
    model: gpt-5.4
    provider: openai
    source_hash: 73e7eb64ef432c3bfc81b949a5cc2a344c64f2310b794228609aae1da817ec41
    source_path: platforms/mac/child-process.md
    workflow: 15
---

# دورة حياة Gateway على macOS

يقوم تطبيق macOS **بإدارة Gateway عبر launchd** افتراضيًا ولا يشغّل
Gateway كعملية فرعية. فهو يحاول أولًا الارتباط بـ Gateway تعمل بالفعل
على المنفذ المكوَّن؛ وإذا لم يجد أي Gateway قابلة للوصول، فإنه يفعّل خدمة launchd
عبر CLI الخارجي الخاص بـ `openclaw` (من دون runtime مضمّن). وهذا يمنحك
بدءًا تلقائيًا موثوقًا عند تسجيل الدخول وإعادة تشغيل عند الأعطال.

وضع العملية الفرعية (تشغيل Gateway مباشرةً من التطبيق) **غير مستخدم** اليوم.
إذا كنت تحتاج إلى اقتران أوثق بواجهة المستخدم، فشغّل Gateway يدويًا في طرفية.

## السلوك الافتراضي (launchd)

- يثبت التطبيق LaunchAgent لكل مستخدم بالاسم `ai.openclaw.gateway`
  (أو `ai.openclaw.<profile>` عند استخدام `--profile`/`OPENCLAW_PROFILE`؛ مع دعم legacy `com.openclaw.*`).
- عند تمكين الوضع المحلي، يضمن التطبيق تحميل LaunchAgent
  ويبدأ Gateway عند الحاجة.
- تُكتب السجلات إلى مسار سجل gateway الخاص بـ launchd (وهو مرئي في Debug Settings).

أوامر شائعة:

```bash
launchctl kickstart -k gui/$UID/ai.openclaw.gateway
launchctl bootout gui/$UID/ai.openclaw.gateway
```

استبدل التسمية بـ `ai.openclaw.<profile>` عند تشغيل ملف شخصي مسمى.

## إصدارات التطوير غير الموقعة

يُستخدم `scripts/restart-mac.sh --no-sign` للبناءات المحلية السريعة عندما لا تكون لديك
مفاتيح توقيع. ولمنع launchd من الإشارة إلى relay binary غير موقّع، فإنه:

- يكتب `~/.openclaw/disable-launchagent`.

تمسح عمليات التشغيل الموقعة الخاصة بـ `scripts/restart-mac.sh` هذا التجاوز إذا كانت العلامة
موجودة. ولإعادة الضبط يدويًا:

```bash
rm ~/.openclaw/disable-launchagent
```

## وضع الارتباط فقط

لإجبار تطبيق macOS على **عدم تثبيت launchd أو إدارتها أبدًا**، شغّله باستخدام
`--attach-only` (أو `--no-launchd`). وهذا يضبط `~/.openclaw/disable-launchagent`,
بحيث لا يقوم التطبيق إلا بالارتباط بـ Gateway تعمل بالفعل. ويمكنك تبديل
السلوك نفسه في Debug Settings.

## الوضع البعيد

لا يبدأ الوضع البعيد أبدًا Gateway محلية. يستخدم التطبيق نفق SSH إلى
المضيف البعيد ويتصل عبر ذلك النفق.

## لماذا نفضل launchd

- بدء تلقائي عند تسجيل الدخول.
- دلالات إعادة تشغيل/‏KeepAlive مضمّنة.
- سجلات وإشراف يمكن التنبؤ بهما.

إذا كانت هناك حاجة فعلًا مرة أخرى إلى وضع عملية فرعية حقيقي، فيجب توثيقه
كوضع منفصل وصريح ومخصص للتطوير فقط.
