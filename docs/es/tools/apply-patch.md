---
read_when:
    - Necesitas ediciones estructuradas en varios archivos
    - Quieres documentar o depurar ediciones basadas en parches
summary: Aplica parches en varios archivos con la herramienta apply_patch
title: Herramienta apply_patch
x-i18n:
    generated_at: "2026-04-05T12:54:33Z"
    model: gpt-5.4
    provider: openai
    source_hash: acca6e702e7ccdf132c71dc6d973f1d435ad6d772e1b620512c8969420cb8f7a
    source_path: tools/apply-patch.md
    workflow: 15
---

# herramienta apply_patch

Aplica cambios en archivos usando un formato de parche estructurado. Esto es ideal para ediciones
en varios archivos o con varios bloques donde una sola llamada a `edit` sería frágil.

La herramienta acepta una sola cadena `input` que envuelve una o más operaciones de archivo:

```
*** Begin Patch
*** Add File: path/to/file.txt
+line 1
+line 2
*** Update File: src/app.ts
@@
-old line
+new line
*** Delete File: obsolete.txt
*** End Patch
```

## Parámetros

- `input` (obligatorio): Contenido completo del parche, incluidos `*** Begin Patch` y `*** End Patch`.

## Notas

- Las rutas del parche admiten rutas relativas (desde el directorio del espacio de trabajo) y rutas absolutas.
- `tools.exec.applyPatch.workspaceOnly` usa `true` de forma predeterminada (limitado al espacio de trabajo). Establécelo en `false` solo si intencionalmente quieres que `apply_patch` escriba o elimine fuera del directorio del espacio de trabajo.
- Usa `*** Move to:` dentro de un bloque `*** Update File:` para renombrar archivos.
- `*** End of File` marca una inserción solo al final del archivo cuando sea necesario.
- Disponible de forma predeterminada para los modelos OpenAI y OpenAI Codex. Establece
  `tools.exec.applyPatch.enabled: false` para desactivarla.
- Opcionalmente, limítala por modelo mediante
  `tools.exec.applyPatch.allowModels`.
- La configuración solo está en `tools.exec`.

## Ejemplo

```json
{
  "tool": "apply_patch",
  "input": "*** Begin Patch\n*** Update File: src/index.ts\n@@\n-const foo = 1\n+const foo = 2\n*** End Patch"
}
```
