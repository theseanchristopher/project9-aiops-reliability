# Troubleshooting

This guide is written in the same style as earlier projects: **symptom → likely causes → concrete checks → fixes**.

---

## 1. AnalysisRun Does Not Appear

### Symptoms
- Rollout reaches the analysis step but no AnalysisRun is created
- `kubectl get analysisrun` shows none

### Likely causes
- Rollout is not actually reaching the analysis step (stuck earlier)
- Incorrect analysis step syntax
- Template reference is invalid

### Checks
```bash
kubectl argo rollouts get rollout -n <ns> <rollout> --watch
kubectl get rollout -n <ns> <rollout> -o yaml | grep -n "analysis\|templateName\|clusterScope" -n
```

### Fixes
- Ensure `clusterScope: true` when referencing ClusterAnalysisTemplate
- Ensure `templateName` matches the cluster template name exactly
- Confirm the template exists: `kubectl get clusteranalysistemplate <name>`

---

## 2. Rollout Shows InvalidSpec

### Symptoms
- Rollout reports InvalidSpec
- Describe output references missing fields

### Likely causes
- Kustomize patch overwrote container spec and removed required fields (e.g., image)

### Checks
```bash
kubectl describe rollout -n <ns> <rollout> | sed -n '1,260p'
kustomize build kustomize/overlays/<env> | grep -n "image:" -n
```

### Fixes
- Adjust patches to target only the intended fields
- Prefer JSON6902 patches for surgical edits when strategic merge is too broad
- Validate rendered output before commit

---

## 3. AnalysisRun Fails Due to Prometheus Errors

### Symptoms
- AnalysisRun shows errors fetching metrics
- Consecutive error limit exceeded quickly

### Likely causes
- Prometheus address incorrect
- Prometheus service not reachable from the namespace
- Query typo or missing metric (kube-state-metrics not scraping/exposing)

### Checks
```bash
kubectl get svc -n monitoring | grep prometheus
kubectl describe analysisrun -n <ns> <name> | sed -n '1,260p'
```

### Fixes
- Verify Prometheus address in the template points to the correct service
- Confirm kube-state-metrics is installed and metrics exist
- Fix PromQL typos and validate in Prometheus UI

---

## 4. `:latest` Guardrail Doesn’t Trigger

### Symptoms
- You expect failure but the AnalysisRun passes

### Likely causes
- No Running pod exists with `:latest` at evaluation time
- Rollout flips revisions quickly (violating pod removed before evaluation)
- ImagePullBackOff prevents runtime state from being observed

### Checks
```bash
kubectl get pods -n <ns> -l app=<label> -o wide
kubectl get pods -n <ns> -o jsonpath='{range .items[*]}{.metadata.name}{"  "}{.status.phase}{"  "}{.spec.containers[0].image}{"\n"}{end}' | grep latest
```

### Fixes
- Make the test deterministic: ensure a Running pod with `:latest` exists long enough for evaluation
- Use a longer interval/count during testing
- Document the limitation (this project does)

---

## 5. Argo CD Sync Confusion

### Symptoms
- You updated local files but the cluster does not change
- Auto-sync doesn’t appear to do anything

### Likely causes
- Changes not committed/pushed
- Application tracks a different branch/revision
- Manual sync required for specific operations

### Checks
```bash
git status
git log --oneline -n 3
kubectl get application -n argocd <app> -o jsonpath='{.spec.source.targetRevision}{"  "}{.status.sync.revision}{"  "}{.status.sync.status}{"\n"}'
```

### Fixes
- Commit + push
- Confirm the Argo CD Application tracks the correct branch
- Use UI sync if needed, understanding it applies everything in the app path
