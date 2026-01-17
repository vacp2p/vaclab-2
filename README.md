# Vaclab-2 Cluster Deployment

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-K3s-326CE5?logo=kubernetes&logoColor=white)](https://k3s.io/)
[![Ansible](https://img.shields.io/badge/Ansible-2.9+-EE0000?logo=ansible&logoColor=white)](https://www.ansible.com/)

> Automated deployment and configuration for the Vaclab-2 Kubernetes cluster using Ansible and Fleet.

## Quick Links

- **Homepage (Dev)**: [home.vaclab](https://home.vaclab.diarra.tech/)

## Project Status

| Component | Status | Description |
|-----------|--------|-------------|
| K3s Cluster Provisioning | ✅ OK | K3s cluster deployment via Ansible |
| Helm | ✅ OK | Helm 3 installation on control plane |
| Rancher Fleet | ✅ OK | GitOps deployment engine |
| traefik | ✅ OK | K3S-friendly ingress |
| containerd | ✅ OK | Container Runtime |
| Cert Manager | ✅ OK | Autamed TLS via HTTP-01 challenges |
| Kube-OVN | ✅ OK | Primary CNI |
| Cilium | ✅ OK | Chained CNI |
| Hubble | ✅ OK | Cilium real-time flow observability UI |
| Longhorn | ✅ OK | CSI - Distributed block storage |
| Rancher UI | ✅ OK | Rancher management UI |
| GetHomePage | ✅ OK | Home Page UI (App Launcher) for vaclab|
| Authentik | ✅ OK | Identity & Access Management |
| VictoriaMetrics K8s Stack | ✅ OK | VictoriaMetrics Helm charts, including Grafana |
| VictoriaLogs Cluster | ✅ OK | VictoriaLogs Helm chart |
| Kyverno | ✅ OK | Admission & mutation webhook manager |
| Kyverno default resource request enforcement | ✅ OK |Injection of default resource requests when missing |
| Kyverno Policy Reporter | ✅ OK | Policy Observability UI |
| Vaclab Bandwidth-Aware Scheduler | ✅ OK | Custom bandwidth-aware scheduler |

---

## Overview

This repository provides automated deployment and management for the Vaclab-2 Kubernetes infrastructure. It combines Ansible playbooks for initial cluster provisioning with Fleet-managed GitOps for continuous reconciliation. The automation handles:

- **K3s Cluster Deployment**
- **GitOps with Fleet** (everyhting under the fleet/ directory is deployed on the cluster)

---

## Prerequisites

### System Requirements

- **Control Machine** (where you run Ansible):
  - Linux, macOS, or WSL2
  - Python 3.8+
  - SSH access to target nodes

- **Target Nodes** (cluster machines):
  - Ubuntu 24.04
  - Sudo privileges for deployment user

### Software Dependencies

#### 1. Install Ansible

**On Ubuntu/Debian:**
```bash
sudo apt update
sudo apt install -y software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install -y ansible
```

#### 2. Install Ansible Collections

Install required Ansible collections for Kubernetes management:

```bash
ansible-galaxy collection install kubernetes.core
ansible-galaxy collection install community.general
```

#### 3. Install Python Dependencies

Install Python packages needed by Ansible modules:

```bash
sudo apt install python3-kubernetes python3-yaml
```

---

## Configuration

### 1. Inventory Setup

Edit `ansible/inventory.yaml` to define your cluster nodes IPs and the ansible_user (by default the user is set to ubuntu):

### 2. SSH Key Setup

Ensure passwordless SSH access to all nodes:

```bash
ssh-copy-id <ansible_user>@<host>
```

---

## Deployment

### Quick Start

Deploy the complete cluster with a single command:

```bash
cd ansible
ansible-playbook -i inventory.yaml playbooks/setup_cluster.yaml --vault-password-file ../.vault_pass.txt
```

This will:
1. Install and configure K3s cluster
2. Install Helm on the control plane
3. Deploy Rancher Fleet and create a GitRepo resource in order to watch the fleet/ directory inside this git repo

### Step-by-Step Deployment

If you prefer to deploy components individually:

#### 1. Deploy K3s Cluster

```bash
cd ansible
ansible-playbook -i inventory.yaml k3s-ansible/playbooks/site.yml
```

#### 2. Install Helm

```bash
ansible-playbook -i inventory.yaml playbooks/setup_cluster.yaml --tags helm
```

#### 3. Install Rancher Fleet

```bash
ansible-playbook -i inventory.yaml playbooks/setup_cluster.yaml --tags fleet
```

---

## Post-Deployment

### Check Fleet Status

```bash
kubectl -n fleet-local get gitrepo
kubectl -n cattle-fleet-system get pods
```

---

## Known Issues

### Authentik Outpost Provider Assignment Failure

**Symptom**: After deploying Authentik, accessing protected services (like Longhorn) returns an error:
```json
{
  "Message": "no app for hostname",
  "Host": "longhorn.vaclab.diarra.tech",
  "Detail": "Check the outpost settings..."
}
```

**Root Cause**: The `k8s-outpost` blueprint may fail to assign the `k8s-forwardauth` provider to the embedded outpost due to a timing/ordering issue during initial deployment.

**Verification**:
```bash
# Check if provider is assigned to outpost
kubectl -n authentik exec authentik-postgresql-0 -- \
  env PGPASSWORD=vaclab psql -U authentik -d authentik \
  -c "SELECT o.name, p.name as provider FROM authentik_outposts_outpost o \
      JOIN authentik_outposts_outpost_providers op ON o.uuid = op.outpost_id \
      JOIN authentik_core_provider p ON op.provider_id = p.id;"
```

If the query returns no results, the provider is not assigned.

**Fix**:
```bash
# Manually apply the outpost blueprint
kubectl exec -n authentik deployment/authentik-worker -- \
  ak apply_blueprint /blueprints/mounted/cm-authentik-blueprints/20-outpost.yaml

# Restart the outpost to pick up changes
kubectl -n authentik rollout restart deployment/authentik-outpost
```

**Verification** (should return `authentik Embedded Outpost | k8s-forwardauth`):
```bash
kubectl -n authentik exec authentik-postgresql-0 -- \
  env PGPASSWORD=vaclab psql -U authentik -d authentik \
  -c "SELECT o.name, p.name as provider FROM authentik_outposts_outpost o \
      JOIN authentik_outposts_outpost_providers op ON o.uuid = op.outpost_id \
      JOIN authentik_core_provider p ON op.provider_id = p.id;"
```

