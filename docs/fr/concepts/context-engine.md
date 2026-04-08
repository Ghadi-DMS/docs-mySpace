---
read_when:
    - Vous voulez comprendre comment OpenClaw assemble le contexte du modèle
    - Vous passez du moteur hérité à un moteur de plugin, ou inversement
    - Vous créez un plugin de moteur de contexte
summary: 'Moteur de contexte : assemblage contextuel modulaire, compactage et cycle de vie des sous-agents'
title: Moteur de contexte
x-i18n:
    generated_at: "2026-04-08T02:14:26Z"
    model: gpt-5.4
    provider: openai
    source_hash: e8290ac73272eee275bce8e481ac7959b65386752caa68044d0c6f3e450acfb1
    source_path: concepts/context-engine.md
    workflow: 15
---

# Moteur de contexte

Un **moteur de contexte** contrôle la façon dont OpenClaw construit le contexte du modèle pour chaque exécution.
Il décide quels messages inclure, comment résumer l’historique plus ancien et
comment gérer le contexte au niveau des frontières entre sous-agents.

OpenClaw est livré avec un moteur intégré `legacy`. Les plugins peuvent enregistrer
des moteurs alternatifs qui remplacent le cycle de vie actif du moteur de contexte.

## Démarrage rapide

Vérifiez quel moteur est actif :

```bash
openclaw doctor
# or inspect config directly:
cat ~/.openclaw/openclaw.json | jq '.plugins.slots.contextEngine'
```

### Installation d’un plugin de moteur de contexte

Les plugins de moteur de contexte s’installent comme n’importe quel autre plugin OpenClaw. Installez-le
d’abord, puis sélectionnez le moteur dans l’emplacement :

```bash
# Install from npm
openclaw plugins install @martian-engineering/lossless-claw

# Or install from a local path (for development)
openclaw plugins install -l ./my-context-engine
```

Ensuite, activez le plugin et sélectionnez-le comme moteur actif dans votre configuration :

```json5
// openclaw.json
{
  plugins: {
    slots: {
      contextEngine: "lossless-claw", // must match the plugin's registered engine id
    },
    entries: {
      "lossless-claw": {
        enabled: true,
        // Plugin-specific config goes here (see the plugin's docs)
      },
    },
  },
}
```

Redémarrez la passerelle après l’installation et la configuration.

Pour revenir au moteur intégré, définissez `contextEngine` sur `"legacy"` (ou
supprimez entièrement la clé — `"legacy"` est la valeur par défaut).

## Fonctionnement

Chaque fois qu’OpenClaw exécute une invite de modèle, le moteur de contexte intervient à
quatre points du cycle de vie :

1. **Ingestion** — appelée lorsqu’un nouveau message est ajouté à la session. Le moteur
   peut stocker ou indexer le message dans son propre magasin de données.
2. **Assemblage** — appelée avant chaque exécution du modèle. Le moteur renvoie un
   ensemble ordonné de messages (et un `systemPromptAddition` facultatif) qui tiennent dans
   le budget de jetons.
3. **Compactage** — appelée lorsque la fenêtre de contexte est pleine, ou lorsque l’utilisateur exécute
   `/compact`. Le moteur résume l’historique plus ancien pour libérer de l’espace.
4. **Après tour** — appelée une fois l’exécution terminée. Le moteur peut persister l’état,
   déclencher un compactage en arrière-plan ou mettre à jour des index.

### Cycle de vie des sous-agents (facultatif)

OpenClaw appelle actuellement un hook de cycle de vie de sous-agent :

- **onSubagentEnded** — nettoie lorsque la session d’un sous-agent se termine ou est balayée.

Le hook `prepareSubagentSpawn` fait partie de l’interface pour une utilisation future, mais
l’environnement d’exécution ne l’invoque pas encore.

### Ajout à l’invite système

La méthode `assemble` peut renvoyer une chaîne `systemPromptAddition`. OpenClaw
la préfixe à l’invite système de l’exécution. Cela permet aux moteurs d’injecter
des indications dynamiques de rappel, des instructions de récupération ou des conseils
sensibles au contexte, sans exiger de fichiers statiques dans l’espace de travail.

## Le moteur hérité

Le moteur intégré `legacy` préserve le comportement d’origine d’OpenClaw :

- **Ingestion** : no-op (le gestionnaire de session gère directement la persistance des messages).
- **Assemblage** : transmission directe (le pipeline sanitize → validate → limit existant
  dans l’environnement d’exécution gère l’assemblage du contexte).
- **Compactage** : délègue au compactage de résumé intégré, qui crée
  un résumé unique des anciens messages et conserve intacts les messages récents.
- **Après tour** : no-op.

Le moteur hérité n’enregistre pas d’outils et ne fournit pas de `systemPromptAddition`.

Lorsqu’aucun `plugins.slots.contextEngine` n’est défini (ou qu’il est défini sur `"legacy"`), ce
moteur est utilisé automatiquement.

## Moteurs de plugin

Un plugin peut enregistrer un moteur de contexte à l’aide de l’API de plugin :

```ts
import { buildMemorySystemPromptAddition } from "openclaw/plugin-sdk/core";

export default function register(api) {
  api.registerContextEngine("my-engine", () => ({
    info: {
      id: "my-engine",
      name: "My Context Engine",
      ownsCompaction: true,
    },

    async ingest({ sessionId, message, isHeartbeat }) {
      // Store the message in your data store
      return { ingested: true };
    },

    async assemble({ sessionId, messages, tokenBudget, availableTools, citationsMode }) {
      // Return messages that fit the budget
      return {
        messages: buildContext(messages, tokenBudget),
        estimatedTokens: countTokens(messages),
        systemPromptAddition: buildMemorySystemPromptAddition({
          availableTools: availableTools ?? new Set(),
          citationsMode,
        }),
      };
    },

    async compact({ sessionId, force }) {
      // Summarize older context
      return { ok: true, compacted: true };
    },
  }));
}
```

Ensuite, activez-le dans la configuration :

```json5
{
  plugins: {
    slots: {
      contextEngine: "my-engine",
    },
    entries: {
      "my-engine": {
        enabled: true,
      },
    },
  },
}
```

### L’interface ContextEngine

Membres requis :

| Member             | Kind     | Purpose                                                  |
| ------------------ | -------- | -------------------------------------------------------- |
| `info`             | Propriété | ID du moteur, nom, version et indication de possession du compactage |
| `ingest(params)`   | Méthode   | Stocker un seul message                                  |
| `assemble(params)` | Méthode   | Construire le contexte pour une exécution du modèle (renvoie `AssembleResult`) |
| `compact(params)`  | Méthode   | Résumer/réduire le contexte                              |

`assemble` renvoie un `AssembleResult` avec :

- `messages` — les messages ordonnés à envoyer au modèle.
- `estimatedTokens` (requis, `number`) — l’estimation par le moteur du nombre total de
  jetons dans le contexte assemblé. OpenClaw l’utilise pour les décisions de seuil de compactage
  et les rapports de diagnostic.
- `systemPromptAddition` (facultatif, `string`) — préfixé à l’invite système.

Membres facultatifs :

| Member                         | Kind   | Purpose                                                                                                         |
| ------------------------------ | ------ | --------------------------------------------------------------------------------------------------------------- |
| `bootstrap(params)`            | Méthode | Initialiser l’état du moteur pour une session. Appelée une fois lorsque le moteur voit une session pour la première fois (par ex. importation de l’historique). |
| `ingestBatch(params)`          | Méthode | Ingérer un tour terminé sous forme de lot. Appelée après la fin d’une exécution, avec tous les messages de ce tour à la fois. |
| `afterTurn(params)`            | Méthode | Travail de cycle de vie après exécution (persister l’état, déclencher un compactage en arrière-plan). |
| `prepareSubagentSpawn(params)` | Méthode | Mettre en place un état partagé pour une session enfant.                                                                        |
| `onSubagentEnded(params)`      | Méthode | Nettoyer après la fin d’un sous-agent.                                                                                 |
| `dispose()`                    | Méthode | Libérer les ressources. Appelée lors de l’arrêt de la passerelle ou du rechargement du plugin — pas par session.                           |

### ownsCompaction

`ownsCompaction` contrôle si l’auto-compactage intégré de Pi pendant une tentative reste
activé pour l’exécution :

- `true` — le moteur prend en charge le comportement de compactage. OpenClaw désactive l’auto-compactage intégré de Pi
  pour cette exécution, et l’implémentation `compact()` du moteur est
  responsable de `/compact`, du compactage de récupération après dépassement, ainsi que de tout compactage
  proactif qu’il souhaite effectuer dans `afterTurn()`.
- `false` ou non défini — l’auto-compactage intégré de Pi peut toujours s’exécuter pendant l’exécution
  de l’invite, mais la méthode `compact()` du moteur actif est toujours appelée pour
  `/compact` et la récupération après dépassement.

`ownsCompaction: false` ne signifie **pas** qu’OpenClaw revient automatiquement
au chemin de compactage du moteur hérité.

Cela signifie qu’il existe deux modèles de plugin valides :

- **Mode propriétaire** — implémentez votre propre algorithme de compactage et définissez
  `ownsCompaction: true`.
- **Mode de délégation** — définissez `ownsCompaction: false` et faites en sorte que `compact()` appelle
  `delegateCompactionToRuntime(...)` depuis `openclaw/plugin-sdk/core` pour utiliser
  le comportement de compactage intégré d’OpenClaw.

Un `compact()` no-op n’est pas sûr pour un moteur actif non propriétaire, car il
désactive le chemin normal de compactage `/compact` et de récupération après dépassement pour cet
emplacement de moteur.

## Référence de configuration

```json5
{
  plugins: {
    slots: {
      // Select the active context engine. Default: "legacy".
      // Set to a plugin id to use a plugin engine.
      contextEngine: "legacy",
    },
  },
}
```

L’emplacement est exclusif au moment de l’exécution — un seul moteur de contexte enregistré est
résolu pour une exécution ou une opération de compactage donnée. D’autres
plugins `kind: "context-engine"` activés peuvent toujours se charger et exécuter leur code
d’enregistrement ; `plugins.slots.contextEngine` sélectionne seulement quel ID de moteur enregistré
OpenClaw résout lorsqu’il a besoin d’un moteur de contexte.

## Relation avec le compactage et la mémoire

- **Le compactage** est l’une des responsabilités du moteur de contexte. Le moteur hérité
  délègue au résumé intégré d’OpenClaw. Les moteurs de plugin peuvent implémenter
  n’importe quelle stratégie de compactage (résumés DAG, récupération vectorielle, etc.).
- **Les plugins de mémoire** (`plugins.slots.memory`) sont distincts des moteurs de contexte.
  Les plugins de mémoire fournissent la recherche/récupération ; les moteurs de contexte contrôlent ce que le
  modèle voit. Ils peuvent fonctionner ensemble — un moteur de contexte peut utiliser des données du plugin de mémoire
  pendant l’assemblage. Les moteurs de plugin qui veulent le chemin actif d’invite mémoire
  devraient privilégier `buildMemorySystemPromptAddition(...)` depuis
  `openclaw/plugin-sdk/core`, qui convertit les sections actives de l’invite mémoire
  en un `systemPromptAddition` prêt à être préfixé. Si un moteur a besoin d’un contrôle de plus bas niveau,
  il peut toujours récupérer les lignes brutes depuis
  `openclaw/plugin-sdk/memory-host-core` via
  `buildActiveMemoryPromptSection(...)`.
- **L’élagage de session** (suppression en mémoire des anciens résultats d’outils) continue de s’exécuter
  quel que soit le moteur de contexte actif.

## Conseils

- Utilisez `openclaw doctor` pour vérifier que votre moteur se charge correctement.
- Si vous changez de moteur, les sessions existantes conservent leur historique actuel.
  Le nouveau moteur prend le relais pour les exécutions futures.
- Les erreurs de moteur sont consignées dans les journaux et affichées dans les diagnostics. Si un moteur de plugin
  ne parvient pas à s’enregistrer ou si l’ID de moteur sélectionné ne peut pas être résolu, OpenClaw
  ne bascule pas automatiquement ; les exécutions échouent jusqu’à ce que vous corrigiez le plugin ou
  que vous redéfinissiez `plugins.slots.contextEngine` sur `"legacy"`.
- Pour le développement, utilisez `openclaw plugins install -l ./my-engine` pour lier un
  répertoire de plugin local sans le copier.

Voir aussi : [Compactage](/fr/concepts/compaction), [Contexte](/fr/concepts/context),
[Plugins](/fr/tools/plugin), [Manifeste de plugin](/fr/plugins/manifest).

## Lié

- [Contexte](/fr/concepts/context) — comment le contexte est construit pour les tours de l’agent
- [Architecture des plugins](/fr/plugins/architecture) — enregistrement de plugins de moteur de contexte
- [Compactage](/fr/concepts/compaction) — résumé des longues conversations
