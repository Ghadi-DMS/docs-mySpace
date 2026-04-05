---
read_when:
    - Necesitas la firma de tipo exacta de `definePluginEntry` o `defineChannelPluginEntry`
    - Quieres entender el modo de registro (full vs setup vs metadatos de CLI)
    - Estás consultando las opciones del punto de entrada
sidebarTitle: Entry Points
summary: Referencia de `definePluginEntry`, `defineChannelPluginEntry` y `defineSetupPluginEntry`
title: Puntos de entrada de plugins
x-i18n:
    generated_at: "2026-04-05T12:49:46Z"
    model: gpt-5.4
    provider: openai
    source_hash: 799dbfe71e681dd8ba929a7a631dfe745c3c5c69530126fea2f9c137b120f51f
    source_path: plugins/sdk-entrypoints.md
    workflow: 15
---

# Puntos de entrada de plugins

Cada plugin exporta un objeto de entrada predeterminado. El SDK proporciona tres helpers para
crearlos.

<Tip>
  **¿Buscas una guía paso a paso?** Consulta [Channel Plugins](/plugins/sdk-channel-plugins)
  o [Provider Plugins](/plugins/sdk-provider-plugins).
</Tip>

## `definePluginEntry`

**Import:** `openclaw/plugin-sdk/plugin-entry`

Para plugins de proveedores, plugins de herramientas, plugins de hooks y cualquier cosa que **no** sea
un canal de mensajería.

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

export default definePluginEntry({
  id: "my-plugin",
  name: "My Plugin",
  description: "Short summary",
  register(api) {
    api.registerProvider({
      /* ... */
    });
    api.registerTool({
      /* ... */
    });
  },
});
```

| Field          | Type                                                             | Required | Predeterminado      |
| -------------- | ---------------------------------------------------------------- | -------- | ------------------- |
| `id`           | `string`                                                         | Sí       | —                   |
| `name`         | `string`                                                         | Sí       | —                   |
| `description`  | `string`                                                         | Sí       | —                   |
| `kind`         | `string`                                                         | No       | —                   |
| `configSchema` | `OpenClawPluginConfigSchema \| () => OpenClawPluginConfigSchema` | No       | Esquema de objeto vacío |
| `register`     | `(api: OpenClawPluginApi) => void`                               | Sí       | —                   |

- `id` debe coincidir con tu manifiesto `openclaw.plugin.json`.
- `kind` es para ranuras exclusivas: `"memory"` o `"context-engine"`.
- `configSchema` puede ser una función para evaluación diferida.
- OpenClaw resuelve y memoiza ese esquema en el primer acceso, de modo que los generadores de esquemas costosos
  se ejecutan solo una vez.

## `defineChannelPluginEntry`

**Import:** `openclaw/plugin-sdk/channel-core`

Envuelve `definePluginEntry` con cableado específico de canal. Llama automáticamente a
`api.registerChannel({ plugin })`, expone una unión opcional para metadatos CLI de ayuda raíz y controla `registerFull` según el modo de registro.

```typescript
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";

export default defineChannelPluginEntry({
  id: "my-channel",
  name: "My Channel",
  description: "Short summary",
  plugin: myChannelPlugin,
  setRuntime: setMyRuntime,
  registerCliMetadata(api) {
    api.registerCli(/* ... */);
  },
  registerFull(api) {
    api.registerGatewayMethod(/* ... */);
  },
});
```

| Field                 | Type                                                             | Required | Predeterminado      |
| --------------------- | ---------------------------------------------------------------- | -------- | ------------------- |
| `id`                  | `string`                                                         | Sí       | —                   |
| `name`                | `string`                                                         | Sí       | —                   |
| `description`         | `string`                                                         | Sí       | —                   |
| `plugin`              | `ChannelPlugin`                                                  | Sí       | —                   |
| `configSchema`        | `OpenClawPluginConfigSchema \| () => OpenClawPluginConfigSchema` | No       | Esquema de objeto vacío |
| `setRuntime`          | `(runtime: PluginRuntime) => void`                               | No       | —                   |
| `registerCliMetadata` | `(api: OpenClawPluginApi) => void`                               | No       | —                   |
| `registerFull`        | `(api: OpenClawPluginApi) => void`                               | No       | —                   |

- `setRuntime` se llama durante el registro para que puedas almacenar la referencia del runtime
  (normalmente mediante `createPluginRuntimeStore`). Se omite durante la captura de
  metadatos CLI.
- `registerCliMetadata` se ejecuta tanto cuando `api.registrationMode === "cli-metadata"`
  como cuando `api.registrationMode === "full"`.
  Úsalo como lugar canónico para los descriptores CLI gestionados por el canal, para que la ayuda raíz
  siga sin activación mientras que el registro normal de comandos CLI siga siendo compatible
  con cargas completas del plugin.
- `registerFull` solo se ejecuta cuando `api.registrationMode === "full"`. Se omite
  durante la carga solo de setup.
- Igual que `definePluginEntry`, `configSchema` puede ser una factoría diferida y OpenClaw
  memoiza el esquema resuelto en el primer acceso.
- Para comandos CLI raíz gestionados por plugins, prefiere `api.registerCli(..., { descriptors: [...] })`
  cuando quieras que el comando permanezca con lazy-loading sin desaparecer del
  árbol de análisis CLI raíz. Para plugins de canal, registra preferiblemente esos descriptores
  desde `registerCliMetadata(...)` y mantén `registerFull(...)` centrado en trabajo solo de runtime.
- Si `registerFull(...)` también registra métodos RPC del gateway, mantenlos en un
  prefijo específico del plugin. Los espacios de nombres reservados de administración del núcleo (`config.*`,
  `exec.approvals.*`, `wizard.*`, `update.*`) siempre se fuerzan a
  `operator.admin`.

## `defineSetupPluginEntry`

**Import:** `openclaw/plugin-sdk/channel-core`

Para el archivo ligero `setup-entry.ts`. Devuelve solo `{ plugin }` sin
cableado de runtime ni CLI.

```typescript
import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";

export default defineSetupPluginEntry(myChannelPlugin);
```

OpenClaw carga esto en lugar de la entrada completa cuando un canal está deshabilitado,
sin configurar o cuando la carga diferida está habilitada. Consulta
[Setup and Config](/plugins/sdk-setup#setup-entry) para saber cuándo importa esto.

En la práctica, combina `defineSetupPluginEntry(...)` con las familias estrechas de helpers de setup:

- `openclaw/plugin-sdk/setup-runtime` para helpers de setup seguros en runtime, como
  adaptadores de parche de setup seguros para importación, salida de notas de búsqueda,
  `promptResolvedAllowFrom`, `splitSetupEntries` y proxies delegados de setup
- `openclaw/plugin-sdk/channel-setup` para superficies de setup de instalación opcional
- `openclaw/plugin-sdk/setup-tools` para helpers de CLI de setup/instalación/archivo/docs

Mantén los SDK pesados, el registro CLI y los servicios de runtime de larga duración en la entrada completa.

## Modo de registro

`api.registrationMode` le indica a tu plugin cómo fue cargado:

| Mode              | Cuándo                             | Qué registrar                                                                             |
| ----------------- | ---------------------------------- | ----------------------------------------------------------------------------------------- |
| `"full"`          | Inicio normal del gateway          | Todo                                                                                      |
| `"setup-only"`    | Canal deshabilitado/sin configurar | Solo registro del canal                                                                   |
| `"setup-runtime"` | Flujo de setup con runtime disponible | Registro del canal más solo el runtime ligero necesario antes de que cargue la entrada completa |
| `"cli-metadata"`  | Ayuda raíz / captura de metadatos CLI | Solo descriptores CLI                                                                    |

`defineChannelPluginEntry` gestiona esta división automáticamente. Si usas
`definePluginEntry` directamente para un canal, comprueba tú mismo el modo:

```typescript
register(api) {
  if (api.registrationMode === "cli-metadata" || api.registrationMode === "full") {
    api.registerCli(/* ... */);
    if (api.registrationMode === "cli-metadata") return;
  }

  api.registerChannel({ plugin: myPlugin });
  if (api.registrationMode !== "full") return;

  // Registros pesados solo de runtime
  api.registerService(/* ... */);
}
```

Trata `"setup-runtime"` como la ventana en la que deben existir superficies de inicio solo de setup
sin volver a entrar en el runtime completo del canal integrado. Encajan bien ahí
el registro del canal, rutas HTTP seguras para setup, métodos de gateway seguros para setup y
helpers delegados de setup. Los servicios pesados en segundo plano, registradores CLI y
arranques de SDK de proveedor/cliente siguen perteneciendo a `"full"`.

Para registradores CLI específicamente:

- usa `descriptors` cuando el registrador gestiona uno o más comandos raíz y quieras
  que OpenClaw cargue con lazy-loading el módulo CLI real en la primera invocación
- asegúrate de que esos descriptores cubran cada raíz de comando de nivel superior expuesta por el
  registrador
- usa solo `commands` para rutas de compatibilidad eager

## Formas de plugins

OpenClaw clasifica los plugins cargados según su comportamiento de registro:

| Forma                 | Descripción                                       |
| --------------------- | ------------------------------------------------- |
| **plain-capability**  | Un tipo de capacidad (por ejemplo, solo proveedor) |
| **hybrid-capability** | Varios tipos de capacidad (por ejemplo, proveedor + voz) |
| **hook-only**         | Solo hooks, sin capacidades                       |
| **non-capability**    | Herramientas/comandos/servicios pero sin capacidades |

Usa `openclaw plugins inspect <id>` para ver la forma de un plugin.

## Relacionado

- [SDK Overview](/plugins/sdk-overview) — API de registro y referencia de subrutas
- [Runtime Helpers](/plugins/sdk-runtime) — `api.runtime` y `createPluginRuntimeStore`
- [Setup and Config](/plugins/sdk-setup) — manifiesto, entrada de setup, carga diferida
- [Channel Plugins](/plugins/sdk-channel-plugins) — crear el objeto `ChannelPlugin`
- [Provider Plugins](/plugins/sdk-provider-plugins) — registro de proveedores y hooks
