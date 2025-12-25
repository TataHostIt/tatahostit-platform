# Monitoring Stack Implementation Guide

**Date:** 2025-12-21
**Version:** 1.0
**Context:** Phase 1 Deployment of `kube-prometheus-stack` with external hardware monitoring.

## 1. Secret Management (Critical)

This stack uses **Sealed Secrets** to manage sensitive credentials for Grafana, Proxmox, and IPMI. We do not store raw secrets in Git.

### Prerequisites

* `kubectl` and `kubeseal` installed locally.
* The cluster's public certificate: `my-pub-cert.pem`.

### ⚠️ Password Safety Policy

When generating secrets via the CLI, certain characters can confuse the Linux shell (e.g., `!`, `$`, `\`).

* **Safe Characters:** `a-z`, `A-Z`, `0-9`, `-`, `_`, `.`, `@`
* **Avoid in CLI:** `!`, `$`, `\`, `'`, `"` (If you must use these, save the password to a file and use `--from-file`).

### Step-by-Step Generation Guide

**1. Generate the Raw Secret (Dry Run)**
Run this command to generate the manifest.

* **Note:** We include the `admin-user` key explicitly so Helm knows what username to assign.
* **Proxmox User Format:** Must be `USER@REALM!TOKEN_ID` (e.g., `monitor@pve!monitoring-token`).
* **Proxmox Token:** UUID only.

```bash
# REPLACE THE DUMMY VALUES BELOW WITH REAL CREDENTIALS
# DO NOT COMMIT THIS OUTPUT TO GIT
kubectl create secret generic monitoring-creds \
  --namespace monitoring \
  --from-literal=admin-user='admin' \
  --from-literal=grafana-admin-password='YourSafePassword123' \
  --from-literal=proxmox-user='monitor@pve!monitoring-token' \
  --from-literal=proxmox-token='00000000-0000-0000-0000-000000000000' \
  --from-literal=ipmi-user='ADMIN' \
  --from-literal=ipmi-pass='hunter2' \
  --dry-run=client -o yaml > plain-secret.yaml

```

**2. Seal the Secret**
Encrypt the file using the public certificate. This output file IS safe to commit to Git.

```bash
kubeseal --cert my-pub-cert.pem \
  --format yaml \
  < plain-secret.yaml > infra/monitoring/monitoring-secrets.yaml

```

**3. Cleanup**
Immediately delete the plain text file.

```bash
rm plain-secret.yaml

```

---

## 2. Monitoring Architecture & Data Sources

We are deploying a hybrid monitoring stack that pulls data from three distinct layers: Internal Kubernetes, Physical Hardware (IPMI), and the Virtualization Layer (Proxmox/Ceph).

### A. Internal Kubernetes (Standard Stack)

* **Source:** `kube-prometheus-stack` default scrapers.
* **Targets:**
* **Kubelet:** Container resource usage (RAM/CPU/Network).
* **Kube API Server:** Control plane health.
* **Node Exporter:** Underlying Talos node metrics (Disk I/O, Kernel stats).
* **CoreDNS & Etcd:** Critical cluster services.



### B. External Hardware & Infrastructure (The "Magic" Config)

These targets are defined in `values.yaml` under `additionalScrapeConfigs` because they live outside the Kubernetes cluster.

#### 1. Proxmox Cluster (PVE)

* **Method:** API Polling via `prometheus-pve-exporter`.
* **Source IPs:** The exporter talks to the cluster API (via `10.10.10.11` etc).
* **Data Points:**
* **Cluster Health:** Quorum status, overall load.
* **Node Status:** CPU/RAM usage of the physical hypervisors.
* **VM/LXC Status:** Up/Down status of VMs (including the Talos nodes themselves).
* **Storage:** Local ZFS/LVM usage on Proxmox nodes.



#### 2. Physical Hardware (IPMI/iLO)

* **Method:** IPMI over UDP (Port 623) via `prometheus-ipmi-exporter`.
* **Targets:**
* `10.10.10.111` (Node 1 iLO)
* `10.10.10.112` (Node 2 iLO)
* `10.10.10.113` (Node 3 iLO)
* `10.10.10.114` (Node 4 Supermicro IPMI)


* **Data Points:**
* **Thermals:** CPU temperatures, Ambient board temps.
* **Power:** Fan speeds (RPM), Power Supply Unit (PSU) health & load.
* **Hardware:** Chassis intrusion, DIMM/RAM faults.



#### 3. External Ceph Storage

* **Method:** Direct scrape of Ceph Manager Prometheus Module.
* **Targets:**
* `10.10.10.11:9283`
* `10.10.10.12:9283`
* `10.10.10.13:9283`


* **Data Points:**
* **Cluster Health:** Overall Ceph Status (Health_OK, Health_WARN).
* **OSD Status:** Number of OSDs Up/In/Down.
* **Pool Usage:** IOPS, Throughput, and Capacity usage for `ceph-nvme` and `ceph-hdd`.
* **Latency:** R/W latency at the storage layer.



---

## 3. Storage & Retention Policy

* **Persistence:** Enabled via `sc-nvme` (Ceph NVMe).
* **Volume Size:** 200 GiB.
* **Retention Period:** 180 Days.
* **Expansion:** Allowed. To increase storage, edit `values.yaml` -> `prometheusSpec.storageSpec` and sync via Argo CD.

## 4. Network Ports Reference

| Service | Port | Protocol | Direction | Description |
| --- | --- | --- | --- | --- |
| **Grafana UI** | 80/443 | HTTP/S | Inbound | Accessed via `grafana.tatahostit.local` (Ingress) |
| **IPMI Exporter** | 9290 | HTTP | Internal | Prometheus scrapes this Pod |
| **IPMI Hardware** | 623 | UDP | Outbound | Exporter Pod -> Physical Node (10.10.10.x) |
| **Ceph Manager** | 9283 | HTTP | Outbound | Prometheus -> Proxmox Node (10.10.10.x) |
| **PVE Exporter** | 9221 | HTTP | Internal | Prometheus scrapes this Pod |