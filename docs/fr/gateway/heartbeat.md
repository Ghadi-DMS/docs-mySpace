---
read_when:
    - Ajustement de la cadence ou de la messagerie du heartbeat
    - Choix entre heartbeat et cron pour les tâches planifiées
summary: Messages de sondage de heartbeat et règles de notification
title: Heartbeat
x-i18n:
    generated_at: "2026-04-08T02:15:18Z"
    model: gpt-5.4
    provider: openai
    source_hash: a8021d747637060eacb91ec5f75904368a08790c19f4fca32acda8c8c0a25e41
    source_path: gateway/heartbeat.md
    workflow: 15
---

# Heartbeat (Gateway)

> **Heartbeat ou Cron ?** Voir [Automatisation et tâches](/fr/automation) pour savoir quand utiliser l’un ou l’autre.

Heartbeat exécute des **tours d’agent périodiques** dans la session principale afin que le modèle puisse
signaler tout ce qui nécessite de l’attention sans vous spammer.

Heartbeat est un tour planifié de la session principale — il **ne** crée pas d’enregistrements de [tâche d’arrière-plan](/fr/automation/tasks).
Les enregistrements de tâche sont destinés au travail détaché (exécutions ACP, sous-agents, tâches cron isolées).

Dépannage : [Tâches planifiées](/fr/automation/cron-jobs#troubleshooting)

## Démarrage rapide (débutant)

1. Laissez les heartbeats activés (la valeur par défaut est `30m`, ou `1h` pour l’authentification Anthropic OAuth/par jeton, y compris la réutilisation de Claude CLI) ou définissez votre propre cadence.
2. Créez une petite liste de contrôle `HEARTBEAT.md` ou un bloc `tasks:` dans l’espace de travail de l’agent (facultatif mais recommandé).
3. Décidez où les messages de heartbeat doivent être envoyés (`target: "none"` est la valeur par défaut ; définissez `target: "last"` pour les acheminer vers le dernier contact).
4. Facultatif : activez la livraison du raisonnement du heartbeat pour plus de transparence.
5. Facultatif : utilisez un contexte de bootstrap léger si les exécutions de heartbeat n’ont besoin que de `HEARTBEAT.md`.
6. Facultatif : activez les sessions isolées pour éviter d’envoyer l’historique complet de la conversation à chaque heartbeat.
7. Facultatif : limitez les heartbeats aux heures actives (heure locale).

Exemple de configuration :

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // livraison explicite au dernier contact (la valeur par défaut est "none")
        directPolicy: "allow", // défaut : autoriser les cibles directes/DM ; définissez "block" pour les supprimer
        lightContext: true, // facultatif : n’injecter que HEARTBEAT.md depuis les fichiers de bootstrap
        isolatedSession: true, // facultatif : nouvelle session à chaque exécution (sans historique de conversation)
        // activeHours: { start: "08:00", end: "24:00" },
        // includeReasoning: true, // facultatif : envoyer aussi un message `Reasoning:` séparé
      },
    },
  },
}
```

## Valeurs par défaut

- Intervalle : `30m` (ou `1h` lorsque l’authentification Anthropic OAuth/par jeton est le mode d’authentification détecté, y compris la réutilisation de Claude CLI). Définissez `agents.defaults.heartbeat.every` ou `agents.list[].heartbeat.every` par agent ; utilisez `0m` pour désactiver.
- Corps du prompt (configurable via `agents.defaults.heartbeat.prompt`) :
  `Read HEARTBEAT.md if it exists (workspace context). Follow it strictly. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.`
- Le prompt de heartbeat est envoyé **tel quel** comme message utilisateur. Le prompt
  système inclut une section « Heartbeat » uniquement lorsque les heartbeats sont activés pour l’agent
  par défaut, et que l’exécution est marquée en interne.
- Lorsque les heartbeats sont désactivés avec `0m`, les exécutions normales omettent aussi `HEARTBEAT.md`
  du contexte de bootstrap afin que le modèle ne voie pas d’instructions réservées au heartbeat.
- Les heures actives (`heartbeat.activeHours`) sont vérifiées dans le fuseau horaire configuré.
  En dehors de cette plage, les heartbeats sont ignorés jusqu’au prochain tick à l’intérieur de la plage.

## À quoi sert le prompt de heartbeat

Le prompt par défaut est volontairement large :

- **Tâches d’arrière-plan** : « Consider outstanding tasks » incite l’agent à examiner
  les suivis en attente (boîte de réception, calendrier, rappels, travail en file d’attente) et à signaler tout ce qui est urgent.
- **Prise de nouvelles humaine** : « Checkup sometimes on your human during day time » incite à
  envoyer occasionnellement un léger message du type « as-tu besoin de quelque chose ? », tout en évitant le spam nocturne
  grâce à votre fuseau horaire local configuré (voir [/concepts/timezone](/fr/concepts/timezone)).

Heartbeat peut réagir à des [tâches d’arrière-plan](/fr/automation/tasks) terminées, mais une exécution de heartbeat ne crée pas elle-même d’enregistrement de tâche.

Si vous souhaitez qu’un heartbeat fasse quelque chose de très précis (par ex. « vérifier les statistiques PubSub Gmail »
ou « vérifier l’état de santé de la gateway »), définissez `agents.defaults.heartbeat.prompt` (ou
`agents.list[].heartbeat.prompt`) sur un corps personnalisé (envoyé tel quel).

## Contrat de réponse

- Si rien ne nécessite d’attention, répondez par **`HEARTBEAT_OK`**.
- Lors des exécutions de heartbeat, OpenClaw traite `HEARTBEAT_OK` comme un accusé de réception lorsqu’il apparaît
  au **début ou à la fin** de la réponse. Le jeton est supprimé et la réponse est
  ignorée si le contenu restant est **≤ `ackMaxChars`** (par défaut : 300).
- Si `HEARTBEAT_OK` apparaît au **milieu** d’une réponse, il n’est pas traité
  de manière spéciale.
- Pour les alertes, **n’incluez pas** `HEARTBEAT_OK` ; renvoyez uniquement le texte de l’alerte.

En dehors des heartbeats, un `HEARTBEAT_OK` isolé au début/à la fin d’un message est supprimé
et journalisé ; un message qui n’est que `HEARTBEAT_OK` est ignoré.

## Configuration

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m", // défaut : 30m (0m désactive)
        model: "anthropic/claude-opus-4-6",
        includeReasoning: false, // défaut : false (livrer un message `Reasoning:` séparé quand disponible)
        lightContext: false, // défaut : false ; true conserve seulement HEARTBEAT.md parmi les fichiers de bootstrap de l’espace de travail
        isolatedSession: false, // défaut : false ; true exécute chaque heartbeat dans une session fraîche (sans historique de conversation)
        target: "last", // défaut : none | options : last | none | <id de canal> (core ou plugin, par ex. "bluebubbles")
        to: "+15551234567", // remplacement facultatif spécifique au canal
        accountId: "ops-bot", // id facultatif de canal multi-compte
        prompt: "Read HEARTBEAT.md if it exists (workspace context). Follow it strictly. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.",
        ackMaxChars: 300, // nombre maximal de caractères autorisés après HEARTBEAT_OK
      },
    },
  },
}
```

### Portée et priorité

- `agents.defaults.heartbeat` définit le comportement global du heartbeat.
- `agents.list[].heartbeat` est fusionné par-dessus ; si un agent possède un bloc `heartbeat`, **seuls ces agents** exécutent des heartbeats.
- `channels.defaults.heartbeat` définit les valeurs de visibilité par défaut pour tous les canaux.
- `channels.<channel>.heartbeat` remplace les valeurs par défaut du canal.
- `channels.<channel>.accounts.<id>.heartbeat` (canaux multi-compte) remplace les paramètres par canal.

### Heartbeats par agent

Si une entrée `agents.list[]` inclut un bloc `heartbeat`, **seuls ces agents**
exécutent des heartbeats. Le bloc par agent est fusionné par-dessus `agents.defaults.heartbeat`
(vous pouvez donc définir des valeurs partagées une seule fois et les remplacer par agent).

Exemple : deux agents, seul le deuxième agent exécute des heartbeats.

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // livraison explicite au dernier contact (la valeur par défaut est "none")
      },
    },
    list: [
      { id: "main", default: true },
      {
        id: "ops",
        heartbeat: {
          every: "1h",
          target: "whatsapp",
          to: "+15551234567",
          prompt: "Read HEARTBEAT.md if it exists (workspace context). Follow it strictly. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.",
        },
      },
    ],
  },
}
```

### Exemple d’heures actives

Limitez les heartbeats aux heures de bureau dans un fuseau horaire spécifique :

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // livraison explicite au dernier contact (la valeur par défaut est "none")
        activeHours: {
          start: "09:00",
          end: "22:00",
          timezone: "America/New_York", // facultatif ; utilise votre userTimezone si défini, sinon le fuseau horaire de l’hôte
        },
      },
    },
  },
}
```

En dehors de cette plage (avant 9 h ou après 22 h, heure de l’Est), les heartbeats sont ignorés. Le prochain tick planifié dans la plage s’exécutera normalement.

### Configuration 24 h/24, 7 j/7

Si vous souhaitez que les heartbeats s’exécutent toute la journée, utilisez l’un de ces modèles :

- Omettez complètement `activeHours` (aucune restriction de plage horaire ; c’est le comportement par défaut).
- Définissez une plage sur toute la journée : `activeHours: { start: "00:00", end: "24:00" }`.

Ne définissez pas la même heure pour `start` et `end` (par exemple `08:00` à `08:00`).
Cela est traité comme une fenêtre de largeur nulle, donc les heartbeats sont toujours ignorés.

### Exemple multi-compte

Utilisez `accountId` pour cibler un compte spécifique sur les canaux multi-compte comme Telegram :

```json5
{
  agents: {
    list: [
      {
        id: "ops",
        heartbeat: {
          every: "1h",
          target: "telegram",
          to: "12345678:topic:42", // facultatif : acheminer vers un sujet/thread spécifique
          accountId: "ops-bot",
        },
      },
    ],
  },
  channels: {
    telegram: {
      accounts: {
        "ops-bot": { botToken: "YOUR_TELEGRAM_BOT_TOKEN" },
      },
    },
  },
}
```

### Notes sur les champs

- `every` : intervalle de heartbeat (chaîne de durée ; unité par défaut = minutes).
- `model` : remplacement facultatif du modèle pour les exécutions de heartbeat (`provider/model`).
- `includeReasoning` : lorsqu’il est activé, livre aussi le message séparé `Reasoning:` lorsqu’il est disponible (même forme que `/reasoning on`).
- `lightContext` : lorsque true, les exécutions de heartbeat utilisent un contexte de bootstrap léger et ne conservent que `HEARTBEAT.md` parmi les fichiers de bootstrap de l’espace de travail.
- `isolatedSession` : lorsque true, chaque heartbeat s’exécute dans une session fraîche, sans historique de conversation précédent. Utilise le même modèle d’isolation que cron `sessionTarget: "isolated"`. Réduit fortement le coût en jetons par heartbeat. Combinez avec `lightContext: true` pour des économies maximales. Le routage de livraison utilise toujours le contexte de la session principale.
- `session` : clé de session facultative pour les exécutions de heartbeat.
  - `main` (par défaut) : session principale de l’agent.
  - Clé de session explicite (copiée depuis `openclaw sessions --json` ou la [CLI sessions](/cli/sessions)).
  - Formats de clé de session : voir [Sessions](/fr/concepts/session) et [Groupes](/fr/channels/groups).
- `target` :
  - `last` : livrer au dernier canal externe utilisé.
  - canal explicite : tout canal configuré ou id de plugin, par exemple `discord`, `matrix`, `telegram`, ou `whatsapp`.
  - `none` (par défaut) : exécuter le heartbeat mais **ne pas livrer** en externe.
- `directPolicy` : contrôle le comportement de livraison directe/DM :
  - `allow` (par défaut) : autoriser la livraison de heartbeat en direct/DM.
  - `block` : supprimer la livraison directe/DM (`reason=dm-blocked`).
- `to` : remplacement facultatif du destinataire (id spécifique au canal, par ex. E.164 pour WhatsApp ou un id de chat Telegram). Pour les sujets/threads Telegram, utilisez `<chatId>:topic:<messageThreadId>`.
- `accountId` : id de compte facultatif pour les canaux multi-compte. Lorsque `target: "last"`, l’id de compte s’applique au dernier canal résolu si celui-ci prend en charge les comptes ; sinon il est ignoré. Si l’id de compte ne correspond pas à un compte configuré pour le canal résolu, la livraison est ignorée.
- `prompt` : remplace le corps du prompt par défaut (pas de fusion).
- `ackMaxChars` : nombre maximal de caractères autorisés après `HEARTBEAT_OK` avant livraison.
- `suppressToolErrorWarnings` : lorsque true, supprime les charges utiles d’avertissement d’erreur d’outil pendant les exécutions de heartbeat.
- `activeHours` : limite les exécutions de heartbeat à une plage horaire. Objet avec `start` (HH:MM, inclusif ; utilisez `00:00` pour le début de journée), `end` (HH:MM exclusif ; `24:00` est autorisé pour la fin de journée), et `timezone` facultatif.
  - Omis ou `"user"` : utilise votre `agents.defaults.userTimezone` s’il est défini, sinon revient au fuseau horaire du système hôte.
  - `"local"` : utilise toujours le fuseau horaire du système hôte.
  - Tout identifiant IANA (par ex. `America/New_York`) : utilisé directement ; s’il est invalide, revient au comportement `"user"` ci-dessus.
  - `start` et `end` ne doivent pas être égaux pour une fenêtre active ; des valeurs égales sont traitées comme une largeur nulle (toujours hors de la fenêtre).
  - En dehors de la fenêtre active, les heartbeats sont ignorés jusqu’au prochain tick à l’intérieur de la fenêtre.

## Comportement de livraison

- Les heartbeats s’exécutent par défaut dans la session principale de l’agent (`agent:<id>:<mainKey>`),
  ou `global` lorsque `session.scope = "global"`. Définissez `session` pour remplacer cela par une
  session de canal spécifique (Discord/WhatsApp/etc.).
- `session` n’affecte que le contexte d’exécution ; la livraison est contrôlée par `target` et `to`.
- Pour livrer à un canal/destinataire spécifique, définissez `target` + `to`. Avec
  `target: "last"`, la livraison utilise le dernier canal externe de cette session.
- Les livraisons de heartbeat autorisent par défaut les cibles directes/DM. Définissez `directPolicy: "block"` pour supprimer les envois vers des cibles directes tout en exécutant quand même le tour de heartbeat.
- Si la file principale est occupée, le heartbeat est ignoré et réessayé plus tard.
- Si `target` ne se résout vers aucune destination externe, l’exécution a quand même lieu mais aucun
  message sortant n’est envoyé.
- Si `showOk`, `showAlerts` et `useIndicator` sont tous désactivés, l’exécution est ignorée immédiatement avec `reason=alerts-disabled`.
- Si seule la livraison des alertes est désactivée, OpenClaw peut quand même exécuter le heartbeat, mettre à jour les horodatages des tâches dues, restaurer l’horodatage d’inactivité de la session, et supprimer la charge utile d’alerte externe.
- Les réponses réservées au heartbeat **ne** maintiennent **pas** la session active ; le dernier `updatedAt`
  est restauré afin que l’expiration par inactivité se comporte normalement.
- Les [tâches d’arrière-plan](/fr/automation/tasks) détachées peuvent mettre en file un événement système et réveiller heartbeat lorsque la session principale doit remarquer rapidement quelque chose. Ce réveil ne transforme pas l’exécution de heartbeat en tâche d’arrière-plan.

## Contrôles de visibilité

Par défaut, les accusés de réception `HEARTBEAT_OK` sont supprimés tandis que le contenu des alertes est
livré. Vous pouvez ajuster cela par canal ou par compte :

```yaml
channels:
  defaults:
    heartbeat:
      showOk: false # Masquer HEARTBEAT_OK (par défaut)
      showAlerts: true # Afficher les messages d’alerte (par défaut)
      useIndicator: true # Émettre des événements d’indicateur (par défaut)
  telegram:
    heartbeat:
      showOk: true # Afficher les accusés de réception OK sur Telegram
  whatsapp:
    accounts:
      work:
        heartbeat:
          showAlerts: false # Supprimer la livraison des alertes pour ce compte
```

Priorité : par compte → par canal → valeurs par défaut du canal → valeurs intégrées par défaut.

### Rôle de chaque indicateur

- `showOk` : envoie un accusé de réception `HEARTBEAT_OK` lorsque le modèle renvoie une réponse composée uniquement de OK.
- `showAlerts` : envoie le contenu de l’alerte lorsque le modèle renvoie une réponse non-OK.
- `useIndicator` : émet des événements d’indicateur pour les surfaces d’état de l’UI.

Si **tous les trois** sont à false, OpenClaw ignore complètement l’exécution du heartbeat (aucun appel au modèle).

### Exemples par canal vs par compte

```yaml
channels:
  defaults:
    heartbeat:
      showOk: false
      showAlerts: true
      useIndicator: true
  slack:
    heartbeat:
      showOk: true # tous les comptes Slack
    accounts:
      ops:
        heartbeat:
          showAlerts: false # supprimer les alertes pour le compte ops uniquement
  telegram:
    heartbeat:
      showOk: true
```

### Modèles courants

| Objectif                                 | Configuration                                                                            |
| ---------------------------------------- | ---------------------------------------------------------------------------------------- |
| Comportement par défaut (OK silencieux, alertes activées) | _(aucune configuration nécessaire)_                                                      |
| Totalement silencieux (aucun message, aucun indicateur) | `channels.defaults.heartbeat: { showOk: false, showAlerts: false, useIndicator: false }` |
| Indicateur uniquement (aucun message)    | `channels.defaults.heartbeat: { showOk: false, showAlerts: false, useIndicator: true }`  |
| OK dans un seul canal                    | `channels.telegram.heartbeat: { showOk: true }`                                          |

## HEARTBEAT.md (facultatif)

Si un fichier `HEARTBEAT.md` existe dans l’espace de travail, le prompt par défaut dit à l’agent de
le lire. Considérez-le comme votre « liste de contrôle heartbeat » : petit, stable, et
sans danger à inclure toutes les 30 minutes.

Lors des exécutions normales, `HEARTBEAT.md` n’est injecté que lorsque les consignes de heartbeat sont
activées pour l’agent par défaut. Désactiver la cadence du heartbeat avec `0m` ou
définir `includeSystemPromptSection: false` l’omet du contexte de bootstrap
normal.

Si `HEARTBEAT.md` existe mais est effectivement vide (seulement des lignes vides et des en-têtes markdown
comme `# Heading`), OpenClaw ignore l’exécution du heartbeat pour économiser des appels API.
Cette omission est signalée comme `reason=empty-heartbeat-file`.
Si le fichier est absent, le heartbeat s’exécute quand même et le modèle décide quoi faire.

Gardez-le petit (courte liste de contrôle ou rappels) pour éviter de gonfler le prompt.

Exemple de `HEARTBEAT.md` :

```md
# Heartbeat checklist

- Quick scan: anything urgent in inboxes?
- If it’s daytime, do a lightweight check-in if nothing else is pending.
- If a task is blocked, write down _what is missing_ and ask Peter next time.
```

### Blocs `tasks:`

`HEARTBEAT.md` prend aussi en charge un petit bloc structuré `tasks:` pour des vérifications
basées sur un intervalle directement dans le heartbeat.

Exemple :

```md
tasks:

- name: inbox-triage
  interval: 30m
  prompt: "Check for urgent unread emails and flag anything time sensitive."
- name: calendar-scan
  interval: 2h
  prompt: "Check for upcoming meetings that need prep or follow-up."

# Instructions supplémentaires

- Garder les alertes courtes.
- Si rien ne nécessite d’attention après toutes les tâches dues, répondre HEARTBEAT_OK.
```

Comportement :

- OpenClaw analyse le bloc `tasks:` et vérifie chaque tâche selon son propre `interval`.
- Seules les tâches **dues** sont incluses dans le prompt de heartbeat pour ce tick.
- Si aucune tâche n’est due, le heartbeat est entièrement ignoré (`reason=no-tasks-due`) pour éviter un appel au modèle inutile.
- Le contenu hors tâche dans `HEARTBEAT.md` est conservé et ajouté comme contexte supplémentaire après la liste des tâches dues.
- Les horodatages de dernière exécution des tâches sont stockés dans l’état de la session (`heartbeatTaskState`), afin que les intervalles survivent aux redémarrages normaux.
- Les horodatages des tâches ne sont avancés qu’après qu’une exécution de heartbeat a terminé son flux de réponse normal. Les exécutions ignorées `empty-heartbeat-file` / `no-tasks-due` ne marquent pas les tâches comme terminées.

Le mode tâche est utile lorsque vous voulez qu’un seul fichier heartbeat contienne plusieurs vérifications périodiques sans devoir les payer toutes à chaque tick.

### L’agent peut-il mettre à jour HEARTBEAT.md ?

Oui — si vous le lui demandez.

`HEARTBEAT.md` est juste un fichier normal dans l’espace de travail de l’agent, vous pouvez donc dire à l’agent
(dans un chat normal) quelque chose comme :

- « Mettez à jour `HEARTBEAT.md` pour ajouter une vérification quotidienne du calendrier. »
- « Réécrivez `HEARTBEAT.md` pour qu’il soit plus court et centré sur les suivis de boîte de réception. »

Si vous voulez que cela se produise de manière proactive, vous pouvez aussi inclure une ligne explicite dans
votre prompt de heartbeat comme : « Si la liste de contrôle devient obsolète, mettez à jour HEARTBEAT.md
avec une meilleure version. »

Note de sécurité : ne mettez pas de secrets (clés API, numéros de téléphone, jetons privés) dans
`HEARTBEAT.md` — il devient partie intégrante du contexte du prompt.

## Réveil manuel (à la demande)

Vous pouvez mettre un événement système en file d’attente et déclencher un heartbeat immédiat avec :

```bash
openclaw system event --text "Check for urgent follow-ups" --mode now
```

Si plusieurs agents ont `heartbeat` configuré, un réveil manuel exécute immédiatement les heartbeats de chacun de ces
agents.

Utilisez `--mode next-heartbeat` pour attendre le prochain tick planifié.

## Livraison du raisonnement (facultatif)

Par défaut, les heartbeats ne livrent que la charge utile finale de « réponse ».

Si vous voulez de la transparence, activez :

- `agents.defaults.heartbeat.includeReasoning: true`

Lorsqu’elle est activée, les heartbeats livrent aussi un message séparé préfixé par
`Reasoning:` (même forme que `/reasoning on`). Cela peut être utile lorsque l’agent
gère plusieurs sessions/codex et que vous voulez voir pourquoi il a décidé de vous
envoyer un signal — mais cela peut aussi révéler plus de détails internes que vous ne le souhaitez. Il est préférable de
le laisser désactivé dans les discussions de groupe.

## Sensibilisation au coût

Les heartbeats exécutent des tours d’agent complets. Des intervalles plus courts consomment plus de jetons. Pour réduire le coût :

- Utilisez `isolatedSession: true` pour éviter d’envoyer tout l’historique de conversation (d’environ 100K jetons à environ 2-5K par exécution).
- Utilisez `lightContext: true` pour limiter les fichiers de bootstrap à `HEARTBEAT.md` uniquement.
- Définissez un `model` moins coûteux (par ex. `ollama/llama3.2:1b`).
- Gardez `HEARTBEAT.md` petit.
- Utilisez `target: "none"` si vous ne voulez que des mises à jour d’état internes.

## Liens associés

- [Automatisation et tâches](/fr/automation) — vue d’ensemble de tous les mécanismes d’automatisation
- [Tâches d’arrière-plan](/fr/automation/tasks) — comment le travail détaché est suivi
- [Fuseau horaire](/fr/concepts/timezone) — comment le fuseau horaire affecte la planification du heartbeat
- [Dépannage](/fr/automation/cron-jobs#troubleshooting) — débogage des problèmes d’automatisation
