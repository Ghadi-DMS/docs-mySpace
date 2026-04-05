---
read_when:
    - Necesitas registros de depuración dirigidos sin aumentar los niveles globales de registro
    - Necesitas capturar registros específicos del subsistema para soporte
summary: Indicadores de diagnóstico para registros de depuración dirigidos
title: Indicadores de diagnóstico
x-i18n:
    generated_at: "2026-04-05T12:41:07Z"
    model: gpt-5.4
    provider: openai
    source_hash: daf0eca0e6bd1cbc2c400b2e94e1698709a96b9cdba1a8cf00bd580a61829124
    source_path: diagnostics/flags.md
    workflow: 15
---

# Indicadores de diagnóstico

Los indicadores de diagnóstico te permiten habilitar registros de depuración dirigidos sin activar el registro detallado en todas partes. Los indicadores son opcionales y no tienen efecto a menos que un subsistema los compruebe.

## Cómo funciona

- Los indicadores son cadenas (sin distinción entre mayúsculas y minúsculas).
- Puedes habilitar indicadores en la configuración o mediante una anulación con variable de entorno.
- Se admiten comodines:
  - `telegram.*` coincide con `telegram.http`
  - `*` habilita todos los indicadores

## Habilitar mediante configuración

```json
{
  "diagnostics": {
    "flags": ["telegram.http"]
  }
}
```

Varios indicadores:

```json
{
  "diagnostics": {
    "flags": ["telegram.http", "gateway.*"]
  }
}
```

Reinicia el gateway después de cambiar los indicadores.

## Anulación por variable de entorno (puntual)

```bash
OPENCLAW_DIAGNOSTICS=telegram.http,telegram.payload
```

Deshabilitar todos los indicadores:

```bash
OPENCLAW_DIAGNOSTICS=0
```

## Dónde van los registros

Los indicadores emiten registros en el archivo estándar de diagnósticos. De forma predeterminada:

```
/tmp/openclaw/openclaw-YYYY-MM-DD.log
```

Si configuras `logging.file`, usa esa ruta en su lugar. Los registros están en formato JSONL (un objeto JSON por línea). La redacción sigue aplicándose según `logging.redactSensitive`.

## Extraer registros

Elige el archivo de registro más reciente:

```bash
ls -t /tmp/openclaw/openclaw-*.log | head -n 1
```

Filtra diagnósticos HTTP de Telegram:

```bash
rg "telegram http error" /tmp/openclaw/openclaw-*.log
```

O sigue los registros mientras reproduces el problema:

```bash
tail -f /tmp/openclaw/openclaw-$(date +%F).log | rg "telegram http error"
```

Para gateways remotos, también puedes usar `openclaw logs --follow` (consulta [/cli/logs](/cli/logs)).

## Notas

- Si `logging.level` está configurado por encima de `warn`, es posible que estos registros se supriman. El valor predeterminado `info` es correcto.
- Es seguro dejar los indicadores habilitados; solo afectan al volumen de registros del subsistema específico.
- Usa [/logging](/logging) para cambiar destinos, niveles y redacción de registros.
