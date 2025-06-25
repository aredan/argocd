# External Secrets Operator with Bitwarden

This directory contains the configuration for deploying the External Secrets Operator (ESO) with Bitwarden Secrets Manager as the secrets backend, secured using Sealed Secrets.

## Prerequisites

1. Bitwarden organization with Secrets Manager enabled
2. Bitwarden CLI (bw) version 2025.5.0 or later
3. `kubeseal` CLI tool installed
4. Kubernetes cluster with Sealed Secrets controller installed
5. ArgoCD ApplicationSet configured to deploy from this repository

## Finding Your Secret ID

### Using Bitwarden CLI

1. **Install or update Bitwarden CLI**:
   ```bash
   # Using Homebrew (macOS)
   brew install bitwarden-cli
   
   # Or download from: https://github.com/bitwarden/cli/releases
   ```

2. **Log in and unlock your vault**:
   ```bash
   # Log in
   bw login
   
   # Unlock your vault
   bw unlock
   ```

3. **List organizations and find your organization ID**:
   ```bash
   bw list organizations
   ```

4. **List secrets in your organization**:
   ```bash
   bw list secrets --organizationid YOUR_ORG_ID
   ```

### Using Web Interface

1. Go to [Bitwarden Web Vault](https://vault.bitwarden.com/)
2. Log in to your account
3. Click on "Secrets Manager" in the left sidebar
4. Find your secret in the list
5. Click on it to view its details
6. Copy the "Secret ID" (UUID format)

## Configuration

### 1. Update SecretStore

Edit `bitwarden-secret-store.yaml` with your Bitwarden configuration:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: bitwarden-secret-store
  namespace: external-secrets
spec:
  provider:
    http:
      url: https://api.bitwarden.com
      headers:
        Authorization: "Bearer ${BITWARDEN_API_KEY}"
      secrets:
      - name: "bitwarden-secrets"
        path: "/secrets/{{ .secretKey }}"
        method: GET
        headers:
          Accept: "application/json"
  template:
    metadata:
      annotations:
        sealedsecrets.bitnami.com/managed: "true"
    data:
      BITWARDEN_API_KEY:
        secretKeyRef:
          name: bitwarden-credentials
          key: client-secret
```

### 2. Create ExternalSecret

Create a new file (e.g., `my-app-secret.yaml`) with the following content:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-app-secret
  namespace: your-namespace
spec:
  refreshInterval: 15m
  secretStoreRef:
    name: bitwarden-secret-store
    kind: SecretStore
  target:
    name: my-app-credentials
    creationPolicy: Owner
  data:
    - secretKey: API_KEY
      remoteRef:
        key: /secrets/YOUR_SECRET_ID_HERE  # Replace with your secret ID
        property: value  # Or the specific property if it's a JSON object
```

### 3. Seal Your Bitwarden Credentials

1. Create a Kubernetes secret with your Bitwarden credentials:
   ```bash
   kubectl create secret generic bitwarden-credentials \
     --from-literal=client-id=your-client-id \
     --from-literal=client-secret=your-client-secret \
     -n external-secrets \
     --dry-run=client \
     -o yaml > bitwarden-credentials.yaml
   ```

2. Seal the secret:
   ```bash
   kubeseal --format=yaml --cert=public-key.pem < bitwarden-credentials.yaml > bitwarden-credentials-sealed.yaml
   ```

3. Commit the sealed secret to your repository

## Troubleshooting

- **Secret not found**: Verify the secret ID and that your API key has the correct permissions
- **Authentication errors**: Check that your Bitwarden credentials are correct and have not expired
- **Sync issues**: Ensure the External Secrets Operator pod is running and check its logs

## Security Notes

- Never commit unsealed secrets to the repository
- Rotate your Bitwarden API keys regularly
- Use the principle of least privilege for API key permissions

## Directory Structure

```
apps/
├── external-secrets/          # Namespace
│   └── operator/              # External Secrets Operator
│       ├── Chart.yaml         # Helm chart for ESO
│       ├── values.yaml        # Configuration values
│       ├── README.md          # This file
│       ├── bitwarden-secret-store.yaml  # Bitwarden configuration
│       └── bitwarden-credentials-sealed.yaml  # Sealed credentials
├── kube-system/               # System components
│   └── sealed-secrets/        # Sealed Secrets controller
└── monitoring/                # Monitoring components
    └── prometheus-stack/      # Prometheus stack
```

## Example Workflow

1. Create a secret in Bitwarden Secrets Manager
2. Find the secret ID using the CLI or web interface
3. Create an ExternalSecret manifest referencing the secret ID
4. Commit and push your changes
5. ArgoCD will deploy the ExternalSecret
6. The External Secrets Operator will sync the secret from Bitwarden to your cluster

## Prerequisites

1. **Bitwarden Account**:
   - Organization with Secrets Manager enabled
   - API credentials (client ID and client secret)
   - Secrets already created in Bitwarden Secrets Manager

2. **Local Tools**:
   - `kubectl` configured with cluster access
   - `helm` (v3+)
   - `kubeseal` for sealing secrets
   - `bw` (Bitwarden CLI) version 2025.5.0 or later

3. **Cluster Components**:
   - Kubernetes cluster with ArgoCD installed
   - Sealed Secrets controller in `kube-system` namespace
   - External Secrets Operator (will be deployed by this chart)

4. **GitOps Setup**:
   - ArgoCD ApplicationSet configured to scan `apps/*/*`
   - Proper RBAC permissions for ArgoCD to manage resources

## Installation

### 1. Install Sealed Secrets (if not already installed)

Ensure the Sealed Secrets controller is installed in your cluster. See the `kube-system/sealed-secrets` directory for details.

### 2. Seal Your Bitwarden Credentials

1. Create a Kubernetes secret with your Bitwarden API credentials:
   ```bash
   kubectl create secret generic bitwarden-credentials \
     --from-literal=client-id=your-client-id \
     --from-literal=client-secret=your-client-secret \
     -n external-secrets \
     --dry-run=client \
     -o yaml > bitwarden-credentials.yaml
   ```

2. Get the public key from the Sealed Secrets controller:
   ```bash
   kubeseal --fetch-cert > public-key.pem
   ```

3. Seal the secret:
   ```bash
   kubeseal --format=yaml --cert=public-key.pem < bitwarden-credentials.yaml > bitwarden-credentials-sealed.yaml
   ```

### 3. Configure Bitwarden Integration

1. Update `bitwarden-secret-store.yaml` with your Bitwarden configuration
2. Create ExternalSecret manifests for each secret you want to sync
3. Commit and push your changes

### 4. Deploy

ArgoCD will automatically detect and deploy the External Secrets Operator and your configuration based on the ApplicationSet.

## Usage

Create ExternalSecret resources to sync secrets from Bitwarden to Kubernetes. Example:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-app-secrets
  namespace: my-namespace
spec:
  refreshInterval: 15m
  secretStoreRef:
    name: bitwarden-secret-store
    kind: SecretStore
  target:
    name: my-app-secrets
    creationPolicy: Owner
  data:
    - secretKey: DATABASE_URL
      remoteRef:
        key: /path/to/your/bitwarden/secret
        property: DATABASE_URL
```

## Notes

- The External Secrets Operator will automatically sync the secrets from Bitwarden to Kubernetes
- Secrets are automatically refreshed based on the `refreshInterval`
- Make sure to properly secure access to the Bitwarden SecretStore resource
