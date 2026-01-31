# PostgreSQL Shared Infrastructure Guide

## Connectivity Overview

| Connection Type | Target URL / Host | Purpose |
| :--- | :--- | :--- |
| **Cluster Internal** | `postgres-shared-rw.databases.svc.cluster.local` | For apps like LiteLLM/Keycloak |
| **Developer DNS** | `postgresdb-dev.internal.tatahostit.com` | Primary access for dev tools (DBeaver, etc) |
| **Static IP** | `10.20.20.201` | Direct access if DNS is unavailable |

## User Access
- **Port:** `5432`
- **Application User:** `app_user` (Owner of `app_db`)
- **Superuser:** `postgres`
- **Password:** Contact Admin (Managed via SealedSecrets)

## Infrastructure Details
- **Architecture:** 3-node HA (1 Primary, 2 Replicas)
- **Storage:** 20Gi Ceph NVMe RBD per instance
- **Platform:** Talos Kubernetes
- **Backup/DR:** Managed by CloudNativePG Operator