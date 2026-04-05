---
read_when:
    - Trabajando en la resolución de perfiles de autenticación o el enrutamiento de credenciales
    - Depurando fallos de autenticación del modelo o el orden de perfiles
summary: Semántica canónica de elegibilidad y resolución de credenciales para perfiles de autenticación
title: Semántica de credenciales de autenticación
x-i18n:
    generated_at: "2026-04-05T12:34:15Z"
    model: gpt-5.4
    provider: openai
    source_hash: a4cd3e16cd25eb22c5e707311d06a19df1a59747ee3261c2d32c534a245fd7fb
    source_path: auth-credential-semantics.md
    workflow: 15
---

# Semántica de credenciales de autenticación

Este documento define la semántica canónica de elegibilidad y resolución de credenciales utilizada en:

- `resolveAuthProfileOrder`
- `resolveApiKeyForProfile`
- `models status --probe`
- `doctor-auth`

El objetivo es mantener alineado el comportamiento en el momento de la selección y en tiempo de ejecución.

## Códigos estables de motivo de sondeo

- `ok`
- `excluded_by_auth_order`
- `missing_credential`
- `invalid_expires`
- `expired`
- `unresolved_ref`
- `no_model`

## Credenciales de token

Las credenciales de token (`type: "token"`) admiten `token` en línea y/o `tokenRef`.

### Reglas de elegibilidad

1. Un perfil de token no es elegible cuando faltan tanto `token` como `tokenRef`.
2. `expires` es opcional.
3. Si `expires` está presente, debe ser un número finito mayor que `0`.
4. Si `expires` no es válido (`NaN`, `0`, negativo, no finito o de tipo incorrecto), el perfil no es elegible con `invalid_expires`.
5. Si `expires` está en el pasado, el perfil no es elegible con `expired`.
6. `tokenRef` no omite la validación de `expires`.

### Reglas de resolución

1. La semántica del resolvedor coincide con la semántica de elegibilidad para `expires`.
2. Para los perfiles elegibles, el material del token puede resolverse desde el valor en línea o desde `tokenRef`.
3. Las referencias que no se pueden resolver producen `unresolved_ref` en la salida de `models status --probe`.

## Filtrado explícito del orden de autenticación

- Cuando `auth.order.<provider>` o la anulación del orden del almacén de autenticación está establecida para un proveedor, `models status --probe` solo sondea los id de perfil que permanecen en el orden de autenticación resuelto para ese proveedor.
- Un perfil almacenado para ese proveedor que se omite del orden explícito no se intenta silenciosamente más tarde. La salida del sondeo lo informa con `reasonCode: excluded_by_auth_order` y el detalle `Excluded by auth.order for this provider.`

## Resolución de destinos de sondeo

- Los destinos de sondeo pueden provenir de perfiles de autenticación, credenciales del entorno o `models.json`.
- Si un proveedor tiene credenciales pero OpenClaw no puede resolver un candidato de modelo apto para sondeo para él, `models status --probe` informa `status: no_model` con `reasonCode: no_model`.

## Protección de la política de SecretRef para OAuth

- La entrada SecretRef es solo para credenciales estáticas.
- Si una credencial de perfil es `type: "oauth"`, los objetos SecretRef no son compatibles con ese material de credencial del perfil.
- Si `auth.profiles.<id>.mode` es `"oauth"`, la entrada `keyRef`/`tokenRef` respaldada por SecretRef para ese perfil se rechaza.
- Las infracciones son fallos graves en las rutas de inicio/recarga de resolución de autenticación.

## Mensajería compatible con versiones heredadas

Para compatibilidad con scripts, los errores de sondeo mantienen esta primera línea sin cambios:

`Auth profile credentials are missing or expired.`

Se pueden agregar detalles legibles para humanos y códigos estables de motivo en las líneas siguientes.
