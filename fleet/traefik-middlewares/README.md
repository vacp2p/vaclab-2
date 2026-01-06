# Traefik Middlewares

This directory contains Traefik middleware configurations for traffic manipulation.

## Overview

Traefik middlewares allow you to modify requests and responses before they reach your services or are sent back to clients.

## Middlewares

### redirect-https

- **Purpose**: Automatically redirect HTTP traffic to HTTPS
- **Type**: RedirectScheme
- **Permanent**: Yes (HTTP 301)
- **Namespace**: kube-system (accessible cluster-wide)

## Usage

### In Ingress Annotations

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example
  annotations:
    traefik.ingress.kubernetes.io/router.middlewares: default-redirect-https@kubernetescrd
spec:
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

### In IngressRoute (Traefik CRD)

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: example
spec:
  entryPoints:
    - web
    - websecure
  routes:
  - match: Host(`example.com`)
    kind: Rule
    middlewares:
    - name: redirect-https
      namespace: kube-system
    services:
    - name: example
      port: 80
```

## How It Works

1. Client makes HTTP request to `http://example.com`
2. Traefik intercepts the request
3. Middleware redirects to `https://example.com` (HTTP 301)
4. Client follows redirect to HTTPS endpoint
5. Request proceeds with TLS encryption

## Dependencies

This middleware works with the default K3s Traefik installation. No additional dependencies required.

## Verify

```bash
# Check middleware exists
kubectl get middleware redirect-https -n default

# Describe middleware
kubectl describe middleware redirect-https -n default

# Test with curl (should return 301)
curl -I http://your-domain.com
```

## Additional Middlewares

You can add more middlewares to this bundle:

### Rate Limiting

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: rate-limit
  namespace: kube-system
spec:
  rateLimit:
    average: 100
    burst: 50
```

### Headers

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: security-headers
  namespace: kube-system
spec:
  headers:
    customResponseHeaders:
      X-Frame-Options: "SAMEORIGIN"
      X-Content-Type-Options: "nosniff"
      X-XSS-Protection: "1; mode=block"
```

### Basic Auth

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: basic-auth
  namespace: default
spec:
  basicAuth:
    secret: auth-secret
```

## References

- [Traefik Middlewares Documentation](https://doc.traefik.io/traefik/middlewares/overview/)
- [RedirectScheme Middleware](https://doc.traefik.io/traefik/middlewares/http/redirectscheme/)
- [Traefik Kubernetes CRD](https://doc.traefik.io/traefik/routing/providers/kubernetes-crd/)
