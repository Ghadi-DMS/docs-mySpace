---
read_when:
    - ضبط سلوك Voice Overlay
summary: دورة حياة Voice Overlay عند تداخل wake-word وpush-to-talk
title: Voice Overlay
x-i18n:
    generated_at: "2026-04-05T12:50:24Z"
    model: gpt-5.4
    provider: openai
    source_hash: 1efcc26ec05d2f421cb2cf462077d002381995b338d00db77d5fdba9b8d938b6
    source_path: platforms/mac/voice-overlay.md
    workflow: 15
---

# دورة حياة Voice Overlay ‏(macOS)

الجمهور المستهدف: المساهمون في تطبيق macOS. الهدف: إبقاء Voice Overlay متوقعة السلوك عندما يتداخل wake-word وpush-to-talk.

## الهدف الحالي

- إذا كانت الـ overlay ظاهرة بالفعل نتيجة wake-word وضغط المستخدم مفتاح الاختصار، فإن جلسة مفتاح الاختصار _تتبنى_ النص الموجود بدلًا من إعادة تعيينه. وتبقى الـ overlay ظاهرة ما دام مفتاح الاختصار مضغوطًا. وعندما يحرر المستخدم المفتاح: يتم الإرسال إذا كان هناك نص بعد التشذيب، وإلا يتم الإخفاء.
- ما تزال wake-word وحدها ترسل تلقائيًا عند الصمت؛ بينما تقوم push-to-talk بالإرسال فورًا عند التحرير.

## ما تم تنفيذه (9 ديسمبر 2025)

- أصبحت جلسات الـ overlay الآن تحمل token لكل عملية التقاط (wake-word أو push-to-talk). ويتم إسقاط تحديثات partial/final/send/dismiss/level عندما لا يتطابق token، مما يمنع callbacks القديمة.
- تتبنى push-to-talk أي نص ظاهر في الـ overlay كبادئة (لذلك فإن الضغط على مفتاح الاختصار بينما تكون wake overlay ظاهرة يبقي النص ويُلحق الكلام الجديد). وهي تنتظر حتى 1.5 ثانية للحصول على نص نهائي قبل العودة إلى النص الحالي.
- يتم إصدار سجلات chime/overlay عند المستوى `info` ضمن الفئات `voicewake.overlay` و`voicewake.ptt` و`voicewake.chime` (بدء الجلسة، والنص الجزئي، والنص النهائي، والإرسال، والإخفاء، وسبب chime).

## الخطوات التالية

1. **VoiceSessionCoordinator ‏(actor)**
   - يملك `VoiceSession` واحدة فقط في كل مرة.
   - API ‏(قائم على token): ‏`beginWakeCapture` و`beginPushToTalk` و`updatePartial` و`endCapture` و`cancel` و`applyCooldown`.
   - يسقط callbacks التي تحمل tokens قديمة (لمنع أدوات التعرف القديمة من إعادة فتح الـ overlay).
2. **VoiceSession ‏(model)**
   - الحقول: `token` و`source` ‏(`wakeWord|pushToTalk`) والنص الملتزم/المتغير، وأعلام chime، والمؤقتات (الإرسال التلقائي، والخمول)، و`overlayMode` ‏(`display|editing|sending`)، وموعد نهائي للـ cooldown.
3. **ربط الـ overlay**
   - تقوم `VoiceSessionPublisher` ‏(`ObservableObject`) بعكس الجلسة النشطة إلى SwiftUI.
   - يقوم `VoiceWakeOverlayView` بالعرض فقط عبر الناشر؛ ولا يغيّر singletons العامة مباشرة أبدًا.
   - تستدعي إجراءات المستخدم في الـ overlay ‏(`sendNow` و`dismiss` و`edit`) الـ coordinator مرة أخرى باستخدام token الخاص بالجلسة.
4. **مسار إرسال موحّد**
   - عند `endCapture`: إذا كان النص المشذب فارغًا ← إخفاء؛ وإلا `performSend(session:)` ‏(يشغّل chime الإرسال مرة واحدة، ثم يمرر، ثم يُخفي).
   - ‏Push-to-talk: من دون تأخير؛ ‏wake-word: تأخير اختياري للإرسال التلقائي.
   - طبّق cooldown قصيرة على وقت تشغيل wake بعد انتهاء push-to-talk حتى لا تُعيد wake-word التشغيل فورًا.
5. **التسجيل**
   - يصدر الـ coordinator سجلات `.info` في subsystem ‏`ai.openclaw`، والفئتين `voicewake.overlay` و`voicewake.chime`.
   - الأحداث الأساسية: `session_started` و`adopted_by_push_to_talk` و`partial` و`finalized` و`send` و`dismiss` و`cancel` و`cooldown`.

## قائمة التحقق من تصحيح الأخطاء

- قم ببث السجلات أثناء إعادة إنتاج overlay العالقة:

  ```bash
  sudo log stream --predicate 'subsystem == "ai.openclaw" AND category CONTAINS "voicewake"' --level info --style compact
  ```

- تحقق من وجود token واحدة فقط لجلسة نشطة؛ يجب أن يُسقط الـ coordinator callbacks القديمة.
- تأكد من أن تحرير push-to-talk يستدعي دائمًا `endCapture` باستخدام token النشطة؛ وإذا كان النص فارغًا، فتوقع `dismiss` من دون chime أو إرسال.

## خطوات الترحيل (مقترحة)

1. أضف `VoiceSessionCoordinator` و`VoiceSession` و`VoiceSessionPublisher`.
2. أعد هيكلة `VoiceWakeRuntime` بحيث ينشئ/يحدّث/ينهي الجلسات بدلًا من لمس `VoiceWakeOverlayController` مباشرة.
3. أعد هيكلة `VoicePushToTalk` بحيث تتبنى الجلسات الحالية وتستدعي `endCapture` عند التحرير؛ ثم طبّق cooldown لوقت التشغيل.
4. اربط `VoiceWakeOverlayController` بالناشر؛ وأزل الاستدعاءات المباشرة من runtime/PTT.
5. أضف اختبارات تكامل لتبني الجلسات، وcooldown، وإخفاء النص الفارغ.
