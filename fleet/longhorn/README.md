# Longhorn Multiple Disk Configuration

## Overview

This configuration uses Longhorn's **annotation-based method** to automatically create multiple disks on agent nodes. This is the production-recommended approach that doesn't require post-installation manual configuration.

## How It Works

### 1. Ansible Preparation (Before Longhorn Install)

The Ansible playbook (`longhorn_node_labels.yaml`) does the following:

**For agent nodes (node-01 to node-05):**
```bash
# Label node to enable automatic disk creation
kubectl label node <node-name> node.longhorn.io/create-default-disk=true

# Annotate with multiple disk paths
kubectl annotate node <node-name> \
  node.longhorn.io/default-disks-config='[
    {"path":"/var/lib/longhorn","allowScheduling":true,"storageReserved":107374182400},
    {"path":"/data/longhorn","allowScheduling":true,"storageReserved":107374182400}
  ]'
```

**For master node (metal-01):**
```bash
# Label to prevent default disk creation
kubectl label node metal-01 node.longhorn.io/create-default-disk=false
```

### 2. Longhorn Helm Configuration

`fleet/longhorn/fleet.yaml` has:
```yaml
defaultSettings:
  createDefaultDiskLabeledNodes: true  # Enable annotation-based disk creation
```

### 3. Automatic Disk Creation

When Longhorn starts, it:
1. Reads node labels and annotations
2. Automatically creates both disks on labeled nodes
3. Skips disk creation on master node

## Disk Layout

### Agent Nodes (node-01 to node-05)
Each agent node automatically gets two disks:
- **Disk 1**: `/var/lib/longhorn` on `/dev/sda2` (~1.5TB)
  - Storage Reserved: 100GB
  - Scheduling: Enabled
- **Disk 2**: `/data/longhorn` on `/dev/nvme0n1` (~1.8TB)
  - Storage Reserved: 100GB
  - Scheduling: Enabled

### Master Node (metal-01)
- No disks created (label set to `false`)
- Node scheduling will be manually disabled in Longhorn UI

## Installation Order

The Ansible playbook ensures correct order:

1. ✅ System tuning applied
2. ✅ K3s installed (metal-01 as master)
3. ✅ Kube-OVN CNI installed
4. ✅ Cilium installed
5. ✅ **Node taints applied** (`dedicated=monitoring:PreferNoSchedule`)
6. ✅ **Longhorn node labels/annotations applied**
7. ✅ Longhorn installed (via Fleet) → disks auto-created
8. ✅ Other components installed

## Verification

### Check Node Labels and Annotations

```bash
# Check agent node configuration
kubectl get node node-01 -o yaml | grep -A 5 "node.longhorn.io"

# Expected output:
# labels:
#   node.longhorn.io/create-default-disk: "true"
# annotations:
#   node.longhorn.io/default-disks-config: '[{"path":"/var/lib/longhorn",...},{"path":"/data/longhorn",...}]'

# Check master node
kubectl get node metal-01 -o yaml | grep "node.longhorn.io"
# Expected: node.longhorn.io/create-default-disk: "false"
```

### Check Longhorn Disks

```bash
# List all Longhorn nodes
kubectl get nodes.longhorn.io -n longhorn-system

# Check specific node disk configuration
kubectl get nodes.longhorn.io node-01 -n longhorn-system -o jsonpath='{.spec.disks}' | jq

# Should show 2 disks with paths /var/lib/longhorn and /data/longhorn
```

### Via Longhorn UI

```bash
# Port-forward to Longhorn UI
kubectl port-forward -n longhorn-system svc/longhorn-frontend 8080:80
```

Open http://localhost:8080 and go to **Node** tab.

**Expected for agent nodes:**
- 2 disks visible
- Both disks "Schedulable"
- Total ~3.1TB per node

**Expected for master node:**
- Need to manually set "Node Scheduling" to **Disabled**

## Storage Capacity

### Per Agent Node
- Disk 1 (`/var/lib/longhorn`): ~1.4TB usable
- Disk 2 (`/data/longhorn`): ~1.7TB usable
- **Total**: ~3.1TB per node

### Cluster Total
- 5 agent nodes × 3.1TB = **~15.5TB total**
- Master node excluded (no disks created)

## Manual Post-Install Step

**Disable scheduling on master node:**

1. Go to Longhorn UI → **Node** tab
2. Click on `metal-01`
3. Click **Edit node and disks**
4. Set **Node Scheduling**: **Disabled**
5. Click **Save**

Or via kubectl:
```bash
kubectl patch nodes.longhorn.io metal-01 -n longhorn-system --type='json' \
  -p='[{"op": "replace", "path": "/spec/allowScheduling", "value": false}]'
```

## Advantages of This Method

✅ **Clean**: No manual disk addition after install  
✅ **Declarative**: Configuration in code (annotations)  
✅ **Automated**: Disks created automatically when Longhorn starts  
✅ **Production-ready**: Recommended by Longhorn documentation  
✅ **Idempotent**: Re-running won't break existing disks

## Troubleshooting

### Disks not created

```bash
# Check if directories exist
ansible k3s_cluster -i inventory.yaml -m shell -a "ls -la /var/lib/longhorn /data/longhorn"

# Check node labels
kubectl get nodes --show-labels | grep longhorn

# Check Longhorn manager logs
kubectl logs -n longhorn-system -l app=longhorn-manager --tail=100 | grep -i disk
```

### Wrong disk configuration

If you need to change disk configuration after install:

1. Delete existing disks in Longhorn UI
2. Update node annotations:
```bash
kubectl annotate node <node-name> node.longhorn.io/default-disks-config='[...]' --overwrite
```
3. Restart Longhorn manager:
```bash
kubectl rollout restart deployment/longhorn-manager -n longhorn-system
```

## References

- [Longhorn Multiple Disks Documentation](https://longhorn.io/docs/1.10.1/volumes-and-nodes/multidisk/)
- [Longhorn Node Configuration](https://longhorn.io/docs/1.10.1/references/longhorn-client-node-level/)
