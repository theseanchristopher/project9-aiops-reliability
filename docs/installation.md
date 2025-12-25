This project assumes an existing Kubernetes cluster with Prometheus and kube-state-metrics installed (as provided in Project 7).

# Installation

## 1. Prerequisites

This project assumes you already have:

1. A Kubernetes cluster with:
   - Argo Rollouts installed
   - Argo CD installed
   - Prometheus + kube-state-metrics installed (Project 7 stack)
2. A working Rollout (Project 8) that can reference analysis templates

You should also confirm the CRDs exist:

```bash
kubectl get crd | grep -E 'analysisrun|analysistemplate|clusteranalysistemplate|rollout'
```

Typical CRDs include:
- `analysisruns.argoproj.io`
- `clusteranalysistemplates.argoproj.io`
- `rollouts.argoproj.io`

---

## 2. What Gets Installed

This repo installs **cluster-scoped policy objects**:

- `ClusterAnalysisTemplate/project9-restart-rollback-freeze`
- `ClusterAnalysisTemplate/project9-restart-rollback-freeze-prod`

These are applied via Kustomize:

- `manifests/analysis/kustomization.yaml`

---

## 3. GitOps Installation (Recommended)

### 3.1 Apply the Argo CD Application

From this repo root:

```bash
kubectl apply -f manifests/analysis/project9-analysis-templates-app.yaml
```

This creates an Argo CD Application (in the `argocd` namespace) that points to this repo’s `manifests/analysis` path.

### 3.2 Verify Sync

```bash
kubectl get application -n argocd project9-analysis-templates -o jsonpath='{.status.sync.status}{"  "}{.status.health.status}{"  "}{.status.sync.revision}{"\n"}'
```

Expected:
- `Synced`
- `Healthy`
- A commit SHA in `status.sync.revision`

### 3.3 Verify Templates Exist

```bash
kubectl get clusteranalysistemplate | grep project9-restart-rollback-freeze
```

---

## 4. Manual Installation (Not Preferred)

You can apply directly (useful for quick validation), but it’s not the intended model:

```bash
kubectl apply -k manifests/analysis
```

In production GitOps practice, prefer the Argo CD Application so cluster state is driven from Git.

---

## 5. Uninstall

If you installed via Argo CD, delete the Argo CD Application (or remove the app manifest from Git).  
If installed manually:

```bash
kubectl delete -k manifests/analysis
```
