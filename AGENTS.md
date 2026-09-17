# k3s

ArgoCD GitOps repo. Every manifest is deployed by ArgoCD (selfHeal, targetRevision HEAD).
Bad YAML merged to main reaches the cluster immediately — main is production.

## Layout
- gitops/apps/         Application CRs (Helm charts or git-path sources)
- gitops/manifests/    raw manifests per service

## Conventions
- One ArgoCD Application per service; CreateNamespace=true, prune+selfHeal.
- Helm values inline in spec.source.helm.values.
- DBs: single-replica StatefulSet on ceph-rbd, startupProbe, subPath.
- Single-writer volumes use strategy: Recreate.
- Per-namespace NetworkPolicies; Traefik-only ingress + IP allowlists.
- Secrets created via kubectl, never committed. Floating image tags forbidden.
- Run: kubeconform, yamllint, helm lint, kustomize build (CI #21).