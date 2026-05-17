# ArgoCD PoC Infrastructure Repository

This repository contains all the infrastructure-as-code (IaC) and auxiliary files used to create the Proof of Concept (PoC) for a Hub-and-Spoke ArgoCD deployment.

## 📦 Contents

| Directory/File | Description |
| :--- | :--- |
| `cluster-configs/` | Kind cluster configuration files for the hub and spoke clusters. |
| `kubeconfigs/` | Exported kubeconfig files for the spoke clusters (used to create ArgoCD cluster secrets). |
| `cluster-secrets/` | Pre-generated Kubernetes Secret manifests for registering spoke clusters with the ArgoCD Hub. |
| `argocd-architecture.html` | High-fidelity, interactive architecture diagram of the Hub-and-Spoke model (with embedded Mermaid). |
| `argocd-deployment-plan.md` | Detailed deployment plan explaining the architecture, responsibilities (RACI), use cases, and IaC integration. |
| `poc_summary.md` | Concise summary of the PoC, including requirements, architecture, issues encountered, and a Mermaid diagram. |
| `poc_summary.html` | HTML version of the PoC summary with rendered Mermaid diagram. |
| `appset_crd*.yaml` | Various attempts to work around the ArgoCD ApplicationSet CRD size limit issue encountered during installation. |
| `fixed_secret.yaml` | Example of a manually constructed secret for cluster registration (used during troubleshooting). |
| `install.yaml` | The original ArgoCD installation manifest (used as a base for troubleshooting). |
| `tmp_secret.yaml` | Temporary file used during secret creation troubleshooting. |

## 🔧 How to Use This Repo

### Recreating the PoC Clusters
1. **Kind Clusters**: Use the files in `cluster-configs/` to create the hub and spoke clusters:
   ```bash
   kind create cluster --name argocd-hub --config cluster-configs/hub-cluster.yaml
   kind create cluster --name laura-app-prod --config cluster-configs/prod-cluster.yaml
   kind create cluster --name laura-app-stg --config cluster-configs/stg-cluster.yaml
   ```
   (Adjust cluster names as desired.)

2. **Export Kubeconfigs** (if not already done):
   ```bash
   kubectl config view --context kind-laura-app-prod > kubeconfigs/prod-kubeconfig.yaml
   kubectl config view --context kind-laura-app-stg > kubeconfigs/stg-kubeconfig.yaml
   ```

3. **Create Cluster Secrets** in the ArgoCD namespace on the hub cluster:
   - Use the files in `cluster-secrets/` as templates, or create them manually:
     ```bash
     kubectl --context kind-argocd-hub create secret generic laura-app-prod-secret \
       --namespace argocd \
       --from-file=config=cluster-secrets/prod-cluster-secret.yaml \
       --label "argocd.argoproj.io/secret-type=cluster"
     ```
   (Repeat for staging.)

### Understanding the Architecture
- Open `argocd-architecture.html` in a browser to view the Hub-and-Spoke diagram.
- Read `argocd-deployment-plan.md` for a deep dive into the design, responsibilities (RACI), and how this integrates with IaC.
- See `poc_summary.md` or `poc_summary.html` for a quick summary and lessons learned.

## 📚 Related Repositories in the `testing-org-egarciam` Org

| Repository | Purpose |
| :--- | :--- |
| `laura-app` | Application source repo (The "What") — contains the Helm chart for the Laura app. |
| `gitops-config` | Environment config repo (The "Where") — contains environment-specific values (e.g., `values/laura-app/prod.yaml`). |
| `argocd-hub` | Hub management repo (The "How") — contains the `AppProject` and `Application` manifests. |
| `argocd-poc-infra` | This repo — infrastructure and auxiliary files for the PoC. |

## 🛠️ Notes and Troubleshooting

- During the PoC, we encountered an issue with the `applicationsets.argoproj.io` Custom Resource Definition (CRD) exceeding the Kubernetes API server's metadata size limit (262KB). As a workaround, we used explicit `Application` manifests instead of an `ApplicationSet`. The various `appset_crd*.yaml` files show our attempts to fix the CRD by stripping annotations.
- The `fixed_secret.yaml` and `tmp_secret.yaml` files are artifacts from troubleshooting the creation of ArgoCD cluster secrets.

## 🔄 How to Extend or Reuse

- To add a new application (e.g., "payment-app"):
  1. Create a new source repo (e.g., `payment-app`) with a Helm chart.
  2. Add environment values to `gitops-config/values/payment-app/` (e.g., `prod.yaml`, `stg.yaml`).
  3. Add `Application` manifests to `argocd-hub` pointing to the new repo and values.
- To add a new spoke cluster:
  1. Provision the cluster (e.g., via Terraform/Terragrunt in GCP).
  2. Create a ServiceAccount and export its kubeconfig.
  3. Create a Secret in the `argocd` namespace on the hub cluster (labelled `argocd.argoproj.io/secret-type: cluster`).
  4. Ensure the `AppProject` in `argocd-hub` allows the cluster as a destination (or uses a wildcard with namespace restrictions).

---
*This repository is a companion to the application and configuration repos, providing everything needed to bootstrap the infrastructure for the ArgoCD Hub-and-Spoke PoC.*
