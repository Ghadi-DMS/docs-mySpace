---
read_when:
    - تريد تمكين code_execution أو تكوينه
    - تريد تحليلًا بعيدًا من دون وصول محلي إلى shell
    - تريد الجمع بين x_search أو web_search وتحليل Python البعيد
summary: code_execution -- تشغيل تحليل Python بعيد ومعزول باستخدام xAI
title: Code Execution
x-i18n:
    generated_at: "2026-04-05T12:57:46Z"
    model: gpt-5.4
    provider: openai
    source_hash: 48ca1ddd026cb14837df90ee74859eb98ba6d1a3fbc78da8a72390d0ecee5e40
    source_path: tools/code-execution.md
    workflow: 15
---

# Code Execution

يشغّل `code_execution` تحليل Python بعيدًا ومعزولًا على Responses API من xAI.
وهذا يختلف عن [`exec`](/tools/exec) المحلي:

- يشغّل `exec` أوامر shell على جهازك أو عقدتك
- يشغّل `code_execution` Python في sandbox البعيد الخاص بـ xAI

استخدم `code_execution` من أجل:

- العمليات الحسابية
- إعداد الجداول
- الإحصاءات السريعة
- التحليل على نمط الرسوم البيانية
- تحليل البيانات التي يعيدها `x_search` أو `web_search`

**لا** تستخدمه عندما تحتاج إلى ملفات محلية، أو shell الخاص بك، أو المستودع، أو
الأجهزة المقترنة. استخدم [`exec`](/tools/exec) لذلك.

## الإعداد

تحتاج إلى مفتاح xAI API. أي من هذه الخيارات يعمل:

- `XAI_API_KEY`
- `plugins.entries.xai.config.webSearch.apiKey`

مثال:

```json5
{
  plugins: {
    entries: {
      xai: {
        config: {
          webSearch: {
            apiKey: "xai-...",
          },
          codeExecution: {
            enabled: true,
            model: "grok-4-1-fast",
            maxTurns: 2,
            timeoutSeconds: 30,
          },
        },
      },
    },
  },
}
```

## كيفية استخدامه

اطلب ذلك بشكل طبيعي واجعل نية التحليل واضحة:

```text
Use code_execution to calculate the 7-day moving average for these numbers: ...
```

```text
Use x_search to find posts mentioning OpenClaw this week, then use code_execution to count them by day.
```

```text
Use web_search to gather the latest AI benchmark numbers, then use code_execution to compare percent changes.
```

تأخذ الأداة داخليًا معامل `task` واحدًا، لذا يجب على الوكيل إرسال
طلب التحليل الكامل وأي بيانات مضمنة في مطالبة واحدة.

## الحدود

- هذا تنفيذ بعيد عبر xAI، وليس تنفيذ عمليات محلية.
- يجب التعامل معه على أنه تحليل مؤقت، وليس دفتر ملاحظات دائم.
- لا تفترض وجود وصول إلى الملفات المحلية أو مساحة العمل الخاصة بك.
- للحصول على بيانات X الحديثة، استخدم [`x_search`](/tools/web#x_search) أولًا.

## انظر أيضًا

- [أدوات الويب](/tools/web)
- [Exec](/tools/exec)
- [xAI](/ar/providers/xai)
