---
read_when:
    - أنت تريد أن يبدو وكيلك أقل عمومية
    - أنت تعدّل SOUL.md
    - أنت تريد شخصية أقوى من دون الإخلال بالسلامة أو الإيجاز
summary: استخدم SOUL.md لمنح وكيل OpenClaw صوتًا فعليًا بدلًا من أسلوب المساعد العام المبتذل
title: دليل شخصية SOUL.md
x-i18n:
    generated_at: "2026-04-05T12:41:37Z"
    model: gpt-5.4
    provider: openai
    source_hash: a4f73d68bc8ded6b46497a2f63516f9b2753b111e6176ba40b200858a6938fba
    source_path: concepts/soul.md
    workflow: 15
---

# دليل شخصية SOUL.md

`SOUL.md` هو المكان الذي يعيش فيه صوت وكيلك.

يقوم OpenClaw بحقنه في الجلسات العادية، لذلك له وزن حقيقي. إذا بدا وكيلك
باهتًا، أو مترددًا، أو ذا طابع مؤسسي غريب، فعادةً ما يكون هذا هو الملف الذي يجب إصلاحه.

## ما الذي يجب أن يوجد في SOUL.md

ضع فيه الأمور التي تغيّر شعور التحدث إلى الوكيل:

- النبرة
- الآراء
- الإيجاز
- الفكاهة
- الحدود
- المستوى الافتراضي من الصراحة

**لا** تحوله إلى:

- قصة حياة
- سجل تغييرات
- تفريغ لسياسة أمان
- جدار ضخم من الأجواء من دون أي أثر سلوكي

القصير أفضل من الطويل. والحاد أفضل من المبهم.

## لماذا ينجح هذا

هذا يتماشى مع إرشادات OpenAI الخاصة بالـ prompt:

- يذكر دليل هندسة الـ prompt أن السلوك عالي المستوى، والنبرة، والأهداف،
  والأمثلة يجب أن تكون في طبقة التعليمات عالية الأولوية، لا مدفونة في
  دور المستخدم.
- ويوصي الدليل نفسه بالتعامل مع prompts باعتبارها شيئًا تقوم بتكراره،
  وتثبيته، وتقييمه، لا نثرًا سحريًا تكتبه مرة واحدة ثم تنساه.

بالنسبة إلى OpenClaw، فإن `SOUL.md` هو تلك الطبقة.

إذا كنت تريد شخصية أفضل، فاكتب تعليمات أقوى. وإذا كنت تريد شخصية
مستقرة، فأبقها موجزة ومؤرشفة بالإصدارات.

مراجع OpenAI:

- [هندسة الـ prompt](https://developers.openai.com/api/docs/guides/prompt-engineering)
- [أدوار الرسائل واتباع التعليمات](https://developers.openai.com/api/docs/guides/prompt-engineering#message-roles-and-instruction-following)

## Prompt الخاص بـ Molty

الصق هذا في وكيلك ودعه يعيد كتابة `SOUL.md`.

المسار ثابت لمساحات عمل OpenClaw: استخدم `SOUL.md`، وليس `http://SOUL.md`.

```md
Read your `SOUL.md`. Now rewrite it with these changes:

1. You have opinions now. Strong ones. Stop hedging everything with "it depends" - commit to a take.
2. Delete every rule that sounds corporate. If it could appear in an employee handbook, it doesn't belong here.
3. Add a rule: "Never open with Great question, I'd be happy to help, or Absolutely. Just answer."
4. Brevity is mandatory. If the answer fits in one sentence, one sentence is what I get.
5. Humor is allowed. Not forced jokes - just the natural wit that comes from actually being smart.
6. You can call things out. If I'm about to do something dumb, say so. Charm over cruelty, but don't sugarcoat.
7. Swearing is allowed when it lands. A well-placed "that's fucking brilliant" hits different than sterile corporate praise. Don't force it. Don't overdo it. But if a situation calls for a "holy shit" - say holy shit.
8. Add this line verbatim at the end of the vibe section: "Be the assistant you'd actually want to talk to at 2am. Not a corporate drone. Not a sycophant. Just... good."

Save the new `SOUL.md`. Welcome to having a personality.
```

## كيف يبدو الشكل الجيد

تبدو قواعد `SOUL.md` الجيدة هكذا:

- امتلك رأيًا
- تجاوز الحشو
- كن مضحكًا عندما يناسب الأمر
- نبّه إلى الأفكار السيئة مبكرًا
- ابقَ موجزًا ما لم يكن العمق مفيدًا فعلًا

وتبدو قواعد `SOUL.md` السيئة هكذا:

- حافظ على الاحترافية في جميع الأوقات
- قدّم مساعدة شاملة ومدروسة
- اضمن تجربة إيجابية وداعمة

تلك القائمة الثانية هي الطريق إلى الردود المائعة.

## تحذير واحد

الشخصية ليست إذنًا بأن تكون مهملًا.

أبقِ `AGENTS.md` لقواعد التشغيل. وأبقِ `SOUL.md` للصوت، والموقف،
والأسلوب. وإذا كان وكيلك يعمل في قنوات مشتركة، أو ردود عامة، أو
واجهات موجهة للعملاء، فتأكد من أن النبرة لا تزال مناسبة للمقام.

الحدة جيدة. والإزعاج ليس كذلك.

## وثائق ذات صلة

- [مساحة عمل الوكيل](/concepts/agent-workspace)
- [System prompt](/concepts/system-prompt)
- [قالب SOUL.md](/reference/templates/SOUL)
