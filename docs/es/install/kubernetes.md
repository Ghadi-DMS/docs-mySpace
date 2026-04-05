---
read_when:
    - Quieres ejecutar OpenClaw en un clúster de Kubernetes
    - Quieres probar OpenClaw en un entorno de Kubernetes
summary: Implementar OpenClaw Gateway en un clúster de Kubernetes con Kustomize
title: Kubernetes
x-i18n:
    generated_at: "2026-04-05T12:45:57Z"
    model: gpt-5.4
    provider: openai
    source_hash: aa39127de5a5571f117db3a1bfefd5815b5e6b594cc1df553e30fda882b2a408
    source_path: install/kubernetes.md
    workflow: 15
---

# OpenClaw en Kubernetes

Un punto de partida mínimo para ejecutar OpenClaw en Kubernetes; no es una implementación lista para producción. Cubre los recursos principales y está pensado para adaptarse a tu entorno.

## ¿Por qué no Helm?

OpenClaw es un único contenedor con algunos archivos de configuración. La personalización interesante está en el contenido del agente (archivos markdown, Skills, anulaciones de configuración), no en las plantillas de infraestructura. Kustomize gestiona overlays sin la sobrecarga de un chart de Helm. Si tu implementación se vuelve más compleja, se puede superponer un chart de Helm sobre estos manifiestos.

## Qué necesitas

- Un clúster de Kubernetes en ejecución (AKS, EKS, GKE, k3s, kind, OpenShift, etc.)
- `kubectl` conectado a tu clúster
- Una clave API para al menos un proveedor de modelos

## Inicio rápido

```bash
# Replace with your provider: ANTHROPIC, GEMINI, OPENAI, or OPENROUTER
export <PROVIDER>_API_KEY="..."
./scripts/k8s/deploy.sh

kubectl port-forward svc/openclaw 18789:18789 -n openclaw
open http://localhost:18789
```

Recupera el secreto compartido configurado para la UI de Control. Este script de implementación
crea autenticación por token de forma predeterminada:

```bash
kubectl get secret openclaw-secrets -n openclaw -o jsonpath='{.data.OPENCLAW_GATEWAY_TOKEN}' | base64 -d
```

Para depuración local, `./scripts/k8s/deploy.sh --show-token` imprime el token tras la implementación.

## Pruebas locales con Kind

Si no tienes un clúster, crea uno localmente con [Kind](https://kind.sigs.k8s.io/):

```bash
./scripts/k8s/create-kind.sh           # auto-detects docker or podman
./scripts/k8s/create-kind.sh --delete  # tear down
```

Después, implementa como de costumbre con `./scripts/k8s/deploy.sh`.

## Paso a paso

### 1) Implementar

**Opción A**: clave API en el entorno (un paso):

```bash
# Replace with your provider: ANTHROPIC, GEMINI, OPENAI, or OPENROUTER
export <PROVIDER>_API_KEY="..."
./scripts/k8s/deploy.sh
```

El script crea un Secret de Kubernetes con la clave API y un token de gateway generado automáticamente, y luego implementa. Si el Secret ya existe, conserva el token de gateway actual y cualquier clave de proveedor que no se esté cambiando.

**Opción B**: crear el secreto por separado:

```bash
export <PROVIDER>_API_KEY="..."
./scripts/k8s/deploy.sh --create-secret
./scripts/k8s/deploy.sh
```

Usa `--show-token` con cualquiera de los dos comandos si quieres que el token se imprima en stdout para pruebas locales.

### 2) Acceder al gateway

```bash
kubectl port-forward svc/openclaw 18789:18789 -n openclaw
open http://localhost:18789
```

## Qué se implementa

```
Namespace: openclaw (configurable via OPENCLAW_NAMESPACE)
├── Deployment/openclaw        # Single pod, init container + gateway
├── Service/openclaw           # ClusterIP on port 18789
├── PersistentVolumeClaim      # 10Gi for agent state and config
└── ConfigMap/openclaw-config  # openclaw.json + AGENTS.md
└── Secret/openclaw-secrets    # Gateway token + API keys
```

## Personalización

### Instrucciones del agente

Edita `AGENTS.md` en `scripts/k8s/manifests/configmap.yaml` y vuelve a implementar:

```bash
./scripts/k8s/deploy.sh
```

### Configuración del gateway

Edita `openclaw.json` en `scripts/k8s/manifests/configmap.yaml`. Consulta [Configuración del gateway](/gateway/configuration) para ver la referencia completa.

### Añadir proveedores

Vuelve a ejecutar con claves adicionales exportadas:

```bash
export ANTHROPIC_API_KEY="..."
export OPENAI_API_KEY="..."
./scripts/k8s/deploy.sh --create-secret
./scripts/k8s/deploy.sh
```

Las claves existentes de proveedores permanecen en el Secret salvo que las sobrescribas.

O modifica el Secret directamente:

```bash
kubectl patch secret openclaw-secrets -n openclaw \
  -p '{"stringData":{"<PROVIDER>_API_KEY":"..."}}'
kubectl rollout restart deployment/openclaw -n openclaw
```

### Namespace personalizado

```bash
OPENCLAW_NAMESPACE=my-namespace ./scripts/k8s/deploy.sh
```

### Imagen personalizada

Edita el campo `image` en `scripts/k8s/manifests/deployment.yaml`:

```yaml
image: ghcr.io/openclaw/openclaw:latest # or pin to a specific version from https://github.com/openclaw/openclaw/releases
```

### Exponer más allá de port-forward

Los manifiestos predeterminados enlazan el gateway a loopback dentro del pod. Eso funciona con `kubectl port-forward`, pero no funciona con una ruta `Service` o Ingress de Kubernetes que necesite alcanzar la IP del pod.

Si quieres exponer el gateway mediante un Ingress o balanceador de carga:

- Cambia el bind del gateway en `scripts/k8s/manifests/configmap.yaml` de `loopback` a un bind no loopback que coincida con tu modelo de implementación
- Mantén habilitada la autenticación del gateway y usa un punto de entrada con terminación TLS adecuado
- Configura la UI de Control para acceso remoto usando el modelo de seguridad web compatible (por ejemplo HTTPS/Tailscale Serve y orígenes permitidos explícitos cuando sea necesario)

## Volver a implementar

```bash
./scripts/k8s/deploy.sh
```

Esto aplica todos los manifiestos y reinicia el pod para recoger cualquier cambio de configuración o secretos.

## Desmontaje

```bash
./scripts/k8s/deploy.sh --delete
```

Esto elimina el namespace y todos los recursos que contiene, incluido el PVC.

## Notas de arquitectura

- El gateway se enlaza a loopback dentro del pod de forma predeterminada, por lo que la configuración incluida es para `kubectl port-forward`
- No hay recursos con alcance de clúster; todo vive en un único namespace
- Seguridad: `readOnlyRootFilesystem`, capacidades `drop: ALL`, usuario no root (UID 1000)
- La configuración predeterminada mantiene la UI de Control en la ruta más segura de acceso local: bind a loopback más `kubectl port-forward` a `http://127.0.0.1:18789`
- Si vas más allá del acceso localhost, usa el modelo remoto compatible: HTTPS/Tailscale más el bind adecuado del gateway y la configuración de origen de la UI de Control
- Los secretos se generan en un directorio temporal y se aplican directamente al clúster; no se escribe material secreto en la copia del repositorio

## Estructura de archivos

```
scripts/k8s/
├── deploy.sh                   # Creates namespace + secret, deploys via kustomize
├── create-kind.sh              # Local Kind cluster (auto-detects docker/podman)
└── manifests/
    ├── kustomization.yaml      # Kustomize base
    ├── configmap.yaml          # openclaw.json + AGENTS.md
    ├── deployment.yaml         # Pod spec with security hardening
    ├── pvc.yaml                # 10Gi persistent storage
    └── service.yaml            # ClusterIP on 18789
```
