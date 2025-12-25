# Failure & Rollback Behavior

This project is explicitly about **how rollouts should behave under failure** and **why**.

---

## 1. Fail‑Closed: Default to Safety

Fail‑closed means:

- If the system sees unsafe metrics → fail
- If the system cannot evaluate metrics reliably → fail
- If the system has unexpected errors → fail

This project implements fail‑closed behavior using:
- `failureLimit: 0`
- `consecutiveErrorLimit: 1`
- explicit success/failure conditions

The goal is to avoid “silent success” caused by missing observability.

---

## 2. What Happens When a Metric Fails

When any metric fails, the AnalysisRun fails.
When the AnalysisRun fails, the Rollout blocks at the analysis step (or aborts/rolls back depending on your rollout strategy).

This prevents additional traffic shifting until safety is proven again.

---

## 3. Interpreting Failures

### 3.1 Restart failures
- Indicates runtime instability
- Often correlates with configuration errors, dependency issues, or bad builds

### 3.2 Readiness failures
- Indicates the rollout cannot safely receive traffic
- Often correlates with failing readiness probes, resource constraints, or startup time regressions

### 3.3 `:latest` failures
- Indicates a reproducibility violation
- Prevents non-immutable deployments from proceeding

---

## 4. The “Runtime State” Limitation (Observed During Testing)

The image hygiene guardrail is implemented using runtime pod metrics.
That means it evaluates **what is running**, not what was intended.

Real behavior observed in Project 9 integration work:

- If a `:latest` pod never becomes Running (e.g., ImagePullBackOff), kube-state-metrics may not produce the expected series.
- If a rollout flips quickly between revisions, the violating pod may not exist long enough to be observed during the analysis interval.

In both cases, the guardrail can evaluate to 0 and pass — not because the guardrail is “wrong,” but because runtime observability only sees state that successfully materializes.

This limitation is important because it explains why production platforms enforce immutability at multiple layers:

- CI policies (disallow :latest tagging)
- admission policies (OPA/Kyverno)
- runtime guardrails (Project 9)

Project 9 intentionally documents this as a learning point.

---

## 5. Failure Modes Encountered in Implementation

### 5.1 Kustomize strategic merge overwrite (container spec lost)

A patch replaced the entire container spec without preserving the image field.
Kubernetes accepted the object, but the Rollout spec became invalid (missing required fields).

Outcome:
- Rollout failed closed before unsafe deployment

Lesson:
- Validate rendered manifests (`kustomize build`) when patching complex specs.

### 5.2 ClusterAnalysisTemplate reference field error

An invalid nested field name was used when referencing a ClusterAnalysisTemplate.
Kubernetes accepted the unknown field.
Argo Rollouts ignored it.
The rollout ended up with no valid analysis templates and failed.

Outcome:
- Fail-closed protection prevented progression

Lesson:
- Controllers can ignore unknown fields; use official schemas and verify behavior in describe output.

---

## 6. Recommended Rollback / Recovery Actions

When a rollout blocks:

1. Inspect AnalysisRun status and metric outputs
2. Confirm which metric failed and why
3. Fix the underlying issue (image, readiness, restarts)
4. Commit/push the fix (GitOps)
5. Allow Argo CD to sync and Argo Rollouts to continue

This mirrors real platform operations: fix cause, re-evaluate, then proceed.
