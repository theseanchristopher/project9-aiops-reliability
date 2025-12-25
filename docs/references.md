# References

This project follows the Project 7–8 standard: references include *why they matter* for this implementation.

---

## 1. Argo Rollouts

**Argo Rollouts Documentation**  
https://argo-rollouts.readthedocs.io/

Used to:
- Understand how AnalysisRuns are created from rollout steps
- Configure metric evaluation behavior (interval/count/failureLimit)
- Validate ClusterAnalysisTemplate referencing using `clusterScope: true`
- Interpret how analysis failures gate rollout progression

Relevant topics:
- Canary Strategy
- Analysis Templates
- AnalysisRun behavior

---

## 2. Prometheus / PromQL

**Prometheus Querying (PromQL)**  
https://prometheus.io/docs/prometheus/latest/querying/

Used to:
- Design correct range-vector queries (`increase()` on counters)
- Avoid incorrect assumptions about counters vs gauges
- Structure queries that are safe under missing data (`or vector(0)`)

Relevant concepts:
- Range vectors
- Counters and `increase()`
- Query evaluation behavior

---

## 3. kube-state-metrics

**kube-state-metrics**  
https://github.com/kubernetes/kube-state-metrics

Used to:
- Identify which Kubernetes state metrics are available for analysis
- Understand labels exposed for images and readiness
- Confirm the semantics of the metrics used in guardrails

Metrics used in this project:
- `kube_pod_container_status_restarts_total`
- `kube_pod_status_ready`
- `kube_pod_container_info`

---

## 4. Argo CD

**Argo CD Documentation**  
https://argo-cd.readthedocs.io/

Used to:
- Manage cluster-scoped policy resources via GitOps
- Understand auto-sync, self-heal, and manual sync implications
- Diagnose why “local changes” do not apply until pushed

Relevant topics:
- Application CRD
- Sync Policy
- Cluster-scoped resource management

---

## 5. Kubernetes API Conventions

**Kubernetes API Concepts**  
https://kubernetes.io/docs/concepts/overview/kubernetes-api/

Used to:
- Explain why unknown fields can be accepted by the API server
- Document how controllers may ignore unknown fields silently
- Frame failure modes encountered with invalid template reference fields

---

## 6. SRE / Reliability Concepts

**Google SRE Book — Monitoring Distributed Systems**  
https://sre.google/sre-book/monitoring-distributed-systems/

Used to:
- Frame fail-closed vs fail-open decisions
- Justify conservative defaults in reliability guardrails
- Explain trade-offs (false negatives vs false positives)

---

## 7. Summary

Project 9 combines these sources to implement a practical, GitOps-managed reliability policy layer that:
- gates progressive delivery using runtime metrics
- defaults to safety on ambiguity
- documents controller and GitOps edge cases honestly
