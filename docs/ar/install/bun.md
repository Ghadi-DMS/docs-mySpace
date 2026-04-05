---
read_when:
    - تريد أسرع حلقة تطوير محلية (bun + watch)
    - واجهت مشكلات في تثبيت Bun / patch / lifecycle script
summary: 'سير عمل Bun ‏(تجريبي): التثبيتات والمشكلات مقارنةً بـ pnpm'
title: Bun (تجريبي)
x-i18n:
    generated_at: "2026-04-05T12:45:39Z"
    model: gpt-5.4
    provider: openai
    source_hash: b0845567834124bb9206db64df013dc29f3b61a04da4f7e7f0c2823a9ecd67a6
    source_path: install/bun.md
    workflow: 15
---

# Bun (تجريبي)

<Warning>
لا يُنصح باستخدام Bun **لوقت تشغيل gateway** (توجد مشكلات معروفة مع WhatsApp وTelegram). استخدم Node في الإنتاج.
</Warning>

Bun هو وقت تشغيل محلي اختياري لتشغيل TypeScript مباشرةً (`bun run ...`, `bun --watch ...`). يظل مدير الحزم الافتراضي هو `pnpm`، وهو مدعوم بالكامل ويُستخدم بواسطة أدوات الوثائق. لا يمكن لـ Bun استخدام `pnpm-lock.yaml` وسيتجاهله.

## التثبيت

<Steps>
  <Step title="تثبيت التبعيات">
    ```sh
    bun install
    ```

    يتم تجاهل `bun.lock` / `bun.lockb` في git، لذلك لا يحدث أي تشويش في المستودع. ولتخطي كتابة lockfile بالكامل:

    ```sh
    bun install --no-save
    ```

  </Step>
  <Step title="البناء والاختبار">
    ```sh
    bun run build
    bun run vitest run
    ```
  </Step>
</Steps>

## Lifecycle Scripts

يمنع Bun lifecycle scripts الخاصة بالتبعيات ما لم تُوثَّق صراحةً. بالنسبة إلى هذا المستودع، لا تكون scripts المحجوبة الشائعة مطلوبة:

- ‏`@whiskeysockets/baileys` ‏`preinstall` -- تتحقق من أن إصدار Node الرئيسي >= 20 (يستخدم OpenClaw افتراضيًا Node 24 وما زال يدعم Node 22 LTS، حاليًا `22.14+`)
- ‏`protobufjs` ‏`postinstall` -- تُصدر تحذيرات حول مخططات الإصدارات غير المتوافقة (من دون مخرجات build)

إذا واجهت مشكلة وقت تشغيل تتطلب هذه scripts، فوثّقها صراحةً:

```sh
bun pm trust @whiskeysockets/baileys protobufjs
```

## محاذير

لا تزال بعض scripts تُثبّت `pnpm` صراحةً (مثل `docs:build`, `ui:*`, `protocol:check`). شغّلها عبر pnpm في الوقت الحالي.
