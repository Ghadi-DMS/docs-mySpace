---
read_when:
    - تريد فهم OAuth في OpenClaw من البداية إلى النهاية
    - تواجه مشكلات في إبطال الرموز المميزة / تسجيل الخروج
    - تريد تدفقات مصادقة Claude CLI أو OAuth
    - تريد استخدام عدة حسابات أو توجيه الملفات الشخصية
summary: 'OAuth في OpenClaw: تبادل الرموز المميزة، والتخزين، وأنماط الحسابات المتعددة'
title: OAuth
x-i18n:
    generated_at: "2026-04-05T12:41:14Z"
    model: gpt-5.4
    provider: openai
    source_hash: 0b364be2182fcf9082834450f39aecc0913c85fb03237eec1228a589d4851dcd
    source_path: concepts/oauth.md
    workflow: 15
---

# OAuth

يدعم OpenClaw "مصادقة الاشتراك" عبر OAuth لموفري الخدمة الذين يتيحون ذلك
(ولا سيما **OpenAI Codex (ChatGPT OAuth)**). بالنسبة إلى اشتراكات Anthropic، يجب أن
يستخدم الإعداد الجديد مسار تسجيل الدخول المحلي **Claude CLI** على مضيف البوابة، لكن
Anthropic تميز بين استخدام Claude Code المباشر ومسار إعادة الاستخدام الخاص بـ OpenClaw.
وتذكر مستندات Claude Code العامة من Anthropic أن استخدام Claude Code المباشر يبقى
ضمن حدود اشتراك Claude. وبشكل منفصل، أبلغت Anthropic مستخدمي OpenClaw في
**4 أبريل 2026 الساعة 12:00 ظهرًا بتوقيت PT / 8:00 مساءً بتوقيت BST** أن OpenClaw يُحتسب
كحزمة تشغيل تابعة لجهة خارجية، ويتطلب الآن **Extra Usage** لهذا النوع من الحركة.
أما OpenAI Codex OAuth فهو مدعوم صراحةً للاستخدام في الأدوات الخارجية مثل
OpenClaw. تشرح هذه الصفحة ما يلي:

بالنسبة إلى Anthropic في بيئات الإنتاج، تظل مصادقة مفتاح API هي المسار الآمن الموصى به.

- كيفية عمل **تبادل الرموز المميزة** في OAuth ‏(PKCE)
- موضع **تخزين** الرموز المميزة (ولماذا)
- كيفية التعامل مع **حسابات متعددة** (الملفات الشخصية + تجاوزات لكل جلسة)

يدعم OpenClaw أيضًا **plugins للمزوّدين** تشحن تدفقات OAuth أو API-key
الخاصة بها. شغّلها عبر:

```bash
openclaw models auth login --provider <id>
```

## مصرف الرموز المميزة (لماذا هو موجود)

غالبًا ما يُصدر موفرو OAuth **رمز refresh جديدًا** أثناء تدفقات تسجيل الدخول/التحديث. ويمكن لبعض الموفّرين (أو عملاء OAuth) إبطال رموز refresh الأقدم عند إصدار رمز جديد للمستخدم/التطبيق نفسه.

العرض العملي للمشكلة:

- تسجل الدخول عبر OpenClaw _وأيضًا_ عبر Claude Code / Codex CLI → ثم يتعرض أحدهما بشكل عشوائي إلى "تسجيل الخروج" لاحقًا

ولتقليل ذلك، يتعامل OpenClaw مع `auth-profiles.json` على أنه **مصرف رموز مميزة**:

- يقرأ وقت التشغيل بيانات الاعتماد من **مكان واحد**
- يمكننا الاحتفاظ بعدة ملفات شخصية وتوجيهها بشكل حتمي
- عند إعادة استخدام بيانات الاعتماد من CLI خارجي مثل Codex CLI، يقوم OpenClaw
  بعكسها مع مصدرها ويعيد قراءة هذا المصدر الخارجي بدلًا من
  تدوير رمز refresh بنفسه

## التخزين (أين توجد الرموز المميزة)

تُخزَّن الأسرار **لكل وكيل على حدة**:

- ملفات تعريف المصادقة (OAuth + مفاتيح API + مراجع اختيارية على مستوى القيمة): `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
- ملف التوافق القديم: `~/.openclaw/agents/<agentId>/agent/auth.json`
  (تُزال إدخالات `api_key` الثابتة عند اكتشافها)

ملف الاستيراد القديم فقط (ما زال مدعومًا، لكنه ليس المخزن الرئيسي):

- `~/.openclaw/credentials/oauth.json` (يُستورد إلى `auth-profiles.json` عند أول استخدام)

تحترم جميع ما سبق أيضًا `$OPENCLAW_STATE_DIR` (تجاوز دليل الحالة). المرجع الكامل: [/gateway/configuration](/gateway/configuration-reference#auth-storage)

بالنسبة إلى مراجع الأسرار الثابتة وسلوك تفعيل snapshot في وقت التشغيل، راجع [إدارة الأسرار](/gateway/secrets).

## توافق الرموز القديمة لـ Anthropic

<Warning>
تشير مستندات Claude Code العامة من Anthropic إلى أن الاستخدام المباشر لـ Claude Code يبقى ضمن
حدود اشتراك Claude. وبشكل منفصل، أخبرت Anthropic مستخدمي OpenClaw في
**4 أبريل 2026 الساعة 12:00 ظهرًا بتوقيت PT / 8:00 مساءً بتوقيت BST** أن **OpenClaw يُحتسب كحزمة تشغيل
تابعة لجهة خارجية**. وتظل ملفات تعريف رموز Anthropic الحالية قابلة للاستخدام تقنيًا في OpenClaw، لكن
Anthropic تقول إن مسار OpenClaw يتطلب الآن **Extra
Usage** (دفع حسب الاستخدام يُفوتر بشكل منفصل عن الاشتراك) لهذا النوع من الحركة.

للاطلاع على مستندات خطة Claude Code المباشرة الحالية من Anthropic، راجع [Using Claude Code
with your Pro or Max
plan](https://support.claude.com/en/articles/11145838-using-claude-code-with-your-pro-or-max-plan)
و[Using Claude Code with your Team or Enterprise
plan](https://support.anthropic.com/en/articles/11845131-using-claude-code-with-your-team-or-enterprise-plan/).

إذا كنت تريد خيارات أخرى على نمط الاشتراك داخل OpenClaw، فراجع [OpenAI
Codex](/providers/openai)، و[Qwen Cloud Coding
Plan](/providers/qwen)، و[MiniMax Coding Plan](/providers/minimax)،
و[Z.AI / GLM Coding Plan](/providers/glm).
</Warning>

يكشف OpenClaw الآن عن setup-token الخاص بـ Anthropic مرة أخرى كمسار قديم/يدوي.
وما زال إشعار الفوترة الخاص بـ Anthropic والمخصص لـ OpenClaw ينطبق على هذا المسار، لذا
استخدمه مع توقع أن Anthropic تتطلب **Extra Usage** لحركة Claude-login
التي يقودها OpenClaw.

## ترحيل Anthropic Claude CLI

إذا كان Claude CLI مثبتًا بالفعل ومسجل الدخول على مضيف البوابة، فيمكنك
تحويل اختيار نموذج Anthropic إلى الواجهة الخلفية المحلية الخاصة بـ CLI. وهذا
مسار مدعوم في OpenClaw عندما تريد إعادة استخدام تسجيل دخول Claude CLI المحلي على
المضيف نفسه.

المتطلبات المسبقة:

- أن يكون الملف التنفيذي `claude` مثبتًا على مضيف البوابة
- أن يكون Claude CLI قد تمت مصادقته بالفعل هناك عبر `claude auth login`

أمر الترحيل:

```bash
openclaw models auth login --provider anthropic --method cli --set-default
```

اختصار الإعداد التفاعلي:

```bash
openclaw onboard --auth-choice anthropic-cli
```

يحتفظ هذا بملفات تعريف مصادقة Anthropic الحالية للتراجع، لكنه يعيد كتابة المسار
الافتراضي الرئيسي للنموذج من `anthropic/...` إلى `claude-cli/...`، ويعيد كتابة
مسارات التراجع المطابقة لـ Anthropic Claude، ويضيف إدخالات allowlist مطابقة
من `claude-cli/...` ضمن `agents.defaults.models`.

التحقق:

```bash
openclaw models status
```

## تبادل OAuth (كيف يعمل تسجيل الدخول)

تُنفَّذ تدفقات تسجيل الدخول التفاعلية في OpenClaw داخل `@mariozechner/pi-ai` وتُوصَل بالأدلة والأوامر.

### Anthropic Claude CLI

شكل التدفق:

مسار Claude CLI:

1. سجّل الدخول باستخدام `claude auth login` على مضيف البوابة
2. شغّل `openclaw models auth login --provider anthropic --method cli --set-default`
3. لا تخزّن ملف تعريف مصادقة جديدًا؛ بدّل اختيار النموذج إلى `claude-cli/...`
4. احتفظ بملفات تعريف مصادقة Anthropic الحالية للتراجع

تصف مستندات Claude Code العامة من Anthropic تدفق تسجيل دخول اشتراك Claude المباشر هذا
للأداة `claude` نفسها. يستطيع OpenClaw إعادة استخدام هذا التسجيل المحلي، لكن
Anthropic تصنف بشكل منفصل المسار الذي يتحكم فيه OpenClaw على أنه استخدام لحزمة تشغيل تابعة لجهة خارجية لأغراض الفوترة.

مسار المساعد التفاعلي:

- `openclaw onboard` / `openclaw configure` → خيار المصادقة `anthropic-cli`

### OpenAI Codex (ChatGPT OAuth)

يتم دعم OpenAI Codex OAuth صراحةً للاستخدام خارج Codex CLI، بما في ذلك تدفقات عمل OpenClaw.

شكل التدفق (PKCE):

1. إنشاء PKCE verifier/challenge + قيمة `state` عشوائية
2. فتح `https://auth.openai.com/oauth/authorize?...`
3. محاولة التقاط callback على `http://127.0.0.1:1455/auth/callback`
4. إذا تعذر ربط callback (أو كنت بعيدًا/من دون واجهة)، الصق URL/code الخاص بإعادة التوجيه
5. إجراء التبادل عند `https://auth.openai.com/oauth/token`
6. استخراج `accountId` من access token وتخزين `{ access, refresh, expires, accountId }`

مسار الدليل هو `openclaw onboard` → خيار المصادقة `openai-codex`.

## التحديث + انتهاء الصلاحية

تخزن الملفات الشخصية الطابع الزمني `expires`.

في وقت التشغيل:

- إذا كانت `expires` في المستقبل → استخدم access token المخزن
- إذا انتهت صلاحيته → حدّثه (ضمن قفل ملف) واكتب بيانات الاعتماد الجديدة فوق القديمة
- الاستثناء: بيانات الاعتماد المعاد استخدامها من CLI خارجي تبقى مُدارة خارجيًا؛ يعيد OpenClaw
  قراءة مخزن مصادقة CLI ولا يستهلك رمز refresh المنسوخ بنفسه أبدًا

تدفق التحديث تلقائي؛ وعادةً لا تحتاج إلى إدارة الرموز المميزة يدويًا.

## حسابات متعددة (ملفات شخصية) + التوجيه

يوجد نمطان:

### 1) المفضل: وكلاء منفصلون

إذا كنت تريد ألا يتفاعل "الشخصي" و"العمل" أبدًا، فاستخدم وكلاء معزولين (جلسات + بيانات اعتماد + مساحة عمل منفصلة):

```bash
openclaw agents add work
openclaw agents add personal
```

ثم كوّن المصادقة لكل وكيل (عبر الدليل) ووجّه الدردشات إلى الوكيل الصحيح.

### 2) متقدم: ملفات شخصية متعددة داخل وكيل واحد

يدعم `auth-profiles.json` عدة معرّفات ملفات شخصية للمزوّد نفسه.

اختر الملف الشخصي المستخدم عبر:

- بشكل عام عبر ترتيب التكوين (`auth.order`)
- لكل جلسة عبر `/model ...@<profileId>`

مثال (تجاوز على مستوى الجلسة):

- `/model Opus@anthropic:work`

كيفية معرفة معرّفات الملفات الشخصية الموجودة:

- `openclaw channels list --json` (يعرض `auth[]`)

مستندات ذات صلة:

- [/concepts/model-failover](/concepts/model-failover) (قواعد التدوير + فترات التهدئة)
- [/tools/slash-commands](/tools/slash-commands) (سطح الأوامر)

## ذو صلة

- [المصادقة](/gateway/authentication) — نظرة عامة على مصادقة مزوّدي النماذج
- [الأسرار](/gateway/secrets) — تخزين بيانات الاعتماد وSecretRef
- [مرجع التكوين](/gateway/configuration-reference#auth-storage) — مفاتيح تكوين المصادقة
