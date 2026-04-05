---
read_when:
    - Estás creando un plugin de OpenClaw
    - Necesitas incluir un esquema de configuración del plugin o depurar errores de validación de plugins
summary: Requisitos del manifiesto de plugin + JSON Schema (validación estricta de configuración)
title: Manifiesto del plugin
x-i18n:
    generated_at: "2026-04-05T12:50:05Z"
    model: gpt-5.4
    provider: openai
    source_hash: 702447ad39f295cfffd4214c3e389bee667d2f9850754f2e02e325dde8e4ac00
    source_path: plugins/manifest.md
    workflow: 15
---

# Manifiesto del plugin (`openclaw.plugin.json`)

Esta página es solo para el **manifiesto nativo de plugins de OpenClaw**.

Para diseños de paquetes compatibles, consulta [Paquetes de plugins](/plugins/bundles).

Los formatos de paquetes compatibles usan archivos de manifiesto distintos:

- Paquete Codex: `.codex-plugin/plugin.json`
- Paquete Claude: `.claude-plugin/plugin.json` o el diseño predeterminado de componentes de Claude
  sin manifiesto
- Paquete Cursor: `.cursor-plugin/plugin.json`

OpenClaw también detecta automáticamente esos diseños de paquetes, pero no se validan
contra el esquema `openclaw.plugin.json` descrito aquí.

Para paquetes compatibles, OpenClaw actualmente lee los metadatos del paquete más las
raíces de Skills declaradas, raíces de comandos de Claude, valores predeterminados de Claude `settings.json`,
valores predeterminados de LSP de Claude y paquetes de hooks compatibles cuando el diseño coincide
con las expectativas de runtime de OpenClaw.

Todo plugin nativo de OpenClaw **debe** incluir un archivo `openclaw.plugin.json` en la
**raíz del plugin**. OpenClaw usa este manifiesto para validar la configuración
**sin ejecutar código del plugin**. Los manifiestos ausentes o no válidos se tratan como
errores del plugin y bloquean la validación de configuración.

Consulta la guía completa del sistema de plugins: [Plugins](/tools/plugin).
Para el modelo nativo de capacidades y la guía actual de compatibilidad externa:
[Modelo de capacidades](/plugins/architecture#public-capability-model).

## Qué hace este archivo

`openclaw.plugin.json` son los metadatos que OpenClaw lee antes de cargar el
código de tu plugin.

Úsalo para:

- identidad del plugin
- validación de configuración
- metadatos de autenticación y onboarding que deban estar disponibles sin iniciar el
  runtime del plugin
- metadatos de alias y autoactivación que deban resolverse antes de cargar el runtime del plugin
- metadatos abreviados de propiedad de familias de modelos que deban activar
  automáticamente el plugin antes de cargar el runtime
- instantáneas estáticas de propiedad de capacidades usadas para el cableado de compatibilidad empaquetada y la cobertura de contratos
- metadatos de configuración específicos de canal que deban fusionarse en superficies de catálogo y validación sin cargar el runtime
- sugerencias de interfaz de configuración

No lo uses para:

- registrar comportamiento en runtime
- declarar puntos de entrada de código
- metadatos de instalación npm

Eso pertenece a tu código del plugin y a `package.json`.

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
  "description": "Plugin proveedor de OpenRouter",
  "version": "1.0.0",
  "providers": ["openrouter"],
  "modelSupport": {
    "modelPrefixes": ["router-"]
  },
  "cliBackends": ["openrouter-cli"],
  "providerAuthEnvVars": {
    "openrouter": ["OPENROUTER_API_KEY"]
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

| Campo                               | Obligatorio | Tipo                             | Qué significa                                                                                                                                                                              |
| ----------------------------------- | ----------- | -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `id`                                | Sí          | `string`                         | Id canónico del plugin. Es el id usado en `plugins.entries.<id>`.                                                                                                                         |
| `configSchema`                      | Sí          | `object`                         | JSON Schema inline para la configuración de este plugin.                                                                                                                                   |
| `enabledByDefault`                  | No          | `true`                           | Marca un plugin empaquetado como habilitado de forma predeterminada. Omítelo, o establece cualquier valor distinto de `true`, para dejar el plugin deshabilitado de forma predeterminada. |
| `legacyPluginIds`                   | No          | `string[]`                       | Ids heredados que se normalizan a este id canónico de plugin.                                                                                                                              |
| `autoEnableWhenConfiguredProviders` | No          | `string[]`                       | Ids de proveedor que deben autoactivar este plugin cuando auth, config o referencias de modelo los mencionen.                                                                             |
| `kind`                              | No          | `"memory"` \| `"context-engine"` | Declara un tipo exclusivo de plugin usado por `plugins.slots.*`.                                                                                                                           |
| `channels`                          | No          | `string[]`                       | Ids de canal propiedad de este plugin. Se usan para discovery y validación de configuración.                                                                                               |
| `providers`                         | No          | `string[]`                       | Ids de proveedor propiedad de este plugin.                                                                                                                                                 |
| `modelSupport`                      | No          | `object`                         | Metadatos abreviados de familias de modelos propiedad del manifiesto usados para cargar automáticamente el plugin antes del runtime.                                                      |
| `cliBackends`                       | No          | `string[]`                       | Ids de backend CLI de inferencia propiedad de este plugin. Se usan para autoactivación al inicio desde referencias explícitas en configuración.                                           |
| `providerAuthEnvVars`               | No          | `Record<string, string[]>`       | Metadatos ligeros de variables de entorno de autenticación de proveedor que OpenClaw puede inspeccionar sin cargar código del plugin.                                                     |
| `providerAuthChoices`               | No          | `object[]`                       | Metadatos ligeros de opciones de autenticación para selectores de onboarding, resolución de proveedores preferidos y cableado simple de flags de CLI.                                    |
| `contracts`                         | No          | `object`                         | Instantánea estática de capacidades empaquetadas para voz, transcripción en tiempo real, voz en tiempo real, media-understanding, image-generation, video-generation, web-fetch, web search y propiedad de herramientas. |
| `channelConfigs`                    | No          | `Record<string, object>`         | Metadatos de configuración de canal propiedad del manifiesto fusionados en superficies de discovery y validación antes de cargar el runtime.                                              |
| `skills`                            | No          | `string[]`                       | Directorios de Skills que se deben cargar, relativos a la raíz del plugin.                                                                                                                 |
| `name`                              | No          | `string`                         | Nombre legible del plugin.                                                                                                                                                                 |
| `description`                       | No          | `string`                         | Resumen breve mostrado en superficies del plugin.                                                                                                                                          |
| `version`                           | No          | `string`                         | Versión informativa del plugin.                                                                                                                                                            |
| `uiHints`                           | No          | `Record<string, object>`         | Etiquetas de interfaz, placeholders y sugerencias de sensibilidad para campos de configuración.                                                                                            |

## Referencia de `providerAuthChoices`

Cada entrada de `providerAuthChoices` describe una opción de onboarding o autenticación.
OpenClaw la lee antes de cargar el runtime del proveedor.

| Campo                 | Obligatorio | Tipo                                            | Qué significa                                                                                                 |
| --------------------- | ----------- | ----------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| `provider`            | Sí          | `string`                                        | Id del proveedor al que pertenece esta opción.                                                                |
| `method`              | Sí          | `string`                                        | Id del método de autenticación al que se debe enviar.                                                         |
| `choiceId`            | Sí          | `string`                                        | Id estable de opción de autenticación usado por onboarding y flujos de CLI.                                  |
| `choiceLabel`         | No          | `string`                                        | Etiqueta visible para el usuario. Si se omite, OpenClaw recurre a `choiceId`.                                |
| `choiceHint`          | No          | `string`                                        | Texto breve de ayuda para el selector.                                                                        |
| `assistantPriority`   | No          | `number`                                        | Los valores más bajos se ordenan antes en selectores interactivos guiados por asistente.                     |
| `assistantVisibility` | No          | `"visible"` \| `"manual-only"`                  | Oculta la opción en selectores del asistente mientras sigue permitiendo selección manual desde la CLI.       |
| `deprecatedChoiceIds` | No          | `string[]`                                      | Ids heredados de opciones que deben redirigir a los usuarios a esta opción de reemplazo.                     |
| `groupId`             | No          | `string`                                        | Id opcional de grupo para agrupar opciones relacionadas.                                                      |
| `groupLabel`          | No          | `string`                                        | Etiqueta visible para el usuario de ese grupo.                                                                |
| `groupHint`           | No          | `string`                                        | Texto breve de ayuda para el grupo.                                                                           |
| `optionKey`           | No          | `string`                                        | Clave de opción interna para flujos simples de autenticación con una sola flag.                              |
| `cliFlag`             | No          | `string`                                        | Nombre de flag de CLI, como `--openrouter-api-key`.                                                           |
| `cliOption`           | No          | `string`                                        | Forma completa de la opción de CLI, como `--openrouter-api-key <key>`.                                        |
| `cliDescription`      | No          | `string`                                        | Descripción usada en la ayuda de la CLI.                                                                      |
| `onboardingScopes`    | No          | `Array<"text-inference" \| "image-generation">` | En qué superficies de onboarding debe aparecer esta opción. Si se omite, el valor predeterminado es `["text-inference"]`. |

## Referencia de `uiHints`

`uiHints` es un mapa desde nombres de campos de configuración hasta pequeñas sugerencias de renderizado.

```json
{
  "uiHints": {
    "apiKey": {
      "label": "Clave API",
      "help": "Usada para solicitudes a OpenRouter",
      "placeholder": "sk-or-v1-...",
      "sensitive": true
    }
  }
}
```

Cada sugerencia de campo puede incluir:

| Campo         | Tipo       | Qué significa                           |
| ------------- | ---------- | --------------------------------------- |
| `label`       | `string`   | Etiqueta visible para el usuario.       |
| `help`        | `string`   | Texto breve de ayuda.                   |
| `tags`        | `string[]` | Etiquetas opcionales de interfaz.       |
| `advanced`    | `boolean`  | Marca el campo como avanzado.           |
| `sensitive`   | `boolean`  | Marca el campo como secreto o sensible. |
| `placeholder` | `string`   | Texto de ejemplo para entradas de formulario. |

## Referencia de `contracts`

Usa `contracts` solo para metadatos estáticos de propiedad de capacidades que OpenClaw pueda
leer sin importar el runtime del plugin.

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

| Campo                            | Tipo       | Qué significa                                                  |
| -------------------------------- | ---------- | -------------------------------------------------------------- |
| `speechProviders`                | `string[]` | Ids de proveedor de voz propiedad de este plugin.              |
| `realtimeTranscriptionProviders` | `string[]` | Ids de proveedor de transcripción en tiempo real propiedad de este plugin. |
| `realtimeVoiceProviders`         | `string[]` | Ids de proveedor de voz en tiempo real propiedad de este plugin. |
| `mediaUnderstandingProviders`    | `string[]` | Ids de proveedor de media-understanding propiedad de este plugin. |
| `imageGenerationProviders`       | `string[]` | Ids de proveedor de image-generation propiedad de este plugin. |
| `videoGenerationProviders`       | `string[]` | Ids de proveedor de video-generation propiedad de este plugin. |
| `webFetchProviders`              | `string[]` | Ids de proveedor de web-fetch propiedad de este plugin.        |
| `webSearchProviders`             | `string[]` | Ids de proveedor de web search propiedad de este plugin.       |
| `tools`                          | `string[]` | Nombres de herramientas de agente propiedad de este plugin para comprobaciones de contrato empaquetado. |

## Referencia de `channelConfigs`

Usa `channelConfigs` cuando un plugin de canal necesite metadatos ligeros de configuración antes de
cargar el runtime.

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

| Campo         | Tipo                     | Qué significa                                                                                   |
| ------------- | ------------------------ | ----------------------------------------------------------------------------------------------- |
| `schema`      | `object`                 | JSON Schema para `channels.<id>`. Obligatorio para cada entrada declarada de configuración de canal. |
| `uiHints`     | `Record<string, object>` | Etiquetas/placeholders/sugerencias de sensibilidad opcionales de interfaz para esa sección de configuración de canal. |
| `label`       | `string`                 | Etiqueta del canal fusionada en superficies de selector e inspección cuando los metadatos del runtime aún no están listos. |
| `description` | `string`                 | Descripción breve del canal para superficies de inspección y catálogo.                          |
| `preferOver`  | `string[]`               | Ids de plugins heredados o de menor prioridad a los que este canal debe superar en superficies de selección. |

## Referencia de `modelSupport`

Usa `modelSupport` cuando OpenClaw deba inferir tu plugin de proveedor a partir de
ids abreviados de modelos como `gpt-5.4` o `claude-sonnet-4.6` antes de cargar el runtime del plugin.

```json
{
  "modelSupport": {
    "modelPrefixes": ["gpt-", "o1", "o3", "o4"],
    "modelPatterns": ["^computer-use-preview"]
  }
}
```

OpenClaw aplica esta precedencia:

- las referencias explícitas `provider/model` usan los metadatos de manifiesto `providers` del propietario
- `modelPatterns` tiene prioridad sobre `modelPrefixes`
- si un plugin no empaquetado y uno empaquetado coinciden ambos, gana el plugin no empaquetado
- la ambigüedad restante se ignora hasta que el usuario o la configuración especifiquen un proveedor

Campos:

| Campo           | Tipo       | Qué significa                                                                   |
| --------------- | ---------- | ------------------------------------------------------------------------------- |
| `modelPrefixes` | `string[]` | Prefijos emparejados con `startsWith` frente a ids abreviados de modelos.      |
| `modelPatterns` | `string[]` | Orígenes regex emparejados frente a ids abreviados de modelos tras eliminar el sufijo del perfil. |

Las claves heredadas de capacidades de nivel superior están obsoletas. Usa `openclaw doctor --fix` para
mover `speechProviders`, `realtimeTranscriptionProviders`,
`realtimeVoiceProviders`, `mediaUnderstandingProviders`,
`imageGenerationProviders`, `videoGenerationProviders`,
`webFetchProviders` y `webSearchProviders` bajo `contracts`; la carga normal
del manifiesto ya no trata esos campos de nivel superior como propiedad de
capacidades.

## Manifiesto frente a package.json

Los dos archivos cumplen funciones distintas:

| Archivo                | Úsalo para                                                                                                                               |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.plugin.json` | Discovery, validación de configuración, metadatos de opciones de autenticación y sugerencias de interfaz que deben existir antes de ejecutar el código del plugin |
| `package.json`         | Metadatos de npm, instalación de dependencias y el bloque `openclaw` usado para puntos de entrada, control de instalación, configuración o metadatos de catálogo |

Si no estás seguro de dónde debe ir un metadato, usa esta regla:

- si OpenClaw debe conocerlo antes de cargar el código del plugin, colócalo en `openclaw.plugin.json`
- si trata de empaquetado, archivos de entrada o comportamiento de instalación con npm, colócalo en `package.json`

### Campos de package.json que afectan a discovery

Algunos metadatos del plugin previos al runtime viven intencionadamente en `package.json` dentro del
bloque `openclaw` en lugar de en `openclaw.plugin.json`.

Ejemplos importantes:

| Campo                                                             | Qué significa                                                                          |
| ----------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| `openclaw.extensions`                                             | Declara puntos de entrada de plugins nativos.                                          |
| `openclaw.setupEntry`                                             | Punto de entrada ligero solo para configuración usado durante onboarding e inicio diferido de canales. |
| `openclaw.channel`                                                | Metadatos ligeros de catálogo de canal como etiquetas, rutas de documentación, alias y texto de selección. |
| `openclaw.install.npmSpec` / `openclaw.install.localPath`         | Sugerencias de instalación/actualización para plugins empaquetados y publicados externamente. |
| `openclaw.install.defaultChoice`                                  | Ruta de instalación preferida cuando hay varias fuentes de instalación disponibles.    |
| `openclaw.install.minHostVersion`                                 | Versión mínima compatible del host OpenClaw, usando un mínimo semver como `>=2026.3.22`. |
| `openclaw.install.allowInvalidConfigRecovery`                     | Permite una ruta estrecha de recuperación por reinstalación de plugin empaquetado cuando la configuración no es válida. |
| `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen` | Permite que las superficies de canal solo de configuración se carguen antes del plugin completo del canal durante el inicio. |

`openclaw.install.minHostVersion` se aplica durante la instalación y la
carga del registro de manifiestos. Los valores no válidos se rechazan; los
valores más recientes pero válidos omiten el plugin en hosts más antiguos.

`openclaw.install.allowInvalidConfigRecovery` es intencionalmente estrecho. No
hace que configuraciones arbitrariamente rotas se puedan instalar. Hoy solo permite que los flujos de instalación
se recuperen de fallos concretos de actualización de plugins empaquetados obsoletos, como una
ruta faltante del plugin empaquetado o una entrada `channels.<id>` obsoleta para ese mismo
plugin empaquetado. Los errores de configuración no relacionados siguen bloqueando la instalación y envían a los operadores
a `openclaw doctor --fix`.

## Requisitos de JSON Schema

- **Todo plugin debe incluir un JSON Schema**, incluso si no acepta configuración.
- Se acepta un esquema vacío (por ejemplo, `{ "type": "object", "additionalProperties": false }`).
- Los esquemas se validan en tiempo de lectura/escritura de configuración, no en runtime.

## Comportamiento de validación

- Las claves desconocidas `channels.*` son **errores**, a menos que el id de canal esté declarado por
  el manifiesto de un plugin.
- `plugins.entries.<id>`, `plugins.allow`, `plugins.deny` y `plugins.slots.*`
  deben hacer referencia a ids de plugins **detectables**. Los ids desconocidos son **errores**.
- Si un plugin está instalado pero tiene un manifiesto o esquema roto o ausente,
  la validación falla y Doctor informa del error del plugin.
- Si la configuración del plugin existe pero el plugin está **deshabilitado**, la configuración se conserva y
  aparece una **advertencia** en Doctor + registros.

Consulta [Referencia de configuración](/gateway/configuration) para ver el esquema completo de `plugins.*`.

## Notas

- El manifiesto es **obligatorio para plugins nativos de OpenClaw**, incluidas las cargas desde el sistema de archivos local.
- El runtime sigue cargando el módulo del plugin por separado; el manifiesto es solo para
  discovery + validación.
- Los manifiestos nativos se analizan con JSON5, por lo que se aceptan comentarios, comas finales y
  claves sin comillas siempre que el valor final siga siendo un objeto.
- El cargador de manifiestos solo lee los campos de manifiesto documentados. Evita añadir
  claves de nivel superior personalizadas aquí.
- `providerAuthEnvVars` es la ruta ligera de metadatos para sondeos de autenticación, validación de
  marcadores de entorno y superficies similares de autenticación de proveedor que no deberían iniciar el
  runtime del plugin solo para inspeccionar nombres de variables de entorno.
- `providerAuthChoices` es la ruta ligera de metadatos para selectores de opciones de autenticación,
  resolución de `--auth-choice`, mapeo de proveedor preferido y registro simple de flags
  de CLI en onboarding antes de cargar el runtime del proveedor. Para metadatos del asistente en runtime que
  requieren código del proveedor, consulta
  [Hooks de runtime del proveedor](/plugins/architecture#provider-runtime-hooks).
- Los tipos exclusivos de plugin se seleccionan mediante `plugins.slots.*`.
  - `kind: "memory"` se selecciona mediante `plugins.slots.memory`.
  - `kind: "context-engine"` se selecciona mediante `plugins.slots.contextEngine`
    (predeterminado: `legacy` integrado).
- `channels`, `providers`, `cliBackends` y `skills` pueden omitirse cuando un
  plugin no los necesite.
- Si tu plugin depende de módulos nativos, documenta los pasos de compilación y cualquier
  requisito de lista de permitidos del gestor de paquetes (por ejemplo, pnpm `allow-build-scripts`
  - `pnpm rebuild <package>`).

## Relacionado

- [Crear plugins](/plugins/building-plugins) — introducción a los plugins
- [Arquitectura de plugins](/plugins/architecture) — arquitectura interna
- [Resumen del SDK](/plugins/sdk-overview) — referencia del SDK de plugins
