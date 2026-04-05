---
read_when:
    - تشغيل أكثر من Gateway واحدة على الجهاز نفسه
    - تحتاج إلى عزل التكوين/الحالة/المنافذ لكل Gateway
summary: تشغيل عدة Gateways من OpenClaw على مضيف واحد (العزل، والمنافذ، والملفات الشخصية)
title: Gateways متعددة
x-i18n:
    generated_at: "2026-04-05T12:43:04Z"
    model: gpt-5.4
    provider: openai
    source_hash: 061f204bf56b28c6bd0e2c9aee6c561a8a162ca219060117fea4d3a007f01899
    source_path: gateway/multiple-gateways.md
    workflow: 15
---

# Gateways متعددة (المضيف نفسه)

يجب أن تستخدم معظم البيئات Gateway واحدة لأن Gateway واحدة يمكنها التعامل مع عدة اتصالات مراسلة وعدة وكلاء. وإذا كنت تحتاج إلى عزل أقوى أو تكرار احتياطي (مثل bot إنقاذ)، فشغّل Gateways منفصلة مع ملفات شخصية/منافذ معزولة.

## قائمة التحقق من العزل (مطلوبة)

- `OPENCLAW_CONFIG_PATH` — ملف تكوين لكل مثيل
- `OPENCLAW_STATE_DIR` — جلسات وبيانات اعتماد وذاكرات cache لكل مثيل
- `agents.defaults.workspace` — جذر مساحة عمل لكل مثيل
- `gateway.port` (أو `--port`) — منفذ فريد لكل مثيل
- يجب ألا تتداخل المنافذ المشتقة (browser/canvas)

إذا كانت هذه العناصر مشتركة، فستواجه سباقات في التكوين وتعارضات في المنافذ.

## الموصى به: الملفات الشخصية (`--profile`)

تقوم الملفات الشخصية تلقائيًا بتحديد نطاق `OPENCLAW_STATE_DIR` و`OPENCLAW_CONFIG_PATH` وتضيف لاحقة إلى أسماء الخدمات.

```bash
# main
openclaw --profile main setup
openclaw --profile main gateway --port 18789

# rescue
openclaw --profile rescue setup
openclaw --profile rescue gateway --port 19001
```

الخدمات لكل ملف شخصي:

```bash
openclaw --profile main gateway install
openclaw --profile rescue gateway install
```

## دليل bot الإنقاذ

شغّل Gateway ثانية على المضيف نفسه مع ما يلي بشكل منفصل:

- ملف شخصي/تكوين
- دليل حالة
- مساحة عمل
- منفذ أساسي (بالإضافة إلى المنافذ المشتقة)

يحافظ هذا على عزل bot الإنقاذ عن bot الرئيسية حتى يتمكن من تصحيح الأخطاء أو تطبيق تغييرات التكوين إذا كانت bot الأساسية متوقفة.

تباعد المنافذ: اترك ما لا يقل عن 20 منفذًا بين المنافذ الأساسية حتى لا تتصادم منافذ browser/canvas/CDP المشتقة مطلقًا.

### كيفية التثبيت (bot الإنقاذ)

```bash
# Main bot (حالية أو جديدة، من دون معامِل --profile)
# تعمل على المنفذ 18789 + منافذ Chrome CDC/Canvas/...‎
openclaw onboard
openclaw gateway install

# Rescue bot (ملف شخصي + منافذ معزولة)
openclaw --profile rescue onboard
# ملاحظات:
# - ستتم إضافة اللاحقة -rescue إلى اسم مساحة العمل افتراضيًا
# - يجب أن يكون المنفذ على الأقل 18789 + 20 منفذًا،
#   ومن الأفضل اختيار منفذ أساسي مختلف تمامًا، مثل 19789،
# - بقية الإعداد التفاعلي مثل الإعداد العادي

# لتثبيت الخدمة (إذا لم يحدث ذلك تلقائيًا أثناء الإعداد)
openclaw --profile rescue gateway install
```

## تعيين المنافذ (المشتقة)

المنفذ الأساسي = `gateway.port` (أو `OPENCLAW_GATEWAY_PORT` / `--port`).

- منفذ خدمة التحكم في browser = الأساسي + 2 (loopback فقط)
- يتم تقديم canvas host على خادم HTTP الخاص بـ Gateway (المنفذ نفسه الخاص بـ `gateway.port`)
- يتم تخصيص منافذ Browser profile CDP تلقائيًا من `browser.controlPort + 9 .. + 108`

إذا قمت بتجاوز أي من هذه القيم في التكوين أو البيئة، فيجب أن تُبقيها فريدة لكل مثيل.

## ملاحظات browser/CDP (مشكلة شائعة)

- **لا** تثبّت `browser.cdpUrl` على القيم نفسها في عدة مثيلات.
- يحتاج كل مثيل إلى منفذ تحكم browser خاص به ونطاق CDP خاص به (مشتق من منفذ gateway الخاص به).
- إذا كنت تحتاج إلى منافذ CDP صريحة، فاضبط `browser.profiles.<name>.cdpPort` لكل مثيل.
- Chrome البعيد: استخدم `browser.profiles.<name>.cdpUrl` (لكل ملف شخصي، ولكل مثيل).

## مثال يدوي باستخدام البيئة

```bash
OPENCLAW_CONFIG_PATH=~/.openclaw/main.json \
OPENCLAW_STATE_DIR=~/.openclaw-main \
openclaw gateway --port 18789

OPENCLAW_CONFIG_PATH=~/.openclaw/rescue.json \
OPENCLAW_STATE_DIR=~/.openclaw-rescue \
openclaw gateway --port 19001
```

## فحوصات سريعة

```bash
openclaw --profile main gateway status --deep
openclaw --profile rescue gateway status --deep
openclaw --profile rescue gateway probe
openclaw --profile main status
openclaw --profile rescue status
openclaw --profile rescue browser status
```

التفسير:

- يساعد `gateway status --deep` في اكتشاف خدمات launchd/systemd/schtasks القديمة المتبقية من عمليات التثبيت الأقدم.
- يُعد نص التحذير في `gateway probe` مثل `multiple reachable gateways detected` متوقعًا فقط عندما تشغّل عمدًا أكثر من gateway معزولة واحدة.
