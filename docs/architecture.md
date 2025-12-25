This project introduces a cluster-scoped reliability policy layer that is independent of any single application.

# Architecture

## 1. Overview

Project 9 introduces a **reliability policy layer** enforced during progressive delivery.  
Instead of changing the data plane, we change *how the rollout decides to proceed*.

Core components involved:

- **Argo Rollouts**: executes canary steps and creates AnalysisRuns
- **Prometheus**: provides runtime metrics (kube-state-metrics)
- **Argo CD**: GitOps sync for cluster-scoped templates
- **ClusterAnalysisTemplate**: reusable analysis policy

This project is intentionally **control-plane heavy** and **platform-oriented**.

---

## 2. Control Flow

1. A Rollout executes canary steps.
2. When it reaches an `analysis` step, Argo Rollouts creates an **AnalysisRun**.
3. The AnalysisRun evaluates one or more metrics on a schedule (`interval`, `count`).
4. Each metric query returns a numeric result (or an error).
5. Success/failure conditions are evaluated.
6. If any metric fails (or errors, under fail‑closed settings), analysis fails.
7. If analysis fails, the rollout is blocked/aborted according to the rollout strategy.

---

## 3. Why ClusterAnalysisTemplate

We use **ClusterAnalysisTemplate** (cluster-scoped) instead of AnalysisTemplate (namespaced) because:

- Multiple environments/namespaces can reuse the same policy
- Argo Rollouts can reference it explicitly using `clusterScope: true`
- GitOps management is simpler (a dedicated “policy app” can sync cluster-wide templates)

---

## 4. Environment Awareness

Project 9 deploys two templates:

- **Dev template**: shorter evaluation for fast feedback
- **Prod template**: longer evaluation for conservative confirmation

These templates share the same logic; only timing parameters differ.

---

## 5. Diagram

See `docs/images/project9-guardrails-flow.svg` for a minimal flow diagram.
