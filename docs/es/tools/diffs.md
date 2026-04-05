---
read_when:
    - Quieres que los agentes muestren ediciones de código o Markdown como diferencias
    - Quieres una URL de visor lista para canvas o un archivo de diferencias renderizado
    - Necesitas artefactos de diferencias temporales y controlados con valores predeterminados seguros
summary: Visor de diferencias de solo lectura y renderizador de archivos para agentes (herramienta opcional de plugin)
title: Diferencias
x-i18n:
    generated_at: "2026-04-05T12:55:41Z"
    model: gpt-5.4
    provider: openai
    source_hash: 935539a6e584980eb7e57067c18112bb40a0be8522b9da649c7cf7f180fb45d4
    source_path: tools/diffs.md
    workflow: 15
---

# Diferencias

`diffs` es una herramienta opcional de plugin con una guía breve integrada en el sistema y una skill complementaria que convierte contenido de cambios en un artefacto de diferencias de solo lectura para agentes.

Acepta una de estas opciones:

- texto `before` y `after`
- un `patch` unificado

Puede devolver:

- una URL de visor del gateway para presentación en canvas
- una ruta de archivo renderizado (PNG o PDF) para entrega por mensaje
- ambas salidas en una sola llamada

Cuando está habilitado, el plugin antepone una guía de uso concisa en el espacio del system prompt y también expone una skill detallada para los casos en los que el agente necesita instrucciones más completas.

## Inicio rápido

1. Habilita el plugin.
2. Llama a `diffs` con `mode: "view"` para flujos centrados primero en canvas.
3. Llama a `diffs` con `mode: "file"` para flujos de entrega de archivos en chat.
4. Llama a `diffs` con `mode: "both"` cuando necesites ambos artefactos.

## Habilitar el plugin

```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
      },
    },
  },
}
```

## Deshabilitar la guía integrada del sistema

Si quieres mantener habilitada la herramienta `diffs` pero deshabilitar su guía integrada del system prompt, establece `plugins.entries.diffs.hooks.allowPromptInjection` en `false`:

```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
        hooks: {
          allowPromptInjection: false,
        },
      },
    },
  },
}
```

Esto bloquea el hook `before_prompt_build` del plugin diffs mientras mantiene disponibles el plugin, la herramienta y la skill complementaria.

Si quieres deshabilitar tanto la guía como la herramienta, deshabilita el plugin en su lugar.

## Flujo de trabajo típico del agente

1. El agente llama a `diffs`.
2. El agente lee los campos `details`.
3. El agente:
   - abre `details.viewerUrl` con `canvas present`
   - envía `details.filePath` con `message` usando `path` o `filePath`
   - hace ambas cosas

## Ejemplos de entrada

Antes y después:

```json
{
  "before": "# Hello\n\nOne",
  "after": "# Hello\n\nTwo",
  "path": "docs/example.md",
  "mode": "view"
}
```

Patch:

```json
{
  "patch": "diff --git a/src/example.ts b/src/example.ts\n--- a/src/example.ts\n+++ b/src/example.ts\n@@ -1 +1 @@\n-const x = 1;\n+const x = 2;\n",
  "mode": "both"
}
```

## Referencia de entrada de la herramienta

Todos los campos son opcionales salvo que se indique lo contrario:

- `before` (`string`): texto original. Obligatorio junto con `after` cuando se omite `patch`.
- `after` (`string`): texto actualizado. Obligatorio junto con `before` cuando se omite `patch`.
- `patch` (`string`): texto de diff unificado. Mutuamente excluyente con `before` y `after`.
- `path` (`string`): nombre de archivo para mostrar en el modo antes y después.
- `lang` (`string`): sugerencia de anulación del idioma para el modo antes y después. Los valores desconocidos vuelven a texto sin formato.
- `title` (`string`): anulación del título del visor.
- `mode` (`"view" | "file" | "both"`): modo de salida. Usa de forma predeterminada el valor del plugin `defaults.mode`.
  Alias obsoleto: `"image"` se comporta como `"file"` y sigue aceptándose por compatibilidad con versiones anteriores.
- `theme` (`"light" | "dark"`): tema del visor. Usa de forma predeterminada el valor del plugin `defaults.theme`.
- `layout` (`"unified" | "split"`): disposición del diff. Usa de forma predeterminada el valor del plugin `defaults.layout`.
- `expandUnchanged` (`boolean`): expande las secciones sin cambios cuando hay contexto completo disponible. Opción solo por llamada (no es una clave predeterminada del plugin).
- `fileFormat` (`"png" | "pdf"`): formato del archivo renderizado. Usa de forma predeterminada el valor del plugin `defaults.fileFormat`.
- `fileQuality` (`"standard" | "hq" | "print"`): ajuste de calidad para renderizado PNG o PDF.
- `fileScale` (`number`): anulación de escala del dispositivo (`1`-`4`).
- `fileMaxWidth` (`number`): ancho máximo de renderizado en píxeles CSS (`640`-`2400`).
- `ttlSeconds` (`number`): TTL del artefacto en segundos para el visor y las salidas de archivo independientes. Predeterminado 1800, máximo 21600.
- `baseUrl` (`string`): anulación del origen de la URL del visor. Anula el `viewerBaseUrl` del plugin. Debe ser `http` o `https`, sin query/hash.

Los alias de entrada heredados todavía se aceptan por compatibilidad con versiones anteriores:

- `format` -> `fileFormat`
- `imageFormat` -> `fileFormat`
- `imageQuality` -> `fileQuality`
- `imageScale` -> `fileScale`
- `imageMaxWidth` -> `fileMaxWidth`

Validación y límites:

- `before` y `after` tienen un máximo de 512 KiB cada uno.
- `patch` tiene un máximo de 2 MiB.
- `path` tiene un máximo de 2048 bytes.
- `lang` tiene un máximo de 128 bytes.
- `title` tiene un máximo de 1024 bytes.
- Límite de complejidad del patch: máximo 128 archivos y 120000 líneas totales.
- Se rechaza usar `patch` junto con `before` o `after`.
- Límites de seguridad del archivo renderizado (se aplican a PNG y PDF):
  - `fileQuality: "standard"`: máximo 8 MP (8,000,000 píxeles renderizados).
  - `fileQuality: "hq"`: máximo 14 MP (14,000,000 píxeles renderizados).
  - `fileQuality: "print"`: máximo 24 MP (24,000,000 píxeles renderizados).
  - PDF también tiene un máximo de 50 páginas.

## Contrato de detalles de salida

La herramienta devuelve metadatos estructurados en `details`.

Campos compartidos para los modos que crean un visor:

- `artifactId`
- `viewerUrl`
- `viewerPath`
- `title`
- `expiresAt`
- `inputKind`
- `fileCount`
- `mode`
- `context` (`agentId`, `sessionId`, `messageChannel`, `agentAccountId` cuando están disponibles)

Campos de archivo cuando se renderiza PNG o PDF:

- `artifactId`
- `expiresAt`
- `filePath`
- `path` (mismo valor que `filePath`, para compatibilidad con la herramienta de mensajes)
- `fileBytes`
- `fileFormat`
- `fileQuality`
- `fileScale`
- `fileMaxWidth`

También se devuelven alias de compatibilidad para llamadores existentes:

- `format` (mismo valor que `fileFormat`)
- `imagePath` (mismo valor que `filePath`)
- `imageBytes` (mismo valor que `fileBytes`)
- `imageQuality` (mismo valor que `fileQuality`)
- `imageScale` (mismo valor que `fileScale`)
- `imageMaxWidth` (mismo valor que `fileMaxWidth`)

Resumen del comportamiento por modo:

- `mode: "view"`: solo campos del visor.
- `mode: "file"`: solo campos de archivo, sin artefacto de visor.
- `mode: "both"`: campos del visor más campos de archivo. Si falla el renderizado del archivo, el visor igualmente se devuelve con `fileError` y el alias de compatibilidad `imageError`.

## Secciones sin cambios colapsadas

- El visor puede mostrar filas como `N unmodified lines`.
- Los controles de expansión en esas filas son condicionales y no están garantizados para cada tipo de entrada.
- Los controles de expansión aparecen cuando el diff renderizado tiene datos de contexto expandibles, algo típico en entradas antes y después.
- Para muchas entradas de patch unificado, los cuerpos de contexto omitidos no están disponibles en los bloques parseados del patch, por lo que la fila puede aparecer sin controles de expansión. Este es el comportamiento esperado.
- `expandUnchanged` se aplica solo cuando existe contexto expandible.

## Valores predeterminados del plugin

Establece valores predeterminados globales del plugin en `~/.openclaw/openclaw.json`:

```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
        config: {
          defaults: {
            fontFamily: "Fira Code",
            fontSize: 15,
            lineSpacing: 1.6,
            layout: "unified",
            showLineNumbers: true,
            diffIndicators: "bars",
            wordWrap: true,
            background: true,
            theme: "dark",
            fileFormat: "png",
            fileQuality: "standard",
            fileScale: 2,
            fileMaxWidth: 960,
            mode: "both",
          },
        },
      },
    },
  },
}
```

Valores predeterminados admitidos:

- `fontFamily`
- `fontSize`
- `lineSpacing`
- `layout`
- `showLineNumbers`
- `diffIndicators`
- `wordWrap`
- `background`
- `theme`
- `fileFormat`
- `fileQuality`
- `fileScale`
- `fileMaxWidth`
- `mode`

Los parámetros explícitos de la herramienta anulan estos valores predeterminados.

Configuración persistente de URL del visor:

- `viewerBaseUrl` (`string`, opcional)
  - Valor de respaldo propiedad del plugin para los enlaces del visor devueltos cuando una llamada a la herramienta no pasa `baseUrl`.
  - Debe ser `http` o `https`, sin query/hash.

Ejemplo:

```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
        config: {
          viewerBaseUrl: "https://gateway.example.com/openclaw",
        },
      },
    },
  },
}
```

## Configuración de seguridad

- `security.allowRemoteViewer` (`boolean`, predeterminado `false`)
  - `false`: se deniegan las solicitudes que no son loopback a las rutas del visor.
  - `true`: se permiten visores remotos si la ruta tokenizada es válida.

Ejemplo:

```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
        config: {
          security: {
            allowRemoteViewer: false,
          },
        },
      },
    },
  },
}
```

## Ciclo de vida y almacenamiento de artefactos

- Los artefactos se almacenan en la subcarpeta temporal: `$TMPDIR/openclaw-diffs`.
- Los metadatos del artefacto del visor contienen:
  - ID de artefacto aleatorio (20 caracteres hexadecimales)
  - token aleatorio (48 caracteres hexadecimales)
  - `createdAt` y `expiresAt`
  - ruta `viewer.html` almacenada
- El TTL predeterminado del artefacto es de 30 minutos cuando no se especifica.
- El TTL máximo aceptado del visor es de 6 horas.
- La limpieza se ejecuta de forma oportunista después de crear el artefacto.
- Los artefactos vencidos se eliminan.
- La limpieza de respaldo elimina carpetas obsoletas de más de 24 horas cuando faltan los metadatos.

## URL del visor y comportamiento de red

Ruta del visor:

- `/plugins/diffs/view/{artifactId}/{token}`

Recursos del visor:

- `/plugins/diffs/assets/viewer.js`
- `/plugins/diffs/assets/viewer-runtime.js`

El documento del visor resuelve esos recursos relativos a la URL del visor, por lo que un prefijo de ruta opcional en `baseUrl` también se conserva para ambas solicitudes de recursos.

Comportamiento de construcción de URL:

- Si se proporciona `baseUrl` en la llamada a la herramienta, se usa después de una validación estricta.
- En caso contrario, si el plugin tiene configurado `viewerBaseUrl`, se usa ese valor.
- Sin ninguna de esas anulaciones, la URL del visor usa de forma predeterminada loopback `127.0.0.1`.
- Si el modo de enlace del gateway es `custom` y `gateway.customBindHost` está configurado, se usa ese host.

Reglas de `baseUrl`:

- Debe ser `http://` o `https://`.
- Se rechazan query y hash.
- Se permite el origen más una ruta base opcional.

## Modelo de seguridad

Fortalecimiento del visor:

- Solo loopback de forma predeterminada.
- Rutas de visor tokenizadas con validación estricta de ID y token.
- CSP de respuesta del visor:
  - `default-src 'none'`
  - scripts y recursos solo desde self
  - sin `connect-src` saliente
- Limitación de fallos remotos cuando el acceso remoto está habilitado:
  - 40 fallos por 60 segundos
  - bloqueo de 60 segundos (`429 Too Many Requests`)

Fortalecimiento del renderizado de archivos:

- El enrutamiento de solicitudes del navegador de capturas está denegado por defecto.
- Solo se permiten recursos locales del visor desde `http://127.0.0.1/plugins/diffs/assets/*`.
- Se bloquean las solicitudes de red externas.

## Requisitos del navegador para el modo de archivo

`mode: "file"` y `mode: "both"` necesitan un navegador compatible con Chromium.

Orden de resolución:

1. `browser.executablePath` en la configuración de OpenClaw.
2. Variables de entorno:
   - `OPENCLAW_BROWSER_EXECUTABLE_PATH`
   - `BROWSER_EXECUTABLE_PATH`
   - `PLAYWRIGHT_CHROMIUM_EXECUTABLE_PATH`
3. Respaldo de detección de comandos/rutas de la plataforma.

Texto de fallo común:

- `Diff PNG/PDF rendering requires a Chromium-compatible browser...`

Corrígelo instalando Chrome, Chromium, Edge o Brave, o configurando una de las opciones de ruta del ejecutable anteriores.

## Solución de problemas

Errores de validación de entrada:

- `Provide patch or both before and after text.`
  - Incluye `before` y `after`, o proporciona `patch`.
- `Provide either patch or before/after input, not both.`
  - No mezcles modos de entrada.
- `Invalid baseUrl: ...`
  - Usa un origen `http(s)` con ruta opcional, sin query/hash.
- `{field} exceeds maximum size (...)`
  - Reduce el tamaño de la carga útil.
- Rechazo de patch grande
  - Reduce el número de archivos del patch o el total de líneas.

Problemas de accesibilidad del visor:

- La URL del visor se resuelve a `127.0.0.1` de forma predeterminada.
- Para escenarios de acceso remoto:
  - configura `viewerBaseUrl` en el plugin, o
  - pasa `baseUrl` por llamada a la herramienta, o
  - usa `gateway.bind=custom` y `gateway.customBindHost`
- Si `gateway.trustedProxies` incluye loopback para un proxy del mismo host (por ejemplo Tailscale Serve), las solicitudes sin procesar del visor en loopback sin encabezados de IP de cliente reenviada fallan de forma cerrada por diseño.
- Para esa topología de proxy:
  - prefiere `mode: "file"` o `mode: "both"` cuando solo necesites un adjunto, o
  - habilita intencionalmente `security.allowRemoteViewer` y configura `viewerBaseUrl` en el plugin o pasa un `baseUrl` de proxy/público cuando necesites una URL de visor compartible
- Habilita `security.allowRemoteViewer` solo cuando realmente quieras acceso externo al visor.

La fila de líneas sin cambios no tiene botón de expansión:

- Esto puede ocurrir con entrada de patch cuando el patch no contiene contexto expandible.
- Esto es esperable y no indica un fallo del visor.

Artefacto no encontrado:

- El artefacto venció por TTL.
- El token o la ruta cambiaron.
- La limpieza eliminó datos obsoletos.

## Guía operativa

- Prefiere `mode: "view"` para revisiones interactivas locales en canvas.
- Prefiere `mode: "file"` para canales de chat salientes que necesitan un adjunto.
- Mantén `allowRemoteViewer` deshabilitado a menos que tu implementación necesite URL remotas del visor.
- Establece un `ttlSeconds` corto y explícito para diferencias sensibles.
- Evita enviar secretos en la entrada del diff cuando no sea necesario.
- Si tu canal comprime imágenes de forma agresiva (por ejemplo Telegram o WhatsApp), prefiere salida PDF (`fileFormat: "pdf"`).

Motor de renderizado de diferencias:

- Impulsado por [Diffs](https://diffs.com).

## Documentación relacionada

- [Resumen de herramientas](/tools)
- [Plugins](/tools/plugin)
- [Navegador](/tools/browser)
