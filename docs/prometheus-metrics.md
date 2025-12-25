# Prometheus Metrics

This document explains the **PromQL** used for each guardrail and why each query is structured the way it is.

---

## 1. Prerequisites

These queries assume:
- Prometheus is scraping kube-state-metrics
- The relevant kube-state-metrics metrics exist

You can confirm availability by searching in Prometheus UI for:
- `kube_pod_container_status_restarts_total`
- `kube_pod_status_ready`
- `kube_pod_container_info`

---

## 2. Metric 1 — Restart Spikes

### 2.1 Why restarts?
In early rollouts, application-level metrics (HTTP 5xx rates, latency histograms) may not exist.
Restarts provide a universally available reliability signal.

A restart during rollout often indicates:
- crash loops
- bad config
- dependency failures
- readiness probe issues that become fatal

### 2.2 Query
```promql
sum(
  increase(
    kube_pod_container_status_restarts_total{namespace="<ns>"}[2m]
  )
)
```

### 2.3 Query reasoning
- `kube_pod_container_status_restarts_total` is a **counter** (monotonic increasing)
- `increase(counter[range])` estimates how much it increased in the range
- `sum(...)` gives a namespace aggregate

### 2.4 Failure semantics
If this returns > 0 in the rollout window, something restarted. For conservative rollouts, that is a failure condition.

---

## 3. Metric 2 — Not Ready Pods

### 3.1 Why readiness?
Readiness is a simple but powerful availability check.
If pods are not ready during a rollout, traffic shifting is risky.

### 3.2 Query
```promql
sum(
  kube_pod_status_ready{condition="false", namespace="<ns>"}
)
or vector(0)
```

### 3.3 Defensive design
- Some queries return no series when there are zero matches.
- `or vector(0)` ensures a stable numeric 0 result.
- This avoids “no data” being interpreted as success implicitly.

---

## 4. Metric 3 — No `:latest` Image

### 4.1 Why `:latest` is risky
`latest` breaks reproducibility:
- the tag can move
- rollbacks are ambiguous
- incident forensics become harder

The goal is to enforce immutable tags (e.g., Git SHA tags).

### 4.2 Query
```promql
sum(
  kube_pod_container_info{namespace="<ns>", image=~".*:latest$"}
)
or vector(0)
```

### 4.3 Limitations
This detects **observed running pods**. It does not inspect desired manifests.
If a violating pod never reaches a state where kube-state-metrics exposes it reliably, the signal can be missed.

This is why production platforms typically enforce image immutability in multiple layers:
- CI policy
- admission policy
- runtime observability guardrails

Project 9 intentionally focuses on the runtime layer.

---

## 5. Practical Debugging Queries

### 5.1 Which pods are not ready?
```promql
kube_pod_status_ready{condition="false", namespace="<ns>"}
```

### 5.2 Which pods have `:latest`?
```promql
kube_pod_container_info{namespace="<ns>", image=~".*:latest$"}
```

### 5.3 Restarts per pod
```promql
increase(kube_pod_container_status_restarts_total{namespace="<ns>"}[2m])
```

These help you correlate analysis failures to concrete objects during troubleshooting.
