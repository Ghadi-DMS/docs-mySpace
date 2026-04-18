---
read_when:
    - Vous créez un nouveau Plugin de canal de messagerie
    - Vous souhaitez connecter OpenClaw à une plateforme de messagerie
    - Vous devez comprendre la surface d’adaptation de ChannelPlugin
sidebarTitle: Channel Plugins
summary: Guide étape par étape pour créer un Plugin de canal de messagerie pour OpenClaw
title: Création de Plugins de canal
x-i18n:
    generated_at: "2026-04-18T06:44:10Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3dda53c969bc7356a450c2a5bf49fb82bf1283c23e301dec832d8724b11e724b
    source_path: plugins/sdk-channel-plugins.md
    workflow: 15
---

# Création de Plugins de canal

Ce guide explique comment créer un Plugin de canal qui connecte OpenClaw à une
plateforme de messagerie. À la fin, vous disposerez d’un canal fonctionnel avec sécurité des DM,
pairing, threading des réponses et messagerie sortante.

<Info>
  Si vous n’avez encore créé aucun Plugin OpenClaw, lisez d’abord
  [Getting Started](/fr/plugins/building-plugins) pour la structure de package
  de base et la configuration du manifeste.
</Info>

## Fonctionnement des Plugins de canal

Les Plugins de canal n’ont pas besoin de leurs propres outils d’envoi/édition/réaction. OpenClaw conserve un
outil `message` partagé dans le cœur. Votre Plugin prend en charge :

- **Configuration** — résolution de compte et assistant de configuration
- **Sécurité** — politique DM et listes d’autorisation
- **Pairing** — flux d’approbation DM
- **Grammaire de session** — comment les identifiants de conversation spécifiques au provider se mappent aux chats de base, aux identifiants de thread et aux solutions de repli parent
- **Sortant** — envoi de texte, de médias et de sondages vers la plateforme
- **Threading** — comment les réponses sont threadées

Le cœur prend en charge l’outil de message partagé, le câblage du prompt, la forme externe de la clé de session,
la tenue générique des registres `:thread:` et la répartition.

Si votre canal ajoute des paramètres d’outil de message qui transportent des sources média, exposez ces
noms de paramètres via `describeMessageTool(...).mediaSourceParams`. Le cœur utilise
cette liste explicite pour la normalisation des chemins du sandbox et la politique d’accès
aux médias sortants, afin que les Plugins n’aient pas besoin de cas particuliers dans le cœur partagé pour les paramètres
spécifiques au provider comme avatar, pièce jointe ou image de couverture.
Préférez renvoyer une map indexée par action comme
`{ "set-profile": ["avatarUrl", "avatarPath"] }` afin que des actions sans rapport n’héritent pas
des arguments média d’une autre action. Un tableau plat fonctionne toujours pour les paramètres
qui sont volontairement partagés par chaque action exposée.

Si votre plateforme stocke une portée supplémentaire dans les identifiants de conversation, gardez cette logique d’analyse
dans le Plugin avec `messaging.resolveSessionConversation(...)`. C’est le hook canonique
pour mapper `rawId` vers l’identifiant de conversation de base, l’identifiant de thread optionnel,
`baseConversationId` explicite et tout `parentConversationCandidates`.
Lorsque vous renvoyez `parentConversationCandidates`, conservez-les ordonnés
du parent le plus étroit au parent le plus large / à la conversation de base.

Les Plugins intégrés qui ont besoin de la même logique d’analyse avant
le démarrage du registre de canal peuvent aussi exposer un fichier `session-key-api.ts`
de niveau supérieur avec une exportation correspondante
`resolveSessionConversation(...)`. Le cœur utilise cette surface sûre pour l’amorçage
uniquement lorsque le registre de Plugin d’exécution n’est pas encore disponible.

`messaging.resolveParentConversationCandidates(...)` reste disponible comme solution de repli de compatibilité héritée lorsqu’un Plugin n’a besoin
que de solutions de repli parent en plus de l’identifiant générique/brut. Si les deux hooks existent, le cœur utilise
d’abord `resolveSessionConversation(...).parentConversationCandidates` et ne
revient à `resolveParentConversationCandidates(...)` que si le hook canonique
les omet.

## Approbations et capacités de canal

La plupart des Plugins de canal n’ont pas besoin de code spécifique aux approbations.

- Le cœur prend en charge `/approve` dans le même chat, les charges utiles de bouton d’approbation partagées et la livraison générique de secours.
- Préférez un seul objet `approvalCapability` sur le Plugin de canal lorsque le canal a besoin d’un comportement spécifique aux approbations.
- `ChannelPlugin.approvals` est supprimé. Placez les faits de livraison/native/render/auth des approbations dans `approvalCapability`.
- `plugin.auth` est réservé à login/logout ; le cœur ne lit plus les hooks d’authentification d’approbation à partir de cet objet.
- `approvalCapability.authorizeActorAction` et `approvalCapability.getActionAvailabilityState` constituent la surface canonique pour l’authentification d’approbation.
- Utilisez `approvalCapability.getActionAvailabilityState` pour la disponibilité de l’authentification d’approbation dans le même chat.
- Si votre canal expose des approbations exec natives, utilisez `approvalCapability.getExecInitiatingSurfaceState` pour l’état de la surface d’initiation / du client natif lorsqu’il diffère de l’authentification d’approbation dans le même chat. Le cœur utilise ce hook spécifique à exec pour distinguer `enabled` de `disabled`, décider si le canal initiateur prend en charge les approbations exec natives et inclure le canal dans les indications de secours pour client natif. `createApproverRestrictedNativeApprovalCapability(...)` remplit cela pour le cas courant.
- Utilisez `outbound.shouldSuppressLocalPayloadPrompt` ou `outbound.beforeDeliverPayload` pour le comportement spécifique au canal dans le cycle de vie des charges utiles, par exemple masquer les prompts locaux d’approbation en double ou envoyer des indicateurs de saisie avant la livraison.
- Utilisez `approvalCapability.delivery` uniquement pour le routage d’approbation natif ou la suppression des solutions de secours.
- Utilisez `approvalCapability.nativeRuntime` pour les faits d’approbation native détenus par le canal. Gardez-le lazy sur les points d’entrée chauds du canal avec `createLazyChannelApprovalNativeRuntimeAdapter(...)`, qui peut importer votre module runtime à la demande tout en permettant au cœur d’assembler le cycle de vie des approbations.
- Utilisez `approvalCapability.render` uniquement lorsqu’un canal a réellement besoin de charges utiles d’approbation personnalisées au lieu du moteur de rendu partagé.
- Utilisez `approvalCapability.describeExecApprovalSetup` lorsque le canal veut que la réponse du chemin désactivé explique les paramètres de configuration exacts nécessaires pour activer les approbations exec natives. Le hook reçoit `{ channel, channelLabel, accountId }` ; les canaux à compte nommé doivent générer des chemins à portée de compte comme `channels.<channel>.accounts.<id>.execApprovals.*` au lieu des valeurs par défaut de niveau supérieur.
- Si un canal peut déduire des identités DM stables de type propriétaire à partir de la configuration existante, utilisez `createResolvedApproverActionAuthAdapter` depuis `openclaw/plugin-sdk/approval-runtime` pour restreindre `/approve` dans le même chat sans ajouter de logique spécifique aux approbations dans le cœur.
- Si un canal a besoin d’une livraison d’approbation native, gardez le code du canal focalisé sur la normalisation des cibles ainsi que sur les faits de transport / présentation. Utilisez `createChannelExecApprovalProfile`, `createChannelNativeOriginTargetResolver`, `createChannelApproverDmTargetResolver` et `createApproverRestrictedNativeApprovalCapability` depuis `openclaw/plugin-sdk/approval-runtime`. Placez les faits spécifiques au canal derrière `approvalCapability.nativeRuntime`, idéalement via `createChannelApprovalNativeRuntimeAdapter(...)` ou `createLazyChannelApprovalNativeRuntimeAdapter(...)`, afin que le cœur puisse assembler le gestionnaire et prendre en charge le filtrage des requêtes, le routage, la déduplication, l’expiration, l’abonnement Gateway et les avis de routage vers un autre emplacement. `nativeRuntime` est découpé en quelques surfaces plus petites :
- `availability` — si le compte est configuré et si une requête doit être traitée
- `presentation` — mapper le view model d’approbation partagé vers des charges utiles natives pending/resolved/expired ou des actions finales
- `transport` — préparer les cibles puis envoyer / mettre à jour / supprimer les messages d’approbation natifs
- `interactions` — hooks facultatifs bind/unbind/clear-action pour les boutons ou réactions natifs
- `observe` — hooks facultatifs de diagnostic de livraison
- Si le canal a besoin d’objets détenus par le runtime comme un client, un token, une application Bolt ou un récepteur Webhook, enregistrez-les via `openclaw/plugin-sdk/channel-runtime-context`. Le registre générique de contexte runtime permet au cœur d’amorcer des gestionnaires pilotés par capacités à partir de l’état de démarrage du canal sans ajouter de glue d’encapsulation spécifique aux approbations.
- N’utilisez `createChannelApprovalHandler` ou `createChannelNativeApprovalRuntime` au niveau inférieur que lorsque la surface pilotée par capacités n’est pas encore assez expressive.
- Les canaux d’approbation native doivent faire transiter à la fois `accountId` et `approvalKind` par ces helpers. `accountId` maintient la portée correcte de la politique d’approbation multi-compte pour le bon compte bot, et `approvalKind` conserve le comportement d’approbation exec vs Plugin disponible pour le canal sans branches codées en dur dans le cœur.
- Le cœur prend désormais aussi en charge les avis de reroutage des approbations. Les Plugins de canal ne doivent pas envoyer leurs propres messages de suivi « l’approbation a été envoyée en DM / vers un autre canal » depuis `createChannelNativeApprovalRuntime` ; exposez plutôt un routage précis de l’origine + des DM d’approbateur via les helpers partagés de capacité d’approbation et laissez le cœur agréger les livraisons réelles avant de publier un avis dans le chat initiateur.
- Préservez le type d’identifiant d’approbation livré de bout en bout. Les clients natifs ne doivent pas
  deviner ou réécrire le routage d’approbation exec vs Plugin à partir de l’état local du canal.
- Des types d’approbation différents peuvent volontairement exposer des surfaces natives différentes.
  Exemples intégrés actuels :
  - Slack conserve le routage d’approbation native disponible à la fois pour les identifiants exec et Plugin.
  - Matrix conserve le même routage DM/canal natif et la même UX par réaction pour les approbations exec
    et Plugin, tout en laissant l’authentification varier selon le type d’approbation.
- `createApproverRestrictedNativeApprovalAdapter` existe toujours comme wrapper de compatibilité, mais le nouveau code doit préférer le générateur de capacités et exposer `approvalCapability` sur le Plugin.

Pour les points d’entrée chauds du canal, préférez les sous-chemins runtime plus étroits lorsque vous n’avez besoin
que d’une partie de cette famille :

- `openclaw/plugin-sdk/approval-auth-runtime`
- `openclaw/plugin-sdk/approval-client-runtime`
- `openclaw/plugin-sdk/approval-delivery-runtime`
- `openclaw/plugin-sdk/approval-gateway-runtime`
- `openclaw/plugin-sdk/approval-handler-adapter-runtime`
- `openclaw/plugin-sdk/approval-handler-runtime`
- `openclaw/plugin-sdk/approval-native-runtime`
- `openclaw/plugin-sdk/approval-reply-runtime`
- `openclaw/plugin-sdk/channel-runtime-context`

De même, préférez `openclaw/plugin-sdk/setup-runtime`,
`openclaw/plugin-sdk/setup-adapter-runtime`,
`openclaw/plugin-sdk/reply-runtime`,
`openclaw/plugin-sdk/reply-dispatch-runtime`,
`openclaw/plugin-sdk/reply-reference` et
`openclaw/plugin-sdk/reply-chunking` lorsque vous n’avez pas besoin de la surface
globale plus large.

Pour la configuration plus précisément :

- `openclaw/plugin-sdk/setup-runtime` couvre les helpers de configuration sûrs pour le runtime :
  adaptateurs de patch de configuration sûrs à l’importation (`createPatchedAccountSetupAdapter`,
  `createEnvPatchedAccountSetupAdapter`,
  `createSetupInputPresenceValidator`), sortie de note de recherche,
  `promptResolvedAllowFrom`, `splitSetupEntries` et les constructeurs
  de proxy de configuration déléguée
- `openclaw/plugin-sdk/setup-adapter-runtime` est la surface étroite d’adaptateur sensible à l’environnement
  pour `createEnvPatchedAccountSetupAdapter`
- `openclaw/plugin-sdk/channel-setup` couvre les constructeurs de configuration
  à installation facultative ainsi que quelques primitives sûres pour la configuration :
  `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`,

Si votre canal prend en charge une configuration ou une authentification pilotée par l’environnement et que les flux génériques de démarrage/configuration
doivent connaître ces noms de variables d’environnement avant le chargement du runtime, déclarez-les dans le
manifeste du Plugin avec `channelEnvVars`. Conservez `envVars` du runtime du canal ou des constantes locales
uniquement pour le texte orienté opérateur.
`createOptionalChannelSetupWizard`, `DEFAULT_ACCOUNT_ID`,
`createTopLevelChannelDmPolicy`, `setSetupChannelEnabled` et
`splitSetupEntries`

- utilisez la surface plus large `openclaw/plugin-sdk/setup` uniquement lorsque vous avez également besoin des
  helpers partagés plus lourds de configuration/configuration comme
  `moveSingleAccountChannelSectionToDefaultAccount(...)`

Si votre canal veut seulement annoncer « installez d’abord ce Plugin » dans les
surfaces de configuration, préférez `createOptionalChannelSetupSurface(...)`. L’adaptateur/l’assistant
généré échoue en mode fermé sur les écritures de configuration et la finalisation, et réutilise
le même message d’installation requise pour la validation, la finalisation et le texte
de lien vers la documentation.

Pour les autres chemins chauds du canal, préférez les helpers étroits aux
surfaces héritées plus larges :

- `openclaw/plugin-sdk/account-core`,
  `openclaw/plugin-sdk/account-id`,
  `openclaw/plugin-sdk/account-resolution` et
  `openclaw/plugin-sdk/account-helpers` pour la configuration multi-compte et
  la solution de repli vers le compte par défaut
- `openclaw/plugin-sdk/inbound-envelope` et
  `openclaw/plugin-sdk/inbound-reply-dispatch` pour le câblage des routes/enveloppes entrantes et
  record-and-dispatch
- `openclaw/plugin-sdk/messaging-targets` pour l’analyse et la correspondance des cibles
- `openclaw/plugin-sdk/outbound-media` et
  `openclaw/plugin-sdk/outbound-runtime` pour le chargement des médias ainsi que les délégués
  d’identité/envoi sortants
- `openclaw/plugin-sdk/thread-bindings-runtime` pour le cycle de vie des liaisons de thread
  et l’enregistrement des adaptateurs
- `openclaw/plugin-sdk/agent-media-payload` uniquement lorsqu’une disposition de champ héritée
  agent/media est encore requise
- `openclaw/plugin-sdk/telegram-command-config` pour la normalisation des commandes personnalisées Telegram,
  la validation des doublons/conflits et un contrat de configuration de commande
  stable en solution de repli

Les canaux uniquement d’authentification peuvent généralement s’arrêter au chemin par défaut : le cœur gère les approbations et le Plugin expose simplement les capacités sortantes/d’authentification. Les canaux d’approbation native comme Matrix, Slack, Telegram et les transports de chat personnalisés doivent utiliser les helpers natifs partagés au lieu de créer leur propre cycle de vie d’approbation.

## Politique de mention entrante

Conservez la gestion des mentions entrantes divisée en deux couches :

- collecte de preuves détenue par le Plugin
- évaluation de politique partagée

Utilisez `openclaw/plugin-sdk/channel-mention-gating` pour les décisions de politique de mention.
Utilisez `openclaw/plugin-sdk/channel-inbound` uniquement lorsque vous avez besoin de la surface
plus large de helpers entrants.

Bon choix pour la logique locale au Plugin :

- détection de réponse au bot
- détection de citation du bot
- vérifications de participation au thread
- exclusions de messages de service/système
- caches natifs de plateforme nécessaires pour prouver la participation du bot

Bon choix pour le helper partagé :

- `requireMention`
- résultat de mention explicite
- liste d’autorisation de mention implicite
- contournement des commandes
- décision finale d’ignorer

Flux recommandé :

1. Calculez les faits de mention locaux.
2. Passez ces faits à `resolveInboundMentionDecision({ facts, policy })`.
3. Utilisez `decision.effectiveWasMentioned`, `decision.shouldBypassMention` et `decision.shouldSkip` dans votre garde entrante.

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

`api.runtime.channel.mentions` expose les mêmes helpers de mention partagés pour les
Plugins de canal intégrés qui dépendent déjà de l’injection runtime :

- `buildMentionRegexes`
- `matchesMentionPatterns`
- `matchesMentionWithExplicit`
- `implicitMentionKindWhen`
- `resolveInboundMentionDecision`

Si vous n’avez besoin que de `implicitMentionKindWhen` et
`resolveInboundMentionDecision`, importez-les depuis
`openclaw/plugin-sdk/channel-mention-gating` afin d’éviter de charger des helpers runtime
entrants sans rapport.

Les anciens helpers `resolveMentionGating*` restent disponibles sur
`openclaw/plugin-sdk/channel-inbound` uniquement comme exportations de compatibilité. Le nouveau code
doit utiliser `resolveInboundMentionDecision({ facts, policy })`.

## Procédure détaillée

<Steps>
  <a id="step-1-package-and-manifest"></a>
  <Step title="Package et manifeste">
    Créez les fichiers standards du Plugin. Le champ `channel` dans `package.json` est
    ce qui en fait un Plugin de canal. Pour la surface complète des métadonnées de package,
    consultez [Setup and Config](/fr/plugins/sdk-setup#openclaw-channel) :

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

  <Step title="Construire l’objet Plugin de canal">
    L’interface `ChannelPlugin` comporte de nombreuses surfaces d’adaptation facultatives. Commencez par
    le minimum — `id` et `setup` — puis ajoutez des adaptateurs selon vos besoins.

    Créez `src/channel.ts` :

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

    <Accordion title="Ce que createChatChannelPlugin fait pour vous">
      Au lieu d’implémenter manuellement des interfaces d’adaptation de bas niveau, vous passez
      des options déclaratives et le constructeur les compose :

      | Option | Ce qui est câblé |
      | --- | --- |
      | `security.dm` | Résolveur de sécurité DM à portée limitée à partir des champs de configuration |
      | `pairing.text` | Flux de pairing DM basé sur du texte avec échange de code |
      | `threading` | Résolveur de mode de réponse (fixe, à portée de compte ou personnalisé) |
      | `outbound.attachedResults` | Fonctions d’envoi qui renvoient des métadonnées de résultat (identifiants de message) |

      Vous pouvez également passer des objets d’adaptateur bruts au lieu des options déclaratives
      si vous avez besoin d’un contrôle total.
    </Accordion>

  </Step>

  <Step title="Câbler le point d’entrée">
    Créez `index.ts` :

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

    Placez les descripteurs CLI détenus par le canal dans `registerCliMetadata(...)` afin qu’OpenClaw
    puisse les afficher dans l’aide racine sans activer le runtime complet du canal,
    tandis que les chargements complets normaux récupèrent toujours les mêmes descripteurs pour le véritable enregistrement
    des commandes. Réservez `registerFull(...)` aux tâches limitées au runtime.
    Si `registerFull(...)` enregistre des méthodes RPC Gateway, utilisez un
    préfixe spécifique au Plugin. Les espaces de noms d’administration du cœur (`config.*`,
    `exec.approvals.*`, `wizard.*`, `update.*`) restent réservés et se
    résolvent toujours vers `operator.admin`.
    `defineChannelPluginEntry` gère automatiquement cette séparation des modes d’enregistrement. Consultez
    [Entry Points](/fr/plugins/sdk-entrypoints#definechannelpluginentry) pour toutes les
    options.

  </Step>

  <Step title="Ajouter un point d’entrée de configuration">
    Créez `setup-entry.ts` pour un chargement léger pendant l’onboarding :

    ```typescript setup-entry.ts
    import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineSetupPluginEntry(acmeChatPlugin);
    ```

    OpenClaw charge ceci au lieu du point d’entrée complet lorsque le canal est désactivé
    ou non configuré. Cela évite de charger du code runtime lourd pendant les flux de configuration.
    Consultez [Setup and Config](/fr/plugins/sdk-setup#setup-entry) pour plus de détails.

    Les canaux d’espace de travail intégrés qui répartissent les exportations sûres pour la configuration dans des modules
    compagnons peuvent utiliser `defineBundledChannelSetupEntry(...)` depuis
    `openclaw/plugin-sdk/channel-entry-contract` lorsqu’ils ont également besoin d’un
    setter runtime explicite au moment de la configuration.

  </Step>

  <Step title="Gérer les messages entrants">
    Votre Plugin doit recevoir les messages depuis la plateforme et les transférer vers
    OpenClaw. Le modèle habituel est un Webhook qui vérifie la requête et
    la répartit via le gestionnaire entrant de votre canal :

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
      La gestion des messages entrants est spécifique au canal. Chaque Plugin de canal possède
      son propre pipeline entrant. Regardez les Plugins de canal intégrés
      (par exemple le package Plugin Microsoft Teams ou Google Chat) pour voir de vrais modèles.
    </Note>

  </Step>

<a id="step-6-test"></a>
<Step title="Tester">
Écrivez des tests colocalisés dans `src/channel.test.ts` :

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

    Pour les helpers de test partagés, consultez [Testing](/fr/plugins/sdk-testing).

  </Step>
</Steps>

## Structure des fichiers

```
<bundled-plugin-root>/acme-chat/
├── package.json              # métadonnées openclaw.channel
├── openclaw.plugin.json      # Manifeste avec schéma de configuration
├── index.ts                  # defineChannelPluginEntry
├── setup-entry.ts            # defineSetupPluginEntry
├── api.ts                    # Exportations publiques (facultatif)
├── runtime-api.ts            # Exportations runtime internes (facultatif)
└── src/
    ├── channel.ts            # ChannelPlugin via createChatChannelPlugin
    ├── channel.test.ts       # Tests
    ├── client.ts             # Client API de la plateforme
    └── runtime.ts            # Store runtime (si nécessaire)
```

## Sujets avancés

<CardGroup cols={2}>
  <Card title="Options de threading" icon="git-branch" href="/fr/plugins/sdk-entrypoints#registration-mode">
    Modes de réponse fixes, à portée de compte ou personnalisés
  </Card>
  <Card title="Intégration de l’outil de message" icon="puzzle" href="/fr/plugins/architecture#channel-plugins-and-the-shared-message-tool">
    describeMessageTool et découverte d’action
  </Card>
  <Card title="Résolution de cible" icon="crosshair" href="/fr/plugins/architecture#channel-target-resolution">
    inferTargetChatType, looksLikeId, resolveTarget
  </Card>
  <Card title="Helpers runtime" icon="settings" href="/fr/plugins/sdk-runtime">
    TTS, STT, média, sous-agent via api.runtime
  </Card>
</CardGroup>

<Note>
Certaines surfaces de helpers intégrés existent encore pour la maintenance et la
compatibilité des Plugins intégrés. Ce n’est pas le modèle recommandé pour les nouveaux Plugins de canal ;
préférez les sous-chemins génériques channel/setup/reply/runtime de la surface
SDK commune, sauf si vous maintenez directement cette famille de Plugins intégrés.
</Note>

## Étapes suivantes

- [Provider Plugins](/fr/plugins/sdk-provider-plugins) — si votre Plugin fournit également des modèles
- [SDK Overview](/fr/plugins/sdk-overview) — référence complète des importations par sous-chemin
- [SDK Testing](/fr/plugins/sdk-testing) — utilitaires de test et tests de contrat
- [Plugin Manifest](/fr/plugins/manifest) — schéma complet du manifeste
