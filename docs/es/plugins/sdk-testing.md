---
read_when:
    - Estás escribiendo pruebas para un plugin
    - Necesitas utilidades de prueba del SDK de plugins
    - Quieres entender las pruebas de contrato para plugins empaquetados
sidebarTitle: Testing
summary: Utilidades y patrones de pruebas para plugins de OpenClaw
title: Pruebas de plugins
x-i18n:
    generated_at: "2026-04-05T12:50:27Z"
    model: gpt-5.4
    provider: openai
    source_hash: 2e95ed58ed180feadad17bb5138bd09e3b45f1f3ecdff4e2fba4874bb80099fe
    source_path: plugins/sdk-testing.md
    workflow: 15
---

# Pruebas de plugins

Referencia de utilidades de prueba, patrones y aplicación de lint para plugins de OpenClaw.

<Tip>
  **¿Buscas ejemplos de pruebas?** Las guías prácticas incluyen ejemplos completos de pruebas:
  [Pruebas de plugins de canal](/plugins/sdk-channel-plugins#step-6-test) y
  [Pruebas de plugins de proveedor](/plugins/sdk-provider-plugins#step-6-test).
</Tip>

## Utilidades de prueba

**Importar:** `openclaw/plugin-sdk/testing`

La subruta de pruebas exporta un conjunto reducido de helpers para autores de plugins:

```typescript
import {
  installCommonResolveTargetErrorCases,
  shouldAckReaction,
  removeAckReactionAfterReply,
} from "openclaw/plugin-sdk/testing";
```

### Exportaciones disponibles

| Exportación                            | Propósito                                              |
| -------------------------------------- | ------------------------------------------------------ |
| `installCommonResolveTargetErrorCases` | Casos de prueba compartidos para manejo de errores de resolución de destino |
| `shouldAckReaction`                    | Comprueba si un canal debe añadir una reacción de confirmación |
| `removeAckReactionAfterReply`          | Elimina la reacción de confirmación después de entregar la respuesta |

### Tipos

La subruta de pruebas también vuelve a exportar tipos útiles en archivos de prueba:

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

## Pruebas de resolución de destino

Usa `installCommonResolveTargetErrorCases` para añadir casos de error estándar para
la resolución de destino de canales:

```typescript
import { describe } from "vitest";
import { installCommonResolveTargetErrorCases } from "openclaw/plugin-sdk/testing";

describe("resolución de destino de my-channel", () => {
  installCommonResolveTargetErrorCases({
    resolveTarget: ({ to, mode, allowFrom }) => {
      // Lógica de resolución de destino de tu canal
      return myChannelResolveTarget({ to, mode, allowFrom });
    },
    implicitAllowFrom: ["user1", "user2"],
  });

  // Añade casos de prueba específicos del canal
  it("should resolve @username targets", () => {
    // ...
  });
});
```

## Patrones de prueba

### Pruebas unitarias de un plugin de canal

```typescript
import { describe, it, expect, vi } from "vitest";

describe("plugin my-channel", () => {
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
    // No se expone el valor del token
    expect(inspection).not.toHaveProperty("token");
  });
});
```

### Pruebas unitarias de un plugin de proveedor

```typescript
import { describe, it, expect } from "vitest";

describe("plugin my-provider", () => {
  it("should resolve dynamic models", () => {
    const model = myProvider.resolveDynamicModel({
      modelId: "custom-model-v2",
      // ... contexto
    });

    expect(model.id).toBe("custom-model-v2");
    expect(model.provider).toBe("my-provider");
    expect(model.api).toBe("openai-completions");
  });

  it("should return catalog when API key is available", async () => {
    const result = await myProvider.catalog.run({
      resolveProviderApiKey: () => ({ apiKey: "test-key" }),
      // ... contexto
    });

    expect(result?.provider?.models).toHaveLength(2);
  });
});
```

### Mock del runtime del plugin

Para código que usa `createPluginRuntimeStore`, haz mock del runtime en las pruebas:

```typescript
import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
import type { PluginRuntime } from "openclaw/plugin-sdk/runtime-store";

const store = createPluginRuntimeStore<PluginRuntime>("test runtime not set");

// En la configuración de la prueba
const mockRuntime = {
  agent: {
    resolveAgentDir: vi.fn().mockReturnValue("/tmp/agent"),
    // ... otros mocks
  },
  config: {
    loadConfig: vi.fn(),
    writeConfigFile: vi.fn(),
  },
  // ... otros espacios de nombres
} as unknown as PluginRuntime;

store.setRuntime(mockRuntime);

// Después de las pruebas
store.clearRuntime();
```

### Pruebas con stubs por instancia

Prefiere stubs por instancia en lugar de mutar prototypes:

```typescript
// Preferido: stub por instancia
const client = new MyChannelClient();
client.sendMessage = vi.fn().mockResolvedValue({ id: "msg-1" });

// Evitar: mutación del prototype
// MyChannelClient.prototype.sendMessage = vi.fn();
```

## Pruebas de contrato (plugins dentro del repositorio)

Los plugins empaquetados tienen pruebas de contrato que verifican la propiedad del registro:

```bash
pnpm test -- src/plugins/contracts/
```

Estas pruebas validan:

- Qué plugins registran qué proveedores
- Qué plugins registran qué proveedores de voz
- La corrección de la forma del registro
- El cumplimiento del contrato de runtime

### Ejecutar pruebas acotadas

Para un plugin específico:

```bash
pnpm test -- <bundled-plugin-root>/my-channel/
```

Solo para pruebas de contrato:

```bash
pnpm test -- src/plugins/contracts/shape.contract.test.ts
pnpm test -- src/plugins/contracts/auth.contract.test.ts
pnpm test -- src/plugins/contracts/runtime.contract.test.ts
```

## Aplicación de lint (plugins dentro del repositorio)

`pnpm check` aplica tres reglas para plugins dentro del repositorio:

1. **Sin importaciones monolíticas de la raíz** -- se rechaza el barrel raíz `openclaw/plugin-sdk`
2. **Sin importaciones directas de `src/`** -- los plugins no pueden importar `../../src/` directamente
3. **Sin autoimportaciones** -- los plugins no pueden importar su propia subruta `plugin-sdk/<name>`

Los plugins externos no están sujetos a estas reglas de lint, pero se recomienda seguir los mismos
patrones.

## Configuración de pruebas

OpenClaw usa Vitest con umbrales de cobertura V8. Para pruebas de plugins:

```bash
# Ejecutar todas las pruebas
pnpm test

# Ejecutar pruebas de un plugin específico
pnpm test -- <bundled-plugin-root>/my-channel/src/channel.test.ts

# Ejecutar con un filtro por nombre de prueba específico
pnpm test -- <bundled-plugin-root>/my-channel/ -t "resolves account"

# Ejecutar con cobertura
pnpm test:coverage
```

Si las ejecuciones locales causan presión de memoria:

```bash
OPENCLAW_VITEST_MAX_WORKERS=1 pnpm test
```

## Relacionado

- [Resumen del SDK](/plugins/sdk-overview) -- convenciones de importación
- [SDK de plugins de canal](/plugins/sdk-channel-plugins) -- interfaz de plugins de canal
- [SDK de plugins de proveedor](/plugins/sdk-provider-plugins) -- hooks de plugins de proveedor
- [Crear plugins](/plugins/building-plugins) -- guía de introducción
