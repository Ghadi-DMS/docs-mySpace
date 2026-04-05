---
read_when:
    - تريد قائمة كاملة بما يدعمه OpenClaw
summary: إمكانات OpenClaw عبر القنوات، والتوجيه، والوسائط، وتجربة الاستخدام.
title: الميزات
x-i18n:
    generated_at: "2026-04-05T12:40:05Z"
    model: gpt-5.4
    provider: openai
    source_hash: 43eae89d9af44ea786dd0221d8d602ebcea15da9d5064396ac9920c0345e2ad3
    source_path: concepts/features.md
    workflow: 15
---

# الميزات

## أبرز المزايا

<Columns>
  <Card title="القنوات" icon="message-square">
    Discord وiMessage وSignal وSlack وTelegram وWhatsApp وWebChat والمزيد عبر Gateway واحد.
  </Card>
  <Card title="الإضافات" icon="plug">
    تضيف الإضافات المدمجة Matrix وNextcloud Talk وNostr وTwitch وZalo والمزيد من دون عمليات تثبيت منفصلة في الإصدارات الحالية العادية.
  </Card>
  <Card title="التوجيه" icon="route">
    توجيه متعدد الوكلاء مع جلسات معزولة.
  </Card>
  <Card title="الوسائط" icon="image">
    صور وصوت وفيديو ومستندات، بالإضافة إلى إنشاء الصور/الفيديو.
  </Card>
  <Card title="التطبيقات وواجهة المستخدم" icon="monitor">
    واجهة تحكم ويب وتطبيق macOS مرافق.
  </Card>
  <Card title="عقد الهاتف المحمول" icon="smartphone">
    عقد iOS وAndroid مع الاقتران، والصوت/الدردشة، وأوامر الجهاز الغنية.
  </Card>
</Columns>

## القائمة الكاملة

**القنوات:**

- تتضمن القنوات المضمنة Discord وGoogle Chat وiMessage (legacy) وIRC وSignal وSlack وTelegram وWebChat وWhatsApp
- تتضمن قنوات الإضافات المدمجة BlueBubbles لـ iMessage وFeishu وLINE وMatrix وMattermost وMicrosoft Teams وNextcloud Talk وNostr وQQ Bot وSynology Chat وTlon وTwitch وZalo وZalo Personal
- تتضمن إضافات القنوات الاختيارية المثبتة بشكل منفصل Voice Call وحزم الجهات الخارجية مثل WeChat
- يمكن لإضافات القنوات الخارجية توسيع Gateway أكثر، مثل WeChat
- دعم الدردشة الجماعية مع تفعيل قائم على الإشارات
- أمان الرسائل المباشرة عبر قوائم السماح والاقتران

**الوكيل:**

- بيئة تشغيل وكيل مدمجة مع بث الأدوات
- توجيه متعدد الوكلاء مع جلسات معزولة لكل مساحة عمل أو مرسل
- الجلسات: تُطوى الدردشات المباشرة إلى `main` مشتركة؛ وتبقى المجموعات معزولة
- البث والتجزئة للردود الطويلة

**المصادقة والموفّرون:**

- أكثر من 35 موفّر نماذج (Anthropic وOpenAI وGoogle وغير ذلك)
- مصادقة اشتراك عبر OAuth (مثل OpenAI Codex)
- دعم الموفّرين المخصصين والمستضافين ذاتيًا (vLLM وSGLang وOllama وأي نقطة نهاية متوافقة مع OpenAI أو Anthropic)

**الوسائط:**

- صور وصوت وفيديو ومستندات في الاتجاهين
- واجهات قدرات مشتركة لإنشاء الصور وإنشاء الفيديو
- نسخ صوتي للملاحظات الصوتية
- تحويل النص إلى كلام عبر عدة موفّرين

**التطبيقات والواجهات:**

- WebChat وواجهة تحكم في المتصفح
- تطبيق مرافق macOS في شريط القوائم
- عقدة iOS مع الاقتران وCanvas والكاميرا وتسجيل الشاشة والموقع والصوت
- عقدة Android مع الاقتران والدردشة والصوت وCanvas والكاميرا وأوامر الجهاز

**الأدوات والأتمتة:**

- أتمتة المتصفح وexec وsandboxing
- بحث الويب (Brave وDuckDuckGo وExa وFirecrawl وGemini وGrok وKimi وMiniMax Search وOllama Web Search وPerplexity وSearXNG وTavily)
- مهام Cron وجدولة heartbeat
- Skills والإضافات ومسارات workflow ‏(Lobster)
