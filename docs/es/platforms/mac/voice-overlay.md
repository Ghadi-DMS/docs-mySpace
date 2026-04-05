---
read_when:
    - Ajustar el comportamiento de la superposición de voz
summary: Ciclo de vida de la superposición de voz cuando se solapan la palabra de activación y push-to-talk
title: Superposición de voz
x-i18n:
    generated_at: "2026-04-05T12:48:43Z"
    model: gpt-5.4
    provider: openai
    source_hash: 1efcc26ec05d2f421cb2cf462077d002381995b338d00db77d5fdba9b8d938b6
    source_path: platforms/mac/voice-overlay.md
    workflow: 15
---

# Ciclo de vida de la superposición de voz (macOS)

Público: colaboradores de la app para macOS. Objetivo: mantener predecible la superposición de voz cuando se solapan la palabra de activación y push-to-talk.

## Intención actual

- Si la superposición ya está visible por la palabra de activación y el usuario pulsa la tecla rápida, la sesión de tecla rápida _adopta_ el texto existente en lugar de restablecerlo. La superposición permanece visible mientras se mantenga pulsada la tecla. Cuando el usuario la suelta: enviar si hay texto con espacios recortados; de lo contrario, descartar.
- La palabra de activación por sí sola sigue enviándose automáticamente al detectar silencio; push-to-talk envía inmediatamente al soltar.

## Implementado (9 de diciembre de 2025)

- Las sesiones de superposición ahora llevan un token por captura (palabra de activación o push-to-talk). Las actualizaciones parciales/finales/de envío/de descarte/de nivel se descartan cuando el token no coincide, evitando callbacks obsoletos.
- Push-to-talk adopta cualquier texto de superposición visible como prefijo (así, al pulsar la tecla rápida mientras la superposición de activación está visible se conserva el texto y se añade el nuevo habla). Espera hasta 1,5 s por una transcripción final antes de recurrir al texto actual.
- El registro de chime/superposición se emite en `info` en las categorías `voicewake.overlay`, `voicewake.ptt` y `voicewake.chime` (inicio de sesión, parcial, final, envío, descarte, motivo del chime).

## Próximos pasos

1. **VoiceSessionCoordinator (actor)**
   - Mantiene exactamente una `VoiceSession` a la vez.
   - API (basada en tokens): `beginWakeCapture`, `beginPushToTalk`, `updatePartial`, `endCapture`, `cancel`, `applyCooldown`.
   - Descarta callbacks que lleven tokens obsoletos (evita que reconocedores antiguos vuelvan a abrir la superposición).
2. **VoiceSession (modelo)**
   - Campos: `token`, `source` (wakeWord|pushToTalk), texto confirmado/volátil, indicadores de chime, temporizadores (auto-send, idle), `overlayMode` (display|editing|sending), fecha límite de cooldown.
3. **Vinculación de superposición**
   - `VoiceSessionPublisher` (`ObservableObject`) refleja la sesión activa en SwiftUI.
   - `VoiceWakeOverlayView` renderiza solo mediante el publisher; nunca muta directamente singletons globales.
   - Las acciones del usuario sobre la superposición (`sendNow`, `dismiss`, `edit`) llaman de vuelta al coordinador con el token de sesión.
4. **Ruta de envío unificada**
   - En `endCapture`: si el texto recortado está vacío → descartar; si no, `performSend(session:)` (reproduce el chime de envío una vez, reenvía, descarta).
   - Push-to-talk: sin retraso; palabra de activación: retraso opcional para envío automático.
   - Aplica un cooldown corto al runtime de activación después de terminar push-to-talk para que la palabra de activación no se vuelva a disparar inmediatamente.
5. **Registro**
   - El coordinador emite registros `.info` en el subsistema `ai.openclaw`, categorías `voicewake.overlay` y `voicewake.chime`.
   - Eventos clave: `session_started`, `adopted_by_push_to_talk`, `partial`, `finalized`, `send`, `dismiss`, `cancel`, `cooldown`.

## Lista de comprobación de depuración

- Transmite los registros mientras reproduces una superposición bloqueada:

  ```bash
  sudo log stream --predicate 'subsystem == "ai.openclaw" AND category CONTAINS "voicewake"' --level info --style compact
  ```

- Verifica que solo haya un token de sesión activo; el coordinador debe descartar los callbacks obsoletos.
- Asegúrate de que al soltar push-to-talk siempre se llame a `endCapture` con el token activo; si el texto está vacío, espera `dismiss` sin chime ni envío.

## Pasos de migración (sugeridos)

1. Agregar `VoiceSessionCoordinator`, `VoiceSession` y `VoiceSessionPublisher`.
2. Refactorizar `VoiceWakeRuntime` para crear/actualizar/finalizar sesiones en lugar de tocar `VoiceWakeOverlayController` directamente.
3. Refactorizar `VoicePushToTalk` para adoptar sesiones existentes y llamar a `endCapture` al soltar; aplicar cooldown al runtime.
4. Conectar `VoiceWakeOverlayController` al publisher; eliminar llamadas directas desde runtime/PTT.
5. Agregar pruebas de integración para adopción de sesiones, cooldown y descarte con texto vacío.
