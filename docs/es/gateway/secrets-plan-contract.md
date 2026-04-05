---
read_when:
    - Generación o revisión de planes de `openclaw secrets apply`
    - Depuración de errores `Invalid plan target path`
    - Comprensión del comportamiento de validación de tipo y ruta de destino
summary: 'Contrato para planes de `secrets apply`: validación de destino, coincidencia de ruta y alcance de destino de `auth-profiles.json`'
title: Contrato del plan de Secrets Apply
x-i18n:
    generated_at: "2026-04-05T12:43:01Z"
    model: gpt-5.4
    provider: openai
    source_hash: cb89a426ca937cf4d745f641b43b330c7fbb1aa9e4359b106ecd28d7a65ca327
    source_path: gateway/secrets-plan-contract.md
    workflow: 15
---

# Contrato del plan de secrets apply

Esta página define el contrato estricto aplicado por `openclaw secrets apply`.

Si un destino no coincide con estas reglas, la aplicación falla antes de mutar la configuración.

## Forma del archivo del plan

`openclaw secrets apply --from <plan.json>` espera un arreglo `targets` de destinos del plan:

```json5
{
  version: 1,
  protocolVersion: 1,
  targets: [
    {
      type: "models.providers.apiKey",
      path: "models.providers.openai.apiKey",
      pathSegments: ["models", "providers", "openai", "apiKey"],
      providerId: "openai",
      ref: { source: "env", provider: "default", id: "OPENAI_API_KEY" },
    },
    {
      type: "auth-profiles.api_key.key",
      path: "profiles.openai:default.key",
      pathSegments: ["profiles", "openai:default", "key"],
      agentId: "main",
      ref: { source: "env", provider: "default", id: "OPENAI_API_KEY" },
    },
  ],
}
```

## Alcance de destino compatible

Los destinos del plan se aceptan para rutas de credenciales compatibles en:

- [Superficie de credenciales SecretRef](/reference/secretref-credential-surface)

## Comportamiento del tipo de destino

Regla general:

- `target.type` debe ser un tipo de destino reconocido y debe coincidir con la forma normalizada de `target.path`.

Se siguen aceptando alias de compatibilidad para planes existentes:

- `models.providers.apiKey`
- `skills.entries.apiKey`
- `channels.googlechat.serviceAccount`

## Reglas de validación de rutas

Cada destino se valida con todo lo siguiente:

- `type` debe ser un tipo de destino reconocido.
- `path` debe ser una ruta con puntos no vacía.
- `pathSegments` puede omitirse. Si se proporciona, debe normalizarse exactamente a la misma ruta que `path`.
- Se rechazan segmentos prohibidos: `__proto__`, `prototype`, `constructor`.
- La ruta normalizada debe coincidir con la forma de ruta registrada para el tipo de destino.
- Si se establece `providerId` o `accountId`, debe coincidir con el id codificado en la ruta.
- Los destinos de `auth-profiles.json` requieren `agentId`.
- Al crear una nueva asignación de `auth-profiles.json`, incluye `authProfileProvider`.

## Comportamiento ante errores

Si un destino falla la validación, la aplicación sale con un error como:

```text
Invalid plan target path for models.providers.apiKey: models.providers.openai.baseUrl
```

No se confirma ninguna escritura para un plan no válido.

## Comportamiento de consentimiento para proveedor exec

- `--dry-run` omite de forma predeterminada las comprobaciones de SecretRef exec.
- Los planes que contienen SecretRefs/proveedores exec se rechazan en modo de escritura a menos que se establezca `--allow-exec`.
- Al validar/aplicar planes que contienen exec, pasa `--allow-exec` tanto en comandos de simulación como de escritura.

## Notas sobre alcance de runtime y auditoría

- Las entradas de `auth-profiles.json` solo con referencias (`keyRef`/`tokenRef`) se incluyen en la resolución en tiempo de ejecución y en la cobertura de auditoría.
- `secrets apply` escribe destinos compatibles de `openclaw.json`, destinos compatibles de `auth-profiles.json` y destinos opcionales de limpieza.

## Comprobaciones del operador

```bash
# Validate plan without writes
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run

# Then apply for real
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json

# For exec-containing plans, opt in explicitly in both modes
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run --allow-exec
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --allow-exec
```

Si apply falla con un mensaje de ruta de destino no válida, vuelve a generar el plan con `openclaw secrets configure` o corrige la ruta del destino a una forma compatible indicada arriba.

## Documentación relacionada

- [Gestión de secretos](/gateway/secrets)
- [CLI `secrets`](/cli/secrets)
- [Superficie de credenciales SecretRef](/reference/secretref-credential-surface)
- [Referencia de configuración](/gateway/configuration-reference)
