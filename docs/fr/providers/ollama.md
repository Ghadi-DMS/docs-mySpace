---
read_when:
    - Vous souhaitez exécuter OpenClaw avec des modèles cloud ou locaux via Ollama
    - Vous avez besoin d'instructions pour l'installation et la configuration d'Ollama
summary: Exécuter OpenClaw avec Ollama (modèles cloud et locaux)
title: Ollama
x-i18n:
    generated_at: "2026-04-08T06:51:18Z"
    model: gpt-5.4
    provider: openai
    source_hash: d3295a7c879d3636a2ffdec05aea6e670e54a990ef52bd9b0cae253bc24aa3f7
    source_path: providers/ollama.md
    workflow: 15
---

# Ollama

Ollama est un environnement d'exécution LLM local qui facilite l'exécution de modèles open source sur votre machine. OpenClaw s'intègre à l'API native d'Ollama (`/api/chat`), prend en charge le streaming et l'appel d'outils, et peut détecter automatiquement les modèles Ollama locaux lorsque vous activez `OLLAMA_API_KEY` (ou un profil d'authentification) et que vous ne définissez pas d'entrée `models.providers.ollama` explicite.

<Warning>
**Utilisateurs d'Ollama à distance** : N'utilisez pas l'URL compatible OpenAI `/v1` (`http://host:11434/v1`) avec OpenClaw. Cela casse l'appel d'outils et les modèles peuvent produire le JSON brut des outils sous forme de texte brut. Utilisez à la place l'URL de l'API native d'Ollama : `baseUrl: "http://host:11434"` (sans `/v1`).
</Warning>

## Démarrage rapide

### Intégration guidée (recommandée)

Le moyen le plus rapide de configurer Ollama consiste à passer par l'intégration guidée :

```bash
openclaw onboard
```

Sélectionnez **Ollama** dans la liste des fournisseurs. L'intégration guidée va :

1. Demander l'URL de base Ollama à laquelle votre instance est accessible (par défaut `http://127.0.0.1:11434`).
2. Vous permettre de choisir **Cloud + Local** (modèles cloud et modèles locaux) ou **Local** (modèles locaux uniquement).
3. Ouvrir un flux de connexion dans le navigateur si vous choisissez **Cloud + Local** et que vous n'êtes pas connecté à ollama.com.
4. Détecter les modèles disponibles et suggérer des valeurs par défaut.
5. Télécharger automatiquement le modèle sélectionné s'il n'est pas disponible localement.

Le mode non interactif est également pris en charge :

```bash
openclaw onboard --non-interactive \
  --auth-choice ollama \
  --accept-risk
```

Vous pouvez éventuellement spécifier une URL de base ou un modèle personnalisés :

```bash
openclaw onboard --non-interactive \
  --auth-choice ollama \
  --custom-base-url "http://ollama-host:11434" \
  --custom-model-id "qwen3.5:27b" \
  --accept-risk
```

### Configuration manuelle

1. Installez Ollama : [https://ollama.com/download](https://ollama.com/download)

2. Téléchargez un modèle local si vous souhaitez une inférence locale :

```bash
ollama pull gemma4
# ou
ollama pull gpt-oss:20b
# ou
ollama pull llama3.3
```

3. Si vous souhaitez aussi des modèles cloud, connectez-vous :

```bash
ollama signin
```

4. Exécutez l'intégration guidée et choisissez `Ollama` :

```bash
openclaw onboard
```

- `Local` : modèles locaux uniquement
- `Cloud + Local` : modèles locaux plus modèles cloud
- Les modèles cloud tels que `kimi-k2.5:cloud`, `minimax-m2.7:cloud` et `glm-5.1:cloud` ne nécessitent **pas** de `ollama pull` local

OpenClaw suggère actuellement :

- valeur par défaut locale : `gemma4`
- valeurs par défaut cloud : `kimi-k2.5:cloud`, `minimax-m2.7:cloud`, `glm-5.1:cloud`

5. Si vous préférez une configuration manuelle, activez directement Ollama pour OpenClaw (n'importe quelle valeur convient ; Ollama n'exige pas de vraie clé) :

```bash
# Définir la variable d'environnement
export OLLAMA_API_KEY="ollama-local"

# Ou configurer dans votre fichier de configuration
openclaw config set models.providers.ollama.apiKey "ollama-local"
```

6. Inspectez ou changez de modèle :

```bash
openclaw models list
openclaw models set ollama/gemma4
```

7. Ou définissez la valeur par défaut dans la configuration :

```json5
{
  agents: {
    defaults: {
      model: { primary: "ollama/gemma4" },
    },
  },
}
```

## Détection des modèles (fournisseur implicite)

Lorsque vous définissez `OLLAMA_API_KEY` (ou un profil d'authentification) et que vous **ne** définissez **pas** `models.providers.ollama`, OpenClaw détecte les modèles depuis l'instance Ollama locale à l'adresse `http://127.0.0.1:11434` :

- Interroge `/api/tags`
- Utilise des recherches `/api/show` en mode best effort pour lire `contextWindow` et détecter les capacités (y compris la vision) lorsque disponibles
- Les modèles avec une capacité `vision` signalée par `/api/show` sont marqués comme compatibles avec les images (`input: ["text", "image"]`), afin qu'OpenClaw injecte automatiquement les images dans l'invite pour ces modèles
- Marque `reasoning` à l'aide d'une heuristique basée sur le nom du modèle (`r1`, `reasoning`, `think`)
- Définit `maxTokens` sur le plafond de jetons maximum Ollama par défaut utilisé par OpenClaw
- Définit tous les coûts à `0`

Cela évite les entrées de modèles manuelles tout en conservant un catalogue aligné sur l'instance Ollama locale.

Pour voir quels modèles sont disponibles :

```bash
ollama list
openclaw models list
```

Pour ajouter un nouveau modèle, téléchargez-le simplement avec Ollama :

```bash
ollama pull mistral
```

Le nouveau modèle sera automatiquement détecté et disponible à l'utilisation.

Si vous définissez explicitement `models.providers.ollama`, la détection automatique est ignorée et vous devez définir les modèles manuellement (voir ci-dessous).

## Configuration

### Configuration de base (détection implicite)

Le moyen le plus simple d'activer Ollama consiste à utiliser une variable d'environnement :

```bash
export OLLAMA_API_KEY="ollama-local"
```

### Configuration explicite (modèles manuels)

Utilisez une configuration explicite lorsque :

- Ollama s'exécute sur un autre hôte/port.
- Vous souhaitez imposer des fenêtres de contexte ou des listes de modèles spécifiques.
- Vous souhaitez des définitions de modèles entièrement manuelles.

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "http://ollama-host:11434",
        apiKey: "ollama-local",
        api: "ollama",
        models: [
          {
            id: "gpt-oss:20b",
            name: "GPT-OSS 20B",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 8192,
            maxTokens: 8192 * 10
          }
        ]
      }
    }
  }
}
```

Si `OLLAMA_API_KEY` est défini, vous pouvez omettre `apiKey` dans l'entrée du fournisseur et OpenClaw le renseignera pour les vérifications de disponibilité.

### URL de base personnalisée (configuration explicite)

Si Ollama s'exécute sur un hôte ou un port différent (la configuration explicite désactive la détection automatique, vous devez donc définir les modèles manuellement) :

```json5
{
  models: {
    providers: {
      ollama: {
        apiKey: "ollama-local",
        baseUrl: "http://ollama-host:11434", // Pas de /v1 - utilisez l'URL de l'API native d'Ollama
        api: "ollama", // Définissez explicitement pour garantir le comportement natif d'appel d'outils
      },
    },
  },
}
```

<Warning>
N'ajoutez pas `/v1` à l'URL. Le chemin `/v1` utilise le mode compatible OpenAI, dans lequel l'appel d'outils n'est pas fiable. Utilisez l'URL de base d'Ollama sans suffixe de chemin.
</Warning>

### Sélection du modèle

Une fois configurés, tous vos modèles Ollama sont disponibles :

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "ollama/gpt-oss:20b",
        fallbacks: ["ollama/llama3.3", "ollama/qwen2.5-coder:32b"],
      },
    },
  },
}
```

## Modèles cloud

Les modèles cloud vous permettent d'exécuter des modèles hébergés dans le cloud (par exemple `kimi-k2.5:cloud`, `minimax-m2.7:cloud`, `glm-5.1:cloud`) en parallèle de vos modèles locaux.

Pour utiliser les modèles cloud, sélectionnez le mode **Cloud + Local** pendant la configuration. L'assistant vérifie si vous êtes connecté et ouvre un flux de connexion dans le navigateur si nécessaire. Si l'authentification ne peut pas être vérifiée, l'assistant revient aux valeurs par défaut des modèles locaux.

Vous pouvez également vous connecter directement sur [ollama.com/signin](https://ollama.com/signin).

## Ollama Web Search

OpenClaw prend également en charge **Ollama Web Search** comme fournisseur `web_search` intégré.

- Il utilise votre hôte Ollama configuré (`models.providers.ollama.baseUrl` lorsqu'il est défini, sinon `http://127.0.0.1:11434`).
- Il ne nécessite aucune clé.
- Ollama doit être en cours d'exécution et connecté avec `ollama signin`.

Choisissez **Ollama Web Search** pendant `openclaw onboard` ou
`openclaw configure --section web`, ou définissez :

```json5
{
  tools: {
    web: {
      search: {
        provider: "ollama",
      },
    },
  },
}
```

Pour la configuration complète et le détail du comportement, consultez [Ollama Web Search](/fr/tools/ollama-search).

## Avancé

### Modèles de raisonnement

OpenClaw traite par défaut les modèles dont le nom contient `deepseek-r1`, `reasoning` ou `think` comme capables de raisonnement :

```bash
ollama pull deepseek-r1:32b
```

### Coûts des modèles

Ollama est gratuit et s'exécute localement, donc tous les coûts des modèles sont définis à 0 $.

### Configuration du streaming

L'intégration Ollama d'OpenClaw utilise par défaut l'**API native d'Ollama** (`/api/chat`), qui prend entièrement en charge le streaming et l'appel d'outils simultanément. Aucune configuration particulière n'est nécessaire.

#### Ancien mode compatible OpenAI

<Warning>
**L'appel d'outils n'est pas fiable en mode compatible OpenAI.** Utilisez ce mode uniquement si vous avez besoin du format OpenAI pour un proxy et que vous ne dépendez pas du comportement natif d'appel d'outils.
</Warning>

Si vous devez utiliser à la place le point de terminaison compatible OpenAI (par exemple derrière un proxy qui ne prend en charge que le format OpenAI), définissez explicitement `api: "openai-completions"` :

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "http://ollama-host:11434/v1",
        api: "openai-completions",
        injectNumCtxForOpenAICompat: true, // par défaut : true
        apiKey: "ollama-local",
        models: [...]
      }
    }
  }
}
```

Ce mode peut ne pas prendre en charge simultanément le streaming et l'appel d'outils. Vous devrez peut-être désactiver le streaming avec `params: { streaming: false }` dans la configuration du modèle.

Lorsque `api: "openai-completions"` est utilisé avec Ollama, OpenClaw injecte `options.num_ctx` par défaut afin qu'Ollama ne revienne pas silencieusement à une fenêtre de contexte de 4096. Si votre proxy/amont rejette les champs `options` inconnus, désactivez ce comportement :

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "http://ollama-host:11434/v1",
        api: "openai-completions",
        injectNumCtxForOpenAICompat: false,
        apiKey: "ollama-local",
        models: [...]
      }
    }
  }
}
```

### Fenêtres de contexte

Pour les modèles détectés automatiquement, OpenClaw utilise la fenêtre de contexte signalée par Ollama lorsqu'elle est disponible ; sinon, il revient à la fenêtre de contexte Ollama par défaut utilisée par OpenClaw. Vous pouvez remplacer `contextWindow` et `maxTokens` dans la configuration explicite du fournisseur.

## Dépannage

### Ollama non détecté

Assurez-vous qu'Ollama est en cours d'exécution, que vous avez défini `OLLAMA_API_KEY` (ou un profil d'authentification) et que vous n'avez **pas** défini d'entrée `models.providers.ollama` explicite :

```bash
ollama serve
```

Et que l'API est accessible :

```bash
curl http://localhost:11434/api/tags
```

### Aucun modèle disponible

Si votre modèle n'est pas répertorié, soit :

- téléchargez le modèle localement, ou
- définissez le modèle explicitement dans `models.providers.ollama`.

Pour ajouter des modèles :

```bash
ollama list  # Voir ce qui est installé
ollama pull gemma4
ollama pull gpt-oss:20b
ollama pull llama3.3     # Ou un autre modèle
```

### Connexion refusée

Vérifiez qu'Ollama s'exécute sur le bon port :

```bash
# Vérifier si Ollama est en cours d'exécution
ps aux | grep ollama

# Ou redémarrer Ollama
ollama serve
```

## Voir aussi

- [Fournisseurs de modèles](/fr/concepts/model-providers) - Vue d'ensemble de tous les fournisseurs
- [Sélection des modèles](/fr/concepts/models) - Comment choisir des modèles
- [Configuration](/fr/gateway/configuration) - Référence complète de la configuration
