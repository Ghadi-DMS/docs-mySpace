---
read_when:
    - تحتاج إلى تعديلات منظّمة على الملفات عبر عدة ملفات
    - تريد توثيق أو تصحيح التعديلات المعتمدة على التصحيحات
summary: تطبيق تصحيحات متعددة الملفات باستخدام أداة apply_patch
title: أداة apply_patch
x-i18n:
    generated_at: "2026-04-05T12:57:07Z"
    model: gpt-5.4
    provider: openai
    source_hash: acca6e702e7ccdf132c71dc6d973f1d435ad6d772e1b620512c8969420cb8f7a
    source_path: tools/apply-patch.md
    workflow: 15
---

# أداة apply_patch

طبّق تغييرات الملفات باستخدام تنسيق تصحيح منظّم. يُعد هذا مثاليًا للتعديلات
متعددة الملفات أو متعددة المقاطع حيث تكون عملية `edit` واحدة هشّة.

تقبل الأداة سلسلة `input` واحدة تغلف عملية ملف واحدة أو أكثر:

```
*** Begin Patch
*** Add File: path/to/file.txt
+line 1
+line 2
*** Update File: src/app.ts
@@
-old line
+new line
*** Delete File: obsolete.txt
*** End Patch
```

## المعلمات

- `input` (مطلوب): محتويات التصحيح الكاملة بما في ذلك `*** Begin Patch` و `*** End Patch`.

## ملاحظات

- تدعم مسارات التصحيح المسارات النسبية (انطلاقًا من دليل مساحة العمل) والمسارات المطلقة.
- تكون القيمة الافتراضية لـ `tools.exec.applyPatch.workspaceOnly` هي `true` (ضمن مساحة العمل فقط). اضبطها على `false` فقط إذا كنت تريد عمدًا أن يقوم `apply_patch` بالكتابة/الحذف خارج دليل مساحة العمل.
- استخدم `*** Move to:` داخل مقطع `*** Update File:` لإعادة تسمية الملفات.
- يحدد `*** End of File` إدراجًا في نهاية الملف فقط عند الحاجة.
- تكون متاحة افتراضيًا لنماذج OpenAI وOpenAI Codex. عيّن
  `tools.exec.applyPatch.enabled: false` لتعطيلها.
- يمكنك اختياريًا تقييدها حسب النموذج عبر
  `tools.exec.applyPatch.allowModels`.
- تكون التهيئة فقط ضمن `tools.exec`.

## مثال

```json
{
  "tool": "apply_patch",
  "input": "*** Begin Patch\n*** Update File: src/index.ts\n@@\n-const foo = 1\n+const foo = 2\n*** End Patch"
}
```
