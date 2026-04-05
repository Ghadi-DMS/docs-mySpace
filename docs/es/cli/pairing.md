---
read_when:
    - Estás usando MD en modo de emparejamiento y necesitas aprobar remitentes
summary: Referencia de la CLI para `openclaw pairing` (aprobar/listar solicitudes de emparejamiento)
title: pairing
x-i18n:
    generated_at: "2026-04-05T12:38:34Z"
    model: gpt-5.4
    provider: openai
    source_hash: 122a608ef83ec2b1011fdfd1b59b94950a4dcc8b598335b0956e2eedece4958f
    source_path: cli/pairing.md
    workflow: 15
---

# `openclaw pairing`

Aprueba o inspecciona solicitudes de emparejamiento de MD (para canales que admiten emparejamiento).

Relacionado:

- Flujo de emparejamiento: [Emparejamiento](/channels/pairing)

## Comandos

```bash
openclaw pairing list telegram
openclaw pairing list --channel telegram --account work
openclaw pairing list telegram --json

openclaw pairing approve <code>
openclaw pairing approve telegram <code>
openclaw pairing approve --channel telegram --account work <code> --notify
```

## `pairing list`

Lista las solicitudes de emparejamiento pendientes para un canal.

Opciones:

- `[channel]`: ID de canal posicional
- `--channel <channel>`: ID de canal explícito
- `--account <accountId>`: ID de cuenta para canales con varias cuentas
- `--json`: salida legible por máquina

Notas:

- Si hay varios canales con capacidad de emparejamiento configurados, debes proporcionar un canal ya sea de forma posicional o con `--channel`.
- Se permiten canales de extensión siempre que el ID del canal sea válido.

## `pairing approve`

Aprueba un código de emparejamiento pendiente y permite ese remitente.

Uso:

- `openclaw pairing approve <channel> <code>`
- `openclaw pairing approve --channel <channel> <code>`
- `openclaw pairing approve <code>` cuando hay exactamente un canal con capacidad de emparejamiento configurado

Opciones:

- `--channel <channel>`: ID de canal explícito
- `--account <accountId>`: ID de cuenta para canales con varias cuentas
- `--notify`: enviar una confirmación de vuelta al solicitante en el mismo canal

## Notas

- Entrada de canal: pásala de forma posicional (`pairing list telegram`) o con `--channel <channel>`.
- `pairing list` admite `--account <accountId>` para canales con varias cuentas.
- `pairing approve` admite `--account <accountId>` y `--notify`.
- Si solo hay un canal con capacidad de emparejamiento configurado, se permite `pairing approve <code>`.
