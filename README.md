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

### 2. Restore the Sealed Secrets Key (Existing Cluster Migration)

The sealed secrets in this repo (like `bitwarden-access-token-sealed.yaml`) are encrypted with a specific sealing key. On a **new cluster**, the Sealed Secrets controller generates a new key, so existing sealed secrets **will fail to unseal**.

You have two options:

**Option A: Restore the sealing key from the previous cluster (recommended)**

Before or right after bootstrapping, restore the backup of the sealing key:

```bash
# From the old cluster (save this somewhere safe)
kubectl get secret sealed-secrets-key -n kube-system -o yaml > sealing-key-backup.yaml

# On the new cluster (apply before the controller starts, or restart it after)
kubectl apply -f sealing-key-backup.yaml
kubectl rollout restart deployment sealed-secrets-controller -n kube-system
```

**Option B: Re-seal all secrets with the new key**

After the controller is running on the new cluster, re-seal every secret:

```bash
# Fetch the new cluster's sealing key
kubeseal --fetch-cert > public-key.pem

# Re-seal the Bitwarden access token
kubectl create secret generic bitwarden-access-token \
  --from-literal=token=YOUR_BITWARDEN_ACCESS_TOKEN \
  -n external-secrets \
  --dry-run=client \
  -o yaml | \
  kubeseal --format=yaml --cert=public-key.pem --scope=namespace-wide \
  > apps/operators/external-secrets/templates/bitwarden-access-token-sealed.yaml

# Re-seal the Bitwarden IDs
kubectl create secret generic bitwarden-ids \
  --from-literal=organizationid=YOUR_ORG_ID \
  --from-literal=projectid=YOUR_PROJECT_ID \
  -n external-secrets \
  --dry-run=client \
  -o yaml | \
  kubeseal --format=yaml --cert=public-key.pem --scope=namespace-wide \
  > apps/operators/external-secrets/templates/bitwarden-ids-sealed.yaml
```

Commit and push the re-sealed files.

### 3. Bootstrapping Order

After the bootstrap command and sealing key are in place, applications self-resolve in this order:

1. **Sealed Secrets controller** (`kube-system`) — starts unsealing SealedSecret resources
2. **cert-manager** — deploys CRDs and controller, starts issuing certificates
3. **External Secrets Operator** (`external-secrets`) — deploys the operator, then via sync-waves:
   - Wave `-3` to `0`: TLS certificate chain (issuers and certificates)
   - Wave `0`: Bitwarden SDK server (mounts TLS certs)
   - Wave `1`: ClusterSecretStore (connects to SDK server)
4. **All other applications** — Cloudflare, Tailscale, external-dns, etc. retry until their ExternalSecrets are available

All applications use infinite retries with exponential backoff. If some apps show errors initially, they will self-heal once their dependencies are running.

### 4. Sealing New Secrets

To add a new sealed secret to the repository, the Sealed Secrets controller must already be running:

```bash
# Fetch the sealing key from the cluster
kubeseal --fetch-cert > public-key.pem

# Create and seal a secret (example: a new secret in the external-secrets namespace)
kubectl create secret generic my-new-secret \
  --from-literal=key=value \
  -n external-secrets \
  --dry-run=client \
  -o yaml | \
  kubeseal --format=yaml --cert=public-key.pem --scope=namespace-wide \
  > apps/operators/external-secrets/templates/my-new-secret-sealed.yaml
```

Commit the sealed YAML to the repo. **Never** commit the unsealed secret or the `public-key.pem` file.

### 5. External Secrets with Bitwarden

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
