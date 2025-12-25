These templates intentionally fail closed to prioritize safety, immutability, and traceability.

# Analysis Templates

This document explains the **ClusterAnalysisTemplates** provided by Project 9, including the reasoning behind each field.

---

## 1. Template Files

Templates live in:

- `manifests/analysis/cluster-restart-rollback-freeze.yaml`
- `manifests/analysis/cluster-restart-rollback-freeze-prod.yaml`

Both are included by:

- `manifests/analysis/kustomization.yaml`

---

## 2. Why Two Templates?

The dev and prod templates share the same logic but differ in **evaluation conservatism**.

### Dev (fast feedback)
- Shorter confirmation window
- Lower total analysis duration
- Useful for iteration and testing

### Prod (prod-safe)
- Longer confirmation window
- More cautious gating
- Designed to reduce false alarms and flapping

This mirrors real production patterns: dev optimizes for speed; prod optimizes for stability.

---

## 3. Template Arguments

Both templates accept:

- `namespace` — which namespace to evaluate
- `window` — lookback range (e.g., `2m`)
- `threshold` — numeric threshold for restarts (usually `0`)

In this project, templates provide sensible defaults, so application rollouts do not need to pass args unless desired.

---

## 4. Metric: restart-spike

### 4.1 Goal
Detect unexpected restarts during a rollout window.

### 4.2 Query
The query sums increases in restart counters:

- Counter: `kube_pod_container_status_restarts_total`
- Function: `increase(counter[window])`

This returns the number of restarts that occurred during the range vector.
Summing produces a namespace-level signal.

### 4.3 Conditions
- Success: restarts ≤ threshold
- Failure: restarts > threshold

### 4.4 Fail-Closed Defaults
- `failureLimit: 0` — any failure stops analysis
- `consecutiveErrorLimit: 1` — any query error stops analysis

This ensures a Prometheus outage or query mistake does not silently allow progression.

---

## 5. Metric: not-ready-pods

### 5.1 Goal
Block rollouts if any pods report not ready during analysis.

### 5.2 Query
We query readiness from kube-state-metrics:

- Metric: `kube_pod_status_ready{condition="false"}`

We sum to get a count of not-ready pods in the namespace.

### 5.3 Defensive `or vector(0)`
Some queries can return **no series** when there are zero matches. To avoid missing-data ambiguity, we use:

`... or vector(0)`

This ensures the expression returns a numeric 0 when there are no not-ready pods.

### 5.4 Conditions
- Success: result == 0
- Failure: result > 0

---

## 6. Metric: no-latest-image

### 6.1 Goal
Prevent rollouts from using non-immutable images (`:latest`).

### 6.2 Query
We look for pods with an image label matching `:latest$`:

- Metric: `kube_pod_container_info`
- Matcher: `image=~".*:latest$"`

We sum matches in the target namespace and again use `or vector(0)` for a stable numeric result.

### 6.3 Conditions
- Success: result == 0
- Failure: result > 0

### 6.4 Important Limitation (Observed in Testing)
This check is based on **observed running pods**, not desired manifests.

If a `:latest` pod never becomes Running (for example, ImagePullBackOff), runtime metrics may never show the violating pod. In that case, this metric can evaluate to 0.

This is not a PromQL bug — it is a property of runtime state metrics.
We document this as a real engineering limitation in `docs/failure-and-rollback.md`.

---

## 7. Production Variant Differences

The prod template increases conservatism by extending evaluation duration (for example, increasing `count` at the same interval).

This reduces the risk of transient issues passing undetected.
