---
read_when:
    - إعداد أو تصحيح التحكم البعيد على macOS
summary: تدفق تطبيق macOS للتحكم في OpenClaw gateway بعيدة عبر SSH
title: التحكم البعيد
x-i18n:
    generated_at: "2026-04-05T12:50:25Z"
    model: gpt-5.4
    provider: openai
    source_hash: 96e46e603c2275d04596b5d1ae0fb6858bd1a102a727dc13924ffcd9808fdf7e
    source_path: platforms/mac/remote.md
    workflow: 15
---

# OpenClaw البعيدة (macOS ⇄ مضيف بعيد)

يتيح هذا التدفق لتطبيق macOS أن يعمل كوحدة تحكم بعيدة كاملة لـ OpenClaw gateway تعمل على مضيف آخر (سطح مكتب/خادم). إنها ميزة التطبيق **Remote over SSH** ‏(التشغيل البعيد). تستخدم جميع الميزات—فحوصات الصحة، وتمرير Voice Wake، وWeb Chat—إعداد SSH البعيد نفسه من _Settings → General_.

## الأوضاع

- **محلي (هذا الـ Mac)**: كل شيء يعمل على الحاسوب المحمول. ولا يوجد SSH.
- **Remote over SSH (الافتراضي)**: تُنفّذ أوامر OpenClaw على المضيف البعيد. يفتح تطبيق mac اتصال SSH مع `-o BatchMode` بالإضافة إلى الهوية/المفتاح المختارين وتمرير منفذ محلي.
- **Remote direct (ws/wss)**: لا يوجد نفق SSH. يتصل تطبيق mac بعنوان URL الخاص بـ gateway مباشرةً (مثلًا عبر Tailscale Serve أو reverse proxy عام عبر HTTPS).

## وسائل النقل البعيدة

يدعم الوضع البعيد وسيلتي نقل:

- **نفق SSH** (الافتراضي): يستخدم `ssh -N -L ...` لتمرير منفذ gateway إلى localhost. سترى gateway عنوان IP الخاص بالعقدة كـ `127.0.0.1` لأن النفق عبر loopback.
- **Direct (ws/wss)**: يتصل مباشرة بعنوان URL الخاص بـ gateway. ترى gateway عنوان IP الحقيقي للعميل.

## المتطلبات الأساسية على المضيف البعيد

1. ثبّت Node + pnpm وابنِ/ثبّت OpenClaw CLI ‏(`pnpm install && pnpm build && pnpm link --global`).
2. تأكد من أن `openclaw` موجودة على PATH لصدفات non-interactive ‏(أنشئ symlink إلى `/usr/local/bin` أو `/opt/homebrew/bin` إذا لزم).
3. افتح SSH مع مصادقة المفاتيح. نوصي باستخدام عناوين IP الخاصة بـ **Tailscale** لسهولة الوصول المستقر خارج LAN.

## إعداد تطبيق macOS

1. افتح _Settings → General_.
2. تحت **OpenClaw runs**، اختر **Remote over SSH** واضبط:
   - **Transport**: ‏**SSH tunnel** أو **Direct (ws/wss)**.
   - **SSH target**: ‏`user@host` ‏(مع `:port` اختياري).
     - إذا كانت gateway على LAN نفسها وتعلن نفسها عبر Bonjour، فاخترها من القائمة المكتشفة لملء هذا الحقل تلقائيًا.
   - **Gateway URL** ‏(في الوضع المباشر فقط): ‏`wss://gateway.example.ts.net` ‏(أو `ws://...` للوضع المحلي/LAN).
   - **Identity file** ‏(متقدم): مسار المفتاح الخاص بك.
   - **Project root** ‏(متقدم): مسار النسخة البعيدة المستخدم للأوامر.
   - **CLI path** ‏(متقدم): مسار اختياري إلى entrypoint/binary قابلة للتشغيل لـ `openclaw` (يُملأ تلقائيًا عند الإعلان عنه).
3. اضغط **Test remote**. يشير النجاح إلى أن `openclaw status --json` البعيدة تعمل بشكل صحيح. تعني الإخفاقات عادةً مشكلات PATH/CLI؛ ويعني الخروج 127 أن CLI غير موجودة على المضيف البعيد.
4. ستعمل فحوصات الصحة وWeb Chat الآن عبر نفق SSH هذا تلقائيًا.

## Web Chat

- **نفق SSH**: تتصل Web Chat بـ gateway عبر منفذ WebSocket control المُمرّر (الافتراضي 18789).
- **Direct (ws/wss)**: تتصل Web Chat مباشرةً بعنوان URL المكوّن لـ gateway.
- لم يعد هناك خادم HTTP منفصل لـ WebChat.

## الأذونات

- يحتاج المضيف البعيد إلى موافقات TCC نفسها الموجودة محليًا (Automation وAccessibility وScreen Recording وMicrophone وSpeech Recognition وNotifications). شغّل onboarding على ذلك الجهاز لمنحها مرة واحدة.
- تعلن العقد عن حالة الأذونات لديها عبر `node.list` / `node.describe` حتى تعرف الوكلاء ما هو المتاح.

## ملاحظات أمنية

- فضّل روابط loopback على المضيف البعيد واتصل عبر SSH أو Tailscale.
- يستخدم تمرير SSH فحصًا صارمًا لمفاتيح المضيف؛ لذا وثّق مفتاح المضيف أولًا حتى يوجد في `~/.ssh/known_hosts`.
- إذا ربطت Gateway بواجهة غير loopback، فاشترط مصادقة Gateway صالحة: رمزًا، أو كلمة مرور، أو reverse proxy واعيًا بالهوية مع `gateway.auth.mode: "trusted-proxy"`.
- راجع [Security](/gateway/security) و[Tailscale](/gateway/tailscale).

## تدفق تسجيل دخول WhatsApp ‏(بعيد)

- شغّل `openclaw channels login --verbose` **على المضيف البعيد**. ثم امسح QR باستخدام WhatsApp على هاتفك.
- أعد تشغيل تسجيل الدخول على ذلك المضيف إذا انتهت صلاحية المصادقة. وستعرض فحوصات الصحة مشكلات الربط.

## استكشاف الأخطاء وإصلاحها

- **exit 127 / not found**: ‏`openclaw` غير موجودة على PATH لصدفات non-login. أضفها إلى `/etc/paths` أو shell rc أو أنشئ symlink إلى `/usr/local/bin`/`/opt/homebrew/bin`.
- **Health probe failed**: تحقق من إمكانية الوصول عبر SSH وPATH وأن Baileys مسجّل الدخول (`openclaw status --json`).
- **Web Chat عالقة**: تأكد من أن gateway تعمل على المضيف البعيد وأن المنفذ المُمرَّر يطابق منفذ WS الخاص بـ gateway؛ إذ تتطلب الواجهة اتصال WS سليمًا.
- **تظهر IP الخاصة بالعقدة كـ 127.0.0.1**: هذا متوقع مع نفق SSH. بدّل **Transport** إلى **Direct (ws/wss)** إذا كنت تريد أن ترى gateway عنوان IP الحقيقي للعميل.
- **Voice Wake**: تُمرَّر عبارات التنشيط تلقائيًا في الوضع البعيد؛ ولا حاجة إلى forwarder منفصل.

## أصوات الإشعارات

اختر أصواتًا لكل إشعار من scripts باستخدام `openclaw` و`node.invoke`، مثلًا:

```bash
openclaw nodes notify --node <id> --title "Ping" --body "Remote gateway ready" --sound Glass
```

لم يعد هناك مفتاح تبديل عام لـ “الصوت الافتراضي” في التطبيق؛ يختار المستدعون صوتًا (أو لا شيء) لكل طلب.
