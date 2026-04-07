---
read_when:
    - Expliquer l'usage des jetons, les coûts ou les fenêtres de contexte
    - Déboguer la croissance du contexte ou le comportement de compaction
summary: Comment OpenClaw construit le contexte du prompt et rapporte l'usage des jetons + les coûts
title: Usage des jetons et coûts
x-i18n:
    generated_at: "2026-04-07T06:54:30Z"
    model: gpt-5.4
    provider: openai
    source_hash: 0683693d6c6fcde7d5fba236064ba97dd4b317ae6bea3069db969fcd178119d9
    source_path: reference/token-use.md
    workflow: 15
---

# Usage des jetons et coûts

OpenClaw suit les **jetons**, pas les caractères. Les jetons sont spécifiques au modèle, mais la plupart
des modèles de style OpenAI ont en moyenne ~4 caractères par jeton pour du texte en anglais.

## Comment le prompt système est construit

OpenClaw assemble son propre prompt système à chaque exécution. Il inclut :

- Liste des outils + descriptions courtes
- Liste des Skills (métadonnées uniquement ; les instructions sont chargées à la demande avec `read`)
- Instructions d'auto-mise à jour
- Espace de travail + fichiers bootstrap (`AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`, `HEARTBEAT.md`, `BOOTSTRAP.md` lorsqu'il est nouveau, plus `MEMORY.md` lorsqu'il est présent ou `memory.md` comme repli en minuscules). Les gros fichiers sont tronqués par `agents.defaults.bootstrapMaxChars` (par défaut : 20000), et l'injection bootstrap totale est plafonnée par `agents.defaults.bootstrapTotalMaxChars` (par défaut : 150000). Les fichiers `memory/*.md` sont à la demande via les outils de mémoire et ne sont pas injectés automatiquement.
- Heure (UTC + fuseau horaire de l'utilisateur)
- Balises de réponse + comportement heartbeat
- Métadonnées d'exécution (hôte/OS/modèle/réflexion)

Voir le détail complet dans [Prompt système](/fr/concepts/system-prompt).

## Ce qui compte dans la fenêtre de contexte

Tout ce que le modèle reçoit compte dans la limite de contexte :

- Prompt système (toutes les sections listées ci-dessus)
- Historique de conversation (messages utilisateur + assistant)
- Appels d'outils et résultats d'outils
- Pièces jointes/transcriptions (images, audio, fichiers)
- Résumés de compaction et artefacts d'élagage
- Wrappers fournisseur ou en-têtes de sécurité (non visibles, mais quand même comptés)

Pour les images, OpenClaw réduit la résolution des charges utiles d'image de transcription/d'outil avant les appels au fournisseur.
Utilisez `agents.defaults.imageMaxDimensionPx` (par défaut : `1200`) pour ajuster cela :

- Des valeurs plus faibles réduisent généralement l'usage des jetons vision et la taille de la charge utile.
- Des valeurs plus élevées conservent davantage de détails visuels pour l'OCR/les captures d'écran riches en UI.

Pour une ventilation pratique (par fichier injecté, outils, Skills et taille du prompt système), utilisez `/context list` ou `/context detail`. Voir [Contexte](/fr/concepts/context).

## Comment voir l'usage actuel des jetons

Utilisez ceci dans la discussion :

- `/status` → **carte d'état riche en emoji** avec le modèle de la session, l'usage du contexte,
  les jetons d'entrée/sortie de la dernière réponse et le **coût estimé** (clé API uniquement).
- `/usage off|tokens|full` → ajoute un **pied de page d'usage par réponse** à chaque réponse.
  - Persiste par session (stocké comme `responseUsage`).
  - L'authentification OAuth **masque le coût** (jetons uniquement).
- `/usage cost` → affiche un résumé local des coûts à partir des journaux de session OpenClaw.

Autres surfaces :

- **TUI/Web TUI :** `/status` + `/usage` sont pris en charge.
- **CLI :** `openclaw status --usage` et `openclaw channels list` affichent
  des fenêtres de quota fournisseur normalisées (`X% left`, pas des coûts par réponse).
  Fournisseurs actuels avec fenêtre d'usage : Anthropic, GitHub Copilot, Gemini CLI,
  OpenAI Codex, MiniMax, Xiaomi et z.ai.

Les surfaces d'usage normalisent les alias courants de champs natifs fournisseur avant affichage.
Pour le trafic Responses de la famille OpenAI, cela inclut à la fois `input_tokens` /
`output_tokens` et `prompt_tokens` / `completion_tokens`, afin que les noms de champs
spécifiques au transport ne changent pas `/status`, `/usage` ou les résumés de session.
L'usage JSON de Gemini CLI est également normalisé : le texte de réponse provient de `response`, et
`stats.cached` est mappé vers `cacheRead` avec `stats.input_tokens - stats.cached`
utilisé lorsque la CLI omet un champ explicite `stats.input`.
Pour le trafic Responses natif de la famille OpenAI, les alias d'usage WebSocket/SSE sont
normalisés de la même manière, et les totaux se replient sur entrée + sortie normalisées lorsque
`total_tokens` est absent ou vaut `0`.
Lorsque l'instantané de la session en cours est limité, `/status` et `session_status` peuvent
également récupérer les compteurs de jetons/cache et le libellé du modèle d'exécution actif depuis le
journal d'usage de transcription le plus récent. Les valeurs en direct non nulles existantes gardent toujours la
priorité sur les valeurs de repli de transcription, et les totaux de transcription plus grands orientés prompt
peuvent l'emporter lorsque les totaux stockés sont absents ou plus faibles.
L'authentification d'usage pour les fenêtres de quota fournisseur provient de hooks spécifiques au fournisseur lorsqu'ils
sont disponibles ; sinon OpenClaw retombe sur les identifiants OAuth/par clé API correspondants
depuis les profils d'authentification, l'environnement ou la configuration.

## Estimation des coûts (lorsqu'elle est affichée)

Les coûts sont estimés à partir de votre configuration tarifaire du modèle :

```
models.providers.<provider>.models[].cost
```

Il s'agit de **USD par 1M de jetons** pour `input`, `output`, `cacheRead` et
`cacheWrite`. Si la tarification est absente, OpenClaw n'affiche que les jetons. Les jetons OAuth
n'affichent jamais de coût en dollars.

## Impact du TTL du cache et de l'élagage

Le cache de prompt du fournisseur ne s'applique qu'à l'intérieur de la fenêtre TTL du cache. OpenClaw peut
facultativement exécuter un **élagage cache-ttl** : il élague la session une fois le TTL du cache
expiré, puis réinitialise la fenêtre du cache afin que les requêtes suivantes puissent réutiliser le
contexte fraîchement mis en cache au lieu de remettre en cache tout l'historique. Cela permet de garder les coûts
d'écriture de cache plus bas lorsqu'une session reste inactive au-delà du TTL.

Configurez cela dans [Configuration Gateway](/fr/gateway/configuration) et voyez les
détails du comportement dans [Élagage de session](/fr/concepts/session-pruning).

Heartbeat peut garder le cache **chaud** pendant les périodes d'inactivité. Si le TTL du cache de votre modèle
est `1h`, définir l'intervalle heartbeat juste en dessous (par ex. `55m`) peut éviter
de remettre en cache l'intégralité du prompt, réduisant ainsi les coûts d'écriture de cache.

Dans les configurations multi-agents, vous pouvez conserver une configuration de modèle partagée et ajuster le comportement du cache
par agent avec `agents.list[].params.cacheRetention`.

Pour un guide complet réglage par réglage, voir [Mise en cache de prompt](/fr/reference/prompt-caching).

Pour la tarification API Anthropic, les lectures de cache sont significativement moins chères que les jetons
d'entrée, tandis que les écritures de cache sont facturées avec un multiplicateur plus élevé. Voir la tarification de mise en cache de prompt d'Anthropic pour les tarifs et multiplicateurs TTL les plus récents :
[https://docs.anthropic.com/docs/build-with-claude/prompt-caching](https://docs.anthropic.com/docs/build-with-claude/prompt-caching)

### Exemple : garder un cache de 1h chaud avec heartbeat

```yaml
agents:
  defaults:
    model:
      primary: "anthropic/claude-opus-4-6"
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "long"
    heartbeat:
      every: "55m"
```

### Exemple : trafic mixte avec stratégie de cache par agent

```yaml
agents:
  defaults:
    model:
      primary: "anthropic/claude-opus-4-6"
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "long" # référence par défaut pour la plupart des agents
  list:
    - id: "research"
      default: true
      heartbeat:
        every: "55m" # garde le cache long chaud pour les sessions profondes
    - id: "alerts"
      params:
        cacheRetention: "none" # évite les écritures de cache pour les notifications en rafale
```

`agents.list[].params` se fusionne au-dessus des `params` du modèle sélectionné, vous pouvez donc
ne remplacer que `cacheRetention` et hériter inchangées des autres valeurs par défaut du modèle.

### Exemple : activer l'en-tête bêta Anthropic 1M context

La fenêtre de contexte Anthropic 1M est actuellement protégée par une bêta. OpenClaw peut injecter la
valeur `anthropic-beta` requise lorsque vous activez `context1m` sur des modèles Opus
ou Sonnet pris en charge.

```yaml
agents:
  defaults:
    models:
      "anthropic/claude-opus-4-6":
        params:
          context1m: true
```

Cela correspond à l'en-tête bêta `context-1m-2025-08-07` d'Anthropic.

Cela ne s'applique que lorsque `context1m: true` est défini sur cette entrée de modèle.

Exigence : l'identifiant doit être éligible à l'usage de contexte long. Sinon,
Anthropic répond avec une erreur de limitation de débit côté fournisseur pour cette requête.

Si vous authentifiez Anthropic avec des jetons OAuth/d'abonnement (`sk-ant-oat-*`),
OpenClaw ignore l'en-tête bêta `context-1m-*` parce qu'Anthropic rejette actuellement
cette combinaison avec HTTP 401.

## Conseils pour réduire la pression sur les jetons

- Utilisez `/compact` pour résumer les longues sessions.
- Réduisez les sorties d'outils volumineuses dans vos workflows.
- Réduisez `agents.defaults.imageMaxDimensionPx` pour les sessions riches en captures d'écran.
- Gardez les descriptions de Skills courtes (la liste des Skills est injectée dans le prompt).
- Préférez des modèles plus petits pour les travaux verbeux et exploratoires.

Voir [Skills](/fr/tools/skills) pour la formule exacte de surcharge de la liste des skills.
