---
read_when:
    - أنت تنشئ Skill مخصصة جديدة في مساحة العمل الخاصة بك
    - تحتاج إلى سير عمل تمهيدي سريع لمهارات قائمة على SKILL.md
summary: إنشاء Skills مخصصة لمساحة العمل واختبارها باستخدام SKILL.md
title: إنشاء Skills
x-i18n:
    generated_at: "2026-04-05T12:57:57Z"
    model: gpt-5.4
    provider: openai
    source_hash: 747cebc5191b96311d1d6760bede1785a099acd7633a0b88de6b7882b57e1db6
    source_path: tools/creating-skills.md
    workflow: 15
---

# إنشاء Skills

تُعلّم Skills العامل كيفية استخدام الأدوات ومتى يستخدمها. كل Skill عبارة عن دليل
يحتوي على ملف `SKILL.md` مع YAML frontmatter وتعليمات بصيغة markdown.

للاطلاع على كيفية تحميل Skills وترتيب أولوياتها، راجع [Skills](/tools/skills).

## أنشئ أول Skill لك

<Steps>
  <Step title="أنشئ دليل Skill">
    توجد Skills في مساحة العمل الخاصة بك. أنشئ مجلدًا جديدًا:

    ```bash
    mkdir -p ~/.openclaw/workspace/skills/hello-world
    ```

  </Step>

  <Step title="اكتب SKILL.md">
    أنشئ `SKILL.md` داخل ذلك الدليل. يحدد frontmatter البيانات الوصفية،
    بينما يحتوي متن markdown على تعليمات للعامل.

    ```markdown
    ---
    name: hello_world
    description: A simple skill that says hello.
    ---

    # Hello World Skill

    When the user asks for a greeting, use the `echo` tool to say
    "Hello from your custom skill!".
    ```

  </Step>

  <Step title="أضف أدوات (اختياري)">
    يمكنك تعريف مخططات أدوات مخصصة في frontmatter أو توجيه العامل
    لاستخدام أدوات النظام الموجودة (مثل `exec` أو `browser`). كما يمكن لـ Skills
    أن تأتي ضمن plugins إلى جانب الأدوات التي توثقها.

  </Step>

  <Step title="حمّل Skill">
    ابدأ جلسة جديدة حتى يلتقط OpenClaw الـ Skill:

    ```bash
    # From chat
    /new

    # Or restart the gateway
    openclaw gateway restart
    ```

    تحقّق من تحميل الـ Skill:

    ```bash
    openclaw skills list
    ```

  </Step>

  <Step title="اختبرها">
    أرسل رسالة يفترض أن تؤدي إلى تشغيل الـ Skill:

    ```bash
    openclaw agent --message "give me a greeting"
    ```

    أو ببساطة تحدث مع العامل واطلب تحية.

  </Step>
</Steps>

## مرجع البيانات الوصفية لـ Skill

يدعم YAML frontmatter هذه الحقول:

| الحقل | مطلوب | الوصف |
| ----------------------------------- | -------- | ------------------------------------------- |
| `name` | نعم | معرّف فريد (`snake_case`) |
| `description` | نعم | وصف من سطر واحد يظهر للعامل |
| `metadata.openclaw.os` | لا | عامل تصفية لنظام التشغيل (`["darwin"]`, `["linux"]`، إلخ) |
| `metadata.openclaw.requires.bins` | لا | الملفات الثنائية المطلوبة على PATH |
| `metadata.openclaw.requires.config` | لا | مفاتيح التهيئة المطلوبة |

## أفضل الممارسات

- **كن موجزًا** — وجّه النموذج إلى _ما_ يجب فعله، لا إلى كيفية أن يكون AI
- **السلامة أولًا** — إذا كانت Skill الخاصة بك تستخدم `exec`، فتأكد من أن الموجهات لا تسمح بحقن أوامر عشوائية من مدخلات غير موثوقة
- **اختبر محليًا** — استخدم `openclaw agent --message "..."` للاختبار قبل المشاركة
- **استخدم ClawHub** — تصفح Skills وساهم بها على [ClawHub](https://clawhub.ai)

## أماكن وجود Skills

| الموقع | الأولوية | النطاق |
| ------------------------------- | ---------- | --------------------- |
| `\<workspace\>/skills/` | الأعلى | لكل عامل |
| `\<workspace\>/.agents/skills/` | عالية | لكل عامل في مساحة العمل |
| `~/.agents/skills/` | متوسطة | ملف تعريف عامل مشترك |
| `~/.openclaw/skills/` | متوسطة | مشتركة (كل العوامل) |
| مضمّنة (تأتي مع OpenClaw) | منخفضة | عامة |
| `skills.load.extraDirs` | الأدنى | مجلدات مشتركة مخصصة |

## ذو صلة

- [مرجع Skills](/tools/skills) — قواعد التحميل، والأولوية، والتقييد
- [تهيئة Skills](/tools/skills-config) — مخطط تهيئة `skills.*`
- [ClawHub](/tools/clawhub) — سجل Skills عام
- [بناء Plugins](/ar/plugins/building-plugins) — يمكن أن تأتي plugins مع Skills
