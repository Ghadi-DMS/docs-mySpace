---
read_when:
    - Implémentation des fonctionnalités de l’app macOS
    - Modification du cycle de vie de Gateway ou du pontage des nœuds sur macOS
summary: Application compagnon macOS OpenClaw (barre des menus + broker Gateway)
title: App macOS
x-i18n:
    generated_at: "2026-04-18T06:43:38Z"
    model: gpt-5.4
    provider: openai
    source_hash: d637df2f73ced110223c48ea3c934045d782e150a46495f434cf924a6a00baf0
    source_path: platforms/macos.md
    workflow: 15
---

# Companion macOS OpenClaw (barre des menus + broker Gateway)

L’app macOS est le **compagnon de barre des menus** pour OpenClaw. Elle gère les autorisations,
gère/se connecte au Gateway localement (launchd ou manuel), et expose les capacités macOS
à l’agent en tant que nœud.

## Ce qu’elle fait

- Affiche des notifications natives et l’état dans la barre des menus.
- Gère les invites TCC (Notifications, Accessibilité, Enregistrement de l’écran, Microphone,
  Reconnaissance vocale, Automation/AppleScript).
- Exécute ou se connecte au Gateway (local ou distant).
- Expose des outils propres à macOS (Canvas, Caméra, Enregistrement de l’écran, `system.run`).
- Démarre le service hôte de nœud local en mode **distant** (launchd), et l’arrête en mode **local**.
- Peut héberger **PeekabooBridge** pour l’automatisation de l’interface utilisateur.
- Installe la CLI globale (`openclaw`) à la demande via npm, pnpm ou bun (l’app préfère npm, puis pnpm, puis bun ; Node reste l’environnement d’exécution recommandé pour Gateway).

## Mode local vs distant

- **Local** (par défaut) : l’app se connecte à un Gateway local en cours d’exécution s’il est présent ;
  sinon elle active le service launchd via `openclaw gateway install`.
- **Distant** : l’app se connecte à un Gateway via SSH/Tailscale et ne démarre jamais
  de processus local.
  L’app démarre le **service hôte de nœud** local afin que le Gateway distant puisse atteindre ce Mac.
  L’app ne lance pas le Gateway comme processus enfant.
  La découverte de Gateway privilégie désormais les noms Tailscale MagicDNS plutôt que les IP tailnet brutes,
  afin que l’app Mac se rétablisse plus fiablement lorsque les IP tailnet changent.

## Contrôle Launchd

L’app gère un LaunchAgent par utilisateur libellé `ai.openclaw.gateway`
(ou `ai.openclaw.<profile>` lors de l’utilisation de `--profile`/`OPENCLAW_PROFILE` ; l’ancien `com.openclaw.*` se décharge toujours).

```bash
launchctl kickstart -k gui/$UID/ai.openclaw.gateway
launchctl bootout gui/$UID/ai.openclaw.gateway
```

Remplacez le libellé par `ai.openclaw.<profile>` lors de l’exécution avec un profil nommé.

Si le LaunchAgent n’est pas installé, activez-le depuis l’app ou exécutez
`openclaw gateway install`.

## Capacités du nœud (mac)

L’app macOS se présente comme un nœud. Commandes courantes :

- Canvas : `canvas.present`, `canvas.navigate`, `canvas.eval`, `canvas.snapshot`, `canvas.a2ui.*`
- Caméra : `camera.snap`, `camera.clip`
- Écran : `screen.snapshot`, `screen.record`
- Système : `system.run`, `system.notify`

Le nœud rapporte une map `permissions` afin que les agents puissent déterminer ce qui est autorisé.

Service de nœud + IPC de l’app :

- Lorsque le service hôte de nœud sans interface est en cours d’exécution (mode distant), il se connecte au Gateway WS en tant que nœud.
- `system.run` s’exécute dans l’app macOS (contexte UI/TCC) via un socket Unix local ; les invites et la sortie restent dans l’app.

Schéma (SCI) :

```
Gateway -> Node Service (WS)
                 |  IPC (UDS + token + HMAC + TTL)
                 v
             Mac App (UI + TCC + system.run)
```

## Approbations d’exécution (system.run)

`system.run` est contrôlé par les **approbations d’exécution** dans l’app macOS (Réglages → Approbations d’exécution).
La sécurité + la demande + la liste d’autorisation sont stockées localement sur le Mac dans :

```
~/.openclaw/exec-approvals.json
```

Exemple :

```json
{
  "version": 1,
  "defaults": {
    "security": "deny",
    "ask": "on-miss"
  },
  "agents": {
    "main": {
      "security": "allowlist",
      "ask": "on-miss",
      "allowlist": [{ "pattern": "/opt/homebrew/bin/rg" }]
    }
  }
}
```

Remarques :

- Les entrées `allowlist` sont des motifs glob pour les chemins de binaires résolus.
- Le texte brut d’une commande shell qui contient une syntaxe de contrôle ou d’expansion shell (`&&`, `||`, `;`, `|`, `` ` ``, `$`, `<`, `>`, `(`, `)`) est traité comme une absence de correspondance de liste d’autorisation et nécessite une approbation explicite (ou l’ajout du binaire shell à la liste d’autorisation).
- Choisir « Toujours autoriser » dans l’invite ajoute cette commande à la liste d’autorisation.
- Les remplacements d’environnement de `system.run` sont filtrés (supprime `PATH`, `DYLD_*`, `LD_*`, `NODE_OPTIONS`, `PYTHON*`, `PERL*`, `RUBYOPT`, `SHELLOPTS`, `PS4`) puis fusionnés avec l’environnement de l’app.
- Pour les wrappers shell (`bash|sh|zsh ... -c/-lc`), les remplacements d’environnement propres à la requête sont réduits à une petite liste d’autorisation explicite (`TERM`, `LANG`, `LC_*`, `COLORTERM`, `NO_COLOR`, `FORCE_COLOR`).
- Pour les décisions toujours autoriser en mode liste d’autorisation, les wrappers de distribution connus (`env`, `nice`, `nohup`, `stdbuf`, `timeout`) conservent les chemins des exécutables internes au lieu des chemins des wrappers. Si le déballage n’est pas sûr, aucune entrée de liste d’autorisation n’est automatiquement conservée.

## Liens profonds

L’app enregistre le schéma d’URL `openclaw://` pour les actions locales.

### `openclaw://agent`

Déclenche une requête Gateway `agent`.
__OC_I18N_900004__
Paramètres de requête :

- `message` (obligatoire)
- `sessionKey` (facultatif)
- `thinking` (facultatif)
- `deliver` / `to` / `channel` (facultatif)
- `timeoutSeconds` (facultatif)
- `key` (facultatif, clé pour le mode sans surveillance)

Sécurité :

- Sans `key`, l’app demande une confirmation.
- Sans `key`, l’app applique une limite courte de message pour l’invite de confirmation et ignore `deliver` / `to` / `channel`.
- Avec une `key` valide, l’exécution se fait sans surveillance (prévu pour des automatisations personnelles).

## Flux d’intégration (typique)

1. Installez et lancez **OpenClaw.app**.
2. Complétez la liste de vérification des autorisations (invites TCC).
3. Assurez-vous que le mode **Local** est actif et que le Gateway est en cours d’exécution.
4. Installez la CLI si vous souhaitez un accès au terminal.

## Emplacement du répertoire d’état (macOS)

Évitez de placer votre répertoire d’état OpenClaw dans iCloud ou dans d’autres dossiers synchronisés dans le cloud.
Les chemins pris en charge par la synchronisation peuvent ajouter de la latence et provoquer occasionnellement des courses de verrouillage/synchronisation de fichiers pour
les sessions et les identifiants.

Préférez un chemin d’état local non synchronisé tel que :
__OC_I18N_900005__
Si `openclaw doctor` détecte un état sous :

- `~/Library/Mobile Documents/com~apple~CloudDocs/...`
- `~/Library/CloudStorage/...`

il affichera un avertissement et recommandera de revenir à un chemin local.

## Workflow de build et de développement (natif)

- `cd apps/macos && swift build`
- `swift run OpenClaw` (ou Xcode)
- Packager l’app : `scripts/package-mac-app.sh`

## Déboguer la connectivité Gateway (CLI macOS)

Utilisez la CLI de débogage pour exercer la même poignée de main WebSocket Gateway et la même logique de découverte
que l’app macOS, sans lancer l’app.
__OC_I18N_900006__
Options de connexion :

- `--url <ws://host:port>` : remplace la config
- `--mode <local|remote>` : résout à partir de la config (par défaut : config ou local)
- `--probe` : force une nouvelle sonde d’état
- `--timeout <ms>` : délai d’expiration de la requête (par défaut : `15000`)
- `--json` : sortie structurée pour les comparaisons

Options de découverte :

- `--include-local` : inclut les gateways qui seraient filtrés comme « locaux »
- `--timeout <ms>` : fenêtre globale de découverte (par défaut : `2000`)
- `--json` : sortie structurée pour les comparaisons

Astuce : comparez avec `openclaw gateway discover --json` pour voir si le pipeline de découverte
de l’app macOS (`local.` plus le domaine étendu configuré, avec
des solutions de repli wide-area et Tailscale Serve) diffère de
la découverte basée sur `dns-sd` de la CLI Node.

## Plomberie des connexions distantes (tunnels SSH)

Lorsque l’app macOS s’exécute en mode **distant**, elle ouvre un tunnel SSH afin que les composants de l’interface locale
puissent communiquer avec un Gateway distant comme s’il était sur localhost.

### Tunnel de contrôle (port WebSocket Gateway)

- **But :** contrôles d’état, statut, Chat Web, config et autres appels du plan de contrôle.
- **Port local :** le port Gateway (par défaut `18789`), toujours stable.
- **Port distant :** le même port Gateway sur l’hôte distant.
- **Comportement :** pas de port local aléatoire ; l’app réutilise un tunnel sain existant
  ou le redémarre si nécessaire.
- **Forme SSH :** `ssh -N -L <local>:127.0.0.1:<remote>` avec les options BatchMode +
  ExitOnForwardFailure + keepalive.
- **Rapport d’IP :** le tunnel SSH utilise le loopback, donc le gateway verra l’IP du nœud
  comme `127.0.0.1`. Utilisez le transport **Direct (ws/wss)** si vous souhaitez que la véritable IP
  cliente apparaisse (voir [accès distant macOS](/fr/platforms/mac/remote)).

Pour les étapes de configuration, voir [accès distant macOS](/fr/platforms/mac/remote). Pour les détails
du protocole, voir [protocole Gateway](/fr/gateway/protocol).

## Documentation associée

- [Runbook Gateway](/fr/gateway)
- [Gateway (macOS)](/fr/platforms/mac/bundled-gateway)
- [Autorisations macOS](/fr/platforms/mac/permissions)
- [Canvas](/fr/platforms/mac/canvas)
