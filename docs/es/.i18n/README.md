---
x-i18n:
    generated_at: "2026-04-05T12:34:17Z"
    model: gpt-5.4
    provider: openai
    source_hash: adff26fa8858af2759b231ea48bfc01f89c110cd9b3774a8f783e282c16f77fb
    source_path: .i18n/README.md
    workflow: 15
---

# Recursos de i18n de la documentación de OpenClaw

Esta carpeta almacena la configuración de traducción para el repositorio fuente de la documentación.

Los árboles de configuraciones regionales generados y la memoria de traducción activa ahora se encuentran en el repositorio de publicación:

- repositorio: `openclaw/docs`
- copia local: `~/Projects/openclaw-docs`

## Fuente de verdad

- La documentación en inglés se redacta en `openclaw/openclaw`.
- El árbol de documentación fuente se encuentra en `docs/`.
- El repositorio fuente ya no conserva árboles de configuraciones regionales generados confirmados, como `docs/zh-CN/**`, `docs/ja-JP/**`, `docs/es/**`, `docs/pt-BR/**`, `docs/ko/**`, `docs/de/**`, `docs/fr/**` o `docs/ar/**`.

## Flujo de extremo a extremo

1. Edita la documentación en inglés en `openclaw/openclaw`.
2. Haz push a `main`.
3. `openclaw/openclaw/.github/workflows/docs-sync-publish.yml` replica el árbol de documentación en `openclaw/docs`.
4. El script de sincronización reescribe `docs/docs.json` de publicación para que los bloques generados del selector de configuraciones regionales existan allí aunque ya no estén confirmados en el repositorio fuente.
5. `openclaw/docs/.github/workflows/translate-zh-cn.yml` actualiza `docs/zh-CN/**` una vez al día, bajo demanda y después de los despachos de publicación del repositorio fuente.
6. `openclaw/docs/.github/workflows/translate-ja-jp.yml` hace lo mismo para `docs/ja-JP/**`.
7. `openclaw/docs/.github/workflows/translate-es.yml`, `translate-pt-br.yml`, `translate-ko.yml`, `translate-de.yml`, `translate-fr.yml` y `translate-ar.yml` hacen lo mismo para `docs/es/**`, `docs/pt-BR/**`, `docs/ko/**`, `docs/de/**`, `docs/fr/**` y `docs/ar/**`.

## Por qué existe esta división

- Mantener la salida de configuraciones regionales generada fuera del repositorio principal del producto.
- Mantener Mintlify en un único árbol de documentación publicado.
- Preservar el selector de idioma integrado permitiendo que el repositorio de publicación sea propietario de los árboles de configuraciones regionales generados.

## Archivos en esta carpeta

- `glossary.<lang>.json` — asignaciones de términos preferidos usadas como guía del prompt.
- `ar-navigation.json`, `de-navigation.json`, `es-navigation.json`, `fr-navigation.json`, `ja-navigation.json`, `ko-navigation.json`, `pt-BR-navigation.json`, `zh-Hans-navigation.json` — bloques del selector de configuraciones regionales de Mintlify reinsertados en el repositorio de publicación durante la sincronización.
- `<lang>.tm.jsonl` — memoria de traducción indexada por flujo de trabajo + modelo + hash de texto.

En este repositorio, los archivos TM de configuraciones regionales generados, como `docs/.i18n/zh-CN.tm.jsonl`, `docs/.i18n/ja-JP.tm.jsonl`, `docs/.i18n/es.tm.jsonl`, `docs/.i18n/pt-BR.tm.jsonl`, `docs/.i18n/ko.tm.jsonl`, `docs/.i18n/de.tm.jsonl`, `docs/.i18n/fr.tm.jsonl` y `docs/.i18n/ar.tm.jsonl`, intencionalmente ya no se confirman.

## Formato del glosario

`glossary.<lang>.json` es un arreglo de entradas:

```json
{
  "source": "troubleshooting",
  "target": "故障排除"
}
```

Campos:

- `source`: frase en inglés (o en el idioma fuente) que se debe preferir.
- `target`: salida de traducción preferida.

## Mecánica de traducción

- `scripts/docs-i18n` sigue siendo responsable de la generación de traducciones.
- El modo de documentación escribe `x-i18n.source_hash` en cada página traducida.
- Cada flujo de trabajo de publicación precalcula una lista de archivos pendientes comparando el hash actual de la fuente en inglés con el `x-i18n.source_hash` almacenado de la configuración regional.
- Si el conteo de pendientes es `0`, el paso de traducción costoso se omite por completo.
- Si hay archivos pendientes, el flujo de trabajo traduce solo esos archivos.
- El flujo de trabajo de publicación vuelve a intentar los fallos transitorios de formato del modelo, pero los archivos sin cambios siguen omitiéndose porque la misma comprobación de hash se ejecuta en cada reintento.
- El repositorio fuente también despacha actualizaciones de zh-CN, ja-JP, es, pt-BR, ko, de, fr y ar después de las versiones publicadas de GitHub para que la documentación de la versión pueda ponerse al día sin esperar al cron diario.

## Notas operativas

- Los metadatos de sincronización se escriben en `.openclaw-sync/source.json` en el repositorio de publicación.
- Secreto del repositorio fuente: `OPENCLAW_DOCS_SYNC_TOKEN`
- Secreto del repositorio de publicación: `OPENCLAW_DOCS_I18N_OPENAI_API_KEY`
- Si la salida de una configuración regional parece desactualizada, revisa primero el flujo de trabajo `Translate <locale>` correspondiente en `openclaw/docs`.
