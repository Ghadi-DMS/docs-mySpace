---
read_when:
    - تريد تشغيل OpenClaw على مدار الساعة على Azure مع تقوية Network Security Group
    - تريد Gateway دائم التشغيل بدرجة إنتاجية على Azure Linux VM خاص بك
    - تريد إدارة آمنة باستخدام Azure Bastion SSH
summary: تشغيل OpenClaw Gateway على مدار الساعة على Azure Linux VM مع حالة دائمة
title: Azure
x-i18n:
    generated_at: "2026-04-05T12:45:54Z"
    model: gpt-5.4
    provider: openai
    source_hash: dcdcf6dcf5096cd21e1b64f455656f7d77b477d03e9a088db74c6e988c3031db
    source_path: install/azure.md
    workflow: 15
---

# OpenClaw على Azure Linux VM

يشرح هذا الدليل كيفية إعداد Azure Linux VM باستخدام Azure CLI، وتطبيق تقوية Network Security Group ‏(NSG)، وتكوين Azure Bastion للوصول عبر SSH، وتثبيت OpenClaw.

## ما الذي ستقوم به

- إنشاء موارد الشبكة والحوسبة في Azure ‏(VNet، وsubnets، وNSG) باستخدام Azure CLI
- تطبيق قواعد Network Security Group بحيث يُسمح بـ SSH إلى VM من Azure Bastion فقط
- استخدام Azure Bastion للوصول عبر SSH ‏(من دون عنوان IP عام على VM)
- تثبيت OpenClaw باستخدام script التثبيت
- التحقق من Gateway

## ما الذي تحتاج إليه

- اشتراك Azure مع صلاحية إنشاء موارد الحوسبة والشبكة
- تثبيت Azure CLI ‏(راجع [خطوات تثبيت Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli) إذا لزم الأمر)
- زوج مفاتيح SSH ‏(يغطي الدليل كيفية إنشائه إذا لزم الأمر)
- نحو 20-30 دقيقة

## تكوين النشر

<Steps>
  <Step title="سجّل الدخول إلى Azure CLI">
    ```bash
    az login
    az extension add -n ssh
    ```

    امتداد `ssh` مطلوب لنفق SSH الأصلي في Azure Bastion.

  </Step>

  <Step title="سجّل موفري الموارد المطلوبين (مرة واحدة)">
    ```bash
    az provider register --namespace Microsoft.Compute
    az provider register --namespace Microsoft.Network
    ```

    تحقّق من التسجيل. انتظر حتى يعرض كلاهما القيمة `Registered`.

    ```bash
    az provider show --namespace Microsoft.Compute --query registrationState -o tsv
    az provider show --namespace Microsoft.Network --query registrationState -o tsv
    ```

  </Step>

  <Step title="اضبط متغيرات النشر">
    ```bash
    RG="rg-openclaw"
    LOCATION="westus2"
    VNET_NAME="vnet-openclaw"
    VNET_PREFIX="10.40.0.0/16"
    VM_SUBNET_NAME="snet-openclaw-vm"
    VM_SUBNET_PREFIX="10.40.2.0/24"
    BASTION_SUBNET_PREFIX="10.40.1.0/26"
    NSG_NAME="nsg-openclaw-vm"
    VM_NAME="vm-openclaw"
    ADMIN_USERNAME="openclaw"
    BASTION_NAME="bas-openclaw"
    BASTION_PIP_NAME="pip-openclaw-bastion"
    ```

    عدّل الأسماء ونطاقات CIDR بما يناسب بيئتك. يجب أن تكون Bastion subnet بحجم لا يقل عن `/26`.

  </Step>

  <Step title="اختر مفتاح SSH">
    استخدم مفتاحك العام الحالي إذا كان لديك واحد:

    ```bash
    SSH_PUB_KEY="$(cat ~/.ssh/id_ed25519.pub)"
    ```

    إذا لم يكن لديك مفتاح SSH بعد، فأنشئ واحدًا:

    ```bash
    ssh-keygen -t ed25519 -a 100 -f ~/.ssh/id_ed25519 -C "you@example.com"
    SSH_PUB_KEY="$(cat ~/.ssh/id_ed25519.pub)"
    ```

  </Step>

  <Step title="اختر حجم VM وحجم قرص نظام التشغيل">
    ```bash
    VM_SIZE="Standard_B2as_v2"
    OS_DISK_SIZE_GB=64
    ```

    اختر حجم VM وحجم قرص نظام تشغيل متاحين في اشتراكك ومنطقتك:

    - ابدأ بحجم أصغر للاستخدام الخفيف ثم قم بالتوسيع لاحقًا
    - استخدم عددًا أكبر من vCPU/RAM/القرص للأتمتة الأثقل، أو مزيد من القنوات، أو أحمال عمل أكبر للنماذج/الأدوات
    - إذا لم يكن حجم VM متاحًا في منطقتك أو ضمن حصة اشتراكك، فاختر أقرب SKU متاح

    اعرض أحجام VM المتاحة في منطقتك المستهدفة:

    ```bash
    az vm list-skus --location "${LOCATION}" --resource-type virtualMachines -o table
    ```

    تحقّق من استخدامك/حصتك الحالية من vCPU والقرص:

    ```bash
    az vm list-usage --location "${LOCATION}" -o table
    ```

  </Step>
</Steps>

## نشر موارد Azure

<Steps>
  <Step title="أنشئ مجموعة الموارد">
    ```bash
    az group create -n "${RG}" -l "${LOCATION}"
    ```
  </Step>

  <Step title="أنشئ Network Security Group">
    أنشئ NSG وأضف قواعد بحيث يمكن لـ Bastion subnet فقط استخدام SSH إلى VM.

    ```bash
    az network nsg create \
      -g "${RG}" -n "${NSG_NAME}" -l "${LOCATION}"

    # السماح بـ SSH من Bastion subnet فقط
    az network nsg rule create \
      -g "${RG}" --nsg-name "${NSG_NAME}" \
      -n AllowSshFromBastionSubnet --priority 100 \
      --access Allow --direction Inbound --protocol Tcp \
      --source-address-prefixes "${BASTION_SUBNET_PREFIX}" \
      --destination-port-ranges 22

    # منع SSH من الإنترنت العام
    az network nsg rule create \
      -g "${RG}" --nsg-name "${NSG_NAME}" \
      -n DenyInternetSsh --priority 110 \
      --access Deny --direction Inbound --protocol Tcp \
      --source-address-prefixes Internet \
      --destination-port-ranges 22

    # منع SSH من مصادر VNet الأخرى
    az network nsg rule create \
      -g "${RG}" --nsg-name "${NSG_NAME}" \
      -n DenyVnetSsh --priority 120 \
      --access Deny --direction Inbound --protocol Tcp \
      --source-address-prefixes VirtualNetwork \
      --destination-port-ranges 22
    ```

    يتم تقييم القواعد حسب الأولوية (الرقم الأصغر أولًا): يُسمح بحركة Bastion عند 100، ثم يُحظر كل SSH آخر عند 110 و120.

  </Step>

  <Step title="أنشئ الشبكة الافتراضية وsubnets">
    أنشئ VNet مع VM subnet ‏(مع إرفاق NSG)، ثم أضف Bastion subnet.

    ```bash
    az network vnet create \
      -g "${RG}" -n "${VNET_NAME}" -l "${LOCATION}" \
      --address-prefixes "${VNET_PREFIX}" \
      --subnet-name "${VM_SUBNET_NAME}" \
      --subnet-prefixes "${VM_SUBNET_PREFIX}"

    # إرفاق NSG بـ VM subnet
    az network vnet subnet update \
      -g "${RG}" --vnet-name "${VNET_NAME}" \
      -n "${VM_SUBNET_NAME}" --nsg "${NSG_NAME}"

    # AzureBastionSubnet — الاسم مطلوب من Azure
    az network vnet subnet create \
      -g "${RG}" --vnet-name "${VNET_NAME}" \
      -n AzureBastionSubnet \
      --address-prefixes "${BASTION_SUBNET_PREFIX}"
    ```

  </Step>

  <Step title="أنشئ VM">
    لا يملك VM عنوان IP عامًا. ويكون الوصول عبر SSH حصريًا من خلال Azure Bastion.

    ```bash
    az vm create \
      -g "${RG}" -n "${VM_NAME}" -l "${LOCATION}" \
      --image "Canonical:ubuntu-24_04-lts:server:latest" \
      --size "${VM_SIZE}" \
      --os-disk-size-gb "${OS_DISK_SIZE_GB}" \
      --storage-sku StandardSSD_LRS \
      --admin-username "${ADMIN_USERNAME}" \
      --ssh-key-values "${SSH_PUB_KEY}" \
      --vnet-name "${VNET_NAME}" \
      --subnet "${VM_SUBNET_NAME}" \
      --public-ip-address "" \
      --nsg ""
    ```

    يمنع `--public-ip-address ""` تعيين عنوان IP عام. ويتخطى `--nsg ""` إنشاء NSG لكل NIC ‏(إذ تتولى NSG على مستوى subnet الأمان).

    **قابلية إعادة الإنتاج:** يستخدم الأمر أعلاه `latest` لصورة Ubuntu. لتثبيت إصدار محدد، اعرض الإصدارات المتاحة واستبدل `latest`:

    ```bash
    az vm image list \
      --publisher Canonical --offer ubuntu-24_04-lts \
      --sku server --all -o table
    ```

  </Step>

  <Step title="أنشئ Azure Bastion">
    يوفّر Azure Bastion وصول SSH مُدارًا إلى VM من دون كشف عنوان IP عام. ويتطلب SKU من نوع Standard مع tunneling من أجل `az network bastion ssh` المعتمد على CLI.

    ```bash
    az network public-ip create \
      -g "${RG}" -n "${BASTION_PIP_NAME}" -l "${LOCATION}" \
      --sku Standard --allocation-method Static

    az network bastion create \
      -g "${RG}" -n "${BASTION_NAME}" -l "${LOCATION}" \
      --vnet-name "${VNET_NAME}" \
      --public-ip-address "${BASTION_PIP_NAME}" \
      --sku Standard --enable-tunneling true
    ```

    يستغرق تجهيز Bastion عادةً 5-10 دقائق، لكنه قد يستغرق حتى 15-30 دقيقة في بعض المناطق.

  </Step>
</Steps>

## تثبيت OpenClaw

<Steps>
  <Step title="اتصل بـ SSH إلى VM عبر Azure Bastion">
    ```bash
    VM_ID="$(az vm show -g "${RG}" -n "${VM_NAME}" --query id -o tsv)"

    az network bastion ssh \
      --name "${BASTION_NAME}" \
      --resource-group "${RG}" \
      --target-resource-id "${VM_ID}" \
      --auth-type ssh-key \
      --username "${ADMIN_USERNAME}" \
      --ssh-key ~/.ssh/id_ed25519
    ```

  </Step>

  <Step title="ثبّت OpenClaw (داخل shell الخاص بـ VM)">
    ```bash
    curl -fsSL https://openclaw.ai/install.sh -o /tmp/install.sh
    bash /tmp/install.sh
    rm -f /tmp/install.sh
    ```

    يقوم المثبّت بتثبيت Node LTS والتبعيات إذا لم تكن موجودة بالفعل، ويثبّت OpenClaw، ويشغّل معالج onboarding. راجع [التثبيت](/install) للتفاصيل.

  </Step>

  <Step title="تحقّق من Gateway">
    بعد اكتمال onboarding:

    ```bash
    openclaw gateway status
    ```

    تمتلك معظم فرق Azure المؤسسية بالفعل تراخيص GitHub Copilot. وإذا كان هذا ينطبق عليك، فنوصي باختيار مزود GitHub Copilot في معالج onboarding الخاص بـ OpenClaw. راجع [مزود GitHub Copilot](/providers/github-copilot).

  </Step>
</Steps>

## اعتبارات التكلفة

تبلغ تكلفة Azure Bastion Standard SKU نحو **\$140/شهريًا** ويبلغ تشغيل VM ‏(Standard_B2as_v2) نحو **\$55/شهريًا**.

لتقليل التكاليف:

- **قم بإلغاء تخصيص VM** عندما لا يكون قيد الاستخدام (يتوقف احتساب تكلفة الحوسبة؛ وتبقى رسوم القرص). لن يكون OpenClaw Gateway قابلاً للوصول أثناء إلغاء تخصيص VM — أعد تشغيله عندما تحتاج إليه مرة أخرى:

  ```bash
  az vm deallocate -g "${RG}" -n "${VM_NAME}"
  az vm start -g "${RG}" -n "${VM_NAME}"   # أعد التشغيل لاحقًا
  ```

- **احذف Bastion عندما لا تحتاج إليه** وأعد إنشاؤه عندما تحتاج إلى وصول SSH. يُعد Bastion أكبر مكوّن تكلفة ويستغرق توفيره بضع دقائق فقط.
- **استخدم Basic Bastion SKU** ‏(~\$38/شهريًا) إذا كنت تحتاج فقط إلى SSH عبر Portal ولا تحتاج إلى tunneling عبر CLI ‏(`az network bastion ssh`).

## التنظيف

لحذف جميع الموارد التي أنشأها هذا الدليل:

```bash
az group delete -n "${RG}" --yes --no-wait
```

يؤدي ذلك إلى إزالة مجموعة الموارد وكل ما بداخلها (VM وVNet وNSG وBastion وpublic IP).

## الخطوات التالية

- إعداد قنوات المراسلة: [القنوات](/channels)
- إقران الأجهزة المحلية كعقد: [Nodes](/nodes)
- تكوين Gateway: [تكوين Gateway](/gateway/configuration)
- لمزيد من التفاصيل حول نشر OpenClaw على Azure مع مزود نموذج GitHub Copilot: [OpenClaw on Azure with GitHub Copilot](https://github.com/johnsonshi/openclaw-azure-github-copilot)
