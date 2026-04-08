---
read_when:
    - Exécution de scripts depuis le dépôt
    - Ajout ou modification de scripts sous ./scripts
summary: 'Scripts du dépôt : objectif, portée et notes de sécurité'
title: Scripts
x-i18n:
    generated_at: "2026-04-08T02:15:25Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3ecf1e9327929948fb75f80e306963af49b353c0aa8d3b6fa532ca964ff8b975
    source_path: help/scripts.md
    workflow: 15
---

# Scripts

Le répertoire `scripts/` contient des scripts d’aide pour les workflows locaux et les tâches d’exploitation.
Utilisez-les lorsqu’une tâche est clairement liée à un script ; sinon, préférez la CLI.

## Conventions

- Les scripts sont **facultatifs** sauf s’ils sont référencés dans la documentation ou dans des checklists de publication.
- Préférez les surfaces CLI lorsqu’elles existent (exemple : la surveillance de l’authentification utilise `openclaw models status --check`).
- Considérez que les scripts sont spécifiques à l’hôte ; lisez-les avant de les exécuter sur une nouvelle machine.

## Scripts de surveillance de l’authentification

La surveillance de l’authentification est couverte dans [Authentification](/fr/gateway/authentication). Les scripts sous `scripts/` sont des extras facultatifs pour les workflows de téléphone systemd/Termux.

## Assistant de lecture GitHub

Utilisez `scripts/gh-read` lorsque vous voulez que `gh` utilise un jeton d’installation GitHub App pour des appels de lecture à portée dépôt, tout en laissant le `gh` normal sur votre connexion personnelle pour les actions d’écriture.

Variables d’environnement requises :

- `OPENCLAW_GH_READ_APP_ID`
- `OPENCLAW_GH_READ_PRIVATE_KEY_FILE`

Variables d’environnement facultatives :

- `OPENCLAW_GH_READ_INSTALLATION_ID` lorsque vous souhaitez ignorer la recherche d’installation basée sur le dépôt
- `OPENCLAW_GH_READ_PERMISSIONS` comme remplacement séparé par des virgules pour le sous-ensemble d’autorisations de lecture à demander

Ordre de résolution du dépôt :

- `gh ... -R owner/repo`
- `GH_REPO`
- `git remote origin`

Exemples :

- `scripts/gh-read pr view 123`
- `scripts/gh-read run list -R openclaw/openclaw`
- `scripts/gh-read api repos/openclaw/openclaw/pulls/123`

## Lors de l’ajout de scripts

- Gardez les scripts ciblés et documentés.
- Ajoutez une courte entrée dans la documentation pertinente (ou créez-en une si elle manque).
