This project implements a cluster-scoped reliability guardrails platform for Kubernetes progressive delivery, managed entirely via GitOps.

# Project 9 — Fail‑Closed Reliability Guardrails (Argo Rollouts + Prometheus + GitOps)

Project 9 implements **fail‑closed reliability guardrails** for Kubernetes progressive delivery using **Argo Rollouts Analysis** and **Prometheus**.  
The focus is **policy + decision logic**, managed via **GitOps**, and validated through real integration work with Project 8 rollouts.

> **Fail‑closed** means: if the system cannot prove safety (metrics missing, query errors, ambiguous state), it **blocks** by default.

---

## 1. Objectives

1. Implement reusable **ClusterAnalysisTemplates** for rollout guardrails
2. Add a **prod‑safe variant** (more conservative evaluation)
3. Add a **second reliability metric** (readiness)
4. Add an **image hygiene guardrail** (`:latest` detection) as a rollout protection check
5. Integrate the templates into a real rollout (Project 8) via GitOps
6. Document real‑world failure modes and troubleshooting patterns

---

## 2. What You Will Learn

- How Argo Rollouts evaluates **AnalysisRun** metrics and gates progression
- How to design **multi‑metric** guardrails (restarts, readiness, image hygiene)
- How to enforce **fail‑closed** behavior using `failureLimit` and `consecutiveErrorLimit`
- How to manage cluster‑scoped reliability policy via Argo CD (GitOps)
- Why some runtime guardrails depend on **observed running state** (not desired YAML)
- How common Kustomize and CRD reference mistakes present in a GitOps platform

---

## 3. Repository Structure

```text
project9-aiops-reliability/
├── README.md
├── docs/
│   ├── architecture.md
│   ├── installation.md
│   ├── analysis-templates.md
│   ├── prometheus-metrics.md
│   ├── rollout-integration.md
│   ├── failure-and-rollback.md
│   ├── troubleshooting.md
│   ├── references.md
│   └── images/
│       └── project9-guardrails-flow.svg
└── manifests/
    └── analysis/
        ├── kustomization.yaml
        ├── project9-analysis-templates-app.yaml
        ├── cluster-restart-rollback-freeze.yaml
        └── cluster-restart-rollback-freeze-prod.yaml
```

---

## 4. High‑Level Architecture

Project 9 is a **control-plane** project. It does not add new workloads. It adds **decision logic** that existing rollouts can reuse.

Flow:

1. A Rollout reaches an **analysis step**
2. Argo Rollouts creates an **AnalysisRun**
3. Prometheus is queried for each metric
4. Each metric result is evaluated (success/failure)
5. If any metric fails (or errors in a fail‑closed configuration), the rollout is blocked

See: `docs/architecture.md` and `docs/images/project9-guardrails-flow.svg`.

---

## 5. Guardrails Implemented

### 5.1 Restart Spike Guardrail
Detects crash‑loop behavior / instability via restarts.

### 5.2 Not‑Ready Pods Guardrail
Blocks progression if pods report not ready during evaluation.

### 5.3 No `:latest` Image Guardrail
Blocks progression if any running pod uses a `:latest` image tag.

> Note: runtime `:latest` detection evaluates **observed pod state**. If a pod never reaches Running (e.g., ImagePullBackOff), the guardrail may not observe it. This is a limitation of runtime observability checks, documented in `docs/failure-and-rollback.md`.

---

## 6. GitOps Model (Argo CD)

Cluster‑scoped templates are managed by an Argo CD Application in this repo:

- `manifests/analysis/project9-analysis-templates-app.yaml`  
- Source path: `manifests/analysis`
- Resources: both ClusterAnalysisTemplates referenced by `kustomization.yaml`

This keeps reliability policy versioned and auditable, and avoids coupling cluster‑scoped resources to application repos.

---

## 7. Integration with Project 8

Project 8 rollouts reference these templates using:

- `templateName: <name>`
- `clusterScope: true`

Dev uses: `project9-restart-rollback-freeze`  
Prod uses: `project9-restart-rollback-freeze-prod`

See: `docs/rollout-integration.md`.

---

## 8. Testing Summary

- Verified AnalysisRuns execute successfully on stable/pinned images
- Validated readiness and restart guardrails behave as expected
- Explored the runtime `:latest` guardrail behavior and documented state‑dependence and edge cases
- Captured real failure modes encountered during implementation (Kustomize patch overwrite + invalid ClusterAnalysisTemplate reference)

See: `docs/failure-and-rollback.md` and `docs/troubleshooting.md`.

---

## 9. References

See `docs/references.md` for official documentation used to implement and validate Project 9.
