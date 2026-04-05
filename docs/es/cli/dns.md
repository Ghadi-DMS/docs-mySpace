---
read_when:
    - Quieres descubrimiento de área amplia (DNS-SD) mediante Tailscale + CoreDNS
    - You’re setting up split DNS for a custom discovery domain (example: openclaw.internal)
summary: Referencia de CLI para `openclaw dns` (ayudantes de descubrimiento de área amplia)
title: dns
x-i18n:
    generated_at: "2026-04-05T12:37:48Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4831fbb7791adfed5195bc4ba36bb248d2bc8830958334211d3c96f824617927
    source_path: cli/dns.md
    workflow: 15
---

# `openclaw dns`

Ayudantes de DNS para descubrimiento de área amplia (Tailscale + CoreDNS). Actualmente centrado en macOS + CoreDNS de Homebrew.

Relacionado:

- Descubrimiento de Gateway: [Discovery](/gateway/discovery)
- Configuración de descubrimiento de área amplia: [Configuration](/gateway/configuration)

## Configuración

```bash
openclaw dns setup
openclaw dns setup --domain openclaw.internal
openclaw dns setup --apply
```

## `dns setup`

Planifica o aplica la configuración de CoreDNS para descubrimiento DNS-SD unicast.

Opciones:

- `--domain <domain>`: dominio de descubrimiento de área amplia (por ejemplo `openclaw.internal`)
- `--apply`: instala o actualiza la configuración de CoreDNS y reinicia el servicio (requiere sudo; solo macOS)

Qué muestra:

- dominio de descubrimiento resuelto
- ruta del archivo de zona
- IPs actuales de tailnet
- configuración recomendada de descubrimiento en `openclaw.json`
- los valores de nameserver/domain de DNS dividido de Tailscale que debes establecer

Notas:

- Sin `--apply`, el comando es solo una ayuda de planificación e imprime la configuración recomendada.
- Si se omite `--domain`, OpenClaw usa `discovery.wideArea.domain` de la configuración.
- `--apply` actualmente solo es compatible con macOS y espera CoreDNS de Homebrew.
- `--apply` inicializa el archivo de zona si es necesario, garantiza que exista la estrofa `import` de CoreDNS y reinicia el servicio brew `coredns`.
