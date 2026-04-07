---
read_when:
    - Vous cherchez une vue d’ensemble des capacités multimédias
    - Vous choisissez quel fournisseur multimédia configurer
    - Vous voulez comprendre le fonctionnement de la génération asynchrone de médias
summary: Page d’accueil unifiée pour les capacités de génération de médias, de compréhension et de parole
title: Vue d’ensemble des médias
x-i18n:
    generated_at: "2026-04-07T06:54:56Z"
    model: gpt-5.4
    provider: openai
    source_hash: cfee08eb91ec3e827724c8fa99bff7465356f6f1ac1b146562f35651798e3fd6
    source_path: tools/media-overview.md
    workflow: 15
---

# Génération et compréhension des médias

OpenClaw génère des images, des vidéos et de la musique, comprend les médias entrants (images, audio, vidéo) et lit les réponses à voix haute avec la synthèse vocale. Toutes les capacités multimédias sont pilotées par des outils : l’agent décide quand les utiliser en fonction de la conversation, et chaque outil n’apparaît que lorsqu’au moins un fournisseur sous-jacent est configuré.

## Capacités en un coup d’œil

| Capacité                  | Outil            | Fournisseurs                                                                                | Ce qu’il fait                                            |
| ------------------------- | ---------------- | -------------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| Génération d’images       | `image_generate` | ComfyUI, fal, Google, MiniMax, OpenAI, Vydra                                                 | Crée ou modifie des images à partir de prompts textuels ou de références |
| Génération vidéo          | `video_generate` | Alibaba, BytePlus, ComfyUI, fal, Google, MiniMax, OpenAI, Qwen, Runway, Together, Vydra, xAI | Crée des vidéos à partir de texte, d’images ou de vidéos existantes |
| Génération musicale       | `music_generate` | ComfyUI, Google, MiniMax                                                                     | Crée de la musique ou des pistes audio à partir de prompts textuels |
| Synthèse vocale (TTS)     | `tts`            | ElevenLabs, Microsoft, MiniMax, OpenAI                                                       | Convertit les réponses sortantes en audio parlé          |
| Compréhension des médias  | (automatique)    | Tout fournisseur de modèles compatible vision/audio, ainsi que des solutions de repli CLI    | Résume les images, l’audio et la vidéo entrants          |

## Matrice des capacités des fournisseurs

Ce tableau montre quels fournisseurs prennent en charge quelles capacités multimédias sur l’ensemble de la plateforme.

| Fournisseur | Image | Vidéo | Musique | TTS | STT / Transcription | Compréhension des médias |
| ----------- | ----- | ----- | ------- | --- | ------------------- | ------------------------ |
| Alibaba     |       | Oui   |         |     |                     |                          |
| BytePlus    |       | Oui   |         |     |                     |                          |
| ComfyUI     | Oui   | Oui   | Oui     |     |                     |                          |
| Deepgram    |       |       |         |     | Oui                 |                          |
| ElevenLabs  |       |       |         | Oui |                     |                          |
| fal         | Oui   | Oui   |         |     |                     |                          |
| Google      | Oui   | Oui   | Oui     |     |                     | Oui                      |
| Microsoft   |       |       |         | Oui |                     |                          |
| MiniMax     | Oui   | Oui   | Oui     | Oui |                     |                          |
| OpenAI      | Oui   | Oui   |         | Oui | Oui                 | Oui                      |
| Qwen        |       | Oui   |         |     |                     |                          |
| Runway      |       | Oui   |         |     |                     |                          |
| Together    |       | Oui   |         |     |                     |                          |
| Vydra       | Oui   | Oui   |         |     |                     |                          |
| xAI         |       | Oui   |         |     |                     |                          |

<Note>
La compréhension des médias utilise tout modèle compatible vision ou audio enregistré dans votre configuration de fournisseur. Le tableau ci-dessus met en évidence les fournisseurs disposant d’une prise en charge dédiée de la compréhension des médias ; la plupart des fournisseurs LLM avec des modèles multimodaux (Anthropic, Google, OpenAI, etc.) peuvent aussi comprendre les médias entrants lorsqu’ils sont configurés comme modèle de réponse actif.
</Note>

## Fonctionnement de la génération asynchrone

La génération vidéo et la génération musicale s’exécutent comme des tâches en arrière-plan, car le traitement côté fournisseur prend généralement de 30 secondes à plusieurs minutes. Lorsque l’agent appelle `video_generate` ou `music_generate`, OpenClaw envoie la requête au fournisseur, renvoie immédiatement un id de tâche et suit le travail dans le registre des tâches. L’agent continue de répondre à d’autres messages pendant l’exécution de la tâche. Lorsque le fournisseur a terminé, OpenClaw réactive l’agent afin qu’il puisse publier le média finalisé dans le canal d’origine. La génération d’images et le TTS sont synchrones et se terminent directement dans la réponse.

## Liens rapides

- [Génération d’images](/fr/tools/image-generation) -- génération et modification d’images
- [Génération vidéo](/fr/tools/video-generation) -- texte-vers-vidéo, image-vers-vidéo et vidéo-vers-vidéo
- [Génération musicale](/fr/tools/music-generation) -- création de musique et de pistes audio
- [Synthèse vocale](/fr/tools/tts) -- conversion des réponses en audio parlé
- [Compréhension des médias](/fr/nodes/media-understanding) -- compréhension des images, de l’audio et de la vidéo entrants
