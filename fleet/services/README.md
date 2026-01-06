# Kubernetes Services

This directory contains additional Kubernetes Service resources for exposing cluster components.

## Services

### hubble-relay-external

- **Type**: NodePort
- **Port**: 31245
- **Target**: Hubble Relay gRPC API (port 4245)
- **Purpose**: External access to Hubble CLI API without port-forwarding

**Usage:**

```bash
# From cluster nodes
export HUBBLE_SERVER=localhost:31245
hubble status
hubble observe --follow

# From external machine
export HUBBLE_SERVER=<node-ip>:31245
hubble status
hubble observe --from-namespace default

# Example
export HUBBLE_SERVER=192.168.200.10:31245
hubble observe --follow
```

**Install Hubble CLI:**

```bash
curl -L --fail --remote-name-all https://github.com/cilium/hubble/releases/download/v1.18.3/hubble-linux-amd64.tar.gz
sudo tar xzvfC hubble-linux-amd64.tar.gz /usr/local/bin
```

## Dependencies

- **Cilium**: Hubble Relay must be running

## Verify

```bash
# Check service
kubectl get svc hubble-relay-external -n kube-system

# Test connectivity
hubble status --server <node-ip>:31245
```

## Security Considerations

NodePort services expose internal APIs externally. Consider:
- Using firewall rules to restrict access to trusted networks
- Implementing NetworkPolicies
- Using TLS for production environments
- Alternative: Use LoadBalancer or Ingress with authentication

## References

- [Hubble Documentation](https://docs.cilium.io/en/stable/observability/hubble/)
- [Kubernetes NodePort Services](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport)
