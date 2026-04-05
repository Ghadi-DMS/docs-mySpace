---
read_when:
    - Actualizar una instalación existente de Matrix
    - Migrar historial cifrado de Matrix y estado del dispositivo
summary: Cómo OpenClaw actualiza in situ el plugin Matrix anterior, incluidos los límites de recuperación de estado cifrado y los pasos de recuperación manual.
title: Migración de Matrix
x-i18n:
    generated_at: "2026-04-05T12:46:34Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7b1ade057d90a524e09756bd981921988c980ea6259f5c4316a796a831e9f83b
    source_path: install/migrating-matrix.md
    workflow: 15
---

# Migración de Matrix

Esta página cubre las actualizaciones desde el plugin público anterior `matrix` hasta la implementación actual.

Para la mayoría de los usuarios, la actualización se realiza in situ:

- el plugin sigue siendo `@openclaw/matrix`
- el canal sigue siendo `matrix`
- tu configuración sigue en `channels.matrix`
- las credenciales en caché siguen en `~/.openclaw/credentials/matrix/`
- el estado de runtime sigue en `~/.openclaw/matrix/`

No necesitas renombrar claves de configuración ni reinstalar el plugin con un nombre nuevo.

## Qué hace automáticamente la migración

Cuando se inicia la gateway, y cuando ejecutas [`openclaw doctor --fix`](/gateway/doctor), OpenClaw intenta reparar automáticamente el estado antiguo de Matrix.
Antes de que cualquier paso accionable de migración de Matrix modifique el estado en disco, OpenClaw crea o reutiliza una instantánea de recuperación específica.

Cuando usas `openclaw update`, el desencadenante exacto depende de cómo esté instalado OpenClaw:

- las instalaciones desde el código fuente ejecutan `openclaw doctor --fix` durante el flujo de actualización y luego reinician la gateway de forma predeterminada
- las instalaciones mediante gestor de paquetes actualizan el paquete, ejecutan un paso de doctor no interactivo y luego dependen del reinicio predeterminado de la gateway para que el arranque pueda completar la migración de Matrix
- si usas `openclaw update --no-restart`, la migración de Matrix respaldada por el arranque se aplaza hasta que más tarde ejecutes `openclaw doctor --fix` y reinicies la gateway

La migración automática cubre:

- crear o reutilizar una instantánea previa a la migración en `~/Backups/openclaw-migrations/`
- reutilizar tus credenciales de Matrix en caché
- mantener la misma selección de cuenta y configuración `channels.matrix`
- mover el almacén sync plano de Matrix más antiguo a la ubicación actual delimitada por cuenta
- mover el almacén crypto plano más antiguo de Matrix a la ubicación actual delimitada por cuenta cuando la cuenta de destino puede resolverse con seguridad
- extraer una clave de descifrado de copia de seguridad de claves de sala de Matrix guardada previamente del antiguo almacén crypto en rust, cuando esa clave existe localmente
- reutilizar la raíz de almacenamiento existente más completa con hash de token para la misma cuenta de Matrix, homeserver y usuario cuando el access token cambia más adelante
- examinar raíces hermanas de almacenamiento con hash de token en busca de metadatos pendientes de restauración de estado cifrado cuando el access token de Matrix cambia pero la identidad de cuenta/dispositivo permanece igual
- restaurar claves de sala respaldadas en el nuevo almacén crypto en el siguiente arranque de Matrix

Detalles de la instantánea:

- OpenClaw escribe un archivo marcador en `~/.openclaw/matrix/migration-snapshot.json` después de una instantánea correcta para que los pasos posteriores de arranque y reparación puedan reutilizar el mismo archivo.
- Estas instantáneas automáticas de migración de Matrix respaldan solo configuración + estado (`includeWorkspace: false`).
- Si Matrix solo tiene estado de migración de tipo advertencia, por ejemplo porque `userId` o `accessToken` aún faltan, OpenClaw todavía no crea la instantánea porque ninguna mutación de Matrix es accionable.
- Si falla el paso de instantánea, OpenClaw omite la migración de Matrix en esa ejecución en lugar de modificar el estado sin un punto de recuperación.

Sobre las actualizaciones de varias cuentas:

- el almacén plano de Matrix más antiguo (`~/.openclaw/matrix/bot-storage.json` y `~/.openclaw/matrix/crypto/`) procedía de un diseño de almacén único, así que OpenClaw solo puede migrarlo a un único destino de cuenta Matrix resuelto
- los almacenes heredados de Matrix ya delimitados por cuenta se detectan y preparan por cada cuenta Matrix configurada

## Qué no puede hacer automáticamente la migración

El plugin público anterior de Matrix **no** creaba automáticamente copias de seguridad de claves de sala de Matrix. Persistía el estado crypto local y solicitaba verificación del dispositivo, pero no garantizaba que tus claves de sala se respaldaran en el homeserver.

Eso significa que algunas instalaciones cifradas solo pueden migrarse parcialmente.

OpenClaw no puede recuperar automáticamente:

- claves de sala solo locales que nunca se respaldaron
- estado cifrado cuando la cuenta Matrix de destino aún no puede resolverse porque `homeserver`, `userId` o `accessToken` todavía no están disponibles
- migración automática de un almacén plano compartido de Matrix cuando se configuran varias cuentas Matrix pero `channels.matrix.defaultAccount` no está establecido
- instalaciones de plugin desde rutas personalizadas fijadas a una ruta del repositorio en lugar del paquete estándar de Matrix
- una recovery key faltante cuando el almacén antiguo tenía claves respaldadas pero no conservaba la clave de descifrado localmente

Ámbito actual de advertencias:

- las instalaciones de Matrix desde rutas personalizadas se muestran tanto en el arranque de la gateway como en `openclaw doctor`

Si tu instalación antigua tenía historial cifrado solo local que nunca se respaldó, algunos mensajes cifrados antiguos pueden seguir siendo ilegibles después de la actualización.

## Flujo de actualización recomendado

1. Actualiza OpenClaw y el plugin de Matrix normalmente.
   Prefiere `openclaw update` sin `--no-restart` para que el arranque pueda completar inmediatamente la migración de Matrix.
2. Ejecuta:

   ```bash
   openclaw doctor --fix
   ```

   Si Matrix tiene trabajo de migración accionable, doctor creará o reutilizará primero la instantánea previa a la migración e imprimirá la ruta del archivo.

3. Inicia o reinicia la gateway.
4. Comprueba el estado actual de verificación y copia de seguridad:

   ```bash
   openclaw matrix verify status
   openclaw matrix verify backup status
   ```

5. Si OpenClaw te indica que se necesita una recovery key, ejecuta:

   ```bash
   openclaw matrix verify backup restore --recovery-key "<your-recovery-key>"
   ```

6. Si este dispositivo sigue sin verificar, ejecuta:

   ```bash
   openclaw matrix verify device "<your-recovery-key>"
   ```

7. Si intencionadamente vas a abandonar historial antiguo irrecuperable y quieres una nueva línea base de copia de seguridad para mensajes futuros, ejecuta:

   ```bash
   openclaw matrix verify backup reset --yes
   ```

8. Si todavía no existe una copia de seguridad de claves del lado del servidor, crea una para recuperaciones futuras:

   ```bash
   openclaw matrix verify bootstrap
   ```

## Cómo funciona la migración cifrada

La migración cifrada es un proceso de dos etapas:

1. El arranque o `openclaw doctor --fix` crea o reutiliza la instantánea previa a la migración si la migración cifrada es accionable.
2. El arranque o `openclaw doctor --fix` inspecciona el antiguo almacén crypto de Matrix a través de la instalación activa del plugin Matrix.
3. Si se encuentra una clave de descifrado de copia de seguridad, OpenClaw la escribe en el nuevo flujo de recovery key y marca la restauración de claves de sala como pendiente.
4. En el siguiente arranque de Matrix, OpenClaw restaura automáticamente las claves de sala respaldadas en el nuevo almacén crypto.

Si el almacén antiguo informa claves de sala que nunca se respaldaron, OpenClaw avisa en lugar de fingir que la recuperación tuvo éxito.

## Mensajes comunes y su significado

### Mensajes de actualización y detección

`Matrix plugin upgraded in place.`

- Significado: se detectó el estado antiguo de Matrix en disco y se migró al diseño actual.
- Qué hacer: nada, salvo que la misma salida incluya también advertencias.

`Matrix migration snapshot created before applying Matrix upgrades.`

- Significado: OpenClaw creó un archivo de recuperación antes de modificar el estado de Matrix.
- Qué hacer: conserva la ruta del archivo impresa hasta que confirmes que la migración se completó correctamente.

`Matrix migration snapshot reused before applying Matrix upgrades.`

- Significado: OpenClaw encontró un marcador existente de instantánea de migración de Matrix y reutilizó ese archivo en lugar de crear una copia de seguridad duplicada.
- Qué hacer: conserva la ruta del archivo impresa hasta que confirmes que la migración se completó correctamente.

`Legacy Matrix state detected at ... but channels.matrix is not configured yet.`

- Significado: existe estado antiguo de Matrix, pero OpenClaw no puede asignarlo a una cuenta Matrix actual porque Matrix no está configurado.
- Qué hacer: configura `channels.matrix`, luego vuelve a ejecutar `openclaw doctor --fix` o reinicia la gateway.

`Legacy Matrix state detected at ... but the new account-scoped target could not be resolved yet (need homeserver, userId, and access token for channels.matrix...).`

- Significado: OpenClaw encontró estado antiguo, pero todavía no puede determinar la raíz exacta de la cuenta/dispositivo actual.
- Qué hacer: inicia la gateway una vez con un inicio de sesión Matrix funcional, o vuelve a ejecutar `openclaw doctor --fix` después de que existan credenciales en caché.

`Legacy Matrix state detected at ... but multiple Matrix accounts are configured and channels.matrix.defaultAccount is not set.`

- Significado: OpenClaw encontró un almacén plano compartido de Matrix, pero se niega a adivinar qué cuenta Matrix con nombre debe recibirlo.
- Qué hacer: establece `channels.matrix.defaultAccount` con la cuenta prevista, luego vuelve a ejecutar `openclaw doctor --fix` o reinicia la gateway.

`Matrix legacy sync store not migrated because the target already exists (...)`

- Significado: la nueva ubicación delimitada por cuenta ya tiene un almacén sync o crypto, así que OpenClaw no la sobrescribió automáticamente.
- Qué hacer: verifica que la cuenta actual sea la correcta antes de eliminar o mover manualmente el destino en conflicto.

`Failed migrating Matrix legacy sync store (...)` o `Failed migrating Matrix legacy crypto store (...)`

- Significado: OpenClaw intentó mover estado antiguo de Matrix, pero la operación del sistema de archivos falló.
- Qué hacer: inspecciona permisos del sistema de archivos y estado del disco, luego vuelve a ejecutar `openclaw doctor --fix`.

`Legacy Matrix encrypted state detected at ... but channels.matrix is not configured yet.`

- Significado: OpenClaw encontró un almacén cifrado antiguo de Matrix, pero no existe una configuración actual de Matrix a la que asociarlo.
- Qué hacer: configura `channels.matrix`, luego vuelve a ejecutar `openclaw doctor --fix` o reinicia la gateway.

`Legacy Matrix encrypted state detected at ... but the account-scoped target could not be resolved yet (need homeserver, userId, and access token for channels.matrix...).`

- Significado: el almacén cifrado existe, pero OpenClaw no puede decidir con seguridad a qué cuenta/dispositivo actual pertenece.
- Qué hacer: inicia la gateway una vez con un inicio de sesión Matrix funcional, o vuelve a ejecutar `openclaw doctor --fix` después de que existan credenciales en caché.

`Legacy Matrix encrypted state detected at ... but multiple Matrix accounts are configured and channels.matrix.defaultAccount is not set.`

- Significado: OpenClaw encontró un único almacén crypto heredado plano compartido, pero se niega a adivinar qué cuenta Matrix con nombre debe recibirlo.
- Qué hacer: establece `channels.matrix.defaultAccount` con la cuenta prevista, luego vuelve a ejecutar `openclaw doctor --fix` o reinicia la gateway.

`Matrix migration warnings are present, but no on-disk Matrix mutation is actionable yet. No pre-migration snapshot was needed.`

- Significado: OpenClaw detectó estado antiguo de Matrix, pero la migración sigue bloqueada por falta de datos de identidad o credenciales.
- Qué hacer: completa el inicio de sesión o la configuración de Matrix, luego vuelve a ejecutar `openclaw doctor --fix` o reinicia la gateway.

`Legacy Matrix encrypted state was detected, but the Matrix plugin helper is unavailable. Install or repair @openclaw/matrix so OpenClaw can inspect the old rust crypto store before upgrading.`

- Significado: OpenClaw encontró estado cifrado antiguo de Matrix, pero no pudo cargar el entrypoint helper del plugin Matrix que normalmente inspecciona ese almacén.
- Qué hacer: reinstala o repara el plugin Matrix (`openclaw plugins install @openclaw/matrix`, o `openclaw plugins install ./path/to/local/matrix-plugin` para un checkout del repositorio), luego vuelve a ejecutar `openclaw doctor --fix` o reinicia la gateway.

`Matrix plugin helper path is unsafe: ... Reinstall @openclaw/matrix and try again.`

- Significado: OpenClaw encontró una ruta de archivo helper que escapa de la raíz del plugin o falla las comprobaciones de límites del plugin, así que se negó a importarla.
- Qué hacer: reinstala el plugin Matrix desde una ruta de confianza, luego vuelve a ejecutar `openclaw doctor --fix` o reinicia la gateway.

`- Failed creating a Matrix migration snapshot before repair: ...`

`- Skipping Matrix migration changes for now. Resolve the snapshot failure, then rerun "openclaw doctor --fix".`

- Significado: OpenClaw se negó a modificar el estado de Matrix porque no pudo crear primero la instantánea de recuperación.
- Qué hacer: resuelve el error de copia de seguridad, luego vuelve a ejecutar `openclaw doctor --fix` o reinicia la gateway.

`Failed migrating legacy Matrix client storage: ...`

- Significado: el fallback del lado cliente de Matrix encontró almacenamiento plano antiguo, pero el movimiento falló. OpenClaw ahora aborta ese fallback en lugar de iniciar silenciosamente con un almacén nuevo.
- Qué hacer: inspecciona permisos o conflictos del sistema de archivos, conserva intacto el estado antiguo y vuelve a intentarlo después de corregir el error.

`Matrix is installed from a custom path: ...`

- Significado: Matrix está fijado a una instalación desde ruta, así que las actualizaciones principales no lo reemplazan automáticamente con el paquete Matrix estándar del repositorio.
- Qué hacer: reinstala con `openclaw plugins install @openclaw/matrix` cuando quieras volver al plugin Matrix predeterminado.

### Mensajes de recuperación de estado cifrado

`matrix: restored X/Y room key(s) from legacy encrypted-state backup`

- Significado: las claves de sala respaldadas se restauraron correctamente en el nuevo almacén crypto.
- Qué hacer: normalmente, nada.

`matrix: N legacy local-only room key(s) were never backed up and could not be restored automatically`

- Significado: algunas claves de sala antiguas existían solo en el almacén local antiguo y nunca se habían subido a la copia de seguridad de Matrix.
- Qué hacer: espera que parte del historial cifrado antiguo siga sin estar disponible a menos que puedas recuperar esas claves manualmente desde otro cliente Matrix verificado.

`Legacy Matrix encrypted state for account "..." has backed-up room keys, but no local backup decryption key was found. Ask the operator to run "openclaw matrix verify backup restore --recovery-key <key>" after upgrade if they have the recovery key.`

- Significado: la copia de seguridad existe, pero OpenClaw no pudo recuperar automáticamente la recovery key.
- Qué hacer: ejecuta `openclaw matrix verify backup restore --recovery-key "<your-recovery-key>"`.

`Failed inspecting legacy Matrix encrypted state for account "..." (...): ...`

- Significado: OpenClaw encontró el almacén cifrado antiguo, pero no pudo inspeccionarlo con suficiente seguridad para preparar la recuperación.
- Qué hacer: vuelve a ejecutar `openclaw doctor --fix`. Si se repite, conserva intacto el directorio del estado antiguo y recupera usando otro cliente Matrix verificado junto con `openclaw matrix verify backup restore --recovery-key "<your-recovery-key>"`.

`Legacy Matrix backup key was found for account "...", but .../recovery-key.json already contains a different recovery key. Leaving the existing file unchanged.`

- Significado: OpenClaw detectó un conflicto de recovery key y se negó a sobrescribir automáticamente el archivo actual de recovery key.
- Qué hacer: verifica cuál recovery key es correcta antes de reintentar cualquier comando de restauración.

`Legacy Matrix encrypted state for account "..." cannot be fully converted automatically because the old rust crypto store does not expose all local room keys for export.`

- Significado: este es el límite duro del antiguo formato de almacenamiento.
- Qué hacer: las claves respaldadas todavía pueden restaurarse, pero el historial cifrado solo local puede seguir sin estar disponible.

`matrix: failed restoring room keys from legacy encrypted-state backup: ...`

- Significado: el nuevo plugin intentó restaurar, pero Matrix devolvió un error.
- Qué hacer: ejecuta `openclaw matrix verify backup status`, luego vuelve a intentarlo con `openclaw matrix verify backup restore --recovery-key "<your-recovery-key>"` si es necesario.

### Mensajes de recuperación manual

`Backup key is not loaded on this device. Run 'openclaw matrix verify backup restore' to load it and restore old room keys.`

- Significado: OpenClaw sabe que deberías tener una backup key, pero no está activa en este dispositivo.
- Qué hacer: ejecuta `openclaw matrix verify backup restore`, o pasa `--recovery-key` si hace falta.

`Store a recovery key with 'openclaw matrix verify device <key>', then run 'openclaw matrix verify backup restore'.`

- Significado: este dispositivo no tiene almacenada actualmente la recovery key.
- Qué hacer: primero verifica el dispositivo con tu recovery key y luego restaura la copia de seguridad.

`Backup key mismatch on this device. Re-run 'openclaw matrix verify device <key>' with the matching recovery key.`

- Significado: la clave almacenada no coincide con la copia de seguridad Matrix activa.
- Qué hacer: vuelve a ejecutar `openclaw matrix verify device "<your-recovery-key>"` con la clave correcta.

Si aceptas perder el historial cifrado antiguo irrecuperable, puedes en su lugar restablecer la
línea base actual de copia de seguridad con `openclaw matrix verify backup reset --yes`. Cuando el
secreto de copia de seguridad almacenado está roto, ese restablecimiento también puede recrear el almacenamiento de secretos para que la
nueva backup key pueda cargarse correctamente después del reinicio.

`Backup trust chain is not verified on this device. Re-run 'openclaw matrix verify device <key>'.`

- Significado: la copia de seguridad existe, pero este dispositivo todavía no confía lo suficiente en la cadena de cross-signing.
- Qué hacer: vuelve a ejecutar `openclaw matrix verify device "<your-recovery-key>"`.

`Matrix recovery key is required`

- Significado: intentaste un paso de recuperación sin proporcionar una recovery key cuando era obligatoria.
- Qué hacer: vuelve a ejecutar el comando con tu recovery key.

`Invalid Matrix recovery key: ...`

- Significado: la clave proporcionada no pudo analizarse o no coincidía con el formato esperado.
- Qué hacer: vuelve a intentarlo con la recovery key exacta de tu cliente Matrix o de tu archivo recovery-key.

`Matrix device is still unverified after applying recovery key. Verify your recovery key and ensure cross-signing is available.`

- Significado: se aplicó la clave, pero el dispositivo aún no pudo completar la verificación.
- Qué hacer: confirma que usaste la clave correcta y que cross-signing está disponible en la cuenta, luego vuelve a intentarlo.

`Matrix key backup is not active on this device after loading from secret storage.`

- Significado: el almacenamiento de secretos no produjo una sesión de copia de seguridad activa en este dispositivo.
- Qué hacer: primero verifica el dispositivo y luego vuelve a comprobar con `openclaw matrix verify backup status`.

`Matrix crypto backend cannot load backup keys from secret storage. Verify this device with 'openclaw matrix verify device <key>' first.`

- Significado: este dispositivo no puede restaurar desde el almacenamiento de secretos hasta que se complete la verificación del dispositivo.
- Qué hacer: ejecuta primero `openclaw matrix verify device "<your-recovery-key>"`.

### Mensajes de instalación de plugin personalizado

`Matrix is installed from a custom path that no longer exists: ...`

- Significado: tu registro de instalación del plugin apunta a una ruta local que ya no existe.
- Qué hacer: reinstala con `openclaw plugins install @openclaw/matrix`, o si estás ejecutando desde un checkout del repositorio, `openclaw plugins install ./path/to/local/matrix-plugin`.

## Si el historial cifrado sigue sin volver

Ejecuta estas comprobaciones en este orden:

```bash
openclaw matrix verify status --verbose
openclaw matrix verify backup status --verbose
openclaw matrix verify backup restore --recovery-key "<your-recovery-key>" --verbose
```

Si la copia de seguridad se restaura correctamente pero algunas salas antiguas siguen sin mostrar historial, probablemente esas claves que faltan nunca fueron respaldadas por el plugin anterior.

## Si quieres empezar desde cero para mensajes futuros

Si aceptas perder el historial cifrado antiguo irrecuperable y solo quieres una línea base limpia de copia de seguridad a partir de ahora, ejecuta estos comandos en orden:

```bash
openclaw matrix verify backup reset --yes
openclaw matrix verify backup status --verbose
openclaw matrix verify status
```

Si el dispositivo sigue sin verificarse después de eso, completa la verificación desde tu cliente Matrix comparando los emoji SAS o los códigos decimales y confirmando que coinciden.

## Páginas relacionadas

- [Matrix](/channels/matrix)
- [Doctor](/gateway/doctor)
- [Migrating](/install/migrating)
- [Plugins](/tools/plugin)
