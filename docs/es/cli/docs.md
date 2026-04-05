---
read_when:
    - Quieres buscar en la documentación activa de OpenClaw desde la terminal
summary: Referencia de CLI para `openclaw docs` (buscar en el índice activo de documentación)
title: docs
x-i18n:
    generated_at: "2026-04-05T12:37:48Z"
    model: gpt-5.4
    provider: openai
    source_hash: cfcceed872d7509b9843af3fae733a136bc5e26ded55c2ac47a16489a1636989
    source_path: cli/docs.md
    workflow: 15
---

# `openclaw docs`

Busca en el índice activo de documentación.

Argumentos:

- `[query...]`: términos de búsqueda para enviar al índice activo de documentación

Ejemplos:

```bash
openclaw docs
openclaw docs browser existing-session
openclaw docs sandbox allowHostControl
openclaw docs gateway token secretref
```

Notas:

- Sin consulta, `openclaw docs` abre el punto de entrada de búsqueda de la documentación activa.
- Las consultas de varias palabras se pasan como una sola solicitud de búsqueda.
