# ☁️ Azure Container Registry (ACR) — Setup & Push Docker Image

**(With Required Fields & Production Notes)**

---



---

## ⚙️ Prerequisites (Required)

| Requirement                | Mandatory      |
| -------------------------- | -------------- |
| Azure Subscription         | ✅ Yes          |
| Azure CLI (`az`)           | ✅ Yes          |
| Docker Engine              | ✅ Yes          |
| Git                        | ✅ Yes          |
| Azure VM (Linux)           | ⚠️ Recommended |
| A Dockerfile-based project | ✅ Yes          |

---

## 🏗️ Step 1 — Create Azure Container Registry (ACR)

### Required Fields (Portal / CLI)

| Field          | Required    | Example          |
| -------------- | ----------- | ---------------- |
| Registry name  | ✅ Yes       | `devops`     |
| Resource group | ✅ Yes       | `devops-rg`         |
| Location       | ✅ Yes       | `eastus`         |
| SKU            | ✅ Yes       | `Basic`          |
| Admin user     | ⚠️ Optional | Enabled for labs |

> 🔐 **Production note**: Admin user is acceptable for labs.
> In production, use **Managed Identity (AKS → ACR attach)** instead.

### Portal Steps

1. Azure Portal → **Container Registries**
2. Click **Create**
3. Fill:

   * **Registry name:** `devops`
   * **Resource group:** `devops-rg`
   * **Region:** `East US`
   * **SKU:** `Basic`
   * **Admin user:** Enable
4. **Review + Create**

---

## 🖥️ Step 2 — Connect to Azure VM

```bash
ssh -i yourkey.pem azureuser@<vm-public-ip>
```

### Install Required Packages

```bash
sudo apt update -y
sudo apt install docker.io git -y
sudo systemctl start docker
sudo systemctl enable docker
```

Verify Docker:

```bash
docker --version
```

---

## 🧰 Step 3 — Install Azure CLI (Required)

```bash
sudo apt-get update
sudo apt-get install azure-cli -y
```

Verify:

```bash
az version
```

Login:

```bash
az login
```

(Optional – if multiple subscriptions)

```bash
az account set --subscription <SUBSCRIPTION_ID>
```

---

## 🔑 Step 4 — Login to ACR (Required)

### Get Credentials

Azure Portal → **ACR → Access keys**

Required values:

* **Login server:** `devops.azurecr.io`
* **Username**
* **Password**

### Docker Login

```bash
sudo docker login devops.azurecr.io
```

✔ Required to push images
✔ Only needed on build machine

---

## 📦 Step 5 — Clone Docker Project

```bash
git clone https://github.com/dhirajnimkande23/docker-hello-world.git
cd docker-hello-world
```

Verify Dockerfile exists:

```bash
ls
```

---

## 🧱 Step 6 — Build & Tag Docker Image (Required Format)

### Required Image Naming Convention (ACR)

```
<registry-name>.azurecr.io/<repository>/<image>:<tag>
```

### Build Image

```bash
sudo docker build -t devops.azurecr.io/cloudnautic/hello-world:latest .
```

Verify:

```bash
sudo docker images
```

---

## ☁️ Step 7 — Push Image to ACR (Required)

```bash
sudo docker push devops.azurecr.io/cloudnautic/hello-world:latest
```

✔ Image uploaded to Azure
✔ Ready for AKS consumption

---

## ✅ Step 8 — Verify in Azure Portal

Navigate to:

**Azure Portal → Container Registry → Repositories**

You should see:

```
cloudnautic/hello-world
```

With tag:

```
latest
```

---

## 🧾 ACR Summary (Required Fields Recap)

| Step                      | Required |
| ------------------------- | -------- |
| ACR created               | ✅        |
| Registry name unique      | ✅        |
| Docker login to ACR       | ✅        |
| Image tagged with ACR URL | ✅        |
| Image pushed successfully | ✅        |

---

# ☁️ AKS + ACR Deployment Guide

**(With Required Fields & Correct Order)**

---

## 🔧 1. Create Resource Group (Required)

```bash
az group create \
  --name devops \
  --location eastus
```

---

## 🚀 2. Create AKS Cluster (Required Fields)

### Minimum Required (Lab)

```bash
az aks create \
  --resource-group devops-rg \
  --name mycluster \
  --node-count 1 \
  --enable-managed-identity \
  --generate-ssh-keys
```

### Recommended (Production-Aligned)

```bash
az aks create \
  --resource-group devops-rg \
  --name mycluster \
  --node-count 1 \
  --node-vm-size Standard_DS2_v2 \
  --enable-managed-identity \
  --enable-addons monitoring \
  --ssh-access disabled \
  --generate-ssh-keys
```

---

## ⚙️ 3. Install kubectl (Required)

```bash
sudo snap install kubectl --classic
```

Verify:

```bash
kubectl version --client
```

---

## 🔗 4. Connect to AKS (Required)

```bash
az aks get-credentials \
  --resource-group devops-rg \
  --name mycluster \
  --overwrite-existing
```

Verify:

```bash
kubectl get nodes
```

---

## 🧩 5. Attach ACR to AKS (Required for Private Images)

```bash
az aks update \
  --resource-group devops-rg \
  --name mycluster \
  --attach-acr devops
```

✔ Eliminates Docker login inside pods
✔ Uses Managed Identity

---

## 🗂️ 6. Create Namespace (Required for Isolation)

```bash
kubectl create namespace cloudnautic
```

---

## 📦 7. Deploy App from ACR (Required Fields)

### deployment.yaml (Required Fields Highlighted)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-web
  namespace: cloudnautic
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-web
  template:
    metadata:
      labels:
        app: hello-web
    spec:
      containers:
      - name: hello-web
        image: devops.azurecr.io/cloudnautic/hello-world:latest
        ports:
        - containerPort: 80
```

### service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-web-svc
  namespace: cloudnautic
spec:
  type: LoadBalancer
  selector:
    app: hello-web
  ports:
  - port: 80
    targetPort: 80
```

Apply:

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

---

## ✅ 8. Verify Deployment

```bash
kubectl get pods -n cloudnautic
kubectl get svc -n cloudnautic
```

Access:

```
http://<EXTERNAL-IP>
```

---

## 🧹 9. Cleanup (Required to Avoid Cost)

```bash
az group delete \
  --name devops \
  --yes --no-wait
```

---
