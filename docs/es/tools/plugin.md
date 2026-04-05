---
read_when:
    - Instalas o configuras plugins
    - Quieres entender las reglas de descubrimiento y carga de plugins
    - Trabajas con paquetes de plugins compatibles con Codex/Claude
sidebarTitle: Install and Configure
summary: Instala, configura y administra plugins de OpenClaw
title: Plugins
x-i18n:
    generated_at: "2026-04-05T12:56:45Z"
    model: gpt-5.4
    provider: openai
    source_hash: 707bd3625596f290322aeac9fecb7f4c6f45d595fdfb82ded7cbc8e04457ac7f
    source_path: tools/plugin.md
    workflow: 15
---

# Plugins

Los plugins amplían OpenClaw con nuevas capacidades: canales, proveedores de modelos,
herramientas, Skills, voz, transcripción en tiempo real, voz en tiempo real,
media-understanding, generación de imágenes, generación de video, obtención web, búsqueda web
y más. Algunos plugins son **core** (incluidos con OpenClaw), otros
son **external** (publicados en npm por la comunidad).

## Inicio rápido

<Steps>
  <Step title="Ver qué está cargado">
    ```bash
    openclaw plugins list
    ```
  </Step>

  <Step title="Instalar un plugin">
    ```bash
    # Desde npm
    openclaw plugins install @openclaw/voice-call

    # Desde un directorio local o archivo
    openclaw plugins install ./my-plugin
    openclaw plugins install ./my-plugin.tgz
    ```

  </Step>

  <Step title="Reiniciar el Gateway">
    ```bash
    openclaw gateway restart
    ```

    Luego configura en `plugins.entries.\<id\>.config` en tu archivo de configuración.

  </Step>
</Steps>

Si prefieres un control nativo desde el chat, habilita `commands.plugins: true` y usa:

```text
/plugin install clawhub:@openclaw/voice-call
/plugin show voice-call
/plugin enable voice-call
```

La ruta de instalación usa el mismo resolvedor que la CLI: ruta/archivo local, `clawhub:<pkg>`
explícito o especificación de paquete simple (primero ClawHub y luego npm como respaldo).

Si la configuración no es válida, la instalación normalmente falla de forma segura y te indica
`openclaw doctor --fix`. La única excepción de recuperación es una ruta limitada de reinstalación
de plugins integrados para plugins que habilitan
`openclaw.install.allowInvalidConfigRecovery`.

## Tipos de plugins

OpenClaw reconoce dos formatos de plugins:

| Formato   | Cómo funciona                                                       | Ejemplos                                               |
| ---------- | ------------------------------------------------------------------ | ------------------------------------------------------ |
| **Native** | `openclaw.plugin.json` + módulo de runtime; se ejecuta en proceso       | Plugins oficiales, paquetes npm de la comunidad               |
| **Bundle** | Diseño compatible con Codex/Claude/Cursor; mapeado a funciones de OpenClaw | `.codex-plugin/`, `.claude-plugin/`, `.cursor-plugin/` |

Ambos aparecen en `openclaw plugins list`. Consulta [Paquetes de plugins](/es/plugins/bundles) para obtener detalles sobre bundles.

Si vas a escribir un plugin nativo, empieza con [Creación de plugins](/es/plugins/building-plugins)
y la [Descripción general del SDK de plugins](/es/plugins/sdk-overview).

## Plugins oficiales

### Instalables (npm)

| Plugin          | Paquete                | Documentación                                 |
| --------------- | ---------------------- | ------------------------------------ |
| Matrix          | `@openclaw/matrix`     | [Matrix](/es/channels/matrix)           |
| Microsoft Teams | `@openclaw/msteams`    | [Microsoft Teams](/es/channels/msteams) |
| Nostr           | `@openclaw/nostr`      | [Nostr](/es/channels/nostr)             |
| Voice Call      | `@openclaw/voice-call` | [Voice Call](/es/plugins/voice-call)    |
| Zalo            | `@openclaw/zalo`       | [Zalo](/es/channels/zalo)               |
| Zalo Personal   | `@openclaw/zalouser`   | [Zalo Personal](/es/plugins/zalouser)   |

### Core (incluidos con OpenClaw)

<AccordionGroup>
  <Accordion title="Proveedores de modelos (habilitados de forma predeterminada)">
    `anthropic`, `byteplus`, `cloudflare-ai-gateway`, `github-copilot`, `google`,
    `huggingface`, `kilocode`, `kimi-coding`, `minimax`, `mistral`, `qwen`,
    `moonshot`, `nvidia`, `openai`, `opencode`, `opencode-go`, `openrouter`,
    `qianfan`, `synthetic`, `together`, `venice`,
    `vercel-ai-gateway`, `volcengine`, `xiaomi`, `zai`
  </Accordion>

  <Accordion title="Plugins de memoria">
    - `memory-core` — búsqueda de memoria integrada (predeterminada mediante `plugins.slots.memory`)
    - `memory-lancedb` — memoria a largo plazo con instalación bajo demanda, recuperación/captura automática (configura `plugins.slots.memory = "memory-lancedb"`)
  </Accordion>

  <Accordion title="Proveedores de voz (habilitados de forma predeterminada)">
    `elevenlabs`, `microsoft`
  </Accordion>

  <Accordion title="Otros">
    - `browser` — plugin de navegador integrado para la herramienta del navegador, la CLI `openclaw browser`, el método de gateway `browser.request`, el runtime del navegador y el servicio de control del navegador predeterminado (habilitado de forma predeterminada; desactívalo antes de reemplazarlo)
    - `copilot-proxy` — puente de VS Code Copilot Proxy (deshabilitado de forma predeterminada)
  </Accordion>
</AccordionGroup>

¿Buscas plugins de terceros? Consulta [Plugins de la comunidad](/es/plugins/community).

## Configuración

```json5
{
  plugins: {
    enabled: true,
    allow: ["voice-call"],
    deny: ["untrusted-plugin"],
    load: { paths: ["~/Projects/oss/voice-call-extension"] },
    entries: {
      "voice-call": { enabled: true, config: { provider: "twilio" } },
    },
  },
}
```

| Campo            | Descripción                                               |
| ---------------- | --------------------------------------------------------- |
| `enabled`        | Interruptor principal (predeterminado: `true`)                           |
| `allow`          | Lista de permitidos de plugins (opcional)                               |
| `deny`           | Lista de denegados de plugins (opcional; `deny` tiene prioridad)                     |
| `load.paths`     | Archivos/directorios de plugins adicionales                            |
| `slots`          | Selectores de slots exclusivos (por ejemplo `memory`, `contextEngine`) |
| `entries.\<id\>` | Interruptores y configuración por plugin                               |

Los cambios de configuración **requieren reiniciar el gateway**. Si el Gateway se está ejecutando con vigilancia de configuración + reinicio en proceso habilitados (la ruta predeterminada de `openclaw gateway`), ese
reinicio normalmente se realiza automáticamente poco después de que se aplique la escritura de configuración.

<Accordion title="Estados del plugin: deshabilitado frente a ausente frente a no válido">
  - **Deshabilitado**: el plugin existe, pero las reglas de habilitación lo desactivaron. La configuración se conserva.
  - **Ausente**: la configuración hace referencia a un id de plugin que el descubrimiento no encontró.
  - **No válido**: el plugin existe, pero su configuración no coincide con el esquema declarado.
</Accordion>

## Descubrimiento y prioridad

OpenClaw busca plugins en este orden (la primera coincidencia gana):

<Steps>
  <Step title="Rutas de configuración">
    `plugins.load.paths`: rutas explícitas a archivos o directorios.
  </Step>

  <Step title="Extensiones del espacio de trabajo">
    `\<workspace\>/.openclaw/<plugin-root>/*.ts` y `\<workspace\>/.openclaw/<plugin-root>/*/index.ts`.
  </Step>

  <Step title="Extensiones globales">
    `~/.openclaw/<plugin-root>/*.ts` y `~/.openclaw/<plugin-root>/*/index.ts`.
  </Step>

  <Step title="Plugins integrados">
    Incluidos con OpenClaw. Muchos están habilitados de forma predeterminada (proveedores de modelos, voz).
    Otros requieren habilitación explícita.
  </Step>
</Steps>

### Reglas de habilitación

- `plugins.enabled: false` deshabilita todos los plugins
- `plugins.deny` siempre tiene prioridad sobre `allow`
- `plugins.entries.\<id\>.enabled: false` deshabilita ese plugin
- Los plugins de origen del espacio de trabajo están **deshabilitados de forma predeterminada** (deben habilitarse explícitamente)
- Los plugins integrados siguen el conjunto incorporado activado por defecto, salvo reemplazo
- Los slots exclusivos pueden forzar la habilitación del plugin seleccionado para ese slot

## Slots de plugins (categorías exclusivas)

Algunas categorías son exclusivas (solo una activa a la vez):

```json5
{
  plugins: {
    slots: {
      memory: "memory-core", // o "none" para deshabilitar
      contextEngine: "legacy", // o un id de plugin
    },
  },
}
```

| Slot            | Qué controla      | Predeterminado             |
| --------------- | --------------------- | ------------------- |
| `memory`        | Plugin de memoria activo  | `memory-core`       |
| `contextEngine` | Motor de contexto activo | `legacy` (integrado) |

## Referencia de CLI

```bash
openclaw plugins list                       # inventario compacto
openclaw plugins list --enabled            # solo plugins cargados
openclaw plugins list --verbose            # líneas detalladas por plugin
openclaw plugins list --json               # inventario legible por máquina
openclaw plugins inspect <id>              # detalle profundo
openclaw plugins inspect <id> --json       # legible por máquina
openclaw plugins inspect --all             # tabla de toda la flota
openclaw plugins info <id>                 # alias de inspect
openclaw plugins doctor                    # diagnósticos

openclaw plugins install <package>         # instalar (primero ClawHub, luego npm)
openclaw plugins install clawhub:<pkg>     # instalar solo desde ClawHub
openclaw plugins install <spec> --force    # sobrescribir instalación existente
openclaw plugins install <path>            # instalar desde ruta local
openclaw plugins install -l <path>         # enlazar (sin copiar) para desarrollo
openclaw plugins install <plugin> --marketplace <source>
openclaw plugins install <plugin> --marketplace https://github.com/<owner>/<repo>
openclaw plugins install <spec> --pin      # registrar la especificación npm exacta resuelta
openclaw plugins install <spec> --dangerously-force-unsafe-install
openclaw plugins update <id>             # actualizar un plugin
openclaw plugins update <id> --dangerously-force-unsafe-install
openclaw plugins update --all            # actualizar todos
openclaw plugins uninstall <id>          # eliminar registros de configuración/instalación
openclaw plugins uninstall <id> --keep-files
openclaw plugins marketplace list <source>
openclaw plugins marketplace list <source> --json

openclaw plugins enable <id>
openclaw plugins disable <id>
```

Los plugins integrados se incluyen con OpenClaw. Muchos están habilitados de forma predeterminada (por ejemplo,
proveedores de modelos integrados, proveedores de voz integrados y el plugin de navegador
integrado). Otros plugins integrados aún necesitan `openclaw plugins enable <id>`.

`--force` sobrescribe en su lugar un plugin o paquete de hooks ya instalado.
No es compatible con `--link`, que reutiliza la ruta de origen en lugar de
copiar sobre un destino de instalación administrado.

`--pin` es solo para npm. No es compatible con `--marketplace`, porque
las instalaciones desde marketplace conservan metadatos del origen del marketplace en lugar de una especificación npm.

`--dangerously-force-unsafe-install` es una anulación de emergencia para falsos
positivos del escáner integrado de código peligroso. Permite que las instalaciones
y actualizaciones de plugins continúen más allá de hallazgos integrados `critical`, pero aun así
no omite los bloqueos de política `before_install` del plugin ni el bloqueo por fallos de escaneo.

Este flag de CLI se aplica solo a flujos de instalación/actualización de plugins. Las
instalaciones de dependencias de Skills respaldadas por gateway usan en su lugar la anulación de solicitud correspondiente
`dangerouslyForceUnsafeInstall`, mientras que `openclaw skills install` sigue siendo el flujo separado de descarga/instalación de Skills de ClawHub.

Los bundles compatibles participan en el mismo flujo de listar/inspeccionar/habilitar/deshabilitar
plugins. La compatibilidad actual de runtime incluye Skills de bundles, command-skills de Claude,
valores predeterminados de Claude `settings.json`, valores predeterminados de Claude `.lsp.json` y de
`lspServers` declarados en el manifiesto, command-skills de Cursor y directorios de hooks compatibles con Codex.

`openclaw plugins inspect <id>` también informa las capacidades de bundle detectadas, además de
las entradas de servidores MCP y LSP compatibles o no compatibles para plugins respaldados por bundles.

Los orígenes de marketplace pueden ser un nombre de marketplace conocido de Claude desde
`~/.claude/plugins/known_marketplaces.json`, un root local de marketplace o una ruta a
`marketplace.json`, una abreviatura de GitHub como `owner/repo`, una URL de repositorio de GitHub
o una URL git. Para marketplaces remotos, las entradas de plugins deben permanecer dentro del
repositorio clonado del marketplace y usar solo orígenes de ruta relativa.

Consulta la [referencia de CLI `openclaw plugins`](/cli/plugins) para todos los detalles.

## Descripción general de la API de plugins

Los plugins nativos exportan un objeto de entrada que expone `register(api)`. Los
plugins más antiguos aún pueden usar `activate(api)` como alias heredado, pero los plugins nuevos deben
usar `register`.

```typescript
export default definePluginEntry({
  id: "my-plugin",
  name: "My Plugin",
  register(api) {
    api.registerProvider({
      /* ... */
    });
    api.registerTool({
      /* ... */
    });
    api.registerChannel({
      /* ... */
    });
  },
});
```

OpenClaw carga el objeto de entrada y llama a `register(api)` durante la
activación del plugin. El cargador sigue recurriendo a `activate(api)` para plugins más antiguos,
pero los plugins integrados y los nuevos plugins externos deben tratar `register` como el
contrato público.

Métodos de registro comunes:

| Método                                  | Qué registra           |
| --------------------------------------- | --------------------------- |
| `registerProvider`                      | Proveedor de modelos (LLM)        |
| `registerChannel`                       | Canal de chat                |
| `registerTool`                          | Herramienta del agente                  |
| `registerHook` / `on(...)`              | Hooks del ciclo de vida             |
| `registerSpeechProvider`                | Texto a voz / STT        |
| `registerRealtimeTranscriptionProvider` | STT en streaming               |
| `registerRealtimeVoiceProvider`         | Voz bidireccional en tiempo real       |
| `registerMediaUnderstandingProvider`    | Análisis de imagen/audio        |
| `registerImageGenerationProvider`       | Generación de imágenes            |
| `registerVideoGenerationProvider`       | Generación de video            |
| `registerWebFetchProvider`              | Proveedor de obtención/scraping web |
| `registerWebSearchProvider`             | Búsqueda web                  |
| `registerHttpRoute`                     | Endpoint HTTP               |
| `registerCommand` / `registerCli`       | Comandos de CLI                |
| `registerContextEngine`                 | Motor de contexto              |
| `registerService`                       | Servicio en segundo plano          |

Comportamiento de guardas de hooks para hooks tipados del ciclo de vida:

- `before_tool_call`: `{ block: true }` es terminal; se omiten los controladores de menor prioridad.
- `before_tool_call`: `{ block: false }` no hace nada y no elimina un bloqueo anterior.
- `before_install`: `{ block: true }` es terminal; se omiten los controladores de menor prioridad.
- `before_install`: `{ block: false }` no hace nada y no elimina un bloqueo anterior.
- `message_sending`: `{ cancel: true }` es terminal; se omiten los controladores de menor prioridad.
- `message_sending`: `{ cancel: false }` no hace nada y no elimina una cancelación anterior.

Para el comportamiento completo de hooks tipados, consulta [Descripción general del SDK](/es/plugins/sdk-overview#hook-decision-semantics).

## Relacionado

- [Creación de plugins](/es/plugins/building-plugins): crea tu propio plugin
- [Paquetes de plugins](/es/plugins/bundles): compatibilidad de bundles con Codex/Claude/Cursor
- [Manifiesto de plugins](/es/plugins/manifest): esquema del manifiesto
- [Registro de herramientas](/es/plugins/building-plugins#registering-agent-tools): añade herramientas de agente en un plugin
- [Detalles internos de plugins](/es/plugins/architecture): modelo de capacidades y canalización de carga
- [Plugins de la comunidad](/es/plugins/community): listados de terceros
