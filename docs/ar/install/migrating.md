---
read_when:
    - أنت تنقل OpenClaw إلى حاسوب محمول/خادم جديد
    - تريد الحفاظ على الجلسات والمصادقة وعمليات تسجيل دخول القنوات (WhatsApp، إلخ)
summary: نقل (ترحيل) تثبيت OpenClaw من جهاز إلى آخر
title: دليل الترحيل
x-i18n:
    generated_at: "2026-04-05T12:47:43Z"
    model: gpt-5.4
    provider: openai
    source_hash: 403f0b9677ce723c84abdbabfad20e0f70fd48392ebf23eabb7f8a111fd6a26d
    source_path: install/migrating.md
    workflow: 15
---

# ترحيل OpenClaw إلى جهاز جديد

ينقل هذا الدليل gateway الخاصة بـ OpenClaw إلى جهاز جديد من دون إعادة تنفيذ التهيئة الأولية.

## ما الذي يتم ترحيله

عندما تنسخ **دليل الحالة** (`~/.openclaw/` افتراضيًا) و**مساحة العمل** الخاصة بك، فإنك تحتفظ بما يلي:

- **التكوين** -- `openclaw.json` وجميع إعدادات gateway
- **المصادقة** -- ملفات `auth-profiles.json` لكل وكيل (مفاتيح API + OAuth)، بالإضافة إلى أي حالة قنوات/مزوّدات تحت `credentials/`
- **الجلسات** -- سجل المحادثة وحالة الوكيل
- **حالة القناة** -- تسجيل دخول WhatsApp، وجلسة Telegram، وغير ذلك
- **ملفات مساحة العمل** -- `MEMORY.md` و`USER.md` وSkills والمطالبات

<Tip>
شغّل `openclaw status` على الجهاز القديم لتأكيد مسار دليل الحالة.
تستخدم profiles المخصصة المسار `~/.openclaw-<profile>/` أو مسارًا مضبوطًا عبر `OPENCLAW_STATE_DIR`.
</Tip>

## خطوات الترحيل

<Steps>
  <Step title="أوقف gateway وخذ نسخة احتياطية">
    على الجهاز **القديم**، أوقف gateway حتى لا تتغير الملفات أثناء النسخ، ثم أنشئ أرشيفًا:

    ```bash
    openclaw gateway stop
    cd ~
    tar -czf openclaw-state.tgz .openclaw
    ```

    إذا كنت تستخدم profiles متعددة (مثل `~/.openclaw-work`)، فأرشف كل واحدة على حدة.

  </Step>

  <Step title="ثبّت OpenClaw على الجهاز الجديد">
    [ثبّت](/install) CLI (وNode إذا لزم) على الجهاز الجديد.
    لا بأس إذا أنشأت التهيئة الأولية `~/.openclaw/` جديدة — فسوف تستبدلها بعد ذلك.
  </Step>

  <Step title="انسخ دليل الحالة ومساحة العمل">
    انقل الأرشيف عبر `scp` أو `rsync -a` أو قرص خارجي، ثم استخرجه:

    ```bash
    cd ~
    tar -xzf openclaw-state.tgz
    ```

    تأكد من تضمين الأدلة المخفية ومن أن ملكية الملفات تطابق المستخدم الذي سيشغّل gateway.

  </Step>

  <Step title="شغّل doctor وتحقق">
    على الجهاز الجديد، شغّل [Doctor](/gateway/doctor) لتطبيق عمليات ترحيل التكوين وإصلاح الخدمات:

    ```bash
    openclaw doctor
    openclaw gateway restart
    openclaw status
    ```

  </Step>
</Steps>

## المشكلات الشائعة

<AccordionGroup>
  <Accordion title="عدم تطابق profile أو state-dir">
    إذا كانت gateway القديمة تستخدم `--profile` أو `OPENCLAW_STATE_DIR` بينما لا تستخدم الجديدة ذلك،
    فستبدو القنوات وكأنها مسجّلة الخروج وستكون الجلسات فارغة.
    شغّل gateway باستخدام **profile أو state-dir نفسها** التي قمت بترحيلها، ثم أعد تشغيل `openclaw doctor`.
  </Accordion>

  <Accordion title="نسخ openclaw.json فقط">
    ملف التكوين وحده لا يكفي. تعيش ملفات تعريف مصادقة النموذج تحت
    `agents/<agentId>/agent/auth-profiles.json`، كما أن حالة القناة/المزوّد ما زالت
    تعيش تحت `credentials/`. احرص دائمًا على ترحيل **دليل الحالة بالكامل**.
  </Accordion>

  <Accordion title="الأذونات والملكية">
    إذا نسخت كـ root أو بدّلت المستخدمين، فقد تفشل gateway في قراءة بيانات الاعتماد.
    تأكد من أن دليل الحالة ومساحة العمل مملوكان للمستخدم الذي يشغّل gateway.
  </Accordion>

  <Accordion title="الوضع البعيد">
    إذا كانت واجهة المستخدم لديك تشير إلى gateway **بعيدة**، فإن المضيف البعيد هو الذي يملك الجلسات ومساحة العمل.
    رحّل مضيف gateway نفسه، وليس حاسوبك المحمول المحلي. راجع [الأسئلة الشائعة](/help/faq#where-things-live-on-disk).
  </Accordion>

  <Accordion title="الأسرار في النسخ الاحتياطية">
    يحتوي دليل الحالة على ملفات تعريف المصادقة وبيانات اعتماد القنوات وحالة
    المزوّدات الأخرى.
    خزّن النسخ الاحتياطية بشكل مشفّر، وتجنب قنوات النقل غير الآمنة، ودوّر المفاتيح إذا شككت في حدوث تعرّض.
  </Accordion>
</AccordionGroup>

## قائمة التحقق من التحقق

على الجهاز الجديد، تأكد من:

- [ ] أن `openclaw status` يظهر أن gateway تعمل
- [ ] أن القنوات ما تزال متصلة (ولا حاجة إلى إعادة pairing)
- [ ] أن dashboard تفتح وتعرض الجلسات الموجودة
- [ ] أن ملفات مساحة العمل (الذاكرة، والتكوينات) موجودة
