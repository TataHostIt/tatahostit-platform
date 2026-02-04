This guide outlines the standard operating procedure for adding a new isolated database and user to the shared PostgreSQL cluster using **CloudNativePG (CNPG)**, **Sealed Secrets**, and **Kubernetes Reflector**.

# Guide: Adding a New Database & User

### Workflow Overview

To maintain "Least Privilege" isolation and GitOps automation, we follow a three-wave sync process:

1. **Wave 0:** Create and seal the credentials.
2. **Wave 1:** Define the role in the shared cluster.
3. **Wave 2:** Provision the database and assign the owner.

---

### Step 1: Create the Plaintext Secret

Create a temporary file (e.g., `temp-secret.yaml`) locally. **Do not commit this file to Git.**

* **Namespace:** Must be `postgres`.
* **Annotations:** Include Reflector settings to mirror the secret to the application's namespace.

```yaml
# File: temp-secret.yaml (DO NOT COMMIT)
apiVersion: v1
kind: Secret
metadata:
  name: <app-name>-db-creds
  namespace: postgres
  annotations:
    argocd.argoproj.io/sync-wave: "0"
    # Allow Reflector to mirror this to the app namespace
    reflector.v1.k8s.emberstack.com/reflection-allowed: "true"
    reflector.v1.k8s.emberstack.com/reflection-auto-enabled: "true"
    reflector.v1.k8s.emberstack.com/reflection-allowed-namespaces: "<app-namespace>"
type: Opaque
stringData:
  username: <app-user-name>
  password: "your-secure-password"

```

---

### Step 2: Seal the Secret

Run the `kubeseal` command to encrypt the credentials. This file is safe to commit to your repository.

```bash
kubeseal --cert infra/sealed-secrets/pub-cert.pem --format yaml \
  < temp-secret.yaml \
  > infra/databases/postgres/<app-name>-secrets-sealed.yaml

```

*Delete `temp-secret.yaml` after this step.*

---

### Step 3: Register the Role in the Cluster

Update the shared cluster manifest to tell CNPG to manage this new role. This ensures the user is created inside the database engine using the password from your Sealed Secret.

**File to update:** `infra/databases/postgres/postgres-cluster.yaml`

```yaml
# ... existing cluster config ...
spec:
  managed:
    roles:
      # Add the new role here
      - name: <app-user-name>
        login: true
        passwordSecret:
          name: <app-name>-db-creds # Matches the Secret name from Step 1

```

---

### Step 4: Provision the Database

Create a new manifest to define the actual database. This links the database to the shared cluster and assigns the new user as the owner.

**New file:** `infra/databases/postgres/<app-name>-db.yaml`

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Database
metadata:
  name: <app-name>-db
  namespace: postgres
  annotations:
    argocd.argoproj.io/sync-wave: "2" # Ensures it runs after the Cluster/Role sync
spec:
  name: <app-database-name>
  owner: <app-user-name> # Matches the role name in Step 3
  cluster:
    name: postgres-shared

```

---

### Step 5: Verification

Once you push these changes and ArgoCD syncs:

1. **Check the Role:** Verify CNPG created the user in the `postgres` namespace.
```bash
kubectl get secrets -n postgres <app-name>-db-creds

```


2. **Check the Reflection:** Verify Reflector copied the secret to your application's namespace.
```bash
kubectl get secrets -n <app-namespace> <app-name>-db-creds

```


3. **Check the DB:** Log into the Postgres primary pod and verify the database exists and is owned by the new user.
```bash
kubectl cnpg status postgres-shared -n postgres

```



### Summary of Rules

* **Namespace Isolation:** Always create the source secret in `postgres`. Let Reflector handle the move to the app namespace.
* **Naming Consistency:** Ensure the `owner` name in the `Database` manifest matches the `name` in the `Cluster.spec.managed.roles` list exactly.
* **Sync Waves:** Always use Wave `0` for Secrets, Wave `1` for the Cluster, and Wave `2` for Databases.