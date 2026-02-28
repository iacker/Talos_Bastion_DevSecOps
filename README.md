# Talos Kubernetes Cluster

Single-node Talos Linux Kubernetes cluster for homelab/DevSecOps. All services are exposed via Traefik ingress on `*.talos.local` with Cilium L2 announcements.

## Prerequisites

| Tool | Version | Install |
|------|---------|---------|
| talosctl | >= 1.10.1 | https://www.talos.dev/latest/introduction/getting-started/ |
| kubectl | >= 1.33.0 | https://kubernetes.io/docs/tasks/tools/ |
| helm | >= 3.17 | https://helm.sh/docs/intro/install/ |

**Hardware**: a machine or VM with at least 4 CPU, 8 GB RAM, 50 GB disk, booted from the [Talos ISO](https://github.com/siderolabs/talos/releases).

**Network requirements**:
- The node must be on a LAN subnet (default config uses `192.168.1.0/24`)
- IPs `192.168.1.240–250` must be **free** (not assigned by DHCP or other devices) — they are used for LoadBalancer services
- Cilium L2 announcements use ARP on interfaces matching `^enp.*` or `^eth.*`. If your node's NIC has a different name (e.g. `ens*`, `wlan*`), edit `manifests/cilium-l2-policy.yaml`
- All HTTPS services use a **self-signed wildcard certificate** (`*.talos.local`). Browsers will show a security warning — this is expected

> **WSL users**: WSL is behind NAT and cannot reach LoadBalancer IPs directly. Test from Windows or use `kubectl port-forward`.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Talos Linux v1.10.1                      │
│                  (Immutable, API-driven OS)                 │
├─────────────────────────────────────────────────────────────┤
│                   Kubernetes v1.33.0                        │
├─────────────────────────────────────────────────────────────┤
│  Cilium CNI (L2 Announcements + LB IPAM)                   │
│  └── 192.168.1.240-250 pool on LAN                         │
├─────────────────────────────────────────────────────────────┤
│  Traefik Ingress (192.168.1.240)                           │
│  ├── HTTPS :443 → IngressRoutes (*.talos.local)            │
│  └── TCP   :8443 → Teleport (TLS passthrough)              │
├─────────────────────────────────────────────────────────────┤
│  CoreDNS External (192.168.1.241)                          │
│  └── *.talos.local → 192.168.1.240                         │
├──────────┬──────────┬──────────┬──────────┬────────────────┤
│  ArgoCD  │  Harbor  │ ZeroClaw │ Teleport │   Homepage     │
│  GitOps  │ Registry │    AI    │  Access  │   Dashboard    │
└──────────┴──────────┴──────────┴──────────┴────────────────┘
```

## Folder Structure

```
├── patches/
│   └── cilium.yaml                # Talos machine patch (disable default CNI)
├── helm-values/
│   ├── argocd.yaml                # ArgoCD GitOps server
│   ├── cilium.yaml                # Cilium CNI + Hubble + L2 announcements
│   ├── coredns-external.yaml      # External CoreDNS (wildcard *.talos.local)
│   ├── harbor.yaml                # Container registry
│   ├── homepage.yaml              # Cluster dashboard
│   ├── teleport.yaml              # Zero-trust access
│   ├── traefik.yaml               # Traefik ingress controller
│   └── zeroclaw.yaml              # ZeroClaw AI runtime
├── manifests/
│   ├── cilium-lb-ipam-pool.yaml   # CiliumLoadBalancerIPPool (192.168.1.240-250)
│   ├── cilium-l2-policy.yaml      # CiliumL2AnnouncementPolicy
│   └── ingress/
│       ├── argocd.yaml            # argocd.talos.local
│       ├── harbor.yaml            # harbor.talos.local
│       ├── hubble.yaml            # hubble.talos.local
│       ├── homepage.yaml          # home.talos.local
│       ├── teleport.yaml          # TCP :8443 TLS passthrough
│       ├── traefik-dashboard.yaml # traefik.talos.local
│       └── zeroclaw.yaml          # zeroclaw.talos.local
└── README.md
```

## Services

| Service    | URL                                | Protocol        |
|------------|------------------------------------|-----------------|
| Homepage   | https://home.talos.local           | HTTPS           |
| Harbor     | https://harbor.talos.local         | HTTPS           |
| ArgoCD     | https://argocd.talos.local         | HTTPS           |
| ZeroClaw   | https://zeroclaw.talos.local       | HTTPS           |
| Hubble     | https://hubble.talos.local         | HTTPS           |
| Teleport   | https://teleport.talos.local:8443  | TLS passthrough |
| Traefik    | https://traefik.talos.local        | HTTPS           |
| DNS        | 192.168.1.241:53                   | UDP/TCP         |

> **Note**: Harbor requires `--insecure-registry` in your Docker daemon config since the certificate is self-signed.

## Bootstrap

Replace `<NODE_IP>` below with your Talos node's IP address.

### 1. Generate Talos configs

```bash
talosctl gen config talos-cluster https://<NODE_IP>:6443 \
  --config-patch @patches/cilium.yaml
```

### 2. Apply to node and bootstrap

```bash
talosctl apply-config --insecure -n <NODE_IP> --file controlplane.yaml
talosctl bootstrap -n <NODE_IP> -e <NODE_IP>
```

### 3. Get kubeconfig

```bash
talosctl kubeconfig -n <NODE_IP> -e <NODE_IP>
```

### 4. Remove control-plane taint (single-node only)

On a single-node cluster, the control-plane taint prevents all workloads from scheduling. Remove it:

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane:NoSchedule-
```

### 5. Install Helm charts

```bash
# Add repos
helm repo add cilium https://helm.cilium.io
helm repo add argo https://argoproj.github.io/argo-helm
helm repo add harbor https://helm.goharbor.io
helm repo add homepage https://jameswynn.github.io/helm-charts
helm repo add zeroclaw https://niklasfrick.github.io/zeroclaw-helm
helm repo add teleport https://charts.releases.teleport.dev
helm repo add traefik https://traefik.github.io/charts
helm repo add coredns https://coredns.github.io/helm
helm repo update

# Install Cilium CNI (must be first)
helm install cilium cilium/cilium --version 1.19.0 \
  -n kube-system -f helm-values/cilium.yaml

# Wait for Cilium to be ready before applying CRDs
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=cilium-agent \
  -n kube-system --timeout=120s

# Apply Cilium L2/IPAM CRDs
kubectl apply -f manifests/cilium-lb-ipam-pool.yaml
kubectl apply -f manifests/cilium-l2-policy.yaml

# Install Traefik ingress controller
helm install traefik traefik/traefik --version 39.0.2 \
  -n traefik --create-namespace -f helm-values/traefik.yaml

# Install CoreDNS external (wildcard DNS)
helm install coredns-external coredns/coredns --version 1.45.2 \
  -n coredns --create-namespace -f helm-values/coredns-external.yaml

# Install workloads
helm install argocd argo/argo-cd --version 9.4.1 \
  -n argocd --create-namespace -f helm-values/argocd.yaml

helm install harbor harbor/harbor --version 1.18.2 \
  -n harbor --create-namespace -f helm-values/harbor.yaml \
  --set harborAdminPassword=<PASSWORD>

helm install homepage homepage/homepage --version 2.1.0 \
  -n homepage --create-namespace -f helm-values/homepage.yaml

helm install teleport teleport/teleport-cluster --version 18.6.6 \
  -n teleport --create-namespace -f helm-values/teleport.yaml

helm install zeroclaw zeroclaw/zeroclaw --version 0.2.1 \
  -n zeroclaw --create-namespace -f helm-values/zeroclaw.yaml \
  --set secret.apiKey=<OPENROUTER_API_KEY>

# Wait for Traefik CRDs to be registered, then apply IngressRoutes
kubectl wait --for=condition=established crd/ingressroutes.traefik.io --timeout=60s
kubectl apply -f manifests/ingress/
```

### 6. Configure DNS on client

Point your DNS to `192.168.1.241` so that `*.talos.local` resolves to the Traefik LoadBalancer IP (`192.168.1.240`).

> **Warning**: Setting `192.168.1.241` as primary DNS means all DNS queries go through the in-cluster CoreDNS. If the cluster is down, DNS resolution will fall back to the secondary (`1.1.1.1`) but may be slow.

**Windows (PowerShell as Admin):**
```powershell
# List your network adapters to find the right name
Get-NetAdapter | Format-Table Name, Status

# Set DNS (replace "Wi-Fi" with your adapter name)
Set-DnsClientServerAddress -InterfaceAlias "Wi-Fi" -ServerAddresses ("192.168.1.241","1.1.1.1")
```

**Alternative — hosts file (no DNS change needed):**
```powershell
Add-Content C:\Windows\System32\drivers\etc\hosts "192.168.1.240 home.talos.local argocd.talos.local harbor.talos.local zeroclaw.talos.local hubble.talos.local traefik.talos.local teleport.talos.local"
```

**Linux / macOS:**
```bash
# Option 1: systemd-resolved
sudo resolvectl dns eth0 192.168.1.241 1.1.1.1

# Option 2: /etc/resolv.conf
sudo tee /etc/resolv.conf <<EOF
nameserver 192.168.1.241
nameserver 1.1.1.1
EOF
```

### 7. Verify deployment

```bash
# All pods should be Running
kubectl get pods -A

# Traefik and CoreDNS should have External IPs
kubectl get svc -n traefik
kubectl get svc -n coredns

# Test DNS (from a machine using 192.168.1.241 as DNS)
nslookup home.talos.local 192.168.1.241

# Test HTTPS (accept self-signed cert)
curl -k https://home.talos.local
```

## GitOps with ArgoCD

To make Helm deployments immutable and version-controlled:

1. Push this config to your Git repo (Gitea or GitHub)
2. Create ArgoCD Applications using multi-source to combine the upstream Helm chart with your values from this repo
3. ArgoCD will sync and manage all deployments automatically

Example ArgoCD Application (multi-source):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: homepage
  namespace: argocd
spec:
  project: default
  sources:
    - repoURL: https://jameswynn.github.io/helm-charts
      chart: homepage
      targetRevision: 2.1.0
      helm:
        valueFiles:
          - $values/helm-values/homepage.yaml
    - repoURL: https://gitea.talos.local/your-org/your-repo.git
      targetRevision: main
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: homepage
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## Secrets Management

Sensitive values (passwords, tokens) are NOT stored in this repo. Options:

1. **`--set` flags** (current) — Pass secrets at install time via `--set harborAdminPassword=XXX`
2. **Kubernetes Secrets** — Create secrets manually or via sealed-secrets
3. **ArgoCD + Vault** — External secrets operator for production

## Immutability Layers

| Layer | Tool | Immutable? |
|-------|------|------------|
| OS | Talos Linux | Yes - API-only, no SSH, read-only rootfs |
| Cluster config | talosctl | Yes - declarative machine configs |
| CNI | Cilium | Yes - Helm values in Git |
| Networking | Cilium L2 + Traefik | Yes - LoadBalancer IPs + IngressRoutes in Git |
| DNS | CoreDNS external | Yes - wildcard *.talos.local in Git |
| Workloads | ArgoCD | Yes - GitOps sync from repo |
| Secrets | --set / sealed-secrets | Partial - not in Git, managed externally |
