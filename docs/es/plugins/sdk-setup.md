---
read_when:
    - Estás agregando un asistente de configuración a un plugin
    - Necesitas entender `setup-entry.ts` frente a `index.ts`
    - Estás definiendo esquemas de configuración de plugins o metadatos `openclaw` en `package.json`
sidebarTitle: Setup and Config
summary: Asistentes de configuración, `setup-entry.ts`, esquemas de configuración y metadatos de `package.json`
title: Setup y configuración de plugins
x-i18n:
    generated_at: "2026-04-05T12:50:39Z"
    model: gpt-5.4
    provider: openai
    source_hash: 68fda27be1c89ea6ba906833113e9190ddd0ab358eb024262fb806746d54f7bf
    source_path: plugins/sdk-setup.md
    workflow: 15
---

# Setup y configuración de plugins

Referencia para el empaquetado de plugins (metadatos de `package.json`), manifiestos
(`openclaw.plugin.json`), entradas de setup y esquemas de configuración.

<Tip>
  **¿Buscas una guía paso a paso?** Las guías prácticas cubren el empaquetado en contexto:
  [Channel Plugins](/plugins/sdk-channel-plugins#step-1-package-and-manifest) y
  [Provider Plugins](/plugins/sdk-provider-plugins#step-1-package-and-manifest).
</Tip>

## Metadatos del paquete

Tu `package.json` necesita un campo `openclaw` que indique al sistema de plugins qué
proporciona tu plugin:

**Plugin de canal:**

```json
{
  "name": "@myorg/openclaw-my-channel",
  "version": "1.0.0",
  "type": "module",
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "channel": {
      "id": "my-channel",
      "label": "My Channel",
      "blurb": "Short description of the channel."
    }
  }
}
```

**Plugin de proveedor / línea base de publicación en ClawHub:**

```json openclaw-clawhub-package.json
{
  "name": "@myorg/openclaw-my-plugin",
  "version": "1.0.0",
  "type": "module",
  "openclaw": {
    "extensions": ["./index.ts"],
    "compat": {
      "pluginApi": ">=2026.3.24-beta.2",
      "minGatewayVersion": "2026.3.24-beta.2"
    },
    "build": {
      "openclawVersion": "2026.3.24-beta.2",
      "pluginSdkVersion": "2026.3.24-beta.2"
    }
  }
}
```

Si publicas el plugin externamente en ClawHub, esos campos `compat` y `build`
son obligatorios. Los fragmentos canónicos de publicación están en
`docs/snippets/plugin-publish/`.

### Campos `openclaw`

| Field        | Type       | Descripción                                                                                         |
| ------------ | ---------- | --------------------------------------------------------------------------------------------------- |
| `extensions` | `string[]` | Archivos de punto de entrada (relativos a la raíz del paquete)                                      |
| `setupEntry` | `string`   | Entrada ligera solo para setup (opcional)                                                           |
| `channel`    | `object`   | Metadatos del catálogo de canales para superficies de setup, selector, quickstart y estado          |
| `providers`  | `string[]` | ID de proveedores registrados por este plugin                                                       |
| `install`    | `object`   | Pistas de instalación: `npmSpec`, `localPath`, `defaultChoice`, `minHostVersion`, `allowInvalidConfigRecovery` |
| `startup`    | `object`   | Banderas de comportamiento de inicio                                                                |

### `openclaw.channel`

`openclaw.channel` es metadato barato del paquete para superficies de descubrimiento y setup
de canales antes de que cargue el runtime.

| Field                                  | Type       | Qué significa                                                                   |
| -------------------------------------- | ---------- | ------------------------------------------------------------------------------- |
| `id`                                   | `string`   | ID canónico del canal.                                                          |
| `label`                                | `string`   | Etiqueta principal del canal.                                                   |
| `selectionLabel`                       | `string`   | Etiqueta del selector/setup cuando deba diferir de `label`.                     |
| `detailLabel`                          | `string`   | Etiqueta secundaria de detalle para catálogos de canales y superficies de estado más ricos. |
| `docsPath`                             | `string`   | Ruta de documentación para enlaces de setup y selección.                        |
| `docsLabel`                            | `string`   | Etiqueta sobrescrita usada para enlaces de documentación cuando deba diferir del id del canal. |
| `blurb`                                | `string`   | Descripción corta para onboarding/catálogo.                                     |
| `order`                                | `number`   | Orden de clasificación en los catálogos de canales.                             |
| `aliases`                              | `string[]` | Alias de búsqueda adicionales para la selección del canal.                      |
| `preferOver`                           | `string[]` | ID de plugin/canal de menor prioridad a los que este canal debe superar.        |
| `systemImage`                          | `string`   | Nombre opcional de icono/system-image para catálogos UI de canales.             |
| `selectionDocsPrefix`                  | `string`   | Texto prefijo antes de los enlaces de documentación en superficies de selección. |
| `selectionDocsOmitLabel`               | `boolean`  | Muestra la ruta de documentación directamente en lugar de un enlace etiquetado en el texto de selección. |
| `selectionExtras`                      | `string[]` | Cadenas cortas extra añadidas al texto de selección.                            |
| `markdownCapable`                      | `boolean`  | Marca el canal como compatible con markdown para decisiones de formato saliente. |
| `showConfigured`                       | `boolean`  | Controla si las superficies de lista de canales configurados muestran este canal. |
| `quickstartAllowFrom`                  | `boolean`  | Incluye este canal en el flujo estándar de setup rápido `allowFrom`.            |
| `forceAccountBinding`                  | `boolean`  | Requiere enlace explícito de cuenta incluso cuando solo existe una cuenta.      |
| `preferSessionLookupForAnnounceTarget` | `boolean`  | Prefiere búsqueda por sesión al resolver destinos de announce para este canal.  |

Ejemplo:

```json
{
  "openclaw": {
    "channel": {
      "id": "my-channel",
      "label": "My Channel",
      "selectionLabel": "My Channel (self-hosted)",
      "detailLabel": "My Channel Bot",
      "docsPath": "/channels/my-channel",
      "docsLabel": "my-channel",
      "blurb": "Webhook-based self-hosted chat integration.",
      "order": 80,
      "aliases": ["mc"],
      "preferOver": ["my-channel-legacy"],
      "selectionDocsPrefix": "Guide:",
      "selectionExtras": ["Markdown"],
      "markdownCapable": true,
      "quickstartAllowFrom": true
    }
  }
}
```

### `openclaw.install`

`openclaw.install` es metadato del paquete, no metadato del manifiesto.

| Field                        | Type                 | Qué significa                                                                      |
| ---------------------------- | -------------------- | ---------------------------------------------------------------------------------- |
| `npmSpec`                    | `string`             | Especificación npm canónica para flujos de instalación/actualización.              |
| `localPath`                  | `string`             | Ruta local de desarrollo o instalación integrada.                                  |
| `defaultChoice`              | `"npm"` \| `"local"` | Fuente de instalación preferida cuando ambas están disponibles.                    |
| `minHostVersion`             | `string`             | Versión mínima compatible de OpenClaw con el formato `>=x.y.z`.                    |
| `allowInvalidConfigRecovery` | `boolean`            | Permite que los flujos de reinstalación de plugins integrados se recuperen de fallos concretos por configuración obsoleta. |

Si `minHostVersion` está configurado, tanto la instalación como la carga del registro de manifiestos
la aplican. Los hosts antiguos omiten el plugin; las cadenas de versión no válidas se rechazan.

`allowInvalidConfigRecovery` no es un bypass general para configuraciones rotas. Es
solo para recuperación limitada de plugins integrados, de modo que la reinstalación/setup pueda reparar restos conocidos de actualizaciones, como una ruta faltante del plugin integrado o una entrada obsoleta `channels.<id>`
para ese mismo plugin. Si la configuración está rota por motivos no relacionados, la instalación
sigue fallando de forma cerrada e indica al operador que ejecute `openclaw doctor --fix`.

### Carga diferida completa

Los plugins de canal pueden optar por carga diferida con:

```json
{
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "startup": {
      "deferConfiguredChannelFullLoadUntilAfterListen": true
    }
  }
}
```

Cuando está habilitado, OpenClaw carga solo `setupEntry` durante la fase de inicio
previa a listen, incluso para canales ya configurados. La entrada completa se carga después de que el
gateway empiece a escuchar.

<Warning>
  Habilita la carga diferida solo cuando tu `setupEntry` registre todo lo que el
  gateway necesita antes de empezar a escuchar (registro del canal, rutas HTTP,
  métodos del gateway). Si la entrada completa gestiona capacidades de inicio requeridas, mantén
  el comportamiento predeterminado.
</Warning>

Si tu setup/full entry registra métodos RPC del gateway, mantenlos en un
prefijo específico del plugin. Los espacios de nombres reservados de administración del núcleo (`config.*`,
`exec.approvals.*`, `wizard.*`, `update.*`) siguen siendo propiedad del núcleo y siempre se resuelven
a `operator.admin`.

## Manifiesto del plugin

Todo plugin nativo debe incluir un `openclaw.plugin.json` en la raíz del paquete.
OpenClaw usa esto para validar la configuración sin ejecutar código del plugin.

```json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "description": "Adds My Plugin capabilities to OpenClaw",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "webhookSecret": {
        "type": "string",
        "description": "Webhook verification secret"
      }
    }
  }
}
```

Para plugins de canal, agrega `kind` y `channels`:

```json
{
  "id": "my-channel",
  "kind": "channel",
  "channels": ["my-channel"],
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

Incluso los plugins sin configuración deben incluir un esquema. Un esquema vacío es válido:

```json
{
  "id": "my-plugin",
  "configSchema": {
    "type": "object",
    "additionalProperties": false
  }
}
```

Consulta [Plugin Manifest](/plugins/manifest) para ver la referencia completa del esquema.

## Publicación en ClawHub

Para paquetes de plugins, usa el comando específico de ClawHub para paquetes:

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

El alias heredado de publicación solo para Skills es para Skills. Los paquetes de plugins
deben usar siempre `clawhub package publish`.

## Entrada de setup

El archivo `setup-entry.ts` es una alternativa ligera a `index.ts` que
OpenClaw carga cuando solo necesita superficies de setup (onboarding, reparación de configuración,
inspección de canales deshabilitados).

```typescript
// setup-entry.ts
import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
import { myChannelPlugin } from "./src/channel.js";

export default defineSetupPluginEntry(myChannelPlugin);
```

Esto evita cargar código pesado de runtime (bibliotecas de crypto, registros CLI,
servicios en segundo plano) durante flujos de setup.

**Cuándo OpenClaw usa `setupEntry` en lugar de la entrada completa:**

- El canal está deshabilitado pero necesita superficies de setup/onboarding
- El canal está habilitado pero sin configurar
- La carga diferida está habilitada (`deferConfiguredChannelFullLoadUntilAfterListen`)

**Qué debe registrar `setupEntry`:**

- El objeto del plugin de canal (mediante `defineSetupPluginEntry`)
- Cualquier ruta HTTP requerida antes del listen del gateway
- Cualquier método del gateway necesario durante el arranque

Esos métodos de gateway de arranque deben seguir evitando espacios de nombres reservados de administración del núcleo
como `config.*` o `update.*`.

**Qué NO debe incluir `setupEntry`:**

- Registros CLI
- Servicios en segundo plano
- Importaciones pesadas de runtime (crypto, SDKs)
- Métodos del gateway necesarios solo después del arranque

### Importaciones estrechas de helpers de setup

Para rutas activas solo de setup, prefiere los seams estrechos de helpers de setup en lugar del
paraguas más amplio `plugin-sdk/setup` cuando solo necesites una parte de la superficie de setup:

| Ruta de importación                 | Úsala para                                                                              | Exportaciones clave                                                                                                                                                                                                                                                                          |
| ----------------------------------- | --------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/setup-runtime`          | helpers de runtime en tiempo de setup que permanecen disponibles en `setupEntry` / inicio diferido de canales | `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
| `plugin-sdk/setup-adapter-runtime`  | adaptadores de setup de cuenta sensibles al entorno                                     | `createEnvPatchedAccountSetupAdapter`                                                                                                                                                                                                                                                        |
| `plugin-sdk/setup-tools`            | helpers de setup/install para CLI/archivo/docs                                          | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR`                                                                                                                                                                              |

Usa el seam más amplio `plugin-sdk/setup` cuando quieras toda la caja de herramientas compartida de setup,
incluidos helpers de parche de configuración como
`moveSingleAccountChannelSectionToDefaultAccount(...)`.

Los adaptadores de parche de setup siguen siendo seguros de importar en rutas activas. Su búsqueda
del surface de contrato integrado para promoción de cuenta única es diferida, de modo que importar
`plugin-sdk/setup-runtime` no carga de forma eager el descubrimiento de ese surface antes de que el adaptador se use realmente.

### Promoción de cuenta única gestionada por el canal

Cuando un canal se actualiza desde una configuración de nivel superior de cuenta única a
`channels.<id>.accounts.*`, el comportamiento compartido predeterminado es mover los valores
con alcance de cuenta promovidos a `accounts.default`.

Los canales integrados pueden restringir o sobrescribir esa promoción mediante su
surface de contrato de setup:

- `singleAccountKeysToMove`: claves adicionales de nivel superior que deben moverse a la
  cuenta promovida
- `namedAccountPromotionKeys`: cuando ya existen cuentas con nombre, solo estas
  claves se mueven a la cuenta promovida; las claves compartidas de política/entrega se mantienen en la raíz del canal
- `resolveSingleAccountPromotionTarget(...)`: elige qué cuenta existente
  recibe los valores promovidos

Matrix es el ejemplo integrado actual. Si ya existe exactamente una cuenta Matrix con nombre,
o si `defaultAccount` apunta a una clave no canónica existente como `Ops`, la promoción conserva esa cuenta en lugar de crear una nueva entrada
`accounts.default`.

## Esquema de configuración

La configuración del plugin se valida contra el JSON Schema de tu manifiesto. Las personas usuarias
configuran plugins mediante:

```json5
{
  plugins: {
    entries: {
      "my-plugin": {
        config: {
          webhookSecret: "abc123",
        },
      },
    },
  },
}
```

Tu plugin recibe esta configuración como `api.pluginConfig` durante el registro.

Para configuración específica del canal, usa en su lugar la sección de configuración del canal:

```json5
{
  channels: {
    "my-channel": {
      token: "bot-token",
      allowFrom: ["user1", "user2"],
    },
  },
}
```

### Crear esquemas de configuración de canal

Usa `buildChannelConfigSchema` desde `openclaw/plugin-sdk/core` para convertir un
esquema Zod en el contenedor `ChannelConfigSchema` que OpenClaw valida:

```typescript
import { z } from "zod";
import { buildChannelConfigSchema } from "openclaw/plugin-sdk/core";

const accountSchema = z.object({
  token: z.string().optional(),
  allowFrom: z.array(z.string()).optional(),
  accounts: z.object({}).catchall(z.any()).optional(),
  defaultAccount: z.string().optional(),
});

const configSchema = buildChannelConfigSchema(accountSchema);
```

## Asistentes de setup

Los plugins de canal pueden proporcionar asistentes interactivos de setup para `openclaw onboard`.
El asistente es un objeto `ChannelSetupWizard` dentro del `ChannelPlugin`:

```typescript
import type { ChannelSetupWizard } from "openclaw/plugin-sdk/channel-setup";

const setupWizard: ChannelSetupWizard = {
  channel: "my-channel",
  status: {
    configuredLabel: "Connected",
    unconfiguredLabel: "Not configured",
    resolveConfigured: ({ cfg }) => Boolean((cfg.channels as any)?.["my-channel"]?.token),
  },
  credentials: [
    {
      inputKey: "token",
      providerHint: "my-channel",
      credentialLabel: "Bot token",
      preferredEnvVar: "MY_CHANNEL_BOT_TOKEN",
      envPrompt: "Use MY_CHANNEL_BOT_TOKEN from environment?",
      keepPrompt: "Keep current token?",
      inputPrompt: "Enter your bot token:",
      inspect: ({ cfg, accountId }) => {
        const token = (cfg.channels as any)?.["my-channel"]?.token;
        return {
          accountConfigured: Boolean(token),
          hasConfiguredValue: Boolean(token),
        };
      },
    },
  ],
};
```

El tipo `ChannelSetupWizard` admite `credentials`, `textInputs`,
`dmPolicy`, `allowFrom`, `groupAccess`, `prepare`, `finalize` y más.
Consulta los paquetes de plugins integrados (por ejemplo el plugin de Discord en `src/channel.setup.ts`) para ver
ejemplos completos.

Para prompts de allowlist DM que solo necesitan el flujo estándar
`note -> prompt -> parse -> merge -> patch`, prefiere los helpers compartidos de setup
de `openclaw/plugin-sdk/setup`: `createPromptParsedAllowFromForAccount(...)`,
`createTopLevelChannelParsedAllowFromPrompt(...)` y
`createNestedChannelParsedAllowFromPrompt(...)`.

Para bloques de estado de setup de canal que solo varían en etiquetas, puntuaciones y líneas extra opcionales, prefiere `createStandardChannelSetupStatus(...)` de
`openclaw/plugin-sdk/setup` en lugar de recrear manualmente el mismo objeto `status` en
cada plugin.

Para superficies opcionales de setup que solo deben aparecer en ciertos contextos, usa
`createOptionalChannelSetupSurface` desde `openclaw/plugin-sdk/channel-setup`:

```typescript
import { createOptionalChannelSetupSurface } from "openclaw/plugin-sdk/channel-setup";

const setupSurface = createOptionalChannelSetupSurface({
  channel: "my-channel",
  label: "My Channel",
  npmSpec: "@myorg/openclaw-my-channel",
  docsPath: "/channels/my-channel",
});
// Returns { setupAdapter, setupWizard }
```

`plugin-sdk/channel-setup` también expone los builders de nivel inferior
`createOptionalChannelSetupAdapter(...)` y
`createOptionalChannelSetupWizard(...)` cuando solo necesitas una mitad de
esa superficie de instalación opcional.

El adaptador/asistente opcional generado falla de forma cerrada en escrituras reales de configuración. Reutiliza un único mensaje de instalación requerida en `validateInput`,
`applyAccountConfig` y `finalize`, y añade un enlace de documentación cuando `docsPath` está
configurado.

Para UI de setup respaldadas por binarios, prefiere los helpers delegados compartidos en lugar de copiar la misma lógica de binario/estado en cada canal:

- `createDetectedBinaryStatus(...)` para bloques de estado que solo varían por etiquetas,
  pistas, puntuaciones y detección de binario
- `createCliPathTextInput(...)` para entradas de texto basadas en rutas
- `createDelegatedSetupWizardStatusResolvers(...)`,
  `createDelegatedPrepare(...)`, `createDelegatedFinalize(...)` y
  `createDelegatedResolveConfigured(...)` cuando `setupEntry` necesita reenviar de forma lazy a
  un asistente completo más pesado
- `createDelegatedTextInputShouldPrompt(...)` cuando `setupEntry` solo necesita
  delegar una decisión `textInputs[*].shouldPrompt`

## Publicación e instalación

**Plugins externos:** publícalos en [ClawHub](/tools/clawhub) o npm, y luego instala con:

```bash
openclaw plugins install @myorg/openclaw-my-plugin
```

OpenClaw primero intenta ClawHub y luego usa npm automáticamente como respaldo. También puedes
forzar explícitamente ClawHub:

```bash
openclaw plugins install clawhub:@myorg/openclaw-my-plugin   # solo ClawHub
```

No hay una sobrescritura equivalente `npm:`. Usa la especificación normal del paquete npm cuando
quieras la ruta npm tras el respaldo desde ClawHub:

```bash
openclaw plugins install @myorg/openclaw-my-plugin
```

**Plugins en el repositorio:** colócalos bajo el árbol de workspace de plugins integrados y se descubrirán automáticamente
durante la compilación.

**Las personas usuarias pueden instalar con:**

```bash
openclaw plugins install <package-name>
```

<Info>
  Para instalaciones desde npm, `openclaw plugins install` ejecuta
  `npm install --ignore-scripts` (sin scripts de ciclo de vida). Mantén los árboles de dependencias del plugin
  en JS/TS puro y evita paquetes que requieran compilaciones `postinstall`.
</Info>

## Relacionado

- [SDK Entry Points](/plugins/sdk-entrypoints) -- `definePluginEntry` y `defineChannelPluginEntry`
- [Plugin Manifest](/plugins/manifest) -- referencia completa del esquema del manifiesto
- [Building Plugins](/plugins/building-plugins) -- guía paso a paso para empezar
