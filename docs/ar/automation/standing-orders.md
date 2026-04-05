---
read_when:
    - عند إعداد مهام سير عمل لوكلاء ذاتيين تعمل من دون مطالبة لكل مهمة
    - عند تحديد ما الذي يمكن للوكيل فعله بشكل مستقل مقابل ما يحتاج إلى موافقة بشرية
    - عند تنظيم وكلاء متعددَي البرامج بحدود واضحة وقواعد تصعيد
summary: تحديد سلطة تشغيل دائمة لبرامج الوكلاء الذاتيين
title: الأوامر الدائمة
x-i18n:
    generated_at: "2026-04-05T12:34:25Z"
    model: gpt-5.4
    provider: openai
    source_hash: 81347d7a51a6ce20e6493277afee92073770f69a91a2e6b3bf87b99bb586d038
    source_path: automation/standing-orders.md
    workflow: 15
---

# الأوامر الدائمة

تمنح الأوامر الدائمة وكيلك **سلطة تشغيل دائمة** للبرامج المحددة. وبدلًا من إعطاء تعليمات لكل مهمة على حدة في كل مرة، فإنك تعرّف برامج ذات نطاق واضح ومشغلات وقواعد تصعيد — وينفذ الوكيل العمل ذاتيًا ضمن تلك الحدود.

وهذا هو الفرق بين أن تقول لمساعدك "أرسل التقرير الأسبوعي" كل يوم جمعة، وبين منحه سلطة دائمة: "أنت مسؤول عن التقرير الأسبوعي. اجمعه كل يوم جمعة، وأرسله، ولا تصعّد إلا إذا بدا أن هناك شيئًا غير صحيح."

## لماذا الأوامر الدائمة؟

**من دون أوامر دائمة:**

- يجب عليك مطالبة الوكيل بكل مهمة
- يظل الوكيل في وضع خمول بين الطلبات
- يُنسى العمل الروتيني أو يتأخر
- تصبح أنت عنق الزجاجة

**مع الأوامر الدائمة:**

- ينفذ الوكيل العمل ذاتيًا ضمن حدود محددة
- يتم إنجاز العمل الروتيني في موعده من دون مطالبة
- لا تتدخل إلا في الاستثناءات وحالات الموافقة
- يستثمر الوكيل وقت الخمول بشكل منتج

## كيف تعمل

تُعرَّف الأوامر الدائمة في ملفات [مساحة عمل الوكيل](/concepts/agent-workspace) الخاصة بك. والنهج الموصى به هو تضمينها مباشرةً في `AGENTS.md` (الذي يُحقن تلقائيًا في كل جلسة) حتى تكون دائمًا ضمن سياق الوكيل. وبالنسبة إلى الإعدادات الأكبر، يمكنك أيضًا وضعها في ملف مخصص مثل `standing-orders.md` والإشارة إليه من `AGENTS.md`.

يحدد كل برنامج ما يلي:

1. **النطاق** — ما الذي يملك الوكيل صلاحية القيام به
2. **المشغلات** — متى يتم التنفيذ (جدول زمني أو حدث أو شرط)
3. **بوابات الموافقة** — ما الذي يتطلب اعتمادًا بشريًا قبل التنفيذ
4. **قواعد التصعيد** — متى يجب التوقف وطلب المساعدة

يحمّل الوكيل هذه التعليمات في كل جلسة عبر ملفات تمهيد مساحة العمل (راجع [مساحة عمل الوكيل](/concepts/agent-workspace) للحصول على القائمة الكاملة للملفات المحقونة تلقائيًا) وينفذ وفقًا لها، مع دمجها مع [مهام cron](/automation/cron-jobs) لفرض التنفيذ المعتمد على الوقت.

<Tip>
ضع الأوامر الدائمة في `AGENTS.md` لضمان تحميلها في كل جلسة. يقوم تمهيد مساحة العمل تلقائيًا بحقن `AGENTS.md` و`SOUL.md` و`TOOLS.md` و`IDENTITY.md` و`USER.md` و`HEARTBEAT.md` و`BOOTSTRAP.md` و`MEMORY.md` — لكنه لا يحقن الملفات العشوائية داخل الأدلة الفرعية.
</Tip>

## مكونات الأمر الدائم

```markdown
## Program: Weekly Status Report

**Authority:** Compile data, generate report, deliver to stakeholders
**Trigger:** Every Friday at 4 PM (enforced via cron job)
**Approval gate:** None for standard reports. Flag anomalies for human review.
**Escalation:** If data source is unavailable or metrics look unusual (>2σ from norm)

### Execution Steps

1. Pull metrics from configured sources
2. Compare to prior week and targets
3. Generate report in Reports/weekly/YYYY-MM-DD.md
4. Deliver summary via configured channel
5. Log completion to Agent/Logs/

### What NOT to Do

- Do not send reports to external parties
- Do not modify source data
- Do not skip delivery if metrics look bad — report accurately
```

## الأوامر الدائمة + مهام Cron

تحدد الأوامر الدائمة **ما الذي** يملك الوكيل صلاحية فعله. وتحدد [مهام Cron](/automation/cron-jobs) **متى** يحدث ذلك. وهما يعملان معًا:

```
Standing Order: "You own the daily inbox triage"
    ↓
Cron Job (8 AM daily): "Execute inbox triage per standing orders"
    ↓
Agent: Reads standing orders → executes steps → reports results
```

يجب أن تشير رسالة مهمة cron إلى الأمر الدائم بدلًا من تكراره:

```bash
openclaw cron add \
  --name daily-inbox-triage \
  --cron "0 8 * * 1-5" \
  --tz America/New_York \
  --timeout-seconds 300 \
  --announce \
  --channel bluebubbles \
  --to "+1XXXXXXXXXX" \
  --message "Execute daily inbox triage per standing orders. Check mail for new alerts. Parse, categorize, and persist each item. Report summary to owner. Escalate unknowns."
```

## أمثلة

### المثال 1: المحتوى ووسائل التواصل الاجتماعي (دورة أسبوعية)

```markdown
## Program: Content & Social Media

**Authority:** Draft content, schedule posts, compile engagement reports
**Approval gate:** All posts require owner review for first 30 days, then standing approval
**Trigger:** Weekly cycle (Monday review → mid-week drafts → Friday brief)

### Weekly Cycle

- **Monday:** Review platform metrics and audience engagement
- **Tuesday–Thursday:** Draft social posts, create blog content
- **Friday:** Compile weekly marketing brief → deliver to owner

### Content Rules

- Voice must match the brand (see SOUL.md or brand voice guide)
- Never identify as AI in public-facing content
- Include metrics when available
- Focus on value to audience, not self-promotion
```

### المثال 2: العمليات المالية (تعمل بالمحفزات الحدثية)

```markdown
## Program: Financial Processing

**Authority:** Process transaction data, generate reports, send summaries
**Approval gate:** None for analysis. Recommendations require owner approval.
**Trigger:** New data file detected OR scheduled monthly cycle

### When New Data Arrives

1. Detect new file in designated input directory
2. Parse and categorize all transactions
3. Compare against budget targets
4. Flag: unusual items, threshold breaches, new recurring charges
5. Generate report in designated output directory
6. Deliver summary to owner via configured channel

### Escalation Rules

- Single item > $500: immediate alert
- Category > budget by 20%: flag in report
- Unrecognizable transaction: ask owner for categorization
- Failed processing after 2 retries: report failure, do not guess
```

### المثال 3: المراقبة والتنبيهات (مستمرة)

```markdown
## Program: System Monitoring

**Authority:** Check system health, restart services, send alerts
**Approval gate:** Restart services automatically. Escalate if restart fails twice.
**Trigger:** Every heartbeat cycle

### Checks

- Service health endpoints responding
- Disk space above threshold
- Pending tasks not stale (>24 hours)
- Delivery channels operational

### Response Matrix

| Condition        | Action                   | Escalate?                |
| ---------------- | ------------------------ | ------------------------ |
| Service down     | Restart automatically    | Only if restart fails 2x |
| Disk space < 10% | Alert owner              | Yes                      |
| Stale task > 24h | Remind owner             | No                       |
| Channel offline  | Log and retry next cycle | If offline > 2 hours     |
```

## نمط التنفيذ-التحقق-الإبلاغ

تعمل الأوامر الدائمة بأفضل شكل عند دمجها مع انضباط صارم في التنفيذ. يجب أن تتبع كل مهمة في أمر دائم هذه الحلقة:

1. **التنفيذ** — نفّذ العمل الفعلي (لا تكتفِ بالإقرار بالتعليمات)
2. **التحقق** — أكّد أن النتيجة صحيحة (الملف موجود، الرسالة تم تسليمها، البيانات جرى تحليلها)
3. **الإبلاغ** — أخبر المالك بما تم إنجازه وما الذي تم التحقق منه

```markdown
### Execution Rules

- Every task follows Execute-Verify-Report. No exceptions.
- "I'll do that" is not execution. Do it, then report.
- "Done" without verification is not acceptable. Prove it.
- If execution fails: retry once with adjusted approach.
- If still fails: report failure with diagnosis. Never silently fail.
- Never retry indefinitely — 3 attempts max, then escalate.
```

يمنع هذا النمط أكثر أوضاع فشل الوكلاء شيوعًا: الإقرار بالمهمة من دون إكمالها.

## بنية متعددة البرامج

بالنسبة إلى الوكلاء الذين يديرون عدة مجالات، نظّم الأوامر الدائمة كبرامج منفصلة ذات حدود واضحة:

```markdown
# Standing Orders

## Program 1: [Domain A] (Weekly)

...

## Program 2: [Domain B] (Monthly + On-Demand)

...

## Program 3: [Domain C] (As-Needed)

...

## Escalation Rules (All Programs)

- [Common escalation criteria]
- [Approval gates that apply across programs]
```

يجب أن يكون لكل برنامج:

- **وتيرة تشغيل** خاصة به (أسبوعية، شهرية، مدفوعة بالأحداث، مستمرة)
- **بوابات موافقة** خاصة به (بعض البرامج تحتاج إلى إشراف أكبر من غيرها)
- **حدود** واضحة (يجب أن يعرف الوكيل أين ينتهي برنامج ويبدأ آخر)

## أفضل الممارسات

### افعل

- ابدأ بسلطة محدودة ووسّعها مع بناء الثقة
- حدّد بوابات موافقة صريحة للإجراءات عالية المخاطر
- أدرج أقسام "ما الذي لا يجب فعله" — فالحدود مهمة بقدر أهمية الأذونات
- ادمجها مع مهام cron لتنفيذ موثوق معتمد على الوقت
- راجع سجلات الوكيل أسبوعيًا للتحقق من الالتزام بالأوامر الدائمة
- حدّث الأوامر الدائمة مع تطور احتياجاتك — فهي وثائق حية

### تجنب

- منح سلطة واسعة من اليوم الأول ("افعل ما تراه أفضل")
- تجاوز قواعد التصعيد — كل برنامج يحتاج إلى بند "متى تتوقف وتسأل"
- افتراض أن الوكيل سيتذكر التعليمات الشفهية — ضع كل شيء في الملف
- خلط المجالات في برنامج واحد — برامج منفصلة لمجالات منفصلة
- نسيان فرض التنفيذ باستخدام مهام cron — فالأوامر الدائمة من دون مشغلات تصبح مجرد اقتراحات

## ذو صلة

- [الأتمتة والمهام](/automation) — نظرة شاملة على جميع آليات الأتمتة
- [مهام Cron](/automation/cron-jobs) — فرض الجدولة للأوامر الدائمة
- [Hooks](/automation/hooks) — نصوص برمجية مدفوعة بالأحداث لأحداث دورة حياة الوكيل
- [Webhooks](/automation/cron-jobs#webhooks) — مشغلات أحداث HTTP واردة
- [مساحة عمل الوكيل](/concepts/agent-workspace) — المكان الذي تعيش فيه الأوامر الدائمة، بما في ذلك القائمة الكاملة لملفات التمهيد المحقونة تلقائيًا (`AGENTS.md` و`SOUL.md` وغيرها)
