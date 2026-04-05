---
read_when: Browser control fails on Linux, especially with snap Chromium
summary: إصلاح مشكلات بدء تشغيل CDP في Chrome/Brave/Edge/Chromium لتحكم متصفح OpenClaw على Linux
title: استكشاف أخطاء المتصفح وإصلاحها
x-i18n:
    generated_at: "2026-04-05T12:57:24Z"
    model: gpt-5.4
    provider: openai
    source_hash: 9ff8e6741558c1b5db86826c5e1cbafe35e35afe5cb2a53296c16653da59e516
    source_path: tools/browser-linux-troubleshooting.md
    workflow: 15
---

# استكشاف أخطاء المتصفح وإصلاحها (Linux)

## المشكلة: "Failed to start Chrome CDP on port 18800"

يفشل خادم التحكم في المتصفح الخاص بـ OpenClaw في تشغيل Chrome/Brave/Edge/Chromium مع الخطأ:

```
{"error":"Error: Failed to start Chrome CDP on port 18800 for profile \"openclaw\"."}
```

### السبب الجذري

في Ubuntu (والعديد من توزيعات Linux)، يكون تثبيت Chromium الافتراضي **حزمة snap**. يتداخل احتواء AppArmor الخاص بـ Snap مع الطريقة التي يشغّل بها OpenClaw عملية المتصفح ويراقبها.

يقوم الأمر `apt install chromium` بتثبيت حزمة بديلة تعيد التوجيه إلى snap:

```
Note, selecting 'chromium-browser' instead of 'chromium'
chromium-browser is already the newest version (2:1snap1-0ubuntu2).
```

هذا **ليس** متصفحًا حقيقيًا — بل مجرد غلاف.

### الحل 1: تثبيت Google Chrome (موصى به)

ثبّت حزمة `.deb` الرسمية من Google Chrome، فهي ليست معزولة بواسطة snap:

```bash
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
sudo dpkg -i google-chrome-stable_current_amd64.deb
sudo apt --fix-broken install -y  # إذا كانت هناك أخطاء تبعيات
```

ثم حدّث إعداد OpenClaw لديك (`~/.openclaw/openclaw.json`):

```json
{
  "browser": {
    "enabled": true,
    "executablePath": "/usr/bin/google-chrome-stable",
    "headless": true,
    "noSandbox": true
  }
}
```

### الحل 2: استخدام Snap Chromium مع وضع الإرفاق فقط

إذا كان لا بد لك من استخدام snap Chromium، فاضبط OpenClaw ليرتبط بمتصفح تم تشغيله يدويًا:

1. حدّث الإعداد:

```json
{
  "browser": {
    "enabled": true,
    "attachOnly": true,
    "headless": true,
    "noSandbox": true
  }
}
```

2. شغّل Chromium يدويًا:

```bash
chromium-browser --headless --no-sandbox --disable-gpu \
  --remote-debugging-port=18800 \
  --user-data-dir=$HOME/.openclaw/browser/openclaw/user-data \
  about:blank &
```

3. اختياريًا، أنشئ خدمة مستخدم systemd لبدء Chrome تلقائيًا:

```ini
# ~/.config/systemd/user/openclaw-browser.service
[Unit]
Description=OpenClaw Browser (Chrome CDP)
After=network.target

[Service]
ExecStart=/snap/bin/chromium --headless --no-sandbox --disable-gpu --remote-debugging-port=18800 --user-data-dir=%h/.openclaw/browser/openclaw/user-data about:blank
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

فعّلها باستخدام: `systemctl --user enable --now openclaw-browser.service`

### التحقق من أن المتصفح يعمل

تحقق من الحالة:

```bash
curl -s http://127.0.0.1:18791/ | jq '{running, pid, chosenBrowser}'
```

اختبر التصفح:

```bash
curl -s -X POST http://127.0.0.1:18791/start
curl -s http://127.0.0.1:18791/tabs
```

### مرجع الإعداد

| الخيار | الوصف | الافتراضي |
| --- | --- | --- |
| `browser.enabled` | تمكين التحكم في المتصفح | `true` |
| `browser.executablePath` | مسار ملف متصفح ثنائي قائم على Chromium ‏(Chrome/Brave/Edge/Chromium) | يُكتشف تلقائيًا (ويُفضّل المتصفح الافتراضي إذا كان قائمًا على Chromium) |
| `browser.headless` | التشغيل دون واجهة رسومية | `false` |
| `browser.noSandbox` | إضافة العلامة `--no-sandbox` (مطلوبة لبعض إعدادات Linux) | `false` |
| `browser.attachOnly` | عدم تشغيل المتصفح، والاكتفاء بالإرفاق بمتصفح موجود | `false` |
| `browser.cdpPort` | منفذ Chrome DevTools Protocol | `18800` |

### المشكلة: "No Chrome tabs found for profile=\"user\""

أنت تستخدم ملف تعريف `existing-session` / Chrome MCP. يمكن لـ OpenClaw رؤية Chrome المحلي،
لكن لا توجد علامات تبويب مفتوحة متاحة للإرفاق بها.

خيارات الإصلاح:

1. **استخدم المتصفح المُدار:** `openclaw browser start --browser-profile openclaw`
   (أو عيّن `browser.defaultProfile: "openclaw"`).
2. **استخدم Chrome MCP:** تأكد من أن Chrome المحلي يعمل مع وجود علامة تبويب واحدة مفتوحة على الأقل، ثم أعد المحاولة باستخدام `--browser-profile user`.

ملاحظات:

- `user` خاص بالمضيف فقط. بالنسبة إلى خوادم Linux أو الحاويات أو المضيفين البعيدين، يُفضّل استخدام ملفات تعريف CDP.
- يحتفظ `user` / وملفات تعريف `existing-session` الأخرى بقيود Chrome MCP الحالية:
  الإجراءات المعتمدة على المراجع، وخطافات رفع ملف واحد، وعدم وجود
  تجاوزات لمهلة مربعات الحوار، وعدم وجود
  `wait --load networkidle`، وعدم وجود `responsebody` أو تصدير PDF أو اعتراض
  التنزيلات أو الإجراءات الدفعية.
- تعيّن ملفات تعريف `openclaw` المحلية `cdpPort`/`cdpUrl` تلقائيًا؛ لا تضبطهما إلا من أجل CDP البعيد.
- تقبل ملفات تعريف CDP البعيدة `http://` و`https://` و`ws://` و`wss://`.
  استخدم HTTP(S) لاكتشاف `/json/version`، أو WS(S) عندما تمنحك خدمة
  المتصفح عنوان socket مباشرًا لـ DevTools.
