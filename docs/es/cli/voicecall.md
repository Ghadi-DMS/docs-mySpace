---
read_when:
    - Usas el plugin de llamadas de voz y quieres los puntos de entrada de CLI
    - Quieres ejemplos rápidos para `voicecall call|continue|status|tail|expose`
summary: Referencia de CLI para `openclaw voicecall` (superficie de comandos del plugin de llamadas de voz)
title: voicecall
x-i18n:
    generated_at: "2026-04-05T12:39:04Z"
    model: gpt-5.4
    provider: openai
    source_hash: 2c99e7a3d256e1c74a0f07faba9675cc5a88b1eb2fc6e22993caf3874d4f340a
    source_path: cli/voicecall.md
    workflow: 15
---

# `openclaw voicecall`

`voicecall` es un comando proporcionado por un plugin. Solo aparece si el plugin de llamadas de voz está instalado y habilitado.

Documento principal:

- Plugin de llamadas de voz: [Voice Call](/plugins/voice-call)

## Comandos comunes

```bash
openclaw voicecall status --call-id <id>
openclaw voicecall call --to "+15555550123" --message "Hello" --mode notify
openclaw voicecall continue --call-id <id> --message "Any questions?"
openclaw voicecall end --call-id <id>
```

## Exponer webhooks (Tailscale)

```bash
openclaw voicecall expose --mode serve
openclaw voicecall expose --mode funnel
openclaw voicecall expose --mode off
```

Nota de seguridad: expón el endpoint de webhook solo a redes en las que confíes. Prefiere Tailscale Serve en lugar de Funnel cuando sea posible.
