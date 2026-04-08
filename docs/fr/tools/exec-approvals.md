---
read_when:
    - Configuration des approbations exec ou des listes d’autorisation
    - Implémentation de l’UX d’approbation exec dans l’app macOS
    - Examen des invites d’échappement du sandbox et de leurs implications
summary: Approbations exec, listes d’autorisation et invites d’échappement du sandbox
title: Approbations exec
x-i18n:
    generated_at: "2026-04-08T02:19:49Z"
    model: gpt-5.4
    provider: openai
    source_hash: 6041929185bab051ad873cc4822288cb7d6f0470e19e7ae7a16b70f76dfc2cd9
    source_path: tools/exec-approvals.md
    workflow: 15
---

# Approbations exec

Les approbations exec sont la **barrière de sécurité de l’app compagnon / de l’hôte de nœud** permettant à un agent sandboxé d’exécuter
des commandes sur un hôte réel (`gateway` ou `node`). Considérez cela comme un verrouillage de sécurité :
les commandes sont autorisées uniquement lorsque la politique + la liste d’autorisation + l’approbation utilisateur (facultative) sont toutes d’accord.
Les approbations exec s’ajoutent à la politique d’outils et au contrôle elevated (sauf si elevated est défini sur `full`, ce qui ignore les approbations).
La politique effective est la **plus stricte** entre `tools.exec.*` et les valeurs par défaut des approbations ; si un champ d’approbation est omis, la valeur `tools.exec` est utilisée.
L’exécution hôte utilise également l’état local des approbations sur cette machine. Un
`ask: "always"` local à l’hôte dans `~/.openclaw/exec-approvals.json` continue d’afficher des invites même si
les valeurs par défaut de la session ou de la configuration demandent `ask: "on-miss"`.
Utilisez `openclaw approvals get`, `openclaw approvals get --gateway` ou
`openclaw approvals get --node <id|name|ip>` pour inspecter la politique demandée,
les sources de politique de l’hôte et le résultat effectif.

Si l’interface de l’app compagnon n’est **pas disponible**, toute requête nécessitant une invite est
résolue par le **repli ask** (par défaut : deny).

Les clients natifs d’approbation dans les discussions peuvent également exposer des affordances spécifiques au canal sur le
message d’approbation en attente. Par exemple, Matrix peut initialiser des raccourcis par réaction sur l’invite
d’approbation (`✅` autoriser une fois, `❌` refuser et `♾️` toujours autoriser si disponible)
tout en laissant les commandes `/approve ...` dans le message comme solution de repli.

## Où cela s’applique

Les approbations exec sont appliquées localement sur l’hôte d’exécution :

- **hôte gateway** → processus `openclaw` sur la machine gateway
- **hôte node** → exécuteur de nœud (app compagnon macOS ou hôte de nœud sans interface)

Remarque sur le modèle de confiance :

- Les appelants authentifiés auprès de la gateway sont des opérateurs de confiance pour cette Gateway.
- Les nœuds appairés étendent cette capacité d’opérateur de confiance à l’hôte de nœud.
- Les approbations exec réduisent le risque d’exécution accidentelle, mais ne constituent pas une limite d’authentification par utilisateur.
- Les exécutions approuvées sur l’hôte de nœud lient le contexte d’exécution canonique : `cwd` canonique, `argv` exact, liaison de l’environnement
  lorsqu’il est présent, et chemin exécutable épinglé si applicable.
- Pour les scripts shell et les invocations directes de fichiers d’interpréteur/runtime, OpenClaw essaie également de lier
  un seul opérande de fichier local concret. Si ce fichier lié change après l’approbation mais avant l’exécution,
  l’exécution est refusée au lieu d’exécuter un contenu qui a dérivé.
- Cette liaison de fichier est volontairement réalisée au mieux, et ne constitue pas un modèle sémantique complet de chaque
  chemin de chargement d’interpréteur/runtime. Si le mode d’approbation ne peut pas identifier exactement un fichier local concret
  à lier, il refuse de créer une exécution appuyée par approbation au lieu de prétendre offrir une couverture complète.

Découpage macOS :

- Le **service hôte de nœud** transmet `system.run` à l’**app macOS** via IPC local.
- L’**app macOS** applique les approbations + exécute la commande dans le contexte de l’interface.

## Paramètres et stockage

Les approbations vivent dans un fichier JSON local sur l’hôte d’exécution :

`~/.openclaw/exec-approvals.json`

Exemple de schéma :

```json
{
  "version": 1,
  "socket": {
    "path": "~/.openclaw/exec-approvals.sock",
    "token": "base64url-token"
  },
  "defaults": {
    "security": "deny",
    "ask": "on-miss",
    "askFallback": "deny",
    "autoAllowSkills": false
  },
  "agents": {
    "main": {
      "security": "allowlist",
      "ask": "on-miss",
      "askFallback": "deny",
      "autoAllowSkills": true,
      "allowlist": [
        {
          "id": "B0C8C0B3-2C2D-4F8A-9A3C-5A4B3C2D1E0F",
          "pattern": "~/Projects/**/bin/rg",
          "lastUsedAt": 1737150000000,
          "lastUsedCommand": "rg -n TODO",
          "lastResolvedPath": "/Users/user/Projects/.../bin/rg"
        }
      ]
    }
  }
}
```

## Mode « YOLO » sans approbation

Si vous voulez que l’exécution hôte se fasse sans invites d’approbation, vous devez ouvrir **les deux** couches de politique :

- la politique exec demandée dans la configuration OpenClaw (`tools.exec.*`)
- la politique locale des approbations sur l’hôte dans `~/.openclaw/exec-approvals.json`

C’est désormais le comportement hôte par défaut sauf si vous le durcissez explicitement :

- `tools.exec.security`: `full` sur `gateway`/`node`
- `tools.exec.ask`: `off`
- hôte `askFallback`: `full`

Distinction importante :

- `tools.exec.host=auto` choisit l’emplacement d’exécution d’exec : sandbox quand disponible, sinon gateway.
- YOLO choisit la façon dont l’exécution hôte est approuvée : `security=full` plus `ask=off`.
- En mode YOLO, OpenClaw n’ajoute pas de filtre distinct d’approbation heuristique sur l’obfuscation de commande en plus de la politique d’exécution hôte configurée.
- `auto` ne transforme pas le routage vers gateway en remplacement gratuit depuis une session sandboxée. Une requête par appel `host=node` est autorisée depuis `auto`, et `host=gateway` n’est autorisé depuis `auto` que lorsqu’aucun runtime sandbox n’est actif. Si vous voulez une valeur par défaut stable non automatique, définissez `tools.exec.host` ou utilisez explicitement `/exec host=...`.

Si vous voulez une configuration plus prudente, resserrez l’une ou l’autre couche à `allowlist` / `on-miss`
ou `deny`.

Configuration persistante « ne jamais demander » sur l’hôte gateway :

```bash
openclaw config set tools.exec.host gateway
openclaw config set tools.exec.security full
openclaw config set tools.exec.ask off
openclaw gateway restart
```

Puis définissez le fichier d’approbations de l’hôte pour qu’il corresponde :

```bash
openclaw approvals set --stdin <<'EOF'
{
  version: 1,
  defaults: {
    security: "full",
    ask: "off",
    askFallback: "full"
  }
}
EOF
```

Pour un hôte node, appliquez plutôt le même fichier d’approbations sur ce nœud :

```bash
openclaw approvals set --node <id|name|ip> --stdin <<'EOF'
{
  version: 1,
  defaults: {
    security: "full",
    ask: "off",
    askFallback: "full"
  }
}
EOF
```

Raccourci limité à la session :

- `/exec security=full ask=off` modifie uniquement la session en cours.
- `/elevated full` est un raccourci de dernier recours qui ignore aussi les approbations exec pour cette session.

Si le fichier d’approbations de l’hôte reste plus strict que la configuration, la politique hôte la plus stricte continue de s’appliquer.

## Paramètres de politique

### Sécurité (`exec.security`)

- **deny** : bloque toutes les requêtes d’exécution hôte.
- **allowlist** : autorise uniquement les commandes présentes dans la liste d’autorisation.
- **full** : autorise tout (équivalent à elevated).

### Ask (`exec.ask`)

- **off** : ne jamais demander.
- **on-miss** : demander uniquement lorsqu’aucune entrée de la liste d’autorisation ne correspond.
- **always** : demander pour chaque commande.
- La confiance durable `allow-always` ne supprime pas les invites lorsque le mode ask effectif est `always`

### Repli ask (`askFallback`)

Si une invite est requise mais qu’aucune interface n’est joignable, le repli décide :

- **deny** : bloquer.
- **allowlist** : autoriser uniquement si la liste d’autorisation correspond.
- **full** : autoriser.

### Durcissement de l’évaluation en ligne par interpréteur (`tools.exec.strictInlineEval`)

Lorsque `tools.exec.strictInlineEval=true`, OpenClaw traite les formes d’évaluation de code en ligne comme nécessitant une approbation uniquement, même si le binaire d’interpréteur lui-même figure dans la liste d’autorisation.

Exemples :

- `python -c`
- `node -e`, `node --eval`, `node -p`
- `ruby -e`
- `perl -e`, `perl -E`
- `php -r`
- `lua -e`
- `osascript -e`

Il s’agit d’une défense en profondeur pour les chargeurs d’interpréteur qui ne se mappent pas proprement à un seul opérande de fichier stable. En mode strict :

- ces commandes nécessitent toujours une approbation explicite ;
- `allow-always` ne conserve pas automatiquement de nouvelles entrées de liste d’autorisation pour elles.

## Liste d’autorisation (par agent)

Les listes d’autorisation sont **par agent**. Si plusieurs agents existent, changez l’agent que vous
modifiez dans l’app macOS. Les motifs sont des **correspondances glob insensibles à la casse**.
Les motifs doivent se résoudre en **chemins binaires** (les entrées avec uniquement le nom de base sont ignorées).
Les anciennes entrées `agents.default` sont migrées vers `agents.main` au chargement.
Les chaînages shell tels que `echo ok && pwd` exigent toujours que chaque segment de premier niveau respecte les règles de la liste d’autorisation.

Exemples :

- `~/Projects/**/bin/peekaboo`
- `~/.local/bin/*`
- `/opt/homebrew/bin/rg`

Chaque entrée de la liste d’autorisation suit :

- **id** UUID stable utilisé pour l’identité dans l’interface (facultatif)
- **dernière utilisation** horodatage
- **dernière commande utilisée**
- **dernier chemin résolu**

## Auto-autoriser les CLI de Skills

Lorsque **Auto-allow skill CLIs** est activé, les exécutables référencés par des Skills connus
sont traités comme étant dans la liste d’autorisation sur les nœuds (nœud macOS ou hôte de nœud sans interface). Cela utilise
`skills.bins` via la RPC Gateway pour récupérer la liste des binaires de skill. Désactivez cette option si vous voulez des listes d’autorisation strictement manuelles.

Remarques importantes sur la confiance :

- Il s’agit d’une **liste d’autorisation implicite de commodité**, distincte des entrées de liste d’autorisation manuelle par chemin.
- Elle est destinée aux environnements d’opérateur de confiance où la Gateway et le nœud sont dans la même limite de confiance.
- Si vous exigez une confiance explicite stricte, gardez `autoAllowSkills: false` et utilisez uniquement des entrées manuelles de liste d’autorisation par chemin.

## Safe bins (stdin-only)

`tools.exec.safeBins` définit une petite liste de binaires **stdin-only** (par exemple `cut`)
pouvant s’exécuter en mode allowlist **sans** entrées explicites dans la liste d’autorisation. Les safe bins rejettent
les arguments de fichier positionnels et les jetons ressemblant à des chemins, ils ne peuvent donc agir que sur le flux entrant.
Considérez cela comme un chemin rapide étroit pour des filtres de flux, pas comme une liste de confiance générale.
N’ajoutez **pas** de binaires d’interpréteur ou de runtime (par exemple `python3`, `node`, `ruby`, `bash`, `sh`, `zsh`) à `safeBins`.
Si une commande peut évaluer du code, exécuter des sous-commandes ou lire des fichiers par conception, préférez des entrées explicites dans la liste d’autorisation et conservez les invites d’approbation activées.
Les safe bins personnalisés doivent définir un profil explicite dans `tools.exec.safeBinProfiles.<bin>`.
La validation est déterministe à partir de la seule forme de `argv` (sans vérifications de présence de fichiers sur le système hôte), ce qui
évite un comportement d’oracle d’existence de fichier via les différences entre allow et deny.
Les options orientées fichier sont refusées pour les safe bins par défaut (par exemple `sort -o`, `sort --output`,
`sort --files0-from`, `sort --compress-program`, `sort --random-source`,
`sort --temporary-directory`/`-T`, `wc --files0-from`, `jq -f/--from-file`,
`grep -f/--file`).
Les safe bins appliquent également une politique explicite de drapeaux par binaire pour les options qui cassent le comportement stdin-only
(par exemple `sort -o/--output/--compress-program` et les drapeaux récursifs de grep).
Les options longues sont validées en mode fermé en safe-bin : les drapeaux inconnus et les abréviations ambiguës sont rejetés.
Drapeaux refusés par profil de safe-bin :

[//]: # "SAFE_BIN_DENIED_FLAGS:START"

- `grep`: `--dereference-recursive`, `--directories`, `--exclude-from`, `--file`, `--recursive`, `-R`, `-d`, `-f`, `-r`
- `jq`: `--argfile`, `--from-file`, `--library-path`, `--rawfile`, `--slurpfile`, `-L`, `-f`
- `sort`: `--compress-program`, `--files0-from`, `--output`, `--random-source`, `--temporary-directory`, `-T`, `-o`
- `wc`: `--files0-from`

[//]: # "SAFE_BIN_DENIED_FLAGS:END"

Les safe bins forcent également les jetons `argv` à être traités comme du **texte littéral** au moment de l’exécution (sans globbing
et sans expansion `$VARS`) pour les segments stdin-only, de sorte que des motifs tels que `*` ou `$HOME/...` ne puissent pas être
utilisés pour dissimuler des lectures de fichiers.
Les safe bins doivent également se résoudre à partir de répertoires binaires de confiance (valeurs système par défaut plus
`tools.exec.safeBinTrustedDirs` facultatif). Les entrées `PATH` ne sont jamais automatiquement dignes de confiance.
Les répertoires de confiance safe-bin par défaut sont volontairement minimaux : `/bin`, `/usr/bin`.
Si votre exécutable safe-bin se trouve dans des chemins de gestionnaire de paquets/utilisateur (par exemple
`/opt/homebrew/bin`, `/usr/local/bin`, `/opt/local/bin`, `/snap/bin`), ajoutez-les explicitement
à `tools.exec.safeBinTrustedDirs`.
Le chaînage shell et les redirections ne sont pas automatiquement autorisés en mode allowlist.

Le chaînage shell (`&&`, `||`, `;`) est autorisé lorsque chaque segment de premier niveau satisfait la liste d’autorisation
(y compris les safe bins ou l’auto-autorisation des skills). Les redirections restent non prises en charge en mode allowlist.
La substitution de commande (`$()` / accents graves) est rejetée lors de l’analyse de la liste d’autorisation, y compris à l’intérieur
des guillemets doubles ; utilisez des guillemets simples si vous avez besoin du texte littéral `$()`.
Dans les approbations de l’app compagnon macOS, le texte shell brut contenant une syntaxe de contrôle ou d’expansion shell
(`&&`, `||`, `;`, `|`, `` ` ``, `$`, `<`, `>`, `(`, `)`) est traité comme une absence de correspondance dans la liste d’autorisation sauf
si le binaire shell lui-même figure dans la liste d’autorisation.
Pour les wrappers shell (`bash|sh|zsh ... -c/-lc`), les remplacements d’environnement limités à la requête sont réduits à une
petite liste d’autorisation explicite (`TERM`, `LANG`, `LC_*`, `COLORTERM`, `NO_COLOR`, `FORCE_COLOR`).
Pour les décisions allow-always en mode allowlist, les wrappers de distribution connus
(`env`, `nice`, `nohup`, `stdbuf`, `timeout`) conservent les chemins des exécutables internes plutôt que ceux des wrappers. Les multiplexeurs shell (`busybox`, `toybox`) sont également déballés pour les applets shell (`sh`, `ash`,
etc.) afin que les exécutables internes soient conservés plutôt que les binaires du multiplexeur. Si un wrapper ou
un multiplexeur ne peut pas être déballé en toute sécurité, aucune entrée de liste d’autorisation n’est automatiquement conservée.
Si vous mettez dans la liste d’autorisation des interpréteurs comme `python3` ou `node`, préférez `tools.exec.strictInlineEval=true` afin que l’évaluation en ligne nécessite toujours une approbation explicite. En mode strict, `allow-always` peut toujours conserver des invocations bénignes d’interpréteur/script, mais les vecteurs d’évaluation en ligne ne sont pas conservés automatiquement.

Safe bins par défaut :

[//]: # "SAFE_BIN_DEFAULTS:START"

`cut`, `uniq`, `head`, `tail`, `tr`, `wc`

[//]: # "SAFE_BIN_DEFAULTS:END"

`grep` et `sort` ne font pas partie de la liste par défaut. Si vous les activez explicitement, conservez des entrées explicites dans la liste d’autorisation pour
leurs workflows non stdin.
Pour `grep` en mode safe-bin, fournissez le motif avec `-e`/`--regexp` ; la forme de motif positionnel est
rejetée afin que des opérandes de fichier ne puissent pas être dissimulés comme positionnels ambigus.

### Safe bins versus allowlist

| Sujet            | `tools.exec.safeBins`                                  | Liste d’autorisation (`exec-approvals.json`)                 |
| ---------------- | ------------------------------------------------------ | ------------------------------------------------------------ |
| Objectif         | Autoriser automatiquement des filtres stdin étroits    | Faire explicitement confiance à des exécutables spécifiques  |
| Type de correspondance | Nom d’exécutable + politique `argv` du profil safe-bin | Motif glob du chemin exécutable résolu                  |
| Portée des arguments | Restreinte par le profil safe-bin et les règles de jetons littéraux | Correspondance sur le chemin uniquement ; les arguments restent sinon sous votre responsabilité |
| Exemples typiques | `head`, `tail`, `tr`, `wc`                             | `jq`, `python3`, `node`, `ffmpeg`, CLI personnalisées        |
| Meilleur usage   | Transformations de texte à faible risque dans des pipelines | Tout outil avec un comportement plus large ou des effets de bord |

Emplacement de la configuration :

- `safeBins` provient de la configuration (`tools.exec.safeBins` ou par agent `agents.list[].tools.exec.safeBins`).
- `safeBinTrustedDirs` provient de la configuration (`tools.exec.safeBinTrustedDirs` ou par agent `agents.list[].tools.exec.safeBinTrustedDirs`).
- `safeBinProfiles` provient de la configuration (`tools.exec.safeBinProfiles` ou par agent `agents.list[].tools.exec.safeBinProfiles`). Les clés de profil par agent remplacent les clés globales.
- les entrées de la liste d’autorisation vivent dans le fichier local à l’hôte `~/.openclaw/exec-approvals.json` sous `agents.<id>.allowlist` (ou via Control UI / `openclaw approvals allowlist ...`).
- `openclaw security audit` avertit avec `tools.exec.safe_bins_interpreter_unprofiled` lorsque des binaires d’interpréteur/runtime apparaissent dans `safeBins` sans profils explicites.
- `openclaw doctor --fix` peut générer des entrées `safeBinProfiles.<bin>` personnalisées manquantes sous forme de `{}` (relisez-les et resserrez-les ensuite). Les binaires d’interpréteur/runtime ne sont pas générés automatiquement.

Exemple de profil personnalisé :
__OC_I18N_900004__
Si vous ajoutez explicitement `jq` à `safeBins`, OpenClaw rejette toujours le builtin `env` en mode safe-bin
afin que `jq -n env` ne puisse pas vider l’environnement du processus hôte sans chemin explicite dans la liste d’autorisation
ou invite d’approbation.

## Modification dans Control UI

Utilisez la carte **Control UI → Nodes → Exec approvals** pour modifier les valeurs par défaut, les remplacements
par agent et les listes d’autorisation. Choisissez une portée (Defaults ou un agent), ajustez la politique,
ajoutez/supprimez des motifs de liste d’autorisation, puis cliquez sur **Save**. L’interface affiche les métadonnées **last used**
par motif pour vous aider à garder la liste propre.

Le sélecteur de cible choisit **Gateway** (approbations locales) ou un **Node**. Les nœuds
doivent annoncer `system.execApprovals.get/set` (app macOS ou hôte de nœud sans interface).
Si un nœud n’annonce pas encore les approbations exec, modifiez directement son
`~/.openclaw/exec-approvals.json` local.

CLI : `openclaw approvals` prend en charge la modification gateway ou node (voir [CLI des approbations](/cli/approvals)).

## Flux d’approbation

Lorsqu’une invite est requise, la gateway diffuse `exec.approval.requested` aux clients opérateur.
Control UI et l’app macOS la résolvent via `exec.approval.resolve`, puis la gateway transmet la
requête approuvée à l’hôte de nœud.

Pour `host=node`, les demandes d’approbation incluent une charge utile canonique `systemRunPlan`. La gateway utilise
ce plan comme contexte faisant autorité pour la commande/le `cwd`/la session lors de la transmission des requêtes `system.run`
approuvées.

C’est important pour la latence d’approbation asynchrone :

- le chemin d’exécution node prépare un plan canonique en amont
- l’enregistrement d’approbation stocke ce plan et ses métadonnées de liaison
- une fois approuvée, l’appel final transmis à `system.run` réutilise le plan stocké
  au lieu de faire confiance à des modifications ultérieures de l’appelant
- si l’appelant modifie `command`, `rawCommand`, `cwd`, `agentId` ou
  `sessionKey` après la création de la demande d’approbation, la gateway rejette l’exécution
  transmise comme incompatibilité d’approbation

## Commandes d’interpréteur/runtime

Les exécutions d’interpréteur/runtime appuyées par approbation sont volontairement prudentes :

- Le contexte exact `argv`/`cwd`/`env` est toujours lié.
- Les formes de script shell direct et de fichier runtime direct sont liées au mieux à un instantané concret d’un seul
  fichier local.
- Les formes courantes de wrapper de gestionnaire de paquets qui se résolvent toujours en un seul fichier local direct (par exemple
  `pnpm exec`, `pnpm node`, `npm exec`, `npx`) sont déballées avant la liaison.
- Si OpenClaw ne peut pas identifier exactement un fichier local concret pour une commande d’interpréteur/runtime
  (par exemple scripts de paquet, formes d’évaluation, chaînes de chargement propres au runtime ou formes ambiguës
  à plusieurs fichiers), l’exécution appuyée par approbation est refusée au lieu de prétendre à une couverture sémantique
  qu’elle n’a pas.
- Pour ces workflows, préférez le sandboxing, une limite d’hôte séparée ou un workflow explicite de confiance
  via allowlist/full où l’opérateur accepte la sémantique plus large du runtime.

Lorsque des approbations sont requises, l’outil exec renvoie immédiatement avec un id d’approbation. Utilisez cet id pour
corréler les événements système ultérieurs (`Exec finished` / `Exec denied`). Si aucune décision n’arrive avant le
délai d’attente, la requête est traitée comme un timeout d’approbation et exposée comme motif de refus.

### Comportement de livraison du suivi

Après la fin d’une exécution asynchrone approuvée, OpenClaw envoie un tour `agent` de suivi à la même session.

- Si une cible de livraison externe valide existe (canal livrable plus cible `to`), la livraison du suivi utilise ce canal.
- Dans les flux webchat uniquement ou les sessions internes sans cible externe, la livraison du suivi reste limitée à la session (`deliver: false`).
- Si un appelant demande explicitement une livraison externe stricte sans canal externe résoluble, la requête échoue avec `INVALID_REQUEST`.
- Si `bestEffortDeliver` est activé et qu’aucun canal externe ne peut être résolu, la livraison est rétrogradée vers une livraison limitée à la session au lieu d’échouer.

La boîte de dialogue de confirmation inclut :

- commande + arguments
- `cwd`
- id d’agent
- chemin exécutable résolu
- métadonnées d’hôte + de politique

Actions :

- **Allow once** → exécuter maintenant
- **Always allow** → ajouter à la liste d’autorisation + exécuter
- **Deny** → bloquer

## Transfert des approbations vers les canaux de discussion

Vous pouvez transférer les invites d’approbation exec vers n’importe quel canal de discussion (y compris les canaux de plugin) et les approuver
avec `/approve`. Cela utilise le pipeline normal de livraison sortante.

Configuration :
__OC_I18N_900005__
Répondez dans la discussion :
__OC_I18N_900006__
La commande `/approve` gère à la fois les approbations exec et les approbations de plugin. Si l’ID ne correspond pas à une approbation exec en attente, elle vérifie automatiquement les approbations de plugin à la place.

### Transfert des approbations de plugin

Le transfert des approbations de plugin utilise le même pipeline de livraison que les approbations exec, mais dispose de sa
propre configuration indépendante sous `approvals.plugin`. Activer ou désactiver l’un n’affecte pas l’autre.
__OC_I18N_900007__
La forme de configuration est identique à `approvals.exec` : `enabled`, `mode`, `agentFilter`,
`sessionFilter` et `targets` fonctionnent de la même manière.

Les canaux qui prennent en charge les réponses interactives partagées affichent les mêmes boutons d’approbation pour les approbations exec et
de plugin. Les canaux sans interface interactive partagée reviennent à du texte brut avec des instructions `/approve`.

### Approbations dans la même discussion sur n’importe quel canal

Lorsqu’une demande d’approbation exec ou de plugin provient d’une surface de discussion livrable, cette même discussion
peut désormais l’approuver avec `/approve` par défaut. Cela s’applique à des canaux tels que Slack, Matrix et
Microsoft Teams en plus des flux existants de l’interface Web et de l’interface terminal.

Ce chemin partagé par commande texte utilise le modèle d’authentification normal du canal pour cette conversation. Si la
discussion d’origine peut déjà envoyer des commandes et recevoir des réponses, les demandes d’approbation n’ont plus besoin d’un
adaptateur distinct de livraison native simplement pour rester en attente.

Discord et Telegram prennent également en charge `/approve` dans la même discussion, mais ces canaux utilisent toujours leur
liste résolue d’approbateurs pour l’autorisation même lorsque la livraison native des approbations est désactivée.

Pour Telegram et les autres clients d’approbation native qui appellent directement la Gateway,
ce repli est volontairement limité aux échecs « approval not found ». Un vrai
refus/une vraie erreur d’approbation exec ne fait pas silencieusement l’objet d’une nouvelle tentative comme approbation de plugin.

### Livraison native des approbations

Certains canaux peuvent aussi agir comme clients natifs d’approbation. Les clients natifs ajoutent les DM des approbateurs, la
diffusion vers la discussion d’origine et une UX interactive d’approbation spécifique au canal en plus du
flux partagé `/approve` dans la même discussion.

Lorsque des cartes/boutons d’approbation native sont disponibles, cette interface native est le
chemin principal côté agent. L’agent ne doit pas également faire écho à une commande de discussion
`/approve` en double sauf si le résultat de l’outil indique que les approbations par discussion ne sont pas disponibles ou que
l’approbation manuelle est le seul chemin restant.

Modèle générique :

- la politique d’exécution hôte décide toujours si une approbation exec est requise
- `approvals.exec` contrôle le transfert des invites d’approbation vers d’autres destinations de discussion
- `channels.<channel>.execApprovals` contrôle si ce canal agit comme client natif d’approbation

Les clients natifs d’approbation activent automatiquement la livraison DM-first lorsque toutes ces conditions sont vraies :

- le canal prend en charge la livraison native des approbations
- les approbateurs peuvent être résolus à partir de `execApprovals.approvers` explicite ou des sources de repli documentées
  pour ce canal
- `channels.<channel>.execApprovals.enabled` n’est pas défini ou vaut `"auto"`

Définissez `enabled: false` pour désactiver explicitement un client natif d’approbation. Définissez `enabled: true` pour
le forcer lorsque les approbateurs sont résolus. La livraison publique vers la discussion d’origine reste explicite via
`channels.<channel>.execApprovals.target`.

FAQ : [Pourquoi y a-t-il deux configurations d’approbation exec pour les approbations de discussion ?](/help/faq#why-are-there-two-exec-approval-configs-for-chat-approvals)

- Discord : `channels.discord.execApprovals.*`
- Slack : `channels.slack.execApprovals.*`
- Telegram : `channels.telegram.execApprovals.*`

Ces clients natifs d’approbation ajoutent le routage DM et la diffusion facultative vers un canal au flux partagé
`/approve` dans la même discussion et aux boutons d’approbation partagés.

Comportement partagé :

- Slack, Matrix, Microsoft Teams et autres discussions livrables similaires utilisent le modèle d’authentification normal du canal
  pour `/approve` dans la même discussion
- lorsqu’un client natif d’approbation s’active automatiquement, la cible de livraison native par défaut est les DM des approbateurs
- pour Discord et Telegram, seuls les approbateurs résolus peuvent approuver ou refuser
- les approbateurs Discord peuvent être explicites (`execApprovals.approvers`) ou déduits de `commands.ownerAllowFrom`
- les approbateurs Telegram peuvent être explicites (`execApprovals.approvers`) ou déduits de la configuration propriétaire existante (`allowFrom`, plus `defaultTo` en message direct lorsque pris en charge)
- les approbateurs Slack peuvent être explicites (`execApprovals.approvers`) ou déduits de `commands.ownerAllowFrom`
- les boutons natifs Slack préservent le type d’id d’approbation, de sorte que les ids `plugin:` puissent résoudre les approbations de plugin
  sans seconde couche de repli locale à Slack
- le routage DM/canal natif de Matrix et ses raccourcis par réaction gèrent à la fois les approbations exec et plugin ;
  l’autorisation de plugin continue cependant de provenir de `channels.matrix.dm.allowFrom`
- le demandeur n’a pas besoin d’être un approbateur
- la discussion d’origine peut approuver directement avec `/approve` lorsque cette discussion prend déjà en charge les commandes et les réponses
- les boutons natifs d’approbation Discord routent selon le type d’id d’approbation : les ids `plugin:` vont
  directement vers les approbations de plugin, tout le reste va vers les approbations exec
- les boutons natifs Telegram suivent le même repli borné d’exec vers plugin que `/approve`
- lorsque `target` natif active la livraison vers la discussion d’origine, les invites d’approbation incluent le texte de la commande
- les approbations exec en attente expirent après 30 minutes par défaut
- si aucune interface opérateur ou aucun client d’approbation configuré ne peut accepter la requête, l’invite revient à `askFallback`

Telegram utilise par défaut les DM des approbateurs (`target: "dm"`). Vous pouvez passer à `channel` ou `both` lorsque vous
voulez que les invites d’approbation apparaissent aussi dans la discussion/le sujet Telegram d’origine. Pour les sujets de forum Telegram,
OpenClaw préserve le sujet pour l’invite d’approbation et pour le suivi post-approbation.

Voir :

- [Discord](/channels/discord)
- [Telegram](/channels/telegram)

### Flux IPC macOS
__OC_I18N_900008__
Remarques de sécurité :

- Socket Unix en mode `0600`, jeton stocké dans `exec-approvals.json`.
- Vérification du pair de même UID.
- Challenge/response (nonce + jeton HMAC + hash de requête) + TTL court.

## Événements système

Le cycle de vie exec est exposé sous forme de messages système :

- `Exec running` (uniquement si la commande dépasse le seuil de notification d’exécution)
- `Exec finished`
- `Exec denied`

Ils sont publiés dans la session de l’agent après que le nœud a signalé l’événement.
Les approbations exec sur l’hôte gateway émettent les mêmes événements de cycle de vie lorsque la commande se termine (et éventuellement lorsqu’elle s’exécute plus longtemps que le seuil).
Les execs contrôlés par approbation réutilisent l’id d’approbation comme `runId` dans ces messages pour faciliter la corrélation.

## Comportement lors d’une approbation refusée

Lorsqu’une approbation exec asynchrone est refusée, OpenClaw empêche l’agent de réutiliser
la sortie d’une exécution antérieure de la même commande dans la session. Le motif du refus
est transmis avec des indications explicites disant qu’aucune sortie de commande n’est disponible, ce qui empêche
l’agent d’affirmer qu’il y a une nouvelle sortie ou de répéter la commande refusée avec
des résultats obsolètes issus d’une exécution antérieure réussie.

## Implications

- **full** est puissant ; préférez les listes d’autorisation lorsque possible.
- **ask** vous garde dans la boucle tout en permettant des approbations rapides.
- Les listes d’autorisation par agent empêchent qu’une approbation d’un agent ne fuit vers d’autres.
- Les approbations s’appliquent uniquement aux requêtes d’exécution hôte provenant d’**expéditeurs autorisés**. Les expéditeurs non autorisés ne peuvent pas émettre `/exec`.
- `/exec security=full` est une commodité au niveau de la session pour les opérateurs autorisés et ignore volontairement les approbations.
  Pour bloquer strictement l’exécution hôte, définissez la sécurité des approbations sur `deny` ou refusez l’outil `exec` via la politique d’outils.

Liens connexes :

- [Outil Exec](/fr/tools/exec)
- [Mode Elevated](/fr/tools/elevated)
- [Skills](/fr/tools/skills)

## Liens connexes

- [Exec](/fr/tools/exec) — outil d’exécution de commandes shell
- [Sandboxing](/fr/gateway/sandboxing) — modes sandbox et accès à l’espace de travail
- [Sécurité](/fr/gateway/security) — modèle de sécurité et durcissement
- [Sandbox vs Tool Policy vs Elevated](/fr/gateway/sandbox-vs-tool-policy-vs-elevated) — quand utiliser chacun
