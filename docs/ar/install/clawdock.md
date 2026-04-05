---
read_when:
    - تشغّل OpenClaw باستخدام Docker كثيرًا وتريد أوامر يومية أقصر
    - تريد طبقة مساعدة للوحة التحكم والسجلات وإعداد الرمز المميز وتدفقات pairing
summary: مساعدات shell الخاصة بـ ClawDock لتثبيتات OpenClaw المعتمدة على Docker
title: ClawDock
x-i18n:
    generated_at: "2026-04-05T12:45:52Z"
    model: gpt-5.4
    provider: openai
    source_hash: 93d67d1d979450d8c9c11854d2f40977c958f1c300e75a5c42ce4c31de86735a
    source_path: install/clawdock.md
    workflow: 15
---

# ClawDock

ClawDock هي طبقة صغيرة من مساعدات shell لتثبيتات OpenClaw المعتمدة على Docker.

تمنحك أوامر مختصرة مثل `clawdock-start` و`clawdock-dashboard` و`clawdock-fix-token` بدلًا من استدعاءات `docker compose ...` الأطول.

إذا لم تكن قد أعددت Docker بعد، فابدأ من [Docker](/install/docker).

## التثبيت

استخدم مسار المساعد القياسي:

```bash
mkdir -p ~/.clawdock && curl -sL https://raw.githubusercontent.com/openclaw/openclaw/main/scripts/clawdock/clawdock-helpers.sh -o ~/.clawdock/clawdock-helpers.sh
echo 'source ~/.clawdock/clawdock-helpers.sh' >> ~/.zshrc && source ~/.zshrc
```

إذا كنت قد ثبّتت ClawDock سابقًا من `scripts/shell-helpers/clawdock-helpers.sh`، فأعد التثبيت من المسار الجديد `scripts/clawdock/clawdock-helpers.sh`. تمت إزالة مسار GitHub الخام القديم.

## ما الذي تحصل عليه

### العمليات الأساسية

| الأمر              | الوصف                    |
| ------------------ | ------------------------ |
| `clawdock-start`   | بدء gateway              |
| `clawdock-stop`    | إيقاف gateway            |
| `clawdock-restart` | إعادة تشغيل gateway      |
| `clawdock-status`  | التحقق من حالة الحاوية   |
| `clawdock-logs`    | متابعة سجلات gateway     |

### الوصول إلى الحاوية

| الأمر                     | الوصف                                       |
| ------------------------- | ------------------------------------------- |
| `clawdock-shell`          | فتح shell داخل حاوية gateway                |
| `clawdock-cli <command>`  | تشغيل أوامر CLI الخاصة بـ OpenClaw في Docker |
| `clawdock-exec <command>` | تنفيذ أمر عشوائي داخل الحاوية               |

### واجهة الويب وpairing

| الأمر                   | الوصف                           |
| ----------------------- | ------------------------------- |
| `clawdock-dashboard`    | فتح عنوان URL الخاص بـ Control UI |
| `clawdock-devices`      | إدراج عمليات pairing المعلقة للأجهزة |
| `clawdock-approve <id>` | الموافقة على طلب pairing         |

### الإعداد والصيانة

| الأمر                | الوصف                                         |
| -------------------- | --------------------------------------------- |
| `clawdock-fix-token` | ضبط رمز gateway داخل الحاوية                  |
| `clawdock-update`    | سحب وإعادة بناء وإعادة تشغيل                  |
| `clawdock-rebuild`   | إعادة بناء صورة Docker فقط                    |
| `clawdock-clean`     | إزالة الحاويات وvolumes                       |

### الأدوات المساعدة

| الأمر                  | الوصف                                   |
| ---------------------- | --------------------------------------- |
| `clawdock-health`      | تشغيل فحص صحة gateway                   |
| `clawdock-token`       | طباعة رمز gateway                       |
| `clawdock-cd`          | الانتقال إلى دليل مشروع OpenClaw        |
| `clawdock-config`      | فتح `~/.openclaw`                       |
| `clawdock-show-config` | طباعة ملفات التكوين مع إخفاء القيم الحساسة |
| `clawdock-workspace`   | فتح دليل مساحة العمل                    |

## تدفق الاستخدام لأول مرة

```bash
clawdock-start
clawdock-fix-token
clawdock-dashboard
```

إذا قال المتصفح إن pairing مطلوبة:

```bash
clawdock-devices
clawdock-approve <request-id>
```

## التكوين والأسرار

يعمل ClawDock مع تقسيم تكوين Docker نفسه الموصوف في [Docker](/install/docker):

- ` <project>/.env` لقيم Docker الخاصة مثل اسم الصورة والمنافذ ورمز gateway
- `~/.openclaw/.env` لمفاتيح المزوّدين المدعومة بـ env ورموز الروبوتات
- `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` للمصادقة المخزنة للمزوّدين عبر OAuth/API-key
- `~/.openclaw/openclaw.json` لتكوين السلوك

استخدم `clawdock-show-config` عندما تريد فحص ملفات `.env` و`openclaw.json` بسرعة. فهو يُخفي قيم `.env` في الإخراج المطبوع.

## صفحات ذات صلة

- [Docker](/install/docker)
- [Docker VM Runtime](/install/docker-vm-runtime)
- [Updating](/install/updating)
