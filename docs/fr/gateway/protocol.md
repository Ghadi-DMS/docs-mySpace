---
read_when:
    - Implémentation ou mise à jour de clients WS de passerelle
    - Débogage des incompatibilités de protocole ou des échecs de connexion
    - Régénération du schéma/des modèles du protocole
summary: 'Protocole WebSocket de la passerelle : handshake, trames, gestion des versions'
title: Protocole de la passerelle
x-i18n:
    generated_at: "2026-04-08T02:15:53Z"
    model: gpt-5.4
    provider: openai
    source_hash: 8635e3ac1dd311dbd3a770b088868aa1495a8d53b3ebc1eae0dfda3b2bf4694a
    source_path: gateway/protocol.md
    workflow: 15
---

# Protocole de la passerelle (WebSocket)

Le protocole WS de la passerelle est le **plan de contrôle unique + transport de nœud** pour
OpenClaw. Tous les clients (CLI, interface web, app macOS, nœuds iOS/Android,
nœuds sans interface) se connectent via WebSocket et déclarent leur **rôle** + **portée** au
moment du handshake.

## Transport

- WebSocket, trames texte avec charge utile JSON.
- La première trame **doit** être une requête `connect`.

## Handshake (`connect`)

Passerelle → Client (défi de pré-connexion) :

```json
{
  "type": "event",
  "event": "connect.challenge",
  "payload": { "nonce": "…", "ts": 1737264000000 }
}
```

Client → Passerelle :

```json
{
  "type": "req",
  "id": "…",
  "method": "connect",
  "params": {
    "minProtocol": 3,
    "maxProtocol": 3,
    "client": {
      "id": "cli",
      "version": "1.2.3",
      "platform": "macos",
      "mode": "operator"
    },
    "role": "operator",
    "scopes": ["operator.read", "operator.write"],
    "caps": [],
    "commands": [],
    "permissions": {},
    "auth": { "token": "…" },
    "locale": "en-US",
    "userAgent": "openclaw-cli/1.2.3",
    "device": {
      "id": "device_fingerprint",
      "publicKey": "…",
      "signature": "…",
      "signedAt": 1737264000000,
      "nonce": "…"
    }
  }
}
```

Passerelle → Client :

```json
{
  "type": "res",
  "id": "…",
  "ok": true,
  "payload": { "type": "hello-ok", "protocol": 3, "policy": { "tickIntervalMs": 15000 } }
}
```

Lorsqu’un jeton d’appareil est émis, `hello-ok` inclut également :

```json
{
  "auth": {
    "deviceToken": "…",
    "role": "operator",
    "scopes": ["operator.read", "operator.write"]
  }
}
```

Pendant un transfert d’amorçage approuvé, `hello-ok.auth` peut aussi inclure des
entrées de rôle supplémentaires bornées dans `deviceTokens` :

```json
{
  "auth": {
    "deviceToken": "…",
    "role": "node",
    "scopes": [],
    "deviceTokens": [
      {
        "deviceToken": "…",
        "role": "operator",
        "scopes": ["operator.approvals", "operator.read", "operator.talk.secrets", "operator.write"]
      }
    ]
  }
}
```

Pour le flux d’amorçage intégré nœud/opérateur, le jeton principal du nœud reste sur
`scopes: []` et tout jeton opérateur transmis reste borné à la liste d’autorisation
de l’opérateur d’amorçage (`operator.approvals`, `operator.read`,
`operator.talk.secrets`, `operator.write`). Les vérifications de portée d’amorçage restent
préfixées par rôle : les entrées opérateur satisfont uniquement les requêtes opérateur, et les rôles non opérateur
ont toujours besoin de portées sous leur propre préfixe de rôle.

### Exemple de nœud

```json
{
  "type": "req",
  "id": "…",
  "method": "connect",
  "params": {
    "minProtocol": 3,
    "maxProtocol": 3,
    "client": {
      "id": "ios-node",
      "version": "1.2.3",
      "platform": "ios",
      "mode": "node"
    },
    "role": "node",
    "scopes": [],
    "caps": ["camera", "canvas", "screen", "location", "voice"],
    "commands": ["camera.snap", "canvas.navigate", "screen.record", "location.get"],
    "permissions": { "camera.capture": true, "screen.record": false },
    "auth": { "token": "…" },
    "locale": "en-US",
    "userAgent": "openclaw-ios/1.2.3",
    "device": {
      "id": "device_fingerprint",
      "publicKey": "…",
      "signature": "…",
      "signedAt": 1737264000000,
      "nonce": "…"
    }
  }
}
```

## Format des trames

- **Requête** : `{type:"req", id, method, params}`
- **Réponse** : `{type:"res", id, ok, payload|error}`
- **Événement** : `{type:"event", event, payload, seq?, stateVersion?}`

Les méthodes avec effets de bord nécessitent des **clés d’idempotence** (voir le schéma).

## Rôles + portées

### Rôles

- `operator` = client du plan de contrôle (CLI/UI/automatisation).
- `node` = hôte de capacités (camera/screen/canvas/system.run).

### Portées (opérateur)

Portées courantes :

- `operator.read`
- `operator.write`
- `operator.admin`
- `operator.approvals`
- `operator.pairing`
- `operator.talk.secrets`

`talk.config` avec `includeSecrets: true` nécessite `operator.talk.secrets`
(ou `operator.admin`).

Les méthodes RPC de passerelle enregistrées par un plugin peuvent demander leur propre portée opérateur, mais
les préfixes d’administration cœur réservés (`config.*`, `exec.approvals.*`, `wizard.*`,
`update.*`) sont toujours résolus vers `operator.admin`.

La portée de méthode n’est que le premier filtre. Certaines commandes slash atteintes via
`chat.send` appliquent des vérifications plus strictes au niveau de la commande. Par exemple, les écritures persistantes
`/config set` et `/config unset` nécessitent `operator.admin`.

`node.pair.approve` a également une vérification de portée supplémentaire au moment de l’approbation en plus de la portée de méthode de base :

- requêtes sans commande : `operator.pairing`
- requêtes avec des commandes de nœud autres que exec : `operator.pairing` + `operator.write`
- requêtes qui incluent `system.run`, `system.run.prepare` ou `system.which` :
  `operator.pairing` + `operator.admin`

### `caps`/`commands`/`permissions` (nœud)

Les nœuds déclarent des revendications de capacité au moment de la connexion :

- `caps` : catégories de capacités de haut niveau.
- `commands` : liste d’autorisation de commandes pour l’invocation.
- `permissions` : bascules granulaires (par ex. `screen.record`, `camera.capture`).

La passerelle les traite comme des **revendications** et applique les listes d’autorisation côté serveur.

## Présence

- `system-presence` renvoie des entrées indexées par identité d’appareil.
- Les entrées de présence incluent `deviceId`, `roles` et `scopes` afin que les interfaces puissent afficher une seule ligne par appareil
  même lorsqu’il se connecte à la fois comme **operator** et **node**.

## Familles de méthodes RPC courantes

Cette page n’est pas un dump complet généré, mais la surface WS publique est plus large
que les exemples de handshake/auth ci-dessus. Voici les principales familles de méthodes que la
passerelle expose aujourd’hui.

`hello-ok.features.methods` est une liste de découverte conservatrice construite à partir de
`src/gateway/server-methods-list.ts` plus les exports de méthodes de plugin/canal chargés.
Traitez-la comme une découverte de fonctionnalités, et non comme un dump généré de tous les helpers appelables
implémentés dans `src/gateway/server-methods/*.ts`.

### Système et identité

- `health` renvoie l’instantané de santé de la passerelle mis en cache ou sondé fraîchement.
- `status` renvoie le résumé de la passerelle de style `/status` ; les champs sensibles sont
  inclus uniquement pour les clients opérateur avec portée admin.
- `gateway.identity.get` renvoie l’identité d’appareil de la passerelle utilisée par les flux de relais et
  d’association.
- `system-presence` renvoie l’instantané de présence actuel des appareils
  operator/node connectés.
- `system-event` ajoute un événement système et peut mettre à jour/diffuser le contexte
  de présence.
- `last-heartbeat` renvoie le dernier événement heartbeat persistant.
- `set-heartbeats` active ou désactive le traitement des heartbeats sur la passerelle.

### Modèles et utilisation

- `models.list` renvoie le catalogue de modèles autorisé à l’exécution.
- `usage.status` renvoie les fenêtres d’utilisation fournisseur / résumés du quota restant.
- `usage.cost` renvoie des résumés agrégés de coût d’utilisation pour une plage de dates.
- `doctor.memory.status` renvoie l’état de préparation de la mémoire vectorielle / des embeddings pour
  l’espace de travail de l’agent par défaut actif.
- `sessions.usage` renvoie des résumés d’utilisation par session.
- `sessions.usage.timeseries` renvoie des séries temporelles d’utilisation pour une session.
- `sessions.usage.logs` renvoie les entrées du journal d’utilisation pour une session.

### Canaux et helpers de connexion

- `channels.status` renvoie les résumés d’état des canaux/plugins intégrés + groupés.
- `channels.logout` déconnecte un canal/compte spécifique là où le canal
  prend en charge la déconnexion.
- `web.login.start` démarre un flux de connexion QR/web pour le fournisseur de canal web
  actuel compatible QR.
- `web.login.wait` attend que ce flux de connexion QR/web se termine et démarre le
  canal en cas de succès.
- `push.test` envoie une notification push APNs de test à un nœud iOS enregistré.
- `voicewake.get` renvoie les déclencheurs de mot d’activation stockés.
- `voicewake.set` met à jour les déclencheurs de mot d’activation et diffuse la modification.

### Messagerie et journaux

- `send` est la RPC de livraison sortante directe pour les envois ciblés par canal/compte/fil
  hors du moteur de chat.
- `logs.tail` renvoie la fin du journal de fichiers configuré de la passerelle avec contrôles de curseur/limite et
  de nombre maximal d’octets.

### Talk et TTS

- `talk.config` renvoie la charge utile de configuration Talk effective ; `includeSecrets`
  nécessite `operator.talk.secrets` (ou `operator.admin`).
- `talk.mode` définit/diffuse l’état actuel du mode Talk pour les clients WebChat/Control UI.
- `talk.speak` synthétise la parole via le fournisseur de parole Talk actif.
- `tts.status` renvoie l’état d’activation TTS, le fournisseur actif, les fournisseurs de secours,
  et l’état de configuration du fournisseur.
- `tts.providers` renvoie l’inventaire des fournisseurs TTS visibles.
- `tts.enable` et `tts.disable` activent/désactivent l’état des préférences TTS.
- `tts.setProvider` met à jour le fournisseur TTS préféré.
- `tts.convert` exécute une conversion ponctuelle de texte en parole.

### Secrets, configuration, mise à jour et assistant

- `secrets.reload` résout à nouveau les SecretRefs actifs et remplace l’état des secrets à l’exécution
  uniquement en cas de succès complet.
- `secrets.resolve` résout les attributions de secrets ciblées par commande pour un ensemble commande/cible spécifique.
- `config.get` renvoie l’instantané de configuration actuel et son hash.
- `config.set` écrit une charge utile de configuration validée.
- `config.patch` fusionne une mise à jour partielle de configuration.
- `config.apply` valide + remplace la charge utile complète de configuration.
- `config.schema` renvoie la charge utile du schéma de configuration en direct utilisée par Control UI et
  les outils CLI : schéma, `uiHints`, version et métadonnées de génération, y compris
  les métadonnées de schéma des plugins + canaux lorsque le runtime peut les charger. Le schéma
  inclut les métadonnées de champ `title` / `description` dérivées des mêmes libellés
  et textes d’aide utilisés par l’interface, y compris pour les branches imbriquées objet, joker, élément de tableau
  et composition `anyOf` / `oneOf` / `allOf` lorsqu’une documentation de champ correspondante
  existe.
- `config.schema.lookup` renvoie une charge utile de recherche limitée à un chemin pour un chemin de configuration :
  chemin normalisé, nœud de schéma superficiel, hint correspondant + `hintPath`, et
  résumés immédiats des enfants pour l’exploration UI/CLI.
  - Les nœuds de schéma de recherche conservent la documentation destinée à l’utilisateur et les champs de validation courants :
    `title`, `description`, `type`, `enum`, `const`, `format`, `pattern`,
    bornes numériques/chaînes/tableaux/objets, et indicateurs booléens comme
    `additionalProperties`, `deprecated`, `readOnly`, `writeOnly`.
  - Les résumés enfants exposent `key`, le `path` normalisé, `type`, `required`,
    `hasChildren`, plus le `hint` / `hintPath` correspondant.
- `update.run` exécute le flux de mise à jour de la passerelle et planifie un redémarrage uniquement lorsque
  la mise à jour elle-même a réussi.
- `wizard.start`, `wizard.next`, `wizard.status` et `wizard.cancel` exposent l’assistant
  d’onboarding via WS RPC.

### Grandes familles existantes

#### Helpers d’agent et d’espace de travail

- `agents.list` renvoie les entrées d’agent configurées.
- `agents.create`, `agents.update` et `agents.delete` gèrent les enregistrements d’agent et
  le câblage de l’espace de travail.
- `agents.files.list`, `agents.files.get` et `agents.files.set` gèrent les
  fichiers d’espace de travail d’amorçage exposés pour un agent.
- `agent.identity.get` renvoie l’identité effective de l’assistant pour un agent ou une
  session.
- `agent.wait` attend qu’une exécution se termine et renvoie l’instantané terminal lorsqu’il est
  disponible.

#### Contrôle de session

- `sessions.list` renvoie l’index actuel des sessions.
- `sessions.subscribe` et `sessions.unsubscribe` activent/désactivent les abonnements aux événements de changement de session
  pour le client WS actuel.
- `sessions.messages.subscribe` et `sessions.messages.unsubscribe` activent/désactivent les
  abonnements aux événements de transcription/message pour une session.
- `sessions.preview` renvoie des aperçus bornés de transcription pour des clés de session spécifiques.
- `sessions.resolve` résout ou canonicalise une cible de session.
- `sessions.create` crée une nouvelle entrée de session.
- `sessions.send` envoie un message dans une session existante.
- `sessions.steer` est la variante interrompre-et-réorienter pour une session active.
- `sessions.abort` interrompt le travail actif pour une session.
- `sessions.patch` met à jour les métadonnées/remplacements de session.
- `sessions.reset`, `sessions.delete` et `sessions.compact` effectuent la
  maintenance de session.
- `sessions.get` renvoie la ligne complète de session stockée.
- l’exécution de chat utilise toujours `chat.history`, `chat.send`, `chat.abort` et
  `chat.inject`.
- `chat.history` est normalisé pour l’affichage pour les clients UI : les balises de directive en ligne sont
  supprimées du texte visible, les charges utiles XML d’appel d’outil en texte brut (y compris
  `<tool_call>...</tool_call>`, `<function_call>...</function_call>`,
  `<tool_calls>...</tool_calls>`, `<function_calls>...</function_calls>`, et
  les blocs d’appel d’outil tronqués) ainsi que les jetons de contrôle du modèle ASCII/pleine largeur divulgués
  sont supprimés, les lignes assistant composées uniquement de jetons silencieux telles que `NO_REPLY` /
  `no_reply` exact sont omises, et les lignes surdimensionnées peuvent être remplacées par des espaces réservés.

#### Association d’appareil et jetons d’appareil

- `device.pair.list` renvoie les appareils associés en attente et approuvés.
- `device.pair.approve`, `device.pair.reject` et `device.pair.remove` gèrent
  les enregistrements d’association d’appareil.
- `device.token.rotate` fait pivoter un jeton d’appareil associé dans les limites de rôle
  et de portée approuvées.
- `device.token.revoke` révoque un jeton d’appareil associé.

#### Association de nœud, invocation et travail en attente

- `node.pair.request`, `node.pair.list`, `node.pair.approve`,
  `node.pair.reject` et `node.pair.verify` couvrent l’association de nœud et la
  vérification d’amorçage.
- `node.list` et `node.describe` renvoient l’état des nœuds connus/connectés.
- `node.rename` met à jour un libellé de nœud associé.
- `node.invoke` transmet une commande à un nœud connecté.
- `node.invoke.result` renvoie le résultat d’une requête d’invocation.
- `node.event` transporte des événements provenant d’un nœud vers la passerelle.
- `node.canvas.capability.refresh` rafraîchit les jetons de capacité canvas à portée limitée.
- `node.pending.pull` et `node.pending.ack` sont les API de file d’attente pour nœuds connectés.
- `node.pending.enqueue` et `node.pending.drain` gèrent le travail en attente durable
  pour les nœuds hors ligne/déconnectés.

#### Familles d’approbation

- `exec.approval.request`, `exec.approval.get`, `exec.approval.list` et
  `exec.approval.resolve` couvrent les demandes ponctuelles d’approbation exec ainsi que la
  recherche/relecture des approbations en attente.
- `exec.approval.waitDecision` attend une approbation exec en attente et renvoie
  la décision finale (ou `null` en cas de dépassement de délai).
- `exec.approvals.get` et `exec.approvals.set` gèrent les instantanés de politique
  d’approbation exec de la passerelle.
- `exec.approvals.node.get` et `exec.approvals.node.set` gèrent la politique locale exec
  du nœud via des commandes relais de nœud.
- `plugin.approval.request`, `plugin.approval.list`,
  `plugin.approval.waitDecision` et `plugin.approval.resolve` couvrent
  les flux d’approbation définis par plugin.

#### Autres grandes familles

- automatisation :
  - `wake` planifie une injection immédiate ou au prochain heartbeat d’un texte de réveil
  - `cron.list`, `cron.status`, `cron.add`, `cron.update`, `cron.remove`,
    `cron.run`, `cron.runs`
- skills/outils : `skills.*`, `tools.catalog`, `tools.effective`

### Familles d’événements courantes

- `chat` : mises à jour de chat UI telles que `chat.inject` et autres événements
  de chat uniquement liés à la transcription.
- `session.message` et `session.tool` : mises à jour de transcription/flux d’événements pour une
  session abonnée.
- `sessions.changed` : l’index de session ou les métadonnées ont changé.
- `presence` : mises à jour de l’instantané de présence système.
- `tick` : événement périodique de keepalive / vivacité.
- `health` : mise à jour de l’instantané de santé de la passerelle.
- `heartbeat` : mise à jour du flux d’événements heartbeat.
- `cron` : événement de changement d’exécution/tâche cron.
- `shutdown` : notification d’arrêt de la passerelle.
- `node.pair.requested` / `node.pair.resolved` : cycle de vie de l’association de nœud.
- `node.invoke.request` : diffusion de requête d’invocation de nœud.
- `device.pair.requested` / `device.pair.resolved` : cycle de vie des appareils associés.
- `voicewake.changed` : modification de la configuration des déclencheurs de mot d’activation.
- `exec.approval.requested` / `exec.approval.resolved` : cycle de vie
  d’approbation exec.
- `plugin.approval.requested` / `plugin.approval.resolved` : cycle de vie d’approbation
  de plugin.

### Méthodes helper pour les nœuds

- Les nœuds peuvent appeler `skills.bins` pour récupérer la liste actuelle des exécutables
  de skill pour les vérifications d’auto-autorisation.

### Méthodes helper pour les opérateurs

- Les opérateurs peuvent appeler `tools.catalog` (`operator.read`) pour récupérer le catalogue d’outils à l’exécution pour un
  agent. La réponse inclut des outils groupés et des métadonnées de provenance :
  - `source` : `core` ou `plugin`
  - `pluginId` : propriétaire du plugin lorsque `source="plugin"`
  - `optional` : indique si un outil de plugin est optionnel
- Les opérateurs peuvent appeler `tools.effective` (`operator.read`) pour récupérer l’inventaire effectif des outils à l’exécution
  pour une session.
  - `sessionKey` est requis.
  - La passerelle dérive le contexte d’exécution de confiance côté serveur à partir de la session au lieu d’accepter
    un contexte d’authentification ou de livraison fourni par l’appelant.
  - La réponse est limitée à la session et reflète ce que la conversation active peut utiliser immédiatement,
    y compris les outils cœur, plugin et canal.
- Les opérateurs peuvent appeler `skills.status` (`operator.read`) pour récupérer l’inventaire
  visible des skills pour un agent.
  - `agentId` est optionnel ; omettez-le pour lire l’espace de travail de l’agent par défaut.
  - La réponse inclut l’éligibilité, les exigences manquantes, les vérifications de configuration et
    les options d’installation nettoyées sans exposer les valeurs brutes des secrets.
- Les opérateurs peuvent appeler `skills.search` et `skills.detail` (`operator.read`) pour
  les métadonnées de découverte ClawHub.
- Les opérateurs peuvent appeler `skills.install` (`operator.admin`) dans deux modes :
  - mode ClawHub : `{ source: "clawhub", slug, version?, force? }` installe un
    dossier de skill dans le répertoire `skills/` de l’espace de travail de l’agent par défaut.
  - mode installateur de passerelle : `{ name, installId, dangerouslyForceUnsafeInstall?, timeoutMs? }`
    exécute une action `metadata.openclaw.install` déclarée sur l’hôte de la passerelle.
- Les opérateurs peuvent appeler `skills.update` (`operator.admin`) dans deux modes :
  - le mode ClawHub met à jour un slug suivi ou toutes les installations ClawHub suivies dans
    l’espace de travail de l’agent par défaut.
  - le mode config modifie les valeurs `skills.entries.<skillKey>` telles que `enabled`,
    `apiKey` et `env`.

## Approbations exec

- Lorsqu’une requête exec nécessite une approbation, la passerelle diffuse `exec.approval.requested`.
- Les clients opérateur résolvent cela en appelant `exec.approval.resolve` (nécessite la portée `operator.approvals`).
- Pour `host=node`, `exec.approval.request` doit inclure `systemRunPlan` (`argv`/`cwd`/`rawCommand`/métadonnées de session canoniques). Les requêtes sans `systemRunPlan` sont rejetées.
- Après approbation, les appels transmis `node.invoke system.run` réutilisent ce
  `systemRunPlan` canonique comme contexte faisant autorité pour la commande/le cwd/la session.
- Si un appelant modifie `command`, `rawCommand`, `cwd`, `agentId` ou
  `sessionKey` entre la préparation et le transfert final approuvé de `system.run`, la
  passerelle rejette l’exécution au lieu de faire confiance à la charge utile modifiée.

## Repli de livraison d’agent

- Les requêtes `agent` peuvent inclure `deliver=true` pour demander une livraison sortante.
- `bestEffortDeliver=false` conserve un comportement strict : les cibles de livraison non résolues ou internes uniquement renvoient `INVALID_REQUEST`.
- `bestEffortDeliver=true` autorise un repli vers une exécution limitée à la session lorsqu’aucune route livrable externe ne peut être résolue (par exemple sessions internes/webchat ou configurations multicanales ambiguës).

## Gestion des versions

- `PROTOCOL_VERSION` se trouve dans `src/gateway/protocol/schema.ts`.
- Les clients envoient `minProtocol` + `maxProtocol` ; le serveur rejette les incompatibilités.
- Les schémas + modèles sont générés à partir des définitions TypeBox :
  - `pnpm protocol:gen`
  - `pnpm protocol:gen:swift`
  - `pnpm protocol:check`

## Auth

- L’authentification de passerelle par secret partagé utilise `connect.params.auth.token` ou
  `connect.params.auth.password`, selon le mode d’authentification configuré.
- Les modes porteurs d’identité comme Tailscale Serve
  (`gateway.auth.allowTailscale: true`) ou le mode non-loopback
  `gateway.auth.mode: "trusted-proxy"` satisfont la vérification d’authentification de connexion à partir des
  en-têtes de requête au lieu de `connect.params.auth.*`.
- Le mode d’entrée privée `gateway.auth.mode: "none"` ignore entièrement l’authentification de connexion par secret partagé ; n’exposez pas ce mode sur une entrée publique/non fiable.
- Après l’association, la passerelle émet un **jeton d’appareil** limité au
  rôle + portées de la connexion. Il est renvoyé dans `hello-ok.auth.deviceToken` et doit être
  persisté par le client pour les connexions futures.
- Les clients doivent persister le `hello-ok.auth.deviceToken` principal après toute
  connexion réussie.
- Une reconnexion avec ce jeton d’appareil **stocké** doit également réutiliser l’ensemble de portées approuvées stocké
  pour ce jeton. Cela préserve l’accès déjà accordé en lecture/sondage/état
  et évite que les reconnexions ne se réduisent silencieusement à une
  portée implicite plus étroite limitée à l’administration.
- L’ordre de priorité normal de l’authentification de connexion est le suivant : jeton/mot de passe partagé explicite d’abord, puis
  `deviceToken` explicite, puis jeton stocké par appareil, puis jeton d’amorçage.
- Les entrées supplémentaires `hello-ok.auth.deviceTokens` sont des jetons de transfert d’amorçage.
  Ne les persistez que lorsque la connexion a utilisé une authentification d’amorçage sur un transport de confiance
  tel que `wss://` ou loopback/association locale.
- Si un client fournit un `deviceToken` **explicite** ou des `scopes` explicites, cet
  ensemble de portées demandé par l’appelant reste prioritaire ; les portées mises en cache ne sont réutilisées que
  lorsque le client réutilise le jeton stocké par appareil.
- Les jetons d’appareil peuvent être tournés/révoqués via `device.token.rotate` et
  `device.token.revoke` (nécessite la portée `operator.pairing`).
- L’émission/la rotation de jeton reste bornée à l’ensemble des rôles approuvés enregistré dans
  l’entrée d’association de cet appareil ; faire pivoter un jeton ne peut pas élargir l’appareil à un
  rôle que l’approbation d’association n’a jamais accordé.
- Pour les sessions de jeton d’appareil associé, la gestion des appareils est limitée à soi-même sauf si
  l’appelant possède également `operator.admin` : les appelants non admin peuvent supprimer/révoquer/faire pivoter uniquement
  leur **propre** entrée d’appareil.
- `device.token.rotate` vérifie également l’ensemble de portées opérateur demandé par rapport aux
  portées de la session actuelle de l’appelant. Les appelants non admin ne peuvent pas faire pivoter un jeton vers
  un ensemble de portées opérateur plus large que celui qu’ils possèdent déjà.
- Les échecs d’authentification incluent `error.details.code` ainsi que des indices de récupération :
  - `error.details.canRetryWithDeviceToken` (booléen)
  - `error.details.recommendedNextStep` (`retry_with_device_token`, `update_auth_configuration`, `update_auth_credentials`, `wait_then_retry`, `review_auth_configuration`)
- Comportement client pour `AUTH_TOKEN_MISMATCH` :
  - Les clients de confiance peuvent tenter une reprise bornée avec un jeton mis en cache par appareil.
  - Si cette reprise échoue, les clients doivent arrêter les boucles de reconnexion automatiques et afficher des indications d’action à l’opérateur.

## Identité d’appareil + association

- Les nœuds doivent inclure une identité d’appareil stable (`device.id`) dérivée de l’empreinte d’une
  paire de clés.
- Les passerelles émettent des jetons par appareil + rôle.
- Des approbations d’association sont requises pour les nouveaux IDs d’appareil sauf si
  l’approbation automatique locale est activée.
- L’approbation automatique d’association est centrée sur les connexions directes locales loopback.
- OpenClaw dispose également d’un chemin restreint d’auto-connexion backend/conteneur-local pour
  des flux helper de secret partagé de confiance.
- Les connexions tailnet ou LAN sur le même hôte sont toujours traitées comme distantes pour l’association et
  nécessitent une approbation.
- Tous les clients WS doivent inclure l’identité `device` pendant `connect` (operator + node).
  Control UI peut l’omettre uniquement dans les modes suivants :
  - `gateway.controlUi.allowInsecureAuth=true` pour la compatibilité HTTP non sécurisée localhost uniquement.
  - authentification opérateur Control UI réussie avec `gateway.auth.mode: "trusted-proxy"`.
  - `gateway.controlUi.dangerouslyDisableDeviceAuth=true` (mode de dernier recours, forte dégradation de sécurité).
- Toutes les connexions doivent signer le nonce fourni par le serveur dans `connect.challenge`.

### Diagnostics de migration de l’authentification d’appareil

Pour les clients hérités qui utilisent encore le comportement de signature antérieur au défi, `connect` renvoie désormais
des codes de détail `DEVICE_AUTH_*` sous `error.details.code` avec une valeur stable `error.details.reason`.

Échecs de migration courants :

| Message                     | details.code                     | details.reason           | Signification                                      |
| --------------------------- | -------------------------------- | ------------------------ | -------------------------------------------------- |
| `device nonce required`     | `DEVICE_AUTH_NONCE_REQUIRED`     | `device-nonce-missing`   | Le client a omis `device.nonce` (ou a envoyé une valeur vide). |
| `device nonce mismatch`     | `DEVICE_AUTH_NONCE_MISMATCH`     | `device-nonce-mismatch`  | Le client a signé avec un nonce obsolète/incorrect. |
| `device signature invalid`  | `DEVICE_AUTH_SIGNATURE_INVALID`  | `device-signature`       | La charge utile de signature ne correspond pas à la charge utile v2. |
| `device signature expired`  | `DEVICE_AUTH_SIGNATURE_EXPIRED`  | `device-signature-stale` | L’horodatage signé est en dehors du décalage autorisé. |
| `device identity mismatch`  | `DEVICE_AUTH_DEVICE_ID_MISMATCH` | `device-id-mismatch`     | `device.id` ne correspond pas à l’empreinte de la clé publique. |
| `device public key invalid` | `DEVICE_AUTH_PUBLIC_KEY_INVALID` | `device-public-key`      | Le format/canonicalisation de la clé publique a échoué. |

Cible de migration :

- Toujours attendre `connect.challenge`.
- Signer la charge utile v2 qui inclut le nonce du serveur.
- Envoyer le même nonce dans `connect.params.device.nonce`.
- La charge utile de signature préférée est `v3`, qui lie `platform` et `deviceFamily`
  en plus des champs device/client/role/scopes/token/nonce.
- Les signatures héritées `v2` restent acceptées pour compatibilité, mais l’épinglage des métadonnées
  des appareils associés continue de contrôler la politique de commande lors de la reconnexion.

## TLS + épinglage

- TLS est pris en charge pour les connexions WS.
- Les clients peuvent éventuellement épingler l’empreinte du certificat de la passerelle (voir la configuration `gateway.tls`
  ainsi que `gateway.remote.tlsFingerprint` ou le CLI `--tls-fingerprint`).

## Portée

Ce protocole expose l’**API complète de la passerelle** (état, canaux, modèles, chat,
agent, sessions, nœuds, approbations, etc.). La surface exacte est définie par les
schémas TypeBox dans `src/gateway/protocol/schema.ts`.
