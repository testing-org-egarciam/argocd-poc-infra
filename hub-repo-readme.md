# ArgoCD Hub Management Repository

This repository serves as the **"How"** in the GitOps model: it defines the **governance, deployment logic, and target mapping** for the Hub-and-Spoke ArgoCD control plane.

## 📦 Repository Purpose

In the **Hub-and-Spoke Architecture** demonstrated in this PoC:

- **Hub Cluster**: Runs the ArgoCD control plane (API server, controller, repo-server, etc.).
- **Spoke Clusters**: Target Kubernetes clusters where workloads are actually deployed (e.g., `laura-app-prod`, `laura-app-stg`).
- **Git Repos**: 
  - **Application Source Repo** (`laura-app`): Contains the *what* — source code and base Helm charts.
  - **Environment Config Repo** (`gitops-config`): Contains the *where* — environment-specific values (e.g., replica counts, labels).
  - **This Repo** (`argocd-hub`): Contains the *how* — who can deploy what where, and how ArgoCD discovers and manages applications.

## 📁 Contents

| File | Description |
| :--- | :--- |
| `laura-app-project.yaml` | Defines an `AppProject` that scopes which source repos and destinations are allowed for the "Laura" application. |
| `laura-app-prod.yaml` | An `Application` resource that tells ArgoCD to deploy the Laura app to the production cluster using values from `gitops-config`. |
| `laura-app-stg.yaml` | An `Application` resource for the staging counterpart. |
| `README.md` | This file. |

## 🔗 How It Works

1. **ArgoCD Hub** watches this repository for changes (via a repo-server configured to point here).
2. When an `Application` or `AppProject` is applied, ArgoCD:
   - Fetches the base Helm chart from the **Application Source Repo** (`laura-app`).
   - Retrieves environment-specific overrides from the **Environment Config Repo** (`gitops-config`).
   - Renders the final manifests and applies them to the target cluster specified in the `Application.spec.destination`.
3. The `AppProject` ensures that:
   - Only the approved source repos (e.g., `laura-app`, `gitops-config`) can be used.
   - Deployments are limited to the predefined destinations (e.g., the spoke clusters).

## 🧩 Example Flow (Laura App)

1. Developer pushes a change to `laura-app` (e.g., new Docker image tag).
2. SRE updates `gitops-config/values/laura-app/prod.yaml` (e.g., increases replica count).
3. ArgoCD detects the change in either repo:
   - Pulls the latest chart from `laura-app`.
   - Pulls the latest values from `gitops-config`.
   - Applies the release to `laura-app-prod` and/or `laura-app-stg` clusters.
4. The cluster state converges to the desired state defined in Git.

## 🛡️ Governance & Isolation

- The `AppProject` (`laura-project`) restricts:
  - **Source Repos**: Only `laura-app` and `gitops-config` are allowed.
  - **Destinations**: Only the clusters registered as destinations in this project (effectively, the spoke clusters we registered via secrets).
- This prevents a misconfigured or malicious application from:
  - Pulling charts from unapproved repositories.
  - Deploying to clusters outside the intended scope (e.g., a dev cluster pulling production values).

## 🚀 How to Use This Repo in Practice

### Adding a New Application (e.g., "Payment App")
1. **Create** the source repo: `payment-app` (with Helm chart).
2. **Add** environment values: `gitops-config/values/payment-app/prod.yaml` and `stg.yaml`.
3. **Add** to this repo:
   - Optionally extend or reuse the `AppProject` (if same team/policy).
   - Create two `Application` manifests:
     - `payment-app-prod.yaml` → points to `values/payment-app/prod.yaml`
     - `payment-app-stg.yaml` → points to `values/payment-app/stg.yaml`
4. Commit and push. ArgoCD will sync and deploy the new application.

### Adding a New Spoke Cluster (via IaC)
1. Provision the cluster (e.g., with Terraform/Terragrunt in GCP).
2. Create a Kubernetes ServiceAccount with limited permissions.
3. Generate a kubeconfig/token.
4. Create a Secret in the `argocd` namespace on the Hub:
   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: <cluster-name>-secret
     namespace: argocd
     labels:
       "argocd.argoproj.io/secret-type": "cluster"
   type: Opaque
   stringData:
     config: |
       <kubeconfig-content>
   ```
5. Ensure the `AppProject`'s `destinations` list includes a match for the cluster’s API server (or use a wildcard with namespace restrictions).

## 📈 Observability

- All ArgoCD components expose metrics at `:8082/metrics` (by default) for Prometheus scraping.
- Application health and sync status are visible via the ArgoCD UI or CLI:
  ```bash
  argocd app list --cluster <cluster-name>
  ```

## 🔄 Sync Policy

Both `Application` manifests in this repo use:
```yaml
syncPolicy:
  automated:
    prune: true     # Delete resources removed from Git
    selfHeal: true  # Reconcile drift automatically
```
This ensures the cluster state always reflects the Git state.

## 📚 Related Documentation

- [PoC Summary (Markdown)](poc_summary.md)
- [PoC Summary (HTML)](poc_summary.html)
- [Application Source Repo](https://github.com/testing-org-egarciam/laura-app/blob/main/README.md)
- [Environment Config Repo](https://github.com/testing-org-egarciam/gitops-config/blob/main/README.md)

---
*This repository is the "brain" of the operation: it decides what gets deployed, where, and by whom. Keep it under strict change control (e.g., protected branch, PR reviews).*