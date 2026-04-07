---
read_when:
    - Le hub de dépannage vous a dirigé ici pour un diagnostic plus approfondi
    - Vous avez besoin de sections de runbook stables basées sur les symptômes avec des commandes exactes
summary: Runbook de dépannage approfondi pour la gateway, les canaux, l'automatisation, les nœuds et le navigateur
title: Dépannage
x-i18n:
    generated_at: "2026-04-07T06:50:28Z"
    model: gpt-5.4
    provider: openai
    source_hash: e0202e8858310a0bfc1c994cd37b01c3b2d6c73c8a74740094e92dc3c4c36729
    source_path: gateway/troubleshooting.md
    workflow: 15
---

# Dépannage de la gateway

Cette page est le runbook approfondi.
Commencez par [/help/troubleshooting](/fr/help/troubleshooting) si vous voulez d'abord le flux de triage rapide.

## Échelle de commandes

Exécutez-les d'abord, dans cet ordre :

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

Signaux attendus en état sain :

- `openclaw gateway status` affiche `Runtime: running` et `RPC probe: ok`.
- `openclaw doctor` ne signale aucun problème bloquant de configuration/service.
- `openclaw channels status --probe` affiche l'état du transport en direct par compte et,
  lorsque pris en charge, les résultats de sonde/d'audit tels que `works` ou `audit ok`.

## Anthropic 429 utilisation supplémentaire requise pour un contexte long

Utilisez ceci lorsque les journaux/erreurs incluent :
`HTTP 429: rate_limit_error: Extra usage is required for long context requests`.

```bash
openclaw logs --follow
openclaw models status
openclaw config get agents.defaults.models
```

Recherchez :

- Le modèle Anthropic Opus/Sonnet sélectionné a `params.context1m: true`.
- L'identifiant Anthropic actuel n'est pas éligible à l'usage en contexte long.
- Les requêtes échouent uniquement sur les longues sessions/exécutions de modèles qui nécessitent le chemin bêta 1M.

Options de correction :

1. Désactivez `context1m` pour ce modèle afin de revenir à la fenêtre de contexte normale.
2. Utilisez un identifiant Anthropic éligible aux requêtes à contexte long, ou passez à une clé API Anthropic.
3. Configurez des modèles de repli pour que les exécutions continuent lorsque les requêtes Anthropic à contexte long sont rejetées.

Documentation associée :

- [/providers/anthropic](/fr/providers/anthropic)
- [/reference/token-use](/fr/reference/token-use)
- [/help/faq#why-am-i-seeing-http-429-ratelimiterror-from-anthropic](/fr/help/faq#why-am-i-seeing-http-429-ratelimiterror-from-anthropic)

## Pas de réponses

Si les canaux sont actifs mais que rien ne répond, vérifiez le routage et la politique avant de reconnecter quoi que ce soit.

```bash
openclaw status
openclaw channels status --probe
openclaw pairing list --channel <channel> [--account <id>]
openclaw config get channels
openclaw logs --follow
```

Recherchez :

- Appairage en attente pour les expéditeurs DM.
- Blocage des mentions de groupe (`requireMention`, `mentionPatterns`).
- Incompatibilités entre allowlists de canal/groupe.

Signatures courantes :

- `drop guild message (mention required` → message de groupe ignoré jusqu'à mention.
- `pairing request` → l'expéditeur a besoin d'une approbation.
- `blocked` / `allowlist` → l'expéditeur/canal a été filtré par la politique.

Documentation associée :

- [/channels/troubleshooting](/fr/channels/troubleshooting)
- [/channels/pairing](/fr/channels/pairing)
- [/channels/groups](/fr/channels/groups)

## Connectivité du dashboard Control UI

Lorsque le dashboard/Control UI ne se connecte pas, validez l'URL, le mode d'authentification et les hypothèses de contexte sécurisé.

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
openclaw doctor
openclaw gateway status --json
```

Recherchez :

- L'URL de sonde correcte et l'URL du dashboard correcte.
- Une incompatibilité de mode d'authentification/de jeton entre le client et la gateway.
- Un usage HTTP là où l'identité de l'appareil est requise.

Signatures courantes :

- `device identity required` → contexte non sécurisé ou authentification d'appareil manquante.
- `origin not allowed` → l'`Origin` du navigateur n'est pas dans `gateway.controlUi.allowedOrigins`
  (ou vous vous connectez depuis une origine de navigateur non loopback sans
  allowlist explicite).
- `device nonce required` / `device nonce mismatch` → le client n'achève pas le
  flux d'authentification d'appareil basé sur challenge (`connect.challenge` + `device.nonce`).
- `device signature invalid` / `device signature expired` → le client a signé la mauvaise
  charge utile (ou un horodatage périmé) pour l'établissement de liaison en cours.
- `AUTH_TOKEN_MISMATCH` avec `canRetryWithDeviceToken=true` → le client peut faire une nouvelle tentative approuvée avec le jeton d'appareil en cache.
- Cette nouvelle tentative avec jeton en cache réutilise l'ensemble de scopes en cache stocké avec le
  jeton d'appareil appairé. Les appelants avec `deviceToken` explicite / `scopes` explicites conservent
  leur ensemble de scopes demandé à la place.
- En dehors de ce chemin de nouvelle tentative, la priorité d'authentification de connexion est :
  jeton partagé/mot de passe explicite d'abord, puis `deviceToken` explicite, puis jeton d'appareil stocké,
  puis jeton d'amorçage.
- Sur le chemin asynchrone Tailscale Serve Control UI, les tentatives échouées pour le même
  `{scope, ip}` sont sérialisées avant que le limiteur n'enregistre l'échec. Deux mauvaises
  nouvelles tentatives concurrentes du même client peuvent donc faire apparaître `retry later`
  lors de la seconde tentative au lieu de deux simples incompatibilités.
- `too many failed authentication attempts (retry later)` depuis un client loopback d'origine navigateur
  → les échecs répétés depuis cette même `Origin` normalisée sont temporairement verrouillés ; une autre origine localhost utilise un compartiment séparé.
- `unauthorized` répété après cette nouvelle tentative → dérive entre jeton partagé et jeton d'appareil ; actualisez la configuration du jeton et réapprouvez/faites tourner le jeton d'appareil si nécessaire.
- `gateway connect failed:` → cible d'hôte/port/url incorrecte.

### Correspondance rapide des codes de détail d'authentification

Utilisez `error.details.code` de la réponse `connect` échouée pour choisir l'action suivante :

| Code de détail               | Signification                                            | Action recommandée                                                                                                                                                                                                                                                                        |
| ---------------------------- | -------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `AUTH_TOKEN_MISSING`         | Le client n'a pas envoyé un jeton partagé requis.        | Collez/définissez le jeton dans le client et réessayez. Pour les chemins dashboard : `openclaw config get gateway.auth.token` puis collez-le dans les paramètres de Control UI.                                                                                                       |
| `AUTH_TOKEN_MISMATCH`        | Le jeton partagé ne correspond pas au jeton d'authentification de la gateway. | Si `canRetryWithDeviceToken=true`, autorisez une nouvelle tentative approuvée. Les nouvelles tentatives avec jeton en cache réutilisent les scopes approuvés stockés ; les appelants avec `deviceToken` / `scopes` explicites conservent les scopes demandés. En cas d'échec persistant, exécutez la [liste de contrôle de récupération de dérive de jeton](/cli/devices#token-drift-recovery-checklist). |
| `AUTH_DEVICE_TOKEN_MISMATCH` | Le jeton par appareil en cache est périmé ou révoqué.    | Faites tourner/réapprouvez le jeton d'appareil à l'aide de la [CLI devices](/cli/devices), puis reconnectez-vous.                                                                                                                                                                       |
| `PAIRING_REQUIRED`           | L'identité de l'appareil est connue mais pas approuvée pour ce rôle. | Approuvez la demande en attente : `openclaw devices list` puis `openclaw devices approve <requestId>`.                                                                                                                                                                                  |

Vérification de migration de l'authentification d'appareil v2 :

```bash
openclaw --version
openclaw doctor
openclaw gateway status
```

Si les journaux affichent des erreurs de nonce/signature, mettez à jour le client qui se connecte et vérifiez qu'il :

1. attend `connect.challenge`
2. signe la charge utile liée au challenge
3. envoie `connect.params.device.nonce` avec le même nonce de challenge

Si `openclaw devices rotate` / `revoke` / `remove` est refusé de manière inattendue :

- les sessions de jeton d'appareil appairé ne peuvent gérer que **leur propre** appareil, sauf si
  l'appelant dispose aussi de `operator.admin`
- `openclaw devices rotate --scope ...` ne peut demander que des scopes opérateur que
  la session appelante détient déjà

Documentation associée :

- [/web/control-ui](/web/control-ui)
- [/gateway/configuration](/fr/gateway/configuration) (modes d'authentification gateway)
- [/gateway/trusted-proxy-auth](/fr/gateway/trusted-proxy-auth)
- [/gateway/remote](/fr/gateway/remote)
- [/cli/devices](/cli/devices)

## Service gateway non exécuté

Utilisez ceci lorsque le service est installé mais que le processus ne reste pas actif.

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
openclaw doctor
openclaw gateway status --deep   # analyse aussi les services au niveau système
```

Recherchez :

- `Runtime: stopped` avec des indications de sortie.
- Incompatibilité de configuration du service (`Config (cli)` vs `Config (service)`).
- Conflits de port/d'écoute.
- Installations launchd/systemd/schtasks supplémentaires lorsque `--deep` est utilisé.
- Indications de nettoyage `Other gateway-like services detected (best effort)`.

Signatures courantes :

- `Gateway start blocked: set gateway.mode=local` ou `existing config is missing gateway.mode` → le mode gateway local n'est pas activé, ou le fichier de configuration a été écrasé et a perdu `gateway.mode`. Correction : définissez `gateway.mode="local"` dans votre configuration, ou réexécutez `openclaw onboard --mode local` / `openclaw setup` pour réappliquer la configuration locale attendue. Si vous exécutez OpenClaw via Podman, le chemin de configuration par défaut est `~/.openclaw/openclaw.json`.
- `refusing to bind gateway ... without auth` → liaison non loopback sans chemin d'authentification gateway valide (jeton/mot de passe, ou trusted-proxy si configuré).
- `another gateway instance is already listening` / `EADDRINUSE` → conflit de port.
- `Other gateway-like services detected (best effort)` → des unités launchd/systemd/schtasks obsolètes ou parallèles existent. La plupart des installations doivent garder une seule gateway par machine ; si vous en avez vraiment besoin de plusieurs, isolez ports + configuration/état/espace de travail. Voir [/gateway#multiple-gateways-same-host](/fr/gateway#multiple-gateways-same-host).

Documentation associée :

- [/gateway/background-process](/fr/gateway/background-process)
- [/gateway/configuration](/fr/gateway/configuration)
- [/gateway/doctor](/fr/gateway/doctor)

## Avertissements de sonde gateway

Utilisez ceci lorsque `openclaw gateway probe` atteint quelque chose, mais affiche quand même un bloc d'avertissement.

```bash
openclaw gateway probe
openclaw gateway probe --json
openclaw gateway probe --ssh user@gateway-host
```

Recherchez :

- `warnings[].code` et `primaryTargetId` dans la sortie JSON.
- Si l'avertissement concerne le repli SSH, plusieurs gateways, des scopes manquants ou des références d'authentification non résolues.

Signatures courantes :

- `SSH tunnel failed to start; falling back to direct probes.` → la configuration SSH a échoué, mais la commande a quand même essayé les cibles directes configurées/loopback.
- `multiple reachable gateways detected` → plus d'une cible a répondu. Cela signifie généralement une configuration multi-gateway intentionnelle ou des écouteurs obsolètes/en double.
- `Probe diagnostics are limited by gateway scopes (missing operator.read)` → la connexion a fonctionné, mais le RPC détaillé est limité par les scopes ; appairez l'identité de l'appareil ou utilisez des identifiants avec `operator.read`.
- texte d'avertissement SecretRef non résolu `gateway.auth.*` / `gateway.remote.*` → le matériel d'authentification n'était pas disponible dans ce chemin de commande pour la cible en échec.

Documentation associée :

- [/cli/gateway](/cli/gateway)
- [/gateway#multiple-gateways-same-host](/fr/gateway#multiple-gateways-same-host)
- [/gateway/remote](/fr/gateway/remote)

## Canal connecté mais messages sans circulation

Si l'état du canal est connecté mais que le flux de messages est interrompu, concentrez-vous sur la politique, les permissions et les règles de livraison propres au canal.

```bash
openclaw channels status --probe
openclaw pairing list --channel <channel> [--account <id>]
openclaw status --deep
openclaw logs --follow
openclaw config get channels
```

Recherchez :

- Politique DM (`pairing`, `allowlist`, `open`, `disabled`).
- Allowlist de groupe et exigences de mention.
- Permissions/scopes API de canal manquants.

Signatures courantes :

- `mention required` → message ignoré par la politique de mention de groupe.
- traces `pairing` / approbation en attente → l'expéditeur n'est pas approuvé.
- `missing_scope`, `not_in_channel`, `Forbidden`, `401/403` → problème d'authentification/de permissions du canal.

Documentation associée :

- [/channels/troubleshooting](/fr/channels/troubleshooting)
- [/channels/whatsapp](/fr/channels/whatsapp)
- [/channels/telegram](/fr/channels/telegram)
- [/channels/discord](/fr/channels/discord)

## Livraison cron et heartbeat

Si cron ou heartbeat ne s'est pas exécuté ou n'a pas livré, vérifiez d'abord l'état du planificateur, puis la cible de livraison.

```bash
openclaw cron status
openclaw cron list
openclaw cron runs --id <jobId> --limit 20
openclaw system heartbeat last
openclaw logs --follow
```

Recherchez :

- Cron activé et prochain réveil présent.
- État de l'historique d'exécution des tâches (`ok`, `skipped`, `error`).
- Raisons d'omission de heartbeat (`quiet-hours`, `requests-in-flight`, `alerts-disabled`, `empty-heartbeat-file`, `no-tasks-due`).

Signatures courantes :

- `cron: scheduler disabled; jobs will not run automatically` → cron désactivé.
- `cron: timer tick failed` → échec du tick du planificateur ; vérifiez les erreurs de fichiers/journaux/exécution.
- `heartbeat skipped` avec `reason=quiet-hours` → en dehors de la fenêtre d'heures actives.
- `heartbeat skipped` avec `reason=empty-heartbeat-file` → `HEARTBEAT.md` existe mais ne contient que des lignes vides / des en-têtes Markdown ; OpenClaw ignore donc l'appel au modèle.
- `heartbeat skipped` avec `reason=no-tasks-due` → `HEARTBEAT.md` contient un bloc `tasks:`, mais aucune tâche n'est due à ce tick.
- `heartbeat: unknown accountId` → identifiant de compte invalide pour la cible de livraison du heartbeat.
- `heartbeat skipped` avec `reason=dm-blocked` → la cible heartbeat a été résolue vers une destination de type DM alors que `agents.defaults.heartbeat.directPolicy` (ou le remplacement par agent) est défini sur `block`.

Documentation associée :

- [/automation/cron-jobs#troubleshooting](/fr/automation/cron-jobs#troubleshooting)
- [/automation/cron-jobs](/fr/automation/cron-jobs)
- [/gateway/heartbeat](/fr/gateway/heartbeat)

## Échec de l'outil de nœud appairé

Si un nœud est appairé mais que les outils échouent, isolez l'état d'avant-plan, les permissions et l'état d'approbation.

```bash
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
openclaw approvals get --node <idOrNameOrIp>
openclaw logs --follow
openclaw status
```

Recherchez :

- Nœud en ligne avec les capacités attendues.
- Permissions OS accordées pour caméra/micro/localisation/écran.
- Approbations d'exécution et état de l'allowlist.

Signatures courantes :

- `NODE_BACKGROUND_UNAVAILABLE` → l'application de nœud doit être au premier plan.
- `*_PERMISSION_REQUIRED` / `LOCATION_PERMISSION_REQUIRED` → permission OS manquante.
- `SYSTEM_RUN_DENIED: approval required` → approbation d'exécution en attente.
- `SYSTEM_RUN_DENIED: allowlist miss` → commande bloquée par l'allowlist.

Documentation associée :

- [/nodes/troubleshooting](/fr/nodes/troubleshooting)
- [/nodes/index](/fr/nodes/index)
- [/tools/exec-approvals](/fr/tools/exec-approvals)

## Échec de l'outil navigateur

Utilisez ceci lorsque les actions de l'outil navigateur échouent alors que la gateway elle-même est saine.

```bash
openclaw browser status
openclaw browser start --browser-profile openclaw
openclaw browser profiles
openclaw logs --follow
openclaw doctor
```

Recherchez :

- Si `plugins.allow` est défini et inclut `browser`.
- Un chemin d'exécutable navigateur valide.
- L'accessibilité du profil CDP.
- La disponibilité de Chrome local pour les profils `existing-session` / `user`.

Signatures courantes :

- `unknown command "browser"` ou `unknown command 'browser'` → le plugin navigateur intégré est exclu par `plugins.allow`.
- outil navigateur manquant / indisponible alors que `browser.enabled=true` → `plugins.allow` exclut `browser`, donc le plugin n'a jamais été chargé.
- `Failed to start Chrome CDP on port` → le processus du navigateur n'a pas pu être lancé.
- `browser.executablePath not found` → le chemin configuré est invalide.
- `browser.cdpUrl must be http(s) or ws(s)` → l'URL CDP configurée utilise un schéma non pris en charge comme `file:` ou `ftp:`.
- `browser.cdpUrl has invalid port` → l'URL CDP configurée a un port incorrect ou hors plage.
- `No Chrome tabs found for profile="user"` → le profil d'attachement Chrome MCP n'a aucun onglet Chrome local ouvert.
- `Remote CDP for profile "<name>" is not reachable` → le point de terminaison CDP distant configuré n'est pas accessible depuis l'hôte gateway.
- `Browser attachOnly is enabled ... not reachable` ou `Browser attachOnly is enabled and CDP websocket ... is not reachable` → le profil attach-only n'a pas de cible accessible, ou le point de terminaison HTTP a répondu mais le WebSocket CDP n'a toujours pas pu être ouvert.
- `Playwright is not available in this gateway build; '<feature>' is unsupported.` → l'installation actuelle de la gateway n'inclut pas le paquet Playwright complet ; les instantanés ARIA et les captures d'écran de page basiques peuvent toujours fonctionner, mais la navigation, les instantanés IA, les captures d'écran d'élément par sélecteur CSS et l'export PDF restent indisponibles.
- `fullPage is not supported for element screenshots` → la requête de capture d'écran a combiné `--full-page` avec `--ref` ou `--element`.
- `element screenshots are not supported for existing-session profiles; use ref from snapshot.` → les appels de capture d'écran Chrome MCP / `existing-session` doivent utiliser une capture de page ou un `--ref` d'instantané, pas un `--element` CSS.
- `existing-session file uploads do not support element selectors; use ref/inputRef.` → les hooks de téléversement Chrome MCP nécessitent des références d'instantané, pas des sélecteurs CSS.
- `existing-session file uploads currently support one file at a time.` → envoyez un seul téléversement par appel sur les profils Chrome MCP.
- `existing-session dialog handling does not support timeoutMs.` → les hooks de dialogue sur les profils Chrome MCP ne prennent pas en charge les remplacements de délai d'expiration.
- `response body is not supported for existing-session profiles yet.` → `responsebody` nécessite encore un navigateur géré ou un profil CDP brut.
- remplacements obsolètes de viewport / mode sombre / langue / hors ligne sur les profils attach-only ou CDP distants → exécutez `openclaw browser stop --browser-profile <name>` pour fermer la session de contrôle active et libérer l'état d'émulation Playwright/CDP sans redémarrer toute la gateway.

Documentation associée :

- [/tools/browser-linux-troubleshooting](/fr/tools/browser-linux-troubleshooting)
- [/tools/browser](/fr/tools/browser)

## Si vous avez effectué une mise à niveau et que quelque chose s'est soudainement cassé

La plupart des dysfonctionnements après mise à niveau sont dus à une dérive de configuration ou à des valeurs par défaut plus strictes désormais appliquées.

### 1) Le comportement d'authentification et de remplacement d'URL a changé

```bash
openclaw gateway status
openclaw config get gateway.mode
openclaw config get gateway.remote.url
openclaw config get gateway.auth.mode
```

À vérifier :

- Si `gateway.mode=remote`, les appels CLI peuvent cibler le distant alors que votre service local fonctionne correctement.
- Les appels explicites `--url` ne se replient pas sur les identifiants stockés.

Signatures courantes :

- `gateway connect failed:` → cible URL incorrecte.
- `unauthorized` → point de terminaison accessible mais authentification incorrecte.

### 2) Les garde-fous de liaison et d'authentification sont plus stricts

```bash
openclaw config get gateway.bind
openclaw config get gateway.auth.mode
openclaw config get gateway.auth.token
openclaw gateway status
openclaw logs --follow
```

À vérifier :

- Les liaisons non loopback (`lan`, `tailnet`, `custom`) nécessitent un chemin d'authentification gateway valide : authentification par jeton partagé/mot de passe, ou déploiement `trusted-proxy` non loopback correctement configuré.
- Les anciennes clés comme `gateway.token` ne remplacent pas `gateway.auth.token`.

Signatures courantes :

- `refusing to bind gateway ... without auth` → liaison non loopback sans chemin d'authentification gateway valide.
- `RPC probe: failed` alors que l'environnement d'exécution est actif → gateway active mais inaccessible avec l'authentification/l'URL actuelles.

### 3) L'état d'appairage et d'identité d'appareil a changé

```bash
openclaw devices list
openclaw pairing list --channel <channel> [--account <id>]
openclaw logs --follow
openclaw doctor
```

À vérifier :

- Approbations d'appareil en attente pour dashboard/nœuds.
- Approbations d'appairage DM en attente après des changements de politique ou d'identité.

Signatures courantes :

- `device identity required` → authentification d'appareil non satisfaite.
- `pairing required` → l'expéditeur/l'appareil doit être approuvé.

Si la configuration du service et l'environnement d'exécution ne concordent toujours pas après les vérifications, réinstallez les métadonnées du service à partir du même répertoire de profil/d'état :

```bash
openclaw gateway install --force
openclaw gateway restart
```

Documentation associée :

- [/gateway/pairing](/fr/gateway/pairing)
- [/gateway/authentication](/fr/gateway/authentication)
- [/gateway/background-process](/fr/gateway/background-process)
