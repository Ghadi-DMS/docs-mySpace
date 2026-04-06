---
read_when:
    - Estensione di qa-lab o qa-channel
    - Aggiunta di scenari QA supportati dal repository
    - Creazione di un'automazione QA più realistica attorno alla dashboard Gateway
summary: Struttura dell'automazione QA privata per qa-lab, qa-channel, scenari inizializzati e report di protocollo
title: Automazione QA E2E
x-i18n:
    generated_at: "2026-04-06T03:06:45Z"
    model: gpt-5.4
    provider: openai
    source_hash: df35f353d5ab0e0432e6a828c82772f9a88edb41c20ec5037315b7ba310b28e6
    source_path: concepts/qa-e2e-automation.md
    workflow: 15
---

# Automazione QA E2E

Lo stack QA privato è pensato per testare OpenClaw in modo più realistico,
con una forma simile a quella dei canali, rispetto a quanto possa fare un
singolo unit test.

Componenti attuali:

- `extensions/qa-channel`: canale di messaggi sintetico con superfici per DM, canale, thread,
  reazione, modifica ed eliminazione.
- `extensions/qa-lab`: UI di debug e bus QA per osservare la trascrizione,
  iniettare messaggi in ingresso ed esportare un report Markdown.
- `qa/`: asset seed supportati dal repository per l'attività di kickoff e gli
  scenari QA di base.

L'obiettivo a lungo termine è un sito QA a due pannelli:

- Sinistra: dashboard Gateway (Control UI) con l'agent.
- Destra: QA Lab, che mostra la trascrizione in stile Slack e il piano dello scenario.

Questo consente a un operatore o a un ciclo di automazione di assegnare
all'agent una missione QA, osservare il comportamento reale del canale e
registrare cosa ha funzionato, cosa ha fallito o cosa è rimasto bloccato.

## Seed supportati dal repository

Gli asset seed si trovano in `qa/`:

- `qa/QA_KICKOFF_TASK.md`
- `qa/seed-scenarios.json`

Questi file sono intenzionalmente in git così il piano QA è visibile sia agli esseri umani sia
all'agent. L'elenco di base dovrebbe restare sufficientemente ampio da coprire:

- chat DM e di canale
- comportamento dei thread
- ciclo di vita delle azioni sui messaggi
- callback cron
- richiamo della memoria
- cambio di modello
- handoff ai subagent
- lettura del repository e della documentazione
- una piccola attività di build come Lobster Invaders

## Report

`qa-lab` esporta un report di protocollo Markdown dalla timeline osservata del bus.
Il report dovrebbe rispondere a queste domande:

- Cosa ha funzionato
- Cosa ha fallito
- Cosa è rimasto bloccato
- Quali scenari di follow-up vale la pena aggiungere

## Documentazione correlata

- [Testing](/it/help/testing)
- [QA Channel](/channels/qa-channel)
- [Dashboard](/web/dashboard)
