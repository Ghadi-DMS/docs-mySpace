---
read_when:
    - تريد العثور على مكونات OpenClaw الإضافية التابعة لجهات خارجية
    - تريد نشر المكوّن الإضافي الخاص بك أو إدراجه
summary: 'مكونات OpenClaw الإضافية التي يصونها المجتمع: التصفح، والتثبيت، وإرسال المكوّن الإضافي الخاص بك'
title: المكونات الإضافية المجتمعية
x-i18n:
    generated_at: "2026-04-05T12:50:55Z"
    model: gpt-5.4
    provider: openai
    source_hash: 01804563a63399fe564b0cd9b9aadef32e5211b63d8467fdbbd1f988200728de
    source_path: plugins/community.md
    workflow: 15
---

# المكونات الإضافية المجتمعية

المكونات الإضافية المجتمعية هي حزم تابعة لجهات خارجية توسّع OpenClaw بإضافة
قنوات، أو أدوات، أو مزوّدين، أو إمكانات أخرى جديدة. وهي تُبنى وتُصان
بواسطة المجتمع، وتُنشر على [ClawHub](/tools/clawhub) أو npm،
ويمكن تثبيتها بأمر واحد.

تُعد ClawHub سطح الاكتشاف الرسمي للمكونات الإضافية المجتمعية. لا تفتح
طلبات PR خاصة بالوثائق فقط لمجرد إضافة المكوّن الإضافي الخاص بك هنا بغرض الاكتشاف؛ انشره على
ClawHub بدلًا من ذلك.

```bash
openclaw plugins install <package-name>
```

يتحقق OpenClaw من ClawHub أولًا ثم يعود إلى npm تلقائيًا عند الحاجة.

## المكونات الإضافية المدرجة

### Codex App Server Bridge

جسر OpenClaw مستقل لمحادثات Codex App Server. اربط دردشةً
بسلسلة Codex، وتحدث إليها بنص عادي، وتحكم فيها بأوامر أصلية من الدردشة
للاستئناف، والتخطيط، والمراجعة، واختيار النموذج، والضغط، وغير ذلك.

- **npm:** `openclaw-codex-app-server`
- **repo:** [github.com/pwrdrvr/openclaw-codex-app-server](https://github.com/pwrdrvr/openclaw-codex-app-server)

```bash
openclaw plugins install openclaw-codex-app-server
```

### DingTalk

تكامل روبوت للمؤسسات باستخدام وضع Stream. يدعم النصوص، والصور،
ورسائل الملفات عبر أي عميل DingTalk.

- **npm:** `@largezhou/ddingtalk`
- **repo:** [github.com/largezhou/openclaw-dingtalk](https://github.com/largezhou/openclaw-dingtalk)

```bash
openclaw plugins install @largezhou/ddingtalk
```

### Lossless Claw (LCM)

مكوّن إضافي لإدارة السياق دون فقدان في OpenClaw. تلخيص محادثات
يعتمد على DAG مع ضغط تزايدي — يحافظ على دقة السياق الكاملة
مع تقليل استخدام tokens.

- **npm:** `@martian-engineering/lossless-claw`
- **repo:** [github.com/Martian-Engineering/lossless-claw](https://github.com/Martian-Engineering/lossless-claw)

```bash
openclaw plugins install @martian-engineering/lossless-claw
```

### Opik

مكوّن إضافي رسمي يصدّر تتبعات الوكيل إلى Opik. راقب سلوك الوكيل،
والتكلفة، وtokens، والأخطاء، وغير ذلك.

- **npm:** `@opik/opik-openclaw`
- **repo:** [github.com/comet-ml/opik-openclaw](https://github.com/comet-ml/opik-openclaw)

```bash
openclaw plugins install @opik/opik-openclaw
```

### QQbot

صِل OpenClaw بـ QQ عبر QQ Bot API. يدعم الدردشات الخاصة، وإشارات المجموعات،
ورسائل القنوات، والوسائط الغنية بما في ذلك الصوت، والصور، والفيديوهات،
والملفات.

- **npm:** `@tencent-connect/openclaw-qqbot`
- **repo:** [github.com/tencent-connect/openclaw-qqbot](https://github.com/tencent-connect/openclaw-qqbot)

```bash
openclaw plugins install @tencent-connect/openclaw-qqbot
```

### wecom

مكوّن إضافي لقناة WeCom في OpenClaw من فريق Tencent WeCom. يعمل
باتصالات WeCom Bot WebSocket الدائمة، ويدعم
الرسائل المباشرة والدردشات الجماعية، والردود المتدفقة، والمراسلة الاستباقية، ومعالجة الصور/الملفات، وتنسيق Markdown،
والتحكم المدمج في الوصول، وSkills الخاصة بالمستندات/الاجتماعات/المراسلة.

- **npm:** `@wecom/wecom-openclaw-plugin`
- **repo:** [github.com/WecomTeam/wecom-openclaw-plugin](https://github.com/WecomTeam/wecom-openclaw-plugin)

```bash
openclaw plugins install @wecom/wecom-openclaw-plugin
```

## أرسل المكوّن الإضافي الخاص بك

نرحب بالمكونات الإضافية المجتمعية المفيدة، والموثقة، والآمنة في التشغيل.

<Steps>
  <Step title="انشر على ClawHub أو npm">
    يجب أن يكون مكوّنك الإضافي قابلاً للتثبيت عبر `openclaw plugins install \<package-name\>`.
    انشره على [ClawHub](/tools/clawhub) (مفضل) أو npm.
    راجع [Building Plugins](/plugins/building-plugins) للدليل الكامل.

  </Step>

  <Step title="استضفه على GitHub">
    يجب أن تكون الشيفرة المصدرية في مستودع عام مع وثائق إعداد ومتتبع
    للمشكلات.

  </Step>

  <Step title="استخدم طلبات PR الخاصة بالوثائق فقط لتغييرات الوثائق المصدرية">
    لا تحتاج إلى طلب PR للوثائق فقط لجعل المكوّن الإضافي الخاص بك قابلاً للاكتشاف. انشره
    على ClawHub بدلًا من ذلك.

    افتح طلب PR للوثائق فقط عندما تحتاج الوثائق المصدرية لـ OpenClaw إلى
    تغيير فعلي في المحتوى،
    مثل تصحيح إرشادات التثبيت أو إضافة وثائق
    مشتركة بين المستودعات وتنتمي إلى مجموعة الوثائق الرئيسية.

  </Step>
</Steps>

## معيار الجودة

| المتطلب                    | السبب                                              |
| -------------------------- | -------------------------------------------------- |
| منشور على ClawHub أو npm   | يحتاج المستخدمون إلى أن يعمل `openclaw plugins install` |
| مستودع GitHub عام          | مراجعة المصدر، وتتبع المشكلات، والشفافية          |
| وثائق إعداد واستخدام       | يحتاج المستخدمون إلى معرفة كيفية تهيئته            |
| صيانة نشطة                 | تحديثات حديثة أو معالجة سريعة للمشكلات             |

قد تُرفض الأغلفة منخفضة الجهد، أو الملكية غير الواضحة، أو الحزم غير المصانة.

## ذو صلة

- [تثبيت المكونات الإضافية وإعدادها](/tools/plugin) — كيفية تثبيت أي مكوّن إضافي
- [Building Plugins](/plugins/building-plugins) — أنشئ المكوّن الإضافي الخاص بك
- [Plugin Manifest](/plugins/manifest) — schema الخاصة بـ manifest
