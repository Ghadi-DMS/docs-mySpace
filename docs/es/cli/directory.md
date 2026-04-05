---
read_when:
    - Quieres buscar contactos/grupos/IDs propios para un canal
    - Estás desarrollando un adaptador de directorio de canal
summary: Referencia de la CLI para `openclaw directory` (yo, pares, grupos)
title: directory
x-i18n:
    generated_at: "2026-04-05T12:37:48Z"
    model: gpt-5.4
    provider: openai
    source_hash: 6a81a037e0a33f77c24b1adabbc4be16ed4d03c419873f3cbdd63f2ce84a1064
    source_path: cli/directory.md
    workflow: 15
---

# `openclaw directory`

Búsquedas de directorio para los canales que lo admiten (contactos/pares, grupos y “yo”).

## Indicadores comunes

- `--channel <name>`: ID/alias del canal (obligatorio cuando hay varios canales configurados; automático cuando solo hay uno configurado)
- `--account <id>`: ID de la cuenta (predeterminado: cuenta predeterminada del canal)
- `--json`: salida JSON

## Notas

- `directory` está pensado para ayudarte a encontrar IDs que puedas pegar en otros comandos (especialmente `openclaw message send --target ...`).
- En muchos canales, los resultados se basan en la configuración (listas de permitidos / grupos configurados) en lugar de un directorio activo del proveedor.
- La salida predeterminada es `id` (y a veces `name`) separada por una tabulación; usa `--json` para scripting.

## Uso de resultados con `message send`

```bash
openclaw directory peers list --channel slack --query "U0"
openclaw message send --channel slack --target user:U012ABCDEF --message "hello"
```

## Formatos de ID (por canal)

- WhatsApp: `+15551234567` (MD), `1234567890-1234567890@g.us` (grupo)
- Telegram: `@username` o ID numérico de chat; los grupos son IDs numéricos
- Slack: `user:U…` y `channel:C…`
- Discord: `user:<id>` y `channel:<id>`
- Matrix (plugin): `user:@user:server`, `room:!roomId:server` o `#alias:server`
- Microsoft Teams (plugin): `user:<id>` y `conversation:<id>`
- Zalo (plugin): ID de usuario (Bot API)
- Zalo Personal / `zalouser` (plugin): ID del hilo (MD/grupo) de `zca` (`me`, `friend list`, `group list`)

## Yo ("me")

```bash
openclaw directory self --channel zalouser
```

## Pares (contactos/usuarios)

```bash
openclaw directory peers list --channel zalouser
openclaw directory peers list --channel zalouser --query "name"
openclaw directory peers list --channel zalouser --limit 50
```

## Grupos

```bash
openclaw directory groups list --channel zalouser
openclaw directory groups list --channel zalouser --query "work"
openclaw directory groups members --channel zalouser --group-id <id>
```
