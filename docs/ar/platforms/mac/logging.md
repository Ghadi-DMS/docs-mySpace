---
read_when:
    - التقاط سجلات macOS أو التحقيق في تسجيل البيانات الخاصة
    - تصحيح مشكلات تنبيه الصوت/دورة حياة الجلسة
summary: 'تسجيل OpenClaw: ملف سجل تشخيصي متجدد + أعلام خصوصية السجل الموحد'
title: التسجيل في macOS
x-i18n:
    generated_at: "2026-04-05T12:50:09Z"
    model: gpt-5.4
    provider: openai
    source_hash: c08d6bc012f8e8bb53353fe654713dede676b4e6127e49fd76e00c2510b9ab0b
    source_path: platforms/mac/logging.md
    workflow: 15
---

# التسجيل (macOS)

## ملف سجل التشخيصات المتجدد (لوحة Debug)

يقوم OpenClaw بتوجيه سجلات تطبيق macOS عبر swift-log ‏(التسجيل الموحد افتراضيًا)، ويمكنه كتابة سجل ملفات محلي ومتجدد على القرص عندما تحتاج إلى التقاط دائم.

- مستوى التفاصيل: **لوحة Debug → Logs → App logging → Verbosity**
- التمكين: **لوحة Debug → Logs → App logging → “Write rolling diagnostics log (JSONL)”**
- الموقع: `~/Library/Logs/OpenClaw/diagnostics.jsonl` ‏(يتم تدويره تلقائيًا؛ وتُلحق الملفات القديمة باللاحقات `.1` و`.2` و…)
- المسح: **لوحة Debug → Logs → App logging → “Clear”**

ملاحظات:

- يكون هذا **معطلًا افتراضيًا**. فعّله فقط أثناء التصحيح النشط.
- تعامل مع الملف على أنه حساس؛ ولا تشاركه من دون مراجعة.

## بيانات السجل الموحد الخاصة في macOS

يقوم التسجيل الموحد بتنقيح معظم الحمولات ما لم يشترك نظام فرعي في `privacy -off`. ووفقًا لشرح Peter حول [حيل خصوصية التسجيل](https://steipete.me/posts/2025/logging-privacy-shenanigans) في macOS ‏(2025)، يتم التحكم في ذلك بواسطة ملف plist في `/Library/Preferences/Logging/Subsystems/` بمفتاح يحمل اسم النظام الفرعي. ولا تلتقط العلم سوى إدخالات السجل الجديدة، لذا فعّله قبل إعادة إنتاج المشكلة.

## التمكين لـ OpenClaw ‏(`ai.openclaw`)

- اكتب ملف plist إلى ملف مؤقت أولًا، ثم ثبّته بشكل ذري بصلاحيات الجذر:

```bash
cat <<'EOF' >/tmp/ai.openclaw.plist
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>DEFAULT-OPTIONS</key>
    <dict>
        <key>Enable-Private-Data</key>
        <true/>
    </dict>
</dict>
</plist>
EOF
sudo install -m 644 -o root -g wheel /tmp/ai.openclaw.plist /Library/Preferences/Logging/Subsystems/ai.openclaw.plist
```

- لا يلزم إعادة التشغيل؛ يلاحظ `logd` الملف بسرعة، لكن أسطر السجل الجديدة فقط ستتضمن الحمولات الخاصة.
- اعرض المخرجات الأكثر غنى باستخدام المساعد الموجود، مثل `./scripts/clawlog.sh --category WebChat --last 5m`.

## التعطيل بعد التصحيح

- أزل التجاوز: `sudo rm /Library/Preferences/Logging/Subsystems/ai.openclaw.plist`.
- ويمكنك اختياريًا تشغيل `sudo log config --reload` لإجبار `logd` على إسقاط التجاوز فورًا.
- تذكّر أن هذا السطح قد يتضمن أرقام الهواتف ونصوص الرسائل؛ لذا أبقِ ملف plist في مكانه فقط أثناء حاجتك الفعلية إلى هذا القدر الإضافي من التفاصيل.
