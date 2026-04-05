---
read_when:
    - العمل على ميزات قناة Google Chat
summary: حالة دعم تطبيق Google Chat وإمكاناته وتكوينه
title: Google Chat
x-i18n:
    generated_at: "2026-04-05T12:35:08Z"
    model: gpt-5.4
    provider: openai
    source_hash: 570894ed798dd0b9ba42806b050927216379a1228fcd2f96de565bc8a4ac7c2c
    source_path: channels/googlechat.md
    workflow: 15
---

# Google Chat (Chat API)

الحالة: جاهز للرسائل المباشرة + المساحات عبر webhooks الخاصة بـ Google Chat API (HTTP فقط).

## إعداد سريع (للمبتدئين)

1. أنشئ مشروع Google Cloud وفعّل **Google Chat API**.
   - انتقل إلى: [Google Chat API Credentials](https://console.cloud.google.com/apis/api/chat.googleapis.com/credentials)
   - فعّل API إذا لم تكن مفعلة بالفعل.
2. أنشئ **Service Account**:
   - اضغط **Create Credentials** > **Service Account**.
   - سمّه كما تريد (مثلًا: `openclaw-chat`).
   - اترك الأذونات فارغة (اضغط **Continue**).
   - اترك الجهات الرئيسية التي لديها وصول فارغة (اضغط **Done**).
3. أنشئ ونزّل **JSON Key**:
   - في قائمة حسابات الخدمة، انقر على الحساب الذي أنشأته للتو.
   - انتقل إلى علامة التبويب **Keys**.
   - انقر **Add Key** > **Create new key**.
   - اختر **JSON** واضغط **Create**.
4. خزّن ملف JSON الذي تم تنزيله على مضيف gateway لديك (مثلًا: `~/.openclaw/googlechat-service-account.json`).
5. أنشئ تطبيق Google Chat في [Google Cloud Console Chat Configuration](https://console.cloud.google.com/apis/api/chat.googleapis.com/hangouts-chat):
   - املأ **Application info**:
     - **App name**: (مثلًا `OpenClaw`)
     - **Avatar URL**: (مثلًا `https://openclaw.ai/logo.png`)
     - **Description**: (مثلًا `Personal AI Assistant`)
   - فعّل **Interactive features**.
   - ضمن **Functionality**، حدّد **Join spaces and group conversations**.
   - ضمن **Connection settings**، اختر **HTTP endpoint URL**.
   - ضمن **Triggers**، اختر **Use a common HTTP endpoint URL for all triggers** واضبطه على عنوان URL العام الخاص بـ gateway متبوعًا بـ `/googlechat`.
     - _نصيحة: شغّل `openclaw status` للعثور على عنوان URL العام الخاص بـ gateway._
   - ضمن **Visibility**، حدّد **Make this Chat app available to specific people and groups in &lt;Your Domain&gt;**.
   - أدخل عنوان بريدك الإلكتروني (مثلًا `user@example.com`) في مربع النص.
   - انقر **Save** في الأسفل.
6. **فعّل حالة التطبيق**:
   - بعد الحفظ، **حدّث الصفحة**.
   - ابحث عن قسم **App status** (عادةً قرب الأعلى أو الأسفل بعد الحفظ).
   - غيّر الحالة إلى **Live - available to users**.
   - انقر **Save** مرة أخرى.
7. اضبط OpenClaw باستخدام مسار حساب الخدمة + webhook audience:
   - متغير البيئة: `GOOGLE_CHAT_SERVICE_ACCOUNT_FILE=/path/to/service-account.json`
   - أو التكوين: `channels.googlechat.serviceAccountFile: "/path/to/service-account.json"`.
8. اضبط نوع وقيمة webhook audience (بحيث يطابق إعداد تطبيق Chat لديك).
9. ابدأ gateway. سيُرسل Google Chat طلبات POST إلى مسار webhook لديك.

## الإضافة إلى Google Chat

بمجرد تشغيل gateway وإضافة بريدك الإلكتروني إلى قائمة الظهور:

1. انتقل إلى [Google Chat](https://chat.google.com/).
2. انقر على أيقونة **+** (زائد) بجانب **Direct Messages**.
3. في شريط البحث (حيث تضيف الأشخاص عادةً)، اكتب **App name** الذي قمت بتكوينه في Google Cloud Console.
   - **ملاحظة**: لن يظهر الروبوت في قائمة التصفح "Marketplace" لأنه تطبيق خاص. يجب أن تبحث عنه بالاسم.
4. اختر الروبوت من النتائج.
5. انقر **Add** أو **Chat** لبدء محادثة فردية.
6. أرسل "Hello" لتشغيل المساعد!

## عنوان URL عام (Webhook فقط)

تتطلب webhooks الخاصة بـ Google Chat نقطة نهاية HTTPS عامة. لأسباب أمنية، **اكشف فقط المسار `/googlechat`** للإنترنت. أبقِ لوحة تحكم OpenClaw ونقاط النهاية الحساسة الأخرى على شبكتك الخاصة.

### الخيار A: Tailscale Funnel (موصى به)

استخدم Tailscale Serve للوحة التحكم الخاصة وFunnel لمسار webhook العام. هذا يُبقي `/` خاصًا مع كشف `/googlechat` فقط.

1. **تحقق من العنوان المرتبط به gateway:**

   ```bash
   ss -tlnp | grep 18789
   ```

   دوّن عنوان IP (مثلًا `127.0.0.1` أو `0.0.0.0` أو عنوان Tailscale الخاص بك مثل `100.x.x.x`).

2. **اكشف لوحة التحكم إلى tailnet فقط (المنفذ 8443):**

   ```bash
   # If bound to localhost (127.0.0.1 or 0.0.0.0):
   tailscale serve --bg --https 8443 http://127.0.0.1:18789

   # If bound to Tailscale IP only (e.g., 100.106.161.80):
   tailscale serve --bg --https 8443 http://100.106.161.80:18789
   ```

3. **اكشف مسار webhook فقط بشكل عام:**

   ```bash
   # If bound to localhost (127.0.0.1 or 0.0.0.0):
   tailscale funnel --bg --set-path /googlechat http://127.0.0.1:18789/googlechat

   # If bound to Tailscale IP only (e.g., 100.106.161.80):
   tailscale funnel --bg --set-path /googlechat http://100.106.161.80:18789/googlechat
   ```

4. **فوّض العقدة للوصول عبر Funnel:**
   إذا طُلب منك ذلك، فزر عنوان URL الخاص بالتفويض الظاهر في المخرجات لتمكين Funnel لهذه العقدة في سياسة tailnet الخاصة بك.

5. **تحقق من التكوين:**

   ```bash
   tailscale serve status
   tailscale funnel status
   ```

سيكون عنوان URL العام لـ webhook لديك:
`https://<node-name>.<tailnet>.ts.net/googlechat`

وستبقى لوحة التحكم الخاصة بك ضمن tailnet فقط:
`https://<node-name>.<tailnet>.ts.net:8443/`

استخدم عنوان URL العام (من دون `:8443`) في إعداد تطبيق Google Chat.

> ملاحظة: يستمر هذا التكوين بعد إعادة التشغيل. لإزالته لاحقًا، شغّل `tailscale funnel reset` و`tailscale serve reset`.

### الخيار B: Reverse Proxy (Caddy)

إذا كنت تستخدم reverse proxy مثل Caddy، فقم بتمرير المسار المحدد فقط:

```caddy
your-domain.com {
    reverse_proxy /googlechat* localhost:18789
}
```

مع هذا التكوين، سيتم تجاهل أي طلب إلى `your-domain.com/` أو إرجاع 404 له، بينما يتم توجيه `your-domain.com/googlechat` بأمان إلى OpenClaw.

### الخيار C: Cloudflare Tunnel

اضبط قواعد ingress الخاصة بـ tunnel لتوجيه مسار webhook فقط:

- **المسار**: `/googlechat` -> `http://localhost:18789/googlechat`
- **القاعدة الافتراضية**: HTTP 404 (غير موجود)

## كيف يعمل

1. يرسل Google Chat طلبات webhook من نوع POST إلى gateway. يتضمن كل طلب ترويسة `Authorization: Bearer <token>`.
   - يتحقق OpenClaw من bearer auth قبل قراءة/تحليل أجسام webhook الكاملة عند وجود الترويسة.
   - يتم دعم طلبات Google Workspace Add-on التي تحمل `authorizationEventObject.systemIdToken` في الجسم من خلال حد أقصى أكثر صرامة لجسم ما قبل المصادقة.
2. يتحقق OpenClaw من الرمز المميز مقابل `audienceType` و`audience` المكوّنين:
   - `audienceType: "app-url"` → يكون audience هو عنوان URL الخاص بـ webhook عبر HTTPS.
   - `audienceType: "project-number"` → يكون audience هو رقم مشروع Cloud.
3. يتم توجيه الرسائل حسب المساحة:
   - تستخدم الرسائل المباشرة مفتاح الجلسة `agent:<agentId>:googlechat:direct:<spaceId>`.
   - تستخدم المساحات مفتاح الجلسة `agent:<agentId>:googlechat:group:<spaceId>`.
4. يكون وصول الرسائل المباشرة عبر pairing افتراضيًا. يتلقى المرسلون غير المعروفين رمز pairing؛ وافق عليه باستخدام:
   - `openclaw pairing approve googlechat <code>`
5. تتطلب المساحات الجماعية إشارة @ افتراضيًا. استخدم `botUser` إذا كان اكتشاف الإشارة يحتاج إلى اسم مستخدم التطبيق.

## الأهداف

استخدم هذه المعرّفات للتسليم وقوائم السماح:

- الرسائل المباشرة: `users/<userId>` (موصى به).
- البريد الإلكتروني الخام `name@example.com` قابل للتغيير ويُستخدم فقط لمطابقة قائمة السماح المباشرة عندما تكون `channels.googlechat.dangerouslyAllowNameMatching: true`.
- متروك: يتم التعامل مع `users/<email>` على أنه معرّف مستخدم، وليس قائمة سماح للبريد الإلكتروني.
- المساحات: `spaces/<spaceId>`.

## أبرز التكوينات

```json5
{
  channels: {
    googlechat: {
      enabled: true,
      serviceAccountFile: "/path/to/service-account.json",
      // or serviceAccountRef: { source: "file", provider: "filemain", id: "/channels/googlechat/serviceAccount" }
      audienceType: "app-url",
      audience: "https://gateway.example.com/googlechat",
      webhookPath: "/googlechat",
      botUser: "users/1234567890", // optional; helps mention detection
      dm: {
        policy: "pairing",
        allowFrom: ["users/1234567890"],
      },
      groupPolicy: "allowlist",
      groups: {
        "spaces/AAAA": {
          allow: true,
          requireMention: true,
          users: ["users/1234567890"],
          systemPrompt: "Short answers only.",
        },
      },
      actions: { reactions: true },
      typingIndicator: "message",
      mediaMaxMb: 20,
    },
  },
}
```

ملاحظات:

- يمكن أيضًا تمرير بيانات اعتماد حساب الخدمة مباشرة باستخدام `serviceAccount` (سلسلة JSON).
- `serviceAccountRef` مدعوم أيضًا (SecretRef للبيئة/الملف)، بما في ذلك المراجع لكل حساب ضمن `channels.googlechat.accounts.<id>.serviceAccountRef`.
- مسار webhook الافتراضي هو `/googlechat` إذا لم يتم تعيين `webhookPath`.
- تعيد `dangerouslyAllowNameMatching` تمكين مطابقة principal للبريد الإلكتروني القابل للتغيير لقوائم السماح (وضع توافقية طارئ).
- تتوفر التفاعلات عبر أداة `reactions` و`channels action` عندما تكون `actions.reactions` مفعلة.
- تعرض إجراءات الرسائل `send` للنص و`upload-file` لإرسال المرفقات بشكل صريح. تقبل `upload-file` القيم `media` / `filePath` / `path` بالإضافة إلى `message` و`filename` الاختياريين واستهداف السلسلة.
- يدعم `typingIndicator` القيم `none` و`message` (الافتراضي) و`reaction` (يتطلب التفاعل OAuth للمستخدم).
- يتم تنزيل المرفقات عبر Chat API وتخزينها في مسار الوسائط (مع حد للحجم تحدده `mediaMaxMb`).

تفاصيل مرجع الأسرار: [Secrets Management](/gateway/secrets).

## استكشاف الأخطاء وإصلاحها

### 405 Method Not Allowed

إذا أظهر Google Cloud Logs Explorer أخطاء مثل:

```
status code: 405, reason phrase: HTTP error response: HTTP/1.1 405 Method Not Allowed
```

فهذا يعني أن معالج webhook غير مسجّل. الأسباب الشائعة:

1. **القناة غير مكوّنة**: قسم `channels.googlechat` مفقود من التكوين. تحقق باستخدام:

   ```bash
   openclaw config get channels.googlechat
   ```

   إذا أعاد "Config path not found"، فأضف التكوين (راجع [أبرز التكوينات](#config-highlights)).

2. **plugin غير مفعّل**: تحقق من حالة plugin:

   ```bash
   openclaw plugins list | grep googlechat
   ```

   إذا أظهر "disabled"، فأضف `plugins.entries.googlechat.enabled: true` إلى التكوين.

3. **لم تتم إعادة تشغيل gateway**: بعد إضافة التكوين، أعد تشغيل gateway:

   ```bash
   openclaw gateway restart
   ```

تحقق من أن القناة تعمل:

```bash
openclaw channels status
# Should show: Google Chat default: enabled, configured, ...
```

### مشكلات أخرى

- تحقق من `openclaw channels status --probe` بحثًا عن أخطاء المصادقة أو غياب إعداد audience.
- إذا لم تصل أي رسائل، فتأكد من عنوان URL الخاص بـ webhook لتطبيق Chat واشتراكات الأحداث.
- إذا كان تقييد الإشارة يمنع الردود، فاضبط `botUser` على اسم مورد مستخدم التطبيق وتحقق من `requireMention`.
- استخدم `openclaw logs --follow` أثناء إرسال رسالة اختبار لمعرفة ما إذا كانت الطلبات تصل إلى gateway.

الوثائق ذات الصلة:

- [تكوين gateway](/gateway/configuration)
- [الأمان](/gateway/security)
- [التفاعلات](/tools/reactions)

## ذو صلة

- [نظرة عامة على القنوات](/channels) — جميع القنوات المدعومة
- [Pairing](/channels/pairing) — مصادقة الرسائل المباشرة وتدفق pairing
- [المجموعات](/channels/groups) — سلوك الدردشة الجماعية وتقييد الإشارات
- [توجيه القنوات](/channels/channel-routing) — توجيه الجلسات للرسائل
- [الأمان](/gateway/security) — نموذج الوصول والتقوية
