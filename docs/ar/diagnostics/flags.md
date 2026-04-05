---
read_when:
    - تحتاج إلى سجلات تصحيح أخطاء مستهدفة من دون رفع مستويات التسجيل العامة
    - تحتاج إلى التقاط سجلات خاصة بنظام فرعي معين للدعم
summary: علامات التشخيص لسجلات تصحيح الأخطاء المستهدفة
title: علامات التشخيص
x-i18n:
    generated_at: "2026-04-05T12:41:53Z"
    model: gpt-5.4
    provider: openai
    source_hash: daf0eca0e6bd1cbc2c400b2e94e1698709a96b9cdba1a8cf00bd580a61829124
    source_path: diagnostics/flags.md
    workflow: 15
---

# علامات التشخيص

تتيح لك علامات التشخيص تمكين سجلات تصحيح أخطاء مستهدفة من دون تشغيل التسجيل المطول في كل مكان. هذه العلامات اختيارية ولا يكون لها أي تأثير ما لم يتحقق منها نظام فرعي.

## كيف يعمل ذلك

- العلامات عبارة عن سلاسل نصية (غير حساسة لحالة الأحرف).
- يمكنك تمكين العلامات في التكوين أو عبر تجاوز بمتغير بيئة.
- البدل مدعوم:
  - `telegram.*` يطابق `telegram.http`
  - `*` يفعّل جميع العلامات

## التمكين عبر التكوين

```json
{
  "diagnostics": {
    "flags": ["telegram.http"]
  }
}
```

علامات متعددة:

```json
{
  "diagnostics": {
    "flags": ["telegram.http", "gateway.*"]
  }
}
```

أعد تشغيل gateway بعد تغيير العلامات.

## تجاوز متغير البيئة (لمرة واحدة)

```bash
OPENCLAW_DIAGNOSTICS=telegram.http,telegram.payload
```

تعطيل جميع العلامات:

```bash
OPENCLAW_DIAGNOSTICS=0
```

## مكان السجلات

تصدر العلامات السجلات إلى ملف سجل التشخيص القياسي. افتراضيًا:

```
/tmp/openclaw/openclaw-YYYY-MM-DD.log
```

إذا قمت بتعيين `logging.file`، فاستخدم ذلك المسار بدلًا من ذلك. تكون السجلات بتنسيق JSONL (كائن JSON واحد في كل سطر). وما زال التنقيح يُطبّق استنادًا إلى `logging.redactSensitive`.

## استخراج السجلات

اختر أحدث ملف سجل:

```bash
ls -t /tmp/openclaw/openclaw-*.log | head -n 1
```

صفِّ سجلات تشخيص Telegram HTTP:

```bash
rg "telegram http error" /tmp/openclaw/openclaw-*.log
```

أو راقبها أثناء إعادة إنتاج المشكلة:

```bash
tail -f /tmp/openclaw/openclaw-$(date +%F).log | rg "telegram http error"
```

وبالنسبة إلى البوابات البعيدة، يمكنك أيضًا استخدام `openclaw logs --follow` ‏(راجع [/cli/logs](/cli/logs)).

## ملاحظات

- إذا كانت قيمة `logging.level` أعلى من `warn`، فقد يتم كتم هذه السجلات. والقيمة الافتراضية `info` مناسبة.
- من الآمن إبقاء العلامات مفعلة؛ فهي تؤثر فقط في حجم السجل للنظام الفرعي المحدد.
- استخدم [/logging](/logging) لتغيير وجهات السجل والمستويات والتنقيح.
