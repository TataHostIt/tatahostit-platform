# CI/CD Stack Setup Guide: ARC & Harbor

**Purpose:** This document details the complete setup of the Actions Runner Controller (ARC) and Harbor Registry integration for the `TataHostIT` platform.
**Architecture:**

* **Controller:** Runs in `arc-systems` namespace.
* **Runners:** Run in `cicd` namespace (Isolated).
* **Registry:** Internal Harbor (caches Docker Hub & stores build layers).
* **Authentication:** GitHub App (for Runners) & Robot Account (for Harbor).

---

## Phase 1: GitHub Configuration (The App)

We use a **GitHub App** instead of a Personal Access Token (PAT) for higher rate limits and independence from user accounts.

1. **Create the App:**
* Go to **Organization Settings** -> **Developer settings** -> **GitHub Apps**.
* Click **New GitHub App**.
* **Name:** `arc-runner-system-tatahostit`
* **Homepage URL:** `https://github.com/actions/actions-runner-controller`
* **Webhook:** Uncheck **Active** (Not needed for Scale Sets).


2. **Set Permissions:**
* **Repository Permissions:**
* `Actions`: **Read-only**
* `Metadata`: **Read-only**


* **Organization Permissions:**
* `Self-hosted runners`: **Read and write**




3. **Install & Generate Credentials:**
* Click **Create GitHub App**.
* Note the **App ID** (at the top of the page).
* Scroll down and click **Generate a private key**. Save the `.pem` file immediately.
* Go to **Install App** (left sidebar) -> **Install** on `TataHostIT`.
* Select **All repositories**.



---

## Phase 2: Harbor Configuration (The Cache)

We configure Harbor to act as a "Pull-Through Cache" for Docker Hub to speed up builds and avoid rate limits.

1. **Configure Docker Hub Proxy:**
* Login to Harbor UI as Admin.
* **Administration** -> **Registries** -> **New Endpoint**.
* Provider: `Docker Hub`
* Name: `dockerhub-upstream`
* URL: `https://hub.docker.com`


* **Projects** -> **New Project**.
* Name: `dockerhub-proxy`
* Access Level: **Private**
* **Proxy Cache:** Toggle **ON**.
* Registry: `dockerhub-upstream`.




2. **Configure Build Destination:**
* **Projects** -> **New Project**.
* Name: `library` (or use default).
* Access Level: **Private**.




3. **Create CI/CD Robot Account:**
* **Administration** -> **Robot Accounts** -> **New Robot Account**.
* **Name:** `cicd-runner-bot`
* **Expiration:** Never.
* **Permissions:**
* `dockerhub-proxy`: List, Pull.
* `library`: List, Pull, Push, Delete.


* **SAVE THE SECRET.** (You will need this for the Kubernetes secret).



---

## Phase 3: Secrets Preparation

You must create these secrets locally and then encrypt them using `kubeseal` before committing to Git.

### 1. The GitHub App Secret

Create a file named `arc-secrets.yaml` (do not commit this version):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: arc-gha-token
  namespace: cicd   # Must be in the runner namespace
type: Opaque
stringData:
  github_app_id: "YOUR_APP_ID_HERE"
  github_app_installation_id: "" # Optional, ARC finds it automatically
  github_app_private_key: |
    -----BEGIN RSA PRIVATE KEY-----
    YOUR_PEM_FILE_CONTENTS_HERE
    -----END RSA PRIVATE KEY-----

```

### 2. The Harbor Docker Config Secret

Create a file named `docker-config.yaml` (do not commit this version):

*First, generate your auth string:*

```bash
# Replace with your Robot Name and Secret
echo -n "robot\$cicd-runner-bot:YOUR_LONG_SECRET_KEY" | base64

```

*Then paste that string into the file:*

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: arc-docker-config
  namespace: cicd   # Must be in the runner namespace
type: Opaque
stringData:
  config.json: |
    {
      "auths": {
        "harbor.tatahostit.com": {
          "auth": "PASTE_YOUR_BASE64_STRING_HERE"
        }
      }
    }

```

### 3. Seal the Secrets

Run these commands to generate the safe files for Git:

```bash
kubeseal --format=yaml < arc-secrets.yaml > infra/cicd/arc/arc-secrets-sealed.yaml
kubeseal --format=yaml < docker-config.yaml > infra/cicd/arc/docker-config-sealed.yaml

```

---

## Phase 4: Git Repository Files

Ensure your `infra` directory has the following structure and content.

### File Structure

```text
infra/
├── namespaces/
│   └── cicd.yaml
└── cicd/
    └── arc/
        ├── arc-app.yaml
        ├── arc-secrets-sealed.yaml
        ├── docker-config-sealed.yaml
        ├── controller/
        │   ├── Chart.yaml
        │   └── values.yaml
        └── runners/
            └── runner-scale-set.yaml

```

### 1. `infra/namespaces/cicd.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cicd
  labels:
    name: cicd

```

### 2. `infra/cicd/arc/arc-app.yaml`

The ArgoCD Application that manages the stack.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: arc-stack
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/TataHostIT/tatahostit-platform.git'
    targetRevision: HEAD
    path: infra/cicd/arc
    directory:
      recurse: true
      exclude: '*-secret.yaml' # Safety: Ignore unsealed secrets if accidentally added
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: arc-systems
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true

```

### 3. `infra/cicd/arc/controller/Chart.yaml`

```yaml
apiVersion: v2
name: arc-controller
version: 0.1.0
dependencies:
  - name: gha-runner-scale-set-controller
    version: 0.9.3
    repository: oci://ghcr.io/actions/actions-runner-controller-charts

```

### 4. `infra/cicd/arc/controller/values.yaml`

```yaml
gha-runner-scale-set-controller:
  replicaCount: 1

```

### 5. `infra/cicd/arc/runners/runner-scale-set.yaml`

This defines the actual build workers. It includes the DNS patch for internal resolution and the Docker config mount.

```yaml
apiVersion: actions.github.com/v1alpha1
kind: AutoscalingRunnerSet
metadata:
  name: tatahostit-runners
  namespace: cicd
spec:
  githubConfigUrl: "https://github.com/TataHostIT"
  
  # Secret references (Must match the sealed secrets above)
  githubConfigSecret: arc-gha-token
  githubConfigSecretRef:
    github_app_id: github_app_id
    github_app_private_key: github_app_private_key

  minRunners: 0
  maxRunners: 5

  template:
    spec:
      # FIX: Force internal DNS resolution for Harbor
      hostAliases:
        - ip: "10.43.x.x"  # REPLACE WITH: kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.spec.clusterIP}'
          hostnames:
            - "harbor.tatahostit.com"

      containers:
        - name: runner
          image: ghcr.io/actions/actions-runner:latest
          command: ["/home/runner/run.sh"]
          env:
            - name: DOCKER_CONFIG
              value: /home/runner/.docker
          volumeMounts:
            - name: docker-config
              mountPath: /home/runner/.docker
              readOnly: true
            - name: work
              mountPath: /work
        - name: dind
          image: docker:dind
          securityContext:
            privileged: true
          volumeMounts:
            - name: work
              mountPath: /work
            - name: docker-config
              mountPath: /root/.docker
              readOnly: true
      volumes:
        - name: work
          emptyDir: {}
        - name: docker-config
          secret:
            secretName: arc-docker-config
            items:
              - key: config.json
                path: config.json

```

---

## Phase 5: Usage (GitHub Workflow)

In your application repositories, use this workflow structure. You do **not** need `docker login` steps because the runner is pre-authenticated via the volume mount.

**File:** `.github/workflows/build.yaml`

```yaml
name: Build
on: [push]

jobs:
  build:
    runs-on: tatahostit-runners # Matches AutoscalingRunnerSet name
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
        with:
          driver-opts: network=host

      - name: Build and Push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          # Pull FROM the proxy, Push TO the library
          tags: harbor.tatahostit.com/library/my-app:${{ github.sha }}
          cache-from: type=registry,ref=harbor.tatahostit.com/library/build-cache:latest
          cache-to: type=registry,ref=harbor.tatahostit.com/library/build-cache:latest,mode=max

```