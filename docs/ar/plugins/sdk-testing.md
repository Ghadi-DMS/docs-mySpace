---
read_when:
    - أنت تكتب اختبارات لـ plugin
    - أنت بحاجة إلى أدوات اختبار من Plugin SDK
    - أنت تريد فهم اختبارات العقد الخاصة بالـ plugins المضمّنة
sidebarTitle: Testing
summary: أدوات وأنماط الاختبار الخاصة بـ Plugins في OpenClaw
title: اختبار Plugins
x-i18n:
    generated_at: "2026-04-05T12:52:16Z"
    model: gpt-5.4
    provider: openai
    source_hash: 2e95ed58ed180feadad17bb5138bd09e3b45f1f3ecdff4e2fba4874bb80099fe
    source_path: plugins/sdk-testing.md
    workflow: 15
---

# اختبار Plugins

مرجع لأدوات الاختبار والأنماط وفرض lint الخاصة بـ Plugins في OpenClaw.

<Tip>
  **هل تبحث عن أمثلة اختبارات؟** تتضمن الأدلة العملية أمثلة اختبارات جاهزة:
  [اختبارات plugin القناة](/plugins/sdk-channel-plugins#step-6-test) و
  [اختبارات plugin الموفّر](/plugins/sdk-provider-plugins#step-6-test).
</Tip>

## أدوات الاختبار

**الاستيراد:** `openclaw/plugin-sdk/testing`

يكشف المسار الفرعي الخاص بالاختبار مجموعة ضيقة من المساعدات لمطوري plugins:

```typescript
import {
  installCommonResolveTargetErrorCases,
  shouldAckReaction,
  removeAckReactionAfterReply,
} from "openclaw/plugin-sdk/testing";
```

### الصادرات المتاحة

| التصدير | الغرض |
| -------------------------------------- | ------------------------------------------------------ |
| `installCommonResolveTargetErrorCases` | حالات اختبار مشتركة لأخطاء تحليل الهدف |
| `shouldAckReaction`                    | التحقق مما إذا كانت القناة يجب أن تضيف تفاعل ack |
| `removeAckReactionAfterReply`          | إزالة تفاعل ack بعد تسليم الرد |

### الأنواع

يعيد المسار الفرعي الخاص بالاختبار أيضًا تصدير أنواع مفيدة داخل ملفات الاختبار:

```typescript
import type {
  ChannelAccountSnapshot,
  ChannelGatewayContext,
  OpenClawConfig,
  PluginRuntime,
  RuntimeEnv,
  MockFn,
} from "openclaw/plugin-sdk/testing";
```

## اختبار تحليل الأهداف

استخدم `installCommonResolveTargetErrorCases` لإضافة حالات الأخطاء القياسية
لتحليل أهداف القناة:

```typescript
import { describe } from "vitest";
import { installCommonResolveTargetErrorCases } from "openclaw/plugin-sdk/testing";

describe("my-channel target resolution", () => {
  installCommonResolveTargetErrorCases({
    resolveTarget: ({ to, mode, allowFrom }) => {
      // Your channel's target resolution logic
      return myChannelResolveTarget({ to, mode, allowFrom });
    },
    implicitAllowFrom: ["user1", "user2"],
  });

  // Add channel-specific test cases
  it("should resolve @username targets", () => {
    // ...
  });
});
```

## أنماط الاختبار

### اختبار وحدة لـ plugin قناة

```typescript
import { describe, it, expect, vi } from "vitest";

describe("my-channel plugin", () => {
  it("should resolve account from config", () => {
    const cfg = {
      channels: {
        "my-channel": {
          token: "test-token",
          allowFrom: ["user1"],
        },
      },
    };

    const account = myPlugin.setup.resolveAccount(cfg, undefined);
    expect(account.token).toBe("test-token");
  });

  it("should inspect account without materializing secrets", () => {
    const cfg = {
      channels: {
        "my-channel": { token: "test-token" },
      },
    };

    const inspection = myPlugin.setup.inspectAccount(cfg, undefined);
    expect(inspection.configured).toBe(true);
    expect(inspection.tokenStatus).toBe("available");
    // No token value exposed
    expect(inspection).not.toHaveProperty("token");
  });
});
```

### اختبار وحدة لـ plugin موفّر

```typescript
import { describe, it, expect } from "vitest";

describe("my-provider plugin", () => {
  it("should resolve dynamic models", () => {
    const model = myProvider.resolveDynamicModel({
      modelId: "custom-model-v2",
      // ... context
    });

    expect(model.id).toBe("custom-model-v2");
    expect(model.provider).toBe("my-provider");
    expect(model.api).toBe("openai-completions");
  });

  it("should return catalog when API key is available", async () => {
    const result = await myProvider.catalog.run({
      resolveProviderApiKey: () => ({ apiKey: "test-key" }),
      // ... context
    });

    expect(result?.provider?.models).toHaveLength(2);
  });
});
```

### محاكاة وقت تشغيل plugin

بالنسبة إلى الشيفرة التي تستخدم `createPluginRuntimeStore`، قم بمحاكاة وقت التشغيل في الاختبارات:

```typescript
import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
import type { PluginRuntime } from "openclaw/plugin-sdk/runtime-store";

const store = createPluginRuntimeStore<PluginRuntime>("test runtime not set");

// In test setup
const mockRuntime = {
  agent: {
    resolveAgentDir: vi.fn().mockReturnValue("/tmp/agent"),
    // ... other mocks
  },
  config: {
    loadConfig: vi.fn(),
    writeConfigFile: vi.fn(),
  },
  // ... other namespaces
} as unknown as PluginRuntime;

store.setRuntime(mockRuntime);

// After tests
store.clearRuntime();
```

### الاختبار باستخدام stubs لكل مثيل

فضّل استخدام stubs لكل مثيل بدلًا من تغيير prototype:

```typescript
// Preferred: per-instance stub
const client = new MyChannelClient();
client.sendMessage = vi.fn().mockResolvedValue({ id: "msg-1" });

// Avoid: prototype mutation
// MyChannelClient.prototype.sendMessage = vi.fn();
```

## اختبارات العقد (plugins داخل المستودع)

تحتوي plugins المضمّنة على اختبارات عقد تتحقق من ملكية التسجيل:

```bash
pnpm test -- src/plugins/contracts/
```

تؤكد هذه الاختبارات:

- أي plugins تسجل أي موفّرين
- أي plugins تسجل أي موفري كلام
- صحة شكل التسجيل
- الامتثال لعقد وقت التشغيل

### تشغيل اختبارات محددة النطاق

بالنسبة إلى plugin محدد:

```bash
pnpm test -- <bundled-plugin-root>/my-channel/
```

لاختبارات العقد فقط:

```bash
pnpm test -- src/plugins/contracts/shape.contract.test.ts
pnpm test -- src/plugins/contracts/auth.contract.test.ts
pnpm test -- src/plugins/contracts/runtime.contract.test.ts
```

## فرض lint ‏(plugins داخل المستودع)

يتم فرض ثلاث قواعد بواسطة `pnpm check` على plugins داخل المستودع:

1. **لا استيرادات جذرية ضخمة** -- يتم رفض root barrel ‏`openclaw/plugin-sdk`
2. **لا استيرادات مباشرة من `src/`** -- لا يمكن للـ plugins الاستيراد من `../../src/` مباشرة
3. **لا استيرادات ذاتية** -- لا يمكن للـ plugins استيراد مسارها الفرعي `plugin-sdk/<name>` الخاص بها

لا تخضع plugins الخارجية لقواعد lint هذه، لكن يُوصى باتباع الأنماط نفسها.

## إعداد الاختبار

يستخدم OpenClaw أداة Vitest مع حدود تغطية V8. وبالنسبة إلى اختبارات plugins:

```bash
# Run all tests
pnpm test

# Run specific plugin tests
pnpm test -- <bundled-plugin-root>/my-channel/src/channel.test.ts

# Run with a specific test name filter
pnpm test -- <bundled-plugin-root>/my-channel/ -t "resolves account"

# Run with coverage
pnpm test:coverage
```

إذا سببت التشغيلات المحلية ضغطًا على الذاكرة:

```bash
OPENCLAW_VITEST_MAX_WORKERS=1 pnpm test
```

## ذو صلة

- [نظرة عامة على SDK](/plugins/sdk-overview) -- اصطلاحات الاستيراد
- [SDK الخاص بـ Plugins القنوات](/plugins/sdk-channel-plugins) -- واجهة plugin القناة
- [SDK الخاص بـ Plugins الموفّر](/plugins/sdk-provider-plugins) -- خطافات plugin الموفّر
- [بناء Plugins](/plugins/building-plugins) -- دليل البدء
