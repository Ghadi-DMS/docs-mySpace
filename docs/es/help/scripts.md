---
read_when:
    - Ejecutar scripts desde el repositorio
    - Agregar o cambiar scripts en ./scripts
summary: 'Scripts del repositorio: propósito, alcance y notas de seguridad'
title: Scripts
x-i18n:
    generated_at: "2026-04-08T02:15:32Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3ecf1e9327929948fb75f80e306963af49b353c0aa8d3b6fa532ca964ff8b975
    source_path: help/scripts.md
    workflow: 15
---

# Scripts

El directorio `scripts/` contiene scripts auxiliares para flujos de trabajo locales y tareas operativas.
Úsalos cuando una tarea esté claramente vinculada a un script; en caso contrario, prefiere la CLI.

## Convenciones

- Los scripts son **opcionales** a menos que se mencionen en la documentación o en listas de verificación de lanzamiento.
- Prefiere las superficies de la CLI cuando existan (ejemplo: la supervisión de autenticación usa `openclaw models status --check`).
- Asume que los scripts son específicos del host; léelos antes de ejecutarlos en una máquina nueva.

## Scripts de supervisión de autenticación

La supervisión de autenticación se cubre en [Autenticación](/es/gateway/authentication). Los scripts en `scripts/` son extras opcionales para flujos de trabajo con systemd/Termux en teléfonos.

## Helper de lectura de GitHub

Usa `scripts/gh-read` cuando quieras que `gh` use un token de instalación de GitHub App para llamadas de lectura con alcance de repositorio, mientras dejas el `gh` normal con tu inicio de sesión personal para acciones de escritura.

Variables de entorno obligatorias:

- `OPENCLAW_GH_READ_APP_ID`
- `OPENCLAW_GH_READ_PRIVATE_KEY_FILE`

Variables de entorno opcionales:

- `OPENCLAW_GH_READ_INSTALLATION_ID` cuando quieras omitir la búsqueda de instalación basada en repositorio
- `OPENCLAW_GH_READ_PERMISSIONS` como anulación separada por comas para el subconjunto de permisos de lectura que se solicitará

Orden de resolución del repositorio:

- `gh ... -R owner/repo`
- `GH_REPO`
- `git remote origin`

Ejemplos:

- `scripts/gh-read pr view 123`
- `scripts/gh-read run list -R openclaw/openclaw`
- `scripts/gh-read api repos/openclaw/openclaw/pulls/123`

## Al agregar scripts

- Mantén los scripts enfocados y documentados.
- Agrega una entrada breve en la documentación pertinente (o crea una si falta).
