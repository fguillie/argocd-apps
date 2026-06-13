# AGENTS.md

This repository is the GitOps source for Argo CD application deployments.

## Repository Layout

- `bootstrap/argocd-apps.yaml`: root Argo CD app-of-apps manifest.
- `apps/`: child Argo CD `Application` manifests watched by the root app.
- `workloads/`: Kubernetes manifests deployed by child apps.
- `projects/`: optional Argo CD `AppProject` manifests.

Keep Argo CD `Application` objects in `apps/`. Put workload Kubernetes
resources under `workloads/<name>/` and point the child app at that path.

## Fresh Cluster Activation

Install Argo CD:

```bash
kubectl create namespace argocd

kubectl apply --server-side --force-conflicts -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for Argo CD:

```bash
kubectl -n argocd rollout status deployment/argocd-server --timeout=300s
kubectl -n argocd rollout status deployment/argocd-repo-server --timeout=300s
kubectl -n argocd rollout status statefulset/argocd-application-controller --timeout=300s
```

Bootstrap this repository:

```bash
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/fguillie/argocd-apps/main/bootstrap/argocd-apps.yaml
```

Use server-side apply for the Argo CD install because the upstream
`applicationsets.argoproj.io` CRD can exceed the client-side apply annotation
limit. This is distinct from Argo CD's `ServerSideApply=true` sync option,
which is not currently set on the child apps in this repository.

## Application Ordering

The root `argocd-apps` app watches `apps/` recursively.

Current child app waves:

- `gpu-operator`: `argocd.argoproj.io/sync-wave: "0"`
- `pytorch`: `argocd.argoproj.io/sync-wave: "10"`

Sync waves order creation of the child `Application` objects. They do not make
Argo CD wait for the GPU Operator child app to become fully healthy before
creating the PyTorch child app. The PyTorch pod requests `nvidia.com/gpu: 1`, so
Kubernetes keeps it pending until the NVIDIA device plugin advertises GPU
capacity.

## Verification

Check Argo CD apps:

```bash
kubectl -n argocd get applications
```

Expected apps:

```text
argocd-apps
gpu-operator
pytorch
```

Check GPU Operator readiness:

```bash
kubectl get clusterpolicy cluster-policy
kubectl -n gpu-operator get daemonsets,deployments
```

Check node GPU capacity:

```bash
kubectl describe node <node-name> | grep -A10 -E 'Capacity:|Allocatable:'
```

Check PyTorch:

```bash
kubectl -n pytorch get pod pytorch
kubectl -n pytorch logs pod/pytorch --tail=5
kubectl -n pytorch exec -i pod/pytorch -- python - <<'PY'
import torch
print('torch', torch.__version__)
print('cuda_available', torch.cuda.is_available())
print('device_count', torch.cuda.device_count())
if torch.cuda.is_available():
    print('device_name', torch.cuda.get_device_name(0))
PY
```

`pytorch` may show `Progressing` in Argo CD because it is a standalone
long-running Pod. The pod should be `Running` and `Ready`.

## Operational Notes

- Keep chart versions pinned. Current GPU Operator chart: `v26.3.2`.
- The GPU Operator app enables the CDI NRI plugin with Helm parameter
  `cdi.nriPluginEnabled=true` in `apps/nvidia-gpu-operator.yaml`.
- Keep workload image tags pinned. Current PyTorch image:
  `nvcr.io/nvidia/pytorch:26.04-py3`.
- Prefer changes through Git and Argo CD. Avoid manual changes to managed
  resources except for one-time bootstrap or teardown tests.
- If testing a full reinstall, delete the root app first, then child apps, then
  app namespaces, then Argo CD.
