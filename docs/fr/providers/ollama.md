---
read_when:
    - Vous souhaitez exécuter OpenClaw avec des modèles cloud ou locaux via Ollama
    - Vous avez besoin d'instructions de configuration et de paramétrage pour Ollama
summary: Exécuter OpenClaw avec Ollama (modèles cloud et locaux)
title: Ollama
x-i18n:
    generated_at: "2026-04-12T23:31:49Z"
    model: gpt-5.4
    provider: openai
    source_hash: ec796241b884ca16ec7077df4f3f1910e2850487bb3ea94f8fdb37c77e02b219
    source_path: providers/ollama.md
    workflow: 15
---

# Ollama

Ollama est un runtime LLM local qui facilite l'exécution de modèles open source sur votre machine. OpenClaw s'intègre à l'API native d'Ollama (`/api/chat`), prend en charge le streaming et l'appel d'outils, et peut découvrir automatiquement les modèles Ollama locaux lorsque vous activez cette option avec `OLLAMA_API_KEY` (ou un profil d'authentification) et que vous ne définissez pas d'entrée explicite `models.providers.ollama`.

<Warning>
**Utilisateurs d'Ollama distant** : n'utilisez pas l'URL compatible OpenAI `/v1` (`http://host:11434/v1`) avec OpenClaw. Cela casse l'appel d'outils et les modèles peuvent produire du JSON d'outil brut en texte brut. Utilisez plutôt l'URL de l'API native Ollama : `baseUrl: "http://host:11434"` (sans `/v1`).
</Warning>

## Prise en main

Choisissez votre méthode de configuration et votre mode préférés.

<Tabs>
  <Tab title="Onboarding (recommandé)">
    **Idéal pour :** le chemin le plus rapide vers une configuration Ollama fonctionnelle avec découverte automatique des modèles.

    <Steps>
      <Step title="Lancer l'onboarding">
        ```bash
        openclaw onboard
        ```

        Sélectionnez **Ollama** dans la liste des fournisseurs.
      </Step>
      <Step title="Choisir votre mode">
        - **Cloud + Local** — modèles hébergés dans le cloud et modèles locaux ensemble
        - **Local** — modèles locaux uniquement

        Si vous choisissez **Cloud + Local** et que vous n'êtes pas connecté à ollama.com, l'onboarding ouvre un flux de connexion dans le navigateur.
      </Step>
      <Step title="Sélectionner un modèle">
        L'onboarding découvre les modèles disponibles et suggère des valeurs par défaut. Il télécharge automatiquement le modèle sélectionné s'il n'est pas disponible localement.
      </Step>
      <Step title="Vérifier que le modèle est disponible">
        ```bash
        openclaw models list --provider ollama
        ```
      </Step>
    </Steps>

    ### Mode non interactif

    ```bash
    openclaw onboard --non-interactive \
      --auth-choice ollama \
      --accept-risk
    ```

    Vous pouvez aussi spécifier une URL de base ou un modèle personnalisés :

    ```bash
    openclaw onboard --non-interactive \
      --auth-choice ollama \
      --custom-base-url "http://ollama-host:11434" \
      --custom-model-id "qwen3.5:27b" \
      --accept-risk
    ```

  </Tab>

  <Tab title="Configuration manuelle">
    **Idéal pour :** un contrôle total de l'installation, du téléchargement des modèles et de la configuration.

    <Steps>
      <Step title="Installer Ollama">
        Téléchargez-le depuis [ollama.com/download](https://ollama.com/download).
      </Step>
      <Step title="Télécharger un modèle local">
        ```bash
        ollama pull gemma4
        # ou
        ollama pull gpt-oss:20b
        # ou
        ollama pull llama3.3
        ```
      </Step>
      <Step title="Se connecter pour les modèles cloud (facultatif)">
        Si vous souhaitez aussi les modèles cloud :

        ```bash
        ollama signin
        ```
      </Step>
      <Step title="Activer Ollama pour OpenClaw">
        Définissez n'importe quelle valeur pour la clé API (Ollama n'exige pas de vraie clé) :

        ```bash
        # Définir la variable d'environnement
        export OLLAMA_API_KEY="ollama-local"

        # Ou configurer dans votre fichier de configuration
        openclaw config set models.providers.ollama.apiKey "ollama-local"
        ```
      </Step>
      <Step title="Inspecter et définir votre modèle">
        ```bash
        openclaw models list
        openclaw models set ollama/gemma4
        ```

        Ou définissez la valeur par défaut dans la configuration :

        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "ollama/gemma4" },
            },
          },
        }
        ```
      </Step>
    </Steps>

  </Tab>
</Tabs>

## Modèles cloud

<Tabs>
  <Tab title="Cloud + Local">
    Les modèles cloud vous permettent d'exécuter des modèles hébergés dans le cloud en parallèle de vos modèles locaux. Exemples : `kimi-k2.5:cloud`, `minimax-m2.7:cloud` et `glm-5.1:cloud` -- ceux-ci **n'exigent pas** de `ollama pull` local.

    Sélectionnez le mode **Cloud + Local** pendant la configuration. L'assistant vérifie si vous êtes connecté et ouvre un flux de connexion dans le navigateur si nécessaire. Si l'authentification ne peut pas être vérifiée, l'assistant revient aux modèles locaux par défaut.

    Vous pouvez aussi vous connecter directement sur [ollama.com/signin](https://ollama.com/signin).

    OpenClaw suggère actuellement ces valeurs cloud par défaut : `kimi-k2.5:cloud`, `minimax-m2.7:cloud`, `glm-5.1:cloud`.

  </Tab>

  <Tab title="Local uniquement">
    En mode local uniquement, OpenClaw découvre les modèles depuis l'instance Ollama locale. Aucune connexion cloud n'est nécessaire.

    OpenClaw suggère actuellement `gemma4` comme modèle local par défaut.

  </Tab>
</Tabs>

## Découverte de modèles (fournisseur implicite)

Lorsque vous définissez `OLLAMA_API_KEY` (ou un profil d'authentification) et que vous **ne** définissez **pas** `models.providers.ollama`, OpenClaw découvre les modèles depuis l'instance Ollama locale à l'adresse `http://127.0.0.1:11434`.

| Comportement        | Détail                                                                                                                                                              |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Requête de catalogue        | Interroge `/api/tags`                                                                                                                                                 |
| Détection des capacités | Utilise des consultations `/api/show` au mieux pour lire `contextWindow` et détecter les capacités (y compris la vision)                                                             |
| Modèles de vision        | Les modèles avec une capacité `vision` signalée par `/api/show` sont marqués comme compatibles image (`input: ["text", "image"]`), OpenClaw injecte donc automatiquement les images dans l'invite |
| Détection du raisonnement  | Marque `reasoning` avec une heuristique sur le nom du modèle (`r1`, `reasoning`, `think`)                                                                                          |
| Limites de jetons         | Définit `maxTokens` sur la limite maximale de jetons Ollama par défaut utilisée par OpenClaw                                                                                               |
| Coûts                | Définit tous les coûts sur `0`                                                                                                                                               |

Cela évite les entrées de modèle manuelles tout en gardant le catalogue aligné avec l'instance Ollama locale.

```bash
# Voir quels modèles sont disponibles
ollama list
openclaw models list
```

Pour ajouter un nouveau modèle, téléchargez-le simplement avec Ollama :

```bash
ollama pull mistral
```

Le nouveau modèle sera automatiquement découvert et disponible à l'utilisation.

<Note>
Si vous définissez explicitement `models.providers.ollama`, la découverte automatique est ignorée et vous devez définir les modèles manuellement. Consultez la section de configuration explicite ci-dessous.
</Note>

## Configuration

<Tabs>
  <Tab title="Basique (découverte implicite)">
    Le moyen le plus simple d'activer Ollama est via une variable d'environnement :

    ```bash
    export OLLAMA_API_KEY="ollama-local"
    ```

    <Tip>
    Si `OLLAMA_API_KEY` est défini, vous pouvez omettre `apiKey` dans l'entrée du fournisseur et OpenClaw le renseignera pour les vérifications de disponibilité.
    </Tip>

  </Tab>

  <Tab title="Explicite (modèles manuels)">
    Utilisez une configuration explicite lorsque Ollama s'exécute sur un autre hôte/port, que vous souhaitez forcer des fenêtres de contexte ou des listes de modèles spécifiques, ou que vous voulez des définitions de modèles entièrement manuelles.

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

  </Tab>

  <Tab title="URL de base personnalisée">
    Si Ollama s'exécute sur un autre hôte ou port (la configuration explicite désactive la découverte automatique, définissez donc les modèles manuellement) :

    ```json5
    {
      models: {
        providers: {
          ollama: {
            apiKey: "ollama-local",
            baseUrl: "http://ollama-host:11434", // Pas de /v1 - utilisez l'URL de l'API native Ollama
            api: "ollama", // Définissez-le explicitement pour garantir le comportement natif d'appel d'outils
          },
        },
      },
    }
    ```

    <Warning>
    N'ajoutez pas `/v1` à l'URL. Le chemin `/v1` utilise le mode compatible OpenAI, où l'appel d'outils n'est pas fiable. Utilisez l'URL de base Ollama sans suffixe de chemin.
    </Warning>

  </Tab>
</Tabs>

### Sélection de modèle

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

## Recherche Web Ollama

OpenClaw prend en charge **Ollama Web Search** comme fournisseur `web_search` intégré.

| Propriété    | Détail                                                                                                            |
| ----------- | ----------------------------------------------------------------------------------------------------------------- |
| Hôte        | Utilise votre hôte Ollama configuré (`models.providers.ollama.baseUrl` lorsqu'il est défini, sinon `http://127.0.0.1:11434`) |
| Authentification        | Sans clé                                                                                                          |
| Condition requise | Ollama doit être en cours d'exécution et connecté avec `ollama signin`                                                         |

Choisissez **Ollama Web Search** pendant `openclaw onboard` ou `openclaw configure --section web`, ou définissez :

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

<Note>
Pour la configuration complète et les détails de comportement, consultez [Ollama Web Search](/fr/tools/ollama-search).
</Note>

## Configuration avancée

<AccordionGroup>
  <Accordion title="Mode hérité compatible OpenAI">
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

    Lorsque `api: "openai-completions"` est utilisé avec Ollama, OpenClaw injecte `options.num_ctx` par défaut afin qu'Ollama ne revienne pas silencieusement à une fenêtre de contexte de 4096. Si votre proxy/upstream rejette les champs `options` inconnus, désactivez ce comportement :

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

  </Accordion>

  <Accordion title="Fenêtres de contexte">
    Pour les modèles découverts automatiquement, OpenClaw utilise la fenêtre de contexte signalée par Ollama lorsqu'elle est disponible ; sinon, il revient à la fenêtre de contexte Ollama par défaut utilisée par OpenClaw.

    Vous pouvez remplacer `contextWindow` et `maxTokens` dans la configuration explicite du fournisseur :

    ```json5
    {
      models: {
        providers: {
          ollama: {
            models: [
              {
                id: "llama3.3",
                contextWindow: 131072,
                maxTokens: 65536,
              }
            ]
          }
        }
      }
    }
    ```

  </Accordion>

  <Accordion title="Modèles de raisonnement">
    OpenClaw traite par défaut comme compatibles avec le raisonnement les modèles dont le nom contient `deepseek-r1`, `reasoning` ou `think`.

    ```bash
    ollama pull deepseek-r1:32b
    ```

    Aucune configuration supplémentaire n'est nécessaire -- OpenClaw les marque automatiquement.

  </Accordion>

  <Accordion title="Coûts des modèles">
    Ollama est gratuit et s'exécute en local, donc tous les coûts de modèle sont définis à $0. Cela s'applique aussi bien aux modèles découverts automatiquement qu'aux modèles définis manuellement.
  </Accordion>

  <Accordion title="Embeddings de mémoire">
    Le Plugin Ollama intégré enregistre un fournisseur d'embeddings de mémoire pour
    la [recherche mémoire](/fr/concepts/memory). Il utilise l'URL de base Ollama
    et la clé API configurées.

    | Propriété      | Valeur               |
    | ------------- | ------------------- |
    | Modèle par défaut | `nomic-embed-text`  |
    | Téléchargement automatique     | Oui — le modèle d'embedding est téléchargé automatiquement s'il n'est pas présent localement |

    Pour sélectionner Ollama comme fournisseur d'embeddings pour la recherche mémoire :

    ```json5
    {
      agents: {
        defaults: {
          memorySearch: { provider: "ollama" },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Configuration du streaming">
    L'intégration Ollama d'OpenClaw utilise par défaut l'**API native Ollama** (`/api/chat`), qui prend entièrement en charge simultanément le streaming et l'appel d'outils. Aucune configuration particulière n'est nécessaire.

    <Tip>
    Si vous devez utiliser le point de terminaison compatible OpenAI, consultez la section « Mode hérité compatible OpenAI » ci-dessus. Le streaming et l'appel d'outils peuvent ne pas fonctionner simultanément dans ce mode.
    </Tip>

  </Accordion>
</AccordionGroup>

## Dépannage

<AccordionGroup>
  <Accordion title="Ollama non détecté">
    Assurez-vous qu'Ollama est en cours d'exécution, que vous avez défini `OLLAMA_API_KEY` (ou un profil d'authentification), et que vous n'avez **pas** défini d'entrée explicite `models.providers.ollama` :

    ```bash
    ollama serve
    ```

    Vérifiez que l'API est accessible :

    ```bash
    curl http://localhost:11434/api/tags
    ```

  </Accordion>

  <Accordion title="Aucun modèle disponible">
    Si votre modèle n'est pas listé, téléchargez-le localement ou définissez-le explicitement dans `models.providers.ollama`.

    ```bash
    ollama list  # Voir ce qui est installé
    ollama pull gemma4
    ollama pull gpt-oss:20b
    ollama pull llama3.3     # Ou un autre modèle
    ```

  </Accordion>

  <Accordion title="Connexion refusée">
    Vérifiez qu'Ollama fonctionne sur le bon port :

    ```bash
    # Vérifier si Ollama est en cours d'exécution
    ps aux | grep ollama

    # Ou redémarrer Ollama
    ollama serve
    ```

  </Accordion>
</AccordionGroup>

<Note>
Aide supplémentaire : [Dépannage](/fr/help/troubleshooting) et [FAQ](/fr/help/faq).
</Note>

## Voir aussi

<CardGroup cols={2}>
  <Card title="Fournisseurs de modèles" href="/fr/concepts/model-providers" icon="layers">
    Vue d'ensemble de tous les fournisseurs, des références de modèles et du comportement de basculement.
  </Card>
  <Card title="Sélection de modèle" href="/fr/concepts/models" icon="brain">
    Comment choisir et configurer les modèles.
  </Card>
  <Card title="Ollama Web Search" href="/fr/tools/ollama-search" icon="magnifying-glass">
    Configuration complète et détails de comportement pour la recherche web alimentée par Ollama.
  </Card>
  <Card title="Configuration" href="/fr/gateway/configuration" icon="gear">
    Référence complète de configuration.
  </Card>
</CardGroup>
