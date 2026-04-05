---
read_when:
    - الإعداد لأول مرة من الصفر
    - تريد أسرع طريق إلى محادثة تعمل بالفعل
summary: ثبّت OpenClaw وابدأ أول محادثة لك خلال دقائق.
title: البدء
x-i18n:
    generated_at: "2026-04-05T11:36:39Z"
    model: gpt-5.4
    provider: openai
    source_hash: c43eee6f0d3f593e3cf0767bfacb3e0ae38f51a2615d594303786ae1d4a6d2c3
    source_path: start/getting-started.md
    workflow: 15
---

# البدء

ثبّت OpenClaw، وشغّل الإعداد التمهيدي، وابدأ الدردشة مع مساعدك المدعوم بالذكاء الاصطناعي — كل ذلك خلال نحو 5 دقائق. في النهاية سيكون لديك Gateway قيد التشغيل، ومصادقة مُعدّة، وجلسة دردشة عاملة.

## ما الذي تحتاج إليه

- **Node.js** — يُوصى بـ Node 24 (كما أن Node 22.14+ مدعوم أيضًا)
- **مفتاح API** من موفّر نماذج (Anthropic أو OpenAI أو Google أو غيرها) — سيطلبه منك الإعداد التمهيدي

<Tip>
تحقق من إصدار Node لديك باستخدام `node --version`.
**لمستخدمي Windows:** كل من Windows الأصلي وWSL2 مدعومان. يُعد WSL2 أكثر استقرارًا ويُنصح به للحصول على التجربة الكاملة. راجع [Windows](/platforms/windows).
هل تحتاج إلى تثبيت Node؟ راجع [إعداد Node](/install/node).
</Tip>

## إعداد سريع

<Steps>
  <Step title="ثبّت OpenClaw">
    <Tabs>
      <Tab title="macOS / Linux">
        ```bash
        curl -fsSL https://openclaw.ai/install.sh | bash
        ```
        <img
  src="/assets/install-script.svg"
  alt="عملية برنامج التثبيت النصي"
  className="rounded-lg"
/>
      </Tab>
      <Tab title="Windows (PowerShell)">
        ```powershell
        iwr -useb https://openclaw.ai/install.ps1 | iex
        ```
      </Tab>
    </Tabs>

    <Note>
    طرق تثبيت أخرى (Docker وNix وnpm): [التثبيت](/install).
    </Note>

  </Step>
  <Step title="شغّل الإعداد التمهيدي">
    ```bash
    openclaw onboard --install-daemon
    ```

    يرشدك المعالج خلال اختيار موفّر نموذج، وتعيين مفتاح API، وتهيئة Gateway. يستغرق ذلك نحو دقيقتين.

    راجع [الإعداد التمهيدي (CLI)](/start/wizard) للحصول على المرجع الكامل.

  </Step>
  <Step title="تحقق من أن Gateway يعمل">
    ```bash
    openclaw gateway status
    ```

    يجب أن ترى أن Gateway يستمع على المنفذ 18789.

  </Step>
  <Step title="افتح لوحة التحكم">
    ```bash
    openclaw dashboard
    ```

    سيؤدي هذا إلى فتح واجهة التحكم في متصفحك. إذا تم تحميلها، فهذا يعني أن كل شيء يعمل.

  </Step>
  <Step title="أرسل أول رسالة لك">
    اكتب رسالة في دردشة واجهة التحكم، ويُفترض أن تحصل على رد من الذكاء الاصطناعي.

    هل تريد الدردشة من هاتفك بدلًا من ذلك؟ أسرع قناة للإعداد هي
    [Telegram](/channels/telegram) (مجرد رمز bot مميز). راجع [القنوات](/channels)
    للاطلاع على جميع الخيارات.

  </Step>
</Steps>

<Accordion title="متقدم: تحميل إصدار مخصص من واجهة التحكم">
  إذا كنت تحتفظ بإصدار مترجم أو مخصص من لوحة التحكم، فوجّه
  `gateway.controlUi.root` إلى دليل يحتوي على الأصول الثابتة
  المضمّنة و`index.html`.

```bash
mkdir -p "$HOME/.openclaw/control-ui-custom"
# انسخ ملفاتك الثابتة المضمّنة إلى ذلك الدليل.
```

ثم عيّن:

```json
{
  "gateway": {
    "controlUi": {
      "enabled": true,
      "root": "$HOME/.openclaw/control-ui-custom"
    }
  }
}
```

أعد تشغيل gateway ثم أعد فتح لوحة التحكم:

```bash
openclaw gateway restart
openclaw dashboard
```

</Accordion>

## ماذا تفعل بعد ذلك

<Columns>
  <Card title="صِل قناة" href="/channels" icon="message-square">
    Discord وFeishu وiMessage وMatrix وMicrosoft Teams وSignal وSlack وTelegram وWhatsApp وZalo والمزيد.
  </Card>
  <Card title="الاقتران والأمان" href="/channels/pairing" icon="shield">
    تحكم فيمن يمكنه مراسلة وكيلك.
  </Card>
  <Card title="هيّئ Gateway" href="/gateway/configuration" icon="settings">
    النماذج والأدوات وsandbox والإعدادات المتقدمة.
  </Card>
  <Card title="تصفح الأدوات" href="/tools" icon="wrench">
    المتصفح وexec والبحث على الويب وSkills والإضافات.
  </Card>
</Columns>

<Accordion title="متقدم: متغيرات البيئة">
  إذا كنت تشغّل OpenClaw كحساب خدمة أو تريد مسارات مخصصة:

- `OPENCLAW_HOME` — الدليل الرئيسي لحل المسارات الداخلية
- `OPENCLAW_STATE_DIR` — تجاوز دليل الحالة
- `OPENCLAW_CONFIG_PATH` — تجاوز مسار ملف الإعدادات

المرجع الكامل: [متغيرات البيئة](/help/environment).
</Accordion>
