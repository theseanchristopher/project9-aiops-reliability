# Rollout Integration (Project 8)

This document explains how Project 8 rollouts reference Project 9 ClusterAnalysisTemplates, and how to verify correct wiring.

---

## 1. Where the Integration Lives

Project 9 hosts cluster-scoped templates.

Project 8 hosts Rollouts that reference them in canary steps.

In Project 8, the reference appears in the Rollout canary steps:

```yaml
- analysis:
    templates:
      - templateName: project9-restart-rollback-freeze
        clusterScope: true
```

Prod references the prod-safe template:

```yaml
- analysis:
    templates:
      - templateName: project9-restart-rollback-freeze-prod
        clusterScope: true
```

---

## 2. Why `clusterScope: true` Matters

A common misconfiguration is referencing a ClusterAnalysisTemplate without setting `clusterScope: true`.
In that case Argo Rollouts searches for a namespaced AnalysisTemplate and cannot find it.

Correct wiring requires:
- `templateName: <cluster template name>`
- `clusterScope: true`

---

## 3. How to Verify the Rollout References

### 3.1 Dev
```bash
kubectl get rollout -n project8-dev project8-nginx-rollout -o yaml | grep -n "analysis\|templateName\|clusterScope" -n
```

### 3.2 Prod
```bash
kubectl get rollout -n project8-prod project8-nginx-rollout -o yaml | grep -n "analysis\|templateName\|clusterScope" -n
```

---

## 4. Confirm Templates Exist

```bash
kubectl get clusteranalysistemplate project9-restart-rollback-freeze -o yaml | head
kubectl get clusteranalysistemplate project9-restart-rollback-freeze-prod -o yaml | head
```

---

## 5. Observing AnalysisRuns

When the rollout reaches the analysis step, you should see an AnalysisRun created in the rollout namespace:

```bash
kubectl get analysisrun -n project8-dev --sort-by=.metadata.creationTimestamp | tail -n 10
```

Describe it:

```bash
kubectl describe analysisrun -n project8-dev <name> | sed -n '1,260p'
```

The AnalysisRun should list each metric (restart-spike, not-ready-pods, no-latest-image) and show evaluation results.

---

## 6. GitOps Notes

Because these rollouts are synced via Argo CD, remember:

- A local file edit does nothing until committed and pushed
- Auto-sync only triggers on new commits or drift
- Manual sync applies everything in the Application path (even prod resources)

Project 9 documents this behavior because it matters for safe platform operations.
