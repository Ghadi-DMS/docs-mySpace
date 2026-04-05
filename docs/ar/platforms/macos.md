---
read_when:
    - تنفيذ ميزات تطبيق macOS
    - تغيير دورة حياة البوابة أو جسر العقدة على macOS
summary: التطبيق المصاحب لـ OpenClaw على macOS (شريط القوائم + وسيط البوابة)
title: تطبيق macOS
x-i18n:
    generated_at: "2026-04-05T12:50:52Z"
    model: gpt-5.4
    provider: openai
    source_hash: bfac937e352ede495f60af47edf3b8e5caa5b692ba0ea01d9fb0de9a44bbc135
    source_path: platforms/macos.md
    workflow: 15
---

# التطبيق المصاحب لـ OpenClaw على macOS ‏(شريط القوائم + وسيط البوابة)

تطبيق macOS هو **الرفيق الموجود في شريط القوائم** لـ OpenClaw. وهو يملك الأذونات،
ويدير/يرتبط بالبوابة محليًا (launchd أو يدويًا)، ويكشف إمكانات macOS
للوكيل باعتبارها عقدة.

## ما الذي يفعله

- يعرض إشعارات أصلية وحالة في شريط القوائم.
- يملك مطالبات TCC ‏(الإشعارات، وإمكانية الوصول، وتسجيل الشاشة، والميكروفون،
  والتعرّف على الكلام، وAutomation/AppleScript).
- يشغّل البوابة أو يتصل بها (محليًا أو عن بُعد).
- يكشف الأدوات الخاصة بـ macOS فقط (Canvas، والكاميرا، وتسجيل الشاشة، و`system.run`).
- يبدأ خدمة مضيف العقدة المحلي في الوضع **Remote** ‏(launchd)، ويوقفها في الوضع **Local**.
- يمكنه استضافة **PeekabooBridge** اختياريًا لأتمتة واجهة المستخدم.
- يثبت CLI العالمي (`openclaw`) عند الطلب عبر npm أو pnpm أو bun (يفضّل التطبيق npm، ثم pnpm، ثم bun؛ ويظل Node هو وقت تشغيل البوابة الموصى به).

## الوضع المحلي مقابل الوضع البعيد

- **Local** ‏(الافتراضي): يرتبط التطبيق ببوابة محلية قيد التشغيل إذا كانت موجودة؛
  وإلا فإنه يفعّل خدمة launchd عبر `openclaw gateway install`.
- **Remote**: يتصل التطبيق ببوابة عبر SSH/Tailscale ولا يبدأ
  عملية محلية أبدًا.
  ويبدأ التطبيق **خدمة مضيف العقدة** المحلية حتى تتمكن البوابة البعيدة من الوصول إلى هذا الـ Mac.
  ولا يقوم التطبيق بتشغيل البوابة كعملية فرعية.
  يفضّل اكتشاف البوابة الآن أسماء Tailscale MagicDNS على عناوين tailnet IP الخام،
  لذلك يتعافى تطبيق Mac بشكل أكثر موثوقية عندما تتغير عناوين tailnet IP.

## التحكم في Launchd

يدير التطبيق LaunchAgent لكل مستخدم بالتصنيف `ai.openclaw.gateway`
(أو `ai.openclaw.<profile>` عند استخدام `--profile`/`OPENCLAW_PROFILE`؛ ولا يزال الاسم القديم `com.openclaw.*` يُزال من التحميل).

```bash
launchctl kickstart -k gui/$UID/ai.openclaw.gateway
launchctl bootout gui/$UID/ai.openclaw.gateway
```

استبدل التصنيف بـ `ai.openclaw.<profile>` عند تشغيل ملف تعريف مسمى.

إذا لم يتم تثبيت LaunchAgent، فقم بتفعيله من التطبيق أو شغّل
`openclaw gateway install`.

## إمكانات العقدة (mac)

يعرض تطبيق macOS نفسه كعقدة. الأوامر الشائعة:

- Canvas: ‏`canvas.present`, `canvas.navigate`, `canvas.eval`, `canvas.snapshot`, `canvas.a2ui.*`
- الكاميرا: ‏`camera.snap`, `camera.clip`
- الشاشة: ‏`screen.record`
- النظام: ‏`system.run`, `system.notify`

تبلّغ العقدة عن خريطة `permissions` حتى تتمكن الوكلاء من تحديد المسموح.

خدمة العقدة + IPC في التطبيق:

- عندما تكون خدمة مضيف العقدة headless قيد التشغيل (الوضع البعيد)، فإنها تتصل بـ Gateway WS كعقدة.
- يتم تنفيذ `system.run` داخل تطبيق macOS ‏(سياق UI/TCC) عبر مقبس Unix محلي؛ وتبقى المطالبات + المخرجات داخل التطبيق.

المخطط (SCI):

```
Gateway -> Node Service (WS)
                 |  IPC (UDS + token + HMAC + TTL)
                 v
             Mac App (UI + TCC + system.run)
```

## موافقات Exec ‏(system.run)

يتم التحكم في `system.run` بواسطة **Exec approvals** في تطبيق macOS ‏(Settings → Exec approvals).
وتُخزَّن إعدادات Security + ask + allowlist محليًا على الـ Mac في:

```
~/.openclaw/exec-approvals.json
```

مثال:

```json
{
  "version": 1,
  "defaults": {
    "security": "deny",
    "ask": "on-miss"
  },
  "agents": {
    "main": {
      "security": "allowlist",
      "ask": "on-miss",
      "allowlist": [{ "pattern": "/opt/homebrew/bin/rg" }]
    }
  }
}
```

ملاحظات:

- إدخالات `allowlist` هي أنماط glob لمسارات الملفات التنفيذية المحلولة.
- يُعامل نص أمر shell الخام الذي يحتوي على صيغة تحكم أو توسيع خاصة بـ shell (`&&`, `||`, `;`, `|`, `` ` ``, `$`, `<`, `>`, `(`, `)`) على أنه فشل في allowlist ويتطلب موافقة صريحة (أو إدراج الملف التنفيذي الخاص بالـ shell في allowlist).
- يؤدي اختيار “Always Allow” في المطالبة إلى إضافة ذلك الأمر إلى allowlist.
- تتم تصفية تجاوزات البيئة في `system.run` (بحذف `PATH`, `DYLD_*`, `LD_*`, `NODE_OPTIONS`, `PYTHON*`, `PERL*`, `RUBYOPT`, `SHELLOPTS`, `PS4`) ثم دمجها مع بيئة التطبيق.
- بالنسبة إلى مغلفات shell ‏(`bash|sh|zsh ... -c/-lc`)، يتم تقليص تجاوزات البيئة الخاصة بالطلب إلى allowlist صغيرة وصريحة (`TERM`, `LANG`, `LC_*`, `COLORTERM`, `NO_COLOR`, `FORCE_COLOR`).
- بالنسبة إلى قرارات allow-always في وضع allowlist، تقوم مغلفات الإرسال المعروفة (`env`, `nice`, `nohup`, `stdbuf`, `timeout`) بحفظ مسارات الملفات التنفيذية الداخلية بدلًا من مسارات المغلفات. وإذا لم يكن فك الالتفاف آمنًا، فلن يتم حفظ أي إدخال allowlist تلقائيًا.

## الروابط العميقة

يسجل التطبيق مخطط URL ‏`openclaw://` للإجراءات المحلية.

### `openclaw://agent`

يشغّل طلب Gateway ‏`agent`.
__OC_I18N_900004__
معلمات الاستعلام:

- `message` ‏(مطلوب)
- `sessionKey` ‏(اختياري)
- `thinking` ‏(اختياري)
- `deliver` / `to` / `channel` ‏(اختيارية)
- `timeoutSeconds` ‏(اختياري)
- `key` ‏(اختياري، مفتاح الوضع غير المراقب)

السلامة:

- من دون `key`، يطلب التطبيق تأكيدًا.
- من دون `key`، يفرض التطبيق حدًا قصيرًا للرسالة في مطالبة التأكيد ويتجاهل `deliver` / `to` / `channel`.
- مع `key` صالح، يكون التشغيل غير مراقب (مقصود للأتمتة الشخصية).

## تدفق الإعداد الأولي (النموذجي)

1. ثبّت وشغّل **OpenClaw.app**.
2. أكمل قائمة التحقق الخاصة بالأذونات (مطالبات TCC).
3. تأكد من أن وضع **Local** نشط وأن البوابة قيد التشغيل.
4. ثبّت CLI إذا كنت تريد الوصول عبر الطرفية.

## موضع دليل الحالة (macOS)

تجنب وضع دليل حالة OpenClaw في iCloud أو مجلدات أخرى متزامنة مع السحابة.
فالمسارات المدعومة بالمزامنة قد تضيف زمن انتقال وقد تسبب أحيانًا سباقات قفل/مزامنة للملفات في
الجلسات وبيانات الاعتماد.

فضّل مسار حالة محليًا غير متزامن مثل:
__OC_I18N_900005__
إذا اكتشف `openclaw doctor` أن الحالة موجودة تحت:

- `~/Library/Mobile Documents/com~apple~CloudDocs/...`
- `~/Library/CloudStorage/...`

فسيصدر تحذيرًا ويوصي بالعودة إلى مسار محلي.

## سير عمل البناء والتطوير (أصلي)

- `cd apps/macos && swift build`
- `swift run OpenClaw` ‏(أو Xcode)
- تغليف التطبيق: `scripts/package-mac-app.sh`

## تصحيح اتصال البوابة (CLI على macOS)

استخدم CLI الخاصة بالتصحيح لتجربة مصافحة Gateway WebSocket ومنطق الاكتشاف نفسيهما
اللذين يستخدمهما تطبيق macOS، من دون تشغيل التطبيق.
__OC_I18N_900006__
خيارات الاتصال:

- `--url <ws://host:port>`: تجاوز الإعداد
- `--mode <local|remote>`: التحليل من الإعداد (الافتراضي: الإعداد أو local)
- `--probe`: فرض فحص سلامة جديد
- `--timeout <ms>`: مهلة الطلب (الافتراضي: `15000`)
- `--json`: إخراج منظم للمقارنة

خيارات الاكتشاف:

- `--include-local`: تضمين البوابات التي كان سيتم تصفيتها على أنها “local”
- `--timeout <ms>`: نافذة الاكتشاف الإجمالية (الافتراضي: `2000`)
- `--json`: إخراج منظم للمقارنة

نصيحة: قارن مع `openclaw gateway discover --json` لمعرفة ما إذا كان
خط اكتشاف تطبيق macOS ‏(`local.` بالإضافة إلى النطاق الواسع المُعد، مع
الرجوع إلى wide-area وTailscale Serve) يختلف عن
اكتشاف Node CLI المعتمد على `dns-sd`.

## بنية الاتصال البعيد (أنفاق SSH)

عندما يعمل تطبيق macOS في وضع **Remote**، فإنه يفتح نفق SSH حتى تتمكن مكونات
واجهة المستخدم المحلية من التحدث إلى البوابة البعيدة وكأنها موجودة على localhost.

### نفق التحكم (منفذ Gateway WebSocket)

- **الغرض:** فحوصات السلامة، والحالة، وWeb Chat، والإعداد، وغيرها من استدعاءات control-plane.
- **المنفذ المحلي:** منفذ البوابة (الافتراضي `18789`) وهو دائم الثبات.
- **المنفذ البعيد:** منفذ البوابة نفسه على المضيف البعيد.
- **السلوك:** لا يوجد منفذ محلي عشوائي؛ يعيد التطبيق استخدام نفق سليم موجود
  أو يعيد تشغيله إذا لزم الأمر.
- **شكل SSH:** ‏`ssh -N -L <local>:127.0.0.1:<remote>` مع BatchMode +
  ExitOnForwardFailure + خيارات keepalive.
- **الإبلاغ عن IP:** يستخدم نفق SSH ‏loopback، لذلك سترى البوابة عنوان IP الخاص بالعقدة
  على أنه `127.0.0.1`. استخدم النقل **Direct (ws/wss)** إذا كنت تريد ظهور عنوان IP الحقيقي للعميل
  (راجع [الوصول البعيد على macOS](/platforms/mac/remote)).

للحصول على خطوات الإعداد، راجع [الوصول البعيد على macOS](/platforms/mac/remote). وبالنسبة إلى تفاصيل
البروتوكول، راجع [بروتوكول البوابة](/gateway/protocol).

## مستندات ذات صلة

- [دليل تشغيل البوابة](/gateway)
- [البوابة (macOS)](/platforms/mac/bundled-gateway)
- [أذونات macOS](/platforms/mac/permissions)
- [Canvas](/platforms/mac/canvas)
