---
read_when:
    - Ajustar valores predeterminados del modo elevado, listas de permitidos o el comportamiento de comandos con barra
    - Entender cómo los agentes en sandbox pueden acceder al host
summary: 'Modo exec elevado: ejecuta comandos fuera del sandbox desde un agente en sandbox'
title: Modo elevado
x-i18n:
    generated_at: "2026-04-05T12:55:18Z"
    model: gpt-5.4
    provider: openai
    source_hash: f6f0ca0a7c03c94554a70fee775aa92085f15015850c3abaa2c1c46ced9d3c2e
    source_path: tools/elevated.md
    workflow: 15
---

# Modo elevado

Cuando un agente se ejecuta dentro de un sandbox, sus comandos `exec` quedan confinados al
entorno del sandbox. **El modo elevado** permite que el agente salga de ese entorno y ejecute comandos
fuera del sandbox, con puertas de aprobación configurables.

<Info>
  El modo elevado solo cambia el comportamiento cuando el agente está **en sandbox**. Para
  agentes sin sandbox, exec ya se ejecuta en el host.
</Info>

## Directivas

Controla el modo elevado por sesión con comandos con barra:

| Directiva       | Qué hace                                                             |
| --------------- | -------------------------------------------------------------------- |
| `/elevated on`   | Ejecuta fuera del sandbox en la ruta de host configurada, mantiene las aprobaciones    |
| `/elevated ask`  | Igual que `on` (alias)                                                   |
| `/elevated full` | Ejecuta fuera del sandbox en la ruta de host configurada y omite las aprobaciones |
| `/elevated off`  | Vuelve a la ejecución confinada al sandbox                                   |

También está disponible como `/elev on|off|ask|full`.

Envía `/elevated` sin argumento para ver el nivel actual.

## Cómo funciona

<Steps>
  <Step title="Comprobar disponibilidad">
    Elevated debe estar habilitado en la configuración y el remitente debe estar en la lista de permitidos:

    ```json5
    {
      tools: {
        elevated: {
          enabled: true,
          allowFrom: {
            discord: ["user-id-123"],
            whatsapp: ["+15555550123"],
          },
        },
      },
    }
    ```

  </Step>

  <Step title="Establecer el nivel">
    Envía un mensaje que solo contenga la directiva para establecer el valor predeterminado de la sesión:

    ```
    /elevated full
    ```

    O úsala en línea (se aplica solo a ese mensaje):

    ```
    /elevated on run the deployment script
    ```

  </Step>

  <Step title="Los comandos se ejecutan fuera del sandbox">
    Con elevated activo, las llamadas a `exec` salen del sandbox. El host efectivo es
    `gateway` de forma predeterminada, o `node` cuando el destino exec configurado/de sesión es
    `node`. En modo `full`, se omiten las aprobaciones de exec. En modo `on`/`ask`,
    siguen aplicándose las reglas de aprobación configuradas.
  </Step>
</Steps>

## Orden de resolución

1. **Directiva en línea** en el mensaje (se aplica solo a ese mensaje)
2. **Anulación de sesión** (se establece enviando un mensaje que solo contenga una directiva)
3. **Valor predeterminado global** (`agents.defaults.elevatedDefault` en la configuración)

## Disponibilidad y listas de permitidos

- **Puerta global**: `tools.elevated.enabled` (debe ser `true`)
- **Lista de permitidos del remitente**: `tools.elevated.allowFrom` con listas por canal
- **Puerta por agente**: `agents.list[].tools.elevated.enabled` (solo puede restringir más)
- **Lista de permitidos por agente**: `agents.list[].tools.elevated.allowFrom` (el remitente debe coincidir tanto con la global como con la del agente)
- **Fallback de Discord**: si se omite `tools.elevated.allowFrom.discord`, se usa `channels.discord.allowFrom` como fallback
- **Todas las puertas deben aprobarse**; en caso contrario, elevated se considera no disponible

Formatos de entrada de la lista de permitidos:

| Prefijo                 | Coincide con                    |
| ----------------------- | ------------------------------- |
| (ninguno)               | ID del remitente, E.164 o campo From |
| `name:`                 | Nombre para mostrar del remitente             |
| `username:`             | Nombre de usuario del remitente                 |
| `tag:`                  | Etiqueta del remitente                      |
| `id:`, `from:`, `e164:` | Selección explícita de identidad     |

## Lo que elevated no controla

- **Política de herramientas**: si `exec` está denegado por la política de herramientas, elevated no puede anularlo
- **Política de selección de host**: elevated no convierte `auto` en una anulación libre entre hosts. Usa las reglas del destino exec configurado/de sesión, y elige `node` solo cuando el destino ya es `node`.
- **Separado de `/exec`**: la directiva `/exec` ajusta los valores predeterminados de exec por sesión para remitentes autorizados y no requiere modo elevado

## Relacionado

- [Herramienta Exec](/tools/exec) — ejecución de comandos de shell
- [Aprobaciones de exec](/tools/exec-approvals) — sistema de aprobación y lista de permitidos
- [Sandboxing](/es/gateway/sandboxing) — configuración del sandbox
- [Sandbox vs Tool Policy vs Elevated](/es/gateway/sandbox-vs-tool-policy-vs-elevated)
