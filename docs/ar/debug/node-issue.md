---
read_when:
    - تصحيح أعطال نصوص التطوير أو وضع المراقبة التي تحدث مع Node فقط
    - التحقيق في أعطال محمل tsx/esbuild في OpenClaw
summary: ملاحظات وحلول بديلة لتعطل Node + tsx برسالة "__name is not a function"
title: تعطل Node + tsx
x-i18n:
    generated_at: "2026-04-05T12:41:55Z"
    model: gpt-5.4
    provider: openai
    source_hash: f5beab7cdfe7679680f65176234a617293ce495886cfffb151518adfa61dc8dc
    source_path: debug/node-issue.md
    workflow: 15
---

# تعطل Node + tsx "\_\_name is not a function"

## الملخص

يفشل تشغيل OpenClaw عبر Node باستخدام `tsx` عند بدء التشغيل مع الخطأ التالي:

```
[openclaw] Failed to start CLI: TypeError: __name is not a function
    at createSubsystemLogger (.../src/logging/subsystem.ts:203:25)
    at .../src/agents/auth-profiles/constants.ts:25:20
```

بدأ هذا بعد تحويل نصوص التطوير من Bun إلى `tsx` (الالتزام `2871657e`، بتاريخ 2026-01-06). وكان مسار وقت التشغيل نفسه يعمل مع Bun.

## البيئة

- Node: ‏v25.x (لوحظ على v25.3.0)
- tsx: ‏4.21.0
- نظام التشغيل: macOS (ومن المرجح أيضًا إعادة إنتاج المشكلة على منصات أخرى تشغّل Node 25)

## إعادة الإنتاج (Node فقط)

```bash
# in repo root
node --version
pnpm install
node --import tsx src/entry.ts status
```

## إعادة إنتاج مصغرة في المستودع

```bash
node --import tsx scripts/repro/tsx-name-repro.ts
```

## التحقق من إصدار Node

- Node 25.3.0: يفشل
- Node 22.22.0 ‏(Homebrew `node@22`): يفشل
- Node 24: غير مثبت هنا بعد؛ يحتاج إلى تحقق

## ملاحظات / فرضية

- يستخدم `tsx` أداة esbuild لتحويل TS/ESM. ويُخرج `keepNames` في esbuild مساعد `__name` ويلف تعريفات الدوال باستخدام `__name(...)`.
- يشير التعطل إلى أن `__name` موجود لكنه ليس دالة وقت التشغيل، ما يعني أن المساعد مفقود أو تمت الكتابة فوقه لهذه الوحدة في مسار محمل Node 25.
- تم الإبلاغ عن مشكلات مشابهة لمساعد `__name` لدى مستهلكين آخرين لـ esbuild عندما يكون المساعد مفقودًا أو أُعيدت كتابته.

## سجل الانحدار

- `2871657e` ‏(2026-01-06): تغيرت النصوص من Bun إلى tsx لجعل Bun اختياريًا.
- قبل ذلك (مسار Bun)، كان `openclaw status` و`gateway:watch` يعملان.

## الحلول البديلة

- استخدم Bun لنصوص التطوير (الرجوع المؤقت الحالي).
- استخدم Node + مراقبة tsc، ثم شغّل الخرج المترجم:

  ```bash
  pnpm exec tsc --watch --preserveWatchOutput
  node --watch openclaw.mjs status
  ```

- تم التأكد محليًا: يعمل `pnpm exec tsc -p tsconfig.json` + `node openclaw.mjs status` على Node 25.
- عطّل esbuild keepNames في محمل TS إن أمكن (فهذا يمنع إدراج المساعد `__name`)؛ ولا يوفّر tsx هذا حاليًا.
- اختبر Node LTS ‏(22/24) مع `tsx` لمعرفة ما إذا كانت المشكلة خاصة بـ Node 25.

## المراجع

- [https://opennext.js.org/cloudflare/howtos/keep_names](https://opennext.js.org/cloudflare/howtos/keep_names)
- [https://esbuild.github.io/api/#keep-names](https://esbuild.github.io/api/#keep-names)
- [https://github.com/evanw/esbuild/issues/1031](https://github.com/evanw/esbuild/issues/1031)

## الخطوات التالية

- أعد الإنتاج على Node 22/24 لتأكيد ما إذا كان هذا انحدارًا خاصًا بـ Node 25.
- اختبر `tsx` الليلي أو ثبّته على إصدار أقدم إذا وُجد انحدار معروف.
- إذا أمكن إعادة الإنتاج على Node LTS، فافتح بلاغًا مصغرًا عند المنبع مع تتبع المكدس الخاص بـ `__name`.
