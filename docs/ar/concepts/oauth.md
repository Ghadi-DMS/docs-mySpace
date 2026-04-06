---
read_when:
    - تريد فهم OAuth في OpenClaw من البداية إلى النهاية
    - واجهت مشكلات في إبطال الرموز / تسجيل الخروج
    - تريد تدفقات مصادقة Claude CLI أو OAuth
    - تريد استخدام حسابات متعددة أو توجيه ملفات التعريف
summary: 'OAuth في OpenClaw: تبادل الرموز، والتخزين، وأنماط تعدد الحسابات'
title: OAuth
x-i18n:
    generated_at: "2026-04-06T03:07:08Z"
    model: gpt-5.4
    provider: openai
    source_hash: 402e20dfeb6ae87a90cba5824a56a7ba3b964f3716508ea5cc48a47e5affdd73
    source_path: concepts/oauth.md
    workflow: 15
---

# OAuth

يدعم OpenClaw "مصادقة الاشتراك" عبر OAuth للموفرين الذين يتيحونها
(ولا سيما **OpenAI Codex (ChatGPT OAuth)**). أما بالنسبة إلى Anthropic، فأصبح
التقسيم العملي الآن كما يلي:

- **مفتاح Anthropic API**: فوترة عادية لواجهة Anthropic API
- **مصادقة اشتراك Anthropic داخل OpenClaw**: أبلغت Anthropic
  مستخدمي OpenClaw في **4 أبريل 2026 الساعة 12:00 ظهرًا بتوقيت PT / 8:00 مساءً بتوقيت BST** أن هذا يتطلب الآن
  **Extra Usage**

إن OpenAI Codex OAuth مدعوم صراحةً للاستخدام في الأدوات الخارجية مثل
OpenClaw. تشرح هذه الصفحة ما يلي:

بالنسبة إلى Anthropic في بيئات الإنتاج، تبقى مصادقة مفتاح API هي المسار الموصى به والأكثر أمانًا.

- كيف يعمل **تبادل الرموز** في OAuth ‏(PKCE)
- أين يتم **تخزين** الرموز (ولماذا)
- كيفية التعامل مع **حسابات متعددة** (ملفات التعريف + التجاوزات لكل جلسة)

كما يدعم OpenClaw أيضًا **plugins الموفّر** التي تتضمن تدفقات OAuth أو مفاتيح API الخاصة بها. شغّلها عبر:

```bash
openclaw models auth login --provider <id>
```

## مستودع الرموز (لماذا هو موجود)

غالبًا ما ينشئ موفرو OAuth **رمز تحديث جديدًا** أثناء تدفقات تسجيل الدخول/التحديث. وقد تقوم بعض الجهات الموفرة (أو عملاء OAuth) بإبطال رموز التحديث الأقدم عندما يتم إصدار رمز جديد للمستخدم/التطبيق نفسه.

العرض العملي للمشكلة:

- تسجّل الدخول عبر OpenClaw _وأيضًا_ عبر Claude Code / Codex CLI ← ثم يُسجَّل خروج أحدهما عشوائيًا لاحقًا

ولتقليل ذلك، يتعامل OpenClaw مع `auth-profiles.json` باعتباره **مستودع رموز**:

- تقرأ بيئة التشغيل بيانات الاعتماد من **مكان واحد**
- يمكننا الاحتفاظ بملفات تعريف متعددة وتوجيهها بشكل حتمي
- عند إعادة استخدام بيانات الاعتماد من CLI خارجي مثل Codex CLI، فإن OpenClaw
  يعكسها مع معلومات المصدر ويعيد قراءة هذا المصدر الخارجي بدلًا من
  تدوير رمز التحديث بنفسه

## التخزين (أين توجد الرموز)

يتم تخزين الأسرار **لكل وكيل على حدة**:

- ملفات تعريف المصادقة (OAuth + مفاتيح API + مراجع اختيارية على مستوى القيمة): `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
- ملف توافق قديم: `~/.openclaw/agents/<agentId>/agent/auth.json`
  (يتم تنظيف إدخالات `api_key` الثابتة عند اكتشافها)

ملف قديم للاستيراد فقط (لا يزال مدعومًا، لكنه ليس المخزن الرئيسي):

- `~/.openclaw/credentials/oauth.json` (يُستورد إلى `auth-profiles.json` عند أول استخدام)

جميع ما سبق يحترم أيضًا `$OPENCLAW_STATE_DIR` (تجاوز دليل الحالة). المرجع الكامل: [/gateway/configuration](/ar/gateway/configuration-reference#auth-storage)

للاطلاع على مراجع الأسرار الثابتة وسلوك تفعيل لقطات بيئة التشغيل، راجع [إدارة الأسرار](/ar/gateway/secrets).

## توافق رموز Anthropic القديمة

<Warning>
تقول مستندات Claude Code العامة من Anthropic إن الاستخدام المباشر لـ Claude Code يظل ضمن
حدود اشتراك Claude. وبشكل منفصل، أخبرت Anthropic مستخدمي OpenClaw في
**4 أبريل 2026 الساعة 12:00 ظهرًا بتوقيت PT / 8:00 مساءً بتوقيت BST** أن **OpenClaw يُحتسب باعتباره
واجهة تشغيل تابعة لجهة خارجية**. تظل ملفات تعريف رموز Anthropic الحالية قابلة للاستخدام
تقنيًا في OpenClaw، لكن Anthropic تقول إن مسار OpenClaw يتطلب الآن **Extra
Usage** (الدفع حسب الاستخدام مع فوترة منفصلة عن الاشتراك) لتلك
الحركة.

للاطلاع على مستندات الخطط الحالية من Anthropic لاستخدام Claude Code مباشرةً، راجع [Using Claude Code
with your Pro or Max
plan](https://support.claude.com/en/articles/11145838-using-claude-code-with-your-pro-or-max-plan)
و[Using Claude Code with your Team or Enterprise
plan](https://support.anthropic.com/en/articles/11845131-using-claude-code-with-your-team-or-enterprise-plan/).

إذا كنت تريد خيارات أخرى بنمط الاشتراك داخل OpenClaw، فراجع [OpenAI
Codex](/ar/providers/openai)، و[Qwen Cloud Coding
Plan](/ar/providers/qwen)، و[MiniMax Coding Plan](/ar/providers/minimax)،
و[Z.AI / GLM Coding Plan](/ar/providers/glm).
</Warning>

يعرض OpenClaw الآن Anthropic setup-token مجددًا باعتباره مسارًا قديمًا/يدويًا.
ولا يزال إشعار الفوترة الخاص بـ Anthropic والمتعلق بـ OpenClaw ينطبق على هذا المسار، لذا
استخدمه مع توقع أن Anthropic تتطلب **Extra Usage** لحركة
تسجيل دخول Claude المدفوعة بواسطة OpenClaw.

## ترحيل Anthropic Claude CLI

لم يعد لدى Anthropic مسار ترحيل محلي مدعوم لـ Claude CLI داخل
OpenClaw. استخدم مفاتيح Anthropic API لحركة Anthropic، أو احتفظ
بالمصادقة القديمة القائمة على الرموز فقط حيث تكون مُعدّة بالفعل ومع توقع
أن Anthropic تتعامل مع هذا المسار في OpenClaw باعتباره **Extra Usage**.

## تبادل OAuth (كيف يعمل تسجيل الدخول)

يتم تنفيذ تدفقات تسجيل الدخول التفاعلية في OpenClaw داخل `@mariozechner/pi-ai` وربطها بالمعالجات/الأوامر.

### Anthropic setup-token

شكل التدفق:

1. ابدأ Anthropic setup-token أو paste-token من OpenClaw
2. يخزّن OpenClaw بيانات اعتماد Anthropic الناتجة في ملف تعريف مصادقة
3. يظل اختيار النموذج على `anthropic/...`
4. تظل ملفات تعريف مصادقة Anthropic الحالية متاحة للتراجع/التحكم في الترتيب

### OpenAI Codex (ChatGPT OAuth)

إن OpenAI Codex OAuth مدعوم صراحةً للاستخدام خارج Codex CLI، بما في ذلك سير عمل OpenClaw.

شكل التدفق (PKCE):

1. أنشئ PKCE verifier/challenge وقيمة `state` عشوائية
2. افتح `https://auth.openai.com/oauth/authorize?...`
3. حاول التقاط رد النداء على `http://127.0.0.1:1455/auth/callback`
4. إذا تعذر ربط رد النداء (أو كنت تعمل عن بُعد/من دون واجهة)، فألصق URL أو الرمز الخاص بإعادة التوجيه
5. بدّل عبر `https://auth.openai.com/oauth/token`
6. استخرج `accountId` من رمز الوصول وخزّن `{ access, refresh, expires, accountId }`

مسار المعالج هو `openclaw onboard` ← خيار المصادقة `openai-codex`.

## التحديث + انتهاء الصلاحية

تخزن ملفات التعريف طابعًا زمنيًا `expires`.

أثناء التشغيل:

- إذا كانت قيمة `expires` في المستقبل ← استخدم رمز الوصول المخزن
- إذا انتهت الصلاحية ← حدّث (ضمن قفل ملف) واستبدل بيانات الاعتماد المخزنة
- الاستثناء: تظل بيانات الاعتماد المعاد استخدامها من CLI خارجي مُدارة خارجيًا؛ إذ يعيد OpenClaw
  قراءة مخزن مصادقة CLI ولا يستهلك رمز التحديث المنسوخ بنفسه أبدًا

تدفق التحديث تلقائي؛ وبوجه عام لا تحتاج إلى إدارة الرموز يدويًا.

## الحسابات المتعددة (ملفات التعريف) + التوجيه

هناك نمطان:

### 1) المفضل: وكلاء منفصلون

إذا كنت تريد ألا يتفاعل "الشخصي" و"العمل" معًا أبدًا، فاستخدم وكلاء معزولين (جلسات + بيانات اعتماد + مساحة عمل منفصلة):

```bash
openclaw agents add work
openclaw agents add personal
```

ثم اضبط المصادقة لكل وكيل على حدة (المعالج) ووجّه الدردشات إلى الوكيل الصحيح.

### 2) متقدم: ملفات تعريف متعددة في وكيل واحد

يدعم `auth-profiles.json` عدة معرّفات لملفات التعريف للموفر نفسه.

اختر أي ملف تعريف سيتم استخدامه:

- عالميًا عبر ترتيب الإعدادات (`auth.order`)
- لكل جلسة عبر `/model ...@<profileId>`

مثال (تجاوز على مستوى الجلسة):

- `/model Opus@anthropic:work`

كيفية معرفة معرّفات ملفات التعريف الموجودة:

- `openclaw channels list --json` (يعرض `auth[]`)

الوثائق ذات الصلة:

- [/concepts/model-failover](/ar/concepts/model-failover) (قواعد التدوير + التهدئة)
- [/tools/slash-commands](/ar/tools/slash-commands) (سطح الأوامر)

## ذو صلة

- [المصادقة](/ar/gateway/authentication) — نظرة عامة على مصادقة موفر النموذج
- [الأسرار](/ar/gateway/secrets) — تخزين بيانات الاعتماد وSecretRef
- [المرجع الخاص بالإعدادات](/ar/gateway/configuration-reference#auth-storage) — مفاتيح إعدادات المصادقة
