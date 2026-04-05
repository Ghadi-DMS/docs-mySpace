---
read_when:
    - Quieres abrir la interfaz de usuario de Control con tu token actual
    - Quieres imprimir la URL sin iniciar un navegador
summary: Referencia de CLI para `openclaw dashboard` (abrir la interfaz de usuario de Control)
title: dashboard
x-i18n:
    generated_at: "2026-04-05T12:37:41Z"
    model: gpt-5.4
    provider: openai
    source_hash: a34cd109a3803e2910fcb4d32f2588aa205a4933819829ef5598f0780f586c94
    source_path: cli/dashboard.md
    workflow: 15
---

# `openclaw dashboard`

Abre la interfaz de usuario de Control usando tu autenticación actual.

```bash
openclaw dashboard
openclaw dashboard --no-open
```

Notas:

- `dashboard` resuelve SecretRefs configurados de `gateway.auth.token` cuando es posible.
- Para tokens gestionados por SecretRef (resueltos o no resueltos), `dashboard` imprime/copia/abre una URL sin token para evitar exponer secretos externos en la salida del terminal, el historial del portapapeles o los argumentos de inicio del navegador.
- Si `gateway.auth.token` está gestionado por SecretRef pero no resuelto en esta ruta del comando, el comando imprime una URL sin token e instrucciones explícitas de corrección en lugar de incrustar un placeholder de token no válido.
