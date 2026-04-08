---
read_when:
    - Estás creando un nuevo plugin de canal de mensajería
    - Quieres conectar OpenClaw a una plataforma de mensajería
    - Necesitas comprender la superficie del adaptador ChannelPlugin
sidebarTitle: Channel Plugins
summary: Guía paso a paso para crear un plugin de canal de mensajería para OpenClaw
title: Creación de plugins de canal
x-i18n:
    generated_at: "2026-04-08T02:17:04Z"
    model: gpt-5.4
    provider: openai
    source_hash: d23365b6d92006b30e671f9f0afdba40a2b88c845c5d2299d71c52a52985672f
    source_path: plugins/sdk-channel-plugins.md
    workflow: 15
---

# Creación de plugins de canal

Esta guía explica cómo crear un plugin de canal que conecte OpenClaw con una
plataforma de mensajería. Al final tendrás un canal funcional con seguridad de mensajes directos,
emparejamiento, encadenamiento de respuestas y mensajería saliente.

<Info>
  Si todavía no has creado ningún plugin de OpenClaw, lee primero
  [Getting Started](/es/plugins/building-plugins) para conocer la estructura básica
  del paquete y la configuración del manifiesto.
</Info>

## Cómo funcionan los plugins de canal

Los plugins de canal no necesitan sus propias herramientas de enviar/editar/reaccionar. OpenClaw mantiene una
herramienta compartida `message` en el núcleo. Tu plugin se encarga de:

- **Configuración** — resolución de cuentas y asistente de configuración
- **Seguridad** — política de mensajes directos y listas permitidas
- **Emparejamiento** — flujo de aprobación de mensajes directos
- **Gramática de sesión** — cómo los identificadores de conversación específicos del proveedor se asignan a chats base, identificadores de hilo y alternativas de padre
- **Salida** — envío de texto, medios y encuestas a la plataforma
- **Encadenamiento** — cómo se encadenan las respuestas

El núcleo se encarga de la herramienta compartida de mensajes, el cableado de prompts, la forma externa de la clave de sesión,
la contabilidad genérica de `:thread:` y el despacho.

Si tu plataforma almacena alcance adicional dentro de los identificadores de conversación, mantén ese análisis
en el plugin con `messaging.resolveSessionConversation(...)`. Ese es el hook
canónico para asignar `rawId` al identificador de conversación base, un identificador de hilo opcional,
`baseConversationId` explícito y cualquier `parentConversationCandidates`.
Cuando devuelvas `parentConversationCandidates`, mantenlos ordenados desde el
padre más específico hasta la conversación base/más amplia.

Los plugins integrados que necesitan el mismo análisis antes de que arranque
el registro de canales también pueden exponer un archivo de nivel superior `session-key-api.ts` con una exportación
`resolveSessionConversation(...)` equivalente. El núcleo usa esa superficie segura para el arranque
solo cuando el registro de plugins en tiempo de ejecución todavía no está disponible.

`messaging.resolveParentConversationCandidates(...)` sigue disponible como una
alternativa de compatibilidad heredada cuando un plugin solo necesita padres alternativos sobre
el identificador genérico/sin procesar. Si existen ambos hooks, el núcleo usa primero
`resolveSessionConversation(...).parentConversationCandidates` y solo
recurre a `resolveParentConversationCandidates(...)` cuando el hook canónico
los omite.

## Aprobaciones y capacidades del canal

La mayoría de los plugins de canal no necesitan código específico para aprobaciones.

- El núcleo se encarga de `/approve` en el mismo chat, las cargas útiles compartidas de botones de aprobación y la entrega genérica de fallback.
- Prefiere un único objeto `approvalCapability` en el plugin de canal cuando el canal necesita comportamiento específico de aprobación.
- `ChannelPlugin.approvals` se eliminó. Coloca los datos de entrega/aprobación nativa/renderizado/autenticación en `approvalCapability`.
- `plugin.auth` es solo para login/logout; el núcleo ya no lee hooks de autenticación de aprobación desde ese objeto.
- `approvalCapability.authorizeActorAction` y `approvalCapability.getActionAvailabilityState` son la separación canónica para la autenticación de aprobación.
- Usa `approvalCapability.getActionAvailabilityState` para la disponibilidad de autenticación de aprobación en el mismo chat.
- Si tu canal expone aprobaciones nativas de ejecución, usa `approvalCapability.getExecInitiatingSurfaceState` para el estado de la superficie iniciadora/cliente nativo cuando difiera de la autenticación de aprobación en el mismo chat. El núcleo usa ese hook específico de ejecución para distinguir `enabled` frente a `disabled`, decidir si el canal iniciador admite aprobaciones nativas de ejecución e incluir el canal en la orientación de fallback del cliente nativo. `createApproverRestrictedNativeApprovalCapability(...)` completa esto en el caso común.
- Usa `outbound.shouldSuppressLocalPayloadPrompt` o `outbound.beforeDeliverPayload` para comportamiento específico del canal en el ciclo de vida de la carga útil, como ocultar prompts locales duplicados de aprobación o enviar indicadores de escritura antes de la entrega.
- Usa `approvalCapability.delivery` solo para el enrutamiento de aprobación nativa o la supresión de fallback.
- Usa `approvalCapability.nativeRuntime` para los datos nativos de aprobación controlados por el canal. Mantenlo diferido en puntos de entrada de canal críticos con `createLazyChannelApprovalNativeRuntimeAdapter(...)`, que puede importar tu módulo de tiempo de ejecución bajo demanda mientras sigue permitiendo que el núcleo ensamble el ciclo de vida de aprobación.
- Usa `approvalCapability.render` solo cuando un canal realmente necesite cargas útiles de aprobación personalizadas en lugar del renderizador compartido.
- Usa `approvalCapability.describeExecApprovalSetup` cuando el canal quiera que la respuesta de la ruta deshabilitada explique las opciones exactas de configuración necesarias para habilitar aprobaciones nativas de ejecución. El hook recibe `{ channel, channelLabel, accountId }`; los canales con cuentas con nombre deben renderizar rutas con alcance por cuenta como `channels.<channel>.accounts.<id>.execApprovals.*` en lugar de valores predeterminados de nivel superior.
- Si un canal puede inferir identidades estables de tipo propietario en mensajes directos a partir de configuración existente, usa `createResolvedApproverActionAuthAdapter` de `openclaw/plugin-sdk/approval-runtime` para restringir `/approve` en el mismo chat sin añadir lógica específica de aprobación al núcleo.
- Si un canal necesita entrega de aprobación nativa, mantén el código del canal centrado en la normalización del destino más los datos de transporte/presentación. Usa `createChannelExecApprovalProfile`, `createChannelNativeOriginTargetResolver`, `createChannelApproverDmTargetResolver` y `createApproverRestrictedNativeApprovalCapability` de `openclaw/plugin-sdk/approval-runtime`. Coloca los datos específicos del canal detrás de `approvalCapability.nativeRuntime`, idealmente mediante `createChannelApprovalNativeRuntimeAdapter(...)` o `createLazyChannelApprovalNativeRuntimeAdapter(...)`, para que el núcleo pueda ensamblar el controlador y encargarse del filtrado de solicitudes, enrutamiento, eliminación de duplicados, expiración, suscripción a la gateway y avisos de “enrutado a otro lugar”. `nativeRuntime` se divide en unas pocas separaciones más pequeñas:
- `availability` — si la cuenta está configurada y si una solicitud debe gestionarse
- `presentation` — asignar el modelo de vista compartido de aprobación a cargas útiles nativas pendientes/resueltas/expiradas o acciones finales
- `transport` — preparar destinos más enviar/actualizar/eliminar mensajes de aprobación nativa
- `interactions` — hooks opcionales para enlazar/desenlazar/borrar acciones en botones o reacciones nativas
- `observe` — hooks opcionales para diagnósticos de entrega
- Si el canal necesita objetos controlados por el tiempo de ejecución como un cliente, token, app Bolt o receptor de webhook, regístralos mediante `openclaw/plugin-sdk/channel-runtime-context`. El registro genérico de contexto de tiempo de ejecución permite al núcleo iniciar controladores orientados por capacidades a partir del estado de inicio del canal sin añadir código de unión específico de aprobación.
- Recurre a `createChannelApprovalHandler` o `createChannelNativeApprovalRuntime` de nivel inferior solo cuando la separación orientada por capacidades todavía no sea lo suficientemente expresiva.
- Los canales de aprobación nativa deben enrutar tanto `accountId` como `approvalKind` a través de esos helpers. `accountId` mantiene la política de aprobación multi-cuenta limitada a la cuenta correcta del bot, y `approvalKind` mantiene disponible para el canal el comportamiento de aprobación de ejecución frente al de plugin sin ramas codificadas en el núcleo.
- El núcleo ahora también se encarga de los avisos de redirección de aprobación. Los plugins de canal no deben enviar sus propios mensajes de seguimiento de “la aprobación fue a mensajes directos / a otro canal” desde `createChannelNativeApprovalRuntime`; en su lugar, expón el enrutamiento preciso de origen + mensaje directo del aprobador mediante los helpers compartidos de capacidad de aprobación y deja que el núcleo agregue las entregas reales antes de publicar cualquier aviso de vuelta en el chat iniciador.
- Conserva el tipo de identificador de aprobación entregado de extremo a extremo. Los clientes nativos no deben
  adivinar ni reescribir el enrutamiento de aprobación de ejecución frente a plugin a partir del estado local del canal.
- Distintos tipos de aprobación pueden exponer intencionalmente distintas superficies nativas.
  Ejemplos integrados actuales:
  - Slack mantiene disponible el enrutamiento de aprobación nativa tanto para identificadores de ejecución como de plugin.
  - Matrix mantiene el mismo enrutamiento nativo de mensajes directos/canal y la misma UX basada en reacciones para aprobaciones de ejecución
    y de plugin, aunque sigue permitiendo que la autenticación difiera según el tipo de aprobación.
- `createApproverRestrictedNativeApprovalAdapter` sigue existiendo como envoltorio de compatibilidad, pero el código nuevo debe preferir el constructor de capacidades y exponer `approvalCapability` en el plugin.

Para puntos de entrada de canal críticos, prefiere las subrutas de tiempo de ejecución más reducidas cuando solo
necesites una parte de esa familia:

- `openclaw/plugin-sdk/approval-auth-runtime`
- `openclaw/plugin-sdk/approval-client-runtime`
- `openclaw/plugin-sdk/approval-delivery-runtime`
- `openclaw/plugin-sdk/approval-gateway-runtime`
- `openclaw/plugin-sdk/approval-handler-adapter-runtime`
- `openclaw/plugin-sdk/approval-handler-runtime`
- `openclaw/plugin-sdk/approval-native-runtime`
- `openclaw/plugin-sdk/approval-reply-runtime`
- `openclaw/plugin-sdk/channel-runtime-context`

Del mismo modo, prefiere `openclaw/plugin-sdk/setup-runtime`,
`openclaw/plugin-sdk/setup-adapter-runtime`,
`openclaw/plugin-sdk/reply-runtime`,
`openclaw/plugin-sdk/reply-dispatch-runtime`,
`openclaw/plugin-sdk/reply-reference` y
`openclaw/plugin-sdk/reply-chunking` cuando no necesites la superficie paraguas
más amplia.

Para la configuración en concreto:

- `openclaw/plugin-sdk/setup-runtime` cubre los helpers de configuración seguros para tiempo de ejecución:
  adaptadores de parcheo de configuración seguros para importación (`createPatchedAccountSetupAdapter`,
  `createEnvPatchedAccountSetupAdapter`,
  `createSetupInputPresenceValidator`), salida de notas de búsqueda,
  `promptResolvedAllowFrom`, `splitSetupEntries` y los constructores
  de proxy de configuración delegada
- `openclaw/plugin-sdk/setup-adapter-runtime` es la separación reducida de adaptador
  con reconocimiento del entorno para `createEnvPatchedAccountSetupAdapter`
- `openclaw/plugin-sdk/channel-setup` cubre los constructores de configuración
  de instalación opcional más algunas primitivas seguras para configuración:
  `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`,

Si tu canal admite configuración o autenticación controladas por variables de entorno y los
flujos genéricos de arranque/configuración deben conocer esos nombres de variables de entorno antes de que se cargue el tiempo de ejecución, decláralos en el
manifiesto del plugin con `channelEnvVars`. Mantén `envVars` del tiempo de ejecución del canal o las constantes locales solo para el texto orientado a operadores.
`createOptionalChannelSetupWizard`, `DEFAULT_ACCOUNT_ID`,
`createTopLevelChannelDmPolicy`, `setSetupChannelEnabled` y
`splitSetupEntries`

- usa la separación más amplia `openclaw/plugin-sdk/setup` solo cuando también necesites los
  helpers compartidos más pesados de configuración/configuración, como
  `moveSingleAccountChannelSectionToDefaultAccount(...)`

Si tu canal solo quiere anunciar “instala este plugin primero” en superficies de configuración, prefiere `createOptionalChannelSetupSurface(...)`. El adaptador/asistente generado falla de forma cerrada en escrituras de configuración y finalización, y reutiliza el mismo mensaje de instalación requerida en la validación, finalización y texto de enlace a documentación.

Para otras rutas críticas del canal, prefiere los helpers reducidos frente a superficies heredadas más amplias:

- `openclaw/plugin-sdk/account-core`,
  `openclaw/plugin-sdk/account-id`,
  `openclaw/plugin-sdk/account-resolution` y
  `openclaw/plugin-sdk/account-helpers` para configuración multi-cuenta y
  fallback de cuenta predeterminada
- `openclaw/plugin-sdk/inbound-envelope` y
  `openclaw/plugin-sdk/inbound-reply-dispatch` para el cableado de rutas/sobres de entrada y
  registro y despacho
- `openclaw/plugin-sdk/messaging-targets` para análisis/coincidencia de destinos
- `openclaw/plugin-sdk/outbound-media` y
  `openclaw/plugin-sdk/outbound-runtime` para carga de medios más delegados de identidad/envío
  salientes
- `openclaw/plugin-sdk/thread-bindings-runtime` para el ciclo de vida de enlace de hilos
  y registro de adaptadores
- `openclaw/plugin-sdk/agent-media-payload` solo cuando todavía se requiera una disposición heredada
  de campos de carga útil de agente/medios
- `openclaw/plugin-sdk/telegram-command-config` para normalización de comandos personalizados de Telegram,
  validación de duplicados/conflictos y un contrato estable de configuración de comandos
  para fallback

Los canales solo de autenticación normalmente pueden quedarse en la ruta predeterminada: el núcleo gestiona las aprobaciones y el plugin solo expone capacidades de salida/autenticación. Los canales de aprobación nativa como Matrix, Slack, Telegram y transportes de chat personalizados deben usar los helpers nativos compartidos en lugar de crear su propio ciclo de vida de aprobación.

## Política de menciones entrantes

Mantén el manejo de menciones entrantes dividido en dos capas:

- recopilación de evidencia controlada por el plugin
- evaluación de políticas compartida

Usa `openclaw/plugin-sdk/channel-inbound` para la capa compartida.

Buena opción para la lógica local del plugin:

- detección de respuesta al bot
- detección de cita del bot
- comprobaciones de participación en el hilo
- exclusiones de mensajes de servicio/sistema
- cachés nativas de la plataforma necesarias para demostrar la participación del bot

Buena opción para el helper compartido:

- `requireMention`
- resultado explícito de mención
- lista permitida de mención implícita
- bypass de comandos
- decisión final de omitir

Flujo recomendado:

1. Calcula los datos locales de mención.
2. Pasa esos datos a `resolveInboundMentionDecision({ facts, policy })`.
3. Usa `decision.effectiveWasMentioned`, `decision.shouldBypassMention` y `decision.shouldSkip` en tu puerta de entrada.

```typescript
import {
  implicitMentionKindWhen,
  matchesMentionWithExplicit,
  resolveInboundMentionDecision,
} from "openclaw/plugin-sdk/channel-inbound";

const mentionMatch = matchesMentionWithExplicit(text, {
  mentionRegexes,
  mentionPatterns,
});

const facts = {
  canDetectMention: true,
  wasMentioned: mentionMatch.matched,
  hasAnyMention: mentionMatch.hasExplicitMention,
  implicitMentionKinds: [
    ...implicitMentionKindWhen("reply_to_bot", isReplyToBot),
    ...implicitMentionKindWhen("quoted_bot", isQuoteOfBot),
  ],
};

const decision = resolveInboundMentionDecision({
  facts,
  policy: {
    isGroup,
    requireMention,
    allowedImplicitMentionKinds: requireExplicitMention ? [] : ["reply_to_bot", "quoted_bot"],
    allowTextCommands,
    hasControlCommand,
    commandAuthorized,
  },
});

if (decision.shouldSkip) return;
```

`api.runtime.channel.mentions` expone los mismos helpers compartidos de menciones para
plugins de canal integrados que ya dependen de la inyección en tiempo de ejecución:

- `buildMentionRegexes`
- `matchesMentionPatterns`
- `matchesMentionWithExplicit`
- `implicitMentionKindWhen`
- `resolveInboundMentionDecision`

Los antiguos helpers `resolveMentionGating*` siguen en
`openclaw/plugin-sdk/channel-inbound` solo como exportaciones de compatibilidad. El código nuevo
debe usar `resolveInboundMentionDecision({ facts, policy })`.

## Recorrido

<Steps>
  <a id="step-1-package-and-manifest"></a>
  <Step title="Paquete y manifiesto">
    Crea los archivos estándar del plugin. El campo `channel` en `package.json` es
    lo que convierte esto en un plugin de canal. Para la superficie completa de metadatos del paquete,
    consulta [Plugin Setup and Config](/es/plugins/sdk-setup#openclawchannel):

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

  <Step title="Construye el objeto del plugin de canal">
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

      // Seguridad de mensajes directos: quién puede enviar mensajes al bot
      security: {
        dm: {
          channelKey: "acme-chat",
          resolvePolicy: (account) => account.dmPolicy,
          resolveAllowFrom: (account) => account.allowFrom,
          defaultPolicy: "allowlist",
        },
      },

      // Emparejamiento: flujo de aprobación para nuevos contactos por mensaje directo
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

    <Accordion title="Qué hace por ti createChatChannelPlugin">
      En lugar de implementar manualmente interfaces de adaptador de bajo nivel, pasas
      opciones declarativas y el constructor las compone:

      | Opción | Lo que conecta |
      | --- | --- |
      | `security.dm` | Resolución de seguridad de mensajes directos con alcance desde los campos de configuración |
      | `pairing.text` | Flujo de emparejamiento por mensaje directo basado en texto con intercambio de código |
      | `threading` | Resolución del modo reply-to (fijo, con alcance por cuenta o personalizado) |
      | `outbound.attachedResults` | Funciones de envío que devuelven metadatos de resultado (identificadores de mensaje) |

      También puedes pasar objetos de adaptador sin procesar en lugar de opciones declarativas
      si necesitas control total.
    </Accordion>

  </Step>

  <Step title="Conecta el punto de entrada">
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

    Coloca los descriptores de CLI controlados por el canal en `registerCliMetadata(...)` para que OpenClaw
    pueda mostrarlos en la ayuda raíz sin activar el tiempo de ejecución completo del canal,
    mientras que las cargas completas normales sigan recogiendo esos mismos descriptores para el registro real de comandos.
    Mantén `registerFull(...)` para trabajo solo de tiempo de ejecución.
    Si `registerFull(...)` registra métodos RPC de gateway, usa un prefijo
    específico del plugin. Los espacios de nombres administrativos del núcleo (`config.*`,
    `exec.approvals.*`, `wizard.*`, `update.*`) permanecen reservados y siempre
    se resuelven en `operator.admin`.
    `defineChannelPluginEntry` gestiona automáticamente la división por modo de registro. Consulta
    [Entry Points](/es/plugins/sdk-entrypoints#definechannelpluginentry) para ver todas las
    opciones.

  </Step>

  <Step title="Añade una entrada de configuración">
    Crea `setup-entry.ts` para carga ligera durante la incorporación:

    ```typescript setup-entry.ts
    import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineSetupPluginEntry(acmeChatPlugin);
    ```

    OpenClaw carga esto en lugar de la entrada completa cuando el canal está deshabilitado
    o no configurado. Evita cargar código pesado de tiempo de ejecución durante los flujos de configuración.
    Consulta [Setup and Config](/es/plugins/sdk-setup#setup-entry) para más detalles.

  </Step>

  <Step title="Gestiona los mensajes entrantes">
    Tu plugin necesita recibir mensajes de la plataforma y reenviarlos a
    OpenClaw. El patrón típico es un webhook que verifica la solicitud y
    la despacha mediante el controlador de entrada de tu canal:

    ```typescript
    registerFull(api) {
      api.registerHttpRoute({
        path: "/acme-chat/webhook",
        auth: "plugin", // autenticación gestionada por el plugin (verifica tú mismo las firmas)
        handler: async (req, res) => {
          const event = parseWebhookPayload(req);

          // Tu controlador de entrada despacha el mensaje a OpenClaw.
          // El cableado exacto depende del SDK de tu plataforma —
          // consulta un ejemplo real en el paquete integrado del plugin de Microsoft Teams o Google Chat.
          await handleAcmeChatInbound(api, event);

          res.statusCode = 200;
          res.end("ok");
          return true;
        },
      });
    }
    ```

    <Note>
      El manejo de mensajes entrantes es específico del canal. Cada plugin de canal se encarga de
      su propia canalización de entrada. Mira los plugins de canal integrados
      (por ejemplo, el paquete del plugin de Microsoft Teams o Google Chat) para ver patrones reales.
    </Note>

  </Step>

<a id="step-6-test"></a>
<Step title="Prueba">
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

    Para helpers de prueba compartidos, consulta [Testing](/es/plugins/sdk-testing).

  </Step>
</Steps>

## Estructura de archivos

```
<bundled-plugin-root>/acme-chat/
├── package.json              # metadatos openclaw.channel
├── openclaw.plugin.json      # Manifiesto con esquema de configuración
├── index.ts                  # defineChannelPluginEntry
├── setup-entry.ts            # defineSetupPluginEntry
├── api.ts                    # Exportaciones públicas (opcional)
├── runtime-api.ts            # Exportaciones internas de tiempo de ejecución (opcional)
└── src/
    ├── channel.ts            # ChannelPlugin mediante createChatChannelPlugin
    ├── channel.test.ts       # Pruebas
    ├── client.ts             # Cliente API de la plataforma
    └── runtime.ts            # Almacén de tiempo de ejecución (si es necesario)
```

## Temas avanzados

<CardGroup cols={2}>
  <Card title="Opciones de encadenamiento" icon="git-branch" href="/es/plugins/sdk-entrypoints#registration-mode">
    Modos de respuesta fijos, con alcance por cuenta o personalizados
  </Card>
  <Card title="Integración de la herramienta de mensajes" icon="puzzle" href="/es/plugins/architecture#channel-plugins-and-the-shared-message-tool">
    describeMessageTool y descubrimiento de acciones
  </Card>
  <Card title="Resolución de destino" icon="crosshair" href="/es/plugins/architecture#channel-target-resolution">
    inferTargetChatType, looksLikeId, resolveTarget
  </Card>
  <Card title="Helpers de tiempo de ejecución" icon="settings" href="/es/plugins/sdk-runtime">
    TTS, STT, medios, subagente mediante api.runtime
  </Card>
</CardGroup>

<Note>
Algunas separaciones auxiliares integradas siguen existiendo para mantenimiento de plugins integrados y
compatibilidad. No son el patrón recomendado para plugins de canal nuevos;
prefiere las subrutas genéricas de channel/setup/reply/runtime de la superficie común del SDK
a menos que estés manteniendo directamente esa familia de plugins integrados.
</Note>

## Siguientes pasos

- [Provider Plugins](/es/plugins/sdk-provider-plugins) — si tu plugin también proporciona modelos
- [SDK Overview](/es/plugins/sdk-overview) — referencia completa de importación de subrutas
- [SDK Testing](/es/plugins/sdk-testing) — utilidades de prueba y pruebas de contrato
- [Plugin Manifest](/es/plugins/manifest) — esquema completo del manifiesto
