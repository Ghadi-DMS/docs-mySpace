---
read_when:
    - Trabajando en código o pruebas de integración con Pi
    - Ejecutando flujos específicos de lint, typecheck y pruebas live para Pi
summary: 'Flujo de trabajo de desarrollo para la integración con Pi: compilación, pruebas y validación live'
title: Flujo de trabajo de desarrollo de Pi
x-i18n:
    generated_at: "2026-04-05T12:47:42Z"
    model: gpt-5.4
    provider: openai
    source_hash: f61ebe29ea38ac953a03fe848fe5ac6b6de4bace5e6955b76ae9a7d093eb0cc5
    source_path: pi-dev.md
    workflow: 15
---

# Flujo de trabajo de desarrollo de Pi

Esta guía resume un flujo de trabajo razonable para trabajar en la integración con pi en OpenClaw.

## Comprobación de tipos y linting

- Puerta local predeterminada: `pnpm check`
- Puerta de compilación: `pnpm build` cuando el cambio puede afectar la salida de compilación, el empaquetado o los límites de carga diferida/módulos
- Puerta completa antes de aterrizar para cambios importantes de Pi: `pnpm check && pnpm test`

## Ejecutar pruebas de Pi

Ejecuta directamente el conjunto de pruebas centrado en Pi con Vitest:

```bash
pnpm test \
  "src/agents/pi-*.test.ts" \
  "src/agents/pi-embedded-*.test.ts" \
  "src/agents/pi-tools*.test.ts" \
  "src/agents/pi-settings.test.ts" \
  "src/agents/pi-tool-definition-adapter*.test.ts" \
  "src/agents/pi-hooks/**/*.test.ts"
```

Para incluir la prueba live del proveedor:

```bash
OPENCLAW_LIVE_TEST=1 pnpm test src/agents/pi-embedded-runner-extraparams.live.test.ts
```

Esto cubre las principales suites unitarias de Pi:

- `src/agents/pi-*.test.ts`
- `src/agents/pi-embedded-*.test.ts`
- `src/agents/pi-tools*.test.ts`
- `src/agents/pi-settings.test.ts`
- `src/agents/pi-tool-definition-adapter.test.ts`
- `src/agents/pi-hooks/*.test.ts`

## Pruebas manuales

Flujo recomendado:

- Ejecuta el gateway en modo dev:
  - `pnpm gateway:dev`
- Activa el agente directamente:
  - `pnpm openclaw agent --message "Hello" --thinking low`
- Usa la TUI para depuración interactiva:
  - `pnpm tui`

Para el comportamiento de llamadas a herramientas, pide una acción `read` o `exec` para que puedas ver el streaming de herramientas y el manejo de la carga útil.

## Reinicio completo

El state vive bajo el directorio de estado de OpenClaw. El predeterminado es `~/.openclaw`. Si `OPENCLAW_STATE_DIR` está definido, usa ese directorio en su lugar.

Para restablecerlo todo:

- `openclaw.json` para la configuración
- `agents/<agentId>/agent/auth-profiles.json` para perfiles de autenticación de modelos (claves API + OAuth)
- `credentials/` para state de proveedor/canal que aún vive fuera del almacén de perfiles de autenticación
- `agents/<agentId>/sessions/` para el historial de sesiones del agente
- `agents/<agentId>/sessions/sessions.json` para el índice de sesiones
- `sessions/` si existen rutas heredadas
- `workspace/` si quieres un workspace en blanco

Si solo quieres restablecer sesiones, elimina `agents/<agentId>/sessions/` para ese agente. Si quieres conservar la autenticación, deja `agents/<agentId>/agent/auth-profiles.json` y cualquier state de proveedor bajo `credentials/` en su lugar.

## Referencias

- [Testing](/help/testing)
- [Getting Started](/es/start/getting-started)
