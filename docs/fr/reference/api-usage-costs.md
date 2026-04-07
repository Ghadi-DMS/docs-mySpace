---
read_when:
    - Vous voulez comprendre quelles fonctionnalités peuvent appeler des API payantes
    - Vous devez auditer les clés, les coûts et la visibilité de l'usage
    - Vous expliquez le reporting des coûts de /status ou /usage
summary: Auditer ce qui peut dépenser de l'argent, quelles clés sont utilisées et comment consulter l'usage
title: Usage API et coûts
x-i18n:
    generated_at: "2026-04-07T06:54:00Z"
    model: gpt-5.4
    provider: openai
    source_hash: ab6eefcde9ac014df6cdda7aaa77ef48f16936ab12eaa883d9fe69425a31a2dd
    source_path: reference/api-usage-costs.md
    workflow: 15
---

# Usage API et coûts

Cette documentation liste les **fonctionnalités pouvant invoquer des clés API** et où leurs coûts apparaissent. Elle se concentre sur les
fonctionnalités OpenClaw qui peuvent générer de l'usage fournisseur ou des appels API payants.

## Où les coûts apparaissent (discussion + CLI)

**Instantané du coût par session**

- `/status` affiche le modèle de la session en cours, l'usage du contexte et les jetons de la dernière réponse.
- Si le modèle utilise une **authentification par clé API**, `/status` affiche aussi le **coût estimé** de la dernière réponse.
- Si les métadonnées de session en direct sont limitées, `/status` peut récupérer les compteurs
  de jetons/cache et le libellé du modèle d'exécution actif à partir de la dernière entrée d'usage
  de la transcription. Les valeurs en direct non nulles existantes gardent la priorité, et les totaux
  de transcription dimensionnés au prompt peuvent l'emporter lorsque les totaux stockés sont absents ou plus faibles.

**Pied de page du coût par message**

- `/usage full` ajoute un pied de page d'usage à chaque réponse, y compris le **coût estimé** (clé API uniquement).
- `/usage tokens` affiche uniquement les jetons ; les flux OAuth/jeton et CLI de type abonnement masquent le coût en dollars.
- Remarque Gemini CLI : lorsque la CLI renvoie une sortie JSON, OpenClaw lit l'usage depuis
  `stats`, normalise `stats.cached` en `cacheRead`, et dérive les jetons d'entrée depuis
  `stats.input_tokens - stats.cached` lorsque nécessaire.

Remarque Anthropic : le personnel d'Anthropic nous a indiqué que l'usage de Claude CLI de type OpenClaw est
de nouveau autorisé ; OpenClaw traite donc la réutilisation de Claude CLI et l'usage de `claude -p` comme
approuvés pour cette intégration, sauf si Anthropic publie une nouvelle politique.
Anthropic n'expose toujours pas d'estimation en dollars par message que OpenClaw puisse
afficher dans `/usage full`.

**Fenêtres d'usage CLI (quotas fournisseur)**

- `openclaw status --usage` et `openclaw channels list` affichent les **fenêtres d'usage**
  du fournisseur (instantanés de quota, pas des coûts par message).
- La sortie lisible est normalisée en `X% left` pour tous les fournisseurs.
- Fournisseurs actuels avec fenêtre d'usage : Anthropic, GitHub Copilot, Gemini CLI,
  OpenAI Codex, MiniMax, Xiaomi et z.ai.
- Remarque MiniMax : ses champs bruts `usage_percent` / `usagePercent` signifient le
  quota restant ; OpenClaw les inverse donc avant affichage. Les champs basés sur un comptage gardent
  néanmoins la priorité lorsqu'ils sont présents. Si le fournisseur renvoie `model_remains`, OpenClaw préfère l'entrée du modèle de discussion, dérive le libellé de fenêtre à partir des horodatages si nécessaire, et
  inclut le nom du modèle dans le libellé du forfait.
- L'authentification d'usage pour ces fenêtres de quota provient de hooks spécifiques au fournisseur lorsqu'ils
  sont disponibles ; sinon OpenClaw retombe sur les identifiants OAuth/par clé API correspondants provenant des profils d'authentification, de l'environnement ou de la configuration.

Voir [Usage des jetons et coûts](/fr/reference/token-use) pour les détails et des exemples.

## Comment les clés sont découvertes

OpenClaw peut récupérer les identifiants depuis :

- **Profils d'authentification** (par agent, stockés dans `auth-profiles.json`).
- **Variables d'environnement** (par ex. `OPENAI_API_KEY`, `BRAVE_API_KEY`, `FIRECRAWL_API_KEY`).
- **Configuration** (`models.providers.*.apiKey`, `plugins.entries.*.config.webSearch.apiKey`,
  `plugins.entries.firecrawl.config.webFetch.apiKey`, `memorySearch.*`,
  `talk.providers.*.apiKey`).
- **Skills** (`skills.entries.<name>.apiKey`) qui peuvent exporter des clés dans l'environnement du processus de la skill.

## Fonctionnalités pouvant dépenser des clés

### 1) Réponses de modèles du cœur (discussion + outils)

Chaque réponse ou appel d'outil utilise le **fournisseur de modèle actuel** (OpenAI, Anthropic, etc.). C'est la
source principale d'usage et de coût.

Cela inclut aussi les fournisseurs hébergés de type abonnement qui facturent toujours en dehors de
l'UI locale d'OpenClaw, comme **OpenAI Codex**, **Alibaba Cloud Model Studio
Coding Plan**, **MiniMax Coding Plan**, **Z.AI / GLM Coding Plan** et
le chemin de connexion Claude d'Anthropic dans OpenClaw avec **Extra Usage** activé.

Voir [Modèles](/fr/providers/models) pour la configuration de tarification et [Usage des jetons et coûts](/fr/reference/token-use) pour l'affichage.

### 2) Compréhension des médias (audio/image/vidéo)

Les médias entrants peuvent être résumés/transcrits avant l'exécution de la réponse. Cela utilise les API de modèle/fournisseur.

- Audio : OpenAI / Groq / Deepgram / Google / Mistral.
- Image : OpenAI / OpenRouter / Anthropic / Google / MiniMax / Moonshot / Qwen / Z.AI.
- Vidéo : Google / Qwen / Moonshot.

Voir [Compréhension des médias](/fr/nodes/media-understanding).

### 3) Génération d'images et de vidéos

Les capacités de génération partagées peuvent également dépenser des clés fournisseur :

- Génération d'images : OpenAI / Google / fal / MiniMax
- Génération vidéo : Qwen

La génération d'images peut déduire un fournisseur par défaut adossé à l'authentification lorsque
`agents.defaults.imageGenerationModel` n'est pas défini. La génération vidéo nécessite actuellement
un `agents.defaults.videoGenerationModel` explicite tel que
`qwen/wan2.6-t2v`.

Voir [Génération d'images](/fr/tools/image-generation), [Qwen Cloud](/fr/providers/qwen)
et [Modèles](/fr/concepts/models).

### 4) Embeddings mémoire + recherche sémantique

La recherche mémoire sémantique utilise des **API d'embedding** lorsqu'elle est configurée pour des fournisseurs distants :

- `memorySearch.provider = "openai"` → embeddings OpenAI
- `memorySearch.provider = "gemini"` → embeddings Gemini
- `memorySearch.provider = "voyage"` → embeddings Voyage
- `memorySearch.provider = "mistral"` → embeddings Mistral
- `memorySearch.provider = "ollama"` → embeddings Ollama (local/autohébergé ; généralement sans facturation d'API hébergée)
- Repli facultatif vers un fournisseur distant si les embeddings locaux échouent

Vous pouvez tout garder en local avec `memorySearch.provider = "local"` (aucun usage d'API).

Voir [Mémoire](/fr/concepts/memory).

### 5) Outil de recherche web

`web_search` peut entraîner des frais d'usage selon votre fournisseur :

- **Brave Search API** : `BRAVE_API_KEY` ou `plugins.entries.brave.config.webSearch.apiKey`
- **Exa** : `EXA_API_KEY` ou `plugins.entries.exa.config.webSearch.apiKey`
- **Firecrawl** : `FIRECRAWL_API_KEY` ou `plugins.entries.firecrawl.config.webSearch.apiKey`
- **Gemini (Google Search)** : `GEMINI_API_KEY` ou `plugins.entries.google.config.webSearch.apiKey`
- **Grok (xAI)** : `XAI_API_KEY` ou `plugins.entries.xai.config.webSearch.apiKey`
- **Kimi (Moonshot)** : `KIMI_API_KEY`, `MOONSHOT_API_KEY` ou `plugins.entries.moonshot.config.webSearch.apiKey`
- **MiniMax Search** : `MINIMAX_CODE_PLAN_KEY`, `MINIMAX_CODING_API_KEY`, `MINIMAX_API_KEY` ou `plugins.entries.minimax.config.webSearch.apiKey`
- **Ollama Web Search** : sans clé par défaut, mais nécessite un hôte Ollama accessible plus `ollama signin` ; peut aussi réutiliser l'authentification bearer normale du fournisseur Ollama lorsque l'hôte l'exige
- **Perplexity Search API** : `PERPLEXITY_API_KEY`, `OPENROUTER_API_KEY` ou `plugins.entries.perplexity.config.webSearch.apiKey`
- **Tavily** : `TAVILY_API_KEY` ou `plugins.entries.tavily.config.webSearch.apiKey`
- **DuckDuckGo** : repli sans clé (pas de facturation API, mais non officiel et basé sur HTML)
- **SearXNG** : `SEARXNG_BASE_URL` ou `plugins.entries.searxng.config.webSearch.baseUrl` (sans clé/autohébergé ; pas de facturation d'API hébergée)

Les chemins fournisseur hérités `tools.web.search.*` continuent d'être chargés via la couche de compatibilité temporaire, mais ils ne constituent plus la surface de configuration recommandée.

**Crédit gratuit Brave Search :** Chaque forfait Brave comprend 5 $/mois de crédit gratuit
renouvelable. Le forfait Search coûte 5 $ pour 1 000 requêtes, donc ce crédit couvre
1 000 requêtes/mois sans frais. Définissez votre limite d'usage dans le tableau de bord Brave
pour éviter des frais inattendus.

Voir [Outils web](/fr/tools/web).

### 5) Outil de récupération web (Firecrawl)

`web_fetch` peut appeler **Firecrawl** lorsqu'une clé API est présente :

- `FIRECRAWL_API_KEY` ou `plugins.entries.firecrawl.config.webFetch.apiKey`

Si Firecrawl n'est pas configuré, l'outil retombe sur un fetch direct + readability (pas d'API payante).

Voir [Outils web](/fr/tools/web).

### 6) Instantanés d'usage fournisseur (état/santé)

Certaines commandes d'état appellent les **points de terminaison d'usage fournisseur** pour afficher les fenêtres de quota ou la santé d'authentification.
Il s'agit généralement d'appels de faible volume, mais ils touchent quand même les API fournisseur :

- `openclaw status --usage`
- `openclaw models status --json`

Voir [CLI des modèles](/cli/models).

### 7) Résumé de sauvegarde pour la compaction

La sauvegarde de compaction peut résumer l'historique de session en utilisant le **modèle actuel**, ce qui
invoque les API du fournisseur lorsqu'elle s'exécute.

Voir [Gestion de session + compaction](/fr/reference/session-management-compaction).

### 8) Scan / sonde de modèles

`openclaw models scan` peut sonder les modèles OpenRouter et utilise `OPENROUTER_API_KEY` lorsque
la sonde est activée.

Voir [CLI des modèles](/cli/models).

### 9) Talk (parole)

Le mode Talk peut invoquer **ElevenLabs** lorsqu'il est configuré :

- `ELEVENLABS_API_KEY` ou `talk.providers.elevenlabs.apiKey`

Voir [Mode Talk](/fr/nodes/talk).

### 10) Skills (API tierces)

Les Skills peuvent stocker `apiKey` dans `skills.entries.<name>.apiKey`. Si une skill utilise cette clé pour des
API externes, cela peut entraîner des coûts selon le fournisseur de la skill.

Voir [Skills](/fr/tools/skills).
