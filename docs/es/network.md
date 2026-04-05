---
read_when:
    - Necesitas la visión general de la arquitectura de red y seguridad
    - Estás depurando acceso local frente a tailnet o emparejamiento
    - Quieres la lista canónica de la documentación de red
summary: 'Centro de red: superficies de gateway, emparejamiento, descubrimiento y seguridad'
title: Red
x-i18n:
    generated_at: "2026-04-05T12:46:56Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4a5f39d4f40ad19646d372000c85b663770eae412af91e1c175eb27b22208118
    source_path: network.md
    workflow: 15
---

# Centro de red

Este centro enlaza la documentación principal sobre cómo OpenClaw conecta, empareja y protege
dispositivos a través de localhost, LAN y tailnet.

## Modelo principal

La mayoría de las operaciones pasan por la Gateway (`openclaw gateway`), un único proceso de larga duración que gestiona las conexiones de canales y el plano de control WebSocket.

- **Loopback primero**: la WS de la Gateway usa por defecto `ws://127.0.0.1:18789`.
  Los enlaces fuera de loopback requieren una ruta de autenticación válida de gateway: autenticación
  con token/contraseña de secreto compartido, o un despliegue `trusted-proxy`
  fuera de loopback configurado correctamente.
- **Se recomienda una Gateway por host**. Para aislamiento, ejecuta varias gateways con perfiles y puertos aislados ([Multiple Gateways](/gateway/multiple-gateways)).
- **Canvas host** se sirve en el mismo puerto que la Gateway (`/__openclaw__/canvas/`, `/__openclaw__/a2ui/`), protegido por la autenticación de Gateway cuando se enlaza fuera de loopback.
- **El acceso remoto** suele hacerse mediante túnel SSH o VPN Tailscale ([Acceso remoto](/gateway/remote)).

Referencias clave:

- [Arquitectura de Gateway](/concepts/architecture)
- [Protocolo de Gateway](/gateway/protocol)
- [Runbook de Gateway](/gateway)
- [Superficies web + modos de enlace](/web)

## Emparejamiento + identidad

- [Resumen de emparejamiento (DM + nodos)](/channels/pairing)
- [Emparejamiento de nodos gestionado por Gateway](/gateway/pairing)
- [CLI de Devices (emparejamiento + rotación de tokens)](/cli/devices)
- [CLI de Pairing (aprobaciones de DM)](/cli/pairing)

Confianza local:

- Las conexiones directas de loopback local pueden aprobarse automáticamente para el emparejamiento, con el fin de mantener una experiencia fluida en el mismo host.
- OpenClaw también tiene una ruta limitada de autoconexión backend/contenedor-local para flujos helper de secreto compartido de confianza.
- Los clientes de tailnet y LAN, incluidos los enlaces de tailnet en el mismo host, siguen requiriendo aprobación explícita de emparejamiento.

## Descubrimiento + transportes

- [Descubrimiento y transportes](/gateway/discovery)
- [Bonjour / mDNS](/gateway/bonjour)
- [Acceso remoto (SSH)](/gateway/remote)
- [Tailscale](/gateway/tailscale)

## Nodos + transportes

- [Resumen de nodos](/nodes)
- [Protocolo de bridge (nodos heredados, histórico)](/gateway/bridge-protocol)
- [Runbook de nodo: iOS](/platforms/ios)
- [Runbook de nodo: Android](/platforms/android)

## Seguridad

- [Resumen de seguridad](/gateway/security)
- [Referencia de configuración de Gateway](/gateway/configuration)
- [Solución de problemas](/gateway/troubleshooting)
- [Doctor](/gateway/doctor)
