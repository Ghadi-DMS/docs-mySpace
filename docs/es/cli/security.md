---
read_when:
    - Quieres ejecutar una auditoría rápida de seguridad sobre la configuración/el estado
    - Quieres aplicar sugerencias seguras de “fix” (permisos, endurecer valores predeterminados)
summary: Referencia de CLI para `openclaw security` (auditar y corregir errores comunes de seguridad)
title: security
x-i18n:
    generated_at: "2026-04-05T12:38:59Z"
    model: gpt-5.4
    provider: openai
    source_hash: e5a3e4ab8e0dfb6c10763097cb4483be2431985f16de877523eb53e2122239ae
    source_path: cli/security.md
    workflow: 15
---

# `openclaw security`

Herramientas de seguridad (auditoría + correcciones opcionales).

Relacionado:

- Guía de seguridad: [Seguridad](/gateway/security)

## Auditoría

```bash
openclaw security audit
openclaw security audit --deep
openclaw security audit --deep --password <password>
openclaw security audit --deep --token <token>
openclaw security audit --fix
openclaw security audit --json
```

La auditoría advierte cuando varios remitentes de mensajes directos comparten la sesión principal y recomienda el **modo seguro de mensajes directos**: `session.dmScope="per-channel-peer"` (o `per-account-channel-peer` para canales con varias cuentas) para bandejas de entrada compartidas.
Esto es para endurecimiento de bandejas de entrada cooperativas/compartidas. Un único Gateway compartido por operadores mutuamente desconfiados/adversarios no es una configuración recomendada; separa los límites de confianza con gateways separados (o usuarios/hosts del sistema operativo separados).
También emite `security.trust_model.multi_user_heuristic` cuando la configuración sugiere un ingreso probable de usuarios compartidos (por ejemplo, política abierta de mensajes directos/grupos, destinos de grupo configurados o reglas comodín de remitentes), y recuerda que OpenClaw usa de forma predeterminada un modelo de confianza de asistente personal.
Para configuraciones intencionales de usuarios compartidos, la guía de auditoría es poner todas las sesiones en sandbox, mantener el acceso al sistema de archivos limitado al espacio de trabajo y mantener identidades o credenciales personales/privadas fuera de ese tiempo de ejecución.
También advierte cuando se usan modelos pequeños (`<=300B`) sin sandboxing y con herramientas web/navegador habilitadas.
Para el ingreso por webhook, advierte cuando `hooks.token` reutiliza el token del Gateway, cuando `hooks.token` es corto, cuando `hooks.path="/"`, cuando `hooks.defaultSessionKey` no está definido, cuando `hooks.allowedAgentIds` no está restringido, cuando están habilitadas las anulaciones de `sessionKey` de la solicitud y cuando esas anulaciones están habilitadas sin `hooks.allowedSessionKeyPrefixes`.
También advierte cuando los ajustes Docker del sandbox están configurados mientras el modo sandbox está desactivado, cuando `gateway.nodes.denyCommands` usa entradas ineficaces de tipo patrón/desconocidas (solo coincidencia exacta por nombre de comando del nodo, no filtrado de texto de shell), cuando `gateway.nodes.allowCommands` habilita explícitamente comandos peligrosos del nodo, cuando el `tools.profile="minimal"` global es anulado por perfiles de herramientas del agente, cuando grupos abiertos exponen herramientas de tiempo de ejecución/sistema de archivos sin protecciones de sandbox/espacio de trabajo y cuando las herramientas de plugins de extensión instalados podrían ser accesibles bajo una política de herramientas permisiva.
También marca `gateway.allowRealIpFallback=true` (riesgo de suplantación de encabezados si los proxies están mal configurados) y `discovery.mdns.mode="full"` (filtración de metadatos mediante registros TXT de mDNS).
También advierte cuando el navegador sandbox usa red Docker `bridge` sin `sandbox.browser.cdpSourceRange`.
También marca modos de red Docker peligrosos para el sandbox (incluidos `host` y uniones de espacios de nombres `container:*`).
También advierte cuando los contenedores Docker existentes del navegador sandbox tienen etiquetas hash faltantes/obsoletas (por ejemplo, contenedores anteriores a la migración sin `openclaw.browserConfigEpoch`) y recomienda `openclaw sandbox recreate --browser --all`.
También advierte cuando los registros de instalación de plugins/hooks basados en npm no están fijados, carecen de metadatos de integridad o se desvían de las versiones de paquetes instaladas actualmente.
Advierte cuando las listas de permitidos de canales dependen de nombres/correos electrónicos/etiquetas mutables en lugar de ID estables (Discord, Slack, Google Chat, Microsoft Teams, Mattermost, ámbitos de IRC cuando corresponda).
Advierte cuando `gateway.auth.mode="none"` deja las API HTTP del Gateway accesibles sin un secreto compartido (`/tools/invoke` más cualquier endpoint `/v1/*` habilitado).
Los ajustes con prefijo `dangerous`/`dangerously` son anulaciones explícitas del operador en caso de emergencia; habilitar uno de ellos no constituye, por sí solo, un informe de vulnerabilidad de seguridad.
Para el inventario completo de parámetros peligrosos, consulta la sección "Insecure or dangerous flags summary" en [Security](/gateway/security).

Comportamiento de SecretRef:

- `security audit` resuelve los SecretRef compatibles en modo de solo lectura para sus rutas objetivo.
- Si un SecretRef no está disponible en la ruta de comando actual, la auditoría continúa e informa `secretDiagnostics` (en lugar de fallar).
- `--token` y `--password` solo anulan la autenticación del sondeo profundo para esa invocación del comando; no reescriben la configuración ni las asignaciones de SecretRef.

## Salida JSON

Usa `--json` para comprobaciones de CI/políticas:

```bash
openclaw security audit --json | jq '.summary'
openclaw security audit --deep --json | jq '.findings[] | select(.severity=="critical") | .checkId'
```

Si se combinan `--fix` y `--json`, la salida incluye tanto las acciones de corrección como el informe final:

```bash
openclaw security audit --fix --json | jq '{fix: .fix.ok, summary: .report.summary}'
```

## Qué cambia `--fix`

`--fix` aplica correcciones seguras y deterministas:

- cambia `groupPolicy="open"` a `groupPolicy="allowlist"` en casos comunes (incluidas variantes por cuenta en canales compatibles)
- cuando la política de grupos de WhatsApp cambia a `allowlist`, inicializa `groupAllowFrom` a partir del
  archivo `allowFrom` almacenado cuando esa lista existe y la configuración aún no
  define `allowFrom`
- establece `logging.redactSensitive` de `"off"` a `"tools"`
- endurece los permisos de estado/configuración y de archivos sensibles comunes
  (`credentials/*.json`, `auth-profiles.json`, `sessions.json`, sesión
  `*.jsonl`)
- también endurece los archivos incluidos de configuración referenciados desde `openclaw.json`
- usa `chmod` en hosts POSIX y restablecimientos `icacls` en Windows

`--fix` **no**:

- rota tokens/contraseñas/claves API
- deshabilita herramientas (`gateway`, `cron`, `exec`, etc.)
- cambia las opciones de exposición de enlace/autenticación/red del gateway
- elimina ni reescribe plugins/Skills
