---
read_when:
    - Implementar el panel Canvas de macOS
    - Añadir controles del agente para el espacio de trabajo visual
    - Depurar cargas de canvas en WKWebView
summary: Panel Canvas controlado por el agente e incrustado mediante WKWebView + esquema de URL personalizado
title: Canvas
x-i18n:
    generated_at: "2026-04-05T12:48:10Z"
    model: gpt-5.4
    provider: openai
    source_hash: b6c71763d693264d943e570a852208cce69fc469976b2a1cdd9e39e2550534c1
    source_path: platforms/mac/canvas.md
    workflow: 15
---

# Canvas (app de macOS)

La app de macOS incrusta un **panel Canvas** controlado por el agente mediante `WKWebView`. Es
un espacio de trabajo visual ligero para HTML/CSS/JS, A2UI y superficies pequeñas
de IU interactiva.

## Dónde vive Canvas

El estado de Canvas se almacena en Application Support:

- `~/Library/Application Support/OpenClaw/canvas/<session>/...`

El panel Canvas sirve esos archivos mediante un **esquema de URL personalizado**:

- `openclaw-canvas://<session>/<path>`

Ejemplos:

- `openclaw-canvas://main/` → `<canvasRoot>/main/index.html`
- `openclaw-canvas://main/assets/app.css` → `<canvasRoot>/main/assets/app.css`
- `openclaw-canvas://main/widgets/todo/` → `<canvasRoot>/main/widgets/todo/index.html`

Si no existe `index.html` en la raíz, la app muestra una **página scaffold integrada**.

## Comportamiento del panel

- Panel sin bordes, redimensionable y anclado cerca de la barra de menús (o del cursor del ratón).
- Recuerda tamaño/posición por sesión.
- Se recarga automáticamente cuando cambian los archivos locales de canvas.
- Solo un panel Canvas es visible a la vez (la sesión se cambia según sea necesario).

Canvas puede desactivarse desde Settings → **Allow Canvas**. Cuando está desactivado, los comandos
de nodo de canvas devuelven `CANVAS_DISABLED`.

## Superficie de API del agente

Canvas se expone mediante el **WebSocket del Gateway**, por lo que el agente puede:

- mostrar/ocultar el panel
- navegar a una ruta o URL
- evaluar JavaScript
- capturar una imagen de instantánea

Ejemplos de CLI:

```bash
openclaw nodes canvas present --node <id>
openclaw nodes canvas navigate --node <id> --url "/"
openclaw nodes canvas eval --node <id> --js "document.title"
openclaw nodes canvas snapshot --node <id>
```

Notas:

- `canvas.navigate` acepta **rutas locales de canvas**, URL `http(s)` y URL `file://`.
- Si pasas `"/"`, Canvas muestra el scaffold local o `index.html`.

## A2UI en Canvas

A2UI está alojado por el host canvas del Gateway y se renderiza dentro del panel Canvas.
Cuando el Gateway anuncia un host Canvas, la app de macOS navega automáticamente a la
página de host de A2UI al abrirse por primera vez.

URL predeterminada del host A2UI:

```
http://<gateway-host>:18789/__openclaw__/a2ui/
```

### Comandos A2UI (v0.8)

Canvas actualmente acepta mensajes A2UI **v0.8** de servidor a cliente:

- `beginRendering`
- `surfaceUpdate`
- `dataModelUpdate`
- `deleteSurface`

`createSurface` (v0.9) no es compatible.

Ejemplo de CLI:

```bash
cat > /tmp/a2ui-v0.8.jsonl <<'EOFA2'
{"surfaceUpdate":{"surfaceId":"main","components":[{"id":"root","component":{"Column":{"children":{"explicitList":["title","content"]}}}},{"id":"title","component":{"Text":{"text":{"literalString":"Canvas (A2UI v0.8)"},"usageHint":"h1"}}},{"id":"content","component":{"Text":{"text":{"literalString":"If you can read this, A2UI push works."},"usageHint":"body"}}}]}}
{"beginRendering":{"surfaceId":"main","root":"root"}}
EOFA2

openclaw nodes canvas a2ui push --jsonl /tmp/a2ui-v0.8.jsonl --node <id>
```

Prueba rápida:

```bash
openclaw nodes canvas a2ui push --node <id> --text "Hello from A2UI"
```

## Activar ejecuciones del agente desde Canvas

Canvas puede activar nuevas ejecuciones del agente mediante enlaces profundos:

- `openclaw://agent?...`

Ejemplo (en JS):

```js
window.location.href = "openclaw://agent?message=Review%20this%20design";
```

La app pide confirmación salvo que se proporcione una clave válida.

## Notas de seguridad

- El esquema Canvas bloquea el recorrido de directorios; los archivos deben vivir bajo la raíz de la sesión.
- El contenido local de Canvas usa un esquema personalizado (no hace falta servidor loopback).
- Las URL externas `http(s)` solo se permiten cuando se navega a ellas explícitamente.
