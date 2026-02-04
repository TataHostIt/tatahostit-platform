# PostgreSQL Shared Infrastructure Guide

## Connectivity Overview

| Connection Type | Target URL / Host                               | Purpose |
| :--- |:------------------------------------------------| :--- |
| **Cluster Internal** | `postgres-shared-rw.postgres.svc.cluster.local` | For apps like LiteLLM/Keycloak |
| **Developer DNS** | `postgresdb-dev.internal.tatahostit.com`        | Primary access for dev tools (DBeaver, etc) |
| **Static IP** | `10.20.20.201`                                  | Direct access if DNS is unavailable |

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

# Database Connectivity Guide

## For Applications (Internal)
All apps in the cluster can reach the primary (writeable) DB using this URL:
**Host:** `postgres-shared-rw.databases.svc.cluster.local`  
**Port:** `5432`

## For Developers (External)
Connect using your local dev tools (DBeaver, DataGrip) via:
**Host:** `postgresdb-dev.internal.tatahostit.com`  
**IP:** `10.20.20.201`  
**Port:** `5432`

## Credentials
- **Superuser:** `postgres`
- **App User:** `app_user`
- **Shared Password:** Get from the admin (stored in SealedSecrets).

## Operations
- **Expand Storage:** Increase `storage.size` in `postgres-cluster.yaml`. Expansion is online/instant with Ceph. 
- **Monitoring:** Open Grafana and search for the "CloudNativePG" dashboard.