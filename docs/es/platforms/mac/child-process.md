---
read_when:
    - Integrando la app de macOS con el ciclo de vida del gateway
summary: Ciclo de vida del Gateway en macOS (launchd)
title: Ciclo de vida del Gateway
x-i18n:
    generated_at: "2026-04-05T12:48:11Z"
    model: gpt-5.4
    provider: openai
    source_hash: 73e7eb64ef432c3bfc81b949a5cc2a344c64f2310b794228609aae1da817ec41
    source_path: platforms/mac/child-process.md
    workflow: 15
---

# Ciclo de vida del gateway en macOS

La app de macOS **gestiona el Gateway mediante launchd** por defecto y no inicia
el Gateway como proceso hijo. Primero intenta conectarse a un
Gateway ya en ejecución en el puerto configurado; si no encuentra ninguno accesible,
habilita el servicio launchd mediante la CLI externa `openclaw` (sin tiempo de ejecución integrado). Esto te da
un inicio automático fiable al iniciar sesión y reinicio tras fallos.

El modo de proceso hijo (Gateway iniciado directamente por la app) **no se usa** hoy.
Si necesitas un acoplamiento más estrecho con la UI, ejecuta el Gateway manualmente en una terminal.

## Comportamiento predeterminado (launchd)

- La app instala un LaunchAgent por usuario con la etiqueta `ai.openclaw.gateway`
  (o `ai.openclaw.<profile>` cuando se usa `--profile`/`OPENCLAW_PROFILE`; se admite el heredado `com.openclaw.*`).
- Cuando el modo local está habilitado, la app se asegura de que el LaunchAgent esté cargado e
  inicia el Gateway si es necesario.
- Los logs se escriben en la ruta del log del gateway de launchd (visible en Debug Settings).

Comandos comunes:

```bash
launchctl kickstart -k gui/$UID/ai.openclaw.gateway
launchctl bootout gui/$UID/ai.openclaw.gateway
```

Sustituye la etiqueta por `ai.openclaw.<profile>` cuando ejecutes un profile con nombre.

## Compilaciones de desarrollo sin firma

`scripts/restart-mac.sh --no-sign` es para compilaciones locales rápidas cuando no tienes
claves de firma. Para impedir que launchd apunte a un binario relay sin firmar, hace lo siguiente:

- Escribe `~/.openclaw/disable-launchagent`.

Las ejecuciones firmadas de `scripts/restart-mac.sh` eliminan esta sobrescritura si el marcador está
presente. Para restablecerlo manualmente:

```bash
rm ~/.openclaw/disable-launchagent
```

## Modo solo conexión

Para forzar que la app de macOS **nunca instale ni gestione launchd**, iníciala con
`--attach-only` (o `--no-launchd`). Esto configura `~/.openclaw/disable-launchagent`,
de modo que la app solo se conecta a un Gateway ya en ejecución. También puedes alternar este
comportamiento en Debug Settings.

## Modo remoto

El modo remoto nunca inicia un Gateway local. La app usa un túnel SSH hacia el
host remoto y se conecta a través de ese túnel.

## Por qué preferimos launchd

- Inicio automático al iniciar sesión.
- Semántica integrada de reinicio/KeepAlive.
- Logs y supervisión predecibles.

Si alguna vez vuelve a necesitarse un verdadero modo de proceso hijo, debería documentarse como un
modo independiente y explícito solo para desarrollo.
