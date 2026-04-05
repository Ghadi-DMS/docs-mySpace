---
read_when:
    - Quieres eliminar el servicio del gateway y/o el estado local
    - Quieres hacer primero una simulación
summary: Referencia de CLI para `openclaw uninstall` (eliminar el servicio del gateway + datos locales)
title: uninstall
x-i18n:
    generated_at: "2026-04-05T12:39:04Z"
    model: gpt-5.4
    provider: openai
    source_hash: 2123a4f9c7a070ef7e13c60dafc189053ef61ce189fa4f29449dd50987c1894c
    source_path: cli/uninstall.md
    workflow: 15
---

# `openclaw uninstall`

Desinstala el servicio del gateway + los datos locales (la CLI permanece).

Opciones:

- `--service`: elimina el servicio del gateway
- `--state`: elimina el estado y la configuración
- `--workspace`: elimina los directorios del espacio de trabajo
- `--app`: elimina la app de macOS
- `--all`: elimina el servicio, el estado, el espacio de trabajo y la app
- `--yes`: omite los prompts de confirmación
- `--non-interactive`: desactiva los prompts; requiere `--yes`
- `--dry-run`: muestra las acciones sin eliminar archivos

Ejemplos:

```bash
openclaw backup create
openclaw uninstall
openclaw uninstall --service --yes --non-interactive
openclaw uninstall --state --workspace --yes --non-interactive
openclaw uninstall --all --yes
openclaw uninstall --dry-run
```

Notas:

- Ejecuta primero `openclaw backup create` si quieres una instantánea restaurable antes de eliminar el estado o los espacios de trabajo.
- `--all` es una forma abreviada de eliminar juntos el servicio, el estado, el espacio de trabajo y la app.
- `--non-interactive` requiere `--yes`.
