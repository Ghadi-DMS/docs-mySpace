---
read_when:
    - Vous créez un nouveau plugin de canal de messagerie
    - Vous voulez connecter OpenClaw à une plateforme de messagerie
    - Vous devez comprendre la surface d'adaptation ChannelPlugin
sidebarTitle: Channel Plugins
summary: Guide pas à pas pour créer un plugin de canal de messagerie pour OpenClaw
title: Créer des plugins de canal
x-i18n:
    generated_at: "2026-04-07T06:52:24Z"
    model: gpt-5.4
    provider: openai
    source_hash: 25ac0591d9b0ba401925b29ae4b9572f18b2cbffc2b6ca6ed5252740e7cf97e9
    source_path: plugins/sdk-channel-plugins.md
    workflow: 15
---

# Créer des plugins de canal

Ce guide vous accompagne dans la création d'un plugin de canal qui connecte OpenClaw à une
plateforme de messagerie. À la fin, vous disposerez d'un canal fonctionnel avec sécurité des messages privés,
appairage, fils de réponse et messagerie sortante.

<Info>
  Si vous n'avez encore créé aucun plugin OpenClaw, lisez d'abord
  [Getting Started](/fr/plugins/building-plugins) pour la structure de package
  de base et la configuration du manifeste.
</Info>

## Fonctionnement des plugins de canal

Les plugins de canal n'ont pas besoin de leurs propres outils send/edit/react. OpenClaw conserve un
outil `message` partagé dans le cœur. Votre plugin gère :

- **Config** — résolution de compte et assistant de configuration
- **Sécurité** — politique des messages privés et allowlists
- **Appairage** — flux d'approbation des messages privés
- **Grammaire de session** — comment les identifiants de conversation spécifiques au fournisseur sont mappés vers les discussions de base, les ID de fil et les replis parents
- **Sortant** — envoi de texte, médias et sondages vers la plateforme
- **Fils** — comment les réponses sont organisées en fils

Le cœur gère l'outil de message partagé, le câblage des prompts, la forme externe de la clé de session,
la gestion générique `:thread:` et la répartition.

Si votre plateforme stocke une portée supplémentaire dans les identifiants de conversation, conservez cette analyse
dans le plugin avec `messaging.resolveSessionConversation(...)`. C'est le hook canonique pour mapper `rawId` vers l'identifiant de conversation de base, l'ID de fil facultatif, `baseConversationId` explicite et d'éventuels `parentConversationCandidates`.
Lorsque vous renvoyez `parentConversationCandidates`, conservez-les ordonnés du
parent le plus spécifique au parent le plus large / à la conversation de base.

Les plugins groupés qui ont besoin de la même analyse avant le démarrage du registre de canaux
peuvent aussi exposer un fichier `session-key-api.ts` de niveau supérieur avec un export
`resolveSessionConversation(...)` correspondant. Le cœur utilise cette surface sûre pour l'amorçage
uniquement lorsque le registre de plugins d'exécution n'est pas encore disponible.

`messaging.resolveParentConversationCandidates(...)` reste disponible comme
repli de compatibilité hérité lorsqu'un plugin n'a besoin de replis parents qu'au-dessus
de l'identifiant générique / brut. Si les deux hooks existent, le cœur utilise
d'abord `resolveSessionConversation(...).parentConversationCandidates` et ne se
rabat sur `resolveParentConversationCandidates(...)` que lorsque le hook canonique
les omet.

## Approbations et capacités du canal

La plupart des plugins de canal n'ont pas besoin de code spécifique aux approbations.

- Le cœur gère `/approve` dans la même discussion, les charges utiles de bouton d'approbation partagées et la livraison générique de repli.
- Préférez un seul objet `approvalCapability` sur le plugin de canal lorsque le canal a besoin d'un comportement spécifique aux approbations.
- `approvalCapability.authorizeActorAction` et `approvalCapability.getActionAvailabilityState` sont la couture canonique d'authentification des approbations.
- Si votre canal expose des approbations exec natives, implémentez `approvalCapability.getActionAvailabilityState` même lorsque le transport natif vit entièrement sous `approvalCapability.native`. Le cœur utilise ce hook de disponibilité pour distinguer `enabled` de `disabled`, déterminer si le canal initiateur prend en charge les approbations natives et inclure le canal dans les indications de repli de client natif.
- Utilisez `outbound.shouldSuppressLocalPayloadPrompt` ou `outbound.beforeDeliverPayload` pour le comportement spécifique au canal dans le cycle de vie des charges utiles, par exemple masquer les invites locales d'approbation en double ou envoyer des indicateurs de frappe avant la livraison.
- Utilisez `approvalCapability.delivery` uniquement pour le routage natif des approbations ou la suppression du repli.
- Utilisez `approvalCapability.render` uniquement lorsqu'un canal a réellement besoin de charges utiles d'approbation personnalisées au lieu du moteur de rendu partagé.
- Utilisez `approvalCapability.describeExecApprovalSetup` lorsque le canal veut que la réponse du chemin désactivé explique les clés de configuration exactes nécessaires pour activer les approbations exec natives. Le hook reçoit `{ channel, channelLabel, accountId }` ; les canaux à comptes nommés doivent afficher des chemins ciblés par compte comme `channels.<channel>.accounts.<id>.execApprovals.*` au lieu des valeurs par défaut de niveau supérieur.
- Si un canal peut déduire des identités de type propriétaire stables en message privé à partir de la configuration existante, utilisez `createResolvedApproverActionAuthAdapter` depuis `openclaw/plugin-sdk/approval-runtime` pour restreindre `/approve` dans la même discussion sans ajouter de logique d'approbation spécifique dans le cœur.
- Si un canal a besoin d'une livraison native des approbations, gardez le code du canal concentré sur la normalisation des cibles et les hooks de transport. Utilisez `createChannelExecApprovalProfile`, `createChannelNativeOriginTargetResolver`, `createChannelApproverDmTargetResolver`, `createApproverRestrictedNativeApprovalCapability` et `createChannelNativeApprovalRuntime` depuis `openclaw/plugin-sdk/approval-runtime` afin que le cœur gère le filtrage des requêtes, le routage, la déduplication, l'expiration et l'abonnement à la passerelle.
- Les canaux d'approbation natifs doivent acheminer à la fois `accountId` et `approvalKind` via ces helpers. `accountId` permet de garder la politique d'approbation multi-comptes ciblée sur le bon compte de bot, et `approvalKind` permet de rendre disponible le comportement d'approbation exec vs plugin au canal sans branches codées en dur dans le cœur.
- Préservez le type d'identifiant d'approbation livré de bout en bout. Les clients natifs ne doivent pas
  deviner ni réécrire le routage d'approbation exec vs plugin à partir d'un état local au canal.
- Des types d'approbation différents peuvent volontairement exposer des surfaces natives différentes.
  Exemples groupés actuels :
  - Slack conserve le routage natif des approbations disponible à la fois pour les identifiants exec et plugin.
  - Matrix conserve le routage natif en message privé / canal pour les approbations exec uniquement et laisse
    les approbations de plugin sur le chemin partagé `/approve` dans la même discussion.
- `createApproverRestrictedNativeApprovalAdapter` existe toujours comme wrapper de compatibilité, mais le nouveau code devrait préférer le générateur de capacité et exposer `approvalCapability` sur le plugin.

Pour les points d'entrée de canal sensibles aux performances, préférez les sous-chemins d'exécution plus étroits lorsque vous n'avez besoin que d'une partie de cette famille :

- `openclaw/plugin-sdk/approval-auth-runtime`
- `openclaw/plugin-sdk/approval-client-runtime`
- `openclaw/plugin-sdk/approval-delivery-runtime`
- `openclaw/plugin-sdk/approval-native-runtime`
- `openclaw/plugin-sdk/approval-reply-runtime`

De même, préférez `openclaw/plugin-sdk/setup-runtime`,
`openclaw/plugin-sdk/setup-adapter-runtime`,
`openclaw/plugin-sdk/reply-runtime`,
`openclaw/plugin-sdk/reply-dispatch-runtime`,
`openclaw/plugin-sdk/reply-reference` et
`openclaw/plugin-sdk/reply-chunking` lorsque vous n'avez pas besoin de la surface
ombrelle plus large.

Pour la configuration initiale en particulier :

- `openclaw/plugin-sdk/setup-runtime` couvre les helpers de configuration sûrs à l'exécution :
  adaptateurs de patch de configuration sûrs à l'import (`createPatchedAccountSetupAdapter`,
  `createEnvPatchedAccountSetupAdapter`,
  `createSetupInputPresenceValidator`), sortie de note de recherche,
  `promptResolvedAllowFrom`, `splitSetupEntries` et les constructeurs
  délégués de proxy de configuration
- `openclaw/plugin-sdk/setup-adapter-runtime` est la couture étroite d'adaptateur
  orientée environnement pour `createEnvPatchedAccountSetupAdapter`
- `openclaw/plugin-sdk/channel-setup` couvre les constructeurs de configuration pour installation facultative
  ainsi que quelques primitives sûres pour la configuration :
  `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`,

Si votre canal prend en charge une configuration ou une authentification pilotée par variables d'environnement et que les flux génériques de démarrage / configuration doivent connaître ces noms de variables d'environnement avant le chargement de l'exécution, déclarez-les dans le manifeste du plugin avec `channelEnvVars`. Conservez les `envVars` de l'exécution du canal ou des constantes locales uniquement pour le texte destiné aux opérateurs.
`createOptionalChannelSetupWizard`, `DEFAULT_ACCOUNT_ID`,
`createTopLevelChannelDmPolicy`, `setSetupChannelEnabled` et
`splitSetupEntries`

- utilisez la couture plus large `openclaw/plugin-sdk/setup` uniquement lorsque vous avez aussi besoin des helpers partagés plus lourds de configuration / config, tels que
  `moveSingleAccountChannelSectionToDefaultAccount(...)`

Si votre canal veut simplement annoncer « installez d'abord ce plugin » dans les surfaces de configuration, préférez `createOptionalChannelSetupSurface(...)`. L'adaptateur / assistant généré échoue en mode fermé sur les écritures de configuration et la finalisation, et réutilise le même message « installation requise » dans la validation, la finalisation et le texte des liens de documentation.

Pour les autres chemins de canal sensibles aux performances, préférez les helpers étroits aux surfaces héritées plus larges :

- `openclaw/plugin-sdk/account-core`,
  `openclaw/plugin-sdk/account-id`,
  `openclaw/plugin-sdk/account-resolution` et
  `openclaw/plugin-sdk/account-helpers` pour la configuration multi-comptes et
  le repli vers le compte par défaut
- `openclaw/plugin-sdk/inbound-envelope` et
  `openclaw/plugin-sdk/inbound-reply-dispatch` pour le câblage des routes / enveloppes entrantes et
  l'enregistrement et la répartition
- `openclaw/plugin-sdk/messaging-targets` pour l'analyse / la correspondance des cibles
- `openclaw/plugin-sdk/outbound-media` et
  `openclaw/plugin-sdk/outbound-runtime` pour le chargement des médias ainsi que les délégués d'identité / d'envoi sortants
- `openclaw/plugin-sdk/thread-bindings-runtime` pour le cycle de vie des liaisons de fils
  et l'enregistrement d'adaptateurs
- `openclaw/plugin-sdk/agent-media-payload` uniquement lorsqu'une ancienne disposition de champ de charge utile agent / média est encore requise
- `openclaw/plugin-sdk/telegram-command-config` pour la normalisation des commandes personnalisées Telegram,
  la validation des doublons / conflits et un contrat de configuration de commande
  stable en repli

Les canaux uniquement orientés authentification peuvent généralement s'arrêter au chemin par défaut : le cœur gère les approbations et le plugin expose simplement les capacités sortantes / d'authentification. Les canaux d'approbation natifs tels que Matrix, Slack, Telegram et les transports de discussion personnalisés devraient utiliser les helpers natifs partagés au lieu de développer leur propre cycle de vie d'approbation.

## Procédure pas à pas

<Steps>
  <a id="step-1-package-and-manifest"></a>
  <Step title="Package et manifeste">
    Créez les fichiers standards du plugin. Le champ `channel` dans `package.json` est
    ce qui en fait un plugin de canal. Pour la surface complète des métadonnées de package,
    consultez [Plugin Setup and Config](/fr/plugins/sdk-setup#openclawchannel) :

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

  <Step title="Créer l'objet plugin de canal">
    L'interface `ChannelPlugin` possède de nombreuses surfaces d'adaptation facultatives. Commencez par
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

      // Sécurité des messages privés : qui peut envoyer un message au bot
      security: {
        dm: {
          channelKey: "acme-chat",
          resolvePolicy: (account) => account.dmPolicy,
          resolveAllowFrom: (account) => account.allowFrom,
          defaultPolicy: "allowlist",
        },
      },

      // Appairage : flux d'approbation pour les nouveaux contacts en message privé
      pairing: {
        text: {
          idLabel: "Nom d'utilisateur Acme Chat",
          message: "Envoyez ce code pour vérifier votre identité :",
          notify: async ({ target, code }) => {
            await acmeChatApi.sendDm(target, `Pairing code: ${code}`);
          },
        },
      },

      // Fils : comment les réponses sont livrées
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
      Au lieu d'implémenter manuellement des interfaces d'adaptation de bas niveau, vous transmettez
      des options déclaratives et le constructeur les compose :

      | Option | Ce qu'elle câble |
      | --- | --- |
      | `security.dm` | Résolveur de sécurité de message privé ciblé depuis les champs de configuration |
      | `pairing.text` | Flux d'appairage de message privé basé sur texte avec échange de code |
      | `threading` | Résolveur de mode de réponse (fixe, ciblé par compte ou personnalisé) |
      | `outbound.attachedResults` | Fonctions d'envoi qui renvoient des métadonnées de résultat (ID de message) |

      Vous pouvez aussi transmettre des objets d'adaptateur bruts au lieu des options déclaratives
      si vous avez besoin d'un contrôle total.
    </Accordion>

  </Step>

  <Step title="Relier le point d'entrée">
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

    Placez les descripteurs CLI appartenant au canal dans `registerCliMetadata(...)` afin qu'OpenClaw
    puisse les afficher dans l'aide racine sans activer l'exécution complète du canal,
    tandis que les chargements complets normaux récupèrent toujours les mêmes descripteurs pour l'enregistrement réel des commandes.
    Conservez `registerFull(...)` pour le travail réservé à l'exécution.
    Si `registerFull(...)` enregistre des méthodes RPC de passerelle, utilisez un préfixe
    spécifique au plugin. Les espaces de noms d'administration du cœur (`config.*`,
    `exec.approvals.*`, `wizard.*`, `update.*`) restent réservés et se
    résolvent toujours vers `operator.admin`.
    `defineChannelPluginEntry` gère automatiquement la séparation des modes d'enregistrement. Consultez
    [Entry Points](/fr/plugins/sdk-entrypoints#definechannelpluginentry) pour toutes les
    options.

  </Step>

  <Step title="Ajouter un point d'entrée de configuration">
    Créez `setup-entry.ts` pour un chargement léger pendant l'onboarding :

    ```typescript setup-entry.ts
    import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineSetupPluginEntry(acmeChatPlugin);
    ```

    OpenClaw charge cela au lieu du point d'entrée complet lorsque le canal est désactivé
    ou non configuré. Cela évite de charger du code d'exécution lourd pendant les flux de configuration.
    Consultez [Setup and Config](/fr/plugins/sdk-setup#setup-entry) pour plus de détails.

  </Step>

  <Step title="Gérer les messages entrants">
    Votre plugin doit recevoir les messages de la plateforme et les transférer vers
    OpenClaw. Le modèle typique est un webhook qui vérifie la requête et
    la répartit via le gestionnaire entrant de votre canal :

    ```typescript
    registerFull(api) {
      api.registerHttpRoute({
        path: "/acme-chat/webhook",
        auth: "plugin", // auth gérée par le plugin (vérifiez vous-même les signatures)
        handler: async (req, res) => {
          const event = parseWebhookPayload(req);

          // Votre gestionnaire entrant répartit le message vers OpenClaw.
          // Le câblage exact dépend du SDK de votre plateforme —
          // voir un exemple réel dans le package de plugin groupé Microsoft Teams ou Google Chat.
          await handleAcmeChatInbound(api, event);

          res.statusCode = 200;
          res.end("ok");
          return true;
        },
      });
    }
    ```

    <Note>
      La gestion des messages entrants est spécifique au canal. Chaque plugin de canal gère
      son propre pipeline entrant. Regardez les plugins de canal groupés
      (par exemple le package de plugin Microsoft Teams ou Google Chat) pour des modèles réels.
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
├── api.ts                    # Exports publics (facultatif)
├── runtime-api.ts            # Exports d'exécution internes (facultatif)
└── src/
    ├── channel.ts            # ChannelPlugin via createChatChannelPlugin
    ├── channel.test.ts       # Tests
    ├── client.ts             # Client API de plateforme
    └── runtime.ts            # Magasin d'exécution (si nécessaire)
```

## Sujets avancés

<CardGroup cols={2}>
  <Card title="Options de fil" icon="git-branch" href="/fr/plugins/sdk-entrypoints#registration-mode">
    Modes de réponse fixes, ciblés par compte ou personnalisés
  </Card>
  <Card title="Intégration de l'outil de message" icon="puzzle" href="/fr/plugins/architecture#channel-plugins-and-the-shared-message-tool">
    describeMessageTool et découverte des actions
  </Card>
  <Card title="Résolution de cible" icon="crosshair" href="/fr/plugins/architecture#channel-target-resolution">
    inferTargetChatType, looksLikeId, resolveTarget
  </Card>
  <Card title="Helpers d'exécution" icon="settings" href="/fr/plugins/sdk-runtime">
    TTS, STT, médias, sous-agent via api.runtime
  </Card>
</CardGroup>

<Note>
Certaines coutures de helpers groupés existent encore pour la maintenance et la
compatibilité des plugins groupés. Ce n'est pas le modèle recommandé pour les nouveaux plugins de canal ;
préférez les sous-chemins génériques channel/setup/reply/runtime de la surface SDK commune
sauf si vous maintenez directement cette famille de plugins groupés.
</Note>

## Étapes suivantes

- [Provider Plugins](/fr/plugins/sdk-provider-plugins) — si votre plugin fournit aussi des modèles
- [SDK Overview](/fr/plugins/sdk-overview) — référence complète des imports par sous-chemin
- [SDK Testing](/fr/plugins/sdk-testing) — utilitaires de test et tests de contrat
- [Plugin Manifest](/fr/plugins/manifest) — schéma complet du manifeste
