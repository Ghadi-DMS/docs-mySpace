---
read_when:
    - تريد تشغيل أو كتابة سير عمل `.prose`
    - تريد تمكين plugin الخاصة بـ OpenProse
    - تحتاج إلى فهم تخزين الحالة
summary: 'OpenProse: سير عمل `.prose`، وأوامر الشرطة المائلة، والحالة في OpenClaw'
title: OpenProse
x-i18n:
    generated_at: "2026-04-05T12:52:28Z"
    model: gpt-5.4
    provider: openai
    source_hash: 95f86ed3029c5599b6a6bed1f75b2e10c8808cf7ffa5e33dbfb1801a7f65f405
    source_path: prose.md
    workflow: 15
---

# OpenProse

OpenProse هو تنسيق سير عمل قابل للنقل ويركّز على markdown لتنظيم جلسات الذكاء الاصطناعي. وفي OpenClaw يأتي على هيئة plugin تثبّت حزمة Skills خاصة بـ OpenProse بالإضافة إلى أمر الشرطة المائلة `/prose`. تعيش البرامج في ملفات `.prose` ويمكنها تشغيل عدة وكلاء فرعيين مع تدفق تحكم صريح.

الموقع الرسمي: [https://www.prose.md](https://www.prose.md)

## ما الذي يمكنه فعله

- بحث وتجميع متعدد الوكلاء مع توازٍ صريح.
- سير عمل قابل للتكرار وآمن من حيث الموافقات (مراجعة الشيفرة، وفرز الحوادث، ومسارات المحتوى).
- برامج `.prose` قابلة لإعادة الاستخدام يمكنك تشغيلها عبر أوقات تشغيل الوكلاء المدعومة.

## التثبيت + التمكين

تكون plugins المضمّنة معطلة افتراضيًا. فعّل OpenProse:

```bash
openclaw plugins enable open-prose
```

أعد تشغيل Gateway بعد تمكين plugin.

نسخة تطوير/محلية: ‏`openclaw plugins install ./path/to/local/open-prose-plugin`

وثائق ذات صلة: [Plugins](/tools/plugin)، [Plugin manifest](/plugins/manifest)، [Skills](/tools/skills).

## أمر الشرطة المائلة

تسجل OpenProse الأمر `/prose` كأمر Skill يمكن للمستخدم استدعاؤه. ويُوجَّه إلى تعليمات OpenProse VM ويستخدم أدوات OpenClaw تحت الغطاء.

الأوامر الشائعة:

```
/prose help
/prose run <file.prose>
/prose run <handle/slug>
/prose run <https://example.com/file.prose>
/prose compile <file.prose>
/prose examples
/prose update
```

## مثال: ملف `.prose` بسيط

```prose
# Research + synthesis with two agents running in parallel.

input topic: "What should we research?"

agent researcher:
  model: sonnet
  prompt: "You research thoroughly and cite sources."

agent writer:
  model: opus
  prompt: "You write a concise summary."

parallel:
  findings = session: researcher
    prompt: "Research {topic}."
  draft = session: writer
    prompt: "Summarize {topic}."

session "Merge the findings + draft into a final answer."
context: { findings, draft }
```

## مواقع الملفات

تحتفظ OpenProse بالحالة تحت `.prose/` في مساحة العمل الخاصة بك:

```
.prose/
├── .env
├── runs/
│   └── {YYYYMMDD}-{HHMMSS}-{random}/
│       ├── program.prose
│       ├── state.md
│       ├── bindings/
│       └── agents/
└── agents/
```

تعيش الوكلاء الدائمون على مستوى المستخدم في:

```
~/.prose/agents/
```

## أوضاع الحالة

تدعم OpenProse عدة خلفيات للحالة:

- **filesystem** (الافتراضي): ‏`.prose/runs/...`
- **in-context**: مؤقتة، للبرامج الصغيرة
- **sqlite** (تجريبي): يتطلب الثنائي `sqlite3`
- **postgres** (تجريبي): يتطلب `psql` وسلسلة اتصال

ملاحظات:

- sqlite/postgres اختيارية وتجريبية.
- تنتقل بيانات اعتماد postgres إلى سجلات الوكلاء الفرعيين؛ استخدم قاعدة بيانات مخصصة بأقل قدر من الامتيازات.

## البرامج البعيدة

تُحل `/prose run <handle/slug>` إلى `https://p.prose.md/<handle>/<slug>`.
أما عناوين URL المباشرة فتُجلب كما هي. ويستخدم هذا أداة `web_fetch` ‏(أو `exec` لطلبات POST).

## ربط وقت تشغيل OpenClaw

تُربط برامج OpenProse بعناصر OpenClaw الأساسية:

| مفهوم OpenProse             | أداة OpenClaw   |
| --------------------------- | --------------- |
| تشغيل session / أداة Task   | `sessions_spawn` |
| قراءة/كتابة الملفات         | `read` / `write` |
| جلب الويب                   | `web_fetch`      |

إذا كانت قائمة السماح الخاصة بالأدوات لديك تحظر هذه الأدوات، فستفشل برامج OpenProse. راجع [Skills config](/tools/skills-config).

## الأمان + الموافقات

تعامل مع ملفات `.prose` كما تتعامل مع الشيفرة. راجعها قبل التشغيل. واستخدم قوائم السماح للأدوات في OpenClaw وبوابات الموافقة للتحكم في الآثار الجانبية.

وبالنسبة إلى سير العمل الحتمي والمقيّد بالموافقات، قارنه مع [Lobster](/tools/lobster).
