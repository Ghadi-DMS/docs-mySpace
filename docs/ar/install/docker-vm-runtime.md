---
read_when:
    - أنت تنشر OpenClaw على cloud VM باستخدام Docker
    - تحتاج إلى التدفق المشترك لخبز الثنائيات والاستمرارية والتحديث
summary: خطوات وقت تشغيل Docker VM المشتركة لمضيفات OpenClaw Gateway طويلة العمر
title: وقت تشغيل Docker VM
x-i18n:
    generated_at: "2026-04-05T12:46:10Z"
    model: gpt-5.4
    provider: openai
    source_hash: 854403a48fe15a88cc9befb9bebe657f1a7c83f1df2ebe2346fac9a6e4b16992
    source_path: install/docker-vm-runtime.md
    workflow: 15
---

# وقت تشغيل Docker VM

خطوات وقت تشغيل مشتركة لتثبيتات Docker المعتمدة على VM مثل GCP وHetzner ومزودي VPS المماثلين.

## اخبز الثنائيات المطلوبة داخل الصورة

يُعد تثبيت الثنائيات داخل حاوية قيد التشغيل فخًا.
فأي شيء يتم تثبيته وقت التشغيل سيُفقد عند إعادة التشغيل.

يجب تثبيت جميع الثنائيات الخارجية التي تتطلبها Skills في وقت بناء الصورة.

تعرض الأمثلة أدناه ثلاث ثنائيات شائعة فقط:

- `gog` للوصول إلى Gmail
- `goplaces` لـ Google Places
- `wacli` لـ WhatsApp

هذه أمثلة وليست قائمة كاملة.
يمكنك تثبيت أي عدد تحتاجه من الثنائيات باستخدام النمط نفسه.

إذا أضفت Skills جديدة لاحقًا تعتمد على ثنائيات إضافية، فيجب عليك:

1. تحديث Dockerfile
2. إعادة بناء الصورة
3. إعادة تشغيل الحاويات

**مثال Dockerfile**

```dockerfile
FROM node:24-bookworm

RUN apt-get update && apt-get install -y socat && rm -rf /var/lib/apt/lists/*

# Example binary 1: Gmail CLI
RUN curl -L https://github.com/steipete/gog/releases/latest/download/gog_Linux_x86_64.tar.gz \
  | tar -xz -C /usr/local/bin && chmod +x /usr/local/bin/gog

# Example binary 2: Google Places CLI
RUN curl -L https://github.com/steipete/goplaces/releases/latest/download/goplaces_Linux_x86_64.tar.gz \
  | tar -xz -C /usr/local/bin && chmod +x /usr/local/bin/goplaces

# Example binary 3: WhatsApp CLI
RUN curl -L https://github.com/steipete/wacli/releases/latest/download/wacli_Linux_x86_64.tar.gz \
  | tar -xz -C /usr/local/bin && chmod +x /usr/local/bin/wacli

# Add more binaries below using the same pattern

WORKDIR /app
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml .npmrc ./
COPY ui/package.json ./ui/package.json
COPY scripts ./scripts

RUN corepack enable
RUN pnpm install --frozen-lockfile

COPY . .
RUN pnpm build
RUN pnpm ui:install
RUN pnpm ui:build

ENV NODE_ENV=production

CMD ["node","dist/index.js"]
```

<Note>
عناوين URL الخاصة بالتنزيل أعلاه مخصصة لـ x86_64 ‏(amd64). وبالنسبة إلى VMs المعتمدة على ARM ‏(مثل Hetzner ARM أو GCP Tau T2A)، استبدل عناوين URL الخاصة بالتنزيل بمتغيرات ARM64 المناسبة من صفحة الإصدارات الخاصة بكل أداة.
</Note>

## البناء والتشغيل

```bash
docker compose build
docker compose up -d openclaw-gateway
```

إذا فشل البناء مع `Killed` أو `exit code 137` أثناء `pnpm install --frozen-lockfile`، فهذا يعني أن VM نفدت منها الذاكرة.
استخدم فئة جهاز أكبر قبل إعادة المحاولة.

تحقق من الثنائيات:

```bash
docker compose exec openclaw-gateway which gog
docker compose exec openclaw-gateway which goplaces
docker compose exec openclaw-gateway which wacli
```

الخرج المتوقع:

```
/usr/local/bin/gog
/usr/local/bin/goplaces
/usr/local/bin/wacli
```

تحقق من Gateway:

```bash
docker compose logs -f openclaw-gateway
```

الخرج المتوقع:

```
[gateway] listening on ws://0.0.0.0:18789
```

## ما الذي يستمر وأين

يعمل OpenClaw داخل Docker، لكن Docker ليس مصدر الحقيقة.
يجب أن تبقى كل الحالة طويلة العمر عبر عمليات إعادة التشغيل وإعادة البناء وإعادة الإقلاع.

| المكوّن             | الموقع                            | آلية الاستمرارية      | ملاحظات                                                       |
| ------------------- | --------------------------------- | --------------------- | ------------------------------------------------------------- |
| تكوين Gateway       | `/home/node/.openclaw/`           | ربط volume من المضيف  | يتضمن `openclaw.json` و`.env`                                 |
| ملفات تعريف مصادقة النموذج | `/home/node/.openclaw/agents/` | ربط volume من المضيف  | `agents/<agentId>/agent/auth-profiles.json` ‏(OAuth، مفاتيح API) |
| تكوينات Skills      | `/home/node/.openclaw/skills/`    | ربط volume من المضيف  | حالة على مستوى Skill                                          |
| مساحة عمل الوكيل    | `/home/node/.openclaw/workspace/` | ربط volume من المضيف  | الشفرة والمواد الخاصة بالوكيل                                  |
| جلسة WhatsApp       | `/home/node/.openclaw/`           | ربط volume من المضيف  | يحافظ على تسجيل الدخول عبر QR                                 |
| Gmail keyring       | `/home/node/.openclaw/`           | volume من المضيف + كلمة مرور | يتطلب `GOG_KEYRING_PASSWORD`                            |
| الثنائيات الخارجية  | `/usr/local/bin/`                 | صورة Docker           | يجب خبزها في وقت البناء                                        |
| وقت تشغيل Node      | نظام ملفات الحاوية                | صورة Docker           | يُعاد بناؤه في كل بناء للصورة                                  |
| حزم نظام التشغيل    | نظام ملفات الحاوية                | صورة Docker           | لا تثبّتها في وقت التشغيل                                       |
| حاوية Docker        | مؤقتة                             | قابلة لإعادة التشغيل  | من الآمن تدميرها                                               |

## التحديثات

لتحديث OpenClaw على VM:

```bash
git pull
docker compose build
docker compose up -d
```
