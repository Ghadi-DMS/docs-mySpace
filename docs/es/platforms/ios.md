---
read_when:
    - Emparejar o volver a conectar el nodo iOS
    - Ejecutar la app iOS desde el código fuente
    - Depurar el descubrimiento de gateway o los comandos de canvas
summary: 'App iOS de nodo: conectarse a la Gateway, emparejamiento, canvas y solución de problemas'
title: App de iOS
x-i18n:
    generated_at: "2026-04-05T12:48:11Z"
    model: gpt-5.4
    provider: openai
    source_hash: 1e9d9cec58afd4003dff81d3e367bfbc6a634c1b229e433e08fd78fbb5f2e5a9
    source_path: platforms/ios.md
    workflow: 15
---

# App de iOS (nodo)

Disponibilidad: vista previa interna. La app de iOS aún no se distribuye públicamente.

## Qué hace

- Se conecta a una Gateway por WebSocket (LAN o tailnet).
- Expone capacidades de nodo: Canvas, instantánea de pantalla, captura de cámara, ubicación, modo Talk, Voice wake.
- Recibe comandos `node.invoke` e informa eventos de estado del nodo.

## Requisitos

- Gateway ejecutándose en otro dispositivo (macOS, Linux o Windows mediante WSL2).
- Ruta de red:
  - La misma LAN mediante Bonjour, **o**
  - tailnet mediante DNS-SD unicast (dominio de ejemplo: `openclaw.internal.`), **o**
  - host/puerto manual (fallback).

## Inicio rápido (emparejar + conectar)

1. Inicia la Gateway:

```bash
openclaw gateway --port 18789
```

2. En la app de iOS, abre Settings y elige una gateway detectada (o habilita Manual Host e introduce host/puerto).

3. Aprueba la solicitud de emparejamiento en el host de la gateway:

```bash
openclaw devices list
openclaw devices approve <requestId>
```

Si la app vuelve a intentar el emparejamiento con detalles de autenticación cambiados (rol/scopes/clave pública),
la solicitud pendiente anterior se reemplaza y se crea un nuevo `requestId`.
Vuelve a ejecutar `openclaw devices list` antes de aprobar.

4. Verifica la conexión:

```bash
openclaw nodes status
openclaw gateway call node.list --params "{}"
```

## Push respaldado por relay para compilaciones oficiales

Las compilaciones oficiales distribuidas de iOS usan el relay push externo en lugar de publicar el token APNs sin procesar
en la gateway.

Requisito del lado de la gateway:

```json5
{
  gateway: {
    push: {
      apns: {
        relay: {
          baseUrl: "https://relay.example.com",
        },
      },
    },
  },
}
```

Cómo funciona el flujo:

- La app de iOS se registra en el relay usando App Attest y el recibo de la app.
- El relay devuelve un identificador opaco del relay junto con un permiso de envío delimitado al registro.
- La app de iOS obtiene la identidad de la gateway emparejada y la incluye en el registro del relay, de modo que el registro respaldado por relay se delega a esa gateway específica.
- La app reenvía ese registro respaldado por relay a la gateway emparejada con `push.apns.register`.
- La gateway usa ese identificador de relay almacenado para `push.test`, activaciones en segundo plano y avisos de activación.
- La URL base del relay de la gateway debe coincidir con la URL del relay incorporada en la compilación oficial/TestFlight de iOS.
- Si la app se conecta más tarde a otra gateway o a una compilación con una URL base de relay distinta, actualiza el registro del relay en lugar de reutilizar el vínculo anterior.

Qué **no** necesita la gateway para esta ruta:

- Ningún token de relay para todo el despliegue.
- Ninguna clave APNs directa para envíos oficiales/TestFlight respaldados por relay.

Flujo esperado para el operador:

1. Instala la compilación oficial/TestFlight de iOS.
2. Establece `gateway.push.apns.relay.baseUrl` en la gateway.
3. Empareja la app con la gateway y deja que termine de conectarse.
4. La app publica `push.apns.register` automáticamente después de tener un token APNs, de que la sesión del operador esté conectada y de que el registro del relay se complete correctamente.
5. Después de eso, `push.test`, las activaciones de reconexión y los avisos de activación pueden usar el registro respaldado por relay almacenado.

Nota de compatibilidad:

- `OPENCLAW_APNS_RELAY_BASE_URL` sigue funcionando como sobrescritura temporal de entorno para la gateway.

## Flujo de autenticación y confianza

El relay existe para imponer dos restricciones que APNs directo en la gateway no puede proporcionar en
compilaciones oficiales de iOS:

- Solo las compilaciones genuinas de OpenClaw para iOS distribuidas mediante Apple pueden usar el relay alojado.
- Una gateway solo puede enviar push respaldado por relay para dispositivos iOS que se emparejaron con esa
  gateway específica.

Salto por salto:

1. `iOS app -> gateway`
   - La app primero se empareja con la gateway mediante el flujo normal de autenticación de Gateway.
   - Eso le da a la app una sesión autenticada de nodo más una sesión autenticada de operador.
   - La sesión de operador se usa para llamar a `gateway.identity.get`.

2. `iOS app -> relay`
   - La app llama a los endpoints de registro del relay mediante HTTPS.
   - El registro incluye prueba App Attest más el recibo de la app.
   - El relay valida el bundle ID, la prueba App Attest y el recibo de Apple, y exige la
     ruta oficial/de producción de distribución.
   - Esto es lo que impide que las compilaciones locales de Xcode/desarrollo usen el relay alojado. Una compilación local puede estar
     firmada, pero no satisface la prueba oficial de distribución de Apple que el relay espera.

3. `delegación de identidad de gateway`
   - Antes del registro del relay, la app obtiene la identidad de la gateway emparejada desde
     `gateway.identity.get`.
   - La app incluye esa identidad de gateway en la carga de registro del relay.
   - El relay devuelve un identificador de relay y un permiso de envío delimitado al registro que se delegan a
     esa identidad de gateway.

4. `gateway -> relay`
   - La gateway almacena el identificador del relay y el permiso de envío de `push.apns.register`.
   - En `push.test`, activaciones de reconexión y avisos de activación, la gateway firma la solicitud de envío con su
     propia identidad de dispositivo.
   - El relay verifica tanto el permiso de envío almacenado como la firma de la gateway frente a la identidad de gateway
     delegada desde el registro.
   - Otra gateway no puede reutilizar ese registro almacenado, aunque de algún modo obtenga el identificador.

5. `relay -> APNs`
   - El relay es propietario de las credenciales de producción de APNs y del token APNs sin procesar para la compilación oficial.
   - La gateway nunca almacena el token APNs sin procesar para compilaciones oficiales respaldadas por relay.
   - El relay envía el push final a APNs en nombre de la gateway emparejada.

Por qué se creó este diseño:

- Para mantener las credenciales de producción de APNs fuera de las gateways de usuario.
- Para evitar almacenar tokens APNs sin procesar de compilaciones oficiales en la gateway.
- Para permitir el uso del relay alojado solo para compilaciones oficiales/TestFlight de OpenClaw.
- Para impedir que una gateway envíe push de activación a dispositivos iOS pertenecientes a otra gateway.

Las compilaciones locales/manuales siguen usando APNs directo. Si estás probando esas compilaciones sin relay, la
gateway aún necesita credenciales APNs directas:

```bash
export OPENCLAW_APNS_TEAM_ID="TEAMID"
export OPENCLAW_APNS_KEY_ID="KEYID"
export OPENCLAW_APNS_PRIVATE_KEY_P8="$(cat /path/to/AuthKey_KEYID.p8)"
```

## Rutas de descubrimiento

### Bonjour (LAN)

La app de iOS explora `_openclaw-gw._tcp` en `local.` y, cuando está configurado, el mismo
dominio de descubrimiento DNS-SD de área amplia. Las gateways en la misma LAN aparecen automáticamente desde `local.`;
el descubrimiento entre redes puede usar el dominio de área amplia configurado sin cambiar el tipo de beacon.

### tailnet (entre redes)

Si mDNS está bloqueado, usa una zona DNS-SD unicast (elige un dominio; ejemplo:
`openclaw.internal.`) y DNS dividido de Tailscale.
Consulta [Bonjour](/gateway/bonjour) para ver el ejemplo con CoreDNS.

### Host/puerto manual

En Settings, habilita **Manual Host** e introduce el host + puerto de la gateway (predeterminado `18789`).

## Canvas + A2UI

El nodo iOS renderiza un canvas WKWebView. Usa `node.invoke` para controlarlo:

```bash
openclaw nodes invoke --node "iOS Node" --command canvas.navigate --params '{"url":"http://<gateway-host>:18789/__openclaw__/canvas/"}'
```

Notas:

- El canvas host de la Gateway sirve `/__openclaw__/canvas/` y `/__openclaw__/a2ui/`.
- Se sirve desde el servidor HTTP de Gateway (mismo puerto que `gateway.port`, predeterminado `18789`).
- El nodo iOS navega automáticamente a A2UI al conectarse cuando se anuncia una URL de canvas host.
- Vuelve al scaffold integrado con `canvas.navigate` y `{"url":""}`.

### Canvas eval / snapshot

```bash
openclaw nodes invoke --node "iOS Node" --command canvas.eval --params '{"javaScript":"(() => { const {ctx} = window.__openclaw; ctx.clearRect(0,0,innerWidth,innerHeight); ctx.lineWidth=6; ctx.strokeStyle=\"#ff2d55\"; ctx.beginPath(); ctx.moveTo(40,40); ctx.lineTo(innerWidth-40, innerHeight-40); ctx.stroke(); return \"ok\"; })()"}'
```

```bash
openclaw nodes invoke --node "iOS Node" --command canvas.snapshot --params '{"maxWidth":900,"format":"jpeg"}'
```

## Voice wake + modo Talk

- Voice wake y el modo Talk están disponibles en Settings.
- iOS puede suspender el audio en segundo plano; trata las funciones de voz como best-effort cuando la app no está activa.

## Errores comunes

- `NODE_BACKGROUND_UNAVAILABLE`: lleva la app de iOS al primer plano (los comandos de canvas/camera/screen lo requieren).
- `A2UI_HOST_NOT_CONFIGURED`: la Gateway no anunció una URL de canvas host; revisa `canvasHost` en [Configuración de Gateway](/gateway/configuration).
- La solicitud de emparejamiento nunca aparece: ejecuta `openclaw devices list` y aprueba manualmente.
- La reconexión falla después de reinstalar: se borró el token de emparejamiento del Keychain; vuelve a emparejar el nodo.

## Documentación relacionada

- [Emparejamiento](/channels/pairing)
- [Discovery](/gateway/discovery)
- [Bonjour](/gateway/bonjour)
