# Worker Node — Join the Cluster

> Install k3s agent on VPS2 and join it to the control plane.

## Prerequisites

- k3s master is running and healthy (`kubectl get nodes` shows `Ready`)
- `K3S_NODE_TOKEN` noted from the master install and added to `.env`
- VPS2 bootstrapped with dotfiles (`make setup-worker` completed)
- UFW ports between VPS1 and VPS2 open (handled by the install scripts)

## Step 1 — Ensure Token is Set

In your local `.env`:

```ini
K3S_NODE_TOKEN=K10abc...::server:xyz
```

Retrieve from master at any time:

```bash
# On VPS1:
cat /var/lib/rancher/k3s/server/node-token

# Or via SSH from local machine:
ssh kevin@<MASTER_IP> "sudo cat /var/lib/rancher/k3s/server/node-token"
```

## Step 2 — Install k3s Agent (automated)

From your **local machine**:

```bash
make k3s-worker
```

This runs `k3s/install-worker.sh` remotely via SSH on VPS2.

## Step 2 — Install k3s Agent (manual)

SSH into **VPS2**:

```bash
ssh kevin@<WORKER_IP>
```

Set environment variables:

```bash
export K3S_VERSION=v1.32.2+k3s1
export MASTER_IP=<MASTER_IP>
export K3S_NODE_TOKEN=<token-from-master>
```

Run the install script:

```bash
bash <(curl -fsSL https://raw.githubusercontent.com/KevinDeBenedetti/infra/main/k3s/install-worker.sh)
```

### What the script does

1. Disables swap permanently
2. Loads `overlay` and `br_netfilter` kernel modules
3. Writes `/etc/sysctl.d/99-z-k3s.conf` (same as master — enables `ip_forward`)
4. Installs k3s agent connecting to `https://<MASTER_IP>:6443`
5. Opens required UFW ports (VXLAN, kubelet, pod/service CIDR)

### Install command

```bash
curl -sfL https://get.k3s.io | \
  INSTALL_K3S_VERSION="${K3S_VERSION}" \
  K3S_TOKEN="${K3S_NODE_TOKEN}" \
  sh -s - agent \
    --server="https://${MASTER_IP}:6443" \
    --node-ip="${NODE_IP}" \
    --protect-kernel-defaults \
    --node-label="node-role=worker"
```

## Step 3 — Verify the Worker Joined

From your **local machine** (or VPS1):

```bash
kubectl get nodes -o wide
```

Expected output:

```
NAME    STATUS   ROLES                  AGE   VERSION          INTERNAL-IP   ...
vps1    Ready    control-plane,master   10m   v1.32.2+k3s1    1.2.3.4       ...
vps2    Ready    <none>                 1m    v1.32.2+k3s1    5.6.7.8       ...
```

Both nodes should show `Ready`. VPS2 has no roles assigned (it is a worker/agent node).

Verify all system pods are running:

```bash
kubectl get pods -A -o wide
# Flannel pods should be Running on both nodes
# CoreDNS might be scheduled on the worker too
```

## Step 4 — Verify Inter-Node Networking

Test pod-to-pod connectivity across nodes:

```bash
# Create one pod on each node
kubectl run pod-master --image=alpine --restart=Never \
  --overrides='{"spec":{"nodeSelector":{"node-role":"master"}}}' -- sleep 600

kubectl run pod-worker --image=alpine --restart=Never \
  --overrides='{"spec":{"nodeSelector":{"node-role":"worker"}}}' -- sleep 600

# Get IPs
kubectl get pods -o wide

# Ping across nodes
WORKER_POD_IP=$(kubectl get pod pod-worker -o jsonpath='{.status.podIP}')
kubectl exec pod-master -- ping -c 3 "${WORKER_POD_IP}"

# Cleanup
kubectl delete pod pod-master pod-worker
```

## k3s Agent Service Management

On **VPS2**:

```bash
# Check k3s-agent service
systemctl status k3s-agent

# Restart agent
systemctl restart k3s-agent

# View agent logs
journalctl -u k3s-agent -f
journalctl -u k3s-agent --since "10 minutes ago"
```

## Node Labels

The worker node is labeled `node-role=worker` at install time. Use this to schedule workloads on specific nodes:

```yaml
# Force a pod onto the worker node
spec:
  nodeSelector:
    node-role: worker
```

```bash
# View node labels
kubectl get nodes --show-labels

# Add a label manually
kubectl label node <node-name> key=value

# Remove a label
kubectl label node <node-name> key-
```

## Cordon / Drain / Uncordon

```bash
# Prevent new pods from scheduling on a node
kubectl cordon <node-name>

# Evict all pods (for maintenance)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Bring node back into rotation
kubectl uncordon <node-name>
```

## Remove Worker from Cluster

On **VPS2** (DESTRUCTIVE):

```bash
# Option 1: via Makefile from local machine
make k3s-uninstall-worker

# Option 2: manually on VPS2
/usr/local/bin/k3s-agent-uninstall.sh
```

From **local machine**, delete the node object:

```bash
kubectl delete node <worker-node-name>
```

## Troubleshooting

| Symptom                              | Likely Cause           | Fix                                                        |
| ------------------------------------ | ---------------------- | ---------------------------------------------------------- |
| Worker stays `NotReady`              | UFW blocking port 6443 | `ufw allow from <WORKER_IP> to any port 6443` on master    |
| Worker stays `NotReady`              | Wrong token            | Re-check `K3S_NODE_TOKEN` on both sides                    |
| VXLAN not working                    | UDP 8472 blocked       | `ufw allow from <MASTER_IP> to any port 8472 proto udp`    |
| Pod on worker can't reach master pod | `ip_forward=0`         | Re-apply `/etc/sysctl.d/99-z-k3s.conf` + `sysctl --system` |

See [troubleshooting.md](troubleshooting.md) for more.
