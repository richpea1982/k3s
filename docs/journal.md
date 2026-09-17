# journal

Score-keeping for decisions made in sessions: what and WHY (not a step log).
Format:

## YYYY-MM-DD — topic
- decision, reason, ticket ref

## 2026-09-17 — opencode project scaffolding
- Added AGENTS.md + opencode.json + journal so the working agreement and
  GitOps conventions load on every session. refs #22

## 2026-09-17 — manifest validation CI
- Added .github/workflows/ci.yml: yamllint (relaxed) + kubeconform
  -strict -ignore-missing-schemas on gitops/. refs #21
- First run found trailing whitespace in
  gitops/manifests/kube-prometheus-stack/telegram-alerts.yaml — queued in
  the yamllint cleanup ticket. refs #26
- Scope decision: no helm lint / kustomize build / checkov yet — kept the
  gate cheap and ArgoCD stays the deploy authority. main remains production
  via selfHeal, so the workflow deliberately never trends toward apply.