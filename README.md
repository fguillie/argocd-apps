# argocd-apps

GitOps repository for Argo CD application deployments.

## Layout

- `bootstrap/`: one-time Argo CD manifests used to connect a cluster to this repository.
- `apps/`: Argo CD `Application` manifests managed by the bootstrap app.
- `projects/`: optional Argo CD `AppProject` manifests.

## Bootstrap

Apply the root application to an Argo CD-enabled cluster:

```bash
kubectl apply -n argocd -f bootstrap/argocd-apps.yaml
```

After that, Argo CD watches the `apps/` path in this repository.
