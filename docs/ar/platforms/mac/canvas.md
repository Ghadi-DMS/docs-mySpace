---
read_when:
    - تنفيذ لوحة Canvas في تطبيق macOS
    - إضافة عناصر تحكم الوكيل لمساحة العمل المرئية
    - تصحيح عمليات تحميل canvas في WKWebView
summary: لوحة Canvas مضمّنة يتحكم بها الوكيل عبر WKWebView + مخطط URL مخصص
title: Canvas
x-i18n:
    generated_at: "2026-04-05T12:49:58Z"
    model: gpt-5.4
    provider: openai
    source_hash: b6c71763d693264d943e570a852208cce69fc469976b2a1cdd9e39e2550534c1
    source_path: platforms/mac/canvas.md
    workflow: 15
---

# Canvas (تطبيق macOS)

يضمّن تطبيق macOS **لوحة Canvas** يتحكم بها الوكيل باستخدام `WKWebView`. وهي
مساحة عمل مرئية خفيفة لـ HTML/CSS/JS وA2UI والواجهات
التفاعلية الصغيرة.

## مكان وجود Canvas

تُخزَّن حالة Canvas ضمن Application Support:

- `~/Library/Application Support/OpenClaw/canvas/<session>/...`

وتعرض لوحة Canvas هذه الملفات عبر **مخطط URL مخصص**:

- `openclaw-canvas://<session>/<path>`

أمثلة:

- `openclaw-canvas://main/` ← `<canvasRoot>/main/index.html`
- `openclaw-canvas://main/assets/app.css` ← `<canvasRoot>/main/assets/app.css`
- `openclaw-canvas://main/widgets/todo/` ← `<canvasRoot>/main/widgets/todo/index.html`

إذا لم يوجد `index.html` في الجذر، يعرض التطبيق **صفحة scaffold مدمجة**.

## سلوك اللوحة

- لوحة بلا حدود وقابلة لتغيير الحجم ومثبتة بالقرب من شريط القائمة (أو مؤشر الفأرة).
- تتذكر الحجم/الموضع لكل جلسة.
- تعيد التحميل تلقائيًا عند تغير ملفات canvas المحلية.
- تكون لوحة Canvas واحدة فقط مرئية في الوقت نفسه (ويتم تبديل الجلسة عند الحاجة).

يمكن تعطيل Canvas من Settings ← **Allow Canvas**. وعند تعطيلها، تعيد أوامر عقدة
canvas القيمة `CANVAS_DISABLED`.

## سطح API الخاص بالوكيل

تُعرَض Canvas عبر **Gateway WebSocket**، بحيث يمكن للوكيل:

- إظهار اللوحة/إخفاؤها
- الانتقال إلى مسار أو URL
- تنفيذ JavaScript
- التقاط صورة snapshot

أمثلة CLI:

```bash
openclaw nodes canvas present --node <id>
openclaw nodes canvas navigate --node <id> --url "/"
openclaw nodes canvas eval --node <id> --js "document.title"
openclaw nodes canvas snapshot --node <id>
```

ملاحظات:

- يقبل `canvas.navigate` **مسارات canvas المحلية**، وعناوين URL من نوع `http(s)`، وعناوين URL من نوع `file://`.
- إذا مررت `"/"`، فستعرض Canvas الـ scaffold المحلي أو `index.html`.

## A2UI داخل Canvas

تُستضاف A2UI بواسطة Gateway canvas host وتُعرَض داخل لوحة Canvas.
وعندما يعلن Gateway عن Canvas host، ينتقل تطبيق macOS تلقائيًا إلى
صفحة مضيف A2UI عند أول فتح.

عنوان URL الافتراضي لمضيف A2UI:

```
http://<gateway-host>:18789/__openclaw__/a2ui/
```

### أوامر A2UI ‏(v0.8)

تقبل Canvas حاليًا رسائل **A2UI v0.8** من الخادم إلى العميل:

- `beginRendering`
- `surfaceUpdate`
- `dataModelUpdate`
- `deleteSurface`

أما `createSurface` ‏(v0.9) فهو غير مدعوم.

مثال CLI:

```bash
cat > /tmp/a2ui-v0.8.jsonl <<'EOFA2'
{"surfaceUpdate":{"surfaceId":"main","components":[{"id":"root","component":{"Column":{"children":{"explicitList":["title","content"]}}}},{"id":"title","component":{"Text":{"text":{"literalString":"Canvas (A2UI v0.8)"},"usageHint":"h1"}}},{"id":"content","component":{"Text":{"text":{"literalString":"If you can read this, A2UI push works."},"usageHint":"body"}}}]}}
{"beginRendering":{"surfaceId":"main","root":"root"}}
EOFA2

openclaw nodes canvas a2ui push --jsonl /tmp/a2ui-v0.8.jsonl --node <id>
```

اختبار سريع:

```bash
openclaw nodes canvas a2ui push --node <id> --text "Hello from A2UI"
```

## تشغيل عمليات وكيل من Canvas

يمكن لـ Canvas تشغيل عمليات وكيل جديدة عبر الروابط العميقة:

- `openclaw://agent?...`

مثال (في JavaScript):

```js
window.location.href = "openclaw://agent?message=Review%20this%20design";
```

يعرض التطبيق مطالبة تأكيد ما لم يتم تقديم مفتاح صالح.

## ملاحظات الأمان

- يمنع مخطط Canvas اجتياز الأدلة؛ ويجب أن توجد الملفات ضمن جذر الجلسة.
- يستخدم محتوى Canvas المحلي مخططًا مخصصًا (ولا يتطلب خادم loopback).
- لا يُسمح بعناوين URL الخارجية من نوع `http(s)` إلا عند الانتقال إليها صراحة.
