---
read_when:
    - Répondre aux questions courantes sur la configuration, l’installation, l’onboarding ou l’assistance à l’exécution
    - Trier les problèmes signalés par les utilisateurs avant un débogage plus approfondi
summary: Questions fréquentes sur l’installation, la configuration et l’utilisation d’OpenClaw
title: FAQ
x-i18n:
    generated_at: "2026-04-08T02:20:59Z"
    model: gpt-5.4
    provider: openai
    source_hash: 001b4605966b45b08108606f76ae937ec348c2179b04cf6fb34fef94833705e6
    source_path: help/faq.md
    workflow: 15
---

# FAQ

Réponses rapides plus dépannage approfondi pour des configurations réelles (développement local, VPS, multi-agent, clés OAuth/API, basculement de modèle). Pour le diagnostic à l’exécution, voir [Dépannage](/fr/gateway/troubleshooting). Pour la référence de configuration complète, voir [Configuration](/fr/gateway/configuration).

## Premières 60 secondes si quelque chose est cassé

1. **État rapide (première vérification)**

   ```bash
   openclaw status
   ```

   Résumé local rapide : OS + mise à jour, accessibilité de la passerelle/du service, agents/sessions, configuration du fournisseur + problèmes d’exécution (lorsque la passerelle est joignable).

2. **Rapport copiable/collable (sans danger à partager)**

   ```bash
   openclaw status --all
   ```

   Diagnostic en lecture seule avec fin de journal (tokens masqués).

3. **État du démon + du port**

   ```bash
   openclaw gateway status
   ```

   Affiche l’environnement d’exécution du superviseur par rapport à l’accessibilité RPC, l’URL cible de la sonde, et quelle configuration le service a probablement utilisée.

4. **Sondes approfondies**

   ```bash
   openclaw status --deep
   ```

   Exécute une sonde de santé active de la passerelle, y compris des sondes de canaux lorsque c’est pris en charge
   (nécessite une passerelle joignable). Voir [Santé](/fr/gateway/health).

5. **Suivre le dernier journal**

   ```bash
   openclaw logs --follow
   ```

   Si le RPC est indisponible, utilisez à la place :

   ```bash
   tail -f "$(ls -t /tmp/openclaw/openclaw-*.log | head -1)"
   ```

   Les journaux de fichier sont distincts des journaux de service ; voir [Journalisation](/fr/logging) et [Dépannage](/fr/gateway/troubleshooting).

6. **Exécuter le doctor (réparations)**

   ```bash
   openclaw doctor
   ```

   Répare/migre la configuration et l’état + exécute des contrôles de santé. Voir [Doctor](/fr/gateway/doctor).

7. **Instantané de la passerelle**

   ```bash
   openclaw health --json
   openclaw health --verbose   # shows the target URL + config path on errors
   ```

   Demande à la passerelle en cours d’exécution un instantané complet (WS uniquement). Voir [Santé](/fr/gateway/health).

## Démarrage rapide et configuration du premier lancement

<AccordionGroup>
  <Accordion title="Je suis bloqué, quelle est la façon la plus rapide de me débloquer ?">
    Utilisez un agent IA local qui peut **voir votre machine**. C’est bien plus efficace que de demander
    dans Discord, car la plupart des cas de type « je suis bloqué » sont des **problèmes de configuration locale ou d’environnement**
    que des personnes à distance ne peuvent pas inspecter.

    - **Claude Code** : [https://www.anthropic.com/claude-code/](https://www.anthropic.com/claude-code/)
    - **OpenAI Codex** : [https://openai.com/codex/](https://openai.com/codex/)

    Ces outils peuvent lire le dépôt, exécuter des commandes, inspecter les journaux et aider à corriger
    votre configuration au niveau de la machine (PATH, services, permissions, fichiers d’authentification). Donnez-leur la **copie complète des sources**
    via l’installation modifiable (git) :

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Cela installe OpenClaw **à partir d’une copie git**, afin que l’agent puisse lire le code + la documentation
    et raisonner sur la version exacte que vous exécutez. Vous pouvez toujours revenir à la version stable plus tard
    en relançant l’installateur sans `--install-method git`.

    Conseil : demandez à l’agent de **planifier et superviser** la correction (étape par étape), puis d’exécuter uniquement les
    commandes nécessaires. Cela garde les changements petits et plus faciles à auditer.

    Si vous découvrez un vrai bug ou correctif, veuillez ouvrir une issue GitHub ou envoyer une PR :
    [https://github.com/openclaw/openclaw/issues](https://github.com/openclaw/openclaw/issues)
    [https://github.com/openclaw/openclaw/pulls](https://github.com/openclaw/openclaw/pulls)

    Commencez par ces commandes (partagez les sorties lorsque vous demandez de l’aide) :

    ```bash
    openclaw status
    openclaw models status
    openclaw doctor
    ```

    Ce qu’elles font :

    - `openclaw status` : instantané rapide de la santé de la passerelle/de l’agent + configuration de base.
    - `openclaw models status` : vérifie l’authentification des fournisseurs + la disponibilité des modèles.
    - `openclaw doctor` : valide et répare les problèmes courants de configuration/état.

    Autres vérifications CLI utiles : `openclaw status --all`, `openclaw logs --follow`,
    `openclaw gateway status`, `openclaw health --verbose`.

    Boucle de débogage rapide : [Premières 60 secondes si quelque chose est cassé](#premières-60-secondes-si-quelque-chose-est-cassé).
    Documentation d’installation : [Installer](/fr/install), [Options de l’installateur](/fr/install/installer), [Mise à jour](/fr/install/updating).

  </Accordion>

  <Accordion title="Heartbeat continue d’être ignoré. Que signifient les raisons d’ignorance ?">
    Raisons courantes d’ignorance de heartbeat :

    - `quiet-hours` : en dehors de la fenêtre d’heures actives configurée
    - `empty-heartbeat-file` : `HEARTBEAT.md` existe mais ne contient qu’une structure vide ou seulement des en-têtes
    - `no-tasks-due` : le mode tâches de `HEARTBEAT.md` est actif mais aucun intervalle de tâche n’est encore arrivé à échéance
    - `alerts-disabled` : toute la visibilité heartbeat est désactivée (`showOk`, `showAlerts` et `useIndicator` sont tous désactivés)

    En mode tâches, les horodatages d’échéance ne sont avancés qu’après une véritable exécution de heartbeat
    terminée. Les exécutions ignorées ne marquent pas les tâches comme terminées.

    Documentation : [Heartbeat](/fr/gateway/heartbeat), [Automatisation et tâches](/fr/automation).

  </Accordion>

  <Accordion title="Méthode recommandée pour installer et configurer OpenClaw">
    Le dépôt recommande une exécution depuis les sources et l’utilisation de l’onboarding :

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash
    openclaw onboard --install-daemon
    ```

    L’assistant peut aussi construire automatiquement les ressources d’interface utilisateur. Après l’onboarding, vous exécutez généralement la passerelle sur le port **18789**.

    Depuis les sources (contributeurs/dev) :

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    pnpm ui:build # auto-installs UI deps on first run
    openclaw onboard
    ```

    Si vous n’avez pas encore d’installation globale, exécutez-la via `pnpm openclaw onboard`.

  </Accordion>

  <Accordion title="Comment ouvrir le tableau de bord après l’onboarding ?">
    L’assistant ouvre votre navigateur avec une URL propre du tableau de bord (sans jeton dans l’URL) juste après l’onboarding et affiche aussi le lien dans le résumé. Gardez cet onglet ouvert ; s’il ne s’est pas lancé, copiez/collez l’URL affichée sur la même machine.
  </Accordion>

  <Accordion title="Comment authentifier le tableau de bord sur localhost par rapport à un hôte distant ?">
    **Localhost (même machine) :**

    - Ouvrez `http://127.0.0.1:18789/`.
    - S’il demande une authentification par secret partagé, collez le token ou le mot de passe configuré dans les paramètres de Control UI.
    - Source du token : `gateway.auth.token` (ou `OPENCLAW_GATEWAY_TOKEN`).
    - Source du mot de passe : `gateway.auth.password` (ou `OPENCLAW_GATEWAY_PASSWORD`).
    - Si aucun secret partagé n’est encore configuré, générez un token avec `openclaw doctor --generate-gateway-token`.

    **Pas sur localhost :**

    - **Tailscale Serve** (recommandé) : gardez la liaison loopback, exécutez `openclaw gateway --tailscale serve`, ouvrez `https://<magicdns>/`. Si `gateway.auth.allowTailscale` vaut `true`, les en-têtes d’identité satisfont l’authentification de Control UI/WebSocket (pas de secret partagé à coller, en supposant un hôte de passerelle de confiance) ; les API HTTP exigent toujours une authentification par secret partagé sauf si vous utilisez volontairement `none` en ingress privé ou une authentification HTTP par proxy de confiance.
      Les tentatives d’authentification Serve concurrentes invalides depuis le même client sont sérialisées avant que le limiteur d’échecs d’authentification ne les enregistre, donc la deuxième mauvaise tentative peut déjà afficher `retry later`.
    - **Liaison Tailnet** : exécutez `openclaw gateway --bind tailnet --token "<token>"` (ou configurez une authentification par mot de passe), ouvrez `http://<tailscale-ip>:18789/`, puis collez le secret partagé correspondant dans les paramètres du tableau de bord.
    - **Proxy inverse avec gestion d’identité** : gardez la passerelle derrière un proxy de confiance non loopback, configurez `gateway.auth.mode: "trusted-proxy"`, puis ouvrez l’URL du proxy.
    - **Tunnel SSH** : `ssh -N -L 18789:127.0.0.1:18789 user@host` puis ouvrez `http://127.0.0.1:18789/`. L’authentification par secret partagé s’applique toujours via le tunnel ; collez le token ou le mot de passe configuré si demandé.

    Voir [Tableau de bord](/web/dashboard) et [Surfaces Web](/web) pour les modes de liaison et les détails d’authentification.

  </Accordion>

  <Accordion title="Pourquoi y a-t-il deux configurations d’approbation exec pour les approbations de chat ?">
    Elles contrôlent des couches différentes :

    - `approvals.exec` : transfère les invites d’approbation vers des destinations de chat
    - `channels.<channel>.execApprovals` : fait agir ce canal comme client d’approbation natif pour les approbations exec

    La politique exec de l’hôte reste la véritable barrière d’approbation. La configuration du chat contrôle seulement où apparaissent les
    invites d’approbation et comment les gens peuvent répondre.

    Dans la plupart des configurations, vous n’avez **pas** besoin des deux :

    - Si le chat prend déjà en charge les commandes et les réponses, `/approve` dans le même chat fonctionne via le chemin partagé.
    - Si un canal natif pris en charge peut déduire les approbateurs en toute sécurité, OpenClaw active désormais automatiquement les approbations natives DM-first lorsque `channels.<channel>.execApprovals.enabled` est non défini ou vaut `"auto"`.
    - Lorsque des cartes/boutons d’approbation natifs sont disponibles, cette interface native est le chemin principal ; l’agent ne devrait inclure une commande manuelle `/approve` que si le résultat de l’outil indique que les approbations de chat ne sont pas disponibles ou que l’approbation manuelle est le seul chemin.
    - Utilisez `approvals.exec` uniquement lorsque les invites doivent aussi être transférées à d’autres chats ou à des salles d’opérations explicites.
    - Utilisez `channels.<channel>.execApprovals.target: "channel"` ou `"both"` uniquement lorsque vous voulez explicitement que les invites d’approbation soient republiées dans la salle/le sujet d’origine.
    - Les approbations de plugin sont encore distinctes : elles utilisent `/approve` dans le même chat par défaut, un transfert optionnel via `approvals.plugin`, et seuls certains canaux natifs conservent en plus une gestion native des approbations de plugin.

    En bref : le transfert sert au routage, la configuration du client natif sert à une UX plus riche spécifique au canal.
    Voir [Approvals Exec](/fr/tools/exec-approvals).

  </Accordion>

  <Accordion title="Quel environnement d’exécution me faut-il ?">
    Node **>= 22** est requis. `pnpm` est recommandé. Bun n’est **pas recommandé** pour la passerelle.
  </Accordion>

  <Accordion title="Est-ce que ça fonctionne sur Raspberry Pi ?">
    Oui. La passerelle est légère : la documentation indique que **512 Mo-1 Go de RAM**, **1 cœur** et environ **500 Mo**
    de disque suffisent pour un usage personnel, et note qu’un **Raspberry Pi 4 peut l’exécuter**.

    Si vous voulez plus de marge (journaux, médias, autres services), **2 Go sont recommandés**, mais ce
    n’est pas un minimum strict.

    Conseil : un petit Pi/VPS peut héberger la passerelle, et vous pouvez associer des **nœuds** sur votre ordinateur portable/téléphone pour
    l’écran/la caméra/le canvas locaux ou l’exécution de commandes. Voir [Nœuds](/fr/nodes).

  </Accordion>

  <Accordion title="Des conseils pour les installations sur Raspberry Pi ?">
    En bref : ça fonctionne, mais attendez-vous à quelques aspérités.

    - Utilisez un OS **64 bits** et gardez Node >= 22.
    - Préférez l’**installation modifiable (git)** afin de pouvoir voir les journaux et mettre à jour rapidement.
    - Commencez sans canaux/Skills, puis ajoutez-les un par un.
    - Si vous rencontrez des problèmes binaires étranges, c’est généralement un problème de **compatibilité ARM**.

    Documentation : [Linux](/fr/platforms/linux), [Installer](/fr/install).

  </Accordion>

  <Accordion title="C’est bloqué sur wake up my friend / l’onboarding n’éclot pas. Que faire ?">
    Cet écran dépend du fait que la passerelle soit joignable et authentifiée. L’interface TUI envoie aussi
    automatiquement « Wake up, my friend! » lors de la première éclosion. Si vous voyez cette ligne **sans réponse**
    et que les tokens restent à 0, l’agent ne s’est jamais exécuté.

    1. Redémarrez la passerelle :

    ```bash
    openclaw gateway restart
    ```

    2. Vérifiez l’état + l’authentification :

    ```bash
    openclaw status
    openclaw models status
    openclaw logs --follow
    ```

    3. Si cela bloque toujours, exécutez :

    ```bash
    openclaw doctor
    ```

    Si la passerelle est distante, assurez-vous que le tunnel/la connexion Tailscale est active et que l’interface utilisateur
    pointe vers la bonne passerelle. Voir [Accès distant](/fr/gateway/remote).

  </Accordion>

  <Accordion title="Puis-je migrer ma configuration vers une nouvelle machine (Mac mini) sans refaire l’onboarding ?">
    Oui. Copiez le **répertoire d’état** et l’**espace de travail**, puis exécutez Doctor une fois. Cela
    conserve votre bot « exactement pareil » (mémoire, historique de session, authentification et état
    des canaux) tant que vous copiez **les deux** emplacements :

    1. Installez OpenClaw sur la nouvelle machine.
    2. Copiez `$OPENCLAW_STATE_DIR` (par défaut : `~/.openclaw`) depuis l’ancienne machine.
    3. Copiez votre espace de travail (par défaut : `~/.openclaw/workspace`).
    4. Exécutez `openclaw doctor` et redémarrez le service de la passerelle.

    Cela préserve la configuration, les profils d’authentification, les identifiants WhatsApp, les sessions et la mémoire. Si vous êtes en
    mode distant, rappelez-vous que l’hôte de la passerelle possède le magasin de sessions et l’espace de travail.

    **Important :** si vous ne faites que valider/pousser votre espace de travail sur GitHub, vous sauvegardez
    **la mémoire + les fichiers de bootstrap**, mais **pas** l’historique de session ni l’authentification. Ceux-ci se trouvent
    sous `~/.openclaw/` (par exemple `~/.openclaw/agents/<agentId>/sessions/`).

    Voir aussi : [Migration](/fr/install/migrating), [Où les choses sont stockées sur le disque](#où-les-choses-sont-stockées-sur-le-disque),
    [Espace de travail d’agent](/fr/concepts/agent-workspace), [Doctor](/fr/gateway/doctor),
    [Mode distant](/fr/gateway/remote).

  </Accordion>

  <Accordion title="Où voir les nouveautés de la dernière version ?">
    Consultez le changelog GitHub :
    [https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md)

    Les entrées les plus récentes sont en haut. Si la section du haut est marquée **Unreleased**, la section datée suivante
    est la dernière version publiée. Les entrées sont regroupées par **Highlights**, **Changes** et
    **Fixes** (plus des sections docs/autres si nécessaire).

  </Accordion>

  <Accordion title="Impossible d’accéder à docs.openclaw.ai (erreur SSL)">
    Certaines connexions Comcast/Xfinity bloquent incorrectement `docs.openclaw.ai` via Xfinity
    Advanced Security. Désactivez-le ou ajoutez `docs.openclaw.ai` à la liste autorisée, puis réessayez.
    Merci de nous aider à le débloquer en le signalant ici : [https://spa.xfinity.com/check_url_status](https://spa.xfinity.com/check_url_status).

    Si vous n’arrivez toujours pas à joindre le site, la documentation est également en miroir sur GitHub :
    [https://github.com/openclaw/openclaw/tree/main/docs](https://github.com/openclaw/openclaw/tree/main/docs)

  </Accordion>

  <Accordion title="Différence entre stable et beta">
    **Stable** et **beta** sont des **npm dist-tags**, pas des lignes de code séparées :

    - `latest` = stable
    - `beta` = build anticipé pour les tests

    Habituellement, une version stable arrive d’abord sur **beta**, puis une étape explicite
    de promotion déplace cette même version vers `latest`. Les mainteneurs peuvent aussi
    publier directement sur `latest` si nécessaire. C’est pourquoi beta et stable peuvent
    pointer vers la **même version** après promotion.

    Voir ce qui a changé :
    [https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md)

    Pour les commandes d’installation en une ligne et la différence entre beta et dev, voir l’accordéon ci-dessous.

  </Accordion>

  <Accordion title="Comment installer la version beta et quelle est la différence entre beta et dev ?">
    **Beta** est le dist-tag npm `beta` (peut correspondre à `latest` après promotion).
    **Dev** est la tête mobile de `main` (git) ; lorsqu’elle est publiée, elle utilise le dist-tag npm `dev`.

    Commandes en une ligne (macOS/Linux) :

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --beta
    ```

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Installateur Windows (PowerShell) :
    [https://openclaw.ai/install.ps1](https://openclaw.ai/install.ps1)

    Plus de détails : [Canaux de développement](/fr/install/development-channels) et [Options de l’installateur](/fr/install/installer).

  </Accordion>

  <Accordion title="Comment essayer les dernières nouveautés ?">
    Deux options :

    1. **Canal dev (copie git) :**

    ```bash
    openclaw update --channel dev
    ```

    Cela bascule sur la branche `main` et met à jour depuis les sources.

    2. **Installation modifiable (depuis le site d’installation) :**

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Cela vous donne un dépôt local que vous pouvez modifier, puis mettre à jour via git.

    Si vous préférez un clone propre manuellement, utilisez :

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    ```

    Documentation : [Mettre à jour](/cli/update), [Canaux de développement](/fr/install/development-channels),
    [Installer](/fr/install).

  </Accordion>

  <Accordion title="Combien de temps prennent généralement l’installation et l’onboarding ?">
    Estimation approximative :

    - **Installation :** 2-5 minutes
    - **Onboarding :** 5-15 minutes selon le nombre de canaux/modèles que vous configurez

    Si cela bloque, utilisez [Installateur bloqué](#démarrage-rapide-et-configuration-du-premier-lancement)
    et la boucle de débogage rapide dans [Je suis bloqué](#démarrage-rapide-et-configuration-du-premier-lancement).

  </Accordion>

  <Accordion title="Installateur bloqué ? Comment obtenir plus de retours ?">
    Relancez l’installateur avec une **sortie verbeuse** :

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

    Plus d’options : [Options de l’installateur](/fr/install/installer).

  </Accordion>

  <Accordion title="L’installation Windows indique git not found ou openclaw not recognized">
    Deux problèmes Windows courants :

    **1) erreur npm spawn git / git not found**

    - Installez **Git for Windows** et assurez-vous que `git` est sur votre PATH.
    - Fermez et rouvrez PowerShell, puis relancez l’installateur.

    **2) openclaw is not recognized après l’installation**

    - Votre dossier npm global bin n’est pas sur le PATH.
    - Vérifiez le chemin :

      ```powershell
      npm config get prefix
      ```

    - Ajoutez ce répertoire à votre PATH utilisateur (pas besoin de suffixe `\bin` sous Windows ; sur la plupart des systèmes, c’est `%AppData%\npm`).
    - Fermez et rouvrez PowerShell après la mise à jour du PATH.

    Si vous voulez l’expérience la plus fluide sur Windows, utilisez **WSL2** au lieu de Windows natif.
    Documentation : [Windows](/fr/platforms/windows).

  </Accordion>

  <Accordion title="La sortie exec sous Windows affiche du texte chinois illisible. Que dois-je faire ?">
    Il s’agit généralement d’un décalage de page de codes de console dans les shells Windows natifs.

    Symptômes :

    - la sortie `system.run`/`exec` affiche le chinois sous forme de texte corrompu
    - la même commande s’affiche correctement dans un autre profil de terminal

    Solution rapide dans PowerShell :

    ```powershell
    chcp 65001
    [Console]::InputEncoding = [System.Text.UTF8Encoding]::new($false)
    [Console]::OutputEncoding = [System.Text.UTF8Encoding]::new($false)
    $OutputEncoding = [System.Text.UTF8Encoding]::new($false)
    ```

    Puis redémarrez la passerelle et réessayez votre commande :

    ```powershell
    openclaw gateway restart
    ```

    Si vous reproduisez encore cela sur la dernière version d’OpenClaw, suivez/signalez-le ici :

    - [Issue #30640](https://github.com/openclaw/openclaw/issues/30640)

  </Accordion>

  <Accordion title="La documentation n’a pas répondu à ma question. Comment obtenir une meilleure réponse ?">
    Utilisez l’**installation modifiable (git)** afin d’avoir localement l’ensemble du code source et de la documentation, puis interrogez
    votre bot (ou Claude/Codex) _depuis ce dossier_ pour qu’il puisse lire le dépôt et répondre précisément.

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Plus de détails : [Installer](/fr/install) et [Options de l’installateur](/fr/install/installer).

  </Accordion>

  <Accordion title="Comment installer OpenClaw sur Linux ?">
    Réponse courte : suivez le guide Linux, puis exécutez l’onboarding.

    - Parcours rapide Linux + installation du service : [Linux](/fr/platforms/linux).
    - Guide complet : [Bien démarrer](/fr/start/getting-started).
    - Installateur + mises à jour : [Installation et mises à jour](/fr/install/updating).

  </Accordion>

  <Accordion title="Comment installer OpenClaw sur un VPS ?">
    N’importe quel VPS Linux fonctionne. Installez sur le serveur, puis utilisez SSH/Tailscale pour joindre la passerelle.

    Guides : [exe.dev](/fr/install/exe-dev), [Hetzner](/fr/install/hetzner), [Fly.io](/fr/install/fly).
    Accès distant : [Passerelle distante](/fr/gateway/remote).

  </Accordion>

  <Accordion title="Où sont les guides d’installation cloud/VPS ?">
    Nous maintenons un **hub d’hébergement** avec les fournisseurs courants. Choisissez-en un et suivez le guide :

    - [Hébergement VPS](/fr/vps) (tous les fournisseurs au même endroit)
    - [Fly.io](/fr/install/fly)
    - [Hetzner](/fr/install/hetzner)
    - [exe.dev](/fr/install/exe-dev)

    Fonctionnement dans le cloud : la **passerelle s’exécute sur le serveur**, et vous y accédez
    depuis votre ordinateur portable/téléphone via Control UI (ou Tailscale/SSH). Votre état + votre espace de travail
    résident sur le serveur, donc considérez l’hôte comme la source de vérité et sauvegardez-le.

    Vous pouvez associer des **nœuds** (Mac/iOS/Android/headless) à cette passerelle cloud pour accéder à
    l’écran/la caméra/le canvas locaux ou exécuter des commandes sur votre ordinateur portable tout en gardant la
    passerelle dans le cloud.

    Hub : [Plateformes](/fr/platforms). Accès distant : [Passerelle distante](/fr/gateway/remote).
    Nœuds : [Nœuds](/fr/nodes), [CLI des nœuds](/cli/nodes).

  </Accordion>

  <Accordion title="Puis-je demander à OpenClaw de se mettre à jour tout seul ?">
    Réponse courte : **possible, mais non recommandé**. Le flux de mise à jour peut redémarrer la
    passerelle (ce qui coupe la session active), peut nécessiter une copie git propre et
    peut demander une confirmation. Plus sûr : exécuter les mises à jour depuis un shell en tant qu’opérateur.

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

    Documentation : [Mettre à jour](/cli/update), [Mise à jour](/fr/install/updating).

  </Accordion>

  <Accordion title="Que fait réellement l’onboarding ?">
    `openclaw onboard` est le chemin recommandé de configuration. En **mode local**, il vous guide à travers :

    - **Configuration du modèle/de l’authentification** (OAuth du fournisseur, clés API, setup-token Anthropic, plus options de modèles locaux comme LM Studio)
    - Emplacement de l’**espace de travail** + fichiers de bootstrap
    - **Paramètres de la passerelle** (liaison/port/authentification/Tailscale)
    - **Canaux** (WhatsApp, Telegram, Discord, Mattermost, Signal, iMessage, plus les plugins de canaux fournis comme QQ Bot)
    - **Installation du démon** (LaunchAgent sur macOS ; unité utilisateur systemd sur Linux/WSL2)
    - **Contrôles de santé** et sélection des **Skills**

    Il avertit également si votre modèle configuré est inconnu ou s’il manque une authentification.

  </Accordion>

  <Accordion title="Ai-je besoin d’un abonnement Claude ou OpenAI pour l’exécuter ?">
    Non. Vous pouvez exécuter OpenClaw avec des **clés API** (Anthropic/OpenAI/autres) ou avec des
    **modèles uniquement locaux** pour que vos données restent sur votre appareil. Les abonnements (Claude
    Pro/Max ou OpenAI Codex) sont des moyens optionnels d’authentifier ces fournisseurs.

    Pour Anthropic dans OpenClaw, la séparation pratique est :

    - **Clé API Anthropic** : facturation API Anthropic normale
    - **Authentification Claude CLI / abonnement Claude dans OpenClaw** : le personnel d’Anthropic
      nous a dit que cet usage est de nouveau autorisé, et OpenClaw traite l’utilisation de `claude -p`
      comme approuvée pour cette intégration sauf si Anthropic publie une nouvelle
      politique

    Pour des hôtes de passerelle durables, les clés API Anthropic restent la
    configuration la plus prévisible. OpenAI Codex OAuth est explicitement pris en charge pour les outils externes comme OpenClaw.

    OpenClaw prend aussi en charge d’autres options hébergées de type abonnement, notamment
    **Qwen Cloud Coding Plan**, **MiniMax Coding Plan** et
    **Z.AI / GLM Coding Plan**.

    Documentation : [Anthropic](/fr/providers/anthropic), [OpenAI](/fr/providers/openai),
    [Qwen Cloud](/fr/providers/qwen),
    [MiniMax](/fr/providers/minimax), [Modèles GLM](/fr/providers/glm),
    [Modèles locaux](/fr/gateway/local-models), [Modèles](/fr/concepts/models).

  </Accordion>

  <Accordion title="Puis-je utiliser un abonnement Claude Max sans clé API ?">
    Oui.

    Le personnel d’Anthropic nous a dit que l’usage de Claude CLI de type OpenClaw est de nouveau autorisé, donc
    OpenClaw traite l’authentification par abonnement Claude et l’usage de `claude -p` comme approuvés
    pour cette intégration sauf si Anthropic publie une nouvelle politique. Si vous voulez
    la configuration côté serveur la plus prévisible, utilisez plutôt une clé API Anthropic.

  </Accordion>

  <Accordion title="Prenez-vous en charge l’authentification par abonnement Claude (Claude Pro ou Max) ?">
    Oui.

    Le personnel d’Anthropic nous a dit que cet usage est de nouveau autorisé, donc OpenClaw traite
    la réutilisation de Claude CLI et l’utilisation de `claude -p` comme approuvées pour cette intégration
    sauf si Anthropic publie une nouvelle politique.

    Le setup-token Anthropic reste disponible comme chemin de jeton pris en charge dans OpenClaw, mais OpenClaw préfère désormais la réutilisation de Claude CLI et `claude -p` lorsqu’ils sont disponibles.
    Pour les charges de travail de production ou multi-utilisateurs, l’authentification par clé API Anthropic reste le
    choix plus sûr et plus prévisible. Si vous voulez d’autres options hébergées de type abonnement
    dans OpenClaw, voir [OpenAI](/fr/providers/openai), [Qwen / Model
    Cloud](/fr/providers/qwen), [MiniMax](/fr/providers/minimax) et [Modèles
    GLM](/fr/providers/glm).

  </Accordion>

<a id="why-am-i-seeing-http-429-ratelimiterror-from-anthropic"></a>
<Accordion title="Pourquoi vois-je HTTP 429 rate_limit_error d’Anthropic ?">
Cela signifie que votre **quota/limite de débit Anthropic** est épuisé pour la fenêtre actuelle. Si vous
utilisez **Claude CLI**, attendez la réinitialisation de la fenêtre ou mettez à niveau votre formule. Si vous
utilisez une **clé API Anthropic**, vérifiez l’usage/la facturation dans la console Anthropic
et augmentez les limites si nécessaire.

    Si le message est précisément :
    `Extra usage is required for long context requests`, la requête essaie d’utiliser
    la bêta 1M de contexte d’Anthropic (`context1m: true`). Cela ne fonctionne que si votre
    identifiant est éligible à la facturation long contexte (facturation par clé API ou
    chemin de connexion Claude d’OpenClaw avec Extra Usage activé).

    Conseil : définissez un **modèle de repli** afin qu’OpenClaw puisse continuer à répondre lorsqu’un fournisseur est limité.
    Voir [Models](/cli/models), [OAuth](/fr/concepts/oauth), et
    [/gateway/troubleshooting#anthropic-429-extra-usage-required-for-long-context](/fr/gateway/troubleshooting#anthropic-429-extra-usage-required-for-long-context).

  </Accordion>

  <Accordion title="AWS Bedrock est-il pris en charge ?">
    Oui. OpenClaw inclut un fournisseur **Amazon Bedrock (Converse)**. Avec des marqueurs d’environnement AWS présents, OpenClaw peut découvrir automatiquement le catalogue Bedrock streaming/texte et le fusionner comme fournisseur implicite `amazon-bedrock` ; sinon vous pouvez activer explicitement `plugins.entries.amazon-bedrock.config.discovery.enabled` ou ajouter une entrée de fournisseur manuellement. Voir [Amazon Bedrock](/fr/providers/bedrock) et [Fournisseurs de modèles](/fr/providers/models). Si vous préférez un flux de clé géré, un proxy compatible OpenAI devant Bedrock reste une option valable.
  </Accordion>

  <Accordion title="Comment fonctionne l’authentification Codex ?">
    OpenClaw prend en charge **OpenAI Code (Codex)** via OAuth (connexion ChatGPT). L’onboarding peut exécuter le flux OAuth et définira le modèle par défaut sur `openai-codex/gpt-5.4` lorsque c’est approprié. Voir [Fournisseurs de modèles](/fr/concepts/model-providers) et [Onboarding (CLI)](/fr/start/wizard).
  </Accordion>

  <Accordion title="Pourquoi ChatGPT GPT-5.4 ne débloque-t-il pas openai/gpt-5.4 dans OpenClaw ?">
    OpenClaw traite les deux chemins séparément :

    - `openai-codex/gpt-5.4` = OAuth ChatGPT/Codex
    - `openai/gpt-5.4` = API OpenAI Platform directe

    Dans OpenClaw, la connexion ChatGPT/Codex est reliée au chemin `openai-codex/*`,
    et non au chemin direct `openai/*`. Si vous voulez le chemin API direct dans
    OpenClaw, définissez `OPENAI_API_KEY` (ou la configuration de fournisseur OpenAI équivalente).
    Si vous voulez la connexion ChatGPT/Codex dans OpenClaw, utilisez `openai-codex/*`.

  </Accordion>

  <Accordion title="Pourquoi les limites OAuth Codex peuvent-elles différer du ChatGPT web ?">
    `openai-codex/*` utilise le chemin OAuth Codex, et ses fenêtres de quota utilisables sont
    gérées par OpenAI et dépendent de l’abonnement. En pratique, ces limites peuvent différer de
    l’expérience sur le site/l’application ChatGPT, même lorsque les deux sont liées au même compte.

    OpenClaw peut afficher les fenêtres de quota/usage actuellement visibles du fournisseur dans
    `openclaw models status`, mais il n’invente ni ne normalise les
    droits ChatGPT web en accès API direct. Si vous voulez le chemin direct de
    facturation/limites OpenAI Platform, utilisez `openai/*` avec une clé API.

  </Accordion>

  <Accordion title="Prenez-vous en charge l’authentification par abonnement OpenAI (Codex OAuth) ?">
    Oui. OpenClaw prend entièrement en charge **l’OAuth d’abonnement OpenAI Code (Codex)**.
    OpenAI autorise explicitement l’utilisation de l’OAuth d’abonnement dans des outils/workflows externes
    comme OpenClaw. L’onboarding peut exécuter le flux OAuth pour vous.

    Voir [OAuth](/fr/concepts/oauth), [Fournisseurs de modèles](/fr/concepts/model-providers), et [Onboarding (CLI)](/fr/start/wizard).

  </Accordion>

  <Accordion title="Comment configurer Gemini CLI OAuth ?">
    Gemini CLI utilise un **flux d’authentification par plugin**, pas un client id ou un secret dans `openclaw.json`.

    Étapes :

    1. Installez Gemini CLI localement afin que `gemini` soit sur le `PATH`
       - Homebrew : `brew install gemini-cli`
       - npm : `npm install -g @google/gemini-cli`
    2. Activez le plugin : `openclaw plugins enable google`
    3. Connectez-vous : `openclaw models auth login --provider google-gemini-cli --set-default`
    4. Modèle par défaut après connexion : `google-gemini-cli/gemini-3-flash-preview`
    5. Si les requêtes échouent, définissez `GOOGLE_CLOUD_PROJECT` ou `GOOGLE_CLOUD_PROJECT_ID` sur l’hôte de la passerelle

    Cela stocke les tokens OAuth dans des profils d’authentification sur l’hôte de la passerelle. Détails : [Fournisseurs de modèles](/fr/concepts/model-providers).

  </Accordion>

  <Accordion title="Un modèle local convient-il pour des conversations occasionnelles ?">
    En général non. OpenClaw a besoin d’un grand contexte + d’une forte sécurité ; les petites cartes tronquent et fuient. Si vous devez le faire, exécutez localement la **plus grande** version de modèle que vous pouvez (LM Studio) et consultez [/gateway/local-models](/fr/gateway/local-models). Les modèles plus petits/quantifiés augmentent le risque d’injection de prompt — voir [Sécurité](/fr/gateway/security).
  </Accordion>

  <Accordion title="Comment garder le trafic des modèles hébergés dans une région donnée ?">
    Choisissez des points de terminaison épinglés par région. OpenRouter expose des options hébergées aux États-Unis pour MiniMax, Kimi et GLM ; choisissez la variante hébergée aux États-Unis pour garder les données dans la région. Vous pouvez toujours lister Anthropic/OpenAI à côté de ceux-ci en utilisant `models.mode: "merge"` afin que les replis restent disponibles tout en respectant le fournisseur régional sélectionné.
  </Accordion>

  <Accordion title="Dois-je acheter un Mac Mini pour installer cela ?">
    Non. OpenClaw fonctionne sur macOS ou Linux (Windows via WSL2). Un Mac mini est facultatif — certaines personnes
    en achètent un comme hôte toujours allumé, mais un petit VPS, un serveur domestique ou une machine de type Raspberry Pi fonctionne aussi.

    Vous n’avez besoin d’un Mac **que pour les outils réservés à macOS**. Pour iMessage, utilisez [BlueBubbles](/fr/channels/bluebubbles) (recommandé) — le serveur BlueBubbles s’exécute sur n’importe quel Mac, et la passerelle peut tourner sur Linux ou ailleurs. Si vous voulez d’autres outils réservés à macOS, exécutez la passerelle sur un Mac ou associez un nœud macOS.

    Documentation : [BlueBubbles](/fr/channels/bluebubbles), [Nœuds](/fr/nodes), [Mode distant Mac](/fr/platforms/mac/remote).

  </Accordion>

  <Accordion title="Ai-je besoin d’un Mac mini pour la prise en charge d’iMessage ?">
    Vous avez besoin d’**un appareil macOS** connecté à Messages. Ce n’est **pas** obligatoirement un Mac mini —
    n’importe quel Mac convient. **Utilisez [BlueBubbles](/fr/channels/bluebubbles)** (recommandé) pour iMessage — le serveur BlueBubbles s’exécute sur macOS, tandis que la passerelle peut tourner sur Linux ou ailleurs.

    Configurations courantes :

    - Exécuter la passerelle sur Linux/VPS, et le serveur BlueBubbles sur n’importe quel Mac connecté à Messages.
    - Tout exécuter sur le Mac si vous voulez la configuration mono-machine la plus simple.

    Documentation : [BlueBubbles](/fr/channels/bluebubbles), [Nœuds](/fr/nodes),
    [Mode distant Mac](/fr/platforms/mac/remote).

  </Accordion>

  <Accordion title="Si j’achète un Mac mini pour faire tourner OpenClaw, puis-je le connecter à mon MacBook Pro ?">
    Oui. Le **Mac mini peut exécuter la passerelle**, et votre MacBook Pro peut se connecter comme
    **nœud** (appareil compagnon). Les nœuds n’exécutent pas la passerelle — ils fournissent des
    capacités supplémentaires comme l’écran/la caméra/le canvas et `system.run` sur cet appareil.

    Modèle courant :

    - Passerelle sur le Mac mini (toujours allumé).
    - Le MacBook Pro exécute l’application macOS ou un hôte de nœud et s’associe à la passerelle.
    - Utilisez `openclaw nodes status` / `openclaw nodes list` pour le voir.

    Documentation : [Nœuds](/fr/nodes), [CLI des nœuds](/cli/nodes).

  </Accordion>

  <Accordion title="Puis-je utiliser Bun ?">
    Bun n’est **pas recommandé**. Nous constatons des bugs d’exécution, en particulier avec WhatsApp et Telegram.
    Utilisez **Node** pour des passerelles stables.

    Si vous souhaitez quand même expérimenter avec Bun, faites-le sur une passerelle hors production
    sans WhatsApp/Telegram.

  </Accordion>

  <Accordion title="Telegram : que faut-il mettre dans allowFrom ?">
    `channels.telegram.allowFrom` correspond à **l’ID utilisateur Telegram de l’expéditeur humain** (numérique). Ce n’est pas le nom d’utilisateur du bot.

    L’onboarding accepte une saisie `@username` et la résout en ID numérique, mais l’autorisation OpenClaw utilise uniquement des ID numériques.

    Plus sûr (sans bot tiers) :

    - Envoyez un DM à votre bot, puis exécutez `openclaw logs --follow` et lisez `from.id`.

    API Bot officielle :

    - Envoyez un DM à votre bot, puis appelez `https://api.telegram.org/bot<bot_token>/getUpdates` et lisez `message.from.id`.

    Tiers (moins privé) :

    - Envoyez un DM à `@userinfobot` ou `@getidsbot`.

    Voir [/channels/telegram](/fr/channels/telegram#access-control-and-activation).

  </Accordion>

  <Accordion title="Plusieurs personnes peuvent-elles utiliser un même numéro WhatsApp avec différentes instances OpenClaw ?">
    Oui, via le **routage multi-agent**. Liez le **DM** WhatsApp de chaque expéditeur (peer `kind: "direct"`, expéditeur E.164 comme `+15551234567`) à un `agentId` différent, afin que chaque personne obtienne son propre espace de travail et son propre magasin de sessions. Les réponses proviennent toujours du **même compte WhatsApp**, et le contrôle d’accès aux DM (`channels.whatsapp.dmPolicy` / `channels.whatsapp.allowFrom`) est global par compte WhatsApp. Voir [Routage multi-agent](/fr/concepts/multi-agent) et [WhatsApp](/fr/channels/whatsapp).
  </Accordion>

  <Accordion title='Puis-je exécuter un agent "chat rapide" et un agent "Opus pour coder" ?'>
    Oui. Utilisez le routage multi-agent : donnez à chaque agent son propre modèle par défaut, puis liez les routes entrantes (compte fournisseur ou peers spécifiques) à chacun. Un exemple de configuration figure dans [Routage multi-agent](/fr/concepts/multi-agent). Voir aussi [Modèles](/fr/concepts/models) et [Configuration](/fr/gateway/configuration).
  </Accordion>

  <Accordion title="Homebrew fonctionne-t-il sur Linux ?">
    Oui. Homebrew prend en charge Linux (Linuxbrew). Configuration rapide :

    ```bash
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
    echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"' >> ~/.profile
    eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
    brew install <formula>
    ```

    Si vous exécutez OpenClaw via systemd, assurez-vous que le PATH du service inclut `/home/linuxbrew/.linuxbrew/bin` (ou votre préfixe brew) afin que les outils installés avec `brew` se résolvent dans les shells non interactifs.
    Les builds récents préfixent aussi les répertoires bin utilisateur courants sur les services systemd Linux (par exemple `~/.local/bin`, `~/.npm-global/bin`, `~/.local/share/pnpm`, `~/.bun/bin`) et respectent `PNPM_HOME`, `NPM_CONFIG_PREFIX`, `BUN_INSTALL`, `VOLTA_HOME`, `ASDF_DATA_DIR`, `NVM_DIR` et `FNM_DIR` lorsqu’ils sont définis.

  </Accordion>

  <Accordion title="Différence entre l’installation git modifiable et npm install">
    - **Installation git modifiable :** copie complète des sources, modifiable, idéale pour les contributeurs.
      Vous exécutez les builds localement et pouvez corriger le code/la documentation.
    - **npm install :** installation globale de la CLI, sans dépôt, idéale pour « l’exécuter simplement ».
      Les mises à jour proviennent des dist-tags npm.

    Documentation : [Bien démarrer](/fr/start/getting-started), [Mise à jour](/fr/install/updating).

  </Accordion>

  <Accordion title="Puis-je basculer plus tard entre les installations npm et git ?">
    Oui. Installez l’autre variante, puis exécutez Doctor pour que le service de la passerelle pointe vers le nouveau point d’entrée.
    Cela **ne supprime pas vos données** — cela change seulement l’installation du code OpenClaw. Votre état
    (`~/.openclaw`) et votre espace de travail (`~/.openclaw/workspace`) restent intacts.

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

    Doctor détecte un décalage de point d’entrée du service de la passerelle et propose de réécrire la configuration du service pour correspondre à l’installation actuelle (utilisez `--repair` en automatisation).

    Conseils de sauvegarde : voir [Stratégie de sauvegarde](#où-les-choses-sont-stockées-sur-le-disque).

  </Accordion>

  <Accordion title="Dois-je exécuter la passerelle sur mon ordinateur portable ou sur un VPS ?">
    Réponse courte : **si vous voulez une fiabilité 24/7, utilisez un VPS**. Si vous voulez la
    friction la plus faible et que les mises en veille/redémarrages ne vous dérangent pas, exécutez-la en local.

    **Ordinateur portable (passerelle locale)**

    - **Avantages :** pas de coût serveur, accès direct aux fichiers locaux, fenêtre de navigateur visible.
    - **Inconvénients :** veille/coupures réseau = déconnexions, mises à jour/redémarrages de l’OS interrompent, doit rester réveillé.

    **VPS / cloud**

    - **Avantages :** toujours allumé, réseau stable, pas de problèmes de veille d’ordinateur portable, plus facile à garder en fonctionnement.
    - **Inconvénients :** souvent headless (utilisez des captures d’écran), accès fichiers à distance uniquement, vous devez utiliser SSH pour les mises à jour.

    **Remarque spécifique à OpenClaw :** WhatsApp/Telegram/Slack/Mattermost/Discord fonctionnent tous très bien depuis un VPS. Le seul vrai compromis est **navigateur headless** contre fenêtre visible. Voir [Navigateur](/fr/tools/browser).

    **Par défaut recommandé :** VPS si vous avez déjà eu des déconnexions de passerelle. Le local est excellent quand vous utilisez activement le Mac et voulez l’accès aux fichiers locaux ou l’automatisation d’interface avec un navigateur visible.

  </Accordion>

  <Accordion title="Est-il important d’exécuter OpenClaw sur une machine dédiée ?">
    Ce n’est pas requis, mais **recommandé pour la fiabilité et l’isolation**.

    - **Hôte dédié (VPS/Mac mini/Pi) :** toujours allumé, moins d’interruptions liées à la veille/aux redémarrages, permissions plus propres, plus facile à maintenir en fonctionnement.
    - **Ordinateur portable/de bureau partagé :** tout à fait acceptable pour tester et pour une utilisation active, mais attendez-vous à des pauses lorsque la machine se met en veille ou se met à jour.

    Si vous voulez le meilleur des deux mondes, gardez la passerelle sur un hôte dédié et associez votre ordinateur portable comme **nœud** pour les outils locaux d’écran/caméra/exec. Voir [Nœuds](/fr/nodes).
    Pour les conseils de sécurité, lisez [Sécurité](/fr/gateway/security).

  </Accordion>

  <Accordion title="Quelles sont les exigences minimales pour un VPS et quel OS recommandez-vous ?">
    OpenClaw est léger. Pour une passerelle de base + un canal de chat :

    - **Minimum absolu :** 1 vCPU, 1 Go de RAM, ~500 Mo de disque.
    - **Recommandé :** 1-2 vCPU, 2 Go de RAM ou plus pour une marge (journaux, médias, plusieurs canaux). Les outils Node et l’automatisation du navigateur peuvent être gourmands en ressources.

    OS : utilisez **Ubuntu LTS** (ou n’importe quel Debian/Ubuntu moderne). Le chemin d’installation Linux y est le mieux testé.

    Documentation : [Linux](/fr/platforms/linux), [Hébergement VPS](/fr/vps).

  </Accordion>

  <Accordion title="Puis-je exécuter OpenClaw dans une VM et quelles sont les exigences ?">
    Oui. Traitez une VM comme un VPS : elle doit être toujours allumée, joignable, et avoir assez de
    RAM pour la passerelle et tous les canaux que vous activez.

    Recommandations de base :

    - **Minimum absolu :** 1 vCPU, 1 Go de RAM.
    - **Recommandé :** 2 Go de RAM ou plus si vous exécutez plusieurs canaux, l’automatisation du navigateur ou des outils média.
    - **OS :** Ubuntu LTS ou un autre Debian/Ubuntu moderne.

    Si vous êtes sous Windows, **WSL2 est la configuration de style VM la plus simple** et offre la meilleure
    compatibilité d’outillage. Voir [Windows](/fr/platforms/windows), [Hébergement VPS](/fr/vps).
    Si vous exécutez macOS dans une VM, voir [VM macOS](/fr/install/macos-vm).

  </Accordion>
</AccordionGroup>

## Qu’est-ce qu’OpenClaw ?

<AccordionGroup>
  <Accordion title="Qu’est-ce qu’OpenClaw, en un paragraphe ?">
    OpenClaw est un assistant IA personnel que vous exécutez sur vos propres appareils. Il répond sur les surfaces de messagerie que vous utilisez déjà (WhatsApp, Telegram, Slack, Mattermost, Discord, Google Chat, Signal, iMessage, WebChat, et les plugins de canal fournis comme QQ Bot) et peut aussi faire de la voix + un Canvas en direct sur les plateformes prises en charge. La **passerelle** est le plan de contrôle toujours actif ; l’assistant est le produit.
  </Accordion>

  <Accordion title="Proposition de valeur">
    OpenClaw n’est pas « juste un wrapper Claude ». C’est un **plan de contrôle local-first** qui vous permet d’exécuter un
    assistant capable sur **votre propre matériel**, accessible depuis les applications de chat que vous utilisez déjà, avec
    des sessions persistantes, de la mémoire et des outils — sans confier le contrôle de vos workflows à un
    SaaS hébergé.

    Points forts :

    - **Vos appareils, vos données :** exécutez la passerelle où vous voulez (Mac, Linux, VPS) et gardez l’espace de travail + l’historique de session en local.
    - **De vrais canaux, pas un bac à sable web :** WhatsApp/Telegram/Slack/Discord/Signal/iMessage/etc.,
      plus la voix mobile et Canvas sur les plateformes prises en charge.
    - **Indépendant du modèle :** utilisez Anthropic, OpenAI, MiniMax, OpenRouter, etc., avec routage
      et basculement par agent.
    - **Option uniquement locale :** exécutez des modèles locaux pour que **toutes les données puissent rester sur votre appareil** si vous le souhaitez.
    - **Routage multi-agent :** agents séparés par canal, compte ou tâche, chacun avec son propre
      espace de travail et ses valeurs par défaut.
    - **Open source et modifiable :** inspectez, étendez et auto-hébergez sans verrouillage fournisseur.

    Documentation : [Passerelle](/fr/gateway), [Canaux](/fr/channels), [Multi-agent](/fr/concepts/multi-agent),
    [Mémoire](/fr/concepts/memory).

  </Accordion>

  <Accordion title="Je viens de l’installer — que dois-je faire en premier ?">
    Bons premiers projets :

    - Créer un site web (WordPress, Shopify ou un simple site statique).
    - Prototyper une application mobile (plan, écrans, API).
    - Organiser des fichiers et des dossiers (nettoyage, nommage, étiquetage).
    - Connecter Gmail et automatiser des résumés ou des relances.

    Il peut gérer de grandes tâches, mais il fonctionne mieux lorsque vous les divisez en phases et
    utilisez des sous-agents pour le travail parallèle.

  </Accordion>

  <Accordion title="Quels sont les cinq principaux cas d’usage quotidiens d’OpenClaw ?">
    Les réussites du quotidien ressemblent généralement à :

    - **Briefings personnels :** résumés de votre boîte de réception, de votre calendrier et des actualités qui vous intéressent.
    - **Recherche et rédaction :** recherches rapides, résumés et premiers jets pour des e-mails ou des documents.
    - **Rappels et relances :** nudges et checklists pilotés par cron ou heartbeat.
    - **Automatisation du navigateur :** remplissage de formulaires, collecte de données et répétition de tâches web.
    - **Coordination multi-appareils :** envoyez une tâche depuis votre téléphone, laissez la passerelle l’exécuter sur un serveur, puis récupérez le résultat dans le chat.

  </Accordion>

  <Accordion title="OpenClaw peut-il aider pour la génération de leads, la prospection, les publicités et les blogs pour un SaaS ?">
    Oui pour la **recherche, la qualification et la rédaction**. Il peut analyser des sites, constituer des listes,
    résumer des prospects et rédiger des brouillons de messages de prospection ou de publicités.

    Pour la **prospection ou les campagnes publicitaires**, gardez un humain dans la boucle. Évitez le spam, respectez les lois locales et
    les politiques des plateformes, et relisez tout avant l’envoi. Le modèle le plus sûr consiste à laisser
    OpenClaw rédiger et à vous faire approuver.

    Documentation : [Sécurité](/fr/gateway/security).

  </Accordion>

  <Accordion title="Quels sont les avantages par rapport à Claude Code pour le développement web ?">
    OpenClaw est un **assistant personnel** et une couche de coordination, pas un remplacement d’IDE. Utilisez
    Claude Code ou Codex pour la boucle de codage direct la plus rapide dans un dépôt. Utilisez OpenClaw lorsque vous
    voulez une mémoire durable, un accès multi-appareils et une orchestration d’outils.

    Avantages :

    - **Mémoire persistante + espace de travail** entre les sessions
    - **Accès multiplateforme** (WhatsApp, Telegram, TUI, WebChat)
    - **Orchestration d’outils** (navigateur, fichiers, planification, hooks)
    - **Passerelle toujours active** (exécutez-la sur un VPS, interagissez de n’importe où)
    - **Nœuds** pour navigateur/écran/caméra/exec locaux

    Vitrine : [https://openclaw.ai/showcase](https://openclaw.ai/showcase)

  </Accordion>
</AccordionGroup>

## Skills et automatisation

<AccordionGroup>
  <Accordion title="Comment personnaliser les Skills sans garder le dépôt modifié ?">
    Utilisez des surcharges gérées au lieu de modifier la copie du dépôt. Placez vos changements dans `~/.openclaw/skills/<name>/SKILL.md` (ou ajoutez un dossier via `skills.load.extraDirs` dans `~/.openclaw/openclaw.json`). La priorité est `<workspace>/skills` → `<workspace>/.agents/skills` → `~/.agents/skills` → `~/.openclaw/skills` → fourni → `skills.load.extraDirs`, donc les surcharges gérées gardent la priorité sur les Skills fournis sans toucher à git. Si vous avez besoin que le Skill soit installé globalement mais visible seulement pour certains agents, gardez la copie partagée dans `~/.openclaw/skills` et contrôlez la visibilité avec `agents.defaults.skills` et `agents.list[].skills`. Seules les modifications dignes d’être remontées en amont devraient vivre dans le dépôt et sortir sous forme de PR.
  </Accordion>

  <Accordion title="Puis-je charger des Skills depuis un dossier personnalisé ?">
    Oui. Ajoutez des répertoires supplémentaires via `skills.load.extraDirs` dans `~/.openclaw/openclaw.json` (priorité la plus faible). La priorité par défaut est `<workspace>/skills` → `<workspace>/.agents/skills` → `~/.agents/skills` → `~/.openclaw/skills` → fourni → `skills.load.extraDirs`. `clawhub` installe par défaut dans `./skills`, qu’OpenClaw traite comme `<workspace>/skills` à la session suivante. Si le Skill ne doit être visible que pour certains agents, associez cela à `agents.defaults.skills` ou `agents.list[].skills`.
  </Accordion>

  <Accordion title="Comment utiliser différents modèles pour différentes tâches ?">
    Aujourd’hui, les modèles pris en charge sont :

    - **Tâches cron** : les tâches isolées peuvent définir une surcharge `model` par tâche.
    - **Sous-agents** : routez les tâches vers des agents distincts avec des modèles par défaut différents.
    - **Changement à la demande** : utilisez `/model` pour changer le modèle de la session en cours à tout moment.

    Voir [Tâches cron](/fr/automation/cron-jobs), [Routage multi-agent](/fr/concepts/multi-agent), et [Commandes slash](/fr/tools/slash-commands).

  </Accordion>

  <Accordion title="Le bot se fige pendant un travail lourd. Comment déporter cela ?">
    Utilisez des **sous-agents** pour les tâches longues ou parallèles. Les sous-agents s’exécutent dans leur propre session,
    renvoient un résumé et gardent votre chat principal réactif.

    Demandez à votre bot de « créer un sous-agent pour cette tâche » ou utilisez `/subagents`.
    Utilisez `/status` dans le chat pour voir ce que la passerelle fait en ce moment (et si elle est occupée).

    Conseil token : les tâches longues et les sous-agents consomment tous deux des tokens. Si le coût vous préoccupe, définissez un
    modèle moins cher pour les sous-agents via `agents.defaults.subagents.model`.

    Documentation : [Sous-agents](/fr/tools/subagents), [Tâches d’arrière-plan](/fr/automation/tasks).

  </Accordion>

  <Accordion title="Comment fonctionnent les sessions de sous-agent liées à un fil sur Discord ?">
    Utilisez des liaisons de fil. Vous pouvez lier un fil Discord à un sous-agent ou à une cible de session afin que les messages de suivi dans ce fil restent sur cette session liée.

    Flux de base :

    - Lancez avec `sessions_spawn` en utilisant `thread: true` (et éventuellement `mode: "session"` pour un suivi persistant).
    - Ou liez manuellement avec `/focus <target>`.
    - Utilisez `/agents` pour inspecter l’état de la liaison.
    - Utilisez `/session idle <duration|off>` et `/session max-age <duration|off>` pour contrôler le retrait automatique du focus.
    - Utilisez `/unfocus` pour détacher le fil.

    Configuration requise :

    - Valeurs globales par défaut : `session.threadBindings.enabled`, `session.threadBindings.idleHours`, `session.threadBindings.maxAgeHours`.
    - Surcharges Discord : `channels.discord.threadBindings.enabled`, `channels.discord.threadBindings.idleHours`, `channels.discord.threadBindings.maxAgeHours`.
    - Liaison automatique au lancement : définissez `channels.discord.threadBindings.spawnSubagentSessions: true`.

    Documentation : [Sous-agents](/fr/tools/subagents), [Discord](/fr/channels/discord), [Référence de configuration](/fr/gateway/configuration-reference), [Commandes slash](/fr/tools/slash-commands).

  </Accordion>

  <Accordion title="Un sous-agent a fini, mais la mise à jour d’achèvement est allée au mauvais endroit ou n’a jamais été publiée. Que vérifier ?">
    Vérifiez d’abord la route du demandeur résolue :

    - La livraison des sous-agents en mode achèvement préfère tout fil ou route de conversation lié lorsqu’il en existe un.
    - Si l’origine de l’achèvement ne contient qu’un canal, OpenClaw revient à la route stockée de la session demandeuse (`lastChannel` / `lastTo` / `lastAccountId`) afin que la livraison directe puisse encore réussir.
    - Si ni route liée ni route stockée exploitable n’existe, la livraison directe peut échouer et le résultat revient alors à une livraison en file d’attente de session au lieu d’une publication immédiate dans le chat.
    - Des cibles invalides ou périmées peuvent encore forcer un repli sur la file d’attente ou un échec final de livraison.
    - Si la dernière réponse visible de l’assistant enfant est exactement le jeton silencieux `NO_REPLY` / `no_reply`, ou exactement `ANNOUNCE_SKIP`, OpenClaw supprime volontairement l’annonce au lieu de publier un ancien progrès devenu obsolète.
    - Si l’enfant a expiré après seulement des appels d’outils, l’annonce peut condenser cela en un court résumé de progression partielle au lieu de rejouer la sortie brute des outils.

    Débogage :

    ```bash
    openclaw tasks show <runId-or-sessionKey>
    ```

    Documentation : [Sous-agents](/fr/tools/subagents), [Tâches d’arrière-plan](/fr/automation/tasks), [Outils de session](/fr/concepts/session-tool).

  </Accordion>

  <Accordion title="Cron ou les rappels ne se déclenchent pas. Que vérifier ?">
    Cron s’exécute dans le processus de la passerelle. Si la passerelle n’est pas exécutée en continu,
    les tâches planifiées ne s’exécuteront pas.

    Liste de vérification :

    - Confirmez que cron est activé (`cron.enabled`) et que `OPENCLAW_SKIP_CRON` n’est pas défini.
    - Vérifiez que la passerelle tourne 24/7 (pas de veille/redémarrage).
    - Vérifiez les paramètres de fuseau horaire de la tâche (`--tz` par rapport au fuseau de l’hôte).

    Débogage :

    ```bash
    openclaw cron run <jobId>
    openclaw cron runs --id <jobId> --limit 50
    ```

    Documentation : [Tâches cron](/fr/automation/cron-jobs), [Automatisation et tâches](/fr/automation).

  </Accordion>

  <Accordion title="Cron s’est déclenché, mais rien n’a été envoyé sur le canal. Pourquoi ?">
    Vérifiez d’abord le mode de livraison :

    - `--no-deliver` / `delivery.mode: "none"` signifie qu’aucun message externe n’est attendu.
    - Une cible d’annonce absente ou invalide (`channel` / `to`) signifie que le lanceur a ignoré la livraison sortante.
    - Les échecs d’authentification du canal (`unauthorized`, `Forbidden`) signifient que le lanceur a essayé de livrer mais que les identifiants l’en ont empêché.
    - Un résultat isolé silencieux (`NO_REPLY` / `no_reply` uniquement) est traité comme volontairement non livrable, donc le lanceur supprime aussi la livraison de repli en file d’attente.

    Pour les tâches cron isolées, c’est le lanceur qui gère la livraison finale. L’agent est censé
    renvoyer un résumé en texte brut que le lanceur pourra envoyer. `--no-deliver` conserve
    ce résultat en interne ; cela ne permet pas à l’agent d’envoyer directement avec
    l’outil message à la place.

    Débogage :

    ```bash
    openclaw cron runs --id <jobId> --limit 50
    openclaw tasks show <runId-or-sessionKey>
    ```

    Documentation : [Tâches cron](/fr/automation/cron-jobs), [Tâches d’arrière-plan](/fr/automation/tasks).

  </Accordion>

  <Accordion title="Pourquoi une exécution cron isolée a-t-elle changé de modèle ou réessayé une fois ?">
    Il s’agit généralement du chemin de changement de modèle en direct, pas d’une planification en double.

    Un cron isolé peut persister un transfert de modèle à l’exécution et réessayer lorsque l’exécution active
    lève `LiveSessionModelSwitchError`. Le nouvel essai conserve le fournisseur/modèle
    changé, et si le changement portait une nouvelle surcharge de profil d’authentification, cron
    la persiste aussi avant de réessayer.

    Règles de sélection associées :

    - La surcharge de modèle du hook Gmail gagne en premier lorsqu’elle s’applique.
    - Puis la valeur `model` propre à la tâche.
    - Puis toute surcharge de modèle de session cron stockée.
    - Puis la sélection normale du modèle agent/par défaut.

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
    Utilisez les commandes natives `openclaw skills` ou déposez les Skills dans votre espace de travail. L’interface Skills macOS n’est pas disponible sur Linux.
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

    L’installation native `openclaw skills install` écrit dans le répertoire `skills/`
    de l’espace de travail actif. Installez la CLI distincte `clawhub` seulement si vous voulez publier ou
    synchroniser vos propres Skills. Pour des installations partagées entre agents, placez le Skill sous
    `~/.openclaw/skills` et utilisez `agents.defaults.skills` ou
    `agents.list[].skills` si vous voulez limiter les agents qui peuvent le voir.

  </Accordion>

  <Accordion title="OpenClaw peut-il exécuter des tâches à intervalles réguliers ou en continu en arrière-plan ?">
    Oui. Utilisez l’ordonnanceur de la passerelle :

    - **Tâches cron** pour les tâches planifiées ou récurrentes (persistent après redémarrage).
    - **Heartbeat** pour les vérifications périodiques de la « session principale ».
    - **Tâches isolées** pour des agents autonomes qui publient des résumés ou livrent dans des chats.

    Documentation : [Tâches cron](/fr/automation/cron-jobs), [Automatisation et tâches](/fr/automation),
    [Heartbeat](/fr/gateway/heartbeat).

  </Accordion>

  <Accordion title="Puis-je exécuter depuis Linux des Skills Apple réservés à macOS ?">
    Pas directement. Les Skills macOS sont contrôlés par `metadata.openclaw.os` plus les binaires requis, et les Skills n’apparaissent dans l’invite système que lorsqu’ils sont éligibles sur **l’hôte de la passerelle**. Sur Linux, les Skills réservés à `darwin` (comme `apple-notes`, `apple-reminders`, `things-mac`) ne se chargeront pas sauf si vous contournez ce filtrage.

    Vous avez trois modèles pris en charge :

    **Option A — exécuter la passerelle sur un Mac (le plus simple).**
    Exécutez la passerelle là où les binaires macOS existent, puis connectez-vous depuis Linux en [mode distant](#gateway-ports-déjà-en-cours-d’exécution-et-mode-distant) ou via Tailscale. Les Skills se chargent normalement car l’hôte de la passerelle est macOS.

    **Option B — utiliser un nœud macOS (sans SSH).**
    Exécutez la passerelle sur Linux, associez un nœud macOS (application de barre de menu) et définissez **Node Run Commands** sur « Always Ask » ou « Always Allow » sur le Mac. OpenClaw peut traiter les Skills réservés à macOS comme éligibles lorsque les binaires requis existent sur le nœud. L’agent exécute ces Skills via l’outil `nodes`. Si vous choisissez « Always Ask », approuver « Always Allow » dans l’invite ajoute cette commande à la liste autorisée.

    **Option C — proxifier les binaires macOS via SSH (avancé).**
    Gardez la passerelle sur Linux, mais faites en sorte que les binaires CLI requis se résolvent vers des wrappers SSH qui s’exécutent sur un Mac. Ensuite, surchargez le Skill pour autoriser Linux afin qu’il reste éligible.

    1. Créez un wrapper SSH pour le binaire (exemple : `memo` pour Apple Notes) :

       ```bash
       #!/usr/bin/env bash
       set -euo pipefail
       exec ssh -T user@mac-host /opt/homebrew/bin/memo "$@"
       ```

    2. Placez le wrapper sur le `PATH` de l’hôte Linux (par exemple `~/bin/memo`).
    3. Surchargez les métadonnées du Skill (espace de travail ou `~/.openclaw/skills`) pour autoriser Linux :

       ```markdown
       ---
       name: apple-notes
       description: Manage Apple Notes via the memo CLI on macOS.
       metadata: { "openclaw": { "os": ["darwin", "linux"], "requires": { "bins": ["memo"] } } }
       ---
       ```

    4. Démarrez une nouvelle session pour rafraîchir l’instantané des Skills.

  </Accordion>

  <Accordion title="Avez-vous une intégration Notion ou HeyGen ?">
    Pas intégrée aujourd’hui.

    Options :

    - **Skill / plugin personnalisé :** le meilleur choix pour un accès API fiable (Notion/HeyGen ont tous deux des API).
    - **Automatisation du navigateur :** fonctionne sans code mais est plus lente et plus fragile.

    Si vous voulez garder le contexte par client (workflows d’agence), un modèle simple est :

    - Une page Notion par client (contexte + préférences + travail actif).
    - Demandez à l’agent de récupérer cette page au début d’une session.

    Si vous voulez une intégration native, ouvrez une demande de fonctionnalité ou créez un Skill
    ciblant ces API.

    Installer des Skills :

    ```bash
    openclaw skills install <skill-slug>
    openclaw skills update --all
    ```

    Les installations natives arrivent dans le répertoire `skills/` de l’espace de travail actif. Pour des Skills partagés entre agents, placez-les dans `~/.openclaw/skills/<name>/SKILL.md`. Si seuls certains agents doivent voir une installation partagée, configurez `agents.defaults.skills` ou `agents.list[].skills`. Certains Skills attendent des binaires installés via Homebrew ; sous Linux, cela signifie Linuxbrew (voir l’entrée FAQ Homebrew Linux ci-dessus). Voir [Skills](/fr/tools/skills), [Configuration des Skills](/fr/tools/skills-config), et [ClawHub](/fr/tools/clawhub).

  </Accordion>

  <Accordion title="Comment utiliser mon Chrome déjà connecté avec OpenClaw ?">
    Utilisez le profil de navigateur intégré `user`, qui se connecte via Chrome DevTools MCP :

    ```bash
    openclaw browser --browser-profile user tabs
    openclaw browser --browser-profile user snapshot
    ```

    Si vous voulez un nom personnalisé, créez un profil MCP explicite :

    ```bash
    openclaw browser create-profile --name chrome-live --driver existing-session
    openclaw browser --browser-profile chrome-live tabs
    ```

    Ce chemin est local à l’hôte. Si la passerelle s’exécute ailleurs, exécutez soit un hôte de nœud sur la machine du navigateur, soit utilisez plutôt un CDP distant.

    Limites actuelles sur `existing-session` / `user` :

    - les actions sont pilotées par `ref`, pas par sélecteur CSS
    - les envois de fichiers nécessitent `ref` / `inputRef` et ne prennent actuellement en charge qu’un seul fichier à la fois
    - `responsebody`, l’export PDF, l’interception de téléchargement et les actions par lot nécessitent toujours un navigateur géré ou un profil CDP brut

  </Accordion>
</AccordionGroup>

## Sandboxing et mémoire

<AccordionGroup>
  <Accordion title="Existe-t-il une documentation dédiée au sandboxing ?">
    Oui. Voir [Sandboxing](/fr/gateway/sandboxing). Pour la configuration spécifique à Docker (passerelle complète dans Docker ou images de sandbox), voir [Docker](/fr/install/docker).
  </Accordion>

  <Accordion title="Docker semble limité — comment activer toutes les fonctionnalités ?">
    L’image par défaut privilégie la sécurité et s’exécute en tant qu’utilisateur `node`, elle n’inclut donc pas
    les paquets système, Homebrew ou les navigateurs fournis. Pour une configuration plus complète :

    - Persistez `/home/node` avec `OPENCLAW_HOME_VOLUME` afin que les caches survivent.
    - Intégrez les dépendances système dans l’image avec `OPENCLAW_DOCKER_APT_PACKAGES`.
    - Installez les navigateurs Playwright via la CLI fournie :
      `node /app/node_modules/playwright-core/cli.js install chromium`
    - Définissez `PLAYWRIGHT_BROWSERS_PATH` et assurez-vous que ce chemin est persistant.

    Documentation : [Docker](/fr/install/docker), [Navigateur](/fr/tools/browser).

  </Accordion>

  <Accordion title="Puis-je garder les DM privés mais rendre les groupes publics/en sandbox avec un seul agent ?">
    Oui — si votre trafic privé est constitué de **DM** et votre trafic public de **groupes**.

    Utilisez `agents.defaults.sandbox.mode: "non-main"` pour que les sessions de groupe/canal (clés non principales) s’exécutent dans Docker, tandis que la session DM principale reste sur l’hôte. Ensuite, limitez les outils disponibles dans les sessions sandboxées via `tools.sandbox.tools`.

    Guide de configuration + exemple : [Groupes : DM personnels + groupes publics](/fr/channels/groups#pattern-personal-dms-public-groups-single-agent)

    Référence de configuration clé : [Configuration de la passerelle](/fr/gateway/configuration-reference#agentsdefaultssandbox)

  </Accordion>

  <Accordion title="Comment lier un dossier hôte dans le sandbox ?">
    Définissez `agents.defaults.sandbox.docker.binds` sur `["host:path:mode"]` (par ex. `"/home/user/src:/src:ro"`). Les liaisons globales + par agent fusionnent ; les liaisons par agent sont ignorées lorsque `scope: "shared"`. Utilisez `:ro` pour tout ce qui est sensible et rappelez-vous que les liaisons contournent les barrières du système de fichiers du sandbox.

    OpenClaw valide les sources de liaison à la fois contre le chemin normalisé et le chemin canonique résolu à travers l’ancêtre existant le plus profond. Cela signifie que les échappements via un parent symlink échouent toujours de manière fermée, même lorsque le dernier segment du chemin n’existe pas encore, et que les vérifications des racines autorisées s’appliquent toujours après résolution des symlinks.

    Voir [Sandboxing](/fr/gateway/sandboxing#custom-bind-mounts) et [Sandbox vs Tool Policy vs Elevated](/fr/gateway/sandbox-vs-tool-policy-vs-elevated#bind-mounts-security-quick-check) pour des exemples et des remarques de sécurité.

  </Accordion>

  <Accordion title="Comment fonctionne la mémoire ?">
    La mémoire d’OpenClaw n’est que des fichiers Markdown dans l’espace de travail de l’agent :

    - Notes quotidiennes dans `memory/YYYY-MM-DD.md`
    - Notes longue durée organisées dans `MEMORY.md` (sessions principales/privées uniquement)

    OpenClaw exécute également une **vidange mémoire silencieuse avant compactage** pour rappeler au modèle
    d’écrire des notes durables avant l’auto-compactage. Cela ne s’exécute que lorsque l’espace de travail
    est accessible en écriture (les sandboxes en lecture seule l’ignorent). Voir [Mémoire](/fr/concepts/memory).

  </Accordion>

  <Accordion title="La mémoire oublie constamment des choses. Comment faire pour que cela tienne ?">
    Demandez au bot **d’écrire le fait dans la mémoire**. Les notes de long terme vont dans `MEMORY.md`,
    le contexte court terme va dans `memory/YYYY-MM-DD.md`.

    C’est encore un domaine que nous améliorons. Il est utile de rappeler au modèle de stocker des souvenirs ;
    il saura quoi faire. S’il continue d’oublier, vérifiez que la passerelle utilise le même
    espace de travail à chaque exécution.

    Documentation : [Mémoire](/fr/concepts/memory), [Espace de travail d’agent](/fr/concepts/agent-workspace).

  </Accordion>

  <Accordion title="La mémoire persiste-t-elle pour toujours ? Quelles sont les limites ?">
    Les fichiers mémoire vivent sur le disque et persistent jusqu’à leur suppression. La limite est votre
    stockage, pas le modèle. Le **contexte de session** reste limité par la fenêtre de contexte du modèle,
    donc les longues conversations peuvent être compactées ou tronquées. C’est pour cela que la
    recherche en mémoire existe — elle ne remet dans le contexte que les parties pertinentes.

    Documentation : [Mémoire](/fr/concepts/memory), [Contexte](/fr/concepts/context).

  </Accordion>

  <Accordion title="La recherche sémantique en mémoire nécessite-t-elle une clé API OpenAI ?">
    Seulement si vous utilisez les **embeddings OpenAI**. Codex OAuth couvre le chat/completions et
    ne donne **pas** accès aux embeddings ; donc **se connecter avec Codex (OAuth ou la
    connexion Codex CLI)** n’aide pas pour la recherche sémantique en mémoire. Les embeddings OpenAI
    nécessitent toujours une vraie clé API (`OPENAI_API_KEY` ou `models.providers.openai.apiKey`).

    Si vous ne définissez pas explicitement de fournisseur, OpenClaw en sélectionne automatiquement un lorsqu’il
    peut résoudre une clé API (profils d’authentification, `models.providers.*.apiKey`, ou variables d’environnement).
    Il préfère OpenAI si une clé OpenAI est résolue, sinon Gemini si une clé Gemini
    est résolue, puis Voyage, puis Mistral. Si aucune clé distante n’est disponible, la
    recherche en mémoire reste désactivée jusqu’à ce que vous la configuriez. Si vous avez un chemin de modèle local
    configuré et présent, OpenClaw
    préfère `local`. Ollama est pris en charge lorsque vous définissez explicitement
    `memorySearch.provider = "ollama"`.

    Si vous préférez rester en local, définissez `memorySearch.provider = "local"` (et éventuellement
    `memorySearch.fallback = "none"`). Si vous voulez les embeddings Gemini, définissez
    `memorySearch.provider = "gemini"` et fournissez `GEMINI_API_KEY` (ou
    `memorySearch.remote.apiKey`). Nous prenons en charge les modèles d’embeddings **OpenAI, Gemini, Voyage, Mistral, Ollama ou local** — voir [Mémoire](/fr/concepts/memory) pour les détails de configuration.

  </Accordion>
</AccordionGroup>

## Où les choses sont stockées sur le disque

<AccordionGroup>
  <Accordion title="Toutes les données utilisées avec OpenClaw sont-elles enregistrées localement ?">
    Non — **l’état d’OpenClaw est local**, mais **les services externes voient toujours ce que vous leur envoyez**.

    - **Local par défaut :** les sessions, fichiers mémoire, configuration et espace de travail vivent sur l’hôte de la passerelle
      (`~/.openclaw` + votre répertoire d’espace de travail).
    - **Distant par nécessité :** les messages que vous envoyez aux fournisseurs de modèles (Anthropic/OpenAI/etc.) vont vers
      leurs API, et les plateformes de chat (WhatsApp/Telegram/Slack/etc.) stockent les données de messages sur leurs
      serveurs.
    - **Vous contrôlez l’empreinte :** l’utilisation de modèles locaux garde les prompts sur votre machine, mais le
      trafic des canaux passe toujours par les serveurs du canal.

    Voir aussi : [Espace de travail d’agent](/fr/concepts/agent-workspace), [Mémoire](/fr/concepts/memory).

  </Accordion>

  <Accordion title="Où OpenClaw stocke-t-il ses données ?">
    Tout vit sous `$OPENCLAW_STATE_DIR` (par défaut : `~/.openclaw`) :

    | Path                                                            | Purpose                                                            |
    | --------------------------------------------------------------- | ------------------------------------------------------------------ |
    | `$OPENCLAW_STATE_DIR/openclaw.json`                             | Configuration principale (JSON5)                                   |
    | `$OPENCLAW_STATE_DIR/credentials/oauth.json`                    | Import OAuth hérité (copié dans les profils d’authentification au premier usage) |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/auth-profiles.json` | Profils d’authentification (OAuth, clés API, et `keyRef`/`tokenRef` facultatifs) |
    | `$OPENCLAW_STATE_DIR/secrets.json`                              | Charge utile de secret facultative basée sur fichier pour les fournisseurs `file` de SecretRef |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/auth.json`          | Fichier de compatibilité hérité (entrées statiques `api_key` nettoyées) |
    | `$OPENCLAW_STATE_DIR/credentials/`                              | État du fournisseur (par ex. `whatsapp/<accountId>/creds.json`)    |
    | `$OPENCLAW_STATE_DIR/agents/`                                   | État par agent (agentDir + sessions)                               |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/sessions/`                | Historique et état des conversations (par agent)                   |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/sessions/sessions.json`   | Métadonnées de session (par agent)                                 |

    Chemin hérité mono-agent : `~/.openclaw/agent/*` (migré par `openclaw doctor`).

    Votre **espace de travail** (AGENTS.md, fichiers mémoire, Skills, etc.) est séparé et configuré via `agents.defaults.workspace` (par défaut : `~/.openclaw/workspace`).

  </Accordion>

  <Accordion title="Où doivent vivre AGENTS.md / SOUL.md / USER.md / MEMORY.md ?">
    Ces fichiers vivent dans l’**espace de travail de l’agent**, pas dans `~/.openclaw`.

    - **Espace de travail (par agent)** : `AGENTS.md`, `SOUL.md`, `IDENTITY.md`, `USER.md`,
      `MEMORY.md` (ou solution de repli héritée `memory.md` lorsque `MEMORY.md` est absent),
      `memory/YYYY-MM-DD.md`, `HEARTBEAT.md` facultatif.
    - **Répertoire d’état (`~/.openclaw`)** : configuration, état des canaux/fournisseurs, profils d’authentification, sessions, journaux,
      et Skills partagés (`~/.openclaw/skills`).

    L’espace de travail par défaut est `~/.openclaw/workspace`, configurable via :

    ```json5
    {
      agents: { defaults: { workspace: "~/.openclaw/workspace" } },
    }
    ```

    Si le bot « oublie » après un redémarrage, confirmez que la passerelle utilise le même
    espace de travail à chaque lancement (et rappelez-vous : le mode distant utilise l’espace de travail de **l’hôte de la passerelle**,
    pas celui de votre ordinateur portable local).

    Conseil : si vous voulez un comportement ou une préférence durable, demandez au bot de **l’écrire dans
    AGENTS.md ou MEMORY.md** plutôt que de compter sur l’historique du chat.

    Voir [Espace de travail d’agent](/fr/concepts/agent-workspace) et [Mémoire](/fr/concepts/memory).

  </Accordion>

  <Accordion title="Stratégie de sauvegarde recommandée">
    Placez votre **espace de travail d’agent** dans un dépôt git **privé** et sauvegardez-le quelque part
    de privé (par exemple GitHub privé). Cela capture la mémoire + les fichiers AGENTS/SOUL/USER
    et vous permet de restaurer plus tard « l’esprit » de l’assistant.

    Ne validez **pas** tout ce qui se trouve sous `~/.openclaw` (identifiants, sessions, tokens, ou charges utiles de secrets chiffrés).
    Si vous avez besoin d’une restauration complète, sauvegardez séparément l’espace de travail et le répertoire d’état
    (voir la question sur la migration ci-dessus).

    Documentation : [Espace de travail d’agent](/fr/concepts/agent-workspace).

  </Accordion>

  <Accordion title="Comment désinstaller complètement OpenClaw ?">
    Voir le guide dédié : [Désinstallation](/fr/install/uninstall).
  </Accordion>

  <Accordion title="Les agents peuvent-ils travailler en dehors de l’espace de travail ?">
    Oui. L’espace de travail est le **cwd par défaut** et l’ancrage mémoire, pas un sandbox strict.
    Les chemins relatifs se résolvent dans l’espace de travail, mais les chemins absolus peuvent accéder à d’autres
    emplacements de l’hôte sauf si le sandboxing est activé. Si vous avez besoin d’isolation, utilisez
    [`agents.defaults.sandbox`](/fr/gateway/sandboxing) ou les paramètres de sandbox par agent. Si vous
    voulez qu’un dépôt soit le répertoire de travail par défaut, pointez le
    `workspace` de cet agent vers la racine du dépôt. Le dépôt OpenClaw n’est que du code source ; gardez l’
    espace de travail séparé sauf si vous voulez délibérément que l’agent travaille dedans.

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

  <Accordion title="Mode distant : où se trouve le magasin de sessions ?">
    L’état des sessions appartient à l’**hôte de la passerelle**. Si vous êtes en mode distant, le magasin de sessions qui vous intéresse se trouve sur la machine distante, pas sur votre ordinateur portable local. Voir [Gestion des sessions](/fr/concepts/session).
  </Accordion>
</AccordionGroup>

## Bases de la configuration

<AccordionGroup>
  <Accordion title="Quel est le format de la configuration ? Où se trouve-t-elle ?">
    OpenClaw lit une configuration **JSON5** facultative depuis `$OPENCLAW_CONFIG_PATH` (par défaut : `~/.openclaw/openclaw.json`) :

    ```
    $OPENCLAW_CONFIG_PATH
    ```

    Si le fichier est absent, il utilise des valeurs par défaut raisonnablement sûres (y compris un espace de travail par défaut `~/.openclaw/workspace`).

  </Accordion>

  <Accordion title='J’ai défini gateway.bind: "lan" (ou "tailnet") et maintenant rien n’écoute / l’interface indique unauthorized'>
    Les liaisons non loopback **exigent un chemin d’authentification de passerelle valide**. En pratique, cela signifie :

    - authentification par secret partagé : token ou mot de passe
    - `gateway.auth.mode: "trusted-proxy"` derrière un proxy inverse conscient de l’identité, correctement configuré et non loopback

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

    - `gateway.remote.token` / `.password` n’activent **pas** à eux seuls l’authentification locale de la passerelle.
    - Les chemins d’appel locaux peuvent utiliser `gateway.remote.*` comme solution de repli seulement lorsque `gateway.auth.*` n’est pas défini.
    - Pour l’authentification par mot de passe, définissez plutôt `gateway.auth.mode: "password"` plus `gateway.auth.password` (ou `OPENCLAW_GATEWAY_PASSWORD`).
    - Si `gateway.auth.token` / `gateway.auth.password` est explicitement configuré via SecretRef et non résolu, la résolution échoue de manière fermée (pas de solution de repli distante masquant cela).
    - Les configurations Control UI à secret partagé s’authentifient via `connect.params.auth.token` ou `connect.params.auth.password` (stockés dans les paramètres de l’application/de l’UI). Les modes porteurs d’identité comme Tailscale Serve ou `trusted-proxy` utilisent à la place des en-têtes de requête. Évitez de placer des secrets partagés dans les URL.
    - Avec `gateway.auth.mode: "trusted-proxy"`, les proxies inverses loopback sur la même machine ne satisfont toujours **pas** l’authentification trusted-proxy. Le proxy de confiance doit être une source non loopback configurée.

  </Accordion>

  <Accordion title="Pourquoi ai-je besoin d’un token sur localhost maintenant ?">
    OpenClaw applique par défaut l’authentification de la passerelle, y compris sur loopback. Dans le chemin par défaut normal, cela signifie une authentification par token : si aucun chemin d’authentification explicite n’est configuré, le démarrage de la passerelle se résout en mode token et en génère un automatiquement, en l’enregistrant dans `gateway.auth.token`, donc **les clients WS locaux doivent s’authentifier**. Cela empêche d’autres processus locaux d’appeler la passerelle.

    Si vous préférez un autre chemin d’authentification, vous pouvez explicitement choisir le mode mot de passe (ou, pour les proxies inverses non loopback conscients de l’identité, `trusted-proxy`). Si vous **voulez vraiment** une loopback ouverte, définissez explicitement `gateway.auth.mode: "none"` dans votre configuration. Doctor peut générer un token pour vous à tout moment : `openclaw doctor --generate-gateway-token`.

  </Accordion>

  <Accordion title="Dois-je redémarrer après avoir modifié la configuration ?">
    La passerelle surveille la configuration et prend en charge le rechargement à chaud :

    - `gateway.reload.mode: "hybrid"` (par défaut) : applique à chaud les changements sûrs, redémarre pour les changements critiques
    - `hot`, `restart`, `off` sont également pris en charge

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

    - `off` : masque le texte du slogan mais conserve la ligne titre/version de la bannière.
    - `default` : utilise `All your chats, one OpenClaw.` à chaque fois.
    - `random` : slogans amusants/de saison en rotation (comportement par défaut).
    - Si vous ne voulez aucune bannière du tout, définissez la variable d’environnement `OPENCLAW_HIDE_BANNER=1`.

  </Accordion>

  <Accordion title="Comment activer la recherche web (et la récupération web) ?">
    `web_fetch` fonctionne sans clé API. `web_search` dépend de votre
    fournisseur sélectionné :

    - Les fournisseurs adossés à une API comme Brave, Exa, Firecrawl, Gemini, Grok, Kimi, MiniMax Search, Perplexity et Tavily exigent leur configuration normale par clé API.
    - Ollama Web Search ne nécessite pas de clé, mais utilise votre hôte Ollama configuré et nécessite `ollama signin`.
    - DuckDuckGo ne nécessite pas de clé, mais c’est une intégration HTML non officielle.
    - SearXNG est sans clé / auto-hébergé ; configurez `SEARXNG_BASE_URL` ou `plugins.entries.searxng.config.webSearch.baseUrl`.

    **Recommandé :** exécutez `openclaw configure --section web` et choisissez un fournisseur.
    Variables d’environnement possibles :

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

    La configuration de recherche web spécifique au fournisseur vit désormais sous `plugins.entries.<plugin>.config.webSearch.*`.
    Les anciens chemins de fournisseur `tools.web.search.*` se chargent encore temporairement pour compatibilité, mais ne doivent plus être utilisés pour les nouvelles configurations.
    La configuration de repli web-fetch de Firecrawl vit sous `plugins.entries.firecrawl.config.webFetch.*`.

    Remarques :

    - Si vous utilisez des listes d’autorisation, ajoutez `web_search`/`web_fetch`/`x_search` ou `group:web`.
    - `web_fetch` est activé par défaut (sauf désactivation explicite).
    - Si `tools.web.fetch.provider` est omis, OpenClaw détecte automatiquement le premier fournisseur de repli fetch prêt parmi les identifiants disponibles. Aujourd’hui, le fournisseur fourni est Firecrawl.
    - Les démons lisent les variables d’environnement depuis `~/.openclaw/.env` (ou l’environnement du service).

    Documentation : [Outils web](/fr/tools/web).

  </Accordion>

  <Accordion title="config.apply a effacé ma configuration. Comment récupérer et éviter cela ?">
    `config.apply` remplace la **configuration entière**. Si vous envoyez un objet partiel, tout le
    reste est supprimé.

    Récupération :

    - Restaurez depuis une sauvegarde (git ou une copie de `~/.openclaw/openclaw.json`).
    - Si vous n’avez pas de sauvegarde, relancez `openclaw doctor` et reconfigurez les canaux/modèles.
    - Si cela était inattendu, signalez un bug et incluez votre dernière configuration connue ou toute sauvegarde.
    - Un agent de codage local peut souvent reconstruire une configuration fonctionnelle à partir des journaux ou de l’historique.

    Pour l’éviter :

    - Utilisez `openclaw config set` pour les petits changements.
    - Utilisez `openclaw configure` pour les modifications interactives.
    - Utilisez d’abord `config.schema.lookup` si vous n’êtes pas sûr d’un chemin exact ou de la forme d’un champ ; il renvoie un nœud de schéma peu profond plus des résumés immédiats des enfants pour un approfondissement.
    - Utilisez `config.patch` pour les modifications RPC partielles ; réservez `config.apply` au remplacement complet de la configuration.
    - Si vous utilisez l’outil `gateway` réservé au propriétaire dans une exécution d’agent, il rejettera toujours les écritures dans `tools.exec.ask` / `tools.exec.security` (y compris les alias hérités `tools.bash.*` qui se normalisent vers les mêmes chemins exec protégés).

    Documentation : [Config](/cli/config), [Configurer](/cli/configure), [Doctor](/fr/gateway/doctor).

  </Accordion>

  <Accordion title="Comment exécuter une passerelle centrale avec des workers spécialisés sur plusieurs appareils ?">
    Le modèle courant est **une passerelle** (par ex. Raspberry Pi) plus des **nœuds** et des **agents** :

    - **Passerelle (centrale) :** possède les canaux (Signal/WhatsApp), le routage et les sessions.
    - **Nœuds (appareils) :** les Mac/iOS/Android se connectent comme périphériques et exposent des outils locaux (`system.run`, `canvas`, `camera`).
    - **Agents (workers) :** cerveaux/espaces de travail séparés pour des rôles spéciaux (par ex. « Hetzner ops », « Données personnelles »).
    - **Sous-agents :** lancent du travail en arrière-plan depuis un agent principal lorsque vous voulez du parallélisme.
    - **TUI :** se connecte à la passerelle et permet de changer d’agent/session.

    Documentation : [Nœuds](/fr/nodes), [Accès distant](/fr/gateway/remote), [Routage multi-agent](/fr/concepts/multi-agent), [Sous-agents](/fr/tools/subagents), [TUI](/web/tui).

  </Accordion>

  <Accordion title="Le navigateur OpenClaw peut-il fonctionner en mode headless ?">
    Oui. C’est une option de configuration :

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

    La valeur par défaut est `false` (avec fenêtre). Le mode headless est plus susceptible de déclencher des contrôles anti-bot sur certains sites. Voir [Navigateur](/fr/tools/browser).

    Le mode headless utilise le **même moteur Chromium** et fonctionne pour la plupart des automatisations (formulaires, clics, scraping, connexions). Les principales différences :

    - Pas de fenêtre de navigateur visible (utilisez des captures d’écran si vous avez besoin de visuels).
    - Certains sites sont plus stricts avec l’automatisation en mode headless (CAPTCHA, anti-bot).
      Par exemple, X/Twitter bloque souvent les sessions headless.

  </Accordion>

  <Accordion title="Comment utiliser Brave pour le contrôle du navigateur ?">
    Définissez `browser.executablePath` sur votre binaire Brave (ou tout autre navigateur basé sur Chromium) et redémarrez la passerelle.
    Voir les exemples complets de configuration dans [Navigateur](/fr/tools/browser#use-brave-or-another-chromium-based-browser).
  </Accordion>
</AccordionGroup>

## Passerelles distantes et nœuds

<AccordionGroup>
  <Accordion title="Comment les commandes se propagent-elles entre Telegram, la passerelle et les nœuds ?">
    Les messages Telegram sont gérés par la **passerelle**. La passerelle exécute l’agent et
    n’appelle les nœuds via le **WebSocket de la passerelle** que lorsqu’un outil de nœud est nécessaire :

    Telegram → Passerelle → Agent → `node.*` → Nœud → Passerelle → Telegram

    Les nœuds ne voient pas le trafic entrant du fournisseur ; ils ne reçoivent que des appels RPC de nœud.

  </Accordion>

  <Accordion title="Comment mon agent peut-il accéder à mon ordinateur si la passerelle est hébergée à distance ?">
    Réponse courte : **associez votre ordinateur comme nœud**. La passerelle s’exécute ailleurs, mais elle peut
    appeler les outils `node.*` (écran, caméra, système) sur votre machine locale via le WebSocket de la passerelle.

    Configuration type :

    1. Exécutez la passerelle sur l’hôte toujours allumé (VPS/serveur domestique).
    2. Placez l’hôte de la passerelle + votre ordinateur sur le même tailnet.
    3. Assurez-vous que le WS de la passerelle est joignable (liaison tailnet ou tunnel SSH).
    4. Ouvrez l’application macOS localement et connectez-vous en mode **Remote over SSH** (ou tailnet direct)
       pour qu’elle puisse s’enregistrer comme nœud.
    5. Approuvez le nœud sur la passerelle :

       ```bash
       openclaw devices list
       openclaw devices approve <requestId>
       ```

    Aucun pont TCP distinct n’est nécessaire ; les nœuds se connectent via le WebSocket de la passerelle.

    Rappel de sécurité : associer un nœud macOS autorise `system.run` sur cette machine. Associez uniquement
    des appareils de confiance, et consultez [Sécurité](/fr/gateway/security).

    Documentation : [Nœuds](/fr/nodes), [Protocole de la passerelle](/fr/gateway/protocol), [Mode distant macOS](/fr/platforms/mac/remote), [Sécurité](/fr/gateway/security).

  </Accordion>

  <Accordion title="Tailscale est connecté mais je n’obtiens aucune réponse. Que faire ?">
    Vérifiez les bases :

    - La passerelle fonctionne : `openclaw gateway status`
    - Santé de la passerelle : `openclaw status`
    - Santé des canaux : `openclaw channels status`

    Puis vérifiez l’authentification et le routage :

    - Si vous utilisez Tailscale Serve, assurez-vous que `gateway.auth.allowTailscale` est correctement défini.
    - Si vous vous connectez via un tunnel SSH, confirmez que le tunnel local est actif et pointe vers le bon port.
    - Confirmez que vos listes d’autorisation (DM ou groupe) incluent votre compte.

    Documentation : [Tailscale](/fr/gateway/tailscale), [Accès distant](/fr/gateway/remote), [Canaux](/fr/channels).

  </Accordion>

  <Accordion title="Deux instances OpenClaw peuvent-elles se parler (local + VPS) ?">
    Oui. Il n’y a pas de pont « bot-à-bot » intégré, mais vous pouvez le relier de plusieurs
    façons fiables :

    **Le plus simple :** utilisez un canal de chat normal auquel les deux bots peuvent accéder (Telegram/Slack/WhatsApp).
    Faites envoyer un message du Bot A au Bot B, puis laissez le Bot B répondre normalement.

    **Pont CLI (générique) :** exécutez un script qui appelle l’autre passerelle avec
    `openclaw agent --message ... --deliver`, en ciblant un chat que l’autre bot
    écoute. Si l’un des bots est sur un VPS distant, pointez votre CLI vers cette passerelle distante
    via SSH/Tailscale (voir [Accès distant](/fr/gateway/remote)).

    Exemple de modèle (à exécuter depuis une machine pouvant joindre la passerelle cible) :

    ```bash
    openclaw agent --message "Hello from local bot" --deliver --channel telegram --reply-to <chat-id>
    ```

    Conseil : ajoutez une garde-fou pour éviter que les deux bots ne bouclent indéfiniment (mention only, listes d’autorisation de canal, ou règle « ne pas répondre aux messages de bot »).

    Documentation : [Accès distant](/fr/gateway/remote), [CLI agent](/cli/agent), [Envoi agent](/fr/tools/agent-send).

  </Accordion>

  <Accordion title="Ai-je besoin de VPS séparés pour plusieurs agents ?">
    Non. Une passerelle peut héberger plusieurs agents, chacun avec son propre espace de travail, ses modèles par défaut
    et son routage. C’est la configuration normale, et c’est bien moins cher et plus simple que
    d’exécuter un VPS par agent.

    Utilisez des VPS séparés uniquement lorsque vous avez besoin d’une isolation forte (frontières de sécurité) ou de
    configurations très différentes que vous ne voulez pas partager. Sinon, gardez une seule passerelle et
    utilisez plusieurs agents ou sous-agents.

  </Accordion>

  <Accordion title="Y a-t-il un avantage à utiliser un nœud sur mon ordinateur portable personnel plutôt que SSH depuis un VPS ?">
    Oui — les nœuds sont la manière de premier niveau d’atteindre votre ordinateur portable depuis une passerelle distante, et ils
    offrent plus qu’un simple accès shell. La passerelle fonctionne sur macOS/Linux (Windows via WSL2) et est
    légère (un petit VPS ou une machine de type Raspberry Pi suffit ; 4 Go de RAM sont largement suffisants), donc une configuration
    courante est un hôte toujours allumé plus votre ordinateur portable comme nœud.

    - **Pas de SSH entrant requis.** Les nœuds se connectent vers l’extérieur au WebSocket de la passerelle et utilisent l’association d’appareil.
    - **Contrôles d’exécution plus sûrs.** `system.run` est protégé par des listes d’autorisation/approbations de nœud sur cet ordinateur portable.
    - **Plus d’outils appareil.** Les nœuds exposent `canvas`, `camera` et `screen` en plus de `system.run`.
    - **Automatisation locale du navigateur.** Gardez la passerelle sur un VPS, mais exécutez Chrome localement via un hôte de nœud sur l’ordinateur portable, ou attachez-vous à Chrome local sur l’hôte via Chrome MCP.

    SSH convient pour un accès shell ponctuel, mais les nœuds sont plus simples pour les workflows d’agent continus et
    l’automatisation des appareils.

    Documentation : [Nœuds](/fr/nodes), [CLI des nœuds](/cli/nodes), [Navigateur](/fr/tools/browser).

  </Accordion>

  <Accordion title="Les nœuds exécutent-ils un service de passerelle ?">
    Non. Une seule **passerelle** doit fonctionner par hôte sauf si vous exécutez volontairement des profils isolés (voir [Passerelles multiples](/fr/gateway/multiple-gateways)). Les nœuds sont des périphériques qui se connectent
    à la passerelle (nœuds iOS/Android, ou « mode nœud » macOS dans l’application de barre de menu). Pour les hôtes de nœuds headless
    et le contrôle en CLI, voir [CLI hôte de nœud](/cli/node).

    Un redémarrage complet est requis pour les changements `gateway`, `discovery` et `canvasHost`.

  </Accordion>

  <Accordion title="Existe-t-il un moyen API / RPC d’appliquer une configuration ?">
    Oui.

    - `config.schema.lookup` : inspecter un sous-arbre de configuration avec son nœud de schéma superficiel, l’indication UI correspondante, et les résumés immédiats des enfants avant l’écriture
    - `config.get` : récupérer l’instantané actuel + le hash
    - `config.patch` : mise à jour partielle sûre (préférée pour la plupart des modifications RPC) ; rechargement à chaud si possible et redémarrage si nécessaire
    - `config.apply` : valider + remplacer la configuration complète ; rechargement à chaud si possible et redémarrage si nécessaire
    - L’outil d’exécution `gateway` réservé au propriétaire refuse toujours de réécrire `tools.exec.ask` / `tools.exec.security` ; les alias hérités `tools.bash.*` se normalisent vers les mêmes chemins exec protégés

  </Accordion>

  <Accordion title="Configuration minimale raisonnable pour une première installation">
    ```json5
    {
      agents: { defaults: { workspace: "~/.openclaw/workspace" } },
      channels: { whatsapp: { allowFrom: ["+15555550123"] } },
    }
    ```

    Cela définit votre espace de travail et limite qui peut déclencher le bot.

  </Accordion>

  <Accordion title="Comment configurer Tailscale sur un VPS et me connecter depuis mon Mac ?">
    Étapes minimales :

    1. **Installer + se connecter sur le VPS**

       ```bash
       curl -fsSL https://tailscale.com/install.sh | sh
       sudo tailscale up
       ```

    2. **Installer + se connecter sur votre Mac**
       - Utilisez l’application Tailscale et connectez-vous au même tailnet.
    3. **Activer MagicDNS (recommandé)**
       - Dans la console d’administration Tailscale, activez MagicDNS afin que le VPS ait un nom stable.
    4. **Utiliser le nom d’hôte tailnet**
       - SSH : `ssh user@your-vps.tailnet-xxxx.ts.net`
       - WS de la passerelle : `ws://your-vps.tailnet-xxxx.ts.net:18789`

    Si vous voulez Control UI sans SSH, utilisez Tailscale Serve sur le VPS :

    ```bash
    openclaw gateway --tailscale serve
    ```

    Cela garde la passerelle liée à loopback et expose du HTTPS via Tailscale. Voir [Tailscale](/fr/gateway/tailscale).

  </Accordion>

  <Accordion title="Comment connecter un nœud Mac à une passerelle distante (Tailscale Serve) ?">
    Serve expose le **Control UI + WS** de la passerelle. Les nœuds se connectent via le même point de terminaison WS de la passerelle.

    Configuration recommandée :

    1. **Assurez-vous que le VPS + le Mac sont sur le même tailnet**.
    2. **Utilisez l’application macOS en mode Remote** (la cible SSH peut être le nom d’hôte tailnet).
       L’application tunnelisera le port de la passerelle et se connectera en tant que nœud.
    3. **Approuvez le nœud** sur la passerelle :

       ```bash
       openclaw devices list
       openclaw devices approve <requestId>
       ```

    Documentation : [Protocole de la passerelle](/fr/gateway/protocol), [Découverte](/fr/gateway/discovery), [Mode distant macOS](/fr/platforms/mac/remote).

  </Accordion>

  <Accordion title="Dois-je installer sur un deuxième ordinateur portable ou simplement ajouter un nœud ?">
    Si vous avez seulement besoin d’**outils locaux** (écran/caméra/exec) sur le deuxième ordinateur portable, ajoutez-le comme
    **nœud**. Cela conserve une seule passerelle et évite de dupliquer la configuration. Les outils de nœud locaux sont
    actuellement réservés à macOS, mais nous prévoyons de les étendre à d’autres OS.

    Installez une deuxième passerelle uniquement si vous avez besoin d’une **isolation forte** ou de deux bots complètement séparés.

    Documentation : [Nœuds](/fr/nodes), [CLI des nœuds](/cli/nodes), [Passerelles multiples](/fr/gateway/multiple-gateways).

  </Accordion>
</AccordionGroup>

## Variables d’environnement et chargement de .env

<AccordionGroup>
  <Accordion title="Comment OpenClaw charge-t-il les variables d’environnement ?">
    OpenClaw lit les variables d’environnement depuis le processus parent (shell, launchd/systemd, CI, etc.) et charge en plus :

    - `.env` depuis le répertoire de travail courant
    - un `.env` global de repli depuis `~/.openclaw/.env` (alias `$OPENCLAW_STATE_DIR/.env`)

    Aucun de ces fichiers `.env` n’écrase les variables d’environnement existantes.

    Vous pouvez aussi définir des variables d’environnement en ligne dans la configuration (appliquées seulement si absentes de l’environnement du processus) :

    ```json5
    {
      env: {
        OPENROUTER_API_KEY: "sk-or-...",
        vars: { GROQ_API_KEY: "gsk-..." },
      },
    }
    ```

    Voir [/environment](/fr/help/environment) pour l’ordre de priorité complet et les sources.

  </Accordion>

  <Accordion title="J’ai démarré la passerelle via le service et mes variables d’environnement ont disparu. Que faire ?">
    Deux correctifs courants :

    1. Placez les clés manquantes dans `~/.openclaw/.env` afin qu’elles soient prises en compte même lorsque le service n’hérite pas de l’environnement de votre shell.
    2. Activez l’import de shell (option pratique explicite) :

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

    Cela exécute votre shell de connexion et importe uniquement les clés attendues manquantes (n’écrase jamais). Variables d’environnement équivalentes :
    `OPENCLAW_LOAD_SHELL_ENV=1`, `OPENCLAW_SHELL_ENV_TIMEOUT_MS=15000`.

  </Accordion>

  <Accordion title='J’ai défini COPILOT_GITHUB_TOKEN, mais models status affiche "Shell env: off." Pourquoi ?'>
    `openclaw models status` indique si **l’import de l’environnement du shell** est activé. « Shell env: off »
    ne signifie **pas** que vos variables d’environnement sont absentes — cela signifie simplement qu’OpenClaw ne chargera pas
    automatiquement votre shell de connexion.

    Si la passerelle s’exécute comme service (launchd/systemd), elle n’héritera pas de votre environnement de shell.
    Corrigez cela de l’une de ces façons :

    1. Placez le token dans `~/.openclaw/.env` :

       ```
       COPILOT_GITHUB_TOKEN=...
       ```

    2. Ou activez l’import de shell (`env.shellEnv.enabled: true`).
    3. Ou ajoutez-le à votre bloc `env` de configuration (appliqué seulement s’il manque).

    Puis redémarrez la passerelle et revérifiez :

    ```bash
    openclaw models status
    ```

    Les tokens Copilot sont lus depuis `COPILOT_GITHUB_TOKEN` (aussi `GH_TOKEN` / `GITHUB_TOKEN`).
    Voir [/concepts/model-providers](/fr/concepts/model-providers) et [/environment](/fr/help/environment).

  </Accordion>
</AccordionGroup>

## Sessions et plusieurs chats

<AccordionGroup>
  <Accordion title="Comment démarrer une nouvelle conversation ?">
    Envoyez `/new` ou `/reset` comme message autonome. Voir [Gestion des sessions](/fr/concepts/session).
  </Accordion>

  <Accordion title="Les sessions se réinitialisent-elles automatiquement si je n’envoie jamais /new ?">
    Les sessions peuvent expirer après `session.idleMinutes`, mais cela est **désactivé par défaut** (valeur par défaut **0**).
    Définissez une valeur positive pour activer l’expiration par inactivité. Lorsqu’elle est activée, le **message suivant**
    après la période d’inactivité démarre un nouvel identifiant de session pour cette clé de chat.
    Cela ne supprime pas les transcriptions — cela démarre simplement une nouvelle session.

    ```json5
    {
      session: {
        idleMinutes: 240,
      },
    }
    ```

  </Accordion>

  <Accordion title="Existe-t-il un moyen de créer une équipe d’instances OpenClaw (un CEO et plusieurs agents) ?">
    Oui, via le **routage multi-agent** et les **sous-agents**. Vous pouvez créer un agent coordinateur
    et plusieurs agents workers avec leurs propres espaces de travail et modèles.

    Cela dit, il vaut mieux voir cela comme une **expérience amusante**. C’est lourd en tokens et souvent
    moins efficace qu’un seul bot avec des sessions séparées. Le modèle typique que nous
    envisageons est un seul bot avec lequel vous parlez, avec différentes sessions pour le travail parallèle. Ce
    bot peut aussi créer des sous-agents lorsque nécessaire.

    Documentation : [Routage multi-agent](/fr/concepts/multi-agent), [Sous-agents](/fr/tools/subagents), [CLI des agents](/cli/agents).

  </Accordion>

  <Accordion title="Pourquoi le contexte a-t-il été tronqué en plein milieu d’une tâche ? Comment l’éviter ?">
    Le contexte de session est limité par la fenêtre du modèle. Les longues discussions, les grosses sorties d’outils, ou de nombreux
    fichiers peuvent déclencher un compactage ou une troncature.

    Ce qui aide :

    - Demandez au bot de résumer l’état actuel et de l’écrire dans un fichier.
    - Utilisez `/compact` avant les longues tâches, et `/new` lorsque vous changez de sujet.
    - Gardez le contexte important dans l’espace de travail et demandez au bot de le relire.
    - Utilisez des sous-agents pour les tâches longues ou parallèles afin que le chat principal reste plus petit.
    - Choisissez un modèle avec une plus grande fenêtre de contexte si cela arrive souvent.

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

    - L’onboarding propose aussi **Reset** s’il détecte une configuration existante. Voir [Onboarding (CLI)](/fr/start/wizard).
    - Si vous utilisez des profils (`--profile` / `OPENCLAW_PROFILE`), réinitialisez chaque répertoire d’état (par défaut `~/.openclaw-<profile>`).
    - Réinitialisation dev : `openclaw gateway --dev --reset` (dev uniquement ; efface la configuration dev + les identifiants + les sessions + l’espace de travail).

  </Accordion>

  <Accordion title='Je reçois des erreurs "context too large" — comment réinitialiser ou compacter ?'>
    Utilisez l’une de ces options :

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

    Si cela continue d’arriver :

    - Activez ou ajustez **l’élagage de session** (`agents.defaults.contextPruning`) pour réduire les anciennes sorties d’outils.
    - Utilisez un modèle avec une fenêtre de contexte plus grande.

    Documentation : [Compactage](/fr/concepts/compaction), [Élagage de session](/fr/concepts/session-pruning), [Gestion des sessions](/fr/concepts/session).

  </Accordion>

  <Accordion title='Pourquoi vois-je "LLM request rejected: messages.content.tool_use.input field required" ?'>
    Il s’agit d’une erreur de validation du fournisseur : le modèle a émis un bloc `tool_use` sans
    `input` requis. Cela signifie généralement que l’historique de session est obsolète ou corrompu (souvent après de longs fils
    ou un changement d’outil/schéma).

    Correctif : démarrez une nouvelle session avec `/new` (message autonome).

  </Accordion>

  <Accordion title="Pourquoi reçois-je des messages heartbeat toutes les 30 minutes ?">
    Les heartbeats s’exécutent toutes les **30m** par défaut (**1h** en cas d’authentification OAuth). Ajustez-les ou désactivez-les :

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

    Si `HEARTBEAT.md` existe mais est en pratique vide (seulement des lignes vides et des en-têtes markdown
    comme `# Heading`), OpenClaw ignore l’exécution heartbeat pour économiser des appels API.
    Si le fichier est absent, heartbeat s’exécute quand même et le modèle décide quoi faire.

    Les surcharges par agent utilisent `agents.list[].heartbeat`. Documentation : [Heartbeat](/fr/gateway/heartbeat).

  </Accordion>

  <Accordion title='Dois-je ajouter un "compte bot" à un groupe WhatsApp ?'>
    Non. OpenClaw fonctionne sur **votre propre compte**, donc si vous êtes dans le groupe, OpenClaw peut le voir.
    Par défaut, les réponses de groupe sont bloquées jusqu’à ce que vous autorisiez des expéditeurs (`groupPolicy: "allowlist"`).

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

  <Accordion title="Comment obtenir le JID d’un groupe WhatsApp ?">
    Option 1 (la plus rapide) : suivre les journaux et envoyer un message de test dans le groupe :

    ```bash
    openclaw logs --follow --json
    ```

    Recherchez `chatId` (ou `from`) se terminant par `@g.us`, par exemple :
    `1234567890-1234567890@g.us`.

    Option 2 (si déjà configuré/autorisé) : lister les groupes depuis la configuration :

    ```bash
    openclaw directory groups list --channel whatsapp
    ```

    Documentation : [WhatsApp](/fr/channels/whatsapp), [Annuaire](/cli/directory), [Journaux](/cli/logs).

  </Accordion>

  <Accordion title="Pourquoi OpenClaw ne répond-il pas dans un groupe ?">
    Deux causes courantes :

    - Le filtrage par mention est activé (par défaut). Vous devez @mentionner le bot (ou correspondre à `mentionPatterns`).
    - Vous avez configuré `channels.whatsapp.groups` sans `"*"` et le groupe n’est pas dans la liste autorisée.

    Voir [Groupes](/fr/channels/groups) et [Messages de groupe](/fr/channels/group-messages).

  </Accordion>

  <Accordion title="Les groupes/fils partagent-ils le contexte avec les DM ?">
    Les chats directs se replient par défaut vers la session principale. Les groupes/canaux ont leurs propres clés de session, et les sujets Telegram / fils Discord sont des sessions séparées. Voir [Groupes](/fr/channels/groups) et [Messages de groupe](/fr/channels/group-messages).
  </Accordion>

  <Accordion title="Combien d’espaces de travail et d’agents puis-je créer ?">
    Il n’y a pas de limite stricte. Des dizaines (voire des centaines) fonctionnent bien, mais surveillez :

    - **Croissance du disque :** les sessions + transcriptions vivent sous `~/.openclaw/agents/<agentId>/sessions/`.
    - **Coût en tokens :** plus d’agents signifie plus d’utilisation simultanée de modèles.
    - **Charge opérationnelle :** profils d’authentification, espaces de travail et routage de canaux par agent.

    Conseils :

    - Gardez un **espace de travail actif** par agent (`agents.defaults.workspace`).
    - Élaguez les anciennes sessions (supprimez JSONL ou les entrées de magasin) si le disque grossit.
    - Utilisez `openclaw doctor` pour détecter les espaces de travail parasites et les décalages de profils.

  </Accordion>

  <Accordion title="Puis-je exécuter plusieurs bots ou chats en même temps (Slack), et comment dois-je configurer cela ?">
    Oui. Utilisez le **routage multi-agent** pour exécuter plusieurs agents isolés et router les messages entrants par
    canal/compte/peer. Slack est pris en charge comme canal et peut être lié à des agents spécifiques.

    L’accès navigateur est puissant mais pas « faire tout ce qu’un humain peut faire » — l’anti-bot, les CAPTCHA et la MFA peuvent
    encore bloquer l’automatisation. Pour le contrôle le plus fiable du navigateur, utilisez Chrome MCP local sur l’hôte,
    ou utilisez CDP sur la machine qui exécute réellement le navigateur.

    Configuration recommandée :

    - Hôte de passerelle toujours actif (VPS/Mac mini).
    - Un agent par rôle (liaisons).
    - Canal(aux) Slack liés à ces agents.
    - Navigateur local via Chrome MCP ou un nœud si nécessaire.

    Documentation : [Routage multi-agent](/fr/concepts/multi-agent), [Slack](/fr/channels/slack),
    [Navigateur](/fr/tools/browser), [Nœuds](/fr/nodes).

  </Accordion>
</AccordionGroup>

## Modèles : valeurs par défaut, sélection, alias, changement

<AccordionGroup>
  <Accordion title='Qu’est-ce que le "modèle par défaut" ?'>
    Le modèle par défaut d’OpenClaw est ce que vous définissez comme :

    ```
    agents.defaults.model.primary
    ```

    Les modèles sont référencés sous la forme `provider/model` (exemple : `openai/gpt-5.4`). Si vous omettez le fournisseur, OpenClaw essaie d’abord un alias, puis une correspondance unique parmi les fournisseurs configurés pour cet identifiant de modèle exact, et seulement ensuite revient au fournisseur par défaut configuré comme chemin de compatibilité obsolète. Si ce fournisseur n’expose plus le modèle par défaut configuré, OpenClaw revient au premier fournisseur/modèle configuré au lieu d’afficher un fournisseur par défaut supprimé devenu obsolète. Vous devriez tout de même définir **explicitement** `provider/model`.

  </Accordion>

  <Accordion title="Quel modèle recommandez-vous ?">
    **Valeur par défaut recommandée :** utilisez le modèle de dernière génération le plus puissant disponible dans votre pile de fournisseurs.
    **Pour les agents avec outils activés ou avec entrées non fiables :** privilégiez la puissance du modèle au coût.
    **Pour les discussions routinières / à faible enjeu :** utilisez des modèles de repli moins chers et routez par rôle d’agent.

    MiniMax a sa propre documentation : [MiniMax](/fr/providers/minimax) et
    [Modèles locaux](/fr/gateway/local-models).

    Règle générale : utilisez le **meilleur modèle que vous pouvez vous permettre** pour les travaux à fort enjeu, et un modèle moins cher
    pour les discussions routinières ou les résumés. Vous pouvez router les modèles par agent et utiliser des sous-agents pour
    paralléliser de longues tâches (chaque sous-agent consomme des tokens). Voir [Modèles](/fr/concepts/models) et
    [Sous-agents](/fr/tools/subagents).

    Avertissement important : les modèles plus faibles ou trop quantifiés sont plus vulnérables à l’injection de prompt
    et aux comportements dangereux. Voir [Sécurité](/fr/gateway/security).

    Plus de contexte : [Modèles](/fr/concepts/models).

  </Accordion>

  <Accordion title="Comment changer de modèle sans effacer ma configuration ?">
    Utilisez les **commandes de modèle** ou modifiez seulement les champs **model**. Évitez les remplacements complets de configuration.

    Options sûres :

    - `/model` dans le chat (rapide, par session)
    - `openclaw models set ...` (met à jour uniquement la configuration du modèle)
    - `openclaw configure --section model` (interactif)
    - modifiez `agents.defaults.model` dans `~/.openclaw/openclaw.json`

    Évitez `config.apply` avec un objet partiel sauf si vous comptez remplacer toute la configuration.
    Pour les modifications RPC, inspectez d’abord avec `config.schema.lookup` et préférez `config.patch`. La charge utile lookup vous donne le chemin normalisé, la documentation/les contraintes du schéma superficiel, et les résumés immédiats des enfants
    pour les mises à jour partielles.
    Si vous avez écrasé la configuration, restaurez-la depuis une sauvegarde ou relancez `openclaw doctor` pour réparer.

    Documentation : [Modèles](/fr/concepts/models), [Configurer](/cli/configure), [Config](/cli/config), [Doctor](/fr/gateway/doctor).

  </Accordion>

  <Accordion title="Puis-je utiliser des modèles auto-hébergés (llama.cpp, vLLM, Ollama) ?">
    Oui. Ollama est le chemin le plus simple pour les modèles locaux.

    Configuration la plus rapide :

    1. Installez Ollama depuis `https://ollama.com/download`
    2. Récupérez un modèle local comme `ollama pull gemma4`
    3. Si vous voulez aussi des modèles cloud, exécutez `ollama signin`
    4. Exécutez `openclaw onboard` et choisissez `Ollama`
    5. Choisissez `Local` ou `Cloud + Local`

    Remarques :

    - `Cloud + Local` vous donne des modèles cloud plus vos modèles Ollama locaux
    - les modèles cloud comme `kimi-k2.5:cloud` ne nécessitent pas de récupération locale
    - pour un changement manuel, utilisez `openclaw models list` et `openclaw models set ollama/<model>`

    Remarque de sécurité : les modèles plus petits ou fortement quantifiés sont plus vulnérables à l’injection de prompt.
    Nous recommandons fortement des **grands modèles** pour tout bot capable d’utiliser des outils.
    Si vous voulez quand même de petits modèles, activez le sandboxing et des listes d’autorisation d’outils strictes.

    Documentation : [Ollama](/fr/providers/ollama), [Modèles locaux](/fr/gateway/local-models),
    [Fournisseurs de modèles](/fr/concepts/model-providers), [Sécurité](/fr/gateway/security),
    [Sandboxing](/fr/gateway/sandboxing).

  </Accordion>

  <Accordion title="Quels modèles utilisent OpenClaw, Flawd et Krill ?">
    - Ces déploiements peuvent différer et évoluer avec le temps ; il n’y a pas de recommandation fixe de fournisseur.
    - Vérifiez le paramètre actuel à l’exécution sur chaque passerelle avec `openclaw models status`.
    - Pour les agents sensibles à la sécurité/avec outils activés, utilisez le modèle de dernière génération le plus puissant disponible.
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

    Vous pouvez aussi forcer un profil d’authentification spécifique pour le fournisseur (par session) :

    ```
    /model opus@anthropic:default
    /model opus@anthropic:work
    ```

    Conseil : `/model status` affiche quel agent est actif, quel fichier `auth-profiles.json` est utilisé, et quel profil d’authentification sera essayé ensuite.
    Il affiche aussi le point de terminaison du fournisseur configuré (`baseUrl`) et le mode API (`api`) lorsque disponibles.

    **Comment désépingler un profil défini avec @profile ?**

    Relancez `/model` **sans** le suffixe `@profile` :

    ```
    /model anthropic/claude-opus-4-6
    ```

    Si vous voulez revenir à la valeur par défaut, choisissez-la depuis `/model` (ou envoyez `/model <default provider/model>`).
    Utilisez `/model status` pour confirmer quel profil d’authentification est actif.

  </Accordion>

  <Accordion title="Puis-je utiliser GPT 5.2 pour les tâches quotidiennes et Codex 5.3 pour le code ?">
    Oui. Définissez-en un comme valeur par défaut et changez si nécessaire :

    - **Changement rapide (par session) :** `/model gpt-5.4` pour les tâches quotidiennes, `/model openai-codex/gpt-5.4` pour coder avec Codex OAuth.
    - **Valeur par défaut + changement :** définissez `agents.defaults.model.primary` sur `openai/gpt-5.4`, puis passez à `openai-codex/gpt-5.4` lorsque vous codez (ou l’inverse).
    - **Sous-agents :** routez les tâches de code vers des sous-agents avec un modèle par défaut différent.

    Voir [Modèles](/fr/concepts/models) et [Commandes slash](/fr/tools/slash-commands).

  </Accordion>

  <Accordion title="Comment configurer le mode rapide pour GPT 5.4 ?">
    Utilisez soit un basculement de session, soit une valeur par défaut de configuration :

    - **Par session :** envoyez `/fast on` pendant que la session utilise `openai/gpt-5.4` ou `openai-codex/gpt-5.4`.
    - **Valeur par défaut par modèle :** définissez `agents.defaults.models["openai/gpt-5.4"].params.fastMode` sur `true`.
    - **OAuth Codex aussi :** si vous utilisez aussi `openai-codex/gpt-5.4`, définissez le même indicateur là aussi.

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

    Pour OpenAI, le mode rapide correspond à `service_tier = "priority"` sur les requêtes Responses natives prises en charge. Les surcharges de session `/fast` priment sur les valeurs par défaut de configuration.

    Voir [Réflexion et mode rapide](/fr/tools/thinking) et [Mode rapide OpenAI](/fr/providers/openai#openai-fast-mode).

  </Accordion>

  <Accordion title='Pourquoi vois-je "Model ... is not allowed" puis aucune réponse ?'>
    Si `agents.defaults.models` est défini, il devient la **liste d’autorisation** pour `/model` et toutes
    les surcharges de session. Choisir un modèle qui n’est pas dans cette liste renvoie :

    ```
    Model "provider/model" is not allowed. Use /model to list available models.
    ```

    Cette erreur est renvoyée **à la place** d’une réponse normale. Correctif : ajoutez le modèle à
    `agents.defaults.models`, supprimez la liste d’autorisation, ou choisissez un modèle depuis `/model list`.

  </Accordion>

  <Accordion title='Pourquoi vois-je "Unknown model: minimax/MiniMax-M2.7" ?'>
    Cela signifie que le **fournisseur n’est pas configuré** (aucune configuration MiniMax ni aucun profil
    d’authentification MiniMax trouvé), donc le modèle ne peut pas être résolu.

    Liste de vérification pour corriger :

    1. Mettez à jour vers une version actuelle d’OpenClaw (ou exécutez depuis `main` des sources), puis redémarrez la passerelle.
    2. Assurez-vous que MiniMax est configuré (assistant ou JSON), ou que l’authentification MiniMax
       existe dans l’environnement/les profils d’authentification afin que le fournisseur correspondant puisse être injecté
       (`MINIMAX_API_KEY` pour `minimax`, `MINIMAX_OAUTH_TOKEN` ou OAuth MiniMax
       stocké pour `minimax-portal`).
    3. Utilisez l’identifiant de modèle exact (sensible à la casse) pour votre chemin d’authentification :
       `minimax/MiniMax-M2.7` ou `minimax/MiniMax-M2.7-highspeed` pour une
       configuration par clé API, ou `minimax-portal/MiniMax-M2.7` /
       `minimax-portal/MiniMax-M2.7-highspeed` pour une configuration OAuth.
    4. Exécutez :

       ```bash
       openclaw models list
       ```

       et choisissez dans la liste (ou `/model list` dans le chat).

    Voir [MiniMax](/fr/providers/minimax) et [Modèles](/fr/concepts/models).

  </Accordion>

  <Accordion title="Puis-je utiliser MiniMax par défaut et OpenAI pour les tâches complexes ?">
    Oui. Utilisez **MiniMax comme valeur par défaut** et changez de modèle **par session** lorsque nécessaire.
    Les modèles de repli sont utilisés en cas **d’erreur**, pas pour les « tâches difficiles », donc utilisez `/model` ou un agent séparé.

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

    - Agent A par défaut : MiniMax
    - Agent B par défaut : OpenAI
    - Routez par agent ou utilisez `/agent` pour changer

    Documentation : [Modèles](/fr/concepts/models), [Routage multi-agent](/fr/concepts/multi-agent), [MiniMax](/fr/providers/minimax), [OpenAI](/fr/providers/openai).

  </Accordion>

  <Accordion title="opus / sonnet / gpt sont-ils des raccourcis intégrés ?">
    Oui. OpenClaw est livré avec quelques raccourcis par défaut (appliqués seulement lorsque le modèle existe dans `agents.defaults.models`) :

    - `opus` → `anthropic/claude-opus-4-6`
    - `sonnet` → `anthropic/claude-sonnet-4-6`
    - `gpt` → `openai/gpt-5.4`
    - `gpt-mini` → `openai/gpt-5.4-mini`
    - `gpt-nano` → `open