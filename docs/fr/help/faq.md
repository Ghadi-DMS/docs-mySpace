---
read_when:
    - Répondre aux questions courantes sur la configuration, l'installation, l'onboarding ou l'assistance à l'exécution
    - Trier les problèmes signalés par les utilisateurs avant un débogage plus approfondi
summary: Questions fréquentes sur l'installation, la configuration et l'utilisation d'OpenClaw
title: FAQ
x-i18n:
    generated_at: "2026-04-07T06:55:50Z"
    model: gpt-5.4
    provider: openai
    source_hash: bddcde55cf4bcec4913aadab4c665b235538104010e445e4c99915a1672b1148
    source_path: help/faq.md
    workflow: 15
---

# FAQ

Réponses rapides et dépannage plus approfondi pour des configurations réelles (développement local, VPS, multi-agent, clés OAuth/API, basculement de modèle). Pour les diagnostics d'exécution, consultez [Dépannage](/fr/gateway/troubleshooting). Pour la référence complète de configuration, consultez [Configuration](/fr/gateway/configuration).

## Les 60 premières secondes si quelque chose est cassé

1. **Statut rapide (première vérification)**

   ```bash
   openclaw status
   ```

   Résumé local rapide : OS + mise à jour, accessibilité de la passerelle/du service, agents/sessions, configuration du fournisseur + problèmes d'exécution (lorsque la passerelle est accessible).

2. **Rapport partageable (sans risque)**

   ```bash
   openclaw status --all
   ```

   Diagnostic en lecture seule avec fin de journal (jetons masqués).

3. **État du démon + du port**

   ```bash
   openclaw gateway status
   ```

   Affiche l'état d'exécution du superviseur par rapport à l'accessibilité RPC, l'URL cible de la sonde et la configuration probablement utilisée par le service.

4. **Sondes approfondies**

   ```bash
   openclaw status --deep
   ```

   Exécute une sonde d'état active de la passerelle, y compris des sondes de canaux lorsque cela est pris en charge
   (nécessite une passerelle accessible). Voir [Health](/fr/gateway/health).

5. **Suivre le dernier journal**

   ```bash
   openclaw logs --follow
   ```

   Si le RPC est indisponible, utilisez à la place :

   ```bash
   tail -f "$(ls -t /tmp/openclaw/openclaw-*.log | head -1)"
   ```

   Les journaux de fichiers sont distincts des journaux du service ; voir [Logging](/fr/logging) et [Dépannage](/fr/gateway/troubleshooting).

6. **Exécuter le doctor (réparations)**

   ```bash
   openclaw doctor
   ```

   Répare/migre la configuration et l'état + exécute des vérifications d'état. Voir [Doctor](/fr/gateway/doctor).

7. **Instantané de la passerelle**

   ```bash
   openclaw health --json
   openclaw health --verbose   # shows the target URL + config path on errors
   ```

   Demande un instantané complet à la passerelle en cours d'exécution (WS uniquement). Voir [Health](/fr/gateway/health).

## Démarrage rapide et configuration du premier lancement

<AccordionGroup>
  <Accordion title="Je suis bloqué, quel est le moyen le plus rapide de me débloquer ?">
    Utilisez un agent IA local capable de **voir votre machine**. C'est bien plus efficace que de demander
    sur Discord, car la plupart des cas de type « je suis bloqué » sont des **problèmes de configuration locale ou d'environnement**
    que les personnes à distance ne peuvent pas inspecter.

    - **Claude Code** : [https://www.anthropic.com/claude-code/](https://www.anthropic.com/claude-code/)
    - **OpenAI Codex** : [https://openai.com/codex/](https://openai.com/codex/)

    Ces outils peuvent lire le dépôt, exécuter des commandes, inspecter les journaux et aider à corriger
    la configuration de votre machine (PATH, services, permissions, fichiers d'authentification). Donnez-leur la **copie complète des sources**
    via l'installation modifiable (git) :

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Cela installe OpenClaw **à partir d'une copie git**, afin que l'agent puisse lire le code + la documentation et
    raisonner sur la version exacte que vous exécutez. Vous pouvez toujours revenir à la version stable plus tard
    en relançant l'installateur sans `--install-method git`.

    Conseil : demandez à l'agent de **planifier et superviser** la correction (étape par étape), puis d'exécuter uniquement les
    commandes nécessaires. Cela garde les changements limités et plus faciles à auditer.

    Si vous découvrez un vrai bug ou une correction, merci d'ouvrir une issue GitHub ou d'envoyer une PR :
    [https://github.com/openclaw/openclaw/issues](https://github.com/openclaw/openclaw/issues)
    [https://github.com/openclaw/openclaw/pulls](https://github.com/openclaw/openclaw/pulls)

    Commencez avec ces commandes (partagez les sorties lorsque vous demandez de l'aide) :

    ```bash
    openclaw status
    openclaw models status
    openclaw doctor
    ```

    Ce qu'elles font :

    - `openclaw status` : instantané rapide de l'état de la passerelle/de l'agent + configuration de base.
    - `openclaw models status` : vérifie l'authentification des fournisseurs + la disponibilité des modèles.
    - `openclaw doctor` : valide et répare les problèmes courants de configuration/d'état.

    Autres vérifications CLI utiles : `openclaw status --all`, `openclaw logs --follow`,
    `openclaw gateway status`, `openclaw health --verbose`.

    Boucle rapide de débogage : [Les 60 premières secondes si quelque chose est cassé](#first-60-seconds-if-something-is-broken).
    Documentation d'installation : [Install](/fr/install), [Options de l'installateur](/fr/install/installer), [Mise à jour](/fr/install/updating).

  </Accordion>

  <Accordion title="Heartbeat continue à être ignoré. Que signifient les raisons d'ignorance ?">
    Raisons courantes d'ignorance de heartbeat :

    - `quiet-hours` : en dehors de la fenêtre d'heures actives configurée
    - `empty-heartbeat-file` : `HEARTBEAT.md` existe mais contient uniquement une structure vide ou des en-têtes
    - `no-tasks-due` : le mode tâche de `HEARTBEAT.md` est actif mais aucun intervalle de tâche n'est encore arrivé à échéance
    - `alerts-disabled` : toute visibilité de heartbeat est désactivée (`showOk`, `showAlerts` et `useIndicator` sont tous désactivés)

    En mode tâche, les horodatages d'échéance ne sont avancés qu'après la fin
    d'une vraie exécution de heartbeat. Les exécutions ignorées ne marquent pas les tâches comme terminées.

    Documentation : [Heartbeat](/fr/gateway/heartbeat), [Automatisation et tâches](/fr/automation).

  </Accordion>

  <Accordion title="Méthode recommandée pour installer et configurer OpenClaw">
    Le dépôt recommande une exécution depuis les sources et l'utilisation de l'onboarding :

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash
    openclaw onboard --install-daemon
    ```

    L'assistant peut aussi construire automatiquement les ressources UI. Après l'onboarding, vous exécutez généralement la Gateway sur le port **18789**.

    Depuis les sources (contributeurs/dev) :

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    pnpm ui:build # auto-installs UI deps on first run
    openclaw onboard
    ```

    Si vous n'avez pas encore d'installation globale, exécutez-la via `pnpm openclaw onboard`.

  </Accordion>

  <Accordion title="Comment ouvrir le tableau de bord après l'onboarding ?">
    L'assistant ouvre votre navigateur avec une URL de tableau de bord propre (sans jeton) juste après l'onboarding et affiche également le lien dans le résumé. Gardez cet onglet ouvert ; s'il ne s'est pas lancé, copiez/collez l'URL affichée sur la même machine.
  </Accordion>

  <Accordion title="Comment authentifier le tableau de bord en localhost ou à distance ?">
    **Localhost (même machine) :**

    - Ouvrez `http://127.0.0.1:18789/`.
    - Si une authentification par secret partagé est demandée, collez le jeton ou le mot de passe configuré dans les paramètres de Control UI.
    - Source du jeton : `gateway.auth.token` (ou `OPENCLAW_GATEWAY_TOKEN`).
    - Source du mot de passe : `gateway.auth.password` (ou `OPENCLAW_GATEWAY_PASSWORD`).
    - Si aucun secret partagé n'est encore configuré, générez un jeton avec `openclaw doctor --generate-gateway-token`.

    **Pas sur localhost :**

    - **Tailscale Serve** (recommandé) : gardez la liaison sur loopback, exécutez `openclaw gateway --tailscale serve`, ouvrez `https://<magicdns>/`. Si `gateway.auth.allowTailscale` vaut `true`, les en-têtes d'identité satisfont l'authentification Control UI/WebSocket (pas besoin de coller un secret partagé, suppose un hôte de passerelle de confiance) ; les API HTTP exigent toujours une authentification par secret partagé sauf si vous utilisez délibérément `none` pour l'ingress privé ou l'authentification HTTP trusted-proxy.
      Les mauvaises tentatives concurrentes d'authentification Serve depuis le même client sont sérialisées avant que le limiteur d'échecs d'authentification n'enregistre l'échec, donc la deuxième mauvaise tentative peut déjà afficher `retry later`.
    - **Liaison tailnet** : exécutez `openclaw gateway --bind tailnet --token "<token>"` (ou configurez l'authentification par mot de passe), ouvrez `http://<tailscale-ip>:18789/`, puis collez le secret partagé correspondant dans les paramètres du tableau de bord.
    - **Proxy inverse avec gestion d'identité** : gardez la Gateway derrière un trusted proxy non loopback, configurez `gateway.auth.mode: "trusted-proxy"`, puis ouvrez l'URL du proxy.
    - **Tunnel SSH** : `ssh -N -L 18789:127.0.0.1:18789 user@host` puis ouvrez `http://127.0.0.1:18789/`. L'authentification par secret partagé s'applique toujours via le tunnel ; collez le jeton ou mot de passe configuré si demandé.

    Voir [Tableau de bord](/web/dashboard) et [Surfaces Web](/web) pour les modes de liaison et les détails d'authentification.

  </Accordion>

  <Accordion title="Pourquoi y a-t-il deux configurations d'approbation exec pour les approbations par chat ?">
    Elles contrôlent des couches différentes :

    - `approvals.exec` : transfère les invites d'approbation vers des destinations de chat
    - `channels.<channel>.execApprovals` : fait agir ce canal comme client d'approbation natif pour les approbations exec

    La politique exec de l'hôte reste la vraie barrière d'approbation. La configuration du chat contrôle seulement où les invites d'approbation
    apparaissent et comment les personnes peuvent y répondre.

    Dans la plupart des configurations, vous n'avez **pas** besoin des deux :

    - Si le chat prend déjà en charge les commandes et réponses, le `/approve` dans le même chat fonctionne via le chemin partagé.
    - Si un canal natif pris en charge peut déduire les approbateurs de manière sûre, OpenClaw active désormais automatiquement les approbations natives en DM d'abord lorsque `channels.<channel>.execApprovals.enabled` n'est pas défini ou vaut `"auto"`.
    - Lorsque des cartes/boutons d'approbation natifs sont disponibles, cette UI native est le chemin principal ; l'agent ne doit inclure une commande `/approve` manuelle que si le résultat de l'outil indique que les approbations par chat ne sont pas disponibles ou que l'approbation manuelle est le seul chemin.
    - Utilisez `approvals.exec` uniquement lorsque les invites doivent aussi être transférées vers d'autres chats ou des salons ops explicites.
    - Utilisez `channels.<channel>.execApprovals.target: "channel"` ou `"both"` uniquement lorsque vous voulez explicitement que les invites d'approbation soient republiées dans la salle/le sujet d'origine.
    - Les approbations de plugin sont encore distinctes : elles utilisent par défaut le `/approve` dans le même chat, le transfert facultatif `approvals.plugin`, et seuls certains canaux natifs conservent en plus une gestion native des approbations de plugin.

    En version courte : la redirection sert au routage, la configuration du client natif sert à une UX plus riche propre au canal.
    Voir [Approbations Exec](/fr/tools/exec-approvals).

  </Accordion>

  <Accordion title="De quel runtime ai-je besoin ?">
    Node **>= 22** est requis. `pnpm` est recommandé. Bun est **déconseillé** pour la Gateway.
  </Accordion>

  <Accordion title="Est-ce que ça fonctionne sur Raspberry Pi ?">
    Oui. La Gateway est légère : la documentation indique que **512 Mo-1 Go de RAM**, **1 cœur** et environ **500 Mo**
    de disque suffisent pour un usage personnel, et précise qu'un **Raspberry Pi 4 peut l'exécuter**.

    Si vous voulez plus de marge (journaux, médias, autres services), **2 Go sont recommandés**, mais ce
    n'est pas un minimum strict.

    Conseil : un petit Pi/VPS peut héberger la Gateway, et vous pouvez appairer des **nodes** sur votre ordinateur/téléphone pour
    l'écran/caméra/canvas local ou l'exécution de commandes. Voir [Nodes](/fr/nodes).

  </Accordion>

  <Accordion title="Des conseils pour les installations sur Raspberry Pi ?">
    En bref : ça fonctionne, mais attendez-vous à quelques aspérités.

    - Utilisez un OS **64 bits** et gardez Node >= 22.
    - Préférez l'**installation modifiable (git)** pour pouvoir voir les journaux et mettre à jour rapidement.
    - Commencez sans canaux/Skills, puis ajoutez-les un par un.
    - Si vous rencontrez des problèmes binaires étranges, il s'agit généralement d'un problème de **compatibilité ARM**.

    Documentation : [Linux](/fr/platforms/linux), [Install](/fr/install).

  </Accordion>

  <Accordion title="C'est bloqué sur wake up my friend / l'onboarding n'éclot pas. Que faire ?">
    Cet écran dépend d'une Gateway accessible et authentifiée. Le TUI envoie aussi
    automatiquement « Wake up, my friend! » au premier démarrage. Si vous voyez cette ligne **sans réponse**
    et que les jetons restent à 0, l'agent ne s'est jamais exécuté.

    1. Redémarrez la Gateway :

    ```bash
    openclaw gateway restart
    ```

    2. Vérifiez l'état + l'authentification :

    ```bash
    openclaw status
    openclaw models status
    openclaw logs --follow
    ```

    3. Si ça bloque toujours, exécutez :

    ```bash
    openclaw doctor
    ```

    Si la Gateway est distante, assurez-vous que le tunnel/la connexion Tailscale est actif et que l'UI
    pointe vers la bonne Gateway. Voir [Accès distant](/fr/gateway/remote).

  </Accordion>

  <Accordion title="Puis-je migrer ma configuration vers une nouvelle machine (Mac mini) sans refaire l'onboarding ?">
    Oui. Copiez le **répertoire d'état** et le **workspace**, puis exécutez Doctor une fois. Cela
    conserve votre bot « exactement identique » (mémoire, historique de session, auth et
    état des canaux) tant que vous copiez **les deux** emplacements :

    1. Installez OpenClaw sur la nouvelle machine.
    2. Copiez `$OPENCLAW_STATE_DIR` (par défaut : `~/.openclaw`) depuis l'ancienne machine.
    3. Copiez votre workspace (par défaut : `~/.openclaw/workspace`).
    4. Exécutez `openclaw doctor` et redémarrez le service Gateway.

    Cela préserve la configuration, les profils d'authentification, les identifiants WhatsApp, les sessions et la mémoire. Si vous êtes en
    mode distant, rappelez-vous que l'hôte de la passerelle possède le stockage des sessions et le workspace.

    **Important :** si vous ne faites que commit/push votre workspace sur GitHub, vous sauvegardez
    **la mémoire + les fichiers bootstrap**, mais **pas** l'historique de session ni l'authentification. Ceux-ci se trouvent
    sous `~/.openclaw/` (par exemple `~/.openclaw/agents/<agentId>/sessions/`).

    Voir aussi : [Migration](/fr/install/migrating), [Emplacement des fichiers sur disque](#where-things-live-on-disk),
    [Workspace de l'agent](/fr/concepts/agent-workspace), [Doctor](/fr/gateway/doctor),
    [Mode distant](/fr/gateway/remote).

  </Accordion>

  <Accordion title="Où puis-je voir les nouveautés de la dernière version ?">
    Consultez le changelog GitHub :
    [https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md)

    Les entrées les plus récentes sont en haut. Si la section du haut est marquée **Unreleased**, la section datée suivante
    correspond à la dernière version publiée. Les entrées sont regroupées par **Highlights**, **Changes** et
    **Fixes** (plus des sections docs/autres si nécessaire).

  </Accordion>

  <Accordion title="Impossible d'accéder à docs.openclaw.ai (erreur SSL)">
    Certaines connexions Comcast/Xfinity bloquent incorrectement `docs.openclaw.ai` via Xfinity
    Advanced Security. Désactivez-le ou ajoutez `docs.openclaw.ai` à la liste d'autorisation, puis réessayez.
    Merci de nous aider à le débloquer en le signalant ici : [https://spa.xfinity.com/check_url_status](https://spa.xfinity.com/check_url_status).

    Si vous n'arrivez toujours pas à accéder au site, la documentation est dupliquée sur GitHub :
    [https://github.com/openclaw/openclaw/tree/main/docs](https://github.com/openclaw/openclaw/tree/main/docs)

  </Accordion>

  <Accordion title="Différence entre stable et beta">
    **Stable** et **beta** sont des **npm dist-tags**, pas des lignes de code séparées :

    - `latest` = stable
    - `beta` = build précoce pour les tests

    En général, une version stable arrive d'abord sur **beta**, puis une étape explicite
    de promotion déplace cette même version vers `latest`. Les mainteneurs peuvent aussi
    publier directement sur `latest` si nécessaire. C'est pourquoi beta et stable peuvent
    pointer vers la **même version** après promotion.

    Voir ce qui a changé :
    [https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md)

    Pour les commandes d'installation en une ligne et la différence entre beta et dev, voir l'accordéon ci-dessous.

  </Accordion>

  <Accordion title="Comment installer la version beta et quelle est la différence entre beta et dev ?">
    **Beta** est le dist-tag npm `beta` (peut correspondre à `latest` après promotion).
    **Dev** est la tête mobile de `main` (git) ; lorsqu'elle est publiée, elle utilise le dist-tag npm `dev`.

    Commandes en une ligne (macOS/Linux) :

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --beta
    ```

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Installateur Windows (PowerShell) :
    [https://openclaw.ai/install.ps1](https://openclaw.ai/install.ps1)

    Plus de détails : [Canaux de développement](/fr/install/development-channels) et [Options de l'installateur](/fr/install/installer).

  </Accordion>

  <Accordion title="Comment essayer les derniers éléments ?">
    Deux options :

    1. **Canal dev (copie git) :**

    ```bash
    openclaw update --channel dev
    ```

    Cela bascule vers la branche `main` et met à jour depuis les sources.

    2. **Installation modifiable (depuis le site de l'installateur) :**

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Cela vous donne un dépôt local que vous pouvez modifier, puis mettre à jour via git.

    Si vous préférez faire un clone propre manuellement, utilisez :

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    ```

    Documentation : [Mise à jour](/cli/update), [Canaux de développement](/fr/install/development-channels),
    [Install](/fr/install).

  </Accordion>

  <Accordion title="Combien de temps prennent généralement l'installation et l'onboarding ?">
    Estimation approximative :

    - **Installation :** 2-5 minutes
    - **Onboarding :** 5-15 minutes selon le nombre de canaux/modèles que vous configurez

    Si cela bloque, utilisez [Installateur bloqué](#quick-start-and-first-run-setup)
    et la boucle de débogage rapide dans [Je suis bloqué](#quick-start-and-first-run-setup).

  </Accordion>

  <Accordion title="Installateur bloqué ? Comment obtenir plus d'informations ?">
    Relancez l'installateur avec une **sortie verbeuse** :

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --verbose
    ```

    Installation beta avec sortie verbeuse :

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --beta --verbose
    ```

    Pour une installation modifiable (git) :

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git --verbose
    ```

    Équivalent Windows (PowerShell) :

    ```powershell
    # install.ps1 has no dedicated -Verbose flag yet.
    Set-PSDebug -Trace 1
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -NoOnboard
    Set-PSDebug -Trace 0
    ```

    Plus d'options : [Options de l'installateur](/fr/install/installer).

  </Accordion>

  <Accordion title="L'installation Windows indique git not found ou openclaw not recognized">
    Deux problèmes Windows courants :

    **1) erreur npm spawn git / git not found**

    - Installez **Git for Windows** et assurez-vous que `git` est sur votre PATH.
    - Fermez et rouvrez PowerShell, puis relancez l'installateur.

    **2) openclaw is not recognized après l'installation**

    - Le dossier npm global bin n'est pas sur le PATH.
    - Vérifiez le chemin :

      ```powershell
      npm config get prefix
      ```

    - Ajoutez ce répertoire à votre PATH utilisateur (pas besoin du suffixe `\bin` sur Windows ; sur la plupart des systèmes c'est `%AppData%\npm`).
    - Fermez et rouvrez PowerShell après la mise à jour du PATH.

    Si vous voulez la configuration Windows la plus fluide, utilisez **WSL2** au lieu de Windows natif.
    Documentation : [Windows](/fr/platforms/windows).

  </Accordion>

  <Accordion title="La sortie exec sur Windows affiche du texte chinois illisible - que faire ?">
    Il s'agit généralement d'un décalage de page de code console dans les shells Windows natifs.

    Symptômes :

    - la sortie `system.run`/`exec` affiche le chinois sous forme de mojibake
    - la même commande s'affiche correctement dans un autre profil de terminal

    Solution rapide dans PowerShell :

    ```powershell
    chcp 65001
    [Console]::InputEncoding = [System.Text.UTF8Encoding]::new($false)
    [Console]::OutputEncoding = [System.Text.UTF8Encoding]::new($false)
    $OutputEncoding = [System.Text.UTF8Encoding]::new($false)
    ```

    Puis redémarrez la Gateway et réessayez votre commande :

    ```powershell
    openclaw gateway restart
    ```

    Si vous reproduisez toujours cela avec la dernière version d'OpenClaw, suivez/signez le problème ici :

    - [Issue #30640](https://github.com/openclaw/openclaw/issues/30640)

  </Accordion>

  <Accordion title="La documentation n'a pas répondu à ma question - comment obtenir une meilleure réponse ?">
    Utilisez l'**installation modifiable (git)** afin d'avoir localement les sources et la documentation complètes, puis demandez
    à votre bot (ou à Claude/Codex) _depuis ce dossier_ afin qu'il puisse lire le dépôt et répondre précisément.

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Plus de détails : [Install](/fr/install) et [Options de l'installateur](/fr/install/installer).

  </Accordion>

  <Accordion title="Comment installer OpenClaw sur Linux ?">
    Réponse courte : suivez le guide Linux, puis lancez l'onboarding.

    - Chemin rapide Linux + installation du service : [Linux](/fr/platforms/linux).
    - Guide complet : [Premiers pas](/fr/start/getting-started).
    - Installateur + mises à jour : [Installation et mises à jour](/fr/install/updating).

  </Accordion>

  <Accordion title="Comment installer OpenClaw sur un VPS ?">
    Tout VPS Linux fonctionne. Installez-le sur le serveur, puis utilisez SSH/Tailscale pour atteindre la Gateway.

    Guides : [exe.dev](/fr/install/exe-dev), [Hetzner](/fr/install/hetzner), [Fly.io](/fr/install/fly).
    Accès distant : [Gateway distante](/fr/gateway/remote).

  </Accordion>

  <Accordion title="Où sont les guides d'installation cloud/VPS ?">
    Nous maintenons un **hub d'hébergement** avec les fournisseurs courants. Choisissez-en un et suivez le guide :

    - [Hébergement VPS](/fr/vps) (tous les fournisseurs au même endroit)
    - [Fly.io](/fr/install/fly)
    - [Hetzner](/fr/install/hetzner)
    - [exe.dev](/fr/install/exe-dev)

    Fonctionnement dans le cloud : la **Gateway s'exécute sur le serveur**, et vous y accédez
    depuis votre ordinateur/téléphone via Control UI (ou Tailscale/SSH). Votre état + workspace
    vivent sur le serveur, traitez donc l'hôte comme source de vérité et sauvegardez-le.

    Vous pouvez appairer des **nodes** (Mac/iOS/Android/headless) à cette Gateway cloud pour accéder
    à l'écran/caméra/canvas local ou exécuter des commandes sur votre ordinateur tout en gardant la
    Gateway dans le cloud.

    Hub : [Plateformes](/fr/platforms). Accès distant : [Gateway distante](/fr/gateway/remote).
    Nodes : [Nodes](/fr/nodes), [CLI Nodes](/cli/nodes).

  </Accordion>

  <Accordion title="Puis-je demander à OpenClaw de se mettre à jour lui-même ?">
    Réponse courte : **possible, mais déconseillé**. Le flux de mise à jour peut redémarrer la
    Gateway (ce qui coupe la session active), peut nécessiter une copie git propre et
    peut demander confirmation. Plus sûr : exécutez les mises à jour depuis un shell en tant qu'opérateur.

    Utilisez la CLI :

    ```bash
    openclaw update
    openclaw update status
    openclaw update --channel stable|beta|dev
    openclaw update --tag <dist-tag|version>
    openclaw update --no-restart
    ```

    Si vous devez automatiser depuis un agent :

    ```bash
    openclaw update --yes --no-restart
    openclaw gateway restart
    ```

    Documentation : [Mise à jour](/cli/update), [Mise à jour](/fr/install/updating).

  </Accordion>

  <Accordion title="Que fait réellement l'onboarding ?">
    `openclaw onboard` est le chemin de configuration recommandé. En **mode local**, il vous guide pour :

    - **Configuration du modèle/de l'authentification** (OAuth fournisseur, clés API, setup-token Anthropic, plus options de modèles locaux comme LM Studio)
    - emplacement du **workspace** + fichiers bootstrap
    - **Paramètres de la Gateway** (liaison/port/auth/tailscale)
    - **Canaux** (WhatsApp, Telegram, Discord, Mattermost, Signal, iMessage, plus des plugins de canal groupés comme QQ Bot)
    - **Installation du démon** (LaunchAgent sur macOS ; unité utilisateur systemd sur Linux/WSL2)
    - **Vérifications d'état** et sélection des **Skills**

    Il avertit aussi si votre modèle configuré est inconnu ou si l'authentification manque.

  </Accordion>

  <Accordion title="Ai-je besoin d'un abonnement Claude ou OpenAI pour exécuter cela ?">
    Non. Vous pouvez exécuter OpenClaw avec des **clés API** (Anthropic/OpenAI/autres) ou avec des
    **modèles uniquement locaux** pour que vos données restent sur votre appareil. Les abonnements (Claude
    Pro/Max ou OpenAI Codex) sont des moyens facultatifs d'authentifier ces fournisseurs.

    Pour Anthropic dans OpenClaw, la distinction pratique est :

    - **Clé API Anthropic** : facturation normale de l'API Anthropic
    - **Claude CLI / authentification par abonnement Claude dans OpenClaw** : le personnel d'Anthropic
      nous a indiqué que cet usage est de nouveau autorisé, et OpenClaw considère l'usage `claude -p`
      comme approuvé pour cette intégration sauf si Anthropic publie une nouvelle
      politique

    Pour les hôtes Gateway de longue durée, les clés API Anthropic restent la configuration la plus
    prévisible. OpenAI Codex OAuth est explicitement pris en charge pour des
    outils externes comme OpenClaw.

    OpenClaw prend également en charge d'autres options hébergées de type abonnement, notamment
    **Qwen Cloud Coding Plan**, **MiniMax Coding Plan**, et
    **Z.AI / GLM Coding Plan**.

    Documentation : [Anthropic](/fr/providers/anthropic), [OpenAI](/fr/providers/openai),
    [Qwen Cloud](/fr/providers/qwen),
    [MiniMax](/fr/providers/minimax), [GLM Models](/fr/providers/glm),
    [Modèles locaux](/fr/gateway/local-models), [Models](/fr/concepts/models).

  </Accordion>

  <Accordion title="Puis-je utiliser l'abonnement Claude Max sans clé API ?">
    Oui.

    Le personnel d'Anthropic nous a indiqué que l'usage de Claude CLI dans le style OpenClaw est de nouveau autorisé, donc
    OpenClaw considère l'authentification par abonnement Claude et l'usage de `claude -p` comme approuvés
    pour cette intégration sauf si Anthropic publie une nouvelle politique. Si vous voulez
    la configuration côté serveur la plus prévisible, utilisez plutôt une clé API Anthropic.

  </Accordion>

  <Accordion title="Prenez-vous en charge l'authentification par abonnement Claude (Claude Pro ou Max) ?">
    Oui.

    Le personnel d'Anthropic nous a indiqué que cet usage est de nouveau autorisé, donc OpenClaw considère
    la réutilisation de Claude CLI et l'usage de `claude -p` comme approuvés pour cette intégration
    sauf si Anthropic publie une nouvelle politique.

    Le setup-token Anthropic reste disponible comme chemin de jeton OpenClaw pris en charge, mais OpenClaw privilégie désormais la réutilisation de Claude CLI et `claude -p` lorsqu'ils sont disponibles.
    Pour des charges de travail de production ou multi-utilisateur, l'authentification par clé API Anthropic reste
    le choix le plus sûr et le plus prévisible. Si vous voulez d'autres options hébergées de type abonnement
    dans OpenClaw, voir [OpenAI](/fr/providers/openai), [Qwen / Model
    Cloud](/fr/providers/qwen), [MiniMax](/fr/providers/minimax), et [GLM
    Models](/fr/providers/glm).

  </Accordion>

<a id="why-am-i-seeing-http-429-ratelimiterror-from-anthropic"></a>
<Accordion title="Pourquoi vois-je HTTP 429 rate_limit_error depuis Anthropic ?">
Cela signifie que votre **quota/limite de débit Anthropic** est épuisé pour la fenêtre en cours. Si vous
utilisez **Claude CLI**, attendez la réinitialisation de la fenêtre ou mettez à niveau votre forfait. Si vous
utilisez une **clé API Anthropic**, vérifiez la console Anthropic
pour l'utilisation/la facturation et augmentez les limites si nécessaire.

    Si le message est spécifiquement :
    `Extra usage is required for long context requests`, la requête tente d'utiliser
    la bêta de contexte 1M d'Anthropic (`context1m: true`). Cela ne fonctionne que si votre
    identifiant est éligible à la facturation long contexte (facturation par clé API ou
    chemin de connexion Claude OpenClaw avec Extra Usage activé).

    Conseil : définissez un **modèle de secours** afin qu'OpenClaw puisse continuer à répondre pendant qu'un fournisseur est limité.
    Voir [Models](/cli/models), [OAuth](/fr/concepts/oauth), et
    [/gateway/troubleshooting#anthropic-429-extra-usage-required-for-long-context](/fr/gateway/troubleshooting#anthropic-429-extra-usage-required-for-long-context).

  </Accordion>

  <Accordion title="AWS Bedrock est-il pris en charge ?">
    Oui. OpenClaw inclut un fournisseur groupé **Amazon Bedrock (Converse)**. En présence des marqueurs d'environnement AWS, OpenClaw peut découvrir automatiquement le catalogue Bedrock streaming/texte et le fusionner comme fournisseur implicite `amazon-bedrock` ; sinon vous pouvez activer explicitement `plugins.entries.amazon-bedrock.config.discovery.enabled` ou ajouter une entrée fournisseur manuelle. Voir [Amazon Bedrock](/fr/providers/bedrock) et [Fournisseurs de modèles](/fr/providers/models). Si vous préférez un flux géré par clé, un proxy compatible OpenAI devant Bedrock reste une option valide.
  </Accordion>

  <Accordion title="Comment fonctionne l'authentification Codex ?">
    OpenClaw prend en charge **OpenAI Code (Codex)** via OAuth (connexion ChatGPT). L'onboarding peut exécuter le flux OAuth et définira le modèle par défaut sur `openai-codex/gpt-5.4` lorsque cela est approprié. Voir [Fournisseurs de modèles](/fr/concepts/model-providers) et [Onboarding (CLI)](/fr/start/wizard).
  </Accordion>

  <Accordion title="Pourquoi ChatGPT GPT-5.4 ne déverrouille-t-il pas openai/gpt-5.4 dans OpenClaw ?">
    OpenClaw traite les deux voies séparément :

    - `openai-codex/gpt-5.4` = OAuth ChatGPT/Codex
    - `openai/gpt-5.4` = API directe OpenAI Platform

    Dans OpenClaw, la connexion ChatGPT/Codex est reliée à la voie `openai-codex/*`,
    et non à la voie directe `openai/*`. Si vous voulez la voie API directe dans
    OpenClaw, définissez `OPENAI_API_KEY` (ou la configuration fournisseur OpenAI équivalente).
    Si vous voulez la connexion ChatGPT/Codex dans OpenClaw, utilisez `openai-codex/*`.

  </Accordion>

  <Accordion title="Pourquoi les limites de Codex OAuth peuvent-elles différer de ChatGPT web ?">
    `openai-codex/*` utilise la voie OAuth Codex, et ses fenêtres de quota utilisables sont
    gérées par OpenAI et dépendent du forfait. En pratique, ces limites peuvent différer de
    l'expérience sur le site/l'application ChatGPT, même si les deux sont liées au même compte.

    OpenClaw peut afficher les fenêtres d'utilisation/quota actuellement visibles du fournisseur dans
    `openclaw models status`, mais il n'invente ni ne normalise les droits du web ChatGPT
    en accès API direct. Si vous voulez la voie directe de facturation/limite OpenAI Platform,
    utilisez `openai/*` avec une clé API.

  </Accordion>

  <Accordion title="Prenez-vous en charge l'authentification par abonnement OpenAI (Codex OAuth) ?">
    Oui. OpenClaw prend entièrement en charge **OpenAI Code (Codex) subscription OAuth**.
    OpenAI autorise explicitement l'usage d'OAuth par abonnement dans des outils/workflows externes
    comme OpenClaw. L'onboarding peut exécuter le flux OAuth pour vous.

    Voir [OAuth](/fr/concepts/oauth), [Fournisseurs de modèles](/fr/concepts/model-providers), et [Onboarding (CLI)](/fr/start/wizard).

  </Accordion>

  <Accordion title="Comment configurer Gemini CLI OAuth ?">
    Gemini CLI utilise un **flux d'authentification de plugin**, pas un client id ou secret dans `openclaw.json`.

    Étapes :

    1. Installez Gemini CLI localement pour que `gemini` soit sur le `PATH`
       - Homebrew : `brew install gemini-cli`
       - npm : `npm install -g @google/gemini-cli`
    2. Activez le plugin : `openclaw plugins enable google`
    3. Connectez-vous : `openclaw models auth login --provider google-gemini-cli --set-default`
    4. Modèle par défaut après connexion : `google-gemini-cli/gemini-3.1-pro-preview`
    5. Si les requêtes échouent, définissez `GOOGLE_CLOUD_PROJECT` ou `GOOGLE_CLOUD_PROJECT_ID` sur l'hôte de la passerelle

    Cela stocke les jetons OAuth dans les profils d'authentification sur l'hôte de la passerelle. Détails : [Fournisseurs de modèles](/fr/concepts/model-providers).

  </Accordion>

  <Accordion title="Un modèle local convient-il pour les discussions occasionnelles ?">
    En général non. OpenClaw a besoin d'un grand contexte + d'une sécurité solide ; les petites cartes tronquent et laissent fuiter. Si vous devez le faire, exécutez localement la **plus grosse** version de modèle possible (LM Studio) et consultez [/gateway/local-models](/fr/gateway/local-models). Les modèles plus petits/quantifiés augmentent le risque d'injection de prompt - voir [Sécurité](/fr/gateway/security).
  </Accordion>

  <Accordion title="Comment garder le trafic de modèles hébergés dans une région spécifique ?">
    Choisissez des endpoints épinglés à une région. OpenRouter expose des options hébergées aux États-Unis pour MiniMax, Kimi et GLM ; choisissez la variante hébergée aux États-Unis pour garder les données dans la région. Vous pouvez toujours lister Anthropic/OpenAI à côté en utilisant `models.mode: "merge"` afin de garder des secours disponibles tout en respectant le fournisseur régional sélectionné.
  </Accordion>

  <Accordion title="Dois-je acheter un Mac Mini pour installer cela ?">
    Non. OpenClaw fonctionne sur macOS ou Linux (Windows via WSL2). Un Mac mini est optionnel - certaines personnes
    en achètent un comme hôte toujours actif, mais un petit VPS, un serveur domestique ou un boîtier de classe Raspberry Pi fonctionne aussi.

    Vous n'avez besoin d'un Mac **que pour les outils réservés à macOS**. Pour iMessage, utilisez [BlueBubbles](/fr/channels/bluebubbles) (recommandé) - le serveur BlueBubbles fonctionne sur n'importe quel Mac, et la Gateway peut fonctionner sur Linux ou ailleurs. Si vous voulez d'autres outils réservés à macOS, exécutez la Gateway sur un Mac ou appairez un nœud macOS.

    Documentation : [BlueBubbles](/fr/channels/bluebubbles), [Nodes](/fr/nodes), [Mode distant Mac](/fr/platforms/mac/remote).

  </Accordion>

  <Accordion title="Ai-je besoin d'un Mac mini pour la prise en charge d'iMessage ?">
    Vous avez besoin d'**un appareil macOS** connecté à Messages. Ce **n'est pas obligé** d'être un Mac mini -
    n'importe quel Mac convient. **Utilisez [BlueBubbles](/fr/channels/bluebubbles)** (recommandé) pour iMessage - le serveur BlueBubbles fonctionne sur macOS, tandis que la Gateway peut fonctionner sur Linux ou ailleurs.

    Configurations courantes :

    - Exécutez la Gateway sur Linux/VPS, et le serveur BlueBubbles sur n'importe quel Mac connecté à Messages.
    - Exécutez tout sur le Mac si vous voulez la configuration la plus simple sur une seule machine.

    Documentation : [BlueBubbles](/fr/channels/bluebubbles), [Nodes](/fr/nodes),
    [Mode distant Mac](/fr/platforms/mac/remote).

  </Accordion>

  <Accordion title="Si j'achète un Mac mini pour exécuter OpenClaw, puis-je le connecter à mon MacBook Pro ?">
    Oui. Le **Mac mini peut exécuter la Gateway**, et votre MacBook Pro peut se connecter comme
    **node** (appareil compagnon). Les nodes n'exécutent pas la Gateway - ils fournissent des
    capacités supplémentaires comme écran/caméra/canvas et `system.run` sur cet appareil.

    Schéma courant :

    - Gateway sur le Mac mini (toujours actif).
    - Le MacBook Pro exécute l'application macOS ou un hôte de nœud et s'appaire à la Gateway.
    - Utilisez `openclaw nodes status` / `openclaw nodes list` pour le voir.

    Documentation : [Nodes](/fr/nodes), [CLI Nodes](/cli/nodes).

  </Accordion>

  <Accordion title="Puis-je utiliser Bun ?">
    Bun est **déconseillé**. Nous observons des bugs d'exécution, surtout avec WhatsApp et Telegram.
    Utilisez **Node** pour des passerelles stables.

    Si vous voulez quand même expérimenter avec Bun, faites-le sur une passerelle non production
    sans WhatsApp/Telegram.

  </Accordion>

  <Accordion title="Telegram : que faut-il mettre dans allowFrom ?">
    `channels.telegram.allowFrom` correspond à **l'ID utilisateur Telegram de la personne** (numérique). Ce n'est pas le nom d'utilisateur du bot.

    L'onboarding accepte une entrée `@username` et la résout en ID numérique, mais l'autorisation OpenClaw utilise uniquement des ID numériques.

    Plus sûr (sans bot tiers) :

    - Envoyez un DM à votre bot, puis exécutez `openclaw logs --follow` et lisez `from.id`.

    API officielle Bot :

    - Envoyez un DM à votre bot, puis appelez `https://api.telegram.org/bot<bot_token>/getUpdates` et lisez `message.from.id`.

    Tiers (moins privé) :

    - Envoyez un DM à `@userinfobot` ou `@getidsbot`.

    Voir [/channels/telegram](/fr/channels/telegram#access-control-and-activation).

  </Accordion>

  <Accordion title="Plusieurs personnes peuvent-elles utiliser un même numéro WhatsApp avec différentes instances OpenClaw ?">
    Oui, via le **routage multi-agent**. Associez le **DM** WhatsApp de chaque expéditeur (pair `kind: "direct"`, expéditeur E.164 comme `+15551234567`) à un `agentId` différent, afin que chaque personne ait son propre workspace et son propre stockage de sessions. Les réponses proviennent toujours du **même compte WhatsApp**, et le contrôle d'accès DM (`channels.whatsapp.dmPolicy` / `channels.whatsapp.allowFrom`) est global par compte WhatsApp. Voir [Routage multi-agent](/fr/concepts/multi-agent) et [WhatsApp](/fr/channels/whatsapp).
  </Accordion>

  <Accordion title='Puis-je exécuter un agent « fast chat » et un agent « Opus for coding » ?'>
    Oui. Utilisez le routage multi-agent : donnez à chaque agent son propre modèle par défaut, puis associez les routes entrantes (compte fournisseur ou pairs spécifiques) à chacun. Un exemple de configuration figure dans [Routage multi-agent](/fr/concepts/multi-agent). Voir aussi [Models](/fr/concepts/models) et [Configuration](/fr/gateway/configuration).
  </Accordion>

  <Accordion title="Homebrew fonctionne-t-il sur Linux ?">
    Oui. Homebrew prend en charge Linux (Linuxbrew). Configuration rapide :

    ```bash
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
    echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"' >> ~/.profile
    eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
    brew install <formula>
    ```

    Si vous exécutez OpenClaw via systemd, assurez-vous que le PATH du service inclut `/home/linuxbrew/.linuxbrew/bin` (ou votre préfixe brew) afin que les outils installés via `brew` soient résolus dans les shells sans connexion.
    Les builds récentes préfixent également les répertoires bin utilisateur courants sur les services systemd Linux (par exemple `~/.local/bin`, `~/.npm-global/bin`, `~/.local/share/pnpm`, `~/.bun/bin`) et honorent `PNPM_HOME`, `NPM_CONFIG_PREFIX`, `BUN_INSTALL`, `VOLTA_HOME`, `ASDF_DATA_DIR`, `NVM_DIR` et `FNM_DIR` lorsqu'ils sont définis.

  </Accordion>

  <Accordion title="Différence entre l'installation git modifiable et npm install">
    - **Installation modifiable (git)** : copie complète des sources, modifiable, idéale pour les contributeurs.
      Vous exécutez les builds localement et pouvez corriger le code/la documentation.
    - **npm install** : installation globale de la CLI, sans dépôt, idéale pour « simplement l'exécuter ».
      Les mises à jour proviennent des dist-tags npm.

    Documentation : [Premiers pas](/fr/start/getting-started), [Mise à jour](/fr/install/updating).

  </Accordion>

  <Accordion title="Puis-je basculer plus tard entre les installations npm et git ?">
    Oui. Installez l'autre variante, puis exécutez Doctor pour que le service de passerelle pointe vers le nouveau point d'entrée.
    Cela **ne supprime pas vos données** - cela change uniquement l'installation du code OpenClaw. Votre état
    (`~/.openclaw`) et votre workspace (`~/.openclaw/workspace`) restent intacts.

    De npm vers git :

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    openclaw doctor
    openclaw gateway restart
    ```

    De git vers npm :

    ```bash
    npm install -g openclaw@latest
    openclaw doctor
    openclaw gateway restart
    ```

    Doctor détecte une divergence de point d'entrée du service Gateway et propose de réécrire la configuration du service pour correspondre à l'installation actuelle (utilisez `--repair` dans l'automatisation).

    Conseils de sauvegarde : voir [Stratégie de sauvegarde](#where-things-live-on-disk).

  </Accordion>

  <Accordion title="Dois-je exécuter la Gateway sur mon ordinateur portable ou sur un VPS ?">
    Réponse courte : **si vous voulez une fiabilité 24/7, utilisez un VPS**. Si vous voulez le
    moins de friction possible et que les mises en veille/redémarrages vous conviennent, exécutez-la localement.

    **Ordinateur portable (Gateway locale)**

    - **Avantages :** pas de coût serveur, accès direct aux fichiers locaux, fenêtre de navigateur visible.
    - **Inconvénients :** veille/perte réseau = déconnexions, mises à jour/redémarrages du système interrompent, la machine doit rester réveillée.

    **VPS / cloud**

    - **Avantages :** toujours actif, réseau stable, pas de problèmes de veille du portable, plus facile à maintenir en fonctionnement.
    - **Inconvénients :** souvent headless (utilisez des captures d'écran), accès uniquement aux fichiers distants, vous devez utiliser SSH pour les mises à jour.

    **Remarque spécifique à OpenClaw :** WhatsApp/Telegram/Slack/Mattermost/Discord fonctionnent tous très bien depuis un VPS. Le seul vrai compromis est **navigateur headless** versus fenêtre visible. Voir [Browser](/fr/tools/browser).

    **Recommandation par défaut :** VPS si vous avez déjà eu des déconnexions de passerelle. Le local est excellent quand vous utilisez activement le Mac et voulez un accès local aux fichiers ou une automatisation UI avec un navigateur visible.

  </Accordion>

  <Accordion title="Est-il important d'exécuter OpenClaw sur une machine dédiée ?">
    Ce n'est pas obligatoire, mais **recommandé pour la fiabilité et l'isolation**.

    - **Hôte dédié (VPS/Mac mini/Pi) :** toujours actif, moins d'interruptions dues à la veille/aux redémarrages, permissions plus propres, plus facile à maintenir.
    - **Ordinateur portable/de bureau partagé :** tout à fait correct pour les tests et l'usage actif, mais attendez-vous à des pauses lorsque la machine se met en veille ou se met à jour.

    Si vous voulez le meilleur des deux mondes, gardez la Gateway sur un hôte dédié et appairez votre ordinateur portable comme **node** pour les outils locaux écran/caméra/exec. Voir [Nodes](/fr/nodes).
    Pour les recommandations de sécurité, consultez [Sécurité](/fr/gateway/security).

  </Accordion>

  <Accordion title="Quelles sont les exigences minimales pour un VPS et quel OS est recommandé ?">
    OpenClaw est léger. Pour une Gateway de base + un canal de chat :

    - **Minimum absolu :** 1 vCPU, 1 Go de RAM, ~500 Mo de disque.
    - **Recommandé :** 1-2 vCPU, 2 Go de RAM ou plus pour de la marge (journaux, médias, plusieurs canaux). Les outils Node et l'automatisation du navigateur peuvent être gourmands en ressources.

    OS : utilisez **Ubuntu LTS** (ou tout Debian/Ubuntu moderne). Le chemin d'installation Linux y est le mieux testé.

    Documentation : [Linux](/fr/platforms/linux), [Hébergement VPS](/fr/vps).

  </Accordion>

  <Accordion title="Puis-je exécuter OpenClaw dans une VM et quelles sont les exigences ?">
    Oui. Traitez une VM comme un VPS : elle doit être toujours active, accessible et disposer de suffisamment
    de RAM pour la Gateway et les canaux que vous activez.

    Recommandations de base :

    - **Minimum absolu :** 1 vCPU, 1 Go de RAM.
    - **Recommandé :** 2 Go de RAM ou plus si vous exécutez plusieurs canaux, de l'automatisation navigateur ou des outils média.
    - **OS :** Ubuntu LTS ou une autre distribution Debian/Ubuntu moderne.

    Si vous êtes sous Windows, **WSL2 est la configuration de style VM la plus simple** et offre la meilleure
    compatibilité d'outillage. Voir [Windows](/fr/platforms/windows), [Hébergement VPS](/fr/vps).
    Si vous exécutez macOS dans une VM, voir [VM macOS](/fr/install/macos-vm).

  </Accordion>
</AccordionGroup>

## Qu'est-ce qu'OpenClaw ?

<AccordionGroup>
  <Accordion title="Qu'est-ce qu'OpenClaw, en un paragraphe ?">
    OpenClaw est un assistant IA personnel que vous exécutez sur vos propres appareils. Il répond sur les surfaces de messagerie que vous utilisez déjà (WhatsApp, Telegram, Slack, Mattermost, Discord, Google Chat, Signal, iMessage, WebChat et des plugins de canal groupés tels que QQ Bot) et peut aussi faire de la voix + un Canvas en direct sur les plateformes prises en charge. La **Gateway** est le plan de contrôle toujours actif ; l'assistant est le produit.
  </Accordion>

  <Accordion title="Proposition de valeur">
    OpenClaw n'est pas « juste un wrapper Claude ». C'est un **plan de contrôle local-first** qui vous permet d'exécuter un
    assistant performant sur **votre propre matériel**, accessible depuis les applications de chat que vous utilisez déjà, avec
    des sessions avec état, de la mémoire et des outils - sans confier le contrôle de vos workflows à un
    SaaS hébergé.

    Points forts :

    - **Vos appareils, vos données :** exécutez la Gateway où vous voulez (Mac, Linux, VPS) et gardez
      le workspace + l'historique des sessions en local.
    - **De vrais canaux, pas un bac à sable web :** WhatsApp/Telegram/Slack/Discord/Signal/iMessage/etc,
      plus la voix mobile et Canvas sur les plateformes prises en charge.
    - **Agnostique au modèle :** utilisez Anthropic, OpenAI, MiniMax, OpenRouter, etc., avec routage
      par agent et basculement.
    - **Option uniquement locale :** exécutez des modèles locaux pour que **toutes les données puissent rester sur votre appareil** si vous le souhaitez.
    - **Routage multi-agent :** agents séparés par canal, compte ou tâche, chacun avec son propre
      workspace et ses valeurs par défaut.
    - **Open source et modifiable :** inspectez, étendez et auto-hébergez sans verrouillage fournisseur.

    Documentation : [Gateway](/fr/gateway), [Canaux](/fr/channels), [Multi-agent](/fr/concepts/multi-agent),
    [Mémoire](/fr/concepts/memory).

  </Accordion>

  <Accordion title="Je viens de l'installer - que devrais-je faire en premier ?">
    Bons premiers projets :

    - Créer un site web (WordPress, Shopify ou un simple site statique).
    - Prototyper une application mobile (plan, écrans, plan d'API).
    - Organiser fichiers et dossiers (nettoyage, nommage, étiquetage).
    - Connecter Gmail et automatiser les résumés ou suivis.

    Il peut gérer de grandes tâches, mais il fonctionne mieux si vous les divisez en phases et
    utilisez des sous-agents pour le travail parallèle.

  </Accordion>

  <Accordion title="Quels sont les cinq principaux cas d'usage quotidiens pour OpenClaw ?">
    Les gains du quotidien ressemblent généralement à ceci :

    - **Briefings personnels :** résumés de boîte de réception, calendrier et actualités qui vous importent.
    - **Recherche et rédaction :** recherche rapide, résumés et premiers brouillons pour des e-mails ou des documents.
    - **Rappels et suivis :** relances et listes de contrôle pilotées par cron ou heartbeat.
    - **Automatisation navigateur :** remplir des formulaires, collecter des données et répéter des tâches web.
    - **Coordination inter-appareils :** envoyez une tâche depuis votre téléphone, laissez la Gateway l'exécuter sur un serveur, puis récupérez le résultat dans le chat.

  </Accordion>

  <Accordion title="OpenClaw peut-il aider pour la génération de leads, l'outreach, la pub et les blogs d'un SaaS ?">
    Oui pour la **recherche, la qualification et la rédaction**. Il peut scanner des sites, construire des listes restreintes,
    résumer des prospects et rédiger des brouillons d'outreach ou de publicité.

    Pour les **campagnes d'outreach ou de publicité**, gardez un humain dans la boucle. Évitez le spam, respectez les lois locales et
    les politiques de plateforme, et relisez tout avant envoi. Le schéma le plus sûr est de laisser
    OpenClaw rédiger et vous approuver.

    Documentation : [Sécurité](/fr/gateway/security).

  </Accordion>

  <Accordion title="Quels sont les avantages par rapport à Claude Code pour le développement web ?">
    OpenClaw est un **assistant personnel** et une couche de coordination, pas un remplacement d'IDE. Utilisez
    Claude Code ou Codex pour la boucle de codage directe la plus rapide dans un dépôt. Utilisez OpenClaw lorsque vous
    voulez une mémoire durable, un accès inter-appareils et une orchestration d'outils.

    Avantages :

    - **Mémoire persistante + workspace** entre les sessions
    - **Accès multiplateforme** (WhatsApp, Telegram, TUI, WebChat)
    - **Orchestration d'outils** (navigateur, fichiers, planification, hooks)
    - **Gateway toujours active** (exécutez-la sur un VPS, interagissez depuis n'importe où)
    - **Nodes** pour navigateur/écran/caméra/exec locaux

    Présentation : [https://openclaw.ai/showcase](https://openclaw.ai/showcase)

  </Accordion>
</AccordionGroup>

## Skills et automatisation

<AccordionGroup>
  <Accordion title="Comment personnaliser les Skills sans garder le dépôt modifié ?">
    Utilisez des remplacements gérés au lieu de modifier la copie du dépôt. Placez vos changements dans `~/.openclaw/skills/<name>/SKILL.md` (ou ajoutez un dossier via `skills.load.extraDirs` dans `~/.openclaw/openclaw.json`). L'ordre de priorité est `<workspace>/skills` → `<workspace>/.agents/skills` → `~/.agents/skills` → `~/.openclaw/skills` → intégré → `skills.load.extraDirs`, donc les remplacements gérés gardent toujours la priorité sur les Skills intégrés sans toucher à git. Si vous avez besoin que le skill soit installé globalement mais visible seulement pour certains agents, gardez la copie partagée dans `~/.openclaw/skills` et contrôlez la visibilité avec `agents.defaults.skills` et `agents.list[].skills`. Seules les modifications dignes d'être intégrées en amont devraient vivre dans le dépôt et partir en PR.
  </Accordion>

  <Accordion title="Puis-je charger des Skills depuis un dossier personnalisé ?">
    Oui. Ajoutez des répertoires supplémentaires via `skills.load.extraDirs` dans `~/.openclaw/openclaw.json` (priorité la plus basse). L'ordre par défaut est `<workspace>/skills` → `<workspace>/.agents/skills` → `~/.agents/skills` → `~/.openclaw/skills` → intégré → `skills.load.extraDirs`. `clawhub` installe dans `./skills` par défaut, qu'OpenClaw traite comme `<workspace>/skills` à la session suivante. Si le skill ne doit être visible que pour certains agents, associez-le à `agents.defaults.skills` ou `agents.list[].skills`.
  </Accordion>

  <Accordion title="Comment utiliser différents modèles pour différentes tâches ?">
    Aujourd'hui, les schémas pris en charge sont :

    - **Tâches cron** : les tâches isolées peuvent définir un remplacement `model` par tâche.
    - **Sous-agents** : routez les tâches vers des agents séparés avec des modèles par défaut différents.
    - **Changement à la demande** : utilisez `/model` pour changer le modèle de la session en cours à tout moment.

    Voir [Tâches cron](/fr/automation/cron-jobs), [Routage multi-agent](/fr/concepts/multi-agent), et [Commandes slash](/fr/tools/slash-commands).

  </Accordion>

  <Accordion title="Le bot se fige pendant un travail lourd. Comment déporter cela ?">
    Utilisez des **sous-agents** pour les tâches longues ou parallèles. Les sous-agents s'exécutent dans leur propre session,
    renvoient un résumé et gardent votre chat principal réactif.

    Demandez à votre bot de « lancer un sous-agent pour cette tâche » ou utilisez `/subagents`.
    Utilisez `/status` dans le chat pour voir ce que la Gateway fait en ce moment (et si elle est occupée).

    Conseil sur les jetons : les tâches longues et les sous-agents consomment tous deux des jetons. Si le coût vous préoccupe, définissez un
    modèle moins cher pour les sous-agents via `agents.defaults.subagents.model`.

    Documentation : [Sous-agents](/fr/tools/subagents), [Tâches en arrière-plan](/fr/automation/tasks).

  </Accordion>

  <Accordion title="Comment fonctionnent les sessions de sous-agent liées à un fil sur Discord ?">
    Utilisez les liaisons de fil. Vous pouvez lier un fil Discord à un sous-agent ou à une cible de session afin que les messages suivants dans ce fil restent sur cette session liée.

    Flux de base :

    - Lancez avec `sessions_spawn` en utilisant `thread: true` (et éventuellement `mode: "session"` pour un suivi persistant).
    - Ou liez manuellement avec `/focus <target>`.
    - Utilisez `/agents` pour inspecter l'état de la liaison.
    - Utilisez `/session idle <duration|off>` et `/session max-age <duration|off>` pour contrôler le détachement automatique.
    - Utilisez `/unfocus` pour détacher le fil.

    Configuration requise :

    - Valeurs globales par défaut : `session.threadBindings.enabled`, `session.threadBindings.idleHours`, `session.threadBindings.maxAgeHours`.
    - Remplacements Discord : `channels.discord.threadBindings.enabled`, `channels.discord.threadBindings.idleHours`, `channels.discord.threadBindings.maxAgeHours`.
    - Liaison automatique au lancement : définissez `channels.discord.threadBindings.spawnSubagentSessions: true`.

    Documentation : [Sous-agents](/fr/tools/subagents), [Discord](/fr/channels/discord), [Référence de configuration](/fr/gateway/configuration-reference), [Commandes slash](/fr/tools/slash-commands).

  </Accordion>

  <Accordion title="Un sous-agent a terminé, mais la mise à jour d'achèvement est arrivée au mauvais endroit ou n'a jamais été publiée. Que dois-je vérifier ?">
    Vérifiez d'abord la route du demandeur résolue :

    - La livraison de sous-agent en mode achèvement privilégie tout fil ou route de conversation liée lorsqu'il en existe une.
    - Si l'origine d'achèvement ne porte qu'un canal, OpenClaw revient à la route stockée de la session demandeur (`lastChannel` / `lastTo` / `lastAccountId`) afin que la livraison directe puisse encore réussir.
    - S'il n'existe ni route liée ni route stockée utilisable, la livraison directe peut échouer et le résultat revient à une livraison de session en file d'attente au lieu d'être publié immédiatement dans le chat.
    - Des cibles invalides ou obsolètes peuvent encore forcer un retour vers la file ou un échec de livraison final.
    - Si la dernière réponse visible de l'assistant enfant est exactement le jeton silencieux `NO_REPLY` / `no_reply`, ou exactement `ANNOUNCE_SKIP`, OpenClaw supprime volontairement l'annonce au lieu de publier d'anciens messages de progression.
    - Si l'enfant a expiré après seulement des appels d'outils, l'annonce peut réduire cela à un court résumé de progression partielle au lieu de rejouer la sortie brute des outils.

    Débogage :

    ```bash
    openclaw tasks show <runId-or-sessionKey>
    ```

    Documentation : [Sous-agents](/fr/tools/subagents), [Tâches en arrière-plan](/fr/automation/tasks), [Outil de session](/fr/concepts/session-tool).

  </Accordion>

  <Accordion title="Cron ou les rappels ne se déclenchent pas. Que dois-je vérifier ?">
    Cron s'exécute dans le processus Gateway. Si la Gateway ne tourne pas en continu,
    les tâches planifiées ne s'exécuteront pas.

    Liste de contrôle :

    - Confirmez que cron est activé (`cron.enabled`) et que `OPENCLAW_SKIP_CRON` n'est pas défini.
    - Vérifiez que la Gateway tourne 24/7 (pas de veille/redémarrages).
    - Vérifiez les paramètres de fuseau horaire pour la tâche (`--tz` versus fuseau de l'hôte).

    Débogage :

    ```bash
    openclaw cron run <jobId>
    openclaw cron runs --id <jobId> --limit 50
    ```

    Documentation : [Tâches cron](/fr/automation/cron-jobs), [Automatisation et tâches](/fr/automation).

  </Accordion>

  <Accordion title="Cron s'est déclenché, mais rien n'a été envoyé au canal. Pourquoi ?">
    Vérifiez d'abord le mode de livraison :

    - `--no-deliver` / `delivery.mode: "none"` signifie qu'aucun message externe n'est attendu.
    - Une cible d'annonce manquante ou invalide (`channel` / `to`) signifie que le runner a ignoré la livraison sortante.
    - Les échecs d'authentification du canal (`unauthorized`, `Forbidden`) signifient que le runner a essayé de livrer mais que les identifiants l'ont bloqué.
    - Un résultat isolé silencieux (`NO_REPLY` / `no_reply` uniquement) est traité comme volontairement non livrable, donc le runner supprime aussi la livraison de secours en file d'attente.

    Pour les tâches cron isolées, le runner possède la livraison finale. L'agent est censé
    renvoyer un résumé en texte brut que le runner enverra. `--no-deliver` conserve
    ce résultat en interne ; cela ne permet pas à l'agent d'envoyer directement via l'outil
    message à la place.

    Débogage :

    ```bash
    openclaw cron runs --id <jobId> --limit 50
    openclaw tasks show <runId-or-sessionKey>
    ```

    Documentation : [Tâches cron](/fr/automation/cron-jobs), [Tâches en arrière-plan](/fr/automation/tasks).

  </Accordion>

  <Accordion title="Pourquoi une exécution cron isolée a-t-elle changé de modèle ou réessayé une fois ?">
    Il s'agit généralement du chemin de changement de modèle en direct, pas d'une double planification.

    Un cron isolé peut persister un transfert de modèle à l'exécution et réessayer lorsque l'exécution active
    lève `LiveSessionModelSwitchError`. La nouvelle tentative conserve le fournisseur/modèle
    basculé, et si le changement a aussi transporté un nouveau remplacement de profil d'authentification, cron
    le persiste aussi avant de réessayer.

    Règles de sélection associées :

    - Le remplacement de modèle du hook Gmail gagne d'abord lorsqu'il s'applique.
    - Ensuite le `model` par tâche.
    - Ensuite tout remplacement de modèle stocké dans la session cron.
    - Ensuite la sélection normale du modèle agent/par défaut.

    La boucle de réessai est bornée. Après la tentative initiale plus 2 réessais de changement,
    cron abandonne au lieu de boucler indéfiniment.

    Débogage :

    ```bash
    openclaw cron runs --id <jobId> --limit 50
    openclaw tasks show <runId-or-sessionKey>
    ```

    Documentation : [Tâches cron](/fr/automation/cron-jobs), [CLI cron](/cli/cron).

  </Accordion>

  <Accordion title="Comment installer des Skills sur Linux ?">
    Utilisez les commandes natives `openclaw skills` ou déposez des Skills dans votre workspace. L'UI Skills macOS n'est pas disponible sur Linux.
    Parcourez les Skills sur [https://clawhub.ai](https://clawhub.ai).

    ```bash
    openclaw skills search "calendar"
    openclaw skills search --limit 20
    openclaw skills install <skill-slug>
    openclaw skills install <skill-slug> --version <version>
    openclaw skills install <skill-slug> --force
    openclaw skills update --all
    openclaw skills list --eligible
    openclaw skills check
    ```

    L'installation native `openclaw skills install` écrit dans le répertoire `skills/`
    du workspace actif. Installez la CLI distincte `clawhub` uniquement si vous voulez publier ou
    synchroniser vos propres Skills. Pour des installations partagées entre agents, placez le skill sous
    `~/.openclaw/skills` et utilisez `agents.defaults.skills` ou
    `agents.list[].skills` si vous voulez limiter quels agents peuvent le voir.

  </Accordion>

  <Accordion title="OpenClaw peut-il exécuter des tâches selon un planning ou en continu en arrière-plan ?">
    Oui. Utilisez le planificateur de la Gateway :

    - **Tâches cron** pour les tâches planifiées ou récurrentes (persistent après les redémarrages).
    - **Heartbeat** pour les vérifications périodiques de la « session principale ».
    - **Tâches isolées** pour des agents autonomes qui publient des résumés ou livrent vers des chats.

    Documentation : [Tâches cron](/fr/automation/cron-jobs), [Automatisation et tâches](/fr/automation),
    [Heartbeat](/fr/gateway/heartbeat).

  </Accordion>

  <Accordion title="Puis-je exécuter des Skills Apple réservés à macOS depuis Linux ?">
    Pas directement. Les Skills macOS sont filtrés par `metadata.openclaw.os` ainsi que par les binaires requis, et les Skills n'apparaissent dans le prompt système que s'ils sont éligibles sur l'**hôte Gateway**. Sous Linux, les Skills réservés à `darwin` (comme `apple-notes`, `apple-reminders`, `things-mac`) ne se chargeront pas sauf si vous remplacez ce filtrage.

    Vous disposez de trois schémas pris en charge :

    **Option A - exécuter la Gateway sur un Mac (le plus simple).**
    Exécutez la Gateway là où les binaires macOS existent, puis connectez-vous depuis Linux en [mode distant](#gateway-ports-already-running-and-remote-mode) ou via Tailscale. Les Skills se chargent normalement parce que l'hôte Gateway est sous macOS.

    **Option B - utiliser un nœud macOS (sans SSH).**
    Exécutez la Gateway sur Linux, appairez un nœud macOS (application de barre de menu), et définissez **Node Run Commands** sur « Always Ask » ou « Always Allow » sur le Mac. OpenClaw peut traiter les Skills réservés à macOS comme éligibles lorsque les binaires requis existent sur le nœud. L'agent exécute ces Skills via l'outil `nodes`. Si vous choisissez « Always Ask », approuver « Always Allow » dans l'invite ajoute cette commande à la liste d'autorisation.

    **Option C - proxifier les binaires macOS via SSH (avancé).**
    Gardez la Gateway sur Linux, mais faites en sorte que les binaires CLI requis soient résolus vers des wrappers SSH exécutés sur un Mac. Ensuite, remplacez le skill pour autoriser Linux afin qu'il reste éligible.

    1. Créez un wrapper SSH pour le binaire (exemple : `memo` pour Apple Notes) :

       ```bash
       #!/usr/bin/env bash
       set -euo pipefail
       exec ssh -T user@mac-host /opt/homebrew/bin/memo "$@"
       ```

    2. Placez le wrapper sur le `PATH` de l'hôte Linux (par exemple `~/bin/memo`).
    3. Remplacez les métadonnées du skill (workspace ou `~/.openclaw/skills`) pour autoriser Linux :

       ```markdown
       ---
       name: apple-notes
       description: Manage Apple Notes via the memo CLI on macOS.
       metadata: { "openclaw": { "os": ["darwin", "linux"], "requires": { "bins": ["memo"] } } }
       ---
       ```

    4. Démarrez une nouvelle session afin de rafraîchir l'instantané des Skills.

  </Accordion>

  <Accordion title="Avez-vous une intégration Notion ou HeyGen ?">
    Pas intégrée aujourd'hui.

    Options :

    - **Skill / plugin personnalisé :** meilleure option pour un accès API fiable (Notion/HeyGen ont tous deux des API).
    - **Automatisation navigateur :** fonctionne sans code mais est plus lente et plus fragile.

    Si vous voulez garder un contexte par client (workflows d'agence), un schéma simple est :

    - Une page Notion par client (contexte + préférences + travail en cours).
    - Demander à l'agent de récupérer cette page au début d'une session.

    Si vous voulez une intégration native, ouvrez une demande de fonctionnalité ou créez un skill
    ciblant ces API.

    Installer des Skills :

    ```bash
    openclaw skills install <skill-slug>
    openclaw skills update --all
    ```

    Les installations natives sont placées dans le répertoire `skills/` du workspace actif. Pour des Skills partagés entre agents, placez-les dans `~/.openclaw/skills/<name>/SKILL.md`. Si seuls certains agents doivent voir une installation partagée, configurez `agents.defaults.skills` ou `agents.list[].skills`. Certains Skills attendent des binaires installés via Homebrew ; sous Linux cela signifie Linuxbrew (voir l'entrée FAQ Homebrew Linux ci-dessus). Voir [Skills](/fr/tools/skills), [Configuration des Skills](/fr/tools/skills-config), et [ClawHub](/fr/tools/clawhub).

  </Accordion>

  <Accordion title="Comment utiliser mon Chrome déjà connecté avec OpenClaw ?">
    Utilisez le profil navigateur intégré `user`, qui se connecte via Chrome DevTools MCP :

    ```bash
    openclaw browser --browser-profile user tabs
    openclaw browser --browser-profile user snapshot
    ```

    Si vous voulez un nom personnalisé, créez un profil MCP explicite :

    ```bash
    openclaw browser create-profile --name chrome-live --driver existing-session
    openclaw browser --browser-profile chrome-live tabs
    ```

    Ce chemin est local à l'hôte. Si la Gateway s'exécute ailleurs, exécutez soit un hôte de nœud sur la machine du navigateur, soit utilisez CDP distant à la place.

    Limites actuelles de `existing-session` / `user` :

    - les actions sont basées sur des refs, pas sur des sélecteurs CSS
    - les uploads exigent `ref` / `inputRef` et prennent actuellement en charge un seul fichier à la fois
    - `responsebody`, l'export PDF, l'interception de téléchargement et les actions par lot exigent encore un navigateur géré ou un profil CDP brut

  </Accordion>
</AccordionGroup>

## Sandboxing et mémoire

<AccordionGroup>
  <Accordion title="Existe-t-il une documentation dédiée au sandboxing ?">
    Oui. Voir [Sandboxing](/fr/gateway/sandboxing). Pour la configuration spécifique à Docker (Gateway complète dans Docker ou images de sandbox), voir [Docker](/fr/install/docker).
  </Accordion>

  <Accordion title="Docker semble limité - comment activer toutes les fonctionnalités ?">
    L'image par défaut privilégie la sécurité et s'exécute comme utilisateur `node`, elle
    n'inclut donc pas de paquets système, Homebrew ou navigateurs groupés. Pour une configuration plus complète :

    - Persistez `/home/node` avec `OPENCLAW_HOME_VOLUME` afin que les caches survivent.
    - Intégrez les dépendances système dans l'image avec `OPENCLAW_DOCKER_APT_PACKAGES`.
    - Installez les navigateurs Playwright via la CLI intégrée :
      `node /app/node_modules/playwright-core/cli.js install chromium`
    - Définissez `PLAYWRIGHT_BROWSERS_PATH` et assurez-vous que ce chemin est persistant.

    Documentation : [Docker](/fr/install/docker), [Browser](/fr/tools/browser).

  </Accordion>

  <Accordion title="Puis-je garder les DM personnels mais rendre les groupes publics/en sandbox avec un seul agent ?">
    Oui - si votre trafic privé correspond aux **DM** et votre trafic public aux **groupes**.

    Utilisez `agents.defaults.sandbox.mode: "non-main"` afin que les sessions de groupe/canal (clés non principales) s'exécutent dans Docker, tandis que la session DM principale reste sur l'hôte. Limitez ensuite les outils disponibles dans les sessions sandboxées via `tools.sandbox.tools`.

    Guide de configuration + exemple : [Groupes : DM personnels + groupes publics](/fr/channels/groups#pattern-personal-dms-public-groups-single-agent)

    Référence de configuration clé : [Configuration Gateway](/fr/gateway/configuration-reference#agentsdefaultssandbox)

  </Accordion>

  <Accordion title="Comment lier un dossier hôte dans le sandbox ?">
    Définissez `agents.defaults.sandbox.docker.binds` à `["host:path:mode"]` (par ex. `"/home/user/src:/src:ro"`). Les liaisons globales + par agent sont fusionnées ; les liaisons par agent sont ignorées lorsque `scope: "shared"`. Utilisez `:ro` pour tout ce qui est sensible et rappelez-vous que les liaisons contournent les barrières du système de fichiers du sandbox.

    OpenClaw valide les sources de liaison par rapport au chemin normalisé et au chemin canonique résolu via l'ancêtre existant le plus profond. Cela signifie que les échappements par parent symlink restent bloqués même lorsque le dernier segment du chemin n'existe pas encore, et que les vérifications de racine autorisée s'appliquent toujours après résolution du symlink.

    Voir [Sandboxing](/fr/gateway/sandboxing#custom-bind-mounts) et [Sandbox vs Tool Policy vs Elevated](/fr/gateway/sandbox-vs-tool-policy-vs-elevated#bind-mounts-security-quick-check) pour des exemples et des notes de sécurité.

  </Accordion>

  <Accordion title="Comment fonctionne la mémoire ?">
    La mémoire OpenClaw n'est que des fichiers Markdown dans le workspace de l'agent :

    - Notes quotidiennes dans `memory/YYYY-MM-DD.md`
    - Notes de long terme organisées dans `MEMORY.md` (sessions principales/privées uniquement)

    OpenClaw exécute aussi un **flush mémoire silencieux avant compaction** pour rappeler au modèle
    d'écrire des notes durables avant l'auto-compaction. Cela ne s'exécute que lorsque le workspace
    est inscriptible (les sandboxes en lecture seule l'ignorent). Voir [Mémoire](/fr/concepts/memory).

  </Accordion>

  <Accordion title="La mémoire oublie toujours des choses. Comment faire pour que cela tienne ?">
    Demandez au bot **d'écrire le fait en mémoire**. Les notes à long terme vont dans `MEMORY.md`,
    le contexte à court terme va dans `memory/YYYY-MM-DD.md`.

    C'est encore un domaine que nous améliorons. Il est utile de rappeler au modèle de stocker les souvenirs ;
    il saura quoi faire. S'il oublie toujours, vérifiez que la Gateway utilise le même
    workspace à chaque exécution.

    Documentation : [Mémoire](/fr/concepts/memory), [Workspace de l'agent](/fr/concepts/agent-workspace).

  </Accordion>

  <Accordion title="La mémoire persiste-t-elle pour toujours ? Quelles sont les limites ?">
    Les fichiers mémoire vivent sur disque et persistent jusqu'à leur suppression. La limite est votre
    stockage, pas le modèle. Le **contexte de session** reste toutefois limité par la fenêtre de contexte
    du modèle, donc les longues conversations peuvent être compactées ou tronquées. C'est pourquoi
    la recherche mémoire existe - elle ne réinjecte que les parties pertinentes dans le contexte.

    Documentation : [Mémoire](/fr/concepts/memory), [Contexte](/fr/concepts/context).

  </Accordion>

  <Accordion title="La recherche de mémoire sémantique nécessite-t-elle une clé API OpenAI ?">
    Seulement si vous utilisez des **embeddings OpenAI**. Codex OAuth couvre le chat/les complétions et
    ne donne **pas** accès aux embeddings, donc **se connecter avec Codex (OAuth ou
    connexion CLI Codex)** n'aide pas pour la recherche de mémoire sémantique. Les embeddings OpenAI
    nécessitent toujours une vraie clé API (`OPENAI_API_KEY` ou `models.providers.openai.apiKey`).

    Si vous ne définissez pas explicitement de fournisseur, OpenClaw en sélectionne automatiquement un lorsqu'il
    peut résoudre une clé API (profils d'authentification, `models.providers.*.apiKey`, ou variables d'environnement).
    Il privilégie OpenAI si une clé OpenAI est résolue, sinon Gemini si une clé Gemini est
    résolue, puis Voyage, puis Mistral. Si aucune clé distante n'est disponible, la recherche mémoire
    reste désactivée jusqu'à votre configuration. Si vous avez un chemin de modèle local
    configuré et présent, OpenClaw
    privilégie `local`. Ollama est pris en charge lorsque vous définissez explicitement
    `memorySearch.provider = "ollama"`.

    Si vous préférez rester en local, définissez `memorySearch.provider = "local"` (et éventuellement
    `memorySearch.fallback = "none"`). Si vous voulez les embeddings Gemini, définissez
    `memorySearch.provider = "gemini"` et fournissez `GEMINI_API_KEY` (ou
    `memorySearch.remote.apiKey`). Nous prenons en charge les modèles d'embeddings **OpenAI, Gemini, Voyage, Mistral, Ollama ou locaux**
    - voir [Mémoire](/fr/concepts/memory) pour les détails de configuration.

  </Accordion>
</AccordionGroup>

## Emplacement des fichiers sur disque

<AccordionGroup>
  <Accordion title="Toutes les données utilisées avec OpenClaw sont-elles enregistrées localement ?">
    Non - **l'état d'OpenClaw est local**, mais **les services externes voient quand même ce que vous leur envoyez**.

    - **Local par défaut :** sessions, fichiers mémoire, configuration et workspace vivent sur l'hôte Gateway
      (`~/.openclaw` + votre répertoire de workspace).
    - **Distant par nécessité :** les messages que vous envoyez aux fournisseurs de modèles (Anthropic/OpenAI/etc.) vont à
      leurs API, et les plateformes de chat (WhatsApp/Telegram/Slack/etc.) stockent les données de message sur leurs
      serveurs.
    - **Vous contrôlez l'empreinte :** utiliser des modèles locaux garde les prompts sur votre machine, mais le
      trafic des canaux passe quand même par les serveurs du canal.

    Voir aussi : [Workspace de l'agent](/fr/concepts/agent-workspace), [Mémoire](/fr/concepts/memory).

  </Accordion>

  <Accordion title="Où OpenClaw stocke-t-il ses données ?">
    Tout vit sous `$OPENCLAW_STATE_DIR` (par défaut : `~/.openclaw`) :

    | Path                                                            | Purpose                                                            |
    | --------------------------------------------------------------- | ------------------------------------------------------------------ |
    | `$OPENCLAW_STATE_DIR/openclaw.json`                             | Configuration principale (JSON5)                                   |
    | `$OPENCLAW_STATE_DIR/credentials/oauth.json`                    | Import OAuth hérité (copié dans les profils d'authentification au premier usage) |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/auth-profiles.json` | Profils d'authentification (OAuth, clés API et `keyRef`/`tokenRef` facultatifs) |
    | `$OPENCLAW_STATE_DIR/secrets.json`                              | Charge utile de secret facultative adossée à un fichier pour les fournisseurs SecretRef `file` |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/auth.json`          | Fichier de compatibilité hérité (entrées statiques `api_key` nettoyées) |
    | `$OPENCLAW_STATE_DIR/credentials/`                              | État des fournisseurs (par ex. `whatsapp/<accountId>/creds.json`)  |
    | `$OPENCLAW_STATE_DIR/agents/`                                   | État par agent (agentDir + sessions)                               |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/sessions/`                | Historique et état des conversations (par agent)                   |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/sessions/sessions.json`   | Métadonnées de session (par agent)                                 |

    Chemin hérité mono-agent : `~/.openclaw/agent/*` (migré par `openclaw doctor`).

    Votre **workspace** (`AGENTS.md`, fichiers mémoire, Skills, etc.) est séparé et configuré via `agents.defaults.workspace` (par défaut : `~/.openclaw/workspace`).

  </Accordion>

  <Accordion title="Où doivent se trouver AGENTS.md / SOUL.md / USER.md / MEMORY.md ?">
    Ces fichiers vivent dans le **workspace de l'agent**, pas dans `~/.openclaw`.

    - **Workspace (par agent)** : `AGENTS.md`, `SOUL.md`, `IDENTITY.md`, `USER.md`,
      `MEMORY.md` (ou le secours hérité `memory.md` quand `MEMORY.md` est absent),
      `memory/YYYY-MM-DD.md`, `HEARTBEAT.md` facultatif.
    - **Répertoire d'état (`~/.openclaw`)** : configuration, état canal/fournisseur, profils d'authentification, sessions, journaux,
      et Skills partagés (`~/.openclaw/skills`).

    Le workspace par défaut est `~/.openclaw/workspace`, configurable via :

    ```json5
    {
      agents: { defaults: { workspace: "~/.openclaw/workspace" } },
    }
    ```

    Si le bot « oublie » après un redémarrage, confirmez que la Gateway utilise le même
    workspace à chaque lancement (et rappelez-vous : le mode distant utilise le workspace de l'**hôte de la passerelle**,
    pas celui de votre ordinateur local).

    Conseil : si vous voulez un comportement ou une préférence durable, demandez au bot de **l'écrire dans
    AGENTS.md ou MEMORY.md** plutôt que de vous appuyer sur l'historique de chat.

    Voir [Workspace de l'agent](/fr/concepts/agent-workspace) et [Mémoire](/fr/concepts/memory).

  </Accordion>

  <Accordion title="Stratégie de sauvegarde recommandée">
    Placez votre **workspace d'agent** dans un dépôt git **privé** et sauvegardez-le
    quelque part de privé (par exemple GitHub privé). Cela capture la mémoire + les fichiers
    AGENTS/SOUL/USER, et vous permet de restaurer plus tard « l'esprit » de l'assistant.

    Ne committez **pas** quoi que ce soit sous `~/.openclaw` (identifiants, sessions, jetons ou charges utiles de secrets chiffrées).
    Si vous avez besoin d'une restauration complète, sauvegardez séparément le workspace et le répertoire d'état
    (voir la question sur la migration ci-dessus).

    Documentation : [Workspace de l'agent](/fr/concepts/agent-workspace).

  </Accordion>

  <Accordion title="Comment désinstaller complètement OpenClaw ?">
    Consultez le guide dédié : [Désinstallation](/fr/install/uninstall).
  </Accordion>

  <Accordion title="Les agents peuvent-ils travailler en dehors du workspace ?">
    Oui. Le workspace est le **cwd par défaut** et l'ancre mémoire, pas un sandbox strict.
    Les chemins relatifs sont résolus dans le workspace, mais les chemins absolus peuvent accéder à d'autres
    emplacements de l'hôte sauf si le sandboxing est activé. Si vous avez besoin d'isolation, utilisez
    [`agents.defaults.sandbox`](/fr/gateway/sandboxing) ou des paramètres de sandbox par agent. Si vous
    voulez qu'un dépôt soit le répertoire de travail par défaut, faites pointer le
    `workspace` de cet agent vers la racine du dépôt. Le dépôt OpenClaw n'est que du code source ; gardez le
    workspace séparé sauf si vous voulez délibérément que l'agent travaille dedans.

    Exemple (dépôt comme cwd par défaut) :

    ```json5
    {
      agents: {
        defaults: {
          workspace: "~/Projects/my-repo",
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Mode distant : où se trouve le stockage des sessions ?">
    L'état des sessions appartient à l'**hôte de la passerelle**. Si vous êtes en mode distant, le stockage des sessions qui vous intéresse est sur la machine distante, pas sur votre ordinateur local. Voir [Gestion des sessions](/fr/concepts/session).
  </Accordion>
</AccordionGroup>

## Bases de la configuration

<AccordionGroup>
  <Accordion title="Quel est le format de la configuration ? Où se trouve-t-elle ?">
    OpenClaw lit une configuration **JSON5** facultative depuis `$OPENCLAW_CONFIG_PATH` (par défaut : `~/.openclaw/openclaw.json`) :

    ```
    $OPENCLAW_CONFIG_PATH
    ```

    Si le fichier est manquant, il utilise des valeurs par défaut relativement sûres (y compris un workspace par défaut à `~/.openclaw/workspace`).

  </Accordion>

  <Accordion title='J’ai défini gateway.bind: "lan" (ou "tailnet") et maintenant rien n’écoute / l’UI indique unauthorized'>
    Les liaisons non loopback **exigent un chemin d'authentification valide de la passerelle**. En pratique, cela signifie :

    - authentification par secret partagé : jeton ou mot de passe
    - `gateway.auth.mode: "trusted-proxy"` derrière un proxy inverse avec gestion d'identité non loopback correctement configuré

    ```json5
    {
      gateway: {
        bind: "lan",
        auth: {
          mode: "token",
          token: "replace-me",
        },
      },
    }
    ```

    Remarques :

    - `gateway.remote.token` / `.password` n'activent pas à eux seuls l'authentification locale de la passerelle.
    - Les chemins d'appel locaux peuvent utiliser `gateway.remote.*` comme secours seulement lorsque `gateway.auth.*` n'est pas défini.
    - Pour l'authentification par mot de passe, définissez `gateway.auth.mode: "password"` ainsi que `gateway.auth.password` (ou `OPENCLAW_GATEWAY_PASSWORD`) à la place.
    - Si `gateway.auth.token` / `gateway.auth.password` est explicitement configuré via SecretRef et non résolu, la résolution échoue de manière fermée (pas de secours distant masquant le problème).
    - Les configurations Control UI à secret partagé s'authentifient via `connect.params.auth.token` ou `connect.params.auth.password` (stockés dans les paramètres de l'application/UI). Les modes à identité portée comme Tailscale Serve ou `trusted-proxy` utilisent à la place des en-têtes de requête. Évitez de placer des secrets partagés dans les URL.
    - Avec `gateway.auth.mode: "trusted-proxy"`, les proxies inverses loopback sur la même machine ne satisfont toujours **pas** l'authentification trusted-proxy. Le trusted proxy doit être une source non loopback configurée.

  </Accordion>

  <Accordion title="Pourquoi ai-je besoin d'un jeton sur localhost maintenant ?">
    OpenClaw applique par défaut l'authentification de la passerelle, y compris sur loopback. Dans le chemin par défaut normal, cela signifie l'authentification par jeton : si aucun chemin d'authentification explicite n'est configuré, le démarrage de la passerelle se résout en mode jeton et en génère automatiquement un, enregistré dans `gateway.auth.token`, donc **les clients WS locaux doivent s'authentifier**. Cela empêche d'autres processus locaux d'appeler la Gateway.

    Si vous préférez un autre chemin d'authentification, vous pouvez choisir explicitement le mode mot de passe (ou, pour des proxies inverses avec gestion d'identité non loopback, `trusted-proxy`). Si vous voulez **vraiment** un loopback ouvert, définissez explicitement `gateway.auth.mode: "none"` dans votre configuration. Doctor peut générer un jeton pour vous à tout moment : `openclaw doctor --generate-gateway-token`.

  </Accordion>

  <Accordion title="Dois-je redémarrer après avoir changé la configuration ?">
    La Gateway surveille la configuration et prend en charge le hot-reload :

    - `gateway.reload.mode: "hybrid"` (par défaut) : applique à chaud les changements sûrs, redémarre pour les changements critiques
    - `hot`, `restart`, `off` sont aussi pris en charge

  </Accordion>

  <Accordion title="Comment désactiver les slogans amusants de la CLI ?">
    Définissez `cli.banner.taglineMode` dans la configuration :

    ```json5
    {
      cli: {
        banner: {
          taglineMode: "off", // random | default | off
        },
      },
    }
    ```

    - `off` : masque le slogan mais conserve le titre de la bannière/la ligne de version.
    - `default` : utilise `All your chats, one OpenClaw.` à chaque fois.
    - `random` : slogans tournants amusants/de saison (comportement par défaut).
    - Si vous ne voulez aucune bannière, définissez la variable d'environnement `OPENCLAW_HIDE_BANNER=1`.

  </Accordion>

  <Accordion title="Comment activer la recherche web (et web fetch) ?">
    `web_fetch` fonctionne sans clé API. `web_search` dépend du
    fournisseur sélectionné :

    - Les fournisseurs adossés à une API comme Brave, Exa, Firecrawl, Gemini, Grok, Kimi, MiniMax Search, Perplexity et Tavily exigent leur configuration normale de clé API.
    - Ollama Web Search ne nécessite pas de clé, mais utilise votre hôte Ollama configuré et exige `ollama signin`.
    - DuckDuckGo ne nécessite pas de clé, mais c'est une intégration non officielle basée sur HTML.
    - SearXNG ne nécessite pas de clé et peut être auto-hébergé ; configurez `SEARXNG_BASE_URL` ou `plugins.entries.searxng.config.webSearch.baseUrl`.

    **Recommandé :** exécutez `openclaw configure --section web` et choisissez un fournisseur.
    Alternatives via variables d'environnement :

    - Brave : `BRAVE_API_KEY`
    - Exa : `EXA_API_KEY`
    - Firecrawl : `FIRECRAWL_API_KEY`
    - Gemini : `GEMINI_API_KEY`
    - Grok : `XAI_API_KEY`
    - Kimi : `KIMI_API_KEY` ou `MOONSHOT_API_KEY`
    - MiniMax Search : `MINIMAX_CODE_PLAN_KEY`, `MINIMAX_CODING_API_KEY`, ou `MINIMAX_API_KEY`
    - Perplexity : `PERPLEXITY_API_KEY` ou `OPENROUTER_API_KEY`
    - SearXNG : `SEARXNG_BASE_URL`
    - Tavily : `TAVILY_API_KEY`

    ```json5
    {
      plugins: {
        entries: {
          brave: {
            config: {
              webSearch: {
                apiKey: "BRAVE_API_KEY_HERE",
              },
            },
          },
        },
        },
        tools: {
          web: {
            search: {
              enabled: true,
              provider: "brave",
              maxResults: 5,
            },
            fetch: {
              enabled: true,
              provider: "firecrawl", // optional; omit for auto-detect
            },
          },
        },
    }
    ```

    La configuration spécifique au fournisseur pour la recherche web vit désormais sous `plugins.entries.<plugin>.config.webSearch.*`.
    Les anciens chemins fournisseur `tools.web.search.*` continuent de se charger temporairement pour compatibilité, mais ils ne doivent pas être utilisés pour les nouvelles configurations.
    La configuration de secours Firecrawl pour web fetch vit sous `plugins.entries.firecrawl.config.webFetch.*`.

    Remarques :

    - Si vous utilisez des listes d'autorisation, ajoutez `web_search`/`web_fetch`/`x_search` ou `group:web`.
    - `web_fetch` est activé par défaut (sauf désactivation explicite).
    - Si `tools.web.fetch.provider` est omis, OpenClaw détecte automatiquement le premier fournisseur de secours prêt pour fetch à partir des identifiants disponibles. Aujourd'hui, le fournisseur groupé est Firecrawl.
    - Les démons lisent les variables d'environnement depuis `~/.openclaw/.env` (ou l'environnement du service).

    Documentation : [Outils Web](/fr/tools/web).

  </Accordion>

  <Accordion title="config.apply a effacé ma configuration. Comment récupérer et éviter cela ?">
    `config.apply` remplace **toute la configuration**. Si vous envoyez un objet partiel, tout le
    reste est supprimé.

    Récupération :

    - Restaurez à partir d'une sauvegarde (git ou une copie de `~/.openclaw/openclaw.json`).
    - Si vous n'avez pas de sauvegarde, relancez `openclaw doctor` puis reconfigurez les canaux/modèles.
    - Si c'était inattendu, ouvrez un bug et incluez votre dernière configuration connue ou toute sauvegarde.
    - Un agent de code local peut souvent reconstruire une configuration fonctionnelle à partir des journaux ou de l'historique.

    Pour éviter cela :

    - Utilisez `openclaw config set` pour les petits changements.
    - Utilisez `openclaw configure` pour les modifications interactives.
    - Utilisez d'abord `config.schema.lookup` lorsque vous n'êtes pas sûr du chemin exact ou de la forme d'un champ ; il renvoie un nœud de schéma superficiel ainsi que les résumés des enfants immédiats pour faciliter l'exploration.
    - Utilisez `config.patch` pour les modifications RPC partielles ; gardez `config.apply` uniquement pour le remplacement complet de configuration.
    - Si vous utilisez l'outil `gateway` réservé au propriétaire depuis une exécution d'agent, il rejettera toujours les écritures vers `tools.exec.ask` / `tools.exec.security` (y compris les alias hérités `tools.bash.*` qui se normalisent vers les mêmes chemins exec protégés).

    Documentation : [Config](/cli/config), [Configure](/cli/configure), [Doctor](/fr/gateway/doctor).

  </Accordion>

  <Accordion title="Comment exécuter une Gateway centrale avec des workers spécialisés sur plusieurs appareils ?">
    Le schéma le plus courant est **une Gateway** (par ex. Raspberry Pi) plus des **nodes** et des **agents** :

    - **Gateway (centrale) :** possède les canaux (Signal/WhatsApp), le routage et les sessions.
    - **Nodes (appareils) :** les Mac/iOS/Android se connectent comme périphériques et exposent des outils locaux (`system.run`, `canvas`, `camera`).
    - **Agents (workers) :** cerveaux/workspaces distincts pour des rôles spécialisés (par ex. « Hetzner ops », « Personal data »).
    - **Sous-agents :** lancent du travail en arrière-plan depuis un agent principal lorsque vous voulez du parallélisme.
    - **TUI :** se connecte à la Gateway et permet de changer d'agent/session.

    Documentation : [Nodes](/fr/nodes), [Accès distant](/fr/gateway/remote), [Routage multi-agent](/fr/concepts/multi-agent), [Sous-agents](/fr/tools/subagents), [TUI](/web/tui).

  </Accordion>

  <Accordion title="Le navigateur OpenClaw peut-il fonctionner en mode headless ?">
    Oui. C'est une option de configuration :

    ```json5
    {
      browser: { headless: true },
      agents: {
        defaults: {
          sandbox: { browser: { headless: true } },
        },
      },
    }
    ```

    La valeur par défaut est `false` (avec interface). Le mode headless a plus de chances de déclencher des contrôles anti-bot sur certains sites. Voir [Browser](/fr/tools/browser).

    Le mode headless utilise le **même moteur Chromium** et fonctionne pour la plupart des automatisations (formulaires, clics, scraping, connexions). Les principales différences :

    - Pas de fenêtre navigateur visible (utilisez des captures d'écran si vous avez besoin de visuels).
    - Certains sites sont plus stricts avec l'automatisation en mode headless (CAPTCHA, anti-bot).
      Par exemple, X/Twitter bloque souvent les sessions headless.

  </Accordion>

  <Accordion title="Comment utiliser Brave pour le contrôle du navigateur ?">
    Définissez `browser.executablePath` vers votre binaire Brave (ou tout navigateur basé sur Chromium) et redémarrez la Gateway.
    Voir les exemples complets de configuration dans [Browser](/fr/tools/browser#use-brave-or-another-chromium-based-browser).
  </Accordion>
</AccordionGroup>

## Passerelles distantes et nodes

<AccordionGroup>
  <Accordion title="Comment les commandes se propagent-elles entre Telegram, la passerelle et les nodes ?">
    Les messages Telegram sont gérés par la **passerelle**. La passerelle exécute l'agent et
    appelle ensuite les nodes via le **WebSocket Gateway** lorsqu'un outil node est nécessaire :

    Telegram → Gateway → Agent → `node.*` → Node → Gateway → Telegram

    Les nodes ne voient pas le trafic fournisseur entrant ; ils ne reçoivent que des appels RPC node.

  </Accordion>

  <Accordion title="Comment mon agent peut-il accéder à mon ordinateur si la Gateway est hébergée à distance ?">
    Réponse courte : **appairez votre ordinateur comme node**. La Gateway s'exécute ailleurs, mais elle peut
    appeler les outils `node.*` (écran, caméra, système) sur votre machine locale via le WebSocket Gateway.

    Configuration typique :

    1. Exécutez la Gateway sur l'hôte toujours actif (VPS/serveur domestique).
    2. Placez l'hôte Gateway + votre ordinateur sur le même tailnet.
    3. Assurez-vous que le WS Gateway est accessible (liaison tailnet ou tunnel SSH).
    4. Ouvrez l'application macOS localement et connectez-vous en mode **Remote over SSH** (ou tailnet direct)
       afin qu'elle puisse s'enregistrer comme node.
    5. Approuvez le node sur la Gateway :

       ```bash
       openclaw devices list
       openclaw devices approve <requestId>
       ```

    Aucun pont TCP séparé n'est nécessaire ; les nodes se connectent via le WebSocket Gateway.

    Rappel de sécurité : appairer un nœud macOS autorise `system.run` sur cette machine. N'appairez
    que des appareils de confiance, et consultez [Sécurité](/fr/gateway/security).

    Documentation : [Nodes](/fr/nodes), [Protocole Gateway](/fr/gateway/protocol), [mode distant macOS](/fr/platforms/mac/remote), [Sécurité](/fr/gateway/security).

  </Accordion>

  <Accordion title="Tailscale est connecté mais je n'obtiens aucune réponse. Que faire ?">
    Vérifiez l'essentiel :

    - La Gateway est en cours d'exécution : `openclaw gateway status`
    - État de la Gateway : `openclaw status`
    - État du canal : `openclaw channels status`

    Vérifiez ensuite l'authentification et le routage :

    - Si vous utilisez Tailscale Serve, assurez-vous que `gateway.auth.allowTailscale` est correctement défini.
    - Si vous vous connectez via tunnel SSH, confirmez que le tunnel local est actif et pointe vers le bon port.
    - Confirmez que vos listes d'autorisation (DM ou groupe) incluent votre compte.

    Documentation : [Tailscale](/fr/gateway/tailscale), [Accès distant](/fr/gateway/remote), [Canaux](/fr/channels).

  </Accordion>

  <Accordion title="Deux instances OpenClaw peuvent-elles se parler (local + VPS) ?">
    Oui. Il n'existe pas de pont « bot-à-bot » intégré, mais vous pouvez le mettre en place de plusieurs
    façons fiables :

    **Le plus simple :** utilisez un canal de chat normal accessible aux deux bots (Telegram/Slack/WhatsApp).
    Demandez au Bot A d'envoyer un message au Bot B, puis laissez le Bot B répondre normalement.

    **Pont CLI (générique) :** exécutez un script qui appelle l'autre Gateway avec
    `openclaw agent --message ... --deliver`, en ciblant un chat où l'autre bot
    écoute. Si un bot est sur un VPS distant, pointez votre CLI vers cette Gateway distante
    via SSH/Tailscale (voir [Accès distant](/fr/gateway/remote)).

    Schéma d'exemple (à exécuter depuis une machine pouvant atteindre la Gateway cible) :

    ```bash
    openclaw agent --message "Hello from local bot" --deliver --channel telegram --reply-to <chat-id>
    ```

    Conseil : ajoutez une garde-fou pour que les deux bots ne bouclent pas indéfiniment (mention uniquement, listes d'autorisation de canaux,
    ou règle « ne pas répondre aux messages de bot »).

    Documentation : [Accès distant](/fr/gateway/remote), [CLI Agent](/cli/agent), [Envoi Agent](/fr/tools/agent-send).

  </Accordion>

  <Accordion title="Ai-je besoin de VPS séparés pour plusieurs agents ?">
    Non. Une Gateway peut héberger plusieurs agents, chacun avec son propre workspace, ses modèles par défaut
    et son routage. C'est la configuration normale et elle est bien moins coûteuse et plus simple que
    d'exécuter un VPS par agent.

    Utilisez des VPS séparés uniquement lorsque vous avez besoin d'une isolation stricte (frontières de sécurité) ou de
    configurations très différentes que vous ne voulez pas partager. Sinon, gardez une seule Gateway et
    utilisez plusieurs agents ou sous-agents.

  </Accordion>

  <Accordion title="Y a-t-il un avantage à utiliser un node sur mon ordinateur portable personnel plutôt que SSH depuis un VPS ?">
    Oui - les nodes sont la méthode de première classe pour atteindre votre ordinateur portable depuis une Gateway distante, et ils
    offrent plus qu'un simple accès shell. La Gateway fonctionne sur macOS/Linux (Windows via WSL2) et est
    légère (un petit VPS ou un boîtier de classe Raspberry Pi convient ; 4 Go de RAM suffisent largement), donc une
    configuration courante est un hôte toujours actif plus votre ordinateur portable comme node.

    - **Aucun SSH entrant requis.** Les nodes se connectent au WebSocket Gateway et utilisent l'appairage d'appareil.
    - **Contrôles d'exécution plus sûrs.** `system.run` est protégé par des listes d'autorisation/approbations node sur cet ordinateur portable.
    - **Davantage d'outils d'appareil.** Les nodes exposent `canvas`, `camera` et `screen` en plus de `system.run`.
    - **Automatisation navigateur locale.** Gardez la Gateway sur un VPS, mais exécutez Chrome localement via un hôte node sur l'ordinateur portable, ou attachez-vous à Chrome local sur l'hôte via Chrome MCP.

    SSH est correct pour un accès shell ponctuel, mais les nodes sont plus simples pour des workflows d'agents continus et
    l'automatisation d'appareil.

    Documentation : [Nodes](/fr/nodes), [CLI Nodes](/cli/nodes), [Browser](/fr/tools/browser).

  </Accordion>

  <Accordion title="Les nodes exécutent-ils un service Gateway ?">
    Non. Une seule **gateway** doit fonctionner par hôte sauf si vous exécutez intentionnellement des profils isolés (voir [Plusieurs Gateways](/fr/gateway/multiple-gateways)). Les nodes sont des périphériques qui se connectent
    à la gateway (nodes iOS/Android, ou « mode node » macOS dans l'application de barre de menu). Pour les hôtes node headless
    et le contrôle CLI, voir [CLI Node host](/cli/node).

    Un redémarrage complet est requis pour les changements `gateway`, `discovery` et `canvasHost`.

  </Accordion>

  <Accordion title="Existe-t-il un moyen API / RPC d'appliquer une configuration ?">
    Oui.

    - `config.schema.lookup` : inspecter un sous-arbre de configuration avec son nœud de schéma superficiel, son indice UI correspondant et les résumés des enfants immédiats avant écriture
    - `config.get` : récupérer l'instantané actuel + le hash
    - `config.patch` : mise à jour partielle sûre (préférée pour la plupart des modifications RPC)
    - `config.apply` : valider + remplacer toute la configuration, puis redémarrer
    - L'outil d'exécution `gateway` réservé au propriétaire refuse toujours de réécrire `tools.exec.ask` / `tools.exec.security` ; les anciens alias `tools.bash.*` se normalisent vers les mêmes chemins exec protégés

  </Accordion>

  <Accordion title="Configuration minimale raisonnable pour une première installation">
    ```json5
    {
      agents: { defaults: { workspace: "~/.openclaw/workspace" } },
      channels: { whatsapp: { allowFrom: ["+15555550123"] } },
    }
    ```

    Cela définit votre workspace et limite qui peut déclencher le bot.

  </Accordion>

  <Accordion title="Comment configurer Tailscale sur un VPS et se connecter depuis mon Mac ?">
    Étapes minimales :

    1. **Installer + se connecter sur le VPS**

       ```bash
       curl -fsSL https://tailscale.com/install.sh | sh
       sudo tailscale up
       ```

    2. **Installer + se connecter sur votre Mac**
       - Utilisez l'application Tailscale et connectez-vous au même tailnet.
    3. **Activer MagicDNS (recommandé)**
       - Dans la console d'administration Tailscale, activez MagicDNS afin que le VPS ait un nom stable.
    4. **Utiliser le nom d'hôte tailnet**
       - SSH : `ssh user@your-vps.tailnet-xxxx.ts.net`
       - Gateway WS : `ws://your-vps.tailnet-xxxx.ts.net:18789`

    Si vous voulez Control UI sans SSH, utilisez Tailscale Serve sur le VPS :

    ```bash
    openclaw gateway --tailscale serve
    ```

    Cela garde la gateway liée à loopback et expose HTTPS via Tailscale. Voir [Tailscale](/fr/gateway/tailscale).

  </Accordion>

  <Accordion title="Comment connecter un node Mac à une Gateway distante (Tailscale Serve) ?">
    Serve expose la **Control UI + le WS Gateway**. Les nodes se connectent via le même endpoint WS Gateway.

    Configuration recommandée :

    1. **Assurez-vous que le VPS + le Mac sont sur le même tailnet**.
    2. **Utilisez l'application macOS en mode distant** (la cible SSH peut être le nom d'hôte tailnet).
       L'application tunnelera le port Gateway et se connectera comme node.
    3. **Approuvez le node** sur la gateway :

       ```bash
       openclaw devices list
       openclaw devices approve <requestId>
       ```

    Documentation : [Protocole Gateway](/fr/gateway/protocol), [Discovery](/fr/gateway/discovery), [mode distant macOS](/fr/platforms/mac/remote).

  </Accordion>

  <Accordion title="Dois-je installer sur un deuxième ordinateur portable ou simplement ajouter un node ?">
    Si vous n'avez besoin que d'**outils locaux** (écran/caméra/exec) sur le deuxième ordinateur, ajoutez-le comme
    **node**. Cela conserve une seule Gateway et évite les configurations dupliquées. Les outils node locaux sont
    actuellement réservés à macOS, mais nous prévoyons de les étendre à d'autres OS.

    Installez une seconde Gateway uniquement lorsque vous avez besoin d'une **isolation stricte** ou de deux bots complètement séparés.

    Documentation : [Nodes](/fr/nodes), [CLI Nodes](/cli/nodes), [Plusieurs Gateways](/fr/gateway/multiple-gateways).

  </Accordion>
</AccordionGroup>

## Variables d'environnement et chargement de .env

<AccordionGroup>
  <Accordion title="Comment OpenClaw charge-t-il les variables d'environnement ?">
    OpenClaw lit les variables d'environnement depuis le processus parent (shell, launchd/systemd, CI, etc.) et charge en plus :

    - `.env` depuis le répertoire de travail courant
    - un `.env` global de secours depuis `~/.openclaw/.env` (alias `$OPENCLAW_STATE_DIR/.env`)

    Aucun des fichiers `.env` n'écrase les variables d'environnement existantes.

    Vous pouvez aussi définir des variables d'environnement inline dans la configuration (appliquées uniquement si elles manquent dans l'environnement du processus) :

    ```json5
    {
      env: {
        OPENROUTER_API_KEY: "sk-or-...",
        vars: { GROQ_API_KEY: "gsk-..." },
      },
    }
    ```

    Voir [/environment](/fr/help/environment) pour la priorité complète et les sources.

  </Accordion>

  <Accordion title="J'ai démarré la Gateway via le service et mes variables d'environnement ont disparu. Que faire ?">
    Deux corrections courantes :

    1. Placez les clés manquantes dans `~/.openclaw/.env` pour qu'elles soient prises en compte même lorsque le service n'hérite pas de l'environnement de votre shell.
    2. Activez l'import du shell (confort facultatif) :

    ```json5
    {
      env: {
        shellEnv: {
          enabled: true,
          timeoutMs: 15000,
        },
      },
    }
    ```

    Cela exécute votre shell de connexion et n'importe que les clés attendues manquantes (sans jamais écraser). Équivalents variables d'environnement :
    `OPENCLAW_LOAD_SHELL_ENV=1`, `OPENCLAW_SHELL_ENV_TIMEOUT_MS=15000`.

  </Accordion>

  <Accordion title='J’ai défini COPILOT_GITHUB_TOKEN, mais models status affiche "Shell env: off." Pourquoi ?'>
    `openclaw models status` indique si l'**import des variables du shell** est activé. « Shell env: off »
    ne signifie **pas** que vos variables d'environnement sont absentes - cela signifie seulement qu'OpenClaw ne chargera
    pas automatiquement votre shell de connexion.

    Si la Gateway s'exécute comme service (launchd/systemd), elle n'héritera pas de l'environnement de votre shell.
    Corrigez cela de l'une des façons suivantes :

    1. Placez le jeton dans `~/.openclaw/.env` :

       ```
       COPILOT_GITHUB_TOKEN=...
       ```

    2. Ou activez l'import du shell (`env.shellEnv.enabled: true`).
    3. Ou ajoutez-le au bloc `env` de votre configuration (s'applique uniquement s'il manque).

    Redémarrez ensuite la gateway et revérifiez :

    ```bash
    openclaw models status
    ```

    Les jetons Copilot sont lus depuis `COPILOT_GITHUB_TOKEN` (ainsi que `GH_TOKEN` / `GITHUB_TOKEN`).
    Voir [/concepts/model-providers](/fr/concepts/model-providers) et [/environment](/fr/help/environment).

  </Accordion>
</AccordionGroup>

## Sessions et plusieurs chats

<AccordionGroup>
  <Accordion title="Comment démarrer une nouvelle conversation ?">
    Envoyez `/new` ou `/reset` comme message autonome. Voir [Gestion des sessions](/fr/concepts/session).
  </Accordion>

  <Accordion title="Les sessions se réinitialisent-elles automatiquement si je n'envoie jamais /new ?">
    Les sessions peuvent expirer après `session.idleMinutes`, mais cette fonction est **désactivée par défaut** (valeur par défaut **0**).
    Définissez une valeur positive pour activer l'expiration par inactivité. Lorsqu'elle est activée, le **message suivant**
    après la période d'inactivité démarre un nouvel ID de session pour cette clé de chat.
    Cela ne supprime pas les transcriptions - cela démarre seulement une nouvelle session.

    ```json5
    {
      session: {
        idleMinutes: 240,
      },
    }
    ```

  </Accordion>

  <Accordion title="Existe-t-il un moyen de créer une équipe d'instances OpenClaw (un CEO et de nombreux agents) ?">
    Oui, via le **routage multi-agent** et les **sous-agents**. Vous pouvez créer un agent
    coordinateur et plusieurs agents workers avec leurs propres workspaces et modèles.

    Cela dit, il vaut mieux voir cela comme une **expérience amusante**. Cela consomme beaucoup de jetons et est souvent
    moins efficace qu'un seul bot avec des sessions séparées. Le modèle typique que nous
    envisageons est un seul bot avec lequel vous parlez, avec différentes sessions pour le travail parallèle. Ce
    bot peut aussi lancer des sous-agents si nécessaire.

    Documentation : [Routage multi-agent](/fr/concepts/multi-agent), [Sous-agents](/fr/tools/subagents), [CLI Agents](/cli/agents).

  </Accordion>

  <Accordion title="Pourquoi le contexte a-t-il été tronqué au milieu d'une tâche ? Comment l'éviter ?">
    Le contexte de session est limité par la fenêtre du modèle. Les longs chats, les sorties d'outils importantes ou de nombreux
    fichiers peuvent déclencher la compaction ou la troncature.

    Ce qui aide :

    - Demandez au bot de résumer l'état actuel et de l'écrire dans un fichier.
    - Utilisez `/compact` avant les longues tâches, et `/new` lors d'un changement de sujet.
    - Gardez le contexte important dans le workspace et demandez au bot de le relire.
    - Utilisez des sous-agents pour les travaux longs ou parallèles afin que le chat principal reste plus petit.
    - Choisissez un modèle avec une fenêtre de contexte plus grande si cela arrive souvent.

  </Accordion>

  <Accordion title="Comment réinitialiser complètement OpenClaw tout en le gardant installé ?">
    Utilisez la commande de réinitialisation :

    ```bash
    openclaw reset
    ```

    Réinitialisation complète non interactive :

    ```bash
    openclaw reset --scope full --yes --non-interactive
    ```

    Puis relancez la configuration :

    ```bash
    openclaw onboard --install-daemon
    ```

    Remarques :

    - L'onboarding propose aussi **Reset** s'il détecte une configuration existante. Voir [Onboarding (CLI)](/fr/start/wizard).
    - Si vous avez utilisé des profils (`--profile` / `OPENCLAW_PROFILE`), réinitialisez chaque répertoire d'état (les valeurs par défaut sont `~/.openclaw-<profile>`).
    - Réinitialisation dev : `openclaw gateway --dev --reset` (dev uniquement ; efface la configuration, les identifiants, les sessions et le workspace de dev).

  </Accordion>

  <Accordion title='J’obtiens des erreurs "context too large" - comment réinitialiser ou compacter ?'>
    Utilisez l'une de ces options :

    - **Compacter** (conserve la conversation mais résume les anciens tours) :

      ```
      /compact
      ```

      ou `/compact <instructions>` pour guider le résumé.

    - **Réinitialiser** (nouvel ID de session pour la même clé de chat) :

      ```
      /new
      /reset
      ```

    Si cela continue à se produire :

    - Activez ou ajustez le **nettoyage de session** (`agents.defaults.contextPruning`) pour élaguer les anciennes sorties d'outils.
    - Utilisez un modèle avec une fenêtre de contexte plus grande.

    Documentation : [Compaction](/fr/concepts/compaction), [Nettoyage de session](/fr/concepts/session-pruning), [Gestion des sessions](/fr/concepts/session).

  </Accordion>

  <Accordion title='Pourquoi vois-je "LLM request rejected: messages.content.tool_use.input field required" ?'>
    Il s'agit d'une erreur de validation du fournisseur : le modèle a émis un bloc `tool_use` sans le
    champ `input` requis. Cela signifie généralement que l'historique de session est obsolète ou corrompu (souvent après de longs fils
    ou un changement d'outil/schéma).

    Correction : démarrez une nouvelle session avec `/new` (message autonome).

  </Accordion>

  <Accordion title="Pourquoi est-ce que je reçois des messages heartbeat toutes les 30 minutes ?">
    Les heartbeats s'exécutent toutes les **30 min** par défaut (**1 h** lorsque vous utilisez l'authentification OAuth). Ajustez-les ou désactivez-les :

    ```json5
    {
      agents: {
        defaults: {
          heartbeat: {
            every: "2h", // or "0m" to disable
          },
        },
      },
    }
    ```

    Si `HEARTBEAT.md` existe mais est effectivement vide (seulement des lignes vides et des
    en-têtes markdown comme `# Heading`), OpenClaw ignore l'exécution heartbeat pour économiser des appels API.
    Si le fichier est manquant, heartbeat s'exécute quand même et le modèle décide quoi faire.

    Les remplacements par agent utilisent `agents.list[].heartbeat`. Documentation : [Heartbeat](/fr/gateway/heartbeat).

  </Accordion>

  <Accordion title='Dois-je ajouter un "compte bot" à un groupe WhatsApp ?'>
    Non. OpenClaw fonctionne sur **votre propre compte**, donc si vous êtes dans le groupe, OpenClaw peut le voir.
    Par défaut, les réponses dans les groupes sont bloquées jusqu'à ce que vous autorisiez des expéditeurs (`groupPolicy: "allowlist"`).

    Si vous voulez que **vous seul** puissiez déclencher des réponses dans le groupe :

    ```json5
    {
      channels: {
        whatsapp: {
          groupPolicy: "allowlist",
          groupAllowFrom: ["+15551234567"],
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Comment obtenir le JID d'un groupe WhatsApp ?">
    Option 1 (la plus rapide) : suivez les journaux et envoyez un message test dans le groupe :

    ```bash
    openclaw logs --follow --json
    ```

    Recherchez `chatId` (ou `from`) se terminant par `@g.us`, par exemple :
    `1234567890-1234567890@g.us`.

    Option 2 (si déjà configuré/autorisé) : lister les groupes depuis la configuration :

    ```bash
    openclaw directory groups list --channel whatsapp
    ```

    Documentation : [WhatsApp](/fr/channels/whatsapp), [Directory](/cli/directory), [Logs](/cli/logs).

  </Accordion>

  <Accordion title="Pourquoi OpenClaw ne répond-il pas dans un groupe ?">
    Deux causes courantes :

    - Le filtrage par mention est activé (par défaut). Vous devez @mentionner le bot (ou correspondre à `mentionPatterns`).
    - Vous avez configuré `channels.whatsapp.groups` sans `"*"` et le groupe n'est pas dans la liste d'autorisation.

    Voir [Groupes](/fr/channels/groups) et [Messages de groupe](/fr/channels/group-messages).

  </Accordion>

  <Accordion title="Les groupes/fils partagent-ils le contexte avec les DM ?">
    Les chats directs sont regroupés dans la session principale par défaut. Les groupes/canaux ont leurs propres clés de session, et les sujets Telegram / fils Discord sont des sessions séparées. Voir [Groupes](/fr/channels/groups) et [Messages de groupe](/fr/channels/group-messages).
  </Accordion>

  <Accordion title="Combien de workspaces et d'agents puis-je créer ?">
    Pas de limites strictes. Des dizaines (voire des centaines) conviennent, mais surveillez :

    - **Croissance du disque :** les sessions + transcriptions vivent sous `~/.openclaw/agents/<agentId>/sessions/`.
    - **Coût des jetons :** plus d'agents signifie plus d'utilisation concurrente de modèles.
    - **Charge opérationnelle :** profils d'authentification, workspaces et routage de canaux par agent.

    Conseils :

    - Gardez un workspace **actif** par agent (`agents.defaults.workspace`).
    - Nettoyez les anciennes sessions (supprimez JSONL ou les entrées du store) si le disque grossit.
    - Utilisez `openclaw doctor` pour repérer les workspaces orphelins et les profils incohérents.

  </Accordion>

  <Accordion title="Puis-je exécuter plusieurs bots ou chats en même temps (Slack), et comment devrais-je configurer cela ?">
    Oui. Utilisez le **routage multi-agent** pour exécuter plusieurs agents isolés et router les messages entrants par
    canal/compte/pair. Slack est pris en charge comme canal et peut être lié à des agents spécifiques.

    L'accès au navigateur est puissant mais ne signifie pas « faire tout ce qu'un humain peut faire » - l'anti-bot, les CAPTCHA et la MFA peuvent
    encore bloquer l'automatisation. Pour le contrôle navigateur le plus fiable, utilisez Chrome MCP local sur l'hôte,
    ou CDP sur la machine qui exécute réellement le navigateur.

    Configuration recommandée :

    - Hôte Gateway toujours actif (VPS/Mac mini).
    - Un agent par rôle (liaisons).
    - Canal/canaux Slack liés à ces agents.
    - Navigateur local via Chrome MCP ou un node lorsque nécessaire.

    Documentation : [Routage multi-agent](/fr/concepts/multi-agent), [Slack](/fr/channels/slack),
    [Browser](/fr/tools/browser), [Nodes](/fr/nodes).

  </Accordion>
</AccordionGroup>

## Modèles : valeurs par défaut, sélection, alias, changement

<AccordionGroup>
  <Accordion title='Qu’est-ce que le "modèle par défaut" ?'>
    Le modèle par défaut d'OpenClaw est celui que vous définissez comme :

    ```
    agents.defaults.model.primary
    ```

    Les modèles sont référencés sous la forme `provider/model` (exemple : `openai/gpt-5.4`). Si vous omettez le fournisseur, OpenClaw essaie d'abord un alias, puis une correspondance unique de fournisseur configuré pour cet ID de modèle exact, et seulement ensuite revient au fournisseur par défaut configuré comme chemin de compatibilité obsolète. Si ce fournisseur n'expose plus le modèle par défaut configuré, OpenClaw revient au premier fournisseur/modèle configuré au lieu d'exposer une valeur par défaut obsolète d'un fournisseur supprimé. Vous devriez quand même **définir explicitement** `provider/model`.

  </Accordion>

  <Accordion title="Quel modèle recommandez-vous ?">
    **Valeur par défaut recommandée :** utilisez le modèle le plus puissant et le plus récent disponible dans votre pile de fournisseurs.
    **Pour les agents avec outils activés ou à entrées non fiables :** privilégiez la puissance du modèle au coût.
    **Pour les discussions de routine/faible enjeu :** utilisez des modèles de secours moins chers et routez par rôle d'agent.

    MiniMax a sa propre documentation : [MiniMax](/fr/providers/minimax) et
    [Modèles locaux](/fr/gateway/local-models).

    Règle empirique : utilisez le **meilleur modèle que vous pouvez vous permettre** pour les travaux à fort enjeu, et un modèle
    moins cher pour les discussions courantes ou les résumés. Vous pouvez router les modèles par agent et utiliser des sous-agents pour
    paralléliser les travaux longs (chaque sous-agent consomme des jetons). Voir [Models](/fr/concepts/models) et
    [Sous-agents](/fr/tools/subagents).

    Avertissement important : les modèles plus faibles/sur-quantifiés sont plus vulnérables aux injections de prompt et aux comportements non sûrs. Voir [Sécurité](/fr/gateway/security).

    Plus de contexte : [Models](/fr/concepts/models).

  </Accordion>

  <Accordion title="Comment changer de modèle sans effacer ma configuration ?">
    Utilisez les **commandes de modèle** ou ne modifiez que les champs **model**. Évitez les remplacements complets de configuration.

    Options sûres :

    - `/model` dans le chat (rapide, par session)
    - `openclaw models set ...` (met à jour uniquement la configuration du modèle)
    - `openclaw configure --section model` (interactif)
    - modifiez `agents.defaults.model` dans `~/.openclaw/openclaw.json`

    Évitez `config.apply` avec un objet partiel sauf si vous avez l'intention de remplacer toute la configuration.
    Pour les modifications RPC, inspectez d'abord avec `config.schema.lookup` et privilégiez `config.patch`. La charge utile de lookup vous donne le chemin normalisé, la doc/les contraintes de schéma superficielles et les résumés des enfants immédiats
    pour les mises à jour partielles.
    Si vous avez écrasé la configuration, restaurez à partir d'une sauvegarde ou relancez `openclaw doctor` pour réparer.

    Documentation : [Models](/fr/concepts/models), [Configure](/cli/configure), [Config](/cli/config), [Doctor](/fr/gateway/doctor).

  </Accordion>

  <Accordion title="Puis-je utiliser des modèles auto-hébergés (llama.cpp, vLLM, Ollama) ?">
    Oui. Ollama est le chemin le plus simple pour les modèles locaux.

    Configuration la plus rapide :

    1. Installez Ollama depuis `https://ollama.com/download`
    2. Téléchargez un modèle local tel que `ollama pull glm-4.7-flash`
    3. Si vous voulez aussi des modèles cloud, exécutez `ollama signin`
    4. Exécutez `openclaw onboard` et choisissez `Ollama`
    5. Choisissez `Local` ou `Cloud + Local`

    Remarques :

    - `Cloud + Local` vous donne des modèles cloud plus vos modèles Ollama locaux
    - les modèles cloud comme `kimi-k2.5:cloud` ne nécessitent pas de téléchargement local
    - pour un changement manuel, utilisez `openclaw models list` et `openclaw models set ollama/<model>`

    Remarque de sécurité : les modèles plus petits ou fortement quantifiés sont plus vulnérables aux injections de prompt.
    Nous recommandons fortement les **grands modèles** pour tout bot pouvant utiliser des outils.
    Si vous voulez quand même de petits modèles, activez le sandboxing et des listes d'autorisation d'outils strictes.

    Documentation : [Ollama](/fr/providers/ollama), [Modèles locaux](/fr/gateway/local-models),
    [Fournisseurs de modèles](/fr/concepts/model-providers), [Sécurité](/fr/gateway/security),
    [Sandboxing](/fr/gateway/sandboxing).

  </Accordion>

  <Accordion title="Quels modèles utilisent OpenClaw, Flawd et Krill ?">
    - Ces déploiements peuvent différer et évoluer dans le temps ; il n'existe pas de recommandation fixe de fournisseur.
    - Vérifiez le paramètre d'exécution actuel sur chaque gateway avec `openclaw models status`.
    - Pour les agents sensibles à la sécurité/avec outils activés, utilisez le modèle le plus puissant et le plus récent disponible.
  </Accordion>

  <Accordion title="Comment changer de modèle à la volée (sans redémarrer) ?">
    Utilisez la commande `/model` comme message autonome :

    ```
    /model sonnet
    /model opus
    /model gpt
    /model gpt-mini
    /model gemini
    /model gemini-flash
    /model gemini-flash-lite
    ```

    Ce sont les alias intégrés. Des alias personnalisés peuvent être ajoutés via `agents.defaults.models`.

    Vous pouvez lister les modèles disponibles avec `/model`, `/model list`, ou `/model status`.

    `/model` (et `/model list`) affiche un sélecteur compact numéroté. Sélectionnez par numéro :

    ```
    /model 3
    ```

    Vous pouvez aussi forcer un profil d'authentification spécifique pour le fournisseur (par session) :

    ```
    /model opus@anthropic:default
    /model opus@anthropic:work
    ```

    Conseil : `/model status` montre quel agent est actif, quel fichier `auth-profiles.json` est utilisé et quel profil d'authentification sera tenté ensuite.
    Il affiche aussi l'endpoint configuré du fournisseur (`baseUrl`) et le mode API (`api`) lorsqu'ils sont disponibles.

    **Comment désépingler un profil défini avec @profile ?**

    Relancez `/model` **sans** le suffixe `@profile` :

    ```
    /model anthropic/claude-opus-4-6
    ```

    Si vous voulez revenir au défaut, choisissez-le depuis `/model` (ou envoyez `/model <default provider/model>`).
    Utilisez `/model status` pour confirmer quel profil d'authentification est actif.

  </Accordion>

  <Accordion title="Puis-je utiliser GPT 5.2 pour les tâches quotidiennes et Codex 5.3 pour le code ?">
    Oui. Définissez-en un comme valeur par défaut et changez selon le besoin :

    - **Changement rapide (par session) :** `/model gpt-5.4` pour les tâches quotidiennes, `/model openai-codex/gpt-5.4` pour le code avec Codex OAuth.
    - **Valeur par défaut + changement :** définissez `agents.defaults.model.primary` sur `openai/gpt-5.4`, puis basculez vers `openai-codex/gpt-5.4` pour le code (ou l'inverse).
    - **Sous-agents :** routez les tâches de code vers des sous-agents avec un autre modèle par défaut.

    Voir [Models](/fr/concepts/models) et [Commandes slash](/fr/tools/slash-commands).

  </Accordion>

  <Accordion title="Comment configurer le mode rapide pour GPT 5.4 ?">
    Utilisez soit un basculement de session, soit une valeur par défaut dans la configuration :

    - **Par session :** envoyez `/fast on` pendant que la session utilise `openai/gpt-5.4` ou `openai-codex/gpt-5.4`.
    - **Valeur par défaut par modèle :** définissez `agents.defaults.models["openai/gpt-5.4"].params.fastMode` à `true`.
    - **Codex OAuth aussi :** si vous utilisez aussi `openai-codex/gpt-5.4`, définissez-y le même indicateur.

    Exemple :

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openai/gpt-5.4": {
              params: {
                fastMode: true,
              },
            },
            "openai-codex/gpt-5.4": {
              params: {
                fastMode: true,
              },
            },
          },
        },
      },
    }
    ```

    Pour OpenAI, le mode rapide correspond à `service_tier = "priority"` sur les requêtes Responses natives prises en charge. Les remplacements de session `/fast` priment sur les valeurs de configuration.

    Voir [Thinking and fast mode](/fr/tools/thinking) et [OpenAI fast mode](/fr/providers/openai#openai-fast-mode).

  </Accordion>

  <Accordion title='Pourquoi vois-je "Model ... is not allowed" puis aucune réponse ?'>
    Si `agents.defaults.models` est défini, il devient la **liste d'autorisation** pour `/model` et tous
    les remplacements de session. Choisir un modèle qui n'est pas dans cette liste renvoie :

    ```
    Model "provider/model" is not allowed. Use /model to list available models.
    ```

    Cette erreur est renvoyée **à la place** d'une réponse normale. Corrigez cela : ajoutez le modèle à
    `agents.defaults.models`, supprimez la liste d'autorisation ou choisissez un modèle depuis `/model list`.

  </Accordion>

  <Accordion title='Pourquoi vois-je "Unknown model: minimax/MiniMax-M2.7" ?'>
    Cela signifie que le **fournisseur n'est pas configuré** (aucune configuration MiniMax ou
    aucun profil d'authentification n'a été trouvé), donc le modèle ne peut pas être résolu.

    Liste de contrôle de correction :

    1. Mettez à jour vers une version récente d'OpenClaw (ou exécutez depuis la source `main`), puis redémarrez la gateway.
    2. Assurez-vous que MiniMax est configuré (assistant ou JSON), ou que l'authentification MiniMax
       existe dans l'environnement/les profils d'authentification afin que le fournisseur correspondant puisse être injecté
       (`MINIMAX_API_KEY` pour `minimax`, `MINIMAX_OAUTH_TOKEN` ou l'OAuth MiniMax
       stocké pour `minimax-portal`).
    3. Utilisez l'ID de modèle exact (respect de la casse) pour votre chemin d'authentification :
       `minimax/MiniMax-M2.7` ou `minimax/MiniMax-M2.7-highspeed` pour une configuration
       avec clé API, ou `minimax-portal/MiniMax-M2.7` /
       `minimax-portal/MiniMax-M2.7-highspeed` pour une configuration
       OAuth.
    4. Exécutez :

       ```bash
       openclaw models list
       ```

       et choisissez dans la liste (ou `/model list` dans le chat).

    Voir [MiniMax](/fr/providers/minimax) et [Models](/fr/concepts/models).

  </Accordion>

  <Accordion title="Puis-je utiliser MiniMax par défaut et OpenAI pour les tâches complexes ?">
    Oui. Utilisez **MiniMax par défaut** et changez de modèle **par session** si nécessaire.
    Les secours servent aux **erreurs**, pas aux « tâches difficiles », donc utilisez `/model` ou un agent séparé.

    **Option A : changer par session**

    ```json5
    {
      env: { MINIMAX_API_KEY: "sk-...", OPENAI_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "minimax/MiniMax-M2.7" },
          models: {
            "minimax/MiniMax-M2.7": { alias: "minimax" },
            "openai/gpt-5.4": { alias: "gpt" },
          },
        },
      },
    }
    ```

    Puis :

    ```
    /model gpt
    ```

    **Option B : agents séparés**

    - Valeur par défaut de l'agent A : MiniMax
    - Valeur par défaut de l'agent B : OpenAI
    - Routez par agent ou utilisez `/agent` pour changer

    Documentation : [Models](/fr/concepts/models), [Routage multi-agent](/fr/concepts/multi-agent), [MiniMax](/fr/providers/minimax), [OpenAI](/fr/providers/openai).

  </Accordion>

  <Accordion title="opus / sonnet / gpt sont-ils des raccourcis intégrés ?">
    Oui. OpenClaw inclut quelques abréviations par défaut (appliquées uniquement lorsque le modèle existe dans `agents.defaults.models`) :

    - `opus` → `anthropic/claude-opus-4-6`
    - `sonnet` → `anthropic/claude-sonnet-4-6`
    - `gpt` → `openai/gpt-5.4`
    - `gpt-mini` → `openai/gpt-5.4-mini`
    - `gpt-nano` → `openai/gpt-5.4-nano`
    - `gemini` → `google/gemini-3.1-pro-preview`
    - `gemini-flash` → `google/gemini-3-flash-preview`
    - `gemini-flash-lite` → `google/gemini-3.1-flash-lite-preview`

    Si vous définissez votre propre alias avec le même nom, votre valeur l'emporte.

  </Accordion>

  <Accordion title="Comment définir/remplacer des raccourcis de modèle (alias) ?">
    Les alias proviennent de `agents.defaults.models.<modelId>.alias`. Exemple :

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "anthropic/claude-opus-4-6" },
          models: {
            "anthropic/claude-opus-4-6": { alias: "opus" },
            "anthropic/claude-sonnet-4-6": { alias: "sonnet" },
            "anthropic/claude-haiku-4-5": { alias: "haiku" },
          },
        },
      },
    }
    ```

    Ensuite, `/model sonnet` (ou `/<alias>` lorsque cela est pris en charge) se résout vers cet ID de modèle.

  </Accordion>

  <Accordion title="Comment ajouter des modèles d'autres fournisseurs comme OpenRouter ou Z.AI ?">
    OpenRouter (paiement au jeton ; nombreux modèles) :

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "openrouter/anthropic/claude-sonnet-4-6" },
          models: { "openrouter/anthropic/claude-sonnet-4-6": {} },
        },
      },
      env: { OPENROUTER_API_KEY: "sk-or-..." },
    }
    ```

    Z.AI (modèles GLM) :

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "zai/glm-5" },
          models: { "zai/glm-5": {} },
        },
      },
      env: { ZAI_API_KEY: "..." },
    }
    ```

    Si vous référencez un fournisseur/modèle mais que la clé requise du fournisseur manque, vous obtiendrez une erreur d'authentification à l'exécution (par exemple `No API key found for provider "zai"`).

    **No API key found for provider après l'ajout d'un nouvel agent**

    Cela signifie généralement que le **nou