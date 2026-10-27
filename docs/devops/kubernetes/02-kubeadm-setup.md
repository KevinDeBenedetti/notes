# k3s Master Setup

> Install and configure the k3s control plane (master node / VPS1).

## Overview

This setup uses **k3s** — a certified, lightweight Kubernetes distribution packaged as a single binary. It includes an embedded etcd datastore, CoreDNS, Flannel, and the Metrics Server out of the box.

> Traefik and ServiceLB are **disabled** here — they are managed separately via Helm for full configurability.

## Step 1 — Configure Environment

On your **local machine**, set up the `.env` file:

```bash
cp .env.example .env
```

```ini
MASTER_IP=1.2.3.4
WORKER_IP=5.6.7.8
SSH_USER=kevin
SSH_PORT=22
SSH_KEY=~/.ssh/id_ed25519
K3S_VERSION=v1.32.2+k3s1
```

> `K3S_NODE_TOKEN` will be printed at the end of the install and should be added to `.env` before step 4.

## Step 2 — Install k3s Server (automated)

From your **local machine**:

```bash
make k3s-master
```

This runs `k3s/install-master.sh` remotely via SSH using the key defined in `SSH_KEY`.

## Step 2 — Install k3s Server (manual alternative)

SSH into **VPS1** as your user:

```bash
ssh kevin@<MASTER_IP>
```

Set environment variables:

```bash
export K3S_VERSION=v1.32.2+k3s1
export PUBLIC_IP=<MASTER_IP>
export WORKER_IP=<WORKER_IP>          # optional, adds VXLAN UFW rule early
export K3S_NODE_TOKEN=<your-secret>   # optional, generated if not set
```

Run the install script:

```bash
bash <(curl -fsSL https://raw.githubusercontent.com/KevinDeBenedetti/infra/main/k3s/install-master.sh)
```

### What the script does

1. Disables swap permanently
2. Loads `overlay` and `br_netfilter` kernel modules
3. Writes `/etc/sysctl.d/99-z-k3s.conf` (enables `ip_forward`, overrides security baseline)
4. Installs k3s server via the official installer with production flags
5. Waits for the node to reach `Ready` state
6. Opens required UFW ports
7. Prints the `K3S_NODE_TOKEN` and kubeconfig path

### Install flags explained

```bash
curl -sfL https://get.k3s.io | \
  INSTALL_K3S_VERSION="${K3S_VERSION}" \
  K3S_TOKEN="${K3S_NODE_TOKEN}" \
  sh -s - server \
    --disable=traefik \          # managed via Helm
    --disable=servicelb \        # managed via Helm
    --node-ip="${NODE_IP}" \     # internal NIC IP
    --advertise-address="${NODE_IP}" \
    --tls-san="${PUBLIC_IP}" \   # API server cert includes public IP
    --flannel-backend=vxlan \    # explicit VXLAN (default, stable)
    --protect-kernel-defaults \  # enforce sysctl requirements at start
    --secrets-encryption \       # encrypt Secrets at rest in etcd
    --write-kubeconfig-mode=600 \
    --node-label="node-role=master"
```

## Step 3 — Verify Master Node

On **VPS1**:

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

kubectl get nodes -o wide
# NAME    STATUS   ROLES                  AGE   VERSION
# vps1    Ready    control-plane,master   1m    v1.32.2+k3s1

kubectl get pods -A
# All pods in kube-system should be Running
```

## Step 4 — Note the Node Token

The token is printed at the end of the script. Copy it to your `.env`:

```bash
# On VPS1:
cat /var/lib/rancher/k3s/server/node-token
# K10abc...::server:xyz

# Or from the script output:
# export K3S_NODE_TOKEN=<token>
```

Add to `.env`:

```ini
K3S_NODE_TOKEN=K10abc...::server:xyz
```

## Step 5 — Fetch Kubeconfig (Local Machine)

```bash
make kubeconfig
# or manually:
./scripts/get-kubeconfig.sh <MASTER_IP> <SSH_USER> k3s-infra
```

The script:
- Fetches `/etc/rancher/k3s/k3s.yaml` from the master via SSH
- Replaces `127.0.0.1` with `<MASTER_IP>`
- Renames cluster/context/user from `default` → `k3s-infra`
- Merges into `~/.kube/config`

Switch to the cluster context:

```bash
kubectl config use-context k3s-infra
kubectl get nodes -o wide
```

## Step 6 — Deploy Base Stack (Local Machine)

Update the cluster-specific values first:

- `kubernetes/cert-manager/clusterissuer.yaml` — replace `admin@example.com` with your email
- `kubernetes/ingress/traefik-dashboard.yaml` — replace the domain with yours

Then deploy:

```bash
make deploy
# or manually:
./scripts/deploy-stack.sh
```

This deploys in order:
1. **Namespaces** — `apps`, `ingress`, `cert-manager`
2. **Traefik** `v34.4.0` via Helm (LoadBalancer, HTTP→HTTPS redirect, TLS hardening)
3. **cert-manager** `v1.17.1` via Helm (CRDs included)
4. **ClusterIssuers** — Let's Encrypt staging + production
5. **Traefik dashboard** IngressRoute (BasicAuth protected)

## Step 7 — Create the Dashboard BasicAuth Secret

```bash
DASHBOARD_PASSWORD=<your-password> make deploy-dashboard-secret
```

This runs:

```bash
kubectl create secret generic traefik-dashboard-auth \
  --from-literal=users="$(htpasswd -nb admin <YOUR_PASSWORD>)" \
  -n ingress \
  --dry-run=client -o yaml | kubectl apply -f -
```

The secret is created imperatively and applied idempotently — no YAML editing needed.

> The dashboard is accessible at `https://dashboard.<DOMAIN>` (as configured in `kubernetes/ingress/traefik-dashboard.yaml`).

## Verify Full Stack

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl get svc -A
kubectl get ingress -A
```

Expected output — all pods `Running`, Traefik has an `EXTERNAL-IP`:

```
NAMESPACE      NAME                       READY   STATUS    ...
cert-manager   cert-manager-...           1/1     Running
ingress        traefik-...                1/1     Running
kube-system    coredns-...                1/1     Running
kube-system    local-path-provisioner-... 1/1     Running
kube-system    metrics-server-...         1/1     Running
```

## Makefile Targets Reference

| Target                         | Description                                  |
| ------------------------------ | -------------------------------------------- |
| `make setup-master`            | Bootstrap VPS1 with dotfiles                 |
| `make setup-worker`            | Bootstrap VPS2 with dotfiles                 |
| `make setup-all`               | Bootstrap both VPS nodes                     |
| `make k3s-master`              | Install k3s server on VPS1                   |
| `make k3s-worker`              | Install k3s agent on VPS2                    |
| `make kubeconfig`              | Fetch and merge kubeconfig locally           |
| `make deploy`                  | Deploy Traefik, cert-manager, ClusterIssuers |
| `make deploy-dashboard-secret` | Create Traefik BasicAuth secret              |
| `make nodes`                   | `kubectl get nodes -o wide`                  |
| `make status`                  | `kubectl get pods -A` (running only)         |
| `make pods`                    | `kubectl top pods -A`                        |
| `make k3s-uninstall-master`    | Remove k3s from master (DESTRUCTIVE)         |
| `make k3s-uninstall-worker`    | Remove k3s from worker (DESTRUCTIVE)         |

## k3s Service Management

```bash
# Check k3s service status
systemctl status k3s

# Restart k3s
systemctl restart k3s

# View k3s logs
journalctl -u k3s -f
journalctl -u k3s --since "10 minutes ago"
```
