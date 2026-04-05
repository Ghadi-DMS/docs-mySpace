---
read_when:
    - تحتاج إلى فهم سبب تشغيل وظيفة CI أو عدم تشغيلها
    - أنت تصحح أخطاء فحوصات GitHub Actions الفاشلة
summary: رسم وظائف CI البياني، وبوابات النطاق، والمكافئات المحلية للأوامر
title: مسار CI
x-i18n:
    generated_at: "2026-04-05T12:37:27Z"
    model: gpt-5.4
    provider: openai
    source_hash: 5a95b6e584b4309bc249866ea436b4dfe30e0298ab8916eadbc344edae3d1194
    source_path: ci.md
    workflow: 15
---

# مسار CI

يعمل CI عند كل دفع إلى `main` وكل pull request. ويستخدم تحديد نطاق ذكيًا لتخطي الوظائف المكلفة عندما تكون التغييرات مقتصرة على مناطق غير مرتبطة.

## نظرة عامة على الوظائف

| الوظيفة                 | الغرض                                                                                     | وقت التشغيل                         |
| ----------------------- | ----------------------------------------------------------------------------------------- | ----------------------------------- |
| `preflight`             | اكتشاف التغييرات الخاصة بالوثائق فقط، والنطاقات المتغيرة، والإضافات المتغيرة، وبناء CI manifest | دائمًا في عمليات الدفع وطلبات السحب غير المسودة |
| `security-fast`         | اكتشاف المفاتيح الخاصة، وتدقيق workflow عبر `zizmor`، وتدقيق تبعيات الإنتاج               | دائمًا في عمليات الدفع وطلبات السحب غير المسودة |
| `build-artifacts`       | بناء `dist/` وControl UI مرة واحدة، ورفع artifacts قابلة لإعادة الاستخدام للوظائف اللاحقة | التغييرات المرتبطة بـ Node          |
| `checks-fast-core`      | مسارات صحة Linux السريعة مثل فحوصات الحزمة/عقد الإضافات/البروتوكول                      | التغييرات المرتبطة بـ Node          |
| `checks-fast-extensions` | تجميع مسارات تقسيم الإضافات بعد اكتمال `checks-fast-extensions-shard`                    | التغييرات المرتبطة بـ Node          |
| `extension-fast`        | اختبارات مركزة للإضافات المضمّنة التي تغيّرت فقط                                         | عند اكتشاف تغييرات في الإضافات      |
| `check`                 | البوابة المحلية الرئيسية في CI: ‏`pnpm check` بالإضافة إلى `pnpm build:strict-smoke`     | التغييرات المرتبطة بـ Node          |
| `check-additional`      | حمايات البنية والحدود بالإضافة إلى حزمة اختبار تراجع مراقبة gateway                      | التغييرات المرتبطة بـ Node          |
| `build-smoke`           | اختبارات smoke للـ CLI المبني واختبار smoke لذاكرة بدء التشغيل                           | التغييرات المرتبطة بـ Node          |
| `checks`                | مسارات Linux Node الأثقل: الاختبارات الكاملة، واختبارات القنوات، وتوافق Node 22 للدفع فقط | التغييرات المرتبطة بـ Node          |
| `check-docs`            | تنسيق الوثائق وفحصها واختبارات الروابط المعطلة                                           | عند تغيير الوثائق                   |
| `skills-python`         | ‏Ruff + pytest للـ Skills المعتمدة على Python                                            | التغييرات المرتبطة بـ Python Skills |
| `checks-windows`        | مسارات اختبارات خاصة بـ Windows                                                          | التغييرات المرتبطة بـ Windows       |
| `macos-node`            | مسار اختبارات TypeScript على macOS باستخدام artifacts المبنية المشتركة                   | التغييرات المرتبطة بـ macOS         |
| `macos-swift`           | فحص Swift وبناؤه واختباراته لتطبيق macOS                                                 | التغييرات المرتبطة بـ macOS         |
| `android`               | مصفوفة بناء Android واختباره                                                              | التغييرات المرتبطة بـ Android       |

## ترتيب الإخفاق السريع

تُرتب الوظائف بحيث تفشل الفحوصات الرخيصة قبل تشغيل الوظائف المكلفة:

1. يقرر `preflight` أي المسارات موجودة أصلًا. ومنطق `docs-scope` و`changed-scope` هما خطوتان داخل هذه الوظيفة، وليسا وظيفتين مستقلتين.
2. تفشل `security-fast` و`check` و`check-additional` و`check-docs` و`skills-python` بسرعة من دون انتظار وظائف artifact ومصفوفات المنصات الأثقل.
3. يتداخل `build-artifacts` مع مسارات Linux السريعة حتى يتمكن المستهلكون اللاحقون من البدء بمجرد جاهزية البنية المشتركة.
4. بعد ذلك تتفرع مسارات المنصات ووقت التشغيل الأثقل: `checks-fast-core` و`checks-fast-extensions` و`extension-fast` و`checks` و`checks-windows` و`macos-node` و`macos-swift` و`android`.

يعيش منطق النطاق في `scripts/ci-changed-scope.mjs` وتغطيه اختبارات الوحدة في `src/scripts/ci-changed-scope.test.ts`.
ويعيد workflow المنفصل `install-smoke` استخدام نص النطاق نفسه عبر وظيفة `preflight` الخاصة به. فهو يحسب `run_install_smoke` من إشارة changed-smoke الأضيق، لذلك لا يعمل smoke الخاص بـ Docker/install إلا للتغييرات المرتبطة بالتثبيت والتعبئة والحاويات.

في عمليات الدفع، تضيف مصفوفة `checks` مسار `compat-node22` الخاص بالدفع فقط. أما في pull requests، فيُتخطى هذا المسار وتظل المصفوفة مركزة على مسارات الاختبار/القنوات العادية.

## المشغّلات

| المشغّل                          | الوظائف                                                                                              |
| -------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `blacksmith-16vcpu-ubuntu-2404`  | `preflight` و`security-fast` و`build-artifacts` وفحوصات Linux وفحوصات الوثائق وPython Skills و`android` |
| `blacksmith-32vcpu-windows-2025` | `checks-windows`                                                                                     |
| `macos-latest`                   | `macos-node` و`macos-swift`                                                                          |

## المكافئات المحلية

```bash
pnpm check          # الأنواع + lint + format
pnpm build:strict-smoke
pnpm test:gateway:watch-regression
pnpm test           # اختبارات vitest
pnpm test:channels
pnpm check:docs     # تنسيق الوثائق + lint + الروابط المعطلة
pnpm build          # بناء dist عندما تكون مسارات CI artifact/build-smoke مهمة
```
