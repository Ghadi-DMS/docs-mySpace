---
read_when:
    - Quieres ver qué Skills están disponibles y listas para ejecutarse
    - Quieres buscar, instalar o actualizar Skills desde ClawHub
    - Quieres depurar binarios/env/config faltantes para Skills
summary: Referencia de CLI para `openclaw skills` (`search`/`install`/`update`/`list`/`info`/`check`)
title: skills
x-i18n:
    generated_at: "2026-04-05T12:39:00Z"
    model: gpt-5.4
    provider: openai
    source_hash: 11af59b1b6bff19cc043acd8d67bdd4303201d3f75f23c948b83bf14882c7bb1
    source_path: cli/skills.md
    workflow: 15
---

# `openclaw skills`

Inspecciona las Skills locales e instala/actualiza Skills desde ClawHub.

Relacionado:

- Sistema de Skills: [Skills](/tools/skills)
- Configuración de Skills: [Skills config](/tools/skills-config)
- Instalaciones de ClawHub: [ClawHub](/tools/clawhub)

## Comandos

```bash
openclaw skills search "calendar"
openclaw skills search --limit 20 --json
openclaw skills install <slug>
openclaw skills install <slug> --version <version>
openclaw skills install <slug> --force
openclaw skills update <slug>
openclaw skills update --all
openclaw skills list
openclaw skills list --eligible
openclaw skills list --json
openclaw skills list --verbose
openclaw skills info <name>
openclaw skills info <name> --json
openclaw skills check
openclaw skills check --json
```

`search`/`install`/`update` usan ClawHub directamente e instalan en el directorio
`skills/` del espacio de trabajo activo. `list`/`info`/`check` siguen inspeccionando las
Skills locales visibles para el espacio de trabajo y la configuración actuales.

Este comando `install` de la CLI descarga carpetas de Skills desde ClawHub. Las
instalaciones de dependencias de Skills respaldadas por el gateway que se activan
desde el onboarding o la configuración de Skills usan en su lugar la ruta de
solicitud independiente `skills.install`.

Notas:

- `search [query...]` acepta una consulta opcional; omítela para explorar el feed de búsqueda predeterminado de ClawHub.
- `search --limit <n>` limita la cantidad de resultados devueltos.
- `install --force` sobrescribe una carpeta de Skill existente en el espacio de trabajo para el mismo `slug`.
- `update --all` solo actualiza las instalaciones rastreadas de ClawHub en el espacio de trabajo activo.
- `list` es la acción predeterminada cuando no se proporciona ningún subcomando.
- `list`, `info` y `check` escriben su salida renderizada en stdout. Con
  `--json`, eso significa que la carga legible por máquina permanece en stdout para tuberías y scripts.
