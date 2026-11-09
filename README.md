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
├── local-path-provisioner.yaml ──► platform/local-path-provisioner (default StorageClass)
├── sealed-secrets.yaml         ──► Helm chart sealed-secrets 2.22.0 (encrypted secrets in git)
└── kube-prometheus-stack.yaml  ──► Helm chart 92.14.0 + values/kube-prometheus-stack.yaml
                                    + platform/monitoring/ (Grafana admin SealedSecret)
```

`root` is an Argo CD Application whose only job is to create the other
Applications. Adding a component to the cluster = committing one more file
to `apps/`. Deleting the file removes the component (`prune: true`).

Every upstream dependency is pinned (Argo CD `v3.6.1`, local-path-provisioner
`v0.0.37`, sealed-secrets chart `2.22.0`, kube-prometheus-stack chart `92.14.0`), so the cluster only changes when this repo changes.

## Layout

```
bootstrap/
  argocd/          Argo CD install: pinned upstream manifest + kustomize patches
  root-app.yaml    the root Application (app of apps)
apps/              one Argo CD Application per component
platform/          extra manifests per component (e.g. SealedSecrets)
values/            Helm values for charts deployed from apps/
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

Any node IP works for every NodePort.

| Service | URL |
|---|---|
| Argo CD | https://192.168.122.152:30443 (self-signed cert) |
| Grafana | http://192.168.122.152:30000 |
| Prometheus | http://192.168.122.152:30090 |
| Alertmanager | http://192.168.122.152:30093 |

```bash
# initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
```

## Secrets

Plain Kubernetes Secrets are only base64-encoded, so they never go in this
repo. Instead they're encrypted with the cluster's public key using
[Sealed Secrets](https://github.com/bitnami/sealed-secrets), and only the
controller running in the cluster can decrypt them:

```bash
kubectl create secret generic my-secret -n my-ns --from-literal=key=value \
  --dry-run=client -o yaml | kubeseal --format yaml > path/to/my-secret.sealed.yaml
```

The decryption key lives only in the cluster. **Back it up outside git** —
without it, a rebuilt cluster can't decrypt anything committed here:

```bash
kubectl -n kube-system get secret -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > sealed-secrets-key.backup.yaml
```

## Roadmap

- [x] Argo CD (self-managed)
- [x] Dynamic storage: local-path-provisioner
- [x] Sealed Secrets
- [x] kube-prometheus-stack (Prometheus Operator, Alertmanager, Grafana)
- [ ] Scrape etcd / scheduler / controller-manager / kube-proxy (needs kubeadm bind-address changes)
- [ ] Loki + Alloy for logs
- [ ] Monitoring the laptop (hypervisor) itself: node_exporter + Alloy
