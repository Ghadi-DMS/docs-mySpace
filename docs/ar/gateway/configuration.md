---
read_when:
    - إعداد OpenClaw للمرة الأولى
    - البحث عن أنماط إعداد شائعة
    - الانتقال إلى أقسام إعدادات محددة
summary: 'نظرة عامة على الإعدادات: المهام الشائعة، والإعداد السريع، وروابط المرجع الكامل'
title: الإعدادات
x-i18n:
    generated_at: "2026-04-05T12:43:39Z"
    model: gpt-5.4
    provider: openai
    source_hash: a39a7de09c5f9540785ec67f37d435a7a86201f0f5f640dae663054f35976712
    source_path: gateway/configuration.md
    workflow: 15
---

# الإعدادات

يقرأ OpenClaw إعدادات <Tooltip tip="يدعم JSON5 التعليقات والفواصل اللاحقة">**JSON5**</Tooltip> اختيارية من `~/.openclaw/openclaw.json`.

إذا كان الملف مفقودًا، يستخدم OpenClaw إعدادات افتراضية آمنة. ومن الأسباب الشائعة لإضافة إعدادات:

- توصيل القنوات والتحكم فيمن يمكنه مراسلة البوت
- ضبط النماذج، والأدوات، وsandboxing، أو الأتمتة (`cron` و`hooks`)
- ضبط الجلسات، والوسائط، والشبكات، أو واجهة المستخدم

راجع [المرجع الكامل](/gateway/configuration-reference) لكل حقل متاح.

<Tip>
**جديد على الإعدادات؟** ابدأ بـ `openclaw onboard` للإعداد التفاعلي، أو اطّلع على دليل [أمثلة الإعدادات](/gateway/configuration-examples) للحصول على إعدادات كاملة جاهزة للنسخ واللصق.
</Tip>

## إعدادات دنيا

```json5
// ~/.openclaw/openclaw.json
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
  channels: { whatsapp: { allowFrom: ["+15555550123"] } },
}
```

## تعديل الإعدادات

<Tabs>
  <Tab title="المعالج التفاعلي">
    ```bash
    openclaw onboard       # full onboarding flow
    openclaw configure     # config wizard
    ```
  </Tab>
  <Tab title="CLI (أوامر مختصرة)">
    ```bash
    openclaw config get agents.defaults.workspace
    openclaw config set agents.defaults.heartbeat.every "2h"
    openclaw config unset plugins.entries.brave.config.webSearch.apiKey
    ```
  </Tab>
  <Tab title="Control UI">
    افتح [http://127.0.0.1:18789](http://127.0.0.1:18789) واستخدم علامة التبويب **Config**.
    تعرض Control UI نموذجًا من مخطط الإعدادات المباشر، بما في ذلك بيانات التوثيق الوصفية
    `title` / `description` بالإضافة إلى مخططات plugins والقنوات عندما تكون
    متاحة، مع محرر **Raw JSON** كخيار احتياطي. وبالنسبة إلى واجهات
    التفصيل والأدوات الأخرى، يكشف gateway أيضًا `config.schema.lookup` من أجل
    جلب عقدة مخطط واحدة ضمن نطاق مسار واحد مع ملخصات الأبناء المباشرين.
  </Tab>
  <Tab title="تعديل مباشر">
    عدّل `~/.openclaw/openclaw.json` مباشرة. يراقب Gateway الملف ويطبّق التغييرات تلقائيًا (راجع [إعادة التحميل السريع](#config-hot-reload)).
  </Tab>
</Tabs>

## التحقق الصارم

<Warning>
لا يقبل OpenClaw إلا الإعدادات التي تطابق المخطط بالكامل. فالمفاتيح غير المعروفة، أو الأنواع المشوّهة، أو القيم غير الصالحة تجعل Gateway **يرفض البدء**. والاستثناء الوحيد على مستوى الجذر هو `$schema` ‏(سلسلة نصية)، حتى تتمكن المحررات من إرفاق بيانات JSON Schema الوصفية.
</Warning>

ملاحظات أدوات المخطط:

- يطبع `openclaw config schema` عائلة JSON Schema نفسها المستخدمة بواسطة Control UI
  والتحقق من الإعدادات.
- تُنقل قيمتا `title` و`description` للحقل إلى خرج المخطط من أجل
  المحررات وأدوات النماذج.
- ترث إدخالات الكائنات المتداخلة، والبدل الشامل (`*`)، وعناصر المصفوفات (`[]`)
  بيانات التوثيق الوصفية نفسها حيث توجد وثائق حقول مطابقة.
- ترث فروع التركيب `anyOf` / `oneOf` / `allOf` أيضًا بيانات التوثيق
  الوصفية نفسها، بحيث تحتفظ متغيرات union/intersection بمساعدة الحقول نفسها.
- يعيد `config.schema.lookup` مسار إعدادات واحدًا مطبّعًا مع عقدة مخطط
  سطحية (`title` و`description` و`type` و`enum` و`const` والحدود الشائعة
  وحقول تحقق مشابهة)، وبيانات تلميحات واجهة مستخدم مطابقة، وملخصات
  الأبناء المباشرين لأدوات التفصيل.
- يتم دمج مخططات plugin/channel وقت التشغيل عندما يتمكن gateway من تحميل
  سجل manifest الحالي.

عندما يفشل التحقق:

- لا يقلع Gateway
- تعمل فقط أوامر التشخيص (`openclaw doctor` و`openclaw logs` و`openclaw health` و`openclaw status`)
- شغّل `openclaw doctor` لرؤية المشكلات الدقيقة
- شغّل `openclaw doctor --fix` ‏(أو `--yes`) لتطبيق الإصلاحات

## المهام الشائعة

<AccordionGroup>
  <Accordion title="إعداد قناة (WhatsApp، Telegram، Discord، إلخ)">
    لكل قناة قسم إعدادات خاص بها تحت `channels.<provider>`. راجع صفحة القناة المخصصة لخطوات الإعداد:

    - [WhatsApp](/channels/whatsapp) — `channels.whatsapp`
    - [Telegram](/channels/telegram) — `channels.telegram`
    - [Discord](/channels/discord) — `channels.discord`
    - [Feishu](/channels/feishu) — `channels.feishu`
    - [Google Chat](/channels/googlechat) — `channels.googlechat`
    - [Microsoft Teams](/channels/msteams) — `channels.msteams`
    - [Slack](/channels/slack) — `channels.slack`
    - [Signal](/channels/signal) — `channels.signal`
    - [iMessage](/channels/imessage) — `channels.imessage`
    - [Mattermost](/channels/mattermost) — `channels.mattermost`

    تشترك جميع القنوات في نمط سياسة الرسائل المباشرة نفسه:

    ```json5
    {
      channels: {
        telegram: {
          enabled: true,
          botToken: "123:abc",
          dmPolicy: "pairing",   // pairing | allowlist | open | disabled
          allowFrom: ["tg:123"], // only for allowlist/open
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="اختيار النماذج وإعدادها">
    اضبط النموذج الأساسي والتراجعات الاختيارية:

    ```json5
    {
      agents: {
        defaults: {
          model: {
            primary: "anthropic/claude-sonnet-4-6",
            fallbacks: ["openai/gpt-5.4"],
          },
          models: {
            "anthropic/claude-sonnet-4-6": { alias: "Sonnet" },
            "openai/gpt-5.4": { alias: "GPT" },
          },
        },
      },
    }
    ```

    - يعرّف `agents.defaults.models` فهرس النماذج ويعمل كقائمة سماح لـ `/model`.
    - تستخدم مراجع النماذج تنسيق `provider/model` ‏(مثل `anthropic/claude-opus-4-6`).
    - يتحكم `agents.defaults.imageMaxDimensionPx` في تصغير صور السجل النصي/الأدوات (الافتراضي `1200`)؛ وتؤدي القيم الأقل عادةً إلى تقليل استخدام رموز الرؤية في التشغيلات الكثيفة لصور الشاشة.
    - راجع [Models CLI](/concepts/models) لتبديل النماذج في الدردشة و[Model Failover](/concepts/model-failover) لسلوك تدوير المصادقة والتراجع.
    - بالنسبة إلى المزوّدين المخصصين/المستضافين ذاتيًا، راجع [المزوّدون المخصصون](/gateway/configuration-reference#custom-providers-and-base-urls) في المرجع.

  </Accordion>

  <Accordion title="التحكم فيمن يمكنه مراسلة البوت">
    يتم التحكم في الوصول إلى الرسائل المباشرة لكل قناة عبر `dmPolicy`:

    - `"pairing"` (الافتراضي): يحصل المرسلون غير المعروفين على رمز اقتران لمرة واحدة للموافقة
    - `"allowlist"`: فقط المرسلون الموجودون في `allowFrom` ‏(أو مخزن السماح المقترن)
    - `"open"`: السماح بجميع الرسائل المباشرة الواردة (يتطلب `allowFrom: ["*"]`)
    - `"disabled"`: تجاهل جميع الرسائل المباشرة

    بالنسبة إلى المجموعات، استخدم `groupPolicy` + `groupAllowFrom` أو قوائم السماح الخاصة بالقناة.

    راجع [المرجع الكامل](/gateway/configuration-reference#dm-and-group-access) لتفاصيل كل قناة.

  </Accordion>

  <Accordion title="إعداد بوابة الإشارات في دردشات المجموعات">
    تكون رسائل المجموعات افتراضيًا **تتطلب إشارة**. قم بإعداد الأنماط لكل وكيل:

    ```json5
    {
      agents: {
        list: [
          {
            id: "main",
            groupChat: {
              mentionPatterns: ["@openclaw", "openclaw"],
            },
          },
        ],
      },
      channels: {
        whatsapp: {
          groups: { "*": { requireMention: true } },
        },
      },
    }
    ```

    - **إشارات البيانات الوصفية**: إشارات @ الأصلية (WhatsApp tap-to-mention، وTelegram @bot، إلخ)
    - **أنماط النص**: أنماط regex آمنة في `mentionPatterns`
    - راجع [المرجع الكامل](/gateway/configuration-reference#group-chat-mention-gating) لتجاوزات كل قناة ووضع الدردشة الذاتية.

  </Accordion>

  <Accordion title="تقييد Skills لكل وكيل">
    استخدم `agents.defaults.skills` لخط أساس مشترك، ثم تجاوز
    الوكلاء المحددين باستخدام `agents.list[].skills`:

    ```json5
    {
      agents: {
        defaults: {
          skills: ["github", "weather"],
        },
        list: [
          { id: "writer" }, // inherits github, weather
          { id: "docs", skills: ["docs-search"] }, // replaces defaults
          { id: "locked-down", skills: [] }, // no skills
        ],
      },
    }
    ```

    - احذف `agents.defaults.skills` للحصول على Skills غير مقيّدة افتراضيًا.
    - احذف `agents.list[].skills` للوراثة من الإعدادات الافتراضية.
    - اضبط `agents.list[].skills: []` لعدم وجود Skills.
    - راجع [Skills](/tools/skills)، و[إعدادات Skills](/tools/skills-config)، و
      [مرجع الإعدادات](/gateway/configuration-reference#agentsdefaultsskills).

  </Accordion>

  <Accordion title="ضبط مراقبة صحة قنوات gateway">
    تحكّم في مدى شدة إعادة تشغيل gateway للقنوات التي تبدو قديمة:

    ```json5
    {
      gateway: {
        channelHealthCheckMinutes: 5,
        channelStaleEventThresholdMinutes: 30,
        channelMaxRestartsPerHour: 10,
      },
      channels: {
        telegram: {
          healthMonitor: { enabled: false },
          accounts: {
            alerts: {
              healthMonitor: { enabled: true },
            },
          },
        },
      },
    }
    ```

    - اضبط `gateway.channelHealthCheckMinutes: 0` لتعطيل عمليات إعادة التشغيل الناتجة عن مراقبة الصحة عالميًا.
    - يجب أن تكون `channelStaleEventThresholdMinutes` أكبر من أو مساوية لفاصل التحقق.
    - استخدم `channels.<provider>.healthMonitor.enabled` أو `channels.<provider>.accounts.<id>.healthMonitor.enabled` لتعطيل إعادة التشغيل التلقائي لقناة أو حساب واحد من دون تعطيل المراقب العام.
    - راجع [فحوصات الصحة](/gateway/health) لتصحيح العمليات، و[المرجع الكامل](/gateway/configuration-reference#gateway) لجميع الحقول.

  </Accordion>

  <Accordion title="إعداد الجلسات وإعادة الضبط">
    تتحكم الجلسات في استمرارية المحادثة والعزل:

    ```json5
    {
      session: {
        dmScope: "per-channel-peer",  // recommended for multi-user
        threadBindings: {
          enabled: true,
          idleHours: 24,
          maxAgeHours: 0,
        },
        reset: {
          mode: "daily",
          atHour: 4,
          idleMinutes: 120,
        },
      },
    }
    ```

    - `dmScope`: ‏`main` ‏(مشترك) | `per-peer` | `per-channel-peer` | `per-account-channel-peer`
    - `threadBindings`: إعدادات افتراضية عامة لتوجيه الجلسات المرتبطة بالسلاسل (يدعم Discord الأوامر `/focus` و`/unfocus` و`/agents` و`/session idle` و`/session max-age`).
    - راجع [إدارة الجلسات](/concepts/session) للنطاقات وروابط الهوية وسياسة الإرسال.
    - راجع [المرجع الكامل](/gateway/configuration-reference#session) لجميع الحقول.

  </Accordion>

  <Accordion title="تمكين sandboxing">
    شغّل جلسات الوكيل داخل حاويات Docker معزولة:

    ```json5
    {
      agents: {
        defaults: {
          sandbox: {
            mode: "non-main",  // off | non-main | all
            scope: "agent",    // session | agent | shared
          },
        },
      },
    }
    ```

    أنشئ الصورة أولًا: `scripts/sandbox-setup.sh`

    راجع [Sandboxing](/gateway/sandboxing) للدليل الكامل و[المرجع الكامل](/gateway/configuration-reference#agentsdefaultssandbox) لجميع الخيارات.

  </Accordion>

  <Accordion title="تمكين push المدعوم عبر relay للإصدارات الرسمية من iOS">
    يتم إعداد push المدعوم عبر relay في `openclaw.json`.

    اضبط هذا في إعدادات gateway:

    ```json5
    {
      gateway: {
        push: {
          apns: {
            relay: {
              baseUrl: "https://relay.example.com",
              // Optional. Default: 10000
              timeoutMs: 10000,
            },
          },
        },
      },
    }
    ```

    مكافئ CLI:

    ```bash
    openclaw config set gateway.push.apns.relay.baseUrl https://relay.example.com
    ```

    ما الذي يفعله هذا:

    - يتيح لـ gateway إرسال `push.test` وطلبات التنبيه wake nudges وتنبيهات إعادة الاتصال عبر relay الخارجي.
    - يستخدم منح إرسال ضمن نطاق التسجيل تعيد توجيهه تطبيقات iOS المقترنة. ولا يحتاج gateway إلى رمز relay عام على مستوى النشر.
    - يربط كل تسجيل مدعوم عبر relay بهوية gateway التي اقترن بها تطبيق iOS، بحيث لا يستطيع gateway آخر إعادة استخدام التسجيل المخزن.
    - يُبقي إصدارات iOS المحلية/اليدوية على APNs المباشر. وتنطبق عمليات الإرسال المدعومة عبر relay فقط على الإصدارات الرسمية الموزعة التي سجّلت عبر relay.
    - يجب أن يطابق عنوان URL الأساسي الخاص بـ relay المضمّن في إصدار iOS الرسمي/TestFlight، حتى تصل حركة التسجيل والإرسال إلى نشر relay نفسه.

    التدفق الكامل:

    1. ثبّت إصدار iOS رسميًا/TestFlight تم تجميعه بعنوان URL أساسي لـ relay نفسه.
    2. اضبط `gateway.push.apns.relay.baseUrl` على gateway.
    3. اقترن بتطبيق iOS مع gateway ودع جلسات node وoperator تتصل.
    4. يجلب تطبيق iOS هوية gateway، ويسجّل مع relay باستخدام App Attest وإيصال التطبيق، ثم ينشر حمولة `push.apns.register` المدعومة عبر relay إلى gateway المقترن.
    5. يخزن gateway مقبض relay ومنح الإرسال، ثم يستخدمهما في `push.test` وطلبات التنبيه وتنبيهات إعادة الاتصال.

    ملاحظات تشغيلية:

    - إذا بدّلت تطبيق iOS إلى gateway مختلف، فأعد توصيل التطبيق حتى يتمكن من نشر تسجيل relay جديد مرتبط بذلك gateway.
    - إذا أصدرت إصدار iOS جديدًا يشير إلى نشر relay مختلف، فسيقوم التطبيق بتحديث تسجيل relay المخبأ بدلًا من إعادة استخدام مصدر relay القديم.

    ملاحظة التوافق:

    - لا يزال كل من `OPENCLAW_APNS_RELAY_BASE_URL` و`OPENCLAW_APNS_RELAY_TIMEOUT_MS` يعملان كتجاوزات بيئة مؤقتة.
    - يظل `OPENCLAW_APNS_RELAY_ALLOW_HTTP=true` منفذ هروب تطويريًا محصورًا بـ loopback فقط؛ لا تحفظ عناوين URL لـ relay عبر HTTP في الإعدادات.

    راجع [تطبيق iOS](/platforms/ios#relay-backed-push-for-official-builds) للتدفق الكامل و[تدفق المصادقة والثقة](/platforms/ios#authentication-and-trust-flow) لنموذج أمان relay.

  </Accordion>

  <Accordion title="إعداد heartbeat (تسجيلات دورية)">
    ```json5
    {
      agents: {
        defaults: {
          heartbeat: {
            every: "30m",
            target: "last",
          },
        },
      },
    }
    ```

    - `every`: سلسلة مدة (`30m` و`2h`). اضبط `0m` للتعطيل.
    - `target`: ‏`last` | `none` | `<channel-id>` ‏(مثل `discord` أو `matrix` أو `telegram` أو `whatsapp`)
    - `directPolicy`: ‏`allow` ‏(الافتراضي) أو `block` لأهداف heartbeat من نمط الرسائل المباشرة
    - راجع [Heartbeat](/gateway/heartbeat) للدليل الكامل.

  </Accordion>

  <Accordion title="إعداد مهام cron">
    ```json5
    {
      cron: {
        enabled: true,
        maxConcurrentRuns: 2,
        sessionRetention: "24h",
        runLog: {
          maxBytes: "2mb",
          keepLines: 2000,
        },
      },
    }
    ```

    - `sessionRetention`: تقليم جلسات التشغيل المعزولة المكتملة من `sessions.json` ‏(الافتراضي `24h`؛ اضبط `false` للتعطيل).
    - `runLog`: تقليم `cron/runs/<jobId>.jsonl` حسب الحجم والأسطر المحتفظ بها.
    - راجع [مهام Cron](/automation/cron-jobs) لنظرة عامة على الميزات وأمثلة CLI.

  </Accordion>

  <Accordion title="إعداد webhooks (hooks)">
    فعّل نقاط نهاية HTTP webhook على Gateway:

    ```json5
    {
      hooks: {
        enabled: true,
        token: "shared-secret",
        path: "/hooks",
        defaultSessionKey: "hook:ingress",
        allowRequestSessionKey: false,
        allowedSessionKeyPrefixes: ["hook:"],
        mappings: [
          {
            match: { path: "gmail" },
            action: "agent",
            agentId: "main",
            deliver: true,
          },
        ],
      },
    }
    ```

    ملاحظة أمان:
    - تعامل مع كل محتوى حمولة hook/webhook على أنه إدخال غير موثوق.
    - استخدم `hooks.token` مخصصًا؛ لا تعِد استخدام رمز Gateway المشترك.
    - تكون مصادقة hook عبر الترويسات فقط (`Authorization: Bearer ...` أو `x-openclaw-token`)؛ وتُرفض الرموز الموجودة في سلسلة الاستعلام.
    - لا يمكن أن يكون `hooks.path` هو `/`؛ أبقِ إدخال webhook على مسار فرعي مخصص مثل `/hooks`.
    - أبقِ أعلام تجاوز المحتوى غير الآمن معطلة (`hooks.gmail.allowUnsafeExternalContent` و`hooks.mappings[].allowUnsafeExternalContent`) ما لم تكن تجري تصحيحًا محكم النطاق.
    - إذا فعّلت `hooks.allowRequestSessionKey`، فاضبط أيضًا `hooks.allowedSessionKeyPrefixes` لتقييد مفاتيح الجلسات التي يحددها المتصل.
    - بالنسبة إلى الوكلاء المدفوعين عبر hook، فضّل مستويات النماذج الحديثة القوية وسياسة الأدوات الصارمة (على سبيل المثال الرسائل فقط مع sandboxing حيثما أمكن).

    راجع [المرجع الكامل](/gateway/configuration-reference#hooks) لجميع خيارات mappings وتكامل Gmail.

  </Accordion>

  <Accordion title="إعداد التوجيه متعدد الوكلاء">
    شغّل عدة وكلاء معزولين مع مساحات عمل وجلسات منفصلة:

    ```json5
    {
      agents: {
        list: [
          { id: "home", default: true, workspace: "~/.openclaw/workspace-home" },
          { id: "work", workspace: "~/.openclaw/workspace-work" },
        ],
      },
      bindings: [
        { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
        { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } },
      ],
    }
    ```

    راجع [Multi-Agent](/concepts/multi-agent) و[المرجع الكامل](/gateway/configuration-reference#multi-agent-routing) لقواعد الارتباط وملفات الوصول لكل وكيل.

  </Accordion>

  <Accordion title="تقسيم الإعدادات إلى عدة ملفات ($include)">
    استخدم `$include` لتنظيم ملفات الإعدادات الكبيرة:

    ```json5
    // ~/.openclaw/openclaw.json
    {
      gateway: { port: 18789 },
      agents: { $include: "./agents.json5" },
      broadcast: {
        $include: ["./clients/a.json5", "./clients/b.json5"],
      },
    }
    ```

    - **ملف واحد**: يستبدل الكائن الحاوي
    - **مصفوفة ملفات**: تُدمج دمجًا عميقًا بالترتيب (الأخير ينتصر)
    - **المفاتيح المجاورة**: تُدمج بعد عمليات التضمين (وتتجاوز القيم المضمّنة)
    - **التضمينات المتداخلة**: مدعومة حتى عمق 10 مستويات
    - **المسارات النسبية**: تُحل نسبةً إلى الملف الذي يتضمنها
    - **التعامل مع الأخطاء**: أخطاء واضحة للملفات المفقودة، وأخطاء التحليل، والتضمينات الدائرية

  </Accordion>
</AccordionGroup>

## إعادة التحميل السريع للإعدادات

يراقب Gateway الملف `~/.openclaw/openclaw.json` ويطبّق التغييرات تلقائيًا — ولا حاجة إلى إعادة تشغيل يدوية لمعظم الإعدادات.

### أوضاع إعادة التحميل

| الوضع                   | السلوك                                                                                  |
| ---------------------- | --------------------------------------------------------------------------------------- |
| **`hybrid`** (الافتراضي) | يطبّق التغييرات الآمنة مباشرة. ويعيد التشغيل تلقائيًا للتغييرات الحرجة.                |
| **`hot`**              | يطبّق التغييرات الآمنة فقط. ويسجل تحذيرًا عند الحاجة إلى إعادة تشغيل — وأنت تتولى الأمر. |
| **`restart`**          | يعيد تشغيل Gateway عند أي تغيير في الإعدادات، سواء كان آمنًا أم لا.                    |
| **`off`**              | يعطّل مراقبة الملفات. وتدخل التغييرات حيز التنفيذ عند إعادة التشغيل اليدوية التالية.   |

```json5
{
  gateway: {
    reload: { mode: "hybrid", debounceMs: 300 },
  },
}
```

### ما الذي يُطبَّق مباشرة وما الذي يحتاج إلى إعادة تشغيل

تُطبّق معظم الحقول مباشرة من دون توقف. وفي وضع `hybrid`، تتم معالجة التغييرات التي تتطلب إعادة تشغيل تلقائيًا.

| الفئة               | الحقول                                                               | هل يلزم إعادة تشغيل؟ |
| ------------------- | -------------------------------------------------------------------- | -------------------- |
| القنوات            | `channels.*` و`web` ‏(WhatsApp) — جميع القنوات المضمنة وقنوات plugins | لا                   |
| الوكيل والنماذج      | `agent` و`agents` و`models` و`routing`                               | لا                   |
| الأتمتة            | `hooks` و`cron` و`agent.heartbeat`                                   | لا                   |
| الجلسات والرسائل    | `session` و`messages`                                                | لا                   |
| الأدوات والوسائط    | `tools` و`browser` و`skills` و`audio` و`talk`                        | لا                   |
| واجهة المستخدم ومتفرقات | `ui` و`logging` و`identity` و`bindings`                              | لا                   |
| خادم Gateway       | `gateway.*` ‏(المنفذ، والربط، والمصادقة، وtailscale، وTLS، وHTTP)     | **نعم**              |
| البنية التحتية      | `discovery` و`canvasHost` و`plugins`                                 | **نعم**              |

<Note>
يُعد كل من `gateway.reload` و`gateway.remote` استثناءين — فالتغيير فيهما **لا** يؤدي إلى إعادة تشغيل.
</Note>

## Config RPC ‏(تحديثات برمجية)

<Note>
تخضع استدعاءات RPC الخاصة بالكتابة في مستوى التحكم (`config.apply` و`config.patch` و`update.run`) لحد معدل يبلغ **3 طلبات لكل 60 ثانية** لكل `deviceId+clientIp`. وعند تطبيق الحد، يعيد RPC القيمة `UNAVAILABLE` مع `retryAfterMs`.
</Note>

التدفق الآمن/الافتراضي:

- `config.schema.lookup`: فحص شجرة إعدادات واحدة ضمن نطاق مسار واحد مع عقدة
  مخطط سطحية، وبيانات تلميحات مطابقة، وملخصات الأبناء المباشرين
- `config.get`: جلب اللقطة الحالية + hash
- `config.patch`: المسار المفضل للتحديث الجزئي
- `config.apply`: استبدال الإعدادات الكاملة فقط
- `update.run`: تحديث ذاتي صريح + إعادة تشغيل

عندما لا تستبدل الإعدادات بالكامل، فافضّل `config.schema.lookup`
ثم `config.patch`.

<AccordionGroup>
  <Accordion title="config.apply (استبدال كامل)">
    يتحقق من الإعدادات الكاملة ويكتبها ثم يعيد تشغيل Gateway في خطوة واحدة.

    <Warning>
    يستبدل `config.apply` **الإعدادات بالكامل**. استخدم `config.patch` للتحديثات الجزئية، أو `openclaw config set` للمفاتيح المفردة.
    </Warning>

    المعلمات:

    - `raw` ‏(string) — حمولة JSON5 للإعدادات الكاملة
    - `baseHash` ‏(اختياري) — hash الإعدادات من `config.get` ‏(مطلوب عندما تكون الإعدادات موجودة)
    - `sessionKey` ‏(اختياري) — مفتاح الجلسة لتنبيه wake-up بعد إعادة التشغيل
    - `note` ‏(اختياري) — ملاحظة لـ restart sentinel
    - `restartDelayMs` ‏(اختياري) — تأخير قبل إعادة التشغيل (الافتراضي 2000)

    يتم دمج طلبات إعادة التشغيل بينما يكون أحدها معلقًا/قيد التنفيذ بالفعل، ويُطبَّق تبريد لمدة 30 ثانية بين دورات إعادة التشغيل.

    ```bash
    openclaw gateway call config.get --params '{}'  # capture payload.hash
    openclaw gateway call config.apply --params '{
      "raw": "{ agents: { defaults: { workspace: \"~/.openclaw/workspace\" } } }",
      "baseHash": "<hash>",
      "sessionKey": "agent:main:whatsapp:direct:+15555550123"
    }'
    ```

  </Accordion>

  <Accordion title="config.patch (تحديث جزئي)">
    يدمج تحديثًا جزئيًا في الإعدادات الحالية (وفق دلالات JSON merge patch):

    - تُدمج الكائنات بشكل متكرر
    - تحذف `null` المفتاح
    - تستبدل المصفوفات

    المعلمات:

    - `raw` ‏(string) — JSON5 يحتوي فقط على المفاتيح المطلوب تغييرها
    - `baseHash` ‏(مطلوب) — hash الإعدادات من `config.get`
    - `sessionKey` و`note` و`restartDelayMs` — مثل `config.apply`

    يطابق سلوك إعادة التشغيل الأمر `config.apply`: إعادة التشغيلات المعلقة المدمجة بالإضافة إلى تبريد 30 ثانية بين دورات إعادة التشغيل.

    ```bash
    openclaw gateway call config.patch --params '{
      "raw": "{ channels: { telegram: { groups: { \"*\": { requireMention: false } } } } }",
      "baseHash": "<hash>"
    }'
    ```

  </Accordion>
</AccordionGroup>

## متغيرات البيئة

يقرأ OpenClaw متغيرات البيئة من العملية الأصلية بالإضافة إلى:

- `.env` من دليل العمل الحالي (إن وجد)
- `~/.openclaw/.env` ‏(تراجع عام)

لا يقوم أي من الملفين بتجاوز متغيرات البيئة الموجودة. ويمكنك أيضًا ضبط متغيرات بيئة مضمنة في الإعدادات:

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    vars: { GROQ_API_KEY: "gsk-..." },
  },
}
```

<Accordion title="استيراد بيئة shell (اختياري)">
  إذا كان مفعّلًا ولم تكن المفاتيح المتوقعة مضبوطة، فإن OpenClaw يشغّل shell تسجيل الدخول الخاص بك ويستورد فقط المفاتيح المفقودة:

```json5
{
  env: {
    shellEnv: { enabled: true, timeoutMs: 15000 },
  },
}
```

مكافئ متغير البيئة: `OPENCLAW_LOAD_SHELL_ENV=1`
</Accordion>

<Accordion title="استبدال متغيرات البيئة داخل قيم الإعدادات">
  أشر إلى متغيرات البيئة في أي قيمة نصية داخل الإعدادات باستخدام `${VAR_NAME}`:

```json5
{
  gateway: { auth: { token: "${OPENCLAW_GATEWAY_TOKEN}" } },
  models: { providers: { custom: { apiKey: "${CUSTOM_API_KEY}" } } },
}
```

القواعد:

- تتم مطابقة الأسماء الكبيرة فقط: `[A-Z_][A-Z0-9_]*`
- تؤدي المتغيرات المفقودة/الفارغة إلى خطأ وقت التحميل
- استخدم `$${VAR}` للحصول على خرج حرفي
- يعمل داخل ملفات `$include`
- استبدال مضمن: `"${BASE}/v1"` → `"https://api.example.com/v1"`

</Accordion>

<Accordion title="مراجع الأسرار (env وfile وexec)">
  بالنسبة إلى الحقول التي تدعم كائنات SecretRef، يمكنك استخدام:

```json5
{
  models: {
    providers: {
      openai: { apiKey: { source: "env", provider: "default", id: "OPENAI_API_KEY" } },
    },
  },
  skills: {
    entries: {
      "image-lab": {
        apiKey: {
          source: "file",
          provider: "filemain",
          id: "/skills/entries/image-lab/apiKey",
        },
      },
    },
  },
  channels: {
    googlechat: {
      serviceAccountRef: {
        source: "exec",
        provider: "vault",
        id: "channels/googlechat/serviceAccount",
      },
    },
  },
}
```

توجد تفاصيل SecretRef ‏(بما في ذلك `secrets.providers` الخاصة بـ `env`/`file`/`exec`) في [إدارة الأسرار](/gateway/secrets).
وتُسرد مسارات بيانات الاعتماد المدعومة في [سطح بيانات اعتماد SecretRef](/reference/secretref-credential-surface).
</Accordion>

راجع [البيئة](/help/environment) للأولوية الكاملة والمصادر.

## المرجع الكامل

للحصول على مرجع كامل لكل حقل، راجع **[مرجع الإعدادات](/gateway/configuration-reference)**.

---

_ذو صلة: [أمثلة الإعدادات](/gateway/configuration-examples) · [مرجع الإعدادات](/gateway/configuration-reference) · [Doctor](/gateway/doctor)_
