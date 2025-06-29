# External Secrets Operator with Bitwarden

This directory contains the configuration for deploying the External Secrets Operator (ESO) with Bitwarden Secrets Manager integration, secured using Sealed Secrets. The operator is deployed in the `external-secrets` namespace.

## Prerequisites

- Bitwarden organization with Secrets Manager enabled
- Bitwarden API credentials (client ID and client secret)
- `kubeseal` CLI tool installed
- Sealed Secrets controller running in the cluster
- ArgoCD installed and configured to manage this repository

## Getting Started

### 1. Configure Bitwarden Secret Store

The `bitwarden-secret-store.yaml` file contains the SecretStore configuration for Bitwarden. It's pre-configured to work with the sealed Bitwarden credentials.

### 2. Add Your Secrets to Bitwarden

1. Log in to the [Bitwarden Web Vault](https://vault.bitwarden.com/)
2. Navigate to "Secrets Manager"
3. Create a new secret or use an existing one
4. Note the Secret ID (UUID) of your secret

## Creating a New External Secret

1. Create a new file in the `external-secrets` directory (e.g., `my-app-secret.yaml`):

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-app-secret
  namespace: your-namespace  # Update with your target namespace
spec:
  refreshInterval: 15m
  secretStoreRef:
    name: bitwarden-secret-store
    kind: SecretStore
  target:
    name: my-app-credentials  # Name of the Kubernetes secret that will be created
  data:
    - secretKey: API_KEY      # Key in the Kubernetes secret
      remoteRef:
        key: /secrets/YOUR_SECRET_ID_HERE  # Replace with your Bitwarden secret ID
        property: value                    # Or specific JSON property if applicable
```

2. Commit and push your changes. ArgoCD will automatically apply the changes.

## Troubleshooting

### Common Issues

- **Secret not found**:
  - Verify the secret ID in Bitwarden
  - Check that the API key has the correct permissions
  - Ensure the secret is in the correct organization

- **Authentication errors**:
  - Verify Bitwarden credentials in the sealed secret
  - Check if the API key has expired
  - Ensure the service account has the correct permissions

- **Sync issues**:
  ```bash
  # Check operator logs
  kubectl logs -n external-secrets -l app.kubernetes.io/name=external-secrets
  
  # Check the status of your ExternalSecret
  kubectl get externalsecret -n <namespace> <secret-name> -o yaml
  ```

## Security Notes

- Never commit unsealed secrets to the repository
- Rotate your Bitwarden API keys regularly (every 90 days recommended)
- Use the principle of least privilege for API key permissions
- Monitor the External Secrets Operator logs for any synchronization issues
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
