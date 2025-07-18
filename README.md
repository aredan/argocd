# ArgoCD GitOps Repository

This repository contains the Kubernetes application configurations managed by ArgoCD. It follows GitOps principles to declaratively manage your Kubernetes cluster's state.

Note: This repository is a work in progress and is not yet ready for production use. The idea is to have a place where to learn ArgoCD and GitOps in general.

## Repository Structure

```
apps/
├── kube-system/               # System components
│   └── sealed-secrets/        # Sealed Secrets controller
│
├── monitoring/                # Monitoring stack
│   └── prometheus-stack/      # Prometheus, Grafana, and Alertmanager
│
├── operators/                 # Cluster operators
│   └── external-secrets/      # External Secrets Operator
│
├── services/                 # Application services
│   ├── adguard/              # AdGuard Home
│   ├── argocd/               # ArgoCD configuration
│   ├── cert-manager/         # Cert Manager
│   └── tailscale/            # Tailscale VPN

```

## Prerequisites

- Kubernetes cluster with ArgoCD installed
- `kubectl` configured to access your cluster
- `helm` (v3+) for local chart development
- `kubeseal` for managing sealed secrets
- `kustomize` for managing Kubernetes resources

## Getting Started

### 1. Bootstrap the Repository

This repository uses an ArgoCD ApplicationSet for bootstrapping applications. The bootstrap configuration is located at `apps/services/argocd/bootstrap-app-set.yaml`.

### 2. Sealed Secrets Setup

Before deploying applications that require secrets:

1. Ensure the Sealed Secrets controller is installed
2. Seal your secrets using `kubeseal`:

```bash
# Create a Kubernetes secret
kubectl create secret generic my-secret \
  --from-literal=username=admin \
  --from-literal=password=secret \
  -n my-namespace \
  --dry-run=client \
  -o yaml > my-secret.yaml

# Seal the secret
kubeseal --format=yaml --cert=public-key.pem < my-secret.yaml > my-sealed-secret.yaml
```

### 3. External Secrets with Bitwarden

To use Bitwarden as a secrets backend:

1. Update `apps/operators/external-secrets/bitwarden-secret-store.yaml` with your Bitwarden API configuration
2. Seal your Bitwarden credentials (see the operator's README)
3. Commit the sealed credentials to the repository

