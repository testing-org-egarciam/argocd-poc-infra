# GitOps Architecture Evolution

This document traces the evolution of our Hub-and-Spoke GitOps architecture, from the initial Proof of Concept (PoC) to the current state, supporting multiple distinct applications across disparate spoke clusters.

## Initial State: The Proof of Concept (PoC)

The initial PoC was designed to validate a strict GitOps separation of concerns for a single application (`laura-app`).

*   **Hub Cluster:** `argocd-hub` running the ArgoCD control plane.
*   **Spoke Clusters:** `laura-app-prod` and `laura-app-stg`.
*   **Repositories:**
    *   `laura-app`: The Application Source code (The "What").
    *   `gitops-config`: The Environment configuration values (The "Where Specifics").
    *   `argocd-hub`: The ArgoCD definitions and Projects (The "Glue").
    *   `argocd-poc-infra`: The Infrastructure as Code (IaC) and documentation.

### Key Milestones & Hurdles Overcome
1.  **Corporate Proxy Restrictions:** ArgoCD initially struggled to fetch external Helm charts (like Bitnami Nginx) due to MITM proxy rules. We bypassed this by generating raw Kubernetes manifests locally (`laura-app/apps/nginx`) and utilizing Helm's `dependency update` to vendor external `.tgz` files directly into our Git repositories.
2.  **CRD Metadata Size Limits:** The `applicationsets.argoproj.io` CRD exceeded the Kubernetes 262KB metadata limit when applied dynamically. We stripped legacy annotations and pulled the upstream CRD directly.
3.  **Dynamic Discovery with ApplicationSets:** We refactored from static individual `Application` manifests (`laura-app-prod.yaml`) to `ApplicationSets` (`appset-laura-app.yaml`). This allowed us to dynamically map clusters to environments (`prod`, `stg`) via List Generators, drastically reducing YAML boilerplate.
4.  **Inotify Limits (kube-proxy crashes):** We encountered complete network failure on the `kind` clusters because Docker nodes were running out of file descriptors. We implemented a permanent fix by injecting `fs.inotify.max_user_instances=8192` via `kubeadmConfigPatches` into all cluster creation scripts.

---

## Current State: Multi-App Scalability

We have proven the model scales horizontally by introducing a completely new application (`printolito` - a WordPress setup) with its own dedicated infrastructure.

### The Printolito Integration
1.  **New Spoke Cluster:** We provisioned `printolito-prod`, isolating its workloads entirely from the `laura-app` clusters. The cluster was successfully registered into the ArgoCD hub.
2.  **New Source Repository:** A new Git repository (`printolito`) was created. It contains a wrapper Helm chart that vendors the official Bitnami WordPress chart (bypassing external registry proxy issues).
3.  **Extended Configuration:** The `gitops-config` repository was expanded to include `values/printolito/prod.yaml`.
4.  **Isolated AppProject:** In `argocd-hub`, a new AppProject (`printolito-project.yaml`) was created to ensure `printolito` can only deploy to `172.18.0.5` (the new prod cluster) inside the `printolito-ns` namespace.
5.  **Dynamic ApplicationSet:** A new `appset-printolito.yaml` was applied. Even though it currently targets only a single production cluster, the `ApplicationSet` structure ensures it is future-proofed for potential staging environments.

### The resulting GitOps workflow remains identical across all teams:
*   **Developers** push chart updates to their respective application repo (`laura-app` or `printolito`).
*   **Platform Engineers** push environment tweaks to `gitops-config`.
*   **ArgoCD** seamlessly merges them via the `sources` array and applies them to the correct, isolated spoke clusters.