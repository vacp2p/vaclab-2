# Homepage Dashboard

Official Kubernetes deployment for [Homepage](https://gethomepage.dev/) - a modern application dashboard with automatic service discovery.

## Overview

- **Namespace**: `homepage`
- **URL**: https://homepage.vaclab.diarra.tech
- **Service Discovery**: Automatic via Ingress/IngressRoute annotations
- **Widgets**: Kubernetes cluster stats, Longhorn storage, service integrations

## Architecture

- ServiceAccount with ClusterRole for reading cluster resources
- ConfigMap for all configuration (services, widgets, bookmarks, settings)
- Deployment with security context (non-root user)
- Service (ClusterIP on port 3000)
- Ingress with TLS via cert-manager

## Icons

Homepage finds icons in several ways:

### 1. Built-in Icons
Homepage includes 100+ built-in icons for popular services. Format: `servicename.png`

Examples: `grafana.png`, `rancher.png`, `longhorn.png`, `prometheus.png`, `traefik.png`

To see all available icons:
```bash
kubectl exec -n homepage deployment/homepage -- ls /app/public/icons
```

### 2. Material Design Icons
Use any Material Design Icon. Format: `mdi-icon-name`

Examples: `mdi-kubernetes`, `mdi-server`, `mdi-chart-line`, `mdi-database`

Browse icons: https://pictogrammers.com/library/mdi/

### 3. Custom URLs
Direct URL to any image. Format: `https://example.com/icon.png`

Example: `https://status.im/favicon.ico`

### 4. SVG (from URLs)
Direct SVG URLs work too:
```yaml
icon: https://raw.githubusercontent.com/walkxcode/dashboard-icons/main/svg/myapp.svg
```

Popular icon repository: https://github.com/walkxcode/dashboard-icons

## Automatic Service Discovery

Add these annotations to any Ingress or IngressRoute to have it automatically appear on Homepage:

### Standard Ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  namespace: myapp
  annotations:
    gethomepage.dev/enabled: "true"
    gethomepage.dev/name: "My App"
    gethomepage.dev/description: "Application description"
    gethomepage.dev/group: "Applications"
    gethomepage.dev/icon: "myapp.png"  # or mdi-apps or https://...
spec:
  rules:
    - host: myapp.vaclab.diarra.tech
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 8080
```

### Traefik IngressRoute
```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: myapp
  annotations:
    gethomepage.dev/enabled: "true"
    gethomepage.dev/href: "https://myapp.vaclab.diarra.tech"  # Required for IngressRoute
    gethomepage.dev/name: "My App"
    gethomepage.dev/description: "Application description"
    gethomepage.dev/group: "Applications"
    gethomepage.dev/icon: "mdi-apps"
spec:
  entryPoints:
    - websecure
  routes:
    - kind: Rule
      match: Host(`myapp.vaclab.diarra.tech`)
      services:
        - name: myapp
          port: 8080
```

## Service Widgets

Add widget configuration to show live stats from services:

### Longhorn
```yaml
gethomepage.dev/widget.type: "longhorn"
gethomepage.dev/widget.url: "http://longhorn-frontend.longhorn-system.svc.cluster.local"
gethomepage.dev/widget.expanded: "true"
```

### Grafana
```yaml
gethomepage.dev/widget.type: "grafana"
gethomepage.dev/widget.url: "http://grafana.namespace.svc.cluster.local"
gethomepage.dev/widget.username: "admin"
gethomepage.dev/widget.password: "${GRAFANA_PASSWORD}"
gethomepage.dev/widget.version: "2"
```

### Traefik
```yaml
gethomepage.dev/widget.type: "traefik"
gethomepage.dev/widget.url: "http://traefik.kube-system.svc.cluster.local:9000"
```

### Authentik
```yaml
gethomepage.dev/widget.type: "authentik"
gethomepage.dev/widget.url: "http://authentik-server.authentik.svc.cluster.local"
gethomepage.dev/widget.key: "your-api-token"
```

See all widgets: https://gethomepage.dev/widgets/services/

## Pod Resource Stats

Show CPU/Memory usage for pods:

```yaml
# Using app label (matches app.kubernetes.io/name)
gethomepage.dev/app: "myapp"

# Or using custom pod selector
gethomepage.dev/pod-selector: "app=myapp,env=production"

# Or all pods in namespace
gethomepage.dev/pod-selector: ""
```

## Configuration

All configuration is in `configmap.yaml`:

- **kubernetes.yaml**: Cluster connection mode and discovery settings
- **settings.yaml**: Theme, layout, title
- **services.yaml**: Manually configured services
- **widgets.yaml**: Information widgets (cluster stats, search, datetime)
- **bookmarks.yaml**: Quick links

To modify configuration:
1. Edit `configmap.yaml`
2. Commit and push
3. Fleet will automatically update the cluster

## Files

- `namespace.yaml` - Creates homepage namespace
- `serviceaccount.yaml` - Service account for the pod
- `secret.yaml` - Service account token
- `clusterrole.yaml` - RBAC permissions for service discovery
- `configmap.yaml` - All Homepage configuration
- `deployment.yaml` - Application deployment
- `service.yaml` - ClusterIP service
- `ingress.yaml` - Traefik ingress with TLS
- `fleet.yaml` - Fleet dependencies

## Troubleshooting

### Service not appearing
```bash
# Check if annotation is correct
kubectl get ingress myapp -n myapp -o yaml | grep gethomepage.dev

# Check Homepage logs
kubectl logs -n homepage -l app.kubernetes.io/name=homepage -f
```

### Widget not working
```bash
# Test connectivity from Homepage pod
kubectl exec -n homepage deployment/homepage -- wget -O- http://service.namespace.svc.cluster.local

# Check logs for errors
kubectl logs -n homepage -l app.kubernetes.io/name=homepage | grep -i error
```

### Configuration not updating
```bash
# Verify ConfigMap updated
kubectl get configmap homepage -n homepage -o yaml

# Restart deployment
kubectl rollout restart deployment/homepage -n homepage
```

## Resources

- [Homepage Docs](https://gethomepage.dev/)
- [Kubernetes Guide](https://gethomepage.dev/configs/kubernetes/)
- [Service Widgets](https://gethomepage.dev/widgets/services/)
- [Dashboard Icons](https://github.com/walkxcode/dashboard-icons)
- [Material Design Icons](https://pictogrammers.com/library/mdi/)
