---
read_when:
    - الإجابة عن أسئلة الدعم الشائعة المتعلقة بالإعداد أو التثبيت أو الإعداد الأولي أو وقت التشغيل
    - فرز المشكلات التي يبلّغ عنها المستخدمون قبل الانتقال إلى تصحيح أعمق للأخطاء
summary: الأسئلة الشائعة حول إعداد OpenClaw وتكوينه واستخدامه
title: الأسئلة الشائعة
x-i18n:
    generated_at: "2026-04-05T12:50:36Z"
    model: gpt-5.4
    provider: openai
    source_hash: 0f71dc12f60aceaa1d095aaa4887d59ecf2a53e349d10a3e2f60e464ae48aff6
    source_path: help/faq.md
    workflow: 15
---

# الأسئلة الشائعة

إجابات سريعة مع خطوات أعمق لاستكشاف الأخطاء وإصلاحها في البيئات الواقعية (التطوير المحلي، وVPS، والوكلاء المتعددون، وOAuth/API keys، وبدائل النماذج الاحتياطية). لتشخيصات وقت التشغيل، راجع [استكشاف الأخطاء وإصلاحها](/gateway/troubleshooting). وللمرجع الكامل للإعداد، راجع [الإعداد](/gateway/configuration).

## أول 60 ثانية إذا كان هناك شيء معطّل

1. **الحالة السريعة (أول فحص)**

   ```bash
   openclaw status
   ```

   ملخص محلي سريع: نظام التشغيل + التحديث، وقابلية الوصول إلى gateway/service، والوكلاء/الجلسات، وإعدادات المزوّد + مشكلات وقت التشغيل (عندما يكون gateway قابلاً للوصول).

2. **تقرير قابل للمشاركة بأمان**

   ```bash
   openclaw status --all
   ```

   تشخيص للقراءة فقط مع ذيل السجل (مع تنقيح الرموز).

3. **حالة daemon + المنفذ**

   ```bash
   openclaw gateway status
   ```

   يعرض وقت تشغيل supervisor مقابل قابلية الوصول عبر RPC، وURL هدف الفحص، وأي إعداد استخدمته الخدمة على الأرجح.

4. **مجسات عميقة**

   ```bash
   openclaw status --deep
   ```

   يشغّل مجس سلامة حيًا لـ gateway، بما في ذلك مجسات القنوات عند دعمها
   (يتطلب gateway قابلاً للوصول). راجع [Health](/gateway/health).

5. **تتبّع أحدث سجل**

   ```bash
   openclaw logs --follow
   ```

   إذا كان RPC متوقفًا، فارجع إلى:

   ```bash
   tail -f "$(ls -t /tmp/openclaw/openclaw-*.log | head -1)"
   ```

   سجلات الملفات منفصلة عن سجلات الخدمة؛ راجع [التسجيل](/logging) و[استكشاف الأخطاء وإصلاحها](/gateway/troubleshooting).

6. **تشغيل doctor (الإصلاحات)**

   ```bash
   openclaw doctor
   ```

   يصلح/يرحّل الإعداد والحالة ويشغّل فحوصات السلامة. راجع [Doctor](/gateway/doctor).

7. **لقطة gateway**

   ```bash
   openclaw health --json
   openclaw health --verbose   # يعرض URL الهدف + مسار الإعداد عند الأخطاء
   ```

   يطلب من gateway الجاري تشغيله لقطة كاملة (WS فقط). راجع [Health](/gateway/health).

## البدء السريع وإعداد التشغيل الأول

<AccordionGroup>
  <Accordion title="أنا عالق، ما أسرع طريقة للخروج من هذا المأزق؟">
    استخدم وكيل AI محليًا يمكنه **رؤية جهازك**. فهذا أكثر فاعلية بكثير من السؤال
    في Discord، لأن معظم حالات "أنا عالق" تكون **مشكلات إعداد أو بيئة محلية**
    لا يستطيع المساعدون عن بُعد فحصها.

    - **Claude Code**: [https://www.anthropic.com/claude-code/](https://www.anthropic.com/claude-code/)
    - **OpenAI Codex**: [https://openai.com/codex/](https://openai.com/codex/)

    يمكن لهذه الأدوات قراءة المستودع، وتشغيل الأوامر، وفحص السجلات، والمساعدة في إصلاح
    الإعداد على مستوى جهازك (PATH، والخدمات، والأذونات، وملفات المصادقة). امنحها
    **النسخة المصدرية الكاملة** عبر التثبيت القابل للتعديل (git):

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    يثبّت هذا OpenClaw **من نسخة git checkout**، بحيث يستطيع الوكيل قراءة الكود + الوثائق
    والاستدلال على الإصدار الدقيق الذي تستخدمه. ويمكنك دائمًا العودة لاحقًا إلى الإصدار المستقر
    بإعادة تشغيل المثبّت من دون `--install-method git`.

    نصيحة: اطلب من الوكيل أن **يخطط ويشرف** على الإصلاح (خطوة بخطوة)، ثم ينفّذ فقط
    الأوامر الضرورية. فهذا يبقي التغييرات صغيرة وأسهل في التدقيق.

    إذا اكتشفت عيبًا حقيقيًا أو إصلاحًا، فالرجاء فتح مشكلة على GitHub أو إرسال PR:
    [https://github.com/openclaw/openclaw/issues](https://github.com/openclaw/openclaw/issues)
    [https://github.com/openclaw/openclaw/pulls](https://github.com/openclaw/openclaw/pulls)

    ابدأ بهذه الأوامر (وشارك المخرجات عند طلب المساعدة):

    ```bash
    openclaw status
    openclaw models status
    openclaw doctor
    ```

    ما الذي تفعله:

    - `openclaw status`: لقطة سريعة عن سلامة gateway/الوكيل + الإعداد الأساسي.
    - `openclaw models status`: يفحص مصادقة المزوّد + توفر النماذج.
    - `openclaw doctor`: يتحقق من مشكلات الإعداد/الحالة الشائعة ويصلحها.

    فحوصات CLI مفيدة أخرى: `openclaw status --all`, `openclaw logs --follow`,
    `openclaw gateway status`, `openclaw health --verbose`.

    حلقة التصحيح السريعة: [أول 60 ثانية إذا كان هناك شيء معطّل](#first-60-seconds-if-something-is-broken).
    وثائق التثبيت: [Install](/install)، [Installer flags](/install/installer)، [Updating](/install/updating).

  </Accordion>

  <Accordion title="Heartbeat يستمر في التخطي. ماذا تعني أسباب التخطي؟">
    أسباب تخطي heartbeat الشائعة:

    - `quiet-hours`: خارج نافذة الساعات النشطة المكوّنة
    - `empty-heartbeat-file`: الملف `HEARTBEAT.md` موجود لكنه يحتوي فقط على هيكل فارغ/عناوين فقط
    - `no-tasks-due`: وضع مهام `HEARTBEAT.md` مفعّل لكن لم يحن وقت أي من فواصل المهام بعد
    - `alerts-disabled`: كل إعدادات ظهور heartbeat معطلة (`showOk`, `showAlerts`, و`useIndicator` كلها متوقفة)

    في وضع المهام، لا تُقدَّم الطوابع الزمنية المستحقة إلا بعد اكتمال تشغيل heartbeat
    فعلي. أما التشغيلات المتخطاة فلا تُعلِّم المهام على أنها مكتملة.

    الوثائق: [Heartbeat](/gateway/heartbeat)، [Automation & Tasks](/automation).

  </Accordion>

  <Accordion title="الطريقة الموصى بها لتثبيت OpenClaw وإعداده">
    يوصي المستودع بالتشغيل من المصدر واستخدام الإعداد الأولي:

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash
    openclaw onboard --install-daemon
    ```

    يمكن للمعالج أيضًا بناء أصول UI تلقائيًا. وبعد الإعداد الأولي، ستشغّل عادةً Gateway على المنفذ **18789**.

    من المصدر (للمساهمين/المطورين):

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    pnpm ui:build # يثبّت تبعيات UI تلقائيًا في أول تشغيل
    openclaw onboard
    ```

    إذا لم يكن لديك تثبيت عام بعد، فشغّله عبر `pnpm openclaw onboard`.

  </Accordion>

  <Accordion title="كيف أفتح dashboard بعد الإعداد الأولي؟">
    يفتح المعالج متصفحك بعنوان dashboard نظيف (من دون token في الرابط) مباشرة بعد الإعداد الأولي ويطبع الرابط أيضًا في الملخص. اترك ذلك التبويب مفتوحًا؛ وإذا لم يُفتح، فانسخ/ألصق URL المطبوع على الجهاز نفسه.
  </Accordion>

  <Accordion title="كيف أوثّق dashboard على localhost مقابل الاستخدام البعيد؟">
    **Localhost (الجهاز نفسه):**

    - افتح `http://127.0.0.1:18789/`.
    - إذا طلب مصادقة shared-secret، فألصق token أو password المكوَّنين في إعدادات Control UI.
    - مصدر token: ‏`gateway.auth.token` ‏(أو `OPENCLAW_GATEWAY_TOKEN`).
    - مصدر password: ‏`gateway.auth.password` ‏(أو `OPENCLAW_GATEWAY_PASSWORD`).
    - إذا لم يكن shared secret مضبوطًا بعد، فأنشئ token عبر `openclaw doctor --generate-gateway-token`.

    **ليس على localhost:**

    - **Tailscale Serve** ‏(موصى به): أبقِ الربط على loopback، وشغّل `openclaw gateway --tailscale serve`، ثم افتح `https://<magicdns>/`. إذا كانت `gateway.auth.allowTailscale` تساوي `true`، فستلبّي ترويسات الهوية مصادقة Control UI/WebSocket (من دون لصق shared secret، بافتراض موثوقية مضيف gateway)؛ أما HTTP APIs فتظل تتطلب مصادقة shared-secret ما لم تستخدم عمدًا وضع private-ingress `none` أو مصادقة HTTP الخاصة بـ trusted-proxy.
      تُسلسل محاولات مصادقة Serve السيئة المتزامنة من العميل نفسه قبل أن يسجّل محدِّد المحاولات الفاشلة المصادقة، لذا قد تُظهر المحاولة السيئة الثانية بالفعل الرسالة `retry later`.
    - **ربط tailnet**: شغّل `openclaw gateway --bind tailnet --token "<token>"` ‏(أو اضبط password auth)، وافتح `http://<tailscale-ip>:18789/`، ثم ألصق shared secret المطابق في إعدادات dashboard.
    - **وكيل عكسي مدرك للهوية**: أبقِ Gateway خلف trusted proxy غير معتمد على loopback، واضبط `gateway.auth.mode: "trusted-proxy"`، ثم افتح URL الخاص بالوكيل.
    - **SSH tunnel**: ‏`ssh -N -L 18789:127.0.0.1:18789 user@host` ثم افتح `http://127.0.0.1:18789/`. وتظل مصادقة shared-secret مطبقة عبر tunnel؛ ألصق token أو password المكوّنين إذا طُلب منك.

    راجع [Dashboard](/web/dashboard) و[Web surfaces](/web) لمعرفة أوضاع الربط وتفاصيل المصادقة.

  </Accordion>

  <Accordion title="لماذا يوجد إعدادان لموافقات exec بالنسبة إلى موافقات الدردشة؟">
    هما يتحكمان في طبقتين مختلفتين:

    - `approvals.exec`: يمرّر مطالبات الموافقة إلى وجهات الدردشة
    - `channels.<channel>.execApprovals`: يجعل تلك القناة تعمل كعميل موافقة أصلي لموافقات exec

    تظل سياسة exec الخاصة بالمضيف هي بوابة الموافقة الحقيقية. أما إعداد الدردشة فيتحكم فقط في مكان
    ظهور مطالبات الموافقة وكيف يمكن للناس الرد.

    في معظم البيئات **لا** تحتاج إلى الاثنين معًا:

    - إذا كانت الدردشة تدعم الأوامر والردود أصلًا، فإن `/approve` في الدردشة نفسها يعمل عبر المسار المشترك.
    - إذا كانت قناة أصلية مدعومة تستطيع استنتاج الموافقين بأمان، فإن OpenClaw يفعّل الآن تلقائيًا الموافقات الأصلية DM-first عند عدم ضبط `channels.<channel>.execApprovals.enabled` أو ضبطها على `"auto"`.
    - عندما تتوفر بطاقات/أزرار الموافقة الأصلية، تكون واجهة القناة الأصلية هي المسار الأساسي؛ ولا ينبغي للوكيل تضمين أمر `/approve` يدوي إلا إذا كانت نتيجة الأداة تقول إن موافقات الدردشة غير متاحة أو أن الموافقة اليدوية هي المسار الوحيد.
    - استخدم `approvals.exec` فقط عندما يجب أيضًا تمرير المطالبات إلى دردشات أخرى أو غرف عمليات صريحة.
    - استخدم `channels.<channel>.execApprovals.target: "channel"` أو `"both"` فقط عندما تريد صراحة نشر مطالبات الموافقة في الغرفة/الموضوع الأصليين.
    - موافقات plugin منفصلة مرة أخرى: فهي تستخدم `/approve` في الدردشة نفسها افتراضيًا، وتمريرًا اختياريًا عبر `approvals.plugin`، وفقط بعض القنوات الأصلية تبقي معالجة plugin-approval-native فوق ذلك.

    باختصار: التمرير مخصص للتوجيه، أما إعداد العميل الأصلي فمخصص لتجربة استخدام أغنى خاصة بالقناة.
    راجع [Exec Approvals](/tools/exec-approvals).

  </Accordion>

  <Accordion title="ما وقت التشغيل الذي أحتاج إليه؟">
    Node **>= 22** مطلوب. و`pnpm` موصى به. أما Bun فـ **غير موصى به** بالنسبة إلى Gateway.
  </Accordion>

  <Accordion title="هل يعمل على Raspberry Pi؟">
    نعم. Gateway خفيف - تذكر الوثائق أن **512MB-1GB RAM**، و**نواة واحدة**، وحوالي **500MB**
    من القرص تكفي للاستخدام الشخصي، وتشير إلى أن **Raspberry Pi 4 يمكنه تشغيله**.

    إذا كنت تريد هامشًا إضافيًا (للسجلات، والوسائط، والخدمات الأخرى)، فـ **2GB موصى بها**، لكنها
    ليست حدًا أدنى صارمًا.

    نصيحة: يمكن لجهاز Pi/VPS صغير استضافة Gateway، ويمكنك إقران **nodes** على حاسوبك المحمول/هاتفك
    للوصول المحلي إلى الشاشة/الكاميرا/canvas أو تنفيذ الأوامر. راجع [Nodes](/nodes).

  </Accordion>

  <Accordion title="هل لديك نصائح لتثبيتات Raspberry Pi؟">
    باختصار: يعمل، لكن توقع بعض الحواف الخشنة.

    - استخدم نظام تشغيل **64-bit** وحافظ على Node >= 22.
    - فضّل **التثبيت القابل للتعديل (git)** حتى تتمكن من رؤية السجلات والتحديث بسرعة.
    - ابدأ من دون قنوات/Skills، ثم أضفها واحدة تلو الأخرى.
    - إذا واجهت مشكلات غريبة في الثنائيات، فعادةً ما تكون مشكلة **توافق ARM**.

    الوثائق: [Linux](/platforms/linux)، [Install](/install).

  </Accordion>

  <Accordion title="إنه عالق عند wake up my friend / الإعداد الأولي لا يكتمل. ماذا الآن؟">
    تعتمد تلك الشاشة على أن يكون Gateway قابلاً للوصول وموثَّقًا. كما يرسل TUI أيضًا
    "Wake up, my friend!" تلقائيًا عند أول hatch. وإذا رأيت هذا السطر **من دون رد**
    وبقيت tokens عند 0، فهذا يعني أن الوكيل لم يعمل مطلقًا.

    1. أعد تشغيل Gateway:

    ```bash
    openclaw gateway restart
    ```

    2. افحص الحالة + المصادقة:

    ```bash
    openclaw status
    openclaw models status
    openclaw logs --follow
    ```

    3. إذا ظل الأمر معلقًا، فشغّل:

    ```bash
    openclaw doctor
    ```

    إذا كان Gateway بعيدًا، فتأكد من أن tunnel/Tailscale يعمل وأن UI
    تشير إلى Gateway الصحيح. راجع [Remote access](/gateway/remote).

  </Accordion>

  <Accordion title="هل يمكنني نقل إعدادي إلى جهاز جديد (Mac mini) من دون إعادة الإعداد الأولي؟">
    نعم. انسخ **دليل الحالة** و**مساحة العمل**، ثم شغّل Doctor مرة واحدة. هذا
    يبقي البوت "كما هو تمامًا" (الذاكرة، وسجل الجلسات، والمصادقة، وحالة القنوات)
    طالما أنك تنسخ **الموقعين** معًا:

    1. ثبّت OpenClaw على الجهاز الجديد.
    2. انسخ `$OPENCLAW_STATE_DIR` ‏(الافتراضي: `~/.openclaw`) من الجهاز القديم.
    3. انسخ مساحة العمل الخاصة بك (الافتراضي: `~/.openclaw/workspace`).
    4. شغّل `openclaw doctor` وأعد تشغيل خدمة Gateway.

    سيحافظ ذلك على الإعداد، وملفات تعريف المصادقة، وبيانات اعتماد WhatsApp، والجلسات، والذاكرة. وإذا كنت في
    الوضع البعيد، فتذكّر أن مضيف gateway هو من يملك مخزن الجلسات ومساحة العمل.

    **مهم:** إذا كنت فقط تُجري commit/push لمساحة العمل إلى GitHub، فأنت
    تحتفظ بنسخة احتياطية من **الذاكرة + ملفات bootstrap**، لكن **ليس** من سجل الجلسات أو المصادقة. فهذه تعيش
    تحت `~/.openclaw/` ‏(مثل `~/.openclaw/agents/<agentId>/sessions/`).

    ذو صلة: [Migrating](/install/migrating)، [أين تعيش الأشياء على القرص](#where-things-live-on-disk)،
    [مساحة عمل الوكيل](/concepts/agent-workspace)، [Doctor](/gateway/doctor)،
    [الوضع البعيد](/gateway/remote).

  </Accordion>

  <Accordion title="أين أرى ما الجديد في أحدث إصدار؟">
    راجع سجل التغييرات على GitHub:
    [https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md)

    أحدث الإدخالات تكون في الأعلى. وإذا كان القسم العلوي موسومًا بـ **Unreleased**، فالقسم التالي المؤرخ
    هو أحدث إصدار مشحون. وتُجمع الإدخالات ضمن **Highlights** و**Changes** و
    **Fixes** (بالإضافة إلى أقسام الوثائق/أخرى عند الحاجة).

  </Accordion>

  <Accordion title="لا أستطيع الوصول إلى docs.openclaw.ai (خطأ SSL)">
    تقوم بعض اتصالات Comcast/Xfinity بحظر `docs.openclaw.ai` بشكل غير صحيح عبر Xfinity
    Advanced Security. عطّله أو أدرج `docs.openclaw.ai` في قائمة السماح، ثم أعد المحاولة.
    يرجى مساعدتنا في رفع هذا الحظر بالإبلاغ هنا: [https://spa.xfinity.com/check_url_status](https://spa.xfinity.com/check_url_status).

    إذا ظل الوصول إلى الموقع غير ممكن، فالوثائق معكوسة على GitHub:
    [https://github.com/openclaw/openclaw/tree/main/docs](https://github.com/openclaw/openclaw/tree/main/docs)

  </Accordion>

  <Accordion title="ما الفرق بين stable وbeta؟">
    إن **Stable** و**beta** هما **npm dist-tags**، وليسا سطرين منفصلين من الكود:

    - `latest` = المستقر
    - `beta` = إصدار مبكر للاختبار

    عادةً، يصل الإصدار المستقر إلى **beta** أولًا، ثم تنقل خطوة ترقية
    صريحة ذلك الإصدار نفسه إلى `latest`. ويمكن للمحافظين أيضًا
    النشر مباشرة إلى `latest` عند الحاجة. ولهذا قد يشير beta وstable
    إلى **الإصدار نفسه** بعد الترقية.

    اطلع على ما تغيّر:
    [https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md)

    وللحصول على أوامر التثبيت المختصرة والفارق بين beta وdev، راجع العنصر المطوي أدناه.

  </Accordion>

  <Accordion title="كيف أثبّت إصدار beta وما الفرق بين beta وdev؟">
    **Beta** هي npm dist-tag المسماة `beta` ‏(وقد تطابق `latest` بعد الترقية).
    أما **Dev** فهو الرأس المتحرك للفرع `main` ‏(git)؛ وعند نشره يستخدم npm dist-tag باسم `dev`.

    أوامر مختصرة (macOS/Linux):

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --beta
    ```

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    مثبت Windows ‏(PowerShell):
    [https://openclaw.ai/install.ps1](https://openclaw.ai/install.ps1)

    مزيد من التفاصيل: [Development channels](/install/development-channels) و[Installer flags](/install/installer).

  </Accordion>

  <Accordion title="كيف أجرّب أحدث التحديثات؟">
    يوجد خياران:

    1. **قناة Dev ‏(نسخة git checkout):**

    ```bash
    openclaw update --channel dev
    ```

    سيحوّلك هذا إلى الفرع `main` ويحدّث من المصدر.

    2. **تثبيت قابل للتعديل (من موقع المثبّت):**

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    يمنحك هذا مستودعًا محليًا يمكنك تعديله، ثم تحديثه عبر git.

    إذا كنت تفضّل clone نظيفًا يدويًا، فاستخدم:

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    ```

    الوثائق: [Update](/cli/update)، [Development channels](/install/development-channels)،
    [Install](/install).

  </Accordion>

  <Accordion title="كم يستغرق التثبيت والإعداد الأولي عادة؟">
    تقدير تقريبي:

    - **التثبيت:** 2-5 دقائق
    - **الإعداد الأولي:** 5-15 دقيقة حسب عدد القنوات/النماذج التي تضبطها

    إذا تعلّق، فاستخدم [Installer stuck](#quick-start-and-first-run-setup)
    وحلقة التصحيح السريعة في [أنا عالق](#quick-start-and-first-run-setup).

  </Accordion>

  <Accordion title="المثبت عالق؟ كيف أحصل على ملاحظات أكثر؟">
    أعد تشغيل المثبّت مع **مخرجات مفصلة**:

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --verbose
    ```

    تثبيت beta مع verbose:

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --beta --verbose
    ```

    لتثبيت قابل للتعديل (git):

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git --verbose
    ```

    ما يعادل ذلك على Windows ‏(PowerShell):

    ```powershell
    # لا يحتوي install.ps1 على علامة -Verbose مخصصة بعد.
    Set-PSDebug -Trace 1
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -NoOnboard
    Set-PSDebug -Trace 0
    ```

    مزيد من الخيارات: [Installer flags](/install/installer).

  </Accordion>

  <Accordion title="تثبيت Windows يقول إن git غير موجود أو إن openclaw غير معروف">
    توجد مشكلتان شائعتان في Windows:

    **1) خطأ npm ‏spawn git / git not found**

    - ثبّت **Git for Windows** وتأكد من وجود `git` في PATH.
    - أغلق PowerShell وأعد فتحه، ثم أعد تشغيل المثبّت.

    **2) openclaw is not recognized بعد التثبيت**

    - مجلد npm global bin غير موجود في PATH.
    - افحص المسار:

      ```powershell
      npm config get prefix
      ```

    - أضف ذلك الدليل إلى PATH الخاص بالمستخدم (من دون لاحقة `\bin` على Windows؛ في معظم الأنظمة يكون `%AppData%\npm`).
    - أغلق PowerShell وأعد فتحه بعد تحديث PATH.

    إذا كنت تريد أسلس إعداد على Windows، فاستخدم **WSL2** بدل Windows الأصلي.
    الوثائق: [Windows](/platforms/windows).

  </Accordion>

  <Accordion title="مخرجات exec على Windows تعرض نصًا صينيًا مشوّهًا - ماذا أفعل؟">
    يكون السبب عادةً عدم تطابق في code page داخل الطرفية على Windows الأصلي.

    الأعراض:

    - تعرض مخرجات `system.run`/`exec` اللغة الصينية بشكل مشوّه
    - يظهر الأمر نفسه بشكل صحيح في ملف تعريف طرفية آخر

    حل سريع في PowerShell:

    ```powershell
    chcp 65001
    [Console]::InputEncoding = [System.Text.UTF8Encoding]::new($false)
    [Console]::OutputEncoding = [System.Text.UTF8Encoding]::new($false)
    $OutputEncoding = [System.Text.UTF8Encoding]::new($false)
    ```

    ثم أعد تشغيل Gateway وأعد تجربة الأمر:

    ```powershell
    openclaw gateway restart
    ```

    إذا ظل يمكنك إعادة إنتاج ذلك في أحدث OpenClaw، فتابعه/أبلِغ عنه في:

    - [Issue #30640](https://github.com/openclaw/openclaw/issues/30640)

  </Accordion>

  <Accordion title="الوثائق لم تجب عن سؤالي - كيف أحصل على إجابة أفضل؟">
    استخدم **التثبيت القابل للتعديل (git)** حتى تكون لديك الشيفرة المصدرية الكاملة والوثائق محليًا، ثم اسأل
    البوت الخاص بك (أو Claude/Codex) _من داخل ذلك المجلد_ حتى يتمكن من قراءة المستودع والإجابة بدقة.

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    مزيد من التفاصيل: [Install](/install) و[Installer flags](/install/installer).

  </Accordion>

  <Accordion title="كيف أُثبّت OpenClaw على Linux؟">
    الإجابة القصيرة: اتبع دليل Linux، ثم شغّل الإعداد الأولي.

    - المسار السريع لـ Linux + تثبيت الخدمة: [Linux](/platforms/linux).
    - الشرح الكامل: [Getting Started](/ar/start/getting-started).
    - المثبت + التحديثات: [Install & updates](/install/updating).

  </Accordion>

  <Accordion title="كيف أُثبّت OpenClaw على VPS؟">
    أي VPS يعمل بنظام Linux مناسب. ثبّت OpenClaw على الخادم، ثم استخدم SSH/Tailscale للوصول إلى Gateway.

    الأدلة: [exe.dev](/install/exe-dev)، [Hetzner](/install/hetzner)، [Fly.io](/install/fly).
    الوصول البعيد: [Gateway remote](/gateway/remote).

  </Accordion>

  <Accordion title="أين توجد أدلة التثبيت السحابي/VPS؟">
    نحتفظ **بمركز استضافة** يضم المزودين الشائعين. اختر واحدًا واتبع الدليل:

    - [استضافة VPS](/vps) ‏(كل المزودين في مكان واحد)
    - [Fly.io](/install/fly)
    - [Hetzner](/install/hetzner)
    - [exe.dev](/install/exe-dev)

    كيف يعمل ذلك في السحابة: يعمل **Gateway على الخادم**، وتصل إليه
    من حاسوبك المحمول/هاتفك عبر Control UI ‏(أو Tailscale/SSH). وتعيش حالتك + مساحة العمل
    على الخادم، لذا تعامل مع المضيف بوصفه مصدر الحقيقة وقم بعمل نسخة احتياطية منه.

    يمكنك إقران **nodes** ‏(Mac/iOS/Android/headless) بذلك Gateway السحابي للوصول إلى
    الشاشة/الكاميرا/canvas المحلية أو تشغيل الأوامر على حاسوبك المحمول مع إبقاء
    Gateway في السحابة.

    المركز: [Platforms](/platforms). الوصول البعيد: [Gateway remote](/gateway/remote).
    العقد: [Nodes](/nodes)، [Nodes CLI](/cli/nodes).

  </Accordion>

  <Accordion title="هل يمكنني أن أطلب من OpenClaw أن يحدّث نفسه؟">
    باختصار: **ممكن، لكنه غير موصى به**. فعملية التحديث قد تعيد تشغيل
    Gateway ‏(مما يسقط الجلسة النشطة)، وقد تحتاج إلى git checkout نظيف، وقد
    تطلب تأكيدًا. والأكثر أمانًا: شغّل التحديثات من shell بصفتك المشغّل.

    استخدم CLI:

    ```bash
    openclaw update
    openclaw update status
    openclaw update --channel stable|beta|dev
    openclaw update --tag <dist-tag|version>
    openclaw update --no-restart
    ```

    وإذا اضطررت إلى الأتمتة من داخل وكيل:

    ```bash
    openclaw update --yes --no-restart
    openclaw gateway restart
    ```

    الوثائق: [Update](/cli/update)، [Updating](/install/updating).

  </Accordion>

  <Accordion title="ماذا يفعل الإعداد الأولي فعليًا؟">
    `openclaw onboard` هو مسار الإعداد الموصى به. وفي **الوضع المحلي** فإنه يوجّهك عبر:

    - **إعداد النموذج/المصادقة** ‏(OAuth للمزوّد، وإعادة استخدام Claude CLI، ودعم API keys، بالإضافة إلى خيارات النماذج المحلية مثل LM Studio)
    - موقع **مساحة العمل** + ملفات bootstrap
    - **إعدادات Gateway** ‏(bind/port/auth/tailscale)
    - **القنوات** ‏(WhatsApp، وTelegram، وDiscord، وMattermost، وSignal، وiMessage، بالإضافة إلى مكونات قنوات إضافية مضمّنة مثل QQ Bot)
    - **تثبيت daemon** ‏(LaunchAgent على macOS؛ ووحدة systemd للمستخدم على Linux/WSL2)
    - **فحوصات السلامة** واختيار **Skills**

    كما يحذرك أيضًا إذا كان النموذج المكوّن غير معروف أو تنقصه المصادقة.

  </Accordion>

  <Accordion title="هل أحتاج إلى اشتراك Claude أو OpenAI لتشغيل هذا؟">
    لا. يمكنك تشغيل OpenClaw باستخدام **API keys** ‏(Anthropic/OpenAI/وغيرها) أو
    باستخدام **نماذج محلية فقط** بحيث تبقى بياناتك على جهازك. والاشتراكات (Claude
    Pro/Max أو OpenAI Codex) هي طرق اختيارية فقط للمصادقة على تلك المزوّدات.

    نعتقد أن fallback الخاص بـ Claude Code CLI مسموح على الأرجح
    لعمليات الأتمتة المحلية التي يديرها المستخدم استنادًا إلى وثائق CLI العامة الخاصة بـ Anthropic. ومع ذلك،
    فإن سياسة Anthropic الخاصة بأحزمة الأطراف الثالثة تخلق غموضًا كافيًا
    حول الاستخدام المدعوم بالاشتراك في المنتجات الخارجية بحيث لا نوصي به
    للإنتاج. كما أبلغت Anthropic مستخدمي OpenClaw في **4 أبريل 2026
    الساعة 12:00 ظهرًا بتوقيت PT / 8:00 مساءً بتوقيت BST** أن مسار
    تسجيل الدخول إلى Claude داخل **OpenClaw** يُعد استخدامًا لحزمة طرف ثالث
    ويتطلب الآن **Extra Usage**
    تُحتسب بشكل منفصل عن الاشتراك. أما OpenAI Codex OAuth فهو مدعوم صراحةً
    للأدوات الخارجية مثل OpenClaw.

    يدعم OpenClaw أيضًا خيارات مستضافة أخرى بنمط الاشتراك بما في ذلك
    **Qwen Cloud Coding Plan**، و**MiniMax Coding Plan**، و
    **Z.AI / GLM Coding Plan**.

    الوثائق: [Anthropic](/providers/anthropic)، [OpenAI](/providers/openai)،
    [Qwen Cloud](/providers/qwen)،
    [MiniMax](/providers/minimax)، [GLM Models](/providers/glm)،
    [النماذج المحلية](/gateway/local-models)، [النماذج](/concepts/models).

  </Accordion>

  <Accordion title="هل يمكنني استخدام اشتراك Claude Max من دون API key؟">
    نعم، عبر تسجيل دخول **Claude CLI** محلي على مضيف gateway.

    لا تتضمن اشتراكات Claude Pro/Max **API key**، لذا فإن
    إعادة استخدام Claude CLI هي المسار الاحتياطي المحلي في OpenClaw. ونعتقد أن fallback الخاص بـ Claude Code CLI
    مسموح على الأرجح لعمليات الأتمتة المحلية التي يديرها المستخدم استنادًا إلى
    وثائق CLI العامة من Anthropic. ومع ذلك، فإن سياسة Anthropic الخاصة بأحزمة الأطراف الثالثة
    تخلق غموضًا كافيًا حول الاستخدام المدعوم بالاشتراك في المنتجات الخارجية
    بحيث لا نوصي به للإنتاج. ونوصي
    باستخدام Anthropic API keys بدلًا من ذلك.

  </Accordion>

  <Accordion title="هل تدعمون مصادقة اشتراك Claude (Claude Pro أو Max)؟">
    نعم. أعد استخدام تسجيل دخول **Claude CLI** المحلي على مضيف gateway باستخدام `openclaw models auth login --provider anthropic --method cli --set-default`.

    كما أصبح Anthropic setup-token متاحًا مرة أخرى كمسار OpenClaw قديم/يدوي. وما يزال إشعار الفوترة الخاص بـ Anthropic والمخصص لـ OpenClaw مطبقًا هناك، لذا استخدمه مع توقع أن Anthropic تتطلب **Extra Usage**. راجع [Anthropic](/providers/anthropic) و[OAuth](/concepts/oauth).

    مهم: نعتقد أن fallback الخاص بـ Claude Code CLI مسموح على الأرجح للاستخدام المحلي
    المُدار من المستخدم استنادًا إلى وثائق CLI العامة لـ Anthropic. ومع ذلك،
    فإن سياسة Anthropic الخاصة بأحزمة الأطراف الثالثة تخلق غموضًا كافيًا
    حول الاستخدام المدعوم بالاشتراك في المنتجات الخارجية بحيث لا نوصي به
    للإنتاج. كما أخبرت Anthropic مستخدمي OpenClaw في **4 أبريل 2026 عند
    الساعة 12:00 ظهرًا بتوقيت PT / 8:00 مساءً بتوقيت BST** أن مسار
    تسجيل الدخول إلى Claude داخل **OpenClaw** يتطلب
    **Extra Usage** تُحتسب بشكل منفصل عن الاشتراك.

    بالنسبة للإنتاج أو أحمال العمل متعددة المستخدمين، تبقى مصادقة Anthropic API key
    الخيار الأكثر أمانًا والمُوصى به. وإذا كنت تريد خيارات مستضافة أخرى بنمط الاشتراك
    داخل OpenClaw، فراجع [OpenAI](/providers/openai)، و[Qwen / Model
    Cloud](/providers/qwen)، و[MiniMax](/providers/minimax)، و
    [GLM Models](/providers/glm).

  </Accordion>

<a id="why-am-i-seeing-http-429-ratelimiterror-from-anthropic"></a>
<Accordion title="لماذا أرى الخطأ HTTP 429 rate_limit_error من Anthropic؟">
هذا يعني أن **الحصة/حدود المعدل** الخاصة بـ Anthropic قد استُنفدت في النافذة الحالية. إذا كنت
تستخدم **Claude CLI**، فانتظر حتى يُعاد ضبط النافذة أو قم بترقية خطتك. وإذا كنت
تستخدم **Anthropic API key**، فتحقق من Anthropic Console
لمعرفة الاستخدام/الفوترة وارفع الحدود عند الحاجة.

    إذا كانت الرسالة تحديدًا هي:
    `Extra usage is required for long context requests`، فهذا يعني أن الطلب يحاول استخدام
    النسخة التجريبية ذات السياق 1M في Anthropic ‏(`context1m: true`). ولا يعمل ذلك إلا عندما تكون
    بيانات اعتمادك مؤهلة لفوترة long-context ‏(فوتر API key أو
    مسار تسجيل الدخول إلى Claude في OpenClaw مع تفعيل Extra Usage).

    نصيحة: اضبط **نموذجًا احتياطيًا** حتى يتمكن OpenClaw من مواصلة الرد عندما يكون أحد المزوّدين واقعًا تحت rate limit.
    راجع [Models](/cli/models)، و[OAuth](/concepts/oauth)، و
    [/gateway/troubleshooting#anthropic-429-extra-usage-required-for-long-context](/gateway/troubleshooting#anthropic-429-extra-usage-required-for-long-context).

  </Accordion>

  <Accordion title="هل AWS Bedrock مدعوم؟">
    نعم. يحتوي OpenClaw على مزوّد **Amazon Bedrock (Converse)** مضمّن. وعند وجود علامات AWS env، يمكن لـ OpenClaw اكتشاف فهرس Bedrock الخاص بالبث/النص تلقائيًا ودمجه كمزوّد ضمني باسم `amazon-bedrock`؛ وإلا يمكنك تفعيل `plugins.entries.amazon-bedrock.config.discovery.enabled` صراحةً أو إضافة إدخال مزوّد يدوي. راجع [Amazon Bedrock](/providers/bedrock) و[Model providers](/providers/models). وإذا كنت تفضل تدفق مفاتيح مُدارًا، فسيظل الوكيل المتوافق مع OpenAI أمام Bedrock خيارًا صالحًا.
  </Accordion>

  <Accordion title="كيف تعمل مصادقة Codex؟">
    يدعم OpenClaw **OpenAI Code (Codex)** عبر OAuth ‏(تسجيل دخول ChatGPT). ويمكن للإعداد الأولي تشغيل تدفق OAuth وسيضبط النموذج الافتراضي على `openai-codex/gpt-5.4` عند الاقتضاء. راجع [Model providers](/concepts/model-providers) و[Onboarding (CLI)](/ar/start/wizard).
  </Accordion>

  <Accordion title="هل تدعمون مصادقة اشتراك OpenAI (Codex OAuth)؟">
    نعم. يدعم OpenClaw بالكامل **OAuth لاشتراك OpenAI Code (Codex)**.
    وتسمح OpenAI صراحةً باستخدام OAuth للاشتراك في الأدوات/سير العمل الخارجية
    مثل OpenClaw. ويمكن للإعداد الأولي تشغيل تدفق OAuth نيابةً عنك.

    راجع [OAuth](/concepts/oauth)، و[Model providers](/concepts/model-providers)، و[Onboarding (CLI)](/ar/start/wizard).

  </Accordion>

  <Accordion title="كيف أعد Gemini CLI OAuth؟">
    يستخدم Gemini CLI **تدفق مصادقة لمكون إضافي**، وليس client id أو secret داخل `openclaw.json`.

    الخطوات:

    1. ثبّت Gemini CLI محليًا بحيث يكون `gemini` موجودًا في `PATH`
       - Homebrew: ‏`brew install gemini-cli`
       - npm: ‏`npm install -g @google/gemini-cli`
    2. فعّل المكون الإضافي: ‏`openclaw plugins enable google`
    3. سجّل الدخول: ‏`openclaw models auth login --provider google-gemini-cli --set-default`
    4. النموذج الافتراضي بعد تسجيل الدخول: ‏`google-gemini-cli/gemini-3.1-pro-preview`
    5. إذا فشلت الطلبات، فاضبط `GOOGLE_CLOUD_PROJECT` أو `GOOGLE_CLOUD_PROJECT_ID` على مضيف gateway

    يُخزّن هذا رموز OAuth في ملفات تعريف المصادقة على مضيف gateway. التفاصيل: [Model providers](/concepts/model-providers).

  </Accordion>

  <Accordion title="هل النموذج المحلي مناسب للدردشات العادية؟">
    عادةً لا. يحتاج OpenClaw إلى سياق كبير + أمان قوي؛ فالبطاقات الصغيرة تقصّ وتسرّب. وإذا اضطررت، فشغّل **أكبر** بناء نموذج يمكنك تشغيله محليًا (LM Studio) وراجع [/gateway/local-models](/gateway/local-models). فالنماذج الأصغر/المكمّمة تزيد من خطر حقن الموجّهات - راجع [Security](/gateway/security).
  </Accordion>

  <Accordion title="كيف أحافظ على حركة النماذج المستضافة داخل منطقة محددة؟">
    اختر نقاط نهاية مثبتة على منطقة بعينها. يوفّر OpenRouter خيارات مستضافة داخل الولايات المتحدة لـ MiniMax وKimi وGLM؛ اختر المتغير المستضاف في الولايات المتحدة لإبقاء البيانات داخل المنطقة. ويمكنك مع ذلك إدراج Anthropic/OpenAI إلى جانب هذه الخيارات باستخدام `models.mode: "merge"` بحيث تظل البدائل الاحتياطية متاحة مع احترام المزوّد المحدد بحسب المنطقة الذي تختاره.
  </Accordion>

  <Accordion title="هل يجب أن أشتري Mac Mini لتثبيت هذا؟">
    لا. يعمل OpenClaw على macOS أو Linux ‏(وWindows عبر WSL2). Mac mini اختياري -
    فبعض الناس يشترونه كمضيف دائم التشغيل، لكن VPS صغيرًا، أو خادمًا منزليًا، أو جهازًا من فئة Raspberry Pi يكفي أيضًا.

    أنت تحتاج إلى Mac **فقط للأدوات الحصرية لـ macOS**. وبالنسبة إلى iMessage، استخدم [BlueBubbles](/channels/bluebubbles) ‏(موصى به) -
    إذ يعمل خادم BlueBubbles على أي Mac، بينما يمكن أن يعمل Gateway على Linux أو في مكان آخر. وإذا كنت تريد أدوات أخرى حصرية لـ macOS، فشغّل Gateway على Mac أو اقترن بعقدة macOS.

    الوثائق: [BlueBubbles](/channels/bluebubbles)، [Nodes](/nodes)، [Mac remote mode](/platforms/mac/remote).

  </Accordion>

  <Accordion title="هل أحتاج إلى Mac mini لدعم iMessage؟">
    أنت بحاجة إلى **جهاز macOS ما** مسجَّل الدخول إلى Messages. ولا **يجب** أن يكون Mac mini -
    فأي Mac يكفي. **استخدم [BlueBubbles](/channels/bluebubbles)** ‏(موصى به) بالنسبة إلى iMessage - يعمل خادم BlueBubbles على macOS، بينما يمكن أن يعمل Gateway على Linux أو في مكان آخر.

    الإعدادات الشائعة:

    - شغّل Gateway على Linux/VPS، وشغّل خادم BlueBubbles على أي Mac مسجَّل الدخول إلى Messages.
    - شغّل كل شيء على جهاز Mac إذا كنت تريد أبسط إعداد على جهاز واحد.

    الوثائق: [BlueBubbles](/channels/bluebubbles)، [Nodes](/nodes)،
    [Mac remote mode](/platforms/mac/remote).

  </Accordion>

  <Accordion title="إذا اشتريت Mac mini لتشغيل OpenClaw، هل يمكنني توصيله بـ MacBook Pro؟">
    نعم. يمكن لـ **Mac mini تشغيل Gateway**، ويمكن لـ MacBook Pro الاتصال به باعتباره
    **node** ‏(جهازًا مرافقًا). ولا تشغّل العقد Gateway - بل توفّر إمكانات إضافية
    مثل الشاشة/الكاميرا/canvas و`system.run` على ذلك الجهاز.

    النمط الشائع:

    - Gateway على Mac mini ‏(دائم التشغيل).
    - يشغّل MacBook Pro تطبيق macOS أو مضيف عقدة ويقترن بـ Gateway.
    - استخدم `openclaw nodes status` / `openclaw nodes list` لرؤيته.

    الوثائق: [Nodes](/nodes)، [Nodes CLI](/cli/nodes).

  </Accordion>

  <Accordion title="هل يمكنني استخدام Bun؟">
    Bun ‏**غير موصى به**. فنحن نرى عيوبًا في وقت التشغيل، وخاصةً مع WhatsApp وTelegram.
    استخدم **Node** للحصول على Gateways مستقرة.

    وإذا كنت لا تزال تريد التجربة باستخدام Bun، فافعل ذلك على Gateway غير إنتاجية
    ومن دون WhatsApp/Telegram.

  </Accordion>

  <Accordion title="Telegram: ماذا أضع في allowFrom؟">
    إن `channels.telegram.allowFrom` هو **معرّف مستخدم Telegram البشري** الخاص بالمرسل (رقمي). وليس اسم مستخدم البوت.

    يقبل الإعداد الأولي إدخال `@username` ويحوّله إلى معرّف رقمي، لكن تفويض OpenClaw يستخدم المعرّفات الرقمية فقط.

    الطريقة الأكثر أمانًا (من دون بوت طرف ثالث):

    - أرسل رسالة مباشرة إلى البوت، ثم شغّل `openclaw logs --follow` واقرأ `from.id`.

    Bot API الرسمي:

    - أرسل رسالة مباشرة إلى البوت، ثم استدعِ `https://api.telegram.org/bot<bot_token>/getUpdates` واقرأ `message.from.id`.

    طرف ثالث (خصوصية أقل):

    - أرسل رسالة مباشرة إلى `@userinfobot` أو `@getidsbot`.

    راجع [/channels/telegram](/channels/telegram#access-control-and-activation).

  </Accordion>

  <Accordion title="هل يمكن لعدة أشخاص استخدام رقم WhatsApp واحد مع مثيلات OpenClaw مختلفة؟">
    نعم، عبر **Multi-Agent Routing**. اربط DM الخاصة بكل مرسل على WhatsApp ‏(peer من النوع `kind: "direct"`، مع E.164 الخاص بالمرسل مثل `+15551234567`) بـ `agentId` مختلف، بحيث يحصل كل شخص على مساحة عمله ومخزن جلساته الخاصين. وستظل الردود تأتي من **حساب WhatsApp نفسه**، كما أن التحكم في الوصول عبر DM ‏(`channels.whatsapp.dmPolicy` / `channels.whatsapp.allowFrom`) يكون عامًا لكل حساب WhatsApp. راجع [Multi-Agent Routing](/concepts/multi-agent) و[WhatsApp](/channels/whatsapp).
  </Accordion>

  <Accordion title='هل يمكنني تشغيل وكيل "دردشة سريعة" ووكيل "Opus للبرمجة"؟'>
    نعم. استخدم Multi-Agent Routing: امنح كل وكيل نموذجه الافتراضي الخاص، ثم اربط المسارات الواردة (حساب المزوّد أو peers محددين) بكل وكيل. يوجد إعداد مثال في [Multi-Agent Routing](/concepts/multi-agent). راجع أيضًا [النماذج](/concepts/models) و[الإعداد](/gateway/configuration).
  </Accordion>

  <Accordion title="هل يعمل Homebrew على Linux؟">
    نعم. يدعم Homebrew نظام Linux ‏(Linuxbrew). إعداد سريع:

    ```bash
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
    echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"' >> ~/.profile
    eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
    brew install <formula>
    ```

    إذا شغّلت OpenClaw عبر systemd، فتأكد من أن PATH الخاصة بالخدمة تتضمن `/home/linuxbrew/.linuxbrew/bin` ‏(أو بادئة brew لديك) حتى تتمكن الأدوات المثبتة عبر `brew` من العمل في shell غير التفاعلية.
    كما أن الإصدارات الحديثة تضيف مسبقًا أدلة bin الشائعة للمستخدم على خدمات Linux systemd ‏(مثل `~/.local/bin`, `~/.npm-global/bin`, `~/.local/share/pnpm`, `~/.bun/bin`) وتحترم `PNPM_HOME`, `NPM_CONFIG_PREFIX`, `BUN_INSTALL`, `VOLTA_HOME`, `ASDF_DATA_DIR`, `NVM_DIR`, و`FNM_DIR` عند ضبطها.

  </Accordion>

  <Accordion title="ما الفرق بين التثبيت القابل للتعديل عبر git وnpm install؟">
    - **التثبيت القابل للتعديل (git):** نسخة مصدرية كاملة قابلة للتحرير، وهي الأفضل للمساهمين.
      إذ تقوم أنت بتشغيل build محليًا ويمكنك تعديل الكود/الوثائق.
    - **npm install:** تثبيت CLI عام، من دون مستودع، وهو الأفضل لمن يريد "فقط تشغيله".
      تأتي التحديثات من npm dist-tags.

    الوثائق: [Getting started](/ar/start/getting-started)، [Updating](/install/updating).

  </Accordion>

  <Accordion title="هل يمكنني التبديل بين تثبيت npm وgit لاحقًا؟">
    نعم. ثبّت النسخة الأخرى، ثم شغّل Doctor حتى تشير خدمة gateway إلى نقطة الدخول الجديدة.
    وهذا **لا يحذف بياناتك** - بل يغيّر تثبيت كود OpenClaw فقط. أما حالتك
    (`~/.openclaw`) ومساحة عملك (`~/.openclaw/workspace`) فتبقيان كما هما.

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

    يكتشف Doctor عدم تطابق نقطة دخول خدمة gateway ويعرض إعادة كتابة إعداد الخدمة ليتوافق مع التثبيت الحالي (استخدم `--repair` في الأتمتة).

    نصائح النسخ الاحتياطي: راجع [استراتيجية النسخ الاحتياطي](#where-things-live-on-disk).

  </Accordion>

  <Accordion title="هل يجب أن أشغّل Gateway على حاسوبي المحمول أم على VPS؟">
    باختصار: **إذا كنت تريد موثوقية على مدار الساعة، فاستخدم VPS**. وإذا كنت تريد
    أقل احتكاك ممكن وكنت تتقبل النوم/إعادة التشغيل، فشغّله محليًا.

    **الحاسوب المحمول (Gateway محلية)**

    - **الإيجابيات:** لا تكلفة خادم، وصول مباشر إلى الملفات المحلية، ونافذة متصفح مرئية.
    - **السلبيات:** السكون/انقطاع الشبكة = انقطاعات، وتحديثات/إعادة تشغيل نظام التشغيل تقطع العمل، ويجب أن يبقى الجهاز مستيقظًا.

    **VPS / السحابة**

    - **الإيجابيات:** دائم التشغيل، شبكة مستقرة، لا توجد مشكلات سكون للحاسوب المحمول، أسهل في الإبقاء عليه قيد التشغيل.
    - **السلبيات:** غالبًا يعمل headless ‏(استخدم لقطات شاشة)، الوصول إلى الملفات يكون عن بُعد فقط، ويجب عليك استخدام SSH للتحديثات.

    **ملاحظة خاصة بـ OpenClaw:** تعمل WhatsApp/Telegram/Slack/Mattermost/Discord كلها جيدًا من VPS. والمقايضة الحقيقية الوحيدة هي **متصفح headless** مقابل نافذة مرئية. راجع [Browser](/tools/browser).

    **الافتراضي الموصى به:** VPS إذا واجهت من قبل انقطاعات في gateway. أما التشغيل المحلي فممتاز عندما تستخدم جهاز Mac فعليًا وتريد الوصول إلى الملفات المحلية أو أتمتة UI مع متصفح مرئي.

  </Accordion>

  <Accordion title="ما مدى أهمية تشغيل OpenClaw على جهاز مخصص؟">
    ليس مطلوبًا، لكنه **موصى به من أجل الموثوقية والعزل**.

    - **مضيف مخصص (VPS/Mac mini/Pi):** دائم التشغيل، وانقطاعات أقل بسبب السكون/إعادة التشغيل، وأذونات أنظف، وأسهل في الإبقاء عليه قيد العمل.
    - **حاسوب محمول/مكتبي مشترك:** مناسب تمامًا للاختبار والاستخدام النشط، لكن توقّع توقفات عندما ينام الجهاز أو يتم تحديثه.

    إذا كنت تريد أفضل ما في العالمين، فأبقِ Gateway على مضيف مخصص واقترن بحاسوبك المحمول كـ **node** للحصول على أدوات الشاشة/الكاميرا/exec المحلية. راجع [Nodes](/nodes).
    ولإرشادات الأمان، اقرأ [Security](/gateway/security).

  </Accordion>

  <Accordion title="ما الحد الأدنى لمتطلبات VPS ونظام التشغيل الموصى به؟">
    OpenClaw خفيف. وبالنسبة إلى Gateway أساسية + قناة دردشة واحدة:

    - **الحد الأدنى المطلق:** 1 vCPU، و1GB RAM، وحوالي 500MB من القرص.
    - **الموصى به:** 1-2 vCPU، و2GB RAM أو أكثر للحصول على هامش أكبر (للسجلات، والوسائط، والقنوات المتعددة). فقد تكون أدوات Node وأتمتة المتصفح شرهة للموارد.

    نظام التشغيل: استخدم **Ubuntu LTS** ‏(أو أي Debian/Ubuntu حديث). فهذا هو المسار الأكثر اختبارًا للتثبيت على Linux.

    الوثائق: [Linux](/platforms/linux)، [VPS hosting](/vps).

  </Accordion>

  <Accordion title="هل يمكنني تشغيل OpenClaw داخل VM وما المتطلبات؟">
    نعم. عامل VM بالطريقة نفسها التي تعامل بها VPS: يجب أن تكون دائمة التشغيل، وقابلة للوصول، وأن تحتوي على
    ما يكفي من RAM لـ Gateway وأي قنوات تفعّلها.

    الإرشادات الأساسية:

    - **الحد الأدنى المطلق:** 1 vCPU، و1GB RAM.
    - **الموصى به:** 2GB RAM أو أكثر إذا كنت تشغّل قنوات متعددة أو أتمتة متصفح أو أدوات وسائط.
    - **نظام التشغيل:** Ubuntu LTS أو Debian/Ubuntu حديثان.

    إذا كنت على Windows، فإن **WSL2 هو أسهل إعداد بنمط VM** ويملك أفضل توافق
    مع الأدوات. راجع [Windows](/platforms/windows)، [VPS hosting](/vps).
    وإذا كنت تشغّل macOS داخل VM، فراجع [macOS VM](/install/macos-vm).

  </Accordion>
</AccordionGroup>

## ما هو OpenClaw؟

<AccordionGroup>
  <Accordion title="ما هو OpenClaw في فقرة واحدة؟">
    OpenClaw هو مساعد AI شخصي تشغّله على أجهزتك الخاصة. وهو يرد على أسطح المراسلة التي تستخدمها بالفعل (WhatsApp، وTelegram، وSlack، وMattermost، وDiscord، وGoogle Chat، وSignal، وiMessage، وWebChat، بالإضافة إلى مكونات قنوات إضافية مضمّنة مثل QQ Bot)، ويمكنه أيضًا التعامل مع الصوت + Canvas حي على المنصات المدعومة. ويمثل **Gateway** سطح التحكم الدائم التشغيل؛ أما المساعد فهو المنتج.
  </Accordion>

  <Accordion title="القيمة الأساسية">
    OpenClaw ليس "مجرد مغلف لـ Claude". إنه **سطح تحكم محلي أولًا** يتيح لك تشغيل
    مساعد قوي على **عتادك الخاص**، يمكن الوصول إليه من تطبيقات الدردشة التي تستخدمها بالفعل، مع
    جلسات ذات حالة، وذاكرة، وأدوات - من دون تسليم التحكم في سير عملك إلى خدمة
    SaaS مستضافة.

    أبرز النقاط:

    - **أجهزتك، بياناتك:** شغّل Gateway أينما شئت (Mac أو Linux أو VPS) واحتفظ
      بمساحة العمل + سجل الجلسات محليًا.
    - **قنوات حقيقية، لا صندوق ويب معزول:** WhatsApp/Telegram/Slack/Discord/Signal/iMessage/etc،
      بالإضافة إلى الصوت على الهاتف المحمول وCanvas على المنصات المدعومة.
    - **غير مرتبط بمزوّد نموذج واحد:** استخدم Anthropic، وOpenAI، وMiniMax، وOpenRouter، وغير ذلك، مع توجيه
      لكل وكيل وبدائل احتياطية.
    - **خيار محلي فقط:** شغّل نماذج محلية بحيث **يمكن أن تبقى كل البيانات على جهازك** إذا أردت.
    - **Multi-agent routing:** وكلاء منفصلون لكل قناة أو حساب أو مهمة، ولكل منهم
      مساحة عمله وافتراضاته الخاصة.
    - **مفتوح المصدر وقابل للتعديل:** افحصه ووسّعه واستضفه بنفسك من دون قفل من مزوّد.

    الوثائق: [Gateway](/gateway)، [Channels](/channels)، [Multi-agent](/concepts/multi-agent)،
    [Memory](/concepts/memory).

  </Accordion>

  <Accordion title="لقد أعددته للتو - ماذا أفعل أولًا؟">
    مشاريع أولى جيدة:

    - بناء موقع ويب (WordPress، أو Shopify، أو موقع ثابت بسيط).
    - إعداد نموذج أولي لتطبيق جوّال (المخطط، والشاشات، وخطة API).
    - تنظيم الملفات والمجلدات (التنظيف، والتسمية، والوسوم).
    - ربط Gmail وأتمتة الملخصات أو المتابعات.

    يمكنه التعامل مع مهام كبيرة، لكنه يعمل بشكل أفضل عندما تقسّمها إلى مراحل
    وتستخدم وكلاء فرعيين للعمل المتوازي.

  </Accordion>

  <Accordion title="ما أهم خمس حالات استخدام يومية لـ OpenClaw؟">
    المكاسب اليومية تبدو عادةً هكذا:

    - **إحاطات شخصية:** ملخصات للبريد الوارد، والتقويم، والأخبار التي تهمك.
    - **البحث والصياغة:** بحث سريع، وملخصات، ومسودات أولية للبريد أو المستندات.
    - **التذكيرات والمتابعات:** تنبيهات وقوائم تحقق تقودها cron أو heartbeat.
    - **أتمتة المتصفح:** ملء النماذج، وجمع البيانات، وتكرار مهام الويب.
    - **التنسيق بين الأجهزة:** أرسل مهمة من هاتفك، ودع Gateway تشغّلها على خادم، واستلم النتيجة في الدردشة.

  </Accordion>

  <Accordion title="هل يمكن لـ OpenClaw أن يساعد في lead gen وoutreach والإعلانات والمدونات الخاصة بـ SaaS؟">
    نعم بالنسبة إلى **البحث، والتأهيل، والصياغة**. إذ يمكنه فحص المواقع، وبناء قوائم مختصرة،
    وتلخيص العملاء المحتملين، وكتابة مسودات outreach أو copy للإعلانات.

    أما بالنسبة إلى **outreach أو تشغيل الإعلانات**، فأبقِ الإنسان في الحلقة. وتجنب الرسائل المزعجة، والتزم بالقوانين المحلية و
    سياسات المنصات، وراجع أي شيء قبل إرساله. وأكثر الأنماط أمانًا هو أن
    يدع OpenClaw يصوغ وأنت توافق.

    الوثائق: [Security](/gateway/security).

  </Accordion>

  <Accordion title="ما مزايا OpenClaw مقارنةً بـ Claude Code لتطوير الويب؟">
    OpenClaw هو **مساعد شخصي** وطبقة تنسيق، وليس بديلًا عن IDE. استخدم
    Claude Code أو Codex للحصول على أسرع حلقة برمجية مباشرة داخل مستودع. واستخدم OpenClaw عندما
    تريد ذاكرة دائمة، ووصولًا عبر الأجهزة، وتنسيقًا للأدوات.

    المزايا:

    - **ذاكرة + مساحة عمل دائمتان** عبر الجلسات
    - **وصول متعدد المنصات** ‏(WhatsApp، وTelegram، وTUI، وWebChat)
    - **تنسيق الأدوات** ‏(المتصفح، والملفات، والجدولة، وhooks)
    - **Gateway دائمة التشغيل** ‏(شغّلها على VPS، وتفاعل معها من أي مكان)
    - **Nodes** للوصول المحلي إلى المتصفح/الشاشة/الكاميرا/exec

    المعرض: [https://openclaw.ai/showcase](https://openclaw.ai/showcase)

  </Accordion>
</AccordionGroup>

## Skills والأتمتة

<AccordionGroup>
  <Accordion title="كيف أخصص Skills من دون إبقاء المستودع متسخًا؟">
    استخدم تجاوزات مُدارة بدل تعديل نسخة المستودع. ضع تغييراتك في `~/.openclaw/skills/<name>/SKILL.md` ‏(أو أضف مجلدًا عبر `skills.load.extraDirs` في `~/.openclaw/openclaw.json`). وترتيب الأسبقية هو `<workspace>/skills` → `<workspace>/.agents/skills` → `~/.agents/skills` → `~/.openclaw/skills` → المضمّنة → `skills.load.extraDirs`، لذا فإن التجاوزات المُدارة تظل تتغلب على Skills المضمّنة من دون لمس git. وإذا كنت تحتاج إلى تثبيت المهارة عالميًا لكن تريد إظهارها لبعض الوكلاء فقط، فأبقِ النسخة المشتركة في `~/.openclaw/skills` وتحكم في ظهورها عبر `agents.defaults.skills` و`agents.list[].skills`. أما التعديلات التي تستحق الدمج upstream فقط فهي التي ينبغي أن تعيش في المستودع وتُرسل كـ PRs.
  </Accordion>

  <Accordion title="هل يمكنني تحميل Skills من مجلد مخصص؟">
    نعم. أضف أدلة إضافية عبر `skills.load.extraDirs` في `~/.openclaw/openclaw.json` ‏(أدنى أولوية). وترتيب الأسبقية الافتراضي هو `<workspace>/skills` → `<workspace>/.agents/skills` → `~/.agents/skills` → `~/.openclaw/skills` → المضمّنة → `skills.load.extraDirs`. وتثبّت `clawhub` في `./skills` افتراضيًا، ويعاملها OpenClaw على أنها `<workspace>/skills` في الجلسة التالية. وإذا كان ينبغي أن تظهر المهارة لبعض الوكلاء فقط، فاقرن ذلك مع `agents.defaults.skills` أو `agents.list[].skills`.
  </Accordion>

  <Accordion title="كيف يمكنني استخدام نماذج مختلفة لمهام مختلفة؟">
    الأنماط المدعومة اليوم هي:

    - **وظائف Cron**: يمكن للوظائف المعزولة ضبط تجاوز `model` لكل وظيفة.
    - **الوكلاء الفرعيون**: وجّه المهام إلى وكلاء منفصلين بقيم افتراضية مختلفة للنماذج.
    - **التبديل عند الطلب**: استخدم `/model` لتبديل نموذج الجلسة الحالية في أي وقت.

    راجع [Cron jobs](/automation/cron-jobs)، و[Multi-Agent Routing](/concepts/multi-agent)، و[Slash commands](/tools/slash-commands).

  </Accordion>

  <Accordion title="يتجمد البوت أثناء العمل الثقيل. كيف أفرّغ ذلك إلى مكان آخر؟">
    استخدم **الوكلاء الفرعيين** في المهام الطويلة أو المتوازية. تعمل الوكلاء الفرعيون في جلساتهم الخاصة،
    ويعيدون ملخصًا، ويحافظون على استجابة الدردشة الرئيسية.

    اطلب من البوت "spawn a sub-agent for this task" أو استخدم `/subagents`.
    واستخدم `/status` في الدردشة لمعرفة ما الذي يفعله Gateway الآن (وما إذا كان مشغولًا).

    نصيحة حول tokens: المهام الطويلة والوكلاء الفرعيون يستهلكون tokens. وإذا كانت التكلفة مهمة، فاضبط
    نموذجًا أرخص للوكلاء الفرعيين عبر `agents.defaults.subagents.model`.

    الوثائق: [Sub-agents](/tools/subagents)، [Background Tasks](/automation/tasks).

  </Accordion>

  <Accordion title="كيف تعمل جلسات الوكلاء الفرعيين المرتبطة بالسلاسل على Discord؟">
    استخدم ربط السلاسل. يمكنك ربط سلسلة Discord بهدف وكيل فرعي أو جلسة، بحيث تبقى الرسائل اللاحقة في تلك السلسلة على الجلسة المرتبطة.

    التدفق الأساسي:

    - Spawn باستخدام `sessions_spawn` مع `thread: true` ‏(واختياريًا `mode: "session"` للمتابعة الدائمة).
    - أو اربط يدويًا باستخدام `/focus <target>`.
    - استخدم `/agents` لفحص حالة الربط.
    - استخدم `/session idle <duration|off>` و`/session max-age <duration|off>` للتحكم في إزالة التركيز التلقائية.
    - استخدم `/unfocus` لفصل السلسلة.

    الإعداد المطلوب:

    - القيم الافتراضية العامة: `session.threadBindings.enabled`, `session.threadBindings.idleHours`, `session.threadBindings.maxAgeHours`.
    - تجاوزات Discord: ‏`channels.discord.threadBindings.enabled`, `channels.discord.threadBindings.idleHours`, `channels.discord.threadBindings.maxAgeHours`.
    - الربط التلقائي عند spawn: اضبط `channels.discord.threadBindings.spawnSubagentSessions: true`.

    الوثائق: [Sub-agents](/tools/subagents)، [Discord](/channels/discord)، [Configuration Reference](/gateway/configuration-reference)، [Slash commands](/tools/slash-commands).

  </Accordion>

  <Accordion title="اكتمل وكيل فرعي، لكن تحديث الإكمال ذهب إلى المكان الخطأ أو لم يُنشر مطلقًا. ما الذي يجب أن أفحصه؟">
    افحص أولًا المسار requester route الذي جرى حله:

    - يفضّل تسليم إكمال الوكيل الفرعي في وضع completion أي thread أو conversation route مرتبط عندما يكون موجودًا.
    - إذا كان origin الخاص بالإكمال يحمل فقط قناة، فإن OpenClaw يعود إلى المسار المخزن في جلسة الطالب (`lastChannel` / `lastTo` / `lastAccountId`) حتى ينجح التسليم المباشر.
    - إذا لم يوجد مسار مرتبط ولا مسار مخزن صالح، فقد يفشل التسليم المباشر وتعود النتيجة إلى التسليم المدرج في طابور الجلسة بدل النشر الفوري في الدردشة.
    - لا تزال الأهداف غير الصالحة أو القديمة قادرة على فرض fallback إلى الطابور أو فشل التسليم النهائي.
    - إذا كان آخر رد مرئي للمساعد في الجلسة الفرعية هو بالضبط الرمز الصامت `NO_REPLY` / `no_reply` أو بالضبط `ANNOUNCE_SKIP`، فإن OpenClaw يتعمد كتم الإعلان بدل نشر تقدم أقدم أصبح قديمًا.
    - إذا انتهت مهلة الفرع بعد استدعاءات أدوات فقط، فقد يختصر الإعلان ذلك إلى ملخص قصير للتقدم الجزئي بدل إعادة تشغيل مخرجات الأدوات الخام.

    التصحيح:

    ```bash
    openclaw tasks show <runId-or-sessionKey>
    ```

    الوثائق: [Sub-agents](/tools/subagents)، [Background Tasks](/automation/tasks)، [Session Tools](/concepts/session-tool).

  </Accordion>

  <Accordion title="لا تعمل cron أو التذكيرات. ما الذي يجب أن أفحصه؟">
    تعمل cron داخل عملية Gateway. وإذا لم يكن Gateway يعمل باستمرار،
    فلن تعمل الوظائف المجدولة.

    قائمة التحقق:

    - تأكد من أن cron مفعلة (`cron.enabled`) وأن `OPENCLAW_SKIP_CRON` غير مضبوط.
    - تحقق من أن Gateway تعمل 24/7 ‏(من دون سكون/إعادة تشغيل).
    - تحقّق من إعدادات المنطقة الزمنية الخاصة بالوظيفة (`--tz` مقابل المنطقة الزمنية للمضيف).

    التصحيح:

    ```bash
    openclaw cron run <jobId>
    openclaw cron runs --id <jobId> --limit 50
    ```

    الوثائق: [Cron jobs](/automation/cron-jobs)، [Automation & Tasks](/automation).

  </Accordion>

  <Accordion title="عملت cron، لكن لم يُرسل شيء إلى القناة. لماذا؟">
    افحص أولًا وضع التسليم:

    - `--no-deliver` / `delivery.mode: "none"` يعني عدم توقّع رسالة خارجية.
    - غياب أو خطأ هدف announce ‏(`channel` / `to`) يعني أن المشغّل تخطى التسليم الصادر.
    - إخفاقات مصادقة القناة (`unauthorized`, `Forbidden`) تعني أن المشغّل حاول التسليم لكن بيانات الاعتماد منعته.
    - النتيجة المعزولة الصامتة (`NO_REPLY` / `no_reply` فقط) تُعامل على أنها غير قابلة للتسليم عمدًا، لذا يكتم المشغّل أيضًا التسليم الاحتياطي المدرج في الطابور.

    بالنسبة إلى وظائف cron المعزولة، يمتلك المشغّل التسليم النهائي. والمفترض
    أن يعيد الوكيل ملخصًا نصيًا عاديًا ليرسله المشغّل. أما `--no-deliver` فيُبقي
    تلك النتيجة داخلية؛ ولا يسمح للوكيل بالإرسال مباشرة باستخدام
    message tool بدل ذلك.

    التصحيح:

    ```bash
    openclaw cron runs --id <jobId> --limit 50
    openclaw tasks show <runId-or-sessionKey>
    ```

    الوثائق: [Cron jobs](/automation/cron-jobs)، [Background Tasks](/automation/tasks).

  </Accordion>

  <Accordion title="لماذا بدّل تشغيل cron المعزول النماذج أو أعاد المحاولة مرة واحدة؟">
    يكون هذا عادةً مسار تبديل النموذج الحي، وليس جدولة مكررة.

    يمكن لـ cron المعزول حفظ handoff لنموذج وقت التشغيل وإعادة المحاولة عندما يلقي التشغيل النشط
    `LiveSessionModelSwitchError`. وتحافظ إعادة المحاولة على
    المزوّد/النموذج اللذين جرى التبديل إليهما، وإذا كان التبديل يحمل معه override لملف تعريف مصادقة جديد، فإن cron
    تحفظه أيضًا قبل إعادة المحاولة.

    قواعد الاختيار ذات الصلة:

    - يربح override نموذج Gmail hook أولًا عند الاقتضاء.
    - ثم `model` الخاصة بالوظيفة.
    - ثم أي override للنموذج مخزنة في جلسة cron.
    - ثم الاختيار العادي للنموذج الافتراضي/وكيل.

    حلقة إعادة المحاولة محدودة. فبعد المحاولة الأولى بالإضافة إلى محاولتي تبديل،
    توقف cron بدل الدوران إلى ما لا نهاية.

    التصحيح:

    ```bash
    openclaw cron runs --id <jobId> --limit 50
    openclaw tasks show <runId-or-sessionKey>
    ```

    الوثائق: [Cron jobs](/automation/cron-jobs)، [cron CLI](/cli/cron).

  </Accordion>

  <Accordion title="كيف أُثبّت Skills على Linux؟">
    استخدم أوامر `openclaw skills` الأصلية أو ضع Skills داخل مساحة العمل. لا تتوفر واجهة Skills في macOS على Linux.
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

    يكتب `openclaw skills install` الأصلي إلى دليل `skills/`
    داخل مساحة العمل النشطة. ولا تثبّت CLI المنفصلة `clawhub` إلا إذا كنت تريد نشر أو
    مزامنة Skills الخاصة بك. وبالنسبة إلى التثبيتات المشتركة عبر الوكلاء، ضع المهارة تحت
    `~/.openclaw/skills` واستخدم `agents.defaults.skills` أو
    `agents.list[].skills` إذا كنت تريد حصر الوكلاء الذين يمكنهم رؤيتها.

  </Accordion>

  <Accordion title="هل يمكن لـ OpenClaw تشغيل مهام وفق جدول أو بشكل مستمر في الخلفية؟">
    نعم. استخدم مجدول Gateway:

    - **وظائف Cron** للمهام المجدولة أو المتكررة (وتبقى عبر إعادة التشغيل).
    - **Heartbeat** للفحوصات الدورية الخاصة بـ "الجلسة الرئيسية".
    - **وظائف معزولة** للوكلاء الذاتيين الذين ينشرون ملخصات أو يسلّمون إلى الدردشات.

    الوثائق: [Cron jobs](/automation/cron-jobs)، [Automation & Tasks](/automation)،
    [Heartbeat](/gateway/heartbeat).

  </Accordion>

  <Accordion title="هل يمكنني تشغيل Skills الخاصة بـ macOS فقط من Linux؟">
    ليس مباشرة. تخضع Skills الخاصة بـ macOS إلى `metadata.openclaw.os` بالإضافة إلى الثنائيات المطلوبة، ولا تظهر Skills في system prompt إلا عندما تكون مؤهلة على **مضيف Gateway**. وعلى Linux، لن تُحمَّل Skills الخاصة بـ `darwin` فقط (مثل `apple-notes`, `apple-reminders`, `things-mac`) ما لم تتجاوز آلية التقييد.

    لديك ثلاثة أنماط مدعومة:

    **الخيار A - تشغيل Gateway على جهاز Mac (الأبسط).**
    شغّل Gateway حيث توجد الثنائيات الخاصة بـ macOS، ثم اتصل من Linux في [remote mode](#gateway-ports-already-running-and-remote-mode) أو عبر Tailscale. سيتم تحميل Skills بشكل طبيعي لأن مضيف Gateway هو macOS.

    **الخيار B - استخدام node على macOS (من دون SSH).**
    شغّل Gateway على Linux، واقرن node على macOS ‏(تطبيق menubar)، واضبط **Node Run Commands** على "Always Ask" أو "Always Allow" على جهاز Mac. يمكن لـ OpenClaw التعامل مع Skills الخاصة بـ macOS على أنها مؤهلة عندما تكون الثنائيات المطلوبة موجودة على node. وسيشغّل الوكيل تلك Skills عبر أداة `nodes`. وإذا اخترت "Always Ask"، فإن الموافقة على "Always Allow" في المطالبة تضيف ذلك الأمر إلى قائمة السماح.

    **الخيار C - تمرير ثنائيات macOS عبر SSH (متقدم).**
    أبقِ Gateway على Linux، لكن اجعل الثنائيات المطلوبة في CLI تُحل إلى أغلفة SSH تعمل على جهاز Mac. ثم تجاوز المهارة للسماح بـ Linux حتى تبقى مؤهلة.

    1. أنشئ غلاف SSH للثنائي (مثال: `memo` لـ Apple Notes):

       ```bash
       #!/usr/bin/env bash
       set -euo pipefail
       exec ssh -T user@mac-host /opt/homebrew/bin/memo "$@"
       ```

    2. ضع الغلاف في `PATH` على مضيف Linux ‏(مثل `~/bin/memo`).
    3. تجاوز metadata الخاصة بالمهارة (في مساحة العمل أو `~/.openclaw/skills`) للسماح بـ Linux:

       ```markdown
       ---
       name: apple-notes
       description: Manage Apple Notes via the memo CLI on macOS.
       metadata: { "openclaw": { "os": ["darwin", "linux"], "requires": { "bins": ["memo"] } } }
       ---
       ```

    4. ابدأ جلسة جديدة حتى تتجدد لقطة Skills.

  </Accordion>

  <Accordion title="هل لديكم تكامل مع Notion أو HeyGen؟">
    ليس مضمّنًا حاليًا.

    الخيارات:

    - **Skill / plugin مخصص:** الأفضل للوصول الموثوق إلى API ‏(كل من Notion وHeyGen لديهما APIs).
    - **أتمتة المتصفح:** تعمل من دون كود لكنها أبطأ وأكثر هشاشة.

    إذا كنت تريد إبقاء السياق لكل عميل (في سير عمل الوكالات)، فالنمط البسيط هو:

    - صفحة Notion واحدة لكل عميل (السياق + التفضيلات + العمل النشط).
    - اطلب من الوكيل جلب تلك الصفحة في بداية الجلسة.

    إذا كنت تريد تكاملًا أصليًا، فافتح طلب ميزة أو ابنِ Skill
    تستهدف تلك APIs.

    تثبيت Skills:

    ```bash
    openclaw skills install <skill-slug>
    openclaw skills update --all
    ```

    تُثبَّت التثبيتات الأصلية في دليل `skills/` ضمن مساحة العمل النشطة. أما بالنسبة إلى Skills المشتركة عبر الوكلاء، فضَعها في `~/.openclaw/skills/<name>/SKILL.md`. وإذا كان يجب أن تراها بعض الوكلاء فقط من بين تثبيت مشترك، فاضبط `agents.defaults.skills` أو `agents.list[].skills`. وتتوقع بعض Skills وجود ثنائيات مثبّتة عبر Homebrew؛ وعلى Linux يعني ذلك Linuxbrew ‏(راجع إدخال Homebrew Linux في الأسئلة الشائعة أعلاه). راجع [Skills](/tools/skills)، و[Skills config](/tools/skills-config)، و[ClawHub](/tools/clawhub).

  </Accordion>

  <Accordion title="كيف أستخدم Chrome الموقّع الدخول فيه مسبقًا مع OpenClaw؟">
    استخدم ملف تعريف المتصفح المضمّن `user`، والذي يتصل عبر Chrome DevTools MCP:

    ```bash
    openclaw browser --browser-profile user tabs
    openclaw browser --browser-profile user snapshot
    ```

    وإذا كنت تريد اسمًا مخصصًا، فأنشئ profile MCP صريحة:

    ```bash
    openclaw browser create-profile --name chrome-live --driver existing-session
    openclaw browser --browser-profile chrome-live tabs
    ```

    هذا المسار محلي على المضيف. وإذا كانت Gateway تعمل في مكان آخر، فإما أن تشغّل مضيف node على جهاز المتصفح أو تستخدم CDP بعيدًا بدلًا من ذلك.

    الحدود الحالية على `existing-session` / `user`:

    - الإجراءات مبنية على المراجع ref-driven، وليست مبنية على محددات CSS
    - تتطلب عمليات الرفع `ref` / `inputRef` وتدعم حاليًا ملفًا واحدًا في كل مرة
    - ما تزال `responsebody`، وتصدير PDF، واعتراض التنزيلات، والإجراءات الدفعية بحاجة إلى متصفح مُدار أو profile CDP خام

  </Accordion>
</AccordionGroup>

## Sandbox والذاكرة

<AccordionGroup>
  <Accordion title="هل توجد وثيقة مخصصة لـ sandboxing؟">
    نعم. راجع [Sandboxing](/gateway/sandboxing). وبالنسبة إلى إعدادات Docker الخاصة (Gateway كاملة داخل Docker أو صور sandbox)، راجع [Docker](/install/docker).
  </Accordion>

  <Accordion title="يبدو Docker محدودًا - كيف أفعّل الميزات الكاملة؟">
    الصورة الافتراضية تركز على الأمان وتعمل كمستخدم `node`، لذا فهي
    لا تتضمن حزم النظام، ولا Homebrew، ولا متصفحات مضمّنة. ولإعداد أكثر اكتمالًا:

    - اجعل `/home/node` دائمًا عبر `OPENCLAW_HOME_VOLUME` حتى تبقى الذاكرات المؤقتة.
    - أضف تبعيات النظام إلى الصورة عبر `OPENCLAW_DOCKER_APT_PACKAGES`.
    - ثبّت متصفحات Playwright عبر CLI المضمّنة:
      `node /app/node_modules/playwright-core/cli.js install chromium`
    - اضبط `PLAYWRIGHT_BROWSERS_PATH` وتأكد من أن المسار دائم.

    الوثائق: [Docker](/install/docker)، [Browser](/tools/browser).

  </Accordion>

  <Accordion title="هل يمكنني إبقاء DMs شخصية وجعل المجموعات عامة/معزولة عبر sandbox باستخدام وكيل واحد؟">
    نعم - إذا كانت الحركة الخاصة لديك هي **DMs** والحركة العامة هي **المجموعات**.

    استخدم `agents.defaults.sandbox.mode: "non-main"` بحيث تعمل جلسات المجموعات/القنوات (المفاتيح غير الرئيسية) داخل Docker، بينما تبقى جلسة DM الرئيسية على المضيف. ثم قيّد الأدوات المتاحة في الجلسات المعزولة عبر `tools.sandbox.tools`.

    شرح الإعداد + مثال الإعداد: [Groups: personal DMs + public groups](/channels/groups#pattern-personal-dms-public-groups-single-agent)

    مرجع الإعداد الأساسي: [Gateway configuration](/gateway/configuration-reference#agentsdefaultssandbox)

  </Accordion>

  <Accordion title="كيف أربط مجلد مضيف داخل sandbox؟">
    اضبط `agents.defaults.sandbox.docker.binds` على `["host:path:mode"]` ‏(مثل `"/home/user/src:/src:ro"`). وتُدمج عمليات الربط العامة + الخاصة بكل وكيل؛ وتُتجاهل عمليات الربط الخاصة بالوكيل عندما يكون `scope: "shared"`. استخدم `:ro` مع أي شيء حساس، وتذكر أن عمليات الربط تتجاوز جدران نظام الملفات الخاصة بـ sandbox.

    يتحقق OpenClaw من مصادر الربط مقابل كل من المسار المطبع والمسار القانوني المحلول من أعمق ancestor موجود. وهذا يعني أن محاولات الهروب عبر parents الرمزية المغطاة بـ symlink ما تزال تفشل بإغلاق آمن حتى عندما لا يكون المقطع الأخير من المسار موجودًا بعد، كما أن فحوصات الجذر المسموح به تبقى مطبقة بعد حل symlink.

    راجع [Sandboxing](/gateway/sandboxing#custom-bind-mounts) و[Sandbox vs Tool Policy vs Elevated](/gateway/sandbox-vs-tool-policy-vs-elevated#bind-mounts-security-quick-check) للأمثلة وملاحظات السلامة.

  </Accordion>

  <Accordion title="كيف تعمل الذاكرة؟">
    ذاكرة OpenClaw هي مجرد ملفات Markdown في مساحة عمل الوكيل:

    - ملاحظات يومية في `memory/YYYY-MM-DD.md`
    - ملاحظات طويلة الأمد مُنقّحة في `MEMORY.md` ‏(للجلسات الرئيسية/الخاصة فقط)

    يشغّل OpenClaw أيضًا **تفريغ ذاكرة صامتًا قبل الضغط** لتذكير النموذج
    بكتابة ملاحظات دائمة قبل الضغط التلقائي. ولا يعمل هذا إلا عندما تكون مساحة العمل
    قابلة للكتابة (فتتخطاه sandboxes للقراءة فقط). راجع [Memory](/concepts/memory).

  </Accordion>

  <Accordion title="الذاكرة تستمر في نسيان الأشياء. كيف أجعلها ثابتة؟">
    اطلب من البوت أن **يكتب الحقيقة في الذاكرة**. فالملاحظات طويلة الأمد توضع في `MEMORY.md`،
    والسياق قصير الأمد يوضع في `memory/YYYY-MM-DD.md`.

    ما زلنا نحسّن هذا المجال. ويساعد تذكير النموذج بتخزين الذكريات؛
    فهو سيعرف ما الذي عليه فعله. وإذا استمر في النسيان، فتحقق من أن Gateway تستخدم
    مساحة العمل نفسها في كل تشغيل.

    الوثائق: [Memory](/concepts/memory)، [مساحة عمل الوكيل](/concepts/agent-workspace).

  </Accordion>

  <Accordion title="هل تستمر الذاكرة إلى الأبد؟ وما الحدود؟">
    تعيش ملفات الذاكرة على القرص وتبقى حتى تحذفها. فالحد هو
    التخزين لديك، وليس النموذج. لكن **سياق الجلسة** يظل محدودًا بنافذة سياق
    النموذج، لذلك يمكن أن تُضغط أو تُقتطع المحادثات الطويلة. ولهذا
    يوجد البحث الدلالي في الذاكرة - فهو يسحب فقط الأجزاء ذات الصلة إلى السياق.

    الوثائق: [Memory](/concepts/memory)، [Context](/concepts/context).

  </Accordion>

  <Accordion title="هل يتطلب البحث الدلالي في الذاكرة OpenAI API key؟">
    فقط إذا كنت تستخدم **OpenAI embeddings**. فـ Codex OAuth يغطي chat/completions
    لكنه **لا** يمنح وصولًا إلى embeddings، لذا فإن **تسجيل الدخول عبر Codex (OAuth أو
    تسجيل دخول Codex CLI)** لا يساعد في البحث الدلالي في الذاكرة. وما تزال OpenAI embeddings
    تحتاج إلى API key حقيقية (`OPENAI_API_KEY` أو `models.providers.openai.apiKey`).

    إذا لم تضبط مزوّدًا صراحةً، فإن OpenClaw تختار مزوّدًا تلقائيًا عندما
    تتمكن من حل API key ‏(من auth profiles، أو `models.providers.*.apiKey`، أو env vars).
    وهي تفضّل OpenAI إذا أمكن حل مفتاح OpenAI، ثم Gemini إذا أمكن حل مفتاح Gemini،
    ثم Voyage، ثم Mistral. وإذا لم يتوفر أي مفتاح بعيد، يبقى بحث الذاكرة
    معطلًا حتى تُهيئه. وإذا كان لديك مسار نموذج محلي مضبوط وموجود، فإن OpenClaw
    تفضّل `local`. كما أن Ollama مدعومة عندما تضبط صراحةً
    `memorySearch.provider = "ollama"`.

    وإذا كنت تفضّل البقاء محليًا، فاضبط `memorySearch.provider = "local"` ‏(واختياريًا
    `memorySearch.fallback = "none"`). وإذا كنت تريد Gemini embeddings، فاضبط
    `memorySearch.provider = "gemini"` ووفّر `GEMINI_API_KEY` ‏(أو
    `memorySearch.remote.apiKey`). نحن ندعم نماذج embedding الخاصة بـ **OpenAI، وGemini، وVoyage، وMistral، وOllama، أو المحلية**
    - راجع [Memory](/concepts/memory) لتفاصيل الإعداد.

  </Accordion>
</AccordionGroup>

## أين تعيش الأشياء على القرص

<AccordionGroup>
  <Accordion title="هل تُحفَظ كل البيانات المستخدمة مع OpenClaw محليًا؟">
    لا - **حالة OpenClaw محلية**، لكن **الخدمات الخارجية ما تزال ترى ما ترسله إليها**.

    - **محلي افتراضيًا:** تعيش الجلسات، وملفات الذاكرة، والإعداد، ومساحة العمل على مضيف Gateway
      (`~/.openclaw` + دليل مساحة العمل لديك).
    - **بعيد بحكم الضرورة:** تذهب الرسائل التي ترسلها إلى مزوّدي النماذج (Anthropic/OpenAI/etc.) إلى
      APIs الخاصة بهم، كما تخزن منصات الدردشة (WhatsApp/Telegram/Slack/etc.) بيانات الرسائل على
      خوادمها.
    - **أنت تتحكم في البصمة:** استخدام النماذج المحلية يبقي الموجّهات على جهازك، لكن حركة
      القنوات ما تزال تمر عبر خوادم القناة نفسها.

    ذو صلة: [مساحة عمل الوكيل](/concepts/agent-workspace)، [Memory](/concepts/memory).

  </Accordion>

  <Accordion title="أين يخزن OpenClaw بياناته؟">
    يعيش كل شيء تحت `$OPENCLAW_STATE_DIR` ‏(الافتراضي: `~/.openclaw`):

    | المسار                                                          | الغرض                                                              |
    | --------------------------------------------------------------- | ------------------------------------------------------------------ |
    | `$OPENCLAW_STATE_DIR/openclaw.json`                             | الإعداد الرئيسي (JSON5)                                            |
    | `$OPENCLAW_STATE_DIR/credentials/oauth.json`                    | استيراد OAuth قديم (يُنسخ إلى auth profiles عند أول استخدام)      |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/auth-profiles.json` | auth profiles ‏(OAuth، وAPI keys، و`keyRef`/`tokenRef` الاختياريان) |
    | `$OPENCLAW_STATE_DIR/secrets.json`                              | حمولة أسرار اختيارية مدعومة بالملف لمزوّدي SecretRef من نوع `file` |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/auth.json`          | ملف توافق قديم (تُزال منه إدخالات `api_key` الثابتة)              |
    | `$OPENCLAW_STATE_DIR/credentials/`                              | حالة المزوّد (مثل `whatsapp/<accountId>/creds.json`)              |
    | `$OPENCLAW_STATE_DIR/agents/`                                   | حالة خاصة بكل وكيل (agentDir + sessions)                          |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/sessions/`                | سجل المحادثات والحالة (لكل وكيل)                                  |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/sessions/sessions.json`   | بيانات الجلسات الوصفية (لكل وكيل)                                 |

    المسار القديم لوكيل واحد: `~/.openclaw/agent/*` ‏(يُرحَّل بواسطة `openclaw doctor`).

    أما **مساحة العمل** لديك (`AGENTS.md`، وملفات الذاكرة، وSkills، وما إلى ذلك) فهي منفصلة وتُضبط عبر `agents.defaults.workspace` ‏(الافتراضي: `~/.openclaw/workspace`).

  </Accordion>

  <Accordion title="أين يجب أن تعيش ملفات AGENTS.md / SOUL.md / USER.md / MEMORY.md؟">
    تعيش هذه الملفات في **مساحة عمل الوكيل**، وليس في `~/.openclaw`.

    - **مساحة العمل (لكل وكيل):** ‏`AGENTS.md`, `SOUL.md`, `IDENTITY.md`, `USER.md`,
      `MEMORY.md` ‏(أو fallback قديم `memory.md` عندما يغيب `MEMORY.md`)،
      `memory/YYYY-MM-DD.md`, و`HEARTBEAT.md` الاختيارية.
    - **دليل الحالة (`~/.openclaw`)**: الإعداد، وحالة القنوات/المزوّدات، وauth profiles، والجلسات، والسجلات،
      وSkills المشتركة (`~/.openclaw/skills`).

    مساحة العمل الافتراضية هي `~/.openclaw/workspace`، ويمكن إعدادها عبر:

    ```json5
    {
      agents: { defaults: { workspace: "~/.openclaw/workspace" } },
    }
    ```

    إذا كان البوت "ينسى" بعد إعادة التشغيل، فتأكد من أن Gateway تستخدم
    مساحة العمل نفسها في كل تشغيل (وتذكّر: في الوضع البعيد تستخدم **مساحة عمل مضيف gateway**
    نفسها، وليس مساحة عمل حاسوبك المحمول المحلي).

    نصيحة: إذا كنت تريد سلوكًا أو تفضيلًا دائمًا، فاطلب من البوت أن **يكتبه داخل
    AGENTS.md أو MEMORY.md** بدل الاعتماد على سجل الدردشة.

    راجع [مساحة عمل الوكيل](/concepts/agent-workspace) و[Memory](/concepts/memory).

  </Accordion>

  <Accordion title="استراتيجية النسخ الاحتياطي الموصى بها">
    ضع **مساحة عمل الوكيل** الخاصة بك داخل مستودع git **خاص** واحتفظ بنسخة احتياطية منها
    في مكان خاص (مثل GitHub private). فهذا يلتقط الذاكرة + ملفات AGENTS/SOUL/USER
    ويتيح لك استعادة "عقل" المساعد لاحقًا.

    لا تقم بعمل commit لأي شيء تحت `~/.openclaw` ‏(بيانات الاعتماد، أو الجلسات، أو الرموز، أو حمولات الأسرار المشفرة).
    وإذا كنت تحتاج إلى استعادة كاملة، فاحتفظ بنسخة احتياطية من مساحة العمل ومن دليل الحالة
    بشكل منفصل (راجع سؤال الترحيل أعلاه).

    الوثائق: [مساحة عمل الوكيل](/concepts/agent-workspace).

  </Accordion>

  <Accordion title="كيف أزيل OpenClaw بالكامل؟">
    راجع الدليل المخصص: [Uninstall](/install/uninstall).
  </Accordion>

  <Accordion title="هل يمكن للوكلاء العمل خارج مساحة العمل؟">
    نعم. تمثل مساحة العمل **cwd الافتراضي** ومرساة الذاكرة، وليست sandbox صارمة.
    تُحل المسارات النسبية داخل مساحة العمل، لكن المسارات المطلقة يمكنها الوصول إلى
    مواقع أخرى على المضيف ما لم يكن sandboxing مفعّلًا. وإذا كنت تحتاج إلى عزل، فاستخدم
    [`agents.defaults.sandbox`](/gateway/sandboxing) أو إعدادات sandbox لكل وكيل. وإذا كنت
    تريد أن يكون المستودع دليل العمل الافتراضي، فوجه
    `workspace` لذلك الوكيل إلى جذر المستودع. مستودع OpenClaw هو مجرد شيفرة مصدرية؛ أبقِ
    مساحة العمل منفصلة ما لم تكن تريد عمدًا أن يعمل الوكيل داخلها.

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
    حالة الجلسات يملكها **مضيف gateway**. وإذا كنت في الوضع البعيد، فإن مخزن الجلسات الذي يهمك موجود على الجهاز البعيد، وليس على حاسوبك المحمول المحلي. راجع [Session management](/concepts/session).
  </Accordion>
</AccordionGroup>

## أساسيات الإعداد

<AccordionGroup>
  <Accordion title="ما صيغة الإعداد؟ وأين يوجد؟">
    يقرأ OpenClaw إعدادًا اختياريًا بصيغة **JSON5** من `$OPENCLAW_CONFIG_PATH` ‏(الافتراضي: `~/.openclaw/openclaw.json`):

    ```
    $OPENCLAW_CONFIG_PATH
    ```

    وإذا كان الملف مفقودًا، فإنه يستخدم افتراضيات آمنة نسبيًا (بما في ذلك مساحة عمل افتراضية عند `~/.openclaw/workspace`).

  </Accordion>

  <Accordion title='لقد ضبطت gateway.bind: "lan" (أو "tailnet") والآن لا يوجد شيء يستمع / وتقول UI unauthorized'>
    يتطلب الربط غير المعتمد على loopback **مسار مصادقة صالحًا لـ gateway**. وعمليًا يعني ذلك:

    - مصادقة shared-secret: ‏token أو password
    - `gateway.auth.mode: "trusted-proxy"` خلف identity-aware reverse proxy غير معتمد على loopback ومكوَّن بصورة صحيحة

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
    - يمكن لمسارات الاستدعاء المحلية استخدام `gateway.remote.*` كبديل احتياطي فقط عندما يكون `gateway.auth.*` غير مضبوط.
    - بالنسبة إلى password auth، اضبط `gateway.auth.mode: "password"` مع `gateway.auth.password` ‏(أو `OPENCLAW_GATEWAY_PASSWORD`) بدلًا من ذلك.
    - إذا كانت `gateway.auth.token` / `gateway.auth.password` مكوّنة صراحة عبر SecretRef وغير قابلة للحل، فإن الحل يفشل بإغلاق آمن (من دون إخفاء fallback بعيد).
    - تَتوثّق إعدادات shared-secret الخاصة بـ Control UI عبر `connect.params.auth.token` أو `connect.params.auth.password` ‏(المحفوظة في إعدادات التطبيق/UI). أما الأوضاع الحاملة للهوية مثل Tailscale Serve أو `trusted-proxy` فتستخدم ترويسات الطلب بدلًا من ذلك. تجنب وضع shared secrets داخل URLs.
    - مع `gateway.auth.mode: "trusted-proxy"`، ما تزال same-host loopback reverse proxies **لا** تلبّي trusted-proxy auth. إذ يجب أن يكون trusted proxy مصدرًا غير معتمد على loopback ومكوَّنًا صراحة.

  </Accordion>

  <Accordion title="لماذا أحتاج إلى token على localhost الآن؟">
    يفرض OpenClaw مصادقة gateway افتراضيًا، بما في ذلك loopback. وفي المسار الافتراضي العادي يعني ذلك مصادقة token: إذا لم يوجد مسار مصادقة صريح مكوَّن، فإن بدء gateway يحل إلى وضع token ويولّد واحدًا تلقائيًا ويحفظه في `gateway.auth.token`، لذلك **يجب على عملاء WS المحليين التوثق**. وهذا يمنع العمليات المحلية الأخرى من استدعاء Gateway.

    وإذا كنت تفضّل مسار مصادقة مختلفًا، فيمكنك اختيار وضع password صراحةً (أو، بالنسبة إلى reverse proxies المدركة للهوية وغير المعتمدة على loopback، وضع `trusted-proxy`). وإذا كنت **حقًا** تريد loopback مفتوحًا، فاضبط `gateway.auth.mode: "none"` صراحةً في إعدادك. ويمكن لـ Doctor توليد token لك في أي وقت: ‏`openclaw doctor --generate-gateway-token`.

  </Accordion>

  <Accordion title="هل يجب أن أعيد التشغيل بعد تغيير الإعداد؟">
    تراقب Gateway الإعداد وتدعم hot-reload:

    - `gateway.reload.mode: "hybrid"` ‏(الافتراضي): يطبّق التغييرات الآمنة فورًا، ويعيد التشغيل للتغييرات الحرجة
    - كما أن الأنماط `hot` و`restart` و`off` مدعومة أيضًا

  </Accordion>

  <Accordion title="كيف أعطّل عبارات CLI الطريفة؟">
    اضبط `cli.banner.taglineMode` في الإعداد:

    ```json5
    {
      cli: {
        banner: {
          taglineMode: "off", // random | default | off
        },
      },
    }
    ```

    - `off`: يخفي نص tagline لكنه يبقي سطر عنوان/إصدار banner.
    - `default`: يستخدم `All your chats, one OpenClaw.` في كل مرة.
    - `random`: عبارات موسمية/طريفة متناوبة (السلوك الافتراضي).
    - وإذا كنت لا تريد أي banner إطلاقًا، فاضبط env ‏`OPENCLAW_HIDE_BANNER=1`.

  </Accordion>

  <Accordion title="كيف أفعّل web search ‏(وweb fetch)؟">
    يعمل `web_fetch` من دون API key. أما `web_search` فيعتمد على
    المزوّد الذي اخترته:

    - تتطلب المزوّدات المدعومة بـ API مثل Brave وExa وFirecrawl وGemini وGrok وKimi وMiniMax Search وPerplexity وTavily إعداد API key العادي الخاص بها.
    - Ollama Web Search لا يحتاج إلى مفتاح، لكنه يستخدم مضيف Ollama المكوَّن لديك ويتطلب `ollama signin`.
    - DuckDuckGo لا يحتاج إلى مفتاح، لكنه تكامل غير رسمي يعتمد على HTML.
    - SearXNG مجاني/مستضاف ذاتيًا؛ اضبط `SEARXNG_BASE_URL` أو `plugins.entries.searxng.config.webSearch.baseUrl`.

    **الموصى به:** شغّل `openclaw configure --section web` واختر مزوّدًا.
    بدائل البيئة:

    - Brave: ‏`BRAVE_API_KEY`
    - Exa: ‏`EXA_API_KEY`
    - Firecrawl: ‏`FIRECRAWL_API_KEY`
    - Gemini: ‏`GEMINI_API_KEY`
    - Grok: ‏`XAI_API_KEY`
    - Kimi: ‏`KIMI_API_KEY` أو `MOONSHOT_API_KEY`
    - MiniMax Search: ‏`MINIMAX_CODE_PLAN_KEY`, `MINIMAX_CODING_API_KEY`, أو `MINIMAX_API_KEY`
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
              provider: "firecrawl", // اختياري؛ احذفه للكشف التلقائي
            },
          },
        },
    }
    ```

    يعيش الإعداد الخاص بالبحث على الويب لكل مزود الآن تحت `plugins.entries.<plugin>.config.webSearch.*`.
    وما تزال مسارات المزوّدات القديمة `tools.web.search.*` تُحمَّل مؤقتًا من أجل التوافق، لكن لا ينبغي استخدامها في الإعدادات الجديدة.
    ويعيش إعداد fallback الخاص بـ Firecrawl web-fetch تحت `plugins.entries.firecrawl.config.webFetch.*`.

    ملاحظات:

    - إذا كنت تستخدم قوائم السماح، فأضف `web_search`/`web_fetch`/`x_search` أو `group:web`.
    - تكون `web_fetch` مفعلة افتراضيًا (ما لم تُعطّل صراحة).
    - إذا حُذفت `tools.web.fetch.provider`، فإن OpenClaw تكتشف تلقائيًا أول fetch fallback provider جاهز من بيانات الاعتماد المتاحة. واليوم المزوّد المضمّن هو Firecrawl.
    - تقرأ daemons متغيرات env من `~/.openclaw/.env` ‏(أو من بيئة الخدمة).

    الوثائق: [Web tools](/tools/web).

  </Accordion>

  <Accordion title="config.apply مسح إعدادي. كيف أستعيده وأتجنب هذا؟">
    يستبدل `config.apply` **الإعداد بالكامل**. فإذا أرسلت كائنًا جزئيًا، فسيُزال كل
    شيء آخر.

    الاستعادة:

    - استعده من نسخة احتياطية (git أو نسخة من `~/.openclaw/openclaw.json`).
    - وإذا لم تكن لديك نسخة احتياطية، فأعد تشغيل `openclaw doctor` ثم أعد تكوين القنوات/النماذج.
    - إذا كان هذا غير متوقع، فافتح bug وأدرج آخر إعداد معروف لديك أو أي نسخة احتياطية.
    - يمكن لوكيل برمجي محلي في كثير من الأحيان إعادة بناء إعداد عامل من السجلات أو السجل السابق.

    تجنّب ذلك عبر:

    - استخدام `openclaw config set` للتغييرات الصغيرة.
    - استخدام `openclaw configure` للتعديلات التفاعلية.
    - استخدام `config.schema.lookup` أولًا عندما لا تكون متأكدًا من مسار أو شكل حقل دقيق؛ فهو يعيد عقدة schema سطحية إضافة إلى ملخصات الأبناء المباشرين للتعمق.
    - استخدم `config.patch` للتعديلات الجزئية عبر RPC؛ وأبقِ `config.apply` لاستبدال الإعداد الكامل فقط.
    - إذا كنت تستخدم أداة `gateway` الخاصة بوقت التشغيل والمخصصة للمالك فقط من داخل تشغيل وكيل، فستظل ترفض الكتابة إلى `tools.exec.ask` / `tools.exec.security` ‏(بما في ذلك الأسماء البديلة القديمة `tools.bash.*` التي تُطبّع إلى مسارات exec المحمية نفسها).

    الوثائق: [Config](/cli/config)، [Configure](/cli/configure)، [Doctor](/gateway/doctor).

  </Accordion>

  <Accordion title="كيف أشغّل Gateway مركزية مع workers متخصصين عبر أجهزة مختلفة؟">
    النمط الشائع هو **Gateway واحد** ‏(مثل Raspberry Pi) بالإضافة إلى **nodes** و**agents**:

    - **Gateway (مركزية):** تملك القنوات (Signal/WhatsApp)، والتوجيه، والجلسات.
    - **Nodes (الأجهزة):** تتصل Macs/iOS/Android كأجهزة طرفية وتعرض أدوات محلية (`system.run`, `canvas`, `camera`).
    - **Agents (workers):** عقول/مساحات عمل منفصلة لأدوار خاصة (مثل "Hetzner ops" أو "Personal data").
    - **الوكلاء الفرعيون:** Spawn لعمل خلفي من وكيل رئيسي عندما تريد التوازي.
    - **TUI:** اتصل بـ Gateway وبدّل بين الوكلاء/الجلسات.

    الوثائق: [Nodes](/nodes)، [Remote access](/gateway/remote)، [Multi-Agent Routing](/concepts/multi-agent)، [Sub-agents](/tools/subagents)، [TUI](/web/tui).

  </Accordion>

  <Accordion title="هل يمكن لمتصفح OpenClaw أن يعمل headless؟">
    نعم. هذا خيار في الإعداد:

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

    القيمة الافتراضية هي `false` ‏(headful). ويكون headless أكثر عرضة لتحفيز فحوصات مكافحة الروبوت في بعض المواقع. راجع [Browser](/tools/browser).

    يستخدم وضع headless **محرك Chromium نفسه** ويعمل في معظم الأتمتة (النماذج، والنقرات، والاستخلاص، وتسجيلات الدخول). والفروق الرئيسية هي:

    - لا توجد نافذة متصفح مرئية (استخدم لقطات شاشة إذا كنت بحاجة إلى مرئيات).
    - بعض المواقع أكثر تشددًا تجاه الأتمتة في وضع headless ‏(CAPTCHAs، ومكافحة الروبوت).
      فعلى سبيل المثال، تمنع X/Twitter غالبًا جلسات headless.

  </Accordion>

  <Accordion title="كيف أستخدم Brave للتحكم في المتصفح؟">
    اضبط `browser.executablePath` على ملف Brave التنفيذي لديك (أو أي متصفح مبني على Chromium) ثم أعد تشغيل Gateway.
    راجع أمثلة الإعداد الكاملة في [Browser](/tools/browser#use-brave-or-another-chromium-based-browser).
  </Accordion>
</AccordionGroup>

## Gateways البعيدة وnodes

<AccordionGroup>
  <Accordion title="كيف تنتقل الأوامر بين Telegram وgateway وnodes؟">
    تتعامل **gateway** مع رسائل Telegram. وتشغّل gateway الوكيل
    ثم تستدعي بعد ذلك nodes عبر **Gateway WebSocket** فقط عندما تحتاج إلى أداة node:

    Telegram → Gateway → Agent → `node.*` → Node → Gateway → Telegram

    لا ترى nodes حركة المزوّد الواردة؛ فهي لا تستقبل إلا استدعاءات node RPC.

  </Accordion>

  <Accordion title="كيف يمكن لوكيلي الوصول إلى حاسوبي إذا كانت Gateway مستضافة عن بُعد؟">
    باختصار: **اقرن حاسوبك كـ node**. تعمل Gateway في مكان آخر، لكنه يستطيع
    استدعاء أدوات `node.*` ‏(الشاشة، والكاميرا، والنظام) على جهازك المحلي عبر Gateway WebSocket.

    إعداد نموذجي:

    1. شغّل Gateway على المضيف الدائم التشغيل (VPS/خادم منزلي).
    2. ضع مضيف Gateway + حاسوبك على tailnet نفسها.
    3. تأكد من أن Gateway WS قابلة للوصول (tailnet bind أو SSH tunnel).
    4. افتح تطبيق macOS محليًا واتصل في وضع **Remote over SSH** ‏(أو عبر tailnet مباشرة)
       حتى يتمكن من التسجيل كـ node.
    5. وافق على node من Gateway:

       ```bash
       openclaw devices list
       openclaw devices approve <requestId>
       ```

    لا حاجة إلى جسر TCP منفصل؛ فالعقد تتصل عبر Gateway WebSocket.

    تذكير أمني: إقران node على macOS يسمح باستخدام `system.run` على ذلك الجهاز. لا
    تقرن إلا الأجهزة التي تثق بها، وراجع [Security](/gateway/security).

    الوثائق: [Nodes](/nodes)، [Gateway protocol](/gateway/protocol)، [macOS remote mode](/platforms/mac/remote)، [Security](/gateway/security).

  </Accordion>

  <Accordion title="Tailscale متصل لكنني لا أتلقى ردودًا. ماذا الآن؟">
    افحص الأساسيات:

    - هل Gateway تعمل؟ ‏`openclaw gateway status`
    - هل Gateway سليمة؟ ‏`openclaw status`
    - هل القناة سليمة؟ ‏`openclaw channels status`

    ثم تحقّق من المصادقة والتوجيه:

    - إذا كنت تستخدم Tailscale Serve، فتأكد من ضبط `gateway.auth.allowTailscale` بشكل صحيح.
    - إذا كنت تتصل عبر SSH tunnel، فتأكد من أن tunnel المحلية تعمل وتشير إلى المنفذ الصحيح.
    - تأكد من أن قوائم السماح لديك (DM أو المجموعة) تتضمن حسابك.

    الوثائق: [Tailscale](/gateway/tailscale)، [Remote access](/gateway/remote)، [Channels](/channels).

  </Accordion>

  <Accordion title="هل يمكن لمثيلي OpenClaw التحدث إلى بعضهما (محلي + VPS)؟">
    نعم. لا يوجد جسر "bot-to-bot" مضمّن، لكن يمكنك توصيلهما بعدة
    طرق موثوقة:

    **الأبسط:** استخدم قناة دردشة عادية يمكن لكلا البوتين الوصول إليها (Telegram/Slack/WhatsApp).
    اجعل Bot A يرسل رسالة إلى Bot B، ثم دع Bot B يرد كالمعتاد.

    **جسر CLI ‏(عام):** شغّل نصًا يستدعي Gateway الأخرى باستخدام
    `openclaw agent --message ... --deliver`، مستهدفًا دردشة يستمع فيها البوت الآخر.
    وإذا كان أحد البوتين موجودًا على VPS بعيد، فاجعل CLI الخاصة بك تشير إلى تلك Gateway البعيدة
    عبر SSH/Tailscale ‏(راجع [Remote access](/gateway/remote)).

    نمط مثال (يعمل من جهاز يمكنه الوصول إلى Gateway الهدف):

    ```bash
    openclaw agent --message "Hello from local bot" --deliver --channel telegram --reply-to <chat-id>
    ```

    نصيحة: أضف قاعدة حماية حتى لا يدخل البوتان في حلقة لا نهائية (mention-only، أو
    قوائم سماح القنوات، أو قاعدة "لا ترد على رسائل البوت").

    الوثائق: [Remote access](/gateway/remote)، [Agent CLI](/cli/agent)، [Agent send](/tools/agent-send).

  </Accordion>

  <Accordion title="هل أحتاج إلى VPS منفصلة لوكلاء متعددين؟">
    لا. يمكن لـ Gateway واحدة استضافة عدة وكلاء، لكل منهم مساحة عمله، وافتراضاته
    الخاصة بالنموذج، وتوجيهه. وهذا هو الإعداد الطبيعي وهو أرخص وأبسط بكثير من تشغيل
    VPS واحدة لكل وكيل.

    استخدم VPS منفصلة فقط عندما تحتاج إلى عزل صارم (حدود أمنية) أو إلى
    إعدادات مختلفة جدًا لا تريد مشاركتها. وإلا، فأبقِ Gateway واحدة
    واستخدم عدة وكلاء أو وكلاء فرعيين.

  </Accordion>

  <Accordion title="هل هناك فائدة من استخدام node على حاسوبي المحمول الشخصي بدل SSH من VPS؟">
    نعم - فالعقد هي الطريقة الأساسية للوصول إلى حاسوبك المحمول من Gateway بعيدة، وهي
    تفتح أكثر من مجرد وصول shell. تعمل Gateway على macOS/Linux ‏(وWindows عبر WSL2) وهي
    خفيفة (VPS صغيرة أو جهاز من فئة Raspberry Pi يكفي؛ و4 GB RAM أكثر من كافٍ)، لذا فالإعداد
    الشائع هو مضيف دائم التشغيل بالإضافة إلى حاسوبك المحمول كـ node.

    - **لا حاجة إلى SSH واردة.** تتصل العقد إلى Gateway WebSocket خارجيًا وتستخدم device pairing.
    - **تحكم أكثر أمانًا في التنفيذ.** يخضع `system.run` لقوائم السماح/الموافقات الخاصة بالعقدة على ذلك الحاسوب المحمول.
    - **أدوات جهاز أكثر.** تعرض العقد `canvas` و`camera` و`screen` بالإضافة إلى `system.run`.
    - **أتمتة المتصفح المحلية.** أبقِ Gateway على VPS، لكن شغّل Chrome محليًا عبر مضيف node على الحاسوب المحمول، أو اتصل بـ Chrome المحلية على المضيف عبر Chrome MCP.

    SSH مناسبة للوصول اليدوي إلى shell، لكن العقد أبسط لسير عمل الوكلاء المستمر
    وأتمتة الأجهزة.

    الوثائق: [Nodes](/nodes)، [Nodes CLI](/cli/nodes)، [Browser](/tools/browser).

  </Accordion>

  <Accordion title="هل تشغّل العقد خدمة gateway؟">
    لا. يجب أن تعمل **gateway واحدة** فقط لكل مضيف ما لم تكن تريد عمدًا تشغيل ملفات تعريف معزولة (راجع [Multiple gateways](/gateway/multiple-gateways)). أما العقد فهي أجهزة طرفية تتصل
    بـ gateway ‏(عقد iOS/Android، أو "وضع العقدة" في تطبيق menubar على macOS). وبالنسبة إلى مضيفات العقد headless والتحكم عبر CLI، راجع [Node host CLI](/cli/node).

    يلزم إعادة تشغيل كاملة لتغييرات `gateway` و`discovery` و`canvasHost`.

  </Accordion>

  <Accordion title="هل توجد طريقة API / RPC لتطبيق الإعداد؟">
    نعم.

    - `config.schema.lookup`: افحص شجرة إعداد فرعية واحدة مع عقدة schema سطحية، وUI hint مطابق، وملخصات الأبناء المباشرين قبل الكتابة
    - `config.get`: اجلب اللقطة الحالية + hash
    - `config.patch`: تحديث جزئي آمن (مفضّل لمعظم تعديلات RPC)
    - `config.apply`: تحقّق + استبدل الإعداد بالكامل، ثم أعد التشغيل
    - ما تزال أداة وقت التشغيل `gateway` المخصصة للمالك فقط ترفض إعادة كتابة `tools.exec.ask` / `tools.exec.security`؛ كما تُطبّع الأسماء البديلة القديمة `tools.bash.*` إلى مسارات exec المحمية نفسها

  </Accordion>

  <Accordion title="الحد الأدنى المنطقي من الإعداد لأول تثبيت">
    ```json5
    {
      agents: { defaults: { workspace: "~/.openclaw/workspace" } },
      channels: { whatsapp: { allowFrom: ["+15555550123"] } },
    }
    ```

    يضبط هذا مساحة عملك ويقيّد من يمكنه تشغيل البوت.

  </Accordion>

  <Accordion title="كيف أعد Tailscale على VPS وأتصل من Mac؟">
    الخطوات الدنيا:

    1. **ثبّت + سجّل الدخول على VPS**

       ```bash
       curl -fsSL https://tailscale.com/install.sh | sh
       sudo tailscale up
       ```

    2. **ثبّت + سجّل الدخول على Mac**
       - استخدم تطبيق Tailscale وسجّل الدخول إلى tailnet نفسها.
    3. **فعّل MagicDNS ‏(موصى به)**
       - في وحدة إدارة Tailscale، فعّل MagicDNS حتى تحصل VPS على اسم ثابت.
    4. **استخدم اسم مضيف tailnet**
       - SSH: ‏`ssh user@your-vps.tailnet-xxxx.ts.net`
       - Gateway WS: ‏`ws://your-vps.tailnet-xxxx.ts.net:18789`

    وإذا كنت تريد Control UI من دون SSH، فاستخدم Tailscale Serve على VPS:

    ```bash
    openclaw gateway --tailscale serve
    ```

    يُبقي هذا gateway مربوطة على loopback ويعرض HTTPS عبر Tailscale. راجع [Tailscale](/gateway/tailscale).

  </Accordion>

  <Accordion title="كيف أوصل node على Mac إلى Gateway بعيدة (Tailscale Serve)؟">
    يعرض Serve **Control UI + WS الخاصة بـ Gateway**. وتتصل العقد عبر نقطة نهاية Gateway WS نفسها.

    الإعداد الموصى به:

    1. **تأكد من أن VPS + Mac موجودان على tailnet نفسها**.
    2. **استخدم تطبيق macOS في الوضع البعيد** ‏(يمكن أن يكون هدف SSH هو اسم مضيف tailnet).
       وسيقوم التطبيق بعمل tunnel لمنفذ Gateway والاتصال بصفته node.
    3. **وافق على node** على gateway:

       ```bash
       openclaw devices list
       openclaw devices approve <requestId>
       ```

    الوثائق: [Gateway protocol](/gateway/protocol)، [Discovery](/gateway/discovery)، [macOS remote mode](/platforms/mac/remote).

  </Accordion>

  <Accordion title="هل يجب أن أثبّت على حاسوب محمول ثانٍ أم أضيف node فقط؟">
    إذا كنت تحتاج فقط إلى **أدوات محلية** ‏(screen/camera/exec) على الحاسوب الثاني، فأضفه كـ
    **node**. فهذا يبقي Gateway واحدة ويتجنب ازدواج الإعداد. أما أدوات العقد المحلية فهي
    حاليًا خاصة بـ macOS فقط، لكننا نخطط لتمديدها إلى أنظمة تشغيل أخرى.

    لا تثبّت Gateway ثانية إلا عندما تحتاج إلى **عزل صارم** أو إلى روبوتين منفصلين بالكامل.

    الوثائق: [Nodes](/nodes)، [Nodes CLI](/cli/nodes)، [Multiple gateways](/gateway/multiple-gateways).

  </Accordion>
</AccordionGroup>

## متغيرات البيئة وتحميل ‎.env

<AccordionGroup>
  <Accordion title="كيف يحمّل OpenClaw متغيرات البيئة؟">
    يقرأ OpenClaw متغيرات env من العملية الأب (shell، أو launchd/systemd، أو CI، إلخ) ويحمّل كذلك:

    - `.env` من دليل العمل الحالي
    - ملف `.env` احتياطيًا عامًا من `~/.openclaw/.env` ‏(أي `$OPENCLAW_STATE_DIR/.env`)

    لا يتجاوز أي من ملفي `.env` متغيرات env الموجودة بالفعل.

    ويمكنك أيضًا تعريف متغيرات env مضمنة داخل الإعداد (تُطبَّق فقط إذا كانت مفقودة من env الخاصة بالعملية):

    ```json5
    {
      env: {
        OPENROUTER_API_KEY: "sk-or-...",
        vars: { GROQ_API_KEY: "gsk-..." },
      },
    }
    ```

    راجع [/environment](/help/environment) لمعرفة الأسبقية الكاملة والمصادر.

  </Accordion>

  <Accordion title="بدأت Gateway عبر الخدمة واختفت متغيرات env. ماذا الآن؟">
    هناك إصلاحان شائعان:

    1. ضع المفاتيح المفقودة في `~/.openclaw/.env` حتى تُحمَّل حتى عندما لا ترث الخدمة env الخاصة بالـ shell.
    2. فعّل استيراد shell ‏(ميزة ملائمة اختيارية):

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

    يشغّل هذا shell تسجيل الدخول لديك ويستورد فقط المفاتيح المتوقعة المفقودة (ولا يتجاوز الموجود أبدًا). مكافئات env:
    `OPENCLAW_LOAD_SHELL_ENV=1`, `OPENCLAW_SHELL_ENV_TIMEOUT_MS=15000`.

  </Accordion>

  <Accordion title='لقد ضبطت COPILOT_GITHUB_TOKEN، لكن models status تعرض "Shell env: off." لماذا؟'>
    تعرض `openclaw models status` ما إذا كان **استيراد env من shell** مفعّلًا. و"Shell env: off"
    **لا** تعني أن متغيرات env مفقودة - بل تعني فقط أن OpenClaw لن تحمّل
    shell login الخاصة بك تلقائيًا.

    إذا كانت Gateway تعمل كخدمة (launchd/systemd)، فلن ترث env الخاصة بـ shell.
    أصلح ذلك بإحدى الطرق التالية:

    1. ضع token في `~/.openclaw/.env`:

       ```
       COPILOT_GITHUB_TOKEN=...
       ```

    2. أو فعّل استيراد shell ‏(`env.shellEnv.enabled: true`).
    3. أو أضفها إلى كتلة `env` داخل الإعداد (تُطبّق فقط إذا كانت مفقودة).

    ثم أعد تشغيل gateway وأعد التحقق:

    ```bash
    openclaw models status
    ```

    تُقرأ رموز Copilot من `COPILOT_GITHUB_TOKEN` ‏(وكذلك `GH_TOKEN` / `GITHUB_TOKEN`).
    راجع [/concepts/model-providers](/concepts/model-providers) و[/environment](/help/environment).

  </Accordion>
</AccordionGroup>

## الجلسات والدردشات المتعددة

<AccordionGroup>
  <Accordion title="كيف أبدأ محادثة جديدة؟">
    أرسل `/new` أو `/reset` كرسالة مستقلة. راجع [Session management](/concepts/session).
  </Accordion>

  <Accordion title="هل تُعاد الجلسات تلقائيًا إذا لم أرسل /new مطلقًا؟">
    يمكن أن تنتهي الجلسات بعد `session.idleMinutes`، لكن هذا **معطل افتراضيًا** (الافتراضي **0**).
    اضبطه إلى قيمة موجبة لتفعيل انتهاء الصلاحية عند الخمول. وعند التفعيل، فإن
    الرسالة **التالية** بعد فترة الخمول تبدأ معرّف جلسة جديدًا لذلك مفتاح الدردشة.
    هذا لا يحذف النصوص - بل يبدأ جلسة جديدة فقط.

    ```json5
    {
      session: {
        idleMinutes: 240,
      },
    }
    ```

  </Accordion>

  <Accordion title="هل توجد طريقة لصنع فريق من مثيلات OpenClaw (رئيس تنفيذي واحد وعدة وكلاء)؟">
    نعم، عبر **Multi-agent routing** و**الوكلاء الفرعيين**. يمكنك إنشاء وكيل منسق واحد
    وعدة وكلاء عاملين لهم مساحات عمل ونماذج خاصة بهم.

    ومع ذلك، فمن الأفضل النظر إلى هذا على أنه **تجربة ممتعة**. فهو يستهلك الكثير من tokens وغالبًا
    ما يكون أقل كفاءة من استخدام بوت واحد مع جلسات منفصلة. والنموذج المعتاد الذي
    نتصوره هو بوت واحد تتحدث إليه، مع جلسات مختلفة للعمل المتوازي. ويمكن لذلك
    البوت أيضًا أن ينشئ وكلاء فرعيين عند الحاجة.

    الوثائق: [Multi-agent routing](/concepts/multi-agent)، [Sub-agents](/tools/subagents)، [Agents CLI](/cli/agents).

  </Accordion>

  <Accordion title="لماذا قُطع السياق في منتصف المهمة؟ وكيف أمنع ذلك؟">
    سياق الجلسة محدود بنافذة النموذج. يمكن للمحادثات الطويلة، أو مخرجات الأدوات الكبيرة، أو كثرة
    الملفات، أن تؤدي إلى الضغط أو الاقتطاع.

    ما الذي يساعد:

    - اطلب من البوت تلخيص الحالة الحالية وكتابتها في ملف.
    - استخدم `/compact` قبل المهام الطويلة، و`/new` عند تغيير المواضيع.
    - أبقِ السياق المهم في مساحة العمل واطلب من البوت قراءته مجددًا.
    - استخدم وكلاء فرعيين للمهام الطويلة أو المتوازية حتى تبقى الدردشة الرئيسية أصغر.
    - اختر نموذجًا ذا نافذة سياق أكبر إذا كان هذا يحدث كثيرًا.

  </Accordion>

  <Accordion title="كيف أعيد ضبط OpenClaw بالكامل لكن مع إبقائه مثبتًا؟">
    استخدم أمر reset:

    ```bash
    openclaw reset
    ```

    إعادة ضبط كاملة غير تفاعلية:

    ```bash
    openclaw reset --scope full --yes --non-interactive
    ```

    ثم أعد تشغيل الإعداد:

    ```bash
    openclaw onboard --install-daemon
    ```

    ملاحظات:

    - يعرض الإعداد الأولي أيضًا خيار **Reset** إذا رأى إعدادًا موجودًا. راجع [Onboarding (CLI)](/ar/start/wizard).
    - إذا كنت تستخدم profiles ‏(`--profile` / `OPENCLAW_PROFILE`)، فأعد ضبط كل دليل حالة (الافتراضي هو `~/.openclaw-<profile>`).
    - إعادة ضبط التطوير: ‏`openclaw gateway --dev --reset` ‏(للتطوير فقط؛ تمسح إعداد dev + بيانات الاعتماد + الجلسات + مساحة العمل).

  </Accordion>

  <Accordion title='أتلقى أخطاء "context too large" - كيف أعيد الضبط أو أضغط؟'>
    استخدم أحد ما يلي:

    - **الضغط** ‏(يبقي المحادثة لكن يلخص الأدوار الأقدم):

      ```
      /compact
      ```

      أو `/compact <instructions>` لتوجيه الملخص.

    - **إعادة الضبط** ‏(معرّف جلسة جديد لمفتاح الدردشة نفسه):

      ```
      /new
      /reset
      ```

    إذا استمر ذلك في الحدوث:

    - فعّل أو اضبط **session pruning** ‏(`agents.defaults.contextPruning`) لقص مخرجات الأدوات القديمة.
    - استخدم نموذجًا ذا نافذة سياق أكبر.

    الوثائق: [Compaction](/concepts/compaction)، [Session pruning](/concepts/session-pruning)، [Session management](/concepts/session).

  </Accordion>

  <Accordion title='لماذا أرى "LLM request rejected: messages.content.tool_use.input field required"؟'>
    هذا خطأ تحقق من المزوّد: أصدر النموذج كتلة `tool_use` من دون
    `input` المطلوبة. وغالبًا ما يعني أن سجل الجلسة قديم أو فاسد (غالبًا بعد سلاسل طويلة
    أو تغيير في الأداة/schema).

    الحل: ابدأ جلسة جديدة باستخدام `/new` ‏(كرسالة مستقلة).

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
    مثل `# Heading`)، فإن OpenClaw تتخطى تشغيل heartbeat لتوفير استدعاءات API.
    وإذا كان الملف مفقودًا، فإن heartbeat تستمر في العمل ويقرر النموذج ما يجب فعله.

    تستخدم التجاوزات لكل وكيل `agents.list[].heartbeat`. الوثائق: [Heartbeat](/gateway/heartbeat).

  </Accordion>

  <Accordion title='هل أحتاج إلى إضافة "bot account" إلى مجموعة WhatsApp؟'>
    لا. يعمل OpenClaw على **حسابك أنت**، لذا إذا كنت موجودًا في المجموعة، يمكن لـ OpenClaw رؤيتها.
    افتراضيًا، تُحظر ردود المجموعات حتى تسمح للمرسلين (`groupPolicy: "allowlist"`).

    إذا كنت تريد أن تكون **أنت فقط** قادرًا على تشغيل الردود في المجموعات:

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

  <Accordion title="كيف أحصل على JID الخاصة بمجموعة WhatsApp؟">
    الخيار 1 ‏(الأسرع): تتبّع السجلات وأرسل رسالة اختبار في المجموعة:

    ```bash
    openclaw logs --follow --json
    ```

    ابحث عن `chatId` ‏(أو `from`) الذي ينتهي بـ `@g.us`، مثل:
    `1234567890-1234567890@g.us`.

    الخيار 2 ‏(إذا كانت مُعدّة/مسموحًا بها بالفعل): أدرج المجموعات من الإعداد:

    ```bash
    openclaw directory groups list --channel whatsapp
    ```

    الوثائق: [WhatsApp](/channels/whatsapp)، [Directory](/cli/directory)، [Logs](/cli/logs).

  </Accordion>

  <Accordion title="لماذا لا يرد OpenClaw في مجموعة؟">
    هناك سببان شائعان:

    - تقييد الإشارات مفعّل (افتراضيًا). يجب أن تقوم بعمل @mention للبوت (أو تطابق `mentionPatterns`).
    - قمت بتكوين `channels.whatsapp.groups` من دون `"*"` ولم تُدرج تلك المجموعة في قائمة السماح.

    راجع [Groups](/channels/groups) و[Group messages](/channels/group-messages).

  </Accordion>

  <Accordion title="هل تشترك المجموعات/السلاسل في السياق مع DMs؟">
    تنطوي الدردشات المباشرة على الجلسة الرئيسية افتراضيًا. أما المجموعات/القنوات فلها مفاتيح جلسات خاصة، كما أن موضوعات Telegram / سلاسل Discord هي جلسات منفصلة. راجع [Groups](/channels/groups) و[Group messages](/channels/group-messages).
  </Accordion>

  <Accordion title="كم عدد مساحات العمل والوكلاء التي يمكنني إنشاؤها؟">
    لا توجد حدود صارمة. فالعشرات (بل والمئات) مقبولة، لكن انتبه إلى:

    - **نمو القرص:** تعيش الجلسات + النصوص تحت `~/.openclaw/agents/<agentId>/sessions/`.
    - **تكلفة tokens:** المزيد من الوكلاء يعني استخدامًا متزامنًا أكبر للنماذج.
    - **عبء التشغيل:** auth profiles، ومساحات العمل، وتوجيه القنوات لكل وكيل.

    نصائح:

    - أبقِ مساحة عمل **نشطة** واحدة لكل وكيل (`agents.defaults.workspace`).
    - قلّم الجلسات القديمة (احذف ملفات JSONL أو إدخالات المخزن) إذا نما استخدام القرص.
    - استخدم `openclaw doctor` لاكتشاف مساحات العمل الشاردة وعدم تطابق profiles.

  </Accordion>

  <Accordion title="هل يمكنني تشغيل عدة بوتات أو محادثات في الوقت نفسه (Slack)، وكيف يجب أن أضبط ذلك؟">
    نعم. استخدم **Multi-Agent Routing** لتشغيل عدة وكلاء معزولين وتوجيه الرسائل الواردة حسب
    القناة/الحساب/peer. Slack مدعومة كقناة ويمكن ربطها بوكلاء محددين.

    الوصول إلى المتصفح قوي، لكنه ليس "افعل كل ما يستطيع الإنسان فعله" - فمكافحة الروبوت، وCAPTCHAs، وMFA قد
    تظل تمنع الأتمتة. ولأوثق تحكم في المتصفح، استخدم Chrome MCP المحلية على المضيف،
    أو استخدم CDP على الجهاز الذي يشغّل المتصفح فعليًا.

    أفضل إعداد عملي:

    - مضيف Gateway دائم التشغيل (VPS/Mac mini).
    - وكيل واحد لكل دور (bindings).
    - ربط قنوات Slack بتلك الوكلاء.
    - متصفح محلي عبر Chrome MCP أو node عند الحاجة.

    الوثائق: [Multi-Agent Routing](/concepts/multi-agent)، [Slack](/channels/slack)،
    [Browser](/tools/browser)، [Nodes](/nodes).

  </Accordion>
</AccordionGroup>

## النماذج: الافتراضيات، والاختيار، والأسماء البديلة، والتبديل

<AccordionGroup>
  <Accordion title='ما هو "النموذج الافتراضي"؟'>
    النموذج الافتراضي في OpenClaw هو أي نموذج تضبطه على الشكل:

    ```
    agents.defaults.model.primary
    ```

    يشار إلى النماذج بصيغة `provider/model` ‏(مثال: `openai/gpt-5.4`). وإذا حذفت المزوّد، فإن OpenClaw تحاول أولًا اسمًا بديلًا، ثم تطابقًا فريدًا لمزوّد مكوَّن لذلك المعرّف الدقيق للنموذج، وبعدها فقط تعود إلى المزوّد الافتراضي المكوَّن باعتباره مسار توافق قديمًا. وإذا لم يعد ذلك المزوّد يعرض النموذج الافتراضي المكوَّن، تعود OpenClaw إلى أول مزوّد/نموذج مُعد بدل إظهار قيمة افتراضية قديمة من مزوّد أزيل. ومع ذلك ينبغي لك **صراحةً** ضبط `provider/model`.

  </Accordion>

  <Accordion title="ما النموذج الذي توصي به؟">
    **الافتراضي الموصى به:** استخدم أقوى نموذج متاح من الجيل الأحدث في مجموعة مزوّديك.
    **بالنسبة إلى الوكلاء المزوّدين بالأدوات أو المدخلات غير الموثوقة:** قدّم قوة النموذج على التكلفة.
    **بالنسبة إلى الدردشة الروتينية/منخفضة المخاطر:** استخدم نماذج احتياطية أرخص ووجّه حسب دور الوكيل.

    لدى MiniMax وثائقها الخاصة: [MiniMax](/providers/minimax) و
    [النماذج المحلية](/gateway/local-models).

    قاعدة عامة: استخدم **أفضل نموذج يمكنك تحمّل تكلفته** للأعمال عالية المخاطر، واستخدم نموذجًا أرخص
    للدردشة الروتينية أو الملخصات. ويمكنك توجيه النماذج لكل وكيل واستخدام وكلاء فرعيين
    لتنفيذ المهام الطويلة بالتوازي (كل وكيل فرعي يستهلك tokens). راجع [Models](/concepts/models) و
    [Sub-agents](/tools/subagents).

    تحذير قوي: النماذج الأضعف/المكمّمة بشكل مفرط أكثر عرضة لحقن
    الموجّهات والسلوك غير الآمن. راجع [Security](/gateway/security).

    مزيد من السياق: [Models](/concepts/models).

  </Accordion>

  <Accordion title="كيف أبدّل النماذج من دون مسح الإعداد؟">
    استخدم **أوامر النماذج** أو عدّل فقط حقول **النموذج**. وتجنب الاستبدال الكامل للإعداد.

    الخيارات الآمنة:

    - `/model` في الدردشة (سريع، ولكل جلسة)
    - `openclaw models set ...` ‏(يحدّث إعداد النموذج فقط)
    - `openclaw configure --section model` ‏(تفاعلي)
    - عدّل `agents.defaults.model` في `~/.openclaw/openclaw.json`

    تجنّب `config.apply` مع كائن جزئي ما لم تكن تقصد استبدال الإعداد بالكامل.
    وبالنسبة إلى تعديلات RPC، افحص أولًا باستخدام `config.schema.lookup` وفضّل `config.patch`. وتمنحك حمولة lookup المسار المطبع، ووثائق/قيود schema السطحية، وملخصات الأبناء المباشرين.
    للتحديثات الجزئية.
    وإذا قمت فعلًا بالكتابة فوق الإعداد، فاستعده من نسخة احتياطية أو أعد تشغيل `openclaw doctor` لإصلاحه.

    الوثائق: [Models](/concepts/models)، [Configure](/cli/configure)، [Config](/cli/config)، [Doctor](/gateway/doctor).

  </Accordion>

  <Accordion title="هل يمكنني استخدام نماذج مستضافة ذاتيًا (llama.cpp, vLLM, Ollama)؟">
    نعم. تعد Ollama أسهل طريق للنماذج المحلية.

    أسرع إعداد:

    1. ثبّت Ollama من `https://ollama.com/download`
    2. اسحب نموذجًا محليًا مثل `ollama pull glm-4.7-flash`
    3. إذا كنت تريد نماذج سحابية أيضًا، فشغّل `ollama signin`
    4. شغّل `openclaw onboard` واختر `Ollama`
    5. اختر `Local` أو `Cloud + Local`

    ملاحظات:

    - يمنحك `Cloud + Local` النماذج السحابية بالإضافة إلى نماذج Ollama المحلية
    - لا تحتاج معرّفات النماذج السحابية مثل `kimi-k2.5:cloud` إلى سحب محلي
    - للتبديل اليدوي، استخدم `openclaw models list` و`openclaw models set ollama/<model>`

    ملاحظة أمنية: النماذج الأصغر أو المكمّمة بشدة أكثر عرضة لحقن
    الموجّهات. ونحن نوصي بشدة باستخدام **نماذج كبيرة** لأي بوت يستطيع استخدام الأدوات.
    وإذا كنت لا تزال تريد نماذج صغيرة، ففعّل sandboxing وقوائم سماح أدوات صارمة.

    الوثائق: [Ollama](/providers/ollama)، [النماذج المحلية](/gateway/local-models)،
    [Model providers](/concepts/model-providers)، [Security](/gateway/security)،
    [Sandboxing](/gateway/sandboxing).

  </Accordion>

  <Accordion title="ما النماذج التي يستخدمها OpenClaw وFlawd وKrill؟">
    - قد تختلف هذه البيئات وقد تتغير بمرور الوقت؛ ولا توجد توصية ثابتة بمزوّد واحد.
    - تحقّق من إعداد وقت التشغيل الحالي على كل gateway باستخدام `openclaw models status`.
    - بالنسبة إلى الوكلاء ذوي الحساسية الأمنية/المزوّدين بالأدوات، استخدم أقوى نموذج متاح من الجيل الأحدث.
  </Accordion>

  <Accordion title="كيف أبدّل النماذج أثناء التشغيل (من دون إعادة التشغيل)؟">
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

    هذه هي الأسماء البديلة المضمنة. ويمكن إضافة أسماء بديلة مخصصة عبر `agents.defaults.models`.

    يمكنك سرد النماذج المتاحة باستخدام `/model` أو `/model list` أو `/model status`.

    يعرض `/model` ‏(و`/model list`) منتقيًا مرقمًا مدمجًا. اختر حسب الرقم:

    ```
    /model 3
    ```

    يمكنك أيضًا فرض auth profile محددة للمزوّد (لكل جلسة):

    ```
    /model opus@anthropic:default
    /model opus@anthropic:work
    ```

    نصيحة: يعرض `/model status` الوكيل النشط، وملف `auth-profiles.json` المستخدم، وauth profile التي ستُجرَّب تاليًا.
    كما يعرض endpoint المزوّد المكوّن (`baseUrl`) ووضع API ‏(`api`) عند توفرهما.

    **كيف ألغي تثبيت profile قمت بتعيينها باستخدام ‎@profile؟**

    أعد تشغيل `/model` **من دون** اللاحقة `@profile`:

    ```
    /model anthropic/claude-opus-4-6
    ```

    وإذا كنت تريد العودة إلى الافتراضي، فاختره من `/model` ‏(أو أرسل `/model <default provider/model>`).
    واستخدم `/model status` للتأكد من auth profile النشطة.

  </Accordion>

  <Accordion title="هل يمكنني استخدام GPT 5.2 للمهام اليومية وCodex 5.3 للبرمجة؟">
    نعم. اضبط أحدهما كافتراضي وبدّل حسب الحاجة:

    - **تبديل سريع (لكل جلسة):** ‏`/model gpt-5.4` للمهام اليومية، و`/model openai-codex/gpt-5.4` للبرمجة باستخدام Codex OAuth.
    - **افتراضي + تبديل:** اضبط `agents.defaults.model.primary` على `openai/gpt-5.4`، ثم بدّل إلى `openai-codex/gpt-5.4` عند البرمجة (أو بالعكس).
    - **الوكلاء الفرعيون:** وجّه مهام البرمجة إلى وكلاء فرعيين يملكون نموذجًا افتراضيًا مختلفًا.

    راجع [Models](/concepts/models) و[Slash commands](/tools/slash-commands).

  </Accordion>

  <Accordion title='لماذا أرى "Model ... is not allowed" ثم لا يوجد رد؟'>
    إذا ضُبط `agents.defaults.models`، فإنه يصبح **قائمة السماح** لأوامر `/model` ولكل
    override على مستوى الجلسة. واختيار نموذج غير موجود في تلك القائمة يعيد:

    ```
    Model "provider/model" is not allowed. Use /model to list available models.
    ```

    ويُعاد هذا الخطأ **بدلًا من** الرد الطبيعي. الحل: أضف النموذج إلى
    `agents.defaults.models`، أو أزل قائمة السماح، أو اختر نموذجًا من `/model list`.

  </Accordion>

  <Accordion title='لماذا أرى "Unknown model: minimax/MiniMax-M2.7"؟'>
    هذا يعني أن **المزوّد غير مكوَّن** ‏(لم يُعثر على إعداد MiniMax provider أو auth
    profile)، لذا لا يمكن حل النموذج.

    قائمة الحل:

    1. حدّث إلى إصدار حديث من OpenClaw ‏(أو شغّل من مصدر `main`)، ثم أعد تشغيل gateway.
    2. تأكد من أن MiniMax مهيأة (عبر المعالج أو JSON)، أو أن مصادقة MiniMax
       موجودة في env/auth profiles حتى يمكن حقن المزوّد المطابق
       (`MINIMAX_API_KEY` لـ `minimax`، و`MINIMAX_OAUTH_TOKEN` أو OAuth MiniMax المخزنة
       لـ `minimax-portal`).
    3. استخدم معرّف النموذج الدقيق (مع مراعاة حالة الأحرف) لمسار المصادقة لديك:
       `minimax/MiniMax-M2.7` أو `minimax/MiniMax-M2.7-highspeed` لإعداد
       API-key، أو `minimax-portal/MiniMax-M2.7` /
       `minimax-portal/MiniMax-M2.7-highspeed` لإعداد OAuth.
    4. شغّل:

       ```bash
       openclaw models list
       ```

       واختر من القائمة (أو `/model list` في الدردشة).

    راجع [MiniMax](/providers/minimax) و[Models](/concepts/models).

  </Accordion>

  <Accordion title="هل يمكنني استخدام MiniMax كافتراضي وOpenAI للمهام المعقدة؟">
    نعم. استخدم **MiniMax كافتراضي** وبدّل النماذج **لكل جلسة** عند الحاجة.
    فالبدائل الاحتياطية مخصصة **للأخطاء**، لا "للمهام الصعبة"، لذا استخدم `/model` أو وكيلًا منفصلًا.

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

    - وكيل A الافتراضي: MiniMax
    - وكيل B الافتراضي: OpenAI
    - وجّه حسب الوكيل أو استخدم `/agent` للتبديل

    الوثائق: [Models](/concepts/models)، [Multi-Agent Routing](/concepts/multi-agent)، [MiniMax](/providers/minimax)، [OpenAI](/providers/openai).

  </Accordion>

  <Accordion title="هل الاختصارات opus / sonnet / gpt مضمّنة؟">
    نعم. يأتي OpenClaw مع بعض الاختصارات الافتراضية (ولا تُطبّق إلا عندما يكون النموذج موجودًا في `agents.defaults.models`):

    - `opus` → `anthropic/claude-opus-4-6`
    - `sonnet` → `anthropic/claude-sonnet-4-6`
    - `gpt` → `openai/gpt-5.4`
    - `gpt-mini` → `openai/gpt-5.4-mini`
    - `gpt-nano` → `openai/gpt-5.4-nano`
    - `gemini` → `google/gemini-3.1-pro-preview`
    - `gemini-flash` → `google/gemini-3