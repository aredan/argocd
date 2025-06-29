# Sealed Secrets Controller

This directory contains the Helm chart for deploying the Bitnami Sealed Secrets controller in the `kube-system` namespace. Sealed Secrets allows you to encrypt your Kubernetes Secrets so they can be safely stored in version control.

## Prerequisites

- `kubectl` with cluster admin access
- `kubeseal` CLI tool installed
- Helm v3+
- ArgoCD installed in the cluster

## Installation

The Sealed Secrets controller is automatically deployed by ArgoCD using the Helm chart in this directory. No manual installation is required.

## Usage

### 1. Get the Public Key

To encrypt secrets, you'll need the public key from the Sealed Secrets controller:

```bash
kubeseal --fetch-cert \
  --controller-namespace=kube-system \
  --controller-name=sealed-secrets \
  > public-key.pem
```

### 2. Create a Sealed Secret

1. Create a Kubernetes secret:
   ```bash
   kubectl create secret generic my-secret \
     --from-literal=username=admin \
     --from-literal=password=secret \
     --dry-run=client \
     -o yaml > my-secret.yaml
   ```

2. Seal the secret using the public key:
   ```bash
   kubeseal --format=yaml \
     --cert=public-key.pem \
     < my-secret.yaml > my-sealed-secret.yaml
   ```

3. The sealed secret will be automatically applied by ArgoCD when committed to the repository.

### 3. Verify the Secret

```bash
# Check the sealed secret
kubectl get SealedSecret -n <namespace> <secret-name>

# Check the decrypted secret
kubectl get secret -n <namespace> <secret-name>
```

## Troubleshooting

### Sealed Secrets controller not running
```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=sealed-secrets

# Check logs if pod is in CrashLoopBackOff
kubectl logs -n kube-system -l app.kubernetes.io/name=sealed-secrets
```

### Secret not being decrypted
- Verify the secret is in the same namespace as the SealedSecret resource
- Check the SealedSecret controller logs for decryption errors
- Ensure the secret name in the SealedSecret matches the target secret name

## Security Notes

- The public key is safe to commit to version control
- The private key is automatically managed by the controller
- Rotate the keys if the private key is compromised
- Access to the `kube-system` namespace should be restricted
