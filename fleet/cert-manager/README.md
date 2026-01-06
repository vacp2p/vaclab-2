# cert-manager Deployment

This directory contains Fleet-managed cert-manager deployment for TLS certificate management.

## Overview

cert-manager is a Kubernetes add-on that automates the management and issuance of TLS certificates from various issuing sources.

## Features

- Automatic certificate provisioning and renewal
- Integration with Let's Encrypt, HashiCorp Vault, Venafi, and more
- CRDs for Certificate, Issuer, and ClusterIssuer resources
- Webhook for validating and mutating cert-manager resources

## Configuration

### Key Settings

- **CRDs**: Installed automatically with the chart (`crds.enabled: true`)
- **Namespace**: Deployed in dedicated `cert-manager` namespace
- **Version**: v1.19.2

## Usage

### Create a ClusterIssuer

Example Let's Encrypt issuer:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
```

### Request a Certificate

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-com
  namespace: default
spec:
  secretName: example-com-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - example.com
  - www.example.com
```

## Fleet Management

This deployment is managed by Rancher Fleet from the Git repository.

### Verify Deployment

```bash
# Check cert-manager pods
kubectl get pods -n cert-manager

# Check CRDs
kubectl get crd | grep cert-manager

# Check webhook
kubectl get validatingwebhookconfigurations | grep cert-manager
```

### Check Certificate Status

```bash
# List certificates
kubectl get certificates -A

# Describe a certificate
kubectl describe certificate <name> -n <namespace>

# Check certificate requests
kubectl get certificaterequests -A
```

## Manual Installation

If not using Fleet:

```bash
# Add Helm repository
helm repo add jetstack https://charts.jetstack.io --force-update

# Install cert-manager
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.19.2 \
  --set crds.enabled=true
```

## Troubleshooting

### Check cert-manager logs

```bash
kubectl logs -n cert-manager -l app=cert-manager --tail=50
kubectl logs -n cert-manager -l app=webhook --tail=50
kubectl logs -n cert-manager -l app=cainjector --tail=50
```

### Verify webhook is running

```bash
kubectl get pods -n cert-manager -l app=webhook
```

### Check certificate events

```bash
kubectl describe certificate <name> -n <namespace>
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```

## References

- [cert-manager Documentation](https://cert-manager.io/docs/)
- [Installation Guide](https://cert-manager.io/docs/installation/)
- [Issuer Configuration](https://cert-manager.io/docs/configuration/)
- [Troubleshooting Guide](https://cert-manager.io/docs/troubleshooting/)
