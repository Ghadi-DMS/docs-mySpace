---
read_when:
    - عند الإجابة عن أسئلة الدعم الشائعة المتعلقة بالإعداد أو التثبيت أو الإعداد الأولي أو وقت التشغيل
    - عند فرز المشكلات التي يبلغ عنها المستخدمون قبل الانتقال إلى تصحيح أعمق
summary: الأسئلة الشائعة حول إعداد OpenClaw وتهيئته واستخدامه
title: الأسئلة الشائعة
x-i18n:
    generated_at: "2026-04-06T03:14:32Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4d6d09621c6033d580cbcf1ff46f81587d69404d6f64c8d8fd8c3f09185bb920
    source_path: help/faq.md
    workflow: 15
---

# الأسئلة الشائعة

إجابات سريعة مع استكشاف أعمق للأخطاء في البيئات الواقعية (التطوير المحلي، وVPS، والوكلاء المتعددون، وOAuth/مفاتيح API، والتحويل الاحتياطي للنماذج). لتشخيصات وقت التشغيل، راجع [استكشاف الأخطاء وإصلاحها](/ar/gateway/troubleshooting). وللمرجع الكامل للإعدادات، راجع [الإعدادات](/ar/gateway/configuration).

## أول 60 ثانية إذا كان هناك شيء معطّل

1. **الحالة السريعة (أول فحص)**

   ```bash
   openclaw status
   ```

   ملخص محلي سريع: نظام التشغيل + التحديث، وقابلية الوصول إلى gateway/service، والوكلاء/الجلسات، وإعدادات الموفر + مشكلات وقت التشغيل (عندما تكون gateway قابلة للوصول).

2. **تقرير قابل للمشاركة**

   ```bash
   openclaw status --all
   ```

   تشخيص للقراءة فقط مع آخر السجلات (مع إخفاء tokens).

3. **حالة daemon + المنفذ**

   ```bash
   openclaw gateway status
   ```

   يعرض وقت تشغيل المشرف مقابل قابلية وصول RPC، وURL الهدف الخاص بـ probe، وأي إعدادات يُرجّح أن تكون الخدمة قد استخدمتها.

4. **تحققات عميقة**

   ```bash
   openclaw status --deep
   ```

   يشغّل probe حيًا لصحة gateway، بما في ذلك probes القنوات عند دعمها
   (يتطلب gateway قابلة للوصول). راجع [الصحة](/ar/gateway/health).

5. **متابعة أحدث سجل**

   ```bash
   openclaw logs --follow
   ```

   إذا كان RPC متوقفًا، فارجع إلى:

   ```bash
   tail -f "$(ls -t /tmp/openclaw/openclaw-*.log | head -1)"
   ```

   سجلات الملفات منفصلة عن سجلات الخدمة؛ راجع [السجلات](/ar/logging) و[استكشاف الأخطاء وإصلاحها](/ar/gateway/troubleshooting).

6. **تشغيل Doctor (إصلاحات)**

   ```bash
   openclaw doctor
   ```

   يصلح/يهاجر الإعدادات والحالة + يشغّل فحوصات الصحة. راجع [Doctor](/ar/gateway/doctor).

7. **لقطة gateway**

   ```bash
   openclaw health --json
   openclaw health --verbose   # shows the target URL + config path on errors
   ```

   يطلب من gateway العاملة لقطة كاملة (WS-only). راجع [الصحة](/ar/gateway/health).

## البدء السريع وإعداد التشغيل الأول

<AccordionGroup>
  <Accordion title="أنا عالق، ما أسرع طريقة للخروج من هذا المأزق؟">
    استخدم وكيل AI محليًا يمكنه **رؤية جهازك**. هذا أكثر فاعلية بكثير من السؤال
    في Discord، لأن معظم حالات "أنا عالق" تكون **مشكلات إعدادات أو بيئة محلية**
    لا يمكن للمساعدين عن بُعد فحصها.

    - **Claude Code**: [https://www.anthropic.com/claude-code/](https://www.anthropic.com/claude-code/)
    - **OpenAI Codex**: [https://openai.com/codex/](https://openai.com/codex/)

    يمكن لهذه الأدوات قراءة المستودع، وتشغيل الأوامر، وفحص السجلات، والمساعدة في إصلاح
    إعداد جهازك (PATH، والخدمات، والصلاحيات، وملفات المصادقة). امنحها **نسخة المصدر الكاملة**
    عبر تثبيت hackable (git):

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    يقوم هذا بتثبيت OpenClaw **من نسخة git**، بحيث يتمكن الوكيل من قراءة الكود + الوثائق
    والاستدلال على الإصدار الدقيق الذي تشغّله. ويمكنك دائمًا العودة إلى الإصدار المستقر لاحقًا
    عبر إعادة تشغيل المُثبّت بدون `--install-method git`.

    نصيحة: اطلب من الوكيل أن **يضع خطة ويشرف** على الإصلاح (خطوة بخطوة)، ثم ينفّذ
    الأوامر الضرورية فقط. هذا يُبقي التغييرات صغيرة وأسهل في المراجعة.

    إذا اكتشفت خطأً حقيقيًا أو إصلاحًا، فيُرجى فتح GitHub issue أو إرسال PR:
    [https://github.com/openclaw/openclaw/issues](https://github.com/openclaw/openclaw/issues)
    [https://github.com/openclaw/openclaw/pulls](https://github.com/openclaw/openclaw/pulls)

    ابدأ بهذه الأوامر (وشارك المخرجات عند طلب المساعدة):

    ```bash
    openclaw status
    openclaw models status
    openclaw doctor
    ```

    ما الذي تفعله:

    - `openclaw status`: لقطة سريعة لصحة gateway/الوكيل + الإعدادات الأساسية.
    - `openclaw models status`: يتحقق من مصادقة الموفر + توفر النماذج.
    - `openclaw doctor`: يتحقق من مشكلات الإعدادات/الحالة الشائعة ويصلحها.

    فحوصات CLI أخرى مفيدة: `openclaw status --all` و`openclaw logs --follow`،
    و`openclaw gateway status`، و`openclaw health --verbose`.

    حلقة تصحيح سريعة: [أول 60 ثانية إذا كان هناك شيء معطّل](#أول-60-ثانية-إذا-كان-هناك-شيء-معطّل).
    وثائق التثبيت: [التثبيت](/ar/install)، [خيارات المُثبّت](/ar/install/installer)، [التحديث](/ar/install/updating).

  </Accordion>

  <Accordion title="Heartbeat يستمر في التخطي. ماذا تعني أسباب التخطي؟">
    أسباب تخطي heartbeat الشائعة:

    - `quiet-hours`: خارج نافذة active-hours المكوّنة
    - `empty-heartbeat-file`: ملف `HEARTBEAT.md` موجود لكنه يحتوي فقط على هيكل فارغ/عناوين فقط
    - `no-tasks-due`: وضع مهام `HEARTBEAT.md` مفعل لكن لم يحن موعد أي من فترات المهام بعد
    - `alerts-disabled`: تم تعطيل كل إظهار heartbeat (`showOk` و`showAlerts` و`useIndicator` كلها معطلة)

    في وضع المهام، لا يتم تقديم الطوابع الزمنية المستحقة إلا بعد اكتمال تشغيل heartbeat
    فعلي. أما التشغيلات المتخطاة فلا تضع علامة اكتمال على المهام.

    الوثائق: [Heartbeat](/ar/gateway/heartbeat)، [الأتمتة والمهام](/ar/automation).

  </Accordion>

  <Accordion title="الطريقة الموصى بها لتثبيت OpenClaw وإعداده">
    يوصي المستودع بالتشغيل من المصدر واستخدام الإعداد الأولي:

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash
    openclaw onboard --install-daemon
    ```

    يمكن للمعالج أيضًا بناء أصول UI تلقائيًا. وبعد الإعداد الأولي، عادةً ما تشغّل Gateway على المنفذ **18789**.

    من المصدر (للمساهمين/المطورين):

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    pnpm ui:build # auto-installs UI deps on first run
    openclaw onboard
    ```

    إذا لم يكن لديك تثبيت عام بعد، فشغّله عبر `pnpm openclaw onboard`.

  </Accordion>

  <Accordion title="كيف أفتح لوحة التحكم بعد الإعداد الأولي؟">
    يفتح المعالج متصفحك مع URL نظيف للوحة التحكم (من دون token في الرابط) مباشرة بعد الإعداد الأولي، ويطبع الرابط أيضًا في الملخص. أبقِ ذلك التبويب مفتوحًا؛ وإذا لم يُفتح، انسخ/ألصق URL المطبوع على الجهاز نفسه.
  </Accordion>

  <Accordion title="كيف أصادق على لوحة التحكم على localhost مقابل الوضع البعيد؟">
    **Localhost (نفس الجهاز):**

    - افتح `http://127.0.0.1:18789/`.
    - إذا طلب مصادقة shared-secret، الصق token أو كلمة المرور المكوّنة في إعدادات Control UI.
    - مصدر token: ‏`gateway.auth.token` (أو `OPENCLAW_GATEWAY_TOKEN`).
    - مصدر كلمة المرور: ‏`gateway.auth.password` (أو `OPENCLAW_GATEWAY_PASSWORD`).
    - إذا لم يتم تكوين shared secret بعد، فأنشئ token عبر `openclaw doctor --generate-gateway-token`.

    **ليس على localhost:**

    - **Tailscale Serve** (مستحسن): أبقِ الربط على loopback، وشغّل `openclaw gateway --tailscale serve`، ثم افتح `https://<magicdns>/`. إذا كانت `gateway.auth.allowTailscale` تساوي `true`، فإن ترويسات الهوية تلبّي مصادقة Control UI/WebSocket (من دون لصق shared secret، مع افتراض الثقة في مضيف gateway)؛ أما HTTP APIs فتظل تتطلب مصادقة shared-secret ما لم تستخدم عمدًا private-ingress `none` أو مصادقة HTTP من trusted-proxy.
      تتم سلسلة محاولات Serve الخاطئة المتزامنة من العميل نفسه قبل أن يسجل محدد failed-auth المحاولة، لذلك قد تُظهر المحاولة الخاطئة الثانية بالفعل الرسالة `retry later`.
    - **ربط Tailnet**: شغّل `openclaw gateway --bind tailnet --token "<token>"` (أو كوّن مصادقة كلمة مرور)، وافتح `http://<tailscale-ip>:18789/`، ثم الصق shared secret المطابق في إعدادات لوحة التحكم.
    - **وكيل عكسي مدرك للهوية**: أبقِ Gateway خلف trusted proxy غير loopback، واضبط `gateway.auth.mode: "trusted-proxy"`، ثم افتح URL الخاص بالوكيل.
    - **SSH tunnel**: ‏`ssh -N -L 18789:127.0.0.1:18789 user@host` ثم افتح `http://127.0.0.1:18789/`. تظل مصادقة shared-secret مطبقة عبر النفق؛ الصق token أو كلمة المرور المكوّنة إذا طُلب منك ذلك.

    راجع [لوحة التحكم](/web/dashboard) و[أسطح الويب](/web) لمعرفة أوضاع الربط وتفاصيل المصادقة.

  </Accordion>

  <Accordion title="لماذا توجد إعداداتان مختلفتان لاعتماد exec لموافقات الدردشة؟">
    تتحكمان في طبقتين مختلفتين:

    - `approvals.exec`: يعيد توجيه مطالبات الموافقة إلى وجهات الدردشة
    - `channels.<channel>.execApprovals`: يجعل تلك القناة تعمل كعميل موافقة أصلي لموافقات exec

    تظل سياسة exec على المضيف هي بوابة الموافقة الحقيقية. أما إعدادات الدردشة فتتحكم فقط في
    مكان ظهور مطالبات الموافقة وكيف يمكن للناس الرد عليها.

    في معظم الإعدادات **لا** تحتاج إلى كليهما:

    - إذا كانت الدردشة تدعم الأوامر والردود بالفعل، فإن `/approve` داخل الدردشة نفسها يعمل عبر المسار المشترك.
    - إذا كانت قناة أصلية مدعومة تستطيع استنتاج الموافقين بأمان، فإن OpenClaw يفعّل الآن تلقائيًا الموافقات الأصلية أولاً في الرسائل الخاصة عندما تكون `channels.<channel>.execApprovals.enabled` غير معيّنة أو تساوي `"auto"`.
    - عندما تكون بطاقات/أزرار الموافقة الأصلية متاحة، تكون UI الأصلية هي المسار الأساسي؛ ويجب على الوكيل أن يضمّن أمر `/approve` يدويًا فقط إذا كانت نتيجة الأداة تقول إن موافقات الدردشة غير متاحة أو أن الموافقة اليدوية هي المسار الوحيد.
    - استخدم `approvals.exec` فقط عندما يجب أيضًا إعادة توجيه المطالبات إلى دردشات أخرى أو غرف عمليات صريحة.
    - استخدم `channels.<channel>.execApprovals.target: "channel"` أو `"both"` فقط عندما تريد صراحةً نشر مطالبات الموافقة مرة أخرى في الغرفة/الموضوع الأصلي.
    - موافقات plugins منفصلة مرة أخرى: فهي تستخدم `/approve` داخل الدردشة نفسها افتراضيًا، مع إعادة توجيه `approvals.plugin` اختيارية، ولا تحتفظ إلا بعض القنوات الأصلية بمعالجة plugin-approval-native فوق ذلك.

    باختصار: إعادة التوجيه للتوجيه، وإعداد العميل الأصلي لتجربة مستخدم أغنى خاصة بالقناة.
    راجع [موافقات Exec](/ar/tools/exec-approvals).

  </Accordion>

  <Accordion title="ما بيئة التشغيل التي أحتاجها؟">
    يلزم Node **>= 22**. يُوصى باستخدام `pnpm`. أما Bun فهو **غير موصى به** للـ Gateway.
  </Accordion>

  <Accordion title="هل يعمل على Raspberry Pi؟">
    نعم. إن Gateway خفيفة - تذكر الوثائق أن **512MB-1GB RAM** و**نواة واحدة** وحوالي **500MB**
    من القرص تكفي للاستخدام الشخصي، وتذكر أن **Raspberry Pi 4 يمكنه تشغيلها**.

    إذا كنت تريد هامشًا إضافيًا (للسجلات، والوسائط، والخدمات الأخرى)، فـ **2GB موصى بها**، لكنها
    ليست حدًا أدنى صارمًا.

    نصيحة: يمكن لجهاز Pi/VPS صغير استضافة Gateway، ويمكنك إقران **nodes** على الكمبيوتر المحمول/الهاتف من أجل
    الشاشة/الكاميرا/canvas المحلية أو تنفيذ الأوامر. راجع [Nodes](/ar/nodes).

  </Accordion>

  <Accordion title="هل توجد نصائح لتثبيت Raspberry Pi؟">
    باختصار: يعمل، لكن توقّع بعض الحواف الخشنة.

    - استخدم نظام تشغيل **64-bit** وحافظ على Node >= 22.
    - فضّل تثبيت **hackable (git)** حتى تتمكن من رؤية السجلات والتحديث بسرعة.
    - ابدأ من دون قنوات/Skills، ثم أضفها واحدة تلو الأخرى.
    - إذا واجهت مشكلات ثنائية غريبة، فعادةً ما تكون مشكلة **توافق ARM**.

    الوثائق: [Linux](/ar/platforms/linux)، [التثبيت](/ar/install).

  </Accordion>

  <Accordion title="إنه عالق على wake up my friend / الإعداد الأولي لا يكتمل. ماذا الآن؟">
    تعتمد تلك الشاشة على أن تكون Gateway قابلة للوصول ومصادقًا عليها. كما أن TUI ترسل
    "Wake up, my friend!" تلقائيًا عند الفقس الأول. إذا رأيت هذا السطر مع **عدم وجود رد**
    وبقيت tokens عند 0، فهذا يعني أن الوكيل لم يعمل مطلقًا.

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

    إذا كانت Gateway بعيدة، فتأكد من أن اتصال tunnel/Tailscale قائم وأن UI
    تشير إلى Gateway الصحيحة. راجع [الوصول البعيد](/ar/gateway/remote).

  </Accordion>

  <Accordion title="هل يمكنني نقل إعدادي إلى جهاز جديد (Mac mini) من دون إعادة الإعداد الأولي؟">
    نعم. انسخ **دليل الحالة** و**مساحة العمل**، ثم شغّل Doctor مرة واحدة. هذا
    يُبقي bot لديك "مطابقًا تمامًا" (الذاكرة، وسجل الجلسات، والمصادقة، وحالة القناة)
    ما دمت تنسخ **الموقعين** معًا:

    1. ثبّت OpenClaw على الجهاز الجديد.
    2. انسخ `$OPENCLAW_STATE_DIR` (الافتراضي: `~/.openclaw`) من الجهاز القديم.
    3. انسخ مساحة العمل الخاصة بك (الافتراضي: `~/.openclaw/workspace`).
    4. شغّل `openclaw doctor` وأعد تشغيل خدمة Gateway.

    سيحافظ ذلك على الإعدادات، وملفات تعريف المصادقة، وبيانات اعتماد WhatsApp، والجلسات، والذاكرة. وإذا كنت في
    الوضع البعيد، فتذكر أن مضيف gateway هو الذي يملك مخزن الجلسات ومساحة العمل.

    **مهم:** إذا قمت فقط بعمل commit/push لمساحة العمل إلى GitHub، فأنت تنسخ
    احتياطيًا **الذاكرة + ملفات bootstrap**، لكن **ليس** سجل الجلسات أو المصادقة. فهذه تعيش
    تحت `~/.openclaw/` (على سبيل المثال `~/.openclaw/agents/<agentId>/sessions/`).

    ذو صلة: [الترحيل](/ar/install/migrating)، [أين توجد الأشياء على القرص](#أين-توجد-الأشياء-على-القرص)،
    [مساحة عمل الوكيل](/ar/concepts/agent-workspace)، [Doctor](/ar/gateway/doctor)،
    [الوضع البعيد](/ar/gateway/remote).

  </Accordion>

  <Accordion title="أين أرى ما الجديد في أحدث إصدار؟">
    تحقق من سجل التغييرات على GitHub:
    [https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md)

    أحدث الإدخالات في الأعلى. إذا كان القسم العلوي يحمل علامة **Unreleased**، فإن القسم المؤرخ التالي
    هو أحدث إصدار تم شحنه. تُجمَّع الإدخالات تحت **Highlights** و**Changes** و
    **Fixes** (مع أقسام docs/أخرى عند الحاجة).

  </Accordion>

  <Accordion title="لا يمكن الوصول إلى docs.openclaw.ai (خطأ SSL)">
    تقوم بعض اتصالات Comcast/Xfinity بحظر `docs.openclaw.ai` بشكل غير صحيح عبر Xfinity
    Advanced Security. عطّلها أو أضف `docs.openclaw.ai` إلى قائمة السماح، ثم أعد المحاولة.
    يرجى مساعدتنا في رفع الحظر عنه بالإبلاغ هنا: [https://spa.xfinity.com/check_url_status](https://spa.xfinity.com/check_url_status).

    وإذا ظل الموقع غير قابل للوصول، فالوثائق معكوسة أيضًا على GitHub:
    [https://github.com/openclaw/openclaw/tree/main/docs](https://github.com/openclaw/openclaw/tree/main/docs)

  </Accordion>

  <Accordion title="الفرق بين stable وbeta">
    كل من **Stable** و**beta** هما **npm dist-tags**، وليسا خطّي كود منفصلين:

    - `latest` = stable
    - `beta` = إصدار مبكر للاختبار

    عادةً، يصل الإصدار المستقر إلى **beta** أولاً، ثم تنقل خطوة ترقية
    صريحة هذا الإصدار نفسه إلى `latest`. ويمكن للمشرفين أيضًا
    النشر مباشرة إلى `latest` عند الحاجة. ولهذا السبب قد يشير beta وstable إلى **الإصدار نفسه** بعد الترقية.

    راجع ما الذي تغيّر:
    [https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md)

    لمعرفة أوامر التثبيت المختصرة والفرق بين beta وdev، راجع العنصر المطوي أدناه.

  </Accordion>

  <Accordion title="كيف أثبت إصدار beta وما الفرق بين beta وdev؟">
    **Beta** هو npm dist-tag باسم `beta` (وقد يطابق `latest` بعد الترقية).
    أما **Dev** فهو رأس الفرع المتحرك `main` (git)؛ وعند نشره يستخدم npm dist-tag باسم `dev`.

    أوامر مختصرة (macOS/Linux):

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --beta
    ```

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    مُثبّت Windows ‏(PowerShell):
    [https://openclaw.ai/install.ps1](https://openclaw.ai/install.ps1)

    مزيد من التفاصيل: [قنوات التطوير](/ar/install/development-channels) و[خيارات المُثبّت](/ar/install/installer).

  </Accordion>

  <Accordion title="كيف أجرب أحدث الأجزاء؟">
    لديك خياران:

    1. **قناة Dev ‏(نسخة git):**

    ```bash
    openclaw update --channel dev
    ```

    سيؤدي هذا إلى التحويل إلى الفرع `main` والتحديث من المصدر.

    2. **تثبيت Hackable ‏(من موقع المُثبّت):**

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    يمنحك هذا مستودعًا محليًا يمكنك تعديله ثم تحديثه عبر git.

    إذا كنت تفضّل نسخة clone نظيفة يدويًا، فاستخدم:

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    ```

    الوثائق: [التحديث](/cli/update)، [قنوات التطوير](/ar/install/development-channels)،
    [التثبيت](/ar/install).

  </Accordion>

  <Accordion title="كم يستغرق التثبيت والإعداد الأولي عادةً؟">
    دليل تقريبي:

    - **التثبيت:** 2-5 دقائق
    - **الإعداد الأولي:** 5-15 دقيقة بحسب عدد القنوات/النماذج التي تهيئها

    إذا علِق، فاستخدم [تعطّل المُثبّت](#البدء-السريع-وإعداد-التشغيل-الأول)
    وحلقة التصحيح السريعة في [أنا عالق](#البدء-السريع-وإعداد-التشغيل-الأول).

  </Accordion>

  <Accordion title="المُثبّت عالق؟ كيف أحصل على مزيد من الملاحظات؟">
    أعد تشغيل المُثبّت مع **مخرجات verbose**:

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --verbose
    ```

    تثبيت Beta مع verbose:

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --beta --verbose
    ```

    لتثبيت hackable ‏(git):

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git --verbose
    ```

    المكافئ في Windows ‏(PowerShell):

    ```powershell
    # install.ps1 has no dedicated -Verbose flag yet.
    Set-PSDebug -Trace 1
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -NoOnboard
    Set-PSDebug -Trace 0
    ```

    مزيد من الخيارات: [خيارات المُثبّت](/ar/install/installer).

  </Accordion>

  <Accordion title="تثبيت Windows يقول git not found أو openclaw not recognized">
    مشكلتان شائعتان في Windows:

    **1) خطأ npm spawn git / git not found**

    - ثبّت **Git for Windows** وتأكد من أن `git` موجود على PATH لديك.
    - أغلق PowerShell وأعد فتحها، ثم أعد تشغيل المُثبّت.

    **2) openclaw is not recognized بعد التثبيت**

    - مجلد npm global bin الخاص بك ليس موجودًا على PATH.
    - تحقق من المسار:

      ```powershell
      npm config get prefix
      ```

    - أضف هذا الدليل إلى PATH الخاص بالمستخدم (لا حاجة إلى لاحقة `\bin` في Windows؛ ففي معظم الأنظمة يكون `%AppData%\npm`).
    - أغلق PowerShell وأعد فتحها بعد تحديث PATH.

    إذا كنت تريد أكثر إعداد سلس في Windows، فاستخدم **WSL2** بدلًا من Windows الأصلي.
    الوثائق: [Windows](/ar/platforms/windows).

  </Accordion>

  <Accordion title="مخرجات exec في Windows تعرض نصًا صينيًا مشوهًا - ماذا أفعل؟">
    يكون هذا عادةً بسبب عدم تطابق console code page في أصداف Windows الأصلية.

    الأعراض:

    - عرض مخرجات `system.run`/`exec` للنص الصيني بشكل mojibake
    - ظهور الأمر نفسه بشكل صحيح في ملف تعريف طرفية آخر

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

    إذا كنت لا تزال تعيد إنتاج ذلك على أحدث إصدار من OpenClaw، فتتبعه/أبلغ عنه هنا:

    - [Issue #30640](https://github.com/openclaw/openclaw/issues/30640)

  </Accordion>

  <Accordion title="لم تجبني الوثائق على سؤالي - كيف أحصل على إجابة أفضل؟">
    استخدم تثبيت **hackable (git)** حتى يكون لديك المصدر والوثائق كاملة محليًا، ثم اسأل
    bot الخاص بك (أو Claude/Codex) _من ذلك المجلد_ حتى يتمكن من قراءة المستودع والإجابة بدقة.

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    مزيد من التفاصيل: [التثبيت](/ar/install) و[خيارات المُثبّت](/ar/install/installer).

  </Accordion>

  <Accordion title="كيف أثبت OpenClaw على Linux؟">
    الإجابة القصيرة: اتبع دليل Linux، ثم شغّل الإعداد الأولي.

    - المسار السريع لـ Linux + تثبيت الخدمة: [Linux](/ar/platforms/linux).
    - الشرح الكامل: [البدء](/ar/start/getting-started).
    - المُثبّت + التحديثات: [التثبيت والتحديثات](/ar/install/updating).

  </Accordion>

  <Accordion title="كيف أثبت OpenClaw على VPS؟">
    يعمل أي Linux VPS. ثبّت على الخادم، ثم استخدم SSH/Tailscale للوصول إلى Gateway.

    الأدلة: [exe.dev](/ar/install/exe-dev)، [Hetzner](/ar/install/hetzner)، [Fly.io](/ar/install/fly).
    الوصول البعيد: [Gateway remote](/ar/gateway/remote).

  </Accordion>

  <Accordion title="أين توجد أدلة التثبيت السحابي/VPS؟">
    نحتفظ بـ **مركز استضافة** يضم الموفرين الشائعين. اختر واحدًا واتبع الدليل:

    - [استضافة VPS](/ar/vps) (كل الموفرين في مكان واحد)
    - [Fly.io](/ar/install/fly)
    - [Hetzner](/ar/install/hetzner)
    - [exe.dev](/ar/install/exe-dev)

    كيف يعمل ذلك في السحابة: **تعمل Gateway على الخادم**، وتصل إليها
    من الكمبيوتر المحمول/الهاتف عبر Control UI ‏(أو Tailscale/SSH). تعيش الحالة + مساحة العمل
    على الخادم، لذا تعامل مع المضيف على أنه مصدر الحقيقة وقم بعمل نسخة احتياطية له.

    يمكنك إقران **nodes** ‏(Mac/iOS/Android/headless) مع Gateway السحابية تلك للوصول إلى
    الشاشة/الكاميرا/canvas المحلية أو تشغيل الأوامر على الكمبيوتر المحمول مع إبقاء
    Gateway في السحابة.

    المركز: [المنصات](/ar/platforms). الوصول البعيد: [Gateway remote](/ar/gateway/remote).
    Nodes: ‏[Nodes](/ar/nodes)، [Nodes CLI](/cli/nodes).

  </Accordion>

  <Accordion title="هل يمكنني أن أطلب من OpenClaw تحديث نفسه؟">
    الإجابة القصيرة: **ممكن، لكنه غير موصى به**. قد تؤدي عملية التحديث إلى إعادة تشغيل
    Gateway ‏(مما يُسقط الجلسة النشطة)، وقد تحتاج إلى نسخة git نظيفة، وقد
    تطلب تأكيدًا. الأكثر أمانًا: نفّذ التحديثات من shell بصفتك المشغّل.

    استخدم CLI:

    ```bash
    openclaw update
    openclaw update status
    openclaw update --channel stable|beta|dev
    openclaw update --tag <dist-tag|version>
    openclaw update --no-restart
    ```

    إذا اضطررت إلى الأتمتة من وكيل:

    ```bash
    openclaw update --yes --no-restart
    openclaw gateway restart
    ```

    الوثائق: [التحديث](/cli/update)، [التحديث](/ar/install/updating).

  </Accordion>

  <Accordion title="ماذا يفعل الإعداد الأولي فعليًا؟">
    يُعد `openclaw onboard` مسار الإعداد الموصى به. في **الوضع المحلي** يرشدك عبر:

    - **إعداد النموذج/المصادقة** (OAuth للموفر، ومفاتيح API، وsetup-token القديم لـ Anthropic، بالإضافة إلى خيارات النماذج المحلية مثل LM Studio)
    - موقع **مساحة العمل** + ملفات bootstrap
    - **إعدادات Gateway** ‏(bind/port/auth/tailscale)
    - **القنوات** ‏(WhatsApp وTelegram وDiscord وMattermost وSignal وiMessage، بالإضافة إلى plugins القنوات المضمّنة مثل QQ Bot)
    - **تثبيت daemon** ‏(LaunchAgent على macOS؛ ووحدة مستخدم systemd على Linux/WSL2)
    - **فحوصات الصحة** واختيار **Skills**

    كما يحذرك إذا كان النموذج الذي أعددته غير معروف أو يفتقد المصادقة.

  </Accordion>

  <Accordion title="هل أحتاج إلى اشتراك Claude أو OpenAI لتشغيل هذا؟">
    لا. يمكنك تشغيل OpenClaw باستخدام **مفاتيح API** ‏(Anthropic/OpenAI/وغيرها) أو باستخدام
    **نماذج محلية فقط** بحيث تبقى بياناتك على جهازك. أما الاشتراكات (Claude
    Pro/Max أو OpenAI Codex) فهي طرق اختيارية لمصادقة تلك الموفرات.

    بالنسبة إلى Anthropic داخل OpenClaw، فالانقسام العملي هو:

    - **مفتاح API لـ Anthropic**: فوترة Anthropic API العادية
    - **مصادقة اشتراك Claude داخل OpenClaw**: أخبرت Anthropic مستخدمي OpenClaw في
      **4 أبريل 2026 الساعة 12:00 ظهرًا بتوقيت PT / 8:00 مساءً بتوقيت BST** أن هذا يتطلب
      **Extra Usage** يُفوَّتر بشكل منفصل عن الاشتراك

    وتُظهر تجاربنا المحلية أيضًا أن `claude -p --append-system-prompt ...` يمكن أن
    يصطدم بحاجز Extra Usage نفسه عندما يعرّف appended prompt
    OpenClaw، بينما لا يُعاد إنتاج هذا الحظر مع النص نفسه
    على مسار Anthropic SDK + API-key. أما OpenAI Codex OAuth فهو مدعوم صراحةً
    للأدوات الخارجية مثل OpenClaw.

    كما يدعم OpenClaw خيارات أخرى مستضافة بنمط الاشتراك، منها
    **Qwen Cloud Coding Plan**، و**MiniMax Coding Plan**، و
    **Z.AI / GLM Coding Plan**.

    الوثائق: [Anthropic](/ar/providers/anthropic)، [OpenAI](/ar/providers/openai)،
    [Qwen Cloud](/ar/providers/qwen)،
    [MiniMax](/ar/providers/minimax)، [GLM Models](/ar/providers/glm)،
    [النماذج المحلية](/ar/gateway/local-models)، [النماذج](/ar/concepts/models).

  </Accordion>

  <Accordion title="هل يمكنني استخدام اشتراك Claude Max من دون مفتاح API؟">
    نعم، لكن تعامل معه على أنه **مصادقة اشتراك Claude مع Extra Usage**.

    لا تتضمن اشتراكات Claude Pro/Max مفتاح API. وفي OpenClaw، يعني هذا
    أن إشعار الفوترة الخاص بـ Anthropic وOpenClaw ينطبق: فحركة المرور عبر الاشتراك
    تتطلب **Extra Usage**. وإذا كنت تريد حركة Anthropic من دون
    هذا المسار، فاستخدم مفتاح API لـ Anthropic بدلًا من ذلك.

  </Accordion>

  <Accordion title="هل تدعمون مصادقة اشتراك Claude (Claude Pro أو Max)؟">
    نعم، لكن التفسير المدعوم الآن هو:

    - Anthropic في OpenClaw مع اشتراك تعني **Extra Usage**
    - Anthropic في OpenClaw من دون هذا المسار تعني **API key**

    لا يزال setup-token من Anthropic متاحًا باعتباره مسارًا قديمًا/يدويًا في OpenClaw،
    ولا يزال إشعار الفوترة الخاص بـ Anthropic وOpenClaw ينطبق هناك. وقد
    أعدنا أيضًا إنتاج حاجز الفوترة نفسه محليًا مع الاستخدام المباشر
    لـ `claude -p --append-system-prompt ...` عندما كان appended prompt
    يعرّف OpenClaw، بينما لم يُعد إنتاج النص نفسه على
    مسار Anthropic SDK + API-key.

    بالنسبة إلى أعباء العمل الإنتاجية أو متعددة المستخدمين، تظل مصادقة مفتاح API من Anthropic
    الخيار الأكثر أمانًا والمُوصى به. وإذا أردت خيارات أخرى مستضافة
    بنمط الاشتراك داخل OpenClaw، فراجع [OpenAI](/ar/providers/openai)، و[Qwen / Model
    Cloud](/ar/providers/qwen)، و[MiniMax](/ar/providers/minimax)، و
    [GLM Models](/ar/providers/glm).

  </Accordion>

<a id="why-am-i-seeing-http-429-ratelimiterror-from-anthropic"></a>
<Accordion title="لماذا أرى HTTP 429 rate_limit_error من Anthropic؟">
هذا يعني أن **حصة/حد rate limit في Anthropic** قد نُفدت في النافذة الحالية. إذا كنت
تستخدم **Claude CLI**، فانتظر إعادة ضبط النافذة أو قم بترقية خطتك. وإذا كنت
تستخدم **مفتاح API لـ Anthropic**، فتحقق من Anthropic Console
من الاستخدام/الفوترة وارفع الحدود حسب الحاجة.

    إذا كانت الرسالة تحديدًا:
    `Extra usage is required for long context requests`، فهذا يعني أن الطلب يحاول استخدام
    نسخة Anthropic التجريبية لسياق 1M (`context1m: true`). ولا يعمل هذا إلا عندما تكون
    بيانات اعتمادك مؤهلة لفوترة long-context (فوترة API key أو
    مسار تسجيل الدخول Claude في OpenClaw مع تفعيل Extra Usage).

    نصيحة: اضبط **نموذجًا احتياطيًا** حتى يتمكن OpenClaw من مواصلة الرد بينما يكون أحد الموفرين مقيّدًا.
    راجع [النماذج](/cli/models)، و[OAuth](/ar/concepts/oauth)، و
    [/gateway/troubleshooting#anthropic-429-extra-usage-required-for-long-context](/ar/gateway/troubleshooting#anthropic-429-extra-usage-required-for-long-context).

  </Accordion>

  <Accordion title="هل AWS Bedrock مدعوم؟">
    نعم. لدى OpenClaw موفر **Amazon Bedrock (Converse)** مضمّن. ومع وجود علامات AWS env، يمكن لـ OpenClaw اكتشاف كتالوج Bedrock الخاص بالبث/النص تلقائيًا ودمجه كموفر ضمني باسم `amazon-bedrock`؛ وإلا فيمكنك تمكين `plugins.entries.amazon-bedrock.config.discovery.enabled` صراحةً أو إضافة إدخال موفر يدويًا. راجع [Amazon Bedrock](/ar/providers/bedrock) و[موفري النماذج](/ar/providers/models). وإذا كنت تفضل تدفق مفاتيح مُدارًا، فسيظل وكيل متوافق مع OpenAI أمام Bedrock خيارًا صالحًا.
  </Accordion>

  <Accordion title="كيف تعمل مصادقة Codex؟">
    يدعم OpenClaw **OpenAI Code (Codex)** عبر OAuth ‏(تسجيل الدخول إلى ChatGPT). ويمكن لعملية الإعداد الأولي تشغيل تدفق OAuth وستضبط النموذج الافتراضي على `openai-codex/gpt-5.4` عند الاقتضاء. راجع [موفري النماذج](/ar/concepts/model-providers) و[الإعداد الأولي (CLI)](/ar/start/wizard).
  </Accordion>

  <Accordion title="هل تدعمون مصادقة اشتراك OpenAI (Codex OAuth)؟">
    نعم. يدعم OpenClaw بالكامل **OpenAI Code (Codex) subscription OAuth**.
    تسمح OpenAI صراحةً باستخدام subscription OAuth في الأدوات/سير العمل الخارجية
    مثل OpenClaw. ويمكن لعملية الإعداد الأولي تشغيل تدفق OAuth نيابةً عنك.

    راجع [OAuth](/ar/concepts/oauth)، و[موفري النماذج](/ar/concepts/model-providers)، و[الإعداد الأولي (CLI)](/ar/start/wizard).

  </Accordion>

  <Accordion title="كيف أعد Gemini CLI OAuth؟">
    يستخدم Gemini CLI **تدفق مصادقة plugin**، وليس client id أو secret داخل `openclaw.json`.

    استخدم موفر Gemini API بدلًا من ذلك:

    1. فعّل plugin: ‏`openclaw plugins enable google`
    2. شغّل `openclaw onboard --auth-choice gemini-api-key`
    3. اضبط نموذج Google مثل `google/gemini-3.1-pro-preview`

  </Accordion>

  <Accordion title="هل النموذج المحلي مناسب للدردشات العادية؟">
    غالبًا لا. يحتاج OpenClaw إلى سياق كبير + أمان قوي؛ البطاقات الصغيرة تُقصّ وتسرّب. وإذا اضطررت، فشغّل **أكبر** بناء نموذج يمكنك تشغيله محليًا (LM Studio) وراجع [/gateway/local-models](/ar/gateway/local-models). تزيد النماذج الأصغر/المكمّمة من خطر prompt injection - راجع [الأمان](/ar/gateway/security).
  </Accordion>

  <Accordion title="كيف أحافظ على حركة المرور الخاصة بالنماذج المستضافة داخل منطقة محددة؟">
    اختر endpoints مثبتة المنطقة. يوفّر OpenRouter خيارات مستضافة في الولايات المتحدة لـ MiniMax وKimi وGLM؛ اختر المتغير المستضاف في الولايات المتحدة للإبقاء على البيانات داخل المنطقة. ويمكنك مع ذلك إدراج Anthropic/OpenAI إلى جانب هذه الخيارات باستخدام `models.mode: "merge"` حتى تظل النماذج الاحتياطية متاحة مع احترام الموفر الإقليمي الذي اخترته.
  </Accordion>

  <Accordion title="هل يجب أن أشتري Mac Mini لتثبيت هذا؟">
    لا. يعمل OpenClaw على macOS أو Linux ‏(وWindows عبر WSL2). إن Mac mini اختياري -
    بعض الأشخاص يشترونه كمضيف دائم التشغيل، لكن VPS صغيرًا أو خادمًا منزليًا أو جهازًا بمستوى Raspberry Pi يكفي أيضًا.

    تحتاج إلى Mac **فقط للأدوات الخاصة بـ macOS**. أما بالنسبة إلى iMessage، فاستخدم [BlueBubbles](/ar/channels/bluebubbles) (مستحسن) - إذ يعمل خادم BlueBubbles على أي Mac، ويمكن أن تعمل Gateway على Linux أو في أي مكان آخر. وإذا أردت أدوات أخرى خاصة بـ macOS، فشغّل Gateway على Mac أو أقرن macOS node.

    الوثائق: [BlueBubbles](/ar/channels/bluebubbles)، [Nodes](/ar/nodes)، [الوضع البعيد على Mac](/ar/platforms/mac/remote).

  </Accordion>

  <Accordion title="هل أحتاج إلى Mac mini لدعم iMessage؟">
    تحتاج إلى **جهاز macOS ما** مسجّل الدخول إلى Messages. وليس من الضروري أن يكون Mac mini -
    فأي Mac يكفي. **استخدم [BlueBubbles](/ar/channels/bluebubbles)** (مستحسن) لـ iMessage - يعمل خادم BlueBubbles على macOS، بينما يمكن لـ Gateway أن تعمل على Linux أو في مكان آخر.

    إعدادات شائعة:

    - شغّل Gateway على Linux/VPS، وشغّل خادم BlueBubbles على أي Mac مسجّل الدخول إلى Messages.
    - شغّل كل شيء على الـ Mac إذا كنت تريد أبسط إعداد أحادي الجهاز.

    الوثائق: [BlueBubbles](/ar/channels/bluebubbles)، [Nodes](/ar/nodes)،
    [الوضع البعيد على Mac](/ar/platforms/mac/remote).

  </Accordion>

  <Accordion title="إذا اشتريت Mac mini لتشغيل OpenClaw، هل يمكنني توصيله بـ MacBook Pro الخاص بي؟">
    نعم. يمكن لـ **Mac mini تشغيل Gateway**، ويمكن لـ MacBook Pro الخاص بك الاتصال به بوصفه
    **node** (جهازًا مرافقًا). لا تشغّل Nodes الـ Gateway - بل توفر
    قدرات إضافية مثل screen/camera/canvas و`system.run` على ذلك الجهاز.

    نمط شائع:

    - Gateway على Mac mini ‏(دائم التشغيل).
    - يشغّل MacBook Pro تطبيق macOS أو مضيف node ويقترن بـ Gateway.
    - استخدم `openclaw nodes status` / `openclaw nodes list` لرؤيته.

    الوثائق: [Nodes](/ar/nodes)، [Nodes CLI](/cli/nodes).

  </Accordion>

  <Accordion title="هل يمكنني استخدام Bun؟">
    Bun **غير موصى به**. نرى أخطاء وقت تشغيل، خاصة مع WhatsApp وTelegram.
    استخدم **Node** من أجل Gateways مستقرة.

    إذا كنت ما زلت تريد التجربة مع Bun، فافعل ذلك على Gateway غير إنتاجية
    ومن دون WhatsApp/Telegram.

  </Accordion>

  <Accordion title="Telegram: ماذا يوضع في allowFrom؟">
    إن `channels.telegram.allowFrom` هو **Telegram user ID للمرسل البشري** (رقمي). وليس اسم مستخدم bot.

    يقبل الإعداد الأولي إدخال `@username` ويحلّه إلى ID رقمي، لكن تفويض OpenClaw يستخدم IDs رقمية فقط.

    الطريقة الأكثر أمانًا (من دون bot تابع لجهة خارجية):

    - أرسل رسالة خاصة إلى bot لديك، ثم شغّل `openclaw logs --follow` واقرأ `from.id`.

    Official Bot API:

    - أرسل رسالة خاصة إلى bot لديك، ثم اتصل بـ `https://api.telegram.org/bot<bot_token>/getUpdates` واقرأ `message.from.id`.

    جهة خارجية (أقل خصوصية):

    - أرسل رسالة خاصة إلى `@userinfobot` أو `@getidsbot`.

    راجع [/channels/telegram](/ar/channels/telegram#access-control-and-activation).

  </Accordion>

  <Accordion title="هل يمكن لعدة أشخاص استخدام رقم WhatsApp واحد مع نُسخ OpenClaw مختلفة؟">
    نعم، عبر **التوجيه متعدد الوكلاء**. اربط WhatsApp **DM** لكل مرسل (peer من النوع `kind: "direct"`، مع E.164 للمرسل مثل `+15551234567`) بـ `agentId` مختلف، بحيث يحصل كل شخص على مساحة عمل ومخزن جلسات خاصين به. وستظل الردود تخرج من **الحساب نفسه على WhatsApp**، كما أن التحكم في الوصول إلى DM ‏(`channels.whatsapp.dmPolicy` / `channels.whatsapp.allowFrom`) عام على مستوى حساب WhatsApp نفسه. راجع [التوجيه متعدد الوكلاء](/ar/concepts/multi-agent) و[WhatsApp](/ar/channels/whatsapp).
  </Accordion>

  <Accordion title='هل يمكنني تشغيل وكيل "fast chat" ووكيل "Opus for coding"؟'>
    نعم. استخدم التوجيه متعدد الوكلاء: امنح كل وكيل نموذجه الافتراضي الخاص، ثم اربط المسارات الواردة (حساب الموفر أو peers محددين) بكل وكيل. يوجد مثال للإعدادات في [التوجيه متعدد الوكلاء](/ar/concepts/multi-agent). راجع أيضًا [النماذج](/ar/concepts/models) و[الإعدادات](/ar/gateway/configuration).
  </Accordion>

  <Accordion title="هل يعمل Homebrew على Linux؟">
    نعم. يدعم Homebrew نظام Linux ‏(Linuxbrew). إعداد سريع:

    ```bash
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
    echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"' >> ~/.profile
    eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
    brew install <formula>
    ```

    إذا كنت تشغّل OpenClaw عبر systemd، فتأكد من أن PATH الخاص بالخدمة يتضمن `/home/linuxbrew/.linuxbrew/bin` (أو brew prefix الخاص بك) حتى تُحل الأدوات المثبتة عبر `brew` في non-login shells.
    كما أن الإصدارات الحديثة تضيف أيضًا أدلة user bin الشائعة مسبقًا على خدمات Linux systemd (مثل `~/.local/bin` و`~/.npm-global/bin` و`~/.local/share/pnpm` و`~/.bun/bin`) وتحترم `PNPM_HOME` و`NPM_CONFIG_PREFIX` و`BUN_INSTALL` و`VOLTA_HOME` و`ASDF_DATA_DIR` و`NVM_DIR` و`FNM_DIR` عند تعيينها.

  </Accordion>

  <Accordion title="الفرق بين تثبيت git القابل للتعديل وnpm install">
    - **تثبيت hackable (git):** نسخة مصدر كاملة قابلة للتعديل، وهي الأفضل للمساهمين.
      يمكنك تشغيل البناء محليًا وتصحيح الكود/الوثائق.
    - **npm install:** تثبيت CLI عام، من دون مستودع، وهو الأفضل لمن يريد "تشغيله فقط".
      تأتي التحديثات من npm dist-tags.

    الوثائق: [البدء](/ar/start/getting-started)، [التحديث](/ar/install/updating).

  </Accordion>

  <Accordion title="هل يمكنني التبديل بين تثبيت npm وgit لاحقًا؟">
    نعم. ثبّت النكهة الأخرى، ثم شغّل Doctor حتى تشير خدمة gateway إلى entrypoint الجديد.
    هذا **لا يحذف بياناتك** - بل يغيّر فقط تثبيت كود OpenClaw. تظل الحالة
    (`~/.openclaw`) ومساحة العمل (`~/.openclaw/workspace`) كما هما.

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

    نصائح النسخ الاحتياطي: راجع [استراتيجية النسخ الاحتياطي](#أين-توجد-الأشياء-على-القرص).

  </Accordion>

  <Accordion title="هل يجب أن أشغّل Gateway على الكمبيوتر المحمول أم على VPS؟">
    الإجابة القصيرة: **إذا كنت تريد موثوقية 24/7 فاستخدم VPS**. أما إذا كنت تريد
    أقل قدر من الاحتكاك وكنت لا تمانع النوم/إعادة التشغيل، فشغّلها محليًا.

    **الكمبيوتر المحمول (Gateway محلية)**

    - **الإيجابيات:** لا توجد تكلفة خادم، وصول مباشر إلى الملفات المحلية، نافذة متصفح مرئية.
    - **السلبيات:** النوم/انقطاع الشبكة = انقطاعات، تحديثات/إعادات تشغيل نظام التشغيل تقطع العمل، ويجب أن يظل الجهاز مستيقظًا.

    **VPS / السحابة**

    - **الإيجابيات:** تشغيل دائم، شبكة مستقرة، لا مشاكل نوم الكمبيوتر المحمول، أسهل في الإبقاء عليها قيد التشغيل.
    - **السلبيات:** غالبًا تعمل headless ‏(استخدم لقطات الشاشة)، وصول إلى الملفات عن بُعد فقط، ويجب عليك استخدام SSH للتحديثات.

    **ملاحظة خاصة بـ OpenClaw:** كل من WhatsApp وTelegram وSlack وMattermost وDiscord تعمل جيدًا من VPS. والمقايضة الحقيقية الوحيدة هي **headless browser** مقابل نافذة مرئية. راجع [Browser](/ar/tools/browser).

    **الافتراضي الموصى به:** VPS إذا كانت لديك انقطاعات Gateway من قبل. أما التشغيل المحلي فهو رائع عندما تستخدم Mac بنشاط وتريد الوصول إلى الملفات المحلية أو أتمتة UI مع متصفح مرئي.

  </Accordion>

  <Accordion title="ما مدى أهمية تشغيل OpenClaw على جهاز مخصص؟">
    ليس مطلوبًا، لكنه **موصى به من أجل الموثوقية والعزل**.

    - **مضيف مخصص (VPS/Mac mini/Pi):** تشغيل دائم، انقطاعات أقل بسبب النوم/إعادة التشغيل، صلاحيات أنظف، وأسهل في الإبقاء عليه قيد التشغيل.
    - **كمبيوتر محمول/مكتبي مشترك:** مناسب تمامًا للاختبار والاستخدام النشط، لكن توقّع توقفات عند نوم الجهاز أو تحديثه.

    إذا كنت تريد أفضل ما في العالمين، فأبقِ Gateway على مضيف مخصص وأقرن الكمبيوتر المحمول بوصفه **node** لأدوات الشاشة/الكاميرا/exec المحلية. راجع [Nodes](/ar/nodes).
    ولإرشادات الأمان، اقرأ [الأمان](/ar/gateway/security).

  </Accordion>

  <Accordion title="ما الحد الأدنى لمتطلبات VPS ونظام التشغيل الموصى به؟">
    OpenClaw خفيف. بالنسبة إلى Gateway أساسية + قناة دردشة واحدة:

    - **الحد الأدنى المطلق:** 1 vCPU، و1GB RAM، وحوالي 500MB من القرص.
    - **الموصى به:** 1-2 vCPU، و2GB RAM أو أكثر من أجل الهامش (للسجلات، والوسائط، والقنوات المتعددة). يمكن أن تكون أدوات Node وأتمتة المتصفح شرهة للموارد.

    نظام التشغيل: استخدم **Ubuntu LTS** ‏(أو أي Debian/Ubuntu حديث). فهذا هو مسار تثبيت Linux الأكثر اختبارًا.

    الوثائق: [Linux](/ar/platforms/linux)، [استضافة VPS](/ar/vps).

  </Accordion>

  <Accordion title="هل يمكنني تشغيل OpenClaw داخل VM وما المتطلبات؟">
    نعم. تعامل مع VM كما تتعامل مع VPS: يجب أن تكون دائمًا قيد التشغيل، وقابلة للوصول، وتملك
    ما يكفي من RAM للـ Gateway وأي قنوات تفعّلها.

    الإرشادات الأساسية:

    - **الحد الأدنى المطلق:** 1 vCPU، و1GB RAM.
    - **الموصى به:** 2GB RAM أو أكثر إذا كنت تشغّل عدة قنوات أو أتمتة متصفح أو أدوات وسائط.
    - **نظام التشغيل:** Ubuntu LTS أو Debian/Ubuntu حديث آخر.

    إذا كنت على Windows، فإن **WSL2 هو أسهل إعداد بأسلوب VM** ويتمتع بأفضل توافق
    للأدوات. راجع [Windows](/ar/platforms/windows)، [استضافة VPS](/ar/vps).
    وإذا كنت تشغّل macOS داخل VM، فراجع [macOS VM](/ar/install/macos-vm).

  </Accordion>
</AccordionGroup>

## ما هو OpenClaw؟

<AccordionGroup>
  <Accordion title="ما هو OpenClaw في فقرة واحدة؟">
    OpenClaw هو مساعد AI شخصي تشغّله على أجهزتك الخاصة. يرد على أسطح المراسلة التي تستخدمها بالفعل (WhatsApp وTelegram وSlack وMattermost وDiscord وGoogle Chat وSignal وiMessage وWebChat وplugins القنوات المضمّنة مثل QQ Bot)، ويمكنه أيضًا التعامل مع الصوت + Canvas حي على المنصات المدعومة. إن **Gateway** هي مستوى التحكم الدائم التشغيل؛ أما المساعد فهو المنتج.
  </Accordion>

  <Accordion title="القيمة المقترحة">
    OpenClaw ليس "مجرد غلاف لـ Claude". بل هو **مستوى تحكم local-first** يتيح لك تشغيل
    مساعد قادر على **عتادك الخاص**، ويمكن الوصول إليه من تطبيقات الدردشة التي تستخدمها بالفعل، مع
    جلسات ذات حالة، وذاكرة، وأدوات - من دون تسليم التحكم في سير عملك إلى
    SaaS مستضاف.

    أبرز الميزات:

    - **أجهزتك، بياناتك:** شغّل Gateway حيثما تريد (Mac أو Linux أو VPS) واحتفظ بـ
      مساحة العمل + سجل الجلسات محليين.
    - **قنوات حقيقية، وليست sandbox على الويب:** WhatsApp/Telegram/Slack/Discord/Signal/iMessage/...إلخ،
      بالإضافة إلى الصوت المحمول وCanvas على المنصات المدعومة.
    - **غير مقيد بموفر واحد للنموذج:** استخدم Anthropic وOpenAI وMiniMax وOpenRouter وغيرها، مع توجيه
      وتحويل احتياطي لكل وكيل.
    - **خيار محلي فقط:** شغّل نماذج محلية حتى **تبقى كل البيانات على جهازك** إذا أردت.
    - **التوجيه متعدد الوكلاء:** وكلاء منفصلون لكل قناة أو حساب أو مهمة، ولكل منهم
      مساحة عمل وافتراضيات خاصة.
    - **مفتوح المصدر وقابل للتعديل:** افحصه ووسّعه واستضفه ذاتيًا من دون ارتهان لمورّد.

    الوثائق: [Gateway](/ar/gateway)، [القنوات](/ar/channels)، [الوكلاء المتعددون](/ar/concepts/multi-agent)،
    [الذاكرة](/ar/concepts/memory).

  </Accordion>

  <Accordion title="لقد أعددته للتو - ماذا أفعل أولاً؟">
    مشاريع أولى جيدة:

    - أنشئ موقعًا إلكترونيًا (WordPress أو Shopify أو موقعًا ثابتًا بسيطًا).
    - أنشئ نموذجًا أوليًا لتطبيق جوال (المخطط، والشاشات، وخطة API).
    - نظّم الملفات والمجلدات (التنظيف، والتسمية، والوسم).
    - اربط Gmail وأتمت الملخصات أو المتابعات.

    يمكنه التعامل مع مهام كبيرة، لكنه يعمل بأفضل شكل عندما تقسّمها إلى مراحل
    وتستخدم وكلاء فرعيين للعمل المتوازي.

  </Accordion>

  <Accordion title="ما أفضل خمس حالات استخدام يومية لـ OpenClaw؟">
    غالبًا ما تبدو المكاسب اليومية كالتالي:

    - **إحاطات شخصية:** ملخصات لصندوق الوارد، والتقويم، والأخبار التي تهمك.
    - **البحث والصياغة:** بحث سريع وملخصات ومسودات أولى للبريد الإلكتروني أو الوثائق.
    - **التذكيرات والمتابعات:** تنبيهات وقوائم تحقق مدفوعة بـ cron أو heartbeat.
    - **أتمتة المتصفح:** تعبئة النماذج، وجمع البيانات، وتكرار مهام الويب.
    - **التنسيق بين الأجهزة:** أرسل مهمة من هاتفك، ودع Gateway تنفذها على خادم، واستلم النتيجة مرة أخرى في الدردشة.

  </Accordion>

  <Accordion title="هل يمكن لـ OpenClaw المساعدة في توليد العملاء المحتملين، والتواصل، والإعلانات، والمدونات لـ SaaS؟">
    نعم فيما يتعلق بـ **البحث، والتأهيل، والصياغة**. يمكنه فحص المواقع، وبناء قوائم مختصرة،
    وتلخيص العملاء المحتملين، وكتابة مسودات التواصل أو نسخ الإعلانات.

    أما بالنسبة إلى **التواصل أو تشغيل الإعلانات**، فأبقِ إنسانًا ضمن الحلقة. تجنب
    الرسائل المزعجة، واتبع القوانين المحلية وسياسات المنصات، وراجع أي شيء قبل إرساله.
    النمط الأكثر أمانًا هو أن يكتب OpenClaw المسودة وأنت توافق عليها.

    الوثائق: [الأمان](/ar/gateway/security).

  </Accordion>

  <Accordion title="ما مزايا هذا مقابل Claude Code لتطوير الويب؟">
    OpenClaw هو **مساعد شخصي** وطبقة تنسيق، وليس بديلًا عن IDE. استخدم
    Claude Code أو Codex لأسرع حلقة ترميز مباشرة داخل مستودع. واستخدم OpenClaw عندما
    تريد ذاكرة دائمة، ووصولًا عبر الأجهزة، وتنسيق الأدوات.

    المزايا:

    - **ذاكرة + مساحة عمل دائمتان** عبر الجلسات
    - **وصول متعدد المنصات** ‏(WhatsApp وTelegram وTUI وWebChat)
    - **تنسيق الأدوات** ‏(المتصفح، والملفات، والجدولة، وhooks)
    - **Gateway دائمة التشغيل** ‏(شغّلها على VPS، وتفاعل معها من أي مكان)
    - **Nodes** للمتصفح/الشاشة/الكاميرا/exec المحلي

    عرض توضيحي: [https://openclaw.ai/showcase](https://openclaw.ai/showcase)

  </Accordion>
</AccordionGroup>

## Skills والأتمتة

<AccordionGroup>
  <Accordion title="كيف أخصص Skills من دون إبقاء المستودع متسخًا؟">
    استخدم managed overrides بدلًا من تعديل نسخة المستودع. ضع تغييراتك في `~/.openclaw/skills/<name>/SKILL.md` (أو أضف مجلدًا عبر `skills.load.extraDirs` في `~/.openclaw/openclaw.json`). ترتيب الأولوية هو `<workspace>/skills` ← `<workspace>/.agents/skills` ← `~/.agents/skills` ← `~/.openclaw/skills` ← المضمّن ← `skills.load.extraDirs`، لذلك ما زالت managed overrides تتغلب على Skills المضمّنة من دون المساس بـ git. وإذا كنت تحتاج إلى تثبيت المهارة عالميًا لكن تريد أن تكون مرئية لبعض الوكلاء فقط، فاحتفظ بالنسخة المشتركة في `~/.openclaw/skills` وتحكم في ظهورها عبر `agents.defaults.skills` و`agents.list[].skills`. ولا ينبغي أن تعيش في المستودع ولا تُرسل على شكل PRs إلا التعديلات الجديرة بالرفع للمصدر.
  </Accordion>

  <Accordion title="هل يمكنني تحميل Skills من مجلد مخصص؟">
    نعم. أضف أدلة إضافية عبر `skills.load.extraDirs` في `~/.openclaw/openclaw.json` (أدنى أولوية). ترتيب الأولوية الافتراضي هو `<workspace>/skills` ← `<workspace>/.agents/skills` ← `~/.agents/skills` ← `~/.openclaw/skills` ← المضمّن ← `skills.load.extraDirs`. يثبّت `clawhub` في `./skills` افتراضيًا، ويتعامل OpenClaw معه بوصفه `<workspace>/skills` في الجلسة التالية. وإذا كان يجب أن تكون المهارة مرئية فقط لوكلاء معيّنين، فاقرن ذلك مع `agents.defaults.skills` أو `agents.list[].skills`.
  </Accordion>

  <Accordion title="كيف يمكنني استخدام نماذج مختلفة لمهام مختلفة؟">
    الأنماط المدعومة اليوم هي:

    - **وظائف Cron**: يمكن للوظائف المعزولة تعيين تجاوز `model` لكل وظيفة.
    - **الوكلاء الفرعيون**: وجّه المهام إلى وكلاء منفصلين لديهم نماذج افتراضية مختلفة.
    - **التبديل عند الطلب**: استخدم `/model` لتبديل نموذج الجلسة الحالية في أي وقت.

    راجع [وظائف Cron](/ar/automation/cron-jobs)، و[التوجيه متعدد الوكلاء](/ar/concepts/multi-agent)، و[Slash commands](/ar/tools/slash-commands).

  </Accordion>

  <Accordion title="يتجمد bot أثناء العمل الثقيل. كيف أنقل هذا الحمل؟">
    استخدم **الوكلاء الفرعيين** للمهام الطويلة أو المتوازية. تعمل الوكلاء الفرعيون في جلساتهم الخاصة،
    وتُرجع ملخصًا، وتحافظ على استجابة الدردشة الرئيسية.

    اطلب من bot أن "ينشئ وكيلًا فرعيًا لهذه المهمة" أو استخدم `/subagents`.
    استخدم `/status` في الدردشة لمعرفة ما الذي تفعله Gateway الآن (وما إذا كانت مشغولة).

    نصيحة حول tokens: كل من المهام الطويلة والوكلاء الفرعيين يستهلك tokens. وإذا كانت التكلفة مصدر قلق، فاضبط
    نموذجًا أرخص للوكلاء الفرعيين عبر `agents.defaults.subagents.model`.

    الوثائق: [الوكلاء الفرعيون](/ar/tools/subagents)، [المهام الخلفية](/ar/automation/tasks).

  </Accordion>

  <Accordion title="كيف تعمل جلسات الوكلاء الفرعيين المرتبطة بالسلاسل على Discord؟">
    استخدم thread bindings. يمكنك ربط thread على Discord بهدف وكيل فرعي أو جلسة بحيث تبقى رسائل المتابعة في ذلك thread على الجلسة المرتبطة.

    التدفق الأساسي:

    - أنشئ باستخدام `sessions_spawn` مع `thread: true` (واختياريًا `mode: "session"` من أجل متابعات دائمة).
    - أو اربط يدويًا عبر `/focus <target>`.
    - استخدم `/agents` لفحص حالة الربط.
    - استخدم `/session idle <duration|off>` و`/session max-age <duration|off>` للتحكم في إلغاء التركيز التلقائي.
    - استخدم `/unfocus` لفصل thread.

    الإعداد المطلوب:

    - الافتراضيات العامة: ‏`session.threadBindings.enabled` و`session.threadBindings.idleHours` و`session.threadBindings.maxAgeHours`.
    - تجاوزات Discord: ‏`channels.discord.threadBindings.enabled` و`channels.discord.threadBindings.idleHours` و`channels.discord.threadBindings.maxAgeHours`.
    - الربط التلقائي عند الإنشاء: اضبط `channels.discord.threadBindings.spawnSubagentSessions: true`.

    الوثائق: [الوكلاء الفرعيون](/ar/tools/subagents)، [Discord](/ar/channels/discord)، [مرجع الإعدادات](/ar/gateway/configuration-reference)، [Slash commands](/ar/tools/slash-commands).

  </Accordion>

  <Accordion title="انتهى وكيل فرعي، لكن تحديث الإكمال ذهب إلى المكان الخطأ أو لم يُنشر أبدًا. ما الذي يجب أن أتحقق منه؟">
    تحقق أولًا من requester route المحلولة:

    - يفضّل تسليم الوكيل الفرعي في وضع الإكمال أي thread أو مسار محادثة مرتبط عندما يكون موجودًا.
    - إذا كان أصل الإكمال لا يحمل إلا قناة، فيعود OpenClaw إلى المسار المخزّن لجلسة الطالب (`lastChannel` / `lastTo` / `lastAccountId`) بحيث يظل التسليم المباشر ممكنًا.
    - إذا لم يوجد لا مسار مرتبط ولا مسار مخزّن صالح، فقد يفشل التسليم المباشر وتعود النتيجة إلى التسليم المؤجل للجلسة بدلًا من النشر الفوري إلى الدردشة.
    - لا تزال الأهداف غير الصالحة أو القديمة قادرة على فرض الرجوع إلى queue fallback أو فشل التسليم النهائي.
    - إذا كان آخر رد مساعد مرئي من الابن هو silent token الدقيق `NO_REPLY` / `no_reply`، أو كان يساوي تمامًا `ANNOUNCE_SKIP`، فإن OpenClaw يقمع الإعلان عمدًا بدلًا من نشر تقدم سابق قديم.
    - إذا انتهت مهلة الابن بعد مجرد استدعاءات أدوات، فقد يختزل الإعلان ذلك إلى ملخص قصير للتقدم الجزئي بدلًا من إعادة تشغيل مخرجات الأدوات الخام.

    التصحيح:

    ```bash
    openclaw tasks show <runId-or-sessionKey>
    ```

    الوثائق: [الوكلاء الفرعيون](/ar/tools/subagents)، [المهام الخلفية](/ar/automation/tasks)، [أداة الجلسات](/ar/concepts/session-tool).

  </Accordion>

  <Accordion title="Cron أو التذكيرات لا تعمل. ما الذي يجب أن أتحقق منه؟">
    تعمل Cron داخل عملية Gateway. وإذا لم تكن Gateway تعمل باستمرار،
    فلن تعمل الوظائف المجدولة.

    قائمة تحقق:

    - تأكد من أن cron مفعّلة (`cron.enabled`) وأن `OPENCLAW_SKIP_CRON` غير مضبوط.
    - تحقق من أن Gateway تعمل 24/7 (من دون نوم/إعادة تشغيل).
    - تحقّق من إعدادات المنطقة الزمنية للوظيفة (`--tz` مقابل المنطقة الزمنية للمضيف).

    التصحيح:

    ```bash
    openclaw cron run <jobId>
    openclaw cron runs --id <jobId> --limit 50
    ```

    الوثائق: [وظائف Cron](/ar/automation/cron-jobs)، [الأتمتة والمهام](/ar/automation).

  </Accordion>

  <Accordion title="تم تشغيل Cron، لكن لم يُرسل شيء إلى القناة. لماذا؟">
    تحقق أولًا من وضع التسليم:

    - `--no-deliver` / `delivery.mode: "none"` يعني أنه لا يُتوقع أي رسالة خارجية.
    - الهدف المفقود أو غير الصالح للإعلان (`channel` / `to`) يعني أن المشغّل تخطى التسليم الصادر.
    - تعني أخطاء مصادقة القناة (`unauthorized` و`Forbidden`) أن المشغّل حاول التسليم لكن بيانات الاعتماد منعته.
    - تُعامل النتيجة المعزولة الصامتة (`NO_REPLY` / `no_reply` فقط) على أنها غير قابلة للتسليم عمدًا، لذلك يقمع المشغّل أيضًا queued fallback delivery.

    بالنسبة إلى وظائف cron المعزولة، يملك المشغّل مسؤولية التسليم النهائي. ومن المتوقع
    أن يعيد الوكيل ملخصًا نصيًا عاديًا ليقوم المشغّل بإرساله. أما `--no-deliver` فيُبقي
    النتيجة داخلية؛ ولا يسمح للوكيل بالإرسال مباشرة بواسطة
    أداة الرسائل بدلًا من ذلك.

    التصحيح:

    ```bash
    openclaw cron runs --id <jobId> --limit 50
    openclaw tasks show <runId-or-sessionKey>
    ```

    الوثائق: [وظائف Cron](/ar/automation/cron-jobs)، [المهام الخلفية](/ar/automation/tasks).

  </Accordion>

  <Accordion title="لماذا بدّلت عملية cron المعزولة النماذج أو أعادت المحاولة مرة واحدة؟">
    يكون هذا عادةً مسار تبديل النموذج الحي، وليس جدولة مكررة.

    يمكن لـ cron المعزولة أن تحفظ handoff لنموذج وقت التشغيل وتعيد المحاولة عندما يرمي
    التشغيل النشط `LiveSessionModelSwitchError`. وتحافظ إعادة المحاولة على
    الموفر/النموذج المُبدَّل، وإذا حمل التبديل تجاوزًا جديدًا لملف تعريف المصادقة، فإن cron
    تحفظ ذلك أيضًا قبل إعادة المحاولة.

    قواعد الاختيار ذات الصلة:

    - يفوز أولًا تجاوز نموذج Gmail hook عند انطباقه.
    - ثم `model` لكل وظيفة.
    - ثم أي تجاوز محفوظ لنموذج جلسة cron.
    - ثم الاختيار العادي للنموذج الافتراضي/نموذج الوكيل.

    حلقة إعادة المحاولة محدودة. بعد المحاولة الأولية + محاولتي تبديل،
    تُجهض cron بدلًا من الاستمرار إلى ما لا نهاية.

    التصحيح:

    ```bash
    openclaw cron runs --id <jobId> --limit 50
    openclaw tasks show <runId-or-sessionKey>
    ```

    الوثائق: [وظائف Cron](/ar/automation/cron-jobs)، [cron CLI](/cli/cron).

  </Accordion>

  <Accordion title="كيف أثبت Skills على Linux؟">
    استخدم أوامر `openclaw skills` الأصلية أو ضع Skills داخل مساحة عملك. واجهة Skills على macOS غير متاحة على Linux.
    تصفّح Skills على [https://clawhub.ai](https://clawhub.ai).

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

    تقوم عملية `openclaw skills install` الأصلية بالكتابة إلى دليل `skills/`
    داخل مساحة العمل النشطة. ثبّت CLI المنفصل `clawhub` فقط إذا أردت نشر أو
    مزامنة Skills الخاصة بك. وللتثبيتات المشتركة بين الوكلاء، ضع المهارة تحت
    `~/.openclaw/skills` واستخدم `agents.defaults.skills` أو
    `agents.list[].skills` إذا أردت تضييق الوكلاء الذين يمكنهم رؤيتها.

  </Accordion>

  <Accordion title="هل يمكن لـ OpenClaw تشغيل مهام وفق جدول أو باستمرار في الخلفية؟">
    نعم. استخدم مجدول Gateway:

    - **وظائف Cron** للمهام المجدولة أو المتكررة (تستمر عبر إعادة التشغيل).
    - **Heartbeat** للتحققات الدورية الخاصة بـ "الجلسة الرئيسية".
    - **وظائف معزولة** لوكلاء مستقلين ينشرون ملخصات أو يسلّمون إلى الدردشات.

    الوثائق: [وظائف Cron](/ar/automation/cron-jobs)، [الأتمتة والمهام](/ar/automation)،
    [Heartbeat](/ar/gateway/heartbeat).

  </Accordion>

  <Accordion title="هل يمكنني تشغيل Skills Apple الخاصة بـ macOS فقط من Linux؟">
    ليس بشكل مباشر. تُقيَّد Skills الخاصة بـ macOS بواسطة `metadata.openclaw.os` مع الثنائيات المطلوبة، ولا تظهر Skills في system prompt إلا عندما تكون مؤهلة على **مضيف Gateway**. على Linux، لن تُحمَّل Skills الخاصة بـ `darwin` (مثل `apple-notes` و`apple-reminders` و`things-mac`) ما لم تتجاوز هذا التقييد.

    لديك ثلاثة أنماط مدعومة:

    **الخيار A - شغّل Gateway على Mac (الأبسط).**
    شغّل Gateway حيث توجد الثنائيات الخاصة بـ macOS، ثم اتصل من Linux في [الوضع البعيد](#gateway-المنافذ-قيد-التشغيل-مسبقًا-والوضع-البعيد) أو عبر Tailscale. تُحمَّل Skills بشكل طبيعي لأن مضيف Gateway يعمل على macOS.

    **الخيار B - استخدم macOS node (من دون SSH).**
    شغّل Gateway على Linux، وأقرِن macOS node ‏(تطبيق شريط القوائم)، واضبط **Node Run Commands** على "Always Ask" أو "Always Allow" على الـ Mac. يمكن لـ OpenClaw اعتبار Skills الخاصة بـ macOS مؤهلة عندما تكون الثنائيات المطلوبة موجودة على node. ويشغّل الوكيل تلك Skills عبر أداة `nodes`. وإذا اخترت "Always Ask"، فإن الموافقة على "Always Allow" في المطالبة تضيف ذلك الأمر إلى allowlist.

    **الخيار C - مرّر ثنائيات macOS عبر SSH (متقدم).**
    أبقِ Gateway على Linux، ولكن اجعل الثنائيات المطلوبة في CLI تُحل إلى أغلفة SSH تعمل على Mac. ثم تجاوز Skill بحيث تسمح بـ Linux حتى تظل مؤهلة.

    1. أنشئ غلاف SSH للثنائي (مثال: `memo` من أجل Apple Notes):

       ```bash
       #!/usr/bin/env bash
       set -euo pipefail
       exec ssh -T user@mac-host /opt/homebrew/bin/memo "$@"
       ```

    2. ضع الغلاف على `PATH` على مضيف Linux ‏(مثل `~/bin/memo`).
    3. تجاوز metadata الخاصة بالمهارة (في مساحة العمل أو `~/.openclaw/skills`) للسماح بـ Linux:

       ```markdown
       ---
       name: apple-notes
       description: Manage Apple Notes via the memo CLI on macOS.
       metadata: { "openclaw": { "os": ["darwin", "linux"], "requires": { "bins": ["memo"] } } }
       ---
       ```

    4. ابدأ جلسة جديدة حتى يتم تحديث لقطة Skills.

  </Accordion>

  <Accordion title="هل لديكم تكامل مع Notion أو HeyGen؟">
    ليس مضمّنًا اليوم.

    الخيارات:

    - **Skill / plugin مخصصة:** الأفضل للوصول الموثوق إلى API ‏(لكل من Notion وHeyGen واجهات API).
    - **أتمتة المتصفح:** تعمل من دون كود لكنها أبطأ وأكثر هشاشة.

    إذا أردت الاحتفاظ بالسياق لكل عميل (سير عمل الوكالات)، فهناك نمط بسيط:

    - صفحة Notion واحدة لكل عميل (السياق + التفضيلات + العمل النشط).
    - اطلب من الوكيل جلب تلك الصفحة في بداية الجلسة.

    وإذا أردت تكاملًا أصليًا، فافتح طلب ميزة أو أنشئ Skill
    تستهدف تلك APIs.

    تثبيت Skills:

    ```bash
    openclaw skills install <skill-slug>
    openclaw skills update --all
    ```

    تذهب التثبيتات الأصلية إلى دليل `skills/` في مساحة العمل النشطة. وللمهارات المشتركة بين الوكلاء، ضعها في `~/.openclaw/skills/<name>/SKILL.md`. وإذا كان يجب أن يراها بعض الوكلاء فقط، فكوّن `agents.defaults.skills` أو `agents.list[].skills`. وتتوقع بعض Skills وجود ثنائيات مثبتة عبر Homebrew؛ وعلى Linux فهذا يعني Linuxbrew (راجع عنصر الأسئلة الشائعة الخاص بـ Homebrew على Linux أعلاه). راجع [Skills](/ar/tools/skills)، و[إعداد Skills](/ar/tools/skills-config)، و[ClawHub](/ar/tools/clawhub).

  </Accordion>

  <Accordion title="كيف أستخدم Chrome الذي أنا مسجل الدخول إليه بالفعل مع OpenClaw؟">
    استخدم ملف browser profile المضمّن `user`، والذي يتصل عبر Chrome DevTools MCP:

    ```bash
    openclaw browser --browser-profile user tabs
    openclaw browser --browser-profile user snapshot
    ```

    وإذا أردت اسمًا مخصصًا، فأنشئ MCP profile صريحًا:

    ```bash
    openclaw browser create-profile --name chrome-live --driver existing-session
    openclaw browser --browser-profile chrome-live tabs
    ```

    هذا المسار محلي على المضيف. وإذا كانت Gateway تعمل في مكان آخر، فإما أن تشغّل node host على الجهاز الذي عليه المتصفح، أو تستخدم CDP عن بُعد بدلًا من ذلك.

    الحدود الحالية لـ `existing-session` / `user`:

    - الإجراءات تعتمد على ref، وليس على CSS-selector
    - تتطلب الرفع قيم `ref` / `inputRef` وتدعم حاليًا ملفًا واحدًا في كل مرة
    - ما زالت `responsebody` وتصدير PDF واعتراض التنزيلات والإجراءات الدُفعية تحتاج إلى managed browser أو raw CDP profile

  </Accordion>
</AccordionGroup>

## Sandboxing والذاكرة

<AccordionGroup>
  <Accordion title="هل توجد وثيقة مخصصة لـ sandboxing؟">
    نعم. راجع [Sandboxing](/ar/gateway/sandboxing). ولإعدادات Docker الخاصة (Gateway كاملة داخل Docker أو صور sandbox)، راجع [Docker](/ar/install/docker).
  </Accordion>

  <Accordion title="يبدو Docker محدودًا - كيف أفعّل الميزات الكاملة؟">
    الصورة الافتراضية تعطي الأولوية للأمان وتعمل كمستخدم `node`، ولذلك فهي
    لا تتضمن حزم النظام أو Homebrew أو المتصفحات المضمنة. ولإعداد أكمل:

    - اجعل `/home/node` دائمًا باستخدام `OPENCLAW_HOME_VOLUME` حتى تبقى الذاكرات المؤقتة.
    - اخبز تبعيات النظام داخل الصورة باستخدام `OPENCLAW_DOCKER_APT_PACKAGES`.
    - ثبّت متصفحات Playwright عبر CLI المضمّن:
      `node /app/node_modules/playwright-core/cli.js install chromium`
    - اضبط `PLAYWRIGHT_BROWSERS_PATH` وتأكد من أن المسار محفوظ بشكل دائم.

    الوثائق: [Docker](/ar/install/docker)، [Browser](/ar/tools/browser).

  </Accordion>

  <Accordion title="هل يمكنني إبقاء الرسائل الخاصة شخصية ولكن جعل المجموعات عامة/ضمن sandbox باستخدام وكيل واحد؟">
    نعم - إذا كانت حركة المرور الخاصة لديك هي **الرسائل الخاصة** وكانت الحركة العامة هي **المجموعات**.

    استخدم `agents.defaults.sandbox.mode: "non-main"` حتى تعمل جلسات المجموعات/القنوات (المفاتيح غير الرئيسية) داخل Docker، بينما تبقى جلسة الرسائل الخاصة الرئيسية على المضيف. ثم قيّد الأدوات المتاحة في الجلسات المعزولة عبر `tools.sandbox.tools`.

    شرح الإعداد + مثال للإعدادات: [المجموعات: الرسائل الخاصة الشخصية + المجموعات العامة](/ar/channels/groups#pattern-personal-dms-public-groups-single-agent)

    مرجع الإعدادات الأساسي: [إعدادات Gateway](/ar/gateway/configuration-reference#agentsdefaultssandbox)

  </Accordion>

  <Accordion title="كيف أربط مجلدًا من المضيف داخل sandbox؟">
    اضبط `agents.defaults.sandbox.docker.binds` على `["host:path:mode"]` (مثل `"/home/user/src:/src:ro"`). يتم دمج bindات المستوى العام + لكل وكيل؛ ويتم تجاهل bindات كل وكيل عند `scope: "shared"`. استخدم `:ro` لأي شيء حساس، وتذكّر أن bindات تتجاوز جدران نظام ملفات sandbox.

    يتحقق OpenClaw من مصادر bind مقابل كل من المسار المُطبَّع والمسار القانوني الذي يُحل عبر أعمق سلف موجود. وهذا يعني أن محاولات الهروب عبر أصل symlink لا تزال تفشل بشكل آمن حتى عندما لا يكون مقطع المسار الأخير موجودًا بعد، كما أن فحوصات allowed-root تظل مطبقة بعد حل symlink.

    راجع [Sandboxing](/ar/gateway/sandboxing#custom-bind-mounts) و[Sandbox مقابل Tool Policy مقابل Elevated](/ar/gateway/sandbox-vs-tool-policy-vs-elevated#bind-mounts-security-quick-check) للاطلاع على أمثلة وملاحظات الأمان.

  </Accordion>

  <Accordion title="كيف تعمل الذاكرة؟">
    ذاكرة OpenClaw ليست سوى ملفات Markdown داخل مساحة عمل الوكيل:

    - ملاحظات يومية في `memory/YYYY-MM-DD.md`
    - ملاحظات طويلة الأمد مُنتقاة في `MEMORY.md` (للجلسات الرئيسية/الخاصة فقط)

    كما يشغّل OpenClaw **تفريغ ذاكرة صامت قبل الضغط** لتذكير النموذج
    بكتابة ملاحظات دائمة قبل الضغط التلقائي. ولا يعمل هذا إلا عندما تكون مساحة العمل
    قابلة للكتابة (تتخطاه sandboxes للقراءة فقط). راجع [الذاكرة](/ar/concepts/memory).

  </Accordion>

  <Accordion title="تستمر الذاكرة في نسيان الأشياء. كيف أجعلها ثابتة؟">
    اطلب من bot أن **يكتب الحقيقة في الذاكرة**. تنتمي الملاحظات طويلة الأمد إلى `MEMORY.md`،
    بينما يذهب السياق قصير الأجل إلى `memory/YYYY-MM-DD.md`.

    ما زال هذا مجالًا نعمل على تحسينه. ويساعد أن تذكّر النموذج بتخزين الذكريات؛
    فهو سيعرف ما الذي يجب فعله. وإذا استمر في النسيان، فتحقق من أن Gateway تستخدم نفس
    مساحة العمل في كل مرة تعمل فيها.

    الوثائق: [الذاكرة](/ar/concepts/memory)، [مساحة عمل الوكيل](/ar/concepts/agent-workspace).

  </Accordion>

  <Accordion title="هل تستمر الذاكرة إلى الأبد؟ ما الحدود؟">
    تعيش ملفات الذاكرة على القرص وتستمر حتى تحذفها. الحد هو
    التخزين لديك، وليس النموذج. أما **سياق الجلسة** فما زال محدودًا بنافذة سياق
    النموذج، لذلك يمكن للمحادثات الطويلة أن تُضغط أو تُقتطع. ولهذا
    يوجد البحث في الذاكرة - فهو يعيد فقط الأجزاء ذات الصلة إلى السياق.

    الوثائق: [الذاكرة](/ar/concepts/memory)، [السياق](/ar/concepts/context).

  </Accordion>

  <Accordion title="هل يتطلب البحث الدلالي في الذاكرة مفتاح OpenAI API؟">
    فقط إذا كنت تستخدم **OpenAI embeddings**. يغطي Codex OAuth الدردشة/الإكمالات و
    **لا** يمنح وصولًا إلى embeddings، لذلك فإن **تسجيل الدخول باستخدام Codex (OAuth أو
    تسجيل دخول Codex CLI)** لا يساعد في البحث الدلالي في الذاكرة. ولا تزال OpenAI embeddings
    تحتاج إلى مفتاح API حقيقي (`OPENAI_API_KEY` أو `models.providers.openai.apiKey`).

    إذا لم تضبط موفرًا صراحةً، يختار OpenClaw موفرًا تلقائيًا عندما يكون
    قادرًا على حل مفتاح API ‏(ملفات تعريف المصادقة، أو `models.providers.*.apiKey`، أو متغيرات env).
    وهو يفضّل OpenAI إذا أمكن حل مفتاح OpenAI، وإلا Gemini إذا أمكن حل مفتاح Gemini،
    ثم Voyage، ثم Mistral. وإذا لم يتوفر مفتاح بعيد، يظل البحث في الذاكرة
    معطّلًا إلى أن تهيئه. وإذا كان لديك مسار نموذج محلي
    مضبوط وموجود، فإن OpenClaw
    يفضّل `local`. كما أن Ollama مدعوم عند ضبط
    `memorySearch.provider = "ollama"` صراحةً.

    وإذا كنت تفضّل البقاء محليًا، فاضبط `memorySearch.provider = "local"` (واختياريًا
    `memorySearch.fallback = "none"`). وإذا أردت Gemini embeddings، فاضبط
    `memorySearch.provider = "gemini"` ووفّر `GEMINI_API_KEY` (أو
    `memorySearch.remote.apiKey`). نحن ندعم نماذج embeddings من **OpenAI وGemini وVoyage وMistral وOllama أو local**
    - راجع [الذاكرة](/ar/concepts/memory) للحصول على تفاصيل الإعداد.

  </Accordion>
</AccordionGroup>

## أين توجد الأشياء على القرص

<AccordionGroup>
  <Accordion title="هل تُحفظ كل البيانات المستخدمة مع OpenClaw محليًا؟">
    لا - **حالة OpenClaw محلية**، لكن **الخدمات الخارجية لا تزال ترى ما ترسله إليها**.

    - **محلية افتراضيًا:** تعيش الجلسات وملفات الذاكرة والإعدادات ومساحة العمل على مضيف Gateway
      (`~/.openclaw` + دليل مساحة العمل الخاص بك).
    - **بعيدة بحكم الضرورة:** الرسائل التي ترسلها إلى موفري النماذج (Anthropic/OpenAI/إلخ) تذهب إلى
      APIs الخاصة بهم، كما أن منصات الدردشة (WhatsApp/Telegram/Slack/إلخ) تخزن بيانات الرسائل على
      خوادمها.
    - **أنت تتحكم في البصمة:** يؤدي استخدام النماذج المحلية إلى إبقاء prompts على جهازك، لكن
      حركة مرور القنوات لا تزال تمر عبر خوادم القناة.

    ذو صلة: [مساحة عمل الوكيل](/ar/concepts/agent-workspace)، [الذاكرة](/ar/concepts/memory).

  </Accordion>

  <Accordion title="أين يخزن OpenClaw بياناته؟">
    يعيش كل شيء تحت `$OPENCLAW_STATE_DIR` (الافتراضي: `~/.openclaw`):

    | المسار | الغرض |
    | --------------------------------------------------------------- | ------------------------------------------------------------------ |
    | `$OPENCLAW_STATE_DIR/openclaw.json` | الإعدادات الرئيسية (JSON5) |
    | `$OPENCLAW_STATE_DIR/credentials/oauth.json` | استيراد OAuth قديم (يُنسخ إلى ملفات تعريف المصادقة عند أول استخدام) |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/auth-profiles.json` | ملفات تعريف المصادقة (OAuth، ومفاتيح API، وخيار `keyRef`/`tokenRef`) |
    | `$OPENCLAW_STATE_DIR/secrets.json` | حمولة سرية اختيارية مدعومة بملف لموفري SecretRef من نوع `file` |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/auth.json` | ملف توافق قديم (مع تنظيف إدخالات `api_key` الثابتة) |
    | `$OPENCLAW_STATE_DIR/credentials/` | حالة الموفّر (مثل `whatsapp/<accountId>/creds.json`) |
    | `$OPENCLAW_STATE_DIR/agents/` | حالة لكل وكيل (agentDir + sessions) |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/sessions/` | سجل المحادثة والحالة (لكل وكيل) |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/sessions/sessions.json` | بيانات الجلسة الوصفية (لكل وكيل) |

    المسار القديم لوكيل واحد: `~/.openclaw/agent/*` (يتم ترحيله عبر `openclaw doctor`).

    أما **مساحة العمل** الخاصة بك (AGENTS.md، وملفات الذاكرة، وSkills، إلخ) فهي منفصلة ويتم تكوينها عبر `agents.defaults.workspace` (الافتراضي: `~/.openclaw/workspace`).

  </Accordion>

  <Accordion title="أين يجب أن تعيش AGENTS.md / SOUL.md / USER.md / MEMORY.md؟">
    تعيش هذه الملفات في **مساحة عمل الوكيل**، وليس في `~/.openclaw`.

    - **مساحة العمل (لكل وكيل)**: ‏`AGENTS.md` و`SOUL.md` و`IDENTITY.md` و`USER.md`،
      و`MEMORY.md` (أو fallback القديم `memory.md` عند غياب `MEMORY.md`)،
      و`memory/YYYY-MM-DD.md`، و`HEARTBEAT.md` اختياري.
    - **دليل الحالة (`~/.openclaw`)**: الإعدادات، وحالة القناة/الموفر، وملفات تعريف المصادقة، والجلسات، والسجلات،
      وSkills المشتركة (`~/.openclaw/skills`).

    مساحة العمل الافتراضية هي `~/.openclaw/workspace`، ويمكن تكوينها عبر:

    ```json5
    {
      agents: { defaults: { workspace: "~/.openclaw/workspace" } },
    }
    ```

    إذا كان bot "ينسى" بعد إعادة التشغيل، فتأكد من أن Gateway تستخدم نفس
    مساحة العمل في كل تشغيل (وتذكّر: يستخدم الوضع البعيد مساحة العمل الخاصة بـ **مضيف gateway**
    وليس الكمبيوتر المحمول المحلي لديك).

    نصيحة: إذا كنت تريد سلوكًا أو تفضيلًا دائمًا، فاطلب من bot أن **يكتبه في
    AGENTS.md أو MEMORY.md** بدلًا من الاعتماد على سجل الدردشة.

    راجع [مساحة عمل الوكيل](/ar/concepts/agent-workspace) و[الذاكرة](/ar/concepts/memory).

  </Accordion>

  <Accordion title="استراتيجية النسخ الاحتياطي الموصى بها">
    ضع **مساحة عمل الوكيل** الخاصة بك في مستودع git **خاص** واحتفظ بنسخة احتياطية منها في مكان
    خاص (مثل GitHub الخاص). هذا يلتقط الذاكرة + ملفات AGENTS/SOUL/USER
    ويتيح لك استعادة "عقل" المساعد لاحقًا.

    **لا** تقم بعمل commit لأي شيء تحت `~/.openclaw` (بيانات الاعتماد، أو الجلسات، أو tokens، أو حمولات الأسرار المشفرة).
    وإذا كنت تحتاج إلى استعادة كاملة، فاحتفظ بنسخة احتياطية من كل من مساحة العمل ودليل الحالة
    بشكل منفصل (راجع سؤال الترحيل أعلاه).

    الوثائق: [مساحة عمل الوكيل](/ar/concepts/agent-workspace).

  </Accordion>

  <Accordion title="كيف أزيل OpenClaw بالكامل؟">
    راجع الدليل المخصص: [إلغاء التثبيت](/ar/install/uninstall).
  </Accordion>

  <Accordion title="هل يمكن للوكلاء العمل خارج مساحة العمل؟">
    نعم. مساحة العمل هي **cwd افتراضي** ومرساة للذاكرة، وليست sandbox صلبة.
    تُحل المسارات النسبية داخل مساحة العمل، لكن المسارات المطلقة يمكنها الوصول إلى
    مواقع أخرى على المضيف ما لم تكن sandboxing مفعلة. وإذا كنت تحتاج إلى عزل، فاستخدم
    [`agents.defaults.sandbox`](/ar/gateway/sandboxing) أو إعدادات sandbox لكل وكيل. وإذا كنت
    تريد أن يكون المستودع هو دليل العمل الافتراضي، فاجعل `workspace`
    الخاصة بذلك الوكيل تشير إلى جذر المستودع. إن مستودع OpenClaw مجرد كود مصدر؛ أبقِ
    مساحة العمل منفصلة ما لم تكن تريد عمدًا أن يعمل الوكيل داخله.

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
    تملك **مضيف gateway** حالة الجلسة. وإذا كنت في الوضع البعيد، فإن مخزن الجلسات الذي يهمك يوجد على الجهاز البعيد، لا على الكمبيوتر المحمول المحلي. راجع [إدارة الجلسات](/ar/concepts/session).
  </Accordion>
</AccordionGroup>

## أساسيات الإعدادات

<AccordionGroup>
  <Accordion title="ما تنسيق الإعدادات؟ وأين توجد؟">
    يقرأ OpenClaw إعدادًا اختياريًا بصيغة **JSON5** من `$OPENCLAW_CONFIG_PATH` (الافتراضي: `~/.openclaw/openclaw.json`):

    ```
    $OPENCLAW_CONFIG_PATH
    ```

    وإذا كان الملف مفقودًا، فسيستخدم افتراضيات آمنة إلى حد ما (بما في ذلك مساحة عمل افتراضية `~/.openclaw/workspace`).

  </Accordion>

  <Accordion title='لقد ضبطت gateway.bind: "lan" (أو "tailnet") والآن لا شيء يستمع / UI تقول unauthorized'>
    تتطلب الربطات غير loopback **مسار مصادقة صالحًا للـ gateway**. عمليًا يعني هذا:

    - مصادقة shared-secret: token أو كلمة مرور
    - `gateway.auth.mode: "trusted-proxy"` خلف وكيل عكسي غير loopback ومدرك للهوية ومكوّن بشكل صحيح

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

    - لا يفعّل `gateway.remote.token` / `.password` مصادقة gateway المحلية بمفردهما.
    - يمكن لمسارات الاتصال المحلية أن تستخدم `gateway.remote.*` كبديل احتياطي فقط عندما تكون `gateway.auth.*` غير معيّنة.
    - لمصادقة كلمة المرور، اضبط `gateway.auth.mode: "password"` مع `gateway.auth.password` (أو `OPENCLAW_GATEWAY_PASSWORD`) بدلًا من ذلك.
    - إذا تم تكوين `gateway.auth.token` / `gateway.auth.password` صراحةً عبر SecretRef وكان غير قابل للحل، يفشل الحل بشكل مغلق (من دون إخفاء fallback بعيد).
    - تصادق إعدادات Control UI ذات shared-secret عبر `connect.params.auth.token` أو `connect.params.auth.password` (المخزّنتين في إعدادات app/UI). أما الأوضاع الحاملة للهوية مثل Tailscale Serve أو `trusted-proxy` فتستخدم ترويسات الطلب بدلًا من ذلك. تجنب وضع shared secrets في URLs.
    - مع `gateway.auth.mode: "trusted-proxy"`، فإن الوكلاء العكسيين من loopback على المضيف نفسه **لا** يلبّون مصادقة trusted-proxy. يجب أن يكون trusted proxy مصدرًا غير loopback ومكوّنًا.

  </Accordion>

  <Accordion title="لماذا أحتاج إلى token على localhost الآن؟">
    يفرض OpenClaw مصادقة gateway افتراضيًا، بما في ذلك loopback. وفي المسار الافتراضي العادي، يعني هذا مصادقة token: فإذا لم يُكوَّن مسار مصادقة صريح، فإن بدء gateway يحل إلى وضع token ويولّد token تلقائيًا، ويحفظه في `gateway.auth.token`، لذلك **يجب على عملاء WS المحليين المصادقة**. وهذا يمنع العمليات المحلية الأخرى من استدعاء Gateway.

    إذا كنت تفضّل مسار مصادقة مختلفًا، فيمكنك اختيار وضع كلمة المرور صراحةً (أو `trusted-proxy` للوكلاء العكسية غير loopback والمدركة للهوية). وإذا كنت **حقًا** تريد loopback مفتوحة، فاضبط `gateway.auth.mode: "none"` صراحةً في إعداداتك. ويمكن لـ Doctor إنشاء token لك في أي وقت: `openclaw doctor --generate-gateway-token`.

  </Accordion>

  <Accordion title="هل يجب أن أعيد التشغيل بعد تغيير الإعدادات؟">
    تراقب Gateway الإعدادات وتدعم hot-reload:

    - `gateway.reload.mode: "hybrid"` (الافتراضي): تطبّق التغييرات الآمنة hot-apply، وتعيد التشغيل للتغييرات الحرجة
    - كما أن `hot` و`restart` و`off` مدعومة أيضًا

  </Accordion>

  <Accordion title="كيف أعطل العبارات الطريفة في CLI؟">
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

    - `off`: يخفي نص tagline لكنه يُبقي سطر عنوان banner/الإصدار.
    - `default`: يستخدم `All your chats, one OpenClaw.` في كل مرة.
    - `random`: taglines طريفة/موسمية متناوبة (السلوك الافتراضي).
    - إذا كنت لا تريد banner إطلاقًا، فاضبط env ‏`OPENCLAW_HIDE_BANNER=1`.

  </Accordion>

  <Accordion title="كيف أفعّل web search (وweb fetch)؟">
    تعمل `web_fetch` من دون مفتاح API. أما `web_search` فتعتمد على
    الموفر الذي اخترته:

    - يتطلب الموفرون المعتمدون على API مثل Brave وExa وFirecrawl وGemini وGrok وKimi وMiniMax Search وPerplexity وTavily إعداد مفتاح API المعتاد لديهم.
    - إن Ollama Web Search لا يحتاج إلى مفتاح، لكنه يستخدم مضيف Ollama المكوَّن لديك ويتطلب `ollama signin`.
    - DuckDuckGo لا يحتاج إلى مفتاح، لكنه تكامل غير رسمي قائم على HTML.
    - SearXNG لا يحتاج إلى مفتاح/مستضاف ذاتيًا؛ اضبط `SEARXNG_BASE_URL` أو `plugins.entries.searxng.config.webSearch.baseUrl`.

    **الموصى به:** شغّل `openclaw configure --section web` واختر موفرًا.
    بدائل متغيرات البيئة:

    - Brave: ‏`BRAVE_API_KEY`
    - Exa: ‏`EXA_API_KEY`
    - Firecrawl: ‏`FIRECRAWL_API_KEY`
    - Gemini: ‏`GEMINI_API_KEY`
    - Grok: ‏`XAI_API_KEY`
    - Kimi: ‏`KIMI_API_KEY` أو `MOONSHOT_API_KEY`
    - MiniMax Search: ‏`MINIMAX_CODE_PLAN_KEY` أو `MINIMAX_CODING_API_KEY` أو `MINIMAX_API_KEY`
    - Perplexity: ‏`PERPLEXITY_API_KEY` أو `OPENROUTER_API_KEY`
    - SearXNG: ‏`SEARXNG_BASE_URL`
    - Tavily: ‏`TAVILY_API_KEY`

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
              provider: "firecrawl", // optional; omit for auto-detect
            },
          },
        },
    }
    ```

    تعيش إعدادات web-search الخاصة بكل موفر الآن تحت `plugins.entries.<plugin>.config.webSearch.*`.
    وما زالت مسارات الموفر القديمة `tools.web.search.*` تُحمَّل مؤقتًا من أجل التوافق، لكن لا ينبغي استخدامها في الإعدادات الجديدة.
    تعيش إعدادات fallback الخاصة بـ Firecrawl web-fetch تحت `plugins.entries.firecrawl.config.webFetch.*`.

    ملاحظات:

    - إذا كنت تستخدم allowlists، فأضف `web_search`/`web_fetch`/`x_search` أو `group:web`.
    - `web_fetch` مفعّلة افتراضيًا (ما لم تُعطّل صراحةً).
    - إذا تم حذف `tools.web.fetch.provider`، يكتشف OpenClaw تلقائيًا أول موفر fetch fallback جاهز من بيانات الاعتماد المتاحة. والموفر المضمّن اليوم هو Firecrawl.
    - تقرأ daemons متغيرات env من `~/.openclaw/.env` (أو من بيئة الخدمة).

    الوثائق: [أدوات الويب](/ar/tools/web).

  </Accordion>

  <Accordion title="قام config.apply بمسح إعداداتي. كيف أستعيدها وأتجنب هذا؟">
    يقوم `config.apply` باستبدال **الإعدادات بالكامل**. فإذا أرسلت كائنًا جزئيًا، فسيُزال
    كل شيء آخر.

    الاستعادة:

    - استعد من نسخة احتياطية (git أو نسخة من `~/.openclaw/openclaw.json`).
    - إذا لم تكن لديك نسخة احتياطية، فأعد تشغيل `openclaw doctor` وأعد تهيئة القنوات/النماذج.
    - إذا كان هذا غير متوقع، فافتح تقرير خطأ وأرفق آخر إعدادات معروفة أو أي نسخة احتياطية.
    - يمكن لوكيل برمجي محلي في كثير من الأحيان إعادة بناء إعدادات عاملة من السجلات أو التاريخ.

    لتجنب ذلك:

    - استخدم `openclaw config set` للتغييرات الصغيرة.
    - استخدم `openclaw configure` للتحرير التفاعلي.
    - استخدم `config.schema.lookup` أولًا عندما لا تكون متأكدًا من المسار الدقيق أو شكل الحقل؛ فهو يعيد عقدة schema سطحية مع ملخصات الأبناء المباشرين من أجل التعمق.
    - استخدم `config.patch` لتحريرات RPC الجزئية؛ وأبقِ `config.apply` لاستبدال الإعدادات بالكامل فقط.
    - إذا كنت تستخدم أداة `gateway` المقتصرة على المالك من داخل تشغيل وكيل، فستظل ترفض الكتابة إلى `tools.exec.ask` / `tools.exec.security` (بما في ذلك الأسماء القديمة `tools.bash.*` التي تُطبّع إلى المسارات المحمية نفسها لـ exec).

    الوثائق: [الإعداد](/cli/config)، [التهيئة](/cli/configure)، [Doctor](/ar/gateway/doctor).

  </Accordion>

  <Accordion title="كيف أشغّل Gateway مركزية مع عمال متخصصين عبر الأجهزة؟">
    النمط الشائع هو **Gateway واحدة** (مثل Raspberry Pi) بالإضافة إلى **nodes** و**agents**:

    - **Gateway (المركزية):** تملك القنوات (Signal/WhatsApp)، والتوجيه، والجلسات.
    - **Nodes (الأجهزة):** تتصل أجهزة Mac/iOS/Android بوصفها ملحقات وتعرض أدوات محلية (`system.run` و`canvas` و`camera`).
    - **Agents (العمال):** عقول/مساحات عمل منفصلة لأدوار متخصصة (مثل "Hetzner ops" أو "Personal data").
    - **الوكلاء الفرعيون:** أنشئ عملاً في الخلفية من وكيل رئيسي عندما تريد التوازي.
    - **TUI:** اتصل بـ Gateway وبدّل بين الوكلاء/الجلسات.

    الوثائق: [Nodes](/ar/nodes)، [الوصول البعيد](/ar/gateway/remote)، [التوجيه متعدد الوكلاء](/ar/concepts/multi-agent)، [الوكلاء الفرعيون](/ar/tools/subagents)، [TUI](/web/tui).

  </Accordion>

  <Accordion title="هل يمكن لـ OpenClaw browser العمل headless؟">
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

    القيمة الافتراضية هي `false` ‏(headful). ويكون headless أكثر عرضة لتفعيل فحوصات anti-bot على بعض المواقع. راجع [Browser](/ar/tools/browser).

    يستخدم وضع headless **محرك Chromium نفسه** ويعمل في معظم سيناريوهات الأتمتة (النماذج، والنقرات، والاستخلاص، وتسجيلات الدخول). والفروق الأساسية هي:

    - لا توجد نافذة متصفح مرئية (استخدم لقطات الشاشة إذا كنت تحتاج إلى عناصر بصرية).
    - بعض المواقع أكثر صرامة مع الأتمتة في وضع headless ‏(CAPTCHAs وanti-bot).
      على سبيل المثال، كثيرًا ما يحظر X/Twitter الجلسات headless.

  </Accordion>

  <Accordion title="كيف أستخدم Brave للتحكم في المتصفح؟">
    اضبط `browser.executablePath` على ملف Brave الثنائي لديك (أو أي متصفح قائم على Chromium) ثم أعد تشغيل Gateway.
    راجع أمثلة الإعدادات الكاملة في [Browser](/ar/tools/browser#use-brave-or-another-chromium-based-browser).
  </Accordion>
</AccordionGroup>

## Gateways البعيدة وnodes

<AccordionGroup>
  <Accordion title="كيف تنتشر الأوامر بين Telegram وgateway وnodes؟">
    تتعامل **gateway** مع رسائل Telegram. تشغّل gateway الوكيل
    وبعد ذلك فقط تستدعي nodes عبر **Gateway WebSocket** عندما تحتاج إلى أداة node:

    Telegram → Gateway → Agent → `node.*` → Node → Gateway → Telegram

    لا ترى Nodes حركة مرور الموفر الواردة؛ فهي تتلقى فقط استدعاءات RPC الخاصة بـ node.

  </Accordion>

  <Accordion title="كيف يمكن لوكيلي الوصول إلى جهازي إذا كانت Gateway مستضافًا عن بُعد؟">
    الإجابة القصيرة: **أقرِن جهازك بوصفه node**. تعمل Gateway في مكان آخر، لكنها تستطيع
    استدعاء أدوات `node.*` ‏(الشاشة، والكاميرا، والنظام) على جهازك المحلي عبر Gateway WebSocket.

    إعداد نموذجي:

    1. شغّل Gateway على المضيف الدائم التشغيل (VPS/خادم منزلي).
    2. ضع مضيف Gateway + جهازك على tailnet نفسها.
    3. تأكد من إمكانية الوصول إلى Gateway WS ‏(tailnet bind أو SSH tunnel).
    4. افتح تطبيق macOS محليًا واتصل في وضع **Remote over SSH** ‏(أو tailnet مباشر)
       حتى يتمكن من التسجيل بوصفه node.
    5. وافق على node على Gateway:

       ```bash
       openclaw devices list
       openclaw devices approve <requestId>
       ```

    لا حاجة إلى جسر TCP منفصل؛ تتصل nodes عبر Gateway WebSocket.

    تذكير أمني: يسمح إقران macOS node بتشغيل `system.run` على ذلك الجهاز. أقرن
    فقط الأجهزة التي تثق بها، وراجع [الأمان](/ar/gateway/security).

    الوثائق: [Nodes](/ar/nodes)، [بروتوكول Gateway](/ar/gateway/protocol)، [الوضع البعيد على macOS](/ar/platforms/mac/remote)، [الأمان](/ar/gateway/security).

  </Accordion>

  <Accordion title="Tailscale متصل لكنني لا أتلقى أي ردود. ماذا الآن؟">
    تحقق من الأساسيات:

    - Gateway تعمل: `openclaw gateway status`
    - صحة Gateway: ‏`openclaw status`
    - صحة القناة: ‏`openclaw channels status`

    ثم تحقق من المصادقة والتوجيه:

    - إذا كنت تستخدم Tailscale Serve، فتأكد من ضبط `gateway.auth.allowTailscale` بشكل صحيح.
    - إذا كنت تتصل عبر SSH tunnel، فتأكد من أن النفق المحلي قائم ويشير إلى المنفذ الصحيح.
    - تحقق من أن allowlists لديك (DM أو المجموعة) تتضمن حسابك.

    الوثائق: [Tailscale](/ar/gateway/tailscale)، [الوصول البعيد](/ar/gateway/remote)، [القنوات](/ar/channels).

  </Accordion>

  <Accordion title="هل يمكن لنسختين من OpenClaw التحدث إلى بعضهما (محلي + VPS)؟">
    نعم. لا يوجد جسر "bot-to-bot" مضمّن، لكن يمكنك توصيل ذلك بعدة
    طرق موثوقة:

    **الأبسط:** استخدم قناة دردشة عادية يمكن لكلا botين الوصول إليها (Telegram/Slack/WhatsApp).
    اجعل Bot A يرسل رسالة إلى Bot B، ثم دع Bot B يرد كالمعتاد.

    **جسر CLI ‏(عام):** شغّل برنامجًا نصيًا يستدعي Gateway الأخرى بواسطة
    `openclaw agent --message ... --deliver`، مع استهداف دردشة يستمع فيها bot الآخر.
    وإذا كان أحد botين موجودًا على VPS بعيدة، فاجعل CLI لديك تشير إلى تلك Gateway البعيدة
    عبر SSH/Tailscale (راجع [الوصول البعيد](/ar/gateway/remote)).

    نمط مثال (شغّله من جهاز يمكنه الوصول إلى Gateway المستهدفة):

    ```bash
    openclaw agent --message "Hello from local bot" --deliver --channel telegram --reply-to <chat-id>
    ```

    نصيحة: أضف قاعدة حماية حتى لا يدخل botان في حلقة لا نهائية (اشترط الإشارة فقط، أو
    قوائم سماح للقنوات، أو قاعدة "لا ترد على رسائل bot").

    الوثائق: [الوصول البعيد](/ar/gateway/remote)، [Agent CLI](/cli/agent)، [إرسال الوكيل](/ar/tools/agent-send).

  </Accordion>

  <Accordion title="هل أحتاج إلى VPSs منفصلة لعدة وكلاء؟">
    لا. يمكن لـ Gateway واحدة استضافة عدة وكلاء، لكل منهم مساحة عمل، وافتراضيات نموذج،
    وتوجيه خاص به. وهذا هو الإعداد المعتاد، وهو أرخص وأبسط بكثير من تشغيل
    VPS واحدة لكل وكيل.

    استخدم VPSs منفصلة فقط عندما تحتاج إلى عزل صارم (حدود أمان) أو إعدادات
    مختلفة جدًا لا تريد مشاركتها. وبخلاف ذلك، أبقِ Gateway واحدة
    واستخدم عدة وكلاء أو وكلاء فرعيين.

  </Accordion>

  <Accordion title="هل هناك فائدة من استخدام node على الكمبيوتر المحمول الشخصي بدلًا من SSH من VPS؟">
    نعم - تُعد nodes الطريقة الأولى للوصول إلى الكمبيوتر المحمول من Gateway بعيدة، وهي
    تفتح أكثر من مجرد وصول shell. تعمل Gateway على macOS/Linux ‏(وWindows عبر WSL2) وهي
    خفيفة (يكفي VPS صغير أو جهاز من فئة Raspberry Pi؛ و4 GB RAM أكثر من كافية)، لذا فإن الإعداد
    الشائع هو مضيف دائم التشغيل مع الكمبيوتر المحمول بوصفه node.

    - **لا حاجة إلى SSH وارد.** تتصل Nodes خارجيًا إلى Gateway WebSocket وتستخدم إقران الأجهزة.
    - **ضوابط تنفيذ أكثر أمانًا.** يُقيَّد `system.run` بواسطة allowlists/موافقات node على ذلك الكمبيوتر المحمول.
    - **أدوات جهاز أكثر.** تعرض Nodes كلًا من `canvas` و`camera` و`screen` إضافة إلى `system.run`.
    - **أتمتة المتصفح محليًا.** أبقِ Gateway على VPS، لكن شغّل Chrome محليًا عبر node host على الكمبيوتر المحمول، أو اتصل بـ Chrome المحلية على المضيف عبر Chrome MCP.

    إن SSH مناسبة للوصول إلى shell عند الحاجة، لكن nodes أبسط لسير عمل الوكلاء المستمر
    وأتمتة الأجهزة.

    الوثائق: [Nodes](/ar/nodes)، [Nodes CLI](/cli/nodes)، [Browser](/ar/tools/browser).

  </Accordion>

  <Accordion title="هل تشغّل nodes خدمة gateway؟">
    لا. يجب أن تعمل **gateway واحدة** فقط على كل مضيف ما لم تكن تشغّل عمدًا ملفات تعريف معزولة (راجع [Gateways متعددة](/ar/gateway/multiple-gateways)). إن nodes ملحقات تتصل
    بـ gateway ‏(nodes iOS/Android، أو "وضع node" في تطبيق شريط القوائم على macOS). ولـ headless node
    hosts والتحكم عبر CLI، راجع [Node host CLI](/cli/node).

    يتطلب التغيير في `gateway` و`discovery` و`canvasHost` إعادة تشغيل كاملة.

  </Accordion>

  <Accordion title="هل توجد طريقة API / RPC لتطبيق الإعدادات؟">
    نعم.

    - `config.schema.lookup`: افحص شجرة إعدادات فرعية واحدة مع عقدة schema السطحية، وتلميح UI المطابق، وملخصات الأبناء المباشرين قبل الكتابة
    - `config.get`: اجلب اللقطة الحالية + hash
    - `config.patch`: تحديث جزئي آمن (المفضل لمعظم تحريرات RPC)
    - `config.apply`: تحقّق + استبدل الإعدادات الكاملة، ثم أعد التشغيل
    - لا تزال أداة وقت التشغيل `gateway` المقتصرة على المالك ترفض إعادة كتابة `tools.exec.ask` / `tools.exec.security`؛ وتُطبّع الأسماء القديمة `tools.bash.*` إلى مسارات exec المحمية نفسها

  </Accordion>

  <Accordion title="أدنى إعداد معقول للتثبيت الأول">
    ```json5
    {
      agents: { defaults: { workspace: "~/.openclaw/workspace" } },
      channels: { whatsapp: { allowFrom: ["+15555550123"] } },
    }
    ```

    يحدد هذا مساحة عملك ويقيّد من يمكنه تشغيل bot.

  </Accordion>

  <Accordion title="كيف أعد Tailscale على VPS وأتصل من Mac؟">
    خطوات دنيا:

    1. **التثبيت + تسجيل الدخول على VPS**

       ```bash
       curl -fsSL https://tailscale.com/install.sh | sh
       sudo tailscale up
       ```

    2. **التثبيت + تسجيل الدخول على Mac**
       - استخدم تطبيق Tailscale وسجّل الدخول إلى tailnet نفسها.
    3. **فعّل MagicDNS (مستحسن)**
       - فعّل MagicDNS في Tailscale admin console حتى تحصل VPS على اسم ثابت.
    4. **استخدم اسم مضيف tailnet**
       - SSH: ‏`ssh user@your-vps.tailnet-xxxx.ts.net`
       - Gateway WS: ‏`ws://your-vps.tailnet-xxxx.ts.net:18789`

    إذا كنت تريد Control UI من دون SSH، فاستخدم Tailscale Serve على VPS:

    ```bash
    openclaw gateway --tailscale serve
    ```

    يؤدي هذا إلى إبقاء gateway مرتبطة بـ loopback مع تعريض HTTPS عبر Tailscale. راجع [Tailscale](/ar/gateway/tailscale).

  </Accordion>

  <Accordion title="كيف أوصل Mac node إلى Gateway بعيدة (Tailscale Serve)؟">
    يعرّض Serve **Gateway Control UI + WS**. وتتصل nodes عبر نقطة نهاية Gateway WS نفسها.

    الإعداد الموصى به:

    1. **تأكد من أن VPS + Mac على tailnet نفسها**.
    2. **استخدم تطبيق macOS في الوضع البعيد** (يمكن أن يكون هدف SSH هو اسم مضيف tailnet).
       سيقوم التطبيق بنفق منفذ Gateway والاتصال بوصفه node.
    3. **وافق على node** على gateway:

       ```bash
       openclaw devices list
       openclaw devices approve <requestId>
       ```

    الوثائق: [بروتوكول Gateway](/ar/gateway/protocol)، [الاكتشاف](/ar/gateway/discovery)، [الوضع البعيد على macOS](/ar/platforms/mac/remote).

  </Accordion>

  <Accordion title="هل ينبغي أن أثبت على كمبيوتر محمول ثانٍ أم أضيف node فقط؟">
    إذا كنت تحتاج فقط إلى **أدوات محلية** (الشاشة/الكاميرا/exec) على الكمبيوتر المحمول الثاني، فأضفه بوصفه
    **node**. فهذا يُبقي Gateway واحدة ويتجنب تكرار الإعدادات. وتقتصر أدوات node المحلية
    حاليًا على macOS، لكننا نخطط لتوسيعها إلى أنظمة تشغيل أخرى.

    ثبّت Gateway ثانية فقط عندما تحتاج إلى **عزل صارم** أو botين منفصلين بالكامل.

    الوثائق: [Nodes](/ar/nodes)، [Nodes CLI](/cli/nodes)، [Gateways متعددة](/ar/gateway/multiple-gateways).

  </Accordion>
</AccordionGroup>

## متغيرات Env وتحميل .env

<AccordionGroup>
  <Accordion title="كيف يحمّل OpenClaw متغيرات البيئة؟">
    يقرأ OpenClaw متغيرات env من العملية الأم (shell أو launchd/systemd أو CI، إلخ) ويحمّل أيضًا:

    - `.env` من دليل العمل الحالي
    - `.env` احتياطيًا عالميًا من `~/.openclaw/.env` (أي `$OPENCLAW_STATE_DIR/.env`)

    لا يكتب أي من ملفي `.env` فوق متغيرات env الموجودة بالفعل.

    يمكنك أيضًا تعريف متغيرات env مضمّنة في الإعدادات (تُطبّق فقط إذا كانت غير موجودة في env الخاصة بالعملية):

    ```json5
    {
      env: {
        OPENROUTER_API_KEY: "sk-or-...",
        vars: { GROQ_API_KEY: "gsk-..." },
      },
    }
    ```

    راجع [/environment](/ar/help/environment) للحصول على الأولوية الكاملة والمصادر.

  </Accordion>

  <Accordion title="لقد بدأت Gateway عبر الخدمة واختفت متغيرات env الخاصة بي. ماذا الآن؟">
    هناك إصلاحان شائعان:

    1. ضع المفاتيح المفقودة في `~/.openclaw/.env` حتى تُلتقط حتى عندما لا ترث الخدمة shell env الخاصة بك.
    2. فعّل استيراد shell ‏(وسيلة راحة اختيارية):

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

    يقوم هذا بتشغيل login shell لديك ويستورد فقط المفاتيح المتوقعة المفقودة (من دون أي override). المكافئات كمتغيرات env:
    `OPENCLAW_LOAD_SHELL_ENV=1` و`OPENCLAW_SHELL_ENV_TIMEOUT_MS=15000`.

  </Accordion>

  <Accordion title='لقد ضبطت COPILOT_GITHUB_TOKEN، لكن models status تعرض "Shell env: off." لماذا؟'>
    يعرض `openclaw models status` ما إذا كان **استيراد shell env** مفعّلًا. والعبارة "Shell env: off"
    لا تعني أن متغيرات env مفقودة - بل تعني فقط أن OpenClaw لن يحمّل
    login shell لديك تلقائيًا.

    إذا كانت Gateway تعمل كخدمة (launchd/systemd)، فلن ترث
    بيئة shell لديك. أصلح ذلك بإحدى الطرق التالية:

    1. ضع token في `~/.openclaw/.env`:

       ```
       COPILOT_GITHUB_TOKEN=...
       ```

    2. أو فعّل استيراد shell (`env.shellEnv.enabled: true`).
    3. أو أضفه إلى كتلة `env` في إعداداتك (يُطبّق فقط إذا كان مفقودًا).

    ثم أعد تشغيل gateway وتحقق مرة أخرى:

    ```bash
    openclaw models status
    ```

    تتم قراءة Copilot tokens من `COPILOT_GITHUB_TOKEN` (وأيضًا `GH_TOKEN` / `GITHUB_TOKEN`).
    راجع [/concepts/model-providers](/ar/concepts/model-providers) و[/environment](/ar/help/environment).

  </Accordion>
</AccordionGroup>

## الجلسات والدردشات المتعددة

<AccordionGroup>
  <Accordion title="كيف أبدأ محادثة جديدة؟">
    أرسل `/new` أو `/reset` كرسالة مستقلة. راجع [إدارة الجلسات](/ar/concepts/session).
  </Accordion>

  <Accordion title="هل تُعاد تعيين الجلسات تلقائيًا إذا لم أرسل /new أبدًا؟">
    يمكن أن تنتهي الجلسات بعد `session.idleMinutes`، لكن هذا **معطّل افتراضيًا** (القيمة الافتراضية **0**).
    اضبطها على قيمة موجبة لتفعيل انتهاء الخمول. وعند تفعيلها، ستبدأ الرسالة **التالية**
    بعد فترة الخمول معرّف جلسة جديدًا لذلك chat key.
    هذا لا يحذف النصوص الحوارية - بل يبدأ جلسة جديدة فقط.

    ```json5
    {
      session: {
        idleMinutes: 240,
      },
    }
    ```

  </Accordion>

  <Accordion title="هل توجد طريقة لتكوين فريق من نُسخ OpenClaw (CEO واحد والعديد من الوكلاء)؟">
    نعم، عبر **التوجيه متعدد الوكلاء** و**الوكلاء الفرعيين**. يمكنك إنشاء
    وكيل تنسيق واحد والعديد من وكلاء العمال بمساحات عمل ونماذج خاصة بهم.

    مع ذلك، من الأفضل النظر إلى هذا باعتباره **تجربة ممتعة**. فهو يستهلك tokens بكثافة وغالبًا
    ما يكون أقل كفاءة من استخدام bot واحد مع جلسات منفصلة. النموذج المعتاد الذي
    نتصوره هو bot واحد تتحدث إليه، مع جلسات مختلفة للعمل المتوازي. ويمكن لهذا
    bot أيضًا إنشاء وكلاء فرعيين عند الحاجة.

    الوثائق: [التوجيه متعدد الوكلاء](/ar/concepts/multi-agent)، [الوكلاء الفرعيون](/ar/tools/subagents)، [Agents CLI](/cli/agents).

  </Accordion>

  <Accordion title="لماذا اقتُطع السياق في منتصف المهمة؟ وكيف أمنع ذلك؟">
    يقتصر سياق الجلسة على نافذة النموذج. يمكن للمحادثات الطويلة أو مخرجات الأدوات الكبيرة أو عدد كبير
    من الملفات أن تحفّز الضغط أو الاقتطاع.

    ما الذي يساعد:

    - اطلب من bot أن تلخّص الحالة الحالية وتكتبها إلى ملف.
    - استخدم `/compact` قبل المهام الطويلة، و`/new` عند تبديل الموضوعات.
    - أبقِ السياق المهم في مساحة العمل واطلب من bot قراءته مرة أخرى.
    - استخدم الوكلاء الفرعيين للعمل الطويل أو المتوازي حتى تبقى الدردشة الرئيسية أصغر.
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

    - يعرض الإعداد الأولي أيضًا **Reset** إذا اكتشف إعدادات موجودة. راجع [الإعداد الأولي (CLI)](/ar/start/wizard).
    - إذا كنت قد استخدمت profiles ‏(`--profile` / `OPENCLAW_PROFILE`)، فأعد تعيين كل دليل حالة (الافتراضيات هي `~/.openclaw-<profile>`).
    - إعادة تعيين Dev: ‏`openclaw gateway --dev --reset` ‏(لـ dev فقط؛ يمسح إعدادات dev + بيانات الاعتماد + الجلسات + مساحة العمل).

  </Accordion>

  <Accordion title='أتلقى أخطاء "context too large" - كيف أعيد التعيين أو أضغط السياق؟'>
    استخدم أحد هذين:

    - **الضغط** (يبقي المحادثة لكنه يلخص الأدوار الأقدم):

      ```
      /compact
      ```

      أو `/compact <instructions>` لتوجيه الملخص.

    - **إعادة التعيين** (معرّف جلسة جديد لنفس chat key):

      ```
      /new
      /reset
      ```

    إذا استمر ذلك:

    - فعّل أو اضبط **session pruning** ‏(`agents.defaults.contextPruning`) لتقليم مخرجات الأدوات القديمة.
    - استخدم نموذجًا بنافذة سياق أكبر.

    الوثائق: [الضغط](/ar/concepts/compaction)، [تشذيب الجلسة](/ar/concepts/session-pruning)، [إدارة الجلسات](/ar/concepts/session).

  </Accordion>

  <Accordion title='لماذا أرى "LLM request rejected: messages.content.tool_use.input field required"؟'>
    هذا خطأ تحقق من الموفّر: أصدر النموذج كتلة `tool_use` من دون `input` المطلوبة.
    ويعني هذا عادةً أن سجل الجلسة قديم أو تالف (غالبًا بعد سلاسل طويلة
    أو تغيير في أداة/schema).

    الحل: ابدأ جلسة جديدة بواسطة `/new` ‏(رسالة مستقلة).

  </Accordion>

  <Accordion title="لماذا أتلقى رسائل heartbeat كل 30 دقيقة؟">
    تعمل Heartbeats كل **30m** افتراضيًا (**1h** عند استخدام مصادقة OAuth). اضبطها أو عطّلها:

    ```json5
    {
      agents: {
        defaults: {
          heartbeat: {
            every: "2h", // or "0m" to disable
          },
        },
      },
    }
    ```

    إذا كان `HEARTBEAT.md` موجودًا لكنه فارغ فعليًا (أسطر فارغة وعناوين markdown
    فقط مثل `# Heading`)، فإن OpenClaw تتخطى تشغيل heartbeat لتوفير مكالمات API.
    وإذا كان الملف مفقودًا، فإن heartbeat تظل تعمل ويقرر النموذج ما الذي سيفعله.

    تستخدم تجاوزات كل وكيل `agents.list[].heartbeat`. الوثائق: [Heartbeat](/ar/gateway/heartbeat).

  </Accordion>

  <Accordion title='هل أحتاج إلى إضافة "حساب bot" إلى مجموعة WhatsApp؟'>
    لا. يعمل OpenClaw على **حسابك أنت**، لذا إذا كنت في المجموعة، يمكن لـ OpenClaw رؤيتها.
    افتراضيًا، يتم حظر الردود في المجموعات حتى تسمح للمرسلين (`groupPolicy: "allowlist"`).

    وإذا كنت تريد أن تكون **أنت فقط** قادرًا على تشغيل الردود داخل المجموعات:

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
    الخيار 1 (الأسرع): تابع السجلات وأرسل رسالة اختبار في المجموعة:

    ```bash
    openclaw logs --follow --json
    ```

    ابحث عن `chatId` (أو `from`) المنتهية بـ `@g.us`، مثل:
    `1234567890-1234567890@g.us`.

    الخيار 2 (إذا كان مضبوطًا/مدرجًا في allowlist بالفعل): اسرد المجموعات من الإعدادات:

    ```bash
    openclaw directory groups list --channel whatsapp
    ```

    الوثائق: [WhatsApp](/ar/channels/whatsapp)، [Directory](/cli/directory)، [Logs](/cli/logs).

  </Accordion>

  <Accordion title="لماذا لا يرد OpenClaw داخل مجموعة؟">
    سببان شائعان:

    - تقييد الإشارات مفعّل (افتراضيًا). يجب أن تقوم @mention للـ bot (أو تطابق `mentionPatterns`).
    - قمت بتكوين `channels.whatsapp.groups` من دون `"*"` والمجموعة ليست في allowlist.

    راجع [المجموعات](/ar/channels/groups) و[رسائل المجموعات](/ar/channels/group-messages).

  </Accordion>

  <Accordion title="هل تشارك المجموعات/السلاسل السياق مع الرسائل الخاصة؟">
    تنهار الدردشات المباشرة إلى الجلسة الرئيسية افتراضيًا. أما المجموعات/القنوات فلها مفاتيح جلسات خاصة بها، كما أن موضوعات Telegram / سلاسل Discord هي جلسات منفصلة. راجع [المجموعات](/ar/channels/groups) و[رسائل المجموعات](/ar/channels/group-messages).
  </Accordion>

  <Accordion title="كم عدد مساحات العمل والوكلاء التي يمكنني إنشاؤها؟">
    لا توجد حدود صارمة. العشرات (بل حتى المئات) مقبولة، لكن انتبه إلى:

    - **نمو القرص:** تعيش الجلسات + النصوص الحوارية تحت `~/.openclaw/agents/<agentId>/sessions/`.
    - **تكلفة tokens:** المزيد من الوكلاء يعني استخدامًا أكبر متزامنًا للنماذج.
    - **أعباء التشغيل:** ملفات تعريف المصادقة ومساحات العمل وتوجيه القنوات لكل وكيل.

    نصائح:

    - أبقِ **مساحة عمل نشطة** واحدة لكل وكيل (`agents.defaults.workspace`).
    - نظّف الجلسات القديمة (احذف JSONL أو إدخالات store) إذا نما القرص.
    - استخدم `openclaw doctor` لاكتشاف مساحات العمل الشاردة وعدم تطابق profiles.

  </Accordion>

  <Accordion title="هل يمكنني تشغيل عدة bots أو دردشات في الوقت نفسه (Slack)، وكيف أعد ذلك؟">
    نعم. استخدم **التوجيه متعدد الوكلاء** لتشغيل عدة وكلاء معزولين وتوجيه الرسائل الواردة حسب
    القناة/الحساب/peer. Slack مدعوم بوصفها قناة ويمكن ربطها بوكلاء محددين.

    الوصول إلى Browser قوي، لكنه ليس "افعل أي شيء يستطيع الإنسان فعله" - فما زالت anti-bot وCAPTCHAs وMFA
    قادرة على حظر الأتمتة. ومن أجل أكثر تحكم موثوق في Browser، استخدم Chrome MCP محليًا على المضيف،
    أو استخدم CDP على الجهاز الذي يشغّل Browser بالفعل.

    إعداد أفضل الممارسات:

    - مضيف Gateway دائم التشغيل (VPS/Mac mini).
    - وكيل واحد لكل دور (bindings).
    - قنوات Slack مرتبطة بهؤلاء الوكلاء.
    - Browser محلي عبر Chrome MCP أو node عند الحاجة.

    الوثائق: [التوجيه متعدد الوكلاء](/ar/concepts/multi-agent)، [Slack](/ar/channels/slack)،
    [Browser](/ar/tools/browser)، [Nodes](/ar/nodes).

  </Accordion>
</AccordionGroup>

## النماذج: الافتراضيات، والاختيار، والأسماء المستعارة، والتبديل

<AccordionGroup>
  <Accordion title='ما هو "النموذج الافتراضي"؟'>
    النموذج الافتراضي في OpenClaw هو ما تضبطه على أنه:

    ```
    agents.defaults.model.primary
    ```

    يُشار إلى النماذج بالشكل `provider/model` (مثال: `openai/gpt-5.4`). وإذا حذفت الموفّر، فإن OpenClaw يحاول أولًا اسمًا مستعارًا، ثم مطابقة فريدة لموفّر مهيأ لذلك model id الدقيق، وبعد ذلك فقط يعود إلى الموفّر الافتراضي المهيأ كمسار توافق قديم. وإذا لم يعد ذلك الموفّر يعرّض النموذج الافتراضي المهيأ، فإن OpenClaw يعود إلى أول موفّر/نموذج مهيأ بدلًا من إظهار افتراضي قديم لموفّر أُزيل. ومع ذلك، ينبغي لك **تحديد `provider/model` صراحةً**.

  </Accordion>

  <Accordion title="ما النموذج الذي توصي به؟">
    **الافتراضي الموصى به:** استخدم أقوى نموذج من أحدث جيل متاح في مجموعة الموفّرين لديك.
    **بالنسبة إلى الوكلاء ذوي الأدوات أو المدخلات غير الموثوقة:** أعطِ قوة النموذج الأولوية على التكلفة.
    **بالنسبة إلى الدردشة الروتينية/منخفضة المخاطر:** استخدم نماذج احتياطية أرخص ووجّه حسب دور الوكيل.

    لدى MiniMax وثائقها الخاصة: [MiniMax](/ar/providers/minimax) و
    [النماذج المحلية](/ar/gateway/local-models).

    القاعدة العامة: استخدم **أفضل نموذج يمكنك تحمّل تكلفته** للعمل عالي المخاطر، ونموذجًا أرخص
    للدردشة الروتينية أو الملخصات. ويمكنك توجيه النماذج لكل وكيل واستخدام الوكلاء الفرعيين
    لموازاة المهام الطويلة (يستهلك كل وكيل فرعي tokens). راجع [النماذج](/ar/concepts/models) و
    [الوكلاء الفرعيون](/ar/tools/subagents).

    تحذير قوي: النماذج الأضعف/المكمّمة أكثر من اللازم أكثر عرضة لـ prompt
    injection والسلوك غير الآمن. راجع [الأمان](/ar/gateway/security).

    مزيد من السياق: [النماذج](/ar/concepts/models).

  </Accordion>

  <Accordion title="كيف أبدّل النماذج من دون مسح إعداداتي؟">
    استخدم **أوامر النموذج** أو عدّل حقول **النموذج** فقط. وتجنب استبدال الإعدادات بالكامل.

    الخيارات الآمنة:

    - `/model` في الدردشة (سريع، لكل جلسة)
    - `openclaw models set ...` ‏(يحدّث إعدادات النموذج فقط)
    - `openclaw configure --section model` ‏(تفاعلي)
    - عدّل `agents.defaults.model` في `~/.openclaw/openclaw.json`

    تجنب `config.apply` مع كائن جزئي ما لم تكن تقصد استبدال الإعدادات كلها.
    أما لتحريرات RPC، فافحص أولًا بواسطة `config.schema.lookup` وفضّل `config.patch`. تمنحك حمولة lookup المسار المُطبّع، ووثائق/قيود schema السطحية، وملخصات الأبناء المباشرين
    من أجل التحديثات الجزئية.
    وإذا كنت قد كتبت فوق الإعدادات، فاستعد من نسخة احتياطية أو أعد تشغيل `openclaw doctor` للإصلاح.

    الوثائق: [النماذج](/ar/concepts/models)، [التهيئة](/cli/configure)، [الإعداد](/cli/config)، [Doctor](/ar/gateway/doctor).

  </Accordion>

  <Accordion title="هل يمكنني استخدام نماذج مستضافة ذاتيًا (llama.cpp, vLLM, Ollama)؟">
    نعم. تُعد Ollama أسهل مسار للنماذج المحلية.

    أسرع إعداد:

    1. ثبّت Ollama من `https://ollama.com/download`
    2. اسحب نموذجًا محليًا مثل `ollama pull glm-4.7-flash`
    3. إذا كنت تريد نماذج سحابية أيضًا، شغّل `ollama signin`
    4. شغّل `openclaw onboard` واختر `Ollama`
    5. اختر `Local` أو `Cloud + Local`

    ملاحظات:

    - يمنحك `Cloud + Local` نماذج سحابية بالإضافة إلى نماذج Ollama المحلية لديك
    - لا تحتاج النماذج السحابية مثل `kimi-k2.5:cloud` إلى سحب محلي
    - للتبديل اليدوي، استخدم `openclaw models list` و`openclaw models set ollama/<model>`

    ملاحظة أمان: النماذج الأصغر أو المكمّمة بشدة أكثر عرضة لـ prompt
    injection. ونوصي بشدة باستخدام **نماذج كبيرة** لأي bot يمكنه استخدام الأدوات.
    وإذا كنت ما زلت تريد نماذج صغيرة، ففعّل sandboxing مع allowlists صارمة للأدوات.

    الوثائق: [Ollama](/ar/providers/ollama)، [النماذج المحلية](/ar/gateway/local-models)،
    [موفرو النماذج](/ar/concepts/model-providers)، [الأمان](/ar/gateway/security)،
    [Sandboxing](/ar/gateway/sandboxing).

  </Accordion>

  <Accordion title="ماذا تستخدم OpenClaw وFlawd وKrill من نماذج؟">
    - قد تختلف هذه البيئات وقد تتغير بمرور الوقت؛ لا توجد توصية ثابتة بموفّر واحد.
    - تحقق من إعداد وقت التشغيل الحالي على كل gateway بواسطة `openclaw models status`.
    - بالنسبة إلى الوكلاء الحسّاسين أمنيًا/المزوّدين بالأدوات، استخدم أقوى نموذج من أحدث جيل متاح.
  </Accordion>

  <Accordion title="كيف أبدّل النماذج أثناء التشغيل (من دون إعادة تشغيل)؟">
    استخدم أمر `/model` كرسالة مستقلة:

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

    يمكنك إدراج النماذج المتاحة بواسطة `/model` أو `/model list` أو `/model status`.

    يعرض `/model` (و`/model list`) منتقيًا مضغوطًا ومرقّمًا. اختر بالرقم:

    ```
    /model 3
    ```

    كما يمكنك فرض ملف تعريف مصادقة محدد للموفّر (لكل جلسة):

    ```
    /model opus@anthropic:default
    /model opus@anthropic:work
    ```

    نصيحة: يعرض `/model status` أي وكيل نشط، وأي ملف `auth-profiles.json` مستخدم، وأي ملف تعريف مصادقة سيُجرَّب بعد ذلك.
    كما يعرض provider endpoint المهيأ (`baseUrl`) ووضع API ‏(`api`) عند توفرهما.

    **كيف ألغي تثبيت ملف تعريف قمت بتعيينه باستخدام @profile؟**

    أعد تشغيل `/model` **من دون** لاحقة `@profile`:

    ```
    /model anthropic/claude-opus-4-6
    ```

    وإذا كنت تريد العودة إلى الافتراضي، فاختره من `/model` (أو أرسل `/model <default provider/model>`).
    استخدم `/model status` للتأكد من ملف تعريف المصادقة النشط.

  </Accordion>

  <Accordion title="هل يمكنني استخدام GPT 5.2 للمهام اليومية وCodex 5.3 للبرمجة؟">
    نعم. اجعل أحدهما افتراضيًا وبدّل حسب الحاجة:

    - **تبديل سريع (لكل جلسة):** ‏`/model gpt-5.4` للمهام اليومية، و`/model openai-codex/gpt-5.4` للبرمجة باستخدام Codex OAuth.
    - **افتراضي + تبديل:** اضبط `agents.defaults.model.primary` على `openai/gpt-5.4`، ثم بدّل إلى `openai-codex/gpt-5.4` عند البرمجة (أو بالعكس).
    - **الوكلاء الفرعيون:** وجّه مهام البرمجة إلى وكلاء فرعيين لديهم نموذج افتراضي مختلف.

    راجع [النماذج](/ar/concepts/models) و[Slash commands](/ar/tools/slash-commands).

  </Accordion>

  <Accordion title="كيف أضبط fast mode لـ GPT 5.4؟">
    استخدم إما تبديلًا لكل جلسة أو افتراضيًا في الإعدادات:

    - **لكل جلسة:** أرسل `/fast on` بينما تستخدم الجلسة `openai/gpt-5.4` أو `openai-codex/gpt-5.4`.
    - **افتراضي لكل نموذج:** اضبط `agents.defaults.models["openai/gpt-5.4"].params.fastMode` على `true`.
    - **Codex OAuth أيضًا:** إذا كنت تستخدم أيضًا `openai-codex/gpt-5.4`، فاضبط العلم نفسه هناك.

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

    بالنسبة إلى OpenAI، يرتبط fast mode بالقيمة `service_tier = "priority"` على طلبات Responses الأصلية المدعومة. وتتغلب إعدادات الجلسة `/fast` على افتراضيات الإعدادات.

    راجع [التفكير وfast mode](/ar/tools/thinking) و[fast mode في OpenAI](/ar/providers/openai#openai-fast-mode).

  </Accordion>

  <Accordion title='لماذا أرى "Model ... is not allowed" ثم لا يوجد رد؟'>
    إذا كانت `agents.defaults.models` مضبوطة، فإنها تصبح **allowlist** لكل من `/model` وأي
    تجاوزات للجلسة. اختيار نموذج غير موجود في تلك القائمة يعيد:

    ```
    Model "provider/model" is not allowed. Use /model to list available models.
    ```

    يُعاد هذا الخطأ **بدلًا من** الرد العادي. الحل: أضف النموذج إلى
    `agents.defaults.models`، أو أزل allowlist، أو اختر نموذجًا من `/model list`.

  </Accordion>

  <Accordion title='لماذا أرى "Unknown model: minimax/MiniMax-M2.7"؟'>
    هذا يعني أن **الموفّر غير مهيأ** (لم يُعثر على إعداد موفّر MiniMax أو ملف
    تعريف مصادقة)، لذلك لا يمكن حل النموذج.

    قائمة تحقق للإصلاح:

    1. قم بالترقية إلى إصدار OpenClaw حديث (أو شغّل من المصدر `main`)، ثم أعد تشغيل gateway.
    2. تأكد من أن MiniMax مهيأة (المعالج أو JSON)، أو من وجود مصادقة MiniMax
       في env/ملفات تعريف المصادقة بحيث يمكن حقن الموفّر المطابق
       (`MINIMAX_API_KEY` من أجل `minimax`، و`MINIMAX_OAUTH_TOKEN` أو MiniMax OAuth
       مخزّن من أجل `minimax-portal`).
    3. استخدم model id الدقيق (حساس لحالة الأحرف) لمسار المصادقة لديك:
       `minimax/MiniMax-M2.7` أو `minimax/MiniMax-M2.7-highspeed` من أجل إعداد
       API-key، أو `minimax-portal/MiniMax-M2.7` /
       `minimax-portal/MiniMax-M2.7-highspeed` من أجل إعداد OAuth.
    4. شغّل:

       ```bash
       openclaw models list
       ```

       واختر من القائمة (أو `/model list` في الدردشة).

    راجع [MiniMax](/ar/providers/minimax) و[النماذج](/ar/concepts/models).

  </Accordion>

  <Accordion title="هل يمكنني استخدام MiniMax كافتراضي وOpenAI للمهام المعقدة؟">
    نعم. استخدم **MiniMax كافتراضي** وبدّل النماذج **لكل جلسة** عند الحاجة.
    النماذج الاحتياطية مخصصة **للأخطاء**، وليس "للمهام الصعبة"، لذا استخدم `/model` أو وكيلًا منفصلًا.

    **الخيار A: التبديل لكل جلسة**

    ```json5
    {
      env: { MINIMAX_API_KEY: "sk-...", OPENAI_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "minimax/MiniMax-M2.7" },
          models: {
            "minimax/MiniMax-M2.7": { alias: "minimax" },
            "openai/gpt-5.4": { alias: "gpt" },
          },
        },
      },
    }
    ```

    ثم:

    ```
    /model gpt
    ```

    **الخيار B: وكلاء منفصلون**

    - افتراضي الوكيل A: ‏MiniMax
    - افتراضي الوكيل B: ‏OpenAI
    - وجّه حسب الوكيل أو استخدم `/agent` للتبديل

    الوثائق: [النماذج](/ar/concepts/models)، [التوجيه متعدد الوكلاء](/ar/concepts/multi-agent)، [MiniMax](/ar/providers/minimax)، [OpenAI](/ar/providers/openai).

  </Accordion>

  <Accordion title="هل opus / sonnet / gpt اختصارات مضمّنة؟">
    نعم. يشحن OpenClaw بعض الاختصارات الافتراضية (تُطبّق فقط عندما يكون النموذج موجودًا في `agents.defaults.models`):

    - `opus` → `anthropic/claude-opus-4-6`
    - `sonnet` → `anthropic/claude-sonnet-4-6`
    - `gpt` → `openai/gpt-5.4`
    - `gpt-mini` → `openai/gpt-5.4