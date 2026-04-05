---
read_when:
    - بناء أو توقيع بنيات تصحيح mac
summary: خطوات التوقيع لبنيات تصحيح macOS التي تُنشئها نصوص التغليف
title: توقيع macOS
x-i18n:
    generated_at: "2026-04-05T12:50:24Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7b16d726549cf6dc34dc9c60e14d8041426ebc0699ab59628aca1d094380334a
    source_path: platforms/mac/signing.md
    workflow: 15
---

# توقيع mac ‏(بنيات التصحيح)

يُبنى هذا التطبيق عادةً من [`scripts/package-mac-app.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/package-mac-app.sh)، والذي يقوم الآن بما يلي:

- يضبط معرّف حزمة تصحيح ثابتًا: `ai.openclaw.mac.debug`
- يكتب `Info.plist` باستخدام معرّف الحزمة هذا (يمكن تجاوزه عبر `BUNDLE_ID=...`)
- يستدعي [`scripts/codesign-mac-app.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/codesign-mac-app.sh) لتوقيع الملف التنفيذي الرئيسي وحزمة التطبيق بحيث يتعامل macOS مع كل إعادة بناء على أنها الحزمة الموقعة نفسها ويحافظ على أذونات TCC ‏(الإشعارات، وإمكانية الوصول، وتسجيل الشاشة، والميكروفون، والكلام). وللحصول على أذونات مستقرة، استخدم هوية توقيع حقيقية؛ فالتوقيع ad-hoc يتطلب تفعيلًا صريحًا وهو هش (راجع [أذونات macOS](/platforms/mac/permissions)).
- يستخدم `CODESIGN_TIMESTAMP=auto` افتراضيًا؛ إذ يفعّل الطوابع الزمنية الموثوقة لتواقيع Developer ID. اضبط `CODESIGN_TIMESTAMP=off` لتخطي وضع الطابع الزمني (لبنيات التصحيح غير المتصلة).
- يحقن بيانات تعريف البناء داخل `Info.plist`: ‏`OpenClawBuildTimestamp` ‏(UTC) و`OpenClawGitCommit` ‏(hash قصير) حتى تتمكن نافذة About من عرض البناء وgit وقناة التصحيح/الإصدار.
- **تستخدم عملية التغليف Node 24 افتراضيًا**: يشغّل النص بنيات TS وبناء Control UI. وما زالت Node 22 LTS، حاليًا `22.14+`، مدعومة من أجل التوافق.
- يقرأ `SIGN_IDENTITY` من البيئة. أضف `export SIGN_IDENTITY="Apple Development: Your Name (TEAMID)"` ‏(أو شهادة Developer ID Application الخاصة بك) إلى shell rc لديك ليتم التوقيع دائمًا بشهادتك. ويتطلب التوقيع ad-hoc تفعيلًا صريحًا عبر `ALLOW_ADHOC_SIGNING=1` أو `SIGN_IDENTITY="-"` ‏(غير موصى به لاختبار الأذونات).
- يشغّل تدقيق Team ID بعد التوقيع ويفشل إذا كان أي ملف Mach-O داخل حزمة التطبيق موقّعًا بواسطة Team ID مختلف. اضبط `SKIP_TEAM_ID_CHECK=1` لتجاوز ذلك.

## الاستخدام

```bash
# من جذر المستودع
scripts/package-mac-app.sh               # يختار الهوية تلقائيًا؛ ويعطي خطأ إذا لم يعثر على أي هوية
SIGN_IDENTITY="Developer ID Application: Your Name" scripts/package-mac-app.sh   # شهادة حقيقية
ALLOW_ADHOC_SIGNING=1 scripts/package-mac-app.sh    # ad-hoc (لن تستمر الأذونات)
SIGN_IDENTITY="-" scripts/package-mac-app.sh        # ad-hoc صريح (مع التحذير نفسه)
DISABLE_LIBRARY_VALIDATION=1 scripts/package-mac-app.sh   # حل بديل خاص بالتطوير فقط لعدم تطابق Sparkle Team ID
```

### ملاحظة حول التوقيع ad-hoc

عند التوقيع باستخدام `SIGN_IDENTITY="-"` ‏(ad-hoc)، يعطل النص تلقائيًا **Hardened Runtime** ‏(`--options runtime`). وهذا ضروري لمنع الأعطال عندما يحاول التطبيق تحميل الأطر المضمنة (مثل Sparkle) التي لا تشترك في Team ID نفسه. كما أن التواقيع ad-hoc تكسر استمرارية أذونات TCC؛ راجع [أذونات macOS](/platforms/mac/permissions) للاطلاع على خطوات الاستعادة.

## بيانات تعريف البناء الخاصة بـ About

يضع `package-mac-app.sh` الختم التالي على الحزمة:

- `OpenClawBuildTimestamp`: قيمة ISO8601 UTC وقت التغليف
- `OpenClawGitCommit`: ‏hash قصير لـ git ‏(أو `unknown` إذا لم يكن متاحًا)

يقرأ تبويب About هذه المفاتيح لعرض الإصدار، وتاريخ البناء، وgit commit، وما إذا كانت البنية بنية تصحيح (عبر `#if DEBUG`). شغّل أداة التغليف لتحديث هذه القيم بعد تغييرات الشيفرة.

## لماذا

ترتبط أذونات TCC بمعرّف الحزمة _وبتوقيع الشيفرة_ معًا. وكانت بنيات التصحيح غير الموقعة ذات UUIDs المتغيرة تتسبب في نسيان macOS للأذونات بعد كل إعادة بناء. إن توقيع الملفات التنفيذية (ad-hoc افتراضيًا) مع الحفاظ على معرّف/مسار حزمة ثابت (`dist/OpenClaw.app`) يحافظ على الأذونات بين البنيات، بما يتوافق مع نهج VibeTunnel.
