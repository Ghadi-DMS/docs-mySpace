---
read_when:
    - Resolver de nuevo refs de secretos en runtime
    - Auditar residuos en texto sin formato y refs sin resolver
    - Configurar SecretRefs y aplicar cambios unidireccionales de depuración
summary: Referencia de CLI para `openclaw secrets` (recargar, auditar, configurar, aplicar)
title: secrets
x-i18n:
    generated_at: "2026-04-05T12:39:00Z"
    model: gpt-5.4
    provider: openai
    source_hash: f436ba089d752edb766c0a3ce746ee6bca1097b22c9b30e3d9715cb0bb50bf47
    source_path: cli/secrets.md
    workflow: 15
---

# `openclaw secrets`

Usa `openclaw secrets` para gestionar SecretRefs y mantener en buen estado la instantánea activa de runtime.

Funciones de los comandos:

- `reload`: RPC de gateway (`secrets.reload`) que vuelve a resolver refs e intercambia la instantánea de runtime solo si todo se completa correctamente (sin escrituras de configuración).
- `audit`: análisis de solo lectura del almacenamiento de configuración/autenticación/modelos generados y residuos heredados para detectar texto sin formato, refs sin resolver y desajustes de precedencia (las refs `exec` se omiten salvo que se establezca `--allow-exec`).
- `configure`: planificador interactivo para configuración de proveedores, asignación de destinos y preflight (requiere TTY).
- `apply`: ejecuta un plan guardado (`--dry-run` solo para validación; el dry-run omite comprobaciones `exec` de forma predeterminada, y el modo de escritura rechaza planes que contienen `exec` salvo que se establezca `--allow-exec`), luego depura los residuos en texto sin formato seleccionados.

Bucle recomendado para operadores:

```bash
openclaw secrets audit --check
openclaw secrets configure
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json
openclaw secrets audit --check
openclaw secrets reload
```

Si tu plan incluye SecretRefs/proveedores `exec`, pasa `--allow-exec` tanto en los comandos `apply` de dry-run como en los de escritura.

Nota sobre códigos de salida para CI/puertas:

- `audit --check` devuelve `1` cuando hay hallazgos.
- las refs sin resolver devuelven `2`.

Relacionado:

- Guía de secretos: [Secrets Management](/gateway/secrets)
- Superficie de credenciales: [SecretRef Credential Surface](/reference/secretref-credential-surface)
- Guía de seguridad: [Security](/gateway/security)

## Recargar la instantánea de runtime

Vuelve a resolver refs de secretos e intercambia atómicamente la instantánea de runtime.

```bash
openclaw secrets reload
openclaw secrets reload --json
openclaw secrets reload --url ws://127.0.0.1:18789 --token <token>
```

Notas:

- Usa el método RPC de gateway `secrets.reload`.
- Si la resolución falla, la gateway conserva la última instantánea válida conocida y devuelve un error (sin activación parcial).
- La respuesta JSON incluye `warningCount`.

Opciones:

- `--url <url>`
- `--token <token>`
- `--timeout <ms>`
- `--json`

## Auditoría

Analiza el estado de OpenClaw para detectar:

- almacenamiento de secretos en texto sin formato
- refs sin resolver
- desajuste de precedencia (credenciales de `auth-profiles.json` que ocultan refs de `openclaw.json`)
- residuos generados en `agents/*/agent/models.json` (valores `apiKey` del proveedor y encabezados sensibles del proveedor)
- residuos heredados (entradas del almacén de autenticación heredado, recordatorios de OAuth)

Nota sobre residuos de encabezados:

- La detección de encabezados sensibles del proveedor se basa en heurísticas de nombre (nombres y fragmentos comunes de encabezados de autenticación/credenciales como `authorization`, `x-api-key`, `token`, `secret`, `password` y `credential`).

```bash
openclaw secrets audit
openclaw secrets audit --check
openclaw secrets audit --json
openclaw secrets audit --allow-exec
```

Comportamiento de salida:

- `--check` sale con código distinto de cero cuando hay hallazgos.
- las refs sin resolver salen con un código distinto de cero de mayor prioridad.

Aspectos destacados de la forma del informe:

- `status`: `clean | findings | unresolved`
- `resolution`: `refsChecked`, `skippedExecRefs`, `resolvabilityComplete`
- `summary`: `plaintextCount`, `unresolvedRefCount`, `shadowedRefCount`, `legacyResidueCount`
- códigos de hallazgo:
  - `PLAINTEXT_FOUND`
  - `REF_UNRESOLVED`
  - `REF_SHADOWED`
  - `LEGACY_RESIDUE`

## Configurar (asistente interactivo)

Crea cambios de proveedor y SecretRef de forma interactiva, ejecuta preflight y, opcionalmente, aplícalos:

```bash
openclaw secrets configure
openclaw secrets configure --plan-out /tmp/openclaw-secrets-plan.json
openclaw secrets configure --apply --yes
openclaw secrets configure --providers-only
openclaw secrets configure --skip-provider-setup
openclaw secrets configure --agent ops
openclaw secrets configure --json
```

Flujo:

- Primero, configuración de proveedores (`add/edit/remove` para alias de `secrets.providers`).
- Segundo, asignación de credenciales (selecciona campos y asigna refs `{source, provider, id}`).
- Por último, preflight y aplicación opcional.

Flags:

- `--providers-only`: configura solo `secrets.providers`; omite la asignación de credenciales.
- `--skip-provider-setup`: omite la configuración de proveedores y asigna credenciales a proveedores existentes.
- `--agent <id>`: limita el descubrimiento de destinos y las escrituras de `auth-profiles.json` a un único almacén de agente.
- `--allow-exec`: permite comprobaciones `exec` de SecretRef durante preflight/apply (puede ejecutar comandos del proveedor).

Notas:

- Requiere un TTY interactivo.
- No puedes combinar `--providers-only` con `--skip-provider-setup`.
- `configure` apunta a campos con secretos en `openclaw.json` además de `auth-profiles.json` para el ámbito de agente seleccionado.
- `configure` admite crear nuevas asignaciones de `auth-profiles.json` directamente en el flujo del selector.
- Superficie canónica compatible: [SecretRef Credential Surface](/reference/secretref-credential-surface).
- Realiza resolución preflight antes de aplicar.
- Si el preflight/apply incluye refs `exec`, mantén `--allow-exec` activado en ambos pasos.
- Los planes generados usan de forma predeterminada opciones de depuración (`scrubEnv`, `scrubAuthProfilesForProviderTargets`, `scrubLegacyAuthJson` todas habilitadas).
- La ruta de aplicación es unidireccional para los valores en texto sin formato depurados.
- Sin `--apply`, la CLI sigue preguntando `Apply this plan now?` después del preflight.
- Con `--apply` (y sin `--yes`), la CLI solicita una confirmación adicional irreversible.
- `--json` imprime el plan + el informe de preflight, pero el comando sigue requiriendo un TTY interactivo.

Nota de seguridad para proveedores exec:

- Las instalaciones de Homebrew suelen exponer binarios enlazados simbólicamente en `/opt/homebrew/bin/*`.
- Establece `allowSymlinkCommand: true` solo cuando sea necesario para rutas de gestores de paquetes de confianza, y combínalo con `trustedDirs` (por ejemplo `["/opt/homebrew"]`).
- En Windows, si la verificación ACL no está disponible para una ruta de proveedor, OpenClaw falla de forma cerrada. Solo para rutas de confianza, establece `allowInsecurePath: true` en ese proveedor para omitir las comprobaciones de seguridad de ruta.

## Aplicar un plan guardado

Aplica o ejecuta preflight de un plan generado previamente:

```bash
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --allow-exec
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run --allow-exec
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --json
```

Comportamiento de exec:

- `--dry-run` valida el preflight sin escribir archivos.
- Las comprobaciones `exec` de SecretRef se omiten de forma predeterminada en dry-run.
- El modo de escritura rechaza planes que contienen SecretRefs/proveedores `exec` salvo que se establezca `--allow-exec`.
- Usa `--allow-exec` para activar explícitamente las comprobaciones/ejecución de proveedores exec en cualquiera de los dos modos.

Detalles del contrato del plan (rutas de destino permitidas, reglas de validación y semántica de errores):

- [Secrets Apply Plan Contract](/gateway/secrets-plan-contract)

Qué puede actualizar `apply`:

- `openclaw.json` (destinos SecretRef + upserts/deletes de proveedores)
- `auth-profiles.json` (depuración de destinos de proveedor)
- residuos heredados en `auth.json`
- claves secretas conocidas de `~/.openclaw/.env` cuyos valores fueron migrados

## Por qué no hay copias de seguridad para rollback

`secrets apply` intencionadamente no escribe copias de seguridad de rollback que contengan los valores antiguos en texto sin formato.

La seguridad proviene de un preflight estricto + una aplicación casi atómica con restauración en memoria en el mejor esfuerzo si se produce un fallo.

## Ejemplo

```bash
openclaw secrets audit --check
openclaw secrets configure
openclaw secrets audit --check
```

Si `audit --check` sigue informando hallazgos de texto sin formato, actualiza las rutas de destino restantes que se indiquen y vuelve a ejecutar la auditoría.
