---
read_when:
    - الإجابة عن الأسئلة الشائعة المتعلقة بالإعداد أو التثبيت أو الإدخال الأولي أو دعم وقت التشغيل
    - فرز المشكلات التي يبلغ عنها المستخدمون قبل بدء تصحيح أخطاء أعمق
summary: الأسئلة الشائعة حول إعداد OpenClaw وتكوينه واستخدامه
title: الأسئلة الشائعة
x-i18n:
    generated_at: "2026-04-08T02:26:46Z"
    model: gpt-5.4
    provider: openai
    source_hash: 001b4605966b45b08108606f76ae937ec348c2179b04cf6fb34fef94833705e6
    source_path: help/faq.md
    workflow: 15
---

# الأسئلة الشائعة

إجابات سريعة مع خطوات أعمق لاستكشاف الأخطاء وإصلاحها لعمليات الإعداد الواقعية (التطوير المحلي، VPS، تعدد الوكلاء، مفاتيح OAuth/API، والتبديل الاحتياطي للنماذج). لتشخيصات وقت التشغيل، راجع [استكشاف الأخطاء وإصلاحها](/ar/gateway/troubleshooting). وللمرجع الكامل للإعدادات، راجع [التكوين](/ar/gateway/configuration).

## أول 60 ثانية إذا كان هناك شيء معطّل

1. **الحالة السريعة (أول فحص)**

   ```bash
   openclaw status
   ```

   ملخص محلي سريع: نظام التشغيل + التحديث، إمكانية الوصول إلى gateway/service، الوكلاء/الجلسات، إعدادات المزوّد + مشكلات وقت التشغيل (عند إمكانية الوصول إلى gateway).

2. **تقرير قابل للمشاركة والنسخ**

   ```bash
   openclaw status --all
   ```

   تشخيص للقراءة فقط مع ذيل السجل (مع إخفاء الرموز المميزة).

3. **حالة daemon + المنفذ**

   ```bash
   openclaw gateway status
   ```

   يعرض وقت تشغيل المشرف مقابل إمكانية الوصول عبر RPC، وعنوان URL المستهدف للفحص، وأي إعدادات يُرجّح أن الخدمة استخدمتها.

4. **فحوصات متعمقة**

   ```bash
   openclaw status --deep
   ```

   يشغّل فحصًا مباشرًا لصحة gateway، بما في ذلك فحوصات القنوات عندما تكون مدعومة
   (يتطلب gateway يمكن الوصول إليه). راجع [الصحة](/ar/gateway/health).

5. **اعرض أحدث سجل**

   ```bash
   openclaw logs --follow
   ```

   إذا كان RPC متوقفًا، فاستخدم بدلًا من ذلك:

   ```bash
   tail -f "$(ls -t /tmp/openclaw/openclaw-*.log | head -1)"
   ```

   سجلات الملفات منفصلة عن سجلات الخدمة؛ راجع [التسجيل](/ar/logging) و[استكشاف الأخطاء وإصلاحها](/ar/gateway/troubleshooting).

6. **شغّل أداة doctor (الإصلاحات)**

   ```bash
   openclaw doctor
   ```

   يصلح/يرحّل الإعدادات/الحالة + يشغّل فحوصات الصحة. راجع [Doctor](/ar/gateway/doctor).

7. **لقطة gateway**

   ```bash
   openclaw health --json
   openclaw health --verbose   # يعرض عنوان URL المستهدف + مسار الإعدادات عند وجود أخطاء
   ```

   يطلب من gateway الجاري تشغيله لقطة كاملة (WS فقط). راجع [الصحة](/ar/gateway/health).

## البدء السريع وإعداد التشغيل الأول

<AccordionGroup>
  <Accordion title="أنا عالق، ما أسرع طريقة للخروج من هذا المأزق؟">
    استخدم وكيل AI محلي يمكنه **رؤية جهازك**. هذا أكثر فاعلية بكثير من السؤال
    في Discord، لأن معظم حالات "أنا عالق" تكون **مشكلات إعدادات أو بيئة محلية**
    لا يستطيع المساعدون البعيدون فحصها.

    - **Claude Code**: [https://www.anthropic.com/claude-code/](https://www.anthropic.com/claude-code/)
    - **OpenAI Codex**: [https://openai.com/codex/](https://openai.com/codex/)

    تستطيع هذه الأدوات قراءة المستودع، وتشغيل الأوامر، وفحص السجلات، والمساعدة في إصلاح
    الإعداد على مستوى الجهاز لديك (PATH، الخدمات، الأذونات، ملفات المصادقة). امنحها
    **نسخة المصدر كاملة** عبر تثبيت git القابل للاختراق:

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    يثبّت هذا OpenClaw **من نسخة git**، بحيث يتمكن الوكيل من قراءة الشيفرة + المستندات
    والاستدلال على الإصدار الدقيق الذي تستخدمه. يمكنك دائمًا العودة إلى الإصدار المستقر لاحقًا
    عبر إعادة تشغيل برنامج التثبيت بدون `--install-method git`.

    نصيحة: اطلب من الوكيل أن **يخطط للإصلاح ويشرف عليه** (خطوة بخطوة)، ثم ينفذ فقط
    الأوامر الضرورية. هذا يُبقي التغييرات صغيرة وأسهل في التدقيق.

    إذا اكتشفت خطأً حقيقيًا أو أصلحت شيئًا، فيرجى فتح issue على GitHub أو إرسال PR:
    [https://github.com/openclaw/openclaw/issues](https://github.com/openclaw/openclaw/issues)
    [https://github.com/openclaw/openclaw/pulls](https://github.com/openclaw/openclaw/pulls)

    ابدأ بهذه الأوامر (شارك المخرجات عند طلب المساعدة):

    ```bash
    openclaw status
    openclaw models status
    openclaw doctor
    ```

    ما الذي تفعله:

    - `openclaw status`: لقطة سريعة لصحة gateway/agent + الإعدادات الأساسية.
    - `openclaw models status`: يتحقق من مصادقة المزوّد + توفر النماذج.
    - `openclaw doctor`: يتحقق من مشكلات الإعدادات/الحالة الشائعة ويصلحها.

    فحوصات CLI أخرى مفيدة: `openclaw status --all`, `openclaw logs --follow`,
    `openclaw gateway status`, `openclaw health --verbose`.

    حلقة تصحيح سريعة: [أول 60 ثانية إذا كان هناك شيء معطّل](#أول-60-ثانية-إذا-كان-هناك-شيء-معطّل).
    مستندات التثبيت: [التثبيت](/ar/install)، [إشارات برنامج التثبيت](/ar/install/installer)، [التحديث](/ar/install/updating).

  </Accordion>

  <Accordion title="Heartbeat يستمر في التخطي. ماذا تعني أسباب التخطي؟">
    أسباب تخطي heartbeat الشائعة:

    - `quiet-hours`: خارج نافذة active-hours المكوّنة
    - `empty-heartbeat-file`: يوجد `HEARTBEAT.md` لكنه يحتوي فقط على هيكل فارغ/عناوين فقط
    - `no-tasks-due`: وضع مهام `HEARTBEAT.md` نشط ولكن لم يحن موعد أي من فترات المهام بعد
    - `alerts-disabled`: تم تعطيل كل ظهور heartbeat (`showOk` و`showAlerts` و`useIndicator` كلها متوقفة)

    في وضع المهام، لا يتم تقديم الطوابع الزمنية المستحقة إلا بعد اكتمال تشغيل heartbeat
    فعليًا. عمليات التشغيل المتخطاة لا تضع علامة على المهام بأنها مكتملة.

    المستندات: [Heartbeat](/ar/gateway/heartbeat)، [الأتمتة والمهام](/ar/automation).

  </Accordion>

  <Accordion title="الطريقة الموصى بها لتثبيت OpenClaw وإعداده">
    يوصي المستودع بالتشغيل من المصدر واستخدام الإدخال الأولي:

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash
    openclaw onboard --install-daemon
    ```

    يمكن للمعالج أيضًا بناء أصول UI تلقائيًا. بعد الإدخال الأولي، ستشغّل عادةً Gateway على المنفذ **18789**.

    من المصدر (للمساهمين/المطورين):

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    pnpm ui:build # يثبت تبعيات UI تلقائيًا عند أول تشغيل
    openclaw onboard
    ```

    إذا لم يكن لديك تثبيت عام بعد، شغّله عبر `pnpm openclaw onboard`.

  </Accordion>

  <Accordion title="كيف أفتح لوحة التحكم بعد الإدخال الأولي؟">
    يفتح المعالج متصفحك بعنوان URL نظيف للوحة التحكم (من دون token) مباشرةً بعد الإدخال الأولي، كما يطبع الرابط في الملخص. أبقِ هذا التبويب مفتوحًا؛ وإذا لم يُفتح تلقائيًا، انسخ/ألصق عنوان URL المطبوع على الجهاز نفسه.
  </Accordion>

  <Accordion title="كيف أصادق لوحة التحكم على localhost مقابل البيئة البعيدة؟">
    **Localhost (الجهاز نفسه):**

    - افتح `http://127.0.0.1:18789/`.
    - إذا طلب مصادقة shared-secret، الصق token أو password المكوّن في إعدادات Control UI.
    - مصدر token: `gateway.auth.token` (أو `OPENCLAW_GATEWAY_TOKEN`).
    - مصدر password: `gateway.auth.password` (أو `OPENCLAW_GATEWAY_PASSWORD`).
    - إذا لم يتم تكوين shared secret بعد، أنشئ token عبر `openclaw doctor --generate-gateway-token`.

    **ليس على localhost:**

    - **Tailscale Serve** (موصى به): أبقِ الربط على loopback، وشغّل `openclaw gateway --tailscale serve`، وافتح `https://<magicdns>/`. إذا كانت `gateway.auth.allowTailscale` تساوي `true`، فإن رؤوس الهوية تفي بمصادقة Control UI/WebSocket (من دون لصق shared secret، مع افتراض الثقة في مضيف gateway)؛ ولا تزال واجهات HTTP APIs تتطلب مصادقة shared-secret ما لم تستخدم عمدًا `none` في private-ingress أو مصادقة HTTP عبر trusted-proxy.
      تتم موازنة محاولات Serve السيئة المتزامنة من العميل نفسه قبل أن يسجل محدِّد المصادقة الفاشلة هذه المحاولات، لذلك قد تُظهر المحاولة السيئة الثانية بالفعل `retry later`.
    - **ربط tailnet**: شغّل `openclaw gateway --bind tailnet --token "<token>"` (أو اضبط مصادقة password)، وافتح `http://<tailscale-ip>:18789/`، ثم الصق shared secret المطابق في إعدادات لوحة التحكم.
    - **وكيل عكسي مدرك للهوية**: أبقِ Gateway خلف trusted proxy غير loopback، واضبط `gateway.auth.mode: "trusted-proxy"`، ثم افتح عنوان URL للوكيل.
    - **نفق SSH**: `ssh -N -L 18789:127.0.0.1:18789 user@host` ثم افتح `http://127.0.0.1:18789/`. لا تزال مصادقة shared-secret مطبقة عبر النفق؛ الصق token أو password المكوّن إذا طُلب منك ذلك.

    راجع [لوحة التحكم](/web/dashboard) و[أسطح الويب](/web) لمعرفة تفاصيل أوضاع الربط والمصادقة.

  </Accordion>

  <Accordion title="لماذا توجد إعداداتا موافقة exec لموافقات الدردشة؟">
    إنهما تتحكمان في طبقتين مختلفتين:

    - `approvals.exec`: يمرر مطالبات الموافقة إلى وجهات الدردشة
    - `channels.<channel>.execApprovals`: يجعل تلك القناة تعمل كعميل موافقة أصلي لموافقات exec

    تبقى سياسة exec على المضيف هي بوابة الموافقة الحقيقية. إعدادات الدردشة تتحكم فقط في مكان
    ظهور مطالبات الموافقة وكيفية تمكين الأشخاص من الرد عليها.

    في معظم الإعدادات **لا** تحتاج إلى الاثنين معًا:

    - إذا كانت الدردشة تدعم الأوامر والردود بالفعل، فإن `/approve` في نفس الدردشة يعمل عبر المسار المشترك.
    - إذا كانت قناة أصلية مدعومة تستطيع استنتاج الموافقين بأمان، فإن OpenClaw يفعّل الآن تلقائيًا الموافقات الأصلية بنمط DM-first عندما تكون `channels.<channel>.execApprovals.enabled` غير مضبوطة أو مضبوطة على `"auto"`.
    - عندما تكون بطاقات/أزرار الموافقة الأصلية متاحة، تكون واجهة المستخدم الأصلية هي المسار الأساسي؛ ويجب على الوكيل تضمين أمر `/approve` يدويًا فقط إذا كانت نتيجة الأداة تقول إن موافقات الدردشة غير متاحة أو أن الموافقة اليدوية هي المسار الوحيد.
    - استخدم `approvals.exec` فقط عندما يلزم أيضًا تمرير المطالبات إلى دردشات أخرى أو غرف تشغيل مخصصة.
    - استخدم `channels.<channel>.execApprovals.target: "channel"` أو `"both"` فقط عندما تريد صراحةً نشر مطالبات الموافقة مرة أخرى في الغرفة/الموضوع الأصلي.
    - موافقات plugin منفصلة مرة أخرى: فهي تستخدم `/approve` في نفس الدردشة افتراضيًا، مع تمرير اختياري عبر `approvals.plugin`، وفقط بعض القنوات الأصلية تُبقي معالجة plugin-approval-native فوق ذلك.

    باختصار: التمرير مخصص للتوجيه، وإعدادات العميل الأصلي مخصصة لتجربة مستخدم أغنى خاصة بالقناة.
    راجع [موافقات Exec](/ar/tools/exec-approvals).

  </Accordion>

  <Accordion title="ما وقت التشغيل الذي أحتاجه؟">
    يلزم Node **>= 22**. ويُنصح باستخدام `pnpm`. ولا يُنصح باستخدام Bun مع Gateway.
  </Accordion>

  <Accordion title="هل يعمل على Raspberry Pi؟">
    نعم. Gateway خفيفة - وتذكر المستندات أن **512MB-1GB RAM** و**نواة واحدة** وحوالي **500MB**
    من مساحة القرص تكفي للاستخدام الشخصي، وتشير إلى أن **Raspberry Pi 4 يمكنه تشغيله**.

    إذا كنت تريد هامشًا إضافيًا (السجلات، الوسائط، خدمات أخرى)، فإن **2GB موصى بها**، لكنها
    ليست حدًا أدنى صارمًا.

    نصيحة: يمكن لجهاز Pi/VPS صغير استضافة Gateway، ويمكنك إقران **nodes** على حاسوبك المحمول/هاتفك من أجل
    الشاشة/الكاميرا/الـ canvas المحلية أو تنفيذ الأوامر. راجع [Nodes](/ar/nodes).

  </Accordion>

  <Accordion title="هل هناك نصائح لتثبيت Raspberry Pi؟">
    باختصار: يعمل، لكن توقّع بعض الزوايا الخشنة.

    - استخدم نظام تشغيل **64-bit** وأبقِ Node >= 22.
    - فضّل **تثبيت git القابل للاختراق** حتى تتمكن من رؤية السجلات والتحديث بسرعة.
    - ابدأ بدون قنوات/Skills، ثم أضفها واحدة تلو الأخرى.
    - إذا واجهت مشكلات غريبة تتعلق بالثنائيات، فعادةً ما تكون مشكلة **توافق ARM**.

    المستندات: [Linux](/ar/platforms/linux)، [التثبيت](/ar/install).

  </Accordion>

  <Accordion title="إنه عالق على wake up my friend / الإدخال الأولي لا يكتمل. ماذا الآن؟">
    تعتمد هذه الشاشة على أن يكون Gateway قابلاً للوصول ومصادقًا عليه. كما يرسل TUI
    "Wake up, my friend!" تلقائيًا عند أول hatch. إذا رأيت هذا السطر **من دون رد**
    وبقيت الرموز عند 0، فهذا يعني أن الوكيل لم يعمل مطلقًا.

    1. أعد تشغيل Gateway:

    ```bash
    openclaw gateway restart
    ```

    2. تحقق من الحالة + المصادقة:

    ```bash
    openclaw status
    openclaw models status
    openclaw logs --follow
    ```

    3. إذا ظل معلقًا، شغّل:

    ```bash
    openclaw doctor
    ```

    إذا كان Gateway بعيدًا، فتأكد من أن اتصال tunnel/Tailscale قائم وأن UI
    يشير إلى Gateway الصحيح. راجع [الوصول البعيد](/ar/gateway/remote).

  </Accordion>

  <Accordion title="هل يمكنني ترحيل الإعداد إلى جهاز جديد (Mac mini) من دون إعادة الإدخال الأولي؟">
    نعم. انسخ **دليل الحالة** و**مساحة العمل**، ثم شغّل Doctor مرة واحدة. هذا
    يُبقي bot لديك "كما هو تمامًا" (الذاكرة، محفوظات الجلسات، المصادقة، وحالة
    القناة) طالما أنك تنسخ **الموقعين** معًا:

    1. ثبّت OpenClaw على الجهاز الجديد.
    2. انسخ `$OPENCLAW_STATE_DIR` (الافتراضي: `~/.openclaw`) من الجهاز القديم.
    3. انسخ مساحة العمل لديك (الافتراضي: `~/.openclaw/workspace`).
    4. شغّل `openclaw doctor` وأعد تشغيل خدمة Gateway.

    يحافظ ذلك على الإعدادات وملفات تعريف المصادقة وبيانات اعتماد WhatsApp والجلسات والذاكرة. إذا كنت في
    الوضع البعيد، فتذكّر أن مضيف gateway هو الذي يملك مخزن الجلسات ومساحة العمل.

    **مهم:** إذا كنت فقط تقوم بعمل commit/push لمساحة العمل إلى GitHub، فأنت تنسخ
    احتياطيًا **ملفات الذاكرة + ملفات bootstrap**، لكن **ليس** محفوظات الجلسات أو المصادقة. هذه تعيش
    ضمن `~/.openclaw/` (مثلًا `~/.openclaw/agents/<agentId>/sessions/`).

    ذو صلة: [الترحيل](/ar/install/migrating)، [مكان وجود الأشياء على القرص](#مكان-وجود-الأشياء-على-القرص)،
    [مساحة عمل الوكيل](/ar/concepts/agent-workspace)، [Doctor](/ar/gateway/doctor)،
    [الوضع البعيد](/ar/gateway/remote).

  </Accordion>

  <Accordion title="أين أرى ما الجديد في أحدث إصدار؟">
    تحقق من سجل التغييرات على GitHub:
    [https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md)

    أحدث الإدخالات تكون في الأعلى. إذا كان القسم العلوي معنونًا **Unreleased**، فالقسم المؤرخ التالي
    هو أحدث إصدار تم شحنه. تُجمع الإدخالات تحت **Highlights** و**Changes** و
    **Fixes** (مع أقسام للمستندات/أمور أخرى عند الحاجة).

  </Accordion>

  <Accordion title="لا يمكن الوصول إلى docs.openclaw.ai (خطأ SSL)">
    تقوم بعض اتصالات Comcast/Xfinity بحظر `docs.openclaw.ai` بشكل غير صحيح عبر Xfinity
    Advanced Security. قم بتعطيله أو أضف `docs.openclaw.ai` إلى allowlist ثم أعد المحاولة.
    نرجو مساعدتنا في فك الحظر عبر الإبلاغ هنا: [https://spa.xfinity.com/check_url_status](https://spa.xfinity.com/check_url_status).

    إذا ظل يتعذر عليك الوصول إلى الموقع، فالمستندات معكوسة على GitHub:
    [https://github.com/openclaw/openclaw/tree/main/docs](https://github.com/openclaw/openclaw/tree/main/docs)

  </Accordion>

  <Accordion title="الفرق بين stable وbeta">
    **Stable** و**beta** هما **npm dist-tags**، وليسا خطّي شيفرة منفصلين:

    - `latest` = مستقر
    - `beta` = إصدار مبكر للاختبار

    عادةً، يصل الإصدار المستقر إلى **beta** أولًا، ثم تنقل خطوة ترقية
    صريحة ذلك الإصدار نفسه إلى `latest`. ويمكن للمشرفين أيضًا
    النشر مباشرةً إلى `latest` عند الحاجة. ولهذا قد يشير beta وstable
    إلى **الإصدار نفسه** بعد الترقية.

    اعرف ما الذي تغيّر:
    [https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md)

    لمعرفة أوامر التثبيت المختصرة والفرق بين beta وdev، راجع القسم المطوي أدناه.

  </Accordion>

  <Accordion title="كيف أثبت إصدار beta وما الفرق بين beta وdev؟">
    **Beta** هو npm dist-tag المسمى `beta` (وقد يطابق `latest` بعد الترقية).
    **Dev** هو الرأس المتحرك لفرع `main` (git)؛ وعند نشره يستخدم npm dist-tag `dev`.

    أوامر مختصرة (macOS/Linux):

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --beta
    ```

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    برنامج تثبيت Windows (PowerShell):
    [https://openclaw.ai/install.ps1](https://openclaw.ai/install.ps1)

    مزيد من التفاصيل: [قنوات التطوير](/ar/install/development-channels) و[إشارات برنامج التثبيت](/ar/install/installer).

  </Accordion>

  <Accordion title="كيف أجرّب أحدث الأجزاء؟">
    هناك خياران:

    1. **قناة Dev (نسخة git):**

    ```bash
    openclaw update --channel dev
    ```

    ينقلك هذا إلى الفرع `main` ويحدّث من المصدر.

    2. **تثبيت قابل للاختراق (من موقع برنامج التثبيت):**

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    يمنحك هذا مستودعًا محليًا يمكنك تعديله، ثم تحديثه عبر git.

    إذا كنت تفضل نسخة نظيفة يدويًا، فاستخدم:

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    ```

    المستندات: [التحديث](/cli/update)، [قنوات التطوير](/ar/install/development-channels)،
    [التثبيت](/ar/install).

  </Accordion>

  <Accordion title="كم يستغرق التثبيت والإدخال الأولي عادةً؟">
    دليل تقريبي:

    - **التثبيت:** 2-5 دقائق
    - **الإدخال الأولي:** 5-15 دقيقة بحسب عدد القنوات/النماذج التي تضبطها

    إذا علق، فاستخدم [تعطل برنامج التثبيت](#البدء-السريع-وإعداد-التشغيل-الأول)
    وحلقة التصحيح السريعة في [أنا عالق](#البدء-السريع-وإعداد-التشغيل-الأول).

  </Accordion>

  <Accordion title="برنامج التثبيت عالق؟ كيف أحصل على مزيد من الملاحظات؟">
    أعد تشغيل برنامج التثبيت مع **مخرجات تفصيلية**:

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --verbose
    ```

    تثبيت beta مع verbose:

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --beta --verbose
    ```

    لتثبيت git قابل للاختراق:

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git --verbose
    ```

    النظير في Windows (PowerShell):

    ```powershell
    # install.ps1 لا يحتوي بعد على علامة -Verbose مخصصة.
    Set-PSDebug -Trace 1
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -NoOnboard
    Set-PSDebug -Trace 0
    ```

    مزيد من الخيارات: [إشارات برنامج التثبيت](/ar/install/installer).

  </Accordion>

  <Accordion title="تثبيت Windows يقول git not found أو openclaw not recognized">
    مشكلتان شائعتان في Windows:

    **1) خطأ npm spawn git / git not found**

    - ثبّت **Git for Windows** وتأكد من أن `git` موجود على PATH.
    - أغلق PowerShell وافتحه مجددًا، ثم أعد تشغيل برنامج التثبيت.

    **2) openclaw is not recognized بعد التثبيت**

    - مجلد npm global bin غير موجود على PATH.
    - تحقق من المسار:

      ```powershell
      npm config get prefix
      ```

    - أضف ذلك الدليل إلى PATH الخاص بالمستخدم لديك (لا حاجة إلى اللاحقة `\bin` على Windows؛ في معظم الأنظمة يكون `%AppData%\npm`).
    - أغلق PowerShell وافتحه مجددًا بعد تحديث PATH.

    إذا أردت أكثر إعداد Windows سلاسة، فاستخدم **WSL2** بدل Windows الأصلي.
    المستندات: [Windows](/ar/platforms/windows).

  </Accordion>

  <Accordion title="تُظهر مخرجات exec على Windows نصًا صينيًا مشوّهًا - ماذا أفعل؟">
    هذا عادةً بسبب عدم تطابق code page في الطرفية على Windows الأصلي.

    الأعراض:

    - مخرجات `system.run`/`exec` تعرض النص الصيني بشكل مشوّه
    - الأمر نفسه يبدو جيدًا في ملف تعريف طرفية آخر

    حل سريع في PowerShell:

    ```powershell
    chcp 65001
    [Console]::InputEncoding = [System.Text.UTF8Encoding]::new($false)
    [Console]::OutputEncoding = [System.Text.UTF8Encoding]::new($false)
    $OutputEncoding = [System.Text.UTF8Encoding]::new($false)
    ```

    ثم أعد تشغيل Gateway وأعد محاولة الأمر:

    ```powershell
    openclaw gateway restart
    ```

    إذا ما زلت تستطيع إعادة إنتاج المشكلة على أحدث إصدار من OpenClaw، فتابعها/أبلغ عنها في:

    - [Issue #30640](https://github.com/openclaw/openclaw/issues/30640)

  </Accordion>

  <Accordion title="لم تجبني المستندات عن سؤالي - كيف أحصل على إجابة أفضل؟">
    استخدم **تثبيت git القابل للاختراق** حتى تكون لديك الشيفرة والمستندات كاملة محليًا، ثم اسأل
    bot لديك (أو Claude/Codex) _من ذلك المجلد_ حتى يتمكن من قراءة المستودع والإجابة بدقة.

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    مزيد من التفاصيل: [التثبيت](/ar/install) و[إشارات برنامج التثبيت](/ar/install/installer).

  </Accordion>

  <Accordion title="كيف أثبت OpenClaw على Linux؟">
    الإجابة المختصرة: اتبع دليل Linux، ثم شغّل الإدخال الأولي.

    - المسار السريع لـ Linux + تثبيت الخدمة: [Linux](/ar/platforms/linux).
    - شرح كامل: [البدء](/ar/start/getting-started).
    - برنامج التثبيت + التحديثات: [التثبيت والتحديثات](/ar/install/updating).

  </Accordion>

  <Accordion title="كيف أثبت OpenClaw على VPS؟">
    أي Linux VPS يعمل. ثبّت على الخادم، ثم استخدم SSH/Tailscale للوصول إلى Gateway.

    الأدلة: [exe.dev](/ar/install/exe-dev)، [Hetzner](/ar/install/hetzner)، [Fly.io](/ar/install/fly).
    الوصول البعيد: [Gateway البعيد](/ar/gateway/remote).

  </Accordion>

  <Accordion title="أين توجد أدلة التثبيت السحابي/VPS؟">
    نحتفظ **بمركز استضافة** يضم المزوّدين الشائعين. اختر واحدًا واتبع الدليل:

    - [استضافة VPS](/ar/vps) (كل المزوّدين في مكان واحد)
    - [Fly.io](/ar/install/fly)
    - [Hetzner](/ar/install/hetzner)
    - [exe.dev](/ar/install/exe-dev)

    كيف يعمل هذا في السحابة: **Gateway يعمل على الخادم**، وتصل إليه
    من حاسوبك المحمول/هاتفك عبر Control UI (أو Tailscale/SSH). تعيش حالتك + مساحة العمل
    على الخادم، لذا اعتبر المضيف مصدر الحقيقة وقم بنسخه احتياطيًا.

    يمكنك إقران **nodes** (Mac/iOS/Android/headless) مع Gateway السحابي هذا للوصول إلى
    الشاشة/الكاميرا/الـ canvas المحلية أو تشغيل الأوامر على حاسوبك المحمول مع إبقاء
    Gateway في السحابة.

    المركز: [المنصات](/ar/platforms). الوصول البعيد: [Gateway البعيد](/ar/gateway/remote).
    Nodes: [Nodes](/ar/nodes)، [CLI الخاص بـ Nodes](/cli/nodes).

  </Accordion>

  <Accordion title="هل يمكنني أن أطلب من OpenClaw أن يحدّث نفسه؟">
    الإجابة المختصرة: **ممكن، لكنه غير موصى به**. يمكن لتدفق التحديث أن يعيد تشغيل
    Gateway (ما يؤدي إلى قطع الجلسة النشطة)، وقد يحتاج إلى نسخة git نظيفة، وقد
    يطلب تأكيدًا. الأكثر أمانًا: شغّل التحديثات من shell بصفتك المشغّل.

    استخدم CLI:

    ```bash
    openclaw update
    openclaw update status
    openclaw update --channel stable|beta|dev
    openclaw update --tag <dist-tag|version>
    openclaw update --no-restart
    ```

    إذا كان لا بد من الأتمتة من agent:

    ```bash
    openclaw update --yes --no-restart
    openclaw gateway restart
    ```

    المستندات: [التحديث](/cli/update)، [التحديث](/ar/install/updating).

  </Accordion>

  <Accordion title="ماذا يفعل الإدخال الأولي فعليًا؟">
    `openclaw onboard` هو مسار الإعداد الموصى به. في **الوضع المحلي** يرشدك خلال:

    - **إعداد النموذج/المصادقة** (OAuth للمزوّد، مفاتيح API، Anthropic setup-token، إضافةً إلى خيارات النماذج المحلية مثل LM Studio)
    - موقع **Workspace** + ملفات bootstrap
    - **إعدادات Gateway** (bind/port/auth/tailscale)
    - **القنوات** (WhatsApp وTelegram وDiscord وMattermost وSignal وiMessage، إضافةً إلى channel plugins المضمّنة مثل QQ Bot)
    - **تثبيت daemon** (LaunchAgent على macOS؛ ووحدة systemd للمستخدم على Linux/WSL2)
    - **فحوصات الصحة** واختيار **Skills**

    كما يحذّر إذا كان النموذج الذي قمت بتكوينه غير معروف أو يفتقد إلى المصادقة.

  </Accordion>

  <Accordion title="هل أحتاج إلى اشتراك Claude أو OpenAI لتشغيل هذا؟">
    لا. يمكنك تشغيل OpenClaw باستخدام **مفاتيح API** (Anthropic/OpenAI/وغيرهما) أو باستخدام
    **نماذج محلية فقط** بحيث تبقى بياناتك على جهازك. الاشتراكات (Claude
    Pro/Max أو OpenAI Codex) هي طرق اختيارية للمصادقة مع هؤلاء المزوّدين.

    بالنسبة إلى Anthropic في OpenClaw، فالتقسيم العملي هو:

    - **Anthropic API key**: فوترة عادية عبر Anthropic API
    - **Claude CLI / Claude subscription auth في OpenClaw**: أخبرنا موظفو Anthropic
      أن هذا الاستخدام مسموح به مجددًا، ويتعامل OpenClaw مع استخدام `claude -p`
      على أنه معتمد لهذا التكامل ما لم تنشر Anthropic
      سياسة جديدة

    بالنسبة إلى مضيفي gateway طويلة الأمد، تظل مفاتيح Anthropic API
    الإعداد الأكثر قابلية للتنبؤ. كما أن OpenAI Codex OAuth مدعوم صراحةً للأدوات
    الخارجية مثل OpenClaw.

    يدعم OpenClaw أيضًا خيارات اشتراك مستضافة أخرى، منها
    **Qwen Cloud Coding Plan** و**MiniMax Coding Plan** و
    **Z.AI / GLM Coding Plan**.

    المستندات: [Anthropic](/ar/providers/anthropic)، [OpenAI](/ar/providers/openai)،
    [Qwen Cloud](/ar/providers/qwen)،
    [MiniMax](/ar/providers/minimax)، [نماذج GLM](/ar/providers/glm)،
    [النماذج المحلية](/ar/gateway/local-models)، [النماذج](/ar/concepts/models).

  </Accordion>

  <Accordion title="هل يمكنني استخدام اشتراك Claude Max بدون مفتاح API؟">
    نعم.

    أخبرنا موظفو Anthropic أن استخدام Claude CLI بنمط OpenClaw مسموح به مجددًا، لذا
    يتعامل OpenClaw مع مصادقة اشتراك Claude واستخدام `claude -p` على أنهما معتمدان
    لهذا التكامل ما لم تنشر Anthropic سياسة جديدة. وإذا أردت
    أكثر إعداد جانب خادم قابلية للتنبؤ، فاستخدم مفتاح Anthropic API بدلًا من ذلك.

  </Accordion>

  <Accordion title="هل تدعمون مصادقة اشتراك Claude (Claude Pro أو Max)؟">
    نعم.

    أخبرنا موظفو Anthropic أن هذا الاستخدام مسموح به مجددًا، لذا يتعامل OpenClaw مع
    إعادة استخدام Claude CLI واستخدام `claude -p` على أنهما معتمدان لهذا التكامل
    ما لم تنشر Anthropic سياسة جديدة.

    لا يزال Anthropic setup-token متاحًا كمسار token مدعوم في OpenClaw، لكن OpenClaw يفضّل الآن إعادة استخدام Claude CLI و`claude -p` عند توفرهما.
    أما لأحمال الإنتاج أو متعددة المستخدمين، فلا تزال مصادقة مفتاح Anthropic API
    الخيار الأكثر أمانًا وقابلية للتنبؤ. وإذا كنت تريد خيارات مستضافة
    أخرى بنمط الاشتراك داخل OpenClaw، فراجع [OpenAI](/ar/providers/openai)، [Qwen / Model
    Cloud](/ar/providers/qwen)، [MiniMax](/ar/providers/minimax)، و[نماذج GLM](/ar/providers/glm).

  </Accordion>

<a id="why-am-i-seeing-http-429-ratelimiterror-from-anthropic"></a>
<Accordion title="لماذا أرى HTTP 429 rate_limit_error من Anthropic؟">
هذا يعني أن **حصة/حد المعدل لدى Anthropic** قد استُنفدت للفترة الحالية. إذا كنت
تستخدم **Claude CLI**، فانتظر حتى تُعاد تهيئة الفترة أو قم بترقية خطتك. وإذا كنت
تستخدم **Anthropic API key**، فتحقق من Anthropic Console
بالنسبة للاستخدام/الفوترة وارفع الحدود عند الحاجة.

    إذا كانت الرسالة تحديدًا:
    `Extra usage is required for long context requests`، فهذا يعني أن الطلب يحاول استخدام
    النسخة التجريبية لسياق 1M لدى Anthropic (`context1m: true`). وهذا لا يعمل إلا عندما تكون
    بيانات الاعتماد لديك مؤهلة لفوترة السياق الطويل (فوترة عبر API key أو
    مسار Claude-login في OpenClaw مع تفعيل Extra Usage).

    نصيحة: اضبط **نموذجًا احتياطيًا** حتى يتمكن OpenClaw من الاستمرار في الرد أثناء تقييد مزوّد ما بالمعدل.
    راجع [النماذج](/cli/models)، [OAuth](/ar/concepts/oauth)، و
    [/gateway/troubleshooting#anthropic-429-extra-usage-required-for-long-context](/ar/gateway/troubleshooting#anthropic-429-extra-usage-required-for-long-context).

  </Accordion>

  <Accordion title="هل AWS Bedrock مدعوم؟">
    نعم. يحتوي OpenClaw على مزوّد **Amazon Bedrock (Converse)** مضمّن. عند وجود محددات AWS env، يستطيع OpenClaw اكتشاف كتالوج Bedrock النصي/المتدفق ودمجه كمزوّد ضمني `amazon-bedrock`؛ وإلا يمكنك تفعيل `plugins.entries.amazon-bedrock.config.discovery.enabled` صراحةً أو إضافة إدخال مزوّد يدوي. راجع [Amazon Bedrock](/ar/providers/bedrock) و[مزودو النماذج](/ar/providers/models). وإذا كنت تفضّل تدفق مفاتيح مُدارًا، فما يزال الوكيل المتوافق مع OpenAI أمام Bedrock خيارًا صالحًا.
  </Accordion>

  <Accordion title="كيف تعمل مصادقة Codex؟">
    يدعم OpenClaw **OpenAI Code (Codex)** عبر OAuth (تسجيل الدخول عبر ChatGPT). يمكن للإدخال الأولي تشغيل تدفق OAuth وسيضبط النموذج الافتراضي على `openai-codex/gpt-5.4` عندما يكون ذلك مناسبًا. راجع [مزودو النماذج](/ar/concepts/model-providers) و[الإدخال الأولي (CLI)](/ar/start/wizard).
  </Accordion>

  <Accordion title="لماذا لا يؤدي ChatGPT GPT-5.4 إلى فتح openai/gpt-5.4 في OpenClaw؟">
    يتعامل OpenClaw مع المسارين بشكل منفصل:

    - `openai-codex/gpt-5.4` = ChatGPT/Codex OAuth
    - `openai/gpt-5.4` = OpenAI Platform API مباشرةً

    في OpenClaw، يتم توصيل تسجيل الدخول عبر ChatGPT/Codex إلى مسار `openai-codex/*`,
    وليس إلى المسار المباشر `openai/*`. إذا كنت تريد المسار المباشر عبر API في
    OpenClaw، فاضبط `OPENAI_API_KEY` (أو إعدادات مزوّد OpenAI المكافئة).
    وإذا كنت تريد تسجيل الدخول عبر ChatGPT/Codex في OpenClaw، فاستخدم `openai-codex/*`.

  </Accordion>

  <Accordion title="لماذا قد تختلف حدود Codex OAuth عن ChatGPT على الويب؟">
    يستخدم `openai-codex/*` مسار Codex OAuth، ونوافذ الحصص القابلة للاستخدام فيه
    تُدار من OpenAI وتعتمد على الخطة. عمليًا، قد تختلف هذه الحدود عن
    تجربة موقع/تطبيق ChatGPT، حتى عندما يكون الاثنان مرتبطين بالحساب نفسه.

    يمكن لـ OpenClaw عرض نوافذ الاستخدام/الحصة المرئية حاليًا لدى المزوّد في
    `openclaw models status`، لكنه لا يخترع ولا يوحّد استحقاقات ChatGPT على الويب
    ضمن الوصول المباشر إلى API. إذا كنت تريد مسار الفوترة/الحدود المباشر عبر OpenAI Platform،
    فاستخدم `openai/*` مع مفتاح API.

  </Accordion>

  <Accordion title="هل تدعمون مصادقة اشتراك OpenAI (Codex OAuth)؟">
    نعم. يدعم OpenClaw بالكامل **OpenAI Code (Codex) subscription OAuth**.
    تسمح OpenAI صراحةً باستخدام subscription OAuth في الأدوات/سير العمل الخارجية
    مثل OpenClaw. ويمكن للإدخال الأولي تشغيل تدفق OAuth نيابةً عنك.

    راجع [OAuth](/ar/concepts/oauth)، [مزودو النماذج](/ar/concepts/model-providers)، و[الإدخال الأولي (CLI)](/ar/start/wizard).

  </Accordion>

  <Accordion title="كيف أضبط Gemini CLI OAuth؟">
    يستخدم Gemini CLI **تدفق مصادقة plugin**، وليس client id أو secret في `openclaw.json`.

    الخطوات:

    1. ثبّت Gemini CLI محليًا بحيث يكون `gemini` موجودًا على `PATH`
       - Homebrew: `brew install gemini-cli`
       - npm: `npm install -g @google/gemini-cli`
    2. فعّل plugin: `openclaw plugins enable google`
    3. سجّل الدخول: `openclaw models auth login --provider google-gemini-cli --set-default`
    4. النموذج الافتراضي بعد تسجيل الدخول: `google-gemini-cli/gemini-3-flash-preview`
    5. إذا فشلت الطلبات، فاضبط `GOOGLE_CLOUD_PROJECT` أو `GOOGLE_CLOUD_PROJECT_ID` على مضيف gateway

    يخزن هذا رموز OAuth في ملفات تعريف المصادقة على مضيف gateway. التفاصيل: [مزودو النماذج](/ar/concepts/model-providers).

  </Accordion>

  <Accordion title="هل النموذج المحلي مناسب للدردشات العادية؟">
    غالبًا لا. يحتاج OpenClaw إلى سياق كبير + أمان قوي؛ البطاقات الصغيرة تقطع السياق وتسرّب. وإذا اضطررت، فشغّل **أكبر** بنية نموذج يمكنك تشغيلها محليًا (LM Studio) وراجع [/gateway/local-models](/ar/gateway/local-models). النماذج الأصغر/الأكثر كمّية تزيد من خطر prompt injection - راجع [الأمان](/ar/gateway/security).
  </Accordion>

  <Accordion title="كيف أبقي حركة مرور النماذج المستضافة داخل منطقة محددة؟">
    اختر نقاط نهاية مثبّتة على المنطقة. يوفّر OpenRouter خيارات مستضافة في الولايات المتحدة لـ MiniMax وKimi وGLM؛ اختر المتغير المستضاف في الولايات المتحدة لإبقاء البيانات داخل المنطقة. ولا يزال بإمكانك إدراج Anthropic/OpenAI إلى جانب هذه الخيارات باستخدام `models.mode: "merge"` بحيث تبقى النماذج الاحتياطية متاحة مع احترام المزوّد الإقليمي الذي تختاره.
  </Accordion>

  <Accordion title="هل يجب أن أشتري Mac Mini لتثبيت هذا؟">
    لا. يعمل OpenClaw على macOS أو Linux (وWindows عبر WSL2). إن Mac mini اختياري -
    فبعض الأشخاص يشترونه كمضيف يعمل دائمًا، لكن VPS صغيرًا أو خادمًا منزليًا أو جهازًا من فئة Raspberry Pi يعمل أيضًا.

    أنت تحتاج إلى Mac فقط **لأدوات macOS فقط**. بالنسبة إلى iMessage، استخدم [BlueBubbles](/ar/channels/bluebubbles) (موصى به) - يعمل خادم BlueBubbles على أي جهاز Mac، بينما يمكن لـ Gateway أن يعمل على Linux أو في مكان آخر. وإذا كنت تريد أدوات أخرى خاصة بـ macOS فقط، فشغّل Gateway على Mac أو اقترن بعقدة macOS.

    المستندات: [BlueBubbles](/ar/channels/bluebubbles)، [Nodes](/ar/nodes)، [الوضع البعيد على Mac](/ar/platforms/mac/remote).

  </Accordion>

  <Accordion title="هل أحتاج إلى Mac mini لدعم iMessage؟">
    أنت تحتاج إلى **أي جهاز macOS** مسجّل الدخول إلى Messages. ولا **يشترط** أن يكون Mac mini -
    أي جهاز Mac يعمل. **استخدم [BlueBubbles](/ar/channels/bluebubbles)** (موصى به) بالنسبة إلى iMessage - يعمل خادم BlueBubbles على macOS، بينما يمكن لـ Gateway أن يعمل على Linux أو في مكان آخر.

    إعدادات شائعة:

    - شغّل Gateway على Linux/VPS، وشغّل خادم BlueBubbles على أي جهاز Mac مسجّل الدخول إلى Messages.
    - شغّل كل شيء على جهاز Mac إذا أردت أبسط إعداد على جهاز واحد.

    المستندات: [BlueBubbles](/ar/channels/bluebubbles)، [Nodes](/ar/nodes)،
    [الوضع البعيد على Mac](/ar/platforms/mac/remote).

  </Accordion>

  <Accordion title="إذا اشتريت Mac mini لتشغيل OpenClaw، هل يمكنني توصيله بـ MacBook Pro؟">
    نعم. يمكن لـ **Mac mini أن يشغّل Gateway**، ويمكن لـ MacBook Pro أن يتصل كـ
    **node** (جهاز مرافق). لا تشغّل Nodes الـ Gateway - بل توفر
    قدرات إضافية مثل screen/camera/canvas و`system.run` على ذلك الجهاز.

    النمط الشائع:

    - Gateway على Mac mini (يعمل دائمًا).
    - يشغّل MacBook Pro تطبيق macOS أو مضيف node ويقترن بالـ Gateway.
    - استخدم `openclaw nodes status` / `openclaw nodes list` لمشاهدته.

    المستندات: [Nodes](/ar/nodes)، [CLI الخاص بـ Nodes](/cli/nodes).

  </Accordion>

  <Accordion title="هل يمكنني استخدام Bun؟">
    لا يُنصح باستخدام Bun. نرى أخطاء وقت تشغيل، خاصةً مع WhatsApp وTelegram.
    استخدم **Node** من أجل gateways مستقرة.

    إذا كنت لا تزال تريد التجريب باستخدام Bun، فافعل ذلك على gateway غير إنتاجية
    ومن دون WhatsApp/Telegram.

  </Accordion>

  <Accordion title="Telegram: ماذا يوضع في allowFrom؟">
    `channels.telegram.allowFrom` هو **معرّف مستخدم Telegram للمرسل البشري** (رقمي). وليس اسم مستخدم bot.

    يقبل الإدخال الأولي إدخال `@username` ويحوّله إلى معرّف رقمي، لكن تفويض OpenClaw يستخدم المعرفات الرقمية فقط.

    الطريقة الأكثر أمانًا (من دون bot طرف ثالث):

    - أرسل رسالة خاصة إلى bot لديك، ثم شغّل `openclaw logs --follow` واقرأ `from.id`.

    Bot API الرسمي:

    - أرسل رسالة خاصة إلى bot لديك، ثم استدعِ `https://api.telegram.org/bot<bot_token>/getUpdates` واقرأ `message.from.id`.

    طرف ثالث (أقل خصوصية):

    - أرسل رسالة خاصة إلى `@userinfobot` أو `@getidsbot`.

    راجع [/channels/telegram](/ar/channels/telegram#access-control-and-activation).

  </Accordion>

  <Accordion title="هل يمكن لعدة أشخاص استخدام رقم WhatsApp واحد مع نُسخ OpenClaw مختلفة؟">
    نعم، عبر **التوجيه متعدد الوكلاء**. اربط **DM** الخاصة بكل مُرسِل في WhatsApp (القرين `kind: "direct"`، والمرسل بصيغة E.164 مثل `+15551234567`) بـ `agentId` مختلف، حتى يحصل كل شخص على مساحة عمله ومخزن جلساته الخاصين. وستظل الردود تصدر من **حساب WhatsApp نفسه**، كما أن التحكم في الوصول إلى DM (`channels.whatsapp.dmPolicy` / `channels.whatsapp.allowFrom`) يكون عامًا لكل حساب WhatsApp. راجع [التوجيه متعدد الوكلاء](/ar/concepts/multi-agent) و[WhatsApp](/ar/channels/whatsapp).
  </Accordion>

  <Accordion title='هل يمكنني تشغيل وكيل "دردشة سريعة" ووكيل "Opus للبرمجة"؟'>
    نعم. استخدم التوجيه متعدد الوكلاء: امنح كل وكيل نموذجه الافتراضي الخاص، ثم اربط المسارات الواردة (حساب المزوّد أو الأقران المحددون) بكل وكيل. يوجد مثال للإعدادات في [التوجيه متعدد الوكلاء](/ar/concepts/multi-agent). راجع أيضًا [النماذج](/ar/concepts/models) و[التكوين](/ar/gateway/configuration).
  </Accordion>

  <Accordion title="هل يعمل Homebrew على Linux؟">
    نعم. يدعم Homebrew نظام Linux (Linuxbrew). إعداد سريع:

    ```bash
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
    echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"' >> ~/.profile
    eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
    brew install <formula>
    ```

    إذا كنت تشغّل OpenClaw عبر systemd، فتأكد من أن PATH الخاص بالخدمة يتضمن `/home/linuxbrew/.linuxbrew/bin` (أو بادئة brew لديك) بحيث يتم العثور على الأدوات المثبتة عبر `brew` في shell غير تسجيل الدخول.
    كما أن الإصدارات الحديثة تسبق أيضًا أدلة المستخدم الشائعة bin على خدمات Linux systemd (مثل `~/.local/bin` و`~/.npm-global/bin` و`~/.local/share/pnpm` و`~/.bun/bin`) وتحترم `PNPM_HOME` و`NPM_CONFIG_PREFIX` و`BUN_INSTALL` و`VOLTA_HOME` و`ASDF_DATA_DIR` و`NVM_DIR` و`FNM_DIR` عندما تكون مضبوطة.

  </Accordion>

  <Accordion title="الفرق بين تثبيت git القابل للاختراق وnpm install">
    - **تثبيت git القابل للاختراق:** نسخة مصدر كاملة، قابلة للتعديل، والأفضل للمساهمين.
      أنت تشغّل البناء محليًا ويمكنك ترقيع الشيفرة/المستندات.
    - **npm install:** تثبيت CLI عام، بدون مستودع، والأفضل لمن يريد "تشغيله فقط."
      تأتي التحديثات من npm dist-tags.

    المستندات: [البدء](/ar/start/getting-started)، [التحديث](/ar/install/updating).

  </Accordion>

  <Accordion title="هل يمكنني التبديل بين تثبيتات npm وgit لاحقًا؟">
    نعم. ثبّت النكهة الأخرى، ثم شغّل Doctor بحيث تشير خدمة gateway إلى نقطة الدخول الجديدة.
    هذا **لا يحذف بياناتك** - بل يغيّر فقط تثبيت شيفرة OpenClaw. تبقى حالتك
    (`~/.openclaw`) ومساحة عملك (`~/.openclaw/workspace`) كما هما.

    من npm إلى git:

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    openclaw doctor
    openclaw gateway restart
    ```

    من git إلى npm:

    ```bash
    npm install -g openclaw@latest
    openclaw doctor
    openclaw gateway restart
    ```

    يكتشف Doctor عدم تطابق نقطة دخول خدمة gateway ويعرض إعادة كتابة إعدادات الخدمة لتطابق التثبيت الحالي (استخدم `--repair` في الأتمتة).

    نصائح النسخ الاحتياطي: راجع [استراتيجية النسخ الاحتياطي](#مكان-وجود-الأشياء-على-القرص).

  </Accordion>

  <Accordion title="هل يجب أن أشغّل Gateway على حاسوبي المحمول أم على VPS؟">
    الإجابة المختصرة: **إذا كنت تريد موثوقية 24/7، فاستخدم VPS**. وإذا كنت تريد
    أقل احتكاك ولا تمانع السكون/إعادة التشغيل، فشغّله محليًا.

    **الحاسوب المحمول (Gateway محلي)**

    - **الإيجابيات:** لا تكلفة خادم، وصول مباشر إلى الملفات المحلية، نافذة متصفح مرئية.
    - **السلبيات:** السكون/انقطاع الشبكة = انقطاعات، تحديثات/إعادات تشغيل النظام تقطع العمل، ويجب أن يبقى الجهاز مستيقظًا.

    **VPS / السحابة**

    - **الإيجابيات:** يعمل دائمًا، شبكة مستقرة، لا توجد مشكلات سكون الحاسوب المحمول، وأسهل في إبقائه قيد التشغيل.
    - **السلبيات:** غالبًا يعمل headless (استخدم لقطات الشاشة)، وصول بعيد إلى الملفات فقط، ويجب عليك استخدام SSH للتحديثات.

    **ملاحظة خاصة بـ OpenClaw:** تعمل WhatsApp/Telegram/Slack/Mattermost/Discord جميعها جيدًا من VPS. والمفاضلة الحقيقية الوحيدة هي **المتصفح headless** مقابل النافذة المرئية. راجع [المتصفح](/ar/tools/browser).

    **الافتراضي الموصى به:** VPS إذا سبق أن واجهت انقطاعات gateway. أما المحلي فممتاز عندما تستخدم جهاز Mac بنشاط وتريد وصولًا إلى الملفات المحلية أو أتمتة UI مع متصفح مرئي.

  </Accordion>

  <Accordion title="ما مدى أهمية تشغيل OpenClaw على جهاز مخصص؟">
    ليس مطلوبًا، لكنه **موصى به من أجل الموثوقية والعزل**.

    - **مضيف مخصص (VPS/Mac mini/Pi):** يعمل دائمًا، انقطاعات أقل بسبب السكون/إعادة التشغيل، أذونات أنظف، وأسهل في إبقائه قيد التشغيل.
    - **حاسوب محمول/مكتبي مشترك:** مناسب تمامًا للاختبار والاستخدام النشط، لكن توقّع توقفات عند سكون الجهاز أو تحديثه.

    إذا كنت تريد أفضل ما في العالمين، فأبقِ Gateway على مضيف مخصص واقرن حاسوبك المحمول كـ **node** من أجل أدوات الشاشة/الكاميرا/exec المحلية. راجع [Nodes](/ar/nodes).
    ولإرشادات الأمان، اقرأ [الأمان](/ar/gateway/security).

  </Accordion>

  <Accordion title="ما الحد الأدنى لمتطلبات VPS ونظام التشغيل الموصى به؟">
    OpenClaw خفيف. بالنسبة إلى Gateway أساسي + قناة دردشة واحدة:

    - **الحد الأدنى المطلق:** 1 vCPU و1GB RAM وحوالي 500MB قرص.
    - **الموصى به:** 1-2 vCPU و2GB RAM أو أكثر كهامش (للسجلات والوسائط والقنوات المتعددة). قد تكون أدوات Node وأتمتة المتصفح شرهة للموارد.

    نظام التشغيل: استخدم **Ubuntu LTS** (أو أي Debian/Ubuntu حديث). فهذا هو مسار تثبيت Linux الأكثر اختبارًا.

    المستندات: [Linux](/ar/platforms/linux)، [استضافة VPS](/ar/vps).

  </Accordion>

  <Accordion title="هل يمكنني تشغيل OpenClaw داخل VM وما المتطلبات؟">
    نعم. تعامل مع VM كما تتعامل مع VPS: يجب أن تكون قيد التشغيل دائمًا، ويمكن الوصول إليها، وبها ما يكفي من
    الذاكرة لـ Gateway وأي قنوات تقوم بتمكينها.

    الإرشادات الأساسية:

    - **الحد الأدنى المطلق:** 1 vCPU و1GB RAM.
    - **الموصى به:** 2GB RAM أو أكثر إذا كنت تشغّل قنوات متعددة أو أتمتة متصفح أو أدوات وسائط.
    - **نظام التشغيل:** Ubuntu LTS أو Debian/Ubuntu حديث آخر.

    إذا كنت على Windows، فإن **WSL2 هو أسهل إعداد على نمط VM** ويتمتع بأفضل توافق
    مع الأدوات. راجع [Windows](/ar/platforms/windows)، [استضافة VPS](/ar/vps).
    وإذا كنت تشغّل macOS داخل VM، فراجع [macOS VM](/ar/install/macos-vm).

  </Accordion>
</AccordionGroup>

## ما هو OpenClaw؟

<AccordionGroup>
  <Accordion title="ما هو OpenClaw في فقرة واحدة؟">
    OpenClaw هو مساعد AI شخصي تشغّله على أجهزتك الخاصة. يرد على أسطح المراسلة التي تستخدمها بالفعل (WhatsApp وTelegram وSlack وMattermost وDiscord وGoogle Chat وSignal وiMessage وWebChat، إضافةً إلى channel plugins المضمّنة مثل QQ Bot) ويمكنه أيضًا القيام بالصوت + Canvas حي على المنصات المدعومة. **Gateway** هو مستوى التحكم الذي يعمل دائمًا؛ أما المساعد فهو المنتج.
  </Accordion>

  <Accordion title="القيمة المقترحة">
    OpenClaw ليس "مجرد غلاف حول Claude". بل هو **مستوى تحكم محلي أولًا** يتيح لك تشغيل
    مساعد قادر على **أجهزتك الخاصة**، ويمكن الوصول إليه من تطبيقات الدردشة التي تستخدمها بالفعل، مع
    جلسات ذات حالة، وذاكرة، وأدوات - من دون تسليم التحكم في سير العمل لديك إلى
    SaaS مستضاف.

    أبرز المزايا:

    - **أجهزتك، بياناتك:** شغّل Gateway حيثما تريد (Mac أو Linux أو VPS) واحتفظ
      بمساحة العمل + محفوظات الجلسات محليًا.
    - **قنوات حقيقية، وليس sandbox ويب:** WhatsApp/Telegram/Slack/Discord/Signal/iMessage/إلخ،
      إضافةً إلى الصوت المحمول وCanvas على المنصات المدعومة.
    - **محايد تجاه النماذج:** استخدم Anthropic وOpenAI وMiniMax وOpenRouter وغيرها، مع توجيه
      احتياطي وفصل حسب الوكيل.
    - **خيار محلي فقط:** شغّل نماذج محلية بحيث **يمكن أن تبقى كل البيانات على جهازك** إذا أردت.
    - **توجيه متعدد الوكلاء:** افصل الوكلاء حسب القناة أو الحساب أو المهمة، ولكل واحد
      مساحة عمله وافتراضياته الخاصة.
    - **مفتوح المصدر وقابل للاختراق:** افحصه ووسّعه واستضفه ذاتيًا من دون ارتهان لمزوّد.

    المستندات: [Gateway](/ar/gateway)، [القنوات](/ar/channels)، [تعدد الوكلاء](/ar/concepts/multi-agent)،
    [الذاكرة](/ar/concepts/memory).

  </Accordion>

  <Accordion title="لقد أعددته للتو - ماذا ينبغي أن أفعل أولًا؟">
    مشاريع أولى جيدة:

    - أنشئ موقعًا إلكترونيًا (WordPress أو Shopify أو موقعًا ثابتًا بسيطًا).
    - أنشئ نموذجًا أوليًا لتطبيق جوال (مخطط، شاشات، خطة API).
    - نظّم الملفات والمجلدات (تنظيف، تسمية، وسم).
    - اربط Gmail وأتمت الملخصات أو المتابعات.

    يمكنه التعامل مع مهام كبيرة، لكنه يعمل بأفضل شكل عندما تقسّمها إلى مراحل
    وتستخدم sub agents للعمل المتوازي.

  </Accordion>

  <Accordion title="ما أفضل خمس حالات استخدام يومية لـ OpenClaw؟">
    عادةً ما تبدو المكاسب اليومية هكذا:

    - **إحاطات شخصية:** ملخصات للبريد الوارد والتقويم والأخبار التي تهمك.
    - **البحث والصياغة:** بحث سريع وملخصات ومسودات أولى لرسائل البريد أو المستندات.
    - **التذكيرات والمتابعات:** تنبيهات وقوائم تحقق مدفوعة عبر cron أو heartbeat.
    - **أتمتة المتصفح:** ملء النماذج وجمع البيانات وتكرار المهام على الويب.
    - **التنسيق بين الأجهزة:** أرسل مهمة من هاتفك، ودع Gateway ينفذها على خادم، ثم استلم النتيجة في الدردشة.

  </Accordion>

  <Accordion title="هل يمكن أن يساعد OpenClaw في توليد العملاء المحتملين والتواصل والإعلانات والمدونات لـ SaaS؟">
    نعم فيما يخص **البحث والتأهيل والصياغة**. يمكنه فحص المواقع، وبناء قوائم مختصرة،
    وتلخيص العملاء المحتملين، وكتابة مسودات للتواصل أو نصوص إعلانات.

    أما بالنسبة إلى **التواصل أو تشغيل الإعلانات**، فأبقِ إنسانًا في الحلقة. تجنّب الرسائل المزعجة، والتزم بالقوانين المحلية
    وسياسات المنصات، وراجع أي شيء قبل إرساله. النمط الأكثر أمانًا هو أن يدع OpenClaw يكتب المسودة ثم توافق أنت عليها.

    المستندات: [الأمان](/ar/gateway/security).

  </Accordion>

  <Accordion title="ما المزايا مقارنةً بـ Claude Code لتطوير الويب؟">
    OpenClaw هو **مساعد شخصي** وطبقة تنسيق، وليس بديلًا عن IDE. استخدم
    Claude Code أو Codex للحصول على أسرع حلقة برمجة مباشرة داخل المستودع. واستخدم OpenClaw عندما
    تريد ذاكرة دائمة، ووصولًا عبر الأجهزة، وتنسيق الأدوات.

    المزايا:

    - **ذاكرة + مساحة عمل دائمة** عبر الجلسات
    - **وصول متعدد المنصات** (WhatsApp وTelegram وTUI وWebChat)
    - **تنسيق الأدوات** (المتصفح، الملفات، الجدولة، hooks)
    - **Gateway تعمل دائمًا** (شغّلها على VPS، وتفاعل معها من أي مكان)
    - **Nodes** من أجل المتصفح/الشاشة/الكاميرا/exec المحلي

    عرض توضيحي: [https://openclaw.ai/showcase](https://openclaw.ai/showcase)

  </Accordion>
</AccordionGroup>

## Skills والأتمتة

<AccordionGroup>
  <Accordion title="كيف أخصص Skills من دون إبقاء المستودع متسخًا؟">
    استخدم التجاوزات المُدارة بدلًا من تعديل نسخة المستودع. ضع تغييراتك في `~/.openclaw/skills/<name>/SKILL.md` (أو أضف مجلدًا عبر `skills.load.extraDirs` في `~/.openclaw/openclaw.json`). الأولوية هي `<workspace>/skills` → `<workspace>/.agents/skills` → `~/.agents/skills` → `~/.openclaw/skills` → المضمّن → `skills.load.extraDirs`، لذلك تظل التجاوزات المُدارة متقدمة على Skills المضمّنة من دون لمس git. وإذا كنت تحتاج إلى تثبيت المهارة عالميًا لكن تريد ظهورها لبعض الوكلاء فقط، فاحتفظ بالنسخة المشتركة في `~/.openclaw/skills` وتحكم في الظهور عبر `agents.defaults.skills` و`agents.list[].skills`. أما التعديلات الجديرة بالإرسال upstream فقط فينبغي أن تعيش في المستودع وتُرسل كـ PRs.
  </Accordion>

  <Accordion title="هل يمكنني تحميل Skills من مجلد مخصص؟">
    نعم. أضف أدلة إضافية عبر `skills.load.extraDirs` في `~/.openclaw/openclaw.json` (أدنى أولوية). ترتيب الأولوية الافتراضي هو `<workspace>/skills` → `<workspace>/.agents/skills` → `~/.agents/skills` → `~/.openclaw/skills` → المضمّن → `skills.load.extraDirs`. يقوم `clawhub` بالتثبيت في `./skills` افتراضيًا، ويتعامل OpenClaw مع ذلك على أنه `<workspace>/skills` في الجلسة التالية. وإذا كان يجب أن تظهر المهارة لوكلاء معينين فقط، فاقرن ذلك بـ `agents.defaults.skills` أو `agents.list[].skills`.
  </Accordion>

  <Accordion title="كيف يمكنني استخدام نماذج مختلفة لمهام مختلفة؟">
    الأنماط المدعومة اليوم هي:

    - **وظائف Cron**: يمكن للوظائف المعزولة تعيين تجاوز `model` لكل وظيفة.
    - **Sub-agents**: وجّه المهام إلى وكلاء منفصلين لهم نماذج افتراضية مختلفة.
    - **التبديل عند الطلب**: استخدم `/model` لتبديل نموذج الجلسة الحالية في أي وقت.

    راجع [وظائف Cron](/ar/automation/cron-jobs)، [التوجيه متعدد الوكلاء](/ar/concepts/multi-agent)، و[أوامر Slash](/ar/tools/slash-commands).

  </Accordion>

  <Accordion title="يتجمد bot أثناء تنفيذ عمل ثقيل. كيف أنقل ذلك بعيدًا؟">
    استخدم **sub-agents** للمهام الطويلة أو المتوازية. تعمل sub-agents في جلساتها الخاصة،
    وتعيد ملخصًا، وتحافظ على استجابة الدردشة الرئيسية.

    اطلب من bot أن "ينشئ sub-agent لهذه المهمة" أو استخدم `/subagents`.
    واستخدم `/status` في الدردشة لمعرفة ما الذي يقوم به Gateway الآن (وما إذا كان مشغولًا).

    نصيحة حول الرموز: المهام الطويلة وsub-agents كلاهما يستهلكان الرموز. إذا كانت التكلفة مقلقة، فاضبط
    نموذجًا أرخص لـ sub-agents عبر `agents.defaults.subagents.model`.

    المستندات: [Sub-agents](/ar/tools/subagents)، [المهام الخلفية](/ar/automation/tasks).

  </Accordion>

  <Accordion title="كيف تعمل جلسات subagent المرتبطة بالخيوط على Discord؟">
    استخدم روابط الخيوط. يمكنك ربط خيط Discord بهدف subagent أو session بحيث تبقى الرسائل اللاحقة في ذلك الخيط ضمن الجلسة المرتبطة.

    التدفق الأساسي:

    - أنشئ عبر `sessions_spawn` باستخدام `thread: true` (واختياريًا `mode: "session"` للمتابعة الدائمة).
    - أو اربط يدويًا عبر `/focus <target>`.
    - استخدم `/agents` لفحص حالة الربط.
    - استخدم `/session idle <duration|off>` و`/session max-age <duration|off>` للتحكم في إلغاء التركيز التلقائي.
    - استخدم `/unfocus` لفصل الخيط.

    الإعدادات المطلوبة:

    - الافتراضيات العامة: `session.threadBindings.enabled`, `session.threadBindings.idleHours`, `session.threadBindings.maxAgeHours`.
    - تجاوزات Discord: `channels.discord.threadBindings.enabled`, `channels.discord.threadBindings.idleHours`, `channels.discord.threadBindings.maxAgeHours`.
    - الربط التلقائي عند الإنشاء: اضبط `channels.discord.threadBindings.spawnSubagentSessions: true`.

    المستندات: [Sub-agents](/ar/tools/subagents)، [Discord](/ar/channels/discord)، [مرجع التكوين](/ar/gateway/configuration-reference)، [أوامر Slash](/ar/tools/slash-commands).

  </Accordion>

  <Accordion title="انتهى subagent، لكن تحديث الإكمال ذهب إلى المكان الخطأ أو لم يُنشر أبدًا. ما الذي يجب أن أتحقق منه؟">
    تحقق أولًا من مسار الطالب الذي تم حله:

    - يفضّل تسليم subagent في وضع الإكمال أي خيط أو مسار محادثة مرتبط عند وجوده.
    - إذا كان أصل الإكمال يحمل قناة فقط، يعود OpenClaw إلى المسار المخزن لجلسة الطالب (`lastChannel` / `lastTo` / `lastAccountId`) بحيث يظل التسليم المباشر ممكنًا.
    - إذا لم يوجد مسار مرتبط ولا مسار مخزن صالح، فقد يفشل التسليم المباشر وتعود النتيجة إلى تسليم الجلسة عبر الطابور بدلًا من النشر الفوري في الدردشة.
    - قد تؤدي الأهداف غير الصالحة أو القديمة أيضًا إلى السقوط إلى الطابور أو فشل التسليم النهائي.
    - إذا كان آخر رد مساعد مرئي للطفل هو الرمز الصامت المطابق تمامًا `NO_REPLY` / `no_reply`، أو المطابق تمامًا `ANNOUNCE_SKIP`، فإن OpenClaw يتعمد كتم الإعلان بدلًا من نشر تقدم أقدم لم يعد صالحًا.
    - إذا انتهت مهلة الطفل بعد استدعاءات أدوات فقط، فقد يختصر الإعلان ذلك إلى ملخص قصير للتقدم الجزئي بدلًا من إعادة عرض مخرجات الأدوات الخام.

    التصحيح:

    ```bash
    openclaw tasks show <runId-or-sessionKey>
    ```

    المستندات: [Sub-agents](/ar/tools/subagents)، [المهام الخلفية](/ar/automation/tasks)، [أدوات الجلسة](/ar/concepts/session-tool).

  </Accordion>

  <Accordion title="لا تعمل Cron أو التذكيرات. ما الذي يجب أن أتحقق منه؟">
    تعمل Cron داخل عملية Gateway. إذا لم يكن Gateway يعمل بشكل مستمر،
    فلن تعمل الوظائف المجدولة.

    قائمة تحقق:

    - أكد أن cron مفعّلة (`cron.enabled`) وأن `OPENCLAW_SKIP_CRON` غير مضبوط.
    - تحقق من أن Gateway يعمل 24/7 (من دون سكون/إعادة تشغيل).
    - تحقق من إعدادات المنطقة الزمنية للوظيفة (`--tz` مقابل المنطقة الزمنية للمضيف).

    التصحيح:

    ```bash
    openclaw cron run <jobId>
    openclaw cron runs --id <jobId> --limit 50
    ```

    المستندات: [وظائف Cron](/ar/automation/cron-jobs)، [الأتمتة والمهام](/ar/automation).

  </Accordion>

  <Accordion title="تم تشغيل Cron، لكن لم يُرسَل شيء إلى القناة. لماذا؟">
    تحقق من وضع التسليم أولًا:

    - `--no-deliver` / `delivery.mode: "none"` يعني أنه لا يُتوقع إرسال أي رسالة خارجية.
    - الهدف المفقود أو غير الصالح للإعلان (`channel` / `to`) يعني أن المشغّل تخطى التسليم الصادر.
    - فشل مصادقة القناة (`unauthorized`, `Forbidden`) يعني أن المشغّل حاول التسليم لكن بيانات الاعتماد منعته.
    - النتيجة المعزولة الصامتة (`NO_REPLY` / `no_reply` فقط) تُعامل على أنها غير قابلة للتسليم عمدًا، لذا يكتم المشغّل أيضًا تسليم الطابور الاحتياطي.

    بالنسبة إلى وظائف cron المعزولة، يملك المشغّل التسليم النهائي. من المتوقع
    أن يعيد الوكيل ملخصًا نصيًا عاديًا كي يرسله المشغّل. يحافظ `--no-deliver`
    على بقاء النتيجة داخلية؛ ولا يسمح للوكيل بالإرسال مباشرةً باستخدام
    أداة الرسائل بدلًا من ذلك.

    التصحيح:

    ```bash
    openclaw cron runs --id <jobId> --limit 50
    openclaw tasks show <runId-or-sessionKey>
    ```

    المستندات: [وظائف Cron](/ar/automation/cron-jobs)، [المهام الخلفية](/ar/automation/tasks).

  </Accordion>

  <Accordion title="لماذا بدّل تشغيل cron معزول النماذج أو أعاد المحاولة مرة واحدة؟">
    غالبًا ما يكون هذا هو مسار التبديل الحي للنموذج، وليس جدولة مكررة.

    يمكن لـ cron المعزول أن يحفظ تحويل نموذج وقت التشغيل ويعيد المحاولة عندما
    يرمي التشغيل النشط `LiveSessionModelSwitchError`. وتحافظ إعادة المحاولة على
    المزوّد/النموذج بعد التبديل، وإذا حمل التبديل تجاوزًا جديدًا لملف تعريف المصادقة،
    فإن cron يحفظه أيضًا قبل إعادة المحاولة.

    قواعد الاختيار ذات الصلة:

    - يتقدم تجاوز نموذج hook الخاص بـ Gmail أولًا عندما يكون منطبقًا.
    - ثم `model` لكل وظيفة.
    - ثم أي تجاوز نموذج مخزن لجلسة cron.
    - ثم الاختيار العادي لنموذج الوكيل/النموذج الافتراضي.

    حلقة إعادة المحاولة محدودة. بعد المحاولة الأولية إضافةً إلى محاولتي تبديل،
    تتوقف cron بدلًا من الاستمرار إلى ما لا نهاية.

    التصحيح:

    ```bash
    openclaw cron runs --id <jobId> --limit 50
    openclaw tasks show <runId-or-sessionKey>
    ```

    المستندات: [وظائف Cron](/ar/automation/cron-jobs)، [CLI الخاص بـ cron](/cli/cron).

  </Accordion>

  <Accordion title="كيف أثبت Skills على Linux؟">
    استخدم أوامر `openclaw skills` الأصلية أو أسقط Skills داخل مساحة العمل لديك. واجهة Skills في macOS غير متاحة على Linux.
    تصفح Skills على [https://clawhub.ai](https://clawhub.ai).

    ```bash
    openclaw skills search "calendar"
    openclaw skills search --limit 20
    openclaw skills install <skill-slug>
    openclaw skills install <skill-slug> --version <version>
    openclaw skills install <skill-slug> --force
    openclaw skills update --all
    openclaw skills list --eligible
    openclaw skills check
    ```

    يكتب `openclaw skills install` الأصلي داخل دليل `skills/`
    في مساحة العمل النشطة. ثبّت CLI المنفصل `clawhub` فقط إذا كنت تريد نشر أو
    مزامنة Skills الخاصة بك. وبالنسبة إلى التثبيتات المشتركة عبر الوكلاء، ضع Skill ضمن
    `~/.openclaw/skills` واستخدم `agents.defaults.skills` أو
    `agents.list[].skills` إذا أردت تضييق الوكلاء الذين يمكنهم رؤيتها.

  </Accordion>

  <Accordion title="هل يمكن لـ OpenClaw تشغيل مهام وفق جدول زمني أو بشكل مستمر في الخلفية؟">
    نعم. استخدم مجدول Gateway:

    - **وظائف Cron** للمهام المجدولة أو المتكررة (تستمر عبر إعادة التشغيل).
    - **Heartbeat** لفحوصات دورية لـ "الجلسة الرئيسية".
    - **وظائف معزولة** لوكلاء مستقلين ينشرون ملخصات أو يسلّمون إلى الدردشات.

    المستندات: [وظائف Cron](/ar/automation/cron-jobs)، [الأتمتة والمهام](/ar/automation)،
    [Heartbeat](/ar/gateway/heartbeat).

  </Accordion>

  <Accordion title="هل يمكنني تشغيل Skills الخاصة بـ Apple macOS فقط من Linux؟">
    ليس مباشرةً. يتم تقييد Skills الخاصة بـ macOS عبر `metadata.openclaw.os` إضافةً إلى الثنائيات المطلوبة، ولا تظهر Skills في system prompt إلا عندما تكون مؤهلة على **مضيف Gateway**. وعلى Linux، لن يتم تحميل Skills الخاصة بـ `darwin` فقط (مثل `apple-notes` و`apple-reminders` و`things-mac`) ما لم تتجاوز هذا التقييد.

    لديك ثلاثة أنماط مدعومة:

    **الخيار A - شغّل Gateway على جهاز Mac (الأبسط).**
    شغّل Gateway حيث توجد ثنائيات macOS، ثم اتصل من Linux في [الوضع البعيد](#gateway-ports-already-running-and-remote-mode) أو عبر Tailscale. تُحمَّل Skills بشكل طبيعي لأن مضيف Gateway هو macOS.

    **الخيار B - استخدم node macOS (من دون SSH).**
    شغّل Gateway على Linux، واقرن node لـ macOS (تطبيق شريط القوائم)، واضبط **Node Run Commands** على "Always Ask" أو "Always Allow" على جهاز Mac. يمكن لـ OpenClaw أن يعامل Skills الخاصة بـ macOS فقط على أنها مؤهلة عندما تكون الثنائيات المطلوبة موجودة على node. يشغّل الوكيل هذه Skills عبر أداة `nodes`. وإذا اخترت "Always Ask"، فإن الموافقة على "Always Allow" في المطالبة تضيف ذلك الأمر إلى allowlist.

    **الخيار C - وكّل ثنائيات macOS عبر SSH (متقدم).**
    أبقِ Gateway على Linux، لكن اجعل ثنائيات CLI المطلوبة تُحل إلى أغلفة SSH تُشغّل على جهاز Mac. ثم تجاوز Skill للسماح بـ Linux بحيث تظل مؤهلة.

    1. أنشئ غلاف SSH للثنائي (مثال: `memo` لـ Apple Notes):

       ```bash
       #!/usr/bin/env bash
       set -euo pipefail
       exec ssh -T user@mac-host /opt/homebrew/bin/memo "$@"
       ```

    2. ضع الغلاف على `PATH` على مضيف Linux (مثل `~/bin/memo`).
    3. تجاوز metadata الخاصة بـ Skill (في workspace أو `~/.openclaw/skills`) للسماح بـ Linux:

       ```markdown
       ---
       name: apple-notes
       description: Manage Apple Notes via the memo CLI on macOS.
       metadata: { "openclaw": { "os": ["darwin", "linux"], "requires": { "bins": ["memo"] } } }
       ---
       ```

    4. ابدأ جلسة جديدة حتى يتم تحديث لقطة Skills.

  </Accordion>

  <Accordion title="هل لديكم تكامل Notion أو HeyGen؟">
    ليس مدمجًا حاليًا.

    الخيارات:

    - **Skill / plugin مخصص:** الأفضل للوصول الموثوق إلى API (لكل من Notion وHeyGen APIs).
    - **أتمتة المتصفح:** تعمل من دون شيفرة لكنها أبطأ وأكثر هشاشة.

    إذا كنت تريد الحفاظ على السياق لكل عميل (سير عمل الوكالات)، فالنمط البسيط هو:

    - صفحة Notion واحدة لكل عميل (السياق + التفضيلات + العمل النشط).
    - اطلب من الوكيل جلب تلك الصفحة في بداية الجلسة.

    إذا كنت تريد تكاملًا أصليًا، فافتح طلب ميزة أو ابنِ Skill
    تستهدف تلك APIs.

    ثبّت Skills:

    ```bash
    openclaw skills install <skill-slug>
    openclaw skills update --all
    ```

    تصل التثبيتات الأصلية إلى دليل `skills/` في مساحة العمل النشطة. وبالنسبة إلى Skills المشتركة بين الوكلاء، ضعها في `~/.openclaw/skills/<name>/SKILL.md`. وإذا كان يجب أن يراها بعض الوكلاء فقط في التثبيت المشترك، فاضبط `agents.defaults.skills` أو `agents.list[].skills`. تتوقع بعض Skills وجود ثنائيات مثبّتة عبر Homebrew؛ وعلى Linux فهذا يعني Linuxbrew (راجع إدخال الأسئلة الشائعة عن Homebrew على Linux أعلاه). راجع [Skills](/ar/tools/skills)، [إعدادات Skills](/ar/tools/skills-config)، و[ClawHub](/ar/tools/clawhub).

  </Accordion>

  <Accordion title="كيف أستخدم Chrome المسجّل الدخول لديّ بالفعل مع OpenClaw؟">
    استخدم ملف تعريف المتصفح المضمّن `user`، الذي يتصل عبر Chrome DevTools MCP:

    ```bash
    openclaw browser --browser-profile user tabs
    openclaw browser --browser-profile user snapshot
    ```

    وإذا أردت اسمًا مخصصًا، فأنشئ ملف تعريف MCP صريحًا:

    ```bash
    openclaw browser create-profile --name chrome-live --driver existing-session
    openclaw browser --browser-profile chrome-live tabs
    ```

    هذا المسار محلي للمضيف. إذا كان Gateway يعمل في مكان آخر، فإما أن تشغّل مضيف node على جهاز المتصفح أو تستخدم CDP بعيدًا بدلًا من ذلك.

    الحدود الحالية لـ `existing-session` / `user`:

    - الإجراءات تعتمد على المرجع ref، لا على محددات CSS
    - تتطلب عمليات الرفع `ref` / `inputRef` وتدعم حاليًا ملفًا واحدًا في كل مرة
    - لا تزال `responsebody` وتصدير PDF واعتراض التنزيلات والإجراءات الدُفعية تحتاج إلى متصفح مُدار أو ملف تعريف CDP خام

  </Accordion>
</AccordionGroup>

## العزل والذاكرة

<AccordionGroup>
  <Accordion title="هل توجد مستندات مخصصة للعزل؟">
    نعم. راجع [العزل](/ar/gateway/sandboxing). وبالنسبة إلى إعدادات Docker تحديدًا (gateway كاملة داخل Docker أو صور sandbox)، راجع [Docker](/ar/install/docker).
  </Accordion>

  <Accordion title="يبدو Docker محدودًا - كيف أفعّل الميزات الكاملة؟">
    الصورة الافتراضية تعتمد الأمان أولًا وتعمل كمستخدم `node`، لذلك فهي لا
    تتضمن حزم النظام أو Homebrew أو المتصفحات المضمّنة. وللحصول على إعداد أكمل:

    - اجعل `/home/node` دائمًا باستخدام `OPENCLAW_HOME_VOLUME` بحيث تبقى الذاكرات المؤقتة.
    - اخبز تبعيات النظام داخل الصورة باستخدام `OPENCLAW_DOCKER_APT_PACKAGES`.
    - ثبّت متصفحات Playwright عبر CLI المضمّن:
      `node /app/node_modules/playwright-core/cli.js install chromium`
    - اضبط `PLAYWRIGHT_BROWSERS_PATH` وتأكد من بقاء المسار دائمًا.

    المستندات: [Docker](/ar/install/docker)، [المتصفح](/ar/tools/browser).

  </Accordion>

  <Accordion title="هل يمكنني إبقاء الرسائل الخاصة شخصية وجعل المجموعات عامة/معزولة باستخدام وكيل واحد؟">
    نعم - إذا كانت الحركة الخاصة لديك هي **رسائل خاصة** وكانت الحركة العامة **مجموعات**.

    استخدم `agents.defaults.sandbox.mode: "non-main"` بحيث تعمل جلسات المجموعات/القنوات (مفاتيح غير رئيسية) داخل Docker، بينما تبقى جلسة DM الرئيسية على المضيف. ثم قيّد الأدوات المتاحة في الجلسات المعزولة عبر `tools.sandbox.tools`.

    شرح الإعداد + مثال للتكوين: [المجموعات: رسائل خاصة شخصية + مجموعات عامة](/ar/channels/groups#pattern-personal-dms-public-groups-single-agent)

    مرجع الإعدادات الأساسي: [تكوين Gateway](/ar/gateway/configuration-reference#agentsdefaultssandbox)

  </Accordion>

  <Accordion title="كيف أربط مجلدًا من المضيف داخل sandbox؟">
    اضبط `agents.defaults.sandbox.docker.binds` على `["host:path:mode"]` (مثل `"/home/user/src:/src:ro"`). تندمج عمليات الربط العامة وعمليات الربط لكل وكيل؛ ويتم تجاهل عمليات الربط لكل وكيل عندما تكون `scope: "shared"`. استخدم `:ro` لأي شيء حساس، وتذكّر أن عمليات الربط تتجاوز جدران نظام الملفات الخاصة بـ sandbox.

    يتحقق OpenClaw من مصادر الربط مقابل كل من المسار المطبع والمسار القانوني الذي يُحل عبر أعمق أصل موجود. وهذا يعني أن محاولات الهروب عبر الوالدين القائمين على symlink ما تزال تُغلق بشكل آمن حتى عندما لا يكون المقطع الأخير من المسار موجودًا بعد، كما تستمر فحوصات allowed-root في التطبيق بعد حل symlink.

    راجع [العزل](/ar/gateway/sandboxing#custom-bind-mounts) و[Sandbox vs Tool Policy vs Elevated](/ar/gateway/sandbox-vs-tool-policy-vs-elevated#bind-mounts-security-quick-check) لأمثلة وملاحظات الأمان.

  </Accordion>

  <Accordion title="كيف تعمل الذاكرة؟">
    ذاكرة OpenClaw ليست سوى ملفات Markdown في مساحة عمل الوكيل:

    - ملاحظات يومية في `memory/YYYY-MM-DD.md`
    - ملاحظات طويلة الأمد منسّقة في `MEMORY.md` (للجلسات الرئيسية/الخاصة فقط)

    يشغّل OpenClaw أيضًا **تفريغ ذاكرة صامت قبل الضغط** لتذكير النموذج
    بكتابة ملاحظات دائمة قبل الضغط التلقائي. لا يعمل هذا إلا عندما تكون مساحة العمل
    قابلة للكتابة (تتخطاه sandboxes للقراءة فقط). راجع [الذاكرة](/ar/concepts/memory).

  </Accordion>

  <Accordion title="الذاكرة تستمر في نسيان الأشياء. كيف أجعلها تترسخ؟">
    اطلب من bot أن **يكتب الحقيقة إلى الذاكرة**. الملاحظات طويلة الأمد تنتمي إلى `MEMORY.md`,
    أما السياق قصير الأمد فيوضع في `memory/YYYY-MM-DD.md`.

    هذا ما يزال مجالًا نعمل على تحسينه. ومن المفيد تذكير النموذج بتخزين الذكريات؛
    فهو سيعرف ما الذي يجب فعله. وإذا استمر في النسيان، فتحقق من أن Gateway يستخدم
    مساحة العمل نفسها في كل تشغيل.

    المستندات: [الذاكرة](/ar/concepts/memory)، [مساحة عمل الوكيل](/ar/concepts/agent-workspace).

  </Accordion>

  <Accordion title="هل تستمر الذاكرة إلى الأبد؟ ما الحدود؟">
    تعيش ملفات الذاكرة على القرص وتستمر حتى تحذفها. الحد هنا هو
    التخزين، وليس النموذج. أما **سياق الجلسة** فلا يزال مقيدًا بنافذة
    سياق النموذج، لذا قد تُضغط المحادثات الطويلة أو تُقتطع. ولهذا
    يوجد بحث الذاكرة - إذ يعيد فقط الأجزاء ذات الصلة إلى السياق.

    المستندات: [الذاكرة](/ar/concepts/memory)، [السياق](/ar/concepts/context).

  </Accordion>

  <Accordion title="هل يتطلب البحث الدلالي في الذاكرة مفتاح OpenAI API؟">
    فقط إذا كنت تستخدم **OpenAI embeddings**. يغطي Codex OAuth الدردشة/الإكمالات
    ولا **يمنح** الوصول إلى embeddings، لذا فإن **تسجيل الدخول عبر Codex (OAuth أو
    تسجيل الدخول عبر Codex CLI)** لا يفيد في البحث الدلالي في الذاكرة. لا تزال OpenAI embeddings
    تحتاج إلى مفتاح API حقيقي (`OPENAI_API_KEY` أو `models.providers.openai.apiKey`).

    إذا لم تضبط مزودًا صراحةً، يختار OpenClaw مزودًا تلقائيًا عندما
    يستطيع حل مفتاح API (ملفات تعريف المصادقة، أو `models.providers.*.apiKey`، أو متغيرات env).
    ويفضّل OpenAI إذا توفّر مفتاح OpenAI، وإلا Gemini إذا توفر مفتاح Gemini،
    ثم Voyage، ثم Mistral. وإذا لم يتوفر أي مفتاح بعيد، فإن بحث
    الذاكرة يبقى معطّلًا حتى تضبطه. وإذا كان لديك مسار نموذج محلي
    مضبوط وموجود، فإن OpenClaw
    يفضّل `local`. كما أن Ollama مدعوم عندما تضبط صراحةً
    `memorySearch.provider = "ollama"`.

    إذا كنت تفضل البقاء محليًا، فاضبط `memorySearch.provider = "local"` (واختياريًا
    `memorySearch.fallback = "none"`). وإذا كنت تريد Gemini embeddings، فاضبط
    `memorySearch.provider = "gemini"` وقدّم `GEMINI_API_KEY` (أو
    `memorySearch.remote.apiKey`). نحن ندعم نماذج embedding من **OpenAI وGemini وVoyage وMistral وOllama أو local**
    - راجع [الذاكرة](/ar/concepts/memory) لمعرفة تفاصيل الإعداد.

  </Accordion>
</AccordionGroup>

## مكان وجود الأشياء على القرص

<AccordionGroup>
  <Accordion title="هل تُحفَظ كل البيانات المستخدمة مع OpenClaw محليًا؟">
    لا - **حالة OpenClaw محلية**، لكن **الخدمات الخارجية لا تزال ترى ما ترسله إليها**.

    - **محلي افتراضيًا:** الجلسات وملفات الذاكرة والإعدادات ومساحة العمل تعيش على مضيف Gateway
      (`~/.openclaw` + دليل مساحة العمل لديك).
    - **بعيد بالضرورة:** الرسائل التي ترسلها إلى مزودي النماذج (Anthropic/OpenAI/إلخ) تذهب إلى
      APIs الخاصة بهم، كما تخزن منصات الدردشة (WhatsApp/Telegram/Slack/إلخ) بيانات الرسائل على
      خوادمها.
    - **أنت تتحكم في الأثر:** استخدام النماذج المحلية يبقي المطالبات على جهازك، لكن
      حركة القناة لا تزال تمر عبر خوادم تلك القناة.

    ذو صلة: [مساحة عمل الوكيل](/ar/concepts/agent-workspace)، [الذاكرة](/ar/concepts/memory).

  </Accordion>

  <Accordion title="أين يخزن OpenClaw بياناته؟">
    كل شيء يعيش تحت `$OPENCLAW_STATE_DIR` (الافتراضي: `~/.openclaw`):

    | المسار                                                            | الغرض                                                            |
    | --------------------------------------------------------------- | ------------------------------------------------------------------ |
    | `$OPENCLAW_STATE_DIR/openclaw.json`                             | الإعدادات الرئيسية (JSON5)                                                |
    | `$OPENCLAW_STATE_DIR/credentials/oauth.json`                    | استيراد OAuth قديم (يُنسخ إلى ملفات تعريف المصادقة عند أول استخدام)       |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/auth-profiles.json` | ملفات تعريف المصادقة (OAuth، مفاتيح API، و`keyRef`/`tokenRef` الاختيارية)  |
    | `$OPENCLAW_STATE_DIR/secrets.json`                              | حمولة سرية اختيارية معتمدة على الملفات لمزودي `file` ضمن SecretRef |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/auth.json`          | ملف توافق قديم (تُنظف منه إدخالات `api_key` الثابتة)      |
    | `$OPENCLAW_STATE_DIR/credentials/`                              | حالة المزوّد (مثل `whatsapp/<accountId>/creds.json`)            |
    | `$OPENCLAW_STATE_DIR/agents/`                                   | حالة لكل وكيل (agentDir + sessions)                              |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/sessions/`                | محفوظات المحادثة والحالة (لكل وكيل)                           |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/sessions/sessions.json`   | بيانات الجلسة الوصفية (لكل وكيل)                                       |

    المسار القديم للوكيل الواحد: `~/.openclaw/agent/*` (يرحّله `openclaw doctor`).

    أما **مساحة العمل** الخاصة بك (AGENTS.md، ملفات الذاكرة، Skills، إلخ) فهي منفصلة وتُضبط عبر `agents.defaults.workspace` (الافتراضي: `~/.openclaw/workspace`).

  </Accordion>

  <Accordion title="أين يجب أن تعيش AGENTS.md / SOUL.md / USER.md / MEMORY.md؟">
    تعيش هذه الملفات في **مساحة عمل الوكيل**، وليس في `~/.openclaw`.

    - **مساحة العمل (لكل وكيل)**: `AGENTS.md`, `SOUL.md`, `IDENTITY.md`, `USER.md`,
      `MEMORY.md` (أو fallback قديم `memory.md` عندما يكون `MEMORY.md` غائبًا)،
      `memory/YYYY-MM-DD.md`, و`HEARTBEAT.md` اختياري.
    - **دليل الحالة (`~/.openclaw`)**: الإعدادات، حالة القناة/المزوّد، ملفات تعريف المصادقة، الجلسات، السجلات،
      وSkills المشتركة (`~/.openclaw/skills`).

    مساحة العمل الافتراضية هي `~/.openclaw/workspace`، ويمكن ضبطها عبر:

    ```json5
    {
      agents: { defaults: { workspace: "~/.openclaw/workspace" } },
    }
    ```

    إذا كان bot "ينسى" بعد إعادة التشغيل، فتأكد من أن Gateway يستخدم مساحة العمل نفسها
    في كل إطلاق (وتذكّر: الوضع البعيد يستخدم مساحة عمل **مضيف gateway**
    وليس حاسوبك المحمول المحلي).

    نصيحة: إذا كنت تريد سلوكًا دائمًا أو تفضيلًا، فاطلب من bot أن **يكتبه في
    AGENTS.md أو MEMORY.md** بدلًا من الاعتماد على محفوظات الدردشة.

    راجع [مساحة عمل الوكيل](/ar/concepts/agent-workspace) و[الذاكرة](/ar/concepts/memory).

  </Accordion>

  <Accordion title="استراتيجية النسخ الاحتياطي الموصى بها">
    ضع **مساحة عمل الوكيل** في مستودع git **خاص** واحتفظ بنسخة احتياطية منه في مكان
    خاص (مثل GitHub الخاص). يلتقط هذا الذاكرة + ملفات AGENTS/SOUL/USER،
    ويتيح لك استعادة "عقل" المساعد لاحقًا.

    **لا** تقم بعمل commit لأي شيء ضمن `~/.openclaw` (بيانات الاعتماد أو الجلسات أو الرموز أو حمولات الأسرار المشفرة).
    وإذا كنت تحتاج إلى استعادة كاملة، فانسخ كلًا من مساحة العمل ودليل الحالة
    بشكل منفصل (راجع سؤال الترحيل أعلاه).

    المستندات: [مساحة عمل الوكيل](/ar/concepts/agent-workspace).

  </Accordion>

  <Accordion title="كيف أزيل OpenClaw بالكامل؟">
    راجع الدليل المخصص: [إزالة التثبيت](/ar/install/uninstall).
  </Accordion>

  <Accordion title="هل يمكن للوكلاء العمل خارج مساحة العمل؟">
    نعم. مساحة العمل هي **cwd الافتراضي** ومرساة الذاكرة، وليست sandbox صارمًا.
    تُحل المسارات النسبية داخل مساحة العمل، لكن المسارات المطلقة يمكنها الوصول إلى مواقع
    أخرى على المضيف ما لم يكن العزل مفعّلًا. وإذا كنت تحتاج إلى عزل، فاستخدم
    [`agents.defaults.sandbox`](/ar/gateway/sandboxing) أو إعدادات sandbox لكل وكيل. وإذا كنت
    تريد أن يكون مستودع ما دليل العمل الافتراضي، فاجعل
    `workspace` لذلك الوكيل يشير إلى جذر المستودع. مستودع OpenClaw هو مجرد
    شيفرة مصدر؛ أبقِ مساحة العمل منفصلة ما لم تكن تريد عمدًا أن يعمل الوكيل داخله.

    مثال (المستودع كـ cwd افتراضي):

    ```json5
    {
      agents: {
        defaults: {
          workspace: "~/Projects/my-repo",
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="الوضع البعيد: أين يوجد مخزن الجلسات؟">
    تملك **مضيف gateway** حالة الجلسة. إذا كنت في الوضع البعيد، فإن مخزن الجلسات الذي يهمك موجود على الجهاز البعيد، لا على حاسوبك المحمول المحلي. راجع [إدارة الجلسات](/ar/concepts/session).
  </Accordion>
</AccordionGroup>

## أساسيات التكوين

<AccordionGroup>
  <Accordion title="ما تنسيق الإعدادات؟ وأين هي؟">
    يقرأ OpenClaw ملف إعدادات **JSON5** اختياريًا من `$OPENCLAW_CONFIG_PATH` (الافتراضي: `~/.openclaw/openclaw.json`):

    ```
    $OPENCLAW_CONFIG_PATH
    ```

    إذا كان الملف مفقودًا، فإنه يستخدم افتراضيات آمنة نسبيًا (بما في ذلك مساحة عمل افتراضية هي `~/.openclaw/workspace`).

  </Accordion>

  <Accordion title='لقد ضبطت gateway.bind: "lan" (أو "tailnet") والآن لا يوجد شيء يستمع / تقول UI unauthorized'>
    تتطلب عمليات الربط غير loopback **مسار مصادقة صالحًا للـ gateway**. وعمليًا يعني هذا:

    - مصادقة shared-secret: token أو password
    - `gateway.auth.mode: "trusted-proxy"` خلف وكيل عكسي غير loopback ومدرك للهوية ومُضبط بصورة صحيحة

    ```json5
    {
      gateway: {
        bind: "lan",
        auth: {
          mode: "token",
          token: "replace-me",
        },
      },
    }
    ```

    ملاحظات:

    - `gateway.remote.token` / `.password` **لا** يفعّلان مصادقة gateway المحلية بمفردهما.
    - يمكن للمسارات المحلية استخدام `gateway.remote.*` كـ fallback فقط عندما تكون `gateway.auth.*` غير مضبوطة.
    - بالنسبة إلى مصادقة password، اضبط `gateway.auth.mode: "password"` مع `gateway.auth.password` (أو `OPENCLAW_GATEWAY_PASSWORD`) بدلًا من ذلك.
    - إذا كانت `gateway.auth.token` / `gateway.auth.password` مضبوطة صراحةً عبر SecretRef وغير محلولة، فإن الحل يفشل بشكل مغلق (من دون fallback بعيد يخفي المشكلة).
    - تتم مصادقة إعدادات Control UI المعتمدة على shared-secret عبر `connect.params.auth.token` أو `connect.params.auth.password` (المخزنين في إعدادات التطبيق/UI). أما الأوضاع المعتمدة على الهوية مثل Tailscale Serve أو `trusted-proxy` فتستخدم رؤوس الطلبات بدلًا من ذلك. تجنب وضع shared secrets في عناوين URL.
    - مع `gateway.auth.mode: "trusted-proxy"`، فإن الوكلاء العكسيين على loopback ضمن المضيف نفسه **لا** يفون بمصادقة trusted-proxy. يجب أن يكون trusted proxy مصدرًا غير loopback ومُضبطًا.

  </Accordion>

  <Accordion title="لماذا أحتاج إلى token على localhost الآن؟">
    يفرض OpenClaw مصادقة gateway افتراضيًا، بما في ذلك loopback. في المسار الافتراضي العادي يعني هذا مصادقة token: إذا لم يُضبط مسار مصادقة صريح، فإن بدء تشغيل gateway يُحل إلى وضع token ويُنشئ واحدًا تلقائيًا، مع حفظه في `gateway.auth.token`، لذا **يجب على عملاء WS المحليين المصادقة**. هذا يمنع العمليات المحلية الأخرى من استدعاء Gateway.

    إذا كنت تفضل مسار مصادقة مختلفًا، فيمكنك اختيار وضع password صراحةً (أو، بالنسبة إلى الوكلاء العكسيين غير loopback والمدركين للهوية، `trusted-proxy`). وإذا **أردت حقًا** loopback مفتوحًا، فاضبط `gateway.auth.mode: "none"` صراحةً في إعداداتك. ويمكن لـ Doctor إنشاء token لك في أي وقت: `openclaw doctor --generate-gateway-token`.

  </Accordion>

  <Accordion title="هل يجب أن أعيد التشغيل بعد تغيير الإعدادات؟">
    يراقب Gateway الإعدادات ويدعم hot-reload:

    - `gateway.reload.mode: "hybrid"` (الافتراضي): يطبّق التغييرات الآمنة مباشرةً، ويعيد التشغيل للتغييرات الحرجة
    - كما أن `hot` و`restart` و`off` مدعومة أيضًا

  </Accordion>

  <Accordion title="كيف أعطّل العبارات الطريفة في CLI؟">
    اضبط `cli.banner.taglineMode` في الإعدادات:

    ```json5
    {
      cli: {
        banner: {
          taglineMode: "off", // random | default | off
        },
      },
    }
    ```

    - `off`: يُخفي نص الشعار لكنه يُبقي سطر عنوان البانر/الإصدار.
    - `default`: يستخدم `All your chats, one OpenClaw.` في كل مرة.
    - `random`: شعارات طريفة/موسمية متناوبة (السلوك الافتراضي).
    - إذا أردت عدم وجود بانر مطلقًا، فاضبط env `OPENCLAW_HIDE_BANNER=1`.

  </Accordion>

  <Accordion title="كيف أفعّل web search (وweb fetch)؟">
    يعمل `web_fetch` من دون مفتاح API. أما `web_search` فيعتمد على
    المزوّد الذي اخترته:

    - تتطلب المزوّدات المعتمدة على API مثل Brave وExa وFirecrawl وGemini وGrok وKimi وMiniMax Search وPerplexity وTavily إعداد مفتاح API المعتاد الخاص بها.
    - Ollama Web Search لا يحتاج إلى مفتاح، لكنه يستخدم مضيف Ollama الذي قمت بتكوينه ويتطلب `ollama signin`.
    - DuckDuckGo لا يحتاج إلى مفتاح، لكنه تكامل غير رسمي قائم على HTML.
    - SearXNG لا يحتاج إلى مفتاح/ويمكن استضافته ذاتيًا؛ اضبط `SEARXNG_BASE_URL` أو `plugins.entries.searxng.config.webSearch.baseUrl`.

    **موصى به:** شغّل `openclaw configure --section web` واختر مزودًا.
    بدائل البيئة:

    - Brave: `BRAVE_API_KEY`
    - Exa: `EXA_API_KEY`
    - Firecrawl: `FIRECRAWL_API_KEY`
    - Gemini: `GEMINI_API_KEY`
    - Grok: `XAI_API_KEY`
    - Kimi: `KIMI_API_KEY` أو `MOONSHOT_API_KEY`
    - MiniMax Search: `MINIMAX_CODE_PLAN_KEY` أو `MINIMAX_CODING_API_KEY` أو `MINIMAX_API_KEY`
    - Perplexity: `PERPLEXITY_API_KEY` أو `OPENROUTER_API_KEY`
    - SearXNG: `SEARXNG_BASE_URL`
    - Tavily: `TAVILY_API_KEY`

    ```json5
    {
      plugins: {
        entries: {
          brave: {
            config: {
              webSearch: {
                apiKey: "BRAVE_API_KEY_HERE",
              },
            },
          },
        },
        },
        tools: {
          web: {
            search: {
              enabled: true,
              provider: "brave",
              maxResults: 5,
            },
            fetch: {
              enabled: true,
              provider: "firecrawl", // اختياري؛ اتركه للكشف التلقائي
            },
          },
        },
    }
    ```

    تعيش إعدادات web-search الخاصة بكل مزوّد الآن تحت `plugins.entries.<plugin>.config.webSearch.*`.
    ولا تزال مسارات المزوّد القديمة `tools.web.search.*` تُحمَّل مؤقتًا من أجل التوافق، لكن لا ينبغي استخدامها في الإعدادات الجديدة.
    تعيش إعدادات fallback الخاصة بـ Firecrawl web-fetch تحت `plugins.entries.firecrawl.config.webFetch.*`.

    ملاحظات:

    - إذا كنت تستخدم allowlists، فأضف `web_search`/`web_fetch`/`x_search` أو `group:web`.
    - `web_fetch` مفعّلة افتراضيًا (ما لم تُعطَّل صراحةً).
    - إذا تُرك `tools.web.fetch.provider` من دون ضبط، فسيكتشف OpenClaw تلقائيًا أول مزوّد fallback جاهز للجلب من بين بيانات الاعتماد المتاحة. المزوّد المضمّن اليوم هو Firecrawl.
    - تقرأ daemons متغيرات env من `~/.openclaw/.env` (أو من بيئة الخدمة).

    المستندات: [أدوات الويب](/ar/tools/web).

  </Accordion>

  <Accordion title="config.apply محا إعداداتي. كيف أتعافى وأتجنب ذلك؟">
    يقوم `config.apply` باستبدال **الإعدادات بالكامل**. إذا أرسلت كائنًا جزئيًا، فسيُزال
    كل شيء آخر.

    الاستعادة:

    - استعد من نسخة احتياطية (git أو نسخة من `~/.openclaw/openclaw.json`).
    - إذا لم تكن لديك نسخة احتياطية، فأعد تشغيل `openclaw doctor` وأعد تهيئة القنوات/النماذج.
    - إذا كان هذا غير متوقع، فافتح تقرير خطأ وأدرج آخر إعدادات معروفة لديك أو أي نسخة احتياطية.
    - غالبًا ما يستطيع وكيل برمجة محلي إعادة بناء إعدادات صالحة من السجلات أو المحفوظات.

    تجنّب ذلك:

    - استخدم `openclaw config set` للتغييرات الصغيرة.
    - استخدم `openclaw configure` للتعديلات التفاعلية.
    - استخدم `config.schema.lookup` أولًا عندما لا تكون متأكدًا من المسار الدقيق أو شكل الحقل؛ فهو يعيد عقدة schema سطحية مع ملخصات الأبناء المباشرين للتعمق.
    - استخدم `config.patch` لتعديلات RPC الجزئية؛ واحتفظ بـ `config.apply` فقط لاستبدال الإعدادات كاملةً.
    - إذا كنت تستخدم أداة `gateway` المخصصة للمالك فقط من داخل تشغيل وكيل، فستظل ترفض الكتابات إلى `tools.exec.ask` / `tools.exec.security` (بما في ذلك أسماء `tools.bash.*` القديمة التي تُطبّع إلى المسارات التنفيذية المحمية نفسها).

    المستندات: [الإعدادات](/cli/config)، [Configure](/cli/configure)، [Doctor](/ar/gateway/doctor).

  </Accordion>

  <Accordion title="كيف أشغّل Gateway مركزية مع عمال متخصصين عبر الأجهزة؟">
    النمط الشائع هو **Gateway واحدة** (مثل Raspberry Pi) بالإضافة إلى **nodes** و**agents**:

    - **Gateway (مركزية):** تملك القنوات (Signal/WhatsApp)، والتوجيه، والجلسات.
    - **Nodes (الأجهزة):** تتصل أجهزة Mac/iOS/Android كأجهزة طرفية وتكشف الأدوات المحلية (`system.run`, `canvas`, `camera`).
    - **Agents (العمال):** عقول/مساحات عمل منفصلة لأدوار خاصة (مثل "عمليات Hetzner"، "البيانات الشخصية").
    - **Sub-agents:** أنشئ عملًا في الخلفية من وكيل رئيسي عندما تريد التوازي.
    - **TUI:** اتصل بـ Gateway وبدّل بين الوكلاء/الجلسات.

    المستندات: [Nodes](/ar/nodes)، [الوصول البعيد](/ar/gateway/remote)، [التوجيه متعدد الوكلاء](/ar/concepts/multi-agent)، [Sub-agents](/ar/tools/subagents)، [TUI](/web/tui).

  </Accordion>

  <Accordion title="هل يمكن لمتصفح OpenClaw أن يعمل headless؟">
    نعم. إنه خيار إعداد:

    ```json5
    {
      browser: { headless: true },
      agents: {
        defaults: {
          sandbox: { browser: { headless: true } },
        },
      },
    }
    ```

    القيمة الافتراضية هي `false` (مرئي). ويكون الوضع headless أكثر عرضة لإثارة فحوصات مكافحة الروبوت على بعض المواقع. راجع [المتصفح](/ar/tools/browser).

    يستخدم الوضع headless **محرك Chromium نفسه** ويعمل مع معظم الأتمتة (النماذج، النقرات، الاستخراج، تسجيلات الدخول). والفروق الأساسية هي:

    - لا توجد نافذة متصفح مرئية (استخدم لقطات الشاشة إذا كنت تحتاج إلى مرئيات).
    - بعض المواقع أكثر تشددًا تجاه الأتمتة في وضع headless (CAPTCHAs، مكافحة الروبوت).
      مثلًا، غالبًا ما يحظر X/Twitter الجلسات headless.

  </Accordion>

  <Accordion title="كيف أستخدم Brave للتحكم في المتصفح؟">
    اضبط `browser.executablePath` على ثنائي Brave لديك (أو أي متصفح قائم على Chromium) ثم أعد تشغيل Gateway.
    راجع أمثلة الإعدادات الكاملة في [المتصفح](/ar/tools/browser#use-brave-or-another-chromium-based-browser).
  </Accordion>
</AccordionGroup>

## Gateways البعيدة وnodes

<AccordionGroup>
  <Accordion title="كيف تنتشر الأوامر بين Telegram وgateway وnodes؟">
    تتم معالجة رسائل Telegram بواسطة **gateway**. تقوم gateway بتشغيل الوكيل
    وعندها فقط تستدعي nodes عبر **Gateway WebSocket** عند الحاجة إلى أداة node:

    Telegram → Gateway → Agent → `node.*` → Node → Gateway → Telegram

    لا ترى Nodes حركة المرور الواردة من المزوّد؛ بل تتلقى فقط استدعاءات node RPC.

  </Accordion>

  <Accordion title="كيف يمكن للوكيل الوصول إلى جهازي إذا كانت Gateway مستضافة عن بُعد؟">
    الإجابة المختصرة: **اقرن جهازك كـ node**. تعمل Gateway في مكان آخر، لكنها تستطيع
    استدعاء أدوات `node.*` (screen وcamera وsystem) على جهازك المحلي عبر Gateway WebSocket.

    إعداد نموذجي:

    1. شغّل Gateway على المضيف الذي يعمل دائمًا (VPS/خادم منزلي).
    2. ضع مضيف Gateway + جهازك على tailnet نفسها.
    3. تأكد من إمكانية الوصول إلى Gateway WS (ربط tailnet أو نفق SSH).
    4. افتح تطبيق macOS محليًا واتصل في وضع **Remote over SSH** (أو tailnet مباشر)
       حتى يتمكن من التسجيل كـ node.
    5. وافق على node على Gateway:

       ```bash
       openclaw devices list
       openclaw devices approve <requestId>
       ```

    لا يلزم أي جسر TCP منفصل؛ إذ تتصل nodes عبر Gateway WebSocket.

    تذكير أمني: إقران node macOS يسمح بـ `system.run` على ذلك الجهاز. لا
    تقرن إلا الأجهزة التي تثق بها، وراجع [الأمان](/ar/gateway/security).

    المستندات: [Nodes](/ar/nodes)، [بروتوكول Gateway](/ar/gateway/protocol)، [الوضع البعيد في macOS](/ar/platforms/mac/remote)، [الأمان](/ar/gateway/security).

  </Accordion>

  <Accordion title="Tailscale متصل لكنني لا أتلقى أي ردود. ماذا الآن؟">
    تحقق من الأساسيات:

    - Gateway تعمل: `openclaw gateway status`
    - صحة Gateway: `openclaw status`
    - صحة القنوات: `openclaw channels status`

    ثم تحقق من المصادقة والتوجيه:

    - إذا كنت تستخدم Tailscale Serve، فتأكد من ضبط `gateway.auth.allowTailscale` بشكل صحيح.
    - إذا كنت تتصل عبر نفق SSH، فتأكد من أن النفق المحلي قائم ويشير إلى المنفذ الصحيح.
    - تأكد من أن allowlists لديك (DM أو group) تتضمن حسابك.

    المستندات: [Tailscale](/ar/gateway/tailscale)، [الوصول البعيد](/ar/gateway/remote)، [القنوات](/ar/channels).

  </Accordion>

  <Accordion title="هل يمكن لنسختين من OpenClaw التحدث إلى بعضهما (محلية + VPS)؟">
    نعم. لا يوجد جسر "bot-to-bot" مدمج، لكن يمكنك توصيل ذلك بعدة
    طرق موثوقة:

    **الأبسط:** استخدم قناة دردشة عادية يستطيع botان الوصول إليها (Telegram/Slack/WhatsApp).
    اجعل Bot A يرسل رسالة إلى Bot B، ثم دع Bot B يرد كالمعتاد.

    **جسر CLI (عام):** شغّل نصًا يستدعي Gateway الأخرى باستخدام
    `openclaw agent --message ... --deliver`، مستهدفًا دردشة يصغي فيها bot
    الآخر. وإذا كان أحد botين على VPS بعيد، فاجعل CLI لديك يشير إلى تلك Gateway البعيدة
    عبر SSH/Tailscale (راجع [الوصول البعيد](/ar/gateway/remote)).

    نمط مثال (شغّله من جهاز يستطيع الوصول إلى Gateway المستهدفة):

    ```bash
    openclaw agent --message "Hello from local bot" --deliver --channel telegram --reply-to <chat-id>
    ```

    نصيحة: أضف حاجزًا يمنع botين من الدخول في حلقة لا نهائية (رد عند الذكر فقط،
    أو allowlists للقناة، أو قاعدة "لا ترد على رسائل bot").

    المستندات: [الوصول البعيد](/ar/gateway/remote)، [CLI الخاص بـ Agent](/cli/agent)، [إرسال Agent](/ar/tools/agent-send).

  </Accordion>

  <Accordion title="هل أحتاج إلى VPS منفصل لكل وكيل؟">
    لا. يمكن لـ Gateway واحدة استضافة عدة وكلاء، لكل منهم مساحة عمله، وافتراضيات نماذجه،
    وتوجيهه. هذا هو الإعداد الطبيعي وهو أرخص بكثير وأبسط من تشغيل
    VPS واحدة لكل وكيل.

    استخدم VPS منفصلة فقط عندما تحتاج إلى عزل صارم (حدود أمان) أو
    إعدادات مختلفة جدًا لا تريد مشاركتها. بخلاف ذلك، أبقِ Gateway واحدة
    واستخدم عدة وكلاء أو sub-agents.

  </Accordion>

  <Accordion title="هل هناك فائدة من استخدام node على حاسوبي المحمول الشخصي بدل SSH من VPS؟">
    نعم - تُعد nodes الطريقة الأساسية للوصول إلى حاسوبك المحمول من Gateway بعيدة، وهي
    تفتح أكثر من مجرد وصول shell. تعمل Gateway على macOS/Linux (وWindows عبر WSL2) وهي
    خفيفة (يكفي VPS صغير أو جهاز من فئة Raspberry Pi؛ و4 GB RAM أكثر من كافٍ)، لذا من الشائع
    إعداد مضيف يعمل دائمًا مع حاسوبك المحمول كـ node.

    - **لا حاجة إلى SSH وارد.** تتصل nodes خارجيًا إلى Gateway WebSocket وتستخدم إقران الأجهزة.
    - **ضوابط تنفيذ أكثر أمانًا.** يتم تقييد `system.run` بقوائم السماح/الموافقات الخاصة بـ node على ذلك الحاسوب المحمول.
    - **مزيد من أدوات الجهاز.** تكشف nodes عن `canvas` و`camera` و`screen` إضافةً إلى `system.run`.
    - **أتمتة متصفح محلية.** أبقِ Gateway على VPS، لكن شغّل Chrome محليًا عبر مضيف node على الحاسوب المحمول، أو اتصل بـ Chrome المحلي على المضيف عبر Chrome MCP.

    لا بأس باستخدام SSH للوصول الصدفي إلى shell، لكن nodes أبسط لسير عمل الوكلاء المستمر
    وأتمتة الأجهزة.

    المستندات: [Nodes](/ar/nodes)، [CLI الخاص بـ Nodes](/cli/nodes)، [المتصفح](/ar/tools/browser).

  </Accordion>

  <Accordion title="هل تشغّل nodes خدمة gateway؟">
    لا. يجب أن تعمل **gateway واحدة** فقط لكل مضيف ما لم تكن تشغّل ملفات تعريف معزولة عمدًا (راجع [Gateways متعددة](/ar/gateway/multiple-gateways)). Nodes هي أجهزة طرفية تتصل
    بالـ gateway (عقد iOS/Android، أو "وضع node" في تطبيق شريط القوائم على macOS). وبالنسبة إلى
    مضيفات node headless والتحكم عبر CLI، راجع [CLI الخاص بمضيف Node](/cli/node).

    يلزم إعادة تشغيل كاملة لتغييرات `gateway` و`discovery` و`canvasHost`.

  </Accordion>

  <Accordion title="هل توجد طريقة API / RPC لتطبيق الإعدادات؟">
    نعم.

    - `config.schema.lookup`: افحص شجرة إعدادات فرعية واحدة مع عقدة schema سطحية وتلميح UI مطابق وملخصات الأبناء المباشرين قبل الكتابة
    - `config.get`: اجلب اللقطة الحالية + hash
    - `config.patch`: تحديث جزئي آمن (المفضل لمعظم تعديلات RPC)؛ يقوم بـ hot-reload عند الإمكان ويعيد التشغيل عند الحاجة
    - `config.apply`: يتحقق من الصلاحية + يستبدل الإعدادات كاملةً؛ يقوم بـ hot-reload عند الإمكان ويعيد التشغيل عند الحاجة
    - لا تزال أداة وقت التشغيل `gateway` المخصصة للمالك فقط ترفض إعادة كتابة `tools.exec.ask` / `tools.exec.security`؛ كما تُطبّع أسماء `tools.bash.*` القديمة إلى مسارات exec المحمية نفسها

  </Accordion>

  <Accordion title="أبسط إعداد معقول لأول تثبيت">
    ```json5
    {
      agents: { defaults: { workspace: "~/.openclaw/workspace" } },
      channels: { whatsapp: { allowFrom: ["+15555550123"] } },
    }
    ```

    يضبط هذا مساحة العمل لديك ويقيّد من يمكنه تشغيل bot.

  </Accordion>

  <Accordion title="كيف أضبط Tailscale على VPS وأتصل من Mac الخاص بي؟">
    الحد الأدنى من الخطوات:

    1. **ثبّت + سجّل الدخول على VPS**

       ```bash
       curl -fsSL https://tailscale.com/install.sh | sh
       sudo tailscale up
       ```

    2. **ثبّت + سجّل الدخول على Mac**
       - استخدم تطبيق Tailscale وسجّل الدخول إلى tailnet نفسها.
    3. **فعّل MagicDNS (موصى به)**
       - في وحدة إدارة Tailscale، فعّل MagicDNS بحيث يكون لـ VPS اسم ثابت.
    4. **استخدم اسم مضيف tailnet**
       - SSH: `ssh user@your-vps.tailnet-xxxx.ts.net`
       - Gateway WS: `ws://your-vps.tailnet-xxxx.ts.net:18789`

    إذا كنت تريد Control UI من دون SSH، فاستخدم Tailscale Serve على VPS:

    ```bash
    openclaw gateway --tailscale serve
    ```

    يُبقي هذا gateway مرتبطة بـ loopback ويكشف HTTPS عبر Tailscale. راجع [Tailscale](/ar/gateway/tailscale).

  </Accordion>

  <Accordion title="كيف أوصل node Mac بـ Gateway بعيدة (Tailscale Serve)؟">
    يكشف Serve **Gateway Control UI + WS**. وتتصل nodes عبر نقطة نهاية Gateway WS نفسها.

    الإعداد الموصى به:

    1. **تأكد من أن VPS + Mac على tailnet نفسها**.
    2. **استخدم تطبيق macOS في الوضع البعيد** (يمكن أن يكون هدف SSH هو اسم مضيف tailnet).
       سيقوم التطبيق بعمل tunnel لمنفذ Gateway والاتصال كـ node.
    3. **وافق على node** على gateway:

       ```bash
       openclaw devices list
       openclaw devices approve <requestId>
       ```

    المستندات: [بروتوكول Gateway](/ar/gateway/protocol)، [Discovery](/ar/gateway/discovery)، [الوضع البعيد في macOS](/ar/platforms/mac/remote).

  </Accordion>

  <Accordion title="هل ينبغي أن أثبّت على حاسوب محمول ثانٍ أم أضيف node فقط؟">
    إذا كنت تحتاج فقط إلى **أدوات محلية** (screen/camera/exec) على الحاسوب المحمول الثاني، فأضفه كـ
    **node**. يحافظ ذلك على Gateway واحدة ويتجنب تكرار الإعدادات. أدوات node المحلية
    خاصة بـ macOS حاليًا، لكننا نخطط لتوسيعها إلى أنظمة تشغيل أخرى.

    ثبّت Gateway ثانية فقط عندما تحتاج إلى **عزل صارم** أو botين منفصلين تمامًا.

    المستندات: [Nodes](/ar/nodes)، [CLI الخاص بـ Nodes](/cli/nodes)، [Gateways متعددة](/ar/gateway/multiple-gateways).

  </Accordion>
</AccordionGroup>

## متغيرات env وتحميل .env

<AccordionGroup>
  <Accordion title="كيف يحمّل OpenClaw متغيرات البيئة؟">
    يقرأ OpenClaw متغيرات env من العملية الأب (shell أو launchd/systemd أو CI، إلخ) ويحمّل أيضًا:

    - `.env` من دليل العمل الحالي
    - ملف `.env` احتياطي عالمي من `~/.openclaw/.env` (المعروف أيضًا باسم `$OPENCLAW_STATE_DIR/.env`)

    لا يقوم أي من ملفي `.env` بتجاوز متغيرات env الموجودة أصلًا.

    يمكنك أيضًا تعريف متغيرات env مضمنة في الإعدادات (تُطبَّق فقط إذا كانت مفقودة من process env):

    ```json5
    {
      env: {
        OPENROUTER_API_KEY: "sk-or-...",
        vars: { GROQ_API_KEY: "gsk-..." },
      },
    }
    ```

    راجع [/environment](/ar/help/environment) لمعرفة الأولوية والمصادر بالكامل.

  </Accordion>

  <Accordion title="لقد بدأت Gateway عبر الخدمة واختفت متغيرات env لدي. ماذا الآن؟">
    هناك إصلاحان شائعان:

    1. ضع المفاتيح المفقودة في `~/.openclaw/.env` حتى تُلتقط حتى عندما لا ترث الخدمة بيئة shell لديك.
    2. فعّل استيراد shell (ميزة اختيارية للراحة):

    ```json5
    {
      env: {
        shellEnv: {
          enabled: true,
          timeoutMs: 15000,
        },
      },
    }
    ```

    يشغّل هذا login shell لديك ويستورد فقط المفاتيح المتوقعة المفقودة (من دون تجاوز أي شيء). المقابلات في env:
    `OPENCLAW_LOAD_SHELL_ENV=1`, `OPENCLAW_SHELL_ENV_TIMEOUT_MS=15000`.

  </Accordion>

  <Accordion title='لقد ضبطت COPILOT_GITHUB_TOKEN، لكن models status تعرض "Shell env: off." لماذا؟'>
    تعرض `openclaw models status` ما إذا كان **استيراد shell env** مفعّلًا. وعبارة "Shell env: off"
    لا **تعني** أن متغيرات env لديك مفقودة - بل تعني فقط أن OpenClaw لن يحمّل
    login shell لديك تلقائيًا.

    إذا كانت Gateway تعمل كخدمة (launchd/systemd)، فلن ترث بيئة shell
    لديك. أصلح ذلك بإحدى الطرق التالية:

    1. ضع token في `~/.openclaw/.env`:

       ```
       COPILOT_GITHUB_TOKEN=...
       ```

    2. أو فعّل استيراد shell (`env.shellEnv.enabled: true`).
    3. أو أضفه إلى كتلة `env` في الإعدادات (يُطبّق فقط إذا كان مفقودًا).

    ثم أعد تشغيل gateway وتحقق مجددًا:

    ```bash
    openclaw models status
    ```

    تُقرأ رموز Copilot من `COPILOT_GITHUB_TOKEN` (وأيضًا `GH_TOKEN` / `GITHUB_TOKEN`).
    راجع [/concepts/model-providers](/ar/concepts/model-providers) و[/environment](/ar/help/environment).

  </Accordion>
</AccordionGroup>

## الجلسات والدردشات المتعددة

<AccordionGroup>
  <Accordion title="كيف أبدأ محادثة جديدة؟">
    أرسل `/new` أو `/reset` كرسالة مستقلة. راجع [إدارة الجلسات](/ar/concepts/session).
  </Accordion>

  <Accordion title="هل تُعاد الجلسات تلقائيًا إذا لم أرسل /new أبدًا؟">
    يمكن أن تنتهي صلاحية الجلسات بعد `session.idleMinutes`، لكن هذا **معطّل افتراضيًا** (القيمة الافتراضية **0**).
    اضبطه على قيمة موجبة لتمكين انتهاء الصلاحية عند الخمول. وعند التمكين، فإن الرسالة **التالية**
    بعد فترة الخمول تبدأ معرّف جلسة جديدًا لذلك المفتاح الخاص بالدردشة.
    لا يحذف هذا النصوص - بل يبدأ جلسة جديدة فقط.

    ```json5
    {
      session: {
        idleMinutes: 240,
      },
    }
    ```

  </Accordion>

  <Accordion title="هل توجد طريقة لصنع فريق من نسخ OpenClaw (CEO واحد والعديد من الوكلاء)؟">
    نعم، عبر **التوجيه متعدد الوكلاء** و**sub-agents**. يمكنك إنشاء وكيل
    منسق واحد وعدة وكلاء عمال مع مساحات عمل ونماذج خاصة بهم.

    ومع ذلك، يُنظر إلى هذا على أنه **تجربة ممتعة**. فهو كثيف الرموز وغالبًا
    أقل كفاءة من استخدام bot واحد مع جلسات منفصلة. والنموذج المعتاد الذي
    نتصوره هو bot واحد تتحدث معه، مع جلسات مختلفة للعمل المتوازي. ويمكن لهذا
    bot أيضًا إنشاء sub-agents عند الحاجة.

    المستندات: [التوجيه متعدد الوكلاء](/ar/concepts/multi-agent)، [Sub-agents](/ar/tools/subagents)، [CLI الخاص بالوكلاء](/cli/agents).

  </Accordion>

  <Accordion title="لماذا اقتُطع السياق في منتصف المهمة؟ وكيف أمنع ذلك؟">
    سياق الجلسة محدود بنافذة النموذج. يمكن للمحادثات الطويلة أو مخرجات الأدوات الكبيرة أو كثرة
    الملفات أن تؤدي إلى الضغط أو الاقتطاع.

    ما الذي يساعد:

    - اطلب من bot أن يلخّص الحالة الحالية ويكتبها إلى ملف.
    - استخدم `/compact` قبل المهام الطويلة، و`/new` عند تغيير الموضوع.
    - احتفظ بالسياق المهم في مساحة العمل واطلب من bot قراءته مجددًا.
    - استخدم sub-agents للمهام الطويلة أو المتوازية حتى تبقى الدردشة الرئيسية أصغر.
    - اختر نموذجًا بنافذة سياق أكبر إذا كان هذا يحدث كثيرًا.

  </Accordion>

  <Accordion title="كيف أعيد تعيين OpenClaw بالكامل مع إبقائه مثبتًا؟">
    استخدم أمر reset:

    ```bash
    openclaw reset
    ```

    إعادة تعيين كاملة غير تفاعلية:

    ```bash
    openclaw reset --scope full --yes --non-interactive
    ```

    ثم أعد تشغيل الإعداد:

    ```bash
    openclaw onboard --install-daemon
    ```

    ملاحظات:

    - يعرض الإدخال الأولي أيضًا **Reset** إذا رأى إعدادات موجودة. راجع [الإدخال الأولي (CLI)](/ar/start/wizard).
    - إذا كنت استخدمت ملفات تعريف (`--profile` / `OPENCLAW_PROFILE`)، فأعد تعيين كل دليل حالة (الافتراضي هو `~/.openclaw-<profile>`).
    - إعادة تعيين Dev: `openclaw gateway --dev --reset` (خاصة بـ dev؛ تمحو إعدادات dev + بيانات الاعتماد + الجلسات + مساحة العمل).

  </Accordion>

  <Accordion title='أحصل على أخطاء "context too large" - كيف أعيد التعيين أو أضغط؟'>
    استخدم أحد هذه الخيارات:

    - **الضغط** (يُبقي المحادثة لكنه يلخّص الأدوار الأقدم):

      ```
      /compact
      ```

      أو `/compact <instructions>` لتوجيه الملخص.

    - **إعادة التعيين** (معرّف جلسة جديد لنفس مفتاح الدردشة):

      ```
      /new
      /reset
      ```

    إذا استمر ذلك:

    - فعّل أو اضبط **session pruning** (`agents.defaults.contextPruning`) لقص مخرجات الأدوات القديمة.
    - استخدم نموذجًا بنافذة سياق أكبر.

    المستندات: [الضغط](/ar/concepts/compaction)، [تشذيب الجلسات](/ar/concepts/session-pruning)، [إدارة الجلسات](/ar/concepts/session).

  </Accordion>

  <Accordion title='لماذا أرى "LLM request rejected: messages.content.tool_use.input field required"؟'>
    هذا خطأ تحقق من المزوّد: فقد أصدر النموذج كتلة `tool_use` من دون
    `input` المطلوب. وهذا يعني عادةً أن محفوظات الجلسة قديمة أو تالفة (غالبًا بعد سلاسل طويلة
    أو تغيير في الأداة/schema).

    الحل: ابدأ جلسة جديدة عبر `/new` (رسالة مستقلة).

  </Accordion>

  <Accordion title="لماذا أتلقى رسائل heartbeat كل 30 دقيقة؟">
    تعمل Heartbeats كل **30m** افتراضيًا (**1h** عند استخدام مصادقة OAuth). اضبطها أو عطّلها:

    ```json5
    {
      agents: {
        defaults: {
          heartbeat: {
            every: "2h", // أو "0m" للتعطيل
          },
        },
      },
    }
    ```

    إذا كان `HEARTBEAT.md` موجودًا لكنه فارغ فعليًا (أسطر فارغة فقط وعناوين markdown
    مثل `# Heading`)، فإن OpenClaw يتخطى تشغيل heartbeat لتوفير استدعاءات API.
    وإذا كان الملف مفقودًا، فسيستمر heartbeat في العمل ويقرّر النموذج ما الذي سيفعله.

    تستخدم التجاوزات لكل وكيل `agents.list[].heartbeat`. المستندات: [Heartbeat](/ar/gateway/heartbeat).

  </Accordion>

  <Accordion title='هل أحتاج إلى إضافة "حساب bot" إلى مجموعة WhatsApp؟'>
    لا. يعمل OpenClaw على **حسابك أنت**، لذا إذا كنت موجودًا في المجموعة، فبإمكان OpenClaw رؤيتها.
    افتراضيًا، تُحظر الردود في المجموعات حتى تسمح للمرسلين (`groupPolicy: "allowlist"`).

    إذا كنت تريد أن تتمكن **أنت فقط** من تشغيل الردود في المجموعة:

    ```json5
    {
      channels: {
        whatsapp: {
          groupPolicy: "allowlist",
          groupAllowFrom: ["+15551234567"],
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="كيف أحصل على JID لمجموعة WhatsApp؟">
    الخيار 1 (الأسرع): اعرض السجلات مباشرةً وأرسل رسالة اختبار في المجموعة:

    ```bash
    openclaw logs --follow --json
    ```

    ابحث عن `chatId` (أو `from`) الذي ينتهي بـ `@g.us`، مثل:
    `1234567890-1234567890@g.us`.

    الخيار 2 (إذا كان قد تم التكوين/الإدراج في allowlist بالفعل): اعرض المجموعات من الإعدادات:

    ```bash
    openclaw directory groups list --channel whatsapp
    ```

    المستندات: [WhatsApp](/ar/channels/whatsapp)، [Directory](/cli/directory)، [السجلات](/cli/logs).

  </Accordion>

  <Accordion title="لماذا لا يرد OpenClaw في مجموعة؟">
    سببان شائعان:

    - تفعيل قيد الذكر (الافتراضي). يجب أن تقوم بعمل @mention للـ bot (أو أن تطابق `mentionPatterns`).
    - لقد قمت بتكوين `channels.whatsapp.groups` بدون `"*"` والمجموعة ليست ضمن allowlist.

    راجع [المجموعات](/ar/channels/groups) و[رسائل المجموعات](/ar/channels/group-messages).

  </Accordion>

  <Accordion title="هل تشترك المجموعات/الخيوط في السياق مع الرسائل الخاصة؟">
    تُطوى الدردشات المباشرة إلى الجلسة الرئيسية افتراضيًا. أما المجموعات/القنوات فلها مفاتيح جلسات خاصة بها، كما أن مواضيع Telegram / خيوط Discord هي جلسات منفصلة. راجع [المجموعات](/ar/channels/groups) و[رسائل المجموعات](/ar/channels/group-messages).
  </Accordion>

  <Accordion title="كم عدد مساحات العمل والوكلاء الذين يمكنني إنشاؤهم؟">
    لا توجد حدود صارمة. العشرات (بل المئات) لا بأس بها، لكن انتبه إلى:

    - **نمو القرص:** تعيش الجلسات + النصوص تحت `~/.openclaw/agents/<agentId>/sessions/`.
    - **تكلفة الرموز:** المزيد من الوكلاء يعني استخدامًا متزامنًا أكبر للنماذج.
    - **العبء التشغيلي:** ملفات تعريف المصادقة لكل وكيل، ومساحات العمل، وتوجيه القنوات.

    نصائح:

    - أبقِ **مساحة عمل نشطة** واحدة لكل وكيل (`agents.defaults.workspace`).
    - شذّب الجلسات القديمة (احذف JSONL أو إدخالات المخزن) إذا نما القرص.
    - استخدم `openclaw doctor` لاكتشاف مساحات العمل الشاردة وعدم تطابق ملفات التعريف.

  </Accordion>

  <Accordion title="هل يمكنني تشغيل عدة bots أو دردشات في الوقت نفسه (Slack)، وكيف يجب أن أضبط ذلك؟">
    نعم. استخدم **التوجيه متعدد الوكلاء** لتشغيل عدة وكلاء معزولين وتوجيه الرسائل الواردة حسب
    القناة/الحساب/القرين. Slack مدعومة كقناة ويمكن ربطها بوكلاء محددين.

    وصول المتصفح قوي، لكنه ليس "افعل كل ما يستطيع الإنسان فعله" - فلا تزال أدوات مكافحة الروبوت وCAPTCHAs وMFA
    قادرة على حظر الأتمتة. وللحصول على أكثر تحكم موثوق بالمتصفح، استخدم Chrome MCP المحلي على المضيف،
    أو استخدم CDP على الجهاز الذي يشغّل المتصفح فعلًا.

    أفضل إعداد عملي:

    - مضيف Gateway يعمل دائمًا (VPS/Mac mini).
    - وكيل واحد لكل دور (bindings).
    - قناة/قنوات Slack مربوطة بتلك الوكلاء.
    - متصفح محلي عبر Chrome MCP أو node عند الحاجة.

    المستندات: [التوجيه متعدد الوكلاء](/ar/concepts/multi-agent)، [Slack](/ar/channels/slack)،
    [المتصفح](/ar/tools/browser)، [Nodes](/ar/nodes).

  </Accordion>
</AccordionGroup>

## النماذج: الافتراضيات، الاختيار، الأسماء المستعارة، التبديل

<AccordionGroup>
  <Accordion title='ما هو "النموذج الافتراضي"؟'>
    النموذج الافتراضي في OpenClaw هو أي نموذج تضبطه كالتالي:

    ```
    agents.defaults.model.primary
    ```

    يُشار إلى النماذج بصيغة `provider/model` (مثال: `openai/gpt-5.4`). وإذا حذفت المزوّد، فإن OpenClaw يحاول أولًا اسمًا مستعارًا، ثم مطابقة فريدة لمزوّد مكوَّن لهذا المعرّف الدقيق للنموذج، ثم فقط يعود إلى مزوّد افتراضي مكوَّن كمسار توافق قديم مُهمَل. وإذا لم يعد هذا المزوّد يكشف النموذج الافتراضي المكوَّن، فإن OpenClaw يعود إلى أول مزوّد/نموذج مكوَّن بدلًا من إظهار افتراضي قديم يشير إلى مزوّد أزيل. ومع ذلك يجب عليك **ضبط `provider/model` صراحةً**.

  </Accordion>

  <Accordion title="ما النموذج الذي توصي به؟">
    **الافتراضي الموصى به:** استخدم أقوى نموذج من الجيل الأحدث متاح في مجموعة المزوّدين لديك.
    **بالنسبة إلى الوكلاء المفعّلة بالأدوات أو ذات المدخلات غير الموثوقة:** قدّم قوة النموذج على التكلفة.
    **بالنسبة إلى الدردشة الروتينية/منخفضة المخاطر:** استخدم نماذج احتياطية أرخص ووجّه حسب دور الوكيل.

    لدى MiniMax مستنداتها الخاصة: [MiniMax](/ar/providers/minimax) و
    [النماذج المحلية](/ar/gateway/local-models).

    القاعدة العامة: استخدم **أفضل نموذج يمكنك تحمّل تكلفته** للأعمال عالية المخاطر، ونموذجًا أرخص
    للدردشة الروتينية أو الملخصات. يمكنك توجيه النماذج لكل وكيل واستخدام sub-agents
    لموازاة المهام الطويلة (كل sub-agent يستهلك رموزًا). راجع [النماذج](/ar/concepts/models) و
    [Sub-agents](/ar/tools/subagents).

    تحذير قوي: النماذج الأضعف/المكمّمة أكثر تكون أكثر عرضة لـ prompt
    injection والسلوك غير الآمن. راجع [الأمان](/ar/gateway/security).

    مزيد من السياق: [النماذج](/ar/concepts/models).

  </Accordion>

  <Accordion title="كيف أبدّل النماذج من دون محو إعداداتي؟">
    استخدم **أوامر النموذج** أو عدّل فقط حقول **model**. تجنب الاستبدال الكامل للإعدادات.

    خيارات آمنة:

    - `/model` في الدردشة (سريع، لكل جلسة)
    - `openclaw models set ...` (يحدّث فقط إعدادات النموذج)
    - `openclaw configure --section model` (تفاعلي)
    - عدّل `agents.defaults.model` في `~/.openclaw/openclaw.json`

    تجنب `config.apply` مع كائن جزئي ما لم تكن تقصد استبدال الإعدادات كاملةً.
    وبالنسبة إلى تعديلات RPC، افحص باستخدام `config.schema.lookup` أولًا وفضّل `config.patch`. تمنحك حمولة lookup المسار المطبع، ووثائق/قيود schema السطحية، وملخصات الأبناء المباشرين.
    للتحديثات الجزئية.
    وإذا كنت قد كتبت فوق الإعدادات، فاستعدها من نسخة احتياطية أو أعد تشغيل `openclaw doctor` لإصلاحها.

    المستندات: [النماذج](/ar/concepts/models)، [Configure](/cli/configure)، [الإعدادات](/cli/config)، [Doctor](/ar/gateway/doctor).

  </Accordion>

  <Accordion title="هل يمكنني استخدام نماذج مستضافة ذاتيًا (llama.cpp, vLLM, Ollama)؟">
    نعم. تُعد Ollama أسهل مسار للنماذج المحلية.

    أسرع إعداد:

    1. ثبّت Ollama من `https://ollama.com/download`
    2. اسحب نموذجًا محليًا مثل `ollama pull gemma4`
    3. إذا كنت تريد نماذج سحابية أيضًا، شغّل `ollama signin`
    4. شغّل `openclaw onboard` واختر `Ollama`
    5. اختر `Local` أو `Cloud + Local`

    ملاحظات:

    - يمنحك `Cloud + Local` نماذج سحابية إضافةً إلى نماذج Ollama المحلية
    - لا تحتاج النماذج السحابية مثل `kimi-k2.5:cloud` إلى سحب محلي
    - للتبديل اليدوي، استخدم `openclaw models list` و`openclaw models set ollama/<model>`

    ملاحظة أمان: النماذج الأصغر أو المكمّمة بشدة أكثر عرضة لـ prompt
    injection. نحن نوصي بشدة باستخدام **نماذج كبيرة** لأي bot
    يستطيع استخدام الأدوات. وإذا كنت لا تزال تريد نماذج صغيرة، ففعّل العزل وقوائم سماح صارمة للأدوات.

    المستندات: [Ollama](/ar/providers/ollama)، [النماذج المحلية](/ar/gateway/local-models)،
    [مزودو النماذج](/ar/concepts/model-providers)، [الأمان](/ar/gateway/security)،
    [العزل](/ar/gateway/sandboxing).

  </Accordion>

  <Accordion title="ما النماذج التي تستخدمها OpenClaw وFlawd وKrill؟">
    - يمكن أن تختلف هذه البيئات وقد تتغير مع الوقت؛ ولا توجد توصية ثابتة بمزوّد بعينه.
    - تحقق من إعداد وقت التشغيل الحالي على كل gateway باستخدام `openclaw models status`.
    - بالنسبة إلى الوكلاء الحسّاسين أمنيًا/المفعّلين بالأدوات، استخدم أقوى نموذج من الجيل الأحدث متاح.
  </Accordion>

  <Accordion title="كيف أبدّل النماذج أثناء التشغيل (من دون إعادة تشغيل)؟">
    استخدم الأمر `/model` كرسالة مستقلة:

    ```
    /model sonnet
    /model opus
    /model gpt
    /model gpt-mini
    /model gemini
    /model gemini-flash
    /model gemini-flash-lite
    ```

    هذه هي الأسماء المستعارة المضمّنة. ويمكن إضافة أسماء مستعارة مخصصة عبر `agents.defaults.models`.

    يمكنك عرض النماذج المتاحة عبر `/model` أو `/model list` أو `/model status`.

    يعرض `/model` (و`/model list`) منتقيًا مدمجًا ومرقمًا. اختر بالرقم:

    ```
    /model 3
    ```

    يمكنك أيضًا فرض ملف تعريف مصادقة محدد للمزوّد (لكل جلسة):

    ```
    /model opus@anthropic:default
    /model opus@anthropic:work
    ```

    نصيحة: يعرض `/model status` الوكيل النشط، وملف `auth-profiles.json` المستخدم، وملف تعريف المصادقة الذي ستتم تجربته بعد ذلك.
    كما يعرض نقطة نهاية المزوّد المكوّنة (`baseUrl`) ووضع API (`api`) عند توفرهما.

    **كيف أفك تثبيت ملف تعريف ثبّتُه باستخدام @profile؟**

    أعد تشغيل `/model` **من دون** اللاحقة `@profile`:

    ```
    /model anthropic/claude-opus-4-6
    ```

    وإذا كنت تريد العودة إلى الافتراضي، فاختره من `/model` (أو أرسل `/model <default provider/model>`).
    استخدم `/model status` لتأكيد ملف تعريف المصادقة النشط.

  </Accordion>

  <Accordion title="هل يمكنني استخدام GPT 5.2 للمهام اليومية وCodex 5.3 للبرمجة؟">
    نعم. اضبط أحدهما كافتراضي وبدّل عند الحاجة:

    - **تبديل سريع (لكل جلسة):** `/model gpt-5.4` للمهام اليومية، و`/model openai-codex/gpt-5.4` للبرمجة باستخدام Codex OAuth.
    - **افتراضي + تبديل:** اضبط `agents.defaults.model.primary` على `openai/gpt-5.4`، ثم بدّل إلى `openai-codex/gpt-5.4` عند البرمجة (أو بالعكس).
    - **Sub-agents:** وجّه مهام البرمجة إلى sub-agents ذات نموذج افتراضي مختلف.

    راجع [النماذج](/ar/concepts/models) و[أوامر Slash](/ar/tools/slash-commands).

  </Accordion>

  <Accordion title="كيف أضبط fast mode لـ GPT 5.4؟">
    استخدم إما مفتاح تبديل للجلسة أو افتراضيًا في الإعدادات:

    - **لكل جلسة:** أرسل `/fast on` بينما تستخدم الجلسة `openai/gpt-5.4` أو `openai-codex/gpt-5.4`.
    - **افتراضي لكل نموذج:** اضبط `agents.defaults.models["openai/gpt-5.4"].params.fastMode` على `true`.
    - **Codex OAuth أيضًا:** إذا كنت تستخدم أيضًا `openai-codex/gpt-5.4`، فاضبط العلامة نفسها هناك.

    مثال:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openai/gpt-5.4": {
              params: {
                fastMode: true,
              },
            },
            "openai-codex/gpt-5.4": {
              params: {
                fastMode: true,
              },
            },
          },
        },
      },
    }
    ```

    بالنسبة إلى OpenAI، يطابق fast mode القيمة `service_tier = "priority"` في طلبات Responses الأصلية المدعومة. وتتغلب قيمة `/fast` الخاصة بالجلسة على الافتراضيات الموجودة في الإعدادات.

    راجع [Thinking وfast mode](/ar/tools/thinking) و[OpenAI fast mode](/ar/providers/openai#openai-fast-mode).

  </Accordion>

  <Accordion title='لماذا أرى "Model ... is not allowed" ثم لا يأتي أي رد؟'>
    إذا كانت `agents.defaults.models` مضبوطة، فإنها تصبح **allowlist** لـ `/model` وأي
    تجاوزات جلسة. ويؤدي اختيار نموذج غير موجود في هذه القائمة إلى إرجاع:

    ```
    Model "provider/model" is not allowed. Use /model to list available models.
    ```

    يتم إرجاع هذا الخطأ **بدلًا من** رد عادي. الحل: أضف النموذج إلى
    `agents.defaults.models`، أو أزل allowlist، أو اختر نموذجًا من `/model list`.

  </Accordion>

  <Accordion title='لماذا أرى "Unknown model: minimax/MiniMax-M2.7"؟'>
    هذا يعني أن **المزوّد غير مكوّن** (لم يتم العثور على إعداد MiniMax أو ملف
    تعريف مصادقة)، لذلك لا يمكن حل النموذج.

    قائمة التحقق للإصلاح:

    1. حدّث إلى إصدار OpenClaw حالي (أو شغّل من المصدر `main`)، ثم أعد تشغيل gateway.
    2. تأكد من تكوين MiniMax (عبر المعالج أو JSON)، أو أن مصادقة MiniMax
       موجودة في env/ملفات تعريف المصادقة بحيث يمكن حقن المزوّد المطابق
       (`MINIMAX_API_KEY` من أجل `minimax`، أو `MINIMAX_OAUTH_TOKEN` أو MiniMax
       OAuth المخزن من أجل `minimax-portal`).
    3. استخدم معرّف النموذج الدقيق (حساس لحالة الأحرف) لمسار المصادقة لديك:
       `minimax/MiniMax-M2.7` أو `minimax/MiniMax-M2.7-highspeed` لإعداد
       API key، أو `minimax-portal/MiniMax-M2.7` /
       `minimax-portal/MiniMax-M2.7-highspeed`