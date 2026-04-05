---
read_when:
    - تريد ربط أحداث Gmail Pub/Sub بـ OpenClaw
    - تريد أوامر مساعدة webhook
summary: مرجع CLI للأمر `openclaw webhooks` (مساعدات webhook + ‏Gmail Pub/Sub)
title: webhooks
x-i18n:
    generated_at: "2026-04-05T12:39:39Z"
    model: gpt-5.4
    provider: openai
    source_hash: 2b22ce879c3a94557be57919b4d2b3e92ff4d41fbae7bc88d2ab07cd4bbeac83
    source_path: cli/webhooks.md
    workflow: 15
---

# `openclaw webhooks`

مساعدات webhook والتكاملات (Gmail Pub/Sub، ومساعدات webhook).

ذو صلة:

- Webhooks: [Webhooks](/automation/cron-jobs#webhooks)
- Gmail Pub/Sub: [Gmail Pub/Sub](/automation/cron-jobs#gmail-pubsub-integration)

## Gmail

```bash
openclaw webhooks gmail setup --account you@example.com
openclaw webhooks gmail run
```

### `webhooks gmail setup`

تهيئة مراقبة Gmail وPub/Sub وتسليم webhook إلى OpenClaw.

مطلوب:

- `--account <email>`

الخيارات:

- `--project <id>`
- `--topic <name>`
- `--subscription <name>`
- `--label <label>`
- `--hook-url <url>`
- `--hook-token <token>`
- `--push-token <token>`
- `--bind <host>`
- `--port <port>`
- `--path <path>`
- `--include-body`
- `--max-bytes <n>`
- `--renew-minutes <n>`
- `--tailscale <funnel|serve|off>`
- `--tailscale-path <path>`
- `--tailscale-target <target>`
- `--push-endpoint <url>`
- `--json`

أمثلة:

```bash
openclaw webhooks gmail setup --account you@example.com
openclaw webhooks gmail setup --account you@example.com --project my-gcp-project --json
openclaw webhooks gmail setup --account you@example.com --hook-url https://gateway.example.com/hooks/gmail
```

### `webhooks gmail run`

شغّل `gog watch serve` بالإضافة إلى حلقة التجديد التلقائي للمراقبة.

الخيارات:

- `--account <email>`
- `--topic <topic>`
- `--subscription <name>`
- `--label <label>`
- `--hook-url <url>`
- `--hook-token <token>`
- `--push-token <token>`
- `--bind <host>`
- `--port <port>`
- `--path <path>`
- `--include-body`
- `--max-bytes <n>`
- `--renew-minutes <n>`
- `--tailscale <funnel|serve|off>`
- `--tailscale-path <path>`
- `--tailscale-target <target>`

مثال:

```bash
openclaw webhooks gmail run --account you@example.com
```

راجع [وثائق Gmail Pub/Sub](/automation/cron-jobs#gmail-pubsub-integration) للحصول على تدفق الإعداد الكامل والتفاصيل التشغيلية.
