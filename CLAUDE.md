# Project Context

Single-node Talos Linux Kubernetes cluster for homelab/DevSecOps.
Owner: Erwan. Language: French preferred.

## Cluster Info

- **Node**: `talos-xf5-l0l` (control-plane) at `192.168.1.10`
- **Second node**: `talos-yki-ytm` — currently **NotReady**, ignore it
- **Talos**: v1.10.1 | **Kubernetes**: v1.33.0
- **Control-plane taint**: removed (single-node, all workloads run on control-plane)
- **Dev machine**: Windows + WSL (NixOS). WSL is behind NAT (`172.21.x.x`), cannot reach LB IPs directly. Use `kubectl port-forward` or test from Windows.

## Networking Architecture

- **CNI**: Cilium with L2 announcements + LB IPAM
- **LB IP pool**: `192.168.1.240–250` (CiliumLoadBalancerIPPool `lan-pool`, API `cilium.io/v2`)
- **L2 policy**: CiliumL2AnnouncementPolicy `lan-l2-policy` on interfaces `^enp.*` and `^eth.*`
- **Ingress**: Traefik v3.6.8 (chart v39.x) at `192.168.1.240`
  - HTTPS :443 with self-signed wildcard cert `*.talos.local`
  - TCP :8443 for Teleport TLS passthrough
  - HTTP :80 redirects to HTTPS
- **DNS**: CoreDNS external at `192.168.1.241`, resolves `*.talos.local` → `192.168.1.240`, forwards everything else to `1.1.1.1`/`8.8.8.8`
- **Domain**: `*.talos.local` (not real TLD, LAN only)

## Services (all ClusterIP, exposed via Traefik IngressRoutes)

| Service | Namespace | Chart | URL | Status |
|---------|-----------|-------|-----|--------|
| ArgoCD | argocd | argo/argo-cd 9.4.1 | https://argocd.talos.local | Working |
| Harbor | harbor | harbor/harbor 1.18.2 | https://harbor.talos.local | Working |
| ZeroClaw | zeroclaw | zeroclaw/zeroclaw 0.2.1 | https://zeroclaw.talos.local | Working |
| Homepage | homepage | homepage/homepage 2.1.0 | https://home.talos.local | Working |
| Teleport | teleport | teleport-cluster 18.6.6 | https://teleport.talos.local:8443 | Working |
| Hubble UI | kube-system | (part of cilium) | https://hubble.talos.local | Working |
| Traefik | traefik | traefik/traefik 39.x | https://traefik.talos.local | Working |
| CoreDNS ext | coredns | coredns/coredns | 192.168.1.241:53 | Working |

## Known Issues

- **Gitea removed**: was broken (valkey CrashLoop), replaced by ZeroClaw.
- **WSL network**: WSL is NAT'd and cannot reach `192.168.1.240/241`. Must test from Windows or use `kubectl port-forward`.
- **CoreDNS external pod**: PodSecurity warning (runAsNonRoot) — non-blocking, pod runs fine.

## Helm Repos

```
cilium    https://helm.cilium.io
argo      https://argoproj.github.io/argo-helm
harbor    https://helm.goharbor.io
homepage  https://jameswynn.github.io/helm-charts
zeroclaw  https://niklasfrick.github.io/zeroclaw-helm
teleport  https://charts.releases.teleport.dev
traefik   https://traefik.github.io/charts
coredns   https://coredns.github.io/helm
```

## Deployed Chart Versions

Pin these in `helm install --version` to ensure reproducibility:

| Release | Chart | Version |
|---------|-------|---------|
| cilium | cilium/cilium | 1.19.0 |
| argocd | argo/argo-cd | 9.4.1 |
| harbor | harbor/harbor | 1.18.2 |
| zeroclaw | zeroclaw/zeroclaw | 0.2.1 |
| homepage | homepage/homepage | 2.1.0 |
| teleport | teleport/teleport-cluster | 18.6.6 |
| traefik | traefik/traefik | 39.0.2 |
| coredns-external | coredns/coredns | 1.45.2 |

## File Structure

- `patches/` — Talos machine patches (cilium.yaml disables default CNI)
- `helm-values/` — Helm values for all services
- `manifests/` — Raw Kubernetes manifests (Cilium CRDs, IngressRoutes)
- `README.md` — User-facing documentation
- `CLAUDE.md` — This file (project context for Claude Code sessions)
- `.gitignore` — Excludes talosconfig, kubeconfig, controlplane.yaml, worker.yaml

## Conventions

- Git commit messages: `feat:`, `fix:`, `docs:` prefix style
- All services use ClusterIP (no NodePort), exposed through Traefik IngressRoutes
- Secrets are NOT in the repo — use `--set` flags at install time
- Traefik chart v39 schema: `ports.web.http.redirections`, `ports.websecure.http.tls` (not top-level)
- CiliumLoadBalancerIPPool uses `apiVersion: cilium.io/v2` (not v2alpha1)
- README uses `<NODE_IP>` placeholder — the actual node IP is in Cluster Info above
- Helm install commands in README are pinned to exact `--version` for reproducibility
