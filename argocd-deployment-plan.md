# ArgoCD Hub-and-Spoke Deployment Model

This document outlines the operational strategy for managing multi-cluster deployments using a centralized ArgoCD Hub and a strategic multi-repository GitOps approach.

## 1. Architecture Overview

The system follows a **Hub-and-Spoke model**. A single dedicated management cluster (The Hub) runs the ArgoCD control plane, which orchestrates the state of multiple remote clusters (The Spokes) by synchronizing desired states from three distinct types of GitHub repositories.

### Infrastructure Topology
- **Hub Cluster:** Hosts the ArgoCD API, Controller, and Repo-server.
- **Spoke Clusters:** Target environment clusters (Dev, Staging, Prod) containing only the business workloads.
- **Connectivity:** Hub connects to Spoke clusters via their Kubernetes API servers (typically via VPC peering or Private Service Connect in GCP).

### The Multi-Repo Strategy
| Repository Type | Scope | Content | Purpose |
| :--- | :--- | :--- | :--- |
| **App Repo** | Per App | Code, Dockerfile, Base Helm Chart | Defines **WHAT** is being deployed. |
| **Hub Repo** | Global | `AppProject`, `ApplicationSet` | Defines **HOW** it is deployed (Governance). |
| **Config Repo** | Global | `values-prod.yaml`, `values-stg.yaml` | Defines **WHERE** it is deployed (State). |

---

## 2. The Logistics Flow (Use Cases)

### Use Case A: Deploying a New Application
1. **Dev Team** creates a new **App Repo** with a Helm chart.
2. **Platform Team** adds a new `ApplicationSet` in the **Hub Repo** to map this app to the target clusters.
3. **Dev Team** adds environment-specific values (e.g., resource limits) in the **Config Repo**.
4. **ArgoCD** detects the new `ApplicationSet` $\rightarrow$ Generates `Applications` $\rightarrow$ Deploys to remote clusters.

### Use Case B: Scaling Production Replicas
1. **SRE** opens a PR in the **Config Repo** changing `replicaCount: 3` $\rightarrow$ `replicaCount: 10` in `values-prod.yaml`.
2. **PR is merged.**
3. **ArgoCD Hub** detects the change in the Config Repo and triggers a sync only on the Production Spoke cluster.

---

## 3. Responsibilities (RACI Matrix)

| Activity | App Dev Team | Platform/SRE Team | Security/Gov Team | Cluster Infrastructure Team |
| :--- | :---: | :---: | :---: | :---: |
| App Code & Base Chart | **R/A** | I | I | - |
| ArgoCD Hub Installation | I | **R/A** | C | C |
| `AppProject` & RBAC | I | **R** | **A** | - |
| Env Values ($\text{config repo}$) | **R** | **A** | I | - |
| Cluster Provisioning (GCP) | - | C | I | **R/A** |
| Network Connectivity (Hub$\rightarrow$Spoke) | - | **R** | C | **A** |

**Legend:** **R**: Responsible | **A**: Accountable | **C**: Consulted | **I**: Informed

---

## 4. Integration with Infrastructure-as-Code (IaC)

The critical link in this chain is the relationship between the **Cluster Infrastructure Team** (the "Foundry") and the **ArgoCD Hub** (the "Orchestrator").

### The IaC Pipeline (Terraform/Terragrunt)
The Infrastructure Team operates a separate **IaC Repo** that manages the lifecycle of GCP Project, VPCs, and GKE Clusters.

**The Integration Workflow:**
1. **Provisioning:** The IaC pipeline deploys a new GKE cluster in GCP.
2. **Bootstrap Handshake:** As part of the Terraform `post-provisioning` step, the IaC pipeline:
    - Creates a K8s ServiceAccount on the new spoke cluster.
    - Generates a Kubeconfig/Token.
    - **Calls the ArgoCD API** (or updates a secret in the Hub) to register the new cluster as a managed destination.
3. **Auto-Discovery:** The `ApplicationSet` in the Hub repo uses a **Cluster Generator**. As soon as the new cluster is registered with the label `env: staging`, ArgoCD automatically detects it and deploys all "Staging" apps to the new cluster without manual intervention.

### Combined Dependency Map
`IaC Repo` $\xrightarrow{\text{Creates}}$ `GCP Cluster` $\xrightarrow{\text{Registers}}$ `ArgoCD Hub` $\xrightarrow{\text{Syncs}}$ `App/Config Repos` $\xrightarrow{\text{Deploys}}$ `Workloads`

---

## 5. Visual Reference
The detailed architectural flow is documented in the accompanying diagram:
**File:** `/home/yayito/projects/random/argocd-architecture.html`
