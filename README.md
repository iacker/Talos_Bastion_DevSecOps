# Talos Kubernetes Cluster

Configuration for a single-node Talos Linux cluster running on `192.168.1.20`.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Talos Linux v1.10.1                      │
│                  (Immutable, API-driven OS)                 │
├─────────────────────────────────────────────────────────────┤
│                   Kubernetes v1.33.0                        │
├──────────┬──────────┬──────────┬──────────┬────────────────┤
│  Cilium  │  ArgoCD  │  Harbor  │  Gitea   │   Teleport     │
│   CNI    │  GitOps  │ Registry │   Git    │    Access      │
│ + Hubble │          │          │          │                │
└──────────┴──────────┴──────────┴──────────┴────────────────┘
```

## Folder Structure

```
talos/
├── patches/
│   └── cilium.yaml          # Talos machine patch (disable default CNI)
├── helm-values/
│   ├── argocd.yaml          # ArgoCD GitOps server
│   ├── cilium.yaml          # Cilium CNI + Hubble observability
│   ├── gitea.yaml           # Git server
│   ├── harbor.yaml          # Container registry
│   ├── homepage.yaml        # Cluster dashboard
│   └── teleport.yaml        # Zero-trust access
└── README.md
```

## Services

| Service  | Port  | URL                          | Purpose              |
|----------|-------|------------------------------|----------------------|
| Homepage | 30000 | http://192.168.1.20:30000    | Cluster dashboard    |
| Harbor   | 30080 | http://192.168.1.20:30080    | Container registry   |
| Gitea    | 30300 | http://192.168.1.20:30300    | Git server           |
| ArgoCD   | 30880 | http://192.168.1.20:30880    | GitOps CD            |
| Hubble   | 31235 | http://192.168.1.20:31235    | Network observability|
| Teleport | 32687 | https://192.168.1.20:32687   | Zero-trust access    |

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

# Install (order matters for CNI)
helm install cilium cilium/cilium -n kube-system -f talos/helm-values/cilium.yaml
helm install argocd argo/argo-cd -n argocd --create-namespace -f talos/helm-values/argocd.yaml
helm install harbor harbor/harbor -n harbor --create-namespace -f talos/helm-values/harbor.yaml --set harborAdminPassword=<PASSWORD>
helm install gitea gitea/gitea -n gitea --create-namespace -f talos/helm-values/gitea.yaml --set gitea.admin.password=<PASSWORD>
helm install homepage homepage/homepage -n homepage --create-namespace -f talos/helm-values/homepage.yaml
helm install teleport teleport/teleport-cluster -n teleport --create-namespace -f talos/helm-values/teleport.yaml
```

## GitOps with ArgoCD (Immutable Deployments)

To make Helm deployments immutable and version-controlled:

1. Push this config to your Git repo (Gitea or GitHub)
2. Create ArgoCD Applications pointing to `talos/helm-values/`
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
    repoURL: http://192.168.1.20:30300/your-org/your-repo.git
    path: talos/helm-values
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
| Workloads | ArgoCD | Yes - GitOps sync from repo |
| Secrets | sops-nix | Yes - encrypted in Git, decrypted at runtime |
