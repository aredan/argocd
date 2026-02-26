# ArgoCD GitOps Repository

This repository contains Kubernetes application configurations managed by ArgoCD. It follows GitOps principles to declaratively manage the cluster state.

> **Note:** This repository is a work in progress and is not yet ready for production use. The idea is to have a place to learn ArgoCD and GitOps in general.

## Repository Structure

```
apps/
├── kube-system/                    # System components
│   └── sealed-secrets/             # Sealed Secrets controller
│
├── monitoring/                     # Monitoring stack
│   └── prometheus-stack/           # Prometheus, Grafana, and Alertmanager
│
├── operators/                      # Cluster operators
│   ├── argo-events/                # Argo Events
│   ├── argo-rollouts/              # Argo Rollouts
│   ├── argo-workflows/             # Argo Workflows
│   ├── cloudflare-operator-system/ # Cloudflare DNS operator
│   ├── external-dns/               # External DNS
│   └── external-secrets/           # External Secrets Operator (Bitwarden)
│
├── services/                       # Application services
│   ├── adguard/                    # AdGuard Home DNS
│   ├── argocd/                     # ArgoCD configuration and bootstrap
│   ├── awx/                        # AWX (Ansible Tower)
│   ├── cert-manager/               # TLS certificate management
│   ├── rancher-versions/           # Rancher versions service
│   └── tailscale/                  # Tailscale VPN
```

## Prerequisites

- Kubernetes cluster with ArgoCD installed
- `kubectl` configured to access your cluster
- `helm` (v3+) for local chart development
- `kubeseal` for managing sealed secrets
- cert-manager installed in the cluster (required for Bitwarden TLS)

## Getting Started

### 1. Bootstrap the Repository

This repository uses an ArgoCD ApplicationSet that auto-discovers all applications under `apps/*/*`. The bootstrap configuration is at `apps/services/argocd/bootstrap-app-set.yaml`.

Apply it to your cluster to start syncing all applications:

```bash
kubectl apply -f apps/services/argocd/bootstrap-app-set.yaml -n argocd
```

### 2. Sealed Secrets Setup

Before deploying applications that require secrets, ensure the Sealed Secrets controller is running. Then seal your secrets:

```bash
# Create a Kubernetes secret
kubectl create secret generic my-secret \
  --from-literal=key=value \
  -n my-namespace \
  --dry-run=client \
  -o yaml > my-secret.yaml

# Fetch the sealing key
kubeseal --fetch-cert > public-key.pem

# Seal the secret
kubeseal --format=yaml --cert=public-key.pem < my-secret.yaml > my-sealed-secret.yaml
```

### 3. External Secrets with Bitwarden

The cluster uses Bitwarden Secrets Manager as the central secrets backend via a `ClusterSecretStore`. See [`apps/operators/external-secrets/README.md`](apps/operators/external-secrets/README.md) for the full setup guide, including:

- Bitwarden access token configuration
- Self-signed TLS certificate chain (cert-manager)
- Creating ExternalSecret resources

## Dependency Management

This repository uses [Renovate](https://docs.renovatebot.com/) to automatically create PRs for Helm chart and container image updates. Configuration is in `renovate.json`.

## Sync Policy

All applications are configured with:

- **Automated sync** with self-heal and pruning
- **Server-side apply** for large CRD resources
- **Infinite retries** with exponential backoff
- **Namespace auto-creation**
