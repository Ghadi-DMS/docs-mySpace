---
read_when:
    - Modification du texte du prompt système, de la liste des outils ou des sections de l’heure/Heartbeat
    - Modification du comportement d’amorçage de l’espace de travail ou d’injection des Skills
summary: Ce que contient le prompt système d’OpenClaw et comment il est assemblé
title: Prompt système
x-i18n:
    generated_at: "2026-04-18T06:43:37Z"
    model: gpt-5.4
    provider: openai
    source_hash: e60705994cebdd9768926168cb1c6d17ab717d7ff02353a5d5e7478ba8191cab
    source_path: concepts/system-prompt.md
    workflow: 15
---

# Prompt système

OpenClaw construit un prompt système personnalisé pour chaque exécution d’agent. Le prompt appartient à **OpenClaw** et n’utilise pas le prompt par défaut de pi-coding-agent.

Le prompt est assemblé par OpenClaw et injecté dans chaque exécution d’agent.

Les plugins de fournisseur peuvent contribuer avec des indications de prompt conscientes du cache sans remplacer l’intégralité du prompt appartenant à OpenClaw. L’environnement d’exécution du fournisseur peut :

- remplacer un petit ensemble de sections principales nommées (`interaction_style`,
  `tool_call_style`, `execution_bias`)
- injecter un **préfixe stable** au-dessus de la limite de cache du prompt
- injecter un **suffixe dynamique** en dessous de la limite de cache du prompt

Utilisez les contributions appartenant au fournisseur pour l’ajustement spécifique à une famille de modèles. Conservez la mutation de prompt héritée `before_prompt_build` pour la compatibilité ou pour de véritables changements globaux du prompt, pas pour le comportement normal d’un fournisseur.

## Structure

Le prompt est intentionnellement compact et utilise des sections fixes :

- **Tooling** : rappel structuré de la source de vérité des outils, plus indications d’exécution sur l’utilisation des outils.
- **Safety** : bref rappel de garde-fous pour éviter les comportements de recherche de pouvoir ou de contournement de la supervision.
- **Skills** (quand disponibles) : indique au modèle comment charger à la demande les instructions de Skills.
- **OpenClaw Self-Update** : comment inspecter la configuration en toute sécurité avec
  `config.schema.lookup`, modifier la configuration avec `config.patch`, remplacer la configuration complète avec `config.apply`, et exécuter `update.run` uniquement à la demande explicite de l’utilisateur. L’outil `gateway`, réservé au propriétaire, refuse aussi de réécrire
  `tools.exec.ask` / `tools.exec.security`, y compris les alias hérités `tools.bash.*`
  qui se normalisent vers ces chemins exec protégés.
- **Workspace** : répertoire de travail (`agents.defaults.workspace`).
- **Documentation** : chemin local vers la documentation OpenClaw (dépôt ou package npm) et quand la lire.
- **Workspace Files (injected)** : indique que des fichiers d’amorçage sont inclus ci-dessous.
- **Sandbox** (quand activé) : indique l’environnement d’exécution sandboxé, les chemins sandbox, et si une exécution avec privilèges élevés est disponible.
- **Current Date & Time** : heure locale de l’utilisateur, fuseau horaire et format d’heure.
- **Reply Tags** : syntaxe facultative des balises de réponse pour les fournisseurs pris en charge.
- **Heartbeats** : prompt Heartbeat et comportement d’accusé de réception, lorsque les Heartbeats sont activés pour l’agent par défaut.
- **Runtime** : hôte, OS, node, modèle, racine du dépôt (quand détectée), niveau de réflexion (une ligne).
- **Reasoning** : niveau de visibilité actuel + indication de bascule `/reasoning`.

La section Tooling inclut également des indications d’exécution pour le travail de longue durée :

- utiliser Cron pour un suivi ultérieur (`check back later`, rappels, travail récurrent)
  au lieu de boucles de veille `exec`, d’astuces de délai `yieldMs`, ou d’un sondage répété de `process`
- utiliser `exec` / `process` uniquement pour les commandes qui démarrent maintenant et continuent de s’exécuter
  en arrière-plan
- lorsque le réveil automatique à l’achèvement est activé, démarrer la commande une seule fois et s’appuyer sur
  le chemin de réveil par push lorsqu’elle émet une sortie ou échoue
- utiliser `process` pour les journaux, l’état, l’entrée ou une intervention lorsque vous devez
  inspecter une commande en cours d’exécution
- si la tâche est plus importante, préférer `sessions_spawn` ; l’achèvement du sous-agent se fait par push et est annoncé automatiquement au demandeur
- ne pas sonder `subagents list` / `sessions_list` en boucle uniquement pour attendre
  l’achèvement

Lorsque l’outil expérimental `update_plan` est activé, Tooling indique aussi au
modèle de ne l’utiliser que pour un travail non trivial à plusieurs étapes, de conserver exactement une étape
`in_progress`, et d’éviter de répéter le plan entier après chaque mise à jour.

Les garde-fous de sécurité dans le prompt système sont indicatifs. Ils guident le comportement du modèle mais n’appliquent pas de politique. Utilisez la politique des outils, les approbations d’exécution, le sandboxing et les listes d’autorisation des canaux pour une application stricte ; les opérateurs peuvent les désactiver par conception.

Sur les canaux disposant de cartes/boutons d’approbation natifs, le prompt d’exécution indique maintenant à
l’agent de s’appuyer d’abord sur cette interface d’approbation native. Il ne doit inclure une commande manuelle
`/approve` que si le résultat de l’outil indique que les approbations par chat ne sont pas disponibles ou
si l’approbation manuelle est la seule voie possible.

## Modes de prompt

OpenClaw peut générer des prompts système plus petits pour les sous-agents. L’environnement d’exécution définit un
`promptMode` pour chaque exécution (pas une configuration destinée à l’utilisateur) :

- `full` (par défaut) : inclut toutes les sections ci-dessus.
- `minimal` : utilisé pour les sous-agents ; omet **Skills**, **Memory Recall**, **OpenClaw
  Self-Update**, **Model Aliases**, **User Identity**, **Reply Tags**,
  **Messaging**, **Silent Replies**, et **Heartbeats**. Tooling, **Safety**,
  Workspace, Sandbox, Current Date & Time (quand connue), Runtime, et le contexte
  injecté restent disponibles.
- `none` : renvoie uniquement la ligne d’identité de base.

Lorsque `promptMode=minimal`, les prompts injectés supplémentaires sont libellés **Subagent
Context** au lieu de **Group Chat Context**.

## Injection de l’amorçage de l’espace de travail

Les fichiers d’amorçage sont tronqués puis ajoutés sous **Project Context** afin que le modèle voie le contexte d’identité et de profil sans nécessiter de lectures explicites :

- `AGENTS.md`
- `SOUL.md`
- `TOOLS.md`
- `IDENTITY.md`
- `USER.md`
- `HEARTBEAT.md`
- `BOOTSTRAP.md` (uniquement dans les espaces de travail tout neufs)
- `MEMORY.md` lorsqu’il est présent, sinon `memory.md` comme solution de repli en minuscules

Tous ces fichiers sont **injectés dans la fenêtre de contexte** à chaque tour, sauf si
un garde-fou spécifique à un fichier s’applique. `HEARTBEAT.md` est omis lors des exécutions normales lorsque les
Heartbeats sont désactivés pour l’agent par défaut ou quand
`agents.defaults.heartbeat.includeSystemPromptSection` vaut false. Gardez les fichiers injectés concis —
en particulier `MEMORY.md`, qui peut grossir avec le temps et entraîner une utilisation
du contexte étonnamment élevée ainsi qu’une Compaction plus fréquente.

> **Remarque :** les fichiers quotidiens `memory/*.md` ne font **pas** partie du
> Project Context d’amorçage normal. Lors des tours ordinaires, on y accède à la
> demande via les outils `memory_search` et `memory_get`, donc ils ne comptent pas dans la
> fenêtre de contexte tant que le modèle ne les lit pas explicitement. Les tours `/new` et
> `/reset` simples constituent l’exception : l’environnement d’exécution peut préfixer la mémoire quotidienne récente
> sous forme de bloc ponctuel de contexte de démarrage pour ce premier tour.

Les gros fichiers sont tronqués avec un marqueur. La taille maximale par fichier est contrôlée par
`agents.defaults.bootstrapMaxChars` (par défaut : 12000). Le contenu total injecté de l’amorçage
sur l’ensemble des fichiers est plafonné par `agents.defaults.bootstrapTotalMaxChars`
(par défaut : 60000). Les fichiers manquants injectent un court marqueur de fichier manquant. Lorsqu’une troncature
se produit, OpenClaw peut injecter un bloc d’avertissement dans Project Context ; contrôlez cela avec
`agents.defaults.bootstrapPromptTruncationWarning` (`off`, `once`, `always` ;
par défaut : `once`).

Les sessions de sous-agent n’injectent que `AGENTS.md` et `TOOLS.md` (les autres fichiers d’amorçage
sont filtrés pour conserver un petit contexte de sous-agent).

Les hooks internes peuvent intercepter cette étape via `agent:bootstrap` pour modifier ou remplacer
les fichiers d’amorçage injectés (par exemple en remplaçant `SOUL.md` par une persona alternative).

Si vous souhaitez que l’agent paraisse moins générique, commencez par
[Guide de personnalité SOUL.md](/fr/concepts/soul).

Pour inspecter la contribution de chaque fichier injecté (brut vs injecté, troncature, plus surcharge du schéma d’outil), utilisez `/context list` ou `/context detail`. Voir [Context](/fr/concepts/context).

## Gestion du temps

Le prompt système inclut une section dédiée **Current Date & Time** lorsque le
fuseau horaire de l’utilisateur est connu. Pour conserver la stabilité du cache du prompt, il inclut désormais uniquement le
**fuseau horaire** (sans horloge dynamique ni format d’heure).

Utilisez `session_status` lorsque l’agent a besoin de l’heure actuelle ; la carte d’état
inclut une ligne d’horodatage. Le même outil peut aussi définir une substitution de modèle par session
(`model=default` l’efface).

Configurez avec :

- `agents.defaults.userTimezone`
- `agents.defaults.timeFormat` (`auto` | `12` | `24`)

Voir [Date & Time](/fr/date-time) pour les détails complets du comportement.

## Skills

Lorsque des Skills admissibles existent, OpenClaw injecte une **liste compacte des Skills disponibles**
(`formatSkillsForPrompt`) qui inclut le **chemin de fichier** de chaque Skill. Le
prompt indique au modèle d’utiliser `read` pour charger le SKILL.md à l’emplacement
indiqué (espace de travail, géré ou intégré). Si aucun Skill n’est admissible, la
section Skills est omise.

L’admissibilité comprend les garde-fous de métadonnées du Skill, les vérifications de l’environnement/configuration d’exécution,
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

Cela permet de conserver un petit prompt de base tout en activant un usage ciblé des Skills.

Le budget de la liste des Skills appartient au sous-système des Skills :

- Valeur globale par défaut : `skills.limits.maxSkillsPromptChars`
- Substitution par agent : `agents.list[].skillsLimits.maxSkillsPromptChars`

Les extraits d’exécution génériques bornés utilisent une surface différente :

- `agents.defaults.contextLimits.*`
- `agents.list[].contextLimits.*`

Cette séparation garde le dimensionnement des Skills distinct du dimensionnement de lecture/injection de l’exécution, par exemple pour
`memory_get`, les résultats d’outils en direct, et les actualisations de AGENTS.md après Compaction.

## Documentation

Lorsqu’elle est disponible, le prompt système inclut une section **Documentation** qui pointe vers le
répertoire local de documentation OpenClaw (soit `docs/` dans l’espace de travail du dépôt, soit la documentation du
package npm intégré) et mentionne également le miroir public, le dépôt source, le Discord de la communauté, et
ClawHub ([https://clawhub.ai](https://clawhub.ai)) pour la découverte de Skills. Le prompt indique au modèle de consulter d’abord la documentation locale
pour le comportement, les commandes, la configuration ou l’architecture d’OpenClaw, et d’exécuter
lui-même `openclaw status` lorsque c’est possible (en ne demandant à l’utilisateur que lorsqu’il n’y a pas accès).
