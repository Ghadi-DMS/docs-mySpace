---
read_when:
    - Estás creando un nuevo Plugin de canal de mensajería
    - Quieres conectar OpenClaw a una plataforma de mensajería
    - Necesitas comprender la superficie del adaptador `ChannelPlugin`
sidebarTitle: Channel Plugins
summary: Guía paso a paso para crear un Plugin de canal de mensajería para OpenClaw
title: Creación de Plugins de canal
x-i18n:
    generated_at: "2026-04-18T04:59:37Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3dda53c969bc7356a450c2a5bf49fb82bf1283c23e301dec832d8724b11e724b
    source_path: plugins/sdk-channel-plugins.md
    workflow: 15
---

# Creación de Plugins de canal

Esta guía recorre la creación de un Plugin de canal que conecta OpenClaw con una
plataforma de mensajería. Al final tendrás un canal funcional con seguridad en MD,
emparejamiento, encadenamiento de respuestas y mensajería saliente.

<Info>
  Si todavía no has creado ningún Plugin de OpenClaw, lee primero
  [Primeros pasos](/es/plugins/building-plugins) para conocer la estructura básica
  del paquete y la configuración del manifiesto.
</Info>

## Cómo funcionan los Plugins de canal

Los Plugins de canal no necesitan sus propias herramientas de enviar/editar/reaccionar. OpenClaw mantiene una
única herramienta `message` compartida en el núcleo. Tu Plugin gestiona:

- **Configuración** — resolución de cuentas y asistente de configuración
- **Seguridad** — política de MD y listas de permitidos
- **Emparejamiento** — flujo de aprobación de MD
- **Gramática de sesión** — cómo los id. de conversación específicos del proveedor se asignan a chats base, id. de hilo y alternativas de respaldo de padre
- **Salida** — envío de texto, contenido multimedia y encuestas a la plataforma
- **Encadenamiento** — cómo se encadenan las respuestas

El núcleo gestiona la herramienta compartida de mensajes, la conexión del prompt, la forma externa de la clave de sesión,
la contabilidad genérica de `:thread:` y el despacho.

Si tu canal añade parámetros a la herramienta de mensajes que transportan fuentes multimedia, expón esos
nombres de parámetros mediante `describeMessageTool(...).mediaSourceParams`. El núcleo usa
esa lista explícita para la normalización de rutas del sandbox y la política de acceso a medios salientes,
así que los Plugins no necesitan casos especiales en el núcleo compartido para parámetros específicos del proveedor como
avatar, adjunto o imagen de portada.
Prefiere devolver un mapa por acción como
`{ "set-profile": ["avatarUrl", "avatarPath"] }` para que acciones no relacionadas no
hereden los argumentos multimedia de otra acción. Un arreglo plano sigue funcionando para parámetros que
se comparten intencionalmente entre todas las acciones expuestas.

Si tu plataforma almacena alcance adicional dentro de los id. de conversación, mantén ese análisis
en el Plugin con `messaging.resolveSessionConversation(...)`. Ese es el hook canónico para mapear
`rawId` al id. base de conversación, id. opcional de hilo, `baseConversationId`
explícito y cualquier `parentConversationCandidates`.
Cuando devuelvas `parentConversationCandidates`, mantenlos ordenados desde el
padre más específico hasta la conversación base más amplia.

Los Plugins incluidos que necesitan el mismo análisis antes de que se inicie el registro de canales
también pueden exponer un archivo `session-key-api.ts` de nivel superior con una exportación
`resolveSessionConversation(...)` equivalente. El núcleo usa esa superficie segura para el arranque
solo cuando el registro de Plugins en tiempo de ejecución todavía no está disponible.

`messaging.resolveParentConversationCandidates(...)` sigue disponible como alternativa heredada de compatibilidad
cuando un Plugin solo necesita padres alternativos además del id. genérico/sin procesar. Si existen ambos hooks, el núcleo usa
primero `resolveSessionConversation(...).parentConversationCandidates` y solo
recurre a `resolveParentConversationCandidates(...)` cuando el hook canónico
los omite.

## Aprobaciones y capacidades del canal

La mayoría de los Plugins de canal no necesitan código específico para aprobaciones.

- El núcleo gestiona `/approve` en el mismo chat, las cargas útiles compartidas de botones de aprobación y la entrega genérica de respaldo.
- Prefiere un único objeto `approvalCapability` en el Plugin del canal cuando el canal necesite comportamiento específico de aprobación.
- `ChannelPlugin.approvals` se eliminó. Coloca los datos de entrega/aprobación nativa/renderizado/autenticación en `approvalCapability`.
- `plugin.auth` es solo para inicio y cierre de sesión; el núcleo ya no lee hooks de autenticación de aprobación desde ese objeto.
- `approvalCapability.authorizeActorAction` y `approvalCapability.getActionAvailabilityState` son la separación canónica para la autenticación de aprobaciones.
- Usa `approvalCapability.getActionAvailabilityState` para la disponibilidad de autenticación de aprobaciones en el mismo chat.
- Si tu canal expone aprobaciones nativas de ejecución, usa `approvalCapability.getExecInitiatingSurfaceState` para el estado de la superficie iniciadora/cliente nativo cuando difiera de la autenticación de aprobación en el mismo chat. El núcleo usa ese hook específico de ejecución para distinguir `enabled` de `disabled`, decidir si el canal iniciador admite aprobaciones nativas de ejecución e incluir el canal en la guía de respaldo del cliente nativo. `createApproverRestrictedNativeApprovalCapability(...)` lo completa para el caso común.
- Usa `outbound.shouldSuppressLocalPayloadPrompt` o `outbound.beforeDeliverPayload` para comportamiento específico del canal en el ciclo de vida de la carga útil, como ocultar prompts locales duplicados de aprobación o enviar indicadores de escritura antes de la entrega.
- Usa `approvalCapability.delivery` solo para enrutamiento de aprobaciones nativas o supresión del respaldo.
- Usa `approvalCapability.nativeRuntime` para datos nativos de aprobación gestionados por el canal. Mantenlo diferido en puntos de entrada calientes del canal con `createLazyChannelApprovalNativeRuntimeAdapter(...)`, que puede importar tu módulo de tiempo de ejecución bajo demanda y aun así permitir que el núcleo ensamble el ciclo de vida de la aprobación.
- Usa `approvalCapability.render` solo cuando un canal realmente necesite cargas útiles de aprobación personalizadas en lugar del renderizador compartido.
- Usa `approvalCapability.describeExecApprovalSetup` cuando el canal quiera que la respuesta de ruta deshabilitada explique los parámetros exactos de configuración necesarios para habilitar aprobaciones nativas de ejecución. El hook recibe `{ channel, channelLabel, accountId }`; los canales con cuentas nombradas deben renderizar rutas con alcance por cuenta como `channels.<channel>.accounts.<id>.execApprovals.*` en lugar de valores predeterminados de nivel superior.
- Si un canal puede inferir identidades de MD estables de tipo propietario a partir de la configuración existente, usa `createResolvedApproverActionAuthAdapter` desde `openclaw/plugin-sdk/approval-runtime` para restringir `/approve` en el mismo chat sin añadir lógica de aprobación específica al núcleo.
- Si un canal necesita entrega de aprobación nativa, mantén el código del canal enfocado en la normalización del destino más los datos de transporte/presentación. Usa `createChannelExecApprovalProfile`, `createChannelNativeOriginTargetResolver`, `createChannelApproverDmTargetResolver` y `createApproverRestrictedNativeApprovalCapability` desde `openclaw/plugin-sdk/approval-runtime`. Coloca los datos específicos del canal detrás de `approvalCapability.nativeRuntime`, idealmente mediante `createChannelApprovalNativeRuntimeAdapter(...)` o `createLazyChannelApprovalNativeRuntimeAdapter(...)`, para que el núcleo pueda ensamblar el controlador y gestionar el filtrado de solicitudes, el enrutamiento, la eliminación de duplicados, la expiración, la suscripción al Gateway y los avisos de redirección. `nativeRuntime` se divide en algunas separaciones más pequeñas:
- `availability` — si la cuenta está configurada y si una solicitud debe gestionarse
- `presentation` — asigna el modelo de vista compartido de aprobación a cargas útiles nativas pendientes/resueltas/expiradas o acciones finales
- `transport` — prepara destinos más envío/actualización/eliminación de mensajes nativos de aprobación
- `interactions` — hooks opcionales para enlazar/desenlazar/borrar acciones de botones o reacciones nativas
- `observe` — hooks opcionales de diagnóstico de entrega
- Si el canal necesita objetos gestionados por el tiempo de ejecución como un cliente, token, app Bolt o receptor de Webhook, regístralos mediante `openclaw/plugin-sdk/channel-runtime-context`. El registro genérico de contexto de tiempo de ejecución permite que el núcleo inicialice controladores guiados por capacidades a partir del estado de arranque del canal sin añadir pegamento contenedor específico de aprobación.
- Recurre a `createChannelApprovalHandler` o `createChannelNativeApprovalRuntime` de nivel inferior solo cuando la separación guiada por capacidades todavía no sea lo bastante expresiva.
- Los canales de aprobación nativa deben enrutar tanto `accountId` como `approvalKind` a través de esos helpers. `accountId` mantiene la política de aprobación multicuenta con el alcance correcto para la cuenta del bot, y `approvalKind` mantiene disponible para el canal el comportamiento de aprobación de ejecución frente al de Plugin sin ramas codificadas en el núcleo.
- El núcleo ahora también gestiona los avisos de redirección de aprobación. Los Plugins de canal no deben enviar sus propios mensajes de seguimiento de "la aprobación fue a MD / a otro canal" desde `createChannelNativeApprovalRuntime`; en su lugar, expón un enrutamiento preciso del origen + MD del aprobador mediante los helpers compartidos de capacidad de aprobación y deja que el núcleo agregue las entregas reales antes de publicar cualquier aviso en el chat iniciador.
- Conserva de extremo a extremo el tipo de id. de aprobación entregado. Los clientes nativos no deben
  adivinar ni reescribir el enrutamiento de aprobaciones de ejecución frente a Plugin a partir del estado local del canal.
- Distintos tipos de aprobación pueden exponer intencionalmente distintas superficies nativas.
  Ejemplos actuales incluidos:
  - Slack mantiene disponible el enrutamiento de aprobación nativa tanto para id. de ejecución como de Plugin.
  - Matrix mantiene el mismo enrutamiento nativo de MD/canal y la misma experiencia de reacciones para aprobaciones de ejecución
    y de Plugin, al tiempo que sigue permitiendo que la autenticación difiera según el tipo de aprobación.
- `createApproverRestrictedNativeApprovalAdapter` sigue existiendo como contenedor de compatibilidad, pero el código nuevo debe preferir el constructor de capacidades y exponer `approvalCapability` en el Plugin.

Para puntos de entrada calientes del canal, prefiere las subrutas de tiempo de ejecución más estrechas cuando solo
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
`openclaw/plugin-sdk/reply-chunking` cuando no necesites la
superficie paraguas más amplia.

Para la configuración específicamente:

- `openclaw/plugin-sdk/setup-runtime` cubre los helpers de configuración seguros para tiempo de ejecución:
  adaptadores de parche de configuración seguros para importación (`createPatchedAccountSetupAdapter`,
  `createEnvPatchedAccountSetupAdapter`,
  `createSetupInputPresenceValidator`), salida de notas de búsqueda,
  `promptResolvedAllowFrom`, `splitSetupEntries` y los constructores
  delegados de proxy de configuración
- `openclaw/plugin-sdk/setup-adapter-runtime` es la separación estrecha de adaptador
  consciente del entorno para `createEnvPatchedAccountSetupAdapter`
- `openclaw/plugin-sdk/channel-setup` cubre los constructores de configuración de instalación opcional
  más algunos primitivos seguros para configuración:
  `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`,

Si tu canal admite configuración o autenticación basada en variables de entorno y los flujos genéricos de inicio/configuración
deben conocer esos nombres de variables antes de que cargue el tiempo de ejecución, decláralos en el
manifiesto del Plugin con `channelEnvVars`. Mantén `envVars` del tiempo de ejecución del canal o
constantes locales solo para el texto dirigido al operador.
`createOptionalChannelSetupWizard`, `DEFAULT_ACCOUNT_ID`,
`createTopLevelChannelDmPolicy`, `setSetupChannelEnabled` y
`splitSetupEntries`

- usa la separación más amplia `openclaw/plugin-sdk/setup` solo cuando también necesites los
  helpers compartidos más pesados de configuración/configuración como
  `moveSingleAccountChannelSectionToDefaultAccount(...)`

Si tu canal solo quiere anunciar "instala primero este Plugin" en las
superficies de configuración, prefiere `createOptionalChannelSetupSurface(...)`. El
adaptador/asistente generado falla de forma cerrada en escrituras de configuración y finalización, y reutiliza
el mismo mensaje de instalación requerida en la validación, finalización y texto
de enlace a la documentación.

Para otras rutas calientes del canal, prefiere los helpers estrechos en lugar de las
superficies heredadas más amplias:

- `openclaw/plugin-sdk/account-core`,
  `openclaw/plugin-sdk/account-id`,
  `openclaw/plugin-sdk/account-resolution` y
  `openclaw/plugin-sdk/account-helpers` para configuración multicuenta y
  respaldo de cuenta predeterminada
- `openclaw/plugin-sdk/inbound-envelope` y
  `openclaw/plugin-sdk/inbound-reply-dispatch` para el cableado de ruta/sobre de entrada y
  registro y despacho
- `openclaw/plugin-sdk/messaging-targets` para análisis/coincidencia de destinos
- `openclaw/plugin-sdk/outbound-media` y
  `openclaw/plugin-sdk/outbound-runtime` para carga de medios más delegados de
  identidad/envío salientes
- `openclaw/plugin-sdk/thread-bindings-runtime` para el ciclo de vida de los vínculos de hilo
  y el registro de adaptadores
- `openclaw/plugin-sdk/agent-media-payload` solo cuando todavía se requiera una disposición heredada
  de campos de carga útil de agente/medios
- `openclaw/plugin-sdk/telegram-command-config` para normalización de comandos personalizados de Telegram, validación de duplicados/conflictos y un contrato de configuración de comandos estable para respaldo

Los canales solo de autenticación normalmente pueden detenerse en la ruta predeterminada: el núcleo gestiona las aprobaciones y el Plugin solo expone capacidades de salida/autenticación. Los canales de aprobación nativa como Matrix, Slack, Telegram y transportes de chat personalizados deben usar los helpers nativos compartidos en lugar de crear su propio ciclo de vida de aprobación.

## Política de menciones entrantes

Mantén el manejo de menciones entrantes dividido en dos capas:

- recopilación de evidencias gestionada por el Plugin
- evaluación de políticas compartidas

Usa `openclaw/plugin-sdk/channel-mention-gating` para decisiones de política de menciones.
Usa `openclaw/plugin-sdk/channel-inbound` solo cuando necesites el barrel
más amplio de helpers de entrada.

Buen encaje para lógica local del Plugin:

- detección de respuesta al bot
- detección de cita del bot
- comprobaciones de participación en hilos
- exclusiones de mensajes de servicio/sistema
- cachés nativos de la plataforma necesarios para demostrar la participación del bot

Buen encaje para el helper compartido:

- `requireMention`
- resultado de mención explícita
- lista de permitidos de menciones implícitas
- omisión de comandos
- decisión final de omitir

Flujo preferido:

1. Calcula los datos locales de mención.
2. Pasa esos datos a `resolveInboundMentionDecision({ facts, policy })`.
3. Usa `decision.effectiveWasMentioned`, `decision.shouldBypassMention` y `decision.shouldSkip` en tu compuerta de entrada.

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
los Plugins de canal incluidos que ya dependen de inyección en tiempo de ejecución:

- `buildMentionRegexes`
- `matchesMentionPatterns`
- `matchesMentionWithExplicit`
- `implicitMentionKindWhen`
- `resolveInboundMentionDecision`

Si solo necesitas `implicitMentionKindWhen` y
`resolveInboundMentionDecision`, importa desde
`openclaw/plugin-sdk/channel-mention-gating` para evitar cargar helpers de tiempo de ejecución
de entrada no relacionados.

Los antiguos helpers `resolveMentionGating*` siguen estando en
`openclaw/plugin-sdk/channel-inbound` solo como exportaciones de compatibilidad. El código nuevo
debe usar `resolveInboundMentionDecision({ facts, policy })`.

## Recorrido

<Steps>
  <a id="step-1-package-and-manifest"></a>
  <Step title="Paquete y manifiesto">
    Crea los archivos estándar del Plugin. El campo `channel` en `package.json` es
    lo que convierte esto en un Plugin de canal. Para la superficie completa de metadatos del paquete,
    consulta [Configuración y Setup del Plugin](/es/plugins/sdk-setup#openclaw-channel):

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
          "blurb": "Conecta OpenClaw con Acme Chat."
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
      "description": "Plugin de canal de Acme Chat",
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

  <Step title="Construye el objeto del Plugin de canal">
    La interfaz `ChannelPlugin` tiene muchas superficies opcionales de adaptador. Comienza con
    lo mínimo — `id` y `setup` — y añade adaptadores según los necesites.

    Crea `src/channel.ts`:

    ```typescript src/channel.ts
    import {
      createChatChannelPlugin,
      createChannelPluginBase,
    } from "openclaw/plugin-sdk/channel-core";
    import type { OpenClawConfig } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatApi } from "./client.js"; // your platform API client

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

      // DM security: who can message the bot
      security: {
        dm: {
          channelKey: "acme-chat",
          resolvePolicy: (account) => account.dmPolicy,
          resolveAllowFrom: (account) => account.allowFrom,
          defaultPolicy: "allowlist",
        },
      },

      // Pairing: approval flow for new DM contacts
      pairing: {
        text: {
          idLabel: "Acme Chat username",
          message: "Send this code to verify your identity:",
          notify: async ({ target, code }) => {
            await acmeChatApi.sendDm(target, `Pairing code: ${code}`);
          },
        },
      },

      // Threading: how replies are delivered
      threading: { topLevelReplyToMode: "reply" },

      // Outbound: send messages to the platform
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

    <Accordion title="Qué hace `createChatChannelPlugin` por ti">
      En lugar de implementar manualmente interfaces de adaptador de bajo nivel, pasas
      opciones declarativas y el constructor las compone:

      | Opción | Qué conecta |
      | --- | --- |
      | `security.dm` | Resolutor de seguridad de MD con alcance desde campos de configuración |
      | `pairing.text` | Flujo de emparejamiento por MD basado en texto con intercambio de código |
      | `threading` | Resolutor del modo de respuesta (fijo, con alcance por cuenta o personalizado) |
      | `outbound.attachedResults` | Funciones de envío que devuelven metadatos del resultado (id. de mensaje) |

      También puedes pasar objetos de adaptador sin procesar en lugar de las opciones declarativas
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

    Coloca los descriptores de CLI gestionados por el canal en `registerCliMetadata(...)` para que OpenClaw
    pueda mostrarlos en la ayuda raíz sin activar el tiempo de ejecución completo del canal,
    mientras que las cargas completas normales sigan recogiendo los mismos descriptores para el registro
    real de comandos. Mantén `registerFull(...)` para trabajo exclusivo de tiempo de ejecución.
    Si `registerFull(...)` registra métodos RPC del Gateway, usa un
    prefijo específico del Plugin. Los espacios de nombres administrativos del núcleo (`config.*`,
    `exec.approvals.*`, `wizard.*`, `update.*`) permanecen reservados y siempre
    se resuelven a `operator.admin`.
    `defineChannelPluginEntry` gestiona automáticamente la división del modo de registro. Consulta
    [Puntos de entrada](/es/plugins/sdk-entrypoints#definechannelpluginentry) para ver todas las
    opciones.

  </Step>

  <Step title="Añade una entrada de configuración">
    Crea `setup-entry.ts` para una carga ligera durante la incorporación:

    ```typescript setup-entry.ts
    import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineSetupPluginEntry(acmeChatPlugin);
    ```

    OpenClaw carga esto en lugar de la entrada completa cuando el canal está deshabilitado
    o sin configurar. Evita cargar código pesado de tiempo de ejecución durante los flujos de configuración.
    Consulta [Setup y Configuración](/es/plugins/sdk-setup#setup-entry) para más detalles.

    Los canales incluidos del espacio de trabajo que dividen las exportaciones seguras para configuración en módulos
    auxiliares pueden usar `defineBundledChannelSetupEntry(...)` desde
    `openclaw/plugin-sdk/channel-entry-contract` cuando también necesitan un
    setter explícito de tiempo de ejecución para el momento de configuración.

  </Step>

  <Step title="Gestiona los mensajes entrantes">
    Tu Plugin necesita recibir mensajes desde la plataforma y reenviarlos a
    OpenClaw. El patrón típico es un Webhook que verifica la solicitud y
    la despacha a través del controlador de entrada de tu canal:

    ```typescript
    registerFull(api) {
      api.registerHttpRoute({
        path: "/acme-chat/webhook",
        auth: "plugin", // plugin-managed auth (verify signatures yourself)
        handler: async (req, res) => {
          const event = parseWebhookPayload(req);

          // Your inbound handler dispatches the message to OpenClaw.
          // The exact wiring depends on your platform SDK —
          // see a real example in the bundled Microsoft Teams or Google Chat plugin package.
          await handleAcmeChatInbound(api, event);

          res.statusCode = 200;
          res.end("ok");
          return true;
        },
      });
    }
    ```

    <Note>
      El manejo de mensajes entrantes es específico de cada canal. Cada Plugin de canal gestiona
      su propia canalización de entrada. Mira los Plugins de canal incluidos
      (por ejemplo, el paquete Plugin de Microsoft Teams o Google Chat) para ver patrones reales.
    </Note>

  </Step>

<a id="step-6-test"></a>
<Step title="Prueba">
Escribe pruebas colocadas junto al código en `src/channel.test.ts`:

    ```typescript src/channel.test.ts
    import { describe, it, expect } from "vitest";
    import { acmeChatPlugin } from "./channel.js";

    describe("plugin acme-chat", () => {
      it("resuelve la cuenta desde la configuración", () => {
        const cfg = {
          channels: {
            "acme-chat": { token: "test-token", allowFrom: ["user1"] },
          },
        } as any;
        const account = acmeChatPlugin.setup!.resolveAccount(cfg, undefined);
        expect(account.token).toBe("test-token");
      });

      it("inspecciona la cuenta sin materializar secretos", () => {
        const cfg = {
          channels: { "acme-chat": { token: "test-token" } },
        } as any;
        const result = acmeChatPlugin.setup!.inspectAccount!(cfg, undefined);
        expect(result.configured).toBe(true);
        expect(result.tokenStatus).toBe("available");
      });

      it("informa de configuración faltante", () => {
        const cfg = { channels: {} } as any;
        const result = acmeChatPlugin.setup!.inspectAccount!(cfg, undefined);
        expect(result.configured).toBe(false);
      });
    });
    ```

    ```bash
    pnpm test -- <bundled-plugin-root>/acme-chat/
    ```

    Para los helpers de pruebas compartidos, consulta [Pruebas](/es/plugins/sdk-testing).

  </Step>
</Steps>

## Estructura de archivos

```
<bundled-plugin-root>/acme-chat/
├── package.json              # metadatos de openclaw.channel
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
Todavía existen algunas separaciones auxiliares incluidas para mantenimiento y
compatibilidad de Plugins incluidos. No son el patrón recomendado para Plugins de canal nuevos;
prefiere las subrutas genéricas de channel/setup/reply/runtime de la superficie común del SDK
salvo que estés manteniendo directamente esa familia de Plugins incluidos.
</Note>

## Siguientes pasos

- [Plugins de proveedor](/es/plugins/sdk-provider-plugins) — si tu Plugin también proporciona modelos
- [Resumen del SDK](/es/plugins/sdk-overview) — referencia completa de importaciones por subruta
- [Pruebas del SDK](/es/plugins/sdk-testing) — utilidades de prueba y pruebas de contrato
- [Manifiesto del Plugin](/es/plugins/manifest) — esquema completo del manifiesto
