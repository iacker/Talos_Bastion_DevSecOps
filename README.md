# Talos Kubernetes Cluster

Configuration for a single-node Talos Linux cluster running on `192.168.1.20`.

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
│  ├── TCP   :22  → Gitea SSH                                │
│  └── TCP   :8443 → Teleport (TLS passthrough)              │
├─────────────────────────────────────────────────────────────┤
│  CoreDNS External (192.168.1.241)                          │
│  └── *.talos.local → 192.168.1.240                         │
├──────────┬──────────┬──────────┬──────────┬────────────────┤
│  ArgoCD  │  Harbor  │  Gitea   │ Teleport │   Homepage     │
│  GitOps  │ Registry │   Git    │  Access  │   Dashboard    │
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
│   ├── gitea.yaml                 # Git server
│   ├── harbor.yaml                # Container registry
│   ├── homepage.yaml              # Cluster dashboard
│   ├── teleport.yaml              # Zero-trust access
│   └── traefik.yaml               # Traefik ingress controller
├── manifests/
│   ├── cilium-lb-ipam-pool.yaml   # CiliumLoadBalancerIPPool (192.168.1.240-250)
│   ├── cilium-l2-policy.yaml      # CiliumL2AnnouncementPolicy
│   └── ingress/
│       ├── argocd.yaml            # argocd.talos.local
│       ├── harbor.yaml            # harbor.talos.local
│       ├── gitea.yaml             # gitea.talos.local
│       ├── gitea-ssh.yaml         # TCP :22 passthrough
│       ├── hubble.yaml            # hubble.talos.local
│       ├── homepage.yaml          # home.talos.local
│       ├── teleport.yaml          # TCP :8443 TLS passthrough
│       └── traefik-dashboard.yaml # traefik.talos.local
└── README.md
```

## Services

| Service    | URL                                | Protocol        |
|------------|------------------------------------|-----------------|
| Homepage   | https://home.talos.local           | HTTPS           |
| Harbor     | https://harbor.talos.local         | HTTPS           |
| Gitea      | https://gitea.talos.local          | HTTPS           |
| Gitea SSH  | ssh://git@192.168.1.240:22         | TCP             |
| ArgoCD     | https://argocd.talos.local         | HTTPS           |
| Hubble     | https://hubble.talos.local         | HTTPS           |
| Teleport   | https://teleport.talos.local:8443  | TLS passthrough |
| Traefik    | https://traefik.talos.local        | HTTPS           |
| DNS        | 192.168.1.241:53                   | UDP/TCP         |

## Bootstrap Commands

### 1. Generate Talos configs

```bash
talosctl gen config talos-cluster https://192.168.1.20:6443 \
  --config-patch @talos/patches/cilium.yaml
```

### 2. Apply to node and bootstrap

```bash
talosctl apply-config --insecure -n 192.168.1.20 --file controlplane.yaml
talosctl bootstrap -n 192.168.1.20 -e 192.168.1.20
```

### 3. Get kubeconfig

```bash
talosctl kubeconfig -n 192.168.1.20 -e 192.168.1.20
```

### 4. Install Helm charts

```bash
# Add repos
helm repo add cilium https://helm.cilium.io
helm repo add argo https://argoproj.github.io/argo-helm
helm repo add harbor https://helm.goharbor.io
helm repo add gitea https://dl.gitea.io/charts
helm repo add homepage https://jameswynn.github.io/helm-charts
helm repo add teleport https://charts.releases.teleport.dev
helm repo add traefik https://traefik.github.io/charts
helm repo add coredns https://coredns.github.io/helm

# Install Cilium CNI (must be first)
helm install cilium cilium/cilium -n kube-system -f helm-values/cilium.yaml

# Apply Cilium L2/IPAM CRDs
kubectl apply -f manifests/cilium-lb-ipam-pool.yaml
kubectl apply -f manifests/cilium-l2-policy.yaml

# Install Traefik ingress controller
helm install traefik traefik/traefik -n traefik --create-namespace -f helm-values/traefik.yaml

# Install CoreDNS external (wildcard DNS)
helm install coredns-external coredns/coredns -n coredns --create-namespace -f helm-values/coredns-external.yaml

# Install workloads
helm install argocd argo/argo-cd -n argocd --create-namespace -f helm-values/argocd.yaml
helm install harbor harbor/harbor -n harbor --create-namespace -f helm-values/harbor.yaml --set harborAdminPassword=<PASSWORD>
helm install gitea gitea/gitea -n gitea --create-namespace -f helm-values/gitea.yaml --set gitea.admin.password=<PASSWORD>
helm install homepage homepage/homepage -n homepage --create-namespace -f helm-values/homepage.yaml
helm install teleport teleport/teleport-cluster -n teleport --create-namespace -f helm-values/teleport.yaml

# Apply IngressRoutes
kubectl apply -f manifests/ingress/
```

### 5. Configure DNS on client

Point your DNS to `192.168.1.241` so that `*.talos.local` resolves to `192.168.1.240`.

**Windows (PowerShell as Admin):**
```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses ("192.168.1.241","1.1.1.1")
```

**WSL:**
```bash
# /etc/resolv.conf (or via /etc/wsl.conf generateResolvConf=false)
nameserver 192.168.1.241
nameserver 1.1.1.1
```

## GitOps with ArgoCD (Immutable Deployments)

To make Helm deployments immutable and version-controlled:

1. Push this config to your Git repo (Gitea or GitHub)
2. Create ArgoCD Applications pointing to `helm-values/`
3. ArgoCD will sync and manage all deployments automatically

Example ArgoCD Application:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: homepage
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://gitea.talos.local/your-org/your-repo.git
    path: helm-values
    targetRevision: main
    helm:
      valueFiles:
        - homepage.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: homepage
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## Secrets Management

Sensitive values (passwords, tokens) are NOT stored in this repo. Use one of:

1. **sops-nix** (current) - Encrypted in `secrets/cluster.yaml`, decrypted at boot to `/run/secrets/`
2. **Kubernetes Secrets** - Create secrets manually or via sealed-secrets
3. **ArgoCD + Vault** - External secrets operator for production

## Immutability Layers

| Layer | Tool | Immutable? |
|-------|------|------------|
| OS | Talos Linux | Yes - API-only, no SSH, read-only rootfs |
| Cluster config | talosctl | Yes - declarative machine configs |
| CNI | Cilium | Yes - Helm values in Git |
| Networking | Cilium L2 + Traefik | Yes - LoadBalancer IPs + IngressRoutes in Git |
| DNS | CoreDNS external | Yes - wildcard *.talos.local in Git |
| Workloads | ArgoCD | Yes - GitOps sync from repo |
| Secrets | sops-nix | Yes - encrypted in Git, decrypted at runtime |
