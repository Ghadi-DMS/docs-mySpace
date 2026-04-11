---
read_when:
    - Utilisation ou configuration des commandes de chat
    - Débogage du routage des commandes ou des permissions
summary: 'Commandes slash : texte vs natif, configuration et commandes prises en charge'
title: Commandes slash
x-i18n:
    generated_at: "2026-04-11T02:48:06Z"
    model: gpt-5.4
    provider: openai
    source_hash: 2cc346361c3b1a63aae9ec0f28706f4cb0b866b6c858a3999101f6927b923b4a
    source_path: tools/slash-commands.md
    workflow: 15
---

# Commandes slash

Les commandes sont gérées par le Gateway. La plupart des commandes doivent être envoyées comme message **autonome** commençant par `/`.
La commande de chat bash réservée à l’hôte utilise `! <cmd>` (avec `/bash <cmd>` comme alias).

Il existe deux systèmes liés :

- **Commandes** : messages autonomes `/...`.
- **Directives** : `/think`, `/fast`, `/verbose`, `/reasoning`, `/elevated`, `/exec`, `/model`, `/queue`.
  - Les directives sont supprimées du message avant que le modèle ne le voie.
  - Dans les messages de chat normaux (pas uniquement des directives), elles sont traitées comme des « indications inline » et ne conservent **pas** les paramètres de session.
  - Dans les messages contenant uniquement des directives (le message ne contient que des directives), elles sont conservées dans la session et renvoient un accusé de réception.
  - Les directives ne sont appliquées que pour les **expéditeurs autorisés**. Si `commands.allowFrom` est défini, c’est la seule allowlist utilisée ; sinon, l’autorisation vient des allowlists/appairages du canal ainsi que de `commands.useAccessGroups`.
    Les expéditeurs non autorisés voient les directives traitées comme du texte brut.

Il existe aussi quelques **raccourcis inline** (expéditeurs autorisés/en allowlist uniquement) : `/help`, `/commands`, `/status`, `/whoami` (`/id`).
Ils s’exécutent immédiatement, sont supprimés avant que le modèle ne voie le message, et le texte restant continue dans le flux normal.

## Configuration

```json5
{
  commands: {
    native: "auto",
    nativeSkills: "auto",
    text: true,
    bash: false,
    bashForegroundMs: 2000,
    config: false,
    mcp: false,
    plugins: false,
    debug: false,
    restart: true,
    ownerAllowFrom: ["discord:123456789012345678"],
    ownerDisplay: "raw",
    ownerDisplaySecret: "${OWNER_ID_HASH_SECRET}",
    allowFrom: {
      "*": ["user1"],
      discord: ["user:123"],
    },
    useAccessGroups: true,
  },
}
```

- `commands.text` (par défaut `true`) active l’analyse de `/...` dans les messages de chat.
  - Sur les surfaces sans commandes natives (WhatsApp/WebChat/Signal/iMessage/Google Chat/Microsoft Teams), les commandes texte fonctionnent toujours même si vous définissez cette valeur sur `false`.
- `commands.native` (par défaut `"auto"`) enregistre les commandes natives.
  - Auto : activé pour Discord/Telegram ; désactivé pour Slack (jusqu’à ce que vous ajoutiez des commandes slash) ; ignoré pour les providers sans prise en charge native.
  - Définissez `channels.discord.commands.native`, `channels.telegram.commands.native` ou `channels.slack.commands.native` pour remplacer le comportement par provider (booléen ou `"auto"`).
  - `false` efface les commandes précédemment enregistrées sur Discord/Telegram au démarrage. Les commandes Slack sont gérées dans l’application Slack et ne sont pas supprimées automatiquement.
- `commands.nativeSkills` (par défaut `"auto"`) enregistre les commandes de **Skills** nativement lorsqu’elles sont prises en charge.
  - Auto : activé pour Discord/Telegram ; désactivé pour Slack (Slack nécessite de créer une commande slash par skill).
  - Définissez `channels.discord.commands.nativeSkills`, `channels.telegram.commands.nativeSkills` ou `channels.slack.commands.nativeSkills` pour remplacer le comportement par provider (booléen ou `"auto"`).
- `commands.bash` (par défaut `false`) active `! <cmd>` pour exécuter des commandes shell sur l’hôte (`/bash <cmd>` est un alias ; nécessite les allowlists `tools.elevated`).
- `commands.bashForegroundMs` (par défaut `2000`) contrôle combien de temps bash attend avant de passer en mode arrière-plan (`0` passe immédiatement en arrière-plan).
- `commands.config` (par défaut `false`) active `/config` (lit/écrit `openclaw.json`).
- `commands.mcp` (par défaut `false`) active `/mcp` (lit/écrit la configuration MCP gérée par OpenClaw sous `mcp.servers`).
- `commands.plugins` (par défaut `false`) active `/plugins` (découverte/état des plugins plus contrôles d’installation et d’activation/désactivation).
- `commands.debug` (par défaut `false`) active `/debug` (remplacements réservés au runtime).
- `commands.restart` (par défaut `true`) active `/restart` ainsi que les actions d’outil de redémarrage du gateway.
- `commands.ownerAllowFrom` (facultatif) définit l’allowlist explicite du propriétaire pour les surfaces de commande/d’outil réservées au propriétaire. C’est distinct de `commands.allowFrom`.
- `commands.ownerDisplay` contrôle la manière dont les IDs du propriétaire apparaissent dans le prompt système : `raw` ou `hash`.
- `commands.ownerDisplaySecret` définit éventuellement le secret HMAC utilisé lorsque `commands.ownerDisplay="hash"`.
- `commands.allowFrom` (facultatif) définit une allowlist par provider pour l’autorisation des commandes. Lorsqu’elle est configurée, c’est la seule source d’autorisation pour les commandes et directives (les allowlists/appairages du canal et `commands.useAccessGroups` sont ignorés). Utilisez `"*"` pour une valeur par défaut globale ; les clés spécifiques aux providers la remplacent.
- `commands.useAccessGroups` (par défaut `true`) applique les allowlists/politiques pour les commandes lorsque `commands.allowFrom` n’est pas défini.

## Liste des commandes

Source de vérité actuelle :

- les commandes intégrées du cœur viennent de `src/auto-reply/commands-registry.shared.ts`
- les commandes dock générées viennent de `src/auto-reply/commands-registry.data.ts`
- les commandes de plugin viennent des appels `registerCommand()` des plugins
- la disponibilité réelle sur votre gateway dépend toujours des indicateurs de configuration, de la surface du canal et des plugins installés/activés

### Commandes intégrées du cœur

Commandes intégrées disponibles aujourd’hui :

- `/new [model]` démarre une nouvelle session ; `/reset` est l’alias de réinitialisation.
- `/compact [instructions]` compacte le contexte de session. Voir [/concepts/compaction](/fr/concepts/compaction).
- `/stop` interrompt l’exécution en cours.
- `/session idle <duration|off>` et `/session max-age <duration|off>` gèrent l’expiration de liaison du thread.
- `/think <off|minimal|low|medium|high|xhigh>` définit le niveau de réflexion. Alias : `/thinking`, `/t`.
- `/verbose on|off|full` active/désactive la sortie détaillée. Alias : `/v`.
- `/fast [status|on|off]` affiche ou définit le mode rapide.
- `/reasoning [on|off|stream]` active/désactive la visibilité du raisonnement. Alias : `/reason`.
- `/elevated [on|off|ask|full]` active/désactive le mode élevé. Alias : `/elev`.
- `/exec host=<auto|sandbox|gateway|node> security=<deny|allowlist|full> ask=<off|on-miss|always> node=<id>` affiche ou définit les valeurs par défaut d’exécution.
- `/model [name|#|status]` affiche ou définit le modèle.
- `/models [provider] [page] [limit=<n>|size=<n>|all]` liste les providers ou les modèles d’un provider.
- `/queue <mode>` gère le comportement de la file (`steer`, `interrupt`, `followup`, `collect`, `steer-backlog`) ainsi que des options comme `debounce:2s cap:25 drop:summarize`.
- `/help` affiche le résumé d’aide court.
- `/commands` affiche le catalogue de commandes généré.
- `/tools [compact|verbose]` affiche ce que l’agent actuel peut utiliser maintenant.
- `/status` affiche l’état du runtime, y compris l’usage/quota du provider lorsqu’il est disponible.
- `/tasks` liste les tâches d’arrière-plan actives/récentes pour la session actuelle.
- `/context [list|detail|json]` explique comment le contexte est assemblé.
- `/export-session [path]` exporte la session actuelle en HTML. Alias : `/export`.
- `/whoami` affiche votre ID d’expéditeur. Alias : `/id`.
- `/skill <name> [input]` exécute une skill par son nom.
- `/allowlist [list|add|remove] ...` gère les entrées d’allowlist. Texte uniquement.
- `/approve <id> <decision>` résout les invites d’approbation exec.
- `/btw <question>` pose une question secondaire sans modifier le contexte futur de la session. Voir [/tools/btw](/fr/tools/btw).
- `/subagents list|kill|log|info|send|steer|spawn` gère les exécutions de sous-agents pour la session actuelle.
- `/acp spawn|cancel|steer|close|sessions|status|set-mode|set|cwd|permissions|timeout|model|reset-options|doctor|install|help` gère les sessions ACP et les options du runtime.
- `/focus <target>` lie le thread Discord actuel ou le sujet/conversation Telegram à une cible de session.
- `/unfocus` supprime la liaison actuelle.
- `/agents` liste les agents liés au thread pour la session actuelle.
- `/kill <id|#|all>` interrompt un ou tous les sous-agents en cours d’exécution.
- `/steer <id|#> <message>` envoie un pilotage à un sous-agent en cours d’exécution. Alias : `/tell`.
- `/config show|get|set|unset` lit ou écrit `openclaw.json`. Réservé au propriétaire. Nécessite `commands.config: true`.
- `/mcp show|get|set|unset` lit ou écrit la configuration du serveur MCP gérée par OpenClaw sous `mcp.servers`. Réservé au propriétaire. Nécessite `commands.mcp: true`.
- `/plugins list|inspect|show|get|install|enable|disable` inspecte ou modifie l’état des plugins. `/plugin` est un alias. Réservé au propriétaire pour les écritures. Nécessite `commands.plugins: true`.
- `/debug show|set|unset|reset` gère les remplacements de configuration réservés au runtime. Réservé au propriétaire. Nécessite `commands.debug: true`.
- `/usage off|tokens|full|cost` contrôle le pied de page d’usage par réponse ou affiche un résumé local des coûts.
- `/tts on|off|status|provider|limit|summary|audio|help` contrôle le TTS. Voir [/tools/tts](/fr/tools/tts).
- `/restart` redémarre OpenClaw lorsqu’il est activé. Par défaut : activé ; définissez `commands.restart: false` pour le désactiver.
- `/activation mention|always` définit le mode d’activation de groupe.
- `/send on|off|inherit` définit la politique d’envoi. Réservé au propriétaire.
- `/bash <command>` exécute une commande shell sur l’hôte. Texte uniquement. Alias : `! <command>`. Nécessite `commands.bash: true` plus les allowlists `tools.elevated`.
- `!poll [sessionId]` vérifie une tâche bash d’arrière-plan.
- `!stop [sessionId]` arrête une tâche bash d’arrière-plan.

### Commandes dock générées

Les commandes dock sont générées à partir des plugins de canal avec prise en charge des commandes natives. Ensemble bundle actuel :

- `/dock-discord` (alias : `/dock_discord`)
- `/dock-mattermost` (alias : `/dock_mattermost`)
- `/dock-slack` (alias : `/dock_slack`)
- `/dock-telegram` (alias : `/dock_telegram`)

### Commandes de plugins bundles

Les plugins bundles peuvent ajouter d’autres commandes slash. Commandes bundles actuelles dans ce dépôt :

- `/dreaming [on|off|status|help]` active/désactive le dreaming de la mémoire. Voir [Dreaming](/fr/concepts/dreaming).
- `/pair [qr|status|pending|approve|cleanup|notify]` gère le flux d’appairage/configuration des appareils. Voir [Appairage](/fr/channels/pairing).
- `/phone status|arm <camera|screen|writes|all> [duration]|disarm` arme temporairement les commandes de nœud téléphone à haut risque.
- `/voice status|list [limit]|set <voiceId|name>` gère la configuration de la voix Talk. Sur Discord, le nom de la commande native est `/talkvoice`.
- `/card ...` envoie des préréglages de cartes enrichies LINE. Voir [LINE](/fr/channels/line).
- `/codex status|models|threads|resume|compact|review|account|mcp|skills` inspecte et contrôle le harnais app-server Codex bundle. Voir [Codex Harness](/fr/plugins/codex-harness).
- Commandes réservées à QQBot :
  - `/bot-ping`
  - `/bot-version`
  - `/bot-help`
  - `/bot-upgrade`
  - `/bot-logs`

### Commandes dynamiques de Skills

Les Skills invoquables par l’utilisateur sont également exposées comme commandes slash :

- `/skill <name> [input]` fonctionne toujours comme point d’entrée générique.
- les Skills peuvent aussi apparaître comme commandes directes telles que `/prose` lorsque la skill/le plugin les enregistre.
- l’enregistrement natif des commandes de Skills est contrôlé par `commands.nativeSkills` et `channels.<provider>.commands.nativeSkills`.

Remarques :

- Les commandes acceptent un `:` facultatif entre la commande et les arguments (par exemple `/think: high`, `/send: on`, `/help:`).
- `/new <model>` accepte un alias de modèle, `provider/model` ou un nom de provider (correspondance approximative) ; s’il n’y a pas de correspondance, le texte est traité comme corps du message.
- Pour le détail complet de l’usage par provider, utilisez `openclaw status --usage`.
- `/allowlist add|remove` nécessite `commands.config=true` et respecte `configWrites` du canal.
- Dans les canaux multi-comptes, `/allowlist --account <id>` ciblé configuration et `/config set channels.<provider>.accounts.<id>...` respectent aussi `configWrites` du compte cible.
- `/usage` contrôle le pied de page d’usage par réponse ; `/usage cost` affiche un résumé local des coûts à partir des logs de session OpenClaw.
- `/restart` est activé par défaut ; définissez `commands.restart: false` pour le désactiver.
- `/plugins install <spec>` accepte les mêmes spécifications de plugin que `openclaw plugins install` : chemin/archive local(e), package npm ou `clawhub:<pkg>`.
- `/plugins enable|disable` met à jour la configuration du plugin et peut demander un redémarrage.
- Commande native réservée à Discord : `/vc join|leave|status` contrôle les canaux vocaux (nécessite `channels.discord.voice` et les commandes natives ; non disponible en texte).
- Les commandes de liaison de thread Discord (`/focus`, `/unfocus`, `/agents`, `/session idle`, `/session max-age`) nécessitent que les liaisons de thread effectives soient activées (`session.threadBindings.enabled` et/ou `channels.discord.threadBindings.enabled`).
- Référence de commande ACP et comportement runtime : [Agents ACP](/fr/tools/acp-agents).
- `/verbose` est destiné au débogage et à une visibilité supplémentaire ; laissez-le **désactivé** en usage normal.
- `/fast on|off` conserve un remplacement de session. Utilisez l’option `inherit` de l’UI Sessions pour le supprimer et revenir aux valeurs par défaut de la configuration.
- `/fast` est spécifique au provider : OpenAI/OpenAI Codex le mappe à `service_tier=priority` sur les endpoints natifs Responses, tandis que les requêtes Anthropic publiques directes, y compris le trafic authentifié par OAuth envoyé vers `api.anthropic.com`, le mappent à `service_tier=auto` ou `standard_only`. Voir [OpenAI](/fr/providers/openai) et [Anthropic](/fr/providers/anthropic).
- Les résumés d’échec d’outil sont toujours affichés lorsqu’ils sont pertinents, mais le texte détaillé des échecs n’est inclus que lorsque `/verbose` est `on` ou `full`.
- `/reasoning` (et `/verbose`) sont risqués en contexte de groupe : ils peuvent révéler un raisonnement interne ou une sortie d’outil que vous n’aviez pas l’intention d’exposer. Préférez les laisser désactivés, surtout dans les discussions de groupe.
- `/model` conserve immédiatement le nouveau modèle de session.
- Si l’agent est inactif, l’exécution suivante l’utilise immédiatement.
- Si une exécution est déjà active, OpenClaw marque un changement en direct comme en attente et ne redémarre dans le nouveau modèle qu’à un point propre de nouvelle tentative.
- Si une activité d’outil ou une sortie de réponse a déjà commencé, le changement en attente peut rester en file jusqu’à une occasion ultérieure de nouvelle tentative ou au prochain tour utilisateur.
- **Chemin rapide :** les messages contenant uniquement une commande provenant d’expéditeurs en allowlist sont traités immédiatement (contournent la file + le modèle).
- **Filtrage par mention en groupe :** les messages contenant uniquement une commande provenant d’expéditeurs en allowlist contournent les exigences de mention.
- **Raccourcis inline (expéditeurs en allowlist uniquement) :** certaines commandes fonctionnent aussi lorsqu’elles sont intégrées dans un message normal et sont supprimées avant que le modèle ne voie le texte restant.
  - Exemple : `hey /status` déclenche une réponse d’état, et le texte restant continue dans le flux normal.
- Actuellement : `/help`, `/commands`, `/status`, `/whoami` (`/id`).
- Les messages contenant uniquement une commande non autorisés sont ignorés silencieusement, et les tokens inline `/...` sont traités comme du texte brut.
- **Commandes de Skills :** les Skills `user-invocable` sont exposées comme commandes slash. Les noms sont normalisés en `a-z0-9_` (32 caractères max) ; les collisions reçoivent des suffixes numériques (par exemple `_2`).
  - `/skill <name> [input]` exécute une skill par son nom (utile lorsque les limites de commandes natives empêchent des commandes par skill).
  - Par défaut, les commandes de skill sont transmises au modèle comme une requête normale.
  - Les Skills peuvent éventuellement déclarer `command-dispatch: tool` pour router directement la commande vers un outil (déterministe, sans modèle).
  - Exemple : `/prose` (plugin OpenProse) — voir [OpenProse](/fr/prose).
- **Arguments de commande native :** Discord utilise l’autocomplétion pour les options dynamiques (et des menus à boutons quand vous omettez des arguments requis). Telegram et Slack affichent un menu à boutons lorsqu’une commande prend en charge des choix et que vous omettez l’argument.

## `/tools`

`/tools` répond à une question de runtime, pas à une question de configuration : **ce que cet agent peut utiliser maintenant
dans cette conversation**.

- Par défaut, `/tools` est compact et optimisé pour une lecture rapide.
- `/tools verbose` ajoute de courtes descriptions.
- Les surfaces à commandes natives qui prennent en charge les arguments exposent le même changement de mode `compact|verbose`.
- Les résultats sont limités à la session, donc changer l’agent, le canal, le thread, l’autorisation de l’expéditeur ou le modèle peut modifier la sortie.
- `/tools` inclut les outils réellement accessibles au runtime, y compris les outils du cœur, les outils de plugin connectés et les outils possédés par le canal.

Pour modifier les profils et les remplacements, utilisez le panneau Tools de la Control UI ou les surfaces de configuration/catalogue au lieu de traiter `/tools` comme un catalogue statique.

## Surfaces d’usage (ce qui s’affiche où)

- **Usage/quota du provider** (exemple : « Claude 80% left ») s’affiche dans `/status` pour le provider du modèle actuel lorsque le suivi d’usage est activé. OpenClaw normalise les fenêtres des providers en `% left` ; pour MiniMax, les champs de pourcentage ne contenant que le restant sont inversés avant affichage, et les réponses `model_remains` privilégient l’entrée du modèle de chat plus un libellé de plan tagué par modèle.
- **Lignes tokens/cache** dans `/status` peuvent revenir à la dernière entrée d’usage du transcript lorsque l’instantané de session en direct est trop pauvre. Les valeurs en direct non nulles existantes restent prioritaires, et le repli sur le transcript peut aussi récupérer le libellé du modèle runtime actif ainsi qu’un total orienté prompt plus élevé lorsque les totaux stockés sont absents ou plus petits.
- **Tokens/coût par réponse** est contrôlé par `/usage off|tokens|full` (ajouté aux réponses normales).
- `/model status` concerne les **modèles/authentification/endpoints**, pas l’usage.

## Sélection de modèle (`/model`)

`/model` est implémenté comme une directive.

Exemples :

```
/model
/model list
/model 3
/model openai/gpt-5.4
/model opus@anthropic:default
/model status
```

Remarques :

- `/model` et `/model list` affichent un sélecteur compact numéroté (famille de modèle + providers disponibles).
- Sur Discord, `/model` et `/models` ouvrent un sélecteur interactif avec menus déroulants de provider et de modèle plus une étape Submit.
- `/model <#>` sélectionne depuis ce sélecteur (et privilégie le provider actuel lorsque possible).
- `/model status` affiche la vue détaillée, y compris l’endpoint provider configuré (`baseUrl`) et le mode API (`api`) lorsqu’ils sont disponibles.

## Remplacements de débogage

`/debug` permet de définir des remplacements de configuration **réservés au runtime** (en mémoire, pas sur disque). Réservé au propriétaire. Désactivé par défaut ; activez-le avec `commands.debug: true`.

Exemples :

```
/debug show
/debug set messages.responsePrefix="[openclaw]"
/debug set channels.whatsapp.allowFrom=["+1555","+4477"]
/debug unset messages.responsePrefix
/debug reset
```

Remarques :

- Les remplacements s’appliquent immédiatement aux nouvelles lectures de configuration, mais n’écrivent pas dans `openclaw.json`.
- Utilisez `/debug reset` pour effacer tous les remplacements et revenir à la configuration sur disque.

## Mises à jour de configuration

`/config` écrit dans votre configuration sur disque (`openclaw.json`). Réservé au propriétaire. Désactivé par défaut ; activez-le avec `commands.config: true`.

Exemples :

```
/config show
/config show messages.responsePrefix
/config get messages.responsePrefix
/config set messages.responsePrefix="[openclaw]"
/config unset messages.responsePrefix
```

Remarques :

- La configuration est validée avant écriture ; les modifications invalides sont rejetées.
- Les mises à jour `/config` persistent après redémarrage.

## Mises à jour MCP

`/mcp` écrit les définitions de serveur MCP gérées par OpenClaw sous `mcp.servers`. Réservé au propriétaire. Désactivé par défaut ; activez-le avec `commands.mcp: true`.

Exemples :

```text
/mcp show
/mcp show context7
/mcp set context7={"command":"uvx","args":["context7-mcp"]}
/mcp unset context7
```

Remarques :

- `/mcp` stocke la configuration dans la configuration OpenClaw, pas dans les paramètres de projet détenus par Pi.
- Les adaptateurs runtime décident quels transports sont réellement exécutables.

## Mises à jour des plugins

`/plugins` permet aux opérateurs d’inspecter les plugins découverts et de basculer leur activation dans la configuration. Les flux en lecture seule peuvent utiliser `/plugin` comme alias. Désactivé par défaut ; activez-le avec `commands.plugins: true`.

Exemples :

```text
/plugins
/plugins list
/plugin show context7
/plugins enable context7
/plugins disable context7
```

Remarques :

- `/plugins list` et `/plugins show` utilisent la découverte réelle des plugins dans l’espace de travail actuel plus la configuration sur disque.
- `/plugins enable|disable` met à jour uniquement la configuration du plugin ; cela n’installe ni ne désinstalle les plugins.
- Après des changements d’activation/désactivation, redémarrez le gateway pour les appliquer.

## Remarques sur les surfaces

- **Les commandes texte** s’exécutent dans la session de chat normale (les DM partagent `main`, les groupes ont leur propre session).
- **Les commandes natives** utilisent des sessions isolées :
  - Discord : `agent:<agentId>:discord:slash:<userId>`
  - Slack : `agent:<agentId>:slack:slash:<userId>` (préfixe configurable via `channels.slack.slashCommand.sessionPrefix`)
  - Telegram : `telegram:slash:<userId>` (cible la session de chat via `CommandTargetSessionKey`)
- **`/stop`** cible la session de chat active afin de pouvoir interrompre l’exécution en cours.
- **Slack :** `channels.slack.slashCommand` est encore pris en charge pour une seule commande de type `/openclaw`. Si vous activez `commands.native`, vous devez créer une commande slash Slack par commande intégrée (mêmes noms que `/help`). Les menus d’arguments de commandes pour Slack sont fournis sous forme de boutons Block Kit éphémères.
  - Exception native Slack : enregistrez `/agentstatus` (pas `/status`) car Slack réserve `/status`. Le texte `/status` fonctionne toujours dans les messages Slack.

## Questions secondaires BTW

`/btw` est une **question secondaire** rapide à propos de la session actuelle.

Contrairement à un chat normal :

- il utilise la session actuelle comme contexte d’arrière-plan,
- il s’exécute comme un appel ponctuel séparé **sans outils**,
- il ne modifie pas le contexte futur de la session,
- il n’est pas écrit dans l’historique du transcript,
- il est livré comme un résultat secondaire en direct au lieu d’un message assistant normal.

Cela rend `/btw` utile lorsque vous voulez une clarification temporaire pendant que la tâche principale continue.

Exemple :

```text
/btw qu’est-ce que nous faisons en ce moment ?
```

Voir [Questions secondaires BTW](/fr/tools/btw) pour le comportement complet et les
détails UX côté client.
