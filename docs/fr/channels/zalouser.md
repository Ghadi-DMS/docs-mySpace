---
read_when:
    - Configurer Zalo Personnel pour OpenClaw
    - Déboguer la connexion ou le flux de messages de Zalo Personnel
summary: Prise en charge du compte personnel Zalo via zca-js natif (connexion par QR), capacités et configuration
title: Zalo Personnel
x-i18n:
    generated_at: "2026-04-08T02:14:14Z"
    model: gpt-5.4
    provider: openai
    source_hash: 08f50edb2f4c6fe24972efe5e321f5fd0572c7d29af5c1db808151c7c943dc66
    source_path: channels/zalouser.md
    workflow: 15
---

# Zalo Personnel (non officiel)

Statut : expérimental. Cette intégration automatise un **compte Zalo personnel** via `zca-js` natif dans OpenClaw.

> **Avertissement :** Il s'agit d'une intégration non officielle et elle peut entraîner la suspension ou le bannissement du compte. Utilisez-la à vos propres risques.

## Plugin inclus

Zalo Personnel est livré comme plugin inclus dans les versions actuelles d'OpenClaw, donc les builds empaquetés normaux ne nécessitent pas d'installation séparée.

Si vous utilisez une ancienne build ou une installation personnalisée qui exclut Zalo Personnel, installez-le manuellement :

- Installer via la CLI : `openclaw plugins install @openclaw/zalouser`
- Ou depuis une extraction des sources : `openclaw plugins install ./path/to/local/zalouser-plugin`
- Détails : [Plugins](/fr/tools/plugin)

Aucun binaire CLI externe `zca`/`openzca` n'est requis.

## Configuration rapide (débutant)

1. Assurez-vous que le plugin Zalo Personnel est disponible.
   - Les versions empaquetées actuelles d'OpenClaw l'incluent déjà.
   - Les installations anciennes/personnalisées peuvent l'ajouter manuellement avec les commandes ci-dessus.
2. Connectez-vous (QR, sur la machine Gateway) :
   - `openclaw channels login --channel zalouser`
   - Scannez le code QR avec l'application mobile Zalo.
3. Activez le canal :

```json5
{
  channels: {
    zalouser: {
      enabled: true,
      dmPolicy: "pairing",
    },
  },
}
```

4. Redémarrez la Gateway (ou terminez la configuration).
5. L'accès par message privé utilise par défaut l'appairage ; approuvez le code d'appairage lors du premier contact.

## Ce que c'est

- Fonctionne entièrement dans le processus via `zca-js`.
- Utilise des écouteurs d'événements natifs pour recevoir les messages entrants.
- Envoie les réponses directement via l'API JS (texte/média/lien).
- Conçu pour les cas d'usage « compte personnel » où l'API Bot Zalo n'est pas disponible.

## Nommage

L'identifiant du canal est `zalouser` afin de rendre explicite le fait que cela automatise un **compte utilisateur Zalo personnel** (non officiel). Nous réservons `zalo` à une éventuelle future intégration officielle de l'API Zalo.

## Trouver des identifiants (répertoire)

Utilisez la CLI de répertoire pour découvrir les pairs/groupes et leurs identifiants :

```bash
openclaw directory self --channel zalouser
openclaw directory peers list --channel zalouser --query "name"
openclaw directory groups list --channel zalouser --query "work"
```

## Limites

- Le texte sortant est découpé en blocs d'environ 2000 caractères (limites du client Zalo).
- Le streaming est bloqué par défaut.

## Contrôle d'accès (messages privés)

`channels.zalouser.dmPolicy` prend en charge : `pairing | allowlist | open | disabled` (par défaut : `pairing`).

`channels.zalouser.allowFrom` accepte des identifiants ou des noms d'utilisateur. Pendant la configuration, les noms sont résolus en identifiants à l'aide de la recherche de contacts du plugin dans le processus.

Approuvez via :

- `openclaw pairing list zalouser`
- `openclaw pairing approve zalouser <code>`

## Accès aux groupes (facultatif)

- Par défaut : `channels.zalouser.groupPolicy = "open"` (groupes autorisés). Utilisez `channels.defaults.groupPolicy` pour remplacer la valeur par défaut lorsqu'elle n'est pas définie.
- Limitez à une liste d'autorisation avec :
  - `channels.zalouser.groupPolicy = "allowlist"`
  - `channels.zalouser.groups` (les clés doivent être des identifiants de groupe stables ; les noms sont résolus en identifiants au démarrage lorsque c'est possible)
  - `channels.zalouser.groupAllowFrom` (contrôle quels expéditeurs, dans les groupes autorisés, peuvent déclencher le bot)
- Bloquez tous les groupes : `channels.zalouser.groupPolicy = "disabled"`.
- L'assistant de configuration peut demander des listes d'autorisation de groupes.
- Au démarrage, OpenClaw résout les noms de groupe/utilisateur dans les listes d'autorisation en identifiants et journalise la correspondance.
- La correspondance avec la liste d'autorisation de groupes se fait par identifiant uniquement par défaut. Les noms non résolus sont ignorés pour l'authentification, sauf si `channels.zalouser.dangerouslyAllowNameMatching: true` est activé.
- `channels.zalouser.dangerouslyAllowNameMatching: true` est un mode de compatibilité de dernier recours qui réactive la correspondance avec des noms de groupe modifiables.
- Si `groupAllowFrom` n'est pas défini, le runtime revient à `allowFrom` pour les vérifications d'expéditeur dans les groupes.
- Les vérifications d'expéditeur s'appliquent à la fois aux messages de groupe normaux et aux commandes de contrôle (par exemple `/new`, `/reset`).

Exemple :

```json5
{
  channels: {
    zalouser: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["1471383327500481391"],
      groups: {
        "123456789": { allow: true },
        "Work Chat": { allow: true },
      },
    },
  },
}
```

### Contrôle des mentions dans les groupes

- `channels.zalouser.groups.<group>.requireMention` contrôle si les réponses dans les groupes nécessitent une mention.
- Ordre de résolution : identifiant/nom exact du groupe -> slug de groupe normalisé -> `*` -> valeur par défaut (`true`).
- Cela s'applique à la fois aux groupes de la liste d'autorisation et au mode groupe ouvert.
- Citer un message du bot compte comme une mention implicite pour l'activation dans le groupe.
- Les commandes de contrôle autorisées (par exemple `/new`) peuvent contourner l'exigence de mention.
- Lorsqu'un message de groupe est ignoré parce qu'une mention est requise, OpenClaw le stocke comme historique de groupe en attente et l'inclut dans le prochain message de groupe traité.
- La limite d'historique de groupe utilise par défaut `messages.groupChat.historyLimit` (repli : `50`). Vous pouvez la remplacer par compte avec `channels.zalouser.historyLimit`.

Exemple :

```json5
{
  channels: {
    zalouser: {
      groupPolicy: "allowlist",
      groups: {
        "*": { allow: true, requireMention: true },
        "Work Chat": { allow: true, requireMention: false },
      },
    },
  },
}
```

## Multi-compte

Les comptes sont mappés à des profils `zalouser` dans l'état d'OpenClaw. Exemple :

```json5
{
  channels: {
    zalouser: {
      enabled: true,
      defaultAccount: "default",
      accounts: {
        work: { enabled: true, profile: "work" },
      },
    },
  },
}
```

## Saisie, réactions et accusés de réception

- OpenClaw envoie un événement de saisie avant de transmettre une réponse (au mieux).
- L'action de réaction aux messages `react` est prise en charge pour `zalouser` dans les actions de canal.
  - Utilisez `remove: true` pour supprimer un emoji de réaction spécifique d'un message.
  - Sémantique des réactions : [Réactions](/fr/tools/reactions)
- Pour les messages entrants qui incluent des métadonnées d'événement, OpenClaw envoie des accusés de réception « livré » et « vu » (au mieux).

## Dépannage

**La connexion n'est pas conservée :**

- `openclaw channels status --probe`
- Reconnectez-vous : `openclaw channels logout --channel zalouser && openclaw channels login --channel zalouser`

**Le nom de la liste d'autorisation/du groupe n'a pas été résolu :**

- Utilisez des identifiants numériques dans `allowFrom`/`groupAllowFrom`/`groups`, ou les noms exacts d'amis/groupes.

**Mise à niveau depuis l'ancienne configuration basée sur la CLI :**

- Supprimez toute hypothèse concernant un ancien processus externe `zca`.
- Le canal fonctionne désormais entièrement dans OpenClaw sans binaires CLI externes.

## Lié

- [Vue d'ensemble des canaux](/fr/channels) — tous les canaux pris en charge
- [Appairage](/fr/channels/pairing) — authentification par message privé et flux d'appairage
- [Groupes](/fr/channels/groups) — comportement du chat de groupe et contrôle des mentions
- [Routage des canaux](/fr/channels/channel-routing) — routage des sessions pour les messages
- [Sécurité](/fr/gateway/security) — modèle d'accès et durcissement
