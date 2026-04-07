---
read_when:
    - Débogage de l'authentification des modèles ou de l'expiration OAuth
    - Documentation de l'authentification ou du stockage des identifiants
summary: 'Authentification des modèles : OAuth, clés API, réutilisation de Claude CLI et setup-token Anthropic'
title: Authentification
x-i18n:
    generated_at: "2026-04-07T06:49:28Z"
    model: gpt-5.4
    provider: openai
    source_hash: 9db0ad9eccd7e3e3ca328adaad260bc4288a8ccdbe2dc0c24d9fd049b7ab9231
    source_path: gateway/authentication.md
    workflow: 15
---

# Authentification (fournisseurs de modèles)

<Note>
Cette page couvre l'authentification des **fournisseurs de modèles** (clés API, OAuth, réutilisation de Claude CLI et setup-token Anthropic). Pour l'authentification de la **connexion gateway** (jeton, mot de passe, trusted-proxy), voir [Configuration](/fr/gateway/configuration) et [Authentification Trusted Proxy](/fr/gateway/trusted-proxy-auth).
</Note>

OpenClaw prend en charge OAuth et les clés API pour les fournisseurs de modèles. Pour les hôtes gateway
toujours actifs, les clés API sont généralement l'option la plus prévisible. Les flux
par abonnement/OAuth sont également pris en charge lorsqu'ils correspondent au modèle
de compte de votre fournisseur.

Voir [/concepts/oauth](/fr/concepts/oauth) pour le flux OAuth complet et la structure
de stockage.
Pour l'authentification basée sur SecretRef (fournisseurs `env`/`file`/`exec`), voir [Gestion des secrets](/fr/gateway/secrets).
Pour les règles d'éligibilité des identifiants/de codes de raison utilisées par `models status --probe`, voir
[Auth Credential Semantics](/fr/auth-credential-semantics).

## Configuration recommandée (clé API, tout fournisseur)

Si vous exécutez une gateway de longue durée, commencez avec une clé API pour le
fournisseur choisi.
Pour Anthropic en particulier, l'authentification par clé API reste la configuration serveur
la plus prévisible, mais OpenClaw prend également en charge la réutilisation d'une connexion Claude CLI locale.

1. Créez une clé API dans la console de votre fournisseur.
2. Placez-la sur l'**hôte gateway** (la machine qui exécute `openclaw gateway`).

```bash
export <PROVIDER>_API_KEY="..."
openclaw models status
```

3. Si la Gateway s'exécute sous systemd/launchd, il est préférable de placer la clé dans
   `~/.openclaw/.env` afin que le démon puisse la lire :

```bash
cat >> ~/.openclaw/.env <<'EOF'
<PROVIDER>_API_KEY=...
EOF
```

Redémarrez ensuite le démon (ou redémarrez votre processus Gateway) et revérifiez :

```bash
openclaw models status
openclaw doctor
```

Si vous préférez ne pas gérer vous-même les variables d'environnement, l'intégration initiale peut stocker
les clés API pour l'usage du démon : `openclaw onboard`.

Voir [Aide](/fr/help) pour des détails sur l'héritage de l'environnement (`env.shellEnv`,
`~/.openclaw/.env`, systemd/launchd).

## Anthropic : compatibilité Claude CLI et jetons

L'authentification Anthropic setup-token reste disponible dans OpenClaw comme chemin
de jeton pris en charge. Depuis, le personnel d'Anthropic nous a indiqué que l'usage de Claude CLI de type OpenClaw est
de nouveau autorisé ; OpenClaw traite donc la réutilisation de Claude CLI et l'usage de `claude -p` comme
approuvés pour cette intégration, sauf si Anthropic publie une nouvelle politique. Lorsque
la réutilisation de Claude CLI est disponible sur l'hôte, c'est désormais le chemin préféré.

Pour les hôtes gateway de longue durée, une clé API Anthropic reste la configuration la plus prévisible.
Si vous voulez réutiliser une connexion Claude existante sur le même hôte, utilisez le
chemin Anthropic Claude CLI dans l'intégration initiale/la configuration.

Saisie manuelle de jeton (tout fournisseur ; écrit dans `auth-profiles.json` + met à jour la configuration) :

```bash
openclaw models auth paste-token --provider openrouter
```

Les références de profil d'authentification sont également prises en charge pour les identifiants statiques :

- les identifiants `api_key` peuvent utiliser `keyRef: { source, provider, id }`
- les identifiants `token` peuvent utiliser `tokenRef: { source, provider, id }`
- les profils en mode OAuth ne prennent pas en charge les identifiants SecretRef ; si `auth.profiles.<id>.mode` est défini sur `"oauth"`, l'entrée `keyRef`/`tokenRef` adossée à SecretRef pour ce profil est rejetée.

Vérification adaptée à l'automatisation (code de sortie `1` si expiré/absent, `2` si expiration proche) :

```bash
openclaw models status --check
```

Sondes d'authentification en direct :

```bash
openclaw models status --probe
```

Remarques :

- Les lignes de sonde peuvent provenir de profils d'authentification, d'identifiants d'environnement ou de `models.json`.
- Si `auth.order.<provider>` explicite omet un profil stocké, la sonde signale
  `excluded_by_auth_order` pour ce profil au lieu de l'essayer.
- Si une authentification existe mais qu'OpenClaw ne peut pas résoudre un candidat de modèle sondable pour
  ce fournisseur, la sonde signale `status: no_model`.
- Les refroidissements de limitation de débit peuvent être portés par modèle. Un profil en refroidissement pour un
  modèle peut encore être utilisable pour un modèle frère chez le même fournisseur.

Les scripts d'exploitation optionnels (systemd/Termux) sont documentés ici :
[Scripts de surveillance de l'authentification](/fr/help/scripts#auth-monitoring-scripts)

## Remarque sur Anthropic

Le backend Anthropic `claude-cli` est de nouveau pris en charge.

- Le personnel d'Anthropic nous a indiqué que ce chemin d'intégration OpenClaw est de nouveau autorisé.
- OpenClaw traite donc la réutilisation de Claude CLI et l'usage de `claude -p` comme approuvés
  pour les exécutions adossées à Anthropic, sauf si Anthropic publie une nouvelle politique.
- Les clés API Anthropic restent le choix le plus prévisible pour les hôtes gateway
  de longue durée et pour un contrôle explicite de la facturation côté serveur.

## Vérification de l'état d'authentification des modèles

```bash
openclaw models status
openclaw doctor
```

## Comportement de rotation des clés API (gateway)

Certains fournisseurs prennent en charge la nouvelle tentative d'une requête avec d'autres clés lorsqu'un appel API
atteint une limite de débit côté fournisseur.

- Ordre de priorité :
  - `OPENCLAW_LIVE_<PROVIDER>_KEY` (remplacement unique)
  - `<PROVIDER>_API_KEYS`
  - `<PROVIDER>_API_KEY`
  - `<PROVIDER>_API_KEY_*`
- Les fournisseurs Google incluent aussi `GOOGLE_API_KEY` comme repli supplémentaire.
- La même liste de clés est dédupliquée avant utilisation.
- OpenClaw ne retente avec la clé suivante que pour les erreurs de limitation de débit (par exemple
  `429`, `rate_limit`, `quota`, `resource exhausted`, `Too many concurrent
requests`, `ThrottlingException`, `concurrency limit reached`, ou
  `workers_ai ... quota limit exceeded`).
- Les erreurs autres que les limitations de débit ne sont pas retentées avec d'autres clés.
- Si toutes les clés échouent, l'erreur finale de la dernière tentative est renvoyée.

## Contrôler quel identifiant est utilisé

### Par session (commande de discussion)

Utilisez `/model <alias-or-id>@<profileId>` pour épingler un identifiant de fournisseur spécifique pour la session en cours (exemples d'identifiants de profil : `anthropic:default`, `anthropic:work`).

Utilisez `/model` (ou `/model list`) pour un sélecteur compact ; utilisez `/model status` pour la vue complète (candidats + profil d'authentification suivant, ainsi que les détails du point de terminaison du fournisseur lorsqu'ils sont configurés).

### Par agent (remplacement CLI)

Définissez un remplacement explicite d'ordre de profil d'authentification pour un agent (stocké dans le `auth-state.json` de cet agent) :

```bash
openclaw models auth order get --provider anthropic
openclaw models auth order set --provider anthropic anthropic:default
openclaw models auth order clear --provider anthropic
```

Utilisez `--agent <id>` pour cibler un agent spécifique ; omettez-le pour utiliser l'agent par défaut configuré.
Lorsque vous déboguez des problèmes d'ordre, `openclaw models status --probe` affiche les profils stockés omis
comme `excluded_by_auth_order` au lieu de les ignorer silencieusement.
Lorsque vous déboguez des problèmes de refroidissement, rappelez-vous que les refroidissements de limitation de débit peuvent être liés
à un identifiant de modèle plutôt qu'à l'ensemble du profil fournisseur.

## Dépannage

### "No credentials found"

Si le profil Anthropic est absent, configurez une clé API Anthropic sur l'**hôte gateway**
ou configurez le chemin Anthropic setup-token, puis revérifiez :

```bash
openclaw models status
```

### Jeton en cours d'expiration/expiré

Exécutez `openclaw models status` pour confirmer quel profil arrive à expiration. Si un
profil de jeton Anthropic est absent ou expiré, actualisez cette configuration via
setup-token ou migrez vers une clé API Anthropic.
