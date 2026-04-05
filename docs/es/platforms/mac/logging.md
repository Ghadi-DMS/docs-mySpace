---
read_when:
    - Capturando logs de macOS o investigando el registro de datos privados
    - Depurando problemas del ciclo de vida de voice wake/sesión
summary: 'Logging de OpenClaw: archivo de diagnóstico rotativo + indicadores de privacidad del log unificado'
title: Logging en macOS
x-i18n:
    generated_at: "2026-04-05T12:48:21Z"
    model: gpt-5.4
    provider: openai
    source_hash: c08d6bc012f8e8bb53353fe654713dede676b4e6127e49fd76e00c2510b9ab0b
    source_path: platforms/mac/logging.md
    workflow: 15
---

# Logging (macOS)

## Archivo de log de diagnóstico rotativo (panel Debug)

OpenClaw enruta los logs de la app de macOS mediante swift-log (logging unificado por defecto) y puede escribir un archivo de log local rotativo en disco cuando necesitas una captura persistente.

- Nivel de detalle: **Debug pane → Logs → App logging → Verbosity**
- Habilitar: **Debug pane → Logs → App logging → “Write rolling diagnostics log (JSONL)”**
- Ubicación: `~/Library/Logs/OpenClaw/diagnostics.jsonl` (rota automáticamente; los archivos antiguos llevan los sufijos `.1`, `.2`, …)
- Borrar: **Debug pane → Logs → App logging → “Clear”**

Notas:

- Esto está **desactivado por defecto**. Habilítalo solo mientras estés depurando activamente.
- Trata el archivo como sensible; no lo compartas sin revisarlo.

## Datos privados del logging unificado en macOS

El logging unificado redacta la mayoría de las cargas útiles salvo que un subsistema active `privacy -off`. Según la explicación de Peter sobre las [logging privacy shenanigans](https://steipete.me/posts/2025/logging-privacy-shenanigans) de macOS (2025), esto se controla con un plist en `/Library/Preferences/Logging/Subsystems/` indexado por el nombre del subsistema. Solo las nuevas entradas de log recogen este indicador, así que actívalo antes de reproducir un problema.

## Habilitar para OpenClaw (`ai.openclaw`)

- Escribe primero el plist en un archivo temporal y luego instálalo de forma atómica como root:

```bash
cat <<'EOF' >/tmp/ai.openclaw.plist
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>DEFAULT-OPTIONS</key>
    <dict>
        <key>Enable-Private-Data</key>
        <true/>
    </dict>
</dict>
</plist>
EOF
sudo install -m 644 -o root -g wheel /tmp/ai.openclaw.plist /Library/Preferences/Logging/Subsystems/ai.openclaw.plist
```

- No se requiere reinicio; `logd` detecta el archivo rápidamente, pero solo las nuevas líneas de log incluirán cargas útiles privadas.
- Consulta la salida más completa con el helper existente, por ejemplo `./scripts/clawlog.sh --category WebChat --last 5m`.

## Deshabilitar después de depurar

- Elimina la sobrescritura: `sudo rm /Library/Preferences/Logging/Subsystems/ai.openclaw.plist`.
- Opcionalmente ejecuta `sudo log config --reload` para forzar que `logd` descarte la sobrescritura de inmediato.
- Recuerda que esta superficie puede incluir números de teléfono y cuerpos de mensajes; mantén el plist activo solo mientras necesites realmente ese nivel adicional de detalle.
