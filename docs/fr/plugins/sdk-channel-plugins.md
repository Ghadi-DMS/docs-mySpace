---
read_when:
    - Vous créez un nouveau plugin de canal de messagerie
    - Vous souhaitez connecter OpenClaw à une plateforme de messagerie
    - Vous devez comprendre la surface d’adaptation ChannelPlugin
sidebarTitle: Channel Plugins
summary: Guide pas à pas pour créer un plugin de canal de messagerie pour OpenClaw
title: Créer des plugins de canal
x-i18n:
    generated_at: "2026-04-08T02:17:06Z"
    model: gpt-5.4
    provider: openai
    source_hash: d23365b6d92006b30e671f9f0afdba40a2b88c845c5d2299d71c52a52985672f
    source_path: plugins/sdk-channel-plugins.md
    workflow: 15
---

# Créer des plugins de canal

Ce guide explique comment créer un plugin de canal qui connecte OpenClaw à une
plateforme de messagerie. À la fin, vous disposerez d’un canal fonctionnel avec sécurité DM,
association, enfilage des réponses et messagerie sortante.

<Info>
  Si vous n’avez encore jamais créé de plugin OpenClaw, lisez d’abord
  [Premiers pas](/fr/plugins/building-plugins) pour la structure de base du package
  et la configuration du manifeste.
</Info>

## Fonctionnement des plugins de canal

Les plugins de canal n’ont pas besoin de leurs propres outils send/edit/react. OpenClaw conserve un
outil `message` partagé dans le cœur. Votre plugin prend en charge :

- **Configuration** — résolution des comptes et assistant de configuration
- **Sécurité** — politique DM et listes d’autorisation
- **Association** — flux d’approbation des DMs
- **Grammaire de session** — comment les identifiants de conversation spécifiques au fournisseur sont mappés vers les discussions de base, les identifiants de fil et les replis parents
- **Sortant** — envoi de texte, médias et sondages vers la plateforme
- **Enfilage** — comment les réponses sont enfilées

Le cœur prend en charge l’outil message partagé, le câblage des prompts, la forme externe des clés de session,
la tenue générique des `:thread:` et la distribution.

Si votre plateforme stocke une portée supplémentaire dans les identifiants de conversation, conservez cette logique d’analyse
dans le plugin avec `messaging.resolveSessionConversation(...)`. C’est le hook canonique pour mapper
`rawId` vers l’identifiant de conversation de base, un identifiant de fil optionnel,
un `baseConversationId` explicite et d’éventuels `parentConversationCandidates`.
Lorsque vous renvoyez `parentConversationCandidates`, conservez-les ordonnés du
parent le plus étroit vers la conversation parente/la conversation de base la plus large.

Les plugins groupés qui ont besoin de la même analyse avant le démarrage du registre de canaux
peuvent également exposer un fichier `session-key-api.ts` de niveau supérieur avec un export
`resolveSessionConversation(...)` correspondant. Le cœur utilise cette surface sûre pour l’amorçage
uniquement lorsque le registre de plugins à l’exécution n’est pas encore disponible.

`messaging.resolveParentConversationCandidates(...)` reste disponible comme repli de compatibilité hérité lorsque
un plugin n’a besoin que de replis parents au-dessus de l’id générique/brut. Si les deux hooks existent, le cœur utilise
d’abord `resolveSessionConversation(...).parentConversationCandidates` et ne
revient à `resolveParentConversationCandidates(...)` que lorsque le hook canonique
les omet.

## Approbations et capacités de canal

La plupart des plugins de canal n’ont pas besoin de code spécifique aux approbations.

- Le cœur prend en charge `/approve` dans la même discussion, les charges utiles partagées des boutons d’approbation et la livraison de repli générique.
- Préférez un unique objet `approvalCapability` sur le plugin de canal lorsque le canal a besoin d’un comportement spécifique aux approbations.
- `ChannelPlugin.approvals` a été supprimé. Placez les informations de livraison/rendu/auth natives d’approbation dans `approvalCapability`.
- `plugin.auth` sert uniquement à login/logout ; le cœur ne lit plus les hooks d’auth d’approbation à partir de cet objet.
- `approvalCapability.authorizeActorAction` et `approvalCapability.getActionAvailabilityState` sont la jonction canonique pour l’auth d’approbation.
- Utilisez `approvalCapability.getActionAvailabilityState` pour la disponibilité de l’auth d’approbation dans la même discussion.
- Si votre canal expose des approbations exec natives, utilisez `approvalCapability.getExecInitiatingSurfaceState` pour l’état de la surface initiatrice/du client natif lorsqu’il diffère de l’auth d’approbation dans la même discussion. Le cœur utilise ce hook spécifique à exec pour distinguer `enabled` et `disabled`, décider si le canal initiateur prend en charge les approbations exec natives et inclure le canal dans les indications de repli vers le client natif. `createApproverRestrictedNativeApprovalCapability(...)` remplit cela pour le cas courant.
- Utilisez `outbound.shouldSuppressLocalPayloadPrompt` ou `outbound.beforeDeliverPayload` pour le comportement spécifique au canal dans le cycle de vie de la charge utile, par exemple masquer des prompts d’approbation locaux en double ou envoyer des indicateurs de saisie avant la livraison.
- Utilisez `approvalCapability.delivery` uniquement pour le routage d’approbation native ou la suppression du repli.
- Utilisez `approvalCapability.nativeRuntime` pour les faits d’approbation native détenus par le canal. Conservez-le paresseux sur les points d’entrée chauds du canal avec `createLazyChannelApprovalNativeRuntimeAdapter(...)`, qui peut importer votre module runtime à la demande tout en permettant au cœur d’assembler le cycle de vie des approbations.
- Utilisez `approvalCapability.render` uniquement lorsqu’un canal a réellement besoin de charges utiles d’approbation personnalisées plutôt que du moteur de rendu partagé.
- Utilisez `approvalCapability.describeExecApprovalSetup` lorsque le canal souhaite que la réponse du chemin désactivé explique les paramètres de configuration exacts nécessaires pour activer les approbations exec natives. Le hook reçoit `{ channel, channelLabel, accountId }` ; les canaux à comptes nommés doivent afficher des chemins limités au compte tels que `channels.<channel>.accounts.<id>.execApprovals.*` plutôt que des valeurs par défaut de niveau supérieur.
- Si un canal peut déduire des identités DM stables de type propriétaire à partir de la configuration existante, utilisez `createResolvedApproverActionAuthAdapter` depuis `openclaw/plugin-sdk/approval-runtime` pour restreindre `/approve` dans la même discussion sans ajouter de logique spécifique aux approbations dans le cœur.
- Si un canal a besoin de livraison d’approbation native, gardez le code du canal concentré sur la normalisation des cibles ainsi que sur les faits de transport/présentation. Utilisez `createChannelExecApprovalProfile`, `createChannelNativeOriginTargetResolver`, `createChannelApproverDmTargetResolver` et `createApproverRestrictedNativeApprovalCapability` depuis `openclaw/plugin-sdk/approval-runtime`. Placez les faits spécifiques au canal derrière `approvalCapability.nativeRuntime`, idéalement via `createChannelApprovalNativeRuntimeAdapter(...)` ou `createLazyChannelApprovalNativeRuntimeAdapter(...)`, afin que le cœur puisse assembler le gestionnaire et prendre en charge le filtrage des requêtes, le routage, la déduplication, l’expiration, l’abonnement à la passerelle et les notifications de reroutage. `nativeRuntime` est divisé en quelques jonctions plus petites :
- `availability` — si le compte est configuré et si une requête doit être traitée
- `presentation` — mappe le modèle de vue d’approbation partagé vers des charges utiles natives en attente/résolues/expirées ou des actions finales
- `transport` — prépare les cibles puis envoie/met à jour/supprime les messages d’approbation natifs
- `interactions` — hooks facultatifs de liaison/déliaison/effacement d’action pour les boutons ou réactions natifs
- `observe` — hooks facultatifs de diagnostic de livraison
- Si le canal a besoin d’objets détenus par le runtime comme un client, un jeton, une app Bolt ou un récepteur webhook, enregistrez-les via `openclaw/plugin-sdk/channel-runtime-context`. Le registre générique de contexte runtime permet au cœur d’amorcer des gestionnaires pilotés par capacités à partir de l’état de démarrage du canal sans ajouter de colle d’enrobage spécifique aux approbations.
- Utilisez les niveaux inférieurs `createChannelApprovalHandler` ou `createChannelNativeApprovalRuntime` uniquement lorsque la jonction pilotée par capacités n’est pas encore assez expressive.
- Les canaux d’approbation native doivent faire transiter à la fois `accountId` et `approvalKind` via ces helpers. `accountId` maintient la portée de la politique d’approbation multi-comptes sur le bon compte bot, et `approvalKind` maintient le comportement d’approbation exec vs plugin disponible pour le canal sans branches codées en dur dans le cœur.
- Le cœur prend désormais aussi en charge les notifications de reroutage d’approbation. Les plugins de canal ne doivent pas envoyer leurs propres messages de suivi « l’approbation a été envoyée vers les DMs / un autre canal » depuis `createChannelNativeApprovalRuntime`; exposez plutôt un routage précis de l’origine + DM de l’approbateur via les helpers partagés de capacité d’approbation et laissez le cœur agréger les livraisons réelles avant de publier une notification dans la discussion initiatrice.
- Préservez de bout en bout le type d’identifiant d’approbation livré. Les clients natifs ne doivent pas
  deviner ni réécrire le routage des approbations exec vs plugin à partir d’un état local au canal.
- Différents types d’approbation peuvent volontairement exposer différentes surfaces natives.
  Exemples groupés actuels :
  - Slack conserve le routage natif d’approbation disponible pour les identifiants exec comme plugin.
  - Matrix conserve le même routage DM/canal natif et la même UX par réaction pour les approbations exec
    et plugin, tout en permettant à l’auth de différer selon le type d’approbation.
- `createApproverRestrictedNativeApprovalAdapter` existe toujours comme wrapper de compatibilité, mais le nouveau code doit préférer le constructeur de capacités et exposer `approvalCapability` sur le plugin.

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
`openclaw/plugin-sdk/reply-chunking` lorsque vous n’avez pas besoin de la surface englobante
plus large.

Spécifiquement pour setup :

- `openclaw/plugin-sdk/setup-runtime` couvre les helpers setup sûrs pour l’exécution :
  adaptateurs de patch setup import-safe (`createPatchedAccountSetupAdapter`,
  `createEnvPatchedAccountSetupAdapter`,
  `createSetupInputPresenceValidator`), sortie de note de recherche,
  `promptResolvedAllowFrom`, `splitSetupEntries` et les constructeurs
  de proxy setup délégués
- `openclaw/plugin-sdk/setup-adapter-runtime` est la jonction d’adaptateur étroite sensible à l’environnement
  pour `createEnvPatchedAccountSetupAdapter`
- `openclaw/plugin-sdk/channel-setup` couvre les constructeurs setup pour installation facultative
  ainsi que quelques primitives sûres pour setup :
  `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`,

Si votre canal prend en charge une configuration ou une auth pilotée par variables d’environnement et que les flux génériques de démarrage/configuration
doivent connaître ces noms de variables avant le chargement du runtime, déclarez-les dans le
manifeste du plugin avec `channelEnvVars`. Conservez `envVars` dans le runtime du canal ou les constantes locales uniquement pour le texte destiné à l’opérateur.
`createOptionalChannelSetupWizard`, `DEFAULT_ACCOUNT_ID`,
`createTopLevelChannelDmPolicy`, `setSetupChannelEnabled` et
`splitSetupEntries`

- utilisez la jonction plus large `openclaw/plugin-sdk/setup` uniquement lorsque vous avez aussi besoin des helpers setup/config partagés plus lourds comme
  `moveSingleAccountChannelSectionToDefaultAccount(...)`

Si votre canal veut seulement afficher « installez d’abord ce plugin » dans les surfaces setup, préférez
`createOptionalChannelSetupSurface(...)`. L’adaptateur/l’assistant généré
échoue en mode fermé sur les écritures de configuration et la finalisation, et réutilise
le même message d’installation requise dans la validation, la finalisation et le texte de lien
vers la documentation.

Pour les autres chemins chauds du canal, préférez les helpers étroits aux surfaces héritées plus larges :

- `openclaw/plugin-sdk/account-core`,
  `openclaw/plugin-sdk/account-id`,
  `openclaw/plugin-sdk/account-resolution` et
  `openclaw/plugin-sdk/account-helpers` pour la configuration multi-comptes et le
  repli vers le compte par défaut
- `openclaw/plugin-sdk/inbound-envelope` et
  `openclaw/plugin-sdk/inbound-reply-dispatch` pour le câblage de route/enveloppe entrante et
  d’enregistrement-et-distribution
- `openclaw/plugin-sdk/messaging-targets` pour l’analyse et la correspondance des cibles
- `openclaw/plugin-sdk/outbound-media` et
  `openclaw/plugin-sdk/outbound-runtime` pour le chargement des médias ainsi que les délégués d’identité/envoi sortants
- `openclaw/plugin-sdk/thread-bindings-runtime` pour le cycle de vie des liaisons de fil
  et l’enregistrement d’adaptateur
- `openclaw/plugin-sdk/agent-media-payload` uniquement lorsqu’une disposition héritée des champs de charge utile agent/média est encore requise
- `openclaw/plugin-sdk/telegram-command-config` pour la normalisation des commandes personnalisées Telegram,
  la validation des doublons/conflits et un contrat de configuration de commandes
  stable en repli

Les canaux à authentification seule peuvent généralement s’arrêter au chemin par défaut : le cœur gère les approbations et le plugin expose simplement les capacités de sortie/auth. Les canaux d’approbation native tels que Matrix, Slack, Telegram et les transports de discussion personnalisés doivent utiliser les helpers natifs partagés au lieu de développer leur propre cycle de vie d’approbation.

## Politique de mention entrante

Conservez la gestion des mentions entrantes divisée en deux couches :

- collecte des preuves détenue par le plugin
- évaluation de politique partagée

Utilisez `openclaw/plugin-sdk/channel-inbound` pour la couche partagée.

Bon cas d’usage pour la logique locale au plugin :

- détection d’une réponse au bot
- détection d’une citation du bot
- vérifications de participation au fil
- exclusions des messages système/de service
- caches natifs à la plateforme nécessaires pour prouver la participation du bot

Bon cas d’usage pour le helper partagé :

- `requireMention`
- résultat de mention explicite
- liste d’autorisation de mention implicite
- contournement de commande
- décision finale d’ignorer

Flux recommandé :

1. Calculer localement les faits de mention.
2. Passer ces faits à `resolveInboundMentionDecision({ facts, policy })`.
3. Utiliser `decision.effectiveWasMentioned`, `decision.shouldBypassMention` et `decision.shouldSkip` dans votre filtre entrant.

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

`api.runtime.channel.mentions` expose les mêmes helpers de mention partagés pour
les plugins de canal groupés qui dépendent déjà de l’injection runtime :

- `buildMentionRegexes`
- `matchesMentionPatterns`
- `matchesMentionWithExplicit`
- `implicitMentionKindWhen`
- `resolveInboundMentionDecision`

Les anciens helpers `resolveMentionGating*` restent présents dans
`openclaw/plugin-sdk/channel-inbound` uniquement comme exports de compatibilité. Le nouveau code
doit utiliser `resolveInboundMentionDecision({ facts, policy })`.

## Tutoriel pas à pas

<Steps>
  <a id="step-1-package-and-manifest"></a>
  <Step title="Package et manifeste">
    Créez les fichiers de plugin standard. Le champ `channel` dans `package.json` est
    ce qui en fait un plugin de canal. Pour la surface complète des métadonnées de package,
    voir [Configuration et setup du plugin](/fr/plugins/sdk-setup#openclawchannel) :

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

  <Step title="Créer l’objet plugin de canal">
    L’interface `ChannelPlugin` comporte de nombreuses surfaces d’adaptation facultatives. Commencez par
    le minimum — `id` et `setup` — puis ajoutez des adaptateurs selon vos besoins.

    Créez `src/channel.ts` :

    ```typescript src/channel.ts
    import {
      createChatChannelPlugin,
      createChannelPluginBase,
    } from "openclaw/plugin-sdk/channel-core";
    import type { OpenClawConfig } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatApi } from "./client.js"; // client API de votre plateforme

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

      // Sécurité DM : qui peut envoyer des messages au bot
      security: {
        dm: {
          channelKey: "acme-chat",
          resolvePolicy: (account) => account.dmPolicy,
          resolveAllowFrom: (account) => account.allowFrom,
          defaultPolicy: "allowlist",
        },
      },

      // Association : flux d’approbation pour les nouveaux contacts DM
      pairing: {
        text: {
          idLabel: "Nom d’utilisateur Acme Chat",
          message: "Envoyez ce code pour vérifier votre identité :",
          notify: async ({ target, code }) => {
            await acmeChatApi.sendDm(target, `Pairing code: ${code}`);
          },
        },
      },

      // Enfilage : comment les réponses sont livrées
      threading: { topLevelReplyToMode: "reply" },

      // Sortant : envoyer des messages vers la plateforme
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
      Au lieu d’implémenter manuellement des interfaces d’adaptation de bas niveau, vous transmettez
      des options déclaratives et le constructeur les compose :

      | Option | Ce qui est câblé |
      | --- | --- |
      | `security.dm` | Résolveur de sécurité DM limité à la configuration à partir des champs de config |
      | `pairing.text` | Flux d’association DM basé sur le texte avec échange de code |
      | `threading` | Résolveur du mode de réponse (fixe, limité au compte ou personnalisé) |
      | `outbound.attachedResults` | Fonctions d’envoi qui renvoient des métadonnées de résultat (IDs de message) |

      Vous pouvez également transmettre des objets d’adaptateur bruts au lieu des options déclaratives
      si vous avez besoin d’un contrôle complet.
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
    tandis que les chargements complets normaux récupèrent toujours les mêmes descripteurs pour le véritable enregistrement des commandes.
    Conservez `registerFull(...)` pour le travail limité au runtime.
    Si `registerFull(...)` enregistre des méthodes RPC de passerelle, utilisez un
    préfixe spécifique au plugin. Les espaces de noms d’administration du cœur (`config.*`,
    `exec.approvals.*`, `wizard.*`, `update.*`) restent réservés et sont toujours
    résolus vers `operator.admin`.
    `defineChannelPluginEntry` gère automatiquement la séparation des modes d’enregistrement. Voir
    [Points d’entrée](/fr/plugins/sdk-entrypoints#definechannelpluginentry) pour toutes les
    options.

  </Step>

  <Step title="Ajouter une entrée setup">
    Créez `setup-entry.ts` pour un chargement léger pendant l’onboarding :

    ```typescript setup-entry.ts
    import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineSetupPluginEntry(acmeChatPlugin);
    ```

    OpenClaw charge cela au lieu du point d’entrée complet lorsque le canal est désactivé
    ou non configuré. Cela évite de charger du code runtime lourd pendant les flux de setup.
    Voir [Setup et configuration](/fr/plugins/sdk-setup#setup-entry) pour plus de détails.

  </Step>

  <Step title="Gérer les messages entrants">
    Votre plugin doit recevoir les messages de la plateforme et les transmettre à
    OpenClaw. Le schéma typique est un webhook qui vérifie la requête et
    la distribue via le gestionnaire entrant de votre canal :

    ```typescript
    registerFull(api) {
      api.registerHttpRoute({
        path: "/acme-chat/webhook",
        auth: "plugin", // auth gérée par le plugin (vérifiez vous-même les signatures)
        handler: async (req, res) => {
          const event = parseWebhookPayload(req);

          // Votre gestionnaire entrant distribue le message à OpenClaw.
          // Le câblage exact dépend du SDK de votre plateforme —
          // voir un exemple réel dans le package plugin Microsoft Teams ou Google Chat groupé.
          await handleAcmeChatInbound(api, event);

          res.statusCode = 200;
          res.end("ok");
          return true;
        },
      });
    }
    ```

    <Note>
      La gestion des messages entrants dépend du canal. Chaque plugin de canal prend en charge
      son propre pipeline entrant. Consultez les plugins de canal groupés
      (par exemple le package plugin Microsoft Teams ou Google Chat) pour voir des schémas réels.
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

    Pour les helpers de test partagés, voir [Tests](/fr/plugins/sdk-testing).

  </Step>
</Steps>

## Structure des fichiers

```
<bundled-plugin-root>/acme-chat/
├── package.json              # métadonnées openclaw.channel
├── openclaw.plugin.json      # Manifeste avec schéma de configuration
├── index.ts                  # defineChannelPluginEntry
├── setup-entry.ts            # defineSetupPluginEntry
├── api.ts                    # Exports publics (facultatif)
├── runtime-api.ts            # Exports runtime internes (facultatif)
└── src/
    ├── channel.ts            # ChannelPlugin via createChatChannelPlugin
    ├── channel.test.ts       # Tests
    ├── client.ts             # Client API de la plateforme
    └── runtime.ts            # Stockage runtime (si nécessaire)
```

## Sujets avancés

<CardGroup cols={2}>
  <Card title="Options d’enfilage" icon="git-branch" href="/fr/plugins/sdk-entrypoints#registration-mode">
    Modes de réponse fixes, limités au compte ou personnalisés
  </Card>
  <Card title="Intégration de l’outil message" icon="puzzle" href="/fr/plugins/architecture#channel-plugins-and-the-shared-message-tool">
    describeMessageTool et découverte des actions
  </Card>
  <Card title="Résolution de cible" icon="crosshair" href="/fr/plugins/architecture#channel-target-resolution">
    inferTargetChatType, looksLikeId, resolveTarget
  </Card>
  <Card title="Helpers runtime" icon="settings" href="/fr/plugins/sdk-runtime">
    TTS, STT, médias, sous-agent via api.runtime
  </Card>
</CardGroup>

<Note>
Certaines jonctions de helpers groupés existent encore pour la maintenance des plugins groupés et
la compatibilité. Elles ne constituent pas le modèle recommandé pour les nouveaux plugins de canal ;
préférez les sous-chemins génériques channel/setup/reply/runtime de la surface SDK commune
à moins de maintenir directement cette famille de plugins groupés.
</Note>

## Étapes suivantes

- [Plugins fournisseur](/fr/plugins/sdk-provider-plugins) — si votre plugin fournit aussi des modèles
- [Vue d’ensemble du SDK](/fr/plugins/sdk-overview) — référence complète des imports par sous-chemin
- [Tests du SDK](/fr/plugins/sdk-testing) — utilitaires de test et tests de contrat
- [Manifeste de plugin](/fr/plugins/manifest) — schéma complet du manifeste
