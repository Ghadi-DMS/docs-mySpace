---
read_when:
    - Expliquer comment le streaming ou le découpage en morceaux fonctionne sur les canaux
    - Modifier le streaming par blocs ou le comportement de découpage en morceaux des canaux
    - Déboguer les réponses par blocs dupliquées/précoces ou le streaming d’aperçu du canal
summary: Comportement du streaming + du découpage en morceaux (réponses par blocs, streaming d’aperçu du canal, mappage des modes)
title: Streaming et découpage en morceaux
x-i18n:
    generated_at: "2026-04-08T06:01:06Z"
    model: gpt-5.4
    provider: openai
    source_hash: a8e847bb7da890818cd79dec7777f6ae488e6d6c0468e948e56b6b6c598e0000
    source_path: concepts/streaming.md
    workflow: 15
---

# Streaming + découpage en morceaux

OpenClaw possède deux couches de streaming distinctes :

- **Streaming par blocs (canaux) :** émet des **blocs** terminés pendant que l’assistant écrit. Ce sont des messages de canal normaux (pas des deltas de jetons).
- **Streaming d’aperçu (Telegram/Discord/Slack) :** met à jour un **message d’aperçu** temporaire pendant la génération.

Il n’existe aujourd’hui **aucun véritable streaming token-delta** vers les messages de canal. Le streaming d’aperçu est basé sur les messages (envoi + modifications/ajouts).

## Streaming par blocs (messages de canal)

Le streaming par blocs envoie la sortie de l’assistant en gros morceaux à mesure qu’elle devient disponible.

```
Model output
  └─ text_delta/events
       ├─ (blockStreamingBreak=text_end)
       │    └─ chunker emits blocks as buffer grows
       └─ (blockStreamingBreak=message_end)
            └─ chunker flushes at message_end
                   └─ channel send (block replies)
```

Légende :

- `text_delta/events` : événements de flux du modèle (peuvent être rares pour les modèles sans streaming).
- `chunker` : `EmbeddedBlockChunker` appliquant des limites min/max + la préférence de coupure.
- `channel send` : messages sortants réels (réponses par blocs).

**Contrôles :**

- `agents.defaults.blockStreamingDefault` : `"on"`/`"off"` (désactivé par défaut).
- Surcharges de canal : `*.blockStreaming` (et variantes par compte) pour forcer `"on"`/`"off"` par canal.
- `agents.defaults.blockStreamingBreak` : `"text_end"` ou `"message_end"`.
- `agents.defaults.blockStreamingChunk` : `{ minChars, maxChars, breakPreference? }`.
- `agents.defaults.blockStreamingCoalesce` : `{ minChars?, maxChars?, idleMs? }` (fusionne les blocs streamés avant l’envoi).
- Limite stricte du canal : `*.textChunkLimit` (par ex. `channels.whatsapp.textChunkLimit`).
- Mode de découpage du canal : `*.chunkMode` (`length` par défaut, `newline` coupe sur les lignes vides (limites de paragraphe) avant le découpage par longueur).
- Limite souple Discord : `channels.discord.maxLinesPerMessage` (17 par défaut) découpe les réponses longues pour éviter le rognage dans l’interface.

**Sémantique des limites :**

- `text_end` : diffuse les blocs dès que le chunker en émet ; vide le tampon à chaque `text_end`.
- `message_end` : attend que le message de l’assistant soit terminé, puis vide la sortie mise en tampon.

`message_end` utilise quand même le chunker si le texte mis en tampon dépasse `maxChars`, donc il peut émettre plusieurs morceaux à la fin.

## Algorithme de découpage en morceaux (limites basse/haute)

Le découpage par blocs est implémenté par `EmbeddedBlockChunker` :

- **Limite basse :** n’émet rien tant que le tampon n’a pas atteint `minChars` (sauf si forcé).
- **Limite haute :** préfère des coupures avant `maxChars` ; si forcé, coupe à `maxChars`.
- **Préférence de coupure :** `paragraph` → `newline` → `sentence` → `whitespace` → coupure forcée.
- **Blocs de code délimités :** ne jamais couper à l’intérieur ; en cas de coupure forcée à `maxChars`, ferme puis rouvre le bloc pour conserver un Markdown valide.

`maxChars` est plafonné à la valeur `textChunkLimit` du canal, donc vous ne pouvez pas dépasser les limites propres à chaque canal.

## Coalescence (fusion des blocs streamés)

Quand le streaming par blocs est activé, OpenClaw peut **fusionner des morceaux de blocs consécutifs**
avant de les envoyer. Cela réduit le « spam de lignes uniques » tout en fournissant
une sortie progressive.

- La coalescence attend des **périodes d’inactivité** (`idleMs`) avant de vider le tampon.
- Les tampons sont plafonnés par `maxChars` et seront vidés s’ils le dépassent.
- `minChars` empêche l’envoi de fragments minuscules tant qu’assez de texte ne s’est pas accumulé
  (le vidage final envoie toujours le texte restant).
- Le séparateur est dérivé de `blockStreamingChunk.breakPreference`
  (`paragraph` → `\n\n`, `newline` → `\n`, `sentence` → espace).
- Des surcharges par canal sont disponibles via `*.blockStreamingCoalesce` (y compris dans les configs par compte).
- La valeur par défaut de `minChars` pour la coalescence est portée à 1500 pour Signal/Slack/Discord sauf surcharge.

## Rythme plus humain entre les blocs

Quand le streaming par blocs est activé, vous pouvez ajouter une **pause aléatoire**
entre les réponses par blocs (après le premier bloc). Cela donne aux réponses en
plusieurs bulles une impression plus naturelle.

- Config : `agents.defaults.humanDelay` (surcharge par agent via `agents.list[].humanDelay`).
- Modes : `off` (par défaut), `natural` (800–2500ms), `custom` (`minMs`/`maxMs`).
- S’applique uniquement aux **réponses par blocs**, pas aux réponses finales ni aux résumés d’outils.

## « Stream les morceaux ou tout »

Cela correspond à :

- **Stream les morceaux :** `blockStreamingDefault: "on"` + `blockStreamingBreak: "text_end"` (émet au fil de l’eau). Les canaux autres que Telegram ont aussi besoin de `*.blockStreaming: true`.
- **Tout streamer à la fin :** `blockStreamingBreak: "message_end"` (vide une fois, éventuellement en plusieurs morceaux si c’est très long).
- **Aucun streaming par blocs :** `blockStreamingDefault: "off"` (réponse finale uniquement).

**Remarque sur les canaux :** Le streaming par blocs est **désactivé sauf si**
`*.blockStreaming` est explicitement défini sur `true`. Les canaux peuvent diffuser un aperçu en direct
(`channels.<channel>.streaming`) sans réponses par blocs.

Rappel sur l’emplacement de la config : les valeurs par défaut `blockStreaming*` se trouvent sous
`agents.defaults`, pas à la racine de la config.

## Modes de streaming d’aperçu

Clé canonique : `channels.<channel>.streaming`

Modes :

- `off` : désactive le streaming d’aperçu.
- `partial` : aperçu unique remplacé par le texte le plus récent.
- `block` : mises à jour d’aperçu par étapes découpées/ajoutées.
- `progress` : aperçu d’avancement/statut pendant la génération, réponse finale à la fin.

### Mappage par canal

| Canal    | `off` | `partial` | `block` | `progress`        |
| -------- | ----- | --------- | ------- | ----------------- |
| Telegram | ✅    | ✅        | ✅      | correspond à `partial` |
| Discord  | ✅    | ✅        | ✅      | correspond à `partial` |
| Slack    | ✅    | ✅        | ✅      | ✅                |

Spécifique à Slack :

- `channels.slack.streaming.nativeTransport` active/désactive les appels à l’API de streaming native de Slack lorsque `channels.slack.streaming.mode="partial"` (par défaut : `true`).
- Le streaming natif Slack et l’état des fils d’assistant Slack nécessitent une cible de fil de réponse ; les DM de niveau supérieur n’affichent pas cet aperçu de type fil.

Migration des clés héritées :

- Telegram : `streamMode` + booléen `streaming` sont automatiquement migrés vers l’énumération `streaming`.
- Discord : `streamMode` + booléen `streaming` sont automatiquement migrés vers l’énumération `streaming`.
- Slack : `streamMode` migre automatiquement vers `streaming.mode` ; le booléen `streaming` migre automatiquement vers `streaming.mode` plus `streaming.nativeTransport` ; l’ancien `nativeStreaming` migre automatiquement vers `streaming.nativeTransport`.

### Comportement à l’exécution

Telegram :

- Utilise les mises à jour d’aperçu `sendMessage` + `editMessageText` dans les DM et les groupes/sujets.
- Le streaming d’aperçu est ignoré lorsque le streaming par blocs Telegram est explicitement activé (pour éviter un double streaming).
- `/reasoning stream` peut écrire le raisonnement dans l’aperçu.

Discord :

- Utilise l’envoi + la modification de messages d’aperçu.
- Le mode `block` utilise le découpage de brouillon (`draftChunk`).
- Le streaming d’aperçu est ignoré lorsque le streaming par blocs Discord est explicitement activé.

Slack :

- `partial` peut utiliser le streaming natif de Slack (`chat.startStream`/`append`/`stop`) lorsqu’il est disponible.
- `block` utilise des aperçus de brouillon de style ajout.
- `progress` utilise un texte d’aperçu de statut, puis la réponse finale.

## Lié

- [Messages](/fr/concepts/messages) — cycle de vie et distribution des messages
- [Retry](/fr/concepts/retry) — comportement de nouvelle tentative en cas d’échec de distribution
- [Channels](/fr/channels) — prise en charge du streaming par canal
