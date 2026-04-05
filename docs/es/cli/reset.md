---
read_when:
    - Quieres borrar el estado local manteniendo la CLI instalada
    - Quieres una simulación de lo que se eliminaría
summary: Referencia de CLI para `openclaw reset` (restablecer estado/configuración local)
title: reset
x-i18n:
    generated_at: "2026-04-05T12:38:37Z"
    model: gpt-5.4
    provider: openai
    source_hash: ad464700f948bebe741ec309f25150714f0b280834084d4f531327418a42c79b
    source_path: cli/reset.md
    workflow: 15
---

# `openclaw reset`

Restablece la configuración y el estado locales (mantiene la CLI instalada).

Opciones:

- `--scope <scope>`: `config`, `config+creds+sessions` o `full`
- `--yes`: omite las solicitudes de confirmación
- `--non-interactive`: desactiva las solicitudes; requiere `--scope` y `--yes`
- `--dry-run`: imprime las acciones sin eliminar archivos

Ejemplos:

```bash
openclaw backup create
openclaw reset
openclaw reset --dry-run
openclaw reset --scope config --yes --non-interactive
openclaw reset --scope config+creds+sessions --yes --non-interactive
openclaw reset --scope full --yes --non-interactive
```

Notas:

- Ejecuta primero `openclaw backup create` si quieres una instantánea restaurable antes de eliminar el estado local.
- Si omites `--scope`, `openclaw reset` usa una indicación interactiva para elegir qué eliminar.
- `--non-interactive` solo es válido cuando tanto `--scope` como `--yes` están establecidos.
