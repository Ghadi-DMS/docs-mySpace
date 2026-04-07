---
read_when:
    - Exécution ou débogage du processus gateway
summary: Runbook pour le service Gateway, son cycle de vie et ses opérations
title: Runbook Gateway
x-i18n:
    generated_at: "2026-04-07T06:50:17Z"
    model: gpt-5.4
    provider: openai
    source_hash: fd2c21036e88612861ef2195b8ff7205aca31386bb11558614ade8d1a54fdebd
    source_path: gateway/index.md
    workflow: 15
---

# Runbook Gateway

Utilisez cette page pour le démarrage au jour 1 et les opérations au jour 2 du service Gateway.

<CardGroup cols={2}>
  <Card title="Dépannage approfondi" icon="siren" href="/fr/gateway/troubleshooting">
    Diagnostics orientés symptômes avec des séquences de commandes exactes et des signatures de journaux.
  </Card>
  <Card title="Configuration" icon="sliders" href="/fr/gateway/configuration">
    Guide de configuration orienté tâches + référence de configuration complète.
  </Card>
  <Card title="Gestion des secrets" icon="key-round" href="/fr/gateway/secrets">
    Contrat SecretRef, comportement des instantanés d’exécution, et opérations de migration/rechargement.
  </Card>
  <Card title="Contrat du plan de secrets" icon="shield-check" href="/fr/gateway/secrets-plan-contract">
    Règles exactes de cible/chemin pour `secrets apply` et comportement des profils d’authentification en mode ref-only.
  </Card>
</CardGroup>

## Démarrage local en 5 minutes

<Steps>
  <Step title="Démarrer la Gateway">

```bash
openclaw gateway --port 18789
# debug/trace mirrored to stdio
openclaw gateway --port 18789 --verbose
# force-kill listener on selected port, then start
openclaw gateway --force
```

  </Step>

  <Step title="Vérifier l’état de santé du service">

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
```

Référence saine : `Runtime: running` et `RPC probe: ok`.

  </Step>

  <Step title="Valider l’état de préparation des canaux">

```bash
openclaw channels status --probe
```

Avec une Gateway joignable, cette commande exécute des sondes de canal en direct par compte ainsi que des audits facultatifs.
Si la Gateway est inaccessible, la CLI revient à des résumés de canaux basés uniquement sur la configuration
au lieu d’une sortie de sonde en direct.

  </Step>
</Steps>

<Note>
Le rechargement de configuration de la Gateway surveille le chemin du fichier de configuration actif (résolu à partir des valeurs par défaut du profil/de l’état, ou `OPENCLAW_CONFIG_PATH` s’il est défini).
Le mode par défaut est `gateway.reload.mode="hybrid"`.
Après le premier chargement réussi, le processus en cours d’exécution sert l’instantané de configuration actif en mémoire ; un rechargement réussi remplace cet instantané de manière atomique.
</Note>

## Modèle d’exécution

- Un processus toujours actif pour le routage, le plan de contrôle et les connexions aux canaux.
- Un port multiplexé unique pour :
  - le contrôle/RPC WebSocket
  - les API HTTP, compatibles OpenAI (`/v1/models`, `/v1/embeddings`, `/v1/chat/completions`, `/v1/responses`, `/tools/invoke`)
  - l’interface de contrôle et les hooks
- Mode de liaison par défaut : `loopback`.
- L’authentification est requise par défaut. Les configurations à secret partagé utilisent
  `gateway.auth.token` / `gateway.auth.password` (ou
  `OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD`), et les configurations
  avec proxy inverse hors loopback peuvent utiliser `gateway.auth.mode: "trusted-proxy"`.

## Endpoints compatibles OpenAI

La surface de compatibilité à plus fort impact d’OpenClaw est désormais :

- `GET /v1/models`
- `GET /v1/models/{id}`
- `POST /v1/embeddings`
- `POST /v1/chat/completions`
- `POST /v1/responses`

Pourquoi cet ensemble est important :

- La plupart des intégrations Open WebUI, LobeChat et LibreChat sondent d’abord `/v1/models`.
- De nombreux pipelines RAG et de mémoire attendent `/v1/embeddings`.
- Les clients natifs pour agents préfèrent de plus en plus `/v1/responses`.

Note de planification :

- `/v1/models` est orienté agent en priorité : il renvoie `openclaw`, `openclaw/default` et `openclaw/<agentId>`.
- `openclaw/default` est l’alias stable qui pointe toujours vers l’agent par défaut configuré.
- Utilisez `x-openclaw-model` lorsque vous souhaitez un remplacement du fournisseur/modèle backend ; sinon, le modèle normal et la configuration d’embeddings de l’agent sélectionné gardent le contrôle.

Tous ces endpoints s’exécutent sur le port principal de la Gateway et utilisent la même frontière d’authentification d’opérateur de confiance que le reste de l’API HTTP Gateway.

### Priorité du port et du mode de liaison

| Paramètre    | Ordre de résolution                                          |
| ------------ | ------------------------------------------------------------ |
| Port Gateway | `--port` → `OPENCLAW_GATEWAY_PORT` → `gateway.port` → `18789` |
| Mode de liaison | CLI/override → `gateway.bind` → `loopback`                |

### Modes de rechargement à chaud

| `gateway.reload.mode` | Comportement                                   |
| --------------------- | ---------------------------------------------- |
| `off`                 | Aucun rechargement de configuration            |
| `hot`                 | Applique uniquement les modifications sûres à chaud |
| `restart`             | Redémarre lors de modifications nécessitant un rechargement |
| `hybrid` (par défaut) | Applique à chaud quand c’est sûr, redémarre si nécessaire |

## Ensemble de commandes opérateur

```bash
openclaw gateway status
openclaw gateway status --deep   # adds a system-level service scan
openclaw gateway status --json
openclaw gateway install
openclaw gateway restart
openclaw gateway stop
openclaw secrets reload
openclaw logs --follow
openclaw doctor
```

`gateway status --deep` sert à la découverte supplémentaire de services (unités système
LaunchDaemons/systemd/schtasks), pas à une sonde de santé RPC plus approfondie.

## Gateways multiples (même hôte)

La plupart des installations doivent exécuter une gateway par machine. Une seule gateway peut héberger plusieurs
agents et canaux.

Vous n’avez besoin de plusieurs gateways que si vous souhaitez délibérément une isolation ou un bot de secours.

Vérifications utiles :

```bash
openclaw gateway status --deep
openclaw gateway probe
```

Ce à quoi s’attendre :

- `gateway status --deep` peut signaler `Other gateway-like services detected (best effort)`
  et afficher des conseils de nettoyage lorsque d’anciennes installations launchd/systemd/schtasks sont encore présentes.
- `gateway probe` peut avertir avec `multiple reachable gateways` lorsque plus d’une cible
  répond.
- Si c’est intentionnel, isolez les ports, la configuration/l’état et les racines d’espace de travail par gateway.

Configuration détaillée : [/gateway/multiple-gateways](/fr/gateway/multiple-gateways).

## Accès à distance

Méthode préférée : Tailscale/VPN.
Solution de repli : tunnel SSH.

```bash
ssh -N -L 18789:127.0.0.1:18789 user@host
```

Connectez ensuite les clients localement à `ws://127.0.0.1:18789`.

<Warning>
Les tunnels SSH ne contournent pas l’authentification de la gateway. Avec une authentification à secret partagé, les clients
doivent toujours envoyer `token`/`password`, même via le tunnel. Pour les modes avec identité,
la requête doit toujours satisfaire ce chemin d’authentification.
</Warning>

Voir : [Gateway distante](/fr/gateway/remote), [Authentification](/fr/gateway/authentication), [Tailscale](/fr/gateway/tailscale).

## Supervision et cycle de vie du service

Utilisez des exécutions supervisées pour une fiabilité proche de la production.

<Tabs>
  <Tab title="macOS (launchd)">

```bash
openclaw gateway install
openclaw gateway status
openclaw gateway restart
openclaw gateway stop
```

Les labels LaunchAgent sont `ai.openclaw.gateway` (par défaut) ou `ai.openclaw.<profile>` (profil nommé). `openclaw doctor` audite et répare la dérive de configuration du service.

  </Tab>

  <Tab title="Linux (systemd user)">

```bash
openclaw gateway install
systemctl --user enable --now openclaw-gateway[-<profile>].service
openclaw gateway status
```

Pour la persistance après déconnexion, activez le lingering :

```bash
sudo loginctl enable-linger <user>
```

Exemple manuel d’unité utilisateur lorsque vous avez besoin d’un chemin d’installation personnalisé :

```ini
[Unit]
Description=OpenClaw Gateway
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/local/bin/openclaw gateway --port 18789
Restart=always
RestartSec=5
TimeoutStopSec=30
TimeoutStartSec=30
SuccessExitStatus=0 143
KillMode=control-group

[Install]
WantedBy=default.target
```

  </Tab>

  <Tab title="Windows (natif)">

```powershell
openclaw gateway install
openclaw gateway status --json
openclaw gateway restart
openclaw gateway stop
```

Le démarrage géré natif sous Windows utilise une tâche planifiée nommée `OpenClaw Gateway`
(ou `OpenClaw Gateway (<profile>)` pour les profils nommés). Si la création de tâche planifiée
est refusée, OpenClaw revient à un lanceur par utilisateur dans le dossier de démarrage
qui pointe vers `gateway.cmd` dans le répertoire d’état.

  </Tab>

  <Tab title="Linux (service système)">

Utilisez une unité système pour les hôtes multi-utilisateurs/toujours actifs.

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now openclaw-gateway[-<profile>].service
```

Utilisez le même corps de service que pour l’unité utilisateur, mais installez-le sous
`/etc/systemd/system/openclaw-gateway[-<profile>].service` et ajustez
`ExecStart=` si votre binaire `openclaw` se trouve ailleurs.

  </Tab>
</Tabs>

## Gateways multiples sur un même hôte

La plupart des configurations doivent exécuter **une seule** Gateway.
N’en utilisez plusieurs que pour une isolation/redondance stricte (par exemple un profil de secours).

Liste de contrôle par instance :

- `gateway.port` unique
- `OPENCLAW_CONFIG_PATH` unique
- `OPENCLAW_STATE_DIR` unique
- `agents.defaults.workspace` unique

Exemple :

```bash
OPENCLAW_CONFIG_PATH=~/.openclaw/a.json OPENCLAW_STATE_DIR=~/.openclaw-a openclaw gateway --port 19001
OPENCLAW_CONFIG_PATH=~/.openclaw/b.json OPENCLAW_STATE_DIR=~/.openclaw-b openclaw gateway --port 19002
```

Voir : [Gateways multiples](/fr/gateway/multiple-gateways).

### Chemin rapide pour le profil de développement

```bash
openclaw --dev setup
openclaw --dev gateway --allow-unconfigured
openclaw --dev status
```

Les valeurs par défaut incluent un état/une configuration isolés et un port de base de gateway `19001`.

## Référence rapide du protocole (vue opérateur)

- La première trame client doit être `connect`.
- La Gateway renvoie un instantané `hello-ok` (`presence`, `health`, `stateVersion`, `uptimeMs`, limites/politique).
- `hello-ok.features.methods` / `events` est une liste de découverte prudente, et non
  un dump généré de chaque route d’assistance appelable.
- Requêtes : `req(method, params)` → `res(ok/payload|error)`.
- Les événements courants incluent `connect.challenge`, `agent`, `chat`,
  `session.message`, `session.tool`, `sessions.changed`, `presence`, `tick`,
  `health`, `heartbeat`, les événements du cycle de vie d’appairage/d’approbation, et `shutdown`.

Les exécutions d’agent comportent deux étapes :

1. Accusé de réception immédiat (`status:"accepted"`)
2. Réponse finale d’achèvement (`status:"ok"|"error"`), avec des événements `agent` diffusés entre les deux.

Voir la documentation complète du protocole : [Protocole Gateway](/fr/gateway/protocol).

## Vérifications opérationnelles

### Disponibilité

- Ouvrez une connexion WS et envoyez `connect`.
- Attendez une réponse `hello-ok` avec instantané.

### État de préparation

```bash
openclaw gateway status
openclaw channels status --probe
openclaw health
```

### Récupération après interruption

Les événements ne sont pas rejoués. En cas de trou dans la séquence, actualisez l’état (`health`, `system-presence`) avant de continuer.

## Signatures d’échec courantes

| Signature                                                      | Problème probable                                                                 |
| -------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| `refusing to bind gateway ... without auth`                    | Liaison hors loopback sans chemin d’authentification gateway valide               |
| `another gateway instance is already listening` / `EADDRINUSE` | Conflit de port                                                                   |
| `Gateway start blocked: set gateway.mode=local`                | Configuration définie en mode distant, ou estampille du mode local absente d’une configuration endommagée |
| `unauthorized` during connect                                  | Incompatibilité d’authentification entre le client et la gateway                  |

Pour les séquences complètes de diagnostic, utilisez [Dépannage Gateway](/fr/gateway/troubleshooting).

## Garanties de sécurité

- Les clients du protocole Gateway échouent rapidement lorsque la Gateway n’est pas disponible (pas de repli implicite vers un canal direct).
- Les premières trames invalides/non `connect` sont rejetées et la connexion est fermée.
- Un arrêt gracieux émet un événement `shutdown` avant la fermeture du socket.

---

Voir aussi :

- [Dépannage](/fr/gateway/troubleshooting)
- [Processus en arrière-plan](/fr/gateway/background-process)
- [Configuration](/fr/gateway/configuration)
- [Santé](/fr/gateway/health)
- [Doctor](/fr/gateway/doctor)
- [Authentification](/fr/gateway/authentication)
