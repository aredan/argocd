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
- `kubeseal` CLI for creating sealed secrets

## Getting Started

### 1. Bootstrap the Repository

This repository uses an ArgoCD ApplicationSet that auto-discovers all applications under `apps/*/*`. The bootstrap configuration is at `apps/services/argocd/bootstrap-app-set.yaml`.

Apply it to your cluster to start syncing all applications:

```bash
kubectl apply -f apps/services/argocd/bootstrap-app-set.yaml -n argocd
```

This single command deploys **everything** — including the Sealed Secrets controller (`apps/kube-system/sealed-secrets/`), cert-manager (`apps/services/cert-manager/`), and all other applications. The ApplicationSet uses infinite retries with exponential backoff, so resources that depend on controllers (like SealedSecrets or Certificates) will keep retrying until their controllers are ready.

### 2. Initial Cluster Bootstrapping Order

On a fresh cluster, the applications will self-resolve in this order:

1. **Sealed Secrets controller** — deploys into `kube-system`, starts unsealing SealedSecret resources
2. **cert-manager** — deploys CRDs and controller, starts issuing certificates
3. **External Secrets Operator** — deploys the operator, Bitwarden SDK server (waits for TLS certs via sync-waves), and ClusterSecretStore
4. **Everything else** — applications that depend on ExternalSecrets (Cloudflare, Tailscale, etc.) will retry until secrets are available

No manual intervention is needed after the bootstrap command. If some apps show errors initially, they will self-heal once their dependencies are running.

### 3. Sealing New Secrets

To add a new sealed secret to the repository, the Sealed Secrets controller must already be running in the cluster:

```bash
# Create a Kubernetes secret
kubectl create secret generic my-secret \
  --from-literal=key=value \
  -n my-namespace \
  --dry-run=client \
  -o yaml > my-secret.yaml

# Fetch the sealing key from the cluster
kubeseal --fetch-cert > public-key.pem

# Seal the secret
kubeseal --format=yaml --cert=public-key.pem < my-secret.yaml > my-sealed-secret.yaml
```

Commit the sealed YAML to the repo. **Never** commit the unsealed secret.

### 4. External Secrets with Bitwarden

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
