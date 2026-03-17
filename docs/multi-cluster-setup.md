# Multi-Cluster Setup Guide

This guide describes how to extend this ArgoCD GitOps repository to manage multiple Kubernetes clusters using the officially recommended **ApplicationSet Cluster Generator** approach.

## Overview

Instead of duplicating directories per cluster, ArgoCD's Cluster Generator dynamically discovers registered clusters and fans out applications to each one. The app manifests in `apps/` stay largely the same — the ApplicationSet handles targeting.

---

## Step 1: Register the Remote Cluster in ArgoCD

```bash
# From the management cluster (where ArgoCD runs)
argocd cluster add <context-name> --name <cluster-name>
```

**Why:** ArgoCD can only deploy to clusters it knows about. This command creates a Secret in the `argocd` namespace containing the remote cluster's API server URL and credentials. The local cluster (`https://kubernetes.default.svc`) is always registered implicitly.

**Verify:**

```bash
argocd cluster list
```

You should see both the in-cluster (`https://kubernetes.default.svc`) and the newly added remote cluster.

---

## Step 2: Label Your Clusters

```bash
# Label via the ArgoCD CLI
argocd cluster set <cluster-name> --label env=production
argocd cluster set <cluster-name> --label role=edge

# Or declaratively via the cluster Secret in the argocd namespace
kubectl edit secret <cluster-secret> -n argocd
```

Add labels under `metadata.labels`, for example:

```yaml
metadata:
  labels:
    argocd.argoproj.io/secret-type: cluster
    env: production
    region: us-east
```

**Why:** Labels allow the ApplicationSet Cluster Generator to selectively target clusters. For example, you might deploy monitoring to all clusters but deploy AdGuard only to edge clusters. Without labels, every registered cluster gets every app.

---

## Step 3: Update the ApplicationSet to Use the Cluster Generator

The current `bootstrap-app-set.yaml` hardcodes the destination to the local cluster:

```yaml
# Current (single-cluster)
destination:
  server: https://kubernetes.default.svc
```

Replace the list generator with a Cluster generator in the matrix, and use the `{{ server }}` and `{{ name }}` template variables:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: bootstrap
spec:
  generators:
    - matrix:
        generators:
          - clusters:
              selector:
                matchLabels:
                  env: production
          - git:
              repoURL: https://github.com/aredan/argocd.git
              revision: HEAD
              directories:
                - path: apps/*/*
  template:
    metadata:
      name: '{{ name }}-{{ path[2] }}'
      namespace: argocd
    spec:
      project: default
      source:
        repoURL: https://github.com/aredan/argocd.git
        targetRevision: HEAD
        path: '{{ path }}'
      destination:
        server: '{{ server }}'
        namespace: '{{ path[2] }}'
      syncPolicy:
        automated:
          selfHeal: true
          prune: true
        syncOptions:
          - CreateNamespace=true
          - ServerSideApply=true
          - ApplyOutOfSyncOnly=true
          - RespectIgnoreDifferences=true
          - SkipDryRunOnMissingResource=true
        retry:
          limit: -1
          backoff:
            duration: 5s
            factor: 2
            maxDuration: 5m
```

**Why:** The Cluster Generator automatically discovers every cluster Secret in the `argocd` namespace that matches the label selector. For each cluster, it produces `{{ server }}`, `{{ name }}`, and any label values as template parameters. Combined with the Git generator in a matrix, this creates one Application per app-directory per cluster.

**Key changes from the current setup:**

| Change | Reason |
|--------|--------|
| `clusters:` generator replaces `list:` generator | Dynamically discovers clusters instead of hardcoding |
| `selector.matchLabels` | Controls which clusters receive which apps |
| `{{ server }}` in destination | Targets each discovered cluster |
| `{{ name }}-{{ path[2] }}` in metadata.name | Prevents Application name collisions across clusters |

---

## Step 4: Handle Per-Cluster Configuration Differences

Some apps need different values per cluster (IPs, domains, storage classes, replicas). There are two approaches depending on the app type:

### For Helm-based apps (Chart.yaml)

Add per-cluster values files alongside the existing `values.yaml`:

```
apps/monitoring/prometheus-stack/
├── Chart.yaml
├── values.yaml              # shared/default values
├── values-cluster-a.yaml    # overrides for cluster-a
└── values-cluster-b.yaml    # overrides for cluster-b
```

Then in the ApplicationSet template, conditionally apply the overlay:

```yaml
source:
  repoURL: https://github.com/aredan/argocd.git
  targetRevision: HEAD
  path: '{{ path }}'
  helm:
    valueFiles:
      - values.yaml
      - values-{{ name }}.yaml
```

**Why:** Helm merges values files in order, so `values-<cluster>.yaml` overrides only the fields that differ. The shared `values.yaml` stays as the baseline, avoiding duplication.

### For Kustomize-based apps (kustomization.yaml)

Use an overlay structure:

```
apps/operators/argo-workflows/
├── base/
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── ts-ingress.yaml
│   └── ts-service.yaml
└── overlays/
    ├── cluster-a/
    │   └── kustomization.yaml    # references ../../base + patches
    └── cluster-b/
        └── kustomization.yaml
```

**Why:** Kustomize overlays let you patch specific fields (hostnames, resource limits) per cluster while sharing the base manifests. This is the standard Kustomize pattern for environment variation.

---

## Step 5: Handle Sealed Secrets Across Clusters

Each cluster has its own Sealed Secrets encryption key. A SealedSecret encrypted for cluster-a **cannot be decrypted** by cluster-b.

Options:

### Option A: Share the sealing key (simpler, less secure)

Copy the sealing key Secret from the primary cluster to all others:

```bash
kubectl get secret sealed-secrets-key -n kube-system -o yaml > sealing-key.yaml
# Apply to each cluster
kubectl apply -f sealing-key.yaml --context <other-cluster>
```

**Why:** All clusters can decrypt the same SealedSecrets. Simple, but a compromised key affects all clusters.

### Option B: Per-cluster sealed secrets (more secure)

Create separate sealed YAML files per cluster:

```
apps/operators/external-secrets/templates/
├── bitwarden-access-token-sealed-cluster-a.yaml
└── bitwarden-access-token-sealed-cluster-b.yaml
```

**Why:** Each cluster's sealing key is independent. More operational overhead but better isolation.

### Option C: Rely entirely on External Secrets / Bitwarden (recommended)

Since the cluster already uses Bitwarden via External Secrets, the only SealedSecret needed is the Bitwarden access token to bootstrap the connection. All other secrets come from Bitwarden and are cluster-agnostic.

**Why:** Minimizes SealedSecret usage to a single bootstrap credential per cluster. Everything else is pulled dynamically from Bitwarden, which works identically across clusters.

---

## Step 6: Decide on Topology

### Option A: Management cluster (hub-and-spoke)

One cluster runs ArgoCD and deploys to all clusters, including itself.

```
┌─────────────────┐
│  Management      │──── deploys to ───▶  Cluster A
│  Cluster (ArgoCD)│──── deploys to ───▶  Cluster B
│                  │──── deploys to ───▶  itself
└─────────────────┘
```

**Why:** Single control plane, single source of truth. Easier to manage, but the management cluster is a single point of failure.

### Option B: Per-cluster ArgoCD (federated)

Each cluster runs its own ArgoCD and manages only itself. All point to the same Git repo.

```
Cluster A (ArgoCD) ──── reads ───▶  Git repo (apps/*)
Cluster B (ArgoCD) ──── reads ───▶  Git repo (apps/*)
```

**Why:** No cross-cluster network dependencies. Each cluster is self-contained. Requires filtering (labels or directory paths) so each ArgoCD instance only deploys what belongs to its cluster.

---

## Summary

| Step | What | Why |
|------|------|-----|
| 1 | Register clusters | ArgoCD needs API credentials to deploy remotely |
| 2 | Label clusters | Enables selective targeting via generators |
| 3 | Update ApplicationSet | Switches from hardcoded destination to dynamic cluster discovery |
| 4 | Per-cluster overrides | Handles configuration differences without duplicating manifests |
| 5 | Sealed secrets strategy | Each cluster has its own encryption key |
| 6 | Choose topology | Hub-and-spoke vs federated ArgoCD |

## References

- [ArgoCD Best Practices](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/)
- [ApplicationSet Cluster Generator](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators-Cluster/)
- [ApplicationSet Overview](https://argo-cd.readthedocs.io/en/stable/user-guide/application-set/)
