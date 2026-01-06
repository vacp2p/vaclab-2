# Cilium CNI Deployment

This directory contains Fleet-managed Cilium deployment for network policy enforcement, chained with Kube-OVN.

## Overview

Cilium is deployed in **CNI chaining mode** with Kube-OVN:
- **Kube-OVN**: Primary CNI for networking (IP allocation, routing, overlay)
- **Cilium**: Network policy enforcement and observability via eBPF

## Architecture

```
Pod Creation Flow:
1. Kube-OVN CNI allocates IP and configures network
2. Cilium CNI attaches eBPF programs for policy enforcement
3. Both CNIs work together via generic-veth chaining
```

## Configuration

### Key Settings

- **CNI Chaining**: `chainingMode: generic-veth` with custom ConfigMap
- **Source IP Verification**: Disabled for NAT gateway compatibility
- **IP Masquerade**: Disabled (Kube-OVN handles SNAT/DNAT)
- **Identity Mark**: Disabled (required for chaining mode)
- **Kube-Proxy**: Kept enabled (K3s uses kube-proxy)

### Hubble Observability

Hubble is enabled for network visibility:
```bash
# Port-forward Hubble UI
kubectl port-forward -n kube-system svc/hubble-ui 12000:80

# Access UI at http://localhost:12000
```

## Dependencies

**Must be deployed AFTER Kube-OVN:**
1. Kube-OVN must be running first (provides primary networking)
2. CNI configuration ConfigMap references Kube-OVN socket
3. Cilium attaches to interfaces created by Kube-OVN

## Fleet Management

This deployment is managed by Rancher Fleet from the Git repository.

### Verify Deployment

```bash
# Check HelmChart CRD
kubectl get helmchart cilium -n kube-system

# Check Cilium pods
kubectl get pods -n kube-system -l k8s-app=cilium

# Check Cilium status
CILIUM_POD=$(kubectl -n kube-system get pod -l k8s-app=cilium -o jsonpath='{.items[0].metadata.name}')
kubectl -n kube-system exec ${CILIUM_POD} -- cilium-dbg status
```

### Manual Installation

If not using Fleet:
```bash
# Apply CNI configuration
kubectl apply -f install.yaml

# Or install via Helm directly
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium --version 1.18.5 \
  --namespace kube-system \
  --values values.yaml
```

## Network Policies

With Cilium deployed, you can use Kubernetes NetworkPolicies:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

Cilium also supports CiliumNetworkPolicy CRD for advanced policies.

## Troubleshooting

### Check CNI Chaining

```bash
# Verify CNI config on nodes
cat /etc/cni/net.d/05-cilium.conflist
cat /etc/cni/net.d/10-kube-ovn.conflist
```

### Check Cilium Logs

```bash
kubectl logs -n kube-system -l k8s-app=cilium --tail=50
```

### Verify Network Policies

```bash
# Check policy enforcement
kubectl -n kube-system exec -ti ${CILIUM_POD} -- cilium-dbg policy get
```

## References

- [Kube-OVN Cilium Integration Guide](https://kubeovn.github.io/docs/stable/en/advance/with-cilium/)
- [Cilium CNI Chaining Documentation](https://docs.cilium.io/en/stable/installation/cni-chaining/)
- [Hubble Observability](https://docs.cilium.io/en/stable/gettingstarted/hubble/)
