---
read_when:
    - Tienes problemas de conectividad/autenticación y quieres correcciones guiadas
    - Actualizaste y quieres una comprobación de estado
summary: Referencia de CLI para `openclaw doctor` (comprobaciones de estado + reparaciones guiadas)
title: doctor
x-i18n:
    generated_at: "2026-04-05T12:37:55Z"
    model: gpt-5.4
    provider: openai
    source_hash: d257a9e2797b4b0b50c1020165c8a1cd6a2342381bf9c351645ca37494c881e1
    source_path: cli/doctor.md
    workflow: 15
---

# `openclaw doctor`

Comprobaciones de estado + correcciones rápidas para el gateway y los canales.

Relacionado:

- Solución de problemas: [Solución de problemas](/gateway/troubleshooting)
- Auditoría de seguridad: [Seguridad](/gateway/security)

## Ejemplos

```bash
openclaw doctor
openclaw doctor --repair
openclaw doctor --deep
openclaw doctor --repair --non-interactive
openclaw doctor --generate-gateway-token
```

## Opciones

- `--no-workspace-suggestions`: desactiva las sugerencias de memoria/búsqueda del espacio de trabajo
- `--yes`: acepta los valores predeterminados sin preguntar
- `--repair`: aplica las reparaciones recomendadas sin preguntar
- `--fix`: alias de `--repair`
- `--force`: aplica reparaciones agresivas, incluida la sobrescritura de configuración de servicio personalizada cuando sea necesario
- `--non-interactive`: se ejecuta sin prompts; solo migraciones seguras
- `--generate-gateway-token`: genera y configura un token de gateway
- `--deep`: escanea los servicios del sistema en busca de instalaciones adicionales del gateway

Notas:

- Los prompts interactivos (como correcciones del llavero/OAuth) solo se ejecutan cuando stdin es un TTY y **no** está definido `--non-interactive`. Las ejecuciones sin interfaz (cron, Telegram, sin terminal) omiten los prompts.
- `--fix` (alias de `--repair`) escribe una copia de seguridad en `~/.openclaw/openclaw.json.bak` y elimina claves de configuración desconocidas, enumerando cada eliminación.
- Las comprobaciones de integridad del estado ahora detectan archivos de transcripción huérfanos en el directorio de sesiones y pueden archivarlos como `.deleted.<timestamp>` para recuperar espacio de forma segura.
- Doctor también escanea `~/.openclaw/cron/jobs.json` (o `cron.store`) en busca de formatos heredados de trabajos cron y puede reescribirlos en su lugar antes de que el programador tenga que normalizarlos automáticamente en tiempo de ejecución.
- Doctor migra automáticamente la configuración plana heredada de Talk (`talk.voiceId`, `talk.modelId` y similares) a `talk.provider` + `talk.providers.<provider>`.
- Las ejecuciones repetidas de `doctor --fix` ya no informan/aplican la normalización de Talk cuando la única diferencia es el orden de las claves del objeto.
- Doctor incluye una comprobación de preparación de memory-search y puede recomendar `openclaw configure --section model` cuando faltan credenciales de embeddings.
- Si el modo sandbox está habilitado pero Docker no está disponible, doctor informa una advertencia de alta señal con la solución (`install Docker` o `openclaw config set agents.defaults.sandbox.mode off`).
- Si `gateway.auth.token`/`gateway.auth.password` están administrados por SecretRef y no están disponibles en la ruta de comando actual, doctor informa una advertencia de solo lectura y no escribe credenciales de respaldo en texto plano.
- Si la inspección de SecretRef del canal falla en una ruta de corrección, doctor continúa e informa una advertencia en lugar de salir antes de tiempo.
- La autorresolución de nombre de usuario `allowFrom` de Telegram (`doctor --fix`) requiere un token de Telegram resoluble en la ruta de comando actual. Si la inspección del token no está disponible, doctor informa una advertencia y omite la autorresolución en esa ejecución.

## macOS: anulaciones de entorno de `launchctl`

Si antes ejecutaste `launchctl setenv OPENCLAW_GATEWAY_TOKEN ...` (o `...PASSWORD`), ese valor anula tu archivo de configuración y puede causar errores persistentes de “unauthorized”.

```bash
launchctl getenv OPENCLAW_GATEWAY_TOKEN
launchctl getenv OPENCLAW_GATEWAY_PASSWORD

launchctl unsetenv OPENCLAW_GATEWAY_TOKEN
launchctl unsetenv OPENCLAW_GATEWAY_PASSWORD
```
