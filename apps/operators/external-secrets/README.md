# External Secrets Operator with Bitwarden

This directory contains the configuration for deploying the External Secrets Operator (ESO) with Bitwarden Secrets Manager integration. The operator is deployed in the `external-secrets` namespace and uses Sealed Secrets for encrypting credentials at rest in Git.

## Prerequisites

- Bitwarden organization with Secrets Manager enabled
- Bitwarden machine account access token
- `kubeseal` CLI tool installed
- Sealed Secrets controller running in the cluster (`kube-system` namespace)
- cert-manager installed and running (required for TLS certificate chain)
- ArgoCD installed and configured to manage this repository

## Architecture

The setup uses a self-signed certificate chain managed by cert-manager:

1. **Bootstrap ClusterIssuer** (self-signed) creates a CA certificate
2. **CA ClusterIssuer** uses that CA to issue leaf certificates
3. **Leaf certificate** (`bitwarden-tls-certs`) is mounted by the Bitwarden SDK server for TLS
4. **Client certificate** (`bitwarden-css-certs`) is used for client authentication
5. **ClusterSecretStore** connects to the SDK server over HTTPS and trusts it via the CA

ArgoCD sync-wave annotations enforce the correct deployment order:

| Wave | Resource |
|------|----------|
| `-3` | Bootstrap ClusterIssuer (self-signed) |
| `-2` | Bootstrap CA Certificate |
| `-1` | CA ClusterIssuer |
| `0`  | Leaf TLS + client certificates, SDK server |
| `1`  | ClusterSecretStore |

## Getting Started

### 1. Create the Bitwarden Access Token

Generate a machine account access token in Bitwarden Secrets Manager, then create a Kubernetes secret:

```bash
kubectl create secret generic bitwarden-access-token \
  --from-literal=token=YOUR_ACCESS_TOKEN \
  -n external-secrets \
  --dry-run=client \
  -o yaml > bitwarden-access-token.yaml
```

### 2. Seal the Access Token

Fetch the Sealed Secrets public key and seal the secret:

```bash
kubeseal --fetch-cert > public-key.pem

kubeseal --format=yaml \
  --cert=public-key.pem \
  --scope=namespace-wide \
  < bitwarden-access-token.yaml \
  > bitwarden-access-token-sealed.yaml
```

Replace the contents of `templates/bitwarden-access-token-sealed.yaml` with the output.

### 3. Configure the ClusterSecretStore

The `templates/bitwarden-secret-store.yaml` file contains the ClusterSecretStore configuration. Update these fields with your Bitwarden organization details:

- `organizationID` — your Bitwarden organization UUID
- `projectID` — your Bitwarden project UUID

These values are also stored as sealed secrets in `templates/bitwarden-ids-sealed.yaml`.

### 4. Deploy

Commit and push your changes. ArgoCD will automatically detect and deploy the configuration via the ApplicationSet.

## Creating a New ExternalSecret

To sync a secret from Bitwarden into Kubernetes, create a YAML file in the target application directory:

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: my-app-secret
  namespace: my-namespace
spec:
  refreshInterval: "1h"
  secretStoreRef:
    name: bitwarden-secretsmanager
    kind: ClusterSecretStore
  data:
    - secretKey: API_KEY
      remoteRef:
        key: my-bitwarden-secret-name
        conversionStrategy: Default
        decodingStrategy: None
        metadataPolicy: None
```

Commit and push. ArgoCD will apply it automatically.

## Troubleshooting

### Common Issues

- **Certificate errors / TLS handshake failures**:
  - Verify cert-manager is running and certificates are issued:
    ```bash
    kubectl get certificates -n external-secrets
    kubectl get certificates -n cert-manager
    ```
  - Check the `ca.crt` key exists in the TLS secret:
    ```bash
    kubectl get secret bitwarden-tls-certs -n external-secrets -o jsonpath='{.data.ca\.crt}' | base64 -d | openssl x509 -text -noout
    ```
  - Check what certificate the SDK server is presenting:
    ```bash
    kubectl exec -n external-secrets deploy/bitwarden-sdk-server -- \
      openssl s_client -connect localhost:9998 -showcerts
    ```

- **Secret not found**:
  - Verify the secret name in Bitwarden matches the `remoteRef.key`
  - Check that the access token has permissions for the project
  - Ensure the secret belongs to the correct organization and project

- **Authentication errors**:
  - Verify the sealed secret was created correctly:
    ```bash
    kubectl get secret bitwarden-access-token -n external-secrets
    ```
  - Check if the access token has expired or been revoked

- **Sync issues**:
  ```bash
  # Check operator logs
  kubectl logs -n external-secrets -l app.kubernetes.io/name=external-secrets

  # Check SDK server logs
  kubectl logs -n external-secrets -l app.kubernetes.io/name=bitwarden-sdk-server

  # Check the status of your ExternalSecret
  kubectl get externalsecret -n <namespace> <secret-name> -o yaml
  ```

## Security Notes

- Never commit unsealed secrets to the repository
- Rotate your Bitwarden access tokens regularly (every 90 days recommended)
- Use the principle of least privilege for access token permissions
- Monitor the External Secrets Operator logs for synchronization issues
- TLS certificates auto-rotate every 90 days (renewal starts 3 days before expiry)
