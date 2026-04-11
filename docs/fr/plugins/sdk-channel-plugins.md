---
read_when:
    - Vous créez un nouveau plugin de canal de messagerie
    - Vous souhaitez connecter OpenClaw à une plateforme de messagerie
    - Vous devez comprendre la surface d’adaptation `ChannelPlugin`
sidebarTitle: Channel Plugins
summary: Guide étape par étape pour créer un plugin de canal de messagerie pour OpenClaw
title: Créer des plugins de canal
x-i18n:
    generated_at: "2026-04-11T02:46:17Z"
    model: gpt-5.4
    provider: openai
    source_hash: 8a026e924f9ae8a3ddd46287674443bcfccb0247be504261522b078e1f440aef
    source_path: plugins/sdk-channel-plugins.md
    workflow: 15
---

# Créer des plugins de canal

Ce guide vous accompagne dans la création d’un plugin de canal qui connecte OpenClaw à une
plateforme de messagerie. À la fin, vous disposerez d’un canal fonctionnel avec sécurité des DM,
appairage, threading des réponses, et messagerie sortante.

<Info>
  Si vous n’avez encore créé aucun plugin OpenClaw, lisez d’abord
  [Getting Started](/fr/plugins/building-plugins) pour la structure de package
  de base et la configuration du manifeste.
</Info>

## Fonctionnement des plugins de canal

Les plugins de canal n’ont pas besoin de leurs propres outils send/edit/react. OpenClaw conserve un
outil `message` partagé dans le core. Votre plugin possède :

- **Config** — résolution des comptes et assistant de configuration
- **Security** — politique DM et listes d’autorisation
- **Pairing** — flux d’approbation des DM
- **Session grammar** — manière dont les ids de conversation propres au fournisseur se mappent aux chats de base, aux ids de thread, et aux fallbacks parents
- **Outbound** — envoi de texte, médias, et sondages vers la plateforme
- **Threading** — manière dont les réponses sont threadées

Le core possède l’outil message partagé, le câblage des prompts, la forme externe de la clé de session,
la gestion générique `:thread:`, et la répartition.

Si votre plateforme stocke une portée supplémentaire dans les ids de conversation, conservez cette analyse
dans le plugin avec `messaging.resolveSessionConversation(...)`. C’est le hook canonique pour mapper
`rawId` vers l’id de conversation de base, l’id de thread optionnel, `baseConversationId` explicite,
et tout `parentConversationCandidates`.
Lorsque vous renvoyez `parentConversationCandidates`, gardez-les ordonnés du parent
le plus étroit vers la conversation parente/la conversation de base la plus large.

Les plugins intégrés qui ont besoin de la même analyse avant le démarrage du registre de canaux
peuvent aussi exposer un fichier `session-key-api.ts` de premier niveau avec un export
`resolveSessionConversation(...)` correspondant. Le core utilise cette surface sûre au bootstrap
uniquement lorsque le registre de plugins runtime n’est pas encore disponible.

`messaging.resolveParentConversationCandidates(...)` reste disponible comme fallback de compatibilité
hérité lorsqu’un plugin n’a besoin que de fallbacks parents au-dessus de l’id générique/brut.
Si les deux hooks existent, le core utilise d’abord
`resolveSessionConversation(...).parentConversationCandidates` et ne retombe sur
`resolveParentConversationCandidates(...)` que lorsque le hook canonique
les omet.

## Approbations et capacités des canaux

La plupart des plugins de canal n’ont pas besoin de code spécifique aux approbations.

- Le core possède `/approve` dans le même chat, les payloads partagés des boutons d’approbation, et la livraison de fallback générique.
- Préférez un seul objet `approvalCapability` sur le plugin de canal lorsque le canal a besoin d’un comportement spécifique aux approbations.
- `ChannelPlugin.approvals` est supprimé. Placez les faits d’approbation de livraison/natifs/rendu/authentification sur `approvalCapability`.
- `plugin.auth` sert uniquement à login/logout ; le core ne lit plus les hooks d’authentification d’approbation depuis cet objet.
- `approvalCapability.authorizeActorAction` et `approvalCapability.getActionAvailabilityState` sont la couture canonique pour l’authentification des approbations.
- Utilisez `approvalCapability.getActionAvailabilityState` pour la disponibilité de l’authentification des approbations dans le même chat.
- Si votre canal expose des approbations d’exécution natives, utilisez `approvalCapability.getExecInitiatingSurfaceState` pour l’état de la surface initiatrice/du client natif lorsqu’il diffère de l’authentification des approbations dans le même chat. Le core utilise ce hook spécifique à l’exécution pour distinguer `enabled` de `disabled`, décider si le canal initiateur prend en charge les approbations d’exécution natives, et inclure le canal dans les indications de fallback du client natif. `createApproverRestrictedNativeApprovalCapability(...)` remplit ce point pour le cas courant.
- Utilisez `outbound.shouldSuppressLocalPayloadPrompt` ou `outbound.beforeDeliverPayload` pour le comportement du cycle de vie des payloads spécifique au canal, par exemple masquer les prompts d’approbation locale en double ou envoyer des indicateurs de frappe avant la livraison.
- Utilisez `approvalCapability.delivery` uniquement pour le routage d’approbation natif ou la suppression de fallback.
- Utilisez `approvalCapability.nativeRuntime` pour les faits d’approbation native appartenant au canal. Gardez-le lazy sur les points d’entrée chauds du canal avec `createLazyChannelApprovalNativeRuntimeAdapter(...)`, qui peut importer votre module runtime à la demande tout en laissant le core assembler le cycle de vie des approbations.
- Utilisez `approvalCapability.render` uniquement lorsqu’un canal a réellement besoin de payloads d’approbation personnalisés au lieu du moteur de rendu partagé.
- Utilisez `approvalCapability.describeExecApprovalSetup` lorsque le canal veut que la réponse du chemin désactivé explique les paramètres de configuration exacts nécessaires pour activer les approbations d’exécution natives. Le hook reçoit `{ channel, channelLabel, accountId }` ; les canaux à compte nommé doivent afficher des chemins à portée de compte tels que `channels.<channel>.accounts.<id>.execApprovals.*` au lieu de valeurs par défaut de niveau supérieur.
- Si un canal peut déduire des identités DM stables de type propriétaire à partir de la configuration existante, utilisez `createResolvedApproverActionAuthAdapter` depuis `openclaw/plugin-sdk/approval-runtime` pour restreindre `/approve` dans le même chat sans ajouter de logique core spécifique aux approbations.
- Si un canal a besoin d’une livraison d’approbation native, gardez le code du canal centré sur la normalisation de la cible ainsi que sur les faits de transport/présentation. Utilisez `createChannelExecApprovalProfile`, `createChannelNativeOriginTargetResolver`, `createChannelApproverDmTargetResolver`, et `createApproverRestrictedNativeApprovalCapability` depuis `openclaw/plugin-sdk/approval-runtime`. Placez les faits spécifiques au canal derrière `approvalCapability.nativeRuntime`, idéalement via `createChannelApprovalNativeRuntimeAdapter(...)` ou `createLazyChannelApprovalNativeRuntimeAdapter(...)`, afin que le core puisse assembler le gestionnaire et posséder le filtrage des requêtes, le routage, la déduplication, l’expiration, l’abonnement gateway, et les avis « routed elsewhere ». `nativeRuntime` est divisé en quelques coutures plus petites :
- `availability` — si le compte est configuré et si une requête doit être traitée
- `presentation` — mapper le modèle de vue d’approbation partagé en payloads natifs en attente/résolus/expirés ou en actions finales
- `transport` — préparer les cibles plus envoyer/mettre à jour/supprimer les messages d’approbation natifs
- `interactions` — hooks optionnels de bind/unbind/clear-action pour les boutons ou réactions natifs
- `observe` — hooks optionnels de diagnostic de livraison
- Si le canal a besoin d’objets possédés par le runtime comme un client, un jeton, une app Bolt, ou un récepteur webhook, enregistrez-les via `openclaw/plugin-sdk/channel-runtime-context`. Le registre générique de contexte runtime permet au core de bootstrapper des gestionnaires pilotés par capacité à partir de l’état de démarrage du canal sans ajouter de glue wrapper spécifique aux approbations.
- Recourez à `createChannelApprovalHandler` ou `createChannelNativeApprovalRuntime` de niveau inférieur uniquement lorsque la couture pilotée par capacité n’est pas encore assez expressive.
- Les canaux d’approbation native doivent router à la fois `accountId` et `approvalKind` via ces helpers. `accountId` conserve la portée de la politique d’approbation multi-comptes sur le bon compte bot, et `approvalKind` conserve le comportement d’approbation d’exécution vs plugin disponible pour le canal sans branches codées en dur dans le core.
- Le core possède désormais aussi les avis de reroutage d’approbation. Les plugins de canal ne doivent pas envoyer leurs propres messages de suivi « approval went to DMs / another channel » depuis `createChannelNativeApprovalRuntime` ; exposez plutôt un routage précis de l’origine + DM de l’approbateur via les helpers partagés de capacité d’approbation et laissez le core agréger les livraisons réelles avant de publier tout avis dans le chat initiateur.
- Préservez le type d’id d’approbation livré de bout en bout. Les clients natifs ne doivent pas
  deviner ni réécrire le routage d’approbation d’exécution vs plugin à partir d’un état local au canal.
- Différents types d’approbation peuvent intentionnellement exposer différentes surfaces natives.
  Exemples intégrés actuels :
  - Slack conserve le routage d’approbation natif disponible pour les ids d’exécution et de plugin.
  - Matrix conserve le même routage DM/canal natif et la même UX par réaction pour les approbations d’exécution
    et de plugin, tout en permettant à l’authentification de différer selon le type d’approbation.
- `createApproverRestrictedNativeApprovalAdapter` existe toujours comme wrapper de compatibilité, mais le nouveau code doit préférer le builder de capacité et exposer `approvalCapability` sur le plugin.

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
`openclaw/plugin-sdk/reply-reference`, et
`openclaw/plugin-sdk/reply-chunking` lorsque vous n’avez pas besoin de la surface
ombrelle plus large.

Pour la configuration en particulier :

- `openclaw/plugin-sdk/setup-runtime` couvre les helpers de configuration sûrs au runtime :
  adaptateurs de patch de configuration import-safe (`createPatchedAccountSetupAdapter`,
  `createEnvPatchedAccountSetupAdapter`,
  `createSetupInputPresenceValidator`), sortie de note de lookup,
  `promptResolvedAllowFrom`, `splitSetupEntries`, et les builders
  délégués de setup-proxy
- `openclaw/plugin-sdk/setup-adapter-runtime` est la couture étroite d’adaptateur sensible à l’environnement
  pour `createEnvPatchedAccountSetupAdapter`
- `openclaw/plugin-sdk/channel-setup` couvre les builders de configuration à installation optionnelle
  ainsi que quelques primitives sûres pour la configuration :
  `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`,

Si votre canal prend en charge une configuration ou une authentification pilotée par l’environnement et que les flux génériques de démarrage/configuration
doivent connaître ces noms de variables d’environnement avant le chargement du runtime, déclarez-les dans le
manifeste du plugin avec `channelEnvVars`. Conservez les `envVars` runtime du canal ou les constantes locales
uniquement pour le texte destiné aux opérateurs.
`createOptionalChannelSetupWizard`, `DEFAULT_ACCOUNT_ID`,
`createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, et
`splitSetupEntries`

- utilisez la couture plus large `openclaw/plugin-sdk/setup` uniquement lorsque vous avez aussi besoin des helpers partagés plus lourds de configuration/setup tels que
  `moveSingleAccountChannelSectionToDefaultAccount(...)`

Si votre canal veut seulement annoncer « installez d’abord ce plugin » dans les
surfaces de configuration, préférez `createOptionalChannelSetupSurface(...)`. L’adaptateur/l’assistant généré
échoue de manière fermée sur les écritures de configuration et la finalisation, et ils réutilisent
le même message d’installation requise dans la validation, la finalisation, et le texte du lien vers la documentation.

Pour les autres chemins chauds du canal, préférez les helpers étroits aux surfaces héritées plus larges :

- `openclaw/plugin-sdk/account-core`,
  `openclaw/plugin-sdk/account-id`,
  `openclaw/plugin-sdk/account-resolution`, et
  `openclaw/plugin-sdk/account-helpers` pour la configuration multi-comptes et
  le fallback de compte par défaut
- `openclaw/plugin-sdk/inbound-envelope` et
  `openclaw/plugin-sdk/inbound-reply-dispatch` pour le câblage de route/enveloppe entrante
  et d’enregistrement-et-répartition
- `openclaw/plugin-sdk/messaging-targets` pour l’analyse/la correspondance des cibles
- `openclaw/plugin-sdk/outbound-media` et
  `openclaw/plugin-sdk/outbound-runtime` pour le chargement des médias ainsi que les délégués
  d’identité/envoi sortants
- `openclaw/plugin-sdk/thread-bindings-runtime` pour le cycle de vie des liaisons de thread
  et l’enregistrement des adaptateurs
- `openclaw/plugin-sdk/agent-media-payload` uniquement lorsqu’une disposition héritée des champs
  de payload agent/média est encore requise
- `openclaw/plugin-sdk/telegram-command-config` pour la normalisation des commandes personnalisées Telegram,
  la validation des doublons/conflits, et un contrat de configuration de commandes
  stable en fallback

Les canaux d’authentification seule peuvent généralement s’arrêter au chemin par défaut : le core gère les approbations et le plugin expose simplement des capacités outbound/auth. Les canaux d’approbation native comme Matrix, Slack, Telegram, et les transports de chat personnalisés doivent utiliser les helpers natifs partagés au lieu de développer leur propre cycle de vie d’approbation.

## Politique de mention entrante

Conservez la gestion des mentions entrantes séparée en deux couches :

- collecte des preuves appartenant au plugin
- évaluation partagée de la politique

Utilisez `openclaw/plugin-sdk/channel-inbound` pour la couche partagée.

Bon cas d’usage pour la logique locale au plugin :

- détection des réponses au bot
- détection des citations du bot
- vérifications de participation au thread
- exclusions des messages de service/système
- caches natifs à la plateforme nécessaires pour prouver la participation du bot

Bon cas d’usage pour le helper partagé :

- `requireMention`
- résultat de mention explicite
- liste d’autorisation de mention implicite
- contournement de commande
- décision finale d’ignorer

Flux recommandé :

1. Calculez les faits de mention locaux.
2. Transmettez ces faits à `resolveInboundMentionDecision({ facts, policy })`.
3. Utilisez `decision.effectiveWasMentioned`, `decision.shouldBypassMention`, et `decision.shouldSkip` dans votre garde d’entrée.

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

`api.runtime.channel.mentions` expose les mêmes helpers partagés de mention pour
les plugins de canal intégrés qui dépendent déjà de l’injection runtime :

- `buildMentionRegexes`
- `matchesMentionPatterns`
- `matchesMentionWithExplicit`
- `implicitMentionKindWhen`
- `resolveInboundMentionDecision`

Les anciens helpers `resolveMentionGating*` restent sur
`openclaw/plugin-sdk/channel-inbound` uniquement comme exports de compatibilité. Le nouveau code
doit utiliser `resolveInboundMentionDecision({ facts, policy })`.

## Procédure pas à pas

<Steps>
  <a id="step-1-package-and-manifest"></a>
  <Step title="Package et manifeste">
    Créez les fichiers de plugin standard. Le champ `channel` dans `package.json` est
    ce qui en fait un plugin de canal. Pour la surface complète des métadonnées de package,
    voir [Plugin Setup and Config](/fr/plugins/sdk-setup#openclaw-channel) :

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
      "description": "Plugin de canal Acme Chat",
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

  <Step title="Construire l’objet plugin de canal">
    L’interface `ChannelPlugin` dispose de nombreuses surfaces d’adaptateur optionnelles. Commencez par
    le minimum — `id` et `setup` — puis ajoutez des adaptateurs selon vos besoins.

    Créez `src/channel.ts` :

    ```typescript src/channel.ts
    import {
      createChatChannelPlugin,
      createChannelPluginBase,
    } from "openclaw/plugin-sdk/channel-core";
    import type { OpenClawConfig } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatApi } from "./client.js"; // votre client API de plateforme

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

      // Sécurité DM : qui peut envoyer un message au bot
      security: {
        dm: {
          channelKey: "acme-chat",
          resolvePolicy: (account) => account.dmPolicy,
          resolveAllowFrom: (account) => account.allowFrom,
          defaultPolicy: "allowlist",
        },
      },

      // Appairage : flux d’approbation pour les nouveaux contacts DM
      pairing: {
        text: {
          idLabel: "Nom d’utilisateur Acme Chat",
          message: "Envoyez ce code pour vérifier votre identité :",
          notify: async ({ target, code }) => {
            await acmeChatApi.sendDm(target, `Pairing code: ${code}`);
          },
        },
      },

      // Threading : comment les réponses sont livrées
      threading: { topLevelReplyToMode: "reply" },

      // Outbound : envoyer des messages vers la plateforme
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
      Au lieu d’implémenter manuellement des interfaces d’adaptateur de bas niveau, vous transmettez
      des options déclaratives et le builder les compose :

      | Option | Ce qu’elle câble |
      | --- | --- |
      | `security.dm` | Résolveur de sécurité DM à portée, dérivé des champs de configuration |
      | `pairing.text` | Flux d’appairage DM basé sur du texte avec échange de code |
      | `threading` | Résolveur de mode de réponse threadée (fixe, à portée de compte, ou personnalisé) |
      | `outbound.attachedResults` | Fonctions d’envoi qui renvoient des métadonnées de résultat (ids de message) |

      Vous pouvez aussi transmettre des objets d’adaptateur bruts à la place des options déclaratives
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
      description: "Plugin de canal Acme Chat",
      plugin: acmeChatPlugin,
      registerCliMetadata(api) {
        api.registerCli(
          ({ program }) => {
            program
              .command("acme-chat")
              .description("Gestion Acme Chat");
          },
          {
            descriptors: [
              {
                name: "acme-chat",
                description: "Gestion Acme Chat",
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

    Placez les descripteurs CLI appartenant au canal dans `registerCliMetadata(...)` afin qu’OpenClaw
    puisse les afficher dans l’aide racine sans activer le runtime complet du canal,
    tandis que les chargements complets normaux récupèrent toujours les mêmes descripteurs pour le véritable enregistrement des commandes.
    Conservez `registerFull(...)` pour le travail réservé au runtime.
    Si `registerFull(...)` enregistre des méthodes RPC gateway, utilisez un
    préfixe spécifique au plugin. Les espaces de noms d’administration core (`config.*`,
    `exec.approvals.*`, `wizard.*`, `update.*`) restent réservés et
    se résolvent toujours vers `operator.admin`.
    `defineChannelPluginEntry` gère automatiquement la séparation des modes d’enregistrement. Voir
    [Entry Points](/fr/plugins/sdk-entrypoints#definechannelpluginentry) pour toutes les
    options.

  </Step>

  <Step title="Ajouter une entrée de configuration">
    Créez `setup-entry.ts` pour un chargement léger pendant l’onboarding :

    ```typescript setup-entry.ts
    import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineSetupPluginEntry(acmeChatPlugin);
    ```

    OpenClaw charge ceci au lieu du point d’entrée complet lorsque le canal est désactivé
    ou non configuré. Cela évite de charger du code runtime lourd pendant les flux de configuration.
    Voir [Setup and Config](/fr/plugins/sdk-setup#setup-entry) pour plus de détails.

  </Step>

  <Step title="Gérer les messages entrants">
    Votre plugin doit recevoir les messages de la plateforme et les transmettre à
    OpenClaw. Le modèle typique est un webhook qui vérifie la requête et
    la répartit via le gestionnaire entrant de votre canal :

    ```typescript
    registerFull(api) {
      api.registerHttpRoute({
        path: "/acme-chat/webhook",
        auth: "plugin", // authentification gérée par le plugin (vérifiez vous-même les signatures)
        handler: async (req, res) => {
          const event = parseWebhookPayload(req);

          // Votre gestionnaire entrant répartit le message vers OpenClaw.
          // Le câblage exact dépend de votre SDK de plateforme —
          // voir un exemple réel dans le package de plugin Microsoft Teams ou Google Chat intégré.
          await handleAcmeChatInbound(api, event);

          res.statusCode = 200;
          res.end("ok");
          return true;
        },
      });
    }
    ```

    <Note>
      La gestion des messages entrants est spécifique à chaque canal. Chaque plugin de canal possède
      son propre pipeline entrant. Regardez les plugins de canal intégrés
      (par exemple le package de plugin Microsoft Teams ou Google Chat) pour voir des modèles réels.
    </Note>

  </Step>

<a id="step-6-test"></a>
<Step title="Tester">
Écrivez des tests colocalisés dans `src/channel.test.ts` :

    ```typescript src/channel.test.ts
    import { describe, it, expect } from "vitest";
    import { acmeChatPlugin } from "./channel.js";

    describe("plugin acme-chat", () => {
      it("résout le compte à partir de la configuration", () => {
        const cfg = {
          channels: {
            "acme-chat": { token: "test-token", allowFrom: ["user1"] },
          },
        } as any;
        const account = acmeChatPlugin.setup!.resolveAccount(cfg, undefined);
        expect(account.token).toBe("test-token");
      });

      it("inspecte le compte sans matérialiser les secrets", () => {
        const cfg = {
          channels: { "acme-chat": { token: "test-token" } },
        } as any;
        const result = acmeChatPlugin.setup!.inspectAccount!(cfg, undefined);
        expect(result.configured).toBe(true);
        expect(result.tokenStatus).toBe("available");
      });

      it("signale une configuration manquante", () => {
        const cfg = { channels: {} } as any;
        const result = acmeChatPlugin.setup!.inspectAccount!(cfg, undefined);
        expect(result.configured).toBe(false);
      });
    });
    ```

    ```bash
    pnpm test -- <bundled-plugin-root>/acme-chat/
    ```

    Pour les helpers de test partagés, voir [Testing](/fr/plugins/sdk-testing).

  </Step>
</Steps>

## Structure des fichiers

```text
<bundled-plugin-root>/acme-chat/
├── package.json              # métadonnées openclaw.channel
├── openclaw.plugin.json      # manifeste avec schéma de configuration
├── index.ts                  # defineChannelPluginEntry
├── setup-entry.ts            # defineSetupPluginEntry
├── api.ts                    # exports publics (optionnel)
├── runtime-api.ts            # exports runtime internes (optionnel)
└── src/
    ├── channel.ts            # ChannelPlugin via createChatChannelPlugin
    ├── channel.test.ts       # tests
    ├── client.ts             # client API de la plateforme
    └── runtime.ts            # store runtime (si nécessaire)
```

## Sujets avancés

<CardGroup cols={2}>
  <Card title="Options de threading" icon="git-branch" href="/fr/plugins/sdk-entrypoints#registration-mode">
    Modes de réponse fixes, à portée de compte, ou personnalisés
  </Card>
  <Card title="Intégration de l’outil message" icon="puzzle" href="/fr/plugins/architecture#channel-plugins-and-the-shared-message-tool">
    describeMessageTool et découverte d’actions
  </Card>
  <Card title="Résolution de cible" icon="crosshair" href="/fr/plugins/architecture#channel-target-resolution">
    inferTargetChatType, looksLikeId, resolveTarget
  </Card>
  <Card title="Helpers runtime" icon="settings" href="/fr/plugins/sdk-runtime">
    TTS, STT, média, sous-agent via api.runtime
  </Card>
</CardGroup>

<Note>
Certaines coutures helper intégrées existent encore pour la maintenance des plugins intégrés et
la compatibilité. Elles ne constituent pas le modèle recommandé pour les nouveaux plugins de canal ;
préférez les sous-chemins génériques channel/setup/reply/runtime de la surface
commune du SDK, sauf si vous maintenez directement cette famille de plugins intégrés.
</Note>

## Étapes suivantes

- [Provider Plugins](/fr/plugins/sdk-provider-plugins) — si votre plugin fournit aussi des modèles
- [SDK Overview](/fr/plugins/sdk-overview) — référence complète des imports de sous-chemins
- [SDK Testing](/fr/plugins/sdk-testing) — utilitaires de test et tests de contrat
- [Plugin Manifest](/fr/plugins/manifest) — schéma complet du manifeste
