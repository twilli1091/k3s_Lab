# K3s Home Lab: GitOps-Driven Bare-Metal Kubernetes

![Kubernetes](https://img.shields.io/badge/kubernetes-%23326ce5.svg?style=for-the-badge&logo=kubernetes&logoColor=white)
![ArgoCD](https://img.shields.io/badge/ArgoCD-ef7b4d.svg?style=for-the-badge&logo=argo&logoColor=white)
![Helm](https://img.shields.io/badge/Helm-0F1689.svg?style=for-the-badge&logo=helm&logoColor=white)

This repository contains the declarative configuration for my personal bare-metal Kubernetes cluster. It is built around **GitOps** principles using **ArgoCD**, demonstrating modern platform engineering practices, infrastructure-as-code, and robust lifecycle management of both infrastructure components and user-facing applications.

## 🎯 Architecture Overview

This cluster is designed to mimic a production-grade on-premise environment. It handles the complete lifecycle of bare-metal provisioning complexities, including L2 networking, distributed storage, secure secret management, and specialized hardware observability (GPUs and Dell iDRAC).

The deployment strategy utilizes ArgoCD's **App of Apps** pattern. A single `root-app` points to this repository, automatically syncing the platform infrastructure and workloads.

### 🏗️ Tech Stack & Components

| Category | Component | Description / Purpose |
| :--- | :--- | :--- |
| **Orchestration** | [K3s](https://k3s.io/) | Lightweight, production-ready Kubernetes distribution optimized for bare-metal. |
| **GitOps** | [ArgoCD](https://argo-cd.readthedocs.io/) | Continuous delivery and cluster state synchronization directly from Git. |
| **Networking** | [MetalLB](https://metallb.universe.tf/) | Bare-metal load balancer providing external IP addresses to services. |
| **Ingress/Routing** | [Envoy Gateway](https://gateway.envoyproxy.io/) | Modern routing utilizing the Kubernetes **Gateway API** for advanced traffic management. |
| **Storage** | [Longhorn](https://longhorn.io/) | Cloud-native, distributed block storage for persistent volumes (PVCs). |
| **Secrets** | [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) | One-way encrypted secrets, allowing safe storage of sensitive data in this public repository. |
| **Hardware / GPU** | Time-Slicing | Configured GPU time-slicing to share NVIDIA resources across multiple workloads. |

### 📊 Observability Stack

Comprehensive monitoring is implemented to track both cluster health and underlying physical hardware:
* **Prometheus & Grafana:** (`kube-prometheus-stack`) Core metrics aggregation and visualization.
* **iDRAC Exporter:** Custom metrics scraping from Dell out-of-band management interfaces to monitor physical server health (temps, power).
* **DCGM Exporter:** NVIDIA Data Center GPU Manager integration to monitor GPU utilization, memory, and thermals.

### 🚀 Applications

* **Jellyfin:** Media server leveraging the NVIDIA GPU for hardware-accelerated transcoding. Configured with persistent Longhorn storage and exposed via Envoy Gateway.
* **Fan Controller:** A custom workload designed to dynamically adjust bare-metal server fans based on real-time metrics, keeping acoustics and thermals balanced.

---

## 📂 Repository Structure

The repository is organized to cleanly separate cluster orchestration, platform components, and end-user applications:

```text
.
├── bootstrap/               # Initial ArgoCD installation manifests
├── clusters/
│   └── k3s-cluster/         # Environment specific configuration
│       ├── root-app.yaml    # The ArgoCD App-of-Apps root
│       └── apps/            # Application manifests linking components to the cluster
├── components/              # Reusable platform infrastructure (The "Platform")
│   ├── gpu/                 # GPU sharing and drivers
│   ├── ingress/             # Envoy Gateway API configs
│   ├── networking/          # MetalLB configurations
│   ├── observability/       # Prometheus, DCGM, iDRAC configurations
│   ├── secrets/             # Sealed Secrets controller
│   └── storage/             # Longhorn configuration
└── application/             # End-user workloads
    ├── jellyfin/
    └── fan-controller/
```

## 🛠️ Bootstrapping the Cluster

To recreate this cluster from scratch:

1. **Install K3s:**
   Provision a Linux machine and install K3s.
   ```bash
   curl -sfL https://get.k3s.io | sh -
   ```

2. **Install Sealed Secrets CLI (kubeseal):**
   Required to encrypt new secrets before committing them to the repository.

3. **Bootstrap ArgoCD:**
   Apply the initial ArgoCD manifests to the cluster.
   ```bash
   kubectl create namespace argocd
   kubectl apply -k gitops/bootstrap/argocd/
   ```

4. **Deploy the Root Application:**
   Point ArgoCD to this repository to begin the automated synchronization of all components.
   ```bash
   kubectl apply -f gitops/clusters/k3s-cluster/root-app.yaml
   ```
   *ArgoCD will take over, provisioning MetalLB, Longhorn, Envoy Gateway, the monitoring stack, and finally the applications.*

## 🔒 Secret Management Workflow

This repository contains no plaintext secrets. All sensitive data is encrypted using Bitnami Sealed Secrets. There are two primary workflows for generating these encrypted secrets:

### Option 1: Web UI
A custom web interface (`sealed-secrets-ui`) is deployed and exposed via Envoy Gateway.
1. Navigate to the Sealed Secrets Web UI (e.g., `secret.10-0-0-200.nip.io`).
2. Enter the secret key-value pairs and the target namespace/name in the form.
3. The UI will generate the encrypted `SealedSecret` YAML.
4. Copy the output and commit it to the repository.

### Option 2: CLI (`kubeseal`)
1. Create a standard Kubernetes Secret locally (do not commit).
2. Encrypt it using the `kubeseal` CLI:
   ```bash
   kubeseal --format=yaml < my-secret.yaml > my-sealed-secret.yaml
   ```
3. Commit `my-sealed-secret.yaml` to the repository. The Sealed Secrets controller inside the cluster will safely decrypt it upon synchronization.

---
*This repository serves as an active home-lab and a demonstration of GitOps methodologies, Kubernetes platform engineering, and bare-metal cluster management.*
