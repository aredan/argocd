# Sealed Secrets Controller

This directory contains the Helm chart for deploying the Bitnami Sealed Secrets controller in your Kubernetes cluster. Sealed Secrets allows you to encrypt your Kubernetes Secrets so they can be safely stored in version control.

## Prerequisites

- `kubectl` with cluster admin access
- `kubeseal` CLI tool installed
- Helm v3+

## Installation

The Sealed Secrets controller is deployed using the Bitnami Sealed Secrets Helm chart. The configuration is managed in `values.yaml`.

## Usage

### 1. Get the Public Key

To encrypt secrets, you'll need the public key from the Sealed Secrets controller:

```bash
# Save the public key to a file
kubeseal --fetch-cert \
  --controller-namespace=kube-system \
  --controller-name=sealed-secrets \
  > public-key.pem

# Or view the public key
kubeseal --fetch-cert \
  --controller-namespace=kube-system \
  --controller-name=sealed-secrets
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

3. Apply the sealed secret:
   ```bash
   kubectl apply -f my-sealed-secret.yaml
   ```

### 3. Verify the Secret

```bash
# Check the sealed secret
kubectl get SealedSecret my-secret -o yaml

# Check the decrypted secret
kubectl get secret my-secret -o yaml
```

## Troubleshooting

### If you get "no endpoints available"
Ensure the Sealed Secrets controller is running:
```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=sealed-secrets
```

### If you get "certificate signed by unknown authority"
Make sure you're using the correct controller name and namespace:
```bash
kubectl get svc -n kube-system -l app.kubernetes.io/name=sealed-secrets
```

## Security Notes

- The public key is safe to commit to version control
- Keep the private key (stored in the controller) secure
- Rotate the keys if the private key is compromised
- Use RBAC to restrict access to the Sealed Secrets controller
