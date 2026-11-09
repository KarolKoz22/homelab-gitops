# homelab-gitops

GitOps configuration for a 3-node kubeadm Kubernetes cluster running on KVM
VMs on my laptop. [Argo CD](https://argo-cd.readthedocs.io/) syncs everything
in this repo to the cluster; nothing is `kubectl apply`'d by hand except the
two bootstrap commands below.

## Cluster

| Node | Role | IP |
|------|------|----|
| `k8s-control` | control plane (tainted `NoSchedule`) | 192.168.122.157 |
| `k8s-worker1` | worker | 192.168.122.152 |
| `k8s-worker2` | worker | 192.168.122.64 |

Kubernetes v1.31, containerd, Flannel CNI. Built with kubeadm. The step-by-step
build guide is in a separate repo (`k8s-kubeadm-setup`), and nodes are provisioned
with Ansible (`../ansible`).

## How it works: app of apps

```
bootstrap/root-app.yaml   ──► watches apps/
apps/
├── argocd.yaml                 ──► bootstrap/argocd               (Argo CD manages itself)
└── local-path-provisioner.yaml ──► platform/local-path-provisioner (default StorageClass)
```

`root` is an Argo CD Application whose only job is to create the other
Applications. Adding a component to the cluster = committing one more file
to `apps/`. Deleting the file removes the component (`prune: true`).

Every upstream dependency is pinned (Argo CD `v3.6.1`, local-path-provisioner
`v0.0.37`), so the cluster only changes when this repo changes.

## Layout

```
bootstrap/
  argocd/          Argo CD install: pinned upstream manifest + kustomize patches
  root-app.yaml    the root Application (app of apps)
apps/              one Argo CD Application per component
platform/          cluster-level components (storage, later: monitoring, secrets)
```

## Bootstrap (one time, on a fresh cluster)

```bash
# 1. Install Argo CD (server-side: its CRDs are too large for client-side apply)
kubectl apply -k bootstrap/argocd --server-side --force-conflicts

# 2. Hand control over to git
kubectl apply -f bootstrap/root-app.yaml
```

From here on, every change goes through a commit.

## Access

Argo CD UI: https://192.168.122.152:30443 (any node IP works; self-signed cert)

```bash
# initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
```

## Roadmap

- [x] Argo CD (self-managed)
- [x] Dynamic storage: local-path-provisioner
- [ ] Sealed Secrets
- [ ] kube-prometheus-stack (Prometheus Operator, Alertmanager, Grafana)
- [ ] Loki + Alloy for logs
- [ ] Monitoring the laptop (hypervisor) itself: node_exporter + Alloy
