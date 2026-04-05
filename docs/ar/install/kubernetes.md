---
read_when:
    - تريد تشغيل OpenClaw على عنقود Kubernetes
    - تريد اختبار OpenClaw في بيئة Kubernetes
summary: نشر OpenClaw Gateway على عنقود Kubernetes باستخدام Kustomize
title: Kubernetes
x-i18n:
    generated_at: "2026-04-05T12:47:27Z"
    model: gpt-5.4
    provider: openai
    source_hash: aa39127de5a5571f117db3a1bfefd5815b5e6b594cc1df553e30fda882b2a408
    source_path: install/kubernetes.md
    workflow: 15
---

# OpenClaw على Kubernetes

نقطة بداية دنيا لتشغيل OpenClaw على Kubernetes — وليست نشرًا جاهزًا للإنتاج. وهي تغطي الموارد الأساسية والمقصود أن تُكيَّف مع بيئتك.

## لماذا ليس Helm؟

OpenClaw عبارة عن حاوية واحدة مع بعض ملفات التكوين. أما التخصيص المثير للاهتمام فهو في محتوى الوكيل (ملفات markdown وSkills وتجاوزات التكوين)، وليس في templating البنية التحتية. يتعامل Kustomize مع overlays من دون الحمل الزائد لمخطط Helm. وإذا أصبح النشر لديك أكثر تعقيدًا، فيمكن إضافة مخطط Helm فوق هذه manifests.

## ما الذي تحتاجه

- عنقود Kubernetes يعمل (AKS أو EKS أو GKE أو k3s أو kind أو OpenShift أو غيرها)
- `kubectl` متصل بالعنقود لديك
- مفتاح API لمزوّد نماذج واحد على الأقل

## بدء سريع

```bash
# Replace with your provider: ANTHROPIC, GEMINI, OPENAI, or OPENROUTER
export <PROVIDER>_API_KEY="..."
./scripts/k8s/deploy.sh

kubectl port-forward svc/openclaw 18789:18789 -n openclaw
open http://localhost:18789
```

استرجع السر المشترك المكوَّن لـ Control UI. ينشئ نص النشر هذا
مصادقة قائمة على الرمز افتراضيًا:

```bash
kubectl get secret openclaw-secrets -n openclaw -o jsonpath='{.data.OPENCLAW_GATEWAY_TOKEN}' | base64 -d
```

لأغراض التصحيح المحلية، يطبع `./scripts/k8s/deploy.sh --show-token` الرمز بعد النشر.

## الاختبار المحلي باستخدام Kind

إذا لم يكن لديك عنقود، فأنشئ واحدًا محليًا باستخدام [Kind](https://kind.sigs.k8s.io/):

```bash
./scripts/k8s/create-kind.sh           # auto-detects docker or podman
./scripts/k8s/create-kind.sh --delete  # tear down
```

ثم انشر كالمعتاد باستخدام `./scripts/k8s/deploy.sh`.

## خطوة بخطوة

### 1) النشر

**الخيار A** — مفتاح API في البيئة (خطوة واحدة):

```bash
# Replace with your provider: ANTHROPIC, GEMINI, OPENAI, or OPENROUTER
export <PROVIDER>_API_KEY="..."
./scripts/k8s/deploy.sh
```

ينشئ النص Kubernetes Secret بمفتاح API ورمز gateway مُولَّد تلقائيًا، ثم ينشر. وإذا كان Secret موجودًا بالفعل، فإنه يحتفظ برمز gateway الحالي وأي مفاتيح مزوّدات لا يجري تغييرها.

**الخيار B** — إنشاء السر بشكل منفصل:

```bash
export <PROVIDER>_API_KEY="..."
./scripts/k8s/deploy.sh --create-secret
./scripts/k8s/deploy.sh
```

استخدم `--show-token` مع أي من الأمرين إذا كنت تريد طباعة الرمز إلى stdout للاختبار المحلي.

### 2) الوصول إلى gateway

```bash
kubectl port-forward svc/openclaw 18789:18789 -n openclaw
open http://localhost:18789
```

## ما الذي يتم نشره

```
Namespace: openclaw (configurable via OPENCLAW_NAMESPACE)
├── Deployment/openclaw        # Single pod, init container + gateway
├── Service/openclaw           # ClusterIP on port 18789
├── PersistentVolumeClaim      # 10Gi for agent state and config
├── ConfigMap/openclaw-config  # openclaw.json + AGENTS.md
└── Secret/openclaw-secrets    # Gateway token + API keys
```

## التخصيص

### تعليمات الوكيل

حرّر `AGENTS.md` في `scripts/k8s/manifests/configmap.yaml` ثم أعد النشر:

```bash
./scripts/k8s/deploy.sh
```

### تكوين gateway

حرّر `openclaw.json` في `scripts/k8s/manifests/configmap.yaml`. راجع [Gateway configuration](/gateway/configuration) للحصول على المرجع الكامل.

### إضافة مزوّدين

أعد التشغيل مع تصدير مفاتيح إضافية:

```bash
export ANTHROPIC_API_KEY="..."
export OPENAI_API_KEY="..."
./scripts/k8s/deploy.sh --create-secret
./scripts/k8s/deploy.sh
```

تظل مفاتيح المزوّدات الحالية في Secret ما لم تستبدلها.

أو رقّع Secret مباشرةً:

```bash
kubectl patch secret openclaw-secrets -n openclaw \
  -p '{"stringData":{"<PROVIDER>_API_KEY":"..."}}'
kubectl rollout restart deployment/openclaw -n openclaw
```

### مساحة أسماء مخصصة

```bash
OPENCLAW_NAMESPACE=my-namespace ./scripts/k8s/deploy.sh
```

### صورة مخصصة

حرّر الحقل `image` في `scripts/k8s/manifests/deployment.yaml`:

```yaml
image: ghcr.io/openclaw/openclaw:latest # or pin to a specific version from https://github.com/openclaw/openclaw/releases
```

### الكشف خارج port-forward

تربط manifests الافتراضية gateway بعنوان loopback داخل pod. وهذا يعمل مع `kubectl port-forward`، لكنه لا يعمل مع مسار Kubernetes `Service` أو Ingress يحتاج إلى الوصول إلى IP الخاص بالـ pod.

إذا كنت تريد كشف gateway عبر Ingress أو موازن حمل:

- غيّر ربط gateway في `scripts/k8s/manifests/configmap.yaml` من `loopback` إلى ربط غير loopback يطابق نموذج النشر لديك
- أبقِ مصادقة gateway مفعلة واستخدم نقطة دخول صحيحة مع إنهاء TLS
- اضبط Control UI للوصول البعيد باستخدام نموذج أمان الويب المدعوم (مثل HTTPS/Tailscale Serve وقوائم allowed origins صريحة عند الحاجة)

## إعادة النشر

```bash
./scripts/k8s/deploy.sh
```

يطبّق هذا جميع manifests ويعيد تشغيل pod لالتقاط أي تغييرات في التكوين أو الأسرار.

## الإزالة

```bash
./scripts/k8s/deploy.sh --delete
```

يحذف هذا مساحة الأسماء وجميع الموارد الموجودة فيها، بما في ذلك PVC.

## ملاحظات معمارية

- ترتبط gateway بعنوان loopback داخل pod افتراضيًا، لذا فإن الإعداد المضمّن مخصص لـ `kubectl port-forward`
- لا توجد موارد على مستوى العنقود — كل شيء يعيش داخل مساحة أسماء واحدة
- الأمان: `readOnlyRootFilesystem`، وإسقاط قدرات `drop: ALL`، ومستخدم غير root ‏(UID 1000)
- يبقي التكوين الافتراضي Control UI على مسار الوصول المحلي الأكثر أمانًا: ربط loopback بالإضافة إلى `kubectl port-forward` إلى `http://127.0.0.1:18789`
- إذا تجاوزت الوصول من localhost، فاستخدم النموذج البعيد المدعوم: HTTPS/Tailscale مع ربط gateway المناسب وإعدادات origin الخاصة بـ Control UI
- تُولَّد الأسرار في دليل مؤقت وتُطبّق مباشرةً على العنقود — ولا تُكتب أي مواد سرية إلى نسخة المستودع المحلية

## بنية الملفات

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
