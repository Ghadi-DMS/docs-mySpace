---
read_when:
    - Quieres autocompletado del shell para zsh/bash/fish/PowerShell
    - Necesitas almacenar en caché scripts de autocompletado en el estado de OpenClaw
summary: Referencia de la CLI para `openclaw completion` (generar/instalar scripts de autocompletado del shell)
title: completion
x-i18n:
    generated_at: "2026-04-05T12:37:29Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7bbf140a880bafdb7140149f85465d66d0d46e5a3da6a1e41fb78be2fd2bd4d0
    source_path: cli/completion.md
    workflow: 15
---

# `openclaw completion`

Genera scripts de autocompletado del shell y, opcionalmente, los instala en el perfil de tu shell.

## Uso

```bash
openclaw completion
openclaw completion --shell zsh
openclaw completion --install
openclaw completion --shell fish --install
openclaw completion --write-state
openclaw completion --shell bash --write-state
```

## Opciones

- `-s, --shell <shell>`: destino del shell (`zsh`, `bash`, `powershell`, `fish`; predeterminado: `zsh`)
- `-i, --install`: instala el autocompletado añadiendo una línea `source` a tu perfil de shell
- `--write-state`: escribe script(s) de autocompletado en `$OPENCLAW_STATE_DIR/completions` sin imprimirlos en stdout
- `-y, --yes`: omite las solicitudes de confirmación de instalación

## Notas

- `--install` escribe un pequeño bloque "OpenClaw Completion" en el perfil de tu shell y lo apunta al script en caché.
- Sin `--install` ni `--write-state`, el comando imprime el script en stdout.
- La generación de autocompletado carga de forma anticipada los árboles de comandos para que se incluyan los subcomandos anidados.
