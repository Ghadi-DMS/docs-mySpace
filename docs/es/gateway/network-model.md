---
read_when:
    - Quieres una vista concisa del modelo de red del Gateway
summary: Cómo se conectan el Gateway, los nodos y el host de canvas.
title: Modelo de red
x-i18n:
    generated_at: "2026-04-05T12:42:10Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7d02d87f38ee5a9fae228f5028892b192c50b473ab4441bbe0b40ee85a1dd402
    source_path: gateway/network-model.md
    workflow: 15
---

# Modelo de red

> Este contenido se ha integrado en [Network](/network#core-model). Consulta esa página para ver la guía actual.

La mayoría de las operaciones fluyen a través del Gateway (`openclaw gateway`), un único
proceso de larga duración que controla las conexiones de canal y el plano de control WebSocket.

## Reglas principales

- Se recomienda un Gateway por host. Es el único proceso autorizado para controlar la sesión de WhatsApp Web. Para bots de rescate o aislamiento estricto, ejecuta varios gateways con perfiles y puertos aislados. Consulta [Multiple gateways](/gateway/multiple-gateways).
- Primero loopback: el WS del Gateway usa por defecto `ws://127.0.0.1:18789`. El asistente crea autenticación con secreto compartido de forma predeterminada y normalmente genera un token, incluso para loopback. Para acceso no loopback, usa una ruta de autenticación válida del gateway: autenticación con token/contraseña de secreto compartido, o un despliegue `trusted-proxy` no loopback configurado correctamente. Las configuraciones de tailnet/móvil suelen funcionar mejor a través de Tailscale Serve u otro endpoint `wss://` en lugar de `ws://` sin procesar sobre tailnet.
- Los nodos se conectan al WS del Gateway por LAN, tailnet o SSH según sea necesario. El
  puente TCP heredado se ha eliminado.
- El host de canvas se sirve mediante el servidor HTTP del Gateway en el **mismo puerto** que el Gateway (predeterminado `18789`):
  - `/__openclaw__/canvas/`
  - `/__openclaw__/a2ui/`
    Cuando `gateway.auth` está configurado y el Gateway se vincula más allá de loopback, estas rutas están protegidas por la autenticación del Gateway. Los clientes de nodo usan URL de capacidad con alcance de nodo vinculadas a su sesión WS activa. Consulta [Gateway configuration](/gateway/configuration) (`canvasHost`, `gateway`).
- El uso remoto suele hacerse mediante túnel SSH o VPN de tailnet. Consulta [Remote access](/gateway/remote) y [Discovery](/gateway/discovery).
