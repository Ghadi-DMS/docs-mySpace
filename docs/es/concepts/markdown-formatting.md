---
read_when:
    - Estás cambiando el formato Markdown o la fragmentación para canales salientes
    - Estás agregando un nuevo formateador de canal o un mapeo de estilos
    - Estás depurando regresiones de formato entre canales
summary: Canalización de formato Markdown para canales salientes
title: Formato Markdown
x-i18n:
    generated_at: "2026-04-05T12:39:42Z"
    model: gpt-5.4
    provider: openai
    source_hash: f3794674e30e265208d14a986ba9bdc4ba52e0cb69c446094f95ca6c674e4566
    source_path: concepts/markdown-formatting.md
    workflow: 15
---

# Formato Markdown

OpenClaw formatea el Markdown saliente convirtiéndolo en una representación intermedia compartida
(IR) antes de renderizar la salida específica de cada canal. La IR mantiene intacto el
texto fuente mientras transporta rangos de estilo/enlace para que la fragmentación y el renderizado puedan
seguir siendo consistentes entre canales.

## Objetivos

- **Consistencia:** un paso de análisis, múltiples renderizadores.
- **Fragmentación segura:** dividir el texto antes de renderizar para que el formato inline nunca
  se rompa entre fragmentos.
- **Ajuste al canal:** mapear la misma IR a Slack mrkdwn, HTML de Telegram y rangos de estilo de Signal
  sin volver a analizar Markdown.

## Canalización

1. **Analizar Markdown -> IR**
   - La IR es texto plano más rangos de estilo (bold/italic/strike/code/spoiler) y rangos de enlace.
   - Los offsets son unidades de código UTF-16 para que los rangos de estilo de Signal se alineen con su API.
   - Las tablas se analizan solo cuando un canal activa la conversión de tablas.
2. **Fragmentar IR (primero formato)**
   - La fragmentación ocurre sobre el texto de la IR antes del renderizado.
   - El formato inline no se divide entre fragmentos; los rangos se recortan por fragmento.
3. **Renderizar por canal**
   - **Slack:** tokens mrkdwn (bold/italic/strike/code), enlaces como `<url|label>`.
   - **Telegram:** etiquetas HTML (`<b>`, `<i>`, `<s>`, `<code>`, `<pre><code>`, `<a href>`).
   - **Signal:** texto plano + rangos `text-style`; los enlaces se convierten en `label (url)` cuando la etiqueta es distinta.

## Ejemplo de IR

Markdown de entrada:

```markdown
Hello **world** — see [docs](https://docs.openclaw.ai).
```

IR (esquemática):

```json
{
  "text": "Hello world — see docs.",
  "styles": [{ "start": 6, "end": 11, "style": "bold" }],
  "links": [{ "start": 19, "end": 23, "href": "https://docs.openclaw.ai" }]
}
```

## Dónde se usa

- Los adaptadores salientes de Slack, Telegram y Signal renderizan desde la IR.
- Otros canales (WhatsApp, iMessage, Microsoft Teams, Discord) siguen usando texto plano o
  sus propias reglas de formato, con conversión de tablas Markdown aplicada antes de
  la fragmentación cuando está habilitada.

## Manejo de tablas

Las tablas Markdown no tienen compatibilidad consistente entre clientes de chat. Usa
`markdown.tables` para controlar la conversión por canal (y por cuenta).

- `code`: renderizar tablas como bloques de código (predeterminado para la mayoría de los canales).
- `bullets`: convertir cada fila en viñetas (predeterminado para Signal + WhatsApp).
- `off`: deshabilitar el análisis y la conversión de tablas; el texto bruto de la tabla pasa sin cambios.

Claves de configuración:

```yaml
channels:
  discord:
    markdown:
      tables: code
    accounts:
      work:
        markdown:
          tables: off
```

## Reglas de fragmentación

- Los límites de fragmentación provienen de los adaptadores/configuración del canal y se aplican al texto de la IR.
- Las vallas de código se conservan como un único bloque con una nueva línea final para que los canales
  las rendericen correctamente.
- Los prefijos de lista y de blockquote forman parte del texto de la IR, así que la fragmentación
  no divide a mitad de prefijo.
- Los estilos inline (bold/italic/strike/inline-code/spoiler) nunca se dividen entre
  fragmentos; el renderizador vuelve a abrir los estilos dentro de cada fragmento.

Si necesitas más información sobre el comportamiento de la fragmentación entre canales, consulta
[Streaming + chunking](/concepts/streaming).

## Política de enlaces

- **Slack:** `[label](url)` -> `<url|label>`; las URL simples permanecen simples. El autolink
  se desactiva durante el análisis para evitar enlaces duplicados.
- **Telegram:** `[label](url)` -> `<a href="url">label</a>` (modo de análisis HTML).
- **Signal:** `[label](url)` -> `label (url)` a menos que la etiqueta coincida con la URL.

## Spoilers

Los marcadores de spoiler (`||spoiler||`) se analizan solo para Signal, donde se asignan a
rangos de estilo SPOILER. Otros canales los tratan como texto plano.

## Cómo agregar o actualizar un formateador de canal

1. **Analizar una vez:** usa el helper compartido `markdownToIR(...)` con opciones apropiadas
   para el canal (autolink, estilo de encabezado, prefijo de blockquote).
2. **Renderizar:** implementa un renderizador con `renderMarkdownWithMarkers(...)` y un
   mapa de marcadores de estilo (o rangos de estilo de Signal).
3. **Fragmentar:** llama a `chunkMarkdownIR(...)` antes de renderizar; renderiza cada fragmento.
4. **Conectar el adaptador:** actualiza el adaptador saliente del canal para usar el nuevo fragmentador
   y renderizador.
5. **Probar:** agrega o actualiza pruebas de formato y una prueba de entrega saliente si el
   canal usa fragmentación.

## Problemas comunes

- Los tokens de Slack entre corchetes angulares (`<@U123>`, `<#C123>`, `<https://...>`) deben conservarse;
  escapa el HTML sin procesar de forma segura.
- El HTML de Telegram requiere escapar el texto fuera de las etiquetas para evitar marcado roto.
- Los rangos de estilo de Signal dependen de offsets UTF-16; no uses offsets de puntos de código.
- Conserva las nuevas líneas finales para los bloques de código con vallas para que los marcadores de cierre queden en
  su propia línea.
