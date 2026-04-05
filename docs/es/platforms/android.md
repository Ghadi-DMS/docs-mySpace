---
read_when:
    - Emparejar o volver a conectar el node de Android
    - Depurar el descubrimiento o la autenticación del gateway en Android
    - Verificar la paridad del historial de chat entre clientes
summary: 'Aplicación de Android (node): manual operativo de conexión + superficie de comandos Connect/Chat/Voice/Canvas'
title: Aplicación de Android
x-i18n:
    generated_at: "2026-04-05T12:48:12Z"
    model: gpt-5.4
    provider: openai
    source_hash: 2223891afc3aa34af4aaf5410b4f1c6aebcf24bab68a6c47dd9832882d5260db
    source_path: platforms/android.md
    workflow: 15
---

# Aplicación de Android (Node)

> **Nota:** La aplicación de Android todavía no se ha lanzado públicamente. El código fuente está disponible en el [repositorio de OpenClaw](https://github.com/openclaw/openclaw) en `apps/android`. Puedes compilarla tú mismo usando Java 17 y el Android SDK (`./gradlew :app:assemblePlayDebug`). Consulta [apps/android/README.md](https://github.com/openclaw/openclaw/blob/main/apps/android/README.md) para ver las instrucciones de compilación.

## Resumen de compatibilidad

- Rol: aplicación node complementaria (Android no aloja el Gateway).
- Gateway requerido: sí (ejecútalo en macOS, Linux o Windows mediante WSL2).
- Instalación: [Primeros pasos](/es/start/getting-started) + [Emparejamiento](/channels/pairing).
- Gateway: [Manual operativo](/gateway) + [Configuración](/gateway/configuration).
  - Protocolos: [Protocolo del Gateway](/gateway/protocol) (nodes + plano de control).

## Control del sistema

El control del sistema (launchd/systemd) reside en el host del Gateway. Consulta [Gateway](/gateway).

## Manual operativo de conexión

Aplicación node de Android ⇄ (mDNS/NSD + WebSocket) ⇄ **Gateway**

Android se conecta directamente al WebSocket del Gateway y usa emparejamiento de dispositivos (`role: node`).

Para hosts Tailscale o públicos, Android requiere un endpoint seguro:

- Preferido: Tailscale Serve / Funnel con `https://<magicdns>` / `wss://<magicdns>`
- También compatible: cualquier otra URL `wss://` del Gateway con un endpoint TLS real
- `ws://` en texto claro sigue siendo compatible en direcciones LAN privadas / hosts `.local`, además de `localhost`, `127.0.0.1` y el puente del emulador de Android (`10.0.2.2`)

### Requisitos previos

- Puedes ejecutar el Gateway en la máquina “principal”.
- El dispositivo/emulador Android puede alcanzar el WebSocket del gateway:
  - En la misma LAN con mDNS/NSD, **o**
  - En la misma tailnet de Tailscale usando Wide-Area Bonjour / DNS-SD unicast (consulta abajo), **o**
  - Host/puerto del gateway manual (respaldo)
- El emparejamiento móvil por tailnet/público **no** usa endpoints `ws://` directos de IP de tailnet. Usa Tailscale Serve u otra URL `wss://` en su lugar.
- Puedes ejecutar la CLI (`openclaw`) en la máquina del gateway (o mediante SSH).

### 1) Iniciar el Gateway

```bash
openclaw gateway --port 18789 --verbose
```

Confirma en los registros que ves algo como:

- `listening on ws://0.0.0.0:18789`

Para acceso remoto de Android mediante Tailscale, prefiere Serve/Funnel en lugar de un enlace directo a tailnet:

```bash
openclaw gateway --tailscale serve
```

Esto proporciona a Android un endpoint seguro `wss://` / `https://`. Una configuración simple de `gateway.bind: "tailnet"` no es suficiente para el primer emparejamiento remoto de Android a menos que también termines TLS por separado.

### 2) Verificar el descubrimiento (opcional)

Desde la máquina del gateway:

```bash
dns-sd -B _openclaw-gw._tcp local.
```

Más notas de depuración: [Bonjour](/gateway/bonjour).

Si también configuraste un dominio de descubrimiento de área amplia, compáralo con:

```bash
openclaw gateway discover --json
```

Eso muestra `local.` junto con el dominio de área amplia configurado en una sola pasada y usa el endpoint de servicio resuelto en lugar de pistas solo de TXT.

#### Descubrimiento por tailnet (Viena ⇄ Londres) mediante DNS-SD unicast

El descubrimiento NSD/mDNS de Android no atraviesa redes. Si tu node de Android y el gateway están en redes distintas pero conectados mediante Tailscale, usa Wide-Area Bonjour / DNS-SD unicast en su lugar.

El descubrimiento por sí solo no es suficiente para el emparejamiento de Android por tailnet/público. La ruta descubierta sigue necesitando un endpoint seguro (`wss://` o Tailscale Serve):

1. Configura una zona DNS-SD (por ejemplo `openclaw.internal.`) en el host del gateway y publica registros `_openclaw-gw._tcp`.
2. Configura split DNS de Tailscale para el dominio elegido apuntando a ese servidor DNS.

Detalles y ejemplo de configuración de CoreDNS: [Bonjour](/gateway/bonjour).

### 3) Conectar desde Android

En la aplicación de Android:

- La aplicación mantiene activa su conexión con el gateway mediante un **servicio en primer plano** (notificación persistente).
- Abre la pestaña **Connect**.
- Usa el modo **Setup Code** o **Manual**.
- Si el descubrimiento está bloqueado, usa host/puerto manual en **Advanced controls**. Para hosts LAN privados, `ws://` sigue funcionando. Para hosts Tailscale/públicos, activa TLS y usa un endpoint `wss://` / de Tailscale Serve.

Después del primer emparejamiento correcto, Android vuelve a conectarse automáticamente al iniciarse:

- Endpoint manual (si está habilitado), o en caso contrario
- El último gateway descubierto (mejor esfuerzo).

### 4) Aprobar el emparejamiento (CLI)

En la máquina del gateway:

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw devices reject <requestId>
```

Detalles del emparejamiento: [Emparejamiento](/channels/pairing).

### 5) Verificar que el node está conectado

- Mediante el estado de nodes:

  ```bash
  openclaw nodes status
  ```

- Mediante el Gateway:

  ```bash
  openclaw gateway call node.list --params "{}"
  ```

### 6) Chat + historial

La pestaña Chat de Android admite selección de sesión (predeterminada `main`, además de otras sesiones existentes):

- Historial: `chat.history` (normalizado para visualización; las etiquetas de directivas en línea
  se eliminan del texto visible, las cargas útiles XML de llamada a herramientas en texto sin formato (incluidas
  `<tool_call>...</tool_call>`, `<function_call>...</function_call>`,
  `<tool_calls>...</tool_calls>`, `<function_calls>...</function_calls>` y
  bloques truncados de llamada a herramientas) y los tokens de control del modelo filtrados en ASCII/de ancho completo
  se eliminan, se omiten filas de asistente compuestas solo por tokens silenciosos exactos como `NO_REPLY` /
  `no_reply`, y las filas sobredimensionadas pueden sustituirse por marcadores)
- Envío: `chat.send`
- Actualizaciones push (mejor esfuerzo): `chat.subscribe` → `event:"chat"`

### 7) Canvas + cámara

#### Host de Canvas del Gateway (recomendado para contenido web)

Si quieres que el node muestre HTML/CSS/JS real que el agente pueda editar en disco, apunta el node al host de canvas del Gateway.

Nota: los nodes cargan canvas desde el servidor HTTP del Gateway (mismo puerto que `gateway.port`, predeterminado `18789`).

1. Crea `~/.openclaw/workspace/canvas/index.html` en el host del gateway.

2. Navega el node hacia él (LAN):

```bash
openclaw nodes invoke --node "<Android Node>" --command canvas.navigate --params '{"url":"http://<gateway-hostname>.local:18789/__openclaw__/canvas/"}'
```

Tailnet (opcional): si ambos dispositivos están en Tailscale, usa un nombre MagicDNS o una IP de tailnet en lugar de `.local`, por ejemplo `http://<gateway-magicdns>:18789/__openclaw__/canvas/`.

Este servidor inyecta un cliente de recarga en vivo en el HTML y recarga al cambiar los archivos.
El host A2UI se encuentra en `http://<gateway-host>:18789/__openclaw__/a2ui/`.

Comandos de canvas (solo en primer plano):

- `canvas.eval`, `canvas.snapshot`, `canvas.navigate` (usa `{"url":""}` o `{"url":"/"}` para volver al scaffold predeterminado). `canvas.snapshot` devuelve `{ format, base64 }` (predeterminado `format="jpeg"`).
- A2UI: `canvas.a2ui.push`, `canvas.a2ui.reset` (alias heredado `canvas.a2ui.pushJSONL`)

Comandos de cámara (solo en primer plano; controlados por permisos):

- `camera.snap` (jpg)
- `camera.clip` (mp4)

Consulta [Node de cámara](/nodes/camera) para parámetros y utilidades de la CLI.

### 8) Voice + superficie de comandos ampliada de Android

- Voice: Android usa un único flujo de activación/desactivación del micrófono en la pestaña Voice con captura de transcripción y reproducción `talk.speak`. El TTS local del sistema solo se usa cuando `talk.speak` no está disponible. Voice se detiene cuando la aplicación sale del primer plano.
- Los conmutadores de activación por voz/modo conversación están eliminados actualmente de la UX/runtime de Android.
- Familias adicionales de comandos de Android (la disponibilidad depende del dispositivo + permisos):
  - `device.status`, `device.info`, `device.permissions`, `device.health`
  - `notifications.list`, `notifications.actions` (consulta [Reenvío de notificaciones](#reenvío-de-notificaciones) abajo)
  - `photos.latest`
  - `contacts.search`, `contacts.add`
  - `calendar.events`, `calendar.add`
  - `callLog.search`
  - `sms.search`
  - `motion.activity`, `motion.pedometer`

## Puntos de entrada del asistente

Android admite iniciar OpenClaw desde el disparador del asistente del sistema (Google
Assistant). Cuando está configurado, mantener pulsado el botón de inicio o decir "Hey Google, ask
OpenClaw..." abre la aplicación y pasa el prompt al compositor del chat.

Esto usa metadatos de **App Actions** de Android declarados en el manifiesto de la aplicación. No
se necesita configuración adicional en el lado del gateway; el intent del asistente lo gestiona completamente la aplicación de Android y se reenvía como un mensaje de chat normal.

<Note>
La disponibilidad de App Actions depende del dispositivo, de la versión de Google Play Services
y de si el usuario ha establecido OpenClaw como la aplicación de asistente predeterminada.
</Note>

## Reenvío de notificaciones

Android puede reenviar notificaciones del dispositivo al gateway como eventos. Hay varios controles para limitar qué notificaciones se reenvían y cuándo.

| Clave                              | Tipo           | Descripción                                                                                       |
| ---------------------------------- | -------------- | ------------------------------------------------------------------------------------------------- |
| `notifications.allowPackages`      | string[]       | Solo reenvía notificaciones de estos nombres de paquete. Si se establece, todos los demás paquetes se ignoran. |
| `notifications.denyPackages`       | string[]       | Nunca reenvía notificaciones de estos nombres de paquete. Se aplica después de `allowPackages`. |
| `notifications.quietHours.start`   | string (HH:mm) | Inicio de la ventana de horas silenciosas (hora local del dispositivo). Las notificaciones se suprimen durante esta ventana. |
| `notifications.quietHours.end`     | string (HH:mm) | Fin de la ventana de horas silenciosas.                                                           |
| `notifications.rateLimit`          | number         | Número máximo de notificaciones reenviadas por paquete por minuto. Las notificaciones en exceso se descartan. |

El selector de notificaciones también usa un comportamiento más seguro para eventos de notificaciones reenviadas, evitando el reenvío accidental de notificaciones sensibles del sistema.

Ejemplo de configuración:

```json5
{
  notifications: {
    allowPackages: ["com.slack", "com.whatsapp"],
    denyPackages: ["com.android.systemui"],
    quietHours: {
      start: "22:00",
      end: "07:00",
    },
    rateLimit: 5,
  },
}
```

<Note>
El reenvío de notificaciones requiere el permiso Android Notification Listener. La aplicación lo solicita durante la configuración.
</Note>
