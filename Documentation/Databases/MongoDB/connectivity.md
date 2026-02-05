# MongoDB Connectivity Guide

This document defines the standard connection methods for the TataHostIT MongoDB HA cluster.

## 1. Connection Requirements & Security

Access to the MongoDB cluster is restricted based on your physical location and network identity.

* **Home/Local Access:** Only the cluster administrator on the primary host network can connect directly to the 10.20.20.x IPs without a VPN.
* **Developer/Remote Access:** All other connections **must** be established through the Tailscale VPN.
* **VPN Setup:** [TODO: Insert link to Tailscale/VPN Setup Guide]
* **Authentication:** All users must authenticate against the `admin` database (`authSource=admin`), regardless of which application database they are accessing.

---

## 2. Developer & External Access (via Tailscale)

Developers connecting from laptops or external services must use the **Replica Set Connection String**. This format allows the MongoDB driver to automatically discover the current **Primary** node among the three cluster members.

### Primary Connection String (The "Source of Truth")

```text
mongodb://<username>:<password>@10.20.20.202:27017,10.20.20.203:27017,10.20.20.204:27017/?replicaSet=rs0&authSource=admin&tls=false&tlsAllowInvalidCertificates=true

```

### Accessing Specific Roles

| User Role | Credentials Source | Use Case |
| --- | --- | --- |
| **Global Admin** | `mongodb-cluster-secrets` | Full cluster visibility, creating indexes, or database maintenance. |
| **Team Developer** | `<app>-secrets-sealed.yaml` | Working within a specific team database (e.g., `voice-dev`). |

---

## 3. Internal Cluster Access (Pod-to-Pod)

Applications running **inside** the same Kubernetes cluster should not use the static 10.20.20.x IPs. Instead, they should use the internal Kubernetes DNS for lower latency and better resilience if nodes are rescheduled.

### Standard Internal URL

```text
mongodb://<username>:<password>@mongodb-cluster-rs0.mongodb.svc.cluster.local:27017/<database>?replicaSet=rs0&authSource=admin

```

### Implementation in Deployment YAML

Always use the **Reflected Secret** in your application namespace to inject the connection string as an environment variable.

```yaml
env:
  - name: MONGODB_URI
    valueFrom:
      secretKeyRef:
        name: <app-name>-secrets # The secret mirrored by Reflector
        key: MONGODB_URL

```

---

## 4. Administrative "Quick Access" Tools

For temporary debugging or local development without a full driver setup.

### A. Port-Forwarding (Localhost Tunnel)

If you need to use a local tool like MongoDB Compass but cannot route to the 10.20.20.x network directly:

```bash
# Forward local port 27017 to the primary pod
kubectl port-forward -n mongodb svc/mongodb-cluster-rs0 27017:27017

```

**Connection String:** `mongodb://clusterAdmin:<password>@localhost:27017/?authSource=admin`

### B. Direct Shell Access (mongosh)

To run a quick command directly on the database node:

```bash
kubectl exec -it -n mongodb mongodb-cluster-rs0-0 -c mongod -- mongosh "mongodb://clusterAdmin:<password>@localhost:27017/?authSource=admin"

```

---

## 5. Connectivity Troubleshooting

If you cannot connect, verify the following in order:

1. **Tailscale Status:** Ensure your VPN is active and the `10.20.20.0/24` subnet is reachable.
2. **Pod Health:** Run `kubectl get pods -n mongodb` to ensure all 3 replica set members are `Running`.
3. **Service Endpoints:** Run `kubectl get endpoints -n mongodb mongodb-cluster-rs0` to confirm the service is pointing to the correct pod IPs.
4. **AuthSource:** Ensure your connection string includes `authSource=admin`. Without this, MongoDB will look for your user in the application database rather than the central identity registry.

---

