---
read_when:
    - Vous modifiez le runtime d’agent embarqué ou le registre de harness
    - Vous enregistrez un harness d’agent à partir d’un plugin intégré ou de confiance
    - Vous devez comprendre la relation entre le plugin Codex et les fournisseurs de modèles
sidebarTitle: Agent Harness
summary: Surface SDK expérimentale pour les plugins qui remplacent l’exécuteur d’agent embarqué de bas niveau
title: Plugins Agent Harness
x-i18n:
    generated_at: "2026-04-11T02:46:14Z"
    model: gpt-5.4
    provider: openai
    source_hash: 43c1f2c087230398b0162ed98449f239c8db1e822e51c7dcd40c54fa6c3374e1
    source_path: plugins/sdk-agent-harness.md
    workflow: 15
---

# Plugins Agent Harness

Un **agent harness** est l’exécuteur de bas niveau pour un tour préparé d’agent OpenClaw.
Ce n’est ni un fournisseur de modèles, ni un canal, ni un registre d’outils.

Utilisez cette surface uniquement pour des plugins natifs intégrés ou de confiance. Le contrat
reste expérimental, car les types de paramètres reflètent volontairement l’exécuteur embarqué actuel.

## Quand utiliser un harness

Enregistrez un agent harness lorsqu’une famille de modèles possède son propre runtime de session natif
et que le transport normal de fournisseur OpenClaw est une mauvaise abstraction.

Exemples :

- un serveur natif d’agent de programmation qui gère lui-même les fils et la compaction
- une CLI ou un daemon local qui doit diffuser en flux des événements natifs de plan/raisonnement/outils
- un runtime de modèle qui nécessite son propre identifiant de reprise en plus de la
  transcription de session OpenClaw

N’enregistrez **pas** un harness simplement pour ajouter une nouvelle API LLM. Pour des API de modèles HTTP ou
WebSocket normales, créez un [plugin fournisseur](/fr/plugins/sdk-provider-plugins).

## Ce que le core continue de gérer

Avant qu’un harness soit sélectionné, OpenClaw a déjà résolu :

- le fournisseur et le modèle
- l’état d’authentification du runtime
- le niveau de réflexion et le budget de contexte
- le fichier de transcription/session OpenClaw
- l’espace de travail, le bac à sable et la politique d’outils
- les callbacks de réponse du canal et les callbacks de streaming
- la politique de fallback de modèle et de changement de modèle actif

Cette séparation est intentionnelle. Un harness exécute une tentative préparée ; il ne choisit
pas les fournisseurs, ne remplace pas la livraison des canaux et ne change pas silencieusement de modèle.

## Enregistrer un harness

**Importation :** `openclaw/plugin-sdk/agent-harness`

```typescript
import type { AgentHarness } from "openclaw/plugin-sdk/agent-harness";
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

const myHarness: AgentHarness = {
  id: "my-harness",
  label: "My native agent harness",

  supports(ctx) {
    return ctx.provider === "my-provider"
      ? { supported: true, priority: 100 }
      : { supported: false };
  },

  async runAttempt(params) {
    // Start or resume your native thread.
    // Use params.prompt, params.tools, params.images, params.onPartialReply,
    // params.onAgentEvent, and the other prepared attempt fields.
    return await runMyNativeTurn(params);
  },
};

export default definePluginEntry({
  id: "my-native-agent",
  name: "My Native Agent",
  description: "Runs selected models through a native agent daemon.",
  register(api) {
    api.registerAgentHarness(myHarness);
  },
});
```

## Politique de sélection

OpenClaw choisit un harness après la résolution du fournisseur/modèle :

1. `OPENCLAW_AGENT_RUNTIME=<id>` force un harness enregistré avec cet id.
2. `OPENCLAW_AGENT_RUNTIME=pi` force le harness PI intégré.
3. `OPENCLAW_AGENT_RUNTIME=auto` demande aux harness enregistrés s’ils prennent en charge le
   fournisseur/modèle résolu.
4. Si aucun harness enregistré ne correspond, OpenClaw utilise PI, sauf si le fallback PI est
   désactivé.

Les échecs d’un harness de plugin forcé apparaissent comme des échecs d’exécution. En mode `auto`,
OpenClaw peut revenir à PI lorsque le harness de plugin sélectionné échoue avant qu’un
tour n’ait produit des effets de bord. Définissez `OPENCLAW_AGENT_HARNESS_FALLBACK=none` ou
`embeddedHarness.fallback: "none"` pour faire de ce fallback un échec bloquant à la place.

Le plugin Codex intégré enregistre `codex` comme identifiant de harness. Le core traite cela
comme un identifiant ordinaire de harness de plugin ; les alias spécifiques à Codex doivent appartenir au plugin
ou à la configuration de l’opérateur, pas au sélecteur de runtime partagé.

## Appairage fournisseur plus harness

La plupart des harness devraient également enregistrer un fournisseur. Le fournisseur rend visibles les références de modèle,
l’état d’authentification, les métadonnées du modèle et la sélection `/model` au reste de
OpenClaw. Le harness revendique ensuite ce fournisseur dans `supports(...)`.

Le plugin Codex intégré suit ce modèle :

- id du fournisseur : `codex`
- références de modèle utilisateur : `codex/gpt-5.4`, `codex/gpt-5.2`, ou un autre modèle renvoyé
  par le serveur d’application Codex
- id du harness : `codex`
- auth : disponibilité synthétique du fournisseur, parce que le harness Codex gère lui-même la
  connexion/session Codex native
- requête du serveur d’application : OpenClaw envoie l’identifiant nu du modèle à Codex et laisse le
  harness dialoguer avec le protocole natif du serveur d’application

Le plugin Codex est additif. Les références simples `openai/gpt-*` restent des références du fournisseur OpenAI
et continuent d’utiliser le chemin normal du fournisseur OpenClaw. Sélectionnez `codex/gpt-*`
lorsque vous voulez une auth gérée par Codex, la découverte des modèles Codex, des fils natifs, et une exécution via le
serveur d’application Codex. `/model` peut basculer entre les modèles Codex renvoyés
par le serveur d’application Codex sans nécessiter d’identifiants du fournisseur OpenAI.

Pour la configuration opérateur, les exemples de préfixe de modèle et les configs réservées à Codex, voir
[Codex Harness](/fr/plugins/codex-harness).

OpenClaw exige la version `0.118.0` ou plus récente du serveur d’application Codex. Le plugin Codex vérifie
la poignée de main d’initialisation du serveur d’application et bloque les serveurs plus anciens ou sans version afin que
OpenClaw ne s’exécute que sur la surface de protocole avec laquelle il a été testé.

## Désactiver le fallback PI

Par défaut, OpenClaw exécute les agents embarqués avec `agents.defaults.embeddedHarness`
défini sur `{ runtime: "auto", fallback: "pi" }`. En mode `auto`, les harness de plugin enregistrés
peuvent revendiquer une paire fournisseur/modèle. Si aucun ne correspond, ou si un harness de plugin sélectionné
automatiquement échoue avant de produire une sortie, OpenClaw revient à PI.

Définissez `fallback: "none"` lorsque vous devez prouver qu’un harness de plugin est le seul
runtime exercé. Cela désactive le fallback automatique vers PI ; cela ne bloque pas
un `runtime: "pi"` explicite ou `OPENCLAW_AGENT_RUNTIME=pi`.

Pour des exécutions embarquées réservées à Codex :

```json
{
  "agents": {
    "defaults": {
      "model": "codex/gpt-5.4",
      "embeddedHarness": {
        "runtime": "codex",
        "fallback": "none"
      }
    }
  }
}
```

Si vous voulez que n’importe quel harness de plugin enregistré puisse revendiquer les modèles correspondants, mais ne
voulez jamais qu’OpenClaw revienne silencieusement à PI, conservez `runtime: "auto"` et désactivez
le fallback :

```json
{
  "agents": {
    "defaults": {
      "embeddedHarness": {
        "runtime": "auto",
        "fallback": "none"
      }
    }
  }
}
```

Les remplacements par agent utilisent la même forme :

```json
{
  "agents": {
    "defaults": {
      "embeddedHarness": {
        "runtime": "auto",
        "fallback": "pi"
      }
    },
    "list": [
      {
        "id": "codex-only",
        "model": "codex/gpt-5.4",
        "embeddedHarness": {
          "runtime": "codex",
          "fallback": "none"
        }
      }
    ]
  }
}
```

`OPENCLAW_AGENT_RUNTIME` remplace toujours le runtime configuré. Utilisez
`OPENCLAW_AGENT_HARNESS_FALLBACK=none` pour désactiver le fallback PI depuis
l’environnement.

```bash
OPENCLAW_AGENT_RUNTIME=codex \
OPENCLAW_AGENT_HARNESS_FALLBACK=none \
openclaw gateway run
```

Avec le fallback désactivé, une session échoue tôt lorsque le harness demandé n’est pas
enregistré, ne prend pas en charge le fournisseur/modèle résolu, ou échoue avant d’avoir
produit des effets de bord du tour. C’est intentionnel pour les déploiements réservés à Codex et
pour les tests actifs qui doivent prouver que le chemin du serveur d’application Codex est réellement utilisé.

Ce paramètre ne contrôle que le harness d’agent embarqué. Il ne désactive pas le routage de modèles spécifique au fournisseur pour
les images, vidéos, musiques, TTS, PDF ou autres.

## Sessions natives et miroir de transcription

Un harness peut conserver un identifiant de session native, un identifiant de fil, ou un jeton de reprise côté daemon.
Conservez cette liaison explicitement associée à la session OpenClaw, et continuez à
recopier la sortie assistant/outils visible par l’utilisateur dans la transcription OpenClaw.

La transcription OpenClaw reste la couche de compatibilité pour :

- l’historique de session visible par le canal
- la recherche et l’indexation dans la transcription
- le retour au harness PI intégré lors d’un tour ultérieur
- le comportement générique de `/new`, `/reset` et de suppression de session

Si votre harness stocke une liaison auxiliaire, implémentez `reset(...)` afin qu’OpenClaw puisse
l’effacer lorsque la session OpenClaw propriétaire est réinitialisée.

## Résultats d’outils et de médias

Le core construit la liste des outils OpenClaw et la transmet à la tentative préparée.
Lorsqu’un harness exécute un appel d’outil dynamique, renvoyez le résultat de l’outil via
la forme de résultat du harness au lieu d’envoyer vous-même les médias du canal.

Cela maintient les sorties de texte, image, vidéo, musique, TTS, approbation et outils de messagerie
sur le même chemin de livraison que les exécutions adossées à PI.

## Limitations actuelles

- Le chemin d’importation public est générique, mais certains alias de types de tentative/résultat portent encore
  des noms `Pi` pour compatibilité.
- L’installation de harness tiers est expérimentale. Préférez les plugins fournisseurs
  jusqu’à ce que vous ayez besoin d’un runtime de session natif.
- Le changement de harness est pris en charge entre les tours. Ne changez pas de harness au
  milieu d’un tour après le début des outils natifs, des approbations, du texte d’assistant ou des
  envois de messages.

## Voir aussi

- [Aperçu du SDK](/fr/plugins/sdk-overview)
- [Assistants runtime](/fr/plugins/sdk-runtime)
- [Plugins fournisseurs](/fr/plugins/sdk-provider-plugins)
- [Codex Harness](/fr/plugins/codex-harness)
- [Fournisseurs de modèles](/fr/concepts/model-providers)
