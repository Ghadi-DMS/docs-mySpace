---
read_when:
    - تحديث واجهة إعدادات Skills في macOS
    - تغيير بوابات Skills أو سلوك التثبيت
summary: واجهة إعدادات Skills في macOS والحالة المدعومة من gateway
title: Skills (macOS)
x-i18n:
    generated_at: "2026-04-05T12:50:17Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7ffd6744646d2c8770fa12a5e511f84a40b5ece67181139250ec4cc4301b49b8
    source_path: platforms/mac/skills.md
    workflow: 15
---

# Skills (macOS)

يعرض تطبيق macOS مهارات OpenClaw عبر gateway؛ ولا يقوم بتحليل Skills محليًا.

## مصدر البيانات

- يعيد `skills.status` ‏(في gateway) جميع Skills بالإضافة إلى الأهلية والمتطلبات المفقودة
  (بما في ذلك حظر قائمة السماح لـ Skills المضمنة).
- تُشتق المتطلبات من `metadata.openclaw.requires` في كل `SKILL.md`.

## إجراءات التثبيت

- يعرّف `metadata.openclaw.install` خيارات التثبيت (`brew`/`node`/`go`/`uv`).
- يستدعي التطبيق `skills.install` لتشغيل المُثبتات على مضيف gateway.
- تمنع نتائج `critical` المضمنة الخاصة بـ dangerous-code تنفيذ `skills.install` افتراضيًا؛ أما النتائج المشبوهة فتظل تحذيرية فقط. ويوجد تجاوز الخطر على طلب gateway، لكن تدفق التطبيق الافتراضي يبقى مغلقًا عند الفشل.
- إذا كانت كل خيارات التثبيت هي `download`، فإن gateway يعرض جميع خيارات
  التنزيل.
- بخلاف ذلك، يختار gateway مثبتًا مفضلاً واحدًا باستخدام تفضيلات
  التثبيت الحالية والملفات التنفيذية الموجودة على المضيف: تكون الأولوية لـ Homebrew عندما
  يكون `skills.install.preferBrew` مفعّلًا ويكون `brew` موجودًا، ثم `uv`، ثم
  مدير node المهيأ من `skills.install.nodeManager`، ثم
  التراجعات اللاحقة مثل `go` أو `download`.
- تعكس تسميات تثبيت Node مدير node المهيأ، بما في ذلك `yarn`.

## مفاتيح env/API

- يخزن التطبيق المفاتيح في `~/.openclaw/openclaw.json` تحت `skills.entries.<skillKey>`.
- يقوم `skills.update` بترقيع `enabled` و`apiKey` و`env`.

## الوضع البعيد

- تحدث تحديثات التثبيت + الإعدادات على مضيف gateway ‏(وليس على جهاز Mac المحلي).
