---
read_when:
    - Ejecutar scripts del repositorio
    - Añadir o cambiar scripts en ./scripts
summary: 'Scripts del repositorio: propósito, alcance y notas de seguridad'
title: Scripts
x-i18n:
    generated_at: "2026-04-05T12:44:02Z"
    model: gpt-5.4
    provider: openai
    source_hash: de53d64d91c564931bdd4e8b9f4a8e88646332a07cc2a6bf1d517b89debb29cd
    source_path: help/scripts.md
    workflow: 15
---

# Scripts

El directorio `scripts/` contiene scripts auxiliares para flujos de trabajo locales y tareas operativas.
Úsalos cuando una tarea esté claramente vinculada a un script; en caso contrario, prefiere la CLI.

## Convenciones

- Los scripts son **opcionales** salvo que se mencionen en la documentación o en listas de verificación de versiones.
- Prefiere las superficies de CLI cuando existan (ejemplo: la supervisión de autenticación usa `openclaw models status --check`).
- Asume que los scripts son específicos del host; léelos antes de ejecutarlos en una máquina nueva.

## Scripts de supervisión de autenticación

La supervisión de autenticación se cubre en [Authentication](/gateway/authentication). Los scripts de `scripts/` son extras opcionales para flujos de systemd/Termux en teléfonos.

## Al añadir scripts

- Mantén los scripts enfocados y documentados.
- Añade una entrada breve en la documentación relevante (o crea una si falta).
