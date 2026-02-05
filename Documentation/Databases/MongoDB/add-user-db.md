This guide outlines the standard operating procedure for adding a new isolated database and user to the shared Percona MongoDB cluster using the **Percona Operator**, **Sealed Secrets**, and **Kubernetes Reflector**.

# Guide: Adding a New MongoDB Database & User

### Workflow Overview

To maintain "Least Privilege" isolation and GitOps automation, we follow a modular secret-per-app process. Unlike PostgreSQL/CNPG, MongoDB creates databases implicitly when the first user or data is assigned to them.

1. **Wave 0:** Create the plaintext secret with all connection details.
2. **Seal:** Encrypt the secret into a `SealedSecret` for Git.
3. **Wave 5:** Register the user and their database permissions in the cluster manifest.

---

### Step 1: Create the Plaintext Secret

Create a temporary file (e.g., `temp-secret.yaml`) locally. **Do not commit this file to Git.**

* **Namespace:** Must be `mongodb`.
* **Annotations:** Include Reflector settings to mirror the secret to the application's target namespace.

```yaml
# File: temp-secret.yaml (DO NOT COMMIT)
apiVersion: v1
kind: Secret
metadata:
  name: <app-name>-secrets # <--- CHANGE THIS
  namespace: mongodb
  annotations:
    argocd.argoproj.io/sync-wave: "0"
    reflector.v1.k8s.emberstack.com/reflection-allowed: "true"
    reflector.v1.k8s.emberstack.com/reflection-auto-enabled: "true"
    reflector.v1.k8s.emberstack.com/reflection-allowed-namespaces: "<app-namespace>" # <--- CHANGE THIS
type: Opaque
stringData:
  # The 'password' key is used by the Percona Operator for internal DB creation
  password: "your-secure-password" # <--- CHANGE THIS

  # App-facing connection details
  MONGODB_USER: "<app-user-name>" # <--- CHANGE THIS
  MONGODB_DATABASE: "<app-database-name>" # <--- CHANGE THIS
  MONGODB_HOST: "mongodb-cluster-rs0.mongodb.svc.cluster.local"
  MONGODB_PORT: "27017"

  # Full Connection URL (includes authSource=admin)
  # Format: mongodb://<user>:<password>@<host>:<port>/<database>?authSource=admin
  MONGODB_URL: "mongodb://<app-user-name>:your-secure-password@mongodb-cluster-rs0.mongodb.svc.cluster.local:27017/<app-database-name>?authSource=admin" # <--- CHANGE ALL FIELDS

```

---

### Step 2: Seal the Secret

Run the `kubeseal` command to encrypt the credentials. This file is safe to commit to your repository.

```bash
kubeseal --cert infra/sealed-secrets/pub-cert.pem --format yaml \
  < temp-secret.yaml \
  > infra/databases/mongodb/<app-name>-secrets-sealed.yaml

```

*Delete `temp-secret.yaml` after this step.*

---

### Step 3: Register the User in the Cluster

Update the shared cluster manifest to define the user and link them to their dedicated database. The operator will use the `password` key from your secret to create the user inside MongoDB.

**File to update:** `infra/databases/mongodb/mongodb-cluster.yaml`

```yaml
# ... existing cluster config ...
spec:
  users:
    # Add the new user block here
    - name: <app-user-name> # <--- Matches MONGODB_USER in Step 1
      db: admin # Identity is always stored in the admin DB
      passwordSecretRef:
        name: <app-name>-secrets # <--- Matches Secret name from Step 1
        key: password
      roles:
        - name: readWrite
          db: <app-database-name> # <--- The isolated DB this user owns

```

---

### Step 4: Verification

Once you push these changes and ArgoCD syncs:

1. **Check the Secret:** Verify the source secret exists in the `mongodb` namespace.
```bash
kubectl get secret <app-name>-secrets -n mongodb

```


2. **Check Reflection:** Verify Reflector mirrored the secret to the app namespace.
```bash
kubectl get secret <app-name>-secrets -n <app-namespace>

```


3. **Check User Creation:** The Percona Operator will log successful user creation in its pod logs. You can also verify by connecting with your `clusterAdmin` credentials.

---

### Standard Connection Pattern
Applications should be configured to use the mirrored secret. The easiest way is to bind the MONGODB_URL directly to an environment variable:

```yaml
# Application Deployment Example
env:
  - name: MONGO_CONNECTION_STRING
    valueFrom:
      secretKeyRef:
        name: <app-name>-secrets
        key: MONGODB_URL
```
---
### Summary of Rules

* **Auth Source:** MongoDB users are authenticated against the `admin` database, regardless of which database they have permission to access.
* **Isolation:** By specifying the `db` name under `roles`, the user is restricted only to that database and cannot see others.
* **URL Maintenance:** If you update the password, you **must** update both the `password` field and the password inside the `MONGODB_URL` string before re-sealing.

---
