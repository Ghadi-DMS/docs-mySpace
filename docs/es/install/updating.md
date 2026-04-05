---
read_when:
    - Actualizar OpenClaw
    - Algo se rompe después de una actualización
summary: Actualizar OpenClaw de forma segura (instalación global o desde código fuente), además de la estrategia de reversión
title: Actualización
x-i18n:
    generated_at: "2026-04-05T12:46:51Z"
    model: gpt-5.4
    provider: openai
    source_hash: b40429d38ca851be4fdf8063ed425faf4610a4b5772703e0481c5f1fb588ba58
    source_path: install/updating.md
    workflow: 15
---

# Actualización

Mantén OpenClaw al día.

## Recomendado: `openclaw update`

La forma más rápida de actualizar. Detecta tu tipo de instalación (npm o git), obtiene la versión más reciente, ejecuta `openclaw doctor` y reinicia el gateway.

```bash
openclaw update
```

Para cambiar de canal o apuntar a una versión específica:

```bash
openclaw update --channel beta
openclaw update --tag main
openclaw update --dry-run   # vista previa sin aplicar
```

`--channel beta` da preferencia a beta, pero el tiempo de ejecución recurre a stable/latest cuando
la etiqueta beta falta o es más antigua que la versión estable más reciente. Usa `--tag beta`
si quieres el dist-tag beta sin procesar de npm para una actualización puntual del paquete.

Consulta [Canales de desarrollo](/install/development-channels) para conocer la semántica de los canales.

## Alternativa: volver a ejecutar el instalador

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

Agrega `--no-onboard` para omitir la incorporación. Para instalaciones desde código fuente, pasa `--install-method git --no-onboard`.

## Alternativa: npm, pnpm o bun manuales

```bash
npm i -g openclaw@latest
```

```bash
pnpm add -g openclaw@latest
```

```bash
bun add -g openclaw@latest
```

## Actualizador automático

El actualizador automático está desactivado de forma predeterminada. Habilítalo en `~/.openclaw/openclaw.json`:

```json5
{
  update: {
    channel: "stable",
    auto: {
      enabled: true,
      stableDelayHours: 6,
      stableJitterHours: 12,
      betaCheckIntervalHours: 1,
    },
  },
}
```

| Canal    | Comportamiento                                                                                                 |
| -------- | -------------------------------------------------------------------------------------------------------------- |
| `stable` | Espera `stableDelayHours`, luego aplica con jitter determinista en `stableJitterHours` (despliegue escalonado). |
| `beta`   | Comprueba cada `betaCheckIntervalHours` (predeterminado: cada hora) y aplica inmediatamente.                  |
| `dev`    | Sin aplicación automática. Usa `openclaw update` manualmente.                                                 |

El gateway también registra una sugerencia de actualización al iniciarse (desactívala con `update.checkOnStart: false`).

## Después de actualizar

<Steps>

### Ejecuta doctor

```bash
openclaw doctor
```

Migra la configuración, audita las políticas de DM y comprueba el estado del gateway. Detalles: [Doctor](/gateway/doctor)

### Reinicia el gateway

```bash
openclaw gateway restart
```

### Verifica

```bash
openclaw health
```

</Steps>

## Reversión

### Fijar una versión (npm)

```bash
npm i -g openclaw@<version>
openclaw doctor
openclaw gateway restart
```

Consejo: `npm view openclaw version` muestra la versión publicada actual.

### Fijar un commit (código fuente)

```bash
git fetch origin
git checkout "$(git rev-list -n 1 --before=\"2026-01-01\" origin/main)"
pnpm install && pnpm build
openclaw gateway restart
```

Para volver a la versión más reciente: `git checkout main && git pull`.

## Si estás atascado

- Ejecuta `openclaw doctor` de nuevo y lee la salida con atención.
- Consulta: [Solución de problemas](/gateway/troubleshooting)
- Pregunta en Discord: [https://discord.gg/clawd](https://discord.gg/clawd)

## Relacionado

- [Resumen de instalación](/install) — todos los métodos de instalación
- [Doctor](/gateway/doctor) — comprobaciones de salud después de las actualizaciones
- [Migración](/install/migrating) — guías de migración de versiones principales
