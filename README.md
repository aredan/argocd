# ArgoCD GitOps Repository

This repository contains the Kubernetes application configurations managed by ArgoCD. It follows GitOps principles to declaratively manage your Kubernetes cluster's state.

Note: This repository is a work in progress and is not yet ready for production use.

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

## Applications

### Sealed Secrets Controller
- **Path**: `apps/kube-system/sealed-secrets`
- **Description**: Encrypts Kubernetes Secrets for safe storage in Git
- **Namespace**: `kube-system`

### External Secrets Operator
- **Path**: `apps/operators/external-secrets`
- **Description**: Syncs secrets from external secret stores (like Bitwarden) to Kubernetes
- **Namespace**: `external-secrets`

### Monitoring Stack
- **Path**: `apps/monitoring/prometheus-stack`
- **Description**: Prometheus, Grafana, and Alertmanager for cluster monitoring
- **Namespace**: `monitoring`

### AdGuard Home
- **Path**: `apps/services/adguard`
- **Description**: Network-wide ad blocking and DNS server
- **Namespace**: `services`

### Cert Manager
- **Path**: `apps/services/cert-manager`
- **Description**: Manages TLS certificates for your applications
- **Namespace**: `services`

### Tailscale
- **Path**: `apps/services/tailscale`
- **Description**: Secure network connectivity and VPN solution
- **Namespace**: `services`

## Best Practices

1. **Secrets Management**:
   - Never commit unencrypted secrets to the repository
   - Use Sealed Secrets for sensitive data
   - Consider External Secrets for dynamic secret management

2. **Application Structure**:
   - Follow the `apps/{namespace}/{app-name}` pattern
   - Include a `README.md` in each application directory
   - Document all required values and configurations

3. **GitOps Practices**:
   - Use pull requests for all changes
   - Require code reviews for production changes
   - Use ArgoCD sync waves for ordering when necessary

## Troubleshooting

- **Application not syncing**: Check the ArgoCD UI for sync errors
- **Secrets not being created**: Verify the Sealed Secrets controller is running and the sealed secret is properly formatted
- **External Secrets not updating**: Check the External Secrets Operator logs and verify the refresh interval
