---
x-i18n:
    generated_at: "2026-04-05T12:34:06Z"
    model: gpt-5.4
    provider: openai
    source_hash: adff26fa8858af2759b231ea48bfc01f89c110cd9b3774a8f783e282c16f77fb
    source_path: .i18n/README.md
    workflow: 15
---

# أصول التدويل لوثائق OpenClaw

يخزن هذا المجلد إعدادات الترجمة لمستودع وثائق المصدر.

توجد الآن أشجار اللغات المُولدة وذاكرة الترجمة الحية في مستودع النشر:

- المستودع: `openclaw/docs`
- النسخة المحلية: `~/Projects/openclaw-docs`

## المصدر المعتمد

- تتم كتابة الوثائق الإنجليزية في `openclaw/openclaw`.
- توجد شجرة وثائق المصدر ضمن `docs/`.
- لم يعد مستودع المصدر يحتفظ بأشجار اللغات المُولدة الملتزم بها مثل `docs/zh-CN/**` و`docs/ja-JP/**` و`docs/es/**` و`docs/pt-BR/**` و`docs/ko/**` و`docs/de/**` و`docs/fr/**` أو `docs/ar/**`.

## التدفق الكامل

1. حرر الوثائق الإنجليزية في `openclaw/openclaw`.
2. ادفع إلى `main`.
3. يقوم `openclaw/openclaw/.github/workflows/docs-sync-publish.yml` بعكس شجرة الوثائق إلى `openclaw/docs`.
4. يعيد نص المزامنة كتابة ملف النشر `docs/docs.json` بحيث توجد كتل منتقي اللغات المُولدة هناك رغم أنها لم تعد مُلتزمًا بها في مستودع المصدر.
5. يقوم `openclaw/docs/.github/workflows/translate-zh-cn.yml` بتحديث `docs/zh-CN/**` مرة يوميًا، وعند الطلب، وبعد عمليات إرسال الإصدارات من مستودع المصدر.
6. يقوم `openclaw/docs/.github/workflows/translate-ja-jp.yml` بالأمر نفسه لـ `docs/ja-JP/**`.
7. تقوم `openclaw/docs/.github/workflows/translate-es.yml` و`translate-pt-br.yml` و`translate-ko.yml` و`translate-de.yml` و`translate-fr.yml` و`translate-ar.yml` بالأمر نفسه لـ `docs/es/**` و`docs/pt-BR/**` و`docs/ko/**` و`docs/de/**` و`docs/fr/**` و`docs/ar/**`.

## سبب هذا الفصل

- إبقاء مخرجات اللغات المُولدة خارج مستودع المنتج الرئيسي.
- إبقاء Mintlify على شجرة وثائق منشورة واحدة.
- الحفاظ على مبدل اللغة المدمج عبر جعل مستودع النشر يملك أشجار اللغات المُولدة.

## الملفات في هذا المجلد

- `glossary.<lang>.json` — تعيينات المصطلحات المفضلة المستخدمة كإرشادات للمطالبة.
- `ar-navigation.json` و`de-navigation.json` و`es-navigation.json` و`fr-navigation.json` و`ja-navigation.json` و`ko-navigation.json` و`pt-BR-navigation.json` و`zh-Hans-navigation.json` — كتل منتقي اللغات في Mintlify التي يُعاد إدراجها في مستودع النشر أثناء المزامنة.
- `<lang>.tm.jsonl` — ذاكرة ترجمة مفهرسة بحسب سير العمل + النموذج + تجزئة النص.

في هذا المستودع، لم تعد ملفات TM الخاصة باللغات المُولدة مثل `docs/.i18n/zh-CN.tm.jsonl` و`docs/.i18n/ja-JP.tm.jsonl` و`docs/.i18n/es.tm.jsonl` و`docs/.i18n/pt-BR.tm.jsonl` و`docs/.i18n/ko.tm.jsonl` و`docs/.i18n/de.tm.jsonl` و`docs/.i18n/fr.tm.jsonl` و`docs/.i18n/ar.tm.jsonl` مُلتزمًا بها عمدًا.

## تنسيق المسرد

`glossary.<lang>.json` عبارة عن مصفوفة من الإدخالات:

```json
{
  "source": "troubleshooting",
  "target": "故障排除"
}
```

الحقول:

- `source`: العبارة الإنجليزية (أو عبارة المصدر) المفضلة.
- `target`: مخرجات الترجمة المفضلة.

## آلية الترجمة

- ما يزال `scripts/docs-i18n` مسؤولًا عن توليد الترجمة.
- يكتب وضع الوثائق `x-i18n.source_hash` داخل كل صفحة مترجمة.
- يحسب كل سير عمل نشر مسبقًا قائمة الملفات المعلقة عبر مقارنة تجزئة المصدر الإنجليزي الحالية مع `x-i18n.source_hash` المخزن للغة.
- إذا كان عدد الملفات المعلقة `0`، فسيتم تخطي خطوة الترجمة المكلفة بالكامل.
- إذا كانت هناك ملفات معلقة، يترجم سير العمل تلك الملفات فقط.
- يعيد سير عمل النشر المحاولة عند حدوث إخفاقات عابرة في تنسيق النموذج، لكن الملفات غير المتغيرة تبقى متخطاة لأن فحص التجزئة نفسه يعمل في كل إعادة محاولة.
- يرسل مستودع المصدر أيضًا تحديثات zh-CN وja-JP وes وpt-BR وko وde وfr وar بعد إصدارات GitHub المنشورة حتى تتمكن وثائق الإصدار من اللحاق دون انتظار المهمة اليومية المجدولة.

## ملاحظات تشغيلية

- تتم كتابة بيانات تعريف المزامنة إلى `.openclaw-sync/source.json` في مستودع النشر.
- السر في مستودع المصدر: `OPENCLAW_DOCS_SYNC_TOKEN`
- السر في مستودع النشر: `OPENCLAW_DOCS_I18N_OPENAI_API_KEY`
- إذا بدا أن مخرجات اللغة قديمة، فتحقق أولًا من سير عمل `Translate <locale>` المطابق في `openclaw/docs`.
