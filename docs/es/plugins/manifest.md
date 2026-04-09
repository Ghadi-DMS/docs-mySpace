---
read_when:
    - Estás creando un plugin de OpenClaw
    - Necesitas distribuir un esquema de configuración de plugin o depurar errores de validación de plugins
summary: Manifiesto del plugin + requisitos del esquema JSON (validación estricta de configuración)
title: Manifiesto del plugin
x-i18n:
    generated_at: "2026-04-09T01:28:54Z"
    model: gpt-5.4
    provider: openai
    source_hash: 9a7ee4b621a801d2a8f32f8976b0e1d9433c7810eb360aca466031fc0ffb286a
    source_path: plugins/manifest.md
    workflow: 15
---

# Manifiesto del plugin (openclaw.plugin.json)

Esta página es solo para el **manifiesto nativo de plugins de OpenClaw**.

Para diseños de paquetes compatibles, consulta [Paquetes de plugins](/es/plugins/bundles).

Los formatos de paquetes compatibles usan archivos de manifiesto diferentes:

- Paquete Codex: `.codex-plugin/plugin.json`
- Paquete Claude: `.claude-plugin/plugin.json` o el diseño predeterminado de componentes de Claude
  sin manifiesto
- Paquete Cursor: `.cursor-plugin/plugin.json`

OpenClaw también detecta automáticamente esos diseños de paquetes, pero no se validan
con el esquema `openclaw.plugin.json` descrito aquí.

Para los paquetes compatibles, OpenClaw actualmente lee los metadatos del paquete más las
raíces de Skills declaradas, las raíces de comandos de Claude, los valores predeterminados de `settings.json` del paquete Claude,
los valores predeterminados de LSP del paquete Claude y los paquetes de hooks compatibles cuando el diseño coincide con
las expectativas del entorno de ejecución de OpenClaw.

Todo plugin nativo de OpenClaw **debe** incluir un archivo `openclaw.plugin.json` en la
**raíz del plugin**. OpenClaw usa este manifiesto para validar la configuración
**sin ejecutar el código del plugin**. Los manifiestos faltantes o no válidos se tratan como
errores del plugin y bloquean la validación de configuración.

Consulta la guía completa del sistema de plugins: [Plugins](/es/tools/plugin).
Para el modelo nativo de capacidades y la guía actual de compatibilidad externa:
[Modelo de capacidades](/es/plugins/architecture#public-capability-model).

## Qué hace este archivo

`openclaw.plugin.json` son los metadatos que OpenClaw lee antes de cargar el
código de tu plugin.

Úsalo para:

- identidad del plugin
- validación de configuración
- metadatos de autenticación e incorporación que deben estar disponibles sin iniciar el
  entorno de ejecución del plugin
- metadatos de alias y autoactivación que deben resolverse antes de que se cargue el entorno de ejecución del plugin
- metadatos abreviados de pertenencia de familias de modelos que deben activar automáticamente el
  plugin antes de que se cargue el entorno de ejecución
- instantáneas estáticas de pertenencia de capacidades usadas para el cableado de compatibilidad incluido y
  la cobertura de contratos
- metadatos de configuración específicos del canal que deben fusionarse en el catálogo y las superficies de validación
  sin cargar el entorno de ejecución
- sugerencias de UI de configuración

No lo uses para:

- registrar comportamiento en tiempo de ejecución
- declarar puntos de entrada de código
- metadatos de instalación de npm

Eso pertenece al código de tu plugin y a `package.json`.

## Ejemplo mínimo

```json
{
  "id": "voice-call",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

## Ejemplo completo

```json
{
  "id": "openrouter",
  "name": "OpenRouter",
  "description": "Plugin de proveedor OpenRouter",
  "version": "1.0.0",
  "providers": ["openrouter"],
  "modelSupport": {
    "modelPrefixes": ["router-"]
  },
  "cliBackends": ["openrouter-cli"],
  "providerAuthEnvVars": {
    "openrouter": ["OPENROUTER_API_KEY"]
  },
  "providerAuthAliases": {
    "openrouter-coding": "openrouter"
  },
  "channelEnvVars": {
    "openrouter-chatops": ["OPENROUTER_CHATOPS_TOKEN"]
  },
  "providerAuthChoices": [
    {
      "provider": "openrouter",
      "method": "api-key",
      "choiceId": "openrouter-api-key",
      "choiceLabel": "Clave API de OpenRouter",
      "groupId": "openrouter",
      "groupLabel": "OpenRouter",
      "optionKey": "openrouterApiKey",
      "cliFlag": "--openrouter-api-key",
      "cliOption": "--openrouter-api-key <key>",
      "cliDescription": "Clave API de OpenRouter",
      "onboardingScopes": ["text-inference"]
    }
  ],
  "uiHints": {
    "apiKey": {
      "label": "Clave API",
      "placeholder": "sk-or-v1-...",
      "sensitive": true
    }
  },
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "apiKey": {
        "type": "string"
      }
    }
  }
}
```

## Referencia de campos de nivel superior

| Field                               | Required | Type                             | What it means                                                                                                                                                                                                |
| ----------------------------------- | -------- | -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `id`                                | Sí       | `string`                         | Id canónico del plugin. Este es el id usado en `plugins.entries.<id>`.                                                                                                                                        |
| `configSchema`                      | Sí       | `object`                         | JSON Schema en línea para la configuración de este plugin.                                                                                                                                                    |
| `enabledByDefault`                  | No       | `true`                           | Marca un plugin incluido como habilitado de forma predeterminada. Omítelo, o establece cualquier valor distinto de `true`, para dejar el plugin deshabilitado de forma predeterminada.                       |
| `legacyPluginIds`                   | No       | `string[]`                       | Id heredados que se normalizan a este id canónico de plugin.                                                                                                                                                  |
| `autoEnableWhenConfiguredProviders` | No       | `string[]`                       | Id de proveedores que deben autohabilitar este plugin cuando la autenticación, la configuración o las referencias de modelos los mencionan.                                                                  |
| `kind`                              | No       | `"memory"` \| `"context-engine"` | Declara un tipo exclusivo de plugin usado por `plugins.slots.*`.                                                                                                                                              |
| `channels`                          | No       | `string[]`                       | Id de canales propiedad de este plugin. Se usan para descubrimiento y validación de configuración.                                                                                                            |
| `providers`                         | No       | `string[]`                       | Id de proveedores propiedad de este plugin.                                                                                                                                                                   |
| `modelSupport`                      | No       | `object`                         | Metadatos abreviados de familias de modelos propiedad del manifiesto usados para cargar automáticamente el plugin antes del entorno de ejecución.                                                             |
| `cliBackends`                       | No       | `string[]`                       | Id de backends de inferencia por CLI propiedad de este plugin. Se usan para la autoactivación al inicio a partir de referencias explícitas en la configuración.                                             |
| `providerAuthEnvVars`               | No       | `Record<string, string[]>`       | Metadatos ligeros de autenticación del proveedor mediante variables de entorno que OpenClaw puede inspeccionar sin cargar el código del plugin.                                                              |
| `providerAuthAliases`               | No       | `Record<string, string>`         | Id de proveedores que deben reutilizar otro id de proveedor para la búsqueda de autenticación, por ejemplo un proveedor de programación que comparte la clave API y los perfiles de autenticación del proveedor base. |
| `channelEnvVars`                    | No       | `Record<string, string[]>`       | Metadatos ligeros de variables de entorno del canal que OpenClaw puede inspeccionar sin cargar el código del plugin. Usa esto para configuraciones de canal guiadas por entorno o superficies de autenticación que los ayudantes genéricos de inicio/configuración deberían ver. |
| `providerAuthChoices`               | No       | `object[]`                       | Metadatos ligeros de opciones de autenticación para selectores de incorporación, resolución de proveedores preferidos y cableado sencillo de banderas CLI.                                                   |
| `contracts`                         | No       | `object`                         | Instantánea estática de capacidades incluidas para voz, transcripción en tiempo real, voz en tiempo real, comprensión de medios, generación de imágenes, generación musical, generación de video, web-fetch, búsqueda web y propiedad de herramientas. |
| `channelConfigs`                    | No       | `Record<string, object>`         | Metadatos de configuración de canal propiedad del manifiesto fusionados en las superficies de descubrimiento y validación antes de que se cargue el entorno de ejecución.                                   |
| `skills`                            | No       | `string[]`                       | Directorios de Skills que se cargarán, relativos a la raíz del plugin.                                                                                                                                        |
| `name`                              | No       | `string`                         | Nombre legible del plugin.                                                                                                                                                                                    |
| `description`                       | No       | `string`                         | Resumen corto mostrado en superficies del plugin.                                                                                                                                                             |
| `version`                           | No       | `string`                         | Versión informativa del plugin.                                                                                                                                                                               |
| `uiHints`                           | No       | `Record<string, object>`         | Etiquetas de UI, marcadores de posición y sugerencias de sensibilidad para campos de configuración.                                                                                                           |

## Referencia de `providerAuthChoices`

Cada entrada de `providerAuthChoices` describe una opción de incorporación o autenticación.
OpenClaw la lee antes de que se cargue el entorno de ejecución del proveedor.

| Field                 | Required | Type                                            | What it means                                                                                            |
| --------------------- | -------- | ----------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `provider`            | Sí       | `string`                                        | Id del proveedor al que pertenece esta opción.                                                           |
| `method`              | Sí       | `string`                                        | Id del método de autenticación al que se enviará.                                                        |
| `choiceId`            | Sí       | `string`                                        | Id estable de opción de autenticación usado por los flujos de incorporación y CLI.                       |
| `choiceLabel`         | No       | `string`                                        | Etiqueta visible para el usuario. Si se omite, OpenClaw usa `choiceId` como valor alternativo.          |
| `choiceHint`          | No       | `string`                                        | Texto de ayuda corto para el selector.                                                                   |
| `assistantPriority`   | No       | `number`                                        | Los valores más bajos se ordenan antes en los selectores interactivos controlados por el asistente.     |
| `assistantVisibility` | No       | `"visible"` \| `"manual-only"`                  | Oculta la opción de los selectores del asistente, pero sigue permitiendo la selección manual por CLI.    |
| `deprecatedChoiceIds` | No       | `string[]`                                      | Id heredados de opciones que deben redirigir a los usuarios a esta opción de reemplazo.                 |
| `groupId`             | No       | `string`                                        | Id de grupo opcional para agrupar opciones relacionadas.                                                 |
| `groupLabel`          | No       | `string`                                        | Etiqueta visible para el usuario de ese grupo.                                                           |
| `groupHint`           | No       | `string`                                        | Texto de ayuda corto para el grupo.                                                                      |
| `optionKey`           | No       | `string`                                        | Clave interna de opción para flujos de autenticación simples de una sola bandera.                        |
| `cliFlag`             | No       | `string`                                        | Nombre de la bandera CLI, como `--openrouter-api-key`.                                                   |
| `cliOption`           | No       | `string`                                        | Forma completa de la opción CLI, como `--openrouter-api-key <key>`.                                      |
| `cliDescription`      | No       | `string`                                        | Descripción usada en la ayuda de la CLI.                                                                 |
| `onboardingScopes`    | No       | `Array<"text-inference" \| "image-generation">` | En qué superficies de incorporación debe aparecer esta opción. Si se omite, el valor predeterminado es `["text-inference"]`. |

## Referencia de `uiHints`

`uiHints` es un mapa de nombres de campos de configuración a pequeñas sugerencias de representación.

```json
{
  "uiHints": {
    "apiKey": {
      "label": "Clave API",
      "help": "Se usa para solicitudes a OpenRouter",
      "placeholder": "sk-or-v1-...",
      "sensitive": true
    }
  }
}
```

Cada sugerencia de campo puede incluir:

| Field         | Type       | What it means                           |
| ------------- | ---------- | --------------------------------------- |
| `label`       | `string`   | Etiqueta del campo visible para el usuario. |
| `help`        | `string`   | Texto de ayuda corto.                   |
| `tags`        | `string[]` | Etiquetas opcionales de UI.             |
| `advanced`    | `boolean`  | Marca el campo como avanzado.           |
| `sensitive`   | `boolean`  | Marca el campo como secreto o sensible. |
| `placeholder` | `string`   | Texto de marcador de posición para entradas de formulario. |

## Referencia de `contracts`

Usa `contracts` solo para metadatos estáticos de propiedad de capacidades que OpenClaw puede
leer sin importar el entorno de ejecución del plugin.

```json
{
  "contracts": {
    "speechProviders": ["openai"],
    "realtimeTranscriptionProviders": ["openai"],
    "realtimeVoiceProviders": ["openai"],
    "mediaUnderstandingProviders": ["openai", "openai-codex"],
    "imageGenerationProviders": ["openai"],
    "videoGenerationProviders": ["qwen"],
    "webFetchProviders": ["firecrawl"],
    "webSearchProviders": ["gemini"],
    "tools": ["firecrawl_search", "firecrawl_scrape"]
  }
}
```

Cada lista es opcional:

| Field                            | Type       | What it means                                                  |
| -------------------------------- | ---------- | -------------------------------------------------------------- |
| `speechProviders`                | `string[]` | Id de proveedores de voz propiedad de este plugin.             |
| `realtimeTranscriptionProviders` | `string[]` | Id de proveedores de transcripción en tiempo real propiedad de este plugin. |
| `realtimeVoiceProviders`         | `string[]` | Id de proveedores de voz en tiempo real propiedad de este plugin. |
| `mediaUnderstandingProviders`    | `string[]` | Id de proveedores de comprensión de medios propiedad de este plugin. |
| `imageGenerationProviders`       | `string[]` | Id de proveedores de generación de imágenes propiedad de este plugin. |
| `videoGenerationProviders`       | `string[]` | Id de proveedores de generación de video propiedad de este plugin. |
| `webFetchProviders`              | `string[]` | Id de proveedores de web-fetch propiedad de este plugin.       |
| `webSearchProviders`             | `string[]` | Id de proveedores de búsqueda web propiedad de este plugin.    |
| `tools`                          | `string[]` | Nombres de herramientas del agente propiedad de este plugin para verificaciones de contratos incluidos. |

## Referencia de `channelConfigs`

Usa `channelConfigs` cuando un plugin de canal necesita metadatos ligeros de configuración antes de que
se cargue el entorno de ejecución.

```json
{
  "channelConfigs": {
    "matrix": {
      "schema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "homeserverUrl": { "type": "string" }
        }
      },
      "uiHints": {
        "homeserverUrl": {
          "label": "URL del homeserver",
          "placeholder": "https://matrix.example.com"
        }
      },
      "label": "Matrix",
      "description": "Conexión al homeserver de Matrix",
      "preferOver": ["matrix-legacy"]
    }
  }
}
```

Cada entrada de canal puede incluir:

| Field         | Type                     | What it means                                                                             |
| ------------- | ------------------------ | ----------------------------------------------------------------------------------------- |
| `schema`      | `object`                 | JSON Schema para `channels.<id>`. Obligatorio para cada entrada declarada de configuración de canal. |
| `uiHints`     | `Record<string, object>` | Etiquetas/placeholders/sugerencias de sensibilidad opcionales de UI para esa sección de configuración del canal. |
| `label`       | `string`                 | Etiqueta del canal fusionada en las superficies de selector e inspección cuando los metadatos del entorno de ejecución no están listos. |
| `description` | `string`                 | Descripción corta del canal para las superficies de inspección y catálogo.                |
| `preferOver`  | `string[]`               | Id de plugins heredados o de menor prioridad que este canal debe superar en las superficies de selección. |

## Referencia de `modelSupport`

Usa `modelSupport` cuando OpenClaw deba inferir tu plugin de proveedor a partir de
id abreviados de modelos como `gpt-5.4` o `claude-sonnet-4.6` antes de que se cargue el entorno de ejecución del plugin.

```json
{
  "modelSupport": {
    "modelPrefixes": ["gpt-", "o1", "o3", "o4"],
    "modelPatterns": ["^computer-use-preview"]
  }
}
```

OpenClaw aplica esta precedencia:

- las referencias explícitas `provider/model` usan los metadatos del manifiesto `providers` propietario
- `modelPatterns` tienen prioridad sobre `modelPrefixes`
- si coinciden un plugin no incluido y un plugin incluido, el plugin no incluido
  gana
- la ambigüedad restante se ignora hasta que el usuario o la configuración especifique un proveedor

Campos:

| Field           | Type       | What it means                                                                   |
| --------------- | ---------- | ------------------------------------------------------------------------------- |
| `modelPrefixes` | `string[]` | Prefijos comparados con `startsWith` frente a id abreviados de modelos.         |
| `modelPatterns` | `string[]` | Fuentes regex comparadas con id abreviados de modelos tras eliminar el sufijo del perfil. |

Las claves heredadas de capacidades de nivel superior están obsoletas. Usa `openclaw doctor --fix` para
mover `speechProviders`, `realtimeTranscriptionProviders`,
`realtimeVoiceProviders`, `mediaUnderstandingProviders`,
`imageGenerationProviders`, `videoGenerationProviders`,
`webFetchProviders` y `webSearchProviders` bajo `contracts`; la carga normal
del manifiesto ya no trata esos campos de nivel superior como
propiedad de capacidades.

## Manifiesto frente a package.json

Los dos archivos cumplen funciones distintas:

| File                   | Use it for                                                                                                                       |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.plugin.json` | Descubrimiento, validación de configuración, metadatos de opciones de autenticación y sugerencias de UI que deben existir antes de que se ejecute el código del plugin |
| `package.json`         | Metadatos de npm, instalación de dependencias y el bloque `openclaw` usado para puntos de entrada, control de instalación, configuración o metadatos de catálogo |

Si no estás seguro de dónde debe ir un dato, usa esta regla:

- si OpenClaw debe conocerlo antes de cargar el código del plugin, ponlo en `openclaw.plugin.json`
- si trata sobre empaquetado, archivos de entrada o comportamiento de instalación de npm, ponlo en `package.json`

### Campos de package.json que afectan al descubrimiento

Algunos metadatos del plugin previos al entorno de ejecución viven intencionalmente en `package.json` bajo el
bloque `openclaw` en lugar de `openclaw.plugin.json`.

Ejemplos importantes:

| Field                                                             | What it means                                                                                                                                |
| ----------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.extensions`                                             | Declara puntos de entrada nativos del plugin.                                                                                                |
| `openclaw.setupEntry`                                             | Punto de entrada ligero solo para configuración usado durante la incorporación y el arranque diferido del canal.                            |
| `openclaw.channel`                                                | Metadatos ligeros del catálogo de canales, como etiquetas, rutas de documentación, alias y texto de selección.                             |
| `openclaw.channel.configuredState`                                | Metadatos ligeros del verificador de estado configurado que pueden responder “¿ya existe una configuración solo por entorno?” sin cargar el entorno de ejecución completo del canal. |
| `openclaw.channel.persistedAuthState`                             | Metadatos ligeros del verificador de autenticación persistida que pueden responder “¿ya hay algo con sesión iniciada?” sin cargar el entorno de ejecución completo del canal. |
| `openclaw.install.npmSpec` / `openclaw.install.localPath`         | Sugerencias de instalación/actualización para plugins incluidos y publicados externamente.                                                  |
| `openclaw.install.defaultChoice`                                  | Ruta de instalación preferida cuando hay varias fuentes de instalación disponibles.                                                         |
| `openclaw.install.minHostVersion`                                 | Versión mínima compatible del host de OpenClaw, usando un umbral semver como `>=2026.3.22`.                                                 |
| `openclaw.install.allowInvalidConfigRecovery`                     | Permite una ruta limitada de recuperación por reinstalación de plugins incluidos cuando la configuración no es válida.                      |
| `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen` | Permite que las superficies de canal solo de configuración se carguen antes que el plugin de canal completo durante el arranque.           |

`openclaw.install.minHostVersion` se aplica durante la instalación y la carga del
registro de manifiestos. Los valores no válidos se rechazan; los valores válidos pero más recientes omiten el
plugin en hosts más antiguos.

`openclaw.install.allowInvalidConfigRecovery` es intencionalmente limitado. No
permite instalar configuraciones arbitrariamente rotas. Actualmente solo permite que los flujos de instalación
se recuperen de errores concretos de actualización obsoleta de plugins incluidos, como una
ruta faltante del plugin incluido o una entrada obsoleta `channels.<id>` para ese mismo
plugin incluido. Los errores de configuración no relacionados siguen bloqueando la instalación y envían a los operadores
a `openclaw doctor --fix`.

`openclaw.channel.persistedAuthState` son metadatos de paquete para un módulo
verificador diminuto:

```json
{
  "openclaw": {
    "channel": {
      "id": "whatsapp",
      "persistedAuthState": {
        "specifier": "./auth-presence",
        "exportName": "hasAnyWhatsAppAuth"
      }
    }
  }
}
```

Úsalo cuando los flujos de configuración, doctor o estado configurado necesiten una sonda barata de autenticación sí/no
antes de que se cargue el plugin completo del canal. La exportación de destino debe ser una función pequeña
que lea solo el estado persistido; no la enrutes a través del barrel completo del entorno de ejecución
del canal.

`openclaw.channel.configuredState` sigue la misma forma para verificaciones ligeras
de estado configurado solo por entorno:

```json
{
  "openclaw": {
    "channel": {
      "id": "telegram",
      "configuredState": {
        "specifier": "./configured-state",
        "exportName": "hasTelegramConfiguredState"
      }
    }
  }
}
```

Úsalo cuando un canal pueda responder el estado configurado desde el entorno u otras
entradas mínimas ajenas al entorno de ejecución. Si la verificación necesita la resolución completa de configuración o el
entorno de ejecución real del canal, deja esa lógica en el hook del plugin `config.hasConfiguredState`.

## Requisitos de JSON Schema

- **Todo plugin debe incluir un JSON Schema**, incluso si no acepta configuración.
- Se acepta un esquema vacío (por ejemplo, `{ "type": "object", "additionalProperties": false }`).
- Los esquemas se validan en el momento de lectura/escritura de la configuración, no en tiempo de ejecución.

## Comportamiento de validación

- Las claves desconocidas `channels.*` son **errores**, a menos que el id del canal esté declarado por
  un manifiesto de plugin.
- `plugins.entries.<id>`, `plugins.allow`, `plugins.deny` y `plugins.slots.*`
  deben hacer referencia a id de plugins **detectables**. Los id desconocidos son **errores**.
- Si un plugin está instalado pero tiene un manifiesto o esquema roto o faltante,
  la validación falla y Doctor informa del error del plugin.
- Si existe configuración del plugin pero el plugin está **deshabilitado**, la configuración se conserva y
  se muestra una **advertencia** en Doctor + registros.

Consulta [Referencia de configuración](/es/gateway/configuration) para ver el esquema completo de `plugins.*`.

## Notas

- El manifiesto es **obligatorio para los plugins nativos de OpenClaw**, incluidas las cargas desde el sistema de archivos local.
- El entorno de ejecución sigue cargando el módulo del plugin por separado; el manifiesto es solo para
  descubrimiento + validación.
- Los manifiestos nativos se analizan con JSON5, por lo que se aceptan comentarios, comas finales y
  claves sin comillas siempre que el valor final siga siendo un objeto.
- El cargador de manifiestos solo lee los campos documentados del manifiesto. Evita agregar
  aquí claves personalizadas de nivel superior.
- `providerAuthEnvVars` es la ruta de metadatos ligera para sondeos de autenticación, validación
  de marcadores de entorno y superficies similares de autenticación del proveedor que no deberían iniciar el entorno de ejecución del plugin
  solo para inspeccionar nombres de entorno.
- `providerAuthAliases` permite que variantes de proveedores reutilicen la autenticación
  por variables de entorno, perfiles de autenticación, autenticación respaldada por configuración y la opción de incorporación
  por clave API de otro proveedor sin codificar esa relación en el núcleo.
- `channelEnvVars` es la ruta de metadatos ligera para fallback de entorno de shell, prompts
  de configuración y superficies de canal similares que no deberían iniciar el entorno de ejecución del plugin
  solo para inspeccionar nombres de entorno.
- `providerAuthChoices` es la ruta de metadatos ligera para selectores de opciones de autenticación,
  resolución de `--auth-choice`, mapeo de proveedores preferidos y registro simple
  de banderas CLI de incorporación antes de que se cargue el entorno de ejecución del proveedor. Para metadatos de asistente en tiempo de ejecución
  que requieren código del proveedor, consulta
  [Hooks de runtime del proveedor](/es/plugins/architecture#provider-runtime-hooks).
- Los tipos exclusivos de plugin se seleccionan mediante `plugins.slots.*`.
  - `kind: "memory"` se selecciona mediante `plugins.slots.memory`.
  - `kind: "context-engine"` se selecciona mediante `plugins.slots.contextEngine`
    (predeterminado: `legacy` integrado).
- `channels`, `providers`, `cliBackends` y `skills` pueden omitirse cuando un
  plugin no los necesita.
- Si tu plugin depende de módulos nativos, documenta los pasos de compilación y cualquier
  requisito de lista de permitidos del administrador de paquetes (por ejemplo, pnpm `allow-build-scripts`
  - `pnpm rebuild <package>`).

## Relacionado

- [Creación de plugins](/es/plugins/building-plugins) — introducción a los plugins
- [Arquitectura de plugins](/es/plugins/architecture) — arquitectura interna
- [Resumen del SDK](/es/plugins/sdk-overview) — referencia del SDK de plugins
