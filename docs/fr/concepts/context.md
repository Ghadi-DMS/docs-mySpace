---
read_when:
    - Vous voulez comprendre ce que signifie « contexte » dans OpenClaw
    - Vous déboguez pourquoi le modèle « sait » quelque chose (ou l’a oublié)
    - Vous souhaitez réduire la surcharge de contexte (/context, /status, /compact)
summary: 'Contexte : ce que voit le modèle, comment il est construit et comment l’inspecter'
title: Contexte
x-i18n:
    generated_at: "2026-04-18T06:43:36Z"
    model: gpt-5.4
    provider: openai
    source_hash: 477ccb1d9654968d0e904b6846b32b8c14db6b6c0d3d2ec2b7409639175629f9
    source_path: concepts/context.md
    workflow: 15
---

# Contexte

Le « contexte » est **tout ce que OpenClaw envoie au modèle pour une exécution**. Il est limité par la **fenêtre de contexte** du modèle (limite de tokens).

Modèle mental pour débutants :

- **Prompt système** (construit par OpenClaw) : règles, outils, liste des Skills, heure/exécution, et fichiers d’espace de travail injectés.
- **Historique de conversation** : vos messages + les messages de l’assistant pour cette session.
- **Appels/résultats d’outils + pièces jointes** : sortie de commandes, lectures de fichiers, images/audio, etc.

Le contexte n’est _pas la même chose_ que la « mémoire » : la mémoire peut être stockée sur disque et rechargée plus tard ; le contexte est ce qui se trouve dans la fenêtre actuelle du modèle.

## Démarrage rapide (inspecter le contexte)

- `/status` → vue rapide « à quel point ma fenêtre est-elle remplie ? » + paramètres de session.
- `/context list` → ce qui est injecté + tailles approximatives (par fichier + totaux).
- `/context detail` → ventilation plus approfondie : par fichier, tailles des schémas d’outils, tailles des entrées de Skills, et taille du prompt système.
- `/usage tokens` → ajoute un pied de page d’utilisation par réponse aux réponses normales.
- `/compact` → résume l’historique plus ancien dans une entrée compacte pour libérer de l’espace dans la fenêtre.

Voir aussi : [Commandes slash](/fr/tools/slash-commands), [Utilisation des tokens et coûts](/fr/reference/token-use), [Compaction](/fr/concepts/compaction).

## Exemple de sortie

Les valeurs varient selon le modèle, le fournisseur, la politique d’outils et le contenu de votre espace de travail.

### `/context list`

```
🧠 Ventilation du contexte
Espace de travail : <workspaceDir>
Bootstrap max/fichier : 12,000 caractères
Sandbox : mode=non-main sandboxed=false
Prompt système (exécution) : 38,412 caractères (~9,603 tok) (Contexte du projet 23,901 caractères (~5,976 tok))

Fichiers de l’espace de travail injectés :
- AGENTS.md: OK | brut 1,742 caractères (~436 tok) | injecté 1,742 caractères (~436 tok)
- SOUL.md: OK | brut 912 caractères (~228 tok) | injecté 912 caractères (~228 tok)
- TOOLS.md: TRONQUÉ | brut 54,210 caractères (~13,553 tok) | injecté 20,962 caractères (~5,241 tok)
- IDENTITY.md: OK | brut 211 caractères (~53 tok) | injecté 211 caractères (~53 tok)
- USER.md: OK | brut 388 caractères (~97 tok) | injecté 388 caractères (~97 tok)
- HEARTBEAT.md: ABSENT | brut 0 | injecté 0
- BOOTSTRAP.md: OK | brut 0 caractères (~0 tok) | injecté 0 caractères (~0 tok)

Liste des Skills (texte du prompt système) : 2,184 caractères (~546 tok) (12 Skills)
Outils : read, edit, write, exec, process, browser, message, sessions_send, …
Liste des outils (texte du prompt système) : 1,032 caractères (~258 tok)
Schémas d’outils (JSON) : 31,988 caractères (~7,997 tok) (comptent dans le contexte ; non affichés comme texte)
Outils : (identiques ci-dessus)

Tokens de session (en cache) : 14,250 au total / ctx=32,000
```

### `/context detail`

```
🧠 Ventilation du contexte (détaillée)
…
Principaux Skills (taille d’entrée du prompt) :
- frontend-design: 412 caractères (~103 tok)
- oracle: 401 caractères (~101 tok)
… (+10 autres Skills)

Principaux outils (taille du schéma) :
- browser: 9,812 caractères (~2,453 tok)
- exec: 6,240 caractères (~1,560 tok)
… (+N autres outils)
```

## Ce qui compte dans la fenêtre de contexte

Tout ce que le modèle reçoit compte, y compris :

- Prompt système (toutes les sections).
- Historique de conversation.
- Appels d’outils + résultats d’outils.
- Pièces jointes/transcriptions (images/audio/fichiers).
- Résumés de Compaction et artefacts d’élagage.
- « Wrappers » du fournisseur ou en-têtes cachés (non visibles, mais tout de même comptés).

## Comment OpenClaw construit le prompt système

Le prompt système appartient à **OpenClaw** et est reconstruit à chaque exécution. Il inclut :

- Liste des outils + courtes descriptions.
- Liste des Skills (métadonnées uniquement ; voir ci-dessous).
- Emplacement de l’espace de travail.
- Heure (UTC + heure utilisateur convertie si configurée).
- Métadonnées d’exécution (hôte/OS/modèle/réflexion).
- Fichiers bootstrap de l’espace de travail injectés sous **Contexte du projet**.

Ventilation complète : [Prompt système](/fr/concepts/system-prompt).

## Fichiers de l’espace de travail injectés (Contexte du projet)

Par défaut, OpenClaw injecte un ensemble fixe de fichiers d’espace de travail (s’ils sont présents) :

- `AGENTS.md`
- `SOUL.md`
- `TOOLS.md`
- `IDENTITY.md`
- `USER.md`
- `HEARTBEAT.md`
- `BOOTSTRAP.md` (première exécution uniquement)

Les fichiers volumineux sont tronqués fichier par fichier à l’aide de `agents.defaults.bootstrapMaxChars` (par défaut `12000` caractères). OpenClaw applique aussi un plafond total d’injection bootstrap sur l’ensemble des fichiers avec `agents.defaults.bootstrapTotalMaxChars` (par défaut `60000` caractères). `/context` affiche les tailles **brutes vs injectées** et indique si une troncature a eu lieu.

Lorsqu’une troncature se produit, l’exécution peut injecter un bloc d’avertissement dans le prompt sous Contexte du projet. Configurez cela avec `agents.defaults.bootstrapPromptTruncationWarning` (`off`, `once`, `always` ; `once` par défaut).

## Skills : injectés ou chargés à la demande

Le prompt système inclut une liste compacte des **Skills** (nom + description + emplacement). Cette liste a un coût réel.

Les instructions des Skills ne sont _pas_ incluses par défaut. Le modèle est censé `read` le `SKILL.md` du skill **uniquement lorsque nécessaire**.

## Outils : il y a deux coûts

Les outils affectent le contexte de deux façons :

1. **Texte de la liste des outils** dans le prompt système (ce que vous voyez comme « Tooling »).
2. **Schémas d’outils** (JSON). Ils sont envoyés au modèle afin qu’il puisse appeler des outils. Ils comptent dans le contexte même si vous ne les voyez pas sous forme de texte brut.

`/context detail` ventile les schémas d’outils les plus volumineux afin que vous puissiez voir ce qui domine.

## Commandes, directives et « raccourcis inline »

Les commandes slash sont gérées par la Gateway. Il existe quelques comportements différents :

- **Commandes autonomes** : un message qui contient uniquement `/...` s’exécute comme une commande.
- **Directives** : `/think`, `/verbose`, `/trace`, `/reasoning`, `/elevated`, `/model`, `/queue` sont retirées avant que le modèle ne voie le message.
  - Les messages contenant uniquement des directives conservent les paramètres de session.
  - Les directives inline dans un message normal agissent comme des indications propres au message.
- **Raccourcis inline** (expéditeurs autorisés uniquement) : certains tokens `/...` dans un message normal peuvent s’exécuter immédiatement (exemple : « hey /status »), et sont retirés avant que le modèle ne voie le texte restant.

Détails : [Commandes slash](/fr/tools/slash-commands).

## Sessions, Compaction et élagage (ce qui persiste)

Ce qui persiste entre les messages dépend du mécanisme :

- **L’historique normal** persiste dans la transcription de session jusqu’à ce qu’il soit compacté/élagué par la politique.
- **Compaction** conserve un résumé dans la transcription et garde intacts les messages récents.
- **L’élagage** retire les anciens résultats d’outils du prompt _en mémoire_ pour une exécution, mais ne réécrit pas la transcription.

Documentation : [Session](/fr/concepts/session), [Compaction](/fr/concepts/compaction), [Élagage de session](/fr/concepts/session-pruning).

Par défaut, OpenClaw utilise le moteur de contexte intégré `legacy` pour l’assemblage et la Compaction. Si vous installez un Plugin qui fournit `kind: "context-engine"` et le sélectionnez avec `plugins.slots.contextEngine`, OpenClaw délègue à ce moteur l’assemblage du contexte, `/compact` et les hooks de cycle de vie de contexte des sous-agents associés. `ownsCompaction: false` ne revient pas automatiquement au moteur legacy ; le moteur actif doit tout de même implémenter correctement `compact()`. Voir [Context Engine](/fr/concepts/context-engine) pour l’interface enfichable complète, les hooks de cycle de vie et la configuration.

## Ce que `/context` rapporte réellement

`/context` privilégie le dernier rapport de prompt système **construit à l’exécution** lorsqu’il est disponible :

- `System prompt (run)` = capturé lors de la dernière exécution embarquée (capable d’utiliser des outils) et conservé dans le stockage de session.
- `System prompt (estimate)` = calculé à la volée lorsqu’aucun rapport d’exécution n’existe (ou lors d’une exécution via un backend CLI qui ne génère pas le rapport).

Dans les deux cas, il rapporte des tailles et les principaux contributeurs ; il **n’affiche pas** le prompt système complet ni les schémas d’outils.

## Lié

- [Context Engine](/fr/concepts/context-engine) — injection de contexte personnalisée via des plugins
- [Compaction](/fr/concepts/compaction) — résumé des longues conversations
- [Prompt système](/fr/concepts/system-prompt) — comment le prompt système est construit
- [Boucle d’agent](/fr/concepts/agent-loop) — le cycle complet d’exécution de l’agent
