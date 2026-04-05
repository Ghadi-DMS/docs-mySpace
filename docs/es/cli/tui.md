---
read_when:
    - Quieres una interfaz de terminal para el Gateway (compatible con acceso remoto)
    - Quieres pasar url/token/session desde scripts
summary: Referencia de la CLI para `openclaw tui` (interfaz de terminal conectada al Gateway)
title: tui
x-i18n:
    generated_at: "2026-04-05T12:39:05Z"
    model: gpt-5.4
    provider: openai
    source_hash: 60e35062c0551f85ce0da604a915b3e1ca2514d00d840afe3b94c529304c2c1a
    source_path: cli/tui.md
    workflow: 15
---

# `openclaw tui`

Abre la interfaz de terminal conectada al Gateway.

Relacionado:

- Guía de la TUI: [TUI](/web/tui)

Notas:

- `tui` resuelve los SecretRefs de autenticación del gateway configurados para autenticación por token/contraseña cuando es posible (proveedores `env`/`file`/`exec`).
- Cuando se inicia desde dentro de un directorio de workspace de agente configurado, la TUI selecciona automáticamente ese agente como valor predeterminado de la clave de sesión (a menos que `--session` sea explícitamente `agent:<id>:...`).

## Ejemplos

```bash
openclaw tui
openclaw tui --url ws://127.0.0.1:18789 --token <token>
openclaw tui --session main --deliver
# cuando se ejecuta dentro de un workspace de agente, deduce ese agente automáticamente
openclaw tui --session bugfix
```
