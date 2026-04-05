---
read_when:
    - أنت تريد عمليات تثبيت قابلة لإعادة الإنتاج وقابلة للتراجع
    - أنت تستخدم بالفعل Nix/NixOS/Home Manager
    - أنت تريد أن يكون كل شيء مثبتًا ومُدارًا بشكل تصريحي
summary: ثبّت OpenClaw بشكل تصريحي باستخدام Nix
title: Nix
x-i18n:
    generated_at: "2026-04-05T12:48:04Z"
    model: gpt-5.4
    provider: openai
    source_hash: 14e1e73533db1350d82d3a786092b4328121a082dfeeedee7c7574021dada546
    source_path: install/nix.md
    workflow: 15
---

# تثبيت Nix

ثبّت OpenClaw بشكل تصريحي باستخدام **[nix-openclaw](https://github.com/openclaw/nix-openclaw)** -- وهي وحدة Home Manager متكاملة.

<Info>
يُعد مستودع [nix-openclaw](https://github.com/openclaw/nix-openclaw) المصدر المرجعي لتثبيت Nix. وهذه الصفحة نظرة عامة سريعة.
</Info>

## ما الذي ستحصل عليه

- Gateway + تطبيق macOS + الأدوات (whisper وspotify والكاميرات) -- كلها مثبتة على إصدارات محددة
- خدمة Launchd تستمر بعد إعادة التشغيل
- نظام Plugins مع إعدادات تصريحية
- تراجع فوري: `home-manager switch --rollback`

## البدء السريع

<Steps>
  <Step title="تثبيت Determinate Nix">
    إذا لم يكن Nix مثبتًا بالفعل، فاتبع تعليمات [مثبّت Determinate Nix](https://github.com/DeterminateSystems/nix-installer).
  </Step>
  <Step title="إنشاء flake محلي">
    استخدم القالب agent-first من مستودع nix-openclaw:
    ```bash
    mkdir -p ~/code/openclaw-local
    # Copy templates/agent-first/flake.nix from the nix-openclaw repo
    ```
  </Step>
  <Step title="إعداد الأسرار">
    اضبط الرمز المميز لبوت المراسلة لديك وAPI key الخاصة بموفّر النموذج. تعمل الملفات النصية العادية ضمن `~/.secrets/` بشكل جيد.
  </Step>
  <Step title="املأ العناصر النائبة في القالب ثم نفّذ switch">
    ```bash
    home-manager switch
    ```
  </Step>
  <Step title="التحقق">
    أكّد أن خدمة launchd تعمل وأن البوت الخاص بك يستجيب للرسائل.
  </Step>
</Steps>

راجع [README الخاص بـ nix-openclaw](https://github.com/openclaw/nix-openclaw) للحصول على خيارات الوحدة الكاملة والأمثلة.

## سلوك وقت التشغيل في Nix Mode

عند تعيين `OPENCLAW_NIX_MODE=1` (تلقائيًا مع nix-openclaw)، يدخل OpenClaw في وضع حتمي يعطّل تدفقات التثبيت التلقائي.

يمكنك أيضًا تعيينه يدويًا:

```bash
export OPENCLAW_NIX_MODE=1
```

على macOS، لا يرث تطبيق GUI متغيرات بيئة shell تلقائيًا. فعّل Nix mode عبر defaults بدلًا من ذلك:

```bash
defaults write ai.openclaw.mac openclaw.nixMode -bool true
```

### ما الذي يتغير في Nix mode

- يتم تعطيل تدفقات التثبيت التلقائي والتعديل الذاتي
- تعرض التبعيات المفقودة رسائل معالجة خاصة بـ Nix
- تعرض واجهة المستخدم لافتة Nix mode للقراءة فقط

### مسارات الإعدادات والحالة

يقرأ OpenClaw إعدادات JSON5 من `OPENCLAW_CONFIG_PATH` ويخزن البيانات القابلة للتغيير في `OPENCLAW_STATE_DIR`. عند التشغيل تحت Nix، اضبط هذه القيم صراحةً إلى مواقع مُدارة بواسطة Nix حتى تبقى حالة وقت التشغيل والإعدادات خارج المتجر غير القابل للتغيير.

| المتغير | الافتراضي |
| ------- | --------- |
| `OPENCLAW_HOME` | `HOME` / `USERPROFILE` / `os.homedir()` |
| `OPENCLAW_STATE_DIR` | `~/.openclaw` |
| `OPENCLAW_CONFIG_PATH` | `$OPENCLAW_STATE_DIR/openclaw.json` |

## ذو صلة

- [nix-openclaw](https://github.com/openclaw/nix-openclaw) -- دليل الإعداد الكامل
- [Wizard](/ar/start/wizard) -- إعداد CLI بدون Nix
- [Docker](/install/docker) -- إعداد قائم على الحاويات
