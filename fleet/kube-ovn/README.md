# Kube-OVN Fleet Deployment

This directory contains the Fleet configuration for deploying Kube-OVN CNI with Cilium chaining enabled.

## What is Kube-OVN?

Kube-OVN is an enterprise-grade SDN (Software Defined Network) and CNI plugin for Kubernetes based on Open vSwitch (OVS) and OVN (Open Virtual Network).

### Key Features:
- Advanced networking with subnet isolation
- Network policies and QoS
- Distributed and centralized gateways
- VPC and multi-tenant support
- DPDK acceleration support
- IPSec encryption

## Configuration

### Network Settings

Edit `values.yaml` to configure the network:

```yaml
POD_CIDR: "10.16.0.0/16"      # Must not overlap with node network
SVC_CIDR: "10.96.0.0/12"      # K8s service CIDR
JOIN_CIDR: "100.64.0.0/16"    # Node-to-pod gateway network
```

### Important: Match Your K3s Configuration

Make sure these CIDRs match K3s installation:
- `POD_CIDR` should match `--cluster-cidr` in K3s
- `SVC_CIDR` should match `--service-cidr` in K3s

### Node Interface

Set `IFACE` in `values.yaml` to your node's primary network interface:
```yaml
IFACE: "eth0"  # or ens3, enp0s3, etc.
```

### Check Deployment Status

```bash
# Check Fleet GitRepo status
kubectl -n fleet-local get gitrepo

# Check Kube-OVN pods
kubectl -n kube-system get pods -l app=kube-ovn

# Check OVN central components
kubectl -n kube-system get pods -l app=ovn-central

# Verify nodes are Ready
kubectl get nodes
```

## Resources

- [Kube-OVN Documentation](https://kubeovn.github.io/docs/)
- [GitHub Repository](https://github.com/kubeovn/kube-ovn)
- [Architecture Guide](https://kubeovn.github.io/docs/en/guide/setup-options/)
