---
x-i18n:
    generated_at: "2026-04-06T03:05:52Z"
    model: gpt-5.4
    provider: openai
    source_hash: 6e1cf417b0c04d001bc494fbe03ac2fcb66866f759e21646dbfd1a9c3a968bff
    source_path: .i18n/README.md
    workflow: 15
---

# Risorse di internazionalizzazione della documentazione di OpenClaw

Questa cartella archivia la configurazione di traduzione per il repository sorgente della documentazione.

Le pagine di localizzazione generate e la memoria di traduzione live delle localizzazioni ora si trovano nel repository di pubblicazione (`openclaw/docs`, checkout locale sibling `~/Projects/openclaw-docs`).

## File

- `glossary.<lang>.json` — mappature dei termini preferiti (usate nella guida del prompt).
- `<lang>.tm.jsonl` — memoria di traduzione (cache) indicizzata per workflow + modello + hash del testo. In questo repository, i file TM delle localizzazioni vengono generati su richiesta.

## Formato del glossario

`glossary.<lang>.json` è un array di voci:

```json
{
  "source": "troubleshooting",
  "target": "故障排除",
  "ignore_case": true,
  "whole_word": false
}
```

Campi:

- `source`: frase inglese (o di origine) da preferire.
- `target`: output di traduzione preferito.

## Note

- Le voci del glossario vengono passate al modello come **guida del prompt** (senza riscritture deterministiche).
- `scripts/docs-i18n` mantiene ancora la responsabilità della generazione delle traduzioni.
- Il repository sorgente sincronizza la documentazione in inglese nel repository di pubblicazione; la generazione delle localizzazioni viene eseguita lì per ogni localizzazione su push, pianificazione e invio della release.
