---
read_when:
    - Modification du texte du prompt système, de la liste d’outils, ou des sections de temps/de heartbeat
    - Modification du bootstrap de l’espace de travail ou du comportement d’injection des Skills
summary: Ce que contient le prompt système d’OpenClaw et comment il est assemblé
title: Prompt système
x-i18n:
    generated_at: "2026-04-08T02:14:18Z"
    model: gpt-5.4
    provider: openai
    source_hash: e55fc886bc8ec47584d07c9e60dfacd964dc69c7db976ea373877dc4fe09a79a
    source_path: concepts/system-prompt.md
    workflow: 15
---

# Prompt système

OpenClaw crée un prompt système personnalisé pour chaque exécution d’agent. Le prompt appartient à **OpenClaw** et n’utilise pas le prompt par défaut de pi-coding-agent.

Le prompt est assemblé par OpenClaw et injecté dans chaque exécution d’agent.

Les plugins de fournisseur peuvent contribuer des consignes de prompt compatibles avec le cache sans remplacer l’intégralité du prompt appartenant à OpenClaw. Le runtime du fournisseur peut :

- remplacer un petit ensemble de sections principales nommées (`interaction_style`,
  `tool_call_style`, `execution_bias`)
- injecter un **préfixe stable** au-dessus de la limite du cache de prompt
- injecter un **suffixe dynamique** au-dessous de la limite du cache de prompt

Utilisez des contributions appartenant au fournisseur pour les ajustements spécifiques à une famille de modèles. Conservez la mutation de prompt héritée `before_prompt_build` pour la compatibilité ou pour des modifications de prompt réellement globales, pas pour le comportement normal d’un fournisseur.

## Structure

Le prompt est volontairement compact et utilise des sections fixes :

- **Outils** : rappel structuré de la source de vérité des outils, plus consignes d’exécution concernant l’utilisation des outils.
- **Sécurité** : bref rappel des garde-fous pour éviter les comportements de recherche de pouvoir ou de contournement de la supervision.
- **Skills** (quand disponibles) : indique au modèle comment charger les instructions des Skills à la demande.
- **Auto-mise à jour d’OpenClaw** : comment inspecter la configuration en toute sécurité avec
  `config.schema.lookup`, corriger la configuration avec `config.patch`, remplacer la
  configuration complète avec `config.apply`, et exécuter `update.run` uniquement sur demande explicite de l’utilisateur. L’outil `gateway`, réservé au propriétaire, refuse aussi de réécrire
  `tools.exec.ask` / `tools.exec.security`, y compris les alias hérités `tools.bash.*`
  qui sont normalisés vers ces chemins exec protégés.
- **Espace de travail** : répertoire de travail (`agents.defaults.workspace`).
- **Documentation** : chemin local vers la documentation OpenClaw (dépôt ou package npm) et quand la lire.
- **Fichiers de l’espace de travail (injectés)** : indique que les fichiers de bootstrap sont inclus ci-dessous.
- **Sandbox** (quand activée) : indique le runtime sandboxé, les chemins de la sandbox, et si l’exécution avec privilèges élevés est disponible.
- **Date et heure actuelles** : heure locale de l’utilisateur, fuseau horaire et format de l’heure.
- **Balises de réponse** : syntaxe facultative des balises de réponse pour les fournisseurs pris en charge.
- **Heartbeats** : prompt de heartbeat et comportement d’accusé de réception, lorsque les heartbeats sont activés pour l’agent par défaut.
- **Runtime** : hôte, OS, node, racine du dépôt (quand détectée), niveau de réflexion (une ligne).
- **Raisonnement** : niveau de visibilité actuel + indication de la bascule /reasoning.

La section Outils inclut aussi des consignes d’exécution pour les travaux de longue durée :

- utiliser cron pour les suivis futurs (`check back later`, rappels, travail récurrent)
  au lieu de boucles de veille `exec`, d’astuces de délai `yieldMs`, ou de sondages `process`
  répétés
- utiliser `exec` / `process` uniquement pour des commandes qui démarrent maintenant et continuent à s’exécuter
  en arrière-plan
- lorsque le réveil automatique à la fin est activé, démarrer la commande une seule fois et s’appuyer sur
  le mécanisme de réveil push lorsqu’elle produit une sortie ou échoue
- utiliser `process` pour les journaux, le statut, l’entrée, ou une intervention lorsque vous devez
  inspecter une commande en cours d’exécution
- si la tâche est plus importante, préférer `sessions_spawn` ; la fin des sous-agents est
  pilotée par push et annoncée automatiquement au demandeur
- ne pas sonder `subagents list` / `sessions_list` en boucle simplement pour attendre
  la fin

Lorsque l’outil expérimental `update_plan` est activé, la section Outils indique aussi au
modèle de ne l’utiliser que pour un travail non trivial en plusieurs étapes, de conserver exactement une étape
`in_progress`, et d’éviter de répéter l’intégralité du plan après chaque mise à jour.

Les garde-fous de sécurité dans le prompt système sont indicatifs. Ils orientent le comportement du modèle, mais n’appliquent pas la politique. Utilisez la politique des outils, les approbations d’exécution, la sandbox et les listes d’autorisation des canaux pour une application stricte ; les opérateurs peuvent les désactiver par conception.

Sur les canaux avec cartes/boutons d’approbation natifs, le prompt d’exécution indique maintenant à l’agent de
s’appuyer d’abord sur cette interface d’approbation native. Il ne doit inclure une commande manuelle
`/approve` que lorsque le résultat de l’outil indique que les approbations dans le chat ne sont pas disponibles ou que
l’approbation manuelle est la seule voie possible.

## Modes de prompt

OpenClaw peut générer des prompts système plus petits pour les sous-agents. Le runtime définit un
`promptMode` pour chaque exécution (pas une configuration visible par l’utilisateur) :

- `full` (par défaut) : inclut toutes les sections ci-dessus.
- `minimal` : utilisé pour les sous-agents ; omet **Skills**, **Rappel de mémoire**, **Auto-mise à jour d’OpenClaw**, **Alias de modèles**, **Identité de l’utilisateur**, **Balises de réponse**,
  **Messagerie**, **Réponses silencieuses**, et **Heartbeats**. Outils, **Sécurité**,
  Espace de travail, Sandbox, Date et heure actuelles (lorsqu’elles sont connues), Runtime, et le
  contexte injecté restent disponibles.
- `none` : renvoie uniquement la ligne d’identité de base.

Lorsque `promptMode=minimal`, les prompts injectés supplémentaires sont étiquetés **Contexte du sous-agent**
au lieu de **Contexte de discussion de groupe**.

## Injection du bootstrap de l’espace de travail

Les fichiers de bootstrap sont tronqués puis ajoutés sous **Contexte du projet** afin que le modèle voie l’identité et le contexte du profil sans nécessiter de lectures explicites :

- `AGENTS.md`
- `SOUL.md`
- `TOOLS.md`
- `IDENTITY.md`
- `USER.md`
- `HEARTBEAT.md`
- `BOOTSTRAP.md` (uniquement dans les espaces de travail tout juste créés)
- `MEMORY.md` lorsqu’il est présent, sinon `memory.md` comme solution de secours en minuscules

Tous ces fichiers sont **injectés dans la fenêtre de contexte** à chaque tour, sauf si
une condition spécifique à un fichier s’applique. `HEARTBEAT.md` est omis dans les exécutions normales lorsque
les heartbeats sont désactivés pour l’agent par défaut ou que
`agents.defaults.heartbeat.includeSystemPromptSection` vaut false. Gardez les fichiers
injectés concis — surtout `MEMORY.md`, qui peut grossir avec le temps et entraîner
une utilisation du contexte étonnamment élevée ainsi qu’une compaction plus fréquente.

> **Remarque :** les fichiers quotidiens `memory/*.md` ne sont **pas** injectés automatiquement. Ils
> sont consultés à la demande via les outils `memory_search` et `memory_get`, et ils
> ne comptent donc pas dans la fenêtre de contexte tant que le modèle ne les lit pas explicitement.

Les fichiers volumineux sont tronqués avec un marqueur. La taille maximale par fichier est contrôlée par
`agents.defaults.bootstrapMaxChars` (par défaut : 20000). Le contenu total du bootstrap injecté
sur l’ensemble des fichiers est plafonné par `agents.defaults.bootstrapTotalMaxChars`
(par défaut : 150000). Les fichiers manquants injectent un court marqueur de fichier manquant. En cas de troncature,
OpenClaw peut injecter un bloc d’avertissement dans le Contexte du projet ; contrôlez cela avec
`agents.defaults.bootstrapPromptTruncationWarning` (`off`, `once`, `always` ;
par défaut : `once`).

Les sessions de sous-agents n’injectent que `AGENTS.md` et `TOOLS.md` (les autres fichiers de bootstrap
sont filtrés afin de garder un petit contexte pour le sous-agent).

Les hooks internes peuvent intercepter cette étape via `agent:bootstrap` pour modifier ou remplacer
les fichiers de bootstrap injectés (par exemple en remplaçant `SOUL.md` par un persona alternatif).

Si vous souhaitez que l’agent paraisse moins générique, commencez par le
[Guide de personnalité SOUL.md](/fr/concepts/soul).

Pour inspecter la contribution de chaque fichier injecté (brut vs injecté, troncature, plus la surcharge du schéma d’outil), utilisez `/context list` ou `/context detail`. Voir [Contexte](/fr/concepts/context).

## Gestion du temps

Le prompt système inclut une section dédiée **Date et heure actuelles** lorsque le
fuseau horaire de l’utilisateur est connu. Pour garder le cache du prompt stable, il n’inclut maintenant que
le **fuseau horaire** (sans horloge dynamique ni format de l’heure).

Utilisez `session_status` lorsque l’agent a besoin de l’heure actuelle ; la carte de statut
inclut une ligne d’horodatage. Le même outil peut aussi définir un remplacement de modèle par session
(`model=default` l’efface).

Configurez avec :

- `agents.defaults.userTimezone`
- `agents.defaults.timeFormat` (`auto` | `12` | `24`)

Voir [Date & Time](/fr/date-time) pour le détail complet du comportement.

## Skills

Lorsque des Skills éligibles existent, OpenClaw injecte une **liste compacte des Skills disponibles**
(`formatSkillsForPrompt`) qui inclut le **chemin du fichier** pour chaque Skill. Le
prompt demande au modèle d’utiliser `read` pour charger le SKILL.md à l’emplacement indiqué
(espace de travail, géré, ou intégré). Si aucun Skill n’est éligible, la section
Skills est omise.

L’éligibilité inclut les conditions des métadonnées du Skill, les vérifications de l’environnement/de la configuration d’exécution,
et la liste d’autorisation effective des Skills de l’agent lorsque `agents.defaults.skills` ou
`agents.list[].skills` est configuré.

```
<available_skills>
  <skill>
    <name>...</name>
    <description>...</description>
    <location>...</location>
  </skill>
</available_skills>
```

Cela permet de garder le prompt de base compact tout en activant un usage ciblé des Skills.

## Documentation

Lorsqu’elle est disponible, le prompt système inclut une section **Documentation** qui pointe vers le
répertoire local de la documentation OpenClaw (soit `docs/` dans l’espace de travail du dépôt, soit la documentation intégrée du
package npm) et mentionne aussi le miroir public, le dépôt source, le Discord de la communauté, et
ClawHub ([https://clawhub.ai](https://clawhub.ai)) pour la découverte des Skills. Le prompt demande au modèle de consulter d’abord la documentation locale
pour le comportement, les commandes, la configuration ou l’architecture d’OpenClaw, et d’exécuter
`openclaw status` lui-même lorsque c’est possible (en ne demandant à l’utilisateur que lorsqu’il n’y a pas accès).
