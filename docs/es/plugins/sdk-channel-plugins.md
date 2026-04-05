---
read_when:
    - Estás creando un nuevo plugin de canal de mensajería
    - Quieres conectar OpenClaw a una plataforma de mensajería
    - Necesitas entender la superficie del adaptador ChannelPlugin
sidebarTitle: Channel Plugins
summary: Guía paso a paso para crear un plugin de canal de mensajería para OpenClaw
title: Creación de plugins de canal
x-i18n:
    generated_at: "2026-04-05T12:50:08Z"
    model: gpt-5.4
    provider: openai
    source_hash: 68a6ad2c75549db8ce54f7e22ca9850d7ed68c5cd651c9bb41c9f73769f48aba
    source_path: plugins/sdk-channel-plugins.md
    workflow: 15
---

# Creación de plugins de canal

Esta guía explica cómo crear un plugin de canal que conecte OpenClaw a una
plataforma de mensajería. Al final tendrás un canal funcional con seguridad en DM,
emparejamiento, encadenamiento de respuestas y mensajería saliente.

<Info>
  Si aún no has creado ningún plugin de OpenClaw, lee primero
  [Getting Started](/plugins/building-plugins) para conocer la estructura básica
  del paquete y la configuración del manifiesto.
</Info>

## Cómo funcionan los plugins de canal

Los plugins de canal no necesitan sus propias herramientas send/edit/react. OpenClaw mantiene una
única herramienta `message` compartida en el núcleo. Tu plugin es propietario de:

- **Configuración** — resolución de cuentas y asistente de configuración
- **Seguridad** — política de DM y listas de permitidos
- **Emparejamiento** — flujo de aprobación de DM
- **Gramática de sesión** — cómo los IDs de conversación específicos del proveedor se asignan a chats base, IDs de hilo y fallbacks del padre
- **Salida** — envío de texto, medios y encuestas a la plataforma
- **Encadenamiento** — cómo se encadenan las respuestas

El núcleo es propietario de la herramienta `message` compartida, del cableado del prompt, de la forma externa de la clave de sesión,
de la gestión genérica de `:thread:` y del dispatch.

Si tu plataforma almacena alcance adicional dentro de los IDs de conversación, mantén ese análisis
en el plugin con `messaging.resolveSessionConversation(...)`. Ese es el hook canónico
para mapear `rawId` al id base de conversación, un id de hilo opcional, `baseConversationId`
explícito y cualquier `parentConversationCandidates`.
Cuando devuelvas `parentConversationCandidates`, mantenlos ordenados desde el
padre más específico hasta la conversación base/más amplia.

Los plugins integrados que necesitan el mismo análisis antes de que se inicie el registro de canales
también pueden exponer un archivo `session-key-api.ts` de nivel superior con un export
`resolveSessionConversation(...)` equivalente. El núcleo usa esa superficie segura para bootstrap
solo cuando el registro de plugins del runtime aún no está disponible.

`messaging.resolveParentConversationCandidates(...)` sigue disponible como fallback heredado de compatibilidad cuando un plugin solo necesita fallbacks del padre sobre el id genérico/raw. Si existen ambos hooks, el núcleo usa primero
`resolveSessionConversation(...).parentConversationCandidates` y solo
recurre a `resolveParentConversationCandidates(...)` cuando el hook canónico
los omite.

## Aprobaciones y capacidades del canal

La mayoría de los plugins de canal no necesitan código específico de aprobaciones.

- El núcleo es propietario de `/approve` en el mismo chat, de las cargas de botones de aprobación compartidas y de la entrega genérica de fallback.
- Prefiere un único objeto `approvalCapability` en el plugin de canal cuando el canal necesita comportamiento específico de aprobación.
- `approvalCapability.authorizeActorAction` y `approvalCapability.getActionAvailabilityState` son la unión canónica para autenticación de aprobación.
- Usa `outbound.shouldSuppressLocalPayloadPrompt` o `outbound.beforeDeliverPayload` para comportamiento específico del canal en el ciclo de vida de la carga, como ocultar solicitudes de aprobación locales duplicadas o enviar indicadores de escritura antes de la entrega.
- Usa `approvalCapability.delivery` solo para enrutamiento nativo de aprobación o supresión de fallback.
- Usa `approvalCapability.render` solo cuando un canal realmente necesita cargas de aprobación personalizadas en lugar del renderizador compartido.
- Si un canal puede inferir identidades DM estables de tipo propietario a partir de la configuración existente, usa `createResolvedApproverActionAuthAdapter` de `openclaw/plugin-sdk/approval-runtime` para restringir `/approve` en el mismo chat sin añadir lógica específica de aprobación al núcleo.
- Si un canal necesita entrega nativa de aprobación, mantén el código del canal centrado en la normalización de destinos y hooks de transporte. Usa `createChannelExecApprovalProfile`, `createChannelNativeOriginTargetResolver`, `createChannelApproverDmTargetResolver`, `createApproverRestrictedNativeApprovalCapability` y `createChannelNativeApprovalRuntime` de `openclaw/plugin-sdk/approval-runtime` para que el núcleo gestione el filtrado de solicitudes, el enrutamiento, la eliminación de duplicados, la caducidad y la suscripción a la gateway.
- Los canales de aprobación nativa deben enrutar tanto `accountId` como `approvalKind` a través de esos helpers. `accountId` mantiene la política de aprobación de varias cuentas delimitada a la cuenta correcta del bot, y `approvalKind` mantiene disponible para el canal el comportamiento de aprobación exec frente a plugin sin ramas codificadas en el núcleo.
- Conserva el tipo de id de aprobación entregado de extremo a extremo. Los clientes nativos no deben
  adivinar ni reescribir el enrutamiento de aprobación exec frente a plugin a partir del estado local del canal.
- Distintos tipos de aprobación pueden exponer intencionadamente superficies nativas diferentes.
  Ejemplos integrados actuales:
  - Slack mantiene disponible el enrutamiento nativo de aprobaciones tanto para IDs exec como plugin.
  - Matrix mantiene el enrutamiento nativo DM/canal solo para aprobaciones exec y deja
    las aprobaciones de plugin en la ruta compartida `/approve` del mismo chat.
- `createApproverRestrictedNativeApprovalAdapter` sigue existiendo como wrapper de compatibilidad, pero el código nuevo debería preferir el constructor de capacidades y exponer `approvalCapability` en el plugin.

Para entrypoints de canal activos, prefiere las subrutas de runtime más acotadas cuando solo
necesitas una parte de esa familia:

- `openclaw/plugin-sdk/approval-auth-runtime`
- `openclaw/plugin-sdk/approval-client-runtime`
- `openclaw/plugin-sdk/approval-delivery-runtime`
- `openclaw/plugin-sdk/approval-native-runtime`
- `openclaw/plugin-sdk/approval-reply-runtime`

Del mismo modo, prefiere `openclaw/plugin-sdk/setup-runtime`,
`openclaw/plugin-sdk/setup-adapter-runtime`,
`openclaw/plugin-sdk/reply-runtime`,
`openclaw/plugin-sdk/reply-dispatch-runtime`,
`openclaw/plugin-sdk/reply-reference` y
`openclaw/plugin-sdk/reply-chunking` cuando no necesites la superficie paraguas más amplia.

En concreto para setup:

- `openclaw/plugin-sdk/setup-runtime` cubre los helpers de setup seguros para runtime:
  adaptadores de parcheo de setup seguros para importación (`createPatchedAccountSetupAdapter`,
  `createEnvPatchedAccountSetupAdapter`,
  `createSetupInputPresenceValidator`), salida de notas de lookup,
  `promptResolvedAllowFrom`, `splitSetupEntries` y los constructores de
  setup-proxy delegado
- `openclaw/plugin-sdk/setup-adapter-runtime` es la unión estrecha con reconocimiento de entorno
  para `createEnvPatchedAccountSetupAdapter`
- `openclaw/plugin-sdk/channel-setup` cubre los constructores de setup con instalación opcional
  más algunas primitivas seguras para setup:
  `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`,
  `createOptionalChannelSetupWizard`, `DEFAULT_ACCOUNT_ID`,
  `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled` y
  `splitSetupEntries`
- usa la unión más amplia `openclaw/plugin-sdk/setup` solo cuando también necesites los
  helpers compartidos más pesados de setup/config como
  `moveSingleAccountChannelSectionToDefaultAccount(...)`

Si tu canal solo quiere anunciar “instala primero este plugin” en superficies
de setup, prefiere `createOptionalChannelSetupSurface(...)`. El
adaptador/asistente generado falla de forma cerrada en escrituras de configuración y finalización, y reutiliza el mismo mensaje de instalación requerida en la validación, finalización y texto con enlace a la documentación.

Para otras rutas activas del canal, prefiere los helpers estrechos en lugar de superficies heredadas más amplias:

- `openclaw/plugin-sdk/account-core`,
  `openclaw/plugin-sdk/account-id`,
  `openclaw/plugin-sdk/account-resolution` y
  `openclaw/plugin-sdk/account-helpers` para configuración de varias cuentas y
  fallback de cuenta predeterminada
- `openclaw/plugin-sdk/inbound-envelope` y
  `openclaw/plugin-sdk/inbound-reply-dispatch` para cableado de ruta/sobre entrante y
  record-and-dispatch
- `openclaw/plugin-sdk/messaging-targets` para análisis/coincidencia de destinos
- `openclaw/plugin-sdk/outbound-media` y
  `openclaw/plugin-sdk/outbound-runtime` para carga de medios más delegados de
  identidad/envío saliente
- `openclaw/plugin-sdk/thread-bindings-runtime` para el ciclo de vida de thread-binding
  y registro del adaptador
- `openclaw/plugin-sdk/agent-media-payload` solo cuando todavía se requiera un diseño heredado de
  campos de carga de agente/medios
- `openclaw/plugin-sdk/telegram-command-config` para normalización de comandos personalizados de Telegram,
  validación de duplicados/conflictos y un contrato de configuración de comandos estable ante fallback

Los canales solo de autenticación normalmente pueden quedarse en la ruta predeterminada: el núcleo gestiona las aprobaciones y el plugin solo expone capacidades de salida/autenticación. Los canales de aprobación nativa como Matrix, Slack, Telegram y transportes de chat personalizados deben usar los helpers nativos compartidos en lugar de implementar su propio ciclo de vida de aprobación.

## Recorrido guiado

<Steps>
  <a id="step-1-package-and-manifest"></a>
  <Step title="Paquete y manifiesto">
    Crea los archivos estándar del plugin. El campo `channel` en `package.json` es
    lo que convierte esto en un plugin de canal. Para ver la superficie completa de metadatos del paquete,
    consulta [Plugin Setup and Config](/plugins/sdk-setup#openclawchannel):

    <CodeGroup>
    ```json package.json
    {
      "name": "@myorg/openclaw-acme-chat",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "setupEntry": "./setup-entry.ts",
        "channel": {
          "id": "acme-chat",
          "label": "Acme Chat",
          "blurb": "Connect OpenClaw to Acme Chat."
        }
      }
    }
    ```

    ```json openclaw.plugin.json
    {
      "id": "acme-chat",
      "kind": "channel",
      "channels": ["acme-chat"],
      "name": "Acme Chat",
      "description": "Acme Chat channel plugin",
      "configSchema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "acme-chat": {
            "type": "object",
            "properties": {
              "token": { "type": "string" },
              "allowFrom": {
                "type": "array",
                "items": { "type": "string" }
              }
            }
          }
        }
      }
    }
    ```
    </CodeGroup>

  </Step>

  <Step title="Construir el objeto del plugin de canal">
    La interfaz `ChannelPlugin` tiene muchas superficies de adaptador opcionales. Empieza con
    lo mínimo — `id` y `setup` — y añade adaptadores según los necesites.

    Crea `src/channel.ts`:

    ```typescript src/channel.ts
    import {
      createChatChannelPlugin,
      createChannelPluginBase,
    } from "openclaw/plugin-sdk/channel-core";
    import type { OpenClawConfig } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatApi } from "./client.js"; // tu cliente API de la plataforma

    type ResolvedAccount = {
      accountId: string | null;
      token: string;
      allowFrom: string[];
      dmPolicy: string | undefined;
    };

    function resolveAccount(
      cfg: OpenClawConfig,
      accountId?: string | null,
    ): ResolvedAccount {
      const section = (cfg.channels as Record<string, any>)?.["acme-chat"];
      const token = section?.token;
      if (!token) throw new Error("acme-chat: token is required");
      return {
        accountId: accountId ?? null,
        token,
        allowFrom: section?.allowFrom ?? [],
        dmPolicy: section?.dmSecurity,
      };
    }

    export const acmeChatPlugin = createChatChannelPlugin<ResolvedAccount>({
      base: createChannelPluginBase({
        id: "acme-chat",
        setup: {
          resolveAccount,
          inspectAccount(cfg, accountId) {
            const section =
              (cfg.channels as Record<string, any>)?.["acme-chat"];
            return {
              enabled: Boolean(section?.token),
              configured: Boolean(section?.token),
              tokenStatus: section?.token ? "available" : "missing",
            };
          },
        },
      }),

      // Seguridad DM: quién puede enviar mensajes al bot
      security: {
        dm: {
          channelKey: "acme-chat",
          resolvePolicy: (account) => account.dmPolicy,
          resolveAllowFrom: (account) => account.allowFrom,
          defaultPolicy: "allowlist",
        },
      },

      // Emparejamiento: flujo de aprobación para nuevos contactos DM
      pairing: {
        text: {
          idLabel: "Acme Chat username",
          message: "Send this code to verify your identity:",
          notify: async ({ target, code }) => {
            await acmeChatApi.sendDm(target, `Pairing code: ${code}`);
          },
        },
      },

      // Encadenamiento: cómo se entregan las respuestas
      threading: { topLevelReplyToMode: "reply" },

      // Salida: enviar mensajes a la plataforma
      outbound: {
        attachedResults: {
          sendText: async (params) => {
            const result = await acmeChatApi.sendMessage(
              params.to,
              params.text,
            );
            return { messageId: result.id };
          },
        },
        base: {
          sendMedia: async (params) => {
            await acmeChatApi.sendFile(params.to, params.filePath);
          },
        },
      },
    });
    ```

    <Accordion title="Qué hace createChatChannelPlugin por ti">
      En lugar de implementar manualmente interfaces de adaptador de bajo nivel, pasas
      opciones declarativas y el constructor las compone:

      | Opción | Qué conecta |
      | --- | --- |
      | `security.dm` | Resolutor de seguridad DM delimitado a partir de campos de configuración |
      | `pairing.text` | Flujo de emparejamiento DM basado en texto con intercambio de código |
      | `threading` | Resolutor de modo reply-to (fijo, delimitado por cuenta o personalizado) |
      | `outbound.attachedResults` | Funciones de envío que devuelven metadatos de resultado (IDs de mensaje) |

      También puedes pasar objetos de adaptador sin procesar en lugar de las opciones declarativas
      si necesitas control total.
    </Accordion>

  </Step>

  <Step title="Conectar el entrypoint">
    Crea `index.ts`:

    ```typescript index.ts
    import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineChannelPluginEntry({
      id: "acme-chat",
      name: "Acme Chat",
      description: "Acme Chat channel plugin",
      plugin: acmeChatPlugin,
      registerCliMetadata(api) {
        api.registerCli(
          ({ program }) => {
            program
              .command("acme-chat")
              .description("Acme Chat management");
          },
          {
            descriptors: [
              {
                name: "acme-chat",
                description: "Acme Chat management",
                hasSubcommands: false,
              },
            ],
          },
        );
      },
      registerFull(api) {
        api.registerGatewayMethod(/* ... */);
      },
    });
    ```

    Coloca los descriptores CLI propiedad del canal en `registerCliMetadata(...)` para que OpenClaw
    pueda mostrarlos en la ayuda raíz sin activar el runtime completo del canal,
    mientras que las cargas completas normales seguirán recogiendo los mismos descriptores para el registro real de comandos. Mantén `registerFull(...)` para trabajo exclusivo de runtime.
    Si `registerFull(...)` registra métodos RPC de gateway, usa un
    prefijo específico del plugin. Los espacios de nombres reservados del núcleo para administración (`config.*`,
    `exec.approvals.*`, `wizard.*`, `update.*`) siguen reservados y siempre
    se resuelven a `operator.admin`.
    `defineChannelPluginEntry` gestiona automáticamente la división entre modos de registro. Consulta
    [Entry Points](/plugins/sdk-entrypoints#definechannelpluginentry) para ver todas las
    opciones.

  </Step>

  <Step title="Añadir una entrada de setup">
    Crea `setup-entry.ts` para una carga ligera durante el onboarding:

    ```typescript setup-entry.ts
    import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineSetupPluginEntry(acmeChatPlugin);
    ```

    OpenClaw carga esto en lugar de la entrada completa cuando el canal está deshabilitado
    o no configurado. Evita arrastrar código pesado de runtime durante los flujos de setup.
    Consulta [Setup and Config](/plugins/sdk-setup#setup-entry) para más detalles.

  </Step>

  <Step title="Gestionar mensajes entrantes">
    Tu plugin necesita recibir mensajes desde la plataforma y reenviarlos a
    OpenClaw. El patrón típico es un webhook que verifica la solicitud y
    la despacha a través del controlador entrante de tu canal:

    ```typescript
    registerFull(api) {
      api.registerHttpRoute({
        path: "/acme-chat/webhook",
        auth: "plugin", // autenticación gestionada por plugin (verifica tú mismo las firmas)
        handler: async (req, res) => {
          const event = parseWebhookPayload(req);

          // Tu controlador entrante despacha el mensaje a OpenClaw.
          // El cableado exacto depende del SDK de tu plataforma:
          // consulta un ejemplo real en el paquete de plugin integrado de Microsoft Teams o Google Chat.
          await handleAcmeChatInbound(api, event);

          res.statusCode = 200;
          res.end("ok");
          return true;
        },
      });
    }
    ```

    <Note>
      El manejo de mensajes entrantes es específico de cada canal. Cada plugin de canal es propietario
      de su propio pipeline de entrada. Mira los plugins de canal integrados
      (por ejemplo el paquete de plugin de Microsoft Teams o Google Chat) para ver patrones reales.
    </Note>

  </Step>

<a id="step-6-test"></a>
<Step title="Probar">
Escribe pruebas colocadas junto al código en `src/channel.test.ts`:

    ```typescript src/channel.test.ts
    import { describe, it, expect } from "vitest";
    import { acmeChatPlugin } from "./channel.js";

    describe("acme-chat plugin", () => {
      it("resolves account from config", () => {
        const cfg = {
          channels: {
            "acme-chat": { token: "test-token", allowFrom: ["user1"] },
          },
        } as any;
        const account = acmeChatPlugin.setup!.resolveAccount(cfg, undefined);
        expect(account.token).toBe("test-token");
      });

      it("inspects account without materializing secrets", () => {
        const cfg = {
          channels: { "acme-chat": { token: "test-token" } },
        } as any;
        const result = acmeChatPlugin.setup!.inspectAccount!(cfg, undefined);
        expect(result.configured).toBe(true);
        expect(result.tokenStatus).toBe("available");
      });

      it("reports missing config", () => {
        const cfg = { channels: {} } as any;
        const result = acmeChatPlugin.setup!.inspectAccount!(cfg, undefined);
        expect(result.configured).toBe(false);
      });
    });
    ```

    ```bash
    pnpm test -- <bundled-plugin-root>/acme-chat/
    ```

    Para helpers de prueba compartidos, consulta [Testing](/plugins/sdk-testing).

  </Step>
</Steps>

## Estructura de archivos

```
<bundled-plugin-root>/acme-chat/
├── package.json              # metadatos openclaw.channel
├── openclaw.plugin.json      # manifiesto con esquema de configuración
├── index.ts                  # defineChannelPluginEntry
├── setup-entry.ts            # defineSetupPluginEntry
├── api.ts                    # exports públicos (opcional)
├── runtime-api.ts            # exports internos de runtime (opcional)
└── src/
    ├── channel.ts            # ChannelPlugin mediante createChatChannelPlugin
    ├── channel.test.ts       # pruebas
    ├── client.ts             # cliente API de la plataforma
    └── runtime.ts            # almacén de runtime (si hace falta)
```

## Temas avanzados

<CardGroup cols={2}>
  <Card title="Opciones de threading" icon="git-branch" href="/plugins/sdk-entrypoints#registration-mode">
    Modos de respuesta fijos, delimitados por cuenta o personalizados
  </Card>
  <Card title="Integración de la herramienta message" icon="puzzle" href="/plugins/architecture#channel-plugins-and-the-shared-message-tool">
    describeMessageTool y descubrimiento de acciones
  </Card>
  <Card title="Resolución de destinos" icon="crosshair" href="/plugins/architecture#channel-target-resolution">
    inferTargetChatType, looksLikeId, resolveTarget
  </Card>
  <Card title="Helpers de runtime" icon="settings" href="/plugins/sdk-runtime">
    TTS, STT, medios, subagente mediante api.runtime
  </Card>
</CardGroup>

<Note>
Algunas uniones helper integradas siguen existiendo para mantenimiento y
compatibilidad de plugins integrados. No son el patrón recomendado para plugins de canal nuevos;
prefiere las subrutas genéricas de channel/setup/reply/runtime de la superficie común del SDK
salvo que estés manteniendo directamente esa familia de plugins integrados.
</Note>

## Siguientes pasos

- [Provider Plugins](/plugins/sdk-provider-plugins) — si tu plugin también proporciona modelos
- [SDK Overview](/plugins/sdk-overview) — referencia completa de importaciones por subruta
- [SDK Testing](/plugins/sdk-testing) — utilidades de prueba y pruebas de contrato
- [Plugin Manifest](/plugins/manifest) — esquema completo del manifiesto
