# Kubernetes — k3s Setup Overview

> Lightweight, production-ready Kubernetes cluster on 2 VPS nodes using [k3s](https://k3s.io).

## Stack

| Tool                                                                         | Role                                              |
| ---------------------------------------------------------------------------- | ------------------------------------------------- |
| [k3s](https://k3s.io) `v1.32.2+k3s1`                                         | Certified Kubernetes distribution (single binary) |
| [Traefik](https://traefik.io) `v34.4.0`                                      | Ingress controller & HTTPS reverse proxy          |
| [cert-manager](https://cert-manager.io) `v1.17.1`                            | Automatic TLS certificates via Let's Encrypt      |
| [local-path-provisioner](https://github.com/rancher/local-path-provisioner)  | Persistent storage (bundled with k3s)             |
| [Flannel](https://github.com/flannel-io/flannel) (VXLAN)                     | Pod overlay network (built into k3s)              |
| [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts) | Prometheus + Grafana + Alertmanager               |

## Cluster Layout

```
[Internet]
    │  80 / 443
    ▼
[VPS2 — Worker Node]          ← Traefik LoadBalancer
  • k3s agent
  • Application pods
    │  private network (VXLAN / Flannel)
    ▼
[VPS1 — Master / Control Plane]
  • k3s server
  • API server (port 6443)
  • etcd (embedded)
```

## Repository

The infrastructure source lives in `KevinDeBenedetti/infra`, a private repository.

```
infra/
├── .env.example               # Environment variable template
├── Makefile                   # Automation targets
├── k3s/
│   ├── install-master.sh      # Bootstrap the control-plane node
│   ├── install-worker.sh      # Join a worker node to the cluster
│   └── uninstall.sh           # Remove k3s from a node
├── kubernetes/
│   ├── namespaces/            # apps, ingress, cert-manager, monitoring
│   ├── ingress/               # Traefik Helm values + dashboard IngressRoute
│   ├── cert-manager/          # Let's Encrypt ClusterIssuers
│   └── apps/                  # Example deployment & service/ingress
└── scripts/
    ├── deploy-stack.sh        # Deploy the full base stack
    ├── get-kubeconfig.sh      # Fetch & merge kubeconfig locally
    └── setup-vps.sh           # Bootstrap a fresh Debian VPS
```

## Documentation Index

| File                                       | Description                               |
| ------------------------------------------ | ----------------------------------------- |
| [architecture.md](architecture.md)         | Master/worker layout, network, ports      |
| [01-prerequisites.md](01-prerequisites.md) | OS, kernel, swap, firewall, VPS bootstrap |
| [02-kubeadm-setup.md](02-kubeadm-setup.md) | k3s master install (control plane)        |
| [03-networking.md](03-networking.md)       | Flannel VXLAN, Traefik, cert-manager      |
| [04-worker-join.md](04-worker-join.md)     | Joining the worker node                   |
| [05-storage.md](05-storage.md)             | local-path PV/PVC                         |
| [06-observability.md](06-observability.md) | Prometheus, Grafana, Alertmanager         |
| [troubleshooting.md](troubleshooting.md)   | Common errors & fixes                     |
| [ROADMAP.md](ROADMAP.md)                   | Full setup journey & next steps           |

## Quick Start (local machine)

```bash
# 1. Clone and configure
cp .env.example .env
# Edit: MASTER_IP, WORKER_IP, SSH_USER, DOMAIN, EMAIL

# 2. Bootstrap both VPS nodes (dotfiles + security hardening)
make setup-all

# 3. Install k3s on master
make k3s-master
# → note the K3S_NODE_TOKEN printed, add it to .env

# 4. Install k3s on worker
make k3s-worker

# 5. Fetch kubeconfig
make kubeconfig
kubectl config use-context k3s-infra
kubectl get nodes -o wide

# 6. Deploy base stack (Traefik, cert-manager, ClusterIssuers)
make deploy

# 7. Create Traefik dashboard BasicAuth secret
DASHBOARD_PASSWORD=<your-password> make deploy-dashboard-secret
```

## Useful Makefile Targets

```bash
make nodes          # kubectl get nodes -o wide
make status         # kubectl get pods -A (running)
make pods           # kubectl top pods -A
make kubeconfig     # fetch & merge kubeconfig from master
make deploy         # deploy Traefik + cert-manager + ClusterIssuers
make deploy-dashboard-secret  # create Traefik BasicAuth secret
make k3s-uninstall-master     # remove k3s from master (DESTRUCTIVE)
make k3s-uninstall-worker     # remove k3s from worker (DESTRUCTIVE)
```

## References

- [k3s Documentation](https://docs.k3s.io)
- [Traefik Docs](https://doc.traefik.io/traefik/)
- [cert-manager Docs](https://cert-manager.io/docs/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
