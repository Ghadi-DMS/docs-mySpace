---
read_when:
    - Vous voulez comprendre ce que signifie « contexte » dans OpenClaw
    - Vous déboguez pourquoi le modèle « sait » quelque chose (ou l’a oublié)
    - Vous voulez réduire la surcharge de contexte (`/context`, `/status`, `/compact`)
summary: 'Contexte : ce que voit le modèle, comment il est construit et comment l’inspecter'
title: Contexte
x-i18n:
    generated_at: "2026-04-07T06:48:59Z"
    model: gpt-5.4
    provider: openai
    source_hash: a75b4cd65bf6385d46265b9ce1643310bc99d220e35ec4b4924096bed3ca4aa0
    source_path: concepts/context.md
    workflow: 15
---

# Contexte

Le « contexte » est **tout ce qu’OpenClaw envoie au modèle pour une exécution**. Il est limité par la **fenêtre de contexte** du modèle (limite de jetons).

Modèle mental pour débutants :

- **Prompt système** (construit par OpenClaw) : règles, outils, liste des compétences, heure/exécution, et fichiers d’espace de travail injectés.
- **Historique de conversation** : vos messages + les messages de l’assistant pour cette session.
- **Appels/résultats d’outils + pièces jointes** : sortie de commande, lectures de fichiers, images/audio, etc.

Le contexte _n’est pas la même chose_ que la « mémoire » : la mémoire peut être stockée sur disque et rechargée plus tard ; le contexte est ce qui se trouve dans la fenêtre actuelle du modèle.

## Démarrage rapide (inspecter le contexte)

- `/status` → vue rapide « à quel point ma fenêtre est-elle remplie ? » + paramètres de session.
- `/context list` → ce qui est injecté + tailles approximatives (par fichier + totaux).
- `/context detail` → ventilation plus détaillée : tailles par fichier, tailles des schémas d’outils, tailles des entrées de compétences et taille du prompt système.
- `/usage tokens` → ajoute un pied de page d’utilisation par réponse aux réponses normales.
- `/compact` → résume l’historique plus ancien dans une entrée compacte pour libérer de l’espace dans la fenêtre.

Voir aussi : [Commandes slash](/fr/tools/slash-commands), [Utilisation des jetons et coûts](/fr/reference/token-use), [Compaction](/fr/concepts/compaction).

## Exemple de sortie

Les valeurs varient selon le modèle, le fournisseur, la politique d’outils et le contenu de votre espace de travail.

### `/context list`

```
🧠 Répartition du contexte
Workspace: <workspaceDir>
Bootstrap max/file: 20,000 chars
Sandbox: mode=non-main sandboxed=false
System prompt (run): 38,412 chars (~9,603 tok) (Project Context 23,901 chars (~5,976 tok))

Injected workspace files:
- AGENTS.md: OK | raw 1,742 chars (~436 tok) | injected 1,742 chars (~436 tok)
- SOUL.md: OK | raw 912 chars (~228 tok) | injected 912 chars (~228 tok)
- TOOLS.md: TRUNCATED | raw 54,210 chars (~13,553 tok) | injected 20,962 chars (~5,241 tok)
- IDENTITY.md: OK | raw 211 chars (~53 tok) | injected 211 chars (~53 tok)
- USER.md: OK | raw 388 chars (~97 tok) | injected 388 chars (~97 tok)
- HEARTBEAT.md: MISSING | raw 0 | injected 0
- BOOTSTRAP.md: OK | raw 0 chars (~0 tok) | injected 0 chars (~0 tok)

Skills list (system prompt text): 2,184 chars (~546 tok) (12 skills)
Tools: read, edit, write, exec, process, browser, message, sessions_send, …
Tool list (system prompt text): 1,032 chars (~258 tok)
Tool schemas (JSON): 31,988 chars (~7,997 tok) (counts toward context; not shown as text)
Tools: (same as above)

Session tokens (cached): 14,250 total / ctx=32,000
```

### `/context detail`

```
🧠 Répartition du contexte (détaillée)
…
Principales compétences (taille d’entrée dans le prompt) :
- frontend-design: 412 chars (~103 tok)
- oracle: 401 chars (~101 tok)
… (+10 de plus)

Principaux outils (taille du schéma) :
- browser: 9,812 chars (~2,453 tok)
- exec: 6,240 chars (~1,560 tok)
… (+N de plus)
```

## Ce qui compte dans la fenêtre de contexte

Tout ce que le modèle reçoit compte, notamment :

- Le prompt système (toutes les sections).
- L’historique de conversation.
- Les appels d’outils + les résultats d’outils.
- Les pièces jointes/transcriptions (images/audio/fichiers).
- Les résumés de compaction et les artefacts d’élagage.
- Les « wrappers » du fournisseur ou les en-têtes masqués (non visibles, mais quand même comptés).

## Comment OpenClaw construit le prompt système

Le prompt système est **géré par OpenClaw** et reconstruit à chaque exécution. Il inclut :

- La liste des outils + de courtes descriptions.
- La liste des compétences (métadonnées uniquement ; voir ci-dessous).
- L’emplacement de l’espace de travail.
- L’heure (UTC + heure utilisateur convertie si configurée).
- Les métadonnées d’exécution (hôte/OS/modèle/réflexion).
- Les fichiers bootstrap de l’espace de travail injectés sous **Project Context**.

Ventilation complète : [Prompt système](/fr/concepts/system-prompt).

## Fichiers d’espace de travail injectés (Project Context)

Par défaut, OpenClaw injecte un ensemble fixe de fichiers de l’espace de travail (s’ils sont présents) :

- `AGENTS.md`
- `SOUL.md`
- `TOOLS.md`
- `IDENTITY.md`
- `USER.md`
- `HEARTBEAT.md`
- `BOOTSTRAP.md` (première exécution uniquement)

Les fichiers volumineux sont tronqués fichier par fichier à l’aide de `agents.defaults.bootstrapMaxChars` (valeur par défaut : `20000` caractères). OpenClaw applique aussi un plafond total d’injection bootstrap sur l’ensemble des fichiers avec `agents.defaults.bootstrapTotalMaxChars` (valeur par défaut : `150000` caractères). `/context` affiche les tailles **brutes vs injectées** et indique si une troncature a eu lieu.

Lorsqu’une troncature se produit, l’exécution peut injecter un bloc d’avertissement dans le prompt sous Project Context. Configurez cela avec `agents.defaults.bootstrapPromptTruncationWarning` (`off`, `once`, `always` ; valeur par défaut : `once`).

## Compétences : injectées vs chargées à la demande

Le prompt système inclut une **liste de compétences** compacte (nom + description + emplacement). Cette liste a une surcharge réelle.

Les instructions des compétences ne sont _pas_ incluses par défaut. Le modèle est censé `read` le `SKILL.md` de la compétence **uniquement lorsque nécessaire**.

## Outils : il y a deux coûts

Les outils affectent le contexte de deux façons :

1. **Texte de la liste des outils** dans le prompt système (ce que vous voyez sous « Tooling »).
2. **Schémas d’outils** (JSON). Ils sont envoyés au modèle pour qu’il puisse appeler des outils. Ils comptent dans le contexte même si vous ne les voyez pas comme du texte brut.

`/context detail` détaille les plus gros schémas d’outils afin que vous puissiez voir ce qui domine.

## Commandes, directives et « raccourcis inline »

Les commandes slash sont gérées par la Gateway. Il existe quelques comportements différents :

- **Commandes autonomes** : un message qui contient uniquement `/...` est exécuté comme commande.
- **Directives** : `/think`, `/verbose`, `/reasoning`, `/elevated`, `/model`, `/queue` sont retirées avant que le modèle ne voie le message.
  - Les messages contenant uniquement des directives conservent les paramètres de session.
  - Les directives inline dans un message normal agissent comme des indications par message.
- **Raccourcis inline** (expéditeurs autorisés uniquement) : certains jetons `/...` à l’intérieur d’un message normal peuvent être exécutés immédiatement (exemple : « hey /status ») et sont retirés avant que le modèle ne voie le texte restant.

Détails : [Commandes slash](/fr/tools/slash-commands).

## Sessions, compaction et élagage (ce qui persiste)

Ce qui persiste d’un message à l’autre dépend du mécanisme :

- **L’historique normal** persiste dans la transcription de session jusqu’à ce qu’il soit compacté/élagué par la politique.
- **La compaction** conserve un résumé dans la transcription et garde intacts les messages récents.
- **L’élagage** supprime les anciens résultats d’outils du prompt _en mémoire_ pour une exécution, mais ne réécrit pas la transcription.

Documentation : [Session](/fr/concepts/session), [Compaction](/fr/concepts/compaction), [Élagage de session](/fr/concepts/session-pruning).

Par défaut, OpenClaw utilise le moteur de contexte intégré `legacy` pour l’assemblage et la
compaction. Si vous installez un plugin qui fournit `kind: "context-engine"` et
le sélectionnez avec `plugins.slots.contextEngine`, OpenClaw délègue à ce
moteur l’assemblage du contexte, `/compact` et les hooks de cycle de vie du
contexte des sous-agents associés. `ownsCompaction: false` ne revient pas
automatiquement au moteur `legacy` ; le moteur actif doit tout de même
implémenter `compact()` correctement. Voir
[Context Engine](/fr/concepts/context-engine) pour l’interface enfichable complète,
les hooks de cycle de vie et la configuration.

## Ce que `/context` rapporte réellement

`/context` privilégie le dernier rapport de prompt système **construit à l’exécution** lorsqu’il est disponible :

- `System prompt (run)` = capturé lors de la dernière exécution embarquée (capable d’utiliser des outils) et conservé dans le stockage de session.
- `System prompt (estimate)` = calculé à la volée lorsqu’aucun rapport d’exécution n’existe (ou lors d’une exécution via un backend CLI qui ne génère pas le rapport).

Dans les deux cas, il rapporte les tailles et les principaux contributeurs ; il **n’affiche pas** le prompt système complet ni les schémas d’outils.

## Lié

- [Context Engine](/fr/concepts/context-engine) — injection de contexte personnalisée via des plugins
- [Compaction](/fr/concepts/compaction) — résumé des longues conversations
- [Prompt système](/fr/concepts/system-prompt) — comment le prompt système est construit
- [Boucle d’agent](/fr/concepts/agent-loop) — le cycle d’exécution complet de l’agent
