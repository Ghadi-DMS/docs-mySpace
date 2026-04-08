---
read_when:
    - OpenClaw ne fonctionne pas et vous avez besoin du chemin le plus rapide vers une correction
    - Vous voulez un flux de triage avant de plonger dans des guides détaillés
summary: Hub de dépannage orienté symptômes pour OpenClaw
title: Dépannage général
x-i18n:
    generated_at: "2026-04-08T02:16:45Z"
    model: gpt-5.4
    provider: openai
    source_hash: 8abda90ef80234c2f91a51c5e1f2c004d4a4da12a5d5631b5927762550c6d5e3
    source_path: help/troubleshooting.md
    workflow: 15
---

# Dépannage

Si vous n'avez que 2 minutes, utilisez cette page comme point d'entrée de triage.

## Premières 60 secondes

Exécutez cette séquence exacte dans l'ordre :

```bash
openclaw status
openclaw status --all
openclaw gateway probe
openclaw gateway status
openclaw doctor
openclaw channels status --probe
openclaw logs --follow
```

Bon résultat en une ligne :

- `openclaw status` → affiche les canaux configurés et aucune erreur d'authentification évidente.
- `openclaw status --all` → le rapport complet est présent et partageable.
- `openclaw gateway probe` → la cible Gateway attendue est joignable (`Reachable: yes`). `RPC: limited - missing scope: operator.read` correspond à des diagnostics dégradés, pas à un échec de connexion.
- `openclaw gateway status` → `Runtime: running` et `RPC probe: ok`.
- `openclaw doctor` → aucune erreur bloquante de configuration/service.
- `openclaw channels status --probe` → une gateway joignable renvoie l'état de transport en direct par compte ainsi que les résultats de sonde/audit comme `works` ou `audit ok` ; si la gateway est inaccessible, la commande revient à des résumés basés uniquement sur la configuration.
- `openclaw logs --follow` → activité stable, aucune erreur fatale répétée.

## Anthropic long context 429

Si vous voyez :
`HTTP 429: rate_limit_error: Extra usage is required for long context requests`,
allez à [/gateway/troubleshooting#anthropic-429-extra-usage-required-for-long-context](/fr/gateway/troubleshooting#anthropic-429-extra-usage-required-for-long-context).

## Le backend local compatible OpenAI fonctionne directement mais échoue dans OpenClaw

Si votre backend local ou auto-hébergé `/v1` répond à de petites sondes directes
`/v1/chat/completions` mais échoue sur `openclaw infer model run` ou sur des tours
d'agent normaux :

1. Si l'erreur mentionne que `messages[].content` doit être une chaîne, définissez
   `models.providers.<provider>.models[].compat.requiresStringContent: true`.
2. Si le backend échoue toujours uniquement sur les tours d'agent OpenClaw, définissez
   `models.providers.<provider>.models[].compat.supportsTools: false` et réessayez.
3. Si de minuscules appels directs fonctionnent toujours mais que de plus gros prompts OpenClaw font planter le backend, traitez le problème restant comme une limitation du modèle/serveur en amont et poursuivez dans le guide détaillé :
   [/gateway/troubleshooting#local-openai-compatible-backend-passes-direct-probes-but-agent-runs-fail](/fr/gateway/troubleshooting#local-openai-compatible-backend-passes-direct-probes-but-agent-runs-fail)

## L'installation du plugin échoue avec des extensions openclaw manquantes

Si l'installation échoue avec `package.json missing openclaw.extensions`, le package du plugin
utilise une ancienne forme qu'OpenClaw n'accepte plus.

Correction dans le package du plugin :

1. Ajoutez `openclaw.extensions` à `package.json`.
2. Faites pointer les entrées vers des fichiers runtime compilés (généralement `./dist/index.js`).
3. Republiez le plugin et exécutez à nouveau `openclaw plugins install <package>`.

Exemple :

```json
{
  "name": "@openclaw/my-plugin",
  "version": "1.2.3",
  "openclaw": {
    "extensions": ["./dist/index.js"]
  }
}
```

Référence : [Architecture des plugins](/fr/plugins/architecture)

## Arbre de décision

```mermaid
flowchart TD
  A[OpenClaw ne fonctionne pas] --> B{Qu'est-ce qui casse en premier}
  B --> C[Pas de réponses]
  B --> D[Le tableau de bord ou le Control UI ne se connecte pas]
  B --> E[La Gateway ne démarre pas ou le service n'est pas en cours d'exécution]
  B --> F[Le canal se connecte mais les messages ne circulent pas]
  B --> G[Cron ou heartbeat ne s'est pas déclenché ou n'a pas été livré]
  B --> H[Le nœud est appairé mais l'outil camera canvas screen exec échoue]
  B --> I[L'outil navigateur échoue]

  C --> C1[/Section Pas de réponses/]
  D --> D1[/Section Control UI/]
  E --> E1[/Section Gateway/]
  F --> F1[/Section Flux de canal/]
  G --> G1[/Section Automatisation/]
  H --> H1[/Section Outils de nœud/]
  I --> I1[/Section Navigateur/]
```

<AccordionGroup>
  <Accordion title="Pas de réponses">
    ```bash
    openclaw status
    openclaw gateway status
    openclaw channels status --probe
    openclaw pairing list --channel <channel> [--account <id>]
    openclaw logs --follow
    ```

    Un bon résultat ressemble à :

    - `Runtime: running`
    - `RPC probe: ok`
    - Votre canal affiche le transport connecté et, lorsqu'il est pris en charge, `works` ou `audit ok` dans `channels status --probe`
    - L'expéditeur apparaît comme approuvé (ou la politique de messages privés est open/allowlist)

    Signatures de journal courantes :

    - `drop guild message (mention required` → le contrôle des mentions a bloqué le message dans Discord.
    - `pairing request` → l'expéditeur n'est pas approuvé et attend l'approbation d'appairage par message privé.
    - `blocked` / `allowlist` dans les journaux de canal → l'expéditeur, le salon ou le groupe est filtré.

    Pages détaillées :

    - [/gateway/troubleshooting#no-replies](/fr/gateway/troubleshooting#no-replies)
    - [/channels/troubleshooting](/fr/channels/troubleshooting)
    - [/channels/pairing](/fr/channels/pairing)

  </Accordion>

  <Accordion title="Le tableau de bord ou le Control UI ne se connecte pas">
    ```bash
    openclaw status
    openclaw gateway status
    openclaw logs --follow
    openclaw doctor
    openclaw channels status --probe
    ```

    Un bon résultat ressemble à :

    - `Dashboard: http://...` s'affiche dans `openclaw gateway status`
    - `RPC probe: ok`
    - Aucune boucle d'authentification dans les journaux

    Signatures de journal courantes :

    - `device identity required` → le contexte HTTP/non sécurisé ne peut pas terminer l'authentification de l'appareil.
    - `origin not allowed` → l'`Origin` du navigateur n'est pas autorisé pour la cible gateway du Control UI.
    - `AUTH_TOKEN_MISMATCH` avec des indications de nouvelle tentative (`canRetryWithDeviceToken=true`) → une nouvelle tentative avec jeton d'appareil de confiance peut se produire automatiquement une fois.
    - Cette nouvelle tentative avec jeton en cache réutilise l'ensemble de scopes en cache stocké avec le jeton d'appareil appairé. Les appelants `deviceToken` explicites / `scopes` explicites conservent à la place l'ensemble de scopes demandé.
    - Sur le chemin asynchrone Tailscale Serve Control UI, les tentatives échouées pour le même `{scope, ip}` sont sérialisées avant que le limiteur n'enregistre l'échec, de sorte qu'une seconde mauvaise nouvelle tentative concurrente peut déjà afficher `retry later`.
    - `too many failed authentication attempts (retry later)` depuis une origine de navigateur localhost → les échecs répétés depuis ce même `Origin` sont temporairement bloqués ; une autre origine localhost utilise un compartiment distinct.
    - `repeated unauthorized` après cette nouvelle tentative → mauvais jeton/mot de passe, incompatibilité de mode d'authentification ou jeton d'appareil appairé obsolète.
    - `gateway connect failed:` → l'UI cible la mauvaise URL/le mauvais port ou une gateway inaccessible.

    Pages détaillées :

    - [/gateway/troubleshooting#dashboard-control-ui-connectivity](/fr/gateway/troubleshooting#dashboard-control-ui-connectivity)
    - [/web/control-ui](/web/control-ui)
    - [/gateway/authentication](/fr/gateway/authentication)

  </Accordion>

  <Accordion title="La Gateway ne démarre pas ou le service installé n'est pas en cours d'exécution">
    ```bash
    openclaw status
    openclaw gateway status
    openclaw logs --follow
    openclaw doctor
    openclaw channels status --probe
    ```

    Un bon résultat ressemble à :

    - `Service: ... (loaded)`
    - `Runtime: running`
    - `RPC probe: ok`

    Signatures de journal courantes :

    - `Gateway start blocked: set gateway.mode=local` ou `existing config is missing gateway.mode` → le mode gateway est distant, ou le fichier de configuration n'a pas le marquage du mode local et doit être réparé.
    - `refusing to bind gateway ... without auth` → liaison hors loopback sans chemin d'authentification gateway valide (jeton/mot de passe, ou trusted-proxy lorsque configuré).
    - `another gateway instance is already listening` ou `EADDRINUSE` → le port est déjà pris.

    Pages détaillées :

    - [/gateway/troubleshooting#gateway-service-not-running](/fr/gateway/troubleshooting#gateway-service-not-running)
    - [/gateway/background-process](/fr/gateway/background-process)
    - [/gateway/configuration](/fr/gateway/configuration)

  </Accordion>

  <Accordion title="Le canal se connecte mais les messages ne circulent pas">
    ```bash
    openclaw status
    openclaw gateway status
    openclaw logs --follow
    openclaw doctor
    openclaw channels status --probe
    ```

    Un bon résultat ressemble à :

    - Le transport du canal est connecté.
    - Les vérifications pairing/allowlist réussissent.
    - Les mentions sont détectées lorsqu'elles sont requises.

    Signatures de journal courantes :

    - `mention required` → le contrôle des mentions dans les groupes a bloqué le traitement.
    - `pairing` / `pending` → l'expéditeur du message privé n'est pas encore approuvé.
    - `not_in_channel`, `missing_scope`, `Forbidden`, `401/403` → problème de jeton d'autorisation du canal.

    Pages détaillées :

    - [/gateway/troubleshooting#channel-connected-messages-not-flowing](/fr/gateway/troubleshooting#channel-connected-messages-not-flowing)
    - [/channels/troubleshooting](/fr/channels/troubleshooting)

  </Accordion>

  <Accordion title="Cron ou heartbeat ne s'est pas déclenché ou n'a pas été livré">
    ```bash
    openclaw status
    openclaw gateway status
    openclaw cron status
    openclaw cron list
    openclaw cron runs --id <jobId> --limit 20
    openclaw logs --follow
    ```

    Un bon résultat ressemble à :

    - `cron.status` indique qu'il est activé avec un prochain réveil.
    - `cron runs` affiche des entrées `ok` récentes.
    - Heartbeat est activé et ne se trouve pas hors des heures actives.

    Signatures de journal courantes :

- `cron: scheduler disabled; jobs will not run automatically` → cron est désactivé.
- `heartbeat skipped` avec `reason=quiet-hours` → en dehors des heures actives configurées.
- `heartbeat skipped` avec `reason=empty-heartbeat-file` → `HEARTBEAT.md` existe mais contient uniquement une structure vide ou des en-têtes.
- `heartbeat skipped` avec `reason=no-tasks-due` → le mode tâche de `HEARTBEAT.md` est actif mais aucun intervalle de tâche n'est encore arrivé à échéance.
- `heartbeat skipped` avec `reason=alerts-disabled` → toute la visibilité heartbeat est désactivée (`showOk`, `showAlerts` et `useIndicator` sont tous désactivés).
- `requests-in-flight` → la voie principale est occupée ; le réveil heartbeat a été différé. - `unknown accountId` → le compte cible de livraison heartbeat n'existe pas.

      Pages détaillées :

      - [/gateway/troubleshooting#cron-and-heartbeat-delivery](/fr/gateway/troubleshooting#cron-and-heartbeat-delivery)
      - [/automation/cron-jobs#troubleshooting](/fr/automation/cron-jobs#troubleshooting)
      - [/gateway/heartbeat](/fr/gateway/heartbeat)

    </Accordion>

    <Accordion title="Le nœud est appairé mais l'outil camera canvas screen exec échoue">
      ```bash
      openclaw status
      openclaw gateway status
      openclaw nodes status
      openclaw nodes describe --node <idOrNameOrIp>
      openclaw logs --follow
      ```

      Un bon résultat ressemble à :

      - Le nœud est listé comme connecté et appairé pour le rôle `node`.
      - La capacité existe pour la commande que vous invoquez.
      - L'état de permission est accordé pour l'outil.

      Signatures de journal courantes :

      - `NODE_BACKGROUND_UNAVAILABLE` → mettez l'application du nœud au premier plan.
      - `*_PERMISSION_REQUIRED` → la permission OS a été refusée ou est manquante.
      - `SYSTEM_RUN_DENIED: approval required` → l'approbation exec est en attente.
      - `SYSTEM_RUN_DENIED: allowlist miss` → la commande n'est pas dans la liste d'autorisation exec.

      Pages détaillées :

      - [/gateway/troubleshooting#node-paired-tool-fails](/fr/gateway/troubleshooting#node-paired-tool-fails)
      - [/nodes/troubleshooting](/fr/nodes/troubleshooting)
      - [/tools/exec-approvals](/fr/tools/exec-approvals)

    </Accordion>

    <Accordion title="Exec demande soudainement une approbation">
      ```bash
      openclaw config get tools.exec.host
      openclaw config get tools.exec.security
      openclaw config get tools.exec.ask
      openclaw gateway restart
      ```

      Ce qui a changé :

      - Si `tools.exec.host` n'est pas défini, la valeur par défaut est `auto`.
      - `host=auto` se résout en `sandbox` lorsqu'un runtime sandbox est actif, sinon en `gateway`.
      - `host=auto` ne concerne que le routage ; le comportement « YOLO » sans invite vient de `security=full` plus `ask=off` sur gateway/node.
      - Sur `gateway` et `node`, si `tools.exec.security` n'est pas défini, la valeur par défaut est `full`.
      - Si `tools.exec.ask` n'est pas défini, la valeur par défaut est `off`.
      - Résultat : si vous voyez des approbations, une politique locale à l'hôte ou par session a resserré exec par rapport aux valeurs par défaut actuelles.

      Rétablir le comportement actuel par défaut sans approbation :

      ```bash
      openclaw config set tools.exec.host gateway
      openclaw config set tools.exec.security full
      openclaw config set tools.exec.ask off
      openclaw gateway restart
      ```

      Alternatives plus sûres :

      - Définissez uniquement `tools.exec.host=gateway` si vous voulez simplement un routage hôte stable.
      - Utilisez `security=allowlist` avec `ask=on-miss` si vous voulez l'exec sur l'hôte tout en conservant une vérification lors des manques dans la liste d'autorisation.
      - Activez le mode sandbox si vous voulez que `host=auto` se résolve de nouveau vers `sandbox`.

      Signatures de journal courantes :

      - `Approval required.` → la commande attend `/approve ...`.
      - `SYSTEM_RUN_DENIED: approval required` → l'approbation exec sur l'hôte du nœud est en attente.
      - `exec host=sandbox requires a sandbox runtime for this session` → sélection sandbox implicite/explicite mais le mode sandbox est désactivé.

      Pages détaillées :

      - [/tools/exec](/fr/tools/exec)
      - [/tools/exec-approvals](/fr/tools/exec-approvals)
      - [/gateway/security#runtime-expectation-drift](/fr/gateway/security#runtime-expectation-drift)

    </Accordion>

    <Accordion title="L'outil navigateur échoue">
      ```bash
      openclaw status
      openclaw gateway status
      openclaw browser status
      openclaw logs --follow
      openclaw doctor
      ```

      Un bon résultat ressemble à :

      - L'état du navigateur indique `running: true` et un navigateur/profil choisi.
      - `openclaw` démarre, ou `user` peut voir les onglets Chrome locaux.

      Signatures de journal courantes :

      - `unknown command "browser"` ou `unknown command 'browser'` → `plugins.allow` est défini et n'inclut pas `browser`.
      - `Failed to start Chrome CDP on port` → le lancement du navigateur local a échoué.
      - `browser.executablePath not found` → le chemin du binaire configuré est incorrect.
      - `browser.cdpUrl must be http(s) or ws(s)` → l'URL CDP configurée utilise un schéma non pris en charge.
      - `browser.cdpUrl has invalid port` → l'URL CDP configurée a un port incorrect ou hors plage.
      - `No Chrome tabs found for profile="user"` → le profil d'attachement Chrome MCP n'a aucun onglet Chrome local ouvert.
      - `Remote CDP for profile "<name>" is not reachable` → le point de terminaison CDP distant configuré n'est pas joignable depuis cet hôte.
      - `Browser attachOnly is enabled ... not reachable` ou `Browser attachOnly is enabled and CDP websocket ... is not reachable` → le profil attach-only n'a pas de cible CDP active.
      - remplacements obsolètes de viewport / mode sombre / paramètres régionaux / hors ligne sur des profils attach-only ou CDP distants → exécutez `openclaw browser stop --browser-profile <name>` pour fermer la session de contrôle active et libérer l'état d'émulation sans redémarrer la gateway.

      Pages détaillées :

      - [/gateway/troubleshooting#browser-tool-fails](/fr/gateway/troubleshooting#browser-tool-fails)
      - [/tools/browser#missing-browser-command-or-tool](/fr/tools/browser#missing-browser-command-or-tool)
      - [/tools/browser-linux-troubleshooting](/fr/tools/browser-linux-troubleshooting)
      - [/tools/browser-wsl2-windows-remote-cdp-troubleshooting](/fr/tools/browser-wsl2-windows-remote-cdp-troubleshooting)

    </Accordion>
  </AccordionGroup>

## Lié

- [FAQ](/fr/help/faq) — questions fréquentes
- [Dépannage Gateway](/fr/gateway/troubleshooting) — problèmes spécifiques à la gateway
- [Doctor](/fr/gateway/doctor) — contrôles de santé et réparations automatisés
- [Dépannage des canaux](/fr/channels/troubleshooting) — problèmes de connectivité des canaux
- [Dépannage de l'automatisation](/fr/automation/cron-jobs#troubleshooting) — problèmes de cron et heartbeat
