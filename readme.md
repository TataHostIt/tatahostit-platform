# TataHostIT Platform (Infrastructure as Code)

This repository contains the complete GitOps state for the **TataHostIT** Kubernetes platform. It manages the entire infrastructure stack (Storage, Networking, Monitoring) using **ArgoCD** running on a **Talos Linux** cluster.

## 🏗 Architecture

  * **OS:** Talos Linux (Proxmox VMs)
  * **GitOps:** ArgoCD (App of Apps pattern)
  * **Storage:** Ceph (via CSI Driver to external Proxmox cluster)
  * **Networking:** MetalLB (L2 Mode) + NGINX Ingress
  * **Secrets:** Bitnami Sealed Secrets

-----

## 🛠 Prerequisites

Before interacting with the cluster, ensure you have these CLI tools installed on your workstation:

1.  **kubectl** (Kubernetes CLI)
2.  **talosctl** (OS Management)
3.  **kubeseal** (Secret Encryption)
      * *Mac:* `brew install kubeseal`

-----

## 🚀 Bootstrap Guide (Start Here)

These steps assume you have a running Talos Cluster and `kubectl` access.

### Step 1: Install Core Controllers

We need to manually install the "engines" that drive the automation (ArgoCD and Sealed Secrets) because they cannot manage themselves until they exist.

```bash
# 1. Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 2. Install Sealed Secrets Controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.33.1/controller.yaml
```

### Step 2: Configure Encryption Keys (CRITICAL)

**⚠️ STOP AND CHOOSE ONE PATH:**

#### Path A: Brand New Cluster (First Time Setup)

If this is the first time you are building this cluster, you need to back up the auto-generated encryption key. If you lose this key, **all secrets in this repo become unreadable forever.**

```bash
# Wait for the controller to start
kubectl rollout status deployment/sealed-secrets-controller -n kube-system

# BACKUP THE MASTER KEY (Save this to 1Password/Bitwarden)
kubectl get secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > master-key-backup.yaml
```

#### Path B: Disaster Recovery (Reinstalling Existing Cluster)

If you wiped your cluster and are rebuilding it, you must restore your old master key so that the existing secrets in this Git repo can be decrypted.

```bash
# 1. Apply your backed-up master key
kubectl apply -f master-key-backup.yaml

# 2. Restart the controller so it picks up the old key
kubectl delete pod -n kube-system -l name=sealed-secrets-controller

# Get new password for argocd.
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

# Open argocd in browser.
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### Step 3: Trigger GitOps

Once the controllers are running and the keys are set, tell ArgoCD to take over.

```bash
kubectl apply -f bootstrap/root-app.yaml
```

*ArgoCD will now pull this repo, verify the structure, and deploy Ceph, MetalLB, NGINX, and all applications automatically.*

-----

## 🔐 Secrets Management Workflow

We **never** commit raw passwords or keys to GitHub. We use **Sealed Secrets**.
A Sealed Secret is an encrypted blob that is safe to make public. Only the controller inside the cluster (which holds the Private Key) can decrypt it.

### How to Add a New Secret

**Example:** Adding a database password.

1.  **Generate the dry-run secret (Local plain text):**

      * *Note: Do not execute this against the cluster, just output to a file.*

    <!-- end list -->

    ```bash
    kubectl create secret generic my-db-pass \
      --namespace test-apps \
      --from-literal=password=SuperSecret123 \
      --dry-run=client -o yaml > plain-secret.yaml
    ```

2.  **Seal (Encrypt) the secret:**

      * This uses the public key from the cluster to encrypt the file.

    <!-- end list -->

    ```bash
    kubeseal --format=yaml < plain-secret.yaml > sealed-secret.yaml
    ```

3.  **Commit the Sealed Secret:**

      * Move `sealed-secret.yaml` into the repo (e.g., `apps/myapp/secret.yaml`).
      * `git add . && git commit -m "Add db secret" && git push`

4.  **Clean Up:**

      * **IMMEDIATELY** delete `plain-secret.yaml`.

-----

## 📂 Repository Structure

```text
tatahostit-platform/
├── bootstrap/               # Entry point for ArgoCD
│   └── root-app.yaml        # The "App of Apps" configuration
├── infra/                   # Core Infrastructure (Platform Layer)
│   ├── namespaces/          # Namespace definitions
│   ├── networking/          # MetalLB, Ingress-Nginx, Cert-Manager
│   └── storage/             # Ceph CSI, StorageClasses
└── apps/                    # User Applications (Workload Layer)
    └── podinfo/             # Example Test App
```

## 🌐 Network Logic

  * **Internal Access:** `http://*.tatahostit.local` resolves to the MetalLB VIP (e.g., `10.20.20.200`).
  * **DNS:** Update your local DNS or hosts file to point domains to the MetalLB IP.
  * **IP Management:** IPs are reserved in the FS Switch (Excluded from DHCP) and managed explicitly by MetalLB config in `infra/networking/metallb-config.yaml`.