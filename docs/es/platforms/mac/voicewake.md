---
read_when:
    - Trabajar en rutas de activación por voz o PTT
summary: Modos de activación por voz y pulsar para hablar, además de detalles de enrutamiento en la app de Mac
title: Activación por voz (macOS)
x-i18n:
    generated_at: "2026-04-05T12:48:47Z"
    model: gpt-5.4
    provider: openai
    source_hash: fed6524a2e1fad5373d34821c920b955a2b5a3fcd9c51cdb97cf4050536602a7
    source_path: platforms/mac/voicewake.md
    workflow: 15
---

# Activación por voz y pulsar para hablar

## Modos

- **Modo de palabra de activación** (predeterminado): el reconocedor de voz siempre activo espera tokens de activación (`swabbleTriggerWords`). Cuando hay coincidencia, inicia la captura, muestra la superposición con texto parcial y envía automáticamente tras el silencio.
- **Pulsar para hablar (mantener pulsada la tecla Opción derecha)**: mantén pulsada la tecla Opción derecha para capturar inmediatamente, sin necesidad de activación. La superposición aparece mientras se mantiene pulsada; al soltarla, se finaliza y se reenvía tras un breve retraso para que puedas ajustar el texto.

## Comportamiento en runtime (palabra de activación)

- El reconocedor de voz reside en `VoiceWakeRuntime`.
- La activación solo se dispara cuando hay una **pausa significativa** entre la palabra de activación y la palabra siguiente (intervalo de ~0,55 s). La superposición/el sonido puede comenzar en la pausa incluso antes de que empiece el comando.
- Ventanas de silencio: 2,0 s cuando el habla fluye, 5,0 s si solo se oyó la activación.
- Detención forzada: 120 s para evitar sesiones descontroladas.
- Antirrebote entre sesiones: 350 ms.
- La superposición se controla mediante `VoiceWakeOverlayController` con coloreado committed/volatile.
- Después del envío, el reconocedor se reinicia limpiamente para escuchar la siguiente activación.

## Invariantes del ciclo de vida

- Si Voice Wake está habilitado y se han concedido permisos, el reconocedor de palabra de activación debería estar escuchando (excepto durante una captura explícita de pulsar para hablar).
- La visibilidad de la superposición (incluida la ocultación manual con el botón X) nunca debe impedir que el reconocedor reanude la escucha.

## Modo de fallo de superposición atascada (anterior)

Anteriormente, si la superposición quedaba visible y atascada y la cerrabas manualmente, Voice Wake podía parecer “muerto” porque el intento de reinicio del runtime podía quedar bloqueado por la visibilidad de la superposición y no se programaba ningún reinicio posterior.

Endurecimiento:

- El reinicio del runtime de activación ya no queda bloqueado por la visibilidad de la superposición.
- La finalización del cierre de la superposición activa un `VoiceWakeRuntime.refresh(...)` mediante `VoiceSessionCoordinator`, así que cerrar manualmente con la X siempre reanuda la escucha.

## Detalles específicos de pulsar para hablar

- La detección de la tecla rápida usa un monitor global `.flagsChanged` para la **tecla Opción derecha** (`keyCode 61` + `.option`). Solo observamos los eventos (sin interceptarlos).
- La canalización de captura reside en `VoicePushToTalk`: inicia Speech inmediatamente, transmite parciales a la superposición y llama a `VoiceWakeForwarder` al soltar la tecla.
- Cuando empieza pulsar para hablar, pausamos el runtime de palabra de activación para evitar taps de audio en conflicto; se reinicia automáticamente después de soltar la tecla.
- Permisos: requiere Micrófono + Speech; para ver eventos se necesita aprobación de Accessibility/Input Monitoring.
- Teclados externos: algunos pueden no exponer la tecla Opción derecha como se espera; ofrece un atajo alternativo si los usuarios informan fallos.

## Ajustes visibles para el usuario

- Interruptor **Voice Wake**: habilita el runtime de palabra de activación.
- **Mantener Cmd+Fn para hablar**: habilita el monitor de pulsar para hablar. Deshabilitado en macOS < 26.
- Selectores de idioma y micrófono, medidor de nivel en vivo, tabla de palabras de activación, probador (solo local; no reenvía).
- El selector de micrófono conserva la última selección si un dispositivo se desconecta, muestra una pista de desconexión y recurre temporalmente al valor predeterminado del sistema hasta que vuelva.
- **Sounds**: sonidos al detectar la activación y al enviar; el valor predeterminado es el sonido del sistema “Glass” de macOS. Puedes elegir cualquier archivo cargable por `NSSound` (por ejemplo MP3/WAV/AIFF) para cada evento o elegir **No Sound**.

## Comportamiento de reenvío

- Cuando Voice Wake está habilitado, las transcripciones se reenvían al gateway/agente activo (el mismo modo local frente a remoto que usa el resto de la app de Mac).
- Las respuestas se entregan al **último proveedor principal usado** (WhatsApp/Telegram/Discord/WebChat). Si la entrega falla, el error se registra y la ejecución sigue siendo visible mediante WebChat/registros de sesión.

## Carga útil de reenvío

- `VoiceWakeForwarder.prefixedTranscript(_:)` antepone la pista de la máquina antes de enviar. Compartido entre las rutas de palabra de activación y pulsar para hablar.

## Verificación rápida

- Activa pulsar para hablar, mantén pulsadas Cmd+Fn, habla y suéltalas: la superposición debería mostrar parciales y luego enviar.
- Mientras mantienes pulsadas las teclas, las orejas de la barra de menús deberían permanecer agrandadas (usa `triggerVoiceEars(ttl:nil)`); vuelven a bajar al soltarlas.
