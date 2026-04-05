---
read_when:
    - العمل على شيفرة تكامل Pi أو اختباراتها
    - تشغيل تدفقات lint وtypecheck والاختبارات المباشرة الخاصة بـ Pi
summary: 'سير عمل المطور لتكامل Pi: البناء والاختبار والتحقق المباشر'
title: سير عمل تطوير Pi
x-i18n:
    generated_at: "2026-04-05T12:49:19Z"
    model: gpt-5.4
    provider: openai
    source_hash: f61ebe29ea38ac953a03fe848fe5ac6b6de4bace5e6955b76ae9a7d093eb0cc5
    source_path: pi-dev.md
    workflow: 15
---

# سير عمل تطوير Pi

يلخص هذا الدليل سير عمل معقول للعمل على تكامل Pi في OpenClaw.

## فحص الأنواع وLinting

- البوابة المحلية الافتراضية: `pnpm check`
- بوابة البناء: `pnpm build` عندما يمكن أن يؤثر التغيير في ناتج البناء أو الحزم أو حدود التحميل الكسول/الوحدات
- بوابة الإنهاء الكاملة للتغييرات الثقيلة الخاصة بـ Pi: ‏`pnpm check && pnpm test`

## تشغيل اختبارات Pi

شغّل مجموعة الاختبارات المركزة على Pi مباشرةً باستخدام Vitest:

```bash
pnpm test \
  "src/agents/pi-*.test.ts" \
  "src/agents/pi-embedded-*.test.ts" \
  "src/agents/pi-tools*.test.ts" \
  "src/agents/pi-settings.test.ts" \
  "src/agents/pi-tool-definition-adapter*.test.ts" \
  "src/agents/pi-hooks/**/*.test.ts"
```

ولتضمين التمرين المباشر للموفّر:

```bash
OPENCLAW_LIVE_TEST=1 pnpm test src/agents/pi-embedded-runner-extraparams.live.test.ts
```

ويغطي هذا مجموعات اختبارات Pi الأساسية:

- `src/agents/pi-*.test.ts`
- `src/agents/pi-embedded-*.test.ts`
- `src/agents/pi-tools*.test.ts`
- `src/agents/pi-settings.test.ts`
- `src/agents/pi-tool-definition-adapter.test.ts`
- `src/agents/pi-hooks/*.test.ts`

## الاختبار اليدوي

التدفق الموصى به:

- شغّل البوابة في وضع التطوير:
  - `pnpm gateway:dev`
- شغّل الوكيل مباشرة:
  - `pnpm openclaw agent --message "Hello" --thinking low`
- استخدم TUI من أجل التصحيح التفاعلي:
  - `pnpm tui`

وبالنسبة إلى سلوك استدعاءات الأدوات، اطلب إجراء `read` أو `exec` حتى تتمكن من رؤية بث الأدوات والتعامل مع الحمولة.

## إعادة تعيين كاملة

تعيش الحالة تحت دليل حالة OpenClaw. والافتراضي هو `~/.openclaw`. وإذا كان `OPENCLAW_STATE_DIR` مضبوطًا، فاستخدم ذلك الدليل بدلًا منه.

لإعادة تعيين كل شيء:

- `openclaw.json` للإعداد
- `agents/<agentId>/agent/auth-profiles.json` لملفات تعريف مصادقة النماذج (مفاتيح API + OAuth)
- `credentials/` لحالة الموفّر/القناة التي لا تزال تعيش خارج مخزن ملفات تعريف المصادقة
- `agents/<agentId>/sessions/` لسجل جلسات الوكيل
- `agents/<agentId>/sessions/sessions.json` لفهرس الجلسات
- `sessions/` إذا كانت توجد مسارات قديمة
- `workspace/` إذا كنت تريد مساحة عمل فارغة

إذا كنت تريد فقط إعادة تعيين الجلسات، فاحذف `agents/<agentId>/sessions/` لذلك الوكيل. وإذا كنت تريد الاحتفاظ بالمصادقة، فاترك `agents/<agentId>/agent/auth-profiles.json` وأي حالة موفّر تحت `credentials/` كما هي.

## مراجع

- [الاختبار](/help/testing)
- [البدء](/ar/start/getting-started)
