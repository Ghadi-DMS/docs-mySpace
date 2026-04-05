---
read_when:
    - Quieres analizar PDF desde agentes
    - Necesitas los parámetros y límites exactos de la herramienta pdf
    - Estás depurando el modo PDF nativo frente al respaldo por extracción
summary: Analiza uno o más documentos PDF con compatibilidad nativa del proveedor y respaldo por extracción
title: Herramienta PDF
x-i18n:
    generated_at: "2026-04-05T12:56:19Z"
    model: gpt-5.4
    provider: openai
    source_hash: d7aaaa7107d7920e7c31f3e38ac19411706e646186acf520bc02f2c3e49c0517
    source_path: tools/pdf.md
    workflow: 15
---

# Herramienta PDF

`pdf` analiza uno o más documentos PDF y devuelve texto.

Comportamiento rápido:

- Modo nativo del proveedor para los proveedores de modelos Anthropic y Google.
- Modo de respaldo por extracción para otros proveedores (primero extrae texto y luego imágenes de páginas cuando sea necesario).
- Admite entrada única (`pdf`) o múltiple (`pdfs`), con un máximo de 10 PDF por llamada.

## Disponibilidad

La herramienta solo se registra cuando OpenClaw puede resolver una configuración de modelo con capacidad para PDF para el agente:

1. `agents.defaults.pdfModel`
2. respaldo a `agents.defaults.imageModel`
3. respaldo al modelo resuelto de la sesión/predeterminado del agente
4. si los proveedores de PDF nativo usan autenticación respaldada, se prefieren por delante de los candidatos genéricos de respaldo de imagen

Si no se puede resolver ningún modelo utilizable, la herramienta `pdf` no se expone.

Notas sobre disponibilidad:

- La cadena de respaldo tiene en cuenta la autenticación. Un `provider/model` configurado solo cuenta si
  OpenClaw realmente puede autenticar ese proveedor para el agente.
- Los proveedores de PDF nativo actualmente son **Anthropic** y **Google**.
- Si el proveedor resuelto de la sesión/predeterminado ya tiene configurado un modelo de visión/PDF,
  la herramienta PDF lo reutiliza antes de recurrir a otros
  proveedores respaldados por autenticación.

## Referencia de entrada

- `pdf` (`string`): una ruta o URL de PDF
- `pdfs` (`string[]`): varias rutas o URL de PDF, hasta 10 en total
- `prompt` (`string`): prompt de análisis, predeterminado `Analyze this PDF document.`
- `pages` (`string`): filtro de páginas como `1-5` o `1,3,7-9`
- `model` (`string`): anulación opcional del modelo (`provider/model`)
- `maxBytesMb` (`number`): límite de tamaño por PDF en MB

Notas sobre la entrada:

- `pdf` y `pdfs` se fusionan y desduplican antes de cargarse.
- Si no se proporciona ninguna entrada PDF, la herramienta devuelve un error.
- `pages` se analiza como números de página con base 1, se desduplica, se ordena y se ajusta al máximo de páginas configurado.
- `maxBytesMb` usa como valor predeterminado `agents.defaults.pdfMaxBytesMb` o `10`.

## Referencias PDF admitidas

- ruta de archivo local (incluida la expansión de `~`)
- URL `file://`
- URL `http://` y `https://`

Notas sobre referencias:

- Otros esquemas URI (por ejemplo, `ftp://`) se rechazan con `unsupported_pdf_reference`.
- En modo sandbox, las URL remotas `http(s)` se rechazan.
- Con la política de archivos solo del espacio de trabajo habilitada, se rechazan las rutas de archivos locales fuera de las raíces permitidas.

## Modos de ejecución

### Modo nativo del proveedor

El modo nativo se usa para los proveedores `anthropic` y `google`.
La herramienta envía bytes PDF sin procesar directamente a las API del proveedor.

Límites del modo nativo:

- `pages` no es compatible. Si se establece, la herramienta devuelve un error.
- Se admite entrada de varios PDF; cada PDF se envía como un bloque de documento nativo /
  parte PDF en línea antes del prompt.

### Modo de respaldo por extracción

El modo de respaldo se usa para proveedores no nativos.

Flujo:

1. Extrae texto de las páginas seleccionadas (hasta `agents.defaults.pdfMaxPages`, valor predeterminado `20`).
2. Si la longitud del texto extraído es inferior a `200` caracteres, renderiza las páginas seleccionadas como imágenes PNG y las incluye.
3. Envía el contenido extraído más el prompt al modelo seleccionado.

Detalles del respaldo:

- La extracción de imágenes de páginas usa un presupuesto de píxeles de `4,000,000`.
- Si el modelo de destino no admite entrada de imágenes y no hay texto extraíble, la herramienta devuelve un error.
- Si la extracción de texto funciona pero la extracción de imágenes requeriría visión en un
  modelo solo de texto, OpenClaw elimina las imágenes renderizadas y continúa con el
  texto extraído.
- El respaldo por extracción requiere `pdfjs-dist` (y `@napi-rs/canvas` para el renderizado de imágenes).

## Configuración

```json5
{
  agents: {
    defaults: {
      pdfModel: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["openai/gpt-5.4-mini"],
      },
      pdfMaxBytesMb: 10,
      pdfMaxPages: 20,
    },
  },
}
```

Consulta [Referencia de configuración](/es/gateway/configuration-reference) para ver todos los detalles de los campos.

## Detalles de salida

La herramienta devuelve texto en `content[0].text` y metadatos estructurados en `details`.

Campos comunes de `details`:

- `model`: referencia del modelo resuelto (`provider/model`)
- `native`: `true` para modo nativo del proveedor, `false` para respaldo
- `attempts`: intentos de respaldo que fallaron antes del éxito

Campos de ruta:

- entrada de un solo PDF: `details.pdf`
- entrada de varios PDF: `details.pdfs[]` con entradas `pdf`
- metadatos de reescritura de ruta de sandbox (cuando corresponda): `rewrittenFrom`

## Comportamiento de error

- Falta entrada PDF: lanza `pdf required: provide a path or URL to a PDF document`
- Demasiados PDF: devuelve error estructurado en `details.error = "too_many_pdfs"`
- Esquema de referencia no admitido: devuelve `details.error = "unsupported_pdf_reference"`
- Modo nativo con `pages`: lanza un error claro `pages is not supported with native PDF providers`

## Ejemplos

PDF único:

```json
{
  "pdf": "/tmp/report.pdf",
  "prompt": "Summarize this report in 5 bullets"
}
```

Varios PDF:

```json
{
  "pdfs": ["/tmp/q1.pdf", "/tmp/q2.pdf"],
  "prompt": "Compare risks and timeline changes across both documents"
}
```

Modelo de respaldo con filtro de páginas:

```json
{
  "pdf": "https://example.com/report.pdf",
  "pages": "1-3,7",
  "model": "openai/gpt-5.4-mini",
  "prompt": "Extract only customer-impacting incidents"
}
```

## Relacionado

- [Resumen de herramientas](/tools) — todas las herramientas de agente disponibles
- [Referencia de configuración](/es/gateway/configuration-reference#agent-defaults) — configuración de pdfMaxBytesMb y pdfMaxPages
