# Roadmap — From Fresh VPS to a Fully Managed Cluster

> Complete journey from bare metal VPS provisioning to a production-grade, observable, secured Kubernetes cluster on 2 nodes.

---

## Overview

```
Phase 0 — Local tools setup
Phase 1 — VPS provisioning (Debian 13)
Phase 2 — Automated OS hardening (dotfiles)
Phase 3 — k3s master (control plane)
Phase 4 — k3s worker (join node)
Phase 5 — Base stack (Traefik + cert-manager)
Phase 6 — Application deployment
Phase 7 — Observability (Prometheus + Grafana)
Phase 8 — Secret management (Vault)           ← next
Phase 9 — GitOps (ArgoCD / Flux)              ← next
Phase 10 — Advanced security (Falco, OPA)     ← future
```

---

## Phase 0 — Local Tools

Install the required CLI tools on your local machine before anything else.

```bash
# macOS
brew install kubernetes-cli helm httpd jq yq

# Verify
kubectl version --client
helm version
htpasswd -v
```

| Tool       | Why                                                      |
| ---------- | -------------------------------------------------------- |
| `kubectl`  | Interact with the cluster API                            |
| `helm`     | Deploy Traefik, cert-manager, kube-prometheus-stack      |
| `htpasswd` | Generate BasicAuth credentials for the Traefik dashboard |
| `jq`       | Parse JSON output from kubectl/API                       |
| `yq`       | Parse and edit YAML files                                |

**Status:** ✅ Done — see [01-prerequisites.md](01-prerequisites.md)

---

## Phase 1 — VPS Provisioning

> Prerequisite: 2 VPS running **Debian 13 (Trixie)**, SSH access as root.

Get two VPS nodes from any provider (OVH, Hetzner, etc.) running Debian 13.

```bash
# Verify SSH access works
ssh root@<MASTER_IP>
ssh root@<WORKER_IP>
```

**Checklist:**
- 2 VPS with Debian 13 created
- Root SSH key authentication configured on both
- Private network or known IPs for both nodes
- `.env` configured with `MASTER_IP`, `WORKER_IP`, `SSH_USER`, `DOMAIN`, `EMAIL`

**Status:** See [01-prerequisites.md](01-prerequisites.md)

---

## Phase 2 — Automated OS Hardening

The `scripts/setup-vps.sh` script bootstraps each VPS with the dotfiles setup:

- Creates a non-root sudo user with your SSH key
- Installs base packages (curl, git, vim, htop, …)
- SSH hardening (no root login, key-only, custom port)
- UFW firewall (default deny-in, allow SSH/HTTP/HTTPS)
- Fail2Ban (SSH brute-force protection)
- AppArmor (mandatory access control)
- Kernel sysctl hardening (`99-security.conf`)
- Docker CE (optional)

```bash
make setup-master     # bootstrap VPS1
make setup-worker     # bootstrap VPS2
# or both at once:
make setup-all
```

**Checklist:**
- Non-root user created with sudo NOPASSWD
- Root login disabled over SSH
- UFW active, SSH port allowed
- Fail2Ban monitoring SSH jail
- `sysctl net.ipv4.ip_forward = 0` (will be overridden by k3s step)

**Status:** See [01-prerequisites.md](01-prerequisites.md)

---

## Phase 3 — k3s Master (Control Plane)

Install the k3s server on VPS1.

```bash
make k3s-master
# → note K3S_NODE_TOKEN, add to .env
```

What this delivers:
- k3s API server (`TCP :6443`)
- Embedded etcd (single-node HA possible later)
- CoreDNS, Flannel VXLAN, Metrics Server
- Secrets encrypted at rest
- UFW rules for pod/service CIDRs

**Checklist:**
- `kubectl get nodes` shows VPS1 as `Ready`
- `K3S_NODE_TOKEN` noted and added to `.env`
- `make kubeconfig` ran and context `k3s-infra` set

**Status:** See [02-kubeadm-setup.md](02-kubeadm-setup.md)

---

## Phase 4 — k3s Worker (Agent Node)

Join VPS2 to the cluster.

```bash
make k3s-worker
```

**Checklist:**
- `kubectl get nodes` shows both nodes `Ready`
- Inter-pod ping test passes across nodes
- Flannel VXLAN traffic flowing (UDP :8472 open)

**Status:** See [04-worker-join.md](04-worker-join.md)

---

## Phase 5 — Base Stack

Deploy ingress, TLS, and routing infrastructure.

```bash
make deploy
DASHBOARD_PASSWORD=<pw> make deploy-dashboard-secret
```

What gets deployed:
1. Namespaces: `apps`, `ingress`, `cert-manager`, `monitoring`
2. **Traefik** `v34.4.0` — HTTP→HTTPS redirect, TLS 1.2+, non-root
3. **cert-manager** `v1.17.1` — Let's Encrypt HTTP-01 ACME
4. **ClusterIssuers** — staging + production
5. **Traefik dashboard** — `https://dashboard.<DOMAIN>` (BasicAuth)

**Checklist:**
- `kubectl get svc -n ingress traefik` shows `EXTERNAL-IP`
- Staging certificate issued successfully for a test domain
- Production certificate issued and trusted
- Dashboard accessible at `https://dashboard.<DOMAIN>`

**Status:** See [03-networking.md](03-networking.md)

---

## Phase 6 — Application Deployment

Deploy your first application to the `apps` namespace.

Use the templates in `kubernetes/apps/` as a starting point:

```bash
# Edit the example app manifests with your image and domain
kubectl apply -f kubernetes/apps/deployment.yaml
kubectl apply -f kubernetes/apps/service-ingress.yaml

# Verify
kubectl get pods -n apps
kubectl get ingress -n apps
kubectl get certificate -n apps
```

**Checklist:**
- Deployment showing desired replicas `Running`
- Ingress created and resolving to the worker IP
- TLS certificate issued and `Ready`
- Application reachable via HTTPS

**Status:** See [05-storage.md](05-storage.md) for stateful applications

---

## Phase 7 — Observability ← you are here

Deploy Prometheus + Grafana + Alertmanager.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values kubernetes/monitoring/kube-prometheus-stack-values.yaml \
  --wait
kubectl apply -f kubernetes/monitoring/grafana-ingress.yaml
```

**Checklist:**
- All monitoring pods `Running` in `monitoring` namespace
- Grafana accessible at `https://grafana.<DOMAIN>`
- Node, pod, and cluster metrics visible in Grafana
- Alertmanager configured with notification channel

**Status:** See [06-observability.md](06-observability.md)

---

## Phase 8 — Secret Management (Vault) ← next

[HashiCorp Vault](https://www.vaultproject.io/) provides dynamic secrets, credential rotation, and encrypted secret storage for Kubernetes workloads.

### Why Vault on k3s?

- Replace hardcoded passwords in Secrets and ConfigMaps
- Dynamic database credentials (Postgres, Redis, etc.)
- PKI certificate authority for internal mTLS
- Audit log for all secret accesses

### High-level plan

```bash
# 1. Add the Vault Helm chart
helm repo add hashicorp https://helm.releases.hashicorp.com

# 2. Install Vault in dev mode (test) or integrated storage (production)
helm upgrade --install vault hashicorp/vault \
  --namespace vault \
  --create-namespace \
  --set "server.ha.enabled=false" \
  --set "server.dataStorage.storageClass=local-path"

# 3. Initialize and unseal Vault
kubectl exec -n vault vault-0 -- vault operator init
kubectl exec -n vault vault-0 -- vault operator unseal <key>

# 4. Enable Kubernetes auth
kubectl exec -n vault vault-0 -- vault auth enable kubernetes

# 5. Inject secrets into pods via the Vault Agent Injector
# (via annotations on pod spec)
```

### Expose Vault via IngressRoute

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: vault
  namespace: vault
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`vault.example.com`)
      kind: Rule
      services:
        - name: vault
          port: 8200
  tls:
    secretName: vault-tls
```

> **Note:** In production, Vault should use raft integrated storage with 3 replicas. For a 2-node cluster, dev mode or standalone with local-path is acceptable.

---

## Phase 9 — GitOps (ArgoCD / Flux) ← next

Replace `kubectl apply` and `helm upgrade` with a Git-driven workflow where every cluster change is a Git commit.

### ArgoCD

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm upgrade --install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --set server.ingress.enabled=false  # managed via IngressRoute

# Expose ArgoCD UI via Traefik IngressRoute
# Get initial admin password:
kubectl get secret -n argocd argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

### Flux

```bash
# Bootstrap Flux from this repo
flux bootstrap github \
  --owner=KevinDeBenedetti \
  --repository=infra \
  --branch=main \
  --path=kubernetes \
  --personal
```

**Why GitOps?**
- Full audit trail of every cluster change
- Easy rollback (git revert)
- Drift detection and auto-reconciliation
- PR-based review for infrastructure changes

---

## Phase 10 — Advanced Security (Falco, OPA/Gatekeeper) ← future

### Falco — Runtime Threat Detection

[Falco](https://falco.org) monitors system calls and detects anomalous behaviour at runtime (unexpected privileged containers, shell in container, unexpected network connections, etc.).

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm upgrade --install falco falcosecurity/falco \
  --namespace falco \
  --create-namespace \
  --set falco.grpc.enabled=true \
  --set falcosidekick.enabled=true \
  --set falcosidekick.config.slack.webhookurl=<WEBHOOK_URL>
```

### OPA Gatekeeper — Policy Enforcement

[OPA Gatekeeper](https://open-policy-agent.github.io/gatekeeper/) enforces custom policies on Kubernetes resources (require labels, deny privileged containers, enforce resource limits, etc.).

```bash
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm upgrade --install gatekeeper gatekeeper/gatekeeper \
  --namespace gatekeeper-system \
  --create-namespace
```

---

## Summary Table

| Phase | Component               | Status    | Doc                                        |
| ----- | ----------------------- | --------- | ------------------------------------------ |
| 0     | Local tools             | ✅ Done    | [01-prerequisites.md](01-prerequisites.md) |
| 1     | VPS provisioning        | ✅ Done    | [01-prerequisites.md](01-prerequisites.md) |
| 2     | OS hardening (dotfiles) | ✅ Done    | [01-prerequisites.md](01-prerequisites.md) |
| 3     | k3s master              | ✅ Done    | [02-kubeadm-setup.md](02-kubeadm-setup.md) |
| 4     | k3s worker              | ✅ Done    | [04-worker-join.md](04-worker-join.md)     |
| 5     | Traefik + cert-manager  | ✅ Done    | [03-networking.md](03-networking.md)       |
| 6     | App deployment          | ✅ Done    | [05-storage.md](05-storage.md)             |
| 7     | Prometheus + Grafana    | 🔄 Next    | [06-observability.md](06-observability.md) |
| 8     | Vault                   | 📋 Planned | This file                                  |
| 9     | ArgoCD / Flux           | 📋 Planned | This file                                  |
| 10    | Falco + OPA             | 🔮 Future  | This file                                  |
