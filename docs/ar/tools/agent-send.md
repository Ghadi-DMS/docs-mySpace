---
read_when:
    - تريد تشغيل الوكيل من السكريبتات أو من سطر الأوامر
    - تحتاج إلى تسليم ردود الوكيل إلى قناة دردشة برمجيًا
summary: شغّل أدوار الوكيل من CLI وسلّم الردود اختياريًا إلى القنوات
title: إرسال الوكيل
x-i18n:
    generated_at: "2026-04-05T12:57:09Z"
    model: gpt-5.4
    provider: openai
    source_hash: 42ea2977e89fb28d2afd07e5f6b1560ad627aea8b72fde36d8e324215c710afc
    source_path: tools/agent-send.md
    workflow: 15
---

# إرسال الوكيل

يشغّل `openclaw agent` دور وكيل واحد من سطر الأوامر دون الحاجة إلى
رسالة دردشة واردة. استخدمه لمسارات العمل المؤتمتة، والاختبار،
والتسليم البرمجي.

## البدء السريع

<Steps>
  <Step title="تشغيل دور وكيل بسيط">
    ```bash
    openclaw agent --message "What is the weather today?"
    ```

    يرسل هذا الرسالة عبر Gateway ويطبع الرد.

  </Step>

  <Step title="استهداف وكيل أو جلسة محددة">
    ```bash
    # استهداف وكيل محدد
    openclaw agent --agent ops --message "Summarize logs"

    # استهداف رقم هاتف (لاشتقاق مفتاح الجلسة)
    openclaw agent --to +15555550123 --message "Status update"

    # إعادة استخدام جلسة موجودة
    openclaw agent --session-id abc123 --message "Continue the task"
    ```

  </Step>

  <Step title="تسليم الرد إلى قناة">
    ```bash
    # التسليم إلى WhatsApp (القناة الافتراضية)
    openclaw agent --to +15555550123 --message "Report ready" --deliver

    # التسليم إلى Slack
    openclaw agent --agent ops --message "Generate report" \
      --deliver --reply-channel slack --reply-to "#reports"
    ```

  </Step>
</Steps>

## العلامات

| العلامة                      | الوصف                                                        |
| ----------------------------- | ----------------------------------------------------------- |
| `--message \<text\>`          | الرسالة المراد إرسالها (مطلوبة)                             |
| `--to \<dest\>`               | اشتقاق مفتاح الجلسة من هدف (هاتف، معرّف دردشة)               |
| `--agent \<id\>`              | استهداف وكيل مهيأ (يستخدم جلسته `main`)                     |
| `--session-id \<id\>`         | إعادة استخدام جلسة موجودة حسب المعرّف                       |
| `--local`                     | فرض runtime المحلي المضمّن (تخطي Gateway)                   |
| `--deliver`                   | إرسال الرد إلى قناة دردشة                                   |
| `--channel \<name\>`          | قناة التسليم (whatsapp، telegram، discord، slack، إلخ)      |
| `--reply-to \<target\>`       | تجاوز هدف التسليم                                           |
| `--reply-channel \<name\>`    | تجاوز قناة التسليم                                          |
| `--reply-account \<id\>`      | تجاوز معرّف حساب التسليم                                    |
| `--thinking \<level\>`        | تعيين مستوى التفكير (off, minimal, low, medium, high, xhigh) |
| `--verbose \<on\|full\|off\>` | تعيين مستوى verbose                                         |
| `--timeout \<seconds\>`       | تجاوز مهلة الوكيل                                            |
| `--json`                      | إخراج JSON منظّم                                            |

## السلوك

- افتراضيًا، يمر CLI **عبر Gateway**. أضف `--local` لفرض
  runtime المضمّن المحلي على الجهاز الحالي.
- إذا تعذر الوصول إلى Gateway، فإن CLI **يرجع احتياطيًا**
  إلى التشغيل المحلي المضمّن.
- اختيار الجلسة: يشتق `--to` مفتاح الجلسة (أهداف المجموعات/القنوات
  تحافظ على العزل؛ بينما تُدمج الدردشات المباشرة في `main`).
- تستمر علامتا التفكير وverbose داخل مخزن الجلسة.
- الإخراج: نص عادي افتراضيًا، أو `--json` لحمولة منظّمة + بيانات وصفية.

## أمثلة

```bash
# دور بسيط مع إخراج JSON
openclaw agent --to +15555550123 --message "Trace logs" --verbose on --json

# دور مع مستوى التفكير
openclaw agent --session-id 1234 --message "Summarize inbox" --thinking medium

# التسليم إلى قناة مختلفة عن الجلسة
openclaw agent --agent ops --message "Alert" --deliver --reply-channel telegram --reply-to "@admin"
```

## ذو صلة

- [مرجع CLI للوكيل](/cli/agent)
- [Sub-agents](/tools/subagents) — إنشاء وكلاء فرعيين في الخلفية
- [الجلسات](/ar/concepts/session) — كيفية عمل مفاتيح الجلسات
