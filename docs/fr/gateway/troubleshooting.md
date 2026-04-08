---
read_when:
    - Le hub de dépannage vous a dirigé ici pour un diagnostic plus approfondi
    - Vous avez besoin de sections de runbook stables basées sur les symptômes avec des commandes exactes
summary: Runbook de dépannage approfondi pour la gateway, les canaux, l’automatisation, les nœuds et le navigateur
title: Dépannage
x-i18n:
    generated_at: "2026-04-08T02:15:53Z"
    model: gpt-5.4
    provider: openai
    source_hash: 02c9537845248db0c9d315bf581338a93215fe6fe3688ed96c7105cbb19fe6ba
    source_path: gateway/troubleshooting.md
    workflow: 15
---

# Dépannage de la gateway

Cette page est le runbook approfondi.
Commencez par [/help/troubleshooting](/fr/help/troubleshooting) si vous voulez d’abord le flux de triage rapide.

## Échelle de commandes

Exécutez-les d’abord, dans cet ordre :

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

Signaux attendus quand tout est sain :

- `openclaw gateway status` affiche `Runtime: running` et `RPC probe: ok`.
- `openclaw doctor` ne signale aucun problème bloquant de configuration ou de service.
- `openclaw channels status --probe` affiche l’état de transport en direct par compte et,
  lorsque c’est pris en charge, des résultats de sonde/audit tels que `works` ou `audit ok`.

## Anthropic 429 extra usage required for long context

Utilisez cette section lorsque les journaux/erreurs incluent :
`HTTP 429: rate_limit_error: Extra usage is required for long context requests`.

```bash
openclaw logs --follow
openclaw models status
openclaw config get agents.defaults.models
```

À rechercher :

- Le modèle Anthropic Opus/Sonnet sélectionné a `params.context1m: true`.
- L’identifiant Anthropic actuel n’est pas admissible à l’utilisation de contexte long.
- Les requêtes échouent uniquement sur les longues sessions/exécutions de modèle qui nécessitent la voie bêta 1M.

Options de correction :

1. Désactivez `context1m` pour ce modèle afin de revenir à la fenêtre de contexte normale.
2. Utilisez un identifiant Anthropic admissible aux requêtes de contexte long, ou passez à une clé API Anthropic.
3. Configurez des modèles de repli afin que les exécutions se poursuivent lorsque les requêtes Anthropic de contexte long sont rejetées.

Associé :

- [/providers/anthropic](/fr/providers/anthropic)
- [/reference/token-use](/fr/reference/token-use)
- [/help/faq#why-am-i-seeing-http-429-ratelimiterror-from-anthropic](/fr/help/faq#why-am-i-seeing-http-429-ratelimiterror-from-anthropic)

## Le backend local compatible OpenAI réussit les sondes directes, mais les exécutions d’agent échouent

Utilisez cette section lorsque :

- `curl ... /v1/models` fonctionne
- les petits appels directs à `/v1/chat/completions` fonctionnent
- les exécutions de modèle OpenClaw échouent uniquement pendant les tours normaux d’agent

```bash
curl http://127.0.0.1:1234/v1/models
curl http://127.0.0.1:1234/v1/chat/completions \
  -H 'content-type: application/json' \
  -d '{"model":"<id>","messages":[{"role":"user","content":"hi"}],"stream":false}'
openclaw infer model run --model <provider/model> --prompt "hi" --json
openclaw logs --follow
```

À rechercher :

- les petits appels directs réussissent, mais les exécutions OpenClaw échouent uniquement avec des prompts plus volumineux
- des erreurs du backend indiquant que `messages[].content` attend une chaîne
- des plantages du backend qui apparaissent uniquement avec des comptes de jetons de prompt plus élevés ou les prompts complets du runtime d’agent

Signatures courantes :

- `messages[...].content: invalid type: sequence, expected a string` → le backend
  rejette les parties de contenu structurées de Chat Completions. Correctif : définissez
  `models.providers.<provider>.models[].compat.requiresStringContent: true`.
- les petites requêtes directes réussissent, mais les exécutions d’agent OpenClaw échouent avec des plantages du backend/modèle
  (par exemple Gemma sur certaines versions de `inferrs`) → le transport OpenClaw est
  probablement déjà correct ; le backend échoue sur la forme de prompt plus volumineuse
  du runtime d’agent.
- les échecs diminuent après la désactivation des outils mais ne disparaissent pas → les schémas d’outils faisaient
  partie de la pression, mais le problème restant relève toujours d’une limitation en amont du modèle/serveur
  ou d’un bogue du backend.

Options de correction :

1. Définissez `compat.requiresStringContent: true` pour les backends Chat Completions qui n’acceptent que des chaînes.
2. Définissez `compat.supportsTools: false` pour les modèles/backends qui ne peuvent pas gérer
   de manière fiable la surface de schéma d’outils d’OpenClaw.
3. Réduisez la pression du prompt lorsque c’est possible : bootstrap d’espace de travail plus petit, historique
   de session plus court, modèle local plus léger ou backend avec une meilleure prise en charge du contexte long.
4. Si les petites requêtes directes continuent de réussir tandis que les tours d’agent OpenClaw plantent encore
   dans le backend, traitez cela comme une limitation du serveur/modèle en amont et ouvrez
   un repro là-bas avec la forme de payload acceptée.

Associé :

- [/gateway/local-models](/fr/gateway/local-models)
- [/gateway/configuration#models](/fr/gateway/configuration#models)
- [/gateway/configuration-reference#openai-compatible-endpoints](/fr/gateway/configuration-reference#openai-compatible-endpoints)

## Aucune réponse

Si les canaux sont actifs mais que rien ne répond, vérifiez le routage et la stratégie avant de reconnecter quoi que ce soit.

```bash
openclaw status
openclaw channels status --probe
openclaw pairing list --channel <channel> [--account <id>]
openclaw config get channels
openclaw logs --follow
```

À rechercher :

- Appairage en attente pour les expéditeurs de MP.
- Blocage des mentions de groupe (`requireMention`, `mentionPatterns`).
- Incohérences de liste d’autorisation de canal/groupe.

Signatures courantes :

- `drop guild message (mention required` → message de groupe ignoré jusqu’à mention.
- `pairing request` → l’expéditeur doit être approuvé.
- `blocked` / `allowlist` → l’expéditeur/le canal a été filtré par la stratégie.

Associé :

- [/channels/troubleshooting](/fr/channels/troubleshooting)
- [/channels/pairing](/fr/channels/pairing)
- [/channels/groups](/fr/channels/groups)

## Connectivité du dashboard / Control UI

Lorsque le dashboard / Control UI ne se connecte pas, validez les hypothèses de l’URL, du mode d’authentification et du contexte sécurisé.

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
openclaw doctor
openclaw gateway status --json
```

À rechercher :

- URL de sonde et URL du dashboard correctes.
- Incohérence du mode d’authentification/jeton entre le client et la gateway.
- Utilisation de HTTP là où une identité d’appareil est requise.

Signatures courantes :

- `device identity required` → contexte non sécurisé ou authentification d’appareil manquante.
- `origin not allowed` → `Origin` du navigateur n’est pas dans `gateway.controlUi.allowedOrigins`
  (ou vous vous connectez depuis une origine de navigateur non loopback sans
  liste d’autorisation explicite).
- `device nonce required` / `device nonce mismatch` → le client n’effectue pas le
  flux d’authentification d’appareil basé sur challenge (`connect.challenge` + `device.nonce`).
- `device signature invalid` / `device signature expired` → le client a signé le mauvais
  payload (ou un horodatage périmé) pour la poignée de main actuelle.
- `AUTH_TOKEN_MISMATCH` avec `canRetryWithDeviceToken=true` → le client peut effectuer une nouvelle tentative fiable avec le jeton d’appareil mis en cache.
- Cette nouvelle tentative avec jeton mis en cache réutilise l’ensemble de portées mis en cache stocké avec le
  jeton d’appareil appairé. Les appelants avec `deviceToken` explicite / `scopes` explicites conservent à la place
  l’ensemble de portées demandé.
- En dehors de cette voie de nouvelle tentative, la priorité d’authentification de connexion est : jeton/mot de passe partagé explicite
  d’abord, puis `deviceToken` explicite, puis jeton d’appareil stocké,
  puis jeton bootstrap.
- Sur le chemin asynchrone Tailscale Serve Control UI, les tentatives échouées pour le même
  `{scope, ip}` sont sérialisées avant que le limiteur n’enregistre l’échec. Deux mauvaises nouvelles tentatives concurrentes du même client peuvent donc afficher `retry later`
  lors de la seconde tentative au lieu de deux simples incohérences.
- `too many failed authentication attempts (retry later)` depuis un client loopback d’origine navigateur
  → les échecs répétés depuis cette même `Origin` normalisée sont
  temporairement bloqués ; une autre origine localhost utilise un compartiment distinct.
- `repeated unauthorized` après cette nouvelle tentative → dérive du jeton partagé/jeton d’appareil ; actualisez la configuration du jeton et réapprouvez/faites tourner le jeton d’appareil si nécessaire.
- `gateway connect failed:` → mauvaise cible hôte/port/url.

### Carte rapide des codes de détail d’authentification

Utilisez `error.details.code` depuis la réponse `connect` en échec pour choisir l’action suivante :

| Code de détail               | Signification                                            | Action recommandée                                                                                                                                                                                                                                                                        |
| ---------------------------- | -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `AUTH_TOKEN_MISSING`         | Le client n’a pas envoyé de jeton partagé requis.        | Collez/définissez le jeton dans le client et réessayez. Pour les chemins dashboard : `openclaw config get gateway.auth.token` puis collez-le dans les paramètres de Control UI.                                                                                                        |
| `AUTH_TOKEN_MISMATCH`        | Le jeton partagé ne correspond pas au jeton d’authentification de la gateway. | Si `canRetryWithDeviceToken=true`, autorisez une nouvelle tentative fiable. Les nouvelles tentatives avec jeton mis en cache réutilisent les portées approuvées stockées ; les appelants avec `deviceToken` / `scopes` explicites conservent les portées demandées. Si cela échoue encore, exécutez la [liste de récupération de dérive de jeton](/cli/devices#token-drift-recovery-checklist). |
| `AUTH_DEVICE_TOKEN_MISMATCH` | Le jeton par appareil mis en cache est périmé ou révoqué. | Faites tourner/réapprouvez le jeton d’appareil à l’aide du [CLI devices](/cli/devices), puis reconnectez-vous.                                                                                                                                                                          |
| `PAIRING_REQUIRED`           | L’identité de l’appareil est connue mais non approuvée pour ce rôle. | Approuvez la demande en attente : `openclaw devices list` puis `openclaw devices approve <requestId>`.                                                                                                                                                                                   |

Vérification de migration auth d’appareil v2 :

```bash
openclaw --version
openclaw doctor
openclaw gateway status
```

Si les journaux affichent des erreurs de nonce/signature, mettez à jour le client qui se connecte et vérifiez qu’il :

1. attend `connect.challenge`
2. signe le payload lié au challenge
3. envoie `connect.params.device.nonce` avec le même nonce de challenge

Si `openclaw devices rotate` / `revoke` / `remove` est refusé de manière inattendue :

- les sessions de jeton d’appareil appairé ne peuvent gérer que **leur propre** appareil, sauf si
  l’appelant possède aussi `operator.admin`
- `openclaw devices rotate --scope ...` ne peut demander que des portées opérateur que
  la session appelante possède déjà

Associé :

- [/web/control-ui](/web/control-ui)
- [/gateway/configuration](/fr/gateway/configuration) (modes d’authentification de la gateway)
- [/gateway/trusted-proxy-auth](/fr/gateway/trusted-proxy-auth)
- [/gateway/remote](/fr/gateway/remote)
- [/cli/devices](/cli/devices)

## Le service gateway ne s’exécute pas

Utilisez cette section lorsque le service est installé mais que le processus ne reste pas actif.

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
openclaw doctor
openclaw gateway status --deep   # aussi analyser les services au niveau système
```

À rechercher :

- `Runtime: stopped` avec des indices de sortie.
- Incohérence de configuration du service (`Config (cli)` vs `Config (service)`).
- Conflits de port/écouteur.
- Installations launchd/systemd/schtasks supplémentaires lorsque `--deep` est utilisé.
- Indices de nettoyage `Other gateway-like services detected (best effort)`.

Signatures courantes :

- `Gateway start blocked: set gateway.mode=local` ou `existing config is missing gateway.mode` → le mode gateway local n’est pas activé, ou le fichier de configuration a été écrasé et a perdu `gateway.mode`. Correctif : définissez `gateway.mode="local"` dans votre configuration, ou relancez `openclaw onboard --mode local` / `openclaw setup` pour rétablir la configuration attendue en mode local. Si vous exécutez OpenClaw via Podman, le chemin de configuration par défaut est `~/.openclaw/openclaw.json`.
- `refusing to bind gateway ... without auth` → liaison non loopback sans chemin d’authentification gateway valide (jeton/mot de passe, ou trusted-proxy là où il est configuré).
- `another gateway instance is already listening` / `EADDRINUSE` → conflit de port.
- `Other gateway-like services detected (best effort)` → des unités launchd/systemd/schtasks périmées ou parallèles existent. La plupart des configurations devraient conserver une gateway par machine ; si vous en avez besoin de plus d’une, isolez ports + config/état/espace de travail. Voir [/gateway#multiple-gateways-same-host](/fr/gateway#multiple-gateways-same-host).

Associé :

- [/gateway/background-process](/fr/gateway/background-process)
- [/gateway/configuration](/fr/gateway/configuration)
- [/gateway/doctor](/fr/gateway/doctor)

## Avertissements de sonde gateway

Utilisez cette section lorsque `openclaw gateway probe` atteint quelque chose, mais affiche quand même un bloc d’avertissement.

```bash
openclaw gateway probe
openclaw gateway probe --json
openclaw gateway probe --ssh user@gateway-host
```

À rechercher :

- `warnings[].code` et `primaryTargetId` dans la sortie JSON.
- Si l’avertissement concerne le repli SSH, plusieurs gateways, des portées manquantes ou des références d’authentification non résolues.

Signatures courantes :

- `SSH tunnel failed to start; falling back to direct probes.` → la configuration SSH a échoué, mais la commande a quand même essayé les cibles configurées/directes loopback.
- `multiple reachable gateways detected` → plus d’une cible a répondu. En général, cela signifie une configuration multi-gateway intentionnelle ou des écouteurs périmés/en double.
- `Probe diagnostics are limited by gateway scopes (missing operator.read)` → la connexion a fonctionné, mais le RPC détaillé est limité par les portées ; appairez l’identité de l’appareil ou utilisez des identifiants avec `operator.read`.
- texte d’avertissement SecretRef `gateway.auth.*` / `gateway.remote.*` non résolu → le matériel d’authentification n’était pas disponible dans ce chemin de commande pour la cible en échec.

Associé :

- [/cli/gateway](/cli/gateway)
- [/gateway#multiple-gateways-same-host](/fr/gateway#multiple-gateways-same-host)
- [/gateway/remote](/fr/gateway/remote)

## Le canal est connecté mais les messages ne circulent pas

Si l’état du canal est connecté mais que le flux de messages est mort, concentrez-vous sur la stratégie, les autorisations et les règles de distribution spécifiques au canal.

```bash
openclaw channels status --probe
openclaw pairing list --channel <channel> [--account <id>]
openclaw status --deep
openclaw logs --follow
openclaw config get channels
```

À rechercher :

- Stratégie MP (`pairing`, `allowlist`, `open`, `disabled`).
- Liste d’autorisation de groupe et exigences de mention.
- Permissions/portées API de canal manquantes.

Signatures courantes :

- `mention required` → message ignoré par la stratégie de mention de groupe.
- traces `pairing` / approbation en attente → l’expéditeur n’est pas approuvé.
- `missing_scope`, `not_in_channel`, `Forbidden`, `401/403` → problème d’authentification/autorisations du canal.

Associé :

- [/channels/troubleshooting](/fr/channels/troubleshooting)
- [/channels/whatsapp](/fr/channels/whatsapp)
- [/channels/telegram](/fr/channels/telegram)
- [/channels/discord](/fr/channels/discord)

## Livraison cron et heartbeat

Si cron ou heartbeat ne s’est pas exécuté ou n’a pas livré, vérifiez d’abord l’état du planificateur, puis la cible de livraison.

```bash
openclaw cron status
openclaw cron list
openclaw cron runs --id <jobId> --limit 20
openclaw system heartbeat last
openclaw logs --follow
```

À rechercher :

- Cron activé et prochain réveil présent.
- État de l’historique d’exécution des tâches (`ok`, `skipped`, `error`).
- Raisons d’omission du heartbeat (`quiet-hours`, `requests-in-flight`, `alerts-disabled`, `empty-heartbeat-file`, `no-tasks-due`).

Signatures courantes :

- `cron: scheduler disabled; jobs will not run automatically` → cron désactivé.
- `cron: timer tick failed` → l’impulsion du planificateur a échoué ; vérifiez les erreurs de fichier/journal/runtime.
- `heartbeat skipped` avec `reason=quiet-hours` → en dehors de la fenêtre d’heures actives.
- `heartbeat skipped` avec `reason=empty-heartbeat-file` → `HEARTBEAT.md` existe mais ne contient que des lignes vides / en-têtes Markdown, donc OpenClaw saute l’appel au modèle.
- `heartbeat skipped` avec `reason=no-tasks-due` → `HEARTBEAT.md` contient un bloc `tasks:`, mais aucune tâche n’est due à cette impulsion.
- `heartbeat: unknown accountId` → id de compte invalide pour la cible de livraison heartbeat.
- `heartbeat skipped` avec `reason=dm-blocked` → la cible heartbeat a été résolue en destination de type MP alors que `agents.defaults.heartbeat.directPolicy` (ou une surcharge par agent) est définie sur `block`.

Associé :

- [/automation/cron-jobs#troubleshooting](/fr/automation/cron-jobs#troubleshooting)
- [/automation/cron-jobs](/fr/automation/cron-jobs)
- [/gateway/heartbeat](/fr/gateway/heartbeat)

## L’outil de nœud appairé échoue

Si un nœud est appairé mais que les outils échouent, isolez l’état premier plan, les autorisations et l’état d’approbation.

```bash
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
openclaw approvals get --node <idOrNameOrIp>
openclaw logs --follow
openclaw status
```

À rechercher :

- Nœud en ligne avec les capacités attendues.
- Autorisations du système d’exploitation pour caméra/micro/localisation/écran.
- État des approbations d’exécution et de la liste d’autorisation.

Signatures courantes :

- `NODE_BACKGROUND_UNAVAILABLE` → l’application du nœud doit être au premier plan.
- `*_PERMISSION_REQUIRED` / `LOCATION_PERMISSION_REQUIRED` → autorisation du système d’exploitation manquante.
- `SYSTEM_RUN_DENIED: approval required` → approbation d’exécution en attente.
- `SYSTEM_RUN_DENIED: allowlist miss` → commande bloquée par la liste d’autorisation.

Associé :

- [/nodes/troubleshooting](/fr/nodes/troubleshooting)
- [/nodes/index](/fr/nodes/index)
- [/tools/exec-approvals](/fr/tools/exec-approvals)

## L’outil de navigateur échoue

Utilisez cette section lorsque les actions de l’outil navigateur échouent même si la gateway elle-même est saine.

```bash
openclaw browser status
openclaw browser start --browser-profile openclaw
openclaw browser profiles
openclaw logs --follow
openclaw doctor
```

À rechercher :

- Si `plugins.allow` est défini et inclut `browser`.
- Chemin valide vers l’exécutable du navigateur.
- Accessibilité du profil CDP.
- Disponibilité de Chrome local pour les profils `existing-session` / `user`.

Signatures courantes :

- `unknown command "browser"` ou `unknown command 'browser'` → le plugin navigateur intégré est exclu par `plugins.allow`.
- outil navigateur manquant / indisponible alors que `browser.enabled=true` → `plugins.allow` exclut `browser`, donc le plugin ne s’est jamais chargé.
- `Failed to start Chrome CDP on port` → le processus du navigateur n’a pas pu se lancer.
- `browser.executablePath not found` → le chemin configuré est invalide.
- `browser.cdpUrl must be http(s) or ws(s)` → l’URL CDP configurée utilise un schéma non pris en charge tel que `file:` ou `ftp:`.
- `browser.cdpUrl has invalid port` → l’URL CDP configurée a un port incorrect ou hors plage.
- `No Chrome tabs found for profile="user"` → le profil d’attachement Chrome MCP n’a aucun onglet Chrome local ouvert.
- `Remote CDP for profile "<name>" is not reachable` → le point de terminaison CDP distant configuré n’est pas accessible depuis l’hôte gateway.
- `Browser attachOnly is enabled ... not reachable` ou `Browser attachOnly is enabled and CDP websocket ... is not reachable` → le profil attach-only n’a aucune cible accessible, ou le point de terminaison HTTP a répondu mais le WebSocket CDP n’a malgré tout pas pu être ouvert.
- `Playwright is not available in this gateway build; '<feature>' is unsupported.` → l’installation actuelle de la gateway ne dispose pas du package Playwright complet ; les instantanés ARIA et les captures d’écran de page de base peuvent toujours fonctionner, mais la navigation, les instantanés AI, les captures d’écran d’éléments par sélecteur CSS et l’export PDF restent indisponibles.
- `fullPage is not supported for element screenshots` → la demande de capture d’écran mélange `--full-page` avec `--ref` ou `--element`.
- `element screenshots are not supported for existing-session profiles; use ref from snapshot.` → les appels de capture d’écran Chrome MCP / `existing-session` doivent utiliser la capture de page ou un `--ref` d’instantané, et non un `--element` CSS.
- `existing-session file uploads do not support element selectors; use ref/inputRef.` → les hooks d’envoi de fichier Chrome MCP nécessitent des refs d’instantané, pas des sélecteurs CSS.
- `existing-session file uploads currently support one file at a time.` → envoyez un seul fichier par appel sur les profils Chrome MCP.
- `existing-session dialog handling does not support timeoutMs.` → les hooks de dialogue sur les profils Chrome MCP ne prennent pas en charge les surcharges de délai.
- `response body is not supported for existing-session profiles yet.` → `responsebody` nécessite encore un navigateur géré ou un profil CDP brut.
- anciennes surcharges de viewport / mode sombre / langue / hors ligne sur des profils attach-only ou CDP distants → exécutez `openclaw browser stop --browser-profile <name>` pour fermer la session de contrôle active et libérer l’état d’émulation Playwright/CDP sans redémarrer toute la gateway.

Associé :

- [/tools/browser-linux-troubleshooting](/fr/tools/browser-linux-troubleshooting)
- [/tools/browser](/fr/tools/browser)

## Si vous avez effectué une mise à niveau et que quelque chose s’est soudainement cassé

La plupart des problèmes après mise à niveau sont dus à une dérive de configuration ou à des valeurs par défaut plus strictes désormais appliquées.

### 1) Le comportement d’authentification et de surcharge d’URL a changé

```bash
openclaw gateway status
openclaw config get gateway.mode
openclaw config get gateway.remote.url
openclaw config get gateway.auth.mode
```

Ce qu’il faut vérifier :

- Si `gateway.mode=remote`, les appels CLI peuvent cibler le distant alors que votre service local fonctionne très bien.
- Les appels explicites `--url` ne reviennent pas aux identifiants stockés.

Signatures courantes :

- `gateway connect failed:` → mauvaise cible URL.
- `unauthorized` → point de terminaison accessible mais mauvaise authentification.

### 2) Les garde-fous de liaison et d’authentification sont plus stricts

```bash
openclaw config get gateway.bind
openclaw config get gateway.auth.mode
openclaw config get gateway.auth.token
openclaw gateway status
openclaw logs --follow
```

Ce qu’il faut vérifier :

- Les liaisons non loopback (`lan`, `tailnet`, `custom`) nécessitent un chemin d’authentification gateway valide : authentification par jeton partagé/mot de passe, ou déploiement `trusted-proxy` non loopback correctement configuré.
- Les anciennes clés comme `gateway.token` ne remplacent pas `gateway.auth.token`.

Signatures courantes :

- `refusing to bind gateway ... without auth` → liaison non loopback sans chemin d’authentification gateway valide.
- `RPC probe: failed` alors que le runtime est en cours d’exécution → gateway active mais inaccessible avec l’authentification/l’URL actuelles.

### 3) L’état d’appairage et d’identité de l’appareil a changé

```bash
openclaw devices list
openclaw pairing list --channel <channel> [--account <id>]
openclaw logs --follow
openclaw doctor
```

Ce qu’il faut vérifier :

- Approbations d’appareil en attente pour dashboard/nœuds.
- Approbations d’appairage MP en attente après des changements de stratégie ou d’identité.

Signatures courantes :

- `device identity required` → auth d’appareil non satisfaite.
- `pairing required` → expéditeur/appareil doit être approuvé.

Si la configuration du service et le runtime ne concordent toujours pas après vérification, réinstallez les métadonnées du service depuis le même répertoire profil/état :

```bash
openclaw gateway install --force
openclaw gateway restart
```

Associé :

- [/gateway/pairing](/fr/gateway/pairing)
- [/gateway/authentication](/fr/gateway/authentication)
- [/gateway/background-process](/fr/gateway/background-process)
