# Certificate Issuers

This directory contains ClusterIssuer resources for automatic TLS certificate provisioning.

## Overview

ClusterIssuers are cluster-scoped resources that can issue certificates for any namespace. They must be deployed **after** cert-manager is running.

## Issuers

### Let's Encrypt Production

- **Name**: `letsencrypt-prod`
- **Server**: Let's Encrypt production ACME server
- **Solver**: HTTP-01 challenge via Traefik ingress
- **Email**: Configured via `${EMAIL}` environment variable

## Configuration

### Email Address

The `${EMAIL}` variable needs to be replaced with your actual email address for Let's Encrypt notifications. You can:

1. **Set via Fleet variables** (recommended):
   ```yaml
   # In GitRepo or Bundle configuration
   helm:
     valuesFrom:
       - secretKeyRef:
           name: letsencrypt-config
           key: email
   ```

2. **Or edit the file directly** and replace `${EMAIL}` with your email

## Dependencies

This bundle depends on `cert-manager` being deployed and ready first (configured in `fleet.yaml`).

## Usage

Once deployed, reference this issuer in Certificate resources:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-tls
  namespace: default
spec:
  secretName: example-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - example.com
  - www.example.com
```

Or use annotations in Ingress resources:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - example.com
    secretName: example-tls
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: example
            port:
              number: 80
```

## Verify

```bash
# Check ClusterIssuer status
kubectl get clusterissuer letsencrypt-prod

# Describe to see more details
kubectl describe clusterissuer letsencrypt-prod

# Check if issuer is ready
kubectl get clusterissuer letsencrypt-prod -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}'
```

## Troubleshooting

### Issuer not ready

```bash
# Check cert-manager logs
kubectl logs -n cert-manager -l app=cert-manager

# Check issuer status
kubectl describe clusterissuer letsencrypt-prod
```

### HTTP-01 challenge failing

Ensure:
- Traefik ingress controller is running
- Port 80 is accessible from the internet
- DNS is correctly pointing to your cluster
- Firewall allows HTTP traffic

## References

- [Let's Encrypt Documentation](https://letsencrypt.org/docs/)
- [cert-manager ACME Configuration](https://cert-manager.io/docs/configuration/acme/)
- [HTTP-01 Challenge](https://cert-manager.io/docs/configuration/acme/http01/)
