---
read_when:
    - تريد استخدام اشتراك Claude Max مع أدوات متوافقة مع OpenAI
    - تريد خادم API محليًا يغلّف Claude Code CLI
    - تريد تقييم الوصول إلى Anthropic القائم على الاشتراك مقابل القائم على API key
summary: وكيل مجتمعي يعرض بيانات اعتماد اشتراك Claude كنقطة نهاية متوافقة مع OpenAI
title: Claude Max API Proxy
x-i18n:
    generated_at: "2026-04-05T12:52:49Z"
    model: gpt-5.4
    provider: openai
    source_hash: 2e125a6a46e48371544adf1331137a1db51e93e905b8c44da482cf2fba180a09
    source_path: providers/claude-max-api-proxy.md
    workflow: 15
---

# Claude Max API Proxy

إن **claude-max-api-proxy** أداة مجتمعية تعرض اشتراك Claude Max/Pro الخاص بك كنقطة نهاية API متوافقة مع OpenAI. وهذا يتيح لك استخدام اشتراكك مع أي أداة تدعم تنسيق OpenAI API.

<Warning>
هذا المسار هو توافق تقني فقط. لقد حجبت Anthropic بعض استخدامات الاشتراك
خارج Claude Code في السابق. ويجب أن تقرر بنفسك ما إذا كنت ستستخدمه
وأن تتحقق من الشروط الحالية لـ Anthropic قبل الاعتماد عليه.
</Warning>

## لماذا قد تستخدم هذا؟

| النهج                    | التكلفة                                              | الأنسب له                                 |
| ------------------------ | ---------------------------------------------------- | ----------------------------------------- |
| Anthropic API            | الدفع لكل token ‏(~$15/M للإدخال، و$75/M للإخراج في Opus) | تطبيقات الإنتاج، والأحجام الكبيرة         |
| اشتراك Claude Max        | $200 شهريًا بسعر ثابت                                | الاستخدام الشخصي، والتطوير، والاستخدام غير المحدود |

إذا كان لديك اشتراك Claude Max وتريد استخدامه مع أدوات متوافقة مع OpenAI، فقد يقلل هذا الوكيل التكلفة لبعض سير العمل. وما تزال API keys تمثل المسار الأوضح من ناحية السياسات في الاستخدامات الإنتاجية.

## كيف يعمل

```
تطبيقك → claude-max-api-proxy → Claude Code CLI → Anthropic (عبر الاشتراك)
   (تنسيق OpenAI)               (يحّول التنسيق)        (يستخدم تسجيل دخولك)
```

يقوم الوكيل بما يلي:

1. يقبل الطلبات بتنسيق OpenAI على `http://localhost:3456/v1/chat/completions`
2. يحولها إلى أوامر Claude Code CLI
3. يعيد الاستجابات بتنسيق OpenAI ‏(مع دعم البث)

## التثبيت

```bash
# يتطلب Node.js 20+ وClaude Code CLI
npm install -g claude-max-api-proxy

# تحقق من أن Claude CLI موثّق
claude --version
```

## الاستخدام

### بدء الخادم

```bash
claude-max-api
# يعمل الخادم على http://localhost:3456
```

### اختبره

```bash
# فحص السلامة
curl http://localhost:3456/health

# عرض النماذج
curl http://localhost:3456/v1/models

# إكمال محادثة
curl http://localhost:3456/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "claude-opus-4",
    "messages": [{"role": "user", "content": "Hello!"}]
  }'
```

### مع OpenClaw

يمكنك توجيه OpenClaw إلى الوكيل باعتباره نقطة نهاية مخصصة متوافقة مع OpenAI:

```json5
{
  env: {
    OPENAI_API_KEY: "not-needed",
    OPENAI_BASE_URL: "http://localhost:3456/v1",
  },
  agents: {
    defaults: {
      model: { primary: "openai/claude-opus-4" },
    },
  },
}
```

يستخدم هذا المسار نفس طريق OpenAI-compatible بنمط الوكيل مثل
واجهات `/v1` المخصصة الأخرى:

- لا يُطبَّق تشكيل الطلبات الأصلي الخاص بـ OpenAI فقط
- لا يوجد `service_tier`، ولا Responses `store`، ولا تلميحات تخزين مؤقت للموجّه،
  ولا تشكيل لحمولة توافق الاستدلال الخاصة بـ OpenAI
- لا تُحقن ترويسات الإسناد المخفية الخاصة بـ OpenClaw (`originator`, `version`, `User-Agent`)
  على URL الخاصة بالوكيل

## النماذج المتاحة

| معرّف النموذج       | يُطابِق           |
| ------------------- | ----------------- |
| `claude-opus-4`     | Claude Opus 4     |
| `claude-sonnet-4`   | Claude Sonnet 4   |
| `claude-haiku-4`    | Claude Haiku 4    |

## التشغيل التلقائي على macOS

أنشئ LaunchAgent لتشغيل الوكيل تلقائيًا:

```bash
cat > ~/Library/LaunchAgents/com.claude-max-api.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.claude-max-api</string>
  <key>RunAtLoad</key>
  <true/>
  <key>KeepAlive</key>
  <true/>
  <key>ProgramArguments</key>
  <array>
    <string>/usr/local/bin/node</string>
    <string>/usr/local/lib/node_modules/claude-max-api-proxy/dist/server/standalone.js</string>
  </array>
  <key>EnvironmentVariables</key>
  <dict>
    <key>PATH</key>
    <string>/usr/local/bin:/opt/homebrew/bin:~/.local/bin:/usr/bin:/bin</string>
  </dict>
</dict>
</plist>
EOF

launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.claude-max-api.plist
```

## الروابط

- **npm:** [https://www.npmjs.com/package/claude-max-api-proxy](https://www.npmjs.com/package/claude-max-api-proxy)
- **GitHub:** [https://github.com/atalovesyou/claude-max-api-proxy](https://github.com/atalovesyou/claude-max-api-proxy)
- **Issues:** [https://github.com/atalovesyou/claude-max-api-proxy/issues](https://github.com/atalovesyou/claude-max-api-proxy/issues)

## ملاحظات

- هذه **أداة مجتمعية**، وليست مدعومة رسميًا من Anthropic أو OpenClaw
- تتطلب اشتراك Claude Max/Pro نشطًا مع Claude Code CLI موثّقًا
- يعمل الوكيل محليًا ولا يرسل البيانات إلى أي خوادم تابعة لطرف ثالث
- الاستجابات المتدفقة مدعومة بالكامل

## انظر أيضًا

- [Anthropic provider](/providers/anthropic) - تكامل OpenClaw الأصلي مع Claude CLI أو API keys
- [OpenAI provider](/providers/openai) - لاشتراكات OpenAI/Codex
