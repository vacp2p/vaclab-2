# Ingress Resources

This directory contains Kubernetes Ingress resources for exposing cluster services externally.

## Ingresses

### Hubble UI

- **Host**: hubble.vaclab.diarra.tech
- **Service**: hubble-ui (kube-system namespace)
- **Port**: 80
- **TLS**: Automatic via cert-manager (letsencrypt-prod)
- **Middleware**: HTTPS redirect

**Purpose**: Provides web-based UI for Cilium Hubble network observability. View real-time network traffic flows, service dependencies, and network policy enforcement.

## Dependencies

- **cert-manager**: For automatic TLS certificate provisioning
- **cert-issuers**: letsencrypt-prod ClusterIssuer must exist
- **traefik**: K3s default ingress controller
- **traefik-middlewares**: redirect-https middleware for HTTP→HTTPS

## Hubble Access

### Via Ingress (Recommended for external access)

```bash
# Access via browser
https://hubble.vaclab.diarra.tech
```

### Via Port Forward (Local development)

```bash
# Forward Hubble UI
kubectl port-forward -n kube-system svc/hubble-ui 8080:80

# Access at http://localhost:8080
```

### Via Hubble CLI

Install Hubble CLI:

```bash
curl -L --fail --remote-name-all https://github.com/cilium/hubble/releases/download/v0.10.0/hubble-linux-amd64.tar.gz
sudo tar xzvfC hubble-linux-amd64.tar.gz /usr/local/bin
```

Port forward Hubble Relay:

```bash
kubectl port-forward -n kube-system deployment/hubble-relay 4245:4245
```

Query flows:

```bash
# Check status
hubble status

# Observe all flows
hubble observe

# Observe from specific namespace
hubble observe --from-namespace kube-system

# Follow live flows
hubble observe --follow
```

## Cilium Connectivity Test

Deploy test workloads:

```bash
cilium connectivity test
```

This creates `cilium-test` namespace with test pods and services. View traffic through Hubble UI or CLI.

## Verify Deployment

```bash
# Check ingress
kubectl get ingress hubble-ui -n kube-system

# Check certificate
kubectl get certificate hubble-tls -n kube-system

# Test HTTPS redirect
curl -I http://hubble.vaclab.diarra.tech

# Access Hubble UI
curl -k https://hubble.vaclab.diarra.tech
```

## Troubleshooting

### Certificate not issued

```bash
# Check certificate request
kubectl describe certificaterequest -n kube-system

# Check cert-manager logs
kubectl logs -n cert-manager deployment/cert-manager
```

### Ingress not working

```bash
# Check Traefik logs
kubectl logs -n kube-system deployment/traefik

# Check ingress events
kubectl describe ingress hubble-ui -n kube-system
```

### Hubble UI not loading

```bash
# Check Hubble UI pods
kubectl get pods -n kube-system -l k8s-app=hubble-ui

# Check Hubble UI logs
kubectl logs -n kube-system deployment/hubble-ui

# Check Hubble Relay
kubectl logs -n kube-system deployment/hubble-relay
```

## Additional Services

You can add more ingresses to this bundle for other services:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example
  namespace: default
  annotations:
    traefik.ingress.kubernetes.io/router.middlewares: default-redirect-https@kubernetescrd
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: traefik
  tls:
  - hosts:
    - example.vaclab.diarra.tech
    secretName: example-tls
  rules:
  - host: example.vaclab.diarra.tech
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: example-service
            port:
              number: 80
```

## References

- [Hubble UI Documentation](https://docs.cilium.io/en/stable/gettingstarted/hubble/)
- [Hubble CLI Documentation](https://docs.cilium.io/en/stable/gettingstarted/hubble_cli/)
- [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [cert-manager Ingress Annotations](https://cert-manager.io/docs/usage/ingress/)
